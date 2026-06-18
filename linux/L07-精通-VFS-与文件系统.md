# 精通 VFS 与文件系统：inode、dentry、ext4 日志、page cache 回写、fsync

> 课程编号：L07
> 路线图来源：Linux · 模块三 文件与 I/O
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟
> 内容基准：2026 年 6 月（Linux 6.12 LTS / 6.15+，本篇相关：VFS 四大对象、ext4/xfs/btrfs/overlayfs、folio 化的 page cache、writeback、fsync/O_DIRECT 语义）

---

## 引言：一切皆文件，VFS 凭什么统一

Unix 的设计哲学里有一句被引用到滥的话——"一切皆文件"。但很少有人真正追问：**凭什么** `read(fd, buf, n)` 这一个系统调用，既能读一个 ext4 上的普通文件，又能读一个 NFS 远端文件，又能读 `/proc/self/status` 这种压根不在磁盘上的虚拟文件，还能读一个管道、一个 socket、一个块设备？

答案是 **VFS（Virtual File System，虚拟文件系统）**——内核在所有具体文件系统之上抽象出来的一层"统一接口层"。它定义了一组对象（superblock、inode、dentry、file）和一组操作表（`super_operations`、`inode_operations`、`dentry_operations`、`file_operations`）。任何文件系统只要实现这些操作表，就能挂进 VFS，从此对用户态呈现完全一致的 `open`/`read`/`write`/`close` 语义。

来看一个反直觉的现象。你在数据库机器上跑：

```bash
$ dd if=/dev/zero of=test.dat bs=1M count=1000 oflag=direct
1048576000 bytes (1.0 GB) copied, 3.2 s, 327 MB/s

$ dd if=/dev/zero of=test.dat bs=1M count=1000
1048576000 bytes (1.0 GB) copied, 0.41 s, 2.5 GB/s
```

同一块盘、同样的 1 GB，加了 `oflag=direct` 慢了 8 倍。第二条"快"的命令其实根本没把数据落盘——它只是写进了 **page cache**，`dd` 返回时数据还在内存里，要等 writeback 线程慢慢刷。如果此刻断电，这 1 GB 大概率丢失。这就是 VFS / page cache 给你的"善意的谎言"：**write 返回 ≠ 数据持久化**。理解这层谎言，是写对数据库、消息队列、任何要持久化的系统的前提。

本篇从 VFS 四大对象拆起，一路到 ext4 的 extent 与日志、overlayfs 的分层、page cache 的脏页回写，最后讲清 `fsync` / `O_DIRECT` 到底保证了什么、又放大了多少。读完你应该能完整还原"一次 `write()` 从用户缓冲区到磁盘盘片"的全路径，并知道每一步在哪里可能丢数据、在哪里可能卡顿。

---

## 第一章 VFS 四大对象：superblock / inode / dentry / file

### 1.1 四个对象各自管什么

VFS 用四个核心对象描述"文件系统的一切"。先建立直觉：

| 对象 | 内核结构 | 代表什么 | 生命周期 | 一句话 |
|---|---|---|---|---|
| superblock | `struct super_block` | 一个已挂载的文件系统实例 | 挂载→卸载 | "这块盘是 ext4，块大小 4K，根 inode 是 2 号" |
| inode | `struct inode` | 一个文件的元数据（与名字无关） | 文件存在期间 | "12345 号文件，4096 字节，属主 1000，权限 644" |
| dentry | `struct dentry` | 路径中的一个名字（目录项） | 缓存，可回收 | "目录 X 里有个名字叫 foo，它指向 inode 12345" |
| file | `struct file` | 一个进程打开文件的上下文 | open→close | "进程 P 以只读方式打开了它，当前偏移 4096" |

它们的关系可以画成：

```
      进程 A fd=3 ──┐
                    ├──► struct file (O_RDONLY, pos=0) ──┐
      进程 B fd=5 ──┘  (dup / fork 后可共享同一个 file)   │
                                                          ▼
      "/var/log/app.log"                            struct dentry "app.log"
        路径解析得到 ──────────────────────────────►  d_inode ──┐
                                                                 ▼
                                                          struct inode #12345
                                                          (元数据 + i_mapping page cache)
                                                                 │
                                                                 ▼
                                                          struct super_block (ext4 /var)
```

**关键洞察**：inode 不含文件名。"文件名"住在 dentry 里。一个 inode 可以被多个 dentry 指向——这就是**硬链接**。一个打开的 file 通过 dentry 找到 inode，所有真正的数据和元数据都挂在 inode 上。

### 1.2 superblock 与 super_operations

`struct super_block` 描述一个**已挂载的文件系统实例**。注意是"实例"——同一个 ext4 驱动可以挂载多个分区，每个分区一个 superblock。

```c
// include/linux/fs.h (大幅简化)
struct super_block {
    struct list_head    s_list;        // 全局 superblock 链表
    dev_t               s_dev;         // 设备号
    unsigned long       s_blocksize;   // 块大小（通常 4096）
    struct file_system_type *s_type;   // 指向 ext4 / xfs 的 fs 类型
    const struct super_operations *s_op; // 操作表
    struct dentry       *s_root;       // 根目录的 dentry
    struct list_head    s_inodes;      // 本 fs 所有 inode
    void                *s_fs_info;    // 具体 fs 的私有数据（如 ext4_sb_info）
};
```

`super_operations` 里是文件系统级别的回调：`alloc_inode`（分配一个 inode 内存对象）、`write_inode`（把 inode 元数据写回磁盘）、`sync_fs`（同步整个 fs）、`statfs`（`df` 背后）等。

```bash
# 查看已挂载的文件系统（每个就是一个 superblock 实例）
$ findmnt -t ext4,xfs
TARGET  SOURCE    FSTYPE OPTIONS
/       /dev/nvme0n1p2 ext4   rw,relatime
/data   /dev/nvme1n1   xfs    rw,relatime,attr2,inode64
```

### 1.3 inode 与 inode_operations

inode（index node）是文件的"身份证"：

```c
struct inode {
    umode_t         i_mode;     // 文件类型 + 权限 (S_IFREG | 0644)
    kuid_t          i_uid;
    kgid_t          i_gid;
    loff_t          i_size;     // 文件大小
    struct timespec64 i_atime, i_mtime, i_ctime;
    unsigned long   i_ino;      // inode 号（fs 内唯一）
    const struct inode_operations *i_op;
    const struct file_operations  *i_fop; // 打开它得到的 file 用哪个 fops
    struct address_space *i_mapping;       // ★ page cache 入口
    struct super_block *i_sb;
    blkcnt_t        i_blocks;   // 占用的 512 字节块数
    // ...
};
```

