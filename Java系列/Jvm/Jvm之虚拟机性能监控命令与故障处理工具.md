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
- 功能：jmap(Memory Map for Java)命令用于查看堆内存信息 、生成堆转储快照
- 参数：
![](https://i.imgur.com/WmuQOEa.png)


I、jmap -heap ${pid}

![](https://img-blog.csdn.net/20150402212015709?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamFjaW4x/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

II、jmap -histo ${pid}  或  jmap -histo ${pid} | grep ${package} 

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

![](https://mmbiz.qpic.cn/mmbiz_jpg/QCu849YTaINP886d9m1cTFWLk2lKic3wibVGA56mgY53ePXZkQsYl1N2RmNsdxxwWzZSZHummDU3cQh5iaeOt1LCA/640?tp=webp&wxfrom=5&wx_lazy=1)

#### （二）、VisualVM——多合一故障处理工具 ####
VisualVM 是一个工具，它提供了一个可视界面，用于查看 Java 虚拟机 (Java Virtual Machine, JVM) 上运行的基于 Java 技术的应用程序（Java 应用程序）的详细信息。VisualVM 对 Java Development Kit (JDK) 工具所检索的 JVM 软件相关数据进行组织，并通过一种使您可以快速查看有关多个 Java 应用程序的数据的方式提供该信息。您可以查看本地应用程序以及远程主机上运行的应用程序的相关数据。此外，还可以捕获有关 JVM 软件实例的数据，并将该数据保存到本地系统，以供后期查看或与其他用户共享。

![](https://mmbiz.qpic.cn/mmbiz_jpg/QCu849YTaINP886d9m1cTFWLk2lKic3wibeB9TqCZ2bhGWBImxz4qrNBCAZj8V1xicnwFQPNm1aI4mQebuvPg4yUA/640?tp=webp&wxfrom=5&wx_lazy=1)


### 三、第三方工具 ###


#### （一）、MAT(Memory Analyzer Tool) ####

MAT(Memory Analyzer Tool)，一个基于Eclipse的内存分析工具，是一个快速、功能丰富的Java heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。使用内存分析工具从众多的对象中进行分析，快速的计算出在内存中对象的占用大小，看看是谁阻止了垃圾收集器的回收工作，并可以通过报表直观的查看到可能造成这种结果的对象。

#### （二）、GChisto ####
GChisto是一款专业分析gc日志的工具，可以通过gc日志来分析：Minor GC、full gc的时间、频率等等，通过列表、报表、图表等不同的形式来反应gc的情况。虽然界面略显粗糙，但是功能还是不错的。



### 四、JVM性能调优 ###

程序在上线前的测试或运行中有时会出现一些大大小小的JVM问题，比如cpu load过高、请求延迟、tps降低等，甚至出现内存泄漏（每次垃圾收集使用的时间越来越长，垃圾收集频率越来越高，每次垃圾收集清理掉的垃圾数据越来越少）、内存溢出导致系统崩溃，因此需要对JVM进行调优，使得程序在正常运行的前提下，获得更高的用户体验和运行效率。JVM调优目标：使用较小的内存占用来获得较高的吞吐量或者较低的延迟。

- 内存占用：程序正常运行需要的内存大小。
- 延迟：由于垃圾收集而引起的程序停顿时间。
- 吞吐量：用户程序运行时间占用户程序和垃圾收集占用总时间的比值。



#### 1.JVM调优工具 ####

除了上面列举出的JDK命令行工具、JDK可视化工具、第三方工具等之外，调优还可以依赖、参考的数据有系统运行日志、堆栈错误信息、gc日志、线程快照、堆转储快照等。

- 系统运行日志：系统运行日志就是在程序代码中打印出的日志，描述了代码级别的系统运行轨迹（执行的方法、入参、返回值等），一般系统出现问题，系统运行日志是首先要查看的日志。
-  堆栈错误信息：当系统出现异常后，可以根据堆栈信息初步定位问题所在，比如根据“java.lang.OutOfMemoryError: Java heap space”可以判断是堆内存溢出；根据“java.lang.StackOverflowError”可以判断是栈溢出；根据“java.lang.OutOfMemoryError: PermGen space”可以判断是方法区溢出等。
-  GC日志：程序启动时用 -XX:+PrintGCDetails 和 -Xloggc:/data/jvm/gc.log 可以在程序运行时把gc的详细过程记录下来，或者直接配置“-verbose:gc”参数把gc日志打印到控制台，通过记录的gc日志可以分析每块内存区域gc的频率、时间等，从而发现问题，进行有针对性的优化。
-  线程快照：顾名思义，根据线程快照可以看到线程在某一时刻的状态，当系统中可能存在请求超时、死循环、死锁等情况是，可以根据线程快照来进一步确定问题。通过执行虚拟机自带的“jstack pid”命令，可以dump出当前进程中线程的快照信息，更详细的使用和分析网上有很多例，贴一篇博客供参考：[java命令--jstack 工具](http://www.cnblogs.com/kongzhongqijing/articles/3630264.html "java命令--jstack 工具")
-  堆转储快照：程序启动时可以使用 “-XX:+HeapDumpOnOutOfMemory” 和 “-XX:HeapDumpPath=/data/jvm/dumpfile.hprof”，当程序发生内存溢出时，把当时的内存快照以文件形式进行转储（也可以直接用jmap命令转储程序运行时任意时刻的内存快照），事后对当时的内存使用情况进行分析。




#### 2.JVM调优经验 ####

VM配置方面，一般情况可以先用默认配置（基本的一些初始参数可以保证一般的应用跑的比较稳定了），在测试中根据系统运行状况（会话并发情况、会话时间等），结合gc日志、内存监控、使用的垃圾收集器等进行合理的调整，当老年代内存过小时可能引起频繁Full GC，当内存过大时Full GC时间会特别长。

那么JVM的配置比如新生代、老年代应该配置多大最合适呢？答案是不一定，调优就是找答案的过程，物理内存一定的情况下，新生代设置越大，老年代就越小，Full GC频率就越高，但Full GC时间越短；相反新生代设置越小，老年代就越大，Full GC频率就越低，但每次Full GC消耗的时间越大。建议如下：



- -Xms和-Xmx的值设置成相等，堆大小默认为-Xms指定的大小，默认空闲堆内存小于40%时，JVM会扩大堆到-Xmx指定的大小；空闲堆内存大于70%时，JVM会减小堆到-Xms指定的大小。如果在Full GC后满足不了内存需求会动态调整，这个阶段比较耗费资源。
- 新生代尽量设置大一些，让对象在新生代多存活一段时间，每次Minor GC 都要尽可能多的收集垃圾对象，防止或延迟对象进入老年代的机会，以减少应用程序发生Full GC的频率。
- 老年代如果使用CMS收集器，新生代可以不用太大，因为CMS的并行收集速度也很快，收集过程比较耗时的并发标记和并发清除阶段都可以与用户线程并发执行。
- 方法区大小的设置，1.6之前的需要考虑系统运行时动态增加的常量、静态变量等，1.7只要差不多能装下启动时和后期动态加载的类信息就行。



代码实现方面，性能出现问题比如程序等待、内存泄漏，除了JVM配置可能存在问题，代码实现上也有很大关系：



- 避免创建过大的对象及数组：过大的对象或数组在新生代没有足够空间容纳时会直接进入老年代，如果是短命的大对象，会提前触发Full GC。
- 避免同时加载大量数据，如一次从数据库中取出大量数据，或者一次从Excel中读取大量记录，可以分批读取，用完尽快清空引用。
- 当集合中有对象的引用，这些对象使用完之后要尽快把集合中的引用清空，这些无用对象尽快回收避免进入老年代。
- 可以在合适的场景（如实现缓存）采用软引用、弱引用，比如用软引用来为ObjectA分配实例：SoftReference objectA=new SoftReference(); 在发生内存溢出前，会将objectA列入回收范围进行二次回收，如果这次回收还没有足够内存，才会抛出内存溢出的异常。 
- 避免产生死循环，产生死循环后，循环体内可能重复产生大量实例，导致内存空间被迅速占满。
- 尽量避免长时间等待外部资源（数据库、网络、设备资源等）的情况，缩小对象的生命周期，避免进入老年代，如果不能及时返回结果可以适当采用异步处理的方式等。

















