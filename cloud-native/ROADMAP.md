# 云原生路线图 · Mermaid 可视化

> 配合 [INDEX.md](./INDEX.md) 与 [QUIZ.md](./QUIZ.md) 使用
>
> **📅 内容基准：2026 年 6 月**——Kubernetes 1.36（2026-04，1.34/1.35 仍广泛在用）、Gateway API GA、Istio Ambient GA、Cilium eBPF 主流、Helm 3.x、ArgoCD 3.x。

---

## 🗺️ 全景路线图

```mermaid
graph TD
    Start([开始云原生之旅]) --> M1[模块 1: 容器与编排]

    M1 --> C01[C01 Docker/OCI]
    M1 --> C02[C02 工作负载]
    M1 --> C03[C03 网络/Service]

    C03 --> M2[模块 2: 流量与配置]
    M2 --> C04[C04 Ingress/Gateway API]
    M2 --> C05[C05 ConfigMap/Secret]

    C05 --> M3[模块 3: 资源与存储]
    M3 --> C06[C06 Scheduling/HPA]
    M3 --> C07[C07 Storage/PVC]

    C07 --> M4[模块 4: 扩展与平台]
    M4 --> C08[C08 Helm/Kustomize]
    M4 --> C09[C09 Operator/CRD]

    C09 --> M5[模块 5: 网格/可观测/安全]
    M5 --> C10[C10 Service Mesh]
    M5 --> C11[C11 可观测性]
    M5 --> C12[C12 安全]

    C12 --> M6[模块 6: 生产化]
    M6 --> C13[C13 生产调优]
    M6 --> C14[C14 GitOps]

    C14 --> M7[模块 7: 扩展形态]
    M7 --> C15[C15 Serverless on K8s]
    M7 --> C16[C16 多集群 / Karmada]

    C16 --> End([生产级云原生工程师])

    classDef module fill:#4a5568,stroke:#2d3748,color:#fff
    classDef base fill:#48bb78,stroke:#2f855a,color:#fff
    classDef traffic fill:#4299e1,stroke:#2b6cb0,color:#fff
    classDef res fill:#ecc94b,stroke:#b7791f,color:#000
    classDef ext fill:#9f7aea,stroke:#6b46c1,color:#fff
    classDef sec fill:#f56565,stroke:#c53030,color:#fff
    classDef prod fill:#ed8936,stroke:#c05621,color:#fff
    classDef adv fill:#38b2ac,stroke:#234e52,color:#fff

    class M1,M2,M3,M4,M5,M6,M7 module
    class C01,C02,C03 base
    class C04,C05 traffic
    class C06,C07 res
    class C08,C09 ext
    class C10,C11,C12 sec
    class C13,C14 prod
    class C15,C16 adv
```

---

## 🟢 模块 1：容器与编排基础（C01-C03）

```mermaid
graph LR
    C01[C01 Docker/OCI<br>镜像/层/BuildKit] --> C02[C02 工作负载<br>Pod/Deployment]
    C02 --> C03[C03 网络/Service<br>ClusterIP/CNI]

    classDef base fill:#48bb78,stroke:#2f855a,color:#fff
    class C01,C02,C03 base
```

**工作负载选型**：

```mermaid
flowchart TD
    Need{应用类型?}
    Need -->|"无状态/水平扩展"| Dep["Deployment"]
    Need -->|"有状态/顺序启停"| Sts["StatefulSet"]
    Need -->|"每节点一份"| Ds["DaemonSet"]
    Need -->|"一次性任务"| Job["Job"]
    Need -->|"周期任务"| Cron["CronJob"]
    Need -->|"长连接/IM"| Custom["StatefulSet 或 Custom Operator"]

    style Dep fill:#48bb78,color:#fff
    style Sts fill:#9f7aea,color:#fff
    style Cron fill:#ed8936,color:#fff
```

---

## 🔵 模块 2：流量与配置（C04-C05）

