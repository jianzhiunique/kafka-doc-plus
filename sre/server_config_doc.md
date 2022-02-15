# Kafka服务端参数详解

从kafka 0.9.0 到 kafka 3.0，服务端配置文件`server.properties`中的参数越来越多，这段时间参数一共有237项，kafka 3.0版本中废弃了6个参数，剩余231项。对于新手，虽然如kafka文档中所说，仅需要几个核心参数就能把kafka运行起来，这主要得益于kafka代码中的默认值。但是对于kafka管理员来说，还是应该知道参数是用来干嘛的，这么多配置项真是让人头大。

kafka的这些服务端参数中，有一部分参数是kafka 0.9.0版本开始就有的（0.9.0版本算是kafka的一个里程碑版本，0.9.0之前的版本没有作为我的研究范围），其他参数则是在kafka版本迭代的过程中，因为新特性/BUG修复/KIP而逐渐引入的，某些参数的默认值也随着版本变化而变化。

本文的目的主要有：

- 相对简单清晰地说明kafka服务端的参数含义，因为官方文档有时晦涩难懂并且含义不清
- 对参数进行用途归类，以及区分核心/重要/其他参数，因为官方文档应该是自动导出的，没有分类
- 在必要时增加官方文档之外的补充说明，比如原理、为什么这么做、生产环境推荐怎么配置
- 明确参数各版本的变化历程，追踪参数的引入版本，追溯部分参数的KIP，以便了解参数引入原因

## 摘要

分类：Kafka、Kafka实战

关键词：

版本说明：Kafka最低版本为0.9.0，最高版本为3.0

## 核心参数

#### broker.id

- 通常Kafka以集群的形式提供服务，集群由多个Kafka程序组成，此参数用于给单个kafka编号，如 1、2、3，这个id在集群中都是唯一的
- 此参数默认值为-1，如果不修改设置，id将会自动生成，见参数`broker.id.generation.enable`
- 如果集群中既有用户手动设定的id，又有自动生成的id，为了避免冲突，通过参数`reserved.broker.max.id`进行区分，自动生成的id是从` reserved.broker.max.id + 1`开始的

#### zookeeper.connect

- 此参数是用来指定kafka所使用的zookeeper的连接地址，地址可以是chroot path的形式
- 示例 `hostname1:port1,hostname2:port2,hostname3:port3/chroot/path`
- Kafka在3.0版本起正式引入KRaft机制（暂且作为测试用途），目的是为了将来的版本中替换zookeeper，但在此之前，Kafka都是用zookeeper进行分布式协同的

#### log.dirs

- Kafka的数据保存在哪些目录下
- 如果不设置，则会使用`log.dir`的值，生产环境多配置`log.dirs`而不是`log.dir`，因为生产环境往往有多个磁盘目录
- 示例 `/data1,/data2,/data3`

## 重要参数

#### listeners

- 此参数用于指明服务端的listener列表，kafka启动时会在这些端口上开启socket监听，每个listener格式为 `名称://hostname:port`, 名称和端口必须是唯一的
- 示例 `PLAINTEXT://myhost:9092,SSL://:9091,CLIENT://0.0.0.0:9092,REPLICATION://localhost:9093`
- 其中hostname处 `0.0.0.0`表示绑定所有网卡，留空则表示绑定到默认网卡
- 如果监听器的名称不是安全协议（PLAINTEXT、SSL之类的）的话，`listener.security.protocol.map`也必须设置，因为`listener.security.protocol.map`表示的是名称与协议的映射关系
- 0.9.0版本引入，对应KIP-2（允许监听多个端口和IP）

#### advertised.listeners

- 此参数用于指明发布到zookeeper上供客户端使用的listener列表，kafka的客户端发现并连接这些地址。
- 与`listeners`可以不一样，因为在IaaS环境下，对外服务地址可能会与服务端绑定的接口不一致。如果此参数没有设置，则会使用`listeners`的值
- 与`listeners`不同，此参数不能用`0.0.0.0`，端口可以重复，以便可以公开另一个地址。这在有多个负载均衡器的场景下很有用。
- 0.9.0版本引入，对应KIP-2（允许监听多个端口和IP）

#### delete.topic.enable

- 能否删除topic
- 如果没有开启，admin工具的删除操作不会有效果
- 生产环境中通常会允许topic删除

#### auto.create.topics.enable

- 能否自动创建topic
- 推荐关闭自动创建，否则客户端可以直接创建，不可控

#### unclean.leader.election.enable

- 是否可以让不在ISR中的副本成为leader（开启的情况下会有丢消息的可能性）
- 推荐关闭
- 此参数在0.11.0之前默认是true，0.11.0开始默认是false

#### auto.leader.rebalance.enable

- 是否开启leader自动平衡
- 自动平衡机制：有一个后台线程会在固定的时间间隔（参数`leader.imbalance.check.interval.seconds`）检查分区leader的分布情况，如果leader的不平衡度达到参数`leader.imbalance.per.broker.percentage`的值，将会触发leader的重新平衡到优先leader副本上
- leader平衡期间生产者消费者可能会短时间不可用，如果不能在业务高峰期承担这种风险，建议关闭

#### log.retention.hours

- 日志文件在删除前保留的小时数，即数据保留多长时间
- 此参数为`log.retention.ms`的第三补充，但却是生产环境的常用配置，因为更易读

#### log.retention.bytes

- 日志文件在删除前保留的最大大小，即数据最多保留多少字节，是单个分区维度的
- 生产环境推荐使用-1，以避免某些情况下的数据丢失

#### num.network.threads

- 服务端用于读取请求 & 发送响应的线程数
- 默认值为3，生产环境可以适当调大，以提高并发能力

#### num.io.threads

- 服务端用于处理请求（KafkaApis具体逻辑）的线程数，逻辑有些包含了磁盘的I/O
- 默认值为8，生产环境可以适当调大，以提高并发能力

#### num.recovery.threads.per.data.dir

- 每个数据目录下用于启动时恢复数据 & 关闭前数据刷盘的线程数
- 默认值为1，适当调大可以加快启动/关闭速度，尤其是大量日志数据文件的场景

#### min.insync.replicas

- 当生产者设置`acks`为`all（-1）`时，此参数指定最小的副本数，至少有这些数量的副本必须确认写入，才认为是生产成功的，如果最小值不能满足，那么生产者会产生一个异常（`NotEnoughReplicas`/`NotEnoughReplicasAfterAppend`）
- 此参数跟`acks`参数一起使用以达到强制的更持久的保证。典型用例是，一个复制因子为3的Topic，配合此参数为2，生产者`acks`为all，来确保当多数副本没有收到写入时，生产者抛出异常
- 生产环境推荐使用2，以防止数据丢失

#### num.partitions

- 每个Topic分区数的默认值
- 默认值为1，生产环境推荐使用3，并且在创建Topic时明确指定分区数
- 在自动创建Topic`auto.create.topics.enable`开启时，此参数也会用到

#### message.max.bytes

- kafka允许的最大的消息批次的大小，如果开启了压缩，则是压缩后的最大大小
- 如果此参数调大了，在0.10.2版本后的消费者，必须也调大拉取大小，以便可以拉取到这个大小的批次
- 在最近的消息格式版本中，为了效率，消息总是会分组为批次，但是在之前的版本中，未被压缩的消息不会被分组为批次，所以此参数也同样限制了一条消息的最大大小
- Topic层面的参数`max.message.bytes`可以覆盖此参数
- 默认值为1M，生产环境可以适当调大，如10M，主要为了防止业务上消息太大发不了，也可以根据实际业务来调整

#### default.replication.factor

- 自动创建Topic时，默认的复制因子
- 生产环境建议至少为2，以防止数据丢失

## 其他参数

### 杂项

#### broker.id.generation.enable

- 是否自动生成broker id
- 如果开启了此参数，应该检查`reserved.broker.max.id`参数是否合理
- 0.9.0版本引入，对应issue KAFKA-1070

#### reserved.broker.max.id

- 手动指定时`broker.id`可以使用的最大值
- 因为此参数是手动指定 & 自动生成的分界
- 0.9.0版本引入，对应issue KAFKA-1070

#### broker.rack

- 指定broker所在的机架，会被用于机架感知的副本分配中，以便提高容错能力
- 在考虑机柜可能故障/断电、多数据中心等场景下可能用到
- 0.10.0版本引入，对应KIP-36（机架感知的副本分配）

