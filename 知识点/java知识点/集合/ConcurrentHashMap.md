## ·ConcurrentHashMap

ConcurrentHashMap是一个线程安全的HashMap，相较于HashTable给每个方法上锁，它采用了更轻量级的分段锁来实现线程安全。

> 此哈希表的主要设计目标是保持并发可读性（通常是get（）方法，还包括迭代器和相关方法），同时最大程度地减少更新竞争。 次要目标是使空间消耗保持与java.util.HashMap相同或更好，并支持许多线程对空表的高初始插入率。

以下分析基于JDK8

jdk8直接抛弃了Segment的设计，采用了较为轻捷的Node + CAS + Synchronized设计，保证线程安全。![image-20200729145459509](图库/ConcurrentHashMap/image-20200729145459509.png)

### 属性

```java
/**
 * 箱子(桶)数组，和HashMap一样，第一次插入的时候初始化，大小为2的次幂，
 * 由迭代器直接访问
 */
transient volatile Node<K,V>[] table;

/**
 * 扩容时使用，平时为 null，只有在扩容的时候才为非 null
 */
private transient volatile Node<K,V>[] nextTable;

/**
 * Base counter value, used mainly when there is no contention,
 * but also as a fallback during table initialization
 * races. Updated via CAS.
 * 该属性保存着整个哈希表中存储的所有的结点的个数总和，有点类似于 HashMap 的 size 属性。
 */
private transient volatile long baseCount;

/**
 * 这个值比较重要，下面解释 
 */
private transient volatile int sizeCtl;

/**
 * The next table index (plus one) to split while resizing.
 * 表示扩容时的nextTable的下标
 */
private transient volatile int transferIndex;

/**
 * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
 *  // 标识当前cell数组是否在初始化或扩容中的CAS标志位
 */
private transient volatile int cellsBusy;

/**
 * Table of counter cells. When non-null, size is a power of 2.
 * 每个Cell的数量，size()方法返回的值就是遍历累加数组所有的值
 */
private transient volatile CounterCell[] counterCells;

// views
private transient KeySetView<K,V> keySet;
private transient ValuesView<K,V> values;
private transient EntrySetView<K,V> entrySet;
```

``sizeCtl``:用volatile修饰的一个变量，用与数组初始化与扩容控制

- ``sizeCtl==0``：未指定初始容量，初始容量就为默认初始容量

- ``sizeCtl>0``：
- 如果当前数组table为null时表示table正在初始化，``sizeCtl``就是要新建数组的长度。由指定的初始容量计算而来，再找最近的2的幂次方。比如传入6，计算公式为6+6/2+1=10，最近的2的幂次方为16，所以sizeCtl就为16
  - 如果当前数据已经初始化完成，表示当前数据的阈值，超过了该临界值就扩容，值为table.length*0.75(扩容因子)

- ``sizeCtl<0``：
  - -1：当前table正在初始化
  - -N：高16位表示的是当前容量，用于记录扩容时容器的大小，低16位等于M，表示当前有M-1个扩容线程

### 方法

