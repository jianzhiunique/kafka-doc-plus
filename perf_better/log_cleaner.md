# 日志清理器

这里是与日志compaction有关的配置

#### log.cleaner.threads
控制日志compaction的后台线程数，增大此配置可能增加compaction的性能，以增加I/O活动为代价。

#### log.cleaner.io.max.bytes.per.second
限制日志清理程序的I/O活动，以便其读和写的总和平均小于此值。

#### log.cleaner.dedupe.buffer.size
指定所有清理线程上用于日志compaction的内存

#### log.cleaner.io.buffer.size
指定所有清理线程用于日志compaction的总内存

#### log.cleaner.min.compaction.lag.ms
控制消息可以在未被压缩状态下待多久

#### log.cleaner.io.buffer.load.factor
控制重复数据消除缓冲区的日志清理器的加载因子。增加此值允许系统一次清理更多日志，但会增加哈希冲突。

#### log.cleaner.backoff.ms
如果没有要压缩的日志，控制下次检查前等待多久