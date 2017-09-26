### 浅析ReentrantLock ###
***
一、ReentrantLock概述

 JDK1.5中引入了新的锁机制——java.util.concurrent.locks中的显式的互斥锁：Lock接口，它提供了比synchronized更加广泛的锁定操作。Lock接口有3个实现它的类：ReentrantLock、ReetrantReadWriteLock.ReadLock和ReetrantReadWriteLock.WriteLock，即重入锁、读锁和写锁。


ReentrantLock（重入锁）拥有与synchronized相同的并发性和内存语义，并提供了超出synchonized的其他高级功能(例如：中断锁等候、条件变量等)，并且使用ReentrantLock比synchronized能获得更好的可伸缩性。


	//默认使用非公平锁，如果要使用公平锁，需要传入参数true  
	Lock lock = new ReentrantLock();
	........  
	lock.lock();  
	try {  
	    doSomething();
	   //如果有return语句，放在这里  
	finally {  
		//锁必须在finally块中释放
	    lock.unlock();          
	}

synchronized VS ReentrantLock（两者都是可同步锁）:

- **性能比较**：在JDK5.0的早期版本中，重入锁的性能远远好于synchronized,但从JDK。0开始，JDK在synchronized上做了大量的优化，使得两者的性能差距并不大。
- **等待可中断**：当持有锁的线程长期不释放锁时，正在等待的线程可以选择放弃等待，改为处理其他事情，它对处理执行时间非常短的同步块很有帮助。而在等待由synchronized产生的互斥锁时，会一直阻塞，是不能被中断的。
- **公平锁/非公平锁**：synchronized中的锁是非公平锁，ReentrantLock默认情况下也是非公平锁，但可以通过构造方法ReentrantLock（ture）来要求使用公平锁。
- **绑定条件**：ReentrantLock对象可以同时绑定多个Condition对象（名曰：条件变量或条件队列），而在synchronized中，锁对象的wait（）和notify（）或notifyAll（）方法可以实现一个隐含条件，但如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无需这么做，只需要多次调用newCondition（）方法即可
- **锁的释放**：用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而ReentrantLockk则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

总结：



- synchronized优点是实现简单，语义清晰，便于JVM堆栈跟踪，加锁解锁过程由JVM自动控制，提供了多种优化方案，使用更广泛；缺点是悲观的排他锁，不能进行高级功能。
- ReentrantLock优点是可定时的、可轮询的与可中断的锁获取操作，提供了读写锁、公平锁和非公平锁；缺点是需手动释放锁unlock，不适合JVM进行堆栈跟踪。




一、ReentrantLock源码浅析

（一）、底层实现

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160412160216863-51025450.png)

ReentrantLock的底层是借助AbstractQueuedSynchronizer实现，所以其数据结构依附于AbstractQueuedSynchronizer的数据结构，关于AQS的数据结构，在前面已经介绍过，不再累赘。ReentrantLock类内部总共存在Sync、NonfairSync、FairSync三个类，NonfairSync与FairSync类继承自Sync类，Sync类继承自AbstractQueuedSynchronizer抽象类。


ReentrantLock重写了AQS的lock、tryAcquire等方法（将作为本篇介绍的重点），提供了公平锁和非公平锁两种选择：
![](https://i.imgur.com/2lLPzuI.png)


	public class ReentrantLock implements Lock, java.io.Serializable {
	    // 序列号
	    private static final long serialVersionUID = 7373984872572414699L;    
		/** 同步器：内部类Sync的一个引用 */
	    private final Sync sync;
	
	    /**
	     * 创建一个非公平锁
	     */
	    public ReentrantLock() {
	        sync = new NonfairSync();
	    }
	
	    /**
	     * 创建一个锁
	     * @param fair true-->公平锁  false-->非公平锁
	     */
	    public ReentrantLock(boolean fair) {
	        sync = (fair)? new FairSync() : new NonfairSync();
	    }
	
	}














![](http://img.blog.csdn.net/20141025105532015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWFubGlud2FuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)






















































































