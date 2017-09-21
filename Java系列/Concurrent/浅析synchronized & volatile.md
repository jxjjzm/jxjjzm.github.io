### 浅析synchronized & volatile ###
***
### 一、synchronized ###

Synchronized的作用主要有三点：

- 互斥性：确保线程互斥的访问同步代码
- 可见性：保证共享变量的修改能够及时可见
- 重排序：有效解决重排序问题。（Synchronized同步中的代码JVM不会轻易优化重排序）

Synchronized的具体用法这里就不做讲解了，我们来分析下Synchronized的实现原理：

#### 1.synchronized修饰代码块 ####

	public class SynchronizedDemo {
	    public void method (){
	        synchronized (this) {
	            System.out.println("method 1 start!!!!");
	        }
	    }
	}

javac -encoding utf-8 SynchronizedDemo.java 编译生成class 后，javap -c 反编译一下，看指令：

![](http://images2015.cnblogs.com/blog/584866/201704/584866-20170405112214082-1974483511.png)

**同步代码块：**同步代码块是使用monitorenter和monitorexit指令实现的。对于synchronized同步块当Java源代码被javac编译成bytecode的时候，会在同步块的入口位置和退出位置分别插入monitorenter和monitorexit字节码指令。 JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个 monitor 与之关联，当且一个monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。

（1）monitorenter监视器准入指令

每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

- 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
- 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
- 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

（2）monitorexit监视器释放指令

- 执行monitorexit的线程必须是objectref所对应的monitor的所有者。
- 指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权


#### 2.synchronized修饰方法 ####


	package lock;
	
	public class SynchronizedDemo0 {
	    public synchronized void method (){
	        System.out.println("method start!!!!");
	    }
	}

javap -v 查看class文件，发现method上加了同步标签，本质上还是monitor

![](http://images2015.cnblogs.com/blog/584866/201704/584866-20170405183523503-2138017328.png)


synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象。


**同步方法：**同步方法是使用修饰符ACC_SYNCHRONIZED实现的。synchronized方法会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示class做为锁对象。

#### 总结 ####



- JVM规范规定JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现的，而方法同步是使用修饰符ACC_SYNCHRONIZED实现的。



### 二、volatile ###

#### 1.volatile 特性 ####

Java 语言中的 Volatile 变量可以被看作是一种 “程度较轻的 synchronized”；与 synchronized 块相比，volatile 变量所需的编码较少，并且运行时开销也较少，但是它所能实现的功能也仅是 synchronized 的一部分。Volatile 变量具有 synchronized 的可见性特性，但是不具备原子特性。

![](https://i.imgur.com/5vs94EK.png)

一旦一个共享变量（类的成员变量、类的静态成员变量）被volatile修饰之后，那么就具备了两层语义：

- 可见性：保证了不同线程对这个变量进行操作时的可见性，即一个线程修改了某个变量的值，这新值对其他线程来说是立即可见的。
- 重排序：禁止进行指令重排序。
               
说明：a）当程序执行到volatile变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；b）在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

因此，您只能在有限的一些情形下使用 volatile 变量替代锁。要使 volatile 变量提供理想的线程安全，必须同时满足下面两个条件：
  

- 对变量的写操作不依赖于当前值。
- 该变量没有包含在具有其他变量的不变式中。

实际上，这些条件表明，可以被写入 volatile 变量的这些有效值独立于任何程序的状态，包括变量的当前状态。


#### 2.volatile 实现原理分析  ####

![](https://i.imgur.com/0ncF9Wx.png)

**（1）可见性**

**可见性：**处理器为了提高处理速度，不直接和内存进行通讯，而是先将系统内存的数据读到内部缓存（L1,L2或其他）后再进行操作，但操作完之后不知道何时会写到内存，如果对声明了Volatile变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器要对这个数据进行修改操作的时候，会强制重新从系统内存里把数据读到处理器缓存里。

lock前缀指令在多核处理器下会引发了两件事情：

- 将当前处理器缓存行的数据会写回到系统内存。
- 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。

这两件事情在IA-32软件开发者架构手册的第三册的多处理器管理章节（第八章）中有详细阐述。



- **Lock前缀指令会引起处理器缓存回写到内存。**Lock前缀指令导致在执行指令期间，声言处理器的 LOCK# 信号。在多处理器环境中，LOCK# 信号确保在声言该信号期间，处理器可以独占使用任何共享内存。（因为它会锁住总线，导致其他CPU不能访问总线，不能访问总线就意味着不能访问系统内存），但是在最近的处理器里，LOCK＃信号一般不锁总线，而是锁缓存，毕竟锁总线开销比较大。在8.1.4章节有详细说明锁定操作对处理器缓存的影响，对于Intel486和Pentium处理器，在锁操作时，总是在总线上声言LOCK#信号。但在P6和最近的处理器中，如果访问的内存区域已经缓存在处理器内部，则不会声言LOCK#信号。相反地，它会锁定这块内存区域的缓存并回写到内存，并使用缓存一致性机制来确保修改的原子性，此操作被称为“缓存锁定”，缓存一致性机制会阻止同时修改被两个以上处理器缓存的内存区域数据。



- **一个处理器的缓存回写到内存会导致其他处理器的缓存无效。**IA-32处理器和Intel 64处理器使用MESI（修改，独占，共享，无效）控制协议去维护内部缓存和其他处理器缓存的一致性。在多核处理器系统中进行操作的时候，IA-32 和Intel 64处理器能嗅探其他处理器访问系统内存和它们的内部缓存。它们使用嗅探技术保证它的内部缓存，系统内存和其他处理器的缓存的数据在总线上保持一致。例如在Pentium和P6 family处理器中，如果通过嗅探一个处理器来检测其他处理器打算写内存地址，而这个地址当前处理共享状态，那么正在嗅探的处理器将无效它的缓存行，在下次访问相同内存地址时，强制执行缓存行填充。

![](https://i.imgur.com/rBcnWpR.png)


**（2）禁止重排序**

在《浅析Java内存模型》一篇中已经对重排序和先行发生规则进行了阐述，其中就有针对volatile的规则，如果对此不熟悉可以先去阅读下《浅析Java内存模型》，下面将对volatile的重排序作进一步说明。

下面是JMM针对编译器指定的volatile重排序规则表：

![](http://img.blog.csdn.net/20170208175021247?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上表我们可以看出：


- 当第二个操作是volatile写时，不管第一个操作是什么都不能进行重排序。这个规则确保volatile写之前的操作不会被编译器重排序到volatile写之后。
- 当第一个操作是volatile读时，不管第二个操作是什么都不能进行重排序。这个规则确保volatile读之后的操作不会被编译器重排序到volatile读之前。
- 当第一个操作是volatile写，第二个操作是volatile读时，不能进行重排序。

为了实现volatile的内存语义，编译器在生成字节码时会在指令序列中插入内存屏障（观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令，lock前缀指令其实就相当于一个内存屏障。）来禁止特定类型的处理器重排序。



![](http://img.blog.csdn.net/20170208175041780?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


这里温习下常见处理器允许的重排序类型的列表：

![](https://i.imgur.com/BzKOlm1.png)

#### 3.volatile 使用场景  ####

下面引用了网上volatile几种典型使用场景供读者参考：

![](https://i.imgur.com/0FHNUiz.png)

![](https://i.imgur.com/YP2FCs3.png)

![](https://i.imgur.com/dIdXR2r.png)

![](https://i.imgur.com/Gnb95MD.png)

![](https://i.imgur.com/7dgG1WC.png)



### 三、锁的优化及注意事项 ###

这里我觉得是时候对锁的相关知识进行普及了，别急.....,接着往下看吧！！！





















































