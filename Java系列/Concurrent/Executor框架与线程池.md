### Executor框架与线程池 ###
***

### 一、线程池概述 ###

线程池的概念：首先创建一些线程，它们的集合称为线程池，当服务器接收到一个客户端请求后，就从线程池中取出一个空闲的线程为之服务，服务完后不关闭该线程，而是将该线程还回到线程池中。（在线程池的编程模式下，任务是提交给整个线程池，而不是直接交给某个线程，线程池在拿到任务后，它就在内部找有无空闲的线程，再把任务交给内部某个空闲的线程——一个线程同时只能执行一个任务，但可以同时向一个线程池提交多个任务）

合理利用线程池能够带来三个好处：


- 第一：降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- 第二：提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
- 第三：提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。



### 二、线程池实现原理 ###


![](https://i.imgur.com/FwuB2Y4.png)

概括一下：

- Executor接口是最基础的任务执行接口，Executor接口中只定义了一个方法execute（Runnable command），该方法接收一个Runable实例，它用来执行一个任务，任务即一个实现了Runnable接口的类。


	public interface Executor {
	
	    /**
	     * Executes the given command at some time in the future.  
	     * 用来执行已经提交的Runnable任务对象，提供了一种将“任务提交”与“任务执行”解耦的方法。
	     * @param command the runnable task
	     * @throws RejectedExecutionException if this task cannot be
	     * accepted for execution
	     * @throws NullPointerException if command is null
	     */
	    void execute(Runnable command);
	}





- ExecutorService接口，“执行者服务”接口，可以说是真正的线程池接口，在Executor接口的基础上做了一些扩展，，主要实现了ExecutorService 和 futureTask 相关的一些任务创建和提交的方法。


		public interface ExecutorService extends Executor {
		    // 关闭线程池，已提交的任务继续执行，不接受继续提交新任务
		    void shutdown();
		    // 关闭线程池，尝试停止正在执行的所有任务，不接受继续提交新任务（它和shutdown()方法相比，加了一个单词“now”，区别在于它会去停止当前正在进行的任务）
		    List<Runnable> shutdownNow();
		    // 线程池是否已关闭
		    boolean isShutdown();
		    // 如果调用了 shutdown() 或 shutdownNow() 方法后，所有任务结束了，那么返回true（这个方法必须在调用shutdown或shutdownNow方法之后调用才会返回true）
		    boolean isTerminated();
		    // 等待所有任务完成，并设置超时时间。
		    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;


		    // 提交一个 Callable 任务，返回一个Future代表这个任务，等到任务成功执行，Future#get()方法会返回null
		    <T> Future<T> submit(Callable<T> task);
		    // 提交一个 Runnable 任务，等到任务执行结束，Future#get()方法会返回这个给定的result
		    <T> Future<T> submit(Runnable task, T result);
		    // 提交一个 Runnable 任务，并返回一个Future代表等待的任务执行的结果， 等到任务成功执行，Future#get()方法会返回任务执行的结果
		    Future<?> submit(Runnable task);
		    // 执行所有任务，返回 Future 类型的一个 list
		    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
		    // 执行所有任务，返回 Future 类型的一个 list，但是这里设置了超时时间
		    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit throws InterruptedException;
		    // 只要其中的一个任务结束了，就可以返回，返回执行完的那个任务的结果
		    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException,ExecutionException;
		  
		    // 只要其中的一个任务结束了，就可以返回，返回执行完的那个任务的结果，不过这个带超时，超过指定的时间，抛出 TimeoutException 异常
		    <T> T invokeAny(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
		}


- ScheduledExecutorService接口继承了ExecutorService接口，提供了带"周期执行"功能ExecutorService；

		/**
		 * 在给定延时后，创建并执行一个一次性的Runnable任务
		 * 任务执行完毕后，ScheduledFuture#get()方法会返回null
		 */
		public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);
		 
		/**
		 * 在给定延时后，创建并执行一个ScheduledFutureTask
		 * ScheduledFuture 可以获取结果或取消任务
		 */
		public <V> ScheduledFuture<V> schedule(Callable<V> callable, ong delay, TimeUnit unit);
		 
		/**
		 * 创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期
		 * 也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推
		 * 如果执行任务发生异常，随后的任务将被禁止，否则任务只会在被取消或者Executor被终止后停止
		 * 如果任何执行的任务超过了周期，随后的执行会延时，不会并发执行
		 */
		public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
		                                                  long initialDelay,
		                                                  long period,
		                                                  TimeUnit unit);
		 
		/**
		 * 创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟
		 * 如果执行任务发生异常，随后的任务将被禁止，否则任务只会在被取消或者Executor被终止后停止
		 */
		public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
		                                                     long initialDelay,
		                                                     long delay,
		                                                     TimeUnit unit);






- ThreadPoolExecutor 继承自AbstractExecutorService，是最核心的一个类，是线程池的内部实现。（后面会详细讲解到）
- ScheduledThreadPoolExecutor既继承了TheadPoolExecutor线程池，也实现了ScheduledExecutorService接口，是带"周期执行"功能的线程池；（后面会详细讲解到）
- Executors是线程池的静态工厂，其提供了快捷创建线程池的静态方法。（后面会详细讲解到）


#### （一）、ThreadPoolExecutor ####

#### 1、ThreadPoolExecutor构造参数 ####

	 public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }





- corePoolSize

线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。



- maximumPoolSize

线程池中允许的最大线程数。线程池的阻塞队列满了之后，如果还有任务提交，如果当前的线程数小于maximumPoolSize，则会新建线程来执行任务。注意，如果使用的是无界队列，该参数也就没有什么效果了。



- keepAliveTime

线程空闲时的存活时间。当线程池线程数量超过corePoolSize时，多余的空闲线程的存活时间。即，超过corePoolSize的空闲线程，在多长时间内，会被销毁。



- workQueue

workQueue必须是BlockingQueue阻塞队列。当线程池中的线程数超过它的corePoolSize的时候，线程会进入阻塞队列进行阻塞等待。通过workQueue，线程池实现了阻塞功能



（1）不排队直接提交的队列

