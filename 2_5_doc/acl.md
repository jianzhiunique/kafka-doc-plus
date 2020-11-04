
# 鉴权ACL
Kafka附带了一个可插拔的授权器和一个现成的授权器实现，它使用zookeeper存储所有ACL。要启用ACL机制，需要修改配置文件
```
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
```

## ACL格式
Kafka ACL的一般格式是“Principal P is[Allowed/Denied]Operation O From Host H on any Resource R matching Resource RP”。

注：委托人P被允许/拒绝   操作O   从主机H   在资源R   匹配RP

为了添加、删除或列出ACL，可以使用Kafka authorizer CLI。

默认情况下，如果没有与特定资源R匹配的ResourcePatterns，则R没有关联的acl，因此除了超级用户之外，不允许任何人访问R。如果要更改该行为，
```
server.properties：allow.everyone.if.no.acl.found=true
```
可以在server.properties添加super users，如
```
super.users=User:Bob;User:Alice
默认PrincipalType字符串“User”区分大小写。
```

## 自定义用户名
#### 自定义SSL用户名（不太需要看）
默认情况下，SSL用户名的格式为

"CN=writeuser,OU=Unknown,O=Unknown,L=Unknown,ST=Unknown,C=Unknown"

可以在server.properties通过设置ssl.principal.map.rules为自定义的rule
此配置允许将X.500可分辨名称映射到短名称的规则列表。规则按顺序计算，第一个与可分辨名称匹配的规则用于将其映射到短名称。将忽略列表中后面的任何规则。
ssl.principal.map.rules是一个列表，其中每个规则都以“rule:”开头，并包含以下格式的表达式。
```
默认规则将返回X.500证书可分辨名称的字符串表示形式。
如果可分辨名称与模式匹配，则替换命令将在该名称上运行。它还支持小写/大写选项，以强制转换结果全部为小写/大写。这是通过在规则末尾添加“/L”或“/U”来完成的。
RULE:pattern/replacement/
RULE:pattern/replacement/[LU]
```

例子
```
RULE:^CN=(.*?),OU=ServiceUsers.*$/$1/,
RULE:^CN=(.*?),OU=(.*?),O=(.*?),L=(.*?),ST=(.*?),C=(.*?)$/$1@$2/L,
RULE:^.*[Cc][Nn]=([a-zA-Z0-9.]*).*$/$1/L,
DEFAULT
```

以上规则将
```
"CN=serviceuser,OU=ServiceUsers,O=Unknown,L=Unknown,ST=Unknown,C=Unknown" 转换为 "serviceuser"
"CN=adminUser,OU=Admin,O=Unknown,L=Unknown,ST=Unknown,C=Unknown" 转换为 "adminuser@admin"
```
对于高级用例，可以通过在server.properties中设置自定义PrincipalBuilder来自定义名称
```
principal.builder.class=CustomizedPrincipalBuilderClass
```

#### 自定义SASL用户名（不太需要看）
默认情况下，SASL用户名将是Kerberos主体的主要部分。

可以通过server.properties设置sasl.kerberos.principal.to.local.rules，是一个列表
其中每个规则的工作方式与Kerberos配置文件（krb5.conf）中的auth_to_local相同。这还支持额外的小写/大写规则，以强制翻译结果全部为小写/大写。这是通过在规则末尾添加“/L”或“/U”来完成的。检查以下格式的语法。每个规则以RULE:开头，并包含以下格式的表达式。有关更多详细信息，请参阅kerberos文档。

```
RULE:[n:string](regexp)s/pattern/replacement/
RULE:[n:string](regexp)s/pattern/replacement/g
RULE:[n:string](regexp)s/pattern/replacement//L
RULE:[n:string](regexp)s/pattern/replacement/g/L
RULE:[n:string](regexp)s/pattern/replacement//U
RULE:[n:string](regexp)s/pattern/replacement/g/U
```

添加规则以正确翻译的示例
```
user@MYDOMAIN.COM -》 user
sasl.kerberos.principal.to.local.rules=RULE:[1:$1@$0](.*@MYDOMAIN.COM)s/@.*//,DEFAULT
```

