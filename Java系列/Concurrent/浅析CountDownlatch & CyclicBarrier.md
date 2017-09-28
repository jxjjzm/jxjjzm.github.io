### 浅析CountDownlatch & CyclicBarrier ###
***
### 一、CountDownlatch ###

#### （一）、CountDownlatch概述 ####

**CountDownlatch简介：**

CountDownLatch是在java1.5被引入的，跟它一起被引入的并发工具类还有CyclicBarrier、Semaphore、ConcurrentHashMap和BlockingQueue，它们都存在于java.util.concurrent包下。CountDownLatch，俗称倒计时器，CountDownLatch这个类能够使一个线程等待其他线程完成各自的工作后再执行。例如，应用程序的主线程希望在负责启动框架服务的线程已经启动所有的框架服务之后再执行。

CountDownLatch是通过一个计数器来实现的，计数器的初始值为等待线程的数量。每当一个线程完成了自己的任务后，计数器的值就会减1。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在闭锁上等待的线程就可以恢复执行任务。

![](http://incdn1.b0.upaiyun.com/2015/04/f65cc83b7b4664916fad5d1398a36005.png)

**CountDownlatch示例：**

在使用示例前不妨先给出CountDownLatch的伪代码示例，方便有一个先行映象：

	//Main thread start
	//Create CountDownLatch for N threads
	//Create and start N threads
	//Main thread wait on latch
	//N threads completes there tasks are returns
	//Main thread resume execution


使用示例：



	/**
	*模拟了一个应用程序启动类，它开始时启动了n个线程类，这些线程将检查外部系统并通知闭锁，并且启动类一直在闭锁上等待着。一旦验证和检查了所有外部服务，那么启动类恢复执行。
	*/
	public class Main {
	    public static void main(String[] args)
	    {
	        boolean result = false;
	        try {
	            result = ApplicationStartupUtil.checkExternalServices();
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	        System.out.println("External services validation completed !! Result was :: "+ result);
	    }
	}
	/**
		*ApplicationStartupUtil.java：这个类是一个主启动类，它负责初始化闭锁，然后等待，直到所有服务都被检测完。
		*/
	public class ApplicationStartupUtil
	{
	    //List of service checkers
	    private static List<BaseHealthChecker> _services;
	 
	    //This latch will be used to wait on
	    private static CountDownLatch _latch;
	 
	    private ApplicationStartupUtil()
	    {
	    }
	 
	    private final static ApplicationStartupUtil INSTANCE = new ApplicationStartupUtil();
	 
	    public static ApplicationStartupUtil getInstance()
	    {
	        return INSTANCE;
	    }
	 
	    public static boolean checkExternalServices() throws Exception
	    {
	        //Initialize the latch with number of service checkers
	        _latch = new CountDownLatch(3);
	 
	        //All add checker in lists
	        _services = new ArrayList<BaseHealthChecker>();
	        _services.add(new NetworkHealthChecker(_latch));
	        _services.add(new CacheHealthChecker(_latch));
	        _services.add(new DatabaseHealthChecker(_latch));
	 
	        //Start service checkers using executor framework
	        Executor executor = Executors.newFixedThreadPool(_services.size());
	 
	        for(final BaseHealthChecker v : _services)
	        {
	            executor.execute(v);
	        }
	 
	        //Now wait till all services are checked
	        _latch.await();
	 
	        //Services are file and now proceed startup
	        for(final BaseHealthChecker v : _services)
	        {
	            if( ! v.isServiceUp())
	            {
	                return false;
	            }
	        }
	        return true;
	    }
	}
	  /**
		*NetworkHealthChecker.java：这个类继承了BaseHealthChecker，实现了verifyService()方法。
		*DatabaseHealthChecker.java和CacheHealthChecker.java除了服务名和休眠时间外，与NetworkHealthChecker.java是一样的。
		*/
	public class NetworkHealthChecker extends BaseHealthChecker
	{
	    public NetworkHealthChecker (CountDownLatch latch)  {
	        super("Network Service", latch);
	    }
	 
	    @Override
	    public void verifyService()
	    {
	        System.out.println("Checking " + this.getServiceName());
	        try
	        {
	            Thread.sleep(7000);
	        }
	        catch (InterruptedException e)
	        {
	            e.printStackTrace();
	        }
	        System.out.println(this.getServiceName() + " is UP");
	    }
	}
	  /**
		*BaseHealthChecker.java：这个类是一个Runnable，负责所有特定的外部服务健康的检测。它删除了重复的代码和闭锁的中心控制代码。
		*/
	public abstract class BaseHealthChecker implements Runnable {
	 
	    private CountDownLatch _latch;
	    private String _serviceName;
	    private boolean _serviceUp;
	 
	    //Get latch object in constructor so that after completing the task, thread can countDown() the latch
	    public BaseHealthChecker(String serviceName, CountDownLatch latch)
	    {
	        super();
	        this._latch = latch;
	        this._serviceName = serviceName;
	        this._serviceUp = false;
	    }
	 
	    @Override
	    public void run() {
	        try {
	            verifyService();
	            _serviceUp = true;
	        } catch (Throwable t) {
	            t.printStackTrace(System.err);
	            _serviceUp = false;
	        } finally {
	            if(_latch != null) {
	                _latch.countDown();
	            }
	        }
	    }
	 
	    public String getServiceName() {
	        return _serviceName;
	    }
	 
	    public boolean isServiceUp() {
	        return _serviceUp;
	    }
	    //This methos needs to be implemented by all specific service checker
	    public abstract void verifyService();
	}


Output in console:

	Checking Network Service
	Checking Cache Service
	Checking Database Service
	Database Service is UP
	Cache Service is UP
	Network Service is UP
	External services validation completed !! Result was :: true


#### （二）、CountDownlatch源码分析 ####

	public class CountDownLatch {
	    // 同步队列——CountDownLatch类的内部只有一个Sync类型的属性，这个属性相当重要
	    private final Sync sync;

		//CountDownLatch类存在一个内部类Sync，继承自AbstractQueuedSynchronizer
		private static final class Sync extends AbstractQueuedSynchronizer {
		        // 版本号
		        private static final long serialVersionUID = 4982264981922014374L;
		        
		        // 构造器
		        Sync(int count) {
		            setState(count);
		        }
		        
		        // 返回当前计数
		        int getCount() {
		            return getState();
		        }
		
		        // 试图在共享模式下获取对象状态
		        protected int tryAcquireShared(int acquires) {
		            return (getState() == 0) ? 1 : -1;
		        }
		
		        // 试图设置状态来反映共享模式下的一个释放
		        protected boolean tryReleaseShared(int releases) {
		            // Decrement count; signal when transition to zero
		            // 无限循环
		            for (;;) {
		                // 获取状态
		                int c = getState();
		                if (c == 0) // 没有被线程占有
		                    return false;
		                // 下一个状态
		                int nextc = c-1;
		                if (compareAndSetState(c, nextc)) // 比较并且设置成功
		                    return nextc == 0;
		            }
		        }
		    }

		//构造一个用给定计数初始化的CountDownLatch，并且构造函数内完成了sync的初始化，并设置了状态数。
		public CountDownLatch(int count) {
		        if (count < 0) throw new IllegalArgumentException("count < 0");
		        // 初始化状态数
		        this.sync = new Sync(count);
		    }

		//await函数将会使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断。
		public void await() throws InterruptedException {
		        // 转发到sync对象上
		        sync.acquireSharedInterruptibly(1);
		    }


		//此函数将递减锁存器的计数，如果计数到达零，则释放所有等待的线程
		public void countDown() {
			        sync.releaseShared(1);
			    }

	}

说明：

**1.底层数据结构**

![](http://img.blog.csdn.net/20151112092345293?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**2.await函数**

由源码可知，对CountDownLatch对象的await的调用会转发为对Sync的acquireSharedInterruptibly（从AQS继承的方法）方法的调用，acquireSharedInterruptibly源码如下　：

		 public final void acquireSharedInterruptibly(int arg)
		            throws InterruptedException {
		        if (Thread.interrupted())
		            throw new InterruptedException();
		        if (tryAcquireShared(arg) < 0)
		            doAcquireSharedInterruptibly(arg);
		    }

acquireSharedInterruptibly又调用了CountDownLatch的内部类Sync的tryAcquireShared和AQS的doAcquireSharedInterruptibly函数。tryAcquireShared函数的源码如下　：

	//该函数只是简单的判断AQS的state是否为0，为0则返回1，不为0则返回-1。
	 protected int tryAcquireShared(int acquires) {
	            return (getState() == 0) ? 1 : -1;
	        }

AQS的doAcquireSharedInterruptibly函数的源码如下　　：

	  private void doAcquireSharedInterruptibly(int arg)
	        throws InterruptedException {
	        // 添加节点至等待队列
	        final Node node = addWaiter(Node.SHARED);
	        boolean failed = true;
	        try {
	            for (;;) { // 无限循环
	                // 获取node的前驱节点
	                final Node p = node.predecessor();
	                if (p == head) { // 前驱节点为头结点
	                    // 试图在共享模式下获取对象状态
	                    int r = tryAcquireShared(arg);
	                    if (r >= 0) { // 获取成功
	                        // 设置头结点并进行繁殖
	                        setHeadAndPropagate(node, r);
	                        // 设置节点next域
	                        p.next = null; // help GC
	                        failed = false;
	                        return;
	                    }
	                }
	                if (shouldParkAfterFailedAcquire(p, node) &&
	                    parkAndCheckInterrupt()) // 在获取失败后是否需要禁止线程并且进行中断检查
	                    // 抛出异常
	                    throw new InterruptedException();
	            }
	        } finally {
	            if (failed)
	                cancelAcquire(node);
	        }
	    }

在AQS的doAcquireSharedInterruptibly中可能会再次调用CountDownLatch的内部类Sync的tryAcquireShared方法和AQS的setHeadAndPropagate方法。AQS的setHeadAndPropagate方法源码如下：

	private void setHeadAndPropagate(Node node, int propagate) {
	        // 获取头结点
	        Node h = head; // Record old head for check below
	        // 设置头结点
	        setHead(node);
	        /*
	         * Try to signal next queued node if:
	         *   Propagation was indicated by caller,
	         *     or was recorded (as h.waitStatus either before
	         *     or after setHead) by a previous operation
	         *     (note: this uses sign-check of waitStatus because
	         *      PROPAGATE status may transition to SIGNAL.)
	         * and
	         *   The next node is waiting in shared mode,
	         *     or we don't know, because it appears null
	         *
	         * The conservatism in both of these checks may cause
	         * unnecessary wake-ups, but only when there are multiple
	         * racing acquires/releases, so most need signals now or soon
	         * anyway.
	         */
	        // 进行判断
	        if (propagate > 0 || h == null || h.waitStatus < 0 ||
	            (h = head) == null || h.waitStatus < 0) {
	            // 获取节点的后继
	            Node s = node.next;
	            if (s == null || s.isShared()) // 后继为空或者为共享模式
	                // 以共享模式进行释放
	                doReleaseShared();
	        }
	    }


AQS的setHeadAndPropagate方法设置头结点并且释放头结点后面的满足条件的结点，该方法中可能会调用到AQS的doReleaseShared方法，其源码如下（前面的AQS部分分析中依然提到，这里再复习下）：

	//该方法在共享模式下释放
	private void doReleaseShared() {
	        /*
	         * Ensure that a release propagates, even if there are other
	         * in-progress acquires/releases.  This proceeds in the usual
	         * way of trying to unparkSuccessor of head if it needs
	         * signal. But if it does not, status is set to PROPAGATE to
	         * ensure that upon release, propagation continues.
	         * Additionally, we must loop in case a new node is added
	         * while we are doing this. Also, unlike other uses of
	         * unparkSuccessor, we need to know if CAS to reset status
	         * fails, if so rechecking.
	         */
	        // 无限循环
	        for (;;) {
	            // 保存头结点
	            Node h = head;
	            if (h != null && h != tail) { // 头结点不为空并且头结点不为尾结点
	                // 获取头结点的等待状态
	                int ws = h.waitStatus; 
	                if (ws == Node.SIGNAL) { // 状态为SIGNAL
	                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 不成功就继续
	                        continue;            // loop to recheck cases
	                    // 释放后继结点
	                    unparkSuccessor(h);
	                }
	                else if (ws == 0 &&
	                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 状态为0并且不成功，继续
	                    continue;                // loop on failed CAS
	            }
	            if (h == head) // 若头结点改变，继续循环  
	                break;
	        }
	    }



综合以上分析，对CountDownLatch的await调用大致会有如下的调用链：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160420173519335-325433593.png)


**3. countDown函数**

	public void countDown() {
	        sync.releaseShared(1);
	    }

由源码可知，对countDown的调用转换为对Sync对象的releaseShared（从AQS继承而来）方法的调用。releaseShared源码在AQS中已经详细分析过，这里不妨再复习下，releaseShared源码如下：
　

		  public final boolean releaseShared(int arg) {
		        if (tryReleaseShared(arg)) {
		            doReleaseShared();
		            return true;
		        }
		        return false;
		    }


releaseShared函数会以共享模式释放对象，并且在函数中会调用到CountDownLatch的tryReleaseShared函数，并且可能会调用AQS的doReleaseShared函数，其中，tryReleaseShared源码如下：

	protected boolean tryReleaseShared(int releases) {
	            // Decrement count; signal when transition to zero
	            // 无限循环
	            for (;;) {
	                // 获取状态
	                int c = getState();
	                if (c == 0) // 没有被线程占有
	                    return false;
	                // 下一个状态
	                int nextc = c-1;
	                if (compareAndSetState(c, nextc)) // 比较并且设置成功
	                    return nextc == 0;
	            }
	        }


AQS的doReleaseShared的源码如下　：

	//此函数在共享模式下释放资源
	private void doReleaseShared() {
	        /*
	         * Ensure that a release propagates, even if there are other
	         * in-progress acquires/releases.  This proceeds in the usual
	         * way of trying to unparkSuccessor of head if it needs
	         * signal. But if it does not, status is set to PROPAGATE to
	         * ensure that upon release, propagation continues.
	         * Additionally, we must loop in case a new node is added
	         * while we are doing this. Also, unlike other uses of
	         * unparkSuccessor, we need to know if CAS to reset status
	         * fails, if so rechecking.
	         */
	        // 无限循环
	        for (;;) {
	            // 保存头结点
	            Node h = head;
	            if (h != null && h != tail) { // 头结点不为空并且头结点不为尾结点
	                // 获取头结点的等待状态
	                int ws = h.waitStatus; 
	                if (ws == Node.SIGNAL) { // 状态为SIGNAL
	                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0)) // 不成功就继续
	                        continue;            // loop to recheck cases
	                    // 释放后继结点
	                    unparkSuccessor(h);
	                }
	                else if (ws == 0 &&
	                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // 状态为0并且不成功，继续
	                    continue;                // loop on failed CAS
	            }
	            if (h == head) // 若头结点改变，继续循环  
	                break;
	        }
	    }




综上分析，对CountDownLatch的countDown调用大致会有如下的调用链：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160420172401757-1374685850.png)



### 二、CyclicBarrier ###

#### （一）、CycliBarrier概述 ####

**CyclicBarrier简介：**

CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。在JDK API中是这样介绍的：一个同步辅助类，它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)。在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时 CyclicBarrier 很有用。因为该 barrier 在释放等待线程后可以重用，所以称它为循环 的 barrier。