#### 1.put

  ```java
	
	public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        //key和map都不能为null fail-fast机制
        if (key == null || value == null) throw new NullPointerException();
        //高低位混合位运算
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                //初始化table
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            /*
             * table 对应下标的头结点为 null
             * 基于 CAS 设置结点，如果成功则本次 put 操作完成，【看1.1】
             * 如果失败则说明期间有并发操作，需要进入一轮新的循环
             */
                if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))】
                   // 设置结点成功，put 操作完成
                    break;                   // no lock when adding to empty bin
            }
            //如果 Map 正在执行扩容操作（MOVED 哈希值表示正在扩容），则帮助扩容
            else if ((fh = f.hash) == MOVED)
                //后续再看
                tab = helpTransfer(tab, f);
            else {
                 // 获取到 hash 值对应下标的头结点，且结点不为 null
                V oldVal = null;
                synchronized (f) {
                    // 再次校验头结点为 f
                    if (tabAt(tab, i) == f) {
                        // 头结点的哈希值大于等于 0，说明是链表，如果是树的话应该是 -2
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果是已经存在的 key，则在允许覆盖的前提下直接覆盖已有的值
                                if (e.hash == hash &&((ek = e.key) == key ||(ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                 // 如果是不存在的 key，则直接在链表尾部插入一个新的结点
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,value, null);
                                    break;
                                }
                            }
                        }
                         // 头结点的哈希值小于0，判断节点是否是树节点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            // 调用红黑树的方法获取到修改的结点，并插入或更新结点（如果允许）
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        //一个占位符节点，为了加锁【看1.2】
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                     /*
                     * 结点数目大于等于 8，对链表执行转换操作
                     * - 如果 table 长度小于 64，则执行扩容
                     * - 如果 table 长度大于等于 64，则转换成红黑树
                     */
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }

    /**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
  ```

##### 1.1 直接操作主存tab数组

```java
  
	//以volatile读的方式读取table数组中的元素
	//直接读取主存中tab的第i个位置的数据
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
    //以volatile写的方式，将元素插入table数组
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
    //以CAS的方式，将元素插入table数组
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        //原子的执行如下逻辑：如果tab[i]==c,则设置tab[i]=v，并返回ture.否则返回false
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    
```

以上的三个方法指的是直接去主存中读取tab的值，因为volatile修饰的是table数组，他只能保证数组的引用在变化时刷新，在数组内元素变化的时候对其他线程并不具有可见性，而且这里的tab还是一个局部变量，它没有volatile修饰，所以更不具有可见性，所以需要直接操作主存。

##### 1.2 ReservationNode

这个点是一个保留节点，或者也叫空节点。``computeIfAbsent``和``compute``这两个函数式api中才会使用。它的hash值固定为-3，就是个占位符，不会保存实际的数据，正常情况是不会出现的，在jdk1.8新的函数式有关的两个方法``computeIfAbsent``和``compute``中才会出现。

ReservationNode在ComputeIfAbsent()及其相关方法中作为一个预留节点使用。computeIfAbsent()方法会先判断相应的Key值是否已经存在，如

果不存在，则调用由用户实现的自定义方法来生成Value值，组成KV键值对，随后插入此哈希集合中。在并发场景下，在从得知Key不存在到插入哈希

集合的时间间隔内，为了防止哈希槽被其他线程抢占，当前线程会使用一个ReservationNode节点放到槽中并加锁，从而保证了线程的安全性。

```java
static final class ReservationNode<K,V> extends Node<K,V> {
    ReservationNode() {
        super(RESERVED, null, null, null);
    }
    // 空节点代表这个hash桶当前为null，所以肯定找不到“相等”的节点
    Node<K,V> find(int h, Object k) {
        return null;
    }
}
```

方法的执行流程可以概括为：

1. 计算 key 的哈希值
2. 如果 table 为空，则执行初始化
3. 否则，计算 key 哈希值对应的下标，并获取 table 中对应下标的头结点
4. 如果头结点为 null，则基于 CAS 尝试添加头结点
5. 否则，如果头结点不为 null，但是头结点的哈希值为 MOVED，说明目前正在执行扩容操作，则帮助扩容
6. 否则，如果头结点不为 null，且未处于扩容状态，则尝试添加或更新结点
7. 如果当前头结点是个占位符，就抛出异常
8. 判断当前 bin 范围内结点数目是否大于阈值，如果大于阈值则执行扩容操作



#### 2.get

