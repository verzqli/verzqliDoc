## Condition

Condition 是在JDK1.5之后出现的一种让线程间写作更加安全高效的类，他使用await()，signal() 和signalAll() 代替传统的Object的wait()、notify()和notifyA()，使用起来更加灵活和简便。

Condition 是一个接口，里面定义了一些操作线程的方法，具体的实现在AQS（AbstractQueuedSynchronizer）的内部类ConditionObject中

```java
public interface Condition {
    //当前线程在接到信号或被中断之前一直处于等待状态。
    void await() throws InterruptedException;
    //当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    //当前线程在接到信号之前一直处于等待状态。【注意：该方法对中断不敏感】
    void awaitUninterruptibly();
    //当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。返回值表示剩余时间，如果在nanosTimesout之前唤醒，那么返回值 = nanosTimeout - 消耗时间，如果返回值 <= 0 ,则可以认定它已经超时了。
    long awaitNanos(long nanosTimeout) throws InterruptedException;
	//当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。如果没有到指定时间就被通知，则返回true，否则表示到了指定时间，返回返回false。
    boolean awaitUntil(Date deadline) throws InterruptedException;
    //唤醒一个等待线程。该线程从等待方法返回前必须获得与Condition相关的锁。
    void signal();
    //唤醒所有等待线程。能够从等待方法返回的线程必须获得与Condition相关的锁。
    void signalAll();
    }
    public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
       	//每个condition对象里面都包含一个队列，等待队列是一个FIFO队列（先进先出）
        private transient Node firstWaiter;
        private transient Node lastWaiter;
     ...
 }
Condition condition =  new ReentrantLock().newCondition();
```

Condition类能实现synchronized和wait、notify搭配的功能，另外比后者更灵活，Condition可以实现多路通知功能，也就是在一个Lock对象里可以创建多个Condition（即对象监视器）实例，线程对象可以注册在指定的Condition中，从而**可以有选择的进行线程通知，在调度线程上更加灵活**。而synchronized就相当于整个Lock对象中只有一个单一的Condition对象，所有的线程都注册在这个对象上。线程开始notifyAll时，需要通知所有的WAITING线程，没有选择权，会有相当大的效率问题。

- Condition是个接口，基本的方法就是await()和signal()方法。

- Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition()，
- **调用Condition的await()和signal()方法，都必须在lock保护之内**，**就是说必须在lock.lock()和lock.unlock之间才可以使用。**
- Conditon.await()\==Object.wait()，Condition.signal()\==Object.notify()，Condition.signalAll()==Object.notifyAll()。

```java
  ReentrantLock lock = new ReentrantLock();
  //为线程A注册一个Condition
  Condition conditionA =  lock.newCondition();
  //为线程B注册一个Condition
  Condition conditionB =  lock.newCondition();
```

**使用Condition来实现等待/唤醒，并且能够唤醒制定线程。**

### 方法

#### 1.await

```java
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            //顾名思义，添加一个node到当前Condition对象的等待队列中【见1.1】
            Node node = addConditionWaiter();
            // 释放当前锁，保存锁状态（后面唤醒的时候需要用到），然后unpark释放同步队列的头结点【见1.3】
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            //第一次循环时，由于结点不在同步队列中，因此会进入到while内部代码中，使用LockSupport.park使线程阻塞
            //注意，这里从队列中查找是否有这个节点是从AQS的线程等待队列中查找，而不是condition的队列
            //判断当前结点是否在同步队列中，当使用sign或signAll唤醒、或者发生中断，结点都会进入同步队列中，才会跳过while循环，执行后续代码
            while (!isOnSyncQueue(node)) {
                //阻塞当前线程，这里注意，线程阻塞到这里就不执行下面的方法了，下面的方法是在这个线程被signal唤醒之后执行的
                LockSupport.park(this);
                 //线程被唤醒之后，我们要判断他是怎么被唤醒的，是中断还是signal唤醒的
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            //唤醒之后，重新尝试获取锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                //这里就是第二种中断情况，唤醒后申请锁的过程中被中断，这里返回后await的线程就重新获得了锁
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
             // 如果之前发生了中断，则根据中断模式重放中断
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
		/**
         * 检查是否断： 
         * 未中断返回0 
         * 被唤醒前中断返回 THROW_IE 
         * 被唤醒后中断返回 REINTERRUPT
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?(transferAfterCancelledWait(node) ? THROW_IE(-1):EINTERRUPT(1) : 0;
        }
        /**
         * 尝试将节点状态从CONDITION状态置为0
         * 如果设置成功，证明没有调用sign或signAll方法，线程唤醒是因为发生中断，因此将结点加入同步队列，并返回true。
         * 如果设置不成功，则是过sign或signAll唤醒，返回false
         */
   	 final boolean transferAfterCancelledWait(Node node) {
        if (node.compareAndSetWaitStatus(Node.CONDITION, 0)) {
            //将节点插入到队列,注意，这个队列也是AQS队列
            enq(node);
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
         //插入失败的情况很少见，如果node节点一直没插入到队列中，就一直在这自旋，并且让出cpu时间片，直到成功
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }
     /**
     * 如果interruptMode为THROW_IE，即在sign之前发送中断，则抛出InterruptedException。
     * 如果interruptMode为REINTERRUPT，即在sign之后发生中断，则调用interrupt方法，将中断补上。
     */
       private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

```

