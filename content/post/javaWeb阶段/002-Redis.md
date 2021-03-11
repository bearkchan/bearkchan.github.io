---
title: "Redis的基本使用"
date: 2020-03-02T00:29:44+08:00
draft: false
Categories: ["redis"]
Tags: ["redis", "jeudis"]
---



## 1. 分布式数据库CAP原理

### 1.1 传统的ACID

- A (Atomicity) 原子性
- C (Consistency) 一致性
- I (Isolation) 独立性
- D (Durability) 持久性

1、A (Atomicity) 原子性 原子性很容易理解，也就是说事务里的所有操作要么全部做完，要么都不做，事务成功的条件是事务里的所有操作都成功，只要有一个操作失败，整个事务就失败，需要回滚。比如银行转账，从A账户转100元至B账户，分为两个步骤：1）从A账户取100元；2）存入100元至B账户。这两步要么一起完成，要么一起不完成，如果只完成第一步，第二步失败，钱会莫名其妙少了100元。

2、C (Consistency) 一致性 一致性也比较容易理解，也就是说数据库要一直处于一致的状态，事务的运行不会改变数据库原本的一致性约束。

3、I (Isolation) 独立性 所谓的独立性是指并发的事务之间不会互相影响，如果一个事务要访问的数据正在被另外一个事务修改，只要另外一个事务未提交，它所访问的数据就不受未提交事务的影响。比如现有有个交易是从A账户转100元至B账户，在这个交易还未完成的情况下，如果此时B查询自己的账户，是看不到新增加的100元的

4、D (Durability) 持久性 持久性是指一旦事务提交后，它所做的修改将会永久的保存在数据库上，即使出现宕机也不会丢失。

### 1.2 CAP原理

- C:Consistency（强一致性）
- A:Availability（可用性）
- P:Partition tolerance（分区容错性）

CAP理论的核心是：一个分布式系统不可能同时很好的满足一致性，可用性和分区容错性这三个需求，最多只能同时较好的满足两个。

因此，根据 CAP 原理将 NoSQL 数据库分成了满足 CA 原则、满足 CP 原则和满足 AP 原则三 大类：

CA - 单点集群，满足一致性，可用性的系统，通常在可扩展性上不太强大。
CP - 满足一致性，分区容忍必的系统，通常性能不是特别高。
AP - 满足可用性，分区容忍性的系统，通常可能对一致性要求低一些。

而由于当前的网络硬件肯定会出现延迟丢包等问题，所以分区容忍性是我们必须需要实现的。所以我们只能在一致性和可用性之间进行权衡，没有NoSQL系统能同时保证这三点。

- CA-传统Oracle数据库

- AP-大多数网站架构的选择

- CP-Redis、Mongodb

  > 注意：分布式架构的时候必须做出取舍。

一致性和可用性之间取一个平衡。多余大多数web应用，其实并不需要强一致性。因此牺牲C换取P，这是目前分布式数据库产品的方向。

### 1.3 BASE

BASE就是为了解决关系数据库强一致性引起的问题而引起的可用性降低而提出的解决方案。

BASE其实是下面三个术语的缩写：

- 基本可用（Basically Available）
- 软状态（Soft state）
- 最终一致（Eventually consistent）

它的思想是通过让系统放松**对某一时刻数据一致性的要求来换取系统整体伸缩性和性能上改观**。为什么这么说呢，缘由就在于大型系统往往由于地域分布和极高性能的要求，不可能采用分布式事务来完成这些指标，要想获得这些指标，我们必须采用另外一种方式来完成，这里BASE就是解决这个问题的办法。

### 1.4 分布式与集群的概念

1. 分布式：不同的多台服务器上面部署**不同**的服务模块（工程），他们之间通过Rpc/Rmi之间通信和调用，对外提供服务和组内协作。
2. 集群：不同的多台服务器上面部署**相同**的服务模块，通过分布式调度软件进行统一的调度，对外提供服务和访问。



## 2. Redis介绍

### 2.1 什么是Redis？

Redis:REmote DIctionary Server(远程字典服务器)是完全开源免费的，用C语言编写的，遵守BSD协议，是一个高性能的(key/value)分布式内存数据库，基于内存运行 并支持持久化的NoSQL数据库，是当前最热门的NoSql数据库之一，也被人们称为数据结构服务器。

