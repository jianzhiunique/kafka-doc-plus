# SASL鉴权
## JAAS配置
### 1.服务端的JAAS配置
#### KafkaServer
服务端用KafkaServer这个名字，这块是为服务端节点的配置，包括节点内部通讯时建立的任何SASL连接
如果多个监听器配置使用SASL，那么名字可以用小写的监听器名作为前缀，如sasl_ssl.KafkaServer

#### Client
Client部分用于与zookeeper之间的认证，还可以设置SASL ACL以便只有代理可以修改
所有节点必须使用相同的名字，如果想用其他名字，需要设置-Dzookeeper.sasl.clientconfig=ZkClient

zookeeper默认使用zookeeper作为服务名，如果更改，-Dzookeeper.sasl.client.username=zk

#### sasl.jaas.config
服务端也可能使用sasl.jaas.config来配置JAAS，属性名必须用（包含SASL授权机制的）监听器作为前缀
如listener.name.{listenerName}.{saslMechanism}.sasl.jaas.config，只能在config值中指定一个登录模块。

如果在侦听器上配置了多个机制，则必须使用侦听器和机制前缀为每个机制提供配置。
如：
```
listener.name.sasl_ssl.scram-sha-256.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";
listener.name.sasl_ssl.plain.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="admin" \
    password="admin-secret" \
    user_admin="admin-secret" \
    user_alice="alice-secret";
```

#### JAAS优先顺序
如果JAAS配置在不同级别定义，则使用的优先顺序为
```
Broker configuration property listener.name.{listenerName}.{saslMechanism}.sasl.jaas.config
{listenerName}.KafkaServer section of static JAAS configuration
KafkaServer section of static JAAS configuration
```

ZooKeeper JAAS配置只能使用静态JAAS配置进行配置。
有关代理配置的示例，请参阅GSSAPI（Kerberos）、PLAIN、SCRAM或OAUTHBEARER。

## SASL配置
SASL可以使用PLAINTEXT or SSL作为传输层，分别使用安全协议SASL_PLAINTEXT or SASL_SSL

### SASL机制
```
GSSAPI (Kerberos)
PLAIN
SCRAM-SHA-256
SCRAM-SHA-512
OAUTHBEARER
```

### 代理的SASL配置
在server.properties中配置一个SASL端口，至少是SASL_PLAINTEXT or SASL_SSL中的一个
```
listeners=SASL_PLAINTEXT://host.name:port
```
如果只配置SASL端口（或者希望Kafka代理使用SASL相互验证），请确保为代理间通信设置相同的SASL协议：
```
security.inter.broker.protocol=SASL_PLAINTEXT (or SASL_SSL)
```

### 选择机制
选择要在代理中启用的一个或多个受支持的机制，然后按照步骤为该机制配置SASL。

#### 以SCRAM-SHA-512为例
Salted Challenge Response Authentication Mechanism（SCRAM）是一系列SASL机制，它解决了传统机制（如PLAIN和DIGEST-MD5）的安全问题。该机制在RFC 5802中定义。Kafka支持SCRAM-SHA-256和SCRAM-SHA-512，它们可以与TLS一起使用来执行安全认证。用户名用作配置ACL等的认证主体。

kafka默认的实现是存储在zookeeper中，适用于zookeeper在私有网络上的情况。

#### 1.创建SCRAM凭据
可以用kafka-configs.sh在zookeeper中创建
在Kafka代理启动之前，内部通讯凭据需要创建
客户端的凭据可以动态创建和更新，更新后的可以被新的连接使用

user alice，password alice-secret的创建方式
```
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=alice-secret],SCRAM-SHA-512=[password=alice-secret]' --entity-type users --entity-name alice
default iteration count of 4096，如果不指定，随机的salt会被创建，salt + iterations + StoredKey和ServerKey组成的SCRAM标识存储在Zookeeper中。
```

下面的例子是内部节点通讯使用的admin用户
```
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-256=[password=admin-secret],SCRAM-SHA-512=[password=admin-secret]' --entity-type users --entity-name admin
```

