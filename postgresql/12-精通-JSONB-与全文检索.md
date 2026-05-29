# 精通 JSONB 与全文检索：PostgreSQL 的半结构化武器库

> 关联章节：[P05 多类型索引](./05-精通-多类型索引.md)、[P10 EXPLAIN 与执行计划](./10-精通-EXPLAIN-与执行计划.md)、[P22 PG18/17 新特性](./22-精通-PG18-17-新特性.md)

---

## 引言：为什么 PG 能"一库当多库用"

很长一段时间里，半结构化数据和全文搜索都被推到独立系统里：MongoDB 存 JSON，Elasticsearch 做全文，ClickHouse 做日志聚合。每接一个系统，就多一份运维成本、多一条数据一致性裂缝、多一套客户端 SDK。

PostgreSQL 用两个一等公民的特性把这两件事吞下来：

1. **JSONB**——二进制存储、键去重、可建 GIN 索引、支持 jsonpath，原生压力测试可以扛住 MongoDB 80% 的查询场景。
2. **全文检索 (tsvector/tsquery)**——倒排索引、词典/词形归并/停用词、相关性排序，2006 年就开始演进，2026 年仍在主流生产中。

这章覆盖的不是手册，而是**真正决策层面的内容**：什么场景 JSONB 优于关系拆表，什么场景全文检索能替代 ES，索引选择如何不踩坑，2026 年 SQL/JSON 标准对老用法的影响有多大。

读完之后你应该能：

- 区分 JSON / JSONB 两种类型的物理差异和性能含义
- 默写出 `->`、`->>`、`#>`、`#>>`、`@>`、`?`、`?|`、`?&`、`@?`、`@@` 的语义
- 用 `jsonpath` 表达式查询深层嵌套 JSON 并配合 GIN 索引
- 用 PG 17+ 的 `JSON_EXISTS / JSON_VALUE / JSON_QUERY` 写 SQL/JSON 标准查询
- 用 PG 17+ 的 `JSON_TABLE` 把 JSON 数组炸成关系表
- 在 `jsonb_ops` 与 `jsonb_path_ops` 之间做正确选择
- 用表达式 GIN 索引把"高频路径"加速 100 倍
- 设计一套生产级中文全文检索（zhparser/pg_jieba）
- 给出 PG 全文检索 vs Elasticsearch 的诚实对比

---

## 第一章：JSON vs JSONB——一个字母两条路

### 1.1 物理存储差异

PostgreSQL 9.2 引入 `json`、9.4 引入 `jsonb`。看似只差一个字母，物理存储天差地别：

| 维度 | `json` | `jsonb` |
|---|---|---|
| 存储格式 | 原始文本 | 解析后的二进制 |
| 写入开销 | 几乎零（直接存字符串） | 解析 + 排序键 + 压缩 |
| 读取开销 | 每次访问要重新 parse | 直接读结构 |
| 键去重 | 保留所有重复键 | 重复键只保留最后一个 |
| 键顺序 | 严格保留输入顺序 | 不保证顺序（按长度+字典序重排） |
| 空白字符 | 保留 | 不保留 |
| 索引支持 | 仅 B-tree（整体相等） | GIN（路径/包含/存在） |
| `=` 比较 | 字符串比较（含空白） | 语义比较（结构相等） |

来一个直观对比：

```sql
SELECT '{"b":1, "a":2, "a":3,  "c": 4}'::json   AS j,
       '{"b":1, "a":2, "a":3,  "c": 4}'::jsonb  AS jb;

-- j  = {"b":1, "a":2, "a":3,  "c": 4}     <- 字符不动
-- jb = {"a": 3, "b": 1, "c": 4}            <- a 去重保留最后，键被重排，空白消失
```

### 1.2 实战决策

**99% 的场景应该用 `jsonb`**，只有以下两个边缘场景考虑 `json`：

- 需要严格保留原始字符（审计 / 签名校验：输入哈希必须等于存储哈希）
- 写多读极少且对解析延迟敏感（罕见）

### 1.3 与 MySQL JSON 的差异

| 维度 | MySQL `JSON` | PG `jsonb` |
|---|---|---|
| 存储格式 | 自定义二进制（含 lookup 表） | TOAST + 自定义二进制 |
| 索引 | 只能通过虚拟生成列 + B-tree | 直接 GIN（jsonb_ops / jsonb_path_ops） |
| 路径语言 | `$.a.b[0]`（JSONPath 弱版本） | `jsonpath`（PG 12+，SQL/JSON 标准） |
| 包含查询 | `JSON_CONTAINS` | `@>`（GIN 加速） |
| 全文配合 | 无原生方案 | 与 `tsvector` 自由组合 |

MySQL 派生用户最容易踩的坑：**MySQL JSON 索引必须借生成列**，而 PG 可以直接 `CREATE INDEX ... ON t USING gin ((data->'tags'))`，免生成列。

### 1.4 TOAST 与 JSONB

一个 JSONB 值如果序列化后超过 `TOAST_TUPLE_THRESHOLD`（默认 2 KB），会被压缩或切片存到 TOAST 表（详见 P02）。注意几个含义：

- **大 JSONB 主键查询会触发额外 TOAST 读** —— 在 EXPLAIN BUFFERS 中体现为 TOAST 表的 read。
- **更新一个 1 MB 的 JSONB**，即使只改了 1 字节，整个 TOAST 值会被重写。
- **JSONB 不支持就地更新单个键**——这是 MVCC 决定的，不是 JSONB 的问题。

