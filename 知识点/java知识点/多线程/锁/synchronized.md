

可重入性：当一个线程获得了某个对象的锁，那么该线程在持有该对象锁的情况下，能够再次获得该对象（或者该对象父类）锁。``synchronized``关键字和``ReentrantLock``形成的锁都是可重入锁。

如下例子

```java
public synchronized void test(){ 
    //business 
    test1(); 
}
 
public synchronized void test1(){ 
    //business 
}
```



在上段代码中，执行test方法需要获取当前对象的监视器锁，而test方法内部又调用了同步方法test1。

如果对象监视器锁是具有可重入性的，那么该线程在调用test1时，可以再次获得对象的监视器锁，从而进入test1方法。

如果对象监视器锁不具有可重入性，那么线程在test方法中调用同步方法test1之前，需要等待当前对象监视器锁的释放，然而，实际上该对象的监视器锁已经被当前线程所持有，不可能再次获得，死锁产生。