Redis 与其他 key - value 缓存产品有以下三个特点：

- Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用
- Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储
- Redis支持数据的备份，即master-slave模式的数据备份

### 2.2 Redis能干什么？

- 内存存储和持久化：redis支持异步将内存中的数据写到硬盘上，同时不影响继续服务
- 取最新N个数据的操作，如：可以将最新的10条评论的ID放在Redis的List集合里面
- 模拟类似于HttpSession这种需要设定过期时间的功能
- 发布、订阅消息系统
- 定时器、计数器

### 2.3 常用知识

- 默认16个数据库，类似数组下表从零开始，初始默认使用零号库，可在配置文件配置
- `select`命令切换数据库
- `dbsize`查看当前数据库的key的数量
- `flushdb`：清空当前库
- `flushall`；通杀全部库

## 3. Redis的五大数据类型

### 3.1 五大数据类型概述

1. **String（字符串）**
   - string是redis最基本的类型，一个key对应一个value。
   - string类型是二进制安全的。意思是redis的string可以包含任何数据。比如jpg图片或者序列化的对象 。
   - string类型是Redis最基本的数据类型，一个redis中字符串value最多可以是**512M**
2. **Hash（哈希、类似Java的map）**
   - Redis hash 是一个键值对集合。
   - Redis hash是一个string类型的field和value的映射表，hash特别适合用于存储对象。
   - 类似Java里面的Map<String,Object>。
3. **List（列表）**
   - Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）。
   - 它的底层实际是个链表。
4. **Set（集合）**
   - Redis的Set是string类型的无序集合。它是通过HashTable实现实现的
5. **Zset（sorted set：有序集合）**
   - Redis zset 和 set 一样也是string类型元素的集合，且不允许重复的成员。
   - 不同的是每个元素都会关联一个double类型的分数。
   - redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的，但分数(score)却可以重复。

> Redis的命令参考地址：http://redisdoc.com/

### 3.2 常用的key指令

| 命令            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
| keys *          | 查看当前数据库所有的key                                      |
| exists  key     | 判断某个key是否存在                                          |
| move key db_num | 将当前数据库的某个key移动到db-num数据库，当前数据库不再有这个key。 |
| expire key 秒钟 | 为给定的key设置过期时间                                      |
| ttl key         | 查看还有多少秒过期，-1表示永不过期，-2表示已过期             |
| type key        | 查看你的key是什么类型                                        |

### 3.3 String数据类型

| 命令            | 描述                                                         |
| --------------- | ------------------------------------------------------------ |
|  SET key value	| 设置指定 key 的值|
|  GET key	| 获取指定 key 的值。|
|  GETRANGE key start end	| 返回 key 中字符串值的子字符|
|  GETSET key value	| 将给定 key 的值设为 value ，并返回 key 的旧值(old value)。|
|  GETBIT key offset	| 对 key 所储存的字符串值，获取指定偏移量上的位(bit)。|
|  MGET key1 [key2…]	| 获取所有(一个或多个)给定 key 的值。|
|  SETBIT key offset value	| 对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。|
|  SETEX key seconds value	| 将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。|
|  SETNX key value	| 只有在 key 不存在时设置 key 的值。|
|  SETRANGE key offset value	| 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始。|
|  STRLEN key	| 返回 key 所储存的字符串值的长度。|
|  MSET key value [key value …]	| 同时设置一个或多个 key-value 对。|
|  MSETNX key value [key value …]	| 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在。|
|  PSETEX key milliseconds value	| 这个命令和 SETEX 命令相似，但它以毫秒为单位设置 key 的生存时间，而不是像 SETEX 命令那样，以秒为单位。|
|  INCR key	| 将 key 中储存的数字值增一。|
|  INCRBY key increment	| 将 key 所储存的值加上给定的增量值（increment） 。|
|  INCRBYFLOAT key increment	| 将 key 所储存的值加上给定的浮点增量值（increment） 。|
|  DECR key	| 将 key 中储存的数字值减一。|
|  DECRBY key decrement	| key 所储存的值减去给定的减量值（decrement） 。|
|  APPEND key value	| 如果 key 已经存在并且是一个字符串， APPEND 命令将指定的 value 追加到该 key 原来值（value）的末尾。 |

常用命令：

