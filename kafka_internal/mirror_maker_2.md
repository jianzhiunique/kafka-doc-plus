# MirrorMaker2源码

## Heartbeat心跳主题

```
MirrorMaker2在每个source集群产生一个心跳topic（heartbeats），然后此topic会通过connector复制到下游；
下游消费者可以根据这个topic来验证：
a) connector正在运行
b) 相关的上游集群是可用的

heartbeats将会通过connector传播，比如backup.us-west.us-east.heartbeat，为了避免产出循环，需要检测

心跳主题的schema如下：
target cluster (String): 心跳数据发送到的集群
source cluster (String): source集群，正在发送心跳数据的集群
timestamp (long): 心跳毫秒时间戳

对于A->B的链路来说，根据target上部署MM2的原则，MM2应该部署在B集群，source=A，target=B
所以心跳topic应该建立在A集群
在非connect集群部署的模式下，B集群上的MM2会向A发送心跳数据，但在connect集群模式部署时不生效
因为实践中发现connect集群下，connector产生的心跳数据一定会写到运行connect集群的kafka上，也就是在B机房配置的会写在B集群
所以需要在A机房的connect集群上部署心跳connector，这样上述a)的效果可能达不到
当在B机房配置heartbeat connector时，相当于是为A进行配置，在B生成心跳主题, 按照schema的解释，target应该是B，source应该是A
而B本身的MirrorSourceConnector工作应该是A->B的链路复制，source是A，target是B
所以在同一个机房配置多个connector时，source和target应该都是一样的
```

#### 如何避免无限复制

heartbeat connector只是生成heartbeats主题，以及向主题生成消息，传播则是通过MirrorSourceConnector完成的

```
    //MirrorSourceConnector
    boolean shouldReplicateTopic(String topic) {
        return (topicFilter.shouldReplicateTopic(topic) || isHeartbeatTopic(topic))
            && !replicationPolicy.isInternalTopic(topic) && !isCycle(topic);
    }
    
    // topicFilter.shouldReplicateTopic(topic) || isHeartbeatTopic(topic) 1.满足topic规则的topic以及心跳topic，其中topic规则好理解
    // !replicationPolicy.isInternalTopic(topic) 2. 并且不是内部主题
    // !isCycle(topic) 3. 并且不是循环
    
    //ReplicationPolicy
    // 是否是内部主题判断比较好理解
    default boolean isInternalTopic(String topic) {
        return topic.endsWith(".internal") || topic.endsWith("-internal") || topic.startsWith("__")
            || topic.startsWith(".");
    }
    
    //接下来是不好理解的
    //MirrorSourceConnector
    // 判断是否是心跳Topic，看原始topic（一层层去掉前缀）是否是heartbeats
    boolean isHeartbeatTopic(String topic) {
        return MirrorClientConfig.HEARTBEATS_TOPIC.equals(replicationPolicy.originalTopic(topic));
    }
    
    //ReplicationPolicy
    //获取原始Topic逻辑：topic的上游topic是null时，返回topic,否则将上游Topic继续递归，再次查看上游Topic的原始Topic
    default String originalTopic(String topic) {
        String upstream = upstreamTopic(topic);
        if (upstream == null) {
            return topic;
        } else {
            return originalTopic(upstream);
        }
    }
    
    //DefaultReplicationPolicy
    //获得上游topic全名逻辑：源头集群名称为null时，返回null；源头集群名称不为null时，截断源头集群名称，返回topic
    public String upstreamTopic(String topic) {
        String source = topicSource(topic);
        if (source == null) {
            return null;
        } else {
            return topic.substring(source.length() + separator.length());
        }
    }
    
    //DefaultReplicationPolicy
    //获得源头集群名称逻辑：使用分隔符-分割topic，如果分割后长度小于2，返回空；如果长度大于2， 返回前缀；即如果topic有源头，返回源头，否则返回空
    public String topicSource(String topic) {
        String[] parts = separatorPattern.split(topic);
        if (parts.length < 2) {
            // this is not a remote topic
            return null;
        } else {
            return parts[0];
        }
    }
    
    //MirrorSourceConnector
    // 判断是否是循环复制
    // 获取源头集群名称，如果集群源头=target名称（alias），那么是循环的，
    // 比如A机房的B->A链路，target为A, 
    // 假如B机房A->B复制时将A：topic -> B: A.topic，
    // A机房发现B机房A.topic，源头集群为A，target相同，判定这会导致循环，所以不复制
    boolean isCycle(String topic) {
        String source = replicationPolicy.topicSource(topic);
        if (source == null) {
            return false;
        } else if (source.equals(sourceAndTarget.target())) {
            return true;
        } else {
            return isCycle(replicationPolicy.upstreamTopic(topic));
        }
    }

   
    
```


