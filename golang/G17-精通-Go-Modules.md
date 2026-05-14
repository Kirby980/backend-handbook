# 精通 Go Modules

> 课程编号：G17
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Modules & Dependencies 章节
> 难度：⭐⭐⭐
> 预计阅读时间：45 分钟

---

## 引言：依赖管理的演进

Go 1.0 没有官方依赖管理；2013 年的 `GOPATH` 是单全局 workspace；2014 年 `vendor/` 目录；2017 年 `dep` 实验工具；2018 年 Go 1.11 引入 **Modules**——终结之前所有方案。

```bash
go mod init github.com/me/myapp
go get github.com/gin-gonic/gin@v1.9.1
go mod tidy
```

这是现代 Go 依赖管理的全部三条命令。但围绕 `go.mod` 仍有大量微妙细节：MVS 算法、`replace`、`exclude`、`retract`、私有仓库配置、proxy 校验。本章拆开看。

---

## 第一章：go.mod 文件结构

### 1.1 最小示例

```
module github.com/me/myapp

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/lib/pq v1.10.9
)
```

四个核心指令：

- `module <path>`：模块的唯一路径（通常是 git URL）
- `go <version>`：要求的最小 Go 版本（同时控制语言特性如循环变量语义）
- `require`：直接依赖
- `toolchain` (Go 1.21+)：指定使用的 Go 工具链
- `tool` **(Go 1.24+)**：直接登记**可执行工具依赖**，告别 `tools.go` 黑魔法

### 1.2 完整指令列表

```
module M
go V
toolchain T
require P V
replace A => B
exclude P V
retract V
```

### 1.3 go.sum 文件

```
github.com/gin-gonic/gin v1.9.1 h1:hash...
github.com/gin-gonic/gin v1.9.1/go.mod h1:hash...
```

每个依赖两行 hash：模块本身 + 它的 go.mod。下次构建验证完整性。**必须** commit。

---

## 第二章：版本语义

### 2.1 SemVer

`vMAJOR.MINOR.PATCH` 例如 `v1.2.3`。

- MAJOR：破坏性变更
- MINOR：向后兼容新增
- PATCH：bug 修复

### 2.2 Module Path 与 Major

**v2+ 必须把版本号写进 module path**：

```
module github.com/me/lib       // v0/v1
module github.com/me/lib/v2    // v2
module github.com/me/lib/v3    // v3
```

调用方：

```go
import "github.com/me/lib/v2"
```

这条规则叫 "Semantic Import Versioning"——避免一个程序同时依赖 v1 和 v2 引起符号冲突。

### 2.3 pseudo-version

当依赖还没正式 tag，go.mod 中会出现：

```
require github.com/x/y v0.0.0-20231015120304-abcdef123456
```

格式：`vX.Y.Z-时间戳-提交哈希前12位`。`go get -u` 升到最新 main 自动生成。

### 2.4 +incompatible

如果一个仓库有 v2.0.0 但 module path 没加 `/v2`，go 把它标记 `v2.0.0+incompatible`，表示"违反 SemVer 路径约定但仍可用"。这是兼容旧仓库的过渡机制。

---

## 第三章：MVS（Minimal Version Selection）

### 3.1 算法

Go modules **不**走 npm/yarn 的"最新满足约束"——它用 **MVS**：每个依赖只取所有 require 路径中**最低能用的**版本。

### 3.2 例子

```
myapp
├── A v1.0.0   (requires C v1.1.0)
└── B v2.0.0   (requires C v1.3.0)
```

MVS 选 C v1.3.0（直接依赖路径中最高的最低要求）。

### 3.3 优点

- 确定性：同一个 go.mod 永远算出同一个 lock
- 可重现：不需要 lock 文件（go.sum 只是 hash）
- 升级显式：`go get` 才会改版本

### 3.4 缺点

- 上游发新版不会自动用
- 旧依赖锁住整个图

---

## 第四章：go mod 子命令

