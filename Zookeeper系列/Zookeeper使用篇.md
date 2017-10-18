### Zookeeper使用篇 ###
***

### 一、Zookeeper安装与部署 ###

Zookeeper主要有两种运行模式：集群模式和单机模式。

#### 1.集群模式 ####

（1）准备：

准备Java运行环境。确保你已经安装了Java 1.6或更高版本的JDK。（以Linux环境为例）

（2）安装：

I、下载Zookeeper安装包并解压到一个指定目录（此处解压到当前用户目录中的 zookeeper目录中）。

	[zk@zookeeper ~]$ tar -xvf zookeeper-3.4.8.tar.gz zookeeper

II、环境变量配置（此步非必须，设置环境变量的目的在于避免每次都到安装目录文件夹内，找到可执行文件来进行操作，这样太繁琐了）

环境变量设置一般有三种方式 —— a)控制台中设置，作用域为当前Shell有效：$PATH="$PATH":/NEW_PATH  ; b)局部环境变量配置,作用域为当前用户：修改个人用户主目录下的.bashrc文件中添加 Export PATH="$PATH:/NEW_PATH" ; c)全局环境变量配置，作用域为所有用户：在/etc/profile的最下面添加 Export PATH="$PATH:/NEW_PATH" 。

	#ZooKeeper环境变量配置
	ZK_HOME=$HOME/zookeeper
	PATH=$ZK_HOME/bin:$PATH
	export ZK_HOME
	export PATH


（3）配置：

I、zoo.cfg配置——(准备两个文件夹data和log用来存放zookeeper的数据和日志，注意两个文件夹权限必须为775)配置文件zoo.cfg —— 初次使用Zookeeper，需要将%ZK_HOME%/conf目录下的zoo_sample.cfg文件重命名为zoo.cfg,并进行相关配置：

	#默认配置项
	tickTime=2000
	initLimit=10
	syncLimit=5
	clientPort=2181
	#数据目录配置（根据本节点情况配置）
	dataDir=/var/lib/zookeeper/data
	dataLogDir=/var/lib/zookeeper/log
	
	#集群节点成员配置
	server.1=IP1:2888:3888
	server.2=IP2:2888:3888
	server.3= IP3:2888:3888



- 在集群模式下，集群中的每台机器都需要感知到整个集群是由哪几台机器组成的，在配置文件中，可以按照这样的格式进行配置，每一行代表一个机器配置：  server.id=host:port:port .(其中，id被称为Server ID，用来标识该机器在集群中的机器序号。同时，在每台Zookeeper机器上，我们都需要在数据目录（即dataDir参数指定的那个目录）下创建一个myid文件，该文件只有一行内容，并且是一个数字，即对应于每台机器的Server ID数字。例如，server.1的myid文件内容就是“1”.)
- 在zookeeper的设计中，集群中所有机器上zoo.cfg文件的内容都应该是一致的。因此最好使用SVN或是GIT把此文件管理起来，确保每个机器都能共享到一份相同的配置。

II、myid配置——创建myid文件。在dataDir所配置的目录下，创建一个名为myid的文件，在该文件的第一行写上一个数字，和zoo.cfg中当前机器的编号对应上。（注意，请确保每个服务器的myid文件中的数字不同，并且和自己所在机器的zoo.cfg中server.id=host:port:port的id值一致。另外，id的范围是1～255）.

（4）按照相同的步骤，为其他机器都安装并配置上zoo.cfg和myid文件。

（6）启动服务器 —— 至此所有的选项都已经基本配置完毕，可以使用%ZK_HOME%/bin目录下的zKServer.sh脚本进行服务器的启动：

	1）. 启动服务器
	[zk@zookeeper ~]$ zkServer.sh start
	Zookeeper JMX enabled by default
	Using config: /home/zk/zookeeper/bin/../conf/zoo.cfg
	Starting zookeeper ... STARTED
	2）. 查看服务器状态
	[zk@zookeeper ~]$ zkServer.sh status
	Zookeeper JMX enabled by def        ault
	Using config: /home/zk/zookeeper/bin/../conf/zoo.cfg
	Mode: standalone
	3）. 停止服务器
	[zk@zookeeper ~]$ zkServer.sh stop
	Zookeeper JMX enabled by default
	Using config: /home/zk/zookeeper/bin/../conf/zoo.cfg
	Stopping zookeeper ... STOPPED
	4）. 重启服务器
	[zk@zookeeper ~]$ zkServer.sh restart
	Zookeeper JMX enabled by default
	Using config: /home/zk/zookeeper/bin/../conf/zoo.cfg
	Zookeeper JMX enabled by default
	Using config: /home/zk/zookeeper/bin/../conf/zoo.cfg
	Stopping zookeeper ... STOPPED
	Zookeeper JMX enabled by default
	Using config: /home/zk/zookeeper/bin/../conf/zoo.cfg
	Starting zookeeper ... STARTED



