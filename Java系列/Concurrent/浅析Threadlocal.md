### 浅析Threadlocal ###
***
一、ThreadLocal概述

相必Java开发者对ThreadLocal应该都不陌生，本篇文章我们一起来探讨下ThreadLocal的相关实现原理。

ThreadLocal，很多地方也叫线程本地变量。ThreadLocal官方解释如下——
ThreadLocal类用来提供线程内部的局部变量。这些变量在多线程环境下访问(通过get或set方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量（每个线程创建一个单独的变量副本），ThreadLocal实例通常来说都是private static类型。

ThreadLocal类提供了四个对外开放的接口方法，这也是用户操作ThreadLocal类的基本方法：  


- (1) void set(Object value)设置当前线程的线程局部变量的值。
- (2) public Object get()该方法返回当前线程所对应的线程局部变量。 
- (3) public void remove()将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK 5.0新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。 
- (4) protected Object initialValue()返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次，ThreadLocal中的缺省实现直接返回一个null。

ThreadLocal使用示例

	package com.test;  
	  
	import java.sql.Connection;  
	import java.sql.DriverManager;  
	import java.sql.SQLException;  
	  
	public class ConnectionManager {  
	  
	    private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {  
	        @Override  
	        protected Connection initialValue() {  
	            Connection conn = null;  
	            try {  
	                conn = DriverManager.getConnection(  
	                        "jdbc:mysql://localhost:3306/test", "username",  
	                        "password");  
	            } catch (SQLException e) {  
	                e.printStackTrace();  
	            }  
	            return conn;  
	        }  
	    };  
	  
	    public static Connection getConnection() {  
	        return connectionHolder.get();  
	    }  
	  
	    public static void setConnection(Connection conn) {  
	        connectionHolder.set(conn);  
	    }  
	}  



二、ThreadLocal源码分析

废话不多说，直接来上源码：

	/**
	 Returns the value in the current thread's copy of this
	 thread-local variable.  If the variable has no value for thecurrent thread, it is first initialized to the value returned by an invocation of the {@link #initialValue} method.
	  @return the current thread's value of this thread-local
	 */
	public T get() {
		//当前线程
	    Thread t = Thread.currentThread();
		//获取当前线程对应的ThreadLocalMap
	    ThreadLocalMap map = getMap(t);
	    if (map != null) {
			//获取对应ThreadLocal的变量值
	        ThreadLocalMap.Entry e = map.getEntry(this);
	        if (e != null) {
	            @SuppressWarnings("unchecked")
	            T result = (T)e.value;
	            return result;
	        }
	    }
		//若当前线程还未创建ThreadLocalMap，则返回调用此方法并在其中调用createMap方法进行创建并返回初始值。
	    return setInitialValue();
	}


	//设置变量的值
	public void set(T value) {
	   Thread t = Thread.currentThread();
	   ThreadLocalMap map = getMap(t);
	   if (map != null)
	       map.set(this, value);
	   else
	       createMap(t, value);
	}


	private T setInitialValue() {
	   T value = initialValue();
	   Thread t = Thread.currentThread();
	   ThreadLocalMap map = getMap(t);
	   if (map != null)
	       map.set(this, value);
	   else
	       createMap(t, value);
	   return value;
	}


	/**
	为当前线程创建一个ThreadLocalMap的threadlocals,并将第一个值存入到当前map中
	@param t the current thread
	@param firstValue value for the initial entry of the map
	*/
	void createMap(Thread t, T firstValue) {
	    t.threadLocals = new ThreadLocalMap(this, firstValue);
	}


	//删除当前线程中ThreadLocalMap对应的ThreadLocal
	public void remove() {
	       ThreadLocalMap m = getMap(Thread.currentThread());
	       if (m != null)
	           m.remove(this);
	}


ThreadLocal类中的几个主要的方法，他们的核心都是对其内部类ThreadLocalMap进行操作，下面看一下ThreadLocalMap类的源代码：

	static class ThreadLocalMap {
	  //map中的每个节点Entry,其键key是ThreadLocal并且还是弱引用，这也导致了后续会产生内存泄漏问题的原因。
	 static class Entry extends WeakReference<ThreadLocal<?>> {
	           Object value;
	           Entry(ThreadLocal<?> k, Object v) {
	               super(k);
	               value = v;
	   }
	    /**
	     * 初始化容量为16，以为对其扩充也必须是2的指数 
	     */
	    private static final int INITIAL_CAPACITY = 16;
	    /**
	     * 真正用于存储线程的每个ThreadLocal的数组，将ThreadLocal和其对应的值包装为一个Entry。
	     */
	    private Entry[] table;
	
	
	    ///....其他的方法和操作都和map的类似
	}


综合源码分析，ThreadLocal的实现是这样的：每个Thread 维护一个 ThreadLocal.ThreadLocalMap 映射表，这个映射表的 key 是 ThreadLocal实例本身，value 是真正需要存储的 Object。也就是说 ThreadLocal 本身并不存储值，它只是作为一个 key 来让线程从 ThreadLocalMap 获取 value。值得注意的是ThreadLocalMap 是使用 ThreadLocal 的弱引用作为 Key 的，弱引用的对象在 GC 时会被回收。





三、ThreadLocal内存泄漏问题

![](http://7xjtfr.com1.z0.glb.clouddn.com/threadlocal.jpg)

在上面提到过，每个thread中都存在一个map, map的类型是ThreadLocal.ThreadLocalMap. Map中的key为一个threadlocal实例. 这个Map的确使用了弱引用,不过弱引用只是针对key. 每个key都弱引用指向threadlocal. 当把threadlocal实例置为null以后,没有任何强引用指向threadlocal实例,所以threadlocal将会被gc回收. 但是,我们的value却不能回收,因为存在一条从current thread连接过来的强引用. 只有当前thread结束以后, current thread就不会存在栈中,强引用断开, Current Thread, Map, value将全部被GC回收. 所以得出一个结论就是只要这个线程对象被gc回收，就不会出现内存泄露，但在threadLocal设为null和线程结束这段时间不会被回收的，就发生了我们认为的内存泄露。其实这是一个对概念理解的不一致，也没什么好争论的。最要命的是线程对象不被回收的情况，这就发生了真正意义上的内存泄露。比如使用线程池的时候，线程结束是不会销毁的，会再次使用的。就可能出现内存泄露。

其实，ThreadLocalMap的设计中已经考虑到这种情况，也加上了一些防护措施：在ThreadLocal的get(),set(),remove()的时候都会清除线程ThreadLocalMap里所有key为null的value。但是这些被动的预防措施并不能保证不会内存泄漏：


- 使用static的ThreadLocal，延长了ThreadLocal的生命周期，可能导致的内存泄漏。
- 分配使用了ThreadLocal又不再调用get(),set(),remove()方法，那么可能会导致内存泄漏。


从表面上看内存泄漏的根源在于使用了弱引用。下面我们分两种情况讨论：



- key 使用强引用：引用的ThreadLocal的对象被回收了，但是ThreadLocalMap还持有ThreadLocal的强引用，如果没有手动删除，ThreadLocal不会被回收，导致Entry内存泄漏。
- key 使用弱引用：引用的ThreadLocal的对象被回收了，由于ThreadLocalMap持有ThreadLocal的弱引用，即使没有手动删除，ThreadLocal也会被回收。value在下一次ThreadLocalMap调用set,get，remove的时候会被清除。

比较两种情况，我们可以发现：由于ThreadLocalMap的生命周期跟Thread一样长，如果都没有手动删除对应key，都会导致内存泄漏，但是使用弱引用可以多一层保障：弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除。因此，**ThreadLocal内存泄漏的根源是：由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应key就会导致内存泄漏，而不是因为弱引用。**
























































