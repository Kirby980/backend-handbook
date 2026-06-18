# 云原生深度课程 · 测验题与答案

> 配合 [INDEX.md](./INDEX.md) 与 [ROADMAP.md](./ROADMAP.md) 使用
>
> **📅 内容基准：2026 年 6 月**——Kubernetes 1.36（2026-04，1.34/1.35 仍广泛在用）、Gateway API v1.x GA、Istio Ambient GA、Cilium 1.16+ eBPF、Helm 3.x、ArgoCD 3.x、Containerd 2.x。

---

## 📖 使用说明

- 每章 **10 道题**，按难度递进：⭐ 概念 → ⭐⭐ 原理 → ⭐⭐⭐ 实战 / 故障排查
- 题型混合：单选、多选、简答、场景题、命令 / YAML 编写
- 答案与详解放在每章末尾的 `<details>` 折叠块里——**请先独立作答，再展开对照**
- 通过标准：**每章 ≥ 7 题正确**；全部通过即可挑战末尾的 **🏆 综合实战题**
- 推荐节奏：读完对应章节后，当天做题；一周后回头复习错题

---

## C01 · 精通 Docker 与 OCI

1. （⭐）OCI 三个核心规范是哪三个？分别管什么？
2. （⭐）`scratch`、`distroless`、`alpine` 三种 base image 有何区别？体积、调试性、glibc 兼容性如何取舍？
3. （⭐⭐）容器底层依赖 Linux 的哪三大内核能力？分别提供什么隔离？
4. （⭐⭐）多阶段构建（multi-stage build）解决了什么问题？写一段最小的 Go 项目 Dockerfile，最终镜像 < 20MB。
5. （⭐⭐）BuildKit 相比传统 `docker build` 的两大核心优势是什么？什么是 cache mount？
6. （⭐⭐）`COPY` 指令的位置为什么会显著影响构建缓存？写出一个"先 COPY go.mod 再 COPY 源码"的反例与正例。
7. （⭐⭐⭐）一个 Go 二进制在 `scratch` 镜像里跑不起来，报 `no such file or directory`。最可能的两个原因是什么？如何修？
8. （⭐⭐⭐）容器内 `ps` 看到的 PID 1 是你的应用，导致 SIGTERM 不被处理。这是为什么？常见的 3 种修法？
9. （⭐⭐⭐）OCI Image Spec v1.1 引入的 **referrers** 机制是干嘛的？跟 cosign 签名怎么联动？
10. （⭐⭐⭐）一台机器同时跑 30 个相同 base image 的容器，磁盘只占了一份 layer；现在你修改了某个容器的 `/etc/passwd`，宿主机上磁盘怎么变化？解释 overlayfs 的 `lowerdir / upperdir / merged`。

<details>
<summary>📝 C01 答案与详解</summary>

1. **Image Spec / Runtime Spec / Distribution Spec**——分别约束镜像格式（manifest + layers）、容器运行时（如何拉起进程 + namespace/cgroup）、Registry HTTP API（push/pull）。
2. **scratch** 空镜像，仅适合静态二进制（如 Go），无 shell 无 libc；**distroless** 由 Google 维护，含 CA、tzdata、可选 glibc，无 shell（除 debug 变体）；**alpine** 基于 musl libc，5MB，含 busybox shell，但 musl 与某些 CGO 库不兼容（DNS / NSS）。
3. **namespace**（pid / net / mnt / ipc / uts / user / cgroup）做视图隔离；**cgroup** 做资源限额（CPU / memory / IO / pids）；**overlayfs / unionfs** 做镜像分层文件系统。
4. 多阶段构建把"构建工具链"和"运行时"分离，最终镜像不带 gcc / go / npm。最小 Go 示例：
   ```dockerfile
   FROM golang:1.23 AS build
   WORKDIR /src
   COPY go.* ./
   RUN go mod download
   COPY . .
   RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /out/app

   FROM gcr.io/distroless/static-debian12:nonroot
   COPY --from=build /out/app /app
   USER nonroot
   ENTRYPOINT ["/app"]
   ```
5. ① **并发与依赖图构建**（DAG）② **可挂载缓存**（`--mount=type=cache,target=/root/.cache/go-build`），让 `go build` 跨构建复用磁盘缓存，速度提升 5–10×。
6. 反例：`COPY . .` 在 `go mod download` 之前——任何源码改动都会让 `go mod download` 重跑。正例：先 `COPY go.mod go.sum` + `RUN go mod download`，再 `COPY . .`，依赖层只在 `go.mod` 变化时失效。
7. ① **二进制是动态链接的**（CGO_ENABLED=1），`scratch` 没有 ld-linux 与 libc——改用 `CGO_ENABLED=0` 或换 distroless/base。② **缺 CA 根证书**导致 TLS 失败被误报为 file not found——从 `alpine` 或 distroless 拷贝 `/etc/ssl/certs/ca-certificates.crt`。
8. PID 1 在 Linux 有特殊语义：不会自动 reap 僵尸子进程；信号默认动作被屏蔽（必须显式 handle）。修法：① 应用层面 `signal.Notify`（Go）；② 用 `tini` / `dumb-init` 当 PID 1；③ K8s 里设置 `shareProcessNamespace: true` 让 kubelet 帮忙。
9. **referrers** 让一个镜像可以挂"附件"（attestations / SBOM / signatures），不再像旧 cosign 那样新建一个 `sha256-xxx.sig` tag 污染 tag 列表。cosign 2.x 默认就是 referrers 模式，Registry 需要支持 OCI 1.1 referrers API。
10. **lowerdir** = 所有镜像层（只读，30 个容器共享）；**upperdir** = 该容器的可写层；**merged** = 用户看到的合并视图。改 `/etc/passwd` 触发 **copy-up**：把 lowerdir 的 `/etc/passwd` 复制到 upperdir 再改，30 个容器各自的 upperdir 独立，宿主机会多出 N 份 `/etc/passwd`。

</details>

---

## C02 · 精通 Kubernetes 工作负载

1. （⭐）Pod 的 `phase` 有哪几种？`Running` 是否等于"健康"？
2. （⭐）Deployment、StatefulSet、DaemonSet 各自的核心使用场景？
3. （⭐⭐）Pod 的 init container、sidecar container（1.33 GA）、普通 container 启动顺序与生命周期有何区别？
4. （⭐⭐）`livenessProbe` 与 `readinessProbe` 的区别？`startupProbe` 解决了什么问题？
5. （⭐⭐）Deployment 滚动升级时，`maxSurge` 与 `maxUnavailable` 分别是什么含义？设 `replicas: 10, maxSurge: 25%, maxUnavailable: 25%`，过程中最多 / 最少有多少个 Pod？
6. （⭐⭐）StatefulSet 的 Pod 名是有序的（`web-0`、`web-1`）。这个顺序在哪些场景下至关重要？删除 `web-1` 会怎样？
7. （⭐⭐⭐）一个 Deployment 的 `terminationGracePeriodSeconds: 30`，但应用 SIGTERM 后还要做 60s 的数据 flush。会发生什么？怎么修？
8. （⭐⭐⭐）Job 的 `pod-failure-policy`（1.31 GA）是干嘛用的？跟 `backoffLimit` 有什么本质区别？
9. （⭐⭐⭐）`In-place Pod Vertical Scaling`（1.27 Alpha → 1.33 Beta → 1.35 GA）允许什么操作？为什么是"近年来 K8s 最重要的资源管理变化之一"？
10. （⭐⭐⭐）有一个 Deployment 升级后所有 Pod 卡在 `ContainerCreating`，`kubectl describe pod` 显示 `Failed to pull image: 429 Too Many Requests`。完整诊断与缓解思路？

<details>
<summary>📝 C02 答案与详解</summary>

1. `Pending / Running / Succeeded / Failed / Unknown`。`Running` 只代表至少一个容器在跑，**不代表 Ready**——Ready 取决于 readinessProbe。Service Endpoints 用的是 Ready，不是 Running。
2. Deployment：无状态、可替换；StatefulSet：稳定网络标识 + 持久存储，如数据库；DaemonSet：每节点一份，用于 log agent / kube-proxy / CNI。
3. Init container 串行运行、全部成功后主容器才启动；sidecar（`restartPolicy: Always` 的 init container，1.33 GA）先于主容器启动、晚于主容器终止，且在主容器运行期间保持运行；普通容器并发启动、终止时收 SIGTERM。
4. liveness 失败 → 重启容器；readiness 失败 → 摘除 Service Endpoint（不重启）；startup 解决"启动慢的应用被 liveness 提前杀掉"——startup 成功前 liveness / readiness 都不生效。
5. maxSurge 控制升级时最多超出 `replicas` 多少个；maxUnavailable 控制最多有多少个不可用。10 + 25% = 最多 12 个；10 − 25% = 最少 7 个 ready。
6. 顺序对**主从复制初始化**（如 MySQL 的 `web-0` 当主、其它做从）、**有序滚动升级**（`podManagementPolicy: OrderedReady`）、**网络标识稳定**（headless service 下的 DNS）至关重要。删 `web-1` 会被 controller 立即用同名 Pod 重建，PVC 也复用，所以**只是重启**而非数据丢失。
7. kubelet 发 SIGTERM 后等 30s 强制 SIGKILL，60s 的 flush 被截断 → 数据损坏。修法：① 把 `terminationGracePeriodSeconds` 调到 90+；② 使用 `preStop` hook 提前触发 flush；③ 调整应用让 flush 异步并幂等。
8. `backoffLimit` 只数失败次数，不区分原因；`pod-failure-policy` 可以按**退出码 / 信号 / DisruptionTarget**条件，决定"忽略 / 失败计数 / 直接终止 Job"。例如：节点抢占（DisruptionTarget）不计入失败，OOM（exit 137）直接放弃。
9. 允许**不重启容器**修改 `resources.requests/limits`。意义：原来调 limits 必须重建 Pod，对有状态服务（DB / 缓存）极不友好；in-place（1.33 Beta、1.35 GA）配合 VPA 可以做真正的"运行时垂直扩缩"。
10. 原因：节点 kubelet 同时从 Docker Hub 拉镜像被限流。缓解：① 用私有 registry mirror（Harbor / ECR / 阿里云 ACR）；② 在节点预热镜像（image preload，C13 详述）；③ 配置 `imagePullPolicy: IfNotPresent`；④ 多副本镜像 secret 轮询；⑤ 长远——OCI registry 内网部署 + pull-through cache。

