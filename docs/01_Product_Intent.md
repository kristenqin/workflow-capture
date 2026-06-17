# 01_Product_Intent

## 1. 文档目的

本文档定义 `workflow-capture` 的产品意图。

本项目的目标不是做一个新的 Workflow 执行引擎，也不是重新实现 n8n、GitHub Actions、Temporal、LangGraph 这类自动化工具已经具备的能力。

本项目真正要解决的是：

> GPT 对话、Markdown 文档、Agent 回复里经常隐式包含任务流程，但这些流程仍然停留在自然语言文本里，无法被稳定捕获、编辑、渲染，并适配到外部自动化工具。

一句话定义：

> Workflow Capture 是一个从 GPT 对话和 Markdown 中提取隐式任务流程，并转换为可视化、可编辑、可导出的 Workflow IR 的工具。

其中 IR 指 Intermediate Representation，即中间表示。

---

## 2. 产品共识

### 2.1 不做 Workflow Runtime

Workflow Capture 不负责真正执行任务。

不做：

- 任务调度
- 状态机运行
- 重试机制
- 队列系统
- 定时任务
- 外部账号授权
- SaaS 连接器
- 执行日志
- 运行时权限控制

这些能力已经有成熟工具负责，例如：

- n8n / Zapier / Make：自动化集成
- GitHub Actions：代码仓库与 CI/CD 自动化
- Temporal：可靠长流程编排
- LangGraph：Agent 状态流转
- Airflow / Prefect：数据管道

Workflow Capture 应该和这些工具兼容，而不是替代它们。

### 2.2 做 Workflow Intent Compiler

Workflow Capture 负责把自然语言里的流程意图编译成轻量中间表示。

核心路径：

```text
GPT Reply / Markdown / Agent Response
  ↓
Workflow Intent Extraction
  ↓
Workflow IR
  ↓
Human Review / Edit
  ↓
Export Adapter
  ↓
n8n / GitHub Actions / Temporal Skeleton / LangGraph Skeleton / Agent Task Pack
```

也就是说，本产品的核心不是 Execute，而是 Capture + Compile + Export。

---

## 3. 背景问题

GPT 经常会输出类似内容：

```text
先分析需求，再拆分模块，然后定义数据结构，接着实现 MVP，最后测试验收。
```

这段文本在人类看来是一个流程，但在系统看来仍然只是 Markdown 或普通文本。

它缺少：

- 可识别的任务节点
- 节点之间的关系
- 哪些部分可以自动化
- 哪些部分必须人工处理
- 哪些节点适合导出到 n8n
- 哪些节点适合导出到 GitHub Actions
- 哪些节点只能作为 Agent Task Pack

当前核心问题不是“没有执行引擎”，而是：

> GPT 生成了大量隐式 Workflow Intent，但这些 intent 没有被捕获为可适配外部工具的中间表示。

---

## 4. 产品目标

### 4.1 MVP 目标

MVP 阶段只解决一个核心问题：

> 把一段包含任务流程的 GPT 回复或 Markdown 文本，解析成轻量 Workflow IR，并渲染成可读、可编辑、可导出的结构。

MVP 需要支持：

1. 粘贴原始 GPT 回复或 Markdown 文本
2. 自动识别流程意图
3. 抽取 TaskNode、Relation、Artifact
4. 判断节点自动化可能性
5. 生成符合 Schema 的 Workflow IR JSON
6. 渲染为线性流程视图和任务卡片视图
7. 支持用户修正节点内容、关系和自动化提示
8. 支持导出 Generic JSON / Markdown / Agent Task Pack

### 4.2 后续目标

后续可以逐步支持外部工具适配器：

```text
MVP-1：Workflow IR + JSON / Markdown / Agent Task Pack
MVP-2：n8n Exporter
MVP-3：GitHub Actions Exporter
MVP-4：LangGraph / Temporal Skeleton Exporter
```

### 4.3 非目标

以下能力明确不属于本项目核心范围：

- 自建 Workflow Runtime
- 自建可视化自动化执行平台
- 自建 SaaS Connector 市场
- 自建凭证管理系统
- 自建任务队列与重试机制
- 自建复杂权限协作系统
- 替代 n8n / GitHub Actions / Temporal / LangGraph

---

## 5. 产品边界

Workflow Capture 负责：

```text
Capture：捕获 GPT / Markdown 中的隐式流程
Compile：转换为 Workflow IR
Review：渲染给用户审查和修正
Export：导出到外部工具或通用格式
```

