# Backend 路线图 · 知识检测测验

> 每章 5 道题，共 125 题。答案在最末尾"参考答案"章节。
> 题目类型：单选（含真假）、概念辨析、设计选择。
> 建议：先盖住答案独立完成；> 80% 正确视为掌握。

---

## B01 — 互联网工作原理

**1.1** DNS TTL=3600，全球客户端何时能看到 A 记录的新 IP？
- A. 立即
- B. 大多在 3600s 内，但部分缓存可能更久
- C. 24 小时后保证
- D. 取决于浏览器

**1.2** TCP 三次握手需要多少 RTT 完成连接建立？
- A. 0.5
- B. 1
- C. 1.5
- D. 2

**1.3** HTTP/2 在 TCP 上仍有"head-of-line blocking"，原因是？
- A. HTTP/2 协议设计缺陷
- B. TCP 保证字节序——丢一个包后续 stream 都要等
- C. HTTP/2 仅一个 stream
- D. 与 HTTP/1.1 相同问题

**1.4** UDP 替代 TCP 的典型场景**不**包括？
- A. DNS 查询
- B. 视频通话
- C. 银行转账
- D. 游戏实时同步

**1.5** 服务报"TIME_WAIT 大量堆积"，最可能原因？
- A. 客户端长连接
- B. 客户端短连接 + 高 QPS
- C. 防火墙问题
- D. DNS 解析慢

---

## B02 — HTTP 语义

**2.1** POST 创建资源成功，最合适的 status code 是？
- A. 200
- B. 201
- C. 204
- D. 202

**2.2** `Cache-Control: no-cache` 与 `Cache-Control: no-store` 的关键差别是？
- A. 没有差别
- B. no-cache 允许存储但每次 revalidate；no-store 完全不存
- C. no-store 仅浏览器；no-cache 仅 CDN
- D. no-cache 已废弃

**2.3** API 返回不同语言版本，cache 应额外加什么 header？
- A. Set-Cookie
- B. Content-Language
- C. Vary: Accept-Language
- D. ETag

**2.4** 401 与 403 的区别？
- A. 没区别
- B. 401 是"我不知道你是谁"；403 是"我知道你但你不能"
- C. 401 用于浏览器；403 用于 API
- D. 401 用于 GET；403 用于 POST

**2.5** SameSite=Strict 解决什么问题？
- A. XSS
- B. CSRF
- C. SQL 注入
- D. SSRF

---

## B03 — TLS 与 HTTPS

**3.1** TLS 1.3 相比 1.2 的关键改进**不**包括？
- A. 握手 RTT 减为 1
- B. 支持 0-RTT
- C. 移除不安全 cipher
- D. 改用 UDP

**3.2** 浏览器显示"证书不受信任"但证书本身没问题，最可能原因？
- A. 证书过期
- B. 服务器没发送中间证书
- C. 客户端时间错
- D. CN 不匹配

**3.3** `*.example.com` 通配符证书**不**保护哪一个？
- A. `a.example.com`
- B. `example.com`（自己）
- C. `b.example.com`
- D. `c.example.com`

**3.4** HSTS preload list 撤回为什么难？
- A. 需要 CA 签名
- B. 编译进浏览器二进制，更新滚动数月
- C. 法律限制
- D. 没法撤回

**3.5** mTLS 与 OAuth 2.0 token 各保护什么？
- A. 都保护用户身份
- B. mTLS 传输层机器身份；OAuth 应用层用户身份
- C. mTLS 加密；OAuth 不加密
- D. 它们互斥

---

## B04 — REST API 设计

**4.1** PUT 与 PATCH 的关键差别？
- A. 没有差别
- B. PUT 全量替换；PATCH 部分更新
- C. PUT 创建；PATCH 修改
- D. PATCH 已废弃

**4.2** 创建资源 API 应返回什么 header 指向新资源？
- A. Refer
- B. Location
- C. Resource-URI
- D. Self

**4.3** 64 位整数 ID 在 JSON 中应该怎么序列化？
- A. number
- B. string（防 JS 精度问题）
- C. BigInt
- D. 任意

**4.4** Idempotency-Key 解决什么？
- A. 让 GET 加速
- B. 让 POST 重试安全（不重复扣款）
- C. 防 SQL 注入
- D. JWT 验证

**4.5** API 删除一个已经删除的资源，返回什么 status 最合适？
- A. 500 服务器错
- B. 204 No Content 或 404 Not Found
- C. 409 Conflict
- D. 200 OK

