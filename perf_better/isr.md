# ISR调优

## in-sync replica (ISR) 

ISR集合的两个条件：

1. follower副本 caught-up leader副本
2. 所在的节点是alive

#### replica.lag.time.max.ms
如果follower副本落后leader”太远“，会被移出ISR集合。这个”太远“的定义是replica.lag.time.max.ms。

如果副本没有发送fetch请求，或者在至少replica.lag.time.max.ms时间内没有消费到leader的LEO位置，副本将会被移出ISR。

#### num.replica.fetchers

这是一个集群层面的配置，来控制节点里有多少个fetcher线程，这些线程负责从leader所在的节点复制消息。

增大此值会导致更高的并行I/O和fetcher的吞吐量。当然，这里是个折衷，节点需要更多的CPU和网络。

#### replica.fetch.min.bytes
控制追随者副本fetch的最小数据量。如果达不到这个量，等待到replica.fetch.wait.max.ms的时间。

#### replica.fetch.wait.max.ms
控制副本拉取到新的消息时最多等待多久，这个值应该小于replica.lag.time.max.ms，否则副本会被踢出ISR。

#### 检查ISR
```
kafka-topics --zookeeper ${ZOOKEEPER_HOSTNAME}:2181/kafka --describe --topic ${TOPIC}
```

#### leader宕机

如果leader宕机，会从ISR中选择一个新的leader。这将没有消息丢失。

如果没有ISR，unclean leader election可能会导致消息丢失。

#### unclean.leader.election.enable

如果此参数设置为true，则运行不完全leader选举，默认值是false


