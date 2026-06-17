# 04_Workflow_Schema

## 1. 文档目的

本文档定义 `workflow-capture` 的第一版 Workflow Schema。

Schema 的作用是约束从 GPT 回复、Markdown 文档、Agent 输出中抽取出来的 Workflow 对象，确保它们可以被校验、渲染、编辑和导出。

一句话定义：

> Workflow Schema 是 Workflow Capture 的对象边界，也是 LLM 抽取结果进入系统前必须通过的结构契约。

---

## 2. 设计原则

### 2.1 字段名使用英文

虽然产品定义语言优先使用中文，但 Schema 字段名统一使用英文。

原因：

- 方便工程实现
- 方便 TypeScript / Zod / JSON Schema 使用
- 方便未来支持英文输入
- 避免多语言数据模型污染

### 2.2 内容语言保留原文

字段名使用英文，但字段内容可以是中文、英文或混合语言。

例如：

```json
{
  "title": "需求分析",
  "intent": "识别用户真实目标和任务边界"
}
```

### 2.3 MVP Schema 保持克制

第一版 Schema 不追求覆盖所有复杂编排场景。

优先支持：

- 线性流程
- 阶段卡片
- 基础关系
- 基础验收
- 基础导出

### 2.4 允许推断，但必须显式标记

如果某个字段不是从原文直接抽取，而是模型推断出来的，需要标记：

```json
"inferred": true
```

### 2.5 保留置信度

由 LLM 抽取或分类的关键字段，需要支持 `confidence`。

置信度范围：

```text
0 <= confidence <= 1
```

---

## 3. TypeScript 类型定义

以下是 MVP 阶段建议使用的核心类型。

```ts
export type SourceType =
  | "chat_message"
  | "chat_thread"
  | "markdown"
  | "document"
  | "agent_response"
  | "manual"

export type StageType =
  | "analysis"
  | "planning"
  | "design"
  | "implementation"
  | "review"
  | "testing"
  | "delivery"
  | "maintenance"
  | "unknown"

export type StageStatus =
  | "draft"
  | "ready"
  | "running"
  | "blocked"
  | "done"
  | "skipped"

export type OwnerType =
  | "user"
  | "gpt"
  | "agent"
  | "human"
  | "system"
  | "unknown"

export type EdgeType =
  | "sequence"
  | "dependency"
  | "condition"
  | "feedback"
  | "parallel"

export type Severity =
  | "info"
  | "warning"
  | "error"

export type CheckStatus =
  | "unchecked"
  | "passed"
  | "failed"
  | "skipped"
```

---

## 4. Workflow

```ts
export type Workflow = {
  id: string
  title: string
  description?: string
  source: Source
  goal: string
  stages: WorkflowStage[]
  edges: WorkflowEdge[]
  outputs?: WorkflowOutput[]
  checkpoints?: WorkflowCheckpoint[]
  risks?: WorkflowRisk[]
  metadata: WorkflowMetadata
}
```

### 4.1 必填字段

| 字段 | 是否必填 | 说明 |
|---|---|---|
| id | 是 | Workflow 唯一标识 |
| title | 是 | 标题 |
| source | 是 | 来源 |
| goal | 是 | 目标 |
| stages | 是 | 阶段列表 |
| edges | 是 | 关系列表 |
| metadata | 是 | 元信息 |

### 4.2 校验规则

- `id` 不可为空
- `title` 不可为空
- `goal` 不可为空
- `stages.length >= 1`
- `edges` 可以为空数组
- `metadata.schemaVersion` 必须存在
- 所有 Stage id 必须唯一
- 所有 Edge 的 `from` / `to` 必须指向已存在 Stage

---

## 5. WorkflowFragment

```ts
export type WorkflowFragment = {
  id: string
  title: string
  scope: string
  reusable: boolean
  stages: WorkflowStage[]
  edges: WorkflowEdge[]
  reuseRule?: ReuseRule
  source?: Source
  metadata: WorkflowMetadata
}
```

### 5.1 校验规则

- `id` 不可为空
- `title` 不可为空
- `scope` 不可为空
- `stages.length >= 1`
- `reusable` 默认为 `true`
- Fragment 可以没有完整 goal

---

## 6. Source

```ts
export type Source = {
  type: SourceType
  rawText: string
  title?: string
  url?: string
  capturedAt: string
}
```

### 6.1 校验规则

- `type` 必须是合法 SourceType
- `rawText` 不可为空
- `capturedAt` 使用 ISO string

---

## 7. WorkflowStage

