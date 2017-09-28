### 浅析Unsafe & CAS & AQS ###
***
### 一、Unsafe ###

java不能直接访问操作系统底层，而是通过本地方法来访问。不过尽管如此，JVM还是开了一个后门，JDK中有一个类Unsafe，它提供了硬件级别的原子操作。这个类尽管里面的方法都是public的，但是并没有办法使用它们，JDK API文档也没有提供任何关于这个类的方法的解释。总而言之，对于Unsafe类的使用都是受限制的，只有授信的代码才能获得该类的实例，当然JDK库里面的类是可以随意使用的。

Unsafe类主要提供了以下功能：

- 通过Unsafe类可以分配内存，可以释放内存：类中提供的3个本地方法allocateMemory、reallocateMemory、freeMemory分别用于分配内存，扩充内存和释放内存，与C语言中的3个方法对应。
- 可以定位对象某字段的内存位置，也可以修改对象的字段值，即使它是私有的。
- 挂起与恢复：将一个线程进行挂起是通过park方法实现的，调用 park后，线程将一直阻塞直到超时或者中断等条件出现。unpark可以终止一个挂起的线程，使其恢复正常。整个并发框架中对线程的挂起操作被封装在 LockSupport类中，LockSupport类中有各种版本pack方法，但最终都调用了Unsafe.park()方法。
- CAS操作：通过compareAndSwapXXX方法实现的。

