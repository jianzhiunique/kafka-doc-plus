# SSL
## 1. 为所有节点生成SSL key和证书
generate the key into a temporary keystore，export and sign it later with CA

```
keytool -keystore server.keystore.jks -alias localhost -validity {validity} -genkey -keyalg RSA
keystore：存储证书的密钥库文件。密钥库文件包含证书的私钥；因此，需要安全地保存它。
validity：证书的有效期，以天为单位。
```

#### 配置主机名验证
从Kafka 2.0.0版开始，默认情况下，客户端连接和代理间连接都会启用服务器的主机名验证，以防止中间人攻击。
```
disabled by setting ssl.endpoint.identification.algorithm to an empty string.
```
对于动态配置的代理listener
```
bin/kafka-configs.sh --bootstrap-server localhost:9093 --entity-type brokers --entity-name 0 --alter --add-config "listener.name.internal.ssl.endpoint.identification.algorithm="
```
对于老版本,增加配置
```
ssl.endpoint.identification.algorithm=HTTPS ，to enable host name verification.
```
必须启用主机名验证，以防止中间人攻击（如果服务器终结点未在外部验证）。

注：即使服务端配置了主机名验证，客户端也可以配置不使用主机名验证。

#### 在证书中配置主机名
如果启用主机名验证，客户端根据Common Name (CN)、Subject Alternative Name (SAN)验证服务器的FQDN
RFC-2818建议使用SAN
To add a SAN field，-ext SAN=DNS:{FQDN} to the keytool command
```
keytool -keystore server.keystore.jks -alias localhost -validity {validity} -genkey -keyalg RSA -ext SAN=DNS:{FQDN}
```
之后可以运行以下命令来验证
```
keytool -list -v -keystore server.keystore.jks
```

## 2.创建自己的CA
第一步之后，集群中的每台机器都有一个公私密钥对，以及一个证书来标识机器。
但是，该证书是未签名的，这意味着攻击者可以创建这样的证书来伪装成任何计算机。
对群集中的每台计算机进行签名来防止伪造证书
证书颁发机构（CA）负责签署证书。只要CA是一个真正可信的权威机构，客户机就可以高度保证他们正在连接到可信的机器上。
```
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
```
生成的CA只是一个公钥-私钥对和证书，它用于签署其他证书。

#### 通过CA证书创建一个客户端信任证书
将生成的CA添加到客户端的信任库，这样客户端就可以信任这个CA
```
keytool -keystore client.truststore.jks -alias CARoot -import -file ca-cert
```

#### 通过CA证书创建一个服务端器端信任证书
如果服务端设置ssl.client.auth 为 "requested" or "required"，还必须为Kafka代理提供一个信任库，并且它应该具有客户端密钥签名的所有CA证书。
```
keytool -keystore server.truststore.jks -alias CARoot -import -file ca-cert
```
与步骤1中存储每台机器自己的标识的密钥库不同，client的信任库存储client应该信任的所有证书。

#### 信任链
将证书导入信任库还意味着信任由该证书签名的所有证书。信任政府（CA）也意味着信任政府签发的所有护照（证书）。这个属性称为信任链，
当在大型Kafka集群上部署SSL时，它特别有用。您可以使用单个CA对群集中的所有证书进行签名，并让所有计算机共享信任CA的同一个信任库。这样，所有计算机都可以对所有其他计算机进行身份验证。

## 3.签署证书
服务器证书的签名处理
是用步骤2中生成的CA对步骤1生成的所有证书进行签名。
#### 3.1首先，需要从密钥库导出证书
```
keytool -keystore server.keystore.jks -alias localhost -certreq -file cert-file
```
#### 3.2然后用CA签署它
```
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days {validity} -CAcreateserial -passin pass:{ca-password}
```
#### 3.3最后，将CA的证书和签名的证书导入密钥库
```
keytool -keystore server.keystore.jks -alias CARoot -import -file ca-cert
keytool -keystore server.keystore.jks -alias localhost -import -file cert-signed

keystore: the location of the keystore 密钥库的位置
ca-cert: the certificate of the CA ca的证书
ca-key: the private key of the CA ca的私钥
ca-password: the passphrase of the CA ca的密码短语
cert-file: the exported, unsigned certificate of the server 导出的、未签名的服务器证书
cert-signed: the signed certificate of the server 服务器的签名证书
```

