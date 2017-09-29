### 浅析ReentrantLock & Condition ###
***
### 一、ReentrantLock概述 ###

 JDK1.5中引入了新的锁机制——java.util.concurrent.locks中的显式的互斥锁：Lock接口，它提供了比synchronized更加广泛的锁定操作。Lock接口有3个实现它的类：ReentrantLock、ReetrantReadWriteLock.ReadLock和ReetrantReadWriteLock.WriteLock，即重入锁、读锁和写锁。

**ReentrantLock 官方API介绍：**一个可重入的互斥锁定 Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁定相同的一些基本行为和语义，但功能更强大。ReentrantLock 将由最近成功获得锁定，并且还没有释放该锁定的线程所拥有。当锁定没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁定并返回。如果当前线程已经拥有该锁定，此方法将立即返回。可以使用 isHeldByCurrentThread() 和 getHoldCount() 方法来检查此情况是否发生。



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
- **等待可中断**：ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。此外，ReentrantLock支持中断处理，而synchronized则不然。
- **公平锁/非公平锁**：synchronized中的锁是非公平锁，ReentrantLock默认情况下也是非公平锁，但可以通过构造方法ReentrantLock（ture）来要求使用公平锁。
- **绑定条件**：ReentrantLock对象可以同时绑定多个Condition对象（名曰：条件变量或条件队列），而在synchronized中，锁对象的wait（）和notify（）或notifyAll（）方法可以实现一个隐含条件，但如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无需这么做，只需要多次调用newCondition（）方法即可
- **锁的释放**：用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而ReentrantLockk则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

总结：



- synchronized优点是实现简单，语义清晰，便于JVM堆栈跟踪，加锁解锁过程由JVM自动控制，提供了多种优化方案，使用更广泛；缺点是悲观的排他锁，不能进行高级功能。
- ReentrantLock优点是可定时的、可轮询的与可中断的锁获取操作，提供了读写锁、公平锁和非公平锁；缺点是需手动释放锁unlock，不适合JVM进行堆栈跟踪。




### 二、ReentrantLock源码浅析（JDK1.8） ###

#### （一）、底层实现 ####

