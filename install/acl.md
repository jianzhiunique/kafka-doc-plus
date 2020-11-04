# ACL配置

再不启用ACL机制的Kafka上可能存在一些问题，比如，客户端可以对任意Topic进行生产/消费，可以访问其他业务的消费者组（大部分可能是无意的，只是消费者组名称重复了而已），这样的话问题非常大，当Kafka的规模够大，用户多了之后，有必要启用ACL机制，方便管理，对Kafka集群的稳定性也很有帮助。

## 启用ACL

修改Kafka配置文件以启用ACL

```
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
# 如果找不到ACL，不允许除super users外的任何人访问
allow.everyone.if.no.acl.found=false
# 超管用户列表
super.users=User:kafka_admin;User:admin
```

## ACL操作

详见Kafka文档的ACL部分

## 代码验证ACL

#### 生产者场景

假设我们使用一个没有生产者权限的用户来往Topic中发送消息，会看到下面的报错
```
客户端报错
Caused by: java.util.concurrent.ExecutionException: org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [renjianzhi-test]

服务端
[2020-10-31 01:55:57,264] INFO Processing notification(s) to /config/changes (kafka.common.ZkNodeChangeNotificationListener)
[2020-10-31 01:55:57,273] INFO Processing override for entityPath: users/renjianzhi with config: Map(SCRAM-SHA-512 -> [hidden]) (kafka.server.DynamicConfigManager)
[2020-10-31 01:55:57,276] INFO Removing PRODUCE quota for user renjianzhi (kafka.server.ClientQuotaManager)
[2020-10-31 01:55:57,278] INFO Removing FETCH quota for user renjianzhi (kafka.server.ClientQuotaManager)
[2020-10-31 01:55:57,278] INFO Removing REQUEST quota for user renjianzhi (kafka.server.ClientRequestQuotaManager)
```

没有权限访问topic，需继续配置ACL才可以
```
# ./kafka-acls.sh --authorizer-properties zookeeper.connect=x.x.x.x:2181  --allow-principal User:renjianzhi --allow-host '*' --producer --topic renjianzhi-test --add
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=renjianzhi-test, patternType=LITERAL)`:
 	(principal=User:renjianzhi, host=*, operation=WRITE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=CREATE, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=renjianzhi-test, patternType=LITERAL)`:
 	(principal=User:renjianzhi, host=*, operation=WRITE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=CREATE, permissionType=ALLOW)
```

这样就正常发送了。

#### 消费者场景

同样，如果消费者没有权限访问Topic以及Group，同样会出现未授权问题。

```
Map<String, Object> config = new HashMap();
config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "x.x.x.x:9092");
config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, org.apache.kafka.common.serialization.StringDeserializer.class);
config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, org.apache.kafka.common.serialization.StringDeserializer.class);
config.put(ConsumerConfig.GROUP_ID_CONFIG, "renjianzhi-group");

config.put("security.protocol", "SSL");
config.put("ssl.truststore.location", "client.truststore.jks");
config.put("ssl.truststore.password", "密码");
config.put("ssl.endpoint.identification.algorithm", "");

config.put("sasl.jaas.config", "org.apache.kafka.common.security.scram.ScramLoginModule required\n" +
        "username=\"renjianzhi\" \n" +
        "password=\"密码\";");
config.put("sasl.mechanism", "SCRAM-SHA-512");
config.put("security.protocol", "SASL_SSL");

KafkaConsumer consumer = new KafkaConsumer<String, String>(config);
consumer.subscribe(Collections.singleton("renjianzhi-test"));
consumer.poll(Duration.ofSeconds(5));
while(true){
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));
    records.forEach(System.out::println);
}
```

报错信息Caused by: org.apache.kafka.common.errors.GroupAuthorizationException: Not authorized to access group: renjianzhi-group

这时需要添加消费权限

```
# ./kafka-acls.sh --authorizer-properties zookeeper.connect=x.x.x.x:2181  --allow-principal User:renjianzhi --allow-host '*' --consumer --topic renjianzhi-test --add --group renjianzhi-group
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=renjianzhi-test, patternType=LITERAL)`:
 	(principal=User:renjianzhi, host=*, operation=READ, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=DESCRIBE, permissionType=ALLOW)

Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=renjianzhi-group, patternType=LITERAL)`:
 	(principal=User:renjianzhi, host=*, operation=READ, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=renjianzhi-group, patternType=LITERAL)`:
 	(principal=User:renjianzhi, host=*, operation=READ, permissionType=ALLOW)

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=renjianzhi-test, patternType=LITERAL)`:
 	(principal=User:renjianzhi, host=*, operation=WRITE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=CREATE, permissionType=ALLOW)
	(principal=User:renjianzhi, host=*, operation=READ, permissionType=ALLOW)
```



