# 精通 systemd 与启动流程：UEFI→initramfs→systemd、unit/target、cgroup 集成、journald、排错

> 课程编号：L20
> 路线图来源：Linux · 模块七 可观测与生产
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，systemd 250+，本篇相关：systemd-boot/UKI、systemd-oomd、cgroup v2 集成）

---

## 引言：从按下电源到 login 提示符

「服务起不来」「机器开机特别慢」「重启后某个挂载没了」——这些日常运维问题，答案都在**启动流程**和 **systemd** 里。现代 Linux 发行版几乎清一色用 systemd 作为 PID 1（init），它不仅拉起服务，还接管了日志（journald）、资源控制（通过 [L17 cgroup](./L17-精通-Cgroup-v2.md)）、定时任务（timer 替代 cron）、网络、DNS 等。

本篇先走一遍「上电 → UEFI → bootloader → 内核 → initramfs → systemd → target」的完整链路，再讲 systemd 的核心模型（unit、依赖、并行启动），最后落到三件最实用的事：**看懂服务为什么 Failed、用 journalctl 查日志、用 systemd-analyze 定位开机慢**。systemd 作为 PID 1 的 init 语义见 [L02 进程](./L02-精通-进程与线程模型.md)；它用 cgroup 管理每个服务见 [L17](./L17-精通-Cgroup-v2.md)；systemd-oomd 见 [L06 OOM](./L06-精通-OOM-与内存诊断.md)。

---

## 第一章 启动全链路

```
上电
  │
  ▼
UEFI / BIOS 固件 ── 自检(POST)，加载 EFI 系统分区(ESP)里的 bootloader
  │
  ▼
bootloader (GRUB2 / systemd-boot) ── 选内核，加载 vmlinuz + initramfs 到内存，传 cmdline
  │
  ▼
内核解压、初始化 ── 探测 CPU/内存，建立基本子系统(见 L01 内核子系统)
  │
  ▼
initramfs (临时根文件系统，内存中) ── 加载真正 rootfs 所需驱动(如磁盘/LVM/加密/网络盘)
  │  挂载真正的 rootfs，switch_root
  ▼
/sbin/init  →  即 systemd (PID 1)
  │  解析 default.target，按依赖并行拉起所有 unit
  ▼
default.target (multi-user.target / graphical.target) ── 系统就绪，login
```

### 1.1 为什么需要 initramfs

内核要挂载根文件系统，但根文件系统可能在需要驱动才能访问的设备上（NVMe、LVM、LUKS 加密盘、iSCSI/NFS 根）。鸡生蛋问题：驱动在根文件系统里，但还没挂载根。**initramfs** 是打包进内存的临时小根，含这些驱动和工具，先把真正的 rootfs 挂起来，再 `switch_root` 切过去，最后执行 `/sbin/init`。

### 1.2 内核命令行（cmdline）

bootloader 传给内核的参数（`cat /proc/cmdline` 查看），如 `root=UUID=...`、`ro`、`quiet`、`systemd.unit=rescue.target`（强制进救援模式）。排障时在 GRUB 里临时加 `systemd.unit=emergency.target` 或 `init=/bin/bash` 是「进不去系统」的救命手段。

### 1.3 UEFI、Secure Boot 与 bootloader

现代机器用 **UEFI**（取代传统 BIOS）：固件直接从 **EFI 系统分区（ESP，FAT32）** 加载 `.efi` 程序。两类引导：

- **GRUB2**：功能全、支持多 OS、配置 `/boot/grub/grub.cfg`；
- **systemd-boot**：极简，直接列 ESP 上的内核条目，配置少，适合纯 UEFI + 单一发行版。

**Secure Boot**：固件只执行被信任密钥签名的引导程序/内核，防篡改。它推动了 **UKI**（统一内核镜像，见 2026 现状）——把内核 + initramfs + cmdline 打包成单个可签名 `.efi`。

### 1.4 initramfs 是怎么来的

initramfs 不是内核自带，而是**本机生成**的（含本机硬件驱动）：

- Debian/Ubuntu：`update-initramfs -u`
- Fedora/RHEL：`dracut -f`
- Arch：`mkinitcpio -P`

换内核、改磁盘加密/LVM 配置后**忘了重建 initramfs**，是「换完内核启动黑屏 / 找不到根」的常见原因。

---

## 第二章 systemd 的核心设计

### 2.1 PID 1、并行启动、按需激活

传统 SysV init 是**串行**跑脚本，慢。systemd 的革新：

