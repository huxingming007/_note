### 什么是线程池

**现状：**

- 如果每个请求到达都创建一个新线程，创建和销毁线程花费的时间和消耗的系统资源都相当大，甚至可能要比在处理实际的用户请求的时间和资源要多的多；
- 如果在一个JVM中创建太多的线程，可能会使系统由于过度消耗内存或者“切换过度”而导致系统资源不足；

**解决：**

- 线程池的核心逻辑是提前创建好若干个线程放在一个容器中。如果有任务需要处理，则将任务直接分配给线程池中的线程来执行就行，任务处理完以后这个线程不会被销毁，而是等待后续分配任务。同时通过线程池来重复管理线程还可以避免创建大量线程增加开销。

**优势：**

1. 降低创建线程和销毁线程的性能开销
2. 提高响应速度，当有新任务需要执行是不需要等待线程创建就可以立马执行 
3. 合理的设置线程池大小可以避免因为线程数超过硬件资源瓶颈带来的问题

### API

```
Executors类（不建议使用，阿里规范）
```

```java
// 固定数量的线程池，若空闲则执行，若没有空闲则加入LinkedBlockingQueue，队列容量非常大，可以一直添加
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());// 默认容量是Integer.MAX_VALUE，相当于没有上限
}
```

```java
// 创建一个线程的线程池，若空闲则执行，若没有空闲则加入LinkedBlockingQueue
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

```java
// 可以按照实际情况调整的线程池，不限制线程最大数量，每一个空闲的线程会在60秒后自动回收
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

```java
// 创建指定线程个数的线程池，延迟和周期性执行任务的功能，类似定时器
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

```java
ThreadPoolExecutor类（建议使用此类构建线程池）
public ThreadPoolExecutor(int corePoolSize,// 核心线程数量
                              int maximumPoolSize,// 最大线程数
                              long keepAliveTime,// 超时时间，超出核心线程数量以外的线程空余存活时间
                              TimeUnit unit,// 存活时间单位
                              BlockingQueue<Runnable> workQueue,// 保存执行任务的队列
                              ThreadFactory threadFactory,// 创建新线程使用的工厂
                              RejectedExecutionHandler handler) // 拒绝策略
```

### 实现原理分析

ThreadPoolExecutor 是线程池的核心，提供了线程池的实现。 ScheduledThreadPoolExecutor 继承了 ThreadPoolExecutor，并另外提供一些调度方法以支 持定时和周期任务。Executers 是工具类，主要用来创建线程池对象 我们把一个任务提交给线程池去处理的时候，线程池的处理过程是什么样的呢?

![image-20190814170852863](http://ww2.sinaimg.cn/large/006tNc79gy1g61n3hqelfj30wv0u0dsg.jpg)

### 源码分析

#### ctl的作用

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

它是一个原子类，主要作用是用来保存线程数量和线程池的状态。一个int数值是32个bit位，这里采用高3位来保存运行状态，低29位来保存线程数量。

#### addWorker

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);// 线程池的状态

        // Check if queue empty only if necessary.
        // 1、线程池已经shutdown，拒绝；2、已经是关闭状态，传过来的任务为null，并且队列不为空，是允许添加新线程
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;// 不可以添加

        for (;;) {
            int wc = workerCountOf(c);// 获得worker工作线程数
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;// 大于最大容量，或者大于核心线程数，最大线程数，不可以添加
            if (compareAndIncrementWorkerCount(c))// cas来增加工作线程数
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
// 上面的代码是对worker数量做原子加一，下面的逻辑才是正式构建一个worker
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);// 构建
        final Thread t = w.thread;// 从worker对象取出线程
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);// 添加到worker集合中
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();// 启动线程
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);// 如果添加失败，递减实际工作线程数-1，下面会讲到
    }
    return workerStarted;
}
```

#### worker类

```java
// 重点：继承了AQS实现独占锁的功能，疑问点？为什么不使用ReentrantLock。下面会讲到
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    // 构建worker函数，创建一个线程，传入的参数是this，worker本身继承了runnable接口，也就是表明worker就是一个线程
    // thread.start的时候，执行的是runWorker(this);
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

worker继承了AQS，实现了独占锁，可以看到 tryAcquire 方法，它是不允许重入的，而 ReentrantLock 是允许重入的: 

```java
// worker的实现是独占锁！
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

lock 方法一旦获取了独占锁，表示当前线程正在执行任务中;那么它会有以下几个作用：

