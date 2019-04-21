---
title: network-Selector-PollSelectorImpl
date: 2016-12-15 10:52:27
---

## PollSelectorImpl

linux 2.6以下 及 unix 系统的 Selector 是由 PollSelectorImpl 类来实现的。

### 创建一个 PollSelectorImpl 的时候，创建了什么

创建一个 PollSelectorImpl 的时候，就是创建了一个 PollArrayWrapper 对象。用来存储 pollfd.

## register

### PollArray 的结构

``` c
// PollArray
struct pollfd {
	// 注册的Socket的文件描述符
    int   fd;         /* file descriptor */
    // 关注的事件， 注册的时候，默认为 0
    short events;     /* requested events */
    // 此时已经发生的事件
    short revents;    /* returned events */
};

+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
|fd  e|     |     |     |     |     |     |     |     |     |     |
+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+
```

PollSelectorImpl 类持有一个 PollArray 数组的实例。其默认大小是10. 这个数组的第一个元素是管道的句柄。

``` java
// 摘自： PollSelectorImpl 的构造函数
// IOUtil.makePipe其实现调用 pipe 创建管道
long pipeFds = IOUtil.makePipe(false);
// the read end of the pipe
fd0 = (int) (pipeFds >>> 32);
// the write end of the pipe
fd1 = (int) pipeFds;
// INIT_CAP = 10
pollWrapper = new PollArrayWrapper(INIT_CAP);
// 将 fd0 放置到 PollArray 的第一个元素的位置处
// fd0 表示 readPipeFD, 可以在 native 代码使用。
pollWrapper.initInterrupt(fd0, fd1);
```

### register 一个 SocketChannel 的时候，发生了什么

其实就是将 SocketChannel 保存到上面的结构中

## select

``` java
protected int doSelect(long timeout) throws IOException {
    if (channelArray == null)
        throw new ClosedSelectorException();
    processDeregisterQueue();
    
    try {
        begin();
        
        // 调用系统底层的 poll 实现
        pollWrapper.poll(totalChannels, 0, timeout);
    } finally {
        end();
    }
    
    processDeregisterQueue();
    
    // 返回结果，处理 pollWrapper
    int numKeysUpdated = updateSelectedKeys();
    if (pollWrapper.getReventOps(0) != 0) {
        // Clear the wakeup pipe
        pollWrapper.putReventOps(0, 0);
        synchronized (interruptLock) {
            IOUtil.drain(fd0);
            interruptTriggered = false;
        }
    }
    return numKeysUpdated;
}

// 1. Copy the information in the pollfd structs 
//    into the opss of the corresponding Channels.
// 2. Add the ready keys to the ready queue.
// 3. 统计channel的状态发生变化的个数
protected int updateSelectedKeys() {
    int numKeysUpdated = 0;
    // Skip zeroth entry; it is for interrupts only
    for (int i=channelOffset; i<totalChannels; i++) {
    	// 获得 pollfd.revents 对象的值。
        int rOps = pollWrapper.getReventOps(i);
        if (rOps != 0) {
            SelectionKeyImpl sk = channelArray[i];
            
            // 重置 pollWrapper
            pollWrapper.putReventOps(i, 0);
            
            if (selectedKeys.contains(sk)) {
            	// 依照 rOps 设置 sk 的 readyOps 字段
            	// translateAndSetReadyOps 函数的返回值表示
            	// readyOps 是否发生了变化。如果发生了变化，则返回 true
                if (sk.channel.translateAndSetReadyOps(rOps, sk)) {
                    numKeysUpdated++;
                }
            } else {
                sk.channel.translateAndSetReadyOps(rOps, sk);
                
                // 表示已经发生的事件中，确实有 客户代码关注的事件
                if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                	// 将 sk 添加到 selectedKeys 中
                    selectedKeys.add(sk);
                    numKeysUpdated++;
                }
            }
        }
    }
    
    // 返回已经更新的数量。
    return numKeysUpdated;
}
```

### 调用 Selector 的 select 方法的时候，发生了什么

Selector.select 方法最终调用系统的 poll 方法，进行等待状态。

当其返回时，主要做下面3件事：

1. Copy the information in the pollfd structs 
   into the opss of the corresponding Channels.

	设置好 SelectionKey 的状态变量，然后这几个查询方法就可以使用了：isAcceptable，isConnectable，isReadable，isWritable，readyOps
	
	通过这些方法，就可以对 SelectionKey 关联的 channel 进行读写操作了。

2. Add the ready keys to the ready queue.

	这个 ready set 由 Selector.selectedKeys 方法，可以获得
	
3. 统计channel的状态发生变化的个数

	这个个数由 select 方法返回。

## Selctor 的唤醒 wakeup

`wakeup` 将导致 Selector 方法的 select 方法直接返回。 

因为 select 操作会在其所关注的 fd 上发生感兴趣的事件的时候返回。所以可以在调用 select 的时候，向其注册一个pipe, 需要唤醒这个 selector 操作的时候，就可以向这个 pipe 中写入数据。此时pipe就会被唤醒。

所以在 pollWrapper 的第一个元素默认是 pipe.

