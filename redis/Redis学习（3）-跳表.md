## 跳表（skiplist）是什么







## 单链表的问题









## 跳表查找的时间复杂度







## 跳表查找的空间复杂度









## 跳表插入数据和删除数据







## 总结







## Redis为什么选择跳表来实现红黑树









## 跳表的其他使用场景

HBase 内部使用跳表实现，HBase属于LSM Tree结构的数据库，LSM Tree特点：实时写入的数据先写到内存，内存达到阈值再往磁盘flush，这个时候会生成一些有序文件，而跳表天然石有序的，所有flush的时候效率非常高。

HBase 使用的是 java.util.concurrent 下的 ConcurrentSkipListMap()

Google 开源的 key/value 存储引擎 LevelDB 以及 Facebook 基于 LevelDB 优化的 RocksDB 都是 LSM Tree 结构的数据库，他们内部的 MemTable 都是使用了跳表这种数据结构