## Command Line Interface
http://kafka.apache.org/25/documentation.html#security_authz_cli

### kafka-acls.sh参数
#### Action类
```
--add
--remove
--list
```

#### Configuration类
```
--authorizer 指定鉴权类的名字 默认值为kafka.security.authorizer.AclAuthorizer
--authorizer-properties 指定鉴权类实例化的参数，key=val，如zookeeper.connect=localhost:2181
--bootstrap-server kafka的地址，与--authorizer只能使用一个
--command-config 配置文件（传给KafkaAdmin的），只能与--bootstrap-server一起用
--resource-pattern-type [pattern-type]
  向脚本指示用户希望使用的资源模式RP类型（for--add）或资源模式RP筛选器filter（for--list and--remove）
  添加ACL时，应该是特定的模式类型，例如“literal”或“prefixed”
  列出或删除ACL时，可以使用特定的filter列出或从特定类型的资源模式中删除ACL，也可以使用“any”或“match”
  'any' will match any pattern type, but will match the resource name exactly
  'match' will perform pattern matching to list or remove all acls that affect the supplied resource(s)
  警告：“match”与“--remove”开关结合使用时，应小心使用
--zk-tls-config-file zookeeper客户端的TLS连接属性
```

#### ResourcePattern类
```
--cluster 告诉脚本，在集群资源上操作ACL
--topic [topic-name] 告诉脚本，在topic资源上操作ACL
--group [group-name] 告诉脚本，在消费者组资源上操作ACL
--transactional-id [transactional-id] ACL要添加或移除的事务ID，*代表ACL将用于所有事务ID
--delegation-token [delegation-token] ACL要添加或移除的Delegation token，*代表ACL将用于所有Delegation token
```

#### Principal类
```
--allow-principal Principal的格式是PrincipalType:name，会被允许，默认的PrincipalType是User，可以在一条命令中配置多个--allow-principal
--deny-principal Principal的格式是PrincipalType:name，会被拒绝，默认的PrincipalType是User，可以在一条命令中配置多个--deny-principal
--principal Principal的格式是PrincipalType:name，这个是与--list一起使用的，将会列出与特定principal下的ACL列表，可以在一条命令中配置多个--principal
```

#### Host类
```
--allow-host IP地址列表，如果--allow-principal 是 *，代表所有hosts
--deny-host IP地址列表，如果--deny-principal 是 *，代表所有hosts
```

#### Operation类
```
--operation 被允许/禁止的操作，合法的值有：Read、Write、Create、Delete、Alter、Describe、ClusterAction、DescribeConfigs、AlterConfigs、IdempotentWrite、All，默认值All
```

#### Convenience类 便捷类
```
--producer 为producer的便捷选项，将会生成ACL：允许在Topic上WRITE, DESCRIBE and CREATE
--consumer 为consumer的便捷选项，将会生成ACL：允许在Topic上READ, DESCRIBE，以及在consumer-group上READ
--idempotent 为生产者启用幂等，应该与--producer结合起来使用。如果producer已经获得事务ID的授权，幂等性将会自动开启
--force assume yes to all queries and 不提示
```

#### ACL操作例子：
```
Principal P is[Allowed/Denied]Operation O From Host H on any Resource R matching Resource RP
```
##### 添加ACL
```
1.Principals User:Bob and User:Alice are allowed to perform Operation Read and Write on Topic Test-Topic from IP 198.51.100.0 and IP 198.51.100.1
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Bob --allow-principal User:Alice --allow-host 198.51.100.0 --allow-host 198.51.100.1 --operation Read --operation Write --topic Test-topic

2.默认情况下，所有没有显式acl（允许对资源的操作进行访问）的主体都将被拒绝。
在极少数情况下,ACL允许给all，使用 --deny-principal and --deny-host拒绝某些
allow all users to Read from Test-topic but only deny User:BadBob from IP 198.51.100.3
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:* --allow-host * --deny-principal User:BadBob --deny-host 198.51.100.3 --operation Read --topic Test-topic
注意--allow-host和deny-host仅支持IP地址，不支持hostname
上面的例子使用--topic [topic-name]作为RP来为topic增加ACL
类似的，可以使用--cluster来对集群增加ACL，--group [group-name]为消费者组增加ACL

3.可以在特定类型的任何资源上添加acl，例如，假如你想添加Principal User:Peter is allowed to produce to any Topic from IP 198.51.200.0，可以使用通配符资源“*”
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Peter --allow-host 198.51.200.1 --producer --topic *

4.可以在带前缀的资源模式上添加acl，Principal User:Jane is allowed to produce to any Topic whose name starts with 'Test-' from any host
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Jane --producer --topic Test- --resource-pattern-type prefixed
注意--resource-pattern-type 默认是 'literal'，将会影响确定名称的资源，以及*
```

