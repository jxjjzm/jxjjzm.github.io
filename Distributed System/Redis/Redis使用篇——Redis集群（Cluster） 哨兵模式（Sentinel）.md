### Redis使用篇——Redis集群（Cluster）| 哨兵模式（Sentinel） ###
***

### 一、哨兵（Redis Sentinel） ###


Sentinel（哨兵）是Redis 的高可用性解决方案：由一个或多个Sentinel 实例 组成的Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。



- 监控（Monitoring）：Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
- 提醒（Notification）：当被监控的某个Redis 服务器出现问题时，Sentinel 可以通过API 向管理员或者其他应用程序发送通知。
- 自动故障迁移（Automatic failover）：当一个主服务器不能正常工作时，Sentinel 会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器；当客户端试图连接失效的主服务器时，集群也会向客户端返回新主服务器的地址，使得集群可以使用新主服务器代替失效服务器。


Sentinel的工作方式：



- 1)：每个Sentinel以每秒钟一次的频率向它所知的Master，Slave以及其他 Sentinel 实例发送一个 PING 命令 
- 2)：如果一个实例（instance）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被 Sentinel 标记为主观下线。 
- 3)：如果一个Master被标记为主观下线，则正在监视这个Master的所有 Sentinel 要以每秒一次的频率确认Master的确进入了主观下线状态。 
- 4)：当有足够数量的 Sentinel（大于等于配置文件指定的值）在指定的时间范围内确认Master的确进入了主观下线状态， 则Master会被标记为客观下线 
- 5)：在一般情况下， 每个 Sentinel 会以每 10 秒一次的频率向它已知的所有Master，Slave发送 INFO 命令 
- 6)：当Master被 Sentinel 标记为客观下线时，Sentinel 向下线的 Master 的所有 Slave 发送 INFO 命令的频率会从 10 秒一次改为每秒一次 
- 7)：若没有足够数量的 Sentinel 同意 Master 已经下线， Master 的客观下线状态就会被移除；若 Master 重新向 Sentinel 的 PING 命令返回有效回复， Master 的主观下线状态就会被移除。




### 二、Redis集群（Redis Cluster） ###

Redis集群是分布式数据库方案，通过分片来进行数据共享，并提供复制和故障转移功能


#### 1.节点 ####

一个Redis 集群通常由多个节点组成，在刚开始的时候，每个节点都是独立的，只处于只包含自己的集群中，当要组成一个真正可工作的集群时，就需要将这些独立的节点连接起来，构建成一个包含多个节点的集群。

如何连接各个节点：向一个节点发送下面的命令，可以让节点与ip和port所指定的节点进行握手，握手成功，节点就会将ip和port指定的节点添加到当前的集群中。

	CLUSTER MEET <ip> <port>


I、数据结构

每个节点都会使用一个 clusterNode 结构来记录自己的状态（比如节点的创建时间， 节点的名字， 节点当前的配置纪元， 节点的 IP 和地址...）， 并为集群中的所有其他节点（包括主节点和从节点）都创建一个相应的 clusterNode 结构， 以此来记录其他节点的状态：


	//一个节点的当前状态
	struct clusterNode{    
	    // 创建节点的时间    
	    mstime_t ctime;    
	    // 节点的名字，由40个16进制字符组成    
	    char name[REDIS_CLUSTER_NAMELEN];    
	    // 节点标识    
	    int flags;    
	    // 节点当前的配置纪元，用于实现故障转移    
	    uint64_t configEpoch;    
	    // 节点的ip地址    
	    char ip[REDIS_IP_STR_LEN];    
	    // 节点的端口号    
	    int port；    
	    // 保存连接节点所需的有关信息   
	    clusterLink *link；    
	    //……
	};


clusterNode 结构的 link 属性是一个 clusterLink 结构， 该结构保存了连接节点所需的有关信息， 比如套接字描述符， 输入缓冲区和输出缓冲区、与这个连接相关联的节点：

	typedef struct clusterLink {
	
	    // 连接的创建时间
	    mstime_t ctime;
	
	    // TCP 套接字描述符
	    int fd;
	
	    // 输出缓冲区，保存着等待发送给其他节点的消息（message）。
	    sds sndbuf;
	
	    // 输入缓冲区，保存着从其他节点接收到的消息。
	    sds rcvbuf;
	
	    // 与这个连接相关联的节点，如果没有的话就为 NULL
	    struct clusterNode *node;
	
	} clusterLink;



