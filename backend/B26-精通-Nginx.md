# 精通 Nginx：配置语义、代理链路与生产调优

> 课程编号：B26
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — Web Servers
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟

---

## 引言：为什么配置"看起来对"但不生效

[B25](./B25-精通-Web-服务器与反向代理.md) 横向对比了 Nginx / Caddy / HAProxy / Envoy / Traefik。这一篇只讲 Nginx，讲透。

先看三段几乎每个团队都写错过的配置：

```nginx
# ① 为什么请求打到上游变成了 /api/api/users？
location /api/ {
    proxy_pass http://backend;
}

# ② 为什么配了 keepalive 但 netstat 里全是 TIME_WAIT？
upstream backend {
    server 10.0.1.1:8080;
    keepalive 32;
}

# ③ 为什么 burst=20 之后延迟飙到 2 秒？
limit_req zone=api burst=20;
```

三段都能正常启动、`nginx -t` 全部通过、日志里没有任何报错。但行为都不是写的人以为的那样。

Nginx 的坑几乎全部集中在**配置指令的真实语义**上——不是语法错，是理解错。这一篇按"链路顺序"拆开：**进程模型 → 请求匹配 → 上游代理 → 超时 → 缓存 → 限流 → 观测 → reload**。

---

## 第一章：进程模型

### 1.1 master + worker

```
master 进程（root）
  ├── 读取/校验配置、绑定端口、管理信号
  ├── worker 进程 #1（nginx 用户）── epoll 事件循环
  ├── worker 进程 #2 ── epoll 事件循环
  └── cache manager / cache loader（配了 proxy_cache 才有）
```

关键点：

- **worker 之间不共享内存**（除非显式声明 `zone`）。这解释了后面很多"为什么限流/健康检查数据对不上"的问题。
- 每个 worker 单线程 + epoll，**永远不要在 worker 里做阻塞操作**（比如同步 DNS 解析、Lua 里的阻塞 IO）。
- worker 以非特权用户跑，master 以 root 跑（为了绑 80/443）。

### 1.2 worker 数量与连接数

```nginx
worker_processes auto;              # = CPU 核数
worker_rlimit_nofile 65535;         # 单 worker 的 fd 上限
events {
    worker_connections 10240;
    multi_accept on;                # 一次事件循环尽量多 accept
}
```

**`worker_connections` 不是最大并发用户数。** 做反向代理时，一个客户端请求要占**两个**连接（客户端侧 + 上游侧）：

```
最大并发客户端 ≈ worker_processes × worker_connections / 2
```

而且 `worker_rlimit_nofile` 必须 ≥ `worker_connections`，否则先撞到 fd 上限，日志里出现 `worker_connections are not enough` 或 `too many open files`。

> ⚠️ 容器里还要检查 `ulimit -n` 和 systemd 的 `LimitNOFILE`——nginx 自己的配置管不到操作系统的硬限制。

### 1.3 惊群与 `SO_REUSEPORT`

多个 worker 监听同一端口，连接到来时全被唤醒（惊群）。老方案是 `accept_mutex on`（轮流 accept）。现代方案：

```nginx
listen 443 ssl reuseport;    # 内核按连接哈希分发给不同 worker
```

`reuseport` 让内核直接把连接分给某个 worker，避免惊群和锁竞争，高并发短连接场景吞吐提升明显。**同一个端口只需在一个 `listen` 上写 `reuseport`。**

---

## 第二章：请求匹配 —— 最大的坑区

### 2.1 `location` 匹配优先级

这是 Nginx 最反直觉的部分。**不是从上到下第一个匹配的胜出。**

| 修饰符 | 类型 | 行为 |
|---|---|---|
| `=` | 精确匹配 | 命中立即使用，**停止一切后续匹配** |
| `^~` | 前缀匹配 | 命中且是最长前缀时使用，**跳过正则检查** |
| `~` | 正则（区分大小写） | 按**配置文件出现顺序**，第一个命中的胜出 |
| `~*` | 正则（不区分大小写） | 同上 |
| （无） | 普通前缀 | 记录最长匹配，但**继续检查正则** |

完整算法：

```
1. 找 = 精确匹配 → 命中则结束
2. 找所有普通前缀匹配，记住最长的那个
3. 若最长前缀带 ^~ → 用它，结束（不查正则）
4. 按配置顺序查正则 → 第一个命中的胜出
5. 正则都没命中 → 用第 2 步记住的最长前缀
```

看一个真实案例：

```nginx
location /images/ { ... }              # A：普通前缀
location ~ \.(gif|jpg)$ { ... }        # B：正则
```

请求 `/images/logo.jpg` 命中的是 **B**，不是 A——因为普通前缀匹配后还会继续查正则，正则赢。想让 A 赢，改成 `location ^~ /images/`。

> 💡 排查技巧：`error_log` 开到 `debug` 级别，日志里会打印 `using configuration "..."`，直接告诉你命中了哪个 location。

### 2.2 `proxy_pass` 的尾斜杠 —— 头号陷阱

```nginx
location /api/ {
    proxy_pass http://backend;      # 无 URI 部分
}
# 请求 /api/users → 上游收到 /api/users   （完整透传）

location /api/ {
    proxy_pass http://backend/;     # 有 URI 部分（就是那个 /）
}
# 请求 /api/users → 上游收到 /users      （替换掉 location 匹配的部分）

location /api/ {
    proxy_pass http://backend/v2/;
}
# 请求 /api/users → 上游收到 /v2/users
```

