# MySQL的OrderBy

在开发的时候，会碰到需要根据指定的字段排序来现实结果的需求

```mysql
select city,name,age from t where city='杭州' order by name limit 1000
```

## 全字段排序

为了避免全表扫描，先在city上添加索引

在explain解释后可以看到Extra内容中存在**Using filesort**表示需要排序，而MySQL会给每个线程分配一块内存用于排序，这块内存叫做**sort_buffer**。

通常情况下，这个语句执行流程如下所示

1. 初始化sort_buffer，确定放入name、city、age三个字段
2. 从索引city找到第一个满足city=”杭州“条件的主键id
3. 到主键id索引中取出整行，取name、city、age三个字段的值，存入sort_buffer中
4. 从索引city中取下一个记录的主键id
5. 重复3，4直到city值不满足查询条件
6. 对sort_buffer中的数据按照字段name做快速排序
7. 按照排序结果取前1000行返回给客户端

过程6，对name进行排序这个动作，由于可能涉及排序的内容过大，导致内存中无法直接完成，可能需要使用外部排序，这取决于排序所需的内存和参数**sort_buffer_size**。

sort_buffer_size就是MySQL为排序开辟的内存的大小。如果排序的数据量小于这个size，排序就会在内存中完成，如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

可以通过如下代码查看是否使用了临时文件

```mysql

/* 打开optimizer_trace，只对本线程有效 */
SET optimizer_trace='enabled=on'; 

/* @a保存Innodb_rows_read的初始值 */
select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 执行语句 */
select city, name,age from t where city='杭州' order by name limit 1000; 

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* @b保存Innodb_rows_read的当前值 */
select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

/* 计算Innodb_rows_read差值 */
select @b-@a;
```

这个方法是通过查看OPTIMIZER_TRACE的结果来确认的，可以通过number_of_tmp_files中看到是否使用了临时文件

![89baf99cdeefe90a22370e1d6f5e6495](MySQL的orderBy.assets/89baf99cdeefe90a22370e1d6f5e6495.png)



内存放不下时，需要使用外部排序，外部排序一般使用归并排序算法，MySQL将需要排序的数据分成12份，每一份单独排序后存在这些临时文件中，然后把这12个有序文件在合并为一个有序的大文件。

- examined_rows表示当前参与排序的行数
- sort_mode
  - packed_additional_fields表示排序过程中对字符串进行了紧凑处理，即就算字段定很大也是按照实际长度来分配空间

> PS：为了避免对结论造成干扰，会把internal_tmp_disk_storage_engine设置为MyISAM，不然b-a结果会显示为4001，由于在查询OPTIMIZER_TRACE这个表的时候，会用到临时表，如果那个值是InnoDB的话，把数据从临时表取出会让innodb_rows_read加1

## rowid排序

如果返回的数据过多，那么内存下能同时放下的行数较少，要分成多个临时文件，排序性能会很差，**如果单行很大，这个方法效率不够好**。

那么如果MySQL认为排序的单行长度太大会怎么做呢。有一个参数可以控制MySQL在这种情况下采用另一种算法。（**max_length_for_sort_data**)。如果单行的长度超过这个值，MySQL就认为单行太大，要换个算法。

1. 初始化sort_buffer，确定放入两个字段，即name和id
2. 从索引city找到第一个满足city=‘杭州’的条件的主键id
3. 到主键id索引取出整行，取name、id两个字段，存入sort_buffer中
4. 从索引city取下一个记录的id
5. 重复3，4直到不满足city=‘杭州’为止
6. 对sort_buffer中数据排序
7. 遍历排序结果，取1000行，按照id的值返回原表取出对应数据返回给客户端

其实对于全字段排序，只是多了步骤7，多了多次回表操作。

此时对于上面查询的排序文件的结果的语句得到的结果会有所不同。

此时b-a居于的值变成5000了，因为在排序后还要根据id取原表取值，limit1000的语句会再次查询1000行

sort_mode变得后面的值为rowid了，因为参与排序的其实就name和id两个字段

number_of_tmp_files变成10了，这时候参与排序的行数虽然是4000行，但是记录变小了，所以对应的文件也变小了

## 两种排序的比较

相对来说MySQL其实是由于使用全字段排序的，毕竟这样省略了回表取数据的步骤。

这也体现出MySQL的一个设计思想：**如果内存够，就多利用内存，尽量减少磁盘访问**

对InnoDB表来说，rowid排序会要求回表多造成磁盘读，因此不会被优先选择。

由此其实可见，MySQL中排序是一个成本比较高的操作。但是也不是所有的orderby都需要排序操作的。MySQL之所以需要生成临时表，并且在临时表上作排序操作，其原因是原来的数据都是无需的。但是，我们也可以设想一下，如果从city这个索引上取出来的行，天生就是按照name递增排序的话，是不是就可以不再排序了呢。

## 使用索引的orderby

1. 从索引（city，name）找到第一个满足city=‘杭州’的主键id
2. 到主键id索引取出整行，取对应值，作为结果集的一部分直接返回
3. 从索引（city，name）取下一个主键id
4. 重复2，3，直到查询到第1000条记录，或者是不满足city=‘杭州’条件时循环结束

这时候使用explain的话会发现Extra字段中已经没有Using filesort了，即不需要排序了，对于（city，name）本来就有序，因此只需要找到前1000就可以直接退出了，只需要扫描1000次

当然：我们甚至可以使用**覆盖索引**避免2的操作，更加简化orderby的操作

## 零零散散的内容

- 255这个边界的问题，一般都用varchar(255)，原因是小于255需要一个字节记录长度，超过就需要两个字节
- 