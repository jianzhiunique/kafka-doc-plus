# SASL配置

此文使用SASL的SCRAM-SHA-512机制，该机制下的用户名密码都存在zookeeper中，方便其他模块如ACL、Quota等使用

其他机制配置都相似

## 修改Kafka配置文件以启用SASL

### 不同时启用ACL的情况下，该配置是可以的
```
# 在不同时启用ACL的情况下，可以让节点的内部通信使用纯文本PLAINTEXT，而不使用SASL、SSL等
# inter.broker.listener.name保持为null，仅设置security.inter.broker.protocol
security.inter.broker.protocol=PLAINTEXT
# 此处的internal是监听器的名称，在多个listener启用SASL时，
# 使用listener.name.监听器名称小写.SASL授权机制.sasl.jaas.config来指定jaas配置
listener.name.internal.scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required;
sasl.enabled.mechanisms=SCRAM-SHA-512
```

### 同时启用ACL的情况下，上述配置可能无法正常运行

如果同时启用了ACL机制，ACL配置部分allow.everyone.if.no.acl.found=false，会发现服务端日志中有报错
```
[2020-10-30 19:14:13,464] ERROR [KafkaApi-0] Error when handling request: clientId=0, correlationId=0, api=UPDATE_METADATA, version=6, body={controller_id=0,controller_epoch=15,broker_epoch=4294968154,topic_states=[],live_brokers=[{id=0,endpoints=[{port=9092,host=x.x.x.x,listener=INTERNAL,security_protocol=3,_tagged_fields={}},{port=9093,host=x.x.x.x,listener=PLAINTEXT,security_protocol=0,_tagged_fields={}}],rack=null,_tagged_fields={}}],_tagged_fields={}} (kafka.server.KafkaApis)
org.apache.kafka.common.errors.ClusterAuthorizationException: Request Request(processor=3, connectionId=x.x.x.x:9093-x.x.x.x:43898-0, session=Session(User:ANONYMOUS,/x.x.x.x), listenerName=ListenerName(PLAINTEXT), securityProtocol=PLAINTEXT, buffer=null) is not authorized.
```

也就是说，在只启用ACL，而内部节点通信为PLAINTEXT时，对应的用户其实是ANONYMOUS，这使得节点之间的请求在ACL验证阶段不可用。除非allow.everyone.if.no.acl.found=true，但既然启用了ACL，我们肯定希望没有ACL的用户不可以操作资源。

### 同时启用SASL与ACL

如果启用ACL，需要让内部节点的通信启用SASL，为了简单，可以让内部节点通信使用SASL_PLAIN

这样的话，SASL的listener可能有多个

假设我们对节点内部使用BROKERS这个listener，对内网的客户端使用INTERNAL这个listener，BROKERS使用PLAIN机制，INTERNAL使用SCRAM机制。

#### 配置文件可能是这样的

```
# ACL部分
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false
super.users=User:admin

listeners=INTERNAL://10.0.0.1:9092,BROKERS://10.0.0.1:9094
listener.security.protocol.map=INTERNAL:SASL_PLAINTEXT,BROKERS:SASL_PLAINTEXT
inter.broker.listener.name=BROKERS
#security.inter.broker.protocol=
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=SCRAM-SHA-512,PLAIN
```

#### 然后为所有的节点配置jaas

让服务器的用户名为admin，admin在ACL的超级管理员列表里，就没有ACL的问题了
```
# cat /usr/local/kafka/config/kafka_server_jaas.conf
internal.KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required;
};
brokers.KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
        username="admin"
        password="合适的密码"
        user_admin="合适的密码";
};
```

#### 将jaas配置传入JVM启动Kafka即可
```
KAFKA_HEAP_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_server_jaas.conf" /usr/local/kafka/bin/kafka-server-start.sh  -daemon /usr/local/kafka/config/server.properties
```

## 代码测试服务端是否正常工作
```
Map<String, Object> config = new HashMap<>();
config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "10.0.0.1:9092");
config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, org.apache.kafka.common.serialization.StringSerializer.class);
config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, org.apache.kafka.common.serialization.StringSerializer.class);
config.put(ProducerConfig.ACKS_CONFIG, "-1");
KafkaProducer producer = new KafkaProducer(config);

Future future = producer.send(new ProducerRecord("renjianzhi-test", "testkey", "testvalue"));
System.out.println(future.get());
```

客户端报错
2020-10-30 19:14:42.321  WARN 66496 --- [ad | producer-1] org.apache.kafka.clients.NetworkClient   : [Producer clientId=producer-1] Bootstrap broker x.x.x.x:9092 (id: -1 rack: null) disconnected

服务端报错
[2020-10-30 19:16:52,507] INFO [SocketServer brokerId=0] Failed authentication with x.x.x.x (Unexpected Kafka request of type METADATA during SASL handshake.) (org.apache.kafka.common.network.Selector)

说明SASL握手时失败了，添加SASL配置
```
config.put("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required\n" +
        "username=\"renjianzhi\" \n" +
        "password=\"密码\";");
config.put("sasl.mechanism", "SCRAM-SHA-512");
config.put("security.protocol", "SASL_SSL");
```

客户端报错
Caused by: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.SaslAuthenticationException: Authentication failed during authentication due to invalid credentials with SASL mechanism SCRAM-SHA-512

服务端报错
[2020-10-31 01:43:22,429] INFO [SocketServer brokerId=0] Failed authentication with x.x.x.x (Authentication failed during authentication due to invalid credentials with SASL mechanism SCRAM-SHA-512) (org.apache.kafka.common.network.Selector)

是因为没有创建这个用户，使用
```
# ./kafka-configs.sh --zookeeper x.x.x.x:2181 --alter --add-config 'SCRAM-SHA-512=[password=密码]' --entity-type users --entity-name renjianzhi
Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
Use --bootstrap-server instead to specify a broker to connect to.
Completed updating config for entity: user-principal 'renjianzhi'.
```

再次重试，SASL部分就一切正常了。
如果同时启用了ACL，但是SASL的用户没有分配权限，还会有这种报错：
```
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics
```

这里的报错跟SASL就没有关系了，需要看看ACL部分的文档。



