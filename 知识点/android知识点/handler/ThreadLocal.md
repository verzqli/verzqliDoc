## ThreadLocal

> 线程本地存储区(Thread Local Storage 简称TLS)，每个线程都有自己的私有的本地存储区域，不用线程之间不能访问对方的TLS区域。ThreadLocal并不是一个Thread，而是Thread的局部变量。

### ThreadLocalMap

ThreadLocal内部存储数据的类，虽然是一个map，但是是用Entry数组以键值对的形式进行存储，这里的key是调用set的ThreadLocal，且用弱引用修饰，value是每个线程自己存储数据的区域（ThreadLocal.ThreadLocalMap threadLocals）。

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

- key（threadlocal）用弱引用，但是value（threadlocalmap）却不是，所以当key为null被GC后，此时的value生命周期和Thread一样不会被回收，就会造成内存泄漏，也就是key没了，value还在。解决办法是在使用完ThreadLocal后，执行remove操作。
- Entry初始化容量为16，扩容临界值为2/3，扩容时数据翻倍
- 用开放寻址法来寻找数组下标，当hashcode所选的index被占用后，选择index+1作为新位置，循环直到找到位置。

```java
public class MyClass {
  ThreadLocal<String> threadLocal2 = new ThreadLocal();【注2】
    public static void main(String[] args) {
        MyClass myClass = new MyClass();
        ThreadLocal<String> threadLocal = new ThreadLocal();
        Thread a =  myClass.new ThreadA(threadLocal);
        Thread b =  myClass.new ThreadB(threadLocal);
        a.start();
        b.start();
    }
    class ThreadA extends Thread{
        ThreadLocal local;
        public ThreadA(ThreadLocal<String> threadLocal) {
            local = threadLocal;
        }
        @Override
        public void run() {
            local.set("ThreadA"+Thread.currentThread());
            System.out.println(local.get());
        }
    }
    class ThreadB extends Thread{
        ThreadLocal local;
        public ThreadB(ThreadLocal<String> threadLocal) {
            local = threadLocal;
        }

        @Override
        public void run() {
            local.set("ThreadB"+Thread.currentThread());
            System.out.println(local.get());
            System.out.println(threadLocal2.get());【注2】

        }
    }
}

```

### ThreadLocal.set

```java
  public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
   Thread.java 
   ThreadLocal.ThreadLocalMap threadLocals = null;
```

通过set可以看到，首先会从当前线程拿threadLocals查看是否已经初始化，如果存在，则以当前调用set方法的ThreadLocal类为key，value值存入map中。

如果不存在就为这个Thread创建一个ThreadLocalMap，然后存入该value。

这个以前想错了，存储的过程是获得当前线程的ThreadLocalMap，然后根据当前调用的ThreadLocal对象作为key去存储对应的值。

### ThreadLocal.get

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
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
```

get方法通过拿到当前线程的ThreadLocalMap，然后以当前的threadlocal对象为key，去拿到对应的值。

### 注2

就像这里，用一个新的ThreadLocal类去调用get，此时的Thread里面的ThreadLocalMap是不为null的，但是该ThreadLocal类作为key在这个map中的值是不存在的，所以拿不到值。

类似于N个线程对应一个ThreadLocal对象，把这个对象当key然后存值，然后这些线程对于这个threadLocal可以拿到对应的值，换一个就拿不到。

Thread里面的sThreadlocals并不是ThreadLocal，而是存储值的ThreadLocalMap

