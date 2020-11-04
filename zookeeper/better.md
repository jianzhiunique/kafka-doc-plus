# zookeeper优化

#### 物理/硬件/网络布局的冗余
尽量不要将它们放在同一个机架中，适当的硬件，尽量保持冗余的电源和网络路径等。一个典型的ZooKeeper集成有5个或7个服务器，分别允许2个和3个服务器停机。

#### I/O隔离
如果你有很多写类型的流量，你几乎肯定会想要一个专用磁盘组上的事务日志。并发写入会显著影响性能。理想情况下应该写在与事务日志分离的磁盘组上。

#### 应用隔离
让zookeeper独享资源。

#### 小心使用虚拟化(虚拟机)
虚拟化：将服务器放置在不同的可用区域中，以避免相关的崩溃，并确保可用的存储系统可以满足事务日志记录和ZooKeeper快照的要求。

#### JVM 
确保给它足够的堆空间，我们通常用3-5G来运行它们，但这主要是由于我们这里的数据集大小。
建议堆大小为1 GB，监视堆使用情况，以确保垃圾收集不会造成任何延迟。

#### 快照
较大的快照会影响恢复时间。增加initLimit参数

#### 不要过度组建集群
节点不能太多，因为有内部通信（风暴）。

从三到五台服务器的小集合开始，然后仅在真正必要时（或在容错时要求）进行扩展。

#### 让ZooKeeper系统尽可能的小

尽量不在配置或应用程序布局上做任何花哨的事情，并尽可能保持其独立性。

#### 不使用操作系统打包的版本
因为它有一种倾向，试图把东西放在操作系统标准层次结构中，这可能是“混乱的”，因为需要更好的方式来描述它。

#### 内存
内存随znode的增加而增加，znode创建的主要驱动因素是集群中的分区数量

最少应有4 GB的RAM供ZooKeeper使用

#### 不使用swap
ZooKeeper对swap很敏感，任何运行ZooKeeper服务器的主机都应避免swap。

#### CPU
ZooKeeper不会大量消耗CPU资源

但是，如果共享ZooKeeper，其他进程竞争CPU，应考虑提供专用的CPU内核以确保上下文切换不成问题。

#### 磁盘SSD
磁盘性能对于维持健康的ZooKeeper群集至关重要。
强烈建议使用固态驱动器（SSD），因为ZooKeeper必须具有低延迟的磁盘写操作才能达到最佳性能。
对于生产部署，建议在每个ZooKeeper服务器上使用至少64 GB的专用SSD。
可以使用autopurge.purgeInterval和 autopurge.snapRetainCount自动清除ZooKeeper数据并降低维护开销。

#### 保持简单

ZooKeeper拥有重要的数据，稳定性和持久性更重要。

#### ZooKeeper不需要对大多数部署进行配置调整
```
clientPort
这是ZooKeeper客户端将监听的端口。代理将在此处连接到ZooKeeper。通常将其设置为2181。
dataDir
ZooKeeper内存数据库快照和数据库dataLogDir更新的事务日志（除非在其中指定）的目录。该位置应该是专用磁盘，最好是SSD。
dataLogDir
事务日志写入的位置。如果未指定此选项，则将日志写入dataDir。通过指定此选项，您可以使用专用的日志设备，并有助于避免日志记录和快照之间的竞争。
tickTime
ZooKeeper的时间单位转换为毫秒。这将控制所有ZooKeeper与时间有关的操作。特别是用于心跳和超时。请注意，最小会话超时将是两个刻度。
maxClientCnxns
ZooKeeper服务器允许的最大客户端连接数。为避免耗尽允许的连接，请将其设置为0（无限制）。
autopurge.snapRetainCount
启用后，ZooKeeper自动清除功能会在dataDir和dataLogDir中分别保留autopurge.snapRetainCount最新快照和相应的事务日志，并删除其余快照。默认值：3
autopurge.purgeInterval
必须触发清除任务的时间间隔（以小时为单位）。设置为正整数（1或更大）以启用自动清除。


示例配置
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=x.x.x.x:2888:3888
server.2=x.x.x.x:2888:3888
server.3=x.x.x.x:2888:3888
autopurge.snapRetainCount=3
autopurge.purgeInterval=24
```
