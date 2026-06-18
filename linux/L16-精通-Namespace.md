# 精通 Namespace：8 种命名空间、clone/setns/unshare、手搓容器、user ns 与 rootless

> 课程编号：L16
> 路线图来源：Linux · 模块六 容器与隔离
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：user namespace/rootless、time namespace 5.6+、runc/containerd）

---

## 引言：容器不是虚拟机，它就是被隔离的普通进程

很多人对容器的想象停留在「轻量虚拟机」。但 `docker run` 起来的进程，在宿主机 `ps aux` 里清清楚楚是一个普通进程——只不过它「看到的世界」被裁剪了：它以为自己是 PID 1，以为自己独占一套文件系统、网络栈、主机名。实现这种「障眼法」的，就是 **namespace（命名空间）**。配合 [L17 Cgroup](./L17-精通-Cgroup-v2.md) 的资源限额，namespace 提供「隔离视图」，二者合起来就是容器的全部内核基础。

本篇讲清 8 种 namespace 各隔离什么、用 `clone/unshare/setns` 怎么操作，并**用约 90 行 C 手搓一个能跑 shell 的迷你容器**，最后讲 user namespace 如何让非 root 用户安全地跑容器（rootless）。容器进程本质上还是 task，其 `task_struct` 与生命周期见 [L02 进程与线程](./L02-精通-进程与线程模型.md)；与 Docker/OCI 的对应见 [cloud-native C01](../cloud-native/C01-精通-Docker-与-OCI.md)。

---

## 第一章 8 种 namespace 总览

| namespace | clone flag | 隔离的资源 | 引入版本 |
|---|---|---|---|
| **mnt** | `CLONE_NEWNS` | 挂载点 / 文件系统视图 | 2.4.19（最早） |
| **uts** | `CLONE_NEWUTS` | hostname / domainname | 2.6.19 |
| **ipc** | `CLONE_NEWIPC` | System V IPC、POSIX 消息队列 | 2.6.19 |
| **pid** | `CLONE_NEWPID` | 进程号空间（独立 PID 1） | 2.6.24 |
| **net** | `CLONE_NEWNET` | 网络栈：网卡 / 路由 / 端口 / iptables | 2.6.29 |
| **user** | `CLONE_NEWUSER` | UID/GID 映射、capabilities | 3.8 |
| **cgroup** | `CLONE_NEWCGROUP` | cgroup 根视图 | 4.6 |
| **time** | `CLONE_NEWTIME` | CLOCK_MONOTONIC / BOOTTIME 偏移 | 5.6 |

每个进程的 namespace 归属体现在 `/proc/[pid]/ns/`：

```bash
ls -l /proc/self/ns/
# lrwxrwxrwx ... mnt -> 'mnt:[4026531840]'
# lrwxrwxrwx ... net -> 'net:[4026531840]'
# ... 方括号里是 inode 号，相同 inode = 同一个 namespace
lsns                    # 列出系统中所有 namespace 及其成员进程
```

两个进程若 `net` 的 inode 相同，就共享同一网络栈。容器就是把若干进程放进一组**新建的** namespace。

### 1.2 namespace 的本质与生命周期

每个 namespace 在内核里是一个带引用计数的对象，`/proc/[pid]/ns/<type>` 是指向它的「句柄」。一个 namespace 被**销毁**当且仅当引用归零：

- 其中所有进程都退出，**且**
- 没有任何 `/proc/[pid]/ns/xxx` 的打开 fd 持有它，**且**
- 没有被 bind mount 钉住。

这带来一个实用技巧——**保活一个空 namespace**。`ip netns add` 正是把 net namespace bind mount 到 `/var/run/netns/<name>`，即使没有进程也不销毁，方便后续反复 `ip netns exec` 进入：

```bash
readlink /proc/$$/ns/net          # net:[4026531840] —— 当前 net ns 的 inode 标识
exec 9< /proc/1234/ns/net         # 用 fd 持有，即便进程 1234 退出，该 ns 仍存活
lsns -t net                       # 列出所有 net namespace 及成员进程
```

---

## 第二章 操作 namespace：clone / unshare / setns

三个系统调用 + 两个命令行工具构成全部操作面：

| 接口 | 作用 |
|---|---|
| `clone(flags)` | 创建新进程**的同时**把它放进新建的 namespace（`CLONE_NEW*`） |
| `unshare(flags)` | **当前进程**脱离原 namespace，进入新建的（不创建新进程） |
| `setns(fd, type)` | 把当前进程**加入**一个已存在的 namespace（fd 指向 `/proc/[pid]/ns/xxx`） |
| `unshare` 命令 | 命令行版 `unshare()`，如 `unshare --net --pid --fork bash` |
| `nsenter` 命令 | 命令行版 `setns()`，进入已有进程的 namespace，如 `nsenter -t <pid> -n ss -tan` |