#### background.threads

- 用于处理多种后台任务的线程数

#### queued.max.requests

- 最大排队的请求数，超过将阻塞网络线程
- 可以理解为最大并发请求数，默认值是500，生产环境可以适当调大

#### queued.max.request.bytes

- 最大排队的字节数
- 1.0.0版本引入，对应KIP-72（允许对进入的请求所耗用的内存进行限制）

#### compression.type

- topic的最终的压缩类型
- 合法值有：标准压缩编码`gzip, snappy, lz4, zstd`、不压缩`uncompressed`、保留生产者指定的原始压缩格式`producer`
- 0.9.0版本引入，相关issue KAFKA-1499

#### fetch.max.bytes

- 服务端对拉取请求最大响应的字节数，必须至少为1024
- 2.5版本引入，对应KIP-541（为服务端创建fetch.max.bytes配置）

#### request.timeout.ms

- 用于控制客户端最大等待请求被响应的时间，超过了这个时间没有收到响应，客户端将会在必要时重新发送请求，或者在重试次数耗尽后失败

#### alter.config.policy.class.name

- 修改配置时用于校验的策略类，需实现`org.apache.kafka.server.policy.AlterConfigPolicy`接口
- 0.11.0版本引入，对应KIP-133（描述和更改配置的Admin API）

#### create.topic.policy.class.name

- 创建Topic时用于校验的策略类，需实现`org.apache.kafka.server.policy.CreateTopicPolicy`接口
- 0.10.2版本引入，对应KIP-108（创建Topic策略）

#### num.replica.alter.log.dirs.threads

- 在日志目录间移动副本的线程数，可能包含磁盘I/O
- 1.1.0版本引入，对应KIP-113（支持副本在日志目录间转移）

### zookeeper相关

#### zookeeper.set.acl

- 是否让kafka在创建znode的时候使用ACL，ACL机制可以防止kafka在zookeeper上的数据被篡改
- 生产环境中，也可以通过让zookeeper的网段仅kafka可访问保证数据安全
- 0.9.0版本引入，对应KIP-38（zookeeper认证）

#### zookeeper.sync.time.ms

- zk follower可以落后zk leader多少时间，默认值为2s
- 参数意义不太明确，在kafka源码中是zkSyncTimeMs，旧版本消费者中也用到了，猜测是zookeeper的客户端应该设置的一项参数
- 经网上查找：zookeeper的sync API是使得client当前连接着的ZooKeeper服务器，和ZooKeeper的Leader节点同步（sync）一下数据

#### zookeeper.connection.timeout.ms

- kafka等待与zookeeper建立连接的最大时间
- 如果没有设置，则使用`zookeeper.session.timeout.ms`

#### zookeeper.session.timeout.ms

- kafka与zookeeper的session超时时间

#### zookeeper.max.in.flight.requests

- 指定kafka可以发往zookeeper的最大未得到响应的请求数，超过则开始阻塞
- 1.1.0版本引入，对应KIP-214（服务端添加zookeeper.max.in.flight.requests）

### 优先副本相关

#### leader.imbalance.check.interval.seconds

- controller进行分区leader平衡检查的频率

#### leader.imbalance.per.broker.percentage

- 每个broker所允许的leader不平衡的比率,这个值使用的是百分比
- 如果大于了设定值，控制器则会触发leader平衡

### 监听与协议相关

#### ~~host.name~~

- 不推荐，应该用`listeners`，kafka 3.0版本起彻底废弃
- 仅在listeners没有设置时使用，用于指明broker的hostname
- 如果设置了，将只绑定到此地址；如果没有设置，将绑定到所有接口

#### ~~port~~

- 不推荐，应该用`listeners`，kafka 3.0版本起彻底废弃
- 仅在listeners没有设置时使用，用于指明监听和接收连接的端口

#### ~~advertised.host.name~~

- 不推荐，应该用`advertised.listeners`，kafka 3.0版本起彻底废弃
- 仅在`advertised.listeners`、`listeners`没有设置时使用，公布到zookeeper上供客户端使用的hostname，因为在IaaS环境下，可能需要不同于服务端绑定地址的地址
- 如果没有设置，如果`host.name`设置了，将使用`host.name`指定的值，否则，将使用`java.net.InetAddress.getCanonicalHostName()`的值

#### ~~advertised.port~~

- 不推荐，应该用`advertised.listeners`，kafka 3.0版本起彻底废弃
- 仅在`advertised.listeners`、`listeners`没有设置时使用，公布到zookeeper上供客户端使用的端口，因为在IaaS环境下，可能需要不同于服务端绑定地址的地址
- 如果没有设置，则会公布与服务端绑定一致的端口。

#### control.plane.listener.name

- 用于controller和broker之间通讯的listener的名字

- broker将使用此参数来找到listener，然后监听，以便建立与controller的连接

- 如果没有明确配置，将不会有端点让控制器来连接

- 2.2.0 版本引入，对应KIP-291（将控制器连接和请求与数据平面分开）

- 由于此参数不容易理解，简单说一下KIP-291，最初所有的内部请求、外部请求、控制器发出的请求都是用了同一个listener，控制器这种高优先级的请求有时会被其他请求卡住，后来有人说为请求区分优先级，但优先级会造成复杂度，kafka直接让内部请求走内部专用listener、控制器请求走控制器专用listener、外部客户端的请求走外部listener，从而在不增加复杂度的前提下解决请求问题

  ```
  比如以下配置
  listeners = INTERNAL://192.1.1.8:9092, EXTERNAL://10.1.1.5:9093, CONTROLLER://192.1.1.8:9094
  listener.security.protocol.map = INTERNAL:PLAINTEXT, EXTERNAL:SSL, CONTROLLER:SSL
  control.plane.listener.name = CONTROLLER
  
  在启动时，broker将启动并使用SSL监听192.1.1.8:9094
  在控制器那边，当他发现broker通过zookeeper公开的端点后，他将使用此参数寻找端点，并与broker建立连接。
  例如：
  如果broker公布在zookeeper上的端点是"endpoints" : ["INTERNAL://broker1.example.com:9092","EXTERNAL://broker1.example.com:9093","CONTROLLER://broker1.example.com:9094"]
  controller broker配置为
  listener.security.protocol.map = INTERNAL:PLAINTEXT, EXTERNAL:SSL, CONTROLLER:SSL
  control.plane.listener.name = CONTROLLER
  
  controller将会使用broker1.example.com:9094和SSL来连接broker
  ```

#### inter.broker.listener.name

- 用于broker之间通讯的监听器
- 如果没有设置，监听器的名字是被参数`security.inter.broker.protocol`所定义的
- 同时定义`security.inter.broker.protocol`和`inter.broker.listener.name`是不正确的
- `inter.broker.listener.name`的引入是为了替代`security.inter.broker.protocol`
- 0.10.2版本引入，对应KIP-103（将内部外部流量区分开）
- 简单说一下KIP-103，kafka从0.9.0版本起引入listeners机制，可以配置多个listener，但有一个限制是同一个安全协议只能有一个listener，KIP-103通过增加`listener.security.protocol.map`解决这一问题，同时为了区分外部流量与内部流量到不同listener上，引入了`inter.broker.listener.name`来替代之前的`security.inter.broker.protocol`，同时KIP-103引入了小写前缀的机制，使得不同listener的配置（如SSL）可以单独设定

#### security.inter.broker.protocol

- broker之间通讯的安全协议
- 合法的值为`PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL`
- 同时定义`security.inter.broker.protocol`和`inter.broker.listener.name`是不正确的
- 0.9.0版本引入，与KIP-2（允许监听多个端口和IP）、KIP-12（Kafka Sasl/Kerberos 和 SSL 实现）有关

#### listener.security.protocol.map

- 监听器名称与协议的映射，当同一个安全协议有多于1个端口或IP时，必须设定此参数
- 例如，内部和外部都使用SSL协议，可以将他们区分开。举例来说，用户可以定义监听器名称为`INTERNAL` 和 `EXTERNAL`，形式为`INTERNAL:SSL,EXTERNAL:SSL`，key和value是通过`：`分隔，多个之间用逗号分隔，每个监听器的名字只应该出现一次
- 不同安全协议的监听器配置可以通过添加一个规范化的前缀来配置。比如为了配置`INTERNAL`的`keystore`为不同的值，可以设置`listener.name.internal.ssl.keystore.location`，如果没有监听器名称，则代表通用配置，如`ssl.keystore.location`（KIP-103引入了小写前缀的机制，使得不同listener的配置（如SSL）可以单独设定）
- 0.10.2版本引入，对应KIP-103（将内部外部流量区分开）

