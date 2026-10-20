# Agent 篇

### 你如何定义一个基于 LLM 的智能体（Agent）？它通常由哪些核心组件构成？

<strong>参考答案：</strong>
一个基于 LLM 的智能体（Agent）是一个能够自主理解环境、进行规划决策、并执行行动以达成特定目标的计算系统。其核心特征是利用一个<strong>大型语言模型（LLM）作为其“大脑”或“中央处理器”</strong>，来进行复杂的推理和决策。

与传统的调用LLM进行问答或文本生成不同，Agent具有<strong>自主性</strong>和<strong>循环执行</strong>的特点，它能主动地、持续地与环境或工具交互，直到完成任务。

一个典型的LLM Agent通常由以下<strong>四个核心组件</strong>构成：

1.  <strong>大脑/核心引擎 (Brain/Core Engine):</strong>
    * <strong>组件：</strong> 一个强大的大型语言模型（LLM），如GPT系列、Gemini、Llama等。
    * <strong>作用：</strong> 这是Agent的认知核心。它负责理解用户目标、感知环境信息、进行常识推理、制定计划、并决定下一步的行动。所有其他组件的输出最终都会汇集到LLM进行处理。

2.  <strong>规划模块 (Planning Module):</strong>
    * <strong>组件：</strong> 可以是LLM的内置能力（如通过CoT、ReAct等提示策略激发），也可以是独立的算法模块。
    * <strong>作用：</strong> 负责将一个复杂、长期的目标分解成一系列更小、更具体的、可执行的子任务。它还负责根据行动的反馈动态地调整 и修正计划。规划能力是Agent处理复杂任务的关键。

3.  <strong>记忆模块 (Memory Module):</strong>
    * <strong>组件：</strong> 通常是外部数据库或数据结构的组合，如向量数据库、键值存储等。
    * <strong>作用：</strong> 弥补LLM有限的上下文窗口。它分为：
      * <strong>短期记忆：</strong> 记录当前的对话历史、中间步骤的“思考过程”（scratchpad），用于维持任务的连贯性。
      * <strong>长期记忆：</strong> 存储过去的经验、知识、用户偏好等，通过检索（通常是RAG）来为当前决策提供信息。

4.  <strong>工具使用模块 (Tool Use Module):</strong>
    * <strong>组件：</strong> 一系列外部API、函数库或硬件接口。
    * <strong>作用：</strong> 扩展Agent的能力边界。LLM本身无法获取实时信息、执行数学计算或与物理世界交互。工具使用模块允许Agent调用外部工具来完成这些任务，例如：
      * <strong>信息获取：</strong> 调用搜索引擎、数据库查询API。
      * <strong>代码执行：</strong> 运行Python解释器、访问终端。
      * <strong>物理操作：</strong> 控制机器人手臂、调用智能家居API。



### 你怎么设计一个 Agent 系统，让它在接收到模糊多轮指令后，自主完成意图识别、任务分解、动态规划和执行，并在子任务失败时具备自适应调整能力？

#### 一、意图理解

我们没有做独立的 NLU 意图分类模块，而是把意图理解能力下沉到 LLM + 上下文注入，分两层实现：
**第一层：宏观意图 — PlanningAgent 理解"用户想干什么"**
用户输入一句模糊指令，比如"帮我用 hello world 技能打个招呼"，PlanningAgent 需要理解：

- 这是一个单步任务还是多步任务
- 应该派给哪个 agent
- 子任务描述怎么写才能让下游 agent 理解

实现方式是通过 system prompt 注入三类上下文：

1. Agent 合约（<available_agents>）— 告诉 LLM 有哪些 agent 可用及各自能力描述
2. 推理规则（<reasoning_rules>）— 要求 LLM 在 thinking 块中显式推理 6 个问题，包括"哪些 agent 能做"、"我是不是卡住了"
3. 结构化输出（PlanDecision）— 强制 LLM 输出 dispatches 列表而非自由文本，避免意图理解停留在模糊描述

关键点：意图理解不是一次性的。每轮 PlanningAgent 都会收到上一轮的 execution_history，如果发现之前的理解有误（比如派错了 agent、任务描述不清导致失败），可以在下一轮修正。

**第二层：微观意图 — ToolCallingAgent 理解"用什么工具完成"**
当 ToolCallingAgent 收到子任务后，需要进一步理解：用哪个 tool 或 skill 来执行。它的 prompt 注入了：

- 工具上下文（<tool_context>）— 所有可用工具的名称和参数 schema
- 技能上下文（<skill_context>）— 已加载技能的摘要（名称 + 描述 + 文件路径）
- 推理链（<reasoning_rules>）— 要求每步都做 evaluation → memory → next_goal → actions 的完整推理

比如用户说"打个招呼"，ToolCallingAgent 看到 <skill_context> 中有 hello-world 技能的描述匹配，就会输出 {"type": "skill", "name": "hello-world"}。

**面试官：如果 LLM 理解错了怎么办？**
靠闭环反馈修正，而不是靠前置的意图分类器：

- PlanningAgent 的 dispatch 描述写得不好 → ToolCallingAgent 执行失败 → 失败信息回传 → 下一轮 PlanningAgent 看到 FAIL 重新规划
- ToolCallingAgent 选错了工具 → 工具返回错误 → agent 自身的 evaluation_previous_goal 字段会标记失败 → 下一步换工具重试

Prompt 中有一条显式规则："Am I stuck? If repeating the same dispatch, consider an alternative."，这就是防止系统在错误意图上死循环。

>一句话总结： 意图理解 = 上下文注入（agent 合约 + 工具/技能描述）+ LLM 显式推理（thinking block）+ 结构化输出（强制承诺）+ 多轮闭环修正，没有硬编码的意图分类，让 LLM 在充分上下文下自己做判断。

#### 二、多问题分解

> 面试官：你的 Agent 系统是怎么处理一个包含多个子问题的复杂任务的？

回答：
我们用的是 LLM 驱动的动态多轮分解，而非硬编码的任务拆分。具体分三层：

**第一层：分解规则注入**
在 PlanningAgent 的 system prompt 中，我们定义了两条核心规则：

- Task Decomposition — 要求 LLM 把复杂任务拆成最小可执行单元，每个单元映射到一次 agent 调用
- Concurrency — 独立子任务放同一轮并发执行，有依赖的放不同轮顺序执行

这两条规则覆盖了"拆什么"和"怎么排"两个问题。

**第二层：结构化输出**
LLM 不是输出自由文本，而是返回一个强类型的 PlanDecision 结构体，其中 dispatches 字段是一个列表，每个元素包含 agent_name（谁来做）和 task（做什么）。这保证了分解结果是可执行的、可路由的。

比如用户说"分析市场趋势并生成报告"，LLM 会返回：

```json
{
  "dispatches": [
    {"agent_name": "data_agent", "task": "获取市场数据"},
    {"agent_name": "analysis_agent", "task": "分析趋势"}
  ]
}
```

第二轮再 dispatch reporter 生成报告——因为它依赖分析结果。

**第三层：闭环反馈调整**
分解不是一次性的。每轮执行完，所有子任务的结果（成功/失败/错误信息）会作为 execution_history 注入下一轮 prompt。PlanningAgent 看到失败结果后可以：

- 重试同一个 agent
- 换另一个 agent 完成
- 重新分解任务
- 判定整体完成

这个机制通过 prompt 中的 Result Review 规则和 thinking 推理链来驱动，代码层面不做硬编码重试。

>一句话总结： 多问题分解 = Prompt 规则驱动 LLM 拆任务 → 结构化 dispatch 并发/串行执行 → 结果回传闭环调整，核心是让 LLM 当调度器，代码只提供执行框架。



#### 三、任务规划

以 Codex、Manus、Claude 为例，任务规划的典型方式：

##### Codex（OpenAI）

**方式：沙箱 + 单步工具调用循环**

- 没有显式的"规划器"，靠 ReAct 循环：think → act → observe → think
- 每步模型自己决定下一步调用什么工具（bash/代码执行器），没有预生成完整计划
- 优点：简单直接，每步都有真实执行结果反馈
- 缺点：容易陷入局部循环，缺乏全局规划视角，复杂任务效率低



##### Manus

方式：显式 Plan-Execute 分离 + 虚拟文件系统驱动

- 规划阶段：先让 LLM 生成一份完整的 step-by-step plan（类似 todo list）
- 执行阶段：逐步执行 plan 中的任务，每步可调用浏览器、代码沙箱、文件系统等工具
- 动态调整：执行结果不符合预期时，修改剩余 plan 而非重头规划
- 核心特点：用文件系统（.md/.json）作为计划的持久化载体，plan 对外可见可追踪
- 优点：全局视野好，复杂任务拆解清晰
- 缺点：初始规划质量依赖模型能力，plan 本身出错会级联影响后续



##### Claude（Anthropic，特别是 Claude Code / computer use）

**方式：Extended Thinking + 隐式规划**

- 不生成显式 plan 文件，而是在 thinking 推理块中完成隐式的链式推理
- 通过 extended thinking（超长上下文推理）能力，在内部完成"理解 - 分解 - 排序 - 执行"
- 每步输出 thinking + tool_use，本质还是 ReAct，但 reasoning 深度远超普通 ReAct
- 多轮对话中靠 system prompt 注入对话历史 + 工具调用记录维持上下文连贯性

优点：

- reasoning 质量极高，不需要额外规划框架，推理过程细腻自然

缺点：

- 规划过程完全黑盒，用户无法在执行前审核或干预计划
- 超长任务链容易 context window 溢出丢失早期信息
- 无法像 Manus 那样拿到独立 plan 文件来审查修改

##### Devin（Cognition）

方式：Plan-Act-Verify 三阶段 + 持久化工作区

- Plan：接到任务后先输出一份 markdown 计划文件（和你的 plan.md 思路一致）
- Act：按计划逐步操作（编辑器、终端、浏览器），每步都是真实的 IDE 操作
- Verify：每步完成后自动验证（跑测试、检查编译），失败则回到 Plan 阶段修订
- 关键区别：Devin 的规划粒度是"IDE 操作级别"（改哪行代码、跑哪个命令），而 Manus 是"任务级别"

##### Cursor / Windsurf

**方式：Tab 式轻量规划 + 用户确认**

- 不做全自动规划，而是生成一个"建议步骤列表"让用户确认

- 用户可编辑 plan 后再执行，本质是 Human-in-the-loop planning

- 执行阶段是标准的 ReAct（think → edit → verify）

- 优点：可控性强，用户对每一步都有决定权

- 缺点：不适合全自动场景

  

##### OpenAI Operator / Deep Research

方式：多模型协作 + 渐进式规划

- Deep Research：先让模型生成研究计划（research plan），然后按计划并行搜索，每轮评估完整性，不完整则补充搜索——和你的 DeepResearcherTool 几乎一样
- Operator：浏览器操作任务，Plan 阶段生成步骤列表，执行阶段逐步操作 DOM，每步截图验证
- 共同点：都用 o3/o4-mini 级别的推理模型做规划，用轻量模型做执行



#### 行业趋势总结

```
规划模式演进路线：

ReAct（无显式规划）
  ↓
Plan-then-Execute（先规划后执行）
  ↓  
Plan-Execute-Verify（加验证闭环）
  ↓
Multi-Agent Plan（规划者和执行者分离）← 你的项目在这里
  ↓
Self-Evolving Plan（规划策略本身可优化）← 你的 SEPL 在这里
```

| 模式                | 代表      | 规划可见性        | 自动调整              | 多Agent | 自进化 |
| ------------------- | --------- | ----------------- | --------------------- | ------- | ------ |
| ReAct               | Codex     | 无                | 每步自适应            | 否      | 否     |
| Plan-Execute        | Manus     | plan 文件         | 修改剩余 plan         | 否      | 否     |
| Plan-Execute-Verify | Devin     | plan 文件         | 失败回退重规划        | 否      | 否     |
| Human-in-loop       | Cursor    | 用户确认          | 用户决策              | 否      | 否     |
| Multi-Agent Plan    | 你的项目  | plan.md           | 多轮 re-plan          | 是      | 否     |
| Self-Evolving       | 你的 SEPL | plan.md + lineage | propose-assess-commit | 是      | 是     |

你的项目相比业界的主要差异化在于两点：多 Agent 协作规划（PlanningAgent 只做决策、ToolCallingAgent 只做执行）和 SEPL 自进化机制（规划策略本身可以通过 propose-assess-commit 持续优化）。面试时这两个点是加分项。



### 请详细解释 ReAct 框架。它是如何将思维链和行动结合起来，以完成复杂任务的？

ReAct (Reason and Act) 是一个强大且基础的Agent行为框架，它通过一种巧妙的提示（Prompting）策略，让LLM能够协同地生成<strong>推理轨迹（reasoning traces）</strong>和<strong>任务相关的行动（actions）</strong>。

**核心思想：**
ReAct的核心思想是，人类在解决复杂问题时，并不仅仅是“思考”或“行动”，而是将两者紧密地交织在一起。我们会先思考一下，然后采取一个行动，观察结果，再根据结果进行思考，决定下一步行动。ReAct就是模仿人类这种“<strong>思考 -> 行动 -> 观察 -> 思考...</strong>”的循环模式。

<strong>工作流程：</strong>
ReAct通过一个精心设计的Prompt来引导LLM生成特定格式的文本。这个循环的每一步如下：

1.  <strong>思考 (Thought):</strong>
    * LLM首先分析当前的任务目标和已有的信息（观察）。
    * 然后，它会生成一段<strong>内心独白</strong>，即“思考”部分。这部分内容是LLM对当前情况的分析、策略的制定或对下一步行动的规划。例如：“我需要查找一下今天新加坡的天气。我应该使用搜索工具。”
    * 思考过程让Agent的行为变得可解释，并且有助于LLM自己进行复杂的规划和错误修正。

2.  <strong>行动 (Action):</strong>
    * 在“思考”之后，LLM会决定并生成一个具体的、可执行的“行动”。
    * 这个行动通常被格式化为 `Action: [Tool_Name, Tool_Input]` 的形式。例如：`Action: [Search, "weather in Singapore today"]`。
    * `Tool_Name` 是要调用的工具名称，`Tool_Input` 是传递给该工具的参数。

3.  <strong>观察 (Observation):</strong>
    * Agent的外部执行器（harness）会解析LLM生成的“行动”，并<strong>实际调用</strong>对应的工具。
    * 工具执行后返回的结果，被格式化为“观察”信息，并反馈给LLM。例如：`Observation: "Today in Singapore, the weather is sunny with a high of 32°C."`

<strong>循环与结合：</strong>
这个“观察”结果会作为新的上下文，与原始目标一起，输入到LLM中，开始下一轮的“思考 -> 行动 -> 观察”循环。

<strong>如何结合思维链（CoT）和行动？</strong>

* <strong>思维链 (Chain of Thought, CoT)</strong> 是一种让LLM通过生成中间推理步骤来解决复杂问题的方法。
* ReAct中的<strong>思考 (Thought)</strong>部分，本质上就是一种<strong>动态的、交互式的思维链</strong>。
* 传统的CoT是一次性生成所有思考步骤，然后得出答案。而ReAct的“思考”是<strong>每一步行动前</strong>都会进行的、<strong>基于最新观察结果</strong>的思维链。
* 这种结合使得Agent能够：
  * <strong>处理动态环境：</strong> 可以根据工具返回的最新信息实时调整策略。
  * <strong>进行错误修正：</strong> 如果一个行动失败或返回了无用的信息，Agent可以在下一步的“思考”中分析失败原因，并尝试不同的行动。
  * <strong>完成复杂任务：</strong> 通过将大任务分解成一系列“思考-行动”的子步骤，ReAct能够完成需要多步推理和工具交互的复杂任务。

