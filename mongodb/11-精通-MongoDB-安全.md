# 精通 MongoDB 安全：SCRAM、x.509、TLS、FLE 与 Queryable Encryption

> 关联章节：[M05 副本集](./05-精通-副本集.md)、[M06 分片集群](./06-精通-分片集群.md)、[M12 生产运维](./12-精通-生产运维-与-替代品.md)

---

## 引言：默认裸奔，必须加固

MongoDB 默认：

- 监听 `localhost:27017`，但生产改成 `0.0.0.0` 后**任何人都能连**
- 没有用户、没有 ACL
- 没有 TLS

历史上几起 MongoDB "数据被勒索"事件都是这个原因（默认裸奔 + 暴露公网）。

生产必须加上：

- **认证**：SCRAM / x.509 / LDAP / Kerberos / AWS IAM
- **授权**：RBAC + 内置 / 自定义角色
- **传输加密**：TLS
- **静态加密**：磁盘加密（OS / WT 加密引擎）
- **字段级加密**：FLE / Queryable Encryption（QE）
- **审计**：操作日志（企业版）

读完本章你应能：

- 配 SCRAM 用户 / RBAC 角色
- 启 mTLS（双向证书认证）
- 用 FLE 加密敏感字段
- 用 Queryable Encryption "加密下仍能查询"
- 设计零信任的 MongoDB 部署

---

## 第一章：威胁模型

| 威胁 | 例子 | 对策 |
|---|---|---|
| 未授权访问 | 任何人连上 → 读所有数据 | 认证 + 防火墙 |
| 越权 | 用户读不该读的 db | RBAC |
| 传输窃听 | 内部 / 公网传输被嗅 | TLS |
| 磁盘失窃 | 盘被偷、备份泄露 | 静态加密 |
| 内鬼 | DBA 直接看敏感字段 | 字段级加密 |
| SQL/NoSQL 注入 | 应用拼 query | 参数化 / driver 强类型 |
| DoS | 一个客户端打爆集群 | quota / connection limit |

不同业务威胁权重不同。先列出来再设计。

---

## 第二章：认证机制

### 2.1 SCRAM（默认）

**SCRAM-SHA-256**（推荐）/ SCRAM-SHA-1（旧）：

- 用户名 + 密码
- 服务端存 hash（salted），客户端不发明文
- 协议层防中间人

### 2.2 开启认证

```yaml
# mongod.conf
security:
  authorization: enabled
```

启动后必须先建管理员：

```js
use admin
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase"]
})
```

然后用 admin 用户继续创建业务用户。

### 2.3 创建业务用户

```js
use mydb
db.createUser({
  user: "appuser",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "mydb" },
    { role: "read", db: "logs" }
  ]
})
```

连接：

```
mongodb://appuser:password@host/mydb?authSource=mydb
```

### 2.4 x.509 证书认证

证书代替密码：

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/server.pem
    CAFile: /etc/mongo/ca.pem
security:
  clusterAuthMode: x509
```

```js
db.getSiblingDB("$external").createUser({
  user: "CN=app1,OU=eng,O=mycorp,L=NYC,ST=NY,C=US",
  roles: [{ role: "readWrite", db: "mydb" }]
})
```

用户名就是证书 subject。

适合：服务对服务的 zero-trust 场景。

### 2.5 LDAP（企业版）

```yaml
security:
  ldap:
    servers: "ldap.example.com"
    transportSecurity: tls
    bind:
      method: simple
      saslMechanisms: PLAIN
```

把 LDAP 用户身份直接用于 MongoDB。常见于企业 AD 集成。

### 2.6 Kerberos（企业版）

```yaml
setParameter:
  authenticationMechanisms: "GSSAPI"