**规则一句话：`proxy_pass` 后面带了 URI（哪怕只是一个 `/`），就用它替换掉 location 匹配的前缀；不带，就原样透传完整路径。**

两个附加限制：

- location 用**正则**时，`proxy_pass` **不能**带 URI（配置校验会报错）
- location 里用了 `rewrite ... break` 时，`proxy_pass` 也不能带 URI（会使用 rewrite 后的 URI）

引言里的问题 ①：`location /api/` + `proxy_pass http://backend;`，如果后端应用本身又挂在 `/api` 前缀下，路径就变成了 `/api/api/users`——不是 nginx 加的，是"完整透传"遇上"后端也带前缀"。

### 2.3 `root` vs `alias`

```nginx
location /static/ {
    root /var/www;          # 拼接：/var/www + /static/a.js = /var/www/static/a.js
}

location /static/ {
    alias /var/www/;        # 替换：/var/www/ + a.js = /var/www/a.js
}
```

- `root` = **拼接**完整 URI
- `alias` = **替换**掉 location 匹配的部分

`alias` 的两个规矩：location 以 `/` 结尾时 alias 也必须以 `/` 结尾（否则路径拼接会错位）；正则 location 里用 alias 必须带捕获组。

**能用 `root` 就别用 `alias`**——alias 的路径拼接错位是经典的目录穿越漏洞来源。

### 2.4 `server_name` 匹配与 default_server

优先级：

```
1. 精确名                  example.com
2. 最长的 * 开头通配符      *.example.com
3. 最长的 * 结尾通配符      mail.*
4. 正则（按出现顺序）       ~^www\d+\.example\.com$
5. default_server（没标记则用该端口第一个 server 块）
```

**生产必配：一个吃掉所有未知 Host 的默认块。**

```nginx
server {
    listen 80 default_server;
    listen 443 ssl default_server;
    server_name _;                      # 匹配不上任何真实域名的占位符

    ssl_certificate     /etc/nginx/ssl/dummy.crt;   # 自签即可
    ssl_certificate_key /etc/nginx/ssl/dummy.key;

    return 444;                          # nginx 特有：直接关连接，不回响应
}
```

不配的话，扫描器直接用 IP 访问会命中你的第一个 server 块，暴露内部服务。

### 2.5 `if` 是恶魔

