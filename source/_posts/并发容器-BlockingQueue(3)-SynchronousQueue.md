---
title: SynchronousQueue
date: 2016-8-4 18:25:02
tags: [BlockingQueue, SynchronousQueue]
---

SynchronousQueue 是 blocking queue 队列的一种，它的作用是，当一个线程执行插入操作（put）的时候，必须等待另一个线程执行对应的删除操作（take），反之，一个线程如果要执行删除操作则必须等待另一个线程执行插入操作。

**What We CAN'T DO IN SynchronousQueue:**

* SynchronousQueue内部实现中并没有容量的概念，所以不能执行 peek 操作，因为只有在线程试图执行remove操作的时候才有一个元素存在。

* 不能成功插入一个元素（add, off, put方法）除非另一个线程试图执行删除操作。

* 无法对 SynchronousQueue 进行 iterator, 因为返回的是一个空的iterator

使用场景：

They are well suited for handoff designs, in which an object running in one thread must sync up with an object running in another thread in order to hand it some information, event, or task. 

它们非常适用于越区切换的设计，其中在一个线程中运行的对象必须与一个对象在另一个线程以便把它的一些信息，事件或任务运行同步。

这个类被用在 java.util.concurrent.Executors.newCachedThreadPool 方法中，这个方法用来创建线程池，这个方法的作用是用来创建一个可以执行短暂的异步任务的线程池。（These pools will typically improve the performance of programs that **execute many short-lived asynchronous tasks.**）

具体实现，如下：
``` java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
```
ThreadPoolExecutor.ThreadPoolExecutor 查看文档，可以看到：
Direct handoffs. A good default choice for a work queue is a SynchronousQueue that hands off tasks to threads without otherwise holding them. Here, an attempt to queue a task will fail if no threads are immediately available to run it, so a new thread will be constructed. This policy avoids lockups when handling sets of requests that might have internal dependencies. Direct handoffs generally require unbounded maximumPoolSizes to avoid rejection of new submitted tasks. This in turn admits the possibility of unbounded thread growth when commands continue to arrive on average faster than they can be processed. 

## 构造过程

``` java
public SynchronousQueue(boolean fair) {

	// 无参数的构造函数默认是：
	// new TransferStack()
	transferer = fair ? new TransferQueue() : new TransferStack();
}
```

## Transferer

## TransferStack

### put过程

``` java
public void put(E o) throws InterruptedException {
    if (o == null) throw new NullPointerException();
    if (transferer.transfer(o, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}
```