`inode_operations` 是**作用在 inode/名字空间上**的操作：`lookup`（在目录里查名字）、`create`、`link`、`unlink`、`mkdir`、`rename`、`setattr`（chmod/chown）等。注意它管的是"元数据和命名"，**不管数据读写**——读写归 `file_operations`。

观测一个真实 inode：

```bash
$ stat /etc/hostname
  File: /etc/hostname
  Size: 12          Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 131074      Links: 1
Access: (0644/-rw-r--r--)  Uid: (0/root)   Gid: (0/root)
Access: 2026-06-17 10:00:01
Modify: 2026-06-01 08:30:00
Change: 2026-06-01 08:30:00
```

`Inode: 131074` 就是 `i_ino`。`Links: 1` 是硬链接数 `i_nlink`——当它降到 0 且没有进程打开时，inode 才真正被释放、数据块才回收。这解释了一个经典现象：**删掉一个正在被进程写的大日志文件，磁盘空间不会释放**——因为 `i_nlink=0` 但还有 open 着的 file 持有引用。

### 1.4 dentry 与挂载点

dentry（directory entry）是路径解析的核心。每个 dentry 代表路径中的**一段名字**：

```c
struct dentry {
    struct dentry      *d_parent;   // 父目录 dentry
    struct qstr        d_name;      // 名字（如 "app.log"）含预算好的 hash
    struct inode       *d_inode;    // 指向的 inode（negative dentry 时为 NULL）
    const struct dentry_operations *d_op;
    struct super_block *d_sb;
    struct hlist_bl_node d_hash;    // 挂进全局 dentry hash 表
    struct list_head   d_subdirs;   // 子 dentry
    // ...
};
```

解析 `/var/log/app.log` 时，VFS 从根 dentry 开始，逐段：`/` → `var` → `log` → `app.log`，每一段都是一次 dentry 查找。dentry 把"路径字符串"翻译成"inode 指针"，是路径解析的加速结构（详见第二章）。

**挂载点**是 dentry 体系里一个精妙的设计。当你 `mount /dev/nvme1n1 /data` 时，`/data` 这个 dentry 上"盖了"一个新的 superblock 的根 dentry。路径解析走到挂载点 dentry 时，VFS 通过 mount 树（`struct vfsmount` / `struct mount`）跳转到被挂载文件系统的根 dentry，继续往下解析。这就是为什么 `cd /data` 后看到的是新盘的内容，而 `/data` 本身的 dentry 还属于父文件系统。

```bash
# 挂载关系的树形视图
$ findmnt
TARGET                SOURCE         FSTYPE   OPTIONS
/                     /dev/nvme0n1p2 ext4     rw,relatime
├─/boot/efi           /dev/nvme0n1p1 vfat     rw
├─/data               /dev/nvme1n1   xfs      rw,relatime
└─/var/lib/docker/... overlay        overlay  rw,...  # ← 容器分层，见第五章
```

### 1.5 file 与 file_operations

`struct file` 代表**一次打开**。同一个文件被打开两次，就有两个 `struct file`，各自有独立的偏移量 `f_pos`：

```c
struct file {
    struct path        f_path;      // 含 dentry + vfsmount
    struct inode       *f_inode;    // 缓存的 inode 指针
    const struct file_operations *f_op;
    spinlock_t         f_lock;
    fmode_t            f_mode;       // FMODE_READ / FMODE_WRITE
    loff_t             f_pos;        // ★ 当前读写偏移
    struct fown_struct f_owner;
    unsigned int       f_flags;      // O_NONBLOCK / O_APPEND / O_DIRECT
    struct address_space *f_mapping;
    atomic_long_t      f_count;      // 引用计数
};
```

`file_operations` 才是真正的读写入口：`read_iter`、`write_iter`、`mmap`、`fsync`、`poll`、`unlocked_ioctl` 等。不同文件系统、不同设备提供不同的 fops——这正是"一切皆文件"的实现机制：socket 有 socket 的 fops，pipe 有 pipe 的 fops，ext4 普通文件用 `ext4_file_operations`。

fd → file → inode 的三级映射会在第三章详述。这里记住一句：**fd 是进程局部的小整数，file 是打开上下文（可共享），inode 是文件本体。**

---

## 第二章 dentry / inode cache：路径解析的加速器

### 2.1 路径解析为什么慢，dcache 怎么救

每次 `open("/usr/local/bin/python3")` 都要从根开始逐段查找。如果每一段都去磁盘读目录块、找名字、读 inode，那 `open` 会慢到不可接受。内核用 **dcache（dentry cache）** 把"路径段 → dentry"的结果缓存在内存中。

dcache 是一个全局哈希表，key 是 `(父 dentry 指针, 名字 hash)`，value 是 dentry。解析路径时，每段先查 dcache，命中就直接拿到 dentry 和其 `d_inode`，完全不碰磁盘。

```
路径解析 /usr/local/bin (walk)
  ┌────────────┐ lookup "usr"   ┌────────────┐
  │ root dentry│───────────────►│ usr dentry │  命中 dcache → 0 次磁盘 IO
  └────────────┘                └────────────┘
                                       │ lookup "local"
                                       ▼
                                 ┌────────────┐
                                 │local dentry│  未命中 → 调 inode->i_op->lookup
                                 └────────────┘     读目录块、建 dentry、插 dcache
```

现代内核的路径解析还有一条**无锁快速路径 RCU-walk**：用 seqlock + RCU 在不取任何 dentry 引用计数的情况下走完整条路径，只有冲突或需要回调时才退化为持引用的 ref-walk。这让高并发下的 `stat`/`open` 几乎零锁竞争。详见 L15 的 RCU 章节。

```bash
# 观测 dcache / inode cache 占用
$ cat /proc/sys/fs/dentry-state
180423  165001  45      0       12003   0   # nr_dentry / nr_unused / ...

$ slabtop -o | grep -E 'dentry|inode'
 412160 410203  99%    0.19K  ...  dentry
 198400 197001  99%    0.62K  ...  ext4_inode_cache
```

### 2.2 negative dentry：缓存"不存在"

dcache 不仅缓存"存在的文件"，还缓存**"不存在的文件"**——这就是 **negative dentry**（`d_inode == NULL`）。

