# 精通 Kubernetes Scheduling 与资源管理：Requests/Limits、QoS、HPA/VPA/KEDA 与 Karpenter

> 课程编号：C06
> 路线图来源：云原生 · 模块三 资源与存储
> 难度：⭐⭐⭐⭐
> 预计阅读时间：80 分钟
> 内容基准：2026 年 6 月（Kubernetes 1.35 / 1.34 仍在支持 / KEDA 2.x / VPA 1.x / Karpenter 1.x）

---

## 引言：调度问题 = 把 Pod 放到合适的节点

```bash
$ kubectl get pod my-app -o jsonpath='{.spec.nodeName}'
ip-10-0-23-58.ec2.internal
```

一行命令背后，是 Kubernetes 调度器（kube-scheduler）做了一系列决策：

```
1. 这个 Pod 要多少 CPU / 内存 / GPU？
2. 哪些节点有这么多剩余资源？（Filter）
3. 在能放下的节点里，哪个"最好"？（Score）
4. 这个 Pod 是否要避开/靠近某些 Pod？（Affinity / AntiAffinity）
5. 这个节点有 Taint，我能不能容忍？（Toleration）
6. 优先级是不是高到可以抢占别人的位置？（Priority / Preemption）
7. 是否要分散到不同的拓扑域？（TopologySpreadConstraints）
```

最后 scheduler 在一两毫秒内决定一个 nodeName，写回 Pod spec，kubelet 拉镜像、起容器。**整个调度发生在 Pod 创建到容器启动之间**——不是运行时持续调度。

但调度只是开始。Pod 跑起来之后还有第二层问题：**资源管理**：

```
- 容器吃了多少 CPU？被 throttle 了吗？
- 内存超过 limit 是 OOM 还是允许 swap？（cgroup v2 加了 memory.high）
- 流量来了要不要扩容？（HPA）
- 我估的 requests 是不是太高了？（VPA 帮你调）
- 消息队列堆积要不要扩 consumer？（KEDA）
- 节点都满了怎么办？（Cluster Autoscaler / Karpenter）
```

调度（**把 Pod 放到节点**）和资源管理（**Pod 跑起来后的资源策略**）是 Kubernetes 平台层最难也最关键的部分。前者关乎"能不能上线"，后者关乎"线上稳不稳、贵不贵"。

2026 年 5 月，K8s 调度生态已经相当成熟，但坑也多——尤其是：

- **Requests 是调度依据，但很多团队抄一个默认值就 N 个服务用同一个值**，结果不是 OOM 就是浪费
- **HPA 默认只看 CPU**，但 IO bound 业务（API 网关、消息消费）CPU 永远不到阈值——业务卡死但 HPA 不扩
- **VPA 与 HPA 同用**会打架（都改同一个维度）——必须用 KEDA 或 VPA 的 `OffMode` 配合
- **In-place Pod Vertical Scaling**（不重启 Pod 改 resources）在 1.33 beta（默认开启）、1.35 GA
- **Karpenter** 取代 Cluster Autoscaler 成为云原生节点伸缩主流（AWS / Azure 都有官方支持）

本章把调度与资源管理的核心拆开：从 Requests/Limits 的语义，到调度器决策链，到 HPA/VPA/KEDA 三种伸缩策略，再到节点级伸缩（Cluster Autoscaler / Karpenter），最后总结生产实践与陷阱。

---

## 第一章：Requests vs Limits——一切的起点

### 1.1 两个值的语义

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  containers:
  - name: app
    image: myapi:1.0
    resources:
      requests:
        cpu: "500m"      # 0.5 核
        memory: "512Mi"
      limits:
        cpu: "2000m"     # 2 核
        memory: "1Gi"
```

**Requests**：

- **调度依据**——scheduler 看节点 `Allocatable - sum(requests)` 是否够放
- 容器**保证**能拿到这么多资源（cgroup 设 cpu.shares / memory.min）
- 计费依据（cloud provider 大多按 requests 算）
- **不是上限**——容器可以用比 requests 多的资源（如果节点空闲）

**Limits**：

- **硬上限**——cgroup 限制
- CPU limit 超过 → throttle（节流，不杀进程，但延迟飙升）
- Memory limit 超过 → OOMKilled（杀容器，**不可恢复**直接重启）
- **调度时不看 limits**——只看 requests

### 1.2 cgroup 映射（cgroup v2，2026 年主流）

| K8s 字段 | cgroup v2 文件 | 含义 |
|---|---|---|
| `requests.cpu: 500m` | `cpu.weight` | CPU 权重——节点繁忙时按权重分配 |
| `limits.cpu: 2000m` | `cpu.max` | CPU 上限——超过 throttle |
| `requests.memory: 512Mi` | `memory.min`（保留）| 内存软保留——内核回收时优先保护（**仅在启用 MemoryQoS feature gate 时**，alpha 且默认关闭；默认配置下 requests.memory 仅用于调度，不写入 cgroup）|
| `limits.memory: 1Gi` | `memory.max` | 内存上限——超过 OOM |

cgroup v1（老节点）映射略不同：

| K8s 字段 | cgroup v1 文件 |
|---|---|
| `requests.cpu: 500m` | `cpu.shares = 512` |
| `limits.cpu: 2000m` | `cpu.cfs_quota_us=200000, cpu.cfs_period_us=100000`（100ms 周期里能用 200ms CPU 时间，相当于 2 核）|
| `requests.memory` | 无对应（仅调度） |
| `limits.memory` | `memory.limit_in_bytes` |

**关键差异**：cgroup v1 的 requests.memory **完全不影响运行时**——只有调度时看。v2 在启用 MemoryQoS feature gate（alpha，默认关闭）后才会通过 `memory.min` 给出软保留语义；默认配置下 v2 同样仅用于调度，requests.memory 不写入 cgroup。

### 1.3 单位与换算

```
CPU 单位：
  1                = 1 个核（1000m）
  500m             = 0.5 核
  100m             = 0.1 核（最小推荐）
  
内存单位：
  1Gi              = 1024^3 = 1073741824 字节（推荐）
  1G               = 1000^3 = 1000000000 字节（容易和 Gi 混淆）
  
始终用 Mi/Gi（二进制），不用 M/G（十进制）。
```

### 1.4 验证 cgroup 设置

```bash
# 进入 pod，看 cgroup 限制（cgroup v2）
kubectl exec -it api-server -- sh
cat /sys/fs/cgroup/cpu.max
# 输出: 200000 100000  ← limit 2 核（200ms / 100ms）

cat /sys/fs/cgroup/memory.max
# 输出: 1073741824  ← limit 1Gi

cat /sys/fs/cgroup/memory.current
# 输出: 234819584  ← 当前使用
```

```bash
# 节点视角看一个容器
crictl inspect <containerid> | jq '.info.runtimeSpec.linux.resources'
```

### 1.5 不设 limits 行不行

**不设 limits 的常见做法**（业界争议大）：

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  # 不设 limits
```

- 优点：CPU 不会被 throttle（IO bound 服务受益），内存可以临时 burst
- 缺点：BestEffort/Burstable 节点资源争抢；单个 Pod 可能吃光节点把其他人挤掉

**Google SRE 倾向不设 CPU limit**（让 CPU 被节点共享），**仍设 memory limit**（防 OOM 影响整个节点）。

**国内大厂多数做法**：CPU 设 limit（防止某个业务突发把整个机器卡住），memory 设 limit（防 OOM 杀其他容器）。

**推荐**：

- 离线 / 批处理：CPU 不设 limit，memory 一定设
- 在线核心服务：两个都设，limit 是 request 的 2-4 倍
- 内存敏感（DB / 缓存）：memory request = limit（强制 Guaranteed QoS）

---

## 第二章：QoS 等级——驱逐时的生死排队

### 2.1 三个等级

Kubernetes 根据 Pod 所有容器的 requests / limits 自动算 QoS：

```
Guaranteed（最高）:
  所有容器 cpu/memory 都设了 requests 与 limits
  且 requests == limits

Burstable（中间）:
  至少有一个容器设了 requests 或 limits
  但不满足 Guaranteed

BestEffort（最低）:
  所有容器都没设 requests 与 limits
```

查看：

```bash
kubectl get pod api-server -o jsonpath='{.status.qosClass}'
# Burstable
```

### 2.2 驱逐顺序

节点资源紧张（CPU / 内存 / 磁盘）时 kubelet 启动 **node-pressure eviction**：

```
驱逐优先级（先驱逐谁）:
  1. BestEffort（直接干掉）
  2. Burstable（超过 requests 的容器优先被杀）
  3. Guaranteed（万不得已才动）
```

**Memory pressure 触发**：

```
kubelet 默认 eviction-hard:
  memory.available < 100Mi
  nodefs.available < 10%
  nodefs.inodesFree < 5%
  imagefs.available < 15%
```

触发后：

1. 算每个 Pod 的 OOM score（QoS + 超过 requests 程度）
2. 优先驱逐 BestEffort，再 Burstable（超得多的先死），最后 Guaranteed

### 2.3 系统级 OOM（内核 OOM Killer）

如果驱逐没赶上，内核会触发系统级 OOM Killer：

