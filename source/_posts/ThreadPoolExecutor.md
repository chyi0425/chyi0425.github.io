---
title: ThreadPoolExecutor
date: 2018-07-08 10:56:58
tags: [Java,concurrency] #文章标签，多于一项时用这种格式
toc: true
---

** ThreadPoolExecutor

下图是Java线程池的机构和相关类的继承结构
![示意图](/img/threadPoolExtends.jpg)

我们经常会使用 Executors 这个工具类来快速构造一个线程池，对于初学者而言，这种工具类是很有用的，开发者不需要关注太多的细节，只要知道自己需要一个线程池，仅仅提供必需的参数就可以了，其他参数都采用作者提供的默认值。

```Java
	public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
它们最终会导向这个构造方法
```Java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
### 线程池的重要属性
基本上，上面的构造方法中列出了我们最需要关心的几个属性了，下面逐个介绍下构造方法中出现的这几个属性：
* corePoolSize

> 核心线程数

* maximumPoolSize

> 最大线程数

* workQueue

> 任务队列，BlockingQueue 接口的某个实现（常使用 ArrayBlockingQueue 和 LinkedBlockingQueue）。

* keepAliveTime

> 空闲线程的保活时间，也可以通过调用 allowCoreThreadTimeOut(true)使核心线程数内的线程也可以被回收。

* handler

> 当线程池已经满了，但是又有新的任务提交的时候，该采取什么策略由这个来指定。有几种方式可供选择，像抛出异常、直接拒绝然后返回等，也可以自己实现相应的接口实现自己的逻辑

除了上面几个属性外，我们再看看其他重要的属性。

Doug Lea 采用一个 32 位的整数来存放线程池的状态和当前池中的线程数，其中高 3 位用于存放线程池状态，低 29 位表示线程数（即使只有 29 位，也已经不小了，大概 5 亿多，现在还没有哪个机器能起这么多线程的吧）。我们知道，java 语言在整数编码上是统一的，都是采用补码的形式，下面是简单的移位操作和布尔操作，都是挺简单的。

```Java
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

	// 这里 COUNT_BITS 设置为 29(32-3)，意味着前三位用于存放线程状态，后29位用于存放线程数
	// 很多初学者很喜欢在自己的代码中写很多 29 这种数字，或者某个特殊的字符串，然后分布在各个地方，这是非常糟糕的
    private static final int COUNT_BITS = Integer.SIZE - 3;

    // 000 11111111111111111111111111111
	// 这里得到的是 29 个 1，也就是说线程池的最大线程数是 2^29-1=536870911
	// 以我们现在计算机的实际情况，这个数量还是够用的
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    // 我们说了，线程池的状态存放在高 3 位中
	// 运算结果为 111跟29个0：111 00000000000000000000000000000
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 000 00000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 001 00000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS;
    // 010 00000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 011 00000000000000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    // 将整数 c 的低 29 位修改为 0，就得到了线程池的状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 将整数 c 的高 3 为修改为 0，就得到了线程池中的线程数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    /*
     * Bit field accessors that don't require unpacking ctl.
     * These depend on the bit layout and on workerCount being never negative.
     */

    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
```

线程池中的各个状态和状态变化的转换过程：
* RUNNING:接受新的任务，处理等待队列中的任务
* SHUTDOWN：不接受新的任务提交，但是会继续处理等待队列中的任务
* STOP：不接受新的任务提交，不再处理等待队列中的任务，中断正在执行任务的线程
* TIDYING：所有的任务都销毁了，workCount 为 0。线程池的状态在转换为 TIDYING 状态时，会执行钩子方法 terminated()
* TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个

> RUNNING 定义为 -1，SHUTDOWN 定义为 0，其他的都比 0 大，所以等于 0 的时候不能提交任务，大于 0 的话，连正在执行的任务也需要中断。