</details>

---

## C03 · 精通 K8s 网络与 Service

1. （⭐）ClusterIP / NodePort / LoadBalancer / ExternalName 四种 Service 各自的转发路径？
2. （⭐）Pod IP 是哪个组件分配的？跨节点 Pod 通信靠什么？
3. （⭐⭐）Endpoint 和 EndpointSlice 的区别？为什么 1.21+ 默认是 EndpointSlice？
4. （⭐⭐）kube-proxy 的 iptables 模式与 IPVS 模式有何性能差异？什么规模时建议切到 IPVS？
5. （⭐⭐）Headless Service（`clusterIP: None`）跟普通 ClusterIP 的最大区别？StatefulSet 为什么默认配 headless？
6. （⭐⭐⭐）一个 Pod `curl` 同 namespace 的 Service `my-svc` 失败，但 `curl <pod-ip>` 成功。给出至少 4 个可能原因。
7. （⭐⭐⭐）CoreDNS 的 `ndots:5` 默认配置带来什么副作用？为什么大规模集群要调小？
8. （⭐⭐⭐）`Topology Aware Routing`（原 Topology Aware Hints）解决什么问题？开启后流量会优先去哪里？
9. （⭐⭐⭐）Cilium / Calico 的 eBPF datapath 相比传统 iptables 有哪些根本性优势？
10. （⭐⭐⭐）描述一个数据包从 client Pod 到 Service 后端 Pod 的**完整链路**（含 DNS → kube-proxy/eBPF → CNI → 目标 Pod）。

<details>
<summary>📝 C03 答案与详解</summary>

1. ClusterIP：集群内虚拟 IP，由 kube-proxy / eBPF DNAT 到 Pod；NodePort：每个节点开端口；LoadBalancer：云厂商分配外部 LB；ExternalName：CoreDNS 返回 CNAME，不分配 IP。
2. **CNI 插件**（Calico / Cilium / flannel 等）分配，写入 IPAM。跨节点：overlay（VXLAN / IP-in-IP）或 underlay（BGP / 路由）。
3. Endpoint 单个对象列所有后端 → 大集群下大 Service 的更新事件爆炸；EndpointSlice 分片（默认 100/片），更新增量化，watcher 压力降一个量级。
4. iptables 规则线性匹配，O(N) 复杂度，几千条规则后延迟肉眼可见；IPVS 是内核哈希表，O(1)，并支持多种负载均衡算法（rr / lc / sh）。Service 数量 > ~1000 或后端 Pod > ~5000 时建议切 IPVS。
5. Headless Service 不分配 ClusterIP，DNS 直接返回所有 Endpoint 的 A 记录列表（或 SRV）。StatefulSet 用它实现 `web-0.web.default.svc.cluster.local` 的稳定 DNS。
6. ① Service selector 与 Pod label 不匹配 → Endpoints 空；② DNS 没解析（CoreDNS 挂、search domain 错）；③ NetworkPolicy 拦了 Service VIP 但放了 Pod IP（典型坑——Cilium / Calico 行为差异）；④ kube-proxy 挂导致 iptables/IPVS 规则没刷新；⑤ Service 的 `port` 与 `targetPort` 配错；⑥ 后端 Pod 没 Ready（readiness 失败）。
7. `ndots:5` 意味着只要 hostname 里 `.` 少于 5 个就走 search domain，导致每次外网查询多 4-5 次 NXDOMAIN 往返。NodeLocal DNSCache + `ndots:2` 是常见优化。
8. 同区域流量本地化——给 EndpointSlice 加 `hints.forZones`，kube-proxy / Cilium 优先选同 zone 的后端，减少跨 AZ 流量费 + 延迟。
9. ① 绕过 conntrack 与 iptables（减少 PPS 路径）；② Service / NAT 在 socket 层而非 packet 层（`sockops`），同节点 0-copy；③ 可观测性原生（Hubble flow log）；④ NetworkPolicy / L7 策略统一框架。
10. **DNS**：`my-svc` → CoreDNS → ClusterIP；**SNAT/DNAT**：kube-proxy 写的 iptables（OUTPUT/PREROUTING）把 ClusterIP DNAT 到某个 Endpoint；**CNI**：源 Pod → veth → 节点 root netns → overlay/underlay → 目标节点 → veth → 目标 Pod；**回包**：conntrack 反向解 NAT 回到客户端。

</details>

---

## C04 · 精通 Ingress 与 Gateway API

1. （⭐）Ingress 与 Gateway API 最大的设计差异是什么？
2. （⭐）Gateway API 的三大核心资源 `GatewayClass / Gateway / HTTPRoute` 分别由谁创建、谁负责实现？
3. （⭐⭐）从 Ingress 迁到 Gateway API 的核心动机有哪些？至少列 3 点。
4. （⭐⭐）`HTTPRoute` 里 `parentRefs` 的作用？一个 HTTPRoute 能 attach 到多个 Gateway 吗？
5. （⭐⭐）`TLSRoute` 跟 `HTTPRoute` + `tls.passthrough` 有什么区别？什么场景必须用 TLSRoute？
6. （⭐⭐）写一段 HTTPRoute YAML，把 `/api/v2/*` 90% 流量发到 `backend-v2`，10% 发到 `backend-v3`。
7. （⭐⭐⭐）多团队多 namespace 共享一个 Gateway 时，如何用 `ReferenceGrant` 做跨 namespace 授权？
8. （⭐⭐⭐）Envoy Gateway / Istio Gateway / Cilium Gateway / Kong 四种 Gateway 实现的核心差异？选型时主要看哪些维度？
9. （⭐⭐⭐）一个 Gateway 配了 HTTPS 监听，客户端报 SNI mismatch。可能的原因有哪些？
10. （⭐⭐⭐）灰度发布场景下，`HTTPRoute` 的 header-based routing 和 weighted routing 各自适合什么？为什么大型业务通常先 header（内部测）再 weighted（线上灰度）？

<details>
<summary>📝 C04 答案与详解</summary>

1. Ingress：单一资源、扩展靠厂商 annotation，缺乏 L4 / 多协议支持；Gateway API：**角色分离 + 多资源 + 厂商无关扩展**（GatewayClass 由平台方提供，Gateway 由集群管理员配，Route 由应用团队配）。
2. GatewayClass：由 Controller 厂商发布（如 `envoy-gateway`、`istio`）；Gateway：集群管理员创建，绑定 GatewayClass；HTTPRoute / TLSRoute：应用方创建，通过 `parentRefs` 挂到 Gateway。
3. ① 多协议（HTTPS / gRPC / TCP / TLS passthrough）原生支持；② 流量切分 / 权重 / header match 标准化（不再依赖 annotation）；③ 角色权限分离（RBAC 可以让应用团队只能改 Route 不能改 Listener）；④ 跨 namespace 用 ReferenceGrant 受控开放。
4. `parentRefs` 指定挂到哪个 Gateway 的哪个 Listener。可以挂多个——同一份 HTTPRoute 可以同时挂到 `prod-gateway` 和 `staging-gateway`。
5. TLSRoute 做 **SNI-based 透传**（不解密、不读 HTTP），适合需要 mTLS 直达后端的场景（如 gRPC 内部服务、数据库），HTTPRoute 默认终结 TLS。
6. ```yaml
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   spec:
     parentRefs: [{name: my-gateway}]
     rules:
     - matches:
       - path: {type: PathPrefix, value: /api/v2}
       backendRefs:
       - {name: backend-v2, port: 80, weight: 90}
       - {name: backend-v3, port: 80, weight: 10}
   ```
