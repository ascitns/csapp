# MySQL-CRUD

## 零、连接与库操作

```sql
mysql -u root -p       -- 连接
SHOW DATABASES;         -- 列出所有库
CREATE DATABASE mydb;   -- 创建库
DROP DATABASE mydb;     -- 删除库（不可回滚）
USE mydb;               -- 选择当前操作的库
```

| 语句 | 说明 |
|------|------|
| `SHOW DATABASES` | 列出 MySQL 中所有数据库 |
| `CREATE DATABASE 库名` | 创建数据库，可追加 `CHARACTER SET` 和 `COLLATE` |
| `DROP DATABASE 库名` | 删除数据库及其中全部表和数据，不可回滚 |
| `USE 库名` | 切换当前会话的默认数据库，后续 SQL 在该库上执行 |

---

## 一、CREATE

### 1.1 创建表

```sql
CREATE TABLE student (
    id         INT AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(50) NOT NULL,
    gender     ENUM('男','女') DEFAULT '男',
    age        TINYINT UNSIGNED,
    class_id   INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
-- ENGINE：存储引擎
-- CHARSET：字符集
```

`created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`：插入时的默认值是当前时间

```sql
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

- `DEFAULT CURRENT_TIMESTAMP` — 插入时自动填入当前时间
- `ON UPDATE CURRENT_TIMESTAMP` — 只要这行被 UPDATE，该列自动更新为当前时间

| 约束 | 作用 |
|------|------|
| **PRIMARY KEY**（主键） | 唯一标识每行，非空 |
| **AUTO_INCREMENT**（自增） | 自动递增整数 |
| **NOT NULL**（非空） | 禁止 NULL |
| **UNIQUE**（唯一） | 值不可重复 |
| **DEFAULT** | 未指定时的默认值 |
| **FOREIGN KEY**（外键） | 引用另一张表的主键 |
| **CHECK** | 值满足条件才允许写入 |

| 类别 | 数据类型 |
|------|------|
| 整数 | `TINYINT` / `SMALLINT` / `INT` / `BIGINT`， `UNSIGNED` |
| 浮点 | `FLOAT` / `DOUBLE` / `DECIMAL(M,D)`（精确小数） |
| 字符串 | `CHAR(N)`（定长）/ `VARCHAR(N)`（变长） |
| 文本 | `TEXT` / `MEDIUMTEXT` / `LONGTEXT` |
| 日期时间 | `DATE` / `DATETIME` / `TIMESTAMP`（自动转 UTC） |
| 枚举 | `ENUM('值1','值2',...)` |
| JSON | `JSON`，支持索引和路径查询 |

### 1.2 插入数据

```sql
-- 指定列
INSERT INTO student (name, gender, age) VALUES ('张三', '男', 20);
-- 多行
INSERT INTO student (name, gender, age) VALUES('李四', '女', 19), ('王五', '男', 21);
-- 从表导入
INSERT INTO student (name, age) SELECT name, age FROM student_bak;
```

| 语法 | 行为 |
|------|------|
| `INSERT ... VALUES` | 标准插入 |
| `INSERT IGNORE` | 遇唯一键冲突静默跳过 |
| `REPLACE INTO` | 遇冲突删旧行插新行 |
| `INSERT ... ON DUPLICATE KEY UPDATE` | 遇冲突更新已有行 |

```sql
-- 假设表有唯一键 email，已存在一行 email='zhangsan@test.com'

-- 遇冲突跳过
INSERT IGNORE INTO student (name, email) VALUES ('王五', 'zhangsan@test.com');

-- 遇冲突删旧插新（改变id）
REPLACE INTO student (name, email) VALUES ('赵六', 'zhangsan@test.com');