> todo:paln 怎么做到步骤的错误修正



<strong>参考答案：</strong>
规划能力是衡量Agent智能水平的核心指标，它决定了Agent能否有效地将复杂目标分解为可执行步骤。目前，赋予LLM规划能力的主流方法，从简单到复杂，大致可以分为以下几个层次：

1.  <strong>基于提示的隐式规划 (Prompt-based Implicit Planning):</strong>
    * <strong>Chain of Thought (CoT):</strong> 这是最基础的规划方法。通过在提示中加入“Let's think step by step”，引导LLM生成一个线性的、一步接一步的思考过程。这个思考过程本身就是一种简单的计划。
      * <strong>优点：</strong> 实现简单，无需修改模型。
      * <strong>缺点：</strong> 规划是线性的，无法进行探索和回溯。一旦某一步出错，整个计划很可能失败。
    * <strong>ReAct 框架:</strong> ReAct将CoT与行动结合，使得规划成为一个动态过程。每一步的“思考”都是基于前一步“观察”的重新规划，比CoT更具鲁棒性。

2.  <strong>基于搜索的显式规划 (Search-based Explicit Planning):</strong>
    * 这类方法将规划问题形式化为一个搜索问题，通过探索不同的“思考”路径来寻找最优解。
    * <strong>Tree of Thoughts (ToT):</strong>
      * <strong>核心思想：</strong> ToT将规划过程构建为一棵“思维树”。从一个初始问题开始，LLM会生成多个不同的、并行的思考路径（树的分支）。
      * <strong>工作流程：</strong> 它采用标准的树搜索算法（如广度优先或深度优先搜索），在每一步都对当前的所有“思维节点”（叶子节点）进行评估（通常也由LLM自己打分），然后选择最有希望的节点进行下一步的扩展。
      * <strong>优点：</strong> 允许模型进行探索、评估和回溯，能解决需要深思熟虑或多路径探索的复杂问题。
      * <strong>缺点：</strong> 计算开销大，因为需要维护和评估一整棵树。

    * <strong>Graph of Thoughts (GoT):</strong>
      * <strong>核心思想：</strong> GoT是对ToT的进一步泛化。它认为思维过程不一定是树状的，而更可能是图状的。
      * <strong>工作流程：</strong> GoT允许不同的思维路径（分支）进行<strong>合并（Merge）</strong>，将多个子问题的解汇集起来形成一个更复杂的解。它还允许<strong>循环（Cycle）</strong>，使得思维过程可以迭代地优化和精炼。
      * <strong>优点：</strong> 提供了比树更灵活的思维结构，能够解决需要整合不同信息流或迭代改进的、更复杂的规划问题。
      * <strong>缺点：</strong> 结构和实现比ToT更复杂。

3.  <strong>基于任务分解的规划 (Task Decomposition Planning):</strong>
    * <strong>方法：</strong> 训练或提示LLM充当一个“规划器”，将主任务显式地分解成一个依赖图或一个步骤列表。然后，另一个“执行器”LLM（或同一个LLM扮演不同角色）再去逐一完成这些子任务。
    * <strong>优点：</strong> 结构清晰，易于管理和监控任务进度。
    * <strong>缺点：</strong> 对LLM的分解能力要求很高，且预先分解的计划可能缺乏对动态变化的适应性。



### 在 Agent 的设计中，“规划能力”至重要。请谈谈目前有哪些主流方法可以赋予 LLM 规划能力？（例如 CoT, ToT, GoT等）

规划能力是衡量Agent智能水平的核心指标，它决定了Agent能否有效地将复杂目标分解为可执行步骤。目前，赋予LLM规划能力的主流方法，从简单到复杂，大致可以分为以下几个层次：

1.  <strong>基于提示的隐式规划 (Prompt-based Implicit Planning):</strong>
    * <strong>Chain of Thought (CoT):</strong> 这是最基础的规划方法。通过在提示中加入“Let's think step by step”，引导LLM生成一个线性的、一步接一步的思考过程。这个思考过程本身就是一种简单的计划。
      * <strong>优点：</strong> 实现简单，无需修改模型。
      * <strong>缺点：</strong> 规划是线性的，无法进行探索和回溯。一旦某一步出错，整个计划很可能失败。
    * <strong>ReAct 框架:</strong> ReAct将CoT与行动结合，使得规划成为一个动态过程。每一步的“思考”都是基于前一步“观察”的重新规划，比CoT更具鲁棒性。

2.  <strong>基于搜索的显式规划 (Search-based Explicit Planning):</strong>
    * 这类方法将规划问题形式化为一个搜索问题，通过探索不同的“思考”路径来寻找最优解。
    * <strong>Tree of Thoughts (ToT):</strong>
      * <strong>核心思想：</strong> ToT将规划过程构建为一棵“思维树”。从一个初始问题开始，LLM会生成多个不同的、并行的思考路径（树的分支）。
      * <strong>工作流程：</strong> 它采用标准的树搜索算法（如广度优先或深度优先搜索），在每一步都对当前的所有“思维节点”（叶子节点）进行评估（通常也由LLM自己打分），然后选择最有希望的节点进行下一步的扩展。
      * <strong>优点：</strong> 允许模型进行探索、评估和回溯，能解决需要深思熟虑或多路径探索的复杂问题。
      * <strong>缺点：</strong> 计算开销大，因为需要维护和评估一整棵树。

    * <strong>Graph of Thoughts (GoT):</strong>
      * <strong>核心思想：</strong> GoT是对ToT的进一步泛化。它认为思维过程不一定是树状的，而更可能是图状的。
      * <strong>工作流程：</strong> GoT允许不同的思维路径（分支）进行<strong>合并（Merge）</strong>，将多个子问题的解汇集起来形成一个更复杂的解。它还允许<strong>循环（Cycle）</strong>，使得思维过程可以迭代地优化和精炼。
      * <strong>优点：</strong> 提供了比树更灵活的思维结构，能够解决需要整合不同信息流或迭代改进的、更复杂的规划问题。
      * <strong>缺点：</strong> 结构和实现比ToT更复杂。

3.  <strong>基于任务分解的规划 (Task Decomposition Planning):</strong>
    * <strong>方法：</strong> 训练或提示LLM充当一个“规划器”，将主任务显式地分解成一个依赖图或一个步骤列表。然后，另一个“执行器”LLM（或同一个LLM扮演不同角色）再去逐一完成这些子任务。
    * <strong>优点：</strong> 结构清晰，易于管理和监控任务进度。
    * <strong>缺点：</strong> 对LLM的分解能力要求很高，且预先分解的计划可能缺乏对动态变化的适应性。





### Tool Use是扩展 Agent 能力的有效途径。请解释 LLM 是如何学会调用外部 API 或工具的？（可以从 Function Calling 的角度解释）

LLM学会调用外部API或工具，是其从一个纯粹的“语言模型”转变为一个“行动执行者”的关键一步。这一能力的核心是让LLM能够<strong>理解何时需要使用工具</strong>，以及<strong>如何以结构化的方式表达使用哪个工具和传递什么参数</strong>。目前，主流的实现方式是<strong>Function Calling</strong>。

**Function Calling的工作原理如下：**

1. <strong>工具定义与注册 (Tool Definition & Registration):</strong>

   * 我们首先需要以一种机器可读的方式，向LLM“描述”我们有哪些可用的工具。这个描述通常是一个<strong>结构化的模式（Schema）</strong>，比如JSON Schema。
   * 对于每一个工具，我们需要定义：
     * <strong>函数名称 (Function Name):</strong> 例如，`get_current_weather`。
     * <strong>函数描述 (Function Description):</strong> 用自然语言清晰地描述这个函数的功能。例如，“获取指定城市的实时天气信息”。这个描述至关重要，因为LLM会根据它来判断何时使用该工具。
     * <strong>参数列表 (Parameters):</strong> 定义函数需要哪些输入参数，每个参数的名称、类型、和描述。例如，参数 `location` (string, "城市名") 和 `unit` (enum, "温度单位，可以是celsius或fahrenheit")。

