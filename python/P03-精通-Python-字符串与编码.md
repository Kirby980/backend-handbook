# 精通 Python 字符串与编码

> 字符串看似简单,却是 bug 重灾区:`str` 和 `bytes` 混用、编码报错、循环拼接拖慢程序。面试常问:"`str` 和 `bytes` 区别?""为什么循环里 `+=` 拼字符串慢?"本篇讲清 Unicode、编解码与 CPython 的字符串实现,串起 [P01 不可变](./P01-精通-Python-数据模型与对象.md)、[P02 容器](./P02-精通-Python-内置容器底层.md)。
>
> **📅 基准:2026 年 6 月,Python 3.13 / 3.14。**

---

## 一、str 是码点,bytes 是字节

Python 3 严格区分两种"字符串":

- **`str`**:**Unicode 文本**——逻辑上是一串 **Unicode 码点(code point)**。它是人类可读的字符序列,不关心底层字节。
- **`bytes`**:**原始字节序列**——0-255 的字节,用于 IO、网络、文件等"二进制世界"。

```python
s = "你好"          # str:2 个码点
b = "你好".encode("utf-8")   # bytes:b'\xe4\xbd\xa0\xe5\xa5\xbd',6 个字节
print(len(s), len(b))        # 2 6
```

二者**不能直接混用/拼接**(`"a" + b"b"` 报 `TypeError`)。文本处理用 `str`,IO 边界用 `bytes`——**读入后尽早 decode 成 str,输出前再 encode 成 bytes**(所谓"Unicode 三明治")。

---

## 二、编码与解码

- **编码(encode)**:`str` → `bytes`,把码点按某种编码规则转成字节。
- **解码(decode)**:`bytes` → `str`,把字节按编码规则还原成码点。

```python
"héllo".encode("utf-8")        # b'h\xc3\xa9llo'
b'h\xc3\xa9llo'.decode("utf-8")  # 'héllo'
```

- **UTF-8** 是事实标准:变长(ASCII 1 字节、中文 3 字节),兼容 ASCII,Web/文件首选。
- **用错编码会出错或乱码**:`UnicodeDecodeError`(字节序列不符合该编码)、`UnicodeEncodeError`。错误处理参数 `errors="ignore"/"replace"/"strict"`。
- **别依赖"默认编码"**:`open()` 的默认编码依赖系统 locale(Windows 常是 GBK),跨平台会坑。**显式写 `encoding="utf-8"`**(3.15 起 `open` 默认 UTF-8 是趋势,但显式最稳)。

---

## 三、字符串不可变 → 拼接要用 join

`str` 是**不可变**的(见 [P01](./P01-精通-Python-数据模型与对象.md)):任何"修改"都生成新字符串。

```python
# ❌ 循环里 += 拼接:每次都创建新字符串,整体可能退化到 O(n²)
s = ""
for w in words:
    s += w
# ✅ 用 list 收集 + 一次 join:O(n)
s = "".join(words)
```

`"".join(iterable)` 一次性算好总长、分配一块内存、拷贝进去,是拼接大量字符串的正确姿势。少量、固定个数的拼接用 `+` 或 f-string 没问题。

> CPython 对 `s += x` 这种"局部变量、引用唯一"的情况有原地扩展优化,但**不可移植、不可依赖**(其他实现/写法就退化),大量拼接一律用 join。

---

## 四、格式化:首选 f-string

```python
name, age = "Alice", 30
f"{name} is {age}"               # ✅ f-string(3.6+):最快、最易读
f"{age=}"                        # 'age=30' 调试神器(3.8+)
f"{3.14159:.2f}"                 # '3.14' 格式说明符
"{} is {}".format(name, age)     # str.format:动态模板时用
"%s is %d" % (name, age)         # %:老式,logging 仍用(惰性求值)
```

- **f-string** 是首选:编译期解析、运行最快、可读性最好;支持 `{expr}`、`{val:format}`、`{val=}`。
- **3.12 起 f-string 更灵活**(PEP 701):可嵌套同种引号、多行、任意表达式。
- **logging 用 `%` 占位、不要用 f-string**:`log.info("x=%s", x)` 惰性求值,日志级别未启用时不会做格式化(见 [P27](./P27-精通-Python-日志与可观测.md))。