```
oom_score_adj：
  Guaranteed:     -997（基本不被杀）
  Burstable:       1000 - (1000*requests / node memory)  ←越接近 1000 越容易被杀
  BestEffort:      1000（最先被杀）
```

```bash
# 看进程 oom_score
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj
```

### 2.4 为什么核心服务一定要 Guaranteed

```yaml
# 错误：核心数据库设成 Burstable，节点 OOM 时被杀
resources:
  requests: { memory: "4Gi" }
  limits: { memory: "8Gi" }

# 正确：Guaranteed
resources:
  requests: { memory: "8Gi", cpu: "4" }
  limits:   { memory: "8Gi", cpu: "4" }
```

Guaranteed QoS 还有一个隐藏好处：**cpu manager 静态分配独占核心**（详见 4.6）。

### 2.5 Pod-level resources（K8s 1.32 alpha，1.34 beta）

历史上 resources 只能在 container 层设。1.32 引入 **Pod-level resources**（feature gate `PodLevelResources`）：

```yaml
apiVersion: v1
kind: Pod
spec:
  resources:                # Pod 整体的 requests/limits
    requests: { cpu: "2", memory: "4Gi" }
    limits:   { cpu: "4", memory: "8Gi" }
  containers:
  - name: app
    # 不再单独设 resources，从 pod 池里共享
```

收益：

- 多容器 sidecar（如 Istio Ambient waypoint）共享资源池，避免每个 sidecar 单独估算
- 同 pod 下"忙容器"可以借"闲容器"的额度

2026-05 状态：1.32 alpha（默认关闭），**1.34 graduated to beta（默认开启）**；部分托管 K8s（GKE）已默认开启；自建集群建议 1.34 之后再启用。

---

## 第三章：调度器决策链——Filter 与 Score

### 3.1 调度阶段

```
Pod 创建 → API Server → etcd → kube-scheduler watch
                                     │
                                     ▼
                          ┌──────────────────────┐
                          │  1. Filter（预选）    │  剔除不能放的节点
                          │  2. Score（优选）     │  对剩余节点打分
                          │  3. Reserve           │  预占资源
                          │  4. Permit            │  允许/等待
                          │  5. PreBind / Bind    │  写回 nodeName
                          └──────────────────────┘
                                     │
                                     ▼
                              kubelet 收到通知 → 起容器
```

### 3.2 Filter 阶段——必须满足

按顺序检查每个节点：

1. **PodFitsResources**：节点 Allocatable - sum(requests) ≥ Pod requests
2. **PodFitsHostPorts**：hostPort 不冲突
3. **MatchNodeSelector / NodeAffinity**：标签匹配
4. **TaintToleration**：能容忍节点的 NoSchedule / NoExecute taint
5. **VolumeBinding / VolumeZone**：PVC 拓扑约束（如 EBS 必须同 AZ）
6. **PodTopologySpread**：拓扑分布约束（DoNotSchedule 等级）
7. **NodeUnschedulable**：节点没被 cordon
8. **PodAffinity / AntiAffinity**：与已有 Pod 的亲和反亲和

**不满足任何一个 → 该节点被剔除**。如果所有节点都被剔除：

```
Events:
  Warning  FailedScheduling  default-scheduler  
  0/12 nodes are available: 6 Insufficient memory, 4 node(s) had untolerated taint, 
  2 node(s) didn't match Pod's node affinity.
```

这个 message 是排查调度失败的金矿——一定要 `kubectl describe pod` 看。

### 3.3 Score 阶段——选最优

在 Filter 通过的节点里打分（0-100）：

| Plugin | 评分逻辑 |
|---|---|
| `NodeResourcesFit` | 剩余资源最多/最少（MostAllocated / LeastAllocated）|
| `InterPodAffinity` | Pod 亲和分数（喜欢/不喜欢哪些 Pod）|
| `PodTopologySpread` | 拓扑分散程度 |
| `NodeAffinity` | preferredDuringScheduling 偏好分 |
| `ImageLocality` | 节点上已有 image 加分（启动快）|
| `TaintToleration` | PreferNoSchedule taint 扣分 |

scoring strategy：

- **LeastAllocated**（默认）：把 Pod 放到**最空闲**的节点（负载均衡）
- **MostAllocated**：把 Pod 放到**最满**的节点（紧凑，方便缩容）
- **RequestedToCapacityRatio**：按目标利用率分配

2026 年多数集群默认 LeastAllocated，但**节点级伸缩（Karpenter）配 MostAllocated 更合适**——容易腾空节点缩容省钱。

### 3.4 Score 自定义示例

通过 KubeSchedulerConfiguration 配置：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: RequestedToCapacityRatio
        requestedToCapacityRatio:
          shape:
          - utilization: 0
            score: 0
          - utilization: 100
            score: 10
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

### 3.5 多调度器

K8s 支持多个 scheduler 并存：

```yaml
spec:
  schedulerName: my-custom-scheduler   # 默认是 default-scheduler
```

常见场景：

- GPU 调度（NVIDIA 的 `volcano` 或 `kueue`，处理 gang scheduling）
- 大数据（Volcano / YuniKorn 做 job queue）
- 自研调度（金融行业的 SLA 调度）

---

## 第四章：nodeSelector / nodeAffinity / podAffinity 与 Topology Spread

### 4.1 nodeSelector（最简单）

```yaml
spec:
  nodeSelector:
    disktype: ssd
    zone: us-east-1a
```

要求节点同时满足这些 label。**全或无**——不满足直接调度失败，没有"prefer"概念。

### 4.2 nodeAffinity（更强大）

两类：

```yaml
spec:
  affinity:
    nodeAffinity:
      # 硬约束（相当于增强版 nodeSelector）
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: ["amd64", "arm64"]
          - key: node.kubernetes.io/instance-type
            operator: NotIn
            values: ["t3.micro"]
      
      # 软约束（打分）
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a"]
```

**operators**：`In` / `NotIn` / `Exists` / `DoesNotExist` / `Gt` / `Lt`（Gt/Lt 数字比较）。

**关键陷阱**：`IgnoredDuringExecution` 意味着 Pod 已经在节点上跑了之后，**label 变了不会被踢走**。要等 Pod 重启才重调度。1.33 仍未实现 `RequiredDuringExecution`（一直在讨论但因为复杂度高没落地）。

### 4.3 podAffinity / podAntiAffinity（与其他 Pod 的关系）

```yaml
spec:
  affinity:
    # 想和 cache pod 同节点（数据局部性）
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: ["redis-cache"]
          topologyKey: kubernetes.io/hostname
    
    # 不想和同 app 的其他副本在同节点（高可用）
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["my-app"]
        topologyKey: kubernetes.io/hostname
```

**topologyKey 的意义**：拓扑域定义。

- `kubernetes.io/hostname` → 单节点
- `topology.kubernetes.io/zone` → 单 AZ
- `topology.kubernetes.io/region` → 单 region

**性能警告**：podAntiAffinity required 在大集群（> 200 节点）非常慢——scheduler 每次都要扫所有节点上所有匹配 Pod。**推荐用 TopologySpreadConstraints 替代**。

### 4.4 TopologySpreadConstraints（推荐）

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule         # 或 ScheduleAnyway
    labelSelector:
      matchLabels:
        app: my-app
    minDomains: 3                            # 至少 3 个 AZ 才生效（防止单 AZ 集群）
    matchLabelKeys:
    - pod-template-hash                      # 仅同一 ReplicaSet 内分散（避免新老版本互相影响）
```

含义：在所有 AZ 之间，**同 label Pod 数量差不超过 1**。

```
zone-a: 3 Pod    zone-b: 2 Pod    zone-c: 2 Pod   ← skew = 3-2 = 1 ✅
zone-a: 4 Pod    zone-b: 2 Pod    zone-c: 1 Pod   ← skew = 4-1 = 3 ❌（被拒绝）
```

**maxSkew=1 + DoNotSchedule** 是高可用业务的标配。

可以多个约束叠加：

```yaml
topologySpreadConstraints:
- maxSkew: 1                                  # zone 之间均衡
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector: {matchLabels: {app: web}}
- maxSkew: 2                                  # node 之间偏好均衡
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector: {matchLabels: {app: web}}
```

### 4.5 Deployment / DefaultPodTopologySpread

K8s 1.24+ 集群级默认拓扑分散（关闭可在 KubeSchedulerConfiguration 里）：

```yaml
profiles:
- pluginConfig:
  - name: PodTopologySpread
    args:
      defaultConstraints:
      - maxSkew: 3
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
      defaultingType: System
```

意思：如果业务 Pod 没显式设 topology spread，默认按这个分散。**这是 2026 年大规模集群必开**——避免新业务忘记设 antiaffinity 全堆一个 AZ。

### 4.6 CPU Manager Policy

```yaml
# kubelet 配置 /var/lib/kubelet/config.yaml
cpuManagerPolicy: static     # 默认 none
reservedSystemCPUs: "0,1"    # 系统保留 CPU
```

**static 策略**：Guaranteed QoS + 整数核 requests 的容器**独占核心**（不被其他容器分时复用）：

```yaml
resources:
  requests: { cpu: "4", memory: "8Gi" }
  limits:   { cpu: "4", memory: "8Gi" }