* RUNNING -> SHUTDOWN：当调用了 shutdown() 后，会发生这个状态转换，这也是最重要的
* (RUNNING or SHUTDOWN) -> STOP：当调用 shutdownNow() 后，会发生这个状态转换，这下要清楚 shutDown() 和 shutDownNow() 的区别了
* SHUTDOWN -> TIDYING：当任务队列和线程池都清空后，会由 SHUTDOWN 转换为 TIDYING
* STOP -> TIDYING：当任务队列清空后，发生这个转换
* TIDYING -> TERMINATED：这个前面说了，当 terminated() 方法结束后

### Worker
内部类 Worker，因为 Doug Lea 把线程池中的线程包装成了一个个 Worker，翻译成工人，就是线程池中做任务的线程。所以到这里，我们知道任务是 Runnable（内部叫 task 或 command），线程是 Worker。

Worker 这里又用到了抽象类 AbstractQueuedSynchronizer。
```Java
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
        // 这个是真正的线程，执行任务
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        // 在创建线程的时候，如果同时指定了
    	// 这个线程起来以后需要执行的第一个任务，那么第一个任务就是存放在这里的(线程可不止执行这一个任务)
    	// 当然了，也可以为 null，这样线程起来了，自己到任务队列（BlockingQueue）中取任务（getTask 方法）就行了
        Runnable firstTask;
        /** Per-thread task counter */
        // 用于存放此线程完全的任务数，注意了，这里用了 volatile，保证可见性
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        // Worker 只有这一个构造方法，传入 firstTask，也可以传 null
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            // 调用 ThreadFactory 来创建一个新的线程
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
        	// 这里调用了外部类的 runWorker 方法
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


```Java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

