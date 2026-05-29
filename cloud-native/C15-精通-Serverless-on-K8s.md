# 精通 Serverless on Kubernetes：Knative、KEDA HTTP 与 OpenFaaS 深度

> 课程编号：C15
> 路线图来源：云原生工程 · 模块七 扩展形态
> 难度：⭐⭐⭐⭐
> 预计阅读时间：65 分钟
> 内容基准：2026 年 5 月

---

## 引言：把"按请求计费 + 自动伸缩到 0"搬进 K8s

公有云 Serverless（AWS Lambda / Cloud Run / Azure Functions）的核心卖点：

- **按用量计费**——没流量就不花钱
- **scale-to-zero**——闲置时不占资源
- **极简部署**——不用管 Node / Pod / 副本数

而 K8s 原生 Deployment 是"常驻 Pod"思路——最小副本 1 就要常年占资源、HPA 最少也要 1 个 Pod。

把 Serverless 体验带进 K8s，就是**云原生 Serverless 框架**要解决的问题：

- **Knative Serving** — CNCF 毕业，Cloud Run 的开源对应物
- **KEDA HTTP add-on** — 给 KEDA 加 HTTP 触发器，scale-to-zero 极简方案
- **OpenFaaS** — Function-as-a-Service 风格，更轻量
- **Fission / Kubeless** — 较早期，2026 影响力下降

本章重点拆 **Knative** 与 **KEDA HTTP**——这两个是 2026 年 K8s Serverless 主流。

---

## 第一章：Serverless 在 K8s 上要解决什么

### 1.1 K8s 原生伸缩的极限

```
HPA min=1 → 永远占 1 个 Pod 资源
HPA min=0 → 不支持（HPA 不允许 0）
```

如果一个微服务一天只被调用 100 次，每次 100ms，**99.9% 时间在空跑**。100 个这样的服务就是 100 个常驻 Pod，浪费巨大。

### 1.2 Serverless 框架要做的三件事

```
1. 流量为 0 时把副本缩到 0
   - 但又要能"听见请求"，所以需要"激活器" (activator) 来兜请求
2. 流量来了从 0 拉起 Pod（冷启动）
   - 拉起期间请求要 buffer 住，不能 502
3. 按精细化指标（QPS / 并发）伸缩，而不是 CPU
   - Web 服务 CPU 30% 不代表能扛流量
```

### 1.3 与 FaaS 的关系

```
FaaS（函数即服务）
  - 写函数 / 单文件
  - 平台代管 runtime
  - 每次冷启动从 zip 解压

Serverless（无服务器服务）
  - 写完整应用 (容器)
  - 跑在容器 runtime
  - scale-to-zero + 按请求伸缩
```

Knative 偏 Serverless（容器为单位），OpenFaaS 偏 FaaS（函数为单位）。

---

## 第二章：Knative Serving 架构深拆

Knative 是 Google 2018 推出、2022 年成为 CNCF 孵化项目、2025-09 毕业（Graduated）的项目，**Cloud Run 在 K8s 上的开源对等**。

### 2.1 核心组件

```
┌────────────────────────────────────────────────────────────┐
│                      Knative Serving                       │
│                                                            │
│  ┌──────────────┐                  ┌─────────────────┐   │
│  │  Controller  │ ─── reconcile ──►│  Service / Conf │   │
│  └──────────────┘                  │  /Revision/Route│   │
│                                    └─────────────────┘   │
│                                                            │
│  ┌──────────────┐    metrics       ┌─────────────────┐   │
│  │     Auto     │ ◄────────────────│  queue-proxy    │   │
│  │    Scaler    │                  │  (sidecar)      │   │
│  └──────┬───────┘                  └─────────────────┘   │
│         │ desired_replicas                                │
│         ▼                                                  │
│  ┌──────────────┐                                         │
│  │   K8s API    │ scale Deployment                        │
│  └──────────────┘                                         │
│                                                            │
│  ┌──────────────┐ ◄── 流量为 0 时所有请求都先进它 ──     │
│  │  Activator   │ ──── 拉起 Pod 后 ──► user pod            │
│  └──────────────┘                                         │
└────────────────────────────────────────────────────────────┘
```

- **Service**：用户面 CRD，对应一个"可伸缩的应用"
- **Configuration**：用户的容器与运行配置（image、env、resources）
- **Revision**：每次变更生成一个不可变快照（类似 ReplicaSet 但更轻）
- **Route**：流量入口，可对多 Revision 做权重切分（金丝雀天然支持）
- **Autoscaler (KPA)**：Knative 自家自动扩缩器，基于 QPS / 并发
- **Activator**：scale-to-zero 时所有请求经它，负责拉起 Pod
- **queue-proxy**：每个 Pod 注入的 sidecar，统计请求并上报