Unsafe完整源码请参考：[http://www.docjar.com/html/api/sun/misc/Unsafe.java.html](http://www.docjar.com/html/api/sun/misc/Unsafe.java.html "Unsafe完整源码") ，下面列举出与JUC实现紧密的部分源码：



	//下面是sun.misc.Unsafe.java类源码
	package sun.misc;
	import java.lang.reflect.Field;
	/***
	 * This class should provide access to low-level operations and its
	 * use should be limited to trusted code.  Fields can be accessed using
	 * memory addresses, with undefined behaviour occurring if invalid memory
	 * addresses are given.
	 * 这个类提供了一个更底层的操作并且应该在受信任的代码中使用。可以通过内存地址
	 * 存取fields,如果给出的内存地址是无效的那么会有一个不确定的运行表现。
	 * 
	 * @author Tom Tromey (tromey@redhat.com)
	 * @author Andrew John Hughes (gnu_andrew@member.fsf.org)
	 */
	public class Unsafe
	{
	  // Singleton class.
	  private static Unsafe unsafe = new Unsafe();
	  /***
	   * Private default constructor to prevent creation of an arbitrary
	   * number of instances.
	   * 使用私有默认构造器防止创建多个实例
	   */
	  private Unsafe()
	  {
	  }
	  /***
	   * Retrieve the singleton instance of <code>Unsafe</code>.  The calling
	   * method should guard this instance from untrusted code, as it provides
	   * access to low-level operations such as direct memory access.
	   * 获取<code>Unsafe</code>的单例,这个方法调用应该防止在不可信的代码中实例，
	   * 因为unsafe类提供了一个低级别的操作，例如直接内存存取。
	   * 
	   * @throws SecurityException if a security manager exists and prevents
	   *                           access to the system properties.
	   *                           如果安全管理器不存在或者禁止访问系统属性
	   */
	  public static Unsafe getUnsafe()
	  {
	    SecurityManager sm = System.getSecurityManager();
	    if (sm != null)
	      sm.checkPropertiesAccess();
	    return unsafe;
	  }
	  
	  /***
	   * Returns the memory address offset of the given static field.
	   * The offset is merely used as a means to access a particular field
	   * in the other methods of this class.  The value is unique to the given
	   * field and the same value should be returned on each subsequent call.
	   * 返回指定静态field的内存地址偏移量,在这个类的其他方法中这个值只是被用作一个访问
	   * 特定field的一个方式。这个值对于 给定的field是唯一的，并且后续对该方法的调用都应该
	   * 返回相同的值。
	   *
	   * @param field the field whose offset should be returned.
	   *              需要返回偏移量的field
	   * @return the offset of the given field.
	   *         指定field的偏移量
	   */
	  public native long objectFieldOffset(Field field);
	  /***
	   * Compares the value of the integer field at the specified offset
	   * in the supplied object with the given expected value, and updates
	   * it if they match.  The operation of this method should be atomic,
	   * thus providing an uninterruptible way of updating an integer field.
	   * 在obj的offset位置比较integer field和期望的值，如果相同则更新。这个方法
	   * 的操作应该是原子的，因此提供了一种不可中断的方式更新integer field。
	   * 
	   * @param obj the object containing the field to modify.
	   *            包含要修改field的对象
	   * @param offset the offset of the integer field within <code>obj</code>.
	   *               <code>obj</code>中整型field的偏移量
	   * @param expect the expected value of the field.
	   *               希望field中存在的值
	   * @param update the new value of the field if it equals <code>expect</code>.
	   *           如果期望值expect与field的当前值相同，设置filed的值为这个新值
	   * @return true if the field was changed.
	   *                             如果field的值被更改
	   */
	  public native boolean compareAndSwapInt(Object obj, long offset,
	                                          int expect, int update);
	  /***
	   * Compares the value of the long field at the specified offset
	   * in the supplied object with the given expected value, and updates
	   * it if they match.  The operation of this method should be atomic,
	   * thus providing an uninterruptible way of updating a long field.
	   * 在obj的offset位置比较long field和期望的值，如果相同则更新。这个方法
	   * 的操作应该是原子的，因此提供了一种不可中断的方式更新long field。
	   * 
	   * @param obj the object containing the field to modify.
	   *              包含要修改field的对象 
	   * @param offset the offset of the long field within <code>obj</code>.
	   *               <code>obj</code>中long型field的偏移量
	   * @param expect the expected value of the field.
	   *               希望field中存在的值
	   * @param update the new value of the field if it equals <code>expect</code>.
	   *               如果期望值expect与field的当前值相同，设置filed的值为这个新值
	   * @return true if the field was changed.
	   *              如果field的值被更改
	   */
	  public native boolean compareAndSwapLong(Object obj, long offset,
	                                           long expect, long update);
	  /***
	   * Compares the value of the object field at the specified offset
	   * in the supplied object with the given expected value, and updates
	   * it if they match.  The operation of this method should be atomic,
	   * thus providing an uninterruptible way of updating an object field.
	   * 在obj的offset位置比较object field和期望的值，如果相同则更新。这个方法
	   * 的操作应该是原子的，因此提供了一种不可中断的方式更新object field。
	   * 
	   * @param obj the object containing the field to modify.
	   *    包含要修改field的对象 
	   * @param offset the offset of the object field within <code>obj</code>.
	   *         <code>obj</code>中object型field的偏移量
	   * @param expect the expected value of the field.
	   *               希望field中存在的值
	   * @param update the new value of the field if it equals <code>expect</code>.
	   *               如果期望值expect与field的当前值相同，设置filed的值为这个新值
	   * @return true if the field was changed.
	   *              如果field的值被更改
	   */
	  public native boolean compareAndSwapObject(Object obj, long offset,
	                                             Object expect, Object update);
	  /***
	   * Sets the value of the integer field at the specified offset in the
	   * supplied object to the given value.  This is an ordered or lazy
	   * version of <code>putIntVolatile(Object,long,int)</code>, which
	   * doesn't guarantee the immediate visibility of the change to other
	   * threads.  It is only really useful where the integer field is
	   * <code>volatile</code>, and is thus expected to change unexpectedly.
	   * 设置obj对象中offset偏移地址对应的整型field的值为指定值。这是一个有序或者
	   * 有延迟的<code>putIntVolatile</cdoe>方法，并且不保证值的改变被其他线程立
	   * 即看到。只有在field被<code>volatile</code>修饰并且期望被意外修改的时候
	   * 使用才有用。
	   * 
	   * @param obj the object containing the field to modify.
	   *    包含需要修改field的对象
	   * @param offset the offset of the integer field within <code>obj</code>.
	   *       <code>obj</code>中整型field的偏移量
	   * @param value the new value of the field.
	   *      field将被设置的新值
	   * @see #putIntVolatile(Object,long,int)
	   */
	  public native void putOrderedInt(Object obj, long offset, int value);
	  /***
	   * Sets the value of the long field at the specified offset in the
	   * supplied object to the given value.  This is an ordered or lazy
	   * version of <code>putLongVolatile(Object,long,long)</code>, which
	   * doesn't guarantee the immediate visibility of the change to other
	   * threads.  It is only really useful where the long field is
	   * <code>volatile</code>, and is thus expected to change unexpectedly.
	   * 设置obj对象中offset偏移地址对应的long型field的值为指定值。这是一个有序或者
	   * 有延迟的<code>putLongVolatile</cdoe>方法，并且不保证值的改变被其他线程立
	   * 即看到。只有在field被<code>volatile</code>修饰并且期望被意外修改的时候
	   * 使用才有用。
	   * 
	   * @param obj the object containing the field to modify.
	   *    包含需要修改field的对象
	   * @param offset the offset of the long field within <code>obj</code>.
	   *       <code>obj</code>中long型field的偏移量
	   * @param value the new value of the field.
	   *      field将被设置的新值
	   * @see #putLongVolatile(Object,long,long)
	   */
	  public native void putOrderedLong(Object obj, long offset, long value);
	  /***
	   * Sets the value of the object field at the specified offset in the
	   * supplied object to the given value.  This is an ordered or lazy
	   * version of <code>putObjectVolatile(Object,long,Object)</code>, which
	   * doesn't guarantee the immediate visibility of the change to other
	   * threads.  It is only really useful where the object field is
	   * <code>volatile</code>, and is thus expected to change unexpectedly.
	   * 设置obj对象中offset偏移地址对应的object型field的值为指定值。这是一个有序或者
	   * 有延迟的<code>putObjectVolatile</cdoe>方法，并且不保证值的改变被其他线程立
	   * 即看到。只有在field被<code>volatile</code>修饰并且期望被意外修改的时候
	   * 使用才有用。
	   *
	   * @param obj the object containing the field to modify.
	   *    包含需要修改field的对象
	   * @param offset the offset of the object field within <code>obj</code>.
	   *       <code>obj</code>中long型field的偏移量
	   * @param value the new value of the field.
	   *      field将被设置的新值
	   */
	  public native void putOrderedObject(Object obj, long offset, Object value);
	  /***
	   * Sets the value of the integer field at the specified offset in the
	   * supplied object to the given value, with volatile store semantics.
	   * 设置obj对象中offset偏移地址对应的整型field的值为指定值。支持volatile store语义
	   * 
	   * @param obj the object containing the field to modify.
	   *    包含需要修改field的对象
	   * @param offset the offset of the integer field within <code>obj</code>.
	   *       <code>obj</code>中整型field的偏移量
	   * @param value the new value of the field.
	   *       field将被设置的新值
	   */
	  public native void putIntVolatile(Object obj, long offset, int value);
	  /***
	   * Retrieves the value of the integer field at the specified offset in the
	   * supplied object with volatile load semantics.
	   * 获取obj对象中offset偏移地址对应的整型field的值,支持volatile load语义。
	   * 
	   * @param obj the object containing the field to read.
	   *    包含需要去读取的field的对象
	   * @param offset the offset of the integer field within <code>obj</code>.
	   *       <code>obj</code>中整型field的偏移量
	   */
	  public native int getIntVolatile(Object obj, long offset);
	  /***
	   * Sets the value of the long field at the specified offset in the
	   * supplied object to the given value, with volatile store semantics.
	   * 设置obj对象中offset偏移地址对应的long型field的值为指定值。支持volatile store语义
	   *
	   * @param obj the object containing the field to modify.
	   *            包含需要修改field的对象
	   * @param offset the offset of the long field within <code>obj</code>.
	   *               <code>obj</code>中long型field的偏移量
	   * @param value the new value of the field.
	   *              field将被设置的新值
	   * @see #putLong(Object,long,long)
	   */
	  public native void putLongVolatile(Object obj, long offset, long value);
	  /***
	   * Sets the value of the long field at the specified offset in the
	   * supplied object to the given value.
	   * 设置obj对象中offset偏移地址对应的long型field的值为指定值。
	   * 
	   * @param obj the object containing the field to modify.
	   *     包含需要修改field的对象
	   * @param offset the offset of the long field within <code>obj</code>.
	   *     <code>obj</code>中long型field的偏移量
	   * @param value the new value of the field.
	   *     field将被设置的新值
	   * @see #putLongVolatile(Object,long,long)
	   */
	  public native void putLong(Object obj, long offset, long value);
	  /***
	   * Retrieves the value of the long field at the specified offset in the
	   * supplied object with volatile load semantics.
	   * 获取obj对象中offset偏移地址对应的long型field的值,支持volatile load语义。
	   * 
	   * @param obj the object containing the field to read.
	   *    包含需要去读取的field的对象
	   * @param offset the offset of the long field within <code>obj</code>.
	   *       <code>obj</code>中long型field的偏移量
	   * @see #getLong(Object,long)
	   */
	  public native long getLongVolatile(Object obj, long offset);
	  /***
	   * Retrieves the value of the long field at the specified offset in the
	   * supplied object.
	   * 获取obj对象中offset偏移地址对应的long型field的值
	   * 
	   * @param obj the object containing the field to read.
	   *    包含需要去读取的field的对象
	   * @param offset the offset of the long field within <code>obj</code>.
	   *       <code>obj</code>中long型field的偏移量
	   * @see #getLongVolatile(Object,long)
	   */
	  public native long getLong(Object obj, long offset);
	  /***
	   * Sets the value of the object field at the specified offset in the
	   * supplied object to the given value, with volatile store semantics.
	   * 设置obj对象中offset偏移地址对应的object型field的值为指定值。支持volatile store语义
	   * 
	   * @param obj the object containing the field to modify.
	   *    包含需要修改field的对象
	   * @param offset the offset of the object field within <code>obj</code>.
	   *     <code>obj</code>中object型field的偏移量
	   * @param value the new value of the field.
	   *       field将被设置的新值
	   * @see #putObject(Object,long,Object)
	   */
	  public native void putObjectVolatile(Object obj, long offset, Object value);
	  /***
	   * Sets the value of the object field at the specified offset in the
	   * supplied object to the given value.
	   * 设置obj对象中offset偏移地址对应的object型field的值为指定值。
	   * 
	   * @param obj the object containing the field to modify.
	   *    包含需要修改field的对象
	   * @param offset the offset of the object field within <code>obj</code>.
	   *     <code>obj</code>中object型field的偏移量
	   * @param value the new value of the field.
	   *       field将被设置的新值
	   * @see #putObjectVolatile(Object,long,Object)
	   */
	  public native void putObject(Object obj, long offset, Object value);
	  /***
	   * Retrieves the value of the object field at the specified offset in the
	   * supplied object with volatile load semantics.
	   * 获取obj对象中offset偏移地址对应的object型field的值,支持volatile load语义。
	   * 
	   * @param obj the object containing the field to read.
	   *    包含需要去读取的field的对象
	   * @param offset the offset of the object field within <code>obj</code>.
	   *       <code>obj</code>中object型field的偏移量
	   */
	  public native Object getObjectVolatile(Object obj, long offset);
	  /***
	   * Returns the offset of the first element for a given array class.
	   * To access elements of the array class, this value may be used along with
	   * with that returned by 
	   * <a href="#arrayIndexScale"><code>arrayIndexScale</code></a>,
	   * if non-zero.
	   * 获取给定数组中第一个元素的偏移地址。
	   * 为了存取数组中的元素，这个偏移地址与<a href="#arrayIndexScale"><code>arrayIndexScale
	   * </code></a>方法的非0返回值一起被使用。
	   * @param arrayClass the class for which the first element's address should
	   *                   be obtained.
	   *                   第一个元素地址被获取的class
	   * @return the offset of the first element of the array class.
	   *    数组第一个元素 的偏移地址
	   * @see arrayIndexScale(Class)
	   */
	  public native int arrayBaseOffset(Class arrayClass);
	  /***
	   * Returns the scale factor used for addressing elements of the supplied
	   * array class.  Where a suitable scale factor can not be returned (e.g.
	   * for primitive types), zero should be returned.  The returned value
	   * can be used with 
	   * <a href="#arrayBaseOffset"><code>arrayBaseOffset</code></a>
	   * to access elements of the class.
	   * 获取用户给定数组寻址的换算因子.一个合适的换算因子不能返回的时候(例如：基本类型),
	   * 返回0.这个返回值能够与<a href="#arrayBaseOffset"><code>arrayBaseOffset</code>
	   * </a>一起使用去存取这个数组class中的元素
	   * 
	   * @param arrayClass the class whose scale factor should be returned.
	   * @return the scale factor, or zero if not supported for this array class.
	   */
	  public native int arrayIndexScale(Class arrayClass);
	  
	  /***
	   * Releases the block on a thread created by 
	   * <a href="#park"><code>park</code></a>.  This method can also be used
	   * to terminate a blockage caused by a prior call to <code>park</code>.
	   * This operation is unsafe, as the thread must be guaranteed to be
	   * live.  This is true of Java, but not native code.
	   * 释放被<a href="#park"><code>park</code></a>创建的在一个线程上的阻塞.这个
	   * 方法也可以被使用来终止一个先前调用<code>park</code>导致的阻塞.
	   * 这个操作操作时不安全的,因此线程必须保证是活的.这是java代码不是native代码。
	   * @param thread the thread to unblock.
	   *           要解除阻塞的线程
	   */
	  public native void unpark(Thread thread);
	  /***
	   * Blocks the thread until a matching 
	   * <a href="#unpark"><code>unpark</code></a> occurs, the thread is
	   * interrupted or the optional timeout expires.  If an <code>unpark</code>
	   * call has already occurred, this also counts.  A timeout value of zero
	   * is defined as no timeout.  When <code>isAbsolute</code> is
	   * <code>true</code>, the timeout is in milliseconds relative to the
	   * epoch.  Otherwise, the value is the number of nanoseconds which must
	   * occur before timeout.  This call may also return spuriously (i.e.
	   * for no apparent reason).
	   * 阻塞一个线程直到<a href="#unpark"><code>unpark</code></a>出现、线程
	   * 被中断或者timeout时间到期。如果一个<code>unpark</code>调用已经出现了，
	   * 这里只计数。timeout为0表示永不过期.当<code>isAbsolute</code>为true时，
	   * timeout是相对于新纪元之后的毫秒。否则这个值就是超时前的纳秒数。这个方法执行时
	   * 也可能不合理地返回(没有具体原因)
	   * 
	   * @param isAbsolute true if the timeout is specified in milliseconds from
	   *                   the epoch.
	   *                   如果为true timeout的值是一个相对于新纪元之后的毫秒数
	   * @param time either the number of nanoseconds to wait, or a time in
	   *             milliseconds from the epoch to wait for.
	   *             可以是一个要等待的纳秒数，或者是一个相对于新纪元之后的毫秒数直到
	   *             到达这个时间点
	   */
	  public native void park(boolean isAbsolute, long time);
	}


针对Unsafe类这里不再过多赘述了，实际生产开发中建议避免使用。之所以会放在这里花费部分篇幅来分析它，主要是因为在后面的CAS、AQS以及JUC相关源码实现中无法避开它。


### 二、CAS ###

Compare And Swap(CAS),即比较并交换，设计并发算法时常用到的一种技术，java.util.concurrent包完全建立在CAS之上，没有CAS也就没有JUC，可见CAS的重要性。

CAS 操作包含三个操作数 —— 内存位置（V）、预期原值（A）和新值(B)。***如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。***CAS 有效地说明了“我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。” Java并发包(java.util.concurrent)中大量使用了CAS操作,涉及到并发的地方都调用了。

CAS也是通过Unsafe实现的，比如这三个方法：

	public final native boolean compareAndSwapObject(Object paramObject1, long paramLong, Object paramObject2, Object paramObject3);
	
	public final native boolean compareAndSwapInt(Object paramObject, long paramLong, int paramInt1, int paramInt2);
	
	public final native boolean compareAndSwapLong(Object paramObject, long paramLong1, long paramLong2, long paramLong3);

CAS的缺点：

CAS虽然很高效的解决原子操作，但是CAS仍然存在三大问题——ABA问题，循环时间长开销大和只能保证一个共享变量的原子操作：



- 因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了（大部分情况下ABA问题并不会影响程序并发的正确性）。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

 

- 循环时间长开销大。自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。



- 只能保证一个共享变量的原子操作。当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。

### 三、AQS ###


![](https://i.imgur.com/wpCmhBa.png)

AQS，AbstractQueuedSynchronizer，即队列同步器。在锁框架中，AbstractQueuedSynchronizer抽象类可以毫不夸张的说，占据着核心地位，它提供了一个基于FIFO队列，可以用于构建锁或者其他相关同步组件的核心基础框架（如CountDownLatch/FutureTask/ReentrantLock/Semaphore/RenntrantReadWriteLock等）），所以很有必要好好分析（以下以JDK1.8为例）。

AQS内封装了实现同步器时涉及的大量细节问题，例如获取同步状态、同步队列。AQS的主要使用方式是继承，子类通过继承同步器并实现它的抽象方法来管理同步状态。继承AQS来构建同步器可以带来很多好处。它不仅能够极大地减少实现工作，而且也不必处理在多个位置上发生的竞争问题。

![](http://upload-images.jianshu.io/upload_images/1603753-d4573009d6b2c8aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)


AQS的实现原理相对还是比较复杂的，简而言之就是AQS内部维护了一个同步状态（private volatile int state）和一个线程等待队列(双向链表实现)，同步状态的值默认为 0，当线程尝试获得锁的时候，需要判断同步状态的值是否为 0，如果为 0 则修改状态的值 （通常为 + 1，也可以是别的数，释放锁的时候对应上就行）线程即获得锁，否则进入队列等待（会被封装成一个Node），释放锁的时候将状态的值修改为0 （如果实现的是可重入锁需要递减该值，直到为 0 方真正释放），此时队列内等待的线程可以继续尝试获取该锁。

有了前面Unsafe和CAS的基础下面可以浅析一下AQS了，因为AQS里大量的使用着Unsafe与CAS ，如果没有前面铺垫，理解起来还是比较费劲的。下面让我们具体来分析一下AQS内部实现的源码机制。




#### （一）、底层数据结构 ####


分析类，首先就要分析底层采用了何种数据结构，抓住核心点进行分析，经过分析可知，AbstractQueuedSynchronizer类的数据结构如下：

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160403170136176-573839888.png)

说明：AbstractQueuedSynchronizer类底层的数据结构是使用双向链表，是队列的一种实现，故也可看成是队列，其中Sync queue，即同步队列，是双向链表，包括head结点和tail结点，head结点主要用作后续的调度。而Condition queue不是必须的，其是一个单向链表，只有当使用Condition时，才会存在此单向链表。并且可能会有多个Condition queue。


#### （二）、AbstractQueuedSynchronizer源码分析 ####

#### 2.1 类的继承关系 ####

	public abstract class AbstractQueuedSynchronizer
	    extends AbstractOwnableSynchronizer
	    implements java.io.Serializable

说明：从类继承关系可知，AbstractQueuedSynchronizer继承自AbstractOwnableSynchronizer抽象类，并且实现了Serializable接口，可以进行序列化。而AbstractOwnableSynchronizer抽象类的源码如下：

	public abstract class AbstractOwnableSynchronizer
	    implements java.io.Serializable {
	    
	    // 版本序列号
	    private static final long serialVersionUID = 3737899427754241961L;
	    // 构造函数
	    protected AbstractOwnableSynchronizer() { }
	    // 独占模式下的线程
	    private transient Thread exclusiveOwnerThread;
	    
	    // 设置独占线程 
	    protected final void setExclusiveOwnerThread(Thread thread) {
	        exclusiveOwnerThread = thread;
	    }
	    
	    // 获取独占线程 
	    protected final Thread getExclusiveOwnerThread() {
	        return exclusiveOwnerThread;
	    }
	}


说明：AbstractOwnableSynchronizer抽象类中，可以设置独占资源线程和获取独占资源线程。分别为setExclusiveOwnerThread与getExclusiveOwnerThread方法，这两个方法会被子类调用。

#### 2.2 类的内部类 ####

　AbstractQueuedSynchronizer类有两个内部类，分别为Node类与ConditionObject类。下面分别做介绍：

**1.Node类**

	static final class Node {
	        // 模式，分为共享与独占
	        // 共享模式
	        static final Node SHARED = new Node();
	        // 独占模式
	        static final Node EXCLUSIVE = null;        
	        // 结点状态
	        // CANCELLED，值为1，表示当前的线程被取消
	        // SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark
	        // CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中
	        // PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行
	        // 值为0，表示当前节点在sync队列中，等待着获取锁
	        static final int CANCELLED =  1;
	        static final int SIGNAL    = -1;
	        static final int CONDITION = -2;
	        static final int PROPAGATE = -3;        
	
	        // 结点状态
	        volatile int waitStatus;        
	        // 前驱结点
	        volatile Node prev;    
	        // 后继结点
	        volatile Node next;        
	        // 结点所对应的线程
	        volatile Thread thread;        
	        // 下一个等待者
	        Node nextWaiter;
	        
	        // 结点是否在共享模式下等待
	        final boolean isShared() {
	            return nextWaiter == SHARED;
	        }
	        
	        // 获取前驱结点，若前驱结点为空，抛出异常
	        final Node predecessor() throws NullPointerException {
	            // 保存前驱结点
	            Node p = prev; 
	            if (p == null) // 前驱结点为空，抛出异常
	                throw new NullPointerException();
	            else // 前驱结点不为空，返回
	                return p;
	        }
	        
	        // 无参构造函数
	        Node() {    // Used to establish initial head or SHARED marker
	        }
	        
	        // 构造函数
	         Node(Thread thread, Node mode) {    // Used by addWaiter
	            this.nextWaiter = mode;
	            this.thread = thread;
	        }
	        
	        // 构造函数
	        Node(Thread thread, int waitStatus) { // Used by Condition
	            this.waitStatus = waitStatus;
	            this.thread = thread;
	        }
	    }

说明：每个线程被阻塞的线程都会被封装成一个Node结点，放入队列。每个节点包含了一个Thread类型的引用，并且每个节点都存在一个状态，具体状态如下。

　　① CANCELLED，值为1，表示当前的线程被取消。（因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；）

　　② SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，需要进行unpark操作。（后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行）

　　③ CONDITION，值为-2，表示当前节点在等待condition，也就是在condition queue中。（节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，该节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中）

　　④ PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行。（表示下一次共享式同步状态获取将会无条件地传播下去）

　　⑤ 值为0，表示当前节点在sync queue中，等待着获取锁。


![](http://upload-images.jianshu.io/upload_images/1603753-b0f4081ae246c129.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/620)


**2.ConditionObject类**

	　// 内部类
	    public class ConditionObject implements Condition, java.io.Serializable {
	        // 版本号
	        private static final long serialVersionUID = 1173984872572414699L;
	        /** First node of condition queue. */
	        // condition队列的头结点
	        private transient Node firstWaiter;
	        /** Last node of condition queue. */
	        // condition队列的尾结点
	        private transient Node lastWaiter;
	
	        /**
	         * Creates a new {@code ConditionObject} instance.
	         */
	        // 构造函数
	        public ConditionObject() { }
	
	        // Internal methods
	
	        /**
	         * Adds a new waiter to wait queue.
	         * @return its new wait node
	         */
	        // 添加新的waiter到wait队列
	        private Node addConditionWaiter() {
	            // 保存尾结点
	            Node t = lastWaiter;
	            // If lastWaiter is cancelled, clean out.
	            if (t != null && t.waitStatus != Node.CONDITION) { // 尾结点不为空，并且尾结点的状态不为CONDITION
	                // 清除状态为CONDITION的结点
	                unlinkCancelledWaiters(); 
	                // 将最后一个结点重新赋值给t
	                t = lastWaiter;
	            }
	            // 新建一个结点
	            Node node = new Node(Thread.currentThread(), Node.CONDITION);
	            if (t == null) // 尾结点为空
	                // 设置condition队列的头结点
	                firstWaiter = node;
	            else // 尾结点不为空
	                // 设置为节点的nextWaiter域为node结点
	                t.nextWaiter = node;
	            // 更新condition队列的尾结点
	            lastWaiter = node;
	            return node;
	        }
	
	        /**
	         * Removes and transfers nodes until hit non-cancelled one or
	         * null. Split out from signal in part to encourage compilers
	         * to inline the case of no waiters.
	         * @param first (non-null) the first node on condition queue
	         */
	        private void doSignal(Node first) {
	            // 循环
	            do {
	                if ( (firstWaiter = first.nextWaiter) == null) // 该节点的nextWaiter为空
	                    // 设置尾结点为空
	                    lastWaiter = null;
	                // 设置first结点的nextWaiter域
	                first.nextWaiter = null;
	            } while (!transferForSignal(first) &&
	                     (first = firstWaiter) != null); // 将结点从condition队列转移到sync队列失败并且condition队列中的头结点不为空，一直循环
	        }
	
	        /**
	         * Removes and transfers all nodes.
	         * @param first (non-null) the first node on condition queue
	         */
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
	
	        /**
	         * Unlinks cancelled waiter nodes from condition queue.
	         * Called only while holding lock. This is called when
	         * cancellation occurred during condition wait, and upon
	         * insertion of a new waiter when lastWaiter is seen to have
	         * been cancelled. This method is needed to avoid garbage
	         * retention in the absence of signals. So even though it may
	         * require a full traversal, it comes into play only when
	         * timeouts or cancellations occur in the absence of
	         * signals. It traverses all nodes rather than stopping at a
	         * particular target to unlink all pointers to garbage nodes
	         * without requiring many re-traversals during cancellation
	         * storms.
	         */
	        // 从condition队列中清除状态为CANCEL的结点
	        private void unlinkCancelledWaiters() {
	            // 保存condition队列头结点
	            Node t = firstWaiter;
	            Node trail = null;
	            while (t != null) { // t不为空
	                // 下一个结点
	                Node next = t.nextWaiter;
	                if (t.waitStatus != Node.CONDITION) { // t结点的状态不为CONDTION状态
	                    // 设置t节点的额nextWaiter域为空
	                    t.nextWaiter = null;
	                    if (trail == null) // trail为空
	                        // 重新设置condition队列的头结点
	                        firstWaiter = next;
	                    else // trail不为空
	                        // 设置trail结点的nextWaiter域为next结点
	                        trail.nextWaiter = next;
	                    if (next == null) // next结点为空
	                        // 设置condition队列的尾结点
	                        lastWaiter = trail;
	                }
	                else // t结点的状态为CONDTION状态
	                    // 设置trail结点
	                    trail = t;
	                // 设置t结点
	                t = next;
	            }
	        }
	
	        // public methods
	
	        /**
	         * Moves the longest-waiting thread, if one exists, from the
	         * wait queue for this condition to the wait queue for the
	         * owning lock.
	         *
	         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
	         *         returns {@code false}
	         */
	        // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
	        public final void signal() {
	            if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
	                throw new IllegalMonitorStateException();
	            // 保存condition队列头结点
	            Node first = firstWaiter;
	            if (first != null) // 头结点不为空
	                // 唤醒一个等待线程
	                doSignal(first);
	        }
	
	        /**
	         * Moves all threads from the wait queue for this condition to
	         * the wait queue for the owning lock.
	         *
	         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
	         *         returns {@code false}
	         */
	        // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
	        public final void signalAll() {
	            if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
	                throw new IllegalMonitorStateException();
	            // 保存condition队列头结点
	            Node first = firstWaiter;
	            if (first != null) // 头结点不为空
	                // 唤醒所有等待线程
	                doSignalAll(first);
	        }
	
	        /**
	         * Implements uninterruptible condition wait.
	         * <ol>
	         * <li> Save lock state returned by {@link #getState}.
	         * <li> Invoke {@link #release} with saved state as argument,
	         *      throwing IllegalMonitorStateException if it fails.
	         * <li> Block until signalled.
	         * <li> Reacquire by invoking specialized version of
	         *      {@link #acquire} with saved state as argument.
	         * </ol>
	         */
	        // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
	        public final void awaitUninterruptibly() {
	            // 添加一个结点到等待队列
	            Node node = addConditionWaiter();
	            // 获取释放的状态
	            int savedState = fullyRelease(node);
	            boolean interrupted = false;
	            while (!isOnSyncQueue(node)) { // 
	                // 阻塞当前线程
	                LockSupport.park(this);
	                if (Thread.interrupted()) // 当前线程被中断
	                    // 设置interrupted状态
	                    interrupted = true; 
	            }
	            if (acquireQueued(node, savedState) || interrupted) // 
	                selfInterrupt();
	        }
	
	        /*
	         * For interruptible waits, we need to track whether to throw
	         * InterruptedException, if interrupted while blocked on
	         * condition, versus reinterrupt current thread, if
	         * interrupted while blocked waiting to re-acquire.
	         */
	
	        /** Mode meaning to reinterrupt on exit from wait */
	        private static final int REINTERRUPT =  1;
	        /** Mode meaning to throw InterruptedException on exit from wait */
	        private static final int THROW_IE    = -1;
	
	        /**
	         * Checks for interrupt, returning THROW_IE if interrupted
	         * before signalled, REINTERRUPT if after signalled, or
	         * 0 if not interrupted.
	         */
	        private int checkInterruptWhileWaiting(Node node) {
	            return Thread.interrupted() ?
	                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
	                0; 
	        }
	
	        /**
	         * Throws InterruptedException, reinterrupts current thread, or
	         * does nothing, depending on mode.
	         */
	        private void reportInterruptAfterWait(int interruptMode)
	            throws InterruptedException {
	            if (interruptMode == THROW_IE)
	                throw new InterruptedException();
	            else if (interruptMode == REINTERRUPT)
	                selfInterrupt();
	        }
	
	        /**
	         * Implements interruptible condition wait.
	         * <ol>
	         * <li> If current thread is interrupted, throw InterruptedException.
	         * <li> Save lock state returned by {@link #getState}.
	         * <li> Invoke {@link #release} with saved state as argument,
	         *      throwing IllegalMonitorStateException if it fails.
	         * <li> Block until signalled or interrupted.
	         * <li> Reacquire by invoking specialized version of
	         *      {@link #acquire} with saved state as argument.
	         * <li> If interrupted while blocked in step 4, throw InterruptedException.
	         * </ol>
	         */
	        // // 等待，当前线程在接到信号或被中断之前一直处于等待状态
	        public final void await() throws InterruptedException {
	            if (Thread.interrupted()) // 当前线程被中断，抛出异常
	                throw new InterruptedException();
	            // 在wait队列上添加一个结点
	            Node node = addConditionWaiter();
	            // 
	            int savedState = fullyRelease(node);
	            int interruptMode = 0;
	            while (!isOnSyncQueue(node)) {
	                // 阻塞当前线程
	                LockSupport.park(this);
	                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 检查结点等待时的中断类型
	                    break;
	            }
	            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	                interruptMode = REINTERRUPT;
	            if (node.nextWaiter != null) // clean up if cancelled
	                unlinkCancelledWaiters();
	            if (interruptMode != 0)
	                reportInterruptAfterWait(interruptMode);
	        }
	
	        /**
	         * Implements timed condition wait.
	         * <ol>
	         * <li> If current thread is interrupted, throw InterruptedException.
	         * <li> Save lock state returned by {@link #getState}.
	         * <li> Invoke {@link #release} with saved state as argument,
	         *      throwing IllegalMonitorStateException if it fails.
	         * <li> Block until signalled, interrupted, or timed out.
	         * <li> Reacquire by invoking specialized version of
	         *      {@link #acquire} with saved state as argument.
	         * <li> If interrupted while blocked in step 4, throw InterruptedException.
	         * </ol>
	         */
	        // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
	        public final long awaitNanos(long nanosTimeout)
	                throws InterruptedException {
	            if (Thread.interrupted())
	                throw new InterruptedException();
	            Node node = addConditionWaiter();
	            int savedState = fullyRelease(node);
	            final long deadline = System.nanoTime() + nanosTimeout;
	            int interruptMode = 0;
	            while (!isOnSyncQueue(node)) {
	                if (nanosTimeout <= 0L) {
	                    transferAfterCancelledWait(node);
	                    break;
	                }
	                if (nanosTimeout >= spinForTimeoutThreshold)
	                    LockSupport.parkNanos(this, nanosTimeout);
	                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
	                    break;
	                nanosTimeout = deadline - System.nanoTime();
	            }
	            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	                interruptMode = REINTERRUPT;
	            if (node.nextWaiter != null)
	                unlinkCancelledWaiters();
	            if (interruptMode != 0)
	                reportInterruptAfterWait(interruptMode);
	            return deadline - System.nanoTime();
	        }
	
	        /**
	         * Implements absolute timed condition wait.
	         * <ol>
	         * <li> If current thread is interrupted, throw InterruptedException.
	         * <li> Save lock state returned by {@link #getState}.
	         * <li> Invoke {@link #release} with saved state as argument,
	         *      throwing IllegalMonitorStateException if it fails.
	         * <li> Block until signalled, interrupted, or timed out.
	         * <li> Reacquire by invoking specialized version of
	         *      {@link #acquire} with saved state as argument.
	         * <li> If interrupted while blocked in step 4, throw InterruptedException.
	         * <li> If timed out while blocked in step 4, return false, else true.
	         * </ol>
	         */
	        // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
	        public final boolean awaitUntil(Date deadline)
	                throws InterruptedException {
	            long abstime = deadline.getTime();
	            if (Thread.interrupted())
	                throw new InterruptedException();
	            Node node = addConditionWaiter();
	            int savedState = fullyRelease(node);
	            boolean timedout = false;
	            int interruptMode = 0;
	            while (!isOnSyncQueue(node)) {
	                if (System.currentTimeMillis() > abstime) {
	                    timedout = transferAfterCancelledWait(node);
	                    break;
	                }
	                LockSupport.parkUntil(this, abstime);
	                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
	                    break;
	            }
	            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	                interruptMode = REINTERRUPT;
	            if (node.nextWaiter != null)
	                unlinkCancelledWaiters();
	            if (interruptMode != 0)
	                reportInterruptAfterWait(interruptMode);
	            return !timedout;
	        }
	
	        /**
	         * Implements timed condition wait.
	         * <ol>
	         * <li> If current thread is interrupted, throw InterruptedException.
	         * <li> Save lock state returned by {@link #getState}.
	         * <li> Invoke {@link #release} with saved state as argument,
	         *      throwing IllegalMonitorStateException if it fails.
	         * <li> Block until signalled, interrupted, or timed out.
	         * <li> Reacquire by invoking specialized version of
	         *      {@link #acquire} with saved state as argument.
	         * <li> If interrupted while blocked in step 4, throw InterruptedException.
	         * <li> If timed out while blocked in step 4, return false, else true.
	         * </ol>
	         */
	        // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于：awaitNanos(unit.toNanos(time)) > 0
	        public final boolean await(long time, TimeUnit unit)
	                throws InterruptedException {
	            long nanosTimeout = unit.toNanos(time);
	            if (Thread.interrupted())
	                throw new InterruptedException();
	            Node node = addConditionWaiter();
	            int savedState = fullyRelease(node);
	            final long deadline = System.nanoTime() + nanosTimeout;
	            boolean timedout = false;
	            int interruptMode = 0;
	            while (!isOnSyncQueue(node)) {
	                if (nanosTimeout <= 0L) {
	                    timedout = transferAfterCancelledWait(node);
	                    break;
	                }
	                if (nanosTimeout >= spinForTimeoutThreshold)
	                    LockSupport.parkNanos(this, nanosTimeout);
	                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
	                    break;
	                nanosTimeout = deadline - System.nanoTime();
	            }
	            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
	                interruptMode = REINTERRUPT;
	            if (node.nextWaiter != null)
	                unlinkCancelledWaiters();
	            if (interruptMode != 0)
	                reportInterruptAfterWait(interruptMode);
	            return !timedout;
	        }
	
	        //  support for instrumentation
	
	        /**
	         * Returns true if this condition was created by the given
	         * synchronization object.
	         *
	         * @return {@code true} if owned
	         */
	        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
	            return sync == AbstractQueuedSynchronizer.this;
	        }
	
	        /**
	         * Queries whether any threads are waiting on this condition.
	         * Implements {@link AbstractQueuedSynchronizer#hasWaiters(ConditionObject)}.
	         *
	         * @return {@code true} if there are any waiting threads
	         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
	         *         returns {@code false}
	         */
	        //  查询是否有正在等待此条件的任何线程
	        protected final boolean hasWaiters() {
	            if (!isHeldExclusively())
	                throw new IllegalMonitorStateException();
	            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
	                if (w.waitStatus == Node.CONDITION)
	                    return true;
	            }
	            return false;
	        }
	
	        /**
	         * Returns an estimate of the number of threads waiting on
	         * this condition.
	         * Implements {@link AbstractQueuedSynchronizer#getWaitQueueLength(ConditionObject)}.
	         *
	         * @return the estimated number of waiting threads
	         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
	         *         returns {@code false}
	         */
	        // 返回正在等待此条件的线程数估计值
	        protected final int getWaitQueueLength() {
	            if (!isHeldExclusively())
	                throw new IllegalMonitorStateException();
	            int n = 0;
	            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
	                if (w.waitStatus == Node.CONDITION)
	                    ++n;
	            }
	            return n;
	        }
	
	        /**
	         * Returns a collection containing those threads that may be
	         * waiting on this Condition.
	         * Implements {@link AbstractQueuedSynchronizer#getWaitingThreads(ConditionObject)}.
	         *
	         * @return the collection of threads
	         * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
	         *         returns {@code false}
	         */
	        // 返回包含那些可能正在等待此条件的线程集合
	        protected final Collection<Thread> getWaitingThreads() {
	            if (!isHeldExclusively())
	                throw new IllegalMonitorStateException();
	            ArrayList<Thread> list = new ArrayList<Thread>();
	            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
	                if (w.waitStatus == Node.CONDITION) {
	                    Thread t = w.thread;
	                    if (t != null)
	                        list.add(t);
	                }
	            }
	            return list;
	        }
	    }

