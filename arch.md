# 乳腺癌副作用评估智能体 — MVP 设计方案

## Context

构建一个最小可运行原型（MVP），让乳腺癌患者输入副作用症状描述，系统返回风险等级、建议、是否联系团队及依据。核心设计理念为 **LLM-in-the-loop + Rule-as-guardrail**：以 LLM 作为主推理引擎，承担语义理解、多证据链推理和个性化建议生成；规则引擎作为安全兜底网络，对 LLM 输出做强制校验和风险升级。系统同时遵循"感知-决策-执行-学习"智能体闭环。

---

## 1. 系统架构图（文字描述）

```
┌──────────────────────────────────────────────────────────────────────┐
│                         前端 (Web UI)                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                         │
│  │ 输入页    │   │ 结果页    │   │ 历史记录页 │                         │
│  │ 症状描述  │   │ 风险等级  │   │ 评估列表  │                         │
│  │ 表单提交  │   │ 建议/联系 │   │ 筛选/查看 │                         │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                         │
│       │              │              │                                 │
│       └──────────────┼──────────────┘                                 │
│                      │  REST API (JSON)                               │
└──────────────────────┼───────────────────────────────────────────────┘
                       │
┌──────────────────────┼───────────────────────────────────────────────┐
│                后端 API Server (FastAPI)                               │
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │ POST /assessments│  │ GET /assessments  │  │ POST /assessments│   │
│  │   提交评估        │  │   /{id} 获取结果  │  │  /{id}/contact   │   │
│  │                  │  │ GET /assessments  │  │   创建协同请求    │   │
│  │                  │  │   获取历史列表    │  │                  │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                     │               │
│           └─────────────────────┼─────────────────────┘               │
│                                 │                                     │
│                    ┌────────────┴────────────┐                       │
│                    │    LLM 推理引擎 (主引擎)  │                       │
│                    │  ┌──────────────────┐   │                       │
│                    │  │ 症状语义理解      │   │  ← 非标准表达、方言、 │
│                    │  │ (实体提取+关联)   │   │    比喻、多症状关联    │
│                    │  ├──────────────────┤   │                       │
│                    │  │ 多证据链推理      │   │  ← 症状组合联合推理、  │
│                    │  │ (风险判定+解释)   │   │    可解释推理链生成    │
│                    │  ├──────────────────┤   │                       │
│                    │  │ 个性化建议生成    │   │  ← 结合病史+用药、    │
│                    │  │ (共情+上下文)     │   │    动态调整措辞        │
│                    │  └────────┬─────────┘   │                       │
│                    └───────────┼─────────────┘                       │
│                                │                                      │
│                    ┌───────────┴─────────────┐                       │
│                    │  规则安全网 (Guardrails) │                       │
│                    │  ┌──────────────────┐   │                       │
│                    │  │ 高危关键词强制拦截│   │  ← LLM 漏掉高危信号   │
│                    │  │                  │   │    时强制升级风险等级   │
│                    │  ├──────────────────┤   │                       │
│                    │  │ 结构化信息注入    │   │  ← 就医指引、医院地址  │
│                    │  │                  │   │    电话等关键信息      │
│                    │  ├──────────────────┤   │                       │
│                    │  │ 规则版本追踪      │   │  ← 每次拦截记录触发    │
│                    │  │                  │   │    规则+版本+时间      │
│                    │  └──────────────────┘   │                       │
│                    └───────────┬─────────────┘                       │
│                                │                                      │
│                    ┌───────────┴─────────────┐                       │
│                    │      事件总线            │                       │
│                    │  assessment_started      │                       │
│                    │  assessment_submitted    │                       │
│                    │  result_viewed           │                       │
│                    │  contact_team_clicked    │                       │
│                    │  assessment_closed       │                       │
│                    └───────────┬─────────────┘                       │
│                                │                                      │
└────────────────────────────────┼──────────────────────────────────────┘
                                 │
┌────────────────────────────────┼──────────────────────────────────────┐
│                        数据存储 (SQLite)                               │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐        │
│  │ assessment │ │   advice   │ │  evidence  │ │ rule_defs  │        │
│  │ 评估记录    │ │  建议记录   │ │ 依据+审计   │ │  规则定义   │        │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘        │
│  ┌────────────┐ ┌────────────┐                                       │
│  │ event_log  │ │ llm_trace  │                                       │
│  │  事件日志   │ │ LLM推理追溯│                                       │
│  └────────────┘ └────────────┘                                       │
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
[2] POST /api/assessments {symptoms_raw, severity, duration, patient_id}
       │
       ▼
[3] LLM 推理引擎 — 第一阶段：语义理解
       │  Prompt: "你是肿瘤科症状评估专家。从以下患者描述中提取结构化症状实体..."
       │  输入: "胸口像压了块大石头，喘气费劲，已经两天了"
       │  输出: {entities: [{symptom:"胸闷", body_part:"胸部", severity:8},
       │                    {symptom:"呼吸困难", severity:7}],
       │          associations: ["胸闷+呼吸困难 → 需排查心源性/肺源性"]}
       │
       ▼
[4] LLM 推理引擎 — 第二阶段：多证据链推理
       │  Prompt: "基于以下结构化症状+患者病史，进行风险判定并给出推理链..."
       │  上下文: 症状实体 + 历史评估 + 当前用药
       │  输出: {risk_level:"high",
       │          reasoning_chain:["化疗后白细胞低+发热39°C→粒缺伴发热",
       │                           "粒缺伴发热为肿瘤急症→需4h内启动抗生素"],
       │          confidence:0.92}
       │
       ▼
[5] 规则安全网校验 (Guardrail Check)
       │  ┌──────────────────────────────────────────┐
       │  │ 高危关键词扫描：呼吸困难、胸痛、意识模糊...  │
       │  │ → 若 LLM 判定为"中/低风险"但命中高危词    │
       │  │ → 强制升级为"高风险"并记录 guardrail_trigger│
       │  │                                          │
       │  │ 结构化信息注入：就医地址、急诊电话等        │
       │  └──────────────────────────────────────────┘
       │
       ▼
[6] LLM 推理引擎 — 第三阶段：个性化建议生成
       │  Prompt: "基于风险判定+推理链+患者上下文，生成共情建议..."
       │  输出: {advice_text:"...", contact_team:true,
       │          tone:"urgent_but_calming"}
       │
       ▼
[7] 持久化：写入 assessment, advice, evidence, llm_trace, event_log
       │
       ▼
[8] 触发 assessment_submitted 事件
       │
       ▼
[9] 返回结果给前端 {risk_level, advice, contact_team, audit}
       │
       ▼
[10] 用户查看结果页面 → 触发 result_viewed 事件
       │
       ├─→ 用户点击"联系团队" → POST /api/assessments/{id}/contact
       │         └─→ 触发 contact_team_clicked 事件
       │
       └─→ 用户关闭/返回 → 触发 assessment_closed 事件
```

