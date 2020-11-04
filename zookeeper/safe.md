# zookeeper安全
## ZooKeeper Authentication 认证

ZooKeeper从3.5.x版本开始支持相互TLS（mTLS）身份验证。同时支持KafkKeeper和Kafk2.5版本。有关更多详细信息，请参阅KIP-515:enablezk-client使用新的TLS支持的身份验证。

当单独使用mTLS时，每个代理和任何CLI工具（例如ZooKeeper安全迁移工具）都应该使用相同的可分辨名称（DN）来标识自己，因为它是ACL'ed的DN。这可以按如下所述进行更改，但它涉及到编写和部署自定义ZooKeeper身份验证提供程序。一般来说，每个证书都应该具有相同的DN，但有一个不同的subjectalternative Name（SAN），这样ZooKeeper对代理和任何CLI工具的主机名验证都将成功。

将SASL身份验证与mTLS一起用于ZooKeeper时，SASL标识和创建znode的DN（即创建代理的证书）或安全迁移工具的DN（如果迁移是在znode创建之后执行的）都将被ACL'd，并且所有代理和CLI工具都将被授权，即使它们都是使用不同的DNs，因为它们都将使用相同的ACL'ed SASL标识。只有在单独使用mTLS身份验证时，所有DNs都必须匹配（san变得非常关键——同样，在没有编写和部署自定义ZooKeeper身份验证提供程序的情况下，如下所述）。

使用代理属性文件为代理设置TLS配置，如下所述。
Use the --zk-tls-config-file <file> option to set TLS configs in the Zookeeper Security Migration Tool. The kafka-acls.sh and kafka-configs.sh CLI tools also support the --zk-tls-config-file <file> option.

Use the -zk-tls-config-file <file> option (note the single-dash rather than double-dash) to set TLS configs for the zookeeper-shell.sh CLI tool.

新zookeeper集群
ZooKeeper SASL认证
要在代理上启用ZooKeeper SASL身份验证，有两个必要步骤：
创建一个JAAS登录文件，并将适当的系统属性设置为如上所述指向它
zookeeper.set.acl in each broker to true
存储在ZooKeeper中的Kafka集群的元数据是world-readable的，但是只能由代理修改。
理由是，ZooKeeper中存储的数据并不敏感，但对这些数据的不当操作可能会导致集群中断。
我们还建议通过网络分段限制对ZooKeeper的访问（只有代理和一些管理工具需要访问ZooKeeper）。

ZooKeeper相互TLS认证
ZooKeeper mTLS身份验证可以启用，也可以不使用SASL身份验证。如上所述，当单独使用mTLS时，每个代理和任何CLI工具（如ZooKeeper安全迁移工具）通常必须使用相同的可分辨名称（DN）来标识自己，因为DN是ACL'ed的，这意味着每个证书都应该有一个适当的subjectalternative Name（SAN），这样ZooKeeper对代理和任何CLI工具的主机名验证都将成功。

通过编写一个扩展的类（继承org.apache.zookeeper.server.auth.X509AuthenticationProvider），可以使用DN以外的东西来标识mTLS客户机重写protected String getClientId(X509Certificate clientCert)
Choose a scheme name and set authProvider.[scheme] in ZooKeeper to be the fully-qualified class name of the custom implementation; then set ssl.authProvider=[scheme] to use it.

下面是一个示例（部分）ZooKeeper配置，用于启用TLS身份验证。
secureClientPort=2182
serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
authProvider.x509=org.apache.zookeeper.server.auth.X509AuthenticationProvider
ssl.keyStore.location=/path/to/zk/keystore.jks
ssl.keyStore.password=zk-ks-passwd
ssl.trustStore.location=/path/to/zk/truststore.jks
ssl.trustStore.password=zk-ts-passwd

重要提示：ZooKeeper不支持将ZooKeeper客户端（即代理）密钥库中的密钥密码设置为与密钥库密码本身不同的值。请确保将密钥密码设置为与密钥库密码相同。
