# 企业机房与办公电脑维修 Agent：完整请求流程与实现边界

本文把本目录中的架构说明和面试回答整理成一条可讲清、可核对的项目流程。原材料曾引用三个教学 Demo，但当前快照没有对应代码，因此这里只把它们作为待实现规格。本文同时回答三个问题：

1. 目标系统的一次完整业务链路如何推进；
2. Router、Supervisor、远端大模型和本地 Qwen3-1.7B 分别负责什么；
3. 当前快照有哪些可核对材料，哪些仍缺少代码、测试或运行日志。

## 0. 先说明证据边界

当前目录只有 Markdown 文档和两张架构图片，**没有可运行代码或测试**。原材料引用了三个教学闭环，但对应文件不在当前快照：

| 原材料引用的教学闭环 | 预期文件 | 当前状态 | 目标验证内容 |
|---|---|---|---|
| FastAPI + LangGraph | `python_fastapi_langgraph_demo/app.py` | 文件缺失，不能本地验证 | StateGraph、Checkpoint、Interrupt、请求重放与幂等 |
| Router + RAG + 工具门禁 | `beginner_examples/agent_rag_demo.py` | 文件缺失，不能本地验证 | `consult/booking/compound`路由、RAG替身、缺槽澄清与确认门禁 |
| 可靠预约事务 | `beginner_examples/reliable_booking_demo.py` | 文件缺失，不能本地验证 | SQLite事务、请求哈希、时段冲突、Outbox和Inbox去重 |

因此，面试时最准确的表述是：

> 项目材料定义了完整目标链路，但当前快照只能证明“文档已经设计”，不能证明 LangGraph 编排、审批恢复、HTTP 幂等、SQLite 事务或 Outbox/Inbox 已经由本地代码验证。补回实现、测试和运行日志后，才能把对应条目标记为已验证。

下文使用三个标记：

- **目标设计**：材料中有明确流程和接口口径，但当前目录没有实现证据；
- **待验证规格**：已经写清预期输入、输出和验收条件，仍需补回代码与测试；
- **生产演进**：用于多实例、高并发和高可用，不能说成个人项目当前已经部署。

## 1. 总体架构与职责

以下 Mermaid 是当前文档的权威架构图，明确表达 LangGraph 节点/子图跳转、异步用户行为分析和记忆旁路。`assets/`中的旧版 PNG 没有完整表达这一口径，不再作为架构依据。

> 图示边界：这是目标流程概览，不表示真实 Qwen、混合 RAG、工程师匹配和生产 Trace 已经在当前快照完成集成。

```mermaid
flowchart TD
    U["用户 / Web 客户端"] --> API["FastAPI 入口"]
    API --> GATE["身份与租户上下文、输入 Schema、限流与安全门禁"]
    GATE --> SESSION["会话状态、请求幂等、Trace 上下文"]
    SESSION --> ROUTER{"确定性 Router：consult / booking / compound"}

    ROUTER -->|"简单咨询"| CONSULT["咨询 Agent"]
    ROUTER -->|"简单预约"| BOOKING["预约 Agent"]
    ROUTER -->|"复合任务"| SUP["Supervisor 决策节点"]
    SUP -->|"Command: goto consult_agent"| CONSULT
    CONSULT --> RAG["元数据过滤 → BM25 + Dense → RRF → Cross-Encoder"]
    RAG --> REMOTE["远端大模型：带证据回答"]
    REMOTE --> CRETURN{"复合任务？"}
    CRETURN -->|"是：State 写回"| SUP
    CRETURN -->|"否"| ANSWER
    SUP -->|"Command: goto booking_agent"| BOOKING
    SUP -->|"Command: goto finalize"| ANSWER

    BOOKING --> LOCAL["本地 Qwen3-1.7B：槽位与动作建议"]
    LOCAL --> VALIDATE["JSON / Pydantic / 时间 / 状态校验"]
    VALIDATE -->|"缺槽或歧义"| CLARIFY["定向澄清 + Checkpoint"]
    CLARIFY -->|"同一 thread 恢复"| BOOKING
    VALIDATE -->|"槽位完整"| CANDIDATE["查询工程师技能与排班"]
    CANDIDATE --> CONFIRM{"用户确认候选方案？"}
    CONFIRM -->|"否或修改"| CLARIFY
    CONFIRM -->|"是"| SERVICE["预约业务 Service"]

    SERVICE --> TX["事务：幂等记录 + Booking + Outbox"]
    TX --> BRETURN{"复合任务？"}
    BRETURN -->|"是：State 写回"| SUP
    BRETURN -->|"否"| ANSWER["finalize：汇总诊断与预约结果"]
    TX --> WORKER["Outbox Worker"]
    WORKER --> CONSUMER["Inbox 去重 + 通知"]
    TX -.-> MEVENT["Memory Event：消息 / 工具 / 业务事件"]
    ANSWER -.-> MEVENT
    MEVENT --> MPOLICY["候选 + Policy + Reconcile"]
    TX -. "Outbox / 已完成事件" .-> BEHAVIOR["用户行为分析 Agent（异步）"]
    ANSWER -. "已完成交互事件" .-> BEHAVIOR
    BEHAVIOR -. "偏好候选" .-> MPOLICY
    MPOLICY --> MSTORE["版本化 Memory Store"]
    MSTORE -.检索 Top-5.-> SESSION
```

