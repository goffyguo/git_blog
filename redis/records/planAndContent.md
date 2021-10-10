## 理论

### Redis + Redis 数据结构底层源码

1. Redis 是字典数据库（k/v 键值对），具体原理
2. 五大数据结构底层 c 语言源码分析
3. skiplist 跳表学习

### Redis 缓存及面临的问题

1. Redis 缓存
2. 缓存雪崩
3. 缓存穿透
4. 缓存击穿
5. 总结

### Redis 缓存过期淘汰策略

1. Redis 缓存淘汰策略
2. LRU？手写LRU（对照HashMap）
3. Redis 中如何实现 LRU

### 布隆过滤器

Bloom filter是什么、特点、作用、原理、优缺点

布谷鸟过滤器

### Redis 分布式锁

1. 为什么需要分布式锁+分布式锁实现
2. 避免死锁以及锁被别人释放怎么办？
3. 锁过期实践如何评估
4. RedLock 是否安全以及争议点
5. Zookeeper 的分布式如何实现，以及和 Redis 实现之间的区别
6. 对与分布式锁的理解

### Redis 高性能设计分析

I/O 多路复用解析

### Redis 与 Mysql 双写一致性问题

1. 工作原理
2. canal 是什么，能干什么，解决了什么问题
3. 数据库和缓存一致性的几种实现方式

### Redis 集群

## 实践

### 数据结构

八大数据类型（五种常用+三种新加）

- String
- hash
- list
- set
- Zset
- bitmap
- GEO
- hyperloglog

### 场景实践

1. 微信文章阅读量统计（小厂可做）
2. 日活统计
3. 基于地图位置的酒店推送服务
4. 好友功能+共同关注列表+关注取关
5. 签到及统计连续签到次数
6. 积分新增+排行榜
7. Feed 功能
8. 流媒体、淘宝购物分享短视频链接推广
9. 微信抢红包
10. 优惠券抢购（分布式锁）

### 分布式锁

## 其他

Redis 实践+使用规范

Redis 变慢的排查方案

Redis 存在的坑