---

## 五、CPython 的字符串实现

- **灵活字符串表示(PEP 393)**:CPython 不固定用 UTF-8 存 str,而是按内容选最省的编码——纯 ASCII/Latin-1 用 1 字节/码点、含较大码点用 2 字节(UCS-2)、再大用 4 字节(UCS-4)。所以 `str` 内部是**定宽**的,`s[i]` 取第 i 个码点是 **O(1)**(不像 UTF-8 变长需要扫描)。
- **intern(驻留)**:相同内容的字符串可共享同一对象省内存。**像标识符的字符串(变量名样式)会被自动 intern**;可用 `sys.intern(s)` 手动驻留高频字符串,加速 `==`(先比身份)。这也是 [P01](./P01-精通-Python-数据模型与对象.md) 里 `is` 比较字符串结果"时灵时不灵"的原因——别用 `is` 比字符串内容。
- `ord(c)`/`chr(n)` 在字符与码点间转换;`\u`/`\U`/`\N{...}` 是码点转义。

---

## 六、常用操作

```python
"  hi ".strip()                  # 去空白
"a,b,c".split(",")               # ['a','b','c']
"a-b-c".replace("-", "_")
"Hello".lower() / .upper() / .title()
"abc".startswith("ab") / .endswith("c")
"name: alice".partition(": ")    # ('name', ': ', 'alice')
text.encode().decode()
```

字符串方法都返回**新字符串**(不可变)。正则用 `re`(预编译 `re.compile` 复用);大文本搜索注意复杂度。

---

## 陷阱清单

- **`str` 与 `bytes` 混用**:`"a" + b"b"`、对 bytes 用文本操作。读入尽早 decode、输出再 encode。
- **循环里 `+=` 拼接大量字符串**:可能 O(n²);用 `"".join(list)`。
- **依赖 `open()` 默认编码**:跨平台乱码。显式 `encoding="utf-8"`。
- **用 `is` 比较字符串内容**:依赖 intern,不可靠;用 `==`。
- **`len(s)` 当作字节数**:`len` 是码点数;字节数是 `len(s.encode("utf-8"))`(且含 emoji 等会更复杂)。
- **`UnicodeDecodeError` 用 `errors="ignore"` 掩盖**:会丢数据;先搞清真实编码。
- **logging 用 f-string**:丧失惰性求值;用 `%` 占位参数。
- **正则在循环里重复编译**:用 `re.compile` 预编译复用。

---

## 2026 现状

- **UTF-8 是绝对主流**;Python 持续推进"默认 UTF-8"(PEP 686,逐步把 `open`/locale 默认转 UTF-8),但**显式 `encoding="utf-8"` 仍是最佳实践**。
- **f-string 是格式化的默认选择**,3.12 的 PEP 701 让它几乎无语法限制。
- **`str` 处理大文本/高频拼接**仍要注意复杂度;模板/大量文本可考虑 `io.StringIO`。
- 与 Java([J03](../java/J03-精通-String与不可变设计.md))对照:都不可变、都有常量池/intern 思想,但 Java 的 `String` 是 UTF-16、Python 的 `str` 是码点 + 灵活表示;拼接都推荐 builder/join。

---

## 练习题

1. Python 3 中 `str` 和 `bytes` 有什么区别?为什么不能直接混用?

<details><summary>参考答案</summary>

`str` 表示 **Unicode 文本**——逻辑上是一串 Unicode 码点(字符),是人类可读、与具体字节编码无关的"文本世界";`bytes` 表示**原始字节序列**——每个元素是 0-255 的字节,是用于文件、网络、二进制协议的"字节世界"。二者是**不同类型、语义不同**:str 关心"是什么字符",bytes 关心"是哪些字节"。它们不能直接混用(如 `"a" + b"b"` 会抛 `TypeError`),因为从字符到字节、字节到字符之间**必须经过明确的编码(encode)/解码(decode)、指定编码方式**——同一串字符用 UTF-8 和 GBK 编码出的字节完全不同,Python 不会替你猜该用哪种编码,所以禁止隐式转换以避免乱码和数据错误。正确做法是遵循"Unicode 三明治"原则:**程序边界(读文件/网络)拿到的是 bytes,尽早 `decode` 成 str 在内部处理;要输出时再 `encode` 成 bytes**,中间逻辑全程用 str。`s.encode("utf-8")` 把 str 转 bytes,`b.decode("utf-8")` 把 bytes 转 str。这种严格区分正是 Python 3 相比 Python 2(str 同时承担字节和文本、隐式转换导致大量乱码 bug)的重大改进。