官方 wiki 有一篇专门的 [IfIsEvil](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#if)。在 `location` 上下文里，**只有两条指令在 `if` 里是安全的**：

```nginx
if ($request_method = POST) { return 405; }        # ✅ return 安全
if ($slow) { rewrite ^ /slow-path last; }          # ✅ rewrite ... last 安全
```

其他指令（`proxy_pass`、`add_header`、`set` 等）放在 `if` 里可能产生未定义行为甚至段错误——因为 `if` 内部创建了一个隐式的嵌套 location，指令继承规则会以极其反直觉的方式失效。

替代方案用 `map`（在 `http` 块里，配置解析期求值，零运行时成本）：

```nginx
map $http_user_agent $is_bot {
    default        0;
    "~*bot|crawler|spider"  1;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Is-Bot $is_bot;
    }
}
```

---

## 第三章：upstream 与负载均衡

### 3.1 到上游的 keepalive —— 两个必配项

引言里的问题 ②：

```nginx
upstream backend {
    server 10.0.1.1:8080;
    keepalive 32;              # 每个 worker 保持的空闲连接数
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;         # ← 必须！默认是 1.0，不支持 keepalive
    proxy_set_header Connection ""; # ← 必须！清掉默认的 Connection: close
}
```

**只写 `keepalive 32;` 而不写后两行，连接池完全不生效。** 这是 Nginx 最高频的配置遗漏，症状是上游机器 TIME_WAIT 堆积、P99 延迟里 `$upstream_connect_time` 明显偏高。

`keepalive` 的数值是**每个 worker** 的空闲连接数，实际总数要乘以 `worker_processes`。设太大会占满上游的连接数上限，一般 `keepalive` ≈ 上游单机连接上限 / worker 数 / 上游实例数，从 32 起调。

### 3.2 负载均衡算法

```nginx
upstream backend {
    # round_robin           默认，加权轮询
    least_conn;           # 连接数最少（长连接场景更均匀）
    # ip_hash;              基于客户端 IP 哈希（粘性，但 NAT 后会倾斜）
    # hash $request_uri consistent;   一致性哈希（扩缩容时迁移量最小）
    # random two least_conn;          随机选两个取连接数少的（"两次随机选择"算法）

    server 10.0.1.1:8080 weight=3;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080 backup;      # 仅当所有主节点都挂了才用

    keepalive 32;
}
```

选型：

- **短连接 HTTP** → 默认 round_robin 够用
- **长连接 / 请求耗时差异大** → `least_conn`
- **需要缓存亲和**（比如按 URI 分片的缓存层）→ `hash $request_uri consistent`
- **大规模集群** → `random two least_conn`，接近全局最优但无锁竞争

### 3.3 健康检查与失败判定（OSS 版的真相）

**Nginx OSS 只有被动健康检查。** 没有主动探活——那是 Nginx Plus 或 `nginx_upstream_check_module` 第三方模块的功能。

```nginx
server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
```

`fail_timeout` 有**双重含义**，这是最常被误解的一点：

1. **统计窗口**：在 30 秒内失败 3 次（`max_fails`）→ 判定该节点不可用
2. **隔离时长**：判定不可用后，接下来 30 秒不再往它发请求

30 秒后 nginx 会**放一个请求过去试探**，成功则恢复，失败则再隔离 30 秒。

> ⚠️ 由于 worker 间不共享状态，**每个 worker 独立计数**。8 个 worker 意味着一个真正挂掉的节点最多可能被试探 8 次。想让状态共享，加 `zone`：
> ```nginx
> upstream backend {
>     zone backend 64k;      # 共享内存区，worker 间共享节点状态
>     server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
> }
> ```

### 3.4 重试语义 —— 别让 POST 重复执行

```nginx
proxy_next_upstream error timeout http_502 http_503;
proxy_next_upstream_tries 2;          # 最多试 2 个上游
proxy_next_upstream_timeout 10s;      # 重试总耗时上限
```

默认值是 `error timeout`。关键的安全默认：**非幂等请求（POST/LOCK/PATCH）默认不重试**。

```nginx
proxy_next_upstream error timeout non_idempotent;   # ⚠️ 危险
```

加了 `non_idempotent`，POST 也会重试——如果上游是"已经处理完但响应超时"，用户就会被**重复下单/重复扣款**。除非你的接口有幂等键保护，否则**永远不要加这个参数**。

同理，`http_500` 也要慎重加入重试列表：500 通常意味着业务逻辑已经执行了一部分。

---

## 第四章：超时全景

超时配错的典型症状是"偶发 502/504，但后端日志里请求是成功的"。把每个超时对应到链路阶段：

```
客户端 ──①──> nginx ──②──> upstream
        <──④──      <──③──
```

| 阶段 | 指令 | 默认值 | 说明 |
|---|---|---|---|
| ① 读客户端请求头 | `client_header_timeout` | 60s | 防 slowloris |
| ① 读客户端请求体 | `client_body_timeout` | 60s | **两次读操作之间**的间隔，不是总时长 |
| ② 连上游 | `proxy_connect_timeout` | 60s | **不能超过 75s**；正常内网应 < 2s |
| ② 发给上游 | `proxy_send_timeout` | 60s | 两次写之间的间隔 |
| ③ 等上游响应 | `proxy_read_timeout` | **60s** | ⚠️ 长连接/慢接口的头号杀手 |
| ④ 发给客户端 | `send_timeout` | 60s | 两次写之间的间隔 |
| 空闲保持 | `keepalive_timeout` | 75s | 客户端侧 keep-alive |

**必须理解的一点**：这些超时（除 `proxy_connect_timeout`）计的都是**两次 IO 操作之间的间隔**，不是请求总时长。`proxy_read_timeout 60s` 的意思是"上游连续 60 秒没吐出任何字节就断"，而不是"响应必须在 60 秒内完成"。

所以一个持续输出的 SSE 流可以跑几小时不触发超时，但一个"思考 90 秒后一次性返回"的接口会在第 60 秒被切断。

生产参考值：

```nginx
client_header_timeout 10s;
client_body_timeout   30s;
proxy_connect_timeout 2s;        # 内网建连超过 2s 一定有问题
proxy_send_timeout    30s;
proxy_read_timeout    60s;       # 长连接场景单独在 location 里调大
send_timeout          30s;
keepalive_timeout     65s;       # 略大于客户端/LB 的空闲超时
```

---

## 第五章：缓存

### 5.1 基础配置

```nginx
# http 块
proxy_cache_path /var/cache/nginx
                 levels=1:2                  # 两级目录，避免单目录文件过多
                 keys_zone=api_cache:100m    # 共享内存：约 80 万个 key
                 max_size=10g                # 磁盘上限
                 inactive=60m                # 60 分钟没被访问就删（与 TTL 无关）
                 use_temp_path=off;          # 直接写目标目录，少一次跨设备移动

location /api/ {
    proxy_cache api_cache;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_methods GET HEAD;

    add_header X-Cache-Status $upstream_cache_status;   # HIT/MISS/EXPIRED/STALE/UPDATING/BYPASS
    proxy_pass http://backend;
}
```

`inactive` 和 `proxy_cache_valid` 是两个独立维度，经常被混淆：

- `proxy_cache_valid 10m` —— 内容 10 分钟后**过期**（需要回源验证）
- `inactive=60m` —— 60 分钟**无人访问**就从磁盘删除，哪怕还没过期

### 5.2 防缓存击穿

热点 key 过期的瞬间，成千上万个请求同时回源打爆后端。三道防线：

```nginx
proxy_cache_lock on;                    # 同一 key 只放一个请求回源，其余等待
proxy_cache_lock_timeout 5s;            # 等超过 5s 就放行回源（避免全员卡死）
proxy_cache_lock_age 5s;                # 持锁请求超过 5s 未完成，再放一个

proxy_cache_use_stale updating error timeout http_500 http_502 http_503 http_504;
proxy_cache_background_update on;       # 立即返回旧内容，后台异步更新
```

`proxy_cache_use_stale` + `background_update` 的组合效果：缓存过期时**立即返回旧内容**（用户零等待），同时后台发一个请求回源刷新。这就是 HTTP 语义里的 `stale-while-revalidate`，也是让后端流量曲线变平滑的最有效手段。

`error timeout http_5xx` 这几个值还有额外收益：**后端整体挂掉时，nginx 会继续用过期缓存兜底**，把故障对用户的影响从"全站 502"降级为"数据有点旧"。

### 5.3 什么不该缓存

```nginx
proxy_no_cache     $http_authorization $cookie_session;   # 非空则不写缓存
proxy_cache_bypass $http_authorization $cookie_session;   # 非空则不读缓存
```

带认证信息的响应缓存到共享层，是**跨用户数据泄漏**的经典事故——A 用户的个人页被缓存，B 用户访问同一 URL 直接命中。凡是响应内容随用户变化的路径，要么不缓存，要么把用户标识放进 `proxy_cache_key`。

---

## 第六章：限流

### 6.1 `limit_req` —— burst 与 nodelay

```nginx
# http 块
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
    limit_req_status 429;               # 默认 503，429 语义更准确
    limit_req_log_level warn;
}
```

`$binary_remote_addr` 而不是 `$remote_addr`：前者是 4/16 字节的二进制，后者是字符串。10MB 的 zone 能存约 16 万个 IPv4 地址。

**算法是漏桶。`rate=10r/s` 内部换算成"每 100ms 放行一个"**，理解这点才能理解 burst：

| 配置 | 行为 | 第 2 个请求在 10ms 后到达时 |
|---|---|---|
| `limit_req zone=api;` | 严格匀速 | **直接 503** |
| `limit_req zone=api burst=20;` | 排队等待 | 延迟 90ms 后处理 |
| `limit_req zone=api burst=20 nodelay;` | 立即处理 | **立即处理**，占用一个槽位 |
| `limit_req zone=api burst=20 delay=5;` | 前 5 个不延迟 | 立即处理 |

引言里的问题 ③：`burst=20` **不带 nodelay** 时，突发的第 20 个请求要排队等 `19 × 100ms = 1.9 秒`。用户体验是"网站变得极慢"而不是"被限流了"——比直接返回 429 更糟。

**绝大多数 API 场景应该用 `burst=N nodelay`**：允许突发立即通过，槽位按 rate 速率恢复。只有在保护脆弱下游（比如老旧数据库）时才用纯排队模式。

### 6.2 `limit_conn` 与限流的正确 key

```nginx
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_conn_zone $server_name        zone=perserver:10m;

location /download/ {
    limit_conn perip 5;             # 单 IP 并发 5 个下载
    limit_conn perserver 500;       # 全站并发 500
    limit_rate 1m;                  # 每连接限速 1MB/s
    limit_rate_after 10m;           # 前 10MB 不限速（小文件不受影响）
}
```

> ⚠️ **限流的 key 依赖真实客户端 IP。** 如果 nginx 前面还有 LB/CDN 而没有正确配置 `real_ip`（下一章），`$binary_remote_addr` 会是 LB 的 IP——所有用户共享一个限流桶，限流形同虚设或者全员被误伤。

同样地，worker 间不共享状态的问题在这里**不存在**：`limit_req_zone` / `limit_conn_zone` 本身就是共享内存区。

---

## 第七章：长连接代理（WebSocket / SSE / gRPC）

这一章直接对应 [G31 — 精通 Go Socket 与 WebSocket](../golang/G31-精通-Go-Socket-与-WebSocket.md) 里提到的代理侧配置。

### 7.1 WebSocket

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;          # 非 WebSocket 请求走正常 keep-alive
}

