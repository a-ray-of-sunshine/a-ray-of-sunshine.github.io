---
title: ReentrantReadWriteLock
date: 2016-7-26 13:58:16
tags: [ReentrantReadWriteLock]
---

## 内部结构

``` java
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

protected ReadLock(ReentrantReadWriteLock lock) {
    sync = lock.sync;
}

protected WriteLock(ReentrantReadWriteLock lock) {
    sync = lock.sync;
}
```

继承关系：

``` java
public class ReentrantReadWriteLock
	implements ReadWriteLock, java.io.Serializable

abstract static class Sync extends AbstractQueuedSynchronizer 

public static class ReadLock implements Lock, java.io.Serializable

public static class WriteLock implements Lock, java.io.Serializable 
```

## java.util.concurrent.locks.ReentrantReadWriteLock.Sync

这个同步器的实现中 state 字段的定义：

* read count: state 的前两个字节（高字节）

* write count: state 的后两个字节（低字节）

Lock state is logically divided into two unsigned shorts:

* The lower one representing the exclusive (writer) lock hold count,

* The upper the shared (reader) hold count.

## ReadLock

* lock 过程

### java.util.concurrent.locks.ReentrantReadWriteLock.ReadLock.lock

``` java
public void lock() {
    sync.acquireShared(1);
}
```

###  java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireShared

``` java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

其中，tryAcquireShared 方法被 Sync类 override

### java.util.concurrent.locks.ReentrantReadWriteLock.Sync.tryAcquireShared

``` java
protected final int tryAcquireShared(int unused) {
        /*
         * Walkthrough:
         * 1. If write lock held by another thread, fail.
         * 2. Otherwise, this thread is eligible for
         *    lock wrt state, so ask if it should block
         *    because of queue policy. If not, try
         *    to grant by CASing state and updating count.
         *    Note that step does not check for reentrant
         *    acquires, which is postponed to full version
         *    to avoid having to check hold count in
         *    the more typical non-reentrant case.
         * 3. If step 2 fails either because thread
         *    apparently not eligible or CAS fails or count
         *    saturated, chain to version with full retry loop.
         */
        Thread current = Thread.currentThread();
        int c = getState();
        
        // 如果写锁被占用，并且当前申请锁的线程
        // 不是写锁的占用者
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
            
        // 代码能够执行到这里表明：
        // a. 写锁未被占用，也即被lock保护的共享资源此时没有线程在进行写操作
        // b. 写锁被占用了，但是占用写锁的，就是当前申请读锁的线程。
        
        // 获得读锁的计数
        int r = sharedCount(c);
        
        // readerShouldBlock 方法最终调用 AbstractQueuedSynchronizer.apparentlyFirstQueuedIsExclusive
        //这个方法用来判断在sync queue的头部是否有线程以独占模式在等待锁。如果是则返回 true.
        if (!readerShouldBlock() &&
            r < MAX_COUNT && // r 值要合法
            compareAndSetState(c, c +  SHARED_UNIT)) {// 对读锁+1,c+SHARED_UNIT==readlockcount + 1
            
            // 维护状态信息
            
            // 维护 firstReader 和 firstReaderHoldCount 字段 
            if (r == 0) {
            	// 记录读锁值为0时的，获得
            	// 读锁的线程。
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
            //维护 cachedHoldCounter，保证它指向
            // 最近一次成功获取 readLock 的线程
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                // rh 引用始终指向当前线程
                // 所以 rh.count++ 就是用来维护当前线程的重入次数。
                rh.count++;
            }
            
            // 返回值 1 > 0, 表明这个线程获取读锁成功
            return 1; 
        }
        
        // 如果上面的尝试都没有申请成功，则调用
        // fullTryAcquireShared 这方法会进行
        // CAS 自旋，直到条件被满足。
        return fullTryAcquireShared(current);
}
```

由上面的代码可知：获取读锁的条件就是

!readerShouldBlock() &&
r < MAX_COUNT &&
compareAndSetState(c, c + SHARED_UNIT)

其实就是 readerShouldBlock 返回 false,读锁就可以被获取到。

### ReentrantReadWriteLock.Sync.fullTryAcquireShared

``` java
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
            	// 0. 有线程在写，但是该线程不是当前申请读锁的线程
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) {
        	// 1. 没有线程在进行写操作，但是线程仍然需要被 block，可能是原因导致的。 
        	
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
            // 如果这个条件成立，则直接证明，当前线程就是在重入申请锁
            // 所以此时必然有 firstReaderHoldCount > 0
            // 对于重入的线程，不应该 block, 所以 do nothing!!!
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId()) {
                    // 上面的条件成立，表示：
                    // 申请锁的线程没有重入，
                    // 但是因为线程即将进行block状态
                    // 所以 if (rh.count == 0)
                    // 则释放线程占有的内存资源，因为线程要进入
                    // sync queue 了，有可能 wait 比较长的时间，
                    // 所以释放内存资源。
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                
                // !!! rh 此时指向当前线程的 HoldCounter !!!
                // 表示线程是第一次申请，读锁
                // 所以直接返回 -1, 如果线程是重入的
                // 则即使 readerShouldBlock 要求
                // 阻塞，也不用理睬，因为rh.count
                // 表示 当前线程是已经获得过锁的，
                // 而且读锁是支持重入的，所以直接
                // 就应该让当前再次获得锁。
                if (rh.count == 0)
                    return -1;
                    
                // 在写锁中使用读锁，就是在这种条件下，
                // 避免 block,而成功的。
            }
        }
        
        // 代码执行到这里其实当前线程已经
        // 可以成功获得锁了，下面使用 CAS 自旋来
        // 设置状态，成功之后 return 1,
        // 锁获取成功。
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