---

## 2. "感知-决策-执行-学习" 闭环设计

### 2.1 感知层 (Perception)

核心转变：从"关键词匹配"升级为 **LLM 语义理解 + 关键词兜底**。

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **症状文本采集** | 前端表单：自由文本（主体）+ 结构化选择题（严重程度 1-10、持续时间等辅助字段） | HTML5 + vanilla JS |
| **语义理解（主）** | LLM 结构化提取：将非标准表达、方言、比喻描述转为结构化症状实体列表，识别症状之间的修饰关系和关联 | OpenAI-compatible API，专用 symptom-extraction prompt |
| **关键词匹配（兜底）** | jieba 分词 + 高危关键词表：当 LLM 调用失败或超时时降级为关键词匹配 | jieba 分词 |
| **上下文聚合** | 合并当前输入 + 患者历史评估记录 + 当前用药信息，组装为 LLM 推理上下文 | 服务层 ContextBuilder |
| **输入校验** | Pydantic 校验 + LLM 判断输入是否有效症状描述（过滤无效/恶意输入） | Pydantic + LLM 分类 |

**感知层数据产出**（LLM 结构化后）：
```json
{
  "symptoms_raw": "胸口像压了块大石头，喘气喘不上来，发烧到39度已经两天了",
  "symptoms_structured": [
    {
      "entity": "胸闷",
      "body_part": "胸部",
      "severity": 8,
      "modifier": "压迫感",
      "duration_hours": 48
    },
    {
      "entity": "呼吸困难",
      "severity": 7,
      "duration_hours": 48
    },
    {
      "entity": "发热",
      "temperature": 39.0,
      "duration_hours": 48
    }
  ],
  "associations": [
    "胸闷 + 呼吸困难 → 需排查心源性/肺源性",
    "化疗后 + 发热39°C + 48h → 粒缺伴发热高风险"
  ],
  "patient_context": {
    "history": ["2026-04-15 评估：化疗后轻度恶心，低风险"],
    "current_medications": ["多西他赛", "环磷酰胺"],
    "last_chemo_date": "2026-04-28"
  }
}
```

