# 精通 Twelve-Factor App

> 课程编号：B19
> 路线图来源：[roadmap.sh/backend](https://roadmap.sh/backend) — Twelve Factor Apps
> 难度：⭐⭐⭐
> 预计阅读时间：40 分钟

---

## 引言：云原生的圣经

2011 年 Heroku 团队总结的 12 条原则，至今仍是云原生应用设计的黄金标准。Kubernetes、Docker、Serverless 的最佳实践都暗合 12-factor。本章逐条讲解，加上现代上下文。

原文：[12factor.net](https://12factor.net/)

---

## I. Codebase（代码库）

> One codebase tracked in revision control, many deploys.

### 含义

- 一份代码库 = 一个 app
- 多个部署（dev、staging、prod）从同一代码库
- 多 app 共享代码 → 抽成 library

### 反例

- 同一仓库多个 app 杂糅（monorepo 是另一回事——monorepo 里每个 app 独立 build）
- 不同环境 fork 代码库

### 现代上下文

- Git/GitHub 标配
- monorepo 工具：Nx、Bazel、Turborepo
- 多 app 共享代码：内部 npm package、Go module replace

---

## II. Dependencies（依赖）

> Explicitly declare and isolate dependencies.

### 含义

- 显式声明依赖（package.json / requirements.txt / go.mod / Cargo.toml）
- 不依赖系统全局安装
- 隔离环境（virtualenv / Docker / nvm）

### 现代

- Container 提供完美隔离
- Lockfile 锁定版本（yarn.lock / poetry.lock / go.sum）
- 多阶段 Dockerfile 区分 build deps 和 runtime

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY go.* ./
RUN go mod download
COPY . .
RUN go build -o /app

FROM gcr.io/distroless/static
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

---

## III. Config（配置）

> Store config in the environment.

### 含义

- 配置（DB URL、API key、feature flag）放**环境变量**，不入代码
- 不同环境用不同 env，代码不变

### 反例

```python
# config.py
PROD_DB_URL = "..."
STAGING_DB_URL = "..."
if env == "prod": ...
```

### 现代

```bash
DATABASE_URL=postgres://...
REDIS_URL=redis://...
JWT_SECRET=...
LOG_LEVEL=info
```

工具：
- `.env` 文件（dev）+ Vault / Secrets Manager（prod）
- K8s ConfigMap + Secret
- AWS Parameter Store / Secrets Manager

### 例外

非敏感**业务配置**（如算法阈值、enum 映射）放代码 OK——它们是代码的一部分，跟版本走。

---

## IV. Backing Services（后端服务）

> Treat backing services as attached resources.

### 含义

DB、缓存、消息队列、邮件服务、对象存储——都视为"附加资源"，通过 URL 连接。本地 / 第三方 / 自托管对应用透明。

### 反例

代码硬编码"localhost:5432"。

### 现代

```
DATABASE_URL=postgres://...
REDIS_URL=redis://...
S3_BUCKET=...
SMTP_URL=smtps://...
```

切换 PostgreSQL on-prem → AWS RDS：改 env 即可，代码不动。

---

## V. Build, Release, Run（构建、发布、运行）

> Strictly separate build and run stages.

### 含义

- **Build**：源码 → artifact（Docker image / jar）
- **Release**：artifact + config → 完整 release（带版本号）
- **Run**：在某环境运行 release

### 反例

直接在生产 `git pull && npm install`——build 在 prod 跑、依赖变化未控、回滚难。

### 现代

CI/CD pipeline：
1. PR 触发 build → Docker image push to registry
2. release stage：image tag + env config → release `v1.2.3`
3. deploy → K8s rollout

回滚 = 切换到旧 release（K8s `kubectl rollout undo`）。

---

## VI. Processes（进程）

> Execute the app as one or more stateless processes.

### 含义

- 应用进程**无状态**——任何状态放 DB / cache / 对象存储
- 重启进程不丢业务数据
- 多实例对等（可水平扩展）

### 反例

- 把 session 存进程内存
- 上传文件存本地 disk
- 在内存累积"今日统计"

### 现代

- session 放 Redis
- 文件放 S3
- 统计累积到 DB / metrics 系统
- K8s pod 任何时刻可被杀

例外：sticky session 在性能极致场景仍存在，但应少用。

---

## VII. Port Binding（端口绑定）

> Export services via port binding.

### 含义

应用自身监听一个端口（HTTP server），不依赖外部 web server（Apache、IIS）。

### 现代

- Go / Node / Python（uvicorn）都内置 server
- 容器化天然符合
- K8s Service 暴露端口

```go
http.ListenAndServe(":8080", handler)
```

例外：传统 PHP 仍 require Apache/nginx；但 PHP-FPM 模式部分实现了这条。

---

## VIII. Concurrency（并发）

> Scale out via the process model.

### 含义

通过**水平扩展进程数**提升并发，而非单进程的内部多线程。

### 现代

- K8s Deployment `replicas: N`
- HPA（Horizontal Pod Autoscaler）按 CPU/QPS 自动扩
- Lambda / serverless 极致

进程内仍可多线程 / goroutine——但水平扩展是核心。

---

## IX. Disposability（易处置性）

> Maximize robustness with fast startup and graceful shutdown.

### 含义

- 进程启动**快**（几秒内）
- 处理 SIGTERM 优雅退出（清理资源、完成 in-flight 请求）
- 任何时刻可被杀仍能恢复

### 现代

K8s rolling update + readiness/liveness probe 依赖这一点。
- 启动慢 → 部署慢、扩容慢
- 不响应 SIGTERM → kubelet 等 grace period 后强杀 → 处理中请求丢

实现：参考 G27 Graceful Shutdown。

```go
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

go server.ListenAndServe()
<-ctx.Done()
server.Shutdown(shutCtx)
```

---

## X. Dev/Prod Parity（开发生产等价）

> Keep development, staging, and production as similar as possible.

### 含义

dev、staging、prod **尽可能相似**：
- 同样的 OS（Docker 解决）
- 同样的依赖
- 同样的 backing service（dev 用 PostgreSQL，不要用 SQLite）

### 反例

dev SQLite 测试通过 → prod PostgreSQL 出现兼容性问题。

### 现代

- docker-compose 起开发环境
- testcontainers 测试时拉真实服务
- Tilt / Skaffold 在本地起类生产环境

---

## XI. Logs（日志）

> Treat logs as event streams.

### 含义

- 应用**不**管理日志文件——直接写 stdout/stderr
- 部署环境（K8s、Docker）收集 + 转发到聚合系统

### 反例

```python
logging.FileHandler("/var/log/app.log")
```

应用维护日志文件、轮转 → 在容器中是反模式。

### 现代

```python
logging.StreamHandler(sys.stdout)
```

```yaml
# K8s 自动收集 stdout
# 转发到 ELK / Loki / Datadog
```

---

## XII. Admin Processes（管理进程）

> Run admin/management tasks as one-off processes.

### 含义

数据迁移、数据修复、批量脚本——作为**一次性进程**运行，使用与生产相同的代码 + env。

### 反例

通过 SSH 进生产容器手动跑命令。

### 现代

- K8s Job
- Rake task、Django management command、Go CLI 子命令
- 数据库迁移用专门工具（Flyway、Atlas、golang-migrate）

```yaml
# K8s job
apiVersion: batch/v1
kind: Job
metadata: { name: db-migrate }
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:v1.2.3
          command: ["./app", "migrate"]
```

---

## 第十三章：常见反模式

### ❌ 反模式 1：单实例假设

应用代码假设"我是唯一实例"——存了进程内 cache、计数器。多副本部署后崩。

### ❌ 反模式 2：本地文件存储

上传图片到 `/uploads`。Pod 重启全丢；多副本看不到对方上传。改 S3。

### ❌ 反模式 3：硬编码配置

production code 含 prod URL。要部署 staging？复制一份代码改 URL。

### ❌ 反模式 4：长启动

启动 30 秒——K8s 误判挂掉重启循环。优化或调 readiness 等待时间。

### ❌ 反模式 5：忽略 SIGTERM

进程被 kill -9 → 处理中请求中断。

### ❌ 反模式 6：dev 用 SQLite，prod 用 Postgres

dev/prod parity 缺失。bug 只在 prod 出现。

### ❌ 反模式 7：手动 SSH 跑迁移

无审计、易错。脚本 + Job + 版本化。

---

## 第十四章：超越 12 因素

### 14.1 Kevin Hoffman 的"Beyond 12-Factor"（2016）

加了几条：
- **API first**：先定义 API contract
- **Telemetry**：内置 metric + log + trace
- **Authentication & Authorization**：从一开始就考虑
- **Backing services 的健康检查**

### 14.2 Cloud Native Foundation 12+

CNCF 鼓励：
- 容器化
- 微服务（视情况）
- 声明式 API（K8s 风格）
- 健康 + 可观测
- 自动化运维

---

## 第十五章：与 K8s 的关系

12-factor 是 K8s 设计的精神基础。每条都对应 K8s 功能：

| Factor | K8s 对应 |
|---|---|
| I Codebase | Container image |
| II Dependencies | Image base |
| III Config | ConfigMap + Secret |
| IV Backing services | Service + ExternalName |
| V Build/Run | Image tags + Deployment versions |
| VI Processes | Pod 无状态 |
| VII Port binding | Service |
| VIII Concurrency | replicas + HPA |
| IX Disposability | preStop + termination grace period |
| X Dev/Prod parity | 同 image |
| XI Logs | stdout → fluentd / Loki |
| XII Admin | Job |

K8s 不强制 12-factor，但**遵循 12-factor 的应用在 K8s 上跑得最顺**。

---

## 第十六章：生产级最佳实践

1. **配置全部 env**：`.env.example` 提交，`.env` 不入 git。
2. **secret 管理**：Vault、AWS Secrets Manager、K8s Secret + sealed-secrets。
3. **Docker 化所有服务**：dev/prod parity。
4. **stdout 日志**：让基础设施收集。
5. **优雅退出 + 启动健康检查**：K8s rolling update 不丢请求。
6. **多副本无状态**：随时可扩 / 杀。
7. **数据库迁移作为 Job**：脚本化、有版本、可回滚。
8. **CI/CD build 一次部署多处**：image tag 是唯一 release ID。
9. **依赖 lockfile commit**：reproducible build。
10. **不在生产 SSH 改东西**：所有变更通过 CI/CD。

---

## 第十七章：常见陷阱清单

### ❌ 陷阱 1：env 包含密钥提交 git
泄漏。`.env` 加 .gitignore；secret 管理工具。

### ❌ 陷阱 2：进程内 cache 跨实例不一致
多副本看到不同数据。改 Redis。

### ❌ 陷阱 3：log 文件 + logrotate
容器内打不通基础设施。stdout 即可。

### ❌ 陷阱 4：build artifact 在 prod 重新编译
慢、不一致。CI 出 image，prod 直接拉。

### ❌ 陷阱 5：dev 用 mocks，prod 才连真服务
集成 bug 只在 prod 暴露。testcontainers 用真服务。

### ❌ 陷阱 6：启动连 backing service 失败 → crash
重启循环。指数退避 + readiness probe 等待。

### ❌ 陷阱 7：在线 patch
hotfix 直接编辑 prod 容器内代码 → 下次重启丢失。永远通过 build pipeline。

---

## 第十八章：练习题

**练习 1**：列出 12 条因素的名字（不看答案）。

**练习 2**：以下违反哪条？修复。
```python
def get_db():
    if os.environ.get("ENV") == "prod":
        return psycopg2.connect("...prod...")
    return psycopg2.connect("...dev...")
```

**练习 3**：上传文件保存到 `/uploads`，多副本后用户看不到——违反哪条？

**练习 4**：解释为何"开发 SQLite + 生产 PostgreSQL"是反模式。

**练习 5**：如何在 K8s 中实现 factor IX（disposability）？

---

## 参考答案

**练习 1**：
I Codebase / II Dependencies / III Config / IV Backing services / V Build, release, run / VI Processes / VII Port binding / VIII Concurrency / IX Disposability / X Dev/Prod parity / XI Logs / XII Admin processes

**练习 2**：违反 III Config（代码含环境判断 + 内嵌连接串）。修：
```python
def get_db():
    return psycopg2.connect(os.environ["DATABASE_URL"])
```

**练习 3**：违反 VI Processes（有状态）。修：上传到 S3 / 共享存储。

**练习 4**：违反 X Dev/Prod parity。SQLite 与 PostgreSQL 语法、并发模型、类型行为都不同——dev 测试通过的代码可能 prod 失败（SERIAL / JSONB / 隔离级别 / 并发锁）。

**练习 5**：
- 进程接收 SIGTERM 信号 → graceful shutdown（参考 G27）
- K8s `terminationGracePeriodSeconds` 给足时间
- `preStop` hook 把流量从 service 中拿走（如 readiness 转 false 后等 ELB 检测）
- readiness probe / liveness probe 配置

---

## 小结

| # | Factor | 一句话 |
|---|---|---|
| I | Codebase | 一份代码多个部署 |
| II | Dependencies | 显式声明 + 隔离 |
| III | Config | 放 env，别入码 |
| IV | Backing services | 视为附加资源 |
| V | Build/Release/Run | 三阶段分离 |
| VI | Processes | 无状态 |
| VII | Port binding | 自监听端口 |
| VIII | Concurrency | 水平扩展进程 |
| IX | Disposability | 快启快停 |
| X | Dev/Prod parity | 环境一致 |
| XI | Logs | stdout 事件流 |
| XII | Admin | 一次性进程 |

下一篇 **B20 — 韧性模式** 将系统讲熔断、重试、超时、Bulkhead、降级。

---