```bash
# 新开一个 shell，拥有独立的网络命名空间（看不到宿主网卡）
sudo unshare --net --uts --fork bash
#   hostname container1
#   ip link        # 只有 lo

# 进入容器 PID 1234 的网络命名空间执行命令（排障神器）
sudo nsenter -t 1234 -n ss -tanp
```

`nsenter -t <容器进程pid> -n` 是**排查容器网络问题**的核心手段：在宿主机上「钻进」容器的网络栈抓包、看连接，无需进容器装工具。

### 2.1 clone 标志与 clone3

`clone()` 的 flags 决定新建哪些 namespace（可组合），还可与 `CLONE_VM`/`CLONE_FILES` 等共享标志组合构成「线程」（同一组 task，见 [L02](./L02-精通-进程与线程模型.md)）。现代内核推荐 `clone3()`——用结构体 `struct clone_args` 传参，更易扩展。组合 `CLONE_NEWUSER` 时，user namespace 会**先于**其他 namespace 建立，从而让新进程在新建的其他 namespace 里拥有特权（这正是 rootless 能建 net ns 的关键）。

### 2.2 setns 进入已有 namespace（代码）

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>

int fd = open("/proc/1234/ns/net", O_RDONLY);  // 目标进程的 net ns 句柄
setns(fd, CLONE_NEWNET);                        // 当前进程加入它
// 此后本进程的网络栈 == 进程 1234 的网络栈
close(fd);
```

`nsenter` 就是这段逻辑的命令行封装。排障常用组合：

```bash
nsenter -t <pid> -n ss -tanp        # 进 net ns 看连接（最常用）
nsenter -t <pid> -m ls /            # 进 mnt ns 看它的文件系统视图
nsenter -t <pid> -p -m ps aux       # 进 pid+mnt ns 看它眼中的进程树
unshare --user --map-root-user --net --pid --fork bash  # 非 root 建一组 ns 起 shell
```

---

## 第三章 各 namespace 详解

### 3.1 PID namespace：独立的 PID 1 与 init 语义

新建 PID namespace 后，第一个进程成为该 namespace 的 **PID 1**，承担 init 职责：**回收孤儿进程（reap zombies）**、对未处理信号有特殊语义。这就是容器里主进程是 PID 1 的来历——也带来一个经典坑：**PID 1 默认忽略没有 handler 的信号**（如 SIGTERM），且不自动 reap 子进程，导致容器无法优雅退出 / 僵尸堆积（呼应 [L02](./L02-精通-进程与线程模型.md) 的僵尸与 reaper、[L14 信号](./L14-精通-信号与IPC.md)）。解决方案是用 `tini`/`dumb-init` 这样的轻量 init 当 PID 1。

PID namespace 可嵌套：外层能看到内层进程（PID 不同），内层看不到外层。

### 3.2 mnt namespace：挂载点与传播

mnt namespace 隔离挂载点视图。难点是**挂载传播（mount propagation）**：

| 类型 | 行为 |
|---|---|
| `shared` | 挂载事件在副本间双向传播 |
| `private` | 不传播（容器常用，避免污染宿主） |
| `slave` | 单向：主→从传播，从的变化不影响主 |
| `unbindable` | 不可被 bind |

容器运行时通常把 rootfs 设为 `private`/`slave`，防止容器内挂载泄漏到宿主。`mount --make-private /` 是常见操作。

### 3.3 net namespace：独立网络栈与 veth pair

新建 net namespace 拥有全新的网络栈：只有 `lo`，无路由、无 iptables 规则。容器要联网，靠 **veth pair**（一对虚拟网卡，一端在容器 ns，一端在宿主接到 bridge）：

```bash
# 手动给一个 net namespace 接网（docker 自动做的事）
ip netns add ctr1
ip link add veth0 type veth peer name veth1
ip link set veth1 netns ctr1                  # 一端塞进容器 ns
ip netns exec ctr1 ip addr add 10.0.0.2/24 dev veth1
ip netns exec ctr1 ip link set veth1 up
# veth0 接到宿主 bridge，配置 NAT/路由即可出网
```

容器网络模型与 K8s Service 见 [cloud-native C03](../cloud-native/C03-精通-K8s-网络与-Service.md)；socket/连接在 net ns 内的行为见 [L13](./L13-精通-Socket-与连接管理.md)。

完整的「容器出网」要在宿主侧补上 bridge + NAT（这就是 Docker 默认 bridge 网络的本质）：

```bash
# 宿主侧：建网桥，把 veth0 接上，开转发与 NAT
ip link add br0 type bridge && ip link set br0 up
ip link set veth0 master br0
ip addr add 10.0.0.1/24 dev br0
ip netns exec ctr1 ip route add default via 10.0.0.1   # 容器默认路由指向网桥
sysctl -w net.ipv4.ip_forward=1                         # 开转发
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE  # 源地址伪装
```

K8s 的 CNI（Calico/Cilium）把这套换成更复杂的实现（见 [cloud-native C03](../cloud-native/C03-精通-K8s-网络与-Service.md)）；NAT 连接由 conntrack 跟踪，表满问题见 [L13](./L13-精通-Socket-与连接管理.md)。

### 3.4 user namespace：UID 映射与 rootless 的钥匙

user namespace 把容器内的 UID/GID **映射**到宿主上的不同 UID/GID。最强大之处：**容器内是 root（uid 0），映射到宿主上是一个普通非特权用户**。

```bash
cat /proc/<pid>/uid_map
#   0      100000      65536
#   ↑内部uid ↑外部起始  ↑范围：内部0~65535 映射到外部100000~165535
```

`/etc/subuid`、`/etc/subgid` 分配每个用户可用的 subordinate UID 段。这是 **rootless 容器**（Podman、rootless Docker）的基础：普通用户无需 sudo 即可跑容器，容器内的「root」在宿主上毫无特权，逃逸风险大幅降低。

**capabilities 与「假 root」**：user namespace 内的 root 拥有的是**该 namespace 内**的 capabilities（如可管理自己新建的 net ns），但对宿主资源无能为力——映射到宿主是非特权 uid，碰宿主文件按真实 uid 鉴权。这正是 rootless 安全的根基。

**嵌套**：user namespace 可层层嵌套（上限由 `/proc/sys/user/max_user_namespaces` 控制），每层进一步缩小特权范围。

### 3.5 time namespace（5.6+）

隔离 `CLOCK_MONOTONIC`、`CLOCK_BOOTTIME` 的偏移（不隔离墙钟 `CLOCK_REALTIME`）。主要用于 **checkpoint/restore（CRIU）** 后保持单调时钟连续。2026 年已稳定但用得相对少。

### 3.6 uts / ipc / cgroup namespace

| namespace | 隔离内容 | 实用点 |
|---|---|---|
| **uts** | hostname、domainname | 容器有独立主机名（`sethostname`）；最简单的 ns |
| **ipc** | System V IPC（共享内存/信号量/消息队列）、POSIX mq、`/dev/shm` | 容器间 IPC 互不可见（呼应 [L14](./L14-精通-信号与IPC.md)） |
| **cgroup** | cgroup 根视图 | 容器内 `/proc/self/cgroup` 看到的是相对自己根的路径，不泄漏宿主 cgroup 层级（呼应 [L17](./L17-精通-Cgroup-v2.md)） |

cgroup namespace 容易被忽略却重要：没有它，容器内能看到宿主完整 cgroup 路径（信息泄漏）；有了它，容器以为自己的 cgroup 就是根。

---

## 第四章 手搓一个迷你容器（约 90 行 C）

下面用 `clone` + 多个 namespace + `pivot_root` 跑一个隔离的 shell。它就是 `runc` 做的事的最小内核——只是没有 cgroup 限额、seccomp、cap drop 等。

```c
#define _GNU_SOURCE
#include <sched.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mount.h>
#include <sys/wait.h>
#include <sys/syscall.h>
#include <signal.h>
#include <string.h>

