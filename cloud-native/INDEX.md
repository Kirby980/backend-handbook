# 云原生深度课程 · 总目录

> 面向 Go / 后端工程师的 Kubernetes & 云原生系统进阶，共 16 篇万字长文
> 每篇约 10000-15000 字，含底层原理、Go 代码示例、生产实践、陷阱清单与练习题
> 适合从"会写 Dockerfile"到"运维生产级 K8s 平台"的进阶
>
> **📅 内容基准：2026 年 5 月**——Kubernetes 1.36（2026-04 release，1.34/1.35 仍广泛在用）、Gateway API GA、Istio Ambient GA、Cilium 1.16+ eBPF、Helm 3.x、ArgoCD 3.x、Containerd 2.x、OCI Image Spec v1.1、KEDA 2.x。

---

## 📚 课程总览

| # | 课程 | 难度 | 关键词 |
|---|---|---|---|
| C01 | [精通 Docker 与 OCI](./C01-精通-Docker-与-OCI.md) | ⭐⭐⭐ | 多阶段构建 / distroless / scratch / 层缓存 / BuildKit |
| C02 | [精通 Kubernetes 工作负载](./C02-精通-K8s-工作负载.md) | ⭐⭐⭐⭐ | Pod / Deployment / StatefulSet / DaemonSet / Job |
| C03 | [精通 K8s 网络与 Service](./C03-精通-K8s-网络与-Service.md) | ⭐⭐⭐⭐ | Service / Endpoint / DNS / kube-proxy / CNI |
| C04 | [精通 Ingress 与 Gateway API](./C04-精通-Ingress-与-Gateway-API.md) | ⭐⭐⭐⭐ | Ingress / Gateway API v1 / HTTPRoute / TLSRoute |
| C05 | [精通 ConfigMap、Secret 与配置](./C05-精通-ConfigMap-与-Secret.md) | ⭐⭐⭐ | ConfigMap / Secret / External Secrets / Vault |
| C06 | [精通 Scheduling 与资源管理](./C06-精通-Scheduling-与资源管理.md) | ⭐⭐⭐⭐ | Requests/Limits / HPA / VPA / KEDA / 节点亲和 |
| C07 | [精通 Storage 与 PVC](./C07-精通-Storage-与-PVC.md) | ⭐⭐⭐⭐ | PV / PVC / StorageClass / CSI / 有状态服务 |
| C08 | [精通 Helm 与 Kustomize](./C08-精通-Helm-与-Kustomize.md) | ⭐⭐⭐ | Helm Chart / templating / Kustomize overlays |
| C09 | [精通 Operator 与 CRD](./C09-精通-Operator-与-CRD.md) | ⭐⭐⭐⭐⭐ | CRD / controller-runtime / Kubebuilder / Reconcile |
| C10 | [精通 Service Mesh](./C10-精通-Service-Mesh.md) | ⭐⭐⭐⭐⭐ | Istio Ambient / Linkerd / Cilium / mTLS / 流量治理 |
| C11 | [精通 K8s 可观测性](./C11-精通-K8s-可观测性.md) | ⭐⭐⭐⭐ | kube-state-metrics / cAdvisor / Loki / Tempo / eBPF |
| C12 | [精通 K8s 安全](./C12-精通-K8s-安全.md) | ⭐⭐⭐⭐⭐ | RBAC / Pod Security / OPA Gatekeeper / Kyverno |
| C13 | [精通生产调优](./C13-精通生产调优.md) | ⭐⭐⭐⭐ | 节点池 / NodeLocal DNS / 容量规划 / 多集群 |
| C14 | [精通 GitOps](./C14-精通-GitOps.md) | ⭐⭐⭐⭐ | ArgoCD / Flux / Sync Wave / 渐进式发布 |
| C15 | [精通 Serverless on K8s](./C15-精通-Serverless-on-K8s.md) | ⭐⭐⭐⭐ | Knative Serving / Eventing / KEDA HTTP / OpenFaaS / 冷启动 |
| C16 | [精通多集群与 Karmada](./C16-精通-多集群与-Karmada.md) | ⭐⭐⭐⭐⭐ | Karmada / Cluster API / Cilium Cluster Mesh / Submariner / MCS |