location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;                        # 必须：1.0 不支持 Upgrade
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host       $host;

    proxy_read_timeout  3600s;                     # 必须：默认 60s 会切断空闲连接
    proxy_send_timeout  3600s;
}
```

用 `map` 而不是硬编码 `Connection "upgrade"`：硬编码会让同一个 location 下的普通 HTTP 请求也带上 Upgrade 头，某些后端框架会报错。

**`proxy_read_timeout` 默认 60 秒是 WebSocket 断连的头号原因**，症状是"连接每分钟准时断一次"。两个解法要同时做：调大这个值，**并且**让应用层心跳间隔小于它（30s 是安全值）。只做前者，中间还有云厂商 NAT 网关的空闲超时在等着你。

### 7.2 SSE / 流式响应

```nginx
location /stream {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection '';

    proxy_buffering off;          # 关键：不缓冲，逐块下发
    proxy_cache off;
    gzip off;                     # gzip 会攒缓冲区，破坏流式
    chunked_transfer_encoding on;
    proxy_read_timeout 3600s;
}
```

`proxy_buffering on`（默认值）会让 nginx 攒够 buffer 才发给客户端——LLM 的流式输出会变成"转半天圈然后一次性全出来"。

也可以让后端用响应头动态控制，不改 nginx 配置：

```
X-Accel-Buffering: no
```

对只有部分接口需要流式的服务，这个办法更省事。

### 7.3 gRPC

```nginx
server {
    listen 443 ssl http2;         # gRPC 必须走 HTTP/2

    location / {
        grpc_pass grpc://backend;             # 注意是 grpc_pass 不是 proxy_pass
        grpc_read_timeout  3600s;
        grpc_send_timeout  3600s;
    }
}
```

TLS 上游用 `grpcs://`。gRPC 的超时指令是独立的一套（`grpc_*`），配 `proxy_*` 不生效。

---

## 第八章：TLS

```nginx
ssl_protocols TLSv1.2 TLSv1.3;                    # TLS 1.0/1.1 早已不安全
ssl_prefer_server_ciphers off;                    # TLS 1.3 下让客户端选更好
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;

ssl_session_cache shared:SSL:50m;                 # 共享 session 缓存（worker 间共享）
ssl_session_timeout 1d;
ssl_session_tickets off;                          # 除非能定期轮换 ticket key，否则关掉

ssl_stapling on;                                  # OCSP stapling，省一次客户端往返
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;              # stapling 需要 DNS 解析
resolver_timeout 5s;

add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
```

三个容易漏的点：