### 2.2 一个最小 Knative Service

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "100"
        autoscaling.knative.dev/target: "10"  # 每 pod 目标并发数
    spec:
      containers:
      - image: ghcr.io/me/hello:v1
        ports:
        - containerPort: 8080
        env:
        - name: TARGET
          value: "World"
```

部署后：

- 没流量时副本数 = 0
- 第一个请求来时 Activator 兜住，拉起 1 个 Pod
- 流量增长时按 target concurrency 扩副本
- 流量停止 60 秒（可配 `scale-to-zero-grace-period`）后缩到 0

### 2.3 KPA vs HPA

| 维度 | HPA | KPA (Knative) |
|---|---|---|
| 指标 | CPU / 内存 / 自定义 | QPS / 并发 |
| 最小副本 | ≥ 1 | 0（默认） |
| 反应速度 | 15-30s 周期 | 2s panic 模式 |
| 适用 | 通用 | 请求-响应型服务 |

KPA 的"panic 模式"：流量突增 2× 时切到秒级 reaction，避免 HPA 的慢响应。

### 2.4 Cold Start 分解

```
0ms   client 发请求
   ↓
2ms   ingress / kourier 进集群
   ↓
5ms   Activator 收到，发现副本=0，向 Autoscaler 申请 scale
   ↓
50ms  Autoscaler 调 K8s API，scale Deployment from 0 to 1
   ↓
200ms K8s Scheduler 选节点（如果资源够，秒级；不够要拉新节点慢得多）
   ↓
500ms kubelet 拉镜像（已缓存秒级，未缓存几十秒）
   ↓
2000ms 容器启动 + readiness probe pass
   ↓
2050ms Activator 把请求转给新 Pod
   ↓
2200ms 用户拿到响应
```

冷启动 P95 通常在 **2-5 秒**，对实时业务可能不能接受。

### 2.5 冷启动优化

1. **min-scale: 1**：放弃 scale-to-zero，常驻 1 个 Pod（关键业务推荐）
2. **镜像 < 50MB**：减拉取时间（用 distroless / scratch）
3. **节点池预热**：保留足够资源不要等 scale-up
4. **`scale-down-delay`**：缩容延迟，避免抖动后又冷启
5. **Image preload**：节点上预先拉好镜像（C13 详述）
6. **JVM AOT / Native Image**：Java 应用启动几秒 → 几百毫秒

---

## 第三章：Knative 流量管理

### 3.1 Revision 与金丝雀

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: api
spec:
  template:
    metadata:
      name: api-v2  # 命名后会创建 Revision: api-v2
    spec:
      containers:
      - image: ghcr.io/me/api:v2
  traffic:
  - revisionName: api-v1
    percent: 90
  - revisionName: api-v2
    percent: 10
```

直接在 Service 里就能配权重切分——**零代码金丝雀**。

### 3.2 Tag 路由

```yaml
traffic:
- revisionName: api-v1
  percent: 100
- revisionName: api-v2
  percent: 0
  tag: canary
```

部署后会自动生成：

- `api.default.example.com` → v1 100%
- `canary-api.default.example.com` → v2 100%（仅内部测试用）

### 3.3 与 Gateway API 整合

Knative 1.13+ 支持 Gateway API 作为流量层（替代默认的 Kourier / Istio）。在已用 Gateway API 的集群上无缝整合。

```bash
kubectl apply -f https://github.com/knative-extensions/net-gateway-api/releases/...
```

---

## 第四章：Knative Eventing——事件驱动架构

Knative 不只 Serving，还有 Eventing：在 K8s 里玩 CloudEvents 风格的事件总线。

### 4.1 核心模型

```
Source（事件源） → Broker（事件总线） → Trigger（订阅） → Sink（事件消费者）
```

```yaml
# 一个 broker（基于 Kafka / RabbitMQ / In-Memory）
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
spec:
  config:
    apiVersion: v1
    kind: ConfigMap
    name: kafka-broker-config

---
# 一个订阅：把 type=order.created 的事件路由到 order-handler 服务
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-handler
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-handler
```

### 4.2 Source 生态

- **KafkaSource**：Kafka topic → Broker
- **PingSource**：定时事件（cron）
- **APIServerSource**：K8s API 事件
- **GitHubSource**：GitHub Webhook
- **AWS / GCP Sources**：S3 / Pub/Sub 等

