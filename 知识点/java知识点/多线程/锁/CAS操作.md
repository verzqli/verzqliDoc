## CAS 操作

JAVA使用 **锁**和**循环CAS**
 先说说悲观锁与乐观锁：

- 悲观锁: 假定会发生并发冲突，即共享资源会被某个线程更改。所以当某个线程获取共享资源时，会阻止别的线程获取共享资源。也称独占锁或者互斥锁，例如java中的synchronized同步锁。
- 乐观锁: 假设不会发生并发冲突,只有在最后更新共享资源的时候会判断一下在此期间有没有别的线程修改了这个共享资源。如果发生冲突就重试，直到没有冲突，更新成功。CAS就是一种乐观锁实现方式。

悲观锁会阻塞其他线程。乐观锁不会阻塞其他线程，如果发生冲突，采用死循环的方式一直重试，直到更新成功。

**CAS，Compare and Swap**
 CAS的思想很简单：三个参数，一个当前内存值V、旧的预期值A、即将更新的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，否则什么都不做，并返回false。
 JVM中的CAS操作正是利用了提到的处理器提供的CMPXCHG指令实现的；循环CAS实现的基本思路就是循环进行CAS操作直到成功为止；
 先来看看CAS在atomic类中的应用



```java
    public final native boolean compareAndSwapObject
       (Object obj, long valueOffset, Object expect, Object update);

    public final native boolean compareAndSwapInt
       (Object obj, long valueOffset, int expect, int update);

    public final native boolean compareAndSwapLong
      (Object obj, long valueOffset, long expect, long update);
```

Atomic类，它们实现了对**确认，更改再赋值**操作的原子性；我们说线程安全，当A线程在运行时被挂起，B线程运行后再回到A，此时B的修改对A不可见，那么便会出现问题。原先我们可以用Synchronized将代码保护起来，利用排他性，但是过于笨重，于是有了CAS，直接将原先确认操作得到的值再对它进行相关计算前，先到主内存中查看是否有被改变过，有则重新获取该值。这样同样解决了上面说的问题。

下面以AtomicInteger为例：



```cpp
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

- Unsafe是CAS的核心类，Java没有方法能访问底层系统，因此需要本地方法来做，Unsafe就是一个后门，被提供来直接操作内存中的数据。
- valueOffset：变量在内存中的偏移地址，Unsafe根据偏移地址找到获取数据。
- value被volatile修饰，保证了内存可见性。

**getAndAdd方法**



```java
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
//---------------unsafe
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

getIntVolatile通过偏移量获取到内存中变量值，compareAndSwapInt会比较获取的值与此时内存中的变量值是否相等，不相等则继续循环重复。整个过程利用CAS保证了对于value的修改的并发安全。

上面我们说CAS利用了处理器的CMPXCHG指令，该指令操作的内存区域就会加锁，导致其他处理器不能同时访问它，保证原子性。

**我们因为笨重放弃了synchronized，CAS是具有原子性的，于是利用循环 + CAS来确保线程安全**

### ABA问题

ABA问题有两种情况：

#### 第一类问题

我们考虑下面一种ABA的情况：

1. 在多线程的环境中，线程a从共享的地址X中读取到了对象A。
2. 在线程a准备对地址X进行更新之前，线程b将地址X中的值修改为了B。
3. 接着线程b将地址X中的值又修改回了A。
4. 最新线程a对地址X执行CAS，发现X中存储的还是对象A，对象匹配，CAS成功。

上面的例子中CAS成功了，但是实际上这个CAS并不是原子操作，如果我们想要依赖CAS来实现原子操作的话可能就会出现隐藏的bug。

第一类问题的关键就在2和3两步。这两步我们可以看到线程b直接替换了内存地址X中的内容。

在拥有自动GC环境的编程语言，比如说java中，2，3的情况是不可能出现的，因为在java中，只要两个对象的地址一致，就表示这两个对象是相等的。

