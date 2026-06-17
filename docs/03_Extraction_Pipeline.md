# 03_Extraction_Pipeline

## 1. 文档目的

本文档定义 `workflow-capture` 从自然语言文本中抽取 Workflow IR 的处理流程。

核心目标是让系统能够从 GPT 回复、Markdown 文档、Agent 输出中识别隐式任务流程，并将其转换为轻量、可校验、可渲染、可导出的 Workflow IR。

一句话定义：

> Extraction Pipeline 是把“文本里的流程意图”转换成“可适配外部工具的 Workflow IR”的编译过程。

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
04 TaskNode 抽取
  ↓
05 NodeKind 分类
  ↓
06 输入 / 动作 / 输出 / 验收抽取
  ↓
07 Relation 关系推断
  ↓
08 Automation Candidate Detection
  ↓
09 Artifact 抽取
  ↓
10 ExportTarget 建议
  ↓
11 Workflow IR Draft 生成
  ↓
12 Schema 校验
  ↓
13 Normalize 标准化
  ↓
14 渲染预览
  ↓
15 用户修正
  ↓
Validated Workflow IR
```

---

## 3. Pipeline 原则

### 3.1 不直接信任 LLM 输出

LLM 可以参与语义抽取，但不能直接决定最终 IR。

最终结果必须经过：

- Schema 校验
- 字段标准化
- 引用关系校验
- 缺失项补全
- 置信度标记
- 人工可修正渲染

### 3.2 先抽 Draft，再生成 Validated IR

不要要求模型一次性输出完美结果。

推荐分两层：

```text
Workflow IR Draft → Validated Workflow IR
```

Draft 可以允许不完整，Validated IR 必须满足最低结构要求。

### 3.3 保留原文证据

每个重要 TaskNode 都应该尽量保留 `sourceText`，方便用户知道它是从哪里抽出来的。

这可以减少 LLM 幻觉带来的不信任感。

### 3.4 优先线性，再识别复杂关系

大部分 GPT 回复都是线性流程。MVP 阶段默认把关系识别为 sequence，只有在文本有明显条件、依赖、并行、反馈语义时，才生成更复杂的 Relation。

### 3.5 自动化判断只是提示

Pipeline 可以判断某个节点是否适合自动化，但不负责执行。

执行交给外部工具。

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
- 有序列表通常是强流程信号
- 无序列表可能是并列动作、注意事项或输出项
- 表格可能包含节点矩阵、字段定义、导出目标
- 代码块默认作为证据或示例，不作为 TaskNode 主来源

---

## 6. Step 03：流程候选识别

### 6.1 目标

判断哪些文本块可能包含 Workflow Intent。

### 6.2 强流程信号

```text
第一步 / 第二步 / Step 1 / Step 2
首先 / 然后 / 接着 / 最后
流程 / workflow / pipeline
阶段 / stage / phase
输入 / 输出 / 验收 / 风险
先……再……
导出到……
交给……执行
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

## 7. Step 04：TaskNode 抽取

### 7.1 目标

从候选文本中抽取任务节点。

### 7.2 抽取规则

以下内容可以成为 TaskNode：

- 明确编号步骤
- 明确任务标题
- 有独立目标的段落
- 多个动作围绕同一意图展开的文本块
- 可以被映射到人工、LLM、脚本、HTTP、GitHub 等执行方向的任务

以下内容不应直接成为 TaskNode：

- 普通解释性句子
- 单独的背景描述
- 单独的字段定义
- 纯视觉修饰建议
- 与任务推进无关的表达

### 7.3 粒度判断

一个 TaskNode 的合理粒度是：

