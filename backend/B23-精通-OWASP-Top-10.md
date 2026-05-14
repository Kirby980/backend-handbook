# 精通 OWASP Top 10：后端工程师必知 Web 安全风险

> 课程编号：B23
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — OWASP Top 10
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：默认不安全

```
"我们的服务太小没人盯"   ← 自动扫描器 24/7 扫整个 IPv4
"用户都是好人"           ← 内部恶意 / 账号被盗
"框架自带安全"           ← 配错一行全军覆没
```

OWASP Top 10 是 Web 安全的"必读"——每个后端工程师都该熟悉。本章按 **OWASP Top 10 2021** 讲清每条风险、识别方法、防御（**OWASP Top 10 2025 草案** 在 2025-11 公布、预计 2026 年内定稿，主要是把 SSRF / 软件供应链 / API misconfig 等独立成项；正式版发布后会更新本章）。

**纯 Web 应用看 Top 10 2021；API-first 服务还要补 [OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x00-header/)**——见 §13.5。

---

## A01：访问控制失效（Broken Access Control）

### 含义

授权检查缺失——用户能访问不该看的数据 / 执行不该的操作。

### 经典场景

**IDOR（不安全直接对象引用）**：
```
GET /api/orders/42       → 我的订单
GET /api/orders/43       → 别人的订单（也返回了！）
```

服务端只验登录、没验"这个 order 是不是你的"。

**强制浏览**：
```
普通用户  访问 /admin/users → 应 403，但实际 200
```

**method override**：
```
GET 改成 DELETE → 删别人资源
```

### 防御

- 每个资源访问都验所有权
- 默认拒绝，白名单允许
- 后端校验，不靠前端
- 使用 framework 的 authz 中间件
- 自动化测试：模拟"另一用户"访问
- 用 UUID 替代连续 ID（提高难度，但不是真正防御）

```python
@require_login
def get_order(request, order_id):
    order = Order.objects.get(id=order_id)
    if order.user_id != request.user.id:
        return 403
    return order
```

---

## A02：加密失败（Cryptographic Failures）

### 含义

敏感数据传输 / 存储不加密或弱加密。

### 经典场景

- HTTP 传密码（无 TLS）
- 密码用 MD5 / SHA256（应 bcrypt）
- 数据库 / 备份 / 日志含敏感字段明文
- 弱算法（DES、RC4）
- 自创加密"算法"

### 防御

- HTTPS 强制 + HSTS
- 密码：bcrypt / argon2id
- 信用卡：PCI-DSS，多数情况不该自己存（用 Stripe）
- DB 加密：column 级（特别敏感）+ 全盘加密
- 日志脱敏：不 log 密码、PII、token
- 用现成的加密库（OpenSSL、libsodium），不自造

### 不要 log

```python
# ❌
log.info(f"login attempt: user={u}, password={p}")
log.info(f"token issued: {jwt_token}")

# ✅
log.info("login attempt", extra={"user": u})
```

---

## A03：注入（Injection）

### 含义

把用户输入当代码执行。

### SQL 注入

```python
# ❌
db.execute(f"SELECT * FROM users WHERE name = '{name}'")
# 输入: '; DROP TABLE users; --
# 实际: SELECT * FROM users WHERE name = ''; DROP TABLE users; --'
```

```python
# ✅ 参数化
db.execute("SELECT * FROM users WHERE name = ?", (name,))
```

ORM 默认参数化。

### Command 注入

```python
# ❌
os.system(f"convert {filename} out.png")
# 输入: a.png; rm -rf /

# ✅
subprocess.run(["convert", filename, "out.png"], shell=False)
```

### NoSQL 注入

```javascript
// ❌
db.users.find({ name: req.body.name })
// 输入 name: { $ne: null } → 返回所有用户

// ✅ 验证 + 强类型
db.users.find({ name: String(req.body.name) })
```

### LDAP / XPath 注入

类似——用户输入拼字符串。改参数化或 escape。

### 防御

- 参数化查询永远第一选
- ORM 默认正确
- 验证 + 类型检查输入
- 最小权限 DB 账号

---

## A04：不安全设计（Insecure Design）

### 含义

架构层面的缺陷——不是 bug，是设计错。

### 经典场景

- 找回密码用"问母亲娘家姓"（社工破解）
- 验证码可重放
- 抢购无幂等 → 一秒内多次购买
- 公开 endpoint 没限流 → 资源耗尽
- 设计未考虑滥用场景（abuse case）

### 防御

