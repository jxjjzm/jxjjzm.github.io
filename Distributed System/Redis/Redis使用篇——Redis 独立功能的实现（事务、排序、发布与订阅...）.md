### Redis使用篇——Redis 独立功能的实现（事务、排序、发布与订阅...） ###
***
### 一、过期时间 ###




### 二、发布与订阅 ###

Redis的发布与订阅功能由PUBLISH、SUBSCRIBE、PSUBSCRIBE等命令组成。



#### 1.频道的订阅与信息发送 ####


**I、PubSub使用**

- 通过执行SUBSCRIBE命令，客户端可以订阅一个或多个频道，从而成为这些频道的订阅者（subscriber）:每当有其他客户端向被订阅的频道发送消息（message）时，频道的所有订阅者都会收到这条消息。

![](https://i.imgur.com/HHBWgFQ.png)

- 当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端：

![](https://i.imgur.com/o7PkLSo.png)


**II、实现原理**

每个 Redis 服务器进程都维持着一个表示服务器状态的 redis.h/redisServer 结构， 结构的 pubsub_channels 属性是一个字典， 这个字典就用于保存订阅频道的信息：

	struct redisServer {
	    // ...
	    dict *pubsub_channels;
	    // ...
	};

其中，字典的键为正在被订阅的频道， 而字典的值则是一个链表， 链表中保存了所有订阅这个频道的客户端。

![](https://i.imgur.com/Ell8YjZ.png)

了解了 pubsub_channels 字典的结构之后， 解释 PUBLISH 命令的实现就非常简单了： 当调用 PUBLISH channel message 命令， 程序首先根据 channel 定位到字典的键， 然后将信息发送给字典值链表中的所有客户端。

使用 UNSUBSCRIBE 命令可以退订指定的频道， 这个命令执行的是订阅的反操作： 它从 pubsub_channels 字典的给定频道（键）中， 删除关于当前客户端的信息， 这样被退订频道的信息就不会再发送给这个客户端。

#### 2.模式的订阅与信息发送 ####

**I、PSubPub使用**

除了订阅频道之外，客户端还可以通过执行PSUBSCRIBE命令订阅一个或多个模式，从而成为这些模式的订阅者：每当有其他客户端向某个频道发送消息时，消息不仅会被发送给这个频道的所有订阅者，它还会被发送给所有与这个频道相匹配的模式的订阅者。

![](https://i.imgur.com/347UOZl.png)

**II、实现原理**

redisServer.pubsub_patterns 属性是一个链表，链表中保存着所有和模式相关的信息：

	struct redisServer {
	    // ...
	    list *pubsub_patterns;
	    // ...
	};


链表中的每个节点都包含一个 redis.h/pubsubPattern 结构：

	typedef struct pubsubPattern {
	    redisClient *client;
	    robj *pattern;
	} pubsubPattern;

client 属性保存着订阅模式的客户端，而 pattern 属性则保存着被订阅的模式。每当调用 PSUBSCRIBE 命令订阅一个模式时， 程序就创建一个包含客户端信息和被订阅模式的 pubsubPattern 结构， 并将该结构添加到 redisServer.pubsub_patterns 链表中。通过遍历整个 pubsub_patterns 链表，程序可以检查所有正在被订阅的模式，以及订阅这些模式的客户端。

![](https://i.imgur.com/yu3c2ds.png)

 PUBLISH 除了将 message 发送到所有订阅 channel 的客户端之外， 它还会将 channel 和 pubsub_patterns 中的模式进行对比， 如果 channel 和某个模式匹配的话， 那么也将 message 发送到订阅那个模式的客户端。

使用 PUNSUBSCRIBE 命令可以退订指定的模式， 这个命令执行的是订阅模式的反操作： 程序会删除 redisServer.pubsub_patterns 链表中， 所有和被退订模式相关联的 pubsubPattern 结构， 这样客户端就不会再收到和模式相匹配的频道发来的信息。


**小结：**




- 订阅信息由服务器进程维持的 redisServer.pubsub_channels 字典保存，字典的键为被订阅的频道，字典的值为订阅频道的所有客户端。
- 当有新消息发送到频道时，程序遍历频道（键）所对应的（值）所有客户端，然后将消息发送到所有订阅频道的客户端上。
- 订阅模式的信息由服务器进程维持的 redisServer.pubsub_patterns 链表保存，链表的每个节点都保存着一个 pubsubPattern 结构，结构中保存着被订阅的模式，以及订阅该模式的客户端。程序通过遍历链表来查找某个频道是否和某个模式匹配。
- 当有新消息发送到频道时，除了订阅频道的客户端会收到消息之外，所有订阅了匹配频道的模式的客户端，也同样会收到消息。
- 退订频道和退订模式分别是订阅频道和订阅模式的反操作。

### 三、事务 ###


每个事务的操作都有 begin、commit 和 rollback，begin 指示事务的开始，commit 指示事务的提交，rollback 指示事务的回滚。它大致的形式如下：

	begin();
	try {
	    command1();
	    command2();
	    ....
	    commit();
	} catch(Exception e) {
	    rollback();
	}

Redis 在形式上看起来也差不多，Redis通过MULTI、EXEC、DISCARD、WATCH等命令来实现事务功能。

![](https://i.imgur.com/7hZzx6j.png)

![](https://i.imgur.com/dJCNsL8.png)

- multi : MULTI 命令的执行标记着事务的开始,这个命令唯一做的就是， 将客户端的 REDIS_MULTI 选项打开， 让客户端从非事务状态切换到事务状态。

- ecec : 如果客户端正处于事务状态， 那么当 EXEC 命令执行时， 服务器根据客户端所保存的事务队列， 以先进先出（FIFO）的方式执行事务队列中的命令： 最先入队的命令最先执行， 而最后入队的命令最后执行。当事务队列里的所有命令被执行完之后， EXEC 命令会将回复队列作为自己的执行结果返回给客户端， 客户端从事务状态返回到非事务状态， 至此， 事务执行完毕。

- discard : DISCARD 命令用于取消一个事务， 它清空客户端的整个事务队列， 然后将客户端从事务状态调整回非事务状态， 最后返回字符串 OK 给客户端， 说明事务已被取消。

- watch : WATCH 命令用于在事务开始之前监视任意数量的键： 当调用 EXEC 命令执行事务时， 如果任意一个被监视的键已经被其他客户端修改了， 那么整个事务不再执行， 直接返回失败。（Redis 禁止在 multi 和 exec 之间执行 watch 指令，而必须在 multi 之前做好盯住关键变量，否则会出错。）

### I、WATCH命令的实现 ###

每个Redis数据库都保存着一个watched_keys字典，这个字典的键是某个被WATCH命令监视的数据库键，而字典的值则是一个链表，链表中记录了所有监视相应数据库键的客户端。通过watched_keys字典，服务器可以清楚地知道哪些数据库键正在被监视以及哪些客户端正在监视这些数据库键。

所有对数据库进行修改的命令，在执行之后都会调用multi.c/touchWatchKey函数对watched_keys字典进行检查，查看是否有客户端正在监视刚刚被命令修改过的数据库键，如果有的话，那么toouchWatchKey函数会将监视被修改键的客户端的REDIS_DIRTY_CAS标识打开，标识该客户端的事务安全性已经被破坏。

当服务器接收到一个客户端发来的EXEC命令时，服务器会根据这个客户端是否打开了REDIS_DIRTY_CAS标识来决定是否执行事务。

![](https://i.imgur.com/4LyZwCt.png)

### II、ACID ###

- 原子性（Atomicity）：Redis的事务和传统的关系型数据库事务的最大区别在于，Redis不支持事务回滚机制，即使事务队列中的某个命令在执行期间出现了错误，整个事务也会继续执行下去，直到将事务队列中的所有命令都执行完毕为止。Redis的作者在事务功能的文档中解释说，不支持事务回滚是因为这种复杂的功能和Redis追求简单高效的设计主旨不相符，并且他认为，Redis事务的执行时错误通常都是编程错误产生的，这种错误通常只会出现在开发环境中，而很少会在实际的生产环境中出现，所以他认为没有必要为Redis开发事务回滚功能。（传统数据库先记录日志，失败再回滚也是为了保证事务执行失败时能回到原始状态，保持数据一致性；redis是先执行指令，然后记录日志）
- 一致性（Consistency）：Redis通过谨慎的错误检测和简单的设计来保证事务的一致性。
- 隔离性（Isolation）：隔离性，在多个事务并发的情况下，事务之间不会被影响。对于Redis来说，事务的执行是串行的，中途不会插入其它命令的执行，所以是满足隔离性的。
- 持久性（Durability）：Redis的事务不过是简单地用队列包裹起了一组Redis命令，Redis并没有为事务提供任何额外的持久化功能，所以Redis事务的耐久性由Redis所使用的持久化模式决定。（当服务器运行在AOF持久化模式下，并且appendfsync选项的值为always时，事务才具有持久性）


总而言之，Redis 的事务保证了 ACID 中的一致性（C）和隔离性（I），但并不保证原子性（A）和持久性（D）。

四、排序






五、慢查询日志





### 六、管道（Pipeline） ###

Redis是一种基于客户端-服务端模型以及请求/响应协议的TCP服务。这意味着通常情况下一个请求会遵循以下步骤：


- 客户端向服务端发送一个查询请求，并监听Socket返回，通常是以阻塞模式，等待服务端响应。
- 服务端处理命令，并将结果返回给客户端。

Redis 管道技术(Pipeline)可以在服务端未响应时，客户端可以继续向服务端发送请求，并最终一次性读取所有服务端的响应。(换而言之，客户端允许将多个请求依次发给服务器，过程中而不需要等待请求的回复，在最后再一并读取结果即可。)

**普通请求模型：**

![](https://images2017.cnblogs.com/blog/242916/201802/242916-20180205184814060-1691050944.png)


**Pipeline请求模型**


![](https://images2017.cnblogs.com/blog/242916/201802/242916-20180205184821607-996015857.png)


Redis Pipeline就是通过减少客户端与redis的通信次数来实现降低往返延时时间，而且Pipeline 实现的原理是队列，而队列的原理是先进先出，这样就保证数据的顺序性。