# 精通 PostgreSQL FDW 与异构集成：跨库查询、外部数据源与现代数据湖

> 关联章节：[P18 扩展生态](./18-精通-扩展生态.md)、[P16 逻辑复制](./16-精通-逻辑复制.md)、[P11 查询调优实战](./11-精通-查询调优实战.md)、[P14 分区表](./14-精通-分区表.md)
>
> Foreign Data Wrapper（FDW，外部数据包装器）是 PostgreSQL 实现 **SQL/MED 标准**（SQL Management of External Data，ISO/IEC 9075-9）的产物。它让你能用一条 SQL 同时查 PG、MySQL、Oracle、MongoDB、Kafka、S3 上的 Parquet 文件——PG 化身一个真正的"查询联邦"。本章把 FDW 的原理、四步建模、主流 wrapper、性能陷阱、与 ETL/CDC 的取舍、以及 2026 年最热的"PG + 数据湖"模式讲透。

---

## 引言：FDW 解决了什么问题

异构系统几乎是大型架构的常态：

- 业务库是 PG，但维度数据在老 MySQL
- 报表系统是 PG + Redshift，需要 join 跨库
- 日志在 CSV / Parquet 文件里，数据科学想直接 SQL
- 多 PG 集群按业务垂直拆分，偶尔需要跨库聚合
- 数据湖在 S3 上，希望统一用 SQL 查询

传统方案：

1. **ETL 同步**（Airflow + dbt + 物化表）：可靠、延迟高、成本高
2. **应用层多源 join**：N+1、事务跨库困难、代码丑
3. **数据虚拟化产品**（Trino / Presto / Denodo）：强大、运维重、贵

FDW 是另一种选择：**在 PG 内**就能挂接外部数据源，对应用透明，查询、JOIN、聚合、写入都可下推到远端。它牺牲了"专业联邦引擎"的极致性能和高级优化，但赢在**部署简单、与现有 PG 工具链兼容**。

读完本章你应该能：

- 用 4 步建模任何 FDW：`CREATE EXTENSION` → `CREATE SERVER` → `CREATE USER MAPPING` → `CREATE FOREIGN TABLE`
- 区分 `postgres_fdw` / `mysql_fdw` / `oracle_fdw` / `mongo_fdw` / `file_fdw` / `duckdb_fdw` 的差异
- 知道哪些操作能 pushdown 到远端（WHERE、JOIN、Aggregate、Sort、LIMIT），哪些不行
- 评估"小表广播 vs 大表传输"的性能陷阱
- 选 FDW vs 物化视图 vs ETL/CDC（Debezium）vs 逻辑复制
- 用 `parquet_fdw` / `duckdb_fdw` 把 PG 接到 S3 数据湖

---

## 第一章：SQL/MED 标准与 FDW 架构

### 1.1 SQL/MED 标准简史

ISO/IEC 9075-9（SQL/MED，2003）规定了"标准 SQL 如何访问外部数据"，定义了以下对象：

- **Foreign Data Wrapper (FDW)**：数据源驱动（如 Oracle 驱动）
- **Foreign Server**：一台外部服务器实例（如生产 Oracle 11.10.0.1:1521）
- **User Mapping**：本地用户 → 远端用户的凭据映射
- **Foreign Table**：远端表在本地的"视图代理"
- **Foreign Schema**：用 `IMPORT FOREIGN SCHEMA` 批量导入

PG 8.4 引入 SQL/MED 基础，PG 9.1 引入第一个 wrapper（`file_fdw`），PG 9.3 引入 `postgres_fdw` 并支持写。**PG 9.6 起 join pushdown、PG 10+ aggregate pushdown、PG 14+ 异步执行**——每个大版本都在补完 FDW 能力。

### 1.2 FDW 工作流（一条 SQL 的旅程）

当你执行 `SELECT * FROM remote_orders WHERE tenant_id = 42`：

```text
┌─────────────────────────────────────────────────────────────┐
│  本地 PG                                                     │
│   ┌──────────┐    1. 解析 SQL                                │
│   │ Parser   │                                              │
│   └────┬─────┘                                              │
│        ▼                                                    │
│   ┌──────────┐    2. Planner 询问 FDW callback             │
│   │ Planner  │       (GetForeignPaths/GetForeignPlan)      │
│   └────┬─────┘       FDW 返回"我可以下推 WHERE"             │
│        ▼                                                    │
│   ┌──────────┐    3. 执行：调用 FDW IterateForeignScan      │
│   │ Executor │       FDW 远端发起: SELECT ... WHERE tenant_id=42
│   └────┬─────┘                                              │
│        ▼                                                    │
│   ┌──────────┐    4. 行返回 → 本地继续 Sort/Aggregate (若没下推) │
│   │ Receiver │                                              │
│   └──────────┘                                              │
└─────────────────────────────────────────────────────────────┘
                       ▼ libpq / mysql client / odbc
              ┌─────────────────┐
              │  远端数据源      │
              └─────────────────┘
```

关键点：**FDW 决定哪些算子能下推**。下推得越多，本地处理越少，性能越好。

---

## 第二章：四步建模 —— 任何 FDW 的通用流程

不论是 postgres_fdw、mysql_fdw 还是 parquet_fdw，建模四步：

### 2.1 第 1 步：安装 wrapper

```sql
CREATE EXTENSION postgres_fdw;
```

### 2.2 第 2 步：定义 SERVER

```sql
CREATE SERVER remote_pg
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host '10.0.1.20', port '5432', dbname 'orders_db');
```

SERVER 是逻辑名，承载连接信息。OPTIONS 的可选项取决于 wrapper。

### 2.3 第 3 步：USER MAPPING

