## Redis 的删除策略

- 立即删除
- 惰性删除
- 定期删除

### 立即删除

Redis 不可能时时刻刻遍历所有被设置了生存时间的 key，来检测数据是否已经到达过期时间，然后对它进行删除。立即删除能保证内存中数据的最大新鲜度，因为它保证过期键值会在过期后立马被删除，其所占用的内存也会随之释放。但是立即删除对cpu是最不友好的。因为删除会占用cpu的时间，如果刚好碰上cpu比较忙的时候，比如正在做交集或排序等计算的时候，就会给cpu造成额外的压力。

优点：保证内存中数据的最大新鲜度

缺点：产生大量的性能消耗，同时也会影响数据的读取操作

总结：**对CPU不友好，用处理器性能换取存储空间（拿时间换空间）**

### 惰性删除

数据到达过期时间，不做处理。等下次访问该数据时，如果未过期，返回数据；发现已经过期，删除，返回不存在。

缺点：对内存是最不友好的。

如果一个键已经过期，而这个键又仍然保留在Redis中，那么只要这个过期键不被删除，他所占用的内存就不会释放。

在使用惰性删除策略时，如果数据库中有非常多的过期键，而这些过期键又恰好没有被访问到的话，那么她们也许永远也不会被删除（除非用户手动执行FLUSHDB），理论上来说，我们将这种情况 看作是一种内存泄漏----无用的垃圾占用了大量的内存，而服务器却不会自己去释放他们。这对于运行状态非常依赖内存的Redis服务器来说，肯定不是什么好事。

总结：**对内存不友好，用存储空间换取处理器性能（拿空间换时间）**

### 定期删除

定期删除是前两种策略的折中策略：**每隔一段时间执行一次删除过期键操作**，并通过限制删除操作执行的时长和频率来减少删除操作对cpu的影响。

周期性轮询redis库中的时效性数据，采用随机抽取的策略，利用过期数据占比的方式控制删除频度 。

特点1：CPU性能占用设置有峰值，检测频度可自定义设置 

特点2：内存压力不是很大，长期占用内存的冷数据会被持续清理

总结：**周期性抽查存储空间 （随机抽查，重点抽查）** 

举例：redis默认每个100ms检查，是否有过期的key，有过期key则删除。 注意： redis不是每隔100ms将所有的key检查一次而是随机抽取进行检查( 如果每隔100ms,全部key进行检查，redis直接进去ICU )。因此，如果只采用定期删除策略，会导致很多key到时间没有删除。

定期删除策略的难点是确定删除操作执行的时长和频率：如果删除操作执行得太频繁，或者执行的时间太长，定期删除策略就会退化成立即删除策略，以至于将CPU时间过多地消耗在删除过期键上面。如果删除操作执行得太少，或者执行的时间太短，定期删除策略又会和惰性删除束略一样，出现浪费内存的情况。因此，如果采用定期删除策略的话，服务器必须根据情况，合理地设置删除操作的 执行时长和执行频率。

上面三种在极端情况下都会有问题：定期删除（从来没有被抽查到）；惰性删除（也从来没有被点中使用过），存在内存泄漏的风险。所以必须要有一个更好的兜底方案。

**Redis自身提供的三大删除策略，都不是最完美的，所有引入缓存淘汰策略。**

## Redis缓存淘汰策略

### 种类（Redis6.0.8）

1. volatile-lru：对所有设置了过期时间的key使用LRU算法进行删除
2. allkeys-lru：对所有key使用LRU算法进行删除
3. volatile-lfu：对所有设置了过期时间的key使用LFU算法进行删除
4. allkeys-lfu：对所有key使用LFU算法进行删除
5. volatile-random：对所有设置了过期时间的key随机删除
6. allkeys-random：对所有key随机删除
7. volatile-ttl：删除马上过期的key
8. noeviction：不会删除任何key（默认）

```shell
# volatile-lru -> Evict using approximated LRU, only keys with an expire set.
# allkeys-lru -> Evict any key using approximated LRU.
# volatile-lfu -> Evict using approximated LFU, only keys with an expire set.
# allkeys-lfu -> Evict any key using approximated LFU.
# volatile-random -> Remove a random key having an expire set.
# allkeys-random -> Remove a random key, any key.
# volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
# noeviction -> Don't evict anything, just return an error on write operations.
# The default is:
#
# maxmemory-policy noeviction（默认）
```

总结：（2 * 4 = 8）

1. 2个维度（过期键中筛选、所有键中筛选）
2. 4个方面（LRU、LFU、random、ttl）
3. 8个方案

### LRU

### LFU























