# 精通 Go Socket 与 WebSocket：从字节流到实时长连接

> 课程编号：G31
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Networking / Realtime 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：60 分钟

---

## 引言：`net/http` 之上还有什么

G27 讲完 `net/http`，你能写出生产级的请求-响应服务。但下面这些需求它一个都答不了：

- 服务端要**主动推**消息给客户端（聊天、行情、通知）
- 一条连接上要跑**双向、乱序**的消息流
- 自定义二进制协议，不想背 HTTP header 的开销

这时候要往下走一层：**socket**。而 WebSocket 是这层之上被标准化、能穿浏览器和代理的那个协议。

很多人把两者当成并列的技术学，这是最大的误区：

```
应用层   HTTP / WebSocket / 自定义协议  ← 协议，定义消息长什么样
─────────────────────────────────────
传输层   TCP / UDP                      ← 内核实现
─────────────────────────────────────
编程接口 socket API                     ← 你调的那组函数
```

**Socket 是 API，WebSocket 是协议。** WebSocket 自己也是用 socket 实现的。跳过 socket 直接学 WebSocket，遇到粘包、半关闭、背压会完全懵，因为那些问题全在下面这层。

本章的路线：socket API → TCP 字节流的真相 → WebSocket 协议 → Go 生产实践。

---

## 第一章：Socket —— 你其实一直在用

### 1.1 系统调用与 Go 的封装

C 里写一个 TCP server 是这样：

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd, ...);
listen(fd, 128);
int conn = accept(fd, ...);
read(conn, buf, sizeof(buf));
write(conn, buf, n);
close(conn);
```

Go 把这一整套压成三个东西：

```go
ln, err := net.Listen("tcp", ":8080")   // socket + bind + listen
conn, err := ln.Accept()                 // accept
n, err := conn.Read(buf)                 // read
```

`net.Conn` 就是 `io.Reader` + `io.Writer` + 地址和 deadline。这意味着**所有 `io` 包的工具对它都有效**——`io.Copy`、`bufio.Scanner`、`json.NewDecoder`，不用学新 API。

### 1.2 Go 为什么不用你管 epoll

C 里要处理 1 万连接，得手写 epoll 事件循环，代码立刻变成状态机。Go 里是：

```go
for {
    conn, err := ln.Accept()
    if err != nil { return err }
    go handle(conn)      // 一个连接一个 goroutine
}
```

这段"每连接一 goroutine"的代码之所以不会炸，是因为 runtime 内部有个 **netpoller**：`conn.Read` 阻塞时，runtime 把这个 goroutine 挂起，把 fd 注册进 epoll（Linux）/ kqueue（BSD），然后把 M 让给别的 goroutine 跑。数据到了 netpoller 再唤醒它。

所以你写的是**同步阻塞代码**，跑的是**异步事件驱动**。这是 Go 做网络服务最大的优势——详见 G11（GMP 调度）。

代价：每个 goroutine 初始栈 2KB，10 万连接约 200MB 起步（实际因栈增长会更多）。百万连接场景才需要考虑 `gobwas/ws` + 手动 epoll 那套，见第六章。

### 1.3 一个能跑的 echo server

```go
package main

import (
    "io"
    "log"
    "net"
    "time"
)

func main() {
    ln, err := net.Listen("tcp", ":9000")
    if err != nil {
        log.Fatal(err)
    }
    defer ln.Close()
    log.Println("listening on :9000")

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Println("accept:", err)
            continue
        }
        go func(c net.Conn) {
            defer c.Close()
            c.SetDeadline(time.Now().Add(5 * time.Minute))
            io.Copy(c, c)   // 读什么写什么
        }(conn)
    }
}
```

用 `nc localhost 9000` 就能测。**注意 `defer c.Close()` 和 deadline**——漏掉任何一个，线上都会攒出泄漏的连接。

### 1.4 必须知道的几个 socket 选项

```go
tcpConn := conn.(*net.TCPConn)

tcpConn.SetNoDelay(true)     // 禁用 Nagle 算法（Go 默认就是 true）
tcpConn.SetKeepAlive(true)   // TCP 层保活
tcpConn.SetLinger(0)         // Close 时直接 RST，不进 TIME_WAIT（慎用）
```

- **Nagle 算法**：把小包攒起来一起发，省带宽但增延迟。Go 默认关掉它（`NoDelay=true`），对交互式协议是对的。**别自作聪明打开。**
- **TCP KeepAlive**：Go 1.23+ 用 `net.KeepAliveConfig` 可以精细配置探测间隔。但注意它的默认探测周期是**分钟级**，检测死连接太慢，业务层还是要自己做心跳（第七章）。
- **`SetLinger(0)`** 会跳过四次挥手直接 RST，可能丢掉发送缓冲区里的数据。除非你确定在解决 TIME_WAIT 耗尽，否则别碰。

---

## 第二章：TCP 是字节流 —— 第一个必踩的坑

### 2.1 粘包与半包

这是所有网络编程新手的第一课，也是面试必问。看这段代码：

```go
// 客户端
conn.Write([]byte("hello"))
conn.Write([]byte("world"))