### 1.1 组件职责边界

| 组件 | 负责 | 不负责 |
|---|---|---|
| FastAPI/API 层 | HTTP 契约、认证上下文、租户识别、输入校验、依赖注入、错误映射 | 规划多步任务、直接拼接业务 SQL |
| 确定性 Router | 粗粒度识别咨询、预约、复合或未知意图；让简单请求走快速路径 | 长时间持有任务状态、执行预约 |
| Supervisor 决策节点 | 读取/更新复合任务 State，返回下一节点、澄清或结束；由条件边/`Command`执行图跳转 | 把专业 Agent 包装成 Tool、绕过 Service 修改数据库 |
| 咨询 Agent 节点/子图 | 查询改写、调用 RAG、生成有依据的故障建议并写回 State | 创建、修改或取消预约 |
| 预约 Agent 节点/子图 | 收集槽位、调用本地结构化模型和查询类业务工具、组织澄清和确认 | 把模型输出当成已授权业务命令 |
| 本地 Qwen3-1.7B | 槽位抽取、多轮继承与修正、缺槽/确认状态、`FINAL` 或 `TOOL_CALL` 建议 | 顶层意图路由、可靠计算日期、权限判断、数据库写入 |
| 远端大模型 | 开放式故障理解、基于证据生成、复杂跨领域规划 | 成为身份、权限或业务事实源 |
| 业务 Service | Schema、身份、租户、状态机、确认令牌、幂等、冲突和事务校验 | 相信模型传入的身份或确认事实 |
| 数据库 | 预约、排班、幂等结果等最终业务事实 | 依赖 Agent State 维持一致性 |
| Outbox/Inbox | 解决业务写入与事件发布的双写窗口，以及消费侧本地去重 | 承诺跨所有系统的绝对 Exactly Once |
| 用户行为分析 Agent | 异步消费已完成咨询、预约和反馈事件，提出带来源的偏好候选 | 进入在线 Supervisor 图、直接覆盖用户画像或记忆状态 |
| Memory Service | 捕获事件、生成原子候选、执行策略与冲突处理、版本化提交、同步派生索引 | 把模型推断当事实、覆盖预约数据库或越过工具授权 |

核心原则是：

> Router 判断“是什么任务”，Supervisor 决定“按什么顺序完成”，模型负责“理解和提出结构化建议”，业务代码决定“是否允许执行”，数据库保存“最终事实”。

## 2. “服务器无法启动并预约维修”的完整多轮链路

用户第一句话是：

> 机房一台服务器无法启动，提示找不到启动设备，帮我预约明天下午维修。

这句话缺少资产编号、机房具体位置、联系人等必要信息，因此完整业务链路通常不是一个 HTTP 请求完成，而是至少包含“初始请求—补充槽位—确认方案”三轮。

### 2.1 时序图