7. 默认 Gateway 只接受**同 namespace** 的 Route。要跨 ns，需要 Route 所在 ns 创建一个 `ReferenceGrant`，明确允许 Gateway 所在 ns 的 Gateway 引用本 ns 的 Route——这是显式授权而非通配。
8. Envoy Gateway：CNCF 主推、Envoy 原生；Istio Gateway：与 Service Mesh 一体；Cilium Gateway：eBPF datapath、性能优；Kong：商业生态丰富、插件多。选型维度：是否已用 mesh、性能 SLA、L7 策略复杂度、企业支持。
9. ① 证书 SAN/CN 与请求 SNI 不匹配；② 多证书时 Gateway 配的 `hostname` 与 listener 不对齐；③ TLS termination 模式选错（passthrough vs terminate）；④ 客户端不发 SNI（老 Java）。
10. **header**：精准——某个 cookie / X-Canary header 命中才走新版本，可以让 QA 内部强制流量；**weighted**：粗放——按百分比放量，无法定向。先 header 灰度内部用户做功能验证，再 weighted 慢慢扩量到全量。

</details>

---

## C05 · 精通 ConfigMap、Secret 与配置

1. （⭐）12-Factor 第 3 条"Config"的核心主张是什么？
2. （⭐）`ConfigMap` 用 `volumeMount` 挂载 vs 用 `envFrom` 注入，行为有何不同？哪种支持热更新？
3. （⭐⭐）ConfigMap 挂载 + `subPath` 的组合有什么大坑？
4. （⭐⭐）Secret 在 etcd 里是 base64 编码还是加密的？如何启用 etcd 静态加密？
5. （⭐⭐）`Reloader` / `Stakater Reloader` 这类工具解决什么问题？为什么 K8s 原生不做？
6. （⭐⭐⭐）External Secrets Operator（ESO）相比 sealed-secrets，定位有何不同？哪个适合多云？
7. （⭐⭐⭐）Vault 与 K8s 集成时，`agent injector` 与 `CSI Secret Driver` 两种模式各自的优缺点？
8. （⭐⭐⭐）什么是 SPIFFE / SPIRE？它跟 K8s 的 ServiceAccount Token 有什么本质差异？
9. （⭐⭐⭐）一个团队所有应用配置都塞进环境变量，启动越来越慢，`exec` 进容器 `env` 输出 2MB。问题在哪？怎么治？
10. （⭐⭐⭐）写出一份完整的 Pod 例子：从 Vault 动态获取 DB 密码（30 分钟轮换），通过 CSI Secret Driver 注入到 `/secrets/db-pass`，并配合 `Reloader` 在配置变更时滚动重启。

<details>
<summary>📝 C05 答案与详解</summary>

1. **代码与配置分离**——配置随环境变（dev/staging/prod），代码不变；同一个镜像通过外部配置在多环境运行。
2. volumeMount 支持热更新（kubelet 定期 syncLoop 更新文件，约 60s 内生效）；envFrom 在容器启动时一次性注入，**不会**热更新。
3. `subPath` 把 ConfigMap 的一个 key 挂为单文件——但 kubelet 的热更新机制依赖**目录级符号链接切换**，subPath 绕过了这个机制，**所以 subPath 挂载不会热更新**。要么挂目录，要么配合 Reloader 重启。
4. **仅 base64 编码，未加密**。启用 etcd 静态加密：API server 加 `--encryption-provider-config`，配置 `aescbc / kms` provider，重启后所有 Secret 写回都会被加密；存量 Secret 需要 `kubectl get secrets -o yaml | kubectl replace -f -` 触发回写。
5. ConfigMap 挂载到 envFrom 不热更新；即使 volumeMount 热更新，应用通常也不会自己 reload。Reloader 监听 ConfigMap/Secret 变化，自动给 Deployment 加 annotation 触发滚动重启。K8s 不做是因为：**这是应用层关注点**（重启策略由业务决定），K8s 只保证文件最终一致。
6. sealed-secrets：把 Secret 加密成 SealedSecret CRD 存 Git，集群内 controller 解密；适合**单集群、Git 即真理源**。ESO：从外部 SecretsManager（AWS / Azure / GCP / Vault / HashiCorp）拉密钥同步成 K8s Secret；适合**多云 + 已有企业级密钥管理**。
7. **agent injector**：sidecar 容器持续渲染 secret 到共享 volume，**支持动态轮换**、应用零侵入；缺点：每 Pod 多一个容器，资源开销。**CSI Driver**：通过 CSI volume 挂载，**无 sidecar**；缺点：轮换语义依赖 driver 实现，老版本不支持。
8. SPIFFE = 一套**工作负载身份**标准（SPIFFE ID `spiffe://trust-domain/workload`），SPIRE 是实现。区别：ServiceAccount Token 是 K8s 内的、无法跨集群跨平台；SPIFFE 是跨平台的，是 mesh / zero-trust 的基础。
9. 启动时 envp 数组要全量加载到进程内存；某些 libc/runtime 还要 dup / scan。2MB env 会让 exec 路径慢、调试更难。治理：拆成 ConfigMap volumeMount 文件、按需读取；非启动期配置改用 ConfigMap reload / 远程配置中心（Apollo / Nacos / Consul）。
10. （略写关键点）`SecretProviderClass`（CSI）指向 Vault role，volume `vault-secrets` 挂到 `/secrets`，Deployment 加 annotation `reloader.stakater.com/auto: "true"`；轮换由 Vault 控制 TTL，文件变了 Reloader 触发滚动。

</details>

---

## C06 · 精通 Scheduling 与资源管理

1. （⭐）Pod 的 QoS 三个等级（Guaranteed / Burstable / BestEffort）由什么决定？OOMKill 顺序？
2. （⭐）`requests` 和 `limits` 各自影响调度还是运行时？
3. （⭐⭐）CPU `limits: 1` 与 `requests: 1` 在 cgroup 上分别变成什么？为什么 `limits` 会带来 throttling？
4. （⭐⭐）HPA 默认基于哪个指标？想用 QPS / 消息队列长度做扩缩，应该用什么？
5. （⭐⭐）VPA 与 HPA 同时启用有什么坑？怎么避免冲突？
6. （⭐⭐）KEDA 跟 HPA 的关系？为什么说 KEDA 是 "HPA 的 metric 来源扩展"？
7. （⭐⭐⭐）`nodeAffinity` / `podAffinity` / `topologySpreadConstraints` 三者用途差异？写一段保证 Pod 跨 3 个 AZ 均匀分布的约束。
8. （⭐⭐⭐）`PriorityClass` + `Preemption` 工作机制？被抢占的 Pod 如何 graceful 退出？
9. （⭐⭐⭐）线上一个 Java Pod CPU `usage` 永远不超 `requests` 的 50%，但应用响应慢且看到大量 `nr_throttled`。怎么回事？怎么修？
10. （⭐⭐⭐）一个 100 节点集群，3 个 GPU 节点，要保证某个推理服务**优先调度到 GPU 节点**，但 GPU 不够时**降级到 CPU 节点**。怎么实现？

<details>
<summary>📝 C06 答案与详解</summary>

1. Guaranteed：每个容器的 requests = limits 且都 > 0；Burstable：至少有一个容器设置了 requests/limits 但不全相等；BestEffort：完全没设。OOMKill 顺序：BestEffort → Burstable → Guaranteed。
2. requests 影响**调度**（scheduler 看节点可分配容量）；limits 影响**运行时**（kubelet/cgroup 强制约束）。
3. CPU requests = `cpu.shares`（相对权重），limits = `cpu.cfs_quota_us / cfs_period_us`（绝对配额）。limits 触发 throttling 因为 CFS 在 100ms 周期内用完配额就让出 CPU——Java / Go 的 GC、Goroutine 调度遇到 throttling 会出现 P99 长尾。
4. 默认 CPU。需要自定义指标：metric-server 之外加 `prometheus-adapter` 或 `KEDA`，HPA `type: Pods / Object / External`。
5. VPA 调 requests 会触发 Pod 重建（除非用 in-place）；HPA 同时基于 CPU 做扩缩会震荡。避免：① VPA 用 `Off` / `Initial` 模式，不要 `Auto`；② HPA 用 CPU 时，VPA 不要管 CPU；③ in-place vertical scaling GA 后会缓解。
6. KEDA 内部就是个 ScaledObject Controller，它会生成对应的 HPA（带 External metric provider）。所以 KEDA = HPA + 各种事件源 adapter（Kafka / RabbitMQ / Redis / Prometheus / Cron / Azure Queue 等）。
7. nodeAffinity：Pod 与节点的关系；podAffinity / antiAffinity：Pod 与 Pod 的关系；topologySpreadConstraints：跨拓扑域均匀。AZ 均匀示例：
   ```yaml
   topologySpreadConstraints:
   - maxSkew: 1
     topologyKey: topology.kubernetes.io/zone
     whenUnsatisfiable: DoNotSchedule
     labelSelector: {matchLabels: {app: web}}
   ```
