---
title: mysql MVCC 的实现
date: 2021-01-10 16:05:59
tags: 
---

# mysql mvcc


MVCC(Multi Version Concurrency Control的简称)，代表多版本并发控制。与MVCC相对的，是基于锁的并发控制，Lock-Based Concurrency Control)。
MVCC最大的优势：读不加锁，读写不冲突。在读多写少的OLTP应用中，读写不冲突是非常重要的，极大的增加了系统的并发性能


## mvcc 的实现 

InnoDB数据组织方式分为主键索引（聚簇索引）和二级索引

聚簇索引和二级索引都包含了**DELETED BIT**标记位来标识记录是否被删除，真正的删除是在事务commit之后且没有读会引用该版本数据的时候。

##### 1. InnoDB 为主键索引上每一行数据增加了三个隐藏列用于实现MVCC。

- DB_TRX_ID, 6byte, 创建这条记录/最后一次更新这条记录的事务ID


- DB_ROLL_PTR, 7byte，回滚指针，指向这条记录的上一个版本（存储于rollback segment里）


- DB_ROW_ID, 6byte，隐含的自增ID，如果数据表没有主键，InnoDB会自动以DB_ROW_ID产生一个聚簇索引


对于主键索引，更新是在原记录位置更新，记录的历史版本是放在专门的rollback segment里（undo log）
    　
    
    UPDATE非主键语句的效果是
    
    　　　　老记录被复制到rollback segment中形成undo log，DB_TRX_ID和DB_ROLL_PTR不动
    
    　　　　新记录的DB_TRX_ID = 当前事务ID，DB_ROLL_PTR指向老记录形成的undo log
    
    　　　　这样就能通过DB_ROLL_PTR找到这条记录的历史版本。如果对同一行记录执行连续的update操作，新记录与undo log会组成一个链表，遍历这个链表可以看到这条记录的变迁）

#####  2. MySQL的一致性读，是通过一个叫做read view的结构来实现的
read_view中维护了系统中活跃事务集合的快照，**这些活跃事务ID的最小值为up_limit_id，最大值为low_limit_id**

###### [ReadView](https://github.com/twitter-forks/mysql/blob/master/storage/innobase/include/read0read.h#L124)主要结构

    low_limit_id。 活跃事务ID的最大值，当事务ID大于等于该值的数据修改不可见
    up_limit_id. 活跃事务ID的最小值，事务ID小于该值的数据修改可见。
    creator_trx_id。创建该ReadView的事务ID，该事务ID的数据修改可见。
    trx_ids。当快照创建时的活跃读写事务列表。
    low_limit_no。 /*!< The view does not need to see the undo
				logs for transactions whose transaction number
				is strictly smaller (<) than this value: they
				can be removed in purge if not needed by other
				views */   用于Purge不需要的Undo。
SELECT操作返回结果的可见性是由以下规则决定的：

    DB_TRX_ID < up_limit_id  -> 此记录的最后一次修改在read_view创建之前，可见
    
    DB_TRX_ID >= low_limit_id   -> 此记录的最后一次修改在read_view创建之后，不可见  ->  需要用DB_ROLL_PTR查找undo log(此记录的上一次修改)，然后根据undo log的DB_TRX_ID再计算一次可见性
    
    up_limit_id <= DB_TRX_ID < low_limit_id -> 需要进一步检查read_view中是否含有DB_TRX_ID
    
    　　　　DB_TRX_ID ∉ m_ids -> 此记录的最后一次修改在read_view创建之前，可见
    
    　　　　DB_TRX_ID ∈ m_ids -> 此记录的最后一次修改在read_view创建时尚未保存，不可见  ->  需要用DB_ROLL_PTR查找undo log(此记录的上一次修改)，然后根据undo log的DB_TRX_ID再从头计算一次可见性
    
    经过上述规则的决议，我们得到了这条记录相对read_view来说，可见的结果。
    
    此时，如果这条记录的delete_flag为true，说明这条记录已被删除，不返回。
    
    　　　如果delete_flag为false，说明此记录可以安全返回给客户端

##### 4 用MVCC这一种手段可以同时实现RR与RC隔离级别
它们的不同之处在于：

RR：read view是在first touch read时创建的，也就是执行事务中的第一条SELECT语句的瞬间，后续所有的SELECT都是复用这个read view，所以能保证每次读取的一致性（可重复读的语义）

RC：每次读取，都会创建一个新的read view。这样就能读取到其他事务已经COMMIT的内容。

所以对于InnoDB来说，RR虽然比RC隔离级别高，但是开销反而相对少。

补充：RU的实现就简单多了，不使用read view，也不需要管什么DB_TRX_ID和DB_ROLL_PTR，直接读取最新的record即可。

##### 5. 二级索引与MVCC
    MySQL的索引分为聚簇索引(clustered index)与二级索引(secondary index)两种。
    
    刚才讲的内容是基于聚簇索引的，只有聚簇索引中含有DB_TRX_ID与DB_ROLL_PTR隐藏列，可以比较容易的实现MVCC
    
    但是二级索引中并不含有这几个隐藏列，只含有1个bit的deleted flag，咋办？
    
     　　好办，如果UPDATE语句涉及到二级索引的键值，将老的二级索引的deleted flag标记为true，然后创建一条新的二级索引记录即可。
    
    但是如果想根据二级索引来做查询，这可就麻烦了。因为二级索引不维护版本信息，无法判断二级索引中记录的可见性。
    
    所以还是需要回到聚簇索引中来：
    
        根据二级索引维护的主键值去聚簇索引中查找记录（使用MVCC规则）
        
        如果查出来的结果跟二级索引里维护的结果相同 -> 返回，如果不同 -> 丢弃
    
    如果对于一条查询语句，二级索引中有很多条满足条件的结果（连续多次更新，导致二级索引中有很多条记录），那上面这个流程就比较低效了。所以InnoDB的作者搞了个机智的小优化：
    
    在二级索引中，用一个额外的名为MAX_TRX_ID的变量来记录最后一次更新二级索引的事务的ID
    
    那么，如果当前语句关联的read_view的 up_limit_id > MAX_TRX_ID，说明在创建read_view时最后一次更新二级索引的事务已经结束，也就是说二级索引里的所有记录对于当前查询都是可见的，此时可以直接根据二级索引的deleted flag来确定记录是否应该被返回。
    
    小结一下：二级索引的MVCC可见性判断在MAX_TRX_ID失效的情况下需要依赖聚簇索引才能完成。

##### 6. purge
https://developer.aliyun.com/article/646471

    从前面的分析可以看出，为了实现InnoDB的MVCC机制，更新或者删除操作都只是设置一下老记录的deleted_bit，并不真正将过时的记录删除。
    
    为了节省磁盘空间，InnoDB有专门的purge线程来清理deleted_bit为true的记录。
    
    为了不影响MVCC的正常工作，purge线程自己也维护了一个read view（这个read view相当于系统中最老活跃事务的read view）
    
    如果某个记录的deleted_bit为true，并且DB_TRX_ID相对于purge线程的read view可见，那么这条记录一定是可以被安全清除的。
    
    由于Undo log会保留直到事务提交同时没有其他快照读引用后才会purge。所以需要尽量避免长语句或长事务的执行，避免因此导致的undo堆积或者undo链太长使读取变慢。

     

## 参考文献

[MySQL · 引擎特性 · InnoDB MVCC 相关实现](http://mysql.taobao.org/monthly/2018/11/04/) 

[MySQL InnoDB MVCC深度分析
](https://www.cnblogs.com/stevenczp/p/8018986.html)

[MySQL Innodb Purge简介](https://developer.aliyun.com/article/646471)