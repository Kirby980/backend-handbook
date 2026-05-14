# 精通 GraphQL：schema、resolver 与 N+1

> 课程编号：B06
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — GraphQL
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：客户端定义需要的数据

```graphql
query {
  user(id: 42) {
    name
    orders(last: 5) {
      total
      items { name price }
    }
  }
}
```

REST 要 3 个请求才能拿这堆数据；GraphQL 一次就行。但 GraphQL 不是 REST 的"升级版"——它有完全不同的 trade-off。本章拆开 schema 设计、resolver 机制、N+1 解决、federation、何时该用何时不该用。

---

## 第一章：GraphQL 是什么

### 1.1 三个核心

- **类型系统**：强类型 schema 描述所有数据
- **查询语言**：client 精确声明想要哪些字段
- **runtime**：解析 query、调 resolver、组装响应

### 1.2 三种操作

```graphql
query GetUser { user(id: 42) { name } }           # 读
mutation CreateUser { createUser(...) { id } }    # 写
subscription Updates { messageAdded { id text } } # 实时
```

### 1.3 与 REST 对比

| 维度 | REST | GraphQL |
|---|---|---|
| Endpoint | 多个 (`/users/`、`/orders/`) | 单个 `/graphql` |
| 数据形态 | 服务端决定 | 客户端决定 |
| 版本 | URL `/v1/` | schema 演进 |
| over-fetch | 常见 | 几乎不（精确字段） |
| under-fetch | 常见 | 几乎不（嵌套） |
| 缓存 | HTTP cache 友好 | 自实现（POST 难 cache） |
| 学习曲线 | 低 | 中 |
| 工具 | curl + browser | GraphiQL / Apollo Studio |

---

## 第二章：Schema

### 2.1 类型定义

```graphql
type User {
    id: ID!
    name: String!
    email: String
    age: Int
    orders: [Order!]!
    manager: User
}

type Order {
    id: ID!
    total: Float!
    items: [OrderItem!]!
    user: User!
}

type OrderItem {
    name: String!
    price: Float!
    quantity: Int!
}
```

- `!` 表示非空
- `[T!]!` 表示非空数组，元素非空
- `[T]` 数组可 null，元素可 null
- 标量：`Int`、`Float`、`String`、`Boolean`、`ID`

### 2.2 query / mutation 入口

```graphql
type Query {
    user(id: ID!): User
    users(first: Int = 10, after: String): UserConnection!
}

type Mutation {
    createUser(input: CreateUserInput!): User!
    updateUser(id: ID!, input: UpdateUserInput!): User!
}

type Subscription {
    userUpdated(id: ID!): User!
}
```

### 2.3 Input type

mutation 输入用 input type（不是普通 type）：

```graphql
input CreateUserInput {
    name: String!
    email: String!
    age: Int
}
```

### 2.4 Enum

```graphql
enum OrderStatus {
    PENDING
    PAID
    SHIPPED
    DELIVERED
    CANCELLED
}
```

### 2.5 Interface 与 Union

```graphql
interface Node { id: ID! }

type User implements Node {
    id: ID!
    name: String!
}

type Order implements Node {
    id: ID!
    total: Float!
}

union SearchResult = User | Order
```

interface 共享字段集；union 允许任意类型组合。

### 2.6 自定义标量

```graphql
scalar DateTime
scalar UUID
scalar JSON
```

解析逻辑在 resolver 注册。

---

## 第三章：Connection / 分页

### 3.1 Relay-style Connection

```graphql
type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
}
type UserEdge {
    node: User!
    cursor: String!
}
type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
}
```

查询：

```graphql
{
  users(first: 10, after: "abc") {
    edges {
      cursor
      node { id name }
    }
    pageInfo { hasNextPage endCursor }
  }
}
```

cursor-based 分页，标准化、稳定。

### 3.2 简单分页

简单场景可不用 Connection：

```graphql
users(limit: 10, offset: 0): [User!]!
```

但失去标准化优势。

---

## 第四章：Resolver

### 4.1 概念

每个字段都有一个 resolver——返回该字段值的函数。

