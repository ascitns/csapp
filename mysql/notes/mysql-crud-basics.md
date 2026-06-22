# MySQL 知识笔记

## SQL 通用语法

SQL 语句可以单行或多行书写，以分号结尾。

MySQL 的 SQL 语句不区分大小写，关键字建议使用大写。

注释：

- 单行注释：`-- 注释内容` 或 `# 注释内容`（MySQL 特有）
- 多行注释：`/* 注释内容 */`

## SQL 分类

| 分类 | 全称 | 说明 |
|------|------|------|
| **DDL** | Data Definition Language | 数据定义语言，用来定义数据库对象（数据库、表、字段） |
| **DML** | Data Manipulation Language | 数据操作语言，用来对数据库表中的数据进行增删改 |
| **DQL** | Data Query Language | 数据查询语言，用来查询数据库中表的记录 |
| **DCL** | Data Control Language | 数据控制语言，用来创建数据库用户、控制数据库的访问权限 |

---

## 一、DDL -- 数据定义语言

### 1.1 数据库操作

```sql
mysql -u root -p                              -- 连接
SHOW DATABASES;                                -- 列出所有库
SELECT DATABASE();                             -- 查询当前使用的库
CREATE DATABASE mydb;                          -- 创建库
CREATE DATABASE IF NOT EXISTS mydb             -- 不存在则创建
    DEFAULT CHARSET utf8mb4
    COLLATE utf8mb4_unicode_ci;
DROP DATABASE mydb;                            -- 删除库（不可回滚）
DROP DATABASE IF EXISTS mydb;                  -- 存在则删除
USE mydb;                                      -- 选择当前操作的库
```

| 语句 | 说明 |
|------|------|
| `SHOW DATABASES` | 列出 MySQL 中所有数据库 |
| `SELECT DATABASE()` | 返回当前会话默认数据库名 |
| `CREATE DATABASE 库名` | 创建数据库，可追加 `CHARACTER SET` 和 `COLLATE` |
| `CREATE DATABASE IF NOT EXISTS 库名` | 仅当库不存在时创建 |
| `DROP DATABASE 库名` | 删除数据库及其中全部表和数据，不可回滚 |
| `DROP DATABASE IF EXISTS 库名` | 仅当库存在时删除，避免报错 |
| `USE 库名` | 切换当前会话的默认数据库，后续 SQL 在该库上执行 |

### 1.2 表查询

#### SHOW TABLES

列出当前数据库中的所有表。

```sql
SHOW TABLES;
```

#### DESC

显示表的列定义：字段名、类型、是否可 NULL、键、默认值等。

```sql
DESC 表名;
```

#### SHOW CREATE TABLE

返回该表的完整 CREATE TABLE 语句，含存储引擎和字符集。

```sql
SHOW CREATE TABLE 表名;
```

### 1.3 创建表

```sql
CREATE TABLE student (
    id         INT AUTO_INCREMENT PRIMARY KEY COMMENT '学生ID',
    name       VARCHAR(50) NOT NULL COMMENT '姓名',
    gender     ENUM('男','女') DEFAULT '男' COMMENT '性别',
    age        TINYINT UNSIGNED COMMENT '年龄',
    class_id   INT COMMENT '班级ID',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='学生表';
-- ENGINE：存储引擎
-- CHARSET：字符集
```

#### DEFAULT — 默认值规则

`DEFAULT` 指定列在 INSERT 时未赋值使用的默认值。

| 默认值类型 | 示例 | 说明 |
|-----------|------|------|
| 字面常量 | `DEFAULT 0`、`DEFAULT '男'`、`DEFAULT '2000-01-01'` | 最常用 |
| 表达式（8.0.13+） | `DEFAULT (UUID_TO_BIN(UUID()))` | 用括号包裹，仅限特定函数 |
| 当前时间 | `DEFAULT CURRENT_TIMESTAMP` | 仅 `TIMESTAMP` 和 `DATETIME` 列可用 |
| NULL | 不写 DEFAULT 且允许 NULL 时 | 隐式默认值为 `NULL` |
| 无默认值 | 列定义有 NOT NULL 且无 DEFAULT | INSERT 时必须显式提供值，否则报错 |

**DEFAULT 与 NOT NULL 的关系**：

```sql
-- 未指定值时用 0（不报错）
CREATE TABLE t (col INT DEFAULT 0);

-- 必须提供值，否则报错
CREATE TABLE t (col INT NOT NULL);

-- 未指定值时用 0（NOT NULL + DEFAULT：安全组合）
CREATE TABLE t (col INT NOT NULL DEFAULT 0);
```

**严格模式下的行为**：

```sql
-- 启用严格模式（MySQL 8.0 默认）
SET sql_mode = 'STRICT_TRANS_TABLES';

-- NOT NULL 列无 DEFAULT 且未赋值 → 报错
INSERT INTO t (id) VALUES (1);       -- 缺 col 列值，报错

-- 宽松模式时 → 使用类型的隐式默认值（数值为 0，字符串为 ''）
SET sql_mode = '';                   -- 不推荐
```

**TIMESTAMP / DATETIME 列的特殊规则**：

```sql
-- 1. 声明时自动添加默认值（显式 DEFAULT 不再自动添加，8.0+）
created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;      -- 插入时自动填入当前时间

-- 2. ON UPDATE CURRENT_TIMESTAMP：行被修改时自动更新
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;

-- 3. 第一个 TIMESTAMP 列的特殊行为（explicit_defaults_for_timestamp=OFF 时）
-- 历史上第一个 TIMESTAMP 列即使不写也会自动 DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
-- MySQL 8.0 默认 explicit_defaults_for_timestamp=ON，取消此行为
```

**BLOB、TEXT、GEOMETRY、JSON 列不能有 DEFAULT**，除非 DEFAULT 为 NULL 或不指定（隐式 NULL）。

#### COMMENT

`COMMENT '注释文本'` 为表或列添加注释。表注释支持换行和特殊字符。

```sql
CREATE TABLE student (
    id   INT AUTO_INCREMENT PRIMARY KEY COMMENT '学生ID',
    name VARCHAR(50) NOT NULL COMMENT '姓名，不允许重名则加 UNIQUE'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='学生信息表';
```

- 最后一个字段后面不写逗号
- 列注释不允许换行（单行文本）

#### 约束

| 约束 | 作用 | 特点 |
|------|------|------|
| **PRIMARY KEY**（主键） | 唯一标识每行 | 非空，一张表只能有一个 |
| **AUTO_INCREMENT**（自增） | 自动递增整数 | 只能用于整数列，通常与主键合用。插入 NULL 或 0 触发自增 |
| **NOT NULL**（非空） | 禁止 NULL | 严格模式下无 DEFAULT 且不赋值则报错 |
| **UNIQUE**（唯一） | 值不可重复 | 允许多个 NULL（MySQL 中 NULL 不算重复） |
| **DEFAULT** | INSERT 未赋值时的默认值 | 见上方 DEFAULT 规则 |
| **FOREIGN KEY**（外键） | 引用另一张表的主键 | 保证参照完整性，详见 1.9 外键 |
| **CHECK**（检查） | 值满足条件才允许写入 | MySQL 8.0.16+ 真正生效，此前语法接受但被忽略 |

#### 数据类型

| 类别 | 类型 | 占用空间 | 说明 |
|------|------|---------|------|
| 整数 | `TINYINT` | 1 字节 | -128~127（无符号 0~255） |
| 整数 | `SMALLINT` | 2 字节 | -32768~32767 |
| 整数 | `INT` / `INTEGER` | 4 字节 | 最常用的整数类型 |
| 整数 | `BIGINT` | 8 字节 | 极大值 |
| 定点小数 | `DECIMAL(M,D)` | M+2 字节 | M=总位数(1~65)，D=小数位数(0~30)。存储精确值，适合金额 |
| 浮点 | `FLOAT` / `DOUBLE` | 4/8 字节 | 近似值，不适合精确计算 |
| 定长字符串 | `CHAR(N)` | N 字符 | 0~255。不足补空格，取出时去掉尾部空格。定长，适合身份证号、MD5 等 |
| 变长字符串 | `VARCHAR(N)` | 实际+1/2字节 | 0~65535。仅存储实际字符，需额外 1~2 字节记录长度 |
| 文本 | `TEXT` / `MEDIUMTEXT` / `LONGTEXT` | 64KB/16MB/4GB | 大文本，不能有 DEFAULT（除 NULL） |
| 二进制 | `BLOB` / `MEDIUMBLOB` / `LONGBLOB` | 对应 TEXT 三个级别 | 二进制数据，同样不能有 DEFAULT |
| 日期 | `DATE` | 3 字节 | 1000-01-01 ~ 9999-12-31 |
| 时间 | `TIME` | 3 字节 | -838:59:59 ~ 838:59:59 |
| 日期时间 | `DATETIME` | 5 字节（旧版8） | 1000-01-01 00:00:00 ~ 9999-12-31 23:59:59。存入什么就是什么，不转换时区 |
| 时间戳 | `TIMESTAMP` | 4 字节 | 1970-01-01 00:00:01 ~ 2038-01-19 03:14:07。存入时从当前时区转 UTC，取出转回当前时区 |
| 枚举 | `ENUM('值1','值2',...)` | 1~2 字节 | 最多 65535 个值，内部存储为编号。非法值在严格模式下报错 |
| 集合 | `SET('值1','值2',...)` | 1~8 字节 | 最多 64 个成员，可多选。内部用位存储 |
| JSON | `JSON` | 变长 | 自动校验 JSON 格式，支持索引和路径查询 |
| 布尔 | `BOOL` / `BOOLEAN` | 1 字节 | TINYINT(1) 的别名，0=FALSE，非0=TRUE |

**VARCHAR 与 CHAR 的选取**：

- 值长度固定或接近 → CHAR（如 MD5 加密串、身份证号、UUID）
- 值长度变化大 → VARCHAR（如姓名、邮箱、地址）
- CHAR 读写略快（不需解长度前缀），但浪费空间
- VARCHAR 省空间，需额外 1~2 字节存长度前缀

**TIMESTAMP vs DATETIME 选取**：

- 需要跨时区应用、自动转时区 → TIMESTAMP
- 仅记录"墙上时间"，与时区无关 → DATETIME
- 存储 1970 年前或 2038 年后的日期 → DATETIME
- 需要自动更新时间戳 → TIMESTAMP（支持 ON UPDATE）

### 1.4 修改表 ALTER TABLE

**ALTER TABLE** 用于修改现有表的结构：增删列、修改列类型、重命名、增删约束和索引。

#### 列操作

```sql
-- 添加列（可指定位置）
ALTER TABLE student ADD COLUMN phone VARCHAR(20) COMMENT '电话' AFTER name;
ALTER TABLE student ADD COLUMN email VARCHAR(100) FIRST;                -- 加到第一列

-- 删除列
ALTER TABLE student DROP COLUMN phone;

-- 修改列定义（MODIFY：只改定义，不改名）
ALTER TABLE student MODIFY COLUMN age SMALLINT UNSIGNED NOT NULL COMMENT '年龄';

-- 修改列名 + 定义（CHANGE：改名 + 改定义，两者都需写出）
ALTER TABLE student CHANGE COLUMN age student_age TINYINT UNSIGNED COMMENT '年龄';

-- 重命名表
ALTER TABLE student RENAME TO students;
RENAME TABLE student TO students;                                       -- 另一种写法
```

| 操作 | 子句 | 说明 |
|------|------|------|
| 添加列 | `ADD COLUMN 列名 类型 [约束] [FIRST\|AFTER 已有列]` | 不指定位置时追加到末尾 |
| 删除列 | `DROP COLUMN 列名` | 列中数据一并删除，不可回滚 |
| 修改列定义 | `MODIFY COLUMN 列名 新类型 [新约束]` | 不改列名，只改类型或约束 |
| 改列名+定义 | `CHANGE COLUMN 旧名 新名 新类型 [约束]` | 两者都需完整写出 |
| 重命名表 | `RENAME TO 新表名` | 等价于 `RENAME TABLE 旧名 TO 新名` |

**MODIFY vs CHANGE 对比**：

```sql
-- MODIFY：仅修改类型/约束，列名不变
-- 假设 age 列原本是 INT，现在想改成 TINYINT UNSIGNED NOT NULL
ALTER TABLE student MODIFY COLUMN age TINYINT UNSIGNED NOT NULL;

-- CHANGE：同时修改列名和定义
-- 把 age 列重命名为 student_age，并改动类型
ALTER TABLE student CHANGE COLUMN age student_age TINYINT UNSIGNED NOT NULL COMMENT '学生年龄';
```

