### Jvm之虚拟机性能监控命令与故障处理工具 ###

***
### 一、JDK命令行工具 ###

#### （一）、Jps——虚拟机进程状况工具 ####



- 格式：jps [options][hostid]
- 功能：Jps（Java Virtual Machine Process Status Tool）命令用于显示指定系统内所有的HotSpot虚拟机进程。
- 参数：
![jps](https://i.imgur.com/4chLbUB.png)

详细请参考：[详细介绍](http://www.hollischuang.com/archives/105)

#### （二）、Jstack——Java堆栈跟踪工具 ####



- 格式：jstack [option] vmid
- 功能：jstack(Stack Trace for Java)命令用于生成虚拟机当前时刻的线程快照。（一般称为threaddump或Javacore文件）。线程快照就是当前虚拟机内每一条线程正在执行的方法堆栈的集合，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如线程间死锁、死循环、请求外部资源导致的长时间等待等都是导致线程长时间停顿的常见原因。线程出现停顿的时候通过jstack来查看各个线程的条用堆栈，就可以知道没有响应的线程到底在后台做些什么事情，或者等待着什么资源。
- 参数：
![](https://i.imgur.com/VzvUpMt.png)



#### （三）、Jstat——虚拟机统计信息监视工具 ####

- 格式：jstat [ option vmid [interval[s|ms] [count]] ]
- 功能：jstat(JVM Statistics Monitoring Tool)命令用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或远程虚拟机进程中的类加载、内存、垃圾收集、JIT编译等运行数据。
- 参数：
![](https://i.imgur.com/jx1WCwB.png)




#### （四）、Jmap——Java内存映像工具 ####

- 格式：jstat [ option vmid [interval[s|ms] [count]] ]
- 功能：jmap(Memory Map for Java)命令用于生成堆转储快照
- 参数：
![](https://i.imgur.com/jx1WCwB.png)





#### （五）、Jhat——虚拟机堆转储快照分析工具 ####



#### (六)、Jinfo——Java配置信息工具 ####



#### （七）、Javap—— ####











### 二、JDK可视化工具 ###

#### （一）、Jconsole——Java监控与管理控制台 ####




#### （二）、VisualVM——多合一故障处理工具 ####




### 三、JVM问题跟踪排查 ###