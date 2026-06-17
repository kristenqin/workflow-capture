# 02_Workflow_Object_Model

## 1. 文档目的

本文档定义 `workflow-capture` 的轻量对象模型。

经过产品边界调整后，本项目不再以自建 Workflow Runtime 为目标，因此对象模型不应该模拟执行引擎，而应该服务于三个目标：

```text
捕获流程意图
形成中间表示
适配外部工具
```

一句话定义：

> Workflow Object Model 是 GPT / Markdown 流程意图进入系统后的轻量 Workflow IR 对象层。

---

## 2. 建模原则

### 2.1 IR 优先，而非 Runtime 优先

对象模型不描述任务如何被本系统执行，而是描述任务如何被理解、审查和导出。

因此不要在 MVP 对象里放入过多运行时字段，例如：

- running
- blocked
- retryCount
- worker
- executionLog
- credential
- schedule

这些是外部执行工具或后续运行时系统关心的内容，不属于 Capture 层核心对象。

### 2.2 节点是可映射任务，不是运行时任务

TaskNode 表示一个可以被人类理解、编辑、映射的任务单元。

它不等于真正运行中的 task。

### 2.3 关系只表达流程结构

Relation 只表达节点之间的逻辑关系，例如顺序、依赖、条件、并行、反馈。

它不负责表达执行状态、调度策略或重试逻辑。

### 2.4 自动化判断只是 Hint

系统可以判断一个节点可能适合自动化，但这只是提示，不代表本系统会执行。

自动化执行交给外部工具。

### 2.5 ExportTarget 是一等对象

既然产品定位是适配外部工具，那么对象模型必须显式表达“这个流程适合导出到哪里”。

---

## 3. 核心对象总览

MVP 阶段只保留 5 个核心对象：

```text
WorkflowIR
├── Source
├── TaskNode[]
├── Relation[]
├── Artifact[]
└── ExportTarget[]
```

| 对象 | 作用 |
|---|---|
| WorkflowIR | 流程意图的中间表示 |
| Source | 原始文本来源 |
| TaskNode | 可被理解、编辑、映射的任务节点 |
| Relation | 任务节点之间的关系 |
| Artifact | 流程输入或产出的材料 |
| ExportTarget | 可导出的目标工具或格式 |

Checkpoint、Risk、Input、Output 不再作为顶层复杂对象独立存在。

它们被压缩为 TaskNode 的轻量字段：

```text
acceptance
risks
inputs
outputs
```

这样可以避免对象模型过重。

---

## 4. WorkflowIR

WorkflowIR 是系统的核心中间表示。

它负责承接从自然语言抽取出的流程意图，并为渲染和导出提供统一数据结构。

### 4.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | IR 唯一标识 |
| title | string | 流程标题 |
| goal | string | 流程目标 |
| source | Source | 原始来源 |
| nodes | TaskNode[] | 任务节点 |
| relations | Relation[] | 节点关系 |
| artifacts | Artifact[] | 输入或产出材料 |
| exportTargets | ExportTarget[] | 建议导出目标 |
| metadata | Metadata | 元信息 |

### 4.2 设计约束

- 一个 WorkflowIR 必须至少有一个 TaskNode
- 一个 WorkflowIR 必须有明确 goal
- TaskNode id 必须在当前 IR 内唯一
- Relation 的 from / to 必须引用已存在 TaskNode
- ExportTarget 只表达适配方向，不保证可直接运行

---

## 5. Source

Source 表示 IR 从哪里被抽取出来。

### 5.1 来源类型

| 类型 | 说明 |
|---|---|
| chat_message | 单条 GPT 回复 |
| chat_thread | 一段对话上下文 |
| markdown | Markdown 文档 |
| document | 普通文档 |
| agent_response | Agent 输出 |
| manual | 用户手动创建 |

### 5.2 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| type | SourceType | 来源类型 |
| rawText | string | 原始文本 |
| title | string | 来源标题，可选 |
| url | string | 可选来源链接 |
| capturedAt | string | 捕获时间 |

---

## 6. TaskNode

TaskNode 是 Workflow IR 的核心节点。

它表示一个自然语言流程中的任务单元。

### 6.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | 节点唯一标识 |
| title | string | 节点标题 |
| description | string | 节点说明 |
| kind | NodeKind | 节点类型提示 |
| inputs | string[] | 输入材料或前置条件 |
| actions | string[] | 节点内部动作 |
| outputs | string[] | 节点产出 |
| acceptance | string[] | 验收或检查条件 |
| automation | AutomationHint | 自动化提示 |
| risks | string[] | 风险或注意事项 |
| sourceText | string | 对应原文证据 |
| confidence | number | 抽取置信度 |

### 6.2 NodeKind

NodeKind 用于提示这个节点大致适合映射为什么类型的外部任务。

| 类型 | 含义 | 可能适配方向 |
|---|---|---|
| manual | 需要人工处理 | 人工审查 / 文档任务 |
| llm | LLM 可处理任务 | Agent Task Pack / LangGraph |
| http | HTTP/API 请求 | n8n / Make / Zapier |
| script | 脚本执行 | GitHub Actions / Temporal |
| github | GitHub 相关操作 | GitHub Actions / GitHub API |
| file | 文件读写或生成 | GitHub Actions / Agent Task |
| decision | 条件判断 | n8n IF / GitHub Actions if / LangGraph branch |
| review | 审查或验收 | 人工 gate / PR review / Agent review |
| unknown | 无法判断 | 需要人工修正 |