```mermaid
graph TB
    User[外部用户] --> LB[Cloud LB]
    LB --> GW["Gateway / Ingress"]
    GW --> Route["HTTPRoute / Ingress 规则"]
    Route --> Svc["Service"]
    Svc --> Pod[Pod 副本组]

    Cfg["ConfigMap"] -.投递.-> Pod
    Sec["Secret"] -.投递.-> Pod
    ES["External Secrets"] -.同步.-> Sec
    Vault[(HashiCorp Vault)] -.源.-> ES

    style GW fill:#4299e1,color:#fff
    style ES fill:#9f7aea,color:#fff
    style Vault fill:#ed8936,color:#fff
```

**Ingress vs Gateway API**：

```mermaid
flowchart TD
    Old["传统 Ingress"]
    Old --> Limit1["Annotation 爆炸"]
    Old --> Limit2["TCP/UDP/gRPC 弱"]
    Old --> Limit3["角色不清"]

    New["Gateway API"]
    New --> Adv1["GatewayClass / Gateway / *Route 分层"]
    New --> Adv2["原生 HTTP/TLS/TCP/UDP/GRPC"]
    New --> Adv3["平台/应用角色分离"]
    New --> Adv4["v1.0 GA（2023-10），Istio/Cilium/Envoy/Kong 全支持"]

    style New fill:#48bb78,color:#fff
    style Old fill:#f56565,color:#fff
```

---

## 🟡 模块 3：资源与存储（C06-C07）

```mermaid
graph TB
    Pod[Pod] --> Req["Requests<br>调度 + 计费"]
    Pod --> Lim["Limits<br>cgroup 上限"]
    Req & Lim --> QoS{"QoS 等级"}
    QoS --> G["Guaranteed<br>req=lim"]
    QoS --> Bu["Burstable<br>req<lim"]
    QoS --> BE["BestEffort<br>无 req/lim"]

    HPA["HPA"] -->|按指标| Pod
    VPA["VPA"] -->|调 req/lim| Pod
    KEDA["KEDA"] -->|事件驱动| HPA

    style G fill:#48bb78,color:#fff
    style BE fill:#f56565,color:#fff
```

**存储抽象层次**：

```mermaid
graph LR
    Pod[Pod] -->|挂载| PVC["PVC<br>应用视角"]
    PVC -->|绑定| PV["PV<br>集群资源"]
    PV -->|由| SC["StorageClass<br>动态供应"]
    SC -->|调用| CSI["CSI 驱动"]
    CSI -->|创建| Disk["云盘/分布式存储"]

    style PVC fill:#4299e1,color:#fff
    style CSI fill:#9f7aea,color:#fff
```

---

## 🔴 模块 4：扩展与平台（C08-C09）

```mermaid
graph TB
    Manifest[K8s YAML] --> Mode{部署方式}
    Mode -->|"参数化模板"| Helm["Helm Chart"]
    Mode -->|"叠加 overlay"| Kust["Kustomize"]
    Mode -->|"声明业务对象"| Op["Operator + CRD"]

    Op --> Recon["Reconcile Loop<br>watch + diff + act"]
    Recon --> API[Kubernetes API]
    API -.event.-> Recon

    style Op fill:#9f7aea,color:#fff
    style Helm fill:#4299e1,color:#fff
```

**Helm vs Kustomize**：

```mermaid
flowchart TD
    Choice{需求}
    Choice -->|"参数化、模板复杂、生态丰富"| Helm
    Choice -->|"少模板、纯 YAML 叠加、kubectl 原生"| Kust["Kustomize"]
    Choice -->|"两者混合"| Both["Kustomize 调 Helm（render）"]

    style Helm fill:#4299e1,color:#fff
    style Kust fill:#48bb78,color:#fff
```

---

## 🟣 模块 5：网格、可观测、安全（C10-C12）

