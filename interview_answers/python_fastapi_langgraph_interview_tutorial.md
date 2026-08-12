# Python + FastAPI + LangGraph：AI 应用开发面试重点教程

> 版本：2026-08-10  
> 定位：面试冲刺，不是三套技术的完整手册。  
> 贯穿案例：企业售后咨询与上门预约 Agent。  
> 配套代码：[FastAPI + LangGraph 教学 Demo](./python_fastapi_langgraph_demo/)

## 0. 先说结论：面试官真正想确认什么

近期面经和招聘要求的共同点很稳定：公司并不缺“能调一次大模型 API”的人，更想确认候选人能否把不确定、昂贵、会失败的模型能力，装进一个可验证、可恢复、可扩展的后端系统。

面试重点可以压缩成五层：

| 层次 | 面试官想听到的能力 | 本教程优先级 |
|---|---|---:|
| Python | 异步、并发、类型、异常、测试 | 必须会写 |
| FastAPI | 接口契约、依赖、生命周期、流式、错误语义 | 必须会写 |
| LangGraph | State、Reducer、路由、checkpoint、interrupt | 必须会写 |
| AI 工程 | RAG、工具调用、评测、Tracing、安全 | 必须会讲 |
| 生产化 | 超时、重试、幂等、限流、持久化、成本 | 决定回答上限 |

公开面经不是严格统计，只能用来判断题型。一个 2026 年牛客复盘称其 40 多场交流中有 15 场现场编码，项目动机和取舍被反复追问；另一批 2025—2026 面经持续出现 RAG、Agent 记忆、LangGraph checkpoint、HITL、FastAPI async、延迟和评测。当前招聘说明也常把 Python、FastAPI、LangGraph、RAG、测试、Tracing 和安全放在同一岗位里。

可参考：

