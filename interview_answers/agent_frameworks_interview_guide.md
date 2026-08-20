# 常见 Agent 开发框架与高频面试题

> 核验时间：2026-08-10。本文按各框架官方文档的当前口径整理，重点是架构原理、生产可靠性和面试回答，不是 API 安装手册。

> 项目映射场景：企业机房、服务器与办公电脑故障咨询，维修工程师匹配和上门预约。

## 0. 先记住三句话

1. **LangGraph 主要解决“流程怎样可靠推进和恢复”**：状态图、循环、并行、Checkpoint、HITL 和长期运行。
2. **LlamaIndex 主要解决“数据怎样进入 Agent 并成为可靠上下文”**：解析、切块、索引、检索、重排和答案合成。
3. 生产 Agent 通常不是完全自治，而是 **确定性 Workflow 骨架 + 少量 LLM 决策节点 + 受控工具执行层**。

因此，LangGraph 与 LlamaIndex 经常组合使用：LlamaIndex 的 Retriever 或 QueryEngine 可以成为 LangGraph 的一个 Node 或 Tool。

## 1. 常见框架定位与选型

| 框架 | 核心定位与抽象 | 适合场景 | 面试重点与局限 |
|---|---|---|---|
| [LangChain / LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) | LangChain 提供模型、工具和高层 Agent；LangGraph 提供 `State`、`Node`、`Edge`、`Reducer`、Checkpoint 等底层运行时 | 长流程、循环、并行、人工审批、故障恢复 | 不要混淆二者；LangGraph 控制力强，但状态、路由和异常策略要自行设计 |
| [LlamaIndex / LlamaAgents](https://developers.llamaindex.ai/python/framework/) | 数据与 RAG 优先：`Document`、`Node`、`Index`、`Retriever`、`QueryEngine`、Workflow | 企业知识库、文档 Agent、复杂 RAG | 强项是数据链路；如果只需要简单业务编排，不一定优先选择 |
| [OpenAI Agents SDK](https://developers.openai.com/api/docs/guides/agents/quickstart) | `Agent`、`Runner`、Tool、Handoff、Guardrail、Session、Tracing | 工具 Agent、客服分流、OpenAI 原生工具集成 | 区分 Handoff 与 Agents-as-Tools；具备 Guardrails、HITL、Tracing 和 Evals |
| [CrewAI](https://docs.crewai.com/en/introduction) | 角色化的 Agent、Task、Crew、Process，加上结构化 Flow | 研究、写作、分析、业务自动化原型 | Crew 偏自主协作，Flow 偏确定性控制；不要为了“像团队”而过度拆 Agent |
| [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview/) | Agent、Session、Middleware、Harness、类型安全图工作流、Checkpoint/HITL | .NET、Azure、企业系统和多 Agent 工作流 | 微软当前将其定位为 AutoGen 与 Semantic Kernel 的直接后继 |
| [PydanticAI](https://pydantic.dev/docs/ai/core-concepts/agent/) | 类型安全 Agent、依赖注入、Pydantic 输出校验、Tools 与 Evals | Python 后端、强结构化输出、重视测试 | 类型体验好，但不是 RAG-first 或角色团队框架 |
| [Google ADK](https://adk.dev/agents/) | Agent、图/多 Agent Workflow、Artifact、Callback、A2A、Session/State/Memory | Google Cloud、Gemini、跨服务 Agent | 状态模型清楚、语言覆盖广；需关注不同语言的能力差异 |
| [Haystack](https://docs.haystack.deepset.ai/docs/agents) | Component、Pipeline、DocumentStore、Retriever、Agent | 搜索、RAG、可组合数据流水线 | Pipeline-first，检索与搜索工程能力较强 |

### 1.1 当前时效口径：AutoGen 与 Semantic Kernel

- AutoGen 的高频概念仍值得掌握：AgentChat、Core Runtime、Team、Termination、Message 和 Handoff。
- Semantic Kernel 的高频概念仍值得掌握：Kernel、Plugin、KernelFunction、Filter 和 AgentThread。
- 但回答新项目选型时，应补充：微软官方现在将 Microsoft Agent Framework 定义为二者的直接后继，并提供迁移指南。

参考资料：[AutoGen → Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/migration-guide/from-autogen/)、[Microsoft Agent Framework Overview](https://learn.microsoft.com/en-us/agent-framework/overview/)。

## 2. 框架之外必须掌握的 Agent 原理

### 2.1 最小 Agent Loop

```text
用户输入
  → 组装 Instructions、State、Memory 与可用 Tools
  → LLM 判断是回答还是调用工具
  → Schema、权限、风险与参数校验
  → 执行 Tool
  → 将 Tool Result 作为 Observation 回填
  → 继续推理，或命中终止条件后输出答案
```

面试回答至少要包含：

- 模型只**建议**工具调用，应用层才拥有执行权。
- Tool 的名称、描述、参数 Schema 会直接影响模型的选择和参数正确率。
- 循环必须有最大步数、超时、预算、取消和终止条件。
- Prompt 是软约束；权限、租户隔离、幂等和事务必须由确定性代码保证。

### 2.2 Agent 与 Workflow

| Agent | Workflow |
|---|---|
| 路径由模型根据上下文动态决定 | 路径由代码、图或规则显式定义 |
| 适合开放问题、探索、动态工具选择 | 适合订单、审批、预约、退款等可审计流程 |
| 灵活但存在概率性、费用和轨迹不稳定 | 稳定、易测试，但对未知情况适应性较弱 |

最佳实践不是二选一，而是：

> 用 Workflow 控制身份、权限、状态、审批和副作用；仅在无法方便写成规则的判断点调用 Agent。

### 2.3 State、Memory 与 RAG

- **State**：当前流程运行到哪里、有哪些中间变量和待执行任务。
- **短期记忆**：当前 thread/session 的消息、摘要和临时偏好。
- **长期记忆**：跨会话保存的用户偏好、历史事件和可复用经验。
- **RAG**：根据当前问题从外部知识源检索事实证据。
- **业务事实源**：数据库中的订单、预约、库存等权威状态，不能被摘要或 Memory 覆盖。

即使 Memory 和 RAG 都使用向量库，它们的业务语义仍然不同。

### 2.4 Function Calling 与 MCP

- Function Calling/Tool Calling：模型根据传入的 Schema 生成结构化工具调用，应用负责执行。
- MCP：标准化 Tool、Resource 等能力的发现、连接和调用方式。
- MCP 不会自动解决鉴权、权限、Prompt Injection、参数校验、沙箱和人工审批。

## 3. LangGraph 面试重点

### 3.1 State、Node、Edge、Reducer

- `State`：图在当前时刻的共享状态快照。
- `Node`：读取 State，执行 LLM、工具或普通代码，返回部分状态更新。
- `Edge`：决定下一步执行哪个 Node，可固定、条件跳转或并行分发。
- `Reducer`：决定 State 某个字段的新旧值如何合并；未定义时通常采用覆盖语义。

节点应返回 partial update，而不是依赖不透明的原地修改。

高频并发陷阱：多个并行 Node 同时写同一 State key 时，需要定义合适的 Reducer，否则可能出现并发更新冲突。若业务依赖固定顺序，不能依赖并行完成顺序，应携带排序键并最终显式排序。

参考：[LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)、[Use Graph API](https://docs.langchain.com/oss/python/langgraph/use-graph-api)。

### 3.2 Superstep 执行语义

LangGraph 采用类似 Pregel/Bulk Synchronous Parallel 的 superstep 模型：

1. 选择本轮应执行的 Node。
2. 本轮 Node 可以并行执行，并读取本轮开始时可见的 State。
3. 本轮结束后统一通过 Reducer 合并和提交更新。
4. 本轮产生的更新在下一轮才对其他 Node 可见。

Checkpoint 的完整恢复点通常也是 superstep 边界，而不是任意一行源代码。

### 3.3 Conditional Edge、Command 与 Send

- Conditional Edge：根据 State 选择下一节点，主要负责路由。
- `Command`：在同一个 Node 返回中同时执行 State update 和 `goto`。
- `Send`：动态创建多份不同输入，适合 fan-out、map-reduce 和并行子任务。

不要无意中为同一 Node 同时配置静态出边和动态跳转，否则两条路径都可能执行。

### 3.4 Checkpoint、thread_id 与 Durable Execution

配置 Checkpointer 后，LangGraph 按 `thread_id` 保存状态快照，可以支持：

- 失败恢复；
- 多轮会话；
- Human-in-the-loop；
- Time travel、Replay 和 Fork；
- 查看、修改历史状态。

`thread_id` 相当于持久游标：使用同一个 ID 才能加载和恢复同一条执行历史。

参考：[LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)。

### 3.5 Durable 不等于 Exactly Once

Checkpoint 能减少重复工作，但不能保证外部副作用严格只执行一次。以下情况可能重新执行 Node：

- Node 在完成外部操作后、保存结果前崩溃；
- Interrupt 恢复；
- 从旧 Checkpoint Replay；
- 失败重试或超时重试。

因此，发送通知、创建预约、扣款、写第三方系统必须使用：

- idempotency key；
- 数据库唯一约束或去重表；
- Outbox/Inbox；
- 执行前检查已有结果；
- 清晰的事务边界。

### 3.6 Interrupt / HITL

`interrupt(payload)` 可以动态暂停图；恢复时使用相同 `thread_id` 并传入 `Command(resume=value)`。

最大面试陷阱：

> 恢复不是从 `interrupt()` 的下一行继续，而是可能从当前 Node 开头重新执行。

所以 Interrupt 前的副作用也必须幂等，或移动到 Interrupt 后面。不要用宽泛的 `try/except` 吞掉 Interrupt 使用的特殊异常。

参考：[LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)。

### 3.7 Checkpointer 与 Store

| Checkpointer | Store |
|---|---|
| 保存完整 Graph State 快照 | 保存应用自定义数据 |
| 主要按 `thread_id` 隔离 | 可按 `tenant_id/user_id` 等 namespace 隔离 |
| 对应线程内短期记忆、恢复和 Time Travel | 对应跨线程长期记忆 |

对话消息无限追加不是良好的 Memory 设计；仍需 trim、summary、delete，并定义记忆的写入、更新和遗忘策略。

### 3.8 Graph API 与 Functional API

- Graph API：显式 State、Node、Edge，易于可视化和检查，适合复杂流程。
- Functional API：使用普通 `if/for/function` 和 `@entrypoint/@task`，改造现有代码更轻。
- 二者共用 LangGraph 运行时，可组合使用。

## 4. LlamaIndex 面试重点

### 4.1 数据与 RAG 主链路

```text
数据源
  → Document
  → Transformations / NodeParser
  → Node
  → Index / Vector Store
  → Retriever
  → Node Postprocessor / Reranker
  → ResponseSynthesizer
  → QueryEngine
  → Response
```

### 4.2 Document 与 Node

- `Document`：PDF、API 返回、数据库记录等原始逻辑数据容器。
- `Node`：由 Document 切分出的检索单元，可以是文本、图像或其他内容。
- Node 通常继承 Document 的 metadata，并参与 embedding、索引和检索。

参考：[LlamaIndex Documents / Nodes](https://developers.llamaindex.ai/python/framework/module_guides/loading/documents_and_nodes/)。

### 4.3 Index、Vector Store 与 Retriever

- `Index`：LlamaIndex 对“怎样组织和检索数据”的上层抽象。
- Vector Store：保存 embedding 并执行相似度搜索的后端。
- `VectorStoreIndex`：使用向量检索的 Index 实现，可以连接不同向量数据库。
- `Retriever`：根据 Query 返回相关 Node，不负责生成最终答案。

因此，LlamaIndex 不是向量数据库，Index 也不等于向量数据库。

### 4.4 Retriever、QueryEngine 与 ChatEngine

- Retriever：只负责“找相关上下文”。
- QueryEngine：通常组合 Retriever、Postprocessor 和 ResponseSynthesizer，返回端到端回答。
- ChatEngine：进一步管理多轮对话。

QueryEngine 可以包装成 Agent Tool，让一条完整 RAG Pipeline 成为 Agent 的一个能力。

参考：[LlamaIndex Query Engine](https://developers.llamaindex.ai/python/framework/module_guides/deploying/query_engine/)。

### 4.5 ResponseSynthesizer 模式

| 模式 | 特点 | 主要权衡 |
|---|---|---|
| `compact` | 尽量合并多个 chunk 后生成 | LLM 调用较少，通常是常用默认方案 |
| `refine` | 逐个 Node 修订已有答案 | 信息利用细致，但调用次数和延迟更高 |
| `tree_summarize` | 分组摘要后递归合并 | 适合长文档和全局总结 |
| `simple_summarize` | 截断后单次生成 | 快，但可能丢失上下文 |
| `no_text` | 只返回检索结果，不调用 LLM | 适合调试 Retriever |

回答时不要只背模式名称，要说明质量、调用次数、延迟和成本的权衡。

### 4.6 Agent 类型

- `FunctionAgent`：使用模型原生 Function/Tool Calling。
- `ReActAgent`：通过 Thought → Action → Observation 循环使用工具。
- `CodeActAgent`：通过生成代码组合工具，灵活但具有任意代码执行风险，必须放在可靠沙箱中。

### 4.7 Event-driven Workflow

当前 LlamaIndex/LlamaAgents Workflow 是事件驱动、Step-based 的控制流：

- Step 接收一种 Event 并返回另一种 Event；
- Event 的类型注解隐式形成连接关系；
- `if` 返回不同 Event 实现分支；
- 返回由先前 Step 消费的 Event 实现循环；
- 返回事件列表可以 fan-out，下游汇总可以 fan-in；
- 支持 async、流式事件、HITL、共享 State 和 Durable Workflow。

参考：[LlamaAgents Workflows](https://developers.llamaindex.ai/python/llamaagents/workflows/)。

### 4.8 Context、State、Memory 与 Resource

- `Context/ctx.store`：当前 Workflow 运行的共享 State。
- Memory：保存与检索对话历史及长期记忆。
- Resource：模型客户端、Index、数据库连接、配置等运行时依赖。

可序列化业务状态放 State；客户端、连接和模型对象放 Resource。并发修改 State 时应使用框架提供的原子编辑机制，避免读改写竞争。

### 4.9 RAG 评估必须拆成两层

1. Retrieval：Recall、Hit Rate、MRR、Precision、排序质量。
2. Generation：Faithfulness、Answer Relevance、Correctness、引用准确率。

只评最终答案无法判断是“没有检索到证据”还是“检索到了但模型没有正确使用”。

## 5. 其他框架重点

### 5.1 OpenAI Agents SDK

核心抽象：Agent、Runner、Tool、Handoff、Session、Guardrail、Tracing。

- Handoff：专家 Agent 接管后续对话。
- Agents-as-Tools：Manager 保持最终回答控制权，只把专家当作受限能力调用。
- Guardrail：自动校验输入、输出或 Tool 行为。
- Human Review：在取消、编辑、Shell、敏感 MCP Tool 等副作用前暂停审批。
- Tracing：记录模型调用、工具、Handoff、Guardrail 和自定义 Span。
- Evals：评估 Tool 选择、Handoff、策略遵循和完整执行轨迹。

参考：[Orchestration and Handoffs](https://developers.openai.com/api/docs/guides/agents/orchestration)、[Guardrails and Human Review](https://developers.openai.com/api/docs/guides/agents/guardrails-approvals)、[Observability](https://developers.openai.com/api/docs/guides/agents/integrations-observability)。

### 5.2 CrewAI

- Agent：role、goal、tools 等角色能力。
- Task：任务描述、预期输出、上下文和 Guardrail。
- Crew：组织多个 Agent 与 Task。
- Sequential Process：按预定义顺序执行。
- Hierarchical Process：Manager 动态分配、审查和协调任务。
- Flow：使用显式事件、State、Router、循环和持久化组织可靠流程。

生产回答建议：**Flow 作为主骨架，Crew 作为局部需要自主协作的能力**。

### 5.3 Microsoft Agent Framework

- Agent：使用模型、Tools/MCP 处理输入并生成回复。
- Session：管理多轮状态。
- Middleware：鉴权、审计、重试、治理等横切能力。
- Harness：包含规划、Todo、Context 压缩、文件和 Memory 的长任务 Agent。
- Workflow：以 Executors、Edges 和类型安全消息路由组合 Agent 与函数，支持并行、Checkpoint 和 HITL。

当前面试最好说明：AutoGen 与 Semantic Kernel 的思想仍有价值，但新项目应主动对比 Microsoft Agent Framework。

### 5.4 PydanticAI

核心优势是 Python 类型安全：

- `Agent[DepsType, OutputType]`；
- typed dependency injection；
- Pydantic Tool 参数和输出校验；
- 校验失败后可向模型反馈并有限重试；
- Usage Limit、Deferred Tool Approval、OpenTelemetry 和 Pydantic Evals；
- 复杂流程可使用 Pydantic Graph 或外部 Durable Execution 引擎。

适合强结构化后端和重视单元测试、静态检查的团队。

### 5.5 Google ADK

- Agent 的基础仍是 Model、Instructions 和 Tools。
- 支持图工作流、Sequential/Loop/Parallel 模板、多 Agent、Artifacts、Callbacks 和 A2A。
- `Session` 表示当前对话线程，包含 Events。
- `State` 是当前 Session 内的数据。
- `Memory` 是可搜索、跨 Session 的长期信息。

参考：[ADK Agents](https://adk.dev/agents/)、[Session、State 与 Memory](https://adk.dev/sessions/)。

## 6. 高频面试题与回答关键词

### 6.1 通用原理

1. **Agent 与普通 Chatbot 有什么区别？**  
   Agent 有目标、状态、工具、决策循环和终止条件；Chatbot 可能只是一次模型调用。

2. **Agent 与 Workflow 怎么选？**  
   开放路径交给 Agent；明确且高风险的路径交给 Workflow。生产系统常是确定性骨架加局部 Agent。

3. **ReAct 与 Plan-and-Execute 的区别？**  
   ReAct 每轮观察后再决定，适应性强；Plan-and-Execute 先规划，便于并行，但需要计划校验和重新规划。

4. **工具调用完整生命周期是什么？**  
   暴露 Schema → 模型生成调用 → 参数/权限校验 → 审批或执行 → 回填结果 → 继续或结束。

5. **为什么 Tool 实现正确，Agent 仍可能调用错误？**  
   Tool name、description、参数约束、相似 Tool 的区分度和上下文都会影响模型决策。

6. **如何防止 Agent 无限循环？**  
   最大步数、最大工具次数、超时、Token/费用预算、重复轨迹检测、明确终止条件和人工接管。

7. **多 Agent 为什么不一定更好？**  
   会增加路由错误、上下文复制、延迟、成本和调试难度。只有需要上下文隔离、独立权限、并行专家或角色接管时才拆分。

8. **Supervisor、Handoff、Router、Agents-as-Tools 如何选？**  
Agents-as-Tools 是一种集中控制实现：主 Agent 通过 Tool Calling 调用子 Agent；LangGraph 自定义 Workflow 也可以把 Supervisor 做成决策节点，再用条件边或`Command`路由到专业节点/子图。Handoff 适合专家接管用户对话；Router 适合一次分类和分发。本机房与电脑维修项目选择 LangGraph 节点/子图编排，不采用 Agents-as-Tools。

### 6.2 可靠性与安全

9. **Checkpoint 是否保证 Exactly Once？**  
   不保证。Checkpoint 管运行状态，外部副作用仍需幂等键、唯一约束、Outbox/Inbox 和去重。

10. **工具超时后是否可以直接重试？**  
    先判断操作是否幂等、是否可能已经成功。查询可有限重试；扣款、通知、预约等需幂等键和结果查询。

11. **如何实现 Human-in-the-loop？**  
    在敏感副作用前持久化状态并暂停，向人展示动作和参数；批准、拒绝或修改后用同一运行标识恢复。

12. **如何防止 Prompt Injection？**  
    把网页、文档、Tool Result 和 Memory 都视为不可信数据；隔离系统指令、最小化工具权限、验证参数、审批高风险动作并保留 Trace。

13. **模型是否可以直接操作数据库？**  
    不应。模型只生成结构化意图；业务服务重新做身份、租户、权限、参数、状态机、事务和幂等校验。

### 6.3 RAG 与评估

14. **为什么大 Context Window 不能完全替代 RAG？**  
    RAG 提供新鲜度、权限过滤、引用、较低成本和更可控的证据选择；长上下文仍会有噪声和注意力分散。

15. **如何定位 RAG 效果差？**  
    先检查问题改写和召回，再检查重排和 Context Packing，最后检查答案是否忠实使用证据。

16. **Observability 与 Evaluation 有什么区别？**  
    Observability 回答发生了什么、哪里慢或失败；Evaluation 回答输出质量是否达标、版本是否变好。

17. **Agent 应评估哪些指标？**  
    任务成功率、Tool 选择与参数准确率、轨迹、冗余调用、回合数、延迟、费用、重试率、人工接管率和安全违规率。

18. **LLM-as-Judge 有什么问题？**  
    存在 Judge 偏差、不稳定和额外成本；应使用固定 Rubric、校准样本、确定性指标和人工抽检组合验证。

### 6.4 LangGraph 专项

19. **LangChain 和 LangGraph 的区别？**  
    LangChain 是高层模型/工具/Agent 框架；LangGraph 是底层状态化编排运行时。

20. **为什么 LangGraph 不只是普通 DAG？**  
    Agent 需要循环、动态路由、并行 fan-out、跨轮状态、人工暂停和恢复。

21. **Reducer 为什么关键？**  
    Node 返回 partial update，Reducer 决定新旧值如何合并，尤其解决并行写同一字段的问题。

22. **Superstep 是什么？**  
    同一轮可并行执行的一组 Node；它们读取轮初状态，更新在轮末统一提交。

23. **Interrupt 恢复时从哪里继续？**  
    可能从当前 Node 开头重跑，因此 Interrupt 前的副作用必须幂等。

24. **Checkpointer 与 Store 的区别？**  
    Checkpointer 保存 thread 内完整 Graph State；Store 保存跨 thread 的长期应用数据。

### 6.5 LlamaIndex 专项

25. **Document 与 Node 的区别？**  
    Document 是原始逻辑数据单位；Node 是参与索引和检索的细粒度 chunk。

26. **Index、Vector Store、Retriever 的关系？**  
    Index 是上层组织/检索抽象；Vector Store 是向量后端；Retriever 通过 Index 或自定义逻辑返回相关 Node。

27. **Retriever 与 QueryEngine 的区别？**  
    Retriever 只找上下文；QueryEngine 组合检索、后处理和答案合成。

28. **Workflow 没有显式 Edge，Step 如何连接？**  
    Step 的 Event 输入和返回类型形成隐式连接，并可在运行前进行图校验。

29. **Context 与 Memory 的区别？**  
    Context 是 Workflow 的运行状态；Memory 保存和检索过去的对话或长期信息。

30. **为什么 RAG 必须分开评估 Retrieval 与 Generation？**  
    否则无法区分是证据没召回，还是模型没有忠实使用已召回证据。

## 7. 场景化选型速记

- 简单工具 Agent：LangChain `create_agent`、OpenAI Agents SDK、PydanticAI。
- 长流程、状态机、暂停恢复：LangGraph、Microsoft Agent Framework Workflow。
- 企业文档、RAG、知识 Agent：LlamaIndex、Haystack。
- Python 角色协作和快速业务自动化：CrewAI，建议 Flow 为骨架、Crew 为局部能力。
- Microsoft/.NET/Azure：Microsoft Agent Framework。
- Google/Gemini/A2A：Google ADK。

一个典型企业 Agent 可以这样组合：

```text
用户请求
  → 身份、租户和风险检查
  → LangGraph / MAF Workflow 负责路由与持久状态
  → LlamaIndex / Haystack 负责检索与重排
  → Agent 选择只读 Tool 或提出业务动作
  → Pydantic/业务服务校验 Schema、权限和状态机
  → 高风险动作进入人工审批
  → 幂等执行并持久化结果
  → Trace、Evaluation 和业务指标持续监控
```

## 8. 面试回答模板

遇到“请设计一个 Agent 系统”时，可以固定按以下七步回答：

1. 目标、成功标准和不允许 Agent 自主执行的边界。
2. 哪些步骤是确定性 Workflow，哪些节点需要 LLM 判断。
3. State、短期 Memory、长期 Memory、RAG 和业务事实源如何隔离。
4. Tool Contract、Schema、权限、超时和错误语义。
5. Checkpoint、重试、幂等、人工审批和补偿方案。
6. Trace、离线评测、线上监控和回归数据集。
7. 模型路由、并发、延迟、Token 和费用预算。

一句话收尾：

> 我不会把可靠性寄托在 Prompt 上；LLM 负责理解与建议，Workflow 和业务代码负责权限、状态、事务、幂等与审计。

## 9. 主要官方资料

- [LangGraph Overview](https://docs.langchain.com/oss/python/langgraph/overview)
- [LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LlamaIndex Framework](https://developers.llamaindex.ai/python/framework/)
- [LlamaIndex Documents / Nodes](https://developers.llamaindex.ai/python/framework/module_guides/loading/documents_and_nodes/)
- [LlamaIndex Query Engine](https://developers.llamaindex.ai/python/framework/module_guides/deploying/query_engine/)
- [LlamaAgents Workflows](https://developers.llamaindex.ai/python/llamaagents/workflows/)
- [OpenAI Agents SDK Quickstart](https://developers.openai.com/api/docs/guides/agents/quickstart)
- [OpenAI Agents SDK Orchestration](https://developers.openai.com/api/docs/guides/agents/orchestration)
- [CrewAI Documentation](https://docs.crewai.com/)
- [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview/)
- [PydanticAI Agents](https://pydantic.dev/docs/ai/core-concepts/agent/)
- [Google ADK Agents](https://adk.dev/agents/)
- [Haystack Agents](https://docs.haystack.deepset.ai/docs/agents)