- CyclicBarrier 支持一个可选的 Runnable 命令，在一组线程中的最后一个线程到达之后（但在释放所有线程之前），该命令只在每个屏障点运行一次。若在继续所有参与线程之前更新共享状态，此屏障操作很有用。
- 对于失败的同步尝试，CyclicBarrier 使用了一种要么全部要么全不 (all-or-none) 的破坏模式：如果因为中断、失败或者超时等原因，导致线程过早地离开了屏障点，那么在该屏障点等待的其他所有线程也将通过 BrokenBarrierException（如果它们几乎同时被中断，则用 InterruptedException）以反常的方式离开。

**CountDownlatch VS CyclicBarrier：**

-  CountDownLatch的作用是允许1或N个线程等待其他线程完成执行；而CyclicBarrier则是允许N个线程相互等待。
-  CountDownLatch的计数器无法被重置；CyclicBarrier的计数器可以被重置后使用，因此它被称为是循环的barrier。



**CyclicBarrier使用示例：**

	import java.util.concurrent.BrokenBarrierException;
	import java.util.concurrent.CyclicBarrier;
	
	class CyclicBarrierWorker implements Runnable {
	    private int id;
	    private CyclicBarrier barrier;
	
	    public CyclicBarrierWorker(int id, final CyclicBarrier barrier) {
	        this.id = id;
	        this.barrier = barrier;
	    }
	
	    @Override
	    public void run() {
	        // TODO Auto-generated method stub
	        try {
	            System.out.println(id + " th people wait");
	            barrier.await(); // 大家等待最后一个线程到达
	        } catch (InterruptedException | BrokenBarrierException e) {
	            // TODO Auto-generated catch block
	            e.printStackTrace();
	        }
	    }
	}
	
	public class TestCyclicBarrier {
	    public static void main(String[] args) {
	        int num = 10;
	        CyclicBarrier barrier = new CyclicBarrier(num, new Runnable() {
	            @Override
	            public void run() {
	                // TODO Auto-generated method stub
	                System.out.println("go on together!");
	            }
	        });
	        for (int i = 1; i <= num; i++) {
	            new Thread(new CyclicBarrierWorker(i, barrier)).start();
	        }
	    }
	}