---

## B05 — gRPC 生产实践

**5.1** Kubernetes 中 gRPC 流量不均匀，最常见原因？
- A. K8s 配置错
- B. ClusterIP service 是 L4 LB，HTTP/2 长连接绑定一个 pod
- C. gRPC 不支持负载均衡
- D. 防火墙规则

**5.2** retry 的"指数退避 + 抖动"中"抖动"是为了？
- A. 兼容旧客户端
- B. 防止"thundering herd"——大量 client 同时重试
- C. 优化网络
- D. 减少日志

**5.3** Circuit Breaker 的三个状态**不**包括？
- A. Closed
- B. Open
- C. Half-Open
- D. Pending

**5.4** `MaxConnectionAge` 配置的作用？
- A. 单次 RPC 超时
- B. 强制连接周期性重建，让 LB 重新分配
- C. TLS 握手超时
- D. 心跳间隔

**5.5** gRPC 错误应该用哪个返回？
- A. errors.New
- B. status.Error(codes.X, msg)
- C. panic
- D. fmt.Errorf

---

## B06 — GraphQL

**6.1** GraphQL N+1 问题最经典的解？
- A. 改 SQL
- B. DataLoader 批量合并
- C. 加更多 resolver
- D. 加缓存

**6.2** GraphQL 错误响应 HTTP status 通常是？
- A. 4xx 或 5xx
- B. 永远 200，错误在 `errors` 数组里
- C. 500
- D. 取决于错误类型

**6.3** Mutation 应该返回什么类型？
- A. 直接返回实体
- B. payload type（包含 entity + errors + 扩展字段）
- C. 仅返回成功 / 失败
- D. 任意

**6.4** GraphQL 防恶意嵌套深度的方式是？
- A. 限流
- B. depth limit 中间件
- C. 缓存
- D. 没法防

**6.5** subscription 一般底层基于？
- A. HTTP polling
- B. WebSocket
- C. gRPC
- D. SSE

---

## B07 — OpenAPI 契约

**7.1** OpenAPI 3.1 与 3.0 的主要变化？
- A. 改名 Swagger
- B. 完整对齐 JSON Schema 2020-12
- C. 删除 servers
- D. 简化大量功能

**7.2** schema 中 `additionalProperties: false` 的作用？
- A. 优化性能
- B. 严格模式——拒绝未定义字段
- C. 必填所有字段
- D. 启用 strict typing

**7.3** CI 中检测"API 破坏性变更"的工具是？
- A. swagger-codegen
- B. oasdiff
- C. spectral
- D. redoc

**7.4** `oneOf` 与 `allOf` 的差别？
- A. 没差
- B. oneOf 匹配恰好一个；allOf 全部匹配（用于继承）
- C. oneOf 必填；allOf 可选
- D. oneOf 是 3.1 新增

**7.5** 修改字段类型属于？
- A. 兼容性变更
- B. 破坏性变更（要 SemVer MAJOR）
- C. 仅在大版本
- D. 取决于具体类型

---

## B08 — 数据库索引

**8.1** 复合索引 `(a, b, c)` 对 `WHERE b = ?` 有效吗？
- A. 有效
- B. 部分有效
- C. 无效（违反最左前缀）
- D. 取决于值

**8.2** 覆盖索引指？
- A. 索引覆盖所有列
- B. 索引包含查询所需的所有列，不需回表
- C. 单一 PRIMARY KEY 索引
- D. 复合索引

**8.3** 函数包裹列（`WHERE LOWER(name) = ?`）通常会？
- A. 加快查询
- B. 索引失效
- C. 自动优化
- D. 报错

**8.4** PostgreSQL `EXPLAIN ANALYZE` 显示 `Seq Scan` 意味着？
- A. 用了正确索引
- B. 全表扫描（可能没匹配索引或 planner 觉得这样更快）
- C. 排序操作
- D. 临时表

**8.5** partial index 的典型用途？
- A. 备份
- B. 仅对满足条件的行建索引（如 `WHERE deleted_at IS NULL`）
- C. 多列索引
- D. 唯一约束

---

## B09 — 数据库事务与 ACID

**9.1** ACID 中"I"指？
- A. Integrity
- B. Indivisible
- C. Isolation
- D. Idempotency

**9.2** MySQL InnoDB 默认隔离级别？
- A. READ UNCOMMITTED
- B. READ COMMITTED
- C. REPEATABLE READ
- D. SERIALIZABLE