为什么要缓存"不存在"？想象一个程序反复 `stat("/etc/myapp.conf")` 来探测配置文件是否出现。如果文件确实不存在，没有 negative dentry 的话，每次 `stat` 都要走完整路径解析、去目录里实际查一遍、确认没有，再返回 ENOENT——浪费。有了 negative dentry，第一次查无此名后建立一个 negative dentry，后续 `stat` 直接命中并秒回 ENOENT。

negative dentry 的典型来源还有：动态链接器 `ld.so` 启动时按 `LD_LIBRARY_PATH` 顺序挨个目录探测 `.so`（大量 ENOENT）、Python `import` 时按 `sys.path` 探测模块文件。一个 Python 进程启动可能制造成千上万个 negative dentry。

**陷阱预告**：negative dentry 通常很轻量且会被内存压力回收，但在某些病态负载下（如海量随机不存在路径探测、邮件队列、缓存目录扫描）它们会堆积到吃掉几个 GB 的可回收内存（`SReclaimable`）。表现为 `free` 里 cache 巨大、`slabtop` 里 dentry 数千万。回收方法：`echo 2 > /proc/sys/vm/drop_caches`（仅排障用，生产慎用，详见陷阱清单）。

### 2.3 缓存效应与 inode cache

inode cache 缓存 `struct inode` 内存对象（每文件系统一个 slab，如 `ext4_inode_cache`）。dentry 命中后通过 `d_inode` 直接拿到 inode，省去从磁盘读 inode 块。

两级缓存的协同效应：**热路径的 open 是纯内存操作，零磁盘 IO**。这就是为什么一个 Web 服务反复 open 同一批静态文件、一个编译反复 stat 同一批头文件，第二次往后飞快。冷启动（首次访问、刚 `drop_caches`、或缓存被内存压力逐出）则要付磁盘 IO 的代价——这是"为什么重启后第一次访问慢"的根因之一。

```bash
# 用 funccount / bpftrace 观察路径解析是否命中缓存
$ bpftrace -e 'kprobe:lookup_slow { @miss = count(); }
               kprobe:__d_lookup_rcu { @fast = count(); }'
# @fast 远大于 @miss 说明缓存命中良好
```

---

## 第三章 文件操作路径：open / read / write / close 内核流程

### 3.1 fd → struct file → inode 的三级映射

每个进程有一张文件描述符表 `struct files_struct`，核心是 `fdtable`，本质是一个 `struct file *` 数组，下标就是 fd：

```c
struct files_struct {
    atomic_t count;
    struct fdtable __rcu *fdt;
    // ...
};
struct fdtable {
    unsigned int max_fds;
    struct file __rcu **fd;   // fd[3] 就是 fd=3 对应的 struct file
    unsigned long *open_fds;  // 位图：哪些 fd 已用
};
```

三级映射：

```
进程 fd 表            打开文件表           inode 表
┌─────────┐         ┌──────────────┐    ┌──────────┐
│ fd[0]   │────────►│ struct file  │───►│  inode   │
│ fd[1]   │────────►│  f_pos, mode │    │ (本体)   │
│ fd[2]   │──┐      └──────────────┘    └──────────┘
│ fd[3]   │  │      ┌──────────────┐         ▲
└─────────┘  └─────►│ struct file  │─────────┘ (同一 inode 多次打开)
                    └──────────────┘
```

**为什么这样设计三级**？因为三种共享语义都要支持：
- `dup(fd)` / `dup2`：两个 fd 指向**同一个 struct file**——共享偏移量。
- `fork()`：子进程复制 fd 表，但 `fd[i]` 仍指向**父进程的同一个 struct file**（`f_count++`）——父子共享偏移量，这就是 shell 里 `(cmd1; cmd2) > out` 两条命令顺序追加同一文件的原理。
- 两次独立 `open()` 同一文件：两个**不同的 struct file**，各自独立偏移，但指向**同一个 inode**。

### 3.2 open 的内核流程

```c
// 用户态
int fd = open("/var/log/app.log", O_WRONLY | O_APPEND);
```

内核侧 `do_sys_openat2` 大致流程：

```
1. 分配一个 fd（在 fd 位图里找最小空闲下标）
2. path_openat：路径解析（走 dcache，见第二章）得到 dentry + inode
   - 若 O_CREAT 且不存在：调 inode->i_op->create 创建
   - 检查权限（capability / DAC / LSM 钩子）
3. 分配并初始化 struct file：
   - f_op = inode->i_fop（ext4 文件用 ext4_file_operations）
   - f_pos = 0（O_APPEND 时写入前会定位到末尾）
   - f_flags 记录 O_APPEND / O_DIRECT 等
4. 调 f_op->open（如果有）
5. fd_install：把 struct file 装进 fd 表 fd[fd] = file
6. 返回 fd
```

### 3.3 read / write 的内核流程

以 buffered（默认带缓存）`read` 为例：

```
read(fd, buf, 4096)
  → ksys_read → vfs_read
  → file->f_op->read_iter   // ext4: 走 generic_file_read_iter
  → 查 inode->i_mapping（page cache）
       命中 → 直接 copy_to_user，返回（零磁盘 IO）★
       未命中 → 触发 readahead，提交 bio 到块层（见 L10），
                等待 IO 完成（进程进入 D 状态），数据进 page cache，
                再 copy_to_user
  → 更新 f_pos += 4096
```

write（buffered）：

```
write(fd, buf, 4096)
  → vfs_write → file->f_op->write_iter → generic_perform_write
  → 找到/分配 page cache 中对应的 page（folio）
  → copy_from_user 把数据拷进 page
  → 标记 page 为 dirty（PG_dirty），挂到 inode 的 dirty 链
  → 返回（★ 数据还在内存！没落盘！）
  → 之后由 writeback 线程异步刷盘（见第六章）
```

记住这条铁律：**buffered write 返回只代表"数据进了 page cache 并标脏"，不代表落盘。** 落盘要么靠 writeback 的延迟回写，要么靠你显式 `fsync`。这正是引言里 `dd` 现象的根源。

### 3.4 硬链接 vs 符号链接

| | 硬链接 (hard link) | 符号链接 (symlink) |
|---|---|---|
| 本质 | 同一目录项指向同一 inode | 独立 inode，内容是一段路径字符串 |
| inode 号 | 与原文件相同 | 与原文件不同 |
| 跨文件系统 | 不能（inode 号是 fs 内局部的） | 可以 |
| 链接目录 | 不允许（防成环） | 可以 |
| 原文件删除后 | 仍可访问（nlink>0，inode 还在） | 悬空（dangling），访问报 ENOENT |
| 占用 | 几乎为零（只多一个目录项） | 一个 inode + 路径串 |