2，3两步可能出现的情况就在像C++这种，不存在自动GC环境的编程语言中。因为可以自己控制对象的生命周期，如果我们从一个list中删除掉了一个对象，然后又重新分配了一个对象，并将其add back到list中去，那么根据 MRU memory allocation算法，这个新的对象很有可能和之前删除对象的内存地址是一样的。这样就会导致ABA的问题。

##### 解决方法

根据官方的说法，第一类问题大概有四种解法：

1. 使用中间节点 – 使用一些不代表任何数据的中间节点来表示某些节点是标记被删除的。
2. 使用自动GC。
3. 使用hazard pointers – hazard pointers 保存了当前线程正在访问的节点的地址，在这些hazard pointers中的节点不能够被修改和删除。
4. 使用read-copy update (RCU) – 在每次更新的之前，都做一份拷贝，每次更新的是拷贝出来的新结构。

#### 第二类问题

如果我们在拥有自动GC的编程语言中，那么是否仍然存在CAS问题呢？

考虑下面的情况，有一个链表里面的数据是A->B->C,我们希望执行一个CAS操作，将A替换成D，生成链表D->B->C。考虑下面的步骤：

1. 线程a读取链表头部节点A。
2. 线程b将链表中的B节点删掉，链表变成了A->C
3. 线程a执行CAS操作，将A替换从D。

最后我们的到的链表是D->C，而不是D->B->C。

问题出在哪呢？CAS比较的节点A和最新的头部节点是不是同一个节点，它并没有关心节点A在步骤1和3之间是否内容发生变化。

我们举个例子：

```java
        TestA a1 = new TestA();
        TestA a2 = new TestA();
        TestA a3 = new TestA();
        AtomicReference<TestA> aAtomicReference = new AtomicReference<>(a1);
        System.out.println(""+aAtomicReference.compareAndSet(a1,a2)+"  "+a1.aa+"  "+aAtomicReference.get().aa);

        System.out.println(""+aAtomicReference.compareAndSet(a2,a1)+"  "+a1.aa+"  "+aAtomicReference.get().aa);
        a1.aa = "a3333";
        System.out.println(""+aAtomicReference.compareAndSet(a1,a3)+"  "+a1.aa+"  "+aAtomicReference.get().aa);
        a1 = new TestA();
        System.out.println(""+aAtomicReference.compareAndSet(a1,a3)+"  "+a1.aa+"  "+aAtomicReference.get().aa);
        

        
```

上面的例子中，我们使用了AtomicReference的CAS方法来判断对象是否发生变化。在CAS b和a之后，我们将a的name进行了修改，我们看下最后的输出结果：

```text
true  aold  aold
true  aold  aold
true  a3333  aold
false  aold  aold
```

三个CAS的结果都是true。说明CAS确实比较的两者是否为统一对象，对其中内容的变化并不关心。最后一个修改了地址之后则失败

这里还需要注意的一点是aAtomicReference修改了对象主存的值，但是线程中的数据并没有更新，还是老数据

第二类问题可能会导致某些集合类的操作并不是原子性的，因为你并不能保证在CAS的过程中，有没有其他的节点发送变化。

##### 解决方法

第二类问题其实算是整体集合对象的CAS问题了。一个简单的解决办法就是每次做CAS更新的时候再添加一个版本号。如果版本号不是预期的版本，就说明有其他的线程更新了集合中的某些节点，这次CAS是失败的。

我们举个AtomicStampedReference的例子：

```text
        Object a= new Object();
        Object b= new Object();
        Object c= new Object();
        AtomicStampedReference<Object> atomicStampedReference= new AtomicStampedReference(a,0);
        log.info("{}",atomicStampedReference.compareAndSet(a,b,0,1));
        log.info("{}",atomicStampedReference.compareAndSet(b,a,1,2));
        log.info("{}",atomicStampedReference.compareAndSet(a,c,0,1));
true
true
false
```

AtomicStampedReference的compareAndSet方法，多出了两个参数，分别是expectedStamp和newStamp，两个参数都是int型的，需要我们手动传入。