**9.3** 经典"lost update"问题的最佳解？
- A. 增加事务隔离级别
- B. 原子 UPDATE（`SET balance = balance - 50 WHERE balance >= 50`）
- C. 加更多重试
- D. 完全避免并发

**9.4** Saga 模式相比 2PC 的优势？
- A. 强一致性
- B. 更适合微服务、避免协调者单点 + 阻塞
- C. 性能更慢
- D. 简单

**9.5** PostgreSQL 中"长事务"为何阻止 VACUUM？
- A. 锁表
- B. 持有旧 snapshot，旧版本仍需保留
- C. 占内存
- D. CPU 高

---

## B10 — 规范化与反规范化

**10.1** 3NF 主要消除什么？
- A. 冗余字段
- B. 传递依赖（非主键依赖其他非主键）
- C. NULL 值
- D. JOIN

**10.2** 订单表存 `customer_name_at_purchase` 字段属于？
- A. 违反规范化
- B. 故意反规范化（快照字段，审计需要）
- C. 临时存储
- D. 缓存

**10.3** MongoDB 嵌入文档的优势主要是？
- A. 减少 JOIN，一次读完
- B. 节省空间
- C. 强一致性
- D. 自动分片

**10.4** PostgreSQL `jsonb` 字段的合理用途？
- A. 所有业务字段
- B. 半结构化 metadata、可演进字段
- C. 主键
- D. 关联字段

**10.5** 物化视图（Materialized View）解决什么？
- A. 替代普通 VIEW
- B. 预计算聚合，避免重复昂贵 query
- C. 备份
- D. 复制

---

## B11 — 分片与分区

**11.1** 何时**不**该分片？
- A. 表 100 万行 + QPS 100
- B. 单表 5TB
- C. 写 QPS 远超单机
- D. 团队需要全球低延迟

**11.2** 分片键选择最重要原则？
- A. 字段名短
- B. 高频查询的等值条件
- C. 索引列
- D. 主键

**11.3** 一致性哈希相比 hash mod N 的优势？
- A. 更均匀
- B. 加节点时数据移动量小（约 1/N 而非全部）
- C. 性能更好
- D. 实现简单

**11.4** 全局唯一 ID 在分布式系统中应该用？
- A. AUTO_INCREMENT
- B. UUID / Snowflake / UUIDv7
- C. 时间戳
- D. 随机数

**11.5** "热点 key"（Twitter 名人）的常见缓解？
- A. 增加分片数
- B. 单独分片 / random suffix 散开 / 应用层聚合
- C. 加索引
- D. 删除该 key

---

## B12 — 复制与 CAP

**12.1** CAP 定理中网络分区时，系统选择？
- A. C 和 A 都能保
- B. P 可以避免
- C. 必须在 C 和 A 之间二选一
- D. 三者都可放弃

**12.2** Raft 共识算法 3 节点集群容忍多少节点宕？
- A. 0
- B. 1
- C. 2
- D. 3

**12.3** 用户写完立刻读副本但看不到，原因？
- A. 副本损坏
- B. 异步复制延迟
- C. 网络问题
- D. 客户端 cache

**12.4** PostgreSQL "synchronous_commit = on" + "synchronous_standby_names = 'rep1'" 是？
- A. 异步复制
- B. 同步复制（等 rep1 ack）
- C. 关闭复制
- D. 仅本地

**12.5** "脑裂"（split-brain）指？
- A. CPU 过载
- B. 两个节点同时认为自己是 primary
- C. 网络分区
- D. 锁冲突

---

## B13 — N+1 问题

**13.1** N+1 问题最常见出现场景？
- A. 单次 SQL 查询
- B. ORM 懒加载 + 循环访问关联
- C. 索引缺失
- D. 锁冲突

**13.2** 检测 N+1 最有效的工具？
- A. 静态代码分析
- B. APM trace 看 span 树 / 框架级 N+1 检测器
- C. 单元测试
- D. CPU profile

**13.3** GraphQL N+1 的标准解？
- A. 取消 GraphQL
- B. DataLoader 每请求批量
- C. 加 query 复杂度限制
- D. 缓存

**13.4** "JOIN 一次性查"对 N+1 是否一定最优？
- A. 是
- B. 否——一对多笛卡尔积膨胀时，prefetch（两次 query）更好
- C. 取决于 DB
- D. 不能替代 ORM 关键字

