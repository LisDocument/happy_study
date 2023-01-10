# Linux 命令合集

```shell
# 按照修改时间正序获取结果
ls -lrt
ll -rt
# 查询满足通配符的文件， -exec后面的内容为删除的附加语句，保持一致即可，如果是要查询的话，使用-exec前面即可，{}会被替换成文件名
find ./ -name "*.o" -exec rm {} \;
# 添加软连接（符号连接）， xx是软连接名称，xxx是实体文件
ln -s xx xxx 
# 将标准输出和标准错误重定向到同一文件；nohup？
ls  proc/*.c > list 2> &l
ls  proc/*.c &> list
# awk '{print $9}'打印第九列的数据（当前例子为文件名），xargs分割参数分别执行file命令，得到结果grep一下，得到ELF相关的文件（文件名，单列），awk打印第一列的数据， tr -d ':' 删除":"这个符号返回结果
ls -lrt | awk '{print $9}'|xargs file|grep  ELF| awk '{print $1}'|tr -d ':'
```

## 1. find

```shell
# 正则查找， iregex为正则忽略大小写
find . -regex  ".*\(\.txt|\.pdf\)$"
# 否定参数
find . ! -name "*.txt" -print
# 最大深度为1的文件， type f为文件， d为文件夹
find . -maxdepth 1 -type f
# 查询7天内访问过的所有文件， -7的负号为默认位，可不填，但是+7必须填，-7为7天内， +7为大于7天的文件
# atime为访问过的文件； mtime为修改过的时间；ctime为变化时间（权限或者元数据变化）
find . -atime -7 -type f -print
# 按照文件大小搜索， +2k表示大于2k的文件
find . -type f -size +2k
# 按照文件权限查找， 具有可执行权限的文件
find . -type f -perm 644 -print 
# 按照用户查找， 查询weber这个用户的文件
find . -type f -user weber -print
# 查找并删除所有结果
find . -type f -name "*.txt" -delete
find . -type f -name "*.txt" | xargs rm

```

## 2. grep

```shell
# -i 忽略大小写， -n 显示行号， 查询文件中test的结果
grep -i -n "test" filename
# 高亮显示, centos7后，--color 已经通过alias变为默认
grep -i -n --color "test" filename
# 返回匹配的行数
grep -i -c "test" filename
# -B before 打印匹配行的前几行， -A after，打印匹配行的后几行
# -C context， 围绕的意思， -B7 -A7 等同于-C7
grep -i -B7 -A7 "test" file
# 不包含test的结果
grep -v "test" file
# 等同于 grep -E 
egrep "正则表达式"
# 支持普通的写死的关键字，速度快
fgrep "test" filename
```

## 3. sed

```shell
# 将每一行出现的第一个test替换为replace_text
sed 's/text/replace_text/' filename
# 将所有出现的test替换为replace_text
sed 's/text/replace_text/g' filename
# -i 可以将修改反应到文件中
sed -i 's/text/repalce_text/g' file
# -i.bak 会生成一个file.bak的原文件，file将存储修改后的结果
sed -i.bak 's/text/repalce_text/g' file
# 移除空白行， ^表示行首，例如^abc表示abc为首的行， $表示尾， abc$表示abc为尾的行， ^$表示空行
sed '/^$/d' file
# 执行多次操作，可以用-e
echo -e 'hello world' | sed -e 's/hello/A/' -e 's/world/B/'
echo -e 'hello world' | sed 's/hello/A/;s/world/B/'
# 用正则表达式
echo "hello world" | sed -r 's/(hello)|(world)/A/g'
# 指定行数查询， 4s,表示只替换4行的对应的文本，其他不管
sed –n '4s/hello/A/' message
# 表示2-4行的文本替换，其他不管
sed –n '2,4s/hello/A/' message
# 表示2行开始向下数4行的文本替换，其他不管，2-6行
sed –n '2,+4s/hello/A/' message
# 表示最后一行 $表示最后一行
sed –n '$s/hello/A/' message
# !是取反的意思，表示除了第一行外
sed -n '1!s/hello/A/' message

## 子命令
# 将message文件中每一行下边都插入添加一行内容是A。
sed 'a A' message
# 将message文件中每一行上边都插入添加一行内容是A。
sed 'i A' message
# 将message文件中的每一行都替换为A
sed 'c A' message
sed '1,2c A' message # 可适用前面加数字
# 将message文件的1-3行删除
sed '1,3d' message
# 将message文件的a替换为A，b替换为B， 只能替换字符，不能替换字符串
sed 'y/ab/AB/' message
# = 打印行号
sed '1,2=' message
# 将a.txt的内容读取出来加入到第二行的下面
sed '2r a.txt' message
# -r表示当前使用正则， 而\3\2\1分别表示，正则括号匹配的第三组，第二组，第一组数据， 例如 h 1 a 得到的结果会是 a 1 h
sed -r ‘s/([a-z]+)( [0-9]+ )([a-z]+)/\3\2\1/’ message
# -r表示当前使用正则， &表示全部匹配的结果， 111&222， 表示111{匹配结果}222， 即如果是 h 1 a 的话结果是 111h 1 a111，如果/&/则没有任何变化
sed -r ‘s/([a-z]+)( [0-9]+ )([a-z]+)/111&222/’ message
# 在每行前后加上111，222
sed -r ‘s/.*/111&222/’ message
# 把每行的i换成A， g可以认为是 global， 没有g的话，只更换每一行的第一个
sed ‘s/i/A/g’ message
# 把每行的第2个匹配项进行替换
sed ‘s/i/A/2’ message
# -n为静默模式，p的话会强制打印被修改的行， 因此此处打印被修改的行
sed -n ‘s/i/A/p’ message
# 完成替换后，将文件写到b.txt中
sed -n ‘s/i/A/w b.txt’ message
# 无视大小写
sed -n ‘s/i/A/i’ message
```

## 4. awk

```shell
# NR:表示记录数量，在执行过程中对应当前行号；
# NF:表示字段数量，在执行过程总对应当前行的字段数；
# $0:这个变量包含执行过程中当前行的文本内容；
# $1:第一个字段的文本内容；
# $2:第二个字段的文本内容；
# 打印记录行号：第一个字段-第二个字段
ls -l| awk '{print NR":"$1"-"$2}'
# 统计文件的行数， 打印最后一行的行号
awk ' END {print NR}' file
# 累加数据，BEGIN表示开始执行，END表示结束执行， 中间的{}中表示每条记录都会重复触发的逻辑
echo -e "1\n 2\n 3\n 4\n" | awk 'BEGIN{num = 0 ;print "begin";} {sum += $1;} END {print "=="; print sum }'
#  输入来自stdin， $var 为定义好的系统变量
echo | awk '{print vara}' vara=$var 
# 输入来自文件，file为文件
awk '{print vara}' vara=$var file 
# 输出小于行号5的结果
awk 'NR < 5'
# 打印行号为1，4的数据
awk 'NR==1,NR==4 {print}' file
# 返回行文本中不带linux的行， ！去掉则相反，可以用正则表达式
awk '!/linux/'
# -F定义定界符， 此处设置:为定界符
awk -F: '{print $NF}' /etc/passwd
```

## 5. lsof

```shell
# 查询端口占用进程状态
lsof -i:3306
# 查询指定用户的进程打开的文件
lsof -u username
# 查询init进程打开的文件
lsof -c init
# 查询23295号进程打开的文件
lsof -p 23295
# 查询mydir1下被打开的文件， +d表示递归
lsof +d mydir1/
```

