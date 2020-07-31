## ConcurrentHashMap

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
                if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
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

关于table数组，有3个重要方法

```java
    //以volatile读的方式读取table数组中的元素
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

https://www.codercto.com/a/57430.html

  ​	