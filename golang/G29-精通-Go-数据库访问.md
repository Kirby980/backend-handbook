# 精通 Go 数据库访问：database/sql、pgx 与 GORM

> 课程编号：G29
> 路线图来源：[roadmap.sh/golang](https://roadmap.sh/golang) — Database 章节
> 难度：⭐⭐⭐⭐
> 预计阅读时间：55 分钟

---

## 引言：标准库够、生态丰富

```go
db, err := sql.Open("postgres", dsn)
rows, err := db.Query("SELECT id, name FROM users")
```

`database/sql` 是 Go 的统一抽象——通过 driver 适配各种数据库。但它只是"接口层"——真正的连接、解析、事务管理大量细节。本章覆盖 database/sql 的连接池、prepared statement、事务、context；以及 pgx（推荐 PostgreSQL）和 GORM（ORM）的使用与权衡。

---

## 第一章：database/sql 基础

### 1.1 打开

```go
import (
    "database/sql"
    _ "github.com/jackc/pgx/v5/stdlib"
)

db, err := sql.Open("pgx", "postgres://user:pass@host/dbname?sslmode=disable")
if err != nil { return err }
defer db.Close()

if err := db.Ping(); err != nil { return err }
```

`sql.Open` **不真的连接**——只是注册。`Ping()` 触发第一次连接。

### 1.2 单行查询

```go
var name string
err := db.QueryRowContext(ctx, "SELECT name FROM users WHERE id=$1", id).Scan(&name)
if errors.Is(err, sql.ErrNoRows) {
    // 没找到
}
```

`QueryRow` 返回 `*Row`，调用 `Scan` 时返回错误（包括 ErrNoRows）。

### 1.3 多行查询

```go
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE age > $1", 18)
if err != nil { return err }
defer rows.Close()

for rows.Next() {
    var id int
    var name string
    if err := rows.Scan(&id, &name); err != nil { return err }
    // ...
}
if err := rows.Err(); err != nil { return err }
```

**必须** `defer rows.Close()`——否则连接不归池，最终池满阻塞。

**必须** 检查 `rows.Err()`——遍历中可能出错，Next() 返回 false 不区分"读完"还是"出错"。

### 1.4 执行（不返回行）

```go
res, err := db.ExecContext(ctx, "UPDATE users SET name=$1 WHERE id=$2", name, id)
rows, _ := res.RowsAffected()
```

`res.LastInsertId()` 在 PostgreSQL 不支持（用 `RETURNING id`）。MySQL/SQLite 支持。

---

## 第二章：连接池

### 2.1 配置

```go
db.SetMaxOpenConns(20)          // 最大并发连接数
db.SetMaxIdleConns(10)          // 最大空闲连接数
db.SetConnMaxLifetime(time.Hour) // 连接最长存活时间
db.SetConnMaxIdleTime(10*time.Minute)  // 连接最长空闲时间
```

### 2.2 为什么要配

- **MaxOpenConns**：DB 服务器有最大连接限制（PostgreSQL 默认 100，分到多个 app 实例）
- **ConnMaxLifetime**：避免长连接因网络中间设备超时被静默断开
- **ConnMaxIdleTime**：DB 资源（如内存）随空闲连接增长

### 2.3 经验值

- 单个 app 实例：MaxOpenConns = 10-30 起步
- 高并发：根据 DB 配置 + 实例数算
- ConnMaxLifetime：30 min - 1 hour
- ConnMaxIdleTime：5-10 min

监控 `db.Stats()` 看实际使用，调整。

### 2.4 连接泄漏

忘记 close rows / 长事务不 commit 都让连接被占。监控 `Stats().InUse` 和 `WaitCount`。

---

## 第三章：Prepared Statement

### 3.1 单语句重用

```go
stmt, err := db.PrepareContext(ctx, "SELECT name FROM users WHERE id=$1")
defer stmt.Close()

for _, id := range ids {
    var name string
    stmt.QueryRowContext(ctx, id).Scan(&name)
}
```

数据库只解析一次，反复执行。理论上更快、更安全（防 SQL 注入）。

### 3.2 实际情况

Go database/sql 的 Prepare 在连接池下表现复杂——每个连接独立预编译，多个连接要重复 prepare。pgx 直接模式可能更优。

### 3.3 参数化查询不一定要 Prepare

```go
db.QueryContext(ctx, "SELECT name FROM users WHERE id=$1", id)
```

这一行**已经是参数化**——driver 内部处理。安全防注入。Prepare 仅当**同语句执行很多次**时收益明显。

---

## 第四章：事务

### 4.1 开始与提交

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil { return err }
defer tx.Rollback()   // 总是 Rollback，Commit 后 Rollback 是 no-op

if _, err := tx.ExecContext(ctx, "UPDATE..."); err != nil { return err }
if _, err := tx.ExecContext(ctx, "INSERT..."); err != nil { return err }

return tx.Commit()
```

`defer tx.Rollback()` 是惯用模式——如果中途 return（错误），自动回滚；如果到达 Commit，Commit 把状态从 active 改为 committed，后续 Rollback 调用变成 no-op。

### 4.2 隔离级别

```go
tx, err := db.BeginTx(ctx, &sql.TxOptions{
    Isolation: sql.LevelSerializable,
    ReadOnly:  true,
})
```

支持：
- LevelReadUncommitted
- LevelReadCommitted（多数 DB 默认）
- LevelRepeatableRead
- LevelSerializable

读多用 ReadOnly = true 让 DB 优化。

### 4.3 嵌套

database/sql 不支持嵌套事务。需要 savepoint，要么自己 SQL `SAVEPOINT`。

### 4.4 wrapper 函数

```go
func WithTx(ctx context.Context, db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil { return err }
    defer tx.Rollback()
    if err := fn(tx); err != nil { return err }
    return tx.Commit()
}

WithTx(ctx, db, func(tx *sql.Tx) error {
    if _, err := tx.ExecContext(ctx, "..."); err != nil { return err }
    return nil
})
```

---

## 第五章：context 与超时

### 5.1 总是用带 Context 的 API

```go
// ❌ 老 API，无 ctx
db.Query("SELECT...")

// ✅ 带 ctx
db.QueryContext(ctx, "SELECT...")
```

ctx 取消时 driver 尝试 abort 查询（驱动支持）。

### 5.2 查询超时

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
db.QueryContext(ctx, "...")
```

或在 server 入口设全局超时，让链路自然继承。

### 5.3 Postgres 的 statement_timeout

```sql
SET statement_timeout = 5000;   -- ms
```

DB 侧也有超时。比 ctx 慢且粗糙，但保护后端避免遗漏 ctx 的代码失控。

---

## 第六章：pgx —— PostgreSQL 首选

### 6.1 两种用法

```go
// A: 通过 database/sql
import _ "github.com/jackc/pgx/v5/stdlib"
db, _ := sql.Open("pgx", dsn)

// B: 直接 pgx API（更强大）
import "github.com/jackc/pgx/v5/pgxpool"
pool, _ := pgxpool.New(ctx, dsn)
defer pool.Close()
```

B 选择：
- 自带连接池，无 database/sql 间接成本
- 支持 PostgreSQL 特有类型（hstore、jsonb、array）
- 批量协议（pipeline / copy）
- 更好的 prepared statement 缓存

### 6.2 基本查询

```go
var name string
err := pool.QueryRow(ctx, "SELECT name FROM users WHERE id=$1", id).Scan(&name)
```

### 6.3 批量 COPY

```go
rows := [][]any{
    {1, "Alice"},
    {2, "Bob"},
}
copyCount, err := pool.CopyFrom(ctx,
    pgx.Identifier{"users"},
    []string{"id", "name"},
    pgx.CopyFromRows(rows),
)
```

COPY 是 PostgreSQL 最快的批量插入方式——10x faster than INSERT。

### 6.4 pgtype

复杂类型有专门的 Go 类型：

```go
var jsonb pgtype.JSONB
err := pool.QueryRow(ctx, "SELECT data FROM events WHERE id=$1", id).Scan(&jsonb)
```

### 6.5 Go 1.24+ 的 `database/sql.Null[T]` —— 终结 `NullString` 系列样板

Go 1.24 把 `database/sql` 里那一堆 `NullString` / `NullInt64` / `NullTime` / `NullBool` 用泛型统一成一个 `sql.Null[T]`：

```go
var name sql.Null[string]
var age  sql.Null[int64]
err := row.Scan(&name, &age)
if name.Valid { use(name.V) }
```

旧 `sql.NullString` 仍然兼容；新代码用泛型版本短得多。配合 PostgreSQL 18 的 `uuidv7()` 主键也很顺：

```go
var id sql.Null[uuid.UUID]
```

### 6.6 `omitzero` JSON tag（Go 1.24+）

把数据库行序列化为 JSON 时，常希望"零值字段不出现在 JSON"。旧 `omitempty` 在零值与"显式设为零"间无法区分；Go 1.24 加了 `omitzero`：

```go
type User struct {
    Name      string    `json:"name"`
    Age       int       `json:"age,omitzero"`       // 0 不输出
    UpdatedAt time.Time `json:"updated_at,omitzero"`// 零时间不输出
}
```

`omitzero` 的判定基于"该类型的零值"，对 struct / time.Time 也生效（`omitempty` 不生效）。

> 参考：[Go 1.24 release notes — database/sql、encoding/json](https://go.dev/doc/go1.24)。

---

## 第七章：sqlc —— 类型安全代码生成

### 7.1 把 SQL 当真理

```sql
-- queries.sql
-- name: GetUser :one
SELECT * FROM users WHERE id = $1;
```

```bash
sqlc generate
```

生成：

```go
func (q *Queries) GetUser(ctx context.Context, id int32) (User, error) { ... }
```

类型安全 + 编译期 SQL 检查 + 性能与手写 SQL 持平。

### 7.2 优点

- IDE 自动补全字段
- 重命名表/字段编译错
- 不写 reflect-based ORM 慢代码
- 完整保留 SQL 能力（CTE、window、复杂 JOIN）

### 7.3 适合

- 业务复杂度中等
- 团队 SQL 熟练
- 性能敏感

不适合：模型频繁变 + 不愿意维护 SQL 的团队（用 ORM 更舒服）。

---

## 第八章：GORM —— ORM

### 8.1 基础

```go
import "gorm.io/gorm"
import "gorm.io/driver/postgres"

db, _ := gorm.Open(postgres.Open(dsn), &gorm.Config{})

type User struct {
    ID   uint
    Name string
    Age  int
}

db.AutoMigrate(&User{})

db.Create(&User{Name: "Alice", Age: 30})

var u User
db.First(&u, 1)
db.Where("age > ?", 18).Find(&users)

db.Model(&u).Update("name", "Bob")
db.Delete(&u)
```

### 8.2 优点

- 快速 CRUD
- 自动 migrate
- 关联查询 simple（Preload）
- Hook（BeforeCreate、AfterUpdate）

### 8.3 缺点

- 反射重，热路径慢 5-10x
- 生成的 SQL 难调优
- 复杂查询要回退 raw SQL
- N+1 容易发生（必须 Preload 才避免）

### 8.4 何时用

- 原型快速开发
- CRUD 多于复杂查询
- 团队熟悉 ORM 模式

复杂业务、性能敏感 → sqlc 或 pgx 直接写。

---

## 第九章：N+1 问题

### 9.1 现象

```go
var users []User
db.Find(&users)
for _, u := range users {
    var orders []Order
    db.Where("user_id=?", u.ID).Find(&orders)   // N 次查询
}
```

100 个用户 → 1 + 100 次 query。慢得离谱。

### 9.2 解决方案

```go
// GORM Preload
db.Preload("Orders").Find(&users)

// 手写 SQL
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id

// 或先取所有 user_id，再批量 SELECT
db.Where("user_id IN ?", userIDs).Find(&orders)
```

### 9.3 检测

dev 环境开 query log，发现连续相同 query 就是 N+1。或用 datadog / new relic APM 看 trace。

---

## 第十章：常见生产配置

### 10.1 模板

```go
cfg, _ := pgxpool.ParseConfig(dsn)
cfg.MaxConns = 20
cfg.MinConns = 2
cfg.MaxConnLifetime = time.Hour
cfg.MaxConnIdleTime = 10 * time.Minute
cfg.HealthCheckPeriod = time.Minute
pool, _ := pgxpool.NewWithConfig(ctx, cfg)

// 超时
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()
pool.QueryRow(ctx, "...", id).Scan(...)

// 监控
go monitorPool(pool)
```

### 10.2 监控

```go
func monitorPool(pool *pgxpool.Pool) {
    for {
        stat := pool.Stat()
        log.Printf("conns: %d/%d, idle: %d, acquired: %d, waited: %v",
            stat.TotalConns(), stat.MaxConns(),
            stat.IdleConns(), stat.AcquiredConns(),
            stat.AcquireDuration())
        time.Sleep(30 * time.Second)
    }
}
```

### 10.3 重试

网络抖动、临时锁冲突——透明重试：

```go
import "github.com/cenkalti/backoff/v4"

backoff.Retry(func() error {
    return op()
}, backoff.NewExponentialBackOff())
```

注意：写操作如果不幂等，重试可能产生重复。用 idempotency key。

---

## 第十一章：生产级最佳实践

1. **永远 defer rows.Close()** + check `rows.Err()`。
2. **每个 DB 调用带 ctx**：超时 + 取消。
3. **连接池配置必须设**：默认值生产不够。
4. **PostgreSQL 用 pgx 直接 API**：性能 + 特性。
5. **sqlc 是 type-safe 首选**：避免 ORM 性能黑洞。
6. **事务用 wrapper + defer Rollback**：避免泄漏。
7. **N+1 必须用 Preload 或 JOIN**：监控 SQL。
8. **批量插入用 COPY** 或 `INSERT ... VALUES (...), (...), ...`。
9. **statement_timeout 在 DB 侧设防御**：避免遗漏 ctx 的代码。
10. **prepared statement 缓存评估**：高频简单查询有用，否则白搭。

---

## 第十二章：常见陷阱清单

### ❌ 陷阱 1：忘记 rows.Close()
```go
rows, _ := db.Query("...")
for rows.Next() { ... }   // 没 close → 连接泄漏
```

### ❌ 陷阱 2：忘记 rows.Err()
Next() 返回 false 可能是出错，看 rows.Err()。

### ❌ 陷阱 3：拼接 SQL 字符串
```go
fmt.Sprintf("SELECT * WHERE id=%s", input)   // SQL 注入
```
永远用 `$1`/`?` 参数化。

### ❌ 陷阱 4：长事务
开了 tx 忘提交 → 持有锁 + 占连接。

### ❌ 陷阱 5：忽略 ErrNoRows
```go
err := db.QueryRow(...).Scan(...)
if err != nil { return err }   // sql.ErrNoRows 也当错误抛
```
区分 not found 与系统错。

### ❌ 陷阱 6：连接池太大压垮 DB
50 个 app 实例 × 100 conn = 5000 连接 → DB 撑不住。

### ❌ 陷阱 7：default sql.Open 后立刻用
没 Ping，第一次查询才发现连不上 → 不易定位。

---

## 第十三章：练习题

**练习 1**：以下代码有何 bug？修复。
```go
rows, _ := db.Query("SELECT ...")
for rows.Next() {
    var x int
    rows.Scan(&x)
}
```

**练习 2**：用 WithTx 写一个"转账"函数：从 from 扣钱、给 to 加钱，原子。

**练习 3**：解释 sql.ErrNoRows 与 sql.ErrConnDone 何时出现。

**练习 4**：N+1 问题给一个 SQL JOIN 修复版（PostgreSQL）。

**练习 5**：解释 ConnMaxLifetime 和 ConnMaxIdleTime 的差异。

---

## 参考答案

**练习 1**：
- 没处理 err
- 没 close rows
- 没 check rows.Err()

修：
```go
rows, err := db.QueryContext(ctx, "...")
if err != nil { return err }
defer rows.Close()
for rows.Next() {
    var x int
    if err := rows.Scan(&x); err != nil { return err }
}
return rows.Err()
```

**练习 2**：
```go
func Transfer(ctx context.Context, db *sql.DB, from, to, amount int) error {
    return WithTx(ctx, db, func(tx *sql.Tx) error {
        if _, err := tx.ExecContext(ctx,
            "UPDATE accounts SET balance = balance - $1 WHERE id=$2 AND balance >= $1",
            amount, from); err != nil { return err }
        if _, err := tx.ExecContext(ctx,
            "UPDATE accounts SET balance = balance + $1 WHERE id=$2",
            amount, to); err != nil { return err }
        return nil
    })
}
```

**练习 3**：
- `sql.ErrNoRows`：QueryRow.Scan 时无行返回
- `sql.ErrConnDone`：连接已经被关闭（比如池已 Close 后还用）

**练习 4**：
```sql
SELECT u.id, u.name, o.id, o.amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.age > 18
ORDER BY u.id;
```
Go 端按 user_id 分组。

**练习 5**：
- ConnMaxLifetime：连接创建后存活总时长，到期重建（无论是否空闲）
- ConnMaxIdleTime：连接空闲多久后关闭（活跃使用时不关）

---

## 小结

| 知识点 | 关键记忆 |
|---|---|
| database/sql | 抽象层；通过 driver |
| QueryRow vs Query | 单行 vs 多行 |
| Close & Err | rows 必关；Err 必查 |
| 连接池 | MaxOpen/Idle/Lifetime 必设 |
| 事务 | defer Rollback；BeginTx |
| context | 所有调用带 ctx |
| pgx | PostgreSQL 首选；直接 API 更快 |
| sqlc | type-safe codegen |
| GORM | 快开发；性能慢；N+1 注意 |

下一篇 **G30 — 精通 Go 结构化日志（slog、Zap、Zerolog）** 将拆开日志库选择、生产级配置、采样、tracing 关联。

---

> 📁 本课程位于 `/data/workspace/dp4/golang/G29-精通-Go-数据库访问.md`
