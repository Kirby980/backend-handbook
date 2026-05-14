# 精通 Web 服务器与反向代理：Nginx、Caddy、Envoy

> 课程编号：B25
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — Web Servers
> 难度：⭐⭐⭐⭐
> 预计阅读时间：50 分钟

---

## 引言：应用服务器前面那一层

```
Client → ?? → Application
```

那个 `??` 是 Nginx、Caddy、HAProxy、Envoy、Traefik 之一。它做什么？为什么不直接用 Go / Node 的内置 server？本章拆开各产品差异、典型配置、生产实践。

---

## 第一章：为什么需要反向代理

### 1.1 应用 server 也能直接监听

```go
http.ListenAndServe(":80", handler)
```

但加一层反向代理能：

- **TLS 终止**：集中管证书
- **负载均衡**：N 个应用实例
- **gzip/br 压缩**：CPU 卸载
- **静态资源**：直接 sendfile，不进应用
- **限流 + WAF**：前置防护
- **缓存**：减少应用压力
- **路径路由**：多 app 共享域名
- **协议转换**：HTTP/3 → HTTP/1.1 应用

### 1.2 应用层职责减轻

应用只关心业务；TLS、压缩、缓存、HSTS header 等基础设施由前置代理处理。

---

## 第二章：Nginx

### 2.1 简介

事实标准。C 写，事件驱动（epoll/kqueue），单进程 + 多 worker。

### 2.2 配置结构

```nginx
worker_processes auto;

events { worker_connections 1024; }

http {
    upstream backend {
        server 10.0.1.1:8080;
        server 10.0.1.2:8080;
    }

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        ssl_certificate /etc/letsencrypt/live/.../fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/.../privkey.pem;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /static/ {
            root /var/www;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }

    server {
        listen 80;
        server_name api.example.com;
        return 301 https://$server_name$request_uri;
    }
}
```

### 2.3 关键配置

**Worker / connections**：
```
worker_processes auto;          # CPU 核数
worker_connections 4096;        # 每 worker 最大连接
```
总并发约 = worker × connections。OS `ulimit -n` 也要足。

**超时**：
```
client_body_timeout 12;
client_header_timeout 12;
keepalive_timeout 65;
send_timeout 10;
```

**gzip**：
```
gzip on;
gzip_types text/plain application/json application/javascript text/css;
gzip_min_length 1024;
```

**Brotli**（要 module）：
```
brotli on;
brotli_types ...;
```

### 2.4 负载均衡算法

```nginx
upstream backend {
    # round_robin (默认)
    # least_conn;        ← 连接最少
    # ip_hash;           ← 按 IP 哈希（sticky）
    # hash $request_uri; ← 按 URI 哈希
    
    server 10.0.1.1:8080 weight=3;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080 backup;
}
```

### 2.5 health check

Nginx OSS：被动（请求失败标记宕）。
Nginx Plus（付费）/ 第三方 module：主动健康检查。

### 2.6 HTTP/3

```
listen 443 quic reuseport;
listen 443 ssl http2;
add_header Alt-Svc 'h3=":443"; ma=86400';
```

Nginx **1.25.0** (2023-04) 起内置 QUIC + HTTP/3，2026 stable 是 1.27.x。

### 2.7 NGINX One 与 F5 治理（2024-2025 重要变化）

- **2024-09**：F5（Nginx 母公司）发布 **NGINX One**——把 NGINX OSS、NGINX Plus、统一管理控制台 (NGINX One Console)、telemetry 整合为一个 SaaS 产品。
- **2024-11 NGINX Plus R33**：开始要求 JWT license 文件 + 周期性使用上报；**OSS 版（nginx.org 编译版）不受影响**——但企业用 Plus 要规划合规。
- **2024-03**：F5 把 ModSecurity WAF 标记 EOL（Trustwave 2024-07 停止维护 ModSecurity）。生产 WAF 改用 Coraza、CRS 直跑、或 Cloudflare/Fastly 等托管 WAF。
- **NGINX OSS 主线持续开源活跃**——开源版本在 1.27.x（含 HTTP/3、experimental QUIC tuning）。

