# 精通 Lua 基于元表的 OOP

> 课程编号：L11
> 路线图来源：Lua 全场景深度课程 — 面向对象
> 难度：⭐⭐⭐⭐
> 预计阅读时间：55 分钟
> 📅 内容基准：**Lua 5.4.8** + **LuaJIT 2.1**

---

## 引言：Lua 没有 class，却能做面向对象

```lua
-- 1) 这段"类"为什么能工作？
local Animal = {}
Animal.__index = Animal
function Animal.new(name) return setmetatable({ name = name }, Animal) end
function Animal:speak() return self.name .. " makes a sound" end

local a = Animal.new("Dog")
print(a:speak())

-- 2) 冒号和点的区别？
print(a:speak() == a.speak(a))   -- ?

-- 3) 继承——输出？
local Dog = setmetatable({}, { __index = Animal })
Dog.__index = Dog
function Dog:speak() return self.name .. " barks" end
local d = setmetatable({ name = "Rex" }, Dog)
print(d:speak(), d.name)
```

答案：① 它能工作是因为 `Animal.__index = Animal` 让实例（缺方法时）回到类表找方法；`new` 用 `setmetatable` 把实例和类关联；② `true`——`a:speak()` 是 `a.speak(a)` 的语法糖，冒号自动传 `self`；③ `Rex barks  Rex`（Dog 重写了 speak，继承了 name 字段机制）。

Lua **没有内置 class 关键字**，但凭借元表（L06），几行代码就能搭出完整的对象系统——类、实例、继承、多态、运算符重载。这一章把"用机制拼出 OOP"讲透，并解析主流类库（middleclass）的原理。

---

## 第一章：从零搭一个类

### 1.1 三件套

一个最简单的"类"需要三步：

```lua
local Point = {}          -- ① 类就是一张表
Point.__index = Point     -- ② 让实例能在类里找方法（关键！）

function Point.new(x, y)  -- ③ 构造函数
    local self = setmetatable({}, Point)   -- 新实例，元表设为类
    self.x = x
    self.y = y
    return self
end

function Point:distance()  -- 方法（冒号 → 隐式 self）
    return math.sqrt(self.x^2 + self.y^2)
end

local p = Point.new(3, 4)
print(p:distance())        -- 5.0
```

**核心机制**：实例 `p` 是个空表，元表是 `Point`。当访问 `p.distance` 时，`p` 自己没有，触发 `__index = Point`，于是去 `Point` 里找到 `distance` 方法。`Point.__index = Point` 这一行是整个 OOP 的枢纽。

```mermaid
graph LR
    P["实例 p<br>{x=3, y=4}"] -->|元表| MT["Point 表<br>__index=Point<br>distance(), new()"]
    P -.读 p.distance 缺失.-> Idx["__index → Point"]
    Idx -.找到.-> Method[distance 方法]
    style MT fill:#4299e1,color:#fff
```

### 1.2 冒号 vs 点

```lua
function Point:distance() ... end      -- 定义：冒号 → 隐式 self 参数
function Point.distance(self) ... end  -- 等价写法（显式 self）

p:distance()        -- 调用：冒号 → 自动传 p 作 self
p.distance(p)       -- 等价（显式传 self）
```

**规则**：
- **定义**用冒号 → 方法体内有隐式 `self`。
- **调用**用冒号 → 自动把"点左边的对象"作第一个参数。
- 静态方法（如 `new`，不需要实例）用**点**定义和调用。

⚠️ 混用是头号 bug 源（见陷阱）。

---

## 第二章：单继承

### 2.1 链式 `__index`

子类继承父类，靠把**子类表的元表设为父类**（这样子类找不到的会去父类找），同时子类要做自己的 `__index`：

```lua
local Animal = {}
Animal.__index = Animal
function Animal.new(name)
    return setmetatable({ name = name }, Animal)
end
function Animal:speak() return self.name .. " makes a sound" end
function Animal:getName() return self.name end

-- Dog 继承 Animal
local Dog = setmetatable({}, { __index = Animal })   -- Dog 的方法找不到 → 去 Animal
Dog.__index = Dog
function Dog.new(name, breed)
    local self = Animal.new(name)        -- 调用父构造
    setmetatable(self, Dog)              -- 但元表设为 Dog
    self.breed = breed
    return self
end
function Dog:speak() return self.name .. " barks" end   -- 重写（多态）

local d = Dog.new("Rex", "Lab")
print(d:speak())      -- Rex barks（Dog 重写的）
print(d:getName())    -- Rex（继承自 Animal）
print(d.breed)        -- Lab
```

