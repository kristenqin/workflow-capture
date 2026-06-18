# 03_Extraction_Pipeline

## 1. 文档目的

本文档定义 `workflow-capture` 从自然语言文本中抽取 Workflow Asset 的处理流程。

当前阶段，解析 Pipeline 的目标不是生成可执行自动化流程，也不是适配外部 workflow 平台，而是把 GPT 回复、Markdown 文档、Agent 输出中的隐式 workflow 沉淀为可管理、可检索、可复用的资产。

一句话定义：

> Extraction Pipeline 是把“文本里的流程经验”转换成“可管理 Workflow Asset”的沉淀过程。

---

## 2. 总体流程

```text
Raw Text
  ↓
01 输入标准化
  ↓
02 文本分块
  ↓
03 Workflow 候选识别
  ↓
04 Workflow Asset 草稿生成
  ↓
05 WorkflowStep 抽取
  ↓
06 输入 / 动作 / 输出 / 验收抽取
  ↓
07 WorkflowRelation 推断
  ↓
08 WorkflowArtifact 抽取
  ↓
09 WorkflowUsage 抽取
  ↓
10 标签 / 分类建议
  ↓
11 Schema 校验
  ↓
12 Normalize 标准化
  ↓
13 渲染预览
  ↓
14 用户修正
  ↓
15 保存到 Workflow Library
```

---

## 3. Pipeline 原则

### 3.1 资产沉淀优先

解析的目标是得到一个可以长期保存和复用的 Workflow Asset，而不是一个一次性执行计划。

### 3.2 不直接信任 LLM 输出

LLM 可以参与抽取，但最终结果必须经过：

- Schema 校验
- 字段标准化
- 引用关系校验
- 置信度标记
- 用户可编辑审查

### 3.3 保留原文证据

每个重要 WorkflowStep 都应该尽量保留 `sourceText`，方便用户知道该步骤从哪里抽取出来。

### 3.4 优先表达 workflow 的复用语境

一个 workflow 是否有资产价值，很大程度取决于它是否能说明：

- 适合什么场景
- 不适合什么场景
- 复用前需要什么条件
- 复用时应该注意什么

因此 Pipeline 不只抽步骤，还要抽 WorkflowUsage。

### 3.5 不做自动化平台推断

当前阶段不抽取：

- n8n 节点
- GitHub Actions job
- Temporal activity
- LangGraph state
- automation hint
- export target

这些可以未来扩展，但不能进入 MVP Pipeline 核心。

---

## 4. Step 01：输入标准化

### 4.1 目标

把不同来源的文本统一为可处理的标准输入。

### 4.2 输入

- GPT 回复文本
- Markdown 文档
- Agent 输出
- 用户手动输入文本

### 4.3 处理动作

- 去除无意义空白
- 保留标题层级
- 保留列表结构
- 保留代码块，但默认不解析代码块内部流程
- 识别语言类型
- 记录原始 rawText

### 4.4 输出

```ts
NormalizedInput
```

建议字段：

```ts
{
  rawText: string
  normalizedText: string
  language: "zh" | "en" | "mixed" | "unknown"
  blocks: TextBlock[]
}
```

---

## 5. Step 02：文本分块

### 5.1 目标

把长文本切成更容易识别语义的块。

### 5.2 Block 类型

| 类型 | 说明 |
|---|---|
| heading | 标题 |
| paragraph | 普通段落 |
| ordered_list | 有序列表 |
| unordered_list | 无序列表 |
| table | 表格 |
| code | 代码块 |
| quote | 引用 |

### 5.3 处理规则

- Markdown 标题天然形成层级
- 有序列表通常是强 workflow 信号
- 无序列表可能是动作、注意事项、输入输出或适用场景
- 表格可能包含步骤矩阵、字段定义、验收项
- 代码块默认作为示例或证据，不作为 WorkflowStep 主来源

---

## 6. Step 03：Workflow 候选识别

### 6.1 目标

判断哪些文本块可能包含可沉淀的 workflow。

### 6.2 强 workflow 信号

```text
第一步 / 第二步 / Step 1 / Step 2
首先 / 然后 / 接着 / 最后
流程 / workflow / pipeline
阶段 / stage / phase
输入 / 输出 / 验收 / 风险
先……再……
适合……
不适合……
复用时……
```

### 6.3 弱 workflow 信号

```text
可以这样做
建议按以下方式
拆成几个部分
先处理核心问题
后续再补充
```

### 6.4 输出

```ts
WorkflowCandidate[]
```

每个候选包含：

- candidateId
- sourceBlocks
- confidence
- reason
- possibleTitle
- possibleScope

---

## 7. Step 04：Workflow Asset 草稿生成

### 7.1 目标

为候选 workflow 生成一个资产草稿。

### 7.2 需要抽取

