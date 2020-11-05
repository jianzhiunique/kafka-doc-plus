# SSL配置

SSL可以用于加密和认证，但是推荐仅使用SSL的加密，不使用SSL认证，使用SASL认证。

## 准备

在所有节点上执行 
```
mkdir -p /usr/ca/{root,server,client,trust}
```

注意：此章节中使用{}标记的变量都应该替换为合适的值，如{keypass}等应该使用合适的密码。

## 1. 生成SSL key和证书（所有节点）

```
keytool -keystore /usr/ca/server/server.keystore.jks -alias `hostname` -validity 3650 -genkey -keypass {keypass} -keyalg RSA -dname "CN={"`hostname`"},OU={ou},O={o},L=beijing,S=beijing,C=cn" -storepass {keypass} -ext SAN=IP:{ip}
```

注意：未知问题，CA签名后会把SAN抹掉，所以客户端要关闭节点认证才可以

## 2.1 创建CA（一个节点）

CA在一个节点生成，在所有节点使用,所以这个命令只需要在一个节点上执行


```
openssl req -new -x509 -keyout /usr/ca/root/ca-key -out /usr/ca/root/ca-cert -days 3650 -passout pass:{pass} -subj "/C=cn/ST=beijing/L=beijing/O={o}/OU={ou}/CN={"`hostname`"}"
```

生成完毕后将/usr/ca/root/*下的所有文件放到其他节点上

## 2.2 通过CA证书创建一个客户端信任证书（一个节点，一次生成即可）

客户端信任证书将会提供给诸如生产者/消费者客户端使用。
```
keytool -keystore /usr/ca/trust/client.truststore.jks -alias CARoot -import -file /usr/ca/root/ca-cert -storepass {storepass}
```

## 2.3 通过CA证书创建一个服务端器端信任证书（所有节点）
服务端信任证书是提供给服务端使用的，让节点之间可以相互信任
```
keytool -keystore /usr/ca/trust/server.truststore.jks -alias CARoot -import -file /usr/ca/root/ca-cert -storepass {storepass}
```

## 3.1 导出服务端证书（所有节点）
```
keytool -keystore /usr/ca/server/server.keystore.jks -alias `hostname` -certreq -file /usr/ca/server/server.cert-file -storepass {storepass}
```

## 3.2 用CA证书给服务端的证书签名（所有节点）
```
openssl x509 -req -CA /usr/ca/root/ca-cert -CAkey /usr/ca/root/ca-key -in /usr/ca/server/server.cert-file -out /usr/ca/server/server.cert-signed -days 3650 -CAcreateserial -passin pass:{pass}
```

## 3.3 将CA证书导入服务端的密钥库（所有节点）
```
keytool -keystore /usr/ca/server/server.keystore.jks -alias CARoot -import -file /usr/ca/root/ca-cert -storepass {storepass}
```

## 3.4 将签名后的证书覆盖未签名的证书（所有节点）
```
keytool -keystore /usr/ca/server/server.keystore.jks -alias `hostname` -import -file /usr/ca/server/server.cert-signed -storepass {storepass}
```

注意：如果要启用ssl的认证，对客户端的签名方式跟上面的步骤相似。

## 自动化
```
#!/bin/bash
mkdir -p /usr/ca/{root,server,client,trust}

ou=xxx
o=xxx
l=beijing
s=beijing
c=cn
validity=3650
keypass=xxx
storepass=xxx
pass=xxx
clienttrustpass=xxx
servertrustpass=xxx
master=yes

# 生成证书
keytool -keystore /usr/ca/server/server.keystore.jks -alias `hostname` -validity $validity -genkey -keypass $keypass -keyalg RSA -dname "CN="`hostname`",OU="$ou",O="$o",L="$l",S="$s",C="$c -storepass $storepass
# 如果是master节点生成CA证书、生成客户端信任证书
if [ "x"$master = "xyes" ]; then
    openssl req -new -x509 -keyout /usr/ca/root/ca-key -out /usr/ca/root/ca-cert -days $validity -passout pass:$pass -subj "/C="$c"/ST="$s"/L="$l"/O="$o"/OU="$ou"/CN="`hostname`
    keytool -keystore /usr/ca/trust/client.truststore.jks -alias CARoot -import -file /usr/ca/root/ca-cert -storepass $clienttrustpass
fi
# 生成服务端信任证书
keytool -keystore /usr/ca/trust/server.truststore.jks -alias CARoot -import -file /usr/ca/root/ca-cert -storepass $servertrustpass
# 为服务端证书签名
keytool -keystore /usr/ca/server/server.keystore.jks -alias `hostname` -certreq -file /usr/ca/server/server.cert-file -storepass $storepass
openssl x509 -req -CA /usr/ca/root/ca-cert -CAkey /usr/ca/root/ca-key -in /usr/ca/server/server.cert-file -out /usr/ca/server/server.cert-signed -days $validity -CAcreateserial -passin pass:$pass
# 将CA证书和签名后的证书导入服务端证书
keytool -keystore /usr/ca/server/server.keystore.jks -alias CARoot -import -file /usr/ca/root/ca-cert -storepass $storepass
keytool -keystore /usr/ca/server/server.keystore.jks -alias `hostname` -import -file /usr/ca/server/server.cert-signed -storepass $storepass
# 验证证书
keytool -list -v -keystore /usr/ca/server/server.keystore.jks
```

## 验证证书
```
keytool -list -v -keystore /usr/ca/server/server.keystore.jks
```

## 修改Kafka配置文件

添加以下配置来启用SSL机制，注意{password}应该是正确的密码
```
### SSL相关
ssl.protocol=TLSv1.2
ssl.trustmanager.algorithm=PKIX
ssl.keystore.location=/usr/ca/server/server.keystore.jks
ssl.keystore.password={password}
ssl.key.password={password}
ssl.truststore.location=/usr/ca/trust/server.truststore.jks
ssl.truststore.password={password}
# 不开启SSL认证，麻烦，只用SASL认证就可以了
ssl.client.auth=none
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
ssl.keystore.type=JKS
ssl.truststore.type=JKS
ssl.endpoint.identification.algorithm=https
```

配置SSL的listener
```
如：
listeners=EXTERNAL://外网IP:9093
listener.security.protocol.map=SSL

或者
listeners=SSL://外网IP:9093
```


## 重启服务器

## 快速验证SSL
```
openssl s_client -debug -connect localhost:9093 -tls1
TLSv1 should be listed under ssl.enabled.protocols
```

## 使用代码验证SSL
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
2020-10-30 14:51:52.875  WARN 39701 --- [ad | producer-1] org.apache.kafka.clients.NetworkClient   : [Producer clientId=producer-1] Bootstrap broker xxx:9092 (id: -1 rack: null) disconnected

服务端报错
[2020-10-30 14:52:25,838] INFO [SocketServer brokerId=0] Failed authentication with xxx (SSL handshake failed) (org.apache.kafka.common.network.Selector)

说明不配置SSL是连接不上的,客户端添加以下配置
```
config.put("security.protocol", "SSL");
config.put("ssl.truststore.location", "client.truststore.jks");
config.put("ssl.truststore.password", "密码");
config.put("ssl.endpoint.identification.algorithm", "");//由于上面SAN的问题，这里使用这句代码来取消主机名认证，否则会报错
```

然后重试代码就可以正常连接了。