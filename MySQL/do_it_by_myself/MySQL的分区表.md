# MySQL的分区表

```mysql
mysql>CREATE TABLE `t` (
  `ftime` datetime NOT NULL,
  `c` int(11) DEFAULT NULL,
  KEY (`ftime`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY RANGE (YEAR(ftime))
(PARTITION p_2017 VALUES LESS THAN (2017) ENGINE = InnoDB,
 PARTITION p_2018 VALUES LESS THAN (2018) ENGINE = InnoDB,
 PARTITION p_2019 VALUES LESS THAN (2019) ENGINE = InnoDB,
PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE = InnoDB);
insert into t values('2017-4-1',1),('2018-4-1',1);
```

这个表包含了一个.frm文件和4个.ibd文件，每个分区对应一个.ibd文件，也就是说对于server层这是一个表，对于引擎层这是4个表。

## 分区表的引擎层行为

对于单表来说，插入ftime如果是‘2017-4-1’和‘2018-4-1’这两个记录之间的的间隙会被锁住，那么插入这两个时间之间的数据应该会被锁住，但是对于分区表来说，其实记录插入的是两个表，因此锁住的内容其实会有区别。

那么手动分表和分区表具体有什么区别，

- 手工分表的逻辑，是找到需要更新的所有分表，再依次更新。在性能上，这和分区表没有实质上的差别
- 分区表和手工分表，一个是由server层来决定使用哪个分区，一个是由应用层代码来决定

### 分区策略

每当第一次访问一个分区表的时候，MySQL需要把所有的分区都访问一遍，如果 一个分区表的分区很多，超过了1000个，但是**open_file_limit**设置的默认值是1024，那么就会在访问这个表的时候，由于需要打开所有的文件，导致打开的表文件的个数超过了上线而报错（在MyISAM）。

MyISAM使用的分区策略，称为通用分区策略，每次访问分区都由server层控制。有严重的性能问题

从MySQL5.7.9开始，InnoDB引擎引入了本地分区策略，这个策略是在InnoDB内部自己管理打开分区的行为。

从5.7.17开始，通用分区策略就被标记了启用

从8开始，就不允许MyISAM分区了。

## 分区表的server层行为

其实从server层来看的话，一个分区表就只是一个表。

因此其实对表做查询操作的时候，持有整个表的MDL锁，会导致所有分区的alter语句被锁住，因此在DDL的时候，分区表表现出来的影响会更大。因为在普通分表的话，truncate一个分表的时候，肯定不会跟另外一个分表的查询语句出现MDL冲突



