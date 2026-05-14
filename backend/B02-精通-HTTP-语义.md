# 精通 HTTP 语义：methods、status、headers 与 caching

> 课程编号：B02
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — HTTP
> 难度：⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：HTTP 不只是 GET 和 POST

```
POST /users HTTP/1.1
PUT  /users/42 HTTP/1.1
PATCH /users/42 HTTP/1.1
```

这三个有何不同？什么时候 200 vs 201 vs 204？`Cache-Control: no-cache` 和 `no-store` 区别在哪？`Vary` 头干什么用？HTTP 是后端工程师的"母语"——这一章把语义全部讲清。

---

## 第一章：HTTP Methods

### 1.1 方法对照

| 方法 | 安全 | 幂等 | 用途 |
|---|---|---|---|
| GET | ✅ | ✅ | 获取资源 |
| HEAD | ✅ | ✅ | 仅获取响应头 |
| OPTIONS | ✅ | ✅ | 查询服务器能力 |
| POST | ❌ | ❌ | 创建资源 / 提交动作 |
| PUT | ❌ | ✅ | 替换整个资源 |
| PATCH | ❌ | ❌（推荐）| 部分修改 |
| DELETE | ❌ | ✅ | 删除资源 |
| TRACE | ✅ | ✅ | 调试用（一般禁） |
| CONNECT | ❌ | ❌ | 建立隧道（HTTPS proxy） |

### 1.2 "安全" 与 "幂等"

- **安全（safe）**：不改变服务器状态。GET/HEAD/OPTIONS 应该安全。
- **幂等（idempotent）**：重复调用与一次调用效果相同。GET/PUT/DELETE 是幂等；POST 不是。

### 1.3 PUT vs PATCH

```
PUT /users/42      → 用请求体完全替换 user 42
PATCH /users/42    → 仅修改请求体提供的字段
```

PUT 必须发送完整资源；PATCH 只发改动。RFC 6902 定义了 JSON Patch 格式：

```json
[
  { "op": "replace", "path": "/email", "value": "new@x.com" },
  { "op": "remove", "path": "/temp" }
]
```

但多数 API 用更简单的 JSON Merge Patch：直接发改动字段。

### 1.4 POST 用于"操作"

```
POST /orders/42/cancel
POST /users/42/reset-password
POST /search                    // body 太大用 query 不合适
```

REST 严格派会用 `PATCH /orders/42 {"status":"cancelled"}`，但 RPC 风格的 `POST /...:action` 在实践中很常见。

---

## 第二章：状态码

### 2.1 五个大类

| 范围 | 含义 |
|---|---|
| 1xx | 信息（很少用） |
| 2xx | 成功 |
| 3xx | 重定向 |
| 4xx | 客户端错 |
| 5xx | 服务端错 |

### 2.2 常用 2xx

| 码 | 何时用 |
|---|---|
| 200 OK | 通用成功 |
| 201 Created | POST 创建资源成功，**Location 头**指向新资源 |
| 202 Accepted | 异步任务已接收，处理中 |
| 204 No Content | 成功但无响应体（DELETE、PUT 常用） |
| 206 Partial Content | Range 请求（断点续传） |

### 2.3 常用 3xx

| 码 | 含义 |
|---|---|
| 301 Moved Permanently | 永久重定向，缓存 |
| 302 Found | 临时重定向 |
| 304 Not Modified | 配合条件请求；客户端用本地缓存 |
| 307 Temporary Redirect | 临时，保留 method（302 历史上有 method 转 GET 的混乱） |
| 308 Permanent Redirect | 永久，保留 method |

### 2.4 常用 4xx

| 码 | 含义 |
|---|---|
| 400 Bad Request | 通用客户端错（语法、字段无效） |
| 401 Unauthorized | 未认证（实际是 unauthenticated） |
| 403 Forbidden | 已认证但无权限 |
| 404 Not Found | 资源不存在 |
| 405 Method Not Allowed | 方法不支持，**返回 Allow 头**列出允许的方法 |
| 409 Conflict | 并发冲突 / 版本冲突 |
| 410 Gone | 永久消失（vs 404 不知道） |
| 412 Precondition Failed | If-Match 等条件不满足 |
| 413 Payload Too Large | 请求体过大 |
| 415 Unsupported Media Type | Content-Type 不支持 |
| 422 Unprocessable Entity | 语法对但语义错（如缺字段） |
| 429 Too Many Requests | 限流，**返回 Retry-After 头** |

### 2.5 常用 5xx

