# 04_Workflow_Schema

## 1. 文档目的

本文档定义 `workflow-capture` 的第一版 Workflow IR Schema。

Schema 的作用是约束从 GPT 回复、Markdown 文档、Agent 输出中抽取出来的 Workflow IR，确保它可以被校验、渲染、编辑，并通过 Export Adapter 导出到外部工具。

一句话定义：

> Workflow IR Schema 是 Workflow Capture 的中间表示契约，不是执行引擎契约。

---

## 2. 设计原则

### 2.1 字段名使用英文

产品定义语言优先使用中文，但 Schema 字段名统一使用英文。

原因：

- 方便工程实现
- 方便 TypeScript / Zod / JSON Schema 使用
- 方便未来支持英文输入
- 避免多语言数据模型污染

### 2.2 内容语言保留原文

字段名使用英文，但字段内容可以是中文、英文或混合语言。

### 2.3 不包含运行时状态

MVP Schema 不包含：

- running
- blocked
- done
- retryCount
- executionLog
- worker
- credential
- schedule

这些属于执行系统，不属于 Capture / IR 层。

### 2.4 自动化判断只是 Hint

`automation` 和 `suggestedTargets` 只表达适配建议，不代表本系统会执行。

### 2.5 保留置信度

由 LLM 抽取或分类的关键字段，需要支持 `confidence`。

置信度范围：

```text
0 <= confidence <= 1
```

---

## 3. TypeScript 类型定义

```ts
export type SourceType =
  | "chat_message"
  | "chat_thread"
  | "markdown"
  | "document"
  | "agent_response"
  | "manual"

export type NodeKind =
  | "manual"
  | "llm"
  | "http"
  | "script"
  | "github"
  | "file"
  | "decision"
  | "review"
  | "unknown"

export type AutomationHint =
  | "manual"
  | "automatable"
  | "external"
  | "unknown"

export type RelationType =
  | "sequence"
  | "dependency"
  | "condition"
  | "parallel"
  | "feedback"

export type ArtifactType =
  | "markdown"
  | "json"
  | "yaml"
  | "code"
  | "issue"
  | "pr"
  | "file"
  | "report"
  | "unknown"

export type ArtifactRole =
  | "input"
  | "output"
  | "intermediate"

export type ExportTargetType =
  | "markdown"
  | "json"
  | "agent_task_pack"
  | "n8n"
  | "github_actions"
  | "langgraph"
  | "temporal"

export type ExportSupportStatus =
  | "supported"
  | "partial"
  | "planned"
  | "not_suitable"

export type Severity =
  | "info"
  | "warning"
  | "error"
```

---

## 4. WorkflowIR

```ts
export type WorkflowIR = {
  id: string
  title: string
  goal: string
  source: Source
  nodes: TaskNode[]
  relations: Relation[]
  artifacts?: Artifact[]
  exportTargets: ExportTarget[]
  metadata: Metadata
}
```

### 4.1 校验规则

- `id` 不可为空
- `title` 不可为空
- `goal` 不可为空
- `nodes.length >= 1`
- `relations` 可以为空数组
- `exportTargets.length >= 1`
- `metadata.schemaVersion` 必须存在
- 所有 TaskNode id 必须唯一
- 所有 Relation 的 `from` / `to` 必须指向已存在 TaskNode

---

## 5. Source

```ts
export type Source = {
  type: SourceType
  rawText: string
  title?: string
  url?: string
  capturedAt: string
}
```

### 5.1 校验规则

- `type` 必须是合法 SourceType
- `rawText` 不可为空
- `capturedAt` 使用 ISO string

---

## 6. TaskNode

```ts
export type TaskNode = {
  id: string
  title: string
  description?: string
  kind: NodeKind
  inputs?: string[]
  actions: string[]
  outputs?: string[]
  acceptance?: string[]
  automation: AutomationHint
  suggestedTargets?: ExportTargetType[]
  risks?: string[]
  sourceText?: string
  confidence?: number
}
```

### 6.1 校验规则

- `id` 不可为空
- `title` 不可为空
- `kind` 必须是合法 NodeKind
- `actions.length >= 1`
- `automation` 必须是合法 AutomationHint
- `suggestedTargets` 如果存在，必须是合法 ExportTargetType
- `confidence` 如果存在，必须在 0 到 1 之间

### 6.2 设计说明

TaskNode 不是运行时任务。

它只表达：

- 这个任务节点是什么
- 它大概属于什么类型
- 它可能如何被自动化
- 它适合导出到哪些外部工具

---

## 7. Relation

```ts
export type Relation = {
  id: string
  from: string
  to: string
  type: RelationType
  condition?: string
  label?: string
  confidence?: number
}
```

### 7.1 校验规则

- `id` 不可为空
- `from` 必须指向已存在 TaskNode
- `to` 必须指向已存在 TaskNode
- `from !== to`
- `type` 必须是合法 RelationType
- 当 `type === "condition"` 时，建议存在 `condition`
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 8. Artifact

```ts
export type Artifact = {
  id: string
  name: string
  type: ArtifactType
  role: ArtifactRole
  producedBy?: string
  consumedBy?: string[]
  description?: string
}
```

