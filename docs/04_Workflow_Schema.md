# 04_Workflow_Schema

## 1. 文档目的

本文档定义 `workflow-capture` 的第一版 Workflow Asset Schema。

Schema 的作用是约束从 GPT 回复、Markdown 文档、Agent 输出中抽取出来的 Workflow Asset，确保它可以被校验、渲染、编辑、保存、检索和复用。

一句话定义：

> Workflow Asset Schema 是 workflow 资产管理的结构契约，不是自动化执行平台的运行契约。

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

### 2.3 不包含执行平台字段

MVP Schema 不包含：

- automation
- exportTarget
- running
- blocked
- retryCount
- executionLog
- worker
- credential
- schedule
- connector

这些属于未来执行或适配层，不属于当前 Workflow Asset 层。

### 2.4 资产管理信息是一等字段

当前 Schema 必须服务资产沉淀，因此要重视：

- tags
- category
- status
- version
- usage
- source
- confidence

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
  | "image"
  | "issue"
  | "pr"
  | "report"
  | "note"
  | "unknown"

export type ArtifactRole =
  | "input"
  | "output"
  | "intermediate"

export type AssetStatus =
  | "draft"
  | "reviewed"
  | "reusable"
  | "archived"

export type Severity =
  | "info"
  | "warning"
  | "error"
```

---

## 4. WorkflowAsset

```ts
export type WorkflowAsset = {
  id: string
  title: string
  summary?: string
  goal: string
  source: Source
  steps: WorkflowStep[]
  relations: WorkflowRelation[]
  artifacts?: WorkflowArtifact[]
  usage?: WorkflowUsage
  metadata: WorkflowMetadata
  versions?: WorkflowVersion[]
}
```

### 4.1 校验规则

- `id` 不可为空
- `title` 不可为空
- `goal` 不可为空
- `steps.length >= 1`
- `relations` 可以为空数组
- `metadata.version` 必须存在
- 所有 WorkflowStep id 必须唯一
- 所有 WorkflowRelation 的 `from` / `to` 必须指向已存在 WorkflowStep

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

## 6. WorkflowStep

```ts
export type WorkflowStep = {
  id: string
  title: string
  purpose?: string
  actions: string[]
  inputs?: string[]
  outputs?: string[]
  acceptance?: string[]
  notes?: string[]
  sourceText?: string
  confidence?: number
}
```

### 6.1 校验规则

- `id` 不可为空
- `title` 不可为空
- `actions.length >= 1`
- `confidence` 如果存在，必须在 0 到 1 之间

### 6.2 设计说明

WorkflowStep 是资产结构单元，不是自动化平台节点。

它应该优先表达：

- 这个步骤为什么存在
- 需要做什么动作
- 需要什么输入
- 产出什么结果
- 如何判断完成得合格

---

## 7. WorkflowRelation

```ts
export type WorkflowRelation = {
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
- `from` 必须指向已存在 WorkflowStep
- `to` 必须指向已存在 WorkflowStep
- `from !== to`
- `type` 必须是合法 RelationType
- 当 `type === "condition"` 时，建议存在 `condition`
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 8. WorkflowArtifact

```ts
export type WorkflowArtifact = {
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
- `producedBy` 如果存在，必须指向已存在 WorkflowStep
- `consumedBy` 如果存在，内部 id 必须指向已存在 WorkflowStep

---

## 9. WorkflowUsage

```ts
export type WorkflowUsage = {
  suitableFor?: string[]
  notSuitableFor?: string[]
  prerequisites?: string[]
  reuseNotes?: string[]
  examples?: string[]
}
```

### 9.1 校验规则

- 所有字段都是字符串数组
- 字段可以为空
- usage 缺失时，仍然允许保存 WorkflowAsset

### 9.2 设计说明

WorkflowUsage 对资产复用很关键。

它回答：

```text
什么时候适合用？
什么时候不适合用？
使用前需要什么？
复用时要注意什么？
有没有典型例子？
```

---

## 10. WorkflowMetadata

```ts
export type WorkflowMetadata = {
  createdAt: string
  updatedAt: string
  version: string
  language: "zh" | "en" | "mixed" | "unknown"
  tags: string[]
  category?: string
  status: AssetStatus
  confidence?: number
}
```

### 10.1 校验规则

- `createdAt` 使用 ISO string
- `updatedAt` 使用 ISO string
- `version` 不可为空
- `language` 必须是合法值
- `tags` 默认为空数组
- `status` 必须是合法 AssetStatus
- `confidence` 如果存在，必须在 0 到 1 之间

---

## 11. WorkflowVersion

```ts
export type WorkflowVersion = {
  version: string
  changedAt: string
  changeSummary: string
  snapshot?: unknown
}
```

### 11.1 校验规则

- `version` 不可为空
- `changedAt` 使用 ISO string
- `changeSummary` 不可为空
- `snapshot` MVP 阶段可选

---

## 12. ValidationIssue

当 Workflow Asset Draft 无法通过校验时，系统应该返回 ValidationIssue。

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
  "path": "steps[0].actions",
  "severity": "error",
  "message": "WorkflowStep 必须至少包含一个 action",
  "suggestion": "从原文中抽取该步骤的主要动作，或要求用户手动补充"
}
```

---

## 13. ExtractionReport

ExtractionReport 用于展示抽取质量。

```ts
export type ExtractionReport = {
  workflowId: string
  stepCount: number
  relationCount: number
  artifactCount: number
  suggestedTags: string[]
  suggestedCategory?: string
  confidence: number
  issues: ValidationIssue[]
  lowConfidenceFields?: string[]
}
```

---

## 14. Zod Schema 草案

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
  "image",
  "issue",
  "pr",
  "report",
  "note",
  "unknown",
])