``` java
Object transfer(Object e, boolean timed, long nanos) {
        /*
         * Basic algorithm is to loop trying one of three actions:
         *
         * 1. If apparently empty or already containing nodes of same
         *    mode, try to push node on stack and wait for a match,
         *    returning it, or null if cancelled.
         *
         * 2. If apparently containing node of complementary mode,
         *    try to push a fulfilling node on to stack, match
         *    with corresponding waiting node, pop both from
         *    stack, and return matched item. The matching or
         *    unlinking might not actually be necessary because of
         *    other threads performing action 3:
         *
         * 3. If top of stack already holds another fulfilling node,
         *    help it out by doing its match and/or pop
         *    operations, and then continue. The code for helping
         *    is essentially the same as for fulfilling, except
         *    that it doesn't return the item.
         */

		// (1) 确定 s: mode = DATA, next = head, item = e,
        SNode s = null; // constructed/reused as needed
        int mode = (e == null) ? REQUEST : DATA;

        for (;;) {
            SNode h = head;
			
			// (2) 第一次 put, 直到 （casHead） 执行成功之前，
			// h == null 始终成立，所以进入下面的if语句。
            if (h == null || h.mode == mode) {  // empty or same-mode
                if (timed && nanos <= 0) {      // can't wait
                    if (h != null && h.isCancelled())
                        casHead(h, h.next);     // pop cancelled node
                    else
                        return null;
				// (3) 创建 s = snode(s, e, h, mode)
				// 使用 casHead 进入设置 head,开始自旋，直到
				// 下面的 if 成立，
                } else if (casHead(h, s = snode(s, e, h, mode))) {
					// 经过数次自旋尝试之后，当前线程进入 block 状态
					// 等待 一个线程的 take 操作。
                    SNode m = awaitFulfill(s, timed, nanos);
                    if (m == s) {               // wait was cancelled
                        clean(s);
                        return null;
                    }
                    if ((h = head) != null && h.next == s)
                        casHead(h, s.next);     // help s's fulfiller
                    return (mode == REQUEST) ? m.item : s.item;
                }
            } else if (!isFulfilling(h.mode)) { // try to fulfill
                if (h.isCancelled())            // already cancelled
                    casHead(h, h.next);         // pop and retry
                else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                    for (;;) { // loop until matched or waiters disappear
                        SNode m = s.next;       // m is s's match
                        if (m == null) {        // all waiters are gone
                            casHead(s, null);   // pop fulfill node
                            s = null;           // use new node next time
                            break;              // restart main loop
                        }
                        SNode mn = m.next;
                        if (m.tryMatch(s)) {
                            casHead(s, mn);     // pop both s and m
                            return (mode == REQUEST) ? m.item : s.item;
                        } else                  // lost match
                            s.casNext(m, mn);   // help unlink
                    }
                }
            } else {                            // help a fulfiller
                SNode m = h.next;               // m is h's match
                if (m == null)                  // waiter is gone
                    casHead(h, null);           // pop fulfilling node
                else {
                    SNode mn = m.next;
                    if (m.tryMatch(h))          // help match
                        casHead(h, mn);         // pop both h and m
                    else                        // lost match
                        h.casNext(m, mn);       // help unlink
                }
            }
        }


SNode awaitFulfill(SNode s, boolean timed, long nanos) {
            /*
             * When a node/thread is about to block, it sets its waiter
             * field and then rechecks state at least one more time
             * before actually parking, thus covering race vs
             * fulfiller noticing that waiter is non-null so should be
             * woken.
             *
             * When invoked by nodes that appear at the point of call
             * to be at the head of the stack, calls to park are
             * preceded by spins to avoid blocking when producers and
             * consumers are arriving very close in time.  This can
             * happen enough to bother only on multiprocessors.
             *
             * The order of checks for returning out of main loop
             * reflects fact that interrupts have precedence over
             * normal returns, which have precedence over
             * timeouts. (So, on timeout, one last check for match is
             * done before giving up.) Except that calls from untimed
             * SynchronousQueue.{poll/offer} don't check interrupts
             * and don't wait at all, so are trapped in transfer
             * method rather than calling awaitFulfill.
             */
			// lastTime = 0
            long lastTime = timed ? System.nanoTime() : 0;
            Thread w = Thread.currentThread();
			// h 上面 put 的 s
            SNode h = head;
			// 自旋次数： maxUntimedSpins
            int spins = (shouldSpin(s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
			// 开始自旋循环，次数就是上面的 spins.
			// 也就是执行spins下面的循环。
            for (;;) {
                if (w.isInterrupted())
                    s.tryCancel();
                SNode m = s.match;
				// 当下面的 s.tryCancel 成功之后， m != null 就会成功
                if (m != null)
                    return m;
				// 如果设置了超时，则在自旋过程中判断
				// 是否在自旋过程中就已经，超时了，如果超时，尝试
				// 调用 tryCancel 来取消，当前结点在 队列中的等待
                if (timed) {
                    long now = System.nanoTime();
                    nanos -= now - lastTime;
                    lastTime = now;
                    if (nanos <= 0) {
						// 这个方法会在外层的 for 循环中不断循环执行
						// 直到 将 s 的 match 字段设置成功，
						// 当 match 字段设置成功之后，继续 continue
						// 执行到上面的 if (m != null) 处
						// 这个判断成功了，所以上面的 if (m != null) 
						// 将成为 tryCancel 的出口，而不会因为 continue
						// 使当前线程陷入死循环
                        s.tryCancel();
                        continue;
                    }
                }

				// 减小自旋计数，直到
                if (spins > 0)
                    spins = shouldSpin(s) ? (spins-1) : 0;
				// 自旋次数已经足够了，设置 s 的 waiter 为当前线程
                else if (s.waiter == null)
                    s.waiter = w; // establish waiter so can park next iter
				// 没有设置超时，直接 park
                else if (!timed)
                    LockSupport.park(this);
				// 否则，依时间进行 park
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }
```

### take 过程

``` java
public E take() throws InterruptedException {
    Object e = transferer.transfer(null, false, 0);
    if (e != null)
        return (E)e;
    Thread.interrupted();
    throw new InterruptedException();
}
```

