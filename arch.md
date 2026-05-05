# 乳腺癌副作用评估智能体 — MVP 设计方案

## Context

构建一个最小可运行原型（MVP），让乳腺癌患者输入副作用症状描述，系统基于规则引擎返回风险等级、建议、是否联系团队及依据。系统需支持前端三页面、后端四接口、规则化风险分层、数据持久化、可观测性埋点及审计追踪。同时需将系统设计为一个具备"感知-决策-执行-学习"闭环的智能体。

---

## 1. 系统架构图（文字描述）

```
┌─────────────────────────────────────────────────────────────────────┐
│                         前端 (Web UI)                                │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │
│  │ 输入页    │  │ 结果页    │  │ 历史记录页 │                          │
│  │ 症状描述  │  │ 风险等级  │  │ 评估列表  │                          │
│  │ 表单提交  │  │ 建议/联系 │  │ 筛选/查看 │                          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                          │
│       │             │             │                                  │
│       └─────────────┼─────────────┘                                  │
│                     │  REST API (JSON)                               │
└─────────────────────┼───────────────────────────────────────────────┘
                      │
┌─────────────────────┼───────────────────────────────────────────────┐
│               后端 API Server (FastAPI)                              │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ POST /assessments│  │ GET /assessments  │  │ POST /assessments│  │
│  │   提交评估        │  │   /{id} 获取结果  │  │  /{id}/contact   │  │
│  │                  │  │ GET /assessments  │  │   创建协同请求    │  │
│  │                  │  │   获取历史列表    │  │                  │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                     │              │
│           └─────────────────────┼─────────────────────┘              │
│                                 │                                    │
│                    ┌────────────┴────────────┐                      │
│                    │    风险评估引擎          │                      │
│                    │  ┌──────────────────┐   │                      │
│                    │  │ 规则匹配器        │   │                      │
│                    │  │ (关键词+模式)     │   │                      │
│                    │  ├──────────────────┤   │                      │
│                    │  │ 风险分类器        │   │                      │
│                    │  │ (高/中/低)        │   │                      │
│                    │  ├──────────────────┤   │                      │
│                    │  │ 建议生成器        │   │                      │
│                    │  │ (模板+上下文)     │   │                      │
│                    │  ├──────────────────┤   │                      │
│                    │  │ 审计追踪器        │   │                      │
│                    │  │ (规则/时间/版本)  │   │                      │
│                    │  └──────────────────┘   │                      │
│                    └────────────┬────────────┘                      │
│                                 │                                    │
│                    ┌────────────┴────────────┐                      │
│                    │     事件总线             │                      │
│                    │  assessment_started      │                      │
│                    │  assessment_submitted    │                      │
│                    │  result_viewed           │                      │
│                    │  contact_team_clicked    │                      │
│                    │  assessment_closed       │                      │
│                    └────────────┬────────────┘                      │
│                                 │                                    │
└─────────────────────────────────┼────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────────────────────┐
│                         数据存储 (SQLite)                             │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │
│  │ assessment │ │   advice   │ │  evidence  │ │ rule_source│       │
│  │ 评估记录    │ │  建议记录   │ │  依据记录   │ │  规则来源   │       │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘       │
│                        ┌────────────┐                                │
│                        │ event_log  │                                │
│                        │  事件日志   │                                │
│                        └────────────┘                                │
└──────────────────────────────────────────────────────────────────────┘
```

### 数据流

```
用户输入症状描述
       │
       ▼
[1] 前端触发 assessment_started 事件
       │
       ▼
[2] POST /api/assessments {symptoms, patient_info}
       │
       ▼
[3] 风险评估引擎接收输入
       │
       ├─→ 规则匹配器：遍历规则库，关键词+模式匹配
       │         │
       │         ▼
       ├─→ 命中规则 → 确定风险等级 (高/中/低)
       │         │
       │         ▼
       ├─→ 建议生成器：根据风险等级+命中规则生成建议
       │         │
       │         ▼
       ├─→ 审计追踪：记录 matched_rule_id, generated_at, rule_version
       │
       ▼
[4] 持久化：写入 assessment, advice, evidence, event_log 表
       │
       ▼
[5] 触发 assessment_submitted 事件
       │
       ▼
[6] 返回结果给前端 {risk_level, advice, contact_team, evidence}
       │
       ▼
[7] 用户查看结果页面 → 触发 result_viewed 事件
       │
       ├─→ 用户点击"联系团队" → POST /api/assessments/{id}/contact
       │         └─→ 触发 contact_team_clicked 事件
       │
       └─→ 用户关闭/返回 → 触发 assessment_closed 事件
```