```mermaid
sequenceDiagram
    autonumber
    actor U as 用户
    participant API as FastAPI
    participant R as Router / Supervisor
    participant C as 咨询 Agent
    participant KB as RAG
    participant B as 预约 Agent / Qwen
    participant S as 预约 Service
    participant DB as 数据库
    participant W as Outbox Worker

    U->>API: 机房服务器无法启动，帮我预约明天下午维修
    API->>API: 校验输入，构造 tenant/user/session/request 上下文
    API->>R: 加载会话状态并路由
    R->>R: 标记 compound，建立“咨询 → 预约”计划
    R->>C: Command: goto consult_agent（State字段投影）
    C->>KB: 元数据过滤 + 两路召回 + 融合 + 精排
    KB-->>C: 证据 Chunk、版本与排名
    C-->>R: diagnosis_summary、risk_level、source_ids
    R->>B: Command: goto booking_agent（用户原话 + State + Schema）
    B-->>R: 日期/时段已识别，资产编号、机房位置和联系人缺失
    R-->>API: 保存 Checkpoint，返回诊断建议和定向澄清
    API-->>U: 请补充资产编号、机房具体位置和联系人

    U->>API: 上海市浦东新区……，联系人张三，138……
    API->>R: 从同一会话 Checkpoint 恢复
    R->>B: 合并新消息与已有槽位
    B->>S: query_engineers（只读建议）
    S->>DB: 查询技能、服务区域与排班
    DB-->>S: 候选工程师和可用时段
    S-->>B: 带版本号的候选方案
    B-->>API: 展示方案并请求明确确认
    API-->>U: 是否确认明日 13:00-18:00 的候选方案？

    U->>API: 确认
    API->>R: request_id + interrupt_id / confirmation_token
    R->>S: create_booking 建议 + 可信 ActorContext
    S->>S: 重校验身份、参数、确认、幂等和时段
    S->>DB: 同一事务写幂等结果、Booking、Outbox
    DB-->>S: booking_id
    S-->>R: 已创建的数据库事实
    R-->>API: 汇总故障建议和预约结果
    API-->>U: 返回 booking_id 与注意事项
    DB-->>W: 扫描已提交 Outbox
    W-->>U: 异步发送预约通知
```

### 2.2 分阶段责任链

| 阶段 | 责任组件 | 主要输入 | 主要输出 | 副作用 | 当前证据 |
|---:|---|---|---|---|---|
| 1 | FastAPI | `thread_id`、`request_id`、用户消息、Actor 上下文 | 合法请求对象 | 无 | **待验证规格**；当前没有 API 实现或测试 |
| 2 | 请求与会话层 | `tenant_id + user_id + thread/session_id`、请求指纹 | 隔离后的线程键、Checkpoint、重放结果 | 无 | **待验证规格**；当前没有 Checkpointer 实现或测试 |
| 3 | 安全门禁 | 认证、租户、速率、输入和不可信内容 | 可信运行上下文或拒绝结果 | 无 | **目标设计** |
| 4 | Router | 当前消息和少量状态 | `consult`、`booking`、`compound` 或 `unknown` | 无 | **待验证规格**；当前没有 Router 代码 |
| 5 | Supervisor 决策节点 | `compound`、任务状态、上一步 Observation、剩余预算 | State 更新和下一节点标签/`Command` | 无 | **目标设计**；专业 Agent 不作为 Tool |
| 6 | 咨询 Agent 节点/子图 | 设备、型号、故障码、租户/文档权限 | 诊断摘要、风险、证据 ID | 无 | **目标设计**；真实 Embedding、Cross-Encoder 和远端模型未接入 |
| 7 | 预约 Agent 节点/子图 | 最新消息、已有槽位、基准时间、时区、Schema | 槽位、缺槽、确认状态、动作建议 | 无 | **目标设计** |
| 8 | 时间与业务校验 | “明天”“下午”等表达 | 标准日期和起止时间、校验错误 | 无 | **目标设计** |
| 9 | 澄清/Checkpoint | 缺失槽位或时间歧义 | 一个最小必要问题、可恢复状态 | 无 | **待验证规格**；需补`interrupt`恢复和越权测试 |
| 10 | 工程师查询 | 设备技能、地区、标准时间段 | 带版本的候选方案 | 只读 | **目标设计** |
| 11 | 用户确认 | 候选方案、版本和确认内容 | 绑定本次动作的确认结果 | 无 | **待验证规格**；需补精确`interrupt_id`和候选版本测试 |
| 12 | 预约 Service | 可信 ActorContext、候选、确认、幂等键 | 允许执行的业务命令 | 无 | **待验证规格**；当前没有业务门禁实现 |
| 13 | 数据库事务 | 预约命令、请求哈希、排班事实 | `booking_id`、幂等终态、Outbox 事件 | 有 | **待验证规格**；当前没有 SQLite 实现或测试 |
| 14 | Supervisor/API | 诊断结果、数据库预约结果 | 面向用户的最终回答 | 无 | **目标设计** |
| 15 | Outbox/Inbox | 已提交事件、稳定 `event_id` | 通知、去重记录 | 有，可重试 | **待验证规格**；真实 Broker/Worker 是生产演进 |
| 16 | Memory Service | 明确记住/更正/删除、消息和业务事件 | Event、唯一 Memory Operation、版本化记忆 | 同步或后台写 | **目标设计** |
| 17 | 用户行为分析 Agent | 已完成咨询、预约和反馈事件 | 带来源的偏好 Candidate | 异步提出候选 | **目标设计**；不进入在线图，Policy 与事务层决定是否提交 |

