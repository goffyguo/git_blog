[TOC]



## Redis 基本数据结构（骨架）

- 简单动态字符串（sds.c）
- 整数集合（intset.c）
- 压缩列表（zippiest.c）
- 快速链表（quicklist.c）
- 字典（dict.c）
- Streams（listpack.c、rax.c）

## Redis基本数据类型

### Redis 对象 Object

从一个简单的例子说起：set hello word

```shell
# guogoffy @ Goffys-MacBook in ~ [10:17:32] 
$ redis-cli
127.0.0.1:6379> set hello word
OK
127.0.0.1:6379> get hello
"word"
127.0.0.1:6379> 
```

Redis 的每一个键值都会有一个 dictEntry（源码位置：dict.h），里面指向了 key 和 value 的指针，next 指向了下一个 dictEntry。

```c++
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as returned
                                 * by dictType's dictEntryMetadataBytes(). */
} dictEntry;
```

key是字符串，没有返回值，值得注意的是Redis没有直接使用c的字符数组，而是存储在Redis自定义的SDS中（后面会专门说SDS）。

value既不是直接作为字符串存储，也不是直接存储在SDS中，而是存储在redisObject中。实际上，五种常用的数据类型的任何一种，都是通过redisObject来存储的。

大概的图示是这样的：

![image-20211013105023932](../imgs/image-20211013105023932.png)

然后再分别查看一下这个key的类型、编码、debug结构：

```shell
127.0.0.1:6379> get hello
"word"
127.0.0.1:6379> type hello
string
127.0.0.1:6379> object encoding hello
"embstr"
127.0.0.1:6379> debug object hello
Value at:0x12f70a380 refcount:1 encoding:embstr serializedlength:5 lru:6703550 lru_seconds_idle:30
```

可以发现，类型是string，但是编码实际上是embstr，那基本可以推出Redis底层自己维护了一套编码规则，debug的时候发现有好几个属性（Value、refcount、encoding、serializedlength、lru和lru_seconds_idle），带着这几个属性去查看源码是这么写的（server.h）。

