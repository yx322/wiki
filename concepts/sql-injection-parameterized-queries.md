---
title: SQL 注入与参数化查询
created: 2026-08-20
updated: 2026-08-20
type: concept
tags: [sql-injection, parameterized-query, security, database]
sources: [raw/articles/sql-injection-parameterized-queries.md]
confidence: high
---

# SQL 注入与参数化查询

从 order-analytics 实际代码出发，理解 SQL 注入的原理、占位符的工作机制、以及 AI 生成 SQL 场景下的安全防护策略。

## SQL 注入的原理

### 前提条件

**用户的输入直接拼进 SQL 里**——这是 SQL 注入发生的唯一前提。

### 攻击示例

#### 场景：用户登录

```python
# 字符串拼接（危险）
username = "admin"
password = "' OR '1'='1"

sql = f"SELECT * FROM users WHERE name = '{username}' AND password = '{password}'"
# 最终 SQL：
# SELECT * FROM users WHERE name = 'admin' AND password = '' OR '1'='1'
```

因为 `'1'='1'` 永远为真，OR 条件让整个 WHERE 变成真 → 不用密码就能登录任何账号。

#### 更恶意的攻击

```python
username = "admin'; DROP TABLE users; --"

sql = f"SELECT * FROM users WHERE name = '{username}' AND password = 'whatever'"
# 最终 SQL：
# SELECT * FROM users WHERE name = 'admin'; DROP TABLE users; --' AND password = 'whatever'
#                                          ↑ 语句结束    ↑ 删表       ↑ 注释掉后面
```

直接删除整个 users 表。

## 占位符（参数化查询）

### 工作原理

**占位符 = SQL 结构和数据分开传。**

```python
# 占位符（安全）
username = "admin'; DROP TABLE users; --"
password = "whatever"

cursor.execute(
    "SELECT * FROM users WHERE name = %s AND password = %s",
    (username, password)  # 参数单独传，不拼进 SQL
)
# pymysql 会自动转义：
# SELECT * FROM users WHERE name = 'admin\'; DROP TABLE users; --' AND password = 'whatever'
#                                      ↑ 单引号被转义，变成普通字符
```

### 核心区别

| 方式 | 特点 | 安全性 |
|------|------|--------|
| **拼接** | `f"WHERE name = '{user_input}'"` — 混在一起 | ❌ 危险 |
| **占位符** | `"WHERE name = %s", (user_input,)` — 分开传 | ✅ 安全 |

分开传了，数据库就知道 `%s` 那个位置只能是数据，不可能变成 SQL 指令。

### 不同数据库驱动的占位符

| 数据库驱动 | 占位符 | 示例 |
|-----------|--------|------|
| **pymysql** | `%s` | `cursor.execute("WHERE id = %s", (id,))` |
| **sqlite3** | `?` | `cursor.execute("WHERE id = ?", (id,))` |
| **psycopg2**（PostgreSQL）| `%s` | `cursor.execute("WHERE id = %s", (id,))` |
| **pymssql**（SQL Server）| `%s` | `cursor.execute("WHERE id = %s", (id,))` |

## AI 生成 SQL 的安全防护

### order-analytics 的防护链

```
用户说: "查北京消费超1万的用户"（自然语言，不是 SQL）
    ↓
AI 生成: SELECT name FROM ... WHERE city='北京' AND amount > 10000
    ↓
validate_sql 校验:
    ✅ 以 SELECT 开头
    ✅ 不含 INSERT/UPDATE/DELETE/DROP
    ↓
权限注入: 自动加上 admin_user_id IN (...)
    ↓
发给 Doris 执行（只读）
```

### 为什么不存在 SQL 注入

**SQL 注入的前提是：用户能直接控制 SQL 的某一段内容。**

order-analytics 的场景里：
- 用户说的是"查北京消费超1万的用户" → **自然语言**
- AI 把自然语言转成 SQL → **AI 控制 SQL 内容**
- 用户不会说"帮我执行 `'; DROP TABLE orders; --`" → **用户不碰 SQL**

即使用户真的说"帮我删除所有订单"，AI 会生成 `DELETE FROM orders`，然后被 validate_sql 拦截。

**三层防护缺一不可：用户说人话 → AI 写 SQL → 后端校验只放行 SELECT。**

## validate_sql 安全校验

### 代码实现