#### inter.broker.protocol.version

- 指定inter-broker协议的版本号，通常在所有broker升级到新的版本时用到
- 0.9.0版本引入，对应KIP-2（允许监听多个端口和IP），后续版本保持了这种机制（后续版本还有其他参数）。目的是从之前的版本平滑升级到0.9.0.0，升级过程如下

```
平滑升级的步骤
1.更新server.properties文件：添加inter.broker.protocol.version=0.8.2.X
2.更新代码，重启服务器
3.当所有服务器都升级版本后，修改inter.broker.protocol.version=0.9.0.0
4.挨个重启服务器，让新版本的协议生效
第2步执行完成后，第4步不必马上执行，可以随时执行
如果能够接受一段时间不可用，可以简单关闭所有服务，更新代码并重启，重启后就是新的协议
```

### 副本机制相关

#### replica.socket.receive.buffer.bytes

- 副本在网络请求时的socket接收缓冲区

#### replica.socket.timeout.ms

- 副本在网络请求时的socket超时时间，默认值为30s
- 这个值应该` >= replica.fetch.wait.max.ms`，否则等待太久会造成超时

#### replica.fetch.wait.max.ms

- follower副本发出的每个拉取请求最大的等待时间，默认值500ms
- 这个值应该`< replica.lag.time.max.ms`，以防止那些低吞吐量Topic的频繁的ISR收缩，因为等待太久会迟迟到不了LEO位置，引起ISR变动

#### replica.lag.time.max.ms

- 如果follower在此参数设定的时间内没有发送任何拉取请求，或者说没有消费到leader的`log end offset （LEO）`位置，leader将会把该follower从ISR中移除，默认值为30s

#### replica.fetch.min.bytes

- 每个拉取响应期望的最小字节数，如果没有足够的字节，将会等待至`replica.fetch.wait.max.ms`
- 默认值为1

#### replica.fetch.max.bytes

- 每个分区尝试拉取消息的最大字节数，这不是一个绝对的值，如果第一个非空分区的第一个消息批次比此值大，消息批次依然会返回以便能够正常处理
- 最大的消息批次大小是`[broker配置] message.max.bytes`、`[topic配置] max.message.bytes`决定的
- 默认值为1M

#### replica.fetch.response.max.bytes

- 与`replica.fetch.max.bytes`含义类似，不同之处在于这是整个拉取响应的最大字节数
- 默认值为10M
- 0.10.1版本引入，对应KIP-74（添加拉取请求响应的字节数限制）

#### replica.high.watermark.checkpoint.interval.ms

- 高水位数据被保存到磁盘的频率，默认值为5s

#### replica.fetch.backoff.ms

- 当拉取出现问题时，睡眠等待的时间，默认值为1s
- 0.9.0版本引入

#### replica.selector.class

- 实现了`ReplicaSelector`接口的全类名，被broker使用，来找到优先读取的副本
- 默认情况下，使用的是返回leader的实现
- 2.4版本引入，对应KIP-392（允许消费者从最近的副本拉取）

#### num.replica.fetchers

- 从一个源broker复制消息时的拉取线程数，增加此值可以增加follower broker的I/O并行度
- 默认值为1

### 控制器相关

#### controller.socket.timeout.ms

- controller与broker之间的socket超时时间

#### controlled.shutdown.enable

- 是否开启受控关闭

#### controlled.shutdown.max.retries

- 受控关闭由于各种原因可以失败的次数，当问题发生时，可以决定重试的次数

#### controlled.shutdown.retry.backoff.ms

- 每次受控关闭重试之前，系统需要时间来恢复造成之前失败的状态，这个参数决定了每次重试之前的等待时间

### 日志相关

#### log.dir

- 不推荐使用，见参数`log.dirs`
- 指定日志数据所在的目录,默认值为`/tmp/kafka-logs`

#### log.retention.minutes

- 不推荐使用，见参数`log.retention.hours`
- 日志文件在删除前保留的分钟数，参数`log.retention.ms`的第二补充
- 如果没有设置，将使用`log.retention.hours`

#### log.retention.ms

- 不推荐使用，见参数`log.retention.hours`
- 日志文件在删除前保留的毫秒数
- 如果没有设置，将使用`log.retention.minutes`
- 如果值为-1，则没有时间限制

#### log.flush.interval.messages

- 一个日志分区在累积到多少条消息后，会被刷写到磁盘中
- 默认值为`9223372036854775807`，可见kafka默认放弃了对刷盘的主动控制，由操作系统决定，应该检查操作系统的配置

#### log.flush.interval.ms

- 任何Topic消息从内存到刷写到磁盘的最大时间间隔
- 如果没有设置，则使用`log.flush.scheduler.interval.ms`的值，默认值是没有设置的

#### log.flush.scheduler.interval.ms

- log flusher检查是否需要将log写入磁盘的时间频率
- 默认值为`9223372036854775807`，可见kafka放弃了对刷盘的主动控制，由操作系统决定，应该检查操作系统的配置

#### log.flush.offset.checkpoint.interval.ms

- 更新上一次flush数据（即`log recovery point`）的频率，默认为1min。

#### log.flush.start.offset.checkpoint.interval.ms

- 更新`log start offset`的频率，默认为1min。

#### log.roll.hours

- 开始一个新的日志段的最大小时数，是`log.roll.ms`的补充，默认为168小时。

#### log.roll.ms

- 开始一个新的日志段的最大毫秒数
- 如果没有设置，则使用`log.roll.hours`的值，默认没有设置

#### log.roll.jitter.hours

- 从`logRollTimeMillis`中减去的最大抖动（小时），是`log.roll.jitter.ms`的补充

#### log.roll.jitter.ms

- 从`logRollTimeMillis`中减去的最大抖动
- 如果没有设置，则使用`log.roll.jitter.hours`的值，默认没有设置
- 如果有很多很多分区，参数`log.roll.ms`是全局的，因此同一时刻需要做很多文件的切分，磁盘IO顶不住，因此需要设置个`rollJitterMs`，来岔开它们，有点类似于redis缓存过期时间加个随机数，防止大量缓存同时过期。

#### log.segment.bytes

- 单个日志文件的最大大小，默认为1G

#### log.segment.delete.delay.ms

- 从文件系统中删除文件之前的最大等待时间，默认为1min

#### log.cleaner.backoff.ms

- 如果没有日志需要清理，睡眠等待时间

#### log.cleaner.dedupe.buffer.size

- 所有日志线程用于处理日志去重的总内存

#### log.cleaner.delete.retention.ms

- 已经删除的记录（墓碑消息，value为null的消息）的保留时间

#### log.cleaner.enable

- 是否开启日志清理程序，默认是开启的
- 当任何Topic，包括内部Topic，使用了`cleanup.policy=compact`时，都应该开启，如果不开启的话，那些Topic将不会被compact，会持续增长

#### log.cleaner.io.buffer.load.factor

- 日志清理器重复数据消除 缓冲区负载系数，重复数据消除缓冲区可能变为满的百分比
- 较高的值将允许一次清理更多日志，但会导致更多哈希冲突

#### log.cleaner.io.buffer.size

- 所有清理线程用于日志清理的IO缓冲区的总内存大小

#### log.cleaner.io.max.bytes.per.second

- 日志清理器会被限速，以便总的读写IO平均值小于此值

#### log.cleaner.max.compaction.lag.ms

- 消息在日志中不符合压缩条件的最长时间，仅适用于正在compact的日志，默认为`9223372036854775807`
- 2.3版本引入，对应KIP-354（添加最大日志压实延迟）

#### log.cleaner.min.compaction.lag.ms

- 消息在日志中保持未压缩状态的最短时间。仅适用于正在compact的日志，默认为0
- 0.10.1版本引入，对应KIP-58（使日志压实点可配置）

#### log.cleaner.min.cleanable.ratio

- 符合清理条件的日志的脏日志与总日志的最小比率

```
日志符合压缩的条件
达到脏率阈值log.cleaner.min.cleanable.ratio，并且日志有脏（未压缩）记录至少log.cleaner.min.compaction.lag.ms时间
或者
日志有脏（未压缩）记录至多log.cleaner.max.compaction.lag.ms时间
```