### 4.3 与 Serving 联动

事件触发 Serving Service → Service 缩到 0 时事件触发会拉起 → 处理完缩回 0。
**典型 Serverless 工作流**——零常驻成本。

---

## 第五章：KEDA HTTP Add-on——轻量级方案

Knative 体量大、组件多（kourier / activator / autoscaler / webhook / controller / queue-proxy），运维负担不小。

**KEDA HTTP add-on**（仍为 beta，0.x，未 GA，生产慎用）只把"HTTP 触发 + scale-to-zero"那部分做好，**保留 K8s 原生 Deployment**。

### 5.1 架构

```
client
  │
  ▼
Ingress / Gateway API
  │
  ▼
HTTPScaledObject (CRD)
  │
  ▼
KEDA HTTP Interceptor (queueing)
  │
  ▼
your Deployment (scale by KEDA HPA, can be 0)
```

### 5.2 用法

```yaml
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: my-app
spec:
  hosts:
  - my-app.example.com
  scaleTargetRef:
    name: my-app             # 你的 Deployment 名
    service: my-app          # 你的 Service 名
    port: 8080
  replicas:
    min: 0
    max: 50
  scalingMetric:
    requestRate:
      granularity: 1s
      targetValue: 100        # 每个副本 100 req/s
      window: 1m
```

- 没流量时 KEDA 缩 Deployment 到 0
- 请求来时 HTTP Interceptor 兜住 → 触发 scale up → 转发请求
- 流量持续时按 requestRate 扩缩

### 5.3 KEDA 整体生态

KEDA HTTP 只是 KEDA 多 scaler 中的一个。KEDA 主要是**事件驱动的 HPA 扩展**：

- **Kafka scaler**：按 consumer lag 扩缩
- **RabbitMQ scaler**：按队列长度扩缩
- **Cron scaler**：按时间段扩缩（白天多副本、晚上少）
- **Prometheus scaler**：按任意 PromQL 扩缩
- **AWS SQS / Azure Queue / GCP Pub/Sub**：云队列扩缩

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer
spec:
  scaleTargetRef:
    name: my-consumer
  minReplicaCount: 0
  maxReplicaCount: 100
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      topic: orders
      consumerGroup: order-consumer
      lagThreshold: "100"
```

Kafka topic 没消息 → Consumer 缩到 0；消息堆积 lag > 100 → 拉起 Pod 消费。

### 5.4 KEDA vs Knative

| 维度 | Knative Serving | KEDA HTTP |
|---|---|---|
| 复杂度 | 高（多组件） | 低（仅一个 controller + interceptor） |
| 流量切分 / 金丝雀 | 原生 | 用 Gateway API 自己拼 |
| Revision / 历史 | 内置 | 无 |
| 冷启动延迟 | 2-5s | 1-3s |
| 事件驱动 | 配合 Eventing | KEDA 本就是事件驱动专家 |
| 学习曲线 | 高 | 低 |

**选型**：

- 想要"Cloud Run on K8s 完整体验" → Knative
- 已有 K8s + 想给 Deployment 加 scale-to-zero → KEDA HTTP
- 事件驱动（Kafka / Queue）为主 → KEDA（不一定要 HTTP add-on）

---

## 第六章：OpenFaaS——FaaS 风格

不同于上面以"容器"为单位，OpenFaaS 以**函数**为单位。

### 6.1 函数定义

```yaml
# stack.yml
provider:
  name: openfaas
  gateway: http://gateway.openfaas:8080

functions:
  echo:
    lang: go
    handler: ./echo
    image: ghcr.io/me/echo:v1
```

handler.go：

```go
package function

import (
    "io/ioutil"
    "fmt"
    handler "github.com/openfaas/templates-sdk/go-http"
)

