# LINUX创建新用户

```shell
# 创建用户
adduser username
# 为用户初始化密码
passwd username

# 查询sudoers文件在哪里（赋予sudo权限）
whereis sudoers

# 查看文件权限
ls -l /etc/sudoers # 文件名

# 如果只有读的权限，直接添加w权限
chmod -v u+w /etc/sudoers

# 进入后在权限配置后加上自己的用户
#
# root          ALL=(ALL)  ALL
# newUsernaeme	ALL=(ALL)  ALL
#
vim /etc/sudoers

# 写权限收回
chmod -v u-w /etc/sudoers
```