```text
有明确意图 + 有动作 + 可以被人类或外部工具处理
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

## 8. Step 05：NodeKind 分类

### 8.1 目标

给每个 TaskNode 标记 kind，方便后续 Export Adapter 判断适配方向。

### 8.2 类型映射规则

| 文本语义 | NodeKind |
|---|---|
| 人工判断、视觉审查、主观评估 | manual / review |
| 总结、改写、生成文档、分析文本 | llm |
| 调用接口、请求 URL、Webhook | http |
| 运行命令、执行测试、构建代码 | script |
| 创建 issue、PR、commit、检查仓库 | github |
| 读写文件、生成文件、整理目录 | file |
| 如果、否则、是否通过、条件判断 | decision |
| 无法判断 | unknown |

### 8.3 置信度

每个分类都应该有 confidence。

不确定时使用：

```text
unknown
```

不要强行分类。

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
生成 Schema
校验结果
导出 JSON
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

这些内容进入 TaskNode.acceptance，而不是独立 Checkpoint 对象。

### 9.5 缺失处理

如果原文没有明确输入或输出，可以谨慎推断，但必须降低 confidence，并尽量保留 sourceText。

---

## 10. Step 07：Relation 关系推断

### 10.1 默认规则

如果文本是有序列表，默认生成 sequence Relation。

```text
TaskNode 1 → TaskNode 2 → TaskNode 3
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

## 11. Step 08：Automation Candidate Detection

### 11.1 目标

判断 TaskNode 是否适合自动化，以及可能适配哪些外部工具。

### 11.2 自动化提示类型

| automation | 含义 |
|---|---|
| manual | 更适合人工处理 |
| automatable | 有明显自动化可能 |
| external | 应交给外部工具执行 |
| unknown | 无法判断 |

### 11.3 判断规则

| 节点特征 | automation | suggestedTargets |
|---|---|---|
| 主观审美、复杂判断、策略选择 | manual | agent_task_pack / markdown |
| 总结、抽取、改写、分类 | automatable | agent_task_pack / langgraph |
| HTTP 请求、Webhook、SaaS 集成 | external | n8n |
| 测试、构建、脚本、仓库事件 | external | github_actions |
| 长时间可靠任务、需要重试 | external | temporal |
| 条件分支、Agent 状态流转 | automatable | langgraph / n8n |

### 11.4 注意事项

- 不要因为一个节点可自动化，就假设本系统执行它
- 不确定时使用 unknown
- suggestedTargets 可以为空
- ExportTarget 是建议，不是承诺

---

## 12. Step 09：Artifact 抽取

### 12.1 目标

识别流程中涉及的输入材料、输出材料和中间产物。

### 12.2 常见 Artifact

```text
Markdown 文档
JSON Schema
YAML workflow
代码文件
GitHub Issue
Pull Request
测试报告
Agent Task Pack
```

### 12.3 处理规则

- 明确“输出/生成/导出”的内容通常是 output artifact
- 明确“基于/使用/读取”的内容通常是 input artifact
- 上一步产物被下一步使用时，可以标记为 intermediate

---

## 13. Step 10：ExportTarget 建议

### 13.1 目标

根据 TaskNode.kind、automation、Relation、Artifact，判断整个 IR 适合导出到哪些目标。

### 13.2 目标映射

| 目标 | 适合情况 |
|---|---|
| markdown | 所有流程都应支持 |
| json | 所有流程都应支持 |
| agent_task_pack | 包含 LLM、人工审查、文档任务 |
| n8n | 包含 http、decision、SaaS 集成 |
| github_actions | 包含 github、script、code、test |
| langgraph | 包含 llm、decision、feedback、agent 状态流 |
| temporal | 包含长流程、可靠执行、重试需求 |

### 13.3 支持状态

| 状态 | 含义 |
|---|---|
| supported | 当前版本可导出 |
| partial | 可部分导出 |
| planned | 计划支持 |
| not_suitable | 不适合 |

MVP 阶段建议：

```text
markdown: supported
json: supported
agent_task_pack: partial
n8n: planned
github_actions: planned
langgraph: planned
temporal: planned
```

---

## 14. Step 11：Workflow IR Draft 生成

### 14.1 目标

生成一个可能不完整但结构清晰的 Workflow IR Draft。

### 14.2 Draft 允许的问题

- 部分 TaskNode kind 为 unknown
- 部分 automation 为 unknown
- Relation 只有 sequence
- Artifact 较少
- ExportTarget 只有 json / markdown

### 14.3 Draft 不允许的问题

- 没有 nodes
- TaskNode 没有 title
- Relation 引用不存在的 TaskNode
- 输出不是 JSON 对象
- 字段类型错误