// 服务端
buf := make([]byte, 1024)
n, _ := conn.Read(buf)
fmt.Println(string(buf[:n]))
```

服务端可能打印：

- `hello` 然后 `world`（你以为的）
- `helloworld`（**粘包**：两次 Write 被合并）
- `hel`（**半包**：一次 Write 被拆开）

三种都是**正确行为**。因为：

> **TCP 提供的是可靠、有序的字节流，不是消息流。它不保证消息边界。**

`Write` 只是把数据塞进内核发送缓冲区，内核按 MSS、拥塞窗口、Nagle 状态自行决定怎么切分成 TCP 段。UDP 才是保留边界的数据报（但不可靠）。

**记住这条规则：`conn.Read(buf)` 返回的 `n` 只告诉你"读到了 n 个字节"，绝不代表"这是一条完整消息"。**

### 2.2 三种消息边界方案

**方案 A：长度前缀（最常用）**

```go
// 协议：4 字节大端长度 + payload
func writeMsg(w io.Writer, payload []byte) error {
    var hdr [4]byte
    binary.BigEndian.PutUint32(hdr[:], uint32(len(payload)))
    if _, err := w.Write(hdr[:]); err != nil {
        return err
    }
    _, err := w.Write(payload)
    return err
}

func readMsg(r io.Reader, maxSize uint32) ([]byte, error) {
    var hdr [4]byte
    if _, err := io.ReadFull(r, hdr[:]); err != nil {   // 关键：ReadFull
        return nil, err
    }
    n := binary.BigEndian.Uint32(hdr[:])
    if n > maxSize {
        return nil, fmt.Errorf("message too large: %d > %d", n, maxSize)   // 关键：上限
    }
    payload := make([]byte, n)
    if _, err := io.ReadFull(r, payload); err != nil {
        return nil, err
    }
    return payload, nil
}
```

两个细节决定了这段代码是生产级还是玩具级：

1. **`io.ReadFull` 而不是 `Read`**——`Read` 可能只返回 2 个字节，头就读残了。
2. **`maxSize` 上限检查**——没有它，对方发一个 `0xFFFFFFFF` 长度头，你的服务立刻尝试 `make([]byte, 4GB)` 然后 OOM。这是最典型的**远程 DoS**，必须挡在协议解析入口。

**方案 B：分隔符**

行协议（Redis RESP、SMTP、HTTP header）用 `\n` 分隔：

```go
scanner := bufio.NewScanner(conn)
scanner.Buffer(make([]byte, 0, 4096), 1024*1024)   // 同样要设上限
for scanner.Scan() {
    handle(scanner.Text())
}
```

`bufio.Scanner` 默认单行上限 64KB，超了会静默返回 `bufio.ErrTooLong`。**一定要检查 `scanner.Err()`**，否则长消息导致的断连你根本看不到。

**方案 C：自描述格式**

Protobuf 的 varint 前缀、JSON 流（`json.Decoder` 能从流里连续解出多个对象）。

```go
dec := json.NewDecoder(conn)
for {
    var msg Message
    if err := dec.Decode(&msg); err != nil { break }   // 自动处理边界
    handle(msg)
}
```

`json.Decoder` 内部有缓冲，能正确处理跨 TCP 段的 JSON 对象。小流量场景这是最省事的一档。

### 2.3 Write 也可能"半写"

```go
n, err := conn.Write(bigBuf)   // n 可能 < len(bigBuf)？
```

对 `net.Conn`，Go 的实现保证要么全写完要么返回 error（内部循环了），所以这里比较安全。但如果你的 `w` 是自己包的 `io.Writer`，就必须遵守 `io.Writer` 契约自己循环——`io.Writer` 接口本身**不保证**写完。

---

## 第三章：从 TCP 到 WebSocket —— 为什么需要它

### 3.1 浏览器的限制

服务端要推消息给浏览器，历史上的方案：

| 方案 | 原理 | 问题 |
|---|---|---|
| 轮询（polling） | 定时 `GET /messages` | 延迟高、大量空请求 |
| 长轮询（long polling） | 服务端 hold 住请求直到有数据 | 每条消息一次完整 HTTP 往返 |
| SSE（`text/event-stream`） | 一条 HTTP 响应持续写 | **只能服务端→客户端单向** |
| WebSocket | HTTP 握手后升级为全双工 | 有状态，扩展成本高 |

**先问一句：你真的需要 WebSocket 吗？**

如果只需要服务端单向推（通知、进度条、LLM 流式输出），**SSE 更简单**：走标准 HTTP、自动重连、代理友好、不用管心跳。见 [A12 — 精通流式输出与 SSE](../ai-backend/A12-精通-流式输出与-SSE.md)。

只有**双向、低延迟、高频**（聊天、协同编辑、游戏、行情）才值得上 WebSocket 的复杂度。

### 3.2 握手：一次伪装成 HTTP 的升级

客户端发一个普通 HTTP 请求，但带几个特殊 header：

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Origin: https://example.com
```