该功能由SynchronousQueue对象提供。SynchronousQueue是一个特殊的
BlockingQueue。SynchronousQueue没有容量，每一个插入操作都要等待一个相应的删除操作，反之，每一个删除操作都要等待对应的插入操作。如果使用SynchronousQueue，提交的任务不会被真实的保存，而总是将新任务提交给线程执行，如果没有空闲的进程，则尝试创建新的进程，如果进程数量已经达到最大值，则执行拒绝策略。因此，使用SynchronousQueue队列，通常要设置很大的maximumPoolSize值，否则很容易执行拒绝策略。Executors.newCachedThreadPool()采用的便是这种策略

（2）有界的任务队列

有界的任务队列可以使用ArrayBlockingQueue实现。ArrayBlockingQueue的构造函数必须带一个容量参数，表示该队列的最大容量（public ArrayBlockingQueue(int capacity)）,当使用有界的任务队列时，若有新的任务需要执行，如果线程池的实际线程数小于corePoolSize,则会优先创建新的线程，若大于corePoolSize，则会将新任务加入等待队列。若等待队列已满，无法加入，则在总线程数不大于maximumPoolSize的前提下，创建新的进程执行任务。若大于maximumPoolSize,则执行拒绝策略。可见，有界队列仅当在任务队列装满时，才可能将线程数提升到corePoolSize以上，换言之，除非系统非常繁忙，否则确保核心线程数维持在corePoolSize。

（3）无界的任务队列

无界的任务队列可以通过LinkedBlockingQueue类实现。与有界队列相比，除非系统资源耗尽，否则无界的任务队列不存在任务入队失败的情况。当有新的任务到来，系统的线程数小于corePoolSize时，线程池会生成新的线程执行任务，但当系统的线程数达到corePoolSize后，就不会继续增加。若后续仍有新的任务加入，而又没有空闲的线程资源，则任务直接进入队列等待。若任务创建和处理的速度差异很大，无界队列会保持快速增长，知道耗尽系统内存。

（4）优先任务队列

优先任务队列是带有执行优先级的队列。它通过priorityBlockingQueue实现，可以控制任务的执行先后顺序。它是一个特殊的无界队列。无论是有界队列ArrayBlockingQueue，还是未指定大小的无界队列LinkedBlockingQueue都是按照先进先出算法处理任务的。而PriorityBlockingQueue则可以根据任务自身的优先级顺序先后执行，在确保系统性能的同时，也能有很好的质量保证（总是确保高优先级的任务先执行）。



- threadFactory 线程工厂, 用于创建线程. 如果我们在创建线程池的时候未指定该 threadFactory 参数, 线程池则会使用Executors.defaultThreadFactory() 方法创建默认的线程工厂. 如果我们想要为线程工厂创建的线程设置一些特殊的属性, 例如: 设置见名知意的名字, 设置特定的优先级等等, 那么我们就需要自己去实现 ThreadFactory 接口, 并在实现其抽象方法 newThread()的时候, 使用Thread类包含 threadName (线程名字)的那个构造方法就可以指定线程的名字(通常可以指定见名知意的名字), 还可以用 setPriority() 方法为线程设置特定的优先级等. 然后在创建线程池的时候, 将我们自己实现的 ThreadFactory 接口的实现类对象作为 threadFactory 参数的值传递给线程池的构造方法即可.


- RejectedExecutionHandler，拒绝策略，拒绝策略可以说是系统超负荷运行时的补救措施，通常由于压力太大而引起的，也就是线程池中的线程已经用完了，无法继续为新任务服务，同时，等待队列中也已经排满了，再也塞不下新任务。这时，我们就需要有一套机制，合理地处理这个问题。JDK内置了四种拒绝策略：

	（1）AbortPolicy：直接抛出异常，默认策略；
	
	（2）CallerRunsPolicy：只要线程池未关闭，该策略直接在调用者线程中，运行当前被丢弃的任务。显然这样做不会真的丢弃任务，但是，任务提交线程的性能极有可能会急剧下降。
	
	（3）DiscardOldestPolicy：该策略将丢弃最老的一个请求，也就是即将被执行的一个任务，并尝试再次提交当前任务。
	
	（4）DiscardPolicy：该策略默默地丢弃无法处理的任务，不予任何处理。
	
	以上内置的策略均实现了RejectedExecutionHandler接口，若以上策略仍无法满足实际应用需要，完全可以自己扩展实现RejectedExecutionHandler接口，自定义拒绝策略（如记录日志或持久化存储不能处理的任务）。



#### 2、ThreadPoolExecutor执行原理 ####

**(1)线程池的执行流程**