```bash
$ echo hello > a.txt
$ ln a.txt b.txt           # 硬链接
$ ln -s a.txt c.txt        # 符号链接
$ stat -c '%i %n' a.txt b.txt c.txt
131074 a.txt
131074 b.txt      # ← 与 a.txt 同一 inode
131099 c.txt      # ← 独立 inode

$ rm a.txt
$ cat b.txt        # hello   （硬链接仍可读，inode 还活着）
$ cat c.txt        # cat: c.txt: No such file or directory（符号链接悬空）
```

符号链接的解析发生在路径解析阶段：遇到 symlink 类型的 inode，读出它的目标路径，从那里继续 walk（受 `/proc/sys/fs/protected_symlinks` 与最大嵌套层数限制，防符号链接炸弹）。

---

## 第四章 ext4：inode、extent、journal 与延迟分配

### 4.1 磁盘布局：block group、inode 表、bitmap

ext4 把分区切成若干 **block group**，每组包含：

```
┌──────────────────────────────────────────────────────────┐
│ Group 0 │ Group 1 │ Group 2 │ ... （每组结构相同）          │
└──────────────────────────────────────────────────────────┘
单个 Group:
┌─────────┬──────────┬──────────┬──────────┬─────────────┐
│Superblock│ Group   │ Block    │ Inode    │ Inode 表 +  │
│(备份)    │ Desc    │ Bitmap   │ Bitmap   │ 数据块      │
└─────────┴──────────┴──────────┴──────────┴─────────────┘
```

- **block bitmap / inode bitmap**：每位表示对应块 / inode 是否已用。
- **inode 表**：固定数量的 inode 槽位，**在 mkfs 时就定死了**。这是 inode 耗尽（见陷阱清单）的根源——盘还有空间，但 inode 用光了，再也建不了新文件。
- 块大小通常 4096 字节（与 page size 对齐）。

```bash
$ dumpe2fs -h /dev/nvme0n1p2 2>/dev/null | grep -iE 'inode count|free inodes|block count|block size'
Inode count:              6553600
Free inodes:              6201033
Block count:              26214400
Block size:               4096

$ df -i /          # ← -i 看 inode 用量！
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/nvme0n1p2 6553600 352567 6201033    6% /
```

### 4.2 extent：取代间接块的连续区间映射

老 ext2/ext3 用**间接块**映射文件逻辑块到物理块：直接块 + 一级 / 二级 / 三级间接块。大文件要走多级间接，元数据开销大、随机访问慢。

ext4 改用 **extent（区段）**：一个 extent 描述"逻辑块 L 起，连续 N 个物理块从 P 起"。一个连续的大文件可能只需一个 extent 就描述完，元数据极小、顺序读写快。

```c
// fs/ext4/ext4_extents.h (简化)
struct ext4_extent {
    __le32  ee_block;     // 逻辑起始块
    __le16  ee_len;       // 块数（最多 32768）
    __le16  ee_start_hi;  // 物理起始块高 16 位
    __le32  ee_start_lo;  // 物理起始块低 32 位
};
```

extent 组织成 **extent tree**（B+ 树变体），inode 里内嵌根节点（`i_block` 复用为 `ext4_extent_header` + 4 个 extent）。文件碎片多时 extent 数量增长、树变深。

```bash
$ filefrag -v /data/big.dat
Filesystem type is: ext4
File size of /data/big.dat is 1073741824 (262144 blocks of 4096 bytes)
 ext:     logical_offset:        physical_offset: length:   flags:
   0:        0..  262143:    1048576.. 1310719: 262144:   eof
# ← 1 个 extent 描述了整个 1GB 文件，零碎片，理想状态

$ filefrag fragmented.dat
fragmented.dat: 4213 extents found    # ← 严重碎片，随机写造成
```

### 4.3 journal 日志：data=ordered / journal / writeback

掉电时，文件系统可能处于"元数据更新了一半"的不一致状态（比如新块已分配但 inode 大小没更新）。ext4 用 **journal（日志）** 保证崩溃一致性：先把要改的内容写进日志区，提交（commit）后再写回最终位置（checkpoint）。崩溃后重放日志即可恢复一致。

journal 由 **JBD2** 子系统实现。三种模式，通过 `mount -o data=` 选择：

| 模式 | 写日志的内容 | 一致性 | 性能 | 说明 |
|---|---|---|---|---|
| `data=journal` | 元数据 **+ 数据** 都先写日志 | 最强（数据也一致） | 最慢（数据写两遍） | 数据进日志再 checkpoint |
| `data=ordered`（默认） | 仅元数据进日志，但**保证数据先于元数据落盘** | 元数据一致，数据不会指向旧垃圾 | 居中 | 绝大多数场景的选择 |
| `data=writeback` | 仅元数据进日志，数据与元数据无序 | 元数据一致，但崩溃后文件可能读到旧/垃圾数据 | 最快 | 只在能容忍数据陈旧时用 |

```bash
$ tune2fs -l /dev/nvme0n1p2 | grep -i 'mount options\|journal'
Default mount options:    user_xattr acl
Journal inode:            8
$ mount | grep nvme0n1p2
/dev/nvme0n1p2 on / type ext4 (rw,relatime)   # 未显式写 data= 即 ordered
```

**为什么默认 ordered**：`data=writeback` 下，扩展一个文件后崩溃，可能 inode 大小已更新（元数据进了日志并恢复）但数据块还没写——于是文件"长大了"，但新区域读出来是磁盘上的旧垃圾（可能是别人删除文件的残留，有信息泄露风险）。`ordered` 通过"数据先落、元数据后落"杜绝了这点。

### 4.4 延迟分配（delayed allocation）

ext4 默认开启 **delalloc（延迟分配）**：`write` 时只在 page cache 里标脏、记下"欠多少块"，**并不立即分配物理块**，等到 writeback 真正刷盘时才一次性分配连续的物理块。

好处：
1. **减少碎片**——攒一批再分配，更容易拿到连续 extent。
2. **吸收覆盖写**——同一区域多次写、或写后很快删除（临时文件）的，可能根本不用落盘。

