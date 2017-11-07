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






