func Handle(req handler.Request) (handler.Response, error) {
    return handler.Response{
        Body:       []byte("hello " + string(req.Body)),
        StatusCode: 200,
    }, nil
}
```

部署：

```bash
faas-cli up -f stack.yml
```

### 6.2 OpenFaaS 优劣

✅ 真正 FaaS 体验（无需写 Dockerfile / yaml）
✅ 模板生态丰富（Go / Python / Node / Rust ...）
✅ 自带 UI / metrics / async invocation
✅ 轻量易部署

❌ 容器封装层多一层，性能不如直接 K8s
❌ scale-to-zero 默认关闭（需配置 OpenFaaS Pro）
❌ 社区版功能受限

适合：**内部脚本平台 / 黑客马拉松 / 数据处理任务**——不适合追求极致 SLA 的核心业务。

---

## 第七章：Serverless 与微服务的本质差异

许多团队混淆"上 Knative"和"做微服务"。

| 维度 | 微服务 | Serverless |
|---|---|---|
| 副本数 | 通常常驻 ≥ 1 | 闲时 0 |
| 启动时间 | 几秒-几十秒 | 必须秒级，否则冷启动暴雷 |
| 适合负载 | 长连接 / 持续高 QPS | 突发 / 偶发 / 低频 |
| 状态 | 可有内存状态（配合 PVC） | 通常无状态 |
| 计费 | 节点常驻 | 按请求时长 |

**适合 Serverless 的场景**：

- **低频管理后台**（一天访问几次，没必要常驻）
- **批量任务 / 数据处理**（按事件触发）
- **Webhook 接收**（GitHub / Stripe webhooks）
- **图像 / 文件处理**（上传后触发）
- **AI 推理 prefill**（请求并发可变）

**不适合的场景**：

- 长连接（WebSocket / gRPC streaming）
- 强一致性数据库主节点
- 启动慢的应用（大 JVM / 大模型加载）—— 冷启动 30 秒用户跑光
- P99 严格的核心交易

---

## 第八章：生产实践

### 8.1 监控

Knative 暴露 metrics：

- `revision_request_count`：每 Revision 请求数
- `revision_request_latencies_*`：延迟分桶
- `actual_pods` / `desired_pods`：实际 vs 期望副本
- `revision_app_request_count` （来自 user app）

KEDA 暴露：

- `keda_metrics_adapter_scaler_*`：每个 scaler 的指标
- `keda_scaled_object_paused`：是否被暂停

必看面板：

```
- Cold start frequency (从 0 拉起的次数)
- Scale-to-zero idle time (闲置缩到 0 的时长占比)
- 冷启动 P50 / P95 延迟
- Activator queue depth
```

### 8.2 限流保护

scale-to-zero 系统在突发 1000 QPS 时仍然要慢慢扩——可能直接打挂。

```yaml
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/max-scale: "100"  # 防止失控扩容
        autoscaling.knative.dev/max-scale-up-rate: "10"  # 每 stable window 最多翻 10×