- title
- summary
- goal
- possible category
- possible tags
- source

### 7.3 标题生成规则

标题应该描述 workflow 的用途，而不是复述原文开头。

好标题：

```text
设计稿视觉审查 Workflow
GPT 回复流程沉淀 Workflow
Agent 文档打包 Workflow
```

差标题：

```text
首先分析需求
可以这样做
一些步骤
```

---

## 8. Step 05：WorkflowStep 抽取

### 8.1 目标

从候选文本中抽取 workflow 步骤。

### 8.2 抽取规则

以下内容可以成为 WorkflowStep：

- 明确编号步骤
- 明确阶段标题
- 有独立目标的段落
- 多个动作围绕同一意图展开的文本块
- 对复用有价值的流程环节

以下内容不应直接成为 WorkflowStep：

- 普通解释性句子
- 单独的背景描述
- 单独的字段定义
- 单独的风险提示
- 与任务推进无关的表达

### 8.3 粒度判断

一个 WorkflowStep 的合理粒度是：

```text
有明确目的 + 有动作 + 有输入或输出 + 可复用
```

过粗示例：

```text
完成整个项目
```

过细示例：

```text
打开页面
点击按钮
复制文本
```

MVP 阶段优先采用中等粒度。

---

## 9. Step 06：输入 / 动作 / 输出 / 验收抽取

### 9.1 输入抽取

识别文本中的输入材料：

```text
基于……
使用……
拿到……之后
输入是……
前置条件是……
```

### 9.2 动作抽取

识别动词短语，例如：

```text
分析需求
拆分任务
定义字段
生成文档
校验结果
整理资产
```

动作应该尽量转换成动宾结构。

### 9.3 输出抽取

识别产出表达：

```text
输出……
产出……
生成……
得到……
交付……
形成……
```

### 9.4 验收抽取

识别验收表达：

```text
验收标准
检查
确认
通过条件
是否满足
必须保证
完成标志
```

这些内容进入 WorkflowStep.acceptance。

---

## 10. Step 07：WorkflowRelation 推断

### 10.1 默认规则

如果文本是有序列表，默认生成 sequence Relation。

```text
Step 1 → Step 2 → Step 3
```

### 10.2 依赖关系

出现以下表达时，生成 dependency：

```text
需要先完成 A 才能 B
B 依赖 A
基于上一步结果
在 A 之后再做 B
```

### 10.3 条件关系

出现以下表达时，生成 condition：

```text
如果……则……
当……时……
否则……
根据……选择……
```

### 10.4 并行关系

出现以下表达时，生成 parallel：

```text
可以同时进行
并行处理
分别由不同角色处理
```

### 10.5 反馈关系

出现以下表达时，生成 feedback：

```text
如果不通过则返回修改
审查后回到上一步
测试失败后重新实现
```

---

## 11. Step 08：WorkflowArtifact 抽取

### 11.1 目标

识别 workflow 中涉及的输入材料、输出材料和中间产物。

### 11.2 常见 Artifact

```text
Markdown 文档
JSON 数据
设计稿
截图
代码文件
Issue
PR
测试报告
验收报告
```

### 11.3 处理规则

- 明确“输出 / 生成 / 导出”的内容通常是 output artifact
- 明确“基于 / 使用 / 读取”的内容通常是 input artifact
- 上一步产物被下一步使用时，可以标记为 intermediate

---

## 12. Step 09：WorkflowUsage 抽取

### 12.1 目标

抽取 workflow 的复用语境。

这是当前阶段最重要的资产管理信息之一。

### 12.2 需要识别

- suitableFor：适合什么任务
- notSuitableFor：不适合什么任务
- prerequisites：使用前提
- reuseNotes：复用说明
- examples：使用示例

### 12.3 触发信号

```text
适合……
不适合……
前提是……
使用前需要……
复用时……
注意……
可以用于……
不要用于……
```

### 12.4 缺失处理

如果原文没有明确说明 usage，可以生成低置信度建议，但必须允许用户手动修改。

---

## 13. Step 10：标签 / 分类建议

### 13.1 目标

为 workflow 资产生成管理信息，方便进入资产库后检索和筛选。

### 13.2 建议标签来源

- 任务类型
- 产物类型
- 使用场景
- 角色
- 来源
- 复杂度

### 13.3 示例标签

```text
design-review
documentation
agent-task
mvp-planning
visual-debug
code-review
schema-design
```

### 13.4 分类建议

分类应该比标签更稳定。

示例：

```text
设计流程
文档流程
工程流程
Agent 流程
产品流程
学习流程
```

---

## 14. Step 11：Schema 校验

### 14.1 目标

确保 Workflow Asset Draft 符合 `04_Workflow_Schema.md` 定义。

### 14.2 校验内容

