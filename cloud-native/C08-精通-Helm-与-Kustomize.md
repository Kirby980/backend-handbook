# 精通 Helm 与 Kustomize：Chart、Release、Overlay、GitOps 与生产实践

> 课程编号：C08
> 路线图来源：云原生 · 模块二 应用打包与发布
> 难度：⭐⭐⭐⭐
> 预计阅读时间：70 分钟
> 内容基准：2026 年 5 月（Helm 3.x、Kustomize 已合入 kubectl、Argo CD 2.x、OCI Chart 主流）

---

## 引言：纯 YAML 为什么不够用

写过几次 Kubernetes 部署的人都见过这个目录：

```
deploy/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── secret.yaml
├── ingress.yaml
├── hpa.yaml
└── pdb.yaml
```

七个 YAML，加起来三百多行。看起来挺整齐。然后业务说："我们要部署到 dev、staging、prod 三个环境，镜像 tag 不同、副本数不同、Ingress host 不同、ConfigMap 里的 endpoint 不同、Secret 里的数据库密码也不同。"

于是有了：

```
deploy/
├── dev/
├── staging/
└── prod/
```

每个目录里七个 YAML——一共 21 个文件。改一个 label，要在三个目录里都改一遍。某次 staging 验证后忘了同步到 prod，事故。再某次跨团队复用一份服务，对方说："我把你的 YAML 全 copy 过去了，只改 image。" 一个月后两份代码漂移成两个版本，没人知道差异在哪。

更难受的事：你做了一个内部组件——比如 Redis Cluster + 监控 + backup CronJob，加起来 12 个 YAML。要分发给公司里 30 个团队用。如果你只是发 YAML 包，每个团队 fork 一份去改 namespace / 改 SC / 改 replicas——三个月后这 30 份代码就漂成 30 个版本，你想推一个 CVE 补丁都做不到。

**纯 YAML 缺三件事**：

1. **参数化**——同一份模板渲染出多个变体（dev/staging/prod、不同集群、不同租户）
2. **包装与版本**——把"一组 YAML + 默认值"作为可分发、可追溯版本的单元
3. **声明式 diff**——上线前要能看到"这次和上次相比改了什么"

业界给出两条路：

| 路径 | 思路 | 代表工具 |
|---|---|---|
| **模板派** | 用模板引擎把 YAML 参数化，渲染时填值 | Helm、Jsonnet、ytt |
| **覆盖派** | 不引入模板，纯 YAML，靠 patch / overlay 改字段 | Kustomize |

Helm 是模板派的事实标准；Kustomize 是覆盖派的事实标准——并且 2019 年起已经合入 `kubectl`（`kubectl apply -k`），不需要单独装。

2026 年的现实是：**两者并存且经常同时用**。Helm 负责"封装可分发的应用包"，Kustomize 负责"在落地时做最后一公里的环境差异化"。本章把这两个工具拆到生产可用的程度——从模板语法、release 状态、OCI 仓库、values schema、CI 校验到 GitOps 集成。

---

## 第一章：Helm 3 架构总览（Tiller 已彻底废弃）

### 1.1 三个核心概念：Chart / Release / Repository

Helm 的世界只有三件事：

```
   Chart (一个版本化的应用模板包)
     │
     │  helm install my-app ./mychart  -f values.yaml
     ▼
   Release (在某个 namespace 里的一次部署实例)
     │
     │  helm upgrade my-app ./mychart -f values.yaml  →  Release Revision N+1
     │  helm rollback my-app 1                        →  回滚到 Revision 1
     ▼
   Kubernetes Cluster (实际运行的 Pod / Service / Ingress ...)
```

- **Chart**：一个目录或 tar 包，包含 `Chart.yaml`、`values.yaml`、`templates/`。本质是"参数化的 YAML 集合 + 元数据"。一个 Chart 有版本（`version`），可发布到 Chart Repository 或 OCI Registry。
- **Release**：把 Chart 安装到集群里的一次实例。同一个 Chart 可以在同一个集群里安装多次（用不同的 release name），每次得到一个独立的 Release。Release 是有状态的——Helm 在集群里以 Secret 形式存它的"历史 Revision"（默认存 10 个版本）。
- **Repository**：Chart 的分发渠道。两种形态：
  - 旧 HTTP repo：一个 index.yaml + 一堆 .tgz 文件（典型如 GitHub Pages 托管）
  - OCI registry：把 chart 当 OCI 镜像存——2026 年的主流

### 1.2 Helm 2 vs Helm 3：Tiller 死了

```
Helm 2 时代:
   Client (helm CLI)  ──(gRPC)──►  Tiller (集群里的 server pod)
                                       │
                                       └──► kubectl apply
       问题：Tiller 是 cluster-admin、没有 RBAC、多租户灾难

Helm 3 时代 (2019 起):
   Client (helm CLI)  ───────────► Kubernetes API
                                       │
                                       └──► 直接用调用方的 kubeconfig
       一切走客户端权限。Release 元数据存在 Secret 里。
```

2026 年还有人提"Tiller"基本是怀旧。**Helm 3 完全是无服务端架构**，安全模型 = 你的 kubeconfig + RBAC。

### 1.3 Release 元数据存在哪

```bash
# 看一下 release 的存储
kubectl get secret -n my-namespace -l owner=helm
# NAME                            TYPE                 DATA   AGE
# sh.helm.release.v1.my-app.v1    helm.sh/release.v1   1      30m
# sh.helm.release.v1.my-app.v2    helm.sh/release.v1   1      5m
```

每个 Revision 一个 Secret，里面是 gzip + base64 的 release 对象（含 chart 内容、values、manifest、status）。这就是为什么 `helm rollback` 能工作——元数据全在集群里。

**坑**：Release Secret 里包含**完整渲染的 manifest**——意味着 Secret 数据也躺在里面，明文 base64。要么把敏感数据从 chart 中分离（用 External Secrets / Sealed Secrets），要么用 `--secret-driver` 把 release 存到外部 KMS 加密的后端。

### 1.4 Helm 命令地图

```
helm create NAME                    # 生成一个 chart 骨架
helm lint  ./mychart                # 静态校验
helm template NAME ./mychart        # 本地渲染——不与集群交互
helm install  NAME ./mychart        # 安装到集群
helm upgrade  NAME ./mychart        # 升级（也可作 install——加 --install）
helm rollback NAME N                # 回滚到 Revision N
helm uninstall NAME                 # 卸载（默认保留 release 历史）
helm list                           # 列出当前 namespace 的 release
helm history NAME                   # 看 release 历史
helm get values NAME                # 拿当前 release 的 values
helm get manifest NAME              # 拿当前 release 渲染出的 manifest
helm package ./mychart              # 打成 .tgz
helm push   mychart.tgz oci://...   # 推到 OCI registry
helm pull   oci://.../mychart       # 拉取
helm registry login ...             # OCI 登录
helm search repo / hub              # 搜索
```

记住一个工作流："**lint → template → install --dry-run → install → upgrade → rollback**"，这套覆盖 90% 日常。

---

## 第二章：Chart 结构与模板语法

### 2.1 目录骨架

`helm create webapp` 生成的目录：