### 8.1 校验规则

- `id` 不可为空
- `name` 不可为空
- `type` 必须是合法 ArtifactType
- `role` 必须是合法 ArtifactRole
- `producedBy` 如果存在，必须指向已存在 TaskNode
- `consumedBy` 如果存在，内部 id 必须指向已存在 TaskNode

---

## 9. ExportTarget

```ts
export type ExportTarget = {
  type: ExportTargetType
  status: ExportSupportStatus
  reason?: string
  warnings?: string[]
}
```

### 9.1 校验规则

- `type` 必须是合法 ExportTargetType
- `status` 必须是合法 ExportSupportStatus
- MVP 默认至少包含 `json` 和 `markdown`

### 9.2 MVP 默认支持状态

```ts
const MVP_EXPORT_TARGETS = [
  { type: "json", status: "supported" },
  { type: "markdown", status: "supported" },
  { type: "agent_task_pack", status: "partial" },
  { type: "n8n", status: "planned" },
  { type: "github_actions", status: "planned" },
  { type: "langgraph", status: "planned" },
  { type: "temporal", status: "planned" },
]
```

---

## 10. Metadata

```ts
export type Metadata = {
  createdAt: string
  updatedAt: string
  extractorVersion: string
  schemaVersion: string
  language: "zh" | "en" | "mixed" | "unknown"
  confidence?: number
}
```

### 10.1 校验规则

- `createdAt` 使用 ISO string
- `updatedAt` 使用 ISO string
- `extractorVersion` 不可为空
- `schemaVersion` 不可为空
- `language` 必须是合法值
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 11. ValidationIssue

当 Workflow IR Draft 无法通过校验时，系统应该返回 ValidationIssue。

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
  "path": "nodes[0].actions",
  "severity": "error",
  "message": "TaskNode 必须至少包含一个 action",
  "suggestion": "从原文中抽取该节点的主要动作，或要求用户手动补充"
}
```

---

## 12. ExtractionReport

ExtractionReport 用于展示抽取质量。

```ts
export type ExtractionReport = {
  workflowId: string
  nodeCount: number
  relationCount: number
  artifactCount: number
  exportTargets: ExportTarget[]
  confidence: number
  issues: ValidationIssue[]
  lowConfidenceFields?: string[]
}
```

---

## 13. Zod Schema 草案

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

export const NodeKindSchema = z.enum([
  "manual",
  "llm",
  "http",
  "script",
  "github",
  "file",
  "decision",
  "review",
  "unknown",
])

export const AutomationHintSchema = z.enum([
  "manual",
  "automatable",
  "external",
  "unknown",
])

export const RelationTypeSchema = z.enum([
  "sequence",
  "dependency",
  "condition",
  "parallel",
  "feedback",
])

export const ArtifactTypeSchema = z.enum([
  "markdown",
  "json",
  "yaml",
  "code",
  "issue",
  "pr",
  "file",
  "report",
  "unknown",
])

export const ArtifactRoleSchema = z.enum([
  "input",
  "output",
  "intermediate",
])

export const ExportTargetTypeSchema = z.enum([
  "markdown",
  "json",
  "agent_task_pack",
  "n8n",
  "github_actions",
  "langgraph",
  "temporal",
])

export const ExportSupportStatusSchema = z.enum([
  "supported",
  "partial",
  "planned",
  "not_suitable",
])

export const SeveritySchema = z.enum(["info", "warning", "error"])

export const SourceSchema = z.object({
  type: SourceTypeSchema,
  rawText: z.string().min(1),
  title: z.string().optional(),
  url: z.string().url().optional(),
  capturedAt: z.string().min(1),
})

export const TaskNodeSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
  description: z.string().optional(),
  kind: NodeKindSchema,
  inputs: z.array(z.string()).optional(),
  actions: z.array(z.string().min(1)).min(1),
  outputs: z.array(z.string()).optional(),
  acceptance: z.array(z.string()).optional(),
  automation: AutomationHintSchema,
  suggestedTargets: z.array(ExportTargetTypeSchema).optional(),
  risks: z.array(z.string()).optional(),
  sourceText: z.string().optional(),
  confidence: z.number().min(0).max(1).optional(),
})

export const RelationSchema = z.object({
  id: z.string().min(1),
  from: z.string().min(1),
  to: z.string().min(1),
  type: RelationTypeSchema,
  condition: z.string().optional(),
  label: z.string().optional(),
  confidence: z.number().min(0).max(1).optional(),
})

export const ArtifactSchema = z.object({
  id: z.string().min(1),
  name: z.string().min(1),
  type: ArtifactTypeSchema,
  role: ArtifactRoleSchema,
  producedBy: z.string().optional(),
  consumedBy: z.array(z.string()).optional(),
  description: z.string().optional(),
})

export const ExportTargetSchema = z.object({
  type: ExportTargetTypeSchema,
  status: ExportSupportStatusSchema,
  reason: z.string().optional(),
  warnings: z.array(z.string()).optional(),
})

export const MetadataSchema = z.object({
  createdAt: z.string().min(1),
  updatedAt: z.string().min(1),
  extractorVersion: z.string().min(1),
  schemaVersion: z.string().min(1),
  language: z.enum(["zh", "en", "mixed", "unknown"]),
  confidence: z.number().min(0).max(1).optional(),
})

export const WorkflowIRSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
  goal: z.string().min(1),
  source: SourceSchema,
  nodes: z.array(TaskNodeSchema).min(1),
  relations: z.array(RelationSchema).default([]),
  artifacts: z.array(ArtifactSchema).optional(),
  exportTargets: z.array(ExportTargetSchema).min(1),
  metadata: MetadataSchema,
})
```