说明：此类实现了Condition接口，Condition接口定义了条件操作规范，具体如下　：

	public interface Condition {
	
	    // 等待，当前线程在接到信号或被中断之前一直处于等待状态
	    void await() throws InterruptedException;
	    
	    // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
	    void awaitUninterruptibly();
	    
	    //等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
	    long awaitNanos(long nanosTimeout) throws InterruptedException;
	    
	    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于：awaitNanos(unit.toNanos(time)) > 0
	    boolean await(long time, TimeUnit unit) throws InterruptedException;
	    
	    // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
	    boolean awaitUntil(Date deadline) throws InterruptedException;
	    
	    // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
	    void signal();
	    
	    // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
	    void signalAll();
	}

说明：Condition接口中定义了await、signal函数，用来等待条件、释放条件。之后会详细分析CondtionObject的源码。


#### 2.3 类的属性 ####



	public abstract class AbstractQueuedSynchronizer
	    extends AbstractOwnableSynchronizer
	    implements java.io.Serializable {    
	    // 版本号
	    private static final long serialVersionUID = 7373984972572414691L;    
	    // 头结点
	    private transient volatile Node head;    
	    // 尾结点
	    private transient volatile Node tail;    
	    // 状态
	    private volatile int state;    
	    // 自旋时间
	    static final long spinForTimeoutThreshold = 1000L;
	    
	    // Unsafe类实例
	    private static final Unsafe unsafe = Unsafe.getUnsafe();
	    // state内存偏移地址
	    private static final long stateOffset;
	    // head内存偏移地址
	    private static final long headOffset;
	    // state内存偏移地址
	    private static final long tailOffset;
	    // tail内存偏移地址
	    private static final long waitStatusOffset;
	    // next内存偏移地址
	    private static final long nextOffset;
	    // 静态初始化块
	    static {
	        try {
	            stateOffset = unsafe.objectFieldOffset
	                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
	            headOffset = unsafe.objectFieldOffset
	                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
	            tailOffset = unsafe.objectFieldOffset
	                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
	            waitStatusOffset = unsafe.objectFieldOffset
	                (Node.class.getDeclaredField("waitStatus"));
	            nextOffset = unsafe.objectFieldOffset
	                (Node.class.getDeclaredField("next"));
	
	        } catch (Exception ex) { throw new Error(ex); }
	    }

		//无参构造函数，供子类调用
		protected AbstractQueuedSynchronizer() { }    
	}




