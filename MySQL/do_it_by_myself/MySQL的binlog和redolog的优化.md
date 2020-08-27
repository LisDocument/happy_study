# MySQL对binlog和redolog的优化（对照MySQL的更新加强篇）

[]: MySQL的更新.md



## binlog的写入机制

binlog的写入逻辑比较简单；事务执行过程中，先把日志写到binlog cache，事务提交的时候再把binlogcache写到binlog中

虽然有binlog cache参与binlog的缓存，但是**一个事务的binlog是不能被拆开的**，因此不论事务多大，也要确保一次性写入，这就涉及到binlog cache的保存问题

系统给binlog cache分配了一片内存，每个线程都有独立的binlog cache内存空间，参数**binlog_cache_size**用于控制单个线程内binlog cache所占内存的大小，如果超过这个大小，就要暂存到磁盘。

事务提交的时候，执行器把binlog cache里的完整事务写入到binlog中，并清空binlog cache。

![9ed86644d5f39efb0efec595abb92e3e](MySQL的binlog和redolog的优化.assets/9ed86644d5f39efb0efec595abb92e3e.png)

即**每个线程都有自己的binlog cache，但是公用一份binlog文件**

- write指的是把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，因此速度很快
- fsync真正讲数据持久化的磁盘，一般情况下，普遍认为fsync才占磁盘IOPS

write和fsync的时机，是由参数**sync_binlog**控制的：

- 0的时候，每次提交事务都只write，不fsync
- 1的时候，每次提交事务都会执行fsync
- n（n>1）的时候，每次提交都write，累积N个事务后fsync

因此在IO瓶颈的场景里，将sync_binlog设置为一个比较大的值，可以提升性能，考虑到丢失日志量的可控性，一般不建议将这个参数设成0，比较常见的是设置100-1000的某个值，但是这么做主机发生异常重启，会丢失最近N个事务的binlog日志

## redolog的写入机制

redologbuffer：事务在执行过程中，生成的redolog是要先写到redologbuffer的

redologbuffer在每次生成后也不需要直接持久化到磁盘。

在事务执行期间MySQL发生异常重启，那这部分日志就丢了。由于事务并没有提交，所以这时日志丢了也不会有损失。

![9d057f61d3962407f413deebc80526d4](MySQL的binlog和redolog的优化.assets/9d057f61d3962407f413deebc80526d4.png)

redolog 存在可能有三种状态

- 存在redologbuffer中，物理上是在MySQL进程内存中的，即红色部分。
- 写到磁盘（write），但是没有持久化（fsync），物理上是在文件系统的page cache里面，也就是黄色部分
- 持久化到磁盘，对应的是hard disk，即绿色部分

日志写道redo log buffer是很快的，write到page cache也差不多，但是持久化到磁盘的速度会比较慢。

为了控制redo log的写入策略，InnoDB提供了**innodb_flush_log_at_trx_commit**参数

- 0的时候，表示每次事务提交时都只是把redo log 留在redo log buffer中
- 1的时候，表示每次事务提交时都将redolog直接持久化到磁盘
- 2的时候，表示每次事务提交时都把redolog写到page cache（后台线程）

> InnoDB有个后台线程，每隔1s，就会把redo log buffer中的日志，调用到write写道文件系统的page cache，然后调用fsync持久化。

> 事务执行中间redolog也是直接写在redologbuffer中的，这些redolog也会被后台线程一起持久化到磁盘，即一个没有提交的事务的redolog，也时有可能被持久化到磁盘的。

除了后台线程，还有场景会触发事务redolog写入到磁盘

1. redologbuffer占用的空间打到**innodb_log_buffer_size**一半的时候，后台线程会主动写盘（写盘只是write，并没有fsync）
2. 并行的事务提交的时候，顺便把这个事务redologbuffer持久化到磁盘。<u>假如一个事务A执行到一般，已经写了一些redolog到buffer中，这时候另外一个线程的事务B提交，如果innodb_flush_log_at_trx_commit设置的是1，那么按照这个参数的逻辑，B要把所有redologbuffer的日志全部持久化到磁盘，这时候会顺带Aredologbuffer的日志一起持久化</u>