---

## 🗺️ 按模块组织

### 🟢 模块一：容器与编排基础（C01-C03）

> 把"应用"装进"声明式集群"，第一步绕不过去。

- **C01 Docker / OCI**：镜像分层、多阶段构建、distroless / scratch、BuildKit cache、镜像签名
- **C02 工作负载**：Pod 生命周期、Deployment 滚动、StatefulSet 顺序、DaemonSet 节点级、Job/CronJob 批
- **C03 网络与 Service**：ClusterIP / NodePort / LoadBalancer、Endpoint / EndpointSlice、kube-proxy iptables/IPVS、CoreDNS、CNI（Calico/Cilium/flannel）

### 🔵 模块二：流量入口与配置（C04-C05）

- **C04 Gateway API**：Gateway / HTTPRoute / TLSRoute / GRPCRoute；从 Ingress 迁移；Istio / Envoy Gateway / Cilium / Kong / Traefik 多实现
- **C05 配置 / 密钥**：ConfigMap 投递机制、Secret 加密（KMS / sealed-secrets）、External Secrets Operator、Vault 动态密钥

### 🟡 模块三：资源与存储（C06-C07）

- **C06 资源管理**：Requests vs Limits、QoS 等级、cpu manager、HPA（metric server / 自定义指标）、VPA、KEDA 事件驱动伸缩、节点亲和与污点
- **C07 存储**：PV / PVC 生命周期、StorageClass、CSI 协议、本地盘 vs 网络盘、StatefulSet + 持久化、备份恢复（Velero）

### 🔴 模块四：扩展与平台化（C08-C09）

- **C08 包管理**：Helm chart 结构、Helmfile、Kustomize overlay、何时用哪个、CI 集成
- **C09 Operator**：CRD 设计、controller-runtime 工作机制、Reconcile 循环、leader election、status 子资源、Kubebuilder 脚手架、Go 写 Operator 实战

### 🟣 模块五：网格、可观测、安全（C10-C12）

- **C10 Service Mesh**：sidecar vs Ambient、Istio Ambient（ztunnel + waypoint）、Linkerd 轻量、Cilium Service Mesh（eBPF）、mTLS、流量切分、熔断
- **C11 可观测性**：kube-state-metrics、cAdvisor、Prometheus 抓取、Loki 日志、Tempo 链路、Pixie / Hubble eBPF 可观测
- **C12 安全**：RBAC 最佳实践、Pod Security Admission（PSA）、OPA Gatekeeper、Kyverno、镜像扫描（Trivy / Grype）、运行时（Falco）、Cosign 镜像签名

### 🟠 模块六：生产化（C13-C14）

- **C13 生产调优**：节点池策略、NodeLocal DNS Cache、Pod 启动加速（image preload）、控制面调优、容量规划、多集群方案（Karmada / Cluster API）
- **C14 GitOps**：ArgoCD 架构、Flux v2、Sync Wave、PR 触发部署、Progressive Delivery（Argo Rollouts / Flagger）、与 Helm/Kustomize 整合

### 🟤 模块七：扩展形态（C15-C16）

> 单集群 + 常驻负载之外，K8s 的两大延伸方向：Serverless 与多集群。

- **C15 Serverless on K8s**：Knative Serving / Eventing、KPA 自动扩缩、scale-to-zero、Activator 兜流、冷启动优化；KEDA HTTP add-on 轻量方案；OpenFaaS / KServe 对比；与 Istio Ambient 整合
- **C16 多集群与 Karmada**：Hub-Spoke / Federation / GitOps 三种拓扑；Karmada 架构（PropagationPolicy / OverridePolicy / Scheduler / Failover）；Cluster API 集群生命周期；Cilium Cluster Mesh / Submariner / MCS API 多集群网络；Thanos 多集群可观测

