---
title: AbstractQueuedSynchronizer
date: 2018-06-20 10:05:41
tags: [Java,concurrency] #文章标签，多于一项时用这种格式
toc: true
---
## AQS结构

先来看看AQS有哪些属性
``` Java
   /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     * 头结点，可以理解为当前持有锁的线程
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

	/**
     * The synchronization state.
     * 当前锁的状态，0表示没有被占用
     */
    private volatile int state;

    // 代表当前持有独占锁的线程，举个最重要的使用例子，因为锁可以重入
	// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
	// if (currentThread == getExclusiveOwnerThread()) {state++}
	/**
     * The current owner of exclusive mode synchronization.
     */
    private transient Thread exclusiveOwnerThread;

```
AbstractQueuedSynchronizer的等待队列示意如下所示，queue(阻塞队列)不包含head。
![示意图](/img/aqs-0.png)
等待队列中每个线程被包装成一个node。
``` Java
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        // 标记节点当前在共享模式下
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        // 标记节点当前在独占模式下
        static final Node EXCLUSIVE = null;

        // ======== 下面的几个int常量是给waitStatus用的 ===========
        /** waitStatus value to indicate thread has cancelled */
        // 代表此线程取消了争抢这个锁
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        // 官方的描述是，其表示当前node的后继节点对应的线程需要被唤醒
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        // 取值为上面的1、-1、-2、-3，或者0(以后会讲到)
		// 这么理解，暂时只需要知道如果这个值 大于0 代表此线程取消了等待，
		// 也许就是说半天抢不到锁，不抢了，ReentrantLock是可以指定timeouot的。。。
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        // 前驱节点的引用
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        // 后继节点的引用
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        // 这个就是线程本尊
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```
Node的数据结构，主要是thread、waitStatus、pre、next四个属性。

