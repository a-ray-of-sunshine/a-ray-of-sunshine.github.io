---
title: network-AsynchronousSocketChannel
date: 2016-12-16 18:05:54
---

AsynchronousServerSocketChannel 和 AsynchronousSocketChannel 的实现借助于 AsynchronousChannelGroup 的线程池，当执行一个异步的时候，可以将这个异步操作提交到 AsynchronousChannelGroup 的线程池中。

AsynchronousChannelGroup 对象可以多个 Channel 所共享。

``` java
// Opens an asynchronous server-socket channel.
static AsynchronousServerSocketChannel open() 
static AsynchronousServerSocketChannel open(AsynchronousChannelGroup group) 
```

当使用无参数的方法，获得一个 AsynchronousServerSocketChannel 的实例的时候，或者调用第二个方法时传入 null 值的时候，创建的这个将会共享整个 JVM 中惟一的一个 AsynchronousChannelGroup 对象实例。

在不同的操作系统平台上 AsynchronousChannelGroup 的实现也有所有不同。

* linux 

	EPollPort

* unix

	SolarisEventPort

* windows

	Iocp

AsynchronousChannelGroup 的作用：

A AsynchronousChannelGroup has an associated thread pool 

* tasks are submitted to handle I/O events
* dispatch to completion-handlers that consume the result of asynchronous operations performed on channels in the group
* execute other tasks required to support the execution of asynchronous I/O operations. 


## AsynchronousServerSocketChannel

### unix


#### SolarisEventPort

#### UnixAsynchronousServerSocketChannelImpl

### linux

#### EPollPort

内部维护了哪些数据，下面方法是如何按照这些数据展开的。

##### 创建

``` java

```


##### 使用

#### UnixAsynchronousServerSocketChannelImpl

### windows

WindowsAsynchronousServerSocketChannelImpl

## AsynchronousSocketChannel

* unix

UnixAsynchronousSocketChannelImpl

* windows

WindowsAsynchronousSocketChannelImpl

## AsynchronousChannelGroup

AsynchronousChannelGroup 创建时会关联一个线程池，如果不进行配置，则默认的线程池大小是：

``` java
// 默认的 poolsize 是当前 JVM 所运行的机器的处理器个数。
initialSize = Runtime.getRuntime().availableProcessors();

ExecutorService executor =
	// 注意到这个配置，线程池的 core线程 数大小为0
	// 而配置的超时时间，几乎是在JVM运行周期内不会出现超时
	// 所以后续在线程池内创建的线程，将一直运行，不会出现
	// 超时。
    new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                           Long.MAX_VALUE, TimeUnit.MILLISECONDS,
                           new SynchronousQueue<Runnable>(),
                           threadFactory);


// AsynchronousChannelGroupImpl
protected final void startThreads(Runnable task) {

	// 默认情况下 isFixedThreadPool 方法返回 false
    if (!isFixedThreadPool()) {
		// 所以这里会启动一个线程
		// 并且这个线程是 后台线程
		// 然后 task 会被调用。
        for (int i=0; i<internalThreadCount; i++) {
            startInternalThread(task);
            threadCount.incrementAndGet();
        }
    }

	// pool.poolSize 也是大于0 的
	// 所以下面的方法会被调用。
    if (pool.poolSize() > 0) {
        task = bindToGroup(task);
        try {
			// 默认 poolSize 是处理器的个数
			// 可见，下面的方法就是将 task 提交到线程池中。
            for (int i=0; i<pool.poolSize(); i++) {
                pool.executor().execute(task);
                threadCount.incrementAndGet();
            }
        } catch (RejectedExecutionException  x) {
            // nothing we can do
        }
    }
}
```

由此，可见，当创建一个 AsynchronousChannelGroup 时，至少会有两个线程，执行 上面的 task 任务。

所以，当下面的调用发生在一个具有4个处理器的机器上时，会创建5个线程。

``` java
defaultPort = new EPollPort(this, ThreadPool.getDefault()).start();
```

这5个线程中的第一个线程是内部线程。

其余的4个线程是 ThreadPoolExecutor 中的4个线程。

这5个线程将执行 `sun.nio.ch.EPollPort.EventHandlerTask` 任务。

## EventHandlerTask

`EventHandlerTask` 实现的逻辑如下：

1. 从 queue 中获取事件
2. 如果获得的事件是 NEED_TO_POLL ，则执行 EventHandlerTask.poll 方法
3. poll 方法将在 pollfd 上执行 `epoll_wait` 方法，直到返回
4. `epoll_wait` 方法返回之后，放置初始化对应的事件 Event 添加到 queue 中
5. run 方法从 queue 中获得事件，进行相应的处理
6. 调用 channel 的  onEvent 方法。

## 调用 channel 的异步操作

* AsynchronousServerSocketChannel.accept
* AsynchronousSocketChannel.connect
* AsynchronousSocketChannel.read
* AsynchronousSocketChannel.write

这四个方法的实现，最终都是调用 startPoll 来实现。startPoll 的作用就是向 epfd 注册当前 fd 所关心的事件。

``` java
// EPollPort.startPoll 方法。
void startPoll(int fd, int events) {
    // update events (or add to epoll on first usage)
    // 调用 epoll_ctl 方法。
    int err = epollCtl(epfd, EPOLL_CTL_MOD, fd, (events | EPOLLONESHOT));
    if (err == ENOENT)
        err = epollCtl(epfd, EPOLL_CTL_ADD, fd, (events | EPOLLONESHOT));
    if (err != 0)
        throw new AssertionError();     // should not happen
}
```

## 事件的处理 onEvent

``` java
// isPooledThread 表示当前线程是否是 线程池中的线程。
// 如果是线程池中的线程则 isPooledThread == true
ev.channel().onEvent(ev.events(), isPooledThread);

// UnixAsynchronousServerSocketChannelImpl
// 和 UnixAsynchronousSocketChannelImpl
// 都实现了 Port.PollableChannel 接口
// 所以，其都实现对了 onEvent 方法用来处理事件。
```

## 资源的回收

可见当使用 AsynchronousServerSocketChannel 时系统，就会创建一个默认的 AsynchronousChannelGroup 这个对象，会分配一个 epoll 对象，及至少 2 个线程。这些都是系统资源，所以当使用完 AsynchronousServerSocketChannel 的时候，需要正确的回收这些资源。

### 隐式关闭

AsynchronousServerSocketChannel 在关闭的时候，会从 AsynchronousChannelGroup 对象上进行 unregister 操作。unregister 方法中会进行判断，如果当前 Group 中已经没有 channel 了，则AsynchronousChannelGroup 会执行 shutdown 操作。正是这个 `shutdonwNow` 方法调用会回收上面些系统资源。

### 显式关闭

AsynchronousChannelGroup 接口提供了 `shutdown` 和 `shutdownNow` 方法用来显式关闭 AsynchronousChannelGroup 所使用的系统资源。