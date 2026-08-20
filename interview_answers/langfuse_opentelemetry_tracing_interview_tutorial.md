# Langfuse + OpenTelemetry：Agent Trace 方案与面试重点

> 资料快照：2026-08-12
>  
> 贯穿案例：企业机房、服务器与办公电脑故障咨询及上门维修预约 Agent
>  
> 定位：目标方案、落地清单与面试讲法，不是当前仓库已经完成生产集成的证明
>  
> 版本提醒：Langfuse 与 OpenTelemetry GenAI 语义约定变化较快，生产落地前必须重新核对兼容矩阵并锁定版本

## 0. 先给结论

一句话概括：

> OpenTelemetry 负责统一埋点、上下文传播、采集、处理和多后端分发；Langfuse 负责把其中的 LLM/Agent 调用解释成 generation、agent、tool、retriever、token、cost、prompt、score、session 等面向 AI 应用的可观测对象。二者不是二选一，而是“OTel 做骨架，Langfuse 做 GenAI 分析面”。

对于本项目，推荐的生产目标是：

1. 一次用户对话轮次或一次 Agent run 对应一个 Trace；
2. Router、Supervisor、专业 Agent、RAG、模型、工具和数据库事务分别形成有业务意义的 Span；
3. 同步调用通过 W3C `traceparent` 维持父子关系，Outbox/消息消费通过上下文传播或 Span Link 表达异步因果；
4. 通用 APM 接收完整的 HTTP、队列、数据库和基础设施链路，Langfuse 接收保留根节点的 LLM/Agent 相关子树；
5. Prompt、Completion、工具参数、检索文档默认不全量采集，先脱敏、截断和分级授权；
6. 同一个 Langfuse 项目只保留一条上报出口，避免 SDK 直传与 Collector 转发同时启用；
7. 错误、慢请求、高成本和关键副作用 Trace 通过尾采样保留，普通成功流量低比例采样；
8. Trace 用于定位单次因果链，Metrics 用于告警和趋势，Logs 用于补充细节，Langfuse Score/Eval 用于回答质量。

### 0.1 当前仓库的证据边界

当前目录的[完整请求流程](./project_request_flow.md)和
[架构师题库](./interviewer_2_architect.md)已有目标 Span 树、
Trace/Span/Span Link 和高基数控制等概念说明，但没有以下证据：

- OTel SDK 初始化代码；
- FastAPI、LangGraph、模型、RAG、工具和数据库的真实 instrumentation；
- Collector 配置与部署记录；
- Langfuse 项目中的真实 Trace 截图或导出数据；
- 上下文传播、脱敏、采样、重复上报和 exporter 故障测试。

因此，本文中的组合架构应表述为：

- **目标设计**：本项目准备怎样组织 Trace；
- **教学骨架**：如何实现和验证；
- **生产演进**：多服务、多后端、集中治理时如何升级。

不能直接表述成“项目已经在生产使用 Langfuse + OpenTelemetry”。

### 0.2 面试时最值得先说的八句话

1. Trace ID 不是 Session ID，也不是幂等键：Trace 解释一次执行，Session 聚合多轮对话，幂等键约束业务副作用。
2. Langfuse 的 Observation 本质上由 OTel Span 映射而来，但额外带有 generation、token/cost、prompt 和 score 等 AI 语义。
3. 一个进程应由一个明确的组件拥有全局 TracerProvider，多个后端通过 Processor/Exporter 扇出。
4. 过滤决定“哪些 Span 去 Langfuse”，采样决定“哪些完整 Trace 被保留”，二者不能混为一谈。
5. SDK 端已经 head-sample 掉的 Trace，Collector 的 tail sampling 无法恢复。
6. 过滤掉根 Span 或父 Span 会制造 orphan observation，所以 Langfuse 分支必须保留完整的必要祖先链。
7. Baggage 会跨服务传播，绝不能放密钥、原始 PII、完整 Prompt 或工具结果。
8. GenAI 语义约定当前仍处于 Development，字段必须经适配层、版本锁定和契约测试后使用。

---

## 1. 先建立正确的概念地图

### 1.1 OpenTelemetry 与 Langfuse 分别解决什么

| 维度 | OpenTelemetry | Langfuse |
|---|---|---|
| 核心定位 | 厂商无关的遥测标准与采集管线 | 面向 LLM/Agent 的观测、调试、评测和成本分析 |
| 主要对象 | Trace、Span、Metric、Log、Baggage | Trace、Observation、Session、Prompt、Score、Dataset |
| 擅长回答 | 哪个服务、HTTP、队列、DB 或依赖变慢/报错 | 哪个模型、Prompt、检索、Agent 决策或工具调用导致质量、成本问题 |
| 跨服务传播 | W3C Trace Context、Baggage | 复用 OTel Context，并传播 Langfuse trace 级属性 |
| 采集与治理 | SDK、Processor、Exporter、Collector | Langfuse SDK/Processor 或原生 OTLP Trace 入口 |
| 多后端 | Collector/多个 Exporter | 作为其中一个 GenAI Trace 后端 |
| Metrics/Logs | 一等公民 | 重点是 LLM/Agent Trace 与 Eval，不应把其 Trace 入口当通用 Logs/Metrics 后端 |

最容易出现的误解是“用了 Langfuse 就不需要 OTel”或“有了通用 APM 就不需要 Langfuse”。更准确的分工是：

~~~text
OpenTelemetry：把整条系统调用链连起来
Langfuse：把调用链中的 AI 行为解释清楚并可评估
~~~

### 1.2 OTel 的五个运行时角色

| 角色 | 作用 | 常见误区 |
|---|---|---|
| API | 创建 Span、读取当前 Context 的接口 | 只引入 API 并不会自动导出数据 |
| SDK | Sampler、SpanProcessor、SpanExporter 等实现 | 多个库各自抢着设置 global provider |
| Instrumentation | 为 FastAPI、HTTP 客户端、数据库、模型框架自动或手工埋点 | 自动埋点不等于业务语义完整 |
| Exporter | 把遥测编码成 OTLP 并发给后端或 Collector | 把协议、端点和信号类型配混 |
| Collector | 接收、处理、采样、脱敏、批量、排队并扇出 | 以为 Collector 能恢复 SDK 已丢弃的 Span |

数据路径可以压缩成：

~~~mermaid
flowchart LR
    APP["业务代码与自动埋点"] --> SDK["OTel SDK / TracerProvider"]
    SDK --> PROC["SpanProcessor"]
    PROC --> EXP["OTLP Exporter"]
    EXP --> COL["OpenTelemetry Collector"]
    COL --> APM["通用 APM"]
    COL --> LF["Langfuse"]
~~~

### 1.3 Trace、Span、Event、Attribute、Link

- **Trace**：一次端到端执行的完整因果链，由同一个 Trace ID 关联。
- **Span**：链路中的一个有开始、结束和状态的操作，例如一次检索、模型调用或工具执行。
- **Event**：Span 内某个时间点发生的事件，例如“首次返回 token”“触发降级”；它没有独立持续时间。
- **Attribute**：可查询的结构化字段，例如模型名、检索 Top-K、错误类型。
- **Link**：把当前 Span 与另一个 Span Context 建立因果关联，但不把它设为唯一父节点；适合消息、批处理、扇出和汇聚。

### 1.4 四种 ID 不要混

| ID | 生命周期 | 用途 | 能否替代其他 ID |
|---|---|---|---|
| `trace_id` | 一次请求/一次 Agent run | 诊断一次执行 | 不能替代 session 或幂等键 |
| `span_id` | 单个操作 | 定位 Trace 内节点 | 不能跨重试充当业务 ID |
| `session_id` | 多轮对话 | 把多次 Trace 聚合成会话 | 不保证副作用唯一 |
| `idempotency_key` / `event_id` | 一次业务意图或事件 | 去重和约束副作用 | 不能用 Trace ID 临时生成 |

推荐口径：

> 用户每说一轮话形成一个 Trace；同一会话的多个 Trace 使用同一个 Session ID；预约创建使用独立、稳定的幂等键；Outbox 事件使用稳定 Event ID。

---

## 2. 本项目应该让 Trace 回答什么

一套可用的 Trace 方案至少要回答六类问题。

| 问题 | 需要的证据 |
|---|---|
| 请求为什么慢 | 根 Span、排队时间、RAG、模型、工具、DB 的持续时间和并行关系 |
| 回答为什么错 | Prompt 版本、模型版本、检索结果标识、降级路径、工具结果、Score |
| 为什么花费高 | 模型、输入/输出 token、重试次数、缓存命中、cost |
| 预约为什么没成功 | 槽位状态、确认门禁、工具状态、幂等结果、DB 冲突、Outbox 状态 |
| 为什么只有部分 Trace | 采样决策、过滤规则、上下文传播、Exporter 队列和失败指标 |
| 是否泄漏隐私 | 捕获策略、脱敏版本、字段白名单、保留期和访问审计 |

设计时遵守以下原则：

1. **业务语义优先**：除了 HTTP/SQL 自动 Span，还要有 Router、Agent、RAG、模型和工具等业务 Span。
2. **根节点稳定**：根 Span 覆盖完整用户轮次，名字低基数，不能包含用户 ID、请求 ID 或原始问题。
3. **链路可解释**：并行任务看时间重叠，重试看层级，异步任务看 Link，不靠日志猜测。
4. **内容最小化**：先记录版本、长度、哈希、分类和引用 ID，再按授权采集少量正文。
5. **失败不影响业务**：遥测导出失败不能让预约或回答失败；但遥测丢失本身必须可监控。
6. **指标与 Trace 分工**：高基数 ID 放 Trace/Log，不放 Metrics 标签。

---

## 3. Trace 粒度与 Span 树

### 3.1 推荐粒度：一轮对话一个 Trace

对聊天和 Agent 应用，默认选择：

~~~text
一个用户 turn / 一次 agent run = 一个 Trace
一个多轮对话 = 一个 Session
~~~

原因是：

- 单轮请求通常有明确的入口、Deadline、输出和错误结果；
- Trace 不会因对话持续数小时而长期不结束；
- 采样、成本、延迟和质量可以按轮次统计；
- Session 仍能把咨询、补槽、确认、结果查询等多轮关联起来。

例外：