- 如果正在执行任务，则不应该中断线程；
- 如果该线程现在不是独占锁的状态，也就是空闲的状态，说明它没有在处理任务，这时可
  以对该线程进行中断;
- 线程池在执行 shutdown 方法或 tryTerminate 方法时会调用 interruptIdleWorkers 方法来 中断空闲的线程，interruptIdleWorkers 方法会使用 tryLock 方法来判断线程池中的线程 是否是空闲状态
- 之所以设置为不可重入，是因为我们不希望任务在调用像 setCorePoolSize 这样的线程池 控制方法时重新获取锁，这样会中断正在运行的线程

#### addWorkerFailed

addWorker 方法中，如果添加 Worker 并且启动线程失败，则会做失败后的处理。 这个方法主要做两件事
1. 如果 worker 已经构造好了，则从 workers 集合中移除这个 worker
2. 原子递减核心线程数(因为在 addWorker 方法中先做了原子增加)
3. 尝试结束线程池

```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

#### runWorker

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;// 先执行这个
    w.firstTask = null;
    w.unlock(); // new worker的时候默认state=-1，此处调用unlock使得state置为0，允许中断！
    boolean completedAbruptly = true;
    try {
       // firstTask执行完了，然后从getTask中获得task继续执行
        while (task != null || (task = getTask()) != null) {
            w.lock();// 上锁，不是为了防止并发执行任务，为了在shutdown()时候不终止正在运行的worker，下面会更深入讲解
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);//可以自己重写
                Throwable thrown = null;
                try {
                    task.run();// 执行任务中的方法，记住调用的是run(),而不是start()!
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);//可以自己重写
                }
            } finally {
               // 置为空，下次循环的时候通过getTask()取队列中的任务
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 把worker从数组workers里删除
        // 根据布尔值 allowCoreThreadTimeOut 来决定是否补充新的 Worker 进数组 workers
        processWorkerExit(w, completedAbruptly);
    }
}
```

#### getTask

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.状态的判断
        // 1、shutdown状态，且workQueue为空，退出
        // 2、stop状态（执行了方法shutDownNow()），退出
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;// 则当前worker线程会退出
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // allowCoreThreadTimeOut默认为false。wc > corePoolSize表示当前线程池的线程数量大于核心线程数；
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

      // 1. 线程数量超过maximumPoolSize可能是线程池在运行时被调用了setMaximumPoolSize() 被改变了大小，否则已经 addWorker()成功不会超过 maximumPoolSize
      // 2. timed && timedOut 如果为 true，表示当前操作需要进行超时控制，并且上次从阻塞队列中 获取任务发生了超时.其实就是体现了空闲线程的存活时间
      if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // poll方法进行超时控制，如果在keepAliveTime时间内没有获取到任务，则返回null。否则通过take方法阻塞式获取队列中的任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;// 如果r==null，表明已经超时，下次自旋的时候进行回收
        } catch (InterruptedException retry) {
            timedOut = false;// 被中断，shutDown() shutDownNow()方法会使中断
        }
    }
}
```

这里重要的地方是第二个if判断，目的是控制线程池的有效线程数量。由上文中的分析可以 知道，在执行 execute 方法时，如果当前线程池的线程数量超过了 corePoolSize 且小于 maximumPoolSize，并且 workQueue 已满时，则可以增加工作线程，但这时如果超时没有 获取到任务，也就是 timedOut 为 true 的情况，说明 workQueue 已经为空了，也就说明了 当前线程池中不需要那么多线程来执行任务了，可以把多于 corePoolSize 数量的线程销毁 掉，保持线程数量在 corePoolSize 即可。什么时候会销毁?当然是 runWorker 方法执行完之后，也就是 Worker 中的 run 方法执行 完，由 JVM 自动回收。getTask 方法返回 null 时，在 runWorker 方法中会跳出 while 循环，然后会执行 processWorkerExit 方法。

#### processWorkerExit

runWorker 的 while 循环执行完毕以后，在 finally 中会调用 processWorkerExit，来销毁工作线程。

#### 拒绝策略

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);// 非运行状态，拒绝
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);// 核心池已满，队列已满，最大线程数maximumPoolSize已满，拒绝
}
```

> 1、AbortPolicy:直接抛出异常，默认策略;
>
>  2、CallerRunsPolicy:用调用者所在的线程来执行任务; 
>
> 3、DiscardOldestPolicy:丢弃阻塞队列中靠最前的任务，并执行当前任务; 
>
> 4、DiscardPolicy:直接丢弃任务;
>
> 当然也可以根据应用场景实现 RejectedExecutionHandler 接口，自定义饱和策略，如记录 日志或持久化存储不能处理的任务