```

企业 AD 单点登录场景。

### 2.7 AWS IAM（Atlas）

```js
mongodb+srv://user@cluster.example.mongodb.net/?authMechanism=MONGODB-AWS
```

不存密码，用 AWS IAM Role 认证。Atlas 上常用。

---

## 第三章：RBAC（基于角色的访问控制）

### 3.1 内置角色

**Database User Roles**（针对单 db）：
- `read`
- `readWrite`
- `dbAdmin`
- `dbOwner`
- `userAdmin`

**Database Administration Roles**（admin db）：
- `userAdminAnyDatabase`
- `readWriteAnyDatabase`
- `readAnyDatabase`
- `dbAdminAnyDatabase`

**Cluster Roles**：
- `clusterAdmin`：集群管理
- `clusterManager`：监控管理
- `clusterMonitor`：只读监控
- `hostManager`：节点管理

**Backup/Restore Roles**：
- `backup`
- `restore`

**Superuser**：
- `root`：所有权限（慎用）

### 3.2 自定义角色

```js
db.createRole({
  role: "analyst",
  privileges: [
    {
      resource: { db: "mydb", collection: "" },
      actions: ["find", "listCollections"]
    },
    {
      resource: { db: "mydb", collection: "audit" },
      actions: []   // 排除 audit
    }
  ],
  roles: []
})
```

最小授权原则：业务侧只用 readWrite，运维用更小子集。

### 3.3 修改用户

```js
db.updateUser("appuser", {
  roles: [{ role: "read", db: "mydb" }]   // 改成只读
})
db.changeUserPassword("appuser", passwordPrompt())
```

### 3.4 列权限

```js
db.getUsers()
db.getUser("appuser")
db.getRolesForUser("appuser")
db.runCommand({usersInfo: "appuser", showPrivileges: true})
```

---

## 第四章：TLS

### 4.1 四种模式

| 模式 | 行为 |
|---|---|
| `disabled` | 不用 TLS（默认） |
| `allowTLS` | 接受 TLS 和非 TLS |
| `preferTLS` | 内部连接用 TLS（外部 TLS 但可选） |
| `requireTLS` | 必须 TLS |

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/server.pem
    CAFile: /etc/mongo/ca.pem
    allowConnectionsWithoutCertificates: false   # mTLS 时不允许无证书
```

### 4.2 mTLS（双向）

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/mongo/server.pem
    CAFile: /etc/mongo/ca.pem
    clusterFile: /etc/mongo/cluster.pem   # 节点间互信证书
```

客户端必须也出示证书。证书 CN 可直接做用户身份（x.509 认证）。

### 4.3 内部 vs 外部 listener

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0
  tls:
    mode: requireTLS
    # 外部：要求强 TLS
```

可以让内部副本集间通信不强 TLS（节省 CPU），但生产推荐**全 TLS**。

### 4.4 证书生成

```bash
# 1. 生成 CA
openssl req -new -x509 -keyout ca-key.pem -out ca.pem -days 3650 -subj "/CN=MongoCA"

# 2. mongod 证书
openssl req -newkey rsa:2048 -nodes -keyout server-key.pem -out server.csr -subj "/CN=mongo1.example.com"
openssl x509 -req -CA ca.pem -CAkey ca-key.pem -in server.csr -out server-cert.pem -days 365 -CAcreateserial
cat server-key.pem server-cert.pem > server.pem

# 3. 客户端证书
openssl req -newkey rsa:2048 -nodes -keyout client-key.pem -out client.csr -subj "/CN=appuser1"
openssl x509 -req -CA ca.pem -CAkey ca-key.pem -in client.csr -out client-cert.pem -days 365
cat client-key.pem client-cert.pem > client.pem
```

生产用 cert-manager / Vault PKI 自动化。

### 4.5 证书过期

证书过期是定时炸弹：

- mongod 证书过期 → 客户端连接失败
- mTLS 下 节点间证书过期 → 副本集挂

预防：

- cert-manager 自动 renew
- 监控有效期 < 60 天告警
- 每年至少一次"模拟过期"演练

---

## 第五章：静态加密

### 5.1 OS 级（推荐起步）

- LUKS（Linux 全盘加密）
- AWS EBS encryption
- GCP / Azure 类似

特点：

- mongod 不感知
- 整盘加密
- 密钥管理外置（KMS / Vault）

### 5.2 WT 加密引擎（企业版）

```yaml
security:
  enableEncryption: true
  encryptionKeyFile: /etc/mongo/encryption-key
```

- WiredTiger 写文件时加密
- 内置 AES-256-CBC / GCM
- 密钥可以挂到 KMS（AWS KMS / Vault / KMIP）

适合：合规要求（HIPAA / PCI DSS / GDPR）"DB 层加密" 标杆。

---

## 第六章：FLE —— Field Level Encryption

### 6.1 用途

**应用层加密**：客户端加密后再发给 MongoDB。

```
Application → 加密 SSN → MongoDB（看不到原文）
```

适合：

- 内部审计场景（DBA 不能看到原始数据）
- 多方信任分离

### 6.2 Client-Side FLE

```js
const fle = new MongoClient(uri, {
  autoEncryption: {
    keyVaultNamespace: "encryption.__keyVault",
    kmsProviders: {
      aws: { accessKeyId: "...", secretAccessKey: "..." }
    },
    schemaMap: {
      "mydb.users": {
        bsonType: "object",
        properties: {
          ssn: {
            encrypt: {
              keyId: [/* dataKeyId */],
              bsonType: "string",
              algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512-Deterministic"
            }
          }
        }
      }
    }
  }
})
```

