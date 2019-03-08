### RocketMQ概述篇 —— RocketMQ 消息存储 ###
***

### 一、MQ消息队列的一般存储方式 ###

当前业界几款主流的MQ消息队列采用的存储方式主要有以下三种方式：

#### I、分布式KV存储 ####

消息存储于分布式KV需要解决的问题在于如何保证MQ整体的可靠性？


#### II、文件系统 ####

目前业界较为常用的几款产品（RocketMQ/Kafka/RabbitMQ）均采用的是消息刷盘至所部署虚拟机/物理机的文件系统来做持久化（刷盘一般可以分为异步刷盘和同步刷盘两种模式）。消息刷盘为消息存储提供了一种高效率、高可靠性和高性能的数据持久化方式，除非部署MQ机器本身或是本地磁盘挂了，否则一般是不会出现无法持久化的故障问题。

#### III、关系型数据库DB ####

Apache下开源的另外一款MQ—ActiveMQ（默认采用的KahaDB做消息存储）可选用JDBC的方式来做消息持久化，通过简单的xml配置信息即可实现JDBC消息存储。由于普通关系型数据库（如Mysql）在单表数据量达到千万级别的情况下，其IO读写性能往往会出现瓶颈。因此，如果要选型或者自研一款性能强劲、吞吐量大、消息堆积能力突出的MQ消息队列，那么小编并不推荐采用关系型数据库作为消息持久化的方案。在可靠性方面，该种方案非常依赖DB，如果一旦DB出现故障，则MQ的消息就无法落盘存储会导致线上故障；

因此，综合上所述从存储效率来说， 文件系统>分布式KV存储>关系型数据库DB


### 二、RocketMQ消息存储整体架构 ###


#### 1.RocketMQ存储相关的模型 ####

![](https://i.imgur.com/5UW7qoj.png)


- CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容。单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件；


- Consume Queue：消息消费的逻辑队列，其中包含了这个MessageQueue在CommitLog中的起始物理位置偏移量offset，消息实体内容的大小和Message Tag的哈希值。从实际物理存储来说，ConsumeQueue对应每个Topic和QueuId下面的文件。单个文件大小约5.72M，每个文件由30W条数据组成，每个文件默认大小为600万个字节，当一个ConsumeQueue类型的文件写满了，则写入下一个文件；


- IndexFile：用于为生成的索引文件提供访问服务，通过消息Key值查询消息真正的实体内容。在实际的物理存储上，文件名则是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引；


#### 2.RocketMQ存储架构 ####


![](https://i.imgur.com/CRLAQ8e.png)



- （1）消息生产与消息消费相互分离，Producer端发送消息最终写入的是CommitLog（消息存储的日志数据文件），Consumer端先从ConsumeQueue（消息逻辑队列）读取持久化消息的起始物理位置偏移量offset、大小size和消息Tag的HashCode值，随后再从CommitLog中进行读取待拉取消费消息的真正实体内容部分；


- （2）RocketMQ的CommitLog文件采用混合型存储（所有的Topic下的消息队列共用同一个CommitLog的日志数据文件），并通过建立类似索引文件—ConsumeQueue的方式来区分不同Topic下面的不同MessageQueue的消息，同时为消费消息起到一定的缓冲作用（只有ReputMessageService异步服务线程通过doDispatch异步生成了ConsumeQueue队列的元素后，Consumer端才能进行消费）。这样，只要消息写入并刷盘至CommitLog文件后，消息就不会丢失，即使ConsumeQueue中的数据丢失，也可以通过CommitLog来恢复。


- （3）RocketMQ每次读写文件的时候真的是完全顺序读写么？这里，发送消息时，生产者端的消息确实是顺序写入CommitLog；订阅消息时，消费者端也是顺序读取ConsumeQueue，然而根据其中的起始物理位置偏移量offset读取消息真实内容却是随机读取CommitLog。 在RocketMQ集群整体的吞吐量、并发量非常高的情况下，随机读取文件带来的性能开销影响还是比较大的，那么这里如何去优化和避免这个问题呢？


**I、Mmap内存映射技术—MappedByteBuffer**

Mmap内存映射和普通标准IO操作的本质区别在于它并不需要将文件中的数据先拷贝至OS的内核IO缓冲区，而是可以直接将用户进程私有地址空间中的一块区域与文件对象建立映射关系，这样程序就好像可以直接从内存中完成对文件读/写操作一样。只有当缺页中断发生时，直接将文件从磁盘拷贝至用户态的进程空间内，只进行了一次数据拷贝。对于容量较大的文件来说（文件大小一般需要限制在1.5~2G以下），采用Mmap的方式其读/写的效率和性能都非常高。

![](https://mmbiz.qpic.cn/mmbiz_png/AtZHGo2bvzKJ66ic95yndHEyPjrntbUKDcScf3IGG13djUPVCMoALuKYIlo4eXpbX4uc9eibwQWjPvtZY0DUYmcQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)






**II、OS的PageCache机制**

PageCache是OS对文件的缓存，用于加速对文件的读写。一般来说，程序对文件进行顺序读写的速度几乎接近于内存的读写访问，这里的主要原因就是在于OS使用PageCache机制对读写访问操作进行了性能优化，将一部分的内存用作PageCache。


- （1）对于数据文件的读取，如果一次读取文件时出现未命中PageCache的情况，OS从物理磁盘上访问读取文件的同时，会顺序对其他相邻块的数据文件进行预读取（ps：顺序读入紧随其后的少数几个页面）。这样，只要下次访问的文件已经被加载至PageCache时，读取操作的速度基本等于访问内存。
- （2）对于数据文件的写入，OS会先写入至Cache内，随后通过异步的方式由pdflush内核线程将Cache内的数据刷盘至物理磁盘上。



对于文件的顺序读写操作来说，读和写的区域都在OS的PageCache内，此时读写性能接近于内存。RocketMQ的大致做法是，将数据文件映射到OS的虚拟内存中（通过JDK NIO的MappedByteBuffer），写消息的时候首先写入PageCache，并通过异步刷盘的方式将消息批量的做持久化（同时也支持同步刷盘）；订阅消费消息时（对CommitLog操作是随机读取），由于PageCache的局部性热点原理且整体情况下还是从旧到新的有序读，因此大部分情况下消息还是可以直接从Page Cache中读取，不会产生太多的缺页（Page Fault）中断而从磁盘读取

![](https://mmbiz.qpic.cn/mmbiz_png/AtZHGo2bvzKJ66ic95yndHEyPjrntbUKDHdibbqyQRAFS4tHxSn68AaRNKtxHb2MHTI0olELD7iaCVpBVFMGFms6w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

PageCache机制也不是完全无缺点的，当遇到OS进行脏页回写，内存回收，内存swap等情况时，就会引起较大的消息读写延迟。


### 附：推荐文献 ###

- [RocketMQ高性能之底层存储设计](https://www.jianshu.com/p/d06e9bc6c463)
- [消息中间件—RocketMQ消息存储（一）](https://mp.weixin.qq.com/s/mPxsA4_ZVI3PUDU3_3o41g)
- [消息中间件—RocketMQ消息存储（二）](https://mp.weixin.qq.com/s/x5_XeMfkVMOX8p3ziOKl5Q)
















































































































