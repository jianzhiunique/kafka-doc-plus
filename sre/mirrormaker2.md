# Kafka多数据中心

## 历史

Kafka早期版本提供了MirrorMaker，可以将Topic数据从一个集群复制到另一个集群，内部原理是消费者加生产者的模式，可以用来做集群数据迁移之类的工作。

然而MirrorMaker固然是简单的，并且功能上有些缺陷，用户对此不满意。

Linkedin的Brooklin https://github.com/linkedin/Brooklin/ 、Uber的uReplicator https://github.com/uber/uReplicator 就是代表产品，
他们是早期MirrorMaker的探索者，但都发现MirrorMaker存在很大的问题，于是他们用自己的方式分布式地管理了MirrorMaker的实例，并取得了很大的成功。

Confluent公司则推出了他们的商业版本，Confluent Replicator，需付费（并且比较贵），功能方面应该是完备的，Confluent在白皮书里提到过多数据中心的细节，感兴趣的可以去找找。

再后来，Kafka Connect框架推出，作为kafka与其他数据源对接的工具；使用Kafka Connect可以将外部数据源的数据导入kafka，也可以从Kafka导出数据到其他数据源；

除此之外，还有一个开源项目Salesforce的Mirus https://github.com/salesforce/mirus ，也是基于kafka connect的多数据中心同步工具。

以上都不是本文的重点，本文重点是MirrorMaker2，最早是在Kafka 2.4版本引入，相关KIP：https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0

## 解决的问题

MirrorMaker2是为了解决MirrorMaker1存在的不足，MirrorMaker1主要有以下痛点：

1. 缺失了对消费者提交位移的数据同步，仅仅对Topic数据进行复制，不对__consumer_offsets进行同步，也不会有位移转换，只能基于时间戳进行failover的位移恢复，显然这是不准确的
2. 部署和监控非常困难，没有中心化的控制面，每个消费者和生产者配置都是分离的，也没有高级指标metric支持
3. 不能让Topic保持同步，因为Topic配置不会同步、分区数不会同步，ACL也不会同步

MirrorMaker2是如何解决这些痛点的：

1. 位移转换，即同一条消息在多个数据中心的offset对应关系；消费者组检查点checkpoints，即消费者位移在不同数据中心的对应关系
2. 高层次的“驱动”管理了多个集群间的复制；高层次的配置文件定义了全局复制拓扑；引入了像是复制延迟这种监控指标
3. 可以同步Topic配置、分区、ACL配置等

## MirrorMaker2原理