```js
const resolvers = {
  Query: {
    user: (parent, args, ctx, info) => {
      return ctx.db.user.findById(args.id);
    }
  },
  User: {
    orders: (user, args, ctx) => {
      return ctx.db.order.findByUserId(user.id);
    }
  }
};
```

### 4.2 参数

- **parent**：上一层 resolver 的返回值
- **args**：query 参数
- **ctx**：跨 resolver 共享（DB client、user info）
- **info**：query AST 等元信息

### 4.3 默认 resolver

如果不写，runtime 自动 `parent[fieldName]`。所以大多字段不需要显式 resolver——只关心计算字段或跨表的。

---

## 第五章：N+1 问题

### 5.1 经典坑

```graphql
{ users(first: 100) { name orders { total } } }
```

resolver 流程：
1. 查 100 个 user：1 query
2. 对每个 user 查 orders：100 query

总共 101 query——慢。

### 5.2 DataLoader 解决

DataLoader 在一个 tick 内合并多次查询：

```js
const orderLoader = new DataLoader(async (userIds) => {
    const all = await db.order.findByUserIds(userIds);   // 单次 IN 查询
    return userIds.map(id => all.filter(o => o.userId === id));
});

const resolvers = {
    User: {
        orders: (user, _, { loaders }) => loaders.order.load(user.id),
    }
};
```

一个 tick 收集所有 load 调用，批量请求，返回。101 query 变 2 query。

### 5.3 dataloader 必备

任何关联字段都该有 dataloader。每请求新建 loader（避免跨请求 cache 污染）：

```js
context: () => ({ loaders: makeLoaders() })
```

---

## 第六章：错误处理

### 6.1 响应结构

```json
{
  "data": { "user": null },
  "errors": [
    {
      "message": "User not found",
      "path": ["user"],
      "extensions": { "code": "NOT_FOUND" }
    }
  ]
}
```

GraphQL 永远返回 200 OK，错误在 `errors` 数组里。这跟 REST 用 HTTP 状态码不一样——常被吐槽。

### 6.2 部分成功

```graphql
{ user { name } broken: undefinedField }
→
{
  "data": { "user": { "name": "Alice" }, "broken": null },
  "errors": [{ ... }]
}
```

一个字段错不阻塞其他字段。

### 6.3 错误分类

extension code：
- `UNAUTHENTICATED`
- `FORBIDDEN`
- `NOT_FOUND`
- `BAD_USER_INPUT`
- `INTERNAL_SERVER_ERROR`

Apollo 等服务器有标准化错误类。

---

## 第七章：Mutation

### 7.1 命名

```graphql
type Mutation {
    createUser(input: CreateUserInput!): CreateUserPayload!
    updateUser(input: UpdateUserInput!): UpdateUserPayload!
    deleteUser(input: DeleteUserInput!): DeleteUserPayload!
}
```

每个 mutation 返回一个 payload type（不是直接返回实体）：

```graphql
type CreateUserPayload {
    user: User
    errors: [UserError!]
}
```

便于扩展（加 client-mutation-id、副作用列表）。

### 7.2 单一 input 参数

```graphql
createUser(input: CreateUserInput!)
```

而不是多个参数 `createUser(name, email, age)`。input 容易演进（新增字段不破坏）。

### 7.3 幂等性

GraphQL 不像 HTTP method 有"PUT 幂等"约束。**实现幂等是开发者责任**——用 client mutation ID 或 idempotency key。

---

## 第八章：Subscription

### 8.1 实时数据

```graphql
subscription { messageAdded(roomId: "42") { id text } }
```

server 推送匹配的事件。底层通常是 WebSocket（`graphql-ws` 协议）。

### 8.2 适合场景

- 聊天
- 实时通知
- 协同编辑
- 直播 stats

### 8.3 不适合

- 高频更新（每秒上百次）：用专门的流（gRPC streaming、SSE）
- 大量并发订阅：服务器要维护 socket 状态

---

## 第九章：Schema 演进

### 9.1 非破坏性变更

