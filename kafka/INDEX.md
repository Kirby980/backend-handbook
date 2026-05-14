# Kafka 深度课程 · 总目录

> 12 篇中文深度课程，从 KRaft 元数据到 Streams 编程
> 每篇约 10000-15000 字，含底层协议、producer/consumer 内部、性能调优、生产陷阱、练习题
> 适合从中级到高级数据 / 后端 / 平台工程师
>
> **📅 内容基准：Apache Kafka 4.0**（2025 GA，默认 KRaft、ZooKeeper 完全移除）+ **Confluent 8.x**
> 现代替代品（Redpanda / Pulsar / WarpStream）在 K12 比对
>
> ⚠️ Kafka 3.3-3.9 还支持 ZK 模式但已 deprecated。新建集群一律 KRaft；老集群升级路径在 K02 介绍

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| K01 | [精通 Topic / Partition / Offset 模型](./01-精通-Topic-Partition-Offset.md) | ⭐⭐⭐⭐ | commit log / partition leadership / offset 语义 |
| K02 | [精通 KRaft 元数据管理](./02-精通-KRaft.md) | ⭐⭐⭐⭐⭐ | controller quorum / metadata log / snapshot / 迁移 |
| K03 | [精通 Producer 内部](./03-精通-Producer.md) | ⭐⭐⭐⭐ | batching / acks / idempotent / sticky partitioner |
| K04 | [精通 Consumer 与 KIP-848 新协议](./04-精通-Consumer-KIP-848.md) | ⭐⭐⭐⭐⭐ | rebalance / sticky / cooperative / broker-side |
| K05 | [精通复制：ISR、HW、LEO](./05-精通-复制-ISR.md) | ⭐⭐⭐⭐⭐ | HW vs LEO / ISR 缩张 / unclean leader |
| K06 | [精通存储与 Segment 文件](./06-精通-存储与-Segment.md) | ⭐⭐⭐⭐ | log segment / index / retention / log compaction / tiered storage |
| K07 | [精通精确一次与事务](./07-精通-精确一次-事务.md) | ⭐⭐⭐⭐⭐ | EOS / transactional producer / isolation_level / 2PC |
| K08 | [精通 Kafka 性能调优](./08-精通-性能调优.md) | ⭐⭐⭐⭐ | 压缩 / 批量 / 零拷贝 / sendfile / Linux 调优 |
| K09 | [精通 Kafka Streams](./09-精通-Kafka-Streams.md) | ⭐⭐⭐⭐⭐ | 状态存储 / 窗口 / interactive query / exactly-once |
| K10 | [精通 Kafka Connect 与 Schema Registry](./10-精通-Connect-Schema-Registry.md) | ⭐⭐⭐⭐ | source/sink / Avro/Protobuf/JSON Schema / 兼容性 |
| K11 | [精通 Kafka 安全：ACL、SASL、mTLS](./11-精通-Kafka-安全.md) | ⭐⭐⭐⭐ | SCRAM / OAuth / ACL / KRaft ACL 迁移 |
| K12 | [精通 Kafka 生产运维 + 现代替代品](./12-精通-生产运维.md) | ⭐⭐⭐⭐ | JMX / lag / Redpanda / Pulsar / WarpStream |

---

## 🗺️ 按模块组织

### 🟢 模块一：核心模型（K01-K02）
> commit log 抽象 + KRaft 元数据。这两章是 Kafka 4.0 时代的入口。

### 🔵 模块二：编程接口（K03-K04）
> Producer / Consumer 的内部实现。KIP-848 新消费组协议彻底改变了 rebalance 行为。

### 🟡 模块三：可靠性（K05、K07）
> ISR 复制机制 + 精确一次语义——分布式日志的两个最难问题。

### 🔴 模块四：性能（K06、K08）
> 存储布局与调优。

### 🟠 模块五：生态（K09-K12）
> Streams、Connect、安全、运维。

---

## 🎯 学习路径