```ts
export type WorkflowStage = {
  id: string
  title: string
  intent: string
  type: StageType
  inputs?: WorkflowInput[]
  actions: WorkflowAction[]
  outputs?: WorkflowOutput[]
  checkpoints?: WorkflowCheckpoint[]
  owner?: OwnerType
  status: StageStatus
  confidence?: number
  sourceSpan?: SourceSpan
}
```

### 7.1 必填字段

| 字段 | 是否必填 | 说明 |
|---|---|---|
| id | 是 | Stage 唯一标识 |
| title | 是 | 阶段标题 |
| intent | 是 | 阶段意图 |
| type | 是 | 阶段类型 |
| actions | 是 | 阶段动作 |
| status | 是 | 阶段状态 |

### 7.2 校验规则

- `id` 不可为空
- `title` 不可为空
- `intent` 不可为空
- `actions.length >= 1`
- `type` 必须是合法 StageType
- `status` 必须是合法 StageStatus
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 8. WorkflowEdge

```ts
export type WorkflowEdge = {
  id: string
  from: string
  to: string
  type: EdgeType
  label?: string
  condition?: string
  confidence?: number
  sourceSpan?: SourceSpan
}
```

### 8.1 校验规则

- `id` 不可为空
- `from` 必须指向已存在 Stage
- `to` 必须指向已存在 Stage
- `from !== to`
- `type` 必须是合法 EdgeType
- 当 `type === "condition"` 时，建议存在 `condition`
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 9. WorkflowInput

```ts
export type WorkflowInput = {
  id: string
  name: string
  description?: string
  required?: boolean
  inferred?: boolean
  sourceSpan?: SourceSpan
}
```

### 9.1 校验规则

- `id` 不可为空
- `name` 不可为空
- `required` 默认为 `false`
- `inferred` 默认为 `false`

---

## 10. WorkflowAction

```ts
export type WorkflowAction = {
  id: string
  description: string
  inferred?: boolean
  sourceSpan?: SourceSpan
}
```

### 10.1 校验规则

- `id` 不可为空
- `description` 不可为空
- `description` 建议使用动宾结构
- `inferred` 默认为 `false`

---

## 11. WorkflowOutput

```ts
export type WorkflowOutput = {
  id: string
  name: string
  description?: string
  format?: string
  inferred?: boolean
  sourceSpan?: SourceSpan
}
```

### 11.1 format 示例

```text
markdown
json
html
task_pack
report
component
unknown
```

### 11.2 校验规则

- `id` 不可为空
- `name` 不可为空
- `inferred` 默认为 `false`

---

## 12. WorkflowCheckpoint

```ts
export type WorkflowCheckpoint = {
  id: string
  stageId?: string
  description: string
  passCondition: string
  severity?: Severity
  status?: CheckStatus
  inferred?: boolean
  sourceSpan?: SourceSpan
}
```

### 12.1 校验规则

- `id` 不可为空
- `description` 不可为空
- `passCondition` 不可为空
- 如果 `stageId` 存在，必须指向已存在 Stage
- `severity` 默认为 `info`
- `status` 默认为 `unchecked`

---

## 13. WorkflowRisk

```ts
export type WorkflowRisk = {
  id: string
  description: string
  severity: Severity
  mitigation?: string
  sourceSpan?: SourceSpan
}
```

### 13.1 校验规则

- `id` 不可为空
- `description` 不可为空
- `severity` 必须是合法 Severity

---

## 14. SourceSpan

SourceSpan 用于保留对象对应的原文位置。

```ts
export type SourceSpan = {
  start?: number
  end?: number
  text?: string
}
```

### 14.1 设计说明

MVP 可以只保留 `text`，不强制计算字符位置。

后续如果需要更精确的原文高亮，再补充 `start` 和 `end`。

---

## 15. WorkflowMetadata

```ts
export type WorkflowMetadata = {
  createdAt: string
  updatedAt: string
  createdBy?: string
  extractorVersion: string
  schemaVersion: string
  language: "zh" | "en" | "mixed" | "unknown"
  confidence?: number
}
```

### 15.1 校验规则

- `createdAt` 使用 ISO string
- `updatedAt` 使用 ISO string
- `extractorVersion` 不可为空
- `schemaVersion` 不可为空
- `language` 必须是合法值
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 16. ReuseRule

ReuseRule 用于描述 Fragment 的复用方式。

```ts
export type ReuseRule = {
  suitableFor?: string[]
  notSuitableFor?: string[]
  requiredInputs?: string[]
  notes?: string
}
```

---

## 17. ValidationIssue

当 Workflow Draft 无法通过校验时，系统应该返回 ValidationIssue。

