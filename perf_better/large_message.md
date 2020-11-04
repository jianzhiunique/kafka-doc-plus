# 处理超大消息

Kafka可以调整来处理很大的消息，涉及以下方面：

1. 服务端的最大消息大小
2. 服务段的文件大小
3. 生产者/消费者的消息大小配置

但是在调整之前，建议先尝试按照下面的方法减少消息的大小

1. 使用生产者的压缩功能，像XML这种格式的文本能获得良好的压缩效果。通过设置compression.type来实现，支持的压缩算法有gzip, lz4, Snappy, and Zstandard
2. 如果有共享存储可以用，如NAS, HDFS, S3，考虑把大文件放到共享存储，只把文件路径发送到kafka，这样通常快得多
3. 将消息分成小段，然后使用分区key使他们发到相同的分区，然后消费者还原消息

如果以上方法都不行，依然要使用大消息，需要配置下面的参数

1. message.max.bytes 服务端参数，默认是1M，服务端可接受的最大消息大小
2. log.segment.bytes 服务端参数，默认是1G，Kafka数据文件大小，必须大于任何单个消息大小
3. replica.fetch.max.bytes 服务端参数，默认是1M，副本复制时能接受的最大大小，必须大于message.max.bytes，否则可能会无法接收，数据丢失
4. max.partition.fetch.bytes，消费者参数，默认10M，服务端返回单分区的最大数据量
5. fetch.max.bytes，消费者参数，默认50M，服务端为每个fetch请求返回的最大数据量

注意：消费者可以消费到大于max.partition.fetch.bytes、fetch.max.bytes的batch，但这意味着batch中只有一条消息，可能会导致性能问题。