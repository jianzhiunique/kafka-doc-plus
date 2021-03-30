# Kafka多数据中心

## 历史

Kafka早期版本提供了MirrorMaker，可以将Topic数据从一个集群复制到另一个集群，内部原理是消费者加生产者的模式，可以用来做集群数据迁移之类的工作。

然而MirrorMaker固然是简单的，并且功能上有些缺陷，用户对此不满意。

Linkedin的Brooklin https://github.com/linkedin/Brooklin/ 、Uber的uReplicator https://github.com/uber/uReplicator 就是代表产品，
他们是早期MirrorMaker的探索者，但都发现MirrorMaker存在很大的问题，于是他们用自己的方式分布式地管理了MirrorMaker的实例，并取得了很大的成功。

Confluent公司则推出了他们的商业版本，Confluent Replicator，需付费（并且比较贵），功能方面应该是完备的，Confluent在白皮书里提到过多数据中心的细节，感兴趣的可以去找找。

再后来，Kafka Connect框架推出，作为kafka与其他数据源对接的工具；使用Kafka Connect可以将外部数据源的数据导入kafka，也可以从Kafka导出数据到其他数据源；

除此之外，还有一个开源项目Salesforce的Mirus https://github.com/salesforce/mirus ，也是基于kafka connect的多数据中心同步工具。

以上都不是本文的重点，本文重点是MirrorMaker2，最早是在Kafka 2.4版本引入

相关KIP：https://cwiki.apache.org/confluence/display/KAFKA/KIP-382%3A+MirrorMaker+2.0

github：https://github.com/apache/kafka/tree/trunk/connect/mirror

## 解决的问题

MirrorMaker2是为了解决MirrorMaker1存在的不足，MirrorMaker1主要有以下痛点：

1. 缺失了对消费者提交位移的数据同步，仅仅对Topic数据进行复制，不对__consumer_offsets进行同步，也不会有位移转换，只能基于时间戳进行failover的位移恢复，显然这是不准确的
2. 部署和监控非常困难，没有中心化的控制面，每个消费者和生产者配置都是分离的，也没有高级指标metric支持
3. 不能让Topic保持同步，因为Topic配置不会同步、分区数不会同步，ACL也不会同步

MirrorMaker2是如何解决这些痛点的：

1. 位移转换，即同一条消息在多个数据中心的offset对应关系；消费者组检查点checkpoints，即消费者位移在不同数据中心的对应关系
2. 高层次的“驱动”管理了多个集群间的复制；高层次的配置文件定义了全局复制拓扑；引入了像是复制延迟这种监控指标
3. 可以同步Topic配置、分区、ACL配置等

## 原理

MirrorMaker2提供了3个Connector：MirrorSourceConnector, MirrorCheckpointConnector, MirrorHeartbeatConnector

#### MirrorSourceConnector

1. 复制远程Topic的数据
2. 同步Topic的配置以及ACL的配置
3. 在*源头集群*写入被复制消息的offset映射关系
4. 如果*源头集群*有heartbeats（心跳主题），也会进行复制

```
clusterA------------------------------- MirrorSourceConnector ----------------clusterB
topic1  ------------------------------> records、configs、acls --------------> clusterA.topic1
mm2-offset-syncs.clusterB.internal <--- offset syncs <---------------------------- 
```

#### MirrorCheckpointConnector

1. 消费MirrorSourceConnector在*源头集群*产生的offset映射关系，配合*源头集群*的__consumer_offsets，生成*目标集群*上的checkpoint内部Topic
2. 支持failover，2.7.0之前，MirrorMaker2提供了一个工具类，可以读取checkpoint主题，获取原来机房消费者的提交位移；2.7.0开始，checkpoint主题可以定期转化到目标集群的__consumer_offsets

```
clusterA------------------------------- MirrorCheckpointConnector ---------------- clusterB
mm2-offset-syncs.clusterB.internal ---> emit checkpoints ------------------------> clusterA.checkpoints.internal
        (+)                                                                        (schedule transfer to)
__consumer_offsets                                                                 __consumer_offsets
```

#### MirrorHeartbeatConnector

