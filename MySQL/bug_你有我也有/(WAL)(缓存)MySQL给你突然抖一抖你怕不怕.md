# MySQL的语句会突然变慢杂费事？

场景：在语句执行的时候，会遇到突然就卡一下，突然变得很慢，并且很难复现，时间出现的也短

## 先说说导致这样的原因

> ps:当内存数据页和磁盘数据页不一致的时候，我们称这个内存页为**脏页**，内存数据写入磁盘后，内存和磁盘上的数据页的内容就一致了，称为**干净页**

对于前面的场景，不难想到其实原来很快的操作就是在写内存和日志，而MySQL偶尔抖一下的时候，那可能就是在刷脏

## 那么说说看为啥会触发刷脏呢

- redo log写满了，系统会停止所有更新操作，把checkpoint往前推进，方便redolog留出空间可以继续写，在《MySQL的更新》中已经说明了redolog类环形写日志方式，尾到头方式一直循环。
- 系统内存不足，需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出来给别的内存页使用，如果淘汰的是**脏页**，就要先执行flush操作。<u>那么这样是否可以考虑脏页再脏一点的概念，新的数据直接修改脏页，下次要请求的时候，从磁盘读入数据页，然后拿redo log出来应用。其实这是从性能考虑的，如果刷脏一定会写盘，就保证了每个数据页有两种状态。这样的效率最高，逻辑上看也是最清晰的</u>
  - 一种是内存里存在，内存里肯定就是正确的结果，直接返回
  - 一种是内存里没有数据，就可以肯定数据文件上是正确的结果，读入内存后返回

- MySQL认为系统空闲的时候，会进行刷脏，但是在更新操作频繁的时候，也会见缝插针的找时间刷脏
- MySQL正常关闭的时候，MySQL会把内存的脏页flush到磁盘上，下次启动的时候就可以直接从磁盘读数据，启动速度会很快

## 那么来分析一波对性能的影响

性能，第三四场景其实不会进行考虑，主要分析第一二场景下

### 第一种

redolog写满，flush刷脏，这种情况InnoDB必须尽量避免，因为出现这种情况的时候，系统就不会再接受更新了，所有的更新都必须堵住，监控上会表示为更新数为0

### 第二种

内存不够用了，先将脏页写到磁盘，这种情况是常态，InnoDB用缓冲池（buffer pool）管理内存，缓冲池的内存页有三种状态：

- 还没使用
- 使用了是干净页
- 使用了是脏页

InnoDB的策略是尽量使用内存，因此对于一个长时间运行的库来说，未使用的页很少。

当要读入的数据页没有在内存的时候，就必须要到缓冲池中申请一个数据页，这时候只能把最久不使用的数据页从内存中淘汰掉，如果是脏页刷脏，如果是干净页直接释放。

虽然这种情形是常态，但是出现这两种情况，会明显影响性能

- 一个查询要淘汰的脏页数量太多，会导致查询的响应时间明显变长
- 日志写满，更新全部堵住，写性能跌为0

因此InnoDB需要控制脏页比例，避免以上的情况

#### 刷脏的控制策略

首先需要告诉InnoDB所在主机的IO能力，这样InnoDB才能知道需要全力刷脏的时候可以刷多快

**innodb_io_capacity**,可以用这个告诉InnoDB磁盘能力，这个值可以设置为磁盘的IOPS。磁盘的IOPS可以用fio工具测试

```shell
# 一般参照写能力
fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 
```

**innodb_max_dirty_pages_pct**是脏页比例上限。默认是75。InnoDB会根据当前的脏页比例（M），算出一个0-100之间的数字

> ps:InnoDB每次写入的日志有一个序号，当前写入的序号和checkpoint对应的序号之间的差值是N的话，InnoDB会根据这个N算出一个范围在0到100之间的数字，大概是N越大，算出的值越大，即**写入与redolog相距越大，脏页比例越大**

根据上面得到的两个结果，取其中比较大的值记为R，之后引擎就可以按照innodb_io_capacity定义的能力乘以R%来控制刷脏的速度

![cc44c1d080141aa50df6a91067475374]((WAL)(缓存)MySQL给你突然抖一抖你怕不怕.assets/cc44c1d080141aa50df6a91067475374.png)

为了避免语句抖动，平时要多多关注脏页比例，不要让其接近75%

脏页比例是通过Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total得到的，具体代码

```mysql
mysql> select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
```

#### 刷脏的时候一个有趣的策略（8.0后默认不开启）

连带机制，在刷脏flush脏页的时候，如果该页旁边的数据页刚刚好也是脏页，会被连带着刷掉，而且连带机制会继续蔓延直接到不是脏页位置，即一块脏了，只要里面有一个要被刷脏，其他全部都一起刷

**innodb_flush_neighbors**参数就是控制这个行为的，1的时候连坐，0就刷自己

连坐在机械硬盘中可以减少很多随机IO，在IOPS比较小的机械硬盘中，相同的逻辑操作减少随机IO就意味着系统性能的大幅度提升。

如果使用SSD的话，可以设置为0，因此此时IOPS往往不是瓶颈了，而只刷自己就能更快的执行完刷脏操作，减少SQL语句响应时间

## 零零散散的内容

- redo log优势在于将磁盘随机写转换成了顺序写，如果将redolog的不同部分刷掉，不就是在redolog里随机读写了么
  - 淘汰的时候，刷脏页过程是不动redolog文件的，有个额外的保证，redolog在**重放**的时候，如果数据页刷过，会识别会跳过

- 重建表的时候，InnoDB不会将整张表沾满，每个页溜了1/16给后续的更新用。即重建之后不是最**紧凑**的