- 添加 type
- 添加可选字段
- 添加 enum value（**慎**——客户端可能没处理新值）
- 添加可选 argument
- 标记 `@deprecated`

### 9.2 破坏性变更

- 删除字段
- 改字段类型
- 改非空 ↔ 可空
- 改 argument 默认值
- 删 enum value

### 9.3 deprecation

```graphql
type User {
    name: String! @deprecated(reason: "Use displayName")
    displayName: String!
}
```

工具能检测 deprecated 字段使用，给出迁移提示。

### 9.4 schema registry

类似 buf for protobuf——把 schema 版本化、检测破坏性变更、强制 PR review。Apollo Studio、GraphQL Hive。

---

## 第十章：性能与缓存

### 10.1 query 复杂度

恶意 query 可能让 server 跑超大 query：

```graphql
{ users { orders { user { orders { user { ... } } } } } }
```

防御：
- **depth limiting**：最大嵌套深度 (`graphql-depth-limit`)
- **complexity scoring**：每字段计 cost，总分超限拒绝
- **rate limiting**：标准限流
- **persisted queries**：仅接受预登记的 query

### 10.2 缓存

GraphQL POST 难走 HTTP cache。常见做法：

- **Apollo Client cache**：浏览器内归一化缓存（按 id 索引）
- **Persisted query + GET**：query → hash → GET `/graphql?id=hash`，CDN 友好
- **服务端 dataloader cache**：单请求内
- **Redis cache by field**：服务端按字段值缓存

### 10.3 字段级权限

```js
const resolvers = {
    User: {
        email: (user, _, ctx) => {
            if (ctx.user.id !== user.id && !ctx.user.isAdmin) return null;
            return user.email;
        }
    }
};
```

按字段控制访问，比 REST 端点级别更细。

---

## 第十一章：Federation

### 11.1 多团队拆分

大公司多个团队，各自维护一部分 schema：

- User 团队负责 User type
- Order 团队负责 Order type
- Payment 团队负责 Payment type

### 11.2 Apollo Federation

每个团队提供 subgraph，gateway 把它们组合成超级 schema：

```graphql
# user-service
type User @key(fields: "id") {
    id: ID!
    name: String!
}

# order-service
type Order {
    id: ID!
    user: User! @provides("name")
    total: Float!
}

extend type User @key(fields: "id") {
    id: ID! @external
    orders: [Order!]!
}
```

gateway 自动跨服务 join。

### 11.3 替代：schema stitching

更老的方式，手工配置 gateway。已基本被 federation 取代。

---

## 第十二章：服务端实现

### 12.1 Go

- `gqlgen`：从 schema 生成代码（类型安全）
- `graph-gophers/graphql-go`

### 12.2 Node.js

- Apollo Server（最流行）
- GraphQL Yoga
- Mercurius（Fastify）

### 12.3 Python

- Strawberry
- Graphene

### 12.4 Java/Kotlin

- DGS Framework（Netflix）
- GraphQL Java

---

## 第十三章：何时用 GraphQL

### 13.1 适合

- **前端有多种视图**：移动版 vs 桌面版字段需求不同
- **多个数据源 aggregator**：单 endpoint 聚合 microservices
- **数据图状关联**：社交、知识图谱
- **快速迭代**：前端不阻塞等后端

### 13.2 不适合

- **简单 CRUD**：REST 足够
- **文件上传 / 二进制流**：GraphQL multipart 扩展笨拙
- **极致性能**：解析 query AST 有开销
- **HTTP cache 重要**：GraphQL POST 难 cache（除非 persisted query）
- **小团队 + 简单业务**：上 GraphQL 复杂度不划算

---

## 第十四章：生产级最佳实践

1. **input type + payload type**：mutation 标准结构。
2. **每请求新 dataloader**：解 N+1 + 防 cache 污染。
3. **复杂度限制 + depth limit**：防恶意 query。
4. **persisted queries**：CDN cache + 减小 body。
5. **schema registry**：CI 检测破坏性变更。
6. **错误带 extensions.code**：machine-readable。
7. **federation 或单 monolithic**：不要混合。
8. **字段级 deprecation**：渐进迁移。
9. **subscription 用专门基础设施**：WebSocket gateway、Redis pub/sub。
10. **观测：每字段延迟 + 错误率**：Apollo Studio、Datadog。