- set/get/del/append/strlen
- Incr/decr/incrby/decrby,一定要是数字才能进行加减
- getrange/setrange
- setex(set with expire)键秒值/setnx(set if not exist)
- mset/mget/msetnx
- getset(先get再set)

### ⭐️3.4 Hash数据类型

| 命令                                           | 描述                                                     |
| ---------------------------------------------- | -------------------------------------------------------- |
| HDEL key field1 [field2]                       | 删除一个或多个哈希表字段                                 |
| HEXISTS key field                              | 查看哈希表 key 中，指定的字段是否存在。                  |
| HGET key field                                 | 获取存储在哈希表中指定字段的值。                         |
| HGETALL key                                    | 获取在哈希表中指定 key 的所有字段和值                    |
| HINCRBY key field increment                    | 为哈希表 key 中的指定字段的整数值加上增量 increment 。   |
| HINCRBYFLOAT key field increment               | 为哈希表 key 中的指定字段的浮点数值加上增量 increment 。 |
| HKEYS key                                      | 获取所有哈希表中的字段                                   |
| HLEN key                                       | 获取哈希表中字段的数量                                   |
| HMGET key field1 [field2]                      | 获取所有给定字段的值                                     |
| HMSET key field1 value1 [field2 value2 ]       | 同时将多个 field-value (域-值)对设置到哈希表 key 中。    |
| HSET key field value                           | 将哈希表 key 中的字段 field 的值设为 value 。            |
| HSETNX key field value                         | 只有在字段 field 不存在时，设置哈希表字段的值。          |
| HVALS key                                      | 获取哈希表中所有值。                                     |
| HSCAN key cursor [MATCH pattern] [COUNT count] | 迭代哈希表中的键值对。                                   |

常用命令：

- hset/hget/hmset/hmget/hgetall/hdel
- hlen
- hexists key 在key里面的某个值的key
- hkeys/hvals
- hincrby/hincrbyfloat
- hsetnx



### 3.5 List数据类型

| 命令                                  | 描述                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| BLPOP key1 [key2 ] timeout            | 移出并获取列表的第一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。 |
| BRPOP key1 [key2 ] timeout            | 移出并获取列表的最后一个元素， 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。 |
| BRPOPLPUSH source destination timeout | 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它； 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止。 |
| LINDEX key index                      | 通过索引获取列表中的元素                                     |
| LINSERT key BEFORE/AFTER pivot value  | 在列表的元素前或者后插入元素                                 |
| LLEN key                              | 获取列表长度                                                 |
| LPOP key                              | 移出并获取列表的第一个元素                                   |
| LPUSH key value1 [value2]             | 将一个或多个值插入到列表头部                                 |
| LPUSHX key value                      | 将一个值插入到已存在的列表头部                               |
| LRANGE key start stop                 | 获取列表指定范围内的元素                                     |
| LREM key count value                  | 移除列表元素                                                 |
| LSET key index value                  | 通过索引设置列表元素的值                                     |
| LTRIM key start stop                  | 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。 |
| RPOP key                              | 移除列表的最后一个元素，返回值为移除的元素。                 |
| RPOPLPUSH source destination          | 移除列表的最后一个元素，并将该元素添加到另一个列表并返回     |
| RPUSH key value1 [value2]             | 在列表中添加一个或多个值                                     |
| RPUSHX key value                      | 为已存在的列表添加值                                         |

常用命令：

- lpush/rpush/lrange
- lpop/rpop
- lindex，按照索引下标获得元素(从上到下)
- llen
- lrem key 删N个value
- ltrim key 开始index 结束index，截取指定范围的值后再赋值给key
- rpoplpush 源列表 目的列表
- lset key index value
- linsert key before/after 值1 值2

性能总结：

- 它是一个字符串链表，left、right都可以插入添加；
- 如果键不存在，创建新的链表；
- 如果键已存在，新增内容；
- 如果值全移除，对应的键也就消失了。
- 链表的操作无论是头和尾效率都极高，但假如是对中间元素进行操作，效率就很惨淡了。

### 3.6 Set数据类型