</details>

2. 为什么在循环里用 `+=` 拼接字符串可能很慢?应该怎么做?

<details><summary>参考答案</summary>

因为 **`str` 是不可变对象**:字符串一旦创建就不能原地修改,任何"拼接/修改"操作都会**创建一个全新的字符串对象**,把原内容和新内容拷贝进去。在循环里 `s += w` 时,第 k 次拼接需要把已有的(长度约正比于 k 的)字符串整个拷贝一遍再加上新片段,于是 N 次拼接的总拷贝量是 1+2+...+N ≈ O(n²),数据量大时会明显变慢。**正确做法**是先把各片段收集到一个 `list` 里,最后用 **`"".join(list)`** 一次性拼接:`join` 会先遍历算出结果总长度、一次性分配足够内存、再把各片段依次拷入,整体是 O(n)。例如 `parts = []; for w in words: parts.append(w); result = "".join(parts)`,或直接 `"".join(words)`。补充:①CPython 对 `s += x` 这种"`s` 是局部变量、且当前是唯一引用"的特殊情况做了原地扩展的优化,有时不会退化,但这**依赖实现、不可移植**(换个写法、换个解释器就退化),所以不能依赖;②少量、固定次数的拼接(如拼两三个变量)用 `+` 或 f-string 完全没问题,join 主要针对"循环中累积大量片段"的场景;③拼接时还可考虑 `io.StringIO` 作为可变缓冲。

</details>

3. 什么是编码和解码?为什么不要依赖 `open()` 的默认编码?

<details><summary>参考答案</summary>

**编码(encode)**是把 `str`(Unicode 文本/码点)按某种规则转换成 `bytes`(字节序列),**解码(decode)**是反过来把 bytes 按某种规则还原成 str。例如字符 'é' 用 UTF-8 编码是字节 `\xc3\xa9`,解码时必须也用 UTF-8 才能正确还原。常见编码 UTF-8(变长、兼容 ASCII、Web 与文件首选)、GBK、Latin-1 等;编码不匹配会导致乱码或抛 `UnicodeDecodeError`/`UnicodeEncodeError`。**不要依赖 `open()` 默认编码的原因**:`open(path)` 在文本模式下如果不显式指定 `encoding`,会使用**依赖操作系统 locale 的默认编码**——在 Linux/macOS 上通常是 UTF-8,但在 **Windows 上历史上常是 GBK/cp1252 等**。这导致同一份代码在不同平台、不同环境变量下读写同一文件时用了不同编码,出现"在我机器上好好的,到别人那里乱码/报错"的隐蔽 bug,且难以复现。**最佳实践是始终显式传 `encoding="utf-8"`**(读写都写),让行为跨平台一致、可预测。虽然 Python 正在推进默认 UTF-8(PEP 686,逐步把默认编码统一为 UTF-8),但在过渡期和为了代码清晰,**显式指定编码仍是最稳妥的做法**;处理无法确定编码的外部数据时,可借助 chardet 之类工具探测,或明确约定编码。

</details>

4. f-string、str.format、% 格式化各有什么特点?logging 为什么推荐用 % 而不是 f-string?

<details><summary>参考答案</summary>