![](http://img.blog.csdn.net/20150819150107452?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

ReentrantLock的底层是借助AbstractQueuedSynchronizer实现，所以其数据结构依附于AbstractQueuedSynchronizer的数据结构，关于AQS的数据结构，在前面已经介绍过，不再累赘。ReentrantLock类内部总共存在Sync、NonfairSync、FairSync三个类，NonfairSync与FairSync类继承自Sync类，Sync类继承自AbstractQueuedSynchronizer抽象类。


ReentrantLock重写了AQS的lock、tryAcquire等方法（将作为本篇介绍的重点），提供了公平锁和非公平锁两种选择：
![](https://i.imgur.com/2lLPzuI.png)

#### 1、ReentrantLock ####

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

		  /**
     * Acquires the lock.
     * 获取锁
     * <p>Acquires the lock if it is not held by another thread and returns
     * immediately, setting the lock hold count to one.
     *
     * <p>If the current thread already holds the lock then the hold
     * count is incremented by one and the method returns immediately.
     *
     * <p>If the lock is held by another thread then the
     * current thread becomes disabled for thread scheduling
     * purposes and lies dormant until the lock has been acquired,
     * at which time the lock hold count is set to one.
     */
    public void lock() {
        sync.lock();
    }

    

    /**
     * Acquires the lock only if it is not held by another thread at the time
     * of invocation.
     * 
     * <p>Acquires the lock if it is not held by another thread and
     * returns immediately with the value {@code true}, setting the
     * lock hold count to one. 
     *
     * <p>If the current thread already holds this lock then the hold
     * count is incremented by one and the method returns {@code true}.
     *
     * <p>If the lock is held by another thread then this method will return
     * immediately with the value {@code false}.
     *
     * @return {@code true} if the lock was free and was acquired by the
     *         current thread, or the lock was already held by the current
     *         thread; and {@code false} otherwise
     */
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    /**
     * Acquires the lock if it is not held by another thread within the given
     * waiting time and the current thread has not been
     * {@linkplain Thread#interrupt interrupted}.
     *
     * @param timeout the time to wait for the lock
     * @param unit the time unit of the timeout argument
     * @return {@code true} if the lock was free and was acquired by the
     *         current thread, or the lock was already held by the current
     *         thread; and {@code false} if the waiting time elapsed before
     *         the lock could be acquired
     * @throws InterruptedException if the current thread is interrupted
     * @throws NullPointerException if the time unit is null
     */
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    /**
     * Attempts to release this lock.
     * 释放锁
     * <p>If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     */
    public void unlock() {
        sync.release(1);
    }

    /**
     * Returns a {@link Condition} instance for use with this
     * {@link Lock} instance.
     *
     * @return the Condition object
     */
    public Condition newCondition() {
        return sync.newCondition();
    }

    /**
     * Queries the number of holds on this lock by the current thread.
     *
     * @return the number of holds on this lock by the current thread,
     *         or zero if this lock is not held by the current thread
     */
    public int getHoldCount() {
        return sync.getHoldCount();
    }

    /**
     * Queries if this lock is held by the current thread.
     * @return {@code true} if current thread holds this lock and
     *         {@code false} otherwise
     */
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    /**
     * Queries if this lock is held by any thread. This method is
     * designed for use in monitoring of the system state,
     * not for synchronization control.
     *
     * @return {@code true} if any thread holds this lock and
     *         {@code false} otherwise
     */
    public boolean isLocked() {
        return sync.isLocked();
    }

    /**
     * Returns {@code true} if this lock has fairness set true.
     *
     * @return {@code true} if this lock has fairness set true
     */
    public final boolean isFair() {
        return sync instanceof FairSync;
    }

    /**
     * Queries whether any threads are waiting to acquire this lock.
     *
     * @return {@code true} if there may be other threads waiting to
     *         acquire the lock
     */
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    /**
     * Queries whether the given thread is waiting to acquire this
     * lock. Note that because cancellations may occur at any time, a
     * {@code true} return does not guarantee that this thread
     * will ever acquire this lock.  This method is designed primarily for use
     * in monitoring of the system state.
     *
     * @param thread the thread
     * @return {@code true} if the given thread is queued waiting for this lock
     * @throws NullPointerException if the thread is null
     */
    public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }

    /**
     * Returns an estimate of the number of threads waiting to
     * acquire this lock. 
     *
     * @return the estimated number of threads waiting for this lock
     */
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    /**
     * Returns a collection containing threads that may be waiting to
     * acquire this lock. 
     *
     * @return the collection of threads
     */
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    /**
     * Queries whether any threads are waiting on the given condition
     * associated with this lock. 
     *
     * @param condition the condition
     * @return {@code true} if there are any waiting threads
     * @throws IllegalMonitorStateException if this lock is not held
     * @throws IllegalArgumentException if the given condition is
     *         not associated with this lock
     * @throws NullPointerException if the condition is null
     */
    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * Returns an estimate of the number of threads waiting on the
     * given condition associated with this lock. 
     *
     * @param condition the condition
     * @return the estimated number of waiting threads
     * @throws IllegalMonitorStateException if this lock is not held
     * @throws IllegalArgumentException if the given condition is
     *         not associated with this lock
     * @throws NullPointerException if the condition is null
     */
    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * Returns a collection containing those threads that may be
     * waiting on the given condition associated with this lock.
     * 
     * @param condition the condition
     * @return the collection of threads
     * @throws IllegalMonitorStateException if this lock is not held
     * @throws IllegalArgumentException if the given condition is
     *         not associated with this lock
     * @throws NullPointerException if the condition is null
     */
    protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
	
	}



通过分析ReentrantLock的源码，可知ReentrantLock都是把具体实现委托给内部类而不是直接继承自bstractQueuedSynchronizer（这样的好处是用户不会看到不需要的方法，也避免了用户错误地使用AbstractQueuedSynchronizer的公开方法而导致错误。），由于Sync继承了AQS，所以基本上都可以转化为对AQS的操作。如将ReentrantLock的lock函数转化为对Sync的lock函数的调用，而具体会根据采用的策略（如公平策略或者非公平策略）的不同而调用到Sync的不同子类。所以可知，在ReentrantLock的背后，是AQS对其服务提供了支持。

