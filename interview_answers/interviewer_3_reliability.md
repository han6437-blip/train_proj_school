# 稳定性、并发与安全：21 道高频面试题

> 下面的回答按面试口述整理。每题先给结论，再说明机制、失败边界和验证方法。

## 1. FastAPI 使用 `async def` 就一定能提高并发吗？

不一定。`async def`主要提升的是 I/O 等待期间的并发利用率，例如等待数据库、HTTP 或模型服务；它不会让 CPU 密集任务自动并行。如果我在协程里直接做本地模型推理、Cross-Encoder 重排或大文档解析，事件循环仍会被阻塞。

我的做法是按负载隔离：原生异步 I/O 直接 `await`；不可异步化的短阻塞调用放进有界线程池；CPU/GPU 密集任务放到独立进程或模型服务，并在入口设置并发信号量、有界队列和绝对 Deadline。`asyncio.to_thread()`只能防止主事件循环被阻塞，不能增加 CPU 或显存容量。验证时我会观察事件循环延迟、队列等待、CPU/显存、吞吐、P95/P99 和超时率，而不是只看代码里有没有 `async`。

**追问：`def` 路由会怎样？** FastAPI 会把同步路径函数和同步依赖放到线程池执行，但普通同步工具函数若在 `async def` 中被直接调用，仍会阻塞事件循环。

## 2. SQLAlchemy `AsyncSession` 能被多个 Agent 并发共享吗？

不能。`AsyncSession`是有状态的事务对象，会跟踪连接、事务、Flush 和 ORM identity map。多个协程同时操作同一个 Session，SQL 顺序、事务状态和异常回滚都会互相干扰。SQLAlchemy 官方建议的并发模型就是“每个 asyncio task 一个 `AsyncSession`”。

我会用 `async_sessionmaker`，让每个并发 Agent 在自己的 `async with` 中创建并关闭 Session。并行只读任务可以各用一个 Session；涉及同一业务原子写入时，我不会让多个 Agent 各自提交，而是把命令交给一个业务 Service，在一个清晰的事务边界里完成校验、写入、幂等记录和 Outbox。若遇到可重试的数据库错误，也要新建事务并重跑完整业务单元，不能从失败的一半继续。

## 3. SQLite 开启 WAL 后是否支持多个并发写入？

WAL 改善的是“读者与写者可以并发”，不是“多个写者可以同时提交”。SQLite 官方文档明确说明，同一时刻仍只有一个 writer；配置 busy handler 或 `busy_timeout` 后，其他写事务会按策略等待，未配置时也可能立即返回 `SQLITE_BUSY`。

我会让每次操作使用独立连接，设置合理的 `busy_timeout`，用 `BEGIN IMMEDIATE`尽早获取写锁，并把事务保持得很短：事务内只做重叠校验、预约写入、幂等终态和 Outbox，不做模型或网络调用。对确认已回滚、幂等安全的 `SQLITE_BUSY`，按扩展错误码在总 Deadline 内有限重试；`SQLITE_LOCKED`更多提示同连接、shared-cache 或游标生命周期问题，默认先诊断而不是当拥塞重试。SQLite 适合单机和中低写并发；写竞争成为瓶颈时迁移 PostgreSQL，并由数据库约束守住业务不变量。

**追问：WAL 还要注意什么？** 长读事务可能阻碍 checkpoint 推进，因此还要监控 WAL 大小、checkpoint 延迟和长事务；版本选择要核对 SQLite 官方变更记录中的 WAL 相关修复，并在升级后重跑并发与崩溃恢复测试。

## 4. 请分析重复预约竞态，并说明最终防线

经典竞态是：请求 A 查询到工程师 14 点空闲，请求 B 也查询到空闲，然后两边都插入。如果“先查再写”不在同一并发控制边界里，两次都可能成功，所以可用性查询只能用于展示候选，不能作为最终事实。

我守住的不变量是：同一租户、同一工程师的两个有效预约区间不能重叠。SQLite 中可以用 `BEGIN IMMEDIATE`串行化短写事务，并在锁内复查；PostgreSQL 中我更倾向用 range type 加 GiST 排斥约束，例如对 `tenant_id`、`engineer_id`做等值约束，对 `during`做 `&&`重叠约束，并只覆盖有效状态。即使两个事务都曾读到空闲，数据库仍会拒绝冲突写入。应用捕获约束冲突后回滚，返回冲突并重新给候选，而不是原样重试过期方案。

