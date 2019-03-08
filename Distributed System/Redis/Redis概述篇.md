### Redis概述篇 ###
***

![](https://i.imgur.com/YTSXrng.png)

### 一、Redis简介 ###

Redis（「Remote Dictionary Service」远程字典服务）是一个**完全开源免费的、遵守BSD协议、使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、高性能的Key-Value数据库**，并提供多种语言的API。和Memcached类似，但它支持存储的value类型相对更多，包括string(字符串)、list(链表)、set(集合)、zset(sorted set --有序集合)和hash（哈希类型）等。这些数据类型都支持push/pop、add/remove及取交集并集和差集及更丰富的操作，而且这些操作都是原子性的。在此基础上，redis支持各种不同方式的排序。

与memcached一样，为了保证效率，Redis采用了**内存中（in-memory）数据集（dataset）的方式** ,即数据都是缓存在内存中的。同时，Redis支持数据的**持久化**，你可以每间隔一段时间将数据集转存到磁盘上（snapshot）或者在日志尾部追加每一条操作命令(append only file,aof).Redis 同样支持主从复制（master-slave replication）,并且具有非常快速的非阻塞首次同步（non_locking first synchronization）、网络断开自动重连等功能。同时Redis还具有其他的一些特性，其中包括简单的事务支持、发布订阅(pub/sub)、管道（pipeline）和虚拟内存(vm)等

除此之外，**Redis 还内置了 复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（high availability）**。


redis的出现，很大程度补偿了memcached这类key/value存储的不足，在部分场合可以对关系数据库起到很好的补充作用。它提供了Java，C/C++，C#，PHP，JavaScript，Perl，Object-C，Python，Ruby，Erlang等客户端，使用很方便。

#### 1.Redis特性（面试题：使用redis有哪些好处？） ####

- **性能极高（所有数据都在内存中）** ：Redis能读的速度是110000次/s,写的速度是81000次/s ;
- **丰富的数据类型**：Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储；
- **丰富的功能** ：缓存、队列、消息发布订阅、支持key的生存时间（数据过期时间支持）、按照一定的规则删除相应的键、不完全的事务支持等；
- **原子性**： Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行；
- **内存存储与持久化** ：Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用；
- **[主从复制](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/Redis/Redis%E4%BD%BF%E7%94%A8%E7%AF%87%E2%80%94%E2%80%94Redis%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%EF%BC%88master-slave%20replication%EF%BC%89.md)**：Redis支持数据的备份，即master-slave模式的数据备份。


#### 2.Redis有什么缺点？ ####


- 由于 Redis 是内存数据库，所以，单台机器，存储的数据量，受机器本身的内存大小限制。虽然 Redis 本身有 Key 过期策略，但是还是需要提前预估和节约内存。如果内存增长过快，需要定期删除数据。（可使用 Redis Cluster、Codis 等方案，对 Redis 进行分区，从单机 Redis 变成集群 Redis 。）
- 如果进行完整重同步，由于需要生成 RDB 文件，并进行传输，会占用主机的 CPU ，并会消耗现网的带宽。不过 Redis2.8 版本，已经有部分重同步的功能，但是还是有可能有完整重同步的。比如，新上线的备机。
- 修改配置文件，进行重启，将硬盘中的数据加载进内存，时间比较久。在这个过程中，Redis 不能提供服务。


#### 3.Redis版本说明 ####

Redis的版本规则如下————次版本号（第一个小数点后的数字）为偶数的版本是稳定版本（2.4、2.6等），奇数为非稳定版本（2.5、2.7），一般推荐在生产环境使用稳定版本。（说明：Redis官方是不支持windows平台的，windows版本是由微软自己建立的分支，基于官方的Redis源码上进行编译、发布、维护的，所以windows平台的Redis版本要略低于官方版本。）


#### 4.主流NSQL比较 ####

![](https://i.imgur.com/IwcXHdQ.png)

![](https://i.imgur.com/Gif1TRG.png)

**Redis VS Memcache(面试题：Memcache与Redis的区别都有哪些？) ：**

- **数据类型支持不同** ———— memcached 仅支持简单的key-value结构的数据类型---String；Redis支持丰富的数据类型：String、Hash、List、Set和Sorted Set.
- **数据持久化支持** ———— memcached 所有数据都存储在内存中，不支持数据的持久化，所以一般只用作缓存；Redis 虽然是基于内存的存储系统，但是它本身是支持内存数据的持久化的，而且提供两种主要的持久化策略：RDB快照和AOF日志，所以Redis既可以用作缓存，也可以用作存储。
- **性能对比** ———— Redis 只使用单核，而 Memcached 可以使用多核，所以平均每一个核上 Redis在存储小数据时比 Memcached 性能更高；在 100k 以上的数据中，Memcached 性能要高于 Redis 。虽然 Redis 最近也在存储大数据的性能上进行优化，但是比起 Memcached，还是稍有逊色。
- **主从复制**（master-slave replication）———— Redis支持数据的复制（备份），即master-slave模式的数据复制（备份）；memcached则不支持。
- **消息订阅发布**（pub/sub）———— Redis实现了消息的订阅/发布功能，memcached则没有。
- **value大小** ———— redis最大可以达到1GB，而memcache只有1MB
- **集群管理的不同**

memcached本身并不支持分布式，因此只能在客户端通过像一致性哈希算法这样的分布式算法来实现memcached的分布式存储。下图给出了Memcached的分布式存储实现架构。当客户端向Memcached集群发送数据之前，首先会通过内置的分布式算法计算出该条数据的目标节点，然后数据会直接发送到该节点上存储。当客户端查询数据时，同样要计算出查询数据所在的节点，然后直接向该节点发送查询请求以获取数据。

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

**（面试题：Redis支持哪些数据类型，底层数据结构是如何实现的？）**

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

Redis支持两种持久化方式：RDB和AOF。（详细请参详:[Redis使用篇——Redis 持久化方式](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Distributed%20System/Redis/Redis%E4%BD%BF%E7%94%A8%E7%AF%87%E2%80%94%E2%80%94Redis%20%E6%8C%81%E4%B9%85%E5%8C%96(Persistence).md)）

### 四、Redis 的回收策略 ###

#### I、过期策略 ####


Redis 提供了 3 种数据过期策略：
	
- 惰性删除（被动删除）：所谓惰性策略就是在客户端访问这个 key 的时候，redis 对 key 的过期时间进行检查，如果过期了就立即删除。
- 主动删除：由于惰性删除策略无法保证冷数据被及时删掉，所以 Redis 会定期主动淘汰一批已过期的 key 。（redis 会将每个设置了过期时间的 key 放入到一个独立的字典中，以后会定时遍历这个字典来删除到期的 key。）
- 主动删除：当前已用内存超过 maxmemory 限定时，触发主动清理策略，即 「数据“淘汰”策略」 。


1.定时扫描策略

Redis 默认会每秒进行十次过期扫描，过期扫描不会遍历过期字典中所有的 key，而是采用了一种简单的贪心策略。

- 1.从过期字典中随机 20 个 key；
- 2.删除这 20 个 key 中已经过期的 key；
- 3.如果过期的 key 比率超过 1/4，那就重复步骤 1；

同时，为了保证过期扫描不会出现循环过度，导致线程卡死现象，算法还增加了扫描时间的上限，默认不会超过 25ms。


2.从库的过期策略

从库不会进行过期扫描，从库对过期的处理是被动的。主库在 key 到期时，会在 AOF 文件里增加一条 del 指令，同步到所有的从库，从库通过执行这条 del 指令来删除过期的 key。

因为指令同步是异步进行的，所以主库过期的 key 的 del 指令没有及时同步到从库的话，会出现主从数据的不一致，主库没有的数据在从库里还存在，之前提到的集群环境分布式锁的算法漏洞就是因为这个同步延迟产生的。

#### II、数据淘汰策略 ####

当 Redis 内存超出物理内存限制时，内存的数据会开始和磁盘产生频繁的交换 (swap)。交换会让 Redis 的性能急剧下降，对于访问量比较频繁的 Redis 来说，这样龟速的存取效率基本上等于不可用。在生产环境中我们是不允许 Redis 出现交换行为的，为了限制最大使用内存，Redis 提供了配置参数 maxmemory 来限制内存超出期望大小。

当实际内存超出 maxmemory 时，Redis 提供了几种可选策略 (maxmemory-policy) 来让用户自己决定该如何腾出新的空间以继续提供读写服务。

- volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰。没有设置过期时间的 key 不会被淘汰，这样可以保证需要持久化的数据不会突然丢失。
- volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰。（这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合，下同）
- allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
- no-enviction（驱逐）：不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行。这样可以保证不会丢失数据，但是会让线上的业务不能持续进行。这是默认的淘汰策略。



**1.LRU算法**

实现 LRU 算法除了需要 key/value 字典外，还需要附加一个链表，链表中的元素按照一定的顺序进行排列。当空间满的时候，会踢掉链表尾部的元素。当字典的某个元素被访问时，它在链表中的位置会被移动到表头。所以链表的元素排列顺序就是元素最近被访问的时间顺序。（位于链表尾部的元素就是不被重用的元素，所以会被踢掉。位于表头的元素就是最近刚刚被人用过的元素，所以暂时不会被踢。）

Redis 使用的是一种近似 LRU 算法，它跟 LRU 算法还不太一样。之所以不使用 LRU 算法，是因为需要消耗大量的额外的内存，需要对现有的数据结构进行较大的改造。近似 LRU 算法则很简单，在现有数据结构的基础上使用随机采样法来淘汰元素，能达到和 LRU 算法非常近似的效果。Redis 为实现近似 LRU 算法，它给每个 key 增加了一个额外的小字段，这个字段的长度是 24 个 bit，也就是最后一次被访问的时间戳。

前面提到处理 key 过期方式分为集中处理和懒惰处理，LRU 淘汰不一样，它的处理方式只有懒惰处理。当 Redis 执行写操作时，发现内存超出 maxmemory，就会执行一次 LRU 淘汰算法。这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。

如何采样就是看 maxmemory-policy 的配置，如果是 allkeys 就是从所有的 key 字典中随机，如果是 volatile 就从带过期时间的 key 字典中随机。每次采样多少个 key 看的是 maxmemory_samples 的配置，默认为 5。

同时 Redis3.0 在算法中增加了淘汰池，进一步提升了近似 LRU 算法的效果。淘汰池是一个数组，它的大小是 maxmemory_samples，在每一次淘汰循环中，新随机出来的 key 列表会和淘汰池中的 key 列表进行融合，淘汰掉最旧的一个 key 之后，保留剩余较旧的 key 列表放入淘汰池中留待下一个循环。


**2.LFU算法**

在 Redis 4.0 里引入了一个新的淘汰策略 —— LFU 模式，作者认为它比 LRU 更加优秀。


LFU 的全称是Least Frequently Used，表示按最近的访问频率进行淘汰，它比 LRU 更加精准地表示了一个 key 被访问的热度。

如果一个 key 长时间不被访问，只是刚刚偶然被用户访问了一下，那么在使用 LRU 算法下它是不容易被淘汰的，因为 LRU 算法认为当前这个 key 是很热的。而 LFU 是需要追踪最近一段时间的访问频率，如果某个 key 只是偶然被访问一次是不足以变得很热的，它需要在近期一段时间内被访问很多次才有机会被认为很热。







### 五、Redis Java Client ———— Jedis ###

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






