> 参考：[F5 NGINX One](https://www.f5.com/products/nginx/one)、[NGINX Plus R33 release](https://docs.nginx.com/nginx/releases/)。

---

## 第三章：Caddy

### 3.1 简介

Go 写。**自动 HTTPS**——零配置 Let's Encrypt 证书 + 自动续期。

### 3.2 Caddyfile 配置

```
api.example.com {
    reverse_proxy 10.0.1.1:8080 10.0.1.2:8080 {
        lb_policy round_robin
        health_uri /health
        health_interval 10s
    }

    handle /static/* {
        root * /var/www
        file_server
    }

    encode gzip zstd
    
    header {
        Strict-Transport-Security "max-age=31536000"
        X-Content-Type-Options nosniff
    }
}
```

对比 Nginx 文件大幅减少。

### 3.3 优点

- **零证书配置**：自动 ACME + DNS-01 / HTTP-01
- **HTTP/3 默认开**
- **配置简单**
- **modular**（plugin via Caddy 2）

### 3.4 适合

- 新项目、个人 / 小型
- 不想花时间在 TLS / config 的项目
- Docker / K8s ingress 替代

### 3.5 不适合

- 极致性能（Nginx 略快）
- 已有大量 Nginx 配置生态依赖

---

## 第四章：HAProxy

### 4.1 简介

老牌 L4/L7 LB，性能极强。专注负载均衡（不做静态文件 / TLS 默认要 enterprise 才好用，新版有改善）。

### 4.2 配置

```haproxy
frontend api
    bind *:443 ssl crt /etc/ssl/cert.pem
    default_backend app_servers

backend app_servers
    balance roundrobin
    option httpchk GET /health
    server app1 10.0.1.1:8080 check
    server app2 10.0.1.2:8080 check
```

### 4.3 优点

- L4 性能极佳
- 详细 stats UI
- 平滑 reload（不丢连接）
- 复杂路由 / 健康检查规则

### 4.4 适合

- L4 高吞吐（数据库代理 / TCP 流量）
- 复杂 health check 逻辑
- 已运维 HAProxy 团队

---

## 第五章：Envoy

### 5.1 简介

CNCF。Lyft 开发。现代 L7 代理，C++ 写，xDS 动态配置。

### 5.2 特点

- **动态配置**：通过 xDS API 实时下发，无需 reload
- **HTTP/2 + gRPC first class**
- **超丰富 filter**：rate limit、auth、WASM
- **observability**：metric / log / trace 全内置

### 5.3 配置（YAML）

```yaml
static_resources:
  listeners:
    - address: { socket_address: { address: 0.0.0.0, port_value: 443 } }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                stat_prefix: ingress
                route_config:
                  virtual_hosts:
                    - name: api
                      domains: ["api.example.com"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: backend }
                http_filters:
                  - name: envoy.filters.http.router
  clusters:
    - name: backend
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      load_assignment:
        cluster_name: backend
        endpoints: [ ... ]
```

复杂。但功能强。

### 5.4 何时用 Envoy

- Service Mesh（Istio、Consul Connect 底层都是 Envoy）
- API Gateway（Emissary、Gloo）
- 动态路由 / canary deployment 频繁
- 多协议（gRPC、HTTP/3、TCP）

### 5.5 不适合

- 简单网站
- 团队没人懂 Envoy 配置（学习曲线陡）

---

## 第六章：Traefik

### 6.1 简介

Go 写，专为容器生态设计。自动从 Docker / K8s / Consul 拉服务发现。

### 6.2 K8s Ingress 配置

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
spec:
  routes:
    - match: Host(`api.example.com`)
      kind: Rule
      services:
        - name: my-service
          port: 8080
```

应用部署 → service 加 label / annotation → Traefik 自动配置。

### 6.3 优点

- 自动服务发现
- 简单配置
- 自动 HTTPS
- 友好 dashboard

### 6.4 适合

- K8s ingress 控制器
- Docker Compose 多服务
- 频繁部署 / canary

---

## 第七章：选型

### 7.1 决策树

```
个人 / 小项目 → Caddy（零配置 HTTPS）
传统部署 / 已有 Nginx → Nginx
K8s ingress → Nginx Ingress / Traefik / Caddy / Contour
Service Mesh → Envoy (Istio / Linkerd)
极致 LB 性能 → HAProxy（特别 L4）
API Gateway → Envoy / Kong / Tyk
```

### 7.2 实际栈

很多公司多个一起用：
- Edge：CDN（Cloudflare）
- Ingress：Nginx 或 Traefik
- East-west（服务间）：Envoy sidecar（Istio）
- L4 LB：cloud LB（AWS NLB / ELB）

---

## 第八章：关键配置主题

### 8.1 X-Forwarded-* headers

```
X-Forwarded-For: client-ip, proxy1-ip, proxy2-ip
X-Forwarded-Proto: https
X-Real-IP: client-ip
```

应用通过这些 header 知道真实客户端 IP / 协议。注意：**信任链**——只信任你的代理设置的，不信客户端发的（用户可以伪造 X-Forwarded-For）。

### 8.2 keep-alive 到上游

```
upstream backend {
    server ...;
    keepalive 32;       # 与上游保持 32 个 keep-alive 连接
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
}
```

让代理与应用之间复用连接——大幅减少 TCP / TLS 握手开销。

### 8.3 buffering

```
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 4k;
```

代理缓冲整个响应再返回 client（适合慢 client）。流式响应（SSE）要关：
```
proxy_buffering off;
```

### 8.4 超时

```
proxy_connect_timeout 5s;
proxy_send_timeout 30s;
proxy_read_timeout 30s;
```

应配合上游配置。

### 8.5 限流

```
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

location /api {
    limit_req zone=api burst=200 nodelay;
    proxy_pass ...;
}
```

100 QPS + burst 200。超 burst 返 503。

---

## 第九章：HTTPS 配置

### 9.1 现代 cipher suite

参考 Mozilla SSL Configuration Generator。Intermediate 兼容性 + 安全平衡。

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:...;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
```

### 9.2 HSTS

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

### 9.3 多证书 SNI

```nginx
server {
    listen 443 ssl;
    server_name a.com;
    ssl_certificate /etc/ssl/a.com/fullchain.pem;
    ...
}
server {
    listen 443 ssl;
    server_name b.com;
    ssl_certificate /etc/ssl/b.com/fullchain.pem;
    ...
}
```

Nginx 根据 SNI 选证书。

### 9.4 OCSP stapling

参考 B03。

---

## 第十章：可观测性

### 10.1 Access log

```nginx
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                'rt=$request_time uct="$upstream_connect_time" '
                'urt="$upstream_response_time"';

access_log /var/log/nginx/access.log main;
```

字段：
- `$request_time`：总耗时
- `$upstream_response_time`：后端耗时
- 差值大 → 网络或代理本身问题

### 10.2 Prometheus exporter

- nginx-prometheus-exporter
- Caddy 内置 metric
- Envoy 内置 stats

监控：
- requests rate / errors
- upstream latency
- active connections
- 4xx / 5xx 分布

### 10.3 实时调试

```bash
# 抓某用户的 traffic
tcpdump -A -i any port 80 | grep "user=42"

# nginx
nginx -t                          # 配置检查
nginx -s reload                   # 重载
```

---

## 第十一章：生产级最佳实践

1. **HTTPS only + HSTS**。
2. **HTTP/2 默认 + HTTP/3 可选**。
3. **上游 keep-alive**：减建连开销。
4. **gzip / brotli**：响应小 80%。
5. **静态资源直接 sendfile + immutable cache**。
6. **超时分级**：client < proxy < upstream。
7. **限流 + WAF**：前置防御。
8. **access log + metric**：流量可见。
9. **零停机 reload**：Nginx -s reload / Caddy / Envoy 都支持。
10. **配置版本化 + CI test**：变更前 `nginx -t` 验证。

---

## 第十二章：常见陷阱清单

### ❌ 陷阱 1：信任 client X-Forwarded-For
攻击者伪造 IP 绕过限流 / 日志。仅信任自己代理链。

### ❌ 陷阱 2：buffering on 但要 SSE
SSE 流被代理缓冲 → 实时性丢。`proxy_buffering off`。

### ❌ 陷阱 3：reload 期间丢请求
新旧 worker 共存优雅切换；强 stop 才丢。

### ❌ 陷阱 4：worker_connections 不够
高 QPS 时连接数受限。配合 `ulimit -n`。

### ❌ 陷阱 5：上游无 keep-alive
每请求新 TCP 连接 → 浪费。

### ❌ 陷阱 6：证书过期没监控
某天突然 prod 红色 → 必须监控告警。

### ❌ 陷阱 7：单实例代理
代理挂全挂。HA 部署 + cloud LB 前置。

---

## 第十三章：练习题

**练习 1**：写一个 Nginx 配置：HTTPS + HTTP/2 + 反代到 3 个后端实例 + 静态资源 + gzip。

**练习 2**：为什么应用前加 Nginx 比应用直接监听公网好？

**练习 3**：解释 `proxy_buffering on` 在什么场景反而不好。

**练习 4**：Caddy 自动 HTTPS 怎么工作？

**练习 5**：何时选 Envoy 而不是 Nginx？

---

## 参考答案

**练习 1**：参考第二章模板。要点：
- TLS（cert + key）
- `listen 443 ssl http2`
- `upstream` 含 3 个 server
- `proxy_set_header X-Forwarded-*`
- `gzip on + gzip_types`
- 静态：`location /static/ { root ...; expires 1y; }`
- HTTP 跳转 HTTPS

**练习 2**：
- 集中 TLS 管理
- 静态资源 sendfile 卸载
- 限流 / WAF 前置
- 多实例 LB
- gzip 卸载
- 平滑部署（rolling 时 LB 自动剔除）
- 应用更专注业务

**练习 3**：流式响应（SSE、long polling、stream API）需要立即推到 client；buffering 缓冲后才发 → 实时性丢。改 `proxy_buffering off` 或针对该 location 关。

**练习 4**：Caddy 启动后：
1. 检查域名 DNS 指向自己
2. 通过 HTTP-01 challenge（监听 80 端口给 CA token）或 DNS-01（如有 plugin）
3. 自动从 Let's Encrypt 申请证书
4. 存到 ~/.local/share/caddy/certificates
5. 定期续期（30 天前重新申请）

完全无需用户干预。

**练习 5**：
- 微服务 / Service Mesh 场景（Istio 底层是 Envoy）
- 需要动态配置（无需 reload）
- 复杂 traffic 管理：金丝雀、流量切割、断路器
- gRPC 友好（HTTP/2 first-class）
- 重度可观测性需求

简单 web server + 反代 → Nginx 更轻；Envoy 配置陡且重。

---

## 小结

| 产品 | 适合 |
|---|---|
| Nginx | 通用、传统部署、性能 |
| Caddy | 个人 / 小项目、零配置 HTTPS |
| HAProxy | L4 LB、极致性能 |
| Envoy | Service Mesh、动态配置、微服务 |
| Traefik | K8s ingress、容器自动发现 |

| 必做 | 项 |
|---|---|
| HTTPS | + HSTS |
| HTTP/2 | 默认 |
| keep-alive | 上游 |
| gzip/br | 响应 |
| 超时 | 分级 |
| log/metric | 必有 |

---

## 课程总结

这是 **B25 / 25**——Backend 路线图深度课程系列的最后一篇。

回顾整个系列覆盖：

| 章节 | 主题 |
|---|---|
| B01-B03 | 网络基础：互联网、HTTP、TLS |
| B04-B07 | API：REST、gRPC、GraphQL、OpenAPI |
| B08-B14 | 数据库：索引、事务、分片、复制、N+1、NoSQL |
| B15-B17 | 数据加速：缓存策略、Redis、消息队列 |
| B18-B21 | 架构与韧性：微服务、12-factor、韧性模式、限流 |
| B22-B24 | 安全与运维：认证、OWASP、可观测性 |
| B25 | Web 服务器与反向代理 |

加上 Go 路线图 30 篇，整套深度课程系列共 **55 篇**。

**下一步建议**：

1. **挑核心 5 篇精读**：B01 网络、B08 索引、B09 事务、B15 缓存、B22 认证 = 后端工程师的核心。
2. **结合实战**：拿一个真实项目，按这些主题逐个 audit。
3. **每月 review**：技术演进，每 6-12 月回看哪些章节需要更新。

祝你成为优秀的后端工程师 🎉

---