-- 遇冲突更新（id 不变）
INSERT INTO student (name, email) VALUES ('钱七', 'zhangsan@test.com')
ON DUPLICATE KEY UPDATE name = VALUES(name);
```

### 1.3 复制表

```sql
CREATE TABLE t_copy AS SELECT * FROM t;   -- 结构+数据，不复制索引
CREATE TABLE t_empty LIKE t;              -- 仅结构，复制索引
```

---

## 二、SELECT

### 2.1 基本查询

```sql
SELECT * FROM student;                              -- * 表示所有列
SELECT id, name FROM student;                       -- 指定列
SELECT DISTINCT gender FROM student;                -- 去重
SELECT name AS 姓名, age AS 年龄 FROM student;       -- 别名
```

| 关键字 | 说明 |
|--------|------|
| `*` | 返回表中所有列，按建表时的列顺序排列 |
| `DISTINCT` | 对结果集去重，作用于其后的所有列：`SELECT DISTINCT a, b` 对 (a,b) 组合去重 |
| `AS` | 为列或表达式起别名，`AS` 可省略（`SELECT name 姓名 FROM student`） |

### 2.2 WHERE 条件过滤

| 运算符 | 说明 |
|--------|------|
| `=` / `!=` / `<>` | 等于/不等于 |
| `>` / `<` / `>=` / `<=` | 大小比较 |
| `BETWEEN a AND b` | 闭区间 `[a, b]` |
| `IN (...)` | 在集合中 |
| `LIKE` | 模糊匹配，`%` 任意多字符，`_` 单字符 |
| `IS NULL` / `IS NOT NULL` | 空值判断（NULL 不能用 `=` 比较） |
| `AND` / `OR` / `NOT` | 逻辑组合，AND 优先于 OR |
| `REGEXP` | 正则匹配 |

```sql
SELECT * FROM student WHERE name LIKE '张%';
SELECT * FROM student WHERE class_id IN (1, 2) AND age >= 20;
SELECT * FROM student WHERE class_id IS NULL;
```

- WHERE 子句中每个条件都是一个**布尔表达式**，计算结果为 TRUE（1）、FALSE（0）或 NULL。值为 TRUE 的行被保留，FALSE 或 NULL 的行被丢弃。

- 布尔表达式的返回值（0 或 1）可以出现在 SQL 的多个位置，并非只能跟在 WHERE 后面：

```sql
-- SELECT 列表：每行返回 1 或 0
SELECT name, age > 18 AS is_adult FROM student;

-- ORDER BY：利用 0/1 的排序特性控制 NULL 位置
SELECT * FROM student ORDER BY age IS NULL, age ASC;

-- CASE：布尔值决定分支
SELECT name, CASE WHEN age >= 18 THEN '成年' ELSE '未成年' END FROM student;

-- HAVING：对分组聚合结果做判断
SELECT class_id, COUNT(*) AS cnt FROM student GROUP BY class_id HAVING cnt > 3;
```

比较运算符（`>`、`=`、`IS NULL` 等）返回 0 或 1，逻辑运算符（`AND`、`OR`、`NOT`）组合这些 0/1。由于返回值是数字，可以直接参与排序（0 先于 1）和运算（如 `score * (age > 18)`），也可以嵌套在 CASE、IF() 等函数中使用。

### 2.3 排序 ORDER BY

**ORDER BY** 对查询结果集按指定列或表达式排序

| 关键字 | 行为 |
|--------|------|
| `ASC` | 升序（默认） |
| `DESC` | 降序 |

```sql
SELECT * FROM student ORDER BY age ASC;    
SELECT * FROM student ORDER BY age DESC;   -
```

**多列排序**

多个排序列按书写顺序依次比较，前一列值相同时才比较后一列：

```sql
SELECT * FROM student ORDER BY class_id ASC, age DESC;
```

- 先按 `class_id` 升序排列
- `class_id` 相同的行，再按 `age` 降序排列

每列可独立指定 ASC 或 DESC，互不影响。

**表达式排序**

ORDER BY 接受表达式，包括算术运算、函数返回值、CASE 表达式：

```sql
SELECT * FROM student ORDER BY age * 10;          -- 算术表达式
SELECT name, age FROM student ORDER BY age DESC, LENGTH(name);
```

SELECT 中定义的别名可在 ORDER BY 中使用（别名在 ORDER BY 阶段已计算完成）：

```sql
SELECT name, age, age * 10 AS score FROM student ORDER BY score DESC;
```

**CASE 条件排序**：

```sql
SELECT * FROM student
ORDER BY CASE
    WHEN age < 18 THEN 1
    WHEN age BETWEEN 18 AND 25 THEN 2
    ELSE 3