### 4.1 常用

```bash
go mod init <module>             # 创建 go.mod
go mod tidy                      # 添加缺失、删除未用、更新 go.sum
go mod download                  # 下载到 module cache
go mod verify                    # 验证缓存与 sum 一致
go mod why <pkg>                 # 解释为何需要这个依赖
go mod graph                     # 依赖图
go mod edit -require=...         # 脚本化编辑
```

### 4.2 `go mod tidy` 最常用

每次 PR 前跑一次：
- 清理未用 require
- 添加 import 了但 require 缺的
- 补全 go.sum

CI 经常加一行 `go mod tidy && git diff --exit-code` 强制保证 PR 中 go.mod 已 tidy。

### 4.3 `go mod why`

```
$ go mod why golang.org/x/text
# golang.org/x/text/transform
github.com/me/app/internal/util
github.com/PuerkitoBio/goquery
golang.org/x/text/transform
```

显示从主模块到该依赖的路径。排查"为什么我没用却被引入"。

---

## 第五章：go get 与升级

### 5.1 添加新依赖

```bash
go get github.com/gin-gonic/gin              # 最新 release
go get github.com/gin-gonic/gin@v1.9.1       # 指定版本
go get github.com/gin-gonic/gin@latest       # 显式最新
go get github.com/gin-gonic/gin@main         # 跟踪分支
```

### 5.2 升级

```bash
go get -u                                    # 直接依赖升级到最新次版本
go get -u ./...                              # 加上间接依赖
go get -u=patch                              # 只升补丁版本
go get github.com/x/y@v1.5.0                 # 指定版本
```

### 5.3 降级

```bash
go get github.com/x/y@v1.2.0   # 降到 1.2.0
```

降级后跑 `go mod tidy` 处理 transitively。

### 5.4 移除

不再 import 就不用手动移除——下次 `go mod tidy` 自动清理。

---

## 第六章：replace / exclude / retract

### 6.1 replace：本地或镜像

```
replace github.com/x/y => ../y                  # 本地路径
replace github.com/x/y => github.com/me/y v1.2  # 替换源
replace github.com/x/y v1.2.0 => v1.3.0         # 版本替换
```

用途：
- 本地开发依赖未发布的版本
- 临时 fork 修 bug 等 upstream merge
- 镜像私有 fork

**只对当前模块生效**——下游 import 你的模块不会继承 replace。

### 6.2 exclude：禁用某版本

```
exclude github.com/x/y v1.5.2   // 这版本有严重 bug
```

MVS 算到这版本会跳过，选下一个最低能用的。少用。

### 6.3 retract：撤回自己发布的版本

```
retract v1.2.3   // 自己发布了但出问题
retract [v1.1.0, v1.1.9]
```

放在自己模块的 go.mod 中。`go get` 用户看到警告，提示升级到非 retracted 版本。是 Go 1.16+ 的官方"召回"机制。

---

## 第七章：vendor 模式

### 7.1 创建

```bash
go mod vendor
```

把所有依赖复制到项目内 `vendor/` 目录。

### 7.2 何时用

- 编译环境没法访问公网（如 air-gap）
- 强制 deterministic builds 即使 proxy 挂了
- 审查/安全：每行依赖代码都进 git

### 7.3 副作用

- 仓库变大
- 升级流程：`go get -u && go mod vendor`
- 默认 build 会优先用 vendor（如果存在）

`go build -mod=mod` 可以强制走 module cache 忽略 vendor。

---

## 第八章：Module Cache 与 Proxy

### 8.1 本地 cache

```
$GOPATH/pkg/mod/cache/download/...
```

所有下载过的模块。可用 `go clean -modcache` 清空。

### 8.2 公共 proxy

`GOPROXY` 默认 `https://proxy.golang.org,direct`。proxy 在 Go 团队维护，加速、缓存、可审计。

```bash
GOPROXY=https://goproxy.cn,direct      # 国内常用
GOPROXY=off                            # 完全离线
GOPROXY=direct                         # 跳过 proxy 直连源
```