> **陷阱**：MODIFY 和 CHANGE 都需要把列定义写完整。比如原本 `age INT NOT NULL DEFAULT 0`，执行 `MODIFY COLUMN age TINYINT` 后，NOT NULL 和 DEFAULT 0 约束会**丢失**。修改列时应补全所有要保留的约束。

#### 约束和索引操作

```sql
-- 添加主键
ALTER TABLE student ADD PRIMARY KEY (id);

-- 删除主键（AUTO_INCREMENT 列必须先 MODIFY 去掉自增）
ALTER TABLE student DROP PRIMARY KEY;

-- 添加/删除唯一约束
ALTER TABLE student ADD UNIQUE uq_email (email);
ALTER TABLE student DROP INDEX uq_email;

-- 添加/删除普通索引
ALTER TABLE student ADD INDEX idx_name (name);
ALTER TABLE student DROP INDEX idx_name;

-- 添加/删除外键
ALTER TABLE student ADD CONSTRAINT fk_class
    FOREIGN KEY (class_id) REFERENCES class(id)
    ON DELETE SET NULL ON UPDATE CASCADE;
ALTER TABLE student DROP FOREIGN KEY fk_class;
```

#### Online DDL

MySQL 5.6+ 支持在线 DDL：添加/删除索引、重命名列等操作**不阻塞读写**。修改列类型通常需要重建表（阻塞写操作）。执行 ALTER 前评估对大表的影响。

```sql
-- 查看 ALTER 操作是否支持在线执行（不锁表）
-- ALGORITHM=INPLACE 表示在线操作，ALGORITHM=COPY 需重建表
ALTER TABLE student ADD INDEX idx_age (age), ALGORITHM=INPLACE, LOCK=NONE;
```

### 1.5 删除表

```sql
DROP TABLE IF EXISTS student;            -- 删除表（结构+数据+索引）
DROP TABLE IF EXISTS student, class;     -- 一次删除多张表
```

`DROP TABLE` 是 DDL，不可回滚。`IF EXISTS` 在表不存在时不报错。删除有外键引用的表需先删除外键或使用 `SET FOREIGN_KEY_CHECKS = 0`。

### 1.6 复制表

```sql
CREATE TABLE t_copy AS SELECT * FROM t;   -- 结构+数据，不复制索引/主键/自增
CREATE TABLE t_empty LIKE t;              -- 仅结构，复制索引/主键，不含数据
```

---

## 二、DML -- 数据操作语言

### 2.1 INSERT

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

### 2.2 UPDATE

**UPDATE** 修改表中已有行。**SET** 指定要更新的列及新值。

```sql
UPDATE 表名 SET 列1 = 值1, 列2 = 值2 WHERE 条件;
```

#### SET 写法

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

#### 基础更新

```sql
UPDATE student SET age = 21 WHERE id = 1;
UPDATE student SET age = age + 1 WHERE class_id = 2;
UPDATE student SET class_id = 4 WHERE age > (SELECT AVG(age) FROM student);
```

#### CASE 条件更新

同一列按不同条件赋不同值，用 CASE 一条语句完成，避免多条 UPDATE：

```sql
UPDATE student
SET age = CASE
    WHEN class_id = 1 THEN age + 1
    WHEN class_id = 2 THEN age + 2
    ELSE age
END;
```

#### 安全更新

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

### 2.3 DELETE

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
| 重置自增 | 否 | 是 | -- |
| WHERE | 支持 | 不支持 | 不支持 |
| 速度 | 逐行，较慢 | 整表回收，快 | 快 |

**多表删除**：

```sql
DELETE s, c FROM student s JOIN class c ON s.class_id = c.id WHERE s.age < 18;
```

---

## 三、DQL -- 数据查询语言

### 3.1 基本查询

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

### 3.2 WHERE 条件过滤

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

WHERE 子句中每个条件都是一个**布尔表达式（Boolean Expression）**，计算结果为 TRUE（1）、FALSE（0）或 NULL。值为 TRUE 的行被保留，FALSE 或 NULL 的行被丢弃。

比较运算符（`>`、`=`、`IS NULL` 等）返回 0 或 1，逻辑运算符（`AND`、`OR`、`NOT`）组合这些 0/1。

布尔表达式返回值可在 SQL 的多个位置使用：

| 位置 | 示例 | 说明 |
|------|------|------|
| SELECT 列表 | `SELECT name, age > 18 AS is_adult FROM student` | 每行返回 1 或 0 |
| ORDER BY | `ORDER BY age IS NULL, age ASC` | 利用 0/1 排序特性控制 NULL 位置（0 先于 1） |
| CASE | `CASE WHEN age >= 18 THEN '成年' ELSE '未成年' END` | 布尔值决定分支 |
| HAVING | `HAVING COUNT(*) > 3` | 对分组聚合结果做判断 |
| 算术运算 | `score * (age > 18)` | 返回值是数字，可参与乘法等运算 |
| 函数参数 | `IF(age >= 18, '成年', '未成年')` | 作为条件函数的第一参数 |

### 3.3 排序 ORDER BY

**ORDER BY** 对查询结果集按指定列或表达式排序

| 关键字 | 行为 |
|--------|------|
| `ASC` | 升序（默认） |
| `DESC` | 降序 |

```sql
SELECT * FROM student ORDER BY age ASC;
SELECT * FROM student ORDER BY age DESC;
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

- 排序数据量 <= `sort_buffer_size` 时，在内存中完成
- 超出 `sort_buffer_size` 时，使用磁盘临时文件，性能显著下降

复合索引与排序的最左前缀规则：

- `INDEX(a, b, c)` 对 `ORDER BY a` 和 `ORDER BY a, b` 有效
- `INDEX(a, b, c)` 对 `ORDER BY b` 和 `ORDER BY b, c` 无效

MySQL 8.0+ 支持**降序索引（descending index）**，创建时指定列的排序方向：

```sql
ALTER TABLE student ADD INDEX idx_age_desc (age DESC);
```

当 ORDER BY 混合 ASC/DESC 且与索引列的排序方向匹配时，可利用索引避免 filesort。

### 3.4 分页 LIMIT

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

### 3.5 聚合函数

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

### 3.6 分组 GROUP BY + HAVING

```sql
SELECT class_id, gender, COUNT(*) AS cnt
FROM student
GROUP BY class_id, gender
HAVING cnt >= 2;
```

WHERE 分组前逐行过滤，HAVING 分组后按聚合结果过滤。WHERE 不可用聚合函数。

### 3.7 子查询

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

### 3.8 多表连接 JOIN

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

### 3.9 联合查询 UNION

```sql
SELECT name FROM t1
UNION ALL              -- 不去重，更快
SELECT name FROM t2;
```

`UNION`（去重）vs `UNION ALL`（不去重）-- 后者无排序去重开销，多数场景优先。

---

## 四、导出与导入

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

## 五、SQL 执行顺序

```
FROM -> JOIN -> WHERE -> GROUP BY -> SELECT -> HAVING -> ORDER BY -> LIMIT
```

别名在 SELECT 阶段才计算，因此 WHERE 不能用 SELECT 中定义的别名，ORDER BY 可以。

---

## 六、DCL — 数据控制语言

DCL 管理 MySQL 用户账户和访问权限。

### 6.1 用户管理

```sql
-- 创建用户
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'secure_pwd';
CREATE USER 'app_user'@'%' IDENTIFIED BY 'secure_pwd';       -- % 允许任意主机连接

-- 修改密码
ALTER USER 'app_user'@'localhost' IDENTIFIED BY 'new_pwd';

-- 删除用户
DROP USER 'app_user'@'localhost';

-- 重命名用户
RENAME USER 'old_name'@'localhost' TO 'new_name'@'localhost';

-- 查看所有用户
SELECT User, Host FROM mysql.user;
```

| 操作 | 说明 |
|------|------|
| `'user'@'host'` | 用户 = 用户名 + 主机。同一用户名可从不同主机连接，视为不同账户 |
| `IDENTIFIED BY` | 设置密码。MySQL 8.0 默认使用 caching_sha2_password 插件 |
| `%` | 通配任意主机。`'user'@'%'` 允许从任何 IP 连接 |

**锁定/解锁账户**：

```sql
ALTER USER 'app_user'@'localhost' ACCOUNT LOCK;      -- 锁定，禁止登录
ALTER USER 'app_user'@'localhost' ACCOUNT UNLOCK;    -- 解锁
```

**密码过期策略**：

```sql
-- 设置密码过期（下次登录强制修改）
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE;
-- 设置密码永不过期
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE NEVER;
-- 设置密码 N 天后过期
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE INTERVAL 90 DAY;
```

### 6.2 权限管理

| 级别 | 语法 | 示例 |
|------|------|------|
| 全局 | `GRANT ... ON *.*` | 所有库的所有表 |
| 数据库 | `GRANT ... ON mydb.*` | 指定库的所有表 |
| 表 | `GRANT ... ON mydb.student` | 指定表 |
| 列 | `GRANT SELECT (col1, col2) ON mydb.student` | 指定表的指定列 |
| 存储过程 | `GRANT EXECUTE ON PROCEDURE mydb.sp_name` | 指定存储过程 |

**常用权限**：

| 权限 | 作用域 | 说明 |
|------|--------|------|
| `ALL PRIVILEGES` | 所有 | 授予所有权限（不含 GRANT OPTION） |
| `SELECT` | 全局/库/表/列 | 查询数据 |
| `INSERT` | 全局/库/表/列 | 插入数据 |
| `UPDATE` | 全局/库/表/列 | 修改数据 |
| `DELETE` | 全局/库/表 | 删除数据 |
| `CREATE` | 全局/库 | 创建库或表 |
| `ALTER` | 全局/库/表 | 修改表结构 |
| `DROP` | 全局/库/表 | 删除库或表 |
| `INDEX` | 全局/库/表 | 创建和删除索引 |
| `CREATE USER` | 全局 | 创建用户 |
| `GRANT OPTION` | 所有 | 将自己拥有的权限授予其他用户 |

```sql
-- 授予权限
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'app_user'@'localhost';

-- 授予所有权限（含 GRANT OPTION）
GRANT ALL PRIVILEGES ON mydb.* TO 'admin'@'localhost' WITH GRANT OPTION;

-- 授予列级别权限（仅允许查询 name, age 两列）
GRANT SELECT (name, age) ON mydb.student TO 'readonly_user'@'localhost';

-- 回收权限
REVOKE INSERT, UPDATE ON mydb.* FROM 'app_user'@'localhost';

-- 回收所有权限
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'app_user'@'localhost';

-- 查看自身权限
SHOW GRANTS;
-- 查看指定用户权限
SHOW GRANTS FOR 'app_user'@'localhost';
```

### 6.3 角色（MySQL 8.0+）

角色是一组权限的命名集合，可授予用户或撤销，简化多用户权限管理。

```sql
-- 创建角色
CREATE ROLE 'read_role', 'write_role', 'admin_role';

-- 为角色授予权限
GRANT SELECT ON mydb.* TO 'read_role';
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'write_role';
GRANT ALL PRIVILEGES ON mydb.* TO 'admin_role';

-- 将角色授予用户
GRANT 'read_role' TO 'analyst'@'localhost';
GRANT 'write_role' TO 'developer'@'localhost';

-- 一个用户可拥有多个角色
GRANT 'read_role', 'write_role' TO 'developer'@'localhost';

-- 设置默认角色（登录时自动激活）
SET DEFAULT ROLE 'write_role' TO 'developer'@'localhost';
SET DEFAULT ROLE ALL TO 'developer'@'localhost';  -- 激活所有角色

-- 撤销角色
REVOKE 'write_role' FROM 'developer'@'localhost';

-- 删除角色
DROP ROLE 'read_role';
```

| 操作 | 说明 |
|------|------|
| 角色与用户不同 | 角色默认不能登录（无密码），但可为角色设密码实现角色切换 |
| 默认角色 | 授予后须用 SET DEFAULT ROLE 设置默认角色或被授予者手动 `SET ROLE` 激活 |
| 角色嵌套 | 角色可授予另一个角色，权限自动传递 |
| `mandatory_roles` | 系统变量，设为全局强制角色，所有用户自动被授予且不可撤销 |

---

## 七、事务

**事务（Transaction）** 将一组操作视为不可分割的单元：要么全部成功，要么全部撤销。

### 7.1 ACID 特性

| 缩写 | 全称 | 含义 |
|------|------|------|
| **A** | Atomicity（原子性） | 事务中所有操作是一个整体。全部成功则提交，任一失败则回滚 |
| **C** | Consistency（一致性） | 事务开始前和结束后，数据库的完整性约束未被破坏 |
| **I** | Isolation（隔离性） | 并发事务之间互不干扰。每个事务看到的数据要么是其他事务提交前的状态，要么是提交后的状态 |
| **D** | Durability（持久性） | 事务提交后，其对数据的修改是永久的，即使系统崩溃也不丢失 |

### 7.2 事务控制

```sql
-- 显式开启事务
START TRANSACTION;
-- 或
BEGIN;