END;
```

**NULL 值排序**

MySQL 中 NULL 被视为最小值：ASC 升序时 NULL 排在最前，DESC 降序时 NULL 排在最后。

| 排序方向 | NULL 位置 |
|----------|-----------|
| `ORDER BY col ASC` | 最前 |
| `ORDER BY col DESC` | 最后 |

显式控制 NULL 位置：

```sql
-- NULL 排最后（ASC 时）
SELECT * FROM student ORDER BY age IS NULL, age ASC;

-- NULL 排最前（DESC 时）
SELECT * FROM student ORDER BY age IS NULL DESC, age DESC;

-- 使用负值取反（仅数值列）
SELECT * FROM student ORDER BY -age DESC;   -- NULL 排最后
```

| 方法 | 原理 | 适用场景 |
|------|------|---------|
| `IS NULL` 作为排序列 | `col IS NULL` 返回 0（非 NULL）或 1（NULL），ASC 排序时 0 在 1 前，因此 NULL 被推到末尾 | 通用，所有数据类型 |
| `COALESCE(col, 默认值)` | 将 NULL 替换为指定值后再排序，NULL 排到默认值所在的位置 | 用指定值替代 NULL 参与排序 |
| 负值取反 | `ORDER BY -col DESC` 中，负值 DESC 等价原值 ASC，但 NULL 不受取反影响留在 DESC 末尾 | 仅数值列，写法简洁但可读性差 |

**中文字段排序**

默认的 utf8mb4_general_ci 排序规则按 Unicode 码点排序，不符合拼音或笔画预期。

**按拼音排序**：

```sql
-- 转换字符集为 GBK，按拼音排序
SELECT * FROM student ORDER BY CONVERT(name USING gbk);

-- 建表时指定排序规则
CREATE TABLE student (
    name VARCHAR(50)
) COLLATE utf8mb4_unicode_ci;
```

`CONVERT(name USING gbk)` 利用 GBK 编码中汉字按拼音排序的特性，是最轻量的方案。

| collation 后缀 | 排序规则 |
|---------------|---------|
| `_general_ci` | 通用，速度优先 |
| `_unicode_ci` | 基于 Unicode 标准，准确但较慢 |
| `_bin` | 按二进制值排序，区分大小写 |

**自定义排序 FIELD()**

`FIELD()` 按指定的值顺序排列，不在列表中的值排到最后：

```sql
-- class_id = 3 排最前，然后 1，然后 2，其余行按默认排序
SELECT * FROM student ORDER BY FIELD(class_id, 3, 1, 2);

