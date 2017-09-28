### 浅析ReentrantLock ###
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

Condition主要是为了在J.U.C框架中提供和Java传统的监视器风格的wait，notify和notifyAll方法类似的功能。**JDK的官方解释如下**：条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，就像 Object.wait 做的那样。 （Condition实质上是被绑定到一个锁上，这也是为什么将ReentrantLock放在一起分析的原因。）


http://blog.csdn.net/chenssy/article/details/69279356

http://blog.csdn.net/wojiushiwo945you/article/details/42239113

http://blog.csdn.net/coslay/article/details/45217069

http://www.cnblogs.com/go2sea/p/5630355.html


到这里我们该告一段落了，我们不难发现，ReentrantLock底层实现基本上都是在AQS基础上完成的，所以如果你对AQS不甚了解建议还是回过头来看下前面的 [《浅析Unsafe & CAS & AQS》](https://github.com/jxjjzm/jxjjzm.github.io/blob/master/Java%E7%B3%BB%E5%88%97/Concurrent/%E6%B5%85%E6%9E%90Unsafe%20%26%20CAS%20%26%20AQS.md) 。




























