![](http://images2015.cnblogs.com/blog/677054/201704/677054-20170408210803050-1576156526.png)



- 如果线程池中的线程数量少于corePoolSize，就创建新的线程来执行新添加的任务
- 如果线程池中的线程数量大于等于corePoolSize，但队列workQueue未满，则将新添加的任务放到workQueue中
- 如果线程池中的线程数量大于等于corePoolSize，且队列workQueue已满，但线程池中的线程数量小于maximumPoolSize，则会创建新的线程来处理被添加的任务
- 如果线程池中的线程数量等于了maximumPoolSize，就用RejectedExecutionHandler来执行拒绝策略

![](http://img.blog.csdn.net/20160228204222307)

**(2)线程池状态**

	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	private static final int COUNT_BITS = Integer.SIZE - 3;
	private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
	 
	// runState is stored in the high-order bits
	private static final int RUNNING    = -1 << COUNT_BITS;
	private static final int SHUTDOWN   =  0 << COUNT_BITS;
	private static final int STOP       =  1 << COUNT_BITS;
	private static final int TIDYING    =  2 << COUNT_BITS;
	private static final int TERMINATED =  3 << COUNT_BITS;
	 
	// Packing and unpacking ctl
	private static int runStateOf(int c)     { return c & ~CAPACITY; }
	private static int workerCountOf(int c)  { return c & CAPACITY; }
	private static int ctlOf(int rs, int wc) { return rs | wc; }


ctl这个AtomicInteger的功能很强大，其高3位用于维护线程池运行状态，低29位维护线程池中线程数量

- RUNNING：-1<<COUNT_BITS，即高3位为1，低29位为0，该状态的线程池会接收新任务，也会处理在阻塞队列中等待处理的任务
- SHUTDOWN：0<<COUNT_BITS，即高3位为0，低29位为0，该状态的线程池不会再接收新任务，但还会处理已经提交到阻塞队列中等待处理的任务
- STOP：1<<COUNT_BITS，即高3位为001，低29位为0，该状态的线程池不会再接收新任务，不会处理在阻塞队列中等待的任务，而且还会中断正在运行的任务
- TIDYING：2<<COUNT_BITS，即高3位为010，低29位为0，所有任务都被终止了，workerCount为0，为此状态时还将调用terminated()方法
- TERMINATED：3<<COUNT_BITS，即高3位为100，低29位为0，terminated()方法调用完成后变成此状态

（这些状态均由int型表示，大小关系为 RUNNING<SHUTDOWN<STOP<TIDYING<TERMINATED，这个顺序基本上也是遵循线程池从运行到终止这个过程。）

 

*runStateOf(int c)  方法*：c & 高3位为1，低29位为0的~CAPACITY，用于获取高3位保存的线程池状态

*workerCountOf(int c)方法*：c & 高3位为0，低29位为1的CAPACITY，用于获取低29位的线程数量

*ctlOf(int rs, int wc)方法*：参数rs表示runState，参数wc表示workerCount，即根据runState和workerCount打包合并成ctl


**(3)任务提交内部原理**

**I、execute()  --  提交任务**

	/**
	 * Executes the given task sometime in the future.  The task
	 * may execute in a new thread or in an existing pooled thread.
	 * 在未来的某个时刻执行给定的任务。这个任务用一个新线程执行，或者用一个线程池中已经存在的线程执行
	 *
	 * If the task cannot be submitted for execution, either because this
	 * executor has been shutdown or because its capacity has been reached,
	 * the task is handled by the current {@code RejectedExecutionHandler}.
	 * 如果任务无法被提交执行，要么是因为这个Executor已经被shutdown关闭，要么是已经达到其容量上限，任务会被当前的RejectedExecutionHandler处理
	 *
	 * @param command the task to execute
	 * @throws RejectedExecutionException at discretion of
	 *         {@code RejectedExecutionHandler}, if the task
	 *         cannot be accepted for execution                 RejectedExecutionException是一个RuntimeException
	 * @throws NullPointerException if {@code command} is null
	 */
	public void execute(Runnable command) {
	    if (command == null)
	        throw new NullPointerException();
	     
	    /*
	     * Proceed in 3 steps:
	     *
	     * 1. If fewer than corePoolSize threads are running, try to
	     * start a new thread with the given command as its first
	     * task.  The call to addWorker atomically checks runState and
	     * workerCount, and so prevents false alarms that would add
	     * threads when it shouldn't, by returning false.
	     * 如果运行的线程少于corePoolSize，尝试开启一个新线程去运行command，command作为这个线程的第一个任务
	     *
	     * 2. If a task can be successfully queued, then we still need
	     * to double-check whether we should have added a thread
	     * (because existing ones died since last checking) or that
	     * the pool shut down since entry into this method. So we
	     * recheck state and if necessary roll back the enqueuing if
	     * stopped, or start a new thread if there are none.
	     * 如果任务成功放入队列，我们仍需要一个双重校验去确认是否应该新建一个线程（因为可能存在有些线程在我们上次检查后死了） 或者 从我们进入这个方法后，pool被关闭了
	     * 所以我们需要再次检查state，如果线程池停止了需要回滚入队列，如果池中没有线程了，新开启 一个线程
	     * 
	     * 3. If we cannot queue task, then we try to add a new
	     * thread.  If it fails, we know we are shut down or saturated
	     * and so reject the task.
	     * 如果无法将任务入队列（可能队列满了），需要新开区一个线程（自己：往maxPoolSize发展）
	     * 如果失败了，说明线程池shutdown 或者 饱和了，所以我们拒绝任务
	     */
	    int c = ctl.get();
	     
	    /**
	     * 1、如果当前线程数少于corePoolSize
	     */
	    if (workerCountOf(c) < corePoolSize) {
	        //addWorker()成功，返回
	        if (addWorker(command, true))
	            return;
	         
	        /**
	         * 没有成功addWorker()，再次获取c（凡是需要再次用ctl做判断时，都会再次调用ctl.get()）
	         * 失败的原因可能是：
	         * 1）、线程池已经shutdown，shutdown的线程池不再接收新任务
	         * 2）、workerCountOf(c) < corePoolSize 判断后，由于并发，别的线程先创建了worker线程，导致workerCount>=corePoolSize
	         */
	        c = ctl.get();
	    }
	     
	    /**
	     * 2、如果线程池RUNNING状态，且入队列成功
	     */
	    if (isRunning(c) && workQueue.offer(command)) {
	        int recheck = ctl.get();//再次校验位
	         
	        /**
	         * 再次校验放入workerQueue中的任务是否能被执行
	         * 1）、如果线程池不是运行状态了，应该拒绝添加新任务，从workQueue中删除任务
	         * 2）、如果线程池是运行状态，或者从workQueue中删除任务失败（刚好有一个线程执行完毕，并消耗了这个任务），确保还有线程执行任务（只要有一个就够了）
	         */
	        //如果再次校验过程中，线程池不是RUNNING状态，并且remove(command)--workQueue.remove()成功，拒绝当前command
	        if (! isRunning(recheck) && remove(command))
	            reject(command);
	   
	        else if (workerCountOf(recheck) == 0)
	            addWorker(null, false);  //第一个参数为null，说明只为新建一个worker线程，没有指定firstTask
	                                     //第二个参数为true代表占用corePoolSize，false占用maxPoolSize
	    }
	    /**
	     * 3、如果线程池不是running状态 或者 无法入队列
	     *   尝试开启新线程，扩容至maxPoolSize，如果addWork(command, false)失败了，拒绝当前command
	     */
	    else if (!addWorker(command, false))
	        reject(command);
	}



**execute(Runnable command)**

**参数：**

Runnable command —— 提交执行的任务，不能为空

**执行流程：**

1）、如果线程池当前线程数量少于corePoolSize，则addWorker(command, true)创建新worker线程，如创建成功返回，如没创建成功，则执行后续步骤；

    addWorker(command, true)失败的原因可能是：
    A、线程池已经shutdown，shutdown的线程池不再接收新任务
    B、workerCountOf(c) < corePoolSize 判断后，由于并发，别的线程先创建了worker线程，导致workerCount>=corePoolSize

2、如果线程池还在running状态，将task加入workQueue阻塞队列中，如果加入成功，进行double-check，如果加入失败（可能是队列已满），则执行后续步骤；

    double-check主要目的是判断刚加入workQueue阻塞队列的task是否能被执行
    A、如果线程池已经不是running状态了，应该拒绝添加新任务，从workQueue中删除任务
    B、如果线程池是运行状态，或者从workQueue中删除任务失败（刚好有一个线程执行完毕，并消耗了这个任务），确保还有线程执行任务（只要有一个就够了）
3、如果线程池不是running状态 或者 无法入队列，尝试开启新线程，扩容至maxPoolSize，如果addWork(command, false)失败了，拒绝当前command。

![](http://images2015.cnblogs.com/blog/677054/201704/677054-20170408210905472-1864459025.png)

**II、addWorker()  --  添加worker线程**

	/**
	 * Checks if a new worker can be added with respect to current
	 * pool state and the given bound (either core or maximum). If so,
	 * the worker count is adjusted accordingly, and, if possible, a
	 * new worker is created and started, running firstTask as its
	 * first task. This method returns false if the pool is stopped or
	 * eligible to shut down. It also returns false if the thread
	 * factory fails to create a thread when asked.  If the thread
	 * creation fails, either due to the thread factory returning
	 * null, or due to an exception (typically OutOfMemoryError in
	 * Thread#start), we roll back cleanly.
	 * 检查根据当前线程池的状态和给定的边界(core or maximum)是否可以创建一个新的worker
	 * 如果是这样的话，worker的数量做相应的调整，如果可能的话，创建一个新的worker并启动，参数中的firstTask作为worker的第一个任务
	 * 如果方法返回false，可能因为pool已经关闭或者调用过了shutdown
	 * 如果线程工厂创建线程失败，也会失败，返回false
	 * 如果线程创建失败，要么是因为线程工厂返回null，要么是发生了OutOfMemoryError
	 *
	 * @param firstTask the task the new thread should run first (or
	 * null if none). Workers are created with an initial first task
	 * (in method execute()) to bypass(绕开) queuing when there are fewer
	 * than corePoolSize threads (in which case we always start one),
	 * or when the queue is full (in which case we must bypass queue).
	 * Initially idle threads are usually created via
	 * prestartCoreThread or to replace other dying workers.
	 *
	 * @param core if true use corePoolSize as bound, else
	 * maximumPoolSize. (A boolean indicator is used here rather than a
	 * value to ensure reads of fresh values after checking other pool
	 * state).
	 * @return true if successful
	 */
	private boolean addWorker(Runnable firstTask, boolean core) {
	    //外层循环，负责判断线程池状态
	    retry:
	    for (;;) {
	        int c = ctl.get();
	        int rs = runStateOf(c); //状态
	 
	        // Check if queue empty only if necessary.
	        /**
	         * 线程池的state越小越是运行状态，runnbale=-1，shutdown=0,stop=1,tidying=2，terminated=3
	         * 1、如果线程池state已经至少是shutdown状态了
	         * 2、并且以下3个条件任意一个是false
	         *   rs == SHUTDOWN         （隐含：rs>=SHUTDOWN）false情况： 线程池状态已经超过shutdown，可能是stop、tidying、terminated其中一个，即线程池已经终止
	         *   firstTask == null      （隐含：rs==SHUTDOWN）false情况： firstTask不为空，rs==SHUTDOWN 且 firstTask不为空，return false，场景是在线程池已经shutdown后，还要添加新的任务，拒绝
	         *   ! workQueue.isEmpty()  （隐含：rs==SHUTDOWN，firstTask==null）false情况： workQueue为空，当firstTask为空时是为了创建一个没有任务的线程，再从workQueue中获取任务，如果workQueue已经为空，那么就没有添加新worker线程的必要了
	         * return false，即无法addWorker()
	         */
	        if (rs >= SHUTDOWN &&
	            ! (rs == SHUTDOWN &&
	               firstTask == null &&
	               ! workQueue.isEmpty()))
	            return false;
	 
	        //内层循环，负责worker数量+1
	        for (;;) {
	            int wc = workerCountOf(c); //worker数量
	             
	            //如果worker数量>线程池最大上限CAPACITY（即使用int低29位可以容纳的最大值）
	            //或者( worker数量>corePoolSize 或  worker数量>maximumPoolSize )，即已经超过了给定的边界
	            if (wc >= CAPACITY ||
	                wc >= (core ? corePoolSize : maximumPoolSize))
	                return false;
	             
	            //调用unsafe CAS操作，使得worker数量+1，成功则跳出retry循环
	            if (compareAndIncrementWorkerCount(c))
	                break retry;
	             
	            //CAS worker数量+1失败，再次读取ctl
	            c = ctl.get();  // Re-read ctl
	             
	            //如果状态不等于之前获取的state，跳出内层循环，继续去外层循环判断
	            if (runStateOf(c) != rs)
	                continue retry;
	            // else CAS failed due to workerCount change; retry inner loop
	            // else CAS失败时因为workerCount改变了，继续内层循环尝试CAS对worker数量+1
	        }
	    }
	 
	    /**
	     * worker数量+1成功的后续操作
	     * 添加到workers Set集合，并启动worker线程
	     */
	    boolean workerStarted = false;
	    boolean workerAdded = false;
	    Worker w = null;
	    try {
	        final ReentrantLock mainLock = this.mainLock; 
	        w = new Worker(firstTask); //1、设置worker这个AQS锁的同步状态state=-1
	                                   //2、将firstTask设置给worker的成员变量firstTask
	                                   //3、使用worker自身这个runnable，调用ThreadFactory创建一个线程，并设置给worker的成员变量thread
	        final Thread t = w.thread;
	        if (t != null) {
	            mainLock.lock();
	            try {
	                //--------------------------------------------这部分代码是上锁的
	                // Recheck while holding lock.
	                // Back out on ThreadFactory failure or if
	                // shut down before lock acquired.
	                // 当获取到锁后，再次检查
	                int c = ctl.get();
	                int rs = runStateOf(c);
	 
	                //如果线程池在运行running<shutdown 或者 线程池已经shutdown，且firstTask==null（可能是workQueue中仍有未执行完成的任务，创建没有初始任务的worker线程执行）
	                //worker数量-1的操作在addWorkerFailed()
	                if (rs < SHUTDOWN ||
	                    (rs == SHUTDOWN && firstTask == null)) {
	                    if (t.isAlive()) // precheck that t is startable   线程已经启动，抛非法线程状态异常
	                        throw new IllegalThreadStateException();
	                     
	                    workers.add(w);//workers是一个HashSet<Worker>
	                     
	                    //设置最大的池大小largestPoolSize，workerAdded设置为true
	                    int s = workers.size();
	                    if (s > largestPoolSize)
	                        largestPoolSize = s;
	                    workerAdded = true;
	                }
	              //--------------------------------------------
	            } 
	            finally {
	                mainLock.unlock();
	            }
	             
	            //如果往HashSet中添加worker成功，启动线程
	            if (workerAdded) {
	                t.start();
	                workerStarted = true;
	            }
	        }
	    } finally {
	        //如果启动线程失败
	        if (! workerStarted)
	            addWorkerFailed(w);
	    }
	    return workerStarted;
	}



**addWorker(Runnable firstTask, boolean core)**

**参数：**

    

- firstTask：    worker线程的初始任务，可以为空
- core：         true——将corePoolSize作为上限，false——将maximumPoolSize作为上限


**addWorker方法有4种传参的方式：**

- addWorker(command, true)——线程数小于corePoolSize时，放一个需要处理的task进Workers Set。如果Workers Set长度超过corePoolSize，就返回false ；
- addWorker(command, false)——当队列被放满时，就尝试将这个新来的task直接放入Workers Set，而此时Workers Set的长度限制是maximumPoolSize。如果线程池也满了的话就返回false ；
- addWorker(null, false)——放入一个空的task进workers Set，长度限制是maximumPoolSize。这样一个task为空的worker在线程执行的时候会去任务队列里拿任务，这样就相当于创建了一个新的线程，只是没有马上分配任务 ；
- addWorker(null, true)——这个方法就是放一个null的task进Workers Set，而且是在小于corePoolSize时，如果此时Set中的数量已经达到corePoolSize那就返回false，什么也不干。实际使用中是在prestartAllCoreThreads()方法，这个方法用来为线程池预先启动corePoolSize个worker等待从workQueue中获取任务执行 。


**执行流程：**

1、判断线程池当前是否为可以添加worker线程的状态，可以则继续下一步，不可以return false：

    A、线程池状态>shutdown，可能为stop、tidying、terminated，不能添加worker线程
    B、线程池状态==shutdown，firstTask不为空，不能添加worker线程，因为shutdown状态的线程池不接收新任务
    C、线程池状态==shutdown，firstTask==null，workQueue为空，不能添加worker线程，因为firstTask为空是为了添加一个没有任务的线程再从workQueue获取task，而workQueue为空，说明添加无任务线程已经没有意义

2、线程池当前线程数量是否超过上限（corePoolSize 或 maximumPoolSize），超过了return false，没超过则对workerCount+1，继续下一步

3、在线程池的ReentrantLock保证下，向Workers Set中添加新创建的worker实例，添加完成后解锁，并启动worker线程，如果这一切都成功了，return true，如果添加worker入Set失败或启动失败，调用addWorkerFailed()逻辑



![](http://images2015.cnblogs.com/blog/677054/201704/677054-20170408211358816-1277836615.png)

**III、runWorker()  --  执行任务**

任务添加成功后实际执行的是runWorker这个方法，这个方法非常重要，简单来说它做的就是：

- 第一次启动会执行初始化传进来的任务firstTask；
- 然后会从workQueue中取任务执行，如果队列为空则等待keepAliveTime这么长时间。


		/**
		 * Main worker run loop.  Repeatedly gets tasks from queue and
		 * executes them, while coping with a number of issues:
		 * 重复的从队列中获取任务并执行，同时应对一些问题：
		 *
		 * 1. We may start out with an initial task, in which case we
		 * don't need to get the first one. Otherwise, as long as pool is
		 * running, we get tasks from getTask. If it returns null then the
		 * worker exits due to changed pool state or configuration
		 * parameters.  Other exits result from exception throws in
		 * external code, in which case completedAbruptly holds, which
		 * usually leads processWorkerExit to replace this thread.
		 * 我们可能使用一个初始化任务开始，即firstTask为null
		 * 然后只要线程池在运行，我们就从getTask()获取任务
		 * 如果getTask()返回null，则worker由于改变了线程池状态或参数配置而退出
		 * 其它退出因为外部代码抛异常了，这会使得completedAbruptly为true，这会导致在processWorkerExit()方法中替换当前线程
		 *
		 * 2. Before running any task, the lock is acquired to prevent
		 * other pool interrupts while the task is executing, and
		 * clearInterruptsForTaskRun called to ensure that unless pool is
		 * stopping, this thread does not have its interrupt set.
		 * 在任何任务执行之前，都需要对worker加锁去防止在任务运行时，其它的线程池中断操作
		 * clearInterruptsForTaskRun保证除非线程池正在stoping，线程不会被设置中断标示
		 *
		 * 3. Each task run is preceded by a call to beforeExecute, which
		 * might throw an exception, in which case we cause thread to die
		 * (breaking loop with completedAbruptly true) without processing
		 * the task.
		 * 每个任务执行前会调用beforeExecute()，其中可能抛出一个异常，这种情况下会导致线程die（跳出循环，且completedAbruptly==true），没有执行任务
		 * 因为beforeExecute()的异常没有cache住，会上抛，跳出循环
		 *
		 * 4. Assuming beforeExecute completes normally, we run the task,
		 * gathering any of its thrown exceptions to send to
		 * afterExecute. We separately handle RuntimeException, Error
		 * (both of which the specs guarantee that we trap) and arbitrary
		 * Throwables.  Because we cannot rethrow Throwables within
		 * Runnable.run, we wrap them within Errors on the way out (to the
		 * thread's UncaughtExceptionHandler).  Any thrown exception also
		 * conservatively causes thread to die.
		 * 假定beforeExecute()正常完成，我们执行任务
		 * 汇总任何抛出的异常并发送给afterExecute(task, thrown)
		 * 因为我们不能在Runnable.run()方法中重新上抛Throwables，我们将Throwables包装到Errors上抛（会到线程的UncaughtExceptionHandler去处理）
		 * 任何上抛的异常都会导致线程die
		 *
		 * 5. After task.run completes, we call afterExecute, which may
		 * also throw an exception, which will also cause thread to
		 * die. According to JLS Sec 14.20, this exception is the one that
		 * will be in effect even if task.run throws.
		 * 任务执行结束后，调用afterExecute()，也可能抛异常，也会导致线程die
		 * 根据JLS Sec 14.20，这个异常（finally中的异常）会生效
		 *
		 * The net effect of the exception mechanics is that afterExecute
		 * and the thread's UncaughtExceptionHandler have as accurate
		 * information as we can provide about any problems encountered by
		 * user code.
		 *
		 * @param w the worker
		 */
		final void runWorker(Worker w) {
		    Thread wt = Thread.currentThread();
		    Runnable task = w.firstTask;
		    w.firstTask = null;
		    w.unlock(); // allow interrupts
		                // new Worker()是state==-1，此处是调用Worker类的tryRelease()方法，将state置为0， 而interruptIfStarted()中只有state>=0才允许调用中断
		    boolean completedAbruptly = true; //是否“突然完成”，如果是由于异常导致的进入finally，那么completedAbruptly==true就是突然完成的
		    try {
		        /**
		         * 如果task不为null，或者从阻塞队列中getTask()不为null
		         */
		        while (task != null || (task = getTask()) != null) {
		            w.lock(); //上锁，不是为了防止并发执行任务，为了在shutdown()时不终止正在运行的worker
		             
		            // If pool is stopping, ensure thread is interrupted;
		            // if not, ensure thread is not interrupted.  This
		            // requires a recheck in second case to deal with
		            // shutdownNow race while clearing interrupt
		            /**
		             * clearInterruptsForTaskRun操作
		             * 确保只有在线程stoping时，才会被设置中断标示，否则清除中断标示
		             * 1、如果线程池状态>=stop，且当前线程没有设置中断状态，wt.interrupt()
		             * 2、如果一开始判断线程池状态<stop，但Thread.interrupted()为true，即线程已经被中断，又清除了中断标示，再次判断线程池状态是否>=stop
		             *   是，再次设置中断标示，wt.interrupt()
		             *   否，不做操作，清除中断标示后进行后续步骤
		             */
		            if ((runStateAtLeast(ctl.get(), STOP) ||
		                 (Thread.interrupted() &&
		                  runStateAtLeast(ctl.get(), STOP))) &&
		                !wt.isInterrupted())
		                wt.interrupt(); //当前线程调用interrupt()中断
		             
		            try {
		                //执行前（子类实现）
		                beforeExecute(wt, task);
		                 
		                Throwable thrown = null;
		                try {
		                    task.run();
		                } 
		                catch (RuntimeException x) {
		                    thrown = x; throw x;
		                } 
		                catch (Error x) {
		                    thrown = x; throw x;
		                } 
		                catch (Throwable x) {
		                    thrown = x; throw new Error(x);
		                } 
		                finally {
		                    //执行后（子类实现）
		                    afterExecute(task, thrown); //这里就考验catch和finally的执行顺序了，因为要以thrown为参数
		                }
		            } 
		            finally {
		                task = null; //task置为null
		                w.completedTasks++; //完成任务数+1
		                w.unlock(); //解锁
		            }
		        }
		         
		        completedAbruptly = false;
		    } 
		    finally {
		        //处理worker的退出
		        processWorkerExit(w, completedAbruptly);
		    }
		}


**runWorker(Worker w) 执行流程：**



- 1、Worker线程启动后，通过Worker类的run()方法调用runWorker(this)
- 2、执行任务之前，首先worker.unlock()，将AQS的state置为0，允许中断当前worker线程
- 3、开始执行firstTask，调用task.run()，在执行任务前会上锁wroker.lock()，在执行完任务后会解锁，为了防止在任务运行时被线程池一些中断操作中断
- 4、在任务执行前后，可以根据业务场景自定义beforeExecute() 和 afterExecute()方法
- 5、无论在beforeExecute()、task.run()、afterExecute()发生异常上抛，都会导致worker线程终止，进入processWorkerExit()处理worker退出的流程
- 6、如正常执行完当前task后，会通过getTask()从阻塞队列中获取新任务，当队列中没有任务，且获取任务超时，那么当前worker也会进入退出流程

![](http://images2015.cnblogs.com/blog/677054/201704/677054-20170408211458878-1033038857.png)


**IV、getTask()  --  获取任务**


	/**
	 * Performs blocking or timed wait for a task, depending on
	 * current configuration settings, or returns null if this worker
	 * must exit because of any of:  以下情况会返回null
	 * 1. There are more than maximumPoolSize workers (due to
	 *    a call to setMaximumPoolSize).
	 *    超过了maximumPoolSize设置的线程数量（因为调用了setMaximumPoolSize()）
	 * 2. The pool is stopped.
	 *    线程池被stop
	 * 3. The pool is shutdown and the queue is empty.
	 *    线程池被shutdown，并且workQueue空了
	 * 4. This worker timed out waiting for a task, and timed-out
	 *    workers are subject to termination (that is,
	 *    {@code allowCoreThreadTimeOut || workerCount > corePoolSize})
	 *    both before and after the timed wait.
	 *    线程等待任务超时
	 *
	 * @return task, or null if the worker must exit, in which case
	 *         workerCount is decremented
	 *         返回null表示这个worker要结束了，这种情况下workerCount-1
	 */
	private Runnable getTask() {
	    boolean timedOut = false; // Did the last poll() time out?
	 
	    /**
	     * 外层循环
	     * 用于判断线程池状态
	     */
	    retry:
	    for (;;) {
	        int c = ctl.get();
	        int rs = runStateOf(c);
	 
	        // Check if queue empty only if necessary.
	        /**
	         * 对线程池状态的判断，两种情况会workerCount-1，并且返回null
	         * 线程池状态为shutdown，且workQueue为空（反映了shutdown状态的线程池还是要执行workQueue中剩余的任务的）
	         * 线程池状态为stop（shutdownNow()会导致变成STOP）（此时不用考虑workQueue的情况）
	         */
	        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
	            decrementWorkerCount(); //循环的CAS减少worker数量，直到成功
	            return null;
	        }
	 
	        boolean timed;      // Are workers subject to culling?
	                            // 是否需要定时从workQueue中获取
	         
	        /**
	         * 内层循环
	         * 要么break去workQueue获取任务
	         * 要么超时了，worker count-1
	         */
	        for (;;) {
	            int wc = workerCountOf(c);
	            timed = allowCoreThreadTimeOut || wc > corePoolSize; //allowCoreThreadTimeOut默认为false
	                                                                 //如果allowCoreThreadTimeOut为true，说明corePoolSize和maximum都需要定时
	             
	            //如果当前执行线程数<maximumPoolSize，并且timedOut 和 timed 任一为false，跳出循环，开始从workQueue获取任务
	            if (wc <= maximumPoolSize && ! (timedOut && timed))
	                break;
	             
	            /**
	             * 如果到了这一步，说明要么线程数量超过了maximumPoolSize（可能maximumPoolSize被修改了）
	             * 要么既需要计时timed==true，也超时了timedOut==true
	             * worker数量-1，减一执行一次就行了，然后返回null，在runWorker()中会有逻辑减少worker线程
	             * 如果本次减一失败，继续内层循环再次尝试减一
	             */
	            if (compareAndDecrementWorkerCount(c))
	                return null;
	             
	            //如果减数量失败，再次读取ctl
	            c = ctl.get();  // Re-read ctl
	             
	            //如果线程池运行状态发生变化，继续外层循环
	            //如果状态没变，继续内层循环
	            if (runStateOf(c) != rs)
	                continue retry;
	            // else CAS failed due to workerCount change; retry inner loop
	        }
	 
	        try {
	            //poll() - 使用  LockSupport.parkNanos(this, nanosTimeout) 挂起一段时间，interrupt()时不会抛异常，但会有中断响应
	            //take() - 使用 LockSupport.park(this) 挂起，interrupt()时不会抛异常，但会有中断响应
	            Runnable r = timed ?
	                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :    //大于corePoolSize
	                workQueue.take();                                        //小于等于corePoolSize
	             
	            //如获取到了任务就返回
	            if (r != null)
	                return r;
	             
	            //没有返回，说明超时，那么在下一次内层循环时会进入worker count减一的步骤
	            timedOut = true;
	        } 
	        /**
	              * blockingQueue的take()阻塞使用LockSupport.park(this)进入wait状态的，对LockSupport.park(this)进行interrupt不会抛异常，但还是会有中断响应
	              * 但AQS的ConditionObject的await()对中断状态做了判断，会报告中断状态 reportInterruptAfterWait(interruptMode)
	              * 就会上抛InterruptedException，在此处捕获，重新开始循环
	              * 如果是由于shutdown()等操作导致的空闲worker中断响应，在外层循环判断状态时，可能return null
	              */
	        catch (InterruptedException retry) { 
	            timedOut = false; //响应中断，重新开始，中断状态会被清除
	        }
	    }
	}

**getTask()执行流程：**

1、首先判断是否可以满足从workQueue中获取任务的条件，不满足return null

    A、线程池状态是否满足：
        （a）shutdown状态 + workQueue为空 或 stop状态，都不满足，因为被shutdown后还是要执行workQueue剩余的任务，但workQueue也为空，就可以退出了
        （b）stop状态，shutdownNow()操作会使线程池进入stop，此时不接受新任务，中断正在执行的任务，workQueue中的任务也不执行了，故return null返回
    B、线程数量是否超过maximumPoolSize 或 获取任务是否超时
        （a）线程数量超过maximumPoolSize可能是线程池在运行时被调用了setMaximumPoolSize()被改变了大小，否则已经addWorker()成功不会超过maximumPoolSize
        （b）如果 当前线程数量>corePoolSize，才会检查是否获取任务超时，这也体现了当线程数量达到maximumPoolSize后，如果一直没有新任务，会逐渐终止worker线程直到corePoolSize

2、如果满足获取任务条件，根据是否需要定时获取调用不同方法：

    A、workQueue.poll()：如果在keepAliveTime时间内，阻塞队列还是没有任务，返回null
    B、workQueue.take()：如果阻塞队列为空，当前线程会被挂起等待；当队列中有任务加入时，线程被唤醒，take方法返回任务

3、在阻塞从workQueue中获取任务时，可以被interrupt()中断，代码中捕获了InterruptedException，重置timedOut为初始值false，再次执行第1步中的判断，满足就继续获取任务，不满足return null，会进入worker退出的流程


![](http://images2015.cnblogs.com/blog/677054/201704/677054-20170408211632300-254189763.png)


**V、processWorkerExit()  --  worker线程退出**

	/**
	 * Performs cleanup and bookkeeping for a dying worker. Called
	 * only from worker threads. Unless completedAbruptly is set,
	 * assumes that workerCount has already been adjusted to account
	 * for exit.  This method removes thread from worker set, and
	 * possibly terminates the pool or replaces the worker if either
	 * it exited due to user task exception or if fewer than
	 * corePoolSize workers are running or queue is non-empty but
	 * there are no workers.
	 *
	 * @param w the worker
	 * @param completedAbruptly if the worker died due to user exception
	 */
	private void processWorkerExit(Worker w, boolean completedAbruptly) {
	    /**
	     * 1、worker数量-1
	     * 如果是突然终止，说明是task执行时异常情况导致，即run()方法执行时发生了异常，那么正在工作的worker线程数量需要-1
	     * 如果不是突然终止，说明是worker线程没有task可执行了，不用-1，因为已经在getTask()方法中-1了
	     */
	    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted 代码和注释正好相反啊
	        decrementWorkerCount();
	 
	    /**
	     * 2、从Workers Set中移除worker
	     */
	    final ReentrantLock mainLock = this.mainLock;
	    mainLock.lock();
	    try {
	        completedTaskCount += w.completedTasks; //把worker的完成任务数加到线程池的完成任务数
	        workers.remove(w); //从HashSet<Worker>中移除
	    } finally {
	        mainLock.unlock();
	    }
	 
	    /**
	     * 3、在对线程池有负效益的操作时，都需要“尝试终止”线程池
	     * 主要是判断线程池是否满足终止的状态
	     * 如果状态满足，但还有线程池还有线程，尝试对其发出中断响应，使其能进入退出流程
	     * 没有线程了，更新状态为tidying->terminated
	     */
	    tryTerminate();
	 
	    /**
	     * 4、是否需要增加worker线程
	     * 线程池状态是running 或 shutdown
	     * 如果当前线程是突然终止的，addWorker()
	     * 如果当前线程不是突然终止的，但当前线程数量 < 要维护的线程数量，addWorker()
	     * 故如果调用线程池shutdown()，直到workQueue为空前，线程池都会维持corePoolSize个线程，然后再逐渐销毁这corePoolSize个线程
	     */
	    int c = ctl.get();
	    //如果状态是running、shutdown，即tryTerminate()没有成功终止线程池，尝试再添加一个worker
	    if (runStateLessThan(c, STOP)) {
	        //不是突然完成的，即没有task任务可以获取而完成的，计算min，并根据当前worker数量判断是否需要addWorker()
	        if (!completedAbruptly) {
	            int min = allowCoreThreadTimeOut ? 0 : corePoolSize; //allowCoreThreadTimeOut默认为false，即min默认为corePoolSize
	             
	            //如果min为0，即不需要维持核心线程数量，且workQueue不为空，至少保持一个线程
	            if (min == 0 && ! workQueue.isEmpty())
	                min = 1;
	             
	            //如果线程数量大于最少数量，直接返回，否则下面至少要addWorker一个
	            if (workerCountOf(c) >= min)
	                return; // replacement not needed
	        }
	         
	        //添加一个没有firstTask的worker
	        //只要worker是completedAbruptly突然终止的，或者线程数量小于要维护的数量，就新添一个worker线程，即使是shutdown状态
	        addWorker(null, false);
	    }
	}

**processWorkerExit(Worker w, boolean completedAbruptly)**

**参数：**

    

- worker： 要结束的worker
- completedAbruptly：是否突然完成（是否因为异常退出）

**执行流程：**

1、worker数量-1

    A、如果是突然终止，说明是task执行时异常情况导致，即run()方法执行时发生了异常，那么正在工作的worker线程数量需要-1
    B、如果不是突然终止，说明是worker线程没有task可执行了，不用-1，因为已经在getTask()方法中-1了

2、从Workers Set中移除worker，删除时需要上锁mainlock

3、tryTerminate()：在对线程池有负效益的操作时，都需要“尝试终止”线程池，大概逻辑：

    判断线程池是否满足终止的状态
    A、如果状态满足，但还有线程池还有线程，尝试对其发出中断响应，使其能进入退出流程
    B、没有线程了，更新状态为tidying->terminated

4、是否需要增加worker线程，如果线程池还没有完全终止，仍需要保持一定数量的线程

    线程池状态是running 或 shutdown
    A、如果当前线程是突然终止的，addWorker()
    B、如果当前线程不是突然终止的，但当前线程数量 < 要维护的线程数量，addWorker()
    故如果调用线程池shutdown()，直到workQueue为空前，线程池都会维持corePoolSize个线程，然后再逐渐销毁这corePoolSize个线程




#### （二）、Executors ####

Exectors工厂类提供了线程池的初始化接口，主要有如下几种：

**1、newFixedThreadPool**

	public static ExecutorService newFixedThreadPool(int nThreads) {
	    return new ThreadPoolExecutor(nThreads, nThreads,
	                                  0L, TimeUnit.MILLISECONDS,
	                                  new LinkedBlockingQueue<Runnable>());
	}
	 
	public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
	    return new ThreadPoolExecutor(nThreads, nThreads,
	                                  0L, TimeUnit.MILLISECONDS,
	                                  new LinkedBlockingQueue<Runnable>(),
	                                  threadFactory);
	}


构造一个固定线程数目的线程池，配置的corePoolSize与maximumPoolSize大小相同，同时使用了一个无界LinkedBlockingQueue存放阻塞任务，因此多余的任务将存在在阻塞队列，不会由RejectedExecutionHandler处理。（该线程池中的线程数量始终不变。当有一个新任务提交时，线程池中若有空闲线程，则立即执行。若没有，则新任务会被暂存在一个任务队列中，待有线程空闲时，便处理在任务队列中的任务）


它是一个典型且优秀的线程池，它具有线程池提高程序效率和节省创建线程时所耗的开销的优点。但是在线程池空闲时，即线程池中没有可运行任务时，它也不会释放工作线程，还会占用一定的系统资源






**2、newSingleThreadExecutor**


	public static ExecutorService newSingleThreadExecutor() {
	    return new FinalizableDelegatedExecutorService
	        (new ThreadPoolExecutor(1, 1,
	                                0L, TimeUnit.MILLISECONDS,
	                                new LinkedBlockingQueue<Runnable>()));
	}
	 
	public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
	    return new FinalizableDelegatedExecutorService
	        (new ThreadPoolExecutor(1, 1,
	                                0L, TimeUnit.MILLISECONDS,
	                                new LinkedBlockingQueue<Runnable>(),
	                                threadFactory));
	}



初始化的线程池中只有一个线程（实质是一个特殊的newFixedThreadPool，配置corePoolSize=maximumPoolSize=1），如果该线程异常结束，会重新创建一个新的线程继续执行任务，唯一的线程可以保证所提交任务的顺序执行，内部使用LinkedBlockingQueue作为阻塞队列。





**3、newCachedThreadPool**


	public static ExecutorService newCachedThreadPool() {
	    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
	                                  60L, TimeUnit.SECONDS,
	                                  new SynchronousQueue<Runnable>());
	}
	 
	public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
	    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
	                                  60L, TimeUnit.SECONDS,
	                                  new SynchronousQueue<Runnable>(),
	                                  threadFactory);
	}

构造一个缓冲功能的线程池，配置corePoolSize=0，maximumPoolSize=Integer.MAX_VALUE，keepAliveTime=60s,以及一个无容量的阻塞队列 SynchronousQueue，因此任务提交之后，将会创建新的线程执行；线程空闲超过60s将会销毁 。（该方法返回一个可根据实际情况调整线程数量的线程池。在没有任务执行时，当线程的空闲时间超过keepAliveTime，则工作线程将会终止，当提交新任务时，如果没有空闲线程，则创建新线程执行任务，会导致一定的系统开销）


**4、newScheduledThreadPool**


	public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
	    return new ScheduledThreadPoolExecutor(corePoolSize);
	}
	 
	public static ScheduledExecutorService newScheduledThreadPool(
	        int corePoolSize, ThreadFactory threadFactory) {
	    return new ScheduledThreadPoolExecutor(corePoolSize, threadFactory);
	}


构造有定时功能的线程池，配置corePoolSize，无界延迟阻塞队列DelayedWorkQueue；有意思的是：maximumPoolSize=Integer.MAX_VALUE，由于DelayedWorkQueue是无界队列，所以这个值是没有意义的 。













































































































































































































































































































