---

## 2. "感知-决策-执行-学习" 闭环设计

### 2.1 感知层 (Perception)

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **症状文本采集** | 前端表单：自由文本 + 结构化选择题（症状类型、持续时间、严重程度 1-10） | HTML5 + vanilla JS / 轻量 React |
| **自然语言理解** | 双层策略：① 中文分词 + 关键词提取（jieba）做规则匹配 ② 可选 LLM 增强：调用大模型做语义理解，提取结构化症状实体 | jieba 分词 / 可选 OpenAI-compatible API |
| **上下文聚合** | 合并当前输入 + 患者历史评估记录（若存在），构建完整评估上下文 | 服务层上下文组装器 |
| **输入校验** | 必填校验、敏感词过滤、输入长度限制 | Pydantic 模型校验 |

**感知层数据产出**：
```json
{
  "symptoms_raw": "患者输入的原始文本",
  "symptoms_structured": ["发热", "疼痛", "恶心"],
  "severity": 7,
  "duration_hours": 48,
  "patient_context": { "history": [], "current_medications": [] }
}
```

### 2.2 决策层 (Decision)

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **规则引擎** | 基于优先级排序的规则链，每条规则包含：条件（关键词/模式）、风险等级、建议模板、依据来源、版本号 | Python 自定义规则引擎 |
| **风险分类** | 三级分层：高风险（立即就医关键词：呼吸困难、高热不退、意识模糊等）/ 中风险（需关注：持续疼痛、发热3天、食欲下降等）/ 低风险（常规症状：轻微疲劳、局部不适等） | 规则匹配 + 严重度评分加权 |
| **建议生成** | 模板化建议 + 动态参数填充（症状名、持续时间等上下文） | Jinja2 模板 / Python f-string |
| **联系团队判定** | 高风险 → 强制建议联系；中风险 → 可选建议；低风险 → 不建议 | 规则引擎直接输出布尔值 |
| **置信度评估** | 规则命中率 + 关键词覆盖度 → 给出置信度分（后续可用于触发人工审核） | 计分算法 |

**规则示例**：
```python
# 规则数据结构
{
    "rule_id": "R001",
    "rule_name": "呼吸困难高风险",
    "version": "1.0.0",
    "priority": 1,
    "conditions": {
        "keywords": ["呼吸困难", "喘不过气", "气短", "窒息"],
        "severity_min": 6,
        "duration_hours_max": null
    },
    "risk_level": "high",
    "advice_template": "您提到{keyword}，这是需要立即关注的症状。建议立即前往就近医院就诊。",
    "contact_team": true,
    "evidence": "呼吸困难可能是药物过敏、肺栓塞或感染加重的表现（NCCN指南 2024）",
    "rule_source": "NCCN Guidelines v2024 + 临床专家共识"
}
```

### 2.3 执行层 (Execution)

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **结果展示** | 前端结果页渲染：风险等级（颜色标识）、建议文本、联系按钮、依据来源 | 前端组件 |
| **协同请求** | POST /assessments/{id}/contact → 创建协同工单（当前MVP可先落库，后续对接通知系统） | API + 数据库写入 |
| **事件发布** | 每个关键动作通过事件总线写入 event_log，同时输出结构化日志 | Python logging + event_log 表 |
| **错误处理** | API 层统一异常处理，规则引擎匹配兜底规则（"无法判断，建议联系团队"） | FastAPI exception handlers |

**执行链路追踪**：
```
request_id → assessment_id → rule_id → event_log (含时间戳、用户标识、会话标识)
```

### 2.4 学习层 (Learning)

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **规则效果分析** | 统计每条规则的命中频次、各风险等级分布、联系团队转化率 | SQL 聚合查询 + event_log |
| **规则迭代** | 根据命中分析调整规则优先级、关键词权重、风险阈值 | 手动分析 → 人工更新规则版本 |
| **反馈收集** | 预留"结果是否有帮助"反馈入口（MVP可做简单👍👎），写入 event_log | 前端按钮 + API |
| **误判回溯** | 通过 event_log 串联完整用户路径，定位误判环节 | event_log 链路查询 |
| **知识库更新** | 医学指南更新时，新增/修改规则并升级版本号 | 规则配置文件版本化 |

**学习闭环可视化**：
```
评估数据积累 → event_log 分析 → 规则效果评估 → 规则优化/新增
      ↑                                                    │
      └──────────── 新规则上线，版本号 +1 ←──────────────────┘
```

---