```mermaid
graph TB
    App[应用 Pod]

    subgraph "Service Mesh"
    Sidecar["Sidecar 模式<br>Istio classic / Linkerd"]
    Ambient["Ambient 模式<br>Istio Ambient / Cilium"]
    end

    App -.可选.-> Sidecar
    App -.可选.-> Ambient

    App -->|metric| Prom["Prometheus / kube-state-metrics"]
    App -->|log| Loki["Loki"]
    App -->|trace| Tempo["Tempo / Jaeger"]
    App -.eBPF.-> Hubble["Hubble / Pixie"]

    App --> RBAC{"RBAC + PSA"}
    RBAC --> Policy["Kyverno / Gatekeeper"]
    Policy --> Image["镜像扫描 Trivy / Cosign 签名"]

    style Ambient fill:#48bb78,color:#fff
    style Hubble fill:#9f7aea,color:#fff
    style Policy fill:#f56565,color:#fff
```

**Service Mesh 演进**：

```mermaid
graph LR
    SC["Sidecar Classic"] -->|每 Pod 一个 proxy| Cost1["资源/延迟开销"]
    Ambient2["Ambient Mesh"] -->|节点级 ztunnel + L7 waypoint| Save["资源更省、热升级"]
    EBPF["Cilium eBPF"] -->|内核态| FastL4["L4 接近零开销"]

    style Ambient2 fill:#48bb78,color:#fff
    style EBPF fill:#9f7aea,color:#fff
```

---

## 🟠 模块 6：生产化（C13-C14）

```mermaid
graph TB
    Git[Git 仓库] --> Argo["ArgoCD / Flux"]
    Argo -->|sync| Cluster1["集群 A"]
    Argo -->|sync| Cluster2["集群 B"]
    Argo -->|sync| ClusterN["集群 N"]

    Argo --> Rollout["Argo Rollouts / Flagger"]
    Rollout --> Canary["Canary / Blue-Green / 渐进式"]

    Argo -.PR 触发.-> CI["CI（构建镜像、更新 tag）"]
    CI --> Git

    style Argo fill:#ed8936,color:#fff
    style Rollout fill:#48bb78,color:#fff
```

**生产架构总览**：

```mermaid
graph TB
    User[Internet] --> CDN
    CDN --> LB[Cloud LB]
    LB --> GW["Gateway API"]
    GW --> Mesh["Service Mesh sidecar/Ambient"]
    Mesh --> App[应用 Pod]

    App --> DB[(PostgreSQL Operator)]
    App --> Cache[(Redis Operator)]
    App --> MQ[(Kafka via Strimzi)]

    App -.metrics.-> Prom
    App -.logs.-> Loki
    App -.traces.-> Tempo

    Prom & Loki & Tempo --> Grafana

    Git --> ArgoCD
    ArgoCD -.sync.-> App

    Falco -.runtime sec.-> App
    Kyverno -.policy.-> App

    style GW fill:#4299e1,color:#fff
    style Mesh fill:#9f7aea,color:#fff
    style ArgoCD fill:#ed8936,color:#fff
    style Falco fill:#f56565,color:#fff
```

---

## 🎯 学习路径可视化

### 路径 A：完整通学（3-4 个月）

```mermaid
gantt
    title 3-4 个月完整云原生通学
    dateFormat YYYY-MM-DD
    section 月 1
    C01-C03 容器与编排    :a1, 2026-05-12, 21d
    section 月 2
    C04-C07 流量配置存储  :a2, after a1, 28d
    section 月 3
    C08-C10 平台扩展+Mesh :a3, after a2, 28d
    section 月 4
    C11-C14 可观测安全生产 :a4, after a3, 28d
```

### 路径 B：应用开发者切入

```mermaid
graph LR
    B1[C01 Docker] --> B2[C02 工作负载]
    B2 --> B3[C03 网络]
    B3 --> B4[C04 Gateway]
    B4 --> B5[C05 配置]
    B5 --> B6[C08 Helm]

    style B1 fill:#48bb78,color:#fff
    style B6 fill:#4299e1,color:#fff
```

### 路径 C：平台 / SRE 工程师

