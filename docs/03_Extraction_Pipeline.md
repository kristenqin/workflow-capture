# 03_Extraction_Pipeline

## 1. 文档目的

本文档定义 `workflow-capture` 从自然语言文本中抽取 Workflow 对象的处理流程。

核心目标是让系统能够从 GPT 回复、Markdown 文档、Agent 输出中识别隐式任务流程，并将其转换为符合 Schema 的 Workflow JSON。

一句话定义：

> Extraction Pipeline 是把“文本里的流程意图”转换成“可验证 Workflow 对象”的编译过程。

---

## 2. 总体流程

```text
Raw Text
  ↓
01 输入标准化
  ↓
02 文本分块
  ↓
03 流程候选识别
  ↓
04 Stage 抽取
  ↓
05 Stage 类型分类
  ↓
06 输入 / 动作 / 输出抽取
  ↓
07 Edge 关系推断
  ↓
08 Checkpoint 抽取
  ↓
09 Risk / Constraint 抽取
  ↓
10 Workflow Draft 生成
  ↓
11 Schema 校验
  ↓
12 Normalize 标准化
  ↓
13 渲染预览
  ↓
14 用户修正
  ↓
Validated Workflow Object
```

---

## 3. Pipeline 原则

### 3.1 不直接信任 LLM 输出

LLM 可以参与语义抽取，但不能直接决定最终对象。

最终结果必须经过：

- Schema 校验
- 字段标准化
- 引用关系校验
- 缺失项补全
- 置信度标记

### 3.2 先抽草稿，再标准化

不要要求模型一次性输出完美 Workflow。

推荐分两层：

```text
Extraction Draft → Validated Workflow Object
```

Draft 可以允许不完整，Validated Object 必须满足最低结构要求。

### 3.3 保留原文证据

每个重要对象都应该尽量保留 `sourceSpan` 或 `sourceText`，方便用户知道某个 Stage / Checkpoint 是从哪里抽出来的。

这可以减少 LLM 幻觉带来的不信任感。

### 3.4 优先线性，再识别复杂关系

大部分 GPT 回复都是线性流程。MVP 阶段默认把阶段关系识别为 sequence，只有在文本有明显条件、依赖、并行、反馈语义时，才生成更复杂的 Edge。

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

字段建议：

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
- 有序列表通常是强流程信号
- 无序列表可能是并列动作或注意事项
- 表格可能包含字段定义、阶段矩阵、验收项
- 代码块默认作为证据或示例，不作为 Stage 主来源

---

## 6. Step 03：流程候选识别

### 6.1 目标

判断哪些文本块可能包含 Workflow 或 Workflow Fragment。

### 6.2 强流程信号

```text
第一步 / 第二步 / Step 1 / Step 2
首先 / 然后 / 接着 / 最后
流程 / workflow / pipeline
阶段 / stage / phase
输入 / 输出 / 验收 / 风险
先……再……
```

### 6.3 弱流程信号

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
- possibleScope

---

## 7. Step 04：Stage 抽取

### 7.1 目标

从候选文本中抽取执行阶段。

### 7.2 抽取规则

以下内容可以成为 Stage：

- 明确编号步骤
- 明确阶段标题
- 有独立目标的段落
- 多个动作围绕同一意图展开的文本块

以下内容不应直接成为 Stage：

- 普通解释性句子
- 单独的注意事项
- 单独的字段定义
- 与执行无关的背景描述

### 7.3 Stage 粒度判断

一个 Stage 的合理粒度是：

