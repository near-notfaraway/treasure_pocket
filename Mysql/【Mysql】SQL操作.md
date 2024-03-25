# 【Mysql】SQL 操作

* [【Mysql】SQL 操作](#mysqlsql操作)
    * [操作类型](#操作类型)
    * [表结构管理](#表结构管理)
    	* [表查询](#表查询)
    	* [表创建](#表创建)
    	* [表修改](#表修改)
    * [记录管理](#记录管理)
    	* [事务提交](#事务提交)
    	* [记录查询](#记录查询)
    	* [记录操作](#记录操作)
    * [系统管理](#系统管理)
    	* [授权操作](#授权操作)
    	* [元信息查询](#元信息查询)

## 操作类型
SQL 的操作类型可主要分为以下四种：
- **DQL（Data Query Language）**：数据查询语言，用于查询记录

  主要的语句结构：`SELECT 子句 + FROM 子句 + WHERE 子句`

- **DML（Data Manipulation Language）**：数据操纵语言，用于插入删除更新记录

  主要的语句结构：
  - 插入，`INSERT INTO 子句 + VALUES 子句`
  - 删除，`DELETE FROM 子句 + WHERE 子句`
  - 更新，`UPDATE 子句 + SET 子句`
  - 加锁，`SELECT 子句 FOR UPDATE`
  
- **DDL（Data Definition Language）**：数据定义语言，用于定义数据库中的各种对象

  主要动作包括：创建 `CREATE`、修改 `ALTER`、删除 `DROP`

  主要对象包括：库 `DATABASE`、表 `TABLE`、索引 `INDEX`、视图 `VIEW`、同义词 `SYN`、聚簇 `CLUSTER`

- **DCL（Data Contorl Language）**：数据控制语言，用于数据库授权、事务控制等

  主要动作包括：授权 `GRANT` 和 `REVOKE`、事务控制 `COMMIT` 和 `ROLLBACK`

## 表结构管理
### 表查询
``` sql
# 显示指定表的字段信息
DESC 表名;

# 显示指定表的创建语句，包含字段、索引、引擎等信息
SHOW CREATE TABLE 表名;
```

### 表创建
``` sql
CREATE TABLE 表名（
    列名 数据类型 其他约束 COMMENT 备注,
    列名 数据类型 其他约束 COMMENT 备注,
    ... ,
    PRIMARY KEY (列名）,
    KEY 索引名 (列名) ,
    UNIQUE KEY 索引名 (列名)
）ENGINE=引擎 DEFAULT CHARACTER=字符集
```

列定义中除数据类型外的其他约束：
``` sql
# 备注信息
COMMENT '备注'
        
# 非空
NOT NULL

# 默认值
DEFAULT 值           

# 自动填充，当数据长度未达到规定长度，则在数据前面自动填充0
ZEROFILL

# 自增长，每张表有且只有一个，必须是索引
AUTO_INCREMENT

# 唯一，值允许为空但不能重复
UNIQUE
```

可以通过定义键来创建索引，因此键也可以看作包含约束的索引：
``` sql
# 主键，每张表有且只有一个主键，非空，唯一，整数类型
## 单主键
PRIMARY KEY (列名)
## 复合主键
PRIMARY KEY (列名,列名)

# 普通索引
KEY 索引名 (列名)

# 唯一索引
## 单列唯一
UNIQUE KEY 索引名 (列名)
## 联合唯一
UNIQUE KEY 索引名 (列名,列名)

# 外键
CONSTRAINT 外键名 FOREIGN KEY (列名) REFERENCES 关联表名 (关联列名)
```
> 键（KEY）和索引（INDEX）虽然都是数据库的物理结构，但键包含两层意义：一是约束（CONSTRAINT），用来约束和规范数据库的结构完整性，二是索引，用来辅助查询
> 
> 软删除与唯一约束冲突的解决：
> 先添加一个 `deleted_at` 列，存放记录被删除的时间，再将原本用于唯一索引的列，改为和 `delete_at` 列联合唯一

### 表修改
**修改表中的列**

``` sql
# 创建列
ALTER TABLE 表名 ADD 列名 数据类型 其他约束;

# 修改列（不包括列名）
ALTER TABLE 表名 MODIFY 列名 数据类型 其他约束;

# 修改列（包括列名）
ALTER TABLE 表名 CHANGE 原列名 新列名 数据类型 其他约束;

# 删除列
ALTER TABLE 表名 DROP 列名;
```

**修改表中的索引**

``` sql
# 创建索引
ALTER TABLE 表名 ADD 索引类型 索引名(字段名);

# 删除索引
ALTER TABLE 表名 DROP INDEX 索引名;
## 或者
DROP INDEX 索引名 ON 表名;
```

通过 SQL 创建索引时，所使用的索引类型：
- **INDEX**：
    - **普通索引**：仅用于辅助查询，没有约束的索引
    - **组合索引**：实质上是将多个字段建到一个普通索引里，`INDEX (字段1, 字段2, ...)`
- **UNIQUE**：唯一索引，除了 NULL 不可以重复
- **PROMARY KEY**：主键所关联的索引，不允许为 NULL 且不可以重复
- **FULLTEXT INDEX**：全文索引，可以针对值中的某个单词，但效率较低

> 索引虽然能大大地提高查询速度，但仍不可滥用，因为索引会降低 DML 的执行速度，因为当增加、删除、修改记录时，MySQL 不仅要保存数据，还要保存索引文件

## 记录管理
### 事务提交
Mysql MyISAM 存储引擎不支持事务，而对于 Mysql InnoDB 存储引擎，客户端的所有 SQL 都在事务中执行，当没有显式开启事务时，Mysql 会隐式地自动开启事务

因此需要提交后 SQL 才能真正作用到数据库，而提交的方式有以下三种：
- **自动提交**：默认设置，或通过 `SET AUTOCOMMIT ON` 设置，在每个 SQL 执行后，系统将自动进行提交
- **显式提交**：使用 `COMMIT` 命令直接完成的提交
- **隐式提交**：用 SQL 语句间接完成的提交，这些语句包括 `ALTER`、`AUDIT`、`COMMENT`，`CONNECT`，`CREATE`，`DISCONNECT`，`DROP`，`EXIT`，`GRANT`，`NOAUDIT`，`QUIT`，`REVOKE`，`RENAME`

### 记录查询
**普通查询**
``` sql
# 查询记录的所有列
SELECT * FROM 表名 WHERE 条件;

# 查询记录的指定列
SELECT 列名1,列名2,... FROM 表名 WHERE 条件;

# 查询记录的指定列并去重
SELECT DISTINCT 列名1,列名2,... FROM 表名 WHERE 条件;

# 查询记录并根据指定列进行升序排列
SELECT * FROM 表名 WHERE 条件 ORDER BY 列名1,列名2,...;

# 查询记录并根据指定列进行降序排列
SELECT * FROM 表名 WHERE 条件 ORDER BY 列名1,列名2,... DESC;

# 查询记录并限制其查询记录数为 N
SELECT * FROM 表名 WHERE 条件 LIMIT N;
```

**联合查询**
``` sql
# 查询表1的记录，并根据条件内联合表2的记录
# INNER JOIN 表示内联合，输出两表间存在关联的记录
SELECT * FROM 表名1 INNER JOIN 表名2 ON 联合条件;

# 可以使用表的别名，减少条件的复杂度
SELECT * FROM 表名1 a INNER JOIN 表名2 b ON a.列名 = b.列名;

# 查询表1的记录，并根据条件左联合表2的记录
# LEFT JOIN 表示左联合，根据左表输出所有记录，若右表不存在关联则用 null 代替
SELECT * FROM 表名1 LEFT JOIN 表名2 ON 联合条件;

# 查询表1的记录，并根据条件右联合表2的记录
# RIGHT JOIN 表示右联合，根据右表输出所有记录，若左表不存在关联则用 null 代替
SELECT * FROM 表名1 RIGHT JOIN 表名2 ON 联合条件;

# 查询表1的记录，并根据条件完全联合表2的记录
# FULL JOIN 表示完全联合，输出两表的所有记录，若任一表不存在关联则用 null 代替
SELECT * FROM 表名1 FULL JOIN 表名2 ON 联合条件;
```

**子查询**
``` sql
# 通过子查询结果作为查询条件
SELECT * FROM 表名 WHERE 列名 = (结果为单行的子查询);
SELECT * FROM 表名 WHERE 列名 IN (结果为多行的子查询);

# 在子查询中，使用目标表的列作为条件
SELECT * FROM 表名1 WHERE 列名1 in (SELECT 列名A FROM 表名2 WHERE 列名B=表名1.列名2);
```

**聚合和分组查询**
``` sql
# 查询记录通过聚合函数计算的结果
SELECT 聚合函数 FROM 表名 WHERE 条件;

# GROUP BY 表示分组，结果的每个列必须为分组列或聚合函数的结果
SELECT 分组列或聚合函数 FROM 表名 GROUP BY 分组列名1,分组列名2,...

# Having 表示对分组后的结果进行筛选
SELECT 分组列或聚合函数 FROM 表名 GROUP BY 分组列名1,分组列名2,... HAVING 条件

# 通过分组和分组后筛选，再结合 `COUNT` 聚合函数来计数，可以查询指定列存在重复的记录
SELECT 列名1,列名2 FROM 表名 GROUP BY 列名1,列名2 HAVING COUNT(列名1或2) > 1;
```

**常用的聚合函数**
``` sql
# 返回记录或非 NULL 列值的数目
COUNT(* 或列名)

# 返回非 NULL 列值的和
SUM(列名)

# 返回非 NULL 列值中的最大值
MAX(列名)

# 返回非 NULL 列值中的最小值
MIN(列名)

# 返回非 NULL 列值中的平均值
AVG(列名)
```

**JSON函数**

[参考文档](https://www.sjkjc.com/mysql-ref/)

``` sql
# 条件，表示指定列保存的 JSON 列表中包含指定值
JSON_CONTAINS(列名, '"值"')

# 条件，表示指定列的指定路径中，所保存的 JSON 列表中包含指定值
# 路径表达式，以 $ 表示列本身，以 .X 表示子字段，[X] 表示子元素
JSON_CONTAINS(列名, '"值"', '路径表达式')
```

### 记录操作
**插入记录**
``` sql
# 不指定列名，需要输入和列相同数量的列值，对于自动生成值的列输入 null
INSERT INTO 表名 VALUES (列值1，列值2，...),(),...;

# 指定列名，可以忽略自动生成值的列
INSERT INTO 表名 (字段名1，字段名2，...) VALUES(值1，值2，...)(),...;
```

**更新记录**
``` sql
UPDATE 表名 SET 列名1=值1,列名2=值2,... WHERE 条件;
```

**删除记录**
``` sql
DELETE FROM 表名 WHERE 条件;
```

## 系统管理
### 授权操作
**创建用户**
``` sql
CREATE USER 用户名[@'地址'] IDENTIFIED BY '密码';
```

**授予权限**
``` sql
GRANT 权限 ON 表.列 TO 用户@'地址' IDENTIFIED BY '密码';
```

**回收权限**
``` sql
REVOKE 权限 ON 表.列 FROM 用户@'地址';
```

**删除用户**
``` sql
DROP USER 用户@'地址';
```

### 元信息查询
**查看线程信息**

包括当前运行的线程，及其对应连接信息、执行命令和状态
``` sql
SHOW PROCESSLIST;
```

**查看表信息**

包括表的记录行数、数据大小、索引大小
``` sql
SELECT * FROM information_schema.TABLES;
```