1. **`ssl_session_cache` 必须用 `shared:`**——写成 `builtin` 是每 worker 独立的，命中率极低。
2. **`ssl_session_tickets off`**：ticket key 不轮换的话，泄漏一次等于历史流量全部可解密（破坏前向保密）。除非有自动轮换机制，否则关掉，只用 session cache。
3. **`add_header` 的 `always` 参数**：不加的话，错误响应（4xx/5xx）不会带上这个头。HSTS 这类安全头必须加 `always`。

> ⚠️ `add_header` 有个继承陷阱：**子级 location 里只要出现任何一条 `add_header`，父级的全部失效**。安全头要么全放在 `server` 块且子级不用 `add_header`，要么在每个用到的 location 里重复一遍。

HTTP/3 见 [B25 §2.6](./B25-精通-Web-服务器与反向代理.md)。

---

## 第九章：真实客户端 IP

nginx 前面有 LB/CDN 时，`$remote_addr` 是 LB 的地址。修正：

```nginx
set_real_ip_from 10.0.0.0/8;          # 只信任这些网段发来的 XFF
set_real_ip_from 172.16.0.0/12;
set_real_ip_from 2400:cb00::/32;      # 比如 Cloudflare 的段
real_ip_header   X-Forwarded-For;
real_ip_recursive on;
```

**`set_real_ip_from` 是安全边界，不是可选优化。** 不配它就等于无条件信任 `X-Forwarded-For`，而这个头客户端可以任意伪造：

```bash
curl -H "X-Forwarded-For: 1.2.3.4" https://your-site/   # 伪造成功
```

后果是限流被绕过、审计日志被污染、基于 IP 的访问控制失效。

`real_ip_recursive on` 的作用：从 XFF 链**从右往左**跳过所有可信地址，取第一个不可信的作为真实 IP。设为 `off` 则直接取最右边一个——在多层代理下会拿到中间层的地址。

### 转发给上游的头

```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
```

`$host` vs `$http_host` 的区别：

| 变量 | 值 |
|---|---|
| `$http_host` | 原始 Host 头，**保留端口和大小写**，缺失时为空 |
| `$host` | 小写、去掉端口；Host 头缺失时回退到匹配的 `server_name` |

**默认用 `$host`。** 需要保留端口的场景（比如后端要生成带端口的绝对 URL）才用 `$http_host`。

---

## 第十章：日志与可观测性

### 10.1 log_format：把排障需要的字段一次配齐

```nginx
log_format main escape=json
  '{"time":"$time_iso8601",'
  '"remote_addr":"$remote_addr",'
  '"request":"$request",'
  '"status":$status,'
  '"body_bytes":$body_bytes_sent,'
  '"request_time":$request_time,'
  '"upstream_addr":"$upstream_addr",'
  '"upstream_status":"$upstream_status",'
  '"upstream_connect_time":"$upstream_connect_time",'
  '"upstream_header_time":"$upstream_header_time",'
  '"upstream_response_time":"$upstream_response_time",'
  '"cache_status":"$upstream_cache_status",'
  '"request_id":"$request_id",'
  '"referer":"$http_referer",'
  '"user_agent":"$http_user_agent"}';

access_log /var/log/nginx/access.log main buffer=64k flush=5s;
```

- **`escape=json`** 是必须的：User-Agent 里的引号和反斜杠会破坏 JSON 结构，日志采集器直接丢弃整行。
- **`$request_id`** 是 nginx 自动生成的 32 位十六进制唯一 ID。用 `proxy_set_header X-Request-ID $request_id;` 传给后端，全链路日志就能串起来。
- **`buffer=64k flush=5s`** 减少磁盘 IO，高 QPS 下效果明显。

### 10.2 三个时间变量的区别 —— 排障的关键

| 变量 | 含义 |
|---|---|
| `$upstream_connect_time` | 与上游建立连接的耗时（https 上游含 TLS 握手） |
| `$upstream_header_time` | 从建连到**收到上游响应头** |
| `$upstream_response_time` | 从建连到**收完上游整个响应体** |
| `$request_time` | 从读到客户端第一个字节，到**最后一个字节发给客户端** |

诊断表：

| 现象 | 结论 |
|---|---|
| `$upstream_response_time` 大 | 后端真的慢 → 去查应用 |
| `$request_time` >> `$upstream_response_time` | **客户端接收慢**（弱网、大响应体），不是后端问题 |
| `$upstream_connect_time` 大 | 上游连接池不足 / keepalive 没生效 / 网络问题 |
| `$upstream_header_time` 小但 `$response_time` 大 | 后端流式输出（正常）或响应体巨大 |
| `$upstream_addr` 有多个值（逗号分隔） | 发生了重试 → 检查上游健康状况 |

**"后端明明很快，用户却说慢"** 的绝大多数案例，答案就在第二行。

### 10.3 stub_status

```nginx
location /nginx_status {
    stub_status;
    allow 10.0.0.0/8;
    deny all;
}
```

输出的 `Active connections`、`accepts/handled/requests`、`Reading/Writing/Waiting` 可以直接喂给 Prometheus 的 nginx-exporter。

**`accepts` 与 `handled` 不相等**是重要信号：说明有连接因为达到 `worker_connections` 上限被丢弃。

---

## 第十一章：reload 与零停机

### 11.1 reload 时发生了什么

```bash
nginx -t              # 先校验！
nginx -s reload       # 等价于 kill -HUP <master_pid>
```

流程：

```
master 收到 SIGHUP
  → 解析新配置（失败则保留旧配置继续跑，服务不受影响）
  → 用新配置启动新 worker
  → 向老 worker 发 SIGQUIT（优雅退出）
      老 worker：停止 accept 新连接 → 处理完手上的请求 → 退出
```

