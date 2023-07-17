# linux查看磁盘使用

```shell
# 查看磁盘占用率
df -h

# 查看哪个文件夹下占用资源最多
sudo du -sh *

# 删除后发现资源还没有释放，可以参考占用文件的某些进程没有关闭，仍然在占用，使用
lsof | grep deleted 

# 目录容量估算
du -sh /opt

# 以容量大小排序文件
du -sk * |sort -rn

# 查询xfs磁盘情况
xfs_info /opt


```