``` java
public Selector wakeup() {
    synchronized (interruptLock) {
        if (!interruptTriggered) {
        	// pollWrapper.interrupt();
        	// 调用的作用就是向上面的 pipe 中写入一个字节的数据
        	// write(fd, fakebuf, 1)
        	// 从而导致 select 被唤醒。
            pollWrapper.interrupt();
            interruptTriggered = true;
        }
    }
    return this;
}
```

## Selctor 的关闭 close

下面分析  Selector 的关闭

``` java
// AbstractSelector
private AtomicBoolean selectorOpen = new AtomicBoolean(true);
public final void close() throws IOException {
    boolean open = selectorOpen.getAndSet(false);
    if (!open)
        return;
    implCloseSelector();
}

// AbstractPollSelectorImpl
protected void implClose() throws IOException {
    synchronized (closeLock) {
        if (closed)
            return;
        closed = true;
        
        // Deregister channels
        // 取消注册 channel
        for(int i=channelOffset; i<totalChannels; i++) {
            SelectionKeyImpl ski = channelArray[i];
            
            // 将 SelectionKey 中持有的 index 无效化
            assert(ski.getIndex() != -1);
            ski.setIndex(-1);
            
            // 一个 channel 可以注册到多个 Selector 上，
            // 所以 channel 内部维护了一个 SelectionKey 的集合
            // 这个方法调用目地就是将当前这个 Selector 和 Channel
            // 对应的 SelectionKey 从 channel 维护的信合中删除(remove)
            // 调用 AbstractSelectableChannel.removeKey 方法。
            // 然后，再调用 SelectionKey.invalidate 方法，将
            // SelectionKey 设置成无效的。此后 SelectionKey.isValide
            // 方法将返回 false.
            deregister(ski);
            
            
            SelectableChannel selch = channelArray[i].channel();
            if (!selch.isOpen() && !selch.isRegistered())
            	// 如果 channel 已经被关闭，并且已经被所有 SelectionKey
            	// 取消注册了。则调用 kill
            	// kill 的主要是：将 SocketChannelImpl 的状态字段（state）
            	// 改成 ST_KILLED 或者 ST_KILLPENDING
                ((SelChImpl)selch).kill();
        }
        
        // 释放 pipe 
        implCloseInterrupt();
        
        // 释放 pollWrapper 占用的 native 内存。
        pollWrapper.free();
        
        // 重置变量
        pollWrapper = null;
        selectedKeys = null;
        channelArray = null;
        totalChannels = 0;
    }
}

// PollSelectorImpl
protected void implCloseInterrupt() throws IOException {
    // prevent further wakeup
    synchronized (interruptLock) {
        interruptTriggered = true;
    }
    
    // 关闭 pipe
    FileDispatcherImpl.closeIntFD(fd0);
    FileDispatcherImpl.closeIntFD(fd1);
    fd0 = -1;
    fd1 = -1;
    
    // donothing
    pollWrapper.release(0);
}
````

注册在 Selector 上的 Channel 也可以主动关闭：

Channel 在关闭的时候，会在将其所注册的所有 Selector 上执行 cancel 操作。将 Selector 中与待关闭的 Channel 相关的数据进行清除。

``` java
protected final void implCloseChannel() throws IOException {
    implCloseSelectableChannel();
    synchronized (keyLock) {
        int count = (keys == null) ? 0 : keys.length;
        for (int i = 0; i < count; i++) {
            SelectionKey k = keys[i];
            if (k != null)
                k.cancel();
        }
    }
}

// 上面的 k.cancel 最终调用 AbstractSelector.cancel 方法
// 将 k 添加到 cancelledKeys 集合中。
void cancel(SelectionKey k) {
    synchronized (cancelledKeys) {
        cancelledKeys.add(k);
    }
}

// 那么这个添加到 cancelledKeys 集合在什么情况下会被使用到呢?
// SelectorImpl.processDeregisterQueue 方法会使用到上面的 cancelledKeys
// 那么 processDeregisterQueue 方法是在什么时候被调用的呢？
// 这个方法会在 select 方法调用的前后进行调用。
void processDeregisterQueue() throws IOException {
    // Precondition: Synchronized on this, keys, and selectedKeys
    Set cks = cancelledKeys();
    synchronized (cks) {
        if (!cks.isEmpty()) {
            Iterator i = cks.iterator();
            while (i.hasNext()) {
                SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                try {
                	// 在 implDereg 方法中会将相关的结构
                	// AbstractPollSelectorImpl.implDereg 
                	// 将相关的数据进行移除。
                    implDereg(ski);
                } catch (SocketException se) {
                    IOException ioe = new IOException(
                        "Error deregistering key");
                    ioe.initCause(se);
                    throw ioe;
                } finally {
                    i.remove();
                }
            }
        }
    }
}
```

## EPollSelectorImpl

在 linux 2.6 以后的平台上，JDK 提供的 Selector 就由 EPollSelectorImpl 来实现。EPollSelectorImpl 则使用系统底层的 epoll.

EPollSelectorImpl 的实现和 PollSelectorImpl 实现类似。其底层主要使用 epoll 机制来实现。

* epoll_create
* epoll_ctl
* epoll_wait

## 参考
1. [nio Selector 阻塞 唤醒 原理](http://zhhphappy.iteye.com/blog/2032893)