readerShouldBlock，在什么情况下会出现，没有写线程的情况下，还需要读线程block呢？估计和 write lock 的 condition 有关。

**fullTryAcquireShared 方法在成功获得锁时，设置状态们时并没有对重入的情况区别对待，而是直接，compareAndSetState(c, c + SHARED_UNIT)，这也说明一个问题，对于通过重入获得的读锁的线程，这个线程也将被认为另是一个获得读锁的线程: read lock count + 1 **

**那么，能不能区分呢，可以：**

``` java
HoldCounter rh = readHolds.get();
if(rh.count > 1){
	// 表示是重入的
}else{
	// 非重入
}
```

之所以不区分，原因估计是出于性能考虑。

**所以读锁有 65535 并不表示可以有 65535 线程可以获得锁，如果有一个线程重入一次，则这次重入也将导致 读锁 减少，此时只有 65535 - 1 个线程可以获得锁，而不是 65535 个。**


* release 过程

读锁的释放

### ReentrantReadWriteLock.Sync.tryReleaseShared


这个方法会被多个 读线程 并发调用

``` java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
        	// 资源清理
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != current.getId())
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
        	// 资源清理
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        
        // 相当于 (read count) - 1
        // 对于读线程来说 nextc 表示，sync queue 中
        // 等待 读锁 的线程个数。
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            // !!!当读写锁都被释放的时候，可以通知在 sync queue 中!!!
            // !!!等待的写请求的线程!!!
            return nextc == 0;
    }
}
```

这个释放 read lock 的，貌似，read lock 如果重入了，只能调用一次 release,否则会出问题。

**这里也是直接释放锁，而没有区分是否是重入的线程，因为获得锁时也没有区分，所以这么做是合理的**

### 对 Condition 的支持

``` java
public Condition newCondition() {
    throw new UnsupportedOperationException();
}
```

**读锁不支持Condition**

## WriteLock

* lock 过程

### ReentrantReadWriteLock.Sync.tryAcquire

``` java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // !!! if c != 0 and w == 0 表示正在有线程进行读操作 !!!
        // 并且，当前的申请并不是当前线程进行重入申请，
        // !!! 那么，直接返回 false, 表示在 读锁中不可以使用写锁 !!!
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
            
        // 执行到这里时，表示申请锁的线程和
        // 当前锁的占有者是同一个线程。所以这里
        // 可以直接获取到锁。
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    
    // NonfairSync 实现的 writerShouldBlock 总是返回 false
    // 所以如果 compareAndSetState 执行成功
    // 则锁获取成功，否则锁获取失败，直接返回 false.
    // c + acquires 相当于 writelockcount + 1
    // 对 写锁 +1
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

* unlock 过程

### ReentrantReadWriteLock.Sync.tryRelease

这个方法是写线程独占调用的。所以可以直接使用 setState
来设置 state 字段。

``` java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    // exclusiveCount(nextc) 提取出写锁的count
    // 如果写锁 == 0, 表明锁可以释放了。
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

### 对 Condition 的支持

* ReentrantReadWriteLock.WriteLock.newCondition()
``` java
public Condition newCondition() {
    return sync.newCondition();
}
```

* ReentrantReadWriteLock.Sync.newCondition()
``` java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

**写锁支持Condition**

## WriteLock & ReadLock

### state
 
这个字段的，高字表示 read lock count，低字表示 write lock count。

但其含义却不相同，read lock count 表示有多少个线程，拥有了锁，而write lock count表示当前拥有锁的线程的请求锁的重入次数。

所以，write lock 可以支持单个线程 65535 次重入，而 read lock 则支持 65535 个线程获得锁。当超过这个数值时，lock 方法就会抛异常。

``` java
throw new Error("Maximum lock count exceeded");
```

This lock supports a maximum of 65535 recursive write locks and 65535 read locks. Attempts to exceed these limits result in Error throws from locking methods.

## 重入的支持

读线程和写线程都支持重入。此外，写线程可以请求读锁，但是反过来却不行，所以如果一个 reader 试图到请求读锁将不会成功。

### read 线程 如何维护其重入状态

state 字段中的 read lock count 表示的是获得 read lock的线程数量。那对于一个 read 线程其重入的数值如何记录呢？同时对于独占的写线程来说，记录其重入的次数，其意义在于当重入次数等于0的时候，释放其对于锁的独占，也就是：

ReentrantReadWriteLock.Sync.tryRelease:

``` java
boolean free = exclusiveCount(nextc) == 0;
if (free)
    setExclusiveOwnerThread(null);
```

但是对于，read锁来说，因为其并不会独占线程，也就是说不会设置 setExclusiveOwnerThread 成当前线程，因为多个 read 线程是并发执行的，也就没有所谓当前线程了。那么对于 read 线程是否还需要维护其重入的次数呢？ ReadLock 对重入状态的维护在 tryAcquireShared 和 tryReleaseShared 中。

其时维护重入次数是有意义的，保证 read lock 的 relase 方法能够被正确地调用：

``` java
int count = rh.count;
if (count <= 1) {
    readHolds.remove();
    if (count <= 0)
        throw unmatchedUnlockException();
}
```

### 写锁中使用读锁

可以使用，不会造成死锁。**符合读写及并发的逻辑，写的时候去读，不会造成并发问题，而读的时候去写，其它正在读的线程则有可能读到脏数据，从而引发同步问题。**


### 读锁中使用写锁

不可以使用，如果这样使用了，则当前线程将被放到 sync queue 中。从而进入 park（block） 状态，死锁就发生了。当前线程持有读锁，但是又被 park 了。而只有所有读写锁free的时候，队列中的wait的 写线程才可以被唤醒，循环了，死锁发生了。