执行结果：

	1 th people wait
	2 th people wait
	3 th people wait
	4 th people wait
	5 th people wait
	7 th people wait
	8 th people wait
	6 th people wait
	9 th people wait
	10 th people wait
	go on together!


#### （二）、CycliBarrier源码分析 ####

#### 1.底层数据结构 ####


	public class CyclicBarrier {
	    
	    /** The lock for guarding barrier entry */
	    // 可重入锁
	    private final ReentrantLock lock = new ReentrantLock();
	    /** Condition to wait on until tripped */
	    // 条件队列
	    private final Condition trip = lock.newCondition();
	    /** The number of parties */
	    // 参与的线程数量
	    private final int parties;
	    /* The command to run when tripped */
	    // 由最后一个进入 barrier 的线程执行的操作
	    private final Runnable barrierCommand;
	    /** The current generation */
	    // 当前代
	    private Generation generation = new Generation();
	    // 正在等待进入屏障的线程数量
	    private int count;
		
		private static class Generation {
				//Generation类有一个属性broken，用来表示当前屏障是否被损坏。
		        boolean broken = false;
		}

	/**
	     * Creates a new {@code CyclicBarrier} that will trip when the
	     * given number of parties (threads) are waiting upon it, and which
	     * will execute the given barrier action when the barrier is tripped,
	     * performed by the last thread entering the barrier.
	     *
	     * @param parties the number of threads that must invoke {@link #await}
	     *        before the barrier is tripped
	     * @param barrierAction the command to execute when the barrier is
	     *        tripped, or {@code null} if there is no action
	     * @throws IllegalArgumentException if {@code parties} is less than 1
	     */
		//该构造函数可以指定关联该CyclicBarrier的线程数量，并且可以指定在所有线程都进入屏障后的执行动作，该执行动作由最后一个进行屏障的线程执行。
	    public CyclicBarrier(int parties, Runnable barrierAction) {
	        if (parties <= 0) throw new IllegalArgumentException();
	        this.parties = parties;
	        this.count = parties;
	        this.barrierCommand = barrierAction;
	    }
	
	    /**
	     * Creates a new {@code CyclicBarrier} that will trip when the
	     * given number of parties (threads) are waiting upon it, and
	     * does not perform a predefined action when the barrier is tripped.
	     *
	     * @param parties the number of threads that must invoke {@link #await}
	     *        before the barrier is tripped
	     * @throws IllegalArgumentException if {@code parties} is less than 1
	     * 
	     */
		//该构造函数仅仅执行了关联该CyclicBarrier的线程数量，没有设置执行动作。
	    public CyclicBarrier(int parties) {
	        this(parties, null);
	    }

	}