- 威胁建模（threat modeling）：设计时想"谁可能怎么坑这？"
- 安全设计原则：默认拒绝、最小权限、纵深防御
- abuse case 测试
- design review 含安全人员

---

## A05：安全配置错误（Security Misconfiguration）

### 含义

软件配置不安全。

### 经典场景

- 默认密码没改（admin/admin）
- 调试模式生产开启（栈 trace 暴露）
- 没必要的功能 / endpoint 开放
- 不安全的默认 header（缺 HSTS、CSP）
- S3 bucket 公开可读
- DB 监听公网 IP
- error message 暴露技术细节

### 防御

- 安全 baseline 配置
- 关掉默认 / 不用的功能
- 定期 security scanning（Trivy、Snyk）
- 容器最小基础镜像（distroless）
- IaC 模板审计

---

## A06：易受攻击和过时的组件（Vulnerable Components）

### 含义

依赖库有已知漏洞。

### 经典案例

- log4j (CVE-2021-44228)：JNDI 注入 → 整个 Java 生态危机
- Equifax 用过时 Apache Struts → 1.5 亿用户数据泄漏

### 防御

- 依赖锁定（package-lock、go.sum）
- 自动扫描（Dependabot、Snyk、Trivy、govulncheck）
- 关注 CVE 通报
- 升级流程：每月 / 严重时立即
- 减少依赖（小即美）

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

---

## A07：身份验证失败（Identification & Authentication Failures）

详见 B22。重点：

- 弱密码策略允许 "password123"
- 无 lockout → 暴力破解
- session 不过期
- session ID 可预测
- JWT 实现错（alg none、密钥泄露）
- MFA 缺失或可绕过

### 防御