-- 提交事务（持久化所有修改）
COMMIT;

-- 回滚事务（撤销所有修改）
ROLLBACK;

-- 示例：转账
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
-- 检查无误后
COMMIT;
-- 若出错
ROLLBACK;
```

**保存点（SAVEPOINT）**：在事务中创建标记，可部分回滚到该标记而非回滚整个事务。

```sql
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
SAVEPOINT sp1;                                          -- 创建保存点
UPDATE account SET balance = balance + 100 WHERE id = 2;
-- 发现第二条有问题，只回滚到保存点
ROLLBACK TO SAVEPOINT sp1;                              -- 撤销第二条，保留第一条
-- 释放保存点
RELEASE SAVEPOINT sp1;
COMMIT;
```

| 命令 | 说明 |
|------|------|
| `SAVEPOINT 名称` | 在事务中创建一个标记点 |
| `ROLLBACK TO SAVEPOINT 名称` | 回滚到指定标记，标记之后的修改全部撤销。保存点本身不释放 |
| `RELEASE SAVEPOINT 名称` | 删除保存点，释放资源 |

### 7.3 自动提交

MySQL 默认每条语句自动提交（autocommit = ON）。

```sql
-- 查看当前设置
SELECT @@autocommit;                          -- 1 = ON, 0 = OFF

-- 关闭自动提交（会话级别）
SET autocommit = 0;
-- 此后每条 DML 需手动 COMMIT 或 ROLLBACK，否则事务一直未关闭
```

| 设置 | 行为 | 影响 |
|------|------|------|
| `autocommit = 1`（默认） | 每条 DML 语句自动作为一个事务提交 | 每条语句立即持久化 |
| `autocommit = 0` | 所有 DML 在同一个未关闭的事务中，直到手动 COMMIT/ROLLBACK | 忘记提交可能导致长时间锁持有 |
| `START TRANSACTION` | 对于 autocommit=1 的连接，显式开始一个多语句事务，直到 COMMIT/ROLLBACK | 临时暂停自动提交 |

### 7.4 隐式提交

以下语句执行时会隐式提交当前未关闭的事务：

| 类别 | 语句 |
|------|------|
| DDL | `CREATE TABLE`、`ALTER TABLE`、`DROP TABLE`、`CREATE INDEX`、`TRUNCATE` |
| DCL | `GRANT`、`REVOKE`、`CREATE USER`、`DROP USER` |
| 管理 | `BEGIN`（提交上一个事务）、`START TRANSACTION`（同上）、`FLUSH`、`RESET` |
| 锁相关 | `LOCK TABLES`、`UNLOCK TABLES` |

### 7.5 事务隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 |
|------|:--:|:--:|:--:|
| **READ UNCOMMITTED** | ✓ | ✓ | ✓ |
| **READ COMMITTED** | ✗ | ✓ | ✓ |
| **REPEATABLE READ**（MySQL 默认） | ✗ | ✗ | 部分避免 |
| **SERIALIZABLE** | ✗ | ✗ | ✗ |

| 现象 | 描述 |
|------|------|
| **脏读（Dirty Read）** | 读取到另一个事务未提交的修改 |
| **不可重复读（Non-Repeatable Read）** | 同一事务内两次读取同一行，结果不同（另一事务在中间提交了 UPDATE） |
| **幻读（Phantom Read）** | 同一事务内两次查询同一条件，结果行数不同（另一事务在中间 INSERT 了匹配行） |

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;
-- 查看全局隔离级别
SELECT @@global.transaction_isolation;

-- 设置当前会话隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- 设置全局隔离级别
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**隔离级别示例**（以两个并发会话演示 REPEATABLE READ）：

```sql
-- 会话 A                               -- 会话 B
START TRANSACTION;                      START TRANSACTION;
SELECT age FROM student WHERE id=1;     -- 结果 20
                                        UPDATE student SET age=21 WHERE id=1;
                                        COMMIT;
SELECT age FROM student WHERE id=1;     -- 结果 20（看不到 B 的修改，MVCC 机制）
COMMIT;
SELECT age FROM student WHERE id=1;     -- 结果 21（提交后看到最新值）
```

### 7.6 MVCC

InnoDB 使用**多版本并发控制（Multi-Version Concurrency Control，MVCC）** 实现事务隔离。

- 每行数据维护多个版本（undo log 中的旧版本链）
- 每个事务开始时创建一个 **Read View**（快照），记录当时活跃的事务 ID 列表
- 事务读取数据时，通过 Read View 和行版本的 trx_id 判断该版本是否对本事务可见
- REPEATABLE READ 下，同一事务内用同一个 Read View，因此读到一致的数据快照
- READ COMMITTED 下，每次查询生成新的 Read View，可看到已提交的修改

MVCC 使得读操作不阻塞写，写操作也不阻塞读，提高并发性能。

---

## 八、索引

**索引（Index）** 是表中一列或多列值排序后的数据结构（默认 B+Tree），用于加速数据检索。类比书的目录。

### 8.1 索引类型

```sql
-- 主键索引（唯一 + 非空，一张表仅一个）
CREATE TABLE student (id INT AUTO_INCREMENT PRIMARY KEY);

-- 唯一索引（值不可重复，允许多个 NULL）
CREATE UNIQUE INDEX uq_email ON student(email);
ALTER TABLE student ADD UNIQUE uq_email (email);

-- 普通索引（加速查询，无约束）
CREATE INDEX idx_name ON student(name);
ALTER TABLE student ADD INDEX idx_name (name);

-- 复合索引（多列联合）
CREATE INDEX idx_class_age ON student(class_id, age);
ALTER TABLE student ADD INDEX idx_class_age (class_id, age);

-- 前缀索引（仅索引字符串前 N 个字符，节省空间）
CREATE INDEX idx_desc_prefix ON product(description(20));

-- 删除索引
DROP INDEX idx_name ON student;
ALTER TABLE student DROP INDEX idx_name;

-- 查看表的索引
SHOW INDEX FROM student;
```

| 索引类型 | 特点 | 适用场景 |
|---------|------|---------|
| **PRIMARY KEY** | 唯一且非空，每表仅一个。InnoDB 中即为聚簇索引 | 主键 |
| **UNIQUE** | 值不可重复，允许多个 NULL | 邮箱、手机号、身份证号 |
| **INDEX**（普通） | 仅加速查询 | WHERE / JOIN / ORDER BY 列 |
| **FULLTEXT** | 全文搜索 | 文章内容、商品描述（详见第十九章） |
| **SPATIAL** | 空间数据索引 | 地理位置坐标（R-Tree 索引） |

### 8.2 B+Tree 索引原理

InnoDB 使用 B+Tree 作为默认索引结构：

- 数据按索引列的值排序存储在叶子节点中
- 非叶子节点只存键值和子节点指针，不存数据
- 所有叶子节点通过双向链表连接，支持范围查询的高效遍历
- 聚簇索引（主键）的叶子节点就是数据行本身
- 辅助索引（二级索引）的叶子节点存的是主键值，回表查数据

### 8.3 最左前缀原则

复合索引遵从最左前缀匹配：只有当查询条件以索引的最左列为起点，且不跳过中间列时，索引才被使用。

假设索引：`INDEX idx_a_b_c (a, b, c)`：

| 查询条件 | 是否使用索引 | 说明 |
|----------|:----------:|------|
| `WHERE a = 1` | ✓ | 匹配最左列 a |
| `WHERE a = 1 AND b = 2` | ✓ | 匹配 (a, b) 前缀 |
| `WHERE a = 1 AND b = 2 AND c = 3` | ✓ | 匹配全部三列 |
| `WHERE a = 1 AND c = 3` | 仅 a | 跳过了 b，c 无法使用索引 |
| `WHERE b = 2` | ✗ | 不以 a 开头 |
| `WHERE b = 2 AND c = 3` | ✗ | 不以 a 开头 |
| `WHERE a = 1 AND b > 2 AND c = 3` | 仅 (a, b) | 范围查询 b > 2 中断了后续列的索引使用 |
| `WHERE a = 1 AND b = 2 ORDER BY c` | ✓ | (a, b) 用于过滤，(a, b, c) 用于排序 |

**实际示例**：

```sql
-- 创建复合索引
CREATE INDEX idx_name_age_gender ON student(name, age, gender);

-- 1. 完全匹配（索引高效使用）
EXPLAIN SELECT * FROM student WHERE name = '张三' AND age = 20 AND gender = '男';
-- key: idx_name_age_gender, key_len: 完整长度

-- 2. 跳过中间列 age，gender 无法用索引
EXPLAIN SELECT * FROM student WHERE name = '张三' AND gender = '男';
-- key: idx_name_age_gender, key_len: 仅 name 部分长度（未用到 gender）

-- 3. 不以 name 开头，整个索引不用
EXPLAIN SELECT * FROM student WHERE age = 20;
-- key: NULL（全表扫描）

-- 4. 范围查询中断后续列
EXPLAIN SELECT * FROM student WHERE name = '张三' AND age > 18 AND gender = '男';
-- key: idx_name_age_gender, key_len: name + age 部分
```

### 8.4 覆盖索引

当查询所需的所有列都包含在索引中时，无需回表查询数据行。EXPLAIN 的 Extra 列显示 `Using index`。

```sql
-- 创建覆盖索引（包含查询所需全部列）
CREATE INDEX idx_cover ON student(name, age, gender);

-- 覆盖索引：SELECT 的列都在索引中，Extra = 'Using index'
EXPLAIN SELECT name, age, gender FROM student WHERE name = '张三';
-- 回表查询：SELECT 包含不在索引中的列，Extra = NULL 或 'Using where'
EXPLAIN SELECT name, age, gender, class_id FROM student WHERE name = '张三';
```

| Extra 值 | 含义 |
|-----------|------|
| `Using index` | 覆盖索引，无需回表 |
| `Using index condition` | 索引下推（ICP），索引过滤部分条件后回表 |
| `Using where` | 存储引擎返回行后，MySQL 在 Server 层再过滤 |
| `Using filesort` | 额外排序操作，排序未用到索引 |
| `Using temporary` | 使用临时表（通常出现在 GROUP BY/DISTINCT/UNION） |

### 8.5 EXPLAIN 解读

```sql
EXPLAIN SELECT * FROM student WHERE name = '张三' AND age > 18;
```

| 列 | 含义 | 关注点 |
|----|------|--------|
| `id` | SELECT 序号 | 多表查询时 id 相同则从上往下执行，不同则从大到小执行 |
| `select_type` | 查询类型 | `SIMPLE`（简单查询）、`PRIMARY`（外层查询）、`SUBQUERY`（子查询）、`DERIVED`（派生表） |
| `table` | 访问的表 | 表名或别名 |
| **`type`** | 访问方式 | **从好到差**：`system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL` |
| `possible_keys` | 可能使用的索引 | 最多列出多个候选索引 |
| **`key`** | 实际使用的索引 | `NULL` 表示未使用索引 |
| `key_len` | 使用的索引字节数 | 用于判断复合索引中用了哪几列 |
| `ref` | 与索引比较的值 | `const`（常量）、列名 |
| **`rows`** | 预估扫描行数 | 值越小越好，关注大表时该值是否过大 |
| `filtered` | 按条件过滤后剩余的比例 | 100% 表示无额外过滤 |
| **`Extra`** | 额外信息 | 见上表 Extra 值含义 |

**type 访问方式详解**：

| type | 含义 | 示例 |
|------|------|------|
| `system` | 表仅一行（系统表） | 极少见 |
| `const` | 主键或唯一索引等值匹配，最多返回一行 | `WHERE id = 1` |
| `eq_ref` | JOIN 中通过主键或唯一索引匹配，每个驱动表行只匹配一行 | `JOIN ... ON t2.id = t1.id` |
| `ref` | 非唯一索引等值匹配，可能返回多行 | `WHERE name = '张三'` |
| `range` | 索引范围扫描 | `WHERE id > 100`、`WHERE name BETWEEN 'A' AND 'C'` |
| `index` | 全索引扫描（扫描整个索引树） | `SELECT name FROM student`（覆盖索引触发） |
| `ALL` | 全表扫描 | 无可用索引，需优化 |

### 8.6 索引优化策略

| 策略 | 说明 | 示例 |
|------|------|------|
| 避免在索引列上使用函数 | 函数破坏了索引值的直接查找 | `WHERE YEAR(created_at) = 2025` → `WHERE created_at BETWEEN '2025-01-01' AND '2025-12-31'` |
| 避免对索引列做计算 | 同样破坏直接查找 | `WHERE age + 1 = 20` → `WHERE age = 19` |
| 隐式类型转换导致索引失效 | 字符串列用数字比较会触发转换 | `WHERE phone = 13800138000`（phone 是 VARCHAR） → `WHERE phone = '13800138000'` |
| 前缀索引节省空间 | 字符串列取前 N 字符建索引 | `INDEX idx_desc (description(20))` |
| 不适合建索引的列 | 区分度低的列（如性别）、频繁更新的列 | 查询优化效果差，维护成本高 |
| 索引合并 | MySQL 5.0+ 一条查询可用多个索引 | `WHERE name='张三' AND age=20`，两列各有独立索引时可合并 |

**索引下推（Index Condition Pushdown，ICP）**：MySQL 5.6+ 将部分 WHERE 条件交给存储引擎层使用索引过滤，减少回表次数。

### 8.7 不可见索引（MySQL 8.0+）

将索引设为不可见，查询优化器不使用该索引，但索引的维护仍继续（INSERT/UPDATE/DELETE 依然更新索引）。用于测试索引的收益：令索引不可见 → 观察慢查询 → 再令其可见。

```sql
ALTER TABLE student ALTER INDEX idx_name INVISIBLE;   -- 设为不可见
ALTER TABLE student ALTER INDEX idx_name VISIBLE;     -- 设为可见
```

### 8.8 索引代价

- 写入性能：每次 INSERT、UPDATE、DELETE 需同步维护所有索引
- 存储空间：索引本身占用磁盘空间
- 过多索引导致查询优化器选择低效索引或费时评估

---

## 九、视图

**视图（View）** 是存储的 SELECT 查询语句，像一个虚拟表。查询视图时，MySQL 执行视图底层的 SELECT 并返回结果。

### 9.1 创建与使用

```sql
-- 创建视图
CREATE VIEW v_student_info AS
SELECT s.name, s.age, c.class_name
FROM student s
LEFT JOIN class c ON s.class_id = c.id;