![](http://images.cnitblog.com/blog/497634/201401/271455098594619.jpg)

从源码很容易看出，CyclicBarrier是由ReentrantLock可重入锁和Condition共同实现的（如上图所示）。概括来讲，就是设置一个计数，每当有线程达到时，计数count-1，Condition.await 进入阻塞， 当count == 0，那么signalAll ，让所有线程得以唤醒，唤醒后立马重置。

#### 2.核心函数分析 ####

**I、await()函数**

在CyclicBarrier中，最重要的方法就是await()，在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待。其源代码如下：


	public int await() throws InterruptedException, BrokenBarrierException {
	        try {
                //底层调用了doWait函数实现
	            return dowait(false, 0L);
	        } catch (TimeoutException toe) {
	            throw new Error(toe); // cannot happen
	        }
	    }
	
	/**
	     * Main barrier code, covering the various policies.
	     */
	    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        // 保存当前锁
        final ReentrantLock lock = this.lock;
        // 锁定
        lock.lock();
        try {
            // 保存当前代
            final Generation g = generation;
            
            if (g.broken) // 屏障被破坏，抛出异常
                throw new BrokenBarrierException();

            if (Thread.interrupted()) { // 线程被中断
                // 损坏当前屏障，并且唤醒所有的线程，只有拥有锁的时候才会调用
                breakBarrier();
                // 抛出异常
                throw new InterruptedException();
            }
            
            // 减少正在等待进入屏障的线程数量
            int index = --count;
            if (index == 0) {  // 正在等待进入屏障的线程数量为0，所有线程都已经进入
                // 运行的动作标识
                boolean ranAction = false;
                try {
                    // 保存运行动作
                    final Runnable command = barrierCommand;
                    if (command != null) // 动作不为空
                        // 运行
                        command.run();
                    // 设置ranAction状态
                    ranAction = true;
                    // 进入下一代
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction) // 没有运行的动作
                        // 损坏当前屏障
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            // 无限循环
            for (;;) {
                try {
                    if (!timed) // 没有设置等待时间
                        // 等待
                        trip.await(); 
                    else if (nanos > 0L) // 设置了等待时间，并且等待时间大于0
                        // 等待指定时长
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) { 
                    if (g == generation && ! g.broken) { // 等于当前代并且屏障没有被损坏
                        // 损坏当前屏障
                        breakBarrier();
                        // 抛出异常
                        throw ie;
                    } else { // 不等于当前带后者是屏障被损坏
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        // 中断当前线程
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken) // 屏障被损坏，抛出异常
                    throw new BrokenBarrierException();

                if (g != generation) // 不等于当前代
                    // 返回索引
                    return index;

                if (timed && nanos <= 0L) { // 设置了等待时间，并且等待时间小于0
                    // 损坏屏障
                    breakBarrier();
                    // 抛出异常
                    throw new TimeoutException();
                }
            }
        } finally {
            // 释放锁
            lock.unlock();
        }
    }


![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160416091245176-498905979.png)

在dowait方法中其实处理逻辑还是比较简单的：

1、首先判断该barrier是否已经断开了，如果断开则抛出BrokenBarrierException异常；

2、判断计算器index是否等于0，如果等于0，则表示所有的线程准备就绪，已经到达某个公共屏障点了，barrier可以进行后续工作了（是否执行某项任务（构造函数决定））；然后调用nextGeneration方法（下面会详细介绍）进行更新换代工作（其中会唤醒所有等待的线程----breakBarrier()中实现，下面会详细讲解到）；

3、否则，通过for循环（for(;;)）使线程一直处于等待状态。直到“有parties个线程到达barrier” 或 “当前线程被中断” 或 “超时”这3者之一发生。

II、 nextGeneration函数

在dowait中有Generation这样一个对象。Generation描述着CyclicBarrier的更显换代。在CyclicBarrier中，同一批线程属于同一代。当有parties个线程到达barrier，generation就会被更新换代。其中broken标识该当前CyclicBarrier是否已经处于中断状态。当index = –count等于0时，标识“有parties个线程到达barrier”，临界条件到达，则执行相应的动作。执行完动作后，则调用nextGeneration进行更新换代

	private void nextGeneration() {
	        // signal completion of last generation
	        // 唤醒所有线程
	        trip.signalAll();
	        // set up next generation
	        // 恢复正在等待进入屏障的线程数量
	        count = parties;
	        // 新生一代
	        generation = new Generation();
	    }

在nextGeneration函数中会调用AQS的signalAll方法，即唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。其源代码如下　：

	public final void signalAll() {
	            if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
	                throw new IllegalMonitorStateException();
	            // 保存condition队列头结点
	            Node first = firstWaiter;
	            if (first != null) // 头结点不为空
	                // 唤醒所有等待线程
	                doSignalAll(first);
	        }

signalAll函数判断头结点是否为空，即条件队列是否为空，然后会调用doSignalAll函数，doSignalAll函数源码如下　：


	private void doSignalAll(Node first) {
	            // condition队列的头结点尾结点都设置为空
	            lastWaiter = firstWaiter = null;
	            // 循环
	            do {
	                // 获取first结点的nextWaiter域结点
	                Node next = first.nextWaiter;
	                // 设置first结点的nextWaiter域为空
	                first.nextWaiter = null;
	                // 将first结点从condition队列转移到sync队列
	                transferForSignal(first);
	                // 重新设置first
	                first = next;
	            } while (first != null);
	        }


doSignalAll函数会依次将条件队列中的节点转移到同步队列中，会调用到transferForSignal函数，其源码如下：

	 final boolean transferForSignal(Node node) {
	        /*
	         * If cannot change waitStatus, the node has been cancelled.
	         */
	        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
	            return false;
	
	        /*
	         * Splice onto queue and try to set waitStatus of predecessor to
	         * indicate that thread is (probably) waiting. If cancelled or
	         * attempt to set waitStatus fails, wake up to resync (in which
	         * case the waitStatus can be transiently and harmlessly wrong).
	         */
	        Node p = enq(node);
	        int ws = p.waitStatus;
	        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
	            LockSupport.unpark(node.thread);
	        return true;
	    }


transferForSignal 函数的作用就是将处于条件队列中的节点转移到同步队列中，并设置结点的状态信息，其中会调用到enq函数，其源代码如下：

	private Node enq(final Node node) {
	        for (;;) { // 无限循环，确保结点能够成功入队列
	            // 保存尾结点
	            Node t = tail;
	            if (t == null) { // 尾结点为空，即还没被初始化
	                if (compareAndSetHead(new Node())) // 头结点为空，并设置头结点为新生成的结点
	                    tail = head; // 头结点与尾结点都指向同一个新生结点
	            } else { // 尾结点不为空，即已经被初始化过
	                // 将node结点的prev域连接到尾结点
	                node.prev = t; 
	                if (compareAndSetTail(t, node)) { // 比较结点t是否为尾结点，若是则将尾结点设置为node
	                    // 设置尾结点的next域为node
	                    t.next = node; 
	                    return t; // 返回尾结点
	                }
	            }
	        }
	    }

enq函数在AQS中已经讲解到了，如不清楚可以先复习下，此处不赘述。

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160416105524973-1191616489.png)

综合上面的分析，实际上在屏障开闸之后重置状态，以待下一次调用。 newGeneration函数的主要方法的调用如上图所示。

III、breakBarrier函数


	private void breakBarrier() {
	        // 设置状态
	        generation.broken = true;
	        // 恢复正在等待进入屏障的线程数量
	        count = parties;
	        // 唤醒所有线程
	        trip.signalAll();
	    }

breakBarrier函数的作用是损坏当前屏障，会唤醒所有在屏障中的线程。可以看到，此函数也调用了AQS的signalAll函数，由signal函数提供支持。


IV、reset函数

reset方法比较简单。但是这里还是要注意一下要先打破当前屏蔽，然后再重建一个新的屏蔽。否则的话可能会导致信号丢失。

	 public void reset() {
	        final ReentrantLock lock = this.lock;
	        lock.lock();
	        try {
	            breakBarrier();   // break the current generation
	            nextGeneration(); // start a new generation
	        } finally {
	            lock.unlock();
	        }
	    }

















































































