# LangChain & LangGraph



## LangChain 和 LangGraph 的区别?

>LangChain 负责提供构建 AI 应用的原子能力（如模型调用、Prompt 模板、工具封装）；而 LangGraph 负责定义这些组件之间的流转逻辑（如循环、分支、状态持久化）。LangChain 适合线性任务，LangGraph 适合复杂的、有状态的 Agent 系统。

### 一、 核心区别对比表

| 维度     | LangChain                              | LangGraph                                                    |
| -------- | -------------------------------------- | ------------------------------------------------------------ |
| 核心抽象 | Chain (链)：线性的 DAG（有向无环图）。 | Graph (图)：支持循环的 FSM（有限状态机）。                   |
| 控制流   | 静态、预先定义好路径。                 | 动态、可根据 LLM 输出实时决定下一步。                        |
| 状态管理 | 弱。通常只传递简单的输入输出字典。     | 强。内置 State 对象，支持跨节点的状态累积和更新。            |
| 持久化   | 较难实现复杂的中断与恢复。             | 原生支持。通过 Checkpointer 实现会话暂停、时间旅行和断点续传 |
| 适用场景 | 简单的问答、RAG、数据提取。            | 自主 Agent、多步推理、人机协同、多 Agent 协作。              |

### 二、 为什么需要从 LangChain 进化到 LangGraph？

1. 解决“循环”难题

  - LangChain 局限：传统的 Chain 像流水线，一旦开始就不能回头。如果 Agent 需要“写代码 -> 运行报错 -> 根据错误修改代码”，这种循环在 LangChain 中需要用极其复杂的递归逻辑来实现。
  - LangGraph 优势：天生支持环（Cycle）。你可以轻松定义 Node A -> Node B -> Node A 的路径，完美契合 ReAct 等 Agent 模式。

2. 解决“状态共享”难题

  - LangChain 局限：在长链路中，如果后面的节点想用前面节点的中间结果，通常需要把数据塞进一个大字典里层层传递，容易出错。
  - LangGraph 优势：提供一个全局的 Shared State。任何节点都可以读取或更新这个状态（例如：messages 列表会自动追加，而不是覆盖），大大简化了上下文管理。

3. 解决“生产级可靠性”难题

  - LangChain 局限：如果服务在 Chain 执行一半时崩溃，很难从断点恢复。
  - LangGraph 优势：内置 Checkpointer（如项目中使用的 PostgresSaver）。它能把每一步的状态存入数据库。即使服务重启，也能根据 thread_id 找回之前的进度，继续执行。

### 三、 项目中的实际体现

在你的 langchain-demo 项目中，这两种范式并存：
**LangChain 范式：chain.py**

```python
# 线性处理：输入 -> Prompt -> LLM -> 输出
chain = {"question": RunnablePassthrough()} | prompt | llm | output_parser
```

LangGraph 范式：使用Graph API.py

```python
# 循环处理：LLM 决定是调工具还是结束
agent_builder.add_conditional_edges("llm_call", should_continue, ["tool_node", END])
agent_builder.add_edge("tool_node", "llm_call") # 关键：这里形成了环 
```

### 四、 总结与建议

- 如果你只是做一个简单的客服问答或文档总结，用 LangChain 足够，开发快、成本低。
- 如果你要做一个能自主规划、会写代码、需要人类审批的复杂 Agent，必须用 LangGraph。

在大厂实践中，通常是“两者结合”： 使用 LangChain 提供的各种 Tool 和 Model 组件，然后放入 LangGraph 的 Node 中进行编排。LangGraph 是大模型应用从“Demo”走向“Production”的必经之路。



## 短期记忆和长期记忆

> 我主要从信息的作用域区分短期记忆和长期记忆。
>
> **短期记忆是同一个会话线程内的上下文和任务状态**，比如之前的对话、工具调用结果、当前任务进度。它让 Agent 知道“刚才聊了什么、现在做到哪一步”。
>
> 在 LangChain / LangGraph 中，可以用 **State + Checkpointer + `thread_id`** 实现。State 承载消息和业务状态，Checkpointer 保存状态检查点，后续调用使用同一个 `thread_id`，就能接着之前的状态继续执行。生产环境用数据库持久化，因此短期记忆也可以在进程重启后恢复。“短期”指线程内的作用域，不代表只能放在内存里。[官方说明](https://docs.langchain.com/oss/python/langchain/short-term-memory)
>
> **长期记忆是跨会话仍然有价值的信息**，比如用户偏好、稳定事实，以及历史任务中积累的经验。在 LangGraph 中，可以通过 **Store + namespace + key** 管理。namespace 通常按租户、用户和记忆类型组织，key 标识具体记忆。换了新的 `thread_id`，仍然可以通过用户身份读取相关记忆，再加入当前模型上下文。[官方说明](https://docs.langchain.com/oss/python/langchain/long-term-memory)
>
> 举个例子，用户这次旅行的预算、已经查过哪些酒店，可以放在当前线程的短期状态里；用户明确要求记住的“不吃辣”，可以提炼成长期偏好，下次新建会话规划旅行时再使用。
>
> 工程上，我会重点处理两件事：**短期记忆控制上下文长度**，通过裁剪和摘要保留重要信息；**长期记忆控制写入和召回质量**，只保存有复用价值的信息，做好去重、更新和过期处理，再按当前问题检索。还要用服务端认证身份做用户隔离，避免串读。

说完先停，等面试官追问。下面四个问题最值得准备：

**① “短期记忆已经落数据库了，为什么还叫短期？”**

因为**持久化解决“数据会不会丢”，作用域解决“哪些会话能用”**。同一个线程隔天恢复，仍然属于短期记忆；让新线程复用用户偏好，才是跨线程的长期记忆。

**② “长期记忆一定要用向量数据库吗？”**

不一定。用户语言偏好等结构化信息，可以按 key 精确读取；历史经验或文本片段，才可能需要语义检索。**向量检索是召回手段，长期记忆还包括提取、写入、更新和删除。**

**③ “把历史聊天全部存下来，不就实现记忆了吗？”**

存下来只完成了保存。还要决定哪些信息值得长期保留、如何处理冲突，以及何时召回并放进模型上下文。否则保存越多，可能召回的噪声越多。

**④ “怎么避免不同用户的记忆串了？”**

`thread_id` 要校验归属；长期记忆的 namespace 要基于服务端认证的租户和用户身份构造，并做访问控制。**namespace 是数据组织方式，本身不等于完整的权限机制。**

你贴的材料里，有一句尤其要修正：**短期记忆不是“只服务一次 graph 执行”**，它可以覆盖同一个线程的多次调用。把这一点讲清楚，比背一串 Memory 类名更能体现你理解了机制。[线程与检查点机制](https://docs.langchain.com/oss/python/langgraph/persistence)