```

→ 拿到 4 个独占 CPU 核心，不被 throttle，缓存命中率高。延迟敏感业务（数据库、实时计算）首选。

代价：容器死了之后那 4 个核心要等 kubelet 释放才能给其他容器用。

---

## 第五章：Taints 与 Tolerations——把节点专用化

### 5.1 概念

```
Taint：给节点打"污点"——告诉 scheduler "默认不要往这里放 Pod"
Toleration：给 Pod 写"容忍"——告诉 scheduler "我能忍受某种污点"
```

```bash
# 给节点打 taint
kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule

# 给 Pod 写 toleration
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

### 5.2 三种 effect

| effect | 含义 |
|---|---|
| `NoSchedule` | 不调度新 Pod；已运行的 Pod 不动 |
| `PreferNoSchedule` | 尽量不调度（但找不到别的节点时还是会放）|
| `NoExecute` | 不调度 + 把已运行的不容忍 Pod **驱逐** |

```yaml
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300         # 节点 NotReady 5 分钟内不被驱逐
```

### 5.3 系统自动 taint

K8s 控制平面自动打的 taint：

```
node.kubernetes.io/not-ready          NoExecute   节点 NotReady
node.kubernetes.io/unreachable        NoExecute   节点失联
node.kubernetes.io/memory-pressure    NoSchedule  内存压力
node.kubernetes.io/disk-pressure      NoSchedule  磁盘压力
node.kubernetes.io/network-unavailable NoSchedule 网络不可用
node.kubernetes.io/unschedulable      NoSchedule  cordon 后
node.cloudprovider.kubernetes.io/uninitialized NoSchedule 云厂商初始化中
```

所有 Pod 默认对 `not-ready` 和 `unreachable` 容忍 300 秒。

### 5.4 典型场景

**GPU 节点专用**：

```bash
kubectl taint nodes gpu-node-1 nvidia.com/gpu=:NoSchedule
```

普通 Pod 不会被调到 GPU 节点（哪怕节点空闲），只有显式写 toleration 的 GPU 任务才能用。

**Spot / Preemptible 节点**：

```bash
kubectl taint nodes spot-node-1 spot=true:PreferNoSchedule
```

无状态业务可写 toleration 享受低价，有状态业务不写就避开。

**Karpenter 自动节点的 startup taint**：

Karpenter 启动新节点时打 `karpenter.sh/disrupted=NoSchedule`（或 `node.cilium.io/agent-not-ready`），等 daemonset（如 CNI、CSI）就绪再去掉——避免 Pod 落到没初始化好的节点上。

---

## 第六章：PriorityClass 与抢占

### 6.1 PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical online services"
preemptionPolicy: PreemptLowerPriority      # 或 Never
```

```yaml
spec:
  priorityClassName: high-priority
```

K8s 内置：

- `system-cluster-critical`（2,000,000,000）—— kube-system 关键组件
- `system-node-critical`（2,000,001,000）—— kubelet、CNI、CSI 等

业务自定义建议：

```
business-critical    1,000,000       核心业务（支付、登录）
high-priority        500,000          重要业务
normal               0                普通业务（默认）
low-priority         -100             离线、批处理
```

### 6.2 抢占（Preemption）

当高优 Pod 调度不下时，scheduler 会找：

1. 哪些节点上有比我低优的 Pod
2. 驱逐它们能不能腾出空间放我
3. 选"代价最小"的方案（最少 Pod、最低优先级）
4. 驱逐被选中的低优 Pod（给 graceful shutdown 时间）

被驱逐的 Pod 会进 pending 队列，等高优 Pod 起来后再找资源。

### 6.3 抢占的副作用

```
high-priority Pod 来了
   ↓
scheduler 找节点 → 选节点 X
   ↓
驱逐节点 X 上的 normal-priority Pod A、B、C  ← 触发 SIGTERM
   ↓
A、B、C 被杀，high-priority 起来
   ↓
A、B、C 进 pending，等 cluster autoscaler 起新节点
```

**陷阱**：

- 如果 A、B、C 是有状态 Pod（StatefulSet），抢占会**强制结束业务**
- 抢占链可能级联：高优逼走中优，中优逼走低优……雪崩
- **生产建议**：核心业务用 `business-critical`，但严格控制总量（< 集群容量 50%），并设 `preemptionPolicy: Never` 给"贵但可等"的批任务

### 6.4 Non-preempting 模式

```yaml
preemptionPolicy: Never
```

意思：我比别人优先排队（pending 队列里我先调），但**不抢占已运行 Pod**。适合：

- ML 训练任务（贵，但可以等）
- 数据迁移 Job（怕中断）

---

## 第七章：调度器扩展——Scheduler Framework

### 7.1 Scheduler Framework（K8s 1.19 GA）

整个 scheduler 被拆成一系列**扩展点**（plugin），可以插自定义代码：

```
QueueSort       → 排序待调度 Pod
PreFilter       → 预检查（如算资源汇总）
Filter          → 过滤节点
PostFilter      → Filter 全失败时（如尝试抢占）
PreScore        → Score 前准备
Score           → 节点打分
NormalizeScore  → 分数标准化
Reserve         → 预占资源
Permit          → 决定是否真的 bind（可延迟，如 gang scheduling）
PreBind         → Bind 前操作（如挂载 PV）
Bind            → 写回 nodeName
PostBind        → bind 后回调
```

### 7.2 自定义 plugin（Go）

```go
package myplugin

import (
    "context"
    "k8s.io/api/core/v1"
    "k8s.io/kubernetes/pkg/scheduler/framework"
)

const Name = "MyPlugin"

type MyPlugin struct{ handle framework.Handle }

func (p *MyPlugin) Name() string { return Name }

// 实现 Filter 扩展点
func (p *MyPlugin) Filter(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeInfo *framework.NodeInfo,
) *framework.Status {
    // 自定义逻辑：例如根据 Pod label 强制要求节点有特定 GPU 型号
    requiredGPU := pod.Labels["gpu-model"]
    if requiredGPU == "" {
        return framework.NewStatus(framework.Success)
    }
    if nodeInfo.Node().Labels["gpu-model"] != requiredGPU {
        return framework.NewStatus(framework.Unschedulable, "GPU model mismatch")
    }
    return framework.NewStatus(framework.Success)
}

// 实现 Score 扩展点
func (p *MyPlugin) Score(
    ctx context.Context,
    state *framework.CycleState,
    pod *v1.Pod,
    nodeName string,
) (int64, *framework.Status) {
    nodeInfo, err := p.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
    if err != nil { return 0, framework.NewStatus(framework.Error, err.Error()) }
    // 例：节点上已运行同 app pod 数越少分越高
    sameAppCount := 0
    for _, p2 := range nodeInfo.Pods {
        if p2.Pod.Labels["app"] == pod.Labels["app"] {
            sameAppCount++
        }
    }
    return int64(100 - sameAppCount*20), framework.NewStatus(framework.Success)
}

// 工厂函数
func New(_ context.Context, _ runtime.Object, h framework.Handle) (framework.Plugin, error) {
    return &MyPlugin{handle: h}, nil
}
```

注册到调度器：

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler
  plugins:
    filter:
      enabled:
      - name: MyPlugin
    score:
      enabled:
      - name: MyPlugin
        weight: 5
```

### 7.3 流行的第三方调度器

| 项目 | 场景 |
|---|---|
| **Volcano** | 大数据 / AI 训练 / gang scheduling（一组 Pod 必须同时调度成功）|
| **Kueue** | K8s 官方"作业排队"组件，配合 batch workload |
| **YuniKorn**（Apache） | 大数据队列调度（hierarchical queue）|
| **Karmada Scheduler** | 多集群调度 |
| **Yunikorn / Crane** | 节省成本的调度（按 utilization 选节点）|

如果你的业务**不是"一个 Pod 一个 Pod"调度**——比如训练要 8 个 GPU Pod 一起起，要么 8 个都起，要么一个都不起——必须用 Volcano 或 Kueue 的 gang scheduling。

---

## 第八章：HPA——按指标横向扩展

### 8.1 HPA v2（2026 主流）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70           # 目标 CPU 利用率 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0      # 立即扩容
      policies:
      - type: Percent
        value: 100                        # 每分钟最多扩 100%
        periodSeconds: 60
      - type: Pods
        value: 4                          # 或每分钟最多扩 4 个
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300    # 5 分钟稳定期再缩
      policies:
      - type: Percent
        value: 10                         # 每分钟最多缩 10%
        periodSeconds: 60
```

### 8.2 工作原理

```
HPA controller 每 15s 算一次:
  for each metric:
    desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))
  
  desired = max(各 metric 算出的值)
  desired = clamp(desired, minReplicas, maxReplicas)
  
  if desired != current:
    Deployment.spec.replicas = desired