- 一个后台批任务本身是独立执行单元，可单独成 Trace；
- 一个异步消费者执行可以单独成 Trace，再通过 Link 指向产生消息的 Span；
- 不要为了“看起来完整”把几天的会话强塞进一个永不结束的 Trace。

### 3.2 机房与电脑维修 Agent 的目标 Span 树

~~~mermaid
flowchart TD
    ROOT["support.chat_turn（根）"]
    ROOT --> LOAD["session.load / idempotency.check"]
    ROOT --> GRAPH["invoke_agent support_graph"]
    GRAPH --> ROUTE["langgraph.node route_intent"]
    GRAPH --> SUP1["langgraph.node supervisor_decide #1"]
    GRAPH --> CONSULT["invoke_agent consult_agent"]
    CONSULT --> RET["retrieval knowledge_base"]
    RET --> BM25["search bm25"]
    RET --> DENSE["search dense"]
    RET --> RERANK["rerank cross_encoder"]
    CONSULT --> GEN["chat answer_generation"]
    GRAPH --> SUP2["langgraph.node supervisor_decide #2"]
    GRAPH --> BOOK["invoke_agent booking_agent"]
    BOOK --> SLOT["chat extract_slots"]
    BOOK --> AVAIL["execute_tool engineer_availability"]
    BOOK --> CREATE["execute_tool booking.create"]
    CREATE --> TX["db transaction"]
    GRAPH --> FINAL["langgraph.node finalize"]
    ROOT --> SERIALIZE["response.serialize"]
    TX -. "Outbox context / Span Link" .-> CONSUMER["notification.process（异步 Trace）"]
~~~

图中的 Router、两次 Supervisor 决策、咨询节点和预约节点都是同一个 LangGraph Run 下的节点执行，不表示 Supervisor 通过 Tool Calling 嵌套调用专业 Agent。节点间的先后和因果关系由 State、条件边和`Command`记录；只有`execute_tool`节点才表示业务工具调用。

根 Trace 应覆盖：

- 入口校验与会话加载；
- Router 与 Supervisor 决策；
- RAG 的检索、融合、精排和降级；
- 模型流式生成到完整结束；
- 工具建议、服务端重新校验和真实副作用；
- 结果序列化；
- 对异步通知保留可追溯的因果关系。

根 Span 的持续时间不是所有子 Span 时长之和。BM25 和 Dense 并行时会重叠，真正影响用户时延的是关键路径。

### 3.3 OTel Span 到 Langfuse Observation 的映射

| 项目步骤 | OTel 语义 | Langfuse 观察类型 | 是否默认进入通用 APM | 是否建议进入 Langfuse |
|---|---|---|---:|---:|
| `support.chat_turn` | 应用根 Span | `span` | 是 | 是，必须保留 |
| LangGraph Run | `invoke_agent support_graph` | `agent`或`chain` | 是 | 是 |
| Router/Supervisor 节点 | INTERNAL 节点 Span，如`langgraph.node supervisor_decide` | `chain`或普通`span` | 是 | 是 |
| 咨询/预约 Agent 节点/子图 | `invoke_agent`或子图 Span | `agent` | 是 | 是 |
| 混合检索 | `retrieval` | `retriever` | 是 | 是 |
| 模型调用 | `chat` / embeddings | `generation` | 是 | 是 |
| 工具决策与执行 | `execute_tool` | `tool` | 是 | 是 |
| HTTP/DB/Redis | CLIENT/PRODUCER/CONSUMER 等基础设施 Span | 普通 `span` | 是 | 按需，通常不全量 |
| 内联 Guardrail | INTERNAL Span/Event | `guardrail` + 可选 Score | 是 | 是 |
| 离线评测 | 不一定是在线 Span | Score / Dataset Run | 可选 | 是 |

注意：

- “模型建议调用工具”和“工具真正产生副作用”应分开。前者记录模型输出契约，后者记录 Service 校验、幂等和事务结果。
- 不记录隐藏思维链。需要解释决策时记录可审计的分类、规则命中、工具选择、引用和简短决策摘要。
- Langfuse v4 以 Observation 为中心；同一 OTel Trace ID 下的 Observation 被聚合成 Trace。

### 3.4 Span 命名规则

推荐：

~~~text
support.chat_turn
route.intent
invoke_agent support_graph
langgraph.node supervisor_decide
invoke_agent consult_agent
retrieval knowledge_base
chat answer_generation
execute_tool booking.create
db transaction
~~~

不要：

~~~text
chat user-182736
retrieve “服务器无法访问启动设备怎么检查”
booking.create request-9f6c...
GET /users/847193/orders/29301
~~~

名字应低基数、跨版本可比较；实例 ID、用户问题、文档 ID 和错误详情放 Attribute。

如果明确声明某个推理 Span 遵循当前 OTel GenAI 语义约定，其名称应按
`{gen_ai.operation.name} {gen_ai.request.model}` 生成，例如
`chat model-alias`。上面的 `chat answer_generation` 是便于讲解的项目业务名；
落地时应选择一种契约并保持一致，可把业务阶段另存为受控 Attribute 或
Langfuse Observation 名。

---

## 4. 属性契约：记录什么、放在哪里

### 4.1 Resource：描述“谁在产生遥测”

| 字段 | 示例 | 说明 |
|---|---|---|
| `service.name` | `support-agent-api` | 必填级别的重要资源字段 |
| `service.version` | `2026.08.12+abc123` | 用于回归定位 |
| `deployment.environment.name` | `staging` | 区分环境 |
| `service.instance.id` | Pod/进程实例 | 高基数，仅 Trace/资源分析 |
| `cloud.region` | `cn-east-1` | 有多地域时记录 |

### 4.2 Trace 级业务属性

Langfuse v4 以 Observation 为中心。为了能在 Observation 层按 Trace 维度查询，
需要把相关 Trace 属性传播到每个目标 Observation，并尽早设置：

| 字段 | 建议值 | 安全要求 |
|---|---|---|
| `langfuse.trace.name` | `support-chat-turn` | 稳定、低基数 |
| `langfuse.user.id` | 伪名化用户 ID | 不用手机号、邮箱、身份证 |
| `langfuse.session.id` | 会话 ID | 可高基数，但不进入 Metrics 标签 |
| `langfuse.trace.tags` | `["support","booking"]` | 使用受控枚举 |
| `langfuse.release` | 应用发布版本 | 与部署版本对应 |
| `langfuse.version` | Agent 图/Prompt 方案版本 | 不要写自由文本 |
| `langfuse.trace.metadata.channel` | `web` | 需要顶层过滤时显式映射 |

只在根 Span 上设置但不传播，会导致子 Observation 按 user/session/version 查询时漏数。使用 Langfuse SDK 时应把 `propagate_attributes()` 放在尽可能靠近根的位置；跨服务需要时才选择 Baggage。

### 4.3 模型调用属性

当前 GenAI 语义约定建议围绕以下字段建立适配层：

| 字段 | 含义 | 时机 |
|---|---|---|
| `gen_ai.operation.name` | `chat`、`embeddings` 等逻辑操作 | 创建 Span 时 |
| `gen_ai.provider.name` | 模型提供方 | 创建 Span 时 |
| `gen_ai.request.model` | 请求的模型别名 | 创建 Span 时 |
| `gen_ai.response.model` | 服务端实际返回的模型/版本 | 得到响应后、Span 结束前 |
| `gen_ai.usage.input_tokens` | 输入 token | Span 结束前 |
| `gen_ai.usage.output_tokens` | 输出 token | Span 结束前 |
| `gen_ai.response.time_to_first_chunk` | 流式首次 chunk 延迟 | 首块到达后、Span 结束前 |
| `error.type` | 异常类型或受控错误类型 | 失败时 |

旧资料中常见 `gen_ai.system`，新 GenAI 约定已转向 `gen_ai.provider.name`。迁移期不要在全项目散落字段名，应通过一个 telemetry adapter 同时兼容旧/新字段，再由契约测试决定何时移除旧字段。

SDK 端 **head-sampling** 决策需要用到的字段必须在 Span 创建时提供；
Collector 的 tail sampling 则可以使用 Span 结束前补齐的 error、token、cost
等结果字段。无论哪种策略，Span 已经结束后再补 token、模型或错误信息都不会生效。

### 4.4 RAG、Agent 和工具属性

| 环节 | 推荐记录 | 默认不记录 |
|---|---|---|
| Router | `route.result`、`route.rule_version`、置信区间/回退原因 | 原始完整用户消息 |
| Agent | agent 名、版本、输入/输出 Schema 版本、结束原因 | 隐藏思维链 |
| Retrieval | index/version、top_k、返回数、来源 ID/哈希、降级原因 | 原始私有文档全文 |
| Rerank | 模型版本、候选数、耗时、是否回退 | 全量候选正文 |
| Tool | tool 名、参数 Schema 版本、校验结果、重试、结果状态 | 密钥、原始敏感参数 |
| Booking | 幂等命中、冲突类型、确认版本、事务结果 | 联系电话、详细住址 |

如果确实需要调试正文，应使用按环境、租户、角色和时间窗口控制的临时采集开关，并设置截断、脱敏、审计和自动过期。

### 4.5 Langfuse 特有映射

直接发送 OTel Span 时，可使用：

- `langfuse.observation.type`；
- `langfuse.observation.input` / `langfuse.observation.output`；
- `langfuse.observation.model.name`；
- `langfuse.observation.usage_details`；
- `langfuse.observation.cost_details`；
- `langfuse.observation.level`；
- `langfuse.observation.status_message`；
- `langfuse.observation.metadata.<key>`。

注意：

1. 手写 OTel 时 input/output 等复杂值通常需要 JSON 字符串；Langfuse SDK 会处理序列化。
2. 显式 `langfuse.*` 字段的映射优先于泛化 OTel/GenAI 字段。
3. 未映射的普通 Span Attribute 通常进入 `metadata.attributes`，Resource Attribute 进入 `metadata.resourceAttributes`；需要顶层筛选的维度要显式声明。
4. 当前主 OTel 映射表对 `langfuse.observation.type` 显式列出的基础值与
   Langfuse SDK 支持的丰富 Observation 类型并不完全等同。手写 direct OTLP
   使用 agent、tool、chain、retriever、evaluator、embedding、guardrail 等
   扩展类型时，应按锁定的 Server 版本做属性契约测试。

