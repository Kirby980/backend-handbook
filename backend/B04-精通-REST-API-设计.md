# 精通 REST API 设计

> 课程编号：B04
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — REST API
> 难度：⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：REST 不是 CRUD

```
GET    /users         → 列表
GET    /users/42      → 单个
POST   /users         → 创建
PUT    /users/42      → 更新
DELETE /users/42      → 删除
```

很多人把 REST 等同于"4 个 HTTP 动词配 URL"——这是简化版。REST 是 Roy Fielding 2000 年博士论文提出的架构风格，强调资源、状态转移、统一接口、超媒体。这一章既讲实用 RESTful API 设计（90% 场景），也讲 REST 真正含义（HATEOAS 等）。

---

## 第一章：资源建模

### 1.1 URI 应表达资源，不是动作

```
❌ /createUser
❌ /getUserList
❌ /deleteOrder/42
✅ /users      (POST 创建, GET 列表)
✅ /users/42   (GET, PUT, DELETE)
```

URL 是名词；HTTP method 是动词。

### 1.2 命名约定

- **复数名词**：`/users` 不是 `/user`（一致性）
- **小写 + 连字符**：`/order-items` 不是 `/orderItems` 或 `/order_items`
- **嵌套表达从属**：`/users/42/orders`
- **过深嵌套避免**：超过 2 层考虑独立顶级资源
- **ID 用稳定标识符**：UUID 比自增 int 更好（隐私 + 跨服务）

### 1.3 资源类型

- **集合（collection）**：`/users`
- **单项（item）**：`/users/42`
- **子集合**：`/users/42/orders`
- **关联（association）**：`/users/42/manager`
- **动作（action）**：`/users/42:lock`（少用，纯 REST 应 PUT/PATCH 状态字段）

### 1.4 子资源 vs 查询参数

```
GET /users/42/orders          ← 强依赖：order 只在 user 上下文有意义
GET /orders?user_id=42        ← 弱依赖：可独立访问
```

订单往往可独立查询 → 推荐 query 参数。强依赖（如 user → preferences）用嵌套。

---

## 第二章：HTTP Method 映射

### 2.1 CRUD 与方法

| 操作 | 方法 | URL | 成功响应 |
|---|---|---|---|
| 创建 | POST | `/users` | 201 + Location |
| 读取列表 | GET | `/users` | 200 |
| 读取单个 | GET | `/users/42` | 200 |
| 全量更新 | PUT | `/users/42` | 200 / 204 |
| 部分更新 | PATCH | `/users/42` | 200 / 204 |
| 删除 | DELETE | `/users/42` | 204 |

### 2.2 POST 的多重身份

- **创建**：`POST /users {...}` → 201
- **非幂等动作**：`POST /orders/42/cancel`
- **复杂查询**：`POST /search` 带大 body（GET URL 太长）

### 2.3 安全与幂等再强调

- GET 必须**安全**（不改状态）—— 不要把"逻辑删除"塞 GET
- PUT、DELETE 必须**幂等**—— 重复调用同结果
- POST 不要求幂等，但**支持幂等 key** 让 client 重试安全：

```http
POST /payments
Idempotency-Key: 7a8b9c10-uuid
```

Server 记录 key，相同 key 第二次请求返回上次的结果。Stripe、Square 等都用这模式。

---

## 第三章：版本化

### 3.1 为何

API 一旦发布，破坏性改动会让现有 client 崩溃。版本化让旧 client 继续用旧版，新 client 用新版。

### 3.2 三种主流方案

**A. URL 路径**
```
GET /v1/users/42
GET /v2/users/42
```
最直观，浏览器、curl 友好。**最常用**。

**B. Header**
```
GET /users/42
Accept: application/vnd.example.v2+json
```
不污染 URL。Stripe、GitHub 部分用。但难调试。

**C. Query 参数**
```
GET /users/42?api-version=2
```
不推荐——容易被 cache 忽略。

### 3.3 版本演进策略

- 加字段：兼容，无需版本
- 改字段含义、删字段：破坏，要新版
- 行为变化（默认值、限制）：根据严重性

发布 v2 时 v1 至少保留 6-12 月，给 client 迁移时间。GitHub 把弃用日期写在 HTTP header：

```
Sunset: Sat, 31 Dec 2026 23:59:59 GMT
Deprecation: true
```

---

## 第四章：分页

### 4.1 三种主流方式

**A. Offset/Limit**
```
GET /users?offset=20&limit=10
```
简单，但深分页慢（DB 仍要扫前 N 行），且数据变化时漂移。

**B. Page/Size**
```
GET /users?page=3&size=10
```
本质同 offset，对用户更直观。

**C. Cursor**
```
GET /users?cursor=abc123&limit=10
→
{
  "data": [...],
  "next_cursor": "def456"
}
```
基于"上次最后一条"的不透明令牌。**稳定、快、推荐**。Twitter、Facebook 都用。

### 4.2 响应结构