```

CPU 当前 90%、目标 70% → desired = current × 90 / 70 = 1.28x 副本。

### 8.3 metric source

```
1. Resource（CPU/memory）         需要 metrics-server
2. Pods（per-Pod custom metric）   需要 custom.metrics.k8s.io API
3. Object（其他对象的 metric）      如 Ingress qps
4. External（外部系统 metric）      如 SQS queue length
5. ContainerResource              container 级 CPU/memory（多容器 Pod 有用）
```

**metrics-server** 几乎必装：

```bash
helm install metrics-server metrics-server/metrics-server \
  -n kube-system \
  --set args="{--kubelet-insecure-tls}"
```

**custom metrics** 走 Prometheus Adapter 或 KEDA External Scaler。

### 8.4 自定义指标（Prometheus Adapter）

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "1000"          # 每 Pod 期望 1000 qps
```

需要 Prometheus Adapter 把 Prometheus 的 metric 暴露给 K8s API：

```yaml
# Adapter rule
- seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    matches: "^(.*)_total"
    as: "${1}_per_second"
  metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

### 8.5 behavior 详解（重点）

K8s 1.18 引入 `behavior`，**这是 HPA 调优的核心**：

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60
    selectPolicy: Min                   # 多 policy 取最保守
```

`stabilizationWindowSeconds`：观察这个窗口内的最大期望副本数，**只缩到这个最大值**——防止抖动。默认 scaleDown 300s，scaleUp 0s。

`policies`：限制每个周期的伸缩幅度：

```yaml
- type: Percent, value: 10, periodSeconds: 60   # 每分钟最多缩 10%
- type: Pods,    value: 5,  periodSeconds: 60   # 或最多缩 5 个 Pod
```

`selectPolicy`：

- `Max`（默认 scaleUp）：取最激进的——扩容快
- `Min`（默认 scaleDown）：取最保守的——缩容慢
- `Disabled`：禁用该方向

**常见调优**：

```yaml
# 流量突发型（电商）：扩快缩慢
scaleUp:
  stabilizationWindowSeconds: 0
  policies:
  - type: Percent, value: 200, periodSeconds: 30   # 30s 翻 3 倍
scaleDown:
  stabilizationWindowSeconds: 600                   # 10 分钟稳定期
  policies:
  - type: Pods, value: 1, periodSeconds: 120        # 每 2 分钟缩 1 个

# 批处理型：扩慢缩快
scaleUp:
  stabilizationWindowSeconds: 180
  policies:
  - type: Pods, value: 1, periodSeconds: 60
scaleDown:
  stabilizationWindowSeconds: 60
  policies:
  - type: Percent, value: 50, periodSeconds: 30
```

### 8.6 排查 HPA

```bash
kubectl describe hpa api-hpa
```

```
Conditions:
  Type            Status  Reason            Message
  AbleToScale     True    ReadyForNewScale  
  ScalingActive   True    ValidMetricFound  
  ScalingLimited  False   DesiredWithinRange

Metrics:
  ( current / target )
  resource cpu on pods    78% / 70%
  resource memory on pods 60% / 80%

Events:
  Normal  SuccessfulRescale  2m  horizontal-pod-autoscaler  
  New size: 5; reason: cpu resource utilization above target
```

如果 `ScalingActive=False reason=FailedGetResourceMetric`，多半是 metrics-server 没装好或没权限。

---

## 第九章：VPA——按历史用量调 Requests

### 9.1 三种模式

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Auto"            # Off | Initial | Auto | InPlace（1.33 alpha）
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

| 模式 | 行为 |
|---|---|
| `Off` | 只算推荐值（写到 status），不改 Pod——观察用 |
| `Initial` | 只在 Pod 新建时设 requests，运行中不动 |
| `Auto` | 推荐值变化大时**重启 Pod** 应用新 requests（侵入式）|
| `InPlace`（1.33 alpha）| **不重启**直接改 cgroup |

### 9.2 工作原理

```
VPA Recommender:
  - watch Prometheus / metrics-server
  - 算每个容器 CPU/Memory 的 P50/P95/P99
  - 输出推荐值（target / lowerBound / upperBound）

VPA Updater（Auto 模式）:
  - 如果当前 requests 偏离推荐值太多
  - 驱逐 Pod（按 PDB 限制速率）
  - VPA Admission Webhook 在新 Pod 创建时改写 requests
```

### 9.3 VPA 与 HPA 同用

**经典陷阱**：HPA 看 CPU 利用率扩容（横向），VPA 调 CPU requests（纵向）。两者都改 CPU 维度 → 死循环：

```
VPA 把 requests 调高 → CPU 利用率（actual / requests）下降 → HPA 看到低利用率 → 缩容
缩容后单 Pod 负载升高 → VPA 又把 requests 调高 → ……
```

**正确做法**：

1. **HPA 用 CPU、VPA 用内存**——不重叠
2. 或 HPA 用 custom metric（qps）、VPA 调 CPU+内存——HPA 不看 CPU 就不冲突
3. 或 VPA `updateMode: Off`，只观察推荐值，人工调整

### 9.4 In-place Pod Vertical Scaling（K8s 1.33 Beta / 1.35 GA）

历史：改 requests/limits **必须重启 Pod**（因为 cgroup 在 Pod 创建时设好，运行时不能改）。

K8s 1.27 alpha → 1.33 beta → 1.35 GA（feature gate `InPlacePodVerticalScaling`）：直接 patch 容器 resources，kubelet 调用 CRI ResizeContainer 改 cgroup，**无需重启**。

```bash
# 1.33+ 集群
kubectl patch pod api-server --subresource resize --patch '{
  "spec": {
    "containers": [{"name": "app", "resources": {"requests": {"cpu": "1"}, "limits": {"cpu": "2"}}}]
  }
}'
```

```yaml
# Container 层指定 resize policy
containers:
- name: app
  resizePolicy:
  - resourceName: cpu
    restartPolicy: NotRequired       # NotRequired 或 RestartContainer
  - resourceName: memory
    restartPolicy: RestartContainer  # 内存缩小可能需要重启
```

**2026-05 状态**：

- 1.33 beta（默认开启）；1.35 GA（2025-12）
- VPA 1.4+ 支持 InPlace 模式（实验）
- 仍有限制：缩 memory（要释放页）不支持完全 in-place；CPU/memory 增大基本无重启
- 生产建议：1.35 GA 后核心业务可逐步采用，最佳实践仍在沉淀

### 9.5 VPA 的局限

- **Auto 模式重启 Pod 影响连接**——长连接业务（WebSocket、gRPC stream）慎用
- 推荐值算法是 P95 + safety margin（默认 15%）——突发负载时可能不够
- **不能与 HPA 同改 CPU**（前述陷阱）
- 历史数据量小时推荐值不稳定（建议至少跑 1 周）

---

## 第十章：KEDA——事件驱动伸缩

### 10.1 为什么需要 KEDA

HPA 的 metric 维度有限：

- **CPU / 内存** 不能反映"队列堆积"
- **custom metric** 要自己搭 Prometheus + Adapter，工程量大
- **从 0 扩容**——HPA 不能从 0 起步（minReplicas ≥ 1，因为 metric 拿不到）

KEDA（CNCF Graduated 2023）= **Kubernetes Event-Driven Autoscaling**：

```
KEDA 监听外部事件源 → 转成 metric → 喂给 HPA（KEDA 内部生成 HPA）
                              ↓
                      Deployment.replicas 变化
```

支持 60+ scaler：Kafka、RabbitMQ、AWS SQS / Kinesis、Azure Service Bus、GCP Pub/Sub、Redis Streams、Prometheus、PostgreSQL、Cron、AWS CloudWatch、Datadog……

### 10.2 安装

```bash
helm install keda kedacore/keda --namespace keda-system --create-namespace
```

### 10.3 ScaledObject——Deployment 伸缩

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kafka-consumer
  minReplicaCount: 0                  # 关键：可以缩到 0
  maxReplicaCount: 100
  pollingInterval: 30                  # 30s 查一次 metric
  cooldownPeriod: 300                  # 缩到 0 前等 5 分钟
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-broker:9092
      consumerGroup: order-consumers
      topic: orders
      lagThreshold: "100"              # 单分区滞后 100 条触发扩容
      offsetResetPolicy: latest
      partitionLimitation: "0,1,2,3"   # 只看这几个分区
```

KEDA 内部：

1. 创建一个 HPA，target deployment 是 kafka-consumer
2. metric source 是 KEDA External Metric API（KEDA Operator 自己实现）
3. KEDA Operator 定期问 Kafka 算 lag → 写到 metric → HPA 看到 → 扩缩

### 10.4 ScaledJob——批任务伸缩

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: worker
          image: img-processor:1.0
        restartPolicy: Never
  maxReplicaCount: 50
  pollingInterval: 30
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123/jobs
      queueLength: "5"                 # 每 5 条消息起一个 Job
      awsRegion: us-east-1
```

ScaledJob 适合"一条消息处理一段时间然后结束"的任务（Job 而非 Deployment）。

### 10.5 多 trigger（AND / OR）

```yaml
triggers:
- type: prometheus
  metadata:
    serverAddress: http://prometheus:9090
    metricName: http_requests
    threshold: "1000"
    query: sum(rate(http_requests_total[2m]))
- type: cron
  metadata:
    timezone: Asia/Shanghai
    start: "0 9 * * *"                 # 早 9 点扩
    end: "0 22 * * *"                  # 晚 10 点缩
    desiredReplicas: "10"
```

