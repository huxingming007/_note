### 除了虚拟机，程序员自己如何优化锁

**减小锁的持有时间**

synchronized代码块的范围，在允许的情况下，尽可能小；

**减小锁的粒度**

从hashtable 到JDK1.7的concurrenthashmap 再到JDK1.8的concurrenthashmap

**读写锁替代独占锁**

ReentrantReadWriteLock到JDK1.8的StampedLock（因为ReentrantReadWriteLock读写互斥，写写互斥，大量的读线程会导致写线程的饥饿，所以有了这个改进：提供了一种乐观的读策略，这种乐观策略的锁非常类似于无锁的操作，使得乐观读完全不会阻塞写线程，使用乐观读，如果在这期间，有线程申请到了写锁，那乐观读失败，要么重试，要么升级成悲观读）；

**锁分离**

```
LinkedBlockingQueue
 private final ReentrantLock takeLock = new ReentrantLock();
 private final ReentrantLock putLock = new ReentrantLock();
```