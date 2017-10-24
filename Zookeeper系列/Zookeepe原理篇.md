### Zookeepe原理篇 ###
***

### 一、Zookeeper系统模型 ###

下面我们将从数据模型、节点特性、版本、Watcher和ACL五个方面来讲述Zookeeper的系统模型。


#### 1.数据模型 ####

Zookeeper的视图结构和标准的Unix文件系统非常类似，但没有引入传统文件系统中目录和文件等相关概念，而是使用了其特有的“数据节点”概念，我们称之为ZNode。ZNode是Zookeeper中数据的最小单元，每个ZNode上都可以保存数据，同时还可以挂载子节点，因此构成了一个层次化的命名空间，称为树。

![](http://img1.51cto.com/attachment/201211/085951545.png)

节点的路径也是使用典型的、绝对的、斜线分隔的路径样式，没有相对的引用。任何unicode字符都可以被用在路径中，除了下面的限制：

	 1）、路径名中不能包括null（\u0000）字符。（这是因为为了兼容C语言的问题）
     2）、下面的字符也不能使用，因为不能很好的显示，或者显示不正常：\u0001-\u0019和\u007F-\u009F。
     3）、下面的字符也不被允许：\ud800-uF8FFF,\uFF0-uFFFF,-uXFFFE-\uXFFFF（这里的X表示1-E）,\uF0000-\uFFFF。
     4）、点号“.”能作为路径名称的一部分，但是“.”和“..”不能单独用来表示整个路径名称，因为ZooKeeper不使用相对路径。下面的路径也是无效的：“/a/b/./c”or“/a/b/../c”。
     5）、“zookeeper”也是被保留的关键字，不允许使用。



**事务ID ———— ZXID ：**

在Zookeeper中，事务是指能够改变Zookeeper服务器状态的操作，我们也称之为事务操作或更新操作，一般包括节点创建与删除，数据节点内容更新和客户端会话创建与失效等操作。对于每一个事务请求，Zookeeper都会为其分配一个全局唯一的事务ID，用ZXID表示，通常是一个64位的数字。每一个ZXID对应一次更新操作，从这些ZXID中可以间接地识别出Zookeeper处理这些更新操作请求的全局顺序。

#### 2.节点特性 ####

在Zookeeper中，每个数据节点都是有生命周期的，其生命周期的长短取决于数据节点的节点类型。在Zookeeper中，节点类型可以分为持久节点（PERSISTENT）、临时节点（EPHEMERAL）和顺序节点（SEQUENTIAL）三大类，具体在节点创建过程中，通过组合使用，可以生成一下四种组合型节点类型：

- 持久节点(PERSISTENT)———— 所谓持久节点，是指该数据节点被创建后，就会一直存在于Zookeeper服务器上，直到有删除操作来主动清除该节点。

- 持久顺序节点(PERSISTENT_SEQUENTIAL) ———— 相比持久节点，其新增了顺序特性，每个父节点都会为它的第一级子节点维护一份顺序，用于记录每个子节点创建的先后顺序。在创建节点时，会自动添加一个数字后缀，作为一个新的、完整的节点名。需要注意的是，这个数字后缀的上限是整型的最大值。
- 临时节点（EPEMERAL）———— 临时节点的生命周期与客户端的会话绑定在一起，也就是说，如果客户端会话失效，那么这个节点会被自动清理掉。（注意，这里提到的是客户端会话失效，而非TCP连接断开。）同时，Zookeeper规定不能基于临时节点来创建子节点，即临时节点只能作为叶子节点。
- 临时顺序节点（EPEMERAL_SEQUENTIAL）———— 在临时节点的基础上添加了顺序特性。

每个数据节点除了存储了数据内容之外，还存储了数据节点本身的一些状态信息，可通过命令get获取：

	[zk: 10.1.39.43:4180(CONNECTED) 7] get /hello  
	world  
	cZxid = 0x10000042c  
	ctime = Fri May 17 17:57:33 CST 2013  
	mZxid = 0x10000042c  
	mtime = Fri May 17 17:57:33 CST 2013  
	pZxid = 0x10000042c  
	cversion = 0  
	dataVersion = 0  
	aclVersion = 0  
	ephemeralOwner = 0x0  
	dataLength = 5  
	numChildren = 0  


我们可以看到，第一行是当前数据节点的数据内容，从第二行开始就是节点的状态信息了，包括事务ID、版本信息和子节点个数等：

- czxid —— 即Created ZXID，表示该数据节点被创建时的事务ID
- mzxid —— 即Modified ZXID，表示该节点最后一次被更新时的事务ID
- ctime —— 即Created Time，表示节点被创建的时间
- mtime —— 即Modified Time，表示该节点最后一次被更新的时间
- dataVersion —— 数据节点的版本号（节点数据的更新次数）.
- cversion —— 子节点的版本号（子节点的更新次数）.
- aclVersion —— 节点的ACL版本号（节点ACL(授权信息)的更新次数）.
- ephemeralOwner —— 创建该临时节点的会话的sessionID.(如果该节点为ephemeral节点, ephemeralOwner值表示与该节点绑定的session id. 如果该节点不是ephemeral节点, ephemeralOwner值为0.)
- dataLength ——  数据内容的长度（节点数据的字节数）.
- numChildren —— 当前节点的子节点个数.
- pzxid —— 表示该节点的子节点列表最后一次被修改时的事务ID。（注意，只有子节点列表变更了才会变更pzxid,子节点内容变更不会影响pzxid）



#### 3.版本 ———— 保证分布式数据原子性操作 ####

Zookeeper中为数据节点引入了版本的概念，每个数据节点都具有三种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化。



- version ———— 当前数据节点数据内容的版本号。
- cversion ———— 当前数据节点子节点的版本号。
- aversino ———— 当前数据节点ACL变更版本号。

Zookeeper中的版本概念和传统意义上的软件版本有很大的区别，它表示的是对数据节点的数据内容、子节点列表或是节点ACL信息的修改次数。如：version为1表示对数据节点的内容变更了一次。即使前后两次变更并没有改变数据内容，version的值仍然会改变。version可以用于写入验证，类似于CAS。

![](http://img.blog.csdn.net/20160702113356844)


#### 4.Watcher ———— 数据变更的通知 ####

在Zookeeper中，引入了Watcher机制来实现这种分布式的通知功能。Zookeeper允许客户端向服务端注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161122094930487-367521759.png)

　Zookeeper的Watcher机制主要包括客户端线程、客户端WatcherManager、Zookeeper服务器三部分。客户端在向Zookeeper服务器注册的同时，会将Watcher对象存储在客户端的WatcherManager当中。当Zookeeper服务器触发Watcher事件后，会向客户端发送通知，客户端线程从WatcherManager中取出对应的Watcher对象来执行回调逻辑。

#### 5.ACL ———— 保障数据的安全 ####

Zookeeper内部存储了分布式系统运行时状态的元数据，这些元数据会直接影响基于Zookeeper进行构造的分布式系统的运行状态，如何保障系统中数据的安全，从而避免因误操作而带来的数据随意变更而导致的数据库异常十分重要，Zookeeper提供了一套完善的ACL权限控制机制来保障数据的安全。

　我们可以从三个方面来理解ACL机制：**权限模式（Scheme）、授权对象（ID）、权限（Permission）**，通常使用"scheme:id:permission"来标识一个有效的ACL信息。

(1)权限模式（Scheme）

权限模式用来确定权限验证过程中使用的检验策略，有如下四种模式：



- IP，通过IP地址粒度来进行权限控制，如"ip:192.168.0.110"表示权限控制针对该IP地址，同时IP模式可以支持按照网段方式进行配置，如"ip:192.168.0.1/24"表示针对192.168.0.*这个网段进行权限控制。
- Digest，使用"username:password"形式的权限标识来进行权限配置，便于区分不同应用来进行权限控制。Zookeeper会对其进行SHA-1加密和BASE64编码。
-  World，最为开放的权限控制模式，数据节点的访问权限对所有用户开放。（World模式也可以看作是一种特殊的Digest模式，它只有一个权限标识，即“World:anyone”.）
-  Super，超级用户，也是一种特殊的Digest模式，超级用户可以对任意Zookeeper上的数据节点进行任何操作。

（2）授权对象

授权对象是指权限赋予的用户或一个指定实体，如IP地址或机器等。不同的权限模式通常有不同的授权对象。

![](https://i.imgur.com/NaUhgAP.png)

（3）权限

权限是指通过权限检查可以被允许执行的操作，Zookeeper对所有数据的操作权限分为：CREATE（节点创建权限）、DELETE（节点删除权限）、READ（节点读取权限）、WRITE（节点更新权限）、ADMIN（节点管理权限）。

- CREATE（C）：数据节点的创建权限，允许授权对象在该数据节点下创建子节点。
- DELETE（D）:子节点的删除权限，允许授权对象删除该数据节点的子节点。
- READ（R）:数据节点的读取权限，允许授权对象访问该数据节点并读取其数据内容或子节点列表等。
- WRITE（W）:数据节点的更新权限，允许授权对象对该数据节点进行更新操作。
- ADMIN（A）:数据节点的管理权限，允许授权对象对该数据节点进行ACL相关的设置操作。

在绝大部分的场景下，这四种权限模式已经能够很好的实现权限控制的目的。同时，Zookeeper提供了特殊的权限控制插件体系，允许开发人员通过制定方式对Zoookeeper的权限进行自定义扩展。

### 二、Zookeeper序列化与协议 ###






### 三、Zookeeper客户端 ###




### 四、Zookeeper会话 ###





### 五、Zookeeper服务端启动 ###




### 六、Zookeeper Leader选举 ###




### 七、Zookeeper各服务器角色介绍 ###




### 八、Zookeeper请求处理 ###





### 九、Zookeeper数据与存储 ###