分布式锁可以削峰，但不能替代数据库最终约束，因为锁可能过期、进程可能暂停，配置也可能失误。

## 5. 创建预约接口如何实现幂等？

我会让客户端为一次业务意图提供稳定的 `Idempotency-Key`，服务端再结合可信的租户/用户作用域生成内部 `operation_id`。首次请求保存 Key、规范化参数哈希、操作状态和结果；同 Key 同参数返回语义等价的原结果，同 Key 不同参数返回 409，避免旧 Key 被拿来表达新意图。

短事务可以在同一个数据库事务里完成：占用幂等 Key、复查时段、写 Booking、写 Outbox、记录成功结果。数据库上同时建立 `UNIQUE(scope, idempotency_key)` 和业务操作唯一约束，解决并发首次请求。长任务则使用持久化状态机，如 `PROCESSING / SUCCEEDED / FAILED_FINAL / FAILED_RETRYABLE`，配合 owner、lease、心跳和陈旧任务对账。

幂等记录至少要覆盖客户端重试、工作流恢复和对账窗口；到期后要明确拒绝旧 Key 或保留轻量墓碑，不能静默把迟到重放当成新请求。若用户确实要创建参数相同的新预约，需要新的业务意图和新 Key。关键验证包括：并发同 Key、同 Key 异参数、首次提交成功但响应丢失、记录过期后的重放。

## 6. 模型为什么可能重复调用同一个有副作用工具？

常见原因是模型没有正确消费上一次工具结果、工具已成功但响应丢失、图在副作用之后而 Checkpoint 之前崩溃，或者 HTTP 网关重放了整次请求。Prompt 和模型生成的 `tool_call_id`都不是可靠的防重边界。

我的做法是在用户确认业务意图时由服务端生成并持久化稳定的 `operation_id`，模型重试、工作流恢复和 HTTP 重放都复用它。工具执行层用 `operation_id + 规范化动作摘要`查结果：摘要相同就返回原 `booking_id`，摘要不同就拒绝。工具超时造成结果未知时，先按 `operation_id`查询；下游支持幂等时，也只能用同一个 Key、同一参数重放。

此外我会限制 Agent 最大步骤和同指纹工具调用次数，但这只是防止模型死循环，不能代替数据库唯一约束。

## 7. 哪些错误可以重试，哪些不能？

我按五个问题判断：失败是否暂时、操作是否有副作用、结果是否确定、调用是否幂等、总 Deadline 是否还有预算。

- 只读调用的 429、短暂 5xx、连接建立失败：遵守 `Retry-After`，在预算内有限重试。
- 已确认回滚的死锁、序列化失败、SQLite BUSY：新建 Session/事务，重跑完整事务并加 Jitter。
- 模型结构化输出不合法：最多做少量带校验错误的修复生成；仍失败就澄清或降级。
- 参数错误、权限拒绝、业务状态不允许、用户拒绝、时段冲突：不能原样重试。
- 副作用调用超时、COMMIT 回包丢失：结果是 `UNKNOWN`，先按幂等键查询和对账，不能换新 Key 再执行。

我只让最接近失败源、理解幂等语义的一层负责重试，避免 SDK、网关、Agent、Worker 层层叠加形成重试风暴。

## 8. 指数退避为什么需要 Jitter？

如果一批请求同时失败，又都在 1、2、4 秒后重试，它们仍会同步形成尖峰，这就是惊群。指数退避只拉长间隔，Jitter 才能打散重试时刻。

我常用 Full Jitter：第 `n` 次从 `0` 到 `min(cap, base × 2^n)`之间随机等待，同时遵守下游 `Retry-After`、最大尝试次数和端到端 Deadline。重试还要有全局预算和并发上限；熔断打开后停止常规请求，只允许少量探测。Jitter 只能缓解恢复流量，不能把权限错误变成可重试错误，也不能解决副作用结果未知。

## 9. 用户取消请求时，后端任务会自动取消吗？