**13.5** 何时 N+1 不值得修？
- A. 永远不修
- B. N 很小（< 5-10）且不会增长
- C. 关联在另一服务
- D. 用 NoSQL

---

## B14 — NoSQL 选型

**14.1** Redis 适合做主数据库吗？
- A. 是
- B. 不——持久化保证弱，量级受 RAM 限制
- C. 仅小规模
- D. 仅 Linux

**14.2** Cassandra 适合什么场景？
- A. 复杂 ad-hoc 查询
- B. 高吞吐写入 + 已知 access pattern + 时间序列
- C. 强一致跨表事务
- D. 图查询

**14.3** "图查询"（X 度好友）用哪类 DB？
- A. PostgreSQL
- B. MongoDB
- C. Neo4j
- D. Redis

**14.4** 全文搜索"包含 X + 价格区间 + 高亮"应该用？
- A. PostgreSQL LIKE
- B. Elasticsearch
- C. Redis
- D. Cassandra

**14.5** 新项目选 DB 优先考虑？
- A. PostgreSQL（默认）
- B. 最新潮 NoSQL
- C. 公司其他团队用什么
- D. 朋友推荐

---

## B15 — 缓存策略

**15.1** "缓存雪崩"指？
- A. 缓存丢失
- B. 大量 cache 同时过期 → 流量打 DB
- C. cache 数据错
- D. cache 服务挂

**15.2** 缓存击穿的解？
- A. 关闭缓存
- B. 互斥锁 / 永不过期 + 后台刷
- C. 增加 TTL
- D. 增加副本

**15.3** "缓存穿透"防御**不**包括？
- A. 缓存空值
- B. Bloom filter
- C. 限流
- D. 增大 cache

**15.4** 写 DB 后是该删 cache 还是更新 cache？
- A. 更新
- B. 删除 cache 更稳健（下次读 miss 自动 reload）
- C. 都行
- D. 不动

**15.5** "Refresh-Ahead" 模式适合？
- A. 写多读少
- B. 热点数据保持永远新鲜
- C. 弱一致场景
- D. 临时数据

---

## B16 — Redis

**16.1** Redis 单线程为何仍 100K+ QPS？
- A. 数据结构特殊
- B. 内存操作 + epoll I/O 复用 + 简单 op
- C. 多核优化
- D. C 语言

**16.2** Sorted Set 不适合做？
- A. 排行榜
- B. 延迟队列
- C. 限流时间窗
- D. 关系型 JOIN

**16.3** 生产 Redis **应该避免**用？
- A. SET / GET
- B. KEYS *（阻塞整库扫描）
- C. INCR
- D. ZADD

**16.4** Redis 持久化推荐配置？
- A. 关闭持久化
- B. RDB only
- C. AOF everysec + RDB（混合）
- D. AOF always

**16.5** 分布式锁实现的关键点？
- A. 用 INCR
- B. SET key val NX EX + Lua 释放（防误删他人锁）
- C. 用 DEL
- D. 用 GETSET

---

## B17 — 消息队列

**17.1** Kafka 与 RabbitMQ 的核心差别？
- A. Kafka 不支持持久化
- B. Kafka 主打高吞吐 + 可重放；RabbitMQ 主打灵活路由
- C. RabbitMQ 更快
- D. 没差别

**17.2** "exactly-once" 跨系统通常如何实现？
- A. 用 Kafka 事务
- B. 不存在；at-least-once + 消费幂等是实际方案
- C. 2PC
- D. 增加 retry

**17.3** Kafka partition 数应该？
- A. 越多越好
- B. 一开始足够多（事后改难，影响顺序）
- C. 1 个
- D. 等于 broker 数

**17.4** Outbox 模式解决什么？
- A. 消息持久化
- B. 业务变更 + 消息发布的原子保证
- C. 重复消息
- D. 限流

**17.5** 消费者 lag 持续增长意味着？
- A. broker 性能问题
- B. 消费速率 < 生产速率，需扩容消费者或加 partition
- C. 网络问题
- D. partition 数过多

---

## B18 — 微服务架构

**18.1** 拆微服务的最重要触发条件？
- A. 代码量大
- B. 团队规模 + 部署节奏冲突
- C. 性能不够
- D. 想用新技术

**18.2** "分布式单体"（Distributed Monolith）反模式指？
- A. 单一二进制
- B. N 个服务但部署必须一起、强同步耦合
- C. 用 K8s 部署
- D. 用 Docker