首先能执行到while循环的线程肯定不是阻塞的，进入循环后第一步就将该线程进行阻塞，关键点来了，线程阻塞到这里之后后面的代码是不执行的，直到该线程被唤醒或者中断之后才会继续执行后面的代码。

后面判断当前这次唤醒是代码被中断了还是被signal唤醒的，如果是被中断唤醒的将状态置为0，0是什么状态都没有，然后将节点重新插入到AQS队列中。

线程被唤醒之后尝试去获得锁



##### 1.1 addConditionWaiter

添加conditionWaiter节点到Condition对象的内部的waiter队列的队尾。

这里的nextWaiter有三种情况：

- nextWaiter==SHARED ，说明这个是共享锁，可以和其他线程共享锁
- nextWaiter==null，这个是独占锁
- nextWaiter==普通Condition Node 他就是普通的节点

```java
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            //如果有尾节点，但是节点状态不等于Node.CONDITION，说明被取消了，清除它，赋值一个新的尾节点。
            if (t != null && t.waitStatus != Node.CONDITION(-2)) {
                【见1.2】
                unlinkCancelledWaiters();
                t = lastWaiter;
            }

            Node node = new Node(Node.CONDITION);
			//尾节点为null说明头结点也没了，这里把头尾节点设置为一个。
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```

到这里，得到两个ReentrantLock与ConditionObject在实现上的重要区别：

- ReentrantLock创建的节点，初始状态为0；而ConditionObject创建的节点，初始状态为`CONDITION==-2`。
- ReentrantLock使用AQS内置的等待队列，由AQS维护；而每个ConditionObject都维护自己的等待队列。

##### 1.2 unlinkCancelledWaiters

`unlinkCancelledWaiters`方法主要将等待队列中等待状态不等于`CONDITION`的结点从队列中剔除。保证condition的waiter队列中的状态只有Node.CONDITION

```java
   private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
```

##### 1.3 fullyRelease(Node)

`fullyRelease`方法释放当前线程结点的资源，因为`ReentrantLock`只有一个资源，因此`ReentrantLock`创建的`Condition`的`await`方法相当于释放锁。并且从同步队列中取出head后面第一个节点通过LockSupport.unpark()释放它

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

##### 1.4 isOnSyncQueue(Node)

```java
  final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        return findNodeFromTail(node);
      
       private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }     
```

在创建节点node后未修改`node.waitStatus`，因此，仍满足`node.waitStatus == Node.CONDITION`，直接返回false，表示当前节点未阻塞。

当然，此时也满足`node.prev == null`，`node.next == null`。因此会进入while循环，然后当前线程陷入阻塞。

#### 2.signal

`signal`方法只唤醒等待队列的头结点，并将头结点出队列，加到同步队列队尾。
`signal`方法需要获取锁才能调用，否则抛出IllegalMonitorStateException异常。

```java
        public final void signal() {
            //判断占有锁的是否是当前线程
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
```

想要唤醒condition的线程必须获得当前的独占锁，否则抛出异常，然后取出condition等待队列的头结点进行唤醒。

```java
        /**
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignal(Node first) {
            do {	
                //如果头结点的下一个节点为null，说明队列中只有头结点一个点，将为节点置位null，然后把头结点出列
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&(first = firstWaiter) != null);
        }
	//将节点从条件队列(condition)转移到同步队列(AQS)。
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        //如果不能修改状态，说明状态被修改了，节点被cancelled了
        if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
            return false;

        /*
         * Splice onto(拼接) queue and try to set waitStatus of predecessor to
         * indicate(表明) that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently(暂时的) and harmlessly wrong).
         */
        Node p = enq(node);
        
        //这里的p是要唤醒节点的前置节点，如果有前置节点，且它的状态>0(CANCELLED)或者他被赋值Node.SIGNAL失败，
        //为什么赋值会失败，就是前一个节点是CANCELLED的时候，这两个是一个判断
        int ws = p.waitStatus;
        if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
	//enq的作用就是把节点插入到同步队列尾部
    private Node enq(Node node) {
        for (;;) {
            Node oldTail = tail;
            if (oldTail != null) {
                U.putObject(node, Node.PREV, oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return oldTail;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }
```

#### 3.signalAll

singAll的作用是唤醒该Condition上所有的Node，直接表现就是把等待队列所有的waiterNode全部插入到AQS同步队列的。signal每次只取头部一个，这里是全部取完。

```java
        public final void signalAll() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignalAll(first);
        }
		//把condition队列中所有的节点全部插入到AQS中
        private void doSignalAll(Node first) {
            lastWaiter = firstWaiter = null;
            do {
                Node next = first.nextWaiter;
                first.nextWaiter = null;
                transferForSignal(first);
                first = next;
            } while (first != null);
        }
```

### 总结

Condition内部维护了一个双向链表，头：firstWaiter，尾：lastWaiter,condition中队列中的作用是在多个线程中调用了await之后，signal根据先后去唤醒对应线程

