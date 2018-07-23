### Redis概述篇 ###
***

![](https://i.imgur.com/YTSXrng.png)

### 一、Redis简介 ###

Redis是一个**完全开源免费的、遵守BSD协议、使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、高性能的Key-Value数据库**，并提供多种语言的API。和Memcached类似，但它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）等。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。

与memcached一样，为了保证效率，Redis采用了**内存中（in-memory）数据集（dataset）的方式** ,即数据都是缓存在内存中的。同时，Redis支持数据的**持久化**，你可以每间隔一段时间将数据集转存到磁盘上（snapshot）或者在日志尾部追加每一条操作命令(append only file,aof).Redis 同样支持主从复制（master-slave replication）,并且具有非常快速的非阻塞首次同步（non_locking first synchronization）、网络断开自动重连等功能。同时Redis还具有其他的一些特性，其中包括简单的事务支持、发布订阅(pub/sub)、管道（pipeline）和虚拟内存(vm)等

除此之外，**Redis 还内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）**。


redis的出现，很大程度补偿了memcached这类key/value存储的不足，在部分场合可以对关系数据库起到很好的补充作用。它提供了Java，C/C++，C#，PHP，JavaScript，Perl，Object-C，Python，Ruby，Erlang等客户端，使用很方便。

#### 1.Redis特性（Feature） ####

- **性能极高（所有数据都在内存中）** ：Redis能读的速度是110000次/s,写的速度是81000次/s ;
- **丰富的数据类型**：Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储；
- **丰富的功能** ：缓存、队列、消息发布订阅、支持key的生存时间（数据过期时间支持）、按照一定的规则删除相应的键、不完全的事务支持等；
- **原子性**： Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行；
- **内存存储与持久化** ：Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用；
- **主从复制**：Redis支持数据的备份，即master-slave模式的数据备份。


#### 2.Redis版本说明 ####

Redis的版本规则如下————次版本号（第一个小数点后的数字）为偶数的版本是稳定版本（2.4、2.6等），奇数为非稳定版本（2.5、2.7），一般推荐在生产环境使用稳定版本。（说明：Redis官方是不支持windows平台的，windows版本是由微软自己建立的分支，基于官方的Redis源码上进行编译、发布、维护的，所以windows平台的Redis版本要略低于官方版本。）


#### 3.主流NSQL比较 ####

![](https://i.imgur.com/IwcXHdQ.png)

![](https://i.imgur.com/Gif1TRG.png)

Redis VS Memcached ：

- **数据类型支持不同** ———— memcached 仅支持简单的key-value结构的数据类型---String；Redis支持丰富的数据类型：String、Hash、List、Set和Sorted Set.
- **数据持久化支持** ———— memcached 所有数据都存储在内存中，不支持数据的持久化，所以一般只用作缓存；Redis 虽然是基于内存的存储系统，但是它本身是支持内存数据的持久化的，而且提供两种主要的持久化策略：RDB快照和AOF日志，所以Redis既可以用作缓存，也可以用作存储。
- **主从复制**（master-slave replication）———— Redis支持数据的复制（备份），即master-slave模式的数据复制（备份）；memcached则不支持。
- **消息订阅发布**（pub/sub）———— Redis实现了消息的订阅/发布功能，memcached则没有。
- **集群管理的不同**

memcached本身并不支持分布式，因此只能在客户端通过像一致性哈希算法这样的分布式算法来实现memcached的分布式存储。下图给出了Memcached的分布式存储实现架构。当客户端向Memcached集群发送数据之前，首先会通过内置的分布式算法计算出该条数据的目标节点，然后数据会直接发送到该节点上存储。但客户端查询数据时，同样要计算出查询数据所在的节点，然后直接向该节点发送查询请求以获取数据。

![](http://www.biaodianfu.com/wp-content/uploads/2014/01/Memcached-node.jpg)

相较于Memcached只能采用客户端实现分布式存储，Redis更偏向于在服务器端构建分布式存储。较新版本的Redis（从3.0版本开始）已经支持了分布式存储功能。Redis Cluster是一个实现了分布式且允许单点故障的Redis高级版本，它没有中心节点，具有线性可伸缩的功能。下图给出Redis Cluster的分布式存储架构，其中节点与节点之间通过二进制协议进行通信，节点与客户端之间通过ascii协议进行通信。在数据的放置策略上，Redis Cluster将整个key的数值域分成4096个哈希槽，每个节点上可以存储一个或多个哈希槽，也就是说当前Redis Cluster支持的最大节点数就是4096。Redis Cluster使用的分布式算法也很简单：**crc16( key ) % HASH_SLOTS_NUMBER**。

![](http://www.biaodianfu.com/wp-content/uploads/2014/01/Redis-Cluster.jpg)

- **内存管理机制不同** ———— memcached使用预分配的内存池的方式，利用Slab Allocation 机制管理内存，其主要思想是按照预先规定的大小，将分配的内存分割成特定长度的块以存储相应长度的key-value数据记录，以完全解决内存碎片问题；Redis使用现场申请内存的方式来存储数据，并且很少使用free-list等方式来优化内存分配，会在一定程度上存在内存碎片，Redis根据存储命令参数，会把带过期时间的数据单独存放在一起，并把它们称为临时数据，非临时数据是永远不会被剔除的，即便物理内存不够，导致swap也不会剔除任何非临时数据(但会尝试剔除部分临时数据)，这点上Redis更适合作为存储而不是cache。

（1）Memcached内存管理机制

Memcached默认使用Slab Allocation机制管理内存，其主要思想是按照预先规定的大小，将分配的内存分割成特定长度的块以存储相应长度的key-value数据记录，以完全解决内存碎片问题。Slab Allocation机制只为存储外部数据而设计，也就是说所有的key-value数据都存储在Slab Allocation系统里，而Memcached的其它内存请求则通过普通的malloc/free来申请，因为这些请求的数量和频率决定了它们不会对整个系统的性能造成影响。Slab Allocation的原理相当简单， 如图所示，它首先从操作系统申请一大块内存，并将其分割成各种尺寸的块Chunk，并把尺寸相同的块分成组Slab Class。其中，Chunk就是用来存储key-value数据的最小单位。每个Slab Class的大小，可以在Memcached启动的时候通过制定Growth Factor来控制。假定图中Growth Factor的取值为1.25，如果第一组Chunk的大小为88个字节，第二组Chunk的大小就为112个字节，依此类推。

![](http://www.biaodianfu.com/wp-content/uploads/2014/01/Slab-Allocation.jpg)

当Memcached接收到客户端发送过来的数据时首先会根据收到数据的大小选择一个最合适的Slab Class，然后通过查询Memcached保存着的该Slab Class内空闲Chunk的列表就可以找到一个可用于存储数据的Chunk。当一条数据库过期或者丢弃时，该记录所占用的Chunk就可以回收，重新添加到空闲列表中。从以上过程我们可以看出Memcached的内存管理制效率高，而且不会造成内存碎片，但是它最大的缺点就是会导致空间浪费。因为每个Chunk都分配了特定长度的内存空间，所以变长数据无法充分利用这些空间。如图所示，将100个字节的数据缓存到128个字节的Chunk中，剩余的28个字节就浪费掉了。

![](http://www.biaodianfu.com/wp-content/uploads/2014/01/Chunk.png)


（2）Redis内存管理机制

Redis的内存管理主要通过源码中zmalloc.h和zmalloc.c两个文件来实现的。Redis为了方便内存的管理，在分配一块内存之后，会将这块内存的大小存入内存块的头部。如图所示，real_ptr是redis调用malloc后返回的指针。redis将内存块的大小size存入头部，size所占据的内存大小是已知的，为size_t类型的长度，然后返回ret_ptr。当需要释放内存的时候，ret_ptr被传给内存管理程序。通过ret_ptr，程序可以很容易的算出real_ptr的值，然后将real_ptr传给free释放内存。

![](http://www.biaodianfu.com/wp-content/uploads/2014/01/zmalloc.png)

总的来看，Redis采用的是包装的mallc/free，相较于Memcached的内存管理方法来说，要简单很多。

（在Redis中，并不是所有的数据都一直存储在内存中的。这是和Memcached相比一个最大的区别。当物理内存用完时，Redis可以将一些很久没用到的value交换到磁盘。Redis只会缓存所有的key的信息，如果Redis发现内存的使用量超过了某一个阀值，将触发swap的操作，Redis根据“swappability = age*log(size_in_memory)”计算出哪些key对应的value需要swap到磁盘。然后再将这些key对应的value持久化到磁盘中，同时在内存中清除。这种特性使得Redis可以保持超过其机器本身内存大小的数据。当然，机器本身的内存必须要能够保持所有的key，毕竟这些数据是不会进行swap操作的。同时由于Redis将内存中的数据swap到磁盘中的时候，提供服务的主线程和进行swap操作的子线程会共享这部分内存，所以如果更新需要swap的数据，Redis将阻塞这个操作，直到子线程完成swap操作后才可以进行修改。当从Redis中读取数据的时候，如果读取的key对应的value不在内存中，那么Redis就需要从swap文件中加载相应数据，然后再返回给请求方。 这里就存在一个I/O线程池的问题。在默认的情况下，Redis会出现阻塞，即完成所有的swap文件加载后才会相应。这种策略在客户端的数量较小，进行批量操作的时候比较合适。但是如果将Redis应用在一个大型的网站应用程序中，这显然是无法满足大并发的情况的。所以Redis运行我们设置I/O线程池的大小，对需要从swap文件中加载相应数据的读取请求进行并发操作，减少阻塞的时间。）


- **网络IO模型不同** ———— memcached 是多线程、非阻塞IO复用的网络模型，分为监听主线程（master）和工作子线程（worker）.主线程监听端口、建立连接，然后顺序分配给各个工作线程。每个工作线程有一个event loop，它们服务不同的客户端；而Redis 使用单线程的非阻塞IO多路复用的网络模型。


### 二、Redis 数据类型 ###

![](https://i.imgur.com/P5V49Gf.png)

与Memcached仅支持简单的key-value结构的数据记录不同，Redis支持的数据类型要丰富得多。最为常用的数据类型主要由五种：String、Hash、List、Set和Sorted Set。Redis内部使用一个redisObject对象来表示所有的key和value。redisObject最主要的信息如图所示：


![](http://www.biaodianfu.com/wp-content/uploads/2014/01/redisObject.jpg)

（type代表一个value对象具体是何种数据类型，encoding是不同数据类型在redis内部的存储方式，比如：type=string代表value存储的是一个普通字符串，那么对应的encoding可以是raw或者是int，如果是int则代表实际redis内部是按数值型类存储和表示这个字符串的，当然前提是这个字符串本身可以用数值表示，比如:”123″ “456”这样的字符串。只有打开了Redis的虚拟内存功能，vm字段字段才会真正的分配内存，该功能默认是关闭状态的。）

![](https://i.imgur.com/sdEgvPC.png)

#### 1.String（字符串）  ####

Redis没有直接使用C语言传统的字符串表示（以空字符结尾的字符数组，以下简称C字符串），而是自己构建了一种名为简单动态字符串（simple dynamic string,SDS）的抽象类型，并将SDS用作Redis的默认字符串表示。下面是SDS的定义：

	struct sdshdr { 
		long len; //len是buf数组的长度。
		long free;//free是数组中剩余可用字节数。
		char buf[];//buf是个char数组用于存贮实际的字符串内容
	 };

![](https://i.imgur.com/9OemM7f.png)


- 常用命令：set/setnx/setex/setrange/mset/msetnx/get/getset/getrange/mget/decr/decrby/incr/incrby/append/strlen等；(详细请参考：[Redis命令参考](http://redisdoc.com/))
- 应用场景：String是最常用的一种数据类型，普通的key/value存储都可以归为此类；
- 实现方式：String在redis内部存储默认就是一个字符串，被redisObject所引用，当遇到incr、decr等操作时会转成数值型进行计算，此时redisObject的encoding字段为int。


#### 2.Hash （哈希表） ####

Redis hash是一个String类型的field和value的映射表。Key-HashMap结构，相比String类型将这整个对象持久化成JSON格式，Hash将对象的各个属性存入Map里，可以只读取/更新对象的某些属性————当前HashMap的实现有两种方式：当HashMap的成员比较少时Redis为了节省内存会采用类似一维数组的方式来紧凑存储，而不会采用真正的HashMap结构，这时对应的value的redisObject的encoding为zipmap，当成员数量增大时会自动转成真正的HashMap,此时encoding为ht。



- 常用命令：hset/hsetnx/hmset/hget/hmget/hincrby/hexists/hlen/hdel/hkeys/hvals/hgetall等;(详细请参考：[Redis命令参考](http://redisdoc.com/))
- 应用场景：我们要存储一个用户信息对象数据，其中包括用户ID、用户姓名、年龄和生日，通过用户ID我们希望获取该用户的姓名或者年龄或者生日；
- 实现方式：Redis的Hash实际是内部存储的Value为一个HashMap，并提供了直接存取这个Map成员的接口。如图所示，Key是用户ID, value是一个Map。这个Map的key是成员的属性名，value是属性值。这样对数据的修改和存取都可以直接通过其内部Map的Key(Redis里称内部Map的key为field), 也就是通过 key(用户ID) + field(属性标签) 就可以操作对应属性数据。

![](http://www.biaodianfu.com/wp-content/uploads/2014/01/hash.jpg)


#### 3.List（列表）  ####

Redis 中的列表是一个有序的字符串集合，您可以向其中添加任意数量的（惟一或非惟一）元素。除了向列表添加元素和从中获取元素的操作之外，Redis 还支持对列表使用取出、推送、范围和其他一些操作。（实质就是一个每个子元素都是string类型的双向链表（链表的最大长度是2的32次方）操作中key可以理解为链表的名字。）


- 常用命令：lpush/rpush/lpop/linsert/lset/lrem/ltrim/lpop/rpop/rpoplpush/lindex/llen/lrange等；(详细请参考：[Redis命令参考](http://redisdoc.com/))
- 应用场景：Redis list的应用场景非常多，也是Redis最重要的数据结构之一，比如twitter的关注列表，粉丝列表等都可以用Redis的list结构来实现；
- 实现方式：Redis list的实现为一个双向链表，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销，Redis内部的很多实现，包括发送缓冲队列等也都是用的这个数据结构。




#### 4.Set（集合） ####

集合（set）是惟一元素的无序集合。

- 常用命令：sadd/srem/spop/sdiff/sdiffstore/sinter/sinterstore/smove/scard/smembers/sismember/srandmember/sunion/sunionstore等；(详细请参考：[Redis命令参考](http://redisdoc.com/))
- 应用场景：Redis set对外提供的功能与list类似是一个列表的功能，特殊之处在于set是可以自动排重的，当你需要存储一个列表数据，又不希望出现重复数据时，set是一个很好的选择，并且set提供了判断某个成员是否在一个set集合内的重要接口，这个也是list所不能提供的；
- 实现方式：set 的内部实现是一个value永远为null的HashMap，实际就是通过计算hash的方式来快速排重的，这也是set能提供判断一个成员是否在集合内的原因。




#### 5.Sorted Set（有序集合） ####

有序集是可基于一个分数进行排序并且仅包含惟一元素的集合。每次您向集合中添加一个值，您都会提供一个用于排序的分数。

- 常用命令：zadd/zrem/zincrby/zrank/zrevrank/zrevrange/zrange/zrangebyscore/zcount/zcard/zscore/zrem/zremrangebyrank等；(详细请参考：[Redis命令参考](http://redisdoc.com/))
- 应用场景：Redis sorted set的使用场景与set类似，区别是set不是自动有序的，而sorted set可以通过用户额外提供一个优先级(score)的参数来为成员排序，并且是插入有序的，即自动排序。当你需要一个有序的并且不重复的集合列表，那么可以选择sorted set数据结构，比如twitter 的public timeline可以以发表时间作为score来存储，这样获取时就是自动按时间排好序的。
- 实现方式：Redis sorted set的内部使用HashMap和跳跃表(SkipList)来保证数据的存储和有序，HashMap里放的是成员到score的映射，而跳跃表里存放的是所有的成员，排序依据是HashMap里存的score,使用跳跃表的结构可以获得比较高的查找效率，并且在实现上比较简单。


附录：

- [通俗易懂的Redis数据结构基础教程](https://juejin.im/post/5b53ee7e5188251aaa2d2e16)
- [Redis底层原理](https://blog.csdn.net/wcf373722432/article/details/78678504)

### 三、Redis 持久化方式 ###

### 三、Redis Java Client ———— Jedis ###

Redis支持的几种不同语言客户端：

![](https://i.imgur.com/zdHUAwQ.png)

Jedis是一个小而全的Redis Java Client，很容易使用。先来看下Jedis architectures：

![](https://i.imgur.com/F975oXq.jpg)

#### 1.Jedis | ShardedJedis | JedisCluster ####

	public class Jedis extends BinaryJedis implements JedisCommands,MultiKeyCommands,AdvancedJedisCommands,ScriptingCommands,BasicCommands,ClusterCommands,SentinelCommands
	public class SharededJedis extends BinaryShardedJedis implements JedisCommands,Closeable	
	public class JedisCluster implements JedisCommands,BasicCommands,Closeable


![](https://i.imgur.com/10oEhaL.png)

#### 2.Jedis Pool ####

![](https://i.imgur.com/rr0vp9Z.png)

![](https://i.imgur.com/6s3gpSJ.png)


附：


- [Redis官网](https://redis.io/)
- [Redis中文官方网站](http://www.redis.cn/)
- [Redis源码脱管](https://github.com/antirez/redis)
- [Jedis源码托管](https://github.com/xetorthio/jedis)
- [Redis设计与实现](http://redisbook.readthedocs.io/en/latest/)
- [Redis命令参考](http://redisdoc.com/)
- [Redis教程](http://www.runoob.com/redis/redis-tutorial.html)






































