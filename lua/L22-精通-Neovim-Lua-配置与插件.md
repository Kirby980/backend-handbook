# 精通 Neovim Lua 配置与插件

> 课程编号：L22
> 路线图来源：Lua 全场景深度课程 — Neovim 应用篇
> 难度：⭐⭐⭐
> 预计阅读时间：50 分钟
> 📅 内容基准：**Neovim 0.10+**（内嵌 LuaJIT）

---

## 引言：编辑器也是一个 Lua 宿主

```lua
-- ~/.config/nvim/init.lua —— 整个配置就是一个 Lua 程序
vim.opt.number = true              -- 显示行号
vim.g.mapleader = " "              -- leader 键设为空格
vim.keymap.set("n", "<leader>w", ":w<CR>")   -- 空格+w 保存

-- 几个要点：
-- 1) vim.opt / vim.g / vim.api 各是什么？
-- 2) Neovim 跑的是哪个 Lua？
-- 3) 为什么 vim.api 的行号是 0-based，但有些地方又 1-based？
```

Neovim 把 **LuaJIT 嵌入编辑器**（用 C API，L15 的思想），让你用 Lua 取代古老的 Vimscript 来配置和扩展编辑器。`init.lua` 就是一个 Lua 程序，`vim.*` 是 Neovim 暴露的 API。这一章讲清 Neovim 的 Lua 配置体系、插件管理、LSP，以及用 Lua 写插件——它和 OpenResty（L17）、Redis（L21）一样，都是"宿主嵌入 Lua"的实例。

> Neovim 内嵌 **LuaJIT**（≈ Lua 5.1），所以前面 LuaJIT 的语言边界（无原生整型、无 `<close>`）同样适用。

---

## 第一章：Neovim 的 Lua 命名空间

### 1.1 `vim` 全局表

所有 Neovim API 挂在全局 `vim` 表下：

| 命名空间 | 用途 |
|---|---|
| `vim.opt` | 选项（option）——新式、对象化 |
| `vim.o` / `vim.bo` / `vim.wo` | 选项的标量访问（全局/缓冲区/窗口） |
| `vim.g` / `vim.b` / `vim.w` | 变量（全局/缓冲区/窗口） |
| `vim.api` | 核心 API（`nvim_*` 函数，最底层） |
| `vim.fn` | 调用 Vimscript 内建函数 |
| `vim.keymap` | 键位映射 |
| `vim.cmd` | 执行 Ex 命令（Vimscript 命令字符串） |
| `vim.loop` / `vim.uv` | libuv 事件循环（异步 IO） |
| `vim.lsp` | LSP 客户端 |
| `vim.diagnostic` | 诊断（错误/警告显示） |
| `vim.treesitter` | 语法树 |

### 1.2 `vim.opt` vs `vim.o`

```lua
-- vim.o：标量赋值（简单选项）
vim.o.number = true
vim.o.tabstop = 4

-- vim.opt：对象化，支持列表/集合选项的方法（推荐用于复杂选项）
vim.opt.shortmess:append("c")        -- 给列表选项追加
vim.opt.listchars = { tab = "▸ ", trail = "·" }   -- 表形式设 map 类选项
vim.opt.wildignore:append({ "*.o", "*.pyc" })
```

⚠️ `vim.opt` 返回的是一个**特殊对象**（有 `:append`/`:remove`/`:get` 方法），不是普通值。要拿原始值用 `vim.opt.x:get()`。简单布尔/数字选项用 `vim.o` 即可，列表/映射类选项用 `vim.opt`。

---

## 第二章：变量、键位与命令

### 2.1 变量

```lua
vim.g.mapleader = " "                -- 全局变量（很多插件读它）
vim.g.loaded_netrw = 1               -- 禁用内置插件
vim.b.my_buffer_var = "x"            -- 当前缓冲区变量
```

### 2.2 键位映射