-- 查询视图（与查询表语法相同）
SELECT * FROM v_student_info WHERE age > 18;

-- 修改视图定义
CREATE OR REPLACE VIEW v_student_info AS
SELECT s.name, s.age, s.gender, c.class_name
FROM student s
LEFT JOIN class c ON s.class_id = c.id;

-- 或使用 ALTER VIEW
ALTER VIEW v_student_info AS
SELECT s.name, s.age, s.gender, c.class_name
FROM student s
LEFT JOIN class c ON s.class_id = c.id;

-- 删除视图
DROP VIEW IF EXISTS v_student_info;
-- 查看视图定义
SHOW CREATE VIEW v_student_info;
```

### 9.2 视图的作用

| 作用 | 说明 | 示例 |
|------|------|------|
| 简化复杂查询 | 把多表 JOIN、聚合等复杂查询封装为视图 | 上例将 student JOIN class 封装为 `v_student_info` |
| 安全隔离 | 只暴露部分列给用户，隐藏敏感列 | `CREATE VIEW v_public AS SELECT id, name FROM student` |
| 兼容旧代码 | 表结构变更后，视图保持旧列名和结构作为过渡层 | 列改名后创建同名视图，旧应用继续工作 |

### 9.3 WITH CHECK OPTION

限制通过视图 INSERT/UPDATE 的数据必须满足视图的 WHERE 条件。

```sql
-- 创建只读视图（年龄 ≥ 18 的学生）
CREATE VIEW v_adult AS
SELECT * FROM student WHERE age >= 18
WITH CHECK OPTION;

-- 允许：插入符合条件的数据
INSERT INTO v_adult (name, age) VALUES ('张三', 20);

-- 拒绝：age=15 不满足 age>=18 条件，报错
INSERT INTO v_adult (name, age) VALUES ('李四', 15);

-- CASCADED vs LOCAL
-- CASCADED（默认）：检查本视图及其所有底层视图的条件
-- LOCAL：仅检查当前视图的条件
```

### 9.4 可更新视图的限制

视图满足以下全部条件才可直接 INSERT/UPDATE/DELETE：

- SELECT 中只包含列名，无聚合函数、DISTINCT、GROUP BY、HAVING
- 只引用一张表（无 JOIN）
- 视图列不来自表达式或函数
- 不包含 UNION

不满足条件的视图不可直接更新（报错），但可用触发器模拟更新。

---

## 十、存储过程与函数

**存储过程（Stored Procedure）** 和**函数（Function）** 是将多条 SQL 封装为可重复调用的程序单元，存储在服务端。

| | 存储过程 | 函数 |
|---|---------|------|
| 返回值 | 通过 OUT/INOUT 参数返回多个值 | 返回单个标量值（必须有 RETURNS 子句） |
| SQL 中调用 | `CALL procedure_name(...)` | `SELECT func_name(...)` |
| 事务控制 | 可在过程中 COMMIT/ROLLBACK | 函数中不允许显式提交/回滚 |
| 适用场景 | 业务逻辑封装、批量操作、日志写入 | 计算、格式化、返回派生值 |

### 10.1 DELIMITER

创建包含多条语句（含分号）的存储程序时，需临时改变 MySQL 客户端的语句分隔符：

```sql
DELIMITER $$

CREATE PROCEDURE sp_name()
BEGIN
    -- 多条以分号结尾的 SQL 语句
END$$

DELIMITER ;
-- 恢复为默认分隔符
```

`DELIMITER` 是 mysql CLI 命令（非 SQL），告诉客户端"语句以 $$ 结束"。常见分隔符：`$$`、`//`、`;;`。

### 10.2 存储过程

```sql
DELIMITER $$

-- 无参过程
CREATE PROCEDURE sp_list_students()
BEGIN
    SELECT id, name, age FROM student ORDER BY id;
END$$

-- IN 参数（传入值）
CREATE PROCEDURE sp_student_by_class(IN p_class_id INT)
BEGIN
    SELECT name, age, gender
    FROM student WHERE class_id = p_class_id;
END$$

-- OUT 参数（传出结果）
CREATE PROCEDURE sp_count_by_gender(IN p_gender ENUM('男','女'), OUT p_count INT)
BEGIN
    SELECT COUNT(*) INTO p_count FROM student WHERE gender = p_gender;
END$$

-- INOUT 参数（传入 + 传出）
CREATE PROCEDURE sp_double(INOUT p_value INT)
BEGIN
    SET p_value = p_value * 2;
END$$

-- 多条语句过程
CREATE PROCEDURE sp_transfer(
    IN from_id INT, IN to_id INT, IN amount DECIMAL(10,2), OUT p_status VARCHAR(20)
)
BEGIN
    DECLARE from_balance DECIMAL(10,2);
    START TRANSACTION;
    SELECT balance INTO from_balance FROM account WHERE id = from_id;
    IF from_balance >= amount THEN
        UPDATE account SET balance = balance - amount WHERE id = from_id;
        UPDATE account SET balance = balance + amount WHERE id = to_id;
        COMMIT;
        SET p_status = 'SUCCESS';
    ELSE
        ROLLBACK;
        SET p_status = 'INSUFFICIENT';
    END IF;
END$$

DELIMITER ;
```

调用存储过程：

```sql
-- 无参
CALL sp_list_students();

-- 带 IN 参数
CALL sp_student_by_class(1);

-- 带 OUT 参数
CALL sp_count_by_gender('男', @cnt);
SELECT @cnt;                        -- 查看输出值

-- 带 INOUT 参数
SET @val = 5;
CALL sp_double(@val);
SELECT @val;                        -- 结果 10
```

**查看与删除**：

```sql
SHOW PROCEDURE STATUS WHERE Db = 'mydb';     -- 列出库中的过程
SHOW CREATE PROCEDURE sp_name;               -- 查看过程源码
DROP PROCEDURE IF EXISTS sp_name;            -- 删除过程
```

### 10.3 函数

```sql
DELIMITER $$

CREATE FUNCTION fn_age_group(p_age INT) RETURNS VARCHAR(10)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE v_result VARCHAR(10);
    IF p_age < 18 THEN
        SET v_result = '未成年';
    ELSEIF p_age < 35 THEN
        SET v_result = '青年';
    ELSEIF p_age < 60 THEN
        SET v_result = '中年';
    ELSE
        SET v_result = '老年';
    END IF;
    RETURN v_result;
END$$

DELIMITER ;
```

函数必须指定以下特性（二选一）：

| 特性 | 含义 |
|------|------|
| `DETERMINISTIC` | 相同输入永远返回相同输出（如 `fn_age_group`） |
| `NOT DETERMINISTIC` | 相同输入可能返回不同输出（如 `NOW()`、`RAND()`） |

| 访问修饰 | 含义 |
|----------|------|
| `READS SQL DATA` | 函数会读取但不修改数据 |
| `NO SQL` | 函数不读取也不修改数据 |
| `MODIFIES SQL DATA` | 函数会修改数据 |

调用函数：

```sql
SELECT name, age, fn_age_group(age) AS age_group FROM student;
```

函数不能执行事务控制语句（COMMIT/ROLLBACK），不能写数据（除非声明 MODIFIES SQL DATA 且不在 SELECT 中调用）。

### 10.4 变量

```sql
DECLARE v_name VARCHAR(50);                 -- 声明局部变量（必须在 BEGIN 后，语句前）
DECLARE v_total INT DEFAULT 0;              -- 声明并赋默认值
SET v_name = '张三';                         -- 赋值
SELECT age INTO v_age FROM student WHERE id = 1;  -- 查询结果赋给变量（必须返回恰好一行）
```

**用户变量**（与过程内局部变量不同）：

```sql
SET @name = '张三';                           -- 定义用户变量（前缀 @），无需声明
SELECT @name;                                 -- 查看
SELECT age INTO @age FROM student WHERE id = 2;  -- 查询结果赋给用户变量
```

| 类型 | 语法 | 作用域 | 生命周期 |
|------|------|--------|---------|
| 局部变量 | `DECLARE` 在 BEGIN...END 内 | 当前存储程序块 | 存储程序执行结束 |
| 用户变量 | `@变量名`，无需声明 | 当前会话 | 会话结束或断开连接 |
| 系统变量 | `@@变量名` | 全局或会话 | 随服务器 |

### 10.5 流程控制

**IF 语句**：

```sql
IF condition1 THEN
    -- statement
ELSEIF condition2 THEN
    -- statement
ELSE
    -- statement
END IF;
```

**CASE 语句**：

```sql
CASE v_grade
    WHEN 'A' THEN SET v_point = 4.0;
    WHEN 'B' THEN SET v_point = 3.0;
    WHEN 'C' THEN SET v_point = 2.0;
    ELSE SET v_point = 0;
END CASE;
```

**循环**：

| 循环类型 | 语法 | 特点 |
|---------|------|------|
| WHILE | `WHILE condition DO ... END WHILE;` | 先判断再执行，条件为 FALSE 则一次不执行 |
| REPEAT | `REPEAT ... UNTIL condition END REPEAT;` | 先执行再判断，至少执行一次 |
| LOOP | `label: LOOP ... LEAVE label; ... END LOOP;` | 无限循环，需用 LEAVE 退出 |

```sql
-- WHILE 示例
SET v_i = 0;
WHILE v_i < 10 DO
    SET v_i = v_i + 1;
    INSERT INTO log (msg) VALUES (CONCAT('loop:', v_i));
END WHILE;

-- LOOP + LEAVE 示例
SET v_i = 0;
my_loop: LOOP
    SET v_i = v_i + 1;
    IF v_i > 10 THEN
        LEAVE my_loop;
    END IF;
    INSERT INTO log (msg) VALUES (CONCAT('loop:', v_i));
END LOOP;

-- ITERATE（相当于 continue）
SET v_i = 0;
my_loop: LOOP
    SET v_i = v_i + 1;
    IF v_i MOD 2 = 0 THEN
        ITERATE my_loop;                    -- 跳过偶数
    END IF;
    INSERT INTO log (msg) VALUES (CONCAT('odd:', v_i));
    IF v_i >= 20 THEN LEAVE my_loop; END IF;
END LOOP;
```

### 10.6 错误处理

