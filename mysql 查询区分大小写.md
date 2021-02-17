---
title: mysql 查询区分大小写.md
date: 2019-05-14 16:58:40
tags:
---


# Mysql查询英文如何严格区分大小写？

MySQL 在查询英文时默认是不区分大小写的，比如下面这两个SQL的效果是一模一样的。

```sql
SELECT * FROM class WHERE name='TOM';
SELECT * FROM class WHERE name='tom';
```
如果想要严格区分大小写改怎么做呢？

## 方法一： 修改collate
collate是指字符检索策略，可以参考这篇博客： https://juejin.im/post/5bfe5cc36fb9a04a082161c2 

Mysql默认的字符检索策略：utf8mb4_general_ci，ci是case insensitive的缩写表示不区分大小写，utf8mb4_bin表示二进制比较可以区分大小写。

### 创建数据库时指定collate 

```sql
CREATE DATABASE <db_name> DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

```
更新库的collate

```sql
ALTER DATABASE <database_name> CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

```

### 在创建表的时候指定collate 为utf8_bin
```sql
CREATE TABLE class(
id INT PRIMARY KEY,
name VARCHAR(64) NOT NULL DEFAULT '',
) ENGINE = INNODB COLLATE =utf8mb4_bin;

```
更新表的collate
```sql
ALTER TABLE <table_name> CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

### 也可以创建表的时候设置列级别的collate，指定单个列的collate

```sql
CREATE TABLE class(
id INT PRIMARY KEY,
name VVARCHAR（64） CHARACTER SET utf8mb4 COLLATE utf8mb4_bin NOT NULL DEFAULT '',
) ENGINE = INNODB; 

```
更新列的collate

```sql
ALTER TABLE <table_name> MODIFY <column_name> VARCHAR(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

```

## 方法二： 修改sql

在写SQL的时候加上关键字BINARY进行二进制比较比如：
```sql
SELECT * FROM class WHERE BINARY name='TOM';
SELECT * FROM class WHERE BINARY name='tom';
```