---

## 5. 四种组合拓扑与选型

### 5.1 方案 A：Langfuse SDK 直接上报

~~~text
Python/Node 服务 → Langfuse OTel-native SDK/Processor → Langfuse
~~~

适用：

- PoC、单体或少量 Python/Node 服务；
- 主要目标是 Prompt、generation、token/cost、session、score；
- 希望最快使用 Langfuse 的高级辅助方法。

优点：

- 接入最简单；
- Langfuse 属性和 Observation 类型映射方便；
- 当前 SDK 默认只导出 Langfuse、`gen_ai.*` 和已知 LLM instrumentation scope，噪声较低。

代价：

- 凭证和上报策略分散在应用；
- 集中脱敏、尾采样、队列治理和多后端扇出较弱；
- 与既有 APM 初始化顺序可能冲突。

### 5.2 方案 B：共享 TracerProvider，进程内多 Processor

~~~text
一个 global TracerProvider
├─ APM Processor/Exporter → 通用 APM
└─ Langfuse Processor → Langfuse
~~~

适用：

- 已有 OTel/APM；
- 希望同一 Context 下形成完整父子树；
- 服务数量不多，能够统一管理 instrumentation 入口。

关键要求：

- 明确一个 TracerProvider 所有者；
- Sampling 由这个 Provider 统一决定；
- Langfuse Processor 使用面向 LLM 的过滤器；
- 同一 Langfuse 后端不能再由通用 OTLP 经 Collector 转发同一批 Span。

Python 当前更稳妥的公开入口是把调用方的 Provider 传给 `Langfuse(tracer_provider=provider)`。部分旧 FAQ 的 `langfuse.opentelemetry.LangfuseSpanProcessor` import 与当前包结构存在版本漂移，不能不核验版本就复制。

### 5.3 方案 C：OTel Collector 集中治理与扇出

~~~text
多语言服务 → OTLP → Collector
                       ├─ 完整基础设施 Trace → APM
                       └─ 保留根节点的 LLM/Agent 子树 → Langfuse
~~~

适用：

- 多语言、多服务；
- 已有 Collector 或多个观测后端；
- 需要集中脱敏、重试、批处理、内存保护、尾采样和凭证治理；
- 需要让应用不直接持有 Langfuse secret key。

这是本文推荐的生产目标，但要注意两个细节：

1. Langfuse 的入口目前只支持 OTLP over HTTP/protobuf 或 HTTP/JSON，不支持 OTLP/gRPC。应用到 Collector 可以用 gRPC，Collector 到 Langfuse 必须用 `otlphttp`。
2. Langfuse 分支的过滤必须保留根 Span 和所需祖先。简单的逐 Span allowlist 可能留下孤儿节点；生产过滤策略应做整棵相关子树验证。

### 5.4 方案 D：独立 TracerProvider

~~~text
Provider A → APM
Provider B → Langfuse
~~~

只在严格隔离需求下选择，例如 Prompt/Completion 绝不能进入既有 APM，或基础设施 Span 绝不能进入 Langfuse。

代价：

- 两个 Provider 虽可能读取同一个当前 Context，但一方没有父 Span 时会出现 orphan；
- 采样和生命周期管理变成两套；
- 自动 instrumentation 归属更难推断；
- 排障复杂度明显上升。

### 5.5 决策矩阵

| 条件 | 首选方案 |
|---|---|
| 单体 PoC，只关注 LLM | A：Langfuse SDK 直传 |
| 已有 APM，要完整同一棵树 | B：共享 Provider，多 Processor |
| 多语言、多服务、多后端、集中治理 | C：Collector |
| 严格数据隔离高于树完整性 | D：独立 Provider |
| 要错误/慢请求尾采样 | C：Collector，并保证 trace affinity |
| 要使用 Langfuse SDK 高级辅助能力 | A/B；若改由 Collector 转发，按当前 SDK/Exporter API 专门验证 |

### 5.6 本项目的分阶段建议

1. **开发阶段**：用 Langfuse SDK 为根、Agent、RAG、generation、tool 建立最小树，100% 采样，验证属性和脱敏。
2. **联调阶段**：接入 FastAPI/HTTP/DB 自动 instrumentation，统一 Provider，验证 Context 不断。
3. **生产演进**：服务只保留“到 Collector”的一条 Trace 出口，Collector
   统一内存保护、脱敏、批处理、重试和扇出。此时要么改用纯 OTel
   instrumentation；要么按当前 Langfuse SDK 的版本化 API 为其配置指向
   Collector 的自定义 `span_exporter`，并关闭默认直传。仅修改通用
   `OTEL_EXPORTER_OTLP_ENDPOINT` 不会自动把 Langfuse SDK 的默认 Processor
   改道，不能据此假设没有双投。
4. **治理阶段**：APM 保留全链路，Langfuse 只保留完整的 AI 相关子树；尾采样保留错误、慢请求、高成本和关键副作用。

---

## 6. 接入骨架

以下示例用于解释结构，不能替代按锁定版本运行的集成测试。

### 6.1 Python：Langfuse v4 风格的业务埋点

~~~python
from langfuse import Langfuse, propagate_attributes


langfuse = Langfuse(sample_rate=1.0)


def handle_turn(command):
    # 不在 input 中放手机号、地址或完整原始 Prompt。
    root_input = {
        "intent_hint": command.intent_hint,
        "message_length_bucket": command.length_bucket,
    }

    with langfuse.start_as_current_observation(
        name="support.chat_turn",
        as_type="span",
        input=root_input,
    ) as root:
        with propagate_attributes(
            trace_name="support-chat-turn",
            user_id=command.pseudonymous_user_id,
            session_id=command.session_id,
            tags=["support-agent", command.intent_hint],
            version="agent-graph-v3",
            metadata={"channel": command.channel},
        ):
            with langfuse.start_as_current_observation(
                name="retrieval knowledge_base",
                as_type="retriever",
            ) as retrieval:
                docs = retrieve(command.safe_query)
                retrieval.update(
                    output={"source_ids": [doc.safe_id for doc in docs]},
                    metadata={"index_version": "kb-2026-08"},
                )

            with langfuse.start_as_current_observation(
                name="chat answer_generation",
                as_type="generation",
                model="model-alias",
            ) as generation:
                result = call_model(docs)
                generation.update(
                    output={"answer_length": len(result.text)},
                    usage_details={
                        "input": result.input_tokens,
                        "output": result.output_tokens,
                    },
                )

        root.update(output={"status": "completed"})
        return result


~~~

实现注意：

- `propagate_attributes()` 要尽早进入；它不会回填已经结束或先前创建的 Span。
- CLI、测试和 serverless 等短任务在入口的 `finally` 或平台结束钩子中调用
  `langfuse.flush()`；不要把它放在模块顶层，否则 import 时就会提前执行。
- 长驻 FastAPI 服务不要每个请求都 `flush()`，应在应用 shutdown/lifespan
  阶段 flush/shutdown。
- generation 的 output 示例只记录长度，生产是否记录正文由数据分类策略决定。
- 自定义 `should_export_span` 会覆盖默认过滤逻辑；若只是扩展，应与 SDK 提供的默认判定辅助函数组合。

### 6.2 Python：由应用统一创建共享 Provider 时

~~~python
from langfuse import Langfuse
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor


provider = TracerProvider(sampler=build_project_sampler())
provider.add_span_processor(BatchSpanProcessor(build_apm_exporter()))
trace.set_tracer_provider(provider)

# 当前公开入口：让 Langfuse 把自己的 processor 注册到调用方 Provider。
langfuse = Langfuse(tracer_provider=provider)
~~~

关键点不是代码行数，而是：

- 这段代码是 instrumentation 入口的初始化骨架，不是在“已经存在全局
  Provider”的进程里再次调用 `set_tracer_provider`；
- 如果 APM 已经创建 Provider，应获取并复用该实例，在确认其允许注册
  Processor 后传给 Langfuse；否则调整启动顺序，由应用统一创建；
- Provider 只初始化一次；
- APM 与 Langfuse 各有一条清晰出口；
- Sampler 在 Provider 创建时确定；
- 进程退出时统一 shutdown；
- 依赖版本和 Processor 注册结果有启动自检。

### 6.3 直接手写 OTel GenAI Span

~~~python
from opentelemetry import trace
from opentelemetry.trace import SpanKind, Status, StatusCode


tracer = trace.get_tracer("support-agent.telemetry")


def call_generation(client, request):
    with tracer.start_as_current_span(
        f"chat {request.model}",
        kind=SpanKind.CLIENT,
        # 本示例显式处理异常，关闭 context manager 的自动异常事件和状态，
        # 避免同一个异常被记录两次。
        record_exception=False,
        set_status_on_exception=False,
        attributes={
            # 参与采样决策的字段在创建 Span 时提供。
            "gen_ai.operation.name": "chat",
            "gen_ai.provider.name": "provider-name",
            "gen_ai.request.model": request.model,
            "langfuse.observation.type": "generation",
            "app.feature": "support-answer",
        },
    ) as span:
        try:
            response = client.generate(request)
            span.set_attribute("gen_ai.response.model", response.model)
            span.set_attribute(
                "gen_ai.usage.input_tokens",
                response.input_tokens,
            )
            span.set_attribute(
                "gen_ai.usage.output_tokens",
                response.output_tokens,
            )
            return response
        except Exception as exc:
            span.record_exception(exc)
            span.set_attribute("error.type", type(exc).__qualname__)
            span.set_status(Status(StatusCode.ERROR))
            raise
~~~

从 OTel API 语义看，`record_exception()` 负责记录异常事件，本身不应被当作
“所有语言都会自动设置 Error Status”的保证。Python 的
`start_as_current_span` context manager 默认会自动记录逃出作用域的异常并
设 Error；示例显式关闭这两个默认开关后再手工处理，避免重复异常 Event，也
把项目的错误契约写清楚。成功路径通常保持默认 Unset 即可，不必给所有 Span
强行设置 OK。

### 6.4 Collector 到 Langfuse 的最小连通配置

下面是“先验证连通与扇出”的基础模板。它会把同一 Trace 发给两个后端，尚未加入 Langfuse 专属过滤和尾采样，不能原样当最终成本方案。

~~~yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 512
    spike_limit_mib: 128
  batch:
    timeout: 5s
    send_batch_size: 1024