### 8.3 校验和数据库

`GOSUMDB=sum.golang.org`：Google 维护的全球公开校验数据库。第一次拉某模块的某版本时，对比 hash。防止 supply-chain 攻击。

### 8.4 私有仓库

```bash
GOPRIVATE=*.corp.example.com,github.com/me/private
```

匹配 GOPRIVATE 的路径：
- 不经过 GOPROXY
- 不查询 GOSUMDB
- 直接从 source（git）拉

或单独配置：

```bash
GONOSUMCHECK / GONOSUMDB / GOPROXY=...,direct
```

### 8.5 私有 git 认证

```bash
git config --global \
  url."git@github.com:".insteadOf "https://github.com/"
```

让 go get 走 SSH。或用 .netrc 配置 token。

---

## 第九章：Workspace 模式（Go 1.18+）

### 9.1 问题

多个 module 同时开发：
```
~/repos/
├── moduleA/
├── moduleB/   (依赖 moduleA)
```

以前要 `replace github.com/me/A => ../moduleA`，每个开发都改一遍 go.mod 容易冲突。

### 9.2 workspace 解决

```bash
cd ~/repos
go work init ./moduleA ./moduleB
```

创建 `go.work`：

```
go 1.22

use (
    ./moduleA
    ./moduleB
)
```

在 workspace 内：moduleB import moduleA 会从本地路径解析。**go.work 不入 git**——它是开发者本地的"链接器"。

### 9.3 命令

```bash
go work init        # 创建
go work use ./mod   # 添加 module
go work sync        # 把 module 间约束同步
```

---

## 第九章半：tool 指令 —— 终结 `tools.go` 时代（Go 1.24+）

### 9.5 旧痛点

构建管线常依赖一些代码生成工具（`stringer`、`mockgen`、`protoc-gen-go`、`sqlc` ……）。在 Go 1.23 及之前，把它们"钉"到模块里要靠这种约定俗成的 hack：

```go
//go:build tools
// +build tools

package tools

import (
    _ "golang.org/x/tools/cmd/stringer"
    _ "github.com/golang/mock/mockgen"
)
```

加上 `//go:build tools` 标签让正常构建忽略它，但 `go mod tidy` 不会把它当未使用的导入删掉——一个纯粹的版本钉子文件。

### 9.6 Go 1.24：`tool` 指令成为正经语法

```
module example.com/myapp

go 1.24

tool (
    golang.org/x/tools/cmd/stringer
    github.com/sqlc-dev/sqlc/cmd/sqlc
)

require (
    golang.org/x/tools v0.30.0
    github.com/sqlc-dev/sqlc/cmd/sqlc v1.27.0
)
```

配套命令：

```bash
go get -tool golang.org/x/tools/cmd/stringer@latest   # 加一个工具
go tool stringer -type=Color color.go                  # 直接调用
go tool                                                # 列出所有 tool
```

`go tool <name>` 会**确保用 go.mod 钉的版本**，整个团队 / CI 跑同一个二进制。

### 9.7 迁移建议

- 删掉 `tools.go`，把里面 `_ "..."` 转成 `tool (...)` 块。
- CI 脚本里的 `go install foo@latest` 换成 `go tool foo`，避免每次拉最新版导致非确定性。
- 仍可用旧 `tools.go` 模式——`tool` 指令是**叠加**，不强制迁移。