首先我们来看下ReentrantLock
ReentrantLock 在内部用了内部类 Sync 来管理锁，所以真正的获取锁和释放锁是由 Sync 的实现类来控制的。
``` Java
/**
     * Base of synchronization control for this lock. Subclassed
     * into fair and nonfair versions below. Uses AQS state to
     * represent the number of holds on the lock.
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {

    }
```
Sync 有两个实现，分别为 NonfairSync（非公平锁）和 FairSync（公平锁），我们看 FairSync 部分。
``` Java
    /**
     * Creates an instance of {@code ReentrantLock} with the
     * given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

## 线程抢锁
``` Java
    /**
     * Sync object for fair locks
     */
	static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        // 争锁
        final void lock() {
            acquire(1);
        }

        // 如果tryAcquire(arg)返回true，也就结束了
        // 否则，acquireQueued方法会将线程压倒队列中
	    /**
	     * Acquires in exclusive mode, ignoring interrupts.  Implemented
	     * by invoking at least once {@link #tryAcquire},
	     * returning on success.  Otherwise the thread is queued, possibly
	     * repeatedly blocking and unblocking, invoking {@link
	     * #tryAcquire} until success.  This method can be used
	     * to implement method {@link Lock#lock}.
	     *
	     * @param arg the acquire argument.  This value is conveyed to
	     *        {@link #tryAcquire} but is otherwise uninterpreted and
	     *        can represent anything you like.
	     */
	    public final void acquire(int arg) {//此时arg == 1
	    	// 首先truAcquire(1)试一下，如果成功，就不需要进队列排队了
	    	// 对于公平锁的语义就是：本来就没人持有锁，根本没必要进队列等待(又是挂起，又是等待被唤醒的)
	        if (!tryAcquire(arg) &&
	        	// tryAcquire(arg)没有成功，这个时候需要把当前线程挂起，放到阻塞队列中。
	            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
	            selfInterrupt();
	    }
        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        // 尝试直接获取锁，返回值是boolean，代表是否获取到锁
		// 返回true：1.没有线程在等待锁；2.重入锁，线程本来就持有锁，也就可以理所当然可以直接获取
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // state == 0 此时此刻没有线程持有锁
            if (c == 0) {
            	// 虽然此时此刻锁是可以用的，但是这是公平锁，既然是公平，就得讲究先来后到，
            	// 看看有没有别人在队列中等了半天了
                if (!hasQueuedPredecessors() &&
                	// 如果没有线程在等待，那就用CAS尝试一下，成功了就获取到锁了，
                	// 不成功的话，只能说明一个问题，就在刚刚几乎同一时刻有个线程抢先了 =_=
                	// 因为刚刚还没人的，我判断过了
                    compareAndSetState(0, acquires)) {
                    // 到这里就是获取到锁了，标记一下，告诉大家，现在是我占用了锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 会进入这个else if分支，说明是重入了，需要操作：state=state+1
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            // 如果到这里，说明前面的if和else if都没有返回true，说明没有获取到锁
        	// 回到上面一个外层调用方法继续看:addWaiter
        	// if (!tryAcquire(arg) 
        	//        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
        	//     selfInterrupt();
            return false;
        }

        /**
	    * Creates and enqueues node for current thread and given mode.
	    *
	    * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
	    * @return the new node
	    */
		// 此方法的作用是把线程包装成node，同时进入到队列中
	    // 参数mode此时是Node.EXCLUSIVE，代表独占模式
	    private Node addWaiter(Node mode) {
	        Node node = new Node(Thread.currentThread(), mode);
	        // Try the fast path of enq; backup to full enq on failure
	        // 以下几行代码想把当前node加到链表的最后面去，也就是进到阻塞队列的最后
	        Node pred = tail;

	        // tail!=null => 队列不为空(tail==head的时候，其实队列是空的)
	        if (pred != null) {
	        	// 设置自己的前驱 为当前的队尾节点
	            node.prev = pred;
	            // 用CAS把自己设置为队尾, 如果成功后，tail == node了
	            if (compareAndSetTail(pred, node)) {
	                pred.next = node;
	                return node;
	            }
	        }

	        // 仔细看看上面的代码，如果会到这里，
	        // 说明 pred==null(队列是空的) 或者 CAS失败(有线程在竞争入队)
	        enq(node);
	        return node;
	    }

	    /**
     	* Inserts node into queue, initializing if necessary. See picture above.
     	* @param node the node to insert
     	* @return node's predecessor
     	*/
     	 // 采用自旋的方式入队
    	// 之前说过，到这个方法只有两种可能：等待队列为空，或者有线程竞争入队，
    	// 自旋在这边的语义是：CAS设置tail过程中，竞争一次竞争不到，我就多次竞争，总会排到的
	    private Node enq(final Node node) {
	        for (;;) {
	            Node t = tail;
	            // 之前说过，队列为空也会进来这里
	            if (t == null) { // Must initialize
	            	// 初始化head节点
                	// 细心的读者会知道原来head和tail初始化的时候都是null，反正我不细心
                	// 还是一步CAS，你懂的，现在可能是很多线程同时进来呢
	                if (compareAndSetHead(new Node()))
						// 给后面用：这个时候head节点的waitStatus==0, 看new Node()构造方法就知道了

                    	// 这个时候有了head，但是tail还是null，设置一下，
                    	// 把tail指向head，放心，马上就有线程要来了，到时候tail就要被抢了
                    	// 注意：这里只是设置了tail=head，这里可没return哦，没有return，没有return
                    	// 所以，设置完了以后，继续for循环，下次就到下面的else分支了
	                    tail = head;
	            } else {
		           	// 下面几行，和上一个方法 addWaiter 是一样的，
	                // 只是这个套在无限循环里，反正就是将当前线程排到队尾，有线程竞争的话排不上重复排
	                node.prev = t;
	                if (compareAndSetTail(t, node)) {
	                    t.next = node;
	                    return t;
	                }
	            }
	        }
	    }

	    /**
	     * Acquires in exclusive uninterruptible mode for thread already in
	     * queue. Used by condition wait methods as well as acquire.
	     *
	     * @param node the node
	     * @param arg the acquire argument
	     * @return {@code true} if interrupted while waiting
	     */
        // 现在，又回到这段代码了
	    // if (!tryAcquire(arg) 
	    //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
	    //     selfInterrupt();

	    // 下面这个方法，参数node，经过addWaiter(Node.EXCLUSIVE)，此时已经进入阻塞队列
    	// 注意一下：如果acquireQueued(addWaiter(Node.EXCLUSIVE), arg))返回true的话，
    	// 意味着上面这段代码将进入selfInterrupt()，所以正常情况下，下面应该返回false
	    // 这个方法非常重要，应该说真正的线程挂起，然后被唤醒后去获取锁，都在这个方法里了
	    final boolean acquireQueued(final Node node, int arg) {
	        boolean failed = true;
	        try {
	            boolean interrupted = false;
	            for (;;) {
	                final Node p = node.predecessor();
	                // p == head 说明当前节点虽然进到了阻塞队列，但是是阻塞队列的第一个，因为它的前驱是head
	                // 注意，阻塞队列不包含head节点，head一般指的是占有锁的线程，head后面的才称为阻塞队列
	                // 所以当前节点可以去试抢一下锁
	                // 这里我们说一下，为什么可以去试试：
	                // 首先，它是队头，这个是第一个条件，其次，当前的head有可能是刚刚初始化的node，
	                // enq(node) 方法里面有提到，head是延时初始化的，而且new Node()的时候没有设置任何线程
	                // 也就是说，当前的head不属于任何一个线程，所以作为队头，可以去试一试，
	                // tryAcquire已经分析过了, 忘记了请往前看一下，就是简单用CAS试操作一下state
	                if (p == head && tryAcquire(arg)) {
	                    setHead(node);
	                    p.next = null; // help GC
	                    failed = false;
	                    return interrupted;
	                }
					// 到这里，说明上面的if分支没有成功，要么当前node本来就不是队头，
	                // 要么就是tryAcquire(arg)没有抢赢别人，继续往下看
	                if (shouldParkAfterFailedAcquire(p, node) &&
	                    parkAndCheckInterrupt())
	                    interrupted = true;
	            }
	        } finally {
	            if (failed)
	                cancelAcquire(node);
	        }
	    }

	    /**
	     * Checks and updates status for a node that failed to acquire.
	     * Returns true if thread should block. This is the main signal
	     * control in all acquire loops.  Requires that pred == node.prev.
	     *
	     * @param pred node's predecessor holding status
	     * @param node the node
	     * @return {@code true} if thread should block
	     */
		// 刚刚说过，会到这里就是没有抢到锁呗，这个方法说的是："当前线程没有抢到锁，是否需要挂起当前线程？"
	    // 第一个参数是前驱节点，第二个参数才是代表当前线程的节点
	    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	        int ws = pred.waitStatus;
	        // 前驱节点的 waitStatus == -1 ，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
	        if (ws == Node.SIGNAL)
	            /*
	             * This node has already set status asking a release
	             * to signal it, so it can safely park.
	             */
	            return true;
	        // 前驱节点 waitStatus大于0 ，之前说过，大于0 说明前驱节点取消了排队。这里需要知道这点：
	        // 进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。
	        // 所以下面这块代码说的是将当前节点的prev指向waitStatus<=0的节点，
	        // 简单说，就是为了找个好爹，因为你还得依赖它来唤醒呢，如果前驱节点取消了排队，
	        // 找前驱节点的前驱节点做爹，往前循环总能找到一个好爹的
	        if (ws > 0) {
	            /*
	             * Predecessor was cancelled. Skip over predecessors and
	             * indicate retry.
	             */
	            do {
	                node.prev = pred = pred.prev;
	            } while (pred.waitStatus > 0);
	            pred.next = node;
	        } else {
	            /*
	             * waitStatus must be 0 or PROPAGATE.  Indicate that we
	             * need a signal, but don't park yet.  Caller will need to
	             * retry to make sure it cannot acquire before parking.
	             */
	        	// 仔细想想，如果进入到这个分支意味着什么
        	    // 前驱节点的waitStatus不等于-1和1，那也就是只可能是0，-2，-3
    	        // 在我们前面的源码中，都没有看到有设置waitStatus的，所以每个新的node入队时，waitStatu都是0
	            // 用CAS将前驱节点的waitStatus设置为Node.SIGNAL(也就是-1)
	            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
	        }
	        return false;
	    }

	    // private static boolean shouldParkAfterFailedAcquire(Node pred, Node node)
    	// 这个方法结束根据返回值我们简单分析下：
   		// 如果返回true, 说明前驱节点的waitStatus==-1，是正常情况，那么当前线程需要被挂起，等待以后被唤醒
    	//        我们也说过，以后是被前驱节点唤醒，就等着前驱节点拿到锁，然后释放锁的时候叫你好了
	    // 如果返回false, 说明当前不需要被挂起，为什么呢？往后看


	    // 跳回到前面是这个方法
	    // if (shouldParkAfterFailedAcquire(p, node) &&
	    //                parkAndCheckInterrupt())
	    //                interrupted = true;

	    // 1. 如果shouldParkAfterFailedAcquire(p, node)返回true，
	    // 那么需要执行parkAndCheckInterrupt():
		// 这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的
	    // 这里用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒=======

	    /**
	     * Convenience method to park and then check if interrupted
	     *
	     * @return {@code true} if interrupted
	     */
	    private final boolean parkAndCheckInterrupt() {
	        LockSupport.park(this);
	        return Thread.interrupted();
	    }
   		// 2. 接下来说说如果shouldParkAfterFailedAcquire(p, node)返回false的情况

   		// 仔细看shouldParkAfterFailedAcquire(p, node)，我们可以发现，其实第一次进来的时候，一般都不会返回true的，原因很简单，前驱节点的waitStatus=-1是依赖于后继节点设置的。也就是说，我都还没给前驱设置-1呢，怎么可能是true呢，但是要看到，这个方法是套在循环里的，所以第二次进来的时候状态就是-1了。

  	  	// 解释下为什么shouldParkAfterFailedAcquire(p, node)返回false的时候不直接挂起线程：
   		// => 是为了应对在经过这个方法后，node已经是head的直接后继节点了。剩下的读者自己想想吧。

    }
