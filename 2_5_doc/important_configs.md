# 一些重点需要关注的配置

## 最重要的生产者配置

acks、compression、batch size

## 最重要的消费者配置

fetch size

## 示例生产环境服务配置

如果是对于新手，以下配置确实是最需要关注的，以便能快速开始kafka，但是这些配置太少了，一般不能直接用于生产。

```
# ZooKeeper
zookeeper.connect=[list of ZooKeeper servers]

# Log configuration
num.partitions=8
default.replication.factor=3
log.dir=[List of directories. Kafka should have its own dedicated disk(s) or SSD(s).]

# Other configurations
broker.id=[An integer. Start with 0 and increment by 1 for each new broker.]
listeners=[list of listeners]
auto.create.topics.enable=false
min.insync.replicas=2
queued.max.requests=[number of concurrent requests]
```