### 2.2 决策层 (Decision)

核心转变：从"单规则命中"升级为 **LLM 多证据链推理 + 规则安全网强制校验**。

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **LLM 推理（主）** | LLM 综合症状实体 + 病史 + 用药做联合推理，输出风险等级 + 完整推理链 + 置信度 | OpenAI-compatible API，专用 risk-assessment prompt（含医学知识上下文） |
| **规则安全网（Guardrail）** | 高危关键词表强制扫描 LLM 输出：若 LLM 判定"中/低风险"但命中高危词（呼吸困难、胸痛、意识模糊等），强制升级为高风险并记录 | Python 规则匹配器，规则 YAML 配置文件，版本化管理 |
| **风险分层** | 三级：高风险（立即线下就医/24h 联系团队）、中风险（联系团队或密切观察）、低风险（继续观察与记录） | LLM 判定 → 规则安全网覆盖 → 最终风险等级 |
| **推理可解释性** | LLM 必须输出 `reasoning_chain`：逐步推理过程，每条引用具体症状或医学知识 | Prompt 约束 + JSON Schema 强制返回 |
| **置信度评估** | LLM 输出 0-1 置信度分 + 规则覆盖标志，低置信度 + 无规则命中 → 建议人工审核 | LLM 自评分 + 规则匹配状态联合判断 |
| **建议生成（主）** | LLM 结合风险等级 + 患者上下文，生成个性化建议，兼顾医学准确性和共情表达 | 专用 advice-generation prompt |
| **结构化信息注入（兜底）** | 规则安全网在 LLM 建议生成后，强制注入关键就医指引（医院地址、急诊电话等结构化信息） | 规则模板注入 |

**LLM 推理链输出示例**：
```json
{
  "risk_level": "high",
  "confidence": 0.92,
  "reasoning_chain": [
    "患者正在接受多西他赛+环磷酰胺化疗（末次 2026-04-28），处于骨髓抑制风险窗口期",
    "发热 39°C 持续 48h + 化疗后 7 天 → 高度符合粒缺伴发热（FN）临床特征",
    "胸闷+呼吸困难组合 → 需排除化疗药物相关心脏毒性或肺栓塞",
    "综合判断：粒缺伴发热为肿瘤治疗相关急症，需 4h 内启动抗生素治疗"
  ],
  "contact_team": true,
  "advice": "根据您的描述——化疗后一周出现高烧、胸闷和呼吸困难——这属于需要**立即处理**的情况...",
  "guardrail_triggered": true,
  "guardrail_detail": {
    "rule_id": "R001",
    "rule_name": "呼吸困难高危拦截",
    "reason": "LLM 已判定高风险，规则安全网确认一致，未做升级"
  }
}
```

### 2.3 执行层 (Execution)

核心转变：从"模板填空"升级为 **LLM 个性化生成 + 规则强制结构化兜底**。

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **结果展示** | 前端结果页渲染：风险等级（颜色标识 + 动画警示）、推理链（可折叠）、共情建议文本、联系按钮 | 前端组件 |
| **个性化建议** | LLM 根据患者病史、用药阶段、症状组合生成差异化建议，语言风格根据风险等级调整（紧急 / 安抚 / 鼓励） | LLM advice-generation prompt |
| **协同请求** | POST /assessments/{id}/contact → 创建协同工单，附带评估摘要和 LLM 推理链 | API + 数据库写入 |
| **事件发布** | 每个关键动作写入 event_log + 同时记录 llm_trace（token 用量、延迟、模型版本） | Python logging + event_log 表 + llm_trace 表 |
| **错误处理** | LLM 调用失败 → 降级为纯规则引擎；LLM 输出格式异常 → JSON Schema 校验 + 重试；全部失败 → 返回保守建议"建议联系团队" | FastAPI exception handlers + 降级策略链 |
| **LLM 输出校验** | 对 LLM 返回的 JSON 做 Schema 校验，确保 risk_level 在枚举值内、contact_team 为布尔值等 | Pydantic 校验 LLM 输出 |

**执行链路追踪**：
```
request_id → llm_call_1 (症状提取) → llm_call_2 (风险评估) → guardrail_check
                                                             → llm_call_3 (建议生成)
→ assessment_id → event_log
→ llm_trace (三次 LLM 调用各自的 model, prompt_version, tokens, latency_ms)
```