```ts
export type ValidationIssue = {
  path: string
  severity: Severity
  message: string
  suggestion?: string
}
```

示例：

```json
{
  "path": "stages[0].actions",
  "severity": "error",
  "message": "Stage 必须至少包含一个 action",
  "suggestion": "从原文中抽取该阶段的主要动作，或要求用户手动补充"
}
```

---

## 18. ExtractionReport

ExtractionReport 用于展示抽取质量。

```ts
export type ExtractionReport = {
  workflowId: string
  stageCount: number
  edgeCount: number
  checkpointCount: number
  riskCount: number
  confidence: number
  issues: ValidationIssue[]
  lowConfidenceFields?: string[]
}
```

---

## 19. Zod Schema 草案

MVP 实现建议使用 Zod。

```ts
import { z } from "zod"

export const SourceTypeSchema = z.enum([
  "chat_message",
  "chat_thread",
  "markdown",
  "document",
  "agent_response",
  "manual",
])

export const StageTypeSchema = z.enum([
  "analysis",
  "planning",
  "design",
  "implementation",
  "review",
  "testing",
  "delivery",
  "maintenance",
  "unknown",
])

export const StageStatusSchema = z.enum([
  "draft",
  "ready",
  "running",
  "blocked",
  "done",
  "skipped",
])

export const OwnerTypeSchema = z.enum([
  "user",
  "gpt",
  "agent",
  "human",
  "system",
  "unknown",
])

export const EdgeTypeSchema = z.enum([
  "sequence",
  "dependency",
  "condition",
  "feedback",
  "parallel",
])

export const SeveritySchema = z.enum(["info", "warning", "error"])
export const CheckStatusSchema = z.enum(["unchecked", "passed", "failed", "skipped"])

export const SourceSpanSchema = z.object({
  start: z.number().int().nonnegative().optional(),
  end: z.number().int().nonnegative().optional(),
  text: z.string().optional(),
})

export const WorkflowInputSchema = z.object({
  id: z.string().min(1),
  name: z.string().min(1),
  description: z.string().optional(),
  required: z.boolean().default(false),
  inferred: z.boolean().default(false),
  sourceSpan: SourceSpanSchema.optional(),
})

export const WorkflowActionSchema = z.object({
  id: z.string().min(1),
  description: z.string().min(1),
  inferred: z.boolean().default(false),
  sourceSpan: SourceSpanSchema.optional(),
})

export const WorkflowOutputSchema = z.object({
  id: z.string().min(1),
  name: z.string().min(1),
  description: z.string().optional(),
  format: z.string().optional(),
  inferred: z.boolean().default(false),
  sourceSpan: SourceSpanSchema.optional(),
})

export const WorkflowCheckpointSchema = z.object({
  id: z.string().min(1),
  stageId: z.string().optional(),
  description: z.string().min(1),
  passCondition: z.string().min(1),
  severity: SeveritySchema.default("info"),
  status: CheckStatusSchema.default("unchecked"),
  inferred: z.boolean().default(false),
  sourceSpan: SourceSpanSchema.optional(),
})

export const WorkflowRiskSchema = z.object({
  id: z.string().min(1),
  description: z.string().min(1),
  severity: SeveritySchema,
  mitigation: z.string().optional(),
  sourceSpan: SourceSpanSchema.optional(),
})

export const SourceSchema = z.object({
  type: SourceTypeSchema,
  rawText: z.string().min(1),
  title: z.string().optional(),
  url: z.string().url().optional(),
  capturedAt: z.string().min(1),
})

export const WorkflowStageSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
  intent: z.string().min(1),
  type: StageTypeSchema,
  inputs: z.array(WorkflowInputSchema).optional(),
  actions: z.array(WorkflowActionSchema).min(1),
  outputs: z.array(WorkflowOutputSchema).optional(),
  checkpoints: z.array(WorkflowCheckpointSchema).optional(),
  owner: OwnerTypeSchema.optional(),
  status: StageStatusSchema,
  confidence: z.number().min(0).max(1).optional(),
  sourceSpan: SourceSpanSchema.optional(),
})

export const WorkflowEdgeSchema = z.object({
  id: z.string().min(1),
  from: z.string().min(1),
  to: z.string().min(1),
  type: EdgeTypeSchema,
  label: z.string().optional(),
  condition: z.string().optional(),
  confidence: z.number().min(0).max(1).optional(),
  sourceSpan: SourceSpanSchema.optional(),
})

export const WorkflowMetadataSchema = z.object({
  createdAt: z.string().min(1),
  updatedAt: z.string().min(1),
  createdBy: z.string().optional(),
  extractorVersion: z.string().min(1),
  schemaVersion: z.string().min(1),
  language: z.enum(["zh", "en", "mixed", "unknown"]),
  confidence: z.number().min(0).max(1).optional(),
})

export const WorkflowSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
  description: z.string().optional(),
  source: SourceSchema,
  goal: z.string().min(1),
  stages: z.array(WorkflowStageSchema).min(1),
  edges: z.array(WorkflowEdgeSchema).default([]),
  outputs: z.array(WorkflowOutputSchema).optional(),
  checkpoints: z.array(WorkflowCheckpointSchema).optional(),
  risks: z.array(WorkflowRiskSchema).optional(),
  metadata: WorkflowMetadataSchema,
})
```