---

## 🎯 学习路径建议

### 路径 A：完整通学（3-4 个月）

按编号顺序，每周 1-2 篇。每篇配套：
1. 在本地或测试集群跑 demo
2. 阅读官方文档的对应章节
3. 做练习题
4. 拿一个真实场景把知识点串起来

### 路径 B：应用开发者切入云原生（1-2 个月）

- **C01 Docker**（先把镜像打好）
- **C02 工作负载**（搞清 Deployment / Pod）
- **C03 网络与 Service**（让服务能被访问）
- **C04 Ingress / Gateway**（流量入口）
- **C05 ConfigMap / Secret**（配置注入）
- **C08 Helm**（部署体感）

### 路径 C：平台 / SRE 工程师（2-3 个月）

- **C02 + C03 + C06**（核心工作负载与资源）
- **C09 Operator**（学会自动化）
- **C10 Service Mesh**（流量治理）
- **C11 可观测性**（生产眼睛）
- **C12 安全**（合规）
- **C13 生产调优 + C14 GitOps**
- **C16 多集群**（规模化必经）

### 路径 D：从 K8s 到 Service Mesh 专家（2 个月）

- **C03 网络**（基础）
- **C04 Gateway API**（南北向）
- **C10 Service Mesh**（东西向 + 治理）
- **C11 可观测**（验证）

### 路径 E：Operator / Controller 开发者（1 个月）

- **C09 Operator**（核心）
- **C02 工作负载**（背景）
- **C12 安全（RBAC）**（必备）
- 配合 golang 系列的 G14 context / G27 net/http

### 路径 F：Serverless / 边缘 / 平台工程师（1-2 个月）

- **C02 工作负载**（基础）
- **C06 调度 + KEDA**（事件驱动伸缩）
- **C15 Serverless on K8s**（Knative + KEDA HTTP）
- **C16 多集群**（边缘 / 多 region）
- **C14 GitOps**（多集群部署一体化）

---

## 📋 配套资源