　说明：属性中包含了头结点head，尾结点tail，状态state、自旋时间spinForTimeoutThreshold，还有AbstractQueuedSynchronizer抽象的属性在内存中的偏移地址，通过该偏移地址，可以获取和设置该属性的值，同时还包括一个静态初始化块，用于加载内存偏移地址。



#### 2.4 类的核心函数 ####


AQS主要提供了如下一些核心方法：

- getState()：返回同步状态的当前值；
- setState(int newState)：设置当前同步状态；
- compareAndSetState(int expect, int update)：使用CAS设置当前状态，该方法能够保证状态设置的原子性；
- tryAcquire(int arg)：独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态；
- tryRelease(int arg)：独占式释放同步状态；
- tryAcquireShared(int arg)：共享式获取同步状态，返回值大于等于0则表示获取成功，否则获取失败；
- tryReleaseShared(int arg)：共享式释放同步状态；
- isHeldExclusively()：当前同步器是否在独占式模式下被线程占用，一般该方法表示是否被当前线程所独占；
- acquire(int arg)：独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用可重写的tryAcquire(int arg)方法；
- acquireInterruptibly(int arg)：与acquire(int arg)相同，但是该方法响应中断，当前线程为获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException异常并返回；
- tryAcquireNanos(int arg,long nanos)：超时获取同步状态，如果当前线程在nanos时间内没有获取到同步状态，那么将会返回false，已经获取则返回true；
- acquireShared(int arg)：共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
- acquireSharedInterruptibly(int arg)：共享式获取同步状态，响应中断；
- tryAcquireSharedNanos(int arg, long nanosTimeout)：共享式获取同步状态，增加超时限制；
- release(int arg)：独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒；
- releaseShared(int arg)：共享式释放同步状态；