```c++
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

可以看出，为了便于操作，Redis 采用 redisObject 结构来统一五种不同的数据类型，这样所有的数据类型就都可以以相同的形式在函数间传递而不用使用特定的类型结构。同时，为了识别不同的数据类型，redisObject中定义了type和encoding字段对不同的数据类型加以区别，简单的说，`redisObject就是string、hash、list、set、zset的父类`，可以在函数间传递是隐藏具体的类型信息，所有就抽象出了一个redisObject来达到同样的目的。

redisObject属性的详细解释：

- type：4位，对象的类型，包括：OBJ_STRING、OBJ_LIST、OBJ_SET、OBJ_ZSET、OBJ_HASH

- encoding：表示该类型的物理编码方式，同一种数据类型可能有不同的编码方式。（比如string就提供了3种：int、embstr、raw）

  ```c
  /* Objects encoding. Some kind of objects like Strings and Hashes can be
   * internally represented in multiple ways. The 'encoding' field of the object
   * is set to one of this fields for this object. */
  #define OBJ_ENCODING_RAW 0     /* Raw representation */
  #define OBJ_ENCODING_INT 1     /* Encoded as integer */
  #define OBJ_ENCODING_HT 2      /* Encoded as hash table */
  #define OBJ_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
  #define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
  #define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
  #define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
  #define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
  #define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
  #define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
  #define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */	
  #define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
  ```

- lru:LRU_BITS：4位，对象最后一次被命令程序访问的时间，与内存回收有关

- refcount：引用计数，当refcount为0的时候，表示该对象已经不被任何对象饮用，则可以进行垃圾回收了

- *ptr：指向真正的底层数据结构的指针

编码体会：

```shell
127.0.0.1:6379> set k1 123
OK
127.0.0.1:6379> get k1
"123"
127.0.0.1:6379> object encoding k1
"int"
127.0.0.1:6379> debug object k1
Value at:0x14f6124b0 refcount:2147483647 encoding:int serializedlength:2 lru:6706100 lru_seconds_idle:64
127.0.0.1:6379> get k1
"123"
127.0.0.1:6379> debug object k1
Value at:0x14f6124b0 refcount:2147483647 encoding:int serializedlength:2 lru:6706167 lru_seconds_idle:2
127.0.0.1:6379> debug object k1
Value at:0x14f6124b0 refcount:2147483647 encoding:int serializedlength:2 lru:6706167 lru_seconds_idle:17
127.0.0.1:6379> get k1
"123"
127.0.0.1:6379> debug object k1
Value at:0x14f6124b0 refcount:2147483647 encoding:int serializedlength:2 lru:6706187 lru_seconds_idle:4
127.0.0.1:6379> set k1 abc
OK
127.0.0.1:6379> object encoding k1
"embstr"
127.0.0.1:6379> set k1 aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
OK
127.0.0.1:6379> object encoding k1
"raw"
127.0.0.1:6379> 
```

所以到现在我们就可以差不多得出一个结论，对于底层C语言服务器来说，它在意的就是dictEntry，而对于我们redis程序员来说，我们只关心redisObject（set k1 123）。

### 基本数据类型

数据类型与数据结构的关系：

- **String**：动态字符串
- **List**：双端链表+压缩列表ZipList
- **Hash**：哈希表+压缩列表ZipList
- **Set**：哈希表+整数数组
- **Sorted Set**：跳表+压缩列表ZipList

#### 字符串（t_string.c）

1. 编码格式

   - **int**

     保存long型（长整型）的64位（8个字节）有符号整数，最大值2^63^-1，最小值-2^63^，主要使用在比较大整数的系统上，默认值是0L；还有一点需要注意的是，只有整数才会使用int，如果是浮点数，Redis内部会先将浮点数转化成字符串值，然后在保存。

   - **embstr**

     代表embstr格式的SDS（Simple Dynamic String，简单动态字符串）表示嵌入式的String，保存长度小于等于44字节的字符串

   - **raw**

     保存长度大于44字节的字符串

2. 编码案例测试

   ```shell
   127.0.0.1:6379> set k1 123 # 普通数字
   OK
   127.0.0.1:6379> object encoding k1
   "int"
   127.0.0.1:6379> set k1 12345678912345678911 # 长度大于19的数
   OK
   127.0.0.1:6379> object encoding k1
   "embstr"
   127.0.0.1:6379> set k1 abc # 普通字符串
   OK
   127.0.0.1:6379> object encoding k1
   "embstr"
   127.0.0.1:6379> set k1 aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa # 长度小于等于44的字符串
   OK
   127.0.0.1:6379> object encoding k1
   "embstr"
   127.0.0.1:6379> set k1 aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa # 长度大于等于44的字符串
   OK
   127.0.0.1:6379> object encoding k1
   "raw"
   127.0.0.1:6379> 
   ```

3. C语言中字符串的展现

   首先需要明确的是，严格来说，C语言中并没有字符串这样的概念，它是将字符串作为字符数组来处理的，默认使用`\0`放在结尾来表示一个字符串。类似于这样：

   ![image-20211014104233083](../imgs/image-20211014104233083.png)

   如果需要获取Redis这个字符串的长度，需要从头开始遍历，直到遇到 `\0`为止。

4. SDS简单动态字符串

   然而在Redis没有直接复用C语言的字符串，而是新建了一个属于自己的结构——SDS

   在Redis数据库里，包含字符串值的键值对都是由SDS实现的（Redis中所有的键都是由字符串对象实现的，即底层是由SDS实现，所有的值对象中包含的字符串对象底层也是由SDS实现的）

   我们看源码一个一个分析（sds.h）：

   ```c
   typedef char *sds;
   
   /* Note: sdshdr5 is never used, we just access the flags byte directly.
    * However is here to document the layout of type 5 SDS strings. */
   struct __attribute__ ((__packed__)) sdshdr5 {
       unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
       char buf[];
   };
   struct __attribute__ ((__packed__)) sdshdr8 {
       uint8_t len; /* used */
       uint8_t alloc; /* excluding the header and null terminator */
       unsigned char flags; /* 3 lsb of type, 5 unused bits */
       char buf[];
   };
   struct __attribute__ ((__packed__)) sdshdr16 {
       uint16_t len; /* used */
       uint16_t alloc; /* excluding the header and null terminator */
       unsigned char flags; /* 3 lsb of type, 5 unused bits */
       char buf[];
   };
   struct __attribute__ ((__packed__)) sdshdr32 {
       uint32_t len; /* used */
       uint32_t alloc; /* excluding the header and null terminator */
       unsigned char flags; /* 3 lsb of type, 5 unused bits */
       char buf[];
   };
   struct __attribute__ ((__packed__)) sdshdr64 {
       uint64_t len; /* used */
       uint64_t alloc; /* excluding the header and null terminator */
       unsigned char flags; /* 3 lsb of type, 5 unused bits */
       char buf[];
   };
   ```

   可以发现Redis中字符串的实现，SDS有多种结构：

   sdshdr5（2^5^）、sdshdr8（2^8^）、sdshdr16（2^16^）、sdshdr32（2^32^）、sdshdr64（2^64^），这个结构用于存储不同长度的字符串

   ```c
   struct __attribute__ ((__packed__)) sdshdr64 {
     	/* len 表示SDS的长度，有这个属性在获取字符串长度的时候可以在O(1)的情况下拿到，而不是像C那样需要遍历一遍自字符串 */
       uint64_t len; /* used */ 
     	/* alloc 用来表示计算free字符串已经分配的未使用的空间，有了这个就可以引入预分配空间的算法了，就可以不用考虑内存分配 */
       uint64_t alloc; /* excluding the header and null terminator */
     	/*  */
       unsigned char flags; /* 3 lsb of type, 5 unused bits */
     	/* 表示真正存储数据的字符串数组 */
       char buf[];
   };
   ```

5. Redis为什么要重新设计一个SDS

   前面说了如果沿用C语言的字符串设计，那么会有一些缺点，所以我们对比下C语言中字符串的设计和Redis中字符串的设计之间的差异（强调SDS的设计的好处）

   |                | C 语言                                                       | SDS                                                          |
   | -------------- | ------------------------------------------------------------ | :----------------------------------------------------------- |
   | 字符串长度处理 | 需要从头开始遍历。直到遇到`\0`为止，时间复杂度O(n)           | 记录当前字符串的长度，直接读取即可，时间复杂度O（1）         |
   | 内存重新分配   | 分配内存空间超过后，会导致数据下标越界或者内存分配溢出       | 主要有两点好处（1）空间预分配：SDS修改后，len长度小于1M，那么将会额外分配与len相同长度的未使用空间，如果修改后长度大于1M，那么将分配1M的空间。（2）惰性空间释放：有空间分配对应的就有空间释放。SDS缩短时并不会回收多余的内存空间，而是使用free字段将多出来的空间记录下来。如果后有变操作，直接使用free中记录的空间，减少了内存的分配 |
   | 二进制安全     | 二进制数据并不是规则的字符串格式，可能会包含一些特殊的字符，比如`\0`等，前面说过，C遇到这个字符串自动结束，那么之后的数据就读取不上了 | 根据len长度来判断字符串是否结束，二进制安全的问题就解决了    |

6. 源码分析

   ![](../imgs/image-20211014134738329.png)

   输入set命令之后后面为什么会有后面这一段提示关键字呢，带着这个问题可以去源码（t_string.c）里面看看

   ```c
   /* SET key value [NX] [XX] [KEEPTTL] [GET] [EX <seconds>] [PX <milliseconds>]
    *     [EXAT <seconds-timestamp>][PXAT <milliseconds-timestamp>] */
   void setCommand(client *c) {
       robj *expire = NULL; /* 过期时间 */
       int unit = UNIT_SECONDS; /* 时间类型 秒*/
       int flags = OBJ_NO_FLAGS; /* 标志：默认没有标志*/
   
       if (parseExtendedStringArgumentsOrReply(c,&flags,&unit,&expire,COMMAND_SET) != C_OK) {
           return;
       }
   
       c->argv[2] = tryObjectEncoding(c->argv[2]); /* 处理编码的时候发现调用了tryObjectEncoding*/
       setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
   }
   ```

   也就是说当你使用set命令的时候，其实是调用了void setCommand(client *c)这个api，也就是说在Redis层面是`set k1 v1`，但是底层调用的就是setCommand这个方法，那为什么会提示这一串呢，读源码可以发现Redis自己拼接出来的：

   ```c
   int parseExtendedStringArgumentsOrReply(client *c, int *flags, int *unit, robj **expire, int command_type) {
   
       int j = command_type == COMMAND_GET ? 2 : 3;
       for (; j < c->argc; j++) {
           char *opt = c->argv[j]->ptr;
           robj *next = (j == c->argc-1) ? NULL : c->argv[j+1];
   
           if ((opt[0] == 'n' || opt[0] == 'N') &&
               (opt[1] == 'x' || opt[1] == 'X') && opt[2] == '\0' &&
               !(*flags & OBJ_SET_XX) && (command_type == COMMAND_SET))
           {
               *flags |= OBJ_SET_NX;
           } else if ((opt[0] == 'x' || opt[0] == 'X') &&
                      (opt[1] == 'x' || opt[1] == 'X') && opt[2] == '\0' &&
                      !(*flags & OBJ_SET_NX) && (command_type == COMMAND_SET))
           {
               *flags |= OBJ_SET_XX;
           } else if ((opt[0] == 'g' || opt[0] == 'G') &&
                      (opt[1] == 'e' || opt[1] == 'E') &&
                      (opt[2] == 't' || opt[2] == 'T') && opt[3] == '\0' &&
                      (command_type == COMMAND_SET))
           {
               *flags |= OBJ_SET_GET;
           } else if (!strcasecmp(opt, "KEEPTTL") && !(*flags & OBJ_PERSIST) &&
               !(*flags & OBJ_EX) && !(*flags & OBJ_EXAT) &&
               !(*flags & OBJ_PX) && !(*flags & OBJ_PXAT) && (command_type == COMMAND_SET))
           {
               *flags |= OBJ_KEEPTTL;
           } else if (!strcasecmp(opt,"PERSIST") && (command_type == COMMAND_GET) &&
                  !(*flags & OBJ_EX) && !(*flags & OBJ_EXAT) &&
                  !(*flags & OBJ_PX) && !(*flags & OBJ_PXAT) &&
                  !(*flags & OBJ_KEEPTTL))
           {
               *flags |= OBJ_PERSIST;
           } else if ((opt[0] == 'e' || opt[0] == 'E') &&
                      (opt[1] == 'x' || opt[1] == 'X') && opt[2] == '\0' &&
                      !(*flags & OBJ_KEEPTTL) && !(*flags & OBJ_PERSIST) &&
                      !(*flags & OBJ_EXAT) && !(*flags & OBJ_PX) &&
                      !(*flags & OBJ_PXAT) && next)
           {
               *flags |= OBJ_EX;
               *expire = next;
               j++;
           } else if ((opt[0] == 'p' || opt[0] == 'P') &&
                      (opt[1] == 'x' || opt[1] == 'X') && opt[2] == '\0' &&
                      !(*flags & OBJ_KEEPTTL) && !(*flags & OBJ_PERSIST) &&
                      !(*flags & OBJ_EX) && !(*flags & OBJ_EXAT) &&
                      !(*flags & OBJ_PXAT) && next)
           {
               *flags |= OBJ_PX;
               *unit = UNIT_MILLISECONDS;
               *expire = next;
               j++;
           } else if ((opt[0] == 'e' || opt[0] == 'E') &&
                      (opt[1] == 'x' || opt[1] == 'X') &&
                      (opt[2] == 'a' || opt[2] == 'A') &&
                      (opt[3] == 't' || opt[3] == 'T') && opt[4] == '\0' &&
                      !(*flags & OBJ_KEEPTTL) && !(*flags & OBJ_PERSIST) &&
                      !(*flags & OBJ_EX) && !(*flags & OBJ_PX) &&
                      !(*flags & OBJ_PXAT) && next)
           {
               *flags |= OBJ_EXAT;
               *expire = next;
               j++;
           } else if ((opt[0] == 'p' || opt[0] == 'P') &&
                      (opt[1] == 'x' || opt[1] == 'X') &&
                      (opt[2] == 'a' || opt[2] == 'A') &&
                      (opt[3] == 't' || opt[3] == 'T') && opt[4] == '\0' &&
                      !(*flags & OBJ_KEEPTTL) && !(*flags & OBJ_PERSIST) &&
                      !(*flags & OBJ_EX) && !(*flags & OBJ_EXAT) &&
                      !(*flags & OBJ_PX) && next)
           {
               *flags |= OBJ_PXAT;
               *unit = UNIT_MILLISECONDS;
               *expire = next;
               j++;
           } else {
               addReplyErrorObject(c,shared.syntaxerr);
               return C_ERR;
           }
       }
       return C_OK;
   }
   ```

   当输入`set k1 123`命令的时候，如果这个键值的内容可以用一个64位有符号整形来表示时，Redis会将键值转化为long型来存储，此时即对应OBJ_ENCODING_INT编码类型。而且更重要的是Redis在启动时会预先建立10000（为什么是10000，稍后解释）个分别存储0～9999的redisObject变量作为共享对象，这就意味着如果set字符串的键值在0～10000之间的话，则可以直接指向共享对象而不需要再建立新对象，此时，键值就不需要再占用内存空间。相应的图示：

   ![image-20211014142747301](../imgs/image-20211014142747301.png)

   处理编码的源码（object.c）：

   ```c
   /* Try to encode a string object in order to save space */
   robj *tryObjectEncoding(robj *o) {
       long value;
       sds s = o->ptr;
       size_t len;
   
       /* Make sure this is a string object, the only type we encode
        * in this function. Other types use encoded memory efficient
        * representations but are handled by the commands implementing
        * the type. */
       serverAssertWithInfo(NULL,o,o->type == OBJ_STRING);
   
       /* We try some specialized encoding only for objects that are
        * RAW or EMBSTR encoded, in other words objects that are still
        * in represented by an actually array of chars. */
       if (!sdsEncodedObject(o)) return o;
   
       /* It's not safe to encode shared objects: shared objects can be shared
        * everywhere in the "object space" of Redis and may end in places where
        * they are not handled. We handle them only as values in the keyspace. */
        if (o->refcount > 1) return o;
   
       /* Check if we can represent this string as a long integer.
        * Note that we are sure that a string larger than 20 chars is not
        * representable as a 32 nor 64 bit integer. */
       len = sdslen(s);
     	/* 在这里可以发现字符串长度小于等于20的时候，字符串进行了转long操作*/
       if (len <= 20 && string2l(s,len,&value)) { 
           /* This object is encodable as a long. Try to use a shared object.
            * Note that we avoid using shared integers when maxmemory is used
            * because every object needs to have a private LRU field for the LRU
            * algorithm to work well. */
           if ((server.maxmemory == 0 ||
               !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS)) &&
               value >= 0 &&
               value < OBJ_SHARED_INTEGERS)
           {
             /* 在这里配置maxmemory值在10000以内的时候，直接使用共享对象值 */
               decrRefCount(o);
               incrRefCount(shared.integers[value]);
               return shared.integers[value];
           } else {
               if (o->encoding == OBJ_ENCODING_RAW) {
                   sdsfree(o->ptr);
                   o->encoding = OBJ_ENCODING_INT;
                   o->ptr = (void*) value;
                   return o;
               } else if (o->encoding == OBJ_ENCODING_EMBSTR) {
                   decrRefCount(o);
                   return createStringObjectFromLongLongForValue(value);
               }
           }
       }
   
       /* If the string is small and is still RAW encoded,
        * try the EMBSTR encoding which is more efficient.
        * In this representation the object and the SDS string are allocated
        * in the same chunk of memory to save space and cache misses. */
       if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT) {
           robj *emb;
   
           if (o->encoding == OBJ_ENCODING_EMBSTR) return o;
           emb = createEmbeddedStringObject(s,sdslen(s));
           decrRefCount(o);
           return emb;
       }
   
       /* We can't encode the object...
        *
        * Do the last try, and at least optimize the SDS string inside
        * the string object to require little space, in case there
        * is more than 10% of free space at the end of the SDS string.
        *
        * We do that only for relatively large strings as this branch
        * is only entered if the length of the string is greater than
        * OBJ_ENCODING_EMBSTR_SIZE_LIMIT. */
       trimStringObjectIfNeeded(o);
   
       /* Return the original object. */
       return o;
   }
   ```

   上面的源码中还有一点需要清楚，那就是对于长度小于44的字符串（为什么是44，稍后解释），Redis对键值采用OBJ_ENCODING_EMBSTR的方式编码，EMBSTR顾名思义就是：embedded string，表示嵌入式的String。从内存结构上来讲，即字符串SDS结构体与其对应的 redisObject对象分配在同一块连续的空间内（为什么说连续，稍后解释），字符串SDS嵌入在redisObject对象之中一样。如果通俗的理解呢，你可以想象成一个糖果里面嵌入了夹心，他们融合在一起了，这样只占用一块内存空间，大大节省了内存。

   ```c
   /* Create a string object with encoding OBJ_ENCODING_EMBSTR, that is
    * an object where the sds string is actually an unmodifiable string
    * allocated in the same chunk as the object itself. */
   robj *createEmbeddedStringObject(const char *ptr, size_t len) {
       robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
       struct sdshdr8 *sh = (void*)(o+1);
   
       o->type = OBJ_STRING;
       o->encoding = OBJ_ENCODING_EMBSTR;
       o->ptr = sh+1;
       o->refcount = 1;
       if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
           o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
       } else {
           o->lru = LRU_CLOCK();
       }
   
       sh->len = len;
       sh->alloc = len;
       sh->flags = SDS_TYPE_8;
       if (ptr == SDS_NOINIT)
           sh->buf[len] = '\0';
       else if (ptr) {
           memcpy(sh->buf,ptr,len);
           sh->buf[len] = '\0';
       } else {
           memset(sh->buf,0,len+1);
       }
       return o;
   }
   ```

   读createEmbeddedStringObject这个方法，就可以发现为什么是连续了，因为他在真正的引用上面+1（o->ptr = sh+1），这就是保证了内存空间的连续性。

   在server.h源码里面可以发现OBJ_SHARED_INTEGERS是10000：

   ```c++
   #define OBJ_SHARED_INTEGERS 10000
   ```

   在object.c里面可以发现OBJ_ENCODING_EMBSTR_SIZE_LIMIT是44：

   ```c
   #define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
   robj *createStringObject(const char *ptr, size_t len) {
       if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
           return createEmbeddedStringObject(ptr,len);
       else
           return createRawStringObject(ptr,len);
   }
   ```

   还可以进一步发现，当字符串的键值长度大于44的时候，Redis则会把键值的内部编码改为OBJ_ENCDING_RAW格式，这与OBJ_ENCODING_EMBSTR编码方式的不同之处就在于，此时动态字符串SDS与其依赖的redisObject的内存不在连续了。

   ```c
   robj *createObject(int type, void *ptr) {
       robj *o = zmalloc(sizeof(*o));
       o->type = type;
       o->encoding = OBJ_ENCODING_RAW;
       o->ptr = ptr;
       o->refcount = 1;
   
       /* Set the LRU to the current lruclock (minutes resolution), or
        * alternatively the LFU counter. */
       if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
           o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
       } else {
           o->lru = LRU_CLOCK();
       }
       return o;
   }
   ```

   图示大概是这样表示：

   ![image-20211014145137598](../imgs/image-20211014145137598.png)

   还有一点需要注意的是，有时候没有超过字符串的键值长度大于44这个阈值时，这个键值的编码类型也变成了 RAW了，类似于这样：

   ```shell
   127.0.0.1:6379> set k1 abc
   OK
   127.0.0.1:6379> object encoding k1
   "embstr"
   127.0.0.1:6379> APPEND k1 d
   (integer) 4
   127.0.0.1:6379> object encoding k1
   "raw"
   127.0.0.1:6379> 
   ```

   这是因为对于embstr编码格式来说，它的实现是只读的，因为在修改时，都会先转化为raw再进行修改，因为Redis不知道你要修改成多大的，索性就直接给你不限制了。因此只要修改embstr对象，修改后的对象一定是RAW的，无论是否达到了44个字节。

7. 结论总结

   通过上面这一系列的案列和源码分析，我们可以发现：

   只有整数才会使用int，如果浮点数，Redis内部其实先将浮点数转化为字符串值，然后再保存。

   embstr和raw类型底层的数据结构其实都是SDS（简单动态字符串，Redis内部定义的一种结构（sdshdr））

   他们之间的区别：

   | 类型   | 意义                                                         |      |
   | ------ | ------------------------------------------------------------ | ---- |
   | int    | Long类型整数时，RedisObject中的ptr指针直接赋值为整数数据，不再额外的指针再指向整数了，节省了指针的空间开销 |      |
   | embstr | 当保存的是字符串数据且字符串小于等于44字节时，embstr类型将会调用内存分配函数， 只分配一块连续的内存空间 ，空间中依次包含 redisObject 与 sdshdr 两个数据结构，让元数据、指针和SDS是一块连续的内存区域，这样就可以避免内存碎片 |      |
   | raw    | 当字符串大于44字节时，SDS的数据量变多变大了，SDS和RedisObject布局分家各自过，会给SDS分配多的空间并用指针指向SDS结构，raw 类型将会 调用两次内存分配函数 ，分配两块内存空间，一块用于包含 redisObject结构，而另一块用于包含 sdshdr 结构 |      |

   一言以蔽之：Redis内部会根据用户提供的不用键值而使用不同的编码格式，自适应的选择较优的内部编码格式，以此来提高性能。

#### 列表（t_list.c）

#### 字典（t_hash.c）

Redis hash 看起来就像一个 “hash” 的样子，由键值对组成：

```shell
127.0.0.1:6379> hmset user:1000 username antirez birthday 197 verified 1
OK
127.0.0.1:6379> hget user:1000 username
"antirez"
127.0.0.1:6379> hgetall user:1000
1) "username"
2) "antirez"
3) "birthday"
4) "197"
5) "verified"
6) "1"
127.0.0.1:6379> 
```

关于hash有两个重要的配置参数：

```shell
127.0.0.1:6379> config get hash*
1) "hash-max-ziplist-entries"
2) "512"
3) "hash-max-ziplist-value"
4) "64"
127.0.0.1:6379> 
```

**hash-max-ziplist-entries**：使用压缩列表保存时哈希集合中的最大元素个数

**hash-max-ziplist-value**：使用压缩列表保存时哈希集合中单个元素的最大长度

Hash类型键的字段个数  小于  hash-max-ziplist-entries 并且每个字段名和字段值的长度  小于  hash-max-ziplist-value 时，Redis才会使用 OBJ_ENCODING_ZIPLIST来存储该键，前述条件任意一个不满足则会转换为 OBJ_ENCODING_HT的编码方式

示例如下：

```shell
127.0.0.1:6379> object encoding user:1000
"ziplist"
127.0.0.1:6379> type user:1000
hash
127.0.0.1:6379> config set hash-max-ziplist-entries 4
OK
127.0.0.1:6379> config set hash-max-ziplist-value 4
OK
127.0.0.1:6379> config get hash*
1) "hash-max-ziplist-entries"
2) "4"
3) "hash-max-ziplist-value"
4) "4"
127.0.0.1:6379> hset person id 1 name guo
(integer) 2
127.0.0.1:6379> type person
hash
127.0.0.1:6379> objcet encoding person
(error) ERR unknown command `objcet`, with args beginning with: `encoding`, `person`, 
127.0.0.1:6379> object encoding person
"ziplist"
127.0.0.1:6379> hset person id 1 name guo age 25 score 99 birth 2021
(integer) 3
127.0.0.1:6379> object encoding person
"hashtable"
127.0.0.1:6379> hset person id 1 name guogu
(integer) 2
127.0.0.1:6379> object encoding person
"hashtable"
```

结论：

1. 哈希对象保存的键值对数量小于512个
2. 所有的键值对象的键和值的字符串对象长度都小于等于64byte（一个英文字母一个字节）是用ziplist，反之用hashtable
3. ziplist升级到hashtable可以，反过来降级不可以

源码分析：

- ziplist

  ziplist压缩列表是一种紧凑编码格式，总体思想：多花时间来换取节约空间。即：以少部分读写性能为代价，来换取极高的内存空间利用率，因此，只会用于字段个数少，且字段值也较小的场景。这个特性主要与连续内存的特性是分不开的，之前学习GC垃圾回收机制：标记-压缩算法的底层其实就是利用了ziplist。当一个hash对象只包含少量键值对且每个键值对的键和值要么就是小整数要么就是长度比较短的字符串，那么它就适合使用ziplist作为底层实现。

  ```c
   /*The ziplist is a specially encoded dually linked list that is designed
   * to be very memory efficient. It stores both strings and integer values,
   * where integers are encoded as actual integers instead of a series of
   * characters. It allows push and pop operations on either side of the list
   * in O(1) time. However, because every operation requires a reallocation of
   * the memory used by the ziplist, the actual complexity is related to the
   * amount of memory used by the ziplist.
   */
  // 这是源码的一部分注释（ziplist.c）
  ```

  ziplist是一个经过特殊编码的双向链表，它不存储指向上一个链表节点和指向下一个链表节点的指针，而是存储上一个节点长度和当前节点长度，通过牺牲部分读写性能，来换取高效的内存空间利用率，节约内存，是一种时间换空间的思想。前面说过了，只用在字段个数小，字段值小的场景里面。

  ```c
  <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
  ```

  图示：

  <img src="../imgs/image-20211019145628784.png" alt="image-20211019145628784" style="zoom:50%;" />

  上述图示可以分为3大部分，分别是header+entry集合+end，其实header由zlbytes、zltail和zllen组成，entry是节点，zlend是一个单字节255（1111 1111），用作ziplist的结尾标识符。

  - zlbytes：4字节，记录整个压缩列表占用的内存字节数
  - zltail：4字节，记录压缩列表表尾节点的位置
  - zllen：2字节，记录压缩列表节点个数
  - entry：列表节点，长度不定，由内容决定
  - zlend：1字节，0xFF 标记压缩的结束（16进制FF对应二进制1111 1111，十进制255）

  数据结构：

  ```c
  
  /* We use this function to receive information about a ziplist entry.
   * Note that this is not how the data is actually encoded, is just what we
   * get filled by a function in order to operate more easily. */
  typedef struct zlentry {
    	/* 上一个链表节点占用的长度 */
      unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    	/* 存储上一个链表节点的长度数值所需要的字节数（上一个节点的长度） */
      unsigned int prevrawlen;     /* Previous entry len. */
    	/* 存储当前链表节点长度数值所需要的字节数 */
      unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                      For example strings have a 1, 2 or 5 bytes
                                      header. Integers always use a single byte.*/
    	/* 当前链表节点占用的长度 */
      unsigned int len;            /* Bytes used to represent the actual entry.
                                      For strings this is just the string length
                                      while for integers it is 1, 2, 3, 4, 8 or
                                      0 (for 4 bit immediate) depending on the
                                      number range. */
    	/* 当前链表节点的头部大小=prevrawlensize + lensize，即非数据域的大小 */
      unsigned int headersize;     /* prevrawlensize + lensize. */
    	/* 编码方式 */
      unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                      the entry encoding. However for 4 bits
                                      immediate integers this can assume a range
                                      of values and must be range-checked. */
    	/* 压缩链表以字符串的形式保存，该指针指向当前节点起始位置 */
      unsigned char *p;            /* Pointer to the very start of the entry, that
                                      is, this points to prev-entry-len field. */
  } zlentry;
  ```

  压缩列表zlentry节点结构：每个zlentry由前一个节点的长度、encoding和entry-data三部分组成

  - 前节点：表示前1个zlentry的长度，prev_len有两种取值情况： 1字节或5字节 。取值1字节时，表示上一个entry的长度小于254字节。虽然1字节的值能表示的数值范围是0到255，但是压缩列表中zlend的取值默认是255，因此，就默认用255表示整个压缩列表的结束，其他表示长度的地方就不能再用255这个值了。所以，当上一个entry长度小于254字节时，prev_len取值为1字节，否则，就取值为5字节。
  - enncoding： 记录节点的content保存数据的类型和长度。
  - entry-data： 保存实际数据内容

  ![image-20211019151411848](../imgs/image-20211019151411848.png)

  压缩列表的遍历：通过指向表尾节点的位置指针p1, 减去节点的previous_entry_length，得到前一个节点起始地址的指针。如此循环，从表尾遍历到表头节点。从表尾向表头遍历操作就是使用这一原理实现的，只要我们拥有了一个指向某个节点起始地址的指针，那么通过这个指针以及这个节点的previous_entry_length属性程序就可以一直向前一个节点回溯，最终到达压缩列表的表头节点。

  使用压缩列表的好处（相比于链表）：

  1. 普通的双向链表会有两个指针，在存储数据很小的情况下，存储的实际数据的大小可能还没有指针占用的内存大，得不偿失。ziplist 是一个特殊的双向链表没有维护双向指针:prev next；而是存储上一个 entry的长度和 当前entry的长度，通过长度推算下一个元素在什么地方。牺牲读取的性能，获得高效的存储空间，因为(简短字符串的情况)存储指针比存储entry长度更费内存。这是典型的“时间换空间”。 
  2. 链表在内存中一般是不连续的，遍历相对比较慢，而ziplist可以很好的解决这个问题，普通数组的遍历是根据数组里存储的数据类型找到下一个元素的(例如int类型的数组访问下一个元素时每次只需要移动一个sizeof(int)就行)，但是ziplist的每个节点的长度是可以不一样的，而我们面对不同长度的节点又不可能直接sizeof(entry)，所以ziplist只好将一些必要的偏移量信息记录在了每一个节点里，使之能跳到上一个节点或下一个节点。
  3. 头节点里有头节点里同时还有一个参数 len，和string类型提到的 SDS 类似，这里是用来记录链表长度的。获取链表长度时不用再遍历整个链表，直接拿到len值就可以了，这个时间复杂度是 O(1)。

- hash

  在Redis中，hashTable被称为字典（dictionary），是一个数组+链表的结构

  ```c
  void hsetCommand(client *c) {
      int i, created = 0;
      robj *o;
  
      if ((c->argc % 2) == 1) {
          addReplyErrorFormat(c,"wrong number of arguments for '%s' command",c->cmd->name);
          return;
      }
  
      if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
      hashTypeTryConversion(o,c->argv,2,c->argc-1);
  
      for (i = 2; i < c->argc; i += 2)
          created += !hashTypeSet(o,c->argv[i]->ptr,c->argv[i+1]->ptr,HASH_SET_COPY);
  
      /* HMSET (deprecated) and HSET return value is different. */
      char *cmdname = c->argv[0]->ptr;
      if (cmdname[1] == 's' || cmdname[1] == 'S') {
          /* HSET */
          addReplyLongLong(c, created);
      } else {
          /* HMSET */
          addReply(c, shared.ok);
      }
      signalModifiedKey(c,c->db,c->argv[1]);
      notifyKeyspaceEvent(NOTIFY_HASH,"hset",c->argv[1],c->db->id);
      server.dirty += (c->argc - 2)/2;
  }
  ```

  之前我们说过有多种编码方式，其中的OBJ_ENCODING_HT 这种编码方式内部才是真正的哈希表结构，或称为字典结构，其可以实现O(1)复杂度的读写操作，因此效率很高。

  在 Redis内部，从 OBJ_ENCODING_HT类型到底层真正的散列表数据结构是一层层嵌套下去的，组织关系如图：

  

  ![image-20211019165425711](../imgs/image-20211019165425711.png)

  也就是说从宏观的编码开始，先是转化字典，而字典的底层又是哈希表，哈希表里面又存了一个个的哈希节点

  ```c
  typedef struct dictEntry { //1、哈希节点
      void *key;
      union {
          void *val;
          uint64_t u64;
          int64_t s64;
          double d;
      } v;
      struct dictEntry *next;     /* Next entry in the same hash bucket. */
      void *metadata[];           /* An arbitrary number of bytes (starting at a
                                   * pointer-aligned address) of size as returned
                                   * by dictType's dictEntryMetadataBytes(). */
  } dictEntry;
  
  typedef struct dict dict;
  
  typedef struct dictType {
      uint64_t (*hashFunction)(const void *key);
      void *(*keyDup)(dict *d, const void *key);
      void *(*valDup)(dict *d, const void *obj);
      int (*keyCompare)(dict *d, const void *key1, const void *key2);
      void (*keyDestructor)(dict *d, void *key);
      void (*valDestructor)(dict *d, void *obj);
      int (*expandAllowed)(size_t moreMem, double usedRatio);
      /* Allow a dictEntry to carry extra caller-defined metadata.  The
       * extra memory is initialized to 0 when a dictEntry is allocated. */
      size_t (*dictEntryMetadataBytes)(dict *d);
  } dictType;
  
  #define DICTHT_SIZE(exp) ((exp) == -1 ? 0 : (unsigned long)1<<(exp)) // 2、哈希表
  #define DICTHT_SIZE_MASK(exp) ((exp) == -1 ? 0 : (DICTHT_SIZE(exp))-1)
  
  struct dict { // 3、字典
      dictType *type;
  
      dictEntry **ht_table[2];
      unsigned long ht_used[2];
  
      long rehashidx; /* rehashing not in progress if rehashidx == -1 */
  
      /* Keep small vars at end for optimal (minimal) struct padding */
      int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
      signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
  };
  ```

总结：简单来说就是类似于String，只不过String使用了不同的编码格式，而hash使用了不同的数据结构，而做这些的目的都是为了实现Redis高性能的特性

#### 集合及有序集合（t_set.c和t_zset.c）













#### 数据流（t_stream.c）