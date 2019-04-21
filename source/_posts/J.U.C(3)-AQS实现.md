---
title: J.U.C(3)-AQS实现
date: 2016-7-19 13:53:29
tags: [J.U.C, AQS]
---

AbstractQueuedSynchronizer是实现锁的关键，Doug Lea关于实现这个类的论文： [The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)

论文中的第3节描述了，AbstractQueuedSynchronizer 的设计和实现过程。

## 设计

对于一个同步器（synchronzier），最基本的操作有两个：acquired(获取锁) 和 release（释放锁）

* acquired

	```
	while (synchronization state does not allow acquire) {
		enqueue current thread if not already queued;
		possibly block current thread;
	}
	dequeue current thread if it was queued;
	```

* release

	```
	update synchronization state;
	if (state may permit a blocked thread to acquire)
	unblock one or more queued threads;
	```

这实现上面的功能，则必须有三个基本的组件（component）:

* Atomically　managing synchronization state

* Blocking and unblocking threads

* Maintaining queues

## 实现

### Synchronization State

``` java
/**
 * The synchronization state.
 */
private volatile int state;

protected final int getState() {
    return state;
}

protected final void setState(int newState) {
        state = newState;
}

protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

state 字段是 volatile 的，保证的其内存可见性，使用 compareAndSetState 方法（CAS）来保证原子性，所以使用 state 是线程安全的。

对于改变 state 字段其实有两个方法：

* setState

* compareAndSetState

其实，提供两个更新 state 的方法是，基于 同步 和 性能 的考虑：当在确定当前线程是锁的拥有者时，直接使用 setState 方法可以提高性能（例如：tryRelease方法中，可以直接使用setState，因为此时，当前线程一定是锁的owner。还有就是请求锁重入的时候，也可以直接使用），当无法确定当前线程是否拥有锁时，使用 compareAndSetState，来保证 state 是同步的。

基于 AbstractQueuedSynchronizer 类的 同步器 的实现，必须 override 其 tryAcquire 和 tryRelease 方法。这两个方法都有一个 int 类型的参数，可以根据这个参数和当前 state 字段的值 来确定 同步器 对锁的获取和释放的行为，从而使用不同功能的同步器。

### Blocking

在 jdk1.5 之前惟一能让一个线程 block 和 unblock 的方法就是：Thread.suspend 和 Thread.resume，但是这两个方法有可能导致线程死锁，所以已经不能使用了。

jdk1.5 中提供了一个支持线程挂起的类：
java.util.concurrent.locks.LockSupport

LockSupport会将使用过它（LockSupport）的线程都会关联一个 permit. 如果当前线程的 permit 是 available 的，则 park 方法的调用将立即返回，	consuming it in the process; 否则线程会 block.

如果线程的 permit 不是 avilable 的，则 unpark 的调用则会使用 permit 变成  avilable 的。

* park

	block 当前线程
	
	park有以下几种：
	
	* park(Object blocker)
	
	* parkNanos(Object blocker, long nanos) 
	
	* parkUntil(Object blocker, long deadline)
	
	* park() 
	
	* parkNanos(long nanos) 
	
	* parkUntil(long deadline) 
	
	前三个 park 与 后三个惟一不同的就是：多一个 blocker 参数，这个参数的作用是：This object is recorded while the thread is blocked to permit monitoring and diagnostic tools to identify the reasons that threads are blocked. 这个参数用来给调试工具使用。文档中推荐使用前三个API。
	
	调用这个方法
	
* unpark(Thread thread)

	unblock 指定的线程

	使用指定的线程的 permit 变成 available 的。如果线程因为 park 而 block,则此时会 unblock.此外，由于 该线程的 permis is available, 所以 在线程上再次调用 park 将立即返回，而不是 block.

The park method also supports optional relative and absolute timeouts, and is integrated with JVM Thread.interrupt support — interrupting a thread unparks it.

park方法是可以被中断的。中断之后，线程会 unpark.

### Queues

同步框架的核心就是 阻塞线程队列 的维护。这个队列是 FIFO的，同步框架不支持基于优先级的同步。

实现使用 CLH queue

### Conditions Queues


## ReentrantLock 实现

### lock

* java.util.concurrent.locks.ReentrantLock.NonfairSync.lock

	``` java
	final void lock() {
	    if (compareAndSetState(0, 1))
	        setExclusiveOwnerThread(Thread.currentThread());
	    else
	        acquire(1);
	}
	```
	
	首先，尝试立即获取锁，如果，注意参数 compareAndSetState(0, 1)， 表示这个调用返回true的，惟一条件就是：这个锁lock的 state 字段是0，才会成功，否则，调用 acquire 获取锁。

* java.util.concurrent.locks.AbstractQueuedSynchronizer.acquire
``` java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
这个方法接受一个int参数，这个参数对于AbstractQueuedSynchronizer来说并没有具体的含义，此参数会直接传递给，具体的 Synchronizer 实现类的 override 的 tryAcquire 方法。所以，这个参数其实就是由具体的实现类决定其含义的。

