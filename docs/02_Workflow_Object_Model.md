# 02_Workflow_Object_Model

## 1. 文档目的

本文档定义 `workflow-capture` 的 Workflow Asset 对象模型。

当前阶段，本项目不以自动化平台适配为中心，也不以执行引擎为中心，而是以 workflow 资产沉淀和管理为中心。

一句话定义：

> Workflow Object Model 是把 GPT / Markdown 中的隐式流程沉淀为可管理 Workflow Asset 的对象层。

---

## 2. 建模原则

### 2.1 Asset 优先，而非 Runtime 优先

对象模型不描述 workflow 如何被本系统执行，而是描述 workflow 如何被保存、理解、检索、复用和迭代。

因此 MVP 不建模：

- running
- blocked
- retryCount
- worker
- executionLog
- credential
- schedule
- connector

这些属于执行平台，不属于 workflow 资产管理层。

### 2.2 Workflow 保持任务认知结构

workflow 的价值不只是步骤列表，而是它沉淀了：

- 任务目标
- 适用场景
- 输入条件
- 执行步骤
- 输出产物
- 验收方式
- 使用注意事项
- 可复用经验

这些内容需要被保留下来，而不是过早压缩成某个自动化平台的节点格式。

### 2.3 Step 是最小流程表达单位

WorkflowStep 表示 workflow 中的一个步骤。

它不是运行时任务，而是资产中的流程结构单元。

### 2.4 Metadata 服务管理和复用

Metadata 重点记录：

- 标签
- 分类
- 版本
- 来源
- 创建时间
- 更新时间
- 复用说明
- 置信度

而不是记录运行时状态。

### 2.5 Export 只是基础导出，不是平台适配

MVP 只需要支持 Markdown / JSON 这种基础导出，用于分发给人或 agent。

n8n、GitHub Actions、Temporal、LangGraph 等适配目标暂时不进入对象核心。

---

## 3. 核心对象总览

MVP 阶段核心对象：

```text
WorkflowAsset
├── Source
├── WorkflowStep[]
├── WorkflowArtifact[]
├── WorkflowRelation[]
├── WorkflowUsage
├── WorkflowMetadata
└── WorkflowVersion[]
```

| 对象 | 作用 |
|---|---|
| WorkflowAsset | 一套可管理、可复用的 workflow 资产 |
| Source | 原始文本来源 |
| WorkflowStep | workflow 内部步骤 |
| WorkflowRelation | 步骤之间的关系 |
| WorkflowArtifact | 输入、输出或中间产物 |
| WorkflowUsage | 适用场景、非适用场景、使用说明 |
| WorkflowMetadata | 管理信息、标签、分类、版本信息 |
| WorkflowVersion | workflow 的历史版本记录 |

---

## 4. WorkflowAsset

WorkflowAsset 是系统的核心对象。

它表示一套已经从自然语言中沉淀出来、可以被保存和复用的 workflow。

### 4.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Workflow Asset 唯一标识 |
| title | string | workflow 标题 |
| summary | string | 简短摘要 |
| goal | string | workflow 目标 |
| source | Source | 原始来源 |
| steps | WorkflowStep[] | 步骤列表 |
| relations | WorkflowRelation[] | 步骤关系 |
| artifacts | WorkflowArtifact[] | 输入、输出、中间产物 |
| usage | WorkflowUsage | 使用说明 |
| metadata | WorkflowMetadata | 管理信息 |
| versions | WorkflowVersion[] | 版本记录 |

### 4.2 设计约束

- 一个 WorkflowAsset 必须至少有一个 WorkflowStep
- 一个 WorkflowAsset 必须有明确 title 和 goal
- WorkflowStep id 必须在当前 asset 内唯一
- WorkflowRelation 的 from / to 必须引用已存在 step
- versions 可以为空，但 metadata.version 必须存在

---

## 5. Source

Source 表示 workflow 从哪里被捕获出来。

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

## 6. WorkflowStep

WorkflowStep 是 workflow 资产中的最小流程表达单位。

### 6.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | 步骤唯一标识 |
| title | string | 步骤标题 |
| purpose | string | 这个步骤为什么存在 |
| actions | string[] | 这个步骤要做什么 |
| inputs | string[] | 这个步骤需要什么 |
| outputs | string[] | 这个步骤产出什么 |
| acceptance | string[] | 如何判断这个步骤完成得合格 |
| notes | string[] | 使用注意事项 |
| sourceText | string | 对应原文证据 |
| confidence | number | 抽取置信度 |

### 6.2 设计说明

WorkflowStep 不应该被设计成某个自动化平台的节点。

它应该优先表达人类可理解的流程结构。

一个好的 WorkflowStep 应该能回答：

```text
为什么做？
做什么？
基于什么做？
做完得到什么？
怎么判断做得好？
```

---

## 7. WorkflowRelation

WorkflowRelation 表示步骤之间的关系。