#### （一）、独占式 ####

**1. acquire函数**

该函数以独占模式获取(资源)，忽略中断，即线程在aquire过程中，中断此线程是无效的。源码如下：

	public final void acquire(int arg) {
	    if (!tryAcquire(arg) &&
	        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
	         　　selfInterrupt();
	}
由上述源码可以知道，当一个线程调用acquire时，调用方法流程如下。

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160404112240734-653491814.png)

说明：

　　① 首先调用tryAcquire函数，调用此方法的线程会试图在独占模式下获取对象状态。此方法应该查询是否允许它在独占模式下获取对象状态，如果允许，则获取它。在AbstractQueuedSynchronizer源码中默认会抛出一个异常，即需要子类去重写此函数完成自己的逻辑。之后会进行分析。

　　② 若tryAcquire失败，则调用addWaiter函数，addWaiter函数完成的功能是将调用此方法的线程封装成为一个结点并放入Sync queue。

　　③ 调用acquireQueued函数，此函数完成的功能是Sync queue中的结点不断尝试获取资源，若成功，则返回true，否则，返回false。

　　（由于tryAcquire默认实现是抛出异常，所以此时，不进行分析，之后会结合一个例子进行分析。）

　　（1）addWaiter函数

	// 添加等待者
	    private Node addWaiter(Node mode) {
	        // 新生成一个结点，默认为独占模式
	        Node node = new Node(Thread.currentThread(), mode);
	        // Try the fast path of enq; backup to full enq on failure
	        // 保存尾结点
	        Node pred = tail;
	        if (pred != null) { // 尾结点不为空，即已经被初始化
	            // 将node结点的prev域连接到尾结点
	            node.prev = pred; 
	            if (compareAndSetTail(pred, node)) { // 比较pred是否为尾结点，是则将尾结点设置为node 
	                // 设置尾结点的next域为node
	                pred.next = node;
	                return node; // 返回新生成的结点
	            }
	        }
	        enq(node); // 尾结点为空(即还没有被初始化过)，或者是compareAndSetTail操作失败，则入队列
	        return node;
	    }