服务端同意就回 **101**：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

`Sec-WebSocket-Accept` 的算法是写死的：

```go
const magicGUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

func acceptKey(clientKey string) string {
    h := sha1.New()
    io.WriteString(h, clientKey+magicGUID)
    return base64.StdEncoding.EncodeToString(h.Sum(nil))
}
// acceptKey("dGhlIHNhbXBsZSBub25jZQ==") == "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

这个魔数字符串来自 RFC 6455，作用**不是安全**（它是公开的），而是**防止缓存投毒**——确保对端真的懂 WebSocket 协议，而不是一个把请求当普通 HTTP 转发的旧代理。

101 之后，这条 TCP 连接上**不再有任何 HTTP**，双方直接按 WebSocket 帧格式收发。

### 3.3 一个必须知道的安全问题：CSWSH

> **WebSocket 握手不受同源策略（SOP）限制，浏览器也不会对它做 CORS 预检。**

意思是：`evil.com` 的页面可以直接 `new WebSocket("wss://yourbank.com/ws")`，而且**浏览器会自动带上 yourbank.com 的 Cookie**。如果你的服务端只靠 Cookie 认证，攻击者就拿到了一条已认证的双向通道——这叫 **CSWSH**（Cross-Site WebSocket Hijacking）。

**服务端必须自己校验 `Origin` header。** 这不是可选项：

```go
// coder/websocket
c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
    OriginPatterns: []string{"example.com", "*.example.com"},
})

// gorilla/websocket
upgrader := websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool {
        return slices.Contains(allowedOrigins, r.Header.Get("Origin"))
    },
}
```

两个库的默认行为都是"只允许同源"，**别为了本地联调把它改成 `return true` 然后忘了改回来**——这是真实泄漏事故的常见成因。

---

## 第四章：帧格式 —— 亲手解一次就通了

### 4.1 结构

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
|                     Payload Data continued ...                |
+---------------------------------------------------------------+
```

关键字段：

| 字段 | 含义 |
|---|---|
| `FIN` (1 bit) | 是否是消息的最后一帧（分片用） |
| `opcode` (4 bit) | `0x0` 续帧 / `0x1` 文本 / `0x2` 二进制 / `0x8` close / `0x9` ping / `0xA` pong |
| `MASK` (1 bit) | payload 是否被掩码 |
| `Payload len` (7 bit) | ≤125 直接用；`126` → 后跟 2 字节；`127` → 后跟 8 字节 |
| `Masking-key` (4 byte) | MASK=1 时存在 |

三条硬规则：

1. **客户端→服务端的帧必须 mask，服务端→客户端的帧必须不 mask。** 违反就得断开连接。
2. **控制帧（0x8/0x9/0xA）payload ≤ 125 字节，且不能分片**（FIN 必须为 1）。
3. 长度用**大端**，`127` 情况下最高位必须为 0（即实际上限 2^63-1）。

### 4.2 掩码：为什么客户端要做这件事

掩码算法极其简单——就是循环异或：

```go
func mask(key [4]byte, payload []byte) {
    for i := range payload {
        payload[i] ^= key[i%4]
    }
}
```

异或两次还原，所以加解掩码是同一个函数。

**它不提供任何加密价值**（key 就明文躺在帧头里）。它存在的唯一理由是**防代理缓存投毒**：早年有些中间代理会把 WebSocket 流当 HTTP 解析，攻击者可以精心构造 payload 让它看起来像一个 HTTP 响应，从而毒化代理缓存。每帧随机掩码让攻击者无法控制线上的字节，这个攻击就废了。

所以：**这是给中间设备看的，不是给你看的。** 但你写解析器时必须实现它。

### 4.3 手写一个帧解析器

这段代码是本章的核心。跑通它，WebSocket 对你就没有黑盒了：

```go
type Frame struct {
    Fin     bool
    Opcode  byte
    Payload []byte
}

const maxFrameSize = 1 << 20   // 1MB 上限，防 OOM

func ReadFrame(r io.Reader) (*Frame, error) {
    var hdr [2]byte
    if _, err := io.ReadFull(r, hdr[:]); err != nil {
        return nil, err
    }

    f := &Frame{
        Fin:    hdr[0]&0x80 != 0,
        Opcode: hdr[0] & 0x0F,
    }
    masked := hdr[1]&0x80 != 0
    length := uint64(hdr[1] & 0x7F)

    // 扩展长度
    switch length {
    case 126:
        var ext [2]byte
        if _, err := io.ReadFull(r, ext[:]); err != nil {
            return nil, err
        }
        length = uint64(binary.BigEndian.Uint16(ext[:]))
    case 127:
        var ext [8]byte
        if _, err := io.ReadFull(r, ext[:]); err != nil {
            return nil, err
        }
        length = binary.BigEndian.Uint64(ext[:])
    }

    if length > maxFrameSize {
        return nil, fmt.Errorf("frame too large: %d", length)
    }
    // 控制帧的两条硬约束
    if f.Opcode >= 0x8 && (length > 125 || !f.Fin) {
        return nil, errors.New("invalid control frame")
    }

    var key [4]byte
    if masked {
        if _, err := io.ReadFull(r, key[:]); err != nil {
            return nil, err
        }
    }

    f.Payload = make([]byte, length)
    if _, err := io.ReadFull(r, f.Payload); err != nil {
        return nil, err
    }
    if masked {
        for i := range f.Payload {
            f.Payload[i] ^= key[i%4]
        }
    }
    return f, nil
}
```