经验法则：单个 JSONB 值控制在 100 KB 以内，超过就考虑拆字段或拆表。

---

## 第二章：操作符全家桶

### 2.1 取值类操作符

| 操作符 | 类型 | 含义 | 示例 | 结果 |
|---|---|---|---|---|
| `->` | text/int → jsonb | 取键/索引返回 jsonb | `'{"a":1}'::jsonb -> 'a'` | `1` (jsonb) |
| `->>` | text/int → text | 取键/索引返回 text | `'{"a":1}'::jsonb ->> 'a'` | `'1'` (text) |
| `#>` | text[] → jsonb | 按路径取 jsonb | `'{"a":{"b":2}}'::jsonb #> '{a,b}'` | `2` |
| `#>>` | text[] → text | 按路径取 text | `'{"a":{"b":2}}'::jsonb #>> '{a,b}'` | `'2'` |

记忆口诀：**箭头数量 1 = jsonb，2 = text；带 `#` = 走多层路径**。

```sql
SELECT data->'user'->>'name' AS name,
       data#>>'{user,addr,city}' AS city
FROM events
WHERE data->>'event' = 'order_paid';
```

### 2.2 包含与存在类（GIN 友好）

| 操作符 | 含义 | 示例 |
|---|---|---|
| `@>` | 左边包含右边 | `'{"a":1,"b":2}'::jsonb @> '{"a":1}'` → true |
| `<@` | 左边被右边包含 | `'{"a":1}' <@ '{"a":1,"b":2}'` → true |
| `?` | 是否存在某 key（顶层） | `'{"a":1}'::jsonb ? 'a'` → true |
| `?|` | 是否存在任一 key | `data ?| ARRAY['tag1','tag2']` |
| `?&` | 是否存在所有 key | `data ?& ARRAY['tag1','tag2']` |

**重要陷阱**：`?`、`?|`、`?&` 只检查**顶层**键。深层用 `@>` 或 jsonpath。

```sql
-- 错误：只看顶层
SELECT * FROM users WHERE data ? 'phone';

-- 正确：检查 contact 子对象
SELECT * FROM users WHERE data->'contact' ? 'phone';
```

### 2.3 jsonpath 类（PG 12+）

| 操作符 | 含义 | 示例 |
|---|---|---|
| `@?` | JSON 路径是否匹配（存在性） | `data @? '$.tags[*] ? (@ == "vip")'` |
| `@@` | JSON 路径谓词是否为真 | `data @@ '$.score > 80'` |

`@?` vs `@@` 的区别在文档读起来抽象，记一句话就够：

- **`@?`** 期望路径表达式产生一个**节点序列**，序列非空即真。
- **`@@`** 期望路径表达式本身是一个**布尔谓词**，谓词为真即真。

```sql
-- 等价的两种写法
SELECT * FROM products WHERE specs @? '$.weight ? (@ < 100)';
SELECT * FROM products WHERE specs @@ '$.weight < 100';
```

### 2.4 修改类（返回新 jsonb，不就地）

| 函数 | 用途 |
|---|---|
| `jsonb_set(target, path, new_value, create_if_missing)` | 设置或更新某路径 |
| `jsonb_insert(target, path, new_value, insert_after)` | 插入新元素 |
| `jsonb_strip_nulls(jsonb)` | 移除所有值为 null 的键 |
| `jsonb_pretty(jsonb)` | 漂亮打印（调试用） |
| `jsonb_concat (||)` | 浅合并两个对象/数组 |
| `jsonb - 'key'` | 删除顶层 key |
| `jsonb #- '{a,b}'` | 按路径删除 |

```sql
-- 把所有用户的 status 改为 'active'
UPDATE users SET data = jsonb_set(data, '{status}', '"active"', true);

-- 给 tags 数组追加一个值
UPDATE users SET data = jsonb_set(data, '{tags,-1}',
                                  (data->'tags') || '"new_tag"'::jsonb, true);
```

### 2.5 构造类

| 函数 | 用途 |
|---|---|
| `jsonb_build_object(k1,v1, k2,v2,...)` | 构造对象 |
| `jsonb_build_array(v1, v2, ...)` | 构造数组 |
| `to_jsonb(any)` | 任意值转 jsonb |
| `jsonb_agg(expr)` | 聚合成数组 |
| `jsonb_object_agg(k, v)` | 聚合成对象 |

```sql
-- 把用户订单聚合成一个 JSON 文档
SELECT u.id,
       jsonb_build_object(
         'name', u.name,
         'orders', jsonb_agg(jsonb_build_object('id', o.id, 'amount', o.amount))
       ) AS profile
FROM users u JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

---

## 第三章：jsonpath 与 SQL/JSON 标准

### 3.1 jsonpath 表达式语法

PG 12 引入了 jsonpath 类型，对标 SQL/JSON 标准（ISO/IEC 9075-2:2016）。它比 MySQL 的 `$.a.b` 强大得多。

```
$               -- 根
@               -- 当前节点（filter 内用）
$.a             -- 取键
$["a"]          -- 等价
$.a[0]          -- 数组索引
$.a[*]          -- 所有数组元素
$.**            -- 任意深度递归（"通配树"）
$.a ? (@ > 5)   -- filter
$.a ? (@.b like_regex "^abc")  -- 正则
```

操作符：`==`、`!=`、`<`、`>`、`<=`、`>=`、`&&`、`||`、`!`、`like_regex`、`starts with`、`exists`

### 3.2 三大查询函数

| 函数 | 返回 | 用途 |
|---|---|---|
| `jsonb_path_exists(target, path)` | bool | 是否存在匹配 |
| `jsonb_path_query(target, path)` | setof jsonb | 返回所有匹配节点 |
| `jsonb_path_query_first(target, path)` | jsonb | 返回首个匹配 |
| `jsonb_path_match(target, path)` | bool | 谓词路径的真值 |

```sql
-- 找所有有 vip 标签的用户
SELECT id FROM users
WHERE jsonb_path_exists(data, '$.tags[*] ? (@ == "vip")');