```lua
-- vim.keymap.set(mode, lhs, rhs, opts)
vim.keymap.set("n", "<leader>w", ":w<CR>", { desc = "保存" })
vim.keymap.set("n", "<leader>f", function()   -- rhs 可以是 Lua 函数！
    require("telescope.builtin").find_files()
end, { desc = "查找文件" })
vim.keymap.set("i", "jk", "<Esc>", { silent = true })

-- 模式：n(普通) i(插入) v(可视) x t c ...；可传表 {"n", "v"}
```

`rhs` 能直接是 **Lua 函数**——这是相对 Vimscript 的巨大改进，逻辑全用 Lua 写。

### 2.3 执行命令

```lua
vim.cmd("colorscheme habamax")       -- 执行 Ex 命令
vim.cmd([[
    syntax on
    set background=dark
]])                                   -- 多行
vim.cmd.colorscheme("habamax")       -- 0.8+ 的函数式写法
```

---

## 第三章：`vim.api` 与自动命令

### 3.1 `vim.api`（nvim_* 核心 API）

最底层、最强大的 API，函数名都是 `nvim_*`：

```lua
local api = vim.api
api.nvim_buf_set_lines(0, 0, -1, false, { "line1", "line2" })   -- 设缓冲区内容
local lines = api.nvim_buf_get_lines(0, 0, -1, false)            -- 读
api.nvim_create_buf(false, true)                                 -- 创建缓冲区
api.nvim_open_win(buf, true, { relative = "editor", ... })       -- 浮动窗口
local pos = api.nvim_win_get_cursor(0)                           -- {行, 列}
```

⚠️ **`vim.api` 的行号是 0-based，列也是 0-based**（C API 风格，L15）——但 `nvim_win_get_cursor` 返回的**行是 1-based、列是 0-based**（历史原因）。这种混合索引是 Neovim Lua 的头号坑，查文档确认每个函数的约定。

### 3.2 自动命令（autocmd）

```lua
-- 创建 augroup（分组，便于清理）
local group = vim.api.nvim_create_augroup("MyConfig", { clear = true })

vim.api.nvim_create_autocmd("BufWritePre", {
    group = group,
    pattern = "*.lua",
    callback = function(args)         -- 用 Lua 回调
        vim.lsp.buf.format()          -- 保存前格式化
    end,
})

vim.api.nvim_create_autocmd("TextYankPost", {
    group = group,
    callback = function() vim.highlight.on_yank() end,   -- 复制时高亮
})
```

`callback` 用 Lua 函数（不再用 Vimscript），`args` 含事件信息（buf、file 等）。

---

## 第四章：插件管理——lazy.nvim

### 4.1 引导安装

现代 Neovim 用 **lazy.nvim** 管理插件（取代老的 packer/vim-plug）。它支持**惰性加载**（按需载入，加快启动）：

```lua
-- 引导 lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
    vim.fn.system({ "git", "clone", "--filter=blob:none",
        "https://github.com/folke/lazy.nvim.git", "--branch=stable", lazypath })
end
vim.opt.rtp:prepend(lazypath)

-- 声明插件
require("lazy").setup({
    { "nvim-treesitter/nvim-treesitter", build = ":TSUpdate" },
    { "neovim/nvim-lspconfig" },
    {
        "nvim-telescope/telescope.nvim",
        dependencies = { "nvim-lua/plenary.nvim" },
        keys = { { "<leader>f", desc = "查找" } },   -- 按键触发时才加载（lazy）
    },
})
```

### 4.2 惰性加载策略

lazy.nvim 可按多种条件延迟加载插件：

```lua
{
    "some/plugin",
    event = "InsertEnter",     -- 进入插入模式才加载
    ft = "python",             -- 打开 python 文件才加载
    cmd = "MyCommand",         -- 执行某命令才加载
    keys = "<leader>x",        -- 按某键才加载
}
```

惰性加载让有几十个插件的配置也能在几十毫秒内启动。

---

## 第五章：内置 LSP

Neovim 0.5+ 内置 **LSP 客户端**——直接和语言服务器（rust-analyzer、pyright、lua-ls）通信，提供补全、跳转、诊断、重命名。