##### 删除ACL
```
删除acl基本上是一样的。唯一的区别是用户必须指定--remove选项而不是--add，要删除上面第一个示例添加的ACL
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --remove --allow-principal User:Bob --allow-principal User:Alice --allow-host 198.51.100.0 --allow-host 198.51.100.1 --operation Read --operation Write --topic Test-topic 

如果你想删除前缀模式的
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --remove --allow-principal User:Jane --producer --topic Test- --resource-pattern-type Prefixed
```

##### 列出ACL
```
可以通过--list列出资源下的ACL，要列出文字RP Test-topic，可以执行
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --list --topic Test-topic

但是，这将只返回已添加到此确切资源模式中的ACL。可能存在影响对主题访问的其他ACL，例如主题通配符“*”上的任何ACL，或前缀资源模式上的任何ACL。
可以显式查询通配符资源模式上的ACL
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --list --topic *

但是，不一定能够在匹配测试主题的前缀资源模式上显式查询acl，因为这些模式的名称可能不知道。我们可以使用'--resource pattern type match'列出所有影响测试主题的ACL，
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --list --topic Test-topic --resource-pattern-type match
这将列出所有匹配的文字、通配符和前缀资源模式上的ACL。
```

##### 添加或删除producer/consumer
```
acl管理最常见的用例是添加/删除作为生产者或消费者的主体，因此我们添加了方便的选项来处理这些情况
User:Bob as a producer of Test-topic
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Bob --producer --topic Test-topic
类似地，将Alice添加为consumer group-1测试主题的消费者
bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Bob --consumer --topic Test-topic --group Group-1 
注意，对于consumer选项，我们还必须指定consumer组。为了从生产者或消费者角色中删除主体，我们只需要传递--remove选项
```

##### 基于管理API的acl管理
```
对ClusterResource具有Alter权限的用户可以使用管理API进行ACL管理
kafka-acls.sh支持AdminClient API来管理ACL，而无需与zookeeper/authorizer直接交互
以上所有示例都可以通过使用--bootstrap server选项来执行
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config /tmp/adminclient-configs.conf --add --allow-principal User:Bob --producer --topic Test-topic
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config /tmp/adminclient-configs.conf --add --allow-principal User:Bob --consumer --topic Test-topic --group Group-1
bin/kafka-acls.sh --bootstrap-server localhost:9092 --command-config /tmp/adminclient-configs.conf --list --topic Test-topic
```

## 授权原语
协议调用通常对Kafka中的某些资源执行一些操作。要建立有效的防护措施，就必须了解操作和资源。在本节中，我们将列出这些操作和资源，然后列出这些操作和资源与协议的组合，以查看有效的场景。

#### Operations in Kafka
有一些操作原语可以用来建立特权。它们可以与某些资源相匹配，以允许对给定用户进行特定的协议调用。这些是：

```
Read
Write
Create
Delete
Alter
Describe
ClusterAction
DescribeConfigs
AlterConfigs
IdempotentWrite
All
```

#### Resources in Kafka
上述操作可以应用于下面描述的某些资源。

Topic：这只是表示一个主题。所有作用于主题（如读、写）的协议调用都需要添加相应的特权。如果主题资源存在授权错误，则将返回topic_authorization_FAILED（错误代码：29）。