1. 发送心跳数据到*源头集群*的心跳主题 heartbeats
2. 在监控复制流方面十分有用
3. 可以帮助客户端发现复制的拓扑
4. 为了附带工具类 mirror-clients的upstreamCluster()方法

```
MirrorHeartbeatConnector --------  clusterA------------------- MirrorSourceConnector ---------------- clusterB
emit heartbeats -----------------> heartbeats ---------------> sync topic --------------------------> clusterA.heartbeats
```

## 部署方式

MirrorMaker2支持两种部署方式

#### 驱动模式

特点

1. 使用connect-mirror-maker.sh 运行程序
2. 配置比较简单
3. 一个程序运行了所有connectors，驱动模式实际上是包装了Target Connect（包含Source Connector和Checkpoint Connector）以及Source Connect(包含Heartbeat Connector)

劣势
1. 目前来说，存在更改配置需要重启的情况

#### Connect模式

1. 运行在现有的Connect集群之上，复用connect集群
2. 可以有完整的控制
3. 需要更多配置

劣势
1. 相比驱动模式，确实需要更多配置，文档说明不充分
2. 只能复制Connect集群所在的kafka集群的数据，不能在*运行在Kafka1上的connect集群*上进行kafka2 -> kafka3的同步

## 备份方式

#### Active/Standby 主/备

主/备模式下，生产者写入Topic，Topic数据将同步其他数据中心进行冷备，其他数据中心的消费者不工作，仅在必要时（如failover）启动，使用历史数据进行恢复

```

DC A                          DC  B
--------                      ---------
P1 -> TopicA -> C1    --->    dc_A-TopicA -> C1(备)

```

驱动模式下，可以通过使用数据流向配置enable A->B, disable B->A 来实现
Connect模式下可以只在DC B部署相关Connector来实现

#### Active/Active 主/主

主/主模式下，生产者写入Topic，Topic数据同步到其他数据中心，被其他数据中心的消费者消费

```

DC A                          DC  B
--------                      ---------
P1 -> TopicA -> C1    --->    dc_A-TopicA -> C2
dc_B-TopicA  -> C1    <---    P2 -> TopicA -> C2

```
驱动模式下，可以通过使用数据流向配置enable A->B, enable B->A 来实现
Connect模式下可以在两个机房都部署相关Connector来实现

## Connect集群上部署例子

截止目前来说，个人推荐使用Connect集群的模式来运行。假设我们有两个数据中心A,B，这里我先配置A->B的数据链路

#### 首先，在B集群上运行Connect集群

下载kafka_2.13-2.7.0.tgz

编写connect distributed配置文件
```
# vim kafka_2.13-2.7.0/config/connect-distributed.properties
bootstrap.servers=B:9092
group.id=connect-cluster
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true

offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=25

config.storage.topic=connect-configs
config.storage.replication.factor=3

status.storage.topic=connect-status
status.storage.replication.factor=3
status.storage.partitions=5

offset.flush.interval.ms=10000
rest.port=8080
plugin.path=/root/kafka_2.13-2.7.0/libs
```
运行Connect集群

```
yum install -y java-1.8.0-openjdk-devel
nohup /root/kafka_2.13-2.7.0/bin/connect-distributed.sh /root/kafka_2.13-2.7.0/config/connect-distributed.properties > /dev/null 2>&1 &
```

#### 编写MirrorSourceConnector配置并提交到Connect集群

```
{
    "name": "public-test",
    "config": {
        "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
        "source.cluster.alias": "A",
        "target.cluster.alias": "B",
        "source.cluster.bootstrap.servers": "A:9092",
        "target.cluster.bootstrap.servers": "B:9092",
        "topics": "mirror-test-.*",
        "replication.policy.separator": "_",
        "refresh.topics.interval.seconds": 60,
        "sync.topic.configs.enabled": "true",
        "sync.topic.acls.enabled": "false"
    }
}

curl -X POST connect:8080/connectors -d "@data.json" --header "content-type:application/json"

```

#### 编写MirrorCheckpointConnector配置并提交到Connect集群

```
{
    "name": "public-test-checkpoint",
    "config": {
        "connector.class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
        "source.cluster.alias": "A",
        "target.cluster.alias": "B",
        "source.cluster.bootstrap.servers": "A:9092",
        "target.cluster.bootstrap.servers": "B:9092"
    }
}

curl -X POST connect:8080/connectors -d "@data.json" --header "content-type:application/json"

```