副作用 / 陷阱：
- `write` 成功不代表有空间——真正分配在 writeback 时才发生，**ENOSPC 可能延迟到 `fsync`/`close` 才报**，甚至直接丢在后台回写里。
- 著名的 **"ext4 0 字节文件"** 历史问题：程序 `open(O_TRUNC)` → `write` 新内容 → `close`（不 fsync），若此时崩溃，元数据（truncate）可能已落但新数据还没落，文件变 0 字节。正确做法是 `write 临时文件 → fsync → rename`（原子替换）。

```bash
$ mount | grep -o 'nodelalloc'   # 空则表示 delalloc 开启（默认）
# 关闭延迟分配（一般不建议）：mount -o remount,nodelalloc /
```

---

## 第五章 xfs / btrfs / overlayfs：选型与容器分层

### 5.1 ext4 vs xfs vs btrfs

| 维度 | ext4 | xfs | btrfs |
|---|---|---|---|
| 定位 | 通用、稳、默认 | 大文件 / 高并发 / 大容量 | 高级特性（快照/校验/池化） |
| 分配 | extent + delalloc | extent + 延迟分配 + 动态 inode | extent + CoW |
| inode | mkfs 时固定数量 | **动态分配**（不会"耗尽"） | 动态 |
| 并发写 | 单文件系统级别锁较多 | **allocation group 并行**，多核扩展性强 | CoW，写放大需注意 |
| 快照 | 无（靠 LVM） | 无（靠 LVM/reflink 部分） | **原生快照 / 子卷** |
| 数据校验 | 仅元数据（metadata_csum） | 元数据校验 | **数据 + 元数据校验** |
| 收缩 | 可（offline） | **不能收缩** | 可 |
| 典型用途 | 根分区、通用 | 数据库 / 大数据 / 大卷 | NAS、需要快照的场景 |

2026 年的主流共识：**根 / 通用用 ext4，大容量数据卷与数据库用 xfs，需要快照/校验/子卷的用 btrfs**。xfs 的"inode64 + allocation group 并行"在多核大盘高并发下尤其能打；它不能在线收缩是唯一明显短板。btrfs 的 CoW 带来写放大，数据库类随机覆盖写场景常用 `chattr +C` 关掉某目录的 CoW。

### 5.2 overlayfs：容器镜像分层的根基

overlayfs 是一个**联合挂载（union mount）**文件系统——把多个目录"叠"成一个视图。这正是 Docker / containerd 镜像分层的实现机制（呼应 cloud-native C01）。

```
              merged（容器看到的根 /）
                     ▲
        ┌────────────┴────────────┐
        │      overlayfs 联合      │
        ├──────────────────────────┤
upperdir│  可写层（容器运行时的改动） │  ← 写都落这里
lowerdir│  只读层 N（镜像顶层）      │
lowerdir│  只读层 ...               │  ← 镜像各层，多个只读 lower 叠加
lowerdir│  只读层 1（base 镜像）     │
        └──────────────────────────┘
workdir │  overlayfs 内部原子操作暂存│
```

挂载示例：

```bash
$ mount -t overlay overlay \
    -o lowerdir=/layers/base:/layers/app,upperdir=/layers/upper,workdir=/layers/work \
    /merged
```

关键语义：

- **读**：从上往下找，upper 优先，找不到逐层往下 lower。
- **写已存在文件**：触发 **copy-up**——把该文件从 lower **整份**复制到 upper，再在 upper 上改。这是 overlayfs 的写放大来源：改一个 1 GB 文件的 1 字节，会先复制整个 1 GB 到 upper。
- **删除 lower 里的文件**：在 upper 里建一个 **whiteout**（特殊的字符设备 0/0），表示"此名已删"，于是 merged 视图里看不到了，但 lower 原文件没动。
- **目录删除**：用 "opaque" 标记。

```bash
# 容器里删一个镜像自带文件，宿主上看 upper
$ ls -la /var/lib/docker/overlay2/<id>/diff/
c--------- 1 root root 0, 0 ... deleted_file   # ← whiteout，主次设备号 0,0
```

**生产含义**：容器里对镜像大文件做小修改会触发整文件 copy-up，造成意外的磁盘占用与延迟。把频繁写的数据放进 **volume（绕过 overlayfs，直挂宿主目录或独立卷）** 而不是容器可写层，是基本功。这也是为什么数据库容器必须用 volume 存数据，而不是写进容器层。

---

## 第六章 page cache 与持久化：脏页回写与 fsync 语义

### 6.1 page cache 与 folio

所有 buffered IO 都经过 **page cache**——内核用空闲内存缓存文件数据，按页（4K）或 **folio**（一组连续页，6.x 起内核内存管理的新基本单位）组织。每个 inode 有一个 `address_space`（`i_mapping`），其中一棵 **XArray** 把"文件页偏移"映射到内存中的 folio。

```
inode #12345
  └── address_space (i_mapping)
        └── XArray: index 0 → folio, index 1 → folio, ...
              脏 folio 标 PG_dirty，挂到 bdi 的 writeback 链
```

读命中省一次磁盘 IO；写先进 cache 标脏。`free` 里的 `buff/cache` 大部分就是 page cache，它**可被回收**（干净页直接丢、脏页先刷再丢），所以"内存几乎用满"在 Linux 上是正常健康状态（详见 L05）。

### 6.2 脏页回写：dirty_ratio 与 writeback 线程

脏页不会无限堆积，由 per-bdi（backing device info）的 writeback 内核线程负责刷盘。触发时机：

1. **周期性**：脏页存在超过 `dirty_expire_centisecs`（默认 30 秒）后被刷；writeback 线程每 `dirty_writeback_centisecs`（默认 5 秒）醒一次。
2. **比例阈值**：
   - 脏页超过 `vm.dirty_background_ratio`（默认约 10%）→ 后台开始异步刷，应用无感。
   - 脏页超过 `vm.dirty_ratio`（默认约 20%）→ **写入的进程被同步阻塞**（throttle），帮忙刷盘直到降下来。这是"写一会儿突然卡住"的常见原因。
3. **显式**：`fsync` / `sync` / `msync`。

```bash
$ sysctl vm.dirty_ratio vm.dirty_background_ratio \
         vm.dirty_expire_centisecs vm.dirty_writeback_centisecs
vm.dirty_ratio = 20
vm.dirty_background_ratio = 10
vm.dirty_expire_centisecs = 3000
vm.dirty_writeback_centisecs = 500

# 实时看脏页量
$ grep -E 'Dirty|Writeback' /proc/meminfo
Dirty:            204800 kB
Writeback:         12000 kB
```