```python
_FORBIDDEN_KEYWORDS = re.compile(
    r"\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|TRUNCATE|REPLACE\s+INTO|MERGE|GRANT|REVOKE|EXEC|EXECUTE)\b",
    re.IGNORECASE,
)

def validate_sql(sql: str) -> None:
    stripped = sql.strip().rstrip(";").strip()

    # 第一层：必须以 SELECT/WITH/SHOW 开头
    upper = stripped.upper()
    if not upper.startswith("SELECT") and not upper.startswith("WITH") and not upper.startswith("SHOW"):
        raise_exit(ExitCode.CONFIG_ERROR, "仅允许 SELECT / SHOW 查询")

    # 第二层：不能包含危险关键词
    if _FORBIDDEN_KEYWORDS.search(stripped):
        raise_exit(ExitCode.CONFIG_ERROR, "SQL 包含禁止操作")
```

### DDL 和 DML

**DDL（数据定义语言）**：修改数据库结构的语句

```sql
CREATE TABLE orders (...)    -- 建表
ALTER TABLE orders ADD ...   -- 改表
DROP TABLE orders            -- 删表
TRUNCATE TABLE orders        -- 清空表
```

**DML（数据操作语言）**：修改数据库数据的语句

```sql
INSERT INTO orders VALUES (...)  -- 插入数据
UPDATE orders SET ...            -- 修改数据
DELETE FROM orders WHERE ...     -- 删除数据
```

### 校验效果

| 用户/AI 想执行 | validate_sql 检查结果 |
|--------------|-------------------|
| `SELECT * FROM orders` | ✅ 通过（以 SELECT 开头）|
| `WITH cte AS (...) SELECT ...` | ✅ 通过（以 WITH 开头）|
| `DELETE FROM orders` | ❌ 拦截（不以 SELECT 开头）|
| `SELECT * FROM orders; DROP TABLE users` | ❌ 拦截（包含 DROP 关键词）|
| `INSERT INTO orders VALUES (...)` | ❌ 拦截（包含 INSERT 关键词）|

## order-analytics 代码分析

### 两处字符串拼接

**1. AI 生成 SQL → 直接传给脚本**

```python
@app.command()
def query(
    sql: str = typer.Argument(...),  # AI 生成的完整 SQL 字符串
):
    validate_sql(sql)  # 只检查禁止关键词
    cursor.execute(sql)  # 直接执行，没有参数绑定
```

**2. 权限过滤 → 在 SQL 里插入 IN 条件**

```python
def inject_permission_filter(sql: str, user_ids: list[int]) -> str:
    id_list = ",".join(str(uid) for uid in user_ids)  # 拼接数字
    condition = f"admin_user_id IN ({id_list})"
    sql = sql[:where_pos + 5] + " " + condition + " AND" + sql[where_pos + 5:]
```

### 为什么在这个场景下是安全的

| 风险点 | 防护 | 评估 |
|--------|------|------|
| AI 生成恶意 SQL | `validate_sql` 检查禁止关键词 | ✅ 够用 |
| 用户输入注入 | 用户不直接写 SQL，AI 生成 + 后端校验 | ✅ 够用 |
| 权限绕过 | `inject_permission_filter` 强制注入 IN 条件 | ✅ 够用 |
| SQL 注入到 Doris | Doris 是只读分析库，不是业务主库 | ✅ 风险可控 |

### 什么时候必须用占位符

| 场景 | 输入来源 | 是否必须占位符 |
|------|---------|:---:|
| 用户搜索订单号 | 用户直接输入 `order_sn` | ✅ 必须 |
| 用户输入手机号查客户 | 用户直接输入 | ✅ 必须 |
| 用户输入商品名搜索 | 用户直接输入 | ✅ 必须 |
| AI 生成完整 SQL | AI 生成 + 后端校验 | ❌ 不需要 |
| 后端注入权限条件 | 代码生成的整数列表 | ❌ 不需要 |

### 权限注入为什么安全

```python
def inject_permission_filter(sql, user_ids):
    id_list = ",".join(str(uid) for uid in user_ids)  # 整数列表
    condition = f"admin_user_id IN ({id_list})"
    # 结果：admin_user_id IN (101,102,103)
    # 全是数字，不可能注入 SQL
```

`user_ids` 是从 `resolve_visibility()` 返回的**整数列表**（查数据库算出来的），不是用户输入，所以安全。

如果 `user_ids` 是用户输入的字符串，就会有风险：

```python
# 危险场景：user_ids 来自用户输入
user_ids = ["101); DROP TABLE orders; --"]
id_list = ",".join(str(uid) for uid in user_ids)
# 结果：admin_user_id IN (101); DROP TABLE orders; --)
# 💀 注入了
```

## 参见

- [[order-analytics]] — 订单分析技能的架构设计
- [[nl-to-sql]] — 自然语言转 SQL 的实现策略
- [[database-security]] — 数据库安全最佳实践

^[raw/articles/sql-injection-parameterized-queries.md]