-- 取出每个订单中金额 > 1000 的商品名
SELECT order_id,
       jsonb_path_query(items, '$[*] ? (@.amount > 1000).name') AS name
FROM orders;
```

### 3.3 严格模式 vs 宽松模式

jsonpath 表达式前可加 `strict` 或 `lax`（默认 lax）：

- **lax**：自动适配——把标量当作单元素数组、忽略不存在的键
- **strict**：完全按标准——类型不匹配/路径不存在直接报错

```sql
-- lax：把 1 视为 [1]，依然能匹配
SELECT '{"a": 1}'::jsonb @? 'lax $.a[*]';   -- true

-- strict：1 不是数组，报错（或返回 null）
SELECT '{"a": 1}'::jsonb @? 'strict $.a[*]';  -- false
```

### 3.4 SQL/JSON 标准函数（PG 17+）

PG 16 引入 `IS JSON` 谓词，PG 17 引入完整的 SQL/JSON 查询函数（`JSON_EXISTS / JSON_VALUE / JSON_QUERY` 与 `JSON_TABLE` 同批）：

| 函数 | 返回 | 标准等价 |
|---|---|---|
| `JSON_EXISTS(jsonb, path)` | bool | `jsonb_path_exists` |
| `JSON_VALUE(jsonb, path RETURNING type)` | 标量 | 提取标量值 |
| `JSON_QUERY(jsonb, path RETURNING type)` | jsonb | 提取对象/数组 |
| `expr IS JSON` | bool | 校验是否合法 JSON |
| `expr IS JSON OBJECT/ARRAY/SCALAR` | bool | 校验类型 |

```sql
-- 标准写法，可移植到 Oracle/SQL Server
SELECT JSON_VALUE(data, '$.user.age' RETURNING int) AS age,
       JSON_QUERY(data, '$.tags' WITH ARRAY WRAPPER) AS tags
FROM users
WHERE JSON_EXISTS(data, '$.tags[*] ? (@ == "vip")');
```

ON ERROR / ON EMPTY 子句可控制异常行为：

```sql
SELECT JSON_VALUE(data, '$.age' RETURNING int DEFAULT -1 ON ERROR)
FROM users;
```

### 3.5 JSON_TABLE（PG 17+）——杀手特性

`JSON_TABLE` 把 JSON 文档"炸开"成关系表，是迄今为止 SQL/JSON 最重要的特性，相当于 MySQL 的 `JSON_TABLE` 但语法更标准。

```sql
SELECT t.*
FROM orders,
     JSON_TABLE(items, '$[*]' COLUMNS (
       name TEXT      PATH '$.name',
       price NUMERIC  PATH '$.price',
       qty INT        PATH '$.qty',
       category TEXT  PATH '$.cat' DEFAULT 'unknown' ON EMPTY
     )) AS t
WHERE orders.id = 42;
```

效果：把 `items` 这个 JSON 数组的每个元素映射为关系表的一行，可以直接 GROUP BY、JOIN、聚合。

嵌套数组（NESTED PATH）：

```sql
SELECT *
FROM employees,
     JSON_TABLE(profile, '$' COLUMNS (
       name TEXT PATH '$.name',
       NESTED PATH '$.skills[*]' COLUMNS (
         skill TEXT PATH '$',
         FOR ORDINALITY AS skill_no
       )
     )) AS t;
```

ASCII 图：

```
JSON_TABLE 处理流程
─────────────────────────────────────────
 输入 JSON ───┐
              ↓
   ┌─ rowpath: $[*]  ──→ 产出多行
   │      └─ COLUMNS: 每行抽出多列
   │             ├─ scalar (PATH 'xx')
   │             ├─ FOR ORDINALITY  (行号)
   │             └─ NESTED PATH   ──→ 子展开（笛卡尔）
   └─ DEFAULT ... ON EMPTY/ERROR
              ↓
       关系结果集
```

---

## 第四章：JSONB 索引——GIN 是主角

### 4.1 GIN 索引基础

GIN（Generalized Inverted Index）= 倒排索引。它会把 JSONB 中的每个**键-值对**（或路径）建立倒排，使得 `@>` / `?` / `@?` / `@@` 等查询可以走索引。

```sql
CREATE INDEX idx_users_data ON users USING gin (data);
```

### 4.2 jsonb_ops vs jsonb_path_ops

GIN 有两个内置操作符类：

| 操作符类 | 索引大小 | 支持的操作符 | 适用场景 |
|---|---|---|---|
| `jsonb_ops`（默认） | 大（每个 key 一条 entry，每个 value 一条 entry） | `@>`、`?`、`?|`、`?&`、`@?`、`@@` | 既需要"键存在"又需要"包含"查询 |
| `jsonb_path_ops` | 小（约 30-50% 小） | 仅 `@>`、`@?`、`@@` | 只用 `@>`，索引体积敏感 |

```sql
-- 默认 jsonb_ops
CREATE INDEX idx_a ON t USING gin (data);

