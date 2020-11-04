# 操作系统调优

## File descriptor limits

at least (number_of_partitions)*(partition_size/segment_size)，recommend at least 100000

## Max socket buffer size

can be increased to enable high-performance data transfer between data centers 

http://www.psc.edu/index.php/networking/641-tcp-tune

## vm.max_map_count

vm.max_map_count is somewhere around 65535.
Each log segment, requires a pair of index/timeindex files, and each of these files consumes 1 map area. 
each log segment uses 2 map areas.
creating 50000 partitions on a broker will result allocation of 100000 map areas and likely cause broker crash with OutOfMemoryError (Map failed) on a system with default vm.max_map_count.

## Disk

1. recommend using multiple drives to get good throughput
2. not sharing the same drives used for Kafka data with application logs or other OS filesystem activity to ensure good latency. 
3. multiple data directories partitions will be assigned round-robin to data directories.
4. If data is not well balanced among partitions this can lead to load imbalance between disks.

## 应用和OS的flush刷盘行为

Kafka always immediately writes all data to the filesystem and supports the ability to configure the flush policy that controls when data is forced out of the OS cache and onto disk using the flush. 

after a period of time or after a certain number of messages has been written.

Kafka must eventually call fsync to know that data was flushed. 

Kafka not require syncing data to disk, as a failed node will always recover from its replicas.

We recommend using the default flush settings which disable application fsync entirely.
我们建议使用默认的刷新设置来完全禁用应用程序fsync。

flush done by the OS and Kafka's own background flush. no knobs to tune, great throughput and latency, and full recovery guarantees. 


In general you don't need to do any low-level tuning of the filesystem

The flushing of data is done by a set of background threads called pdflush (or in post 2.6.32 kernels "flusher threads").

好处
1. The I/O scheduler will batch together consecutive small writes into bigger physical writes which improves throughput.
2. The I/O scheduler will attempt to re-sequence writes to minimize movement of the disk head which improves throughput.
3. It automatically uses all the free memory on the machine

## FileSystem 文件系统

EXT4 and XFS. XFS recommend

## 文件系统调优

1. noatime ：last access time

#### XFS
1. largeio: had minimal or no effect on performance.
2. nobarrier: For underlying devices that have battery-backed cache, this option can provide a little more performance by disabling periodic write flushes. However, if the underlying device is well-behaved, it will report to the filesystem that it does not require flushes, and this option will have no effect.

#### EXT4 
1. data=writeback: Ext4 defaults to data=ordered which puts a strong order on some writes. Kafka does not require this ordering as it does very paranoid data recovery on all unflushed log. This setting removes the ordering constraint and seems to significantly reduce latency.
2. Disabling journaling: Journaling is a tradeoff: it makes reboots faster after server crashes but it introduces a great deal of additional locking which adds variance to write performance. Those who don't care about reboot time and want to reduce a major source of write latency spikes can turn off journaling entirely.
3. commit=num_secs: This tunes the frequency with which ext4 commits to its metadata journal. Setting this to a lower value reduces the loss of unflushed data during a crash. Setting this to a higher value will improve throughput.
4. nobh: This setting controls additional ordering guarantees when using data=writeback mode. This should be safe with Kafka as we do not depend on write ordering and improves throughput and latency.
5. delalloc: Delayed allocation means that the filesystem avoid allocating any blocks until the physical write occurs. This allows ext4 to allocate a large extent instead of smaller pages and helps ensure the data is written sequentially. This feature is great for throughput. It does seem to involve some locking in the filesystem which adds a bit of latency variance.