#### log.cleaner.threads

- 用于日志清理的后台线程数

#### log.cleanup.policy

- 日志清理策略，默认为delete
- 是一个用逗号分隔的列表，合法的值有`delete`  `compact`

#### log.index.interval.bytes

- offset索引项的添加间隔，默认值为4k。

#### log.index.size.max.bytes

- offset索引的最大字节数。默认为10M

#### log.message.format.version

- 指定代理将用于将消息附加到日志的消息格式版本。该值应为有效版本，如`0.8.2、0.9.0.0、0.10.0`
- 通过设置特定的消息格式版本，用户可以证明磁盘上的所有现有消息都小于或等于指定的版本
- 错误设置此值将导致使用旧版本的用户中断，因为他们将收到格式不清楚的消息
- 0.10.0.0版本引入，平滑升级到0.10.0.0文档是这么说的

```
1.更新server.properties文件
inter.broker.protocol.version=CURRENT_KAFKA_VERSION (e.g. 0.8.2 or 0.9.0.0)
log.message.format.version=CURRENT_KAFKA_VERSION（0.11.0.0起是CURRENT_MESSAGE_FORMAT_VERSION）
2.更新代码，重启服务器
3.当所有服务器都升级版本后，修改inter.broker.protocol.version=0.10.0.0
但保持log.message.format.version不变
4.挨个重启服务器，让新版本的协议生效
5.当所有的消费者升级到0.10.0之后，再把log.message.format.version=0.10.0
```

为什么0.10.0.0版本引入了此参数呢？

- 0.10.0中消息格式新增了时间戳，并且压缩的消息使用了相对偏移量
- `log.message.format.version`指定的是磁盘上消息的格式
- 消费者如果版本在0.10.0之前，那么它只能理解之前的消息格式，为了兼容，服务端可以把消息转换为之前的消息格式，但零拷贝会失效。消息格式转换使kafka CPU飙升，消费者需要立即升级才能恢复正常。所以为了避免这种情况，在消费者没有升级之前，先把`log.message.format.version`设置为之前的版本能避免转换。零拷贝能正常工作，直到消费者升级完成再改到最新的值。
- 所以消息转换能够兼容一些还没有升级的消费者，但要兼容所有消费者的情况是不明智的，所以kafka升级但消费者没有升级的时候，要尽量避免消息转换
- 通过设置消息格式版本，我们可以确定所有的消息要么是当前设置的版本，要么是之前的版本。否则，0.10.0之前的消费者可能不能工作
- 特别的是，当消息格式版本已经设置为0.10.0之后，不能再改回到之前的版本，因为那样可能会让0.10.0之前的消费者不能正常工作

#### log.message.downconversion.enable

- 此配置控制是否启用消息格式的下转换以满足消费请求。当设置为false时，broker不会为希望使用旧消息格式的消费者执行向下转换
- 对于来自此类旧客户端的消费请求，代理会以`UNSUPPORTED_VERSION`错误进行响应。
- 此配置不适用于复制到follower可能需要的任何消息格式转换。
- 2.0版本引入，对应KIP-283（向下转换的高效内存使用）

#### log.message.timestamp.difference.max.ms

- 代理收到消息时的时间戳与消息中指定的时间戳之间允许的最大差值
- 如果`log.message.timestamp.type=CreateTime`，如果时间戳的差异超过此阈值，则消息将被拒绝
- 如果`log.message.timestamp.type=LogAppendTime`，则忽略此配置
- 允许的最大时间戳差异不应大于`log.retention.ms`，以避免不必要的频繁日志滚动
- 0.10.0版本引入，对应KIP-32（为kafka的消息添加了时间戳）

#### log.message.timestamp.type

- 定义消息中的时间戳是消息创建时间还是日志附加时间
- 该值应为`CreateTime`或`LogAppendTime`
- 0.10.0版本引入，对应KIP-32（为kafka的消息添加了时间戳）

#### log.preallocate

- 创建新段时是否应预分配文件？如果在Windows上使用Kafka，可能需要将其设置为true
- 0.9.0版本引入，对应KIP-20（在windows和一些旧的Linux文件系统下，启用日志预分配以提高性能）

#### log.retention.check.interval.ms

- 日志清理器检查任何日志是否符合删除条件的频率（以毫秒为单位），默认为5min

### offset相关

#### offset.metadata.max.bytes

- 与offset提交相关的元数据项的最大大小

#### offsets.commit.required.acks

- 位移提交可被接受所需要的`acks`，通常默认值-1不应被修改

#### offsets.commit.timeout.ms

- offset提交将会被推迟直到所有副本接收到commit或者超时时间到达，与生产者请求的超时时间类似，默认为5s

#### offsets.load.buffer.size

- 在载入offset数据到缓存时，读取offset日志段的批次大小，软限制，如果记录太大会覆盖

#### offsets.retention.check.interval.ms

- 检查过期偏移的频率，默认为10min

#### offsets.retention.minutes

- 在消费者组失去所有消费者（即变为空）后，其offset将保留此保留期，然后再被丢弃
- 对于`standalone consumers`（使用手动分配），偏移量将在上次提交时间加上此保留期后过期
- 默认值为`10080`，可以适当调大，防止消费者长时间不消费后找不到之前的位移（根据业务需求）

#### offsets.topic.compression.codec

- 偏移量主题的压缩编解码器
- 压缩可用于实现“原子”提交
- 0.9.0版本引入

#### offsets.topic.num.partitions

- offset提交topic的分区数，部署之后不应该改变，默认为50

#### offsets.topic.replication.factor

- offset提交topic的复制因子，设置更高的值来保证可用性，默认为3
- 如果集群broker数量不满足复制因子的需求，内部主题创建将会失败

#### offsets.topic.segment.bytes

- offset提交topic的日志段的大小，应该被保留为相对较小的值，以便更快地进行日志压缩和缓存加载
- 默认值为100M，比普通的topic的1G要小

### 炼狱相关

#### producer.purgatory.purge.interval.requests

- 生产者请求炼狱的清除间隔（以请求数为单位）

#### delete.records.purgatory.purge.interval.requests

- 删除记录请求炼狱的清除间隔（以请求数为单位）
- 0.11.0版本引入

#### fetch.purgatory.purge.interval.requests

- 获取请求炼狱的清除间隔（以请求数为单位）

### 群组协调者相关

#### group.initial.rebalance.delay.ms

- 组协调员在执行第一次重新平衡之前等待更多消费者加入新组的时间
- 更长的延迟意味着可能更少的再平衡，但会增加处理开始之前的时间
- 默认为3s
- 0.11.0版本引入，对应KIP-134（推迟最初的消费者组再平衡），由于消费者组再平衡的代价很大，所以这是一个权衡，用延迟换取更少的重平衡次数。

#### group.max.session.timeout.ms

- 注册的消费者允许的最大会话超时
- 超时时间越长，消费者就有更多的时间处理心跳之间的消息，而检测失败的时间就越长
- 默认为30min
- 0.9.0版本引入

#### group.min.session.timeout.ms

- 注册的消费者允许的最小会话超时
- 超时时间越短，故障检测速度越快，但代价是消费者心跳频率越高，这可能会压倒服务端的资源
- 默认为6s
- 0.9.0版本引入

#### group.max.size

- 单个消费者组可以容纳的最大消费者数量
- 默认为2147483647
- 2.2版本引入，对应KIP-389（引入一个可配置的消费者组大小限制）

### 限速相关

#### ~~quota.consumer.default~~

- 不推荐：仅在未配置动态默认配额时使用，kafka 3.0版本起彻底废弃
- 如果任何按`clientId/consumer group`区分的使用者每秒获取的字节数超过此值，则该使用者将被限制
- 0.9.0 版本引入，对应KIP-13（限速），KIP-55（身份认证用户的安全限速）开始就不建议用了

#### ~~quota.producer.default~~

- 不推荐：仅在未配置动态默认配额时使用，kafka 3.0版本起彻底废弃
- 任何以`clientId`区分的生产者如果每秒产生的字节数超过此值，将被限制
- 0.9.0 版本引入，对应KIP-13（限速），KIP-55（身份认证用户的安全限速）开始就不建议用了

#### quota.window.num

