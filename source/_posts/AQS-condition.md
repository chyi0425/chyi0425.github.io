---
title: AQS-condition
date: 2018-06-30 12:27:15
tags: [Java,concurrency] #文章标签，多于一项时用这种格式
---

## Condition
每个ReentrantLock实例可以通过调用多次newCondition产生多个ConditionObject的实例：
```Java
final ConditionObject newCondition() {
    return new ConditionObject();
}

public class ConditionObject implements Condition,java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        // 条件队列的第一个节点
          // 不要管这里的关键字 transient，是不参与序列化的意思
        private transient Node firstWaiter;
        // 条件队列的最后一个节点
        private transient Node lastWaiter;
        ......
```
在上一篇介绍AQS时，我们有一个**阻塞队列**，用于保存等待获取锁的线程的队列。这里我们引入另一个概念，叫做**条件队列(Condition queue)**。
![示意图](/img/aqs2-2.png)
回顾下Node的属性：
```Java
// prev 和 next 用于实现阻塞队列的双向链表，nextWaiter 用于实现条件队列的单向链表

volatile int waitStatus; // 可取值 0、CANCELLED(1)、SIGNAL(-1)、CONDITION(-2)、PROPAGATE(-3)
volatile Node prev;
volatile Node next;
volatile Thread thread;
Node nextWaiter;

// prev 和 next 用于实现阻塞队列的双向链表，nextWaiter 用于实现条件队列的单向链表
```
基本上，上图很形象的形容了condition的处理流程。
1. 我们知道一个ReentrantLock实例可以通过调用newCondition()来产生多个Condition实例。注意ConditionObject只有2个属性firstWaiter和lastWaiter;
2. 每个condition有一个关联的条件队列，如线程1调用condition.await()方法即可将当前线程1包装成Node后加入到**条件队列**中，然后阻塞在这里，不继续往下执行，条件队列是一个单向链表;
3. 调用condition1.signal()会将condition1对应的条件队列firstWaiter移到**阻塞队列**的队尾，等待获取锁，获取到锁后await方法返回，继续往下执行。

接下来，我们一步步按照流程来走代码分析，我们先来看看 wait 方法：
```Java
// 首先，这个方法是可被中断的，不可被中断的是另一个方法 awaitUninterruptibly()
// 这个方法会阻塞，直到调用 signal 方法（指 signal() 和 signalAll()，下同），或被中断
public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
			// 添加到condition的条件队列中
            Node node = addConditionWaiter();
            // 释放锁，返回值是释放锁之前的state值
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            // 这里退出循环有2个情况
            // 1. isOnSyncQueue(node)返回true，表示当前node已经转移到阻塞队列了
            // 2. checkInterruptWhileWaiting(node))！=0 会执行break，表示线程中断
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 被唤醒后，将进入阻塞队列，等待获取锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```
### 将节点加入到条件队列
addConditionWaiter()是将当前节点加入到条件队列
```Java
		// 将当前线程包装成Node，插入队尾
		private Node addConditionWaiter() {
            Node t = lastWaiter;
            // 如果条件队列的最后一个节点取消了，将其清除出去
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
            	// 遍历整个条件队列，将已取消的所有节点移出队列
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            // 如果队列为空
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
```
当 await 的时候如果发生了取消操作（这点之后会说），或者是在节点入队的时候，发现最后一个节点是被取消的，会调用一次unlinkCancelledWaiters()。
```Java
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
### 完全释放独占锁
回到 wait 方法，节点入队了以后，会调用**int savedState = fullyRelease(node)**; 方法释放锁，注意，这里是完全释放独占锁，因为 ReentrantLock 是可以重入的。
```Java
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
### 等待进入阻塞队列
```Java
	// 在节点入条件队列的时候，初始化时设置了waitStatus=Node.CONDITION
	// signal 的时候需要将节点从条件队列移到阻塞队列
	// 这个方法就是判断 node 是否已经移动到阻塞队列了
	final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);
    }

	/**
     * Returns true if node is on sync queue by searching backwards from tail.
     * Called only when needed by isOnSyncQueue.
     * @return true if present
     */
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
isOnSyncQueue(node) 返回 false 的话，那么进到 LockSupport.park(this); 这里线程挂起。

### signal 唤醒线程，转移到阻塞队列
唤醒操作通常由另一个线程来操作，就像生产者-消费者模式中，如果线程因为等待消费而挂起，那么当生产者生产了一个东西后，会调用 signal 唤醒正在等待的线程来消费。
```Java
		// 唤醒等待了最久的线程，将这个线程对应的node从条件队列转移到阻塞队列
		public final void signal() {
			// 调用signal方法的线程必须持有当前的独占锁
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        // 遍历条件队列，找出第一个**需要**转移(**到阻塞队列**)的Node
        private void doSignal(Node first) {
            do {
            	// 将firstWaiter指向first节点后面的第一个
            	// 如果将对头移除后，后面没有节点在等待了，那么需要将lastWaiter置为null
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                // 因为first马上要被移进阻塞队列了，和条件队列的链接关系在这里断掉
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
                     // 这里while循环，如果first转移不成功，那么选择first后面的第一个节点
        }

        final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
         // CAS 如果失败，说明此 node 的 waitStatus 已不是 Node.CONDITION，说明节点已经取消，
    	// 既然已经取消，也就不需要转移了，方法返回，转移后面一个节点
    	// 否则，将 waitStatus 置为 0
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        // 自旋进入阻塞队列的队尾
        // 注意，这里的返回值 p 是 node 在阻塞队列的前驱节点
        Node p = enq(node);
        int ws = p.waitStatus;
        // 如果ws>0 说明node在阻塞队列中的前驱节点取消了等待锁，直接唤醒node对应的线程
        // 如果 ws <= 0, 那么 compareAndSetWaitStatus 将会被调用，上篇介绍的时候说过，节点入队后，需要把前驱节点的状态设为 Node.SIGNAL(-1)
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        	// 如果前驱节点取消或者 CAS 失败，会进到这里唤醒线程
            LockSupport.unpark(node.thread);
        return true;
    }
```

### 唤醒后检查中断状态
上一步 signal 之后，我们的线程由条件队列转移到了阻塞队列，之后就准备获取锁了。只要重新获取到锁了以后，继续往下执行。
```Java
int interruptMode = 0;
while (!isOnSyncQueue(node)) {
    // 线程挂起
    LockSupport.park(this);

    if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
        break;
}
```
interruptMode 可以取值为 REINTERRUPT（1），THROW_IE（-1），0
- REINTERRUPT: 代表await返回的时候，需要重新设置中断状态
- THROW_IE: 代表await返回的时候，需要抛出InterruptedException
- 0: 说明await期间，没有发生中断
如下三种情况会让LockSUpport.park(this);这句返回继续往下执行：
1. 常规路径。signal->转移节点到阻塞队列->获取了锁(unpark)
2. 线程中断。在park的时候，另外一个线程对这个线程进行了中断
3. signal的时候，如果前驱节点取消或者 CAS 失败
4. 假唤醒。这个也是存在的，和Object.wait()类似，都有这个问题
线程唤醒后第一步是调用checkInterruptWhileWaiting(node),用于判断是否在线程挂起期间发生了中断，如果发生了中断，是signal调用之前中断的，还是signal之后发生的中断。
```Java
        /**
         * Checks for interrupt, returning THROW_IE if interrupted
         * before signalled, REINTERRUPT if after signalled, or
         * 0 if not interrupted.
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

	    /**
	     * Transfers node, if necessary, to sync queue after a cancelled wait.
	     * Returns true if thread was cancelled before being signalled.
	     *
	     * @param node the node
	     * @return true if cancelled before the node was signalled
	     */
	    final boolean transferAfterCancelledWait(Node node) {
	    	// 用 CAS 将节点状态设置为 0 
    		// 如果这步 CAS 成功，说明是 signal 方法之前发生的中断，因为如果 signal 先发生的话，signal 中会将 waitStatus 设置为 0
	        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
	        	// 将节点放入阻塞队列
        		// 即使中断了，依然会转移到阻塞队列
	            enq(node);
	            return true;
	        }
	        /*
	         * If we lost out to a signal(), then we can't proceed
	         * until it finishes its enq().  Cancelling during an
	         * incomplete transfer is both rare and transient, so just
	         * spin.
	         */
	    	// 到这里是因为 CAS 失败，肯定是因为 signal 方法已经将 waitStatus 设置为了 0
		    // signal 方法会将节点转移到阻塞队列，但是可能还没完成，这边自旋等待其完成
		    // 当然，这种事情还是比较少的吧：signal 调用之后，没完成转移之前，发生了中断
	        while (!isOnSyncQueue(node))
	            Thread.yield();
	        return false;
	    }


