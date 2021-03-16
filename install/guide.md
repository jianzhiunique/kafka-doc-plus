# Kafka 2.5.1完整安装向导

## 安装zookeeper

kafka目前依赖zookeeper来完成核心的分布式逻辑，但在不久的将来，kafka将不再依赖zookeeoper

有关zookeeper的安装，请直接阅读[zookeeper安装](../zookeeper/install.md)

安装前请确定kafka版本当时对应的zookeeper版本，避免因为zookeeper与kafka不兼容导致问题

## 安装前确认

按照Kafka调优部分的文档，建议生产环境使用以下机器配置
1. CPU核数 16+
2. 内存32G+
3. 磁盘使用普通硬盘即可，转速7200+，磁盘可以是多个（官方也推荐多个），磁盘容量越大越好
4. 网卡带宽越高越好

确认操作系统的某些设置：
1. 至少100000的文件描述符，可根据分区数 * （分区大小 / segment段大小）计算
2. 确认socket缓冲区的配置
3. vm.max_map_count
4. kafka的数据磁盘不要与其他应用共享
5. 最好用XFS格式化kafka的数据盘
6. 磁盘挂载使用noatime，可选XFS：largeio、nobarrier
7. 安装了JDK1.8 官方推荐JDK 1.8 u5
8. 确认节点间端口、与zookeeper端口是连通的


## 下载kafka

http://kafka.apache.org/downloads

例如
```
wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.5.1/kafka_2.12-2.5.1.tgz
tar -zxvf kafka_2.12-2.5.1.tgz
mv kafka_2.12-2.5.1 /usr/local/kafka

# kafka的数据将放在这里
mkdir -p /data/
# 程序的日志将放在这里
mkdir -p /home/logs/
```

## 修改配置文件前的准备

如果启用SSL、SASL、ACL等机制，需要在修改配置前完成相关准备工作。

[SSL配置](ssl.md)、[SASL配置](sasl.md)、[ACL配置](acl.md)

## 修改kafka配置文件

配置文件应当通过机器配置、业务需求等进行合理的配置。

#### 不使用Kafka安全机制
```
### 服务器基本配置
broker.id=0

### socket server配置
# 服务端监听的端口
listeners=INTERNAL://x.x.x.x:9092
# 对外暴露的服务地址,如果这个不设置将会用listeners的值
#advertised.listeners=
# 名称与协议的映射map
listener.security.protocol.map=INTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
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

security.inter.broker.protocol=PLAINTEXT

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
zookeeper.connect=x.x.x.x:2181,x.x.x.x:2181
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

#### 使用Kafka安全机制

参考[完整配置例子](demo_config.md)、[SSL配置](ssl.md)、[SASL配置](sasl.md)、[ACL配置](acl.md)

在启用SASL认证机制的情况下，需要类似以下配置jaas文件
```
# cat /usr/local/kafka/config/kafka_server_jaas.conf
internal.KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required;
};
external.KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required;
};
brokers.KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin"
        password="xxxxx"
        user_admin="xxxxx";
};
controller.KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin"
        password="xxxxx"
        user_admin="xxxxx";
};
```

## 编写Kafka启动脚本/etc/init.d/kafka
```
#!/bin/bash
### BEGIN INIT INFO
# Provides:          baojizhong
# Required-Start:    $local_fs $network $named $time $syslog
# Required-Stop:     $local_fs $network $named $time $syslog
# Default-Start:     3 4 5
# Default-Stop:      0 1 2 6
# Description:       kafka
### END INIT INFO
export PATH=/usr/bin:$PATH
export JMX_PORT=9999
KAFKA_PATH=/usr/local/kafka
KAFKA_LOG_PATH=/home/logs/kafka
KAFKA_HEAP_OPTS="-Xms12g -Xmx12g -XX:MetaspaceSize=96m -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80 -XX:ParallelGCThreads=16 -Djava.security.auth.login.config=/usr/local/kafka/config/kafka_server_jaas.conf -Dzookeeper.sasl.client=false"
case $1 in
          start)
              LOG_DIR=$KAFKA_LOG_PATH KAFKA_HEAP_OPTS=$KAFKA_HEAP_OPTS $KAFKA_PATH/bin/kafka-server-start.sh -daemon $KAFKA_PATH/config/server.properties
              ;;
          stop)
              $KAFKA_PATH/bin/kafka-server-stop.sh
              echo -n '正在关闭'
              while :
              do
                success=`jps | grep Kafka | wc -l`
                if [[ $success -eq 1 ]];then
                    echo -n '- '
                    sleep 3s
                else
                    echo ''
                    echo '关闭完成- - - - - - '
                    exit 0
                fi
              done
              ;;
          status)
              jps
              ;;
          restart)
              $KAFKA_PATH/bin/kafka-server-stop.sh
              echo -n '正在关闭'
              while :
              do
                success=`jps | grep Kafka | wc -l`
                if [[ $success -eq 1 ]];then
                    echo -n '- '
                    sleep 3s
                else
                    echo ''
                    echo '关闭完成- - - - - - '
                    break
                fi
              done
              LOG_DIR=$KAFKA_LOG_PATH KAFKA_HEAP_OPTS=$KAFKA_HEAP_OPTS $KAFKA_PATH/bin/kafka-server-start.sh -daemon $KAFKA_PATH/config/server.properties
              ;;
          *)
              echo "$0 start|stop|status|restart"
              ;;
esac
```

```
chmod +x /etc/init.d/kafka
```

## 逐台启动Kafka并检查错误日志

```
service kafka start
tail -f server.log
```