8. PriorityClass 给 Pod 优先级整数；调度不下时高优先级 Pod 抢占低优先级，scheduler 标记低优 Pod 待删除，kubelet 发 SIGTERM 走 `terminationGracePeriodSeconds`；可配 `PreemptionPolicy: Never` 关掉抢占只做排队。
9. CPU **平均**利用率低不代表瞬时不超 limit——Java GC / 短时高并发会瞬时打满 limit 触发 throttling。修法：① 调高 CPU limits 或干脆**取消 CPU limits**（社区争论已久的"只设 requests"派）；② 用 in-place resize 临时拉高；③ 监控 `container_cpu_cfs_throttled_periods_total` 而不是只看 usage。
10. nodeAffinity `preferredDuringSchedulingIgnoredDuringExecution`（软约束）打到 GPU 节点；不要用 `required`。GPU 不足时 scheduler 自动降级到 CPU 节点。镜像内 runtime 自适应（`if GPU available` 走 CUDA，否则走 CPU 推理）。

</details>

---

## C07 · 精通 Storage 与 PVC

1. （⭐）PV / PVC / StorageClass 三者关系？谁创建谁、谁绑定谁？
2. （⭐）`ReadWriteOnce` / `ReadOnlyMany` / `ReadWriteMany` / `ReadWriteOncePod` 各自含义？哪些后端支持 RWX？
3. （⭐⭐）StorageClass 的 `reclaimPolicy: Delete` 与 `Retain` 的区别？什么场景必须用 Retain？
4. （⭐⭐）CSI 协议主要解决什么问题？相比 in-tree volume plugin 的优势？
5. （⭐⭐）`volumeBindingMode: WaitForFirstConsumer` 的意义？不开会有什么坑？
6. （⭐⭐⭐）StatefulSet 的 PVC 生命周期：StatefulSet 删除时 PVC 会不会被删？升级时 PVC 会不会丢？
7. （⭐⭐⭐）本地盘（local PV）与网络盘（EBS / Ceph）的 trade-off？哪些场景必须用 local？
8. （⭐⭐⭐）`VolumeSnapshot` / `VolumeSnapshotClass` 与 Velero 备份的关系？Velero 备份 PV 时底层走的是什么？
9. （⭐⭐⭐）一个 Postgres Pod 重启后数据丢了，PVC 还在。可能原因？
10. （⭐⭐⭐）描述生产 K8s 上跑 MySQL 主从的存储方案选型：StorageClass 类型、reclaimPolicy、备份、扩容、故障恢复链路。

<details>
<summary>📝 C07 答案与详解</summary>

1. StorageClass 是模板（描述"如何动态创建 PV"），PVC 是申请书（要多少、什么 class），PV 是真实卷。动态供给：PVC → StorageClass → CSI Driver 创建 PV → PVC 绑 PV。
2. RWO：单节点读写；ROX：多节点只读；RWX：多节点读写；RWOP（1.29 GA）：单 Pod 读写（比 RWO 更严格）。RWX 支持：NFS、CephFS、GlusterFS、EFS 等共享文件系统；块设备（EBS / RBD）只能 RWO。
3. Delete：PVC 删除时 PV 与底层卷一起删；Retain：保留 PV，需手动清理。**数据库、金融数据必须 Retain**——防止 PVC 误删导致数据消失。
4. CSI 让存储厂商作为**独立 Pod 部署**，不再需要把代码合到 kubelet。优势：① 解耦版本升级；② 厂商生态独立迭代；③ snapshot / clone / expand 标准化。
5. WaitForFirstConsumer：PVC 创建时**不立即**分配 PV，等到 Pod 调度后再分配——保证 PV 与 Pod 在同一 zone。不开的坑：AWS EBS 跨 AZ 不能挂载，PV 提前分配在 zone A，Pod 被调到 zone B，永远 Pending。
6. StatefulSet 删除**默认不删 PVC**（保护数据）；1.27+ 引入 `persistentVolumeClaimRetentionPolicy` 可配 `WhenDeleted: Delete`。升级时 PVC 完全保留，新 Pod 复用同名 PVC。
7. local PV：低延迟、高 IOPS、低成本；但**节点损坏数据可能丢**、不能跨节点迁移。场景：ClickHouse / Kafka / TiKV / Ceph OSD 自己做副本，节点死了从其他副本恢复。
8. VolumeSnapshot 标准化的快照 API，依赖 CSI driver 的 snapshot 能力（底层调云厂商 EBS snapshot 或 Ceph snapshot）。Velero 备份 PV 默认通过 CSI snapshot；老版本 / 不支持 CSI snapshot 时走 restic / kopia 文件级备份。
9. ① emptyDir 而不是 PVC（YAML 写错）；② subPath 挂错了路径；③ 应用启动时清空数据目录（迁移脚本 bug）；④ PVC 绑了空的新 PV（StorageClass 误改）；⑤ 容器内 `/var/lib/postgresql/data` 跟 PVC 挂载点不一致。
10. **方案**：StorageClass 用云盘 SSD（gp3 / Premium SSD）、reclaimPolicy=Retain、volumeBindingMode=WaitForFirstConsumer；备份用 VolumeSnapshot + Velero 异地复制；扩容用 `allowVolumeExpansion: true`；故障恢复：主死 → operator (如 Zalando postgres-operator) 提升 standby → 重建主从。

</details>

---

## C08 · 精通 Helm 与 Kustomize

1. （⭐）Helm 与 Kustomize 的本质差异？
2. （⭐）Helm Chart 的目录结构里 `templates/`、`values.yaml`、`Chart.yaml` 各自的角色？
3. （⭐⭐）`helm template` 与 `helm install` 的区别？什么时候用 `helm template | kubectl apply`？
4. （⭐⭐）Helm Hook（`pre-install` / `post-upgrade`）的工作原理？做数据库 migration 适不适合？
5. （⭐⭐）Kustomize overlay 的 `patches` / `patchesStrategicMerge` / `patchesJson6902` 区别？
6. （⭐⭐⭐）Helm 升级失败后，`helm rollback` 能完全还原吗？哪些资源（如 PVC、CRD）会出问题？
7. （⭐⭐⭐）一个 chart 既要支持 Helm 又要支持 Kustomize 的用户，怎么设计？（提示：post-renderer）
8. （⭐⭐⭐）`Helmfile` / `Helm Umbrella Chart` / `ArgoCD ApplicationSet` 都能做"多 chart 编排"，差异是什么？
9. （⭐⭐⭐）chart 里如何安全地管理 secrets？为什么 `--set secret.password=xxx` 是反模式？
10. （⭐⭐⭐）写一个 Kustomize 例子：base 是通用 Deployment，prod overlay 增加 3 副本、加 nodeAffinity、把 image tag 改成 v2.0。

<details>
<summary>📝 C08 答案与详解</summary>

1. Helm = 模板引擎 + 包管理 + Release 生命周期（state in cluster）；Kustomize = 纯声明式 overlay，无模板、无运行时状态。
2. `Chart.yaml` 元信息（name、version、appVersion）；`values.yaml` 默认值；`templates/` Go template，渲染后生成 YAML。
3. `helm template` 纯渲染输出 YAML，不连集群、不存 release；`helm install` 渲染 + apply + 在 cluster 存 Secret 记录 release 历史。`helm template | kubectl apply` 适合 GitOps（ArgoCD 内部就是这么做的）——不需要 release 状态。
4. Hook 给资源加 `helm.sh/hook` annotation，按 weight 顺序执行（pre-install Job → install Service/Deployment → post-install Job）。**数据库 migration 适合**——用 Job 做 pre-upgrade hook，失败则升级中止。
5. `patchesStrategicMerge`：K8s 战略合并（按字段类型决定 replace/merge）；`patchesJson6902`：JSON Patch（精准操作如 `replace` `add` 指定路径）；`patches`（新统一字段）支持 strategic 或 json6902。
6. 资源会回滚到上一版本，但**PVC 不会**（生产正确）、**CRD 与 namespace 默认不删**、**Job 失败的 hook 资源也可能残留**。慎用——常见做法是固定版本前向兼容。
7. 提供原生 chart，让用户用 `helm template ... | kubectl apply -k <kustomize-overlay>` 模式；或用 Helm 的 **post-renderer** 把 Kustomize 作为后处理器。
8. Helmfile：YAML 声明多个 release，CLI 工具；Umbrella chart：一个 chart 依赖多个 subchart，**强耦合**升级；ApplicationSet：ArgoCD CRD，按 generator（cluster / git / list）批量生成 Application，**多集群**首选。
9. `--set` 进程参数会留 shell history、CI 日志；正确做法：`--values secrets.yaml`（用 sealed-secrets / SOPS / ESO 加密）或集成 Vault。
10. base/`deployment.yaml` 标准 Deployment，overlays/prod/`kustomization.yaml`：
    ```yaml
    resources: [../../base]
    replicas: [{name: web, count: 3}]
    images: [{name: web, newTag: v2.0}]
    patches:
    - target: {kind: Deployment, name: web}
      patch: |
        - op: add
          path: /spec/template/spec/affinity
          value: {...}
    ```

</details>

---

## C09 · 精通 Operator 与 CRD