```

### 获取独占锁
```Java
// 被唤醒后，将进入阻塞队列，等待获取锁
if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
```
由于 while 出来后，我们确定节点已经进入了阻塞队列，准备获取锁。
acquireQueued(node, savedState) 的返回值就是代表线程是否被中断。如果返回 true，说明被中断了，而且 interruptMode != THROW_IE，说明在 signal 之前就发生中断了，这里将 interruptMode 设置为 REINTERRUPT，用于待会重新中断。
```Java
			//在signal的时候会将节点转移到阻塞队列，有一步node.nextWaiter = null，将断开节点和条件队列的联系
			//但是如果signal 之前就中断了，也需要将节点进行转移到阻塞队列，这部分转移的时候，是没有设置 node.nextWaiter = null 
			if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
```

### 处理中断状态
interruptMode的意义
- 0: 什么都不做
- THROW_IE: await方法抛出InterruptException
- REINTERRUPT: 重新中断当前线程
```Java
        /**
         * Throws InterruptedException, reinterrupts current thread, or
         * does nothing, depending on mode.
         */
        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }
```

带超时机制的await
```Java
public final long awaitNanos(long nanosTimeout) 
                  throws InterruptedException
public final boolean awaitUntil(Date deadline)
                throws InterruptedException
public final boolean await(long time, TimeUnit unit)
                throws InterruptedException
```
挑选await(long time, TimeUnit unit)进行分析
```Java
public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
			// 转换时间为纳秒
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            // 过期时间 = 当前时间 + 等待时间 
            final long deadline = System.nanoTime() + nanosTimeout;
            // 用于返回await是否超时
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
            	// 表示时间到了
                if (nanosTimeout <= 0L) {
					// 这里因为要 break 取消等待了。取消等待的话一定要调用 transferAfterCancelledWait(node) 这个方法
		            // 如果这个方法返回 true，在这个方法内，将节点转移到阻塞队列成功
		            // 返回 false 的话，说明 signal 已经发生，signal 方法将节点转移了。也就是说没有超时
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                // spinForTimeoutThreshold 的值是 1000 纳秒，也就是 1 毫秒
        		// 也就是说，如果不到 1 毫秒了，那就不要选择 parkNanos 了，自旋的性能反而更好
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                // 得到剩余时间
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }
```
超时的思路还是很简单的，不带超时参数的 await 是 park，然后等待别人唤醒。而现在就是调用 parkNanos 方法来休眠指定的时间，醒来后判断是否 signal 调用了，调用了就是没有超时，否则就是超时了。超时的话，自己来进行转移到阻塞队列，然后抢锁。

### 不抛出 InterruptedException 的 await