- 当调用await时，通过``addConditionWaiter``新建一个node添加到Condition内部的等待队列。
- fullyRelease(node)->release(savedState)->tryRelease(arg)->unparkSuccessor(h)->LockSupport.unpark(s.thread)
- tryRelease清空当前线程状态和资源ExclusiveOwnerThread也设为null
- unparkSuccessor从同步队列中通过unpark唤醒结点（head后面一个），此时ExclusiveOwnerThread也设置成了这个新的线程
- while (!isOnSyncQueue(node)) 判断当前线程节点在不在同步队列（AQS）中，然后调用LockSupport.park阻塞当前线程，等待被唤醒
- 唤醒
- 
- 当一个condition调用signal时，先isHeldExclusively判断当前线程是否获得了锁，只有获得锁的线程才可以处理
- 再判断condition内部的队列有没有节点，每一次signal都会处理一个节点，当没有节点就无效，也就是多次调用没效果
- 取出等待队列的头结点（**和同步队列不同的是，这里的队列头结点可用**）然后添加到同步队列的尾部，因为unparkSuccessor是从同步队列的尾部开始取的
- signalAll则是把所有的等待节点全部添加到同步队列中。

```java

public class MyClass {
    ReentrantLock lock = new ReentrantLock();
    Condition conditionA = lock.newCondition();
    Condition conditionB = lock.newCondition();
    Condition conditionC = lock.newCondition();

    public static void main(String[] args) throws InterruptedException {
        MyClass myClass = new MyClass();
        Thread a = myClass.new ThreadA();
        Thread b = myClass.new ThreadB();
        Thread c = myClass.new ThreadC();
        Thread d = myClass.new ThreadD();
        a.start();
        b.start();
        c.start();
        d.start();

    }


    class ThreadD extends Thread {
        @Override
        public void run() {
            lock.lock();
            conditionB.signal();
            conditionA.signal();
            conditionA.signal();
            conditionA.signal();
            lock.unlock();

        }
    }

    class ThreadA extends Thread {
        @Override
        public void run() {
            lock.lock();
            try {
                System.out.println("A await");
                conditionA.await();
                System.out.println("A 苏醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.unlock();
        }
    }

    class ThreadB extends Thread {
        @Override
        public void run() {
            lock.lock();
            try {
                System.out.println("B await");
                conditionB.await();
                System.out.println("B 苏醒 A await");
                conditionA.await();
                System.out.println("B 苏醒 A await A 苏醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            lock.unlock();
        }
    }

    class ThreadC extends Thread {
        @Override
        public void run() {
            lock.lock();
            try {
                System.out.println("C await");
                conditionC.await();
                System.out.println("C 苏醒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            lock.unlock();
        }
    }
}

//这里的输出结果是：
A await
B await
C await
B 苏醒 A await
A 苏醒
```

为什么这里的 conditionA.signal();没有苏醒B中的后续，因为ThreadB中conditionA.await();是在ThreadD中unlock之后唤醒了B之后，B又阻塞在conditionA.await()这里，想唤醒这里，需要再去开一个线程调用conditionA.signal。

此时的conditionA内部队列数量变化为1-0-1；

如果把ThreadC中的conditionC改为conditionA，那么输出结果为：

```java
A await
B await
C await
B 苏醒 A await
A 苏醒
C 苏醒
```

可以看到，ThreadC和ThreadA中conditionA.await();是同一批，所以此时队列中数量为2，但是这样还是唤醒不了B中的conditionA.await();

通俗一点说就是这两次await不是同一批的，B中的await是在唤醒AC后重新再阻塞的。

如果把三个conditionA.signal()换成conditionA.signalAll()，结果和上面一样，也是唤醒不了B中的conditionA.await()。

### 双线程打印：

```java
  ReentrantLock lock = new ReentrantLock();
    Condition conditionA = lock.newCondition();
    Condition conditionB = lock.newCondition();


    public static void main(String[] args) throws InterruptedException {
        MyClass myClass = new MyClass();
        Thread a = myClass.new ThreadA();
        Thread b = myClass.new ThreadB();

        b.start();
        Thread.sleep(11);
        a.start();
    }

    class ThreadA extends Thread {
        @Override
        public void run() {
            lock.lock();
            for (int i = 1; i < 100; i+=2) {
                System.out.println("线程A   "+i);
                conditionB.signal();
                try {
                    conditionA.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
            lock.unlock();
        }
    }

    class ThreadB extends Thread {
        @Override
        public void run() {
            lock.lock();
            for (int i = 0; i < 100; i+=2) {
                System.out.println("线程B   "+i);
                try {
                    conditionB.await();
                    conditionA.signal();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

            lock.unlock();
        }
    }

```



##### InterruptedException 和 Thread#interrupt() 的区别

- 如果抛出InterruptedException，则外界必须处理（捕获或继续外抛）。
- 如果调用Thread#interrupt()，则仅设置中断标志。JDK提供的某些阻塞方法会处理该标志，详见Javadoc；但用户自己实现的方法，是否会处理该标志，处理是否及时，都无法做出保证。