- **并行启动**：根据依赖关系并行拉起无依赖冲突的服务；
- **socket activation**：systemd 先监听服务的 socket，**首个连接到来时才拉起服务**，既加速启动又实现按需；
- **D-Bus activation**：按 D-Bus 名称请求拉起服务；
- **基于依赖**而非固定顺序编排。

作为 PID 1，systemd 还负责 reap 孤儿进程（init 语义，见 [L02](./L02-精通-进程与线程模型.md)）。

**socket activation 完整示例**（`.socket` + `.service` 配对，按需拉起、加速启动）：

```ini
# myapp.socket
[Socket]
ListenStream=8080
[Install]
WantedBy=sockets.target
```
```ini
# myapp.service（无需 [Install]，由 socket 触发）
[Service]
ExecStart=/usr/bin/myapp     # 通过 sd_listen_fds() 接管 systemd 传入的 fd
```

`systemctl enable --now myapp.socket` 后，systemd 监听 8080；首个连接到来才启动 `myapp.service`，并把已建立的 socket fd 传给它。好处：开机快（不等服务真正起来）、空闲不占资源、平滑重启（socket 一直在，重启服务时连接不丢）。

### 2.2 依赖与排序（最易混淆）

**「依赖关系」和「启动顺序」是两件独立的事**：

| 指令 | 含义 |
|---|---|
| `Wants=` | **弱依赖**：尝试一起启动，目标失败不影响自己（最常用） |
| `Requires=` | **强依赖**：依赖失败则自己也失败/停止 |
| `Requisite=` | 要求依赖**已经启动**，否则立即失败（不主动拉起） |
| `BindsTo=` | 绑定生命周期：依赖停了自己也停 |
| `After=` / `Before=` | **仅排序**，不产生依赖关系 |
| `Conflicts=` | 互斥：启动一个会停另一个 |

经典坑：只写 `After=network.target` 不会拉起网络，它只管「如果网络要启动，排在它之后」。要确保网络就绪用 `Wants=network-online.target` + `After=network-online.target`。

---

## 第三章 unit 类型

systemd 把一切管理对象抽象为 **unit**，按后缀分类：

| 类型 | 后缀 | 作用 |
|---|---|---|
| **service** | `.service` | 守护进程/服务（最常见） |
| **socket** | `.socket` | 监听 socket，做 socket activation |
| **timer** | `.timer` | 定时任务，**替代 cron** |
| **mount** / **automount** | `.mount` | 挂载点（也可由 /etc/fstab 生成） |
| **target** | `.target` | 一组 unit 的同步点，**替代 runlevel** |
| **slice** | `.slice` | cgroup 资源层级分组（见 [L17](./L17-精通-Cgroup-v2.md)） |
| **scope** | `.scope` | 管理外部创建的进程（如容器、会话） |
| **path** | `.path` | 监控文件路径变化触发 |

### 3.1 service 的 Type

```ini
[Unit]
Description=My App
Wants=network-online.target
After=network-online.target

[Service]
Type=notify            # 进程就绪后调 sd_notify 告知 systemd
ExecStart=/usr/bin/myapp
Restart=on-failure
RestartSec=2
MemoryMax=512M         # → cgroup memory.max (见 L17)
CPUQuota=50%           # → cgroup cpu.max

[Install]
WantedBy=multi-user.target
```

| Type | 语义 |
|---|---|
| `simple` | ExecStart 即主进程（默认） |
| `forking` | 进程会 fork 后台并退出父进程，需配 `PIDFile=` |
| `notify` | 进程就绪后用 `sd_notify()` 通知，systemd 才认为启动完成（最可靠） |
| `oneshot` | 跑完即退出，常配 `RemainAfterExit=yes` |
| `dbus` | 取得 D-Bus 名称后视为就绪 |

### 3.2 target 替代 runlevel

| 传统 runlevel | systemd target |
|---|---|
| 3（多用户文本） | `multi-user.target` |
| 5（图形） | `graphical.target` |
| 1（单用户） | `rescue.target` |
| 紧急 | `emergency.target` |

```bash
systemctl get-default                          # 当前默认 target
systemctl set-default multi-user.target        # 改为文本模式启动
systemctl isolate rescue.target                # 立即切到救援模式
```

### 3.3 timer 替代 cron

```ini
# backup.timer
[Timer]
OnCalendar=*-*-* 03:00:00      # 每天 3 点
Persistent=true                # 错过了开机补跑
[Install]
WantedBy=timers.target
```