**多 trigger 取最大值**——任何一个 trigger 想扩，就扩。

### 10.6 TriggerAuthentication——访问外部系统

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-trigger-auth
spec:
  secretTargetRef:
  - parameter: sasl
    name: kafka-secret
    key: sasl
  - parameter: username
    name: kafka-secret
    key: username
```

```yaml
triggers:
- type: kafka
  authenticationRef:
    name: kafka-trigger-auth
```

也支持 Workload Identity、Azure Pod Identity 等云原生方式，避免把 secret 硬塞到 ScaledObject。

### 10.7 Go 写自定义 External Scaler

如果 KEDA 内置 scaler 不够，可以写 gRPC scaler：

```go
package main

import (
    "context"
    "log"
    "net"
    "google.golang.org/grpc"
    pb "github.com/kedacore/keda/v2/pkg/scalers/externalscaler"
)

type myScaler struct{ pb.UnimplementedExternalScalerServer }

func (s *myScaler) IsActive(ctx context.Context, req *pb.ScaledObjectRef) (*pb.IsActiveResponse, error) {
    // 业务自己定义"是否有事件"
    active := checkMyQueue(req.ScalerMetadata["queueName"])
    return &pb.IsActiveResponse{Result: active}, nil
}

func (s *myScaler) GetMetricSpec(ctx context.Context, req *pb.ScaledObjectRef) (*pb.GetMetricSpecResponse, error) {
    return &pb.GetMetricSpecResponse{
        MetricSpecs: []*pb.MetricSpec{{
            MetricName: "myqueue_length",
            TargetSize: 10,
        }},
    }, nil
}

func (s *myScaler) GetMetrics(ctx context.Context, req *pb.GetMetricsRequest) (*pb.GetMetricsResponse, error) {
    length := queryMyQueue(req.ScaledObjectRef.ScalerMetadata["queueName"])
    return &pb.GetMetricsResponse{
        MetricValues: []*pb.MetricValue{{
            MetricName:  "myqueue_length",
            MetricValue: int64(length),
        }},
    }, nil
}

func main() {
    lis, _ := net.Listen("tcp", ":6000")
    s := grpc.NewServer()
    pb.RegisterExternalScalerServer(s, &myScaler{})
    log.Println("scaler listening on :6000")
    s.Serve(lis)
}
```

```yaml
triggers:
- type: external
  metadata:
    scalerAddress: my-scaler.default.svc:6000
    queueName: my-queue
```

### 10.8 KEDA + HPA 同用

可以！KEDA 会生成 HPA，**但你自己手写的 HPA 不能再管 KEDA 管的 Deployment**——会冲突。如果想保留自定义 HPA behavior，在 ScaledObject 里加：

```yaml
spec:
  advanced:
    horizontalPodAutoscalerConfig:
      name: my-keda-hpa
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 600
          policies:
          - type: Percent
            value: 50
            periodSeconds: 60
```

---

## 第十一章：Karpenter 与 Cluster Autoscaler——节点级伸缩

### 11.1 为什么需要节点级伸缩

Pod 横向扩到 minReplicas → maxReplicas，但如果**节点不够**，新 Pod 永远 Pending：

```
Events:
  Warning  FailedScheduling  default-scheduler  
  0/12 nodes are available: 12 Insufficient cpu.
```

→ 必须有组件去**新增节点**（startup 1-5 分钟）。

### 11.2 Cluster Autoscaler（经典）

工作原理：

```
1. CA watch Pending Pod
2. 模拟"如果加一个节点能不能放下"
3. 调用 cloud provider API 起新节点（AWS ASG / GCP MIG）
4. 节点 Ready → kubelet 注册 → scheduler 重新调度 Pending Pod
5. 反向：如果节点利用率 < 50% 持续 10 分钟，标记缩容
```

**特点**：

- 基于 **node group / ASG**——需要预先定义机型组合
- 同一组内节点机型一致——不灵活
- 起节点慢（依赖云厂商 ASG）
- 缩容保守（怕影响 Pod）

### 11.3 Karpenter（2026 主流）

AWS 2021 开源，2023 捐给 CNCF（Autoscaling SIG），2024 发布 v1.0，2025-2026 成为主流。Azure、GCP 都有官方支持。

**关键差异**：

- **不需要 node group**——Karpenter 直接调 EC2 API（或 Azure / GCP），按需选机型
- **bin-packing 优化**——根据 Pending Pod 的需求精确选机型（不会因为一个 16 核 Pod 起一台 64 核机器）
- **快**——从 Pending 到节点 Ready 通常 < 60s（vs CA 2-5 分钟）
- **节省成本**——容易腾空节点缩容，spot 支持好

### 11.4 NodePool（Karpenter v1）

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    metadata:
      labels:
        nodepool: default
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand", "spot"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m6i.large", "m6i.xlarge", "m6i.2xlarge", "c6i.large", "c6i.xlarge"]
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-east-1a", "us-east-1b", "us-east-1c"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      taints:
      - key: spot-instance
        value: "true"
        effect: NoSchedule
      startupTaints:
      - key: node.cilium.io/agent-not-ready
        value: "true"
        effect: NoSchedule
  limits:
    cpu: 1000
    memory: 4000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
    budgets:
    - nodes: "10%"                        # 同时最多扰动 10% 节点
```

### 11.5 EC2NodeClass

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  amiSelectorTerms:
  - alias: al2023@latest
  subnetSelectorTerms:
  - tags: { karpenter.sh/discovery: my-cluster }
  securityGroupSelectorTerms:
  - tags: { karpenter.sh/discovery: my-cluster }
  role: KarpenterNodeRole-my-cluster
  blockDeviceMappings:
  - deviceName: /dev/xvda
    ebs:
      volumeSize: 100Gi
      volumeType: gp3
      iops: 3000
      throughput: 125
      encrypted: true
  userData: |
    #!/bin/bash
    echo "custom bootstrap"
  tags:
    Team: platform
```

### 11.6 Karpenter 决策流程

```
Pending Pod 出现
    │
    ▼
模拟 bin-pack：
  - 把 Pending Pod 加上"还没分配的容量"算 requests
  - 选最小、最便宜、最快的机型
  - 优先 spot 然后 on-demand（按 NodePool 配置）
    │
    ▼
调 EC2 API 起实例
    │
    ▼
节点 Ready → kubelet 注册 → scheduler 调度 Pod
```

**Consolidation**（合并）：

- WhenEmpty：节点空了就删（保守）
- WhenEmptyOrUnderutilized：发现某节点的 Pod 可以塞到其他节点 → 标记删除（激进，省钱）

Karpenter 会主动**移动**Pod：把多台低利用率节点上的 Pod 集中到一台，然后删空节点。这是和 CA 最大区别。

### 11.7 Disruption Budget

```yaml
disruption:
  budgets:
  - nodes: "20%"
    schedule: "@daily"
    duration: 10m
  - nodes: "0"                            # 工作时间不扰动
    schedule: "0 9 * * *"
    duration: 8h
```

避免在业务高峰期间合并节点导致 Pod 重启风暴。

### 11.8 Spot 实战

```yaml
requirements:
- key: karpenter.sh/capacity-type
  operator: In
  values: ["spot", "on-demand"]            # spot 优先（按价格选）
- key: karpenter.k8s.aws/instance-family
  operator: In
  values: ["m6i", "m6a", "c6i", "c6a"]
- key: karpenter.k8s.aws/instance-cpu
  operator: In
  values: ["2", "4", "8", "16"]
```

Karpenter 看到价格表，自动选最便宜的 spot 实例。spot 中断时（2 分钟通知）Karpenter 立即起新节点，graceful drain 旧节点。

**建议**：

- 无状态业务 → spot
- 关键有状态 → on-demand
- 用 PriorityClass 区分

---

## 第十二章：In-place Pod Vertical Scaling 2026 状态

### 12.1 历史

```
1.27 alpha (2023-04)：feature gate InPlacePodVerticalScaling
1.30+：稳定中
1.33 (2025-04)：beta，默认开启
1.35 (2025-12)：GA
```

### 12.2 工作机制

```
kubectl patch pod ... --subresource resize
        ↓
API Server 校验 resize 请求
        ↓
Pod.spec.containers[].resources 改变
Pod.status.containerStatuses[].resources 显示当前实际值
        ↓
kubelet watch 到变化
        ↓
调用 CRI runtime ContainerUpdate
        ↓
runc/crun 改 cgroup（cpu.max / memory.max）
        ↓
status 更新为新值
```

### 12.3 限制

- **CPU 增大** → 通常无需重启
- **CPU 减小** → 大多无需重启（cgroup 直接生效）
- **Memory 增大** → 无需重启
- **Memory 减小** → kernel 不一定能立即回收 → 可能要等或重启
- **Resize 失败时** → status 写 ResizePending，kubelet 不重试（业务侧要看 status）

### 12.4 与 VPA InPlace 模式

```yaml
updatePolicy:
  updateMode: "InPlace"                  # 1.33 alpha 的 VPA
