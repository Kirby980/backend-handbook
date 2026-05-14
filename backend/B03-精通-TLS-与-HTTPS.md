# 精通 TLS、HTTPS 与证书管理

> 课程编号：B03
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — TLS/HTTPS
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：HTTPS 是默认而非奢侈

```
http://example.com   →   不安全
https://example.com  →   安全
```

2016 年前 HTTPS 还被视为"金融/登录页才需要"；今天，浏览器把 HTTP 标红、Let's Encrypt 提供免费证书、HTTP/2 和 HTTP/3 默认需要 TLS。**所有公网服务都该 HTTPS**。这一章讲清 TLS 握手机制、证书链、Cipher suite、SNI、0-RTT、HSTS、生产配置与陷阱。

---

## 第一章：TLS 解决什么问题

### 1.1 三大目标

1. **机密性**（confidentiality）：第三方无法窃听
2. **完整性**（integrity）：第三方无法篡改
3. **认证**（authentication）：客户端确认对端真的是 example.com

没有 TLS：HTTP 明文走在网线上，咖啡馆 Wi-Fi 任意人都能读取你的密码。

### 1.2 TLS vs SSL

历史上有 SSL 1/2/3 → TLS 1.0/1.1/1.2/1.3。SSL 全部已淘汰；TLS 1.0/1.1 也已废弃；**生产用 TLS 1.2 或 1.3**。

### 1.3 端口

- HTTPS：443
- HTTP/3（QUIC）：443/UDP

---

## 第二章：TLS 1.3 握手

### 2.1 简化的 1-RTT 握手

```
Client                              Server
  | ── ClientHello ─────────────────→ |
  |    (含 supported ciphers, key share, SNI)
  |                                   |
  | ←── ServerHello ────────────────── |
  |     (选 cipher, key share)
  | ←── EncryptedExtensions ────────── |
  | ←── Certificate ─────────────────── |
  | ←── CertificateVerify ────────────── |
  | ←── Finished ───────────────────── |
  |                                   |
  | ── Finished ────────────────────→ |
  |                                   |
  | ←─ Application Data ──────────────→ |
```

仅 1 个 RTT 协商完密钥与身份，第二个 RTT 已经传业务数据。比 TLS 1.2 节省 1 个 RTT。

### 2.2 0-RTT（早期数据）

客户端已经连过这服务器、有 PSK（pre-shared key），可以在第一个包**就携带应用数据**：

```
Client                              Server
  | ── ClientHello + early data ─────→ |
  |                                   |
  | ←── ServerHello + ... ───────────── |
```

延迟接近 TCP 三次握手——极快。但有**重放攻击风险**——同一请求可能被重发。所以 0-RTT 只应用于幂等请求（GET）。

### 2.3 关键阶段

- **协商**：双方告诉对方支持哪些 cipher / curve / version
- **密钥交换**：用 ECDHE 在不可信信道协商共享密钥
- **身份验证**：服务端发证书；客户端验证证书链
- **finished**：双方确认握手没被篡改

### 2.4 后量子密钥交换：X25519MLKEM768（2026 现状）

> **如果你在 2026 年才搭新系统，密钥交换默认就该开后量子混合算法。** 这不是边缘特性，是浏览器 + CDN 的事实标准。

为什么现在就要后量子？**SNDL（Store Now, Decrypt Later）攻击**：对手现在抓取你的 TLS 流量存档，等量子计算机能跑 Shor 算法时（学界估计 2030 年代）回头解密。今天的 X25519、ECDHE、RSA 都会被一次性破解。**应对：在密钥协商阶段，把传统椭圆曲线和后量子 KEM 同时跑，最终密钥从两者派生**——任一算法仍安全，整体就安全。

主流方案是 **`X25519MLKEM768`**（IANA NamedGroup `0x11ec`）：

- **X25519**：经典 ECDHE，今天已知安全
- **ML-KEM-768**（FIPS 203 / Kyber 标准化产物）：NIST 选定的后量子 KEM 标准

部署现状（2026-05）：

| 实体 | 状态 |
|---|---|
| **Chrome / Edge** | ≥ 131 (2024-12) 默认；≥ 138 用户无法禁用 |
| **Firefox** | ≥ 132 (2024-11) 默认；135+ 在 QUIC/HTTP3 也开启 |
| **Cloudflare** | 已默认启用 |
| **Akamai** | 2025-09 客户端 → 边缘普及；origin 端 2025-06 已支持 |
| **Apple Safari** | iOS 26 / macOS Tahoe 26 (2025 秋) 起支持 |
| **Go `crypto/tls`** | Go 1.24+ 支持 ML-KEM-768；Go 1.26 新增 `crypto/hpke` 含后量子混合 KEM |