- **Mermaid 路线图**：见 [ROADMAP.md](./ROADMAP.md)
- **测验题与答案**：见 [QUIZ.md](./QUIZ.md)
- **官方文档**：
  - [Kubernetes Docs](https://kubernetes.io/docs/)
  - [Gateway API](https://gateway-api.sigs.k8s.io/)
  - [Istio](https://istio.io/)
  - [Helm](https://helm.sh/)
  - [ArgoCD](https://argo-cd.readthedocs.io/)
- **Go 生态**：
  - [`sigs.k8s.io/controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime)
  - [`k8s.io/client-go`](https://github.com/kubernetes/client-go)
  - [Kubebuilder](https://book.kubebuilder.io/)

---

## 🛠️ 工具速查

| 任务 | 工具 / 命令 |
|---|---|
| 本地集群 | kind / minikube / k3d / Docker Desktop |
| 镜像构建 | `docker buildx` / `podman build` / `nerdctl build` / Bazel |
| 镜像扫描 | trivy / grype / docker scout |
| 镜像签名 | cosign / notation |
| kubectl 增强 | k9s / kubectx / kubens / stern / kubie |
| 集群管理 | kubeadm / Cluster API / kops / eksctl |
| 资源诊断 | `kubectl describe / logs / top / get events` |
| 调试 Pod | `kubectl debug` / ephemeral container |
| 网络抓包 | `kubectl-trace` / cilium hubble / ksniff |
| Helm | `helm template / lint / install --dry-run` |
| Operator 脚手架 | kubebuilder / operator-sdk |
| GitOps | argocd / flux |
| 策略 | OPA Gatekeeper / Kyverno |
| 可观测 | Prometheus / Grafana / Loki / Tempo / Pixie |
| 服务网格 | istioctl / linkerd / cilium |
| 压测 | k6 / hey / vegeta / ghz |
| 备份恢复 | Velero |

---

## ✅ 完读检查清单

读完整个系列后，应该能：

- [ ] 写出最小化的 distroless 镜像（< 20MB）并签名
- [ ] 解释 Deployment 滚动升级的所有步骤（含 readiness gate / maxSurge）
- [ ] 给出 Service 转发到 Pod 的全链路（iptables/IPVS vs eBPF）
- [ ] 用 Gateway API 实现金丝雀 / A-B 流量切分
- [ ] 用 External Secrets + Vault 给 Pod 注入动态密钥
- [ ] 设计 HPA + VPA + KEDA 多策略联动伸缩
- [ ] 为 StatefulSet 选合适的 StorageClass 与回收策略
- [ ] 用 Go + controller-runtime 写一个最小 Operator 并部署
- [ ] 解释 Istio Ambient 的 ztunnel + waypoint 工作模型
- [ ] 排查一个 Pod 的 CPU throttling 问题（指标 / 工具 / 代码）
- [ ] 写 OPA Gatekeeper 或 Kyverno 策略禁止 `:latest` 镜像
- [ ] 设计百节点集群的 NodeLocal DNS / image preload 优化
- [ ] 用 ArgoCD ApplicationSet 把同一个 chart 部署到多环境
- [ ] 把一个低频后台服务部署成 Knative Service，scale-to-zero 后冷启动 < 2s
- [ ] 用 Karmada PropagationPolicy + OverridePolicy 把同一份 Deployment 分发到 3 个集群且每个集群差异化镜像

---

## 🆕 2026 关键变化速查

| 章节 | 2026 必知 |
|---|---|
| **C01 Docker** | OCI Image Spec v1.1（带 referrers）；distroless 流行；Wolfi 兴起；BuildKit 默认；Containerd 2.x 主流 |
| **C02 工作负载** | `apps/v1` 稳定多年；Sidecar Container 1.33 GA；Pod Resource Resize 1.27 alpha→1.33 beta→1.35 GA；Job pod-failure-policy GA |
| **C03 网络** | EndpointSlice 默认；Topology Aware Routing GA；Cilium / Calico eBPF 主导；NetworkPolicy v2 推进 |
| **C04 Gateway API** | **Gateway API v1.0 GA（2023-10）→ 后续版本主流**；逐步替代 Ingress；Istio / Cilium / Envoy Gateway / Kong 都支持 |
| **C06 调度** | KEDA 2.x 事件驱动；Karpenter 跨云普及；In-place Pod Vertical Scaling 临近 GA |
| **C09 Operator** | controller-runtime 0.18+；CRD v1 早 GA；webhook conversion 主流；Operator Hub 海量 |
| **C10 Service Mesh** | **Istio Ambient Mesh GA（2024-11）**；Cilium Service Mesh eBPF 路线；Linkerd 2.16+；Sidecarless 趋势确立 |
| **C11 可观测** | OpenTelemetry CNCF Graduated；Prometheus 3.0；Pixie / Hubble / Coroot eBPF 可观测崛起 |
| **C12 安全** | Pod Security Admission 默认替代 PSP（已删）；Kyverno 与 Gatekeeper 并立；SBOM / SLSA 在企业普及 |
| **C14 GitOps** | ArgoCD 3.x / Flux v2 稳定；Argo Rollouts / Flagger 用于渐进式发布；OCI 仓库存 Helm chart 主流 |
| **C15 Serverless** | Knative CNCF Graduated（2024-03）；KEDA 2.16+ Graduated；KEDA HTTP add-on GA；Cloud Run for K8s 推 Knative 兼容；KServe AI 推理 scale-to-zero 标配 |
| **C16 多集群** | Karmada CNCF Incubation；Cluster API v1.7+；Cilium Cluster Mesh 事实标准；MCS API alpha→beta；KubeFed v2 archive；Liqo 1.0 GA |