---

## 14. 跨对象校验规则

Zod 单字段校验不够，还需要跨对象校验。

### 14.1 TaskNode id 唯一

```text
所有 nodes[*].id 必须唯一
```

### 14.2 Relation 引用合法

```text
relation.from 必须存在于 node id 集合
relation.to 必须存在于 node id 集合
relation.from !== relation.to
```

### 14.3 Artifact 引用合法

```text
artifact.producedBy 如果存在，必须存在于 node id 集合
artifact.consumedBy 如果存在，其中每个 id 都必须存在于 node id 集合
```

### 14.4 条件 Relation 建议包含 condition

```text
当 relation.type = condition 时，condition 不强制必填，但缺失时生成 warning
```

### 14.5 ExportTarget 默认补全

```text
如果 exportTargets 缺少 json / markdown，Normalize 阶段应补全
```

---

## 15. 示例 Workflow IR JSON

```json
{
  "id": "workflow_capture_ir_mvp",
  "title": "Workflow Capture IR 定义流程",
  "goal": "将 GPT 回复中的隐式流程转换为可导出的 Workflow IR",
  "source": {
    "type": "chat_message",
    "rawText": "先分析需求，然后设计 IR Schema，接着实现 JSON 导出，最后考虑 n8n 适配。",
    "capturedAt": "2026-06-17T00:00:00.000Z"
  },
  "nodes": [
    {
      "id": "node_01_analysis",
      "title": "分析需求",
      "kind": "manual",
      "actions": ["识别用户想从 GPT 回复中捕获流程意图的真实目标"],
      "outputs": ["产品意图判断"],
      "automation": "manual",
      "suggestedTargets": ["markdown", "agent_task_pack"],
      "sourceText": "先分析需求",
      "confidence": 0.9
    },
    {
      "id": "node_02_schema",
      "title": "设计 IR Schema",
      "kind": "llm",
      "actions": ["定义 WorkflowIR、TaskNode、Relation、Artifact、ExportTarget"],
      "outputs": ["Workflow IR Schema"],
      "acceptance": ["Schema 不包含运行时执行状态"],
      "automation": "automatable",
      "suggestedTargets": ["agent_task_pack", "json"],
      "sourceText": "然后设计 IR Schema",
      "confidence": 0.88
    },
    {
      "id": "node_03_export",
      "title": "实现 JSON 导出",
      "kind": "file",
      "actions": ["将 Workflow IR 序列化为通用 JSON"],
      "outputs": ["workflow-ir.json"],
      "automation": "automatable",
      "suggestedTargets": ["json"],
      "sourceText": "接着实现 JSON 导出",
      "confidence": 0.86
    }
  ],
  "relations": [
    {
      "id": "relation_01",
      "from": "node_01_analysis",
      "to": "node_02_schema",
      "type": "sequence",
      "confidence": 0.95
    },
    {
      "id": "relation_02",
      "from": "node_02_schema",
      "to": "node_03_export",
      "type": "sequence",
      "confidence": 0.95
    }
  ],
  "artifacts": [
    {
      "id": "artifact_01",
      "name": "Workflow IR Schema",
      "type": "json",
      "role": "intermediate",
      "producedBy": "node_02_schema"
    },
    {
      "id": "artifact_02",
      "name": "workflow-ir.json",
      "type": "json",
      "role": "output",
      "producedBy": "node_03_export"
    }
  ],
  "exportTargets": [
    {
      "type": "json",
      "status": "supported",
      "reason": "Workflow IR 原生支持 JSON 导出"
    },
    {
      "type": "markdown",
      "status": "supported",
      "reason": "可将节点和关系重新渲染为 Markdown 文档"
    },
    {
      "type": "n8n",
      "status": "planned",
      "reason": "后续可将 http、decision、llm 节点映射为 n8n 节点"
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

## 16. MVP Schema 版本

当前文档定义的 Schema 版本为：

```text
0.1.0
```

版本含义：

- `0`：未进入稳定 API
- `1`：第一版 IR 对象结构
- `0`：尚未进行兼容性修订

---

## 17. 后续扩展方向

未来可以扩展，但不进入 MVP：

- n8n Exporter Schema
- GitHub Actions Exporter Schema
- Agent Task Pack Schema
- LangGraph Skeleton Generator
- Temporal TypeScript Skeleton Generator
- Export Compatibility Report
- Node Mapping Rules
- Tool-specific Warning Rules

仍然不建议在本项目内扩展完整 Runtime。Workflow Capture 的核心边界应保持为：

```text
Capture → IR → Review → Export
```