```

VPA 推荐值变化时不再驱逐 Pod，而是直接 resize。

**潜在收益**：

- 长连接业务（WebSocket、gRPC stream）不丢连接
- 数据库 / 缓存（启动慢的）秒级 resize
- 节省 Pod 重启的开销和短暂不可用

### 12.5 仍未解决

- HPA 与 in-place VPA 共用 CPU 仍冲突（VPA 调 requests 后 HPA 利用率公式变化）
- 仅 ResizePolicy NotRequired 才"真在线"——RestartContainer 仍重启
- 集群级 binpacking 需要 scheduler 重新评估（in-place 增大可能导致节点超卖）

**2026-05 建议**：1.33 起默认开启 beta，1.35 已 GA，非长连接业务可以采用；关键长连接业务仍优先 HPA + 容量留足。

---

## 第十三章：生产实践

### 13.1 合理估算 requests / limits

**步骤**：

1. 装 metrics-server + Prometheus（看历史）
2. 跑业务 1-2 周收集真实负载
3. 算 P50、P95、P99
4. **requests = P95 + 20% buffer**（不要用 P50——会频繁 throttle）
5. **limits = P99 × 2** 或 requests × 2-4 倍

```promql
# 单 Pod CPU P95（过去 7 天）
quantile_over_time(0.95, rate(container_cpu_usage_seconds_total{pod=~"api-.*"}[5m])[7d:1m])

# 单 Pod 内存 P95
quantile_over_time(0.95, container_memory_working_set_bytes{pod=~"api-.*"}[7d:1m])
```

### 13.2 VPA Off 模式做 sizing

让 VPA 持续推荐但不改 Pod：

```yaml
updatePolicy:
  updateMode: "Off"
```

每周看 `kubectl describe vpa` 的推荐值，对照实际配置调整。

### 13.3 HPA 抖动问题

**症状**：副本数频繁震荡（5 → 8 → 5 → 8）

**原因**：

1. CPU spike 触发扩容 → 新 Pod 起来后总 CPU 降低 → 触发缩容
2. 缩容后单 Pod 负载又高 → 又扩容

**解决**：

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 600    # 缩容必须连续低 10 分钟
    policies:
    - type: Pods
      value: 1
      periodSeconds: 180                # 每 3 分钟最多缩 1 个
  scaleUp:
    stabilizationWindowSeconds: 30
    policies:
    - type: Percent
      value: 50
      periodSeconds: 60
```

### 13.4 资源浪费监控

**关键指标**：

```promql
# CPU 申请 vs 实际
sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))

# 浪费率 = 1 - 实际/申请
1 - (
  sum by (namespace) (rate(container_cpu_usage_seconds_total[5m])) 
  / 
  sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
)
```

工具：

- **kube-resource-report**：按 namespace 出资源利用 report
- **goldilocks**：基于 VPA 自动推荐 requests
- **Krane / Kor**：找闲置资源
- **Crane**（华为）：集成的资源优化平台

### 13.5 PodDisruptionBudget

伸缩 / 节点缩容时保证最小可用副本：

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 3                       # 或 maxUnavailable: 1
  selector:
    matchLabels:
      app: api
```

VPA、Karpenter、kubectl drain 都尊重 PDB——保证扰动期间至少 3 个副本可用。

### 13.6 节点池策略

**典型分层**：

```
on-demand 池      → 控制面、监控、有状态服务
spot 池            → 无状态业务、批任务
gpu 池             → ML / 推理
arm 池             → 部分支持 ARM 的服务（省钱）
high-mem 池        → 缓存、内存数据库
```

不同节点池打不同 taint，业务通过 toleration 选。

### 13.7 容量预留（OverProvisioning）

为了让 Karpenter 提前起节点，部署"占位 Pod"：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
spec:
  replicas: 10
  template:
    spec:
      priorityClassName: overprovisioning            # 低优先级
      containers:
      - name: reserve
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: 1
            memory: 2Gi
```

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1
```

业务 Pod 一来抢占这些低优 Pod → Karpenter 起新节点放占位 Pod → 业务 Pod 直接调到现成节点。

**省下"节点启动 60s"的等待**。

### 13.8 GPU / 加速器调度

```yaml
spec:
  containers:
  - name: inference
    image: my-model:1.0
    resources:
      limits:
        nvidia.com/gpu: 1
```

GPU 是 extended resource——必须装 NVIDIA Device Plugin（或对应厂商 plugin）。

**高级用法**：

- **MIG（Multi-Instance GPU）**：A100/H100 分片成多个虚拟 GPU，按 `nvidia.com/mig-1g.10gb` 申请
- **GPU sharing**：Time-slicing 或 MPS，多个 Pod 共享一个 GPU（牺牲隔离换利用率）
- **NVIDIA GPU Operator**：自动管理 driver、device plugin、DCGM exporter

---

## 第十四章：陷阱清单

### 1. CPU throttling 看不出来

```
你看 CPU 使用率 60% 心想"还有富余"
但 P99 延迟飙升——其实正在 throttle
```

throttle 不是"CPU 使用满"，而是**单位时间内 CPU 时间片用光**：

- limit 500m 意味着每 100ms 周期最多用 50ms CPU
- 哪怕你大部分时间空闲，**spike 时间内**会被打断

**怎么看**：

```promql
rate(container_cpu_cfs_throttled_periods_total[5m]) 
/ 
rate(container_cpu_cfs_periods_total[5m])
```

> 10% 就要警惕；> 50% 严重。

**Linux kernel 5.14+** 优化了 CFS bandwidth 算法，throttle 没以前那么严重，但还在。

**解决**：去 CPU limit 或显著提高 limit。

### 2. Memory limit 把进程杀了，没告诉你

```bash
$ kubectl describe pod api-server
Last State: Terminated
  Reason: OOMKilled
  Exit Code: 137
```

进程根本没机会清理。常见原因：

- Java 应用没设 `-XX:MaxRAMPercentage`（JVM 看的是 cgroup limit，但默认只用 25%）
- Go 应用 `GOGC` 调得太高，GC 不及时
- 内存泄漏

**解决**：

```bash
# Go
GOMEMLIMIT=1750MiB                       # 比 limit 略小，触发 GC

# Java
-XX:MaxRAMPercentage=75
```

### 3. HPA 只看 CPU 不够

IO bound 服务（DB 连接池满、IO wait 高）的 CPU 不会 100%，但业务已经卡死。

**HPA 看 CPU → 永远不扩容 → 请求堆积 → 超时雪崩**。

解决：上自定义指标（qps、p99 延迟、并发数、队列长度）。

### 4. requests 设得太高浪费

```yaml
requests: { cpu: "2", memory: "4Gi" }   # 实际 P95 只用 200m / 500Mi
```

节点资源浪费 10x。**调度器看 requests**，节点很快"满"了但实际负载很低。

**症状**：节点 CPU 利用率 20%，但已经"Insufficient cpu"调不下 Pod 了。

**解决**：上 VPA Off 模式持续 sizing，或人工每月 review。

### 5. namespace ResourceQuota 没设

业务 Pod 没限制 → 一个 namespace 把整个集群资源吃光。

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
    persistentvolumeclaims: "20"
    pods: "100"
```

**核心集群一定要给每个 namespace 设 quota**。

### 6. LimitRange 没设

部分 Pod 不写 resources（BestEffort）→ 抢占其他 Pod 资源。

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:                            # 默认 limit
      cpu: 500m
      memory: 512Mi
    defaultRequest:                     # 默认 request
      cpu: 100m
      memory: 128Mi
    max:                                # 上限
      cpu: 4
      memory: 8Gi
    type: Container
```

### 7. Node Affinity preferred 没生效

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 1                              # 太小了
  preference: ...
```

权重 1 在和其他 plugin 一起 score 时几乎没影响。**weight 至少 50 才算偏好**，100 算强偏好。

### 8. PodAntiAffinity 全集群慢

```yaml
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - topologyKey: kubernetes.io/hostname  # 大集群每次调度扫所有节点上所有 Pod
```

500+ 节点集群 → 调度延迟从 ms 飙到秒级。**改用 TopologySpreadConstraints**。

### 9. PriorityClass 全打成高优

如果所有业务都打 `system-cluster-critical`，PriorityClass 退化为没有优先级。**只给真正不可缺失的业务高优**（控制面、监控、关键支付）。

### 10. KEDA 缩到 0 后没回来

```
trigger 失败 → KEDA 拉不到 metric → 默认 IsActive=false → minReplicas=0
```

如果 Kafka 短暂不可达，KEDA 会把消费者缩到 0——结果消息真来了反而无法处理。

**解决**：

- 设 `minReplicaCount: 1`（不缩到 0）
- 或 `idleReplicaCount: 0` + `minReplicaCount: 1`（有事件至少 1 个，空闲缩到 0）

### 11. VPA 把生产 Pod 重启了

生产环境开 `updateMode: Auto`，VPA 觉得 requests 不合理 → 驱逐 → 重启 Pod。**核心业务一定先 Off 模式观察**。