文档描述：以独占模式请求锁，忽略中断。至少调用一次 tryAcquire，如果成功则这个方法直接返回。否则，当前线程（调用acquire方法的线程）将会进入请求队列，并可能重复的 blocking and unblocking, 直到 tryAcquire 调用成功。

**acquireQueued(addWaiter(Node.EXCLUSIVE), arg)**

**addWaiter 是 lock-free 的方法，所以对于 上面的调用，acuireQueued中 addWaiter(Node.EXCLUSIVE) 中返回的node其实有可能已经不是处于 tail 了（在 addWaiter 方法中始终将当前的node 放置到queue的 tail. 由于 addWaiter的并发，），所以当 acquireQueued 执行的时候传递进去的 node ，其实已经不在 tail 位置了。**

* tryAcquire

	**AbstractQueuedSynchronizer类定义了这个方法，但是没有实现，这个方法就是实现各个功能的锁的核心，这个方法的作用就是，判定当前线程是否应该获得锁 ，如果可以返回 true, 否则返回 false。子类通过不同的逻辑判定线程是否可以获取锁，就可以实现各种功能的锁。**
	
	
	ReentrantLock 的内部类Sync实现的  tryAcquire 直接调用了下面的方法

	java.util.concurrent.locks.ReentrantLock.Sync.nonfairTryAcquire

	```java
	final boolean nonfairTryAcquire(int acquires) {
	    final Thread current = Thread.currentThread();
	    int c = getState();
	    // 尝试一次，是否可以直接获取
	    if (c == 0) {
	        if (compareAndSetState(0, acquires)) {
	            setExclusiveOwnerThread(current);
	            return true;
	        }
	    }
	    // 判断是否是同一个线程再次获取锁，支持可重入
	    else if (current == getExclusiveOwnerThread()) {
	        int nextc = c + acquires;
	        if (nextc < 0) // overflow
	            throw new Error("Maximum lock count exceeded");
	        // 由此可知在 ReentrantLock 的实现中，
	        // state 字段还表示，当前持有锁的线程的重入次数
	        setState(nextc);
	        return true;
	    }
	    
	    // 上面两种尝试都未成功表示，线程获取锁失败
	    return false;
	}
	```

由上面的代码可知，ReentrantLock 支持可重入，所以对于 AbstractQueuedSynchronizer 所维护的 queue 中的 Node 和 所以 请求锁的线程是1：1（一一对应）。每一个Thread在queue只有惟一一个node和其对应。

由 acquire 的实现可知：

``` java
if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
```

如果 tryAcquire 方法获取锁成功，则返回 true, 上面的表达将不再执行，acquire方法就返回。如果锁获取失败，则将使用 addWaiter 方法将当前请求线程以 exclusive 模式的 Node 结点添加到 请求队列的尾部。然后，将返回代表当前线程的Node 对象，作为参数传递给 acquireQueued 方法，而这个方法将会使用当前线程在queue中等待获取锁。具体分析，如下：

* java.util.concurrent.locks.AbstractQueuedSynchronizer.addWaiter

	将当前线程添加到请求队列的 tail 处。整个添加过程使用 CAS 来确保线程安全。添加成功之后将这个 node 返回。

* java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireQueued

这个方法有两个参数：

1. node: 这个 node 始终代表当前线程。node.thread == Thread.currentThread()

2. arg 传递给 tryAcquire