| 命令                                           | 描述                                                |
| ---------------------------------------------- | --------------------------------------------------- |
| SADD key member1 [member2]                     | 向集合添加一个或多个成员                            |
| SCARD key                                      | 获取集合的成员数                                    |
| SDIFF key1 [key2]                              | 返回给定所有集合的差集                              |
| SDIFFSTORE destination key1 [key2]             | 返回给定所有集合的差集并存储在 destination 中       |
| SINTER key1 [key2]                             | 返回给定所有集合的交集                              |
| SINTERSTORE destination key1 [key2]            | 返回给定所有集合的交集并存储在 destination 中       |
| SISMEMBER key member                           | 判断 member 元素是否是集合 key 的成员               |
| SMEMBERS key                                   | 返回集合中的所有成员                                |
| SMOVE source destination member                | 将 member 元素从 source 集合移动到 destination 集合 |
| SPOP key                                       | 移除并返回集合中的一个随机元素                      |
| SRANDMEMBER key [count]                        | 返回集合中一个或多个随机数                          |
| SREM key member1 [member2]                     | 移除集合中一个或多个成员                            |
| SUNION key1 [key2]                             | 返回所有给定集合的并集                              |
| SUNIONSTORE destination key1 [key2]            | 所有给定集合的并集存储在 destination 集合中         |
| SSCAN key cursor [MATCH pattern] [COUNT count] | 迭代集合中的元素                                    |

常用命令：

- sadd/smembers/sismember
- scard，获取集合里面的元素个数
- srem key value 删除集合中元素
- srandmember key 某个整数(随机出几个数)
- spop key 随机出栈
- smove key1 key2 在key1里某个值 作用是将key1里的某个值赋给key2
- 数学集合类
- 差集：sdiff
- 交集：sinter
- 并集：sunion



### 3.7 Zset数据类型

在set基础上，加一个score值。 之前set是k1 v1 v2 v3， 现在zset是k1 score1 v1 score2 v2。

| 命令                                           | 描述                                                         |
| ---------------------------------------------- | ------------------------------------------------------------ |
| ZADD key score1 member1 [score2 member2]       | 向有序集合添加一个或多个成员，或者更新已存在成员的分数       |
| ZCARD key                                      | 获取有序集合的成员数                                         |
| ZCOUNT key min max                             | 计算在有序集合中指定区间分数的成员数                         |
| ZINCRBY key increment member                   | 有序集合中对指定成员的分数加上增量 increment                 |
| ZINTERSTORE destination numkeys key [key …]    | 计算给定的一个或多个有序集的交集并将结果集存储在新的有序集合 key 中 |
| ZLEXCOUNT key min max                          | 在有序集合中计算指定字典区间内成员数量                       |
| ZRANGE key start stop [WITHSCORES]             | 通过索引区间返回有序集合指定区间内的成员                     |
| ZRANGEBYLEX key min max [LIMIT offset count]   | 通过字典区间返回有序集合的成员                               |
| ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT] | 通过分数返回有序集合指定区间内的成员                         |
| ZRANK key member                               | 返回有序集合中指定成员的索引                                 |
| ZREM key member [member …]                     | 移除有序集合中的一个或多个成员                               |
| ZREMRANGEBYLEX key min max                     | 移除有序集合中给定的字典区间的所有成员                       |
| ZREMRANGEBYRANK key start stop                 | 移除有序集合中给定的排名区间的所有成员                       |
| ZREMRANGEBYSCORE key min max                   | 移除有序集合中给定的分数区间的所有成员                       |
| ZREVRANGE key start stop [WITHSCORES]          | 返回有序集中指定区间内的成员，通过索引，分数从高到低         |
| ZREVRANGEBYSCORE key max min [WITHSCORES]      | 返回有序集中指定分数区间内的成员，分数从高到低排序           |
| ZREVRANK key member                            | 返回有序集合中指定成员的排名，有序集成员按分数值递减(从大到小)排序 |
| ZSCORE key member                              | 返回有序集中，成员的分数值                                   |
| ZUNIONSTORE destination numkeys key [key …]    | 计算给定的一个或多个有序集的并集，并存储在新的 key 中        |
| ZSCAN key cursor [MATCH pattern] [COUNT count] | 迭代有序集合中的元素（包括元素成员和元素分值）               |

常用命令：

- zadd/zrange
  - Withscores
