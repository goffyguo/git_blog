mysql 数据库表设计规约

### 一、前言

其实数据库的设计没有什么绝对对错，包括三大范式也只是理论意义上参考，多数需要从业务场景出发设计每一张表。该冗余的冗余，该固定的固定

但是有一些规则或者约束需要在设计前期在团队进行统一约束，这也是为了大家都遵守这个规定，配合团队协作衍生的规则而已

下面会从【高危】、【强制】、【建议】三个级别进行标注，遵守优先级从高到低：

### 二、设计规范

**2.1、库名**

1. 【强制】库的名称必须控制在32个字符以内。

2. 【强制】库的名称格式：业务系统名称_子系统名，同一模块使用的表名需要使用同一前缀。

3. 【强制】创建数据库时必须显式指定字符集，并且字符集只能是utf8或者utf8mb4。

   创建数据库SQL举例：`ENGINE=InnoDB DEFAULT CHARSET=utf8mb4`

**2.2、表结构**

1. 【强制】表和列的名称必须控制在32个字符以内，表名只能使用字母、数字和下划线，一律小写。
2. 【强制】表名要求模块名强相关，如交易系统采用”exchange”作为前缀，积分系统采用”integral”作为前缀等。
3. 【强制】创建表时必须显式指定字符集为utf8或utf8mb4。
4. 【强制】创建表时必须显式指定表存储引擎类型，如无特殊需求，一律为InnoDB。因为Innodb表支持事务、行锁、宕机恢复、MVCC等关系型数据库重要特性，为业界使用最多的MySQL存储引擎。而这是其他大多数存储引擎不具备的，因此首推InnoDB。
5. 【强制】建表必须有comment
6. 【建议】建表时关于主键：(1)强制要求主键为id，类型为int或bigint，且为`auto_increment` (2)标识表里每一行主体的字段不要设为主键，建议设为其他字段如`user_id`，`order_id`等，并建立unique key索引。因为如果设为主键且主键值为随机插入，则会导致innodb内部page分裂和大量随机I/O，性能下降。
7. 【建议】核心表（如用户表，金钱相关的表）必须有行数据的创建时间字段`create_time`和最后更新时间字段`update_time`，便于查问题。
8. 【建议】表中所有字段必须都是`NOT NULL`属性，业务可以根据需要定义`DEFAULT`值。因为使用NULL值会存在每一行都会占用额外存储空间、数据迁移容易出错、聚合函数计算结果偏差等问题。
9. 【建议】建议对表里的`blob`、`text`等大字段，垂直拆分到其他表里，仅在需要读这些对象的时候才去select。
10. 【建议】反范式设计：把经常需要join查询的字段，在其他表里冗余一份。如`user_name`属性在`user_account`，`user_login_log`等表里冗余一份，减少join查询。
11. 【强制】中间表用于保留中间结果集，名称必须以`tmp_`开头。备份表用于备份或抓取源表快照，名称必须以`bak_`开头。中间表和备份表定期清理。
12. 【强制】对于超过100W行的大表进行`alter table`，必须经过DBA审核，并在业务低峰期执行。因为`alter table`会产生表锁，期间阻塞对于该表的所有写入，对于业务可能会产生极大影响。

**2.3、列数据数据类型选择**

1. 建议】表中的自增列（`auto_increment`属性），推荐使用`bigint`类型。因为无符号`int`存储范围为`2147483648~2147483647`（大约21亿左右），溢出后会导致报错。

2. 【建议】业务中选择性很少的状态`status`、类型`type`等字段推荐使用`tinytint`或者`smallint`类型节省存储空间。

3. 【建议】业务中IP地址字段推荐使用`int`类型，不推荐用`char(15)`。因为`int`只占4字节，可以用如下函数相互转换，而`char(15)`占用至少15字节。一旦表数据行数到了1亿，那么要多用1.1G存储空间。SQL：`select inet_aton('192.168.2.12'); select inet_ntoa(3232236044);`PHP: `ip2long(‘192.168.2.12’); long2ip(3530427185);`

