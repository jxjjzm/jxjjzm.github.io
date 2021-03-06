### RocketMQ面试篇 ###
***


### 1.请描述下 RocketMQ 的整体流程？ ###

![](http://www.iocoder.cn/images/RocketMQ/2019_11_12/02.png)

**RocketMQ的整体流程：**



- 1、启动 Namesrv，Namesrv起来后监听端口，等待 Broker、Producer、Consumer 连上来，相当于一个路由控制中心。
- 2、Broker 启动，跟所有的 Namesrv 保持长连接，定时发送心跳包。（心跳包中，包含当前 Broker 信息(IP+端口等)以及存储所有 Topic 信息。注册成功后，Namesrv 集群中就有 Topic 跟 Broker 的映射关系。）
- 3、收发消息前，先创建 Topic 。创建 Topic 时，需要指定该 Topic 要存储在 哪些 Broker上。也可以在发送消息时自动创建Topic。
- 4、Producer 发送消息。（启动时，先跟 Namesrv 集群中的其中一台建立长连接，并从Namesrv 中获取当前发送的 Topic 存在哪些 Broker 上，然后跟对应的 Broker 建立长连接，直接向 Broker 发消息。）
- 5、Consumer 消费消息。（Consumer 跟 Producer 类似。跟其中一台 Namesrv 建立长连接，获取当前订阅 Topic 存在哪些 Broker 上，然后直接跟 Broker 建立连接通道，开始消费消息。）


### 2.请说说你对 Namesrv 的了解？ ###

1、 Namesrv 用于存储 Topic、Broker 关系信息，功能简单，稳定性高。



- 多个 Namesrv 之间相互没有通信，单台 Namesrv 宕机不影响其它 Namesrv 与集群。（多个 Namesrv 之间的信息共享，通过 Broker 主动向多个 Namesrv 都发起心跳。正如前文所说，Broker 需要跟所有 Namesrv 连接。）


- 即使整个 Namesrv 集群宕机，已经正常工作的 Producer、Consumer、Broker 仍然能正常工作，但新起的 Producer、Consumer、Broker 就无法工作。（这点和 Dubbo 有些不同，不会缓存 Topic 等元信息到本地文件。）


2、 Namesrv 压力不会太大，平时主要开销是在维持心跳和提供 Topic-Broker 的关系数据。但有一点需要注意，Broker 向 Namesr 发心跳时，会带上当前自己所负责的所有 Topic 信息，如果 Topic 个数太多（万级别），会导致一次心跳中，就 Topic 的数据就几十 M，网络情况差的话，网络传输失败，心跳失败，导致 Namesrv 误认为 Broker 心跳失败。（当然，一般公司，很难达到过万级的 Topic ，因为一方面体量达不到，另一方面 RocketMQ 提供了 Tag 属性。另外，内网环境网络相对是比较稳定的，传输几十 M 问题不大。同时，如果真的要优化，Broker 可以把心跳包做压缩，再发送给 Namesrv 。不过，这样也会带来 CPU 的占用率的提升。）


### 3.请说说你对 Broker 的了解？ ###

1、 高并发读写服务。Broker的高并发读写主要是依靠以下两点：

- 消息顺序写，所有 Topic 数据同时只会写一个文件，一个文件满1G ，再写新文件，真正的顺序写盘，使得发消息 TPS 大幅提高。
- 消息随机读，RocketMQ 尽可能让读命中系统 Pagecache ，因为操作系统访问 Pagecache 时，即使只访问 1K 的消息，系统也会提前预读出更多的数据，在下次读时就可能命中 Pagecache ，减少 IO 操作。

2、 负载均衡与动态伸缩。



- 负载均衡：Broker 上存 Topic 信息，Topic 由多个队列组成，队列会平均分散在多个 Broker 上，而 Producer 的发送机制保证消息尽量平均分布到所有队列中，最终效果就是所有消息都平均落在每个 Broker 上。
- 动态伸缩能力（非顺序消息）：Broker 的伸缩性体现在两个维度：Topic、Broker。
	- Topic 维度：假如一个 Topic 的消息量特别大，但集群水位压力还是很低，就可以扩大该 Topic 的队列数， Topic 的队列数跟发送、消费速度成正比。（Topic 的队列数一旦扩大，就无法很方便的缩小。因为，生产者和消费者都是基于相同的队列数来处理。如果真的想要缩小，只能新建一个 Topic ，然后使用它。不过，Topic 的队列数，也不存在什么影响的，淡定。）


- Broker 维度：如果集群水位很高了，需要扩容，直接加机器部署 Broker 就可以。Broker 启动后向 Namesrv 注册，Producer、Consumer 通过 Namesrv 发现新Broker，立即跟该 Broker 直连，收发消息。（新增的 Broker 想要下线，想要下线也比较麻烦，暂时没特别好的方案。大体的前提是，消费者消费完该 Broker 的消息，生产者不往这个 Broker 发送消息。）


3、 高可用 & 高可靠。



- 高可用：集群部署时一般都为主备，备机实时从主机同步消息，如果其中一个主机宕机，备机提供消费服务，但不提供写服务。
- 高可靠：所有发往 Broker 的消息，有同步刷盘和异步刷盘机制。
	- 同步刷盘时，消息写入物理文件才会返回成功。
	- 异步刷盘时，只有机器宕机，才会产生消息丢失，Broker 挂掉可能会发生，但是机器宕机崩溃是很少发生的，除非突然断电。（如果 Broker 挂掉，未同步到硬盘的消息，还在 Pagecache 中呆着。）


4、 Broker 与 Namesrv 的心跳机制。


- 单个 Broker 跟所有 Namesrv 保持心跳请求，心跳间隔为30秒，心跳请求中包括当前 Broker 所有的 Topic 信息。
- Namesrv 会反查 Broker 的心跳信息，如果某个 Broker 在 2 分钟之内都没有心跳，则认为该 Broker 下线，调整 Topic 跟 Broker 的对应关系。但此时 Namesrv 不会主动通知Producer、Consumer 有 Broker 宕机。也就说，只能等 Producer、Consumer 下次定时拉取 Topic 信息的时候，才会发现有 Broker 宕机。




### 4.请说说你对 Producer 的了解？ ###


1、获得 Topic-Broker 的映射关系。



- Producer 启动时，也需要指定 Namesrv 的地址，从 Namesrv 集群中选一台建立长连接。如果该 Namesrv 宕机，会自动连其他 Namesrv ，直到有可用的 Namesrv 为止。
- 生产者每 30 秒从 Namesrv 获取 Topic 跟 Broker 的映射关系，更新到本地内存中。然后再跟 Topic 涉及的所有 Broker 建立长连接，每隔 30 秒发一次心跳。
- 在 Broker 端也会每 10 秒扫描一次当前注册的 Producer ，如果发现某个 Producer 超过 2 分钟都没有发心跳，则断开连接。

2、生产者端的负载均衡。

生产者发送时，会自动轮询当前所有可发送的broker，一条消息发送成功，下次换另外一个broker发送，以达到消息平均落到所有的broker上。（这里需要注意一点：假如某个 Broker 宕机，意味生产者最长需要 30 秒才能感知到。在这期间会向宕机的 Broker 发送消息。当一条消息发送到某个 Broker 失败后，会自动再重发 2 次，假如还是发送失败，则抛出发送失败异常。（客户端里会自动轮询另外一个 Broker 重新发送，这个对于用户是透明的。））



#### 5.请说说你对 Consumer 的了解？ ####

1、获得 Topic-Broker 的映射关系。


- Consumer 启动时需要指定 Namesrv 地址，与其中一个 Namesrv 建立长连接。消费者每隔 30 秒从 Namesrv 获取所有Topic 的最新队列情况，这意味着某个 Broker 如果宕机，客户端最多要 30 秒才能感知。连接建立后，从 Namesrv 中获取当前消费 Topic 所涉及的 Broker，直连 Broker 。
- Consumer 跟 Broker 是长连接，会每隔 30 秒发心跳信息到Broker 。Broker 端每 10 秒检查一次当前存活的 Consumer ，若发现某个 Consumer 2 分钟内没有心跳，就断开与该 Consumer 的连接，并且向该消费组的其他实例发送通知，触发该消费者集群的负载均衡。

2、消费者端的负载均衡。根据消费者的消费模式不同（消费者有两种消费模式：集群消费和广播消费。），负载均衡方式也不同。


-  集群消费：一个 Topic 可以由同一个消费这分组( Consumer Group )下所有消费者分担消费。 > 具体例子：假如 TopicA 有 6 个队列，某个消费者分组起了 2 个消费者实例，那么每个消费者负责消费 3 个队列。如果再增加一个消费者分组相同消费者实例，即当前共有 3 个消费者同时消费 6 个队列，那每个消费者负责 2 个队列的消费。
广播消费：每个消费者消费 Topic 下的所有队列。



附：如何对消息进行重放？

消费位点就是一个数字，把 Consumer Offset 改一下，就可以达到重放的目的了。

#### 6.如何保证消费者的消费消息的幂等性？ ####

I、框架层统一封装

- 首先，需要有一个消息排重的唯一标识，该编号只能由 Producer 生成，例如说使用 uuid、或者其它唯一编号的算法 。
- 然后，就需要有一个排重的存储器
	- 使用关系数据库，增加一个排重表，使用消息编号作为唯一主键。
	- 使用 KV 数据库，KEY 存储消息编号，VALUE 任一。此处，暂时不考虑 KV 数据库持久化的问题

那么，我们要什么时候插入这条排重记录呢？



- 在消息消费执行业务逻辑之前，插入这条排重记录。但是，此时会有可能 JVM 异常崩溃。那么 JVM 重启后，这条消息就无法被消费了。因为，已经存在这条排重记录。
- 在消息消费执行业务逻辑之后，插入这条排重记录。
	- 如果业务逻辑执行失败，显然，我们不能插入这条排重记录，因为我们后续要消费重试。
	- 如果业务逻辑执行成功，此时，我们可以插入这条排重记录。但是，万一插入这条排重记录失败呢？那么，需要让插入记录和业务逻辑在同一个事务当中，此时，我们只能使用数据库。

II、业务层自己实现


- 先查询数据库，判断数据是否已经被更新过。如果是，则直接返回消费完成，否则执行消费。
- 更新数据库时，带上数据的状态。如果更新失败，则直接返回消费完成，否则执行消费。
- …


正常情况下，出现重复消息的概率其实很小，如果由框架层统一封装来实现的话，肯定会对消息系统的吞吐量和高可用有影响，所以最好还是由业务层自己实现处理消息重复的问题。


#### 7.RocketMQ 是否会弄丢数据？ ####

I、消费端弄丢了数据？

对于消费端，如果我们在使用 Push 模式的情况下，只有我们消费返回成功，才会异步定期更新消费进度到 Broker 上。如果消费端异常崩溃，可能导致消费进度未更新到 Broker 上，那么无非是 Consumer 可能重复拉取到已经消费过的消息。关于这个，就需要消费端做好消费的幂等性。


II、Broker 弄丢了数据？

刷盘方式：同步刷盘、异步刷盘。
复制方式：同步复制、异步复制。

如果要保证 Broker 数据最大化的不丢，需要在搭建 Broker 集群时，设置为同步刷盘、同步复制。当然，带来了可靠性，也会一定程度降低性能。如果想要在可靠性和性能之间做一个平衡，可以选择同步复制，加主从 Broker 都适合异步刷盘。因为，刷盘比较消耗性能。


III、生产者会不会弄丢数据？

Producer 可以设置三次发送消息重试。



#### 7.RocketMQ 如何保证消息的顺序性？ ####


消费消息的顺序要同发送消息的顺序一致。由于 Consumer 消费消息的时候是针对 Message Queue 顺序拉取并开始消费，且一条 Message Queue 只会给一个消费者（集群模式下），所以能够保证同一个消费者实例对于 Queue 上消息的消费是顺序地开始消费（不一定顺序消费完成，因为消费可能并行）。



- Consumer ：在 RocketMQ 中，顺序消费主要指的是都是 Queue 级别的局部顺序。这一类消息为满足顺序性，必须 Producer 单线程顺序发送，且发送到同一个队列，这样 Consumer 就可以按照 Producer 发送的顺序去消费消息。
- Producer ：生产者发送的时候可以用 MessageQueueSelector 为某一批消息（通常是有相同的唯一标示id）选择同一个 Queue ，则这一批消息的消费将是顺序消息（并由同一个consumer完成消息）。或者 Message Queue 的数量只有 1 ，但这样消费的实例只能有一个，多出来的实例都会空跑。

总的来说，RocketMQ 提供了两种顺序级别：



- 普通顺序消息 ：Producer 将相关联的消息发送到相同的消息队列。———— 顺序消息的一种，正常情况下可以保证完全的顺序消息，但是一旦发生异常，Broker 宕机或重启，由于队列总数发生发化，消费者会触发负载均衡，而默认地负载均衡算法采取哈希取模平均，这样负载均衡分配到定位的队列会发化，使得队列可能分配到别的实例上，则会短暂地出现消息顺序不一致。（如果业务能容忍在集群异常情况（如某个 Broker 宕机或者重启）下，消息短暂的乱序，使用普通顺序方式比较合适。）

- 严格顺序消息 ：在【普通顺序消息】的基础上，Consumer 严格顺序消费。———— 顺序消息的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式 Failover 特性，即 Broker 集群中只要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。（如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟的服务不可用。（依赖同步双写，主备自动切换，自动切换功能目前并未实现））


目前已知的应用只有数据库 binlog 同步强依赖严格顺序消息，其他应用绝大部分都可以容忍短暂乱序，推荐使用普通的顺序消息。


#### 8.RocketMQ 如何实现定时消息？ ####

定时消息，是指消息发到 Broker 后，不能立刻被 Consumer 消费，要到特定的时间点或者等待特定的时间后才能被消费。目前，开源版本的 RocketMQ 只支持固定延迟级别的延迟消息，不支持任一时刻的延迟消息。


实现原理：

- 定时消息发送到 Broker 后，会被存储 Topic 为 SCHEDULE_TOPIC_XXXX 中，并且所在 Queue 编号为延迟级别 - 1 。（需要 -1 的原因是，延迟级别是从 1 开始的。如果延迟级别为 0 ，意味着无需延迟。）

- Broker 针对每个 SCHEDULE_TOPIC_XXXX 的队列，都创建一个定时任务，顺序扫描到达时间的延迟消息，重新存储到延迟消息原始的 Topic 的原始 Queue 中，这样它就可以被 Consumer 消费到。此处会有两个问题：
	- 为什么是“顺序扫描到达时间的延迟消息”？因为先进 SCHEDULE_TOPIC_XXXX 的延迟消息，在其所在的队列，意味着先到达延迟时间。
	- 会不会存在重复扫描的情况？每个 SCHEDULE_TOPIC_XXXX 的扫描进度，会每 10s 存储到 config/delayOffset.json 文件中，所以正常情况下，不会存在重复扫描。如果异常关闭，则可能导致重复扫描。


#### 8.RocketMQ 如何实现消息重试？ ####


消息重试，Consumer 消费消息失败后，要提供一种重试机制，令消息再消费一次。

- Consumer 会将消费失败的消息发回 Broker，进入延迟消息队列。即，消费失败的消息，不会立即消费。
- 也就是说，消息重试是构建在定时消息之上的功能。

消息重试的主要流程：


- Consumer 消息失败后，会将消息的 Topic 修改为 %RETRY% + Topic 进行，添加 "RETRY_TOPIC" 属性为原始 Topic ，然后再返回给 Broker 中。
- Broker 收到重试消息之后，会有两次修改消息的 Topic 。
	- 首先，会将消息的 Topic 修改为 %RETRY% + ConsumerGroup ，因为这个消息是当前消费这分组消费失败，只能被这个消费组所重新消费。😈 注意噢，消费者会默认订阅 Topic 为 %RETRY% + ConsumerGroup 的消息。
	- 然后，会将消息的 Topic 修改为 SCHEDULE_TOPIC_XXXX ，添加 "REAL_TOPIC" 属性为 %RETRY% + ConsumerGroup ，因为重试消息需要延迟消费。

- Consumer 会拉取该 Topic 对应的 retryTopic 的消息，此处的 retryTopic 为 %RETRY% + ConsumerGroup 。
- Consumer 拉取到 retryTopic 消息之后，置换到原始的 Topic ，因为有消息的 "RETRY_TOPIC" 属性是原始 Topic ，然后把消息交给 Listener 消费。


附：多次消费失败后，怎么办？

默认情况下，当一条消息被消费失败 16 次后，会被存储到 Topic 为 "%DLQ%" + ConsumerGroup 到死信队列。（为什么 Topic 是 "%DLQ%" + ConsumerGroup 呢？因为，是这个 ConsumerGroup 对消息的消费失败，所以 Topic 里要以 ConsumerGroup 为维度。后续，我们可以通过订阅 "%DLQ%" + ConsumerGroup ，做相应的告警。）


#### 9.RocketMQ 如何实现事务消息？ ####

参考：https://www.infoq.cn/article/2018%2F08%2Frocketmq-4.3-release

























































