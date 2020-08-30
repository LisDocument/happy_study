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

> ps:时序上redo log先prepare，再binlog，然后redo log commit

如果把**innodb_flush_log_at_trx_commit**设置为1，那么redo log在prepare阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于prepare的redo log，再加上binlog来恢复的（如果redolog在prepare状态，crash了，binlog已经写入，默认commit，按照事务已经提交恢复数据）。

每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB就认为redo log在commit的时候就不需要fsync了，只会write到文件系统的pageCache中就够了。

> ps:通常说的MySQL的双1配置，指的是sync_binlog和innodb_flush_log_at_trx_commit都设置成1.也就是说，一个事务完整提交前，需要等待两次刷盘，一次是redolog（prepare阶段），一次是binlog。

虽然双1的情况下，每一次事务提交要等待两次刷盘，但是对于MySQL来说，其实对这个操作是有优化措施的，即TPS2w的情况下，实际上要写4W次其实并不会有这么多。这源于MySQL的组提交机制（group commit）。

> ps:日志逻辑序列号（log sequence number，LSN），LSN是单调递增的，对应redolog的一个个写入点，每次写入长度为length的redo log，LSN的值就会加上length，LSN也会写到InnoDB的数据页中，来确保数据页不会被多次执行重复的redo log。

![933fdc052c6339de2aa3bf3f65b188cc](MySQL的binlog和redolog的优化.assets/933fdc052c6339de2aa3bf3f65b188cc-1598756326188.png)

1. trx1是第一个到达的，会被选为这组的leader
2. 等trx1要开始写盘的时候，这个组里已经有了三个事务，此时LSN也变成了160
3. trx1去写盘的时候，带的就是LSN=160，因此等trx1返回时，所有小于160的redolog，都已经被持久化到磁盘
4. 此时，trx2和trx3就可以直接返回了

一次组提交里面，组员越多，节约磁盘IOPS效果越好。在并发更新场景下，第一个事务写完redologbuffer以后，接下来这个fsync越晚调用，组员可能越多，节约IOPS的效果就越好。

为了一次性fsync携带更多的组员，MySQL会主动的拖时间

![5ae7d074c34bc5bd55c82781de670c28](MySQL的binlog和redolog的优化.assets/5ae7d074c34bc5bd55c82781de670c28.png)

其实binlog是有两步操作的

1. 先把binlog从binlogcache中写到磁盘上的binlog文件
2. 调用fsync持久化

MySQL为了让组提交的效果更好，把redolog做fsync的时间拖到了步骤1之后

这样一来binlog也可以组提交了，如果有多个事务的binlog已经写完了，也是一起持久化，减少IOPS的消耗。但是通常redolog的fsync很快，因此binlog的write和fsync的效果一般没有redolog那么好，当然也可以通过设置一些参数来实现

1. **binlog_group_commit_sync_delay**,表示延迟多少微妙后才调用fsync
2. **binlog_group_commit_sync_no_delay_count**，表示累积多少次以后才调用fsync

两个条件是或的关系，因此当1设置为0的时候，2也不会有用了。

因此WAL机制主要得以于

1. redolog和binlog都是顺序写，磁盘的顺序写比随机写速度要快
2. 组提交机制，可以大幅度降低磁盘的IOPS消耗。

## 因此想要突破MySQL的IO性能瓶颈

1. 设置**binlog_group_commit_sync_delay**和**binlog_group_commit_sync_no_delay_count**的参数，减少binlog的写盘次数。这个方法基于**额外的故意等待**来实现的，可能会增加语句的响应时间，但没有丢失数据的风险
2. 将**sync_binlog**设置为大于1的值，这样可能会导致主机掉电的时候丢失binlog日志
3. 将**innodb_flush_log_at_trx_commit**设置为2，这样的话，主机掉电的时候会丢数据

当然是不建议**innodb_flush_log_at_trx_commit**设置为0的，因为这样表示redolog只存在内存中，这样MySQL异常重启也会丢失数据，而redolog写到文件系统的page cache的速度也是很快的，因此这个设置为2的时候和设置为0的性能差不多。风险更小

## crash-safe

1. 如果客户端收到事务成功的消息，那么事务就一定持久化了
2. 如果客户端收到事务失败的消息，事务一定失败了
3. 如果客户端收到执行异常的消息，应用需要重连后通过查询当前状态来继续后续的逻辑。此时数据库只需要保证内部一致（数据和日志之间，主库和备库之间）就可以了

## 零零散散的小事

sync_binlog和binlog_group_commit_sync_no_delay_count这两个参数区别：

-  sync_binlog = N，binlog_group_commit_sync_no_delay_count = M，binlog_group_commit_sync_delay = 很大值，这种情况下，达到N次以后刷盘，然后进入(sync_delay和no_delay_count)这个逻辑，只要sync_binlog=0，也会有前面的等待逻辑，但是等完后不调用fsync

