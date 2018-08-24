### Java LinkedHashMap源码解读 ###
***
前面我们已经分析了HashMap的源码，已经了解了HashMap底层数据结构以及增删查改操作实现原理，知道了可以用在哪种场合，如果这样一种情形，我们需要按照元素插入顺序或者访问顺序来迭代元素，此时，LinkedHashMap就派上用场了。LinkedHashMap 可以按元素插入（先进先出）顺序或元素最近访问顺序(LRU)排列。

![](https://i.imgur.com/mKE6N7O.png)

由于LinkedHashMap底层基本上都是基于HashMap基础上扩展的，所以如果你对HashMap源码实现不太清楚，可以先看上篇《Java HashMap 源码解读》，本篇着重就LinkedHashMap的不同之处进行分析。



### 一、LinkedHashMap底层数据结构 ###

**JDK1.7：**

![](https://i.imgur.com/3D5pEp1.png)


	package java.util;
	import java.io.*;
	
	
	public class LinkedHashMap<K,V>
	    extends HashMap<K,V>
	    implements Map<K,V>
	{
	
	    private static final long serialVersionUID = 3801124242820219131L;
	
	    /**
	     * 双向循环链表，  头结点(空节点)
	     */
	    private transient Entry<K,V> header;
	
	    /**
	     * accessOrder为true时，按访问顺序排序，false时，按插入顺序排序
	     */
	    private final boolean accessOrder;
	
	    /**
	     * 生成一个空的LinkedHashMap,并指定其容量大小和负载因子，
	     * 默认将accessOrder设为false，按插入顺序排序
	     */
	    public LinkedHashMap(int initialCapacity, float loadFactor) {
	        super(initialCapacity, loadFactor);
	        accessOrder = false;
	    }
	
	    /**
	     * 生成一个空的LinkedHashMap,并指定其容量大小，负载因子使用默认的0.75，
	     * 默认将accessOrder设为false，按插入顺序排序
	     */
	    public LinkedHashMap(int initialCapacity) {
	        super(initialCapacity);
	        accessOrder = false;
	    }
	
	    /**
	     * 生成一个空的HashMap,容量大小使用默认值16，负载因子使用默认值0.75
	     * 默认将accessOrder设为false，按插入顺序排序.
	     */
	    public LinkedHashMap() {
	        super();
	        accessOrder = false;
	    }
	
	    /**
	     * 根据指定的map生成一个新的HashMap,负载因子使用默认值，初始容量大小为Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,DEFAULT_INITIAL_CAPACITY)
	     * 默认将accessOrder设为false，按插入顺序排序.
	     */
	    public LinkedHashMap(Map<? extends K, ? extends V> m) {
	        super(m);
	        accessOrder = false;
	    }
	
	    /**
	     * 生成一个空的LinkedHashMap,并指定其容量大小和负载因子，
	     * 默认将accessOrder设为true，按访问顺序排序
	     */
	    public LinkedHashMap(int initialCapacity,
	                         float loadFactor,
	                         boolean accessOrder) {
	        super(initialCapacity, loadFactor);
	        this.accessOrder = accessOrder;
	    }
	
		 /**
	     * LinkedHashMap节点对象
	     */
	    private static class Entry<K,V> extends HashMap.Entry<K,V> {
	        // 节点前后引用
	        Entry<K,V> before, after;
	
	        //构造函数与HashMap一致
	        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
	            super(hash, key, value, next);
	        }
	
	        /**
	         * 移除节点，并修改前后引用
	         */
	        private void remove() {
	            before.after = after;
	            after.before = before;
	        }
	
	        /**
	         * 将当前节点插入到existingEntry的前面
	         */
	        private void addBefore(Entry<K,V> existingEntry) {
	            after  = existingEntry;
	            before = existingEntry.before;
	            before.after = this;
	            after.before = this;
	        }
	
	        /**
	         * 在HashMap的put和get方法中，会调用该方法，在HashMap中该方法为空
	         * 在LinkedHashMap中，当按访问顺序排序时，该方法会将当前节点插入到链表尾部(头结点的前一个节点)，否则不做任何事
	         */
	        void recordAccess(HashMap<K,V> m) {
	            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
	            //当LinkedHashMap按访问排序时
	            if (lm.accessOrder) {
	                lm.modCount++;
	                //移除当前节点
	                remove();
	                //将当前节点插入到头结点前面
	                addBefore(lm.header);
	            }
	        }
	
	        void recordRemoval(HashMap<K,V> m) {
	            remove();
	        }
	    }
	
	}



JDK1.7 LinkedHashMap底层采用（位桶（动态数组）+ 单向链表 + 双向循环链表）

![](http://images2015.cnblogs.com/blog/249993/201612/249993-20161213140338917-602479781.png)

![](http://images2015.cnblogs.com/blog/249993/201612/249993-20161215143120620-1544337380.png)

**JDK1.8**

	public class LinkedHashMap<K,V>  extends HashMap<K,V> implements Map<K,V> {
		    static class Entry<K,V> extends HashMap.Node<K,V> {
		        Entry<K,V> before, after;
		        Entry(int hash, K key, V value, Node<K,V> next) {
		            super(hash, key, value, next);
		        }
		    }
			//由于继承HashMap，所以HashMap中的非private方法和字段，都可以在LinkedHashMap直接中访问
		    // 版本序列号
		    private static final long serialVersionUID = 3801124242820219131L;
		
		    // 链表头结点
		    transient LinkedHashMap.Entry<K,V> head;
		
		    // 链表尾结点
		    transient LinkedHashMap.Entry<K,V> tail;
		
		    // 访问顺序
		    final boolean accessOrder;
		
		//LinkedHashMap(int, float)型构造函数
		public LinkedHashMap(int initialCapacity, float loadFactor) {
		        super(initialCapacity, loadFactor);
		        accessOrder = false;
		}
		
		//LinkedHashMap(int)型构造函数
		public LinkedHashMap(int initialCapacity) {
		        super(initialCapacity);
		        accessOrder = false;
		}
		
		//LinkedHashMap()型构造函数
		public LinkedHashMap() {
		        super();
		        accessOrder = false;
		}
		
		//LinkedHashMap(Map<? extends K, ? extends V>)型构造函数
		public LinkedHashMap(Map<? extends K, ? extends V> m) {
		    super();
		    accessOrder = false;
		    putMapEntries(m, false);
		}
		
		//LinkedHashMap(int, float, boolean)型构造函数：可以指定accessOrder的值，从而控制访问顺序。
		public LinkedHashMap(int initialCapacity,
		                         float loadFactor,
		                         boolean accessOrder) {
		    super(initialCapacity, loadFactor);
		    this.accessOrder = accessOrder;
		}
	
	}


JDK1.8 LinkedHashMap底层采用（位桶（动态数组）+ 单向链表 + 红黑二叉树 + 双向循环链表）

![](https://i.imgur.com/H09qhya.png)


总结说明：

- 从源码中可以看出，LinkedHashMap中加入了一个header节点，将所有插入到该LinkedHashMap中的节点按照插入的先后顺序依次加入到以head为头结点的双向循环链表的尾部。注意源码中的accessOrder标志位，当它false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致，这也是默认的双向链表的存储顺序；当它为true时，表示双向链表中的元素按照访问的先后顺序排列，可以看到，虽然Entry插入链表的顺序依然是按照其put到LinkedHashMap中的顺序，但put和get方法均有调用recordAccess方法（put方法在key相同，覆盖原有的Entry的情况下调用recordAccess方法），该方法判断accessOrder是否为true，如果是，则将当前访问的Entry（put进来的Entry或get出来的Entry）移到双向链表的尾部（key不相同时，put新Entry时，会调用addEntry，它会调用creatEntry，该方法同样将新插入的元素放入到双向链表的尾部，既符合插入的先后顺序，又符合访问的先后顺序，因为这时该Entry也被访问了），否则，什么也不做。
- 注意这个header，hash值为-1，其他都为null，也就是说这个header不放在数组中，就是用来指示开始元素和标志结束元素的。header的目的是为了记录第一个插入的元素是谁，在遍历的时候能够找到第一个元素。
![](http://images2015.cnblogs.com/blog/249993/201612/249993-20161213164756542-224440628.png)
- 不要搞错了next和before、After，next是用于维护HashMap指定table位置上连接的Entry的顺序的，before、After是用于维护Entry插入的先后顺序的。


### 二、LinkedHashMap 重要函数分析 ###




**JDK1.7：**


	
	
		    /**
		     * 覆盖HashMap的init方法，在构造方法、Clone、readObject方法里会调用该方法
		     * 作用是生成一个双向链表头节点，初始化其前后节点引用
		     */
		    @Override
		    void init() {
		        header = new Entry<>(-1, null, null, null);
		        header.before = header.after = header;
		    }
		
		    /**
		     * 覆盖HashMap的transfer方法，性能优化，这里遍历方式不采用HashMap的双重循环方式
		     * 而是直接通过双向链表遍历Map中的所有key-value映射
		     */
		    @Override
		    void transfer(HashMap.Entry[] newTable, boolean rehash) {
		        int newCapacity = newTable.length;
		        //遍历旧Map中的所有key-value
		        for (Entry<K,V> e = header.after; e != header; e = e.after) {
		            if (rehash)
		                e.hash = (e.key == null) ? 0 : hash(e.key);
		            //根据新的数组长度，重新计算索引，
		            int index = indexFor(e.hash, newCapacity);
		            //插入到链表表头
		            e.next = newTable[index];
		            //将e放到索引为i的数组处
		            newTable[index] = e;
		        }
		    }
		
		
		    /**
		     * 覆盖HashMap的transfer方法，性能优化，这里遍历方式不采用HashMap的双重循环方式
		     * 而是直接通过双向链表遍历Map中的所有key-value映射,
		     */
		    public boolean containsValue(Object value) {
		        // Overridden to take advantage of faster iterator
		        if (value==null) {
		            for (Entry e = header.after; e != header; e = e.after)
		                if (e.value==null)
		                    return true;
		        } else {
		            for (Entry e = header.after; e != header; e = e.after)
		                if (value.equals(e.value))
		                    return true;
		        }
		        return false;
		    }
		
		    /**
		     * 通过key获取value，与HashMap的区别是：当LinkedHashMap按访问顺序排序的时候，会将访问的当前节点移到链表尾部(头结点的前一个节点)
		     */
		    public V get(Object key) {
		        Entry<K,V> e = (Entry<K,V>)getEntry(key);
		        if (e == null)
		            return null;
		        e.recordAccess(this);
		        return e.value;
		    }
		
		    /**
		     * 调用HashMap的clear方法，并将LinkedHashMap的头结点前后引用指向自己
		     */
		    public void clear() {
		        super.clear();
		        header.before = header.after = header;
		    }

	 		/**
		     * 创建节点，插入到LinkedHashMap中，该方法覆盖HashMap的addEntry方法
		     */
		    void addEntry(int hash, K key, V value, int bucketIndex) {
		        super.addEntry(hash, key, value, bucketIndex);
		
		        // 注意头结点的下个节点即header.after，存放于链表头部，是最不经常访问或第一个插入的节点，
		        //有必要的情况下(如容量不够,具体看removeEldestEntry方法的实现，这里默认为false，不删除)，可以先删除
		        Entry<K,V> eldest = header.after;
		        if (removeEldestEntry(eldest)) {
		            removeEntryForKey(eldest.key);
		        }
		    }
		
		    /**
		     * 创建节点，并将该节点插入到链表尾部
		     */
		    void createEntry(int hash, K key, V value, int bucketIndex) {
		        HashMap.Entry<K,V> old = table[bucketIndex];
		        Entry<K,V> e = new Entry<>(hash, key, value, old);
		        table[bucketIndex] = e;
		        //将该节点插入到链表尾部
		        e.addBefore(header);
		        size++;
		    }
		
		    /**
		     * 该方法在创建新节点的时候调用，
		     * 判断是否有必要删除链表头部的第一个节点(最不经常访问或最先插入的节点，由accessOrder决定)
		     */
		    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
		        return false;
		    }
		

![](http://images2015.cnblogs.com/blog/879896/201603/879896-20160319113023553-1433533882.jpg)


**JDK1.8：**

(1) newNode函数


	// 当桶中结点类型为HashMap.Node类型时，调用此函数
	Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
	    // 生成Node结点
	    LinkedHashMap.Entry<K,V> p =
	        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
	    // 将该结点插入双链表末尾
	    linkNodeLast(p);
	    return p;
	}

说明：此函数在HashMap类中也有实现，LinkedHashMap重写了该函数，所以当实际对象为LinkedHashMap，桶中结点类型为Node时，我们调用的是LinkedHashMap的newNode函数，而非HashMap的函数，newNode函数会在调用put函数时被调用。可以看到，除了新建一个结点之外，还把这个结点链接到双链表的末尾了，这个操作维护了插入顺序。（其中LinkedHashMap.Entry继承自HashMap.Node）

（2）newTreeNode函数

	// 当桶中结点类型为HashMap.TreeNode时，调用此函数
	TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
	    // 生成TreeNode结点
	    TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
	    // 将该结点插入双链表末尾
	    linkNodeLast(p);
	    return p;
	}

说明：当桶中结点类型为TreeNode时候，插入结点时调用的此函数，也会链接到末尾。

（3）afterNodeAccess函数

	void afterNodeAccess(Node<K,V> e) { // move node to last
	    LinkedHashMap.Entry<K,V> last;
	    // 若访问顺序为true，且访问的对象不是尾结点
	    if (accessOrder && (last = tail) != e) {
	        // 向下转型，记录p的前后结点
	        LinkedHashMap.Entry<K,V> p =
	            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
	        // p的后结点为空
	        p.after = null;
	        // 如果p的前结点为空
	        if (b == null)
	            // a为头结点
	            head = a;
	        else // p的前结点不为空
	            // b的后结点为a
	            b.after = a;
	        // p的后结点不为空
	        if (a != null)
	            // a的前结点为b
	            a.before = b;
	        else // p的后结点为空
	            // 后结点为最后一个结点
	            last = b;
	        // 若最后一个结点为空
	        if (last == null)
	            // 头结点为p
	            head = p;
	        else { // p链入最后一个结点后面
	            p.before = last;
	            last.after = p;
	        }
	        // 尾结点为p
	        tail = p;
	        // 增加结构性修改数量
	        ++modCount;
	    }
	}


说明：此函数在很多函数（如put）中都会被回调，LinkedHashMap重写了HashMap中的此函数。若访问顺序为true则将该节点移到双向链表的尾部.下面的图展示了访问前和访问后的状态，假设访问的结点为结点3.（从图中可以看到，结点3链接到了尾结点后面。）

![](http://images2015.cnblogs.com/blog/616953/201603/616953-20160307085938194-117935380.png)

（4）transferLinks函数

	// 用dst替换src
	private void transferLinks(LinkedHashMap.Entry<K,V> src,
	                               LinkedHashMap.Entry<K,V> dst) {
	    LinkedHashMap.Entry<K,V> b = dst.before = src.before;
	    LinkedHashMap.Entry<K,V> a = dst.after = src.after;
	    if (b == null)
	        head = dst;
	    else
	        b.after = dst;
	    if (a == null)
	        tail = dst;
	    else
	        a.before = dst;
	}

说明：此函数用dst结点替换结点，示意图如下（其中只考虑了before与after域，并没有考虑next域，next会在调用tranferLinks函数中进行设定）

[http://images2015.cnblogs.com/blog/616953/201603/616953-20160307091450882-1342476579.png](http://images2015.cnblogs.com/blog/616953/201603/616953-20160307091450882-1342476579.png)

（5）containsValue函数　

	public boolean containsValue(Object value) {
	    // 使用双链表结构进行遍历查找
	    for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
	        V v = e.value;
	        if (v == value || (value != null && value.equals(v)))
	            return true;
	    }
	    return false;
	}

说明：containsValue函数根据双链表结构来查找是否包含value，是按照插入顺序进行查找的，与HashMap中的此函数查找方式不同，HashMap是使用按照桶遍历，没有考虑插入顺序。

### 三、LinkedHashMap 迭代器实现 ###



![](http://images2015.cnblogs.com/blog/616953/201603/616953-20160307154600210-474527278.png)

**JDK1.7：**


	    //迭代器
	    private abstract class LinkedHashIterator<T> implements Iterator<T> {
	        //初始化下个节点引用
	        Entry<K,V> nextEntry    = header.after;
	        Entry<K,V> lastReturned = null;
	
	        /**
	         * 用于迭代期间快速失败行为
	         */
	        int expectedModCount = modCount;
	        
	        //链表遍历结束标志，当下个节点为头节点的时候
	        public boolean hasNext() {
	            return nextEntry != header;
	        }
	
	        //移除当前访问的节点
	        public void remove() {
	            //lastReturned会在nextEntry方法中赋值
	            if (lastReturned == null)
	                throw new IllegalStateException();
	            //快速失败机制
	            if (modCount != expectedModCount)
	                throw new ConcurrentModificationException();
	
	            LinkedHashMap.this.remove(lastReturned.key);
	            lastReturned = null;
	            //迭代器自身删除节点，并不是其他线程修改Map结构，所以这里要修改expectedModCount
	            expectedModCount = modCount;
	        }
	
	        //返回链表下个节点的引用
	        Entry<K,V> nextEntry() {
	            //快速失败机制
	            if (modCount != expectedModCount)
	                throw new ConcurrentModificationException();
	            //链表为空情况
	            if (nextEntry == header)
	                throw new NoSuchElementException();
	            
	            //给lastReturned赋值，最近一个从迭代器返回的节点对象
	            Entry<K,V> e = lastReturned = nextEntry;
	            nextEntry = e.after;
	            return e;
	        }
	    }
	    //key迭代器
	    private class KeyIterator extends LinkedHashIterator<K> {
	        public K next() { return nextEntry().getKey(); }
	    }
	    //value迭代器
	    private class ValueIterator extends LinkedHashIterator<V> {
	        public V next() { return nextEntry().value; }
	    }
	    //key-value迭代器
	    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
	        public Map.Entry<K,V> next() { return nextEntry(); }
	    }
	
	    // 返回不同的迭代器对象
	    Iterator<K> newKeyIterator()   { return new KeyIterator();   }
	    Iterator<V> newValueIterator() { return new ValueIterator(); }
	    Iterator<Map.Entry<K,V>> newEntryIterator() { return new EntryIterator(); }


![](http://images2015.cnblogs.com/blog/249993/201612/249993-20161215143544401-1850524627.jpg)

	
	   
**JDK1.8：**

　
	//LinkedHashIterator是LinkedHashMap的迭代器，为抽象类，用于对LinkedHashMap进行迭代。
	abstract class LinkedHashIterator {
	    // 下一个结点
	    LinkedHashMap.Entry<K,V> next;
	    // 当前结点
	    LinkedHashMap.Entry<K,V> current;
	    // 期望的修改次数（主要用于在遍历HashMap同时，程序对其结构是否进行了修改。若遍历同时修改了，则会抛出异常。）
	    int expectedModCount;
	}

	LinkedHashIterator() {
	    // next赋值为头结点
	    next = head;
	    // 赋值修改次数
	    expectedModCount = modCount;
	    // 当前结点赋值为空
	    current = null;
	}


	// 是否存在下一个结点
	public final boolean hasNext() {
	    return next != null;
	}
	
	//说明：由于所有的结点构成双链表结构，所以nextNode函数也很好理解，直接取得下一个结点即可。
	final LinkedHashMap.Entry<K,V> nextNode() {
	    LinkedHashMap.Entry<K,V> e = next;
	    // 检查是否存在结构性修改
	    if (modCount != expectedModCount)
	        throw new ConcurrentModificationException();
	    // 当前结点是否为空
	    if (e == null)
	        throw new NoSuchElementException();
	    // 赋值当前节点
	    current = e;
	    // 赋值下一个结点
	    next = e.after;
	    return e;
	}
	
	//LinkedHashMap的键迭代器，继承自LinkedHashIterator，实现了Iterator接口，对LinkedHashMap中的键进行迭代。　
	final class LinkedKeyIterator extends LinkedHashIterator
	    implements Iterator<K> {
	    public final K next() { return nextNode().getKey(); }
	}
	
	//LinkedHashMap的值迭代器，继承自LinkedHashIterator，实现了Iterator接口，对LinkedHashMap中的值进行迭代。
	final class LinkedValueIterator extends LinkedHashIterator
	    implements Iterator<V> {
	    public final V next() { return nextNode().value; }
	}
	
	//LinkedHashMap的结点迭代器，继承自LinkedHashIterator，实现了Iterator接口，对LinkedHashMap中的结点进行迭代。　
	final class LinkedEntryIterator extends LinkedHashIterator
	    implements Iterator<Map.Entry<K,V>> {
	    public final Map.Entry<K,V> next() { return nextNode(); }
	}




综上源码分析所述，LinkedHashMap与HashMap最大的不同之处也就是顺序性（以1.8为例）：


- LinkedHashMap底层采用“位桶（动态数组）+ 单向链表 + 红黑树 + 双向循环链表 ”实现，HashMap底层采用“位桶（动态数组）+ 单向链表 + 红黑树 ”实现.LinkedHashMap 结构中的双向循环链表记录着元素顺序（插入顺序或访问顺序）。
- LinkedhashMap遍历的时候考虑了顺序性，通过双向循环链表来遍历，HashMap通过遍历哈希桶实现。


#### 附-Java集合专题面试宝典 ####

- Collection 集合体系
- LinkedList、ArrayList、Vector 比较（线程安全角度、实现原理角度、优缺点）
- LinkedList 底层实现原理
- ArrayList 底层实现原理（赠、删、查、改、扩容（初始容量、扩容大小、扩容机制））
- HashMap、HashTable 比较 （线程安全角度、Key Null角度、Fail Fast）
- HashMap源码（增、删、查、改、扩容（初始容量、扩容因子、扩容机制...）、JDK1.7与JDK1.8）
- LinkedHashMap底层实现原理
- ConcurrentHashMap底层实现原理

（[集合面试必问](https://mp.weixin.qq.com/s?__biz=MzU4NDQ4MzU5OA==&mid=2247484308&idx=1&sn=e3607919aed604be629617f867f46844&chksm=fd9855f5caefdce3f1ee72cb33b9b3bf9899fa2b64bbb92f1e820c0ef3985245b1f7dfc05358&mpshare=1&scene=1&srcid=08248lErlPI3QKHCLNEfSqBN#rd)）

























