**reload 本身是安全的**：配置有错时 nginx 会拒绝加载并继续用旧配置。但 `nginx -t` 仍要先跑——它能在不影响服务的前提下暴露问题。

### 11.2 长连接会让老 worker 永远不退出

老 worker 要等**所有现有连接**结束才退出。对 WebSocket/SSE 这种小时级的连接，意味着：

```bash
$ ps aux | grep nginx
nginx: worker process is shutting down    # 已经存在 3 天了
nginx: worker process is shutting down
nginx: worker process is shutting down    # 每次 reload 攒一批
nginx: worker process
```

每次 reload 攒一批僵死 worker，内存持续上涨。解药：

```nginx
worker_shutdown_timeout 30s;      # 默认无限制！
```

超时后强制关闭剩余连接。对 WebSocket 服务，30 秒配合客户端自动重连是合理的取舍——参考 [G31 §6.4](../golang/G31-精通-Go-Socket-与-WebSocket.md) 的优雅重启部分。

### 11.3 其他信号

| 信号 | 作用 |
|---|---|
| `SIGHUP` (`-s reload`) | 重载配置 |
| `SIGQUIT` (`-s quit`) | 优雅退出 |
| `SIGTERM` (`-s stop`) | 立即退出（**会断掉正在处理的请求**） |
| `SIGUSR1` (`-s reopen`) | 重开日志文件（**logrotate 必须用这个**） |
| `SIGUSR2` | 热升级二进制（不断连接换 nginx 版本） |

> ⚠️ logrotate 配置里如果写的是 `kill -HUP`（reload）而不是 `-s reopen`，每天凌晨会白白重载一次全量配置。而如果什么信号都不发，nginx 会继续往已被重命名的旧文件句柄里写——磁盘满了都找不到日志在哪。

---

## 第十二章：性能调优

### 12.1 静态文件

```nginx
sendfile on;              # 零拷贝：内核直接把文件送到 socket，不经用户态
tcp_nopush on;            # 配合 sendfile：攒满一个包再发（Linux TCP_CORK）
tcp_nodelay on;           # 对 keepalive 连接禁用 Nagle
```

三者是标配组合。注意 **`sendfile` 只对本地文件有效**，代理响应用不上；`tcp_nopush` 只在 `sendfile on` 时生效。

```nginx
open_file_cache max=10000 inactive=60s;    # 缓存 fd + 大小 + mtime
open_file_cache_valid 60s;
open_file_cache_min_uses 2;
open_file_cache_errors on;                 # 连"文件不存在"也缓存
```

代价：文件更新后最长 60 秒内可能还返回旧内容。**部署时用"新文件名 + 新路径"而不是原地覆盖**，就没有这个问题。

### 12.2 缓冲区

```nginx
client_max_body_size 10m;         # 默认 1m，上传大文件必调（否则 413）
client_body_buffer_size 128k;     # 超出部分落磁盘临时文件

proxy_buffers 8 16k;              # 每连接的缓冲区个数 × 大小
proxy_buffer_size 8k;             # 单独用于响应头
proxy_busy_buffers_size 32k;
```

`proxy_buffer_size` 太小会导致 `upstream sent too big header` 502——常见于响应头里塞了大 JWT 或超多 Set-Cookie。响应头超过 8k 时调到 16k/32k。

### 12.3 压缩

```nginx
gzip on;
gzip_vary on;                     # 加 Vary: Accept-Encoding，避免 CDN 缓存串味
gzip_comp_level 5;                # 6 以上收益递减、CPU 陡增
gzip_min_length 1024;             # 小响应压缩后可能更大
gzip_proxied any;
gzip_types text/plain text/css application/json application/javascript
           application/xml image/svg+xml;
```

**不要压缩已压缩的内容**（jpg/png/mp4/zip）——浪费 CPU 且体积可能变大。默认的 `gzip_types` 只有 `text/html`，其余必须显式列出。

**`gzip_vary on` 不加会出事**：CDN 可能把压缩版本返回给不支持 gzip 的客户端。

---

