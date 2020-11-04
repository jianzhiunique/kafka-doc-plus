# zookeeper安装

建议将zookeeper集群与Kafka集群分开部署，防止资源争抢。这里假设你已经阅读过相关文档。

### 下载与准备
```
tar -zxvf apache-zookeeper-3.5.8-bin.tar.gz
mv apache-zookeeper-3.5.8-bin /usr/local/zookeeper
# zookeeper的数据将放在这里
mkdir -p /data/zookeeper
# 程序的日志将放在这里
mkdir -p /home/logs/
```

### 修改所有节点的配置文件
```
/usr/local/zookeeper/conf/zoo.cfg

dataDir=/data/zookeeper
autopurge.snapRetainCount=3
autopurge.purgeInterval=1
clientPort=2181
maxClientCnxns=0
initLimit=10
syncLimit=5
server.1=10.0.0.1:2888:3888
server.2=10.0.0.2:2888:3888
server.3=10.0.0.3:2888:3888
```

### 配置所有节点的myid
```
echo 1 > /data/zookeeper/myid
```

### 配置所有节点的启动脚本
```
/etc/init.d/zookeeper

#!/bin/bash
export PATH=/usr/bin:$PATH
### BEGIN INIT INFO
# Provides:          baojizhong
# Required-Start:    $local_fs $network $named $time $syslog
# Required-Stop:     $local_fs $network $named $time $syslog
# Default-Start:     3 4 5
# Default-Stop:      0 1 2 6
# Description:       zookeeper
### END INIT INFO
case $1 in
          start)
              ZOO_LOG_DIR="/home/logs/zookeeper" ZOO_LOG4J_PROP="INFO,ROLLINGFILE"  /usr/local/zookeeper/bin/zkServer.sh start
              ;;
          stop)
              /usr/local/zookeeper/bin/zkServer.sh stop
              ;;
          status)
              /usr/local/zookeeper/bin/zkServer.sh status
              ;;
          restart)
              ZOO_LOG_DIR="/home/logs/zookeeper" ZOO_LOG4J_PROP="INFO,ROLLINGFILE" /usr/local/zookeeper/bin/zkServer.sh restart
              ;;
          *)
              echo "$0 start|stop|status|restart"
              ;;
esac
```
```
chmod +x /etc/init.d/zookeeper
```

### 启动所有节点的zookeeper并验证
```
service zookeeper start
service zookeeper status
```

####  维护文档
https://zookeeper.apache.org/doc/r3.4.10/zookeeperAdmin.html#sc_administering