exporters:
  otlphttp/langfuse:
    endpoint: "https://cloud.langfuse.com/api/public/otel"
    headers:
      Authorization: "Basic ${env:LANGFUSE_AUTH_STRING}"
      x-langfuse-ingestion-version: "4"

  otlphttp/apm:
    endpoint: "${env:APM_OTLP_HTTP_ENDPOINT}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/langfuse, otlphttp/apm]
~~~

其中：

- `LANGFUSE_AUTH_STRING` 是 `base64("<public-key>:<secret-key>")` 的结果，不把原始 key 写入仓库；
- Langfuse Cloud 区域地址必须与项目一致；
- Collector 的重试和发送队列能力由 exporter/helper 版本与配置决定，要做断网压测；
- 真正生产配置应让 Langfuse 使用独立的 filter/redaction/tail-sampling 分支，避免其策略影响 APM；
- Collector Processor 和 OTTL 配置语法也会随版本变化，必须锁定 Collector 发行版并跑配置测试。

Langfuse 当前原生 OTLP Trace 入口支持 HTTP/protobuf 和 HTTP/JSON，不支持 gRPC，也不接收通用 Logs。直接或自定义 OTel Exporter 还要显式带 `x-langfuse-ingestion-version: 4`；当前官方 Langfuse SDK 会自行处理，使用 SDK 时不需要重复手配。没有该 Header 的直连数据可能最多延迟约十分钟。

### 6.5 Langfuse Cloud 地址与认证

| 区域 | OTLP base endpoint |
|---|---|
| EU | `https://cloud.langfuse.com/api/public/otel` |
| US | `https://us.cloud.langfuse.com/api/public/otel` |
| Japan | `https://jp.cloud.langfuse.com/api/public/otel` |
| HIPAA | `https://hipaa.cloud.langfuse.com/api/public/otel` |

Trace 专用地址是在 base 后追加 `/v1/traces`。认证是：

~~~text
Authorization: Basic base64("<public-key>:<secret-key>")
~~~

凭证只放服务端 Secret Manager 或 Collector，不放前端、Baggage、Span Attribute、日志和仓库。必须使用 TLS；自签名部署应配置可信 CA，不能通过关闭证书校验解决问题。

### 6.6 当前版本快照

截至 2026-08-12，官方发布页可核对到：

| 组件 | 快照版本 | 兼容性重点 |
|---|---:|---|
| Langfuse Server | v4.9.0 | v4 为当前 GA 主线 |
| Python SDK | v4.14.4 | Python 3.9+；自 v3 起基于 OTel |
| JS/TS SDK | v5.10.0 | Node.js 20+；自 v4 起基于 OTel |

实时 ingestion 的兼容门槛是 Python SDK >= 4.7.0、JS SDK >= 5.4.0；当前官方 SDK 自动处理实时通道要求。版本号只是本文快照，面试中更重要的是说明“锁定并验证版本”，不要死背成永久事实。

---

## 7. Context、Baggage 与异步链路

### 7.1 同步跨服务：W3C Trace Context

HTTP/RPC 调用通过 `traceparent` 和可选 `tracestate` 传播：

~~~text
traceparent: 00-<trace-id>-<parent-span-id>-<flags>
~~~

接收端提取 Context 后，新 Span 与上游成为父子关系。不要手工把 Trace ID 拼进业务请求并试图代替标准 Propagator。

### 7.2 Baggage 传的是业务维度，不是父子关系

| 项目 | Trace Context | Baggage |
|---|---|---|
| 作用 | 传播 Trace/父 Span/采样标记 | 传播任意键值业务维度 |
| 是否自动成为 Span Attribute | 提供 Context 关系 | 通常不会，需 Processor/Instrumentation 显式复制 |
| 风险 | 伪造上游 Context、信任边界 | 泄漏、膨胀、被下游转发、缺少内建完整性保护 |
| 建议内容 | 标准字段 | 低敏、必要、短小、受控枚举 |

适合放入 Baggage 的例子：

- 低敏环境标识；
- 受控的 feature/experiment 标识；
- 已伪名化且确实需要跨服务筛选的维度。

禁止放：

- API key、Authorization；
- 手机号、邮箱、地址；
- Prompt/Completion；
- 工具参数、检索正文；
- 高体积 JSON。

进入第三方服务前应清理内部 Baggage。来自公网的 Baggage 只能视为不可信提示，不能用于权限、租户隔离或计费事实。

### 7.3 Outbox 和消息消费为什么常用 Link

生产者 Span 结束后，消息可能几秒甚至几小时后由另一个进程消费。消费者还可能：

- 重试；
- 批量消费多条消息；
- 一条消息扇出多个消费者；
- 多条消息汇聚成一个批任务。

Span 只能有一个父节点，而 Link 可以表达多个因果来源。因此建议：

~~~mermaid
sequenceDiagram
    participant API as Booking API
    participant DB as DB / Outbox
    participant W as Outbox Worker
    participant C as Notification Consumer

    API->>DB: 事务写 Booking + Outbox
    API->>API: producer span 写入 message creation context
    W->>C: 发布 event_id + trace context
    C->>C: 建立 consumer/process span
    Note right of C: process span 添加 Span Link<br/>指向 message creation context
~~~

选型口径：

- 单条消息、立即消费、希望明确串联时可以延续父子 Context；
- 批处理、扇出、长延迟或重试链更适合 Link；
- 无论怎样，业务去重使用 `event_id` / Inbox，不使用 Trace ID；
- 消息 Header 中只放传播 Context 和必要业务标识，不复制敏感正文。

### 7.4 异步、线程和进程中的 Context

- `asyncio` 任务通常可借助语言运行时 Context 传播，但要测试自定义 task 创建和回调。
- 线程池需要对应的线程 instrumentation 或显式 `context.attach/detach`。
- 多进程/worker 不要假设 Provider、队列和后台线程可以安全继承；按当前 SDK 指引在 worker 生命周期初始化。
- JS/TS 的 instrumentation 入口必须早于 OpenAI、LangChain、HTTP 等被观测库导入。
- CLI、serverless 和测试进程必须在退出前 force flush；长驻服务在优雅停机阶段 shutdown。

---

## 8. 过滤、采样与成本控制

### 8.1 过滤和采样解决不同问题

| 机制 | 回答的问题 | 典型粒度 | 错误做法 |
|---|---|---|---|
| Filter | 哪些类型的 Span 应进入某个后端 | Span/属性 | 留下子节点却删除根和父节点 |
| Head sampling | Trace 刚开始时要不要记录/导出 | 完整 Trace 的早期决策 | 期望事后保留错误或慢 Trace |
| Tail sampling | 看过一段或全部 Trace 后要不要保留 | 完整 Trace 的后置决策 | 多实例间不按 Trace ID 粘性路由 |
| Langfuse Score/Eval sampling | 哪些 Trace 需要人工/模型评测 | Trace/数据集 | 把它与遥测采样当成同一开关 |

过滤的目标是减少无关 Span，例如不把每个 Redis、DNS、健康检查都计为 Langfuse Observation。采样的目标是控制完整 Trace 的量。先逐 Span 过滤再独立采样，很容易形成不完整树，必须用 Trace 结构测试验证。

### 8.2 Head sampling 与 tail sampling

**Head sampling**

- 在根 Span 创建时决定；
- 开销低、无状态、容易水平扩展；
- 只能依据创建时已有属性；
- 不知道请求最终是否报错、很慢或 token 超预算。

**Tail sampling**

- Collector 收集一段时间或完整 Trace 后决定；
- 能保留错误、超时、高延迟、高成本、关键工具调用；
- 需要缓存 Span，消耗内存并引入决策等待；
- 多 Collector 实例必须让同一个 Trace ID 的 Span 到达同一尾采样实例。

最重要的限制：

> 上游 SDK 已经 head-sample 掉的 Trace 不会发到 Collector，尾采样无法把它“复活”。

因此，如果目标是“错误和慢请求全部保留”，必须保证到 tail sampler 的整条
上游路径都 RecordAndSample，例如受控入口使用 AlwaysOn，并核验入站
`traceparent` 的 sampled flag。简单写成 ParentBased(root=AlwaysOn) 仍会服从
“未采样远程父”的决定，不能保证所有 Span 到达 Collector。是否承受得住这一
开销，要用流量、Span 数、平均大小和 Collector 内存预算计算。

### 8.3 本项目的采样策略

建议从以下策略开始，再用实际流量校准：

| Trace 类型 | 建议 |
|---|---|
| 开发/集成环境 | 短期 100%，但仍执行脱敏 |
| 生产普通成功请求 | 低比例随机保留 |
| Error、超时、取消 | 100% 保留 |
| 超过动态 P95/P99 阈值 | 100% 保留 |
| token/cost 超预算 | 100% 保留 |
| 预约创建、取消等关键副作用 | 按合规和成本提高保留率 |
| Guardrail 拒绝、权限拒绝、Prompt 注入命中 | 100% 或高比例保留，但正文仍脱敏 |
| 特定 release/canary | 在受控时间窗提高比例 |

不要把所有阈值硬编码成永久常量。延迟和成本门槛应由 SLO、模型、场景和历史分布决定。

### 8.4 两个后端如何避免采样不一致

如果 APM 与 Langfuse 独立随机采样，同一 Trace 可能只在一个后端出现，跨系统排障困难。更稳妥的做法：

1. 在扇出前统一决定 Trace 是否保留；
2. 对 Langfuse 再做“内容/类型过滤”，但保留必要父链；
3. 两端都使用同一个 Trace ID，可从 APM 深链到 Langfuse；
4. 如果业务必须使用不同采样率，明确记录 sampling policy/version，并接受数据集口径不同。

不要同时设置 Langfuse `sample_rate` 和外部 Provider Sampler 并误以为它们会
形成两层相乘采样。当前 Python SDK 只在自己创建 Provider 时用
`sample_rate` 构造 Sampler；调用方传入或全局已有 Provider 时，应由该
Provider 的 Sampler 决定，Langfuse `sample_rate` 可能不生效。工程上仍应
指定唯一采样所有者，并用固定 Trace 验证。

### 8.5 避免重复上报

最常见的重复路径：

~~~text
同一个 Span
├─ LangfuseSpanProcessor → Langfuse
└─ Generic OTLP → Collector → Langfuse
~~~

