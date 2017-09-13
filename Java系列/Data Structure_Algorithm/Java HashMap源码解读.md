### Java HashMap源码解读 ###
***

在JDK1.8以前，HashMap采用位桶（动态数组）+单向链表实现，这种实现存在一个问题：即使负载因子和Hash算法设计的再合理，当出现大量Hash冲突的时候也免不了会出现链表拉链过长的情况，一旦出现拉链过长，则会严重影响HashMap的性能。于是，在JDK1.8版本中，对数据结构（做了进一步优化，引入了红黑树，底层采用位桶（动态数组）+单向链表+红黑树实现。当链表长度超过阈值（8）时，将链表转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能。此外，在JDK1.8的实现中，在高位运算的算法等方面也进行了优化。下面我们对比来看下JDK1.7和JDK1.8 HashMap底层的不同实现。

- **JDK1.7** ：

![HashMap1.7](https://i.imgur.com/UN7DGe5.png)



- **JDK1.8** ：

![HashMap1.8](https://i.imgur.com/40eBxeI.png)




### 一、HashMap 底层结构 ###

- JDK1.7

		public class HashMap<K,V> extends AbstractMap<K,V> 
		        implements Map<K,V>, Cloneable, Serializable {
		
		    //hashMap中的数组初始化大小：1 << 4=2^4=16
		    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
		
		    //1<<30 表示1左移30位，每左移一位乘以2，所以就是1*2^30=1073741824。
		    static final int MAXIMUM_CAPACITY = 1 << 30;
		
		    //默认装载因子：
		    static final float DEFAULT_LOAD_FACTOR = 0.75f;
		
		    //HashMap默认初始化的空数组：
		    static final java.util.HashMap.Entry<?,?>[] EMPTY_TABLE = {};
		
		    //HashMap中底层保存数据的数组：HashMap其实就是一个Entry数组
		    transient java.util.HashMap.Entry<K,V>[] table = (java.util.HashMap.Entry<K,V>[]) EMPTY_TABLE;
		
		    //Hashmap中元素的个数：
		    transient int size;
		
		    //threshold：等于capacity * loadFactory，决定了HashMap能够放进去的数据量
		    int threshold;
		
		    //loadFactor：装载因子，默认值为0.75，它决定了bucket填充程度；
		    final float loadFactor;
		
		    //HashMap被操作的次数：
		    transient int modCount;


			static class Entry<K,V> implements Map.Entry<K,V> {
			    //Entry属性-也就是HashMap的key
			    final K key;
			
			    //Entry属性-也就是HashMap的value
			    V value;
			
			    //指向下一个节点的引用：实现单向链表结构
			    java.util.HashMap.Entry<K,V> next;
			
			    //此Entry的hash值：也就是key的hash值
			    int hash;
			
			    // 构造函数：
			    Entry(int h, K k, V v, java.util.HashMap.Entry<K,V> n) {
			        value = v;
			        next = n;
			        key = k;
			        hash = h;
			    }
			    public final K getKey() { return key;}
			    public final V getValue() { return value; }
			    public final V setValue(V newValue) {
			        V oldValue = value;
			        value = newValue;
			        return oldValue;
			    }
			    public final boolean equals(Object o) {
			
			        if (!(o instanceof Map.Entry))
			            return false;
			        Map.Entry e = (Map.Entry)o;
			        Object k1 = getKey();
			        Object k2 = e.getKey();
			        if (k1 == k2 || (k1 != null && k1.equals(k2))) {
			            Object v1 = getValue();
			            Object v2 = e.getValue();
			            if (v1 == v2 || (v1 != null && v1.equals(v2)))
			                return true;
			        }
			        return false;
			    }
			    public final int hashCode() {
			        return Objects.hashCode(getKey()) ^ Objects.hashCode(getValue());
			    }
			    public final String toString() {
			        return getKey() + "=" + getValue();
			    }
			    void recordAccess(java.util.HashMap<K,V> m) {
			    }
			    void recordRemoval(java.util.HashMap<K,V> m) {
			    }
			}


			//构造一个指定初始容量和指定加载因子的空HashMap
			    public HashMap(int initialCapacity, float loadFactor) {
			        if (initialCapacity < 0)
			            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
			        //指定初始化容量大于 HashMap规定最大容量的话，就将其设置为最大容量；
			        if (initialCapacity > MAXIMUM_CAPACITY)
			            initialCapacity = MAXIMUM_CAPACITY;
			        //不能小于0，判断参数float的值是否是数字
			        if (loadFactor <= 0 || Float.isNaN(loadFactor))
			            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
			        this.loadFactor = loadFactor;
			        threshold = initialCapacity;
			        //空方法：没有任何实现
			        init();
			    }
			
			    //构造一个指定初始容量和默认加载因子（0.75）的空HashMap
			    public HashMap(int initialCapacity) {
			        this(initialCapacity, DEFAULT_LOAD_FACTOR);
			    }
			
			    //构造一个默认初始容量（1）和默认装载因子（0.75）的空HashMap
			    public HashMap() {
			        this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
			    }
			
			    public HashMap(Map<? extends K, ? extends V> m) {
			        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
			                DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
			        inflateTable(threshold);
			
			        putAllForCreate(m);
			    }
		}

	

JDK1.7 HashMap底层数据结构是"位桶（动态数组）+单向链表"：

![](https://user-gold-cdn.xitu.io/2017/9/12/7b6ff02aa46f475f5621924f5a4892c0?imageView2/0/w/1280/h/960)


- JDK1.8

	public class HashMap<K,V> extends AbstractMap<K,V>
	    implements Map<K,V>, Cloneable, Serializable {
	
	    private static final long serialVersionUID = 362498820763181265L;
	
	    
	    /**
	     * The default initial capacity - MUST be a power of two.
	     */
	    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
	
	    /**
	     * The maximum capacity, used if a higher value is implicitly specified
	     * by either of the constructors with arguments.
	     * MUST be a power of two <= 1<<30.
	     */
	    static final int MAXIMUM_CAPACITY = 1 << 30;
	
	    /**
	     * The load factor used when none specified in constructor.
	     */
	    static final float DEFAULT_LOAD_FACTOR = 0.75f;
	
	    /**
	     * The bin count threshold for using a tree rather than list for a
	     * bin.  Bins are converted to trees when adding an element to a
	     * bin with at least this many nodes. The value must be greater
	     * than 2 and should be at least 8 to mesh with assumptions in
	     * tree removal about conversion back to plain bins upon
	     * shrinkage.
	     */
	    static final int TREEIFY_THRESHOLD = 8;
	
	    /**
	     * The bin count threshold for untreeifying a (split) bin during a
	     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
	     * most 6 to mesh with shrinkage detection under removal.
	     */
	    static final int UNTREEIFY_THRESHOLD = 6;
	
	    /**
	     * The smallest table capacity for which bins may be treeified.
	     * (Otherwise the table is resized if too many nodes in a bin.)
	     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
	     * between resizing and treeification thresholds.
	     */
	    static final int MIN_TREEIFY_CAPACITY = 64;
	
	    /**
	     * Basic hash bin node, used for most entries.  (See below for
	     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
	     */
	    static class Node<K,V> implements Map.Entry<K,V> {
	        final int hash;
	        final K key;
	        V value;
	        Node<K,V> next;
	
	        Node(int hash, K key, V value, Node<K,V> next) {
	            this.hash = hash;
	            this.key = key;
	            this.value = value;
	            this.next = next;
	        }
	
	        public final K getKey()        { return key; }
	        public final V getValue()      { return value; }
	        public final String toString() { return key + "=" + value; }
	
	        public final int hashCode() {
	            return Objects.hashCode(key) ^ Objects.hashCode(value);
	        }
	
	        public final V setValue(V newValue) {
	            V oldValue = value;
	            value = newValue;
	            return oldValue;
	        }
	
	        public final boolean equals(Object o) {
	            if (o == this)
	                return true;
	            if (o instanceof Map.Entry) {
	                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
	                if (Objects.equals(key, e.getKey()) &&
	                    Objects.equals(value, e.getValue()))
	                    return true;
	            }
	            return false;
	        }
	    }


	
		    /**
		     * 
		     * 构造一个指定初始容量和指定加载因子的空HashMap
		     *
		     * @param  initialCapacity 初始容量
		     * @param  loadFactor      加载因子
		     * @throws IllegalArgumentException if the initial capacity is negative
		     *         or the load factor is nonpositive
		     */
		    public HashMap(int initialCapacity, float loadFactor) {
		        if (initialCapacity < 0)
		            throw new IllegalArgumentException("Illegal initial capacity: " +
		                                               initialCapacity);
		        if (initialCapacity > MAXIMUM_CAPACITY)
		            initialCapacity = MAXIMUM_CAPACITY;
		        if (loadFactor <= 0 || Float.isNaN(loadFactor))
		            throw new IllegalArgumentException("Illegal load factor: " +
		                                               loadFactor);
		        this.loadFactor = loadFactor;
		        this.threshold = tableSizeFor(initialCapacity);
		    }
		
	
	    
			/**
		     * 构造一个指定初始容量和默认加载因子（0.75）的空HashMap
		     *
			 *	
		     * @param  initialCapacity the initial capacity.
		     * @throws IllegalArgumentException if the initial capacity is negative.
		     */
		    public HashMap(int initialCapacity) {
		        this(initialCapacity, DEFAULT_LOAD_FACTOR);
		    }
		


		    /**
		     * 构造一个默认初始容量（16）和默认加载因子（0.75）的空HashMap
		     */
		    public HashMap() {
		        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
		    }
		
		    /**
		     * Constructs a new <tt>HashMap</tt> with the same mappings as the
		     * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
		     * default load factor (0.75) and an initial capacity sufficient to
		     * hold the mappings in the specified <tt>Map</tt>.
		     *
		     * @param   m the map whose mappings are to be placed in this map
		     * @throws  NullPointerException if the specified map is null
		     */
		    public HashMap(Map<? extends K, ? extends V> m) {
		        this.loadFactor = DEFAULT_LOAD_FACTOR;
		        putMapEntries(m, false);
		    }

		}



JDK1.8 HashMap底层数据结构是"位桶（动态数组）+单向链表+红黑树":

![](http://images2015.cnblogs.comblog/616953/201603/616953-20160304192851940-1880633940.png)



### 二、增 ###



- JDK1.7

		public V put(K key, V value) {
		    //如果调用put方法时(第一次调用put方法)，还是空数组，则进行初始化操作
		    if (table == EMPTY_TABLE) {
		        //进行初始化HashMap的table属性，进行容量大小设置
		        inflateTable(threshold);
		    }
		    //如果新增的key为null:
		    if (key == null)
		        //调用key为null的新增方法：
		        return putForNullKey(value);
		
		    //计算新增元素的hash值：
		    int hash = hash(key);
		
		    //根据hash值和数组长度，计算出新增的元素应该位于数组的哪个角标下：
		    int i = indexFor(hash, table.length);
		
		    //判断计算出的角标下，是否有相同的key，可以理解遍历该角标下的链表
		    for (java.util.HashMap.Entry<K,V> e = table[i]; e != null; e = e.next) {
		        Object k;
		        //根据计算出的hash值，以及 equals方法 / == 来判断key是否相同：
		        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
		            //key相同，则替换原有key下的value：
		            V oldValue = e.value;
		            e.value = value;
		            e.recordAccess(this);
		            //返回被替换的值：
		            return oldValue;
		        }
		    }
		    modCount++;
		    //向HashMap中增加元素：
		    addEntry(hash, key, value, i);
		    return null;
		}


		//初始化Entry数组，默认16个大小：
		private void inflateTable(int toSize) {
		    //获取要创建的数组容量大小：计算大于等于toSize的2的次幂（2的几次方）
		    int capacity = roundUpToPowerOf2(toSize);
		    //计算HashMap的阀值：
		    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
		    //创建Entry数组对象，重新对table赋值：
		    table = new java.util.HashMap.Entry[capacity];
		    //判断是否需要初始化hashSeed属性：
		    initHashSeedAsNeeded(capacity);
		}
		
		//计算大于等于number的2的幂数（2的几次方）
		private static int roundUpToPowerOf2(int number) {
		    //在number不大于MAXIMUM_CAPACITY的情况下：
		    return number >= MAXIMUM_CAPACITY ? 
		        MAXIMUM_CAPACITY : 
		            //再次进行三目运算：核心方法Integer.highestOneBit()
					//补充：Integer.highestOneBit(num)只保留二进制的最高位的1，其余全为0；
		            (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
		}

		//新增元素的key为null的话：
		private V putForNullKey(V value) {
		    //遍历数组的第一个元素，看是否已存在为null的key：
		    for (java.util.HashMap.Entry<K,V> e = table[0]; e != null; e = e.next) {
		        //如果存在，则将原有的value替换
		        if (e.key == null) {
		            V oldValue = e.value;
		            e.value = value;
		            e.recordAccess(this);
		            //返回原来的value：
		            return oldValue;
		        }
		    }
		    //如果不包含null，则进行添加操作：
		    modCount++;
		    //向数组中的第一个角标下插入为null的key--value：
		    addEntry(0, null, value, 0);
		    return null;
		}


		void createEntry(int hash, K key, V value, int bucketIndex) {
		    //获取此角标下的Entry对象：
		    java.util.HashMap.Entry<K,V> e = table[bucketIndex];
		    
		    //无论该角标下是否有元素，都将新元素插入该位置下，将原来的元素置为第二个。
		    table[bucketIndex] = new java.util.HashMap.Entry<>(hash, key, value, e);
		
		    //Map集合长度+1
		    size++;
		}









- JDK1.8

		 public V put(K key, V value) {
		      // 对key的hashCode()做hash
		      return putVal(hash(key), key, value, false, true);
		  }
		 
		  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
		                 boolean evict) {
		      Node<K,V>[] tab; Node<K,V> p; int n, i;
		      // 步骤①：table未初始化或者长度为0，进行扩容
		     if ((tab = table) == null || (n = tab.length) == 0)
		         n = (tab = resize()).length;
		     // 步骤②： (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
		     if ((p = tab[i = (n - 1) & hash]) == null) 
		         tab[i] = newNode(hash, key, value, null);
		     else {
		         Node<K,V> e; K k;
		         // 步骤③：节点key存在，直接覆盖value
		         if (p.hash == hash &&
		             ((k = p.key) == key || (key != null && key.equals(k))))
		             e = p;
		         // 步骤④：判断该链为红黑树
		         else if (p instanceof TreeNode)
		             e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		         // 步骤⑤：该链为链表
		         else {
		             for (int binCount = 0; ; ++binCount) {
						//p第一次指向表头,以后依次后移
		                 if ((e = p.next) == null) {
							//e为空，表示已到表尾也没有找到key值相同节点，则新建节点
		                     p.next = newNode(hash, key,value,null);
		                        //链表长度大于阀值8转换为红黑树进行处理
		                     if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
		                         treeifyBin(tab, hash);
		                     break;
		                 }
		                    // key已经存在直接覆盖value
		                 if (e.hash == hash &&
		                     ((k = e.key) == key || (key != null && key.equals(k))))                                            break;
		                 p = e;
		             }
		         }
		        
		         if (e != null) { // existing mapping for key
		             V oldValue = e.value;
		             if (!onlyIfAbsent || oldValue == null)
		                 e.value = value;
		             afterNodeAccess(e);
		             return oldValue;
		         }
		     }
		 
		     ++modCount;
		     // 步骤⑥：超过最大容量 就扩容
		     if (++size > threshold)
		         resize();
		     afterNodeInsertion(evict);
		     return null;
		 }




![HashMap1.8_put](http://tech.meituan.com/img/java-hashmap/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

JDK1.8 HashMap添加操作实现步骤大致如下：

- ①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；
- ②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；
- ③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；
- ④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；
- ⑤.遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；
- ⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。




### 三、删 ###



- JDK1.7

			//移除hashMap中的元素，通过key：
			public V remove(Object key) {
			    //移除HashMap数组中的Entry对象：
			    java.util.HashMap.Entry<K,V> e = removeEntryForKey(key);
			    return (e == null ? null : e.value);
			}
			
			//通过key移除数组中Entry：
			final java.util.HashMap.Entry<K,V> removeEntryForKey(Object key) {
			    //如果Hashmap集合为0，则返回null：
			    if (size == 0) {
			        return null;
			    }
			    //计算key的hash值：
			    int hash = (key == null) ? 0 : hash(key);
			    //计算hash值对应的数组角标：
			    int i = indexFor(hash, table.length);
			
			    //获取key对应的Entry对象：
			    java.util.HashMap.Entry<K,V> prev = table[i];
			    //将此对象赋值给e：
			    java.util.HashMap.Entry<K,V> e = prev;
			
			    //单向链表的遍历：
			    while (e != null) {
			        //获取当前元素的下一个元素：
			        java.util.HashMap.Entry<K,V> next = e.next;
			
			        Object k;
			        //判断元素的hash值、equals方法，是否和传入的key相同：
			        if (e.hash == hash &&
			                ((k = e.key) == key || (key != null && key.equals(k)))) {
			            //增加操作数：
			            modCount++;
			
			            //减少元素数量：
			            size--;
			            //当为链表的第一个元素时，直接将下一个元素顶到链表头部：
			            if (prev == e)
			                table[i] = next;
			            else
			                //当前元素的下下一个元素
			                prev.next = next;
			            //删除当前遍历到的元素：空方法，
			            // 将被删除的元素不再与map有关联，没有置为null之类的操作；
			            e.recordRemoval(this);
			            return e;
			        }
			        //不相同的话，就把
			        prev = e;
			        e = next;
			    }
			    return e;
			}



与查找相同，删除元素的方法也比较简单，主要是将元素移除HashMap的Entry[]数组。如果为数组角标下的第一个元素，则直接链表的第二个元素移动到头部来。如果不为第一个元素，则将当前元素的前一个元素的next属性指向当前元素的下一个元素即可；



- JDK1.8

			public V remove(Object key) {
			        Node<K,V> e;
			        return (e = removeNode(hash(key), key, null, false, true)) == null ?
			            null : e.value;
			    }



			 /**
			     * Implements Map.remove and related methods
			     *
			     * @param hash hash for key
			     * @param key the key
			     * @param value the value to match if matchValue, else ignored
			     * @param matchValue if true only remove if value is equal
			     * @param movable if false do not move other nodes while removing
			     * @return the node, or null if none
			     */
			    final Node<K,V> removeNode(int hash, Object key, Object value,
			                               boolean matchValue, boolean movable) {
			        Node<K,V>[] tab; Node<K,V> p; int n, index;
			        if ((tab = table) != null && (n = tab.length) > 0 &&
			            (p = tab[index = (n - 1) & hash]) != null) {
			            Node<K,V> node = null, e; K k; V v;
			            if (p.hash == hash &&
			                ((k = p.key) == key || (key != null && key.equals(k))))
			                node = p;
			            else if ((e = p.next) != null) {
			                if (p instanceof TreeNode)
			                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
			                else {
			                    do {
			                        if (e.hash == hash &&
			                            ((k = e.key) == key ||
			                             (key != null && key.equals(k)))) {
			                            node = e;
			                            break;
			                        }
			                        p = e;
			                    } while ((e = e.next) != null);
			                }
			            }
			            if (node != null && (!matchValue || (v = node.value) == value ||
			                                 (value != null && value.equals(v)))) {
			                if (node instanceof TreeNode)
			                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
			                else if (node == p)
			                    tab[index] = node.next;
			                else
			                    p.next = node.next;
			                ++modCount;
			                --size;
			                afterNodeRemoval(node);
			                return node;
			            }
			        }
			        return null;
			    }





### 四、查 ###



- JDK1.7


			//通过key 获取对应的value:
			public V get(Object key) {
			    if (key == null)
			        //获取key==null的value：
			        return getForNullKey();
			
			    //获取key不为null的value值：
			    java.util.HashMap.Entry<K,V> entry = getEntry(key);
			
			    //返回对应的value：
			    return null == entry ? null : entry.getValue();
			}
			
			//获取hashMap中key为 null的value值：
			private V getForNullKey() {
			    //hashmap中没有元素，则返回null：
			    if (size == 0) {
			        return null;
			    }
			    //获取Entry数组中，角标为0的Entry对象（put的时候如果有null的key，就存放到角标为0的位置）
			    //获取角标为0的Entry对象，遍历整个链表，看是否有key为null的key：返回对应的value：
			    for (java.util.HashMap.Entry<K,V> e = table[0]; e != null; e = e.next) {
			        if (e.key == null)
			            return e.value;
			    }
			    //如果都不存在，则返回null：
			    return null;
			}
			
			 //通过对应的key获取 Entry对象：
			final java.util.HashMap.Entry<K,V> getEntry(Object key) {
			    //hashmap长度为0 ，则返回null：
			    if (size == 0) {
			        return null;
			    }
			    //获取key对应的hash值：若key为null,则hash值为0；
			    int hash = (key == null) ? 0 : hash(key);
			    //计算hash值对应数组的角标，获取数组中角标下的Entry对象：对该元素所属的链表进行遍历；
			    for (java.util.HashMap.Entry<K,V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {
			        Object k;
			        //判断key的hash值，再调用equlas方法进行判断：hash值可能会相同，equals进一步验证；
			        if (e.hash == hash &&
			                ((k = e.key) == key || (key != null && key.equals(k))))
			            return e;
			    }
			    //没找到返回null:
			    return null;
			}


相比于新增来说，HashMap的查找方法就很简单了。一种是获取key为null的情况，一种是key非null的情况。


- JDK1.8

			public V get(Object key) {
			    Node<k,v> e;
			    return (e = getNode(hash(key), key)) == null ? null : e.value;
			}
			 
			 
			final Node<k,v> getNode(int hash, Object key) {
			    Node<k,v>[] tab; Node<k,v> first, e; int n; K k;
			    //hash & (length-1)得到对象的保存位
			    if ((tab = table) != null && (n = tab.length) > 0 &&
			        (first = tab[(n - 1) & hash]) != null) {
			        if (first.hash == hash && // always check first node
			            ((k = first.key) == key || (key != null && key.equals(k))))
			            return first;
			        if ((e = first.next) != null) {
			            //如果第一个节点是TreeNode,说明采用的是数组+红黑树结构处理冲突
			            //遍历红黑树，得到节点值
			            if (first instanceof TreeNode)
			                return ((TreeNode<k,v>)first).getTreeNode(hash, key);
			            //链表结构处理
			            do {
			                if (e.hash == hash &&
			                    ((k = e.key) == key || (key != null && key.equals(k))))
			                    return e;
			            } while ((e = e.next) != null);
			        }
			    }
			    return null;
			}






### 五、扩容机制 ###



- JDK1.7

		//将HashMap进行扩容：
		void resize(int newCapacity) {
		    //将原有Entry数组赋值给oldTable参数：
		    java.util.HashMap.Entry[] oldTable = table;
		    
		    //获取现阶段Entry数组的长度：
		    int oldCapacity = oldTable.length;
		    
		    //如果现阶段Entry数组的长度 == MAXIMUM_CAPACITY的话：
		    if (oldCapacity == MAXIMUM_CAPACITY) {
		        //将阈值设置为Integer的最大值，并返回
		        threshold = Integer.MAX_VALUE;
		        //没有进行扩容操作：
		        return;
		    }
		    
		    //创建新Entry数组：容量为现有2倍大小
		    java.util.HashMap.Entry[] newTable = new java.util.HashMap.Entry[newCapacity];
		    
		    //将原有Entry数组中的元素，添加到新数组中：
		    transfer(newTable, initHashSeedAsNeeded(newCapacity));
		    
		    //新数组赋值给table属性：
		    table = newTable;
		    //重新计算扩容阈值：
		    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
		}
		
		//原有元素复制到新数组中：
		void transfer(java.util.HashMap.Entry[] newTable, boolean rehash) {
		    //新数组长度：
		    int newCapacity = newTable.length;
		    
		    //遍历原有Entry数组：
		    for (java.util.HashMap.Entry<K,V> e : table) {
		        //判断Entry[]中不为null的对象： 
		        while(null != e) {
		            java.util.HashMap.Entry<K,V> next = e.next;
		            //此处需要再次判断hashSeed是否进行初始化：
		            if (rehash) {
		                //对于Strig类型来说，hashSeed初始化后，需要调用sun.misc.Hashing.stringHash32来计算hash值
		                e.hash = null == e.key ? 0 : hash(e.key);
		            }
		            //计算新数组中元素所处于的角标：
		            int i = indexFor(e.hash, newCapacity);
		            e.next = newTable[i];
		            newTable[i] = e;
		            e = next;
		        }
		    }
		}



			//元素key的hash值计算：
			final int hash(Object k) {
			    int h = hashSeed;
			    //如果为String类型，并且hashSeed不等于0，则会调用sun.misc.Hashing.stringHash32()进行hash值计算
			    if (0 != h && k instanceof String) {
			        return sun.misc.Hashing.stringHash32((String) k);
			    }
			
			    //计算key的hash值，调用key的hashCode方法：
			    h ^= k.hashCode();
			    h ^= (h >>> 20) ^ (h >>> 12);
			
			    //经过一系列位运算，得出一个hash值：
			    return h ^ (h >>> 7) ^ (h >>> 4);
			}


			/计算元素所处于数组的位置，进行与运算
			//一般使用hash值对length取模（即除法散列法）
			static int indexFor(int h, int length) {
				return h & (length-1);
			}

- JDK1.8

			final Node<K,V>[] resize() {
			      Node<K,V>[] oldTab = table;
			      int oldCap = (oldTab == null) ? 0 : oldTab.length;
			      int oldThr = threshold;
			      int newCap, newThr = 0;
			      if (oldCap > 0) {
			          // 超过最大值就不再扩充了，就只好随你碰撞去吧
			          if (oldCap >= MAXIMUM_CAPACITY) {
			              threshold = Integer.MAX_VALUE;
			             return oldTab;
			         }
			         // 没超过最大值，就扩充为原来的2倍
			         else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
			                  oldCap >= DEFAULT_INITIAL_CAPACITY)
			             newThr = oldThr << 1; // double threshold
			     }
			     else if (oldThr > 0) // initial capacity was placed in threshold
			         newCap = oldThr;
			     else {               // zero initial threshold signifies using defaults
			         newCap = DEFAULT_INITIAL_CAPACITY;
			         newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
			     }
			     // 计算新的resize上限
			     if (newThr == 0) {
			
			         float ft = (float)newCap * loadFactor;
			         newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
			                   (int)ft : Integer.MAX_VALUE);
			     }
			     threshold = newThr;
			     @SuppressWarnings({"rawtypes"，"unchecked"})
			         Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
			     table = newTab;
			     if (oldTab != null) {
			         // 把每个bucket都移动到新的buckets中
			         for (int j = 0; j < oldCap; ++j) {
			             Node<K,V> e;
			             if ((e = oldTab[j]) != null) {
			                 oldTab[j] = null;
			                 if (e.next == null)
			                     newTab[e.hash & (newCap - 1)] = e;
			                 else if (e instanceof TreeNode)
			                     ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
			                 else { // 链表优化重hash的代码块
			                     Node<K,V> loHead = null, loTail = null;
			                     Node<K,V> hiHead = null, hiTail = null;
			                     Node<K,V> next;
			                     do {
			                         next = e.next;
			                         // 原索引
			                         if ((e.hash & oldCap) == 0) {
			                             if (loTail == null)
			                                 loHead = e;
			                             else
			                                 loTail.next = e;
			                             loTail = e;
			                         }
			                         // 原索引+oldCap
			                         else {
			                             if (hiTail == null)
			                                 hiHead = e;
			                             else
			                                 hiTail.next = e;
			                             hiTail = e;
			                         }
			                     } while ((e = next) != null);
			                     // 原索引放到bucket里
			                     if (loTail != null) {
			                         loTail.next = null;
			                         newTab[j] = loHead;
			                     }
			                     // 原索引+oldCap放到bucket里
			                     if (hiTail != null) {
			                         hiTail.next = null;
			                         newTab[j + oldCap] = hiHead;
			                     }
			                 }
			             }
			         }
			     }
			     return newTab;
			 }



			static final int hash(Object key) {
			    int h;
			    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
			}







确定哈希桶数组索引位置:

![](http://tech.meituan.com/img/java-hashmap/hashMap%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE.png)


在JDK1.8 的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。




































































