# 精通多集群与 Karmada：跨集群编排、网络与可观测

> 课程编号：C16
> 路线图来源：云原生工程 · 模块七 扩展形态
> 难度：⭐⭐⭐⭐⭐
> 预计阅读时间：70 分钟
> 内容基准：2026 年 6 月

---

## 引言：单集群天花板与多集群必然性

C13 提到单集群上限：官方 5000 节点 / 15 万 Pod，实操 1500-2000 节点最舒服。但即使没顶到这个上限，企业到一定规模都会做多集群——这是**架构必然**而非"扩容副产品"。

为什么必然？

1. **故障域隔离**：单集群 etcd / API server / 调度器全挂时全业务受影响
2. **大版本升级回滚域**：K8s 1.30 → 1.31 升级带风险，按集群灰度
3. **区域 / 合规分隔**：中国数据不出境、欧盟 GDPR、HIPAA 医疗
4. **多云策略**：避免单一云厂锁定，跨 AWS / GCP / Azure 部署
5. **租户 / 团队隔离**：金融机构每业务线独立集群
6. **边缘场景**：CDN POP / 工厂 / 门店各一个 K3s 集群
7. **混合部署**：核心业务自建 + 弹性能力上云

多集群带来的挑战：

- 应用怎么部署到 N 个集群、怎么按权重分流？
- 集群间网络怎么打通？
- 监控 / 日志怎么聚合？
- 集群本身的生命周期（创建 / 升级 / 销毁）怎么管？

本章拆解四条主线：**Karmada（编排）**、**Cluster API（生命周期）**、**多集群网络（Submariner / Cilium Cluster Mesh）**、**多集群可观测**。

---

## 第一章：多集群拓扑模型

### 1.1 三种主流模型

```
模型 A：Hub-Spoke（中心辐射）
  ┌─────────────┐
  │  Hub (控制) │
  └──────┬──────┘
         │
   ┌─────┼─────┐
   ▼     ▼     ▼
 Spoke1 Spoke2 Spoke3
 (业务) (业务) (业务)

  Karmada / Open Cluster Management / Rancher Fleet
  → 单一控制面，编排所有 spoke


模型 B：Federation（联邦）
  Cluster A ◄──► Cluster B ◄──► Cluster C
  各自平等，互相同步 API 对象
  → KubeFed v2（已不推荐，项目放缓）


模型 C：GitOps-driven（去中心化）
  ┌─────────┐
  │   Git   │
  └────┬────┘
       │ (各集群独立 pull)
   ┌───┼───┐
   ▼   ▼   ▼
   A   B   C
   ArgoCD/Flux

  → ArgoCD ApplicationSet 多集群部署
  → 没有"控制面"，每集群独立 reconcile
```

### 1.2 选型决策

| 需求 | 推荐 |
|---|---|
| 想要"一次部署到 N 个集群"的 K8s 原生体验 | Karmada |
| 已有 ArgoCD 生态，想 GitOps 多集群 | ArgoCD ApplicationSet |
| Red Hat / OpenShift 生态 | Open Cluster Management (OCM) |
| 边缘海量集群（千-万级） | KubeEdge / Rancher Fleet |
| 简单几集群、不想引入中间件 | kubectx + Helmfile / ArgoCD 起步 |

### 1.3 Karmada 为什么主流

CNCF 沙盒（2021-09）→ 孵化（2023-09），华为 + 大量企业贡献。

优势：

- **K8s 原生 API**——`kubectl apply` 一份 Deployment YAML，Karmada 自动分发到指定集群
- **PropagationPolicy / OverridePolicy** 声明式控制分发与差异化
- **调度器跨集群**——按资源 / 权重 / 副本 / 多目标做调度
- **不要求 spoke 集群网络互通**——比 Federation 更灵活
- **支持失败转移、副本调度、动态权重**

劣势：

- 控制面单点（高 HA 需自己搭）
- 复杂度比 ArgoCD ApplicationSet 高

---

## 第二章：Karmada 架构深拆

### 2.1 组件