多个 Processor 本身没有问题；两个目的地都指向同一个 Langfuse 项目才是问题。应在启动配置中输出不含密钥的 topology summary，并用固定 Trace 做唯一性测试：

- UI 中同一个 logical operation 是否只出现一次；
- Trace ID、Span ID 是否重复；
- token/cost 是否翻倍；
- 两条 exporter 队列是否都命中 Langfuse 地址。

### 8.6 基数与账单

每个到达 Langfuse 的 Span 都可能成为可计费 Observation。打开 export-all 后，HTTP、SQL、Redis 和队列 Span 数可能远大于 generation 数。

控制手段：

- Span 名低基数；
- 先分析 `metadata.scope.name` 和各 scope 的 Span 数；
- 对健康检查、静态资源、轮询、内部 DB 明细做受控过滤；
- 保留根节点和 Agent/LLM 子树；
- 设每 Trace 最大 Span 数、事件数、属性长度和正文长度；
- 监控 exporter bytes、accepted/refused/dropped spans 和 Langfuse 用量。

---

## 9. 隐私、安全与数据治理

### 9.1 默认把 LLM 数据视为敏感数据

Prompt、Completion、工具参数、工具结果、检索文档、长期记忆都可能包含：

- 姓名、手机号、邮箱、地址、工单内容；
- 账号、API key、Cookie、Authorization；
- 商业机密和内部知识库正文；
- 模型生成但不应持久化的敏感推断；
- 被 Prompt Injection 诱导输出的系统信息。

“只发到内部观测系统”不能替代数据最小化。

### 9.2 数据分级与默认采集

| 数据 | 默认策略 | 调试增强 |
|---|---|---|
| Trace/Span ID、版本、耗时、状态 | 采集 | 无需增强 |
| 模型名、token、cost、finish reason | 采集 | 无需增强 |
| 用户/会话 | 伪名化后采集 | 仅受控人员可反查 |
| Prompt/Completion | 默认关闭或脱敏摘要 | 临时、抽样、截断、审批 |
| 工具参数/结果 | 白名单字段 | 按工具风险单独开关 |
| 检索正文 | 仅 source ID、版本、分数 | 必要时短片段脱敏 |
| 系统 Prompt/工具定义 | 版本/哈希 | 不默认上传全文 |
| 隐藏思维链 | 不采集 | 不提供调试开关 |
| 凭证 | 永不采集 | 无例外 |

### 9.3 三层防线

1. **应用边界前**：字段白名单、伪名化、SDK export-stage masking；这是“敏感数据不能离开应用”场景的主防线。
2. **Collector**：redaction/transform/filter，作为统一出口的第二层防线。
3. **后端**：RBAC、项目隔离、保留期、审计、加密和服务端 masking，作为兜底。

当前 Python v4 若还要覆盖第三方 instrumentation 产生的最终 OTel Span，优先
评估 `mask_otel_spans`；旧式 `mask` 主要处理经 Langfuse SDK API 设置的数据。
`mask_otel_spans` 抛异常或返回无效结果时，Langfuse Processor 可能丢弃整个
导出批次；它只改变发往 Langfuse 的副本，不会自动清洗发给其他 Exporter 的
副本。因此各出口都要有自己的数据治理策略，并为 masking 失败设置 canary
和告警。

对于自托管 Langfuse，服务端 masking 是 Enterprise 功能，并发生在异步
Worker 处理阶段；原始 ingestion 事件会先写入 event blob/object storage。
因此它不能证明“原始数据从未离开应用边界”。官方当前默认是约 500 ms
超时、最多重试一次且 fail-open；回调失败时数据会未脱敏继续处理。严格场景
必须在客户端或 Collector 先脱敏，并根据可用性与合规要求评估
fail-open/fail-closed 策略。

### 9.4 安全检查清单

- Langfuse secret key 只放 Secret Manager，定期轮换；
- Collector 到后端启用 TLS，验证服务端证书；
- 不记录 exporter Header 和完整环境变量；
- 公网入口限制请求体、Batch 大小、速率和认证失败告警；
- 跨租户字段由可信服务端上下文写入，不信任模型或客户端自报；
- 对 Prompt/Completion 设置最大长度、保留期、导出权限和删除流程；
- 生产、测试、开发使用独立项目和密钥；
- 评估 trace_id 暴露是否会成为内部查询越权入口；
- 对遥测供应链依赖做版本、许可证和漏洞管理；
- 任何调试“临时全量开关”都要自动过期并留下审计记录。

---

## 10. Trace、Metrics、Logs 与 Eval 如何配合

### 10.1 四类信号的分工

| 信号 | 最适合回答 | 例子 |
|---|---|---|
| Trace | 单次请求为什么发生 | 某次咨询因 Dense 超时回退 BM25，随后模型重试 |
| Metric | 系统是否整体恶化 | P95、错误率、token/s、队列年龄、cost/turn |
| Log | 离散细节和审计 | 配置加载失败、结构化校验错误摘要 |
| Score/Eval | 回答质量是否达标 | Faithfulness、工具选择正确性、用户反馈 |

日志应带 `trace_id` 和 `span_id` 以便跳转，但不要重复存一份完整 Prompt。指标不能使用 `user_id`、`session_id`、`request_id`、`document_id` 等高基数标签。

### 10.2 本项目的指标面板

**入口与体验**

- 请求数、成功率、Error/timeout/cancel rate；
- end-to-end P50/P95/P99；
- TTFT/首次 chunk、完整生成时间；
- 按 intent、release、模型和环境切分。

**RAG**

- BM25/Dense/Rerank 时延和失败率；
- 降级比例；
- 返回空结果率；
- 离线 Recall@K、MRR、NDCG、Faithfulness。

**模型**

- 输入/输出 token；
- token/s、限流、重试和缓存命中；
- cost/turn、cost/successful turn；
- 模型/Prompt 版本对应的质量 Score。

**Agent/工具**

- Router 分布、Supervisor 步数；
- 缺槽澄清率、无效工具参数率；
- 工具成功率、幂等命中、预约冲突；
- 每 Trace Agent/工具循环次数。

**遥测管线**

- accepted、sent、failed、dropped spans；
- exporter queue size、retry、send latency；
- Collector memory、CPU、refused spans；
- orphan rate、缺根率、重复率；
- UI ingestion 延迟和账单用量。

### 10.3 SLO、告警与 Trace exemplar

告警应基于指标，而不是对每个 Error Span 发通知。告警触发后再用 exemplar 或 Trace ID 定位样本：

~~~text
P95 或错误率越界
        ↓
按 release / model / intent 缩小范围
        ↓
打开对应 Trace 样本
        ↓
在 Langfuse 看 Prompt、generation、token、retrieval、score
        ↓
在 APM 看 HTTP、队列、DB、资源与下游依赖
~~~

质量告警不要只看用户点赞。应组合：

- 冻结集回归；
- 线上抽样评测；
- 规则型结构化输出检查；
- 用户反馈；
- Guardrail 命中；
- 人工复核。

### 10.4 Eval 与 Trace 的连接

- 在线 Trace 记录 Prompt/model/retrieval/tool 版本；
- Langfuse Score 附着在 Trace 或 Observation 上；
- 失败样本进入 Dataset；
- 新版本离线重放并与基线比较；
- 通过质量门禁后灰度；
- 灰度 Trace 按 release/tag 分组对照。

离线 Eval 和在线遥测采样可以使用不同样本策略，但要保存入选原因和数据版本，避免拿偏置样本冒充总体质量。

---

## 11. 落地步骤与验收标准

### 11.1 第 0 阶段：先冻结契约

输出一份短 RFC，至少确定：

- Trace、Session、幂等键和 Event ID 的边界；
- 根 Span 名称和每类业务 Span；
- Resource、Trace 级、Observation 级属性；
- 内容采集白名单、脱敏和保留期；
- Provider、Sampler 和 exporter 的唯一所有者；
- Langfuse 唯一出口；
- 版本矩阵和回滚方法。

同时建立 telemetry adapter，业务代码调用项目自己的薄接口，不直接散落版本敏感的 `gen_ai.*` 或 `langfuse.*` 字符串。

### 11.2 第 1 阶段：单服务最小闭环

只埋：

1. `support.chat_turn` 根 Span；
2. 一个 retrieval；
3. 一个 generation；
4. 一个 tool；
5. 一个错误分支。

100% 采样跑固定夹具，检查：

- 一棵树只有一个根；
- user/session/version 传播到所有目标 Observation；
- token、model、status 在 Span end 前出现；
- 敏感夹具不会出现在导出数据；
- CLI/服务 shutdown 后无缓冲丢失。

### 11.3 第 2 阶段：分布式和异步传播

- FastAPI → RAG/模型服务保持同一 Trace ID；
- 线程池和异步任务保持正确当前 Context；
- Outbox producer 注入 Context；
- Consumer 通过父子或 Link 表达因果；
- 重试不会生成错误的并列根；
- 幂等键与 Trace ID 保持独立。

### 11.4 第 3 阶段：Collector 和多后端

- 应用只指向测试 Collector；
- Collector 到 APM 和 Langfuse 各做连通测试；
- Langfuse endpoint、HTTP 协议、Basic Auth 和 ingestion header 正确；
- 断网、401、429、5xx、超时下业务请求不被 exporter 阻塞；
- 队列、重试、丢弃和恢复行为有指标；
- 用固定 Trace 证明没有双投。

### 11.5 第 4 阶段：过滤、采样和成本

- 统计各 instrumentation scope 的 Span 数与体积；
- 为 Langfuse 设计保根的 allowlist；
- 加入尾采样并做 Trace ID 粘性路由；
- 验证错误、慢请求、高成本和关键副作用保留；
- 比较采样前后 Trace 完整率、成本和定位能力；
- 对配置做 canary，再逐步扩大。

### 11.6 必须自动化的测试矩阵

