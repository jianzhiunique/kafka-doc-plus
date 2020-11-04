# 这是Kafka 2.5.x版本的完整服务端配置例子

```
### 服务器基本配置
broker.id=0

### socket server配置
# 服务端监听的端口
listeners=INTERNAL://x.x.x.x:9092,EXTERNAL://x.x.x.x:9093,BROKERS://x.x.x.x:9094,CONTROLLER://x.x.x.x:9095
# 对外暴露的服务地址,如果这个不设置将会用listeners的值
#advertised.listeners=
# 名称与协议的映射map
listener.security.protocol.map=INTERNAL:SASL_SSL,EXTERNAL:SASL_SSL,BROKERS:SASL_PLAINTEXT,CONTROLLER:SASL_PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
# 接收请求和发送响应用的线程数
num.network.threads=3
# 处理请求用的线程数，可能包含磁盘IO
num.io.threads=8
# 缓冲区设置
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
# 最大的request，防止OOM
socket.request.max.bytes=104857600
# 最多允许的排队请求数，超过后阻塞网络线程
queued.max.requests=500
# 为控制器分配单独的监听
control.plane.listener.name=CONTROLLER

### 日志基本配置
log.dirs=/data/kafka
num.partitions=3
num.recovery.threads.per.data.dir=2
compression.type=producer
message.max.bytes=10485880
min.insync.replicas=1
default.replication.factor=3
log.message.format.version=2.5-IV0
log.message.timestamp.type=CreateTime
log.message.downconversion.enable=true

### SSL相关
ssl.protocol=TLSv1.2
ssl.trustmanager.algorithm=PKIX
ssl.keystore.location=/usr/ca/server/server.keystore.jks
ssl.keystore.password=poMBsx8quMyRatpl
ssl.key.password=poMBsx8quMyRatpl
ssl.truststore.location=/usr/ca/trust/server.truststore.jks
ssl.truststore.password=podVMfAxzGB1jAV8
# 不开启SSL认证，麻烦，只用SASL认证就可以了
ssl.client.auth=none
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
ssl.keystore.type=JKS
ssl.truststore.type=JKS
ssl.endpoint.identification.algorithm=https

### SASL相关
# 启用ACL时，使用inter.broker.listener.name=BROKER来屏蔽security.inter.broker.protocol=PLAINTEXT，因为需要用户名
# 不启用ACL时，内部通讯使用PLAINTEXT，inter.broker.listener.name保持为null，security.inter.broker.protocol=PLAINTEXT是可以的
# 同时设置inter.broker.listener.name、security.inter.broker.protocol会报错
inter.broker.listener.name=BROKERS
#security.inter.broker.protocol=PLAINTEXT
# 让节点内部通信用PLAIN，定义固定的用户名admin和密码，外部的则用SCRAM
sasl.mechanism.inter.broker.protocol=PLAIN
# 需要将所有SASL机制都加进来
sasl.enabled.mechanisms=SCRAM-SHA-512,PLAIN
# 各监听器的sasl.jaas.config统一走kafka_server_jaas.conf文件，各监听器分开xxx.KafkaServer

### ACL相关
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false
super.users=User:kafka_admin;User:admin;User:mqproxy;User:backup


### 内部主题配置 "__consumer_offsets" and "__transaction_state"
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=1
offsets.retention.minutes=10080
offsets.topic.num.partitions=50

### 日志flush策略，kafka推荐依赖操作系统而不是自己设定
# 在持久性、未flush的消息可能丢失如果没有使用副本的话
# 延迟、非常大的刷新间隔可能会导致延迟峰值，因为有大量消息要flush
# 吞吐、flush昂贵，较小的刷新间隔可能导致过度寻道
#log.flush.interval.messages=10000
#log.flush.interval.ms=1000

### 日志保留策略
log.retention.hours=168
log.retention.bytes=2147483648
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

### zookeeper配置
zookeeper.connect=x.x.x.x:2181,x.x.x.x:2181,x.x.x.x:2181
zookeeper.connection.timeout.ms=18000

### 群组协调者配置
# 群组协调者在第一个消费者加入后 ～ rebalance开始前的毫秒延迟
# 主要用来避免应用启动时不必要的rebalance
# 新成员加入后group.initial.rebalance.delay.ms ～ max.poll.interval.ms
# 0 用于开发和测试
# 3000 用于生产环境
group.initial.rebalance.delay.ms=3000

### 管理相关配置
auto.create.topics.enable=false
auto.leader.rebalance.enable=false
unclean.leader.election.enable=false
delete.topic.enable=true
controlled.shutdown.enable=true

### 副本机制相关
# fetcher线程数，可以增加follower的并行IO度
num.replica.fetchers=5
replica.fetch.min.bytes=1
# 如果follower赶不上，会被移出ISR
replica.lag.time.max.ms=30000
# 副本拉取的最长等待时间需要比replica.lag.time.max.ms小，否则会被移出ISR
replica.fetch.wait.max.ms=500
# 副本单分区拉取的最大消息，不是绝对最大值，与message.max.bytes相关
replica.fetch.max.bytes=10485760
# 整个fetch响应的最大值，非绝对最大值，与message.max.bytes相关
replica.fetch.response.max.bytes=104857600

### 日志清理相关
log.cleaner.min.cleanable.ratio=0.5
log.cleaner.threads=1
log.cleanup.policy=delete
```