```mermaid
graph LR
    P1[C02+C03 工作负载+网络] --> P2[C06 资源]
    P2 --> P3[C09 Operator]
    P3 --> P4[C10 Mesh]
    P4 --> P5[C11 观测]
    P5 --> P6[C12 安全]
    P6 --> P7[C13-C14 生产 GitOps]

    style P3 fill:#9f7aea,color:#fff
    style P4 fill:#f56565,color:#fff
```

---

## 🧠 云原生核心知识思维导图

```mermaid
mindmap
  root((云原生))
    容器
      Docker/OCI C01
      工作负载 C02
      网络 C03
    流量
      Gateway API C04
      ConfigMap C05
    资源
      Scheduling C06
      Storage C07
    扩展
      Helm C08
      Operator C09
    治理
      Service Mesh C10
      可观测 C11
      安全 C12
    生产
      调优 C13
      GitOps C14
```

---

## 📊 难度与重要性矩阵

| 课程 | 难度 | 重要性 | 备注 |
|---|---|---|---|
| C01 Docker/OCI | ⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 一切起点 |
| C02 工作负载 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | K8s 心智模型 |
| C03 网络/Service | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 流量怎么转一定要懂 |
| C04 Gateway API | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 2026 新标准 |
| C05 ConfigMap/Secret | ⭐⭐⭐ | 🔥🔥🔥🔥 | 配置不能硬编码 |
| C06 Scheduling | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 容量与稳定基础 |
| C07 Storage | ⭐⭐⭐⭐ | 🔥🔥🔥 | 有状态服务必备 |
| C08 Helm/Kustomize | ⭐⭐⭐ | 🔥🔥🔥🔥 | 部署体感 |
| C09 Operator | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 自动化天花板 |
| C10 Service Mesh | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 大规模微服务必选 |
| C11 可观测性 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 没观测 = 黑盒 |
| C12 安全 | ⭐⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 越早越省事 |
| C13 生产调优 | ⭐⭐⭐⭐ | 🔥🔥🔥🔥 | 规模化必读 |
| C14 GitOps | ⭐⭐⭐⭐ | 🔥🔥🔥🔥🔥 | 现代发布事实标准 |

---

## 🔗 与已有课程的关系

| 云原生章节 | 关联已有课程 |
|---|---|
| C01 Docker | golang/G17 Go Modules（多阶段构建） |
| C02 工作负载 | backend/B19 12-Factor App |
| C03 网络 | backend/B01 互联网、B02 HTTP |
| C04 Gateway API | backend/B25 Web 服务器与反向代理 |
| C09 Operator | golang/G27 net/http、G14 context |
| C10 Service Mesh | backend/B18 微服务、B20 韧性 |
| C11 可观测性 | backend/B24 可观测性 |
| C12 安全 | backend/B22 认证、B23 OWASP |
| C13 生产调优 | golang/G22 pprof（容器内 profiling） |
| C14 GitOps | backend/B19 12-Factor |

---

## 🆕 2026 关键技术演进

```mermaid
graph LR
    subgraph 编排基线
    K1[K8s 1.28] --> K2[K8s 1.30 LTS]
    K2 --> K3[K8s 1.36<br>2026 主流]
    end

    subgraph 流量入口
    I1[Ingress + 注解] --> I2[Gateway API v1.0 GA]
    I2 --> I3["Gateway API v1.x<br>多实现稳定"]
    end

    subgraph Service Mesh
    M1[Istio sidecar] --> M2[Istio Ambient GA<br>2024-11]
    M3[Cilium L4] --> M4[Cilium Service Mesh<br>L7 + eBPF]
    end

    subgraph 安全合规
    S1[PSP] --> S2[Pod Security Admission]
    S2 --> S3[Kyverno / Gatekeeper<br>+ SBOM + SLSA]
    end

    subgraph 发布
    R1[手动 kubectl apply] --> R2[Helm + CI]
    R2 --> R3[ArgoCD / Flux GitOps]
    R3 --> R4[Argo Rollouts<br>渐进式发布]
    end

    style K3 fill:#fff3e0
    style I3 fill:#fff3e0
    style M2 fill:#fff3e0
    style S3 fill:#fff3e0
    style R4 fill:#fff3e0
```