说明：addWaiter函数使用快速添加的方式往sync queue尾部添加结点，如果sync queue队列还没有初始化，则会使用enq方法插入队列中。


　　（2）enq函数

	// 入队列
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


　（3）acquireQueue函数

	// sync队列中的结点在独占且忽略中断的模式下获取(资源)
	    final boolean acquireQueued(final Node node, int arg) {
	        // 标志
	        boolean failed = true;
	        try {
	            // 中断标志
	            boolean interrupted = false;
	            for (;;) { // 无限循环
	                // 获取node节点的前驱结点
	                final Node p = node.predecessor(); 
	                if (p == head && tryAcquire(arg)) { // 前驱为头结点并且成功获得锁
	                    setHead(node); // 设置头结点
	                    p.next = null; // help GC
	                    failed = false; // 设置标志
	                    return interrupted; 
	                }
	                if (shouldParkAfterFailedAcquire(p, node) &&
	                    parkAndCheckInterrupt())
	                    interrupted = true;
	            }
	        } finally {
	            if (failed)
	                cancelAcquire(node);
	        }
	    }

说明：首先获取当前节点的前驱节点，如果前驱节点是头结点并且能够获取(资源)，代表该当前节点能够占有锁，设置头结点为当前节点，返回。否则，调用shouldParkAfterFailedAcquire和parkAndCheckInterrupt函数，首先，我们看shouldParkAfterFailedAcquire函数，代码如下:

	// 当获取(资源)失败后，检查并且更新结点状态
	    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	        // 获取前驱结点的状态
	        int ws = pred.waitStatus;
	        if (ws == Node.SIGNAL) // 状态为SIGNAL，为-1
	            /*
	             * This node has already set status asking a release
	             * to signal it, so it can safely park.
	             */
	            // 可以进行park操作
	            return true; 
	        if (ws > 0) { // 表示状态为CANCELLED，为1
	            /*
	             * Predecessor was cancelled. Skip over predecessors and
	             * indicate retry.
	             */
	            do {
	                node.prev = pred = pred.prev;
	            } while (pred.waitStatus > 0); // 找到pred结点前面最近的一个状态不为CANCELLED的结点
	            // 赋值pred结点的next域
	            pred.next = node; 
	        } else { // 为PROPAGATE -3 或者是0 表示无状态,(为CONDITION -2时，表示此节点在condition queue中) 
	            /*
	             * waitStatus must be 0 or PROPAGATE.  Indicate that we
	             * need a signal, but don't park yet.  Caller will need to
	             * retry to make sure it cannot acquire before parking.
	             */
	            // 比较并设置前驱结点的状态为SIGNAL
	            compareAndSetWaitStatus(pred, ws, Node.SIGNAL); 
	        }
	        // 不能进行park操作
	        return false;
	    }