## 3. 技术栈总览

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端 | HTML + vanilla JS + Tailwind CSS | 最简可运行，无需构建工具 |
| 后端 | Python 3.11+ + FastAPI | 异步支持、自动文档、数据校验 |
| 数据库 | SQLite + SQLAlchemy | 零配置、单文件、ORM 支持 |
| 分词 | jieba | 中文症状关键词提取 |
| 规则引擎 | 自定义 Python RuleEngine | 规则配置文件 (YAML/JSON)，版本化管理 |
| 可观测性 | Python logging (JSON格式) + event_log 表 | 结构化日志 + 数据库事件追踪 |
| 部署 | 单进程 FastAPI + uvicorn | MVP 阶段单机部署 |

## 4. 数据库模型

### 5 张核心表

| 表名 | 核心字段 | 说明 |
|------|---------|------|
| `assessments` | id, patient_id, symptoms_raw, symptoms_structured(JSON), severity, risk_level, matched_rule_id, created_at | 评估主记录 |
| `advice` | id, assessment_id, advice_text, contact_team(bool), generated_at | 建议记录，1:1 关联评估 |
| `evidence` | id, assessment_id, rule_id, rule_name, rule_version, evidence_text, rule_source | 依据/审计记录，记录命中规则详情 |
| `rule_definitions` | id, rule_id, rule_name, version, priority, conditions(JSON), risk_level, advice_template, evidence_text, rule_source, is_active, created_at | 规则定义表（规则即数据） |
| `event_log` | id, assessment_id, event_type, event_data(JSON), client_timestamp, server_timestamp, session_id | 事件日志，5 种事件类型 |

### 5 个埋点事件

```
assessment_started     → 用户打开输入页面
assessment_submitted   → 用户提交评估表单
result_viewed          → 用户查看评估结果
contact_team_clicked   → 用户点击"联系团队"按钮
assessment_closed      → 用户关闭结果页 / 返回列表
```

## 5. 审计要求实现

每次评估结果通过 `evidence` 表记录：
- **命中了哪条规则** → `evidence.rule_id` + `evidence.rule_name`
- **生成时间** → `evidence.generated_at` + `advice.generated_at`
- **版本号** → `evidence.rule_version`

通过 API 返回给前端时，结果 JSON 中明确包含 `audit` 字段：
```json
{
  "risk_level": "high",
  "advice": "...",
  "contact_team": true,
  "audit": {
    "matched_rule_id": "R001",
    "matched_rule_name": "呼吸困难高风险",
    "rule_version": "1.0.0",
    "generated_at": "2026-05-05T12:00:00Z",
    "evidence": "呼吸困难可能是药物过敏、肺栓塞或感染加重的表现（NCCN指南 2024）"
  }
}
```

## 6. 项目文件结构

```
HealthHub/
├── main.py                  # FastAPI 入口，路由注册
├── models.py                # SQLAlchemy 模型定义
├── schemas.py               # Pydantic 请求/响应模型
├── rule_engine/
│   ├── __init__.py
│   ├── engine.py            # 规则引擎核心：加载、匹配、执行
│   ├── rules.yaml           # 规则定义文件（版本化管理）
│   └── matcher.py           # 关键词/模式匹配器（jieba 分词）
├── services/
│   ├── __init__.py
│   ├── assessment.py        # 评估业务逻辑
│   └── event.py             # 事件埋点服务
├── static/
│   ├── index.html           # 输入页
│   ├── result.html          # 结果页
│   └── history.html         # 历史记录页
├── requirements.txt         # Python 依赖
└── README.md                # 运行说明
```

## 7. 验证方案

1. **启动验证**：`uvicorn main:app --reload`，访问 `http://localhost:8000` 确认三页面可访问
2. **接口验证**：通过 Swagger UI (`/docs`) 测试 4 个 API 端点
3. **规则验证**：输入预定义的典型症状（高/中/低各 2 条），验证返回结果正确
4. **事件验证**：执行完整用户流程后，查询 `event_log` 表确认 5 个事件均已写入
5. **审计验证**：查询 `evidence` 表确认 rule_id、rule_version、generated_at 均正确记录
6. **端到端验证**：
   - 打开页面 → event_log 有 `assessment_started`
   - 输入"我发烧39度3天了，感觉喘不过气" → 返回高风险 + R001规则命中
   - 查看结果 → event_log 有 `assessment_submitted` + `result_viewed`
   - 点击"联系团队" → event_log 有 `contact_team_clicked`
   - 返回列表 → event_log 有 `assessment_closed`