**给后端的实践**：

1. **服务端启用 ML-KEM-768**：用 OpenSSL 3.5+、BoringSSL 最新版、Go 1.24+ 的 `crypto/tls`、Nginx 1.27+ + 对应 OpenSSL 即可。
2. **不要禁用经典曲线**——混合方案就是为了"后量子失败时仍有 X25519 兜底"。
3. **观察握手大小**：ML-KEM-768 公钥 + 密文比 X25519 大约 1184 + 1088 字节，整个 ClientHello/ServerHello 显著膨胀。某些老中间盒（旧 DPI、防火墙）会因为单包超 1500 MTU 而丢包——所以发布前用 `openssl s_client -groups X25519MLKEM768 -connect ...` 测一遍真实路径。
4. **HTTP/3 ECH 也用同一组算法**——参见 6.3 节 ECH。

> 参考：[Cloudflare — Post-Quantum Key Agreement](https://pq.cloudflareresearch.com/)、[Chromium — PQC Update](https://www.chromium.org/Home/chromium-security/post-quantum-cryptography/)、[FIPS 203 ML-KEM](https://csrc.nist.gov/pubs/fips/203/final)。

---

## 第三章：证书

### 3.1 证书包含什么

```
Subject: CN=example.com
Issuer: Let's Encrypt R3
Validity: 2026-01-01 to 2026-04-01
Subject Alternative Name (SAN):
  - DNS:example.com
  - DNS:www.example.com
  - DNS:*.api.example.com  ← 通配符
Public Key: <RSA-2048 or ECDSA-P256>
Signature: <CA's signature over the above>
```

### 3.2 证书链

```
Root CA (Let's Encrypt ISRG Root X1)
  └─ Intermediate (Let's Encrypt R3)
       └─ Your cert (example.com)
```

服务器要发送**叶子证书 + 中间证书**，客户端通过预装的 Root 验证整条链。**漏发中间证书** → "证书不受信" 错误（虽然实际证书没问题）。

### 3.3 SAN（Subject Alternative Name）

现代证书用 SAN 列出所有支持的域名。CN（Common Name）已废弃用法。**一张证书可覆盖多个域名**——通配符（`*.example.com`）或显式列举。

### 3.4 通配符与 multi-domain

- `*.example.com`：匹配 `a.example.com`、`b.example.com`，**不**匹配 `example.com` 或 `a.b.example.com`
- 一张证书可包含多个非相关域（SAN）

### 3.5 ECDSA vs RSA

- **RSA 2048**：兼容性最广；签名/验证慢
- **ECDSA P-256**：更快、key 更小；现代主流
- 双证书部署：服务端同时挂 ECDSA + RSA，按客户端能力发对应证书

---

## 第四章：证书颁发与管理

### 4.1 Let's Encrypt

免费、自动化、90 天有效期。两种验证方式：

- **HTTP-01**：CA 访问 `http://example.com/.well-known/acme-challenge/<token>`
- **DNS-01**：在 DNS 加 TXT 记录 `_acme-challenge.example.com`

DNS-01 支持通配符证书。

### 4.2 自动续期

90 天有效期 → 必须自动续期。常用工具：

- **certbot**：标准客户端
- **acme.sh**：纯 shell 实现
- **cert-manager**：K8s 内自动签发与轮换
- **Caddy**：内置自动 TLS

```bash
certbot certonly --webroot -w /var/www -d example.com
# crontab: 0 0,12 * * * certbot renew --quiet
```

### 4.3 商业 CA vs Let's Encrypt

商业 CA（DigiCert、Sectigo）：
- EV（Extended Validation）证书（浏览器显示组织名——但 Chrome 已淡化）
- 更长有效期（1-2 年）
- 支持非 HTTP（如代码签名）
- 客服

90% 场景 Let's Encrypt 够用。

### 4.4 私有 CA（内部服务）

内部微服务用 mTLS 时，自建 CA：
- HashiCorp Vault PKI
- SmallStep step-ca
- cert-manager + 自签 CA
- Cloudflare Origin CA

服务实例预装 CA cert 信任链。

---

## 第五章：Cipher suite

### 5.1 命名格式（TLS 1.2）

```
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
    密钥交换   认证   加密             MAC
```

### 5.2 TLS 1.3 简化

TLS 1.3 强制 ECDHE + AEAD，cipher suite 只剩加密算法：

```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
```

ChaCha20 在 ARM / 移动设备上比 AES-GCM 快（无硬件加速时）。

### 5.3 禁用旧 cipher

不允许：
- 静态 RSA 密钥交换（无前向保密）
- 3DES、RC4、MD5
- CBC 模式（除非必须）

### 5.4 前向保密（Forward Secrecy）

ECDHE 每次握手生成临时密钥——即使服务器私钥日后泄漏，**旧流量也无法解密**。这就是为何 ECDHE 必备。

---

## 第六章：SNI（Server Name Indication）

### 6.1 问题

一个 IP 可以挂多个域名。TLS 握手时服务器还不知道你要访问哪个域，无法选对的证书。

### 6.2 SNI 解决

`ClientHello` 携带目标域名（明文）：

```
ClientHello:
  server_name: example.com
```

服务器据此选证书。

### 6.3 SNI 明文 → ECH（Encrypted Client Hello）

SNI 明文暴露你访问哪个域——**直接被审查 / 监控 / 被动指纹识别**。

**ECH（Encrypted Client Hello）** 是 ESNI 的演进版本。不再只加密 SNI，而是**把整个 ClientHello（含扩展、ALPN、cipher list 等所有指纹位）加密**——窃听者只看到一个空壳，无法识别访问的具体网站。

**2026 现状**：

- **Cloudflare**：所有客户域名默认 ECH 已启用，公钥通过 DNS HTTPS RR 分发
- **Firefox** ≥ 119：默认开启 ECH（DoH 模式下）
- **Chrome** ≥ 123：默认开启 ECH（HTTPS RR 命中时）
- **服务端**：BoringSSL、wolfSSL 已经支持；OpenSSL 3.5+ 实验支持
- **DNS**：必须发布 `HTTPS` RR 携带 `ech=...` 公钥（base64-encoded ECHConfigList）

部署前提：你的 DNS 记录 + TLS 终端必须协同部署 ECH 公钥；中间没有 SNI-based 路由（如某些 LB）才能起作用。

> 参考：[draft-ietf-tls-esni（即将定稿为 RFC）](https://datatracker.ietf.org/doc/draft-ietf-tls-esni/)、[Cloudflare ECH](https://blog.cloudflare.com/announcing-encrypted-client-hello/)。

---

## 第七章：HSTS——强制 HTTPS

### 7.1 头部

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

浏览器看到后：1 年内（max-age）所有 example.com 流量自动走 HTTPS。即使用户输入 http://example.com，浏览器内部改成 https://。

### 7.2 preload list

Chrome / Firefox 内置"绝对走 HTTPS"的域名清单。提交：

https://hstspreload.org

要求：
- HTTPS 服务正常
- 所有子域名也 HTTPS
- max-age ≥ 1 年
- includeSubDomains

加入后**无法快速撤回**——慎用。

### 7.3 注意

第一次访问的请求仍可能是 HTTP（在 HSTS 头返回前）。preload 解决这个 first-trip 攻击窗口。

---

## 第八章：mTLS（Mutual TLS）

### 8.1 双向认证

普通 TLS 仅服务端展示证书；mTLS 让**客户端**也展示证书：

```
ClientHello → Server
Server → ClientHello + Certificate
Client → Certificate + Finished
```

服务端验证客户端证书是否由内部 CA 签发 → 通过。

### 8.2 使用场景

- 微服务之间（service mesh 内 sidecar 自动 mTLS）
- 高安全 API（金融、医疗）
- IoT 设备
- API gateway 间

### 8.3 vs Token 认证

- Token：在应用层
- mTLS：在传输层
- 可叠加：mTLS 确认服务身份 + Token 确认用户身份

---

## 第九章：HTTPS 部署

### 9.1 nginx 示例

```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;
    ssl_prefer_server_ciphers off;

    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 valid=300s;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # HTTP → HTTPS 重定向
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### 9.2 OCSP Stapling

证书吊销检查：客户端在握手中验证证书没被吊销。
- OCSP（Online Certificate Status Protocol）：直接查 CA（慢、隐私问题）
- OCSP Stapling：服务端预先取 OCSP response 附在握手里（推荐）

### 9.3 Cipher suite 排序

Mozilla SSL Configuration Generator (`https://ssl-config.mozilla.org/`) 给出经过验证的最佳实践。选 Intermediate（兼容性 + 安全平衡）或 Modern（仅 TLS 1.3 客户端）。

---

## 第十章：测试与诊断

### 10.1 在线工具

- **SSL Labs**：`https://www.ssllabs.com/ssltest/`——综合评分 A+ 起步
- **Mozilla Observatory**：综合安全头部检查

### 10.2 命令行

```bash
# 看证书
openssl s_client -connect example.com:443 -servername example.com
echo | openssl s_client -connect example.com:443 -showcerts 2>/dev/null

# 单独看证书内容
openssl x509 -in cert.pem -noout -text

# 验证证书链
openssl verify -CAfile chain.pem cert.pem

# 测试 cipher
nmap --script ssl-enum-ciphers -p 443 example.com

# 测试支持的 TLS 版本
openssl s_client -tls1_3 -connect example.com:443
```

### 10.3 常见错误

- `ERR_CERT_AUTHORITY_INVALID`：未挂中间证书
- `ERR_CERT_DATE_INVALID`：证书过期
- `ERR_CERT_COMMON_NAME_INVALID`：CN/SAN 不匹配域名
- `ERR_SSL_VERSION_OR_CIPHER_MISMATCH`：客户端与服务端无共同协议/cipher

---

## 第十一章：生产级最佳实践

1. **强制 HTTPS**：HTTP 302 → HTTPS；加 HSTS。
2. **自动证书管理**：cert-manager / Caddy / certbot cron。
3. **TLS 1.2 + 1.3，禁用 1.0/1.1**。
4. **现代 cipher suite**：参考 Mozilla generator。
5. **ECDSA P-256 优先**（+ RSA 2048 兼容）。
6. **OCSP Stapling 开启**：握手快。
7. **HSTS preload**：高价值域名提交。
8. **隔离私钥权限**：服务账户 only。
9. **mTLS 内部服务**：服务网格（Istio、Linkerd）自动化。
10. **定期 SSL Labs 测试**：监控配置漂移。

---

## 第十二章：常见陷阱清单

### ❌ 陷阱 1：忘记中间证书
浏览器看 "untrusted"。一定 nginx ssl_certificate 用 `fullchain.pem` 而非 `cert.pem`。

### ❌ 陷阱 2：证书过期不告警
设监控提前 14 天告警续期。

### ❌ 陷阱 3：通配符证书覆盖范围误解
`*.example.com` 不覆盖 `example.com` 自己；要加 SAN。

### ❌ 陷阱 4：HSTS preload 撤不回
一旦加入 Chrome list 慢慢消失需数月。先小 max-age 测试。

### ❌ 陷阱 5：mTLS 证书也会过期
轮换流程要有；cert-manager 自动化。

### ❌ 陷阱 6：私钥泄漏不撤销证书
泄漏立刻向 CA revoke + 重签 + 改密码。

### ❌ 陷阱 7：内网用自签证书但不验证
开发期开"忽略证书"代码混入生产 → MITM 风险。

---

## 第十三章：练习题

**练习 1**：解释 TLS 1.3 比 1.2 快在哪。

**练习 2**：拥有 `*.api.example.com` 通配符证书，可以保护哪些域名？

**练习 3**：为何 HSTS preload 后撤回困难？

**练习 4**：用 openssl 命令查看 example.com 证书的 SAN。

**练习 5**：mTLS 与 OAuth 2.0 token 各保护什么？能同时用吗？

---

## 参考答案

**练习 1**：
- 握手 RTT 从 2 减为 1
- 支持 0-RTT 重连
- 强制 ECDHE，移除 RSA 密钥交换
- AEAD only，简化 cipher
- 加密更多握手数据

**练习 2**：`a.api.example.com`、`b.api.example.com`。**不**覆盖 `api.example.com`（自身）、`x.y.api.example.com`（两级子域）、`example.com`。

**练习 3**：preload list 编译进浏览器二进制；一旦发布，所有 Chrome/Firefox 用户的更新滚动数月才完成。撤回意味着一段时间内仍强制 HTTPS。

**练习 4**：
```bash
openssl s_client -connect example.com:443 -servername example.com < /dev/null 2>/dev/null \
  | openssl x509 -noout -text \
  | grep -A 1 "Subject Alternative Name"
```

**练习 5**：
- mTLS：保护**传输**——确认双方机器身份、防中间人
- OAuth token：保护**用户身份**——确认调用者代表谁

可同时用——mTLS 限制只有受信 client 能连，token 标识具体 user。常见微服务 + 用户 API 组合。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| TLS 1.3 | 1-RTT 握手 + 0-RTT 重连 |
| 证书 | leaf + intermediate + root；SAN 列域名 |
| Let's Encrypt | 免费 90 天；自动续期必备 |
| Cipher | ECDHE + AEAD；禁用 CBC/RC4/MD5 |
| SNI | ClientHello 明文域名；ECH 加密中 |
| HSTS | max-age + includeSubDomains + preload |
| mTLS | 双向认证；服务网格自动化 |
| Stapling | OCSP 预附在握手 |

下一篇 **B04 — 精通 REST API 设计** 会讲清 resource 建模、版本化、分页、排序、过滤、HATEOAS。

---