#### 编写MirrorHeartbeatConnector配置并提交到Connect集群

```
{
    "name": "public-test-heartbeats",
    "config": {
        "connector.class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
        "source.cluster.alias": "A",
        "target.cluster.alias": "B",
        "source.cluster.bootstrap.servers": "A:9092",
        "target.cluster.bootstrap.servers": "B:9092"
    }
}

curl -X POST connect:8080/connectors -d "@data.json" --header "content-type:application/json"

```

#### 同理A->B的配置也是类似的，不再赘述

## Connector相关配置
#### 通用配置
```
source.cluster.alias 源集群的别名
target.cluster.alias 目标集群的别名
source.cluster.bootstrap.servers
target.cluster.bootstrap.servers

public static final String ADMIN_CLIENT_PREFIX = "admin.";
public static final String CONSUMER_CLIENT_PREFIX = "consumer.";
public static final String PRODUCER_CLIENT_PREFIX = "producer.";

source.cluster.consumer.* overrides for the source-cluster consumer
source.cluster.producer.*	overrides for the source-cluster producer
source.cluster.admin.*	overrides for the source-cluster admin
target.cluster.consumer.* overrides for the target-cluster consumer
target.cluster.producer.*	overrides for the target-cluster producer
target.cluster.admin.*	overrides for the target-cluster admin

覆盖关系
props.putAll(originalsWithPrefix(SOURCE_CLUSTER_PREFIX)); 集群配置
props.putAll(originalsWithPrefix(ADMIN_CLIENT_PREFIX)); admin通配
props.putAll(originalsWithPrefix(SOURCE_ADMIN_CLIENT_PREFIX)); 特定集群的admin通配
```

#### MirrorSourceConnector
```
replication.policy.class  org.apache.kafka.connect.mirror.DefaultReplicationPolicy
replication.policy.separator 重命名Topic分隔符，默认为.
refresh.topics.enable 定期检查新Topic
refresh.topics.interval.seconds 
sync.topic.configs.enabled 是否检查源头集群的配置变更
sync.topic.configs.interval.seconds 
sync.topic.acls.enabled 是否检查源头集群的ACL变更
sync.topic.acls.interval.seconds 
replication.factor 创建远程Topic时使用的复制因子，默认为2
topics Topic白名单，支持逗号分割和正则，默认为.*
topics.exclude Topic黑名单，默认为.*[\\-\\.]internal, .*\\.replica, __.*
config.properties.exclude （config.properties.blacklist） 哪些Topic是排除配置检查的 
    follower\\.replication\\.throttled\\.replicas, "
    + "leader\\.replication\\.throttled\\.replicas, "
    + "message\\.timestamp\\.difference\\.max\\.ms, "
    + "message\\.timestamp\\.type, "
    + "unclean\\.leader\\.election\\.enable, "
    + "min\\.insync\\.replicas
offset-syncs.topic.replication.factor 位移同步(位移映射)Topic的复制因子
consumer.poll.timeout.ms 从源头集群pull的超时时间
admin.timeout.ms 从源头集群调用admin方法的超时时间
topic.filter.class DefaultTopicFilter.class
config.property.filter.class DefaultConfigPropertyFilter.class
offset.lag.max 最大offset lag
task.assigned.partitions 
task.assigned.groups
```
#### MirrorCheckpointConnector
```
refresh.groups.enabled 定期检查新的消费者组
refresh.groups.interval.seconds 
sync.group.offsets.enabled 是否定期转化位移到__consumer_offsets
sync.group.offsets.interval.seconds 
groups Groups白名单，默认.*
groups.exclude Groups黑名单，默认console-consumer-.*, connect-.*, __.*
checkpoints.topic.replication.factor 检查点（offset映射）Topic的复制因子
group.filter.class DefaultGroupFilter.class
```
#### MirrorHeartbeatConnector

```
emit.heartbeats.enabled 是否定期产生心跳
emit.heartbeats.interval.seconds 
emit.checkpoints.enabled 是否定期生成消费者位移信息
emit.checkpoints.interval.seconds
heartbeats.topic.replication.factor 心跳Topic的复制因子
```






