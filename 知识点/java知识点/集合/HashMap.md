## HashMap

> HashMap作为一种优秀的数据存储结构，本质是一种(key,value)结构，它结合的数组和链表的优点对数据进行存储，通过对数据key的hashcode计算分配到对应的数组中，如果出现碰撞，使用链地址法将数据转化为链表，在Java8中，当链表长度大于8时转化为红黑树。

![image-20200724175245615](图库/HashMap/image-20200724175245615.png)

### 优点

- 数组结构在查询和插入删除复杂度方面分别为O(1)和O(n)
- 链表结构在查询和插入删除复杂度方面分别为O(n)和O(1)
- 二叉树做了平衡 两者都为O(lgn)
- 而哈希表两者结合了数组和链表的优点，在大部分情况下基本查询都为O(1)，就算数据很大产生了很长的链表，那么在转换成红黑树的情况下查询和插入的速度都基本为O(logN)

### 属性

```java
  	/**
     * 默认的初始化桶容量，必须为2的次幂
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * 最大容量为2^30 依旧是2的次幂
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 扩容加载因子，例如初始值16在有16 * 0.75 = 12对键值对的情况下就要扩容
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 当桶中的链表太长时，会将其自动转换成红黑树，
     * 这个值就是链表的转换点，当链表长度大于8时转换成红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * 在哈希表扩容时，如果发现链表长度小于 6，则会由树重新退化为链表。
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * 红黑树化的最小值，想要将链表中的数据转化成红黑树，键值对数量最少要64
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

    /**
     * 也就是桶，存储后续链表的头结点，也是红黑树的root节点
     * 该数组在存入数据的时候才开始初始化，创建HashMap的时候并不会初始化
     */
    transient Node<K,V>[] table;
```

我们通过[Hash算法](./Hash算法.md)可以让每个数据尽可能平均的分配到每个桶中，理想状态下哈希表的每个箱子中，元素的数量遵守泊松分布：

![image-20200724174947608](图库/HashMap/image-20200724174947608.png)

当负载因子为0.75时，上述公式中的$\lambda $约等于 0.5，因此箱子中元素个数和概率的关系如下:

| 数量 | 概率       |
| ---- | ---------- |
| 0    | 0.60653066 |
| 1    | 0.30326533 |
| 2    | 0.07581633 |
| 3    | 0.01263606 |
| 4    | 0.00157952 |
| 5    | 0.00015795 |
| 6    | 0.00001316 |
| 7    | 0.00000094 |
| 8    | 0.00000006 |

这就是为什么箱子中链表长度超过 8 以后要变成红黑树，因为在正常情况下出现这种现象的几率小到忽略不计。一旦出现，几乎可以认为是哈希函数设计有问题导致的。

### 方法

#### 1.put

```java
   public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    /**
     * @param onlyIfAbsent 为true时，key相同的情况不下更新value值
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
   final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
       	//如果table还没初始化，先初始化table，resize()方法用于初始化或者扩容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //i = (n - 1) & hash]通过hash方法取模后获得桶的下标，如果为空就存入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //如果当前点hash key value值都想同，说明这个要存的值在Hashmap中已经存在了，后面略过
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                //如果红黑树中存在改节点则返回，否则创建新节点加入树后返回null
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //循环遍历，直到遍历到链表尾部插入或者找到一个键值对相同的节点
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度大于8(包含了头结点)，那么需要把链表红黑树化
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果Map中存在key对应的Node节点
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //默认是需要更新value值，或者原来的value值为null时也要更新value值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
       //和ArrayList一样，因为是线程不安全的，所以需要用modCount来计算版本号
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

##### 1.1红黑树化

```java
 final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
     	//前面说了转换成红黑树最少需要64个数据，当小于这个数据时，就只进行扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            //hd代表头节点，tl代表尾节点，形成一个双向链表
            TreeNode<K,V> hd = null, tl = null;
            do {
                //将链表中的结点转换为树结点，形成一个TreeNode形式的新链表
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
             //将新的树结点链表原来的位置
            if ((tab[index] = hd) != null)
                //在前面我们已经把新节点加到链表最后了，这里的hd是树节点形成的链表的头结点
                //调用treeify把链表转换成树
                hd.treeify把链表转换成树(tab);
        }
    }
	TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        return new TreeNode<>(p.hash, p.key, p.value, next);
    }
```

#### 2.get

```java
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //桶数组存在，且长度大于0且该桶中存在节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //如果第一个节点就相同就返回，否则就红黑树节点或者链表中遍历查找
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

#### 3.remove