用 RFC 6455 §5.7 的官方样例验证——掩码后的 `"Hello"` 文本帧：

```
81 85 37 fa 21 3d 7f 9f 4d 51 58
│  │  └── mask key ──┘ └─ payload ─┘
│  └── MASK=1, len=5
└── FIN=1, opcode=0x1 (text)

0x48('H') ^ 0x37 = 0x7f  ✓
0x65('e') ^ 0xfa = 0x9f  ✓
0x6c('l') ^ 0x21 = 0x4d  ✓
0x6c('l') ^ 0x3d = 0x51  ✓
0x6f('o') ^ 0x37 = 0x58  ✓
```

写个测试喂这 11 个字节进去，能解出 `"Hello"` 就对了。

### 4.4 分片与关闭

**分片**：一条大消息可以拆成 `[opcode=0x1, FIN=0]` + `[opcode=0x0, FIN=0]` + `[opcode=0x0, FIN=1]`。中间**允许插入控制帧**（比如 ping），所以解析器不能假设续帧是连续的。

**关闭握手**：一方发 `0x8` close 帧，另一方回一个 close 帧，然后才关 TCP。

```
[2 字节大端状态码][UTF-8 原因(可选)]
```

常用状态码：

| 码 | 含义 |
|---|---|
| 1000 | 正常关闭 |
| 1001 | 端点离开（页面关闭、服务重启） |
| 1008 | 策略违规（鉴权失败常用这个） |
| 1009 | 消息过大 |
| 1011 | 服务端内部错误 |
| **1006** | **异常关闭——保留码，永远不会出现在线上帧里** |

`1006` 是本地库在"连接没走关闭握手就断了"时自己填的。**日志里看到一堆 1006，说明是网络断开或对端崩溃，不是对方发给你的。**

---

## 第五章：Go 里怎么用

### 5.1 三个库的选型

| 库 | 定位 | 什么时候用 |
|---|---|---|
| **`github.com/coder/websocket`** | context 原生、API 现代、零依赖 | **新项目默认选它** |
| `github.com/gorilla/websocket` | 生态最广、老项目标配 | 已有代码在用；需要它的特定扩展 |
| `github.com/gobwas/ws` | 零拷贝、可配合手写 epoll | 十万级以上连接、内存吃紧时 |

`coder/websocket` 就是原来的 `nhooyr.io/websocket`，2024 年迁移到 coder 组织，**import 路径变了**，老博客里的 `nhooyr.io/websocket` 需要替换。

`gorilla/websocket` 在 2022 年归档过一段时间，后来 gorilla 组织恢复维护，现在是活的——但新项目没理由不用 context 原生的那个。

**标准库没有 WebSocket。** `golang.org/x/net/websocket` 是个残废实现（官方文档自己写着"不完整，建议用 gorilla"），别用。

### 5.2 服务端（coder/websocket）

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    c, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        OriginPatterns: []string{"example.com"},
    })
    if err != nil {
        return   // Accept 已经写过 HTTP 错误响应了
    }
    defer c.CloseNow()   // 兜底：正常路径应该走 c.Close(StatusNormalClosure, "")

    // 每条连接一个独立超时上下文
    ctx, cancel := context.WithTimeout(r.Context(), 30*time.Minute)
    defer cancel()

    c.SetReadLimit(64 * 1024)   // 必设：默认 32KB，超了直接断

    for {
        var msg Message
        if err := wsjson.Read(ctx, c, &msg); err != nil {
            // 正常关闭不算错误
            if websocket.CloseStatus(err) == websocket.StatusNormalClosure ||
                websocket.CloseStatus(err) == websocket.StatusGoingAway {
                return
            }
            log.Printf("read: %v", err)
            return
        }
        if err := wsjson.Write(ctx, c, handle(msg)); err != nil {
            return
        }
    }
}
```

要点：

- **`SetReadLimit`**：不设的话默认 32KB，业务消息稍大就神秘断连。设了则明确拒绝超大消息。
- **`CloseStatus(err)`** 判断是不是正常关闭——不判的话日志里全是噪音。
- **`defer c.CloseNow()`** 是兜底，正常退出路径应该显式 `c.Close(websocket.StatusNormalClosure, "")` 走完关闭握手。

### 5.3 客户端

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

c, _, err := websocket.Dial(ctx, "wss://example.com/ws", &websocket.DialOptions{
    HTTPHeader: http.Header{"Authorization": {"Bearer " + token}},
})
if err != nil {
    return err
}
defer c.CloseNow()

if err := wsjson.Write(ctx, c, Message{Type: "hello"}); err != nil {
    return err
}
```