| 码 | 含义 |
|---|---|
| 500 Internal Server Error | 通用服务端错 |
| 501 Not Implemented | 服务器不实现这个 method |
| 502 Bad Gateway | 网关/代理收到上游无效响应 |
| 503 Service Unavailable | 暂时不可用（维护、过载） |
| 504 Gateway Timeout | 网关上游超时 |

### 2.6 选择技巧

- "我不知道是 4xx 还是 5xx" → 通常是 5xx（服务端逻辑错）
- 资源不存在 → 404；存在但你看不到 → 403
- 字段缺失 → 422 比 400 更精确
- 限流 → 429 + Retry-After
- 维护 → 503 + Retry-After

---

## 第三章：关键 Headers

### 3.1 Request headers

| Header | 用途 |
|---|---|
| `Host` | 目标主机（HTTP/1.1 必须）|
| `User-Agent` | 客户端标识 |
| `Accept` | 期望的响应内容类型 |
| `Accept-Encoding` | 接受的压缩（gzip, br） |
| `Accept-Language` | 期望语言 |
| `Authorization` | 凭据（Bearer token、Basic 等） |
| `Cookie` | 携带 cookie |
| `Content-Type` | 请求体类型 |
| `Content-Length` | 请求体长度 |
| `If-None-Match` / `If-Modified-Since` | 条件请求（缓存验证） |
| `Range` | 部分请求 |
| `X-Forwarded-For` | 经过代理的客户端 IP 链 |
| `X-Forwarded-Proto` | 原始协议（http/https）|

### 3.2 Response headers

| Header | 用途 |
|---|---|
| `Content-Type` | 响应体类型 + charset |
| `Content-Length` | 响应体长度 |
| `Content-Encoding` | 压缩方式 |
| `Cache-Control` | 缓存策略 |
| `ETag` | 内容版本指纹 |
| `Last-Modified` | 内容最后修改时间 |
| `Location` | 重定向目标 / 新资源 URI |
| `Set-Cookie` | 设置 cookie |
| `Vary` | 告诉 cache 按这些 header 区分 |
| `Server` | 服务端标识（生产建议隐藏） |
| `WWW-Authenticate` | 401 时配套，说明认证方式 |
| `Retry-After` | 503/429 时建议重试间隔 |
| `Access-Control-*` | CORS 系列 |
| `Strict-Transport-Security` | HSTS 强制 HTTPS |
| `X-Content-Type-Options` | nosniff 防 MIME 嗅探 |
| `X-Frame-Options` | 防点击劫持 |

### 3.3 Content-Type 关键值

| 值 | 用途 |
|---|---|
| `application/json` | JSON API |
| `application/x-www-form-urlencoded` | HTML 表单 |
| `multipart/form-data` | 文件上传 |
| `text/html; charset=utf-8` | HTML |
| `application/octet-stream` | 二进制 |
| `text/event-stream` | SSE |
| `application/grpc+proto` | gRPC |

---

## 第四章：HTTP Caching

### 4.1 Cache-Control 指令

```
Cache-Control: public, max-age=3600
Cache-Control: private, max-age=600
Cache-Control: no-cache
Cache-Control: no-store
Cache-Control: must-revalidate
Cache-Control: immutable
```

| 指令 | 含义 |
|---|---|
| `public` | 任何缓存（包括 CDN）可缓存 |
| `private` | 仅浏览器缓存（不要 CDN） |
| `max-age=N` | N 秒内视为新鲜 |
| `s-maxage=N` | 共享缓存的 max-age（覆盖 max-age） |
| `no-cache` | **可以缓存**但每次用前必须 revalidate |
| `no-store` | **完全不缓存** |
| `must-revalidate` | 过期后必须 revalidate（不能用过期版本） |
| `immutable` | 客户端永不需要 revalidate（带 hash 的资源） |

`no-cache` 和 `no-store` 是初学者最大混淆——记住：no-cache 还能 cache，no-store 真的不存。

### 4.2 Validation 头：ETag 与 Last-Modified

```
# 第一次响应
ETag: "abc123"
Last-Modified: Wed, 12 May 2026 10:00:00 GMT

# 第二次请求
If-None-Match: "abc123"
If-Modified-Since: Wed, 12 May 2026 10:00:00 GMT

# 服务端可回
304 Not Modified              （无 body）
```

ETag 更精确（哈希）；Last-Modified 粒度秒。两者都有时 ETag 优先。

### 4.3 Vary