可以使用--describe选项列出现有凭据
```
bin/kafka-configs.sh --zookeeper localhost:2181 --describe --entity-type users --entity-name alice
```
可以用--delete选项删除一个或多个凭据
```
bin/kafka-configs.sh --zookeeper localhost:2181 --alter --delete-config 'SCRAM-SHA-512' --entity-type users --entity-name alice
```

#### 2.配置kafka服务端
2.1 在每个Kafka代理的config目录中添加一个与下面类似的经过适当修改的JAAS文件 kafka_server_jaas.conf
```
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-secret";
};
```
代理使用KafkaServer部分中的username和password属性初始化与其他代理的连接。在本例中，admin是代理间通信的用户。
2.2 将JAAS配置文件位置作为JVM参数传递给每个Kafka代理：
```
-Djava.security.auth.login.config=/etc/kafka/kafka_server_jaas.conf
```
2.3 在server.properties中配置SASL port and SASL mechanisms
```
listeners=SASL_SSL://host.name:port
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-256 (or SCRAM-SHA-512)
sasl.enabled.mechanisms=SCRAM-SHA-256 (or SCRAM-SHA-512)
```

#### 3.配置kafka客户端
3.1 在producer.properties or consumer.properties中配置JAAS配置
下面是个示例的客户端配置
```
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
        username="alice" \
        password="alice-secret";
```

客户端使用username和password选项来配置用户进行客户端连接。在本例中，客户机作为用户alice连接到代理。JVM中的不同客户端可以通过在中指定不同的用户名和密码作为不同的用户进行连接sasl.jaas.config文件.
客户端使用KafkaClient这个名字作为登录section。这样的话，同一个JVM的所有连接都只能用这一个用户。

3.2 在producer.properties or consumer.properties中配置
```
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-256 (or SCRAM-SHA-512)
```

#### 4.SASL/SCRAM的安全考虑
凭证存储在Zookeeper中。这适用于Zookeeper安全的安装和专用网络中的生产使用。
Kafka只支持强散列函数SHA-256和SHA-512，最小迭代次数为4096次。如果Zookeeper的安全性受到威胁，强大的哈希函数与强大的密码和高迭代次数相结合，可以防止暴力攻击。
SCRAM 只能与TLS加密一起使用，以防止截获 SCRAM exchanges.这可以防止字典或暴力攻击，并在Zookeeper受损时防止冒充。
从Kafka 2.0版起，可以通过配置自定义回调处理程序

#### JVM JAAS
客户机的JAAS配置也可以指定为JVM参数，类似与http://kafka.apache.org/25/documentation.html#security_client_staticjaas

#### 启用多个机制
要在代理中启用多个机制，请执行以下步骤。

##### Specify configuration for the login modules of all enabled mechanisms in the KafkaServer section of the JAAS config file. For example:
```
KafkaServer {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    storeKey=true
    keyTab="/etc/security/keytabs/kafka_server.keytab"
    principal="kafka/kafka1.hostname.com@EXAMPLE.COM";

    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="admin-secret"
    user_admin="admin-secret"
    user_alice="alice-secret";
};
```
##### Enable the SASL mechanisms in server.properties:
```
    sasl.enabled.mechanisms=GSSAPI,PLAIN,SCRAM-SHA-256,SCRAM-SHA-512,OAUTHBEARER
```
##### Specify the SASL security protocol and mechanism for inter-broker communication in server.properties if required:
```
    security.inter.broker.protocol=SASL_PLAINTEXT (or SASL_SSL)
    sasl.mechanism.inter.broker.protocol=GSSAPI (or one of the other enabled mechanisms)
```
##### Follow the mechanism-specific steps in GSSAPI (Kerberos), PLAIN, SCRAM and OAUTHBEARER to configure SASL for the enabled mechanisms.

### 客户端的SASL配置
SASL认证只支持新的javakafka生产者和消费者，不支持旧的API。
要在客户端上配置SASL身份验证，请选择在代理中为客户端身份验证启用的SASL机制，然后按照步骤为所选机制配置SASL。