```
webapp/
├── Chart.yaml              # chart 元数据
├── values.yaml             # 默认 values
├── values.schema.json      # （可选）values 的 JSON Schema 校验
├── charts/                 # 依赖 chart 的本地副本（helm dep update 填充）
├── templates/
│   ├── _helpers.tpl        # 命名模板（partial）
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── NOTES.txt           # install 后打印给用户看的提示
│   └── tests/
│       └── test-connection.yaml  # helm test 运行的测试 Pod
├── crds/                   # （可选）chart 安装前自动 apply 的 CRD
└── README.md
```

`Chart.yaml` 最关键的几个字段：

```yaml
apiVersion: v2                # v2 是 Helm 3 标准
name: webapp
description: A demo web app
type: application             # application | library
version: 0.3.1                # chart 自身版本——SemVer
appVersion: "2.6.0"           # 应用版本（display 用、不用 SemVer 也行）
icon: https://example.com/icon.png
home: https://github.com/me/webapp
sources:
  - https://github.com/me/webapp
maintainers:
  - name: Alice
    email: alice@example.com
keywords: [web, demo]
dependencies:
  - name: redis
    version: "18.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts
    condition: redis.enabled
```

**version** 与 **appVersion** 必须分清：前者是 chart 模板本身的版本，后者是被打包应用的版本。改 chart 模板必须改 `version`；只改 `appVersion`（如升级了镜像 tag）也建议同步 bump `version` 的 patch 位，否则 OCI registry 里覆盖推送会失败。

### 2.2 模板语法基础（Go template + Sprig）

