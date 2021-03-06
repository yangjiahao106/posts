---
title: redis笔记
date: 2021-02-17 23:28:10
tags:
---

# redis 

### 关于redis 单线程
redis 的单线程是指网络IO和键值对的读写是由一个线程完成的，但redis的其他功能 如持久化，异步删除，集群数据同步等，其实是由额外的线程完成的。

### 为什么使用单线程？

系统中通常会存在被多个线程同时访问的共享资源，当多个线程要修改这个资源时为了保证资源的正确性，需要额外的机制来进行保证（比如互斥锁），这个额外的机制就会带来额外的开销

### 为什么单线程这么快？

Linux 中的 IO 多路复用机制是指一个线程处理多个 IO 流，就是我们经常听到的 select/epoll 机制。简单来说，在 Redis 只运行单线程的情况下，该机制允许内核中，同时存在多个监听套接字和已连接套接字。内核会一直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给 Redis 线程处理，这就实现了一个 Redis 线程处理多个 IO 流的效果。

![基于多路复用的Redis高性能IO模型](https://static001.geekbang.org/resource/image/00/ea/00ff790d4f6225aaeeebba34a71d8bea.jpg)

为了在请求到达时能通知到 Redis 线程，select/epoll 提供了基于事件的回调机制，即针对不同事件的发生，调用相应的处理函数。那么，回调机制是怎么工作的呢？其实，select/epoll 一旦监测到 FD 上有请求到达时，就会触发相应的事件。这些事件会被放进一个事件队列，Redis 单线程对该事件队列不断进行处理。这样一来，Redis 无需一直轮询是否有请求实际发生，这就可以避免造成 CPU 资源浪费。同时，Redis 在对事件队列中的事件进行处理时，会调用相应的处理函数，这就实现了基于事件的回调。因为 Redis 一直在对事件队列进行处理，所以能及时响应客户端请求，提升 Redis 的响应性能。

### 性能瓶颈

由于主要是主进程在执行操作所以主进程任何耗时的操作都可能是性能瓶颈
，比如bigkey, 全量返回等。

AOF重写创建bgrewriteaof子进程 和RDB创建bgsave子进程时要拷贝父进程的页表。如果 Redis 实例内存大，页表就会大，fork 执行时间就会长，这就会给主线程带来阻塞风险。(数据量过大可以使用切片集群)

## 持久化
目前，Redis 的持久化主要有两大机制，即 AOF（Append Only File）日志和 RDB 快照。
### AOF 

AOF 日志是写后日志，即redis先执行命名将数据写入内存，然后才记录日志。这是

后些日志的两个好处：

1. 避免额外的检查开销，Redis 在向 AOF 里面记录日志的时候，并不会先去对这些命令进行语法检查。所以，如果先记日志再执行命令的话，日志中就有可能记录了错误的命令，Redis 在使用日志恢复数据时，就可能会出错。
2. 不会阻塞redis的写入 

AOF日志的两个风险

1. 未来得及写入AOF日志时如果发生宕机则可能丢失数据。
2. AOF日志写入是在主进程中进行的，如果写盘的压力大则可能阻塞后续的操作。


#### 三种写回策略


![image](https://static001.geekbang.org/resource/image/72/f8/72f547f18dbac788c7d11yy167d7ebf8.jpg)

#### AOF重写

![image](https://static001.geekbang.org/resource/image/6b/e8/6b054eb1aed0734bd81ddab9a31d0be8.jpg)

每次 AOF 重写时，Redis 会先执行一个**内存拷贝** (Copy On Write机制)，用于重写；然后，使用两个日志保证在重写过程中，新写入的数据不会丢失。而且，因为 Redis 采用额外的线程进行数据重写，所以，这个过程并不会阻塞主线程。

#### RDB 内存快照

和 AOF 相比，RDB 记录的是某一时刻的数据，并不是操作，所以，在做数据恢复时，我们可以直接把 RDB 文件读入内存，很快地完成恢复

Redis 提供了两个命令来生成 RDB 文件，分别是 save 和 bgsave。

1. save：在主线程中执行，会导致阻塞；
2. bgsave：创建一个子进程，专门用于写入 RDB 文件，避免了主线程的阻塞，这也是 Redis RDB 文件生成的默认配置。

#### Copy-On-Write (写时复制技术)

Redis 就会借助操作系统提供的写时复制技术（Copy-On-Write, COW），在执行快照的同时，正常处理写操作。

简单来说，bgsave 子进程是由主线程 fork 生成的，可以共享主线程的所有内存数据。bgsave 子进程运行后，开始读取主线程的内存数据，并把它们写入 RDB 文件。

此时，如果主线程对这些数据也都是读操作（例如图中的键值对 A），那么，主线程和 bgsave 子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对 C），那么，这块数据就会被复制一份，生成该数据的副本。然后，bgsave 子进程会把这个副本数据写入 RDB 文件，
![image](https://static001.geekbang.org/resource/image/4d/cc/4dc5fb99a1c94f70957cce1ffef419cc.jpg)

#### 关于 AOF 和 RDB 的选择问题的三点建议：
- 数据不能丢失时，内存快照和 AOF 的混合使用是一个很好的选择；
- 如果允许分钟级别的数据丢失，可以只使用 RDB；
- 如果只用 AOF，优先使用 everysec 的配置选项，因为它在可靠性和性能之间取了一个平衡。