## 3. Router 与 Supervisor 的具体流程

### 3.1 Router

Router 是前置、轻量、尽量确定性的粗粒度分类器：

```text
consult  → 直接进入咨询 Agent
booking  → 直接进入预约 Agent
compound → 交给 Supervisor 持续编排
unknown  → 普通说明、澄清或人工入口
```

目标 Router 可以先用确定性规则检测咨询与预约信号，产生`consult/booking/compound/unknown`。`compound`不是调用某个子 Agent 工具，而是路由到 Supervisor 决策节点；后者根据共享 State 返回下一节点。当前没有`agent_rag_demo.py`或 FastAPI 图代码，因此这条复合链路属于待实现规格。

### 3.2 Supervisor

Supervisor 只在复合任务中出现。它是 LangGraph 决策节点，不是持有子 Agent Tool 列表的 ReAct Manager；每一步读取共享 State，返回 State 更新与下一节点标签或`Command(goto=...)`。State 至少维护：

- 当前意图与 `task_stage`；
- 已完成能力与未决问题；
- 剩余步骤、Deadline 和调用预算；
- 咨询 Agent 返回的摘要、风险和证据引用；
- 预约 Agent 返回的槽位、候选方案和待确认动作。

咨询和预约能力实现为 LangGraph 节点或子图，只接收完成本次执行所需的字段。例如咨询节点返回`diagnosis_summary`、`risk_level`和`source_ids`，不会把全部候选 Chunk 和内部过程复制给预约节点。专业节点内部可以调用 RAG、模型、排班查询和预约 Service 等业务工具。

## 4. 咨询 Agent 与 RAG 流程

目标检索链路为：

```text
查询规范化/改写
  → tenant、产品、型号、文档版本和权限过滤
  → BM25 与 Dense 并行召回
  → RRF 融合
  → Cross-Encoder 精排
  → 远端模型基于证据生成
  → 返回 diagnosis_summary + risk_level + source_ids
```

### 4.1 为什么使用混合召回

- 故障码、型号和部件名适合 BM25 精确匹配；
- 口语症状、同义表达适合 Dense 语义召回；
- RRF 使用名次而非直接混合不同量纲的分数；
- Cross-Encoder 对较小候选集做查询—文档联合判断，提高 Top 结果相关性。

### 4.2 回退顺序

```text
Cross-Encoder 超时/失败 → 直接使用 RRF 排名
Dense 服务不可用       → BM25-only，并标记 degraded_reason
远端模型不可用         → 返回带来源的检索摘要，或稍后重试/转人工
```

降级只能降低回答质量，不能扩大工具权限。RAG 文档必须作为 `UNTRUSTED_CONTEXT`，其中的指令不能触发预约、取消或跨租户读取。

## 5. 本地 Qwen3-1.7B 的预约子流程

### 5.1 小模型所处位置

```text
Router 已确定 booking/compound
  → Supervisor 返回 Command(goto="booking_agent")
  → LangGraph 进入预约 Agent 节点/子图
  → 预约 Agent 构造结构化 Prompt
  → Qwen3-1.7B 输出槽位与动作建议
  → 代码解析并验证
  → 澄清、查候选或等待确认
```

Qwen 不负责顶层 Router，也不直接执行工具。它解决的是高频、边界清晰、Schema 固定的“预约域结构化理解”。

### 5.2 模型输入

预约 Agent 应传入：

- 当前用户消息；
- 已有会话槽位和每个槽位的来源；
- 当前业务阶段；
- `request_time` 和 `user_timezone`；
- 本阶段允许建议的工具白名单；
- 固定 JSON Schema，以及“缺失字段不得猜测”的约束。

概念输入示例：

```json
{
  "request_time": "2026-08-12T10:00:00+08:00",
  "user_timezone": "Asia/Shanghai",
  "task_stage": "COLLECTING",
  "latest_user_message": "机房服务器无法启动并提示找不到启动设备，帮我预约明天下午维修",
  "current_slots": {},
  "allowed_tools": ["query_engineers"]
}
```