| 测试 | 断言 |
|---|---|
| 基本树 | 一个根，父子关系和名称符合快照 |
| 并行 RAG | BM25/Dense 时间重叠，父节点正确 |
| 流式生成 | Span 到完整响应/取消才结束，TTFT 单独记录 |
| 自动重试 | 一个逻辑 GenAI Span，网络尝试为子 Span 或事件 |
| 错误语义 | exception、`error.type`、Status 一致 |
| 跨 HTTP | 下游 Trace ID 相同，parent 正确 |
| Outbox | 消费 Span 有正确 Link/context，event_id 独立 |
| 属性传播 | user/session/release 出现在目标 Observation |
| 内容安全 | canary PII/secret 不出现在 OTLP payload 和后端 |
| 根过滤 | Langfuse 无 orphan，根和必要祖先保留 |
| 采样 | error/slow/cost 规则命中，普通流量比例在容差内 |
| 双投 | 同一 Span ID 在 Langfuse 只出现一次 |
| Provider 冲突 | 初始化顺序变化时启动自检能失败或告警 |
| Exporter 故障 | 后端故障不影响业务，丢弃/重试可见 |
| 生命周期 | 短任务 flush，长驻服务优雅 shutdown |
| 负载 | Collector 内存、队列、CPU 和背压符合预算 |

### 11.7 什么证据齐全后才能说“已落地”

至少需要：

- SDK/Provider/Exporter 初始化代码；
- 版本锁文件；
- Collector 配置与部署清单；
- 属性契约与数据分级文档；
- 自动化树结构、传播、脱敏、采样和重复测试；
- 一条正常、一条错误、一条慢请求、一条异步消息的真实 Trace；
- exporter/Collector 自监控面板；
- 用量与保留期报告；
- 运行手册、告警和回滚记录。

只有架构图或 UI 截图不能证明端到端可靠。

---

## 12. 高频故障排查手册

### 12.1 现象到原因

| 现象 | 优先检查 | 常见根因 | 修复方向 |
|---|---|---|---|
| APM 有 Trace，Langfuse 没有 | Langfuse Processor/filter、endpoint、401/429、exporter 指标 | Span 不符合默认过滤、凭证/区域错误、直连漏 header | 用固定 GenAI Span 连通；看 Processor 决策与响应码 |
| Langfuse 有子节点但无根 | 父 Span 是否被 filter/sample | 逐 Span allowlist 删除父链 | 保留根和必要祖先；做树结构测试 |
| Trace 被拆成多棵 | `traceparent`、当前 Context、异步边界 | 下游未提取、线程/任务 Context 丢失、独立 Provider | 校验传播 Header 与 worker 初始化 |
| 同一 generation 出现两次 | 所有 Processor/Exporter 目标 | SDK 直传 + Collector 转发双投 | 同一 Langfuse 项目只留一条出口 |
| token/cost 为空 | 字段名、设置时机、Observation 类型、模型定价 | Span end 后补属性、字段版本不兼容、未标 generation | 在 end 前写 usage/model，做属性契约测试 |
| user/session 只能在 Trace 顶层看到 | 子 Span 的传播属性 | 只更新根、`propagate_attributes` 调用太晚 | 在创建 children 前传播 |
| Serverless 数据偶发丢失 | flush/shutdown、export mode、冻结时机 | Batch 队列未发完进程就结束 | 退出前 forceFlush；短任务评估 immediate |
| Langfuse 数据晚约数分钟 | ingestion header、SDK 版本 | 直接 OTLP 未带 v4 header | 直连加 header；官方 SDK 升级到兼容版本 |
| 全量接入后成本暴涨 | Span 数按 scope/type 分布 | export-all 把 HTTP/DB/queue 全送 Langfuse | 收紧 filter、限制属性和每 Trace Span 数 |
| 错误 Trace 仍丢失 | SDK Sampler、Collector tail policy | 上游先做低比例 head sampling | 上游保留后再 tail sample |
| 尾采样 Trace 缺 Span | Collector 负载均衡 | 同一 Trace 被分散到多个 stateful 实例 | 在尾采样前按 Trace ID 粘性路由 |
| 采样率与配置不符 | global Provider 与 SDK 启动顺序 | Provider 已被 APM 提前创建，后设 sampler 无效 | 单一入口初始化并启动自检 |
| 直接 OTLP 返回 400 | ID、数组结构、编码、Content-Type | 手写 JSON 不符合 OTLP、非法 Trace/Span ID | 使用官方 exporter，抓取最小失败批次 |
| Langfuse gRPC 不通 | endpoint/protocol | 把 4317/gRPC 思路直接用于 Langfuse | Collector 到 Langfuse 改 `otlphttp` |

### 12.2 系统化排障顺序

不要从 UI 盲猜，按数据路径逐段证明：

1. 应用内 Span 是否 Recording；
2. Trace/Span ID、父节点、属性是否在 end 前完整；
3. Processor 是否接受或过滤；
4. Batch 队列是否 flush；
5. Exporter 的 endpoint、协议、Header、TLS 是否正确；
6. Collector receiver 是否 accepted；
7. Processor 是否 dropped/refused；
8. Exporter 是否成功，重试队列是否积压；
9. Langfuse ingestion 是否返回 2xx；
10. 后端数据模型是否按预期映射和可查询。

准备一个不含 PII 的 canary Trace，固定：

- Trace 名；
- Span 树；
- 模型和 token；
- user/session 伪名；
- 一条 error；
- 一个 Link；
- 一个 Score。

每次 SDK、Collector、Langfuse Server 或语义约定升级后都运行。

### 12.3 遥测可靠性本身也要可观测

遥测系统应监控自己：

- Processor queue 使用率；
- export success/failure/retry/drop；
- Collector accepted/refused/sent；
- tail sampling decision wait、dropped trace；
- 后端 ingestion latency；
- canary Trace 到达率和端到端延迟；
- 配置版本与 reload 结果。

遥测是 best effort，不应阻塞主业务；但“best effort”不等于静默丢失。

---

## 13. 32 道面试题与参考回答

下面每题都按“先说结论—再展开—最后指出陷阱”的顺序组织。

### 基础题

#### 1. Trace、Span、Event、Attribute 有什么区别？

**结论**：Trace 是一次端到端执行；Span 是其中有持续时间的操作；Event 是 Span 内的瞬时事件；Attribute 是可查询结构化字段。

**展开**：一次机房电脑维修咨询或预约请求是 Trace，RAG 和模型调用是 Span，“首次返回 chunk”是 Event，模型名、token 和错误类型是 Attribute。

**陷阱**：不要把 Session 当 Trace。Session 可以包含多次 Trace。

#### 2. Trace ID、Span ID、Session ID、幂等键是什么关系？

**结论**：它们解决诊断层级、会话聚合和业务去重三个不同问题，不能互相替代。

**展开**：一次对话轮次有 Trace ID；其中每个操作有 Span ID；多轮共享 Session ID；预约写入使用稳定幂等键。

**陷阱**：用随机 Trace ID 做幂等键会让客户端重试产生新 key，失去去重意义。

#### 3. OTel API、SDK、Processor、Exporter、Collector 各负责什么？

**结论**：API 提供埋点接口；SDK 实现采样与处理；Processor 接收结束的 Span；Exporter 发送数据；Collector 负责集中接收、处理和扇出。

**展开**：Langfuse 的 Processor 可以挂到 OTel Provider；通用 OTLP Exporter 可以先发 Collector，再由 Collector 送 APM/Langfuse。

**陷阱**：安装 OTel API 不等于已经有 SDK 和 exporter。

#### 4. Context、Baggage、Span Attribute 有什么区别？

**结论**：Context 维持当前 Span 和跨服务父子关系；Baggage 携带跨服务键值；Attribute 属于某个 Span。

**展开**：`traceparent` 传播 Trace Context；user/session 等必要维度可谨慎通过 Baggage 传播，再复制成子 Span Attribute。

**陷阱**：Baggage 不会天然变成 Attribute，也不能放凭证和 PII。

#### 5. Parent/Child 与 Span Link 应如何选择？

**结论**：同步、单一因果通常用父子；消息、批处理、扇出、汇聚和长延迟任务更适合 Link。

**展开**：预约 API 到 DB 是父子；Outbox 消费者可新建处理 Span 并 Link 到 producer context。

**陷阱**：Span 只有一个父节点，批量消费多个消息不能伪装成多个 parent。

#### 6. 如何正确记录错误？

**结论**：记录异常事件、稳定的 `error.type`，并按语义设置 Span Status；业务拒绝不一定都是系统 Error。

**展开**：模型 HTTP 失败应 record_exception 并设 Error；用户没授权而被正常拒绝，可以记录受控 outcome 而不一定把系统标红。

**陷阱**：只 record_exception 不一定自动设置 Error Status；也不要把每个 4xx 都无脑标系统故障。

### 中级题

#### 7. 流式 LLM 调用的 Span 何时结束？

**结论**：在完整响应结束、取消或失败后结束，而不是收到第一个 token 时结束。

**展开**：当前 GenAI 语义约定使用
`gen_ai.response.time_to_first_chunk` Span Attribute（单位秒）记录首次 chunk
延迟；也可另加项目 Event，但它不是标准 TTFT Event。Span 持续时间表示完整
生成。取消时记录取消原因和已有 usage。

**陷阱**：不要每个 token 建一个 Span，会造成极高体积和错误时延语义。

#### 8. LLM SDK 自动重试时应该有几个 Span？

**结论**：一个用户可理解的逻辑推理操作应保留一个 GenAI Span，底层 HTTP 尝试可以是子 Span或 Event。

**展开**：这样 generation latency/cost 是完整逻辑调用，网络尝试仍可定位。

**陷阱**：把每次尝试都当并列 generation 会让调用次数和成本口径混乱。

#### 9. `gen_ai.request.model` 和 `gen_ai.response.model` 为什么要分开？

**结论**：前者是请求别名，后者是服务端实际执行或返回的模型版本。

**展开**：路由、别名、灰度或供应商升级都可能让两者不同；排查质量回归时 response model 更接近事实。

**陷阱**：只记请求模型会把后端路由变化隐藏掉。

#### 10. Prompt 和 Completion 应不应该全量记录？

**结论**：默认不应全量；先按数据分类、最小化、脱敏、截断、抽样和权限设计。

**展开**：普通流量只保存版本、长度、哈希、引用和 token；受控调试窗才采少量正文。

**陷阱**：“内部系统”不是忽略 PII、Secret 和保留期的理由。

#### 11. Head sampling 与 tail sampling 有什么差别？

**结论**：Head 在开始时低成本决策，Tail 在看过结果后可保留错误、慢和高成本 Trace。