2. <strong>LLM的决策与意图识别 (LLM's Decision & Intent Recognition):</strong>

   * 在与用户交互时，我们将用户的提问<strong>连同所有已注册的工具描述</strong>一起发送给LLM。
   * LLM（如GPT-4, Gemini等）经过了特殊的指令微调，使其能够理解这种“工具描述”的格式。
   * LLM会分析用户的意图。如果它认为只靠自身知识无法回答，且用户的意图与某个工具的功能相匹配，它就会决定调用该工具。

3. <strong>生成结构化的调用指令 (Generating Structured Calling Instructions):</strong>

   * 当LLM决定调用工具时，它的输出<strong>不再是自然语言文本</strong>，而是一个特殊格式的、结构化的<strong>JSON对象</strong>（或其他格式）。

   * 这个JSON对象会精确地包含：

     * <strong>要调用的函数名称</strong>。
     * <strong>一个包含所有参数名和值的对象</strong>。

   * 例如，对于用户提问“今天新加坡天气怎么样？”，LLM可能输出：

     ```json
     {
       "tool_call": {
         "name": "get_current_weather",
         "arguments": {
           "location": "Singapore",
           "unit": "celsius"
         }
       }
     }
     ```

4. <strong>外部执行与结果返回 (External Execution & Result Return):</strong>

   * Agent的控制代码（Orchestrator）会捕获这个特殊的JSON输出。
   * 它会解析JSON，找到函数名和参数，然后在<strong>外部环境中实际执行</strong>这个函数（例如，调用一个真实的天气API）。
   * 函数执行完毕后，会返回一个结果（例如，`{"temperature": 32, "condition": "sunny"}`）。

5. <strong>整合结果并生成最终回复 (Integrating Result & Generating Final Response):</strong>

   * 控制代码将工具的返回结果<strong>再次格式化</strong>，并将其作为新的上下文信息，连同之前的对话历史一起，再次发送给LLM。
   * 这一次，LLM已经获得了它需要的信息。它会基于这个结果，生成一个最终的、流畅的自然语言回答给用户，例如：“今天新加坡的天气是晴天，温度为32摄氏度。”



#### 真实示例

工具调用代码：

```python
import json
from langchain.tools import tool
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-4.1")

@tool
def get_weather(location: str) -> str:
    """获取某个位置的天气。"""
    return f"{location} 天气晴朗。"

# ===== 查看 @tool 自动生成的 Schema =====
print("=== @tool 生成的 args_schema (JSON Schema) ===")
print(json.dumps(get_weather.args_schema.model_json_schema(), indent=2, ensure_ascii=False))

# ===== 查看 bind_tools 转换后发给 LLM 的格式 =====
print("\n=== bind_tools 转换后的 OpenAI tools 格式 ===")
# 将 tool 转换为 OpenAI 格式的 tool dict
from langchain_core.utils.function_calling import convert_to_openai_tool
print(json.dumps(convert_to_openai_tool(get_weather), indent=2, ensure_ascii=False))

model_with_tools = model.bind_tools([get_weather])  # [!code highlight]

response = model_with_tools.invoke("波士顿的天气怎么样？")
for tool_call in response.tool_calls:
    # 查看模型发出的工具调用
    print(f"工具：{tool_call['name']}")
    print(f"参数：{tool_call['args']}")
```

在当前代码中，你看不到 JSON Schema——但它确实在背后生成了。整个流程分为两层：
第一层：@tool 装饰器 → 自动生成 Schema

```python
@tool
def get_weather(location: str) -> str:
    """获取某个位置的天气。"""
    return f"{location} 天气晴朗。"
```

**@tool 装饰器做了以下事情：**

| 从代码中提取                         | 生成的 Schema 字段                            |
| ------------------------------------ | --------------------------------------------- |
| 函数名 get_weather                   | function.name                                 |
| docstring """获取某个位置的天气。""" | function.description                          |
| 参数 location: str                   | parameters.properties.location (type: string) |
| 返回类型 -> str                      | 不发送给 LLM，仅内部使用）                    |

**第二层：bind_tools → 转换为 LLM 提供商格式**

```python
model_with_tools = model.bind_tools([get_weather])
```

bind_tools 会将上面的 Schema 转换为 OpenAI API 要求的格式，最终发送给 OpenAI 的请求体中会包含类似这样的 JSON：
```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "获取某个位置的天气。",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string"
            }
          },
          "required": ["location"]
        }
      }
    }
  ]
}
```

现在运行这个文件，你会看到两段 JSON 输出：
**输出 1：@tool 生成的原始 JSON Schema**

```json
=== @tool 生成的 args_schema (JSON Schema) ===
{
  "description": "获取某个位置的天气。",
  "properties": {
    "location": {
      "title": "Location",
      "type": "string"
    }
  },
  "required": [
    "location"
  ],
  "title": "get_weather",
  "type": "object"
}
```

这是 Pydantic 根据你的函数签名自动生成的标准 JSON Schema。

**输出 2：bind_tools 转换后的 OpenAI 格式**

```json
=== bind_tools 转换后的 OpenAI tools 格式 ===
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "获取某个位置的天气。",
    "parameters": {
      "properties": {
        "location": {
          "type": "string"
        }
      },
      "required": [
        "location"
      ],
      "type": "object"
    }
  }
}
```

**整个流程总结**

```
Python 函数                    @tool 装饰器                   bind_tools
─────────────                  ──────────                    ──────────
def get_weather(location: str)  →  Pydantic Model  →  OpenAI tools JSON  →  发送给 LLM
    """获取某个位置的天气。"""       (JSON Schema)      (function calling 格式)
```

核心要点：

- 函数名 → name
- docstring → description（LLM 靠它判断何时调用）
- 参数类型注解 → parameters（LLM 靠它知道要传什么参数）
- 这一切都是自动完成的，LangChain 帮你隐藏了 Schema 生成的复杂性

**当LLM决定调用工具时，它的输出一个特殊格式的、结构化的JSON对象**
![image-20260907233958742](Agent 篇.assets/image-20260907233958742.png)



### MCP协议

#### **一、先看问题：没有 MCP 之前世界是什么样的**

假设你是一个开发者，想让 AI 助手帮你做三件事：查天气、读数据库、发 Slack 消息。
没有 MCP 的时代：

```
                        ┌─ 天气 API ──── 你要写一套对接代码（REST 调用）
Claude Desktop ─────────┤─ 数据库 ────── 你要写一套对接代码（SQL 连接）
                        └─ Slack ─────── 你要写一套对接代码（Webhook）

                        ┌─ 天气 API ──── 又要重新写一套（格式不同！）
Cline (VS Code) ────────┤─ 数据库 ────── 又要重新写一套
                        └─ Slack ─────── 又要重新写一套

                        ┌─ 天气 API ──── 还得再写一套
你的自研 Agent ───────────┤─ 数据库 ────── 还得再写一套
                        └─ Slack ─────── 还得再写一套

总共 3 × 3 = 9 种对接方式，每种都是"烟囱式"的定制开发
```

每个 AI 客户端有自己的插件格式，每个工具有自己的 API 风格。这就是 N × M 集成困境。

有了 MCP 之后：

```
Claude Desktop ──┐
                 │
Cline ───────────┼── MCP ──→ MCP Servers ──→ Weather
                 │                    ├──────→ Database
Your Agent ──────┘                    └──────→ Slack


每个工具只开发 1 个 MCP Server
每个客户端只实现 1 个 MCP Client

总共 3 + 3 = 6，而且各自独立开发，互不影响
```

#### 二、MCP 协议到底定义了什么

MCP 协议本质上是一份合同规范，规定了客户端和服务端之间"怎么对话"。这份合同包含三个层面：

##### 层面 1：消息格式 — "用什么语言说话"

MCP 选择了 JSON-RPC 2.0 作为消息格式。每条消息都长这样：

```json
// 请求（客户端 → 服务端）
{
  "jsonrpc": "2.0",        // 固定版本号
  "method": "tools/list",  // 要做什么
  "id": 2                  // 请求编号，用于匹配响应
}

// 响应（服务端 → 客户端）
{
  "jsonrpc": "2.0",
  "result": { ... },       // 执行结果
  "id": 2                  // 对应请求的编号
}
```

这跟你在 client_handshake_demo.py 里打印出来的完全一致。

JSON-RPC 有三种消息类型：

| 类型         | 类型               | 例子                                    |
| ------------ | ------------------ | --------------------------------------- |
| Request      | 请求，期望对方回复 | tools/list（给我工具列表）              |
| Response     | 对 Request 的回复  | 返回工具列表的 JSON                     |
| Notification | 通知，不需要回复   | notifications/initialized（我准备好了） |

你在 demo 里也看到了 —— notifications/initialized 发送后没有等待响应，因为它是通知类型。

##### **层面 2：传输方式 — "用什么管道传输"**

MCP 定义了两种传输方式：
```
方式 A：SSE（Server-Sent Events）—— 远程通信
────────────────────────────────────────────
  客户端                              服务端
    │                                    │
    │── GET /sse ───────────────────────>│  建立 SSE 长连接（一直开着）
    │<── event: endpoint ───────────────│  服务端告诉客户端：发请求用这个URL
    │                                    │
    │── POST /messages/?session_id=xxx ─>│  客户端发 JSON-RPC 请求
    │   (HTTP 202, 空 body)              │  （POST 只是"投递"，不带结果）
    │                                    │
    │<── event: message ────────────────│  服务端通过 SSE 推送 JSON-RPC 响应
    │   (data: {"jsonrpc":"2.0",...})    │
    │                                    │


方式 B：stdio（标准输入/输出）—— 本地通信
────────────────────────────────────────────
  客户端（父进程）              服务端（子进程）
    │                              │
    │── stdin 写入 JSON-RPC ──────>│  客户端把请求写到子进程的标准输入
    │                              │
    │<── stdout 读出 JSON-RPC ────│  服务端把响应写到标准输出
    │                              │
```

SE 适合远程服务（比如你的 server.py 跑在某个服务器上），stdio 适合本地工具（比如 Claude Desktop 在本地启动一个 MCP Server 进程）。

##### 层面 3：协议方法 — "具体说哪些话"

MCP 协议定义了一组标准方法（就像 HTTP 定义了 GET、POST、PUT 一样）：
```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP 协议定义的标准方法                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  生命周期管理：                                                  │
│  ├── initialize          客户端说："我要连你，这是我的能力"        │
│  ├── initialized         客户端说："好的，握手完成"               │
│  └── ping                任意一方说："你还活着吗？"               │
│                                                                 │
│  Tools（工具）：                                                 │
│  ├── tools/list          客户端说："你有哪些工具？"               │
│  └── tools/call          客户端说："调用 get_weather 这个工具"    │
│                                                                 │
│  Resources（资源）：                                             │
│  ├── resources/list      客户端说："你有哪些资源？"               │
│  ├── resources/read      客户端说："读一下 weather://tokyo"       │
│  └── resources/templates/list  客户端说："有哪些资源模板？"        │
│                                                                 │
│  Prompts（提示词模板）：                                          │
│  ├── prompts/list        客户端说："你有哪些提示词模板？"          │
│  └── prompts/get         客户端说："给我 weather_report 模板"     │
│                                                                 │
│  Sampling（采样 — 服务端反过来请求客户端）：                       │
│  └── sampling/createMessage  服务端说："帮我问问 LLM 这个事"      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 三、用你的代码把协议"落地"

你的 server.py 只有 27 行，但它背后做的事情是：

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather Service")   # ① 创建 MCP Server，声明服务名

@mcp.tool()                         # ② 注册一个 Tool
def get_weather(location: str) -> str:
    """Get the current weather for a specified location."""
    return f"Weather in {location}: Sunny, 72°F"

@mcp.resource("weather://{location}")  # ③ 注册一个 Resource
def weather_resource(location: str) -> str:
    return f"Weather data for {location}: Sunny, 72°F"

@mcp.prompt()                       # ④ 注册一个 Prompt
def weather_report(location: str) -> str:
    return f"""You are a weather reporter..."""

mcp.run(transport="sse")            # ⑤ 启动 SSE 传输，监听端口 8000
```

FastMCP 框架在背后自动帮你做了这些事：

| 你写的代码      | 框架自动生成的 MCP 响应                                      |
| --------------- | ------------------------------------------------------------ |
| @mcp.tool()     | tools/list 返回 {"name": "get_weather", "inputSchema": {...}} |
| @mcp.resource() | resources/list 返回 {"uri": "weather://{location}", ...}     |
| @mcp.prompt()   | prompts/list 返回 {"name": "weather_report", ...}            |
| mcp.run()       | 启动 HTTP 服务器，挂载 /sse 和 /messages/ 路由               |

你在 demo 脚本中看到的 tools/list 响应，就是这个 Server 自动生成的：
```json
{
  "tools": [
    {
      "name": "get_weather",
      "description": "Get the current weather for a specified location.",
      "inputSchema": {
        "type": "object",
        "properties": {
          "location": { "type": "string" }
        },
        "required": ["location"]
      }
    }
  ]
}
```

AI 客户端（如 Cline）拿到这个 JSON 后就知道："哦，有个 get_weather 工具，需要传一个 location 字符串参数"。然后当用户问"新加坡天气怎么样"时，AI 就会生成一个 tools/call 请求：
```json
{
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "location": "Singapore" }
  }
}
```

#### 四、完整时序图：从连接到使用

把以上所有内容串起来，一次完整的 MCP 交互长这样：
```
Cline (MCP Client)                            你的 server.py (MCP Server)
      │                                              │
      │  ── 第一阶段：建立连接 ──                      │
      │                                              │
      │══ GET /sse ═════════════════════════════════>│  建立 SSE 长连接
      │<══ event: endpoint ══════════════════════════│  "/messages/?session_id=abc"
      │                                              │
      │  ── 第二阶段：握手 ──                          │
      │                                              │
      │── POST initialize ──────────────────────────>│  "我是 Cline，支持这些能力"
      │<══ SSE: initialize response ════════════════│  "我是 Weather Service，支持 tools/resources/prompts"
      │── POST initialized ─────────────────────────>│  "好的，握手完成"（通知，无响应）
      │                                              │
      │  ── 第三阶段：发现 ──                          │
      │                                              │
      │── POST tools/list ──────────────────────────>│  "你有哪些工具？"
      │<══ SSE: [{"name":"get_weather",...}] ═══════│  "有 get_weather，参数是 location"
      │── POST resources/list ──────────────────────>│  "你有哪些资源？"
      │<══ SSE: [{"uri":"weather://..."}] ══════════│  "有 weather://{location} 资源"
      │── POST prompts/list ────────────────────────>│  "你有哪些提示词？"
      │<══ SSE: [{"name":"weather_report"}] ════════│  "有 weather_report 提示词"
      │                                              │
      │  ── 第四阶段：使用 ──                          │
      │                                              │
      │── POST tools/call ──────────────────────────>│  "调用 get_weather，location=Singapore"
      │                                              │  → 执行你的 Python 函数
      │<══ SSE: "Sunny, 72°F" ══════════════════════│  ← 返回函数执行结果
      │                                              │
      │  ── 更多调用... ──                            │
      │                                              │
      │── POST tools/call ──────────────────────────>│  "调用 get_weather，location=Tokyo"
      │<══ SSE: "Sunny, 72°F" ══════════════════════│
```

#### 五、一句话总结

MCP = JSON-RPC 2.0（消息格式） + SSE/stdio（传输方式） + 标准方法集（工具/资源/提示词）
它不是某个框架或 SDK，而是一份协议规范 —— 只要你的程序能收发 JSON-RPC 消息、实现那几个标准方法，你就是一个合规的 MCP Server 或 MCP Client。就像任何能处理 HTTP 请求/响应的程序都是 Web 服务器一样。

#### 真实示例

```sh
D:\Anaconda\envs\python_all\python.exe D:\allevery\python_all\mcp\sdk_example\client_handshake_demo.py 
🔌 MCP 协议握手全过程演示
   目标服务: http://localhost:8000


############################################################
  步骤 0：连接 SSE 端点，获取消息发送地址
############################################################
  ✅ 收到消息端点: http://localhost:8000/messages/?session_id=30689645d59e487a919d5e46dcd74e1e
  ✅ SSE 监听线程已启动（复用同一连接/session）

############################################################
  步骤 1：initialize — 客户端告知自身能力，请求初始化
############################################################

============================================================
  →  客户端发送: initialize
============================================================
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {},
    "clientInfo": {
      "name": "HandshakeDemoClient",
      "version": "1.0.0"
    }
  },
  "id": 1
}
  (HTTP POST 返回: 202，空 body = 服务端已接收)

============================================================
  ←  服务端响应 (通过SSE推送): initialize
============================================================
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "experimental": {},
      "prompts": {
        "listChanged": false
      },
      "resources": {
        "subscribe": false,
        "listChanged": false
      },
      "tools": {
        "listChanged": false
      }
    },
    "serverInfo": {
      "name": "Weather Service",
      "version": "1.25.0"
    }
  }
}

############################################################
  步骤 2：initialized — 客户端确认初始化完成
############################################################

============================================================
  →  客户端发送: notifications/initialized
============================================================
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
  (HTTP POST 返回: 202，空 body = 服务端已接收)
  ← (通知，服务端不会返回响应)

############################################################
  步骤 3：tools/list — 🔑 客户端主动拉取服务端所有工具
############################################################

============================================================
  →  客户端发送: tools/list
============================================================
{
  "jsonrpc": "2.0",
  "method": "tools/list",
  "id": 2
}
  (HTTP POST 返回: 202，空 body = 服务端已接收)

============================================================
  ←  服务端响应 (通过SSE推送): tools/list
============================================================
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "Get the current weather for a specified location.",
        "inputSchema": {
          "properties": {
            "location": {
              "title": "Location",
              "type": "string"
            }
          },
          "required": [
            "location"
          ],
          "title": "get_weatherArguments",
          "type": "object"
        },
        "outputSchema": {
          "properties": {
            "result": {
              "title": "Result",
              "type": "string"
            }
          },
          "required": [
            "result"
          ],
          "title": "get_weatherOutput",
          "type": "object"
        }
      }
    ]
  }
}

############################################################
  步骤 4：resources/list — 客户端主动拉取服务端所有资源
############################################################

============================================================
  →  客户端发送: resources/list
============================================================
{
  "jsonrpc": "2.0",
  "method": "resources/list",
  "id": 3
}
  (HTTP POST 返回: 202，空 body = 服务端已接收)

============================================================
  ←  服务端响应 (通过SSE推送): resources/list
============================================================
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resources": []
  }
}

############################################################
  步骤 5：prompts/list — 客户端主动拉取服务端所有提示词
############################################################

============================================================
  →  客户端发送: prompts/list
============================================================
{
  "jsonrpc": "2.0",
  "method": "prompts/list",
  "id": 4
}
  (HTTP POST 返回: 202，空 body = 服务端已接收)

============================================================
  ←  服务端响应 (通过SSE推送): prompts/list
============================================================
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "prompts": [
      {
        "name": "weather_report",
        "description": "Create a weather report prompt.",
        "arguments": [
          {
            "name": "location",
            "required": true
          }
        ]
      }
    ]
  }
}

############################################################
  步骤 6：tools/call — 客户端实际调用 get_weather 工具
############################################################

============================================================
  →  客户端发送: tools/call
============================================================
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "Singapore"
    }
  },
  "id": 5
}
  (HTTP POST 返回: 202，空 body = 服务端已接收)

============================================================
  ←  服务端响应 (通过SSE推送): tools/call
============================================================
{
  "jsonrpc": "2.0",
  "id": 5,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Weather in Singapore: Sunny, 72°F"
      }
    ],
    "structuredContent": {
      "result": "Weather in Singapore: Sunny, 72°F"
    },
    "isError": false
  }
}


============================================================
  ✅ 握手全过程完成！
============================================================

  核心要点：
  ┌──────────────┐                              ┌───────────┐
  │    Client     │                              │ MCP Server│
  │   (Cline)    │                              │(server.py)│
  └──────┬───────┘                              └─────┬─────┘
         │                                            │
         │═══ GET /sse (建立SSE长连接) ═══════════════>│
         │<══ SSE event "endpoint" ═══════════════════│  服务端告知POST地址
         │    (含 session_id，后续必须用同一个)          │
         │                                            │
         │─── POST /messages/?session_id=xxx ─────────>│  客户端发请求
         │    (HTTP 202 空 body，仅表示"已接收")         │
         │                                            │
         │<══ SSE event "message" (JSON-RPC) ════════│  服务端通过同一SSE连接推送响应
         │                                            │
    

Process finished with exit code 0

```



### Skill是什么

> 我理解的 Skill，是给 Agent 使用的、可复用的任务技能包。它把某个领域的操作步骤、业务规范和相关资源组织起来，让 Agent 遇到同类任务时，可以按照已有经验执行。
>
> 以常见的 Agent Skills 格式为例，核心是一个 `SKILL.md` 文件，里面写明技能的名称、适用场景和具体操作指令，还可以附带脚本、参考文档和模板。
>
> 它和 Tool 的区别是：**Tool 提供可调用的操作，Skill 指导 Agent 在什么场景下、按什么步骤使用这些能力。** 比如运行测试是一个工具操作，而代码修复 Skill 可以规定：先复现问题，再定位原因、修改代码，最后运行测试验证。
>
> 它的一个关键机制是**按需加载，也叫渐进式披露**。Agent 先看到技能的名称和描述，判断是否适用；匹配任务后，再加载完整指令，执行过程中按需读取附带的资料或运行脚本。这样不需要一开始就把所有领域知识塞进上下文。
>
> 所以，Skill 的核心价值是**把团队经验变成可复用、可维护的操作规范，同时控制上下文开销，减少 Agent 每次从零摸索的成本。**

上面关于文件结构与按需加载的描述，对应 [Agent Skills 官方说明](https://agentskills.io/home)。

**如果追问：“自动编码 Skill 怎么工作？”**

你就沿用同一个例子，不用再背一套架构：

> 一个自动编码 Skill，可以指导 Agent 按照“理解需求和仓库规范 → 修改代码 → 运行编译或测试 → 根据反馈修复 → 交付结果”的流程工作。
>
> **Skill 提供操作规范，Agent 负责执行和决策，Tool 负责实际读写文件、运行命令。** 验证失败后可以继续修复，但需要设置重试次数或成本上限，无法解决时报告阻塞原因，不能无限循环，也不能把“编译通过”等同于“业务正确”。

记住这句，回答就不容易散：**Agent 负责决策和执行，Tool 提供操作能力，Skill 沉淀完成任务的方法。**



#### Skill 深度原理解析：从 Function Calling 到 Skill Context Protocol

##### 一、Tool vs Skill：两种扩展 Agent 能力的范式

在理解 Skill 之前，需要先理解它与 Tool 的本质区别。本项目同时实现了两套协议，形成了一种双轨制的能力扩展体系：

| 维度        | Tool (TCP)                           | Skill (SCP)                                     |
| ----------- | ------------------------------------ | ----------------------------------------------- |
| 定义方式    | Python 类（继承 Tool）               | Markdown 包（SKILL.md + scripts/ + resources/） |
| Schema 生成 | 自动从类签名生成 JSON Schema         | 从 YAML frontmatter 解析元数据                  |
| 注册方式    | @TOOL.register_module 装饰器         | SCP Server 扫描目录或运行时注册                 |
| 告知 LLM    | Function Calling 格式（结构化 JSON） | Prompt Injection（XML 标签包裹的文本摘要）      |
| 执行机制    | 定性代码执行（Python __call__）      | LLM 解释执行（SKILL.md 内容 + 用户参数 → LLM）  |
| 本质        | 刚性工具：代码即逻辑                 | 柔性知识包：Markdown 即指令                     |

核心洞察：Tool 是给机器执行的，Skill 是给 LLM 阅读的。

##### 二、Skill 的完整生命周期

###### 2.1 Skill 的定义：以 Markdown 为中心的知识包

一个 Skill 是一个自包含的目录，其结构如下（以 hello_world 为例）：
```
hello_world/
├── SKILL.md                    # 核心：YAML frontmatter + 完整的Markdown指令
├── scripts/
│   └── hello.py                # 可选：可执行的Python脚本
├── resources/
│   └── greetings.json          # 可选：数据/配置文件
├── examples.md                 # 可选：用法示例文档
└── reference.md                # 可选：API参考文档
```

SKILL.md 是 Skill 的灵魂，其结构为：

```markdown
---
name: hello-world
description: A demonstration skill that greets users...
---

# Hello World Skill
## Quick Start
## Instructions
## Workflow
## Examples
...
```

- YAML frontmatter（--- 之间的部分）：提供机器可读的元数据（名称、描述、版本）
- Markdown body：提供 LLM 可读的完整指令（工作流、步骤、条件逻辑、模板等）

###### 2.2 Skill 的发现与加载

SkillContextManager.initialize() 的完整流程：
```
阶段1: 扫描 default_skills/ 目录 → 发现包含 SKILL.md 的子目录
阶段2: 从 workdir/skill/skill.json 加载已持久化的历史技能
阶段3: 合并两个来源（版本号更高者优先）
阶段4: 按 skill_names 过滤（可选）
阶段5: 为每个技能构建文本摘要 → 注册版本 → 存入内存
阶段6: 持久化到 skill.json + contract.md
```

关键代码在 _parse_skill_dir()，它从目录中解析出 SkillConfig：
```python
# 解析 YAML frontmatter → 提取 name, description, version
frontmatter, body = self._parse_frontmatter(raw)

# 扫描子目录 → 收集 scripts, resources, reference_files
scripts = [str(p) for p in sorted(scripts_dir.rglob("*")) if p.is_file()]
resources = [str(p) for p in sorted(resources_dir.rglob("*")) if p.is_file()]
```

###### 2.3 Skill 的"Schema"生成：文本摘要而非 JSON Schema

这是 Skill 与 Tool 最根本的区别。Tool 通过 dynamic_manager.build_function_calling() 生成标准的 JSON Schema 发给 LLM，而 Skill 通过 _build_text_representation() 生成一段纯文本摘要：
```python
@staticmethod
def _build_text_representation(skill_config: SkillConfig) -> str:
    parts = [
        f"Skill: {skill_config.name}",
        f"Description: {skill_config.description}",
        f"Version: {skill_config.version}",
        f"Skill Directory: {skill_config.skill_dir}",
        f"SKILL.md: {os.path.join(skill_config.skill_dir, 'SKILL.md')}",
    ]
    if skill_config.scripts:
        parts.append("Scripts:")
        for s in skill_config.scripts:
            parts.append(f"  - {s}")
    # ... resources, references
    return "\n".join(parts)
```

生成结果示例：
```markdown
Skill: hello-world
Description: A demonstration skill that greets users...
Version: 1.0.0
Skill Directory: D:\xx\DeepResearchAgent\src\skill\default_skills\hello_world
SKILL.md: D:\xx\DeepResearchAgent\src\skill\default_skills\hello_world\SKILL.md
Scripts:
  - D:\xx\DeepResearchAgent\src\skill\default_skills\hello_world\scripts\hello.py
Resources:
  - D:\xx\DeepResearchAgent\src\skill\default_skills\hello_world\resources\greetings.json
```

注意：这里不包含完整的 SKILL.md 内容，只是一个"目录卡片"。完整内容在执行时才被加载。

##### 三、Skill 的运行时执行：两阶段 LLM 调用

###### 3.1 第一阶段：Prompt Injection — 让 LLM 知道有哪些 Skill

在 ToolCallingAgent._get_messages() 中，Skill 上下文作为动态模块注入到 prompt 模板中：
```python
# 4. 注入技能上下文：已加载技能的元数据摘要
agent_message_modules.update(await self._get_skill_context(ctx=ctx))
```

_get_skill_context() 通过 SCP 获取所有技能的摘要，包裹在 XML 标签中：

```
skill_content = await scp.get_context()
# 输出格式：
# <skill name="hello-world">
# Skill: hello-world
# Description: ...
# </skill>
skill_context = f"<skill_context>\n{skill_content}\n</skill_context>"
```

这个 skill_context 被填充到 AGENT_MESSAGE_PROMPT_TEMPLATE 的 {{ skill_context }} 占位符中，最终成为发给 LLM 的 user message 的一部分。
同时，系统 prompt 中包含 SKILL_CONTEXT_RULES，告诉 LLM 如何使用技能：

```
- <skill_context> only contains a brief summary (name, description, file paths)
- When a task matches a skill's description, directly use action type="skill"
- The system will automatically load and interpret the SKILL.md content
- Skill scripts can be executed via tools (e.g., bash or python_interpreter)
```

###### 3.2 LLM 的决策：选择调用 Skill

LLM 在输出中通过 type: "skill" 来表达使用技能的意图。这是 OUTPUT 中定义的 action 格式：
```json
{
    "actions": [
        {
            "type": "skill",
            "name": "hello-world",
            "args": "{\"name\": \"Alice\"}"
        }
    ]
}
```

对比 Function Calling 中的 tool_call：
```json
{
    "tool_call": {
        "name": "get_current_weather",
        "arguments": {"location": "Singapore"}
    }
}
```

形式上高度一致——都是 "名称 + 参数" 的结构化调用指令。区别在于路由目标不同。

###### 3.3 第二阶段：Skill 执行 — LLM 即解释器

在 ToolCallingAgent._think_and_tool() 中，根据 action_type 路由到不同的处理器：
```python
if action_type == "skill":
    # 路由到 SCP（Skill Context Protocol）
    response = await scp(name=action_name, input=action_args, ctx=ctx)
else:
    # 路由到 TCP（Tool Context Protocol）
    tool_response = await tcp(name=action_name, input=action_args, ctx=ctx)
```

SCP 的执行逻辑（SkillContextManager.call()）是 Skill 系统最精妙的部分——它发起第二次 LLM 调用：
```python
# 构建系统提示：告诉 LLM 它是一个"技能执行引擎"
system_content = (
    "You are a skill execution engine. You are given a skill's full instructions "
    "(from its SKILL.md) and user-provided arguments. Your job is to:\n"
    "1. Read and understand the skill instructions.\n"
    "2. Follow the workflow described in the skill.\n"
    "3. Apply the user arguments to generate the appropriate output.\n"
    "4. Return ONLY the final result."
)

# 构建用户提示：注入完整的 SKILL.md 内容 + 用户参数
user_content = (
    f"## Skill Instructions (from SKILL.md)\n\n"
    f"{skill_config.content}\n\n"
    f"## User Arguments\n\n"
    f"```json\n{json.dumps(input, ensure_ascii=False, indent=2)}\n```\n\n"
    f"Execute this skill and return the result."
)
```

第二次 LLM 调用
```python
llm_response = await model_manager(model=effective_model, messages=messages)
```

**核心机制**：Skill 的执行不是运行代码，而是**让另一个 LLM 阅读 SKILL.md 并按指令生成结果**。

##### 核心洞察：Skill 的本质

1. **Skill 是 "Prompt as Code"**：Tool 的执行逻辑写在 Python 代码里，Skill 的执行逻辑写在 Markdown 里。SKILL.md 本质上是一段**可被 LLM 解释执行的程序**。

2. **两层 LLM 调用架构**：第一层 LLM（Agent）负责**决策**——判断何时使用哪个技能；第二层 LLM（Skill Executor）负责**执行**——阅读 SKILL.md 并按指令生成结果。这种分离使得 Skill 可以独立于 Agent 进行迭代优化。

3. **版本化与可训练性**：`require_grad=True` 的 Skill 可以通过 TextGrad 等优化器自动改进 SKILL.md 的内容。每次修改自动创建新版本（1.0.0 → 1.0.1），支持回滚。这使得 Skill 不仅是知识包，更是**可微分的知识单元**。

4. **与 Function Calling 的统一接口**：尽管内部机制完全不同，但在 Agent 的输出层面，Tool 和 Skill 使用统一的 `actions` 格式，只是 `type` 字段不同。这种设计让 LLM 不需要关心底层执行机制的差异。

- Tool = 代码 + JSON Schema + 确定性执行 
- Skill = Markdown + 文本摘要 + LLM 解释执行
- Agent 的决策 = f(tool_context, skill_context, task) 
- → type="tool" → TCP → Python call() → 确定结果 
- → type="skill" → SCP → LLM(SKILL.md + args) → 生成结果

>Skill 系统是对 Function Calling 思想的**高层抽象扩展**：Function Calling 让 LLM 能调用函数，Skill 让 LLM 能调用**另一个 LLM 按 Markdown 指令执行任务**——这是 "LLM as Tool" 的终极形态。



### A2A框架和普通Agent框架的区别在哪，挑一个最关键的不同点说明。

A2A（Agent-to-Agent）框架或协议代表了多智能体系统研究中的一个重要方向。

<strong>和普通Agent框架的区别：</strong>
一个普通的Agent框架，如LangChain或Auto-GPT，其核心关注点是<strong>单个Agent的内部工作循环和能力</strong>。它定义了一个Agent如何<strong>感知环境、进行规划（思考）、调用工具（行动）、并处理反馈（观察）</strong>。它的设计蓝图是围绕着一个独立的、自主的个体。

而A2A框架的核心关注点则完全不同，它关注的是<strong>多个异构Agent之间的通信和协作</strong>。它试图定义一套<strong>通用的标准、协议和语言</strong>，使得由不同开发者、使用不同技术栈、为了不同目标而构建的Agent们，能够相互发现、理解和交互。

<strong>最关键的不同点：</strong>

<strong>普通Agent框架关注的是“个体的实现”（Implementation of an individual），而A2A框架关注的是“群体的交互标准”（Interaction standard for a collective）。</strong>

* <strong>举例来说：</strong>
  * <strong>LangChain</strong>告诉你如何用Python代码构建一个能使用Google搜索和计算器的Agent。它关心的是这个Agent内部的逻辑流（`AgentExecutor`, `Chains`, `Tools`）。
  * 一个<strong>A2A框架</strong>则试图回答这样的问题：“我的LangChain Agent如何向一个完全不认识的、由别人用Java写的Agent有效地传达一个任务：‘帮我用你的专业金融数据库分析一下这只股票，并把结果以JSON格式返回给我？’”
  * 它需要定义消息的格式、能力的描述方式（如何声明自己会用什么工具）、任务的分解和委托协议、以及信任和验证机制。

所以，最关键的不同点在于<strong>抽象层次</strong>。普通Agent框架在“<strong>应用层</strong>”，致力于构建能干活的个体；而A2A框架在“<strong>协议层</strong>”，致力于构建一个能让所有个体互相交流的“社会规则”或“互联网协议”。A2A是实现真正复杂的、去中心化的多智能体协作的必要基础。

### <u>你用过哪些Agent框架？选型是如何选的？你最终场景的评价指标是什么？</u>

是的，我在多个项目中实践过不同的Agent框架。我最常用的主要有两个：<strong>LangChain</strong> 和 <strong>LlamaIndex</strong>，偶尔也会使用更轻量级的库如 <strong>AutoGen</strong> 进行多智能体实验。

<strong>选型是如何选的？</strong>
我的选型过程主要基于项目的<strong>核心需求</strong>，我通常会从“<strong>逻辑编排驱动</strong>”还是“<strong>数据驱动</strong>”这两个角度来考虑：

1.  <strong>当项目是“逻辑编排驱动”时，我首选LangChain。</strong>
    * <strong>场景：</strong> 这类项目的核心是构建一个复杂的、需要执行一系列步骤、并与多种外部工具（APIs, 数据库, 文件系统）交互的Agent。例如，一个自动化的研究助手，需要先上网搜索，然后对结果进行总结，再用代码执行器进行数据分析。
    * <strong>选择理由：</strong> LangChain提供了非常强大和灵活的<strong>Agent Executor</strong>和<strong>Chains</strong>（特别是LCEL表达式语言），能够很好地编排和控制复杂的执行流。它的工具集成生态也是最丰富的。

2.  <strong>当项目是“数据驱动”时，我首选LlamaIndex。</strong>
    * <strong>场景：</strong> 这类项目的核心是构建一个围绕特定知识库的问答或分析系统，即高级RAG（Retrieval-Augmented Generation）。例如，一个能回答公司内部上千份PDF技术文档的客服机器人。
    * <strong>选择理由：</strong> LlamaIndex在<strong>数据的摄入、索引、和检索</strong>方面做得比LangChain更深入、更专业。它提供了更多样化和高级的索引结构（如树索引、知识图谱索引）和检索策略（如混合检索、重排序），对于优化RAG的质量至关重要。

<strong>最终场景的评价指标是什么？</strong>
评价指标是高度依赖于具体场景的，但我通常会从以下三个维度来综合评估一个Agent的性能：

1.  <strong>任务成功率 (Task Success Rate):</strong>
    * <strong>定义：</strong> 这是最重要的结果导向指标。它衡量Agent在多大比例上成功地、完整地完成了最终任务。
    * <strong>举例：</strong> 对于一个代码生成Agent，能否生成无语法错误且能通过所有单元测试的代码。对于一个问答Agent，答案的准确率和完整性。

2.  <strong>过程效率 (Process Efficiency):</strong>
    * <strong>定义：</strong> 衡量Agent在完成任务过程中的资源消耗。
    * <strong>举例：</strong>
      * <strong>成本 (Cost):</strong> 完成一次任务的总Token消耗量或API调用费用。
      * <strong>延迟 (Latency):</strong> 从用户发出指令到Agent给出最终结果的总耗时。
      * <strong>步骤数 (Number of Steps):</strong> Agent执行的“思考-行动”循环次数。次数越少通常意味着规划能力越强。

3.  <strong>鲁棒性与可预测性 (Robustness & Predictability):</strong>
    * <strong>定义：</strong> 衡量Agent在面对非理想情况（如工具报错、模糊指令、环境变化）时的表现。
    * <strong>举例：</strong>
      * <strong>错误处理能力：</strong> 当一个API调用失败时，Agent能否识别错误并尝试备用方案。
      * <strong>一致性：</strong> 对于相似的输入，Agent能否产生相似的、可预测的输出。
      * <strong>安全评估：</strong> 在红队测试中，Agent抵抗提示注入等攻击的能力。

### 有微调过Agent能力吗？数据集如何收集？

是的，我对通过微调来提升Agent特定能力的实践有所了解和尝试。单纯依靠提示（Prompting）来驱动的Agent（zero-shot Agent）在复杂或特定领域的任务上，其稳定性和效率往往不够理想。微调是让Agent变得更可靠、更高效的关键步骤。

微调Agent能力的核心是<strong>教会模型如何更好地“思考”和“使用工具”</strong>，本质上是一种<strong>行为克隆（Behavioral Cloning）</strong>。

<strong>数据集如何收集？</strong>
Agent微调的数据集不是简单的（输入，输出）对，而是一系列高质量的 <strong>“决策轨迹”（decision-making trajectories）</strong>。收集这类数据集主要有以下几种方法：

1.  <strong>使用强大的“教师模型”生成合成数据 :</strong>
    * <strong>流程：</strong> 这是目前最主流和高效的方法。
      1.  <strong>定义任务和工具：</strong> 首先明确Agent需要完成的任务和可用的工具集。
      2.  <strong>编写任务样本：</strong> 创建一系列该任务的实例（prompts）。
      3.  <strong>使用教师模型生成轨迹：</strong> 利用一个非常强大的闭源模型（如GPT-4o）作为“教师”，让它在ReAct或其他Agent框架下执行这些任务。
      4.  <strong>记录完整轨迹：</strong> 详细记录下教师模型每一步的“思考（Thought）”和“行动（Action）”。这个（任务, 思考, 行动）序列就是我们的一条数据。
      5.  <strong>过滤和清洗：</strong> 自动或人工地筛选掉那些教师模型执行失败或质量不高的轨迹，确保数据集的质量。

2.  <strong>人工编写或修正轨迹:</strong>

3.  <strong>从真实用户交互中收集数据 </strong>



## 智能体框架

智能体框架主要帮助开发者解决模型调用、工具使用、任务编排、状态管理和多个 Agent 协作等问题。不同框架的区别，主要在于它们如何组织这些能力。

本项目采用 **Planner + Bus + 子 Agent** 的架构。Planner 根据用户目标和当前状态制定下一步计划，通过 Bus 将任务分发给子 Agent。子 Agent 执行后返回结果，Planner 再根据反馈调整计划，直到任务完成或触发终止条件。整个过程形成“读取状态 → 制定决策 → 执行任务 → 接收反馈”的循环。

| 维度     | 本项目                                 | AutoGen                       | CrewAI                            | LangGraph                      |
| -------- | -------------------------------------- | ----------------------------- | --------------------------------- | ------------------------------ |
| 组织方式 | Planner 统一调度子 Agent               | 通过消息和对话组织 Agent 协作 | 围绕角色、任务和团队组织协作      | 通过节点、边和状态描述执行流程 |
| 通信方式 | 通过 Bus 传递任务和反馈                | 消息传递，底层支持事件驱动    | 任务上下文传递与 Flow 状态管理    | 状态更新与节点间数据传递       |
| 执行控制 | Planner 输出结构化决策，程序执行调度   | 通过团队策略或自定义逻辑控制  | 通过任务流程、管理者和 Flows 控制 | 通过边、条件路由和节点逻辑控制 |
| 并发方式 | 使用 `asyncio.gather` 执行独立异步任务 | 取决于运行时和协作方式        | 支持异步任务及流程编排            | 支持并行分支与结果合并         |

这些区别体现的是框架的设计重点，而不是严格的能力边界。AutoGen 除了对话式协作，还提供事件驱动的运行基础；CrewAI 除了顺序任务，也提供层级协作和 Flows；LangGraph 则允许开发者更明确地控制执行路径和状态变化。[AutoGen 文档](https://microsoft.github.io/autogen/stable/)、[CrewAI 文档](https://docs.crewai.com/)、[LangGraph 文档](https://docs.langchain.com/oss/python/langgraph/overview)

本项目的特点是将调度决策集中在 Planner 中，并通过 `PlanDecision` 约束决策格式，使程序能够解析和校验下一步操作。Bus 负责传递消息，子 Agent 负责具体执行。这种设计便于统一管理任务进度和终止条件，但也需要自行实现超时、重试、并发结果合并和故障恢复等机制。

## LangChain 与 LlamaIndex

LangChain 和 LlamaIndex 都可以用于构建 LLM 应用，它们的主要区别在于设计重心：**LangChain 更侧重模型与工具如何协同完成任务，LlamaIndex 更侧重外部数据如何被模型有效利用。**

LangChain 提供统一的模型接口、工具集成和 Agent 构建能力。开发者可以将模型与搜索、数据库查询、代码执行等工具连接起来，让 Agent 根据任务需要选择工具，并利用执行结果继续推进任务。在其生态中，LangGraph 提供更底层的编排能力，用于控制复杂流程中的分支、循环、状态持久化和暂停恢复。[LangChain 文档](https://docs.langchain.com/oss/python/langchain/overview)

LlamaIndex 以外部数据的接入和利用为重点，提供数据连接器、索引、检索器和查询引擎等组件。它帮助开发者将文档、PDF、数据库等数据源接入应用，在用户提问时检索相关内容，并将其提供给模型生成回答。同时，LlamaIndex 也提供 Agent 和 Workflows，支持工具调用和多步骤任务执行。[LlamaIndex 文档](https://developers.llamaindex.ai/python/framework/)

| 比较维度     | LangChain                              | LlamaIndex                             |
| ------------ | -------------------------------------- | -------------------------------------- |
| 设计重心     | 连接模型、工具和执行逻辑               | 连接外部数据与模型                     |
| 主要关注点   | Agent 如何选择工具、处理反馈并完成任务 | 数据如何被读取、索引、检索并用于回答   |
| 典型场景     | 自动化客服、研究助手、多工具任务执行   | 企业知识库问答、文档分析、私有数据查询 |
| 复杂流程支持 | 可以结合 LangGraph 控制流程和状态      | 可以使用 Workflows 编排事件驱动的流程  |
| 功能交叉     | 也支持数据加载、向量存储和 RAG         | 也支持 Agent、工具调用和流程编排       |

例如，构建一个财报分析助手时，需要解决两类问题：一类是从大量财报中找到可靠的数据和证据，另一类是组织检索、计算、分析和报告生成等操作。前者可以使用 LlamaIndex 构建数据检索链路，后者可以使用 LangChain Agent 或 LangGraph 组织执行流程。

实际选型时，应根据项目的主要难点判断。如果重点在于文档处理和检索质量，可以优先评估 LlamaIndex；如果重点在于工具调用和任务执行，可以优先评估 LangChain，并在需要精细控制流程时使用 LangGraph。两者也可以组合使用，但应以实际需求为依据，避免引入重复的抽象和维护成本。



## **Harness又是什么**

![image-20260624114639065](Agent 篇.assets/image-20260624114639065.png)

我们在AI的外面裹了一整圈“防护网”，包括输入过滤、格式清洗、参数校验、工具调用、输出过滤、错误重试等等，而这套包在 AI 外面的防护网，有一个名字叫做 Harness 。从最早只会聊天的ChatGPT，到能调工具、能循环、能记忆、能协作的智能体，再到外面包了一层Harness的工程系统，AI就是这样一步步，从一个只会”动嘴"的聊天机器人，变成了一个能在真实世界里干活的”打工仔”。如果说 Agent 解决的是”AI怎么干活”的问题，那Harness解决的是”AI怎么把活干靠谱”的问题。



## 记忆模块的数据存在哪种介质上?

记忆模块的数据存储介质不是单一的，而是根据记忆类型（短期 vs 长期） 和运行环境（开发 vs 生产） 采用分层存储策略。核心涉及 RAM（内存） 和 PostgreSQL（磁盘数据库） 两种介质。

### 一、短期记忆（Short-term Memory）— Checkpointer 机制

短期记忆 = 会话级记忆，按 thread_id 隔离，保存单次对话的完整状态（messages + 自定义字段）。

| 环境      | 存储类                                                   | 存储介质                    | 持久性             |
| --------- | -------------------------------------------------------- | --------------------------- | ------------------ |
| 开发/测试 | InMemorySaver <br />(来自 langgraph.checkpoint.memory)   | RAM（内存中的 Python dict） | 进程退出即丢失     |
| 生产环境  | PostgresSaver<br /> (来自 langgraph.checkpoint.postgres) | PostgreSQL 数据库（磁盘）   | 持久化，跨进程保留 |

**生产环境：**

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:123456@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 自动在 Postgres 中建表
```

**开发环境：**

```python
from langgraph.checkpoint.memory import InMemorySaver
agent = create_agent(..., checkpointer=InMemorySaver())  # 纯内存
```

>✅ 持久化 - 可以存到 Postgres，永久保存
>✅ 会话隔离 - 每个 thread_id 独立 
>❌ 跨会话不可见 - thread A 不知道 thread B 的历史

### 二、长期记忆（Long-term Memory）— Store 机制

长期记忆 = 用户级记忆，跨会话共享，存储用户画像、偏好等数据，通过 namespace + key 组织。

| 环境      | 存储类                                            | 存储介质                                    | 持久性         |
| --------- | ------------------------------------------------- | ------------------------------------------- | -------------- |
| 开发/测试 | InMemoryStore <br />(来自 langgraph.store.memory) | RAM（内存中的字典）                         | 进程退出即丢失 |
| 生产环境  | PostgresStore 等数据库实现                        | PostgreSQL 数据库 (磁盘持久化+向量语义检索) | 持久化         |

代码证据（长期记忆.py）：

```python
# InMemoryStore 将数据保存到内存字典中。在生产环境中使用基于数据库的存储。
store = InMemoryStore(index={"embed": embed, "dims": 2})
```

Store 还支持 向量相似度搜索（语义检索），当配置了 embedding 函数后，可以对记忆内容做语义匹配。

### 三、语义记忆（Semantic Memory）

当需要让 Agent 根据历史内容进行召回时。

存储介质：向量数据库，例如：

- Milvus
- Weaviate
- Qdrant
- Pinecone

存储内容：

```
原始文本
Embedding向量
元数据
```

检索流程：

```
历史对话
    ↓
Embedding
    ↓
向量库存储
    ↓
相似度检索
    ↓
召回相关记忆
```

>**可以回答：**
>AI Agent 的记忆通常采用分层存储。短期记忆存放在内存或 Redis 中，用于维持当前会话状态；长期记忆存储在 MySQL、PostgreSQL 等数据库中，用于保存用户画像和事实信息；对于需要语义检索的记忆，会将文本 Embedding 后存入 Milvus、Qdrant 等向量数据库，通过相似度召回相关历史信息。在生产环境中，通常是 Redis + 关系型数据库 + 向量数据库的组合方案，而不是单一存储介质。
>
>**面试加分点：**
>
>1. 为什么不全部用内存？ — 内存方案无法支持多实例部署、服务重启后状态恢复、水平扩展等生产需求。
>2. 为什么选 PostgreSQL？ — Postgres 同时支持关系型存储（checkpointer 的 checkpoint 表）和向量检索（pgvector 扩展），一套数据库即可覆盖结构化记忆和语义搜索两种场景。
>3. Redis 可以作为替代吗？ — 可以，Redis 适合做短期记忆的存储介质（TTL 自动过期很适合会话隔离），但不适合长期记忆（需要持久化保证）。LangGraph 社区也有 Redis checkpointer 实现。
>4. 本质区别 — 短期记忆是"状态快照"（State Snapshot），记录图执行的完整状态；长期记忆是"知识条目"（Knowledge Entry），是用户级的结构化/半结构化数据。



### Memory是 Agent 的一个关键模块。请问如何为 Agent 设计短期记忆和长期记忆系统？可以借助哪些外部工具或技术？

记忆模块是Agent打破LLM上下文窗口限制、实现持续学习和个性化的关键。设计Agent的记忆系统通常会模仿人类的记忆机制，分为短期记忆和长期记忆。

<strong>1. 短期记忆 (Short-Term Memory):</strong>

* **作用：**存储当前任务的上下文信息，包括即时对话历史、中间的思考步骤（如ReAct的Scratchpad）、工具的调用结果等。它是Agent进行连贯思考和行动的基础。
* **实现方式：**
  * <strong>LLM的上下文窗口 (Context Window):</strong> 这是最直接的短期记忆载体。所有最近的交互都会被放入Prompt中。
  * <strong>缓冲区 (Buffers):</strong> 在Agent框架（如LangChain）中，通常会使用不同类型的缓冲区来管理对话历史，例如：
    * <strong>ConversationBufferMemory:</strong> 存储完整的对话历史。
    * <strong>ConversationBufferWindowMemory:</strong> 只保留最近的K轮对话。
    * <strong>ConversationSummaryBufferMemory:</strong> 在历史对话过长时，动态地用LLM进行总结，以节省Token。
  * <strong>暂存器 (Scratchpad):</strong> 用于记录ReAct框架中的“Thought-Action-Observation”轨迹，是Agent进行逐步推理的关键。

<strong>2. 长期记忆 (Long-Term Memory):</strong>

* <strong>作用：</strong> 存储跨越任务和时间维度的信息，如用户的个人偏好、过去的成功/失败经验、领域知识等。它使得Agent能够“学习”和“成长”。
* <strong>实现方式与外部工具：</strong> 长期记忆的核心是“<strong>存储</strong>”和“<strong>检索</strong>”，这通常需要借助外部技术，最主流的是<strong>RAG (Retrieval-Augmented Generation)</strong> 范式。
  * <strong>核心技术：向量数据库 (Vector Database)</strong>
    * <strong>工具：</strong> Pinecone, ChromaDB, FAISS, Weaviate等。
    * <strong>工作流程：</strong>
      1.  <strong>存储（Storing/Writing）：</strong> 当Agent获得一个有价值的信息（如用户明确给出的偏好、一个成功解决问题的完整流程）时，它会使用一个<strong>嵌入模型（Embedding Model）</strong>将这段文本信息转换成一个高维向量。然后，将这个向量及其原始文本存入向量数据库。
      2.  <strong>检索（Retrieving/Reading）：</strong> 在Agent进行规划或决策时，它会把当前的任务或问题也转换成一个查询向量。然后，用这个查询向量去向量数据库中进行<strong>相似度搜索</strong>，找出与当前情况最相关的历史记忆。
      3.  <strong>使用（Using）：</strong> 检索到的记忆（原始文本）会被插入到LLM的Prompt中，作为额外的上下文，来指导LLM做出更明智的决策。
  * <strong>其他技术：</strong>
    * <strong>传统数据库/知识图谱：</strong> 对于结构化或关系型数据，使用SQL数据库或图数据库（如Neoj）进行存储和精确查询也是一种有效的长期记忆形式。



## 如何跨会话压缩对话并保留关键信息?

核心思路是分层处理：会话内通过消息修剪 + LLM 摘要控制上下文窗口；会话间通过Store 长期记忆将关键信息（用户偏好、事实、决策）提取为结构化知识，实现跨会话持久化。本质上是 "压缩原始消息流 + 蒸馏结构化知识" 的组合策略。

### **一、会话内压缩：三种策略**

项目中 核心组件\短期记忆\常见模式 目录给出了三种范式：

##### 策略 1：消息修剪（Trim Messages）— 最轻量

来源：修剪消息 (Trim messages).py

```python
@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    messages = state["messages"]
    if len(messages) <= 3:
        return None
    # 保留第一条（系统消息）+ 最近 3~4 条
    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages
    return {
        "messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *new_messages]
    }
```

>原理：滑动窗口，只保留 [系统消息] + [最近 N 轮对话]，丢弃中间消息。 
>适用场景：简单对话、对历史信息依赖低的场景。
>缺点：粗暴丢弃，会丢失中间的关键信息。

##### 策略 2：选择性删除（Delete Messages）— 中等

来源：删除消息 (Delete messages).py

```python
@after_model
def delete_old_messages(state: AgentState, runtime: Runtime) -> dict | None:
    messages = state["messages"]
    if len(messages) > 2:
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
    return None
```

>原理：每轮对话后删除最早的消息，渐进式清理。 
>适用场景：流式对话中逐步淘汰旧消息。

##### 策略 3：LLM 摘要（Summarize Messages）— 最推荐，面试核心答案

来源：总结消息 (Summarize messages).py

```python
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="openai:gpt-4o",
    middleware=[
        SummarizationMiddleware(
            model="openai:gpt-4o-mini",           # 用小模型做摘要，控制成本
            max_tokens_before_summary=4000,        # 累积到 4000 tokens 触发
            messages_to_keep=20,                   # 摘要后保留最近 20 条原始消息
            summary_prompt="Custom prompt...",     # 可自定义摘要提示词
        )
    ],
    checkpointer=checkpointer,
)
```

>原理：
>
>1. 监控 messages 列表的 token 总量
>2. 超过 max_tokens_before_summary 阈值时，触发摘要
>3. 用 LLM（通常用小模型降低成本）将旧消息压缩为一条摘要消息
>4. 替换旧消息，保留最近 messages_to_keep 条原始消息
>
>消息流变化：
>
>压缩前: [msg1, msg2, msg3, ... msg50]   (50条，超过4000 tokens)
>                                   ↓ 触发摘要
>压缩后: [Summary: "用户名叫Bob，之前讨论了猫和狗的诗歌..."], msg31, msg32, ... msg50
>       ↑ 一条摘要替代了前30条                        ↑ 保留最近20条原始消息
>
>这是会话内压缩的最佳实践，因为：
>
>- 不会粗暴丢失信息，LLM 会提取关键事实
>- token 可控，不会爆上下文窗口
>- 用小模型做摘要，经济高效

##### 补充策略：上下文编辑（Context Editing）

来源：上下文编辑 (Context editing).py

```python
ContextEditingMiddleware(
    edits=[
        ClearToolUsesEdit(max_tokens=1000),  # 清除旧的工具调用结果
    ],
)
```

原理：Agent 场景中，工具调用（tool calls）的输入/输出往往占大量 token。当超过阈值后，清除旧的工具调用详情，只保留最终结果。这是 Agent 特有的压缩手段。

### 二、跨会话持久化：关键信息蒸馏到 Store

会话内压缩解决的是"单次对话不爆 token"，但跨会话需要把关键信息持久化。
来源：内存（存储）(Memory (Store)).py

```python
# 写入关键信息
@tool
def save_user_info(user_id: str, user_info: dict, runtime: ToolRuntime) -> str:
    """Save user info."""
    store = runtime.store
    store.put(("users",), user_id, user_info)  # namespace, key, value
    return "Successfully saved user info."

# 跨会话读取
@tool
def get_user_info(user_id: str, runtime: ToolRuntime) -> str:
    """Look up user info."""
    store = runtime.store
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"
```

跨会话信息流：

```sh
Session A: "我叫 Bob，喜欢简洁风格"
     ↓ Agent 调用 save_user_info 工具
     ↓ Store.put(("users",), "bob_123", {name: "Bob", style: "concise"})
     ↓ 写入 PostgreSQL（生产环境）

Session B（几天后新会话）:
     ↓ 中间件 @dynamic_prompt 从 Store 读取用户偏好
     ↓ 动态注入到 system prompt: "User prefers concise responses"
     ↓ Agent 无需知道 Session A 的原始对话，但拥有了关键上下文
```

**来源：系统提示 存储 (Store).py**

```python
@dynamic_prompt
def store_aware_prompt(request: ModelRequest) -> str:
    store = request.runtime.store
    user_prefs = store.get(("preferences",), user_id)
    base = "You are a helpful assistant."
    if user_prefs:
        style = user_prefs.value.get("communication_style", "balanced")
        base += f"\nUser prefers {style} responses."
    return base
```

### 三、完整架构总结

```
                    ┌─────────────────────────────────────────┐
                    │         跨会话压缩 & 关键信息保留         │
                    └─────────────────┬───────────────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
   ┌─── 会话内压缩 ───┐    ┌─── 跨会话持久化 ───┐    ┌── 上下文注入 ──┐
   │                   │    │                    │    │                 │
   │ ① Trim (修剪)     │    │ ④ Store 长期记忆    │    │ ⑤ 动态Prompt   │
   │   滑动窗口保留     │    │   关键事实→Store    │    │   从Store读取   │
   │   最近N条消息      │    │   持久化到Postgres  │    │   注入system    │
   │                   │    │                    │    │   prompt        │
   │ ② Summarize (摘要)│    │ ⑥ 用户偏好/画像     │    │                 │
   │   LLM压缩旧消息   │    │   name, age, style  │    │ ⑦ 风格注入     │
   │   为一条Summary    │    │   → Store           │    │   从Store读取   │
   │                   │    │                    │    │   写作风格示例   │
   │ ③ ClearToolUses   │    │                    │    │   附加到消息     │
   │   清除旧工具调用   │    │                    │    │                 │
   └───────────────────┘    └────────────────────┘    └─────────────────┘
   
    ← 控制 token 窗口 →         ← 持久化关键知识 →         ← 新会话自动恢复 →
```

>面试加分点
>为什么摘要用小模型？ — gpt-4o-mini 做摘要的成本只有 gpt-4o 的 1/15，摘要任务不需要强推理能力，用小模型性价比最高。
>摘要 vs 修剪的选择：修剪是 O(1) 的操作，零成本但信息损失大；摘要有一次 LLM 调用成本，但能保留关键语义。生产环境推荐两者结合——先用摘要压缩到阈值内，如果还超再做修剪兜底。
>关键信息提取时机：不是等到会话结束才提取。最佳实践是在对话过程中，Agent 通过工具（如 save_user_info）实时将关键信息写入 Store，这样即使会话中断也不会丢失。
>与传统 RAG 的区别：传统 RAG 是被动检索文档；这里的 Store 记忆是 Agent 主动写入和读取的结构化知识，粒度更精准（按 user_id + namespace），**且支持向量语义搜索。**

### 四、核心思路：不是压缩文本，而是压缩“信息”

跨会话不能简单做 truncate，而是做：

```
原始对话 → 信息抽取 → 结构化存储 → 语义压缩 → 记忆更新
```

#### 一、大厂标准方案（4层结构）

##### 1️⃣ 关键信息抽取（Information Extraction）

从对话中提取：

- 用户偏好
- 事实信息（fact）
- 任务状态（task state）
- 决策结果
- 长期稳定信息

例如：

```
用户：我喜欢 Go，不喜欢 Java
→ 结构化：
{
  "preference_language": "Go",
  "dislike_language": "Java"
}
```

------

##### 2️⃣ 摘要压缩（Conversation Summarization）

将历史对话压缩成：

- 会话摘要（session summary）
- 关键事件链（event chain）

```
原始：
10轮对话

压缩后：
用户在做一个AI Agent项目 → 使用LangGraph → 关注记忆模块设计
```

##### 3️⃣ 结构化记忆存储（Structured Memory）

存入长期记忆 Store：

| 类型     | 示例                     |
| -------- | ------------------------ |
| 用户偏好 | Go语言、中文回答         |
| 用户身份 | AI应用开发工程师         |
| 项目状态 | 正在做Agent系统          |
| 关键决策 | 使用PostgreSQL做记忆存储 |

存储介质：

- PostgreSQL（结构化）
- Redis（热数据）
- 对象存储（原始日志）

##### 4️⃣ 语义记忆压缩（Vector Memory）

对历史对话做 embedding：

```
对话 → embedding → 向量数据库
```

用于：

- 相似问题召回
- 历史经验匹配
- 隐式信息恢复

向量库：

- Milvus
- Qdrant
- Weaviate

####  二、跨会话压缩整体流程（面试重点）

```
            用户多轮对话
                   │
                   ▼
        ┌──────────────────────┐
        │  信息抽取（LLM）      │
        └────────┬─────────────┘
                 ▼
     ┌──────────────────────────┐
     │  结构化记忆（Fact/Pref） │
     └────────┬─────────────────┘
              ▼
   ┌────────────────────────────┐
   │  会话摘要压缩（Summary）   │
   └────────┬───────────────────┘
            ▼
   ┌────────────────────────────┐
   │  向量化存储（Embedding）   │
   └────────┬───────────────────┘
            ▼
     长期记忆库（Postgres + Vector DB）
```



#### 三、工程实现策略（大厂加分点）

#####  1. Sliding Window + Summary混合

```
保留最近 N 轮 + 历史摘要
```

##### 2. Token Budget控制

```
Context = Recent Messages + Summary + Retrieved Memory
```

动态分配 token：

- 30% 最近对话
- 30% 历史摘要
- 40% 检索记忆

##### 3. 重要性评分（Memory Scoring）

给每条记忆打分：

- 是否用户主动强调
- 是否长期稳定
- 是否可复用
- 是否影响决策

##### 4. 去重与合并（Deduplication）

例如：

```
“喜欢Go”
“偏好Go语言”
→ 合并为一条用户偏好
```



#### 四、面试一句话总结（重点）

> 跨会话对话压缩的核心不是简单截断，而是通过“信息抽取 + 会话摘要 + 结构化记忆 + 向量语义存储”的多层机制，将对话转化为可持久化的事实与语义表示，并在后续会话中通过检索与重组恢复关键信息，从而实现低成本高信息密度的上下文管理。



## FSM状态机分哪几部分?如何代码实现?

FSM（有限状态机）由 5 个核心要素 组成：状态集合（States）、事件/输入（Events）、转移函数（Transitions）、初始状态（Initial State）、终止状态（Final States）。在 AI 应用领域，LangGraph 的 StateGraph 就是对 FSM 的工程化封装，用 State 定义 + Node 节点 + Edge 边 + Conditional Edge 条件边 四件套来实现。

### 一、代码实现：三层递进

第一层：纯 Python 原生实现（面试手撕必备）

```python
from enum import Enum, auto
from typing import Callable

# ========== 1. 定义状态集合 ==========
class State(Enum):
    IDLE = auto()           # 空闲，等待用户输入
    THINKING = auto()       # LLM 正在思考
    CALLING_TOOL = auto()   # 正在调用工具
    RESPONDING = auto()     # 正在生成回答
    DONE = auto()           # 结束

# ========== 2. 定义事件/输入 ==========
class Event(Enum):
    USER_INPUT = auto()     # 收到用户消息
    LLM_DECIDE_TOOL = auto()  # LLM 决定调用工具
    LLM_DECIDE_REPLY = auto() # LLM 决定直接回答
    TOOL_RESULT = auto()    # 工具返回结果
    RESPONSE_DONE = auto()  # 回答完成

# ========== 3. 定义转移函数 δ ==========
class FSM:
    def __init__(self):
        self.state = State.IDLE  # 初始状态 q₀
        
        # 转移表: (当前状态, 事件) → (新状态, 动作)
        self.transitions: dict[tuple[State, Event], tuple[State, Callable | None]] = {
            (State.IDLE, Event.USER_INPUT):         (State.THINKING, self._call_llm),
            (State.THINKING, Event.LLM_DECIDE_TOOL): (State.CALLING_TOOL, self._execute_tool),
            (State.THINKING, Event.LLM_DECIDE_REPLY):(State.RESPONDING, self._stream_response),
            (State.CALLING_TOOL, Event.TOOL_RESULT): (State.THINKING, self._call_llm),  # 循环！
            (State.RESPONDING, Event.RESPONSE_DONE): (State.DONE, None),
        }
    
    def handle_event(self, event: Event):
        """核心循环：根据当前状态和事件，执行转移"""
        key = (self.state, event)
        if key not in self.transitions:
            raise ValueError(f"非法转移: {self.state} + {event}")
        
        new_state, action = self.transitions[key]
        print(f"  {self.state.name} --[{event.name}]--> {new_state.name}")
        self.state = new_state
        
        if action:
            action()
    
    # ========== 4. 定义动作 ==========
    def _call_llm(self):
        print("  [动作] 调用 LLM，分析用户意图...")
    
    def _execute_tool(self):
        print("  [动作] 执行工具调用...")
    
    def _stream_response(self):
        print("  [动作] 流式输出回答...")
    
    @property
    def is_done(self) -> bool:
        return self.state == State.DONE  # 终止状态 F

# ========== 运行 ==========
fsm = FSM()
# 模拟一次 ReAct 循环: IDLE → THINKING → CALLING_TOOL → THINKING → RESPONDING → DONE
events = [
    Event.USER_INPUT,         # IDLE → THINKING
    Event.LLM_DECIDE_TOOL,    # THINKING → CALLING_TOOL
    Event.TOOL_RESULT,        # CALLING_TOOL → THINKING
    Event.LLM_DECIDE_REPLY,   # THINKING → RESPONDING
    Event.RESPONSE_DONE,      # RESPONDING → DONE
]
for event in events:
    fsm.handle_event(event)
```

输出：

```sh
  IDLE --[USER_INPUT]--> THINKING
  [动作] 调用 LLM，分析用户意图...
  THINKING --[LLM_DECIDE_TOOL]--> CALLING_TOOL
  [动作] 执行工具调用...
  CALLING_TOOL --[TOOL_RESULT]--> THINKING
  [动作] 调用 LLM，分析用户意图...
  THINKING --[LLM_DECIDE_REPLY]--> RESPONDING
  [动作] 流式输出回答...
  RESPONDING --[RESPONSE_DONE]--> DONE
```



**第二层：LangGraph StateGraph 实现（工程化 FSM）**
项目中 使用Graph API.py 就是一个完整的 FSM 实现，精确对应五要素：

```python
from langgraph.graph import StateGraph, START, END
from typing import Literal

# ========== 1. 状态集合 (State) ==========
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], operator.add]  # Reducer 模式
    llm_calls: int

# ========== 2. 节点 (Nodes) = 每个节点代表一个"处理状态" ==========
def llm_call(state: dict):       # 状态: LLM 思考中
    """LLM 决定是否调用工具"""
    return {
        "messages": [model_with_tools.invoke(...)],
        "llm_calls": state.get('llm_calls', 0) + 1
    }

def tool_node(state: dict):      # 状态: 工具执行中
    """执行工具调用"""
    result = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tools_by_name[tool_call["name"]]
        observation = tool.invoke(tool_call["args"])
        result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
    return {"messages": result}

# ========== 3. 转移函数 (Conditional Edge) ==========
def should_continue(state: MessagesState) -> Literal["tool_node", END]:
    """条件路由：LLM 是否发起了工具调用？"""
    if state["messages"][-1].tool_calls:
        return "tool_node"    # → 转移到工具节点
    return END                # → 转移到终止状态

# ========== 4 & 5. 构建 FSM：初始状态(START) + 终止状态(END) ==========
agent_builder = StateGraph(MessagesState)

agent_builder.add_node("llm_call", llm_call)       # 注册节点
agent_builder.add_node("tool_node", tool_node)      # 注册节点

agent_builder.add_edge(START, "llm_call")           # START → llm_call (初始转移)
agent_builder.add_conditional_edges(                 # llm_call → tool_node 或 END (条件转移)
    "llm_call", should_continue, ["tool_node", END]
)
agent_builder.add_edge("tool_node", "llm_call")     # tool_node → llm_call (循环)

agent = agent_builder.compile()                      # 编译
```

对应的状态转移图：

```
         START
           │
           ▼
    ┌──► llm_call ◄──────────┐
    │        │                │
    │   should_continue       │
    │    /          \         │
    │  tool_calls   no calls  │
    │  /              \       │
    │ ▼               ▼      │
    │ tool_node      END     │
    │   │                    │
    └───┘  (循环回到 llm_call)
```

第三层：Hello World 极简版（面试快速展示）
项目中 Hello World示例.py 展示了最小可用的 FSM：

```python
from langgraph.graph import StateGraph, MessagesState, START, END

def mock_llm(state: MessagesState):
    return {"messages": [{"role": "ai", "content": "hello world"}]}

graph = StateGraph(MessagesState)
graph.add_node(mock_llm)                    # 一个节点
graph.add_edge(START, "mock_llm")          # START → 节点
graph.add_edge("mock_llm", END)            # 节点 → END
graph = graph.compile()
```

这是最简单的线性 FSM：START → mock_llm → END，没有条件分支和循环。

### 二、FSM 各部分与 LangGraph API 的精确映射

| FSM 要素      | LangGraph API                 | 代码中的体现                          |
| ------------- | ----------------------------- | ------------------------------------- |
| 状态集合 Q    | TypedDict / State             | MessagesState(TypedDict) 定义所有字段 |
| 输入事件 Σ    | invoke() 传入的数据           | {"messages": [HumanMessage(...)]}     |
| 转移函数 δ    | add_conditional_edges()       | should_continue() 返回下一个节点名    |
| 初始状态 q₀   | START 常量                    | add_edge(START, "llm_call")           |
| 终止状态 F    | END 常量                      | should_continue 返回 END              |
| 节点/处理逻辑 | add_node()                    | llm_call, tool_node 函数              |
| 无条件转移    | add_edge()                    | add_edge("tool_node", "llm_call")     |
| Reducer       | Annotated[list, operator.add] | messages 列表自动追加而非覆盖         |

### 三、高级模式：中断（Interrupt）= FSM 的暂停/恢复

项目中 配置中断.py 展示了 FSM 在特定状态暂停等待外部输入，然后恢复执行——这本质上是 FSM 的扩展能力：

```python
# 在 tool_node 执行前暂停，等待人类审批
HumanInTheLoopMiddleware(
    interrupt_on={
        "write_file": True,           # 所有工具调用都需审批
        "execute_sql": {"allowed_decisions": ["approve", "reject"]},
        "read_data": False,           # 安全操作，无需审批
    },
)
```

FSM 状态流变为：... → tool_node(暂停) → [等待人类] → tool_node(继续) → llm_call → ...

>面试加分点
>
>1. 为什么 AI Agent 天然是 FSM？ — ReAct 循环（Think → Act → Observe → Think...）就是一个带循环的状态机。LLM 的输出（是否调用工具、调用哪个工具）就是"事件"，驱动状态转移。
>2. FSM vs DAG 的区别？ — DAG（有向无环图）是 FSM 的子集，不允许循环。项目中的 提示链 工作流是 DAG，而 Agent（有 tool call 循环）是完整 FSM。面试关键区分点：有循环 = FSM，无循环 = DAG/工作流。
>3. State 中的 Reducer 模式 — Annotated[list[AnyMessage], operator.add] 意味着每次节点返回的 messages 不是覆盖而是追加。这解决了 FSM 中"状态如何累积"的问题。如果不用 Reducer，默认行为是覆盖（last-write-wins）。
>4. Functional API 是 FSM 的另一种表达 — 项目中 使用Functional API.py 用 @entrypoint + @task + while True 循环实现同样的 ReAct FSM，区别是用代码控制流（while/if）代替了显式的图边定义，适合简单场景。



## 为什么选择ReAct，而不是纯Chain或ToolUse?

一句话回答

> Chain 是确定性流水线，ToolUse 是单次工具调用，ReAct 是带推理循环的自主决策。 选择 ReAct 的核心原因是：面对不确定性任务时，LLM 需要根据中间结果动态决定下一步做什么、做几次、何时停止——这是 Chain 和单次 ToolUse 做不到的。

### 一、三种模式在本质上的区别

项目中恰好有这三种模式的完整实现，可以直接对比：

#### 模式 1：纯 Chain — 确定性流水线，无 LLM 决策

来源：chains.py

```python
chain = (
    topic_chain                              # 步骤1：固定执行 - 生成主题
    | {"good": goods_chain, "bad": bads_chain, ...}  # 步骤2：固定执行 - 并行分析
    | summary_chain                          # 步骤3：固定执行 - 汇总
)
chain.invoke({"input": "常见水果"})
```

特点：执行路径在编译时就确定了（DAG），每一步必定执行，LLM 只是"加工厂"，不参与流程决策。

#### 模式 2：纯 ToolUse — 单次工具调用，无循环

来源：tool.py

```python
chain = (
    RunnablePassthrough()
    | llm.bind_tools(tools=[get_temperature])  # LLM 决定调用哪个工具
    | output_parser                            # 解析工具调用 → 结束
)
chain.invoke("杭州今天多少度？")
```

特点：LLM 能选择工具，但只调一次就结束了。没有"拿到结果后再判断是否需要继续"的能力。

#### 模式 3：ReAct — 推理 + 行动循环

来源：使用Graph API.py 和 React.py

```python
# ReAct 的核心：条件路由 + 循环
def should_continue(state) -> Literal["tool_node", END]:
    if state["messages"][-1].tool_calls:  # LLM 决定要调用工具？
        return "tool_node"                # → 执行工具，然后回到 LLM 继续推理
    return END                            # → LLM 认为信息够了，输出最终回答

agent_builder.add_edge("tool_node", "llm_call")  # 关键！工具执行完 → 回到 LLM
```

特点：LLM 推理 → 调用工具 → 观察结果 → LLM 再推理 → ... 形成循环，直到 LLM 认为信息足够才输出。

### 二、用一个真实场景说明三者的差距

以项目中的 Agentic RAG 为例，用户问：
"用 create_react_agent 写一个能查询股价的 agent 示例"

| 模式       | 执行过程                                                     | 结果                                                   |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| 纯 Chain   | 无法处理。Chain 没有"查文档"这个步骤，除非预先硬编码         | 只能靠 LLM 自己猜，答案可能过时                        |
| 纯 ToolUse | 调一次 fetch_documentation，拿到文档后直接结束               | 如果第一次 fetch 的 URL 不对，无法重试，直接给错误答案 |
| ReAct      | ① Think: "需要查文档" → 调 fetch_documentation(url1) <br/> ② Observe: "这个页面是概览，没有代码示例" <br/> ③ Think: "需要找具体 API 页面" → 调 fetch_documentation(url2) <br/> ④ Observe: "找到了 create_react_agent 用法" <br/> ⑤ Think: "信息够了" → 输出最终答案 | 正确，因为能根据中间结果自主调整策略                   |

### 三、决策矩阵：什么时候用什么？

```
任务复杂度
    ▲
    │         ┌────────────────────┐
    │         │     ReAct Agent     │  ← 不确定要几步、需要根据中间结果决策
    │         │  "LangChain是什么？  │     如：Agentic RAG、多步推理、
    │         │   帮我搜索一下再回答" │     工具调用后需判断是否继续
    │         └────────────────────┘
    │
    │    ┌──────────────────┐
    │    │    纯 ToolUse      │  ← 确定只需调一次工具
    │    │  "杭州多少度？"     │     如：单工具查询、函数调用
    │    └──────────────────┘
    │
    │ ┌────────────────┐
    │ │   纯 Chain       │  ← 步骤固定、无需 LLM 决策
    │ │ "分析水果优缺点"  │     如：提示链、ETL 管道
    │ └────────────────┘
    └──────────────────────────────────► 不确定性
```

![image-20260623010530238](Agent 篇.assets/image-20260623010530238.png)

>ReAct 的本质：Thought + Action + Observation 循环

### 四、ReAct 的风险和应对（面试必说）

| 风险         | 原因                         | 应对                                                         |
| ------------ | ---------------------------- | ------------------------------------------------------------ |
| 死循环       | LLM 反复调用工具不收         | 设置 max_iterations，超过强制结束                            |
| 成本高       | 每轮循环都是一次 LLM 调用    | 用小模型做路由判断，大模型只做最终回                         |
| 工具选择错误 | LLM 选错工具或传错参数       | 项目中 自定义工具错误处理 用 wrap_tool_call 中间件捕获错误并返回提示信息，让 LLM 自我修正 |
| 幻觉         | 推理链中间步骤出错但继续推进 | Human-in-the-loop 中间件在关键步骤暂停让人类审批             |

>面试加分点
>
>1. 不是"ReAct 最好"，而是"按需选择" — 生产环境中 80% 的场景用 Chain 或单次 ToolUse 就够了（如 RAG 检索 + 生成、分类、格式化），成本低、可控性高。只有当任务不确定性高、需要多步推理、需要错误恢复时才上 ReAct。
>2. 现代大模型的原生 Tool Calling 已经内置了 ReAct 循环 — 项目中的 构建真实世界的代理 用 create_agent 一行代码就能创建 ReAct Agent，因为 GPT-4/Claude 等模型原生支持 function calling，框架只需要实现 "调工具→回传结果→再调模型" 的循环即可，不需要再手写 ReAct prompt 模板。
>3. ReAct vs Plan-and-Execute — ReAct 是"边想边做"，Plan-and-Execute 是"先制定完整计划再逐步执行"。对于复杂任务，Plan-and-Execute 更高效（减少 LLM 调用次数），但灵活性更低。实际项目中常结合使用：用 Plan-and-Execute 做宏观规划，每个子任务内部用 ReAct。



## 用户今天问A、三天后问B，如何关联两次提问?

用户今天问 A，三天后问 B，本质上属于 **跨会话上下文关联问题**。

生产级 AI Agent 一般不会保存全部历史对话直接塞给 LLM，而是采用：

```
记忆存储（Memory）
        +
语义检索（Semantic Retrieval）
        +
上下文重构（Context Reconstruction）
```

来完成关联。

>通过 用户 ID（User ID） 作为唯一标识，将三天前的提问 A 的关键信息（意图、事实、偏好）从短期对话历史中提取并持久化到长期记忆 Store 中；三天后提问 B 时，Agent 在初始化阶段从 Store 检索与 B 相关的历史上下文，注入到 Prompt 或 State 中，从而实现语义关联。

### 一、 为什么不能只靠短期记忆？

项目中 理解.md 明确指出：
短期记忆 (Checkpointer)：按 thread_id 隔离。三天后的提问 B 通常是一个新的 thread_id（新会话），无法自动看到旧会话 A 的内容。
长期记忆 (Store)：按 namespace + user_id 组织，跨会话、跨时间持久化。

### 二、 技术实现路径（三步走）

1. 会话 A（今天）：主动写入长期记忆

  在 Agent 执行过程中，通过工具或中间件识别关键信息并存入 Store

  ```python
# 代码参考：langchain-demo/高级用法/长期记忆/从工具中写入长期记忆.py
from langgraph.store.memory import InMemoryStore # 生产环境用 PostgresStore

store = InMemoryStore()

@tool
def save_user_context(info: dict, runtime: ToolRuntime):
    """保存用户上下文"""
    user_id = runtime.context.user_id
    # 存入 namespace: ("users", user_id), key: "context_A"
    store.put(("users", user_id), "context_A", info) 
    return "已记录"

agent = create_agent(model, tools=[save_user_context], store=store, context_schema=Context)
  ```

2. 存储介质选择（大厂实践）

  根据数据量级和检索方式选择：

  - **结构化偏好**：存 PostgreSQL (Key-Value)，适合存“用户喜欢 Python”。
  - 非结构化语义：存 Vector Store (如 pgvector)，适合存“三天前问了关于 React 的问题”，因为三天后的 B 可能是“怎么配置 Hooks”，需要向量相似度检索才能关联上。

3. 会话 B（三天后）：动态检索与注入
   在 Agent 启动时（或在 before_model 中间件中），根据 B 的内容去 Store 捞取相关背景。

  ```python
# 代码参考：langchain-demo/高级用法/智能体中的上下文工程/模型上下文/消息/存储 (Store).py
@wrap_model_call
def inject_history(request: ModelRequest, handler):
    user_id = request.runtime.context.user_id
    store = request.runtime.store
    
    # 1. 向量检索：查找与当前问题 B 相关的历史记录 A
    # query 是当前用户输入 B
    results = store.search(("users", user_id), query=request.messages[-1].content)
    
    # 2. 注入上下文
    if results:
        history_context = f"参考之前的讨论: {results[0].value}"
        # 将历史背景拼接到 System Prompt 或作为一条 User Message 插入
        new_messages = [{"role": "system", "content": history_context}] + request.messages
        request = request.override(messages=new_messages)
        
    return handler(request)
  ```

**大厂 Memory 一般是双存储**

```
                    User Memory
                          │
         ┌────────────────┴────────────────┐
         │                                 │
         ▼                                 ▼

 Structured Memory             Semantic Memory

 PostgreSQL                    Vector DB

 用户画像                       历史对话
 用户偏好                       会话摘要
 用户配置                       项目经验
 业务事实                       推理过程
```



### 三、 进阶场景：如何定义“关联”？

| 关联类型 | 实现方式             | 示例                                                         |
| -------- | -------------------- | ------------------------------------------------------------ |
| 显式引用 | 关键词匹配 / ID 关联 | A: "帮我记下项目ID是123" -> B: "项目123进度如何？"           |
| 隐式语义 | 向量检索 (Embedding) | A: "推荐个后端框架" -> B: "那个框架怎么安装？" (通过向量相似度召回 A) |
| 用户画像 | 结构化字段更新       | A: "我是 Java 开发" -> Store 更新 user_profile.language = "Java" -> B: "写个 Demo" (自动用 Java 写) |



### 四、 腾讯大厂面试加分项

1. **隐私与合规 (PII)**：在写入 Store 前，必须经过 PII 过滤（项目中 PII 检测），确保不持久化敏感个人信息。
2. **记忆衰减与清理**：三天前的 A 可能已经过时。需要在 Store 中增加 metadata（如时间戳、过期时间），检索时优先召回近期且相关的记忆。
3. **人机协同 (HITL)**：对于重要的跨会话关联（如修改用户核心偏好），应触发 HumanInTheLoopMiddleware，让用户确认“是否要将此信息记为长期偏好”。

> 总结：关联的核心不是“记住所有对话”，而是**“在正确的时机，从长期存储中检索出最相关的片段”**。如果面试官问：
>
> > 已经有语义记忆了，为什么还需要结构化记忆？
>
> 可以回答：
>
> > 因为语义记忆和结构化记忆解决的是不同问题。语义记忆基于 Embedding 和向量检索，适合处理模糊关联、经验召回和上下文补充，但其结果具有概率性，无法保证精确命中。结构化记忆则用于存储确定性事实，例如用户画像、偏好、配置和业务状态，可以通过 Key-Value 或 SQL 精确查询。因此生产环境通常采用双层记忆架构：结构化记忆负责事实存储，语义记忆负责关联召回，两者结合才能兼顾准确性和泛化能力。



## 多Agent框架怎么设计?

> 设计多 Agent 系统时，我首先会判断任务是否真的需要多个 Agent。如果单个 Agent 配合工具或 Skill 就能稳定完成，就没有必要增加协作成本。当不同子任务需要明显不同的专业知识、上下文或工具权限，或者可以独立并行执行时，我才考虑拆分。
>
> **划分 Agent 的核心依据，是能否形成相对独立、可验证的任务边界。** 每个 Agent 应该有明确的目标、必要的上下文和工具，以及可以验收的输出。比如生成一份行业研究报告，可以让不同研究 Agent 分别调查不同领域，返回带来源的结论，再由一个汇总 Agent 完成综合分析。单纯的格式转换或一次数据库查询，通常作为工具或工作流步骤就够了。
>
> **划分完成后，再根据任务依赖设计编排。** 流程固定、前后依赖明确的任务，用工作流串联；任务需要动态拆解和分配时，再引入调度 Agent。没有依赖的子任务可以并行，有依赖的任务按顺序执行，由调度层汇总结果、判断是否需要补充信息，以及何时结束。
>
> **协作时，我会明确输入输出和状态归属。** 每个 Agent 只接收完成任务所需的信息，返回结构化的结果、证据和执行状态，而不是互相传递全部聊天记录。公共任务状态由编排层统一管理，并明确并行结果的合并规则，避免多个 Agent 随意覆盖同一份数据。
>
> 最后，生产环境还需要设置超时、重试次数、最大交互轮数和成本预算，并记录各 Agent 的执行过程。是否采用多 Agent，要通过任务成功率、延迟和成本与单 Agent 方案对比，确认拆分带来的收益超过协作开销。

**如果面试官追问：“拆成 Agent 和拆成 Skill，有什么区别？”**

可以接着回答：

> Skill 主要复用“任务应该怎么做”的规范，仍然可以由同一个 Agent 加载并执行；拆成多个 Agent，则是把子目标交给具有各自上下文和执行过程的执行单元。因此，如果只是操作步骤不同，优先考虑 Skill；如果需要独立处理上下文、限制工具权限，或者并行完成子任务，再考虑拆 Agent。权限隔离仍然需要运行环境实际强制执行。

### 多 Agent 框架的三种主流架构

项目中虽然没有直接的多 Agent 示例，但 LangGraph 是实现多 Agent 的核心底座。大厂通常采用以下三种模式：

1. **流水线式 (Pipeline / Sequential)**
   逻辑：Agent A 的输出 → Agent B 的输入 → Agent C 的输出。
   适用：步骤固定、前后依赖强的场景（如：翻译 -> 润色 -> 格式校验）。
   项目对应：提示链 DAG 的升级版，每个节点换成一个 Agent。
2. **编排者-工作者式 (Supervisor / Orchestrator)**
   逻辑：一个“大脑” Agent（Supervisor）负责分析意图，动态决定调用哪个“手脚” Agent（Worker），并汇总结果。
   适用：任务不确定、需要跨领域协作的场景（如：用户问“帮我写个 Python 脚本并查下最新文档”，Supervisor 分配给 Coding Agent 和 Search Agent）。
   技术实现：利用 LangGraph 的 StateGraph 实现条件路由。
3. **辩论/协作式 (Debate / Collaborative)**
   逻辑：多个 Agent 针对同一问题提出方案，互相评审（Critic），直到达成一致。
   适用：高准确率要求的场景（如：代码生成后的 Code Review Agent）。

### Agent 如何通信

#### 方案1：共享状态

最常见。

```
Agent A
    ↓
Shared State
    ↓
Agent B
```

例如 LangGraph：

```
state["documents"]
state["analysis"]
state["result"]
```

所有 Agent 读写同一个 State。

------

#### 方案2：消息机制

类似 Actor Model。

```
Agent A
    ↓
Message
    ↓
Agent B
```

例如：

```
{
  "task":"analyze",
  "data":"财报内容"
}
```

------

#### 方案3：事件驱动

```
Research Done
        ↓
触发
        ↓
Analysis Agent
```

适用于长流程。

### 划分 Agent 的依据是什么

#### 原则1：按职责划分（最常见）

例如：

```
Research Agent
Analysis Agent
Coding Agent
Report Agent
```

每个 Agent 专注一个领域。

类似微服务思想：

```
单一职责原则（SRP）
```

------

#### 原则2：按工具域划分

例如：

```
Search Agent
Database Agent
Excel Agent
Code Agent
```

每个 Agent 负责一类工具。

```
Agent
   ↓
Tool Set
```

------

#### 原则3：按知识域划分

例如企业知识库：

```
HR Agent
Finance Agent
Legal Agent
IT Agent
```

不同 Agent 使用不同知识库。

------

#### 原则4：按推理能力划分

例如：

```
Planner Agent
Executor Agent
Reviewer Agent
```

对应：

```
规划
执行
评审
```

这也是目前比较流行的模式。

### 代码实现思路（基于 LangGraph）

在 LangGraph 中，多 Agent 本质上是多个 StateGraph 的嵌套或连接。

```python
from langgraph.graph import StateGraph, END

# 1. 定义子 Agent（可以是独立的 Graph）
def researcher_agent(state):
    # 执行搜索逻辑
    return {"context": search_results}

def writer_agent(state):
    # 执行写作逻辑
    return {"draft": generated_text}

# 2. 定义编排逻辑（Supervisor）
def supervisor(state):
    # 根据用户意图决定下一步去哪个 Agent
    if state["step"] == "research":
        return "researcher"
    elif state["step"] == "write":
        return "writer"
    return END

# 3. 构建多 Agent 图
workflow = StateGraph(AgentState)
workflow.add_node("supervisor", supervisor)
workflow.add_node("researcher", researcher_agent)
workflow.add_node("writer", writer_agent)

workflow.add_edge("supervisor", "researcher") # 动态边可以通过 add_conditional_edges 实现
workflow.add_edge("researcher", "writer")
workflow.add_edge("writer", END)
```



## 在构建一个复杂的 Agent 时，你认为最主要的挑战是什么？

构建一个复杂的Agent（例如，需要多步规划、多工具交互、长期记忆的Agent）时，会遇到一系列从理论到工程的挑战。我认为最主要的挑战可以归结为以下几点：

1.  <strong>规划与推理的鲁棒性 (Robustness of Planning and Reasoning):</strong>
    * <strong>挑战描述：</strong> 复杂的任务往往需要长期、多步的规划。当前的LLM虽然强大，但其推理链条仍然很脆弱。Agent很容易在执行过程中“迷失”——忘记最初的目标、陷入无效的循环、或者因为某一步的错误（如工具返回非预期结果）而导致整个任务失败。如何让Agent具备强大的纠错能力和动态重规划能力，是最大的挑战之一。
    * <strong>具体表现：</strong> Agent卡在重复的“思考-行动”循环中；对工具的失败没有备用方案；过早地认为任务已完成。

2.  <strong>可靠且可复现的评估 (Reliable and Reproducible Evaluation):</strong>
    * <strong>挑战描述：</strong> 如何科学地评估一个Agent的性能极其困难。对于一个复杂的、开放式的任务（如“帮我规划一次为期一周的新加坡旅游”），没有唯一的正确答案。
    * <strong>具体表现：</strong>
      * <strong>评估指标难以定义：</strong> 仅看最终结果是否“好”是主观的。需要评估过程的效率（调用了多少次工具）、成本（花费了多少token）、鲁棒性（在不同干扰下的表现）等。
      * <strong>环境不可复现：</strong> 如果Agent使用了搜索引擎等动态工具，两次执行的结果可能完全不同，导致评估无法复现。
      * <strong>评估成本高：</strong> 目前最可靠的评估方式仍然是人工评估，但成本高昂且难以规模化。

3.  <strong>成本、延迟与可扩展性 (Cost, Latency, and Scalability):</strong>
    * <strong>挑战描述：</strong> 一个复杂的任务可能需要Agent进行数十次甚至上百次的LLM调用（每次思考、每次总结、每次决策都需要一次调用）。
    * <strong>具体表现：</strong>
      * <strong>高昂的API费用：</strong> 使用GPT-4等强大模型作为Agent大脑，一次复杂任务的成本可能高达数美元。
      * <strong>不可接受的延迟：</strong> 用户需要等待很长时间才能得到最终结果，因为整个过程是串行的。
      * <strong>服务扩展性差：</strong> 高成本和高延迟使得将这类复杂Agent大规模部署给海量用户变得不切实际。

4.  <strong>安全与可控性 (Safety and Controllability):</strong>
    * <strong>挑战描述：</strong> 赋予Agent调用工具的能力，本质上是赋予了它在数字世界甚至物理世界中“行动”的能力。
    * <strong>具体表现：</strong>
      * <strong>权限管理困难：</strong> 如何精确控制Agent的权限，防止它执行危险操作（如删除文件、发送恶意邮件）？
      * <strong>提示注入攻击（Prompt Injection）：</strong> 恶意用户或被Agent处理的外部数据（如网页内容）可能包含恶意指令，劫持Agent去执行非预期的任务。
      * <strong>不可预测性：</strong> Agent的自主性使其行为难以被完全预测，可能会产生意料之外的负面后果。



## 当一个 Agent 需要在真实或模拟环境中（如机器人、游戏）执行任务时，它与纯粹基于软件工具的 Agent 有什么本质区别？

当Agent从纯粹的软件环境（调用API、读写文件）进入到真实或模拟的物理环境（如机器人、游戏）时，我们称之为<strong>具身智能体（Embodied Agent）</strong>。这种转变引入了几个本质的区别，极大地增加了任务的复杂性。

<strong>本质区别主要体现在以下几个方面：</strong>

1.  <strong>感知与世界接地 (Perception & World Grounding):</strong>
    * <strong>软件Agent：</strong> 感知的是<strong>结构化的、符号化的</strong>信息（如API返回的JSON，数据库的表格）。
    * <strong>具身Agent：</strong> 感知的是<strong>非结构化的、高维的、充满噪声的</strong>传感器数据（如摄像头的像素流、激光雷达的点云）。它必须解决“符号接地”（Symbol Grounding）问题，即将语言中的概念（如“苹果”）与现实世界的物理实体（像素集合）对应起来。

2.  <strong>状态的可观测性 (State Observability):</strong>
    * <strong>软件Agent：</strong> 环境状态通常是<strong>完全可观测的</strong>（Full Observability）。通过API可以获取到所有需要的信息。
    * <strong>具身Agent：</strong> 环境状态是<strong>部分可观测的</strong>（Partial Observability）。机器人只能看到它面前的景象，无法知道房间另一边发生了什么。Agent必须基于不完整的观测历史来推断世界的状态。

3.  <strong>行动空间与不确定性 (Action Space & Uncertainty):</strong>
    * <strong>软件Agent：</strong> 行动空间是<strong>离散的、确定的</strong>。调用一个API要么成功要么失败，结果是可预测的。
    * <strong>具身Agent：</strong> 行动空间通常是<strong>连续的、随机的</strong>。控制机器人手臂移动一个精确的距离，会因为电机误差、摩擦力等因素而存在不确定性。每个行动的结果都需要通过传感器反馈来确认。

4.  <strong>实时性与反馈循环 (Real-time & Feedback Loop):</strong>
    * <strong>软件Agent：</strong> 交互是<strong>回合制的、异步的</strong>。Agent可以花很长时间思考，然后调用工具，等待结果。
    * <strong>具身Agent：</strong> 必须在<strong>实时（real-time）</strong>中运行。它需要持续地感知、决策和行动，以应对动态变化的环境。反馈循环是即时的、连续的。

5.  <strong>安全与不可逆性 (Safety & Irreversibility):</strong>
    * <strong>软件Agent：</strong> 错误行动的后果通常是<strong>可逆的、有限的</strong>。一个失败的API调用可以重试，最坏的情况可能是数据错误。
    * <strong>具身Agent：</strong> 错误行动的后果可能是<strong>物理性的、不可逆的、甚至是危险的</strong>。一个机器人错误的动作可能会打碎一个杯子、损坏自身或对人类造成伤害。因此，安全是具身Agent的首要考虑。



## 如何确保一个 Agent 的行为是安全、可控且符合人类意图的？在 Agent 的设计中，有哪些保障对齐方法？

确保Agent的安全、可控和对齐是Agent技术能够被信任和应用的前提，这是一个系统性工程，需要在多个层面进行设计。

主要的保障对齐方法包括：

1.  <strong>核心模型的对齐（Core Model Alignment）：</strong>
    * <strong>基础：</strong> Agent的大脑是一个LLM，因此，这个LLM本身必须是高度对齐的。
    * <strong>方法：</strong> 使用如<strong>RLHF（从人类反馈中强化学习）</strong>、<strong>DPO（直接偏好优化）</strong>、<strong>Constitutional AI（宪法AI）</strong>等技术，对基础LLM进行微调，使其遵循“有用、诚实、无害”的原则，这是所有安全措施的基石。

2.  <strong>工具和权限的严格管理（Tool and Permission Scrutiny）：</strong>
    * <strong>原则：</strong> 最小权限原则（Principle of Least Privilege）。只给Agent完成其任务所必需的最少的工具和权限。
    * <strong>方法：</strong>
      * <strong>工具白名单：</strong> 明确列出Agent可以调用的安全工具，而不是让它任意调用。
      * <strong>权限控制：</strong> 对文件系统、数据库、API的访问进行严格的读/写/执行权限控制。
      * <strong>资源限制：</strong> 限制Agent的计算资源、API调用次数和执行时间，防止其失控或造成资源滥用。

3.  <strong>人类在环（Human-in-the-Loop, HITL）：</strong>
    * <strong>原则：</strong> 对于高风险或不可逆的操作，必须有人类监督和确认。
    * <strong>方法：</strong>
      * <strong>操作确认：</strong> 在执行如“删除文件”、“发送邮件”、“执行金融交易”等敏感操作前，Agent必须生成一个执行计划，并暂停等待人类用户的明确批准。
      * <strong>监督与干预：</strong> 人类可以实时监控Agent的行为轨迹，并随时暂停、修改或终止其任务。

4.  <strong>执行环境沙箱化（Sandboxed Execution Environment）：</strong>
    * <strong>原则：</strong> 将Agent的执行环境与宿主系统隔离。
    * <strong>方法：</strong> 让Agent生成的代码或命令在一个受控的沙箱（如Docker容器、虚拟机）中执行。这样即使Agent被劫持或产生恶意代码，其破坏范围也被限制在沙箱内部，不会影响到外部系统。

5.  <strong>明确的规则与护栏（Explicit Rules and Guardrails）：</strong>
    * <strong>方法：</strong> 除了LLM内在的对齐，可以在Agent的控制逻辑中加入硬编码的规则或“护栏”。例如，可以设置一个正则表达式过滤器，禁止Agent生成或执行包含特定危险命令（如 `rm -rf /`）的指令。

6.  <strong>持续的红队测试与审计（Continuous Red Teaming and Auditing）：</strong>
    * <strong>方法：</strong>
      * <strong>红队测试：</strong> 组织专门的团队，像黑客一样，从各种角度（如提示注入、越狱、滥用工具）来攻击Agent，主动发现其安全漏洞和对齐缺陷。
      * <strong>行为审计：</strong> 详细记录Agent所有的思考链、工具调用和最终输出，进行事后审计，分析失败案例和非预期行为，并据此迭代改进安全设计。