1. （⭐）CRD 是什么？为什么说它是 "K8s 可扩展的核心"？
2. （⭐）`Controller` 与 `Operator` 的区别？Operator 一定是 controller 吗？
3. （⭐⭐）Reconcile 循环的核心思想（desired vs actual）？为什么"幂等"是 Reconcile 的第一原则？
4. （⭐⭐）controller-runtime 的 `Manager` / `Cache` / `Client` / `EventRecorder` 各自职责？
5. （⭐⭐）Leader Election 解决什么问题？两个副本同时 Reconcile 会怎样？
6. （⭐⭐⭐）`status` 子资源（subresource）跟 `spec` 在 RBAC 上为何要分开？什么是 "spec/status split"？
7. （⭐⭐⭐）CRD `webhook conversion` 怎么工作？升级 v1alpha1 → v1beta1 → v1 时，集群里旧对象怎么办？
8. （⭐⭐⭐）一个 Operator Reconcile 时 `client.Get` 拿到的对象很可能是**过期的缓存**。这会带来什么后果？应该如何写代码？
9. （⭐⭐⭐）`finalizer` 是什么？写 Operator 处理外部资源（如云 LB）时为什么必须用？
10. （⭐⭐⭐）用 Kubebuilder 4 写一个最小 CRD `Database`，包含 `spec.size` 与 `status.phase`，并在 Reconcile 中创建对应的 StatefulSet。给出关键代码骨架。

<details>
<summary>📝 C09 答案与详解</summary>

1. CRD 让用户**定义新的 K8s 资源类型**，由 API server 持久化、列表、watch，所有 K8s 工具链（kubectl / RBAC / admission）天然支持。Operator / Argo / Tekton / Istio / Cert-Manager 都是建立在 CRD 上的。
2. Controller 是**通用模式**——watch + reconcile；Operator 是 **"领域特定 Controller"**——把"运维 SRE 操作"编码（备份、升级、扩容）。Operator 通常是 Controller，但 Controller 不一定是 Operator（如 kube-controller-manager 内的 ReplicaSet controller）。
3. Reconcile：从当前状态趋近期望状态，**不假设上次执行成功**——所以每次都重新 List + Diff + Apply。幂等保证多次执行结果相同——网络抖动、leader 切换、controller 重启都不能破坏正确性。
4. Manager：进程级控制器集合管理；Cache：基于 informer 的本地缓存（watch event 同步）；Client：对 API 的 Get/List/Create/Update 抽象（默认 Get/List 走 cache，Update/Delete 走 API）；EventRecorder：写 Pod Event 给用户看。
5. Operator 通常多副本部署做 HA，但 Reconcile 同时执行会导致**重复创建子资源 / 状态冲突**。Leader Election 用 Lease 资源选一个活跃实例，其它只待命。
6. status 由 Controller 自己写，**不能由用户改**；spec 由用户改。RBAC 区分：用户有 `update CRD` 但不能 `update CRD/status`；Controller 反之。这样 status 不会被人工乱改，spec 不会被 Controller 自己改回去。
7. CRD 多版本时，etcd 只存一个 "storage version"。读不同版本时 API Server 调 webhook 把对象在版本间转换。升级路径：v1alpha1 写入 → conversion webhook 转 v1beta1 存 → 客户端读 v1 时再转。
8. 后果：基于过期 status 做决策 → 重复创建子资源、写冲突、status 反复跳。写法：① Reconcile 末尾 `return ctrl.Result{Requeue: true}` 等下次；② Update 用 `UpdateStatus` 配合 resourceVersion 乐观锁；③ 关键操作直接调 API 而不是 cache。
9. finalizer = `metadata.finalizers` 列表里的字符串。对象被 delete 时，API server 只置 `deletionTimestamp`，**等所有 finalizer 被去掉才真正删**。这样 Operator 有机会**清理外部资源**（云 LB、DNS、对象存储桶）后再放行删除。
10. （骨架）
    ```go
    type DatabaseSpec struct { Size int32 `json:"size"` }
    type DatabaseStatus struct { Phase string `json:"phase"` }

    func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
      var db v1.Database
      if err := r.Get(ctx, req.NamespacedName, &db); err != nil { return ctrl.Result{}, client.IgnoreNotFound(err) }
      // build desired STS
      sts := buildStatefulSet(&db)
      if err := controllerutil.SetControllerReference(&db, sts, r.Scheme); err != nil { return ctrl.Result{}, err }
      if err := r.Patch(ctx, sts, client.Apply, client.FieldOwner("db-operator")); err != nil { return ctrl.Result{}, err }
      db.Status.Phase = "Ready"
      return ctrl.Result{}, r.Status().Update(ctx, &db)
    }
    ```

</details>

---

## C10 · 精通 Service Mesh

1. （⭐）Service Mesh 解决什么问题？为什么不能用 Ingress 替代？
2. （⭐）Sidecar 模式与 Ambient 模式的最大区别？
3. （⭐⭐）Istio Ambient 的 `ztunnel` 与 `waypoint` 分别承担什么角色？
4. （⭐⭐）mTLS 在 mesh 里实现"零信任"的两个核心要素是什么？
5. （⭐⭐）Linkerd 相比 Istio 更轻量，主要差异在哪？
6. （⭐⭐）Cilium Service Mesh（基于 eBPF）相比传统 sidecar mesh，性能优势的本质？
7. （⭐⭐⭐）一个 sidecar 模式的应用容器 SIGTERM 后挂在 30s 不退出。最可能的原因？
8. （⭐⭐⭐）`VirtualService` / `DestinationRule` 在 Ambient 模式下还能用吗？怎么演化？
9. （⭐⭐⭐）流量切分（10% canary）用 mesh 实现 vs 用 Gateway API 实现，各自适合什么场景？
10. （⭐⭐⭐）描述一次跨集群 mTLS 流量打通的关键步骤（Trust domain、root CA、east-west gateway）。

<details>
<summary>📝 C10 答案与详解</summary>

1. mesh 管**东西向流量**（service-to-service）：mTLS、重试、超时、熔断、流量切分、L7 可观测；Ingress / Gateway 只管**南北向**入口流量。
2. Sidecar：每 Pod 注 Envoy，应用 + sidecar 跑同 netns；Ambient：用节点级 ztunnel 接管 L4，按需用 waypoint 做 L7——**Pod 不再 2/2**。
3. ztunnel：节点级 DaemonSet，做 mTLS + L4 转发（HBONE 隧道）；waypoint：按需部署的 L7 代理（Envoy），只有需要 L7 治理（重试、路由、JWT 验证）的 namespace/service 才用。
4. ① 工作负载身份（SPIFFE ID）+ 证书；② 双向证书校验（client 验 server，server 也验 client）。任何一边没证书 / 证书过期都拒绝。
5. Linkerd 用 Rust 写的轻量代理（linkerd2-proxy），不是 Envoy；功能集少（无 WASM、L7 治理较弱），但启动快、资源占用低、运维简单——"够用主义"。
6. eBPF 在内核态做 socket-level 重定向，**同节点通信无需经过 sidecar**（零拷贝 sockmap）；跨节点也比 sidecar 少一次 user/kernel 切换。
7. sidecar 先于应用退出，导致应用最后的请求（如 graceful shutdown 时往外部发的）打不出去，反复重试到超时。解法：① `holdApplicationUntilProxyStarts: true` 启动顺序；② `EXIT_ON_ZERO_ACTIVE_CONNECTIONS=true` 让 Envoy 等连接清空；③ Istio 1.24+ 用 `nativeSidecar`（K8s sidecar container）让 K8s 管顺序；④ Ambient 模式直接没这问题。
8. 旧的 `VirtualService` / `DestinationRule` 在 Ambient 通过 waypoint 仍兼容；社区推动用 **Gateway API + Mesh extension** (`GAMMA` 工作组) 统一 API，长期趋势是替代。
9. mesh：精细到 service 级、按 header / weight 切分，应用对客户端不变；Gateway API：在入口处切分，简单且支持任何下游协议——对**南北向单点入口**最方便。混合用：入口 Gateway 做粗切，mesh 做内部精细路由。
10. ① 两集群同一 trust domain 或互信 root CA；② 每集群部署 east-west gateway（专用 LoadBalancer Service，监听 15443）；③ 对端集群 secret 注册（提供 KubeConfig 给本地 istiod）；④ DNS 解析 `*.global` 跨集群名字；⑤ 走 SNI 路由让流量打通。

</details>

---

## C11 · 精通 K8s 可观测性

1. （⭐）K8s 可观测的"三大支柱"是什么？
2. （⭐）`kube-state-metrics` 与 `metrics-server` 分别提供什么指标？为什么需要两个？
3. （⭐⭐）cAdvisor 在哪个组件里？它的指标从哪里来？
4. （⭐⭐）Prometheus 监控 K8s 的 ServiceMonitor 跟传统 scrape config 的关系？
5. （⭐⭐）Loki 与 ELK 在日志方案上的核心差异？为什么 Loki 更适合 K8s？
6. （⭐⭐）OpenTelemetry 的 collector 怎么部署？sidecar / DaemonSet / Gateway 三种模式各适合什么？
7. （⭐⭐⭐）排查一个 Pod CPU 飙高，但 `kubectl top pod` 显示正常。可能原因？
8. （⭐⭐⭐）Hubble / Pixie / Coroot 等基于 eBPF 的可观测工具，相比传统 sidecar / agent 有什么本质优势？
9. （⭐⭐⭐）一个 Service 偶发 P99 延迟尖刺。给出一份"全链路定位 checklist"（指标 / 日志 / trace / eBPF）。
10. （⭐⭐⭐）Prometheus 单机性能瓶颈在哪？千万级 series 时如何扩展（Thanos / Mimir / VictoriaMetrics）？