```java
    public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    /**
     * Implements Map.remove and related methods
     * @param value the value to match if matchValue, else ignored
     * @param matchValue为true时，只有value也相等时才可以删除
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //照例判断桶数组是否为空，然后找到对应的桶中的头结点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //相同就删除
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    //从红黑树中找对应点，没找到则为null
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    //遍历链表找点
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
            //这里判断相等还要用Value类型自身的equal方法，这里是为了适配自定义的equal方法
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
  final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
            int n;
            if (tab == null || (n = tab.length) == 0)
                return;
            int index = (n - 1) & hash;
      		//first为这个桶的头结点
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
      		// succ为这个node的next节点，pred为前驱节点
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
      		//如果前驱节点为null，说明这个节点为头结点，所以把下一个succ赋值给头结点
            if (pred == null)
                tab[index] = first = succ;
            else
                //否则让前驱的next节点跳过当前节点，直接链接next节点，达到删除的目的
                pred.next = succ;
      		//和前面的pred.next = succ;对应
            if (succ != null)
                succ.prev = pred;
            if (first == null)
                return;
      		//拿到红黑树的root节点
            if (root.parent != null)
                root = root.root();
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  // too small
                return;
            }
      		//上面这行判断是判断当前的红黑树数量是否在【0，6】之间，因为最极端的情况下是这种树
      		//					x
      		//				  /   \
      		//				 x     x
      		//                \   / \ 
      		//                 x x   x
      		//
      		//也就是当小于6个节点时，把树转换为链表，其余树退化的地方只有在resize扩容的时候
      		
      //...下面是红黑树过程略过
  }
```

#### resize

```java
  final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
      	//原扩容阈值，就是容量乘以扩容系数(0.75)
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //当大于2^30时，阈值直接设为Integer.MAX_VALUE;再返回原来的，再超过就报错
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //如果两倍的新容量小于最大且原容量大于等于16，就扩容一杯
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY(2^30) &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY(16))
                newThr = oldThr << 1; // double threshold
        }
       //oldCap为空表示当前没有数据，oldThr大于0表示有扩容阈值
       //然而扩容阈值是下面这行设置的，那么哪里还能设置呢？
       //当调用了   public HashMap(int initialCapacity) 传入了我们自己的需要的初始容量时
       //HashMap会先根据这个initialCapacity把容量调成2的次幂，在设置threshold
       //所以这里的newCap就等于是我们自己设置的
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //普通的HashMap初始化，设置默认的新容量和初始值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
       //对临界值做判断，确保其不为0，因为在上面第二种情况(oldThr > 0)，并没有计算newThr
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
         //构造新表，初始化表中数据
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
       //将刚创建的新表赋值给table
        table = newTab;
        if (oldTab != null) {
             //遍历将原来table中的数据放到扩容后的新表中来
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //没有链表Node节点，直接放到新的table中下标为【e.hash & (newCap - 1)】位置即可
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //如果e后面还有链表节点，则遍历e所在的链表，注意保证顺序
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
 /**
  * newTab的容量是以前旧表容量的两倍,因为数组table下标并不是根据循环逐步递增
  * 的，而是通过（table.length-1）& hash计算得到，因此扩容后，存放的位置就
  * 可能发生变化，那么到底发生怎样的变化呢，就是由下面的算法得到.
  *
  * 通过e.hash & oldCap来判断节点位置通过再次hash算法后，是否会发生改变，
  * 如果是小于原容量的数据下标，如下，那么在新容量中的下标也不会变
  * e.hash = 13 二进制：0000 1101
  * oldCap = 16 二进制：0001 0000
  * newCap = 32 二进制：0010 0000
  *  &运算：  0  二进制：0000 0000
  * 结论：元素位置在扩容后不会发生改变
  */
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                       /**
                         * 如果原hash大于原容量小于新容量，那么下标数组就会改变
                         * e.hash = 18 二进制：0001 0010
                         * oldCap = 16 二进制：0001 0000
                         * &运算： = 16 二进制：0001 0000
                         * 结论：元素位置在扩容后会发生改变，那么如何改变呢？
                         * newCap = 32 二进制：0010 0000
                         * 通过(newCap-1)&hash
                         * 即0001 1111 & 0001 0010 得0001 0010，16+2 = 18
                         * 也即当hash在新容量中的下标为  oldCap+在oldCap容量中的下标          
                         */
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                        /**
                         * 若(e.hash & oldCap) == 0，下标不变，将原表某个下标的元素放到扩容表同样
                         * 下标的位置上
                         */
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        /**
                         * 若(e.hash & oldCap) != 0，将原表某个下标的元素放到扩容表中
                         * [下标+增加的扩容量]的位置上
                         */
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
```

扩容过程中有几个注意点

- HashMap初始化的时候并不会创建桶数组table，而是在第一次put的时候调用resize初始化

- 如果创建HashMap的时候传入了初始容量和扩容因子会把初始容量近似计算成2的次幂
- 扩容过程中需要把原数组下标移到新位置，这里分成了low和hight两个类。（e.hash & oldCap) == 0表示的是hash值在[0，16）和[32,无限)之间，不等于0表示在[16,31]之间。
  - low：表示第一种情况，这种情况下扩容后下标不变，因为当小于16时，不变，大于32时，除以32的余数也就是除以16的余数，所以下标不变
  - high：第二种情况，这种情况的新下标=原容量+原容量的下标，例如18，在原容量中是2，在新容量中就是18

- 52行的splite也是根据上面这种划分分配新的下表地址，然后判断树中节点的数量，小于6就取消树化。