```sql
CREATE USER MAPPING FOR app_user
  SERVER remote_pg
  OPTIONS (user 'reader', password 'secret123');

-- 公共映射（所有本地用户共享）
CREATE USER MAPPING FOR PUBLIC
  SERVER remote_pg
  OPTIONS (user 'guest', password 'guest');
```

USER MAPPING 把"本地角色"映射到"远端凭据"，每个本地用户可有独立映射。**密码在系统目录里以明文存储**——只有 superuser 能 `\des+`。

### 2.4 第 4 步：建外部表

两种方式。

**手工建**：

```sql
CREATE FOREIGN TABLE orders (
  id BIGINT,
  tenant_id BIGINT,
  amount NUMERIC,
  created_at TIMESTAMPTZ
)
SERVER remote_pg
OPTIONS (schema_name 'public', table_name 'orders');
```

**批量导入**（PG 9.5+）：

```sql
CREATE SCHEMA remote_orders;

IMPORT FOREIGN SCHEMA public
  LIMIT TO (orders, customers, items)
  FROM SERVER remote_pg
  INTO remote_orders;

-- 也可以排除
IMPORT FOREIGN SCHEMA public
  EXCEPT (audit_log, big_table)
  FROM SERVER remote_pg
  INTO remote_orders;
```

`IMPORT FOREIGN SCHEMA` 会自动读取远端的列定义并建对应的 FOREIGN TABLE，省去手写。

完事后：

```sql
SELECT * FROM remote_orders.orders WHERE tenant_id = 42 LIMIT 10;
```

---

## 第三章：postgres_fdw —— 最成熟的 FDW

`postgres_fdw` 用 libpq 连远端 PG，特性最完整、性能最好。

### 3.1 完整建模与连接池

```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER pg_orders
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (
    host '10.0.1.20', port '5432', dbname 'orders',
    use_remote_estimate 'true',          -- 重要：让 planner 真去远端 EXPLAIN 拿估算
    fetch_size '10000',                  -- 一次拉多少行（默认 100）
    extensions 'pg_trgm,citext',         -- 远端启用的扩展（让本地知道操作符可下推）
    application_name 'fdw_from_app1'
  );

CREATE USER MAPPING FOR CURRENT_USER
  SERVER pg_orders
  OPTIONS (user 'reader', password 'xxx');

IMPORT FOREIGN SCHEMA public FROM SERVER pg_orders INTO remote;
```

### 3.2 pushdown 能力（PG 18 现状）

| 算子 | 自 PG | 备注 |
|---|---|---|
| WHERE 过滤 | 9.3 | 一开始就有 |
| 列裁剪（SELECT 列） | 9.3 | |
| ORDER BY 排序 | 9.6 | |
| JOIN（同 SERVER） | **9.6** | 必须同 SERVER 的两张外部表 |
| Aggregate（SUM/COUNT/...） | **10** | |
| LIMIT/OFFSET | 12 | |
| INSERT/UPDATE/DELETE | 9.3 / 9.3 / 9.3 | |
| 批量 INSERT | 14 | `batch_size` 选项 |
| TRUNCATE | 14 | |
| 异步执行（多 SERVER 并行） | **14** | `async_capable = on` |
| Partition-wise join + FDW | 11 | |

例子：能不能下推 JOIN？

```sql
EXPLAIN VERBOSE
SELECT o.id, c.name
FROM remote.orders o
JOIN remote.customers c ON o.customer_id = c.id
WHERE o.created_at > now() - interval '1 day';

-- 输出
Foreign Scan
  Remote SQL: SELECT r1.id, r2.name FROM public.orders r1
              JOIN public.customers r2 ON r1.customer_id = r2.id
              WHERE r1.created_at > '2026-05-27...'
```

完美下推，本地只接收最终结果。

但如果两表来自**不同 SERVER**，PG 必须本地 join：

```sql
SELECT o.id, c.name
FROM remote_pg_a.orders o
JOIN remote_pg_b.customers c ON o.customer_id = c.id;
-- → 两次远端拉全表，本地 Hash Join，慢。
```

### 3.3 use_remote_estimate 的威力

默认 `false`，planner 用本地的"默认 row count = 1000"猜远端表大小 → 计划极差。

开 `true` 后，每次 plan 会发送 `EXPLAIN` 到远端真拿估算。代价是 plan 阶段多一次网络往返；收益是 join 顺序、是否下推、用什么 join 算法都能算准。

建议：**生产开启**，配合 prepared statement 复用计划。

### 3.4 异步执行（PG 14+）

老 FDW 跨多 SERVER 的查询是串行的（A 查完查 B）。PG 14+ 引入异步 FDW：

```sql
ALTER SERVER pg_orders OPTIONS (ADD async_capable 'true');
ALTER SERVER pg_users  OPTIONS (ADD async_capable 'true');

EXPLAIN
SELECT * FROM remote_pg_orders.orders
UNION ALL
SELECT * FROM remote_pg_users.events;
-- → Append (async): 两个 ForeignScan 并行
```

对于"分库联合视图"场景（按地区分多 PG 集群），异步带来 N 倍提速。

### 3.5 写入与事务

```sql
INSERT INTO remote.orders (tenant_id, amount) VALUES (1, 99.9);
UPDATE remote.orders SET amount = amount * 1.1 WHERE id = 42;
```

写入也下推（用远端 SQL 执行）。事务方面：

- 远端连接的事务由本地 backend 管理（subtransaction）
- 默认 **不是** XA / 2PC，仅"本地提交时 commit 远端"
- 远端事务失败回滚不可逆 → **跨节点强一致性不保证**

要 2PC，需要设置 `enable_partitionwise_aggregate = on` 之类配合 PG 16+ 的 distributed PREPARE，比较脆弱；强一致跨库写仍建议在应用层用 Saga。