```

## 解锁操作
最后，就是要需要介绍下唤醒的动作了，我们知道，正常情况下，如果线程没有获取到锁
``` Java
    /**
     * Attempts to release this lock.
     *
     * <p>If the current thread is the holder of this lock then the hold
     * count is decremented.  If the hold count is now zero then the lock
     * is released.  If the current thread is not the holder of this
     * lock then {@link IllegalMonitorStateException} is thrown.
     *
     * @throws IllegalMonitorStateException if the current thread does not
     *         hold this lock
     */
    public void unlock() {
        sync.release(1);
    }

    /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
			// 是否完全释放锁
            boolean free = false;
			// 其实就是重入的问题，如果c==0，也就是说没有嵌套锁了，可以释放了，否则还不能释放掉
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
	}

	// 唤醒后继节点
	// 从上面调用处知道，参数node是head头结点
    /**
     * Wakes up node's successor, if one exists.
     *
     * @param node the node
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
        	// 如果head节点当前waitStatus<0, 将其修改为0
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    	// 下面的代码就是唤醒后继节点，但是有可能后继节点取消了等待（waitStatus==1）
	    // 从队尾往前找，找到waitStatus<=0的所有节点中排在最前面的
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 从后往前找，仔细看代码，不必担心中间有节点取消(waitStatus==1)的情况
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
        	// 唤醒线程
            LockSupport.unpark(s.thread);
    }


```
唤醒线程以后，被唤醒的线程将从以下代码中继续往前走：
``` Java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 刚刚线程被挂起在这里了
    return Thread.interrupted();
}
// 又回到这个方法了：acquireQueued(final Node node, int arg)，这个时候，node的前驱是head了
```

## 总结
在并发环境下，加锁和解锁需要以下三个部件的协调：
1. 锁状态。我们要知道是不是被别的线程占有了，这个就是state的作用，为0的时候表示没有线程占有锁，用CAS将state设为1，如果CAS成功，说明抢到了锁，这样其他线程就抢不到了，如果锁重入的话，state进行+1 就可以，解锁就是减 1，直到 state 又变为 0，代表释放锁，所以 lock() 和 unlock() 必须要配对啊。然后唤醒等待队列中的第一个线程，让其来占有锁。
2. 线程的阻塞和解除阻塞。AQS中采用了LockSupport.park(thread) 来挂起线程，用 unpark 来唤醒线程。
3. 阻塞队列。因为争抢锁的线程可能很多，但是只有一个线程能拿到锁，其他的线程都必须等待，这个时候就需要一个queue来管理这些锁，AQS 用的是一个 FIFO 的队列，就是一个链表，每个 node 都持有后继节点的引用。

最后自己整理了lock的xmind图
http://naotu.baidu.com/file/6fae44361ac80b5429fdf1a40b0c4de2?token=e35a37d6d08f12a8

![示意图](/img/aqs-1.png)
![示意图](/img/aqs-2.png)
![示意图](/img/aqs-3.png)