### 5.3 建议统一的模型输出协议

当前材料中的日期字段同时出现过 `date` 和 `service_date`。正式实现前应冻结唯一 Schema。为了把“语言识别”和“日期计算”分开，建议模型保留原始时间表达，再由确定性时间服务生成标准字段：

```json
{
  "action": "FINAL",
  "slots": {
    "device": "机房服务器",
    "asset_id": null,
    "operating_system": null,
    "fault_code": "NO_BOOT_DEVICE",
    "service_date_text": "明天",
    "time_range_text": "下午",
    "server_room_location": null,
    "contact_name": null,
    "contact_phone": null
  },
  "missing_slots": [
    "asset_id",
    "server_room_location",
    "contact_name",
    "contact_phone"
  ],
  "need_confirmation": false
}
```

这里的 `FINAL` 只表示“本轮返回结构化结果，不提出工具调用”，不表示预约已经创建。只有在状态机允许、参数完整且服务端门禁通过时，模型才可以提出 `TOOL_CALL`；它仍然只是一条不可信建议。

### 5.4 校验顺序

```text
JSON 解析
  → Pydantic / JSON Schema
  → 字段枚举和长度
  → 确定性时间解析
  → 多轮槽位继承与冲突
  → 当前业务状态
  → 工具白名单
  → 身份与租户
  → 用户确认与确认令牌
  → 幂等和数据库事务
```

模型只识别“明天、下周五、下午”等语言片段；时间服务使用 `request_time + user_timezone` 计算标准时间，再校验歧义、营业时间和时区。不能让模型自行把相对日期当成最终业务事实。

### 5.5 失败与升级

- JSON 或 Schema 错误：在剩余 Deadline 内最多做一次带错误反馈的受控修复；
- 仍然失败、关键字段低置信或输入过于复杂：升级远端结构化模型；
- 本地模型服务不可用：低风险场景可使用规则抽取或远端模型；
- 推理槽位拥塞：使用有界队列，超过等待预算后返回 `Retry-After` 或对低风险任务降级；
- 高风险写操作：不得为了可用性切换到未经验证的模型后自动执行。

无论如何降级，都不能跳过身份、确认、幂等和事务门禁。

## 6. 会话状态、槽位状态和业务状态机

### 6.1 三种状态不能混用

| 状态 | 示例 | 事实地位 |
|---|---|---|
| Agent 工作状态 | 当前意图、已调用能力、下一步、剩余预算 | 仅用于工作流推进 |
| 会话/槽位状态 | 设备、故障、时间表达、地址、候选方案、确认令牌 | 辅助对话，必须保留来源 |
| 数据库业务事实 | 已创建预约、工程师排班、幂等终态 | 最终事实源 |

Checkpoint 只保存工作流恢复点，不能替代预约事务、幂等记录或数据库约束。

### 6.2 建议的槽位元数据

每个槽位不是单独一个字符串，而应至少包含：

```json
{
  "value": "明天下午",
  "source_message_id": "message-001",
  "updated_at": "2026-08-12T10:00:00+08:00",
  "confidence": 0.98,
  "confirmed": false
}
```

用户后来明确说“改成周五上午”时，只覆盖日期和时段；设备、故障和地址继续继承。最新显式指令优先，但要保留来源和修改记录。

### 6.3 预约状态机

```mermaid
stateDiagram-v2
    [*] --> COLLECTING
    COLLECTING --> COLLECTING: 缺槽 / 歧义 / 用户补充
    COLLECTING --> READY_TO_CONFIRM: 槽位合法且候选方案已生成
    READY_TO_CONFIRM --> COLLECTING: 用户修改槽位，旧确认失效
    READY_TO_CONFIRM --> COLLECTING: 用户拒绝并要求新方案
    READY_TO_CONFIRM --> CREATING: 有效确认 + Service 门禁通过
    CREATING --> CREATED: 事务提交成功
    CREATING --> READY_TO_CONFIRM: 冲突，重新生成候选
    CREATED --> [*]
```

候选方案必须带版本或摘要哈希。确认之后如果用户修改时间、地址或工程师，旧确认令牌立即失效，不能拿旧确认执行新参数。

## 7. 确认、幂等和预约事务

### 7.1 用户确认

补齐 FastAPI + LangGraph 实现后，确认链路必须通过以下验收：

