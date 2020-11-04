# Kafka Zookeeper相关调优
Kafka使用Zookeeper存储有关主题、分区、代理和系统协调（如成员状态）的元数据信息。

Zookeeper的不可用或缓慢使Kafka集群不稳定，Kafka代理无法自动从中恢复。

建议使用专门为Kafka提供的3-5台机器Zookeeper集成（应用程序的同一位置可能会导致不必要的服务干扰）。

#### zookeeper.session.timeout.ms
指定Zookeeper在认为客户端（Kafka代理）不可用之前应等待心跳消息多长时间。

如果发生这种情况，将重新分配代理所拥有的partition leadership的元数据信息。

如果此设置太高，则系统可能需要很长时间才能检测到代理宕机。
如果设置得太小，可能会导致频繁的leadership reassignments。

#### jute.maxbuffer
控制znode可以包含的数据的最大大小。对于生产环境，默认值1MB可能要增大。

#### maxClientCnxns
有些情况下Zookeeper可能需要更多的连接。

#### 停用旧消费者客户端

旧的Kafka消费者在Zookeeper中存储消费者偏移提交（已弃用）。

建议使用将偏移量存储在内部Kafka主题中的新消费者，因为这样可以减少Zookeeper的负载。