```sql
DELIMITER $$

CREATE PROCEDURE sp_safe_insert(IN p_name VARCHAR(50))
BEGIN
    -- 声明 CONTINUE HANDLER：遇错误不终止，继续执行
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
        INSERT INTO error_log (msg, ts) VALUES (CONCAT('insert failed: ', p_name), NOW());

    -- 声明 EXIT HANDLER：遇错误直接退出过程
    DECLARE EXIT HANDLER FOR 1062            -- 1062 = 唯一键冲突错误码
    BEGIN
        -- 错误处理逻辑
        ROLLBACK;
    END;

    -- 也可以按 SQLSTATE 捕获
    DECLARE EXIT HANDLER FOR SQLSTATE '23000';    -- 完整性约束违反

    INSERT INTO student (name) VALUES (p_name);
    COMMIT;
END$$

DELIMITER ;
```

| HANDLER 类型 | 行为 |
|-------------|------|
| `CONTINUE` | 执行 handler 后继续执行原语句之后的语句 |
| `EXIT` | 执行 handler 后退出当前 BEGIN...END 块 |

### 10.7 游标

逐行处理 SELECT 结果集：

```sql
DELIMITER $$

CREATE PROCEDURE sp_process_students()
BEGIN
    DECLARE done INT DEFAULT 0;
    DECLARE v_name VARCHAR(50);
    DECLARE v_age INT;

    -- 声明游标
    DECLARE cur CURSOR FOR SELECT name, age FROM student;

    -- 声明未找到处理：游标遍历完时 SET done = 1
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

    OPEN cur;                                         -- 打开游标

    read_loop: LOOP
        FETCH cur INTO v_name, v_age;                 -- 取当前行，移到下一行
        IF done THEN
            LEAVE read_loop;
        END IF;
        -- 处理当前行数据
        UPDATE student SET age = v_age + 1 WHERE name = v_name;
    END LOOP;

    CLOSE cur;                                        -- 关闭游标
END$$

DELIMITER ;
```

游标开销大，能用一条 SQL 批量完成的操作不要用游标逐行处理。

---

## 十一、触发器

**触发器（Trigger）** 在 INSERT、UPDATE 或 DELETE 事件发生之前或之后自动执行一段代码。

### 11.1 创建触发器

```sql
-- BEFORE INSERT 触发器：插入前自动设置 created_at
CREATE TRIGGER trg_student_before_insert
BEFORE INSERT ON student
FOR EACH ROW
SET NEW.created_at = IFNULL(NEW.created_at, NOW());

-- AFTER INSERT 触发器：插入后记录日志
CREATE TRIGGER trg_student_after_insert
AFTER INSERT ON student
FOR EACH ROW
INSERT INTO audit_log (table_name, action, record_id, ts)
VALUES ('student', 'INSERT', NEW.id, NOW());

-- BEFORE UPDATE 触发器：更新前自动设 updated_at
CREATE TRIGGER trg_student_before_update
BEFORE UPDATE ON student
FOR EACH ROW
SET NEW.updated_at = NOW();

-- AFTER DELETE 触发器：删除后记录日志
CREATE TRIGGER trg_student_after_delete
AFTER DELETE ON student
FOR EACH ROW
INSERT INTO audit_log (table_name, action, record_id, ts)
VALUES ('student', 'DELETE', OLD.id, NOW());
```

| 时机 | 说明 |
|------|------|
| **BEFORE** | 在数据写入前触发。可修改 NEW 值 |
| **AFTER** | 在数据写入后触发。通常用于日志记录 |
| `FOR EACH ROW` | 每条受影响的行触发一次。MySQL 触发器仅支持行级 |

### 11.2 NEW 和 OLD

| 操作 | NEW | OLD |
|------|-----|-----|
| INSERT | 即将插入的新行 | 无（OLD 不可用） |
| UPDATE | 更新后的新值 | 更新前的旧值 |
| DELETE | 无（NEW 不可用） | 被删除的行 |

```sql
-- 示例：UPDATE 同时引用 NEW 和 OLD
CREATE TRIGGER trg_age_change
BEFORE UPDATE ON student
FOR EACH ROW
BEGIN
    IF NEW.age != OLD.age THEN
        INSERT INTO age_log (student_id, old_age, new_age, ts)
        VALUES (OLD.id, OLD.age, NEW.age, NOW());
    END IF;
END;
```

### 11.3 触发器管理

```sql
SHOW TRIGGERS;                                    -- 列出所有触发器
SHOW CREATE TRIGGER trg_name;                     -- 查看触发器定义
DROP TRIGGER IF EXISTS trg_name;                  -- 删除触发器
```

### 11.4 限制

- 一张表每个事件（BEFORE INSERT、AFTER INSERT、BEFORE UPDATE...）最多只能有一个触发器
- 触发器内不能使用事务控制语句（COMMIT/ROLLBACK）
- 触发器内 RETURN 会退出当前语句块
- 触发器增加 DML 操作开销，高并发写入慎重使用

---

## 十二、事件（定时任务）

**事件（Event）** 是 MySQL 的定时调度机制，在指定时间点或按周期执行 SQL 代码。

### 12.1 开启事件调度器

```sql
-- 查看调度器状态
SHOW VARIABLES LIKE 'event_scheduler';
SELECT @@global.event_scheduler;

-- 开启（重启后失效，永久需修改 my.ini）
SET GLOBAL event_scheduler = ON;
```

### 12.2 创建事件

```sql
-- 一次性事件：2026-07-01 00:00:00 刷新报表
CREATE EVENT evt_daily_report
ON SCHEDULE AT '2026-07-01 00:00:00'
DO
    INSERT INTO report_log (msg, ts) VALUES ('report generated', NOW());

-- 周期性事件：每天凌晨 3 点清理 90 天前的日志
CREATE EVENT evt_clean_log
ON SCHEDULE EVERY 1 DAY STARTS '2026-06-22 03:00:00'
DO
    DELETE FROM audit_log WHERE ts < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- 每 30 分钟执行一次
CREATE EVENT evt_refresh_cache
ON SCHEDULE EVERY 30 MINUTE
DO
    TRUNCATE TABLE t_cache;
    -- INSERT INTO t_cache SELECT ...
```

| 子句 | 说明 |
|------|------|
| `AT '时间点'` | 一次性事件 |
| `EVERY N {SECOND\|MINUTE\|HOUR\|DAY\|MONTH\|YEAR}` | 周期性事件 |
| `STARTS '时间点'` | 首次执行时间 |
| `ENDS '时间点'` | 事件终止时间（过了之后不再执行） |
| `ON COMPLETION PRESERVE` | 事件执行完后保留（默认删除），下次可复用 |

### 12.3 管理事件

```sql
-- 查看事件列表
SHOW EVENTS;
SHOW EVENTS FROM mydb;

-- 查看事件定义
SHOW CREATE EVENT evt_name;

-- 禁用事件（临时暂停）
ALTER EVENT evt_name DISABLE;

-- 启用事件
ALTER EVENT evt_name ENABLE;

-- 修改事件
ALTER EVENT evt_clean_log
ON SCHEDULE EVERY 1 DAY STARTS '2026-06-22 02:00:00';

-- 删除事件
DROP EVENT IF EXISTS evt_name;

---

## 十三、窗口函数

**窗口函数（Window Function）** 在查询结果的某个"窗口"（一组相关行）上执行计算。与 GROUP BY 不同，窗口函数不折叠行——每行保留，聚合值附加在每行上。

### 13.1 基本语法

```sql
SELECT
    列名,
    窗口函数() OVER (
        PARTITION BY 列名    -- 分区（可选）：按此列分组计算
        ORDER BY 列名        -- 排序（必需）：窗口内行的排列顺序
    ) AS 别名
FROM 表名;
```

**窗口函数 vs GROUP BY**：

```sql
-- GROUP BY：折叠为 3 行（每个 class_id 一行）
SELECT class_id, COUNT(*) AS cnt FROM student GROUP BY class_id;

-- 窗口函数：每行保留，COUNT 按 class_id 分区计算后附加
SELECT name, class_id, COUNT(*) OVER (PARTITION BY class_id) AS class_cnt FROM student;
```

| 行为 | GROUP BY | 窗口函数 OVER() |
|------|----------|----------------|
| 结果行数 | 折叠，与分组数相同 | 不折叠，与原表相同 |
| 每一行 | 聚合值一行 | 聚合值附加在原行旁 |

### 13.2 排名函数

准备示例数据：

```sql
-- 假设 student 表数据：
-- id=1, name='张三', class_id=1, score=95
-- id=2, name='李四', class_id=1, score=88
-- id=3, name='王五', class_id=1, score=95
-- id=4, name='赵六', class_id=2, score=76
-- id=5, name='钱七', class_id=2, score=88
```

| 函数 | 行为 | 示例结果（按 score DESC 排名） |
|------|------|------|
| `ROW_NUMBER()` | 连续编号，即使值相同也不重复 | 95→1, 95→2, 88→3, 88→4, 76→5 |
| `RANK()` | 值相同则排名相同，后续排名跳跃 | 95→1, 95→1, 88→3, 88→3, 76→5 |
| `DENSE_RANK()` | 值相同则排名相同，后续排名不跳跃 | 95→1, 95→1, 88→2, 88→2, 76→3 |
| `NTILE(N)` | 将行均分为 N 个桶，返回桶编号 | 分 3 组：1, 1, 2, 2, 3 |

```sql
-- 按 class_id 分区，分区内按 score 降序排名
SELECT
    name,
    class_id,
    score,
    ROW_NUMBER() OVER (PARTITION BY class_id ORDER BY score DESC) AS row_num,
    RANK()       OVER (PARTITION BY class_id ORDER BY score DESC) AS rank_num,
    DENSE_RANK() OVER (PARTITION BY class_id ORDER BY score DESC) AS dense_num
FROM student;

-- 查询每个班分数最高的学生
SELECT * FROM (
    SELECT *,
        RANK() OVER (PARTITION BY class_id ORDER BY score DESC) AS rn
    FROM student
) AS ranked WHERE rn = 1;
```

### 13.3 偏移函数

| 函数 | 说明 |
|------|------|
| `LAG(col, n, default)` | 从当前行向前（向上）找第 n 行。n 默认 1。到达边界时返回 default（可选，默认 NULL） |
| `LEAD(col, n, default)` | 从当前行向后（向下）找第 n 行。n 默认 1。到达边界时返回 default（可选，默认 NULL） |
| `FIRST_VALUE(col)` | 窗口内第一行的值 |
| `LAST_VALUE(col)` | 窗口内最后一行的值（注意：默认帧范围是 RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW，取到的可能是当前行） |
| `NTH_VALUE(col, n)` | 窗口内第 n 行的值（n 从 1 开始） |

```sql
SELECT
    name,
    score,
    LAG(name, 1)  OVER (ORDER BY score DESC) AS prev_name,   -- 上一个人
    LEAD(name, 1) OVER (ORDER BY score DESC) AS next_name,   -- 下一个人
    LAG(score, 1) OVER (ORDER BY score DESC) AS prev_score,  -- 上一个分数
    score - LAG(score, 1, 0) OVER (ORDER BY score DESC) AS diff  -- 分数差值
FROM student;
```

### 13.4 聚合窗口函数

常规聚合函数（SUM、AVG、COUNT、MAX、MIN）均可作为窗口函数：

```sql
-- 每行显示所在班的平均分和总分
SELECT
    name, class_id, score,
    AVG(score) OVER (PARTITION BY class_id) AS class_avg,
    SUM(score) OVER (PARTITION BY class_id) AS class_total,
    score - AVG(score) OVER (PARTITION BY class_id) AS diff_from_avg
FROM student;

