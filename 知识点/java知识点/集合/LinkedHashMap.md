## LinkedHashMap

LinkedHashMap继承于HashMap，它在内部记录了插入的顺序，使得通过迭代方式取出的Map数据的顺序使按照插入的顺序排列的。

```java
        Map<String, String> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("name1", "value1");
        linkedHashMap.put("name2", "value2");
        linkedHashMap.put("name3", "value3");
        Set<Entry<String, String>> set = linkedHashMap.entrySet();
        Iterator<Entry<String, String>> iterator = set.iterator();
        while(iterator.hasNext()) {
            Entry entry = iterator.next();
            String key = (String) entry.getKey();
            String value = (String) entry.getValue();
            System.out.println("key:" + key + ",value:" + value);
        }
// 输出   key:name1,value:value1
//       key:name2,value:value2
//       key:name3,value:value3
```

他的使用方式和HashMap一模一样，其内部大部分属性也和HashMap基本一致，下面列举的是不一样的内部属性。

### 属性

```java
/**
 * The head (eldest) of the doubly linked list.
 */
transient LinkedHashMapEntry<K,V> head;

/**
 * The tail (youngest) of the doubly linked list.
 */
transient LinkedHashMapEntry<K,V> tail;

//访问排序，初始化默认为false，如果为true的话，
//调用例如get访问过的某个键值对就会移到上面双向链表的最后一 个
//lrucache使用的原理就是使用LinkedHashMap然后把accessOrdersheweitrue，把最近访问的放在最后，然后
//从头部删除不常用的Node
final boolean accessOrder;
```

有意思的一点是它的桶数据结构继承了``HashMap``的``Node``,然后通过头尾两个指针来实现双向链表搞定插入排序。

```java
    static class LinkedHashMapEntry<K,V> extends HashMap.Node<K,V> {
        LinkedHashMapEntry<K,V> before, after;
        LinkedHashMapEntry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    private static final long serialVersionUID = 3801124242820219131L;

    /**
     * The head (eldest) of the doubly linked list.
     */
    transient LinkedHashMapEntry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.
     */
    transient LinkedHashMapEntry<K,V> tail;
```

然后``HashMap``的``TreeNode``又是继承它的``LinkedHashMapEntry``,因为``TreeNode``要实现在链表和红黑树之间的相互转换，所以它内部也维护了一条双向链表。

```java
    static final class TreeNode<K,V> extends LinkedHashMap.LinkedHashMapEntry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
```

### 方法

#### 1.put

 ```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
 ```

``LinkedHashMap``没有put方法，直接调用了父类HashMap的putVal方法，那这两个区别在哪呢？

```java
 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
			...
            tab[i] = newNode(hash, key, value, null);
			...
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            ->  TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            ...
    }
```

差别点就在newNode和newTreeNode这里

```java
  // Create a regular (non-tree) node
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
    }

    // For conversion from TreeNodes to plain nodes
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        return new Node<>(p.hash, p.key, p.value, next);
    }

    // Create a tree bin node
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        return new TreeNode<>(hash, key, value, next);
    }

    // For treeifyBin
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        return new TreeNode<>(p.hash, p.key, p.value, next);
    }
--------------------------------LinkedHashMap------------------------------------------

    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMapEntry<K,V> p =
            new LinkedHashMapEntry<K,V>(hash, key, value, e);
        linkNodeLast(p);
        return p;
    }

    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMapEntry<K,V> q = (LinkedHashMapEntry<K,V>)p;
        LinkedHashMapEntry<K,V> t =
            new LinkedHashMapEntry<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);
        return t;
    }

    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
        linkNodeLast(p);
        return p;
    }

    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMapEntry<K,V> q = (LinkedHashMapEntry<K,V>)p;
        TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);
        return t;
    }
```

可以看到，HashMap就是直接创建了一个Node和TreeNode，但是LinkedHashMap复写了这两个方法，在其中做了其他内容给。

```java
    // link at the end of list   
	private void linkNodeLast(LinkedHashMapEntry<K,V> p) {
        LinkedHashMapEntry<K,V> last = tail;
        tail = p;
        //last为null，说明还没有数据，所以头尾都是这个新节点
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }
```

可以看到，当插入一个Node节点时，会插入到前面说过的双向链表的尾部，这样从头部遍历取出的就是最早插入的数据。插入TreeNode时的操作是一样的，只不过两者类型不一致而已。

```java
    // Callbacks to allow LinkedHashMap post-actions
	//找到了对应NOde
    void afterNodeAccess(Node<K,V> p) { }
	//插入了对应Node
    void afterNodeInsertion(boolean evict) { }
	//删除了对应Node
    void afterNodeRemoval(Node<K,V> p) { }
```

HashMap中有三个空方法，交给继承Hashmap的子类使用的，HashMap的子类全部逻辑和HashMap基本一致，他们的区别就体现在这些待实现的方法中。

```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMapEntry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        //默认为false，
        return false;
    }
```

不过这个方法不是提供给LinkedHashMap的，因为removeEldestEntry返回的永远是false，我们可以重写改为true，作用是每次从尾部添加一个新数据是就从头部删除一个老数据。

#### 2.remove

remove和put一样，全部调用的是HashMap的方法，差别就在重写的``afterNodeRemoval``

```java
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMapEntry<K,V> p =
            (LinkedHashMapEntry<K,V>)e, b = p.before, a = p.after;
        //把前驱和后继节点赋值给临时变量后设为null
        p.before = p.after = null;
        //前驱为null，说明该节点是头结点，把后继节点赋值给head即可。
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

删除对应节点。

可以看出来，``LinkedHashMap``所有对存储数据的操作全部是交给了HashMap处理，他内部只是维护处理了一个插入节点的顺序双向链表，然后在插入替换删除的时候用过重写方法对双向链表的数据做一些处理。