不能假设会。客户端断开、ASGI 请求任务、线程池函数、远端 HTTP 调用和数据库 COMMIT 的取消语义都不同。用户不再等待，不等于预约被撤销；Python 的 `shield()`也只是在调用方取消时保护某个 awaitable，不是持久任务机制。

我的边界是：尚未产生副作用的只读调用可以协作式取消并释放资源；一旦业务命令被接受，就持久化 `operation_id`和状态，由短事务或可靠任务系统推进，不能依赖裸 `create_task()`。提交前若数据库确认回滚，可以判定未执行；提交结果不确定则标记 `UNKNOWN`，由状态查询或恢复任务对账。用户要撤销已创建的预约，必须另发一个经过鉴权、归属校验、确认和幂等保护的 `cancel_booking`命令。

## 10. 流式响应输出一半后异常怎么办？

核心原则是：数据库提交前绝不流式宣告“预约成功”。解释文本可以提前发，但业务结果只在事务成功提交后，通过结构化 `final_result`事件发送。每个事件携带 `operation_id`、事件类型和递增序号，例如 `content_delta`、`tool_status`、`final_result`、`error`；序号用于去重和发现缺口，真正恢复还需要持久事件缓冲配合 `Last-Event-ID` 重放，或者直接按 `operation_id` 查询数据库终态。

HTTP 响应头发出后不能再改成 500；连接仍在就发送结构化错误事件，连接已断就结束。客户端没收到 `final_result`只能判断结果未知，应按 `operation_id`查询终态。若事务已提交而最终事件丢失，重连或幂等重放仍返回同一个业务结果。高风险结论使用数据库事实生成的模板事件，避免模型提前自由表述成功状态。

## 11. 分布式租约过期，但旧任务仍在运行怎么办？

租约过期后，新 Worker 可能取得锁，而暂停过的旧 Worker 又恢复写入，这叫 stale holder。只有租约不足以阻止旧持有者，所以我会增加 fencing token。

获取租约时按业务资源原子递增 `run_epoch`，每次写偏好、聚合结果或 Checkpoint 都携带该 token。存储层使用条件更新，只接受当前 epoch，或者拒绝小于目标行 `last_fence`的 token。判断必须在存储层原子完成，不能让客户端基于旧快照比较。续租失败后 Worker 停止取新批次；即使旧任务后来恢复，其写入也会被存储层拒绝。事件再用稳定 `event_id`去重，业务更新与对应 Checkpoint 尽量同事务提交。

## 12. 两个会话同时更新同一用户偏好怎么办？

如果两边都读旧值再覆盖，会发生 Lost Update。我会把会话行为先追加成不可变事件，例如“本次选择周六”和“明确拒绝周末”，每条带稳定 `event_id`、可信服务端顺序和作用域，再由幂等消费者聚合长期画像。

聚合视图带 `version`，更新使用 `WHERE version = old_version`做乐观锁；CAS 失败就重读未消费事件并重算。计数累加通常可交换，但“最新显式纠正覆盖旧偏好”需要定义来源优先级、服务端顺序和适用范围。短期会话约束与长期偏好必须分开，偏好也永远不能覆盖排班和预约的数据库事实。

## 13. llama.cpp 的 `--parallel`、`--ctx-size` 与并发是什么关系？

`--parallel`控制 server slot 上限，也就是最多可并行承载的序列数；`--ctx-size`控制 server 的总上下文/KV Token 预算。固定非 unified KV 配置下，每个 slot 的有效上下文大致受 `ctx-size / parallel`约束；启用 unified KV 后可以动态共享，具体行为取决于构建版本。增加 slot 可以配合 continuous batching 提高总吞吐，但在总上下文固定时不必然增加 KV 内存，反而会缩小单请求可用上下文，并加剧算力竞争，TTFT 和尾延迟可能变差。

这两个参数的细节会随版本和 KV 配置变化，我不会把 `ctx-size / parallel`当成永久业务契约。我会固定模型、量化、构建版本、Chat Template、硬件、输入/输出长度，分别测试不同 slot 数，并以启动日志中的 `n_ctx`、每序列上下文和 `kv_unified` 等实际值为准，再记录吞吐、排队时间、TTFT、P50/P95/P99、超时和峰值显存。slot 也不等于 HTTP 线程数；入口并发超过推理容量时，额外请求必须受控排队或被拒绝。