<details>
<summary>📝 C11 答案与详解</summary>

1. **Metrics / Logs / Traces**（指标 / 日志 / 链路）；近年加上 **Profiles**（持续性能采样）变四支柱。
2. **kube-state-metrics**：K8s API 对象状态（Deployment 副本数、Pod phase、PVC 状态）—— "K8s 资源的状态"；**metrics-server**：Pod / Node 的实际 CPU / 内存使用 ——"workload 资源消耗"。两者维度完全不同。
3. cAdvisor 已经**集成在 kubelet** 里；指标采自 Linux cgroup、namespace 接口。
4. ServiceMonitor / PodMonitor 是 Prometheus Operator 的 CRD，**自动生成** Prometheus scrape config。运行时 Operator 渲染配置注入 Prometheus Pod，对用户屏蔽了 scrape config 编写。
5. Loki 索引**只索引标签**（label），日志内容压缩存对象存储（S3）；ELK 索引全文。Loki 成本低 1-2 个数量级，K8s 场景标签天然丰富（pod/namespace/container）。
6. **sidecar**：每应用 Pod 内一个 collector，最贴近应用、最准确但开销大；**DaemonSet**：每节点一个，采集节点级数据（Falco events、kubelet metrics）；**Gateway/Deployment**：集中接收所有 collector 的数据，做采样 / 转发 / 富化。生产常态：sidecar/DS 采集 → Gateway 聚合 → 后端。
7. ① `kubectl top` 数据延迟 60-90s；② 容器内多线程被 throttle，user CPU 没涨但 host CPU 涨（看 `container_cpu_cfs_throttled_periods_total`）；③ kernel CPU 高（系统调用风暴），cAdvisor 默认只算 user；④ 同节点别的 Pod 抢 CPU。
8. ① 零侵入（不改应用、不注入 sidecar）；② 内核态采集 socket / syscall，覆盖率高且开销低（μs 级）；③ 自动获取 L7 协议（HTTP / gRPC / Redis），不依赖应用打点。
9. **指标**：Pod CPU throttle、GC 时间、网络重传率；**日志**：upstream 慢请求样本；**trace**：找到 P99 trace 看每跳耗时；**eBPF**（Hubble/Pixie）：socket 队列、TCP retrans、内核 stack；**节点**：是否 node 抖动、kubelet 重启、CNI 抖动。
10. 单机瓶颈：内存（每 series ~3KB） + WAL 写 IO。扩展：Thanos sidecar 上传块到对象存储 + Querier 全局视图；Mimir 原生分片 / 多租户；VictoriaMetrics 高压缩比单二进制集群。

</details>

---

## C12 · 精通 K8s 安全

1. （⭐）K8s 的 "四 C 模型"是哪四个？
2. （⭐）`Role` / `ClusterRole` / `RoleBinding` / `ClusterRoleBinding` 四者关系？
3. （⭐⭐）`ServiceAccount` 的 token 怎么注入到 Pod 里？1.22+ 后 token 行为有什么大变化？
4. （⭐⭐）`Pod Security Admission`（PSA）的 `privileged` / `baseline` / `restricted` 三档分别约束什么？为什么替代了 PSP？
5. （⭐⭐）NetworkPolicy 默认行为是 allow 还是 deny？如何做"默认拒绝 + 显式放行"？
6. （⭐⭐）OPA Gatekeeper 与 Kyverno 在策略表达上的核心差异？
7. （⭐⭐⭐）写一个 Kyverno 策略：**禁止任何 Deployment 使用 `:latest` 镜像**。
8. （⭐⭐⭐）镜像供应链安全的三大要素：SBOM、签名、SLSA 各自解决什么？
9. （⭐⭐⭐）一个 Pod 被攻破，攻击者用什么手段可以"逃逸"到节点？常见的几条 path？
10. （⭐⭐⭐）一个集群所有 namespace 都默认 `cluster-admin` 给应用 SA。给出一份"渐进收敛"治理方案。

<details>
<summary>📝 C12 答案与详解</summary>

1. **Cloud / Cluster / Container / Code**——从外到内依次防护，每层都有自己的攻击面。
2. Role / ClusterRole 描述权限；Binding 把权限授予 subject（User/Group/ServiceAccount）。Role 仅命名空间内，ClusterRole 跨命名空间或集群级。RoleBinding 可以引用 ClusterRole，效果是把 ClusterRole 的权限"借"到本 ns。
3. 1.22+ 默认**投影 token**：kubelet 通过 projected volume 注入有 TTL（默认 1 小时）的 token，自动轮换；不再创建那种永不过期的 Secret 类型 token。需要长期 token 必须显式建 `Secret` 资源。
4. baseline：禁特权容器 / hostNetwork / hostPath（大部分允许，挡明显高危）；restricted：进一步禁 `runAsNonRoot` 必须为 true、seccomp 默认 RuntimeDefault 等（最严）。PSP 1.25 已删，因为：① 全局 RBAC 模型复杂；② 没有逐 namespace 控制；③ 策略表达力差。PSA 简化为"标签 + 三档"。
5. 默认 allow（无 NetworkPolicy 时不拦截）。默认拒绝：每个 ns 部署一个 selector 空的 NetworkPolicy + 没有 ingress/egress rule（即拒绝一切），然后逐个为业务加 allow 规则。
6. Gatekeeper 用 Rego（OPA 的 DSL，可表达任意逻辑，学习曲线陡）；Kyverno 用 K8s 风格的 YAML（声明式 match + mutate / validate），更易上手但表达力略弱。
7. ```yaml
   apiVersion: kyverno.io/v1
   kind: ClusterPolicy
   metadata: {name: disallow-latest-tag}
   spec:
     validationFailureAction: enforce
     rules:
     - name: require-image-tag
       match: {resources: {kinds: [Deployment]}}
       validate:
         message: "禁止使用 :latest 镜像"
         pattern:
           spec:
             template:
               spec:
                 containers:
                 - image: "!*:latest"
   ```
8. SBOM：成分清单，回答"镜像里有哪些组件 + 版本"——CVE 出现时可秒级筛查；签名（cosign）：回答"这个镜像是不是我发布的"——抗篡改；SLSA：构建过程的可证明性等级（L1-L4）——回答"构建链路本身可不可信"。
9. ① Pod 用了 `hostPath: /` 挂宿主机根目录；② Pod 跑了 `privileged: true`；③ 利用 CVE（runc / containerd 漏洞）；④ DockerSocket（`/var/run/docker.sock`）挂载到容器；⑤ 高权限 SA 调 API 创建 privileged Pod；⑥ kernel module（cap_sys_module）。
10. ① 全集群扫描：列出所有 SA 的实际权限（`kubectl auth can-i --as=system:serviceaccount:ns:sa --list`）；② 用 Kyverno mutate 拒绝新建 cluster-admin binding；③ 给应用建专门 Role，按 verb 收敛；④ 灰度 namespace 一组组迁移，回归测试；⑤ 长期上 PSA `restricted` 标签。

</details>

---

## C13 · 精通生产调优

1. （⭐）什么是"节点池"（node pool）？为什么生产集群通常有多个？
2. （⭐）`NodeLocal DNSCache` 解决什么问题？性能能提升多少？
3. （⭐⭐）镜像预热（image preload）有哪些做法？冷启动如何降到秒级？
4. （⭐⭐）控制面（API server / etcd / scheduler）的常见瓶颈与定位指标？
5. （⭐⭐）一个集群 5000 节点时，scheduler 调度延迟从 50ms 涨到 1s。可能原因？
6. （⭐⭐）容量规划：单集群推荐节点上限是多少？为什么大厂会拆多集群？
7. （⭐⭐⭐）Karpenter 与 Cluster Autoscaler 的核心区别？为什么 Karpenter 更适合 spot 场景？
8. （⭐⭐⭐）`PodDisruptionBudget` 与 `terminationGracePeriodSeconds` 在节点 drain 时如何配合？
9. （⭐⭐⭐）多集群方案：Karmada / KubeFed v2 / Cluster API / Argo ApplicationSet 各自定位？
10. （⭐⭐⭐）描述一次"线上 Pod 启动慢，从分钟降到 10 秒"的完整优化清单（镜像 / 调度 / 网络 / 应用）。

<details>
<summary>📝 C13 答案与详解</summary>

