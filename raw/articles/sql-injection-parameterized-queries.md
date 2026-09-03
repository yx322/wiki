# SQL 注入与参数化查询：从 order-analytics 实际代码理解

> 日期：2026-08-20
> 参与者：orbit（用户）、AI
> 基于：D:\communicate-skills\skills\order-analytics 实际代码分析

---

## 讨论 1：AI 生成 SQL 后后端做安全校验，就不需要担心 SQL 注入？

> **用户**：意思是 AI 通过用户输入的语言生成 SQL 后会在后端做安全校验，不包含 DDL 和 DML 的才会执行，所以就不需要担心 SQL 注入的风险

对，理解完全正确。

### 完整的防护链

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

你的场景里：
- 用户说的是"查北京消费超1万的用户" → **自然语言**
- AI 把自然语言转成 SQL → **AI 控制 SQL 内容**
- 用户不会说"帮我执行 `'; DROP TABLE orders; --`" → **用户不碰 SQL**

即使用户真的说"帮我删除所有订单"，AI 会生成 `DELETE FROM orders`，然后被 validate_sql 拦截。

### 一句话总结

**用户说人话 → AI 写 SQL → 后端校验只放行 SELECT → 所以安全。** 三层防护缺一不可。

---

## 讨论 2：占位符是什么情况？

> **用户**：那那种占位符的情况是什么，给我详细介绍一下

### 什么时候会有 SQL 注入

**前提条件：用户的输入直接拼进 SQL 里。**

#### 场景：用户登录

用户输入用户名和密码，后端拼 SQL 去数据库查：

```python
# 字符串拼接（危险）
username = "admin"
password = "123456"

sql = f"SELECT * FROM users WHERE name = '{username}' AND password = '{password}'"
# 最终 SQL：
# SELECT * FROM users WHERE name = 'admin' AND password = '123456'
# 没问题，能查到这个用户
```

#### 攻击者输入：

```python
username = "admin"
password = "' OR '1'='1"

sql = f"SELECT * FROM users WHERE name = '{username}' AND password = '{password}'"
# 最终 SQL：
# SELECT * FROM users WHERE name = 'admin' AND password = '' OR '1'='1'
```

因为 `'1'='1'` 永远为真，**OR 条件让整个 WHERE 变成真** → 不用密码就能登录任何账号。

#### 更恶意的：

```python
username = "admin'; DROP TABLE users; --"
password = "whatever"

sql = f"SELECT * FROM users WHERE name = '{username}' AND password = '{password}'"
# 最终 SQL：
# SELECT * FROM users WHERE name = 'admin'; DROP TABLE users; --' AND password = 'whatever'
#                                          ↑ 语句结束    ↑ 删表       ↑ 注释掉后面
# 直接把你的 users 表删了
```

### 占位符怎么解决

```python
# 占位符（安全）
username = "admin'; DROP TABLE users; --"
password = "whatever"

cursor.execute(
    "SELECT * FROM users WHERE name = %s AND password = %s",
    (username, password)  # 参数单独传，不拼进 SQL
)
# pymysql 会自动把参数转义：
# SELECT * FROM users WHERE name = 'admin\'; DROP TABLE users; --' AND password = 'whatever'
#                                      ↑ 单引号被转义了，变成普通字符，不会截断 SQL
```

**核心区别**：
- 拼接：用户输入和 SQL 结构**混在一起**，数据库分不清哪段是指令、哪段是数据
- 占位符：SQL 结构和参数**分开传**，数据库知道 `%s` 的位置只能是数据，不是指令

### 具体对比

```python
# ===== 拼接（危险）=====
user_input = "'; DELETE FROM users; --"
sql = f"SELECT * FROM orders WHERE order_sn = '{user_input}'"
cursor.execute(sql)
# 执行了两条语句：
# 1. SELECT * FROM orders WHERE order_sn = ''
# 2. DELETE FROM users
# 3. --' (注释，忽略)

# ===== 占位符（安全）=====
user_input = "'; DELETE FROM users; --"
cursor.execute("SELECT * FROM orders WHERE order_sn = %s", (user_input,))
# 只执行一条语句：
# SELECT * FROM orders WHERE order_sn = '\'; DELETE FROM users; --'
# 整个恶意输入被当成一个字符串值，不会执行 DELETE
```

### 哪些场景必须用占位符

| 场景 | 输入来源 | 是否必须占位符 |
|------|---------|:---:|
| 用户搜索订单号 | 用户直接输入 `order_sn` | ✅ 必须 |
| 用户输入手机号查客户 | 用户直接输入 | ✅ 必须 |
| 用户输入商品名搜索 | 用户直接输入 | ✅ 必须 |
| AI 生成完整 SQL | AI 生成 + 后端校验 | ❌ 不需要 |
| 后端注入权限条件 | 代码生成的整数列表 | ❌ 不需要 |

### 权限注入为什么安全（order-analytics 代码）