```lua
-- 用 nvim-lspconfig 简化配置
local lspconfig = require("lspconfig")

lspconfig.lua_ls.setup({
    settings = { Lua = { diagnostics = { globals = { "vim" } } } },
    on_attach = function(client, bufnr)
        -- LSP 附加到缓冲区时设置键位
        vim.keymap.set("n", "gd", vim.lsp.buf.definition, { buffer = bufnr })
        vim.keymap.set("n", "K", vim.lsp.buf.hover, { buffer = bufnr })
        vim.keymap.set("n", "<leader>rn", vim.lsp.buf.rename, { buffer = bufnr })
    end,
})

lspconfig.pyright.setup({})
```

`vim.lsp.buf.*` 提供 LSP 动作；诊断通过 `vim.diagnostic.*` 显示。0.11 起 Neovim 提供了更直接的 `vim.lsp.config`/`vim.lsp.enable` API，逐步减少对 lspconfig 的依赖。

---

## 第六章：写一个插件 + 异步

### 6.1 一个简单插件

插件就是一个 Lua 模块（L10），放在 `runtimepath` 的 `lua/` 下：

```lua
-- lua/myplugin/init.lua
local M = {}

function M.setup(opts)
    opts = opts or {}
    vim.api.nvim_create_user_command("Greet", function(args)
        vim.notify("Hello, " .. (args.args ~= "" and args.args or "Neovim"))
    end, { nargs = "?" })
end

return M
```

用户 `require("myplugin").setup()` 后，`:Greet World` 就可用。`vim.notify` 是标准通知接口。

### 6.2 异步——`vim.loop`/`vim.uv` 与 `vim.schedule`

Neovim 通过 libuv 提供异步（`vim.loop`，0.10+ 别名 `vim.uv`）。但**大多数 `vim.api` 只能在主循环调用**——在 libuv 回调里直接调 API 会出错，要用 `vim.schedule` 切回主线程：

```lua
-- 异步执行外部命令
vim.loop.spawn("ls", { args = { "-la" } }, function(code)
    vim.schedule(function()           -- 切回主循环才能调 vim.api
        vim.notify("命令完成，退出码 " .. code)
    end)
end)

-- 定时器
local timer = vim.loop.new_timer()
timer:start(1000, 0, vim.schedule_wrap(function()
    vim.notify("1 秒后")
end))
```

⚠️ **libuv 回调不在主循环**——里面不能直接调 `vim.api`/`vim.fn`（会报 "API called from fast event context"），必须 `vim.schedule(fn)` 或 `vim.schedule_wrap(fn)` 包裹。这是 Neovim 异步编程的头号坑（类似 OpenResty"哪个阶段能 yield"，L17）。

---

## 第七章：常见陷阱清单

### ❌ 陷阱 1：`vim.opt` 当普通值用

```lua
local n = vim.opt.number   -- ❌ 这是对象不是布尔
local n = vim.opt.number:get()   -- ✅ 或 vim.o.number
```

### ❌ 陷阱 2：混淆行号索引

```lua
vim.api.nvim_buf_set_lines(0, 0, -1, ...)   -- 0-based 行
local row = vim.api.nvim_win_get_cursor(0)[1]  -- 行是 1-based！
```
每个 API 查文档确认索引基。

### ❌ 陷阱 3：libuv 回调里直接调 vim.api

```lua
vim.loop.new_timer():start(100, 0, function()
    vim.api.nvim_command("echo 'hi'")   -- ❌ fast event context 报错
end)
```
用 `vim.schedule_wrap`。

### ❌ 陷阱 4：插件加载顺序/leader 设置晚了

```lua
-- mapleader 必须在加载插件、定义映射之前设置
require("lazy").setup(...)
vim.g.mapleader = " "   -- ❌ 太晚，插件已用默认 leader
```
`vim.g.mapleader` 放最前面。

### ❌ 陷阱 5：autocmd 不分组导致重复

