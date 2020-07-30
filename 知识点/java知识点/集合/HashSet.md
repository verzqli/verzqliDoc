## HashSet

和TreeSet类似，HashSet也是一个套着HashMap的壳，内部数据的存储实现依靠的是HashMap，通过存储默认的相同value和不同的key值来实现对key的去重过滤

```java
   public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    static final long serialVersionUID = -5024744406713321676L;
 	private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
    public HashSet() {
        map = new HashMap<>();
    }
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
```

原理和TreeSet一模一样。

## LinkedHashSet

如果说HashSet是套了个壳，那么LinkedHashSet就是在HashSet上又套了个壳。

```java
    public LinkedHashSet() {
        super(16, .75f, true);
    }
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

第三个参数无用，只是为了区分这是属于LinkedHashSet的构造函数，可以看到LinkedHashSet的具体实现是靠在HshSet中初始化的LinkedHashMap，原理和HashSet一模一样，只不过是两者底层处理数据的类不同而已。