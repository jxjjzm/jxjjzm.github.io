###  List  ###
***
List接口的实现众多，但其常用的实现莫过于ArrayList和LinkedList，这两者我们使用的实在是太多了，非常熟悉，所以在这里将跳过它的使用方法，从源码（JDK1.7）的角度来分析ArrayList和LinkedList的相关实现。




### 一、ArrayList ###

![](https://i.imgur.com/jCcnjHc.png)

针对ArrayList常用操作去看底层源码实现之前不得不先普及两个方法，因为在ArrayList常用操作底层实现中不可避免的用到这两个方法，可以说这两个方法是ArrayList底层实现的精髓。

- Array.copy(U[] original， int newLength, Class<? extends T[]> newType) ：在该方法的内部创建了一个新数组，底层实现是调用System.arraycopy()；

![Arrays.copy](https://i.imgur.com/m7RLfXF.png)	

- System.arraycopy() : 该方法是用了native关键字，调用的为C++编写的底层函数.

![System.arraycopy](https://i.imgur.com/Uaz4bs3.png)

不再多言，走起！！！

#### （一）、底层构造实现 ####

	public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

	    //实现Serializable接口，生成的序列版本号：
	    private static final long serialVersionUID = 8683452581122892189L;
	
	    //ArrayList初始容量大小：在无参构造中不使用了
	    private static final int DEFAULT_CAPACITY = 10;
	
	    //空数组对象：初始化中默认赋值给elementData
	    private static final Object[] EMPTY_ELEMENTDATA = {};
	
	    //ArrayList中实际存储元素的数组：
	    private transient Object[] elementData;
	
	    //集合实际存储元素长度：
	    private int size;
	
	    //ArrayList有参构造：容量大小
	    public ArrayList(int initialCapacity) {
	        //即父类构造：protected AbstractList() {}空方法
	        super();
	        //如果传递的初始容量小于0 ，抛出异常
	        if (initialCapacity < 0)
	            throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
	        //初始化数据：创建Object数组
	        this.elementData = new Object[initialCapacity];
	    }
	
	    //ArrayList无参构造：
	    public ArrayList() {
	        //即父类构造：protected AbstractList() {}空方法
	        super();
	        //初始化数组：空数组，容量为0
	        this.elementData = EMPTY_ELEMENTDATA;
	    }
	
	    //ArrayList有参构造：Java集合
	    public ArrayList(Collection<? extends E> c) {
	        //将集合转换为数组：
	        elementData = c.toArray();
	        //设置数组的长度：
	        size = elementData.length;
	        if (elementData.getClass() != Object[].class)
	            elementData = Arrays.copyOf(elementData, size, Object[].class);
	    }
	}


说明：



1. private static final int DEFAULT_CAPACITY = 10 ： ArrayList ***默认初始容量为10***
2. private transient Object[] elementData ： ArrayList底层使用***动态数组***(JDK1.8后去掉私有private修饰符来简化嵌套类访问)—— ***transient ?*** 为Java变量修饰符，当我们序列化对象时，如果对象中某个属性不进行序列化操作，那么在该属性前添加transient修饰符即可实现.那么，为什么ArrayList不想对elementData属性进行序列化呢？elementData可是集合中保存元素的数组啊，如果不序列化elementData属性，那么在反序列化时候，岂不是丢失了原先的元素？例如：我们创建了new Object[10]数组对象，但是我们只向其中添加了1个元素，而剩余的9个位置并没有添加元素。当我们进行序列化时，并不会只序列化其中一个元素，而是将整个数组进行序列化操作，那些没有被元素填充的位置也进行了序列化操作，间接的浪费了磁盘的空间，以及程序的性能。所以，ArrayList才会在elementData属性前加上transient修饰符。但是ArrayList 实现了 writeObject() 和 readObject() 来控制只序列化数组中有元素填充那部分内容。


3. ArrayList提供了三个构造函数：（1）ArrayList()：默认构造函数，提供初始容量为10的空列表。（2）ArrayList(int initialCapacity)：构造一个具有指定初始容量的空列表。（3）ArrayList(Collection<? extends E> c)：构造一个包含指定 collection 的元素的列表，这些元素是按照该 collection 的迭代器返回它们的顺序排列的。


#### （二）、增  ####

ArrayList增加元素的方法很重要，我们都知道ArrayList底层是由数组实现的，并且可以随着元素的增加而动态扩容，那么具体是如何实现的呢？我们来看下相关源码：



	//添加元素e
	public boolean add(E e) {
		//判断是否需要扩容：传入当前元素大小+1
	    ensureCapacityInternal(size + 1);
	    //将对应角标下的元素赋值为e：
	    elementData[size++] = e;
	    return true;
	}

	//在ArrayList的index位置，添加元素element
    public void add(int index, E element) {
        //判断index角标的合法性：
        rangeCheckForAdd(index);
        //判断是否需要扩容：传入当前元素大小+1
        ensureCapacityInternal(size + 1);
        System.arraycopy(elementData, index, elementData, index + 1, size - index);
        elementData[index] = element;
        size++;
    }
	
	//添加一个集合元素
	public boolean addAll(Collection<? extends E> c) {
		//将c转化为数组
        Object[] a = c.toArray();
        int numNew = a.length;
		//判断是否需要扩容：传入当前元素大小+集合元素大小
        ensureCapacityInternal(size + numNew);  // Increments modCount
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
    }

	//在指定位置添加一个集合元素
	public boolean addAll(int index, Collection<? extends E> c) {
		//判断index角标的合法性：
        rangeCheckForAdd(index);
		//将c转化为数组
        Object[] a = c.toArray();
        int numNew = a.length;
		//判断是否需要扩容：传入当前元素大小+集合元素大小
        ensureCapacityInternal(size + numNew);  // Increments modCount
		// 计算需要移动的长度（index之后的元素个数）
        int numMoved = size - index;
		// 数组复制，空出第index到index+numNum的位置，即将数组index后的元素向右移动numNew个位置
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
		// 将要插入的集合元素复制到数组空出的位置中
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
    }

	//得到最小扩容量
	private void ensureCapacityInternal(int minCapacity) {
	    //如果此时ArrayList是空数组,则将最小扩容大小设置为10：
	    if (elementData == EMPTY_ELEMENTDATA) {
	        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
	    }
	    //判断是否需要扩容：
	    ensureExplicitCapacity(minCapacity);
	}
	//判断是否需要扩容
	private void ensureExplicitCapacity(int minCapacity) {
	    //操作数+1
	    modCount++;
	    //判断最小扩容容量-数组大小是否大于0：
	    if (minCapacity - elementData.length > 0)
	        //扩容：
	        grow(minCapacity);
	}
	//ArrayList动态扩容的核心方法:
	private void grow(int minCapacity) {
	    //获取现有数组大小：
	    int oldCapacity = elementData.length;
	    //位运算，得到新的数组容量大小，为原有的1.5倍：
	    int newCapacity = oldCapacity + (oldCapacity >> 1);
	    //如果新扩容的大小依旧小于传入的容量值，那么将传入的值设为新容器大小：
	    if (newCapacity - minCapacity < 0)
	        newCapacity = minCapacity;
	
	    //如果新容器大小，大于ArrayList最大长度：
	    if (newCapacity - MAX_ARRAY_SIZE > 0)
	        //计算出最大容量值：
	        newCapacity = hugeCapacity(minCapacity);
	    //数组复制：
	    elementData = Arrays.copyOf(elementData, newCapacity);
	}
	//计算ArrayList最大容量：
	private static int hugeCapacity(int minCapacity) {
	    if (minCapacity < 0)
	        throw new OutOfMemoryError();
	    //如果新的容量大于MAX_ARRAY_SIZE。将会调用hugeCapacity将int的最大值赋给newCapacity:
	    return (minCapacity > MAX_ARRAY_SIZE) ?
	            Integer.MAX_VALUE :
	            MAX_ARRAY_SIZE;
	}


在JDK1.7中当添加元素时，ensureCapacityInternal（）方法会计算ArrayList需要的最小扩容大小（默认为10），然后通过比较最小扩容大小和当前数组长度来判断是否需要扩容，如果需要扩容，则进行扩容操作（***默认的扩容后大小为原来的1.5倍***（JDK1.7、JDK1.8是通过"oldCapacity + (oldCapacity >> 1"这种位运算来计算的，JDK1.6是直接通过"oldCapacity * 3)/2 + 1"直接来计算的。如果计算出来的扩容后大小依然不能满足最小扩容大小，则再扩大为按照最小扩容大小来扩容）。而实际的扩容操作是通过***Arrays.copyOf(elementData, newCapacity)***来进行的。最后再将待添加的元素添加到扩容后新的数组中。

#### （三）、删  ####


	//在ArrayList的移除index位置的元素
    public E remove(int index) {
        //检查角标是否合法：不合法抛异常
        rangeCheck(index);
        //操作数+1：
        modCount++;
        //获取当前角标的value:
        E oldValue = elementData(index);
        //获取需要删除元素 到最后一个元素的长度，也就是删除元素后，后续元素移动的个数；
        int numMoved = size - index - 1;
        //如果移动元素个数大于0 ，也就是说删除的不是最后一个元素：
        if (numMoved > 0)
            // 将elementData数组index+1位置开始拷贝到elementData从index开始的空间
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        //size减1，并将最后一个元素置为null
        elementData[--size] = null;
        //返回被删除的元素：
        return oldValue;
    }

    //在ArrayList的移除对象为O的元素，不返回被删除的元素：
    public boolean remove(Object o) {
        //如果o==null，则遍历集合，判断哪个元素为null：
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    //快速删除，和前面的remove（index）一样的逻辑
                    fastRemove(index);
                    return true;
                }
        } else {
            //同理：
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }

    //快速删除：
    private void fastRemove(int index) {
        //操作数+1
        modCount++;
        //获取需要删除元素 到最后一个元素的长度，也就是删除元素后，后续元素移动的个数；
        int numMoved = size - index - 1;
        //如果移动元素个数大于0 ，也就是说删除的不是最后一个元素：
        if (numMoved > 0)
            // 将elementData数组index+1位置开始拷贝到elementData从index开始的空间
            System.arraycopy(elementData, index+1, elementData, index, numMoved);
        //size减1，并将最后一个元素置为null
        elementData[--size] = null;
    }

	/检查角标是否合法：不合法抛异常
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }





- remove(int index)是针对于角标来进行删除，不需要去遍历整个集合，效率更高；
- remove(Object o)是针对于对象来进行删除，需要遍历整个集合进行equals()方法比对，所以效率较低；

不过，无论是哪种形式的删除，最终都会调用System.arraycopy()方法进行数组复制操作，所以效率都会受到影响；


#### （四）、查  ####


	//获取index位置的元素
	public E get(int index) {
	    //检查index是否合法：
	    rangeCheck(index);
	    //获取元素：
	    return elementData(index);
	}
	//获取数组index位置的元素：返回时类型转换
	E elementData(int index) {
	    return (E) elementData[index];
	}

ArrayList的get(int index)操作是通过elementData()方法获取对应角标元素，在返回时候进行类型转换（ArrayList查找操作实质是通过数组下标获取对应数组元素的方式实现的，这也是ArrayList查找操作效率比较高的原因所在）；

#### （五）、改  ####

	//设置index位置的元素值了element，返回该位置的之前的值
	public E set(int index, E element) {
	    //检查index是否合法：判断index是否大于size
	    rangeCheck(index);
	    //获取该index原来的元素：
	    E oldValue = elementData(index);
	    //替换成新的元素：
	    elementData[index] = element;
	    //返回旧的元素：
	    return oldValue;
	}


