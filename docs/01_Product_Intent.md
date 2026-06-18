# 01_Product_Intent

## 1. 文档目的

本文档定义 `workflow-capture` 的产品意图。

经过需求优先级调整后，本项目当前最紧要的目标不是接入外部自动化平台，也不是设计一个执行引擎，而是先把 GPT 对话、Markdown 文档、Agent 回复中隐式出现的 workflow 沉淀为可管理、可检索、可复用的 workflow 资产。

一句话定义：

> Workflow Capture 是一个从 GPT 对话和 Markdown 中捕获隐式 workflow，并将其沉淀为可管理 workflow 资产的工具。

当前阶段的核心不是 Execute，也不是 Adapter，而是 Asset。

---

## 2. 产品共识

### 2.1 Workflow 首先是资产，不是自动化脚本

很多 workflow 的价值并不在于立刻被自动执行，而在于它们沉淀了任务理解、执行经验、判断结构和复用路径。

例如：

- 需求澄清 workflow
- 设计审查 workflow
- 文档定义 workflow
- 视觉 debug workflow
- Agent 分工 workflow
- 测试验收 workflow
- 代码重构 workflow
- 产品讨论沉淀 workflow

这些 workflow 即使暂时不接入 n8n、GitHub Actions、Temporal 或 LangGraph，也已经具有很高资产价值。

### 2.2 过早考虑自动化适配会损耗抽象质量

如果一开始就把 workflow 设计成适配某些自动化平台，容易导致对象模型过早被外部工具结构污染。

典型风险：

- 为了适配 n8n，过早把节点设计成工具调用节点
- 为了适配 GitHub Actions，过早把流程压成 job / step
- 为了适配 LangGraph，过早引入状态图概念
- 为了适配 Temporal，过早引入 activity / retry / worker

这些设计会让 workflow 的原始抽象价值变窄。

当前阶段应该优先保留 workflow 作为“任务认知结构”的完整性。

### 2.3 外部自动化平台是远期出口，不是 MVP 中心

外部平台适配仍然有价值，但不进入当前 MVP 的核心目标。

它应该被放在未来阶段：

```text
Workflow Asset Management
  ↓
Workflow Reuse / Composition
  ↓
Exporter / Automation Adapter
```

当前只需要保证对象结构未来可以扩展，不需要现在就围绕外部平台建模。

---

## 3. 当前核心问题

GPT 经常会在对话中隐式生成 workflow，例如：

```text
先分析需求，再拆分模块，然后定义文档，接着交给 agent 执行，最后做验收。
```

这些内容当下的问题不是“不能自动执行”，而是：

> 它们很容易在聊天流中消失，无法被命名、归档、检索、比较、复用和持续迭代。

因此当前最重要的问题是 workflow 资产管理：

- 如何从对话中捕获 workflow
- 如何给 workflow 命名
- 如何描述适用场景
- 如何标记输入和输出
- 如何记录步骤结构
- 如何保存版本
- 如何复用到新的任务
- 如何从一堆 workflow 中找到合适的那个
- 如何比较相似 workflow 的差异

---

## 4. 产品目标

### 4.1 MVP 目标

MVP 阶段只解决一个核心问题：

> 把一段包含 workflow 的 GPT 回复或 Markdown 文本，沉淀为可保存、可管理、可检索、可复用的 Workflow Asset。

MVP 需要支持：

1. 粘贴原始 GPT 回复或 Markdown 文本
2. 自动识别其中的 workflow 候选
3. 抽取 workflow 的标题、目标、适用场景、步骤、输入、输出、验收点
4. 生成轻量 Workflow Asset 对象
5. 渲染为可读的流程卡片或步骤视图
6. 支持用户手动修正
7. 支持添加标签、适用场景、复用说明
8. 支持保存到 workflow 资产库
9. 支持搜索、筛选、查看、复制和导出 Markdown / JSON

### 4.2 当前不做

当前阶段不做：

- 自动执行 workflow
- 接入 n8n / Zapier / Make
- 接入 GitHub Actions
- 接入 Temporal
- 接入 LangGraph
- 多 Agent 调度
- 运行时状态管理
- 任务队列
- 凭证管理
- SaaS Connector

这些能力可以未来再考虑，但不能干扰 MVP 对 workflow 资产本身的建模。

---

## 5. 产品边界

Workflow Capture 当前负责：