``` java
        Object transfer(Object e, boolean timed, long nanos) {

			// (1) s: mode = REQUEST, 
            SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA;

            for (;;) {
                SNode h = head;
                if (h == null || h.mode == mode) {  // empty or same-mode
                    if (timed && nanos <= 0) {      // can't wait
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     // pop cancelled node
                        else
                            return null;
                    } else if (casHead(h, s = snode(s, e, h, mode))) {
						// B. 在 A 处的 tryMatch 会将 fulfill 的结点唤醒
						// 下面的方法将会返回
                        SNode m = awaitFulfill(s, timed, nanos);
						// 如果是正常返回的话，m = s.next, 
						// 所以 应该有 m != s， 但是，如果 m==s, 则说明
						// awaitFulfill 这个方法返回的原因是 s 结点被cancell
						// 而不是被 fulfill , 所以 调用 clean 进行清理。
                        if (m == s) {               // wait was cancelled
                            clean(s);
                            return null;
                        }
                        if ((h = head) != null && h.next == s)
                            casHead(h, s.next);     // help s's fulfiller
						// 返回 数据
                        return (mode == REQUEST) ? m.item : s.item;
                    }
				// h.mode = DATA , isFulfilling 返回 false,
				// 所以进入下面的循环，进入下面循环之后，会将 head 指向正在
				// fulfill 的结点，而这个结点的 mode 将会是 FULFILLING|mode
				// 则此时下面的调用 isFulfilling 将返回 true, 
				// 所以这个if也不会进入。执行流程跳转到了最后的 else 语句
                } else if (!isFulfilling(h.mode)) { // try to fulfill
				// 在开始 take （取元素之前判断，head元素是否已经被 cancell）
				// 如果被 cancell, 则调整head,
                    if (h.isCancelled())            // already cancelled
						// 因为在 外层的循环中，所以casHead最终会执行成功，
                        casHead(h, h.next);         // pop and retry
					// 经过上面的head调整，将所有已经被 cancell的结点全部
					// 移除了，此时的 h 指向的一定不是被标记为移除的结点
					// FULFILLING|mode = 2|0 = 2
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            if (m == null) {        // all waiters are gone
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
							// 将当前结点与下一个结点，组合，
                            SNode mn = m.next;
							// 如果 match成功，其实就是一个 put 和 take 操作
							// s <==> take, m <==> put 操作
							// 其实 s 和 m,到底谁是 take ,put
							// 并不一定，可以由其 mode 字段来判定，
							// A. 这里 tryMatch 会将 block的线程unpark
							// m 结点对应的线程，将在上面的 awaitFulfill 方法
							// 处被唤醒
                            if (m.tryMatch(s)) {
							// 将 s 结点 和 m 结点，一起 pop 出 stack
                                casHead(s, mn);     // pop both s and m
                                return (mode == REQUEST) ? m.item : s.item;
                            } else                  // lost match
                                s.casNext(m, mn);   // help unlink
                        }
                    }
				// 这个 else中头部的结点正在被 fulfill ,
				// 当前线程，并不会，将当前 s 结点 入栈，而是
				// 帮助， h ,进行 match,可以看到下面的代码和
				// 和上面的 for 循环类似，最后，当前的 fulfill 一定会成功，
				// 成功之后，s 结点，才会开始被处理，处理流程类似
				// 其实就是上面的两个过程： 如果 mode 相同，则 block
				// 否则开始 fulfill.
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }

```

transfer的过程

队列中的block元素mode一定是全部都是相同的，只有栈顶的 node 有可能不同，如果栈顶的node的mode和栈内的node不同，则，此时，可以进行 fulfill 了，fulfill 成功之后，一对 put 和 take 操作将出栈。线程就从 transfer 方法退出了。

``` java
for (;;) { // loop until matched or waiters disappear

    SNode m = s.next;       // m is s's match

	// 表示已经循环到栈底了，所以
	// all waiters are gone pop fulfill node
	// 所以使用 casHead 将 stack 重置（head 设置成 null）
	// s = null 这条语句应该是 help for GC
    if (m == null) {        // all waiters are gone
        casHead(s, null);   // pop fulfill node
        s = null;           // use new node next time
        break;              // restart main loop
    }

	// 找到 m 的 next 结点
    SNode mn = m.next;
    if (m.tryMatch(s)) { // 尝试匹配 m 和 s
		// 如果成功，将 s 和 m 弹出栈，
        casHead(s, mn);     // pop both s and m
		// 如果 s.mode == REQUEST, 则 m 结点应该是 put 操作，
		// 所以 返回 m.item, 否则 s 是 put 结点，则返回 s.item
		// 由此可知，不管是 put 还是 take 的 transfer 操作都将
		// 返回 操作的 item
        return (mode == REQUEST) ? m.item : s.item;
    } else                  // lost match
        s.casNext(m, mn);   // help unlink
}


boolean tryMatch(SNode s) {
	// 所有并发调用 put 和 take 的线程，最终将按照调用的时间顺序
	// 进入 stack 中，所以，其实对于下面的方法，对于具体的某个结点
	// 是没有竞争的，所以下面的compareAndSwapObject方法调用一般都会直接成功
    if (match == null &&
        UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
        Thread w = waiter;
        if (w != null) {    // waiters need at most one unpark
            waiter = null;
			// 唤醒线程 w
            LockSupport.unpark(w);
        }
        return true;
    }
    return match == s;
}
```

使用场景参考：

[使用 SynchronousQueue 实现生产者/消费者模型](http://www.oschina.net/translate/implementing-producer-consumer-using-synchronousqueue)