由于ArrayList实现了RandomAccess，所以具备了随机访问特性，调用elementData()可以获取到对应元素的值；


#### （六）、迭代器实现  ####

	//返回一个Iterator对象，Itr为ArrayList的一个内部类，其实现了Iterator<E>接口
    public Iterator<E> iterator() {
        return new java.util.ArrayList.Itr();
    }

    //其中的Itr是实现了Iterator接口，同时重写了里面的hasNext()，next()，remove()等方法；
    private class Itr implements Iterator<E> {
        int cursor; //类似游标，指向迭代器下一个值的位置
        int lastRet = -1; //迭代器最后一次取出的元素的位置。
        int expectedModCount = modCount;//Itr初始化时候ArrayList的modCount的值。
        //modCount用于记录ArrayList内发生结构性改变的次数，
        // 而Itr每次进行next或remove的时候都会去检查expectedModCount值是否还和现在的modCount值，
        // 从而保证了迭代器和ArrayList内数据的一致性。

        //利用游标，与size之前的比较，判断迭代器是否还有下一个元素
        public boolean hasNext() {
            return cursor != size;
        }

        //迭代器获取下一个元素：
        public E next() {
            //检查modCount是否改变：
            checkForComodification();

            int i = cursor;

            //游标不会大于等于集合的长度：
            if (i >= size)
                throw new NoSuchElementException();

            Object[] elementData = java.util.ArrayList.this.elementData;
            //游标不会大于集合中数组的长度：
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            //游标+1
            cursor = i + 1;
            //取出元素：
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            //检查modCount是否改变：防止并发操作集合
            checkForComodification();
            try {
                //删除这个元素：
                java.util.ArrayList.this.remove(lastRet);
                //删除后，重置游标，和当前指向元素的角标 lastRet
                cursor = lastRet;
                lastRet = -1;
                //重置expectedModCount：
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        //并发检查：在Itr初始化时，将modCount赋值给了expectedModCount
        //如果后续modCount还有变化，则抛出异常，所以在迭代器迭代过程中，不能调List增删操作；
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }




- modCount: 在Itr迭代器初始化时,将ArrayList的modCount属性的值赋值给了expectedModCount。通过阅读ArrayList源码可发现在ArrayList进行增删改时，modCount会随着每一次的操作而+1，modCount记录了ArrayList内发生改变的次数。当迭代器在迭代时，会判断expectedModCount的值是否还与modCount的值保持一致，如果不一致则抛出异常，所以在迭代器迭代过程中不能调用List的增删改操作。（ 这就是Fail-Fast机制）


#### （七）、其他  ####

	//将ArrayList的数组大小，变更为实际元素大小：
    public void trimToSize() {
        //操作数+1
        modCount++;
        //如果集合内元素的个数，小于数组的长度，那么将数组中空余元素删除
        if (size < elementData.length) {
            elementData = Arrays.copyOf(elementData, size);
        }
    }

 
	//将ArrayList里面的元素赋值到一个数组中去 生成Object数组：
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }

	//序列化写入：
    private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException{
        int expectedModCount = modCount;
        s.defaultWriteObject();
        s.writeInt(size);
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    // 序列化读取：
    private void readObject(java.io.ObjectInputStream s) throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        s.defaultReadObject();
        s.readInt();
        if (size > 0) {
            ensureCapacityInternal(size);
            Object[] a = elementData;
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }



### 二、Vector ###

我们知道Vector与ArrayList类似，底层也是基于动态数组，与ArrayList主要有两点不同：

- Vector 是同步的（使用了 synchronized 进行同步），因此开销就比 ArrayList 要大，访问速度更慢。最好使用 ArrayList 而不是 Vector，因为同步操作完全可以由程序员自己来控制。（可以使用 Collections.synchronizedList()，得到一个线程安全的 ArrayList；也可以使用 concurrent 并发包下的 CopyOnWriteArrayList 类。）

		public synchronized boolean add(E e) {
		    modCount++;
		    ensureCapacityHelper(elementCount + 1);
		    elementData[elementCount++] = e;
		    return true;
		}
		
		public synchronized E get(int index) {
		    if (index >= elementCount)
		        throw new ArrayIndexOutOfBoundsException(index);
		
		    return elementData(index);
		}

- Vector 每次扩容请求其大小的 2 倍空间，而 ArrayList 是 1.5 倍。

### 三、LinkedList ###

![](https://i.imgur.com/synrDIQ.png)


#### （一）、底层构造实现 ####

	public class LinkedList<E>
	        extends AbstractSequentialList<E>
	        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
	
	    //LinkedList的元素个数：
	    transient int size = 0;
	
	    //LinkedList的头结点：Node内部类
	    transient java.util.LinkedList.Node<E> first;//头结点
	
	    //LinkedList尾结点：Node内部类
	    transient java.util.LinkedList.Node<E> last;//尾结点
	
	    //空实现：头尾结点均为null，链表不存在
	    public LinkedList() {
	    }
	
	    //调用添加方法：
	    public LinkedList(Collection<? extends E> c) {
	        this();
	        addAll(c);
	    }
	    
	    //节点的数据结构，包含前后节点的引用和当前节点
	    private static class Node<E> {
	        //结点元素：
	        E item;
	        //结点后指针
	        java.util.LinkedList.Node<E> next;
	        //结点前指针
	        java.util.LinkedList.Node<E> prev;
	
	        Node(java.util.LinkedList.Node<E> prev, E element, java.util.LinkedList.Node<E> next) {
	            this.item = element;
	            this.next = next;
	            this.prev = prev;
	        }
	    }
	}


在LinkedList中，内部类Node对象最为重要，它组成了LinkedList集合的整个链表，分别指向上一个点、下一个结点，存储着集合中的元素；

![](https://github.com/CyC2018/CS-Notes/raw/master/pics/49495c95-52e5-4c9a-b27b-92cf235ff5ec.png)


#### （二）、数据结构之链表基础操作 ####

		//first节点插入新元素e
	    private void linkFirst(E e) {
	        //头结点：
	        final java.util.LinkedList.Node<E> f = first;
	        //创建一个新节点，并指向头结点f：
	        final java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(null, e, f);
	        //将新节点赋值给头节点：
	        first = newNode;
	        //如果头节点为null，则是第一个元素插入，将新节点也设置为尾结点；
	        if (f == null)
	            last = newNode;
	        else
	            //将之前的头结点的前指针指向新的结点：
	            f.prev = newNode;
	        //长度+1
	        size++;
	        //操作数+1
	        modCount++;
	    }



		//last节点插入新元素e
	    void linkLast(E e) {
	        //将尾结点赋值个体L:
	        final java.util.LinkedList.Node<E> l = last;
	        //创建新的结点，将新节点的前指针指向l:
	        final java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(l, e, null);
	        //新节点置为尾结点：
	        last = newNode;
	        //如果尾结点l为null：则是空集合新插入
	        if (l == null)
	            //头结点也置为 新节点：
	            first = newNode;
	        else
	            //l节点的后指针指向新节点：
	            l.next = newNode;
	        //长度+1
	        size++;
	        //操作数+1
	        modCount++;
	    }



		//在succ前插入 新元素e：
	    void linkBefore(E e, java.util.LinkedList.Node<E> succ) {
	        //获取被插入元素succ的前指针元素：
	        final java.util.LinkedList.Node<E> pred = succ.prev;
	        //创建新增元素节点，前指针 和 后指针分别指向对应元素：
	        final java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(pred, e, succ);
	        succ.prev = newNode;
	        //succ的前指针元素可能为null，为null的话说明succ是头结点，则把新建立的结点置为头结点：
	        if (pred == null)
	            first = newNode;
	        else
	            //succ前指针不为null，则将前指针的结点的后指针指向新节点：
	            pred.next = newNode;
	        //长度+1
	        size++;
	        //操作数+1
	        modCount++;
	    }



		//删除LinkedList的头结点f
	    private E unlinkFirst(java.util.LinkedList.Node<E> f) {
	        //f为头结点：获取头结点元素E
	        final E element = f.item;
	        //获取头结点的尾指针结点：
	        final java.util.LinkedList.Node<E> next = f.next;
	        //将头结点元素置为null：
	        f.item = null;
	        //头结点尾指针置为null：
	        f.next = null;
	        //将头结点的尾指针指向的结点next置为first
	        first = next;
	        //尾指针指向结点next为null的话，就将尾结点也置为null（LinkedList中只有一个元素时出现）
	        if (next == null)
	            last = null;
	        else
	            //将尾指针指向结点next的 前指针置为null；
	            next.prev = null;
	        // 长度减1
	        size--;
	        //操作数+1
	        modCount++;
	        //返回被删除的元素：
	        return element;
	    }
	
	   

	    //删除LinkedList的尾结点l
	    private E unlinkLast(java.util.LinkedList.Node<E> l) {
	        final E element = l.item;
	        final java.util.LinkedList.Node<E> prev = l.prev;
	        l.item = null;
	        l.prev = null; // help GC
	        last = prev;
	        if (prev == null)
	            first = null;
	        else
	            prev.next = null;
	        size--;
	        modCount++;
	        return element;
	    }
	
	    

	    //移除LinkedList结点
	    E unlink(java.util.LinkedList.Node<E> x) {
	        //获取被删除结点的元素E：
	        final E element = x.item;
	        //获取被删除元素的后指针结点：
	        final java.util.LinkedList.Node<E> next = x.next;
	        //获取被删除元素的前指针结点：
	        final java.util.LinkedList.Node<E> prev = x.prev;
	
	        //被删除结点的 前结点为null的话：
	        if (prev == null) {
	            //将后指针指向的结点置为头结点
	            first = next;
	        } else {
	            //前置结点的  尾结点指向被删除的next结点；
	            prev.next = next;
	            //被删除结点前指针置为null:
	            x.prev = null;
	        }
	        //对尾结点同样处理：
	        if (next == null) {
	            last = prev;
	        } else {
	            next.prev = prev;
	            x.next = null;
	        }
	
	        x.item = null;
	        size--;
	        modCount++;
	        return element;
	    }


	//获取对应角标所属于的结点：
		java.util.LinkedList.Node<E> node(int index) {
		    //位运算：如果位置索引小于列表长度的一半，则从头开始遍历；否则，从后开始遍历；
		    if (index < (size >> 1)) {
		        java.util.LinkedList.Node<E> x = first;
		        //从头结点开始遍历：遍历的长度就是index的长度，获取对应的index的元素
		        for (int i = 0; i < index; i++)
		            x = x.next;
		        return x;
		    } else {
		        //从集合尾结点遍历：
		        java.util.LinkedList.Node<E> x = last;
		        //同样道理：
		        for (int i = size - 1; i > index; i--)
		            x = x.prev;
		        return x;
		    }
		}


这些数据结构中的链表基本操作将是LinkedList常用操作的基础，是整个链表操作的精髓所在。同样，废话不多说，代码见真章，走起！！！



#### （三）、增 ####


	//LinkedList添加首结点 first：
    public void addFirst(E e) {
        linkFirst(e);
    }


	//添加元素：添加到最后一个结点；
	public boolean add(E e) {
	    linkLast(e);
	    return true;
	}


	//向对应角标添加元素：
	public void add(int index, E element) {
	    //检查传入的角标 是否正确：
	    checkPositionIndex(index);
	    //如果插入角标==集合长度，则插入到集合的最后面：
	    if (index == size)
	        linkLast(element);
	    else
	        //插入到对应角标的位置：获取此角标下的元素先
	        linkBefore(element, node(index));
	}

	//添加集合：从最后size所在的index开始：
	    public boolean addAll(Collection<? extends E> c) {
	        return addAll(size, c);
	    }
	    //LinkedList添加集合：
	    public boolean addAll(int index, Collection<? extends E> c) {
	        //检查角标是否正确：
	        checkPositionIndex(index);
	        Object[] a = c.toArray();
	        int numNew = a.length;
	        if (numNew == 0)
	            return false;
	        java.util.LinkedList.Node<E> pred, succ;
	        if (index == size) {
	            succ = null;
	            pred = last;
	        } else {
	            succ = node(index);
	            pred = succ.prev;
	        }
	        for (Object o : a) {
	            E e = (E) o;
	            java.util.LinkedList.Node<E> newNode = new java.util.LinkedList.Node<>(pred, e, null);
	            if (pred == null)
	                first = newNode;
	            else
	                pred.next = newNode;
	            pred = newNode;
	        }
	        if (succ == null) {
	            last = pred;
	        } else {
	            pred.next = succ;
	            succ.prev = pred;
	        }
	        size += numNew;
	        modCount++;
	        return true;
	    }




LinkedList 的添加方法主要分为两种，一是直接添加一个元素，而是在指定角标小添加一个元素：

- add(E e)底层调用linkLast(E e)方法，就是在链表的最后面插入一个元素；
- add(int index, E element)，插入的角标如果==size，则插入到链表最后；否则，按照角标大小插入到对应位置；


对于LinkedList集合添加元素步骤可以归纳为：


1. 将添加的元素转换为新建的一个LinkedList的Node对象节点；
2. 增加新建的Node节点的前后引用，即该Node节点的prev、next属性，让其分别指向哪一个节点）；
3. 修改新建的Node节点的前后Node节点中pre/next属性，使其指向该新建的节点。

![](http://upload-images.jianshu.io/upload_images/5621908-b9b9a5292a25c6cd.jpg?imageMogr2/auto-orient/strip)


#### （四）、删 ####

	//移除首个结点：如果集合还没有元素则报错
	    public E removeFirst() {
	        final java.util.LinkedList.Node<E> f = first;
	        //首节点为null，则抛出异常；
	        if (f == null)
	            throw new NoSuchElementException();
	        return unlinkFirst(f);
	    }


	//移除最后一个结点：如果集合还没有元素则报错
	    public E removeLast() {
	        //获取最后一个结点：
	        final java.util.LinkedList.Node<E> l = last;
	        if (l == null)
	            throw new NoSuchElementException();
	        //删除尾结点：
	        return unlinkLast(l);
	    }


	//删除对应角标的元素：
	public E remove(int index) {
	    checkElementIndex(index);
	    //node()方法通过角标获取对应的元素，在后面介绍
	    return unlink(node(index));
	}
	

	//删除LinkedList中的元素，可以删除为null的元素，逐个遍历LinkedList的元素，重复元素只删除第一个：
	public boolean remove(Object o) {
	    //如果删除元素为null：
	    if (o == null) {
	        for (java.util.LinkedList.Node<E> x = first; x != null; x = x.next) {
	            if (x.item == null) {
	                unlink(x);
	                return true;
	            }
	        }
	    } else {
	        //如果删除元素不为null：
	        for (java.util.LinkedList.Node<E> x = first; x != null; x = x.next) {
	            if (o.equals(x.item)) {
	                unlink(x);
	                return true;
	            }
	        }
	    }
	    return false;
	}




#### （五）、查 ####

	//得到首个结点：集合没有元素报错
	    public E getFirst() {
	        //获取first结点：
	        final java.util.LinkedList.Node<E> f = first;
	        if (f == null)
	            throw new NoSuchElementException();
	        return f.item;
	    }
	

	    //得到最后一个结点：集合没有元素报错
	    public E getLast() {
	        //获取last结点：
	        final java.util.LinkedList.Node<E> l = last;
	        if (l == null)
	            throw new NoSuchElementException();
	        return l.item;
	    }



	//获取相应角标的元素：
		public E get(int index) {
		    //检查角标是否正确：
		    checkElementIndex(index);
		    //获取角标所属结点的 元素值：
		    return node(index).item;
		}


LinkedList查找操作是通过node(int index)获取到对应节点后，返回节点中的item属性，该属性就是我们所保存的元素。

可以看到，node(int index)中是根据角标的大小选择从前遍历还是从后遍历整个集合。也可以间接的说明，LinkedList在随机获取元素时性能很低，每次的获取都得从头或者从尾遍历半个集合。

#### （六）、改 ####

	//设置对应角标的元素：
	public E set(int index, E element) {
	    checkElementIndex(index);
	    //通过node()方法，获取到对应角标的元素：
	    java.util.LinkedList.Node<E> x = node(index);
	    E oldVal = x.item;
	    x.item = element;
	    return oldVal;
	}
	
LinkedList修改你操作也是调用node(int index)方法通过对应角标获取到对应的集合元素，然后再进行元素修改。




#### （七）、迭代器 ####

LinkedList并没有自己实现iterator()方法，而是使用其父类AbstractSequentialList的iterator()方法：

	public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {            
	    public ListIterator<E> listIterator() {
	        return listIterator(0);
	    }
	}


AbstractSequentialList的iterator()方法又调用了AbstractSequentialList的父类AbstractList中的 listIterator()：

	public ListIterator<E> listIterator(int index) {
	    checkPositionIndex(index);
	    return new ListItr(index);
	}
	
	private class ListItr implements ListIterator<E> {...略...}



#### （八）、队列相关操作 ####


		//出队，获取第一个元素，不出队列，只是获取
	    // 队列先进先出；不存在不抛异常，返回null
	    public E peek() {
	        //获取头结点：
	        final java.util.LinkedList.Node<E> f = first;
	        //存在获取头结点元素，不存在返回null
	        return (f == null) ? null : f.item;
	    }
	
	    //出队，并移除第一个元素；不存在不抛异常。
	    public E poll() {
	        final java.util.LinkedList.Node<E> f = first;
	        return (f == null) ? null : unlinkFirst(f);
	    }
	
	    //出队（删除第一个结点），如果不存在会抛出异常而不是返回null，存在的话会返回值并移除这个元素（节点）
	    public E remove() {
	        return removeFirst();
	    }
	
	    //入队(插入最后一个结点)从最后一个元素
	    public boolean offer(E e) {
	        return add(e);
	    }
	
	    //入队（插入头结点），始终返回true
	    public boolean offerFirst(E e) {
	        addFirst(e);
	        return true;
	    }
	
	    //入队（插入尾结点），始终返回true
	    public boolean offerLast(E e) {
	        addLast(e);
	        return true;
	    }
	
	    //出队（从前端），获得第一个元素，不存在会返回null，不会删除元素（节点）
	    public E peekFirst() {
	        final java.util.LinkedList.Node<E> f = first;
	        return (f == null) ? null : f.item;
	    }
	
	    //出队（从后端），获得最后一个元素，不存在会返回null，不会删除元素（节点）
	    public E peekLast() {
	        final java.util.LinkedList.Node<E> l = last;
	        return (l == null) ? null : l.item;
	    }
	
	    //出队（从前端），获得第一个元素，不存在会返回null，会删除元素（节点）
	    public E pollFirst() {
	        final java.util.LinkedList.Node<E> f = first;
	        return (f == null) ? null : unlinkFirst(f);
	    }
	
	    //出队（从后端），获得最后一个元素，不存在会返回null，会删除元素（节点）
	    public E pollLast() {
	        final java.util.LinkedList.Node<E> l = last;
	        return (l == null) ? null : unlinkLast(l);
	    }
	
	    //入栈，从前面添加  栈 后进先出
	    public void push(E e) {
	        addFirst(e);
	    }
	
	    //出栈，返回栈顶元素，从前面移除（会删除）
	    public E pop() {
	        return removeFirst();
	    }


LinkedList实现了Deque接口，实现了队列相关的操作。（这里不再详述Deque，有机会在Deque中我们再见）



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


