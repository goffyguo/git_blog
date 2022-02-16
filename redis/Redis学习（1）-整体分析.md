[TOC]



## 什么是 Redis

贴一段来自官网的介绍

> Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。它支持多种类型的数据结构，如字符串（strings），散列（hashes），列表（lists），集合（sets），有序集合（sorted sets）与范围查询，bitmaps，hyperloglogs 和地理空间（geospatial）索引半径查询。 Redis 内置了复制（replication），LUA脚本（Lua scripting），LRU驱动事件（LRU eviction），事务（transactions）和不同级别的磁盘持久化（persistence），并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

## Redis 的源码概要

Redis源码的官方仓库：https://github.com/redis/redis

源码地址：redis/src(version:6.2.6)

<img src="/Users/guogoffy/Library/Application Support/typora-user-images/image-20211008164019232.png" alt="image-20211008164019232" style="zoom:50%;" />

## Redis 源码的核心

### Redis 基本数据结构

- 简单动态字符串（sds.c）
- 整数集合（intset.c）
- 压缩列表（ziplist.c）
- 快速列表（quickest.c）
- 字典（dict.c）
- Streams 的底层实现结构（listpack.c和rax.c）

### Redis 数据类型的底层实现

贴一段Redis源码的说明：

> `t_hash.c`, `t_list.c`, `t_set.c`, `t_string.c`, `t_zset.c` and `t_stream.c` contains the implementation of the Redis data types. They implement both an API to access a given data type, and the client command implementations for these data types.

- Redis对象（Object.c）
- 字符串（t_string.c）
- 列表（t_list.c）
- 字典（t_hash.c）
- 集合有有序集合（t_set.c和t_zset.c）
- 数据流（t_stream.c）

### Redis 数据库的实现

- 数据库的底层实现（db.c）
- 持久化的实现（rdb.c和aof.c）

### Redis 服务端与客户端实现

- 事件驱动（ae.c和ae_poll.c）
- 网络连接（anet.c和networking.c）
- 服务端程序（server.c）
- 客户端程序（redic_cli.c）

### 其他

- 主从复制（replication.c）
- 哨兵（sentinel.c）
- 集群（cluster.c）
- 其他数据结构（如hyperloglog.c、geo.c）
- 其他功能（pub/sub、Lua脚本）

## Redis 为什么是字典数据库

因为我是写Java的，所以以Java对照着学习Redis

在Java中，一切皆对象（object），对应的在Redis里面，一切皆字典（dict），也就是k/v键值对。它的落地实现就是RedisObject。

这样的类比并不严谨，只是为了对redis先有个基本的概念。切勿深究。

一言以蔽之：Redis是key-value存储系统，其中key一般是字符串，value类型则为redis对象（redisObject）

<img src="/Users/guogoffy/Library/Application Support/typora-user-images/image-20211011152915392.png" alt="image-20211011152915392" style="zoom:50%;" />



可以这么理解：最右侧的String、Hash、List、Set、Zset是对外提供的，提供给我们开发人员用的，但事实上对于Redis本身来说，这些都是RedisObject，所以可以总结下：Redis为什么是key-value存储系统，其实就是说字符串是key，value是高度抽取的对象。可以和Java类比起来，JVM不在乎String、List还是Map，他只认Object。



### 六大数据类型（粗分）

传统的五大类型：String、Hash、List、Set、Zset + 新添加的Bitmap(String)、HyperLogLog(String)、GEO(Zset)。

### C语言**struct**结构体学习

**struct 结构体**：

为了定义结构，您必须使用 **struct** 语句。struct 语句定义了一个包含多个成员的新的数据类型，struct 语句的格式如下：

```c
struct tag { 
    member-list
    member-list 
    member-list  
    ...
} variable-list ;
```

**tag** 是结构体标签。

**member-list** 是标准的变量定义，比如 int i; 或者 float f，或者其他有效的变量定义。

**variable-list** 结构变量，定义在结构的末尾，最后一个分号之前，您可以指定一个或多个结构变量。下面是声明 Book 结构的方式：

```c
struct Books
{
   char  title[50];
   char  author[50];
   char  subject[100];
   int   book_id;
} book;
```

**C typedef:**

C 语言提供了 **typedef** 关键字，您可以使用它来为类型取一个新的名字。例如：

```c
typedef struct Books
{
  char  title[50];
  char  author[50];
  char  subject[100];
  int   book_id;
}Books;
```

现在，就可以直接只用Books来定义Books类型的变量，而不需要使用struct关键字，下面是实例：

```c
Books Book1,Book2;
```

差不多可以看懂这个结构体就可以了。

看完了这个我们再返回去从编码的角度说一下Redis为什么是k/v数据库：

<img src="/Users/guogoffy/Library/Application Support/typora-user-images/image-20211011160841323.png" alt="image-20211011160841323" style="zoom:50%;" />



从这个代码中就可以看出Redis每一个键值对都会有一个dicEntry（字典实体），也就是说在C语言层面也就是一个dicEntry。

> void *key 可以看成是key, union {... } v 可以看成是v,联合在一起就是dicEntry，也就是字典实体，也就是k/v键值对。

再可以直白的理解就是 dicEntry = string + RedisObject。上面也说过了，一言以蔽之：Redis是key-value存储系统，其中key一般是字符串，value类型则为redis对象（redisObject）。

## 总结

第一篇文章简单介绍了下Redis源码的核心以及Redis为什么是字典数据库，后面一步一步深入Redis，体会Redis设计精髓和伟大。

[深入学习Redis（1）：Redis内存模型 - 编程迷思 - 博客园 (cnblogs.com)](https://www.cnblogs.com/kismetv/p/8654978.html)