- `interrupt()` 暂停预约图；
- 客户端必须带回精确的 `request_id + interrupt_id`；
- `approved` 使用严格布尔值；
- 错误、过期或跨用户的 interrupt 不能恢复；
- 并发批准/拒绝只能有一个决定生效；
- 审批后节点失败可以从 Checkpoint 恢复。

生产实现还应把确认绑定到候选方案版本、规范化参数摘要和有效期，而不只是绑定一个布尔值。

### 7.2 Service 执行前重新校验

Service 不接受模型提供的身份作为可信数据，而是从 ActorContext 和数据库重新取得事实，至少校验：

1. 用户和租户；
2. 工具名与动作 Schema；
3. 资产编号、设备型号、机房位置、联系人和时间范围；
4. 候选工程师的技能、服务区域和最新排班；
5. 确认令牌、候选版本和当前状态；
6. 幂等键与规范化参数哈希；
7. 数据库中的最新时段冲突。

### 7.3 SQLite 目标事务顺序

待实现的`reliable_booking_demo.py`应验证以下顺序：

```text
BEGIN IMMEDIATE
  → 查询 idempotency_record
  → 相同 Key + 不同 request_hash：拒绝
  → 相同 Key + SUCCEEDED：返回原 booking_id
  → 首次请求：插入 PROCESSING
  → 在事务内重新检查工程师时间冲突
  → 插入 Booking
  → 插入 BOOKING_CREATED Outbox 事件
  → 更新幂等记录为 SUCCEEDED 并保存响应
COMMIT
```

目标 SQLite 原型应包含 Booking、Idempotency、Outbox、Inbox 和 Notification Log；审计表和 OpenTelemetry Trace 需要单独设计。当前没有数据库脚本或测试，不能描述为已经实现。

## 8. Outbox、Inbox 与异步处理

预约和通知不能用“先写数据库，再直接发消息”的方式完成，否则任意一步失败都会造成双写不一致。

目标语义为：

```text
预约业务效果：相同幂等键至多创建一份
Outbox 事件：至少一次发布
Inbox 本地消费效果：按 event_id 去重
外部短信：是否重复仍取决于供应商幂等能力
```

可靠预约实现必须覆盖以下故障测试：Worker 发布事件后、标记 Outbox 为`SENT`前崩溃；恢复后事件再次发布，消费者使用 Inbox 识别相同`message_id`，避免重复写本地通知记录。当前没有对应测试代码或运行结果。

生产环境还需要真实 Broker、独立 Worker、指数退避和 Jitter、死信处理、Outbox 最老积压告警，以及对外部供应商传递稳定幂等键。

## 9. 记忆读写流程

记忆分成在线读取和独立写入两条链路。在线请求先加载当前 Checkpoint 和最近对话，再按可信`tenant_id + user_id`过滤长期记忆，最多注入首版 Top-5；预约、排班、权限仍回查业务数据库。长期记忆被视为带来源的不可信辅助上下文，不能直接授权工具或证明业务状态。

写入侧不把模型摘要直接写进向量库，而使用统一协议：

```text
消息 / 工具结果 / Booking 业务事件 / 任务检查点
  → 不可变 Memory Event
  → 原子 Memory Candidate
  → 作用域、授权、敏感信息、来源、价值与 TTL Policy
  → 按 memory_type 生成稳定 Key 并读取旧版本
  → ADD / UPDATE / INVALIDATE / DELETE / NOP / REVIEW
  → operation_id 幂等提交 + version CAS
  → 权威记忆状态、历史版本、操作日志、索引 Outbox
  → 全文 / 向量派生索引
```

用户明确要求记住、更正、删除，以及下一轮必须可见的任务状态走同步路径；普通摘要、重复合并、TTL 清理和多次行为巩固走异步 Worker。两种路径共用相同 Event、Policy、Operation 和事务协议。

本项目的首个实现增量应只支持少量低风险偏好，例如机房上门时段、联系渠道和经过确认的设备类型偏好，并验证跨租户隔离、重复事件、两个会话并发纠正、一次性例外和删除后不再召回。详细方案见[《企业机房与办公电脑维修 Agent：记忆系统工程化设计》](./memory_system_design.md)。当前目录没有这套服务的可运行实现，因此以上均为**目标设计**。

## 10. Trace、审计和可观测流程

目标链路在入口创建一个 Trace ID，并为以下阶段建立 Span：