```
┌────────────────── Karmada Control Plane ──────────────────┐
│                                                            │
│   karmada-apiserver  (用户面 API 入口)                      │
│        │                                                   │
│   karmada-controller-manager  (PropagationPolicy 等)        │
│   karmada-scheduler           (跨集群调度)                  │
│   karmada-webhook             (admission 校验)              │
│   karmada-aggregated-apiserver(聚合查询)                    │
│   karmada-search              (跨集群搜索)                  │
│                                                            │
└──────────────────┬─────────────────────────────────────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
       ▼           ▼           ▼
  Member Cluster A  B  C    (普通 K8s 集群)
  + karmada-agent (push 模式可选)
```

Karmada 控制面通常**自己装在一个独立的 host 集群**（不是 member 之一）。

### 2.2 资源类型

Karmada 的"工作模型"：

```
1. ResourceTemplate
   - 用户写的标准 K8s 资源（Deployment / Service / ConfigMap）
   - 提交到 karmada-apiserver

2. PropagationPolicy / ClusterPropagationPolicy
   - 描述"这个资源要分发到哪些集群、按什么策略"
   - 选 cluster / region / label / 副本切分

3. OverridePolicy / ClusterOverridePolicy
   - 描述"在某个集群里要对资源做什么修改"
   - 例如 image 替换 / replicas 调整 / env 注入

4. Work
   - Karmada 内部生成，对应"在某个 member 集群中实际部署的版本"

5. ResourceBinding
   - 资源与目标集群的绑定记录
```

### 2.3 一个最小例子

```yaml
# 1. 普通 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 6
  selector:
    matchLabels: {app: nginx}
  template:
    metadata: {labels: {app: nginx}}
    spec:
      containers:
      - name: nginx
        image: nginx:1.27

---
# 2. PropagationPolicy：分发到 3 个 member，按 1:2:3 切分副本
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-pp
  namespace: default
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: nginx
  placement:
    clusterAffinity:
      clusterNames: [member1, member2, member3]
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
        - targetCluster: {clusterNames: [member1]}
          weight: 1
        - targetCluster: {clusterNames: [member2]}
          weight: 2
        - targetCluster: {clusterNames: [member3]}
          weight: 3
```

效果：6 个副本分别在 member1 / 2 / 3 部署 1 / 2 / 3 个。

### 2.4 OverridePolicy 差异化

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: nginx-op
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: nginx
  overrideRules:
  - targetCluster:
      clusterNames: [member3]
    overriders:
      imageOverrider:
      - component: Tag
        operator: replace
        value: "1.28-arm"  # member3 是 ARM 节点池
```

让 member3 集群用 ARM 镜像，其它集群保持原 image。这是同一份模板适配多环境的核心机制。

### 2.5 调度策略

Karmada Scheduler 支持多种 placement：

- **Static weight**：手动权重
- **Dynamic weight by AvailableReplicas**：按集群可用资源动态分配
- **Duplicated**：每个集群一份完整副本（如全局 DaemonSet）
- **Divided**：副本切分到多集群
- **Aggregated**：尽量集中到能容纳的集群（避免碎片化）
- **Failover**：集群故障时自动迁移到其它集群

```yaml
placement:
  replicaScheduling:
    replicaSchedulingType: Divided
    replicaDivisionPreference: Aggregated   # 尽量集中
  clusterTolerations:
  - key: cluster.karmada.io/not-ready
    effect: NoExecute
    tolerationSeconds: 60   # 60s 不可达就 failover
```

---

## 第三章：Karmada 实战部署

### 3.1 安装控制面

```bash
# karmadactl 一键
karmadactl init --kubeconfig=$HOME/.kube/config

# 或 helm
helm install karmada karmada-charts/karmada \
  -n karmada-system --create-namespace
```

控制面包含 6+ 个组件，建议给独立 namespace 与节点池。

### 3.2 加入 member 集群

```bash
# 方式 1: push 模式（控制面直接连 member）
karmadactl join member1 \
  --kubeconfig=$HOME/.kube/karmada-config \
  --cluster-kubeconfig=$HOME/.kube/member1-config