---

## 第四章：mysql_fdw —— 接 MySQL/MariaDB/TiDB

由 EnterpriseDB 维护的 [mysql_fdw](https://github.com/EnterpriseDB/mysql_fdw)，依赖 MySQL C 客户端库。

### 4.1 安装与建模

```bash
# Debian
apt install postgresql-18-mysql-fdw libmariadb-dev
```

```sql
CREATE EXTENSION mysql_fdw;

CREATE SERVER mysql_legacy
  FOREIGN DATA WRAPPER mysql_fdw
  OPTIONS (host '10.0.2.5', port '3306');

CREATE USER MAPPING FOR app_user
  SERVER mysql_legacy
  OPTIONS (username 'reader', password 'xxx');

CREATE FOREIGN TABLE products (
  id BIGINT,
  sku TEXT,
  price NUMERIC(10,2)
) SERVER mysql_legacy
  OPTIONS (dbname 'erp', table_name 'products');
```

### 4.2 pushdown 能力

| 算子 | 支持 |
|---|---|
| WHERE | 是 |
| 列裁剪 | 是 |
| ORDER BY | 是 |
| LIMIT | 是 |
| JOIN（同 SERVER） | 部分（2.5+） |
| Aggregate | 部分（基本聚合） |
| 写入（INSERT/UPDATE/DELETE） | 是 |
| 事务 | MySQL 端按 InnoDB 自动提交 |

### 4.3 类型映射陷阱

| MySQL | PG | 备注 |
|---|---|---|
| INT UNSIGNED | BIGINT | unsigned 在 PG 没原生 |
| DATETIME | TIMESTAMP | 时区丢失 |
| TIMESTAMP | TIMESTAMPTZ | MySQL TS 隐式 UTC，注意时区 |
| TINYINT(1) | BOOLEAN | 部分 wrapper 配置可控 |
| TEXT | TEXT | 大字段慢 |
| BLOB | BYTEA | |
| JSON | JSONB | wrapper 2.x+ |

### 4.4 典型用例

"维度数据在 MySQL，事实数据在 PG"——把维表挂成 FDW，避免双写。

```sql
-- 报表
SELECT p.sku, o.amount
FROM   orders o
JOIN   mysql_products p ON p.id = o.product_id
WHERE  o.created_at > now() - interval '1 day';
```

若 `mysql_products` 较小（<100 万行），可以**用物化视图本地化**（见第八章），避免每次查都跨网。

---

## 第五章：oracle_fdw、tds_fdw、mongo_fdw

### 5.1 oracle_fdw

由 Cybertec 维护，依赖 Oracle Instant Client（OCI）。性能 / pushdown 在第三方 FDW 中最好——专门为 Oracle 优化。

```bash
# 配置 OCI
export ORACLE_HOME=/opt/oracle/instantclient
export LD_LIBRARY_PATH=$ORACLE_HOME:$LD_LIBRARY_PATH
```

```sql
CREATE EXTENSION oracle_fdw;

CREATE SERVER ora
  FOREIGN DATA WRAPPER oracle_fdw
  OPTIONS (dbserver '//10.0.3.5:1521/ORCL');

CREATE USER MAPPING FOR CURRENT_USER
  SERVER ora
  OPTIONS (user 'reader', password 'xxx');

IMPORT FOREIGN SCHEMA "HR" FROM SERVER ora INTO ora_hr;
```

支持 WHERE / JOIN（同 SERVER）/ ORDER BY / LIMIT pushdown，是从 Oracle 渐进迁移到 PG 的常用桥梁。

### 5.2 tds_fdw

接 SQL Server / Sybase（TDS 协议），依赖 FreeTDS。比 oracle_fdw 简单，pushdown 较弱（基本 WHERE）。

```sql
CREATE EXTENSION tds_fdw;
CREATE SERVER mssql_dw
  FOREIGN DATA WRAPPER tds_fdw
  OPTIONS (servername '10.0.4.5', port '1433', database 'DW');
```

### 5.3 mongo_fdw

接 MongoDB（一般用 EDB 出品的版本，支持嵌套文档转 JSONB）。

```sql
CREATE EXTENSION mongo_fdw;
CREATE SERVER mongo
  FOREIGN DATA WRAPPER mongo_fdw
  OPTIONS (address '10.0.5.5', port '27017');

CREATE FOREIGN TABLE events (
  _id NAME,
  doc JSONB
) SERVER mongo
  OPTIONS (database 'logs', collection 'events');
```

性能与 pushdown 普遍弱（Mongo 不是关系模型）。生产更常见做法：用 Debezium / Kafka Connect 把 Mongo CDC 流到 PG，本地查。

---

## 第六章：file_fdw —— 把 CSV / TSV 当成表

PG 自带的 wrapper，把本地文件（CSV/TSV/管道）当 readonly 表查询。

### 6.1 用法

```sql
CREATE EXTENSION file_fdw;

CREATE SERVER fileserv FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE access_log (
  ts TIMESTAMPTZ,
  ip INET,
  method TEXT,
  path TEXT,
  status INT,
  bytes BIGINT
)
SERVER fileserv
OPTIONS (
  filename '/var/log/nginx/access.log.csv',
  format   'csv',
  header   'true',
  delimiter ','
);

SELECT status, count(*) FROM access_log
WHERE ts > now() - interval '1 hour'
GROUP BY status;
```

也可以接一个 shell 命令的输出（program 选项，PG 9.6+）：

```sql
CREATE FOREIGN TABLE current_processes (
  user_name TEXT, pid INT, cpu NUMERIC, mem NUMERIC, cmd TEXT
)
SERVER fileserv
OPTIONS (program 'ps -eo user,pid,%cpu,%mem,comm', format 'csv', header 'true');
```

### 6.2 用例

- 一次性把 CSV 数据用 SQL 探索（不走 COPY，省导入步骤）
- 日志文件即查即用（注意性能：全文件扫，每次查都扫）
- 与 `pg_cron` 配合做"每日加载日志到归档表"

### 6.3 限制

- 只读
- 不支持 pushdown（PG 在本地全文件扫 + 过滤）
- 大文件性能差；超过 GB 就用 COPY 进真实表

---

## 第七章：multicorn —— 用 Python 写 FDW

[Multicorn](https://multicorn.org/)（Multicorn2 是当前主流）让你用 Python 而非 C 写 FDW。开发效率高，性能勉强。

### 7.1 例子：HTTP API 当表

```python
# multicorn_http.py
from multicorn import ForeignDataWrapper
import requests

class GithubFDW(ForeignDataWrapper):
    def __init__(self, options, columns):
        super().__init__(options, columns)
        self.repo = options.get('repo')

    def execute(self, quals, columns):
        url = f'https://api.github.com/repos/{self.repo}/issues'
        for issue in requests.get(url).json():
            yield {
                'number': issue['number'],
                'title':  issue['title'],
                'state':  issue['state'],
            }
```

```sql
CREATE EXTENSION multicorn;
CREATE SERVER gh
  FOREIGN DATA WRAPPER multicorn
  OPTIONS (wrapper 'multicorn_http.GithubFDW');

CREATE FOREIGN TABLE issues (
  number INT, title TEXT, state TEXT
) SERVER gh OPTIONS (repo 'postgres/postgres');

SELECT * FROM issues WHERE state = 'open' LIMIT 10;
```

适合：原型、低 QPS 集成、内部工具。**生产高 QPS 不要用**——Python GIL 限制并发，FDW 调用阻塞 backend。

### 7.2 Wrappers (Supabase 出品)

[Wrappers](https://github.com/supabase/wrappers) 是用 Rust 写的 FDW 框架（基于 pgrx），Supabase 用它接 Stripe / Firebase / S3 / BigQuery / Clickhouse 等。性能比 multicorn 好一个量级。

---

## 第八章：FDW 的性能黑洞 —— 小表广播 vs 大表传输

跨库 JOIN 是 FDW 最大的性能陷阱。考虑：

```sql
SELECT o.id, o.amount, p.name
FROM local_orders o                       -- 1 亿行
JOIN remote_products p ON o.product_id = p.id;  -- 10 万行
```

PG planner 可能选三种执行方式：

1. **拉全表 + 本地 Hash Join**：把 10 万 products 拉到本地建 hash 表 → 1 亿次探测 → OK
2. **逐行 Nested Loop**：1 亿次远端查询 → 灾难
3. **下推 JOIN（同 SERVER 才行）**：把 1 亿 orders 也送过去 → 网络爆炸

判断指标：

- 远端表 < 10 万行 + 用作 join 内表 → **拉过来，做物化视图本地化更好**
- 远端表 > 100 万行 + 同 SERVER → 下推 JOIN
- 跨 SERVER + 都大表 → **不要在 PG 里 join**，考虑 ETL 同步

实战技巧：

```sql
-- 强制把小远端表预拉到 CTE（PG 12+ CTE 不再 inline 时）
WITH p AS MATERIALIZED (
  SELECT id, name FROM remote_products
)
SELECT o.id, o.amount, p.name
FROM local_orders o JOIN p ON o.product_id = p.id;
```

---

## 第九章：物化视图 vs FDW —— 何时落地

物化视图 (Materialized View) 是把"查询结果"存成一张真实表，定期 REFRESH。它和 FDW 的取舍：

| 维度 | FDW（外部表） | Materialized View（本地物化） |
|---|---|---|
| 数据新鲜度 | 实时 | 看刷新周期 |
| 查询延迟 | 跨网，慢 | 本地，快 |
| 写入支持 | postgres_fdw 支持 | 不支持（只读） |
| 复杂查询 | 看 pushdown | 任意 |
| 索引 | 不能直接建 | 可建 |
| 维护 | 无 | 需要定期 REFRESH |

常见组合：用 FDW 拉远端，落到 MV：

```sql
CREATE MATERIALIZED VIEW local_products AS
SELECT id, sku, price FROM remote_products;

CREATE UNIQUE INDEX ON local_products(id);
CREATE INDEX ON local_products(sku);

-- 增量刷新（PG 9.4+）
REFRESH MATERIALIZED VIEW CONCURRENTLY local_products;
```

`CONCURRENTLY` 关键：**不阻塞读**，前提是 MV 上有唯一索引。它通过 diff 算法只更新变化行；代价是慢一些（要扫两遍）。

配合 pg_cron：

```sql
SELECT cron.schedule('refresh-products', '*/10 * * * *',
  $$REFRESH MATERIALIZED VIEW CONCURRENTLY local_products$$);
```

经验法则：

- 远端数据 **每小时更新** → MV 每 5-15 分钟刷
- 远端数据 **实时** + 查询频次低 → 直接 FDW
- 远端数据 **实时** + 查询频次高 → CDC（Debezium）流入本地表

---

## 第十章：FDW vs ETL vs CDC —— 一张决策矩阵

| 方案 | 实时性 | 一致性 | 写入 | 复杂度 | 适合场景 |
|---|---|---|---|---|---|
| **FDW + MV** | 准实时（分钟） | 最终 | 单向 | 低 | 维表同步、报表 |
| **FDW 直连** | 实时 | 强（远端最新） | 双向（postgres_fdw） | 低 | 偶发跨库查询、低 QPS |
| **逻辑复制**（同 PG） | 实时（秒） | 最终 | 单向 | 中 | PG → PG 同步、跨大版本升级 |
| **CDC (Debezium + Kafka)** | 实时（亚秒） | 最终 | 单向 | 高 | 异构源 → 多消费者 |
| **批 ETL (Airflow + dbt)** | 小时/天级 | 强（批次内） | 单向 | 中-高 | 数仓、报表、ML 特征 |
| **数据湖 + Trino** | 取决于湖 | 弱 | 不写回 | 高 | 大规模分析 |

实践口诀：

- 偶尔跨库 + 数据量不大 → FDW 直连
- 周期性同步 + 中等数据量 → FDW + 物化视图 + pg_cron
- 高一致性 + PG → PG → 逻辑复制
- 异构 + 多下游 + 实时 → Debezium + Kafka
- 大规模分析 + 历史归档 → 数据湖 + Trino/DuckDB

---

## 第十一章：异构数据湖 —— PG + S3 + Parquet

2026 年最热的模式：**PG 作为查询入口 + S3 + Parquet 作为存储底层**。两种实现：

### 11.1 parquet_fdw

[parquet_fdw](https://github.com/adjust/parquet_fdw) 直接读 Parquet 文件（本地或 S3）。

```sql
CREATE EXTENSION parquet_fdw;
CREATE SERVER parquet_srv FOREIGN DATA WRAPPER parquet_fdw;

CREATE FOREIGN TABLE events_2025 (
  ts TIMESTAMPTZ, user_id BIGINT, event TEXT, payload JSONB
)
SERVER parquet_srv
OPTIONS (
  filename '/data/lake/events/2025-*.parquet',  -- 通配
  sorted   'ts',                                -- 文件已按 ts 排序，可用于优化
  use_threads 'true'
);

SELECT date_trunc('day', ts), count(*)
FROM events_2025
WHERE ts >= '2025-01-01' AND ts < '2025-02-01'
GROUP BY 1;
```

特点：

- 读 Parquet 列存，**列裁剪 + 谓词下推到 row group level**
- 不支持写
- 不依赖外部进程，C 实现
- 适合 GB - 数 TB 级冷数据

### 11.2 duckdb_fdw / pg_duckdb

[DuckDB](https://duckdb.org/) 是嵌入式列存分析数据库。`duckdb_fdw`（旧版）和 `pg_duckdb`（Hydra + MotherDuck 出品，2024 新）把 DuckDB 嵌进 PG 当执行引擎用。

```sql
CREATE EXTENSION pg_duckdb;

-- 直接查 S3 Parquet（DuckDB 内核扫描）
SELECT * FROM read_parquet('s3://bucket/events/*.parquet')
WHERE ts > '2025-01-01' LIMIT 100;

-- 把 DuckDB 表作为 FOREIGN TABLE
CREATE FOREIGN TABLE lakehouse_events ()
SERVER duckdb_server
OPTIONS (table 'read_parquet(''s3://bucket/events/*.parquet'')');
```

DuckDB 的优势：

- 真正的列存 + 向量化执行 → OLAP 查询比 PG 原生快 10-100x
- 原生支持 Parquet / CSV / Iceberg / Delta Lake
- 在 PG 进程内运行，无网络开销

**这是 2026 年 PG OLAP 的事实路径**：事务表在 PG，分析表在 S3 Parquet，pg_duckdb 桥接。

### 11.3 与 Iceberg / Delta Lake

社区正在做 `iceberg_fdw`、`delta_fdw`，让 PG 直接查 Iceberg/Delta 表。截至 2026-05，**通过 pg_duckdb + DuckDB 的 Iceberg 扩展**是最稳的路径。

---

## 第十二章：案例 1 —— MySQL 维度表 JOIN 进 PG

背景：业务库是 PG（订单），商品主数据在老 MySQL。报表需要 join 商品名。

### 12.1 方案 A：纯 FDW

```sql
CREATE EXTENSION mysql_fdw;

CREATE SERVER mysql_dim
  FOREIGN DATA WRAPPER mysql_fdw
  OPTIONS (host '10.0.2.5', port '3306');

CREATE USER MAPPING FOR app
  SERVER mysql_dim OPTIONS (username 'reader', password 'xxx');

CREATE FOREIGN TABLE mysql_products (
  id BIGINT, sku TEXT, name TEXT, price NUMERIC(10,2)
) SERVER mysql_dim OPTIONS (dbname 'erp', table_name 'products');

SELECT o.id, o.amount, p.name
FROM   orders o
JOIN   mysql_products p ON p.id = o.product_id
WHERE  o.created_at > now() - interval '1 day';
```

问题：每次查都跨 mysqld，QPS 高时打爆 MySQL。

### 12.2 方案 B：物化视图 + pg_cron

```sql
CREATE MATERIALIZED VIEW dim_products AS
SELECT id, sku, name, price FROM mysql_products;

CREATE UNIQUE INDEX ON dim_products(id);

SELECT cron.schedule('refresh-dim-products', '*/15 * * * *',
  $$REFRESH MATERIALIZED VIEW CONCURRENTLY dim_products$$);
```

之后报表只查 `dim_products`（本地），延迟 < 15 分钟可接受。

### 12.3 方案 C：Debezium CDC

如果 MySQL 表写入频繁、要求秒级：用 Debezium → Kafka → Sink Connector → PG `dim_products` 实表。

---

## 第十三章：案例 2 —— file_fdw 直查日志

背景：运维想直接查 Nginx access.log 排查 5xx。

```sql
CREATE EXTENSION file_fdw;
CREATE SERVER logserv FOREIGN DATA WRAPPER file_fdw;

CREATE FOREIGN TABLE access_log (
  ts TIMESTAMPTZ, ip INET, method TEXT,
  path TEXT, status INT, bytes BIGINT, ua TEXT
)
SERVER logserv
OPTIONS (filename '/var/log/nginx/access.csv', format 'csv', header 'true');

-- 立刻可查
SELECT path, count(*) AS errs
FROM access_log
WHERE ts > now() - interval '10 min' AND status >= 500
GROUP BY path
ORDER BY errs DESC
LIMIT 20;

-- 落到本地表归档
CREATE TABLE access_log_archive (LIKE access_log INCLUDING ALL);

INSERT INTO access_log_archive
SELECT * FROM access_log
WHERE ts >= '2026-05-27' AND ts < '2026-05-28';
```

提示：让 Nginx 日志直接输出 CSV 格式（`log_format csv`），省解析。

---

## 第十四章：案例 3 —— 跨 PG 集群读写分离 + 异步

背景：5 个地区的 PG 集群（北京 / 上海 / 广州 / 成都 / 深圳），需要一个"全局视图"做合规报表。

```sql
-- 5 个 SERVER
CREATE SERVER pg_bj FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'pg-bj', port '5432', dbname 'orders', async_capable 'true');
CREATE SERVER pg_sh FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'pg-sh', port '5432', dbname 'orders', async_capable 'true');
-- ... gz / cd / sz

-- 5 张 FOREIGN TABLE
CREATE FOREIGN TABLE orders_bj (LIKE orders) SERVER pg_bj OPTIONS (table_name 'orders');
-- ...

-- 用分区表组装统一视图
CREATE TABLE orders_all (
  id BIGINT, region TEXT, amount NUMERIC, created_at TIMESTAMPTZ
) PARTITION BY LIST (region);

ALTER TABLE orders_all ATTACH PARTITION orders_bj FOR VALUES IN ('bj');
ALTER TABLE orders_all ATTACH PARTITION orders_sh FOR VALUES IN ('sh');
-- ...

-- 跨地区聚合（5 个 FOREIGN SCAN 异步并行）
SELECT region, count(*), sum(amount)
FROM orders_all
WHERE created_at >= now() - interval '1 day'
GROUP BY region;
```

要点：

- `async_capable = true` 让 5 个远端并行
- 分区表 + Append（async）是 PG 14+ 跨库报表标准模式
- 注意 ETL 一致性：跨多个 PG 的 snapshot 不是同一个时间点

---

## 第十五章：FDW 安全与权限

### 15.1 凭据保护

USER MAPPING 中的 password 默认在 `pg_user_mappings` 中明文。要"普通用户也能用，但看不到密码"：

```sql
-- 给用户 SERVER 上的 USAGE 权限
GRANT USAGE ON FOREIGN SERVER pg_orders TO app;

-- 但不要让 app 自己 CREATE USER MAPPING（superuser 操作）
```

PG 16+ 支持 `password_required = false` 给 trusted 角色（如 Kerberos 接力），生产更安全。

### 15.2 行级安全（RLS）

外部表上 PG 的 RLS **不会**自动下推到远端。要在两端都开 RLS，并保证策略一致。

### 15.3 网络与 TLS

`postgres_fdw` 可走 SSL：

```sql
CREATE SERVER ... OPTIONS (sslmode 'require', sslrootcert '/etc/ssl/ca.crt');
```

云上跨 VPC FDW 建议走 PrivateLink / VPC Peering，不走公网。

### 15.4 审计

外部表上的查询在本地 `pg_stat_statements` 里出现；远端按其自身审计。**双端日志要对齐**才能完整追溯一条跨库 SQL。

---

## 生产实践

1. **必开 use_remote_estimate**（postgres_fdw）：否则 planner 估算永远错，复杂 join 必崩。
2. **fetch_size 调大**：默认 100 行/次，跨地域网络下严重拖慢；调到 5000-50000 视带宽。
3. **VIEW 包装 FOREIGN TABLE**：业务层只用本地 view，未来切换实现（FDW → MV → 本地表）时无侵入。
4. **MV 增量化**：用 pg_ivm（incremental view maintenance）或自己用触发器维护差量；REFRESH 全量是大表灾难。
5. **同 SERVER 强制下推**：建模时让需要 join 的远端表都放同一 SERVER（即同一 DB 实例），享受 JOIN/Aggregate pushdown。
6. **Pool 远端连接**：FDW 在每个 backend 第一次访问远端时建一条连接，长生命周期。配合 PgBouncer 远端侧聚合连接。
7. **跨集群事务用 Saga**：FDW 写入跨节点不可信，业务上必须设计补偿。
8. **大对象别 FDW**：BYTEA / TEXT 大字段跨网慢得吓人；专门 COPY。
9. **pg_duckdb 是 OLAP 救命稻草**：报表查询从分钟到秒；尤其与 Parquet/S3 配合。
10. **故障演练**：远端不可达时 FDW 查询会卡 `connect_timeout`（默认 30s），生产上要显式设 5s 并在客户端有降级。

---

## 陷阱清单

- **JOIN 跨 SERVER**：planner 不下推，必全拉本地 → 慢 + OOM 风险。
- **use_remote_estimate 默认 off**：复杂查询计划永远错。
- **fetch_size 太小**：网络带宽利用率 < 5%。
- **远端表元数据变了**：FOREIGN TABLE 不自动同步，要重新 `IMPORT FOREIGN SCHEMA` 或 ALTER。
- **远端没启用扩展**：本地写 `WHERE col % 'xxx'`（pg_trgm 操作符），远端没装 pg_trgm → 操作符无法下推 → 全表扫。
- **MySQL TINYINT(1) 映射混乱**：默认 SMALLINT，老代码可能期望 BOOLEAN，要显式 OPTIONS 控制。
- **MV REFRESH 阻塞**：不加 CONCURRENTLY 会持 AccessExclusive 锁；CONCURRENTLY 又必须有唯一索引。
- **MV CONCURRENTLY 慢**：底层用 diff 算法，大表可能比全量 REFRESH 还慢。
- **file_fdw 程序输出失败**：program 退出非 0 不报错，sql 返回空行集；要在 SQL 外层校验。
- **mongo_fdw 嵌套字段**：默认拉成 JSONB，但深层路径无法下推过滤。
- **multicorn Python 异常**：吞掉异常返回空集，调试痛苦；务必加日志。
- **跨地域 FDW 延迟**：单次 RTT 100ms，扫 1 万次 = 1000s。批量化或本地化。
- **TLS 配置错误**：sslmode 'verify-full' 但没设 sslrootcert → 连接失败但错误信息含糊。
- **DROP FOREIGN TABLE 没清缓存**：长连接 backend 可能仍持有旧定义；reload 失败时让客户端重连。
- **parquet_fdw 文件被删/改**：FOREIGN TABLE 查询时直接报错；用 `parquet_fdw.use_threads = false` 避免部分坏文件影响整批。
- **pg_duckdb 与 PG 类型不全兼容**：DuckDB 的 DECIMAL/INTERVAL 行为细节不同；金融场景仔细 review。
- **FDW 长事务 vacuum**：FDW 写入打开远端事务，本地长事务会让远端的 xmin 也 hold 住，VACUUM 失效。
- **审计盲区**：远端如果不是 PG 或没开审计，跨库查询无完整记录。

---

## 2026 现状

- **postgres_fdw PG 18 进展**：异步执行更稳；新增 `parallel_commit`/`parallel_abort` 选项，分布式提交体验提升。
- **pg_duckdb 1.0**（2025 Q4）：Hydra + MotherDuck 联合发布，与 PG 18 配套。社区把它视作"PG 内 OLAP 引擎"事实标准。
- **Wrappers (Supabase)**：Rust 实现的 FDW 集合，覆盖 Stripe / Firebase / S3 / Snowflake / Clickhouse / Cognito / Logflare 等，**完全替代 multicorn** 的趋势明显。
- **Iceberg / Delta**：通过 pg_duckdb + DuckDB 的对应扩展查询；原生 iceberg_fdw 在路上。
- **CockroachDB / YugabyteDB**：作为分布式 PG-compatible，也实现了 postgres_fdw，可与 PG 互联。
- **CDC 主流**：Debezium 2.x + Kafka Connect 在异构源场景压倒 FDW；FDW 退守"轻量场景"。
- **Neon FDW 桥接**：Neon 提供 "branch" 间的逻辑复制 + FDW，AI agent 用一句 SQL 跨多个分支 DB 调试数据。
- **AI 工作流**：pgvector + FDW + pg_duckdb 组合常出现在 RAG / 特征工程管线（向量在 PG / 大表在 Parquet / 元数据在 MySQL）。

---

## 练习题

1. **四步建模**：用一个非 PG 数据源（如 SQLite / Clickhouse / 一个 HTTP API），写出从 `CREATE EXTENSION` 到 `SELECT` 的完整 SQL 与所需 OS 依赖。
2. **pushdown 验证**：写一个跨 postgres_fdw 的 JOIN，并通过 `EXPLAIN VERBOSE` 看 Remote SQL，验证 JOIN 是否真的下推到远端。
3. **use_remote_estimate 影响**：构造一个 4 表 JOIN，对比 on/off 两种状态下的 plan，描述差异并说明为什么。
4. **决策矩阵**：你的公司有 PG 主库（订单）、MySQL（商品）、MongoDB（点击日志）、S3 Parquet（历史归档）。报表系统要每天看跨域指标。给出每对数据源的接入方案（FDW / MV / CDC / 数据湖）与理由。
5. **物化视图 + 增量**：把一个 100 万行的 FOREIGN TABLE 落到本地 MV，CONCURRENTLY 刷新一次要 5 分钟。给出"做到 30 秒内"的两种思路。
6. **file_fdw 实战**：写一个 SQL，把昨天的 nginx access.log（CSV）找出 TOP 10 错误率路径，按小时分组。
7. **parquet_fdw vs pg_duckdb**：扫 100GB Parquet 文件做 `SELECT user_id, count(*) GROUP BY user_id ORDER BY 2 DESC LIMIT 100`，预期哪个快？给出底层原因。
8. **安全场景**：你要让开发者通过 FDW 查只读副本，但绝不能让他们看到远端密码。设计 SERVER / USER MAPPING / GRANT 方案。

---

## 参考答案

1. **四步建模（以 sqlite_fdw 为例）**：OS 依赖：装 `sqlite_fdw` 扩展及 libsqlite3。SQL：
   ```sql
   CREATE EXTENSION sqlite_fdw;
   CREATE SERVER sqlite_srv FOREIGN DATA WRAPPER sqlite_fdw OPTIONS (database '/data/app.db');
   CREATE FOREIGN TABLE ext_users (id int, name text)
     SERVER sqlite_srv OPTIONS (table 'users');
   SELECT * FROM ext_users;
   ```
   通用四步：CREATE EXTENSION → CREATE SERVER（连接信息）→（需认证时）CREATE USER MAPPING →CREATE FOREIGN TABLE / IMPORT FOREIGN SCHEMA → SELECT。HTTP API 类用 Multicorn2/Wrappers 写 Python/Rust handler。

2. **pushdown 验证**：
   ```sql
   EXPLAIN VERBOSE
   SELECT a.*, b.* FROM remote_a a JOIN remote_b b ON a.id=b.a_id WHERE a.x=1;
   ```
   若两张外表在**同一 server**且 `use_remote_estimate`/版本支持，计划里会出现单个 `Foreign Scan`，其 `Remote SQL:` 行显示完整 `SELECT ... FROM a JOIN b ... WHERE ...`，说明 JOIN 真下推。若计划是两个独立 Foreign Scan + 本地 Hash/Nested Loop Join，则未下推。

3. **use_remote_estimate 影响**：`off` 时本地 planner 用粗略默认估算（猜远端行数/代价），4 表 JOIN 容易选错 join 顺序、少下推、把大量行拉回本地再 join。`on` 时 postgres_fdw 对远端发 `EXPLAIN` 获取真实代价/基数，本地据此选更优计划、更多 join 下推到远端执行。差异根因：`on` 让本地拿到远端真实统计，估算准 → 计划更优，代价是规划期多几次远端往返。

4. **决策矩阵**：
   - PG↔PG（订单，只读副本）：**postgres_fdw**，原生 pushdown 最好。
   - PG↔MySQL（商品，需实时查/JOIN）：mysql_fdw（小量实时）或对热点表用 **CDC（Debezium）落地到 PG** 做物化（量大/高频）。
   - PG↔MongoDB（点击日志，半结构化、海量）：**CDC/批量 ETL** 进数据湖或落 JSONB 表，mongo_fdw 仅做偶发查询。
   - PG↔S3 Parquet（历史归档，分析型大扫）：**pg_duckdb / parquet_fdw / 数据湖引擎**，列存扫描快。
   报表每天跑 → 倾向把多源数据**物化/同步**到一个分析库，避免每次跨域实时拉取。

5. **MV 刷新提速**：`REFRESH MATERIALIZED VIEW CONCURRENTLY` 全量刷 5 分钟。两种思路：(a) **增量刷新**——只拉远端变更（基于时间戳/版本列或 CDC 增量），写入本地基表/用 pg_ivm 等增量物化，避免全量重扫；(b) **分区 + 局部刷新**——按时间分区，仅刷最近变动的分区（或用 pg_cron 调度高频小批刷），把单次工作量从 100 万行降到当日增量。

6. **file_fdw nginx 日志**：
   ```sql
   CREATE FOREIGN TABLE nginx_log (
     ts timestamptz, ip text, method text, path text, status int, bytes int)
   SERVER files OPTIONS (filename '/var/log/nginx/access.csv', format 'csv');

   SELECT date_trunc('hour', ts) AS hr, path,
          count(*) FILTER (WHERE status >= 400)::float / count(*) AS err_rate
   FROM nginx_log
   WHERE ts >= date_trunc('day', now()) - interval '1 day'
     AND ts <  date_trunc('day', now())
   GROUP BY hr, path
   ORDER BY err_rate DESC
   LIMIT 10;
   ```

7. **parquet_fdw vs pg_duckdb**：**pg_duckdb 更快**。底层原因：pg_duckdb 内嵌 DuckDB 的**向量化、列式、并行**执行引擎，GROUP BY/聚合在列存上批量处理；parquet_fdw 仅做 FDW 扫描把行喂回 PG 的行式执行器，聚合走 PG 单线程/有限并行，扫 100GB 时 IO 与执行都吃亏。

8. **安全方案（隐藏远端密码）**：把连接信息放 SERVER（不含密码），密码放在**只有特权角色拥有**的 USER MAPPING 上，开发者用普通角色映射或公共映射但无权查看 mapping 选项：
   ```sql
   CREATE SERVER ro_replica FOREIGN DATA WRAPPER postgres_fdw
     OPTIONS (host 'replica', dbname 'app', port '5432');
   -- 仅 DBA 角色持有真实凭据的 mapping
   CREATE USER MAPPING FOR app_ro SERVER ro_replica
     OPTIONS (user 'ro_user', password '***');
   GRANT USAGE ON FOREIGN SERVER ro_replica TO developer;  -- 仅用，不能改/看密码
   GRANT SELECT ON foreign_table TO developer;
   ```
   非超级用户**无法读取** USER MAPPING 中的 password 选项（`pg_user_mappings` 对非属主隐藏敏感字段），开发者只能查询、看不到密码。

---

## 延伸阅读

- [PostgreSQL 官方手册 - postgres_fdw](https://www.postgresql.org/docs/18/postgres-fdw.html) / [file_fdw](https://www.postgresql.org/docs/18/file-fdw.html)
- [SQL/MED 标准简介](https://wiki.postgresql.org/wiki/SQL/MED)
- [mysql_fdw (EDB)](https://github.com/EnterpriseDB/mysql_fdw) / [oracle_fdw](https://github.com/laurenz/oracle_fdw) / [tds_fdw](https://github.com/tds-fdw/tds_fdw)
- [mongo_fdw](https://github.com/EnterpriseDB/mongo_fdw) / [duckdb_fdw](https://github.com/alitrack/duckdb_fdw) / [pg_duckdb](https://github.com/duckdb/pg_duckdb)
- [Multicorn2](https://github.com/pgsql-io/multicorn2) / [Supabase Wrappers](https://github.com/supabase/wrappers)
- [parquet_fdw](https://github.com/adjust/parquet_fdw)
- [Debezium 官方文档](https://debezium.io/documentation/reference/stable/)
- [DuckDB 文档](https://duckdb.org/docs/)
- 关联章节：[P18 扩展生态](./18-精通-扩展生态.md)、[P16 逻辑复制](./16-精通-逻辑复制.md)、[P11 查询调优实战](./11-精通-查询调优实战.md)、[P14 分区表](./14-精通-分区表.md)、[P22 PG 18/17 新特性](./22-精通-PG18-17-新特性.md)