---

## 20. 跨对象校验规则

Zod 单字段校验不够，还需要跨对象校验。

### 20.1 Stage id 唯一

```text
所有 stages[*].id 必须唯一
```

### 20.2 Edge 引用合法

```text
edge.from 必须存在于 stage id 集合
edge.to 必须存在于 stage id 集合
edge.from !== edge.to
```

### 20.3 Checkpoint 引用合法

```text
checkpoint.stageId 如果存在，必须存在于 stage id 集合
```

### 20.4 条件 Edge 建议包含 condition

```text
当 edge.type = condition 时，condition 不强制必填，但缺失时生成 warning
```

### 20.5 空数组标准化

```text
edges 缺失时标准化为空数组
inputs / outputs / checkpoints 缺失时可以保持 undefined，也可以在 Normalize 阶段转为空数组
```

---

## 21. 示例 Workflow JSON

```json
{
  "id": "workflow_capture_mvp",
  "title": "Workflow Capture MVP 定义流程",
  "description": "从 GPT 回复中抽取隐式 Workflow，并渲染为可编辑对象。",
  "source": {
    "type": "chat_message",
    "rawText": "先分析需求，然后设计对象模型，接着定义 Schema，最后实现渲染。",
    "capturedAt": "2026-06-17T00:00:00.000Z"
  },
  "goal": "将自然语言任务流程转换为结构化 Workflow 对象",
  "stages": [
    {
      "id": "stage_01_analysis",
      "title": "需求分析",
      "intent": "识别用户想要从 GPT 回复中沉淀流程资产的真实目标。",
      "type": "analysis",
      "actions": [
        {
          "id": "action_01",
          "description": "分析原始文本中的任务目标"
        }
      ],
      "outputs": [
        {
          "id": "output_01",
          "name": "需求理解结果",
          "format": "markdown",
          "inferred": true
        }
      ],
      "status": "draft",
      "confidence": 0.86
    },
    {
      "id": "stage_02_model",
      "title": "对象模型设计",
      "intent": "定义 Workflow、Stage、Edge、Checkpoint 等核心对象。",
      "type": "design",
      "actions": [
        {
          "id": "action_02",
          "description": "定义 Workflow 核心对象关系"
        }
      ],
      "status": "draft",
      "confidence": 0.9
    }
  ],
  "edges": [
    {
      "id": "edge_01",
      "from": "stage_01_analysis",
      "to": "stage_02_model",
      "type": "sequence",
      "confidence": 0.95
    }
  ],
  "checkpoints": [
    {
      "id": "checkpoint_01",
      "description": "检查 Workflow 是否至少包含一个 Stage 和一个明确目标。",
      "passCondition": "goal 不为空，stages.length >= 1",
      "severity": "info",
      "status": "unchecked",
      "inferred": true
    }
  ],
  "metadata": {
    "createdAt": "2026-06-17T00:00:00.000Z",
    "updatedAt": "2026-06-17T00:00:00.000Z",
    "extractorVersion": "0.1.0",
    "schemaVersion": "0.1.0",
    "language": "zh",
    "confidence": 0.88
  }
}
```

---

## 22. MVP Schema 版本

当前文档定义的 Schema 版本为：

```text
0.1.0
```

版本含义：

- `0`：未进入稳定 API
- `1`：第一版对象结构
- `0`：尚未进行兼容性修订

---

## 23. 后续扩展方向

未来可以扩展：

- `WorkflowRun`：执行实例
- `AgentAssignment`：Agent 分配
- `Artifact`：产物对象
- `VersionHistory`：版本记录
- `Permission`：权限控制
- `Comment`：人工批注
- `ExecutionLog`：执行日志
- `ToolCall`：工具调用记录

这些对象不进入 MVP Schema，避免第一版过重。
