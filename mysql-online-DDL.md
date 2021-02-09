---
title: mysql-online-DDL
date: 2020-11-29 22:00:39
tags:
---


# mysql 在线DDL 操作

SQL 语句主要可以划分为以下 3 个类别。

- **DDL（Data Definition Languages）** 语句：数据定义语言，这些语句定义了不同的数据段、数据库、表、列、索引等数据库对象的定义。常用的语句关键字主要包括 create、drop、alter等。

- **DML（Data Manipulation Language）**  语句：数据操纵语句，用于添加、删除、更新和查询数据库记录，并检查数据完整性，常用的语句关键字主要包括 insert、delete、udpate 和select 等。(增添改查）

- **DCL（Data Control Language）**  语句：数据控制语句，用于控制不同数据段直接的许可和访问级别的语句。这些语句定义了数据库、表、字段、用户的访问权限和安全级别。主要的语句关键字包括 grant、revoke 等。


---

**Online DDL** 操作，MySQL5.6以上支持，相较于一般DDL，它在实现修改表结构的同时，依然允许DML操作（SELECT,INSERT,UPDATE,DELETE）。

Onlne DDL 具体过程为 

1. 拿到MDL（metadata lock） 写锁; 

2. 降级为MDL 读锁； 

3. 做真正的DDL; 
4. 升级为MDL写锁；
5. 释放MDL 锁
<font color=red>
* 这里需要注意
alter table 前判断是否有未提交的事务， 或者增加超时时间， 
如果有未提交的事务会出现Waiting for table metadata lock， 导致对表的任何操作都无法执行（包括读），如果是生产环境会造成灾难性的后果
。可以通过设置，超时时间来避免这种问题
</font>

```sql  
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ...
```

Online DDL主要有两种方式：IN PLACE和COPY。

IN PLACE：直接在原表上进行修改，相比于COPY方式可以避免重建表带来的IO和CPU消耗，有更好的性能并支持并发DML操作
COPY：创建修改后的临时表，然后将原表的数据复制到临时表，执行期间不允许并发DML写操作，否则会导致脏数据。
在MySQL之前，我们一般使用COPY的方式，借助临时表，手动修改。

需要注意的是：并不是所有的Online DDL操作都支持IN PLACE方式。

## MySQL InnoDB数据存储方式
在MySQL中，一张表的数据分为两种，一种是结构数据，记录者站表包含哪些字段，哪些数据类型，另一种是记录数据，保存每天记录的原始数据。它们是用不同的文件进行存储的。

> 在mysql指定的data_dir数据存储目录可以看到每张表对应一个**frm**文件，这个文件就是存放着表的结构数据。

## 1.索引操作
Operation	|In Place	|Rebuilds Table	|Permits Concurrent DML|	Only Modifies Metadata
---|---|---|---|---
Creating or adding a secondary index|	Yes	|No|	Yes|No
Dropping an index|	Yes|	No	|Yes	|Yes
Adding a FULLTEXT index	|Yes*	|No*	|No	|No
Changing the index type	|Yes|	No|	Yes|	Yes


### 添加索引

``` sql 
CREATE INDEX name ON table (col_list);

ALTER TABLE tbl_name ADD INDEX name (col_list);
```

当创建索引的时候数据依旧可以读写， 创建索引会等待在当前表相关的所有事务结束后才会完成， 所以索引会反映表里的最新的内容。

新创建的二级索引只会包含已经提交的数据，不会包含未提交的和旧版本的数据或者已经被标记需要删除的数据。

当创建索引时如果服务退出了，MySQL在恢复之后会丢弃未完成的索引，所以需要重新执行操作。

### 删除索引
```sql 
DROP INDEX name ON table;

ALTER TABLE tbl_name DROP INDEX name;

```

## 2. 主键操作

## 添加主键

Operation|	In Place|	Rebuilds Table|	Permits Concurrent DML|	Only Modifies Metadata
---|---|---|---|---
Adding a primary key|	Yes*|	Yes*|	Yes|	No
Dropping a primary key|	No|	Yes	|No	|No
Dropping a primary key and adding another	|Yes|	Yes|	Yes|	No

```sql 
ALTER TABLE tbl_name ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;
```
会在重建表，数据会被重新组织，这是一个成本很高的操作，
重建聚簇索引需要拷贝表里的数据所以最好在创建表的时候定义好主键

如果创建表的时候没加主键，mysql 会自动选择 一个 uniqu 且not null 的索引 作为主键，或者系统自动生成一个隐藏的主键

由于数据的存储是根据聚簇索引组织的，即使ALGORITHM=INPLACE 数据依旧会被拷贝

### 删除主键

``` sql 
ALTER TABLE tbl_name DROP PRIMARY KEY, ALGORITHM=COPY;
```

### 替换主键

```sql 
ALTER TABLE tbl_name DROP PRIMARY KEY, ADD PRIMARY KEY (column), ALGORITHM=INPLACE, LOCK=NONE;
```

## 3. 列操作 

Operation	|In Place|	Rebuilds Table|	Permits Concurrent DML|	Only Modifies Metadata
---|---|---|---|---
Adding a column|	Yes|	Yes|	Yes*	| No
Dropping a column	|Yes	|Yes	|Yes	| No
Renaming a column	|Yes|	No|	Yes*|	Yes
Reordering columns	|Yes	|Yes	|Yes	| No
Setting a column default value	|Yes|	No|	Yes| Yes
Changing the column data type|	No	|Yes	|No	| No
Dropping the column default value	|Yes	|No	| Yes|	Yes
Changing the auto-increment value|	Yes|	No	| Yes	|No*
Making a column NULL	|Yes	|Yes*	|Yes	| No
Making a column NOT NULL	|Yes*	|Yes*|	Yes |	No
Modifying the definition of an ENUM or SET column | Yes|	No	|Yes|	Yes

### 添加列

```sql
ALTER TABLE tbl_name ADD COLUMN column_name column_definition, ALGORITHM=INPLACE, LOCK=NONE;
```

### 删除列
```sql
ALTER TABLE tbl_name DROP COLUMN column_name, ALGORITHM=INPLACE, LOCK=NONE;
```

###  重命名列
```sql 
ALTER TABLE tbl CHANGE old_col_name new_col_name data_type, ALGORITHM=INPLACE, LOCK=NONE;
```

### 调整列顺序
```sql 
ALTER TABLE tbl_name MODIFY COLUMN col_name column_definition FIRST, ALGORITHM=INPLACE, LOCK=NONE;
```
**注意： 数据会被重新组织,成本较高**

### 修改类型
```sql
ALTER TABLE tbl_name CHANGE c1 c1 BIGINT, ALGORITHM=COPY;
```

仅支持  ALGORITHM=COPY.

### 设置默认值
```sql
ALTER TABLE tbl_name ALTER COLUMN col SET DEFAULT literal, ALGORITHM=INPLACE, LOCK=NONE;
```
### 删除默认值

```sql 
ALTER TABLE tbl ALTER COLUMN col DROP DEFAULT, ALGORITHM=INPLACE, LOCK=NONE;
C
```

###  修改auto-increment

```sql
ALTER TABLE table AUTO_INCREMENT=next_value, ALGORITHM=INPLACE, LOCK=NONE;
```
### 设置列可以为NULL
```sql
ALTER TABLE tbl_name MODIFY COLUMN column_name data_type NULL, ALGORITHM=INPLACE, LOCK=NONE;
```

### 设置列不可以为NULL
```sql
ALTER TABLE tbl_name MODIFY COLUMN column_name data_type NOT NULL, ALGORITHM=INPLACE, LOCK=NONE;
```
如果有值为NULL此操作会失败
**STRICT_ALL_TABLES or STRICT_TRANS_TABLES SQL_MODE is required for the operation to succeed**

### 修改 ENUM 或 SET 定义
```sql
CREATE TABLE t1 (c1 ENUM('a', 'b', 'c'));
ALTER TABLE t1 MODIFY COLUMN c1 ENUM('a', 'b', 'c', 'd'), ALGORITHM=INPLACE, LOCK=NONE;
```


## 4.外键操作 


Operation	|In Place|	Rebuilds Table|	Permits Concurrent DML|	Only Modifies Metadata
---|---|---|---|---
Adding a foreign key constraint|	Yes* |	No|	Yes	| Yes
Dropping a foreign key constraint	|Yes	|No	|Yes	| Yes

### 添加外键
```sql
ALTER TABLE tbl1 ADD CONSTRAINT fk_name FOREIGN KEY index (col1)
  REFERENCES tbl2(col2) referential_actions;
```

### 删除外键
```sql
ALTER TABLE tbl DROP FOREIGN KEY fk_name;
```

## 表操作 
Operation	|In Place|	Rebuilds Table|	Permits Concurrent DML|	Only Modifies Metadata
---|---|---|---|---
Changing the ROW_FORMAT | Yes | Yes | Yes | No
Changing the KEY_BLOCK_SIZE | Yes | Yes | Yes | No
Setting persistent table statistics | Yes | No | Yes | Yes
Specifying a character set | Yes | Yes* | No | No
Converting a character set | No | Yes | No | No
Optimizing a table | Yes* | Yes | Yes | No
Rebuilding with the FORCE option | Yes* | Yes | Yes | No
Performing a null rebuild | Yes* | Yes | Yes | No
Renaming a table | Yes | No | Yes | Yes

### 参考
https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html#online-ddl-index-operations

https://www.cnblogs.com/youyoui/p/9545621.html