```
Cache-Control: max-age=3600
Vary: Accept-Encoding, Accept-Language
```

告诉缓存：相同 URL 但 `Accept-Encoding` 或 `Accept-Language` 不同要分别存。否则中文用户拿到英文版的 cache 命中——bug。

### 4.4 缓存层级

```
浏览器 → ISP 缓存 → CDN → 反向代理 → origin server
```

每层都可独立缓存。`s-maxage` 控制共享缓存层；`max-age` 控制浏览器。

### 4.5 典型策略

**静态资源（含 hash）**：`Cache-Control: public, max-age=31536000, immutable`
**HTML**：`Cache-Control: no-cache`（每次 revalidate）
**API 列表**：`Cache-Control: private, max-age=60`（短缓存）
**敏感数据**：`Cache-Control: no-store`
**用户特定**：`Cache-Control: private`

---

## 第五章：内容协商

### 5.1 Accept 头家族

```
Accept: application/json, application/xml;q=0.5
Accept-Language: zh-CN, en;q=0.8
Accept-Encoding: br, gzip;q=0.9, identity;q=0.5
```

q 是优先级（0-1）。服务端按客户端偏好选最优可用。

### 5.2 服务端选择

```http
Accept: application/json
→ Content-Type: application/json
   Vary: Accept

Accept: application/xml
→ Content-Type: application/xml
   Vary: Accept
```

确保 `Vary: Accept` 让缓存正确区分。

---

## 第六章：Body 与编码

### 6.1 Content-Length vs Chunked

```
Content-Length: 1024
[exactly 1024 bytes of body]
```

或：

```
Transfer-Encoding: chunked

10\r\n
[16 bytes]\r\n
8\r\n
[8 bytes]\r\n
0\r\n              ← 长度 0 表示结束
\r\n
```

chunked：传输前不知道总长度（如流式生成）。

### 6.2 压缩

```
请求: Accept-Encoding: gzip, br
响应: Content-Encoding: gzip
```

常见编码：
- gzip：广泛支持
- br（Brotli）：压缩率更高，HTTPS 上多
- deflate：少用
- identity：不压缩

API 服务 JSON 响应启用 gzip/br 一般能省 60-80%。

### 6.3 multipart

文件上传：

```
Content-Type: multipart/form-data; boundary=----xyz

------xyz
Content-Disposition: form-data; name="title"

Hello
------xyz
Content-Disposition: form-data; name="file"; filename="a.jpg"
Content-Type: image/jpeg

<binary data>
------xyz--
```

---

## 第七章：Cookies 与会话

### 7.1 Set-Cookie

```
Set-Cookie: session=abc123; Domain=.example.com; Path=/;
            Max-Age=86400; HttpOnly; Secure; SameSite=Strict
```

属性：
- `Domain` / `Path`：作用范围
- `Max-Age` / `Expires`：生命周期
- `HttpOnly`：JS 不可读（防 XSS）
- `Secure`：仅 HTTPS 上发
- `SameSite=Strict|Lax|None`：跨站发送策略（防 CSRF）

### 7.2 SameSite 策略

- `Strict`：跨站请求一律不发 cookie
- `Lax`（多数浏览器默认）：top-level GET 可以
- `None`：任意发，**必须 Secure**

### 7.3 安全建议

- 会话 cookie：HttpOnly + Secure + SameSite=Lax/Strict
- 不要把敏感数据放 cookie（用 server-side session 配 ID）

---

## 第八章：CORS

### 8.1 简单请求

