# ES的CRUD

## 1. 写数据的流程

1. 客户端通过hash选择一个node发送请求，这个node被称为协调节点
2. 协调节点对document进行路由，将请求发送到对应的primary shard
3. primary shard处理请求
   1. 先写入内存buffer，buffer的数据无法查询到，同时生成一个translog文件
   2. 执行refresh指令（buffer满了，或者每隔1s（默认，可改）进行refresh，或者手动调用api进行refresh）
   3. buffer数据写入os cache中，代表可以被查询到
   4. 将buffer中的数据写入segment文件中（新建），同时建立倒排索引库
   5. buffer清空，translog保留，在translog越来越大的时候（一定长度），会触发commit操作，即刷到os cache中等待刷盘。segment越来越多的时候会触发merge操作，多个segment合并为一个大的segment。同时删除deleted的doc（更新和删除文档的时候进行标记）
   6. buffer、translog os cache, segment file os cache有5s的数据不在磁盘上，如果这时候宕机，可能会导致当时的数据丢失。可改
4. primary shard 同步到replica shard
5. 协调节点在其他都处理完的情况下，返回客户端