# zookeeper监控

## 监控方式
#### 打开文件句柄的数量
应在系统范围内以及运行ZooKeeper进程的用户中完成。应考虑最大允许数量的打开文件句柄。

ZooKeeper经常打开和关闭连接，并且需要可用的文件句柄池供您选择。

#### 网络带宽使用率
因为ZooKeeper跟踪状态，所以它对由网络延迟引起的超时很敏感。如果网络带宽饱和，您可能很难用客户端会话解释超时，这会使Kafka群集的可靠性降低。

#### “Four Letter Words” 

https://zookeeper.apache.org/doc/r3.4.10/zookeeperAdmin.html#sc_zkCommands

```
echo stat | nc localhost 2181
# 如果报错：
You should monitor the STAT and MNTR four letter words. 
# 编辑zookeeper启动脚本，找到这个位置

else
    echo "JMX disabled by user request" >&2
    ZOOMAIN="org.apache.zookeeper.server.quorum.QuorumPeerMain" # 注意找到这个信息
fi

# 在这里添加一行
ZOOMAIN="-Dzookeeper.4lw.commands.whitelist=* ${ZOOMAIN}"
```

#### JMX Monitoring

```
NumAliveConnections - make sure you are not close to maximum as set with maxClientCnxns
OutstandingRequests - should be below 10 in general
AvgRequestLatency - target below 10 ms
HeapMemoryUsage (Java built-in) - should be relatively flat and well below max heap size
```