```text
FastAPI 请求
  ├─ 会话读取 / 幂等检查
  ├─ Router
  ├─ Supervisor step
  ├─ 咨询 Agent
  │   ├─ BM25
  │   ├─ Dense
  │   ├─ RRF
  │   ├─ Rerank
  │   └─ 远端模型
  ├─ 预约 Agent
  │   ├─ 本地 Qwen
  │   ├─ 时间解析
  │   └─ 工程师查询
  └─ Booking Service / DB transaction
```

建议记录模型版本、Token、耗时、重试、检索候选 ID、排名变化、工具名、参数摘要、数据库结果和 `degraded_reason`，但不记录模型隐藏思维链，也不默认保存完整敏感原文。

Agent State 中的`events`字段即使未来实现，也不等于 OpenTelemetry Span。当前快照没有可展示的真实 Trace、运行事件或持久审计表，因此这部分必须标为**目标设计**。

记忆写入还应建立`event_id → candidate_id → operation_id → memory_id/version → index_outbox_id`关联。这样不仅能看到数据库里有什么，也能解释为何写入、为何拒绝、依据哪条消息、覆盖了哪个版本，以及索引何时可见。

## 11. 失败与降级总表

| 故障 | 处理方式 | 不允许的做法 |
|---|---|---|
| Router 低置信 | 澄清或进入受限 Supervisor | 随机选择高风险工具 |
| Dense 不可用 | 回退 BM25-only，记录降级原因 | 伪装成完整混合检索 |
| Cross-Encoder 超时 | 使用 RRF 排名 | 无限重试拖垮总 Deadline |
| 远端模型 429/超时 | 有依据摘要、稍后重试或转人工 | 无证据生成复杂故障结论 |
| 本地 Qwen Schema 错误 | 一次受控修复，仍失败则升级远端 | 循环重试或解析残缺 JSON |
| 本地模型拥塞 | 有界队列、低风险降级、`Retry-After` | 无限排队推高 P95 |
| 槽位缺失/时间歧义 | 只询问资产编号、机房位置等最小必要信息 | 脑补资产、机房位置、联系人或日期 |
| 用户确认后修改槽位 | 旧令牌失效，重新查候选和确认 | 用旧确认执行新参数 |
| 数据库时段冲突 | 回滚并返回新候选 | 在内存中假装预约成功 |
| 副作用结果未知 | 先按幂等键/operation ID 对账 | 直接再次创建预约 |
| 通知失败 | 预约保持已提交，Outbox 重试 | 回滚已经成功的预约 |
| Outbox 重复发布 | Inbox/业务唯一约束去重 | 宣称 Broker 自动保证端到端 Exactly Once |
| 重复 Memory Event | event/candidate/operation 稳定 ID 去重 | 重复生成两份偏好 |
| 两个会话同时改偏好 | version CAS，失败后重读并重新 Reconcile | 最后提交者静默覆盖 |
| 记忆索引延迟或失败 | 从权威记忆表读取关键 Key，Outbox 重试/重建索引 | 把向量库当唯一事实源 |
| 记忆冲突或来源不足 | 保留旧状态并进入 REVIEW | 让模型猜一个新值覆盖 |
| 用户删除记忆 | 确定性 DELETE 操作并传播到索引与缓存 | 只删 Prompt 中的一行文本 |

## 12. 待实现模块与目标流程的映射

### 12.1 FastAPI + LangGraph 目标模块

入口与状态：

- `ChatRequest`：`thread_id`、`request_id`、`message`；
- `ReviewRequest`：额外包含 `interrupt_id` 和严格布尔 `approved`；
- `RunContext`：可信的 `tenant_id`、`user_id`、`operation_id`；
- `AgentState`：消息、事件、意图、文档、待确认动作、审批结果和预约号。

目标图：

```text
START
  → classify_intent
      ├─ faq → retrieve_documents → answer_faq → END
      ├─ chat → answer_chat → END
      └─ booking → draft_booking → interrupt
                       ├─ approve → execute_booking → END
                       └─ reject  → reject_booking  → END
```

实现时可以先使用规则分类器、固定检索文档、`InMemorySaver`、内存请求 Store、BookingService 和 thread 锁作为教学替身，但必须补齐`python_fastapi_langgraph_demo`目录、测试与运行说明后才能声称验证。该基础图只验证单流程路由与审批；完整多 Agent 版本还应把咨询和预约实现为节点/子图，并增加 Supervisor 决策节点。