- 密码强度要求 + 检查 [HaveIBeenPwned](https://haveibeenpwned.com/Passwords) API
- 登录失败限流 + lockout
- 强随机 session ID（>= 128 bit）
- session 过期 + 退出真退出
- MFA 高价值账户

---

## A08：软件和数据完整性失败（Software & Data Integrity Failures）

### 含义

软件 / 数据来源不验证或反序列化未验证数据。

### 经典场景

- 从不可信源拉镜像 / package
- 不验证 CI/CD pipeline（攻击者篡改）
- 反序列化用户数据（Java、Python pickle）→ 代码执行

### 反序列化攻击

```python
# ❌
import pickle
data = pickle.loads(request.body)   # 反序列化执行 __reduce__ → 任意代码

# ✅
data = json.loads(request.body)
```

Java Jackson、PHP unserialize 同类风险。

### 防御

- 仅反序列化可信源
- 用 JSON 等"无代码执行"格式
- 包验证（hash、签名）
- supply-chain security：SBOM、Sigstore

---

## A09：日志和监控失败（Logging & Monitoring Failures）

### 含义

事件没记录 / 没监控 → 攻击发生几个月才发现。

### 经典案例

- Equifax 入侵 76 天才发现
- Target 数据泄漏 → 信用卡公司告知后才知

### 应该 log 的事件

- 登录成功 / 失败
- 权限变更
- 关键资源访问 / 修改
- 异常错误
- 重要业务操作（支付、转账）

### 不该 log

- 密码、token
- 完整信用卡号
- PII（部分场景）

### 监控

- 异常登录模式
- 权限提升
- 速率突变
- 错误率飙升

详见 B24 可观测性。

---

## A10：服务端请求伪造（SSRF）

### 含义

服务端代用户发请求 → 攻击者控制目标 URL → 访问内部资源。

### 经典场景

```python
# 用户提供 URL，服务端下载
def fetch_avatar(url):
    return requests.get(url)

# 攻击者输入: http://169.254.169.254/latest/meta-data/  (AWS 元数据)
# 或: http://internal-admin.local/users
```

### 防御

- URL 白名单（允许的 host）
- 禁止内网 / 元数据地址（127.x、10.x、169.254.x、metadata.google.internal）
- DNS rebind 防护（解析后比对 IP）
- 出网代理 + 限制
- 单独 network namespace

```python
def safe_fetch(url):
    p = urlparse(url)
    if p.hostname in BLACKLIST or is_private_ip(resolve(p.hostname)):
        raise SecurityError
    return requests.get(url, timeout=5)
```

---

## 第十一章：XSS（跨站脚本）—— 严格不在 Top 10 单独列了

但仍是 web 重大风险。

### 类型

- **Reflected**：URL 参数注入到响应 HTML
- **Stored**：恶意脚本存到 DB（评论），他人访问触发
- **DOM-based**：JS 处理用户输入插入 DOM

### 防御

- 输出 escape（按上下文：HTML / attribute / JS / URL）
- 模板引擎默认 escape（Jinja、React {} 等）
- Content Security Policy header
- HttpOnly cookie 防 cookie 偷

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
```

---

## 第十二章：CSRF

### 含义

诱导用户点击恶意链接，浏览器自动带 cookie → 攻击者代用户操作。

### 防御

- SameSite cookie（参考 B22）
- CSRF token（form 中藏 token，server 验证）
- 验证 Origin / Referer
- 关键操作要求重新验证（如改密码要求当前密码）

---

## 第十二章半：OWASP API Security Top 10 (2023) —— API-first 服务必读

> 通用 OWASP Top 10 主要面向"传统 Web 应用 + form 提交"。**API 服务（REST/GraphQL/gRPC）的攻击面不同**——OWASP 单独维护一份 API Security Top 10，2023-06 发布最新版（目前最新有效版本）。

| # | 名称 | 一句话本质 |
|---|---|---|
| **API1** | **Broken Object Level Authorization (BOLA)** | `GET /orders/123` 没检查"123 是不是当前用户的"——占所有 API 攻击 ~40% |
| **API2** | **Broken Authentication** | JWT 不验签、token 不过期、API key 暴露在 git；含 OAuth/服务间认证 |
| **API3** | **Broken Object Property Level Authorization** | 返回了 `is_admin` 字段、或 `PATCH /user` 接受 `role` 字段（旧"Excessive Data Exposure" + "Mass Assignment" 合并） |
| **API4** | **Unrestricted Resource Consumption** | 没有 rate limit / payload size limit / 复杂度限制 → DoS / 大账单 |
| **API5** | **Broken Function Level Authorization (BFLA)** | `POST /admin/users` 没鉴权；普通用户能调管理员端点 |
| **API6** | **Unrestricted Access to Sensitive Business Flows** ⭐ NEW | 票务秒杀、积分兑换、点赞 —— 不是 bug 是被自动化滥用；要业务流防刷 |
| **API7** | **Server-Side Request Forgery (SSRF)** ⭐ NEW | API 接受 URL 参数去 fetch；在云上能拿到 IMDS metadata 凭证 |
| **API8** | **Security Misconfiguration** | 默认 cred、CORS `*`、verbose error、debug endpoint 上线 |
| **API9** | **Improper Inventory Management** | 老版本 API（`/v1/`）忘下线、staging 环境暴露在公网、影子 API |
| **API10** | **Unsafe Consumption of APIs** ⭐ NEW | 你**调**第三方 API 时盲目信任返回值（没校验/没超时/没大小限制） |

### 与 2019 版的关键变化

- **Injection 移除**——不是攻击消失了，而是合并到 API8 (Security Misconfiguration)，因为 API 注入主要靠正确的参数化 + 框架默认防护。
- **新增 API6 / API7 / API10**——分别覆盖业务流滥用、SSRF（云时代的高危）、调用第三方时的盲信任。
- **Mass Assignment + Excessive Data Exposure 合并**为 API3（同源问题：对象属性级授权缺失）。

### 防御速查（按层）

| 层 | 必备措施 |
|---|---|
| **网关 / WAF** | rate limit、payload size、JWT 验签前置、IP / geo 风控 |
| **路由 / RPC 框架** | 中间件统一鉴权（不是每个 handler 自己写）、自动 schema validation |
| **业务逻辑** | 每个 `:id` / 资源标识都做"属于当前用户?"检查；DTO 严格白名单字段 |
| **依赖** | 调外部 API 必带 timeout / size cap / SSRF 黑名单（IMDS、内网段） |
| **运维** | API inventory（OpenAPI / GraphQL schema 全收集）、过期版本下线、staging 不公开 |

> 参考：[OWASP API Security Top 10 2023](https://owasp.org/API-Security/editions/2023/en/0x00-header/)、配套 [API Security Project](https://owasp.org/www-project-api-security/)。

---

## 第十三章：generic 防护清单

### 13.1 所有应用都该有

1. HTTPS only
2. Security headers：
   ```
   Strict-Transport-Security: max-age=31536000
   X-Content-Type-Options: nosniff
   X-Frame-Options: DENY
   Content-Security-Policy: ...
   Referrer-Policy: strict-origin-when-cross-origin
   ```
3. 限流 + WAF
4. 依赖扫描
5. 安全 baseline 配置
6. 定期 pen-test

### 13.2 framework 选好

Django、Rails、Spring Boot 等成熟框架默认抵御大多数 OWASP——除非你乱配。手写的小框架更可能踩坑。

---

## 第十四章：生产级最佳实践

1. **threat model 每个新 feature**：想"怎么被坑"。
2. **OWASP ZAP / Burp 扫描**：CI 集成。
3. **CSP + CORS 严格**：不要 * + credentials。
4. **依赖扫描自动化**：Dependabot / Snyk。
5. **secret 不入 git**：Vault / Secrets Manager。
6. **日志 + 监控异常模式**：登录失败暴增等。
7. **测试 abuse case**：不仅 happy path。
8. **定期 pen-test**：年度内外部。
9. **bug bounty**：HackerOne 公开收。
10. **incident response 流程**：泄漏后第一时间响应。

---

## 第十五章：常见陷阱清单

### ❌ 陷阱 1：信任客户端
"前端校验过了" + "JWT 内含 role 字段" → 后端必复检。

### ❌ 陷阱 2：日志含 secret
排查时帮自己 = 帮攻击者。

### ❌ 陷阱 3：error message 暴露
"SQL syntax error near 'DROP TABLE'" → 知道你用什么 DB。统一 generic message。

### ❌ 陷阱 4：debug mode 生产
栈 trace、敏感配置全暴露。

### ❌ 陷阱 5：忘了 patch 依赖
log4j 出来后还有人 6 月没升级。

### ❌ 陷阱 6：API 没限流
枚举 user_id 探测 → 拿到全用户。

### ❌ 陷阱 7：reset token 长期有效
一次性 + 短 TTL（10 min）+ 用一次就废。

---

## 第十六章：练习题

**练习 1**：以下代码有什么注入风险？修复。
```python
def search(keyword):
    return db.execute(f"SELECT * FROM products WHERE name LIKE '%{keyword}%'")
```

**练习 2**：用户能 GET /api/users/{id} 看到任意用户邮箱——什么 OWASP 类别？怎么修？

**练习 3**：用户上传图片，server 用 ImageMagick 转换 → 历史上多次 RCE（imagetragick）。怎么防？

**练习 4**：解释 SSRF 如何让攻击者拿到云服务器的临时凭据。

**练习 5**：CSP 头如何防 XSS？给一个 strict 配置例子。

---

## 参考答案

**练习 1**：SQL 注入。输入 `' OR 1=1; --` → 返回全部 + 后续语句。修：
```python
db.execute("SELECT * FROM products WHERE name LIKE ?", (f"%{keyword}%",))
```
或用 ORM：`Product.objects.filter(name__icontains=keyword)`。

**练习 2**：A01 Broken Access Control（IDOR）。修：
- 后端校验"id 属于当前 user 或当前 user 有权限"
- 邮箱字段返回前判断："仅自己 / 管理员可见"
- 使用 UUID + 后端 ACL（不依赖混淆 ID 防御）

**练习 3**：
- 沙箱隔离（gVisor、Firecracker、Docker user namespace）
- 用更安全的库（libvips、Sharp）
- 限制文件类型 + magic number 检查
- 限资源（CPU / memory / time）
- 关闭 IM 不必要的 coders：`/etc/ImageMagick/policy.xml` 禁用 EPHEMERAL 等危险类型

**练习 4**：AWS EC2 有 metadata endpoint：
```
http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>
```
返回临时 access key / secret / session token。SSRF 让 server 访问这 URL → 攻击者拿到凭据 → 调任何 AWS API。
（IMDSv2 通过要求 token 缓解，但仍要后端防护。）

**练习 5**：
```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```
- `default-src 'self'`：默认只从自家域加载
- `script-src` 限制 JS 来源
- 攻击者注入的 `<script src="evil.com/...">` 被 CSP 拒绝
- 即使 `<script>alert(1)</script>` 也被拒（除非加 `'unsafe-inline'`）

---

## 小结

| OWASP | 关键 |
|---|---|
| A01 Access Control | 后端必复检所有权 |
| A02 Crypto | bcrypt 密码、TLS 全程 |
| A03 Injection | 参数化查询 |
| A04 Insecure Design | 威胁建模 |
| A05 Misconfig | 默认拒绝、最小化 |
| A06 Vulnerable Components | 依赖扫描 + 升级 |
| A07 Authn Failure | 强密码 + lockout + MFA |
| A08 Integrity | 不反序列化用户数据 |
| A09 Logging | 记关键事件 + 监控 |
| A10 SSRF | URL 白名单 + 内网阻断 |

下一篇 **B24 — 可观测性** 将拆开 logs、metrics、traces 三柱、OpenTelemetry、SLI/SLO。

---