### 12. Karpenter 把节点合并了导致业务重启

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
```

激进合并 → 把多台节点的 Pod 集中到一台 → 大量 Pod 重启。

**解决**：

- 用 disruption budget 限制扰动速率
- 业务侧上 PDB
- 工作时间禁用合并：

```yaml
disruption:
  budgets:
  - nodes: "0"
    schedule: "0 9 * * 1-5"
    duration: 8h
```

### 13. cgroup v1 vs v2 行为不同

某些云厂商默认还是 cgroup v1（如 GKE 在 1.26 前默认 v1）：

- memory.min 不生效
- in-place pod resize 部分不支持
- 监控指标路径不同（`/sys/fs/cgroup/memory/...` vs `/sys/fs/cgroup/...`）

**确认**：

```bash
stat -fc %T /sys/fs/cgroup/
# cgroup2fs → v2
# tmpfs → v1
```

### 14. 节点资源被 system reserved 吃光不知道

```bash
# Allocatable = Capacity - system-reserved - kube-reserved - eviction-threshold
$ kubectl describe node node-1
Capacity:    cpu: 16, memory: 64Gi
Allocatable: cpu: 15500m, memory: 60Gi
```

500m CPU 给系统进程、kubelet、daemonset 之类。算容量时**用 Allocatable**，不是 Capacity。

### 15. HPA scaleDown 默认 5 分钟稳定期太长

低 QPS 服务流量降下来 5 分钟才开始缩容→ 资源浪费严重。**根据业务调**。

---

## 第十五章：2026 现状

### 15.1 调度器进展

| 特性 | 状态（2026-05）|
|---|---|
| Scheduler Framework | GA（1.19）|
| TopologySpread defaultConstraints | GA |
| Pod-level resources | alpha（1.32）→ beta（1.34）|
| In-place Pod Vertical Scaling | beta（1.33），默认开启的发行版增多 |
| DRA（Dynamic Resource Allocation，GPU/FPGA 新接口）| GA（1.34）替代 Device Plugin 长期方向 |
| MatchLabelKeys in PodTopologySpread | GA（1.30）|
| MinDomains in PodTopologySpread | GA（1.30）|
| NodeInclusionPolicy in PodTopologySpread | GA（1.31）|

### 15.2 弹性伸缩生态

```
横向 (replicas):    HPA (内置) + KEDA (事件驱动)
纵向 (resources):   VPA + In-place resize
节点级:             Karpenter (云厂商主推) / Cluster Autoscaler (传统)
作业排队:           Kueue (K8s 官方) / Volcano (大数据 AI)
多集群:             Karmada / KubeFed v2 / Cluster API
```

**2026 主流组合**：

- 平台层：Karpenter（K8s 节点伸缩，AWS/Azure/GCP 都有支持）
- 业务横向：HPA + KEDA 双管齐下
- Sizing：VPA Off 模式观察 + goldilocks 自动推荐
- AI/批：Kueue（K8s 官方排队）+ Volcano（gang scheduling）

### 15.3 与去年比变化

- **VPA InPlace 模式临近 GA**：长连接业务终于不用重启 Pod 改 requests
- **DRA 取代 Device Plugin**：GPU/FPGA 调度更灵活（按属性匹配 GPU 型号、显存、拓扑等）
- **Karpenter 多云**：Azure / GCP 官方 provider 稳定，多云团队也开始用
- **KEDA HTTP scaler**：新增 HTTP 入口流量 scaler，不再依赖 Prometheus 算 qps
- **Kueue 1.0**：K8s 官方批处理队列管理，配合 Volcano 用
- **Pod-level resources beta（1.34）**：sidecar pattern（如 Istio Ambient、OpenTelemetry agent）更省资源

### 15.4 与 Spark / Flink / Ray 集成

```
传统：YARN / Mesos 调度
2026：Kubernetes + Volcano/Kueue 调度 + 各项目原生 K8s operator

Spark on K8s （SparkOperator）
Flink on K8s（FlinkOperator）  
Ray on K8s（KubeRay）
```

大数据 / AI 训练全面 K8s 化——调度器层面的需求（gang scheduling、queue、preemption）已成熟。

### 15.5 成本优化趋势

| 维度 | 工具 / 策略 |
|---|---|
| Spot/Preemptible 利用 | Karpenter spot 池 + interruption handler |
| Right-sizing | VPA + goldilocks + kube-resource-report |
| 节点 binpacking | Karpenter consolidation |
| Zero scale | KEDA scale-to-zero |
| 多租户隔离 | ResourceQuota + LimitRange + Hierarchical Namespace |
| ARM 迁移 | Graviton/Ampere 比 x86 便宜 20-40% |

**2026 平均集群利用率**：

- 传统（CA + 固定机型）：CPU 20-30%、内存 30-40%
- Karpenter + Right-sizing + KEDA：CPU 50-60%、内存 50-70%

利用率翻倍——直接体现在云账单上。

---

## 第十六章：练习题

**练习 1**：解释为什么 Guaranteed QoS 的 Pod 比 Burstable 更不容易被 OOMKilled，且能享受 cpu manager static 策略。

**练习 2**：以下 Deployment 在 100 节点集群里部署 30 个副本，你希望：

- 跨 3 个可用区均匀分布
- 同一个 ReplicaSet 内**严格**分散，新老版本不互相影响
- 不与 redis-cache 在同一节点（增加故障域隔离）

写出 Pod spec 的 affinity / topologySpreadConstraints。

**练习 3**：用户上线了一个 Go 后端服务，单 Pod 处理 500 qps，CPU 用 1.5 核，内存 800Mi。流量预计 0-50k qps 之间波动。设计 HPA + KEDA 方案，包括：

- requests / limits
- HPA metric 与 behavior
- 是否需要 KEDA、用什么 trigger
- 节点是否需要 spot

**练习 4**：以下代码（kube-scheduler plugin）有什么问题？

```go
func (p *MyPlugin) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
    pods, _ := p.clientset.CoreV1().Pods("").List(ctx, metav1.ListOptions{})
    sameAppCount := 0
    for _, p := range pods.Items {
        if p.Labels["app"] == pod.Labels["app"] && p.Spec.NodeName == nodeInfo.Node().Name {
            sameAppCount++
        }
    }
    if sameAppCount >= 3 {
        return framework.NewStatus(framework.Unschedulable, "too many same app pods")
    }
    return framework.NewStatus(framework.Success)
}
```

**练习 5**：业务遇到"CPU 使用率只有 40%，但 p99 延迟突然飙到 2 秒"。给出排查步骤与可能原因。

**练习 6**：你的 Kafka 消费者部署用了 KEDA，每个 partition lag > 100 触发扩容，maxReplicaCount=20。但发现高峰期"扩到 20 也消化不掉"。除了加 maxReplicaCount，还有什么办法？

**练习 7**：解释为什么 HPA + VPA 都管 CPU 会形成"死循环"。如何同时用 HPA 和 VPA？

**练习 8**：写一段 Go 代码，用 `client-go` 创建一个 Pod，要求：

- Guaranteed QoS
- 容忍 `dedicated=gpu:NoSchedule` taint
- 申请 1 个 NVIDIA GPU
- 与已有 `app=trainer` Pod 在同一节点（数据局部性）

---

## 参考答案

**练习 1**：

- OOMKilled：QoS 决定 oom_score_adj。Guaranteed 是 -997（几乎不被内核 OOM killer 选中），Burstable 在 1000 区间随 requests 占比浮动，BestEffort 是 1000（最先被杀）。kubelet 的 eviction 也按 QoS 排序，BestEffort 最先被驱逐。
- cpu manager static：必须 Guaranteed + 整数核 requests 的容器才能拿到独占 CPU。kubelet 把这些核从默认 cgroup `cpuset` 池里剥离，专给该容器。**非 Guaranteed 即使设了整数核也不会独占**。

**练习 2**：

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
    matchLabelKeys:
    - pod-template-hash
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: redis-cache
        topologyKey: kubernetes.io/hostname
```

要点：

- `matchLabelKeys: [pod-template-hash]` 是 K8s 1.27+ 新特性——同一 ReplicaSet 内分散，发版时新老版本不互相干扰
- 与 redis-cache 反亲和用 `requiredDuringSchedulingIgnoredDuringExecution` 强制
- 不用 podAntiAffinity 做 zone 分散——大集群太慢，改用 topologySpread

**练习 3**：

```yaml
# Deployment
resources:
  requests: { cpu: "2", memory: "1Gi" }     # P95 用 1.5C 800M + 30% buffer
  limits:   { cpu: "4", memory: "2Gi" }     # 2x，留 burst

# HPA
spec:
  minReplicas: 3                              # 至少跨 3 AZ
  maxReplicas: 100                            # 50k qps / 500 qps = 100
  metrics:
  - type: Pods                                # 自定义 qps
    pods:
      metric: { name: http_requests_per_second }
      target: { type: AverageValue, averageValue: "400" }   # 留 20% buffer
  - type: Resource                            # 同时看 CPU 兜底
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 70 }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent, value: 100, periodSeconds: 30     # 30s 内可翻倍
    scaleDown:
      stabilizationWindowSeconds: 300                     # 5 分钟稳定期
      policies:
      - type: Pods, value: 2, periodSeconds: 60
```

