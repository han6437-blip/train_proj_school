# Milvus + Elasticsearch + Chroma：AI 应用开发向量数据库面试重点教程

> 版本：2026-08-10  
> 定位：面向 AI 应用开发与面试，不是三套产品的完整运维手册。  
> 贯穿案例：企业机房与办公电脑维修知识库，按租户检索服务器/电脑手册、操作系统与固件说明、故障方案和服务政策。  
> 示例版本口径：Milvus 3.0.0 + PyMilvus 3.0.1、Elasticsearch 9.4.x、Chroma 1.5.9；生产项目应固定客户端与服务端版本并核对兼容矩阵。

## 0. 先说结论：真正需要掌握的不是三套 CRUD

公开面经不是严格统计，只能用来判断题型。近期 AI 应用开发面经反复出现的是：为什么需要向量检索、Embedding 与分块、HNSW/IVF、混合检索、向量库选型、检索评估、租户隔离和生产调优，而不是让候选人背完某个产品的所有命令。

可参考这些题型样本：

- [AI 后端、LLM 与向量库面试总结](https://www.nowcoder.com/discuss/861205328803762176)
- [AI 应用开发高频题：索引、Embedding、召回与重排](https://www.nowcoder.com/feed/main/detail/2b1964d4df704bd08cd00373c3cf0161)
- [真实面经追问：为什么向量化、如何分块、为何这样选库](https://www.nowcoder.com/discuss/comment/13692054)
- [RAG 项目从原型到生产的常见追问](https://www.nowcoder.com/discuss/863430474180333568)

面试时最重要的六句话：

1. **向量数据库是检索系统，不是 RAG 的全部。** 文档解析、分块、Embedding、过滤、混合召回、重排和评测中的任何一环都可能成为瓶颈。
2. **ANN 用可控的召回损失换取延迟与吞吐。** `top_k`、`ef`、`num_candidates`、`nprobe` 都不是越大越好，要在自己的查询集上画 Recall—Latency 曲线。
3. **Embedding 模型、维度、归一化方式和距离度量必须作为同一份数据契约管理。** 即使新旧模型维度相同，两个向量空间通常也不能混搜。
4. **权限条件必须进入检索过滤表达式。** 不能先跨租户召回，再依赖 LLM 或应用代码“尽量删掉”。
5. **关键词与语义检索互补。** 电脑/服务器型号、部件编号、蓝屏或固件错误码等精确词常由 BM25 占优；同义改写和自然语言故障描述常由 Dense 检索占优，RRF 是稳健的融合起点。
6. **没有通用最优向量库。** Chroma 适合快速原型和轻量本地应用；Elasticsearch 适合搜索本来就是主业务、需要 BM25/过滤/聚合的团队；Milvus 适合把大规模向量检索当成独立基础设施的场景。最终用数据规模、过滤分布、Recall、P95/P99、QPS、成本和运维能力做基准测试。

### 建议学习顺序

如果只剩 90 分钟，按这个顺序读：

1. 相似度、ANN、HNSW 和过滤；
2. 三库选型表；
3. Elasticsearch 混合检索和 Milvus 一致性；
4. 评测与模型迁移；
5. 精讲面试题。

这份教程刻意不展开：分布式共识算法源码、Lucene segment 内核、Milvus 每个协调组件、Chroma Cloud 管理、全部索引类型和各语言 SDK。

---

# 第一部分：先建立正确的向量检索模型

## 1. 一条知识库记录到底存什么

向量库里通常不是只有一个浮点数组。RAG 的最小检索单元是 `chunk`，建议至少保存：

| 字段 | 例子 | 用途 |
|---|---|---|
| `id` | `acme:manual-7:v3:chunk-12` | 幂等写入、更新和删除 |
| `embedding` | 768 维浮点向量 | 语义近邻检索 |
| `text` | 故障处理原文 | 返回给重排器与 LLM |
| `tenant_id` | `acme` | 强制租户隔离 |
| `doc_id` / `chunk_no` | `manual-7` / `12` | 引用、去重、相邻块扩展 |
| `status` / `acl` | `published` / `engineer` | 权限与发布状态过滤 |
| `source_uri` | 对象存储地址或页面 URL | 回溯原文 |
| `doc_version` | `3` | 增量更新与旧版本清理 |
| `embedding_version` | `bge-m3@2026-07` | 向量空间版本治理 |
| `chunker_version` | `heading-v2` | 解释检索变化、支持回滚 |

关键设计：业务数据库或对象存储通常是**事实源**，向量库是可重建的派生检索索引。预约状态、订单金额、库存等实时业务事实不应只保存在向量库里，更不应让相似度结果覆盖事务数据库的结果。

## 2. Cosine、内积和 L2 怎么选

设查询向量为 $q$，文档向量为 $x$：

$$
\operatorname{cos}(q,x)=\frac{q\cdot x}{\lVert q\rVert\lVert x\rVert}
$$

$$
\operatorname{IP}(q,x)=q\cdot x
$$

$$
\operatorname{L2}(q,x)=\lVert q-x\rVert_2
$$

| 度量 | 排序方向 | 常见场景 | 重点 |
|---|---|---|---|
| Cosine | 相似度越大越近；有些库返回 `1-cos` 距离 | 文本语义检索 | 忽略向量长度，关注方向 |
| Inner Product | 越大越近 | 模型按点积训练、推荐系统 | 同时受方向与模长影响 |
| L2 | 距离越小越近 | 图像、空间距离或模型明确按 L2 训练 | 关注绝对几何距离 |

如果向量都做了 L2 单位归一化，则：

$$
q\cdot x=\operatorname{cos}(q,x),\qquad
\lVert q-x\rVert_2^2=2-2\operatorname{cos}(q,x)
$$

所以三者在这个前提下可产生等价排序，但这不代表可以随便切换：

- 先遵循 Embedding 模型说明和训练目标；
- 文档向量与查询向量必须采用同一预处理和归一化约定；
- 不同数据库返回的可能是“距离”或经过变换的 `_score`，阈值不能照搬；
- RRF 融合排名比直接相加 BM25 分数与向量分数更稳，因为二者分数尺度通常不同。

**常见错误**：看到 Chroma 返回的 `distance` 越大，误以为越相似；或把 Elasticsearch 的 `_score > 0.8` 原样复制成 Milvus 的过滤阈值。应先确认产品、度量和版本的分数定义。

## 3. 精确检索与 ANN

精确检索会对候选集合中的所有向量计算距离。粗略计算量是 $O(Nd)$：$N$ 是向量数，$d$ 是维度。它结果准确、实现简单，适合：

- 小数据集；
- 强过滤后只剩少量候选；
- 建立 ANN 的真值基线；
- 离线评测或低 QPS 场景。

ANN（Approximate Nearest Neighbor）不会承诺返回数学上绝对正确的 Top-K。它通过少访问一部分向量换取速度。面试里不要只说“ANN 快”，要说清楚交换关系：

> 提高搜索宽度通常会提高索引召回率，同时增加延迟和 CPU；提高索引质量通常会增加建索引时间与内存；压缩会降低内存，但可能损失精度并需要重打分。

### 两种 Recall@K 不要混淆

**ANN 索引召回率**用精确搜索结果作真值：

$$
\operatorname{ANNRecall@K}=\frac{|TopK_{ANN}\cap TopK_{Exact}|}{K}
$$

它回答“索引有没有找回精确近邻”。

**业务检索 Recall@K**用人工标注的相关文档作真值：

$$
\operatorname{RetrievalRecall@K}=\frac{\text{Top-K 中召回的相关文档数}}{\text{该问题全部相关文档数}}
$$

它回答“RAG 有没有找到能回答问题的材料”。ANN Recall 很高而业务 Recall 很低，通常说明 Embedding、分块、数据清洗或查询表达有问题，而不是索引参数不够大。

## 4. 检索结构与压缩策略只精讲这些

| 结构/策略 | 核心思路 | 优点 | 代价与适用场景 |
|---|---|---|---|
| FLAT / brute force | 扫描全部候选 | 精确、无需训练索引 | 数据小、过滤后候选少、评测基线 |
| HNSW | 多层近邻图，从稀疏上层导航到底层 | 高召回、低延迟、常用 | 内存和建索引成本高；更新、过滤分布会影响表现 |
| IVF_FLAT | 先聚类，查询只搜索若干中心对应的桶 | 容量与速度易折中 | 要训练聚类；`nlist`、`nprobe` 需要调优 |
| PQ/SQ/量化 | 压缩向量或索引表示 | 显著省内存，提高缓存命中 | 牺牲召回；常用扩大候选集 + 原始向量重打分补偿 |

DiskANN、GPU 索引、稀疏倒排索引知道用途即可，除非 JD 明确要求，不必在第一轮回答里展开实现。

### 4.1 HNSW 三个高频参数

HNSW 可以类比为多层道路网络：高层是稀疏“高速公路”，快速接近目标区域；底层是密集“街道”，完成局部搜索。

- `M` / `max_neighbors`：每个节点连接数上限。增大通常改善图连通性和召回，但增加内存与建索引时间；
- `efConstruction` / `ef_construction`：建图时保留的候选范围。增大通常提高索引质量，但建索引更慢；
- `efSearch`、Milvus 的 `ef`、Chroma 的 `ef_search`：查询时探索宽度。增大通常提高召回，但增加延迟。

Elasticsearch 暴露的常用查询旋钮是 `num_candidates`，含义同样是扩大每个分片近似检索的候选范围。它不是 HNSW 论文里参数名的简单一一映射，但调优方向相同：候选越多，通常召回越高、查询越慢。

不要机械背“查询复杂度一定是 $O(\log N)$”。HNSW 是启发式图搜索，实际延迟取决于数据分布、维度、图参数、过滤条件、缓存和硬件；面试中用基准测试数据说话更严谨。

### 4.2 IVF 三个概念

- `nlist`：聚类桶数量；
- `nprobe`：一次查询探测多少个桶；增大通常提高召回并增加延迟；
- PQ：将向量切成子空间，用短码近似距离，适合内存紧张的大规模场景。

直觉上，HNSW 像沿路网找邻居，IVF 像先选几个最可能的城区再逐户搜索。强过滤后候选分布可能与原聚类或图结构不一致，因此“无过滤基准很快”不能代表真实业务查询也快。

## 5. 元数据过滤是正确性边界，不只是性能优化

企业知识库常见过滤：

~~~text
tenant_id = 当前租户
AND status = published
AND acl_group IN 当前用户角色
AND valid_from <= now < valid_to
AND product = 当前产品
~~~

正确做法是把从认证上下文得到的租户和权限条件注入数据库查询。不要接受模型输出或用户输入的 `tenant_id` 作为可信权限依据。

后文代码为便于比较三套 API，都用单值 `acl_group = customer`；真实用户拥有多个角色时，应使用各产品的集合/OR 过滤表达式，并集中在检索适配器中生成，不能字符串拼接不可信输入。

两类执行策略：

- **预过滤**：先限制允许集合，再在其中做 ANN。安全且能保证结果都符合条件，但高选择性或复杂过滤可能让图搜索更难；
- **后过滤**：先取向量候选再过滤。实现简单，但可能过滤后不足 K 条；若用于权限，候选内容还可能在链路中暴露，因此不能依赖它做安全边界。

Milvus 当前文档区分标准过滤和迭代过滤；Elasticsearch 的 kNN `filter` 是检索过程中的 pre-filter；Chroma 的 `where` 用于元数据过滤。三者 API 不同，但权限过滤必须在召回阶段生效这一原则相同。

## 6. 为什么混合检索通常优于只做 Dense

Dense 语义检索擅长“表达不同但意思相近”，BM25/稀疏检索擅长精确词项：

| 查询 | 更可能占优的通道 |
|---|---|
| “服务器开机后找不到系统盘”与“无法访问启动设备” | Dense |
| 蓝屏停止代码 `0x0000007B` | BM25 |
| 服务器型号 `PowerEdge-R750` | BM25 |
| 用户只描述现象、未使用手册术语 | Dense |

RRF（Reciprocal Rank Fusion）按各通道的**名次**融合，而不要求分数同尺度：

$$
score(d)=\sum_{r\in R}\frac{1}{c+rank_r(d)}
$$

其中 $c$ 是 RRF 的 `rank_constant`，不是向量检索的 Top-K。

推荐的生产检索漏斗：

~~~text
Query
  ├─ BM25 / sparse：召回 50～200
  └─ Dense ANN：召回 50～200
           ↓ RRF 去重融合
      候选 30～100
           ↓ Cross-Encoder / LLM reranker
        最终 5～10
           ↓ 邻块扩展、权限复核、上下文装配
          交给 LLM
~~~

这里的数字只是起始实验范围，不是生产默认值。候选数要由离线相关性和线上延迟共同决定。

---

# 第二部分：从入库到查询的工程主线

## 7. 离线入库链路

~~~text
事实源文档
  → 解析版面/OCR/去页眉页脚
  → 按标题与语义分块
  → 生成稳定 chunk_id
  → 批量 Embedding
  → upsert 向量 + 原文 + 元数据
  → 校验数量、维度、重复项与抽样可搜性
~~~

### 7.1 分块决定检索上限

- 太大：一个向量混入多个主题，语义被稀释，送给 LLM 又浪费上下文；
- 太小：缺少上下文，产生大量近重复块，答案证据不完整；
- 固定长度可以做基线，生产更常结合标题、段落、表格和代码结构；
- overlap 不是越大越好，过大会增加容量、重复召回和上下文冗余；
- 保存 `parent_doc_id`、标题路径和 `chunk_no`，召回后才能扩展父块或相邻块。

不要声称“512 token 最优”。不同文档、Embedding 模型和问法需要在标注集上比较。

### 7.2 稳定 ID 保证幂等

可以用以下字段生成确定性 ID：

~~~python
from hashlib import sha256


def make_chunk_id(
    tenant_id: str,
    doc_id: str,
    doc_version: str,
    chunk_no: int,
) -> str:
    raw = f"{tenant_id}|{doc_id}|{doc_version}|{chunk_no}"
    return sha256(raw.encode("utf-8")).hexdigest()
~~~

重跑入库任务时使用 `upsert`，避免网络重试把同一块插入多次。更新文档时先写新版本，验证完成后切换可见版本，再异步删除旧版本，能避免更新窗口内知识库暂时为空。

## 8. 在线查询链路

~~~text
认证上下文
  + 用户 Query
  → 术语归一化/必要时改写
  → 构造不可绕过的 ACL 过滤
  → Query Embedding
  → Dense + BM25/sparse 并行召回
  → RRF、去重、按文档限额
  → 精排
  → 相关性阈值/无答案判断
  → 上下文拼装与引用
  → LLM 生成
~~~

查询与文档 Embedding 有时需要不同提示前缀，例如某些模型要求 `query:` 与 `passage:`。这仍是“同一模型契约”，不能因为文本不同就随意换模型。

一个数据库无关的服务接口可以这样设计：

~~~python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class SearchHit:
    chunk_id: str
    text: str
    score: float
    metadata: dict[str, object]


class Retriever(Protocol):
    async def search(
        self,
        *,
        tenant_id: str,
        query: str,
        roles: tuple[str, ...],
        limit: int,
    ) -> list[SearchHit]: ...
~~~

应用层只依赖 `Retriever`，Milvus、Elasticsearch 和 Chroma 分别实现适配器。这样便于做影子流量对比和迁移，也避免把 LangChain 某个 VectorStore 对象泄露到整个业务层。

---

# 第三部分：三库怎么选

## 9. 一张表先做判断

| 维度 | Chroma | Elasticsearch | Milvus |
|---|---|---|---|
| 最适合 | Notebook、原型、单机本地知识库、轻量服务 | 搜索平台、站内搜索、日志/文档检索、BM25 + 向量混合 | 独立向量检索服务、大规模 Dense/Sparse/多向量场景 |
| 上手成本 | 最低 | 团队已有 ES 时较低，否则中等 | Lite 低；Standalone/Distributed 运维成本更高 |
| 关键词搜索 | 本地 `where_document` 是内容过滤，不等于 BM25 混合排序；稀疏 + RRF 高级 Search API 当前仅限 Cloud | 强项，分析器、BM25、过滤、聚合、Highlight 完整 | 支持稀疏向量、全文与混合检索，核心定位仍偏向量工作负载 |
| 向量索引 | 单节点 HNSW；分布式/Cloud 另有实现 | HNSW（含量化变体）及磁盘友好的 DiskBBQ | HNSW、IVF、FLAT、DiskANN、量化与 GPU 等选择多 |
| 横向扩展/高可用 | 本地客户端不应当作分布式 HA 服务 | 成熟分片、副本与集群生态 | Distributed 计算存储分离、面向大规模扩展 |
| 元数据/聚合 | 常用元数据过滤够直接 | 复杂过滤与聚合最强 | 标量过滤、分区/分区键、多租户层次 |
| 典型选择理由 | “最快验证检索方案” | “关键词和向量必须在同一搜索平台融合” | “向量规模、吞吐或索引选择已成为主要问题” |

### 9.1 可直接使用的选型回答

> 我不会先按品牌选。先明确向量数、维度、写入速率、QPS、P95/P99、业务 Recall@K、过滤选择性、关键词搜索、可用性和团队运维能力。个人原型优先 Chroma；已有 Elasticsearch 且关键词、过滤、聚合很重时优先在 ES 内做混合检索，减少数据同步；当向量规模和检索吞吐需要独立扩展、需要更多专用索引或多向量能力时评估 Milvus。最后用同一批真实向量、真实过滤比例和并发做基准，而不是引用厂商榜单。

### 9.2 什么时候三个都可能不是最佳答案

- 数据强关系、事务更新频繁、规模中等且团队只维护 PostgreSQL：可以评估 `pgvector`；
- 仅做单机离线近邻计算，不需要服务、权限、持久化或增量数据管理：FAISS 可能够用；
- 团队无法维护集群：托管服务的总成本可能低于自建；
- 数据只有几千条且强过滤后更少：关系库加精确向量扫描也可能已经满足 SLA。

面试加分点不是“永远上 Milvus”，而是能解释为什么少引入一个系统有时更可靠。

---

# 第四部分：Chroma——把原型做对

## 10. 重点概念

- `Client()`：内存型，适合实验；
- `PersistentClient(path=...)`：自动把本地数据持久化并在启动时加载；
- `HttpClient`：连接独立 Chroma 服务；
- `Collection`：存储与查询的基本单位，绑定索引配置和可选 Embedding Function；
- `add`：添加新记录；`upsert`：存在则更新，不存在则创建；
- `query`：近邻检索；`get`：按 ID/过滤读取，不做相似度排序；
- `where`：元数据过滤；`where_document`：文档内容过滤。

官方当前文档说明，单节点 Chroma 使用 HNSW，默认距离空间为 L2；文本 Embedding 场景若想使用 Cosine，应明确配置，不要靠猜测默认值。[Chroma Collection 配置](https://docs.trychroma.com/docs/collections/configure)

还要避免一个概念偷换：`where_document` 的 contains/regex 是内容约束，不是“BM25 与 Dense 两路召回后融合”。截至本教程核对日期，带稀疏向量和 RRF 的高级 Search API 仅限 Chroma Cloud，单节点文档仍标为计划支持；后续应随版本核对。[Chroma Search API Overview](https://docs.trychroma.com/cloud/search-api/overview)

## 11. Chroma 核心 API 片段

以下片段突出 Chroma 调用，不是独立可运行程序；`embed_documents()` 和 `embed_query()` 需要替换为同一版本的 Embedding 适配器，均输出 768 维向量：

~~~python
import chromadb


client = chromadb.PersistentClient(path="./data/chroma")

collection = client.get_or_create_collection(
    name="support-kb-v1",
    embedding_function=None,  # 由应用统一生成，便于显式版本治理
    configuration={
        "hnsw": {
            "space": "cosine",
            "ef_construction": 200,
        }
    },
    metadata={
        "embedding_version": "my-embedding@2026-07",
        "chunker_version": "heading-v2",
    },
)

texts = [
    "Windows 停止代码 0x0000007B 表示无法访问启动设备，应检查磁盘连接、存储控制器模式和启动配置。",
    "办公电脑无法开机时，先检查电源、指示灯、显示器连接和主板自检提示。",
]

collection.upsert(
    ids=["acme:manual-7:v3:0", "acme:manual-7:v3:1"],
    embeddings=embed_documents(texts),
    documents=texts,
    metadatas=[
        {
            "tenant_id": "acme",
            "doc_id": "manual-7",
            "doc_version": 3,
            "status": "published",
            "acl_group": "customer",
            "chunk_no": 0,
        },
        {
            "tenant_id": "acme",
            "doc_id": "manual-7",
            "doc_version": 3,
            "status": "published",
            "acl_group": "customer",
            "chunk_no": 1,
        },
    ],
)

result = collection.query(
    query_embeddings=[embed_query("蓝屏代码 0x0000007B 怎么处理")],
    n_results=10,
    where={
        "$and": [
            {"tenant_id": {"$eq": "acme"}},
            {"status": {"$eq": "published"}},
            {"acl_group": {"$eq": "customer"}},
        ]
    },
    include=["documents", "metadatas", "distances"],
)
~~~

如果希望 Chroma 自动调用 Embedding，可以给 Collection 绑定明确的 `embedding_function`，然后 `upsert(documents=...)`、`query(query_texts=...)`。生产中仍应固定模型名、版本、预处理和密钥来源；默认 Embedding Function 适合演示，不应成为隐式生产契约。

## 12. Chroma 最值得记住的坑

1. `get_or_create_collection` 遇到已存在 Collection 时，不代表你新传入的全部配置都会覆盖旧配置；部署启动时要检查实际配置与版本。
2. `PersistentClient` 是方便的本地持久化方式，不等于已经具备跨机器高可用、备份恢复、限流和多租户鉴权。
3. `distance` 的方向取决于空间定义；Cosine distance 通常越小越近，不要与 similarity 混用。Chroma 的 `l2` 定义是 squared L2，与前文写作 $\lVert q-x\rVert_2$ 的欧氏距离数值不同，但排序方向相同。
4. `upsert` 的四组列表按位置对应，长度和 ID 必须校验。
5. 本地服务变成多人共享服务后，应改用 Client-Server/Cloud 形态并重新评估认证、备份、并发和资源隔离。
6. 当前版本会把 Collection 的 Embedding Function 信息保存在服务端；旧客户端/服务端在取回 Collection 时可能仍要求再次提供。升级前要核对兼容矩阵。
7. 首批向量写入后维度固定，距离空间也不能随意原地切换；换模型或度量通常新建 Collection。
8. `PersistentClient` 面向本地开发、测试和嵌入式应用；避免多个进程同时写同一路径，生产共享服务优先使用 server-backed Chroma。
9. 自托管服务不能裸露公网。TLS、认证、授权、备份和恢复都要显式建设，落盘不等于高可用。

---

# 第五部分：Milvus——专用向量基础设施

## 13. 只理解这些架构概念

- **Collection**：类似表，定义字段、向量维度和索引；
- **Partition / Partition Key**：缩小搜索范围、组织多租户或冷热数据；不要无上限地“每个用户一个 Collection”；
- **Segment**：Milvus 的数据与索引处理单元，写入数据会经历 growing 与 sealed 等状态；
- **Proxy/接入层**：接收请求并汇总分布式结果；
- **Streaming、Query、Data Worker**：分别承担实时写入/增长数据、历史数据查询、压实和建索引等工作；
- **对象存储、元数据存储、WAL**：实现存算分离、恢复与扩展。

这套架构的面试意义是：写入成功、数据可搜索、数据落入历史 Segment 和索引建好不是同一个瞬间；因此 Milvus 提供不同一致性级别来控制“新写数据何时必须可见”。官方架构说明见 [Milvus Architecture Overview](https://milvus.io/docs/architecture_overview.md)。

截至本教程核对日期，Milvus 官方文档主线已经进入 3.0。3.0 的新 Storage V3、外部 Collection、Snapshot、在线字段演进等属于进阶能力，其中一些需要显式开启并带有升级/回滚约束；面试基础回答先把读写路径、Segment、索引、过滤和一致性讲清，不要把“3.0 已提供”误说成“默认全部启用”。[Milvus Release Notes](https://milvus.io/docs/release_notes.md)

### 13.1 五个容易混淆的名词

| 名词 | 主要目的 | 不要混淆成 |
|---|---|---|
| Partition | 数据组织与查询裁剪 | 权限系统或写入并行度 |
| Partition Key | 按高基数字段自动路由到物理 Partition | 一租户一个物理 Partition |
| Shard | 写入通道与并行度 | 租户隔离或读副本 |
| Segment | 数据持久化、建索引、加载与压实的物理单位 | Collection 的逻辑 Schema |
| Replica | 提升读吞吐与可用性，复制已加载 Segment | Strong/Session 等一致性级别 |

一个常见加分点：Replica 决定“有几份查询副本”，一致性级别决定“查询必须看到哪个时间点的数据”，两者不是同一个旋钮。

## 14. 部署方式

| 方式 | 用途 | 重要限制/代价 |
|---|---|---|
| Milvus Lite | Notebook、笔记本、边缘、小规模验证 | 当前官方文档仅列 Ubuntu/macOS 支持；只使用 FLAT，即使指定其他索引类型也会按 FLAT 处理 |
| Standalone | 单机服务、无需 Kubernetes 的中等生产负载 | 单机容量和故障域，仍要做备份、监控和恢复演练 |
| Distributed | Kubernetes、大规模与高可用 | 组件和运维复杂度最高，需要容量、对象存储与监控治理 |

Lite、Standalone、Distributed 使用接近的 `MilvusClient` API，连接 URI 不同。但“代码容易迁移”不代表性能与功能完全相同。[Milvus Lite 官方限制](https://milvus.io/docs/milvus_lite.md)

## 15. Milvus HNSW + 过滤检索片段

以下是突出数据库调用的片段，不是独立可运行程序；`embed_documents()` 与 `embed_query()` 需接入同一 Embedding 适配器。它需要 Standalone、Distributed 或对应托管服务；不要用 Milvus Lite 验证 HNSW 参数，因为当前 Lite 只使用 FLAT。

~~~python
from pymilvus import DataType, MilvusClient


client = MilvusClient(
    uri="http://localhost:19530",
    token="root:Milvus",  # 示例值；生产从密钥系统读取
)

schema = MilvusClient.create_schema(
    auto_id=False,
    enable_dynamic_field=False,
)
schema.add_field(
    field_name="id",
    datatype=DataType.VARCHAR,
    max_length=128,
    is_primary=True,
)
schema.add_field(
    field_name="tenant_id",
    datatype=DataType.VARCHAR,
    max_length=64,
)
schema.add_field(
    field_name="doc_id",
    datatype=DataType.VARCHAR,
    max_length=128,
)
schema.add_field(
    field_name="status",
    datatype=DataType.VARCHAR,
    max_length=32,
)
schema.add_field(
    field_name="acl_group",
    datatype=DataType.VARCHAR,
    max_length=64,
)
schema.add_field(
    field_name="text",
    datatype=DataType.VARCHAR,
    max_length=8192,
)
schema.add_field(
    field_name="embedding",
    datatype=DataType.FLOAT_VECTOR,
    dim=768,
)

index_params = client.prepare_index_params()
index_params.add_index(
    field_name="embedding",
    index_name="embedding_hnsw",
    index_type="HNSW",
    metric_type="COSINE",
    params={"M": 32, "efConstruction": 200},
)
index_params.add_index(
    field_name="tenant_id",
    index_type="INVERTED",
)
index_params.add_index(
    field_name="status",
    index_type="INVERTED",
)
index_params.add_index(
    field_name="acl_group",
    index_type="INVERTED",
)

client.create_collection(
    collection_name="support_kb_v1",
    schema=schema,
    index_params=index_params,
    consistency_level="Bounded",
)

client.upsert(
    collection_name="support_kb_v1",
    data=[
        {
            "id": "acme:manual-7:v3:0",
            "tenant_id": "acme",
            "doc_id": "manual-7",
            "status": "published",
            "acl_group": "customer",
            "text": "Windows 停止代码 0x0000007B 表示无法访问启动设备。",
            "embedding": embed_documents(
                ["Windows 停止代码 0x0000007B 表示无法访问启动设备。"]
            )[0],
        }
    ],
)

hits = client.search(
    collection_name="support_kb_v1",
    anns_field="embedding",
    data=[embed_query("蓝屏代码 0x0000007B 怎么处理")],
    filter=(
        'tenant_id == "acme" and status == "published" '
        'and acl_group == "customer"'
    ),
    limit=10,
    search_params={
        "ef": 128,
    },
    output_fields=["doc_id", "text", "status"],
    consistency_level="Session",
)
~~~

这些参数只是合理的实验起点，不是通用生产配置。`ef` 至少要与返回 K 协调，并用精确搜索真值对比。官方 HNSW 参数说明见 [Milvus HNSW](https://milvus.io/docs/hnsw.md)。

## 16. 四种一致性怎么答

Milvus 当前提供四种一致性级别，默认是 Bounded Staleness：

| 级别 | 语义 | 适用 |
|---|---|---|
| Strong | 查询等待到最新写入对当前视图可见 | 功能测试、必须读到最新数据且可接受更高延迟 |
| Bounded | 允许受控时间范围内的数据陈旧 | 通用检索与推荐，默认折中 |
| Session | 同一客户端会话能读到自己的写入 | 用户上传文档后立即在同一会话搜索 |
| Eventually | 不等待最新写入，副本最终收敛 | 对低延迟极敏感、能容忍短时不可见 |

面试不要只背名字。可以举例：用户刚上传一份手册并马上发问，使用 Session 能表达 read-your-writes；公共推荐流允许短暂不可见，可用 Bounded 或 Eventually；真正的订单状态仍应查事务数据库，而不是把向量库设成 Strong 后当订单库使用。

这里讲的是 Milvus Server。当前 Milvus Lite 只支持 Strong，一致性配置会按 Strong 处理；这也是不能把 Lite 的功能与性能结论直接外推到 Standalone/Distributed 的原因之一。

官方说明见 [Milvus Consistency](https://milvus.io/docs/consistency.md)。

## 17. Milvus 过滤、分区与混合检索

### 17.1 标准过滤与迭代过滤

- 标准过滤先计算符合表达式的标量位图，再在允许集合中执行 ANN；
- 过滤表达式非常复杂时，Milvus 还提供迭代过滤思路，逐步取向量候选并检查标量条件，可能减少复杂标量计算，但逐项处理也可能增加延迟；
- 应按真实 ACL、时间范围和产品条件压测，而不是只测无过滤的随机向量。

### 17.2 多租户层次

Milvus 可在 database、collection、partition、partition key 等层次组织租户。选择原则：

- 需要独立配额、生命周期或强隔离的大租户，可考虑独立 database/collection；
- 大量中小租户更适合共享 Collection，并用可信的 `tenant_id` 过滤与 partition key 路由；
- 不要把每个终端用户都映射成一个物理 Collection，元数据对象数量和运维成本会失控；
- 权限安全仍要在网关和应用服务执行，分区不是完整授权系统。

### 17.3 Dense + Sparse

Milvus 支持一个 Collection 中的多个向量字段，可分别发起 ANN 请求，再用 RRF 或 Weighted Ranker 合并；也支持基于文本字段生成 BM25 稀疏表示的能力。重点不是把所有通道都打开，而是用离线数据判断 Dense、Sparse 和多模态字段是否提供互补增益。[Milvus Multi-Vector Hybrid Search](https://milvus.io/docs/multi-vector-search.md)

---

# 第六部分：Elasticsearch——搜索与向量的一体化

## 18. 为什么 AI 应用常选 Elasticsearch

Elasticsearch 的优势不是“它也能存向量”这么简单，而是：

- 同一文档同时拥有 `text`、`keyword`、数值、日期和 `dense_vector`；
- BM25、分析器、同义词、Filter、聚合、Highlight 与向量检索在同一平台；
- 使用 Retriever API 可以把关键词和 kNN 组合成检索树；
- 团队已有 ES 监控、备份、权限和数据管道时，可以少维护一次跨库同步。

如果工作负载几乎全是超大规模向量、需要更多专用索引或独立扩缩容，Milvus 仍可能更合适。选型要比较真实负载，不要只比较功能清单。

## 19. 显式 Mapping：不要依赖动态推断

为了让下面两个查询 JSON 可以直接复用，本节使用 3 维**教学向量**；真实文本 Embedding 必须把 Mapping 的 `dims` 和所有文档/查询向量一起改成模型输出维度，例如 768。

~~~json
PUT support-kb-v1
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "tenant_id": {"type": "keyword"},
      "doc_id": {"type": "keyword"},
      "chunk_no": {"type": "integer"},
      "status": {"type": "keyword"},
      "acl_group": {"type": "keyword"},
      "title": {"type": "text"},
      "text": {"type": "text"},
      "embedding_version": {"type": "keyword"},
      "embedding": {
        "type": "dense_vector",
        "dims": 3,
        "index": true,
        "similarity": "cosine",
        "index_options": {
          "type": "int8_hnsw",
          "m": 16,
          "ef_construction": 100
        }
      }
    }
  }
}
~~~

为什么显式 Mapping：

- 维度与相似度是检索契约；
- `tenant_id`、状态和版本应该是 `keyword`，不能被文本分析器切词；
- 量化类型会影响内存、召回和写入成本；
- 动态映射一旦误判，常需要新建索引再 Reindex，而不是原地轻松修复。

Elastic 当前文档说明，`dense_vector` 可使用 HNSW，并提供 `int8_hnsw`、`int4_hnsw`、`bbq_hnsw` 等量化变体，也有磁盘友好的 `bbq_disk` 路径。本例显式选择 `int8_hnsw`，避免依赖会随版本与许可变化的默认值。量化能显著减少检索内存，但原始浮点向量仍会保存在磁盘上用于重打分与后续操作，因此“内存缩小 4 倍”不等于“总磁盘也缩小 4 倍”。[Elasticsearch dense_vector](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector)

## 20. kNN + 权限预过滤

~~~json
POST support-kb-v1/_search
{
  "size": 10,
  "_source": ["doc_id", "chunk_no", "title", "text"],
  "query": {
    "knn": {
      "field": "embedding",
      "query_vector": [0.013, -0.027, 0.041],
      "k": 20,
      "num_candidates": 200,
      "filter": [
        {"term": {"tenant_id": "acme"}},
        {"term": {"status": "published"}},
        {"term": {"acl_group": "customer"}},
        {"term": {"embedding_version": "my-embedding@2026-07"}}
      ]
    }
  }
}
~~~

本节 Mapping 与查询都使用 3 维教学向量，因此维度一致；它只用于解释 API，不代表适合真实语义检索。

- `k`：每个分片收集的近邻数，协调节点再合并全局结果；
- `num_candidates`：每个分片近似检索考虑的候选数，必须不小于 `k`；提高它通常改善召回并增加延迟；
- `size`：最终返回给客户端的数量；
- `filter`：kNN 过程中的 pre-filter，保证候选符合条件。

分片数变化会改变候选合并、资源与延迟，因此不要把单分片开发环境调出的 `num_candidates` 无条件搬到生产集群。[Elasticsearch kNN query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-knn-query)

## 21. BM25 + Dense + RRF

下面的关键点是：**两个子检索器都带相同的租户与状态过滤**。只给 kNN 分支加租户条件，BM25 分支仍可能返回其他租户的文本。

~~~json
POST support-kb-v1/_search
{
  "size": 10,
  "_source": ["doc_id", "chunk_no", "title", "text"],
  "retriever": {
    "rrf": {
      "retrievers": [
        {
          "standard": {
            "query": {
              "bool": {
                "must": {
                  "multi_match": {
                    "query": "蓝屏代码 0x0000007B 怎么处理",
                    "fields": ["title^2", "text"]
                  }
                },
                "filter": [
                  {"term": {"tenant_id": "acme"}},
                  {"term": {"status": "published"}},
                  {"term": {"acl_group": "customer"}}
                ]
              }
            }
          }
        },
        {
          "knn": {
            "field": "embedding",
            "query_vector": [0.013, -0.027, 0.041],
            "k": 50,
            "num_candidates": 200,
            "filter": [
              {"term": {"tenant_id": "acme"}},
              {"term": {"status": "published"}},
              {"term": {"acl_group": "customer"}},
              {"term": {"embedding_version": "my-embedding@2026-07"}}
            ]
          }
        }
      ],
      "rank_window_size": 50,
      "rank_constant": 60
    }
  }
}
~~~

RRF 根据排名融合，不需要手工把无界的 BM25 分数归一化到向量分数区间。`rank_window_size` 增大可能改善融合质量，但也提高开销；如果 kNN 的 `k` 大于窗口，多出的结果会被截断。官方说明见 [Elasticsearch RRF](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion)。

## 22. Elasticsearch 最值得记住的坑

1. **近实时不是每次写入立刻可搜。** 文档 GET 可见与 Search 可见不是同一概念；需要理解 refresh，批量入库不要对每条记录强制刷新。
2. **HNSW 写入昂贵。** 向量索引开启后，建图会增加 CPU、内存和写入时间；大批导入要做批量、刷新和副本策略评估。
3. **分片不是越多越快。** 每个分片都有 HNSW 图、候选收集和协调开销，过度分片会浪费内存并增加尾延迟。
4. **量化不是免费优化。** 要用扩大候选 + 原向量重打分或业务评测验证召回损失。
5. **`script_score` 精确扫描不是一无是处。** 小候选集合、强过滤和构造 ANN 真值时有价值，但不能无条件扫描千万级向量。
6. **混合检索的所有分支都要执行 ACL。** 这是正确性和安全要求。
7. **RRF 有 API 边界。** 当前官方文档列出 `scroll`、`sort`、`rescore` 等限制；需要二阶段重排时可改用 Retriever 树中的专用重排器或由应用层处理，具体以部署版本为准。

---

# 第七部分：生产化与调优

## 23. 评测：先隔离问题，再调参数

### 23.1 建一个最小 Golden Set

每条样本至少包含：

~~~json
{
  "query": "蓝屏代码 0x0000007B 怎么处理",
  "tenant_id": "acme",
  "roles": ["customer"],
  "relevant_chunk_ids": ["acme:manual-7:v3:0"],
  "forbidden_chunk_ids": ["other-tenant:any"],
  "tags": ["error_code", "exact_term"]
}
~~~

样本要覆盖：短问法、口语改写、精确型号/错误码、多跳问题、无答案、权限边界、新写后立刻查询和高选择性过滤。只用模型生成的“漂亮问题”会高估效果，必须加入真实查询与人工审核。

### 23.2 指标分层

| 层次 | 指标 | 回答的问题 |
|---|---|---|
| ANN 索引 | ANN Recall@K、P50/P95/P99、QPS、内存 | 近似索引相对精确近邻丢了多少 |
| 业务召回 | Recall@K、MRR、nDCG、Precision@K | 相关证据是否被找到、排得是否靠前 |
| 精排 | nDCG、MRR、Top-N 命中 | Reranker 是否真正改善顺序 |
| RAG 端到端 | 引用正确性、Faithfulness、答案正确率、拒答准确率 | LLM 是否依据正确证据作答 |
| 安全与新鲜度 | ACL 泄漏数必须为 0、写入可见时间、删除传播时间 | 是否越权、数据多久生效 |
| 成本 | Embedding/重排成本、索引构建时间、存储、CPU/GPU | 效果提升是否值得 |

### 23.3 推荐实验顺序

1. 固定 Embedding、分块和过滤，用 FLAT/精确扫描生成近邻真值；
2. 扫描 HNSW `ef`、ES `num_candidates`、IVF `nprobe`，画 ANN Recall—P95 曲线；
3. 固定达到目标 ANN Recall 的参数，再比较业务 Recall@K；
4. 比较 BM25、Dense、RRF、RRF + Rerank；
5. 再改分块与 Embedding，并做消融实验；
6. 最后在真实并发、真实过滤分布、冷热缓存、更新和删除下压测 P99。

如果不先分层，一看到答案错就把 `top_k` 从 10 调成 100，通常只会让延迟、重排成本和上下文噪声一起上升。

## 24. 容量估算的第一步

仅原始 Float32 向量的大小约为：

$$
N\times d\times4\text{ bytes}
$$

例如 1000 万条、768 维：

$$
10,000,000\times768\times4\approx30.72\text{ GB}
$$

这还没有包含 HNSW 图、ID、元数据、原文、段结构、进程开销、副本、缓存和索引构建峰值。面试里只报 30.72 GB 作为“集群内存”是不完整的；正确说法是先算原始向量下限，再用真实索引和副本基准测容量。

## 25. Embedding 模型升级：必须隔离新旧向量空间

不要让新旧模型向量进入同一检索空间，即使维度相同。多数场景推荐新建 Elasticsearch Index 或 Milvus/Chroma Collection；若产品支持用独立向量字段和明确路由完全隔离，也必须保证查询不会混搜。推荐流程：

~~~text
建立 kb-v2（记录新 embedding_version）
  → 从事实源回填全部文档
  → 双写新旧索引
  → 离线 Golden Set 对比
  → 影子流量 + 线上指标
  → 原子切换 Alias / 应用配置
  → 保留回滚窗口
  → 停止旧写入并清理 kb-v1
~~~

若维度变化，很多数据库也无法原地修改向量字段 Mapping/Schema。新建数据空间通常是更安全的迁移方式。

## 26. 更新、删除与一致性

建议把向量索引同步设计成至少一次投递 + 幂等消费：

1. 业务事务写事实源，同时写 Outbox 事件；
2. 索引器读取事件，按稳定 ID `upsert` 或 `delete`；
3. 记录 `doc_version`，旧事件不能覆盖新版本；
4. 删除传播前可先写 tombstone/状态过滤，阻止旧内容被召回；
5. 对账任务比较事实源版本、索引数量与抽样内容；
6. 备份中的合规删除要按组织的数据保留策略单独处理。

不要承诺跨业务数据库、消息系统和向量库的严格 Exactly Once。工程目标通常是：事件可重试、写入幂等、版本单调、结果可对账、失败可重放。

## 27. 多租户与安全

- 租户 ID 来自认证 Token/会话，不来自 LLM；
- 检索适配器强制拼接 ACL，调用者不能关闭；
- BM25、Dense、多模态等所有召回分支执行同一权限策略；
- 日志避免记录完整敏感原文和向量；
- 被召回文档是**不可信数据**，可能包含 Prompt Injection，LLM 不能因文档指令而绕过工具权限；
- 缓存 Key 必须包含租户、角色、查询版本、Embedding 版本与过滤条件；
- 用专门的跨租户负样本做自动化测试，目标泄漏数为 0。

## 28. 生产监控看什么

- 请求：QPS、P50/P95/P99、超时、错误码、限流；
- 检索：候选数、过滤后数量、空结果率、分数/距离分布、重复文档率；
- 索引：向量数、Segment/Shard 状态、建索引积压、刷新/压实、写入可见延迟；
- 资源：CPU、内存、磁盘、缓存命中、网络、对象存储延迟；
- 质量：按查询类型分桶的 Recall@K、MRR、nDCG、引用点击、人工差评；
- 数据：Embedding/分块版本分布、旧版本残留、事实源与索引对账差异。

只看平均延迟会掩盖高选择性过滤、冷缓存和大租户导致的尾延迟。

---

# 第八部分：精选面试题——只精讲最值得追问的 13 题

本部分精讲 13 题；下一部分另列 24 道自测题但不展开，共覆盖 37 道题型。

## Q1. 向量数据库与关系数据库有什么区别？

**30 秒回答：**

> 关系数据库擅长精确条件、事务和结构化关系；向量数据库擅长按 Embedding 距离做近邻检索，并提供 ANN 索引、元数据过滤、持久化和分布式服务。向量库不是主业务库的替代品：订单状态查关系库，知识语义召回查向量库。小规模时关系库的向量扩展也可能够用，是否引入专用向量库取决于规模、SLA 和运维成本。

**追问：FAISS 是向量数据库吗？**

FAISS 首先是向量相似度搜索库，提供索引算法；完整向量数据库通常还需要持久化、增删改查、过滤、并发服务、权限、备份和分布式能力。不要因为两者都能搜向量就说完全等价。

## Q2. 为什么不用 BM25，只用向量检索？

正确答案不是二选一：

- BM25 对精确词、错误码、型号和稀有实体敏感；
- Dense 对同义表达、口语描述和跨措辞语义更强；
- 两路先分别召回，再用 RRF 融合，最后用 Cross-Encoder 精排，是稳健起点；
- 是否提升必须按查询类型分桶评估，有些纯精确检索业务未必需要 Dense。

## Q3. Cosine、内积、L2 如何选择？

先看 Embedding 模型训练和官方建议，再决定是否归一化与数据库度量。单位归一化后，Cosine、IP 与 L2 可形成等价排序；未归一化时 IP 会受到向量模长影响。还要说明数据库返回的是 distance 还是 transformed score，阈值要在同一产品、同一度量、同一模型上标定。

## Q4. HNSW 为什么快？怎么调？

> HNSW 建多层近邻图，从稀疏高层做长距离导航，再到底层局部扩展，因此不扫描全部向量。`M` 控制图连接密度，`efConstruction` 控制建图候选范围，查询 `ef`/`ef_search` 控制搜索宽度。提高这些值通常改善召回，但分别增加内存、建索引时间或查询延迟。我会用 FLAT 真值扫描不同参数，选择满足 ANN Recall 目标下 P95 最低的配置。

加分点：高维数据、过滤分布、插入顺序、量化、缓存和硬件都会影响结果，不承诺固定复杂度或通用参数。

## Q5. `top_k` 与 `num_candidates`/`ef` 有什么区别？

- `top_k`/`limit` 是最终想返回多少条；
- `num_candidates` 与 HNSW 的 `ef` 都是各产品中的查询搜索宽度旋钮，但含义和实现不是可互换的同一个参数；IVF 则用 `nprobe` 控制探测桶数；
- 只增大 `top_k` 会把更多低相关结果带给重排器，不一定提高前几名质量；
- 增大候选宽度通常更直接地提高 ANN 召回，但会增加延迟；
- 混合检索还有 RRF 的 `rank_window_size` 和重排候选数，四层数量要协同调优。

## Q6. 元数据过滤为什么会让 ANN 变慢或漏结果？

ANN 图或聚类是按整个向量空间建立的，过滤后允许集合可能非常稀疏。图搜索走到的近邻很多不符合条件，或者预过滤位图与向量遍历组合开销上升。解决思路：

- 高频、强选择性条件使用分区/partition key/合理路由；
- 给标量字段建合适索引；
- 比较标准预过滤、迭代过滤或强过滤后的 FLAT；
- 增大 ANN 候选并评测，不盲调；
- 按真实租户大小和过滤比例压测。

安全条件仍必须前置，不能为了速度取消 ACL。

## Q7. 为什么用 RRF，不直接把 BM25 与 Cosine 分数相加？

BM25 分数无固定上界，向量分数的范围又受模型、归一化和数据库变换影响，直接相加需要持续做归一化与权重标定。RRF 只使用各通道排名，对分数尺度不敏感，是可靠基线。缺点是丢失绝对分数信息，通道权重和窗口仍可能需要业务调优；有足够标注数据时可以评估加权线性融合或 Learning to Rank。

## Q8. Milvus、Elasticsearch、Chroma 怎么选？

不要复读产品清单，按需求回答：

1. 原型、Notebook、低运维：Chroma；
2. 已有 ES，且 BM25、过滤、聚合、Highlight 是核心：Elasticsearch；
3. 大规模向量、独立扩缩容、专用索引/多向量需求明显：Milvus；
4. 三者都要用同一真实数据集比较业务 Recall、ANN Recall、P95/P99、QPS、写入、过滤和总成本；
5. 避免没有收益地同时维护 ES 与向量库，因为双写、删除、版本迁移和对账会增加复杂度。

## Q9. 换 Embedding 模型如何不停机？

创建新索引/Collection，不混写旧空间；从事实源回填；新旧双写；跑离线 Golden Set；用影子流量验证；通过 Alias 或配置原子切换；保留回滚窗口；最后停止旧写入并清理。ID 和元数据里保存 Embedding/Chunker 版本，即使维度不变也不混搜。

## Q10. 如何评估向量检索？

先区分两层：

- 用 FLAT 精确 Top-K 评估 ANN Recall 与延迟，判断索引是否丢近邻；
- 用人工标注相关文档评估业务 Recall@K、MRR、nDCG，判断是否找到答案证据。

然后比较 BM25、Dense、Hybrid、Rerank 的消融实验，并在真实过滤、并发、冷热缓存和更新删除下看 P95/P99。端到端还要看 Faithfulness、引用正确性、无答案拒答和 ACL 泄漏。

## Q11. 用户上传文档后立刻搜不到，怎么排查？

按链路逐层排查：

1. 事实源与 Outbox 是否提交；
2. 索引任务是否成功、ID 是否被旧版本覆盖；
3. Embedding 维度与版本是否正确；
4. Collection/Index 是否加载、ES 是否 refresh、Milvus 一致性是否满足 read-your-writes；
5. `status`、租户、ACL 与版本过滤是否把数据排除；
6. 用 ID `get/query` 验证记录存在，再用该记录自身向量做近邻测试；
7. 比较 FLAT 与 ANN，区分“不可见、过滤错误、索引召回低”。

不要上来就把 `top_k` 调大。

更一般地排查“RAG 回答错”：先人工检查 Top-K 是否包含足以回答问题的证据。没有证据是解析、分块、Embedding、过滤、索引或新鲜度问题；已有正确证据却答错，再排查精排、上下文装配、Prompt 与生成模型忠实度。这样能把检索故障和生成故障分开。

## Q12. 如何防止多租户知识泄漏？

> 租户和角色来自认证上下文，检索适配器强制注入 ACL；所有 BM25/Dense/多模态分支都执行相同过滤；大租户可做物理隔离，中小租户共享 Collection 时用 partition key 与标量过滤；缓存 Key 包含租户与角色；建立跨租户负样本并要求泄漏数为 0。LLM 和检索文档都无权修改租户条件。

## Q13. 设计一个企业级知识库检索系统

可以按下面顺序作答，避免一上来堆产品名：

1. **需求与 SLO**：文档量、向量量、日更新、QPS、P99、Recall@K、租户与权限、数据保留；
2. **事实源与解析**：对象存储/业务库，版面解析、OCR、表格、结构化分块、版本；
3. **索引链路**：Outbox、幂等任务、批量 Embedding、稳定 ID、向量与标量索引、对账；
4. **在线检索**：认证 ACL、Query 规范化、BM25 + Dense、RRF、Rerank、去重与邻块扩展；
5. **生成边界**：引用、无答案拒答、Prompt Injection 防护，实时业务数据回源；
6. **可用性**：超时、降级到 BM25/RRF、备份、恢复、容量和滚动迁移；
7. **评测监控**：Golden Set、ANN/业务指标分层、P99、空召回、泄漏测试、影子流量；
8. **选型**：基于上述负载比较 Chroma、ES、Milvus，而不是先选库再补理由。

---

# 第九部分：只列题，不逐题展开

这些题适合自测，答案都能从前文主线推导：

## 基础与索引

1. 为什么高维空间会出现距离集中，Embedding 质量如何影响近邻？
2. HNSW 为什么内存占用高？删除与大量更新对图索引有什么影响？
3. IVF 的 `nlist` 与 `nprobe` 如何联合调优？
4. PQ、SQ、Int8、Int4、Binary Quantization 的误差从哪里来？
5. 什么情况下过滤后的 FLAT 可能比 ANN 更快？
6. 为什么不能把不同数据库的相似度阈值直接互相复制？

## RAG 与数据工程

7. Chunk 太大、太小分别会怎样？如何用实验选 Chunk Size 与 overlap？
8. 父子块检索、邻块扩展与按文档去重有什么区别？
9. Query Rewrite、HyDE、Multi-Query 何时提升召回，何时增加噪声？
10. 如何处理一个文档多次导入、局部更新和顺序乱到的事件？
11. 如何让删除在索引、缓存、备份和审计链路中都正确传播？
12. 为什么只用 LLM-as-a-Judge 不能替代人工相关性标注？

## 三库专题

13. Milvus growing/sealed Segment 与 compaction 对查询和资源有什么影响？
14. Milvus Strong、Session、Bounded、Eventually 如何映射到实际产品体验？
15. Elasticsearch refresh、segment merge、shard 与 replica 如何影响向量写入和搜索？
16. ES 的 RRF `rank_window_size` 与 kNN `k` 不协调会怎样？
17. Chroma 的内存 Client、PersistentClient 和 HttpClient 分别适合什么生命周期？
18. 为什么从 Chroma 原型迁移到集群产品时仍要重新压测，而不是只替换 URI？

## 生产场景

19. 一千万条 768 维 Float32 向量至少多大？为什么实际内存远高于这个数？
20. 如果 P50 很好但 P99 很差，你会按哪些维度拆指标？
21. 如何做向量库限流、超时、熔断和降级？
22. Reranker 宕机时如何降级且保持结果可解释？
23. 如何做蓝绿索引、影子流量与回滚？
24. 如何证明系统没有跨租户泄漏？

---

# 第十部分：面试前一页速记

~~~text
数据契约：同模型 + 同版本 + 同维度 + 同归一化 + 同距离度量

索引：
FLAT = 精确基线
HNSW = 图；M / efConstruction / ef
IVF = 聚类桶；nlist / nprobe
量化 = 内存换精度，扩大候选 + 重打分补偿

检索：
可信 ACL pre-filter
BM25 + Dense → RRF → Rerank → Top 5～10

评测：
ANN Recall@K 对比 FLAT
业务 Recall@K / MRR / nDCG 对比人工相关文档
线上看 P95/P99、QPS、空召回、写入可见、ACL 泄漏

选型：
Chroma = 快速原型/轻量本地
ES = 搜索一体化、BM25/过滤/聚合
Milvus = 独立大规模向量基础设施

迁移：
新空间 → 回填 → 双写 → 离线/影子评测 → 原子切换 → 回滚窗口
~~~

# 附录：主要检索依据

### Milvus 官方资料

- [Milvus Overview](https://milvus.io/docs/overview.md)
- [Milvus Index Explained](https://milvus.io/docs/index-explained.md)
- [Milvus HNSW](https://milvus.io/docs/hnsw.md)
- [Milvus Filtered Search](https://milvus.io/docs/filtered-search.md)
- [Milvus Multi-Vector Hybrid Search](https://milvus.io/docs/multi-vector-search.md)
- [Milvus Consistency](https://milvus.io/docs/consistency.md)
- [Milvus Architecture Overview](https://milvus.io/docs/architecture_overview.md)
- [Milvus 3.0 Release Notes](https://milvus.io/docs/release_notes.md)
- [Milvus Lite 与限制](https://milvus.io/docs/milvus_lite.md)
- [安装匹配 Milvus 3.0 的 PyMilvus 3.0.1](https://milvus.io/docs/install-pymilvus.md)
- [PyMilvus 3.0 create_collection API](https://milvus.io/api-reference/pymilvus/v3.0.x/MilvusClient/Collections/create_collection.md)

### Elasticsearch 官方资料

- [Elasticsearch Release Notes](https://www.elastic.co/docs/release-notes/elasticsearch)
- [Elasticsearch dense_vector](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/dense-vector)
- [Elasticsearch kNN Query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-knn-query)
- [Elasticsearch Vector Search](https://www.elastic.co/docs/solutions/search/vector)
- [Elasticsearch Hybrid Search](https://www.elastic.co/docs/solutions/search/hybrid-search)
- [Elasticsearch Retriever API](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/retrievers)
- [Elasticsearch RRF](https://www.elastic.co/docs/reference/elasticsearch/rest-apis/reciprocal-rank-fusion)

### Chroma 官方资料

- [Chroma 1.5.9 Package Release](https://pypi.org/project/chromadb/)
- [Chroma Getting Started](https://docs.trychroma.com/docs/overview/getting-started)
- [Chroma Clients](https://docs.trychroma.com/docs/run-chroma/clients)
- [Chroma Architecture Overview](https://docs.trychroma.com/reference/architecture/overview)
- [Chroma Manage Collections](https://docs.trychroma.com/docs/collections/manage-collections)
- [Chroma Configure Collections](https://docs.trychroma.com/docs/collections/configure)
- [Chroma Query and Get](https://docs.trychroma.com/docs/querying-collections/query-and-get)
- [Chroma Python Collection API](https://docs.trychroma.com/reference/python/collection)
- [Chroma Cloud Search API Overview](https://docs.trychroma.com/cloud/search-api/overview)
- [Chroma Cloud Hybrid Search](https://docs.trychroma.com/cloud/search-api/hybrid-search)

### 面试题型样本

- [AI 后端、LLM 与向量库](https://www.nowcoder.com/discuss/861205328803762176)
- [AI 应用开发高频题](https://www.nowcoder.com/feed/main/detail/2b1964d4df704bd08cd00373c3cf0161)
- [RAG 各模块优化策略](https://www.nowcoder.com/discuss/1611939)
- [RAG 项目生产化追问](https://www.nowcoder.com/discuss/863430474180333568)
- [真实面经：向量库、分块与方案取舍](https://www.nowcoder.com/discuss/comment/13692054)

> 说明：面经用于识别常见题型，不作为技术事实来源；技术结论与 API 以对应产品官方文档和项目实测为准。厂商文档中的规模建议与性能数字也不能替代自己的基准测试。