相比 cron，timer 的优势：日志进 journald、可设依赖、`Persistent` 补跑、资源可控（在 cgroup 内）。

### 3.4 模板单元（一份模板，多个实例）

`name@.service` 是**模板**，`@` 后的实例名作为 `%i` 传入，一份文件起多个实例：

```ini
# worker@.service
[Service]
ExecStart=/usr/bin/worker --id %i
```
```bash
systemctl start worker@1 worker@2 worker@3   # 起三个实例
```

`getty@tty1.service` 就是模板单元的经典应用。

### 3.5 drop-in、条件与安全加固

- **drop-in 覆盖**：`systemctl edit myapp` 生成 `/etc/systemd/system/myapp.service.d/override.conf`，**只覆盖个别字段**，不动原 unit（包升级不丢改动）。
- **条件**：`ConditionPathExists=`、`ConditionFileNotEmpty=` 等，不满足则跳过启动（不计失败）。
- **启停钩子**：`ExecStartPre=`/`ExecStartPost=`/`ExecStopPost=`。
- **安全加固指令**（把容器级隔离带给普通服务，呼应 [L16](./L16-精通-Namespace.md) 的 namespace/能力）：`ProtectSystem=strict`（系统目录只读）、`PrivateTmp=yes`（独立 /tmp）、`NoNewPrivileges=yes`、`ProtectHome=yes`、`CapabilityBoundingSet=`、`SystemCallFilter=`（seccomp 过滤）。

```bash
systemd-analyze security myapp.service   # 给服务暴露面打分，指导加固
```

---

## 第四章 systemd 与 cgroup 集成

systemd 是 cgroup 的「**唯一合法写者**」：每个 service 自动获得一个 cgroup，用 slice 组织成层级：

```
-.slice (根)
├── system.slice/          # 系统服务
│   ├── nginx.service/
│   └── sshd.service/
├── user.slice/            # 用户会话
│   └── user-1000.slice/
└── machine.slice/         # 虚拟机/容器
```

资源控制指令（`MemoryMax`/`CPUQuota`/`IOWeight`/`TasksMax`）直接落到对应 cgroup 文件（见 [L17](./L17-精通-Cgroup-v2.md)）。运行时调整用 `systemctl set-property`，**不要手改 `/sys/fs/cgroup`**（systemd 会覆盖）。

```bash
systemd-cgls                       # 查看 cgroup 树（按 slice/service）
systemd-cgtop                      # 按 cgroup 实时看 CPU/内存/IO
systemctl set-property nginx.service MemoryMax=1G
```

---

## 第五章 journald：结构化日志

systemd 自带日志系统 journald，收集内核、服务 stdout/stderr、syslog，统一结构化存储。

```bash
journalctl -u nginx.service            # 某服务的日志
journalctl -u nginx -f                 # 实时跟随（像 tail -f）
journalctl -b                          # 本次开机以来；-b -1 上次开机
journalctl -p err                      # 仅 error 及以上优先级
journalctl --since "1 hour ago" --until now
journalctl -k                          # 仅内核消息（dmesg）
journalctl -u myapp -o json-pretty     # 结构化字段（含 _PID/_UID/_SYSTEMD_UNIT 等）
```

**持久化**：默认日志可能只在内存（`/run/log/journal`，重启丢失）。要持久化：

```bash
mkdir -p /var/log/journal            # 存在即持久化（Storage=auto）
# 或在 /etc/systemd/journald.conf 设 Storage=persistent
systemctl restart systemd-journald
```

可与 rsyslog 共存（journald 转发给 rsyslog 落地 `/var/log/messages`）。

**高级查询与磁盘管理**：

```bash
journalctl _SYSTEMD_UNIT=nginx.service _PID=1234   # 按结构化字段精确过滤
journalctl -g 'timeout|refused' -p warning          # -g 正则搜索 + 优先级过滤
journalctl --disk-usage                              # journal 占了多少磁盘
journalctl --vacuum-size=500M                        # 收缩到 500M
journalctl --vacuum-time=7d                          # 只保留 7 天
```

journald 有**速率限制**（`RateLimitIntervalSec`/`RateLimitBurst`）：刷日志的服务可能被丢弃部分日志——发现日志「缺了一段」时，先排查是否触发限流。

---

## 第六章 排错三板斧

### 6.1 服务为什么 Failed

```bash
systemctl status myapp.service        # 状态 + 最近几行日志 + 主进程退出码
systemctl list-units --failed         # 所有失败的 unit
journalctl -u myapp -b --no-pager      # 本次启动该服务完整日志
systemctl cat myapp.service           # 看最终生效的 unit 文件（含 drop-in）
```