```text
有明确意图 + 有动作 + 有阶段产出
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

## 8. Step 05：Stage 类型分类

### 8.1 目标

给每个 Stage 标记类型，方便后续渲染、筛选和执行分发。

### 8.2 类型映射规则

| 文本语义 | StageType |
|---|---|
| 分析、理解、识别、判断 | analysis |
| 计划、拆分、排期、安排 | planning |
| 设计、定义、建模、规格 | design |
| 实现、开发、编码、接入 | implementation |
| 审查、review、检查 | review |
| 测试、验证、验收 | testing |
| 交付、发布、导出、打包 | delivery |
| 维护、迭代、修复 | maintenance |

### 8.3 置信度

每个类型分类都应该有 confidence。

当无法判断时，使用：

```text
unknown
```

不要强行分类。

---

## 9. Step 06：输入 / 动作 / 输出抽取

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
拆分模块
定义字段
生成 Schema
校验结果
渲染预览
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

### 9.4 缺失处理

如果原文没有明确输入或输出，可以生成推断项，但必须标记：

```ts
inferred: true
```

---

## 10. Step 07：Edge 关系推断

### 10.1 默认规则

如果文本是有序列表，默认生成 sequence Edge。

```text
Stage 1 → Stage 2 → Stage 3
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
分别由不同 Agent 执行
```

### 10.5 反馈关系

出现以下表达时，生成 feedback：

```text
如果不通过则返回修改
审查后回到上一步
测试失败后重新实现
```

---

## 11. Step 08：Checkpoint 抽取

### 11.1 目标

识别阶段完成的判断标准。

### 11.2 触发词

```text
验收标准
检查
确认
通过条件
是否满足
必须保证
完成标志
```

### 11.3 Checkpoint 示例

原文：

```text
验收标准是每个阶段都要有输入、动作和输出。
```

抽取结果：

```json
{
  "description": "检查每个阶段是否具备输入、动作和输出",
  "passCondition": "所有 Stage 至少包含 actions，且 inputs / outputs 字段不存在结构性缺失"
}
```

---

## 12. Step 09：Risk / Constraint 抽取

### 12.1 Risk

Risk 表示可能影响结果质量的问题。

常见触发词：

```text
风险
问题
容易
避免
不要
注意
可能导致
```

### 12.2 Constraint

Constraint 表示必须遵守的限制。

常见示例：

```text
中文作为首要定义语言
不要一开始做自动执行
必须通过 Schema 校验
默认不要复杂 DAG
```

MVP 阶段 Risk 和 Constraint 可以先统一放入 `risks`，后续再拆分。

---

## 13. Step 10：Workflow Draft 生成

### 13.1 目标

生成一个可能不完整但结构清晰的 Workflow Draft。

### 13.2 Draft 允许的问题

- 缺少部分输入
- 缺少部分输出
- 阶段类型为 unknown
- Edge 只有 sequence
- Checkpoint 较少
- Risk 较少

### 13.3 Draft 不允许的问题

- 没有 stages
- Stage 没有 title
- Edge 引用不存在的 Stage
- 输出不是 JSON 对象
- 字段类型错误

---

## 14. Step 11：Schema 校验

### 14.1 目标

确保 Draft 符合 `04_Workflow_Schema.md` 定义。

### 14.2 校验内容

- 必填字段是否存在
- 枚举值是否合法
- Stage id 是否唯一
- Edge 引用是否有效
- Checkpoint 引用是否有效
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

把合法 Draft 整理成稳定 Workflow Object。

### 15.2 标准化规则

- 自动生成缺失 id
- 统一 Stage 编号
- 修正空白字符
- 动作改为动宾短语
- 默认 status 为 draft
- 默认 owner 为 unknown
- 默认 Edge type 为 sequence
- 补充 schemaVersion
- 补充 extractorVersion

---

## 16. Step 13：渲染预览

### 16.1 目标

将 Workflow Object 渲染给用户审查。

MVP 支持两种视图：

1. Linear Workflow View
2. Stage Card View

### 16.2 渲染重点

渲染结果应该突出：

- 阶段顺序
- 阶段意图
- 输入 / 动作 / 输出
- 验收点
- 风险提示
- 低置信度字段

低置信度字段应该显式标记，提醒用户修正。

---

## 17. Step 14：用户修正

### 17.1 目标

让用户把 LLM 抽取结果修正为可信 Workflow。

### 17.2 可编辑字段

MVP 至少支持编辑：

- Workflow title
- Workflow goal
- Stage title
- Stage intent
- Stage type
- Stage actions
- Stage inputs
- Stage outputs
- Checkpoint description
- Edge type

### 17.3 修正后处理

用户修正后，需要重新：

```text
Validate → Normalize → Render
```

---

## 18. LLM 抽取 Prompt 要求

LLM 抽取时应遵守：

1. 不要扩写成新方案
2. 优先忠实原文
3. 可以推断缺失输入输出，但必须标记 inferred
4. 不要把普通解释段落强行变成 Stage
5. 不要强行生成复杂 DAG
6. 不确定时标记 confidence 低
7. 输出必须是 JSON
8. 字段名使用英文
9. 内容语言优先保持原文语言

---

## 19. MVP 测试样例

### 19.1 简单线性流程

输入：

```text
先分析需求，然后设计对象模型，接着实现 MVP，最后测试验收。
```

期望：

- 4 个 Stage
- 3 条 sequence Edge
- StageType 分别接近 analysis / design / implementation / testing

### 19.2 带验收标准

输入：

```text
先抽取 Stage。验收标准是每个 Stage 都必须有标题和动作。
```

期望：

- 识别 1 个 Stage
- 识别 1 个 Checkpoint

### 19.3 带条件关系

输入：

```text
如果 Schema 校验失败，则返回修正；如果通过，则进入渲染。
```

期望：

- 识别 condition Edge
- 识别 feedback 或 condition 分支

---

## 20. 输出结果要求

Pipeline 最终必须输出：

```text
Validated Workflow Object
```

并附带：

```text
ValidationIssue[]
ExtractionReport
```

ExtractionReport 应包括：

- 总体置信度
- 识别到的 Stage 数量
- 识别到的 Edge 数量
- 低置信度字段
- 可能需要人工修正的问题

---

## 21. 第一版实现建议

MVP 可以采用以下实现方式：

```text
前端输入 Markdown
  ↓
调用 LLM 抽取 Draft JSON
  ↓
Zod 校验
  ↓
Normalize
  ↓
React 渲染 Stage Card
  ↓
用户编辑
  ↓
重新校验
  ↓
导出 JSON / Markdown
```

第一版重点不是自动化执行，而是稳定产出结构清晰的 Workflow Object。