**大内存机器的陷阱**：256 GB 内存机器，`dirty_ratio=20%` 意味着可堆积约 50 GB 脏页。一旦触顶，所有写进程被 throttle，IO 突发卡死。生产数据库 / 存储节点常把这两个阈值调小（或改用绝对字节版 `dirty_bytes` / `dirty_background_bytes`）来平滑回写、避免脉冲式卡顿。

### 6.3 fsync / fdatasync / sync 的区别

| 调用 | 刷什么 | 不刷什么 | 用途 |
|---|---|---|---|
| `fsync(fd)` | 该文件的数据 + 所有元数据（含 mtime/size 等） | 其他文件 | 严格持久化（含元数据） |
| `fdatasync(fd)` | 该文件数据 + **影响后续读取所必需的元数据**（如文件变大时的 size） | 不必要的元数据（如纯 mtime 变化） | 数据库 WAL 首选（更快） |
| `sync()` | **所有**文件系统的所有脏页 | — | 系统级，粗粒度 |
| `syncfs(fd)` | fd 所在的**单个文件系统**全部脏页 | 其他 fs | 比 sync 精准 |

`fdatasync` 比 `fsync` 快的原因：很多写只改了数据和 mtime/ctime，`fdatasync` 可以跳过仅为时间戳变化而产生的元数据日志提交（一次额外的 journal commit），减少 IO 与延迟。数据库 WAL（如 PostgreSQL、Redis AOF、MySQL redo log）几乎都用 `fdatasync`。

### 6.4 fsync 的"放大"与 journal 的纠缠

`fsync` 的真实成本常被低估。一次 `fsync` 在 ext4 `data=ordered` 下可能触发：

```
fsync(fd)
  1. 把该文件的脏 page 提交为 bio，写到磁盘数据区
  2. 等待数据 IO 完成
  3. 提交 journal（JBD2 commit）：元数据日志落盘
  4. 发 FLUSH/FUA：让磁盘把自己的易失写缓存也刷到盘片（屏障）
```

**fsync 放大**：你只想持久化一个文件，但 ext4 的 journal 是**全文件系统共享**的——一次 commit 可能连带把别的进程攒在 journal 里的元数据一起刷。于是高并发下 A 进程的 `fsync` 会被 B 进程的写拖慢，反之亦然。这是"明明我的写很小，fsync 却很慢"的经典原因。

观测 fsync 延迟：

```bash
# bcc / bpftrace 看每次 fsync 耗时分布
$ /usr/share/bcc/tools/ext4dist 10        # ext4 各操作延迟直方图
$ bpftrace -e 'kprobe:vfs_fsync { @start[tid] = nsecs; }
   kretprobe:vfs_fsync /@start[tid]/ {
     @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'
```

### 6.5 O_DIRECT 与 O_SYNC

- **`O_DIRECT`**：绕过 page cache，DMA 直接在用户缓冲区与磁盘之间传输。要求缓冲区地址、偏移、长度按块（通常 512 或 4096）对齐，否则 `EINVAL`。**它不保证持久化**（数据可能还在磁盘的易失缓存里），要持久化仍需 `fsync` 或配合 `O_DSYNC`。用途：数据库自己管缓存（如 Oracle、MySQL InnoDB `O_DIRECT` flush method），避免内核 page cache 双重缓存与回写不可控。
- **`O_SYNC`**：每次 `write` 都在返回前把数据**和元数据**落盘（等价于每写一次 `fsync`）。
- **`O_DSYNC`**：每次 `write` 返回前落数据 + 必要元数据（等价每写一次 `fdatasync`）。

```c
// O_DIRECT 必须对齐，否则 EINVAL
void *buf;
posix_memalign(&buf, 4096, 4096);     // 4K 对齐缓冲区
int fd = open("data", O_WRONLY | O_DIRECT);
pwrite(fd, buf, 4096, 0);             // 偏移与长度均须 4K 对齐
```

数据库 WAL 的两种主流策略：
1. **page cache + fdatasync**：写进 cache，提交时 `fdatasync`。简单，依赖内核回写。
2. **O_DIRECT + O_DSYNC**（或 O_DIRECT + 显式 fsync）：自己管缓冲，绕开内核 cache，回写时机完全可控。

到此，"一次 write 到盘片"的全路径就闭环了：用户缓冲区 → `write` 系统调用 → page cache 标脏 → writeback/fsync → bio → 块层 blk-mq（见 L10）→ 设备驱动 → 磁盘易失缓存 → FLUSH/FUA → 盘片。

---

## 生产实践

1. **原子替换文件配置**：更新配置文件用"写临时文件 → `fsync(临时文件)` → `fsync(目录)` → `rename`"，避免崩溃后读到半截文件。`rename` 本身原子，但要 `fsync` 目录才能保证 rename 这个目录项变更落盘。

2. **数据库类负载调小 dirty 阈值**：大内存机器把 `vm.dirty_background_bytes` / `vm.dirty_bytes` 设为绝对值（如 256MB / 1GB），用绝对字节而非百分比，避免几十 GB 脏页脉冲式回写造成的周期性卡顿。

3. **容器数据走 volume，别写容器层**：overlayfs 的 copy-up 对大文件小改是灾难。数据库、消息队列的数据目录一律挂 volume 或独立卷，绕过 overlayfs（呼应 cloud-native C01、C06）。

4. **监控 inode 用量**：`df -i` 与 `df` 同等重要。小文件海量的场景（邮件、缓存、session）建文件系统时用 `mkfs.ext4 -N <数量>` 或 `-i <bytes-per-inode>` 调高 inode 密度，或直接选 xfs（动态 inode）。

5. **fsync 延迟纳入 SLO**：用 `ext4dist` / `fsyncstall`（bcc）或 bpftrace 持续采集 fsync p99，它直接决定数据库提交延迟。NVMe 上 fsync 通常亚毫秒，若飙到几十毫秒，查盘的写缓存策略、journal 竞争、或是否误用了机械盘。

6. **filefrag 巡检碎片**：长期随机写的文件（数据库、虚拟磁盘镜像）extent 数会膨胀。`filefrag` 数千 extent 时，顺序读会退化为随机读。可用 `e4defrag`（ext4）或重写文件来整理。

---

## 陷阱清单

