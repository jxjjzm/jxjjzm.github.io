### RocketMQ概述篇 —— Rocket 概述 ###
***

### 一、概述 ###

#### 1. RocketMQ  是什么？ ####

![](http://img3.tbcdn.cn/5476e8b07b923/TB1rdyvPXXXXXcBapXXXXXXXXXX)

上图是一个典型的消息中间件收发消息的模型，RocketMQ也是这样的设计，简单说来，RocketMQ具有以下特点：

- 是一个队列模型的消息中间件，具有高性能、高可靠、高实时、分布式特点。
-  Producer、 Consumer、队列都可以分布式。
-  Producer 向一些队列轮流发送消息，队列集合称为 Topic，Consumer 如果做广播消费，则一个 consumer实例消费这个 Topic 对应的所有队列；如果做集群消费，则多个 Consumer 实例平均消费这个 topic 对应的队列集合。
-  能够保证严格的消息顺序
-  提供丰富的消息拉取模式（Push or Pull）
-  高效的订阅者水平扩展能力
-  实时的消息订阅机制
-  亿级消息堆积能力
-  较少的依赖

**选用理由：**

- 强调集群无单点，可扩展，任意一点高可用，水平可扩展。
- 海量消息堆积能力，消息堆积后，写入低延迟。
- 支持上万个队列。
- 消息失败重试机制。
- 消息可查询。
- 开源社区活跃。
- 成熟度（经过双十一考验）。


#### 2.RocketMQ发展历程 ####

三个主要版本迭代：


- 1）.Metaq(Metamorphosis) 1.x
    
 由开源社区killme2008维护，开源社区非常活跃。[（METAQ）](https://github.com/killme2008/Metamorphosis)

- 2）.Metaq 2.x
   
 于2012年10月份上线，在淘宝内部被广泛使用。



- 3）.RocketMQ 3.x
    
基于公司内部开源共建原则，RocketMQ项目只维护核心功能，且去除了所有其他运行时的依赖，核心功能最简化。每个BU的个性化需求都在RocketMQ项目之上进行深度定制。RocketMQ向其他BU提供的仅仅是jar包，例如要定制一个Broker，那么只需要依赖rocketmq-broker这个jar包即可，可通过API进行交互，如果定制client,则依赖rocketmq-client这个jar包，对其提供的api进行再封装。

开源社区地址：[https://github.com/alibaba/RocketMQ](https://github.com/alibaba/RocketMQ)
    
在RocketMQ项目基础上衍生的项目如下：    

- com.taobao.metaq v3.0 = RocketMQ + 淘宝个性化需求 为淘宝应用提供消息服务
- com.alipay.zpullmsg v1.0 = RocketMQ + 支付宝个性化需求 为支付宝应用提供消息服务
- com.alibaba.commonmq v1.0 = Notify + RocketMQ + B2B个性化需求 为B2B应用提供消息服务  



#### 3.RocketMQ物理部署结构 ####

![](http://img3.tbcdn.cn/5476e8b07b923/TB18GKUPXXXXXXRXFXXXXXXXXXX)

如上图所示， RocketMQ的部署结构有以下特点：


- Name Server是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。


- Broker部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master与Slave的对应关系通过指定相同的BrokerName，不同的BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master也可以部署多个。每个Broker与Name Server集群中的所有节点建立长连接，定时注册Topic信息到所有Name Server。


- Producer与Name Server集群中的其中一个节点（随机选择）建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。（Producer只和Mbroker建立连接，只向Mbroker发消息。）


- Consumer与Name Server集群中的其中一个节点（随机选择）建立长连接，定期从Name Server取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定。（Consumer和Mbroker、Sbroker都建立连接。）


#### 4.RocketMQ逻辑部署结构 ####

![](http://img3.tbcdn.cn/5476e8b07b923/TB1lEPePXXXXXX8XXXXXXXXXXXX)

如上图所示，RocketMQ的逻辑部署结构有Producer和Consumer两个特点。

**Producer Group**

用来表示一个发送消息应用，一个Producer Group下包含多个Producer实例，可以是多台机器，也可以是一台机器的多个进程，或者一个进程的多个Producer对象。一个Producer Group可以发送多个Topic消息，Producer Group作用如下：



- 1）. 标识一类 Producer
- 2）. 可以通过运维工具查询这个发送消息应用下有多个 Producer 实例
- 3）. 发送分布式事务消息时，如果 Producer 中途意外宕机，Broker 会主动回调 Producer Group 内的任意一台机器来确认事务状态。


**Consumer Group**

用来表示一个消费消息应用，一个Consumer Group下包含多个Consumer实例，可以是多台机器，也可以是多个进程，或者是一个进程的多个Consumer对象。一个Consumer Group下的多个Consumer以均摊方式消费消息，如果设置为广播方式，那么这个Consumer Group下的每个实例都消费全量数据。



#### 5.RocketMQ 数据存储结构 ####

![](http://img3.tbcdn.cn/5476e8b07b923/TB1Ali2PXXXXXXuXFXXXXXXXXXX)

如上图所示，RocketMQ采取了一种数据与索引分离的存储方法。有效降低文件资源、IO资源，内存资源的损耗。即便是阿里这种海量数据，高并发场景也能够有效降低端到端延迟，并具备较强的横向扩展能力。



### 二、Rocket专业术语 ###



-  **Producer** ： 消息生产者，负责产生消息，一般由业务系统负责产生消息。


-  **Consumer** ： 消息消费者，负责消费消息，一般是后台系统负责异步消费。


-  **Push Consumer** ： Consumer 的一种，应用通常向Consumer 对象注册一个 Listener 接口，一旦收到消息，Consumer 对象立
刻回调 Listener 接口方法。


-  **Pull Consumer** ： Consumer 的一种，应用通常主动调用 Consumer 的拉消息方法从 Broker 拉消息，主动权由应用控制。


-  **Producer Group** ： 一类 Producer 的集合名称，这类 Producer 通常发送一类消息，且发送逻辑一致。


-  **Consumer Group** ： 一类 Consumer 的集合名称，这类 Consumer 通常消费一类消息，且消费逻辑一致。


-  **Broker** ： 消息中转角色，负责存储消息，转发消息，一般也称为 Server。在 JMS 规范中称为 Provider。


-  **广播消费** ： 一条消息被多个 Consumer 消费，即使这些 Consumer 属亍同一个 Consumer Group，消息也会被 Consumer
Group 中的每个 Consumer 都消费一次，广播消费中的 Consumer Group 概念可以认为在消息划分方面无意义。（在 CORBA Notification 规范中，消费方式都属亍广播消费；在 JMS 规范中，相当亍 JMS publish/subscribe model。）


-  **集群消费** ： 一个 Consumer Group 中的 Consumer 实例平均分摊消费消息。例如某个 Topic 有 9 条消息，其中一个
Consumer Group 有 3 个实例（可能是 3 个进程，或者 3 台机器），那么每个实例只消费其中的 3 条消息。（在 CORBA Notification 规范中，无此消费方式；在 JMS 规范中，JMS point-to-point model 与之类似，但是 RocketMQ 的集群消费功能不等亍 point-to-point model 模型，因为 RocketMQ 单个 Consumer Group 内的消费者类似亍 point-to-point model ，但是一个 Topic/Queue 可以被多个 ConsumerGroup 消费。）


-  **顺序消息** ： 消费消息的顺序要同发送消息的顺序一致，在 RocketMQ 中，主要指的是局部顺序，即一类消息为满足顺序性，必须 Producer 单线程顺序发送，且发送到同一个队列，这样 Consumer 就可以按照 Producer 发送的顺序去消费消息。


-  **普通顺序消息** ： 顺序消息的一种，正常情冴下可以保证完全的顺序消息，但是一旦发生通信异常，Broker 重启，由亍队列
总数发生变化，哈希取模后定位的队列会变化，产生短暂的消息顺序不一致。如果业务能容忍在集群异常情况（如某个 Broker 宕机或者重启）下，消息短暂的乱序，使用普通顺序方式比较合适。


-  **严格顺序消息** ： 顺序消息的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式 Failover 特性，即 Broker 集群中只
要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟的服务不可用。目前已知的应用只有数据库 binlog 同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序，推荐使用普通的顺序消息。（依赖同步双写，主备自动切换，自动切换功能目前还未实现）


-  **Topic** ： topic表示消息的第一级类型，比如一个电商系统的消息可以分为：交易消息、物流消息...... 一条消息必须有一个Topic。


-  **Tag** ： Tag表示消息的第二级类型，比如交易消息又可以分为：交易创建消息，交易完成消息..... 一条消息可以没有Tag。RocketMQ提供2级消息分类，方便大家灵活控制。


-  **Message Queue** ： 在 RocketMQ 中，所有消息队列都是持久化，长度无限的数据结构，所谓长度无限是指队列中的每个存储单元都是定长，访问其中的存储单元使用 Offset 来访问，offset 为 java long 类型，64 位，理论上在 100年内不会溢出，所以认为是长度无限，另外队列中只保存最近几天的数据，之前的数据会按照过期时间来删除。也可以认为 Message Queue 是一个长度无限的数组，offset 就是下标。一个topic下，我们可以设置多个Message Queue(消息队列)。当我们发送消息时，需要要指定该消息的topic。RocketMQ会轮询该topic下的所有队列，将消息发送出去。


-  **Name Server** ： NameServer即名称服务，NameServer没有状态，可以横向扩展。每个broker在启动的时候会到NameServer注册；Producer在发送消息前会根据topic到NameServer获取路由(到broker)信息；Consumer也会定时获取topic路由信息。NameServer主要两个功能：（1）、接收broker的请求，注册broker的路由信息；（2）、 接收client的请求，根据某个topic获取其到broker的路由信息。




### 三、RocketMQ关键特性 ###

#### 1.单机支持1 万以上持久化队列 ####

- 1）所有数据单独存储到一个Commit Log，完全顺序写，随机读。
- 2）对最终用户展现的队列实际只存储消息在Commit Log 的位置信息，幵丏串行方式刷盘。

![](https://i.imgur.com/5UW7qoj.png)

优点：

- 1）队列轻量化，单个数据非常少。 
- 2）对磁盘的访问串行化，避免竟争不会因为队列增加导致 IO WAIT增高。 


缺点：

- 1）写虽然完全是顺序，但读却变成了的随机读。
- 2）读一条消息，会先读 Consume Queue，再读 Commit Log，增加了开销。
- 3）要保证Commit Log与Consume Queue完全的一致，增加了编程的复杂度。


以上缺点如何克服：

（1）随机读，尽可能让读命中PAGECACHE，减少IO读操作，所以内存越大越好。如果系统中堆积的消息过多，读数据要访问磁盘会不会由于随机读导致系统性能急剧下降，答案是否定的。

- 访问PAGECACHE时，即使只访问1k的消息，系统也会提前预读出更多数据，在下次读时，就可能命中内存。
- 随机访问Commit Log磁盘数据，系统IO调度算法设置为NOOP方式，会在一定程度上将完全的随机读变成顺序跳跃方式，而顺序跳跃方式读较完全的随机读性能会高5倍以上。另外4K的消息在完全随机访问情况下，仍然可以达到8K次每秒以上的读性能。

（2）由于Consume Queue存储数据量极少，而且是顺序读，在PAGECACHE预读作用下，Consume Queue的读性能几乎与内存一致，即使堆积情况下。所以可认为Consume Queue完全不会阻碍读性能。

（3）Commit Log中存储了所有的元信息，包含消息体，类似于Mysql、Oracle的redolog,所以只要有Commit Log在，Consume Queue 即使数据丢失，仍然可以恢复出来。


#### 2.刷盘策略 ####

RocketMQ的所有消息都是持久化的，先写入系统PAGECACHE，然后刷盘，可以保证内存与磁盘都有一份数据，访问时，直接从内存读取。

**I、异步刷盘**

![](https://i.imgur.com/mqnlWQf.png)

在有RAID 卡，SAS 15000 转磁盘测试顺序写文件，速度可以达到300M 每秒左右，而线上的网卡一般都为千兆网卡，写磁盘速度明显快亍数据网络入口速度，那么是否可以做到写完内存就向用户返回，由后台线程刷盘呢？

（1）由亍磁盘速度大亍网卡速度，那举刷盘的迕度肯定可以跟上消息的写入速度。

（2）万一由于此时系统压力过大，可能堆积消息，除了写入IO，还有读取IO，万一出现磁盘读取落后情况，会不会导致内存溢出，答案是否定的，原因如下：

- 写入消息到PAGECACHE时，如果内存不足，则尝试丢弃干净的PAGE，腾出内存供新消息使用，策略是LRU方式。
- 如果干净页不足，此时写入PAGECACHE会被阻塞，系统尝试刷盘部分数据，大约每次尝试32个PAGE，来找出更多干净PAGE。



**II、同步刷盘**

![](https://i.imgur.com/MsaI8ag.png)

同步刷盘与异步刷盘的唯一区别是异步刷盘写完PAGECACHE 直接迒回，而同步刷盘需要等待刷盘完成才返回，同步刷盘流程如下：

- 1）写入PAGECACHE后，线程等待，通知刷盘线程刷盘。
- 2）刷盘线程刷盘后，唤醒前端等待线程，可能是一批线程。
- 3）前端等待线程向用户返回成功。


#### 3.消息查询 ####

**I、按照Message Id查询消息**

![](https://i.imgur.com/FREPIqw.png)

MsgId总共16字节，包含消息存储主机地址，消息Commit Log offset。从MsgId中解析出Broker 的地址和Commit Log的偏移地址，然后按照存储格式所在位置消息buffer解析成一个完整的消息。



**II、按照Message Key查询消息**

![](https://i.imgur.com/PvoXg6K.png)



- 1）根据查询的key的hashcode%slotNum得到具体的槽的位置 （slotNum是一个索引文件里面包含的最大槽的数目，例如图中所示 slotNum=5000000） 。
- 2）根据 slotValue（slot 位置对应的值）查找到索引项列表的最后一项（倒序排列，slotValue 总是指向最新的一个索引项） 。
- 3）遍历索引项列表返回查询时间范围内的结果集（默认一次最大返回的 32 条记录）
- 4）Hash 冲突；寻找 key 的 slot 位置时相当于执行了两次散列函数，一次 key 的 hash，一次 key 的 hash 值取模，因此这里存在两次冲突的情况；第一种，key 的 hash 值不同但模数相同，此时查询的时候会在比较一次 key 的hash 值（每个索引项保存了 key 的 hash 值） ，过滤掉 hash 值不相等的项。第二种，hash 值相等但 key 不等，出于性能的考虑冲突的检测放到客户端处理（key 的原始值是存储在消息文件中的，避免对数据文件的解析） ，客户端比较一次消息体的 key 是否相同。
- 5）存储；为了节省空间索引项中存储的时间是时间差值（存储时间-开始时间，开始时间存储在索引文件头中） ，整个索引文件是定长的，结构也是固定的。索引文件存储结构参见上图 。



#### 4.服务器消息过滤 ####

RocketMQ的消息过滤方式有别于其他消息中间件，是在订阅时，再做过滤，先来看下Consume Queue的存储结构。

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/3f39e422477e43da22ded6a2cb82cb54.png)



- 1）在Broker端进行Message Tag比对，先遍历Consume Queue，如果存储的Message Tag与订阅的Message Tag不符合，则跳过，继续比对下一个，符合则传输给Consumer。注意：Message Tag是字符串形式，Consume Queue中存储的是其对应的hashcode，比对时也是比对hashcode。
- 2）Consumer收到过滤后的消息后，同样也要执行在Broker端的操作，但是比对的是真实的Message Tag字符串，而不是Hashcode。


为什么过滤要这样做？



- Message Tag存储Hashcode，是为了在Consume Queue定长方式存储，节约空间。
- 过滤过程中不会访问Commit Log数据，可以保证堆积情况下也能高效过滤。
- 即使存在Hash冲突，也可以在Consumer端进行修正，保证万无一失。


#### 5.顺序消息 ####

很多业务有顺序消息的需求，RocketMQ支持全局和局部的顺序，一般推荐使用局部顺序，将具有顺序要求的一类消息hash到同一个队列中便可保持有序，如下图所示。

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/ecddb3cfe047a15cde60191021219a39.png)

详细请参考 [RocketMQ概述篇 —— Rocket 顺序消息与重复消息、事务消息](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/RocketMQ%E7%B3%BB%E5%88%97/RocketMQ%E6%A6%82%E8%BF%B0%E7%AF%87%20%E2%80%94%E2%80%94%20Rocket%20%E9%A1%BA%E5%BA%8F%E6%B6%88%E6%81%AF%E4%B8%8E%E9%87%8D%E5%A4%8D%E6%B6%88%E6%81%AF%E3%80%81%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF.md)

#### 6.事务消息 ####

详细请参考 [RocketMQ概述篇 —— Rocket 顺序消息与重复消息、事务消息](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/RocketMQ%E7%B3%BB%E5%88%97/RocketMQ%E6%A6%82%E8%BF%B0%E7%AF%87%20%E2%80%94%E2%80%94%20Rocket%20%E9%A1%BA%E5%BA%8F%E6%B6%88%E6%81%AF%E4%B8%8E%E9%87%8D%E5%A4%8D%E6%B6%88%E6%81%AF%E3%80%81%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF.md)


#### 7.定时消息 ####

**I、概念介绍**

- 定时消息：Producer 将消息发送到 MQ 服务端，但并不期望这条消息立马投递，而是推迟到在当前时间点之后的某一个时间投递到 Consumer 进行消费，该消息即定时消息。
- 延时消息：Producer 将消息发送到 MQ 服务端，但并不期望这条消息立马投递，而是延迟一定时间后才投递到 Consumer 进行消费，该消息即延时消息。

RocketMQ为了实现定时消息，引入延时级别，牺牲部分灵活性，事实上很少有业务需要随意指定定时时间的灵活性。定时消息内容被存储在数据文件中，索引按延时级别堆积在定时消息队列中，具有跟普通消息一致的堆积能力，如下图所示。

![](http://ata2-img.cn-hangzhou.img-pub.aliyun-inc.com/f32846631e1fe40b69c1c7057965e948.png)


**II、使用场景**


- 消息生产和消费有时间窗口要求：比如在电商交易中超时未支付关闭订单的场景，在订单创建时会发送一条 MQ 延时消息，这条消息将会在30分钟以后投递给消费者，消费者收到此消息后需要判断对应的订单是否已完成支付。如支付未完成，则关闭订单，如已完成支付则忽略。
- 通过消息触发一些定时任务，比如在某一固定时间点向用户发送提醒消息。

**III、使用方式**

定时消息、延时消息的使用在代码编写上存在略微的区别：

- 发送定时消息需要明确指定消息发送时间点之后的某一时间点作为消息投递的时间点。
- 发送延时消息时需要设定一个延时时间长度，消息将从当前发送时间点开始延迟固定时间之后才开始投递。

**IV.注意事项：**


- 定时/延时消息 msg.setStartDeliverTime 的参数需要设置成当前时间戳之后的某个时刻（单位毫秒），如果被设置成当前时间戳之前的某个时刻，消息将立刻投递给消费者。
- 定时/延时消息 msg.setStartDeliverTime 的参数可设置40天内的任何时刻（单位毫秒），超过40天消息发送将失败。
- StartDeliverTime 是服务端开始向消费端投递的时间。如果消费者当前有消息堆积，那么定时、延时消息会排在堆积消息后面，将不能严格按照配置的时间进行投递。
- 由于客户端和服务端可能存在时间差，定时消息/延时消息的投递也可能与客户端设置的时间存在偏差。
- 设置定时、延时消息的投递时间后，依然受 3 天的消息保存时长限制。例如，设置定时消息 5 天后才能被消费，如果第 5 天后一直没被消费，那么这条消息将在第8天被删除。
- 除 TCP 协议接入的 Java 语言支持延时消息，其他方式都不支持延时消息。


**V.示例代码**



- [发送定时消息](http://help.aliyun.com/document_detail/29550.html?spm=5176.doc43349.2.4.IBG7gE)



- [发送延时消息](https://help.aliyun.com/document_detail/29549.html?spm=5176.doc43349.2.5.IBG7gE)

#### 8.单个JVM进程也能利用机器超大内存 ####

![](http://www.uml.org.cn/zjjs/images/2015040118.png)



- 1）.Producer发送消息，消息从socket进入java 堆
- 2）.Producer发送消息，消息从java堆进入pagecache，物理内存
- 3）.Producer发送消息，由异步线程刷盘，消息从pagecache刷入磁盘
- 4）.Consumer拉消息（正常消费），消息直接从pagecache（数据在物理内存）转入socket，到达Consumer，不经过java堆。这种消费场景最多，线上96G物理内存，按照1K消息算，可以物理缓存1亿条消息
- 5）.Consumer拉消息（异常消费），消息直接从pagecache转入socket
- 6）.Consumer拉消息（异常消费），由于socket访问了虚拟内存，产生缺页中断，此时会产生磁盘IO，从磁盘Load消息到pagecache，然后直接从socket发出去
- 7）.同5
- 8）.同6


#### 9.消息堆积问题解决办法 ####

![](https://i.imgur.com/e2Yrrbr.png)

在有Slave情况下，Master一旦发现Consumer访问堆积在磁盘的数据时，会向Consumer下达一个重定向指令，令Consumer从Slave拉取数据，这样正常的发消息与正常消费的Consumer都不会因为消息堆积受影响，因为系统将堆积场景与非堆积场景分割在了两个不同的节点处理。这里会产生另一个问题，Slave会不会写性能下降，答案是否定的。因为Slave的消息写入只追求吞吐量，不追求实时性，只要整体的吞吐量高就行了，而Slave每次都是从Master拉取一批数据，如1M，这种批量顺序写入方式使堆积情况，整体吞吐量影响相对较小，只是写入RT会变长.

















