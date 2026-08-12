# AI 应用开发：消息队列面试重点

> 只保留高频面试内容。主线是：**长任务异步化，通常采用至少一次投递；消费者幂等；失败有限重试；用背压和并发限制保护模型/GPU。**

## 1. AI 应用中的典型场景

- 文档解析、切片、Embedding、向量入库；
- 图片/视频生成、批量推理、离线评测；
- 模型调用审计、埋点日志、数据同步；
- 通知、Webhook、异步索引刷新。

关键边界：**不是所有模型调用都应进外部队列。** 实时聊天和 Token 流通常走同步流式链路；可接受 `202 + job_id` 的长任务才更适合任务队列。FastAPI `BackgroundTasks` 适合当前进程内的短小、非关键任务，不适合必须持久化、重试或跨机器执行的工作。

## 2. 常见方案怎么选

| 方案 | 优先使用场景 | 高频关键词与取舍 |
|---|---|---|
| **RabbitMQ** | 异步推理、通知、离散任务分发 | Exchange、Routing Key、手动 Ack、Prefetch、DLX；任务控制灵活，传统队列模式不以长期事件回放为强项 |
| **Kafka** | 审计日志、数据管道、RAG 批量索引、多个独立下游 | Topic、Partition、Consumer Group、Offset、Lag、Replay；吞吐与回放强，顺序只在分区日志内 |
| **Redis Streams** | 已有 Redis，且现有架构能满足吞吐与持久化目标 | Consumer Group、PEL、`XACK`、`XAUTOCLAIM`；接入轻，但要管理持久化、裁剪、Pending 和内存 |
| **Celery** | Python Worker、重试、定时任务与工作流 | **它是任务队列框架，不是 Broker**；常搭配 RabbitMQ 或 Redis，Result Backend 也要单独考虑 |