**18.3** 每个微服务应该？
- A. 共享 DB
- B. 独占自己的 DB
- C. 直连其他服务的 DB
- D. 无所谓

**18.4** API Gateway 应**避免**做？
- A. TLS 终止
- B. 业务逻辑
- C. 限流
- D. 路由

**18.5** 跨微服务事务最实用方案？
- A. 2PC
- B. Saga + 补偿
- C. 跨 DB 事务
- D. 不解决

---

## B19 — 12-Factor App

**19.1** Factor III "Config" 主张？
- A. 配置文件随代码 commit
- B. 配置放环境变量
- C. 数据库存配置
- D. 硬编码

**19.2** Factor VI "Processes" 要求？
- A. 多线程
- B. 进程无状态
- C. 单进程
- D. 守护进程

**19.3** Factor XI "Logs" 要求应用？
- A. 写本地文件 + 轮转
- B. 直接写 stdout/stderr
- C. 发送 syslog
- D. 调日志 API

**19.4** Factor IX "Disposability" 关键？
- A. 容器化
- B. 快启动 + 优雅 SIGTERM 关闭
- C. 多副本
- D. 无 bug

**19.5** Factor V "Build/Release/Run" 三阶段要严格？
- A. 加快开发
- B. 让 release artifact 不可变 + 易回滚
- C. 减少 build 次数
- D. 简化 CI

---

## B20 — 韧性模式

**20.1** retry 安全的前提是？
- A. 网络稳定
- B. 操作幂等（或带 Idempotency-Key）
- C. 重试次数少
- D. 用 GET 方法

**20.2** Circuit Breaker 主要解决？
- A. 偶发故障
- B. 持续故障 → 减少对下游压力让其恢复
- C. 限流
- D. 限并发

**20.3** Bulkhead 模式核心思想？
- A. 加副本
- B. 资源隔离防止故障扩散
- C. 重试
- D. 降级

**20.4** 调用外部 API **必须**设置？
- A. 缓存
- B. timeout
- C. retry
- D. 日志

**20.5** Chaos Engineering 的目的？
- A. 故意破坏生产
- B. 主动注入故障验证系统韧性
- C. 减少测试
- D. 防止攻击

---

## B21 — 背压与限流

**21.1** Token Bucket 与 Leaky Bucket 关键差别？
- A. 没差别
- B. Token Bucket 允许 burst；Leaky Bucket 平滑输出
- C. Token Bucket 用 Redis
- D. Leaky Bucket 更快

**21.2** Fixed Window 限流的问题？
- A. 实现复杂
- B. 边界突发（59 秒满 + 下秒又满 = 2 秒内双倍）
- C. 内存大
- D. 不准

**21.3** API 返回 429 应该带什么 header？
- A. Cache-Control
- B. Retry-After
- C. Location
- D. Vary

**21.4** "背压"（backpressure）指？
- A. 下游主动慢下来
- B. 下游处理不过来 → 信号传给上游让其慢下来
- C. 上游加缓冲
- D. 全部 drop

**21.5** 自适应并发控制（AIMD）相对固定阈值的优势？
- A. 简单
- B. 自动适应负载和硬件变化
- C. 性能更高
- D. 实现简单

---

## B22 — 后端认证

**22.1** 密码应该用什么 hash？
- A. MD5
- B. SHA-256
- C. bcrypt / argon2id（慢 hash）
- D. base64

**22.2** JWT 不能即时吊销，最常见缓解？
- A. 加密 token
- B. 短 exp（15min） + refresh token（可吊销）
- C. 改 secret
- D. 加 IP 白名单

**22.3** OAuth 2.0 公开 client（SPA、移动）必须用？
- A. client_secret
- B. PKCE
- C. Basic auth
- D. mTLS

**22.4** access token 存哪里最安全？
- A. localStorage
- B. HttpOnly + Secure + SameSite cookie
- C. URL 参数
- D. 普通 cookie

**22.5** MFA 中**最不**推荐的因素？
- A. TOTP (Google Authenticator)
- B. WebAuthn (硬件 key)
- C. SMS（SIM swap 风险）
- D. push notification

---

## B23 — OWASP Top 10

**23.1** SQL 注入的根本防御？
- A. 输入过滤
- B. 参数化查询
- C. 加 WAF
- D. 关防火墙

**23.2** A01 Broken Access Control 中"IDOR"指？
- A. 注入
- B. 不安全直接对象引用 — `/api/orders/43` 返回别人的订单
- C. 跨站脚本
- D. 重定向