### 线程池的注意事项

#### 阿里规范

线程池的构建不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式。分析完原理以后，大家自己一定要有一个答案。我来简单分析下，用 Executors 使得用户不 需要关心线程池的参数配置，意味着大家对于线程池的运行规则也会慢慢的忽略。这会导致 一个问题，比如我们用 newFixdThreadPool 或者 singleThreadPool.允许的队列长度为 Integer.MAX_VALUE，如果使用不当会导致大量请求堆积到队列中导致 OOM 的风险
而 newCachedThreadPool，允许创建线程数量为 Integer.MAX_VALUE，也可能会导致大量 线程的创建出现 CPU 使用过高或者 OOM 的问题，而如果我们通过 ThreadPoolExecutor 来构造线程池的话，我们势必要了解线程池构造中每个 参数的具体含义，使得开发者在配置参数的时候能够更加谨慎。不至于像有些同学去面试的 时候被问到:构造一个线程池需要哪些参数，都回答不上来。

#### 如何配置线程池的大小

在遇到这类问题时，先冷静下来分析
1. 需要分析线程池执行的任务的特性: CPU 密集型还是 IO 密集型
2. 每个任务执行的平均时长大概是多少，这个任务的执行时长可能还跟任务处理逻辑是否涉 及到网络传输以及底层系统资源依赖有关系

如果是 CPU 密集型，主要是执行计算任务，响应时间很快，cpu 一直在运行，这种任务 cpu 的利用率很高，那么线程数的配置应该根据 CPU 核心数来决定，CPU 核心数=最大同时执行 线程数，加入 CPU 核心数为 4，那么服务器最多能同时执行 4 个线程。过多的线程会导致上 下文切换反而使得效率降低。那线程池的最大线程数可以配置为 cpu 核心数+1
如果是 IO 密集型，主要是进行 IO 操作，执行 IO 操作的时间较长，这是 cpu 出于空闲状态， 导致 cpu 的利用率不高，这种情况下可以增加线程池的大小。这种情况下可以结合线程的等 待时长来做判断，等待时间越高，那么线程数也相对越多。一般可以配置 cpu 核心数的 2 倍。 一个公式:线程池设定最佳线程数目 = (线程池设定的线程等待时间+线程 CPU 时间)/ 线程 CPU 时间 )* CPU 数目，这个公式的线程 cpu 时间是预估的程序单个线程在 cpu 上运行的时间(通常使用 loadrunner 测试大量运行次数求出平均值)

#### 线程池的关闭

