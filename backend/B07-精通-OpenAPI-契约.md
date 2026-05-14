# 精通 OpenAPI 契约与代码生成

> 课程编号：B07
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — OpenAPI
> 难度：⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：契约优先

```yaml
openapi: 3.1.0
info:
  title: User API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: integer }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

OpenAPI（前身 Swagger）是 REST API 的 IDL。一份 YAML/JSON 同时是：API 文档、客户端 SDK 源、mock server、契约测试基础、IDE 自动补全数据源。本章讲清写好 spec、code gen、与 CI 集成、版本管理。

---

## 第一章：OpenAPI 历史与版本

### 1.1 演进

- **Swagger 1.0** (2011)：Wordnik 公司开发
- **Swagger 2.0** (2014)：广泛流行
- **OpenAPI 3.0** (2017)：捐给 Linux Foundation
- **OpenAPI 3.1** (2021)：完整对齐 JSON Schema 2020-12

新项目用 **3.1**；很多生态仍是 3.0。两者大体兼容。

### 1.2 与 Swagger 工具的关系

- **Swagger Editor**：在线编辑 + 验证
- **Swagger UI**：从 spec 生成可交互文档
- **OpenAPI** 是规范本身，"Swagger" 现在指工具集

---

## 第二章：Spec 结构

### 2.1 顶层

```yaml
openapi: 3.1.0
info:
  title: User API
  description: ...
  version: 1.0.0
  contact: { email: api@example.com }
  license: { name: MIT }
servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://api-staging.example.com/v1
    description: Staging
paths: ...
components: ...
security: ...
tags: ...
```

### 2.2 paths

```yaml
paths:
  /users:
    get:
      summary: List users
      operationId: listUsers
      tags: [users]
      parameters:
        - $ref: '#/components/parameters/PageParam'
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserList'
    post:
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Created
          headers:
            Location:
              schema: { type: string }
```

每个 endpoint：method + summary + parameters + requestBody + responses。

### 2.3 components

```yaml
components:
  schemas:
    User:
      type: object
      required: [id, name]
      properties:
        id: { type: integer, format: int64 }
        name: { type: string, minLength: 1, maxLength: 100 }
        email: { type: string, format: email }
        age: { type: integer, minimum: 0, maximum: 150 }
    CreateUserRequest:
      type: object
      required: [name]
      properties:
        name: { type: string }
        email: { type: string, format: email }
  parameters:
    PageParam:
      name: page
      in: query
      schema: { type: integer, minimum: 1, default: 1 }
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
```

`components` 集中定义可复用部分，`$ref` 引用。

---

## 第三章：Schema 定义

### 3.1 类型

```yaml
schema:
  type: string
  format: email | uri | date | date-time | uuid | binary | byte
  minLength: 1
  maxLength: 100
  pattern: '^[a-z]+$'
  enum: [active, inactive]
```

```yaml
schema:
  type: integer
  format: int32 | int64
  minimum: 0
  exclusiveMinimum: false
  maximum: 100
  multipleOf: 5
```

### 3.2 object

```yaml
schema:
  type: object
  required: [id, name]
  properties:
    id: { type: integer }
    name: { type: string }
  additionalProperties: false   ← 严格模式
```

### 3.3 array

```yaml
schema:
  type: array
  items: { type: string }
  minItems: 1
  maxItems: 100
  uniqueItems: true
```

### 3.4 组合

```yaml
oneOf:    # 必须匹配恰好一个
  - $ref: '#/components/schemas/Cat'
  - $ref: '#/components/schemas/Dog'
anyOf:    # 至少一个
  - ...
allOf:    # 全部（继承）
  - $ref: '#/components/schemas/Base'
  - type: object
    properties: { extra: { type: string } }
```

### 3.5 nullable

OpenAPI 3.1（对齐 JSON Schema）：
```yaml
type: [string, "null"]
```

3.0：
```yaml
type: string
nullable: true
```

---

## 第四章：参数与请求体

### 4.1 参数位置

```yaml
parameters:
  - name: id
    in: path           # path / query / header / cookie
    required: true
    schema: { type: integer }
  - name: filter
    in: query
    schema: { type: string }
    style: form
    explode: true
```

### 4.2 array query parameter

```yaml
- name: tags
  in: query
  schema:
    type: array
    items: { type: string }
  style: form           # 默认；逗号分隔
  explode: true         # ?tags=a&tags=b
```

不同 style 序列化方式不同，跨语言 client 注意一致。

### 4.3 requestBody

```yaml
requestBody:
  required: true
  content:
    application/json:
      schema: { $ref: '#/components/schemas/CreateUser' }
    application/x-www-form-urlencoded:
      schema: ...
    multipart/form-data:
      schema:
        type: object
        properties:
          file:
            type: string
            format: binary