## 整体shell
```
#!/bin/bash
#Step 1
keytool -keystore server.keystore.jks -alias localhost -validity 365 -keyalg RSA -genkey
#Step 2
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
keytool -keystore server.truststore.jks -alias CARoot -import -file ca-cert
keytool -keystore client.truststore.jks -alias CARoot -import -file ca-cert
#Step 3
keytool -keystore server.keystore.jks -alias localhost -certreq -file cert-file
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:test1234
keytool -keystore server.keystore.jks -alias CARoot -import -file ca-cert
keytool -keystore server.keystore.jks -alias localhost -import -file cert-signed
```

## 便捷配置

官方的方法需要每次都输入密码之类的信息，可以在命令行传入这些信息，具体请直接看安装文档里的SSL配置部分。


## 4.配置kafka
Kafka代理支持监听多个端口上的连接。在server.properties有一个或多个逗号分割的值
```
listeners=PLAINTEXT://host.name:port,SSL://host.name:port
ssl.keystore.location=/var/private/ssl/server.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
ssl.truststore.location=/var/private/ssl/server.truststore.jks
ssl.truststore.password=test1234 技术上是可选的，但强烈建议使用。如果没有设置密码，对信任库的访问仍然可用，但完整性检查被禁用。
```
#### 值得考虑的可选设置：
```
ssl.client.auth=none
"required" => client authentication is required, 
"requested" => client authentication is requested and client without certs can still connect. 
不鼓励使用“requested”，因为它提供了一种错误的安全感，并且配置错误的客户端仍将成功连接。
ssl.cipher.suites 密码套件是认证、加密、MAC和密钥交换算法的命名组合，用于协商使用TLS或SSL网络协议的网络连接的安全设置。（默认为空列表）
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1 列出您将从客户端接受的SSL协议。请注意，SSL已被弃用，取而代之的是TLS，不建议在生产中使用SSL
ssl.keystore.type=JKS
ssl.truststore.type=JKS
ssl.secure.random.implementation=SHA1PRNG 有些实现存在性能问题，使用全局锁，在SSL连接的性能成为问题的情况下，考虑显式地设置要使用的实现，SHA1PRNG is non-blocking，在高负载下表现出非常好的性能特征
如果想启用内部SSL：security.inter.broker.protocol=SSL
```

由于某些国家的进口法规，Oracle的实现限制了默认情况下可用的加密算法的强度。如果需要更强大的算法（例如，具有256位密钥的AES），则必须获取JCE Unlimited-Strength辖区策略文件并将其安装在JDK/JRE中。

#### 快速检查
要快速检查服务器密钥库和信任库是否设置正确，可以运行以下命令。
```
openssl s_client -debug -connect localhost:9093 -tls1
TLSv1 should be listed under ssl.enabled.protocols
```

## 5.配置kafka客户端
SSL只支持新的Kafka生产者和消费者，对于生产者和消费者，SSL的配置是相同的。
如果代理中不需要客户端身份验证，则以下是最小配置示例：
```
security.protocol=SSL
ssl.truststore.location=/var/private/ssl/client.truststore.jks
ssl.truststore.password=test1234 可选的，但强烈推荐。
```
如果需要客户端身份验证，则必须创建密钥库，如步骤1所示，并且还必须配置以下内容：
```
ssl.keystore.location=/var/private/ssl/client.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
```
可能还需要其他配置设置：
```
ssl.provider 用于SSL连接的安全提供程序的名称。 Default value is the default security provider of the JVM.
ssl.cipher.suites 密码套件是认证、加密、MAC和密钥交换算法的命名组合，用于协商使用TLS或SSL网络协议的网络连接的安全设置。
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1 应该列出在代理端配置的至少一个协议
ssl.truststore.type=JKS
ssl.keystore.type=JKS
```

## Examples using console-producer and console-consumer:
```
kafka-console-producer.sh --bootstrap-server localhost:9093 --topic test --producer.config client-ssl.properties
kafka-console-consumer.sh --bootstrap-server localhost:9093 --topic test --consumer.config client-ssl.properties
```