**展开**：Tail 更适合 Agent，因为最终错误、循环次数和 token 在结束前未知。

**陷阱**：Head 已丢的数据 Tail 无法恢复；Tail 有状态并消耗内存。

#### 12. 多个 SDK 为什么会发生 global TracerProvider 冲突？

**结论**：一个进程通常只能明确设置一次全局 Provider，谁先初始化可能决定 sampler、processor 和 instrumentation 的归属。

**展开**：把 APM、Langfuse、自动 instrumentation 统一放进一个入口，由应用拥有 Provider。

**陷阱**：后初始化的 `sample_rate` 不一定能修改已经存在的 Provider。

#### 13. 为什么 Langfuse 会出现 orphan observation？

**结论**：常见原因是父 Span 未导出、过滤掉根节点、跨服务 Context 断裂或使用独立 Provider。

**展开**：检查 Trace ID、parent Span ID、filter、sampling 和 propagation，优先做树快照测试。

**陷阱**：看到子 Span 有 Trace ID，不代表同一后端一定收到了其父节点。

#### 14. token 和 cost 为什么会缺失？

**结论**：通常是字段名/版本不匹配、模型信息不足、Span end 后才写，或没有被识别成 generation。

**展开**：在 end 前写 request/response model 和 usage；自定义模型还要配置定价或显式 cost。

**陷阱**：网络层 Content-Length 不能替代 token usage。

#### 15. Resource、Instrumentation Scope 和 Span Attribute 如何分工？

**结论**：Resource 描述服务实例；Scope 描述产生 Span 的库；Attribute 描述本次操作。

**展开**：`service.name` 放 Resource，`support-agent.telemetry` 是 Scope，`gen_ai.request.model` 是 Span Attribute。

**陷阱**：把 request ID 放 Resource 会让资源基数爆炸。

#### 16. Trace 与 Metrics 如何配合？

**结论**：Metrics 发现总体异常并告警，Trace 解释单次异常的因果链。

**展开**：先看 P95/错误率按 release 分组，再从 exemplar 进入 APM/Langfuse Trace。

**陷阱**：不要把 user/session/trace ID 作为 Metric label。

### 高级题

#### 17. 请设计一条 RAG Trace。

**结论**：根下建立 retrieval 父 Span，内部包含查询改写、ACL/元数据过滤、BM25/Dense 并行召回、RRF、Rerank，最后是 generation。

**展开**：记录 index/query/prompt/model 版本、top_k、source IDs、分数区间、降级原因和各阶段时延；正文默认不记录。

**陷阱**：只有一个“RAG 500ms” Span 无法区分检索慢、精排慢还是模型慢；也不能从总时长相加推断并行耗时。

#### 18. 请设计一条多 Agent Trace。

**结论**：一个 turn 为根，Router、Supervisor、每个 Agent、generation 和 tool 是子层级；循环与交接要能看出顺序和原因。

**展开**：记录 agent/graph/prompt 版本、路由结果、步数、结束原因、工具 Schema 结果，不记录隐藏思维链。

**陷阱**：不要为每个 Agent 建完全独立且无法关联的 Trace，也不要把业务 Service 写库伪装成模型已经成功执行。

#### 19. OTel GenAI 语义约定能否直接当稳定标准？

**结论**：当前不能。GenAI 约定已迁到独立仓库，整体仍标为 Development，应锁定 instrumentation/semconv 版本。

**展开**：通过项目 adapter 封装属性，保存映射版本，用 canary Trace 和属性契约测试验证升级。

**陷阱**：不要直接追随 main 分支，也不要把“标准化方向”说成“所有字段已稳定”。

#### 20. 如何从 `gen_ai.system` 迁移到 `gen_ai.provider.name`？

**结论**：在适配层做一段双读或双写兼容期，查询和后端映射验证后再移除旧字段。

**展开**：为旧/新版本各准备固定 Span，检查 Langfuse 与 APM 中模型/provider 是否正确。

**陷阱**：全代码搜索替换不等于数据管线、仪表盘和历史查询已经兼容。

#### 21. Tail-sampling Collector 为什么需要 Trace affinity？

**结论**：尾采样器必须看到同一 Trace 的完整 Span 集合；如果被负载均衡到不同实例，每个实例都只看到残片。

**展开**：在 stateful tail sampler 前增加按 Trace ID 的一致路由，并为负载变化设计缓冲和重分配。

**陷阱**：普通随机负载均衡适合无状态 receiver，不适合直接放在尾采样实例之前。

#### 22. 如何同时接通用 APM 和 Langfuse？

**结论**：共用一个 Context/Provider 或统一发 Collector；APM 保留完整基础设施树，Langfuse 保留有根的 AI 子树。

**展开**：统一 Trace ID，分别配置内容过滤、保留期和权限；同一 Langfuse 项目只设一条出口。

**陷阱**：不要让 SDK 直发与 Collector 再发同一 Span；也不要用 Langfuse Trace endpoint 接 Logs。

#### 23. Span 名为什么必须低基数？

**结论**：名字用于聚合、面板和延迟分布；把用户/问题/ID 放进去会产生无限时间序列和不可比较数据。

**展开**：用 `execute_tool booking.create` 作名字，把 booking_id 放 Attribute。

**陷阱**：将 URL 实际路径作为名字可能把每个资源 ID 都变成新操作。

#### 24. 如何保证遥测管线本身可靠？

**结论**：业务不被遥测阻塞，同时监控 Processor/Exporter/Collector 的队列、重试、拒绝、丢弃和 canary 到达率。

**展开**：使用 Batch、有限队列、超时、重试和内存限制；短进程 flush；通过断网/限流/后端故障演练验证。

**陷阱**：同步 SimpleSpanProcessor 直发可能放大请求延迟；“不影响业务”也不能等同于无上限丢数据。

### 场景题

#### 25. APM 有完整 Trace，但 Langfuse 没有，怎么查？

**结论**：沿“Span Recording → Processor filter → Batch → endpoint/auth/protocol → ingestion mapping”逐段验证。

**展开**：先发固定 `gen_ai.*` canary；检查 Langfuse 默认过滤、自定义 filter 是否覆盖、区域 endpoint、Basic Auth、HTTP 协议、v4 header 和响应码。

**陷阱**：不要先归因于 UI 缓存；用 exporter/Collector 指标确定数据是否真正离开应用。

#### 26. Langfuse 中大量孤儿 Observation，怎么修？

**结论**：检查是否删除根/父 Span，再检查跨服务 Context、独立 Provider 和异步任务。

**展开**：导出一条完整 OTLP fixture，对比 APM 与 Langfuse 的 Span ID 集合，定位在哪一层丢父节点。

**陷阱**：单独放开所有子 Span 不能修复父链，反而可能加大成本。

#### 27. 打开 OTel 全量导出后账单暴涨，怎么办？

**结论**：按 instrumentation scope 和 Observation 类型统计，再为 Langfuse 建立保根的 LLM-focused filter。

**展开**：通常 HTTP、SQL、Redis、消息轮询数量远高于 generation；APM 可保留它们，Langfuse 只留必要子树。

**陷阱**：直接删除所有非 `gen_ai.*` Span 可能把应用根和 Agent 父节点删掉。

#### 28. 发现 Prompt 中有 PII，如何止损和改造？

**结论**：先停对应内容采集/轮换暴露凭证并按制度处理存量，再在应用边界做字段白名单和 export-stage masking。

**展开**：检查 SDK、Collector、对象存储、日志、备份和下游导出；建立 PII canary 测试和自动过期调试开关。

**陷阱**：只打开 Langfuse 服务端 masking 不能证明原始事件从未被写入 ingestion storage。

#### 29. 根 Trace P95 上升，但模型时延稳定，怎么定位？

**结论**：看关键路径上的排队、会话/幂等、检索、工具、DB、序列化和并行等待，不要只盯 generation。

**展开**：比较 Span self-time、并行重叠、队列等待和 release；用通用 APM 看资源/数据库，用 Langfuse 看 Agent/RAG。

**陷阱**：把所有子 Span 时长相加会误判并行链路。

#### 30. 如何做到错误全留、正常流量 1%？

**结论**：上游不能先随机丢弃；将 Trace 发到 Collector，由 tail policy 保留 Error/timeout，普通成功按 1% 采样。

**展开**：为 decision wait、内存和 Trace affinity 配容量；回放错误、成功、慢请求夹具验证策略。

**陷阱**：应用先用 1% head sampler 后，99% 中发生的错误已经不可恢复。

#### 31. Trace 顶层能看到 user/session，Observation 过滤却漏数，为什么？

**结论**：Langfuse v4 的 Observation 查询依赖相关属性出现在每个 Observation；只写根节点或传播太晚都会漏。

**展开**：在根附近调用 `propagate_attributes`，跨服务时谨慎使用 Baggage，检查每个目标 Span 的最终属性。

**陷阱**：修改已经结束的 Span 或之后才进入传播作用域不会回填旧子节点。

#### 32. 升级 SDK 后字段变灰或模型/usage 映射消失，怎么处理？

**结论**：先冻结升级，比较版本、scope 和 OTLP payload，再用兼容矩阵、属性映射和 canary 契约定位。

**展开**：GenAI 语义、Langfuse SDK 和 Server 任一层都可能变化；通过 adapter 做兼容，不在业务代码散改字段。

**陷阱**：不要只修 UI 查询；先证明 Exporter 发出的字段、Server 接收和映射各自是否正确。

---

## 14. 面试口述模板

### 14.1 30 秒版本

> 我把 OpenTelemetry 和 Langfuse 分层使用：OTel 负责统一 Trace、W3C 上下文传播、Collector 采集与多后端扇出，Langfuse 负责 generation、Prompt、token/cost、session 和质量 Score。一次用户轮次是一个 Trace，Router、Supervisor、RAG、模型和工具是 Span；同步调用传 trace context，Outbox 消费通常用 Link 表达因果，单消息场景也可按 messaging 约定选择 creation context 作为 parent。生产上一个全局 Provider，APM 收完整基础设施链，Langfuse 收保留根节点的 AI 子树；先在应用边界脱敏，再集中采样和治理，并确保同一 Langfuse 项目只有一条出口。

### 14.2 3 分钟项目版本

