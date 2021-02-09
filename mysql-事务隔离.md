---
title: mysql 事务隔离
date: 2021-01-10 16:05:59
tags: 
---


隔离级别 |  脏读（Dirty Read | 不可重复读（NonRepeatable Read）|幻读（Phantom Read） 
---|--- | --- | --- | ---
读未提交（READ UNCOMMITTED | 可能   | 可能 | 可能
读已提交（READ COMMITTED） | 不可能 | 可能 | 可能
可重复读（REPEATABLE READ）| 不可能 | 不可能 | 可能
可串行化（Serializable）   | 不可能 | 不可能 | 不可能


## 读未提交（READ UNCOMMITTED）


![image](https://cdn.learnku.com/uploads/images/202002/05/32495/iL6jfZxiHJ.png!large)


> 
> 在读未提交隔离级别下，事务A可以读取到事务B修改过但未提交的数据。
> 
> 可能发生脏读、不可重复读和幻读问题，一般很少使用此隔离级别。

## 读已提交 (READ COMMITTED)

![image](https://cdn.learnku.com/uploads/images/202002/05/32495/BsMcuysaIB.png!large)
> 
> 在读已提交隔离级别下，事务B只能在事务A修改过并且已提交后才能读取到事务B修改的数据。
> 
> 读已提交隔离级别解决了脏读的问题，但可能发生不可重复读和幻读问题，一般很少使用此隔离级别。

## 可重复读 (REPEATABLE READ)
![image](https://cdn.learnku.com/uploads/images/202002/05/32495/yjRtVOpMBZ.png!large)

> 在可重复读隔离级别下，事务B只能在事务A修改过数据并提交后，自己也提交事务后，才能读取到事务B修改的数据。
> 
> 可重复读隔离级别解决了脏读和不可重复读的问题，但可能发生幻读问题。
> 
> 提问：为什么上了写锁（写操作），别的事务还可以读操作？
> 
> 因为InnoDB有MVCC机制（多版本并发控制），可以使用快照读，而不会被阻塞。

## 可串行化（SERIALIZABLE）

![image](https://cdn.learnku.com/uploads/images/202002/05/32495/S0Y1nk8yv6.png!large)

![image](https://cdn.learnku.com/uploads/images/202002/05/32495/LIfaeTxwPL.png!large)

![image](https://cdn.learnku.com/uploads/images/202002/05/32495/q4vVuHzqO0.png!large)

![image](https://cdn.learnku.com/uploads/images/202002/05/32495/l1BwLlDlYp.png!large)

> 各种问题（脏读、不可重复读、幻读）都不会发生，通过加锁实现（读锁和写锁）。

## 脏读（Dirty Read）
> 一个事务读到了另一个未提交事务修改过的数据

![image](https://cdn.learnku.com/uploads/images/202002/04/32495/Wcv8DTijTL.png!large)

## 不可重复读（Non-Repeatable Read）


> 一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值。（不可重复读在读未提交和读已提交隔离级别都可能会出现）

![image](https://cdn.learnku.com/uploads/images/202002/05/32495/YdNemia6wc.png!large)

## 幻读（Phantom）

> 一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来。（幻读在读未提交、读已提交、可重复读隔离级别都可能会出现）

![image](https://cdn.learnku.com/uploads/images/202002/04/32495/0sCtxw1Jno.png!large)

### 不可重复读和幻读到底有什么区别呢？

1. 不可重复读是读取了其他事务更改的数据，针对update操作

解决：使用行级锁，锁定该行，事务A多次读取操作完成后才释放该锁，这个时候才允许其他事务更改刚才的数据。

2.  幻读是读取了其他事务新增的数据，针对insert和delete操作

解决：使用表级锁，锁定整张表，事务A多次读取数据总量之后才释放该锁，这个时候才允许其他事务新增数据。


### MVCC 快照读和当前读

#### 快照读

``` sql
select * from table ....
```

#### 当前读 
> 对于会对数据修改的操作(update、insert、delete)都是采用当前读的模式。在执行这几个操作时会读取最新的记录，即使是别的事务提交的数据也可以查询到。假设要update一条记录，但是在另一个事务中已经delete掉这条数据并且commit了，如果update就会产生冲突，所以在update的时候需要知道最新的数据。读取的是最新的数据，需要加锁。以下第一个语句需要加共享锁，其它都需要加排它锁。
``` sql 
select * from table where ? lock in share mode; 
select * from table where ? for update; 
insert; 
update; 
delete;
```

---

###### 个人理解

> 加锁需要在最新的数据上加锁所以加锁一定会触发当前读
> 
> **如果触发了当前读RR隔离级别下就会出现不可重复读和幻读的**
> 

## 参考 

[彻底搞懂 MySQL 事务的隔离级别](https://developer.aliyun.com/article/743691)

[快速理解脏读、不可重复读、幻读和MVCC](https://cloud.tencent.com/developer/article/1450773)