看 `status` 的 `Active:` 行与 `Main PID:` 的退出码/信号，结合 journal 即可定位。改配置用 `systemctl edit`（生成 drop-in，不动原文件），改完 `systemctl daemon-reload`。

### 6.2 开机为什么慢

```bash
systemd-analyze                       # 总耗时（firmware/loader/kernel/userspace 分段）
systemd-analyze blame                 # 各 unit 耗时降序——找最慢的服务
systemd-analyze critical-chain        # 关键路径（串行依赖链上的耗时）
systemd-analyze plot > boot.svg       # 可视化时间线
```

`blame` 列出耗时最长的服务，但要结合 `critical-chain`——`blame` 里慢的服务若是并行的，未必拖慢总时间；关键路径上的才是真凶。

### 6.3 进不去系统

- GRUB 里临时把启动 target 改成 `emergency.target`（最小环境）或 `rescue.target`；
- 或加 `init=/bin/bash` 直接拿到 shell（根可能只读，`mount -o remount,rw /`）；
- 常用于修坏的 `/etc/fstab`（某挂载失败导致启动卡住）。

---

## 生产实践

- **服务起不来先三连**：`systemctl status` → `journalctl -u xxx -b` → `systemctl cat xxx`，九成问题当场定位（配置语法、ExecStart 路径、权限、依赖未满足）。
- **优雅退出**：服务要正确处理 SIGTERM（systemd `stop` 先发 SIGTERM，`TimeoutStopSec` 后才 SIGKILL）；容器同理（见 [L16](./L16-精通-Namespace.md)、[L14 信号](./L14-精通-信号与IPC.md)）。
- **资源失控的服务**：用 `MemoryMax`/`CPUQuota` 给它套上 cgroup 限额（L17），避免拖垮整机；配合 `systemd-oomd`（基于 PSI）按压力杀（见 [L06](./L06-精通-OOM-与内存诊断.md)）。
- **`/etc/fstab` 改完别急着重启**：`systemctl daemon-reload && mount -a` 先验证，避免下次开机因挂载失败卡在 emergency。

---

## 陷阱清单

1. **现象**：服务依赖网络却在网络就绪前启动失败 → **原因**：只写了 `After=network.target`（仅排序，且该 target 不代表网络可用）→ **修法**：`Wants=network-online.target` + `After=network-online.target`。
2. **现象**：改了 unit 文件不生效 → **原因**：没 `daemon-reload` → **修法**：`systemctl daemon-reload` 后再 restart。
3. **现象**：`Type=forking` 服务被误判为启动失败 → **原因**：没设 `PIDFile=` 或进程没正确 fork → **修法**：正确配置 PIDFile，或改用 `Type=notify`/`simple`。
4. **现象**：日志重启后全没了 → **原因**：journald 默认存内存（无 `/var/log/journal`）→ **修法**：建 `/var/log/journal` 或设 `Storage=persistent`。
5. **现象**：`systemd-analyze blame` 显示某服务很慢，但优化它总时间没变 → **原因**：它不在关键路径上（并行）→ **修法**：看 `critical-chain` 找真正串行瓶颈。
6. **现象**：手改 `/sys/fs/cgroup` 下服务限额，重启失效 → **原因**：systemd 覆盖 → **修法**：`systemctl set-property` 或 unit 指令（见 [L17](./L17-精通-Cgroup-v2.md)）。
7. **现象**：改坏 `/etc/fstab` 后系统卡在启动 → **原因**：某挂载失败阻塞 → **修法**：进 emergency，`mount -o remount,rw /` 修复 fstab；平时改完先 `mount -a` 验证。
8. **现象**：换内核后启动黑屏 / 找不到根 → **原因**：忘了重建 initramfs（缺新驱动）→ **修法**：`dracut -f` 或 `update-initramfs -u` 重建。
9. **现象**：`systemctl start` 某服务无反应、`status` 显示 masked → **原因**：被 `systemctl mask` 屏蔽（软链到 /dev/null）→ **修法**：`systemctl unmask`。
10. **现象**：服务日志「缺了一段」 → **原因**：journald 速率限制丢弃 → **修法**：调 `RateLimitBurst`/`RateLimitIntervalSec` 或降低日志量。

---

## 2026 现状

