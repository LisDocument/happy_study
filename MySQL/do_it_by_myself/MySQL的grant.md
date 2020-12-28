# MySQL的grant

grant是用来给用户赋权的。

在一些文档中grant后需要执行一个flush privileges，才能使赋权语句生效。

```mysql
mysql>create user 'ua'@'%' identified by 'pa'
```

在MySQL里面，用户名+地址才表示一个用户。

这个命令做了两个动作：

1. 磁盘上，往mysql.user表里插入一行，由于没有指定权限，所以这行数据上所有表示权限的字段都是N；
2. 内存里，往数据acl_users里插入一个acl_user对象，这个对象的access字段值为0；

在MySQL中，用户权限是有不同的范围的，

## 全局权限

全局权限，作用于整个MySQL实例，这些权限信息保存在mysql库的user表里。如果我要给用户ua赋一个最高权限的话

```mysql
mysql>grant all privileges on *.* to 'ua'@'%' with grant option;
```

1. 磁盘上，将mysql.user表里，用户'ua'@'%'这一行的所有表示权限的字段的值都修改为‘Y’
2. 内存里，从数组acl_users中找到这个用户对应的对象，将access值（权限位）修改为二进制的全1

在这个grant命令执行完成后，如果有新的客户端使用用户名ua登录成功，MySQL会为新链接维护一个线程对象，然后从acl_users数组里查到这个用户的权限，并将权限值拷贝到这个线程对象中，之后这个链接中执行的语句，所有关于全局权限的判断，都直接使用线程对象内部保存的权限位。

1. grant命令对于全局权限，同时更新了磁盘和内存。命令完成后即时生效，接下来新创建的链接会使用新的权限
2. 对于一个已经存在的链接，他的全局权限不受grant命令的影响

如果要收回上面的grant语句赋予的权限，可以使用如下命令

```mysql
mysql> revoke all privileges on *.* from 'ua'@'%';
```

这个将grant的动作全部重置

## db权限

MySQL支持库级别的权限定义

```mysql
mysql>grant all privileges on db1.* to 'ua'@'%' with grant option;
```

1. 磁盘上，往mysql.db表中插入一行记录，所有权限位字段设置为‘Y’；
2. 内存里，增加一个对象到数据acl_dbs中，这个对象的权限位为“全1”。

每次要判断一个用户对一个数据库读写权限的时候，都需要遍历一次acl_dbs数组，根据user,host,db找到匹配对象，然后根据对象的权限位来判断。即grant修改db权限的时候，是同时对磁盘和内存生效的。

## 表权限和列权限

除了db权限外，还支持更细粒度的表权限和列权限，其中表权限定义放在表mysql.table_priv，列权限定义存放在表mysql.columns_priv中，这两类权限，组合起来存放在内存的hash结构column_priv_hash中。

```mysql
mysql>grant all privileges on db1.t1 to 'ua'@'%' with grant option;
mysql>GRANT select(id), insert(id,a) on mydb.mytbl TO 'ua'@'%' with grant option;
```

和db权限类似，这两个权限每次grant的时候都会修改数据表，同步修改内存中的hash结构。即这两个的操作都会实时影响链接。

## flush privileges命令

flush privileges会清空acl_users数组，从mysql.user表中读取数据并重新加载，重新构造一个acl_users数组。

因此其实在正常情况下，grant命令之后，没必要跟着执行flush privileges命令

但是，当数据表中的权限数据跟内存中的权限数据不一致的时候，flush privileges语句可以用来重建内存数据，达到一致状态。