说明：只有当该节点的前驱结点的状态为SIGNAL时，才可以对该结点所封装的线程进行park操作。否则，将不能进行park操作。再看parkAndCheckInterrupt函数，源码如下

	// 进行park操作并且返回该线程是否被中断
	    private final boolean parkAndCheckInterrupt() {
	        // 在许可可用之前禁用当前线程，并且设置了blocker
	        LockSupport.park(this);
	        return Thread.interrupted(); // 当前线程是否已被中断，并清除中断标记位
	    }

说明：parkAndCheckInterrupt函数里的逻辑是首先执行park操作，即禁用当前线程，然后返回该线程是否已经被中断。再看final块中的cancelAcquire函数，其源码如下

	// 取消继续获取(资源)
	    private void cancelAcquire(Node node) {
	        // Ignore if node doesn't exist
	        // node为空，返回
	        if (node == null)
	            return;
	        // 设置node结点的thread为空
	        node.thread = null;
	
	        // Skip cancelled predecessors
	        // 保存node的前驱结点
	        Node pred = node.prev;
	        while (pred.waitStatus > 0) // 找到node前驱结点中第一个状态小于0的结点，即不为CANCELLED状态的结点
	            node.prev = pred = pred.prev;
	
	        // predNext is the apparent node to unsplice. CASes below will
	        // fail if not, in which case, we lost race vs another cancel
	        // or signal, so no further action is necessary.
	        // 获取pred结点的下一个结点
	        Node predNext = pred.next;
	
	        // Can use unconditional write instead of CAS here.
	        // After this atomic step, other Nodes can skip past us.
	        // Before, we are free of interference from other threads.
	        // 设置node结点的状态为CANCELLED
	        node.waitStatus = Node.CANCELLED;
	
	        // If we are the tail, remove ourselves.
	        if (node == tail && compareAndSetTail(node, pred)) { // node结点为尾结点，则设置尾结点为pred结点
	            // 比较并设置pred结点的next节点为null
	            compareAndSetNext(pred, predNext, null); 
	        } else { // node结点不为尾结点，或者比较设置不成功
	            // If successor needs signal, try to set pred's next-link
	            // so it will get one. Otherwise wake it up to propagate.
	            int ws;
	            if (pred != head &&
	                ((ws = pred.waitStatus) == Node.SIGNAL ||
	                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
	                pred.thread != null) { // （pred结点不为头结点，并且pred结点的状态为SIGNAL）或者 
	                                    // pred结点状态小于等于0，并且比较并设置等待状态为SIGNAL成功，并且pred结点所封装的线程不为空
	                // 保存结点的后继
	                Node next = node.next;
	                if (next != null && next.waitStatus <= 0) // 后继不为空并且后继的状态小于等于0
	                    compareAndSetNext(pred, predNext, next); // 比较并设置pred.next = next;
	            } else {
	                unparkSuccessor(node); // 释放node的前一个结点
	            }
	
	            node.next = node; // help GC
	        }
	    }


说明：该函数完成的功能就是取消当前线程对资源的获取，即设置该结点的状态为CANCELLED，接着我们再看unparkSuccessor函数，源码如下

	// 释放后继结点
	    private void unparkSuccessor(Node node) {
	        /*
	         * If status is negative (i.e., possibly needing signal) try
	         * to clear in anticipation of signalling.  It is OK if this
	         * fails or if status is changed by waiting thread.
	         */
	        // 获取node结点的等待状态
	        int ws = node.waitStatus;
	        if (ws < 0) // 状态值小于0，为SIGNAL -1 或 CONDITION -2 或 PROPAGATE -3
	            // 比较并且设置结点等待状态，设置为0
	            compareAndSetWaitStatus(node, ws, 0);
	
	        /*
	         * Thread to unpark is held in successor, which is normally
	         * just the next node.  But if cancelled or apparently null,
	         * traverse backwards from tail to find the actual
	         * non-cancelled successor.
	         */
	        // 获取node节点的下一个结点
	        Node s = node.next;
	        if (s == null || s.waitStatus > 0) { // 下一个结点为空或者下一个节点的等待状态大于0，即为CANCELLED
	            // s赋值为空
	            s = null; 
	            // 从尾结点开始从后往前开始遍历
	            for (Node t = tail; t != null && t != node; t = t.prev)
	                if (t.waitStatus <= 0) // 找到等待状态小于等于0的结点，找到最前的状态小于等于0的结点
	                    // 保存结点
	                    s = t;
	        }
	        if (s != null) // 该结点不为为空，释放许可
	            LockSupport.unpark(s.thread);
	    }


说明：该函数的作用就是为了释放node节点的后继结点。对于cancelAcquire与unparkSuccessor函数，如下示意图可以清晰的表示。

![](http://images2015.cnblogs.com/blog/616953/201604/616953-20160406221022500-1247417874.png)

说明：其中node为参数，在执行完cancelAcquire函数后的效果就是unpark了s结点所包含的t4线程。

　　现在，再来看acquireQueued函数的整个的逻辑。逻辑如下

　　① 判断结点的前驱是否为head并且是否成功获取(资源)。

　　② 若步骤①均满足，则设置结点为head，之后会判断是否finally模块，然后返回。

　　③ 若步骤①不满足，则判断是否需要park当前线程，是否需要park当前线程的逻辑是判断结点的前驱结点的状态是否为SIGNAL，若是，则park当前结点，否则，不进行park操作。

　　④ 若park了当前线程，之后某个线程对本线程unpark后，并且本线程也获得机会运行。那么，将会继续进行步骤①的判断。


acquire方法不响应中断，但AQS也提供了响应中断、超时的方法，分别是：acquireInterruptibly(int arg)、tryAcquireNanos(int arg,long nanos)，这里就不做解释了。




**2. release函数**

	public final boolean release(int arg) {
	        if (tryRelease(arg)) { // 释放成功
	            // 保存头结点
	            Node h = head; 
	            if (h != null && h.waitStatus != 0) // 头结点不为空并且头结点状态不为0
	                unparkSuccessor(h); //释放头结点的后继结点
	            return true;
	        }
	        return false;
	    }
	

说明：其中，tryRelease的默认实现是抛出异常，需要具体的子类实现，如果tryRelease成功，那么如果头结点不为空并且头结点的状态不为0，则释放头结点的后继结点，unparkSuccessor函数已经分析过，不再累赘。






#### （二）、共享式 ####

共享式与独占式的最主要区别在于同一时刻独占式只能有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。例如读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。

**1. acquireShared(int arg)函数**

	public final void acquireShared(int arg) {
	        if (tryAcquireShared(arg) < 0)
	            //获取失败，自旋获取同步状态
	            doAcquireShared(arg);
	    }

方法首先是调用tryAcquireShared(int arg)方法尝试获取同步状态，如果获取失败则调用doAcquireShared(int arg)自旋方式获取同步状态，共享式获取同步状态的标志是返回 >= 0 的值表示获取成功。自选式获取同步状态如下：

	   private void doAcquireShared(int arg) {
	        /共享式节点
	        final Node node = addWaiter(Node.SHARED);
	        boolean failed = true;
	        try {
	            boolean interrupted = false;
	            for (;;) {
	                //前驱节点
	                final Node p = node.predecessor();
	                //如果其前驱节点，获取同步状态
	                if (p == head) {
	                    //尝试获取同步
	                    int r = tryAcquireShared(arg);
	                    if (r >= 0) {
	                        setHeadAndPropagate(node, r);
	                        p.next = null; // help GC
	                        if (interrupted)
	                            selfInterrupt();
	                        failed = false;
	                        return;
	                    }
	                }
	                if (shouldParkAfterFailedAcquire(p, node) &&
	                        parkAndCheckInterrupt())
	                    interrupted = true;
	            }
	        } finally {
	            if (failed)
	                cancelAcquire(node);
	        }
	    }


tryAcquireShared(int arg)方法尝试获取同步状态，返回值为int，当其 >= 0 时，表示能够获取到同步状态，这个时候就可以从自旋过程中退出。（这里tryAcquireShared的默认实现是抛出一个UnsupportedOperationException异常，真正的实现逻辑交给具体子类实现）

	protected int tryAcquireShared(int arg) {
	        throw new UnsupportedOperationException();
	    }


acquireShared整个步骤如下：

1. 尝试获取共享状态；
调用tryAcquireShared来获取共享状态，该方法是非阻塞的，如果获取成功则立刻返回，也就表示获取共享锁成功。
2. 获取失败进入sync队列；
在获取共享状态失败后，当前时刻有可能是独占锁被其他线程所把持，那么将当前线程构造成为节点（共享模式）加入到sync队列中。
3. 循环内判断退出队列条件；
如果当前节点的前驱节点是头结点并且获取共享状态成功，这里和独占锁acquire的退出队列条件类似。
4. 获取共享状态成功；
在退出队列的条件上，和独占锁之间的主要区别在于获取共享状态成功之后的行为，而如果共享状态获取成功之后会判断后继节点是否是共享模式，如果是共享模式，那么就直接对其进行唤醒操作，也就是同时激发多个线程并发的运行。
5. 获取共享状态失败。

![](http://img1.tbcdn.cn/L1/461/1/t_9683_1379329217_1542967524.png)

（绿色表示共享节点，它们之间的通知和唤醒操作是在前驱节点获取状态时就进行的，红色表示独占节点，它的被唤醒必须取决于前驱节点的释放，也就是release操作）

acquireShared(int arg)方法不响应中断，与独占式相似，AQS也提供了响应中断、超时的方法，分别是：acquireSharedInterruptibly(int arg)、tryAcquireSharedNanos(int arg,long nanos)，这里就不做解释了。



**2. releaseShared(int arg)函数**

	 public final boolean releaseShared(int arg) {
	        if (tryReleaseShared(arg)) {
	            doReleaseShared();
	            return true;
	        }
	        return false;
	    }

	 protected boolean tryReleaseShared(int arg) {
	        throw new UnsupportedOperationException();
	    }

	private void doReleaseShared() {
	        
	        for (;;) {
	            Node h = head;
	            if (h != null && h != tail) {
	                int ws = h.waitStatus;
	                if (ws == Node.SIGNAL) {
	                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
	                        continue;            // loop to recheck cases
	                    unparkSuccessor(h);
	                }
	                else if (ws == 0 &&
	                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
	                    continue;                // loop on failed CAS
	            }
	            if (h == head)                   // loop if head changed
	                break;
	        }
	    }


此方法的流程也比较简单，一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，但有一点稍微需要注意：独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以自定义同步器可以根据需要决定tryReleaseShared()的返回值。







#### 总结 ####

I、对于AbstractQueuedSynchronizer的分析，最核心的就是sync queue的分析。

　　① 每一个结点都是由前一个结点唤醒

　　② 当结点发现前驱结点是head并且尝试获取成功，则会轮到该线程运行。

　　③ condition queue中的结点向sync queue中转移是通过signal操作完成的。

　　④ 当结点的状态为SIGNAL时，表示后面的结点需要运行。

II、AQS 的设计基于模板方法模式，其内部已经实现的方法基本上分为 3 类： 独占式获取与释放锁、共享式获取与释放锁以及查询等待队列中的等待线程情况。

	void acquire(int)//独占式获取锁，如果当前线程成功获取锁，那么方法就返回，否则会将当前线程放入同步队列等待。该方法会调用子类覆写的 tryAcquire(int arg) 方法判断是否可以获得锁
	void acquireInterruptibly(int)//和 acquire(int) 相同，但是该方法响应中断，当线程在同步队列中等待时，如果线程被中断，会抛出 InterruptedException 异常并返回。
	boolean tryAcquireNanos(int, long)//在 acquireInterruptibly(int) 基础上添加了超时控制，同时支持中断和超时，当在指定时间内没有获得锁时，会返回 false，获取到了返回 true
	void acquireShared(int)//共享式获得锁，如果成功获得锁就返回，否则将当前线程放入同步队列等待，与独占式获取锁的不同是，同一时刻可以有多个线程获得共享锁，该方法调用 tryAcquireShared(int)
	void acquireSharedInterruptibly(int)//与 acquireShared(int) 相同，该方法响应中断
	void tryAcquireSharedNanos(int, long)//在 acquireSharedInterruptibly(int) 基础上添加了超时控制
	boolean release(int)    //独占式释放锁，该方法会在释放锁后，将同步队列中第一个等待节点唤醒
	boolean releaseShared(int)//共享式释放锁
	Collection getQueuedThreads()//获得同步队列中等待的线程集合


在使用AQS构造自定义的同步组件时需要继承 AQS 并覆写一些指定的方法，通常需要覆写的方法主要有下面几个。

	protected boolean tryAcquire(int)//独占式获取锁，实现该方法需要查询当前状态并判断同步状态是否和预期值相同，然后使用 CAS 操作设置同步状态
	protected boolean tryRelease(int)//独占式释放锁，实际也是修改同步变量
	protected int tryAcquireShared(int)//共享式获取锁，返回大于等于 0 的值，表示获取锁成功，反之获取失败
	protected boolean tryReleaseShared(int)//共享式释放锁
	protected boolean isHeldExclusively()//判断调用该方法的线程是否持有互斥锁





最后推荐两篇参考链接：

[http://ifeve.com/introduce-abstractqueuedsynchronizer/](http://ifeve.com/introduce-abstractqueuedsynchronizer/)

[http://www.cnblogs.com/leesf456/p/5350186.html](http://www.cnblogs.com/leesf456/p/5350186.html)
































































































