Go 客户端可以随便加 header。**浏览器不行**——这是下一节的重点。

### 5.4 浏览器端的两个硬限制

**限制一：不能自定义 header。**

```js
// ❌ 浏览器 WebSocket API 没有这个能力
new WebSocket(url, { headers: { Authorization: "Bearer ..." } })
```

所以浏览器场景的鉴权只有这几条路：

| 方案 | 评价 |
|---|---|
| Cookie（握手自动带） | 可行，但**必须**配合 Origin 校验防 CSWSH |
| URL query 带 token | 简单，但 token 会进 access log、Referer、浏览器历史 —— **只用短期一次性 token** |
| 塞进 `Sec-WebSocket-Protocol` | 常见 hack，服务端要在响应里回显选中的子协议 |
| **连上后第一条消息发 token** | **推荐**：不进日志，逻辑清晰。代价是要处理"已连接但未认证"状态，加个认证超时（比如 5 秒没发 token 就 1008 断开） |

**限制二：JS 不能主动发 ping 帧。**

浏览器会自动**回**服务端的 ping（回 pong），但 JS API 没有暴露"发 ping"的能力。所以：

- 服务端→客户端保活：用协议级 ping（`c.Ping(ctx)`），浏览器自动回 pong ✓
- 客户端→服务端保活：只能用**应用层心跳**，比如发 `{"type":"ping"}` 让服务端回 `{"type":"pong"}`

很多人在这里绕半天，以为库有 bug。不是 bug，是浏览器 API 就这样。

---

## 第六章：生产问题

### 6.1 心跳与死连接检测

**为什么 TCP KeepAlive 不够**：默认探测周期分钟级（Linux 默认 7200 秒才开始首次探测），而且中间的 NAT/LB 可能在 60 秒空闲后就静默丢弃连接映射——你的服务端还以为连接活着，写进去石沉大海。

正确做法：**应用层定期 ping，超时没 pong 就主动断**。

```go
func keepalive(ctx context.Context, c *websocket.Conn) {
    t := time.NewTicker(30 * time.Second)
    defer t.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-t.C:
            pingCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
            err := c.Ping(pingCtx)   // 阻塞等 pong
            cancel()
            if err != nil {
                c.Close(websocket.StatusPolicyViolation, "ping timeout")
                return
            }
        }
    }
}
```

间隔选择：要**小于**链路上最短的空闲超时。常见 LB/Nginx 默认 60 秒，所以 30 秒是安全值。

### 6.2 并发写：最容易踩的一个

> **WebSocket 连接支持一个并发读 + 一个并发写，但不支持多个 goroutine 同时写。**

两个 goroutine 同时 `Write` 会让两条消息的帧交错，产出非法流，对端直接断开。gorilla 会直接 panic（`concurrent write to websocket connection`）。

标准解法是 **writePump 模式**——所有写集中到一个 goroutine：

```go
type Client struct {
    conn *websocket.Conn
    send chan []byte    // 带缓冲
}

func (c *Client) writePump(ctx context.Context) {
    defer c.conn.CloseNow()
    for {
        select {
        case <-ctx.Done():
            return
        case msg, ok := <-c.send:
            if !ok {
                c.conn.Close(websocket.StatusNormalClosure, "")
                return
            }
            wctx, cancel := context.WithTimeout(ctx, 10*time.Second)
            err := c.conn.Write(wctx, websocket.MessageText, msg)
            cancel()
            if err != nil {
                return
            }
        }
    }
}

// 广播方只往 channel 里塞，不碰 conn
func (c *Client) Send(msg []byte) {
    select {
    case c.send <- msg:
    default:
        // 慢客户端：缓冲满了，丢弃并断开
        close(c.send)
    }
}
```

### 6.3 背压：慢客户端会拖垮你

一个手机在电梯里的客户端，TCP 窗口满了，你的 `Write` 就阻塞了。如果广播逻辑是同步循环写 1 万个客户端，**一个慢客户端能卡住整个广播**。

三层防护：

1. **每客户端带缓冲 channel**（上面的 `send chan []byte`，容量 64~256）
2. **满了就丢弃 + 断开**（`select` 的 `default` 分支）——实时系统里，过期的数据没有价值，断开让客户端重连补齐更好
3. **写超时**（`context.WithTimeout`）——绝不能无限等

```
慢客户端 → 缓冲满 → 主动断开 → 客户端重连 → 拉一次全量快照
```

