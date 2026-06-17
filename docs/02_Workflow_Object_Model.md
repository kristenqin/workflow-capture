# 02_Workflow_Object_Model

## 1. 文档目的

本文档定义 `workflow-capture` 的核心对象模型。

对象模型的目标是把 GPT 回复或 Markdown 文本中隐式存在的任务流程，转换为稳定、可验证、可渲染、可编辑、可导出的结构化对象。

一句话定义：

> Workflow Object Model 是自然语言任务流程进入系统后的标准对象层。

---

## 2. 建模原则

### 2.1 对象优先

系统不直接围绕 Markdown 段落建模，而是围绕任务流程对象建模。

Markdown 是输入，Workflow Object 才是产品资产。

### 2.2 Stage 是基本执行单位

一个 Workflow 可以很复杂，但最小可理解的执行单元应该是 Stage。

Stage 必须回答四个问题：

```text
这个阶段为什么存在？
它需要什么输入？
它要做什么动作？
它产出什么结果？
```

### 2.3 Edge 表达关系，不嵌入 Stage 内部

阶段之间的顺序、依赖、条件、反馈关系应该由 Edge 表达，而不是写死在 Stage 文本里。

这样后续才能支持不同渲染方式，例如线性流程、卡片视图、DAG 图。

### 2.4 Checkpoint 独立建模

验收标准不能只是 Stage 的普通描述，它应该是一个独立对象。

原因：Checkpoint 后续会用于：

- 标记阶段是否完成
- 支持 Agent 执行验收
- 支持人类审查
- 支持 Workflow 质量评估

### 2.5 Fragment 是一等对象

GPT 对话中经常出现的是局部流程，而不是完整 Workflow。

因此系统必须把 Workflow Fragment 作为一等对象，而不是完整 Workflow 的附属品。

---

## 3. 核心对象总览

```text
Workflow
├── Source
├── Goal
├── Stage[]
├── Edge[]
├── Output[]
├── Checkpoint[]
├── Risk[]
└── Metadata

WorkflowFragment
├── Scope
├── Stage[]
├── Edge[]
├── ReuseRule
└── Metadata
```

核心对象包括：

| 对象 | 作用 |
|---|---|
| Workflow | 完整任务流程 |
| WorkflowFragment | 可复用局部流程 |
| Source | 原始文本来源 |
| Stage | 主要执行阶段 |
| Edge | 阶段关系 |
| Action | 阶段内动作 |
| Input | 阶段输入 |
| Output | 阶段输出 |
| Checkpoint | 验收点 |
| Risk | 风险与注意事项 |
| Metadata | 解析、版本、状态等元信息 |

---

## 4. Workflow

Workflow 表示一个完整的任务流程。

它应该包含任务目标、阶段、阶段关系、产出、验收点和风险。

### 4.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Workflow 唯一标识 |
| title | string | Workflow 标题 |
| description | string | 简要说明 |
| source | Source | 原始来源 |
| goal | string | 任务目标 |
| stages | Stage[] | 执行阶段 |
| edges | Edge[] | 阶段关系 |
| outputs | Output[] | 总体产出 |
| checkpoints | Checkpoint[] | 验收点 |
| risks | Risk[] | 风险 |
| metadata | Metadata | 元信息 |

### 4.2 设计约束

- 一个 Workflow 必须至少有一个 Stage
- 一个 Workflow 必须有明确 goal
- Stage 的 id 必须在当前 Workflow 内唯一
- Edge 的 from / to 必须引用已存在 Stage
- Checkpoint 可以绑定到 Stage，也可以绑定到整个 Workflow

---

## 5. WorkflowFragment

WorkflowFragment 表示可复用的局部流程。

它适合沉淀 GPT 对话中反复出现的小流程，例如：

- 需求澄清流程
- 设计审查流程
- 文档打包流程
- 代码重构流程
- 测试验收流程

### 5.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Fragment 唯一标识 |
| title | string | Fragment 标题 |
| scope | string | 适用范围 |
| reusable | boolean | 是否可复用 |
| stages | Stage[] | 局部阶段 |
| edges | Edge[] | 局部阶段关系 |
| reuseRule | ReuseRule | 复用规则 |
| metadata | Metadata | 元信息 |

### 5.2 设计约束

- Fragment 不要求拥有完整的任务目标
- Fragment 可以被嵌入 Workflow
- Fragment 应该能独立渲染
- Fragment 必须保留来源信息，避免失去语境

---

## 6. Source

Source 表示 Workflow 从哪里被抽取出来。

### 6.1 来源类型

| 类型 | 说明 |
|---|---|
| chat_message | 单条 GPT 回复 |
| chat_thread | 一段对话上下文 |
| markdown | Markdown 文档 |
| document | 普通文档 |
| agent_response | Agent 输出 |
| manual | 用户手动创建 |

### 6.2 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| type | SourceType | 来源类型 |
| rawText | string | 原始文本 |
| title | string | 来源标题 |
| url | string | 可选来源链接 |
| capturedAt | string | 捕获时间 |