``` java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
        	// 获得 当前线程 node 的上一个 node.
            final Node p = node.predecessor();
            
            // 因为 head 指向的 node 是 dummy 的，所以如果 p == head, 则说明当前 queue 中 node结点已经到头部（但是并不能证明只有当前线程在请求lock,与此同时可能还有其它线程刚好请求锁，所以这个条件判断成功之后，并不会直接将锁给当前线程，而是使用tryAcquire方法来获取锁）, 然后调用 tryAcquire，如果成功进入if代码块中
            if (p == head && tryAcquire(arg)) {
                // 把当前node从队列中移除
                setHead(node);
                p.next = null; // help GC
                
                // 因为当前结点（在队列中的最后一个结点）已经获取了锁，所以不需要处理了（调用cancelAcquire方法）
                failed = false;
                return interrupted;
            }
            
            
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这个方法其实就是 CLH 锁(是一个 FIFO 的 queue)的一个实现：

0. 获得当前 node 的前驱结点 p.

	 final Node p = node.predecessor();

1. 判断 p == head, 如果相同，则表示，当前结点已经到了这个queue的头部，按照 CLH queue 的算法，这时候，这个线程应该就是锁的拥有者了，但是，存在一种可能就是，当前正有一个线程也在请求锁，但这个线程还未入queue（例如：一个线程正在执行，acquire 方法中的 tryAcquire去获取锁，而这个线程尚未入 queue）,所以不能直接将锁交给 node, 而是调用 tryAcquire 去获取。

	```java
	if (p == head && tryAcquire(arg)) {
		// 获取锁成功，当前 node 出队列，
	    setHead(node);
	    
	    // 将 p 对 node 的引用 next 断开，for help GC
	    // 虽然，上面通过 setHead 将 node 设置为新的 head
	    // 但 这里 p 仍旧指向，以前的 head(也就是被移出队列的head)
	    // 把它的 next 设置成 null, 减少对 当前 head (node) 的引用
	    p.next = null; // help GC
	    
	    // 获取锁，成功了，没失败，所以 failed = false
	    failed = false;
	    
	    // interrupted 表示 阻塞返回的原因，也就是
	    // parkAndCheckInterrupt 调用返回的原因，当这个方法返回后到
	    // acquire中，acquire会根据这个状态，如果 ture,则中断当前线程，
	    // 否则，直接成功返回，由此可知锁的获取是可以被中断的。
	    return interrupted;
	}
	```

2. 根据前驱结点 p 的 waitStatus 状态位来判定是否park

	shouldParkAfterFailedAcquire(p, node)

``` java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park. 如果前驱结点，已经被设置成 Node.SIGNAL， 则直接返回 true 表示 node 可以安全park了。
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry. 表示 ws == Node.CANCELLED node的前驱结点自身被取消了。所以下面通过循环来进行前向遍历，直到找到未被取消的结点。然后把这个结点作为 node 的新的前驱。
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
         	
         	waitStatus <= 0 , 则 waitStatus 是 0 或者是 PROPAGATE(-3),
			CONDITION(-2)的结点在 同步队列中是不存在的。将 pred 的 ws 设置成 Node.SIGNAL， 调用者下次调用的时候，直接返回 true. node就会被 block.
         
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

由于 shouldParkAfterFailedAcquire 是在 acquireQueued 的循环内部调用的，所以 shouldParkAfterFailedAcquire 这执行逻辑应该是这样的。

* 第一次循环，调用前继可能被取消了也就是 ws > 0 的情况，这时候，从前继向前遍历，将所以被取消的前继从队列中移除。**这时候，其实 sync queue 发生了结构性修改，所有的 Node.CANCELLED 结点被移出**

* 第二次循环，基本可以确定 ws <= 0了，其实在并发的情况下有可能，上次调整的结点也被取消了。所以，第二次循环的时候，有可能还会出现 ws > 0 的情况，那就继续循环呗。

3. 如果上面返回 true 则表示，当前线程应该 park，调用parkAndCheckInterrupt方法，使得当前线程 block.

	if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())

	到此为止，**所有在 queue 中，但没在头部的 node**（线程者是并发执行的，而且Lock的实现是 lock-free 的，所以，所有的线程都开始进行自旋等待了） 都将进入 park 状态，直到头部的 node 的释放锁。

获取锁的流程：

1. 尝试锁是否可以被获取，如果可以直接获取返回，否则，进入步骤2.