#define STACK_SIZE (1024 * 1024)
static char child_stack[STACK_SIZE];

// 新 rootfs 路径（需提前准备一个含 /bin/sh 的目录，如用 busybox 解压）
static const char *NEW_ROOT = "/tmp/mini-rootfs";

static int child_fn(void *arg) {
    // 此刻已在新的 mnt/pid/uts/ipc/net namespace 中
    sethostname("mini", 4);

    // 让挂载变为 private，避免污染宿主
    mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL);

    // bind 挂载新 rootfs 到自身（pivot_root 要求是挂载点）
    mount(NEW_ROOT, NEW_ROOT, NULL, MS_BIND | MS_REC, NULL);

    // 切换根：把旧根藏到 NEW_ROOT/.old
    char old[256];
    snprintf(old, sizeof(old), "%s/.old", NEW_ROOT);
    mkdir(old, 0777);
    if (syscall(SYS_pivot_root, NEW_ROOT, old) != 0) {
        perror("pivot_root"); return 1;
    }
    chdir("/");
    // 挂上新的 /proc（PID namespace 内独立的进程视图）
    mount("proc", "/proc", "proc", 0, NULL);
    // 卸掉旧根
    umount2("/.old", MNT_DETACH);

    char *args[] = {"/bin/sh", NULL};
    execv(args[0], args);     // 变身为 shell
    perror("execv");
    return 1;
}