### 2.4 学习层 (Learning)

核心转变：从"人工分析"升级为 **LLM 辅助洞察 + 规则半自动演进**。

| 维度 | 实现方式 | 技术选型 |
|------|---------|---------|
| **LLM 推理质量分析** | 统计 LLM 置信度分布、规则升级率（guardrail 触发频率）、LLM 与规则判定一致率 | SQL 聚合查询 + llm_trace 分析 |
| **未命中案例分析** | 定期用 LLM 分析 event_log 中的"规则升级"案例（LLM 判低/中但规则强制升级的），自动归纳为候选新规则 | LLM 批量分析 + 人工确认 |
| **规则半自动演进** | LLM 从升级案例中提取共性模式 → 生成候选规则 → 人工审核 → 入库，版本号 +1 | LLM 规则生成 prompt + 人工审核工作流 |
| **反馈收集** | 前端"这个建议有帮助吗？👍👎"按钮 + 可选文字反馈，写入 event_log | 前端按钮 + API |
| **误判回溯** | 通过 event_log + llm_trace 串联完整推理路径，定位是语义提取错误 / 推理偏差 / 建议不当 | 联合查询 + 可视化还原 |
| **医学知识同步** | 定期输入最新指南摘要，LLM 对比现有规则库，标记需新增/修改/废弃的规则 | LLM 知识更新 prompt |
| **趋势检测** | LLM 分析评估数据中的症状集群变化（如某种新药的不良反应模式聚集），预警潜在安全信号 | LLM 趋势分析 + 统计校验 |

**学习闭环可视化**：
```
用户评估 → event_log + llm_trace 积累
                │
                ▼
        LLM 分析升级案例 + 低置信度案例 + 负面反馈
                │
                ▼
        LLM 生成候选规则 / 现有规则修改建议
                │
                ▼
        人工审核确认 → 规则库更新，版本号 +1
                │
                ▼
        LLM prompt 同步更新（补充新发现的症状模式）
                │
                ▼
        新规则 + 新 prompt 上线 → 新一轮评估收集
```

---

## 3. 技术栈总览

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端 | HTML + vanilla JS + Tailwind CSS | 最简可运行，无需构建工具 |
| 后端 | Python 3.11+ + FastAPI | 异步支持、自动文档、数据校验 |
| LLM 推理 | OpenAI-compatible API（可对接 GPT-4/Claude/国产模型） | 三个独立 prompt：症状提取 / 风险评估 / 建议生成 |
| Prompt 管理 | YAML 文件 + 版本号 | 每个 prompt 独立版本化，可追溯 |
| LLM 可观测 | llm_trace 表（model, prompt_version, tokens, latency_ms, response_snippet） | 每次 LLM 调用全记录 |
| 规则引擎 | 自定义 Python RuleEngine + rules.yaml | 规则作为安全网，版本化管理 |
| 分词（降级用） | jieba | LLM 不可用时的关键词兜底 |
| 数据库 | SQLite + SQLAlchemy | 零配置、单文件、ORM 支持 |
| 结构化日志 | Python logging (JSON格式) + event_log 表 | 双通道日志 |
| 部署 | 单进程 FastAPI + uvicorn | MVP 阶段单机部署 |

---

## 4. 数据库模型

### 6 张核心表（新增 llm_trace）

| 表名 | 核心字段 | 说明 |
|------|---------|------|
| `assessments` | id, patient_id, symptoms_raw, symptoms_structured(JSON), severity, duration_hours, risk_level, confidence, created_at | 评估主记录 |
| `advice` | id, assessment_id, advice_text, reasoning_chain(JSON), contact_team(bool), tone, generated_at | 建议记录，含 LLM 推理链 |
| `evidence` | id, assessment_id, primary_engine(LLM\|Rule), llm_risk_level, llm_confidence, guardrail_triggered(bool), guardrail_rule_id, guardrail_rule_version, guardrail_action(confirm\|upgrade\|inject), final_risk_level, generated_at | 审计记录：记录 LLM 原始判定 + 规则安全网动作 + 最终结果 |
| `llm_trace` | id, assessment_id, call_stage(symptom_extraction\|risk_assessment\|advice_generation), model_name, model_version, prompt_name, prompt_version, input_tokens, output_tokens, latency_ms, request_json(JSON), response_json(JSON), created_at | LLM 调用全链路追溯 |
| `rule_definitions` | id, rule_id, rule_name, version, priority, keywords(JSON), risk_level, guardrail_action, structured_info_template, is_active, created_at | 规则定义表（安全网规则） |
| `event_log` | id, assessment_id, event_type, event_data(JSON), client_timestamp, server_timestamp, session_id | 事件日志，5 种事件类型 |