-- 显式 jsonb_path_ops（更紧凑、更快，但只支持 @> 系）
CREATE INDEX idx_b ON t USING gin (data jsonb_path_ops);
```

经验法则：
- 业务中 99% 是 `@>` 包含查询 → 用 `jsonb_path_ops`，省空间又快
- 需要混合 `?` / `?|` 等"键存在" → 用 `jsonb_ops`

### 4.3 表达式 GIN——加速"高频路径"

整张 JSONB GIN 索引虽然万能，但如果你只查 `data->'tags'`，没必要给整张索引：

```sql
-- 只对 tags 字段建 GIN
CREATE INDEX idx_users_tags ON users USING gin ((data->'tags'));

-- 查询必须用同样的表达式
SELECT * FROM users WHERE data->'tags' @> '["vip"]';
```

表达式 GIN 通常只有整库 GIN 的 5%-20% 大小，**查询快 5-50 倍**。

### 4.4 单值 B-tree 索引（数值 / 时间）

如果 JSONB 中某个字段是高基数标量（如 `age`、`created_at`），B-tree 表达式索引远胜 GIN：

```sql
CREATE INDEX idx_users_age ON users (((data->>'age')::int));

-- 查询必须类型匹配
SELECT * FROM users WHERE (data->>'age')::int > 18;
```

### 4.5 部分索引 + 表达式索引

```sql
-- 只对 VIP 用户建索引（部分索引）
CREATE INDEX idx_vip_users ON users ((data->>'email'))
WHERE data @> '{"vip": true}';
```

### 4.6 GIN 的劣势

- **更新慢**：每次 UPDATE 都要重建倒排链。高频更新表慎用整张 GIN
- **fastupdate 缓冲**：GIN 默认开启 `fastupdate=on`，把更新缓存到待清理列表，VACUUM 时合并。可能导致**查询时合并**（查询变慢）
  - 关闭：`ALTER INDEX idx_xxx SET (fastupdate = off)`
- **不支持等值/范围**：GIN 不能加速 `data->>'name' = 'alice'`，要用 B-tree 表达式索引

### 4.7 EXPLAIN 看 GIN 是否走

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE data @> '{"tags": ["vip"]}';

-- 良性结果：
-- Bitmap Heap Scan on users
--   Recheck Cond: (data @> '{"tags": ["vip"]}'::jsonb)
--   Heap Blocks: exact=15
--   ->  Bitmap Index Scan on idx_users_data
--         Index Cond: (data @> '{"tags": ["vip"]}'::jsonb)
```

GIN 永远是 Bitmap Index Scan（不是 Index Scan），因为它的结果是无序集合。

---

## 第五章：JSONB 在生产中的常见模式

### 5.1 模式 A：稀疏字段表（用户自定义属性）

```sql
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  category TEXT,
  attrs JSONB     -- 每个品类的属性不同，关系拆表代价大
);

CREATE INDEX idx_products_attrs ON products USING gin (attrs jsonb_path_ops);
```

查询"红色 256GB iPhone"：

```sql
SELECT * FROM products
WHERE category = 'phone'
  AND attrs @> '{"color":"red", "storage":"256GB", "brand":"apple"}';
```

### 5.2 模式 B：事件日志（schemaless）

```sql
CREATE TABLE events (
  id BIGSERIAL,
  ts TIMESTAMPTZ DEFAULT now(),
  event_type TEXT,
  payload JSONB
) PARTITION BY RANGE (ts);
```

按 `payload->>'user_id'` 查询？建表达式 B-tree：

```sql
CREATE INDEX idx_events_user ON events ((payload->>'user_id'));
```

### 5.3 模式 C：动态配置/Feature Flag

```sql
CREATE TABLE flags (
  key TEXT PRIMARY KEY,
  value JSONB,
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 嵌套合并
UPDATE flags SET value = value || '{"max_qps": 200}'::jsonb
WHERE key = 'rate_limit';
```

### 5.4 模式 D：聚合视图（替代物化关系）

把 1:N 关系打包成 JSON 数组，避免业务侧 N+1：

```sql
SELECT u.id,
       jsonb_build_object(
         'profile', row_to_json(u),
         'orders',  COALESCE(jsonb_agg(o) FILTER (WHERE o.id IS NOT NULL), '[]'::jsonb)
       ) AS doc
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

### 5.5 反模式：把 JSONB 当作关系表替代品

- ❌ 强关系（订单-订单项 1:N，每天百万行）硬塞进 JSONB
- ❌ JSONB 中存大文本（>1 MB），导致 TOAST 频繁
- ❌ 每次 UPDATE 改 1 字段但整个 JSONB 1 MB（写放大）
- ❌ 在 JSONB 上做 GROUP BY（数据形状不稳，统计直方图差）

---

## 第六章：全文检索基础

### 6.1 概念三件套

| 类型 | 含义 |
|---|---|
| `tsvector` | 文档的"词袋"——分词、归一化、去停用词后的有序词项集合 |
| `tsquery` | 查询表达式（带 `&`、`|`、`!`、`<->` 等操作符） |
| `regconfig` | 文本配置：语言、字典链、parser |

匹配操作符：`tsvector @@ tsquery`

```sql
SELECT to_tsvector('english', 'The quick brown fox jumps') @@
       to_tsquery('english', 'quick & fox');