-- 与 DESC 组合：不在列表中的值排到最前
SELECT * FROM student ORDER BY FIELD(class_id, 3, 1, 2) DESC;
```

**排序与索引**

ORDER BY 列有匹配索引时，MySQL 按索引顺序直接读取数据，避免额外排序操作（**filesort**）。EXPLAIN 输出中 `Extra` 列出现 `Using filesort` 表示排序未用到索引。

filesort 的行为：

- 排序数据量 ≤ `sort_buffer_size` 时，在内存中完成
- 超出 `sort_buffer_size` 时，使用磁盘临时文件，性能显著下降

复合索引与排序的最左前缀规则：

- `INDEX(a, b, c)` 对 `ORDER BY a` 和 `ORDER BY a, b` 有效
- `INDEX(a, b, c)` 对 `ORDER BY b` 和 `ORDER BY b, c` 无效

MySQL 8.0+ 支持**降序索引（descending index）**，创建时指定列的排序方向：

```sql
ALTER TABLE student ADD INDEX idx_age_desc (age DESC);
```

当 ORDER BY 混合 ASC/DESC 且与索引列的排序方向匹配时，可利用索引避免 filesort。

### 2.4 分页 LIMIT

```sql
SELECT * FROM student LIMIT 10;           -- 前 10 行
SELECT * FROM student LIMIT 5 OFFSET 10;  -- 跳过 10 行取 5 行
```

大偏移量分页性能差，MySQL 仍需扫描并丢弃 OFFSET 之前的全部行。改用基于主键的游标分页：

```sql
SELECT * FROM student WHERE id > 1000 ORDER BY id LIMIT 20;
```

| 分页方式 | 适用场景 | 缺点 |
|---------|---------|------|
| `LIMIT N OFFSET M` | 小偏移量、管理后台 | M 大时扫描行数多 |
| 游标分页 `WHERE id > last_id` | 大表、APP 翻页、流式加载 | 不能跳页，需依赖连续主键 |

### 2.5 聚合函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `COUNT(*)` | 统计行数，包含 NULL 行 | `SELECT COUNT(*) FROM student` |
| `COUNT(col)` | 统计该列非 NULL 的行数 | `SELECT COUNT(age) FROM student` |
| `COUNT(DISTINCT col)` | 统计该列去重后非 NULL 的行数 | `SELECT COUNT(DISTINCT class_id) FROM student` |
| `SUM(col)` | 对列求和，忽略 NULL | `SELECT SUM(age) FROM student` |
| `AVG(col)` | 对列求平均值，忽略 NULL | `SELECT AVG(age) FROM student` |
| `MAX(col)` | 返回列最大值 | `SELECT MAX(age) FROM student` |
| `MIN(col)` | 返回列最小值 | `SELECT MIN(age) FROM student` |
| `GROUP_CONCAT(col)` | 将组内该列的值拼接为逗号分隔的字符串 | `SELECT GROUP_CONCAT(name) FROM student GROUP BY class_id` |

- 除 `COUNT(*)` 外，所有聚合函数忽略 NULL 值
- `SUM` 和 `AVG` 只用于数值类型
- `COUNT(*)` 与 `COUNT(col)` 结果不同：`COUNT(*)` 统计所有行，`COUNT(col)` 只统计 col IS NOT NULL 的行
- `GROUP_CONCAT` 可通过 `SEPARATOR` 指定分隔符：`GROUP_CONCAT(name SEPARATOR ';')`

### 2.6 分组 GROUP BY + HAVING

```sql
SELECT class_id, gender, COUNT(*) AS cnt
FROM student
GROUP BY class_id, gender
HAVING cnt >= 2;
```

WHERE 分组前逐行过滤，HAVING 分组后按聚合结果过滤。WHERE 不可用聚合函数。

### 2.7 子查询

**子查询（subquery）** 是嵌套在另一条 SQL 中的 SELECT 语句

子查询返回的就是一个普通的查询结果集，区别只在于这个结果不展示给用户，而是直接交给外层 SQL 继续使用。

#### 执行方式

子查询分为两种执行模式，取决于子查询是否引用外层列。

**非关联子查询（non-correlated）**：子查询不依赖外层，只执行一次，结果供外层使用

```sql
-- 子查询先算出平均年龄（一个值），外层再逐行与该值比较
SELECT * FROM student WHERE age > (SELECT AVG(age) FROM student);
```

**关联子查询（correlated）**：

```sql
-- 子查询引用了外层 c.id，外层每行都带着不同的 c.id 值让子查询重跑
SELECT * FROM class c
WHERE EXISTS (SELECT 1 FROM student s WHERE s.class_id = c.id);
```

#### 按返回形态分类

子查询的返回内容就是 SELECT 的结果，形态取决于 SELECT 了什么。

| 类型 | 返回形态 | 示例 | 注意事项 |
|------|---------|------|---------|
| **标量子查询** | 单个值（一行一列） | `WHERE age > (SELECT AVG(age) FROM student)` | 返回超过一行报错 `Subquery returns more than 1 row`；返回 0 行则结果为 NULL。可放在 WHERE 比较右侧，也可放在 SELECT 列表中 |
| **列子查询** | 一列多行 | `WHERE age IN (SELECT age FROM student WHERE class_id = 1)` | 配合 `IN` / `NOT IN` / `ANY` / `ALL` 使用。`NOT IN` 遇 NULL 陷阱（见下文） |
| **行子查询** | 一行多列 | `WHERE (col1, col2) = (SELECT MAX(col1), col2 FROM t)` | 较少使用 |
| **表子查询** | 多行多列 | `FROM (SELECT class_id, AVG(age) AS a FROM student GROUP BY class_id) AS t` | 必须提供别名，MySQL 要求 FROM 中的每张表都有名称 |

#### 子查询放在 SELECT 列表中

标量子查询可直接放在 SELECT 列表中，每行计算一次：

```sql
SELECT name, age, (SELECT AVG(age) FROM student) AS avg_age FROM student;
```

#### EXISTS

检查子查询是否有结果返回。有结果返回 TRUE，无结果返回 FALSE。通常为关联子查询，配合外层列做存在性判断。子查询里的 SELECT 列内容无意义，习惯写 `SELECT 1`

```sql
-- 找出有学生的班级
SELECT * FROM class c
WHERE EXISTS (SELECT 1 FROM student s WHERE s.class_id = c.id);

