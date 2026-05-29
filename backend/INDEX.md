# Backend 路线图深度课程 · 总目录

> 基于 [roadmap.sh/backend](https://roadmap.sh/backend) 生成的 25 篇中文深度课程
> 每篇约 10000-15000 字，含底层原理、代码示例、生产实践、陷阱清单、练习题
> 适合后端工程师从入门到高级的系统进阶
>
> **📅 内容基准：2026 年 5 月**——TLS 1.3 + post-quantum (X25519MLKEM768) 默认、HTTP/3 主流化、PostgreSQL 18、Redis 8.0 / Valkey、Kafka 4.0 (KRaft)、Istio Ambient GA、K8s Gateway API GA、Passkeys 主流、OAuth 2.1 + DPoP、OpenTelemetry 事实标准、Prometheus 3.0。
> 每章末尾的"2026 现状"小节标注最新变化。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| B01 | [精通互联网工作原理](./B01-精通互联网工作原理.md) | ⭐⭐⭐ | DNS / TCP/IP / HTTP/1.1/2/3 / QUIC |
| B02 | [精通 HTTP 语义](./B02-精通-HTTP-语义.md) | ⭐⭐⭐ | methods / status / headers / caching / CORS / cookies |
| B03 | [精通 TLS 与 HTTPS](./B03-精通-TLS-与-HTTPS.md) | ⭐⭐⭐⭐ | 握手 / 证书 / Let's Encrypt / SNI / HSTS / mTLS |
| B04 | [精通 REST API 设计](./B04-精通-REST-API-设计.md) | ⭐⭐⭐ | 资源 / 版本 / 分页 / Idempotency / HATEOAS |
| B05 | [精通 gRPC 生产实践](./B05-精通-gRPC-生产实践.md) | ⭐⭐⭐⭐ | 服务发现 / LB / 重试 / 熔断 / 流控 |
| B06 | [精通 GraphQL](./B06-精通-GraphQL.md) | ⭐⭐⭐⭐ | schema / resolver / N+1 / DataLoader / federation |
| B07 | [精通 OpenAPI 契约](./B07-精通-OpenAPI-契约.md) | ⭐⭐⭐ | spec / code gen / 契约测试 / 版本管理 |
| B08 | [精通数据库索引](./B08-精通数据库索引.md) | ⭐⭐⭐⭐ | B-Tree / 复合索引 / 覆盖索引 / EXPLAIN |
| B09 | [精通数据库事务与 ACID](./B09-精通数据库事务与-ACID.md) | ⭐⭐⭐⭐ | 隔离级别 / MVCC / 锁 / Saga |
| B10 | [精通规范化与反规范化](./B10-精通规范化与反规范化.md) | ⭐⭐⭐ | 1NF/2NF/3NF / 反规范化时机 |
| B11 | [精通分片与分区](./B11-精通分片与分区.md) | ⭐⭐⭐⭐⭐ | range/hash/consistent / 跨片 query / resharding |
| B12 | [精通数据库复制与 CAP](./B12-精通数据库复制与-CAP.md) | ⭐⭐⭐⭐⭐ | 主从 / Raft / CAP / 一致性级别 |
| B13 | [精通 N+1 问题](./B13-精通-N加1-问题.md) | ⭐⭐⭐ | 检测 / DataLoader / eager loading |
| B14 | [精通 NoSQL 选型](./B14-精通-NoSQL-选型.md) | ⭐⭐⭐⭐ | KV / 文档 / 列族 / 图 / TS / 搜索 |
| B15 | [精通缓存策略](./B15-精通缓存策略.md) | ⭐⭐⭐⭐ | cache-aside / 雪崩 / 穿透 / 击穿 |
| B16 | [精通 Redis](./B16-精通-Redis.md) | ⭐⭐⭐⭐ | 数据结构 / 持久化 / cluster / 锁 |
| B17 | [精通消息队列](./B17-精通消息队列.md) | ⭐⭐⭐⭐⭐ | Kafka / RabbitMQ / NATS / 幂等消费 |
| B18 | [精通微服务架构](./B18-精通微服务架构.md) | ⭐⭐⭐⭐ | 边界 / 服务发现 / API gateway / Saga |
| B19 | [精通 12-Factor App](./B19-精通-12-Factor-App.md) | ⭐⭐⭐ | 配置 / 无状态 / disposability |
| B20 | [精通韧性模式](./B20-精通韧性模式.md) | ⭐⭐⭐⭐ | timeout / retry / circuit breaker / bulkhead |
| B21 | [精通背压与限流](./B21-精通背压与限流.md) | ⭐⭐⭐⭐ | token bucket / sliding window / 背压 |
| B22 | [精通后端认证](./B22-精通后端认证.md) | ⭐⭐⭐⭐ | session / JWT / OAuth 2.0 / OIDC / MFA |
| B23 | [精通 OWASP Top 10](./B23-精通-OWASP-Top-10.md) | ⭐⭐⭐⭐ | 注入 / IDOR / SSRF / XSS / CSRF |
| B24 | [精通可观测性](./B24-精通可观测性.md) | ⭐⭐⭐⭐ | logs / metrics / traces / OTel / SLO |
| B25 | [精通 Web 服务器与反向代理](./B25-精通-Web-服务器与反向代理.md) | ⭐⭐⭐⭐ | Nginx / Caddy / HAProxy / Envoy / Traefik |

---

## 🗺️ 按模块组织

### 🟢 模块一：网络基础（B01-B03）

> 后端工程师的"母语"——HTTP、TLS、协议栈。

- **B01 互联网原理**：DNS、TCP/UDP、HTTP/1.1/2/3、QUIC、网络工具
- **B02 HTTP 语义**：methods、status、headers、缓存、CORS、cookie
- **B03 TLS 与 HTTPS**：握手、证书链、Let's Encrypt、SNI、HSTS、mTLS

### 🔵 模块二：API 设计（B04-B07）

> 服务对外的契约——四种主流风格。

- **B04 REST**：资源建模、版本、分页、Idempotency-Key、HATEOAS
- **B05 gRPC**：服务发现、负载均衡、熔断、重试、流控、Health
- **B06 GraphQL**：schema、resolver、N+1、DataLoader、federation
- **B07 OpenAPI**：spec、代码生成、契约测试、版本管理

### 🟡 模块三：数据库（B08-B14）

> 数据是后端的根基——关系 + NoSQL 全谱。

- **B08 索引**：B-Tree、复合索引、覆盖索引、EXPLAIN、partial index
- **B09 事务 ACID**：隔离级别、MVCC、锁、死锁、Saga
- **B10 规范化**：1NF-3NF、反规范化时机、物化视图
- **B11 分片**：range/hash/consistent、跨片 query、resharding
- **B12 复制 CAP**：主从、Raft、CAP 三选二、一致性级别
- **B13 N+1**：检测、DataLoader、eager loading、典型模式
- **B14 NoSQL**：KV/文档/列族/图/时间序列/搜索的选型

### 🔴 模块四：数据加速（B15-B17）

> 让数据访问更快——缓存与消息。

- **B15 缓存策略**：cache-aside、雪崩/穿透/击穿、一致性
- **B16 Redis**：数据结构、持久化、集群、典型用法、分布式锁
- **B17 消息队列**：Kafka vs RabbitMQ vs NATS、幂等消费、Outbox

### 🟣 模块五：架构与韧性（B18-B21）

> 让系统扛得住——分布式架构与容错。

- **B18 微服务架构**：何时拆、按业务能力、API gateway、Saga
- **B19 12-Factor App**：云原生应用 12 条原则
- **B20 韧性模式**：timeout、retry、circuit breaker、bulkhead、降级
- **B21 背压与限流**：token bucket、sliding window、自适应并发

### 🟠 模块六：安全与运维（B22-B25）

> 让系统安全又能看见——authn/authz、observability、infra。

- **B22 认证**：session、JWT、OAuth 2.0、OIDC、MFA、密码 hash
- **B23 OWASP Top 10**：注入、IDOR、SSRF、加密失败等十大风险
- **B24 可观测性**：logs/metrics/traces、OTel、SLO、错误预算
- **B25 Web 服务器**：Nginx、Caddy、HAProxy、Envoy、Traefik

---

## 🎯 学习路径建议

### 路径 A：从零到生产（半年）

按编号顺序，每周 2-3 篇。每篇配套：
1. 通读概念
2. 跑代码示例 / 配置
3. 做练习题
4. 把生产实践和陷阱清单挂在工位

### 路径 B：API 工程师特化（2 个月）

要做好的 API 设计师：
- **B01-B03 网络基础**（必备底盘）
- **B04 REST + B07 OpenAPI**（核心契约）
- **B05 gRPC + B06 GraphQL**（替代方案）
- **B22 认证**（API 的门面）
- **B20-B21 韧性 + 限流**（生产级 API）

### 路径 C：数据库专精（2 个月）

数据库性能 + 一致性专家：
- **B08 索引**（基础）
- **B09 事务 + B12 复制**（一致性）
- **B10-B11 规范化 + 分片**（设计）
- **B13 N+1 + B15 缓存**（加速）
- **B14 NoSQL**（替代选项）

### 路径 D：架构师视角（2 个月）

系统设计能力：
- **B18 微服务 + B19 12-Factor**（架构基础）
- **B20-B21 韧性 + 限流**
- **B17 消息队列 + B16 Redis**
- **B11-B12 分片 + 复制**（数据架构）
- **B24 可观测性**（运维基础）

### 路径 E：安全审视（1 个月）

安全合规：
- **B03 TLS**
- **B22 认证**
- **B23 OWASP Top 10**
- **B24 可观测性**（log + audit）
- **B25 Web 服务器**（WAF、限流前置）

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：见 [QUIZ.md](./QUIZ.md)
- **官方 roadmap**：[roadmap.sh/backend](https://roadmap.sh/backend)
- **Go 路线图配套课程**：`../golang/` 目录

---

## 🛠️ 工具速查

| 任务 | 工具 / 命令 |
|---|---|
| DNS 解析 | `dig`、`nslookup`、`host` |
| 路由追踪 | `traceroute`、`mtr` |
| TCP 端口 | `nc -zv host port`、`ss -tlnp` |
| HTTP 调试 | `curl -v`、`HTTPie`、`Postman` |
| 抓包 | `tcpdump`、`wireshark` |
| TLS 调试 | `openssl s_client -connect host:443` |
| TLS 评分 | [SSL Labs](https://www.ssllabs.com/ssltest/) |
| 证书申请 | `certbot`、`acme.sh`、`cert-manager` |
| DB 执行计划 | `EXPLAIN ANALYZE`（PG）/ `EXPLAIN`（MySQL） |
| DB 慢日志 | `pg_stat_statements` / MySQL slow log |
| Redis | `redis-cli`、`INFO`、`SLOWLOG` |
| Kafka | `kafka-console-consumer`、`kafka-consumer-groups` |
| 限流测试 | `wrk`、`vegeta`、`hey` |
| Web 服务器 | Nginx `-t` 校验、`-s reload` 重载 |
| 容器扫描 | `trivy`、`grype` |
| 依赖扫描 | `dependabot`、`snyk`、`govulncheck` |
| OpenAPI 校验 | `oasdiff`、`spectral` |
| 可观测性 | OpenTelemetry、Prometheus、Grafana、Loki、Tempo |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 解释从浏览器输入 URL 到看到页面的完整网络流程
- [ ] 选择正确的 HTTP status code（404 vs 410 vs 422 等）
- [ ] 给一个域名配置完整生产 HTTPS（cert + cipher + HSTS）
- [ ] 设计 REST API 含分页、版本、错误统一
- [ ] 设计复合索引并解读 EXPLAIN 输出
- [ ] 解释为何并发转账可能 lost update + 给三种解
- [ ] 决定 PostgreSQL / MongoDB / Cassandra / Redis 选型
- [ ] 设计缓存层并避免雪崩 / 穿透 / 击穿
- [ ] 说出 OAuth 2.0 Authorization Code flow + PKCE
- [ ] 给一个 service 加 timeout + retry + circuit breaker
- [ ] 解释 token bucket vs sliding window vs leaky bucket
- [ ] 检查 web app 是否中 OWASP Top 10 的每一条
- [ ] 设计 SLO + 错误预算 + 告警
- [ ] 写出生产级 Nginx 反代配置

---

## 🔁 与 Go 路线图的关系

Backend 路线图是语言无关的——讲概念、设计、协议。Go 路线图（`../golang/`）讲特定语言实现。

互补搭配建议：

| Backend 章节 | Go 配套 |
|---|---|
| B01 互联网 | G11 GMP / G14 context |
| B02 HTTP | G27 net/http |
| B05 gRPC | G28 gRPC |
| B08-B09 数据库 | G29 数据库访问 |
| B15-B16 缓存 | G13 sync.Pool |
| B17 消息队列 | G11-G15 并发 |
| B18 微服务 | G27-G28 net/http + gRPC |
| B19 12-Factor | G14 context |
| B20 韧性 | G14 context |
| B22 认证 | G27 net/http middleware |
| B24 可观测性 | G22 pprof + G30 slog |

Backend 读"该做什么"；Go 读"在 Go 里怎么做"。

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **B01 互联网** | HTTP/3 约 21% 请求流量（Cloudflare Radar，仍略低于 HTTP/1.x ~28%）；IPv6 ~46% 全球；QUIC v2 / Multipath / WebTransport 落地 |
| **B03 TLS** | **X25519MLKEM768 后量子混合** Chrome/Firefox/Cloudflare 默认；ECH 在 CF + Firefox/Chrome 主流；Apple iOS 26+ 跟进 |
| **B07 OpenAPI** | OpenAPI 3.1 已对齐 JSON Schema 2020-12；AsyncAPI 3.0 用于事件驱动 |
| **B12 复制 / CAP** | **PostgreSQL 18** (2025-09)：异步 I/O / UUIDv7 / virtual generated columns / OAuth；分布式 SQL：CockroachDB / YugabyteDB / TiDB / Spanner |
| **B16 Redis** | **Redis 8.0** (2025-05) 回归开源 (AGPLv3)；**Valkey** Linux Foundation BSD fork 已成 AWS / GCP 默认 |
| **B17 消息队列** | **Kafka 4.0** (2025) 默认 KRaft、移除 ZooKeeper；**KIP-848** 新消费组协议（无 stop-the-world rebalance）；Redpanda / Pulsar / WarpStream 现代替代 |
| **B18 微服务** | **Istio Ambient Mesh GA** (2024-11) 无 sidecar；Cilium eBPF Service Mesh；**K8s Gateway API GA** 替代 Ingress |
| **B22 认证** | **Passkeys 主流化**（5 亿活跃，Apple/Google/Microsoft 全栈支持）；**OAuth 2.1** PKCE 强制 + Implicit/ROPC 移除；**DPoP** (RFC 9449) sender-constrained token |
| **B23 OWASP** | OWASP Top 10 2021 仍是当前正式版；**OWASP API Security Top 10 2023** API-first 必读；2025 草案讨论中 |
| **B24 可观测性** | **OpenTelemetry CNCF Graduated**——logs/metrics/traces 三柱协议事实标准；**Prometheus 3.0** native histograms / UTF-8 / OTLP 接收 |
| **B25 Web 服务器** | **NGINX One** (2024-09) 统一管理；NGINX Plus R33+ 要 JWT license；ModSecurity EOL（用 Coraza） |

---

> 🔁 反馈：发现错误或建议改进
