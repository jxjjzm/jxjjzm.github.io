### 浅析Semaphore ###
***
#### 一、Semaphore 概述 ####

**Semaphore简介**

Semaphore称为计数信号量，它允许n个任务同时访问某个资源，可以将信号量看做是在向外分发使用资源的许可证，只有成功获取许可证，才能使用资源。在Java API中是这样介绍的，一个计数信号量。从概念上讲，信号量维护了一个许可集。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个 release() 添加一个许可，从而可能释放一个正在阻塞的获取者。但是，不使用实际的许可对象，Semaphore 只对可用许可的号码进行计数，并采取相应的行动。

广义上来说，信号量是对锁的扩展。无论是内部锁synchronized还是重入锁reentrantLock,一次都只允许一个线程访问一个资源，而信号量却可以指定多个线程同时访问某一个资源。Semaphore内部维护了一个计数器，其值为可以访问的共享资源的个数。一个线程要访问共享资源，先获得信号量，如果信号量的计数器值大于1，意味着有共享资源可以访问，则使其计数器值减去1，再访问共享资源。
如果计数器值为0,线程进入休眠。当某个线程使用完共享资源后，释放信号量，并将信号量内部的计数器加1，之前进入休眠的线程将被唤醒并再次试图获得信号量。

使用示例：

	package com.hust.grid.leesf.semaphore;
	
	import java.util.concurrent.Semaphore;
	
	class MyThread extends Thread {
	    private Semaphore semaphore;
	    
	    public MyThread(String name, Semaphore semaphore) {
	        super(name);
	        this.semaphore = semaphore;
	    }
	    
	    public void run() {        
	        int count = 3;
	        System.out.println(Thread.currentThread().getName() + " trying to acquire");
	        try {
	            semaphore.acquire(count);
	            System.out.println(Thread.currentThread().getName() + " acquire successfully");
	            Thread.sleep(1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            semaphore.release(count);
	            System.out.println(Thread.currentThread().getName() + " release successfully");
	        }
	    }
	}
	
	public class SemaphoreDemo {
	    public final static int SEM_SIZE = 10;
	    
	    public static void main(String[] args) {
	        Semaphore semaphore = new Semaphore(SEM_SIZE);
	        MyThread t1 = new MyThread("t1", semaphore);
	        MyThread t2 = new MyThread("t2", semaphore);
	        t1.start();
	        t2.start();
	        int permits = 5;
	        System.out.println(Thread.currentThread().getName() + " trying to acquire");
	        try {
	            semaphore.acquire(permits);
	            System.out.println(Thread.currentThread().getName() + " acquire successfully");
	            Thread.sleep(1000);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        } finally {
	            semaphore.release();
	            System.out.println(Thread.currentThread().getName() + " release successfully");
	        }
	        
	                
	    }
	}


运行结果（某一次）：　

	main trying to acquire
	main acquire successfully
	t1 trying to acquire
	t1 acquire successfully
	t2 trying to acquire
	t1 release successfully
	main release successfully
	t2 acquire successfully
	t2 release successfully


#### 二、Semaphore 源码分析 ####


	public class Semaphore implements java.io.Serializable {
	    // 版本号
	    private static final long serialVersionUID = -3222578661600680210L;
	    // Semaphore自身只有两个属性，最重要的是sync属性，基于Semaphore对象的操作绝大多数都转移到了对sync的操作。
	    private final Sync sync;

		//创建具有给定的许可数和非公平的公平设置的Semaphore
		public Semaphore(int permits) {
		        sync = new NonfairSync(permits);
		    }
		
		//创建具有给定的许可数和给定的公平设置的Semaphore
		public Semaphore(int permits, boolean fair) {
		        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
		    }

		 	// 内部类，继承自AQS
	    abstract static class Sync extends AbstractQueuedSynchronizer {
	        // 版本号
	        private static final long serialVersionUID = 1192457210091910933L;
	        
	        // 构造函数
	        Sync(int permits) {
	            // 设置状态数
	            setState(permits);
	        }
	        
	        // 获取许可
	        final int getPermits() {
	            return getState();
	        }
	
	        // 共享模式下非公平策略获取
	        final int nonfairTryAcquireShared(int acquires) {
	            for (;;) { // 无限循环
	                // 获取许可数
	                int available = getState();
	                // 剩余的许可
	                int remaining = available - acquires;
	                if (remaining < 0 ||
	                    compareAndSetState(available, remaining)) // 许可小于0或者比较并且设置状态成功
	                    return remaining;
	            }
	        }
	        
	        // 共享模式下进行释放
	        protected final boolean tryReleaseShared(int releases) {
	            for (;;) { // 无限循环
	                // 获取许可
	                int current = getState();
	                // 可用的许可
	                int next = current + releases;
	                if (next < current) // overflow
	                    throw new Error("Maximum permit count exceeded");
	                if (compareAndSetState(current, next)) // 比较并进行设置成功
	                    return true;
	            }
	        }
	
	        // 根据指定的缩减量减小可用许可的数目
	        final void reducePermits(int reductions) {
	            for (;;) { // 无限循环
	                // 获取许可
	                int current = getState();
	                // 可用的许可
	                int next = current - reductions;
	                if (next > current) // underflow
	                    throw new Error("Permit count underflow");
	                if (compareAndSetState(current, next)) // 比较并进行设置成功
	                    return;
	            }
	        }
	
	        // 获取并返回立即可用的所有许可
	        final int drainPermits() {
	            for (;;) { // 无限循环
	                // 获取许可
	                int current = getState();
	                if (current == 0 || compareAndSetState(current, 0)) // 许可为0或者比较并设置成功
	                    return current;
	            }
	        }
	    }


		//NonfairSync类继承了Sync类，表示采用非公平策略获取资源，其只有一个tryAcquireShared方法，重写了AQS的该方法
		static final class NonfairSync extends Sync {
	        // 版本号
	        private static final long serialVersionUID = -2694183684443567898L;
	        
	        // 构造函数
	        NonfairSync(int permits) {
	            super(permits);
	        }
	        // 共享模式下获取
	        protected int tryAcquireShared(int acquires) {
	            return nonfairTryAcquireShared(acquires);
	        }
	    }	


		//FairSync类继承了Sync类，表示采用公平策略获取资源，其只有一个tryAcquireShared方法，重写了AQS的该方法
	 	protected int tryAcquireShared(int acquires) {
	            for (;;) { // 无限循环
					//判断该线程是否位于CLH队列的列头，如果是的话返回 -1
	                if (hasQueuedPredecessors()) 
	                    return -1;
	                //获取当前的信号量许可
	                int available = getState();
	                // 剩余的信号量许可数
	                int remaining = available - acquires;
	                if (remaining < 0 ||
	                    compareAndSetState(available, remaining)) // 剩余的许可小于0或者比较设置成功
	                    return remaining;
	            }
	        }


		//此方法从信号量获取一个许可，在提供一个许可前一直将线程阻塞，或者线程被中断
		public void acquire() throws InterruptedException {
		        sync.acquireSharedInterruptibly(1);
		    }


	}


![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160420212139476-6022765.png)

Semaphore与ReentrantLock的内部类的结构相同，类内部总共存在Sync、NonfairSync、FairSync三个类，NonfairSync与FairSync类继承自Sync类，Sync类继承自AbstractQueuedSynchronizer抽象类。

（1）Sync类的属性相对简单，只有一个版本号，Sync类存在如下方法和作用如下：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160420212905913-940764610.png)