Helm 用 Go 标准库的 `text/template`，并融合了 [Sprig](http://masterminds.github.io/sprig/) 函数库。下面这段 `templates/deployment.yaml` 包含了 90% 常用语法：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount | default 2 }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "webapp.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.port }}
              protocol: TCP
          {{- if .Values.probes.enabled }}
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
          readinessProbe:
            httpGet:
              path: /ready
              port: http
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: APP_ENV
              value: {{ .Values.env.name | quote }}
            {{- range $k, $v := .Values.env.extra }}
            - name: {{ $k }}
              value: {{ $v | quote }}
            {{- end }}
```

要点拆解：

1. `{{ }}` 是模板分隔符。前后加 `-` 表示**吃掉相邻空白**：`{{- ... -}}`。这对维持 YAML 缩进至关重要。
2. `.` 是当前作用域。顶层时 `.` = 全部上下文，包含 `.Values`、`.Chart`、`.Release`、`.Files`、`.Capabilities`。
3. `include "name" .` 调用命名模板（partial）并把当前 `.` 作为参数。`include` 比 `template` 强——它返回字符串可被 pipeline 消费。
4. `| nindent N` Sprig 函数：先加换行再缩进 N 个空格——这是 Helm 模板缩进的灵魂。
5. `with` 切换作用域；`range` 遍历 map/slice；`if` 条件。
6. `toYaml`：把对象转成 YAML 字符串——通常配合 `nindent` 嵌进父结构。
7. `default`、`quote`、`required` 是高频 Sprig 函数。`required "msg" .Values.x` 在 .x 为空时直接报错并打印 msg——比模板默认渲染出 `null` 安全得多。

内置对象：

| 对象 | 含义 | 常用字段 |
|---|---|---|
| `.Values` | values.yaml + -f / --set 合并后的值 | 自定义 |
| `.Chart` | Chart.yaml 内容 | `.Name`、`.Version`、`.AppVersion` |
| `.Release` | 本次安装的 release 元数据 | `.Name`、`.Namespace`、`.Service`、`.IsInstall`、`.IsUpgrade`、`.Revision` |
| `.Files` | chart 目录下的文件 | `.Get`、`.Glob`、`.AsConfig`、`.AsSecrets` |
| `.Capabilities` | 集群能力（API versions） | `.KubeVersion`、`.APIVersions.Has "..."` |
| `.Template` | 当前模板自身 | `.Name`、`.BasePath` |

### 2.3 `_helpers.tpl`：DRY 的关键

`templates/_helpers.tpl` 里定义命名模板（不会被 render 成 manifest，只供 include 调用）：

```yaml
{{/* 生成 fullname：release-chart 或者用户指定的 fullnameOverride */}}
{{- define "webapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/* 公共标签——所有资源都该带 */}}
{{- define "webapp.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{ include "webapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/* selectorLabels：Deployment.spec.selector 与 PodTemplate.labels 必须一致 */}}
{{- define "webapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "webapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

为什么 selector labels 要单独抽出来？因为 Deployment 的 `.spec.selector` **immutable** ——一旦上线就不能改。如果你把所有 labels 都塞 selector，将来加 label 就升级失败。**selector 用最小集合（name + instance）；其他 label 走 `labels`**——这是 Helm 官方约定。

### 2.4 钩子（Hooks）

Helm 通过资源 annotation 把 manifest 标记为"安装某阶段执行"：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "webapp.fullname" . }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          command: ["./migrate", "up"]
```

钩子时机：

| Hook | 触发点 |
|---|---|
| `pre-install` / `post-install` | install 前/后 |
| `pre-upgrade` / `post-upgrade` | upgrade 前/后 |
| `pre-delete` / `post-delete` | uninstall 前/后 |
| `pre-rollback` / `post-rollback` | rollback 前/后 |
| `test` | `helm test` 时执行 |

`hook-weight` 控制同一阶段内的执行顺序（数字小先执行，可为负）。`hook-delete-policy` 控制清理时机——`hook-succeeded` 成功后删 Job、`before-hook-creation` 下次创建前先删旧的。

**常见用法**：DB 迁移、CRD 安装（虽然现在更推荐 `crds/` 目录）、license 检查、运行自检 smoke test。

**坑**：hook 资源**不被 release 跟踪**——`helm uninstall` 不会删 hook Job（除非你设了 `hook-delete-policy`）。也意味着 `helm get manifest` 看不到 hook 资源。

### 2.5 子 chart 与依赖

`Chart.yaml`：

```yaml
dependencies:
  - name: redis
    version: "18.x.x"
    repository: oci://registry-1.docker.io/bitnamicharts
    condition: redis.enabled
    alias: cache              # 在 .Values 里以 cache 名字暴露
    import-values:            # 把子 chart 的 values 导出到父 chart
      - child: master.service.port
        parent: cacheServicePort
```

`helm dependency update` 会把依赖下载到 `charts/` 目录。父 chart 的 values 可以覆盖子 chart：

```yaml
# 父 chart values.yaml
redis:
  enabled: true
  auth:
    password: "supersecret"   # 覆盖 bitnami/redis 的 auth.password
  replica:
    replicaCount: 3
```

**库 chart（library chart）**：`type: library` 的 chart 不渲染资源，只提供命名模板供其他 chart `include`。适合公司内部抽公共标签、公共 sidecar 注入模板。

---

## 第三章：values 覆盖优先级与 schema 校验

### 3.1 values 覆盖链

从低到高：

```
chart/values.yaml                                  ← chart 自带默认
   ↓
父 chart values.yaml 中的子 chart 节                 ← 用作子 chart 时
   ↓
-f / --values myvalues.yaml （可多个，后者覆盖前者）  ← 命令行文件
   ↓
--set key=value  /  --set-string  /  --set-file     ← 命令行参数
```

`--set` 用 dot 语法：

```bash
helm upgrade webapp ./chart \
  -f values-prod.yaml \
  -f values-prod-secrets.yaml \
  --set image.tag=2.6.1 \
  --set 'env.extra.DEBUG=false' \
  --set 'ingress.hosts[0].host=app.example.com'
```

注意 `--set` 处理类型有 quirk：

- `--set replicaCount=3` 解析成数字 3
- `--set image.tag=2.6.1` 解析成字符串 "2.6.1"——但 `2.6` 会被解析成 float64！要用 `--set-string` 强制字符串
- 字符串里有 `,` 要转义：`--set 'list=a\,b\,c'`

**生产建议**：`--set` 仅用于一两个动态值（CI 注入的 image.tag、commit sha 等）。其他全走 `-f`，**values 文件用 Git 版本管理**。

### 3.2 values.schema.json：把"打字错误"挡在 render 前

values 太自由是 chart 维护者的噩梦——用户可能：

- 把 `replicaCount` 写成字符串
- 写错字段名 `repclias`、`Replcicas`
- 漏配必填字段（如 image.repository）
- 写超出范围的值（如 minReplicas > maxReplicas）

`values.schema.json` 让 Helm 在 install/upgrade 前自动校验：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["image", "service"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": { "type": "string", "minLength": 1 },
        "tag":        { "type": "string" },
        "pullPolicy": { "enum": ["Always", "IfNotPresent", "Never"] }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": { "enum": ["ClusterIP", "NodePort", "LoadBalancer"] },
        "port": { "type": "integer", "minimum": 1, "maximum": 65535 }
      }
    },
    "resources": {
      "type": "object",
      "properties": {
        "limits":   { "type": "object" },
        "requests": { "type": "object" }
      }
    }
  }
}
```

`helm install --dry-run` 时如果 values 不合规会立即报错并指出具体路径。**对外发布的 chart 必须配 schema**——否则用户的 typo 会在你的模板里渲染出诡异的 manifest，然后报一堆云山雾罩的错。

### 3.3 模板渲染调试套件

```bash
# 渲染所有模板，结果打到 stdout——不连集群
helm template myrelease ./chart -f values.yaml

# 只渲染指定模板
helm template myrelease ./chart -s templates/deployment.yaml

# 模拟 install，连集群但不真的部署——会做 admission webhook 校验
helm install myrelease ./chart --dry-run --debug

# 看一下 release 已渲染的 manifest（已部署的）
helm get manifest myrelease

# 看一下 release 用了哪些 values
helm get values myrelease         # 仅用户提供的
helm get values myrelease --all   # 含 defaults
```

**debug 法宝**：当 `helm install` 报 "error converting YAML to JSON" 时，几乎一定是缩进炸了。这时立刻 `helm template ... --debug` 看渲染输出——`--debug` 会把渲染失败的 YAML 也吐出来（带行号）。

---

## 第四章：OCI Chart Repository（2026 主流）

### 4.1 传统 HTTP Chart Repository

旧的 Chart Repo 是这样的：

```
https://charts.example.com/
├── index.yaml              # 元数据文件
├── webapp-0.3.0.tgz
├── webapp-0.3.1.tgz
└── webapp-0.3.1.tgz.prov   # 签名（GPG）
```

`index.yaml` 列出所有 chart 的版本、digest、URL。GitHub Pages 就能托管，门槛极低。但有几个缺点：

- 没有 RBAC、没有 audit log
- 签名机制是 GPG（用得很少）
- 与镜像仓库分离——同一应用要维护两套发布

### 4.2 OCI Chart：把 Chart 当镜像存

Helm 3.8（2022 起）开始 stable 支持 OCI——把 chart 推到任何 OCI 兼容的 registry（Harbor、ACR、ECR、ghcr.io、docker hub）。2026 年已是事实主流。

```bash
# 登录
helm registry login registry.example.com -u user -p pass

# 打包
helm package ./webapp           # 生成 webapp-0.3.1.tgz

# 推送
helm push webapp-0.3.1.tgz oci://registry.example.com/charts
# Pushed: registry.example.com/charts/webapp:0.3.1
# Digest: sha256:abcdef...

# 拉取
helm pull oci://registry.example.com/charts/webapp --version 0.3.1
# 或直接 install
helm install myrelease oci://registry.example.com/charts/webapp --version 0.3.1
```

OCI Chart 的优势：

1. **复用镜像仓库的 RBAC、签名、scan**——chart 也走 cosign、走 Trivy 扫描
2. **不可变**：OCI registry 的 digest 不可变；同 tag 推送会失败（除非允许 overwrite）
3. **零额外基础设施**：你已经在用 Harbor/ACR/ECR 存镜像了，加 chart 就是几条命令
4. **支持 cross-region replication**——chart 跟镜像一起复制

**坑**：

- OCI Chart 的 URL 是 `oci://<host>/<path>/<chart-name>:<version>`——注意没有 `index.yaml`，搜索要靠 registry 的 API
- `helm search` 不直接搜 OCI（v3.13+ 部分支持 OCI catalog API，仍不通用）。生产环境通常自建一个 chart catalog 页面
- 推送时 chart name 必须等于 OCI 仓库的 last path segment——否则报错 "name mismatch"

### 4.3 签名：cosign + provenance

```bash
# 用 cosign 签 chart
cosign sign --key cosign.key registry.example.com/charts/webapp:0.3.1

# 验签
cosign verify --key cosign.pub registry.example.com/charts/webapp:0.3.1
```

供应链安全要求严的场景（金融、政府）几乎都强制 chart 签名 + admission policy（用 Sigstore Policy Controller / Kyverno）拦截未签名 chart。

---

## 第五章：Kustomize 设计哲学——拒绝模板

### 5.1 三段宣言

Kustomize 的设计原则在它的 [FAQ](https://kubectl.docs.kubernetes.io/) 反复强调：

1. **没有模板**——所有文件都是合法的 YAML，可以直接 `kubectl apply -f` 跑
2. **没有 DSL**——没有 if、没有 for、没有变量插值
3. **声明式 patch**——你不"写一个变体"，你"声明它和 base 的差异"

```
        base/  (一组标准 YAML——可直接 kubectl apply)
       /  |  \
      /   |   \
   dev/  staging/  prod/
   (overlay：声明在 base 之上的差异)
```

`overlay` = `base` + `patches/transformers/generators`。Kustomize 拿到 overlay 后会读 base，按声明的修改进行变换，输出最终 YAML。

### 5.2 最小可工作示例

```
myapp/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── ingress.yaml
```

`base/kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
  app.kubernetes.io/name: myapp
  app.kubernetes.io/part-of: platform

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

images:
  - name: myapp                 # 镜像 placeholder 名
    newName: registry.io/myapp
    newTag: "1.0.0"
```

`overlays/prod/kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: prod
namePrefix: prod-

resources:
  - ../../base
  - ingress.yaml

patches:
  - path: replica-patch.yaml
    target:
      kind: Deployment
      name: myapp

images:
  - name: myapp
    newName: registry.io/myapp
    newTag: "1.2.3"

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - FEATURE_X=true

replicas:
  - name: myapp
    count: 5
```

`replica-patch.yaml`（Strategic Merge）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1
              memory: 1Gi
```

构建：

```bash
kubectl kustomize overlays/prod          # 渲染输出
kubectl apply -k overlays/prod           # 渲染并 apply
kubectl diff -k overlays/prod            # diff 当前集群
```

### 5.3 Kustomize 已合入 kubectl

2019 年起 `kubectl -k` 就内置了 Kustomize（基于 client-go 调用 Kustomize 库）。**2026 年不需要单独装 kustomize 二进制**，除非你要用最新版的 Kustomize（kubectl 内置版本通常滞后 6-12 个月）。

最新独立版本支持的特性 kubectl 不一定支持。对生产建议：

```bash
# 安装独立 kustomize（保持最新）
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

# CI 中显式调用
kustomize build overlays/prod | kubectl apply -f -
```

---

## 第六章：Kustomize 的五个关键概念

### 6.1 base / overlays

`resources` 字段引用其他 kustomization——可以是目录（递归 build）或单个 YAML 文件。`base` 可以叠多层：

```
common/        (公司全局 base)
   ↑
   │
my-app-base/   (单服务 base)
   ↑
   │
my-app-dev/    (环境 overlay)
```

只要每层都是合法的 kustomization，就能任意组合。

### 6.2 Patches：四种打补丁姿势

**1. Strategic Merge Patch**

```yaml
# patches/resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp           # 关键：同名字段才会合并
          resources:
            requests:
              cpu: 1
```

只列你要改的字段——其他保持 base。**坑**：默认 list 用 "replace" 策略——比如 `containers` 是 list，patch 里写一个 container 就替换整个列表。靠 Kubernetes 类型里定义的 strategic merge key（多数 list 字段配的是 `name`）来识别同元素并合并。

**2. JSON 6902 Patch**

```yaml
# kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: NEW_VAR
          value: "true"
```

JSON Patch 显式、可精确控制 list 操作（add at index、remove、move）。Strategic Merge 操作 list 困难时上 JSON Patch。

**3. Inline Patch**

Kustomize 4+ 支持把 patch 内容直接写在 kustomization.yaml 里：

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: myapp
      spec:
        replicas: 3
```

**4. patchesStrategicMerge / patchesJson6902（旧字段，已 deprecated）**

老 yaml 里常见 `patchesStrategicMerge:` 和 `patchesJson6902:`。Kustomize 4 起统一用 `patches:` 字段+`target` 判定类型——**新代码不要再用旧字段**。

### 6.3 Generators：从字面量/文件生成 ConfigMap & Secret

```yaml
configMapGenerator:
  - name: app-config
    files:
      - config.json
      - logging.conf
    literals:
      - LOG_LEVEL=info
      - REGION=us-west-2

secretGenerator:
  - name: db-password
    literals:
      - password=supersecret
    type: Opaque
```

Generator 生成的 ConfigMap/Secret 会自动加 **hash 后缀**：

```
app-config-h7t8m2k9bd
db-password-9d8sk3f2c1
```

这是 Kustomize 的精髓之一：**hash 变了 → name 变了 → 引用的 Deployment 自动 rollout**。再也不用手动 `kubectl rollout restart` 来让 Pod 重读 ConfigMap。

引用方式：

```yaml
# deployment.yaml（base）
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config    # 写 base name，Kustomize 自动补 hash
        key: LOG_LEVEL
volumes:
  - name: config
    configMap:
      name: app-config      # 同理
```

要禁用 hash（比如外部工具引用、调试不想 rolling）：

```yaml
generatorOptions:
  disableNameSuffixHash: true
```

`behavior: merge` 让 overlay 的 generator 与 base 的同名 ConfigMap 合并；`behavior: replace` 整个换掉。

### 6.4 Transformers：批量改字段

`commonLabels` / `commonAnnotations` / `namespace` / `namePrefix` / `nameSuffix` / `images` 都是 transformer。它们在所有资源上**全局**应用。

`images` 字段尤其重要——CI 流水线用它注入新 tag：

```bash
cd overlays/prod
kustomize edit set image myapp=registry.io/myapp:${COMMIT_SHA}
# 等价于修改 kustomization.yaml 里 images 字段
git diff kustomization.yaml
git commit -am "deploy: bump myapp to ${COMMIT_SHA}"
```

GitOps 工具（Argo CD、Flux）轮询到 Git 变更后自动 sync。**不需要 dynamic value 渲染**——这是 Kustomize 与 Helm 在 GitOps 场景的最大差异之一。

### 6.5 Components：可复用横切关注点

Kustomize 4 引入 Component——比 base 更轻量，允许"选择性叠加"：

```
components/
├── monitoring/
│   ├── kustomization.yaml
│   └── servicemonitor.yaml
├── mtls/
│   ├── kustomization.yaml
│   └── peer-authentication.yaml
└── network-policy/
    └── ...
```

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component                    # 注意：Component 不是 Kustomization

resources:
  - servicemonitor.yaml

commonLabels:
  monitoring: enabled
```

overlay 使用：

```yaml
# overlays/prod/kustomization.yaml
components:
  - ../../components/monitoring
  - ../../components/mtls
```

适合："要不要开监控？要不要开 mTLS？要不要带 NetworkPolicy？"这种**正交开关**。

---

## 第七章：Helm vs Kustomize——何时用哪个

### 7.1 决策矩阵

| 维度 | Helm 更合适 | Kustomize 更合适 |
|---|---|---|
| 分发给外部 | ✅ 版本化 chart、OCI 发布、values schema | ❌ Kustomize 没有打包格式 |
| 应用复杂、字段差异多 | ✅ Go template 灵活 | ⚠️ 大量 patch 文件不易维护 |
| 多环境（dev/staging/prod）| ⚠️ 多个 values.yaml | ✅ overlay 天然适合 |
| GitOps（声明式 Git→集群）| ⚠️ values 文件可入 Git，渲染过程不透明 | ✅ 全是 YAML，diff 直观 |
| 包含复杂逻辑（条件、循环）| ✅ template 支持 | ❌ 不支持，要硬塞 component |
| 强烈想"看到最终 YAML" | ⚠️ `helm template` 输出 | ✅ 本来就只有 YAML |
| 有 release lifecycle（升级、回滚）| ✅ Helm 自带 | ❌ Kustomize 是 stateless |
| 新人学习成本 | ⚠️ Go template 学一阵 | ✅ 几乎零成本 |

### 7.2 混合用法（最常见）

**模式一：Helm 渲染 → Kustomize 覆盖**

社区/公司发布一个 Helm chart——你拿来用，但要做"chart 作者没想到"的字段调整。

```yaml
# kustomization.yaml
helmCharts:
  - name: prometheus
    repo: https://prometheus-community.github.io/helm-charts
    version: 25.x.x
    releaseName: monitoring
    namespace: monitoring
    valuesFile: prometheus-values.yaml

patches:
  - target:
      kind: Deployment
      name: monitoring-prometheus-server
    patch: |
      - op: add
        path: /spec/template/spec/tolerations
        value:
          - key: dedicated
            value: monitoring
            effect: NoSchedule
```

`kustomize build --enable-helm` 会先用 Helm 渲染 chart，然后 Kustomize 在结果上叠 patch。

**坑**：`--enable-helm` 是显式开关。CI 里 `kustomize build` 漏了这个 flag 会直接报 "must specify --enable-helm" 然后失败——记得在 Argo CD `argocd-cm` 里加 `kustomize.buildOptions: "--enable-helm"`。

**模式二：Helm 作打包，Kustomize 不参与**

如果你的应用：

- 要发布到多个团队/客户
- 有大量字段需要参数化
- 有 lifecycle 概念

直接用 Helm，**别引入 Kustomize**。多套环境就用多个 values 文件。

**模式三：Kustomize 直接管，Helm 不参与**

如果你的应用：

- 完全内部使用
- 字段差异主要是 namespace、replicas、image tag
- 团队对 Helm template DSL 不熟

直接用 Kustomize。base + overlay 就够了。

---

## 第八章：Helmfile / Argo CD ApplicationSet 配合 Helm

### 8.1 Helmfile：声明式管理多 release

Helm 本身管单 release 很顺，但管"一组 release"（如一套 dev 环境 = nginx-ingress + cert-manager + redis + my-app）就笨——要写 shell 脚本调用 `helm upgrade --install` 一遍。Helmfile 解决这个：

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: oci://registry-1.docker.io/bitnamicharts
    oci: true

releases:
  - name: redis
    namespace: data
    chart: bitnami/redis
    version: 18.x.x
    values:
      - replica.replicaCount: 3
      - auth.password: {{ env "REDIS_PASSWORD" }}

  - name: webapp
    namespace: app
    chart: ./charts/webapp
    values:
      - values-{{ .Environment.Name }}.yaml
    needs:
      - data/redis
```

`helmfile sync` 把所有 release 拉齐到声明状态；`helmfile diff` 显示与集群差异。**适合本地开发 / 自建 PaaS 的多组件部署**。

### 8.2 Argo CD ApplicationSet：批量 Application

GitOps 主流是 Argo CD（Flux 是同等替代品）。一个 Argo CD Application 对应一个 release：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/me/webapp
    targetRevision: main
    path: deploy/overlays/prod        # Kustomize overlay
    # 或 Helm：
    # chart: webapp
    # helm:
    #   valueFiles:
    #     - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

ApplicationSet 让你**生成一组 Application**——典型场景：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp-all-envs
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://dev.example.com
          - env: staging
            cluster: https://staging.example.com
          - env: prod
            cluster: https://prod.example.com
  template:
    metadata:
      name: 'webapp-{{env}}'
    spec:
      source:
        repoURL: https://github.com/me/webapp
        targetRevision: main
        path: 'deploy/overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: webapp
      syncPolicy:
        automated:
          prune: true
```

或者 git generator（自动发现新环境目录）：

```yaml
generators:
  - git:
      repoURL: https://github.com/me/webapp
      revision: main
      directories:
        - path: deploy/overlays/*
```

新增 `overlays/qa/` 目录 → ApplicationSet 自动生成 `webapp-qa` Application → Argo CD 部署到 QA 集群。**这是企业级多环境 GitOps 的事实方案**。

---

## 第九章：CI / GitOps 集成实战

### 9.1 PR 检查：lint + dry-run + diff

`.github/workflows/chart.yml`：

```yaml
name: chart-ci

on:
  pull_request:
    paths:
      - 'charts/**'
      - 'overlays/**'

jobs:
  helm-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4

      - name: helm lint
        run: helm lint charts/webapp --strict

      - name: schema validate
        run: |
          # 跑 values schema 校验（隐式：lint 已校验）
          helm template charts/webapp -f charts/webapp/values.yaml > /tmp/out.yaml

      - name: kubeconform
        run: |
          curl -sL https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-amd64.tar.gz | tar xz
          helm template charts/webapp | ./kubeconform -strict -kubernetes-version 1.30.0

      - name: kyverno policy
        run: |
          curl -sLo kyverno.tar.gz https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli_linux_x86_64.tar.gz
          tar xf kyverno.tar.gz
          helm template charts/webapp | ./kyverno apply policies/ --resource -

  helm-diff-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
      - uses: azure/setup-kubectl@v4

      - name: install helm-diff
        run: helm plugin install https://github.com/databus23/helm-diff

      - name: connect to prod (read-only)
        run: |
          echo "${{ secrets.PROD_KUBECONFIG_RO }}" > $HOME/.kube/config

      - name: helm diff
        run: |
          helm diff upgrade webapp ./charts/webapp \
            -f ./charts/webapp/values-prod.yaml \
            --namespace prod \
            --detailed-exitcode || true   # 0=无差异 1=报错 2=有差异
```

**关键工具**：

| 工具 | 用途 |
|---|---|
| `helm lint --strict` | chart 静态校验 |
| `helm template` | 离线渲染 |
| [`kubeconform`](https://github.com/yannh/kubeconform) | 校验 manifest 符合 K8s schema |
| [`kyverno`](https://kyverno.io/) / [`OPA gatekeeper`](https://open-policy-agent.github.io/gatekeeper/) | 策略校验（必须有 resources、必须配 PDB...） |
| [`helm-diff`](https://github.com/databus23/helm-diff) | 看 upgrade 会改什么 |
| [`kustomize-diff`](https://github.com/kubernetes-sigs/kustomize) (`kubectl diff -k`) | Kustomize 等价物 |

### 9.2 Kustomize 在 CI 的等价方案

```bash
# 验证 build 通过
kustomize build overlays/prod > /tmp/prod.yaml

# kubeconform 校验
kubeconform -strict /tmp/prod.yaml

# kyverno 校验
kyverno apply policies/ --resource /tmp/prod.yaml

# 对比集群（要 ro kubeconfig）
kubectl diff -k overlays/prod
```

### 9.3 GitOps 部署模式

**模式一：CI 渲染 → 推 manifest 到 GitOps repo**

```
[App repo]                  [GitOps repo]                [Cluster]
   │                             │                          │
   │  build & test               │                          │
   │  helm template / kustomize  │                          │
   │  push manifest YAML ───────►│                          │
   │                             │  Argo CD watch ─────────►│
```

优点：集群里跑的是"最终 YAML"，可审计；GitOps repo 就是 source of truth。
缺点：要维护两个 repo + 双向同步。

**模式二：GitOps repo 直接放 chart / overlay，Argo CD 渲染**

```
[GitOps repo]                                            [Cluster]
   │  chart / overlay 源码                                    │
   │                                                           │
   │  Argo CD pull → helm template / kustomize build → apply  │
```

优点：单 repo，简单。
缺点：渲染发生在 Argo CD 里——values 改了之后预览要靠 Argo CD UI 或 `argocd app diff`。

2026 年企业级架构两种都存在。**Argo CD 2.x 默认走模式二**——这是最主流方案。

---

## 第十章：生产实践

### 10.1 Chart 版本管理纪律

**规则一：SemVer 严肃执行**

- `1.2.3 → 1.2.4`：纯 bug fix（模板逻辑无破坏性）、appVersion 微调
- `1.2.3 → 1.3.0`：新增字段、新增功能、values 字段有新增（向前兼容）
- `1.2.3 → 2.0.0`：values 字段名/类型变更、删除 deprecated 字段、模板生成的资源 kind 改变

**规则二：appVersion 与 version 同时改**

只升 appVersion 不升 version 等于"chart 内容没变"——所有 OCI registry 都会因 digest 冲突拒绝推送。即使只是改 image tag 默认值，也要 bump version 的 patch 位。

**规则三：CHANGELOG.md 必填**

```
## [0.3.1] - 2026-05-14
### Added
- Support for HorizontalPodAutoscaler v2 (requires Kubernetes 1.23+)
- New value `tolerations` (default `[]`)
### Changed
- `image.tag` default bumped from 2.5.4 to 2.6.0
- BREAKING: `service.port` renamed to `service.ports.http` (migration: ...)
```

OCI registry 不存 CHANGELOG，所以放 Git。

### 10.2 values 分层组织

不要把所有 values 塞一个文件。生产 chart 推荐：

```
values.yaml                  # 默认值（最小可用）
values-dev.yaml              # dev 环境覆盖
values-staging.yaml
values-prod.yaml
values-prod-secrets.enc.yaml # 加密 secrets（sops + age）
```

部署时叠加：

```bash
helm upgrade webapp ./chart \
  -f values.yaml \
  -f values-prod.yaml \
  -f <(sops -d values-prod-secrets.enc.yaml)
```

**Secrets 不入 Git 明文**——用 [sops](https://github.com/getsops/sops) + age/KMS、或 [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)、或 External Secrets Operator 拉外部秘密管理（Vault、AWS Secrets Manager、GCP Secret Manager）。后者在 2026 是首选。

### 10.3 Resource、PDB、HPA 强制配置

公共 chart 必须默认带：

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi   # CPU limit 多数场景建议不配——CPU throttling 危害大

podDisruptionBudget:
  enabled: true
  minAvailable: 1

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

用户禁用时显式 `enabled: false`——比"没配"安全得多。

### 10.4 Chart test：smoke test 写在 chart 里

`templates/tests/test-connection.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test"
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox:1.36
      command:
        - sh
        - -c
        - 'wget -qO- http://{{ include "webapp.fullname" . }}:{{ .Values.service.port }}/healthz | grep ok'
```

```bash
helm test webapp
```

`helm test` 在集群跑这个 Pod，退出码 0 视为通过。**CI 升级后跑 helm test 验证业务可达**。

### 10.5 命名与 namespace 漂移

`helm upgrade --install` 默认在当前 kubeconfig context 的 default namespace 安装。生产**永远显式带 namespace**：

```bash
helm upgrade webapp ./chart -n prod --create-namespace -f values-prod.yaml
```

`--create-namespace` 防止 namespace 不存在导致 install 失败（首次）。

**Helm 不"管理"namespace**——chart 里写 `kind: Namespace` 有副作用：`helm uninstall` 会把 namespace 一并删——连带删掉**所有同命名空间下别人的资源**。安全做法是 namespace 用外部工具创建（Argo CD AppProject、Terraform、单独 yaml）。

### 10.6 升级保留历史 vs 清理

```bash
helm upgrade webapp ./chart --history-max 10        # 默认 10，保留最近 10 个 revision
helm upgrade webapp ./chart --atomic --timeout 5m   # 失败自动 rollback
helm upgrade webapp ./chart --wait                  # 等 resources 全部 ready 再返回
helm upgrade webapp ./chart --cleanup-on-fail       # 失败时清理新建的资源
```

`--atomic` 等价 `--wait --cleanup-on-fail`——生产几乎默认开。`--timeout` 大点（5m+），避免大集群拉镜像慢被误判。

---

## 第十一章：陷阱清单

### 11.1 模板渲染失败常见原因

```
Error: YAML parse error on webapp/templates/deployment.yaml:
  error converting YAML to JSON: yaml: line 23: mapping values are not allowed in this context
```

**原因 1：缩进炸了**

```yaml
{{- with .Values.podAnnotations }}
annotations:
  {{- toYaml . | nindent 8 }}    # 应该 nindent 4 才对——多了 4 个空格
{{- end }}
```

修：`helm template ... --debug | sed -n '20,30p'` 看实际行。`nindent` 的数字 = 父字段的列号。

**原因 2：missing `{{-` 或 `-}}` 留下空行**

```yaml
apiVersion: v1
kind: Service

metadata:                  # 这个空行可能是模板里 {{ end }} 留下的
  name: foo
```

空行有时无害，但在嵌套 list 里能让 YAML parser 觉得是新 doc。

**原因 3：`toYaml` 误用**

```yaml
env:
  {{ toYaml .Values.env }}     # 错——toYaml 输出多行，会破坏 env 的 list 起始符
```

修：

```yaml
env:
  {{- toYaml .Values.env | nindent 2 }}
```

### 11.2 values 覆盖意外

```yaml
# values.yaml
ingress:
  enabled: true
  hosts:
    - host: app.example.com
      paths: [/]
```

```bash
helm upgrade ... --set ingress.hosts[0].host=new.example.com
```

得到——

```yaml
ingress:
  enabled: true
  hosts:
    - host: new.example.com
      # paths 没了！
```

**原因**：`--set ingress.hosts[0].host=...` 把整个 list 第 0 项替换成只有 host 字段的对象，paths 被覆盖。

**修法**：

- 用 `-f values-override.yaml` 而不是 `--set`
- 或一起 set：`--set 'ingress.hosts[0].host=new,ingress.hosts[0].paths[0]=/'`——丑且易错

经验：**只要 values 是 list-of-object 结构，几乎一定用 `-f` 覆盖**。

### 11.3 namespace 漂移

```bash
# 第一次：忘了 -n
helm install webapp ./chart -f values-prod.yaml
# Helm 看 kubeconfig current context → namespace=default
# Pod 装到 default 了

# 第二次：发现错了
helm upgrade webapp ./chart -n prod -f values-prod.yaml
# Helm 在 prod 找不到 release webapp → 报 "no release found"
```

Helm release 是 **(namespace, name) 二元组**唯一。第一次安装的 namespace 决定了将来所有 upgrade/rollback/uninstall 都要用同一个 namespace。

**修法**：

```bash
# 卸载错误位置
helm uninstall webapp                 # 删 default 里的
# 重装到 prod
helm install webapp ./chart -n prod -f values-prod.yaml
```

CI 脚本里**永远显式 `-n <namespace>`**——不要依赖 kubeconfig context。

### 11.4 Kustomize 路径变更引发的 rollout

ConfigMap hash 后缀变化是好事——除非：

```yaml
# 外部工具引用了 ConfigMap
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-rules
spec:
  groups:
    - name: app
      rules:
        - record: app:rate
          expr: rate(app_requests[5m])
```

PrometheusRule 没受 Kustomize hash 影响——但如果它通过 ConfigMap 被 Prometheus operator 加载，那个 ConfigMap 改名会让 operator 找不到。

**修法**：把"非滚动场景"的 ConfigMap 禁掉 hash：

```yaml
configMapGenerator:
  - name: app-rules
    options:
      disableNameSuffixHash: true
    files:
      - rules.yaml
```

### 11.5 helm rollback 不能跨越破坏性变更

```bash
# v3：把 Service.spec.selector 改了
helm upgrade webapp ./chart      # 成功
# 发现新版本有 bug
helm rollback webapp 2
# Error: cannot patch "webapp" with kind Service: Service "webapp" is invalid:
#   spec.clusterIP: Invalid value: "": field is immutable
```

Service.spec.clusterIP、Deployment.spec.selector 等 immutable 字段在 rollback 时会失败。

**修法**：

```bash
helm rollback webapp 2 --force         # 强制：delete + recreate（有 downtime！）
# 或手动 kubectl delete service webapp; helm rollback webapp 2
```

写 chart 时尽量避免 immutable 字段的变化。

### 11.6 Helm hook 资源不被卸载

```bash
helm uninstall webapp
# 一个 db-migrate Job 还留着
kubectl get jobs
# webapp-db-migrate   1/1   1h
```

`hook-delete-policy` 没设 `before-hook-creation` → 老的不删。每次 upgrade 都堆一个 Job。

**修法**：所有 hook 都默认带：

```yaml
annotations:
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

### 11.7 OCI Chart push 报 "name mismatch"

```bash
helm package ./mychart
# Successfully packaged chart and saved it to: /tmp/foo-0.1.0.tgz
helm push /tmp/foo-0.1.0.tgz oci://registry.example.com/charts
# Error: chart name "mychart" does not match expected name "foo"
```

`helm push` 要求文件名的 chart 名等于 Chart.yaml 的 `name`——`foo-0.1.0.tgz` 解析的 chart name 是 `foo`，但 Chart.yaml 里写的是 `mychart`。

**修法**：

- 改 Chart.yaml 的 `name` 与目录一致
- 或 `helm package` 时显式 `--destination`：但 chart name 由 Chart.yaml 决定，不能改输出名

### 11.8 Chart 依赖未更新

```bash
helm install webapp ./chart
# Error: found in Chart.yaml, but missing in charts/ directory: redis
```

`Chart.yaml` 写了依赖但 `charts/` 目录里没下载——一定是忘了 `helm dependency update`。

**修法**：CI 在 lint/install 前必跑：

```bash
helm dependency update ./chart
helm lint ./chart
```

提交时把 `Chart.lock` 入 Git（锁定子 chart 版本）。`charts/` 里的 tgz 一般不入 Git（用 `.helmignore` 忽略），CI 重新 download。

### 11.9 Kustomize patch target 找不到资源

```yaml
patches:
  - target:
      kind: Deployment
      name: myapp           # 实际是 app-myapp（base 用了 namePrefix）
    path: replicas.yaml
```

`target` 是在**当前 kustomization build 输出后**做匹配——如果 base 加了 namePrefix，target.name 也要写带 prefix 的。或者用 `labelSelector` 而不是 name：

```yaml
target:
  kind: Deployment
  labelSelector: app.kubernetes.io/component=api
```

### 11.10 helm dep update 慢 / 拉外网

公司内网无外网访问，helm dep update 死活拉不到 charts。

**修法**：

- 把所有 chart 放公司 OCI registry 镜像一份
- Chart.yaml 里改用内部 registry URL
- 或本地预先下载好（vendoring）：把 `charts/` 目录入 Git（违反一般实践，但内网严格场景常见）

---

## 第十二章：2026 现状

### 12.1 Helm 3.x 主流稳态

- Helm 3.14+（2026 主流）：稳态运维，没有破坏性变更
- OCI 是默认 chart 分发——传统 HTTP repo 仍在但新建议都走 OCI
- `helm install --dry-run=server` （3.13 起）支持服务端 dry-run，比纯客户端 dry-run 准
- Helm 没有 Helm 4 路线图——OCI 与 SDK 改进都是渐进式

### 12.2 Kustomize 5.x

- Kustomize 5.0 (2023) 起稳定。`patches:` 替代旧 `patchesStrategicMerge:` / `patchesJson6902:`
- kubectl 5.30+ 内置 Kustomize 5.x
- 主推 Component 模式（横切关注点正交化）

### 12.3 Argo CD / Flux GitOps 双雄

- Argo CD 2.x（2026 主流）成熟，2.10+ 引入 ApplicationSet 增强、原生 OCI Chart 支持、PR-based preview environment
- Flux 2.x 与 Argo CD 功能近似；社区分歧主要在工程哲学（Argo CD 更"应用中心"、Flux 更"控制器组合"）

### 12.4 OCI 全面接管

- **镜像** + **Chart** + **OPA bundle** + **WASM module** 都用 OCI 仓库存——一个 Harbor/ACR/ECR 解决所有
- Cosign 签名成行业标配
- 供应链验证（admission webhook 校验签名）越来越普及

### 12.5 Helm-and-Kustomize 不再"二选一"

主流大型项目（Cilium、Istio、Prometheus）都同时发布 Helm chart 与 Kustomize manifest。**用户选哪个看团队偏好**——但**别同一个项目两种都自己维护**，会漂移成两份不一致的版本。

### 12.6 替代/新兴

- **Carvel ytt + kapp**：vmware 出品，模板语法用 starlark（python-like），社区规模小但企业有信众
- **CDK8s**：用 TypeScript/Go/Python 写 K8s manifest——把"模板"换成"代码"。利基场景，没进入主流
- **Timoni**：基于 CUE，类型安全，目标取代 Helm。2024 起活跃，2026 仍是利基

主流"Helm + Kustomize"组合在 2026 没有真正威胁。

---

## 第十三章：练习题

**练习 1**：你正在写一个 chart，需要根据 `.Values.persistence.storageClass` 是空字符串还是非空字符串，决定 PVC 的 `spec.storageClassName` 字段是否输出。空字符串时**不输出**该字段（让 K8s 用 default SC）。写出 templates 片段。

**练习 2**：Helm chart 里有一个 ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yaml: |
    {{- .Values.appConfig | toYaml | nindent 4 }}
```

CI 渲染失败，报 "function 'toYaml' not defined"。问题在哪？

**练习 3**：在生产环境 `helm upgrade webapp ./chart -f values-prod.yaml` 后，新 Pod 一直起不来。`kubectl get pods` 看到老 Pod 还在，新 ReplicaSet 0/3 ready。如何排查？最快回滚？

**练习 4**：写一个 Kustomize overlay，把 base 里所有 Deployment 的 `replicas` 改为 5，所有 Service 的 `type` 改为 `LoadBalancer`，所有 ConfigMap 的 `data.LOG_LEVEL` 改为 `debug`。

**练习 5**：解释为什么 Kustomize 的 `configMapGenerator` 默认加 hash 后缀比手写 ConfigMap + `kubectl rollout restart` 更可靠。

**练习 6**：你有一个 chart，values 里 `image.tag` 默认空字符串。`Chart.yaml` 里 `appVersion: "2.6.0"`。templates 里写：

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```

用户用 `--set image.tag=""` 显式置空，期望 fallback 到 appVersion。但实际渲染出 `repository:`——为什么？怎么修？

**练习 7**：设计一个 GitOps 工作流，要求：每次合并到 main 自动 deploy 到 staging；prod 部署必须有人工 PR 审批（merge 后自动 sync）。给出 Argo CD Application 与 CI 的协作方案。

**练习 8**：解释 Helm hook 的 `pre-install` 与 Kubernetes 的 Init Container 在执行时机、失败语义、可重入性上的差异，以及各自适合的场景。

---

## 参考答案

**练习 1**：

```yaml
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.persistence.size }}
  {{- with .Values.persistence.storageClass }}
  storageClassName: {{ . | quote }}
  {{- end }}
```

`with` 在值为 falsy（空字符串、nil、空 list）时整段不渲染——天然解决"空值时不输出字段"。

**练习 2**：用了 `helm lint`/`helm template` 但用的是非 Helm 渲染器（如纯 `text/template`）。`toYaml` 是 Helm 通过 Sprig 注入的——纯 Go template 不带它。或者更常见的是：你在 Argo CD 的 `directory` 类型 Application 里直接放了带 Helm 模板语法的 YAML——Argo CD 不会 helm template 它。修：把 Application 改成 `helm` 类型，或确保用 `helm template` 渲染。

**练习 3**：

```bash
# 1. 看 ReplicaSet 状态
kubectl get rs -n prod
kubectl describe rs webapp-xxxxx -n prod

# 2. 看 Pod events
kubectl describe pod webapp-yyy -n prod
# 常见：ImagePullBackOff、CrashLoopBackOff、FailedScheduling

# 3. 看 Pod logs
kubectl logs webapp-yyy -n prod --previous

# 快速回滚：
helm rollback webapp -n prod
# 默认回滚到上一个 successful revision
# 或指定 revision
helm history webapp -n prod
helm rollback webapp 5 -n prod --wait --timeout 5m
```

如果 upgrade 是 `--atomic` 触发的，Helm 已自动回滚——只需查看 release 历史确认。

**练习 4**：

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: ""              # 空字符串 = 匹配所有 Deployment（实际 Kustomize 仍需 name）
    count: 5

# 实际写法：分别 patch
patches:
  - target:
      kind: Deployment
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
  - target:
      kind: Service
    patch: |-
      - op: replace
        path: /spec/type
        value: LoadBalancer

configMapGenerator:
  - name: ""              # ConfigMap data 修改不能用 patch（generator hash 会变）
    behavior: merge
    literals:
      - LOG_LEVEL=debug
```

ConfigMap 的批量 merge 在 Kustomize 里要靠 generator + behavior: merge——前提是知道 ConfigMap name。如果有多个 ConfigMap，要逐个写。

**练习 5**：

- 手写 ConfigMap：name 固定。改 data 后 ConfigMap 更新，但 Deployment 的 PodTemplate 不变 → Pod 不会重启 → 老 Pod 仍读旧值（除非通过 ConfigMap mount 自动 reload，且应用支持热加载）
- `configMapGenerator`：data 变 → hash 变 → ConfigMap name 变（如 `app-config-h7t8m2k9bd → app-config-c2f9e1a8d4`）→ Deployment env/volume 引用变更 → PodTemplate spec 变 → 自动 rolling restart
- 副作用："读到一致配置"是 Kubernetes 滚动更新天然保证的，不依赖应用热加载

**练习 6**：

`default` 函数对**空字符串**的判定是 falsy，所以 `"" | default .Chart.AppVersion` 应该返回 appVersion。**这其实正确——但**用户期望与现实可能脱节，比如他做了：

```yaml
image:
  tag: ""              # values-override.yaml
```

merge 后 `.Values.image.tag` 是空字符串——`default` 会 fallback。所以 `repository:` 不会出现。

实际"渲染出 repository:" 的原因可能是 image.tag 类型为 nil（不是空字符串）。比如 user 用 `--set image.tag=null`——这时 Helm 把它当作显式 nil，但 `toYaml` / 字符串模板可能渲染成 `<no value>` 或 `null`。

修：

```yaml
{{- $tag := default .Chart.AppVersion .Values.image.tag -}}
{{- if not $tag -}}
{{- fail "image.tag must be set or Chart.appVersion must be defined" -}}
{{- end -}}
image: "{{ .Values.image.repository }}:{{ $tag }}"
```

把 fallback 显式化，并用 `fail` 在 tag 仍为空时直接报错。

**练习 7**：

```
Git repo: webapp
├── deploy/
│   ├── overlays/
│   │   ├── staging/
│   │   └── prod/

Argo CD ApplicationSet:
  - webapp-staging:
      autoSync: true
      prune: true
  - webapp-prod:
      autoSync: true        # 自动 sync 但只 sync 已 merge 内容
      prune: true

GitHub branch protection:
  - main: require PR review (1+ approval), require status checks
  - 对 deploy/overlays/prod/** 路径加 CODEOWNERS（要求 SRE/lead 审批）

CI 流程:
  PR 阶段:
    - helm/kustomize lint
    - kubeconform / kyverno 校验
    - 渲染 diff 评论（用 argocd-diff-bot 或类似工具）
  Merge to main:
    - 写入 Git → Argo CD 检测 → 自动 sync staging
    - prod 文件夹改动也会自动 sync prod——但因为 PR 审批要求,所有 prod 改动必须经过 CODEOWNERS 审批
  手动控制 prod 节奏 (可选):
    - prod Application 设 autoSync: false + autoPrune
    - SRE 在 Argo CD UI 点击 Sync——或者 ApplicationSet 加 SyncWindow
```

关键点：**审批不在 Argo CD，在 GitHub PR**。Argo CD 只是把 Git 状态 → 集群状态——你能 merge 就能 sync。

**练习 8**：

| 维度 | Helm pre-install hook | Init Container |
|---|---|---|
| 执行时机 | install/upgrade 前一次性运行 | 每个 Pod 启动前运行（每次重启都跑） |
| 作用域 | 整个 release，一次执行 | 单个 Pod |
| 失败语义 | 失败 → upgrade 终止；若 atomic 则 rollback | 失败 → Pod 进入 CrashLoopBackOff，不影响其他 Pod |
| 可重入性 | 一次执行，多副本不会并发跑 | 每个 Pod 各自跑，会并发 |
| 适合场景 | DB schema migration、CRD 安装、配置初始化、license 注册（一次性） | 等待依赖（如等 DB 起来）、初始化本 Pod 文件系统、下载配置、注入 token |

DB migration 用 hook 安全（一次跑、不并发）；用 Init Container 危险（多副本并发跑 migration，可能锁冲突）。等待依赖（wait-for-db）反过来——用 Init Container（每个 Pod 都需要确认依赖）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Helm 三概念 | Chart（包）/ Release（实例）/ Repository（OCI 主流） |
| Helm 架构 | 客户端无服务端（Tiller 已废），release 元数据存 Secret |
| 模板语法 | Go template + Sprig；`{{- ... -}}` 吃空白；`nindent`、`toYaml`、`required` 三件套 |
| values 优先级 | chart 默认 < 父 chart 子节 < -f 文件 < --set |
| values schema | values.schema.json 把 typo 挡在 render 前 |
| OCI Chart | helm push/pull oci://，2026 主流；cosign 签名 |
| Kustomize 哲学 | 无模板、纯 YAML、声明式 patch |
| Kustomize 五件套 | base/overlays/patches/generators/transformers + components |
| hash 后缀 | configMapGenerator 改 data → hash 变 → 自动 rollout |
| Helm vs Kustomize | 模板派 vs 覆盖派；分发用 Helm、环境差异用 Kustomize |
| 混合用法 | kustomize build --enable-helm；helm chart + 末尾 patch |
| GitOps | Argo CD ApplicationSet 批量管理多环境 |
| CI 校验 | lint + template + kubeconform + kyverno + helm-diff |
| 生产纪律 | SemVer 严肃执行、CHANGELOG、secrets 用外部 ESO/sops |
| 陷阱 | 模板缩进、values list 覆盖、namespace 漂移、hook 残留 |
| 2026 现状 | Helm 3.14、Kustomize 5.x、Argo CD 2.x、OCI 全面接管 |

铁律：

- 对外分发的 chart 必有 values.schema.json + CHANGELOG + helm test
- 永远 `helm upgrade --install --atomic --wait` + 明确 `-n namespace`
- Secrets 不入 Git 明文（External Secrets / sops / Sealed Secrets）
- 子 chart 用 OCI 仓库，Chart.lock 入 Git
- Kustomize overlay 不要超过两层叠加（可读性 vs 灵活性的平衡）
- ConfigMap 用 generator + hash；只有"外部引用"场景才禁 hash
- GitOps 必有 CI 渲染 + diff 评论环节
- chart 升级前永远 `helm diff upgrade` 看变更
- 升级失败优先 rollback，事后再 debug；不要在 prod 反复试错
- Helm 与 Kustomize 同一项目挑一个主——另一个只做"末端微调"

下一篇 **C09 — 精通 Argo CD 与 Flux GitOps** 将拆开 GitOps 控制器的内部模型、ApplicationSet 高级用法、多集群分发、与渐进式发布（Argo Rollouts、Flagger）的协作。

---
