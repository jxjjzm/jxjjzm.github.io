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


详细请参考：[详细介绍](http://www.hollischuang.com/archives/110)

#### （三）、Jstat——虚拟机统计信息监视工具 ####

- 格式：jstat [ option vmid [interval[s|ms] [count]] ]
- 功能：jstat(JVM Statistics Monitoring Tool)命令用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或远程虚拟机进程中的类加载、内存、垃圾收集、JIT编译等运行数据。
- 参数：
![](https://i.imgur.com/jx1WCwB.png)

详细请参考：[详细介绍](http://www.hollischuang.com/archives/481)


#### （四）、Jmap——Java内存映像工具 ####

- 格式：jstat [ option vmid [interval[s|ms] [count]] ]
- 功能：jmap(Memory Map for Java)命令用于生成堆转储快照
- 参数：
![](https://i.imgur.com/WmuQOEa.png)

详细请参考：[详细介绍](http://www.hollischuang.com/archives/303)



#### （五）、Jhat——虚拟机堆转储快照分析工具 ####

- 格式：jhat [dumpfile]
- 功能：一般与jmap搭配使用，用来分析jmap生成的堆转储文件。（由于有很多可视化工具（Eclipse Memory Analyzer 、IBM HeapAnalyzer）可以替代，所以很少用。）



详细请参考：[详细介绍](http://www.hollischuang.com/archives/1047)

#### (六)、Jinfo——Java配置信息工具 ####


- 格式：jinfo [ option vmid
- 功能：实时查看和调整虚拟机的各项参数，可以显示虚拟机启动时未被显式指定的参数的默认值（jps -v ：查看虚拟机启动时显式指定的参数列表）------注意：jdk8中已经不支持该命令。

![](https://i.imgur.com/jx1WCwB.png)

详细请参考：[详细介绍](http://www.hollischuang.com/archives/1094)



#### （七）、Javap——反编译和查看编译器编译后的字节码工具 ####


- 功能：javap可以用于反编译和查看编译器编译后的字节码。平时一般用javap -c比较多，该命令用于列出每个方法所执行的JVM指令，并显示每个方法的字节码的实际作用。可以通过字节码和源代码的对比，深入分析java的编译原理，了解和解决各种Java原理级别的问题。

详细请参考：[详细介绍](http://www.hollischuang.com/archives/1107)








### 二、JDK可视化工具 ###

#### （一）、Jconsole——Java监控与管理控制台 ####
Jconsole（Java Monitoring and Management Console）是从java5开始，在JDK中自带的java监控和管理控制台，用于对JVM中内存，线程和类等的监控，是一个基于JMX（java management extensions）的GUI性能监测工具。jconsole使用jvm的扩展机制获取并展示虚拟机中运行的应用程序的性能和资源消耗等信息。



#### （二）、VisualVM——多合一故障处理工具 ####
VisualVM 是一个工具，它提供了一个可视界面，用于查看 Java 虚拟机 (Java Virtual Machine, JVM) 上运行的基于 Java 技术的应用程序（Java 应用程序）的详细信息。VisualVM 对 Java Development Kit (JDK) 工具所检索的 JVM 软件相关数据进行组织，并通过一种使您可以快速查看有关多个 Java 应用程序的数据的方式提供该信息。您可以查看本地应用程序以及远程主机上运行的应用程序的相关数据。此外，还可以捕获有关 JVM 软件实例的数据，并将该数据保存到本地系统，以供后期查看或与其他用户共享。



### 二、第三方工具 ###


#### （一）、MAT(Memory Analyzer Tool) ####

MAT(Memory Analyzer Tool)，一个基于Eclipse的内存分析工具，是一个快速、功能丰富的Java heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。使用内存分析工具从众多的对象中进行分析，快速的计算出在内存中对象的占用大小，看看是谁阻止了垃圾收集器的回收工作，并可以通过报表直观的查看到可能造成这种结果的对象。

#### （二）、GChisto ####
GChisto是一款专业分析gc日志的工具，可以通过gc日志来分析：Minor GC、full gc的时间、频率等等，通过列表、报表、图表等不同的形式来反应gc的情况。虽然界面略显粗糙，但是功能还是不错的。