这个"断开-重连-补全量"的模式，比任何缓冲策略都可靠。

### 6.4 水平扩展：WebSocket 是有状态的

HTTP 无状态，随便加机器。WebSocket 不行：用户 A 连在节点 1，用户 B 连在节点 2，A 发给 B 的消息怎么过去？

```
        ┌─ 节点1 ─┐        用户A
客户端 ─┤         ├─ Redis Pub/Sub ─┐
        └─ 节点2 ─┘        用户B    │
              ▲                     │
              └─────────────────────┘
```

标准方案：**节点间用 Redis Pub/Sub / NATS / Kafka 转发**。节点只管本地连接，跨节点消息走消息总线。

配套要处理的：

- **粘性会话**：LB 要配 sticky session，否则重连可能落到别的节点（用 Redis 存 `userID → nodeID` 映射也行）
- **优雅重启**：发布新版本时，先给所有连接发 `1001 (Going Away)`，让客户端主动重连到新节点，而不是等 TCP 超时
- **连接数监控**：`当前连接数`、`每秒消息数`、`发送队列积压` 是三个必须上报的指标

### 6.5 反向代理配置

WebSocket 走 HTTP/1.1 的 Upgrade 机制，**代理必须显式放行**：

```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;                      # 必须，默认 1.0 不支持 Upgrade
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;                    # 默认 60s，长连接会被切
    proxy_send_timeout 3600s;
}
```

**`proxy_read_timeout` 是最常见的坑**——默认 60 秒，表现为"连接每分钟准时断一次"。要么调大，要么把心跳间隔压到 60 秒以内（推荐后者，两个都做最保险）。

K8s Ingress 同理，nginx-ingress 用 `nginx.ingress.kubernetes.io/proxy-read-timeout` 注解。

### 6.6 关于 HTTP/2 和 HTTP/3

WebSocket 握手依赖 HTTP/1.1 的 `Upgrade`，HTTP/2 里没有这个机制。RFC 8441 定义了用 `CONNECT` + `:protocol` 伪头在 HTTP/2 上跑 WebSocket，但：

- **Go 标准库不支持**（`http.Hijacker` 在 HTTP/2 下直接返回错误）
- 实际部署中，即使前端是 HTTP/2，WebSocket 也会协商降到 HTTP/1.1

所以现阶段：**WebSocket 就是 HTTP/1.1 的东西**。如果确实需要 HTTP/2 上的多路复用流，考虑 gRPC 双向流（G28）或 WebTransport（HTTP/3，生态还很早期）。

---

## 第七章：常见陷阱清单

### ❌ 陷阱 1：把 `Read` 当消息边界
`n, _ := conn.Read(buf)` 的 `n` 只是"读到多少字节"。必须用长度前缀 / 分隔符 / `io.ReadFull`。

### ❌ 陷阱 2：解析长度头不设上限
对方发 `0xFFFFFFFF`，你 `make([]byte, 4GB)` → OOM。**任何从网络读来的长度都必须有 max 校验。**

### ❌ 陷阱 3：`CheckOrigin: return true`
本地联调改的，上线忘了改回来 → CSWSH，攻击者的页面能带着用户 Cookie 连你的服务。

### ❌ 陷阱 4：多 goroutine 并发写同一连接
帧交错 → 非法流 → 对端断开（gorilla 直接 panic）。用 writePump 集中写。

### ❌ 陷阱 5：只依赖 TCP KeepAlive
分钟级探测 + NAT 静默丢弃 = 你以为连着，其实早断了。必须应用层心跳。

### ❌ 陷阱 6：忘了 `SetReadLimit`
coder/websocket 默认 32KB，稍大的业务消息就被无声掐断，排查半天。

### ❌ 陷阱 7：广播时同步写所有客户端
一个慢客户端阻塞整个广播循环。每客户端一个带缓冲 channel + 满了就断。

### ❌ 陷阱 8：Nginx 没配 `proxy_read_timeout`
默认 60 秒，连接每分钟准时断，看起来像"网络不稳定"。

### ❌ 陷阱 9：日志里追查 close code 1006
1006 是本地生成的保留码，代表异常断开，不是对端发的。别去对端日志找对应的 1006。

### ❌ 陷阱 10：单向推送也上 WebSocket
只需要服务端推 → SSE 更省事（自动重连、走标准 HTTP、无心跳负担）。

---

## 第八章：学习路径 —— 按顺序打勾

理论看再多不如自己写一遍。下面每一项都是能在 1-3 小时内做完的最小实验，**按顺序做，每步都能验证**。

### 阶段一：Socket 基础（1 周）

- [ ] 用 `net.Listen` / `net.Dial` 写一个 TCP echo server + client，`nc` 能连上
- [ ] **复现粘包**：客户端 `for i := 0; i < 1000; i++ { conn.Write([]byte("msg")) }`，服务端打印每次 `Read` 的 `n`，观察它不等于 3
- [ ] 用第二章的**长度前缀协议**修好它，加上 `maxSize` 校验
- [ ] 给 server 加 `SetReadDeadline`，验证空闲连接会被踢掉
- [ ] 用 `netstat -an | grep 9000` 观察 TIME_WAIT，理解它为什么存在