```python
def inject_permission_filter(sql, user_ids):
    id_list = ",".join(str(uid) for uid in user_ids)  # 整数列表
    condition = f"admin_user_id IN ({id_list})"
    # 结果：admin_user_id IN (101,102,103)
    # 全是数字，不可能注入 SQL
```

如果 `user_ids` 是用户输入的字符串呢？

```python
# 危险场景：user_ids 来自用户输入
user_ids = ["101); DROP TABLE orders; --"]
id_list = ",".join(str(uid) for uid in user_ids)
# 结果：admin_user_id IN (101); DROP TABLE orders; --)
# 💀 注入了
```

但 order-analytics 的 `user_ids` 是从 `resolve_visibility()` 返回的**整数列表**（查数据库算出来的），不是用户输入，所以安全。

### 占位符的不同写法

| 数据库驱动 | 占位符 | 示例 |
|-----------|--------|------|
| **pymysql** | `%s` | `cursor.execute("WHERE id = %s", (id,))` |
| **sqlite3** | `?` | `cursor.execute("WHERE id = ?", (id,))` |
| **psycopg2**（PostgreSQL）| `%s` | `cursor.execute("WHERE id = %s", (id,))` |
| **pymssql**（SQL Server）| `%s` | `cursor.execute("WHERE id = %s", (id,))` |

### 一句话总结

**占位符 = SQL 结构和数据分开传。** 拼接是 `f"WHERE name = '{user_input}'"`（混在一起），占位符是 `"WHERE name = %s", (user_input,)`（分开传）。分开传了，数据库就知道 `%s` 那个位置只能是数据，不可能变成 SQL 指令。

---

## 讨论 3：validate_sql 是什么？"挡了 DDL/DML"什么意思？

> **用户**：validate_sql 是什么？validate_sql 挡了 DDL/DML 这句话不理解

### validate_sql 代码

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

### DDL（数据定义语言）

修改数据库**结构**的语句：
```sql
CREATE TABLE orders (...)    -- 建表
ALTER TABLE orders ADD ...   -- 改表
DROP TABLE orders            -- 删表
TRUNCATE TABLE orders        -- 清空表
```

### DML（数据操作语言）

修改数据库**数据**的语句：
```sql
INSERT INTO orders VALUES (...)  -- 插入数据
UPDATE orders SET ...            -- 修改数据
DELETE FROM orders WHERE ...     -- 删除数据
```

### "挡了 DDL/DML"是什么意思

validate_sql 通过**检查关键词**，阻止了这些危险操作：

| 用户/AI 想执行 | validate_sql 检查结果 |
|--------------|-------------------|
| `SELECT * FROM orders` | ✅ 通过（以 SELECT 开头）|
| `WITH cte AS (...) SELECT ...` | ✅ 通过（以 WITH 开头）|
| `DELETE FROM orders` | ❌ 拦截（不以 SELECT 开头）|
| `SELECT * FROM orders; DROP TABLE users` | ❌ 拦截（包含 DROP 关键词）|
| `INSERT INTO orders VALUES (...)` | ❌ 拦截（包含 INSERT 关键词）|

### 为什么这样做

因为 order-analytics 只连 Doris 做**只读查询**，不应该有任何写操作。validate_sql 就是守门员——AI 生成的 SQL 必须先过这一关，才能执行。

### 一句话总结

**DDL/DML = 会改数据库的语句（建表/删表/插入/修改/删除）。validate_sql 通过检查关键词，确保只有 SELECT 查询能通过，其他全拦截。**

---

## 讨论 4：order-analytics 用的是字符串拼接还是占位符？

> **用户**：我代码当中不是占位符，是拼接字符串吗？

是的，代码里确实是字符串拼接，不是占位符。但在这个场景下是安全的。

### 代码里的两处拼接

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

### 为什么在这里是安全的

| 风险点 | 防护 | 评估 |
|--------|------|------|
| AI 生成恶意 SQL | `validate_sql` 检查禁止关键词 | ✅ 够用 |
| 用户输入注入 | 用户不直接写 SQL，AI 生成 + 后端校验 | ✅ 够用 |
| 权限绕过 | `inject_permission_filter` 强制注入 IN 条件 | ✅ 够用 |
| SQL 注入到 Doris | Doris 是只读分析库，不是业务主库 | ✅ 风险可控 |

### 什么时候必须用占位符

```python
# 如果用户输入直接进 SQL → 必须用占位符
cursor.execute("SELECT * FROM orders WHERE order_sn = %s", (user_input,))

# 如果是后端生成的固定值 → 拼接可以
cursor.execute(f"SELECT * FROM orders WHERE admin_user_id IN ({id_list})")
```

### 一句话总结

**你的代码是字符串拼接，但在你这个场景下是安全的**——SQL 是 AI 生成的（有校验），权限条件是后端注入的（整数列表），用户输入不直接进 SQL。
