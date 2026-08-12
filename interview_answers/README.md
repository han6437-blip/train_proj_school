# 企业售后智能客服与预约 Agent 系统：面试回答总册

本目录包含上一轮题库中全部问题的参考回答。回答统一采用“先给结论—解释实现—说明异常与权衡—回到项目结果”的结构，适合整理成 3～5 分钟口述。

如果你是初学者，请先阅读[项目基础课：从后端、幂等与Outbox/Inbox到Agent、RAG和微调](./beginner_tutorial.md)，并运行其中两个最小示例，再进入题库。

## 使用前必须确认的真实信息

以下内容来自已经给出的项目材料，可以在回答中保持一致：

- 业务场景：企业售后咨询、故障排查、工程师匹配、上门预约和用户偏好沉淀。
- Agent：主管 Agent、咨询 Agent、预约 Agent、用户行为分析 Agent；对外概括为 3 类专业 Agent，由主管 Agent 编排。
- 工程规模：Web/API/Agents/Services/DB 五层、16 个 API、30 个场景测试。
- RAG：BM25 + Dense 双路召回、RRF 融合、Cross-Encoder 精排，精排失败回退 RRF。
- 记忆：最近 10 轮、窗口使用率 60% 触发摘要、召回 Top-5；长期存储咨询摘要、预约事件和偏好。
- 模型评测：材料记录51条冻结集及Qwen3 0.6B/1.7B/4B基线；1.7B相比4B的组合任务正确性低2.9个百分点，平均时延低34.4%，吞吐高32.9%。组合公式、逐项结果和运行日志仍需补齐才能独立复现。
- 训练数据：500 条 SFT、150 组 DPO，训练集与冻结评估集隔离。
- 本地部署路线：Qwen3-1.7B、LoRA/QLoRA、SFT、DPO、GGUF、Q4_K_M、llama.cpp OpenAI 兼容服务。

“材料中出现”不等于“已经生产落地”。凡是缺少对应代码、测试、日志或报告的RAG、长期记忆、AutoDream、训练部署和可靠消息链路，只能表述为设计方案或教学验证；证据分级方法见初学者基础课。

以下信息没有真实记录，不能由公开资料代填：

- 实际 CPU/GPU、内存、操作系统和训练时长。
- Base/SFT/DPO 的最终正确率、协议遵循率和有效通过数。
- Q4_K_M 文件大小、TTFT、P95 和峰值内存。
- RAG 各阶段 Recall@K、MRR、Faithfulness 的最终实测提升。

文中遇到这些问题时会回答评测方法和工程判断，但不会编造个人实验结果。正式面试前，应将训练日志和压测报告中的真实数据补进去。

## 全局技术口径

1. **编排方式**：确定性安全规则在前，简单单意图可以直接路由；需要跨领域、多步骤协作时进入主管 Agent。专业 Agent 作为工具被主管调用，最终副作用由业务服务执行，而不是由模型直接操作数据库。
2. **状态边界**：会话内状态按 `tenant_id + user_id + session_id` 隔离；长期记忆至少按 `tenant_id + user_id` 隔离；预约记录和工程师排班以数据库为事实源；摘要和偏好只是辅助上下文，不能覆盖业务事实。
3. **数据库定位**：当前个人项目用 SQLite 验证流程。通过短事务、WAL、忙等待和数据库约束降低冲突，但不宣称支持高写并发；生产方案迁移 PostgreSQL，预约时间段使用范围类型和排斥约束。
4. **异步模型**：FastAPI 负责 I/O 并发；CPU 密集的重排和本地推理不直接堵塞事件循环，而是独立模型服务或受控线程/进程池。每个并发任务使用自己的 `AsyncSession`。
5. **一致性语义**：不承诺跨 HTTP、模型、数据库和通知系统的严格 Exactly Once；通过幂等键、数据库事务、Outbox和Inbox实现“预约效果至多一次、事件至少一次、本地消费效果去重”。外部短信是否重复还取决于供应商幂等能力。
6. **模型职责**：远端大模型负责复杂咨询和规划；本地 Qwen3-1.7B 负责边界清晰的槽位提取与工具调用决策。LLM 负责理解，Pydantic/业务代码负责 Schema、权限、状态和事务校验。
7. **评测口径**：若任务正确性按动作、关键字段和状态继承做连续组合评分，可以出现非整数百分点；但必须提供公式和逐项结果。严格有效通过率另以 `通过数/51` 统计。速度对比必须同一硬件、模板、生成参数、预热和并发条件。
8. **安全口径**：检索文档、工具输出和长期记忆均视为不可信数据；模型只提出工具调用建议，工具执行层重新做身份、租户、参数、风险、幂等和确认校验。