- zrangebyscore key 开始score 结束score
  - withscores
  - ( 不包含
  - Limit 作用是返回限制
    - limit 开始下标步 多少步
- zrem key 某score下对应的value值，作用是删除元素
- zcard/zcount key score区间/zrank key values值，作用是获得下标值/zscore key 对应值,获得分数
- zrevrank key values值，作用是逆序获得下标值
- zrevrange
- zrevrangebyscore key 结束score 开始score



## 4. Redis配置文件介绍

Redis 的配置文件位于 Redis 安装目录下，在Redis客户端，可以通过`config`命令查看或者设置配置项。

**语法如下：**

```bash
redis 127.0.0.1:6379> CONFIG GET CONFIG_SETTING_NAME
```

Redis的配置文件常用参数说明：

配置项	说明

1. **daemonize no**	Redis 默认不是以守护进程的方式运行，可以通过该配置项修改，使用 yes 启用守护进程（Windows 不支持守护线程的配置为 no ）

2. **pidfile /var/run/redis.pid**	当 Redis 以守护进程方式运行时，Redis 默认会把 pid 写入 /var/run/redis.pid 文件，可以通过 pidfile 指定

3. port 6379	指定 Redis 监听端口，默认端口为 6379，作者在自己的一篇博文中解释了为什么选用 6379 作为默认端口，因为 6379 在手机按键上 MERZ 对应的号码，而 MERZ 取自意大利歌女 Alessia Merz 的名字

4. **bind 127.0.0.1**	绑定的主机地址

5. **timeout 300**	当客户端闲置多长秒后关闭连接，如果指定为 0 ，表示关闭该功能

6. **loglevel notice**	指定日志记录级别，Redis 总共支持四个级别：debug、verbose、notice、warning，默认为 notice

7. **logfile stdout**	日志记录方式，默认为标准输出，如果配置 Redis 为守护进程方式运行，而这里又配置为日志记录方式为标准输出，则日志将会发送给 /dev/null

8. databases 16	设置数据库的数量，默认数据库为0，可以使用SELECT 命令在连接上指定数据库id

9. **save <seconds> <changes>**
   Redis 默认配置文件中提供了三个条件：
   
   - save 900 1
   - save 300 10
   - save 60 10000分别表示 900 秒（15 分钟）内有 1 个更改，300 秒（5 分钟）内有 10 个更改以及 60 秒内有 10000 个更改。
   
   > 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
   >     
   
10. **rdbcompression yes**	指定存储至本地数据库时是否压缩数据，默认为 yes，Redis 采用 LZF 压缩，如果为了节省 CPU 时间，可以关闭该选项，但会导致数据库文件变的巨大

11. dbfilename dump.rdb	指定本地数据库文件名，默认值为 dump.rdb

12. **dir ./**	指定本地数据库存放目录
    
13. **slaveof <masterip> <masterport>**	设置当本机为 slave 服务时，设置 master 服务的 IP 地址及端口，在 Redis 启动时，它会自动从 master 进行数据同步

14. **masterauth <master-password>**	当 master 服务设置了密码保护时，slav 服务连接 master 的密码

15. **requirepass foobared**	设置 Redis 连接密码，如果配置了连接密码，客户端在连接 Redis 时需要通过 AUTH 命令提供密码，默认关闭

16. maxclients 128	设置同一时间最大客户端连接数，默认无限制，Redis 可以同时打开的客户端连接数为 Redis 进程可以打开的最大文件描述符数，如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis 会关闭新的连接并向客户端返回 max number of clients reached 错误信息

17. **maxmemory <bytes>**	指定 Redis 最大内存限制，Redis 在启动时会把数据加载到内存中，达到最大内存后，Redis 会先尝试清除已到期或即将到期的 Key，当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis 新的 vm 机制，会把 Key 存放内存，Value 会存放在 swap 区

18. appendonly no	指定是否在每次更新操作后进行日志记录，Redis 在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。因为 redis 本身同步数据文件是按上面 save 条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为 no

19. **appendfilename appendonly.aof**	指定更新日志文件名，默认为 appendonly.aof

20. **appendfsync everysec**	指定更新日志条件，共有 3 个可选值

    - no：表示等操作系统进行数据缓存同步到磁盘（快）
    -  always：表示每次更新操作后手动调用 fsync() 将数据写到磁盘（慢，安全）
    -  everysec：表示每秒同步一次（折中，默认值）

21. **vm-enabled no**	指定是否启用虚拟内存机制，默认值为 no，简单的介绍一下，VM 机制将数据分页存放，由 Redis 将访问量较少的页即冷数据 swap 到磁盘上，访问多的页面由磁盘自动换出到内存中（在后面的文章我会仔细分析 Redis 的 VM 机制）

22. **vm-swap-file /tmp/redis.swap**	虚拟内存文件路径，默认值为 /tmp/redis.swap，不可多个 Redis 实例共享

23. **vm-max-memory 0**	将所有大于 vm-max-memory 的数据存入虚拟内存，无论 vm-max-memory 设置多小，所有索引数据都是内存存储的(Redis 的索引数据 就是 keys)，也就是说，当 vm-max-memory 设置为 0 的时候，其实是所有 value 都存在于磁盘。默认值为 0

24. vm-page-size 32	Redis swap 文件分成了很多的 page，一个对象可以保存在多个 page 上面，但一个 page 上不能被多个对象共享，vm-page-size 是要根据存储的 数据大小来设定的，作者建议如果存储很多小对象，page 大小最好设置为 32 或者 64bytes；如果存储很大大对象，则可以使用更大的 page，如果不确定，就使用默认值

25. **vm-pages 134217728**	设置 swap 文件中的 page 数量，由于页表（一种表示页面空闲或使用的 bitmap）是在放在内存中的，，在磁盘上每 8 个 pages 将消耗 1byte 的内存。

26. vm-max-threads 4	设置访问swap文件的线程数,最好不要超过机器的核数,如果设置为0,那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟。默认值为4

27. **glueoutputbuf yes**	设置在向客户端应答时，是否把较小的包合并为一个包发送，默认为开启

28. **hash-max-zipmap-entries 64**
    **hash-max-zipmap-value 512**	

    指定在超过一定的数量或者最大的元素超过某一临界值时，采用一种特殊的哈希算法

29. **activerehashing yes**	指定是否激活重置哈希，默认为开启（后面在介绍 Redis 的哈希算法时具体介绍）

30. **include /path/to/local.conf**	指定包含其它的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各个实例又拥有自己的特定配置文件



## 5. Redis持久化之RDB

### 5.1 什么是RDB？

- RDB（Redis Database）会在指定的时间间隔内将内存中的数据集快照写入磁盘，也就是行话讲的Snapshot快照，它恢复时是将快照文件直接读到内存里。

- Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到 一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。 整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能。

  > Fork
  >
  > Fork的作用是复制一个与当前进程一样的进程。新进程的所有数据（变量、环境变量、程序计数器等） 数值都和原进程一致，但是是一个全新的进程，并作为原进程的子进程

- 如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。
- rdb 保存的是dump.rdb文件
- 相关配置在配置文件的位置 - 在redis.conf搜寻`### SNAPSHOTTING ###`

### 5.2 如何触发RDB快照

- 配置文件中默认的快照配置dbfilename dump.rdb，冷拷贝后重新使用
  - 可以`cp dump.rdb dump_new.rdb`
- 命令save或者是bgsave
  - Save：save时只管保存，其它不管，全部阻塞
  - BGSAVE：Redis会在后台异步进行快照操作， 快照同时还可以响应客户端请求。可以通过lastsave 命令获取最后一次成功执行快照的时间
- 执行flushall命令，也会产生dump.rdb文件，但里面是空的，无意义

### 5.3 如何恢复数据

- 将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可
- `CONFIG GET dir`获取目录

### 5.4 优势和劣势

- 优势
  - 适合大规模的数据恢复
  - 对数据完整性和一致性要求不高
- 劣势
  - 在一定间隔时间做一次备份，所以如果redis意外down掉的话，就会丢失最后一次快照后的所有修改
  - Fork的时候，内存中的数据被克隆了一份，大致2倍的膨胀性需要考虑

### 5.5 如何停止RDB

动态所有停止RDB保存规则的方法：`redis-cli config set save ""`

### 5.6 小结

![img](http://cdn.bearkchan.top/format,png.png)

- RDB是一个非常紧凑的文件。
- RDB在保存RDB文件时父进程唯一需要做的就是fork出一个子进程，接下来的工作全部由子进程来做，父进程不需要再做其他I0操作，所以RDB持久化方式可以最大化redis的性能。
- 与AOF相比，在恢复大的数据集的时候，RDB方式会更快一一些。
- 数据丢失风险大。
- RDB需要经常fork子进程来保存数据集到硬盘上，当数据集比较大的时候fork的过程是非常耗时的吗，可能会导致Redis在一些毫秒级不能回应客户端请求。

## 6. Redis持久化之AOF

### 6.1 什么是AOF？

AOF（Append Only File）是以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来(读操作不记录)， **只许追加文件但不可以改写文件**，redis启动之初会读取该文件重新构建数据，换言之，**redis 重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作**。

### 6.2 AOF配置

- 相关配置在配置文件的位置 - 在redis.conf搜寻`### APPEND ONLY MODE ###`
- aof保存的是appendonly.aof文件（在配置文件可修改文件名）

### 6.3 AOF启动/恢复/修复

- 启动：
  - 修改默认的appendonly no，改为yes
- 恢复AOP文件：
  - 将有数据的append_only.aof文件复制一份到对应的目录。重启redis然后重新加载。
- 异常恢复被写坏的AOF文件：
  - 使用`redis-check-aof --fix`命令进行恢复aof文件，然后重启redis重新加载aof文件。

### 6.4 rewrite

AOF采用文件追加模式，文件会越来越大。为了避免这种情况，新增了重写机制，当AOF文件的大小超过所设定的阈值时，Redis就会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集。

**重写原理：**

AOF文件持续增长而过大时，会fork出一条新进程来将文件重写(也是先写临时文件最后再rename)， 遍历新进程的内存中数据，每条记录有一条的Set语句。重写aof文件的操作，并没有读取旧的aof文件， 而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这点和快照有点类似

**触发机制：**

Redis会记录上次重写时的AOF大小，默认配置是当**AOF文件大小是上次rewrite后大小的一倍且文件大于64M**时触发

### 6.5 优势和劣势

- 优势
  - **每修改同步：appendfsync always 同步持久化** 每次发生数据变更会被立即记录到磁盘 性能较差但数据完整性比较好
  - **每秒同步：appendfsync everysec 异步操作**每秒记录 如果一秒内宕机，有数据丢失
  - **不同步：appendfsync no 从不同步**
- 劣势
  - 相同数据集的数据而言aof文件要远大于rdb文件，恢复速度慢于rdb
  - Aof运行效率要慢于rdb,每秒同步策略效率较好，不同步效率和rdb相同

### 6.6 小结

![img](http://cdn.bearkchan.top/format,png-20210308195905697.png)

- AOF文件是一个只进行追加的日志文件
- Redis可以在AOF文件体积变得过大时，自动地在后台对AOF进行重写
- AOF文件有序地保存了对数据库执行的所有写入操作，这些写入操作以Redis协议的格式保存，因此AOF文件的内容非常容易被人读懂，对文件进行分析也很轻松
- 对于相同的数据集来说，AOF文件的体积通常要大于RDB文件的体积
- 根据所使用的fsync 策略，AOF的速度可能会慢于RDB

> 当Redis中同时开启了RDB和AOF持久化策略时，Redis恢复时会默认使用AOF文件，如果AOF文件出现错误，则会直接提示数据恢复失败，而不会自动的去加载RDB数据。

## 7. Redis中的事务







## Redis常见问题

### 1. 缓存雪崩

![image-20210309052543310](http://cdn.bearkchan.top/image-20210309052543310.png)

大量缓存数据同时时间失效，导致用户直接发起大量的请求到数据库，产生瓶颈。

**解决方案：**

- 生成随机失效的缓存时间数据。
- 让缓存节点分布在不同的物理节点上。
- 生成不失效的缓存数据。
- 定时任务更新缓存数据。



### 2. 缓存穿透（攻击）

![image-20210309052740819](http://cdn.bearkchan.top/image-20210309052740819.png)

用户恶意请求数据，例如ID为负数，不存在缓存中，夜不存在在数据库中，会造成缓存穿透攻击。

**解决方案：**

- 无意义的数据也放入缓存中，下一次相同的请求就会命中缓存。
- IP过滤。
- 参数校验。
- 布隆过滤器。

### 3.缓存（热点）击穿

![image-20210309052804926](http://cdn.bearkchan.top/image-20210309052804926.png)

由于缓存热点key到时间失效导致用户直接请求访问数据库。

**解决方案：**

- 永久缓存
- 分布式锁
  - zookeeper
  - redis实现