# 01_Product_Intent

## 1. 文档目的

本文档定义 `workflow-capture` 的产品意图。

本项目的目标不是做一个普通的 Markdown 美化工具，也不是单纯把列表转换成流程图，而是把 GPT 对话、Markdown 文档、Agent 回复中隐式存在的任务流程定义，提取为可编辑、可渲染、可复用、可分发的 Workflow 对象。

一句话定义：

> Workflow Capture 是一个将自然语言任务流程编译成结构化 Workflow 对象的工具。

---

## 2. 背景问题

在与 GPT 交互时，GPT 经常会输出任务执行流程，例如：

```text
先分析需求 → 再拆分模块 → 然后定义数据结构 → 接着实现 MVP → 最后测试验收
```

这些内容在人类阅读时是清楚的，但在系统层面仍然只是普通文本。它们缺少对象结构，因此无法被稳定地：

- 渲染为流程视图
- 编辑为可维护资产
- 导出给 Agent 执行
- 追踪执行状态
- 复用到相似任务
- 形成长期流程资产库

当前的核心问题是：

> GPT 已经在隐式生成 Workflow，但这些 Workflow 被困在 Markdown 载体里，没有变成真正的对象。

---

## 3. 产品判断

本产品不把 Markdown 当成最终产物，而是把 Markdown 当成一种流程意图的输入载体。

因此，系统真正要处理的不是 Markdown 语法，而是文本背后的任务语义。

需要识别的语义包括：

| 文本信号 | 结构化含义 |
|---|---|
| 首先、第一步、先 | 阶段开始 |
| 然后、接着、再 | 顺序关系 |
| 如果、否则、当……时 | 条件关系 |
| 需要先完成 A 再做 B | 依赖关系 |
| 输出、产出、交付 | Output |
| 验收、检查、确认 | Checkpoint |
| 注意、风险、避免 | Risk / Constraint |
| 交给 Agent 执行 | Executor / Assignment |

---

## 4. 产品目标

### 4.1 MVP 目标

MVP 阶段只解决一个核心问题：

> 把一段包含任务流程的 GPT 回复或 Markdown 文本，解析成结构化 Workflow 对象，并渲染成可读、可编辑的流程视图。

MVP 需要支持：

1. 粘贴原始 GPT 回复或 Markdown 文本
2. 自动抽取 Workflow 草稿
3. 生成符合 Schema 的 Workflow JSON
4. 渲染为阶段卡片视图
5. 渲染为简单线性流程视图
6. 支持用户编辑节点内容
7. 支持导出 JSON / Markdown / Agent Task Pack

### 4.2 非 MVP 目标

以下能力暂不进入第一阶段：

- 自动执行 Workflow
- 多 Agent 调度
- 复杂 DAG 编排
- 权限系统
- 团队协作
- 版本管理
- 外部工具调用
- 自动创建 GitHub Issue / PR

原因：第一阶段最重要的问题不是执行，而是“能否稳定地从文本中抽出干净的流程对象”。

---

## 5. 核心用户场景

### 场景一：从 GPT 回复中捕获隐式流程

用户和 GPT 讨论一个任务后，GPT 输出了一段执行建议。用户希望把这段建议转成 Workflow，而不是复制到普通文档里。

流程：

```text
GPT 回复 → 粘贴到 Workflow Capture → 自动解析 → 渲染为 Workflow → 用户修正 → 保存
```

### 场景二：将流程沉淀为可复用资产

用户在多个项目中反复使用类似流程，例如设计审查、需求定义、文档打包、重构验收。用户希望把这些流程沉淀为 Workflow Fragment。

流程：

```text
一次性对话流程 → Workflow Fragment → 复用 / 改造 / 分发
```

### 场景三：将 Workflow 分发给 Agent 执行

用户把一个整理后的 Workflow 导出为 Agent Task Pack，让不同 Agent 按阶段执行。

流程：

```text
Workflow Object → Agent Task Pack → 分发给实现 / 审查 / 测试 Agent
```

---

## 6. 产品核心概念

### 6.1 Workflow

Workflow 是一个完整任务流程对象，描述一个任务从目标到交付的主要阶段、关系、产出和验收标准。

### 6.2 Workflow Fragment

Workflow Fragment 是可复用的局部流程。GPT 对话中经常出现的不是完整 Workflow，而是某个局部任务流程，例如：

- 视觉审查流程
- 文档定义流程
- MVP 拆解流程
- 代码重构流程
- 测试验收流程

因此系统必须支持 Fragment，而不是只支持完整大流程。

### 6.3 Stage

Stage 是 Workflow 的主要阶段，表达一个有明确意图、输入、动作和输出的执行单元。

### 6.4 Edge

Edge 表示 Stage 之间的关系，包括顺序、依赖、条件和反馈。

### 6.5 Checkpoint

Checkpoint 表示某个阶段是否完成、是否合格的判断点。

---

## 7. 产品边界

### 7.1 是什么

Workflow Capture 是：

- GPT 对话流程提取器
- Markdown 到 Workflow Object 的转换器
- Workflow 结构化编辑器
- Workflow 资产沉淀工具
- Agent Task Pack 的前置生成工具

### 7.2 不是什么

Workflow Capture 不是：

- 普通 Markdown 编辑器
- 纯流程图绘制工具
- 项目管理软件
- 自动化执行平台
- 多 Agent 编排平台
- Prompt 管理工具

---

## 8. 成功标准

MVP 是否成功，主要看以下标准：

1. 用户粘贴一段 GPT 回复后，系统能识别出主要阶段
2. 每个阶段能包含合理的意图、动作、输入、输出
3. 阶段之间能生成基本顺序关系
4. 用户能快速发现并修正解析错误
5. 修正后的 Workflow 可以导出为 JSON
6. 同一个 Workflow 可以被重新渲染为不同视图
7. 生成结果能作为后续 Agent 执行的输入

---

## 9. 产品原则

### 原则一：结构优先于视觉

先确保对象结构正确，再追求流程图视觉表达。

### 原则二：可编辑优先于自动化

LLM 抽取结果不能被当成绝对正确，必须允许用户快速修正。

### 原则三：Fragment 优先于大而全 Workflow

系统应该优先沉淀可复用局部流程，而不是强迫所有内容都变成完整流程。

### 原则四：Schema 校验优先于自由生成

LLM 可以参与抽取，但最终产物必须通过 Schema 校验。

### 原则五：中文优先，多语言兼容

产品定义语言优先使用中文，但对象字段应保持英文命名，方便工程实现与跨语言兼容。

---

## 10. 后续文档关系

本文档负责回答“为什么做”和“产品边界是什么”。

后续文档分工：

- `02_Workflow_Object_Model.md`：定义核心对象和对象关系
- `03_Extraction_Pipeline.md`：定义从文本到 Workflow 的解析流程
- `04_Workflow_Schema.md`：定义可落地的 JSON Schema / TypeScript 类型