Group：它代表经纪人中的消费群体。与使用者组一起工作的所有协议调用（如加入组）都必须在主题中对该组具有特权。如果未授予特权，则协议响应中将返回组“授权”失败（错误代码：30）。

Cluster：这个资源代表集群。影响整个群集的操作（如受控关闭）受群集资源上的权限保护。如果群集资源上存在授权问题，则将返回群集“授权”失败（错误代码：31）。

TransactionalId：此资源表示与事务相关的操作，例如提交。如果发生任何错误，则代理将返回事务性的“身份验证”失败（错误代码：53）。

DelegationToken：表示集群中的委托令牌。描述委派令牌等操作可能受DelegationToken资源上的特权保护。由于这些对象在Kafka中有一些特殊的行为，因此建议在使用委托令牌进行身份验证时阅读KIP-48和相关的上游文档。

#### Operations and Resources on Protocols Kafka协议层的操作和资源
http://kafka.apache.org/25/documentation.html#operations_resources_and_protocols

PROTOCOL (API KEY)       |OPERATION&RESOURCE	        |NOTE                   
-------------------------|------------------------------|-----------------------
PRODUCE (0)             |Write TransactionalId          |事务性的生产者，transactional.id设置时，需要此权限
PRODUCE (0)             |IdempotentWrite Cluster        | 幂等性的生产需要
PRODUCE (0)             |Write Topic                    |普通生产者
FETCH (1)               |ClusterAction Cluster          |follower需要
FETCH (1)               |Read Topic                     |普通消费者需要
LIST_OFFSETS (2)        |Describe Topic                 |
METADATA (3)            |Describe   Topic               |
METADATA (3)            |Create Cluster                 |如果自动创建topic是开启的，服务端API将会检查是否存在集群层面的权限，如果有，允许创建，否则将会遍历Topic层面的权限，看下一个
METADATA (3)            |Create Topic	                |这授权自动创建topic，但是如果用户没有集群层面的权限
LEADER_AND_ISR (4)      |ClusterAction Cluster          |
STOP_REPLICA (5)        |ClusterAction Cluster	        |
UPDATE_METADATA (6)     |ClusterAction Cluster	        |
CONTROLLED_SHUTDOWN (7) |ClusterAction Cluster          |
OFFSET_COMMIT (8)       |Read Group                     |只有在授权了消费者组和topic后才能提交，首先验证组，然后验证topic
OFFSET_COMMIT (8)       |Read Topic                     |因为offset提交是消费进程的一环，所以他需要读权限
OFFSET_FETCH (9)        |Describe Group                 |与OFFSET_COMMIT类似，需要组、topic权限，但是这里需要的是Describe
OFFSET_FETCH (9)        |Describe Topic                 |
FIND_COORDINATOR (10)   |Describe Group                 |寻找消费者的协调者
FIND_COORDINATOR (10)   |Describe TransactionalId       |事务性生产者用，当生产者尝试寻找事务协调者
JOIN_GROUP (11)         |Read Group	                    |
HEARTBEAT (12)          |Read Group	                    |
LEAVE_GROUP (13)        |Read Group	                    |
SYNC_GROUP (14)         |Read Group                     |
DESCRIBE_GROUPS (15)    |Describe Group                 |
LIST_GROUPS (16)        |Describe Cluster               |服务端检查LIST_GROUPS请求的授权时，首先检查集群层面的权限，如果没有，会尝试单独的组。这个操作不返回CLUSTER_AUTHORIZATION_FAILED
LIST_GROUPS (16)        |Describe Group                 |如果没有群组权限，返回空而不是错误。不返回CLUSTER_AUTHORIZATION_FAILED，从2.1版本可用
SASL_HANDSHAKE (17)     |x x                            |SASL握手是认证的一部分，所以不太可能给哪种类型授权
API_VERSIONS (18)       |x x                            |是kafka协议握手的一部分，在连接和认证前发生，不太可能被授权控制
CREATE_TOPICS (19)      |Create Cluster                 |如果没有集群层面的授权，不会返回CLUSTER_AUTHORIZATION_FAILED，但会转移到topic层面（下面，会返回错误）
CREATE_TOPICS (19)      |Create Topic                   |2.0版本起可用
DELETE_TOPICS (20)      |Delete Topic	                |
DELETE_RECORDS (21)     |Delete Topic	                |
INIT_PRODUCER_ID (22)   |Write TransactionalId	        |
INIT_PRODUCER_ID (22)   |IdempotentWrite Cluster        |
OFFSET_FOR_LEADER_EPOCH (23)|ClusterAction Cluster      |如果没有集群层面的权限，那么将检查topic层面
OFFSET_FOR_LEADER_EPOCH (23)|Describe Topic             |2.1版本起可用
ADD_PARTITIONS_TO_TXN (24)|Write TransactionalId        |事务性请求使用，首先检查事务ID资源上的权限，然后检查topic上的权限
ADD_PARTITIONS_TO_TXN (24)|Write Topic	                |
ADD_OFFSETS_TO_TXN (25) |Write TransactionalId          |TransactionalId 事务性请求使用，首先检查事务ID资源上的权限，然后检查是否可以读取group
ADD_OFFSETS_TO_TXN (25) |Read Group	                    |
END_TXN (26)            |Write TransactionalId	        |
WRITE_TXN_MARKERS (27)  |ClusterAction Cluster	        |
TXN_OFFSET_COMMIT (28)  |Write TransactionalId	        |
TXN_OFFSET_COMMIT (28)  |Read Group	                    |
TXN_OFFSET_COMMIT (28)  |Read Topic	                    |
DESCRIBE_ACLS (29)      |Describe Cluster	            |
CREATE_ACLS (30)        |Alter Cluster	                |
DELETE_ACLS (31)        |Alter Cluster                  |
DESCRIBE_CONFIGS (32)   |DescribeConfigs Cluster        | 如果集群配置被请求，将会检查集群层面的权限
DESCRIBE_CONFIGS (32)   |DescribeConfigs Topic          | 如果topic配置被请求，将会检查topic层面的权限
ALTER_CONFIGS (33)      |AlterConfigs Cluster           | 如果集群配置被更改，将会检查集群层面的权限
ALTER_CONFIGS (33)      |AlterConfigs Topic             | 如果topic配置被更改，将会检查topic层面的权限
ALTER_REPLICA_LOG_DIRS (34)|Alter Cluster               |
DESCRIBE_LOG_DIRS (35)  |Describe Cluster               | 如果授权失败，将返回空响应
SASL_AUTHENTICATE (36)  |x x                            | SASL认证是认证的一部分，所以不太可能给哪种类型授权
CREATE_PARTITIONS (37)  |Alter Topic                    | 
CREATE_DELEGATION_TOKEN (38)|x x	                    |Creating delegation tokens has special rules, for this please see the Authentication using Delegation Tokens section.
RENEW_DELEGATION_TOKEN (39)|x x                         |Renewing delegation tokens has special rules, for this please see the Authentication using Delegation Tokens section.
EXPIRE_DELEGATION_TOKEN (40)|x  x	                    |Expiring delegation tokens has special rules, for this please see the Authentication using Delegation Tokens section.
DESCRIBE_DELEGATION_TOKEN (41)|Describe                 |DelegationToken Describing delegation tokens has special rules, for this please see the Authentication using Delegation Tokens section.
DELETE_GROUPS (42)      |Delete Group	                |
ELECT_PREFERRED_LEADERS (43)|ClusterAction Cluster      |
INCREMENTAL_ALTER_CONFIGS (44)|AlterConfigs Cluster     |  如果集群配置被更改，将会检查集群层面的权限
INCREMENTAL_ALTER_CONFIGS (44)|AlterConfigs Topic       | 如果topic配置被更改，将会检查topic层面的权限
ALTER_PARTITION_REASSIGNMENTS (45)|Alter Cluster	    |
LIST_PARTITION_REASSIGNMENTS (46)|Describe Cluster	    |
OFFSET_DELETE (47)              |Delete Group	        |
OFFSET_DELETE (47)              |Read Topic             |