-- 累积求和（按 score DESC 排序，逐行累加）
SELECT
    name, score,
    SUM(score) OVER (ORDER BY score DESC) AS running_total,
    SUM(score) OVER (ORDER BY score DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total_explicit
FROM student;
```

### 13.5 窗口帧（Frame）

`ROWS` 或 `RANGE` 子句限定窗口帧范围：

| 帧子句 | 含义 |
|--------|------|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | 从窗口第一行到当前行 |
| `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` | 当前行及前后各 1 行 |
| `ROWS BETWEEN 3 PRECEDING AND CURRENT ROW` | 当前行及前 3 行（滑动窗口） |
| `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | 默认帧（RANGE），同值行视为一组 |

```sql
-- 滑动平均（当前行及前后各 1 行的平均分）
SELECT
    name, score,
    AVG(score) OVER (ORDER BY score
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS moving_avg
FROM student;
```

### 13.6 命名窗口

多个 OVER 使用相同窗口定义时，用 WINDOW 子句提取：

```sql
SELECT
    name, class_id, score,
    RANK()       OVER w AS rank_num,
    AVG(score)   OVER w AS class_avg,
    LAG(score,1) OVER w AS prev_score
FROM student
WINDOW w AS (PARTITION BY class_id ORDER BY score DESC);
```

---

## 十四、CTE（公用表表达式）

**公用表表达式（Common Table Expression，CTE）** 是一个在单条语句内命名的临时结果集，只在该语句执行期间存在。

### 14.1 基本 CTE

```sql
-- 写法：WITH 临时表名 AS (子查询) 后续的 SELECT 引用
WITH class_stats AS (
    SELECT class_id, COUNT(*) AS cnt, AVG(score) AS avg_score
    FROM student
    GROUP BY class_id
)
SELECT c.class_name, cs.cnt, cs.avg_score
FROM class c
JOIN class_stats cs ON c.id = cs.class_id
WHERE cs.cnt >= 3;
```

**多个 CTE 链式定义**：

```sql
WITH
    adult AS (
        SELECT * FROM student WHERE age >= 18
    ),
    class_rank AS (
        SELECT *, RANK() OVER (PARTITION BY class_id ORDER BY score DESC) AS rn
        FROM adult
    )
SELECT * FROM class_rank WHERE rn <= 2;
```

### 14.2 递归 CTE

`WITH RECURSIVE` 生成递推结果，常用于树形结构（部门层级、分类树、评论嵌套）和生成序列。

```sql
-- 生成 1~10 的序列
WITH RECURSIVE seq(n) AS (
    SELECT 1                        -- 初始值
    UNION ALL
    SELECT n + 1 FROM seq WHERE n < 10   -- 递推（直到条件不满足）
)
SELECT * FROM seq;
```

**递归 CTE 结构**：

| 部分 | 说明 |
|------|------|
| 种子查询（非递归部分） | `SELECT 1`，直接执行，结果为初始行集 |
| UNION ALL | 连接种子结果和每次递归的新结果 |
| 递归查询 | `SELECT n + 1 FROM seq WHERE n < 10`，前一步的输出是本次的输入，每次迭代产生新行 |
| 终止条件 | `n < 10`，不满足时递归停止 |

**树形结构示例**（组织架构、菜单、分类等）：

```sql
CREATE TABLE dept (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    parent_id INT               -- 上级部门，顶级为 NULL
);
-- 数据：总公司(id=1, parent_id=NULL)、研发部(id=2, parent_id=1)、前端组(id=5, parent_id=2)...

-- 查总公司及其所有下级
WITH RECURSIVE dept_tree AS (
    -- 种子：总公司
    SELECT id, name, parent_id, 0 AS depth
    FROM dept WHERE id = 1
    UNION ALL
    -- 递归：找直接下级
    SELECT d.id, d.name, d.parent_id, dt.depth + 1
    FROM dept d
    JOIN dept_tree dt ON d.parent_id = dt.id
)
SELECT CONCAT(REPEAT('  ', depth), name) AS dept_path FROM dept_tree;
```

**递归深度限制**：

```sql
-- 限制最大递归深度（防止无限循环）
WITH RECURSIVE cte AS (
    SELECT ... 
    UNION ALL
    SELECT ... FROM cte WHERE cte.depth < 100
)
SELECT * FROM cte;

-- 设置全局最大深度（默认 1000）
SET SESSION cte_max_recursion_depth = 500;
```

### 14.3 CTE vs 子查询 vs 临时表

| | CTE | 子查询 | 临时表 |
|---|-----|--------|--------|
| 可读性 | 好（前置定义，逻辑抽离） | 嵌套深时差 | 一般 |
| 复用 | 同一语句可引用多次 | 需重复写 | 会话内任意次 |
| 生命周期 | 语句执行完即释放 | 语句执行完即释放 | 会话结束或手动删除 |
| 索引 | 不支持 | 不支持 | 支持（可建索引） |
| 递归 | 支持（WITH RECURSIVE） | 不支持 | 不支持 |
| 场景 | 复杂查询拆解、递归 | 简单单次查询 | 多步操作、需索引加速 |

---

## 十五、内置函数大全

### 15.1 字符串函数

| 函数 | 说明 | 示例 | 结果 |
|------|------|------|------|
| `CONCAT(s1, s2, ...)` | 拼接字符串 | `CONCAT('Hello', ' ', 'World')` | `Hello World` |
| `CONCAT_WS(sep, s1, s2, ...)` | 用分隔符拼接 | `CONCAT_WS('-', '2026', '06', '22')` | `2026-06-22` |
| `SUBSTRING(str, start, len)` | 截取子串（start 从 1 开始） | `SUBSTRING('MySQL', 1, 2)` | `My` |
| `LEFT(str, n)` / `RIGHT(str, n)` | 左/右截 n 个字符 | `LEFT('MySQL', 3)` | `MyS` |
| `REPLACE(str, old, new)` | 替换子串 | `REPLACE('a-b-c', '-', '_')` | `a_b_c` |
| `TRIM(str)` | 去首尾空格 | `TRIM('  hello  ')` | `hello` |
| `LTRIM(str)` / `RTRIM(str)` | 去左/右空格 | `LTRIM('  hello')` | `hello` |
| `UPPER(str)` / `LOWER(str)` | 转大写/小写 | `UPPER('mysql')` | `MYSQL` |
| `LENGTH(str)` | 字节长度 | `LENGTH('中文')` | `6`（UTF8MB4，中文每字 3 字节） |
| `CHAR_LENGTH(str)` | 字符数 | `CHAR_LENGTH('中文')` | `2` |
| `INSTR(str, substr)` | 子串首次出现位置 | `INSTR('hello world', 'world')` | `7` |
| `LOCATE(substr, str, pos)` | 从 pos 开始找子串位置 | `LOCATE('o', 'hello', 1)` | `5` |
| `LPAD(str, len, pad)` / `RPAD()` | 左/右填充到指定长度 | `LPAD('5', 3, '0')` | `005` |
| `REVERSE(str)` | 反转字符串 | `REVERSE('abc')` | `cba` |
| `REPEAT(str, n)` | 重复 n 次 | `REPEAT('ab', 3)` | `ababab` |
| `SPACE(n)` | 返回 n 个空格 | `SPACE(5)` | `     ` |
| `FORMAT(num, d)` | 千分位格式化（d=小数位数） | `FORMAT(1234567.89, 2)` | `1,234,567.89` |
| `ASCII(char)` | 字符的 ASCII 码 | `ASCII('A')` | `65` |
| `CHAR(n, ...)` | ASCII 码转字符 | `CHAR(65)` | `A` |

**字符串拼接陷阱**：`CONCAT` 中任一参数为 NULL，整个结果为 NULL。

```sql
SELECT CONCAT('Hello', NULL, 'World');           -- NULL
SELECT CONCAT(IFNULL(middle_name, ''), last_name) FROM student;  -- 用 IFNULL 避免
```

### 15.2 日期时间函数

| 函数 | 说明 | 示例 | 结果 |
|------|------|------|------|
| `NOW()` | 当前日期时间 | `SELECT NOW()` | `2026-06-22 14:30:00` |
| `CURDATE()` | 当前日期 | `SELECT CURDATE()` | `2026-06-22` |
| `CURTIME()` | 当前时间 | `SELECT CURTIME()` | `14:30:00` |
| `DATE(expr)` | 提取日期部分 | `DATE(NOW())` | `2026-06-22` |
| `TIME(expr)` | 提取时间部分 | `TIME(NOW())` | `14:30:00` |
| `YEAR(d)` / `MONTH(d)` / `DAY(d)` | 提取年/月/日 | `YEAR('2026-06-22')` | `2026` |
| `HOUR(t)` / `MINUTE(t)` / `SECOND(t)` | 提取时/分/秒 | `HOUR('14:30:00')` | `14` |
| `DAYOFWEEK(d)` | 星期几（1=周日） | `DAYOFWEEK('2026-06-22')` | `2`（周一） |
| `WEEKDAY(d)` | 星期几（0=周一） | `WEEKDAY('2026-06-22')` | `0` |
| `DAYNAME(d)` | 星期名称 | `DAYNAME('2026-06-22')` | `Monday` |
| `DATE_FORMAT(d, fmt)` | 格式化日期 | `DATE_FORMAT(NOW(), '%Y年%m月%d日')` | `2026年06月22日` |
| `STR_TO_DATE(str, fmt)` | 字符串转日期 | `STR_TO_DATE('2026-06-22', '%Y-%m-%d')` | `2026-06-22` |
| `DATE_ADD(d, INTERVAL n UNIT)` | 日期加 | `DATE_ADD('2026-06-22', INTERVAL 7 DAY)` | `2026-06-29` |
| `DATE_SUB(d, INTERVAL n UNIT)` | 日期减 | `DATE_SUB('2026-06-22', INTERVAL 1 MONTH)` | `2026-05-22` |
| `DATEDIFF(d1, d2)` | 日期差（天数） | `DATEDIFF('2026-07-01', '2026-06-22')` | `9` |
| `TIMESTAMPDIFF(UNIT, t1, t2)` | 精确时间差 | `TIMESTAMPDIFF(HOUR, '09:00', '18:00')` | `9` |
| `EXTRACT(UNIT FROM d)` | 提取指定部分 | `EXTRACT(YEAR FROM '2026-06-22')` | `2026` |
| `UNIX_TIMESTAMP(d)` | 日期转 Unix 时间戳 | `UNIX_TIMESTAMP('2026-06-22')` | `1750896000` |
| `FROM_UNIXTIME(ts)` | Unix 时间戳转日期 | `FROM_UNIXTIME(1750896000)` | `2026-06-22 00:00:00` |
| `LAST_DAY(d)` | 当月最后一天 | `LAST_DAY('2026-06-22')` | `2026-06-30` |

**DATE_FORMAT 常用格式符**：

| 格式符 | 含义 | 格式符 | 含义 | 格式符 | 含义 |
|--------|------|--------|------|--------|------|
| `%Y` | 四位年 | `%m` | 两位月(01~12) | `%d` | 两位日(01~31) |
| `%y` | 两位年 | `%c` | 月(1~12) | `%e` | 日(1~31) |
| `%H` | 24小时制(00~23) | `%h` | 12小时制(01~12) | `%i` | 分钟(00~59) |
| `%s` | 秒(00~59) | `%p` | AM/PM | `%W` | 星期全名 |
| `%a` | 星期缩写 | `%M` | 月份全名 | `%b` | 月份缩写 |
| `%T` | 24小时时间 `HH:MM:SS` | `%r` | 12小时时间 | `%j` | 一年中的天数 |

**实际查询示例**：

```sql
-- 查询近 7 天注册的用户
SELECT * FROM student WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY);

-- 查询本月出生的学生
SELECT * FROM student WHERE MONTH(birthday) = MONTH(CURDATE());

-- 按周统计新增
SELECT DATE_FORMAT(created_at, '%Y-%u') AS week_label, COUNT(*) AS cnt
FROM student GROUP BY week_label;

-- 计算年龄
SELECT name, TIMESTAMPDIFF(YEAR, birthday, CURDATE()) AS age FROM student;
```

### 15.3 数学函数

| 函数 | 说明 | 示例 | 结果 |
|------|------|------|------|
| `ROUND(x, d)` | 四舍五入到 d 位小数 | `ROUND(3.14159, 2)` | `3.14` |
| `CEIL(x)` / `CEILING(x)` | 向上取整 | `CEIL(3.14)` | `4` |
| `FLOOR(x)` | 向下取整 | `FLOOR(3.14)` | `3` |
| `TRUNCATE(x, d)` | 截断到 d 位小数（不舍入） | `TRUNCATE(3.149, 2)` | `3.14` |
| `ABS(x)` | 绝对值 | `ABS(-5)` | `5` |
| `MOD(x, y)` | 取模 | `MOD(17, 5)` | `2` |
| `POW(x, y)` / `POWER(x, y)` | x 的 y 次幂 | `POW(2, 10)` | `1024` |
| `SQRT(x)` | 平方根 | `SQRT(16)` | `4` |
| `RAND()` | 0~1 的随机浮点数 | `SELECT RAND()` | `0.7248...` |
| `RAND(n)` | 指定种子的随机数（n 相同时结果相同） | `SELECT RAND(42)` | 固定值 |
| `SIGN(x)` | 符号（正=1，零=0，负=-1） | `SIGN(-5)` | `-1` |
| `GREATEST(v1, v2, ...)` | 最大值 | `GREATEST(3, 7, 5)` | `7` |
| `LEAST(v1, v2, ...)` | 最小值 | `LEAST(3, 7, 5)` | `3` |

### 15.4 条件函数

| 函数 | 说明 | 示例 | 结果 |
|------|------|------|------|
| `IF(cond, true_val, false_val)` | 简单条件判断 | `IF(age >= 18, '成年', '未成年')` | `成年` 或 `未成年` |
| `IFNULL(val, default)` | val 为 NULL 则返回 default | `IFNULL(phone, '未知')` | phone 的值或 `未知` |
| `NULLIF(v1, v2)` | v1 == v2 时返回 NULL，否则返回 v1 | `NULLIF(0, 0)` | `NULL` |
| `COALESCE(v1, v2, ...)` | 返回第一个非 NULL 的值 | `COALESCE(mobile, phone, 'N/A')` | 第一个有值的列 |
| `CASE WHEN ... THEN ... ELSE ... END` | 多条件分支 | 见下例 | — |

```sql
-- CASE：学生成绩等级
SELECT name, score,
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 'B'
        WHEN score >= 70 THEN 'C'
        WHEN score >= 60 THEN 'D'
        ELSE 'F'
    END AS grade
FROM student;

-- COALESCE 链：取第一个有值的联系方式
SELECT name, COALESCE(mobile, phone, email, 'N/A') AS contact FROM student;

-- NULLIF 典型用途：防止除零错误
SELECT score / NULLIF(total, 0) AS ratio FROM exam;
```

| 函数 | NULL 处理方式 | 使用场景 |
|------|-------------|---------|
| `IFNULL(a, b)` | a 为 NULL → b | 单列空值替换 |
| `COALESCE(a, b, c)` | 返回第一个非 NULL | 多列依次尝试 |
| `NULLIF(a, b)` | a==b → NULL（反直觉） | 除零保护、空字符串转 NULL |

### 15.5 类型转换

| 函数 | 说明 | 示例 |
|------|------|------|
| `CAST(expr AS type)` | SQL 标准转换 | `CAST('123' AS UNSIGNED)` → `123` |
| `CONVERT(expr, type)` | MySQL 风格转换 | `CONVERT('2026-06-22', DATE)` |
| `CONVERT(expr USING charset)` | 字符集转换（注意与类型转换同名不同义） | `CONVERT('中文' USING gbk)` |

```sql
-- 类型转换示例
SELECT CAST('123' AS UNSIGNED) + 10;                            -- 133
SELECT CAST(3.14159 AS DECIMAL(10,2));                          -- 3.14
SELECT CAST('2026-06-22' AS DATE);                              -- 2026-06-22
SELECT CAST(birthday AS CHAR) FROM student;                     -- 日期 → 字符串

-- 隐式转换：字符串与数字比较时字符串被转为数字
SELECT '5' + 2;                                                 -- 7（'5' 隐式转 5）
SELECT 'abc' + 2;                                               -- 2（'abc' 隐式转 0）

-- 隐式转换导致索引失效的陷阱
SELECT * FROM student WHERE phone = 13800138000;                -- phone 是 VARCHAR，全表扫描
SELECT * FROM student WHERE phone = '13800138000';              -- 正确，用索引
```

### 15.6 加密函数

```sql
SELECT MD5('password');                                         -- 32 位十六进制（128 位）
SELECT SHA2('password', 256);                                   -- 64 位十六进制（256 位）
-- SHA2 的第二个参数：224 / 256 / 384 / 512
-- 不推荐用 MD5/SHA1 存密码，偏好应用层 bcrypt/argon2

-- AES 加解密
SELECT TO_BASE64(AES_ENCRYPT('机密数据', 'encryption_key'));   -- 加密后 Base64 输出
SELECT AES_DECRYPT(FROM_BASE64('...'), 'encryption_key');       -- 解密
```

### 15.7 信息函数

| 函数 | 返回值 |
|------|--------|
| `VERSION()` | MySQL 版本号 |
| `DATABASE()` | 当前数据库名 |
| `USER()` / `CURRENT_USER()` | 连接用户 / 当前权限用户 |
| `CONNECTION_ID()` | 当前连接 ID（用于 KILL） |
| `LAST_INSERT_ID()` | 最近 INSERT 生成的自增 ID |
| `ROW_COUNT()` | 上条 DML 影响的行数 |
| `FOUND_ROWS()` | SELECT 本应返回的行数（配合 `SQL_CALC_FOUND_ROWS` 使用） |
| `UUID()` | 生成 UUID（36 字符） |
| `UUID_SHORT()` | 生成短 UUID（64 位整数） |

```sql
INSERT INTO student (name) VALUES ('张三');
SELECT LAST_INSERT_ID();                  -- 获取刚插入行的自增 ID

SELECT CONNECTION_ID();                   -- 获取当前连接 ID，用于 KILL CONNECTION_ID()

-- 获取影响行数（配合 SQL_CALC_FOUND_ROWS 忽略 LIMIT）
SELECT SQL_CALC_FOUND_ROWS * FROM student LIMIT 10;
SELECT FOUND_ROWS();                      -- 返回 LIMIT 前的总行数

---

## 十六、锁机制与并发

InnoDB 是 MySQL 默认存储引擎，支持行级锁和事务。这里只讨论 InnoDB 的锁。

### 16.1 锁粒度

| 级别 | 锁对象 | 并发度 | 开销 |
|------|--------|:----:|------|
| **行锁（Row Lock）** | 单行记录 | 高 | 锁管理开销大 |
| **表锁（Table Lock）** | 整张表 | 低 | 锁管理开销小 |
| **页锁（Page Lock）** | 数据页 | 中 | BDB 引擎使用，极少见 |

InnoDB 默认使用行锁，通过索引实现。没有索引的查询会锁全表（升级为表锁）。

### 16.2 行锁类型

| 锁类型 | 符号 | 说明 | 冲突 |
|--------|------|------|:--:|
| **共享锁（S Lock）** | S | 允许事务读取一行数据 | 与 X 锁冲突，与 S 锁兼容 |
| **排他锁（X Lock）** | X | 允许事务更新或删除一行数据 | 与 S 锁和 X 锁均冲突 |
| **意向共享锁（IS Lock）** | IS | 表级锁，表示事务打算在表中的某些行加 S 锁 | 与 X 表锁冲突 |
| **意向排他锁（IX Lock）** | IX | 表级锁，表示事务打算在表中的某些行加 X 锁 | 与 S/X 表锁冲突 |

意向锁是表级锁，用于在表级快速判断表中是否有行锁，避免逐行检查。

### 16.3 显式加锁

```sql
-- 排他锁：查询行并锁定，阻止其他事务修改或加排他锁，直到当前事务提交
SELECT * FROM student WHERE id = 1 FOR UPDATE;

-- 共享锁：查询行并加共享锁，允许其他事务读，但不允许修改
SELECT * FROM student WHERE id = 1 FOR SHARE;      -- MySQL 8.0+
SELECT * FROM student WHERE id = 1 LOCK IN SHARE MODE;  -- MySQL 8.0 前

-- 跳过锁：读数据时不等待锁，只读已提交的行
SELECT * FROM student FOR UPDATE SKIP LOCKED;      -- 跳过被锁定的行
SELECT * FROM student FOR UPDATE NOWAIT;           -- 遇到锁立即报错，不等待
```

| 子句 | 行为 |
|------|------|
| `FOR UPDATE` | 加排他锁，其他事务的 UPDATE/DELETE/SELECT FOR UPDATE 阻塞 |
| `FOR SHARE` | 加共享锁，其他事务可读但不可写 |
| `SKIP LOCKED` | 跳过已被其他事务锁定的行，常用于任务队列 |
| `NOWAIT` | 遇到锁不等待，直接报错 |

### 16.4 死锁

两个或多个事务互相等待对方持有的锁，形成循环等待，谁也无法继续。

```sql
-- 事务 A                               -- 事务 B
START TRANSACTION;                      START TRANSACTION;
UPDATE student SET age=20 WHERE id=1;   -- A 锁住 id=1
                                        UPDATE student SET age=21 WHERE id=2;  -- B 锁住 id=2
UPDATE student SET age=22 WHERE id=2;   -- A 等 B 释放 id=2 的锁
                                        UPDATE student SET age=23 WHERE id=1;  -- B 等 A 释放 id=1 的锁
-- 死锁！InnoDB 自动检测并回滚其中一个事务
```

**避免死锁的策略**：

- 以相同顺序访问资源。如上例两事务都以 id ASC 顺序更新，则不会死锁
- 保持事务尽可能短，减少锁持有时间
- 对于热点数据，用 `SELECT ... FOR UPDATE` 先锁住所有需要的行
- 对允许失败的操作，捕获死锁异常并重试

**查看锁状态**：

```sql
SHOW ENGINE INNODB STATUS\G                  -- 包含最近死锁日志
SELECT * FROM performance_schema.data_locks;  -- 当前持有的锁（MySQL 8.0+）
SELECT * FROM performance_schema.data_lock_waits;  -- 当前锁等待（MySQL 8.0+）
```

### 16.5 元数据锁（MDL）

MDL 保护表结构不被并发修改。访问表时自动加 MDL 共享锁，ALTER TABLE 时加 MDL 排他锁。DDL 操作会排队等待所有 MDL 共享锁释放，阻塞期间该表的所有读写也排队。

```sql
-- 查看 MDL 锁等待
SELECT * FROM performance_schema.metadata_locks;
```

---

## 十七、性能调优基础

### 17.1 慢查询日志

```sql
-- 启用慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 2;              -- 超过 2 秒的查询被记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';

-- 查看当前慢查询数量
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

**mysqldumpslow 分析日志**：

```bash
mysqldumpslow -s t -t 10 slow.log           # 按时间排序，显示 top 10
mysqldumpslow -s c -t 20 slow.log           # 按出现次数排序，显示 top 20
mysqldumpslow -g 'student' slow.log         # 只分析含 student 的查询
```

### 17.2 EXPLAIN ANALYZE

MySQL 8.0.18+ 的 EXPLAIN ANALYZE 执行查询并输出实际执行时间和行数，而非估算值。

```sql
EXPLAIN ANALYZE SELECT * FROM student WHERE name = '张三';
-- 输出包含每一步的实际时间（毫秒）和实际行数
```

### 17.3 关键配置参数

| 参数 | 说明 | 建议值 |
|------|------|--------|
| `innodb_buffer_pool_size` | InnoDB 缓存池大小（数据 + 索引） | 物理内存的 60%~80% |
| `max_connections` | 最大连接数 | 视业务量，通常 100~500 |
| `innodb_flush_log_at_trx_commit` | 事务日志刷新策略 | 1=最安全（每次提交刷盘），2=性能优先 |
| `sync_binlog` | 二进制日志刷盘策略 | 1=最安全（每次事务刷盘），N=缓冲 N 次刷盘 |
| `join_buffer_size` | JOIN 缓冲区大小 | 默认 256KB，复杂 JOIN 可调大 |
| `sort_buffer_size` | 排序缓冲区大小 | 默认 256KB，大量排序可调大（会话级分配，慎） |
| `tmp_table_size` | 内存临时表最大尺寸 | 超过则转为磁盘临时表 |
| `innodb_log_file_size` | 重做日志文件大小 | 越大写性能越好但恢复越慢 |
| `innodb_io_capacity` | InnoDB 后台 I/O 能力 | SSD 可设 2000+，HDD 设 200 左右 |

### 17.4 查询性能分析

```sql
-- 查看当前连接的线程状态
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;                -- SQL 文本不截断

-- 杀掉指定连接
KILL <connection_id>;                 -- 回滚事务并断开
KILL QUERY <connection_id>;           -- 仅终止当前查询，不断开连接

-- 查看服务器运行状态
SHOW STATUS LIKE 'Threads_connected'; -- 当前连接数
SHOW STATUS LIKE 'Threads_running';   -- 正在执行查询的线程数
SHOW STATUS LIKE 'Innodb_row_lock%';  -- 行锁相关指标
SHOW STATUS LIKE 'Created_tmp%';      -- 临时表创建次数
SHOW STATUS LIKE 'Select_scan';       -- 全表扫描次数
```

### 17.5 优化器提示

强制指定索引或 JOIN 顺序（慎用）：

```sql
-- 强制使用指定索引
SELECT * FROM student USE INDEX (idx_name) WHERE name = '张三';

-- 强制使用指定索引（忽略其他可能的索引）
SELECT * FROM student FORCE INDEX (idx_name) WHERE name = '张三';

-- 指定 JOIN 顺序（驱动表写在前面）
SELECT STRAIGHT_JOIN s.name, c.class_name
FROM student s JOIN class c ON s.class_id = c.id;
```

---

## 十八、JSON 操作

MySQL 5.7+ 引入 JSON 数据类型，支持自动校验格式和高效的 JSON 操作。

### 18.1 JSON 数据类型

```sql
CREATE TABLE t_config (
    id INT PRIMARY KEY,
    profile JSON
);

INSERT INTO t_config VALUES
(1, '{"name": "张三", "age": 20, "skills": ["Java", "Python"]}'),
(2, '{"name": "李四", "age": 22, "contact": {"email": "lisi@test.com"}}');

-- 插入非法 JSON 时自动报错
INSERT INTO t_config VALUES (3, 'not json');   -- ERROR 3140
```

### 18.2 JSON 路径查询

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `->` | 按路径提取 JSON 值（保留 JSON 格式） | `profile->'$.name'` → `"张三"` |
| `->>` | 按路径提取 JSON 值（取消引号，转为字符串） | `profile->>'$.name'` → `张三` |
| `$` | 根路径 | `profile->'$'` → 整个 JSON |
| `$.key` | 成员访问 | `profile->'$.name'` |
| `$[n]` | 数组索引（从 0 开始） | `profile->'$.skills[0]'` → `"Java"` |
| `$[*]` | 数组所有元素 | `profile->'$.skills[*]'` → `["Java", "Python"]` |
| `$.**` | 通配搜索（递归） | `profile->'$**.email'` → 查找任意深度的 email 字段 |

```sql
SELECT
    profile->>'$.name' AS name,
    profile->>'$.age' AS age,
    profile->>'$.skills[0]' AS first_skill,
    profile->>'$.contact.email' AS email
FROM t_config;

-- 结果：
-- name='张三', age='20', first_skill='Java', email=NULL
-- name='李四', age='22', first_skill=NULL, email='lisi@test.com'
```

### 18.3 JSON 函数

| 函数 | 说明 | 示例 |
|------|------|------|
| `JSON_EXTRACT(doc, path)` | 等价于 `->` | `JSON_EXTRACT(profile, '$.name')` |
| `JSON_UNQUOTE(expr)` | 去掉 JSON 字符串的引号 | `JSON_UNQUOTE(profile->'$.name')` |
| `JSON_SET(doc, path, val, ...)` | 插入/更新 JSON 中的值 | `JSON_SET(profile, '$.age', 21)` |
| `JSON_INSERT(doc, path, val, ...)` | 仅插入（键不存在才加） | `JSON_INSERT(profile, '$.weight', 60)` |
| `JSON_REPLACE(doc, path, val, ...)` | 仅替换（键存在才改） | `JSON_REPLACE(profile, '$.name', '王五')` |
| `JSON_REMOVE(doc, path, ...)` | 删除指定路径的值 | `JSON_REMOVE(profile, '$.skills')` |
| `JSON_ARRAY(val, ...)` | 创建 JSON 数组 | `JSON_ARRAY(1, 2, 'three')` → `[1, 2, "three"]` |
| `JSON_OBJECT(key, val, ...)` | 创建 JSON 对象 | `JSON_OBJECT('name', '张三', 'age', 20)` |
| `JSON_CONTAINS(doc, val)` | 判断 JSON 是否包含值 | `JSON_CONTAINS(profile, '"张三"', '$.name')` |
| `JSON_CONTAINS_PATH(doc, 'one'/'all', path, ...)` | 检查路径是否存在 | `JSON_CONTAINS_PATH(profile, 'one', '$.email')` |
| `JSON_LENGTH(doc)` | JSON 数组长度或对象键数 | `JSON_LENGTH(profile->'$.skills')` → `2` |
| `JSON_KEYS(doc)` | 获取 JSON 对象的所有键 | `JSON_KEYS(profile)` → `["name", "age", "skills"]` |
| `JSON_ARRAY_APPEND(doc, path, val)` | 向数组追加元素 | `JSON_ARRAY_APPEND(profile, '$.skills', 'Go')` |
| `JSON_TYPE(doc)` | 返回 JSON 值类型 | `JSON_TYPE(profile->'$.age')` → `INTEGER` |
| `JSON_VALID(str)` | 检查字符串是否为合法 JSON | `JSON_VALID('[1,2]')` → `1` |

```sql
-- 更新 JSON 列
UPDATE t_config SET profile = JSON_SET(profile, '$.age', 21) WHERE id = 1;

-- 向 skills 数组追加一项
UPDATE t_config SET profile = JSON_ARRAY_APPEND(profile, '$.skills', 'Go') WHERE id = 1;

-- 查询 JSON 数组包含某值的行
SELECT * FROM t_config WHERE JSON_CONTAINS(profile->'$.skills', '"Java"');

-- 与普通字段结合
SELECT profile->>'$.name' AS name
FROM t_config
WHERE CAST(profile->>'$.age' AS UNSIGNED) > 20;
```

### 18.4 JSON 索引

JSON 列本身不能直接建索引，通过**生成列（Generated Column）** + 普通索引实现：

```sql
-- 基于 JSON 字段创建生成列
ALTER TABLE t_config ADD COLUMN name VARCHAR(50)
    GENERATED ALWAYS AS (profile->>'$.name') STORED;

-- 在生成列上建索引
ALTER TABLE t_config ADD INDEX idx_json_name (name);

-- 查询即可使用索引
SELECT * FROM t_config WHERE name = '张三';
```

**多值索引（MySQL 8.0.17+）**：对 JSON 数组建索引，针对 `MEMBER OF` 查询优化：

```sql
CREATE TABLE t_post (
    id INT PRIMARY KEY,
    tags JSON
);
INSERT INTO t_post VALUES (1, '["mysql", "database"]'), (2, '["python", "web"]');

-- 创建多值索引
ALTER TABLE t_post ADD INDEX idx_tags ((CAST(tags AS CHAR(20) ARRAY)));

-- 使用 MEMBER OF 查询（利用多值索引）
SELECT * FROM t_post WHERE 'mysql' MEMBER OF(tags);
```

---

## 十九、全文索引

**全文索引（FULLTEXT Index）** 用于文本内容的快速搜索，比 `LIKE '%keyword%'` 高效得多。

### 19.1 创建全文索引

```sql
-- 建表时创建
CREATE TABLE article (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT idx_ft_content (content)
) ENGINE=InnoDB;

-- 在已有表上加全文索引
ALTER TABLE article ADD FULLTEXT idx_ft_title_content (title, content);

-- 插入数据
INSERT INTO article (title, content) VALUES
('MySQL入门', 'MySQL是最流行的开源关系型数据库管理系统'),
('Python教程', 'Python是一种解释型、面向对象的编程语言'),
('数据分析', '使用Python和MySQL进行数据分析和处理');
```

### 19.2 全文搜索模式

**自然语言模式（Natural Language Mode）**：

```sql
-- 返回按相关度排序的结果
SELECT *, MATCH(title, content) AGAINST('MySQL 数据库') AS relevance
FROM article
WHERE MATCH(title, content) AGAINST('MySQL 数据库' IN NATURAL LANGUAGE MODE)
ORDER BY relevance DESC;
```

**布尔模式（Boolean Mode）**：支持操作符精确控制：

```sql
-- 包含 MySQL 且不包含 PostgreSQL
SELECT * FROM article
WHERE MATCH(content) AGAINST('+MySQL -PostgreSQL' IN BOOLEAN MODE);

-- 包含 MySQL 或 Python，按相关度排序
SELECT *, MATCH(content) AGAINST('MySQL Python' IN BOOLEAN MODE) AS score
FROM article
WHERE MATCH(content) AGAINST('MySQL Python' IN BOOLEAN MODE);
```

| 操作符 | 说明 | 示例 |
|--------|------|------|
| `+` | 必须包含该词 | `+MySQL` |
| `-` | 必须不包含该词 | `-PostgreSQL` |
| `>` / `<` | 提高/降低词的相关度权重 | `>MySQL <tutorial` |
| `*` | 通配符（词尾） | `datab*` 匹配 database、databases |
| `"..."` | 短语匹配 | `"relational database"` |
| `()` | 分组 | `(+MySQL +Python) (+Java -Kotlin)` |
| `~` | 否定操作符，降低相关度 | `~MySQL` |

**查询扩展模式（Query Expansion）**：执行两次搜索，第二次用第一次找到的最相关词扩展查询。

```sql
SELECT * FROM article
WHERE MATCH(content) AGAINST('database' WITH QUERY EXPANSION);
```

### 19.3 全文索引 vs LIKE

| | FULLTEXT | LIKE '%keyword%' |
|---|:---:|:---:|
| 搜索方式 | 倒排索引 | 全表扫描 |
| 大表性能 | 快（毫秒级） | 慢（秒级或更多） |
| 相关度排序 | 支持 | 不支持 |
| 多词搜索 | 支持布尔操作符 | 需用多个 LIKE 拼接 |
| 最短词长 | `innodb_ft_min_token_size`（默认 3） | 无限制 |
| 停用词 | 忽略常见词 | 不忽略 |

**中文分词**：InnoDB 默认分词器按空格分词，对中文无效。需使用 **ngram parser**：

```sql
-- 在全文索引中指定 ngram 分词器
ALTER TABLE article ADD FULLTEXT idx_ft_content (content) WITH PARSER ngram;

-- 配置 ngram token 大小（默认 2，常见设为 2~3）
SET GLOBAL ngram_token_size = 2;
```

---

## 二十、表维护

### 20.1 常用维护命令

| 命令 | 说明 | 适用引擎 |
|------|------|---------|
| `ANALYZE TABLE` | 更新索引统计信息（影响优化器选择） | InnoDB/MyISAM |
| `OPTIMIZE TABLE` | 整理碎片、重建表 | InnoDB/MyISAM |
| `CHECK TABLE` | 检查表是否有错误 | InnoDB/MyISAM |
| `REPAIR TABLE` | 修复损坏的表 | MyISAM（InnoDB 不支持） |

```sql
ANALYZE TABLE student;                              -- 更新统计信息
OPTIMIZE TABLE student;                             -- 重组表空间
CHECK TABLE student;                                -- 检查表状态
```

### 20.2 表信息查询

```sql
-- 查看表状态（行数、数据大小、索引大小、碎片等）
SHOW TABLE STATUS LIKE 'student'\G

-- 查看表的列信息
SHOW COLUMNS FROM student;

-- 查看表的所有索引
SHOW INDEX FROM student;

-- 查看建表语句
SHOW CREATE TABLE student\G

-- 从 information_schema 查询（更灵活）
SELECT TABLE_NAME, TABLE_ROWS, DATA_LENGTH, INDEX_LENGTH,
       (DATA_FREE / (DATA_LENGTH + INDEX_LENGTH)) * 100 AS fragmentation_pct
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'student';
```

| 字段 | 含义 |
|------|------|
| `TABLE_ROWS` | InnoDB 的估算行数（误差可能较大） |
| `DATA_LENGTH` | 数据所占磁盘空间（字节） |
| `INDEX_LENGTH` | 索引所占磁盘空间（字节） |
| `DATA_FREE` | 已分配但未使用的空间（碎片指标，值大时考虑 OPTIMIZE） |
| `AUTO_INCREMENT` | 下一个自增值 |
| `CREATE_TIME` | 表创建时间 |
| `UPDATE_TIME` | 表最后更新时间 |

### 20.3 各存储引擎简介

| 引擎 | 事务 | 行锁 | 外键 | 全文 | 适用场景 |
|------|:---:|:---:|:---:|:---:|------|
| **InnoDB** | ✓ | ✓ | ✓ | ✓（5.6+） | 通用 OLTP，99% 场景，默认引擎 |
| **MyISAM** | ✗ | ✗ | ✗ | ✓ | 只读或极少写入、全文搜索（旧版 MySQL） |
| **MEMORY** | ✗ | ✗ | ✗ | ✗ | 临时表、缓存数据，重启后数据丢失 |
| **ARCHIVE** | ✗ | ✗ | ✗ | ✗ | 归档/日志存储，高压缩比，只支持 INSERT 和 SELECT |

```sql
-- 查看表使用哪个引擎
SHOW TABLE STATUS LIKE 'student';
-- 修改表的存储引擎
ALTER TABLE student ENGINE = InnoDB;

-- 查看所有支持的引擎
SHOW ENGINES;
```

### 20.4 临时表

```sql
-- 创建临时表（会话私有，会话结束自动删除）
CREATE TEMPORARY TABLE tmp_result AS
SELECT class_id, AVG(score) AS avg_score
FROM student
GROUP BY class_id;

-- 临时表与永久表同名时，优先访问临时表
-- 会话结束或断开连接后自动删除
DROP TEMPORARY TABLE IF EXISTS tmp_result;
```

- 临时表对其他会话不可见
- 临时表在连接断开后自动删除
- 临时表物理存储在临时文件目录（`tmpdir`）
```
```