- **systemd 250+** 普及，功能持续扩张（networkd/resolved/homed/oomd）。
- **systemd-boot** 与 **UKI（Unified Kernel Image，统一内核镜像）** 兴起：把内核、initramfs、cmdline 打包签名为单个 EFI 可执行，简化安全启动（Secure Boot）链路。
- **systemd-oomd** 基于 PSI 做用户态 OOM，优于内核 OOM killer 的「最后一刻才动手」（见 [L06](./L06-精通-OOM-与内存诊断.md)）。
- cgroup v2 + systemd unified 是默认形态，K8s `cgroupDriver: systemd` 推荐（见 [L17](./L17-精通-Cgroup-v2.md)、[cloud-native C06](../cloud-native/C06-精通-Scheduling-与资源管理.md)）。
- timer 在很多场景替代 cron；journald 结构化日志成主流，常与 Loki/rsyslog 对接（见 [cloud-native C11](../cloud-native/C11-精通-K8s-可观测性.md)）。

---

## 练习题

1. （⭐）按顺序写出从上电到 login 的启动链路主要阶段。initramfs 解决了什么「鸡生蛋」问题？
2. （⭐）`multi-user.target` 与 `graphical.target` 对应传统哪个 runlevel？如何查看和修改默认 target？
3. （⭐⭐）`Wants=`、`Requires=`、`After=` 三者的区别？为什么 `After=network.target` 不能保证网络可用？
4. （⭐⭐）`Type=simple`/`forking`/`notify`/`oneshot` 各自的就绪判定语义是什么？什么时候该用 notify？
5. （⭐⭐）systemd 如何用 slice/service 把服务组织进 cgroup？运行时调整某服务内存上限的正确命令是什么？
6. （⭐⭐⭐）一个服务反复 Failed，给出用 `systemctl status`/`journalctl`/`systemctl cat` 的完整定位流程，并说明各自能看到什么。
7. （⭐⭐⭐）开机变慢，如何用 `systemd-analyze blame` 与 `critical-chain` 配合定位真正瓶颈？为什么不能只看 blame？
8. （⭐⭐⭐）改坏 `/etc/fstab` 后系统无法正常启动，如何通过内核 cmdline / 救援 target 进入并修复？
9. （⭐⭐）socket activation 的工作流程是什么？相比常驻服务有哪些好处？
10. （⭐⭐）模板单元 `name@.service` 与 drop-in 覆盖各解决什么问题？
11. （⭐⭐⭐）要把一个普通服务「沙箱化」，列举至少 4 个 systemd 安全加固指令及作用，并说明它们与 namespace/seccomp 的关系。

---

## 参考答案

1. 阶段：上电 → UEFI/BIOS 固件自检并从 ESP 加载 bootloader → bootloader（GRUB2/systemd-boot）选内核、把 vmlinuz + initramfs 加载到内存并传 cmdline → 内核解压初始化 → initramfs（内存中临时根）加载访问真正 rootfs 所需驱动、挂载 rootfs 并 `switch_root` → 执行 `/sbin/init`（即 systemd，PID 1）→ 解析 default.target 并按依赖并行拉起 unit → 到达 default.target，login。initramfs 解决的鸡生蛋问题：内核要挂载根文件系统，但根可能在需要驱动才能访问的设备上（NVMe/LVM/LUKS/iSCSI/NFS），而驱动又在根里；initramfs 打包这些驱动先把真正的 rootfs 挂起来再切过去。

2. `multi-user.target` 对应 runlevel 3（多用户文本），`graphical.target` 对应 runlevel 5（图形）。查看默认 target：`systemctl get-default`；修改：`systemctl set-default multi-user.target`（或用 `systemctl isolate <target>` 立即切换当前运行的 target）。

3. `Wants=` 是弱依赖：尝试一起启动，被依赖目标失败不影响自己（最常用）；`Requires=` 是强依赖：依赖失败则自己也失败/停止；`After=` 仅定义启动顺序（排在某 unit 之后），不产生任何依赖、不会主动拉起对方。`After=network.target` 不能保证网络可用，是因为它只管「如果网络要启动，我排在它之后」，且 `network.target` 本身只代表网络栈配置阶段、不代表网络真正连通；要确保网络就绪须用 `Wants=network-online.target` + `After=network-online.target`。

4. 就绪判定语义：`simple`——ExecStart 拉起的进程即主进程，fork/exec 成功即视为启动完成（默认）；`forking`——进程会 fork 出后台守护并退出父进程，需配 `PIDFile=` 让 systemd 跟踪真正的主进程；`notify`——进程初始化完成后主动调 `sd_notify()` 通知 systemd，systemd 收到才认为启动完成（最可靠）；`oneshot`——跑完即退出，常配 `RemainAfterExit=yes`。当服务有明确的「我已就绪可服务」时点、且希望依赖它的服务等到它真正可用时，应用 `notify`。