> 📖 配套阅读：[Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)（免费，socket 入门最经典，C 语言但概念通用）

### 阶段二：WebSocket 协议（3-5 天）

- [ ] **通读 [RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455) 第 4 节（握手）和第 5 节（帧格式）** —— 一个下午能读完，收益最大的一步
- [ ] 手写 `Sec-WebSocket-Accept` 计算，用 RFC 的样例验证（`dGhlIHNhbXBsZSBub25jZQ==` → `s3pPLMBiTxaQ9kYGzzhZRbK+xOo=`）
- [ ] **手写帧解析器**（第 4.3 节的代码），用 RFC §5.7 的 `81 85 37 fa 21 3d 7f 9f 4d 51 58` 做单元测试
- [ ] 用 Chrome DevTools → Network → WS → Messages 面板看真实帧，对照你的理解

> 📖 配套阅读：[MDN WebSocket API](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSockets_API)（客户端视角）
> 🎬 视频：YouTube 搜 **Hussein Nasser — "WebSockets Crash Course"**，协议设计动机讲得最透

### 阶段三：Go 实战（1 周）

- [ ] 用 `coder/websocket` + `net/http` 写一个聊天室，浏览器 `new WebSocket()` 连上收发消息
- [ ] 加 **writePump 模式**，用 `-race` 验证没有并发写（`go run -race`）
- [ ] 加**心跳**（30s ping）和**客户端重连**（指数退避 + jitter）
- [ ] 加**鉴权**：连上后 5 秒内必须发 token，否则 `1008` 断开
- [ ] 故意做一个**慢客户端**（收到消息 `time.Sleep(10s)`），验证背压策略生效
- [ ] 起两个服务端实例 + Redis Pub/Sub，验证跨节点消息能通

### 阶段四：深入（可选，长期）