### execute
各种方法都最终依赖于 execute 方法
```Java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        //表示 “线程池状态” 和 “线程数” 的整数
        int c = ctl.get();
        // 如果当前线程数少于核心线程数，那么直接添加一个 worker 来执行任务，
        // 创建一个新的线程，并把当前任务 command 作为这个线程的第一个任务(firstTask)
        if (workerCountOf(c) < corePoolSize) {
        	// 添加任务成功，那么就结束了。提交任务嘛，线程池已经接受了这个任务，这个方法也就可以返回了
    	    // 至于执行的结果，到时候会包装到 FutureTask 中。
	        // 返回 false 代表线程池不允许提交任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 到这里说明，要么当前线程数大于等于核心线程数，要么刚刚 addWorker 失败了
        // 如果线程池处于 RUNNING 状态，把这个任务添加到任务队列 workQueue 中
        if (isRunning(c) && workQueue.offer(command)) {
        	/* 这里面说的是，如果任务进入了 workQueue，我们是否需要开启新的线程
	         * 因为线程数在 [0, corePoolSize) 是无条件开启新的线程
	         * 如果线程数已经大于等于 corePoolSize，那么将任务添加到队列中，然后进到这里
	         */
            int recheck = ctl.get();
            // 如果线程池已不处于 RUNNING 状态，那么移除已经入队的这个任务，并且执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池还是 RUNNING 的，并且线程数为 0，那么开启新的线程
            // 到这里，我们知道了，这块代码的真正意图是：担心任务提交到队列中了，但是线程都关闭了
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        } 
		// 如果 workQueue 队列满了，那么进入到这个分支
    	// 以 maximumPoolSize 为界创建新的 worker，
    	// 如果失败，说明当前线程数已经达到 maximumPoolSize，执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

addWorker(Runnable firstTask, boolean core) 方法，我们看看它是怎么创建新的线程的：
```Java
    // 第一个参数是准备提交给这个线程执行的任务，可以为 null
    // 第二个参数为 true 代表使用核心线程数 corePoolSize 作为创建线程的界线，也就说创建这个线程的时候
    // 如果线程池中的线程总数已经达到 corePoolSize，那么不能响应这次创建线程的请求
    // 如果是 false，代表使用最大线程数 maximumPoolSize 作为界线
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 满足以下条件之一，不创建新的worker
            // 1. 线程池>SHUTDOWN(STOP, TIDYING, 或 TERMINATED)
            // 2. firstTask != null
            // 3. workQueue.isEmpty()
            // 还是状态控制的问题，当线程池处于 SHUTDOWN 的时候，不允许提交任务，但是已有的任务继续执行
            // 当状态大于 SHUTDOWN 时，不允许提交任务，且中断正在执行的任务
            // 如果线程池处于 SHUTDOWN，但是 firstTask 为 null，且 workQueue 非空，那么是允许创建 worker 的
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 如果成功，那么就是所有创建线程前的条件校验都满足了，准备创建线程执行任务了
                // 这里失败的话，说明有其他线程也在尝试往线程池中创建线程
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 由于有并发，重新再读取一下 ctl
                c = ctl.get();  // Re-read ctl
                // 正常如果是 CAS 失败的话，进到下一个里层的for循环就可以了
                // 可是如果是因为其他线程的操作，导致线程池的状态发生了变更，如有其他线程关闭了这个线程池
                // 那么需要回到外层的for循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        //到这里，我们认为在当前这个时刻，可以开始创建线程来执行任务了

        // worker 是否已经启动
        boolean workerStarted = false;
        // 是否已将这个 worker 添加到 workers 这个 HashSet 中
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            // 取 worker 中的线程对象
            final Thread t = w.thread;
            if (t != null) {
                // 这个是整个类的全局锁，持有这个锁才能让下面的操作“顺理成章”，
                // 因为关闭一个线程池需要这个锁，至少我持有锁的期间，线程池不会被关闭
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                   // 小于 SHUTTDOWN 那就是 RUNNING，这个自不必说，是最正常的情况
                   // 如果等于 SHUTDOWN，前面说了，不接受新的任务，但是会继续执行等待队列中的任务
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // worker 里面的 thread 不能是已经启动的
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 加到 workers 这个 HashSet 中
                        workers.add(w);
                        int s = workers.size();
                        // largestPoolSize 用于记录 workers 中的个数的最大值
                        // 因为 workers 是不断增加减少的，通过这个值可以知道线程池的大小曾经达到的最大值
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 添加成功的话，启动这个线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                // 如果线程没有启动，需要做一些清理工作，如前面 workCount 加了 1，将其减掉
                addWorkerFailed(w);
        }
        // 返回线程是否启动成功
        return workerStarted;
    }

    // workers 中删除掉相应的 worker
    // workCount 减 1
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

worker 中的线程 start 后，其 run 方法会调用 runWorker 方法：

```Java
// Worker 类的 run() 方法
public void run() {
    runWorker(this);
}

// 此方法由 worker 线程启动后调用，这里用一个 while 循环来不断地从等待队列中获取任务并执行
// 前面说了，worker 在初始化的时候，可以指定 firstTask，那么第一个任务也就可以不需要从队列中获取
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        // 该线程的第一个任务(如果有的话)
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 循环调用 getTask 获取任务
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 如果线程池状态大于等于 STOP，那么意味着该线程也要中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 这是一个钩子方法，留给需要的子类实现
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        // 这里不允许抛出 Throwable，所以转换为 Error
                        thrown = x; throw new Error(x);
                    } finally {
                        // 也是一个钩子方法，将 task 和异常作为参数，留给需要的子类实现
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 置空 task，准备 getTask 获取下一个任务
                    task = null;
                    // 累加完成的任务数
                    w.completedTasks++;
                    // 释放掉 worker 的独占锁
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 如果到这里，需要执行线程关闭：
            // 1. 说明 getTask 返回 null，也就是说，这个 worker 的使命结束了，执行关闭
            // 2. 任务执行过程中发生了异常
            // 第一种情况，已经在代码处理了将 workCount 减 1，这个在 getTask 方法分析中会说
            // 第二种情况，workCount 没有进行处理，所以需要在 processWorkerExit 中处理
            processWorkerExit(w, completedAbruptly);
        }
    }
```