1. **inode 耗尽，盘还空着却建不了文件**
   - 现象：`df` 显示有空间，写文件却报 `No space left on device`；`df -i` 显示 IUse% 100%。
   - 原因：ext4 的 inode 数量在 mkfs 时定死，海量小文件把 inode 用光了，与数据块空间无关。
   - 修法：删无用小文件释放 inode；或备份后用 `mkfs.ext4 -N <更大inode数>` 重建；新建文件系统直接选 xfs（动态 inode，不会耗尽）。

2. **删了大日志文件磁盘空间不释放**
   - 现象：`rm huge.log` 后 `df` 空间没回来，重启才恢复。
   - 原因：有进程仍 open 着该文件（`i_nlink=0` 但 `f_count>0`），inode 与数据块不会回收。
   - 修法：`lsof | grep deleted` 找到持有者，重启或让进程 reopen；应急可 `: > /proc/<pid>/fd/<n>` 截断（truncate）该 fd 释放空间。

3. **write 成功但数据丢失（崩溃 / 断电）**
   - 现象：程序 write 返回成功，掉电后文件内容丢失或变 0 字节。
   - 原因：buffered write 只进 page cache，未落盘；ext4 delalloc 下崩溃时元数据/数据落盘不同步。
   - 修法：关键数据 `fsync`/`fdatasync` 后再认为持久化；配置文件用"临时文件 + fsync + rename"原子替换模式。

4. **fsync 异常慢，且与"我无关"的进程相关**
   - 现象：自己只写了几 KB，fsync 却要几十毫秒，时快时慢。
   - 原因：ext4 journal 全文件系统共享，一次 JBD2 commit 把别的进程的脏元数据一起刷；或磁盘写缓存 FLUSH 慢。
   - 修法：高并发写 fsync 的服务彼此隔离到不同文件系统/盘；用 `data=ordered`（默认）而非 `journal`；确认是 SSD/NVMe；必要时数据库自管 O_DIRECT。

5. **negative dentry / slab 占用暴涨**
   - 现象：`free` 里 cache 巨大、`slabtop` dentry 数千万、`SReclaimable` 几个 GB。
   - 原因：程序海量探测不存在的路径（动态库搜索、import、缓存目录扫描）制造大量 negative dentry。
   - 修法：通常会被内存压力自动回收，无需干预；排障时 `echo 2 > /proc/sys/vm/drop_caches`（生产慎用，会丢热缓存致后续变慢）；根治是减少无谓的路径探测。

6. **大内存机周期性写卡顿**
   - 现象：写入吞吐周期性骤降、应用 p99 周期性毛刺，`/proc/meminfo` 的 Dirty 大幅波动。
   - 原因：脏页攒到 `dirty_ratio`（默认 20%）触发同步 throttle，写进程被拉去刷盘。
   - 修法：调小 `vm.dirty_bytes`/`vm.dirty_background_bytes` 用绝对值平滑回写；检查是否单盘扛不住写入峰值。

7. **O_DIRECT 写报 EINVAL**
   - 现象：加了 `O_DIRECT` 后 `pwrite` 返回 `EINVAL`。
   - 原因：缓冲区地址、文件偏移或长度未按块对齐（通常需 512 或 4096 对齐）。
   - 修法：用 `posix_memalign` 分配对齐缓冲区，偏移和长度都取块大小整数倍；用 `statx`/`ioctl` 查设备的逻辑块大小。

8. **容器磁盘占用莫名暴涨**
   - 现象：容器跑久了占用远超镜像大小，删容器后宿主空间回收。
   - 原因：容器在可写层（upperdir）反复 copy-up 大文件、写日志、临时文件，overlayfs 写放大。
   - 修法：数据/日志走 volume；`docker system df -v` 排查；只读文件系统 + tmpfs 临时目录。

---

## 2026 现状

- **folio 化基本完成**：page cache 与 IO 路径以 folio（连续多页）为基本单位，减少 per-page 开销、改善大 IO 与大页协同；ext4/xfs/btrfs 的回写路径均已 folio 化。
- **ext4 / xfs 仍是主流二选一**：ext4 做通用与根分区，xfs 做大容量数据卷与高并发数据库；btrfs 在需要快照 / 校验 / 子卷的 NAS、桌面和部分发行版默认根（如某些 openSUSE / Fedora 变体）站稳。
- **bcache / dm-cache / 本地 NVMe 直挂**普及，page cache 不再是唯一缓存层；NVMe 让 fsync 进入亚毫秒，数据库提交延迟瓶颈更多转到 journal 竞争与软件路径。
- **overlayfs 是容器分层事实标准**：containerd / Docker 默认 overlay2 snapshotter；针对 copy-up 的优化（如 metacopy、volatile mount、composefs/erofs 只读镜像）在镜像分发与启动加速场景推进。
- **O_DIRECT + io_uring** 成为高性能存储引擎（数据库、对象存储）的首选组合：绕开 page cache 自管缓冲，用 io_uring 批量异步提交（见 L09），把 CPU 从同步 IO 等待中解放。
- **fs-verity / 数据校验**在安全敏感场景增多；ext4/xfs 的 metadata_csum 默认开启，btrfs 提供端到端数据校验。

---

## 练习题

1. 用一句话解释：为什么 `stat` 同一个文件，inode 号相同的两个路径一定是硬链接而非符号链接？符号链接的 inode 号与目标相同吗？

2. 画出 `fork()` 后父子进程的 fd 表、struct file、inode 三级关系图，并解释为什么 `(echo a; echo b) > out` 会得到 `a\nb` 而不是相互覆盖。

3. ext4 默认 `data=ordered`，请描述"扩展一个文件后立即断电"在 `ordered` 与 `writeback` 两种模式下分别可能读到什么，以及为什么 `ordered` 不会读到磁盘旧垃圾。

4. `fsync` 与 `fdatasync` 的区别是什么？为什么数据库 WAL 多用 `fdatasync`？给出一种你会**必须**用 `fsync` 而不能用 `fdatasync` 的场景。

5. 解释 overlayfs 的 copy-up 与 whiteout 机制。为什么"容器里修改镜像自带的一个大文件的 1 字节"会导致磁盘占用增加约该文件大小？

6. `vm.dirty_ratio` 与 `vm.dirty_background_ratio` 各触发什么行为？在 256 GB 内存机上保持默认 20%/10% 可能引发什么生产问题，你会怎么调？

7. **实战**：给定现象——业务 `df` 显示根分区还有 40 GB 空闲，但应用频繁报 `No space left on device`，新建任何小文件都失败。写出你的排查命令序列和最可能的根因。