### 6.3 AutomationHint

AutomationHint 表示自动化可能性。

| 值 | 含义 |
|---|---|
| manual | 当前更适合人工处理 |
| automatable | 有明显自动化可能 |
| external | 应交给外部工具执行 |
| unknown | 无法判断 |

注意：

> automatable 不等于本系统自动执行。

它只表示这个节点后续可能被导出到外部工具。

---

## 7. Relation

Relation 表示 TaskNode 之间的关系。

### 7.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | 关系唯一标识 |
| from | string | 起始 TaskNode id |
| to | string | 目标 TaskNode id |
| type | RelationType | 关系类型 |
| condition | string | 条件表达，可选 |
| label | string | 关系说明，可选 |
| confidence | number | 抽取置信度 |

### 7.2 RelationType

| 类型 | 含义 |
|---|---|
| sequence | 顺序关系 |
| dependency | 依赖关系 |
| condition | 条件关系 |
| parallel | 并行关系 |
| feedback | 反馈回路 |

### 7.3 设计约束

- 默认关系为 sequence
- 只有文本中出现明确条件时，才标记 condition
- 只有文本中出现明显前置依赖时，才标记 dependency
- feedback 不应过度推断，必须有明显返工、审查、修正语义

---

## 8. Artifact

Artifact 表示流程中的输入材料或产出物。

Artifact 不负责存储文件本体，只描述这个流程涉及到什么材料。

### 8.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Artifact 唯一标识 |
| name | string | 名称 |
| type | ArtifactType | 类型 |
| role | ArtifactRole | 输入或输出 |
| producedBy | string | 由哪个 TaskNode 产生，可选 |
| consumedBy | string[] | 被哪些 TaskNode 消费，可选 |
| description | string | 说明，可选 |

### 8.2 ArtifactType

| 类型 | 含义 |
|---|---|
| markdown | Markdown 文档 |
| json | JSON 数据 |
| yaml | YAML 配置 |
| code | 代码文件 |
| issue | GitHub Issue |
| pr | Pull Request |
| file | 普通文件 |
| report | 报告 |
| unknown | 未知 |

### 8.3 ArtifactRole

| 类型 | 含义 |
|---|---|
| input | 输入 |
| output | 输出 |
| intermediate | 中间产物 |

---

## 9. ExportTarget

ExportTarget 表示当前 IR 适合导出到哪些格式或工具。

### 9.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| type | ExportTargetType | 目标类型 |
| status | ExportSupportStatus | 支持状态 |
| reason | string | 为什么适合或不适合 |
| warnings | string[] | 导出注意事项 |

### 9.2 ExportTargetType

| 类型 | 含义 |
|---|---|
| markdown | Markdown 文档 |
| json | 通用 JSON |
| agent_task_pack | Agent 任务包 |
| n8n | n8n workflow JSON |
| github_actions | GitHub Actions YAML |
| langgraph | LangGraph skeleton |
| temporal | Temporal TypeScript skeleton |

### 9.3 ExportSupportStatus

| 状态 | 含义 |
|---|---|
| supported | 当前版本支持 |
| partial | 可部分导出 |
| planned | 计划支持 |
| not_suitable | 不适合该目标 |

---

## 10. Metadata

Metadata 只记录解析和版本信息，不记录运行时状态。

建议字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| createdAt | string | 创建时间 |
| updatedAt | string | 更新时间 |
| extractorVersion | string | 抽取器版本 |
| schemaVersion | string | Schema 版本 |
| language | string | 主语言 |
| confidence | number | 总体置信度 |

---

## 11. 对象关系

```text
WorkflowIR owns TaskNode
WorkflowIR owns Relation
WorkflowIR owns Artifact
WorkflowIR owns ExportTarget
WorkflowIR owns Metadata

Relation references TaskNode
Artifact can reference TaskNode
ExportTarget reads TaskNode.kind / automation / relations / artifacts
```

---

## 12. 最小对象闭环

MVP 阶段最小闭环如下：

```text
Source.rawText
  ↓
WorkflowIR.goal
  ↓
TaskNode[]
  ↓
Relation[]
  ↓
AutomationHint
  ↓
ExportTarget[]
  ↓
Renderer / Exporter
```

只要这个闭环稳定，产品就可以进入第一版实现。

---

## 13. 和旧对象模型的差异

旧模型更接近自建 Workflow 系统：

```text
Workflow / WorkflowFragment / Stage / Edge / Input / Output / Checkpoint / Risk
```

新模型更接近适配层 IR：

```text
WorkflowIR / TaskNode / Relation / Artifact / ExportTarget
```

核心变化：

- Stage 改为 TaskNode
- Edge 改为 Relation
- Input / Output 压缩为 TaskNode 字段和 Artifact
- Checkpoint 压缩为 acceptance 字段
- Risk 压缩为 risks 字段
- 删除 StageStatus 这类运行时状态
- 新增 AutomationHint
- 新增 ExportTarget

---

## 14. 设计结论

Workflow Capture 的对象模型应该足够表达流程意图，但不能重到像一个执行平台。

最佳边界是：

> 用轻量 IR 承接 GPT 的隐式流程，用 Export Adapter 适配外部自动化工具。