**23.3** SSRF 让攻击者能做什么？
- A. 偷 cookie
- B. 让服务端访问内网 / 元数据，泄漏凭据
- C. 改密码
- D. 删数据

**23.4** XSS 防御中最有效的纵深？
- A. 关 JS
- B. CSP header + 输出 escape + HttpOnly cookie
- C. 限流
- D. mTLS

**23.5** "依赖有已知 CVE"属于 OWASP 的？
- A. A01
- B. A06 Vulnerable Components
- C. A03
- D. A10

---

## B24 — 可观测性

**24.1** 可观测性三柱？
- A. CPU / 内存 / 磁盘
- B. Logs / Metrics / Traces
- C. Alert / Dashboard / Log
- D. Request / Response / Error

**24.2** Prometheus metric 标签（label）**不应该**用？
- A. method（GET/POST）
- B. status（200/404/500）
- C. user_id（每用户独立时序）
- D. endpoint 名

**24.3** SLO 错误预算的作用？
- A. 决定服务质量
- B. 平衡发布速度与稳定性——预算未用完可激进发布
- C. 计算成本
- D. 报告 KPI

**24.4** Tail sampling（trace）相比 head sampling？
- A. 性能更好
- B. 能选择性保留错误 / 慢的 trace
- C. 简单
- D. 没差别

**24.5** 应用日志推荐输出到？
- A. 本地文件 + logrotate
- B. stdout（容器收集）
- C. 数据库
- D. 邮件

---

## B25 — Web 服务器与反向代理

**25.1** Caddy 相对 Nginx 的关键优势？
- A. 性能
- B. 自动 HTTPS（零配置 ACME）
- C. 配置语法
- D. 历史悠久

**25.2** 何时选 Envoy？
- A. 个人博客
- B. Service Mesh / 动态路由 / gRPC first-class 需求
- C. 静态网站
- D. 最简单场景

**25.3** Nginx 与上游应用之间应该？
- A. 每请求新连接
- B. 启用 keep-alive 减少建连开销
- C. 用 SSE
- D. 用 UDP

**25.4** SSE（Server-Sent Events）在 Nginx 后面要？
- A. 加 gzip
- B. proxy_buffering off（关闭代理缓冲）
- C. 不能用 Nginx
- D. 加 TLS

**25.5** 信任 `X-Forwarded-For` 的安全前提？
- A. 永远信任
- B. 只信任自己代理链设置的——直接客户端的可伪造
- C. 不信任
- D. 加 hash 验证

---

## ✅ 参考答案

### B01
1.1 **B**（理论 TTL 内但实际客户端缓存可能更久）
1.2 **C**（SYN, SYN+ACK, ACK = 1.5 RTT 用于建立）
1.3 **B**（TCP 保证字节序）
1.4 **C**（银行转账要可靠传输）
1.5 **B**（短连接 + 高 QPS = TIME_WAIT 累积）

### B02
2.1 **B**（201 Created）
2.2 **B**
2.3 **C**（Vary: Accept-Language）
2.4 **B**
2.5 **B**（CSRF）

### B03
3.1 **D**（仍基于 TCP，不是 UDP；QUIC 才是 UDP）
3.2 **B**（漏发中间证书）
3.3 **B**（通配符不含父域自身；要加 SAN）
3.4 **B**
3.5 **B**

### B04
4.1 **B**
4.2 **B**（Location）
4.3 **B**（JS number 仅 53 位精度）
4.4 **B**
4.5 **B**（DELETE 应幂等）

### B05
5.1 **B**（K8s ClusterIP 是 L4 + HTTP/2 长连接绑定）
5.2 **B**（防 thundering herd）
5.3 **D**（无 Pending；是 Closed/Open/Half-Open）
5.4 **B**
5.5 **B**

### B06
6.1 **B**
6.2 **B**（GraphQL 总是 200）
6.3 **B**
6.4 **B**
6.5 **B**

### B07
7.1 **B**
7.2 **B**
7.3 **B**（oasdiff）
7.4 **B**
7.5 **B**

### B08
8.1 **C**（违反最左前缀，跳过 a）
8.2 **B**
8.3 **B**（除非建表达式索引 `(LOWER(name))`）
8.4 **B**
8.5 **B**

### B09
9.1 **C**（Isolation）
9.2 **C**（REPEATABLE READ）
9.3 **B**（原子 UPDATE）
9.4 **B**
9.5 **B**