- 保留在内存中的样本数`for client quotas`
- 0.9.0 版本引入，对应KIP-13（限速）

#### replication.quota.window.num

- 保留在内存中的样本数`for replication quotas`
- 0.10.1版本引入，对应KIP-73（复制限速）

#### controller.quota.window.num

- 保留在内存中的样本数`for controller mutation quotas`
- 2.7版本引入，对应KIP-599（创建Topic、创建分区、删除Topic等操作限速）

#### alter.log.dirs.replication.quota.window.num

- 保留在内存中的样本数`for alter log dirs replication quotas`
- 1.1.0版本引入

#### quota.window.size.seconds

- 每个样本的时间跨度`for client quotas`
- 0.9.0 版本引入，对应KIP-13（限速）

#### replication.quota.window.size.seconds

- 每个样本的时间跨度`for replication quotas`
- 0.10.1版本引入，对应KIP-73（复制限速）

#### controller.quota.window.size.seconds

- 每个样本的时间跨度`for controller mutation quotas`
- 2.7版本引入，对应KIP-599（创建Topic、创建分区、删除Topic等操作限速）

#### alter.log.dirs.replication.quota.window.size.seconds

- 每个样本的时间跨度`for alter log dirs replication quotas`
- 1.1.0版本引入

#### client.quota.callback.class

- 实现`ClientQuotaCallback接口`的类的完全限定名，用于确定应用于客户端请求的配额限制
- 默认情况下，或应用ZooKeeper中存储的配额。对于任何给定的请求，将应用与会话的用户主体和请求的客户端id匹配的最特定配额
- 2.0版本引入，对应KIP-257（可配置的限速管理）

### 事务相关

以下事务相关参数均在0.11.0版本引入，对应KIP-98（仅一次投递和事务消息）

#### transaction.max.timeout.ms

- 事务允许的最大超时，默认值为900000
- 如果客户机请求的事务时间超过此时间，则代理将在`initProducerdRequest`中返回一个错误。这可以防止客户端超时过大，这可能会使消费者无法阅读事务中包含的主题

#### transaction.state.log.load.buffer.size

- 将生产者ID和事务加载到缓存时，从事务日志段读取的批大小（软限制，如果记录太大，则覆盖）

#### transaction.state.log.min.isr

- 为事务Topic重写`min.insync.replicas`，默认值为2

#### transaction.state.log.num.partitions

- 事务主题的分区数（部署后不应更改），默认为50

#### transaction.state.log.replication.factor

- 事务主题的复制系数（设置得更高以确保可用性），默认值为3
- 如果集群broker数量不满足复制因子的需求，内部主题创建将会失败

#### transaction.state.log.segment.bytes

- 事务主题段字节应保持相对较小，以便加快日志压缩和缓存加载

#### transaction.abort.timed.out.transaction.cleanup.interval.ms

- 回滚已超时事务的时间间隔，默认值为10s

#### transaction.remove.expired.transaction.cleanup.interval.ms

- 删除过期事务（因超过`transactional.id.expiration.ms`）的间隔，默认值为3600s

#### transactional.id.expiration.ms

- 事务协调器在事务id过期之前，在没有收到当前事务的任何事务状态更新的情况下等待的时间（以毫秒为单位）
- 此设置还影响生产者id过期: 在使用给定生产者id进行最后一次写入之后，生产者id将过期。如果由于主题的保留设置而删除了生产者id的最后一次写入，则id可能会提前过期。

### 授权令牌相关

如无特殊说明，大部分参数为1.1版本引入，对应KIP-48（授权令牌支持）

Kafka 0.9.0中引入了安全支持，用kerberos作为身份验证层，kafka通常会有大量的生产者和消费者，所以在安全环境中所有客户端都需要获得keytab或者TGT来确保他们可以与安全的kafka broker通讯，这有一些缺点：

- KDC性能问题/压力，因为每个客户端都需要前往KDC来获取ticket
- 续约需要经过KDC，而续约过的TGT需要重新分配给所有客户端
- 如果TGT受到威胁，那么辐射范围会很广，因为TGT可能会给Kafka之外的应用服务
- 仅仅与kerberos身份验证方案兼容
- 管理成本，对任何新客户端来说，必须获取keytab或者以某种方式从其他节点获得TGT

为了解决上述问题，建立了对授权令牌的支持，以保护kafka。授权令牌时kafka服务器与客户端之间共享的秘密，因此无需通过KDC即可进行身份验证。

授权令牌将有助于处理框架在安全环境中将工作负载分配给可用的worker，而不需要增加分发keytabs或TGT的成本。在Storm中，storm的master是唯一需要keytab的节点，通过这个keytab，这个master节点将与kafka进行身份认证，并获得授权令牌，之后master节点可以向他管理的所有worker分发这个授权令牌，所有的客户端就都可以使用此令牌与kafka取得身份认证，从而可以访问master节点对应的keytab 主体。

授权令牌的授权流程

- Admin/DelegationToken客户端连接kafka，并通过安全channel的身份认证
- 认证成功后，Admin/DelegationToken客户端向kafka申请授权令牌
- 服务端生成令牌，在本地内存、zookeeper保存（让其他broker也知道）并返回给客户端
- 客户端进行令牌分发给其他客户端
- 其他客户端通过复用SASL channel的方式进行基于令牌的认证，其中SCRAM是合适的（KIP-84支持SASL SCRAM机制）
- 服务端进行token查找，如果找到令牌并且没有过期，则认证通过，否则会抛出异常

授权令牌的刷新

- Admin/DelegationToken客户端可以，而通过令牌认证的客户端不能刷新令牌
- 客户端发送请求，可以附加一个生存时间，来进行续订
- 如果令牌已经过期，那么会有异常

#### delegation.token.expiry.time.ms

- 令牌需要续订之前的令牌有效时间（以毫秒为单位）

#### delegation.token.master.key

- `delegation.token.secret.key`的别名，不推荐

#### delegation.token.max.lifetime.ms

- 令牌有一个最长的生存期，超过该生存期，将无法再续订

#### delegation.token.expiry.check.interval.ms

- 扫描时间间隔以删除过期的委派令牌

#### delegation.token.secret.key

- 用于生成和验证委派令牌的密钥
- 必须在所有代理中配置相同的密钥
- 如果密钥未设置或设置为空字符串，代理将禁用授权令牌支持
- 2.8版本引入，对应KIP-681（在委派令牌功能中重新命名master key）

### 其他连接/socket相关

#### connections.max.idle.ms

- 在此配置指定的毫秒数后关闭空闲连接

#### connections.max.reauth.ms

- 当显式设置为正数（默认值为0，不是正数）时，进行身份验证时，不超过配置值的会话生存期`session lifetime`将被传递给2.2.0版本及以后版本的客户端
- 代理将断开在会话生存期内未重新验证的任何此类连接，然后将其用于除重新验证以外的任何目的。
- 配置名称可以选择以侦听器前缀和SASL机制名称（小写）作为前缀。如`listener.name.sasl_ssl.oauthbearer.connections.max.reauth.ms=3600000`
- 2.2版本引入，对应KIP-368（允许SASL连接间断性的重新认证）

#### max.connection.creation.rate

- 代理中随时允许的最大连接创建速率
- 侦听器级别限制也可以通过在配置名称前面加上侦听器前缀来配置，例如`listener.name.internal.max.connection.creation.rate`
- 代理范围内的连接速率限制应根据代理容量进行配置，而listener限制应根据应用程序要求进行配置
- 如果达到侦听器或代理限制，新连接将被限制，但代理间侦听器除外。只有在达到侦听器级别的速率限制时，才会限制代理间侦听器上的连接
- 2.7版本引入，对应KIP-612（能够限制broker上的连接创建速率）

#### max.connections

- 代理中随时允许的最大连接数
- 在`max.connections.per.ip`的限制之上还要应用此限制
- 侦听器级别限制也可以通过在配置名称前面加上侦听器前缀来配置。例如`listener.name.internal.max.connections`
- 代理范围的限制应根据代理容量进行配置，而侦听器限制应根据应用程序要求进行配置
- 如果达到侦听器或代理限制，新连接将被阻止。即使达到了代理范围的限制，也允许在代理间侦听器上进行连接。在这种情况下，另一个侦听器上最近使用最少的连接将被关闭
- 2.3版本引入，对应KIP-402（提高SocketServer处理器的公平性）

#### max.connections.per.ip