# 方式 2: pull 模式（member 主动连 hub，适合 NAT / 防火墙场景）
karmadactl register --karmada-server=... --token=...
```

```bash
kubectl --kubeconfig karmada-config get clusters
# NAME      VERSION   MODE   READY
# member1   v1.32     Push   True
# member2   v1.32     Pull   True
# member3   v1.33     Push   True
```

### 3.3 联邦操作

```bash
# 提交到 Karmada API server
kubectl --kubeconfig karmada-config apply -f deployment.yaml -f pp.yaml

# 查 ResourceBinding（自动生成）
kubectl --kubeconfig karmada-config get resourcebinding

# 查每个 member 实际部署
kubectl --kubeconfig member1-config get deploy
kubectl --kubeconfig member2-config get deploy
```

### 3.4 Failover 演练

```bash
# 模拟 member1 失联
karmadactl cordon member1

# 等 toleration 时间过后，Karmada 自动把 member1 的副本调到其它 member
kubectl --kubeconfig karmada-config get resourcebinding -o yaml | grep cluster
```

---

## 第四章：集群生命周期——Cluster API

Karmada 解决"如何把应用部署到集群"，**Cluster API（CAPI）** 解决"如何创建 / 升级 / 销毁集群本身"。

### 4.1 思想：把集群当 K8s 资源

CAPI 把 K8s 集群本身建模为 K8s CRD。在 management cluster 里创建一个 `Cluster` 资源，CAPI Controller 就会调用云 API（AWS / Azure / GCP / vSphere / metal）创建出真实集群。

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: prod-us-east-1
spec:
  clusterNetwork:
    services:
      cidrBlocks: ["10.96.0.0/12"]
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  infrastructureRef:
    kind: AWSCluster
    name: prod-us-east-1
  controlPlaneRef:
    kind: KubeadmControlPlane
    name: prod-us-east-1-cp
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: prod-us-east-1
spec:
  region: us-east-1
  ...
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
spec:
  replicas: 3
  machineTemplate:
    infrastructureRef:
      kind: AWSMachineTemplate
      name: prod-us-east-1-cp
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        kubeletExtraArgs: {cloud-provider: aws}
```

apply 上去，几分钟内得到一个全新集群。

### 4.2 升级集群

```yaml
spec:
  version: v1.33.0   # 改个版本号
```

CAPI 触发**滚动升级**：先升 control plane（依次替换 master），再升 worker（按 MachineDeployment 滚动）。整个过程像升级 Deployment 一样声明式。

### 4.3 与 Karmada 联动

CAPI 创建出来的集群可以自动注册到 Karmada：

```
1. CAPI 创建 cluster X
2. cluster X ready 后通过 cluster-api-provider-karmada（社区插件）
3. 自动调用 karmadactl join 把 cluster X 接入
4. Karmada PropagationPolicy 自动开始把工作负载分发到 X
```

完整闭环：**集群按需扩张，工作负载自动跟随**。

### 4.4 GitOps 化集群管理

```
Git repo:
  clusters/
    prod-us-east-1.yaml   ← Cluster + AWSCluster + KubeadmControlPlane
    prod-eu-west-1.yaml
    prod-ap-1.yaml
  apps/
    web/deployment.yaml
    web/pp.yaml
```

ArgoCD 监听 `clusters/`：一旦合入新 cluster yaml，CAPI 创建集群；CAPI 自动接入 Karmada；Karmada 部署 `apps/` 到新集群。

**这就是 "Infrastructure-as-Code on K8s" 的终极形态**——从集群到应用一切声明式 + Git。

---

## 第五章：多集群网络

应用部署到多集群后，跨集群通信怎么办？

### 5.1 三种解法

**A. 完全独立**

每集群独立，只有南北向流量（Gateway / LB），跨集群通信走公网 / VPN。简单但跨集群不通信场景才适用。

**B. East-West Gateway（Mesh-based）**