```json
{
  "data": [ ... ],
  "pagination": {
    "next_cursor": "def456",
    "has_more": true,
    "total": 12345        // 可选；昂贵的话不提供
  }
}
```

或 RFC 5988 Link header：

```
Link: <...?cursor=def>; rel="next", <...>; rel="prev"
```

### 4.3 选择

- 简单列表 + 用户跳页 → page/size
- API + 数据频繁变化 → cursor
- 内部工具 + 必须知 total → offset

---

## 第五章：排序、过滤、字段选择

### 5.1 排序

```
GET /users?sort=name              # 升序
GET /users?sort=-created_at       # 降序
GET /users?sort=name,-created_at  # 多字段
```

`-` 前缀表降序。

### 5.2 过滤

```
GET /users?status=active
GET /users?age_gt=18
GET /users?created_after=2026-01-01
GET /products?price[gte]=100&price[lte]=500
```

约定一种格式即可。复杂查询用 POST body 是合理的折中（如 elasticsearch、algolia）。

### 5.3 字段选择

```
GET /users/42?fields=id,name,email
```

按需返回，减小 payload。GraphQL 在这点上完胜。

### 5.4 嵌套展开

```
GET /users/42?include=orders,manager
→
{
  "id": 42,
  "name": "Alice",
  "orders": [ ... ],
  "manager": { ... }
}
```

避免 client 多次请求（参考 B13 N+1）。

---

## 第六章：错误响应

### 6.1 统一格式

```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User 42 does not exist",
    "details": {
      "user_id": 42
    },
    "request_id": "req-abc-123"
  }
}
```

字段：
- `code`：机器可读的错误码（不变）
- `message`：人类可读
- `details`：额外信息
- `request_id`：便于客服查日志

### 6.2 RFC 7807 Problem Details

标准化错误格式：

```json
{
  "type": "https://example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 400,
  "detail": "Your balance is 50 but you tried to charge 100.",
  "instance": "/transactions/12345"
}
```

Content-Type: `application/problem+json`。

### 6.3 字段验证错

```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Request validation failed",
    "fields": [
      { "field": "email", "code": "INVALID_FORMAT" },
      { "field": "age", "code": "MUST_BE_POSITIVE" }
    ]
  }
}
```

422 Unprocessable Entity。

---

## 第七章：HATEOAS

### 7.1 含义

Hypermedia As The Engine Of Application State——响应中包含**指向下一步操作的链接**：

```json
{
  "id": 42,
  "status": "pending",
  "_links": {
    "self":   { "href": "/orders/42" },
    "cancel": { "href": "/orders/42:cancel", "method": "POST" },
    "ship":   { "href": "/orders/42:ship",   "method": "POST" }
  }
}
```

Client 不需要硬编码 URL，跟着 link 走即可。

### 7.2 实际接受度

理想很美。实际：
- 浏览器 + JS SPA 直接用 URL 模板，HATEOAS 没普及
- 增加 payload 大小
- 改 link 名仍是破坏性变更

**多数实用 REST API 不用 HATEOAS**。但理解概念有助于看到完整 REST 设计。

### 7.3 中间方案

只在状态机式资源里加 links（如订单状态转移）。简单 CRUD 资源不必。

---

## 第八章：典型 API 设计

### 8.1 完整 User 资源

```
GET    /v1/users                    # 列表（cursor 分页）
POST   /v1/users                    # 创建
GET    /v1/users/{id}               # 单个
PATCH  /v1/users/{id}               # 部分更新
DELETE /v1/users/{id}               # 删除（软）

POST   /v1/users/{id}:reset-password
POST   /v1/users/{id}:lock
POST   /v1/users/{id}:unlock

GET    /v1/users/{id}/orders        # 用户的订单
```

### 8.2 软删除

```
DELETE /users/42       → 204 (标记 deleted_at)
GET    /users/42       → 404
GET    /users?include_deleted=true → 仍可见
```

业务"删除"通常不真删 DB 记录，保留审计。

### 8.3 异步操作

```
POST /jobs/export
→ 202 Accepted
   Location: /jobs/export/job-123

GET /jobs/export/job-123
→ 200 { "status": "running", "progress": 0.45 }
GET /jobs/export/job-123
→ 200 { "status": "completed", "result_url": "..." }
```

长任务返回 202，提供轮询 URL（或 webhook）。

---

## 第九章：实用细节

### 9.1 时间格式

ISO 8601 UTC：`2026-05-12T10:30:00Z`。永远不要用 epoch 数字（混乱：秒？毫秒？）或地区时间字符串。

### 9.2 数字 vs 字符串

大整数（如 64 位 ID）：JSON number 在 JS 中精度只有 53 位（safe integer 2^53-1）。**用字符串**：`"id": "9007199254740993"`。

货币用整数（分）或字符串带精度，**不要 float**。

### 9.3 枚举

```json
{ "status": "pending" }     ← 字符串枚举，可扩展
{ "status": 1 }              ← 数字，client 要查表
```