### 5 个埋点事件

```
assessment_started     → 用户打开输入页面
assessment_submitted   → 用户提交评估表单（含 LLM 推理完成）
result_viewed          → 用户查看评估结果
contact_team_clicked   → 用户点击"联系团队"按钮
assessment_closed      → 用户关闭结果页 / 返回列表
```

---

## 5. 审计要求实现

每次评估结果通过 `evidence` 表实现完整双向追溯：

| 审计维度 | 字段 | 说明 |
|---------|------|------|
| **LLM 判了什么** | `llm_risk_level`, `llm_confidence` | LLM 原始判定结果和置信度 |
| **安全网做了什么** | `guardrail_triggered`, `guardrail_rule_id`, `guardrail_action` | 哪条规则被触发，做了确认/升级/注入 |
| **最终结果** | `final_risk_level` | 经过规则安全网修正后的最终风险等级 |
| **推理过程** | `advice.reasoning_chain` (JSON) | LLM 完整推理步骤，每步可独立审视 |
| **规则版本** | `guardrail_rule_version` | 触发时的规则版本号 |
| **LLM 版本** | `llm_trace.model_name`, `llm_trace.model_version`, `llm_trace.prompt_version` | 使用了哪个模型和 prompt 版本 |
| **生成时间** | `advice.generated_at`, `evidence.generated_at`, `llm_trace.created_at` | 各环节精确时间戳 |

API 返回给前端的 `audit` 字段：
```json
{
  "risk_level": "high",
  "advice": "根据您的描述——化疗后一周出现高烧39°C、胸闷和呼吸困难...",
  "contact_team": true,
  "reasoning_chain": [
    "患者正在接受化疗（末次2026-04-28），处于骨髓抑制风险窗口期",
    "发热39°C持续48h + 化疗后7天 → 高度符合粒缺伴发热（FN）",
    "胸闷+呼吸困难组合 → 需排除心脏毒性或肺栓塞",
    "综合判断：粒缺伴发热为肿瘤急症，需4h内启动抗生素"
  ],
  "audit": {
    "primary_engine": "LLM",
    "llm_model": "claude-opus-4-6",
    "llm_confidence": 0.92,
    "guardrail_triggered": true,
    "guardrail_rule_id": "R001",
    "guardrail_rule_name": "呼吸困难高危拦截",
    "guardrail_action": "confirm",
    "guardrail_rule_version": "1.0.0",
    "prompt_version": "risk-assessment-v1",
    "generated_at": "2026-05-05T12:00:00Z"
  }
}
```

---

## 6. 规则安全网定义

规则从"主引擎"降级为"安全兜底"，只做两件事：**高危拦截** 和 **结构化信息注入**。

```yaml
# rules.yaml — 安全网规则（LLM 输出后执行）
rules:
  - rule_id: "G001"
    rule_name: "呼吸困难高危拦截"
    version: "1.0.0"
    priority: 1
    trigger_keywords: ["呼吸困难", "喘不过气", "气短", "窒息", "胸口压大石"]
    action: "force_high_risk"        # 若 LLM 判中/低，强制升级
    structured_info: |
      如出现呼吸困难，请立即拨打 120 或前往最近的急诊科。
      就诊时请携带您的化疗方案记录和当前用药清单。

  - rule_id: "G002"
    rule_name: "胸痛高危拦截"
    version: "1.0.0"
    priority: 2
    trigger_keywords: ["胸痛", "心口痛", "胸口疼", "心慌"]
    action: "force_high_risk"
    structured_info: |
      胸痛可能提示心脏问题或肺栓塞，需在急诊科进行心电图和血液检查。

  - rule_id: "G003"
    rule_name: "意识改变高危拦截"
    version: "1.0.0"
    priority: 3
    trigger_keywords: ["意识模糊", "晕倒", "昏迷", "叫不醒", "胡言乱语"]
    action: "force_high_risk"
    structured_info: |
      意识改变属于急危重症信号，请立即拨打 120，不要自行驾车前往医院。
```

---

## 7. Prompt 管理

三个核心 prompt，独立版本化，与 LLM trace 关联。