## 文档索引

- [项目完整请求流程与代码映射](./project_request_flow.md)
- [Kafka、RabbitMQ、Redis Streams、Celery：AI 应用消息队列面试重点](./message_queue_interview_tutorial.md)
- [OpenClaw、Claude Code、Hermes Agent、Pi：Agent 项目高频面试重点](./open_source_agent_projects_interview_tutorial.md)
- [Milvus + Elasticsearch + Chroma：AI 应用开发向量数据库面试重点教程](./vector_database_interview_tutorial.md)
- [Python + FastAPI + LangGraph：AI 应用开发面试重点教程](./python_fastapi_langgraph_interview_tutorial.md)
- [FastAPI + LangGraph 面试教学 Demo](./python_fastapi_langgraph_demo/)
- [初学者项目基础课](./beginner_tutorial.md)
- [两个可运行教学示例](./beginner_examples/)
- [一线 Agent 开发面试官：20 题](./interviewer_1_agent.md)
- [系统架构师：18 题](./interviewer_2_architect.md)
- [稳定性与并发安全：21 题](./interviewer_3_reliability.md)
- [现场题：10 题](./onsite_tasks.md)
- [前后一致性校验](./consistency_check.md)

## 主要检索依据

- [Milvus：索引原理与选型](https://milvus.io/docs/index-explained.md)
- [Milvus：一致性级别](https://milvus.io/docs/consistency.md)
- [Elasticsearch：dense_vector](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector)
- [Elasticsearch：kNN Query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-knn-query)
- [Chroma：Collection 配置](https://docs.trychroma.com/docs/collections/configure)
- [Chroma：Query 与 Get](https://docs.trychroma.com/docs/querying-collections/query-and-get)
- [LangChain：Supervisor 与 Router](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents)
- [LangChain：Agent 轨迹评测](https://docs.langchain.com/oss/python/langchain/test/evals)
- [Sentence Transformers：Retrieve & Re-Rank](https://www.sbert.net/examples/sentence_transformer/applications/retrieve_rerank/README.html)
- [Elasticsearch：RRF 公式](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion)
- [LangChain：短期与长期记忆](https://docs.langchain.com/oss/python/concepts/memory)
- [Qwen3：Thinking、Non-Thinking 与 Tool Calling](https://qwenlm.github.io/blog/qwen3/)
- [LLaMA-Factory：Qwen3 SFT、DPO、QLoRA 示例](https://github.com/hiyouga/LlamaFactory/blob/main/examples/README.md)
- [llama.cpp Server：并发槽位、结构化输出和 OpenAI 兼容接口](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)
- [SQLAlchemy：Session/AsyncSession 并发模型](https://docs.sqlalchemy.org/en/20/orm/session_basics.html)
- [SQLite：WAL 锁模型](https://www.sqlite.org/walformat.html)
- [PostgreSQL：范围类型与排斥约束](https://www.postgresql.org/docs/17/rangetypes.html)
- [AWS：幂等 API 与安全重试](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- [OWASP：Agent Excessive Agency](https://genai.owasp.org/llmrisk/llm062025-excessive-agency/)
- [MCP：工具安全与用户授权](https://modelcontextprotocol.io/specification/2025-03-26/index)