-- true

SELECT to_tsvector('english', 'The quick brown fox jumps');
-- 'brown':3 'fox':4 'jump':5 'quick':2     <- 词位置 + 词干化
```

### 6.2 文本配置（ts_config）

```sql
\dF
--                       List of text search configurations
--   Schema   |    Name    |              Description
-- ------------+------------+---------------------------------------
--  pg_catalog | english    | configuration for english language
--  pg_catalog | simple     | simple configuration
--  pg_catalog | russian    | configuration for russian language
--  ...
```

- `simple`：不做词干化、不去停用词，纯小写化 + 分词
- `english`：英文 Snowball 词干 + 停用词
- 各语种都有内置，但**中文需要扩展**

### 6.3 中文方案：zhparser / pg_jieba

PostgreSQL 内置 parser 不能分中文（按空格 + 标点切，中文是连续字符）。两个主流扩展：

| 扩展 | 算法 | 优缺点 |
|---|---|---|
| `zhparser` | SCWS | 老牌，C 写的，性能好；词库不太活跃 |
| `pg_jieba` | jieba | 词库丰富、可加自定义词；性能略慢于 scws |

安装后：

```sql
CREATE EXTENSION zhparser;
CREATE TEXT SEARCH CONFIGURATION chinese (PARSER = zhparser);
ALTER TEXT SEARCH CONFIGURATION chinese
  ADD MAPPING FOR n,v,a,i,e,l,j WITH simple;

SELECT to_tsvector('chinese', '深圳市委市政府公布了最新的人工智能产业规划');
-- '人工智能':4 '产业':5 '公布':3 '市政府':2 '市委':1 '最新':6 '规划':7
```

### 6.4 to_tsvector / to_tsquery / plainto_tsquery / websearch_to_tsquery

| 函数 | 输入要求 | 适用场景 |
|---|---|---|
| `to_tsquery(config, text)` | 严格 tsquery 语法（`a & b | c`） | 程序生成 |
| `plainto_tsquery(config, text)` | 自然语言 → 全部 AND | 简单搜索框 |
| `phraseto_tsquery(config, text)` | 自然语言 → 短语（`a<->b`） | 严格短语 |
| `websearch_to_tsquery(config, text)`（PG 11+） | Google 风格："quoted" -negated OR | 通用搜索框首选 |

```sql
SELECT websearch_to_tsquery('english',
  '"machine learning" -python OR rust');
-- 'machin' <-> 'learn' & !'python' | 'rust'
```

`websearch_to_tsquery` 永远是面向用户输入的最佳选择，可防御非法字符。

---

## 第七章：构建生产级全文检索

### 7.1 表设计

```sql
CREATE TABLE articles (
  id BIGSERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  body  TEXT NOT NULL,
  -- 持久化 tsvector（避免每次查询重算）
  tsv tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('chinese', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('chinese', coalesce(body,'')),  'B')
  ) STORED,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

要点：
- 用**生成列**（PG 12+）让 tsvector 始终自动更新，避免触发器
- `setweight('A'/'B'/'C'/'D')` 为不同字段加权，影响后续 `ts_rank`

PG 18 引入**虚拟生成列**，对 tsvector 这种"每次查询都要存"的场景仍然用 `STORED`。

### 7.2 索引

```sql
CREATE INDEX idx_articles_tsv ON articles USING gin (tsv);
```

### 7.3 查询 + 排名

```sql
WITH q AS (
  SELECT websearch_to_tsquery('chinese', '人工智能 大模型') AS query
)
SELECT a.id, a.title,
       ts_rank_cd(a.tsv, q.query) AS score,
       ts_headline('chinese', a.body, q.query,
                   'StartSel=<b>, StopSel=</b>, MaxFragments=2, MaxWords=20')
         AS snippet
FROM articles a, q
WHERE a.tsv @@ q.query
ORDER BY score DESC
LIMIT 20;
```

### 7.4 GIN vs GiST 选择

| 维度 | GIN | GiST |
|---|---|---|
| 查询速度 | 快 3 倍 | 较慢 |
| 索引体积 | 大 2-3 倍 | 小 |
| 更新速度 | 慢（fastupdate 缓冲后好转） | 快 |
| 适用 | 静态文档（博客/商品） | 频繁更新（IM 消息/评论流） |

经验：**99% 全文检索场景用 GIN**。GiST 在 2026 年已经基本退场（除非有奇异的写比读还重 10 倍的场景）。

### 7.5 ts_rank vs ts_rank_cd

- `ts_rank`：基于词频
- `ts_rank_cd`（Cover Density）：基于词距离（关键词越紧凑分越高）

短语搜索（"machine learning"）应用 `ts_rank_cd`；纯关键词用 `ts_rank` 即可。

### 7.6 ts_headline 高亮

```sql
SELECT ts_headline('chinese', body, query,
       'StartSel=<mark>, StopSel=</mark>, ' ||
       'MaxWords=35, MinWords=15, ShortWord=3, HighlightAll=false, MaxFragments=3')
FROM ...;
```

注意：`ts_headline` 在原始文本上回扫一次（**不走索引**），所以在大文本上很慢，仅对最终 LIMIT 后的几行调用。

### 7.7 自定义词典：同义词、停用词、词形归并