效果：

- driver 自动加密 `ssn` 字段
- 存到 DB 是密文
- 查询时 driver 加密查询条件（deterministic 才能 match）

### 6.3 两种算法

- **Deterministic**：相同明文 → 相同密文。**可等值查询**。
- **Random**：每次加密产生不同密文。**不能查询**，只能读出来解。

### 6.4 限制

- 范围查询不行（Deterministic 也只 match 完全相等）
- 索引只对 deterministic 字段有意义（同密文才能找到）
- 性能：客户端 CPU 占用 +5-10%

---

## 第七章：Queryable Encryption（6.0 Preview / 7.0 GA；等值查询 7.0 GA，范围查询自 8.0 起 GA）

### 7.1 解决什么问题

FLE Deterministic 能等值查，但 server 看到的是确定性密文 → frequency analysis 仍可能泄露信息（如知道 SSN 有 10 万种，统计密文出现频率推测）。

**Queryable Encryption（QE）** 解决：

- 同明文产生**不同密文**
- 但**仍支持等值 / 范围查询**
- server 完全看不到 frequency 模式

### 7.2 用法

```js
const clientEncryption = new ClientEncryption(client, {
  keyVaultNamespace: "encryption.__keyVault",
  kmsProviders: {...}
})

await db.createEncryptedCollection("patients", {
  encryptedFields: {
    fields: [
      { path: "ssn", bsonType: "string", queries: { queryType: "equality" } },
      { path: "salary", bsonType: "int", queries: { queryType: "range", min: 0, max: 1000000 } }
    ]
  },
  provider: "local"
})
```

写入 / 查询自动加解密。

> 注：等值（equality）查询 7.0 GA；范围（range，如上例 salary min/max）查询自 8.0 起 GA，示例需 8.0+ 服务端与驱动。

### 7.3 性能与代价

- 存储：每加密字段额外 metadata，约 1.5× 增长
- 查询：等值 query 比明文慢 2-5×
- 索引：内部用 secure index（特殊结构）

### 7.4 适用场景

- 医疗（HIPAA）：诊断码、SSN
- 金融：账户号
- 个人信息：身份证号
- 数据合作场景（外包给云）

---

## 第八章：网络隔离

### 8.1 防火墙规则

```
mongod 端口 27017 / 27018 / 27019 只允许：
- 应用网段
- 运维跳板机
- 监控
```

外部一律 DENY。

### 8.2 VPC / VLAN

云上用专用 VPC：

- MongoDB 集群在私网
- 应用通过 VPC peering / private endpoint 访问
- 不暴露公网 IP

### 8.3 Network Encryption Tunnels

跨 region 用 VPN / WireGuard 加密内网。

### 8.4 bindIp 限制

```yaml
net:
  bindIp: 10.0.0.1,127.0.0.1   # 只 listen 内网 + 本地
```

避免误把 mongod listen 在 0.0.0.0 暴露公网。

---

## 第九章：审计

### 9.1 企业版审计

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log
  filter: '{ atype: { $in: ["authenticate", "createCollection", "dropDatabase"] } }'