### B10
10.1 **B**
10.2 **B**（审计快照）
10.3 **A**
10.4 **B**
10.5 **B**

### B11
11.1 **A**（小规模不需要分片）
11.2 **B**
11.3 **B**
11.4 **B**
11.5 **B**

### B12
12.1 **C**
12.2 **B**（majority = 2 of 3，容忍 1 挂）
12.3 **B**（复制延迟）
12.4 **B**
12.5 **B**

### B13
13.1 **B**
13.2 **B**
13.3 **B**
13.4 **B**（一对多大时 JOIN 会笛卡尔膨胀）
13.5 **B**

### B14
14.1 **B**
14.2 **B**
14.3 **C**
14.4 **B**
14.5 **A**（PostgreSQL 默认；90% 项目够）

### B15
15.1 **B**
15.2 **B**
15.3 **D**（增大不能防穿透）
15.4 **B**
15.5 **B**

### B16
16.1 **B**
16.2 **D**（Redis 没原生 JOIN）
16.3 **B**（KEYS 阻塞）
16.4 **C**（AOF everysec + RDB 混合）
16.5 **B**

### B17
17.1 **B**
17.2 **B**
17.3 **B**
17.4 **B**
17.5 **B**

### B18
18.1 **B**
18.2 **B**
18.3 **B**
18.4 **B**
18.5 **B**

### B19
19.1 **B**
19.2 **B**
19.3 **B**
19.4 **B**
19.5 **B**

### B20
20.1 **B**
20.2 **B**
20.3 **B**
20.4 **B**
20.5 **B**

### B21
21.1 **B**
21.2 **B**
21.3 **B**
21.4 **B**
21.5 **B**

### B22
22.1 **C**（bcrypt / argon2id 慢 hash）
22.2 **B**
22.3 **B**（PKCE）
22.4 **B**（HttpOnly cookie）
22.5 **C**（SMS 易被 SIM swap）

### B23
23.1 **B**（参数化查询）
23.2 **B**
23.3 **B**
23.4 **B**
23.5 **B**

### B24
24.1 **B**
24.2 **C**（user_id 高 cardinality 会爆 Prometheus）
24.3 **B**
24.4 **B**
24.5 **B**

### B25
25.1 **B**
25.2 **B**
25.3 **B**（keep-alive 必开）
25.4 **B**（buffering 会缓冲流，破坏实时性）
25.5 **B**

---

## 📊 评分标准

| 分数 | 评价 |
|---|---|
| 113+/125（>90%） | 🏆 精通—— Backend 高级工程师候选 |
| 88-112（70-89%） | 🥇 熟练—— 能胜任生产开发 |
| 63-87（50-69%） | 🥈 入门—— 重点补足薄弱章节 |
| <63（<50%） | 📖 建议重读对应章节 + 动手实践 |

按模块统计错题：

- **模块 1（网络）错 >2 道**：B01-B03 重读
- **模块 2（API）错 >3 道**：B04-B07 重读
- **模块 3（数据库）错 >5 道**：B08-B14 是大头，强烈建议重做并 hands-on
- **模块 4（数据加速）错 >3 道**：B15-B17 实战补
- **模块 5（架构）错 >3 道**：分布式系统经验补
- **模块 6（安全运维）错 >3 道**：安全审计意识不能少

---

## 🔁 与 Go 路线图测验组合

55 篇课程合起来涉及的测验题：
- Go 路线图：150 题
- Backend 路线图：125 题
- 合计 **275 题**

建议：
1. 先完整 Go QUIZ 一遍
2. 再 Backend QUIZ
3. 重错题
4. 一个月后再做一遍——长期记忆检验

---

## 🎯 重点章节速攻清单

如果只想攻破 5 章 + 25 题（每章 5 题）：

| 章节 | 为什么必读 |
|---|---|
| B01 互联网 | 后端"母语" |
| B08 数据库索引 | 性能第一杀手 |
| B15 缓存策略 | 90% 项目都用 |
| B22 认证 | 安全门面 |
| B24 可观测性 | 没有就是黑盒 |

25 题答对 22+ 即视为掌握 backend 主干。

---

## 🆕 2026 版补充题（关键技术现状）

> 检测你是否掌握 INDEX 末尾"2026 关键变化速查"。