```sql
-- 同义词字典（thesaurus）
CREATE TEXT SEARCH DICTIONARY my_thesaurus (
  TEMPLATE = thesaurus,
  DictFile = mythes,
  Dictionary = english_stem
);

-- 自定义停用词
CREATE TEXT SEARCH DICTIONARY my_stop (
  TEMPLATE = simple,
  STOPWORDS = my_stopwords
);
```

把它们插入到 ts_config 的映射链中即可生效。

---

## 第八章：与 Elasticsearch 的诚实对比

| 维度 | PG 全文检索 | Elasticsearch |
|---|---|---|
| 部署复杂度 | 0（PG 自带） | 独立集群 + JVM 调优 |
| 一致性 | 强一致（事务） | 近实时（refresh_interval） |
| 写吞吐（持续） | 中（GIN 写慢） | 高（Lucene 段合并） |
| 查询能力 | 全套布尔/短语/模糊（带 pg_trgm） | 更丰富（aggregations / function_score） |
| 中文 | zhparser/pg_jieba 可用 | IK/jieba/HanLP 完整生态 |
| 高亮 | ts_headline（够用） | 内置 highlighter（更灵活） |
| 相关性算法 | ts_rank/ts_rank_cd（朴素） | BM25 + TF-IDF + 自定义打分 |
| 聚合分析 | 用 SQL，灵活 | aggs DSL，专业 |
| 数据规模 | 千万到亿（单库） | 十亿到百亿（分布式） |
| 与业务数据 | 同库事务，无需同步 | 需要 CDC/同步组件 |
| 运维成本 | 与 PG 一体 | 独立运维 |

**决策矩阵**：

```
                              查询复杂度
                       简单    │    复杂
              ┌──────────────┴──────────────┐
   数据   小  │  PG 全文检索  │  PG 全文检索  │
   规模      │  (100% 推荐)  │   (够用)     │
          大  │  PG 或 ES    │   ES         │
              │              │  (跨集群分析) │
              └──────────────┴──────────────┘
```

简单粗暴版："数据 < 5000 万 + 普通搜索 → PG；超过这个量级或要做日志分析、地图聚合、function_score → ES"。

---

## 第九章：JSONB + 全文检索的杀手组合

最近三年最火的两个组合模式：

### 9.1 模式 A：商品搜索

```sql
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  attrs JSONB,                  -- 颜色/容量/品牌等动态属性
  name TEXT,
  desc TEXT,
  tsv tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('chinese', name), 'A') ||
    setweight(to_tsvector('chinese', desc), 'B') ||
    setweight(to_tsvector('chinese',
              coalesce(attrs->>'brand',''))    , 'A')
  ) STORED
);

CREATE INDEX idx_p_tsv   ON products USING gin (tsv);
CREATE INDEX idx_p_attrs ON products USING gin (attrs jsonb_path_ops);

-- "红色 iPhone" + 256GB + 价格 < 8000
SELECT * FROM products
WHERE tsv @@ websearch_to_tsquery('chinese', '红色 iPhone')
  AND attrs @> '{"storage":"256GB"}'
  AND (attrs->>'price')::int < 8000
ORDER BY ts_rank_cd(tsv, websearch_to_tsquery('chinese', '红色 iPhone')) DESC
LIMIT 20;
```

### 9.2 模式 B：日志搜索（替代 ELK 的轻量方案）

```sql
CREATE TABLE app_logs (
  ts TIMESTAMPTZ DEFAULT now(),
  level TEXT,
  service TEXT,
  message TEXT,
  ctx JSONB,
  tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', message)) STORED
) PARTITION BY RANGE (ts);

CREATE INDEX idx_logs_tsv ON app_logs USING gin (tsv);
CREATE INDEX idx_logs_ctx ON app_logs USING gin (ctx jsonb_path_ops);

-- 查近 24h 中 service=api、含 "timeout" 的错误
SELECT ts, message FROM app_logs
WHERE ts > now() - interval '24h'
  AND level = 'ERROR'
  AND service = 'api'
  AND tsv @@ to_tsquery('english', 'timeout');
```

配合分区 + BRIN 索引（ts），可以撑住中等规模日志查询。

---

## 第十章：实战 SQL 速查

```sql
-- ============================
-- JSONB 基础
-- ============================

-- 取字段
SELECT data->'user'->>'name' FROM t;

-- 包含查询（GIN 友好）
SELECT * FROM t WHERE data @> '{"tags":["vip"]}';

-- 顶层 key 存在
SELECT * FROM t WHERE data ? 'phone';

-- jsonpath：深层过滤
SELECT * FROM t WHERE data @? '$.orders[*] ? (@.amount > 1000)';

-- SQL/JSON 标准（PG 17+）
SELECT JSON_VALUE(data, '$.user.age' RETURNING int) FROM t;

-- JSON_TABLE 炸开（PG 17+）
SELECT t.*
FROM orders, JSON_TABLE(items, '$[*]' COLUMNS (
  name TEXT PATH '$.name',
  qty INT  PATH '$.qty'
)) AS t;

-- 修改
UPDATE t SET data = jsonb_set(data, '{status}', '"paid"');

-- 删除路径
UPDATE t SET data = data #- '{user, deprecated_field}';

-- 合并
UPDATE t SET data = data || '{"version":2}'::jsonb;

-- ============================
-- 索引
-- ============================

-- 整张 GIN
CREATE INDEX ON t USING gin (data);

-- jsonb_path_ops（更小更快，仅 @>）
CREATE INDEX ON t USING gin (data jsonb_path_ops);

-- 表达式 GIN
CREATE INDEX ON t USING gin ((data->'tags'));

-- 表达式 B-tree
CREATE INDEX ON t (((data->>'age')::int));

-- 部分索引
CREATE INDEX ON t ((data->>'email')) WHERE data @> '{"vip":true}';

-- ============================
-- 全文检索
-- ============================

-- 创建配置 + 索引
ALTER TABLE t ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (to_tsvector('english', body)) STORED;
CREATE INDEX ON t USING gin (tsv);

-- 查询
SELECT * FROM t
WHERE tsv @@ websearch_to_tsquery('english', 'rust OR golang -python')
ORDER BY ts_rank_cd(tsv, websearch_to_tsquery('english', 'rust OR golang -python')) DESC;

-- 高亮
SELECT ts_headline('english', body, query, 'StartSel=<b>,StopSel=</b>') FROM ...;
```

