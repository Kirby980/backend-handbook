# 精通 ConfigMap、Secret 与配置管理：从 12-Factor 到 KMS 与 SPIFFE

> 课程编号：C05
> 路线图来源：云原生 · 模块二 流量与配置
> 难度：⭐⭐⭐
> 预计阅读时间：70 分钟
> 内容基准：2026 年 5 月（Kubernetes 1.33、External Secrets Operator 0.10+、Sealed Secrets v0.27+、Vault 1.18+、Reloader 1.x、SPIFFE/SPIRE 1.10+）

---

## 引言：12-Factor 与配置分离

```go
// 反模式 1：把数据库密码写死在代码里
const dbDSN = "postgres://admin:s3cret@10.0.0.5:5432/prod"

// 反模式 2：写在 Dockerfile 里
// ENV DB_PASSWORD=s3cret

// 反模式 3：写在 Deployment YAML 的 env 里（明文）
// - name: DB_PASSWORD
//   value: "s3cret"

// 正确：从环境变量读取，由 K8s 注入 Secret
dsn := os.Getenv("DATABASE_URL")
```

这三个反模式是 12-Factor App 第三条"配置存在环境里（Config in the environment）"的反面教材。Heroku 工程师 Adam Wiggins 在 2011 年写下 [12-Factor](https://12factor.net/) 时，云原生还没出现；但他立下的"严格分离代码与配置"的原则，今天每一个 Kubernetes 集群仍在执行。

**核心问题**：

- 配置随环境变（dev / staging / prod）——代码不能跟着改
- 密钥要轮换、要审计、要最小权限——明文存 Git 是合规事故
- 配置变更要能"重载"——重启整个集群代价太大
- 多团队协作下，开发者不该知道生产密钥

Kubernetes 给的三件套：

```
ConfigMap   ─── 非敏感配置（明文）
Secret      ─── 敏感数据（base64，etcd 可加密）
DownwardAPI ─── 自身元数据（pod name、namespace、labels）
```

但只有这三件还不够。2026 年的真实生产环境，配置管理是一个完整生态：

| 层 | 工具 | 解决什么 |
|---|---|---|
| 原生 | ConfigMap / Secret | 集群内最小单位 |
| 静态加密 | EncryptionConfiguration / KMS provider | etcd 落盘加密 |
| Git 化 | Sealed Secrets / SOPS | 把密文 commit 进 Git |
| 同步 | External Secrets Operator | 从 Vault / AWS SM / GCP SM 同步 |
| 动态 | Vault dynamic secrets | 短期凭证、按需生成 |
| 注入 | CSI Secrets Store / Vault Agent / SPIRE | 不落盘、运行时投递 |
| 身份 | SPIFFE / IRSA / Workload Identity | 用身份换 token，跳过"密钥换密钥" |
| 重载 | Reloader / 应用 watch | 配置变了 Pod 自动重启或热加载 |

本章把这个生态从底到顶拆开，并给一个 Go 后端工程师能落地的"最小生产工具箱"。

---

## 第一章：ConfigMap——明文配置的标准容器

### 1.1 资源结构

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  # 字符串键值对
  LOG_LEVEL: "info"
  FEATURE_FLAGS: "v2,new-billing"
  # 也可以是配置文件内容
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    cache:
      driver: redis
      addr: redis:6379
binaryData:
  # base64 编码的二进制内容
  truststore.jks: "MIIDdz..."
```

要点：

- `data` 字段是字符串到字符串的 map（**单个 ConfigMap 总大小 ≤ 1 MiB**，etcd 单 key 限制）
- `binaryData` 用于二进制（也算入总大小，base64 后膨胀 ~33%）
- `metadata.namespace` 隔离——ConfigMap 是 namespace scope，不能跨 ns 引用
- `immutable: true` 标记后不可改（kube-apiserver 拒绝 PATCH/UPDATE）——大幅降低 kubelet watch 压力，**生产推荐**对静态配置开启

### 1.2 创建方式

```bash
# 1. 从字面值
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=FEATURE_FLAGS=v2

# 2. 从单个文件（key 为文件名，value 为内容）
kubectl create configmap app-config --from-file=config.yaml

# 3. 从目录（每个文件成一个 key）
kubectl create configmap app-config --from-file=./configs/

# 4. 从 env 文件（每行一个 KEY=VALUE）
kubectl create configmap app-config --from-env-file=app.env

# 5. 声明式 YAML（CI/CD 推荐）
kubectl apply -f configmap.yaml
```

CI/CD 场景**永远用声明式**。命令式 create 只适合 dev / debug。

### 1.3 三种投递方式

```mermaid
flowchart LR
    CM["ConfigMap"]
    CM -->|envFrom / env| EV["容器环境变量"]
    CM -->|volumeMounts| FS["容器文件系统"]
    CM -->|projected volume| PV["projected 合并卷"]
    EV --> APP["Go 应用 os.Getenv"]
    FS --> APP2["Go 应用读文件"]
    PV --> APP3["Go 应用统一读"]

    style CM fill:#4299e1,color:#fff
```

#### 方式 1：env vars（环境变量）

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: LOG_LEVEL                # 取 ConfigMap 的某个 key
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
          optional: false            # 缺失则 Pod 启动失败（true 则空字符串）
    envFrom:                          # 一次性导入全部 key
    - configMapRef:
        name: app-config
        prefix: APP_                  # 可选：所有 key 加前缀
```

**特点**：

- **不会热更新**——容器启动时一次性注入，ConfigMap 改了进程内 env 不会变
- 想生效要重启 Pod
- 适合稳定不变的配置（日志级别、feature flag 静态版本）
- 注意 **POSIX 限制**：所有环境变量总长度（在 Linux 上）由 `ARG_MAX` 限制，大约 2 MiB；几百个 env 也可能超

#### 方式 2：volume mount（卷挂载）

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: config
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
      defaultMode: 0644
      items:                          # 可选：只暴露部分 key
      - key: config.yaml
        path: config.yaml
        mode: 0644
```

ConfigMap 的每个 key 变成 `/etc/app/<key>` 文件。

**特点**：

- **支持热更新**（默认每个节点 kubelet 周期同步，约 1 分钟级；可由 `--sync-frequency` 调）
- 写时复制——kubelet 把更新写到临时目录，再原子 symlink 切换；进程读到的总是一致快照
- 适合配置文件（YAML / JSON / properties）
- **只读**——容器写入会失败

#### 方式 3：projected volume（合并卷）

```yaml
volumes:
- name: combined
  projected:
    sources:
    - configMap:
        name: app-config
    - configMap:
        name: feature-flags
    - secret:
        name: tls-cert
    - downwardAPI:
        items:
        - path: namespace
          fieldRef:
            fieldPath: metadata.namespace
    - serviceAccountToken:           # 投递短期 SA token
        path: token
        audience: vault
        expirationSeconds: 3600
```

**特点**：

- 多源合并到同一个挂载点——避免大量 volumeMounts
- `serviceAccountToken` 投递是 IRSA / Workload Identity 的底座（详见第十二章）

### 1.4 热更新机制详解

```mermaid
sequenceDiagram
    participant K as kubectl/CI
    participant API as kube-apiserver
    participant ETCD as etcd
    participant KL as kubelet (Node)
    participant POD as Pod 文件系统

    K->>API: PATCH ConfigMap (新内容)
    API->>ETCD: 写入新 revision
    Note over KL: 周期 watch 或 sync (默认 ~60s)
    KL->>API: GET ConfigMap (检查 ResourceVersion)
    API->>KL: 新版本内容
    KL->>POD: 写入 ..data_xxx/ 临时目录
    KL->>POD: 原子 ln -sfn 切换 ..data symlink
    Note over POD: 容器读 /etc/app/config.yaml 始终一致
```

**关键事实**：

1. **ConfigMap 改了，env vars 不会变**——必须重启 Pod 进程才会读新值
2. **volume mount 会同步更新**——但有 ~60s 延迟（由 kubelet `--configMapAndSecretChangeDetectionStrategy` 与 `--sync-frequency` 控制）
3. **subPath 挂载不更新**——这是最常见的坑（见 1.5）
4. **immutable: true** 的 ConfigMap 改不了——只能重建（生产推荐，能减小 watch 压力）

让 Go 应用感知 volume 更新：

```go
import "github.com/fsnotify/fsnotify"

func watchConfig(path string, onChange func()) error {
    w, err := fsnotify.NewWatcher()
    if err != nil { return err }
    defer w.Close()
    // 监听挂载目录（注意：要监听目录而非文件本身——symlink 切换时文件 inode 变了）
    if err := w.Add("/etc/app"); err != nil { return err }
    for {
        select {
        case ev := <-w.Events:
            // kubelet 原子切换 ..data symlink，会产生 CREATE / REMOVE 事件
            if ev.Op&(fsnotify.Create|fsnotify.Remove) != 0 {
                onChange()
            }
        case err := <-w.Errors:
            return err
        }
    }
}
```

### 1.5 subPath 挂载——著名陷阱

```yaml
# 反模式：subPath 不会跟随 ConfigMap 热更新
volumes:
- name: config
  configMap:
    name: app-config
containers:
- volumeMounts:
  - name: config
    mountPath: /etc/nginx/nginx.conf   # 想覆盖单个文件
    subPath: nginx.conf                # ← 这一行让热更新失效
```

原理：subPath 是把 volume 内的"一个文件"硬链接到目标路径——它不是 symlink，因此 kubelet 后续对 `..data` symlink 的切换不会传递到 subPath 目标。

**修复方法**：

```yaml
# 方法 1：挂载整个目录，不用 subPath
volumeMounts:
- name: config
  mountPath: /etc/nginx          # 挂目录
  readOnly: true

# 方法 2：用 nginx 的 include
# 在镜像里写好基础 nginx.conf，include /etc/nginx/conf.d/*.conf
# ConfigMap 只投递 conf.d/ 下的文件，挂目录就行

# 方法 3：明知不热更，配合 Reloader 重启 Pod（见第八章）
```

**判断口诀**：subPath = 不热更。如果业务依赖热更，必须挂目录。

### 1.6 大配置怎么办

ConfigMap 单个 1 MiB 上限。超过怎么办？

```bash
# 方法 1：分片
configmap-a / configmap-b / configmap-c  # 各 <1MiB

# 方法 2：放对象存储 + initContainer 拉取
# 但失去了 K8s 原生的版本控制与 RBAC 优势

# 方法 3：用 CRD 自定义 + Operator 分发（C09）

# 方法 4：检查是不是真的需要这么大——把数据迁出到 DB
```

90% 情况下你不需要 1MiB 的配置。**配置过大通常是设计问题**。

---

## 第二章：Secret——加密版本的 ConfigMap

### 2.1 资源结构与编码

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-cred
  namespace: prod
type: Opaque
data:
  username: YWRtaW4=              # admin (base64)
  password: czNjcjN0           # s3cr3t (base64)
stringData:                       # 写时用 stringData，apply 后自动转 data
  api-key: sk-live-xxxxx
```

**关键事实**：

- `data` 的 value 是 **base64**，**不是加密**——任何能 `kubectl get secret -o yaml` 的人都能 decode
- `stringData` 是写时方便字段，apply 后会被服务器端转换到 `data`（base64）；读时永远是 `data`
- 单 Secret 也是 1 MiB 上限
- 真正的"加密"要靠 **etcd encryption at rest**（见第四章）或外部密钥管理（第六章）

### 2.2 Secret 类型一览

```yaml
type: Opaque                            # 默认通用类型
type: kubernetes.io/service-account-token  # SA token
type: kubernetes.io/dockerconfigjson    # 镜像仓库认证
type: kubernetes.io/dockercfg           # 旧版 docker auth
type: kubernetes.io/basic-auth          # username + password
type: kubernetes.io/ssh-auth            # SSH 私钥
type: kubernetes.io/tls                 # TLS 证书 + key
type: bootstrap.kubernetes.io/token     # 节点 bootstrap token
```

不同 `type` 强制要求 data 里包含某些 key（apiserver 校验）。

#### Opaque——通用

最常用，自由字段。

```yaml
type: Opaque
data:
  whatever-key: <base64>
```

#### kubernetes.io/tls——TLS 证书

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-prod
type: kubernetes.io/tls
data:
  tls.crt: <base64 of PEM cert chain>
  tls.key: <base64 of PEM private key>
```

`tls.crt` 与 `tls.key` 必填。被 Ingress / Gateway API listener / Pod sidecar 等消费。

```bash
# 创建快捷命令
kubectl create secret tls tls-prod \
  --cert=fullchain.pem --key=privkey.pem
```

#### kubernetes.io/dockerconfigjson——镜像仓库

```yaml
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64 of docker config json>
```

config json 内容：

```json
{
  "auths": {
    "registry.example.com": {
      "username": "ci",
      "password": "xxx",
      "email": "ci@example.com",
      "auth": "Y2k6eHh4"
    }
  }
}
```

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=ci \
  --docker-password=xxx \
  --docker-email=ci@example.com
```

```yaml
# Pod 引用
spec:
  imagePullSecrets:
  - name: regcred
```

也可以加在 ServiceAccount 上：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-sa
imagePullSecrets:
- name: regcred
```

`spec.serviceAccountName: deploy-sa` 的 Pod 自动继承。

#### kubernetes.io/service-account-token——SA token

```yaml
type: kubernetes.io/service-account-token
metadata:
  annotations:
    kubernetes.io/service-account.name: my-sa
```

**注意**：1.24 起 K8s **不再自动生成长期 SA token Secret**。Pod 里 `/var/run/secrets/kubernetes.io/serviceaccount/token` 现在是 **projected volume 投递的短期 token**（默认 1h 自动轮换）。如果你显式创建这种 Secret，是为了 long-lived token（CI 用）——**生产强烈建议不要**，改用 TokenRequest API + 短期 token 或 OIDC。

### 2.3 三种投递方式（同 ConfigMap）

```yaml
# env
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-cred
      key: password

# volume
volumes:
- name: secret
  secret:
    secretName: db-cred
    defaultMode: 0400               # 强烈建议 0400（只有 owner 可读）
volumeMounts:
- name: secret
  mountPath: /etc/secret
  readOnly: true

# projected
volumes:
- projected:
    sources:
    - secret:
        name: db-cred
```

**与 ConfigMap 的差异**：

- Secret 卷默认是 **tmpfs**（内存）挂载——不写盘，Pod 删除立即消失
- 同样支持热更新（非 subPath、~60s 延迟）
- env 方式注入的 Secret 在 `kubectl describe pod` 不会显示明文，但 `kubectl exec -- env` 会显示——**仍然不安全**
- 默认 mode 0644——但 Secret 建议 0400（见陷阱清单）

### 2.4 创建 Secret 的 N 种姿势

```bash
# 1. 命令式
kubectl create secret generic db-cred \
  --from-literal=username=admin \
  --from-literal=password=s3cr3t

# 2. 从文件
kubectl create secret generic tls --from-file=tls.crt --from-file=tls.key

# 3. 声明式 YAML（推荐用 stringData 写明文，避免手动 base64）
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-cred
type: Opaque
stringData:
  username: admin
  password: s3cr3t
EOF

# 4. 从 .env 文件
kubectl create secret generic app --from-env-file=.env.prod
```

**但永远不要**把第 3 步的明文 YAML commit 进 Git。后面章节会讲怎么"加密后再 commit"。

---

## 第三章：Pod 视角——从挂载到 Go 应用

### 3.1 Pod spec 完整示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      serviceAccountName: api-sa
      containers:
      - name: api
        image: api:v1.2.3
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: LOG_LEVEL
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        envFrom:
        - configMapRef:
            name: api-config
            optional: true
        volumeMounts:
        - name: app-config
          mountPath: /etc/app
          readOnly: true
        - name: db-cred
          mountPath: /etc/secrets
          readOnly: true
        - name: tls
          mountPath: /etc/tls
          readOnly: true
      volumes:
      - name: app-config
        configMap:
          name: api-config
      - name: db-cred
        secret:
          secretName: db-cred
          defaultMode: 0400
      - name: tls
        secret:
          secretName: tls-prod
```

### 3.2 Go 应用读取 env

```go
import (
    "log/slog"
    "os"
    "strconv"
    "time"
)

type Config struct {
    LogLevel    string
    Port        int
    DBPassword  string
    DialTimeout time.Duration
}

func loadFromEnv() Config {
    return Config{
        LogLevel:    getenv("LOG_LEVEL", "info"),
        Port:        getenvInt("PORT", 8080),
        DBPassword:  os.Getenv("DB_PASSWORD"),         // 不给默认值——空就是空
        DialTimeout: getenvDur("DIAL_TIMEOUT", 5*time.Second),
    }
}

func getenv(k, def string) string {
    if v := os.Getenv(k); v != "" { return v }
    return def
}
func getenvInt(k string, def int) int {
    if v := os.Getenv(k); v != "" {
        if i, err := strconv.Atoi(v); err == nil { return i }
    }
    return def
}
func getenvDur(k string, def time.Duration) time.Duration {
    if v := os.Getenv(k); v != "" {
        if d, err := time.ParseDuration(v); err == nil { return d }
    }
    return def
}
```

**生产铁律**：

- 密钥**永远从 env 或文件读**，不要 commit 到代码
- 启动时校验所有必需变量；缺失立即退出（fail-fast）
- 不要把 `os.Environ()` 整个 dump 进日志——env 里可能有别的服务的 Secret

### 3.3 Go 应用读 volume mount

```go
func loadFromFile(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil { return Config{}, err }
    var c Config
    if err := yaml.Unmarshal(data, &c); err != nil {
        return c, err
    }
    return c, nil
}

// Secret 也是文件：
dbPassword, err := os.ReadFile("/etc/secrets/password")
if err != nil { log.Fatal(err) }
// Trim trailing newline——kubectl 创建时不加 \n，手写 YAML 可能加
password := strings.TrimSpace(string(dbPassword))
```

### 3.4 viper 集成

```go
import "github.com/spf13/viper"

func loadConfig() (*Config, error) {
    v := viper.New()
    // 1. 默认值
    v.SetDefault("log_level", "info")
    v.SetDefault("port", 8080)

    // 2. 配置文件（ConfigMap 挂载到 /etc/app/config.yaml）
    v.SetConfigFile("/etc/app/config.yaml")
    if err := v.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, err
        }
    }

    // 3. 环境变量（优先级最高）
    v.SetEnvPrefix("APP")
    v.AutomaticEnv()
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

    // 4. 热更新
    v.OnConfigChange(func(e fsnotify.Event) {
        slog.Info("config reloaded", "path", e.Name)
        applyConfig(v)
    })
    v.WatchConfig()

    var c Config
    if err := v.Unmarshal(&c); err != nil { return nil, err }
    return &c, nil
}
```

注意 viper 的 `WatchConfig()` 用的是 fsnotify。前文提到过——**kubelet 是 symlink 切换**，所以 viper 在 ConfigMap 卷上工作得很好（默认就监听目录变化），但**在 subPath 挂载上不工作**。

### 3.5 配置优先级建议

```
启动参数 (--port=8080)        ← 最高（debug 用）
↓
环境变量 (PORT=8080)
↓
配置文件 (/etc/app/config.yaml)
↓
内置默认值 (port: 8080)        ← 最低
```

Go 工程师圈最常用的栈：

| 库 | 风格 | 何时选 |
|---|---|---|
| `os.Getenv` + flag | 标准库，零依赖 | 极简服务 |
| viper | 多源、热重载、Watch | 业务复杂度中等以上 |
| envconfig (kelseyhightower) | 结构体标签 + env | 纯 env 风格 |
| koanf | viper 替代，更现代 | 想去 viper |

---

## 第四章：etcd 加密——把 base64 变成真加密

### 4.1 默认状态：明文（base64）

```bash
# 在 master 上直接看 etcd——能看到明文
ETCDCTL_API=3 etcdctl get /registry/secrets/prod/db-cred --print-value-only
# 输出包含 base64 的 password
```

**这是默认行为**。kube-apiserver 把 Secret 序列化为 protobuf 后直接写 etcd。任何 etcd 读权限的人 = 拿到所有 Secret 明文。

修复方法：**EncryptionConfiguration**。

### 4.2 EncryptionConfiguration

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  - configmaps                  # 可选：连 configmaps 也加密
  providers:
  - aescbc:                     # 用 AES-CBC 加密
      keys:
      - name: key1
        secret: <base64 of 32 random bytes>
  - identity: {}                # 兜底：用上面解不开就当明文（用于迁移）
```

```bash
# 启用方式：kube-apiserver flag
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

**provider 类型**：

| provider | 算法 | 适用 |
|---|---|---|
| `identity` | 不加密 | 默认 / 兜底 |
| `aesgcm` | AES-GCM | 强，但需要频繁轮 key（≤ 200k 次写） |
| `aescbc` | AES-CBC | 弱于 GCM，无频繁轮 key 需求 |
| `secretbox` | XSalsa20+Poly1305 | 强、纯 Go 实现 |
| `kms` | KMS provider v2（推荐） | 外接 HSM / cloud KMS |

**生产推荐**：`kms` 或 `secretbox`。`aesgcm` 单 key 用太久会重复 nonce → 灾难。

### 4.3 KMS provider v2

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - kms:
      apiVersion: v2          # K8s 1.29 GA
      name: aws-kms
      endpoint: unix:///var/run/kmsplugin/socket.sock
      timeout: 3s
      cachesize: 1000         # 缓存解密的 DEK
```

**KMS v2 架构**：

```mermaid
sequenceDiagram
    participant API as kube-apiserver
    participant KP as kms-plugin
    participant CKMS as Cloud KMS (HSM)
    participant ETCD as etcd

    API->>KP: gRPC: Encrypt(DEK plaintext)
    KP->>CKMS: KMS.Encrypt with KEK
    CKMS-->>KP: encrypted DEK
    KP-->>API: encrypted DEK + key id
    Note over API: 用 DEK 加密 Secret，存入 etcd
    API->>ETCD: PUT secret (envelope encrypted)
```

**Envelope 加密**：每个 Secret 由一个 **DEK**（数据密钥）加密，DEK 本身由 **KEK**（来自云 KMS）加密。读取时 plugin 解开 DEK，apiserver 用 DEK 解 Secret。**KEK 永远不离开 HSM**。

主流云厂支持：

- AWS KMS（aws-encryption-provider）
- GCP KMS（cloud-kms-provider）
- Azure Key Vault
- HashiCorp Vault（kms-plugin-for-vault）

### 4.4 启用后的迁移

```bash
# 1. 部署 encryption-config，apiserver 重启
# 2. 重新加密所有现有 Secret（让它们走新 provider）
kubectl get secrets --all-namespaces -o json \
  | kubectl replace -f -
# 这一步把所有 Secret "读出再写回"——触发新的加密
```

未做这一步 → 老 Secret 仍是明文（因为 `identity` provider 兜底）。

### 4.5 验证

```bash
# 装上 encryption 后再去 etcd 看
ETCDCTL_API=3 etcdctl get /registry/secrets/prod/db-cred
# 应该是 k8s:enc:aescbc:v1:key1:<密文>...
```

如果还是 base64 → 没生效。

---

## 第五章：Sealed Secrets——把密文 commit 进 Git

### 5.1 痛点

GitOps（C14）要求所有声明（Deployment、Service、ConfigMap、Secret）都在 Git 里。但 Secret 是 base64 ≈ 明文——不能 commit。

Bitnami 的 **Sealed Secrets** 给的方案：

- 集群内运行一个 controller，持有 **私钥**
- 用户用 `kubeseal` 工具拿 controller 的 **公钥** 加密 Secret → `SealedSecret` CRD
- SealedSecret 是加密的，可以放 Git
- controller 看到 SealedSecret → 解密 → 创建对应的 Secret

```mermaid
graph LR
    Dev["开发者"] -->|kubeseal 公钥加密| SS["SealedSecret YAML"]
    SS -->|commit| Git[Git 仓库]
    Git -->|ArgoCD sync| K8s[K8s 集群]
    K8s --> CTL["sealed-secrets-controller"]
    CTL -->|用私钥解密| Sec["普通 Secret"]
    Sec --> Pod[Pod]

    style SS fill:#48bb78,color:#fff
    style CTL fill:#9f7aea,color:#fff
```

### 5.2 安装

```bash
# Helm
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets -n kube-system sealed-secrets/sealed-secrets

# kubeseal 客户端
brew install kubeseal     # mac
# 或下载 release binary
```

### 5.3 用法

```bash
# 1. 写普通 Secret YAML
cat > db-cred.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-cred
  namespace: prod
type: Opaque
stringData:
  password: s3cr3t
EOF

# 2. 用 kubeseal 加密
kubeseal --format=yaml < db-cred.yaml > db-cred.sealed.yaml
# 删掉明文
rm db-cred.yaml

# 3. commit sealed 文件
git add db-cred.sealed.yaml
git commit -m "add prod db-cred"
```

得到的 YAML 大致：

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-cred
  namespace: prod
spec:
  encryptedData:
    password: AgBh...long base64 ciphertext...
  template:
    metadata:
      name: db-cred
      namespace: prod
    type: Opaque
```

apply 后 controller 解密 → 生成 `Secret/db-cred`。

### 5.4 作用域（scope）

```bash
# strict（默认）：名字 + namespace + key 都参与加密，迁移困难
kubeseal --format yaml < s.yaml > sealed.yaml

# namespace-wide：同 ns 内可改 Secret 名字
kubeseal --scope=namespace-wide --format yaml < s.yaml > sealed.yaml

# cluster-wide：任意 ns 都能解密
kubeseal --scope=cluster-wide --format yaml < s.yaml > sealed.yaml
```

**生产建议**：默认 strict。除非有重命名 / 复制到多 ns 的明确需求，否则不要放松 scope。

### 5.5 私钥备份与轮换

```bash
# 备份私钥（极其重要！集群崩了你只剩这一根稻草）
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o yaml > master-key-backup.yaml

# 离线保存到密码管理器 / HSM / KMS-encrypted S3
```

私钥**永远不能**进 Git。controller 周期生成新 key（默认 30 天），新 Sealed 用新 key、老 Sealed 用老 key 解。轮 key 时不要删旧 key——会破坏现有 SealedSecret。

### 5.6 局限

- 只能加密**单个集群**的 Secret——多集群要分别 kubeseal
- 加密后 diff 不可读——review 时只能看 ciphertext
- 私钥泄漏 = 所有历史 Sealed 暴露
- 不支持密钥轮换的"自动同步"——业务密钥变了仍要重新 seal + commit

对**单集群 + 简单密钥**场景够用。多集群 / 大规模 → External Secrets。

---

## 第六章：External Secrets Operator——把"密钥源"放在 K8s 外

### 6.1 设计哲学

```
单一可信源 (Source of Truth)
   ↓
Vault / AWS Secrets Manager / GCP SM / Azure KV / 1Password / Doppler / ...
   ↓ pull
External Secrets Operator (集群内)
   ↓ create/update
普通 Kubernetes Secret
   ↓ mount/env
Pod
```

**优点**：

- 密钥**真正的源**在 Vault / 云 SM——支持轮换、审计、IAM
- 集群里只是"投影"——丢了重新拉
- Git 里不存任何密文——只存 `ExternalSecret` 引用
- 跨集群天然共享——每个集群配同一个 Vault store

### 6.2 安装

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

### 6.3 三种核心资源

```mermaid
graph LR
    SS["SecretStore<br>命名空间作用域"] -->|认证到| Backend["Vault / AWS SM / GCP SM"]
    CSS["ClusterSecretStore<br>集群作用域"] -->|认证到| Backend
    ES["ExternalSecret"] -->|声明拉取| SS
    ES2["ExternalSecret"] -->|声明拉取| CSS
    ES --> Sec1["K8s Secret"]
    ES2 --> Sec2["K8s Secret"]
    Sec1 --> Pod1[Pod]
    Sec2 --> Pod2[Pod]

    style Backend fill:#ed8936,color:#fff
    style ES fill:#48bb78,color:#fff
```

#### SecretStore：声明"密钥源"

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: vault-prod
  namespace: prod
spec:
  provider:
    vault:
      server: https://vault.example.com
      path: secret               # KV v2 路径前缀
      version: v2
      auth:
        kubernetes:              # 用 K8s SA 认证 Vault
          mountPath: kubernetes
          role: prod-role
          serviceAccountRef:
            name: api-sa
```

#### ExternalSecret：声明"我要哪些密钥"

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-cred
  namespace: prod
spec:
  refreshInterval: 1h            # 多久重新同步
  secretStoreRef:
    name: vault-prod
    kind: SecretStore
  target:
    name: db-cred                # 生成的 K8s Secret 名
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        DATABASE_URL: "postgres://{{ .username }}:{{ .password }}@db:5432/prod"
  data:
  - secretKey: username
    remoteRef:
      key: prod/db/cred          # Vault 路径
      property: username
  - secretKey: password
    remoteRef:
      key: prod/db/cred
      property: password
```

ESO 每 1h 从 Vault 拉一次 → 更新 `Secret/db-cred`。Pod volume 挂载会跟着热更新（~60s 延迟）。

#### ClusterSecretStore：跨 namespace 共享

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: vault-cluster
spec:
  provider:
    vault:
      server: ...
      auth:
        kubernetes:
          mountPath: kubernetes
          role: cluster-role
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

### 6.4 AWS Secrets Manager provider

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-sm
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: api-sa         # 使用 IRSA（详见第十二章）
```

ExternalSecret 引用：

```yaml
spec:
  data:
  - secretKey: api-key
    remoteRef:
      key: prod/api/openai       # AWS SM 中的 secret 名
      property: api_key          # 如果该 secret 存的是 JSON
```

### 6.5 PushSecret——反向同步

ESO 0.10+ 还支持反向：把 K8s Secret push 到 Vault。

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: PushSecret
metadata:
  name: push-tls
spec:
  refreshInterval: 1h
  secretStoreRefs:
  - name: vault-prod
    kind: SecretStore
  selector:
    secret:
      name: tls-prod             # K8s Secret 名
  data:
  - match:
      secretKey: tls.crt
      remoteRef:
        remoteKey: prod/tls/cert
        property: cert
```

适合 cert-manager 签发的证书自动 push 回 Vault 让其他系统消费。

### 6.6 与 Sealed Secrets 选型

| 维度 | Sealed Secrets | External Secrets |
|---|---|---|
| 密钥源 | Git 仓库（密文） | Vault / 云 SM |
| 轮换 | 手动 reseal | 改源即可（自动同步） |
| 多集群 | 每个集群独立 seal | 共享一个源 |
| 审计 | Git history | 源系统 audit log（更强） |
| 部署难度 | 单 controller，简单 | 需要管理 Vault / SM |
| 适合 | 小团队 / 单集群 | 中大型 / 多集群 / 强合规 |

**2026 年生产共识**：ESO 是主流。Sealed Secrets 在小项目或离线集群仍有市场。

---

## 第七章：Vault 动态密钥——短期凭证

### 7.1 静态 vs 动态

```
静态密钥：admin / s3cr3t（手动创建，长期不变）
   问题：泄漏后影响面巨大；轮换难
   
动态密钥：每次按需生成（如 db_user_app_1715680000 / 临时密码）
   优点：默认短期（minutes ~ hours）、按身份发放、自动 revoke
```

### 7.2 Vault Database 引擎

配置 Vault：

```bash
# 启用 database secrets engine
vault secrets enable database

# 配置 PostgreSQL 连接
vault write database/config/prod-pg \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@pg:5432/prod?sslmode=disable" \
  allowed_roles="app-role" \
  username="vault-admin" \
  password="vault-admin-pass"

# 定义 role
vault write database/roles/app-role \
  db_name=prod-pg \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

应用拿凭证：

```bash
vault read database/creds/app-role
# Key            Value
# lease_id       database/creds/app-role/abc123
# lease_duration 1h
# username       v-token-app-role-xyz
# password       random-strong-pw
```

每次 read 都返回**新的** username/password（Vault 在 DB 里 CREATE 真实用户），1h 后自动 revoke。

### 7.3 ESO + Vault 动态密钥

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-dynamic
spec:
  refreshInterval: 30m           # 比 lease 短，提前轮换
  secretStoreRef:
    name: vault-prod
    kind: SecretStore
  target:
    name: db-cred
  dataFrom:
  - extract:
      key: database/creds/app-role   # ← 注意这是 creds 路径，每读一次发新凭证
```

应用每 30 分钟会拿到新的 username/password。Pod volume mount 在 ~60s 内同步——但 **Go 程序里的 DB 连接池怎么办**？

### 7.4 Go 应用配合连接池

```go
import (
    "context"
    "database/sql"
    "github.com/fsnotify/fsnotify"
    _ "github.com/lib/pq"
)

type DynamicDB struct {
    mu  sync.RWMutex
    db  *sql.DB
}

func (d *DynamicDB) reload(credPath string) error {
    user, _ := os.ReadFile(filepath.Join(credPath, "username"))
    pass, _ := os.ReadFile(filepath.Join(credPath, "password"))
    dsn := fmt.Sprintf("postgres://%s:%s@db:5432/prod", strings.TrimSpace(string(user)), strings.TrimSpace(string(pass)))

    newDB, err := sql.Open("postgres", dsn)
    if err != nil { return err }
    if err := newDB.PingContext(context.Background()); err != nil {
        newDB.Close()
        return err
    }

    d.mu.Lock()
    old := d.db
    d.db = newDB
    d.mu.Unlock()

    if old != nil {
        // 给在飞请求 5 秒优雅 drain
        go func() { time.Sleep(5*time.Second); old.Close() }()
    }
    return nil
}

func (d *DynamicDB) DB() *sql.DB {
    d.mu.RLock()
    defer d.mu.RUnlock()
    return d.db
}
```

**重要原则**：

- 应用 **不持有长期 DSN**——每次取 `d.DB()`
- 旧 pool 不立即 close——给在飞 query 时间完成
- 重载失败保留旧 pool 继续服务

### 7.5 Vault Agent Injector——sidecar 方案

不想改应用代码？Vault Agent Injector 用 webhook 给 Pod 自动注入一个 sidecar / initContainer：

```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "app-role"
    vault.hashicorp.com/agent-inject-secret-db: "database/creds/app-role"
    vault.hashicorp.com/agent-inject-template-db: |
      {{- with secret "database/creds/app-role" -}}
      DATABASE_URL="postgres://{{ .Data.username }}:{{ .Data.password }}@db:5432/prod"
      {{- end }}
```

Vault Agent 把凭证写到 `/vault/secrets/db`，Pod 用 `source /vault/secrets/db` 读。Lease 续期由 Agent 自动处理。

### 7.6 CSI Secrets Store

更现代的方案：CSI 驱动从 Vault / 云 SM 拉密钥，挂载到 Pod 不落 etcd。

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db
spec:
  provider: vault
  parameters:
    vaultAddress: https://vault.example.com
    roleName: app-role
    objects: |
      - objectName: "username"
        secretPath: "database/creds/app-role"
        secretKey: "username"
      - objectName: "password"
        secretPath: "database/creds/app-role"
        secretKey: "password"
```

```yaml
volumes:
- name: secrets-store
  csi:
    driver: secrets-store.csi.k8s.io
    readOnly: true
    volumeAttributes:
      secretProviderClass: vault-db
```

**优点**：密钥**不进 etcd**——不需要 etcd encryption；不需要 ESO 同步出一个 K8s Secret。

---

## 第八章：配置变更与 Pod 重启策略

### 8.1 三种态度

```
配置变了，怎么办？

策略 A：什么都不做（让 env / mount 滞后）
策略 B：手动滚动 (kubectl rollout restart deployment/api)
策略 C：自动（Reloader 或应用自己 watch）
```

### 8.2 应用层热重载（推荐）

最优雅的方案：应用 watch 配置文件，热加载。

```go
// 完整的"配置原子替换"模式
type Config struct {
    LogLevel string `yaml:"log_level"`
    // ...
}

type ConfigStore struct {
    cur atomic.Pointer[Config]
}

func (s *ConfigStore) Get() *Config { return s.cur.Load() }
func (s *ConfigStore) Set(c *Config) { s.cur.Store(c) }

func (s *ConfigStore) Watch(path string) error {
    s.reload(path)
    w, _ := fsnotify.NewWatcher()
    dir := filepath.Dir(path)
    w.Add(dir)
    go func() {
        for {
            select {
            case ev := <-w.Events:
                if ev.Op&(fsnotify.Create|fsnotify.Remove|fsnotify.Write) != 0 {
                    if err := s.reload(path); err != nil {
                        slog.Error("reload failed", "err", err)
                    }
                }
            case err := <-w.Errors:
                slog.Error("fsnotify error", "err", err)
            }
        }
    }()
    return nil
}

func (s *ConfigStore) reload(path string) error {
    data, err := os.ReadFile(path)
    if err != nil { return err }
    var c Config
    if err := yaml.Unmarshal(data, &c); err != nil { return err }
    if err := validate(&c); err != nil { return err }   // 校验失败 → 保留旧
    s.Set(&c)
    slog.Info("config reloaded", "version", c.Version)
    return nil
}
```

**铁律**：

- 用 atomic.Pointer，多 goroutine 安全
- reload 失败保留旧配置（不要置 nil）
- 校验通过才替换
- 监听**目录**而非文件（ConfigMap symlink 切换）

但**有些配置不能热重载**：

- 监听端口（要重新 listen）
- 数据库连接参数（连接池要重建）
- 内存中的限流桶大小（要重置桶）

这类配置只能重启 Pod。

### 8.3 Reloader——自动重启

[Stakater Reloader](https://github.com/stakater/Reloader) 是个 controller：watch ConfigMap / Secret 变化，自动触发依赖它的 Deployment / StatefulSet rolling restart。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  annotations:
    # 任一关联 ConfigMap 改了就重启
    configmap.reloader.stakater.com/reload: "api-config"
    # 任一关联 Secret 改了就重启
    secret.reloader.stakater.com/reload: "db-cred"
    # 或者监控全部
    # reloader.stakater.com/auto: "true"
```

Reloader 把 ConfigMap/Secret 的 hash 写到 Pod template 的 annotation → Deployment 控制器看到 spec 变化 → 滚动更新。

### 8.4 手动滚动

```bash
# 立即触发 rolling restart（保留 spec 不变，只重启 Pod）
kubectl rollout restart deployment/api -n prod

# 监控滚动状态
kubectl rollout status deployment/api -n prod

# 回滚到上一版（如果新配置坏了）
kubectl rollout undo deployment/api -n prod
```

`rollout restart` 1.15+ 原生支持——它给 Pod template 加一个 `kubectl.kubernetes.io/restartedAt` annotation 触发滚动。

### 8.5 GitOps 集成

```yaml
# ArgoCD Application 监控 Git
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api
spec:
  source:
    repoURL: https://github.com/org/configs
    path: prod
  destination:
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Git 改 ConfigMap → ArgoCD 同步 → ConfigMap 更新 → Reloader 触发 rolling restart。**全自动**。这是 C14 GitOps 的核心闭环。

---

## 第九章：多环境配置——Kustomize 与 Helm

### 9.1 痛点：dev / staging / prod 差异

```
- 同一个应用，3 个环境
- 每个环境：不同 replicas、image tag、资源限额、ConfigMap、Secret
- 90% 的 YAML 是一样的
- 不能复制三份维护
```

### 9.2 Kustomize overlay

```
config-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── configmap-patch.yaml
    ├── staging/
    │   └── ...
    └── prod/
        └── ...
```

**base/configmap.yaml**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: info
  CACHE_TTL: "300"
```

**overlays/prod/kustomization.yaml**：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources:
- ../../base
configMapGenerator:
- name: api-config
  behavior: merge
  literals:
  - LOG_LEVEL=warn
  - CACHE_TTL=3600
images:
- name: myapp
  newTag: v1.2.3-prod
```

```bash
kustomize build overlays/prod | kubectl apply -f -
# 或
kubectl apply -k overlays/prod
```

**亮点**：

- 不用模板（不会因为模板 bug 跪）
- configMapGenerator **自动加 hash 后缀**（如 `api-config-7d8h4f`）→ ConfigMap 变了名字变了 → Deployment template 引用变了 → 自动 rolling restart！**无需 Reloader**！

### 9.3 Helm values

```
chart/
├── Chart.yaml
├── values.yaml              # 默认值
├── values-dev.yaml
├── values-staging.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── configmap.yaml
    └── secret.yaml
```

**templates/configmap.yaml**：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
data:
  LOG_LEVEL: {{ .Values.logLevel | quote }}
  CACHE_TTL: {{ .Values.cache.ttl | quote }}
  config.yaml: |
    {{- toYaml .Values.appConfig | nindent 4 }}
```

**values-prod.yaml**：

```yaml
logLevel: warn
cache:
  ttl: 3600
appConfig:
  server:
    port: 8080
    timeout: 30s
```

```bash
helm install api ./chart -f values-prod.yaml -n prod
helm upgrade api ./chart -f values-prod.yaml -n prod
```

Helm 默认 ConfigMap 改了**不会**触发滚动——常见模式是：

```yaml
# Deployment template 加 checksum annotation
metadata:
  annotations:
    checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

或上 Reloader / 用 helm-secrets plugin 集成 SOPS。

### 9.4 Helm + ESO 的"密钥分离"

Helm chart 里**不放 Secret**：

```yaml
# templates/externalsecret.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: {{ .Release.Name }}-db
spec:
  secretStoreRef:
    name: vault-{{ .Values.env }}
    kind: ClusterSecretStore
  target:
    name: {{ .Release.Name }}-db
  data:
  - secretKey: password
    remoteRef:
      key: {{ .Values.env }}/db/cred
      property: password
```

```yaml
# values-prod.yaml
env: prod
```

环境差异只是 `env: prod` vs `env: staging` 这一个字符——其他全部由 Vault 路径自动隔离。**这是 2026 年主流"配置最佳实践"**。

---

## 第十章：生产实践

### 10.1 密钥轮换

```mermaid
flowchart LR
    A["密钥进入"] --> B["每 90 天<br>(或更频繁) 轮换"]
    B --> C["新密钥写入 Vault"]
    C --> D["ESO 同步到集群"]
    D --> E["应用热加载 / 重启"]
    E --> F["验证新密钥工作"]
    F --> G["Vault revoke 旧密钥"]
    G --> H["审计 log 留存"]

    style B fill:#ed8936,color:#fff
    style E fill:#48bb78,color:#fff
    style G fill:#f56565,color:#fff
```

**双密钥窗口**（无停机轮换）：

1. Vault 同时持有 `current` + `previous` 两份密钥
2. 应用同时尝试两份（rotate-ready 客户端）
3. ESO 写 K8s Secret 同时包含 `password.current` 与 `password.previous`
4. 新 Pod 起来后用 current；旧连接继续用 previous
5. 等所有连接迁移完，revoke previous

数据库等支持双密码的（如 PostgreSQL 配 `pg_hba.conf` 行 + 多次 `ALTER USER`）能做到完全零停机。

### 10.2 最小权限（RBAC）

```yaml
# 只允许特定 SA 读特定 Secret
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: api-secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-cred", "api-keys"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: prod
  name: api-secret-reader
subjects:
- kind: ServiceAccount
  name: api-sa
roleRef:
  kind: Role
  name: api-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**铁律**：

- **不要** `verbs: ["list", "watch"]` Secret——会暴露所有 Secret 内容（包括别人的）
- **不要** namespace 全权限给应用 SA
- Pod 不需要的 Secret 别注入
- 集群管理员的 Secret 操作要走 break-glass + 审计

### 10.3 审计与监控

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["create", "update", "patch", "delete"]
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

`RequestResponse` 级把请求体（含 Secret 明文！）写进 audit log——所以 audit log **本身要加密 + 受限访问**。`Metadata` 级只记请求元数据，避免明文泄漏。

**监控信号**：

- `apiserver_request_total{resource="secrets", verb="get"}` 异常突增 → 可能有人 dump
- ESO controller 失败次数（external-secrets metrics）
- Sealed Secrets 解密失败 → 私钥不一致或被破坏
- Vault audit log → 异常凭证请求模式

### 10.4 镜像扫描里的 secret leakage

```bash
# trivy 也能扫到镜像里的明文密钥
trivy image --scanners secret myapp:v1
# Found 1 secret(s)
# - /app/.env: AWS Access Key
```

**所以** Dockerfile 里**永远不要**：

```dockerfile
# 反模式
COPY .env /app/.env
ENV AWS_SECRET_KEY=xxx
RUN echo "password=xxx" > /etc/cfg
```

正确做法：

```dockerfile
# 基础镜像不带任何 secret
COPY app /app/app
# 运行时由 K8s 注入
```

CI 阶段把 `trivy fs --scanners secret` 作为强制门禁。

### 10.5 break-glass 流程

紧急情况（生产 secret 紧急轮换 / 调试时需要明文）需要"破玻璃"：

```
1. 申请：开 ticket，标记 break-glass + 业务 leader 审批
2. 时窗：临时给 RBAC（5 分钟，最多 1 小时）
3. 操作：在 break-glass 窗口拿密钥
4. 审计：所有操作 audit-log + Slack 通知
5. 撤销：自动到期撤权
6. 复盘：事后 review
```

可以用 Vault 的 [break-glass policy](https://developer.hashicorp.com/vault/docs/concepts/policies#break-glass) 或自建短期 token 颁发系统。

### 10.6 完整 Provider 接口（Go）

```go
// 一个生产可用的 secret 抽象层
package secret

import "context"

type Provider interface {
    Get(ctx context.Context, key string) (string, error)
    GetMulti(ctx context.Context, keys []string) (map[string]string, error)
    OnChange(key string, fn func(newValue string)) func()  // 返回 unsubscribe
    Close() error
}

// 文件实现（从 mount path 读）
type FileProvider struct {
    root    string
    watcher *fsnotify.Watcher
    subs    map[string][]func(string)
    mu      sync.Mutex
}

func (p *FileProvider) Get(_ context.Context, key string) (string, error) {
    data, err := os.ReadFile(filepath.Join(p.root, key))
    if err != nil { return "", err }
    return strings.TrimSpace(string(data)), nil
}

// 直接 K8s API 实现（如果 Pod 没挂载、要 client-go 直读）
type APIProvider struct {
    client    kubernetes.Interface
    namespace string
    name      string
    cached    *corev1.Secret
    mu        sync.RWMutex
}

func (p *APIProvider) Get(ctx context.Context, key string) (string, error) {
    p.mu.RLock()
    s := p.cached
    p.mu.RUnlock()
    if s == nil {
        var err error
        s, err = p.client.CoreV1().Secrets(p.namespace).Get(ctx, p.name, metav1.GetOptions{})
        if err != nil { return "", err }
        p.mu.Lock()
        p.cached = s
        p.mu.Unlock()
    }
    if v, ok := s.Data[key]; ok { return string(v), nil }
    return "", fmt.Errorf("key not found: %s", key)
}
```

应用 main：

```go
secretProvider, err := secret.NewFileProvider("/etc/secrets")
if err != nil { log.Fatal(err) }
defer secretProvider.Close()

dbURL, err := secretProvider.Get(ctx, "DATABASE_URL")
if err != nil { log.Fatal(err) }

// 热更新
unsub := secretProvider.OnChange("DATABASE_URL", func(newVal string) {
    db.Reload(newVal)
})
defer unsub()
```

---

## 第十一章：陷阱清单

### 1. subPath 不热更

```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/nginx/nginx.conf
  subPath: nginx.conf            # ← ConfigMap 改了不会同步！
```

修：挂目录而非单文件。

### 2. env var dump 泄漏

```go
// 反模式
for _, e := range os.Environ() {
    log.Println(e)                 // 会把所有 Secret 明文打进日志！
}
```

修：白名单输出，或用 `sigs.k8s.io/yaml`+redaction。

### 3. Secret 默认 0644 mode

```yaml
# 默认 mode 0644 = 其他容器用户也可读
volumes:
- name: secret
  secret:
    secretName: db
    # defaultMode 没指定 = 0644
```

修：

```yaml
secret:
  secretName: db
  defaultMode: 0400               # 只 owner 读
```

### 4. 单 Secret 超过 1 MiB

```bash
Error: secrets "huge" is invalid: data: Too long: must have at most 1048576 bytes
```

修：拆分；或本就不该放 K8s（如证书 chain 太长 → 改 cert-manager）。

### 5. cache key 因毫秒变化 → 重启风暴

```yaml
data:
  config.yaml: |
    generated_at: 2026-05-14T10:23:45.123Z   # ← 每次 CI 都不一样
```

ArgoCD 看到 ConfigMap 总在变 → 总在 sync → 频繁触发 rolling restart。

修：CI 不要往配置里写 timestamp / commit hash（确实要的话，放到 annotation 而非 data，或 Pod 启动时读 downward API）。

### 6. Helm 升级 Secret 不变 → 应用不重启

Helm 默认不滚动重启 Deployment——除非 spec 真变了。Secret 内容变 ≠ Deployment 变。

修：

- 用 checksum annotation（前述）
- 用 Reloader
- 用 Kustomize secretGenerator（带 hash 后缀）

### 7. RBAC 给了 list secrets

```yaml
verbs: ["list"]                  # ← 拿一份 list 就把所有 Secret data 拖走了
```

K8s 的 list 默认返回完整对象（包括 data）。

修：only `get` 必要的具体 Secret。

### 8. SA token 写进 image

```dockerfile
# 反模式
COPY service-account.json /app/
```

修：用 IRSA / Workload Identity / projected SA token，让 K8s 在运行时投递。

### 9. EncryptionConfiguration 改了忘了 rewrite

```bash
# 启用 KMS 后老 Secret 还是 base64
kubectl get secret old-secret -o yaml      # 仍是 identity 加密（即明文）
```

修：apply 后**必须**重写所有 Secret（前述 `kubectl get secret -o json | kubectl replace -f -`）。

### 10. Sealed Secrets 私钥丢失

```
controller pod 重建丢了 key Secret → 所有历史 SealedSecret 永远解不开
```

修：备份 master key 到离线安全位置。

### 11. ESO refreshInterval 太短

```yaml
refreshInterval: 10s             # 每 10s 拉一次 Vault → Vault 限流爆炸
```

修：30m ~ 1h 是合理区间；动态密钥按 lease 的 1/2 设置。

### 12. 配置变更没做 canary

```bash
kubectl apply -f new-config.yaml  # 直接 prod 全量
# new config 有 bug → 全 Pod 崩
```

修：

- 用 ArgoCD Sync Wave 分阶段
- 用 Argo Rollouts 把 ConfigMap 关联 deploy 走金丝雀
- 在 staging 充分验证

### 13. immutable 之后没法改

```yaml
immutable: true
```

修：必须先 delete 再 create——或预知会变的不要打 immutable。

### 14. envFrom 覆盖 PATH

```yaml
envFrom:
- configMapRef:
    name: app-config             # 里面有个 key 叫 PATH = /custom/bin
```

容器的 PATH 被覆盖 → `/bin/sh` 找不到。修：不要在 ConfigMap 里放跟系统 env 重名的 key（PATH / HOME / USER 等）。

### 15. CRD 里塞密钥

```yaml
# 反模式
apiVersion: example.com/v1
kind: MyApp
spec:
  dbPassword: s3cr3t              # ← CRD spec 不加密，等于明文
```

修：CRD 引用 SecretRef，由 Operator 在调谐时去读 Secret。

---

## 第十二章：2026 现状——SPIFFE、IRSA 与 Workload Identity

### 12.1 从"密钥"到"身份"的演进

```
2010s: 应用持有静态密钥 → 泄漏即灾难
   ↓
2020s 初: K8s Secret + etcd 加密 → 集群内是密文，但应用代码仍认密钥
   ↓
2020s 中: 短期凭证（SA token、Vault 动态）→ 期限内有效，过期自动失效
   ↓
2025+: 工作负载身份（SPIFFE / IRSA / WI）→ 应用无需任何密钥，凭"身份"换 token
```

### 12.2 IRSA（AWS）

**IAM Roles for Service Accounts**：把 K8s SA 映射到 IAM Role，Pod 用 SA token 换 AWS STS token。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/api-prod-role
```

EKS 自动给绑定该 SA 的 Pod 注入：

- `AWS_ROLE_ARN` 环境变量
- `AWS_WEB_IDENTITY_TOKEN_FILE` 环境变量（指向 projected SA token）

Go 应用零密钥：

```go
import "github.com/aws/aws-sdk-go-v2/config"

cfg, err := config.LoadDefaultConfig(context.Background())
// SDK 自动用 AssumeRoleWithWebIdentity 换 AWS 凭证！
```

**好处**：Pod 完全不持有 AWS access key / secret。审计粒度到"哪个 Pod 在哪个 SA 下做了什么"。

### 12.3 Workload Identity（GCP）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  annotations:
    iam.gke.io/gcp-service-account: api-prod@project.iam.gserviceaccount.com
```

```bash
gcloud iam service-accounts add-iam-policy-binding \
  api-prod@project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:project.svc.id.goog[default/api-sa]"
```

效果同 IRSA。Pod 用 GCP SDK 时无需密钥文件。

### 12.4 SPIFFE / SPIRE——跨云通用身份

SPIFFE（**S**ecure **P**roduction **I**dentity **F**ramework **F**or **E**veryone）是 CNCF 项目，给工作负载发放标准身份（SPIFFE ID + SVID 证书）：

```
spiffe://prod.example.com/ns/prod/sa/api-sa
```

SPIRE 是 SPIFFE 的参考实现。架构：

```mermaid
graph LR
    Agent["SPIRE Agent<br>(DaemonSet)"] --> Workload[Pod 通过 Workload API]
    Server["SPIRE Server"] -->|配置 attestation| Agent
    Workload -->|拿 SVID| Service["调用其他服务<br>(mTLS)"]

    style Server fill:#4299e1,color:#fff
    style Agent fill:#9f7aea,color:#fff
```

Pod 通过 unix socket 调 Workload API 拿到 X.509 SVID 证书（短期），用它做 mTLS 双向认证。**完全无 Secret**。

Go 应用：

```go
import "github.com/spiffe/go-spiffe/v2/workloadapi"

src, err := workloadapi.NewX509Source(ctx)
if err != nil { log.Fatal(err) }
defer src.Close()

// 创建 mTLS 服务器
server := &http.Server{
    Addr: ":8443",
    TLSConfig: tlsconfig.MTLSServerConfig(src, src, tlsconfig.AuthorizeAny()),
}
```

**2026 现状**：

- AWS / GCP 在自己生态内 IRSA / WI 已是默认
- 跨云 / 多云 / 混合云 → SPIFFE 成为事实标准
- Istio Ambient（C10）底层就用 SPIFFE 做 mTLS

### 12.5 Secretless Architecture

终极愿景：

```
应用 ── 持有身份（SA / SPIFFE） ── 换取临时 token / 证书 ── 调用 API
       (零静态密钥)
```

| 资源 | 传统 | Secretless |
|---|---|---|
| AWS API | access_key / secret_key | IRSA |
| GCP API | service-account.json | Workload Identity |
| Azure | client_id / client_secret | Azure AD Workload Identity |
| 数据库 | username / password | Vault dynamic creds / RDS IAM auth |
| 内部 API | shared token | mTLS (SPIFFE) |
| Vault | static token | K8s auth backend |

**2026 年成熟度**：云厂内 90% 场景已可 secretless。剩余 10% 是历史包袱与第三方 SaaS（很多 SaaS 仍只支持 long-lived API key）。

### 12.6 OIDC discovery

SA token 在 1.20+ 起就是 **OIDC JWT**——任何能信任 K8s OIDC issuer 的系统都能直接验证它，无需共享密钥。

```bash
# 集群 issuer
kubectl get --raw /.well-known/openid-configuration | jq

# 公钥
kubectl get --raw /openid/v1/jwks | jq
```

这让"K8s SA → 任意外部系统"的身份传递成为可能（如 GitHub Actions OIDC → AWS、Vault Kubernetes auth、Google IAM）。

---

## 第十三章：把所有东西串起来——一个完整例子

业务场景：Go 后端 API 服务，部署到 EKS prod 环境。

需求：

- 应用读 ConfigMap 拿日志等级、cache 配置
- 数据库密码用 Vault 动态密钥
- 第三方 API key 来自 AWS Secrets Manager
- 镜像从 ECR 拉取（IRSA）
- TLS 证书由 cert-manager 签发
- 配置 / 密钥变更自动滚动

### 13.1 资源拓扑

```mermaid
graph TB
    Vault["Vault<br>(动态 DB creds)"]
    AWS["AWS Secrets Manager<br>(第三方 API key)"]
    CM[ConfigMap]
    ESO[External Secrets Operator]
    Cert["cert-manager"]
    ACME["Let's Encrypt"]

    Vault -->|ExternalSecret 同步| Sec1[Secret/db-cred]
    AWS -->|ExternalSecret 同步| Sec2[Secret/api-keys]
    Cert -->|签发| Sec3[Secret/tls]
    ACME -->|ACME challenge| Cert

    CM --> Pod
    Sec1 --> Pod
    Sec2 --> Pod
    Sec3 --> Pod

    SA[ServiceAccount<br>annotated with IRSA] --> Pod
    SA -->|AssumeRoleWithWebIdentity| AWS

    style Vault fill:#ed8936,color:#fff
    style AWS fill:#4299e1,color:#fff
    style ESO fill:#9f7aea,color:#fff
```

### 13.2 完整 YAML

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/api-prod
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: prod
data:
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
    log:
      level: info
    cache:
      driver: redis
      ttl: 3600
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-cred
  namespace: prod
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault-prod
    kind: SecretStore
  target:
    name: db-cred
  dataFrom:
  - extract:
      key: database/creds/api-role
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: api-keys
  namespace: prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-sm
    kind: SecretStore
  target:
    name: api-keys
  data:
  - secretKey: openai
    remoteRef:
      key: prod/api/openai
      property: api_key
  - secretKey: stripe
    remoteRef:
      key: prod/api/stripe
      property: secret_key
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls
  namespace: prod
spec:
  secretName: tls
  dnsNames: [api.example.com]
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: prod
  annotations:
    configmap.reloader.stakater.com/reload: "api-config"
    secret.reloader.stakater.com/reload: "db-cred,api-keys"
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      serviceAccountName: api-sa
      containers:
      - name: api
        image: 123.dkr.ecr.us-east-1.amazonaws.com/api:v1.2.3
        ports: [{ containerPort: 8080 }]
        envFrom:
        - secretRef:
            name: db-cred
        - secretRef:
            name: api-keys
        volumeMounts:
        - name: config
          mountPath: /etc/app
          readOnly: true
        - name: tls
          mountPath: /etc/tls
          readOnly: true
        resources:
          requests: { cpu: 100m, memory: 256Mi }
          limits: { cpu: 1, memory: 1Gi }
      volumes:
      - name: config
        configMap:
          name: api-config
      - name: tls
        secret:
          secretName: tls
          defaultMode: 0400
```

### 13.3 应用 main.go

```go
package main

import (
    "context"
    "database/sql"
    "log/slog"
    "os"
    "path/filepath"
    "sync/atomic"
    "time"

    _ "github.com/lib/pq"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/fsnotify/fsnotify"
    "gopkg.in/yaml.v3"
)

type AppConfig struct {
    Server struct {
        Port    int           `yaml:"port"`
        Timeout time.Duration `yaml:"timeout"`
    } `yaml:"server"`
    Log struct {
        Level string `yaml:"level"`
    } `yaml:"log"`
    Cache struct {
        Driver string `yaml:"driver"`
        TTL    int    `yaml:"ttl"`
    } `yaml:"cache"`
}

var (
    cfg    atomic.Pointer[AppConfig]
    dbPool atomic.Pointer[sql.DB]
)

func main() {
    ctx := context.Background()
    
    // 1. 加载配置（ConfigMap mount）
    if err := loadConfig("/etc/app/config.yaml"); err != nil {
        slog.Error("load config failed", "err", err); os.Exit(1)
    }
    go watchFiles("/etc/app", reloadConfig)
    
    // 2. 数据库（ESO 同步的 Vault 动态密钥）
    if err := reloadDB(); err != nil {
        slog.Error("init db failed", "err", err); os.Exit(1)
    }
    go watchFiles("/etc/secrets/db", func() { _ = reloadDB() })
    
    // 3. AWS SDK 自动用 IRSA（无需手动配置）
    awsCfg, err := config.LoadDefaultConfig(ctx)
    if err != nil { slog.Error("aws config", "err", err); os.Exit(1) }
    _ = awsCfg
    
    // 4. 启动 HTTP server（用 TLS Secret）
    startServer("/etc/tls/tls.crt", "/etc/tls/tls.key")
}

func loadConfig(path string) error {
    data, err := os.ReadFile(path)
    if err != nil { return err }
    var c AppConfig
    if err := yaml.Unmarshal(data, &c); err != nil { return err }
    cfg.Store(&c)
    slog.Info("config loaded", "log_level", c.Log.Level)
    return nil
}

func reloadConfig() {
    if err := loadConfig("/etc/app/config.yaml"); err != nil {
        slog.Error("reload config", "err", err)
    }
}

func reloadDB() error {
    user := os.Getenv("USERNAME")     // ESO 写到 envFrom 里
    pass := os.Getenv("PASSWORD")
    dsn := "postgres://" + user + ":" + pass + "@db:5432/prod?sslmode=require"
    newDB, err := sql.Open("postgres", dsn)
    if err != nil { return err }
    if err := newDB.PingContext(context.Background()); err != nil {
        newDB.Close(); return err
    }
    old := dbPool.Swap(newDB)
    if old != nil {
        go func() { time.Sleep(5 * time.Second); old.Close() }()
    }
    slog.Info("db pool rotated")
    return nil
}

func watchFiles(dir string, onChange func()) {
    w, _ := fsnotify.NewWatcher()
    defer w.Close()
    w.Add(dir)
    for ev := range w.Events {
        if ev.Op&(fsnotify.Create|fsnotify.Remove|fsnotify.Write) != 0 {
            slog.Info("file changed", "path", filepath.Base(ev.Name))
            onChange()
        }
    }
}

func startServer(certFile, keyFile string) {
    // 略——业务 handler 用 cfg.Load() 与 dbPool.Load()
}
```

### 13.4 一次配置变更的完整链路

```
1. SRE 改 Vault DB role TTL（30m → 15m）
2. 30 分钟后 ESO refreshInterval 到 → 拉新 username/password
3. ESO 更新 Secret/db-cred
4. Pod volume mount 在 ~60s 内同步（kubelet）
5. 应用 fsnotify 监听到变化 → reloadDB()
6. 新 sql.DB pool 用新 user/pass，旧 pool 5 秒后 close
7. 业务零中断
```

整个流程 **没有人**碰过明文密钥。Vault audit log 记录"谁在何时申请了哪个 lease"。一切可追溯、可轮换、可撤销。

---

## 第十四章：练习题

**练习 1**：解释下面这个 Pod 启动后，为什么 `kubectl exec pod -- env | grep DB_PASSWORD` 能看到密码，但 `kubectl describe pod` 看不到？这有什么安全含义？

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-cred
      key: password
```

**练习 2**：以下配置——为什么应用层"配置文件改了热加载"工作不了？

```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/app/config.yaml
  subPath: config.yaml
volumes:
- name: cfg
  configMap:
    name: api-config
```

**练习 3**：写一个 Go 函数 `reloadOnSignal(ctx, path)`：监听 `path` 所在目录的文件变化，每次变化都调用 `applyConfig(path)`。要求：

- 用 fsnotify
- 失败重试（如果 watcher 出错重建）
- ctx cancel 时优雅退出
- 50ms debounce（避免 kubelet 多次原子操作触发多次重载）

**练习 4**：你的集群 30 个 Pod 都引用同一个 ConfigMap。kubectl edit configmap 后，所有 Pod 的 `volumeMount` 内容大约多久能看到变化？哪些因素影响这个延迟？如果你想"立即生效"，有哪些办法？

**练习 5**：设计一个无停机的"数据库密码轮换"流程：

- 当前密码 `old` 用了 90 天，要换成 `new`
- 数据库支持多个有效密码
- 集群有 100 个应用副本
- 不能有任何请求失败

给出步骤与对应的 K8s/Vault 操作。

**练习 6**：以下 RBAC 给了应用 SA 什么权限？有什么风险？怎么收敛？

```yaml
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

**练习 7**：你要把 Sealed Secrets 从集群 A 迁移到集群 B（A 即将下线）。如何让 B 能解开所有 A 的 SealedSecret？过程中有哪些风险？

**练习 8**：解释为什么 ESO 的 `refreshInterval: 10s` 在生产是个坏主意。给出一个合理的 refreshInterval 选择框架（针对：静态密钥 / 1h lease Vault dynamic / 5min lease）。

**练习 9**：IRSA 在底层是怎么工作的？画出一次"Pod 拿到 AWS STS 临时凭证"的完整流程（含 K8s SA token、AWS STS、IAM）。为什么这样做比"把 AWS access key 写进 Secret"更安全？

**练习 10**：你的 Helm chart 里有 ConfigMap。`helm upgrade` 后 ConfigMap 内容确实变了，但 Deployment 没有 rolling restart。给出三种解决方案，比较优劣。

---

## 参考答案

**练习 1**：`describe pod` 显示的是 Pod spec 的元数据——`secretKeyRef` 只表明"引用了哪个 Secret 的哪个 key"，不会显示值。但 `exec ... env` 是进入容器内执行，env vars 已经被 kubelet 注入 → 当然能看到。

**安全含义**：

- 任何拥有 `pods/exec` 权限的人 = 拥有 Pod 里所有 env Secret 明文权限
- `pods/log` 也类似（如果应用打印了 env）
- 因此**生产 `exec` 权限要严格控制**——只给 break-glass SRE，不给开发
- 更好的做法：用 volume mount 而非 env，至少 mode 0400 + 文件系统级权限

**练习 2**：subPath 是把 ConfigMap volume 里的"一个文件"硬链接到挂载路径——硬链接不会随 kubelet 后续对 `..data` symlink 的切换而更新。要热更新必须挂目录（而非单文件）。

修复：

```yaml
volumeMounts:
- name: cfg
  mountPath: /etc/app          # 挂目录
volumes:
- name: cfg
  configMap:
    name: api-config
```

或者改用 `subPathExpr` 不能解决这个问题（subPathExpr 只是动态计算 subPath 的值，挂载本质相同）。

**练习 3**：

```go
func reloadOnSignal(ctx context.Context, path string, applyConfig func(string) error) error {
    dir := filepath.Dir(path)
    for {
        if err := watchOnce(ctx, dir, path, applyConfig); err != nil {
            select {
            case <-ctx.Done(): return ctx.Err()
            case <-time.After(time.Second):  // 重建 watcher 退避
            }
            continue
        }
        return nil
    }
}

func watchOnce(ctx context.Context, dir, path string, apply func(string) error) error {
    w, err := fsnotify.NewWatcher()
    if err != nil { return err }
    defer w.Close()
    if err := w.Add(dir); err != nil { return err }

    var (
        debounce *time.Timer
        debounceCh = make(chan struct{}, 1)
    )
    defer func() { if debounce != nil { debounce.Stop() } }()

    for {
        select {
        case <-ctx.Done():
            return nil
        case ev, ok := <-w.Events:
            if !ok { return errors.New("watcher closed") }
            if ev.Op&(fsnotify.Create|fsnotify.Remove|fsnotify.Write) == 0 { continue }
            // debounce
            if debounce != nil { debounce.Stop() }
            debounce = time.AfterFunc(50*time.Millisecond, func() {
                select { case debounceCh <- struct{}{}: default: }
            })
        case <-debounceCh:
            if err := apply(path); err != nil {
                slog.Error("apply config failed", "err", err)
            }
        case err, ok := <-w.Errors:
            if !ok { return errors.New("watcher closed") }
            return err
        }
    }
}
```

**练习 4**：约 1 分钟内（默认 kubelet `--sync-frequency=1m`、`configMapAndSecretChangeDetectionStrategy=Watch`）。影响因素：

- kubelet sync 周期
- API server 延迟
- 节点 inotify 资源（容器多时可能 throttle）
- ConfigMap 是否打 immutable（打了就不更新，需重建）

立即生效办法：

- `kubectl rollout restart deployment/api`（推荐）
- Reloader 自动触发
- Kustomize secretGenerator/configMapGenerator（带 hash → spec 变化 → 自动滚动）
- 应用启动时主动 GET Secret（不挂载，client-go 实时读）

**练习 5**：

```
0. 准备
   - 应用支持"同时尝试多个密码"（或多个 sql.DB pool）
   - Vault 或外部存储能管理多个 valid 密码

1. T0: ALTER USER app PASSWORD 'new';                # PostgreSQL 改密码（仍可用旧）
       SELECT pg_reload_conf();                       # 加载新配置
       此时数据库同时认 old 和 new（如 pg_hba 配置）
       
2. Vault: 写入 new password 到 secret/db/cred-v2
3. ESO 同步：Secret 现在有 password.current=new、password.previous=old
4. 应用 fsnotify 监听到变化 → 加载新 DSN，建立新 pool
5. 旧 pool 优雅 drain（5 分钟）
6. 验证：所有 Pod 都在用新 pool
7. T+10m: 在 DB 上 revoke old；删除 Vault 旧 secret
8. 审计 log 留存

应用代码：
- 维护两个 sql.DB pool（current/previous）
- 新 query 走 current；如果 current ping 失败临时 fallback 到 previous
- 配置变更触发 atomic swap
```

**练习 6**：风险——`list` 在 Kubernetes API 默认返回**完整对象**（含 data）。任何能 list 的人 = 能看到该 namespace 内所有 Secret 明文。`watch` 类似——长连接流式发送变更，每个变更也带完整 data。

收敛：

```yaml
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["api-config-secret"]    # 限定具体 Secret
  verbs: ["get"]                           # 仅 get
```

**练习 7**：

```
1. 在 A 集群导出 master key（私钥）：
   kubectl get secret -n kube-system \
     -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
     -o yaml > sealed-master.yaml

2. 在 B 集群（已部署 sealed-secrets-controller）apply：
   kubectl apply -f sealed-master.yaml

3. 重启 B 的 controller 让它加载新 key：
   kubectl rollout restart deploy/sealed-secrets-controller -n kube-system

4. 把 SealedSecret YAML 同步到 B（GitOps 或 kubectl apply）

5. 验证：检查 B 集群 SealedSecret 状态都是 Synced
```

风险：

- 私钥**通过 kubectl export 落盘**——离线传输路径必须加密（不能 Slack 贴）
- A 还在运行时有重复解密 = 两边都能 decode → 短期内 OK，但 audit 上不清晰
- key 转移完后**确认从 A 彻底清理**——防止 A 离线后被反向取回数据
- 历史版本 sealed-secrets 私钥格式可能不兼容

**练习 8**：refreshInterval 太短会：

- 高频压 Vault → Vault 限流 / 集群下线
- 高频 K8s API write（Secret 更新）→ apiserver 与 etcd 压力
- 应用频繁热加载 → 业务抖动

合理选择：

| 密钥类型 | refreshInterval | 原因 |
|---|---|---|
| 静态密钥（API key 一年不变） | 24h | 实际不变，refresh 是兜底 |
| Vault 1h lease | 30m | lease 的 1/2，留时间轮换 |
| Vault 5min lease | 2m | 同上，1/2 + 安全余量 |
| 证书（90 天） | 6h | 远早于过期即可 |

通用规则：**比 lease/expiry 的 1/2 短**，给应用足够时间感知 + 重载 + 失败重试。

**练习 9**：

```
1. K8s 给 Pod projected SA token（OIDC JWT），claim 包含 SA 名 + namespace
2. Pod 启动，AWS SDK 读 AWS_WEB_IDENTITY_TOKEN_FILE
3. SDK 调用 STS:AssumeRoleWithWebIdentity，传 OIDC token + IAM role ARN
4. STS 用 K8s OIDC provider 的公钥验证 token（K8s OIDC 已注册到 IAM）
5. 验证通过 → IAM 检查 role 的 trust policy（允许该 SA assume）
6. 返回临时凭证 (access_key + secret + session_token)，15min ~ 12h
7. SDK 缓存凭证，过期前自动 refresh
```

更安全因为：

- 凭证短期（1h 默认），泄漏窗口窄
- 凭证不存 K8s Secret → 不进 etcd → 没有"静态密钥被 dump"问题
- 身份与 Pod 强绑定（SA + namespace），不能跨 SA 借用
- 撤销容易（删 SA 或改 role trust）
- 审计可追溯到具体 SA（CloudTrail）

**练习 10**：

方案 A — Pod template 加 checksum annotation：

```yaml
template:
  metadata:
    annotations:
      checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

优点：纯 Helm，零依赖；缺点：所有 chart 都得改。

方案 B — Reloader：

```yaml
annotations:
  configmap.reloader.stakater.com/reload: "api-config"
```

优点：模板不动；缺点：需要部署 Reloader controller。

方案 C — 改用 Kustomize（或 Helm 模板里用 hash 后缀）：

```yaml
configMapGenerator:
- name: api-config
  files: [config.yaml]
```

生成的 ConfigMap 是 `api-config-<hash>`，Deployment 引用变了 → 自动滚动。优点：原生 K8s pattern；缺点：要切换工具或在 Helm 模板里手写 hash。

**生产推荐**：Reloader（解耦） + GitOps 流程（C14）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| ConfigMap | 1 MiB 上限；env 不热更、volume 热更（subPath 不热更） |
| Secret 类型 | Opaque / tls / dockerconfigjson / SA token；base64 ≠ 加密 |
| etcd 加密 | EncryptionConfiguration + KMS v2；apply 后必须 rewrite 老 Secret |
| Sealed Secrets | 集群私钥加密，密文可 commit；私钥必须备份 |
| External Secrets | Vault / 云 SM 是源，refreshInterval 按 lease 一半设 |
| Vault 动态密钥 | 短期 user/pass，按身份生成；配合 sql.DB pool 双 pool 切换 |
| 配置变更 | 应用层热重载 + Reloader/checksum + GitOps |
| 多环境 | Kustomize overlay 或 Helm values；Secret 与 chart 分离 |
| 最小权限 | 不要 list secrets；resourceNames 显式枚举 |
| 工作负载身份 | IRSA / Workload Identity / SPIFFE——零静态密钥 |

铁律：

- **永远**不要把明文密钥 commit 进 Git
- **永远**给 Secret volume mode 0400
- **永远**给 RBAC 写 resourceNames，不给 list/watch
- subPath 挂载 = 不热更，配合 Reloader
- 动态密钥的 lease ≥ 应用 refresh × 2
- etcd encryption 启用后必须重写所有 Secret
- 生产用 ESO 或 CSI Secrets Store，少用 long-lived K8s Secret
- 优先 IRSA / WI / SPIFFE 实现 secretless

下一篇 **C06 — 精通 Scheduling 与资源管理** 将拆开 Requests/Limits、QoS、HPA/VPA/KEDA、节点亲和与污点——把"配置好"的应用真正"调度好"。

---