---

## 第十五章：常见陷阱清单

### ❌ 陷阱 1：N+1
没用 dataloader → 一个 query 触发上百 DB 调用。

### ❌ 陷阱 2：返回敏感字段
没字段级权限 → client 一查全暴露。

### ❌ 陷阱 3：用 graphql 当 REST
仅暴露简单 CRUD → 没用上 GraphQL 优势，反而多一层复杂。

### ❌ 陷阱 4：依赖 errors[].message 做业务判断
message 是给人看的；用 extensions.code。

### ❌ 陷阱 5：subscription 容易泄漏 socket
每个订阅占资源，重连风暴时炸 server。限流 + 心跳。

### ❌ 陷阱 6：mutation 不幂等
重复创建。加 idempotency key 或 unique constraint。

### ❌ 陷阱 7：schema 频繁破坏性变更
没注册 + 监控 → client 频繁挂。

---

## 第十六章：练习题

**练习 1**：设计一个博客系统的 GraphQL schema（Post、Comment、User、Tag）。

**练习 2**：以下 query 触发 N+1，怎么修？
```graphql
{ posts { author { name } comments { author { name } } } }
```

**练习 3**：解释为何 GraphQL 一般 POST 而难做 HTTP caching。

**练习 4**：写一个简单的 depth limiter，限制最大嵌套 5 层。

**练习 5**：subscription 与 SSE 各适合什么场景？

---

## 参考答案

**练习 1**：
```graphql
type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
    comments(first: Int = 10): CommentConnection!
    tags: [Tag!]!
}
type Comment {
    id: ID!
    text: String!
    author: User!
    post: Post!
}
type User { id: ID! name: String! posts: [Post!]! }
type Tag { id: ID! name: String! posts: [Post!]! }

type Query {
    post(id: ID!): Post
    posts(first: Int = 10, after: String): PostConnection!
    user(id: ID!): User
}
type Mutation {
    createPost(input: CreatePostInput!): CreatePostPayload!
    addComment(input: AddCommentInput!): AddCommentPayload!
}
```

**练习 2**：每个 `author` 字段触发一次 DB 查询。用 dataloader：
```js
{ User: { /* default */ }, Post: { author: (post, _, {l}) => l.user.load(post.authorId) },
  Comment: { author: (c, _, {l}) => l.user.load(c.authorId) } }
```

**练习 3**：GraphQL 多数用 POST + body。HTTP cache 按 URL key，POST body 不参与 key。解决：persisted query（query → hash，用 GET ?id=hash）。

**练习 4**：遍历 AST，找最深 depth：
```js
function maxDepth(doc, currentDepth = 0) {
    let max = currentDepth;
    if (doc.selectionSet) {
        for (const sel of doc.selectionSet.selections) {
            max = Math.max(max, maxDepth(sel, currentDepth + 1));
        }
    }
    return max;
}
// in middleware: if maxDepth > 5 → reject
```

或用 `graphql-depth-limit` 包。

**练习 5**：
- **subscription**：双向、复杂事件、需要 GraphQL schema 集成
- **SSE**：单向 server → client、简单事件流、HTTP 兼容性好

简单实时通知用 SSE；GraphQL 重度应用 + 客户端已用 Apollo 用 subscription。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Schema | type / input / interface / union / enum |
| Resolver | 每字段一个；默认 parent[field] |
| N+1 | DataLoader 批量 |
| Mutation | input + payload type |
| Subscription | WebSocket；高频用专门设施 |
| 复杂度 | depth + cost 限制 |
| Federation | 多服务组合超级 schema |
| 何时不用 | 简单 CRUD、文件上传、HTTP cache 重要 |

下一篇 **B07 — 精通 OpenAPI 契约** 会讲清 spec 编写、code generation、契约测试、版本管理。

---