### 7.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | 关系唯一标识 |
| from | string | 起始 step id |
| to | string | 目标 step id |
| type | RelationType | 关系类型 |
| condition | string | 条件表达，可选 |
| label | string | 关系说明，可选 |

### 7.2 RelationType

| 类型 | 含义 |
|---|---|
| sequence | 顺序关系 |
| dependency | 依赖关系 |
| condition | 条件关系 |
| parallel | 并行关系 |
| feedback | 反馈回路 |

---

## 8. WorkflowArtifact

WorkflowArtifact 表示 workflow 中涉及的输入、输出或中间产物。

### 8.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Artifact 唯一标识 |
| name | string | 名称 |
| type | ArtifactType | 类型 |
| role | ArtifactRole | 输入、输出或中间产物 |
| producedBy | string | 由哪个 step 产生，可选 |
| consumedBy | string[] | 被哪些 step 使用，可选 |
| description | string | 说明，可选 |

### 8.2 ArtifactType

| 类型 | 含义 |
|---|---|
| markdown | Markdown 文档 |
| json | JSON 数据 |
| yaml | YAML 配置 |
| code | 代码文件 |
| image | 图片或截图 |
| issue | Issue |
| pr | Pull Request |
| report | 报告 |
| note | 说明或备注 |
| unknown | 未知 |

### 8.3 ArtifactRole

| 类型 | 含义 |
|---|---|
| input | 输入 |
| output | 输出 |
| intermediate | 中间产物 |

---

## 9. WorkflowUsage

WorkflowUsage 描述这个 workflow 如何被复用。

这是资产管理中非常重要的对象。

### 9.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| suitableFor | string[] | 适合的任务场景 |
| notSuitableFor | string[] | 不适合的任务场景 |
| prerequisites | string[] | 使用前提 |
| reuseNotes | string[] | 复用说明 |
| examples | string[] | 使用示例 |

### 9.2 示例

```text
适合：设计稿视觉审查、Agent 输出页面审查
不适合：需要真实用户数据验证的产品决策
前提：必须有截图或页面预览
复用说明：每次使用前需要替换目标页面和验收重点
```

---

## 10. WorkflowMetadata

WorkflowMetadata 用于支持管理、搜索和版本化。

### 10.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| createdAt | string | 创建时间 |
| updatedAt | string | 更新时间 |
| version | string | 当前版本 |
| language | string | 主语言 |
| tags | string[] | 标签 |
| category | string | 分类 |
| status | AssetStatus | 资产状态 |
| confidence | number | 抽取置信度 |

### 10.2 AssetStatus

| 状态 | 含义 |
|---|---|
| draft | 草稿，尚未整理好 |
| reviewed | 已人工审查 |
| reusable | 可复用 |
| archived | 已归档 |

注意：AssetStatus 是资产管理状态，不是执行状态。

---

## 11. WorkflowVersion

WorkflowVersion 表示 workflow 资产的版本记录。

### 11.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| version | string | 版本号 |
| changedAt | string | 修改时间 |
| changeSummary | string | 修改摘要 |
| snapshot | object | 可选快照 |

MVP 可以先只记录 version 和 changeSummary，不必实现完整 diff。

---

## 12. 对象关系

```text
WorkflowAsset owns WorkflowStep
WorkflowAsset owns WorkflowRelation
WorkflowAsset owns WorkflowArtifact
WorkflowAsset owns WorkflowUsage
WorkflowAsset owns WorkflowMetadata
WorkflowAsset owns WorkflowVersion

WorkflowRelation references WorkflowStep
WorkflowArtifact can reference WorkflowStep
```

---

## 13. 最小对象闭环

MVP 阶段最小闭环如下：

```text
Source.rawText
  ↓
WorkflowAsset.title / goal
  ↓
WorkflowStep[]
  ↓
WorkflowRelation[]
  ↓
WorkflowUsage / Metadata
  ↓
Workflow Library
  ↓
Search / Reuse / Export
```

---

## 14. 和上一版 IR 模型的差异

上一版模型强调：

```text
WorkflowIR / TaskNode / Relation / Artifact / ExportTarget
```

新版模型强调：

```text
WorkflowAsset / WorkflowStep / WorkflowUsage / WorkflowMetadata / WorkflowVersion
```

核心变化：

- WorkflowIR 改为 WorkflowAsset
- TaskNode 改为 WorkflowStep
- ExportTarget 从核心对象中移除
- AutomationHint 从 MVP 核心中移除
- 增加 WorkflowUsage，用于表达复用条件
- 增加 WorkflowVersion，用于资产迭代
- Metadata 更关注标签、分类、资产状态

---

## 15. 设计结论

Workflow Capture 当前应该优先解决 workflow 资产沉淀与管理，而不是自动化执行或平台适配。

最佳边界是：

> 把 GPT 对话里隐式出现的 workflow 变成可命名、可保存、可检索、可复用、可迭代的资产。