最后， 每个节点都保存着一个 clusterState 结构， 这个结构记录了在当前节点的视角下， 集群目前所处的状态 —— 比如集群是在线还是下线， 集群包含多少个节点， 集群当前的配置纪元， 诸如此类：

	typedef struct clusterState {
	
	    // 指向当前节点的指针
	    clusterNode *myself;
	
	    // 集群当前的配置纪元，用于实现故障转移
	    uint64_t currentEpoch;
	
	    // 集群当前的状态：是在线还是下线
	    int state;
	
	    // 集群中至少处理着一个槽的节点的数量
	    int size;
	
	    // 集群节点名单（包括 myself 节点）
	    // 字典的键为节点的名字，字典的值为节点对应的 clusterNode 结构
	    dict *nodes;
	
	    // ...
	} clusterState;


II、CLUSTER MEET命令的实现

通过向节点 A 发送 CLUSTER MEET 命令， 客户端可以让接收命令的节点 A 将另一个节点 B 添加到节点 A 当前所在的集群里面：

	CLUSTER MEET <ip> <port>

收到命令的节点 A 将与节点 B 进行握手（handshake）， 以此来确认彼此的存在， 并为将来的进一步通信打好基础：


- 节点 A 会为节点 B 创建一个 clusterNode 结构， 并将该结构添加到自己的 clusterState.nodes 字典里面。
- 之后， 节点 A 将根据 CLUSTER MEET 命令给定的 IP 地址和端口号， 向节点 B 发送一条 MEET 消息（message）。
- 如果一切顺利， 节点 B 将接收到节点 A 发送的 MEET 消息， 节点 B 会为节点 A 创建一个 clusterNode 结构， 并将该结构添加到自己的 clusterState.nodes 字典里面。
- 之后， 节点 B 将向节点 A 返回一条 PONG 消息。
- 如果一切顺利， 节点 A 将接收到节点 B 返回的 PONG 消息， 通过这条 PONG 消息节点 A 可以知道节点 B 已经成功地接收到了自己发送的 MEET 消息。
- 之后， 节点 A 将向节点 B 返回一条 PING 消息。
- 如果一切顺利， 节点 B 将接收到节点 A 返回的 PING 消息， 通过这条 PING 消息节点 B 可以知道节点 A 已经成功地接收到了自己返回的 PONG 消息， 握手完成。