两条查找链：
- **实例 → Dog**（实例的元表是 Dog，`__index = Dog`）。
- **Dog → Animal**（Dog 表的元表 `__index = Animal`）。

所以 `d:getName()`：实例没有 → 去 Dog（没有）→ 去 Animal（找到）。

### 2.2 调用父类方法（super）

```lua
function Dog:speak()
    local base = Animal.speak(self)        -- 显式调父类方法，传 self
    return base .. " (and barks)"
end
```

Lua 没有 `super` 关键字，直接 `Parent.method(self, ...)`。

---

## 第三章：用闭包实现"真私有"

元表方案里实例字段都是公开的（`d.name` 谁都能读写）。要**真私有**可用闭包（L07）——不需要元表：

```lua
local function Counter(start)
    local count = start or 0           -- 私有，外部无法访问
    return {
        inc = function() count = count + 1; return count end,
        get = function() return count end,
    }
end

local c = Counter(10)
print(c.inc(), c.get())   -- 11  11
-- c.count                 -- nil，外部碰不到
```

**取舍**：闭包方案私有性强，但每个实例的方法都是独立闭包（内存开销大、无法用元表共享方法、不便继承）。元表方案省内存、易继承，但无真私有（靠约定，如 `_私有` 前缀）。**多数 Lua 代码用元表方案**，私有靠命名约定。

---

## 第四章：多重继承

### 4.1 `__index` 为函数遍历多个父

单个 `__index` 只能指一张表。多重继承让 `__index` 为**函数**，依次查多个父：

```lua
local function createClass(...)
    local parents = { ... }
    local cls = {}
    cls.__index = function(obj, key)
        for _, parent in ipairs(parents) do
            local v = parent[key]
            if v ~= nil then return v end   -- 第一个找到的优先
        end
    end
    return cls
end

local Walker = { walk = function() return "walking" end }
local Swimmer = { swim = function() return "swimming" end }
local Amphibian = createClass(Walker, Swimmer)
Amphibian.__index = Amphibian   -- 注意：上面被覆盖了，实际要小心组织

local frog = setmetatable({}, Amphibian)
-- 简化演示：实际多重继承要谨慎处理方法解析顺序
```

多重继承在 Lua 里能做但容易出歧义（菱形问题、解析顺序），实践中**组合（composition）优于继承**：

```lua
-- 组合：把能力作为字段
local frog = {
    walker = Walker,
    swimmer = Swimmer,
}
print(frog.walker.walk())
```

### 4.2 Mixin（混入）

更轻量的复用——把一组方法"混入"类表：

```lua
local Comparable = {
    lt = function(self, o) return self.value < o.value end,
}
local function mixin(cls, ...)
    for _, m in ipairs({...}) do
        for k, v in pairs(m) do cls[k] = v end
    end
    return cls
end

local Money = {}
Money.__index = Money
mixin(Money, Comparable)   -- Money 现在有 lt 方法
```

---

## 第五章：类库——middleclass 的原理

手写 OOP 样板代码多，社区有成熟类库，最流行的是 **middleclass**：

```lua
local class = require("middleclass")

local Animal = class("Animal")
function Animal:initialize(name) self.name = name end   -- 构造
function Animal:speak() return self.name .. " sound" end

local Dog = class("Dog", Animal)        -- 继承
function Dog:speak() return self.name .. " barks" end

local d = Dog("Rex")                     -- 调用类 = 创建实例
print(d:speak())                         -- Rex barks
print(d:isInstanceOf(Animal))            -- true
```

middleclass 内部正是用本章的元表机制封装的：
- `class(name)` 创建类表 + 元表（`__call` 让 `Dog("Rex")` 等于实例化，见 L06）。
- `initialize` 约定为构造方法。
- 维护 `super`、`isInstanceOf`、`isSubclassOf` 等内省。

理解了前几章，你会发现类库只是把样板代码包了一层。其它选择：`30log`、`classic`、LÖVE 社区的各种实现。

---

## 第六章：常见陷阱清单

### ❌ 陷阱 1：忘记 `Class.__index = Class`

```lua
local C = {}
function C:m() end
local o = setmetatable({}, C)   -- ❌ 漏了 C.__index = C
o:m()                            -- attempt to call a nil value（找不到 m）
```
修正：`C.__index = C`。

### ❌ 陷阱 2：定义/调用冒号点混用

```lua
function Obj.speak() return self.name end   -- ❌ 点定义，self 是全局 nil
local o = Obj.new()
o.speak()                                    -- ❌ 点调用，没传 self
```
修正：方法用冒号定义和调用。