Workflow Capture 不负责：

```text
Execute：真正运行 workflow
Schedule：调度任务
Retry：失败重试
Credential：管理外部服务密钥
Runtime State：维护长期运行状态
Connector：连接所有 SaaS 服务
```

更短的边界表达：

> Capture，不 Execute。Compile，不 Runtime。IR，不 Engine。Adapter，不 Platform。

---

## 6. 核心用户场景

### 场景一：从 GPT 回复中捕获流程意图

用户和 GPT 讨论一个任务后，GPT 输出了一段执行建议。用户希望把这段建议转成结构化 IR，而不是复制到普通文档里。

```text
GPT 回复 → Workflow Capture → Workflow IR → 可视化审查 → 保存
```

### 场景二：判断哪些步骤适合自动化

用户拿到一个流程后，不确定哪些步骤可以交给工具执行，哪些必须人工处理。

```text
Workflow IR → Automation Hint → manual / automatable / external
```

示例：

| 任务节点 | 自动化判断 | 适配方向 |
|---|---|---|
| 审查页面视觉问题 | manual | 人工审查 / Agent Task |
| 调用 GitHub API 创建 issue | automatable | n8n / GitHub Actions |
| 运行测试脚本 | automatable | GitHub Actions |
| 总结文档 | automatable | LLM / Agent Task Pack |
| 判断是否通过验收 | manual / external | Review Gate |

### 场景三：导出到外部自动化工具

用户希望把整理后的 IR 导出给外部工具。

```text
Workflow IR → Export Adapter → n8n JSON / GitHub Actions YAML / Agent Task Pack
```

MVP 不要求所有导出格式都可直接运行，但必须保持结构清晰、映射关系明确。

---

## 7. 产品核心概念

### 7.1 Workflow IR

Workflow IR 是 Workflow Capture 的核心中间表示。

它不负责执行，只负责表达：

- 这个流程想完成什么
- 有哪些任务节点
- 节点之间是什么关系
- 产生哪些产物
- 哪些节点可能被自动化
- 适合导出到哪些外部工具

### 7.2 TaskNode

TaskNode 是流程里的最小任务节点。

它不是运行时任务，而是一个可被理解、编辑和映射的任务单元。

### 7.3 Relation

Relation 表示 TaskNode 之间的关系，包括顺序、依赖、条件、并行、反馈。

### 7.4 Artifact

Artifact 表示流程中产生或消费的产物，例如 Markdown 文档、JSON、代码文件、Issue、PR、报告。

### 7.5 ExportTarget

ExportTarget 表示 IR 可以适配的目标格式或外部工具，例如：

- markdown
- json
- agent_task_pack
- n8n
- github_actions
- langgraph
- temporal

---

## 8. 成功标准

MVP 是否成功，主要看以下标准：

1. 用户粘贴 GPT 回复后，系统能识别主要任务节点
2. 系统能识别节点之间的基本关系
3. 系统能判断节点是 manual、automatable 还是 unknown
4. 系统能给出 suggestedTargets，例如 n8n、github_actions、agent_task_pack
5. 用户能快速修正节点、关系和自动化提示
6. Workflow IR 可以导出为通用 JSON
7. 同一个 IR 可以渲染为线性视图和任务卡片视图
8. 后续 Exporter 可以基于 IR 映射到外部工具，而不需要重新解析原文

---

## 9. 产品原则

### 原则一：轻量 IR 优先

不要把执行引擎的复杂状态提前塞进对象模型。

### 原则二：适配优先于自建

能导出到外部工具，就不要自己实现执行能力。

### 原则三：可编辑优先于自动化

LLM 抽取结果不能被当成绝对正确，必须允许用户快速修正。

### 原则四：自动化提示不是自动执行

系统可以判断某个节点适合自动化，但不代表本系统负责执行它。

### 原则五：Schema 校验优先于自由生成

LLM 可以参与抽取，但最终 IR 必须通过 Schema 校验。

### 原则六：中文优先，多语言兼容

产品定义语言优先使用中文，但对象字段保持英文命名，方便工程实现与跨语言兼容。

---

## 10. 后续文档关系

本文档负责回答“为什么做”和“产品边界是什么”。

后续文档分工：

- `02_Workflow_Object_Model.md`：定义轻量 Workflow IR 对象模型
- `03_Extraction_Pipeline.md`：定义从文本到 Workflow IR 的解析流程
- `04_Workflow_Schema.md`：定义可落地的 IR Schema / TypeScript / Zod 草案