```java
public V get(Object key) {
     Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
 	   // 计算 key 的 hash 值	
        int h = spread(key.hashCode());
    	 // table 表不为空，且 key 对应的 table 头结点存在
        if ((tab = table) != null && (n = tab.length) > 0 &&(e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                     // 找到对应的 key，返回 value
                    return e.val;
            }
            //eh=-1，说明该节点是一个ForwardingNode，正在迁移，此时调用ForwardingNode的find方法去nextTable里找。
            //eh=-2，说明该节点是一个TreeBin，此时调用TreeBin的find方法遍历红黑树，由于红黑树有可能正在旋转变色，所以find里会有读写锁。
            //eh>=0，说明该节点下挂的是一个链表，直接遍历该链表即可
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 当前 bin 是链表，直接遍历检索
            while ((e = e.next) != null) {
                if (e.hash == h &&((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
}
```

方法首先依据相同的实现计算 key 的哈希值，然后定位 key 在 table 中的 bin 位置。如果 bin 结点存在，则依据当前 bin 类型（链表或红黑树）检索目标值。

但是可以看到get方法并没有任何一处加锁，那么get方法是怎么保证多线程并发？

```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
```

get操作可以无锁是由于Node的元素val和指针next是用volatile修饰的，在多线程环境下线程A修改结点的val或者新增节点的时候是对线程B可见的。

另外，table数组也是volatile修饰的，但是volatile修饰数组时只对其引用地址保持可见性，对其元素的改变没有引用性，那么加volatile的作用是什么？

主要是:**为了使得Node数组在扩容的时候对其他线程具有可见性而加的volatile**

- 在1.8中ConcurrentHashMap的get操作全程不需要加锁，这也是它比其他并发集合比如hashtable、用Collections.synchronizedMap()包装的hashmap安全效率高的原因之一。
- get操作全程不需要加锁是因为Node的成员val是用volatile修饰的和数组用volatile修饰没有关系。
- 数组用volatile修饰主要是保证在数组扩容的时候保证可见性。

#####  3.remove

```java
 public V remove(Object key) {
        return replaceNode(key, null, null);
    }

    /**
     * value:当 value==null 时 ，删除节点 。否则 更新节点的值为value
     * cv:一个期望值， 当cv==null:表示直接更新value/删除节点,
     * cv不为空，则只有在key的oldValue等于期望值的时候，才更新value/删除节点
     */
    final V replaceNode(Object key, V value, Object cv) {
        int hash = spread(key.hashCode());
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                boolean validated = false;
                synchronized (f) {
                    //如果这里的tabAt(tab, i)不等于f，说明在这个判断时间内其他线程修改了tab[i]，继续for循环重新来过
                    if (tabAt(tab, i) == f) {
                        //大于0位node链表
                        if (fh >= 0) {
                            validated = true;
                            for (Node<K,V> e = f, pred = null;;) {
                                K ek;
                               //找的key对应的node
                                if (e.hash == hash &&((ek = e.key) == key ||(ek != null && key.equals(ek)))) {
                                    V ev = e.val;
                                    //这里cv为null
                                    if (cv == null || cv == ev || (ev != null && cv.equals(ev))) {
                                        oldVal = ev;
                                        //更新，这里并不是
                                        if (value != null)
                                            e.val = value;
                                        //删除非头节点
                                        else if (pred != null)
                                            pred.next = e.next;
                                        //删除头结点，当pred为null时才会进入这里，e就是头结点，所以把next设为新的头结点
                                        else
                                             //因为已经获取了头结点锁，所以此时不需要使用casTabAt	
                                            setTabAt(tab, i, e.next);
                                    }
                                    break;
                                }//这里对应第一个if
                                pred = e;
                                if ((e = e.next) == null)
                                    //到达链表尾部，依旧没有找到，跳出循环
                                    break;
                            }//这里对应for循环结尾
                        }
                        else if (f instanceof TreeBin) {
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&                                                   
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                V pv = p.val;
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    if (value != null)
                                        p.val = value;
                                    else if (t.removeTreeNode(p))
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (validated) {
                    if (oldVal != null) {
                        if (value == null)
                            //减去一个容量，第二个-1表示不需要检测去扩容
                            addCount(-1L, -1);
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
```

#### 4.扩容机制



https://www.codercto.com/a/57430.html

  ​	