![](http://img.blog.csdn.net/20141025105532015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWFubGlud2FuZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)





#### 2、Sync内部类 ####

	abstract static class Sync extends AbstractQueuedSynchronizer {
	        // 序列号
	        private static final long serialVersionUID = -5179523762034025860L;
	        
	        // 获取锁
	        abstract void lock();
	        
	        // 非公平方式获取
	        final boolean nonfairTryAcquire(int acquires) {
	            // 当前线程
	            final Thread current = Thread.currentThread();
	            // 获取状态
	            int c = getState();
	            if (c == 0) { // 表示没有线程正在竞争该锁
	                if (compareAndSetState(0, acquires)) { // 比较并设置状态成功，状态0表示锁没有被占用
	                    // 设置当前线程独占
	                    setExclusiveOwnerThread(current); 
	                    return true; // 成功
	                }
	            }
	            else if (current == getExclusiveOwnerThread()) { // 当前线程拥有该锁
	                int nextc = c + acquires; // 增加重入次数
	                if (nextc < 0) // overflow
	                    throw new Error("Maximum lock count exceeded");
	                // 设置状态
	                setState(nextc); 
	                // 成功
	                return true; 
	            }
	            // 失败
	            return false;
	        }
	        
	        // 试图在共享模式下获取对象状态，此方法应该查询是否允许它在共享模式下获取对象状态，如果允许，则获取它
	        protected final boolean tryRelease(int releases) {
	            int c = getState() - releases;
	            if (Thread.currentThread() != getExclusiveOwnerThread()) // 当前线程不为独占线程
	                throw new IllegalMonitorStateException(); // 抛出异常
	            // 释放标识
	            boolean free = false; 
	            if (c == 0) {
	                free = true;
	                // 已经释放，清空独占
	                setExclusiveOwnerThread(null); 
	            }
	            // 设置标识
	            setState(c); 
	            return free; 
	        }
	        
	        // 判断资源是否被当前线程占有
	        protected final boolean isHeldExclusively() {
	            // While we must in general read state before owner,
	            // we don't need to do so to check if current thread is owner
	            return getExclusiveOwnerThread() == Thread.currentThread();
	        }
	
	        // 新生一个条件
	        final ConditionObject newCondition() {
	            return new ConditionObject();
	        }
	
	        // Methods relayed from outer class
	        // 返回资源的占用线程
	        final Thread getOwner() {        
	            return getState() == 0 ? null : getExclusiveOwnerThread();
	        }
	        // 返回状态
	        final int getHoldCount() {            
	            return isHeldExclusively() ? getState() : 0;
	        }
	
	        // 资源是否被占用
	        final boolean isLocked() {        
	            return getState() != 0;
	        }
	
	        /**
	         * Reconstitutes the instance from a stream (that is, deserializes it).
	         */
	        // 自定义反序列化逻辑
	        private void readObject(java.io.ObjectInputStream s)
	            throws java.io.IOException, ClassNotFoundException {
	            s.defaultReadObject();
	            setState(0); // reset to unlocked state
	        }
	    }


说明


- Sync类主要方法归纳如下：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160412165914863-1444593791.png)



- ReentrantLock类的sync非常重要，对ReentrantLock类的操作大部分都直接转化为对Sync和AbstractQueuedSynchronizer类的操作。
ReentrantLock类中有三个内部类——Sync、NonfairSync、FairSync。其中，Sync是另外两个类的父类，ReentrantLock的公平锁和非公平锁的实现就是通过Sync的两个子类NonfairSync和FairSync来完成的。



	


#### （二）、核心方法分析 ####

#### 1.锁的获取 ####

**I、非公平锁（默认）**

