## 穿透

### 是什么

某个请求去查询一条记录，先去Redis查询发现没有，后去mysql查询也没有，也就是先Redis后mysql发现都查询不到该条记录，但是请求每次都会打到mysql上面去，导致后台数据库压力暴增，这种现象称为缓存穿透。说直白点就是查询一个一定不存在的数据。

### 危害

第一次来查询后，一般业务都会有回写Redis机制，第二次来查询的时候Redis就有了，偶尔出现穿透现象一般情况也无关紧要

### 解决

- 空对象缓存或者缺省值
  - 一旦发生缓存穿透，可以针对查询的数据，在Redis中缓存一个空值或是和业务层协商确定的缺省值（例如，库存的缺省值可以设为0）。紧接着，应用发送到后续请求再进行查询时，就可以直接从Redis中读取空值或缺省值，返回给业务应用，避免了把大量请求发送给数据库处理，保持了数据库的正常运行。
  - 黑客或者恶意攻击，这类的一般情况下会拿一个不存在的id去查询数据，会产生大量的请求到数据库去查询，可能导致数据库由于压力过大而宕掉。
- Google 布隆过滤器 Guava 解决缓存穿透
- Redis 布隆过滤器解决缓存穿透



## 击穿

### 是什么

大量的请求同时查询一个key时，此时这个key正好失效了，就会导致大量的请求都打到数据库上面去。简单来说就是热点key突然失效，暴打mysql。

### 危害

造成某一时刻mysql请求量过大，压力剧增。

### 解决

- 对于频繁访问的热点key，干脆不需要设置过期时间，也就是永不过期

  -  从redis上看，确实没有设置过期时间，这就保证了，不会出现热点key过期问题，也就是“物理”不过期。
  - 从功能上看，如果不过期，那不就成静态的了吗？所以我们把过期时间存在key对应的value里，如果发现要过期了，通过一个后台的异步线程进行缓存的构建，也就是“逻辑”过期
  - 从实战看，这种方法对于性能非常友好，唯一不足的就是构建缓存时候，其余线程(非构建缓存的线程)可能访问的是老数据，但是对于一般的互联网功能来说这个还是可以忍受。

- 互斥独占锁（mutex key）

  业界比较常用的做法，是使用mutex。简单地来说，就是在缓存失效的时候（判断拿出来的值为空），不是立即去load db，而是先使用缓存工具的某些带成功操作返回值的操作（比如Redis的SETNX或者Memcache的ADD）去set一个mutex key，当操作返回成功时，再进行load db的操作并回设缓存；否则，就重试整个get缓存的方法。

  示例代码：

  ```java
  //2.6.1前单机版本锁
  String get(String key) {  
     String value = redis.get(key);  
     if (value  == null) {  
      if (redis.setnx(key_mutex, "1")) {  
          // 3 min timeout to avoid mutex holder crash  
          redis.expire(key_mutex, 3 * 60)  
          value = db.get(key);  
          redis.set(key, value);  
          redis.delete(key_mutex);  
      } else {  
          //其他线程休息50毫秒后重试  
          Thread.sleep(50);  
          get(key);  
      }  
    }  
  }
  //最新版本代码
  public String get(key) {
        String value = redis.get(key);
        if (value == null) { //代表缓存值过期
            //设置3min的超时，防止del操作失败的时候，下次缓存过期一直不能load db
  		  if (redis.setnx(key_mutex, 1, 3 * 60) == 1) {  //代表设置成功
                value = db.get(key);
                redis.set(key, value, expire_secs);
                redis.del(key_mutex);
                } else {  //这个时候代表同时候的其他线程已经load db并回设到缓存了，这时候重试获取缓存值即可
                        sleep(50);
                        get(key);  //重试
                }
            } else {
                return value;      
            }
   }
  ```

## 雪崩

### 是什么

设置缓存时采用了相同的过期时间，导致缓存在某一时刻同时失效，请求全部转发到DB，DB瞬时压力过重雪崩

### 危害

缓存失效时的雪崩效应对底层系统的冲击非常可怕。因为这个时候Redis全盘崩溃。

### 解决

- 将缓存失效时间分散开
- Redis 集群实现高可用（主从+哨兵、Redis Cluster）
- ehcache 本地缓存 + Hystrix 或者阿里 sentinel 限流降级
- 开启Redis持久化机制 aof/rdb，尽快恢复缓存集群







## 总结