RocketMQ 在国内企业栈较常见。JD 明确出现时，再重点准备按业务 Key 顺序、延迟消息和[事务消息](https://rocketmq.apache.org/docs/featureBehavior/04transactionmessage/)；事务消息也不等于消费端 Exactly Once。

## 3. 高频面试题与精简回答

### Q1：为什么 AI 长任务要用消息队列？

为了让 API 与耗时任务解耦、削峰、失败恢复和横向扩容。标准链路是：API 创建任务并返回稳定 `job_id`，Worker 异步处理，结果写数据库/对象存储，客户端轮询、SSE 或 Webhook 获取状态。队列只能缓冲流量，不能增加 GPU 或模型 API 的真实容量。

### Q2：怎样尽量避免消息丢失和重复？

无法承诺绝对不丢、不重复，常用方案是 **At-least-once + 业务幂等**：

1. 生产者使用发送确认，Broker 配置合适的持久化与副本；
2. RabbitMQ 使用手动 Ack，Kafka 在处理完成后提交 Offset；Celery 若需要执行后确认，要显式评估 [late ack](https://docs.celeryq.dev/en/stable/userguide/tasks.html) 配置，不能假设默认如此；
3. 消费者用 `task_id/event_id`、唯一约束或条件更新原子领取任务，而不是“先查再执行”；
4. 结果持久化后再确认；已有成功结果直接复用，外部模型支持幂等键时传递稳定幂等键。

只重试超时、429、临时 5xx 等暂时性错误，并使用共享重试预算、指数退避、Jitter 和最大次数。永久错误或超限任务进入死信队列/死信主题或补偿表，避免立即重入形成毒消息循环。

### Q3：任务已提交，但 `202` 响应丢失怎么办？

客户端用原 `Idempotency-Key` 重试。服务端在同一事务中写“幂等记录 + Job + Outbox”，相同 Key 与相同请求应返回同一个 `job_id`；同一 Key 对应不同请求则报冲突。不能用“客户端是否收到 202”判断任务是否创建成功。

### Q4：数据库写入和发消息怎样保持一致？Exactly Once 能跨系统吗？

用 Transactional Outbox 把业务记录与待发布事件写进同一本地事务，再由发布器重试发送。Outbox 仍可能重复发布，所以消费侧要用 Inbox 或业务唯一约束幂等。

Kafka 的事务能力只覆盖明确的 Kafka 读写边界，不能自动覆盖 PostgreSQL、向量库、短信或模型 API。跨系统通常采用“至少一次投递 + 幂等 + 对账/补偿”，不要笼统承诺端到端 Exactly Once。

### Q5：长时间推理任务有什么特殊风险？

不同中间件没有统一的“续租”机制：Kafka 要关注 `max.poll.interval.ms` 与 Rebalance；RabbitMQ 要关注手动 Ack 和确认超时；Redis Streams 通过 PEL 与 `XAUTOCLAIM` 接管失联任务；Celery 的行为取决于 Broker、可见性超时和 Ack 配置。

长任务还应支持拆分、checkpoint、超时取消和幂等恢复。重复推理会产生费用且结果可能变化，因此同一任务重试以 `task_id` 幂等；跨任务缓存则必须同时考虑租户、输入、模型版本和完整生成参数。

### Q6：Embedding 任务大量积压怎么办？

先看 Consumer Lag/队列深度、**最老消息等待时间**、处理耗时、失败/重试率和 GPU 利用率，判断瓶颈在消费端、模型 API 还是向量库。然后再选择批量 Embedding、增加消费者/GPU、入口背压、租户配额和高低优先级队列。

Kafka 的消费并行度受 Partition 数限制；只有确认分区不足且下游仍有容量时才评估扩分区，并考虑 Key 路由与局部顺序变化。RabbitMQ 用 Prefetch 限制未确认任务数；AI Worker 的并发还必须结合显存、批大小和模型限流压测。

### Q7：如何保证顺序？

通常只保证同一文档、会话或业务实体内的局部顺序，不追求全局顺序。Kafka 用稳定业务 Key 路由到同一 Partition，但消费者内部并行、重试或扩分区仍可能打乱完成顺序。RabbitMQ 若要求严格处理顺序，应使用单活消费者或明确的单线程串行处理并控制 Prefetch；重投时仍需业务序号或版本校验。

### Q8：消息体应该放什么？

只放任务 ID、对象地址、租户、文档版本、模型参数和 `schema_version`；PDF、图片、长 Prompt 和结果放对象存储或数据库。`tenant_id` 应由认证后的服务端写入，Worker 重新校验 Schema、租户 ACL、对象版本和允许的模型，不直接信任用户或 LLM 提供的 URI。

向量入库使用稳定 Chunk ID + `upsert`，旧版本不得覆盖新版本；删除事件使用 tombstone，并定期对账事实源与向量索引。

## 30 秒口述模板

> AI 应用里我会把文档向量化、异步生成、离线评测和通知放进队列，但实时聊天通常保持流式链路。任务代理与逐条控制偏 RabbitMQ，Python Worker 可用 Celery；高吞吐、回放和多下游偏 Kafka；已有 Redis 且可靠性目标匹配时可用 Redis Streams。可靠性采用至少一次投递、业务幂等、结果落库后确认、有限重试和死信兜底；双写用 Outbox/Inbox。GPU 场景再补并发限制、批处理、租户配额，并重点监控最老消息等待时间、Lag、重试率和 GPU 利用率。

## 精简参考

- [本项目基础课：消息语义、Outbox 与 Inbox](./beginner_tutorial.md#9-消息队列和交付语义)
- [FastAPI：Background Tasks 的适用边界](https://fastapi.tiangolo.com/tutorial/background-tasks/)
- [Apache Kafka：分区、顺序与事件保留](https://kafka.apache.org/documentation/)
- [RabbitMQ：Ack、Confirm 与 Prefetch](https://www.rabbitmq.com/docs/confirms)
- [Redis Streams：Consumer Group、PEL 与 XACK](https://redis.io/docs/latest/develop/data-types/streams/)
- [Celery：Brokers 与 Result Backends](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/index.html)
- [AWS：Transactional Outbox](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)