- [ ] **[Stanford CS144](https://cs144.github.io/)** —— lab 要求手写 TCP 协议栈。做完之后 socket 对你彻底没有黑盒。投入大（2-3 个月），收益也最大
- [ ] 《UNIX 网络编程 卷1》(UNP, Stevens) —— 不用通读，重点第 3/4/5/6/7 章（基本 TCP、I/O 复用、套接字选项）
- [ ] 读 `gorilla/websocket` 的 `conn.go` 源码，对照你手写的解析器看差距
- [ ] 读 Go 源码 `runtime/netpoll.go`，理解 goroutine 阻塞在 `Read` 时到底发生了什么（配合 G11）
- [ ] 用 `gobwas/ws` + epoll 实现单机 10 万连接，用 pprof 对比内存（配合 G22）

### 时间不够的最短路径

只有一个周末：**RFC 6455 第 4、5 节 + 阶段二的帧解析器 + 阶段三的前两项**。够你上手写生产代码了，剩下的遇到问题再回来补。

---

## 第九章：练习题

**练习 1**：下面代码有什么问题？至少说出 3 点。

```go
ln, _ := net.Listen("tcp", ":9000")
for {
    conn, _ := ln.Accept()
    go func() {
        buf := make([]byte, 1024)
        conn.Read(buf)
        process(buf)
    }()
}
```

**练习 2**：客户端发的 WebSocket 帧头是 `82 FE 04 00`，解析出：FIN、opcode、是否 mask、payload 长度。

**练习 3**：为什么 WebSocket 要求客户端 mask 而服务端不 mask？如果反过来会怎样？

**练习 4**：你的服务端日志里 90% 的断开都是 close code 1006，怎么排查？

**练习 5**：设计一个支持 10 万在线用户的聊天室，说明：连接如何分布、跨节点消息如何路由、慢客户端如何处理、发版时如何不影响用户。

---

## 参考答案

**练习 1**（至少 5 个问题）：

1. **`conn` 被闭包捕获** —— 循环变量在 Go 1.22+ 每轮是新变量所以这条不再是 bug，但显式传参 `go func(c net.Conn)` 仍是更安全的写法
2. **没有 `defer conn.Close()`** → fd 泄漏，最终 `too many open files`
3. **忽略所有 error** → `Accept` 失败会变成死循环空转烧 CPU
4. **`Read` 只调一次** → 粘包/半包，`buf` 里大概率不是完整消息
5. **没有 deadline** → 恶意客户端连上不发数据，goroutine 永久挂起（slowloris）
6. **`process(buf)` 传的是整个 1024 字节** → 应该是 `buf[:n]`

**练习 2**：

```
0x82 = 1000 0010 → FIN=1, RSV=000, opcode=0x2 (binary)
0xFE = 1111 1110 → MASK=1, len=126 → 读后续 2 字节
0x0400 = 1024    → payload 长度 1024 字节
后面还有 4 字节 masking key，然后才是 payload
```

**练习 3**：防止代理缓存投毒。中间代理若把 WebSocket 流误当 HTTP 解析，攻击者可构造 payload 伪装成 HTTP 响应毒化缓存。每帧随机 mask 让客户端无法控制线上的确定字节。反过来（服务端 mask）没有意义：攻击者控制的是客户端，服务端本来就发不出攻击者指定的内容。

**练习 4**：1006 是**本地库**在"没走关闭握手就断了"时填的保留码，不是对端发来的。排查方向：

- 中间设备超时（Nginx `proxy_read_timeout`、LB idle timeout、云厂商 NAT 网关）→ 缩短心跳间隔到 30s 以内
- 客户端网络切换（Wi-Fi ↔ 4G）→ 正常现象，做好重连
- 服务端 panic 或 OOM 导致进程重启 → 看服务端日志和 restart 次数
- 消息超过 `SetReadLimit` 被强制断开 → 日志里会有对应错误

**练习 5**（要点）：

| 维度 | 方案 |
|---|---|
| 连接分布 | LB（4 层优先，7 层要配 Upgrade 透传）+ sticky session；10 万连接约 10-20 个节点，单节点 5k-10k 连接 |
| 跨节点路由 | Redis Pub/Sub 按房间 channel 订阅；量大改 NATS / Kafka |
| 在线状态 | Redis 存 `userID → nodeID`，带 TTL，心跳时续期 |
| 慢客户端 | 每连接 128 缓冲 channel，满则丢弃并断开，客户端重连拉全量快照 |
| 发版 | 优雅退出：停止接受新连接 → 给存量连接发 `1001 Going Away` → 等 30s → 强制关闭；客户端带 jitter 的指数退避重连，避免惊群 |
| 监控 | 连接数、消息 QPS、发送队列积压、close code 分布、ping 往返延迟 |

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| Socket vs WebSocket | 前者是 API，后者是协议；WebSocket 也用 socket 实现 |
| TCP 语义 | 字节流不是消息流，**不保证消息边界** |
| 消息边界 | 长度前缀（配 `io.ReadFull` + maxSize）/ 分隔符 / 自描述格式 |
| Go 网络模型 | 每连接一 goroutine，runtime netpoller 底层跑 epoll |
| 握手 | HTTP `Upgrade` → 101；`SHA1(key + 258EAFA5-…)` |
| 安全 | **必须校验 Origin**，否则 CSWSH（WebSocket 不受同源策略保护） |
| 帧格式 | FIN + opcode + MASK + 变长 length；控制帧 ≤125 且不可分片 |
| 掩码 | 客户端必须 mask，防代理缓存投毒，非加密 |
| close code | 1000 正常 / 1001 离开 / 1008 策略 / **1006 是本地生成的异常码** |
| Go 库 | 新项目 `coder/websocket`；标准库无可用实现 |
| 并发写 | **一个连接只能一个 goroutine 写** → writePump 模式 |
| 心跳 | 应用层 ping，30s，必须短于链路最短 idle timeout |
| 浏览器限制 | 不能自定义 header，不能主动发 ping 帧 |
| 背压 | 缓冲 channel + 满则断开 + 写超时；重连补全量 |
| 扩展 | 有状态 → Redis Pub/Sub 跨节点 + sticky session |
| 代理 | Nginx 必配 `proxy_http_version 1.1` + Upgrade header + 大 `proxy_read_timeout` |
| 选型 | 单向推送优先 SSE，双向高频才用 WebSocket |

### 📅 2026 更新

| 变化 | 说明 |
|---|---|
| `nhooyr.io/websocket` → `github.com/coder/websocket` | 2024 年迁移，import 路径变更，老资料需替换 |
| `gorilla/websocket` 恢复维护 | 2022 归档后由 gorilla 组织复活，可以继续用 |
| `net.KeepAliveConfig`（Go 1.23+） | 可精细配置 TCP 保活探测间隔，但仍不能替代应用层心跳 |
| WebSocket over HTTP/2（RFC 8441） | Go 标准库仍不支持；实际部署仍走 HTTP/1.1 |

---

本篇是 **G31**，作为 G27（net/http）之后的网络编程延伸。想继续深入：

- **协议层**：[B01 — 精通互联网工作原理](../backend/B01-精通互联网工作原理.md)、[B02 — 精通 HTTP 语义](../backend/B02-精通-HTTP-语义.md)
- **单向流式**：[A12 — 精通流式输出与 SSE](../ai-backend/A12-精通-流式输出与-SSE.md)
- **RPC 双向流**：[G28 — 精通 gRPC 与 Protobuf](./G28-精通-Go-gRPC-与-Protobuf.md)
- **底层调度**：[G11 — 精通 Goroutines 与 GMP 调度](./G11-精通-Goroutines-与-GMP-调度.md)

---