KEDA 通常用在外部事件源（Kafka / SQS）。HTTP qps 用 HPA + Prometheus Adapter 即可。如果有"非工作时间静默"需求可加 KEDA Cron trigger。

节点：

- 主流量用 on-demand
- 高峰 burst 配 spot（spot 池 + 业务 toleration）
- Karpenter 自动选 m6i/c6i 系列；spot 中断率高的实例族（如 m5d）排除

**练习 4**：

问题：

1. **每次 Filter 调用都 list 全集群 Pod**——10000+ Pod 集群直接打挂 API server
2. 用了 clientset 而不是 framework.Handle().SnapshotSharedLister()，每次都是网络请求
3. Filter 应该是只读、高性能的——不应有外部 IO
4. `pods.Items` 包含所有 namespace 所有 Pod，没过滤
5. label 比较应该用 selector

正确做法：

```go
func (p *MyPlugin) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
    sameAppCount := 0
    targetApp := pod.Labels["app"]
    for _, p2 := range nodeInfo.Pods {                 // ← 节点局部，缓存里取
        if p2.Pod.Labels["app"] == targetApp {
            sameAppCount++
        }
    }
    if sameAppCount >= 3 {
        return framework.NewStatus(framework.Unschedulable, "too many same app pods")
    }
    return framework.NewStatus(framework.Success)
}
```

**练习 5**：

排查步骤：

1. `kubectl top pod` 看是否真 40%（可能 metric 不准，看 cAdvisor 原始）
2. **CPU throttle 检查**：

   ```promql
   rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])
   ```

   > 10% 就要怀疑

3. 看 Pod limit：如果 limit 设得太低（如 0.5 核），CPU 平均 40% 但 spike 时被 throttle
4. 看 GC / 锁：

   - Go：pprof 看 `goroutine` 和 `mutex`
   - Java：jstack + jstat
   - DB：连接池满 / 慢查询

5. **网络**：检查节点 IO wait、conntrack 满、DNS 慢
6. **下游依赖**：DB / 外部 API 慢导致请求堆积

可能原因（按概率）：

1. CPU throttle（limit 设低了或 cgroup v1 早期 kernel bug）
2. 数据库 / 外部依赖慢
3. GC 暂停（Java fullGC、Go stop-the-world）
4. 锁竞争（select-for-update、mutex）
5. 网络：DNS / TCP retrans / conntrack
6. kernel 调度延迟（CPU manager 没开 static，频繁迁核）

**练习 6**：

如果"扩到 maxReplicas=20 仍消化不完"，加 replica 已无效——因为 **Kafka 消费者并发上限 = partition 数量**。20 个消费者只能消费 20 个 partition；如果 topic 只有 20 个 partition，再加 Pod 也是空跑。

解决方案（按优先级）：

1. **增加 partition 数量**（`kafka-topics.sh --alter --partitions 100`）——下游必须支持重平衡
2. 提高单 Pod 消费速度：

   - 增大 batch size、fetch.min.bytes
   - 拉到本地后并发处理（消费 goroutine 池 + 业务 worker pool 分离）
   - 减少业务逻辑里的同步 IO（DB 批量写、缓存预热）
   - 优化 CPU bound 路径

3. **下游瓶颈**：

   - 写 DB → DB 性能上限就是消费上限，需要 partition by key 分桶 + 水平扩 DB
   - 调外部 API → 限流，加重试队列

4. 用 ScaledJob：每条消息独立 Job，避免长连接复用瓶颈（适合处理时间长的任务）
5. 上 Karpenter 保证节点供给（避免节点不足导致 Pod Pending）

**练习 7**：

死循环：

```
VPA 看到 CPU usage 高 → 调大 requests
    ↓
单 Pod requests 变大，HPA 公式：utilization = usage / requests
requests 变大 → utilization 降低
    ↓
HPA 看到 utilization < target → 缩容
    ↓
副本变少，单 Pod 实际负载升高
    ↓
VPA 又调大 requests → 循环
```

正确同用方式：

1. **HPA 用 CPU + VPA 用 memory**：维度不重叠
2. **HPA 用 custom metric（qps）+ VPA 调 CPU、memory**：HPA 不看 CPU 就不冲突
3. **VPA `updateMode: Off`**：只观察推荐值，人工调整 requests
4. **VPA `Initial` 模式**：只在新 Pod 创建时设 requests，运行中不动——HPA 扩出来的 Pod 自动用新推荐值

K8s SIG 也在讨论 **HPA-aware VPA**（VPA 在 HPA 启用时跳过 CPU 资源），2026-05 还在实验。

**练习 8**：

```go
package main

import (
    "context"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/api/resource"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
)

func createGPUPod(ctx context.Context) error {
    config, err := clientcmd.BuildConfigFromFlags("", "/path/to/kubeconfig")
    if err != nil { return err }
    cs, err := kubernetes.NewForConfig(config)
    if err != nil { return err }

    cpu := resource.MustParse("4")
    mem := resource.MustParse("16Gi")
    gpu := resource.MustParse("1")

    pod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "training-worker",
            Namespace: "default",
            Labels:    map[string]string{"app": "training-worker"},
        },
        Spec: corev1.PodSpec{
            Tolerations: []corev1.Toleration{
                {
                    Key:      "dedicated",
                    Operator: corev1.TolerationOpEqual,
                    Value:    "gpu",
                    Effect:   corev1.TaintEffectNoSchedule,
                },
            },
            Affinity: &corev1.Affinity{
                PodAffinity: &corev1.PodAffinity{
                    RequiredDuringSchedulingIgnoredDuringExecution: []corev1.PodAffinityTerm{
                        {
                            LabelSelector: &metav1.LabelSelector{
                                MatchLabels: map[string]string{"app": "trainer"},
                            },
                            TopologyKey: "kubernetes.io/hostname",
                        },
                    },
                },
            },
            Containers: []corev1.Container{
                {
                    Name:  "worker",
                    Image: "my-trainer:1.0",
                    Resources: corev1.ResourceRequirements{
                        Requests: corev1.ResourceList{
                            corev1.ResourceCPU:                cpu,
                            corev1.ResourceMemory:             mem,
                            corev1.ResourceName("nvidia.com/gpu"): gpu,
                        },
                        Limits: corev1.ResourceList{
                            corev1.ResourceCPU:                cpu,
                            corev1.ResourceMemory:             mem,
                            corev1.ResourceName("nvidia.com/gpu"): gpu,
                        },
                    },
                },
            },
        },
    }

    _, err = cs.CoreV1().Pods("default").Create(ctx, pod, metav1.CreateOptions{})
    return err
}
```

关键点：

- requests == limits → Guaranteed QoS
- tolerations 容忍 gpu taint
- podAffinity hostname → 同节点
- nvidia.com/gpu 是 extended resource，requests 与 limits 必须相同（不能 burst）
- 实际生产 ML 任务应该用 Job/StatefulSet 或 Volcano gang scheduling

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Requests | 调度依据；cpu.weight / memory.min |
| Limits | cgroup 上限；CPU throttle / Memory OOM |
| QoS | Guaranteed (req=lim) / Burstable / BestEffort |
| 驱逐顺序 | BestEffort → Burstable → Guaranteed |
| NodeAffinity | required（硬）/ preferred（软）|
| TopologySpread | maxSkew 控制拓扑分散；推荐替代 podAntiAffinity |
| Taint / Toleration | NoSchedule / PreferNoSchedule / NoExecute |
| PriorityClass | value 越大优先级越高；可抢占低优 |
| HPA | v2、resource + custom metric、behavior 调抖动 |
| VPA | Off 观察 / Initial 新 Pod / Auto 重启 / InPlace 1.33 alpha |
| KEDA | 事件驱动、scale-to-zero、60+ scaler |
| Cluster Autoscaler | node group 模型，传统 |
| Karpenter | 按需选机型、binpacking、spot 友好，2026 主流 |
| In-place Resize | 1.33 beta，长连接业务受益 |
| 调度器扩展 | Framework + 自定义 plugin |
| Volcano / Kueue | gang scheduling / job queue |

铁律：

- requests 用 P95+20%，limits 用 2-4x requests（除非业务有特殊需求）
- 核心业务 Guaranteed QoS + PriorityClass + PDB
- HPA 不止 CPU——上 qps / 延迟 / 队列长度 custom metric
- HPA + VPA 不要都管 CPU
- 大集群用 TopologySpreadConstraints 替代 podAntiAffinity
- Karpenter 用 consolidation 省钱，但加 disruption budget 防扰动
- KEDA scale-to-0 适合事件型，HTTP API 留 minReplicas ≥ 1
- 监控 CPU throttle 而不仅是 CPU 利用率
- ResourceQuota + LimitRange 每个 namespace 必装
- 上线前用 VPA Off 模式 sizing 1-2 周

下一篇 **C07 — 精通 Storage 与 PVC** 将拆开 PV / PVC 生命周期、StorageClass、CSI 协议、StatefulSet 持久化与备份恢复（Velero）。