- 每个ip地址允许的最大连接数
- 如果使用`max.connections.per.ip.overrides`属性配置了覆盖，则可以将其设置为0。如果达到限制，则会断开来自ip地址的新连接

#### max.connections.per.ip.overrides

- 以逗号分隔的每ip或主机名覆盖列表，默认最大连接数
- 示例值为`主机名：100 127.0.0.1:200`

#### max.incremental.fetch.session.cache.slots

- 维护的最大增量获取会话数。
- 1.1版本引入，对应KIP-227（引入增量请求以提高分区可伸缩性）

#### connection.failed.authentication.delay.ms

- 身份验证失败时连接关闭延迟：这是身份验证失败时连接关闭延迟的时间（毫秒）
- 必须将其配置为少于`connections.max.idle.ms`来防止连接超时
- 2.1版本引入，对应KIP-306（对于认证失败客户端的延迟响应配置）

#### socket.connection.setup.timeout.max.ms

- 客户端等待socket连接建立的最长时间
- 每次连续连接失败时，连接设置超时将以指数形式增加，直至达到此最大值
- 为了避免连接风暴，将对超时应用0.2的随机化因子，导致计算值的随机范围在20%以下和20%以上
- 2.7版本引入，对应KIP-601（NetworkClient的socket连接超时时间可配置）

#### socket.connection.setup.timeout.ms

- 客户端等待socket连接建立的时间
- 如果在超时时间过去之前未建立连接，客户端将关闭socket通道
- 2.7版本引入，对应KIP-601（NetworkClient的socket连接超时时间可配置）

#### socket.receive.buffer.bytes

- 服务器socket的`SO_RCVBUF`缓冲区。如果值为-1，将使用操作系统默认值。

#### socket.request.max.bytes

- socket请求中的最大字节数

#### socket.send.buffer.bytes

- 服务器socket的`SO_SNDBUF`缓冲区。如果值为-1，将使用操作系统默认值。

### 监控指标相关

#### kafka.metrics.polling.interval.secs

- 可以在`kafka.metrics.reporters`实现中使用的度量轮询间隔（以秒为单位）
- 2.1版本引入

#### kafka.metrics.reporters

- 要用作`Yammer指标`的类的列表
- reporters应该实现`kafka.metrics.KafkaMetricsReporter`，如果客户机希望在定制报告器上公开JMX操作，那么定制报告器还需要实现MBean特性`extends kafka.metrics.KafkaMetricsReporterMBean trait`，以便注册的MBean符合标准MBean约定
- 2.1版本引入

#### metric.reporters

- 用作度量报告器的类的列表
- 实现`org.apache.kafka.common.metrics.MetricsReporter`接口的类会被插入到类列表，新指标创建时可被通知
- JmxReporter始终包含在注册JMX统计数据中
- 0.9.0版本引入，在0.8.2的生产者中已有此参数，0.9.0版本服务端首次引入此参数

#### metrics.num.samples

- 为计算指标而维护的样本数。
- 0.9.0版本引入，在0.8.2的生产者中已有此参数，0.9.0版本服务端首次引入此参数

#### metrics.recording.level

- 指标的最高纪录级别。
- 0.10.2版本引入，对应KIP-105（增加传感器的记录级别）

#### metrics.sample.window.ms

- 计算度量样本的时间窗口。
- 0.9.0版本引入，在0.8.2的生产者中已有此参数，0.9.0版本服务端首次引入此参数

### KRaft相关

以下参数均为3.0版本引入，如无特殊说明，均对应KIP-631（基于仲裁的kafka控制器）

#### node.id

- 当`process.roles`不为空时，`node.id`与程序扮演的角色相关连，在KRaft下运行时，这是必需的配置。

#### process.roles

- 程序所扮演的角色是，`broker`, `controller` 或者` broker,controller`
- 此配置仅适用于KRaft（Kafka Raft）模式下的集群（而不是ZooKeeper）。对于Zookeeper群集，请保留此配置未定义或为空。

#### controller.listener.names

- 控制器使用的，用逗号分隔的监听器名称列表
- 在KRaft模式下运行时需要。基于zk的控制器不会使用此配置

#### metadata.log.dir

- 这个配置决定了我们在KRaft模式下把集群的元数据日志放在哪里
- 如果未设置，元数据日志将被放置在`log.dirs`的第一个日志目录

#### metadata.log.segment.bytes

- 单个元数据日志文件的最大大小。

#### metadata.log.segment.ms

- 创建新元数据日志文件之前的最长时间（以毫秒为单位）

#### metadata.max.retention.bytes

- 删除旧快照和日志文件之前元数据日志和快照的最大组合大小
- 由于在删除任何日志之前必须至少存在一个快照，因此这是一个软限制

#### metadata.max.retention.ms

- 删除元数据日志文件或快照之前保留它的毫秒数。由于在删除任何日志之前必须至少存在一个快照，因此这是一个软限制

#### controller.quorum.election.backoff.max.ms

- 开始新选举前的最长时间（毫秒）。这在二进制指数退避机制中使用，有助于防止选举陷入僵局

#### controller.quorum.election.timeout.ms

- 在触发新的选举之前，在无法从领导人那里获取信息的情况下等待的最长时间（以毫秒为单位）

#### controller.quorum.fetch.timeout.ms

- 在成为候选人并引发选民选举之前，没有成功从现任领导人手中接手的最长时间；在询问是否有一个新的领导者时代之前，在没有收到大多数法定人数回执的情况下的最长时间

#### controller.quorum.voters

- 投票人集合的id/端点信息映射，以逗号分隔的`{id}@{host}:{port}`条目列表。
- 例如：`1@localhost:9092,2@localhost:9093,3@localhost:9094`

#### controller.quorum.append.linger.ms

- 在将写入刷新到磁盘之前，leader等待写入累积的持续时间（以毫秒为单位）

#### controller.quorum.request.timeout.ms

- 配置控制客户端等待请求响应的最长时间。
- 如果在超时时间过去之前未收到响应，则客户端将在必要时重新发送请求，或者在重试次数用尽时使请求失败。
- 0.9.0版本引入，对应KIP-19（为NetworkClient添加请求超时）

#### controller.quorum.retry.backoff.ms

- 尝试重试对给定主题分区的失败请求之前等待的时间。
- 这避免了在某些故障情况下在紧密循环中重复发送请求。

#### broker.heartbeat.interval.ms

- 代理心跳之间的时间长度（以毫秒为单位）。在KRaft模式下运行时使用。

#### broker.session.timeout.ms

- 如果没有心跳，经纪人租约持续的时间长度（毫秒）。在KRaft模式下运行时使用。

#### initial.broker.registration.timeout.ms

- 最初向控制器仲裁注册时，声明失败并退出代理进程之前等待的毫秒数。

#### metadata.log.max.record.bytes.between.snapshots

- 这是在生成新快照之前，日志中最新快照和高水位线之间所需的最大字节数
- 此参数对应KIP-630（Kafka Raft 快照）

#### sasl.mechanism.controller.protocol

-  用于与控制器通信的SASL机制。默认为`GSSAPI`。
-  3.0 版本引入，对应KIP-631

### 安全机制相关

#### security.providers

- 可配置的creator类列表，每个类返回一个实现安全算法的提供者。类需实现`org.apache.kafka.common.security.auth.SecurityProviderCreator接口`。
- 2.4版本引入

#### authorizer.class.name

- 实现`org.apache.kafka.server.authorizer.Authorizer接口`的类的完全限定名，代理使用该接口进行授权。
- 0.9.0版本引入，对应KIP-11（授权接口）

#### principal.builder.class

- 实现`KafkaPrincipalBuilder接口`的类的完全限定名，用于构建授权期间使用的`KafkaPrincipal`对象。
- 如果未定义主体生成器，则默认行为取决于使用的安全协议。
- 对于SSL身份验证，主体将使用`ssl.principal.mapping.rules`定义的规则派生，这些规则应用于客户端证书中的可分辨名称（如果提供）
- 对于SASL身份验证，如果GSSAPI正在使用，主体将使用`sasl.kerberos.principal.to.local.rules`定义的规则派生，对于其他机制，主体将使用SASL身份验证ID派生
- 对于明文，主体将是`ANONYMOUS`
- 0.9.0版本引入，与KIP-12（Kafka Sasl/Kerberos 和 SSL 实现）有关

