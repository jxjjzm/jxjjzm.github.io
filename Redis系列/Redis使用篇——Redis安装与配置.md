### Redis使用篇——Redis安装与配置 ###
***

### 一、单机环境搭建（Linux） ###

#### （一)、下载Redis源码安装包 ####


1.**下载源码安装包** 

		 wget http://download.redis.io/releases/redis-3.0.0.tar.gz

#### （二)、解压缩源码并进入目录 ####

1.**解压源码安装包** 

	tar -zxvf redis-3.0.0.tar.gz

2.**进入安装目录**

	 cd redis-3.0.0

#### （三)、编译源码（Linus系统软件安装三部曲：configure - make - make install） ####

**1.make  (如果是32位机器 make 32bit)**

	make

make命令执行完成后，会在当前目录下生成本个可执行文件，分别是redis-server、redis-cli、redis-benchmark、redis-stat，它们的作用如下：

![](https://i.imgur.com/iZslW8D.png)


	

-  redis-server：Redis服务器的daemon启动程序
-  redis-cli：Redis命令行操作工具。当然，你也可以用telnet根据其纯文本协议来操作
-  redis-benchmark：Redis性能测试工具，测试Redis在你的系统及你的配置下的读写性能
-  redis-stat：Redis状态检测工具，可以检测Redis当前状态参数及延迟状况 
-  redis-check-aof:  日志文件检测工(比如断电造成日志损坏,可以检测并修复)
-  redis-check-dump: 快照文件检测工具,效果类上


备注：（容易碰到的常见问题）

(1)

	问题：时间错误
	原因: 源码是官方configure过的,但官方configure时,生成的文件有时间戳信息,Make只能发生在configure之后,如果你的虚拟机的时间不对,比如说是2012年
	解决: date -s ‘yyyy-mm-dd hh:mm:ss’   重写时间，再 clock -w  写入cmos


(2)

	问题：编译环境问题
	原因： GCC编译环境未安装
	解决：先安装Gcc 编译器 （yum install gcc）



#### （四)、测试编译情况（可选步骤） ####

	make test

备注：（容易碰到的常见问题）

（1）

	问题： ttl问题
	原因： need tcl > 8.4
	解决： yum install tcl


#### （五)、修改配置文件 ####

在我们成功安装Redis后，我们直接执行redis-server即可运行Redis，此时它是按照默认配置来运行的（默认配置甚至不是后台运行）。我们希望Redis按我们的要求运行，则我们需要修改配置文件。下面是redis.conf的主要配置参数的意义：


	

-  daemonize：是否以后台daemon方式运行（yes: 后台方式运行，默认为no ）
-  pidfile：pid文件位置
-  port：监听的端口号(默认6379)
-  timeout：请求超时时间
-  loglevel：log信息级别
-  logfile：log文件位置
-  databases：开启数据库的数量
-  save * * ：保存快照的频率，第一个* 表示多长时间，第三个* 表示执行多少次写操作。在一定时间内执行一定数量的写操作时，自动保存快照。可设置多个条件。
-  rdbcompression：是否使用压缩
-  dbfilename：数据快照文件名（只是文件名，不包括目录）
-  dir：数据快照的保存目录（这个是目录）
-  appendonly：是否开启appendonlylog，开启的话每次写操作会记一条log，这会提高数据抗风险能力，但影响效率。
-  appendfsync：appendonlylog如何同步到磁盘（三个选项，分别是每次写都强制调用fsync、每秒启用一次fsync、不调用fsync等待系统自己同步）


#### （六)、服务启停 ####

**1、脚本方式**

(1)启动服务器

	redis-server                            #默认找redis.conf配置文件
	redis-server redis6380.conf             #指定配置文件，这样可以启动多个实例


（2）启动客户端
	
I、默认连接：IP 127.0.0.1 端口 6379

	redis-cli

II、指定IP端口：

 	redis-cli –h 127.0.0.1 –p 6379


备注——查看是否成功启动：

	 ps -ef | grep redis  或 ./redis-cli ping

（3）退出客户端

I、默认关闭：

	redis-cli shutdown

II、指定IP端口关闭：
       
	redis-cli  –p 6379 shutdown


**2、telnet方式**



 （1）.启动服务器

	redis-server                            #默认找redis.conf配置文件
	redis-server redis6380.conf             #指定配置文件，这样可以启动多个实例
     
（2）.启动客户端

	telnet 127.0.0.1 6379                            
     
（3）.退出客户端

	quit



### 二、集群环境搭建 ###

#### （一）、安装redis-cluster依赖（redis集群管理工具redis-trid.rb依赖ruby环境，首先需要安装ruby环境）. ####

(1)确保系统安装zlib,否则gem install会报(no such file to load -- zlib)
 


	1. #download:zlib-1.2.6.tar  
	2. ./configure  
	3. make  
	4. make install  

  
 
 (2)安装ruby:version(1.9.2)
 
方式一：源码包安装


	1. # ruby1.9.2   
	2. cd /path/ruby  
	3. ./configure -prefix=/usr/local/ruby  
	4. make  
	5. make install  
	6. sudo cp ruby /usr/local/bin  

 方式二：yum安装

	yum install ruby 


(3)安装rubygem:version(1.8.16)
 
方式一：源码包安装
  

	1. # rubygems-1.8.16.tgz  
	2. cd /path/gem  
	3. sudo ruby setup.rb  
	4. sudo cp bin/gem /usr/local/bin  

方式二：yum安装

	yum install rubygems 

(4)安装ruby和redis的接口程序：gem-redis:version(3.0.0)
 

	1. gem install redis --version 3.0.0  
	2. #由于源的原因，可能下载失败，就手动下载下来安装  
	3. #download地址:http://rubygems.org/gems/redis/versions/3.0.0  
	4. gem install -l /data/soft/redis-3.0.0.gem  

 
#### （二）、redis-cluster安装准备 ####

 

	1. cd /path/redis  
	2. make  
	3. sudo cp /opt/redis/src/redis-server /usr/local/bin  
	4. sudo cp /opt/redis/src/redis-cli /usr/local/bin  
	5. sudo cp /opt/redis/src/redis-trib.rb /usr/local/bin  

####  （三）、redis cluster配置 ####

(1)redis配置文件结构

![](https://i.imgur.com/QFJZ07n.jpg)

使用包含(include)把通用配置和特殊配置分离,方便维护.

（2）redis通用配置


	1. #GENERAL  
	2. daemonize no  
	3. tcp-backlog 511  
	4. timeout 0            #当客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
	5. tcp-keepalive 0  
	6. loglevel notice      #指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning,默认为verbose
	7. databases 16         #设置数据库的数量，默认数据库为0，可以使用 SELECT <dbid> 命令在连接上指定数据库id 
	8. dir /opt/redis/data  
	9. slave-serve-stale-data yes  
	10. #slave只读  
	11. slave-read-only yes  
	12. #not use default  
	13. repl-disable-tcp-nodelay yes  
	14. slave-priority 100  
	15. #打开aof持久化  
	16. appendonly yes  
	17. #每秒一次aof写  
	18. appendfsync everysec  
	19. #关闭在aof rewrite的时候对新的写操作进行fsync  
	20. no-appendfsync-on-rewrite yes  
	21. auto-aof-rewrite-min-size 64mb  
	22. lua-time-limit 5000  
	23. #打开redis集群  
	24. cluster-enabled yes  
	25. #节点互连超时的阀值  
	26. cluster-node-timeout 15000  
	27. cluster-migration-barrier 1  
	28. slowlog-log-slower-than 10000  
	29. slowlog-max-len 128  
	30. notify-keyspace-events ""  
	31. hash-max-ziplist-entries 512  
	32. hash-max-ziplist-value 64  
	33. list-max-ziplist-entries 512  
	34. list-max-ziplist-value 64  
	35. set-max-intset-entries 512  
	36. zset-max-ziplist-entries 128  
	37. zset-max-ziplist-value 64  
	38. activerehashing yes  
	39. client-output-buffer-limit normal 0 0 0  
	40. client-output-buffer-limit slave 256mb 64mb 60  
	41. client-output-buffer-limit pubsub 32mb 8mb 60  
	42. hz 10  
	43. aof-rewrite-incremental-fsync yes  

 
(3)redis特殊配置.
 

	1. #包含通用配置  
	2. include /opt/redis/redis-common.conf  
	3. #指定Redis监听tcp端口,默认端口为6379  
	4. port 6379  
	5. #最大可用内存  
	6. maxmemory 100m  
	7. #内存耗尽时采用的淘汰策略:  
	8. # volatile-lru -> remove the key with an expire set using an LRU algorithm  
	9. # allkeys-lru -> remove any key accordingly to the LRU algorithm  
	10. # volatile-random -> remove a random key with an expire set  
	11. # allkeys-random -> remove a random key, any key  
	12. # volatile-ttl -> remove the key with the nearest expire time (minor TTL)  
	13. # noeviction -> don't expire at all, just return an error on write operations  
	14. maxmemory-policy allkeys-lru  
	15. #aof存储文件  
	16. appendfilename "appendonly-6379.aof"  
	17. #rdb文件,只用于动态添加slave过程  
	18. dbfilename dump-6379.rdb  
	19. #cluster配置文件(启动自动生成)  
	20. cluster-config-file nodes-6379.conf  
	21. #部署在同一机器的redis实例，把<span style="font-size: 1em; line-height: 1.5;">auto-aof-rewrite搓开，防止瞬间fork所有redis进程做rewrite,占用大量内存</span>  
	22. auto-aof-rewrite-percentage 80-100  

####  （四）、redis cluster 操作 ####

**1、初始化并构建集群**

(1)：启动集群相关节点（必须是空节点）,指定配置文件和输出日志
  

	1. redis-server /opt/redis/conf/redis-6380.conf > /opt/redis/logs/redis-6380.log 2>&1 &  
	2. redis-server /opt/redis/conf/redis-6381.conf > /opt/redis/logs/redis-6381.log 2>&1 &  
	3. redis-server /opt/redis/conf/redis-6382.conf > /opt/redis/logs/redis-6382.log 2>&1 &  
	4. redis-server /opt/redis/conf/redis-7380.conf > /opt/redis/logs/redis-7380.log 2>&1 &  
	5. redis-server /opt/redis/conf/redis-7381.conf > /opt/redis/logs/redis-7381.log 2>&1 &  
	6. redis-server /opt/redis/conf/redis-7382.conf > /opt/redis/logs/redis-7382.log 2>&1 &  

 
(2):使用自带的ruby工具(redis-trib.rb)构建集群
 

	1. #redis-trib.rb的create子命令构建  
	2. #--replicas 则指定了为Redis Cluster中的每个Master节点配备几个Slave节点  
	3. #节点角色由顺序决定,先master之后是slave(为方便辨认,slave的端口比master大1000)  
	4. redis-trib.rb create --replicas 1 10.10.34.14:6380 10.10.34.14:6381 10.10.34.14:6382 10.10.34.14:7380 10.10.34.14:7381 10.10.34.14:7382  

注意：
如果执行时报如下错误：

	[ERR] Node XXXXXX is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0

解决方法是删除生成的配置文件nodes.conf，如果不行则说明现在创建的结点包括了旧集群的结点信息，需要删除redis的持久化文件后再重启redis，比如：appendonly.aof、dump.rdb 

(3):检查集群状态
 

	1. #redis-trib.rb的check子命令构建  
	2. #ip:port可以是集群的任意节点  
	3. redis-trib.rb check 1 10.10.34.14:6380  

 最后输出如下信息,没有任何警告或错误，表示集群启动成功并处于ok状态
  

	1. [OK] All nodes agree about slots configuration.  
	2. >>> Check for open slots...  
	3. >>> Check slots coverage...  
	4. [OK] All 16384 slots covered.  

 
**2、添加新master节点**

添加一个master节点 : 创建一个空节点（empty node），然后将某些slot移动到这个空节点上,这个过程目前需要人工干预。

(a):创建node.conf(此处为6386.conf，下同) 配置文件（和其他节点创建方式一样，只要把端口号改一下即可）

(b):启动节点
 

	1. nohup redis-server /opt/redis/conf/redis-6386.conf > /opt/redis/logs/redis-6386.log 2>&1 &  

 
(c):加入空节点到集群

add-node  将一个节点添加到集群里面， 第一个是新节点ip:port, 第二个是任意一个已存在节点ip:port
 

	1. redis-trib.rb add-node 10.10.34.14:6386 10.10.34.14:6381  

新节点现在已经连接上了集群， 成为集群的一份子， 并且可以对客户端的命令请求进行转向了， 但是和其他主节点相比， 新节点还有两点区别：

	

-  新节点没有包含任何数据， 因为它没有包含任何哈希槽.
-  尽管新节点没有包含任何哈希槽， 但它仍然是一个主节点， 所以在集群需要将某个从节点升级为新的主节点时， 这个新节点不会被选中。

接下来， 只要使用 redis-trib 程序， 将集群中的某些哈希桶移动到新节点里面， 新节点就会成为真正的主节点了。 

(d):为新节点分配slot
 
 

	1. redis-trib.rb reshard 10.10.34.14:6386  
	2. #根据提示选择要迁移的slot数量(ps:这里选择500)  
	3. How many slots do you want to move (from 1 to 16384)? 500  
	4. #选择要接受这些slot的node-id  
	5. What is the receiving node ID? f51e26b5d5ff74f85341f06f28f125b7254e61bf  
	6. #选择slot来源:  
	7. #all表示从所有的master重新分配，  
	8. #或者数据要提取slot的master节点id,最后用done结束  
	9. Please enter all the source node IDs.  
	10.   Type 'all' to use all the nodes as source nodes for the hash slots.  
	11.   Type 'done' once you entered all the source nodes IDs.  
	12. Source node #1:all  
	13. #打印被移动的slot后，输入yes开始移动slot以及对应的数据.  
	14. #Do you want to proceed with the proposed reshard plan (yes/no)? yes  
	15. #结束  

**3、添加新的slave节点**

(a):前三步操作同添加master一样

(b)第四步:redis-cli连接上新节点shell,输入命令:cluster replicate 对应master的node-id
 

	1. cluster replicate 2b9ebcbd627ff0fd7a7bbcc5332fb09e72788835  

 
note:在线添加slave 时，需要dump整个master进程，并传递到slave，再由 slave加载rdb文件到内存，rdb传输过程中Master可能无法提供服务,整个过程消耗大量io,小心操作.
例如本次添加slave操作产生的rdb文件
  

	1. -rw-r--r-- 1 root root  34946 Apr 17 18:23 dump-6386.rdb  
	2. -rw-r--r-- 1 root root  34946 Apr 17 18:23 dump-7386.rdb  

 
**4、在线reshard 数据**

**5、删除一个slave节点**
 

	1. #redis-trib del-node ip:port '<node-id>'  
	2. redis-trib.rb del-node 10.10.34.14:7386 'c7ee2fca17cb79fe3c9822ced1d4f6c5e169e378'  

**6、删除一个master节点**
 
（a）（移除主节点前，需要确保这个主节点是空的）删除master节点之前首先要使用reshard移除master的全部slot,然后再删除当前节点(目前只能把被删除master的slot迁移到一个节点上)
   

	1. #把10.10.34.14:6386当前master迁移到10.10.34.14:6380上  
	2. redis-trib.rb reshard 10.10.34.14:6380  
	3. #根据提示选择要迁移的slot数量(ps:这里选择500)  
	4. How many slots do you want to move (from 1 to 16384)? 500(被删除master的所有slot数量)  
	5. #选择要接受这些slot的node-id(10.10.34.14:6380)  
	6. What is the receiving node ID? c4a31c852f81686f6ed8bcd6d1b13accdc947fd2 (ps:10.10.34.14:6380的node-id)  
	7. Please enter all the source node IDs.  
	8.   Type 'all' to use all the nodes as source nodes for the hash slots.  
	9.   Type 'done' once you entered all the source nodes IDs.  
	10. Source node #1:f51e26b5d5ff74f85341f06f28f125b7254e61bf(被删除master的node-id)  
	11. Source node #2:done  
	12. #打印被移动的slot后，输入yes开始移动slot以及对应的数据.  
	13. #Do you want to proceed with the proposed reshard plan (yes/no)? yes  

 
(b):删除空master节点
 

	1. redis-trib.rb del-node 10.10.34.14:6386 'f51e26b5d5ff74f85341f06f28f125b7254e61  bf'  


**7、从节点的迁移**

在特定的场景下，不需要系统管理员的协助下，自动将一个从节点从当前的主节点切换到另一个主节 的自动重新配置的过程叫做复制迁移（从节点迁移），从节点的迁移能够提高整个Redis集群的可用性.




