```

记录关键操作。社区版没有内置。

### 9.2 替代：profile + Change Streams

社区版用 profiler + Change Streams 自建简易审计：

```js
db.setProfilingLevel(1, { slowms: 0 })   // 全记
// 后台脚本读 system.profile 写到外部 SIEM
```

代价：profiler 全开有性能影响。

### 9.3 必审计事件

- 认证失败
- DB / collection 创建 / 删除
- 用户 / 角色变更
- 敏感字段读取（用 FLE 日志或 Change Streams）
- 配置变更

---

## 第十章：典型故障

### 10.1 案例：上线后才发现没认证

**症状**：测试环境跑了一个月，没人想起开 auth。

**修复**：

```js
// 1. 加用户
db.createUser({user:"admin", pwd:"...", roles:["root"]})
// 2. 改配置：security.authorization: enabled
// 3. 重启
// 4. 业务侧改连接字符串带 credentials
```

预防：CI 检查 server 配置文件、startup script 加 auth 校验。

### 10.2 案例：证书过期，业务全挂

**症状**：周一早上业务全报 SSL handshake failed。

**根因**：证书周末过期，cert-manager 没自动 renew（配置错）。

**修复**：

- 紧急更新证书
- cert-manager 配置 review
- 加 expiry 监控 60 天 / 30 天 / 7 天三级告警

### 10.3 案例：被勒索

**症状**：MongoDB 集合被清空，留一条 "Pay BTC to ..." 消息。

**根因**：

- 没认证（默认）
- bindIp 0.0.0.0
- 暴露公网

**修复**：

- 立即下线公网入口
- 从备份恢复
- 上认证 + TLS + 私网

预防：永远不要"省事不开 auth"。

### 10.4 案例：开 FLE 后查询全 miss

**症状**：业务上线 FLE，所有 `find({ssn:"123-45-6789"})` 返回 0 条。

**根因**：

- 索引建在明文字段，但密文是不同字节
- 或者 FLE 配置用了 Random 算法（不可查）

**修复**：

- 改 Deterministic 算法
- 索引建在加密字段（自动用密文做 key）

### 10.5 案例：用户权限过大

**症状**：审计发现某 app 用户用了 `root` 角色。

**修复**：

```js
db.updateUser("appuser", { roles: [{role: "readWrite", db: "mydb"}] })
```

最小授权：app 用户只对自己 db 有 readWrite，不要给 admin / userAdmin。

---

## 第十一章：合规

### 11.1 GDPR

- 数据驻留：用 zone sharding 把欧洲用户数据放欧洲 shard
- 删除权（right to erasure）：删用户文档 + 关联数据
- 加密：传输 + 存储 + 字段级
- 审计：保留访问 log

### 11.2 HIPAA

- 字段级加密医疗信息
- 审计所有访问
- 访问最小化
- 数据中心物理隔离（专属 region）

### 11.3 PCI DSS

- 支付卡数据不存在 MongoDB（用专业 tokenization 服务）
- 如果必须存：FLE + 密钥分离
- 审计 + 漏洞扫描

---

## 第十二章：安全 Checklist

### 12.1 部署阶段

- [ ] 启用 authorization
- [ ] 创建 admin 用户 + 业务用户（最小授权）
- [ ] bindIp 限制到内网
- [ ] TLS 强制（requireTLS）
- [ ] 证书自动 renew（cert-manager / Vault）
- [ ] 防火墙规则（只开必要端口）
- [ ] 不暴露公网

### 12.2 运行阶段

- [ ] 定期 review 用户 / 角色
- [ ] 监控认证失败
- [ ] 监控证书过期
- [ ] 备份加密（备份本身可能比库还敏感）
- [ ] 审计敏感操作

### 12.3 应急

- [ ] 凭证泄露：立刻轮换 + 审计入侵
- [ ] 数据泄露：通知监管 + 用户
- [ ] 勒索：报警，从备份恢复
- [ ] DR 演练：定期跑

---

## 总结 · 安全一图

```mermaid
graph TD
    Client[Client] -->|TLS encrypt + cert| Auth[Authentication<br>SCRAM / x.509 / LDAP / IAM]
    Auth -->|principal=User:app1| Authz[Authorization<br>RBAC]
    Authz -->|有 readWrite mydb 权限?| Op[操作]

    Op --> DB[(MongoDB)]
    DB -.OS / WT 加密.-> Disk[(加密磁盘)]

    Client -.FLE / QE.-> DB
    Note[敏感字段已加密<br>DB 看不到明文]

    Audit[审计 log] -.记录.-> Op

    style Auth fill:#fff3e0
    style Authz fill:#c8e6c9
    style Note fill:#fce4ec
```

安全心法：

1. **永远开 auth** —— 没有例外
2. **TLS 强制** —— 内部 + 外部
3. **最小授权** —— app 用户不要 root
4. **不暴露公网** —— bindIp + 防火墙
5. **证书自动化** —— 定时炸弹
6. **FLE / QE 保护敏感字段** —— 多一层防线
7. **审计 + 监控** —— 没有可观测性就没有安全

---

## 练习题

1. SCRAM 与 x.509 认证适用场景区别？
2. mTLS 比单向 TLS 多了什么？
3. RBAC 的"最小授权"具体怎么做？
4. FLE Deterministic 与 Random 算法区别？
5. Queryable Encryption 解决了 FLE 的什么问题？
6. requireTLS 与 preferTLS 的区别？
7. 静态加密的 3 种方式（OS / WT / FLE）取舍？
8. 证书过期前没轮换会怎样？
9. zone sharding 怎么帮助 GDPR 合规？
10. 一个公司开始上线 MongoDB，列 5 个必做安全项。

> 答案见 [QUIZ.md](./QUIZ.md)

---

> 🔁 反馈：用 nmap 扫自家 MongoDB 端口，看暴露面有多大