**Q1**：2026 年新建公网 HTTPS 服务，密钥协商应该选什么？为什么？
<details><summary>答</summary>**X25519MLKEM768**（hybrid post-quantum）。原因：SNDL 攻击——对手现在抓密文存档，等量子计算机出现回头解密。混合方案让 X25519 和 ML-KEM-768 任一不破整体就安全。Chrome ≥ 131 / Firefox ≥ 132 / Cloudflare 已默认。</details>

**Q2**：Kafka 4.0 默认运行模式是什么？还需要 ZooKeeper 吗？老集群怎么升？
<details><summary>答</summary>**KRaft 模式**（KIP-833 自 3.3 production-ready，4.0 强制）。完全不需要 ZooKeeper。老 ZK 集群必须先迁 KRaft，再升 4.0；3.3 之前版本要先升 3.9.x 再迁。</details>

**Q3**：什么是 KIP-848？为什么生产关心？
<details><summary>答</summary>新一代 consumer rebalance 协议。旧协议任一 consumer 加入/离开整个 group "stop the world" 全部 revoke 重分；KIP-848 由 broker 增量分配 partition，**不再 stop-the-world**。客户端 `group.protocol=consumer` 启用。注意：启用后只能降级到 3.4.1+。</details>

**Q4**：Redis 与 Valkey 在 2026 年应该怎么选？
<details><summary>答</summary>纯 cache/queue 选 **Valkey**（BSD-3、LF 治理、AWS/GCP 默认、与 Redis API 完全兼容）。需要原生 vector / JSON / 时序模块且能接受 AGPLv3 选 **Redis 8**。两者命令行为基本一致，代码层面通常无感切换。</details>

**Q5**：PostgreSQL 18 (2025-09) 的 "asynchronous I/O" 主要解决了什么？怎么开 io_uring？
<details><summary>答</summary>顺序扫描 / bitmap heap scan / vacuum 不再一个 read 等一个；异步发起多个 I/O。`io_method = io_uring`（仅 Linux 5.1+），`io_method = worker`（默认，跨平台），`io_method = sync`（旧行为）。某些读密集场景达 3x 加速。</details>

**Q6**：什么是 Passkey？跟 WebAuthn / FIDO2 是什么关系？
<details><summary>答</summary>Passkey = 建立在 **FIDO2 = WebAuthn (W3C) + CTAP (FIDO Alliance)** 之上的、**可同步 / 可跨设备使用**的 credential。区别于"硬件 YubiKey"那种 device-bound passkey，syncable passkey 通过 iCloud Keychain / Google Password Manager 在设备间同步。后端实现 = WebAuthn registration + assertion。</details>

**Q7**：OAuth 2.1 相比 OAuth 2.0 移除了哪些 grant？强制了什么？
<details><summary>答</summary>**移除**：Implicit grant、Resource Owner Password Credentials (ROPC) grant。**强制**：所有 Authorization Code flow 必须 PKCE（不只 SPA/mobile）；redirect URI 严格匹配；refresh token 必须 rotation 或 sender-constrained。</details>

**Q8**：DPoP（RFC 9449）的核心思想是什么？解决什么问题？
<details><summary>答</summary>把 access token 绑定到 client 持有的 keypair。请求时除 `Authorization: DPoP <token>` 外还要带 `DPoP: <signed proof JWT>`。资源服务器验证 proof 公钥与 token 内 jkt 匹配。**token 被偷也用不了**——攻击者没 private key。比 mTLS 简单（不要 PKI），比 bearer 安全。</details>

**Q9**：OWASP API Security Top 10 2023 中新增了哪三项？
<details><summary>答</summary>**API6** Unrestricted Access to Sensitive Business Flows（业务流被自动化滥用）、**API7** Server-Side Request Forgery (SSRF)（云时代的高危）、**API10** Unsafe Consumption of APIs（你调第三方 API 时盲目信任返回值）。</details>

**Q10**：Istio Ambient Mesh 与传统 sidecar 模式相比，最大差异在哪？
<details><summary>答</summary>**无 sidecar**。L4 用每节点共享的 ztunnel（DaemonSet），L7 按 namespace 部署 waypoint Envoy。pod 数量减 40-50%、无 sidecar 启动竞态、控制平面升级不重启业务。Istio 1.24（2024-11）GA。</details>

按答案对照打分。错 ≥ 4 题：回去看 INDEX 末尾"2026 关键变化速查"+ 对应章节末尾的"2026 现状"小节。

---

> 🔁 配套：[INDEX.md](./INDEX.md) 总目录 / [ROADMAP.md](./ROADMAP.md) 路线图
