# Elasticsearch 深度课程 · 总目录

> 11 篇中文深度课程，从 Lucene 倒排索引到生产集群运维
> 每篇约 10000-15000 字，含底层结构、查询 DSL、向量检索、生产陷阱、练习题
> 适合从中级到高级搜索 / 数据 / 后端工程师
>
> **📅 内容基准：Elasticsearch 9.x**（2025 GA，AGPLv3 + ELv2 + SSPL 三选一）+ **OpenSearch 3.x**（AWS / LF 主导的 Apache 2.0 fork）
> 章节同时适用两者；许可证 / 治理 / 兼容性差异在 E11 拆解

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| E01 | [精通倒排索引与 Lucene](./01-精通-倒排索引-与-Lucene.md) | ⭐⭐⭐⭐⭐ | FST / posting list / doc values / stored fields |
| E02 | [精通 Segment 与合并策略](./02-精通-Segment-与合并.md) | ⭐⭐⭐⭐ | 段不变 / refresh / flush / force_merge / tiered |
| E03 | [精通集群拓扑与节点角色](./03-精通-集群拓扑.md) | ⭐⭐⭐⭐ | master / data_hot/warm/cold/frozen / coordinating |
| E04 | [精通分片、副本与路由](./04-精通-分片与路由.md) | ⭐⭐⭐⭐⭐ | primary/replica / routing / allocation awareness |
| E05 | [精通 Mapping 与 Analyzer](./05-精通-Mapping-与-Analyzer.md) | ⭐⭐⭐⭐ | dynamic / runtime field / 中文分词 IK / normalizer |
| E06 | [精通 Query DSL 与 Aggregation](./06-精通-Query-DSL.md) | ⭐⭐⭐⭐⭐ | bool / multi_match / agg / runtime field / ESQL |
| E07 | [精通评分 BM25 与 Reranking](./07-精通-BM25-与-Reranking.md) | ⭐⭐⭐⭐ | TF-IDF / BM25 / function_score / LTR |
| E08 | [精通向量检索与 Hybrid Search](./08-精通-向量检索.md) | ⭐⭐⭐⭐⭐ | dense_vector / HNSW / kNN / RAG / hybrid |
| E09 | [精通 ES 写入流程与近实时](./09-精通-写入与近实时.md) | ⭐⭐⭐⭐ | translog / refresh / flush / indexing pressure |
| E10 | [精通 ES 性能调优](./10-精通-性能调优.md) | ⭐⭐⭐⭐⭐ | heap / shard sizing / hot threads / slow log |
| E11 | [Elasticsearch vs OpenSearch](./11-ES-vs-OpenSearch.md) | ⭐⭐⭐ | license / 兼容性 / 迁移 |

---

## 🗺️ 按模块组织

### 🟢 模块一：内核（E01-E02）
> Lucene 倒排索引、segment 不变性、合并——所有性能与可靠性问题的物理根因。

### 🔵 模块二：分布式（E03-E04）
> 节点角色与数据分布。理解了这两章才能做容量规划。

### 🟡 模块三：建模与查询（E05-E07）
> Mapping、DSL、Aggregation、评分——业务侧最常用的部分。

### 🔴 模块四：现代化（E08-E09）
> 向量检索（2024-2026 的最大革命）+ 写入路径深度。

### 🟠 模块五：生产（E10-E11）
> 调优与选型。

---

## 🎯 学习路径

### 路径 A：全面进阶（5 周）
按编号顺序通读。每篇配套：起一个 docker es 单节点，按章节跑实验。

### 路径 B：搜索工程师速成（2 周）
**E01 倒排索引** + **E05 Mapping/Analyzer** + **E06 DSL** + **E07 BM25** + **E08 向量** —— 5 篇覆盖建模 + 查询 80%。

### 路径 C：SRE / 运维特化（2 周）
**E02 Segment** + **E03 集群** + **E04 分片** + **E09 写入** + **E10 性能** —— 5 篇覆盖容量、稳定性、故障排查。

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **官方文档**：[elastic.co/guide/](https://www.elastic.co/guide/en/elasticsearch/reference/current/) / [OpenSearch docs](https://opensearch.org/docs/)
- **Lucene 文档**：[lucene.apache.org/core/](https://lucene.apache.org/core/) （ES/OS 共同的底层引擎）
- **源码**：[github.com/elastic/elasticsearch](https://github.com/elastic/elasticsearch) / [github.com/opensearch-project/OpenSearch](https://github.com/opensearch-project/OpenSearch)

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 集群健康 | `GET _cluster/health` |
| 节点信息 | `GET _cat/nodes?v` / `GET _nodes/stats` |
| 索引列表 | `GET _cat/indices?v&s=index` |
| 分片分布 | `GET _cat/shards?v&s=index` |
| 节点磁盘 | `GET _cat/allocation?v` |
| 分配解释 | `GET _cluster/allocation/explain` |
| 当前任务 | `GET _cat/tasks?v` |
| 热点线程 | `GET _nodes/hot_threads` |
| Pending task | `GET _cluster/pending_tasks` |
| 慢查询日志 | `PUT logs/_settings {"index.search.slowlog.threshold.query.warn":"5s"}` |
| 索引内存使用 | `GET _stats/segments?human` |
| 强制 merge | `POST logs-2024.05/_forcemerge?max_num_segments=1` |
| 关闭索引 | `POST old_idx/_close` / `POST old_idx/_open` |
| 模板预览 | `POST _index_template/_simulate_index/logs-2026.05.13` |
| 字段映射 | `GET myindex/_mapping/field/timestamp` |
| Explain 评分 | `POST myindex/_explain/{doc_id} { "query": ... }` |
| EQL / ESQL | `POST _query { "query": "FROM logs \| WHERE level=='error' \| STATS count=COUNT(*) BY host" }` |
| 向量 kNN | `POST myindex/_search {"knn":{"field":"vec","query_vector":[...],"k":10,"num_candidates":100}}` |

---

## ✅ 完读检查清单

- [ ] 解释 FST + posting list 的二级查找过程
- [ ] 解释 segment 不变性如何同时带来高吞吐写入和昂贵的合并
- [ ] 设计一个 1TB/天日志场景的 ILM（hot/warm/cold/frozen）配置
- [ ] 区分 routing / shard / index / alias 在路由层的作用
- [ ] 写一个 BM25 评分能解释的 multi_match 查询
- [ ] 实现 hybrid search：BM25 + dense_vector + RRF 融合
- [ ] 设计一次 force_merge 的执行计划（什么时段、几个 segment、内存预算）
- [ ] 给 P99 写入延迟飙升的集群列出 5 条诊断假设
- [ ] 解释 ES 9 的 Lookup join（tech preview）/ Lucene 10 / logsdb 默认 等新特性（ES|QL 已在 8.14 GA）
- [ ] 选 ES 9 还是 OpenSearch 3 并说出理由

---

> 🔁 反馈：本地 `docker run elasticsearch:9` 跑一节点，配合 Kibana 试每个 DSL