```

外加 Gateway API rate limit 兜底。

### 8.3 与 Mesh 整合

- Knative 默认用 kourier / Istio gateway，可换 Gateway API
- 与 Istio Ambient 配合：把 Knative Service 所在 namespace 加 `istio.io/dataplane-mode: ambient`，自动 mTLS + L4 mesh
- 注意：sidecar-mode Istio 与 Knative queue-proxy 共存会出错（多个 sidecar 抢端口）—— Ambient 模式没问题

### 8.4 多租户与配额

scale-to-zero 看起来省资源，但容易被滥用：

- 100 个 namespace 各部署 1 个 service，闲时都是 0 但任何时候触发都会同时拉起
- 突发流量场景下"几乎无限扩"可能压垮集群

配 ResourceQuota / LimitRange：

```yaml
apiVersion: v1
kind: ResourceQuota
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "50"
```

---

## 第九章：陷阱清单

1. **冷启动 5 秒被用户骂**——核心业务 `min-scale: 1` 是良心解。
2. **JVM Serverless 冷启动 30s**——Java 不适合 scale-to-zero，除非用 GraalVM Native。
3. **数据库连接池冷启动建池**——每次冷启动建 20 个连接，DB 受不了。要么用 PgBouncer 中间层，要么 lazy init。
4. **Knative 与 Istio sidecar 并存**——queue-proxy + envoy 两个 sidecar 抢端口 / 流量；用 Ambient 或关 sidecar 注入。
5. **`max-scale` 没设上限**——突发流量扩到 1000 Pod，资源耗尽 + Provider 配额爆。
6. **冷启动时 readiness probe 太慢**——initialDelaySeconds 50s，前 50s 全报 502。
7. **CronJob 触发 Knative 不可靠**——cron 5min 跑一次，5min 内 Pod 缩 0，再触发又冷启动——配合 PingSource 更稳。
8. **不监控 cold start 指标**——用户体验跌而仪表盘平静。
9. **Knative Revision 累积**——每次 deploy 留一个 Revision，半年后 ETCD 几千 Revision。要 GC。
10. **Eventing Broker 选型错**——in-memory broker 重启丢消息，生产用 Kafka broker。
11. **KEDA HTTP 与 Ingress 配错**——流量绕过 Interceptor 直接打 Service，scale-to-zero 不生效。
12. **OpenFaaS scale-to-zero 默认关**——以为开了实际没开，资源照样耗。

---

## 第十章：2026 现状

- **Knative 1.22（2026-04）/1.21** 受支持版本，CNCF Graduated（2025-09）
- **Knative ServiceMesh integration** 与 Istio Ambient 官方协同方案 GA
- **KEDA 2.16+** 是 CNCF Graduated（2023-08），HTTP add-on 仍为 beta（0.x），未 GA，生产慎用
- **Cloud Run for K8s** Google 推动 Knative 与 Cloud Run API 兼容
- **OpenFaaS Pro** 商业化推进，社区版相对停滞
- **Wasm Serverless 兴起**：SpinKube / Knative + WasmEdge / WASM as runtime，毫秒级冷启动（详见 C17）
- **AI 推理 Serverless**：KServe 用 Knative 给 ML 模型做 scale-to-zero，AI/ML 平台标配
- **多云 Serverless**：Crossplane + Knative 跨云一致体验
- **Edge Serverless**：K3s + Knative 在边缘节点（CDN POP）部署函数，毫秒级响应

---

## 第十一章：练习题

1. ⭐ Knative Service 缩到 0 后请求怎么"被听见"？Activator 的角色是什么？
2. ⭐ KPA 与 HPA 三个核心差异？
3. ⭐⭐ 写一个 Knative Service YAML：min-scale=0、max-scale=50、target concurrency=20，并配 v1 70% / v2 30% 流量切分。
4. ⭐⭐ KEDA 与 Knative 都能做 scale-to-zero，关键差异是什么？什么场景该用哪个？
5. ⭐⭐⭐ 给一个 Java Spring Boot 服务部署到 Knative，冷启动 12 秒。给出 5 条优化方案。
6. ⭐⭐⭐ 用 KEDA Kafka scaler + Knative 设计一个"订单事件 → 异步处理"的 Serverless pipeline。
7. ⭐⭐⭐ Knative Service 配合 Istio Ambient 时为什么不会冲突？sidecar 模式为什么会冲突？

<details>
<summary>📝 参考思路</summary>

1. Activator 是流量为 0 时的"代答者"——所有请求先到它，它检查目标 Revision 副本数，若为 0 就触发 Autoscaler 拉起 Pod，并在 buffer 内 hold 请求，新 Pod ready 后转发。
2. ① KPA 指标 = 并发 / QPS，HPA = CPU / Mem / 自定义；② KPA 最小副本可为 0，HPA ≥ 1；③ KPA 有 panic 模式秒级反应，HPA 周期 15-30s。
3. 见第三章 3.1 节，把 min/max/target 加上即可。
4. KEDA 保留 K8s 原生 Deployment + 多 scaler 事件驱动；Knative 有完整 Revision / Route / Eventing 生态。事件驱动队列触发 → KEDA；HTTP 流量金丝雀 / Cloud Run 体验 → Knative。
5. ① GraalVM Native Image；② 镜像减肥；③ JIT warmup endpoint；④ min-scale=1 给关键 endpoint；⑤ lazy-init Beans + 提早 readiness（Spring Boot 3 的 `spring.main.lazy-initialization=true`）。
6. KafkaSource → Broker → Trigger → Knative Service `order-processor`；processor 缩 0 时事件触发拉起，处理完缩回 0。
7. Ambient 在节点级 ztunnel 拦流量，不注 sidecar，所以与 queue-proxy 不冲突；sidecar 模式注 envoy 到 Pod，会与 queue-proxy 抢端口 / netfilter 规则导致流量乱。

</details>

---

## 小结

K8s 上跑 Serverless 不是要替代微服务，而是给**低频 / 突发 / 事件驱动**的负载一条更经济的路径。

```
选 Knative          → 想要 Cloud Run 体验、要 Revision / 流量切分
选 KEDA (HTTP)      → 给现有 Deployment 加 scale-to-zero，最小侵入
选 OpenFaaS         → FaaS 风格、内部脚本平台
选 KServe           → AI 推理 scale-to-zero
不选 Serverless     → 长连接 / 高并发 / 极致 P99 / 启动慢
```

下一步去看 [C16 多集群与 Karmada](./C16-精通-多集群与-Karmada.md)——单集群里 scale-to-zero 解决资源效率，多集群则解决可用性与 region 容灾。