## 14. 请求超过本地模型槽位后怎么办？

不能无限排队，因为队列会吞掉端到端 Deadline，还会把过载转成内存问题。我会在模型网关做准入控制：按租户限流，用信号量和有界优先队列，根据队列年龄、预计等待、剩余 Deadline、任务风险和模型健康决定接收、降级或拒绝；排队请求被客户端取消后及时移除。

低风险抽取可以切到经过同一 Schema 和质量门槛验证的备用模型；高风险写操作不能为了可用性绕过确认或换到未验证模型后自动执行。过载时明确返回 429 或 503，并给合理的 `Retry-After`。我会重点监控 slot 利用率、队列深度与最老年龄、TTFT、超时、取消、降级率和拒绝率，并用不同到达率做容量测试。

## 15. 如何防止超长 Prompt 拖垮服务？

限制要分层。网关限制请求体、附件、单用户速率和并发；上下文构建器用实际 Tokenizer 和 Chat Template 计数，限制消息、检索 Chunk、单个工具结果和总输入，并为输出预留预算；Agent 再限制最大步骤、工具次数和绝对 Deadline。

超限时不能简单删掉末尾，因为那通常包含最新指令。我会优先保留最新用户原文、结构化槽位及来源、未决问题、待确认动作和必要证据；较早对话做滚动摘要，大型工具结果外置后只传引用。摘要不是授权或事实源，执行前仍由业务 Service 重查数据库。压缩炸弹、重复字符和超大文档在进入模型前拒绝或转异步摄取。验证时覆盖长对话、极端中英文比例、工具巨量回包和摘要丢关键信息。

## 16. RAG 文档中藏有 Prompt Injection 怎么办？

我把用户输入、检索文档、工具结果和长期记忆都视为不可信数据，而不是指令。结构化区分 instructions 与 untrusted context 有帮助，但 Prompt 隔离不是安全边界；真正的防线在执行层。

检索阶段由服务端强制注入租户、ACL 和文档版本过滤，所有 BM25、Dense、缓存路径使用同一权限语义。Agent 只获得完成任务所需的最小工具；工具调用必须经过可信身份、权限、资源归属、严格 Schema、参数白名单、风险分级、确认和幂等校验。外部内容在摄取时做格式与可疑指令扫描，输出渲染防 HTML/Markdown 外带；高风险动作保留人工确认。

红队测试至少覆盖：文档要求泄露系统提示、伪造 Tool Result、跨租户召回、诱导取消预约、恶意内容写入长期记忆，以及编码/隐藏文本变体。目标不是宣称完全消灭 Prompt Injection，而是让模型即使受影响也拿不到越权能力。

## 17. 只用 System Prompt 禁止越权够不够？

不够。System Prompt 是行为引导，模型输出必须当作不可信建议。真正授权由确定性服务完成：从验证过的 JWT 或 Session claims 构造 ActorContext，拒绝模型提供的 `user_id`、`tenant_id`或“已确认”字段；工具从服务端注册表选择，参数按严格 Schema 校验并拒绝未知字段。

高风险确认要绑定动作名、规范化参数摘要、资源版本、用户与租户、有效期和一次性语义。执行前再检查资源归属、权限、限额、最新数据库事实、幂等键和事务约束。即使 Prompt Injection 生成了格式完全正确的 Tool Call，也不代表它获得了执行权。测试时我会做跨用户恢复、篡改确认参数、重放过期确认、并发批准/拒绝和 IDOR 负样本。

## 18. 如何防止跨用户、跨租户数据泄露？

我不会信任用户可控的 `thread_id`、请求头里的 `tenant_id`或模型生成的身份参数。服务端从已验证 Token 获取 actor、tenant 和 role，把它们放入内部操作与线程命名空间。仓储层强制按 `tenant_id + resource_id`查询并复核对象归属；高隔离场景再用 PostgreSQL RLS、独立 schema 或数据库做纵深防御。

向量检索、BM25、记忆、对象存储和缓存都在检索前按租户与 ACL 过滤，不能先全库召回再末端过滤。缓存键包含 tenant、权限范围、权限版本和数据版本，撤权后主动失效或使用短 TTL。日志、Trace、备份和评估导出也执行同样的隔离和脱敏。验收时创建两个租户的同名资源，对每条读取路径、缓存命中与批量接口做负向测试，跨租户返回必须为零。