| Prompt 名称 | 版本 | 功能 | 关键约束 |
|------------|------|------|---------|
| `symptom-extraction-v1` | 1.0.0 | 症状语义提取 + 实体结构化 | 必须返回 JSON，必须识别修饰关系和症状关联 |
| `risk-assessment-v1` | 1.0.0 | 多证据链推理 + 风险判定 | 必须返回 reasoning_chain，risk_level 只能是 high/medium/low |
| `advice-generation-v1` | 1.0.0 | 个性化建议 + 共情表达 | 根据风险等级调整语气，高风险需包含紧急就医指引 |

prompt 文件存储：
```
HealthHub/prompts/
├── symptom-extraction-v1.yaml
├── risk-assessment-v1.yaml
└── advice-generation-v1.yaml
```

每个 prompt 文件含：
```yaml
name: risk-assessment
version: "1.0.0"
model: claude-opus-4-6
system_prompt: |
  你是肿瘤科症状评估专家...
user_prompt_template: |
  基于以下结构化症状和患者上下文进行风险评估...
output_schema:
  type: object
  properties:
    risk_level: {enum: [high, medium, low]}
    confidence: {type: number, minimum: 0, maximum: 1}
    reasoning_chain: {type: array, items: {type: string}}
    contact_team: {type: boolean}
```

---

## 8. 项目文件结构

```
HealthHub/
├── main.py                     # FastAPI 入口，路由注册
├── models.py                   # SQLAlchemy 模型定义（含 llm_trace）
├── schemas.py                  # Pydantic 请求/响应模型
├── config.py                   # 配置：LLM API key/endpoint、DB路径
├── prompts/
│   ├── symptom-extraction-v1.yaml
│   ├── risk-assessment-v1.yaml
│   └── advice-generation-v1.yaml
├── llm/
│   ├── __init__.py
│   ├── client.py               # LLM API 调用封装（统一接口）
│   ├── prompt_loader.py        # Prompt 加载与版本管理
│   └── trace.py                # LLM 调用追踪写入
├── rule_engine/
│   ├── __init__.py
│   ├── guardrails.py           # 规则安全网：高危拦截 + 结构化注入
│   ├── rules.yaml              # 安全网规则定义
│   └── matcher.py              # 关键词匹配器（jieba，LLM 降级用）
├── services/
│   ├── __init__.py
│   ├── assessment.py           # 评估业务逻辑：编排 LLM 调用 + 安全网校验
│   └── event.py                # 事件埋点服务
├── static/
│   ├── index.html              # 输入页
│   ├── result.html             # 结果页（含推理链展示）
│   └── history.html            # 历史记录页
├── requirements.txt            # Python 依赖
└── README.md                   # 运行说明
```

---

## 9. 验证方案

1. **启动验证**：`uvicorn main:app --reload`，访问 `http://localhost:8000` 确认三页面可访问
2. **接口验证**：通过 Swagger UI (`/docs`) 测试 4 个 API 端点
3. **LLM 推理验证**：
   - 输入典型高风险描述："化疗后一周，发烧39度不退，胸口闷喘不上气" → 期望 LLM 返回高风险 + 推理链含"粒缺伴发热"
   - 输入中风险描述："化疗后有点恶心，吃不下饭，持续3天" → 期望 LLM 返回中风险
   - 输入低风险描述："手臂有点酸，打针的地方有点红" → 期望 LLM 返回低风险
4. **规则安全网验证**：
   - 模拟 LLM 返回低风险但输入含"呼吸困难" → 验证 guardrail 强制升级为高风险
   - 检查 evidence 表 `guardrail_action = "upgrade"`
5. **LLM 降级验证**：断开 LLM API → 验证系统降级为关键词规则匹配，仍能正常返回结果
6. **事件验证**：执行完整流程后查询 event_log 表，确认 5 个事件均已写入
7. **审计验证**：
   - 查询 `evidence` 表确认 llm_risk_level、guardrail_action、final_risk_level 正确记录
   - 查询 `llm_trace` 表确认 3 次 LLM 调用（症状提取/风险评估/建议生成）分别记录
   - 确认 audit 字段包含 prompt_version + model_version + rule_version + generated_at
8. **端到端验证**：
   - 打开页面 → event_log 有 `assessment_started`
   - 输入"我发烧39度3天了，感觉喘不过气" → llm_trace 有 3 条记录 → 返回高风险 + 推理链
   - 查看结果 → event_log 有 `assessment_submitted` + `result_viewed`
   - 点击"联系团队" → event_log 有 `contact_team_clicked`
   - 返回列表 → event_log 有 `assessment_closed`