```text
Capture：从 GPT / Markdown 中捕获 workflow
Structure：转换为清晰的 Workflow Asset
Manage：保存、命名、分类、检索、版本化
Reuse：复制、改造、组合、导出
```

Workflow Capture 当前不负责：

```text
Execute：真正运行 workflow
Schedule：调度任务
Retry：失败重试
Credential：管理外部服务密钥
Connector：连接外部平台
Runtime State：维护长期运行状态
```

更短的边界表达：

> 先沉淀资产，再考虑执行。先管理 workflow，再适配 workflow engine。

---

## 6. 核心用户场景

### 场景一：从 GPT 回复中保存一个 workflow

用户和 GPT 讨论一个任务后，GPT 输出了一套执行流程。用户希望把它保存为资产，而不是让它沉没在聊天记录里。

```text
GPT 回复 → Workflow Capture → Workflow Asset → 保存到资产库
```

### 场景二：复用已有 workflow

用户遇到一个类似任务，希望从资产库里找到之前沉淀过的流程。

```text
搜索 / 标签筛选 → 找到 workflow → 复制为新实例 → 根据当前任务修改
```

### 场景三：比较相似 workflow

用户已经沉淀了多个相近流程，例如多个“设计审查 workflow”，希望比较它们的差异。

```text
选择多个 Workflow Asset → 对比目标 / 步骤 / 输入 / 输出 / 验收点
```

### 场景四：将 workflow 作为 agent 执行前的规范材料

用户暂时不需要系统自动执行，只需要把 workflow 导出成清晰文档，交给人或 agent 使用。

```text
Workflow Asset → Markdown / JSON → 分发给 Agent / 人工执行者
```

---

## 7. 产品核心概念

### 7.1 Workflow Asset

Workflow Asset 是当前产品的核心对象。

它表示一套可保存、可管理、可复用的任务流程资产。

它不等于自动化脚本，也不等于执行实例。

### 7.2 Workflow Step

Workflow Step 表示 workflow 中的一个步骤。

它描述一个阶段要做什么、需要什么输入、会产出什么结果、如何判断完成。

### 7.3 Workflow Library

Workflow Library 是保存 workflow 资产的地方。

它需要支持：

- 搜索
- 标签
- 分类
- 收藏
- 版本
- 复制
- 对比

### 7.4 Workflow Variant

Workflow Variant 表示同一个 workflow 在不同场景下的变体。

例如：

```text
设计审查 workflow
├── UI 页面审查版
├── 设计系统审查版
├── Agent 输出审查版
└── 移动端审查版
```

### 7.5 Workflow Usage Note

Usage Note 表示这个 workflow 适合什么时候使用、不适合什么时候使用、使用时要注意什么。

这对资产复用非常关键。

---

## 8. 成功标准

MVP 是否成功，主要看以下标准：

1. 用户能从 GPT 回复中快速保存一个 workflow
2. 保存后的 workflow 有清晰标题、目标、步骤和适用场景
3. 用户能手动修正抽取错误
4. 用户能给 workflow 添加标签和分类
5. 用户能在资产库中搜索和筛选 workflow
6. 用户能复制已有 workflow 并改造成新 workflow
7. 用户能导出 Markdown / JSON 给 agent 或人工执行者
8. 用户能逐步形成自己的 workflow 资产库

---

## 9. 产品原则

### 原则一：资产优先于自动化

先把 workflow 变成可管理资产，再考虑自动执行。

### 原则二：抽象完整性优先于平台适配

不要为了适配某个外部工具，过早扭曲 workflow 本身的表达。

### 原则三：可复用优先于一次性生成

保存下来的 workflow 应该能被未来任务复用，而不是只服务当前对话。

### 原则四：可编辑优先于自动抽取

LLM 抽取结果不能被当成绝对正确，必须允许用户快速修正。

### 原则五：管理能力优先于执行能力

MVP 更重要的是命名、分类、检索、版本、复用，而不是执行。

### 原则六：中文优先，多语言兼容

产品定义语言优先使用中文，但对象字段保持英文命名，方便工程实现与跨语言兼容。

---

## 10. 后续文档关系

本文档负责回答“为什么做”和“产品边界是什么”。

后续文档分工：

- `02_Workflow_Object_Model.md`：定义 Workflow Asset 对象模型
- `03_Extraction_Pipeline.md`：定义从文本到 Workflow Asset 的解析流程
- `04_Workflow_Schema.md`：定义可落地的 Asset Schema / TypeScript / Zod 草案