-- NOT EXISTS：找出没有学生的班级
SELECT * FROM class c
WHERE NOT EXISTS (SELECT 1 FROM student s WHERE s.class_id = c.id);
```

- 子查询结果含 NULL 时行为不同。`NOT IN` 遇到 NULL 导致整个条件返回 NULL（视为 FALSE），查询结果可能为空；`NOT EXISTS` 不受 NULL 影响。

```sql
-- col 含 NULL 时 NOT IN 可能返回空集（危险）
SELECT * FROM t1 WHERE col NOT IN (SELECT col FROM t2);

-- NOT EXISTS 不受 NULL 影响（安全）
SELECT * FROM t1 WHERE NOT EXISTS (SELECT 1 FROM t2 WHERE t2.col = t1.col);
```

- 默认用 `NOT EXISTS` 替代 `NOT IN`。

#### ANY / ALL

将外层值与子查询结果集中的每个值逐一比较。`ANY` 等价于 `SOME`。

| 写法 | 等价语义 |
|------|---------|
| `> ANY (子查询)` | 大于子查询结果中至少一个即满足，即大于最小值 |
| `> ALL (子查询)` | 大于子查询结果中所有才满足，即大于最大值 |
| `= ANY (子查询)` | 等价于 `IN` |
| `<> ALL (子查询)` | 等价于 `NOT IN` |

```sql
SELECT * FROM student WHERE age > ANY (SELECT age FROM student WHERE class_id = 1);
SELECT * FROM student WHERE age > ALL (SELECT age FROM student WHERE class_id = 1);
```

### 2.8 多表连接 JOIN

`ON` 指定连接条件，两表中满足条件的行组成结果行。

```sql
SELECT s.name, c.class_name
FROM student s
INNER JOIN class c ON s.class_id = c.id;

SELECT s.name, c.class_name
FROM student s
LEFT JOIN class c ON s.class_id = c.id;
```

| 类型 | 行为 |
|------|------|
| `INNER JOIN` | 仅匹配行，两表交集 |
| `LEFT JOIN`（大多数） | 左表全保留，右表无匹配填 NULL |
| `RIGHT JOIN` | 右表全保留，左表无匹配填 NULL |
| `CROSS JOIN` | 笛卡尔积（无 ON 条件） |

MySQL 不直接支持 `FULL OUTER JOIN`，用 `LEFT JOIN UNION RIGHT JOIN` 模拟。

### 2.9 联合查询 UNION

```sql
SELECT name FROM t1
UNION ALL              -- 不去重，更快
SELECT name FROM t2;
```

`UNION`（去重）vs `UNION ALL`（不去重）——后者无排序去重开销，多数场景优先。

---

## 三、UPDATE

**UPDATE** 修改表中已有行。**SET** 指定要更新的列及新值。

```sql
UPDATE 表名 SET 列1 = 值1, 列2 = 值2 WHERE 条件;
```

### SET 写法

| 写法 | 示例 | 说明 |
|------|------|------|
| 直接赋值 | `SET age = 21` | 设为常量 |
| 引用当前列值 | `SET age = age + 1` | 在现有值上计算 |
| 引用其他列 | `SET a = b, b = a` | 同一条 UPDATE 中列引用取旧值，不会交叉污染 |
| 多列同时更新 | `SET age = 21, class_id = 3` | 逗号分隔 |
| 子查询 | `SET age = (SELECT AVG(age) FROM student)` | 子查询返回值赋给列 |
| CASE 条件更新 | `SET age = CASE WHEN ... THEN ... END` | 按条件赋不同值 |
| 回默认值 | `SET age = DEFAULT` | 设为列定义中 DEFAULT 指定的值 |
| 表达式 | `SET score = score * 1.1 + 5` | 算术运算 |

**SET 中引用列的取值规则**：同一条 UPDATE 中，赋给列的新值在当前语句内不可见，所有列引用取的都是更新前的旧值。

```sql
-- 设 a=1, b=2，执行后 a=2, b=1，而不是 a=2, b=2
UPDATE t SET a = b, b = a;
```

### 基础更新

```sql
UPDATE student SET age = 21 WHERE id = 1;
UPDATE student SET age = age + 1 WHERE class_id = 2;
UPDATE student SET class_id = 4 WHERE age > (SELECT AVG(age) FROM student);
```

### CASE 条件更新

同一列按不同条件赋不同值，用 CASE 一条语句完成，避免多条 UPDATE：

```sql
UPDATE student
SET age = CASE
    WHEN class_id = 1 THEN age + 1
    WHEN class_id = 2 THEN age + 2
    ELSE age