---

## 生产实践

### 实践 1：JSONB Schema 演化

虽然 JSONB 是 schemaless，**业务上仍然要约束 schema**：

- 用 CHECK 约束 + jsonpath：`CHECK (data @? '$.version')`
- 用应用层 JSON Schema 校验（Go: gojsonschema / Rust: jsonschema）
- 版本化每条记录：`data->>'_v'`，演化时分版本读

### 实践 2：JSONB 与 GIN 写性能

如果一张高写入表（>5k QPS）需要 GIN，遵循：

1. 关闭 `fastupdate` 避免查询时被迫合并
2. 定期 `VACUUM` 确保 pending list 清空
3. 考虑用 `jsonb_path_ops`（减小一半索引）
4. 拆出热字段建表达式索引而非整张索引

### 实践 3：全文检索的渐进式演进

不要一开始就上 ES：

1. **阶段 1**（< 100 万行）：`ILIKE '%xxx%'` 配合 pg_trgm 索引
2. **阶段 2**（< 1000 万行）：`tsvector + GIN` 标准全文检索
3. **阶段 3**（< 1 亿行）：分区 + tsvector + websearch_to_tsquery
4. **阶段 4**（> 1 亿行 / 复杂相关性）：考虑 ES，PG 仍作主存

### 实践 4：jsonpath 缓存

PG 12+ 编译后的 jsonpath 会被缓存，但每次写不同字面量都会重建。
对高频查询用 prepared statement 让 jsonpath 表达式不变：

```sql
PREPARE q(int) AS
SELECT * FROM users WHERE data @? format('$.age ? (@ > %s)', $1)::jsonpath;
```

更好的方式是用 `@@` 配合参数：

```sql
SELECT * FROM users WHERE jsonb_path_exists(data, '$.age ? (@ > $threshold)',
                                            jsonb_build_object('threshold', 18));
```

### 实践 5：中文全文检索的"准确率 vs 召回率"

- 商品搜索：高召回率优先 → 用 `simple` 词典 + n-gram parser
- 文档检索：高准确率优先 → 用 zhparser/pg_jieba + 自定义同义词

### 实践 6：与向量检索（pgvector）的混合检索

2025 年的主流 RAG 检索是 **BM25/全文 + Embedding 向量** 的混合：

```sql
WITH bm25 AS (
  SELECT id, ts_rank_cd(tsv, q) AS score
  FROM docs, websearch_to_tsquery('chinese', '人工智能') AS q
  WHERE tsv @@ q
  ORDER BY score DESC LIMIT 100
),
vec AS (
  SELECT id, 1 - (embedding <=> $1::vector) AS score
  FROM docs
  ORDER BY embedding <=> $1::vector LIMIT 100
)
SELECT id, (COALESCE(bm25.score,0) * 0.4 + COALESCE(vec.score,0) * 0.6) AS hybrid
FROM bm25 FULL OUTER JOIN vec USING (id)
ORDER BY hybrid DESC LIMIT 20;
```

详见 P06 pgvector 章节。

---

## 陷阱清单

| # | 陷阱 | 后果 | 解决 |
|---|---|---|---|
| 1 | 用 `json` 而非 `jsonb` | 每次解析、无 GIN、不能 `@>` | 全部改 `jsonb` |
| 2 | `?` 只看顶层 | 漏匹配深层 key | 用 `@>` 或 `@?` jsonpath |
| 3 | 整张 GIN 拖慢 UPDATE | 写吞吐崩塌 | 改表达式 GIN + 拆字段 |
| 4 | `fastupdate=on` 导致查询时合并 | 偶发慢查询 | 关闭 fastupdate 或定期 VACUUM |
| 5 | JSONB > 1 MB 频繁 UPDATE | TOAST 写放大 | 拆字段或拆表 |
| 6 | 表达式索引与查询表达式不一致 | 不走索引 | 表达式严格匹配（包括类型转换） |
| 7 | `(data->>'age')::int` 没建索引 | 全表扫 | `CREATE INDEX ON t (((data->>'age')::int))` |
| 8 | tsvector 每次查询重算 | CPU 暴涨 | 用生成列持久化 |
| 9 | ts_headline 在大文本上慢 | 查询变慢 | 仅对 LIMIT 后的行调用 |
| 10 | 没有中文 parser 直接 `to_tsvector('simple', '中文')` | 整段当一个词 | 装 zhparser/pg_jieba |
| 11 | `to_tsquery` 接受用户输入 | 语法错误抛异常 | 改用 `websearch_to_tsquery` |
| 12 | GIN 多列分别建 vs 单列 GIN | 多列 GIN 不存在 | GIN 可对 ROW 表达式建（不常用） |
| 13 | 用 `LIKE '%xxx%'` 做全文检索 | 全表扫 | 用 tsvector + GIN，或 pg_trgm |
| 14 | JSONB 数组顺序敏感 | `@>` 不区分顺序 | 需要顺序用 jsonpath 索引数组下标 |
| 15 | `jsonb_path_query` 返回 setof | LIMIT 在子查询外才生效 | 包一层子查询再 LIMIT |

