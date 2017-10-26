### Zookeepe运维篇 ###
***

### 一、配置详解 ###

#### 1.基本配置 ####

所谓基本的配置参数是指这些配置参数都是Zookeeper运行时所必须的，如果不配置这些参数，将无法启动Zookeeper服务器。同时，Zookeeper也会为这些参数设置默认值。这些基本的配置参数包括clientPort、dataDir和tickTime.

![](https://i.imgur.com/TCj3enx.png)


#### 2.高级配置 ####

![](https://i.imgur.com/frQL7Fk.png)
![](https://i.imgur.com/HwPt7lR.jpg)
![](https://i.imgur.com/u5ra33Z.jpg)
![](https://i.imgur.com/pjxCtJs.jpg)
![](https://i.imgur.com/d8p6LAS.png)


### 二、四字命令 ###



- telnet ip port —— 使用telnet客户端登录Zookeeper的对外服务端口；
- nc ip port —— 模拟http heads
- conf —— 用于输出Zookeeper服务器运行时使用的基本配置信息；
- cons —— 用于输出当前这台服务器上所有客户端连接的详细信息；
- crst —— 用于重置所有的客户端连接信息；
- dump —— 用于输出当前集群的所有会话信息；
- envi —— 用于输出Zookeeper所在服务器运行时的环境信息；
- ruok —— 用于输出当前Zookeeper服务器是否正在运行；
- stat —— 用于获取Zookeeper服务器的运行时状态信息；
- srvr —— 和stat命令功能一致，唯一的区别是srvr不会将客户端的连接情况输出，仅仅输出服务器的自身信息；
- srst —— 用于重置所有服务器的统计信息；
- wchs —— 用于输出当前服务器管理的Watcher的概要信息；
- wchc —— 用与输出当前服务器上管理的Watcher的详细信息，以会话为单位进行归组，同时列出被该会话注册了Watcher的节点路径；
- wchp —— 用于输出当前服务器上管理的Watcher的详细信息，不同点在于wchp命令的输出信息以节点路径为单位进行归组；
- mntr —— 用于输出比stat命令更为详尽的服务器统计信息；


### 三、JMX ###


从Apache官方网站上下载的Zookeeper，默认开启了JMX功能，但是却只限本地连接，无法通过远程连接，读者可以打开%ZK_HOME%bin目录下的zkServer.sh 文件，在这个配置中并没有开启远程连接JMX的端口信息，通常需要加入以下三个配置才能开启远程JMX：

-Dcom.sun.management.jmxremote.port=5000
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false



### 四、监控 ###


#### 1.实时监控 ####


#### 2.数据统计 ####




### 五、构建一个高可用的集群 ###

#### 1.集群组成 ####


#### 2.容灾 ####



#### 3.扩容与缩容 ####





### 六、日常运维 ###

#### 1.数据与日志管理 ####



#### 2.Too many connections ####



#### 3.磁盘管理 ####




