---

## 15. Step 12：Schema 校验

### 15.1 目标

确保 Draft 符合 `04_Workflow_Schema.md` 定义。

### 15.2 校验内容

- 必填字段是否存在
- 枚举值是否合法
- TaskNode id 是否唯一
- Relation 引用是否有效
- Artifact 引用是否有效
- 数组字段是否为数组
- confidence 是否在 0 到 1 之间

### 15.3 校验失败处理

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

## 16. Step 13：Normalize 标准化

### 16.1 目标

把合法 Draft 整理成稳定 Workflow IR。

### 16.2 标准化规则

- 自动生成缺失 id
- 统一节点编号
- 修正空白字符
- 动作改为动宾短语
- 默认 Relation type 为 sequence
- 默认 automation 为 unknown
- 默认 ExportTarget 包含 json / markdown
- 补充 schemaVersion
- 补充 extractorVersion

---

## 17. Step 14：渲染预览

### 17.1 目标

将 Workflow IR 渲染给用户审查。

MVP 支持两种视图：

1. Linear Workflow View
2. Task Card View

### 17.2 渲染重点

渲染结果应该突出：

- 节点顺序
- 节点动作
- 节点输入 / 输出
- 自动化提示
- 适配目标
- 低置信度字段

低置信度字段应该显式标记，提醒用户修正。

---

## 18. Step 15：用户修正

### 18.1 目标

让用户把 LLM 抽取结果修正为可信 IR。

### 18.2 可编辑字段

MVP 至少支持编辑：

- WorkflowIR title
- WorkflowIR goal
- TaskNode title
- TaskNode kind
- TaskNode actions
- TaskNode inputs
- TaskNode outputs
- TaskNode acceptance
- TaskNode automation
- Relation type
- ExportTarget type / status

### 18.3 修正后处理

用户修正后，需要重新：

```text
Validate → Normalize → Render
```

---

## 19. LLM 抽取 Prompt 要求

LLM 抽取时应遵守：

1. 不要扩写成新方案
2. 优先忠实原文
3. 不要把普通解释段落强行变成 TaskNode
4. 不要强行生成复杂 DAG
5. 不确定时标记 kind = unknown 或 automation = unknown
6. 输出必须是 JSON
7. 字段名使用英文
8. 内容语言优先保持原文语言
9. 自动化提示只是 hint，不要生成执行结果
10. ExportTarget 是建议，不是承诺可运行

---

## 20. MVP 测试样例

### 20.1 简单线性流程

输入：

```text
先分析需求，然后设计对象模型，接着实现 MVP，最后测试验收。
```

期望：

- 4 个 TaskNode
- 3 条 sequence Relation
- kind 分别接近 manual / llm / script / review 或 unknown

### 20.2 带自动化候选

输入：

```text
当 GitHub Issue 更新时，自动调用 Agent 分析 issue，然后创建实现任务。
```

期望：

- 识别 github / llm / github 或 file 类节点
- automation 为 external / automatable
- suggestedTargets 包含 github_actions 或 agent_task_pack

### 20.3 带条件关系

输入：

```text
如果 Schema 校验失败，则返回修正；如果通过，则进入渲染。
```

期望：

- 识别 decision 节点或 condition Relation
- 识别 feedback 关系

---

## 21. 输出结果要求

Pipeline 最终必须输出：

```text
Validated Workflow IR
```

并附带：

```text
ValidationIssue[]
ExtractionReport
```

ExtractionReport 应包括：

- 总体置信度
- TaskNode 数量
- Relation 数量
- Artifact 数量
- ExportTarget 建议
- 低置信度字段
- 可能需要人工修正的问题

---

## 22. 第一版实现建议

MVP 可以采用以下实现方式：

```text
前端输入 Markdown
  ↓
调用 LLM 抽取 IR Draft JSON
  ↓
Zod 校验
  ↓
Normalize
  ↓
React 渲染 Task Card
  ↓
用户编辑
  ↓
重新校验
  ↓
导出 JSON / Markdown / Agent Task Pack
```

第一版重点不是自动化执行，而是稳定产出结构清晰、可适配外部工具的 Workflow IR。