NonfairSync类继承了Sync类，表示采用非公平策略获取锁，其实现了Sync类中抽象的lock方法，源码如下。

		// 非公平锁
	    static final class NonfairSync extends Sync {
	        // 版本号
	        private static final long serialVersionUID = 7316153563782823691L;
	
	        // 获得锁：首先会第一次尝试快速获取锁，如果获取失败，则调用acquire(int arg)方法，该方法定义在AQS中
	        final void lock() {
	            if (compareAndSetState(0, 1)) // 比较并设置状态成功，状态0表示锁没有被占用
	                // 把当前线程设置独占了锁
	                setExclusiveOwnerThread(Thread.currentThread());
	            else // 锁已经被占用，或者set失败
	                // 以独占模式获取对象，忽略中断
	                acquire(1); 
	        }
	
	        protected final boolean tryAcquire(int acquires) {
				//调用父类sync中的nonfairTryAcquire(int acquires)方法
	            return nonfairTryAcquire(acquires);
	        }
	    }


说明：非公平锁与公平锁相比不同之处在于其尝试获取锁的机制不同，其他基本上都是由AQS实现。从lock方法的源码可知，每一次都尝试获取锁，而并不会按照公平等待的原则进行等待，让等待时间最久的线程获得锁。下面是其调用的父类的nonfairTryAcquire(int acquires)方法源码如下。

	final boolean nonfairTryAcquire(int acquires) {
		            final Thread current = Thread.currentThread();
		            int c = getState();
		            if (c == 0) {
		                if (compareAndSetState(0, acquires)) {
		                    setExclusiveOwnerThread(current);
		                    return true;
		                }
		            }
		            else if (current == getExclusiveOwnerThread()) {
		                int nextc = c + acquires;
		                if (nextc < 0) // overflow
		                    throw new Error("Maximum lock count exceeded");
		                setState(nextc);
		                return true;
		            }
		            return false;
		        }

非公平锁尝试获取锁的步骤：


- 如果当前状态为初始状态，那么尝试设置状态；
- 如果状态设置成功后就返回；
- 如果状态被设置，且获取锁的线程又是当前线程的时候，进行状态的自增；
- 如果未设置成功状态且当前线程不是获取锁的线程，那么返回失败。


**II、公平锁**

![](http://img.blog.csdn.net/20150819150517509?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

FairSync类也继承了Sync类，表示采用公平策略获取锁，其实现了Sync类中的抽象lock方法，源码如下。　

		// 公平锁
	    static final class FairSync extends Sync {
	        // 版本序列化
	        private static final long serialVersionUID = -3000897897090466540L;
	
	        final void lock() {
	            // 以独占模式获取对象，忽略中断（acquire(int arg)是AbstractQueuedSynchronizer中的方法）
	            acquire(1);
	        }
	
	        /**
	         * Fair version of tryAcquire.  Don't grant access unless
	         * recursive call or no waiters or is first.
	         */
	        // 尝试公平获取锁
	        protected final boolean tryAcquire(int acquires) {
	            // 获取当前线程
	            final Thread current = Thread.currentThread();
	            // 获取状态
	            int c = getState();
				// 状态为0，表示锁没有被任何线程占用
	            if (c == 0) { 
					// 不存在已经等待更久的线程并且比较并且设置状态成功
	                if (!hasQueuedPredecessors() &&
	                    compareAndSetState(0, acquires)) { 
	                    // 设置当前线程独占
	                    setExclusiveOwnerThread(current);
	                    return true;
	                }
	            }
				 //  如果c != 0，表示该锁已经被线程占有，则判断该锁是否是当前线程占有，若是设置state，否则直接返回false 
	            else if (current == getExclusiveOwnerThread()) {
	                // 下一个状态
	                int nextc = c + acquires;
					// 超过了int的表示范围
	                if (nextc < 0) 
	                    throw new Error("Maximum lock count exceeded");
	                // 设置状态
	                setState(nextc);
	                return true;
	            }
	            return false;
	        }
	    }

比较非公平锁和公平锁获取同步状态的过程，会发现两者唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors()该方法主要做一件事情：主要是判断当前线程是否位于CLH同步队列中的第一个（即当前线程（Node）之前是否有前置节点在等待的判断）。如果是则返回true，否则返回false。

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160412171846863-1305937448.png)


#### 2.锁的释放 ####

通过前面的分析，我们了解到ReentrantLock实现机制有公平锁和非公平锁，两者的主要区别在于公平锁要按照CLH队列等待获取锁，而非公平锁无视CLH队列直接获取锁。但是对于unlock()而已，它是不分为公平锁和非公平锁的。

	public void unlock() {
	        sync.release(1);
	    }