```

支持多 content-type，client 选合适的。

---

## 第五章：Response

### 5.1 多状态码

```yaml
responses:
  '200':
    description: OK
    content:
      application/json:
        schema: { $ref: '#/components/schemas/User' }
  '404':
    $ref: '#/components/responses/NotFound'
  '422':
    $ref: '#/components/responses/ValidationError'
```

### 5.2 错误统一

```yaml
components:
  responses:
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
  schemas:
    Error:
      type: object
      required: [code, message]
      properties:
        code: { type: string }
        message: { type: string }
        details: { type: object, additionalProperties: true }
```

### 5.3 响应 headers

```yaml
responses:
  '201':
    description: Created
    headers:
      Location:
        schema: { type: string, format: uri }
      X-Rate-Limit:
        schema: { type: integer }
```

---

## 第六章：安全方案

### 6.1 三种主流

**Bearer token (JWT)**
```yaml
securitySchemes:
  BearerAuth:
    type: http
    scheme: bearer
    bearerFormat: JWT
security:
  - BearerAuth: []
```

**API Key**
```yaml
securitySchemes:
  ApiKeyAuth:
    type: apiKey
    in: header
    name: X-API-Key
```

**OAuth 2.0**
```yaml
securitySchemes:
  OAuth2:
    type: oauth2
    flows:
      authorizationCode:
        authorizationUrl: https://example.com/oauth/authorize
        tokenUrl: https://example.com/oauth/token
        scopes:
          read: Read access
          write: Write access
```

### 6.2 endpoint 级别覆盖

```yaml
paths:
  /public:
    get:
      security: []   ← 不需要认证
  /admin:
    get:
      security:
        - OAuth2: [write]
```

---

## 第七章：代码生成

### 7.1 工具

| 工具 | 语言 |
|---|---|
| **openapi-generator** | Java + 25+ 语言 |
| **swagger-codegen** | 老版本，已合并到 openapi-generator |
| **oapi-codegen** | Go 专用，速度快、quality 好 |
| **NSwag** | C# / TypeScript |
| **Orval** | TypeScript + React Query 友好 |

### 7.2 生成 client SDK

```bash
openapi-generator generate \
  -i openapi.yaml \
  -g typescript-fetch \
  -o ./sdk
```

生成的代码包含 type、HTTP client、错误处理。前端直接 import 用。

### 7.3 生成 server stub

```bash
oapi-codegen -package api -generate types,server openapi.yaml > api.gen.go
```

生成 Go 接口 + 路由绑定。开发者只需实现接口方法。

### 7.4 schema → 类型

仅类型也常用：

```bash
openapi-typescript openapi.yaml --output api-types.ts
```

前端拿到类型而不接受全套生成的 client，更灵活。

---

## 第八章：契约测试

### 8.1 验证响应符合 schema

```js
import Ajv from 'ajv';
const ajv = new Ajv();
const validate = ajv.compile(userSchema);

test('GET /users/42 returns valid User', async () => {
    const res = await fetch('/users/42');
    const body = await res.json();
    expect(validate(body)).toBe(true);
});
```

### 8.2 服务端中间件验证请求

```js
import { OpenAPIValidator } from 'express-openapi-validator';
app.use(OpenAPIValidator.middleware({ apiSpec: './openapi.yaml' }));
```

请求自动校验：错误字段 → 400；自动响应错误。

### 8.3 Contract testing 工具

- **Dredd**：用 OpenAPI 直接打 API + 验证
- **Postman + Newman**：collection 跑断言
- **Pact**：双向契约测试（consumer-driven）

---

## 第九章：文档与 mock

### 9.1 Swagger UI

托管或自托管：
```html
<script src="https://unpkg.com/swagger-ui-dist@4/swagger-ui-bundle.js"></script>
```

加载 `openapi.yaml` 生成交互文档——开发者可在浏览器试 endpoint。

### 9.2 Redoc

Swagger UI 替代，专注阅读体验（不可交互但更美观）。

### 9.3 Mock server

```bash
prism mock openapi.yaml      # localhost:4010 模拟所有 endpoint
```

前端不等后端就能开发；CI 集成 + 行为验证。

### 9.4 在 Stoplight Studio / Insomnia / Postman

GUI 编辑 spec、调试 endpoint、生成文档。Stoplight Studio 是 OpenAPI 专门编辑器。

---

## 第十章：版本管理

### 10.1 SemVer

- MAJOR：破坏性
- MINOR：兼容性新增
- PATCH：bug 修复

### 10.2 检测破坏性变更

```bash
oasdiff breaking old.yaml new.yaml
```

输出：
- 删除 endpoint
- 改字段类型
- 新增必填字段
- ...

CI 集成防止误发破坏性变更。

### 10.3 多版本并存

```
/v1/users
/v2/users
```

OpenAPI 中维护多个 spec（每版本一个）。或单一 spec 含多版本路径。

---

## 第十一章：组织 spec

### 11.1 单文件 vs 拆分

小项目单文件 OK。大项目拆：

```
openapi.yaml
paths/
  users.yaml
  orders.yaml