(7)验证服务器 —— 启动完成后，可以检查下服务器启动是否正常。

方式1：使用Java客户端连接Zookeeper服务器

	[zk@zookeeper ~]$ zkCli.sh -server 127.0.0.1:2181
	Connecting to 127.0.0.1:2181
	...
	[zk: 127.0.0.1:2181(CONNECTED) 0] help
	ZooKeeper -server host:port cmd args
	         connect host:port
	         get path [watch]
	         ls path [watch]
	         set path data [version]
	         rmr path
	         delquota [-n|-b] path
	         quit
	         printwatches on|off
	         create [-s] [-e] path data acl
	         stat path [watch]
	         close
	         ls2 path [watch]
	         history
	         listquota path
	         setAcl path acl
	         getAcl path
	         sync path
	         redo cmdno
	         addauth scheme auth
	         delete path [version]
	         setquota -n|-b val path
	[zk: 127.0.0.1:2181(CONNECTED) 1]


方式2：通过Telnet方式，使用stat命令进行服务器启动的验证，如果出现和上面类似的输出信息，就说明服务器已经正常启动了。

![](https://i.imgur.com/41dFN93.png)


#### 2.单机模式 ####

一般情况在开发测试环境，我们没有那么多机器资源，而且平时的开发调试并不需要极好的稳定性。幸运的是，Zookeeper支持单机部署，只要启动一台Zookeeper机器，就可以提供正常服务了。其实，单机模式只是一种特殊的集群模式而已（只有一台机器的集群）。

单机模式的部署步骤和集群模式的部署步骤基本一致，只是在zoo.cfg文件的配置上有些差异。由于现在我们是单机模式，整个Zookeeper集群中只有一台机器，所以需要对zoo.cfg做如下修改：

	#默认配置项
	tickTime=2000
	initLimit=10
	syncLimit=5
	clientPort=2181
	#数据目录配置（根据本节点情况配置）
	dataDir=/var/lib/zookeeper/data
	dataLogDir=/var/lib/zookeeper/log
	
	#集群节点成员配置（可以省略）
	server.1=IP1:2888:3888
	

和集群模式唯一的区别就在机器列表上，在单机模式的zoo.cfg文件中，只有server.1这一项（可以省略）。修改完这个文件后，就可以启动服务器了。


#### 3.伪集群模式 ####

在集群和单机两种模式下，我们基本完成了分别针对生产环境和开发环境Zookeeper服务的搭建，已经可以满足绝大部分场景了。现在我们再来看看另外一种情况，如果你手上有且只有一台比较好的机器，那么这个时候，如果作为单机模式进行部署，资源明显有点浪费；而如果想要按照集群模式来部署的话，那么就需要借助硬件上的虚拟化技术，将一台物理机器转换成几台虚拟机，不过这样操作成本太高。所幸，和其他分布式系统(如Hadoop)一样，zookeeper也允许你在一台机器上完成一个伪集群的搭建。

所谓伪集群，用一句话说就是，集群所有的机器都在一台机器上，但是还是以集群的特性来对外提供服务。这种模式和集群模式非常类似，只是把zoo.cfg做了如下修改：

	#默认配置项
	tickTime=2000
	initLimit=10
	syncLimit=5
	clientPort=2181
	#数据目录配置（根据本节点情况配置）
	dataDir=/var/lib/zookeeper/data
	dataLogDir=/var/lib/zookeeper/log
	
	#集群节点成员配置（可以省略）
	server.1=IP1:2888:3888
	server.2=IP1:2889:3889
	server.3=IP1:2890:3890

在zoo.cfg配置中，每一行的机器列表配置都是同一个IP地址：IP1，但是后面的端口配置都已经不一样了。这其实并不难理解，在同一台机器上启动多个进程，就必须绑定不同的端口。

### 二、Zookeeper客户端命令 ###

#### 1.客户端连接 ####

在服务端开启的情况下，运行客户端，使用如下命令：

	$ sh zkCli.sh 或者 $ ./zkCli.sh

上面的命令没有显示地制定zookeeper服务器地址，那么默认是连接本地的zookeeper服务器。如果希望连接指定的zookeeper服务器，可以通过如下方式实现：

	$ sh zkCli.sh -server ip:port 或者 $ ./zkCli.sh -server ip:port

#### 2.创建 ####

使用create命令，可以创建一个Zookeeper节点：

    create [-s] [-e] path data acl

其中，-s或-e分别指定节点特性，顺序或临时节点，若不指定，则表示持久节点；acl用来进行权限控制。


#### 3.读取 ####

与读取相关的命令有ls 命令和get 命令。

ls——使用ls命令，可以列出Zookeeper指定节点下的所有子节点。当然，这个命令只能查看指定节点下的第一级的所有子节点。用法如下(其中，path表示的是指定数据节点的节点路径)：

	ls path [watch]



get —— 使用get命令，可以获取Zookeeper指定节点的数据内容和属性信息。用法如下：

	get path [watch]


#### 4.更新 ####

使用set命令，可以更新指定节点的数据内容，用法如下（其中，data就是要更新的新内容，version表示数据版本，如将/zk-permanent节点的数据更新为456，可以使用如下命令：set /zk-permanent 456）：

	set path data [version]

#### 5.删除 ####

使用delete命令可以删除Zookeeper上的指定节点，用法如下（其中version表示数据版本，使用delete /zk-permanent 命令即可删除/zk-permanent节点）：

	delete path [version]

***注意：若删除节点存在子节点，那么无法删除该节点，必须先删除子节点，再删除父节点。***


### 三、Java客户端API使用 ###

#### 1.创建会话 ####

Zookeeper的4种构造方法：

	ZooKeeper（String connectString,int sessionTimeout,Watcher watcher）;
	ZooKeeper（String connectString,int sessionTimeout,Watcher watcher,boolean canBeReadOnly）;
	ZooKeeper（String connectString,int sessionTimeout,Watcher watcher,long sessionId,byte[] sessionPasswd）;
	ZooKeeper（String connectString,int sessionTimeout,Watcher watcher,long sessionId,byte[] sessionPasswd,boolean canBeReadOnly）;

使用任意一个构造方法都可以顺利完成与Zookeeper服务器的会话（Session）创建，笔者偷下懒借用网上的一幅截图列出对每个参数的说明.

![](https://i.imgur.com/4RzLgS6.jpg)

注意，Zookeeper客户端和服务端会话的建立是一个异步的过程，也就是说在程序中，构造方法会在处理完客户端初始化工作后立即返回，在大多数情况下，此时并没有真正建立好一个可用的会话，在会话的生命周期中处于"CONNECTING"的状态。当该会话真正创建完毕后，Zookeeper服务端会向会话对应的客户端发送一个事件通知，以告知客户端，客户端只有在获取这个通知之后，才算真正建立了会话。


#### 2.主要方法 ####

![](https://i.imgur.com/txu8vcb.png)

这里仅列举出了zookeeper API比较常用的几个方法，更详细的介绍请参考： [API DOCS](http://zookeeper.apache.org/doc/r3.4.8/api/index.html)

#### （1）创建节点 ####

Java客户端可以通过ZooKeeper的API来创建一个数据节点，有如下两个接口：
	
	/**同步方式创建节点**/
	String create(final String path,byte data[],List<ACL> acl,CreateMode createMode)
	/**异步方式创建节点**/
	void create(final String path,byte data[],List<ACL> acl,CreateMode createMode,StringCallback cb,Object ctx)

这两个接口分别以同步和异步方式创建节点，API方法的参数说明如下：

![](https://i.imgur.com/RMgdh21.png)

需要注意几点：


- 无论是同步还是异步接口，Zookeeper都不支持递归创建，即无法在父节点不存在的情况下创建一个子节点。另外，如果一个节点已经存在了，那么创建同名节点的时候，会抛出NodeExistsException异常。
- zookeeper的节点内容只支持字节数组（byte[]）类型，也就是说zookeeper不负责为节点内容进行序列化，开发人员需要自己使用序列化工具将节点内容进行序列化和反序列化。对于字符串，可以简单的使用"string".getBytes()来生成一个字节数组；对于其他复杂对象可以使用Hessian或是Kryo等专门的序列化工具来进行序列化。
- 关于权限控制，如果你的应用场景没有太高的权限要求，那么可以不关注这个参数，只需要在acl参数中传入参数Ids。OPEN_ACL_UNSAFE，这就表明之后对这个节点的任何操作都不受权限控制。（关于Zookeeper的权限控制，《Zookeeper原理篇》将做详细讲解）
- 使用异步方式与同步方式的区别在于节点的创建过程（包括网络通信和服务端的节点创建过程）是异步的。并且，在同步接口调用过程中，开发者需要关注接口抛出异常的可能；但是在异步接口中，接口本身是不会抛出异常，所有的异常都会在回调函数中通过Result Code（响应码rc）来体现。(使用异步方式创建接口也很简单，用户仅仅需要实现AsyncCallback中的回调接口（AsyncCallback包含了StatCallback、DataCallback、ACLCallback、ChildrenCallback、Children2Callback、StringCallback和VoidCallback七种不同的回调接口，用户可以在不同的异步接口中实现不同的接口）)。


#### （2）删除节点 ####

	public void delete(final String path,int version)
	public void delete(final String path,int versino,voidCallback cb,Object ctx)

![](https://i.imgur.com/cew00la.png)


#### （3）读取数据 ####











### 四、Zookeeper的典型应用场景 ###
