export const ArtifactRoleSchema = z.enum([
  "input",
  "output",
  "intermediate",
])

export const AssetStatusSchema = z.enum([
  "draft",
  "reviewed",
  "reusable",
  "archived",
])

export const SeveritySchema = z.enum(["info", "warning", "error"])

export const SourceSchema = z.object({
  type: SourceTypeSchema,
  rawText: z.string().min(1),
  title: z.string().optional(),
  url: z.string().url().optional(),
  capturedAt: z.string().min(1),
})

export const WorkflowStepSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
  purpose: z.string().optional(),
  actions: z.array(z.string().min(1)).min(1),
  inputs: z.array(z.string()).optional(),
  outputs: z.array(z.string()).optional(),
  acceptance: z.array(z.string()).optional(),
  notes: z.array(z.string()).optional(),
  sourceText: z.string().optional(),
  confidence: z.number().min(0).max(1).optional(),
})

export const WorkflowRelationSchema = z.object({
  id: z.string().min(1),
  from: z.string().min(1),
  to: z.string().min(1),
  type: RelationTypeSchema,
  condition: z.string().optional(),
  label: z.string().optional(),
  confidence: z.number().min(0).max(1).optional(),
})

export const WorkflowArtifactSchema = z.object({
  id: z.string().min(1),
  name: z.string().min(1),
  type: ArtifactTypeSchema,
  role: ArtifactRoleSchema,
  producedBy: z.string().optional(),
  consumedBy: z.array(z.string()).optional(),
  description: z.string().optional(),
})

export const WorkflowUsageSchema = z.object({
  suitableFor: z.array(z.string()).optional(),
  notSuitableFor: z.array(z.string()).optional(),
  prerequisites: z.array(z.string()).optional(),
  reuseNotes: z.array(z.string()).optional(),
  examples: z.array(z.string()).optional(),
})

export const WorkflowMetadataSchema = z.object({
  createdAt: z.string().min(1),
  updatedAt: z.string().min(1),
  version: z.string().min(1),
  language: z.enum(["zh", "en", "mixed", "unknown"]),
  tags: z.array(z.string()).default([]),
  category: z.string().optional(),
  status: AssetStatusSchema,
  confidence: z.number().min(0).max(1).optional(),
})

export const WorkflowVersionSchema = z.object({
  version: z.string().min(1),
  changedAt: z.string().min(1),
  changeSummary: z.string().min(1),
  snapshot: z.unknown().optional(),
})