```
GET /api/data
Origin: https://app.example.com
```

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com
```

### 8.2 预检（Preflight）

非简单请求（如 PUT、自定义 header）触发 OPTIONS preflight：

```
OPTIONS /api/data
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: Authorization, Content-Type
```

```
HTTP/1.1 200
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 3600
```

`Max-Age` 让浏览器在期内不重复 preflight。

### 8.3 Credentials

```
Access-Control-Allow-Origin: https://app.example.com   # 不能是 *
Access-Control-Allow-Credentials: true
```

带 cookie 跨域必须显式 origin（不能 wildcard）+ Allow-Credentials。

---

## 第九章：性能与诊断

### 9.1 Server-Timing

```
Server-Timing: db;dur=120, cache;dur=15, total;dur=200
```

浏览器开发者工具直接显示——服务端报告各阶段耗时。

### 9.2 关键 metric

- TTFB（time to first byte）
- 完整传输时间
- 状态码分布
- 缓存命中率

### 9.3 排查

```bash
curl -v https://api.example.com/users/42
curl -w "@curl-format.txt" -o /dev/null -s https://api.example.com/
```

`curl-format.txt`：
```
time_namelookup:  %{time_namelookup}\n
time_connect:     %{time_connect}\n
time_appconnect:  %{time_appconnect}\n
time_pretransfer: %{time_pretransfer}\n
time_starttransfer: %{time_starttransfer}\n
time_total:       %{time_total}\n
```

---

## 第十章：生产级最佳实践

1. **状态码语义化**：404 vs 410 vs 422 都不同含义。
2. **POST 创建必带 Location 头**：让 client 知道新资源 URI。
3. **错误体 JSON 化**：含 code、message、details，不要纯字符串。
4. **API 版本化**：URL 路径（`/v1/`）或 header。
5. **gzip / br 压缩开启**：API 响应能省 70%+。
6. **静态资源加 hash + immutable**：CDN 永久缓存。
7. **HTML no-cache + ETag**：用户立刻看到更新。
8. **CORS 精确配置**：不要无脑 `*`，尤其带 credentials 时。
9. **HSTS 头**：强制 HTTPS。
10. **隐藏 Server 头**：减少攻击面。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：401 用错
"我没权限" 是 403，不是 401。401 是"我不知道你是谁"。

### ❌ 陷阱 2：no-cache 不等于 no-store
no-cache 仍存在缓存，每次 revalidate。

### ❌ 陷阱 3：Vary 漏配
按 Accept-Language 返回不同语言但没 `Vary: Accept-Language` → 缓存混乱。

### ❌ 陷阱 4：跨域问题怪 CORS
预检失败常因服务端没正确返回 OPTIONS。在 LB 层而非服务里处理常更省心。

### ❌ 陷阱 5：301 难撤回
浏览器永久缓存。改用 302 直到确定。

### ❌ 陷阱 6：用 GET 做修改
"GET /delete?id=1" 反模式。爬虫、预取、cache 都会"删除"。

### ❌ 陷阱 7：DELETE 没幂等
DELETE 应该幂等：第一次删返回 204，第二次也应该 204（或 404）。不要"已删除"返回 500。

---

## 第十二章：练习题

**练习 1**：以下场景用哪个 status code？
- (a) 创建 user 成功
- (b) user 不存在
- (c) 已认证但权限不够
- (d) 请求体 JSON 缺少必填字段
- (e) 调用 API 太频繁

**练习 2**：解释 `Cache-Control: no-cache, max-age=0` 与 `Cache-Control: no-store` 的区别。

**练习 3**：服务端要返回 user list 多语言；如何配置 cache headers？

**练习 4**：API `PATCH /users/42 {"email":"x"}` 应当 200 还是 204？理由？

**练习 5**：为何 `Access-Control-Allow-Origin: *` + `Access-Control-Allow-Credentials: true` 会被浏览器拒绝？

---

## 参考答案

**练习 1**：
- (a) **201** + Location 头
- (b) **404**
- (c) **403**
- (d) **422**（语法对、语义错）；400 也可接受
- (e) **429** + Retry-After

**练习 2**：no-cache 允许缓存存储，但每次使用前必须向 origin revalidate（304/200）；no-store 完全不存储，每次都重传。

**练习 3**：
```
Cache-Control: private, max-age=60
Vary: Accept-Language
```
按语言区分缓存，私有（不要 CDN）。

**练习 4**：两者都合理。201 不合适（不是创建）。
- 返回更新后资源 → 200
- 不返回 body → 204
团队约定一致即可。常见选 200 让 client 拿到当前状态。

**练习 5**：安全规则。带 credentials（cookie/auth header）的跨域请求必须**明确**指定 origin，不能用通配符。否则任何站点都可代表用户做请求 → CSRF 风险。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Methods | safe / idempotent 表 |
| Status | 2xx 成功；3xx 重定向；4xx 客户；5xx 服务 |
| Headers | Authorization、Cache-Control、Vary、ETag |
| Cache | no-cache vs no-store；ETag + 304 |
| Vary | 不同维度返回不同内容必加 |
| CORS | preflight；credentials 不能 * |
| Cookie | HttpOnly + Secure + SameSite |

下一篇 **B03 — 精通 TLS、HTTPS 与证书管理** 会讲清 TLS 握手、证书链、Let's Encrypt、Cipher suite、HSTS、SNI、0-RTT。

---

> 📁 本课程位于 `/data/workspace/dp4/backend/B02-精通-HTTP-语义.md`