[优雅关闭线程池](https://www.cnblogs.com/qingquanzi/p/9018627.html)

- shutdownNow方法的解释是：线程池拒接收新提交的任务，同时立马关闭线程池，线程池里的任务不再执行。可能会引发报错，一定要对任务进行异常捕获。task.run();run方法中有发生了阻塞，这个线程一旦中断，就会抛出异常。task.run还会执行。
- shutdown方法的解释是：线程池拒接收新提交的任务，同时等待线程池里的任务执行完毕后关闭线程池。一定要确保任务里不会有永久阻塞的逻辑，否则线程池就关闭不了。
- 最后，一定要记得，shutdownNow和shuwdown调用完，线程池并不是立马就关闭了，要想等待线程池关闭，还需调用awaitTermination方法来阻塞等待。

#### 线程池容量的动态调整

ThreadPoolExecutor 提供了动态调整线程池容量大小的方法:setCorePoolSize()和 setMaximumPoolSize()，setCorePoolSize:设置核心池大小 setMaximumPoolSize:设置线 程池最大能创建的线程数目大小
任务缓存队列及排队策略
在前面我们多次提到了任务缓存队列，即 workQueue，它用来存放等待执行的任务。 workQueue 的类型为 BlockingQueue，通常可以取下面三种类型:

1. ArrayBlockingQueue:基于数组的先进先出队列，此队列创建时必须指定大小;
2. LinkedBlockingQueue:基于链表的先进先出队列，如果创建时没有指定此队列大小，则默
认为 Integer.MAX_VALUE;
3. SynchronousQueue:这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个
线程来执行新来的任务。

#### 线程池的监控

如果在项目中大规模的使用了线程池，那么必须要有一套监控体系，来指导当前线程池的状 态，当出现问题的时候可以快速定位到问题。而线程池提供了相应的扩展方法，我们通过重 写线程池的 beforeExecute、afterExecute 和 shutdown 等方式就可以实现对线程的监控

~~~java
public class Demo1 extends ThreadPoolExecutor {
// 保存任务开始执行的时间,当任务结束时,用任务结束时间减去开始时间计算任务执行时间 
private ConcurrentHashMap<String,Date> startTimes;
public Demo1(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
this.startTimes=new ConcurrentHashMap<>(); 
}

@Override
public void shutdown() { 
  System.out.println("已经执行的任务数:"+this.getCompletedTaskCount()+"," + "当前活动线程数:"+this.getActiveCount()+",当前排队线程数:"+this.getQueue().size()); System.out.println();
  super.shutdown(); 
}
//任务开始之前记录任务开始时间
@Override
protected void beforeExecute(Thread t, Runnable r) {
startTimes.put(String.valueOf(r.hashCode()),new Date());
super.beforeExecute(t, r); 
}
@Override
protected void afterExecute(Runnable r, Throwable t) {
Date startDate = startTimes.remove(String.valueOf(r.hashCode())); 
Date finishDate = new Date();
long diff = finishDate.getTime() - startDate.getTime(); // 统计任务耗时、初始线程数、核心线程数、正在执行的任务数量、
// 已完成任务数量、任务总数、队列里缓存的任务数量、
// 池中存在的最大线程数、最大允许的线程数、线程空闲时间、线程池是否关闭、线程池 是否终止
System.out.print("任务耗时:"+diff+"\n"); System.out.print("初始线程数:"+this.getPoolSize()+"\n"); System.out.print("核心线程数:"+this.getCorePoolSize()+"\n"); System.out.print("正在执行的任务数量:"+this.getActiveCount()+"\n"); System.out.print("已经执行的任务数:"+this.getCompletedTaskCount()+"\n"); System.out.print("任务总数:"+this.getTaskCount()+"\n"); System.out.print("最大允许的线程数:"+this.getMaximumPoolSize()+"\n"); System.out.print("线程空闲时间:"+this.getKeepAliveTime(TimeUnit.MILLISECONDS)+"\n"); System.out.println();
super.afterExecute(r, t); 
}
  
public static ExecutorService newCachedThreadPool() {
     return new Demo1(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue ());
  }
}

测试脚本
public class Test implements Runnable{

private static ExecutorService es =Demo1.newCachedThreadPool(); 

@Override
public void run() {
try {

Thread.sleep(1000);
} catch (InterruptedException e) {
          e.printStackTrace();
       }
}
public static void main(String[] args) throws Exception {
for (int i = 0; i < 100; i++) { es.execute(new Test());
}
es.shutdown(); }
}
~~~

### Callable/Future 使用及原理分析

#### execute 和 submit 区别

1. execute 只可以接收一个 Runnable 的参数 
2.  execute 如果出现异常会抛出
3. execute 没有返回值
4. submit 可以接收 Runable 和 Callable 这两种类型的参数，
5. 对于 submit 方法，如果传入一个 Callable，可以得到一个 Future 的返回值
6. submit 方法调用不会抛异常，除非调用 Future.get

~~~java
public class CallableDemo implements Callable<String> {
@Override
public String call() throws Exception {
//Thread.sleep(3000);//阻塞案例演示
return "hello world"; 
}
public static void main(String[] args) throws ExecutionException, InterruptedException {
CallableDemo callableDemo=new CallableDemo(); 
FutureTask futureTask=new FutureTask(callableDemo); 
new Thread(futureTask).start(); 
System.out.println(futureTask.get());
} 
}
~~~

想一想我们为什么需要使用回调呢?那是因为结果值是由另一线程计算的，当前线程是不知 道结果值什么时候计算完成，所以它传递一个回调接口给计算线程，当计算完成时，调用这 个回调接口，回传结果值。
这个在很多地方有用到，比如 Dubbo 的异步调用，比如消息中间件的异步通信等等...
利用 FutureTask、Callable、Thread 对耗时任务(如查询数据库)做预处理，在需要计算结 果之前就启动计算。
所以我们来看一下 Future/Callable 是如何实现的。

#### Callable/Future 原理分析

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

![image-20190815184227931](http://ww1.sinaimg.cn/large/006tNc79gy1g61n44dj2aj30qw0lgn05.jpg)

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

分析到这里我们其实有一些初步的头绪了，FutureTask 是 Runnable 和 Future 的结合，如果 我们把 Runnable 比作是生产者，Future 比作是消费者，那么 FutureTask 是被这两者共享的， 生产者运行 run 方法计算结果，消费者通过 get 方法获取结果。 作为生产者消费者模式，有一个很重要的机制，就是如果生产者数据还没准备的时候，消费 者会被阻塞。当生产者数据准备好了以后会唤醒消费者继续执行。 这个有点像我们上次可分析的阻塞队列，那么在 FutureTask 里面是基于什么方式实现的呢?

#### State的含义

~~~java
private static final int NEW = 0; // NEW 新建状态，表示这个 FutureTask 还没有开始运行
// COMPLETING 完成状态， 表示 FutureTask 任务已经计算完毕了
// 但是还有一些后续操作，例如唤醒等待线程操作，还没有完成。
private static final int COMPLETING = 1; // FutureTask任务完结，正常完成，没有发生异常
private static final int NORMAL = 2; // FutureTask任务完结，因为发生异常。
private static final int EXCEPTIONAL = 3; // FutureTask任务完结，因为取消任务
private static final int CANCELLED = 4;
// FutureTask任务完结，也是取消任务，不过发起了中断运行任务线程的中断请求
private static final int INTERRUPTING = 5;
// FutureTask任务完结，也是取消任务，已经完成了中断运行任务线程的中断请求
private static final int INTERRUPTED = 6;
~~~



#### run方法

```java
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;// 如果状态不是new，或者设置runner值失败，保证只有一个线程可以运行try代码块中的代码
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();// 调用callable的call方法，并获得返回结果
                ran = true;
            } catch (Throwable ex) {// catch住异常，没有往外抛，这是个坑点！
                result = null;
                ran = false;
                setException(ex);// 设置异常结果，如果异常了，想知道什么异常，只能通过get的方式进行获取！
            }
            if (ran)
                set(result);// 设置
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

其实 run 方法作用非常简单，就是调用 callable 的 call 方法返回结果值 result，根据是否发生 异常，调用 set(result)或 setException(ex)方法表示 FutureTask 任务完结。
不过因为 FutureTask 任务都是在多线程环境中使用，所以要注意并发冲突问题。注意在 run 方法中，我们没有使用 synchronized 代码块或者 Lock 来解决并发问题，而是使用了 CAS 这 个乐观锁来实现并发安全，保证只有一个线程能运行 FutureTask 任务。

#### get方法

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

get 方法就是阻塞获取线程执行结果，这里主要做了两个事情
1. 判断当前的状态，如果状态小于等于 COMPLETING，表示 FutureTask 任务还没有完结，
所以调用 awaitDone 方法，让当前线程等待。
2. report 返回结果值或者抛出异常

#### awaitDone

如果当前的结果还没有被执行完，把当前线程线程和插入到等待队列

~~~java
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {// 中断的处理方式
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {// 表示任务已结束
                if (q != null)
                    q.thread = null;// 线程移除
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet 表示任务还有一些后续操作没有完成，那么当前线程让出执行权
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)// 把新节点CAS添加到链表中，如果添加失败，那么queued为false，下次循环时，会继续添加，知道成功
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {// 为true表示需要设置超时
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);// 让线程等待nanos时间
            }
            else
                LockSupport.park(this);// 让线程等待
        }
    }
~~~

#### report

report 方法就是根据传入的状态值 s，来决定是抛出异常，还是返回结果值。这个两种情况都 表示 FutureTask 完结了

~~~java
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)// 正常完结状态
            return (V)x;
        if (s >= CANCELLED)// 手动取消任务
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);// 运行过程中，发送了异常，这里就抛出这个异常
    }
~~~



#### 线程池对于Future/Callable的执行

~~~java
public class CallableDemo implements Callable<String> {
@Override
public String call() throws Exception {
//Thread.sleep(3000);//阻塞案例演示
return "hello world";
}
public static void main(String[] args) throws ExecutionException, InterruptedException {
ExecutorService es=Executors.newFixedThreadPool(1);
CallableDemo callableDemo=new CallableDemo(); 
  Future future=es.submit(callableDemo); 
  System.out.println(future.get());
} }

~~~

```java
AbstractExecutorService#submit
```

~~~java

public <T> Future<T> submit(Callable<T> task) {
if (task == null) 
  throw new NullPointerException(); 
RunnableFuture<T> ftask = newTaskFor(task); // 包装成FutureTask，FutureTask是个runnable
execute(ftask);// ThreadpoolExecutor.execute,前面分析过了,最后执行FutureTask的run方法
return ftask;
}
~~~