2. 再次尝试是否可以直接被获取，可以则获取成功。

3. 尝试是否，是同一个线程再次获取，如果是，则成功。

4. 将当前线程添加到请求队列中。

5. 依据 前一个结点 的状态位来，判定是否进行 park

6. 直到 unpark, 然后返回

### unlock

* 
java.util.concurrent.locks.AbstractQueuedSynchronizer.release
``` java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

由于，lock方法返回的时候，已经将当前线程，设置为 head, 所以，如果一个线程将持有的锁释放了，就应该通知 queue 中的 head 元素，让其后继结点，参与到锁的竞争中来。由于 后继通常是在 blocking 状态，所以要 调用 unparkSuccessor 对后继结点进行 唤醒 操作。

*  java.util.concurrent.locks.ReentrantLock.Sync.tryRelease
``` java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
ReentrantLock 由于 支持可重入，所以 state 字段，也表示，锁的持有线程的对锁的重入次数，如上面所示，直到，重入为0，时才释放锁，否则unlock，只将是调整重入次数（state）,并不释放锁。返回值代表，锁是否被成功释放。

* unparkSuccessor
``` java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread. 如果有后继在当前 node 上等待，则这个状态应该是 Node.SIGNAL ，如果当前 node 没有后继，则这个 状态字段应该是 0. 但是对于这个操作是否成功已经不重要了，因为，随着下面的 unpark 的调用，这个 node 将被 永远地移除，因为它的作用已经完成了：0. 成功获取锁（node 由 head.next 变成 head） 1.下面 unpark 调用 将通知在其上 wait 的 node ( node将被其next取代，node被移除 ), 这两个功能完成，这个 node 就 head 处移除。
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor. 如果后继结点是null或者后继被取消了，则从同步队列的 tail 处开始 前向遍历 直到 node 处，找到 node 的真正的后继结点。
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    
    // 最终经过上面的查找，如果后继不为空，则通知后继可以unpark了。后继 unpark 之后，再一次参与到锁的竞争中去。
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

* cancelAcquire
``` java
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

	// help for GC
    node.thread = null;

    // Skip cancelled predecessors
    // 跳过已经被取消的前继，直到 pred.waitStatus <= 0
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // 上面的代码执行完成之后，node.prev 被直接改成了 pred,
    // 而这个 pred 的 next 元素该如果处理。

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    // 如果按照上面的循环处理的话，其实 predNex这个
    // 结点本身是被取消（CANCELLED）的，但是其前继却
    // 是 <= 0 的，所以这个元素，需要特殊处理。
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference（干扰） from other threads.
    // 将 node 结点设置成 取消的。
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
    	// 如果 node 是 最后一个结点，则直接将 pred
    	// 设置成 tail,然后，将其 tail 的 next 域
    	// 设置成 null
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        // 由于 node 不是 tail,所以其后面还有结点
        // 需要被通知，所以 1. 设置 pred 的域为
        // Node.SIGNAL, 
        // 2. 将 node 的 next 接到 pred 的 next 域
        // 中
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
        	// 唤醒后继
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```


## Node

Node 中的 waitStatus 字段，5个字段。

* SIGNAL（-1）: 表示线程的successor线程需要被unparking 

	表示这个node的后继者（successor）是(将会是)blocked的（通过使用 park 方法），所以当前node在release或cancel锁的时候，必须unpark它的successor。

	To avoid races, acquire methods must first indicate they need a signal, then retry the atomic acquire, and then, on failure, block.

* CANCELLED（1）: 表示线程被取消

	因为超时或者中断，这个node被取消了。

* CONDITION（-2）： 表示线程在条件上等待

	这个node当前在一个condition 队列中。它不会被当作同步队列结点（sync queue node）,直到被 transferred（转移，调动），与此同时这个结点的 waitStatus 会被设置成 0.

* PROPAGATE（传播，-3）： 表示下一个 acquireShared 应该被无条件的传播。

Status field, taking on only the values: 

SIGNAL: The successor of this node is (or will soon be) blocked (via park), so the current node must unpark its successor when it releases or cancels. **To avoid races, acquire methods must first indicate they need a signal, then retry the atomic acquire, and then, on failure, block.** 

SIGNAL: 当前结点的后继结点正在（或将会）被阻塞（通过park）,所以当 当前结点 在 release 或者 cancel 锁时，必须 unpark 其后继结点。

CANCELLED: This node is cancelled due to timeout or interrupt. Nodes never leave this state. In particular, a thread with cancelled node never again blocks. 

CANCELLED: 这个结点因为超时或者中断被取消了。结点一般不会保留这个状态状态。事实上，被取消的线程将不会再被 block. 

CONDITION: This node is currently on a condition queue. It will not be used as a sync queue node until transferred, at which time the status will be set to 0. (Use of this value here has nothing to do with the other uses of the field, but simplifies mechanics.) 

CONDITION: 这个结点当前处于条件队列中。在被 transferred 之前，它不会被当前 同步队列 中的结点。transferred 之后，这个状态位会被设置成 0

PROPAGATE: A releaseShared should be propagated to other nodes. This is set (for head node only) in doReleaseShared to ensure propagation continues, even if other operations have since intervened. 

PROPAGATE: 

0: None of the above 

The values are arranged numerically to simplify use. Non-negative values mean that a node doesn't need to signal. So, most code doesn't need to check for particular values, just for sign. 

waitStatus的取值都是数字类型是为了简化操作。当一个结点的该值为 waitStatus > 0 时，表示这个结点不需要被 signal. 所以在代码中一般不需要检查某个特殊的值，只需要判断其符号即可。

The field is initialized to 0 for normal sync nodes, and CONDITION for condition nodes. It is modified using CAS (or when possible, unconditional volatile writes).

对于普通的 sync node 这个字段的默认初始值是 0.
对于 condition node 这个字段的默认初始值是 CONDITION. 这个字段在被修改时使用CAS，保证原子性。

## java.util.concurrent.locks.AbstractQueuedSynchronizer.ConditionObject

``` java
public class ConditionObject implements Condition, java.io.Serializable 
```

这个类实现了 Condition 接口

### java.util.concurrent.locks.Condition

Condition 接口将原来的 Object 的 monitor 方法（wait, notify and notifyAll）分离了出来，使得其和 Lock 对象独立。Lock 接口替代了 synchronized 方法和语句块，而 Condition 接口替代了 Object 对象的 monitor methods。

Conditions（也被称为 condition queue 或者 condition variables）提供了一种挂起线程的操作，直到其它的线程通知它可以继续了。

The key property that waiting for a condition provides is that it atomically releases the associated lock and suspends the current thread, just like Object.wait. 

当在一个 condition 等待（wait）的时候，它其自动释放相关联的锁，然后挂起线程。就像 Object.wait 的行为那样。

condition的实例是内置于 lock中的。所以要获取一个condition,就需要调用 Lock 的 newCondition 方法。

#### 对象结构
**ConditionObject在使用时是天然线程安全的，因为它在lock的包围之中**

关于 ConditionObject 的结构：ConditionObject 一个的使用在锁内部使用的，当一个对象调用 await 进行等待时，当前线程会放弃锁，而其它获得锁的线程，也有可以调用 锁的 ConditionObject 的方法，这样就会有多个线程在一个 条件对象 上等待，那这多个 等待对象 如果安置呢？ Node 对象有一个 

	  Node nextWaiter;

这个字段用来，构成一个单向链表，按时间顺序保存所有在这个条件对象上等待的线程。

ConditionObject对象中的 

* firstWaiter 指向 First node of condition queue.

* lastWaiter 指向 Last node of condition queue. 

#### await

使用当前线程处于，等待状态。

``` java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
        
    // 将当前线程放置到 condition queue 的lastWaiter
    Node node = addConditionWaiter();
    
    // 将当前线程拥有的锁释放
    // 返回的线程当前的 state 状态的字段
    // 这个字段必须保存起来，将来等 await
    // 醒来之后，必须恢复这个state字段
    // 这线程的同步状态就正常了。
    // 这里会把，拥有的锁释放！！！
    // 当前线程在 sync 队列中的 node 被移除
    // 相当于将当前的node 从 sync queue转移到 condition queue，当某个线程调用 signal 时，则会
    // 把 node 从 condition queue 再转移到
    // sync queue 中。
    int savedState = fullyRelease(node);
    
    int interruptMode = 0;
    // isOnSyncQueue 方法 返回 false
    // 所以进入下面的循环, 
    // 当其它线程调用 signal 方法,方法
    // park 方法返回，node的状态也应该
    // 发生变化了，然后 isOnSyncQueue 返回 ture
    // 这个 while 循环结束，
    // 退出，等待了。
    while (!isOnSyncQueue(node)) {
    	// block 当前线程
        LockSupport.park(this);
        // 如果 park 是正常被某个线程signal而
        // 唤醒的话，node 结点应该已经从 condition
       // queue 中转换到 sync queue 中，
       // 所以在上面，要进行判断 node 的状态判断
        
        // 检测在 wait 过程中当前线程是否被
        // 中断，如果被中断，则 break;
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    
    // 前面，释放了锁，而对于 await 方法来说
    // 醒来之后必须，重新获取锁才可以，继续执行
    // 因为，await 方法调用是在，lock 保护的临界区
    // 所以，需要调用 acquireQueued ，方法将当前
    // 线程重新进入 sync queue 进行锁竞争，
    // savedState 非常重要，使用当前线程恢复到原来
    // (await方法调用之前)的状态，后续可以使用 
    // unlock 解锁线程。不然，后面代码的 unlock 
    // 调用会出问题
    // acquireQueued 相当于 lock,此时线程，参与
    // sync queue 中线程的锁竞争。
    // 在 condition queue 中，wait 的线程，不会
    // 更改全局锁的 state ，wait 之后将其恢复。
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    
    // 如果在这个 condition 上，还有其它 waiter
    // 则将其中已经 cancelled 的 node,从 condition
    // queue 中移除。
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

* addConditionWaiter

Node node = new Node(Thread.currentThread(), Node.CONDITION);

其中，新添加的 node 的 waitStates 是 Node.CONDITION

* fullyRelease

核心代码如下：
``` java
int savedState = getState();
release(savedState)
return savedState
```
release通过传递参数 savedState，使得当前线程拥有的锁释放。注意，如果是重入的锁，这个调用也会将其release所以这个方法名为： fullyRelase.

* isOnSyncQueue

判断 node 是否已经被转移到 sync queue 中。

``` java
final boolean isOnSyncQueue(Node node) {

	// 0. 未被被 signal 之前，node.waitStatus 始终是
	// Node.CONDITION
	// 1. 如果 node 被 入队列了，则其 prev 字段
	// 一定不是空的
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    
	// 有后继也表明，一定已经在队列中了。    
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
     // node.prev 是 非空的时候，并不能确保，node
     // 已经在 queue 中了，原因如上。所以使用
     // 调用 findNodeFromTail 从尾到头，查找 node
     // 是否存在，如果存在返回true,否则返回false.
    return findNodeFromTail(node);
}
```

#### signal

``` java
public final void signal() {

	// 如果当前线程不是锁的拥有者，返回 false
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
   // 通知在 conditon queue 中的第一个
   // wait 的线程可以 unpark 了，然后可以可以参与锁
   // 的竞争。
        doSignal(first);
}
```

#### doSignal

``` java
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

从 first 结点开始后向遍历，直到，某一个结点上 transferForSignal 方法调用成功。最终，firstWaiter字段指向前面被通知的结点的 nextWaiter.

#### transferForSignal

**signal要做的惟一的事就是将node从 condition queue 中 转移到 sync queue.**

signal的逻辑应该是：

1. 将当前 在 condition queue 中的 node 转移到 sync
queue 中

2. 设置 node 的前继的 waitStates，为 Node.SIGNAL 表示，其后继结点 需要在其上自旋等待。其实就是当前node需要重新请求锁了。

3.  如果 node 的前继，被取消了（ws>0）或者 2 中的设置失败，直接强制 unpark 线程。

``` java
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     * 如果不能改变当前 node 的 status, 直接返回 false,惟一的原因应该是 这个结点被 取消了
     将 node 的 waitStatus 设置成0
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
     // 如果代码，执行到这里则，则 node 一定是
     // 没有被取消
    // 将 node 加入到 sync queue 队列，并返回 node 的前继
    Node p = enq(node);
    // 返回的p是node的前继
    int ws = p.waitStatus;
    
    // 如果 node 的前继，被取消了（ws>0）
    // 
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