![](http://redisbook.com/_images/graphviz-76550f578052d5f52e35414facb5ac680d38c7f5.png)

之后， 节点 A 会将节点 B 的信息通过 Gossip 协议传播给集群中的其他节点， 让其他节点也与节点 B 进行握手， 最终， 经过一段时间之后， 节点 B 会被集群中的所有节点认识。


#### 2.槽指派 ####

Redis集群通过分片的方式来保存数据库中的键值对：集群的整个数据库被分为1384个槽(slot),数据库中的每个键都属于这16384个槽的其中一个，集群中的每个节点可以处理0个或最多16384个槽。

当数据库中的16384个槽都有节点在处理时，集群处于上线状态(OK);相反地，如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态(fail)。

I、数据结构

	// 一个节点的当前状态
	struct clusterNode{
	    //……
	    // 记录处理那些槽
	    // 二进制位数组，长度为 2048 个字节，包含 16384 个二进制位
	    // 如果slots数组在索引i上的二进制位的值为1，那么表示节点负责处理槽i；否则表示节点不负责处理槽i
	    unsigned char slots[16384/8];
	    //记录自己负责处理的槽的数量
	    int numslots;
	
	    //……
	};


![](https://i.imgur.com/LmPW0QK.png)

- 一个节点除了会将自己负责处理的槽记录在clusterNode结构的slots属性和numslots属性之外，它还会将自己的slots数组通过消息发送给集群中其他的节点，以此来告知其他节点自己目前负责处理哪些槽。
- 当节点A通过消息从节点B那里接收到节点B的slots数组时，节点A会在自己的clusterState.nodes字典中查找节点B对应的clusterNode结构，并对结构中的slots数组进行保存或者更新。

clusterState结构中的slots数组记录了集群中所有16384个槽的指派信息：

	    // 当前节点视角下集群目前所处的状态,集群中所有16384个槽的指派信息
    	typedef struct clusterState{
    		// ……
    		// slots[i]指针如果指向NULL，说明槽i尚未被指派给任何节点；
    		// slots[i]指针如果指向一个clusterNode 结构，
    		// 说明槽i已经被指派给了这个clusterNode结构所代表的节点；
    		clusterNode *slots[16384];
    		// ……
    	};


slots数组包含16384个项，每个数组项都是一个指向clusterNode结构的指针

![](https://i.imgur.com/DnJQrXT.png)

- 如果slots[i]指针指向NULL，那么表示槽i尚未指派给任何节点。
- 如果slots[i]指针指向一个clusterNode结构，那么表示槽i已经指派给了clusterNode结构所代表的节点。


从这里我们可以发现，clusterNode 和 clusterState


如果只将槽指派信息保存在各个节点的clusterNode.slots数组里，会出现一些无法高效地解决的问题，而clusterState.slots数组的存在解决了这些问题：

- 如果节点只使用clusterNode.slots数组来记录槽的指派信息，那么为了知道槽i是否已经被指派，或者槽i被指派给了哪个节点，程序需要遍历clusterState。nodes字典中的所有clusterNode结构，检查这些结构的slots数组，直到找到负责处理槽i的节点位置，这个过程的复杂度为O(N)
- 而通过将所有槽的指派信息保存在clusterState.slots数组里面，程序要检查槽i是否已经被指派，又或者取得负责处理槽i的节点，只需要访问clusterState.slots[i]的值即可，这个操作的复杂度仅为O(1).


虽然clusterState.slots数组记录了集群中所有槽的指派信息，但使用clusterNode结构的slots数组来记录单个节点的槽指派信息仍然是有必要的：

- 因为当程序员需要将某个节点的槽指派信息通过消息发送给其他节点时，程序只需要将相应节点的clusterNode.slots数组整个发送出去就可以了。
- 另一方面，如果Redis不适用clusterNode.slots数组，而单独使用clusterState.slots数组的话，那么每次要将节点A的槽指派信息传播给其他节点时，程序必须先遍历整个clusterState.slots数组，记录节点A负责处理哪些槽，然后才能发送A的槽指派信息，这比直接发送clusterNode.slots数组要麻烦和低效得多。



II、CLUSTER ADDSLOTS <slot>

通过执行CLUSTER ADDSLOTS <slot>命令指派槽给当前节点，这个命令接受一个或多个槽作为参数，将所有输入的槽指派给接受该命令的节点负责，这个命令的实现如下：
	
	CLUSTER ADDSLOTS <slot>

- 检查参数中的槽是否存在已经被其它节点处理的，如果存在直接返回错误。
- 对i号槽，clusterState.slots[i]设成当前节点对应的clusterNode，并且更新clusterNode的slots数组。

#### 3.在集群中执行命令 ####

当客户端向节点发送与数据库键有关的命令时，接收命令的节点会计算出命令要处理的数据库键属于哪个槽，并检查这个槽是否指派给了自己：

- 如果键所在的槽正好就指派给了当前节点，那么节点直接执行这个命令。
- 如果键所在的槽并没有指派给当前节点，那么节点会向客户端返回一个MOVED错误，指引客户端转向(redirect)至正确的节点，并再次发送之前想要执行的命令。

![](https://img-blog.csdn.net/20181012154559151?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2p1bjgxNDg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


其中，节点使用以下算法来计算给定键Key属于哪个槽：slot = CRC16(key)&16383  （说明：CRC1(key)语句用于计算键key的CRC-16校验和）

#### 4.重新分片 ####

Redis集群的重新分片操作可以将任意数量已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点移动到目标节点。 重新分片可以在线进行，在这过程中，集群不用下线，且源节点和目标节点都可以继续处理命令。

Redis集群的重新分片操作是由Redis的集群管理软件redis-trib负责执行的，Redis提供了进行重新分片所需的所有命令，而redis-trib则通过向源节点和目标节点发送命令来进行重新分片操作。

redis-trib对集群的单个槽 slot 进行重新分片的步骤如下：

- 1. redis-trib 对目标节点发送 CLUSTER SETSLOT <slot> IMPORTING <source_id> 命令，让目标节点准备好从源节点导入槽 slot 的键值对
- 2. redis-trib 对源节点发送 CLUSTER SETSLOT <slot> MIGRATING <source_id> 命令，让源节点准备好将属于槽 slot的键值对迁移至目标节点
- 3. redis-trib 对源节点发送 CLUSTER GETKEYSINSLOT<slot> <count> 命令，获得最多 count 个属于槽 slot 的键值对的键名。
- 4. 对于步骤三获得的每个键名，redis-trib 都向源节点发送一个MIGRATE <target_ip> <target_port> <key_name> 0 <timeout> 命令，将被选中的键原子的从源节点迁移至目标节点.
- 5. 重复步骤3和4，直到源节点保存的所有属于槽slot的键值对都被迁移到目标节点为止。
- 6. redis-trib向集群中的任意一个节点发送 CLUSTER SETSLOT <slot> NODE <target_id> 命令，将槽slot指派给目标节点的信息发送给整个集群。

![](https://i.imgur.com/REWFStP.png)

#### 5.ASK错误 ####

在进行重新分片期间，源节点向目标节点迁移一个槽的过程中，可能会出现这样一种情况：属于被迁移槽的一部分键值对保存在源节点里面，而另一部分键值对则保存在目标节点里面。

当客户端向源节点发送一个与数据库键有关的命令，并且命令要处理的数据库键恰好就属于正在被迁移的槽时：

- 1. 源节点在本节点的数据库中查找指定的键，如果找到直接执行客户端发送的命令。
- 2. 如果找不到，说明键已经被迁移到目标节点，源节点向客户端返回一个ASK错误，指引客户端转向目标节点。
- 3.客户端执行ASKING命令到目标节点，目标节点接收到ASKING命令之后会打开REDIS_ASKING标识。
- 4. 客户端重新发送原本想要执行的命令到目标节点。


![](https://i.imgur.com/jbs9JOs.png)


#### 6.复制与故障转移 ####

Redis集群中的节点分为主节点和从节点，其中主节点用于处理槽，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。

**故障检测**

集群中每个节点都会定期向集群中其它节点发送PING消息，以此来检查对方是否在线，如果接收PING消息的结点在规定的时间内没有返回PONG消息，那么发送PING消息的节点会将接收消息的PING节点标记为疑似下线（PFAIL），修改节点对应的clusterNode结构的flags属性，打开REDIS_NODE_PFAIL标识。之后节点A会把节点B的疑似下线消息传播给集群中其它节点，当结点C通过消息获知节点B进入疑似下线状态时，把节点A发送的节点B的下线报告记录到B对应的clusterNode结构的fail_reports链表中。当一个集群中半数以上负责处理槽的主节点都将某个主节x点报告为疑似下线时，那么这个主节点将会被标记为下线（FAIL），并发送一条节点x的FAIL消息通知其它主节点主节点x已下线。



**故障转移**

当一个从节点发现自己正在复制的主节点进入了已下线状态，从节点将开始对下线主节点进行故障转移:



- 复制下线主节点的所有从节点里面，会有一个从节点被选中
- 被选中的从节点将执行 slaveof no one 命令，成为新的主节点
- 新的主节点撤销已下线主节点对指派槽的管理，并将这些槽全部指派给自己
- 新的主节点向集群广播一条PONG消息，告诉集群中的其他节点自己成为了新的主节点。
- 新的主节点开始接收和自己负责处理的槽有关的命令请求，故障转移完成。


**选举新的主节点：**



- 集群的配置纪元是一个自增计数器，初始值为0
- 当集群里的某个节点开始一次故障转移操作时，集群配置纪元的值就会加一
- 对于每个配置纪元，集群中的每个负责处理槽的主节点都有一次投票机会，而第一个向主节点要求投票的从节点将获得主节点的投票
- 当从节点发现自己正在复制的主节点进入已下线状态时，从节点会向集群广播一条 CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST 消息，要求所有收到这条消息、并具有投票权的主节点向这个从节点投票
- 如果一个主节点具有投票权(它正在负责处理槽),并且这个主节点尚未投票给其他从节点，那么主节点将向要求投票的从节点返回一条 CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 消息，表示这个主节点支持从节点成为新的主节点
- 每个参与选举的从节点都会接受 CLUSTERMSG_TYPE_FAILOVER_AUTH_ACK 消息，并根据自己受到了多少条这种消息来统计自己获得了多少主节点 的支持
- 如果集群库有N个具有投票权的主节点，那么当一个从节点收集到大于等于N/2+1张支持票，这个从节点当选为新的主节点
- 因为在每个配置纪元里面，每个具有投票权的主节点只能投一次票，所以如果有N个节点进行投票，那么具有大于等于N/2+1张支持票的从节点只会有一个，这确保了新的主节点只会有一个
- 如果在一个配置纪元里没有从节点能手机到足够多的支持票，那么集群进入一个新的配置纪元，并再次进行选举，直到选出新的主节点为止。




#### 7.消息 ####


### 三、Codis ###





















































