## 第十三章：生产配置模板

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
worker_shutdown_timeout 30s;          # ← 别忘了

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 10240;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    server_tokens off;                 # 隐藏版本号

    # ---- 日志 ----
    log_format main escape=json '{"time":"$time_iso8601","status":$status,'
        '"request":"$request","request_time":$request_time,'
        '"upstream_time":"$upstream_response_time","addr":"$remote_addr",'
        '"request_id":"$request_id"}';
    access_log /var/log/nginx/access.log main buffer=64k flush=5s;

    # ---- 基础性能 ----
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65s;
    client_max_body_size 10m;

    # ---- 超时 ----
    client_header_timeout 10s;
    client_body_timeout   30s;
    proxy_connect_timeout 2s;
    proxy_send_timeout    30s;
    proxy_read_timeout    60s;

    # ---- 压缩 ----
    gzip on;
    gzip_vary on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript
               application/xml image/svg+xml;

    # ---- 真实 IP ----
    set_real_ip_from 10.0.0.0/8;
    real_ip_header X-Forwarded-For;
    real_ip_recursive on;

    # ---- 限流 ----
    limit_req_zone  $binary_remote_addr zone=api:10m rate=20r/s;
    limit_conn_zone $binary_remote_addr zone=perip:10m;

    # ---- WebSocket 用的 map ----
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    upstream backend {
        zone backend 64k;              # worker 间共享节点状态
        least_conn;
        server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
        server 10.0.1.2:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    # ---- 吃掉未知 Host ----
    server {
        listen 80 default_server;
        server_name _;
        return 444;
    }

    server {
        listen 80;
        server_name api.example.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl reuseport;
        http2 on;                       # nginx 1.25.1+ 的新写法
        server_name api.example.com;

        ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_session_cache shared:SSL:50m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;
        ssl_stapling on;
        ssl_stapling_verify on;

        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
        add_header X-Content-Type-Options nosniff always;
        add_header X-Frame-Options DENY always;

        location / {
            limit_req  zone=api burst=40 nodelay;
            limit_conn perip 20;
            limit_req_status 429;

            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Request-ID      $request_id;
        }

        location /ws {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host       $host;
            proxy_read_timeout 3600s;
        }

        location /stream {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection '';
            proxy_buffering off;
            gzip off;
            proxy_read_timeout 3600s;
        }

        location /nginx_status {
            stub_status;
            allow 10.0.0.0/8;
            deny all;
        }
    }
}
```

---

## 第十四章：常见陷阱清单

### ❌ 陷阱 1：`proxy_pass` 尾斜杠
带 URI 就替换 location 前缀，不带就完整透传。路径变成 `/api/api/x` 或 404 的根源。

### ❌ 陷阱 2：以为 location 从上往下第一个匹配
精确 `=` > `^~` 前缀 > 正则（按序）> 最长普通前缀。写了 `location /images/` 却被后面的正则抢走。

### ❌ 陷阱 3：配了 `keepalive` 但漏了两行
必须同时有 `proxy_http_version 1.1;` 和 `proxy_set_header Connection "";`，否则连接池完全不生效。

### ❌ 陷阱 4：`proxy_read_timeout` 默认 60s
WebSocket/SSE/慢接口被准时切断。注意它计的是**两次读之间的间隔**，不是总时长。

### ❌ 陷阱 5：`limit_req burst` 不加 `nodelay`
突发请求变成排队，延迟飙到秒级。用户感受到的是"卡"而不是"被限流"。

### ❌ 陷阱 6：不配 `set_real_ip_from` 就用 XFF
等于无条件信任客户端伪造的 IP，限流和访问控制全部失效。

### ❌ 陷阱 7：子 location 的 `add_header` 覆盖父级全部
子级出现任何一条 `add_header`，父级的安全头**全部**失效。加 `always` 也救不了这个——那是另一个问题（错误响应不带头）。

### ❌ 陷阱 8：`worker_shutdown_timeout` 不设
长连接场景下每次 reload 攒一批 `is shutting down` 的僵死 worker，内存持续上涨。

### ❌ 陷阱 9：SSE 忘了关 `proxy_buffering`
流式输出被缓冲成一次性返回。LLM 应用里表现为"转圈半天然后全部吐出来"。

### ❌ 陷阱 10：在 `if` 里写 `proxy_pass` / `add_header`
未定义行为。只有 `return` 和 `rewrite ... last` 在 `if` 里安全，其余用 `map` 代替。

### ❌ 陷阱 11：logrotate 用 `-s reload` 而不是 `-s reopen`
每天白白重载一次全量配置。什么都不发则更糟——日志写进已删除的 fd，磁盘满了都找不到文件。

### ❌ 陷阱 12：给 POST 加 `proxy_next_upstream ... non_idempotent`
上游"处理完但响应超时"时会重复执行，用户重复下单。

### ❌ 陷阱 13：缓存带认证的响应
`proxy_no_cache $http_authorization` 不配，A 用户的数据会被返回给 B 用户。

### ❌ 陷阱 14：`gzip` 没配 `gzip_vary`
CDN 把压缩版本返回给不支持 gzip 的客户端，页面变乱码。

---

## 第十五章：练习题

**练习 1**：以下配置，请求 `/api/v1/users` 时上游收到的路径是什么？

```nginx
location /api/ {
    proxy_pass http://backend/service/;
}
```

**练习 2**：以下两个 location，请求 `/static/app.js` 命中哪个？想让第一个命中该怎么改？

```nginx
location /static/          { root /var/www; }
location ~ \.js$           { proxy_pass http://jsbackend; }
```

**练习 3**：监控发现某接口 `$request_time` 平均 3.2 秒，但 `$upstream_response_time` 只有 0.08 秒。可能的原因是什么？怎么验证？

**练习 4**：`rate=10r/s burst=20`（不带 nodelay），瞬间来了 21 个请求。第 21 个请求的结果是什么？如果加上 `nodelay` 呢？

**练习 5**：一个 WebSocket 服务用 nginx 代理，运维反馈"每次发版后 nginx 内存涨一截，重启才能降下来"。请给出原因和修复配置。

---

## 参考答案

**练习 1**：`/service/v1/users`。

`proxy_pass` 带了 URI（`/service/`），所以 location 匹配的前缀 `/api/` 被替换成 `/service/`，剩余部分 `v1/users` 原样拼接。

**练习 2**：命中**第二个**（正则）。

普通前缀 `location /static/` 匹配后 nginx 会继续检查正则，正则命中则正则胜出。改法：

```nginx
location ^~ /static/ { root /var/www; }    # ^~ 命中后跳过正则检查
```

**练习 3**：**客户端接收慢**，不是后端慢。

`$request_time` 包含把响应体发给客户端的全过程；`$upstream_response_time` 只算 nginx ↔ 上游。差值 3.1 秒说明时间花在客户端下行上。

验证方向：

- 看 `$body_bytes_sent`——响应体是不是过大（比如没分页的列表接口）
- 看 User-Agent / IP 分布——是不是集中在移动端或某个地区
- 服务端内网 `curl` 同一接口对比 `$request_time`——如果内网正常就坐实了

顺带一提，如果配了 `limit_rate`，也会直接体现在这里。

**练习 4**：

- **不带 nodelay**：第 21 个请求**被拒绝**（503/429）。前 1 个立即处理，第 2–21 个（共 20 个）填满 burst 队列排队，队列已满，第 21 个没位置。队列里最后一个要等约 `19 × 100ms = 1.9` 秒。
- **带 nodelay**：前 21 个中有 1 + 20 = 21 个立即处理（1 个正常 + 20 个占 burst 槽位），**第 21 个能通过**但槽位已满；紧接着的第 22 个会被拒绝。槽位按 10/s 的速率释放。

关键区别：nodelay 决定的是"排队的请求要不要等"，不是"能不能进队列"。

**练习 5**：

**原因**：reload 时老 worker 要等所有连接结束才退出，而 WebSocket 连接是小时级的。`worker_shutdown_timeout` 默认无限制，于是每次发版都留下一批 `nginx: worker process is shutting down`，内存不断累积。

用 `ps aux | grep "shutting down" | wc -l` 可以直接确认。

**修复**：

```nginx
worker_shutdown_timeout 30s;      # 30 秒后强制关闭残余连接
```

配套动作（缺一不可）：

1. 客户端实现**自动重连**（指数退避 + jitter，避免重连惊群）
2. 应用层心跳 ≤ 30s，让客户端能快速感知断开
3. 有条件的话，发版走**蓝绿/滚动**而不是原地 reload，让连接自然漂移到新实例

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 进程模型 | master + N worker；**worker 间不共享状态**（除非 `zone`） |
| worker_connections | 反代场景一个请求占 2 个连接；÷2 才是并发用户数 |
| location 优先级 | `=` > `^~` > 正则（按序）> 最长普通前缀 |
| proxy_pass 斜杠 | **带 URI 就替换 location 前缀，不带就完整透传** |
| root vs alias | root 拼接完整 URI，alias 替换匹配部分 |
| if | 只有 `return` 和 `rewrite ... last` 安全，其余用 `map` |
| upstream keepalive | 必须同时配 `proxy_http_version 1.1` + `Connection ""` |
| max_fails/fail_timeout | fail_timeout 双重含义：统计窗口 + 隔离时长 |
| proxy_next_upstream | 默认不重试非幂等请求；**别加 `non_idempotent`** |
| 超时 | 计的是**两次 IO 之间的间隔**，不是总时长 |
| proxy_read_timeout | 默认 60s，长连接必调大 + 配心跳 |
| 缓存击穿 | `proxy_cache_lock` + `use_stale updating` + `background_update` |
| 缓存安全 | `proxy_no_cache $http_authorization` 防跨用户泄漏 |
| limit_req | 漏桶；**burst 不加 nodelay = 排队变慢**而非拒绝 |
| real_ip | `set_real_ip_from` 是安全边界，不配等于信任伪造的 XFF |
| WebSocket | `map $http_upgrade` + `proxy_read_timeout 3600s` |
| SSE | `proxy_buffering off` + `gzip off`，或后端发 `X-Accel-Buffering: no` |
| gRPC | 用 `grpc_pass` 和 `grpc_*` 系列超时，`proxy_*` 不生效 |
| add_header | 子 location 有任一条则父级**全部**失效；安全头加 `always` |
| 时间变量 | `$request_time` >> `$upstream_response_time` = **客户端慢** |
| reload | 配置错会保留旧配置；`worker_shutdown_timeout` 必设 |
| 日志轮转 | 用 `-s reopen`（SIGUSR1），不是 reload |
| default_server | 必配一个 `return 444` 吃掉未知 Host |

### 📅 2026 现状

| 项 | 说明 |
|---|---|
| 版本 | stable 1.30.x / mainline 1.31.x；1.27、1.29 已 EOL |
| `http2` 指令 | 1.25.1+ 起用独立的 `http2 on;`，`listen ... http2` 写法已废弃 |
| HTTP/3 | 1.25.0+ 内置 QUIC，`listen 443 quic reuseport;` + `Alt-Svc` 头 |
| WAF | ModSecurity 已 EOL（2024-07），改用 Coraza / CRS / 托管 WAF |
| 商业化 | NGINX One（2024-09）；Plus R33+ 要 JWT license，**OSS 不受影响** |
| 替代品 | 需要动态配置/xDS → Envoy；要自动 HTTPS → Caddy；K8s 入口 → Gateway API |

详见 [B25 §2.7](./B25-精通-Web-服务器与反向代理.md)。

---

本篇是 **B26**，[B25](./B25-精通-Web-服务器与反向代理.md)（横向选型）的纵向补充。关联阅读：

- **协议基础**：[B02 — 精通 HTTP 语义](./B02-精通-HTTP-语义.md)、[B03 — 精通 TLS 与 HTTPS](./B03-精通-TLS-与-HTTPS.md)
- **限流原理**：[B21 — 精通背压与限流](./B21-精通背压与限流.md)
- **缓存策略**：[B15 — 精通缓存策略](./B15-精通缓存策略.md)
- **长连接后端**：[G31 — 精通 Go Socket 与 WebSocket](../golang/G31-精通-Go-Socket-与-WebSocket.md)
- **K8s 入口**：[C04 — 精通 Ingress 与 Gateway API](../cloud-native/C04-精通-Ingress-与-Gateway-API.md)

---