#### password.encoder.cipher.algorithm

- 用于对动态配置的密码进行编码的密码算法。

#### password.encoder.iterations

- 用于对动态配置的密码进行编码的迭代计数。

#### password.encoder.key.length

- 用于对动态配置的密码进行编码的密钥长度。

#### password.encoder.keyfactory.algorithm

- 用于对动态配置的密码进行编码的`SecretKeyFactory`算法。如果可用，默认值为`PBKDF2WithHmacSHA512`，否则默认值为`PBKDF2WithHmacSHA1`。

#### password.encoder.old.secret

- 用于对动态配置的密码进行编码的旧密码。只有在更新机密时才需要此选项。如果指定，所有动态编码的密码将使用此旧密码解码，并使用密码重新编码。编码器。经纪人启动时的秘密。

#### password.encoder.secret

- 用于对此代理动态配置的密码进行编码的密码。

以上`password.*`相关参数为1.1版本引入，对应KIP-226（动态broker配置），主要目的是为了保护动态配置在zookeeper中的密码，对密码进行加密，防止原始密码被获取

### SSL相关参数

#### 0.9.0版本引入，对应KIP-12（Kafka Sasl/Kerberos 和 SSL 实现）

##### ssl.cipher.suites

- 密码套件列表。这是身份验证、加密、MAC和密钥交换算法的命名组合，用于协商使用TLS或SSL网络协议的网络连接的安全设置。
- 默认情况下，支持所有可用的密码套件。

##### ssl.client.auth 

- 是否使用SSL进行客户端身份认证，默认是none
- `ssl.client.auth=required` 如果设置为required，则需要客户端身份验证。
- `ssl.client.auth=requested` 这意味着客户端身份验证是可选的。与“必需”不同，如果设置了此选项，客户端可以选择不提供有关自身的身份验证信息
- `ssl.client.auth=none` 这意味着不需要客户端身份验证。

##### ssl.enabled.protocols

- 为SSL连接启用的协议列表。
- 使用Java 11或更高版本运行时，默认值为`TLSv1.2,TLSv1.3`，否则为`TLSv1.2`。
- 使用Java 11的默认值，客户端和服务器将更喜欢 `TLSv1.3`，如果双方都支持的话，否则退回到`TLSv1.2`否则（假设两者都至少支持`TLSv1.2`）
- 在大多数情况下，这种默认设置应该是可以接受的。另请参阅`ssl.protocol`

##### ssl.key.password

- 密钥存储文件中私钥的密码或`ssl.keystore.key`中指定的PEM密钥。
- 仅当配置了双向身份验证时，客户端才需要这样做。

##### ssl.keymanager.algorithm

- 密钥管理器工厂用于SSL连接的算法。
- 默认值是为Java虚拟机配置的密钥管理器工厂算法（SunX509）。

##### ssl.keystore.location

- 密钥存储文件的位置。
- 这对于客户端是可选的，可以用于客户端的双向身份验证。

##### ssl.keystore.password

- 密钥存储文件的存储密码。
- 这对于客户端是可选的，只有在配置了`ssl.keystore.location`时才需要。PEM格式不支持密钥存储密码。

##### ssl.keystore.type

- 密钥存储文件的文件格式。这对于客户端是可选的。
- 默认是JKS

##### ssl.protocol

- 用于生成`SSLContext`的SSL协议。当使用Java 11或更新版本运行时，默认值为`TLSv1.3`，否则为`TLSv1.2`。
- 对于大多数用例，这个值应该是合适的。最近的JVM中允许的值是`TLSv1.2` 和 `TLSv1.3`。
- 旧版JVM可能支持`TLS`, `TLSv1.1`, `SSL`, `SSLv2` , `SSLv3`，但由于已知的安全漏洞，不鼓励使用它们。
- 使用此配置的默认值和`ssl.enabled.protocols`，如果服务器不支持`TLSv1.3`，客户端将降级为`TLSv1.2`。
- 如果此配置设置为`TLSv1.2`，则即使`TLSv1.3`是`ssl.enabled.protocols`中的值之一，并且仅服务器支持`TLSv1.3`，客户端也不会使用`TLSv1.3`。

##### ssl.provider

- 用于SSL连接的安全提供程序的名称。
- 默认值是JVM的默认安全提供程序。

##### ssl.trustmanager.algorithm

- 信任管理器工厂用于SSL连接的算法。
- 默认值是为Java虚拟机配置的`trust manager`工厂算法（PKIX）。

##### ssl.truststore.location

- 信任存储文件的位置。

##### ssl.truststore.password

- 信任存储文件的密码。如果未设置密码，则仍将使用已配置的信任存储文件，但将禁用完整性检查。PEM格式不支持信任存储密码。

##### ssl.truststore.type

- 信任存储文件的文件格式。
- 默认为JKS

##### ssl.endpoint.identification.algorithm

- 使用服务器证书验证服务器主机名的端点识别算法。

#### 0.10.1版本引入

##### ssl.secure.random.implementation

- 用于SSL加密操作的`SecureRandom PRNG`实现

#### 2.2版本引入，对应KIP-371（添加一个配置来构建自定义的SSL主体名称）

##### ssl.principal.mapping.rules

- 用于从客户端证书的可分辨名称映射到短名称的规则列表。规则按顺序求值，并使用与主体名称匹配的第一条规则将其映射到短名称。列表中任何后面的规则都将被忽略。默认情况下，X.500证书的可分辨名称将是主体。有关格式的更多详细信息，请参阅安全授权和ACL
- 如果`principal.builder.class`配置提供了`KafkaPrincipalBuilder`的扩展，则会忽略此配置。

#### 2.6版本引入，对应KIP-519（使SSL上下文/引擎配置可扩展）

##### ssl.engine.factory.class 

- 用于提供`SSLEngine对象`的`org.apache.kafka.common.security.auth.SslEngineFactory 类`。
- 默认值为`org.apache.kafka.common.security.ssl.DefaultSslEngineFactory`

#### 2.7版本引入，对应KIP-651（支持PEM格式的SSL证书和私钥）

##### ssl.keystore.certificate.chain

- `ssl.keystore.type`指定格式的证书链。
- 默认的SSL引擎工厂只支持PEM格式和X.509证书列表

##### ssl.keystore.key

- `ssl.keystore.type`指定格式的私钥。
- 默认SSL引擎工厂仅支持带有`PKCS#8`密钥的PEM格式。如果密钥已加密，则必须使用`ssl.key.password`指定密钥密码

##### ssl.truststore.certificates

- `ssl.truststore.type`指定格式的受信任证书。
- 默认SSL引擎工厂仅支持带有X.509证书的PEM格式。

### SASL相关参数

#### 0.9.0版本引入，对应KIP-12（Kafka Sasl/Kerberos 和 SSL 实现）

##### sasl.kerberos.kinit.cmd

- Kerberos kinit命令路径。

##### sasl.kerberos.min.time.before.relogin

- 刷新尝试之间的登录线程睡眠时间。

##### sasl.kerberos.principal.to.local.rules 

- 从主体名称映射到短名称（通常是操作系统用户名）的规则列表。
- 规则按顺序求值，并使用与主体名称匹配的第一条规则将其映射到短名称。列表中任何后面的规则都将被忽略。
- 默认情况下，`{username}/{hostname}@{REALM}`形式的主体名称映射到`{username}`。

- 有关格式的更多详细信息，请参阅安全授权和ACL。
- 请注意，如果`principal.builder.class`配置提供了`KafkaPrincipalBuilder`的扩展，则会忽略此配置。

##### sasl.kerberos.service.name

- kafka运行的`Kerberos`主体名称。
- 这可以在kafka的JAAS配置或kafka的配置中定义。

##### sasl.kerberos.ticket.renew.jitter

- 添加到续订时间的随机抖动百分比。

##### sasl.kerberos.ticket.renew.window.factor

- 登录线程将一直休眠，直到达到从上次刷新到票证到期的指定时间窗口系数，此时它将尝试续订票证。

#### 0.10.0版本引入，对应KIP-43（Kafka SASL 增强）

##### sasl.enabled.mechanisms

- Kafka服务器中启用的SASL机制列表。该列表可能包含安全提供程序可用的任何机制。
- 默认情况下，只有`GSSAPI`处于启用状态。

##### sasl.mechanism.inter.broker.protocol

- 用于代理间通信的SASL机制。默认为`GSSAPI`。