## 19. 日志能保存完整 Prompt 和工具参数吗？

默认不保存。Prompt 和工具参数常含手机号、地址、设备信息、会话标识、Token 或第三方凭证。常规结构化日志只保留 `trace_id/operation_id`、事件类型、组件/模型版本、Token 数、耗时、状态、重试与降级原因、字段名，以及必要的脱敏值；需要稳定关联时使用带密钥的摘要，避免低熵个人信息被字典反查。

Authorization、Cookie、密码、连接串、密钥等在日志 SDK 入口统一删除；用户控制的日志字段还要防 CR/LF 注入。确需排障的原文进入与普通日志隔离的短期受控存储，采用字段加密、最小权限、访问审计、用途限制和到期删除。进入评估或训练前再做去标识化和抽检。验证不仅查业务日志，还要查异常栈、网关日志、Tracing 属性和第三方可观测平台，防止旁路泄露。

## 20. 能否保证预约 Exactly Once？

我不会笼统承诺跨 HTTP、数据库、Broker 和外部短信的端到端 Exactly Once，而是按边界定义语义：

1. Booking：稳定幂等键加数据库唯一/排斥约束，使同一业务尝试至多创建一份。
2. Booking + Outbox：同一个数据库事务提交，要么都有，要么都没有。
3. Outbox 发布：发布成功但标记前崩溃会重发，所以通常是至少一次；多 Worker 用行锁、lease 或 `SKIP LOCKED`认领。
4. 消费者：以稳定 `event_id`写 Inbox，并让 Inbox 和本地业务变化同事务，重复消息直接返回已处理。
5. 外部供应商：传稳定幂等键；超时进入 `UNKNOWN`，优先查询供应商状态或对账。若供应商既不支持幂等也不能查询状态，就无法同时避免重复和漏发，只能明确选择更符合业务风险的语义。

Transactional Outbox 解决的是数据库与消息发送的双写一致性，不会自动让所有外部副作用 Exactly Once。

## 21. 如何设计故障演练和不变量？

我先定义安全不变量，再定义可用性目标。前者包括：有效预约不重叠；未绑定有效确认不能写；同 Key 同语义最多一个 Booking；同 Key 异参数必须冲突；Booking 与 Outbox 同事务；跨租户数据不返回；通知失败不能回滚已提交预约。后者再用成功率、P95/P99、恢复时间、队列年龄和积压衡量，不能为了成功率放松权限或一致性。

故障注入围绕关键窗口设计：并发同 Key 与异 Payload；提交成功但响应丢失；工具成功后 Checkpoint 前崩溃；COMMIT 回包丢失；预约并发冲突；Outbox 发布后标记前崩溃；消费者本地事务失败；供应商成功但本地记账失败；模型 429、进程退出、过载和 Deadline 耗尽；恶意 RAG、伪造身份、缓存污染和敏感日志泄露。

演练验收不只看 HTTP 200，而要核对数据库终态、幂等表、Booking 数量、Outbox/Inbox、重试次数、审计与 Trace，并把发现的失败交错固化成自动化回归测试。可靠性的核心不是“系统从不失败”，而是失败发生时，不变量仍成立且状态可恢复、可对账。

## 参考资料

- [FastAPI：Concurrency and `async` / `await`](https://fastapi.tiangolo.com/async/)
- [SQLAlchemy：Session per thread, AsyncSession per task](https://docs.sqlalchemy.org/en/20/orm/session_basics.html#is-the-session-thread-safe-is-asyncsession-safe-to-share-in-concurrent-tasks)
- [SQLite：Write-Ahead Logging 与并发](https://sqlite.org/wal.html)
- [PostgreSQL：Range Types 与排斥约束](https://www.postgresql.org/docs/current/rangetypes.html#RANGETYPES-CONSTRAINT)
- [AWS Builders' Library：Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [AWS Builders' Library：Timeouts, retries, backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
- [AWS Prescriptive Guidance：Transactional Outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
- [llama.cpp Server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [OWASP：LLM Prompt Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)
- [OWASP：Multi-Tenant Security](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html)
- [OWASP：Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