- [2026 春招 40 多场复盘：项目、RAG、Agent 与现场编码](https://ac.nowcoder.com/discuss/1626144?channel=-1&source_id=0&type=0)
- [2025 大模型应用开发面经：RAG、Agent、FastAPI 与工程基础](https://www.nowcoder.com/feed/main/detail/129eaa1c20444651ac3b932e200d3da4?sourceSSR=dynamic)
- [2026 快手 AI 应用开发面经：LangGraph、checkpoint、记忆与算法题](https://www.nowcoder.com/discuss/882573284426932224?sourceSSR=enterprise)
- [当前 EPAM Agentic/RAG 岗位：FastAPI、LangGraph、SSE、评测与安全](https://careers.epam.com/en/vacancy/senior-ai-engineer-agentic-and-rag-systems-blty5mp8mok8tyd6a36_en)
- [当前埃森哲上海 Agent 岗位：工作流、RAG、异步后端与部署](https://www.accenture.com/gb-en/careers/jobdetails?id=14477302_en)

### 这份教程刻意不展开什么

- Python 元类、描述符、解释器源码；
- FastAPI 自定义 OpenAPI、复杂 WebSocket 集群；
- LangGraph Functional API 的全部细节；
- Transformer 公式推导、全套微调与推理优化；
- Kubernetes、消息队列和数据库的完整运维。

如果目标 JD 明确偏算法、训练或平台，再单独补这些内容。现在先把最常被追问的主线打透。

---

# 第一部分：Python 面试重点

## 1. 类型不是装饰，而是边界

AI 应用中至少有三种结构不要混：

- TypedDict：描述 LangGraph 的轻量 State，主要帮助静态检查，运行时仍是普通 dict；
- dataclass：适合内部领域对象和运行时上下文，支持默认值；
- Pydantic BaseModel：适合不可信的外部输入、输出和模型结构化结果，提供运行时校验。

~~~python
from dataclasses import dataclass
from typing import TypedDict
from uuid import UUID
from pydantic import BaseModel, ConfigDict, Field


class AgentState(TypedDict, total=False):
    intent: str
    answer: str


@dataclass(frozen=True)
class Actor:
    tenant_id: str
    user_id: str


class ChatRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    thread_id: str = Field(min_length=1, max_length=128)
    request_id: UUID
    message: str = Field(min_length=1, max_length=4_000)
~~~

高频追问：

1. TypedDict 会在运行时校验吗？  
   不会，它主要给类型检查器使用。外部 JSON 要用 Pydantic 校验。

2. Optional 或 str | None 是否表示字段可省略？  
   不一定。name: str | None 仍是必填但允许为 None；name: str | None = None 才可省略。

3. 为什么 extra="forbid"？  
   Pydantic 默认可能忽略多余字段。对外命令型 API 禁止未知字段，能更早发现客户端版本或参数错误。

## 2. async/await：最容易答得似是而非

一句话：

> asyncio 是单线程内的协作式并发；协程会在所 await 的 awaitable **实际挂起**时让出事件循环。它不限于 I/O（也可能是 sleep、锁或队列），而 await 一个已完成对象可能不会让出。async 不等于多线程，也不让 CPU 推理自动变快。

### 2.1 连续 await 仍是串行

~~~python
# 串行：先等检索，再等画像
docs = await retrieve(query)
profile = await load_profile(user_id)
~~~

两个任务相互独立时才并发：

~~~python
import asyncio


async def build_context(query: str, user_id: str):
    async with asyncio.timeout(3):
        async with asyncio.TaskGroup() as group:
            docs_task = group.create_task(retrieve(query))
            profile_task = group.create_task(load_profile(user_id))

    return docs_task.result(), profile_task.result()
~~~

TaskGroup 比无约束地 create_task 更容易管理生命周期。一个子任务失败时，它会取消同组其他任务，并在退出时汇总异常；适合“这些步骤共同组成一次请求”的场景。Python 官方说明见 [Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html)。

### 2.2 并发必须有上限

~~~python
import asyncio

llm_slots = asyncio.Semaphore(8)


async def call_llm_limited(prompt: str) -> str:
    async with llm_slots:
        async with asyncio.timeout(12):
            return await llm_client.generate(prompt)
~~~

如果对任意长度列表直接 gather，可能耗尽连接池、触发 429 或让账单失控。生产中还要区分：

- 单实例并发上限：进程内 Semaphore；
- 集群全局配额：网关、Redis 或供应商配额系统；
- 外部服务连接池上限：HTTP/数据库客户端配置。

### 2.3 取消不是普通错误

~~~python
import asyncio


async def stream_answer():
    try:
        async for token in llm_client.stream():
            yield token
    except asyncio.CancelledError:
        await llm_client.aclose()
        raise
    finally:
        await release_request_resources()
~~~

客户端断开或超时可能触发取消。清理后通常要重新抛出 CancelledError；吞掉它会破坏 timeout 和 TaskGroup 的语义。

### 2.4 I/O 并发、线程与进程怎么选

| 工作 | 推荐方式 | 原因 |
|---|---|---|
| 异步 LLM/HTTP/数据库调用 | async SDK + await | 等待时释放事件循环 |
| 只有同步 SDK 的短阻塞 I/O | 普通 def 入口或 asyncio.to_thread | 移出事件循环 |
| 纯 Python CPU 密集计算 | 进程池/独立服务 | 默认 CPython 下线程受 GIL 限制 |
| GPU/本地大模型推理 | 独立推理服务 | 便于批处理、显存治理和扩缩容 |
| 必须持久化的长任务 | 外部任务队列 | 可恢复、可重试、可观测 |

不要绝对地说“Python 永远有 GIL”：Python 3.13 起存在可选 free-threaded 构建；面试里说“默认 CPython 构建下”更准确。

## 3. Python 高频短题

### 可变默认参数

~~~python
# 错误：所有调用共享同一个列表
def add_event(event: str, events=[]):
    events.append(event)
    return events


# 正确
def add_event(event: str, events: list[str] | None = None):
    result = [] if events is None else events
    result.append(event)
    return result
~~~

### 浅拷贝与深拷贝

- 浅拷贝只复制外层容器，内部可变对象仍共享；
- 深拷贝递归复制，但成本高，也未必适合数据库连接、锁等资源对象；
- 最好的方案常是使用不可变对象，或明确复制需要变化的字段。

### 生成器为什么适合流式 AI

生成器按需产生数据，不必等全部 token 或文档都进入内存。异步生成器还能在每次 yield 之间 await 上游模型。

### 装饰器与上下文管理器

- 装饰器适合重试、Tracing、权限检查，但要避免把所有业务逻辑藏在装饰器里；
- with/async with 适合连接、事务、锁和生命周期，保证异常时清理；
- FastAPI 的 yield 依赖和 lifespan，本质上都利用了上下文管理思想。

---

# 第二部分：FastAPI 面试重点

## 4. FastAPI 在 AI 系统中的位置

FastAPI 不是 Agent，也不是模型服务器。它负责 HTTP/ASGI 边界：

~~~text
客户端
  ↓ HTTP / SSE
FastAPI：认证、校验、限流、错误码、请求上下文
  ↓
Application Service：超时、幂等、事务、业务规则
  ↓
LangGraph：状态与流程编排
  ↓
LLM / RAG / Tool / Database
~~~

口述请求链路：

> Uvicorn 接收 HTTP 并交给 ASGI 应用，经过中间件和路由匹配；FastAPI 解析参数与依赖图，Pydantic 校验请求；处理函数调用业务层；返回值再经过 response model 过滤、校验和序列化，最后返回客户端。

## 5. async def 和 def 到底怎么选

FastAPI 官方给出的关键语义是：

- 路由或依赖写成普通 def，FastAPI 会在线程池执行；
- 路由或依赖写成 async def，直接在事件循环执行；
- 但你在 async def 内手动调用的普通工具函数，不会被 FastAPI 自动移到线程池。

~~~python
@app.post("/chat")
async def chat(body: ChatRequest):
    # LangChain Runnable / ChatModel 的异步入口是 ainvoke
    return await llm_client.ainvoke(body.message)


@app.post("/bad")
async def bad_chat(body: ChatRequest):
    # 错误：同步 HTTP 和 sleep 会堵住当前 worker 的事件循环
    response = requests.post(MODEL_URL, json={"q": body.message})
    time.sleep(1)
    return response.json()
~~~

官方依据：[FastAPI Concurrency and async/await](https://fastapi.tiangolo.com/async/)。

面试回答不要停在“FastAPI 很快”。应补一句：

> 它对 I/O 密集 AI 网关很合适，但 CPU/GPU 推理不会因 async 变快；阻塞调用、连接池和上游限流仍决定吞吐。

## 6. Pydantic：结构校验不等于业务正确

~~~python
from datetime import datetime
from pydantic import BaseModel, ConfigDict, Field, model_validator


class BookingRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    engineer_id: str = Field(min_length=1)
    start_at: datetime
    end_at: datetime

    @model_validator(mode="after")
    def validate_range(self):
        if self.start_at >= self.end_at:
            raise ValueError("end_at 必须晚于 start_at")
        return self
~~~

这个模型能证明：

- 字段存在；
- 类型和格式合法；
- 开始时间早于结束时间。

它不能证明：

- 当前用户能操作该租户；
- 工程师真实存在且有空；
- 用户已经确认；
- 并发请求不会重复占用同一时段。

这些必须在鉴权、Service、事务和数据库约束中完成。不要在 Pydantic validator 里查询数据库或调用模型。

response_model 的价值也不只是生成文档：它会过滤未声明字段，可减少密码哈希、内部 Prompt、密钥和调试信息误泄露。

Pydantic v2 常用迁移口径：

| v1 | v2 |
|---|---|
| .dict() | .model_dump() |
| parse_obj() | model_validate() |
| @validator | @field_validator |
| @root_validator | @model_validator |
| orm_mode | from_attributes |

参考：[Pydantic Models](https://docs.pydantic.dev/latest/concepts/models/) 与 [Validators](https://docs.pydantic.dev/latest/concepts/validators/)。

## 7. Dependency Injection：不是为了少传两个参数

依赖适合承载：

- 当前用户、租户和权限；
- 数据库 Session；
- 配置与客户端；
- 请求级 Trace；
- 限流或 feature flag。

~~~python
from collections.abc import AsyncIterator
from typing import Annotated
from fastapi import Depends


async def get_session() -> AsyncIterator[AsyncSession]:
    async with session_factory() as session:
        yield session


SessionDep = Annotated[AsyncSession, Depends(get_session)]


@app.get("/bookings/{booking_id}")
async def get_booking(booking_id: str, session: SessionDep):
    return await booking_service.get(session, booking_id)
~~~

同一依赖在单个请求内默认只执行一次并复用。yield 前创建资源，yield 后清理；官方说明见 [Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/) 和 [Dependencies with yield](https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/)。

### Middleware 和 Depends 怎么选

| 需求 | 更适合 |
|---|---|
| 所有请求的 request_id、基础日志、耗时 | Middleware |
| 只有部分路由需要的用户与权限 | Depends |
| 需要把解析结果传给 endpoint | Depends |
| 统一响应头 | Middleware |
| 业务级租户授权 | Depends + Service 再校验 |

## 8. lifespan：共享客户端只初始化一次

~~~python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from httpx import AsyncClient


@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.http = AsyncClient(timeout=10)
    app.state.graph = build_graph(checkpointer)
    yield
    await app.state.http.aclose()


app = FastAPI(lifespan=lifespan)
~~~

适合放在 lifespan 的资源：

- HTTP/数据库连接池；
- LangGraph 编译结果；
- tokenizer、小型只读模型；
- tracing exporter。

不要每个请求重建这些资源。也要知道：每个 worker 是独立进程，每个 worker 都会执行一次 lifespan；四个 worker 可能加载四份模型。FastAPI 当前推荐 lifespan，而不是旧式 startup/shutdown 事件。参考 [Lifespan Events](https://fastapi.tiangolo.com/advanced/events/)。

## 9. 状态码要表达“客户端下一步做什么”

| 状态码 | 在 AI 应用中的典型含义 |
|---:|---|
| 200 | 同步处理完成 |
| 201 | 资源已创建 |
| 202 | 任务已接收，稍后查询 job |
| 400 | 语义错误或统一约定的坏请求 |
| 401 | 未认证 |
| 403 | 已认证但无权限 |
| 404 | 资源不存在，且不泄露他人资源存在性 |
| 409 | 幂等键冲突、状态冲突、时段冲突 |
| 422 | FastAPI 默认请求校验失败 |
| 429 | 限流 |
| 502 | 上游返回无效响应 |
| 503 | 服务暂不可用 |
| 504 | 上游超时 |

HTTPException 要 raise，不是 return。

## 10. 流式输出：优先会 SSE

AI 聊天通常是服务端单向推 token 或进度，SSE 比 WebSocket 简单；只有持续双向实时交互才优先 WebSocket。

FastAPI 0.135.0 起提供原生 SSE；配套 Demo 因此把最低版本锁在 0.135。旧版本可用 StreamingResponse 或第三方 EventSourceResponse。面试不必背版本 API，但必须说出：

- Content-Type 为 text/event-stream；
- token、progress、done、error 应有明确事件类型；
- 代理缓冲和 keepalive；
- 客户端断开后取消上游调用；
- 流已经开始后不能再改成普通 JSON 错误响应；
- 需要背压和并发上限。

~~~python
from collections.abc import AsyncIterable
from fastapi.sse import EventSourceResponse, ServerSentEvent


@app.post("/chat/stream", response_class=EventSourceResponse)
async def stream_chat(body: ChatRequest) -> AsyncIterable[ServerSentEvent]:
    async for token in graph_token_stream(body):
        yield ServerSentEvent(event="token", data={"text": token})
    yield ServerSentEvent(event="done", data={"ok": True})
~~~

参考：[FastAPI Server-Sent Events](https://fastapi.tiangolo.com/tutorial/server-sent-events/)。

## 11. BackgroundTasks 不是可靠任务队列

适用：写一条非关键审计、发送可丢失通知等短任务。

不适用：

- 大文件解析和 embedding；
- 长时间本地推理；
- 必须重试、不能丢失的业务动作；
- 要跨机器调度的任务。

这些应进入持久化任务队列，接口返回 202 + job_id。FastAPI 官方也建议重计算使用更完整的队列工具：[Background Tasks](https://fastapi.tiangolo.com/tutorial/background-tasks/)。

## 12. 测试：模型边界必须可替换

~~~python
from fastapi.testclient import TestClient


async def fake_llm():
    return DeterministicFakeLLM(answer="固定答案")


def test_chat():
    app.dependency_overrides[get_llm] = fake_llm
    try:
        with TestClient(app) as client:
            response = client.post("/chat", json={"message": "hi"})
        assert response.status_code == 200
    finally:
        app.dependency_overrides.clear()
~~~

应该覆盖：

- 正常输出；
- 422 输入；
- 上游 429、超时、无效 JSON；
- 模型返回错误工具参数；
- 流中断和取消；
- 重试是否重复副作用；
- 跨租户 thread_id 是否被拒绝。

参考：[Testing](https://fastapi.tiangolo.com/tutorial/testing/) 与 [Dependency Overrides](https://fastapi.tiangolo.com/advanced/testing-dependencies/)。

---

# 第三部分：LangGraph 面试重点

## 13. LangChain 和 LangGraph 的关系

建议回答：

> LangChain 提供模型、工具、检索器和高层 Agent 抽象；LangGraph 是更底层的有状态编排运行时，擅长显式循环、条件路由、持久化、人审中断和故障恢复。LangChain 的 create_agent 底层也是 LangGraph。流程固定、一步模型调用能解决的问题不需要上 LangGraph。

不要回答“LangGraph 是 LangChain 的升级版”。二者职责不同。

## 14. State、Node、Edge、Reducer

~~~python
import operator
from typing import Annotated, Literal, TypedDict
from langgraph.graph import END, START, StateGraph


class State(TypedDict, total=False):
    query: str
    intent: Literal["faq", "booking"]
    answer: str
    traces: Annotated[list[str], operator.add]


def classify(state: State):
    intent = "booking" if "预约" in state["query"] else "faq"
    return {"intent": intent, "traces": [f"intent:{intent}"]}


def route(state: State) -> Literal["retrieve", "draft_booking"]:
    return "draft_booking" if state["intent"] == "booking" else "retrieve"


builder = StateGraph(State)
builder.add_node("classify", classify)
builder.add_node("retrieve", retrieve)
builder.add_node("draft_booking", draft_booking)
builder.add_edge(START, "classify")
builder.add_conditional_edges("classify", route)
builder.add_edge("retrieve", END)
builder.add_edge("draft_booking", END)
graph = builder.compile()
~~~

心智模型：

- State 是应用快照，不是随意堆聊天文本的垃圾袋；
- Node 是同步或异步 Python 函数，读 State，返回局部更新；
- Edge 只描述控制流；
- Reducer 决定一个字段如何合并更新；没有 Reducer 时默认覆盖；
- 同一 superstep 的多个节点可以并行。

消息字段优先用 MessagesState 或 add_messages，而不是简单 operator.add，因为消息可能按 ID 更新、删除或反序列化。官方依据：[Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)。

### 高频坑：并发写同一字段

两个并行节点都写 answer，而 answer 没有 Reducer，运行时不知道如何合并，可能出现 INVALID_CONCURRENT_GRAPH_UPDATE。解决方案不是“随便追加列表”，而是先确认业务语义：

- 本应只有一个答案：重新设计路由；
- 确实要收集多个结果：定义列表 Reducer；
- 结果顺序重要：保存 index，汇合后显式排序。

## 15. 条件边、Command 和 Send

| 能力 | 何时使用 |
|---|---|
| conditional edge | 只需要根据 State 选择下一节点 |
| Command | 节点需要同时更新 State 和决定 goto，或恢复 interrupt |
| Send | 运行时才知道要创建多少个、输入各异的并行任务 |

~~~python
from typing import Literal
from langgraph.types import Command


def validate(state: State) -> Command[Literal["retrieve", "fallback"]]:
    if state["query"].strip():
        return Command(
            update={"traces": ["validated"]},
            goto="retrieve",
        )
    return Command(goto="fallback")
~~~

关键坑：

- Command(goto=...) 不会自动取消这个节点已经声明的静态边；二者同时存在可能两条路都跑；
- 普通同一会话的新一轮输入传 dict；
- 只有恢复 interrupt 时，才把 Command(resume=...) 作为 invoke 输入；
- Send 适合 map-reduce，例如为动态文档列表创建并行摘要任务。

参考：[Graph API 中的 Command 与 Send](https://docs.langchain.com/oss/python/langgraph/graph-api)。

## 16. Tool Calling：模型提议，应用执行

ReAct/工具循环可以压缩成：

~~~text
START → model → 是否有 tool_calls？
                 ├─ 有 → tools → model
                 └─ 无 → END
~~~

面试必须说出：

1. 工具名称、描述和参数 Schema会进入模型上下文；
2. 模型生成结构化 tool call，但不会自己执行代码；
3. ToolNode 或应用运行时校验并执行；
4. 工具结果以对应 tool_call_id 返回模型；
5. 权限、租户、参数、额度和副作用确认必须由工具层再次校验；
6. 工具超时、失败和重复执行要有明确策略。

工具结果也是不可信数据，可能包含 Prompt Injection。不要把“模型决定调用”误说成“模型拥有数据库权限”。

## 17. Checkpointer、thread_id 和长期记忆

| 概念 | 作用域 | 用途 |
|---|---|---|
| Checkpointer | 单个 thread | 状态快照、短期记忆、恢复、HITL、time travel |
| Store | 跨 thread | 用户偏好、长期事实、共享数据 |
| thread_id | 一条会话历史 | 定位 checkpoint |
| user_id | 一个用户 | 长期记忆 namespace、鉴权 |

~~~python
from langgraph.checkpoint.memory import InMemorySaver

graph = builder.compile(checkpointer=InMemorySaver())
# 这是服务端生成或由规范 tuple 摘要得到的不透明 ID，不是字符串直接拼接
config = {"configurable": {"thread_id": "thread-7f0c..."}}

first = graph.invoke({"messages": [{"role": "user", "content": "我叫小林"}]}, config)
second = graph.invoke({"messages": [{"role": "user", "content": "我叫什么？"}]}, config)
~~~

面试加分点：

- InMemorySaver 只适合测试；
- 本地可用 SQLite saver；生产多实例通常用数据库型 saver，如 Postgres；
- API 不应允许“知道 thread_id 就能恢复”，要把 thread 归属绑定到已认证租户和用户；
- 不要用 `tenant:user:thread` 直接拼 key：字段本身若含分隔符会碰撞。应使用服务端生成的 opaque ID + owner 表，至少也要对规范 tuple 做无歧义编码/HMAC；
- checkpoint 保存状态，不代表能把无限历史全塞进模型上下文；仍需 trim、delete、summarize；
- 短期对话状态不能替代订单、余额、预约等业务事实表。

官方依据：[Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)。

## 18. interrupt 与 Human-in-the-loop

~~~python
from langgraph.types import Command, interrupt


def approval_node(state: State):
    approved = interrupt({
        "action": "create_booking",
        "preview": state["pending_action"],
    })
    if type(approved) is not bool:
        raise ValueError("resume value must be a strict bool")
    return {"approved": approved}


config = {"configurable": {"thread_id": "thread-7f0c..."}}

# 第一次运行，在 approval_node 暂停
paused = graph.invoke(input_state, config=config)
pending = paused["__interrupt__"][0]

# 使用同一个 thread_id，并把决定精确绑定到该 interrupt ID
finished = graph.invoke(
    Command(resume={pending.id: True}),
    config=config,
)
~~~

最重要的执行语义：

> 恢复时节点会从函数开头重新执行，不是从 Python 调用栈中 interrupt 那一行继续。

因此：

- interrupt 前不要扣款、写订单、发邮件；
- 如果不可避免，操作必须幂等；
- 更好的图结构是“生成草案 → interrupt 确认 → 独立副作用节点”；
- payload 和 resume value 应可 JSON 序列化；
- HTTP 审批请求要携带 `request_id + interrupt_id`（必要时再带 checkpoint 版本或 action digest），不能只凭 thread_id 恢复“当前动作”；
- 同一个审批只能原子地从 pending 转成 approved/rejected；重复同决定可返回旧结果，相反决定应报 409；
- Runtime context 不会作为普通 State 自动写入 checkpoint；每次 resume 都要从已认证请求重新构造并传入，不能信任模型状态里的 tenant/user；
- 不要用普通 try/except 吞掉 interrupt 的控制流；
- interrupt 的顺序不要在版本升级后随意改变。

官方依据：[Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)。

## 19. Streaming、Subgraph 和多 Agent

传统 stream_mode 常见含义：

- updates：节点产生的局部更新；
- values：每一步后的完整 State；
- messages：模型 token 与 metadata；
- custom：节点主动上报的进度；
- debug/tasks/checkpoints：排错和观测。

当前 LangGraph 文档还提供事件流 API，用统一流暴露消息、状态、子图生命周期、interrupt 和最终结果。版本变化较快，面试时说清概念，并以项目锁定版本的官方文档为准：[Event streaming](https://docs.langchain.com/oss/python/langgraph/event-streaming)。

多 Agent 不是默认答案。优先顺序通常是：

1. 一次模型调用；
2. 确定性 workflow；
3. 单 Agent + tools；
4. 有明确上下文隔离或职责边界时再用 subgraph/multi-agent。

多 Agent 的成本是额外 token、延迟、错误面和可观测难度。它的价值应来自上下文隔离、权限隔离、可并行分工或不同模型专长，而不是为了简历上多一个名词。

---

# 第四部分：把 FastAPI 和 LangGraph 串起来

## 20. 配套 Demo 的架构

~~~mermaid
flowchart LR
    C["Client"] --> F["FastAPI：校验 / 身份 / HTTP"]
    F --> G["LangGraph：State / Route / Checkpoint"]
    G --> Q["FAQ：Retrieve → Answer"]
    G --> D["Booking：Draft"]
    D --> H["Interrupt：人工确认"]
    H -->|approve| S["BookingService：幂等写入"]
    H -->|reject| R["取消"]
~~~

代码位置：

- [应用代码](./python_fastapi_langgraph_demo/app.py)
- [接口测试](./python_fastapi_langgraph_demo/test_app.py)
- [运行说明](./python_fastapi_langgraph_demo/README.md)

这个 Demo 故意不用真实 LLM：

- 分类器是确定性规则，便于看懂流程；
- RAG 是固定文档桩；
- InMemorySaver 展示 checkpoint；
- BookingService 会再次校验可信 Runtime context、动作字段和 operation_id；
- 相同 request_id + 相同 payload 返回缓存结果，不同 payload 返回 409；
- 请求记录有 processing / waiting_approval / succeeded 状态；若图完成后 HTTP 响应缓存失败，重试会用 checkpoint 和 operation_id 对账，而不是永久卡死；
- 请求头身份只是教学替身，不能当成真实鉴权。

面试中可以这样解释：

> 我先用确定性替身把编排、恢复和接口契约测通，再把分类节点替换为模型结构化输出，把检索节点替换为真实混合检索。这样工作流测试不依赖模型概率和外部账单。

## 21. 运行顺序

~~~powershell
cd ./interview_answers/python_fastapi_langgraph_demo
python -m venv .venv
./.venv/Scripts/Activate.ps1
python -m pip install -r requirements.txt
uvicorn app:app --reload
~~~

打开 http://127.0.0.1:8000/docs。

### FAQ 请求

POST /v1/chat，带请求头：

~~~text
X-Tenant-ID: tenant-a
X-User-ID: user-1
~~~

请求体：

~~~json
{
  "thread_id": "chat-001",
  "request_id": "018f6f10-90c3-7b52-8a64-9b24940d55d7",
  "message": "洗衣机不进水应该怎么检查？"
}
~~~

FAQ 路径直接完成。预约类输入会在 review_booking 节点 interrupt，返回 approval_required、request_id 和 interrupt_id。随后把这两个 ID 原样带回，用相同 thread_id 调 POST /v1/reviews：

~~~json
{
  "thread_id": "chat-001",
  "request_id": "018f6f10-90c3-7b52-8a64-9b24940d55d7",
  "interrupt_id": "响应中的 interrupt_id",
  "approved": true
}
~~~

## 22. Demo 中值得被追问的设计

### 为什么内部 thread key 来自 tenant_id、user_id 和公开 thread_id

客户端提供的 thread_id 只是公开标识。Demo 把可信 Actor tuple 做规范序列化和 SHA-256 摘要，避免 `a:b:c` 拼接产生分隔符碰撞。它演示的是 namespace 隔离，不是真实鉴权：裸 X-User-ID 可以伪造，生产必须使用已验证的 token claims，并检查 owner ACL。

生产中还要：

- 在数据库中保存 thread 归属；
- 每次读取、恢复、删除都重新授权；
- Store 的 namespace 同样包含 tenant/user；
- 管理员跨租户操作单独审计。

### 为什么副作用在 interrupt 之后

恢复会重跑节点。若先创建预约再 interrupt，重复恢复可能重复创建。Demo 使用“草案 → 确认 → 幂等创建”的结构；相同 operation_id 若对应不同动作会冲突，而不是静默复用旧结果。生产再用数据库唯一约束兜底。

### 为什么 graph 在 lifespan 编译

图结构、客户端和连接池可跨请求复用。每次请求重新构图既浪费资源，也容易让内存 checkpoint 丢失。

### 为什么内存锁仍不够

Demo 用每个内部 thread 一把 asyncio.Lock，把“检查 pending → 恢复 checkpoint”串行化；但它只能保护一个进程。多 worker、多实例或进程重启后仍会竞态，生产要用持久审批记录和数据库 CAS 原子完成 `PENDING → APPROVED | REJECTED`，写操作再以事务和唯一约束兜底。

### 为什么还要做 checkpoint 对账

图、业务数据库和 HTTP 幂等响应不是天然的一笔事务。预约可能已成功，但 handler 可能在缓存响应时异常或被取消。Demo 会先 claim 审批决定，并演示**同一进程内**的对账：若 interrupt 仍在就按同一决定恢复，若后续节点失败就从最新 checkpoint 继续，若图已完成就重建最终响应。它的三份存储都是内存实现，真实进程退出后无法恢复；生产必须持久化 checkpointer、请求状态机和业务结果，再使用 lease/reconciliation job 按 operation_id 对账。`asyncio.shield()` 只能缩小取消窗口，不能消除进程崩溃窗口。

---

# 第五部分：系统设计与项目深挖

## 23. 一次请求的正确责任边界

| 层 | 负责 | 不负责 |
|---|---|---|
| FastAPI | HTTP、认证、Schema、错误码、流式连接 | 让模型决定权限 |
| Agent/Graph | 理解、路由、规划、上下文 | 直接绕过业务规则写库 |
| Service | 权限、状态机、幂等、事务 | 生成自然语言 |
| Repository/DB | 约束、查询、持久化 | 理解用户意图 |
| Worker | 长任务、重试、异步通知 | 假装同步请求已完成 |

一句高质量回答：

> LLM 负责提出结构化意图或工具调用建议，Pydantic 检查形状，Service 再做身份、租户、业务状态和幂等校验，数据库约束处理最终竞态。

## 24. 超时、重试、降级

不要只说“加重试”。先分错误：

| 错误 | 是否重试 | 典型处理 |
|---|---|---|
| 连接抖动、部分 429/5xx | 有上限重试 | 指数退避 + jitter |
| 参数/Schema 错误 | 不重试 | 修正输入 |
| 401/403 | 不重试 | 重新认证/授权 |
| 业务冲突 409 | 通常不原样重试 | 换时段或读取原结果 |
| 模型输出不合法 | 可有限修复 | 结构化输出、一次纠错、fallback |
| 写操作结果未知 | 先按幂等键查询 | 不能盲目重复执行 |

端到端 deadline 可以拆成：

~~~text
总预算 12s
├─ 路由/安全 0.3s
├─ 检索 1.5s
├─ 重排 1.0s
├─ 模型首 token 4.0s
├─ 生成 4.0s
└─ 余量 1.2s
~~~

每层都用 12 秒超时会导致总时延失控。上游 10 秒时，重试三次也不可能满足 12 秒端到端预算。

## 25. RAG 只记住这条面试主线

~~~text
解析/清洗
→ 按结构分块
→ Embedding + 元数据 + ACL 入库
→ Query Rewrite
→ BM25 + Dense 混合召回
→ RRF 融合
→ Cross-Encoder 精排
→ 上下文拼装
→ 有引用生成 / 无证据拒答
~~~

必须能解释：

- chunk 不是越小越好：小块利于精确召回，但上下文不足；大块语义完整但噪声多；
- Dense 擅长语义，BM25 擅长专有名词、编号和精确词；
- RRF 基于排名融合，避免直接比较不同检索器的原始分数；
- reranker 成本高，只对候选集做精排；
- 文档 ACL 必须在检索层执行，不能等生成后再删答案；
- RAG 评测要拆检索与生成，不能只看最终“感觉不错”。

更完整内容见 [项目基础课的 RAG 部分](./beginner_tutorial.md)。

## 26. 评测与可观测性

### 离线评测

| 层 | 重点指标 |
|---|---|
| 路由 | intent accuracy、宏平均 F1、拒识率 |
| 检索 | Recall@K、MRR、nDCG、ACL 泄漏数 |
| 生成 | correctness、faithfulness、citation precision、拒答率 |
| Tool | 参数合法率、执行成功率、副作用重复数 |
| Agent | 任务成功率、轨迹正确率、平均步数、人工接管率 |
| 系统 | TTFT、P50/P95/P99、错误率、token、单请求成本 |

### 在线观测

一条 Trace 至少串起：

~~~text
request_id
  └─ graph_run_id / thread_id
      ├─ route span
      ├─ retrieval span：query、doc IDs、index version
      ├─ model span：provider、model、prompt version、tokens
      └─ tool span：tool、result、error、idempotency key
~~~

注意日志脱敏。不要记录原始身份证、电话、密钥、完整私有文档和模型供应商 Token。

### LLM-as-Judge 的边界

- 先用人工标注小集校准 Judge；
- Judge Prompt、模型和版本固定；
- 对关键安全和业务约束使用确定性断言；
- 报告置信区间或至少给出样本数；
- 模型分数不能替代线上任务成功率。

## 27. 安全回答的四层

1. 输入层：认证、租户隔离、限流、大小限制；
2. 上下文层：检索 ACL、数据最小化、Prompt Injection 标记；
3. 工具层：白名单、Schema、最小权限、二次授权、HITL；
4. 输出层：敏感信息检查、引用、审计、可撤销动作。

模型、检索文档、工具返回和长期记忆都视为不可信数据。System Prompt 是说明，不是权限系统。

---

# 第六部分：高频面试问答

## 28. Python

### Q1：await 就是并发吗？

不是。await 只表示当前协程在此等待并让出控制权；连续 await 通常仍是串行。独立任务需用 TaskGroup/gather 调度，并设置超时和并发上限。

### Q2：TaskGroup 和 gather 有什么区别？

gather 适合收集多个结果；默认一个任务抛错时，其他任务不会因此全部自动取消。TaskGroup 提供结构化并发，一个子任务失败会取消兄弟任务并统一传播异常，更适合共同组成一次业务操作的任务。

### Q3：async endpoint 里调用 requests.get 会怎样？

它会阻塞事件循环，拖慢同 worker 的其他请求。优先使用异步 HTTP 客户端；无法替换时用普通 def 入口或 asyncio.to_thread 处理短阻塞 I/O。

### Q4：线程和进程怎么选？

线程适合阻塞 I/O；默认 CPython 下纯 Python CPU 密集任务用进程获得多核并行。模型推理通常放独立推理服务，而不是塞进 Web worker。

## 29. FastAPI

### Q5：FastAPI 为什么适合 AI 应用？

AI 网关多数时间在等待模型、向量库和数据库，ASGI + async 能提高 I/O 并发；Pydantic 约束输入输出，DI 管理鉴权和客户端，SSE 适合 token 流。但它不自动解决推理吞吐、重试、限流和状态持久化。

### Q6：Depends 的工程价值是什么？

它显式声明鉴权、session、配置、客户端和请求上下文，可形成子依赖并被测试替换。yield 依赖还能保证资源清理。

### Q7：为何不能在 Pydantic validator 中查询数据库？

validator 应做纯结构与字段不变量校验。外部 I/O 会让模型校验难测试、难控制事务且通常是同步接口；存在性、唯一性和权限应在异步 Service 中检查，并由数据库约束兜底。

### Q8：BackgroundTasks 能跑文档向量化吗？

小 Demo 可以，生产不宜。它依附当前应用进程，没有持久化、跨机器调度和可靠重试保证；长任务应进入外部队列，并返回 202 + job_id。

### Q9：多 worker 为什么不能用内存会话？

worker 是独立进程，内存不共享；请求可能落到另一个 worker。checkpoint、限流、任务状态和幂等结果应放共享存储。

### Q10：SSE 和 WebSocket 如何选？

单向 token/进度推送优先 SSE；需要持续双向低延迟交互时用 WebSocket。两者都要处理断连、取消、背压和代理配置。

## 30. LangGraph

### Q11：LangGraph 和 LangChain 的区别？

LangChain 偏模型、工具、检索和高层 Agent；LangGraph 偏底层有状态编排、循环、checkpoint、HITL 和恢复。固定链路无需为了框架而用图。

### Q12：Node 为什么只返回局部 dict？

节点返回的是 State update，运行时按字段的 Reducer 合并；无需复制整个 State。这样并发与状态变更语义更清楚。

### Q13：Reducer 有什么用？

它定义旧值与新更新如何合并。默认覆盖；累计消息或并行结果时需定义合适 Reducer，否则可能发生并发更新冲突。

### Q14：条件边、Command、Send 怎么区分？

只选路用条件边；同时更新状态和 goto 用 Command；运行时动态 fan-out 多个不同输入任务用 Send。

### Q15：thread_id 是 user_id 吗？

不是。thread_id 标识一条 checkpoint 历史，一个用户可以有多条 thread。长期记忆通常按 user_id/tenant_id namespace 存 Store。

### Q16：Checkpointer 和 Store 的区别？

Checkpointer 保存单 thread 的图状态和恢复点；Store 保存跨 thread 的长期应用数据，如偏好或事实。

### Q17：为什么 interrupt 会造成重复副作用？

恢复时节点从开头重跑。interrupt 之前的写库、扣款或发信可能再次执行，所以要移到 interrupt 后的独立节点，并使用幂等键和数据库约束。

### Q18：生产为什么不用 InMemorySaver？

重启即丢失，多进程/多副本不共享，也无法可靠恢复。生产需要持久化 saver，并设计备份、连接池和租户隔离。

### Q19：为什么不默认上多 Agent？

多 Agent 增加 token、延迟、循环风险和调试难度。只有在上下文、权限、职责、模型专长或并行分工确实需要隔离时才值得。

### Q20：如何防止 Agent 无限循环？

使用最大步数/recursion limit、状态不变量、重复动作检测、工具预算、端到端 deadline 和人工接管；同时记录轨迹，找到循环来源，而不是只靠 Prompt 说“不要循环”。

---

# 第七部分：面试练习路线

## 31. 三道必须能现场写出的题

### 题一：受控并发调用

输入一组 query，最多并发 5 个请求，总超时 10 秒，任何失败都能正确取消和清理。考 asyncio、Semaphore、TaskGroup、timeout。

### 题二：FastAPI 接口

写 POST /chat：

- Pydantic v2 请求/响应；
- extra forbid；
- 当前用户依赖；
- 异步 LLM 客户端；
- 504 超时映射；
- 测试时替换 fake client。

### 题三：LangGraph 审批工作流

~~~text
START → classify
  ├─ faq → retrieve → answer → END
  └─ booking → draft → interrupt
                         ├─ approve → idempotent execute → END
                         └─ reject → END
~~~

必须解释 checkpoint、thread_id、resume 重跑语义和幂等。

基础算法仍可能出现。至少练熟：

- 哈希：两数之和、去重、频次；
- 滑动窗口：无重复字符最长子串；
- 堆：Top-K；
- 链表：反转、判环；
- 图：BFS/DFS、拓扑排序；
- LRU Cache。

## 32. 七天冲刺安排

| 天 | 学习与输出 |
|---:|---|
| 1 | Python async、TaskGroup、timeout；手写受控并发 |
| 2 | Pydantic、FastAPI 路由/Depends/lifespan；写 /chat |
| 3 | SSE、错误码、测试；模拟超时、429、坏 JSON |
| 4 | LangGraph State/Reducer/路由；默写最小图 |
| 5 | checkpoint、thread_id、interrupt；跑通配套 Demo |
| 6 | RAG、评测、Tracing、安全；画一次完整架构 |
| 7 | 20 道问答录音 + 3 道现场题 + 项目五分钟陈述 |

## 33. 项目陈述模板

不要从“我用了 LangGraph”开始。用五段式：

1. 问题：什么用户、什么流程、原方案哪里失败；
2. 约束：数据、延迟、成本、安全和团队规模；
3. 方案：为什么选 workflow/Agent、FastAPI、RAG、持久化；
4. 失败与权衡：超时、幻觉、重复工具调用、并发冲突如何处理；
5. 证据：测试集、任务成功率、P95、成本、真实日志和还未验证的边界。

示例口述：

> 我做的是企业售后咨询和上门预约。咨询需要 RAG，预约有真实副作用，所以没有让模型直接写库。我用 FastAPI 承接认证与接口契约，用 LangGraph 显式表达路由、状态和人工确认，预约执行放在确认之后，并以业务操作 ID 做幂等。离线按路由、检索、工具和端到端任务分别评测，线上记录 TTFT、P95、工具错误率、token 和人工接管率。当前 Demo 的 checkpoint 和预约存储仍是内存实现，生产会迁移到 Postgres，并补租户 ACL、限流和持久任务队列。

## 34. 最后自检

如果下面每题都能在两分钟内说明并画出关键代码，核心准备就够用了：

- 连续 await 为什么不是并发？
- async def 里为什么不能直接用 requests？
- Pydantic 校验和业务校验的边界？
- FastAPI yield 依赖和 lifespan 有何不同？
- SSE 为什么适合 token 流？
- State 和数据库业务状态有什么区别？
- Reducer 何时必须存在？
- Command、Send、conditional edge 如何选择？
- thread_id、user_id、checkpoint、Store 如何区分？
- interrupt 恢复为何要求副作用幂等？
- Agent 卡循环、工具超时、模型无效 JSON 怎么降级？
- 如何证明 RAG/Agent 真的变好了？

---

# 参考资料

## 官方技术资料

- [Python asyncio：Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html)
- [Python typing：TypedDict、Protocol、Annotated](https://docs.python.org/3/library/typing.html)
- [FastAPI async/await](https://fastapi.tiangolo.com/async/)
- [FastAPI Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [FastAPI lifespan](https://fastapi.tiangolo.com/advanced/events/)
- [FastAPI SSE](https://fastapi.tiangolo.com/tutorial/server-sent-events/)
- [FastAPI Testing](https://fastapi.tiangolo.com/tutorial/testing/)
- [Pydantic Models](https://docs.pydantic.dev/latest/concepts/models/)
- [Pydantic Validators](https://docs.pydantic.dev/latest/concepts/validators/)
- [LangGraph Graph API](https://docs.langchain.com/oss/python/langgraph/graph-api)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [LangGraph Event streaming](https://docs.langchain.com/oss/python/langgraph/event-streaming)
- [LangGraph Subgraphs](https://docs.langchain.com/oss/python/langgraph/use-subgraphs)
- [LangChain Agents 与 Tools](https://docs.langchain.com/oss/python/langchain/agents)

## 面经与岗位信号

- [2026 春招 40 多场面试复盘](https://ac.nowcoder.com/discuss/1626144?channel=-1&source_id=0&type=0)
- [2025 大模型应用开发面经](https://www.nowcoder.com/feed/main/detail/129eaa1c20444651ac3b932e200d3da4?sourceSSR=dynamic)
- [2026 快手 AI 应用开发面经整理](https://www.nowcoder.com/discuss/882573284426932224?sourceSSR=enterprise)
- [2026 DataInfra Agent 后端一面](https://www.nowcoder.com/feed/main/detail/a2f2140fb6ae405995c68b4d4b04d0f8)
- [2025 LangGraph Agentic RAG 项目追问复盘](https://www.reddit.com/r/LangChain/comments/1k662xc/got_grilled_in_an_ml_interview_today_for_my/)
- [当前 EPAM Agentic and RAG Systems 岗位](https://careers.epam.com/en/vacancy/senior-ai-engineer-agentic-and-rag-systems-blty5mp8mok8tyd6a36_en)
- [当前埃森哲上海 AI Agent 岗位](https://www.accenture.com/gb-en/careers/jobdetails?id=14477302_en)

> 面经属于个人样本，可能有记忆偏差、幸存者偏差或转载误差；本教程只用它们判断复习优先级。技术语义以对应版本官方文档为准。