```lua
-- 重载配置时 autocmd 累积，触发多次
```
用 `nvim_create_augroup(name, { clear = true })`。

### ❌ 陷阱 6：以为能用 Lua 5.3+ 特性

```lua
local n = 7 // 2   -- LuaJIT 下是 double
```
Neovim 是 LuaJIT，遵守 5.1 边界。

---

## 第八章：练习题

**练习 1**：`vim.opt.listchars = { tab = "▸ " }` 和 `vim.o` 能这样写吗？区别？

**练习 2**：映射 `<leader>q` 为关闭窗口，rhs 用 Lua 函数。

**练习 3**：为什么 libuv 回调里要用 `vim.schedule`？

**练习 4**：写一个保存 `.lua` 文件前自动格式化的 autocmd。

**练习 5**：判断真假——"Neovim 的 `vim.api` 行号和列号都统一是 1-based。"

---

## 参考答案与解析

**练习 1**：`vim.opt` 能（它支持 table 形式设置 map/list 类选项）；`vim.o.listchars` 只能赋字符串（如 `"tab:▸ ,trail:·"`）。复杂选项用 `vim.opt` 更方便。

**练习 2**：
```lua
vim.keymap.set("n", "<leader>q", function()
    vim.api.nvim_win_close(0, false)
end, { desc = "关闭窗口" })
```

**练习 3**：libuv 回调运行在"fast event context"（非主循环），大多数 `vim.api`/`vim.fn` 不能在此调用（会报错）。`vim.schedule(fn)` 把 fn 排到主循环安全执行。

**练习 4**：
```lua
vim.api.nvim_create_autocmd("BufWritePre", {
    group = vim.api.nvim_create_augroup("fmt", { clear = true }),
    pattern = "*.lua",
    callback = function() vim.lsp.buf.format() end,
})
```

**练习 5**：**假**。`vim.api` 多数是 0-based（行、列），但部分函数如 `nvim_win_get_cursor` 返回行 1-based、列 0-based。索引基不统一，是常见坑。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 宿主 | Neovim 嵌入 **LuaJIT**；`init.lua` 是 Lua 程序，`vim.*` 是 API |
| 命名空间 | opt/o(选项) g/b/w(变量) api(核心) fn(vimscript) keymap cmd |
| vim.opt | 对象化，`:append`/`:get`；复杂选项用它，简单用 vim.o |
| 键位 | `vim.keymap.set`，rhs 可为 **Lua 函数** |
| api | `nvim_*`；**索引基不统一**（多 0-based，cursor 行 1-based）|
| autocmd | `nvim_create_autocmd` + augroup(clear) + Lua callback |
| 插件 | **lazy.nvim** + 惰性加载（event/ft/cmd/keys） |
| LSP | 内置客户端 + lspconfig；`vim.lsp.buf.*` |
| 异步 | `vim.loop`/`vim.uv`(libuv)；回调里必须 `vim.schedule` |

---

## 📅 2026 现状/更新

- **Neovim 0.10/0.11** 持续强化原生 LSP（`vim.lsp.config`/`enable`）、Treesitter、`vim.uv` 等，逐步减少对外部样板的依赖。
- **lazy.nvim** 是 2026 插件管理的事实标准；配置社区（LazyVim、kickstart.nvim）大量基于它。
- Neovim 是"编辑器嵌入 Lua"的典范——与 OpenResty（网关嵌 Lua）、Redis（DB 嵌 Lua）、游戏引擎（L23）并列，展示 Lua "可嵌入"设计哲学（L01）的广度。

---

> 🔁 下一篇 **L23 — 精通 Lua 游戏脚本：Love2D 与嵌入**：最后一个应用场景。为什么游戏钟爱 Lua、Love2D 的游戏循环、把 Lua 嵌入 C/C++ 引擎、热重载与沙箱（防不可信脚本逃逸）。
>
> 反馈：把"libuv 回调要 vim.schedule"这条记牢——它和 OpenResty 的阶段限制是同一类"执行上下文"问题。