END;
```

### 安全更新

无 WHERE 时 SET 作用于全表。安全操作：先 SELECT 确认范围，或使用事务包裹：

```sql
START TRANSACTION;
UPDATE student SET age = 21 WHERE id = 1;
-- 结果不对则 ROLLBACK
COMMIT;
```

启用安全模式防止无 WHERE 的 UPDATE/DELETE：

```sql
SET sql_safe_updates = 1;
```

---

## 四、DELETE

**DELETE FROM** 删除表中符合条件的行。无 WHERE 时删除全表数据，但表结构保留。

```sql
DELETE FROM student WHERE id = 5;
DELETE FROM student WHERE age > 25;
TRUNCATE TABLE student;   -- 清空表，DDL，不可回滚，重置自增
DELETE FROM student;      -- 清空表，DML，事务内可回滚
```

| | DELETE | TRUNCATE | DROP |
|---|---|---|---|
| 类型 | DML | DDL | DDL |
| 可回滚 | 是（事务内） | 否 | 否 |
| 触发触发器 | 是 | 否 | 否 |
| 重置自增 | 否 | 是 | — |
| WHERE | 支持 | 不支持 | 不支持 |
| 速度 | 逐行，较慢 | 整表回收，快 | 快 |

**多表删除**：

```sql
DELETE s, c FROM student s JOIN class c ON s.class_id = c.id WHERE s.age < 18;
```



---

## 五、导出与导入

mysqldump 将数据库或表导出为 SQL 文件，mysql 执行 SQL 文件完成导入。

```bash
mysqldump -u root -p mydb > mydb.sql                       # 导出库：结构 + 数据
mysqldump -u root -p --no-data mydb > schema.sql           # --no-data 仅结构，不含 INSERT
mysqldump -u root -p --no-create-info mydb > data.sql      # --no-create-info 仅数据，不含 CREATE
mysqldump -u root -p mydb t1 t2 > tables.sql               # 指定表导出
mysql -u root -p mydb < backup.sql                         # 导入 SQL 文件到 mydb
```

| 参数 | 作用 |
|------|------|
| `--no-data` / `-d` | 只导出结构（CREATE TABLE），不导出数据行 |
| `--no-create-info` / `-t` | 只导出数据行（INSERT），不导出结构 |
| `--databases` / `-B` | 导出指定数据库，每条 CREATE DATABASE + USE 都包含 |
| `--all-databases` / `-A` | 导出全部数据库 |
| `--routines` / `-R` | 导出存储过程和函数 |
| `--triggers` | 导出触发器（默认开启） |
| `--single-transaction` | 使用一致性快照导出 InnoDB 表，不锁表 |

---

## 六、SQL 执行顺序

```
FROM → JOIN → WHERE → GROUP BY → SELECT → HAVING → ORDER BY → LIMIT
```

别名在 SELECT 阶段才计算，因此 WHERE 不能用 SELECT 中定义的别名，ORDER BY 可以