int main(void) {
    int flags = CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWUTS |
                CLONE_NEWIPC | CLONE_NEWNET | SIGCHLD;
    pid_t pid = clone(child_fn, child_stack + STACK_SIZE, flags, NULL);
    if (pid < 0) { perror("clone"); exit(1); }
    printf("container started, host-side pid = %d\n", pid);
    waitpid(pid, NULL, 0);    // 父进程回收（它是容器 PID 1 在宿主的影子）
    return 0;
}
```

```bash
# 准备 rootfs（用 busybox 演示）
mkdir -p /tmp/mini-rootfs/bin /tmp/mini-rootfs/proc
cp $(which busybox) /tmp/mini-rootfs/bin/
ln -s busybox /tmp/mini-rootfs/bin/sh
gcc mini.c -o mini && sudo ./mini
# 进去后：hostname 是 mini；ps 只看到自己是 PID 1；ip link 只有 lo
```

加上 `CLONE_NEWUSER` 并写 `uid_map` 即可做到 rootless；加上把进程写入某个 cgroup（见 [L17](./L17-精通-Cgroup-v2.md)）即可限额。这就是容器的全部「魔法」。

---

## 第五章 user namespace 与 rootless 容器

rootless 的价值：**不给 docker daemon root，普通用户也能跑容器**。Podman 是代表。

- 容器内 `uid 0` → 宿主某个 subuid（如 100000），逃逸出来也只是个普通用户。
- 限制：绑定 <1024 端口、某些挂载、部分需要真 root 的操作受限，常配合 slirp4netns/pasta 做用户态网络。
- 安全收益显著，已成为 CI、开发环境、多租户的推荐形态。

**rootless 落地细节**：

- `newuidmap`/`newgidmap`（setuid 辅助程序）按 `/etc/subuid`、`/etc/subgid` 把 user ns 的 UID 段写进 `uid_map`/`gid_map`——普通用户自己不能随意写映射，靠这两个受控的特权小工具完成。
- 网络：rootless 无权建 veth/bridge，改用用户态网络栈 **slirp4netns** 或更快的 **pasta**，把容器流量在用户态转发。
- 存储：overlayfs 在 user ns 内的支持依赖较新内核；否则回退 fuse-overlayfs。

---

## 第六章 namespace 与容器安全

namespace 提供隔离，但**它不是安全边界的全部**。生产加固要点：

- **user namespace** 是最强的逃逸缓解：即便容器内 root 逃出，对宿主也只是非特权用户。
- **seccomp** 过滤危险系统调用（呼应 [L01](./L01-精通-Linux-架构与系统调用.md)）；**capabilities** 按需 drop（容器默认已 drop 大部分 cap，仅保留必要项）。
- **LSM**（AppArmor / SELinux / Landlock）再加一层强制访问控制。
- 已知风险面：共享内核（一个内核漏洞可能击穿所有容器）、`/proc` 与 `/sys` 暴露、以及 `--privileged`（等于拆掉隔离，极危险）。

容器安全 = namespace + cgroup + seccomp + capabilities + LSM 的**纵深防御**，详见 [cloud-native C12 安全](../cloud-native/C12-精通-K8s-安全.md)。

---

## 生产实践

- **进容器排障不靠 `docker exec`**：当容器内没有工具时，`nsenter -t <pid> -n/-m/-p ...` 从宿主钻进对应 namespace，用宿主的 `ss`/`tcpdump`/`ls` 直接看，最实用。
- **定位「幽灵挂载」**：容器内 mount 泄漏到宿主，多半是 rootfs 没设 `private`/`slave`。
- **PID 1 优雅退出**：容器主进程务必能处理 SIGTERM 或用 tini，否则 `docker stop` 要等 10s 超时被 SIGKILL（呼应 [L14](./L14-精通-信号与IPC.md)、[cloud-native C02](../cloud-native/C02-精通-K8s-工作负载.md)）。

---

## 陷阱清单

1. **现象**：容器 `docker stop` 总要等 10 秒 → **原因**：PID 1 没处理 SIGTERM（PID namespace 的 init 语义不自动响应）→ **修法**：应用 handle SIGTERM，或用 tini/dumb-init 当 PID 1。
2. **现象**：容器内产生大量僵尸进程 → **原因**：PID 1 不 reap 子进程 → **修法**：同上，用合格 init。
3. **现象**：容器内挂载污染了宿主 → **原因**：mnt 传播为 shared → **修法**：`mount --make-rprivate /` 或运行时设 private/slave。
4. **现象**：`pivot_root` 失败 EINVAL → **原因**：新 root 不是挂载点 → **修法**：先对新 rootfs 做 `mount --bind` 自身。
5. **现象**：rootless 容器无法监听 80 端口 → **原因**：user ns 内 root 在宿主无 `CAP_NET_BIND_SERVICE` → **修法**：用 >1024 端口或 `net.ipv4.ip_unprivileged_port_start`，或前置反代。
6. **现象**：`nsenter -n` 看不到容器连接 → **原因**：进错 pid 或 namespace 类型 → **修法**：确认目标 pid（容器主进程在宿主的 pid）与 `-n`（net）正确。
7. **现象**：容器内 `/proc` 显示宿主进程 → **原因**：没在 PID ns 内重新挂 `/proc` → **修法**：进 PID ns 后 `mount -t proc proc /proc`。
8. **现象**：rootless 容器 overlayfs 挂载失败 → **原因**：旧内核不支持 user ns 内 overlayfs → **修法**：升级内核或用 fuse-overlayfs。
9. **现象**：`ip netns exec` 找不到 Docker 容器的 ns → **原因**：Docker 默认不在 `/var/run/netns` 建链接 → **修法**：`ln -s /proc/<pid>/ns/net /var/run/netns/<name>`，或直接 `nsenter -t <pid> -n`。
10. **现象**：`--privileged` 容器影响到宿主 → **原因**：特权容器拆掉了 cap drop/seccomp，近乎无隔离 → **修法**：避免 privileged，按需 `--cap-add` 单个能力。

---

## 2026 现状

- **user namespace + rootless（Podman）** 已是安全容器主流形态；rootless Docker 也成熟。
- **time namespace（5.6+）** 稳定，主要服务 CRIU checkpoint/restore。
- cgroup namespace 标配，容器内只看到自己的 cgroup 子树（配合 [L17](./L17-精通-Cgroup-v2.md) 的 cgroup v2）。
- `runc`/`crun`/`containerd` 是落地 namespace+cgroup 的标准运行时；`crun`（C 实现）更轻量。
- 内核持续加固容器隔离：seccomp、user ns、Landlock LSM 等叠加使用。

---

## 练习题

1. （⭐）列出 8 种 namespace 及其隔离的资源。如何通过 `/proc/[pid]/ns/` 判断两个进程是否共享某个 namespace？
2. （⭐）`clone`、`unshare`、`setns` 三者的区别？`nsenter` 对应哪一个？
3. （⭐⭐）为什么容器主进程是 PID 1？这会带来哪两个经典问题？怎么解决？
4. （⭐⭐）net namespace 新建后只有 `lo`，容器是如何获得对外网络的？描述 veth pair 的接法。
5. （⭐⭐）user namespace 的 UID 映射如何让「容器内 root」在宿主上没有特权？这对 rootless 安全意味着什么？
6. （⭐⭐⭐）解释手搓容器中 `pivot_root` 前为什么要先 `mount --bind` 新 rootfs、为什么要把 `/` 设为 private。
7. （⭐⭐⭐）容器网络不通，你在宿主机上如何用 `nsenter` 进入它的网络命名空间抓包定位？给出命令。
8. （⭐⭐⭐）把第四章的迷你容器改造成 rootless（加 `CLONE_NEWUSER`），需要额外做哪些事（uid_map/gid_map、capabilities）？
9. （⭐⭐）uts / ipc / cgroup namespace 各隔离什么？为什么 cgroup namespace 对避免信息泄漏重要？
10. （⭐⭐⭐）rootless 容器为什么需要 `newuidmap`/`newgidmap` 和 slirp4netns/pasta？它们分别解决什么问题？
11. （⭐⭐⭐）为什么说「namespace 不是安全边界的全部」？容器安全还需叠加哪些机制？`--privileged` 为何危险？
