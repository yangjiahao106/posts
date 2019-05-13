---
title: mysql中for update
date: 2019-04-15 22:32:24
tags:
---

# mysql中for update

## 创建一些数据

```sql
CREATE TABLE account (`id` INT(10),`name` VARCHAR(20),`balance`,BIGINT(20)) ENGINE=InnoDB;
INSERT INTO account(`id`, `name`, `balance`)
VALUES
(1,'user1',100),
(2,'user2',200),
(3,'user3',500);
```

假设我没需要查询用户的余额，进行结算后更新余额。

### 连接一

```sql
BEGIN;
SELECT * FROM account WHERE `id` =  1 FOR UPDATE;
UPDATE account set `balance` = 200 where `id` = 1;

```

### 连接二

```sql
BEGIN;
SELECT * FROM account WHERE `id` =  1 FOR UPDATE;
UPDATE account set `balance` = 50 where `id` = 1;
COMMIT;
```

### 连接三

```sql
BEGIN;
SELECT * FROM account WHERE `id` =  2 FOR UPDATE;
UPDATE account set `balance` = 100 where `id` = 2;
COMMIT;
```

我们建立了三个事务，连接一执行后事务不提交，执行连接二会阻塞在SELECT，直到连接一的事务提交，连接三可以顺利执行。连接一SELECT使用了FOR UPDATE会在查询到的结果上加上排它锁(exclusive locks),由于连接二SELECT 也使用了 FOR UPDATE在尝试加锁的时候就会阻塞。InnodDB引擎下 UPDATE DELETE INSERT 操作都会默认添加排它锁。

这里的排它锁是行级锁，不会影响表中其他数据的操作，所以连接三可以顺利执行，不会阻塞。

## 使用数据库进行计算

对于一些简单的计算和更新可以放到数据库中，InnoDB UPDATE 操作默认使用排他锁，所以是并发安全的
例如：

### 连接 一

```sql
BEGIN;
SELECT * FROM account WHERE `id` =  1;
UPDATE account set `balance` = `balance` + 100 where `id` = 1;
```

### 连接 二

```sql
BEGIN;
SELECT * FROM account WHERE `id` =  1;
UPDATE account set `balance` = `balance` - 50 where `id` = 1;
```

## 行级锁

### 共享锁和排他锁

#### 1.共享锁(S锁)： SELECT ... FOR SHARE

1.使用S后，其他的事务可以读取数据，但是在你的事务提交前其他事务是不可以修改数据的。2.如果你查询这些数据中有在其他的事务中被修改，并且事务还没有提交，你的查询也会被阻塞，直到其他的事务提交。

#### 2.排他锁(X锁): SELECT ... FOR UPDATE；

对数据行添加X锁后其他的线程再添加X锁就会阻塞。

#### 简单概括

添加S锁后 其他事务可以添加S锁但不能添加X锁
添加X锁后 其他事务即不能添加S锁也不能添加S锁

## 隔离级别