---

## 2026 现状

| 主题 | 2026 状态 |
|---|---|
| JSONB 引擎 | 稳定多年，PG 17 进一步优化大对象更新 |
| jsonpath | 默认开启；执行计划成本估算改进 |
| SQL/JSON 标准 | PG 16 `IS JSON` 谓词 + PG 17 查询函数/JSON_TABLE，已超越 MySQL/SQL Server |
| UUIDv7 + JSONB | PG 18 内置 `uuidv7()`，与 JSONB 文档主键天然契合 |
| 中文 parser | zhparser 仍维护，pg_jieba 增加 PG 17/18 兼容 |
| 全文检索 | 进入"够用"阶段，pgvector 抢走一部分场景（语义检索） |
| 混合检索（BM25 + 向量） | RAG 主流方案，PG 单库可完成 |
| Elasticsearch 8/9 vs PG | 中等规模数据更多选择 PG；超大规模仍是 ES |
| ParadeDB | 基于 PG 的搜索/分析数据库（封装了 BM25 + 列存），2025 兴起 |
| pg_search | ParadeDB 开源的 PG 扩展，把 BM25 带入 PG，更接近 ES 体验 |

---

## 练习题

> 答案见 [QUIZ.md](./QUIZ.md)。每题给出"先思考"提示，再尝试用 psql 验证。

1. **JSONB vs JSON**：以下 SQL 的结果为什么不同？
   ```sql
   SELECT '{"a":1,"a":2}'::json   = '{"a":2}'::json;
   SELECT '{"a":1,"a":2}'::jsonb  = '{"a":2}'::jsonb;
   ```
   提示：考虑 jsonb 的键去重规则与 json 的字符比较。

2. **索引类型选择**：一个 `events(payload jsonb)` 表，写入 5000 QPS，查询主要是 `payload @> '{"type":"click"}'` 和 `payload->>'user_id' = 'xxx'`，写出最优索引方案（可以多个）。提示：表达式 GIN + 表达式 B-tree 组合。

3. **jsonpath**：给定文档 `{"orders":[{"amount":100},{"amount":2000},{"amount":50}]}`，写出 jsonpath 查询所有金额大于 500 的订单金额，并用 `jsonb_path_query` 验证。

4. **JSON_TABLE 改写**：把下面用 `jsonb_array_elements` 的查询改写为 PG 17+ 的 `JSON_TABLE` 风格。
   ```sql
   SELECT id, (e->>'name')::text, (e->>'qty')::int
   FROM orders, jsonb_array_elements(items) e;
   ```

5. **GIN 操作符类**：什么场景应该用 `jsonb_path_ops` 而非默认的 `jsonb_ops`？给出一个"必须用 `jsonb_ops`"的反例。

6. **全文检索权重**：设计一个 articles 表，使得标题命中的相关性权重为正文的 2 倍，使用 `setweight` 实现。

7. **websearch 解析**：手算 `websearch_to_tsquery('english', '"rust performance" -gc OR golang')` 的结果。

8. **混合检索**：用一个 SQL 实现"BM25 召回 100 条 + 向量召回 100 条 + 用 RRF (Reciprocal Rank Fusion) 重排"。提示：用 `ROW_NUMBER()` 算 rank。

---

## 延伸阅读

- 官方文档：[JSON Functions and Operators](https://www.postgresql.org/docs/current/functions-json.html)
- 官方文档：[Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- 官方文档：[SQL/JSON Path Language](https://www.postgresql.org/docs/current/datatype-json.html#DATATYPE-JSONPATH)
- 论文：[Jsonb Indexes in PostgreSQL](https://www.postgresql.org/files/developer/jsonb_index.pdf)
- 博客：[Crunchy Data — Using JSONB in PostgreSQL: How to Effectively Store & Index JSON Data](https://www.crunchydata.com/blog/using-jsonb-in-postgresql-how-to-effectively-store-index-json-data)
- 博客：[pganalyze — Understanding GIN Indexes](https://pganalyze.com/blog/gin-index)
- 博客：[Supabase Blog — Postgres Full-Text Search vs the Rest](https://supabase.com/blog/postgres-full-text-search-vs-the-rest)
- 扩展：[zhparser](https://github.com/amutu/zhparser) / [pg_jieba](https://github.com/jaiminpan/pg_jieba)
- 扩展：[pg_search (ParadeDB)](https://github.com/paradedb/paradedb)——把 BM25 带入 PG
- 工具：[pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html)——三元组相似度（模糊匹配）
- 关联章节：[P05 多类型索引](./05-精通-多类型索引.md) / [P06 pgvector](./06-精通-pgvector-与向量检索.md) / [P22 PG18/17 新特性](./22-精通-PG18-17-新特性.md)
