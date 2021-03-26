# Kafka多数据中心

## 历史

Kafka早期版本提供了MirrorMaker，可以将Topic数据从一个集群复制到另一个集群，内部原理是消费者加生产者的模式，可以用来做集群数据迁移之类的工作。

然而MirrorMaker固然是简单的，并且功能上有些缺陷，用户对此不满意。

Linkedin的Brooklin https://github.com/linkedin/Brooklin/ 、Uber的uReplicator https://github.com/uber/uReplicator 就是代表产品，
他们是早期MirrorMaker的探索者，但都发现MirrorMaker存在很大的问题，于是他们用自己的方式分布式地管理了MirrorMaker的实例，并取得了很大的成功。

Confluent公司默默地推出了他们的商业版本，Confluent Replicator，需付费（并且hen），

MirrorMaker1，Kafka的早期版本自带

Salesforce的Mirus，基于kafka connect

Confluent的Confluent Replicator 商业版本，

MirrorMaker2，Kafka 2.4版本引入，基于Kafka connect