1. 节点池 = 一组同规格 / 同用途的节点（同机型 + label + taint）。原因：① 工作负载差异化（GPU / 计算密集 / 内存密集）；② 抢占式 vs on-demand；③ 系统组件（kube-system）独立池避免被业务 Pod 挤。
2. Pod 默认走 kube-dns/CoreDNS 的 ClusterIP，跨节点 conntrack 压力大，且 conntrack race 导致偶发 DNS 失败。NodeLocal DNSCache 在**每个节点**部署 DNS 缓存（DaemonSet + 链路本地 IP 169.254.20.10），节点内 Pod DNS 命中本地缓存——失败率从 1% 降到 0.01%，P99 降 70%+。
3. ① 节点起来时 `crictl pull` 预拉常用镜像；② Image streaming（如 stargz / SOCI）按需拉取文件；③ 节点本地 registry 镜像（kraken / Spegel P2P）；④ 制作 AMI 时把镜像 bake 进去。
4. API server：watch 数量 / 长连接、`apiserver_request_duration_seconds`；etcd：`etcd_disk_wal_fsync_duration_seconds`（> 25ms 报警）、leader 切换次数；scheduler：`scheduler_e2e_scheduling_duration_seconds`。
5. ① scheduler 单线程瓶颈 + 大量待调度 Pod；② 过多 Pod affinity / topology constraint 让算法慢；③ etcd 慢导致 List Pod 慢；④ 资源碎片化让 fit predicate 大量失败。修：开 `PercentageOfNodesToScore`、拆调度域、升级到调度器 multi-profile。
6. 官方上限 5000 节点、150000 Pod，但**大厂经验 1500-2000 节点最舒服**，再多 etcd / scheduler 压力陡增。拆多集群理由：① 故障域隔离；② 区域 / 合规分隔；③ 大版本升级回滚域；④ 团队权限隔离。
7. CA：基于 nodegroup（云厂商 ASG），按 Pending Pod 添加节点；Karpenter：不依赖 nodegroup，直接调用云 API 按需开机器，**可挑机型 + 即时启动**。spot 场景：Karpenter 可在 spot 中断时秒级换机型，CA 受 ASG 模板限制反应慢。
8. drain 流程：① 找该节点所有 Pod；② 检查 PDB（minAvailable / maxUnavailable）允许后 evict；③ kubelet 收 evict → 发 SIGTERM → 等 `terminationGracePeriodSeconds` → SIGKILL。PDB 保业务可用，TGP 保单 Pod 优雅退出。
9. Karmada：CNCF 孵化（Incubating），多集群资源编排 + 调度，活跃；KubeFed v2：联邦控制面，已不推荐（项目放缓）；Cluster API：**多集群生命周期管理**（创建 / 升级集群本身）；Argo ApplicationSet：GitOps 视角的多集群部署，不管集群本身。
10. ① 镜像层瘦身（distroless < 50MB）；② 节点镜像预热；③ Pod readiness probe 调到 1s 间隔但 failureThreshold 给余量；④ initContainer 串行 → 并行（同时 fetch 多个依赖）；⑤ 应用本身 lazy init → eager init 分离热路径；⑥ JVM `--XX:+TieredCompilation` 或 GraalVM AOT；⑦ NodeLocal DNS 避免冷启动 DNS 失败重试。

</details>

---

## C14 · 精通 GitOps

1. （⭐）GitOps 的四大原则是什么？
2. （⭐）ArgoCD 与 Flux 的核心定位差异？
3. （⭐⭐）`Application` / `ApplicationSet` / `AppProject` 三者在 ArgoCD 里的关系？
4. （⭐⭐）"Push" 与 "Pull" 部署模型的差异？为什么 GitOps 强调 Pull？
5. （⭐⭐）Sync Wave 是什么？什么时候必须用？
6. （⭐⭐）`Argo Rollouts` 与 `Flagger` 都做渐进式发布，主要差异在哪？
7. （⭐⭐⭐）GitOps 下数据库 migration 怎么做？为什么不能简单地"declarative"？
8. （⭐⭐⭐）一个 ArgoCD Application 一直 `OutOfSync`，但你看每个 manifest 都对。可能原因？
9. （⭐⭐⭐）多集群 + 多环境（dev/staging/prod）的 Git 仓库结构应该怎么设计？monorepo 还是分仓？
10. （⭐⭐⭐）描述一次"PR merge 到 main → 生产 Pod 真的换镜像版本"的完整链路（CI / image tag / config repo / ArgoCD / Rollout）。

<details>
<summary>📝 C14 答案与详解</summary>

1. ① **声明式**（Git 描述期望状态）；② **版本化**（一切变更走 PR）；③ **自动 reconcile**（agent 持续把集群拉到期望状态）；④ **可观测 + 可审计**（drift detection / 历史 trail）。
2. ArgoCD：UI 优秀、CRD 多、运营友好、生态 ApplicationSet / Rollouts；Flux v2：模块化（source / kustomize / helm / notification controller），Toolkit 风格，更适合自动化嵌入。
3. AppProject：权限边界（哪些 Git repo / 集群 / namespace 可被 sync）；Application：单个部署单元；ApplicationSet：按 generator 批量生成 Application（多集群同 chart / 多环境）。
4. Push（传统 CI 部署）：CI 直接 `kubectl apply`，**需要把 kubeconfig 给 CI**，权限大；Pull：agent 跑在集群内，从 Git 拉，**CI 只改 Git**——攻击面小、审计强、解耦。
5. Sync Wave = `argocd.argoproj.io/sync-wave` annotation 数值，控制 manifest 应用顺序。必用场景：先创建 CRD → 再创建 CR；先建 namespace → 再建里面资源；DB migration job → app deployment。
6. Argo Rollouts：CRD 替代 Deployment，**深度** 集成 Argo 生态，支持 blue/green / canary / experiment 多策略；Flagger：保留原生 Deployment，**外挂**控制（适合不想换 CRD 的团队），自动 metric analysis 集成更早。
7. migration 本身有状态、不可幂等回滚，不能"删旧 → 建新"那种声明式。常用模式：① 单独 migration Job + Helm hook / Sync Wave，先于应用部署；② 应用层**向前兼容**——新代码兼容旧 schema，schema 改完后老代码下线，避免 lockstep。
8. ① 资源被 mutating webhook（istio sidecar、Kyverno）注入字段，Argo 看到 cluster vs git diff；② `ignoreDifferences` 没配；③ 资源是 namespaced 但 Argo 配的 cluster scope；④ Helm values 渲染时机不一致（CI 渲染 vs Argo 渲染）。
9. **monorepo 推荐**（apps + envs/{dev,staging,prod}/ overlay），单一真理源、易跨服务原子变更；分仓适合大组织权限隔离。多集群用 ApplicationSet 的 list/cluster generator 自动生成。
10. ① 业务 PR merge → 业务镜像 CI 构建 → 推到 registry，tag 为 commit SHA；② CI 写一个 PR 到 **config repo**（envs/prod/values.yaml 改 image tag）；③ Reviewer 批准 PR merge；④ ArgoCD watch config repo 检测 drift → sync；⑤ Argo Rollouts 接管 canary（10%→25%→50%→100%），每步 Prometheus 指标 analysis；⑥ 全量 → status Healthy。

</details>

---

## C15 · 精通 Serverless on K8s

1. （⭐）K8s 原生 HPA 不能 scale 到 0，Knative / KEDA 是怎么做到 scale-to-zero 的？
2. （⭐）Knative Service / Configuration / Revision / Route 四个 CRD 的关系？
3. （⭐⭐）KPA 与 HPA 在指标维度、最小副本、反应速度上的核心差异？
4. （⭐⭐）描述 Knative Service 从 0 副本接收第一个请求的完整链路（含 Activator 的角色）。
5. （⭐⭐）KEDA HTTP add-on 与 Knative Serving 都做 scale-to-zero，关键差异是什么？什么场景该用哪个？
6. （⭐⭐⭐）一个 Java Spring Boot 服务部署到 Knative，冷启动 12 秒被用户骂，给出至少 4 条优化方案。
7. （⭐⭐⭐）Knative 默认 sidecar-mode Istio 会冲突，为什么？换 Ambient 为什么能解决？
8. （⭐⭐⭐）用 KEDA + Knative 设计一个 "Kafka 订单事件 → 异步处理 Service" 的完整事件驱动 pipeline。
9. （⭐⭐⭐）KServe 在 AI 推理场景为什么大量采用 Knative？scale-to-zero 在 AI 推理场景的特殊价值？
10. （⭐⭐⭐）一个低频后台 Service 配 `min-scale: 0`，但每次冷启动建 20 个数据库连接，DB 受不了。如何解决？

<details>
<summary>📝 C15 答案与详解</summary>