- 必填字段是否存在
- 枚举值是否合法
- WorkflowStep id 是否唯一
- WorkflowRelation 引用是否有效
- WorkflowArtifact 引用是否有效
- 数组字段是否为数组
- confidence 是否在 0 到 1 之间

### 14.3 校验失败处理

校验失败时，不应直接丢弃结果，而应该返回：

```ts
ValidationIssue[]
```

每个 issue 包含：

- path
- severity
- message
- suggestion

---

## 15. Step 12：Normalize 标准化

### 15.1 目标

把合法 Draft 整理成稳定 Workflow Asset。

### 15.2 标准化规则

- 自动生成缺失 id
- 统一步骤编号
- 修正空白字符
- 动作改为动宾短语
- 默认 Relation type 为 sequence
- 默认 metadata.status 为 draft
- 默认 metadata.version 为 0.1.0
- 补充 createdAt / updatedAt
- 补充 tags / category 建议

---

## 16. Step 13：渲染预览

### 16.1 目标

将 Workflow Asset 渲染给用户审查。

MVP 支持两种视图：

1. Linear Workflow View
2. Workflow Asset Card View

### 16.2 渲染重点

渲染结果应该突出：

- workflow 标题和目标
- 适用场景
- 步骤顺序
- 每个步骤的输入 / 动作 / 输出 / 验收
- 标签和分类
- 低置信度字段

低置信度字段应该显式标记，提醒用户修正。

---

## 17. Step 14：用户修正

### 17.1 目标

让用户把 LLM 抽取结果修正为可信 workflow 资产。

### 17.2 可编辑字段

MVP 至少支持编辑：

- WorkflowAsset title
- WorkflowAsset summary
- WorkflowAsset goal
- WorkflowStep title
- WorkflowStep purpose
- WorkflowStep actions
- WorkflowStep inputs
- WorkflowStep outputs
- WorkflowStep acceptance
- WorkflowUsage
- tags
- category
- status

### 17.3 修正后处理

用户修正后，需要重新：

```text
Validate → Normalize → Render
```

---

## 18. Step 15：保存到 Workflow Library

### 18.1 目标

将修正后的 Workflow Asset 保存到资产库。

### 18.2 Library 需要支持

MVP 阶段至少需要：

- 保存
- 列表查看
- 搜索
- 标签筛选
- 分类筛选
- 查看详情
- 复制 workflow
- 导出 Markdown / JSON

### 18.3 保存后的状态

新保存的 workflow 默认状态为：

```text
draft
```

用户审查后可以改为：

```text
reviewed / reusable / archived
```

---

## 19. LLM 抽取 Prompt 要求

LLM 抽取时应遵守：

1. 不要扩写成新的 workflow
2. 优先忠实原文
3. 不要把普通解释段落强行变成 WorkflowStep
4. 不要为了自动化平台改变抽象结构
5. 不要生成 n8n / GitHub Actions / LangGraph 适配字段
6. 不确定时降低 confidence
7. 输出必须是 JSON
8. 字段名使用英文
9. 内容语言优先保持原文语言
10. usage / tags / category 可以建议，但必须允许用户修改

---

## 20. MVP 测试样例

### 20.1 简单线性 workflow

输入：

```text
先分析需求，然后设计对象模型，接着实现 MVP，最后测试验收。
```

期望：

- 1 个 WorkflowAsset
- 4 个 WorkflowStep
- 3 条 sequence Relation
- 有 title / goal / tags / category 建议

### 20.2 带复用语境

输入：

```text
这套流程适合用于设计稿视觉审查，不适合用于真实用户数据验证。使用前需要有页面截图。
```

期望：

- 抽取 suitableFor
- 抽取 notSuitableFor
- 抽取 prerequisites

### 20.3 带验收条件

输入：

```text
最后做验收，确认每个步骤都有输入、动作和输出。
```

期望：

- 抽取 acceptance
- 绑定到对应 WorkflowStep

---

## 21. 输出结果要求

Pipeline 最终必须输出：

```text
Validated Workflow Asset
```

并附带：

```text
ValidationIssue[]
ExtractionReport
```

ExtractionReport 应包括：

- 总体置信度
- WorkflowStep 数量
- WorkflowRelation 数量
- WorkflowArtifact 数量
- 标签和分类建议
- 低置信度字段
- 可能需要人工修正的问题

---

## 22. 第一版实现建议

MVP 可以采用以下实现方式：

```text
前端输入 Markdown
  ↓
调用 LLM 抽取 Workflow Asset Draft JSON
  ↓
Zod 校验
  ↓
Normalize
  ↓
React 渲染 Workflow Asset Card
  ↓
用户编辑
  ↓
重新校验
  ↓
保存到 Workflow Library
  ↓
支持搜索 / 复制 / 导出
```

第一版重点不是自动化执行，也不是外部平台适配，而是稳定沉淀 workflow 资产。