export const WorkflowAssetSchema = z.object({
  id: z.string().min(1),
  title: z.string().min(1),
  summary: z.string().optional(),
  goal: z.string().min(1),
  source: SourceSchema,
  steps: z.array(WorkflowStepSchema).min(1),
  relations: z.array(WorkflowRelationSchema).default([]),
  artifacts: z.array(WorkflowArtifactSchema).optional(),
  usage: WorkflowUsageSchema.optional(),
  metadata: WorkflowMetadataSchema,
  versions: z.array(WorkflowVersionSchema).optional(),
})
```

---

## 15. 跨对象校验规则

Zod 单字段校验不够，还需要跨对象校验。

### 15.1 WorkflowStep id 唯一

```text
所有 steps[*].id 必须唯一
```

### 15.2 WorkflowRelation 引用合法

```text
relation.from 必须存在于 step id 集合
relation.to 必须存在于 step id 集合
relation.from !== relation.to
```

### 15.3 WorkflowArtifact 引用合法

```text
artifact.producedBy 如果存在，必须存在于 step id 集合
artifact.consumedBy 如果存在，其中每个 id 都必须存在于 step id 集合
```

### 15.4 条件 Relation 建议包含 condition

```text
当 relation.type = condition 时，condition 不强制必填，但缺失时生成 warning
```

### 15.5 Metadata 默认值

```text
metadata.tags 默认为空数组
metadata.status 默认为 draft
metadata.version 默认为 0.1.0
```

---

## 16. 示例 Workflow Asset JSON

```json
{
  "id": "workflow_asset_design_review",
  "title": "设计稿视觉审查 Workflow",
  "summary": "用于从截图或页面预览中审查视觉问题，并整理成可执行修改建议。",
  "goal": "将页面视觉问题结构化为可修正的设计审查结果。",
  "source": {
    "type": "chat_message",
    "rawText": "先观察整体布局，再检查间距、字体、分割线和信息层级，最后输出修改建议。",
    "capturedAt": "2026-06-18T00:00:00.000Z"
  },
  "steps": [
    {
      "id": "step_01_overview",
      "title": "观察整体布局",
      "purpose": "先判断页面整体结构是否清晰。",
      "actions": ["观察页面布局", "识别主要信息区域", "判断视觉重心是否合理"],
      "inputs": ["页面截图或页面预览"],
      "outputs": ["整体布局判断"],
      "acceptance": ["能够说明页面结构是否清晰，以及主要问题在哪里"],
      "sourceText": "先观察整体布局",
      "confidence": 0.9
    },
    {
      "id": "step_02_detail_check",
      "title": "检查视觉细节",
      "purpose": "定位具体视觉噪音和可修正问题。",
      "actions": ["检查间距", "检查字体", "检查分割线", "检查信息层级"],
      "inputs": ["整体布局判断"],
      "outputs": ["视觉问题清单"],
      "acceptance": ["问题描述具体到可被设计或前端修改"],
      "sourceText": "再检查间距、字体、分割线和信息层级",
      "confidence": 0.92
    },
    {
      "id": "step_03_suggestion",
      "title": "输出修改建议",
      "purpose": "将审查结果转化为后续可执行的修改方向。",
      "actions": ["整理问题优先级", "输出修改建议"],
      "inputs": ["视觉问题清单"],
      "outputs": ["设计审查建议"],
      "acceptance": ["建议能直接分配给设计或前端执行"],
      "sourceText": "最后输出修改建议",
      "confidence": 0.9
    }
  ],
  "relations": [
    {
      "id": "relation_01",
      "from": "step_01_overview",
      "to": "step_02_detail_check",
      "type": "sequence",
      "confidence": 0.95
    },
    {
      "id": "relation_02",
      "from": "step_02_detail_check",
      "to": "step_03_suggestion",
      "type": "sequence",
      "confidence": 0.95
    }
  ],
  "artifacts": [
    {
      "id": "artifact_01",
      "name": "页面截图或页面预览",
      "type": "image",
      "role": "input",
      "consumedBy": ["step_01_overview"]
    },
    {
      "id": "artifact_02",
      "name": "设计审查建议",
      "type": "markdown",
      "role": "output",
      "producedBy": "step_03_suggestion"
    }
  ],
  "usage": {
    "suitableFor": ["页面视觉审查", "设计系统审查", "Agent 输出页面验收"],
    "notSuitableFor": ["真实用户行为验证", "需要业务数据支持的产品决策"],
    "prerequisites": ["需要有截图、页面预览或可访问页面"],
    "reuseNotes": ["每次复用前需要替换目标页面和审查重点"]
  },
  "metadata": {
    "createdAt": "2026-06-18T00:00:00.000Z",
    "updatedAt": "2026-06-18T00:00:00.000Z",
    "version": "0.1.0",
    "language": "zh",
    "tags": ["design-review", "visual-debug", "workflow-asset"],
    "category": "设计流程",
    "status": "draft",
    "confidence": 0.9
  },
  "versions": []
}
```

---

## 17. MVP Schema 版本

当前文档定义的 Schema 版本为：

```text
0.1.0
```

版本含义：

- `0`：未进入稳定 API
- `1`：第一版 Workflow Asset 对象结构
- `0`：尚未进行兼容性修订

---

## 18. 后续扩展方向

未来可以扩展，但不进入 MVP：

- Workflow Comparison Schema
- Workflow Composition Schema
- Workflow Template Schema
- Workflow Search Index Schema
- Workflow Export Adapter Schema
- n8n / GitHub Actions / LangGraph / Temporal 映射规则

当前核心边界保持为：

```text
Capture → Structure → Manage → Reuse
```
