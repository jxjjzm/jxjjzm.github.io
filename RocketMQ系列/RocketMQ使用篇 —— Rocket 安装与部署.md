### RocketMQ使用篇 —— Rocket 安装与部署 ###
***


### 一、RocketMQ 集群模式 ###

#### 1.单个Master ####

这种方式风险比较大，一旦Broker重启或者宕机时，会导致整个服务不可用，不建议线上环境使用。


#### 2.多Master模式 ####

一个集群无Slave，全是Master，例如2个Master或者3个Master



- 优点：配置简单，单个Master 宕机或重启维护对应用无影响，在磁盘配置为RAID10 时，即使机器宕机不可恢复情况下，由于 RAID10
磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢），性能最高。



- 缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到受到影响。


		### 先启动 NameServer
		### 在机器 A，启动第一个 Master
		### 在机器 B，启动第二个 Master


#### 3.多Master多Slave模式，异步复制 ####

每个 Master 配置一个 Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟，毫秒级。



- 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为Master 宕机后，消费者仍然可以从 Slave消费，此过程对应用透明，不需要人工干预。性能同多 Master 模式几乎一样。




- 缺点：Master 宕机，磁盘损坏情况，会丢失少量消息。

		### 先启动 NameServer
		### 在机器 A，启动第一个 Master
		### 在机器 B，启动第二个 Master
		### 在机器 C，启动第一个 Slave
		### 在机器 D，启动第二个 Slave




#### 4.多Master多Slave模式，同步双写 ####

每个 Master 配置一个 Slave，有多对Master-Slave，HA采用同步双写方式，主备都写成功，向应用返回成功。



- 优点：数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高


- 缺点：性能比异步复制模式略低，大约低 10%左右，发送单个消息的 RT
会略高。目前主宕机后，备机不能自动切换为主机，后续会支持自动切换功能。

		### 先启动 NameServer
		### 在机器 A，启动第一个 Master
		### 在机器 B，启动第二个 Master
		### 在机器 C，启动第一个 Slave
		### 在机器 D，启动第二个 Slave


以上 Broker 与 Slave 配对是通过指定相同的brokerName 参数来配对，Master的 BrokerId 必须是 0，Slave 的BrokerId 必须是大与 0 的数。另外一个 Master下面可以挂载多个 Slave，同一 Master 下的多个 Slave通过指定不同的 BrokerId来区分。


### 二、RocketMQ 部署（以Master方式为例） ###


笔者在此不打算花费更多的篇幅来陈述RocketMQ的具体部署步骤，特附上网上的一篇博客仅供参考：[分布式消息队列RocketMQ部署与监控](http://sofar.blog.51cto.com/353572/1540874)

尽管这篇博客已经讲述的比较详细了，但是笔者觉得还是补充下相关的配置说明，如下所示：



	#所属集群名字
	brokerClusterName=rocketmq-cluster
	#broker名字，注意此处不同的配置文件填写的不一样
	brokerName=broker-a|broker-b
	#0 表示 Master，>0 表示 Slave
	brokerId=0
	#nameServer地址，分号分割
	namesrvAddr=rocketmq-nameserver1:9876;rocketmq-nameserver2:9876
	#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
	defaultTopicQueueNums=4
	#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
	autoCreateTopicEnable=true
	#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
	autoCreateSubscriptionGroup=true
	#Broker 对外服务的监听端口
	listenPort=10911
	#删除文件时间点，默认凌晨 4点
	deleteWhen=04
	#文件保留时间，默认 48 小时
	fileReservedTime=120
	#commitLog每个文件的大小默认1G
	mapedFileSizeCommitLog=1073741824
	#ConsumeQueue每个文件默认存30W条，根据业务情况调整
	mapedFileSizeConsumeQueue=300000
	#destroyMapedFileIntervalForcibly=120000
	#redeleteHangedFileInterval=120000
	#检测物理文件磁盘空间
	diskMaxUsedSpaceRatio=88
	#存储路径
	storePathRootDir=/usr/local/rocketmq/store
	#commitLog 存储路径
	storePathCommitLog=/usr/local/rocketmq/store/commitlog
	#消费队列存储路径存储路径
	storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
	#消息索引存储路径
	storePathIndex=/usr/local/rocketmq/store/index
	#checkpoint 文件存储路径
	storeCheckpoint=/usr/local/rocketmq/store/checkpoint
	#abort 文件存储路径
	abortFile=/usr/local/rocketmq/store/abort
	#限制的消息大小
	maxMessageSize=65536
	#flushCommitLogLeastPages=4
	#flushConsumeQueueLeastPages=2
	#flushCommitLogThoroughInterval=10000
	#flushConsumeQueueThoroughInterval=60000
	#Broker 的角色
	#- ASYNC_MASTER 异步复制Master
	#- SYNC_MASTER 同步双写Master
	#- SLAVE
	brokerRole=ASYNC_MASTER
	#刷盘方式
	#- ASYNC_FLUSH 异步刷盘
	#- SYNC_FLUSH 同步刷盘
	flushDiskType=ASYNC_FLUSH
	#checkTransactionMessageEnable=false
	#发消息线程池数量
	#sendMessageThreadPoolNums=128
	#拉消息线程池数量
	#pullMessageThreadPoolNums=128