### ❌ 陷阱 3：所有实例共享了可变默认值

```lua
local C = { items = {} }   -- ❌ 这个 items 表被所有实例共享！
C.__index = C
function C.new() return setmetatable({}, C) end
local a, b = C.new(), C.new()
a.items[1] = "x"
print(b.items[1])          -- "x"！共享了同一个 items
```
修正：在 `new` 里给每个实例创建自己的 `self.items = {}`。

### ❌ 陷阱 4：继承时元表设置顺序错

```lua
-- 子类既要 __index 指自己（给实例用），又要元表指父类（给类找父方法）
local Sub = setmetatable({}, { __index = Base })  -- 类→父
Sub.__index = Sub                                  -- 实例→类
```
两条链都要正确建立。

### ❌ 陷阱 5：用 `super` 关键字

```lua
function Dog:speak() return super.speak(self) end   -- ❌ Lua 没有 super
```
修正：`Animal.speak(self)`。

### ❌ 陷阱 6：在方法里忘了 `self`

```lua
function Account:deposit(n) balance = balance + n end   -- ❌ balance 是全局
```
修正：`self.balance = self.balance + n`。

---

## 第七章：练习题

**练习 1**：补全使这段能工作：
```lua
local Circle = {}
___________________
function Circle.new(r) return setmetatable({ r = r }, Circle) end
function Circle:area() return math.pi * self.r^2 end
print(Circle.new(2):area())
```

**练习 2**：实现一个 `Stack` 类（push/pop/size），用元表方案。

**练习 3**：找 bug：
```lua
local Vec = { data = {} }
Vec.__index = Vec
function Vec.new() return setmetatable({}, Vec) end
function Vec:add(x) table.insert(self.data, x) end
local a = Vec.new(); a:add(1)
local b = Vec.new()
print(#b.data)   -- 期望 0，实际？
```

**练习 4**：给 `Dog` 实现 `speak`，调用父类 `Animal:speak` 并拼接。

**练习 5**：判断真假——"Lua 必须用 middleclass 这样的库才能做面向对象。"

---

## 参考答案与解析

**练习 1**：`Circle.__index = Circle`。这是 OOP 枢纽。输出 `12.566...`。

**练习 2**：
```lua
local Stack = {}
Stack.__index = Stack
function Stack.new() return setmetatable({ items = {} }, Stack) end
function Stack:push(v) self.items[#self.items + 1] = v end
function Stack:pop() return table.remove(self.items) end
function Stack:size() return #self.items end
```

**练习 3**：`1`，不是 0！`data` 定义在类表 `Vec` 上被所有实例共享，`a:add(1)` 改的是共享的 `Vec.data`，`b.data` 也指向它。修正：`new` 里 `setmetatable({ data = {} }, Vec)`。

**练习 4**：
```lua
function Dog:speak()
    return Animal.speak(self) .. "，而且汪汪叫"
end
```

**练习 5**：**假**。Lua 用元表机制即可完整实现 OOP，类库只是封装样板。

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| 类 | 一张表 + `Class.__index = Class`（枢纽） |
| 实例化 | `setmetatable({}, Class)`，缺方法回类表找 |
| 冒号 | 定义/调用都用冒号 → 隐式/自动 `self` |
| 继承 | 子类元表 `__index` 指父；两条查找链 |
| super | 没有关键字，直接 `Parent.method(self,...)` |
| 私有 | 元表方案靠约定；要真私有用闭包（代价：内存/继承） |
| 多重继承 | `__index` 为函数遍历父；优先**组合/mixin** |
| 共享陷阱 | 可变默认值放类表会被所有实例共享；实例字段在 new 里建 |

---

## 📅 2026 现状/更新

- 元表 OOP 跨 5.1–5.4 与 LuaJIT 完全一致，是最可移植的对象方案。
- 类库生态：**middleclass** 仍是最流行选择；游戏（LÖVE）社区有大量自家实现（L23）。
- OpenResty 的 `lua-resty-*` 库大量用"返回带方法的 table + `new()`"模式封装连接对象（L19），正是本章机制的工程应用。

---

> 🔁 下一篇 **L12 — 精通 Lua 垃圾回收与弱表**：增量 GC 与 5.4 的分代 GC、`collectgarbage` 调优、`__gc` 终结器、弱表（weak table）做缓存与对象关联，以及"对象复活"等微妙问题。
>
> 反馈：把"实例字段 vs 类共享字段"的区别彻底搞清——练习 3 是真实项目里常见的 bug。