### 路径 A：全面进阶（5 周）
按编号顺序通读。每篇配套：本地 docker 起 3 broker KRaft 集群，跑实验。

### 路径 B：后端开发速成（2 周）
**K01 模型** + **K03 Producer** + **K04 Consumer** + **K07 精确一次** + **K08 调优**——5 篇覆盖应用侧 80%。

### 路径 C：平台 / SRE 特化（2 周）
**K02 KRaft** + **K05 ISR** + **K06 存储** + **K11 安全** + **K12 运维**——5 篇覆盖运维。

---

## 📋 配套资源

- **路线图**：[ROADMAP.md](./ROADMAP.md)
- **测验题**：[QUIZ.md](./QUIZ.md)
- **官方文档**：[kafka.apache.org/documentation/](https://kafka.apache.org/documentation/)
- **KIP 列表**：[Kafka Improvement Proposals](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)
- **源码**：[github.com/apache/kafka](https://github.com/apache/kafka)（重点 `core/`、`clients/`、`metadata/`）
- **Confluent 文档**：[docs.confluent.io](https://docs.confluent.io)（业界最权威实战来源）

---

## 🛠️ 工具速查

| 任务 | 命令 |
|---|---|
| 起 KRaft 单节点 | `kafka-storage.sh format --config server.properties --cluster-id $(kafka-storage.sh random-uuid)` + `kafka-server-start.sh server.properties` |
| 创建 topic | `kafka-topics.sh --bootstrap-server :9092 --create --topic foo --partitions 6 --replication-factor 3` |
| 查看 topic | `kafka-topics.sh --bootstrap-server :9092 --describe --topic foo` |
| 列 consumer group | `kafka-consumer-groups.sh --bootstrap-server :9092 --list` |
| 查 lag | `kafka-consumer-groups.sh --bootstrap-server :9092 --describe --group my-group` |
| 重置 offset | `kafka-consumer-groups.sh --bootstrap-server :9092 --group g --reset-offsets --to-earliest --topic foo --execute` |
| 控制台消费 | `kafka-console-consumer.sh --bootstrap-server :9092 --topic foo --from-beginning` |
| 控制台生产 | `kafka-console-producer.sh --bootstrap-server :9092 --topic foo` |
| ACL 列表 | `kafka-acls.sh --bootstrap-server :9092 --list` |
| 节点状态 | `kafka-metadata-quorum.sh --bootstrap-server :9092 describe --status` |
| 复制状态 | `kafka-metadata-quorum.sh --bootstrap-server :9092 describe --replication` |
| Segment dump | `kafka-dump-log.sh --files /var/lib/kafka/foo-0/00000000000000000000.log --print-data-log` |
| Performance test | `kafka-producer-perf-test.sh --topic foo --num-records 1000000 --record-size 1024 --throughput -1` |
| JMX 监控 | `JMX_PORT=9999 kafka-server-start.sh ...` + `kafka_exporter` |

---

## ✅ 完读检查清单

- [ ] 解释 commit log 抽象比传统 MQ 模型强在哪
- [ ] 区分 producer 三种 acks（0 / 1 / all）的可靠性与吞吐权衡
- [ ] 解释 idempotent producer + transactional producer 的实现原理
- [ ] 画出 ISR / HW / LEO 三者关系及 unclean leader election 的代价
- [ ] 设计一次从 ZK 模式到 KRaft 模式的零停机迁移
- [ ] 解释 KIP-848 新协议消除 stop-the-world rebalance 的关键技术
- [ ] 设计 log compaction 的 key 策略（什么数据适合 compact 而非 retention）
- [ ] 给一个百万 QPS 写入场景做调优清单（producer / broker / OS）
- [ ] 选 Kafka 4 / Redpanda / Pulsar / WarpStream 并说出理由
- [ ] 实现 Kafka Streams 的 exactly-once 应用 + Interactive Query

---

> 🔁 反馈：本地 3 broker KRaft 集群最直观；docker compose 起一组试
