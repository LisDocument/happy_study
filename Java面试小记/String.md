# String/StringBuilder/StringBuffer

| String                                                       | StringBuffer                                         | StringBuilder                                                |
| ------------------------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------ |
|                                                              | JDK1.0                                               | JDK5.0                                                       |
| 字符串常量，内容不可变                                       | 字符串变量，线程安全                                 | 字符串变量，线程不安全                                       |
| 发生改变的时候，一般是生成一个新的String对象，然后这个指针指向新的对象，所以内容经常改变的对象最好不要用String，因为每次生成新对象的时候，都会对系统的性能产生影响，尤其当内存中无引用对象多了之后，jvm的GC垃圾处理机制就会运行，导致性能下降 | 由于线程安全，一般用于全局变量，操作是append和insert | 由于线程不安全，一般用于方法内或者确认的单线程中使用，操作时append和insert |
| 操作少量数据                                                 | 多线程操作大量数据                                   | 单线程操作大量数据                                           |
| 不要使用stirng+                                              | 指定容量初始化                                       | 指定容量初始化                                               |
| 执行速度快                                                   | 低于builder10%-15%                                   |                                                              |

StringBuffer和StringBuilder在Java9后将内部的char数据修改为了byte数组实现。

在Java9中是利用InvokeDynamic将字符串拼接的优化与javac生成的字节码解耦。

**（-XX:+PrintStringTableStatistics）**打印字符串空间

**（-XX:StringTableSize=N）**手动调整大小

**（-XX:+UseStringDeduplication）**G1 GC下字符串的排重，会将相同数据的字符串指向同一份数据。

Jdk1.8之后intern方法返回的String才是对应元空间地址的String