schemas/
  user.yaml
  order.yaml
```

主文件用 `$ref` 引用：

```yaml
paths:
  /users:
    $ref: './paths/users.yaml#/users'
```

工具如 `swagger-cli bundle` 把多文件合成单文件分发。

### 11.2 复用 schema

```yaml
$ref: '#/components/schemas/User'              # 同文件
$ref: 'schemas/user.yaml#/User'                # 跨文件
$ref: 'https://...common/schemas.yaml#/User'   # URL
```

---

## 第十二章：生产级最佳实践

1. **API 先 spec，再实现**：spec-first 工作流。
2. **生成 server stub + client SDK**：减少手写 boilerplate。
3. **schema 严格 + required 字段精确**。
4. **错误统一 schema**：每个 response 4xx/5xx 都引用。
5. **`additionalProperties: false`**：防止未知字段悄然进入。
6. **format 严格**：date-time、email、uuid 等。
7. **CI 跑 oasdiff**：检测破坏性。
8. **服务端用 middleware 验证**：spec 与实现一致。
9. **托管 Swagger UI**：开发体验。
10. **变更通过 PR**：spec 文件像代码一样 review。

---

## 第十三章：常见陷阱清单

### ❌ 陷阱 1：spec 与实现不一致
spec 说 200 返回 User，实际返回 { user: User }。
解决：契约测试 + middleware 验证。

### ❌ 陷阱 2：太多 anyOf / oneOf
client SDK 生成代码烂；类型推导难。尽量平展。

### ❌ 陷阱 3：用 number 而非 integer
JSON number 浮点；预期整数时显式 type: integer。

### ❌ 陷阱 4：忘加 required
默认所有字段可选——client 不安全。

### ❌ 陷阱 5：嵌套太深
6 层嵌套 schema → 生成的代码可读性差。提取子 schema。

### ❌ 陷阱 6：在 paths 内联 schema
重复定义。永远放 `components/schemas` 复用。

### ❌ 陷阱 7：example 与 schema 矛盾
example 写错值，IDE / 文档显示错误。CI 校验 example 满足 schema。

---

## 第十四章：练习题

**练习 1**：写一个 `POST /users` 的完整 OpenAPI 定义（含 input、201/422 响应）。

**练习 2**：以下 schema 有什么问题？
```yaml
type: object
properties:
  age: { type: number }
```

**练习 3**：在 CI 中如何检测"不小心删了一个字段"？

**练习 4**：解释 `oneOf` vs `anyOf` vs `allOf`。

**练习 5**：service A 改了字段类型，导致 client SDK 编译失败。这是技术上的破坏还是业务上的破坏？

---

## 参考答案

**练习 1**：
```yaml
paths:
  /users:
    post:
      operationId: createUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: Created
          headers:
            Location: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
        '422':
          $ref: '#/components/responses/ValidationError'
components:
  schemas:
    CreateUserRequest:
      type: object
      required: [name, email]
      properties:
        name: { type: string, minLength: 1 }
        email: { type: string, format: email }
```

**练习 2**：① age 用 `integer` 更精确；② 没 minimum；③ 不限 max（200 岁？）；④ 没 required 标记。

**练习 3**：
```bash
oasdiff breaking old.yaml new.yaml
```
或 GitHub Action `oasdiff/breaking-changes-action`，PR 失败阻止合并。

**练习 4**：
- `oneOf`：必须匹配**恰好一个**子 schema（互斥）
- `anyOf`：匹配**至少一个**（可重叠）
- `allOf`：**全部**匹配（用于继承 / 组合）

**练习 5**：两者都是。**技术上**破坏（旧 client 编译失败）+ **业务上**破坏（旧 server 与新 client 不能通信）。要 SemVer MAJOR 升 + 维护过渡期。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| spec 优先 | spec → 生成 server stub + client SDK |
| 结构 | info / servers / paths / components |
| schema | type + format + 约束 + required |
| security | Bearer / ApiKey / OAuth2 |
| code gen | openapi-generator / oapi-codegen |
| 契约测试 | middleware 验证 + Dredd / Pact |
| 版本 | SemVer + oasdiff |
| 工具 | Swagger UI / Redoc / Prism mock |

下一篇 **B08 — 精通数据库索引** 将拆开 B-tree、Hash、复合索引、覆盖索引、慢查询分析。

---