5. systemd 是 cgroup 的唯一合法写者：每个 service 自动获得一个 cgroup，用 slice 组织成层级（`-.slice` → `system.slice`/`user.slice`/`machine.slice` → 具体 `.service`）。`MemoryMax`/`CPUQuota`/`IOWeight`/`TasksMax` 等指令直接落到对应 cgroup 文件。运行时调整内存上限的正确命令：`systemctl set-property nginx.service MemoryMax=1G`（不要手改 `/sys/fs/cgroup`，systemd 会覆盖）。

6. 流程：(1) `systemctl status myapp.service` —— 看 `Active:` 状态、`Main PID:` 的退出码/信号和最近几行日志，初判是崩溃、配置错还是依赖未满足；(2) `journalctl -u myapp -b --no-pager` —— 看本次启动该服务的完整日志，定位具体报错（ExecStart 路径错、权限、端口占用、依赖服务未起等）；(3) `systemctl cat myapp.service` —— 看最终生效的 unit 文件（含 drop-in 覆盖），确认配置是否如预期。改配置用 `systemctl edit` 生成 drop-in，改完 `systemctl daemon-reload` 再 restart。

7. `systemd-analyze blame` 按耗时降序列出各 unit 的启动时间，`systemd-analyze critical-chain` 显示串行依赖链（关键路径）上的耗时。配合用法：先看 critical-chain 找出真正在串行关键路径上的 unit，再到 blame 里看它的耗时去优化。不能只看 blame，是因为 blame 里耗时长的服务若是并行启动的，优化它并不会缩短总启动时间——只有关键路径上的串行瓶颈才决定总耗时。

8. fstab 改坏导致某挂载失败会阻塞启动卡在 emergency。修复：在 GRUB 编辑启动项，在内核 cmdline 临时加 `systemd.unit=emergency.target`（或 `rescue.target`），进入最小环境；或加 `init=/bin/bash` 直接拿 shell。此时根常为只读，先 `mount -o remount,rw /` 改为可写，再编辑 `/etc/fstab` 修正错误条目，`systemctl daemon-reload && mount -a` 验证无误后重启。平时改 fstab 后应先 `mount -a` 验证再重启。

9. socket activation 流程：先 `enable --now` 一个 `.socket` unit，systemd 自己监听该 socket（如 `ListenStream=8080`）；首个连接到来时 systemd 才拉起配对的 `.service`，并通过 `sd_listen_fds()` 把已建立的 socket fd 传给服务进程。相比常驻服务的好处：开机更快（不必等服务真正起来即可「就绪」）、空闲时不占资源（按需拉起）、平滑重启（socket 一直由 systemd 持有，重启服务时已排队/在途的连接不丢）。

10. 模板单元 `name@.service` 解决「一份配置起多个实例」——`@` 后的实例名作为 `%i` 传入，`systemctl start worker@1 worker@2` 即用同一模板起多个实例（如 `getty@tty1`）。drop-in 覆盖（`systemctl edit` 生成 `xxx.service.d/override.conf`）解决「只改个别字段而不动原 unit 文件」——包升级覆盖原 unit 时自定义改动不丢失。

11. 至少四个加固指令：`ProtectSystem=strict`（把系统目录设为只读，防服务篡改系统文件）、`PrivateTmp=yes`（给服务独立的 /tmp，隔离临时文件）、`NoNewPrivileges=yes`（禁止通过 setuid 等提权）、`ProtectHome=yes`（隐藏/只读用户家目录）、`CapabilityBoundingSet=`（限制可保留的 Linux capabilities）、`SystemCallFilter=`（seccomp 系统调用过滤）。它们与 namespace/seccomp 的关系：systemd 在底层正是用 Linux namespace（mount/user 等，见 L16）实现 `PrivateTmp`/`ProtectSystem`/`ProtectHome` 这类文件系统/视图隔离，用 capabilities drop（`CapabilityBoundingSet`）和 seccomp（`SystemCallFilter` 即 seccomp-bpf 过滤危险系统调用）收紧权限——本质是把容器级的隔离手段（namespace + capabilities + seccomp）以声明式指令带给普通服务。可用 `systemd-analyze security <unit>` 给暴露面打分指导加固。