---

## 7. Stage

Stage 是 Workflow 的核心执行单元。

### 7.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Stage 唯一标识 |
| title | string | 阶段标题 |
| intent | string | 阶段意图 |
| type | StageType | 阶段类型 |
| inputs | Input[] | 阶段输入 |
| actions | Action[] | 阶段动作 |
| outputs | Output[] | 阶段输出 |
| checkpoints | Checkpoint[] | 阶段验收点 |
| owner | OwnerType | 负责人 |
| status | StageStatus | 阶段状态 |
| confidence | number | 抽取置信度 |

### 7.2 StageType

| 类型 | 含义 |
|---|---|
| analysis | 分析 |
| planning | 计划 |
| design | 设计 |
| implementation | 实现 |
| review | 审查 |
| testing | 测试 |
| delivery | 交付 |
| maintenance | 维护 |
| unknown | 无法判断 |

### 7.3 StageStatus

| 状态 | 含义 |
|---|---|
| draft | 草稿 |
| ready | 可执行 |
| running | 执行中 |
| blocked | 阻塞 |
| done | 已完成 |
| skipped | 已跳过 |

### 7.4 OwnerType

| 类型 | 含义 |
|---|---|
| user | 用户 |
| gpt | GPT |
| agent | Agent |
| human | 其他人类执行者 |
| system | 系统 |
| unknown | 未知 |

---

## 8. Edge

Edge 表示 Stage 之间的关系。

### 8.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Edge 唯一标识 |
| from | string | 起始 Stage id |
| to | string | 目标 Stage id |
| type | EdgeType | 关系类型 |
| label | string | 关系说明 |
| condition | string | 条件表达 |

### 8.2 EdgeType

| 类型 | 含义 |
|---|---|
| sequence | 顺序关系 |
| dependency | 依赖关系 |
| condition | 条件关系 |
| feedback | 反馈回路 |
| parallel | 并行关系 |

### 8.3 设计约束

- 默认关系为 sequence
- 只有文本中出现明确前置条件时，才标记 dependency
- 只有文本中出现 if / 如果 / 当……时，才标记 condition
- feedback 不应过度推断，必须有明显返工、审查、修正语义

---

## 9. Input / Action / Output

### 9.1 Input

Input 表示某个阶段需要接收什么信息或材料。

常见输入包括：

- 用户想法
- 原始 Markdown
- 需求上下文
- 设计稿
- 代码文件
- 上一阶段产出

### 9.2 Action

Action 表示阶段内部要做的动作。

Action 应该使用动词开头，例如：

```text
识别任务目标
拆分执行阶段
定义对象字段
校验 Schema
渲染阶段卡片
```

### 9.3 Output

Output 表示阶段或 Workflow 的产出。

常见输出包括：

- 需求定义文档
- Workflow JSON
- Workflow Fragment
- 渲染视图
- Agent Task Pack
- 验收报告

---

## 10. Checkpoint

Checkpoint 表示判断任务是否完成或合格的标准。

### 10.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| id | string | Checkpoint 唯一标识 |
| stageId | string | 绑定的 Stage，可为空 |
| description | string | 验收描述 |
| passCondition | string | 通过条件 |
| severity | string | 严重程度 |
| status | string | 检查状态 |

### 10.2 示例

```text
描述：每个 Stage 都必须包含 title、intent、actions
通过条件：不存在空 title；actions 数量大于 0；intent 可被人类理解
```

---

## 11. Risk

Risk 表示解析、执行或使用过程中的风险。

常见风险包括：

- LLM 过度推断
- 阶段粒度过粗
- 阶段粒度过细
- 输入输出缺失
- 关系错误
- 验收标准不明确
- 原始上下文不足

Risk 应该可以被渲染到右侧审查面板，帮助用户修正 Workflow。

---

## 12. Metadata

Metadata 用于记录对象生成与维护信息。

建议字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| createdAt | string | 创建时间 |
| updatedAt | string | 更新时间 |
| createdBy | string | 创建者 |
| extractorVersion | string | 抽取器版本 |
| schemaVersion | string | Schema 版本 |
| language | string | 主语言 |
| confidence | number | 总体置信度 |

---

## 13. 对象关系

```text
Workflow owns Stage
Workflow owns Edge
Workflow owns Output
Workflow owns Checkpoint
Workflow owns Risk

Stage owns Input
Stage owns Action
Stage owns Output
Stage owns Checkpoint

Edge references Stage
Checkpoint references Stage
WorkflowFragment can be embedded into Workflow
```

---

## 14. 最小对象闭环

MVP 阶段最小闭环如下：

```text
Source.rawText
  ↓
Workflow.goal
  ↓
Stage[]
  ↓
Edge[]
  ↓
Checkpoint[]
  ↓
Renderer
```

只要这个闭环稳定，产品就可以进入第一版实现。