4. 【建议】不推荐使用`enum`，`set`。 因为它们浪费空间，且枚举值写死了，变更不方便。推荐使用`tinyint`或`smallint`。

5. 【建议】不推荐使用`blob`，`text`等类型。它们都比较浪费硬盘和内存空间。在加载表数据时，会读取大字段到内存里从而浪费内存空间，影响系统性能。建议和PM、RD沟通，是否真的需要这么大字段。Innodb中当一行记录超过8098字节时，会将该记录中选取最长的一个字段将其768字节放在原始page里，该字段余下内容放在`overflow-page`里。不幸的是在`compact`行格式下，原始`page`和`overflow-page`都会加载。

6. 【建议】存储金钱的字段，建议用`int`，程序端乘以100和除以100进行存取。因为`int`占用4字节，而`double`占用8字节，空间浪费。

7. 【建议】文本数据尽量用`varchar`存储。因为`varchar`是变长存储，比`char`更省空间。MySQL server层规定一行所有文本最多存65535字节，因此在utf8字符集下最多存21844个字符，超过会自动转换为`mediumtext`字段。而`text`在utf8字符集下最多存21844个字符，`mediumtext`最多存232个字符。一般建议用`varchar`类型，字符数不要超过2700。

   24/3个字符，`longtext`最多存2

8. 【建议】时间类型尽量选取`timestamp`。因为`datetime`占用8字节，`timestamp`仅占用4字节，但是范围为`1970-01-01 00:00:01`到`2038-01-01 00:00:00`。更为高阶的方法，选用`int`来存储时间，使用SQL函数`unix_timestamp()`和`from_unixtime()`来进行转换，但是timestamp可读性太差，具体可根据团队情况来定。

附一张数据类型存储大小参考图：

| 类型      | 大小   | 范围（有符号）                                               | 范围（无符号）/[时间格式] | 用途                     |
| --------- | ------ | ------------------------------------------------------------ | ------------------------- | ------------------------ |
| TINYINT   | 1 字节 | （-128，127）                                                | （0，255）                | 小整数值                 |
| SMALLINT  | 2 字节 | （-32768，32767）                                            | （0，65535）              | 大整数值                 |
| DATE      | 3      | 1000-01-01/9999-12-31                                        | YYYY-MM-DD                | 日期值                   |
| TIME      | 3      | '-838:59:59'/'838:59:59'                                     | HH:MM:SS                  |                          |
| YEAR      | 1      | 1901/2155                                                    | YYYY                      |                          |
| DATETIME  | 8      | 1000-01-01 00:00:00/9999-12-31 23:59:59                      | YYYY-MM-DD HH:MM:SS       | 混合日期和时间值         |
| TIMESTAMP | 4      | 1970-01-01 00:00:00/2038 结束时间是第 2147483647 秒，北京时间 2038-1-19 11:14:07，格林尼治时间 2038年1月19日凌晨 03:14:07 | YYYYMMDD HHMMSS           | 混合日期和时间值，时间戳 |
|           |        |                                                              |                           |                          |



**2.3、索引设计**

1. 【强制】InnoDB表必须主键为`id int/bigint auto_increment`,且主键值禁止被更新。
2. 【建议】主键的名称以“`pk_`”开头，唯一键以“`uk_`”或“`uq_`”开头，普通索引以“`idx_`”开头，一律使用小写格式，以表名/字段的名称或缩写作为后缀。
3. 【强制】InnoDB和MyISAM存储引擎表，索引类型必须为`BTREE`；MEMORY表可以根据需要选择`HASH`或者`BTREE`类型索引。
4. 【强制】单个索引中每个索引记录的长度不能超过64KB。
5. 【建议】单个表上的索引个数不能超过7个。
6. 【建议】在建立索引时，多考虑建立联合索引，并把区分度最高的字段放在最前面。如列`userid`的区分度可由`select count(distinct userid)`计算出来。
7. 【建议】在多表join的SQL里，保证被驱动表的连接列上有索引，这样join执行效率最高。
8. 【建议】建表或加索引时，保证表里互相不存在冗余索引。对于MySQL来说，如果表里已经存在`key(a,b)`，则`key(a)`为冗余索引，需要删除。