（2）Semaphore有两个核心的函数，其中之一是caquire函数：此方法从信号量获取一个许可，在提供一个许可前一直将线程阻塞，或者线程被中断。该方法中将会调用Sync对象的acquireSharedInterruptibly（从AQS继承而来的方法）方法，acquireSharedInterruptibly方法在CountDownLatch中已经进行了分析，在此不再累赘。假设使用非公平策略，主要调用链如下：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160420222127366-1952818264.png)

（3）Semaphore另一个核心函数是release函数：此方法释放一个（多个）许可，将其返回给信号量。该方法中将会调用Sync对象的releaseShared（从AQS继承而来的方法）方法，而releaseShared方法在CountDownLatch中也已经进行了分析，在此不再累赘。同样假设使用非公平策略，主要调用链如下：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160420222923460-914178481.png)

（4）同ReentrantLock一样，对于公平信号量和非公平信号量，他们机制的差异就体现在tryAcquireShared()方法中,公平信号量多了个hasQueuedPredecessors判断的限制。（这个和ReentrantLock中完全一样，不再赘述）



**总结：**

经过分析可知Semaphore的内部工作流程也是基于AQS，并且不同于CyclicBarrier和ReentrantLock，单独使用Semaphore是不会使用到AQS的条件队列的，其实，只有进行await操作才会进入条件队列，其他的都是在同步队列中，只是当前线程会被park。