> 这个机房与电脑维修 Agent 的 Trace 目标是既能解释系统性能，也能解释回答质量。粒度上，我把一次用户对话轮次作为根 Trace，多轮用 Session ID 聚合，维修预约幂等键单独管理。根节点下面是 Router、Supervisor、咨询/预约 Agent；咨询 Agent 下有 BM25、Dense、RRF、Rerank 和 generation；预约 Agent 下有槽位模型、工程师排班查询、确认门禁、Booking Service 和数据库事务。Outbox 通知是异步链路，用消息 Context 或 Span Link 关联，不用 Trace ID 去重。
>
> 技术分工上，OpenTelemetry 是厂商无关骨架，负责 Provider、Context、Instrumentation、OTLP 和 Collector；Langfuse 是 GenAI 分析面，把 Span 映射成 agent、retriever、generation、tool，补充 Prompt 版本、token、cost 和 Score。已有 APM 时我优先共用一个全局 Provider，或者统一发到 Collector。APM 保留 HTTP、DB、队列等完整调用树；Langfuse 通过 LLM-focused filter 保留根节点和必要父链，避免每个基础设施 Span 都计费。
>
> 生产上我重点防四个问题。第一，SDK 直传和 Collector 不能同时把同一个 Span 发到同一 Langfuse 项目，否则重复计数。第二，过滤不能删根，否则出现 orphan。第三，Prompt、工具参数和检索正文默认不全量采集，先在应用边界做白名单和脱敏，Baggage 不放 PII 或密钥。第四，如果要保留错误、慢请求和高成本 Trace，上游不能先 head-sample 丢掉，而是尽量送到 Collector 做尾采样，并按 Trace ID 粘性路由。
>
> 最后，我不会只用一张 UI 截图宣称落地。验收要覆盖正常、错误、流式、跨服务、Outbox、脱敏、采样、Exporter 故障和重复上报，并监控 Collector 的队列、重试和丢弃。当前仓库只有目标设计，没有真实集成证据，所以面试时会明确把这部分称为生产演进方案。

### 14.3 面试官追问“为什么不用一个平台全包”

> 通用 APM 能告诉我哪个服务或数据库慢，但通常不擅长 Prompt 版本、token/cost、Agent 轨迹和质量 Score；Langfuse 擅长 AI 语义，但它的 OTLP Trace 入口不是完整 Logs/Metrics 平台。把 OTel 作为统一协议后，应用不绑定单一后端，Collector 负责治理和扇出，Langfuse 与 APM 各自做最擅长的分析。代价是要管理属性契约、采样一致性和避免双投，这些通过单一 Provider 所有者、唯一出口和契约测试控制。

---

## 15. 最后一页速记

### 15.1 架构

- OTel 是骨架，Langfuse 是 GenAI 分析面；
- 一轮一个 Trace，多轮一个 Session；
- 一个进程一个明确的 global Provider；
- 一个 Langfuse 项目一条出口；
- 生产多服务优先 Collector；
- App → Collector 可 gRPC；Collector → Langfuse 只用 HTTP OTLP。

### 15.2 语义

- 根：`support.chat_turn`；
- Agent：`invoke_agent`；
- RAG：`retrieval`；
- 模型：`chat` / `embeddings`；
- 工具：`execute_tool`；
- 新字段优先 `gen_ai.provider.name`；
- GenAI 语义约定仍是 Development。

### 15.3 传播

- 父子关系：Trace Context；
- 业务维度：Baggage；
- 消息/批处理：默认/通常用 Link；单消息可按约定把 creation context
  作为 parent；
- Baggage 无 Secret/PII；
- `propagate_attributes` 尽早调用；
- 幂等键不是 Trace ID。

### 15.4 成本与安全

- Filter 选 Span，Sampler 选 Trace；
- Head 丢弃后 Tail 不能恢复；
- Tail sampling 多实例需要 Trace affinity；
- 保留根和必要祖先；
- Prompt/Completion 默认最小化；
- 高基数不进 Metrics 标签；
- 短任务 flush，长驻服务 shutdown。

### 15.5 验收

- 正常、错误、慢、流式；
- 跨 HTTP、线程/任务、Outbox；
- user/session/version 传播；
- token/cost/status 正确；
- 无 PII/Secret；
- 无 orphan、无重复；
- exporter 故障不拖垮业务；
- Collector 自监控完整。

---

## 16. 版本风险与官方资料

### 16.1 必须知道的版本风险

OpenTelemetry 的 GenAI 语义约定已从 core semantic-conventions 仓库迁到独立的 `open-telemetry/semantic-conventions-genai` 仓库。本文快照时：

- 文档整体仍标为 **Development**；
- GitHub Releases 页面没有稳定 release；
- README 的 Schema URL 仍是 TODO。

工程结论：

1. 锁定 OTel SDK、instrumentation、Collector 和 semconv adapter 版本；
2. 不直接依赖仓库 main 分支作为生产协议；
3. 对模型、usage、agent、tool 和 retrieval 字段做契约测试；
4. 升级采用 canary Trace、双读/双写兼容和可回滚发布；
5. 仪表盘、过滤器和采样规则也纳入版本化测试。

### 16.2 Langfuse 官方资料

- [Langfuse Server v4.9.0 Release](https://github.com/langfuse/langfuse/releases/tag/v4.9.0)
- [Langfuse Python SDK v4.14.4 Release](https://github.com/langfuse/langfuse-python/releases/tag/v4.14.4)
- [Langfuse JS/TS SDK v5.10.0 Release](https://github.com/langfuse/langfuse-js/releases/tag/v5.10.0)
- [Langfuse 与 OpenTelemetry 原生集成、端点、认证及属性映射](https://langfuse.com/integrations/native/opentelemetry)
- [Langfuse 兼容性矩阵](https://langfuse.com/docs/compatibility)
- [已有 OpenTelemetry 设置如何与 Langfuse 共存](https://langfuse.com/faq/all/existing-otel-setup)
- [Langfuse Observability 数据模型](https://langfuse.com/docs/observability/data-model)
- [Langfuse Tracing 最佳实践](https://langfuse.com/docs/observability/best-practices)
- [Langfuse v4 的 Observation-first 迁移说明](https://langfuse.com/integrations/native/opentelemetry/migration-to-v4)
- [Langfuse SDK 高级过滤、采样与 masking](https://langfuse.com/docs/observability/sdk/advanced-features)
- [Langfuse Python/JS 数据 Masking](https://langfuse.com/docs/observability/features/masking)
- [Langfuse Trace ID 与分布式追踪](https://langfuse.com/docs/observability/features/trace-ids-and-distributed-tracing)
- [Langfuse Token 与成本追踪](https://langfuse.com/docs/observability/features/token-and-cost-tracking)
- [Langfuse Scores 与 Evaluation](https://langfuse.com/docs/evaluation/scores/overview)
- [Langfuse SDK 排障指南](https://langfuse.com/docs/observability/sdk/troubleshooting-and-faq)
- [Langfuse 服务端数据脱敏](https://langfuse.com/self-hosting/security/data-masking)
- [Langfuse Self-hosting 架构](https://langfuse.com/self-hosting)
- [Langfuse Python v4.14.4 SpanProcessor 源码](https://github.com/langfuse/langfuse-python/blob/v4.14.4/langfuse/_client/span_processor.py)
- [Langfuse Python v4.14.4 Provider/资源管理源码](https://github.com/langfuse/langfuse-python/blob/v4.14.4/langfuse/_client/resource_manager.py)
- [Langfuse Python v4.14.4 Client 与自定义 Span Exporter 入口](https://github.com/langfuse/langfuse-python/blob/v4.14.4/langfuse/_client/client.py)
- [Langfuse Server v4.9.0 OTLP Trace endpoint 实现](https://github.com/langfuse/langfuse/blob/v4.9.0/web/src/pages/api/public/otel/v1/traces/index.ts)

### 16.3 OpenTelemetry 官方资料

- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Collector 架构与 Pipeline](https://opentelemetry.io/docs/collector/architecture/)
- [Collector 扩缩容与 Tail Sampling](https://opentelemetry.io/docs/collector/scaling/)
- [Trace API 规范](https://opentelemetry.io/docs/specs/otel/trace/api/)
- [Trace SDK 规范](https://opentelemetry.io/docs/specs/otel/trace/sdk/)
- [Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [Baggage 规范](https://opentelemetry.io/docs/specs/otel/baggage/api/)
- [消息系统 Span 语义与 Link](https://opentelemetry.io/docs/specs/semconv/messaging/messaging-spans/)
- [独立的 OpenTelemetry GenAI 语义约定仓库](https://github.com/open-telemetry/semantic-conventions-genai)
- [GenAI Inference Span 约定](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md)
- [GenAI Agent Span 约定](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md)
- [Collector Filter Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor)
- [Collector Redaction Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor)

### 16.4 使用资料时的口径

- 官方文档说明产品支持什么；
- GitHub release 和带 tag 的源码说明本文快照核验了哪个版本；
- 开放 issue 只能说明存在待确认问题，不能替代稳定行为；
- 所有精确版本、端点和字段在生产实施前重新验证；
- 本文没有使用公开面经的频次数字冒充统计结论，题库依据是技术方案本身的常见追问链。

---

## 17. 自测：能答出这些才算真正理解

不看正文，尝试在十分钟内回答：

1. 画出本项目一轮对话的 Span 树，并指出关键路径；
2. 解释为什么 Session ID、Trace ID 和幂等键不能合并；
3. 在“SDK 直传、共享 Provider、Collector、独立 Provider”中做选型；
4. 解释 Langfuse 为什么只支持 HTTP OTLP 仍不影响应用到 Collector 使用 gRPC；
5. 设计“错误全留、普通 1%”的采样管线；
6. 解释 filter 如何产生 orphan，以及如何测试；
7. 列出绝不能放进 Baggage 的数据；
8. 描述一个流式 generation Span 的开始、TTFT、usage 和结束；
9. 说明如何证明没有 SDK/Collector 双投；
10. 说明为什么服务端 masking 不等于原始数据未离开应用；
11. 给出 Outbox 消费使用 Link 的理由；
12. 说出 GenAI 语义约定的当前稳定性边界和升级策略。

如果这些问题能用“结论—实现—异常—权衡—验证”讲清楚，就不只是会看 Langfuse 页面，而是已经具备设计生产 Trace 方案的能力。