8. **排障**：一个 PostgreSQL 实例 commit 延迟 p99 从 1 ms 突然涨到 30 ms，CPU、内存、网络都正常。请设计一套用 `iostat -x`、`bpftrace`/`ext4dist`、`mount` 选项检查的排查流程，列出至少三个候选根因（提示：fsync 放大、journal 竞争、磁盘写缓存策略、是否误挂了机械盘）。

---

## 参考答案

1. inode 号是文件系统内文件本体的唯一标识，硬链接是两个目录项指向同一个 inode，所以 inode 号相同必为硬链接（或就是同一文件名）。符号链接是独立 inode，其内容是一段目标路径字符串，inode 号与目标文件**不同**——`stat` 符号链接（不加 `-L`/对目标 stat）会显示它自己的 inode 号，与目标不同。

2. 三级关系：fork 后子进程复制了 fd 表（fd 数组），但每个 `fd[i]` 仍指向父进程的同一个 `struct file`（`f_count++`），该 file 再指向同一个 inode。即父子的 fd 表是各自的，但底层共享同一个 struct file（含同一个 `f_pos` 偏移）和同一个 inode。`(echo a; echo b) > out` 中两条命令（子进程）继承同一个 struct file，共享偏移量：echo a 写后偏移前进到 2，echo b 从偏移 2 继续写，于是得到 `a\nb` 顺序追加而非相互从 0 覆盖。

3. `ordered`（默认）：仅元数据进 journal，但保证数据先于元数据落盘。扩展文件后断电，要么元数据未提交（文件没变大）、要么数据已先落再提交元数据，绝不会出现"文件变大但新区域是旧垃圾"，最多读到"文件没变大"或正确的新数据。`writeback`：仅元数据进 journal，数据与元数据无序。扩展文件后断电，可能 inode 的 size（元数据）已经过日志恢复变大，但对应数据块还没落盘——于是新增区域读出磁盘上的旧内容/垃圾（可能是别人删除文件的残留，有信息泄露风险）。`ordered` 通过强制"数据先落、元数据后落"杜绝了这种垃圾暴露。

4. `fsync` 刷该文件的数据 + 全部元数据（含 mtime/ctime 等）；`fdatasync` 刷数据 + 仅影响后续读取所必需的元数据（如文件变大时的 size），跳过纯时间戳变化的元数据。数据库 WAL 多用 `fdatasync` 是因为它能省掉仅为 mtime/ctime 变化而触发的额外 journal commit，减少一次 IO 和延迟，而 WAL 顺序追加只需保证数据和 size 落盘即可。必须用 `fsync` 而非 `fdatasync` 的场景：当文件的元数据本身就是你要持久化的关键信息时——例如刚 `chmod`/`chown` 改了权限、或依赖精确 mtime 做一致性/同步判断（如备份、rsync 增量、构建系统按 mtime 判断），必须 `fsync` 保证这些元数据落盘。

5. copy-up：overlayfs 写一个只存在于 lower 只读层的文件时，会把该文件**整份**从 lower 复制到 upper 可写层，再在 upper 上修改。whiteout：删除 lower 里的文件时，在 upper 建一个特殊的字符设备（主次设备号 0/0）标记"此名已删"，使 merged 视图看不到它，但 lower 原文件不动。"改镜像大文件 1 字节导致占用增加约该文件大小"的原因：写入触发 copy-up，把整个大文件复制到 upper 层再改那一字节，于是 upper 多出一份完整副本，磁盘占用增加约等于文件大小（这就是 overlayfs 的写放大，也是数据要走 volume 而非容器层的原因）。

6. `dirty_background_ratio`（默认约 10%）：脏页占比超过它时后台 writeback 线程开始**异步**刷盘，应用无感。`dirty_ratio`（默认约 20%）：脏页超过它时**写入进程被同步阻塞**（throttle）、被拉去帮忙刷盘直到降下来。256 GB 机上 20%/10% 意味着可堆积约 50 GB 脏页才触发同步限流，一旦写入突发触顶，所有写进程被集体 throttle、IO 脉冲式卡死，造成 p99 周期性毛刺。调法：改用绝对字节版 `vm.dirty_bytes`/`vm.dirty_background_bytes`（如 1GB/256MB），让回写尽早、平滑启动、上限受控，避免几十 GB 脏页决堤。

7. 最可能根因：inode 耗尽（海量小文件用光了 mkfs 时固定的 inode 数）。排查命令序列：(1) `df -i /` 看 IUse%，若为 100% 即 inode 耗尽（与数据空间无关）；(2) `df -h /` 确认数据空间确实还有（40 GB 空闲）；(3) 定位哪个目录小文件多——`for d in /*; do echo "$(find $d -xdev | wc -l) $d"; done | sort -rn`（或用 `du --inodes`）找 inode 大户。次要可能：被进程占用的已删除文件（`lsof | grep deleted`）。根因确认为 inode 耗尽后，修法：清理无用小文件释放 inode，或备份后 `mkfs.ext4 -N` 提高 inode 数重建，或换 xfs（动态 inode）。

8. 排查流程：(1) `iostat -x 1` 看该盘的 `w_await`/`await`（单次 IO 平均等待）、`%util`、`aqu-sz`——若 await 飙到几十毫秒且 %util 接近 100%，是磁盘层瓶颈（盘慢或排队）；(2) `bpftrace`/`ext4dist 1` 看 fsync/写操作的延迟直方图，确认是不是 fsync 本身变慢；(3) `mount | grep <pg数据盘>` 与 `tune2fs -l` 看 `data=` 模式、是否 SSD/NVMe、写缓存策略。至少三个候选根因：(a) fsync 放大 / journal 竞争——ext4 journal 全文件系统共享，同盘上别的高频写进程（或 PG 自身多后端）的元数据让 JBD2 commit 变慢，把你的 fsync 拖慢；(b) 磁盘易失写缓存策略变化——磁盘 write cache 被关闭或 FLUSH/FUA 屏障变慢（如固件/电池保护单元 BBU 失效后阵列改 write-through），fsync 必须等盘片，延迟陡增；(c) 误挂了机械盘或盘性能退化——数据/WAL 落在 HDD 或 SSD 磨损/掉速，单次 fsync 从亚毫秒变几十毫秒。缓解按根因：把 WAL 与数据隔离到独立盘/文件系统降低 journal 竞争、确认用 NVMe/SSD、检查 RAID 卡缓存与 BBU 状态、考虑 PG 自管 O_DIRECT。