三种格式化:①**f-string(f"...")**(3.6+)是首选——它在**编译期**解析,运行时最快,可读性最好,直接在 `{}` 里写表达式 `f"{a+b}"`、格式说明符 `f"{x:.2f}"`、自文档调试 `f"{x=}"`;3.12(PEP 701)后几乎无语法限制(可嵌套引号、多行)。②**str.format()** 用 `{}` 占位 + `.format(...)` 填充,适合**模板与数据分离**的场景(模板字符串先定义、稍后填充,或从配置读取模板),支持位置/命名/属性/索引访问。③**% 格式化**(`"%s" % x`)是最老式的 printf 风格,功能较弱,但在某些场景仍有用。**logging 推荐用 % 占位参数而非 f-string 的关键原因是惰性求值(lazy formatting)**:写 `logging.debug("user=%s data=%s", user, big_data)` 时,字符串的**实际格式化拼接被推迟到"确实要输出这条日志"时才进行**——如果当前日志级别高于 DEBUG(这条日志不会输出),logging 就**根本不会执行格式化**,省掉了拼接 `user`、`big_data` 的开销(尤其当参数的 `__str__`/`__repr__` 很昂贵时)。而如果写 `logging.debug(f"user={user} data={big_data}")`,f-string 在**调用 logging 之前就已经被求值拼接**成完整字符串了,无论该日志最终是否输出,格式化开销都已经发生,白白浪费。所以高频/调试日志、或参数构造昂贵时,用 `%` 占位 + 参数传入的形式更高效;此外 logging 的结构化处理也依赖这种参数化形式。

</details>

5. CPython 内部是怎么存储 str 的?为什么 `s[i]` 取单个字符是 O(1)?

<details><summary>参考答案</summary>

CPython 使用**灵活字符串表示(Flexible String Representation,PEP 393,3.3 引入)**:它不固定用某一种编码(如不是统一用 UTF-8)存储 `str`,而是**根据字符串中最大码点选择最省空间的定宽编码**——如果所有字符都在 Latin-1 范围(码点 ≤ 255),每个码点用 **1 字节**存;若最大码点在 BMP 内(≤ 0xFFFF),用 **2 字节**(UCS-2);若有更大的码点(如某些 emoji、罕见字),用 **4 字节**(UCS-4)。关键点是:对某个给定的字符串,它内部用的是**固定宽度**(每个码点占的字节数相同)的数组来存所有码点。正因为是定宽存储,**按下标取第 i 个字符 `s[i]` 只需计算 `基址 + i × 宽度` 直接寻址,是 O(1)**;切片、`len()` 也都很高效。这与变长编码(如直接用 UTF-8 字节流存)形成对比——UTF-8 中不同字符占 1-4 字节不等,要定位第 i 个字符必须从头扫描累加,是 O(n)。代价是灵活表示可能比 UTF-8 占更多内存(纯 ASCII 时 1 字节/字符与 UTF-8 一样省,但含少量大码点时整串都升到 2/4 字节)。另外 CPython 还会对像标识符那样的字符串做 **intern(驻留)** 以共享对象、节省内存并加速比较(先比身份)——这也是为什么不能用 `is` 比较字符串内容是否相等(结果依赖是否被 intern)。

</details>

6. `len(s)` 返回的是字符数还是字节数?如何得到一个字符串的 UTF-8 字节长度?

<details><summary>参考答案</summary>

对于 `str`,`len(s)` 返回的是**Unicode 码点的数量**(可以粗略理解为"字符数",但严格说是码点数),而**不是字节数**。例如 `len("你好")` 是 2(两个码点),`len("a")` 是 1。这与底层用多少字节存储无关——CPython 的灵活字符串表示里每个中文字符可能占 2 或 4 字节,但 `len` 只数码点个数。**要得到 UTF-8 编码后的字节长度**,需要先把 str 编码成 bytes 再取长度:`len(s.encode("utf-8"))`。例如 `len("你好".encode("utf-8"))` 是 6(每个汉字 UTF-8 占 3 字节),`len("héllo".encode("utf-8"))` 是 6('é' 占 2 字节、其余各 1)。这个区别在很多场景很重要:①给数据库字段、网络协议、缓冲区**按字节限长**时(如"最多 255 字节"),必须用编码后的字节数判断,而不是 `len(s)`;②不同编码字节数不同(GBK 中汉字是 2 字节、UTF-8 是 3 字节),所以一定要指定编码再 encode。还要注意一个更深的坑:即使是"码点数",也不完全等于"用户感知的字符数"——某些字符由多个码点组合而成(如带变音符号的字母、emoji 的组合序列、肤色修饰符等),`len` 会把它们数成多个码点,要按"用户可见字符(grapheme cluster)"计数需要专门的库(如 regex 的 `\X` 或第三方 grapheme 库)。

</details>