unlock内部使用Sync的release(int arg)释放锁，而release(int arg)是在AQS中定义的：


	 public final boolean release(int arg) {
	        if (tryRelease(arg)) {
	            Node h = head;
	            if (h != null && h.waitStatus != 0)
	                unparkSuccessor(h);
	            return true;
	        }
	        return false;
	    }


与获取同步状态的acquire(int arg)方法相似，释放同步状态的tryRelease(int arg)同样是需要自定义同步组件自己实现（AQS默认实现抛出UnsupportedOperationException）：


	protected final boolean tryRelease(int releases) {
	        //减掉releases
	        int c = getState() - releases;
	        //如果释放的不是持有锁的线程，抛出异常
	        if (Thread.currentThread() != getExclusiveOwnerThread())
	            throw new IllegalMonitorStateException();
	        boolean free = false;
	        //state == 0 表示已经释放完全了，其他线程可以获取同步状态了
	        if (c == 0) {
	            free = true;
	            setExclusiveOwnerThread(null);
	        }
	        setState(c);
	        return free;
	    }


只有当同步状态彻底释放后该方法才会返回true。当state == 0 时，则将锁持有线程设置为null，free= true，表示释放成功。


### 三、说说Condition ###

#### （一）、Condition概述 ####

Condition主要是为了在J.U.C框架中提供和Java传统的监视器风格的wait，notify和notifyAll方法类似的功能。**JDK的官方解释如下**：条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，就像 Object.wait 做的那样。 （Condition实质上是被绑定到一个锁上，这也是为什么将ReentrantLock放在一起分析的原因。）

下图是网上摘抄的关于Condition与Object的监视器方法的对比：

