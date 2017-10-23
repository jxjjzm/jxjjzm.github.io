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

![](https://i.imgur.com/uEW7tQ0.png)

详细请参考： [API DOCS](http://zookeeper.apache.org/doc/r3.4.8/api/index.html)

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

需要注意：

- 在ZooKeeper中，只允许删除叶子节点。也就是说，如果一个节点存在至少一个子节点的话，那么该节点将无法被直接删除，必须先删除掉其所有子节点。

#### （3）读取数据 ####

I、getChildren

Java客户端可以通过Zookeeper的API来获取一个节点的所有子节点，有如下接口可供使用：

	List<String> getChildren(final String path,Watcher watcher)
	List<String> getChildren(final String path,boolean watch)
	void getChildren(final String path,Watcher watcher,ChildrenCallback cb,Object ctx)
	void getChildren(final String path,boolean watch,ChildrenCallback cb,Object ctx)
	List<String> getChildren(final String path,Watcher watcher,Stat stat)
	List<String> getChildren(final String path,boolean watch,Stat stat)
	void getChildren(final String path,Watcher watcher,Children2Callback cb,Object ctx)
	void getChildren(final String path,boolean watch,Children2Callback cb,Object ctx)

![](https://i.imgur.com/HtT3Yko.png)
![](https://i.imgur.com/lCCTepd.png)

需要注意：


- Watcher通知是一次性的，即一旦触发一次通知后，该Watcher就失效了，因此客户端需要反复注册Watcher


II、getData

客户端可以通过Zookeeper的API来获取一个节点的数据内容，有如下接口:

	byte[] getData(final String path,Watcher watcher,Stat stat)
	byte[] getData(final String path,boolean watch,Stat stat)
	void getData(final String path,Watcher watcher,DataCallback cb,Object ctx)
	void getData(final String path,boolean watche,DataCallback cb,Object ctx)

![](https://i.imgur.com/eSUmues.png)

#### （4）更新数据 ####

客户端可以通过Zookeeper的API来更新一个节点的数据内容，有如下接口：

	Stat setData(final String path,byte data[],int version)
	void setData(final String path,byte data[],int version,StatCallback cb,Object ctx)

![](https://i.imgur.com/xaJzasw.png)

#### （5）检测节点是否存在 ####

客户端可以通过Zookeeper的API来检测指定节点是否存在，有如下接口：

	public Stat exists(final String path,Watcher watcher)
	public Stat exists(final String path,boolean watch)
	public void exists(final String path,Watcher watcher,StatCallback cb,Object ctx)
	public void exists(final String path,boolean watch,StatCallback cb,Object ctx)

![](https://i.imgur.com/ZQm1zKm.png)

需要注意：



- 无论指定节点是否存在，通过调用exists接口都可以注册Watcher。
- exists接口中注册的Watcher,能够对节点创建、节点删除和节点数据更新事件进行监听。
- 对于指定节点的子节点的各种变化，都不会通知客户端。

#### （6）权限控制 ####

开发人员如果要使用Zookeeper的权限控制功能，需要在完成Zookeeper会话创建后，给该会话添加相关的权限信息（AuthInfo）。Zookeeper客户端提供了相应的API接口来进行权限信息的设置。

	addAuthInfo(String name,byte[] auth)

该接口主要用于为当前Zookeeper会话添加权限信息，之后凡是通过该会话对Zookeeper服务端进行的任何操作，都会带上该权限信息。




### 四、开源客户端 ###

（一）、ZkClient

ZkClient是Github上一个开源的Zookeeper客户端，ZkClient是在Zookeeper原生API接口之上进行了包装，是一个更易用的Zookeeper客户端。同时，zkClient在内部实现了诸如Session超时重连、Watcher反复注册等功能，使得Zookeeper客户端的这些繁琐的细节工作对开发人员透明。

  		<dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>${zkclient.version}</version>
        </dependency>



ZKclient源码请参考： [zkclient](https://github.com/sgroschupf/zkclient)

ZKclient API请参考： [ZkClient API](http://helix.apache.org/apidocs/reference/org/apache/helix/manager/zk/ZkClient.html)

（二）、Curator

Curator是Netflix公司开源的一套Zookeeper客户端框架，现已成为Apache的顶级项目，是全世界范围内使用最广泛的Zookeeper客户端之一———— Guava is to Java What Curator is to Zookeeper。

<!-- https://mvnrepository.com/artifact/org.apache.curator/apache-curator -->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>2.4.2</version>
        </dependency>

- Curator解决了很多Zookeeper客户端非常底层的细节开发工作，包括连接重连，反复注册Watcher和NodeExistsException异常等。
- Curator还在Zookeeper原生API的基础上进行了包装，提供了一套易用性和可读性更强的Fluent风格的客户端API框架。
- Curator中还提供了Zookeeper各种应用场景（Recipe，如时间监听、Master选举、分布式锁、分布式计数器、分布式Barrier等）的抽象封装。这些使用参考都在recipes包中，读者需要单独依赖以下Maven依赖来获取:

		<dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>2.4.2</version>
        </dependency>



- Curator也提供了很多工具类，其中用的最多的就是ZKPaths和EnsurePath和TestingServer —— ZKPaths提供了一些简单的API来构建ZNode路径、递归创建和删除节点等；EnsurePath提供了一种能够确保数据节点存在的机制，多用于这样的业务：上层业务希望对一个数据节点进行一些操作，但是操作之前需要确保该节点存在；TestingServer允许开放人员非常方便地启动一个标准的Zookeeper服务器，并以此来进行一系列的单元测试（便于开发人员进行Zookeeper的开发与测试）。TestingServer在Curator的test包中，读者需要单独依赖一下Maven依赖来获取：

		<dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-test</artifactId>
            <version>2.4.2</version>
        </dependency>


Curator源码请参考：[curator](https://github.com/jxjjzm/curator)

Curator API请参考：[curator API](http://curator.apache.org/apidocs/index.html)


总结：不管是ZKclient，还是Curator，由于底层实现还是对Zookeeper原生API的包装，因此本节中不会太过详细地进行阐述。具体使用个人推荐一篇博文，请参考：[Zookeeper使用--开源客户端](http://www.cnblogs.com/leesf456/p/6032716.html)
	  


### 五、Zookeeper的典型应用场景 ###

Zookeeper是一个典型的发布/订阅模式的分布式数据管理与协调框架，开发人员可以使用它来进行分布式数据的发布与订阅。另一方面，通过对Zookeeper中丰富的数据节点类型进行交叉使用，配合Watcher事件通知机制，可以非常方便地构建一系列分布式应用中都会涉及的核心功能，如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等。

#### （一）数据发布/订阅 ####

数据发布/订阅（Publish/Subscribe）系统，即所谓的配置中心，顾名思义就是发布者将数据发布到Zookeeper的一个或一系列节点上，供订阅者进行数据订阅，进而达到动态获取数据的目的，实现配置信息的集中式管理和数据的动态更新。

发布/订阅系统一般有两种设计模式，分别是推（Push）模式和拉（Pull）模式。在推模式中，服务端主动将数据更新发送给所有订阅的客户端；而拉模式则是由客户端主动发起请求来获取最新数据，通常客户端都采用定时进行轮询拉取的方式。***Zookeeper采用的是推拉相结合的方式:客户端向服务端注册自己需要关注的节点，一旦该节点的数据发生变更，那么服务端就会向相应的客户端发送Watcher事件通知，客户端接收到这个消息通知之后，需要主动到服务端获取最新的数据。***

在我们平常的应用系统开发中，经常会碰到这样的需求：系统中需要使用一些通用的配置信息，例如机器列表信息、运行时的开关配置、数据库配置信息等。这些全局配置信息通常具备以下3个特性。

- 数据量通常比较小。
- 数据内容在运行时会发生动态变化。
- 集群中各机器共享，配置一致。

对于这类配置信息，一般的做法通常可以选择将其存储在本地配置文件或是内存变量中。无论采用哪种方式，其实都可以简单地实现配置管理。如果采取本地配置文件的方式，那么通常系统可以在应用启动的时候读取到本地磁盘的一个文件来进行初始化，并且在运行过程中定时地进行文件的读取，以此来检测文件内容的变更。另外一种借助内存变量来实现配置管理的方式也非常简单，以Java系统为例，通常可以采用JMX方式来实现对系统运行时内存变量的更新。在集群机器规模不大、配置变更不是特别频繁的情况下，无论上面提到的哪种方式，都能够非常方便地解决配置管理的问题。但是，一旦机器规模变大且配置信息变更越来越频繁后，我们发现依靠现有的这两种方式解决配置管理就变得越来越困难了。我们既希望能够快速地做到全局配置信息的变更，同时希望变更成本足够小，因此我们必须寻求一种更为分布式化的解决方案。Zookeeper可以用来实现类似这种场景的配置管理 ———— ***如果将配置信息存放到Zookeeper上进行集中管理，那么通常情况下，应用在启动的时候都会主动到Zookeeper服务端上进行一次配置信息的获取，同时，在指定节点上注册一个Watcher监听，这样一来，但凡配置信息发生变更，服务端都会实时通知到所有订阅的客户端，从而达到实时获取最新配置信息的目的。***

#### （二）负载均衡 ####

负载均衡是一种相当常见的计算机网络技术，用来对多个计算机、网络连接、CPU、磁盘驱动或其他资源进行分配负载，以达到优化资源使用、最大化吞吐率、最小化响应时间和避免过载的目的。

例如：使用Zookeeper实现动态DNS服务

- 域名配置，首先在Zookeeper上创建一个节点来进行域名配置，如DDNS/app1/server.app1.company1.com。
- 域名解析，应用首先从域名节点中获取IP地址和端口的配置，进行自行解析。同时，应用程序还会在域名节点上注册一个数据变更Watcher监听，以便及时收到域名变更的通知。
- 域名变更，若发生IP或端口号变更，此时需要进行域名变更操作，此时，只需要对指定的域名节点进行更新操作，Zookeeper就会向订阅的客户端发送这个事件通知，客户端之后就再次进行域名配置的获取。


#### （三）命名服务 ####

命名服务（Name Service）也是分布式系统中比较常见的一类场景，在分布式系统中，被命名的实体通常可以是集群中的机器、提供的服务地址或远程对象等 ———— 这些我们都可以统称它们为名字（Name），其中较为常见的就是一些分布式服务框架（如RPC、RMI）中的服务地址列表，通过使用命名服务，客户端应用能够根据指定名字来获取资源的实体、服务地址和提供者的信息等。

Zookeeper可以实现一套分布式全局唯一ID（所谓ID就是一个能够唯一标识某个对象的标识符）的分配机制。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161111191903983-1360060273.png)

通过调用Zookeeper节点创建的API接口可以创建一个顺序节点，并且在API返回值中会返回这个节点的完整名字，利用这个特性，我们就可以借助Zookeeper来生成全局唯一的ID了，其步骤如下：



- 1）. 客户端根据任务类型，在指定类型的任务下通过调用接口创建一个顺序节点，如"job-"。
- 2）. 创建完成后，会返回一个完整的节点名，如"job-00000001"。
- 3）. 客户端拿到这个返回值后，拼接上type类型和返回值后，就可以作为全局唯一ID了，如"type2-job-00000001"。

#### （四）分布式协调/通知 ####

分布式协调/通知服务是分布式系统中不可缺少的一个环节，是将不同的分布式组件有机结合起来的关键所在。对于一个在多台机器上部署运行的应用而言，通常需要一个协调者（Coordinator）来控制整个系统的运行流程，例如分布式事务的处理、机器间的互相协调等。同时，引入这样一个协调者，便于将分布式协调的职责从应用中分离出来，从而可以大大减少系统之间的耦合性，而且能够显著提高系统的可扩展性。

Zookeeper中特有的Watcher注册于异步通知机制，能够很好地实现分布式环境下不同机器，甚至不同系统之间的协调与通知，从而实现对数据变更的实时处理。通常的做法是不同的客户端都对Zookeeper上的同一个数据节点进行Watcher注册，监听数据节点的变化（包括节点本身和子节点），若数据节点发生变化，那么所有订阅的客户端都能够接收到相应的Watcher通知，并作出相应处理。

在绝大多数分布式系统中，系统机器间的通信无外乎***心跳检测、工作进度汇报和系统调度***。

- ① 心跳检测 ———— 机器间的心跳检测机制是指在分布式环境中，不同机器间需要检测到彼此是否在正常运行。在传统的开发中我们通常是通过主机之间是否可以相互PING通来判断，更复杂一点的话，则会通过在机器之间建立长连接，通过TCP连接固有的心跳检测机制来实现上层机器的心跳检测，这些确实都是一些非常常见的心跳检测方法。Zookeeper也可以用来实现机器间的心跳检测，基于Zookeeper临时节点特性（临时节点的生存周期是客户端会话，客户端若当即后，其临时节点自然不再存在），可以让不同机器都在Zookeeper的一个指定节点下创建临时子节点，不同的机器之间可以根据这个临时子节点来判断对应的客户端机器是否存活。通过这种方式检测系统和被检测系统之间并不需要直接相关联，而是通过Zookeeper上的某个节点进行关联，大大减少了系统耦合。

- ② 工作进度汇报 ———— 在一个常见的任务分发系统中，通常任务被分发到不同机器上执行后，需要实时地将自己的任务执行进度汇报给分发系统。这个时候就可以通过Zookeeper来实现，在Zookeeper上选择一个节点，每个任务客户端都在这个节点下面创建临时子节点，这样不仅可以判断机器是否存活，同时各个任务机器可以实时地将自己的任务执行进度写到该临时节点中去，以便中心系统能够实时获取任务的执行进度。

- ③ 系统调度 ———— Zookeeper能够实现如下系统调度模式：一个分布式系统由控制台和一些客户端系统两部分组成，控制台的职责就是需要将一些指令信息发送给所有的客户端，以控制他们进行相应的业务逻辑，后台管理人员在控制台上做的一些操作，实际上就是修改了Zookeeper上某些节点的数据，而Zookeeper进一步把这些数据变更以事件通知的形式发送给了对应的订阅客户端。

总之，使用Zookeeper来实现分布式系统机器间的通信，不仅能省去大量底层网络通信和协议设计上重复的工作，更为重要的一点是大大降低了系统之间的耦合，能够非常方便地实现异构系统之间的灵活通信。


#### （五）集群管理 ####

随着分布式系统规模的日益扩大，集群中的机器规模也随之变大，因此，如何更好地进行集群管理也显得越来越重要了。

所谓集群管理，包括集群监控与集群控制两大块，前者侧重对集群运行时状态的收集，后者则是对集群进行操作与控制。在日常开发和运维过程中，我们经常会有类似于如下的需求。

- 希望知道当前集群中究竟有多少机器在工作。
- 对集群中每台机器的运行时状态进行数据收集。
- 对集群中机器进行上下线操作。

在传统的基于Agent的分布式集群管理体系中，都是通过在集群中的每台机器上部署一个Agent，由这个Agent负责主动向指定的一个监控中心系统汇报自己所在机器的状态。在集群规模适中的场景下，这确实是一种在生产实践中广泛使用的解决方案，能够快速有效地实现分布式环境集群监控，但是一旦系统的业务场景增多，集群规模变大之后，该解决方案的弊端也就显现出来了：


- 大规模升级困难
- 统一的Agent无法满足多样的需求
- 编程语言多样性

Zookeeper具有以下两大特性：


- 客户端如果对Zookeeper的一个数据节点注册Watcher监听，那么当该数据节点的内容或是其子节点列表发生变更时，Zookeeper服务器就会向订阅的客户端发送变更通知。
- 对在Zookeeper上创建的临时节点，一旦客户端与服务器之间的会话失效，那么临时节点也会被自动删除。

利用Zookeeper的这两大特性，就可以实现另一种集群机器存活性监控的系统。若监控系统在/clusterServers节点上注册一个Watcher监听，那么但凡进行动态添加机器的操作，就会在/clusterServers节点下创建一个临时节点：/clusterServers/[Hostname]，这样，监控系统就能够实时监测机器的变动情况。







#### （六）Master选举 ####

Master选举是一个在分布式系统中非常常见的应用场景。分布式最核心的特性就是能够将具有独立计算能力的系统单元部署在不同的机器上，构成一个完整的分布式系统。而与此同时，实际场景中往往也需要在这些分布在不同机器上的独立系统单元中选出一个所谓的“老大”，在计算机科学中，我们称之为Master。

在分布式系统中，Master往往用来协调集群中其他系统单元，具有对分布式系统状态变更的决定权，如在一些读写分离的应用场景中，客户端的写请求往往是由Master来处理；而在另一些场景中，Master则常常负责处理一些复杂的逻辑并将处理结果同步给集群中其他系统单元。Master选举可以说是Zookeeper最典型的应用场景。利用Zookeeper的强一致性，能够很好地保证在分布式高并发情况下节点的创建一定能够保证全局唯一性，即Zookeeper将会保证客户端无法重复创建一个已经存在的数据节点。也就是说，如果同时有多个客户端请求创建同一个节点，那么最终一定只有一个客户端请求能够创建成功。利用这个特性，就能很容易地在分布式环境中进行Master选举了。（如果仅仅只是想实现Master选举的话，那么其实只需要有一个能够保证数据唯一性的组件即可，例如关系型数据库的主键模型就是非常不错的选择（依靠关系型数据库的主键特性能够很好地保证在集群中选举出唯一的一个Master，但是需要考虑另一个问题是，如果当前选举出的Master挂了，那么该如何处理？）。但是，如果希望能够快速地进行集群Master动态选举，那么基于Zookeeper来实现是一个不错的新思路。）



#### （七）分布式锁 ####

分布式锁是控制分布式系统之间同步访问共享资源的一种方式，可以保证不同的系统或是同一个系统的不同主机之间共享了一个或一组资源时保证一致性，主要分为排他锁和共享锁。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161112100514577-471030324.png)

#### 1.排他锁 ####

排他锁（Exclusive Locks，简称X锁）。又称为写锁或独占锁，是一种基本的锁类型。如果事务T1对数据对象O1加上了排他锁，那么在整个加锁期间，只允许事务T1对O1进行读取和更新操作，其他任何事务都不能再对这个数据对象进行任何类型的操作 —— 直到T1释放了排他锁。（排他锁的核心是如何保证当前有且仅有一个事务获得锁，并且锁被释放后，所有正在等待获取锁的事务都能够被通知到。）

![](https://i.imgur.com/WRgFplM.png)




- 获取锁 ———— 在需要获取排它锁时，所有的客户端都会试图通过调用create()接口，在/exclusive_lock节点下创建临时子节点/exclusive_lock/lock。Zookeeper可以保证在所有的客户端中，最终只有一个客户端能够创建成功，那么就可以认为该客户端获取了锁。同时，所有没有获取到锁的客户端就需要到/exclusive_lock节点上注册一个子节点变更的Watcher监听，以便实时监听到lock节点的变更情况。


- 释放锁 ———— 当获取锁的客户端宕机或者正常完成业务逻辑都会导致临时节点的删除，此时，所有在/exclusive_lock节点上注册监听的客户端都会收到通知，可以重新发起分布式锁获取。


#### 2.共享锁 ####

共享锁（Shared Locks，简称S锁），又称为读锁，同样是一种基本的锁类型。如果事务T1对数据对象O1加上了共享锁，那么当前事务只能对O1进行读取操作，其他事务也只能对这个数据对象加共享锁 —— 直到该数据对象上的所有共享锁都被释放。（共享锁和排他锁最根本的区别在于，加上排他锁后，数据对象只对一个事务可见，而加上共享锁后，数据对所有事务都可见。）

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161112102635827-1113002930.png)

**改进前的共享锁：**

- 获取锁 ———— 在需要获取共享锁时，所有客户端都会到/shared_lock这个节点下面创建一个临时顺序节点，如果当前是读请求，那么就创建例如/shared_lock/host-R-00000001的节点；如果是写请求，那么就创建例如/shared_lock/host-W-00000001的节点。

- 判断读写顺序 ———— 根据共享锁的定义，不同的事务都可以同时对同一个数据对象进行读取操作，而更新操作必须在当前没有任何事务进行读写操作的情况下进行。基于这个原则，我们来看看如何通过Zookeeper的节点来确定分布式读写顺序，大致可以分为如下4个步骤。

		（1）创建完节点后，获取/shared_lock节点下的所有子节点，并对该节点变注册子节点变更的Watcher监听。
		（2） 确定自己的节点序号在所有子节点中的顺序。
		（3）对于读请求：若没有比自己序号小的子节点或是所有比自己序号小的子节点都是读请求，那么表明自己已经成功获取到了共享锁，同时开始执行读取逻辑；如果比自己序号小的子节点中有写请求，那么就需要进入等待。对于写请求：若自己不是序号最小的子节点，那么就需要进入等待。
		（4）接收到Watcher通知后，重复步骤1。


- 释放锁 ———— 释放锁的逻辑和排他锁是一致的，这里不再赘述。


![](https://i.imgur.com/yaRAQug.png)


**改进后的共享锁：**

上面讲解的这个共享锁实现，大体上能够满足一般的分布式集群竞争锁的需求，并且性能都还可以 ———— 这里说的一般场景是指集群规模不是特别大，一般是在10台机器以内。

上面这个Zookeeper整个分布式共享锁的实现过程中，大量的"Watcher通知"和"子节点列表获取"两个操作重复运行，并且绝大多数的运行结果都是判断出自己并非是序号最小的节点，从而继续等待下一次通知 ———— 这个看起来显然不怎么科学。客户端无端地接收到过多和自己并不相关的事件通知，如果在集群规模比较大的情况下，不仅会对Zookeeper服务器造成巨大的性能影响和网络冲击，更为严重的是，如果同一时间有多个节点对应的客户端完成事务或是事务中断引起节点消息，Zookeeper服务器就会在短时间内向其余客户端发送大量的事件通知 ———— 这就是所谓的羊群效应。（根源在于没有找准客户端真正的关注点）

现在我们来看看如何改进上面的分布式锁实现。这里主要的改动在于：每个锁竞争者，只需要关注/shared_lock节点下序号比自己小的那个节点是否存在即可，具体实现如下：



- 客户端调用create方法创建类似于/shared_lock/[Hostname]-请求类型-序号的临时顺序节点。
-  客户端调用getChildren接口获取所有已经创建的子节点列表（注意，这里不注册任何Watcher）。
-   如果无法获取共享锁，那么就调用exist来对比自己小的那个节点注册Watcher。注意，这里“比自己小的节点”只是一个笼统的说法，具体对于读请求和写请求不一样。 ———— 对于读请求：向比自己序号小的最后一个写请求节点注册Watcher监听；对于写请求：向比自己序号小的最后一个节点注册Watcher监听。
-   等待Watcher通知，继续进入步骤2。

![](https://i.imgur.com/6ieOe5z.png)




#### （八）分布式队列 ####

业界有不少分布式队列产品，不过绝大多数都是类似于ActivityMQ、Kafka等的消息中间件（或称为消息队列）。这里我们主要介绍基于Zookeeper实现的分布式队列。分布式队列，简单地讲分为两大类，一种是常规的先入先出队列，另一种则是要等到队列元素集聚之后才统一安排执行的Barrier模型。


#### 1.FIFO：先入先出 ####

FIFO队列是一种非常典型且应用广泛的按序执行的队列模型：先进入队列的请求操作先完成后，才开始处理后面的请求。

使用Zookeeper实现FIFO队列与Zookeeper分布式共享锁的实现非常类似。FIFO队列就类似于一个全写的共享锁模型，大体的设计思路其实非常简单：所有客户端都会到/queue_fifo这个节点下面创建一个临时顺序节点，创建完节点之后，根据如下4个步骤来确定执行顺序。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161112104959530-1061396244.png)



- （1）通过调用getChildren()来获取/queue_fifo节点下的所有子节点，即获取队列中所有的元素。
- （2）确定自己的节点序号在所有子节点中的顺序。
- （3）如果自己的序号不是最小，那么需要等待，同时向比自己序号小的最后一个节点注册Watcher监听。
- （4）接收到Watcher通知后，重复步骤（1）。

![](https://i.imgur.com/xysj0qi.png)

#### 2.Barrier：分布式屏障 ####

Barrier愿意是指障碍物、屏障，而在分布式系统中，特指系统之间的一个协调条件，规定了一个队列的元素必须都集聚后才能统一进行安排，否则一直等待。这往往出现在那些大规模分布式并行计算的应用场景下：最终的合并计算需要基于很多并行计算的子结果来进行。这些队列其实是在FIFO队列的基础上进行了增强，大致的设计思想如下：开始时，/queue_barrier节点是一个已经存在的默认节点，并且将其节点的数据内容赋值为一个数字n来代表Barrier值，例如n=10表示只有当/queue_barrier节点下的子节点个数达到10后，才会打开Barrier。之后，所有的客户端都会到/queue_barrier节点下创建一个临时节点。创建完节点之后，根据如下5个步骤来确定执行顺序。 

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161112105044686-1718104359.png)

- （1）通过调用getData接口获取/queue_barrier节点的数据内容，如10。
- （2） 通过调用getChildren接口获取/queue_barrier节点下的所有子节点，同时注册对子节点变更的Watcher监听。
- （3） 统计子节点的个数。
- （4）如果子节点个数还不足10个，那么需要等待。
- （5）接受到Wacher通知后，重复步骤3。

![](https://i.imgur.com/r9l9aSz.png)
