#### 2.0版本引入，对应KIP-86（可配置的SASL回调处理器）

##### sasl.client.callback.handler.class

- 实现`AuthenticateCallbackHandler接口`的SASL客户端回调处理程序类的完全限定名

##### sasl.login.callback.handler.class

- 实现AuthenticateCallbackHandler接口的SASL登录回调处理程序类的完全限定名。
- 对于代理，登录回调处理程序配置必须以侦听器前缀和SASL机制名称（小写）作为前缀。例如`listener.name.sasl_ssl.scram-sha-256.sasl.login.callback.handler.class=com.example.CustomScramLoginCallbackHandler`

##### sasl.server.callback.handler.class

- 实现AuthenticateCallbackHandler接口的SASL服务器回调处理程序类的完全限定名。
- 服务器回调处理程序必须以侦听器前缀和SASL机制名称（小写）作为前缀。例如，`listener.name.sasl_ssl.plain.sasl.server.callback.handler.class=com.example.CustomPlainCallback`

##### sasl.login.class 

- 实现登录接口的类的完全限定名。
- 对于代理，登录配置必须以侦听器前缀和SASL机制名称（小写）作为前缀。例如，`listener.name.sasl_ssl.scram-sha-256.sasl.login.class=com.example.CustomScramLogin`

#### 2.0版本引入，对应KIP-255（OAuth Authentication via SASL/OAUTHBEARER）

##### sasl.login.refresh.buffer.seconds

- 刷新凭证时，凭证过期前要保留的缓冲时间量，以秒为单位。
- 如果刷新时间比缓冲区秒数更接近到期时间，则刷新时间将向上移动，以尽可能多地保持缓冲区时间。法定值在0到3600（1小时）之间；如果未指定值，则使用默认值300（5分钟）。
- 这个值和`sasl.login.refresh.min.period.seconds`的总和超过凭证的剩余生存期，则秒数和秒数都将被忽略。目前只适用于`OAUTHBEARER`。

##### sasl.login.refresh.min.period.seconds

- 登录刷新线程在刷新凭据之前所需的最短等待时间（秒）。法定值在0到900（15分钟）之间；如果未指定值，则使用默认值60（1分钟）。
- 这个值和`sasl.login.refresh.buffer.seconds`的总和超过凭证的剩余生存期，则秒数和秒数都将被忽略。目前只适用于`OAUTHBEARER`。

##### sasl.login.refresh.window.factor

- 登录刷新线程将休眠，直到达到与凭据生命周期相关的指定窗口系数，此时它将尝试刷新凭据。
- 合理值介于0.5（50%）和1.0（100%）之间；如果未指定值，则使用默认值0.8（80%）。目前只适用于`OAUTHBEARER`。

##### sasl.login.refresh.window.jitter

- 添加到登录刷新线程睡眠时间中的相对于凭据生命周期的最大随机抖动量。
- 合理值介于0和0.25（25%）之间；如果未指定值，则使用默认值0.05（5%）。目前只适用于`OAUTHBEARER`。

#### 1.1版本引入，在此之前服务端没有此配置，但客户端有

##### sasl.jaas.config

- SASL连接的JAAS登录上下文参数，格式由JAAS配置文件使用。这里描述了JAAS配置文件格式。值的格式为：`loginModuleClass controlFlag (optionName=optionValue)*;`
- 对于代理，配置必须以侦听器前缀和SASL机制名称（小写）作为前缀。例如,`listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=com.example.ScramLoginModule required;`

#### 3.0 版本引入，对应KIP-631

##### sasl.mechanism.controller.protocol

- 用于与控制器通信的SASL机制。默认为`GSSAPI`。

### zookeeper TLS相关

这部分的参数是在2.5.0版本引入，对应KIP-515（使zk客户端使用新的TLS支持的认证），主要目的是为了支持zookeeper 3.5.7版本以来的TLS，通过系统属性可以进行配置，但不够安全，所以新增了这些参数，从而使broker以及命令行都可以以TLS的方式连接到zookeeper。

#### zookeeper.ssl.client.enable

- 与zookeeper连接时使用TLS，默认为false，覆盖`zookeeper.client.secure`系统属性，如果两者均未设置，则默认不使用TLS
- 如果为true，则必须设置`zookeeper.clientCnxnSocket`，其余的`zookeeper.ssl`开头的参数也需要设置

#### zookeeper.clientCnxnSocket 

- 当使用TLS连接ZooKeeper时，通常设置为`org.apache.zookeeper.ClientCnxnSocketNetty`

- 覆盖`zookeeper.clientCnxnSocket`系统属性

#### zookeeper.ssl.cipher.suites

- 指定要在ZooKeeper TLS协商中使用的已启用密码套件
- 覆盖`zookeeper.ssl.ciphersuites`系统属性
- 默认值null表示启用的密码套件列表由所使用的Java运行时确定。

#### zookeeper.ssl.crl.enable

- 指定是否在ZooKeeper TLS协议中启用证书吊销列表`Certificate Revocation List`
- 覆盖`zookeeper.ssl.crl`系统属性。

#### zookeeper.ssl.enabled.protocols 

- 指定ZooKeeper TLS协商（csv）中启用的协议
- 覆盖`zookeeper.ssl.enabledProtocols`系统属性
- 默认值null表示启用的协议将是`zookeeper.ssl.protocol`配置属性的值。

#### zookeeper.ssl.endpoint.identification.algorithm

- 指定是否在ZooKeeper TLS协商过程中启用主机名验证`hostname verification`
- 其中`https`表示启用ZooKeeper主机名验证，显式空白值表示禁用（建议仅出于测试目的禁用）
- 覆盖`zookeeper.ssl.hostnameVerification`系统属性（true表示https，false表示空白）

#### zookeeper.ssl.ocsp.enable

- 指定是否在ZooKeeper TLS协议中启用联机证书状态协议`Online Certificate Status Protocol`
- 覆盖`zookeeper.ssl.ocsp`系统属性

#### zookeeper.ssl.protocol 

- 指定ZooKeeper TLS协商中使用的协议
- 覆盖`zookeeper.ssl.protocol`系统属性

#### zookeeper.ssl.keystore.location 

- 当客户端使用TLS连接zookeeper时，客户端侧证书的密钥库位置
- 覆盖`zookeeper.ssl.keyStore.location`系统属性

#### zookeeper.ssl.keystore.password 

- 当客户端使用TLS连接zookeeper时，客户端侧证书的密钥库密码
- 覆盖`zookeeper.ssl.keyStore.password`系统属性
- 注意：zookeeper中`key password`必须与`keystore password`一致，否则连接会失败

#### zookeeper.ssl.keystore.type 

- 当客户端使用TLS连接zookeeper时，客户端侧证书的密钥库类型
- 覆盖`zookeeper.ssl.keyStore.type`系统属性
- 默认值null表示将根据密钥库的文件扩展名自动检测该类型。

#### zookeeper.ssl.truststore.location 

- 使用与ZooKeeper的TLS连接时信任存储`Truststore`位置
- 覆盖`zookeeper.ssl.trustStore.location`系统属性

#### zookeeper.ssl.truststore.password 

- 使用与ZooKeeper的TLS连接时使用Truststore密码
- 覆盖`zookeeper.ssl.trustStore.password`系统属性

#### zookeeper.ssl.truststore.type

- 使用与ZooKeeper的TLS连接时的信任库类型
- 覆盖`zookeeper.ssl.trustStore.type`系统属性
- 默认值null表示将根据信任库的文件扩展名自动检测该类型

### 总结与说明

由于kafka安全方面的配置比较多，本文并未对SASL、SSL等安全配置做展开说明

核心参数、重要参数部分是重点需要关注的，而其他参数是在某些场景下需要配置，有些则可以保持默认值

服务端参数变化也可以反映kafka的发展历程

- 0.9.0 版本的多listener、压缩、限速、sasl、ssl、metric等
- 0.10.0版本的消息时间戳
- 0.11.0版本的事务支持
- 1.1版本的动态配置、授权令牌、更改日志目录等
- 2.0版本的sasl自定义增强、限速自定义增强等
- 2.4版本的支持从follower读取数据
- 2.5版本对zookeeper安全的支持
- 3.0版本的KRaft

也可以发现因kafka用途越来越广泛，一系列的接入方式、连接管控、安全性、性能等方面的改进。