> 参考：[Go 1.24 release notes — `tool` directive](https://go.dev/doc/go1.24#go-command)。

---

## 第十章：生产级最佳实践

1. **每次 commit 跑 `go mod tidy`**：CI 强制干净。
2. **`go.sum` 必须 commit**：保证可重现。
3. **私有仓库设置 GOPRIVATE**：避免 sumdb 502。
4. **依赖升级有节奏**：每月或每季度统一升一次。
5. **`go get -u=patch` 优先**：风险最小。
6. **`replace` 仅用于本地开发或紧急 hotfix**：合并后立刻去掉。
7. **`vendor/` 仅当需要**：增大仓库；除非有明确理由别加。
8. **v2+ 加 `/v2` 后缀**：违反约定客户端拉到错版本。
9. **release 用 git tag**：`git tag v1.2.3 && git push --tags`。
10. **重要模块开 `retract`**：发现严重问题后及时撤回。

---

## 第十一章：常见陷阱清单

### ❌ 陷阱 1：忘记 commit go.sum
其他开发者 hash 不匹配，构建失败。

### ❌ 陷阱 2：手动改 go.mod 版本
应该用 `go get`。手改容易让 go.sum 失同步。

### ❌ 陷阱 3：v2 包路径不加 `/v2`
```
require github.com/x/y v2.0.0+incompatible
```
说明库作者没遵守约定。最好不要发 v2+ 不带后缀。

### ❌ 陷阱 4：replace 留在 release
开发时本地 path replace，merge 后客户端拉不到 → 编译失败。

### ❌ 陷阱 5：直接依赖间接依赖
"我以为它一定可用"——上游可能改间接依赖。要直接依赖就 `go get` 显式声明。

### ❌ 陷阱 6：GOPROXY 设错导致 SumDB 失败
国内常见 `GOPROXY=https://goproxy.cn` 但忘了对私有仓库设 `GOPRIVATE`，公司内网仓库走 sumdb 失败。

### ❌ 陷阱 7：mod cache 占满磁盘
`$GOPATH/pkg/mod` 可能几 GB。定期 `go clean -modcache`。

---

## 第十二章：练习题

**练习 1**：以下 go.mod 是否合法？
```
module github.com/me/app
go 1.22
require github.com/x/y v2.5.0
```

**练习 2**：项目突然报错 `verifying module: checksum mismatch`，可能是什么原因？如何排查？

**练习 3**：你 fork 了 `github.com/orig/lib` 修复 bug，本地路径 `~/lib`。如何让当前项目用你的 fork？

**练习 4**：写一个 shell 脚本：自动升级所有间接依赖到最新补丁版本，并跑测试。

**练习 5**：解释 `go.work` 与 `replace` 的区别。

---

## 参考答案

**练习 1**：不合法。v2.5.0 必须 module path 带 `/v2`。要么 require `github.com/x/y/v2 v2.5.0`，要么 `github.com/x/y v2.5.0+incompatible`。

**练习 2**：
- 上游强制改了 tag（恶意或失误）
- proxy 缓存与 sumdb 不一致
- go.sum 被手动改坏

排查：`go mod verify`、`go clean -modcache && go mod download`、对比 `GOSUMDB` 显示的 hash。

**练习 3**：
```
replace github.com/orig/lib => /Users/me/lib
```
或者 `go work init ./lib ./app`。

**练习 4**：
```bash
#!/bin/bash
set -e
go get -u=patch ./...
go mod tidy
go test -race ./...
```

**练习 5**：
- `replace`：写入 `go.mod`，**进入项目** commit；改变发布的依赖图
- `go.work`：写入 `go.work`，**本地** 文件不 commit；仅影响当前开发机器
开发用 work，发布用 replace（且应在发布前移除）。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| go.mod | module/go/require/replace/exclude/retract |
| go.sum | hash 文件；必 commit |
| SemVer | v2+ 路径带 /v2 |
| MVS | 最低能用版本；确定性 |
| go mod tidy | PR 前必跑 |
| replace | 本地或紧急 hotfix；不传递 |
| GOPROXY | 默认 proxy.golang.org |
| GOPRIVATE | 私有仓库绕过 sumdb |
| Workspace | 多模块联合开发；不 commit |

下一篇 **G18 — 精通 Go 测试** 会讲清 table-driven、subtests、parallel、httptest、mocks 与 fakes 设计、t.Cleanup、testify 何时用。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G17-精通-Go-Modules.md`