### 12.2 Router + RAG 目标模块

目标概念流程：

```text
route_node(state)
  → consult：goto consult_agent
  → booking：goto booking_agent
  → compound：goto supervisor_decide
supervisor_decide(state)
  → Command(goto="consult_agent" | "booking_agent" | "clarify" | "finalize")
```

实现后应验证`compound`分流、LangGraph节点跳转、BM25/Dense/RRF顺序、产品元数据过滤、缺地址澄清，以及未确认时禁止执行创建工具。首版可以使用手写概念向量、规则 Rerank 和规则槽位抽取，但必须明确这些都是教学替身。

### 12.3 SQLite 可靠预约目标模块

实现后应独立验证：

- 同一 Idempotency-Key + 相同参数返回原预约；
- 同一 Key + 不同参数产生冲突；
- 事务内重新检查工程师时段；
- Booking 和 Outbox 同时提交或同时回滚；
- Outbox 重发时由 Inbox 去重本地通知效果。

## 13. 建议的集成顺序

1. 创建 FastAPI + LangGraph 基础工程，实现`consult/booking/compound/unknown` Router，以及咨询、预约两个节点/子图；
2. 为复合分支实现 Supervisor 决策节点，以条件边或`Command(goto=...)`跳转，禁止把专业 Agent 包装成 Supervisor Tool，并补齐图路由测试；
3. 在咨询节点内部接入 RAG 接口替身，在预约节点内部接入槽位抽取和业务工具替身，补齐端到端测试；
4. 冻结预约槽位 JSON Schema，统一日期字段，并实现时间服务和多轮槽位合并；
5. 接入 Qwen 适配器，加入 Schema 修复、低置信升级和固定 51 条回归集；
6. 实现可靠预约 Repository/Service 和 SQLite 事务测试；
7. 把确认绑定到候选版本和参数哈希，补工程师查询与并发冲突测试；
8. 接入 Outbox Worker/Inbox，再补真实消息代理故障演练；用户行为分析 Agent 从这条异步链路消费事件，不进入在线 Supervisor 图；
9. 增加最小记忆闭环：不可变 Event、2～3 个低风险偏好 Schema、Policy、稳定 Key、操作日志、CAS、更正与删除测试；先使用结构化查询；
10. 再接记忆异步 Worker、Outbox 和 FTS5；离线评测证明有收益后才增加向量召回和后台巩固；
11. 最后接入真实混合 RAG、远端模型、JWT/ACL、持久 Checkpoint、审计和 OpenTelemetry。

每完成一步，都应保留上一步的契约测试，避免同时替换模型、检索、状态和事务后无法定位问题。

## 14. 面试时的一句话版本

> 请求进入 FastAPI 后先建立可信的用户、租户、会话和幂等上下文，再进入 LangGraph `StateGraph`。确定性 Router 让简单请求直达咨询或预约节点；复合任务进入 Supervisor 决策节点，由条件边或`Command`在咨询、预约、澄清和结束节点之间路由。咨询节点内部使用混合 RAG 和远端模型生成带证据建议，预约节点内部使用本地 Qwen 提取槽位，并调用工程师查询等业务工具；缺槽就`interrupt`澄清，完整后展示候选并要求确认。确认后业务 Service 重新校验身份、参数、状态、幂等和时段冲突，在事务中写 Booking 与 Outbox。用户行为分析和跨会话记忆消费异步事件，不参与在线 Supervisor 编排。模型始终不能直接写数据库。

## 15. 主要依据

- 总体口径与证据边界：[`README.md`](./README.md)、[`consistency_check.md`](./consistency_check.md)
- 完整责任链与基础原理：[`beginner_tutorial.md`](./beginner_tutorial.md)
- 记忆写入协议与落地阶段：[`memory_system_design.md`](./memory_system_design.md)
- 一线 Agent 面试回答：[`interviewer_1_agent.md`](./interviewer_1_agent.md)
- 架构、模型路由和可观测性：[`interviewer_2_architect.md`](./interviewer_2_architect.md)
- 稳定性、重试和并发：[`interviewer_3_reliability.md`](./interviewer_3_reliability.md)
- 状态图、事务和训练样例：[`onsite_tasks.md`](./onsite_tasks.md)
- 待补实现：`python_fastapi_langgraph_demo/`、`beginner_examples/agent_rag_demo.py`、`beginner_examples/reliable_booking_demo.py`；当前快照不存在这些路径，不能作为证据引用