1. Knative 在副本为 0 时，把 Service 的流量先送到 **Activator**（共享组件），Activator 触发 Autoscaler 把 Deployment scale up 后再转发；KEDA 通过自定义 metric provider 让 HPA 支持 0 副本（KEDA 内部接管 0→1 的切换）。
2. Service 是用户面 CRD；Configuration 持有容器与运行配置；每次变更生成不可变 Revision；Route 控制流量在多 Revision 间切分。
3. KPA：指标 QPS / 并发、min 可为 0、有 panic 模式秒级响应；HPA：CPU / 内存 / 自定义指标、min ≥ 1、15-30s 周期。
4. client → ingress → Activator（发现副本=0）→ 调 Autoscaler 申请 scale → K8s API scale Deployment from 0 to 1 → scheduler 选节点 → 拉镜像 + 启动 + readiness pass → Activator 把 buffer 住的请求转给新 Pod。
5. Knative 复杂度高但功能全（Revision / 流量切分 / Eventing）；KEDA HTTP 轻量、保留原生 Deployment、只解决 scale-to-zero。需 Cloud Run 体验用 Knative，已有 Deployment 想 scale-to-zero 用 KEDA。
6. ① GraalVM Native Image / Spring AOT；② 减镜像层 + distroless；③ JIT warmup endpoint；④ 关键 endpoint `min-scale: 1`；⑤ Spring `lazy-initialization=true`；⑥ 节点 image preload。
7. queue-proxy（Knative）与 envoy sidecar（Istio sidecar mode）都想占用一些 netfilter / 端口，行为冲突；Ambient 没有 sidecar，ztunnel 在节点级处理，与 queue-proxy 不冲突。
8. KafkaSource 把 Kafka topic 事件转发到 Knative Broker；Trigger 按 type=order.created 过滤路由到 Knative Service `order-processor`；processor 缩 0 时事件触发拉起，处理完缩回 0；配合 KEDA Kafka scaler 也能直接驱动 Deployment 扩缩。
9. AI 推理模型常常很大（GB 级显存），同时各模型流量不均匀；scale-to-zero 让闲置模型不占 GPU，按需拉起；KServe 直接复用 Knative 的 KPA + Activator。
10. ① 用 PgBouncer / 连接池中间层共享连接；② 应用 lazy init 数据库连接（按需建）；③ 配 `min-scale: 1` 牺牲 scale-to-zero 保连接稳定；④ 把高频接口拆出去，低频接口才用 Serverless。

</details>

---

## C16 · 精通多集群与 Karmada

1. （⭐）单集群即使没顶到 5000 节点上限，企业为什么仍要做多集群？至少 4 个理由。
2. （⭐）Karmada / ArgoCD ApplicationSet / OCM 三种多集群编排方案，核心定位有何不同？
3. （⭐⭐）解释 Karmada 的 PropagationPolicy 与 OverridePolicy 各自的职责。
4. （⭐⭐）写一个 Karmada YAML：把同一 Deployment（6 副本）按 1:2:3 权重分到 member1 / 2 / 3 三个集群。
5. （⭐⭐）Cluster API 把"集群"建模为 K8s CRD，这种设计带来哪三个核心好处？
6. （⭐⭐）Karmada 调度策略 Divided / Duplicated / Aggregated 各自适合什么场景？
7. （⭐⭐⭐）Cilium Cluster Mesh / Submariner / Istio multi-cluster 三种跨集群网络方案的核心差异？分别需要满足什么前提？
8. （⭐⭐⭐）MCS API 与传统跨集群服务发现（east-west gateway / DNS）相比有什么优势？
9. （⭐⭐⭐）某 region 整个集群挂了 5 分钟，业务自动迁移到其它 region。列出从 SLO 监控到流量切换的完整链路。
10. （⭐⭐⭐）100 集群、1 万节点、多云架构。列出从集群创建、应用部署到可观测的完整工具链（每个环节给一个主选 + 一个备选）。

<details>
<summary>📝 C16 答案与详解</summary>

1. ① 故障域隔离（etcd / API server 单点）；② 大版本升级灰度域；③ 区域 / 合规分隔（数据出境 / GDPR）；④ 多云策略；⑤ 团队 / 租户隔离；⑥ 边缘 / CDN POP；⑦ 混合云弹性。
2. Karmada：K8s 风格 CRD + 跨集群调度器，强调 propagation / override；ApplicationSet：GitOps 视角，每集群独立 reconcile；OCM：Red Hat 推动，与 ACM 商业化对应，强调 placement + addon 框架。
3. PropagationPolicy 决定"这份资源分发到哪些集群、副本怎么切分"；OverridePolicy 决定"在某个集群里要做什么差异化修改"（image / replicas / env）。
4. 见 C16 第二章 2.3 节，PropagationPolicy + clusterAffinity + replicaScheduling.staticWeightList。
5. ① 集群本身像 Deployment 一样声明式管理；② 创建 / 升级 / 销毁全用 K8s API；③ 与 GitOps / 其他 controller 天然集成（同一 reconcile 模式）。
6. Divided：副本切分到多集群（最常见，可按权重）；Duplicated：每个集群一份完整副本（全局 DaemonSet / 配置）；Aggregated：尽量集中到能容纳的集群（减少跨集群碎片）。
7. Cilium Cluster Mesh：eBPF / CNI 层 Pod-to-Pod 直通，前提 Pod CIDR 不冲突 + 节点网络可达；Submariner：tunnel-based（IPsec / WireGuard / VxLAN）跨 CNI 兼容性强；Istio multi-cluster：east-west gateway + mTLS，应用层级、对 CIDR 无要求但延迟稍高。
8. MCS API 是 SIG 标准（ServiceExport / ServiceImport），统一了跨集群 service 发现的 API，应用代码用 `svc.ns.svc.clusterset.local` 一致访问；east-west gateway 是实现层、不统一；DNS 方案没有副本健康感知。
9. ① Prometheus 监控本集群健康，SLO breach 5min 触发 Alertmanager；② Karmada toleration 60s 触发 failover，把副本调度到其它集群；③ 全局 DNS / GSLB（Route53 health check）发现 region down 切流量到健康 region；④ Mesh 跨集群路由策略也兜底；⑤ 故障演练（Chaos）定期验证。
10. Cluster API（主）/ kops（备）创建集群；Karmada（主）/ ApplicationSet（备）联邦部署；Cilium Cluster Mesh（主）/ Submariner（备）多集群网络；Thanos + Loki + OTel + Tempo 可观测；ArgoCD（主）/ Flux（备）GitOps；Cosign + Kyverno 安全；Backstage 服务目录。

</details>

---

## 🏆 综合实战题

> 全部章节通过后挑战。每题建议花 30-60 分钟设计，写出**完整方案文档**（资源清单 + 关键决策 + 风险评估）。

### 实战 1：从单体到云原生迁移

一个 Go 单体服务（DB + Redis + 后端 + 前端），跑在 3 台 VM 上。你要把它迁到 K8s，给出：
- 镜像方案（多阶段、distroless、< 50MB）
- 工作负载选型（哪些用 Deployment / StatefulSet）
- 流量入口（Ingress vs Gateway API）
- 配置与密钥（ConfigMap / ExternalSecrets / Vault）
- 资源 requests/limits 与 HPA 策略
- 监控告警最小集（USE / RED）

### 实战 2：多团队多租户 K8s 平台

公司 10 个团队共用 1 个集群，要求：团队 A 不能访问团队 B 的 Pod；team-platform 可以发布共享中间件；流量入口共享 Gateway。给出：
- Namespace 与 RBAC 边界
- NetworkPolicy 默认拒绝 + 跨 ns 显式放行
- Gateway API + ReferenceGrant 共享方案
- Resource Quota / LimitRange 防止资源霸占
- 审计与计费方案

### 实战 3：生产级 Service Mesh + GitOps 平台

要求：所有南北向走 Gateway API、东西向 mTLS、灰度发布 PR 自动触发、可观测 + 全链路 trace。给出：
- 选型决策（Istio Ambient vs Linkerd vs Cilium）
- ArgoCD 多集群结构
- Argo Rollouts 灰度策略（指标 analysis）
- OpenTelemetry 部署（collector 分层）
- 故障演练计划

### 实战 4：自己写一个 Operator

业务需求：自动管理 Redis 主从集群。CRD `RedisCluster` 字段：`replicas`、`version`、`storageSize`、`backupSchedule`。要求：
- CRD 定义 + 校验 webhook
- Reconcile 创建 StatefulSet + Service + PDB
- 主从切换（主死时自动 promote slave）
- 备份 Job（按 schedule 创建 Snapshot）
- finalizer 清理 PVC（可选）
- 测试方案（envtest + e2e）

---

## 📊 自评打分表

| 章节 | 满分 | 自评 | 关键弱点 | 下一步行动 |
|---|---|---|---|---|
| C01 Docker / OCI | 10 |   |   |   |
| C02 工作负载 | 10 |   |   |   |
| C03 网络 / Service | 10 |   |   |   |
| C04 Gateway API | 10 |   |   |   |
| C05 ConfigMap / Secret | 10 |   |   |   |
| C06 调度 / 资源 | 10 |   |   |   |
| C07 存储 / PVC | 10 |   |   |   |
| C08 Helm / Kustomize | 10 |   |   |   |
| C09 Operator / CRD | 10 |   |   |   |
| C10 Service Mesh | 10 |   |   |   |
| C11 可观测 | 10 |   |   |   |
| C12 安全 | 10 |   |   |   |
| C13 生产调优 | 10 |   |   |   |
| C14 GitOps | 10 |   |   |   |
| C15 Serverless on K8s | 10 |   |   |   |
| C16 多集群 / Karmada | 10 |   |   |   |
| **合计** | **160** |   |   |   |

- **< 80**：重读对应章节，优先补底层原理（容器三大基石、Reconcile、mTLS）
- **80-115**：把"陷阱清单"和实际线上场景对照，做实战题 1-2
- **115-145**：合格的中级云原生工程师，挑战实战题 3-4
- **145+**：你大概可以面试 SRE / 平台工程师 senior 岗位了 🎉

---

> 题目有遗漏或答案存疑？欢迎到对应章节末尾的"练习题"小节里看更深入的题目，或在 PR / issue 里讨论。