字符串枚举更人友好；server 端用类型保证不出错（如 Go iota + Stringer 或 protobuf enum）。

### 9.4 时区

```json
{ "created_at": "2026-05-12T10:30:00Z" }  ← 总是 UTC + Z
```

Client 显示时按用户时区转换。

---

## 第十章：生产级最佳实践

1. **URL 用复数名词 + 小写连字符**。
2. **版本在 URL 路径**：`/v1/`。
3. **POST 创建必带 Location 头**。
4. **错误统一 schema** + `code` 字段。
5. **支持 Idempotency-Key**：让 client 重试安全。
6. **Cursor 分页**：稳定 + 快。
7. **时间 ISO 8601 UTC**。
8. **大整数 ID 用字符串**：防 JS 精度。
9. **`Sunset` header + 6-12 月迁移期**。
10. **OpenAPI 规范 + 自动生成 SDK / docs**（B07）。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：动词在 URL
```
POST /createUser
```
应 `POST /users`。

### ❌ 陷阱 2：硬编码 ID 大小
JS client 把 64 位 int 当 number → 精度丢失。用字符串。

### ❌ 陷阱 3：错误返回 200
```
{ "status": 200, "error": "...", "data": null }
```
让 client 必须检查 body。直接用 HTTP status code。

### ❌ 陷阱 4：DELETE 不幂等
第二次 DELETE 已删除资源返回 500——应当 204 或 404。

### ❌ 陷阱 5：POST 创建返回 200 而非 201
小事，但 client 库（如 axios）的拦截器可能按 201 判断。

### ❌ 陷阱 6：分页 total 在大表上慢
SELECT COUNT(*) 全表扫描。或省略 total，或单独 endpoint 返回近似值。

### ❌ 陷阱 7：版本随便升
v1 → v2 → v3 短时间内 → client 痛苦。新字段加在 v1，仅破坏性变更升版。

---

## 第十二章：练习题

**练习 1**：设计一个 Comment 资源的 API（创建、列表、点赞、删除）。

**练习 2**：以下 endpoint 哪些违反 REST 风格？
- (a) `GET /listOrders`
- (b) `POST /orders/42/refund`
- (c) `PUT /users/42 { name: "x" }`
- (d) `DELETE /users/42/orders/all`

**练习 3**：API 现有字段 `phone: string`。要支持多个号码。怎么演进？

**练习 4**：cursor-based 分页与 offset 的差异，举例说明 cursor 解决什么问题。

**练习 5**：如何让"创建订单"幂等？

---

## 参考答案

**练习 1**：
```
GET    /v1/posts/{post_id}/comments        列表
POST   /v1/posts/{post_id}/comments        创建
GET    /v1/comments/{id}                    单个
DELETE /v1/comments/{id}                    删除
POST   /v1/comments/{id}:like              点赞
DELETE /v1/comments/{id}/like              取消赞
```
或把 like 当独立资源 `POST /comments/{id}/likes`、`DELETE /comments/{id}/likes/{user_id}`。

**练习 2**：
- (a) ❌ URL 含动词，应 `GET /orders`
- (b) ⚠️ 半 REST（业务动作）；纯 REST 应 `POST /orders/42/refunds`
- (c) ⚠️ PUT 应全量替换；部分更新用 PATCH
- (d) ❌ 用 query：`DELETE /users/42/orders` + 加条件或不允许"删除所有"

**练习 3**：
方案 A：保留 phone 字段，新增 phones 数组；deprecate phone（v1 给单值，v2 给数组）。
方案 B：新版 v2 字段类型改 `phones: string[]`，v1 保留单值。
方案 C：扩展为对象 `{primary: ..., others: []}`。

**练习 4**：offset 分页在数据频繁变化时漂移——第二页可能重复或跳过条目；DB 查询 OFFSET 100000 仍扫前 100000 行慢。cursor 用上一页最后 ID 作为起点（`WHERE id > last_id ORDER BY id LIMIT 10`），稳定 + 快。

**练习 5**：让 client 提供 `Idempotency-Key`：
```http
POST /orders
Idempotency-Key: 7a8b9c-uuid
```
Server 记 key 24 小时；重复 key → 返回上次结果。Stripe 文档是经典参考。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 资源 | 复数名词；URL 表资源，method 表动作 |
| 方法 | 安全/幂等表；PUT 全量、PATCH 部分 |
| 状态码 | 201/204/422/429 等用对 |
| 版本 | URL 路径 /v1/ 最常用 |
| 分页 | cursor > page/offset |
| 错误 | 统一 schema + code |
| 幂等 | Idempotency-Key 让 POST 安全重试 |
| 时间/ID | ISO 8601 UTC；大 ID 用字符串 |

下一篇 **B05 — 精通 gRPC 生产实践** 将基于 G28 进一步讲清服务发现、负载均衡、熔断、流量控制等运维主题。

---

