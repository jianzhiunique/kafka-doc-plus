# 为Topic选择分区数

为Topic选择合适的分区数是实现高并发的关键，分区上均衡的负载是高吞吐量的关键因素。需要基于期望的生产者和消费者的吞吐量为每个分区作出好的选择。

例如，如果希望每秒读取1GB，但是消费者每秒只能处理50MB数据，那么你至少需要20个分区，20个消费者组成的消费者组。

类似的，加入想为生产者达到这一效果，每个生产者每秒可以写入100MB的数据，那么至少需要10个分区。

在这种情况下，如果有20个分区，可以保持1G/s的速度来生产消费。

应该根据生产者和消费者的数量调整分区的确切数量，以便生产者和消费者都能达到目标吞吐。

所以简答的公式是：
```
#Partitions = max(NP, NC)

NP is the number of required producers determined by calculating: TT/TP
NC is the number of required consumers determined by calculating: TT/TC
TT is the total expected throughput for our system
TP is the max throughput of a single producer to a single partition
TC is the max throughput of a single consumer from a single partition
```

这个计算可以粗略地计算分区的数量。这是好的开始，要提高分区的数量，需要记住以下事项：

1. 分区数可以在创建Topic或者以后指定
2. 增加分区数量会影响打开的文件描述符的数量，确保文件描述符限制
3. 重新分配分区可能会非常昂贵，分区数多一点比少一点好
4. 更改基于key的分区是一项挑战，并且需要手动复制
5. 不支持减少分区数，但可以创建新的主题，并复制数据
6. 分区的元数据是存在zookeeper的，以znode的形式。大量分区会影响zookeeper和客户端：1.不必要的分区给zookeeper更大的压力（更多网络请求），可能会增加控制器的延迟，或者分区leader选举的延迟 2.生产者和消费者需要更多的内存，因为他们需要追踪更多的分区以及相关的缓冲数据
7. 作为性能优化的准则，每个代理的分区数不应该超过4000，集群中的分区数不应该超过200000个。

通过监控消费者lag，确保消费者不会落后于生产者。