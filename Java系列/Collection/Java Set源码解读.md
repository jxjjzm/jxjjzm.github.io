### Java Set源码解读 ###
***

### 一、HashSet ###

![HashSet](https://i.imgur.com/kktDhKT.png)

#### (一)、底层结构 ####

	public class HashSet<E>
	    extends AbstractSet<E>
	    implements Set<E>, Cloneable, java.io.Serializable
	{
	    static final long serialVersionUID = -5024744406713321676L;
		// HashSet通过HashMap保存集合元素的：
	    private transient HashMap<E,Object> map;
	
	    //HashSet底层由HashMap实现，新增的元素为map的key，而value则默认为PRESENT。
	    private static final Object PRESENT = new Object();
	
	    /**
	     * 无参构造方法,默认new一个HashMap
	     * default initial capacity (16) and load factor (0.75).
	     */
	    public HashSet() {
	        map = new HashMap<>();
	    }
	
	    /**
	     * 
	     * 带集合的构造函数,new一个默认加载因子为0.75，足以包含所有集合元素的HashMap
	     *
	     * @param c the collection whose elements are to be placed into this set
	     * @throws NullPointerException if the specified collection is null
	     */
	    public HashSet(Collection<? extends E> c) {
	        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
	        addAll(c);
	    }
	
	    /**
	     *  指定HashSet初始容量和加载因子的构造函数：主要用于Map内部的扩容机制
	     *
	     * @param      initialCapacity   the initial capacity of the hash map
	     * @param      loadFactor        the load factor of the hash map
	     * @throws     IllegalArgumentException if the initial capacity is less
	     *             than zero, or if the load factor is nonpositive
	     */
	    public HashSet(int initialCapacity, float loadFactor) {
	        map = new HashMap<>(initialCapacity, loadFactor);
	    }
	
	    /**
	     * 指定HashSet初始容量的构造函数，加载因子默认0.75
	     *
	     * @param      initialCapacity   the initial capacity of the hash table
	     * @throws     IllegalArgumentException if the initial capacity is less
	     *             than zero
	     */
	    public HashSet(int initialCapacity) {
	        map = new HashMap<>(initialCapacity);
	    }
	
	    /**
	     * Constructs a new, empty linked hash set.  (This package private
	     * constructor is only used by LinkedHashSet.) The backing
	     * HashMap instance is a LinkedHashMap with the specified initial
	     * capacity and the specified load factor.
	     *
	     * @param      initialCapacity   the initial capacity of the hash map
	     * @param      loadFactor        the load factor of the hash map
	     * @param      dummy             ignored (distinguishes this
	     *             constructor from other int, float constructor.)
	     * @throws     IllegalArgumentException if the initial capacity is less
	     *             than zero, or if the load factor is nonpositive
	     */
	    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
	        map = new LinkedHashMap<>(initialCapacity, loadFactor);
	    }
	}



### 二、SortedSet ###


![SortedSet](https://i.imgur.com/3rdhLH0.png)

#### (一)、底层结构 ####




### 三、NavigableSet ###

![](https://i.imgur.com/EP26ceN.png)

#### (一)、底层结构 ####




### 四、TreeSet ###

![TreeSet](https://i.imgur.com/HEn2zIq.png)

#### (一)、底层结构 ####