```
集群 A 内部服务 ──► A 的 east-west gateway ──公网/专线──► B 的 east-west gateway ──► B 内部服务
                  (mTLS 终结)                            (mTLS 重新建立)
```

Istio multi-cluster / Linkerd multi-cluster 都走这种思路。SNI 路由 + mTLS。

**C. Pod-to-Pod 直通（CNI-based）**

```
集群 A 的 Pod IP 10.0.1.5 ──► 集群 B 的 Pod IP 10.0.2.10
                            (CNI 把跨集群路由打平)
```

Cilium Cluster Mesh / Submariner 是代表。需要 Pod CIDR 不冲突。

### 5.2 Cilium Cluster Mesh

```bash
# 两个 Cilium 集群互相注册
cilium clustermesh enable --context=cluster-a
cilium clustermesh enable --context=cluster-b

cilium clustermesh connect --context cluster-a --destination-context cluster-b
```

实现：Cilium 在每集群保有对端的 service / endpoint 元数据，eBPF 直接路由 Pod-to-Pod（前提 underlay 网络可达）。

**优势**：

- L4/L7 NetworkPolicy 跨集群一致
- Service "Global" 标记后，对端集群也能解析（`service-a.cluster-a.local` / 直接 svc DNS）
- 性能接近单集群（不经过额外代理）

**前提**：

- 两集群 Pod CIDR / Service CIDR 不重叠
- 节点间网络可达（VPC peering / VPN / 专线）

### 5.3 Submariner

CNCF 沙盒项目，专注 K8s 多集群 Pod-to-Pod。

- 部署 **Submariner Operator** 在每集群
- 自动建立 **IPsec / WireGuard / VxLAN tunnel** 跨集群
- **Service Discovery via Lighthouse**（MCS API 实现）

适合不同 CNI / 不同云厂商的集群互通。

### 5.4 MCS API（Multi-Cluster Services）

K8s SIG 标准化的"多集群服务发现"API：

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: web   # 把这个 ns/svc "导出"到 ClusterSet
  namespace: prod
```

```yaml
# 任何 cluster 都会自动生成
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: web
  namespace: prod
```

应用可以用 `web.prod.svc.clusterset.local` 解析，跨集群 LB。

Cilium Cluster Mesh / Submariner / Istio multi-cluster 都开始支持 MCS API。

---

## 第六章：多集群可观测

### 6.1 Metrics 联邦

```
方案 A：每集群独立 Prometheus + Thanos sidecar → 对象存储 → 全局 Querier
方案 B：每集群 Prometheus remote_write → 中心 VictoriaMetrics / Mimir
方案 C：每集群独立，Grafana 配多 datasource 跨集群
```

最常用：**Thanos**

```
cluster A         cluster B         cluster C
Prom + Thanos     Prom + Thanos     Prom + Thanos
sidecar           sidecar           sidecar
   │                 │                 │
   └────► S3 / Object Storage ◄────────┘
                     │
                     ▼
              Thanos Querier
                     │
                     ▼
                  Grafana
```

`{cluster="prod-us-east-1"}` label 区分集群。

### 6.2 日志

- 每集群独立 Loki + Promtail → S3
- 或 fluentbit → 中心 ES / Loki / S3
- 维度：`cluster` / `namespace` / `pod`

### 6.3 Tracing

OpenTelemetry Collector 每集群 DaemonSet → 中心 Gateway → Tempo / Jaeger / Datadog。

trace 跨集群天然支持（W3C TraceContext header 跨 HTTP 透传）。

### 6.4 统一控制台

- **Karmada Dashboard / kosmos**：Karmada 自带
- **Rancher**：商业多集群管控
- **Backstage**：开发者门户，跨集群展示资源 + 服务目录
- **OCM**：Red Hat 多集群管控

---

## 第七章：高级形态——Liqo / Clusternet

### 7.1 Liqo：集群"虚拟化"

Liqo 把多个集群看成"超级集群"：

- 集群 B 向集群 A "出借"资源
- 集群 A 上看起来有更多 Node（virtual node 对应集群 B）
- Pod 调度到 virtual node 实际跑在集群 B 上

适合边缘 + 中心、burst 到公有云等场景。

### 7.2 Clusternet：阿里 / 华为风格

类似 Karmada 但更强调中心调度 + agent 模式，CNCF 沙盒。功能集与 Karmada 高度重合，社区动态较弱于 Karmada。

---

## 第八章：生产实践

### 8.1 集群命名与标签

```yaml
metadata:
  name: prod-us-east-1
  labels:
    env: prod
    region: us-east-1
    provider: aws
    fleet: web-platform