![](http://img.blog.csdn.net/20170405182049570?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbnNzeQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Condition提供了一系列的方法来对阻塞和唤醒线程：



- await() ：造成当前线程在接到信号或被中断之前一直处于等待状态。
- await(long time, TimeUnit unit) ：造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。
- awaitNanos(long nanosTimeout) ：造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。返回值表示剩余时间，如
- 在nanosTimesout之前唤醒，那么返回值 = nanosTimeout - 消耗时间，如果返回值 <= 0 ,则可以认定它已经超时了。
- awaitUninterruptibly() ：造成当前线程在接到信号之前一直处于等待状态。【注意：该方法对中断不敏感】。
- awaitUntil(Date deadline) ：造成当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。如果没有到指定时间就被通知，则返回true，否则表示到了指定时间，返回返回false。
- signal()：唤醒一个等待线程。该线程从等待方法返回前必须获得与Condition相关的锁。
- signalAll()：唤醒所有等待线程。能够从等待方法返回的线程必须获得与Condition相关的锁。

Condition是一种广义上的条件队列。他为线程提供了一种更为灵活的等待/通知模式，线程在调用await方法后执行挂起操作，直到线程等待的某个条件为真时才会被唤醒。Condition必须要配合锁一起使用，因为对共享状态变量的访问发生在多线程环境下。一个Condition的实例必须与一个Lock绑定，因此Condition一般都是作为Lock的内部实现。

**使用示例：**



	/**
	 * 生产者、消费者示例
	 */
	public class ConditionTest {
	    private int storage;
	    private int putCounter;
	    private int getCounter;
	    private Lock lock = new ReentrantLock();
	    private Condition putCondition = lock.newCondition();
	    private Condition getCondition = lock.newCondition();
	
	    public void put() throws InterruptedException {
	        try {
	            lock.lock();
	            if (storage > 0) {
	                putCondition.await();
	            }
	            storage++;
	            System.out.println("put => " + ++putCounter );
	            getCondition.signal();
	        } finally {
	            lock.unlock();
	        }
	    }
	
	    public void get() throws InterruptedException {
	        try {
	            lock.lock();
	            lock.lock();
	            if (storage <= 0) {
	                getCondition.await();
	            }
	            storage--;
	            System.out.println("get  => " + ++getCounter);
	            putCondition.signal();
	        } finally {
	            lock.unlock();
	            lock.unlock();
	        }
	    }
	
	    public class PutThread extends Thread {
	        @Override
	        public void run() {
	            for (int i = 0; i < 100; i++) {
	                try {
	                    put();
	                } catch (InterruptedException e) {
	                }
	            }
	        }
	    }
	
	    public class GetThread extends Thread {
	        @Override
	        public void run() {
	            for (int i = 0; i < 100; i++) {
	                try {
	                    get();
	                } catch (InterruptedException e) {
	                }
	            }
	        }
	    }
	
	    public static void main(String[] args) {
	        final ConditionTest test = new ConditionTest();
	        Thread put = test.new PutThread();
	        Thread get = test.new GetThread();
	        put.start();
	        get.start();
	    }



#### （二）、Condition源码分析 ####

一个Condition的实例必须跟一个Lock绑定。获取一个Condition必须要通过Lock的newCondition()方法。该方法定义在接口Lock下面，返回的结果是绑定到此 Lock 实例的新 Condition 实例。Condition为一个接口，其下仅有一个实现类ConditionObject，由于Condition的操作需要获取相关的锁，而AQS则是同步锁的实现基础，所以ConditionObject则定义为AQS的内部类。每个Condition对象都包含着一个FIFO队列，该队列是Condition对象通知/等待功能的关键。在队列中每一个节点都包含着一个线程引用，该线程就是在该Condition对象上等待的线程。我们看ConditionObject的定义就明白了（其实在AQS中已经讲到过，这里不妨在加深下）：

	public class ConditionObject implements Condition, java.io.Serializable {
	    private static final long serialVersionUID = 1173984872572414699L;
	
	    //头节点
	    private transient Node firstWaiter;
	    //尾节点
	    private transient Node lastWaiter;
	
	    public ConditionObject() {
	    }
	
	    /** 省略方法 **/
	}



![](http://img.blog.csdn.net/20141229154506727?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd29qaXVzaGl3bzk0NXlvdQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Node里面包含了当前线程的引用。Node定义与AQS的CLH同步队列的节点使用的都是同一个类（AbstractQueuedSynchronized.Node静态内部类），在分析核心函数之前，我们有必要复习下Node状态相关知识：

![](http://img.blog.csdn.net/20141229155010366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd29qaXVzaGl3bzk0NXlvdQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

Node的各个状态的主要作用：Cancelled主要是解决线程在持有锁时被外部中断的逻辑，AQS的可中断锁获取方法lockInterrutible()是基于该状态实现的。显式锁必须手动释放锁，尤其是有中断的环境中，一个线程被中断可能仍然持有锁，所以必须注意在finally中unlock。Condition则是支持条件队列的等待操作，是Lock与条件队列关联的基础。Signal是阻塞后继线程的标识，一个等待线程只有在其前驱节点的状态是SIGNAL时才会被阻塞，否则一直执行自旋尝试操作，以减少线程调度的开销。


该了解的知识普及完了，该进入正题了。JUC锁 AQS中，我们了解到AQS有一个队列，同样Condition也有一个等待队列，两者是相对独立的队列，因此一个Lock可以有多个Condition，Lock(AQS)的队列主要是阻塞线程的，而Condition的队列也是阻塞线程，但是它是有阻塞和通知解除阻塞的功能 Condition阻塞时会释放Lock的锁，阻塞、解除阻塞流程请看下面的Condition的await()、signal、signalAll方法。

**1.等待——await方法**

调用Condition的await()方法会使当前线程进入等待状态，同时会加入到Condition等待队列同时释放锁。当从await()方法返回时，当前线程一定是获取了Condition相关连的锁。

	public final void await() throws InterruptedException {
	    // 1.如果当前线程被中断，则抛出中断异常
	    if (Thread.interrupted())
	        throw new InterruptedException();
	    // 2.将当前线程加入到Condition队列中去，这里如果lastWaiter是cancel状态，那么会把它踢出Condition队列。
	    Node node = addConditionWaiter();
	    // 3.调用tryRelease，释放当前线程的锁
	    long savedState = fullyRelease(node);
	    int interruptMode = 0;
	    // 4.为什么会有在AQS的等待队列的判断？
	    // 解答：signal操作会将Node从Condition队列中拿出并且放入到等待队列中去，在不在AQS等待队列就看signal是否执行了
	    // 如果不在AQS等待队列中，就park当前线程，如果在，就退出循环，这个时候如果被中断，那么就退出循环
	    while (!isOnSyncQueue(node)) {
	        LockSupport.park(this);
	        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
	            break;
	    }
	    // 5.这个时候线程已经被signal()或者signalAll()操作给唤醒了，退出了4中的while循环
	    // 自旋等待尝试再次获取锁，调用acquireQueued方法
	    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	        interruptMode = REINTERRUPT;
	    if (node.nextWaiter != null)
			//清除条件队列中所有状态不为Condition的节点
	        unlinkCancelledWaiters();
	    if (interruptMode != 0)
	        reportInterruptAfterWait(interruptMode);
	}



	//加入条件队列（addConditionWaiter()）
	  private Node addConditionWaiter() {
	        Node t = lastWaiter;    //尾节点
	        //Node的节点状态如果不为CONDITION，则表示该节点不处于等待状态，需要清除节点
	        if (t != null && t.waitStatus != Node.CONDITION) {
	            //清除条件队列中所有状态不为Condition的节点
	            unlinkCancelledWaiters();
	            t = lastWaiter;
	        }
	        //当前线程新建节点，状态CONDITION
	        Node node = new Node(Thread.currentThread(), Node.CONDITION);
	        /**
	         * 将该节点加入到条件队列中最后一个位置
	         */
	        if (t == null)
	            firstWaiter = node;
	        else
	            t.nextWaiter = node;
	        lastWaiter = node;
	        return node;
	    }



	//fullyRelease(Node node)，负责释放该线程持有的锁。
	 final long fullyRelease(Node node) {
	        boolean failed = true;
	        try {
	            //节点状态--其实就是持有锁的数量
	            long savedState = getState();
	            //释放锁
	            if (release(savedState)) {
	                failed = false;
	                return savedState;
	            } else {
	                throw new IllegalMonitorStateException();
	            }
	        } finally {
	            if (failed)
	                node.waitStatus = Node.CANCELLED;
	        }
	    }


	
	final boolean isOnSyncQueue(Node node) {
	        //状态为Condition，获取前驱节点为null，返回false
	        if (node.waitStatus == Node.CONDITION || node.prev == null)
	            return false;
	        //后继节点不为null，肯定在CLH同步队列中
	        if (node.next != null)
	            return true;
	
	        return findNodeFromTail(node);
	    }


	  //unlinkCancelledWaiters()：负责将条件队列中状态不为Condition的节点删除
	  private void unlinkCancelledWaiters() {
	            Node t = firstWaiter;
	            Node trail = null;
	            while (t != null) {
	                Node next = t.nextWaiter;
	                if (t.waitStatus != Node.CONDITION) {
	                    t.nextWaiter = null;
	                    if (trail == null)
	                        firstWaiter = next;
	                    else
	                        trail.nextWaiter = next;
	                    if (next == null)
	                        lastWaiter = trail;
	                }
	                else
	                    trail = t;
	                t = next;
	            }
	        }



整个await的过程如下： 
　　

- 1.将当前线程加入Condition锁队列。特别说明的是，这里不同于AQS的队列，这里进入的是Condition的FIFO队列。进行2。 
- 2.释放锁。这里可以看到将锁释放了，否则别的线程就无法拿到锁而发生死锁。进行3。 
- 3.自旋(while)挂起，直到被唤醒或者超时或者CACELLED等。进行4。 
- 4.获取锁(acquireQueued)。并将自己从Condition的FIFO队列中释放，表明自己不再需要锁（我已经拿到锁了）。

可以看到，这个await的操作过程和Object.wait()方法是一样，只不过await()采用了Condition队列的方式实现了Object.wait()的功能。


**2.唤醒——signal和signalAll方法**

await*()清楚了，现在再来看signal/signalAll就容易多了。按照signal/signalAll的需求，就是要将Condition.await()中FIFO队列中第一个Node唤醒（或者全部Node）唤醒。尽管所有Node可能都被唤醒，但是要知道的是仍然只有一个线程能够拿到锁，其它没有拿到锁的线程仍然需要自旋等待，就上上面提到的第4步(acquireQueued)。


	public final void signal() {
	    if (!isHeldExclusively())
	        throw new IllegalMonitorStateException();
	    Node first = firstWaiter;
	    if (first != null)
	        doSignal(first);
	}

这里先判断当前线程是否持有锁，如果没有持有，则抛出异常，然后判断整个condition队列是否为空，不为空则调用doSignal方法来唤醒线程，看看doSignal方法都干了一些什么：

	private void doSignal(Node first) {
	    do {
	        if ( (firstWaiter = first.nextWaiter) == null)
	            lastWaiter = null;
	        first.nextWaiter = null;
	    } while (!transferForSignal(first) &&
	             (first = firstWaiter) != null);
	}

上面的代码很容易看出来，signal就是唤醒Condition队列中的第一个非CANCELLED节点线程，而signalAll就是唤醒所有非CANCELLED节点线程。当然了遇到CANCELLED线程就需要将其从FIFO队列中剔除。

	final boolean transferForSignal(Node node) {
	    /*
	     * 设置node的waitStatus：Condition->0
	     */
	    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
	        return false;
	
	    /*
	     * 加入到AQS的等待队列，让节点继续获取锁
	     * 设置前置节点状态为SIGNAL
	     */
	    Node p = enq(node);
	    int c = p.waitStatus;
	    if (c > 0 || !compareAndSetWaitStatus(p, c, Node.SIGNAL))
	        LockSupport.unpark(node.thread);
	    return true;
	}



signalAll和signal方法类似，主要的不同在于它不是调用doSignal方法，而是调用doSignalAll方法：

	private void doSignalAll(Node first) {
	    lastWaiter = firstWaiter  = null;
	    do {
	        Node next = first.nextWaiter;
	        first.nextWaiter = null;
	        transferForSignal(first);
	        first = next;
	    } while (first != null);
	}

这个方法就相当于把Condition队列中的所有Node全部取出插入到等待队列中去。



3.归纳总结：


![](http://img.blog.csdn.net/20141229155633393?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd29qaXVzaGl3bzk0NXlvdQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



- AQS等待队列与Condition队列是两个相互独立的队列 
- await()就是在当前线程持有锁的基础上释放锁资源，并新建Condition节点加入到Condition的队列尾部，阻塞当前线程 
- signal()就是将Condition的头节点移动到AQS等待节点尾部，让其等待再次获取锁




以下是AQS队列和Condition队列的出入结点的示意图，可以通过这几张图看出线程结点在两个队列中的出入关系和条件。

**I.初始化状态：**AQS等待队列有3个Node，Condition队列有1个Node(也有可能1个都没有)

![](http://img.blog.csdn.net/20150423091636088)

**II.节点1执行Condition.await()** 


- 1.将head后移 
- 2.释放节点1的锁并从AQS等待队列中移除 
- 3.将节点1加入到Condition的等待队列中 
- 4.更新lastWaiter为节点1

![](http://img.blog.csdn.net/20150423091555989)

**III.节点2执行signal()操作**

- 5.将firstWaiter后移 
- 6.将节点4移出Condition队列 
- 7.将节点4加入到AQS的等待队列中去 
- 8.更新AQS的等待队列的tail

![](http://img.blog.csdn.net/20150423091621011)


总而言之，一个线程获取锁后，通过调用Condition的await()方法，会将当前线程先加入到条件队列中，然后释放锁，最后通过isOnSyncQueue(Node node)方法不断自检看节点是否已经在CLH同步队列了，如果是则尝试获取锁，否则一直挂起。当线程调用signal()方法后，程序首先检查当前线程是否获取了锁，然后通过doSignal(Node first)方法唤醒CLH同步队列的首节点。被唤醒的线程，将从await()方法中的while循环中退出来，然后调用acquireQueued()方法竞争同步状态。


到这里我们该告一段落了，我们不难发现，ReentrantLock和Condition底层实现基本上都是在AQS基础上完成的，所以如果你对AQS不甚了解建议还是回过头来看下前面的 [《浅析Unsafe & CAS & AQS》](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90Unsafe%20%26%20CAS%20%26%20AQS.md) 。




























