```

后续 PropagationPolicy 按 label selector 选目标更灵活：

```yaml
placement:
  clusterAffinity:
    labelSelector:
      matchExpressions:
      - key: env
        operator: In
        values: [prod]
      - key: region
        operator: In
        values: [us-east-1, us-west-2]
```

### 8.2 灰度发布

```yaml
# 按集群分批
placement:
  replicaScheduling:
    replicaSchedulingType: Divided
    weightPreference:
      staticWeightList:
      - targetCluster: {clusterNames: [staging]}
        weight: 100
      - targetCluster: {clusterNames: [prod-us-east-1, prod-eu-west-1]}
        weight: 0  # 暂时 0，灰度通过后再加权重
```

配合 ArgoCD Rollouts，集群内也做金丝雀。

### 8.3 故障转移

```yaml
spec:
  failover:
    application:
      decisionConditions:
        tolerationSeconds: 60
      gracePeriodSeconds: 300
      purgeMode: Graceful   # 优雅迁移
```

集群 down → 60s 后副本调到其它集群 → 300s 内不强删（给 mesh 收敛时间）。

### 8.4 SLO 与降级

多集群 ≠ 100% 可用。仍需：

- 每集群独立的本地 SLO 监控
- 跨集群健康检查 + 流量切换（GSLB / DNS-based failover）
- 把 "单集群可用" 与 "全局可用" 区分上报

---

## 第九章：陷阱清单

1. **Pod CIDR 冲突** 是跨集群网络第一坑——规划时给每集群分独立 CIDR。
2. **Karmada 控制面单点** 默认 1 副本，生产至少 3 副本 + 独立 etcd。
3. **PropagationPolicy 写错触发全集群部署** ——先 dry-run 看 ResourceBinding。
4. **OverridePolicy 顺序敏感**——多条 override 按顺序合并，写错顺序结果不可预期。
5. **集群版本错配**——Karmada 控制面 v1.32，member v1.30，某些 CRD 不兼容。
6. **跨集群 mTLS 复杂度**——根证书要互信，过期了全断；建议 cert-manager + SPIRE 自动化。
7. **观测链路上行带宽**——每集群 Prom 远程写中心，几百节点几十 MB/s 持续占带宽。
8. **CAPI 升级时 control plane 抖动**——5 分钟内 API server 间歇性 unavailable，业务无感但 controllers 受影响。
9. **member 集群被 reset** 后 Karmada 还以为它在线——配 keepalive 检测。
10. **ApplicationSet vs Karmada 两套并用**——同一份 yaml 被两个系统 reconcile，相互覆盖。
11. **Liqo 资源借用计费不清**——A 集群的工作负载吃 B 集群资源，谁付钱？
12. **多集群审计盲点**——本集群审计日志看不到 Karmada 下发动作，要把 Karmada 操作日志单独留。

---

## 第十章：2026 现状

- **Karmada** CNCF Incubation（2023-09），社区活跃，华为云 + 多家厂商
- **Cluster API** v1.7+，AWS / Azure / GCP / vSphere / OpenStack 等 provider 完备
- **Cilium Cluster Mesh** 是 eBPF 多集群方案事实标准
- **MCS API** alpha → beta（2025），多家实现支持
- **OCM (Open Cluster Management)** Red Hat 推动，与 ACM 商业化对应
- **KubeFed v2** 项目已 archive
- **Liqo** 1.0 GA（2025-03），生态扩张中
- **GitOps 多集群** ArgoCD ApplicationSet + Cluster API + Karmada 三角是主流栈
- **AI 训练多集群**：Karmada + Volcano 调度跨集群 GPU，2025 大量场景落地
- **Edge multi-cluster**：KubeEdge / Akri / Rancher Fleet 千万级边缘节点管理

---

## 第十一章：练习题

1. ⭐ 为什么大公司即使没顶到单集群上限也要做多集群？至少 4 个理由。
2. ⭐ Karmada 与 ArgoCD ApplicationSet 的核心差异？
3. ⭐⭐ 解释 PropagationPolicy 与 OverridePolicy 的分工。写一个例子：把 Deployment 部署到 3 个集群，其中 1 个用不同镜像。
4. ⭐⭐ Cluster API 的核心抽象是什么？为什么说它把"集群"提升为 K8s 一等公民？
5. ⭐⭐ Cilium Cluster Mesh / Submariner / Istio multi-cluster 三种多集群网络方案的核心差异？
6. ⭐⭐⭐ 设计一个"集群挂了 5 分钟内业务自动迁移到其它集群"的方案，包含：CAPI / Karmada / DNS / SLO 监控。
7. ⭐⭐⭐ 多集群 Prometheus 联邦的三种方案对比（独立 + Querier / remote_write 集中 / Grafana 多 ds），优劣？
8. ⭐⭐⭐ 一个 100 集群、1 万节点的多云架构，列出从集群创建到应用部署到观测的完整工具链。

<details>
<summary>📝 参考思路</summary>

1. ① 故障域隔离；② 升级灰度域；③ 区域 / 合规；④ 多云策略；⑤ 团队 / 租户隔离；⑥ 边缘场景；⑦ 混合云弹性。
2. Karmada：K8s 风格 CRD + 跨集群调度器，支持副本切分；ApplicationSet：GitOps 视角，每集群独立 sync，不做跨集群调度。Karmada 重编排，ApplicationSet 重 Git/Pull。
3. 见第二章。PP 控制"分发到哪些集群、按什么权重"；OP 控制"在某集群里要做什么差异化修改"。例子见 2.3-2.4 节。
4. Cluster API 把 Cluster / ControlPlane / MachineDeployment 等都建模为 CRD，management cluster 通过 controller 创建 / 升级 / 销毁真实集群——集群本身像 Deployment 一样声明式。
5. Cilium Cluster Mesh：CNI 层 Pod-to-Pod，eBPF 直通，要求 CIDR 不冲突；Submariner：tunnel-based，跨 CNI 兼容性强；Istio multi-cluster：east-west gateway + mTLS，应用层级、对 CIDR 无要求但延迟高一些。
6. CAPI 提供备用集群；Karmada PropagationPolicy 配 failover toleration 60s；Service 用 GSLB（route53/cloud DNS）按健康检查切换 region；SLO 监控 + Alertmanager 触发 PagerDuty。
7. 独立 + Thanos Querier：S3 共享、查询延迟稍高；remote_write 集中：实时性好但中心存储压力大；Grafana 多 ds：简单但缺统一查询 / 跨集群聚合。
8. CAPI 建集群 → Karmada 联邦 → Cilium Cluster Mesh 网络 → ArgoCD ApplicationSet GitOps → OpenTelemetry + Thanos + Loki + Tempo 可观测 → Backstage 服务目录 → Cosign / Kyverno 安全策略。

</details>

---

## 小结

多集群不是"扩容副作用"，是**架构必然**。一旦有合规 / 故障域 / 多云 / 边缘需求，单集群就到天花板。

```
应用编排  : Karmada / OCM / ArgoCD ApplicationSet
集群生命周期: Cluster API
跨集群网络: Cilium Cluster Mesh / Submariner / Istio multi-cluster
跨集群服务: MCS API
可观测   : Thanos + Loki + OTel
GitOps整合: 全部 yaml in Git，ArgoCD 双层 reconcile
```

至此 cloud-native 16 章覆盖了**从单容器到多集群多云的完整生命周期**——再往下是 Wasm / 边缘 / AI 平台等"扩展形态"，可按需深挖。
