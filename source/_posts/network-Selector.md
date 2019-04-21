---
title: Selector
date: 2016-12-1 14:52:10
---

## 1. socket 的 I/O 编程模型

socket 在 c 中的使用 select / poll / epoll

### select

``` c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout)

// Removes the descriptor fd from set.
void FD_CLR(int fd, fd_set *set);

// Nonzero if fd is a member of the set. Otherwise, zero.
int  FD_ISSET(int fd, fd_set *set);

// Adds descriptor fd to set.
void FD_SET(int fd, fd_set *set);

// Initializes the set to the null set.
void FD_ZERO(fd_set *set);

struct timeval {
    long    tv_sec;         /* seconds */
    long    tv_usec;        /* microseconds */
};
```

> select() and pselect() allow a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become "ready" for some class of I/O operation (e.g., input possible). A file descriptor is considered ready if it is possible to perform the corresponding I/O operation (e.g., read(2)) without blocking.
>
> Three independent sets of file descriptors are watched. 
>
> On exit, the sets are modified in place to indicate which file descriptors actually changed status. Each of the three file descriptor sets may be specified as NULL if no file descriptors are to be watched for the corresponding class of events.
>
> On success, select() and pselect() return the number of file descriptors contained in the three returned descriptor sets (that is, the total number of bits that are set in readfds, writefds, exceptfds) which may be zero if the timeout expires before anything interesting happens. On error, -1 is returned, and errno is set appropriately; the sets and timeout become undefined, so do not rely on their contents after an error.
> 
> FD_ZERO() clears a set. FD_SET() and FD_CLR() respectively add and remove a given file descriptor from a set. FD_ISSET() tests to see if a file descriptor is part of the set;


### poll

``` c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```

> poll() performs a similar task to select(2): it waits for one of a set of file descriptors to become ready to perform I/O.
>
> **fds:** The set of file descriptors to be monitored is specified in the fds argument, which is an array of structures of pollfd
>
> **nfds:** specify the number of items in the fds array
>
> **timeout:** specifies the minimum number of milliseconds that poll() will block.

poll 调用后执行的效果

> On success, a positive number is returned; this is the number of structures which have nonzero revents fields (in other words, those descriptors with events or errors reported). A value of 0 indicates that the call timed out and no file descriptors were ready. On error, -1 is returned, and errno is set appropriately.

其中 pollfd 结构体中的 `revents`, The field revents is an **output parameter**, filled by the kernel with the events that actually occurred. 

### epoll

> The epoll API performs a similar task to poll(2): monitoring multiple file descriptors to see if I/O is possible on any of them.
>
> Three system calls are provided to set up and control an epoll set: epoll_create(2), epoll_ctl(2), epoll_wait(2).
>
> An epoll set is connected to a file descriptor created by epoll_create(2). Interest for certain file descriptors is then registered via epoll_ctl(2). Finally, the actual wait is started by epoll_wait(2).

#### epoll_create

``` c
int epoll_create(int size);
```

> epoll_create() creates an epoll(7) instance. Since Linux 2.6.8, the size argument is ignored, but must be greater than zero;
>
> **epoll_create() returns a file descriptor referring to the new epoll instance. This file descriptor is used for all the subsequent calls to the epoll interface.** When no longer required, the file descriptor returned by epoll_create() should be closed by using close(2). When all file descriptors referring to an epoll instance have been closed, the kernel destroys the instance and releases the associated resources for reuse.

#### epoll_ctl

This system call performs control operations on the epoll(7) instance referred to by the file descriptor epfd. It requests that the operation op be performed for the target file descriptor fd.

``` c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```

* epfd

	the epoll(7) instance referred to by the file descriptor epfd.
	
* op

	the operation op be performed for the target file descriptor **fd**
	 
	* EPOLL_CTL_ADD
	
		Register the target file descriptor fd on the epoll instance referred to by the file descriptor epfd and associate the event event with the internal file linked to fd.
		
	* EPOLL_CTL_MOD
	
		Change the event event associated with the target file descriptor fd.
		
	* EPOLL_CTL_DEL
	
		Remove (deregister) the target file descriptor fd from the epoll instance referred to by epfd. The event is ignored and can be NUL.

* fd

	执行上述操作的目标 file descriptor
	
* event

	The event argument describes the object linked to the file descriptor **fd**.
	
	这个参数是一个 epoll_event 结构体的对象
	
	其中的 events 表示关注的事件类型，可以是以下值：
	
	* EPOLLIN
	
		The associated file is available for read(2) operations.
		
	* EPOLLOUT
	
		The associated file is available for write(2) operations.

#### epoll_wait

##### 函数定义 

``` c
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
```

##### 函数行为

The epoll_wait() system call waits for events on the epoll(7) instance referred to by the file descriptor epfd.

##### 函数参数

* epfd

	the epoll(7) instance referred to by the file descriptor epfd
	
* events

	The memory area pointed to by events will contain the events that will be available for the caller.
	
* maxevents

	Up to maxevents are returned by epoll_wait(). The maxevents argument must be greater than zero.
	
* timeout

	The timeout argument specifies the minimum number of milliseconds that epoll_wait() will block. 

##### 函数返回值

> When successful, epoll_wait() returns the number of file descriptors ready for the requested I/O, or zero if no file descriptor became ready during the requested timeout milliseconds. When an error occurs, epoll_wait() returns -1 and errno is set appropriately.

## 2. jdk 中实现的 select

### 2.1 SelectableChannel

> A channel that can be multiplexed via a Selector. 
>
> SelectableChannel 是一种可以被 Selector 使用多路复用的 Channel
> 
> **In order to be used with a selector, an instance of this class must first be registered via the register method.** This method returns a new SelectionKey object that represents the channel's registration with the selector. 
>
> **A channel must be placed into non-blocking mode before being registered with a selector,** and may not be returned to blocking mode until it has been deregistered.

调用 `register` 返回一个 SelectionKey

这个接口的功能：

* 注册 channel 到一个 selector

	调用 `register` 将当前 channel 注册到 selector

* 一个 channel 可以注册到多个 selector

	一个 channel 可以注册到多个完全不同的 selector，同时对于同一个 selector 多次调用 `register` 只是改变当前已经注册的 SelectionKey 的相关属性。

* channel 取消注册

	channel 关闭时，默认将从 selector 取消注册。（调用其 cancel 方法）

由于 SelectableChannel 可以注册到多个 selector 上，所有这个接口的实现中持有一个SelectionKey 的数组，用来维护，所有已经注册到当前Channel上的 selector。

``` java
// AbstractSelectableChannel
public abstract class AbstractSelectableChannel extends SelectableChannel{

	// ...

	// Keys that have been created by registering this channel with selectors.
    // They are saved because if this channel is closed the keys must be
    // deregistered.  Protected by keyLock.
    //
    private SelectionKey[] keys = null;
    private int keyCount = 0;

    public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        if (!isOpen())
            throw new ClosedChannelException();
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
		// 在注册时对 regLock 进行加锁操作，保证线程安全
        synchronized (regLock) {
			// 使用 select io模型时，channel 必须牌 non-blocking 模式。
            if (blocking)
                throw new IllegalBlockingModeException();

			// 查找待注册的 Selector 的 SelectionKey
            SelectionKey k = findKey(sel);
            if (k != null) {
				// k 不为空，则表示当前 channel 已经在 sel 注册过了
				// 所以只是改变下面的属性即可。
				// 覆盖 interestOps
                k.interestOps(ops);
				// 覆盖 attachment
                k.attach(att);
            }
            if (k == null) {
				// 表明是第一次注册，调用 sel 的 register 方法进行注册。
                k = ((AbstractSelector)sel).register(this, ops, att);
				// 将注册成功之后的 SelectionKey 添加到 当前 channel 维护的
				// keys 数组中。
                addKey(k);
            }
            return k;
        }
    }

	// ...
}
```

由于不同的 SelectableChannel 可能关注的操作不同，所以这个类的 `validOps` 方法由子类来实现。只有具体的子类才知道，自己关注什么操作。

同时 `AbstractSelectableChannel` 重写了 `implCloseChannel`：

``` java
protected final void implCloseChannel() throws IOException {
    implCloseSelectableChannel();
    synchronized (keyLock) {
        int count = (keys == null) ? 0 : keys.length;
		// 当 channel 被关闭的时候，释放 channel 在 selector 的注册。
        for (int i = 0; i < count; i++) {
            SelectionKey k = keys[i];
            if (k != null)
                k.cancel();
        }
    }
}
```

`keyFor` 方法用来，查询 channel 在 selector 上的注册关系 SelectionKey

### 2.2 SelectionKey

> A token representing the registration of a SelectableChannel with a Selector. 
> 
> 代表 Channel 和 Selector 的注册关系

所以 SelectionKey 持有具有注册关系的 <Channel, Selector> 对。

其 `channel` 方法返回注册关系中的 SelectableChannel 对象

其 `selector` 方法返回注册关系中的 Selector 对象

#### 2.2.1 断开注册关系

SelectionKey 的 `cancel` 方法可以断开这种注册关系

#### 2.2.2 注册关系的属性

* attachment

	表示当前 key 对象，附加的一个参数。

	可以使用 `attach` 方法将一个对象附加到这个 key 上，同时这个方法将返回之前已经附加过的对象。

	使用 `attachment` 返回当前附加的对象。

	**这个属性存在的意义：`register` 方法的调用(SelectionKey对象的创建)和 SelectionKey的使用通常不在一个线程中，所以可以通过这个对象，实现，跨线程传递参数。**

* interestOps

	表示这个 key 所关注的操作。
	
	使用 `interestOps` 方法可以进行设置，注意这个设置是完全覆盖，而不是和已经存在的值进行异或。

#### 2.2.3 用来标识在 Selector 上注册的某种操作，已经准备就绪

``` java
class SelectionKeyImpl extends AbstractSelectionKey {

	// 用来标识，已经准备就绪的操作。
    private int readyOps;

	// 读取 readyOps
    public int readyOps() {
        ensureValid();
        return readyOps;
    }

    int nioReadyOps() {
        return readyOps;
    }

	// 设置 readyOps
    void nioReadyOps(int ops) {
        readyOps = ops;
    }

	// 来自父类 SelectionKey
	public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;

	// Tests whether this key's channel is ready for reading. 
    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

	// Tests whether this key's channel is ready for writing. 
    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

	// Tests whether this key's channel has either finished, or failed to finish, its socket-connection operation.
    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

	// Tests whether this key's channel is ready to accept a new socket connection. 
    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }
}
```



### 2.3 Selector

> A multiplexor of SelectableChannel objects. 
> 
> 多路复用器 for SelectableChannel 对象
> 

**select 方法的实现**

``` java
abstract class SelectorImpl extends AbstractSelector {

	// 由具体的子类实现 select 方法。
    protected abstract int doSelect(long timeout) throws IOException;

    private int lockAndDoSelect(long timeout) throws IOException {
        synchronized (this) {
            if (!isOpen())
                throw new ClosedSelectorException();
            synchronized (publicKeys) {
                synchronized (publicSelectedKeys) {
                    return doSelect(timeout);
                }
            }
        }
    }

    public int select(long timeout)
        throws IOException
    {
        if (timeout < 0)
            throw new IllegalArgumentException("Negative timeout");
        return lockAndDoSelect((timeout == 0) ? -1 : timeout);
    }

    public int select() throws IOException {
        return select(0);
    }

    public int selectNow() throws IOException {
        return lockAndDoSelect(0);
    }

```

#### 2.3.1 linux 平台下的实现

* SunOS

	`DevPollSelectorImpl`

* Linux & Linux kernels >= 2.6

	`EPollSelectorImpl`

* Linux

	`PollSelectorImpl`

#### 2.3.2 windows 平台下的实现

`WindowsSelectorImpl`

* register 的实现

	``` java
	// SelectorImpl
    protected final SelectionKey register(AbstractSelectableChannel ch,
                                          int ops,
                                          Object attachment)
    {
        if (!(ch instanceof SelChImpl))
            throw new IllegalSelectorException();
		// 创建 SelectionKey
        SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);

		// 添加附件
        k.attach(attachment);

        synchronized (publicKeys) {
			// 调用子类的实现，进行注册。
            implRegister(k);
        }

		// 设置 interestOps
        k.interestOps(ops);
        return k;
    }

    private final int INIT_CAP = 8;
    private SelectionKeyImpl[] channelArray = new SelectionKeyImpl[INIT_CAP];
	// WindowsSelectorImpl
    protected void implRegister(SelectionKeyImpl ski) {
        synchronized (closeLock) {
            if (pollWrapper == null)
                throw new ClosedSelectorException();
            growIfNeeded();

			// 将 SelectionKey 存储到 Selector 相关的结构中
			
			// 以数组的形式存储 SelectionKey
            channelArray[totalChannels] = ski;
            // 设置 SelectionKey 在 channelArray 数组中的索引
            ski.setIndex(totalChannels);
            // 可以通过 fdMap.get(fd); 查找到 fd 对应的 SelectionKey 对象，
            // 用在当 select 操作完成之后，需要对 fd 所对应的 SelectionKey 进行设置时。
            fdMap.put(ski);
            // The set of keys registered with this Selector
            // keys 存储所有在 Selector 上注册的 SelectionKey
            // Selector.keys 方法将返回这个集合
            keys.add(ski);
            
            // totalChannels 表示 ski 在 channelArray 中的索引位置
            // 将注册的Socket的 fd 存储到 pollWrapper 中的 pollArray 中
            // pollArray 的结构如下：
            //  +---+---+---+---+---+---+---+---+---+---+---+---+
            //  |       | fd  e | fd  e | fd  e | fd  e | fd  e |
            //  +---+---+---+---+---+---+---+---+---+---+---+---+
            // 其中 fd 占4个字节，表示 socket 的文件描述符
            //      e  占4个字节，表示 fd 此时的关注的事件
            pollWrapper.addEntry(totalChannels, ski);

            totalChannels++;
        }
    }
	```
* `doSelect`

	``` java
    protected int doSelect(long timeout) throws IOException {
        if (channelArray == null)
            throw new ClosedSelectorException();
        this.timeout = timeout; // set selector timeout
        processDeregisterQueue();
        if (interruptTriggered) {
            resetWakeupSocket();
            return 0;
        }
        // Calculate number of helper threads needed for poll. If necessary
        // threads are created here and start waiting on startLock
        adjustThreadsCount();
        finishLock.reset(); // reset finishLock
        // Wakeup helper threads, waiting on startLock, so they start polling.
        // Redundant threads will exit here after wakeup.
        startLock.startThreads();
        // do polling in the main thread. Main thread is responsible for
        // first MAX_SELECTABLE_FDS entries in pollArray.
        try {
            begin();
            try {
				// poll 方法最终调用 系统底层的 select 方法
                subSelector.poll();
            } catch (IOException e) {
                finishLock.setException(e); // Save this exception
            }
            // Main thread is out of poll(). Wakeup others and wait for them
            if (threads.size() > 0)
                finishLock.waitForHelperThreads();
          } finally {
              end();
          }
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        finishLock.checkForException();
        processDeregisterQueue();

		// 更新 selectedKeys, 则 Selector.selectedKeys 返回这些值
        int updated = updateSelectedKeys();
        // Done with poll(). Set wakeupSocket to nonsignaled  for the next run.
        resetWakeupSocket();
        return updated;
    }
	```

updateSelectedKeys 方法的使用主要有两个：

1. 更新 selectedKeys 

	SelectorImpl.selectedKeys 中存储着已经可以进行操作的 SelectionKey，这里的操作是指 read, write, connection 等操作。
	
	也就是说 selectedKeys 集合里面的 SelectionKey 所指向的 Socket 可以进行上述操作，而不会被 block.
	
2. 设置 SelectionKey 中的兴趣集

	上面更新的 selectedKeys 集合，可以通过 Selecor.selectedKeys 方法获得。但是对于 selectedKeys 中的每一个 SelectionKey， 还需要知道它此时是可以进行什么操作，所以需要对其进行设置。
	
	也就是设置 SelectionKeyImpl.readyOps 字段。
	
	这样 SelectionKey 的 isWritable， isReadable， isConnectable，isAcceptable 就可以依据 readyOps 进行判断了。
	
	核心实现在 SocketChannelImpl.translateReadyOps 方法中。

* subSelector.poll() 的实现

``` java
private int processSelectedKeys(long updateCount) {
    int numKeysUpdated = 0;
    numKeysUpdated += processFDSet(updateCount, readFds,
                                   PollArrayWrapper.POLLIN,
                                   false);
    numKeysUpdated += processFDSet(updateCount, writeFds,
                                   PollArrayWrapper.POLLCONN |
                                   PollArrayWrapper.POLLOUT,
                                   false);
    numKeysUpdated += processFDSet(updateCount, exceptFds,
                                   PollArrayWrapper.POLLIN |
                                   PollArrayWrapper.POLLCONN |
                                   PollArrayWrapper.POLLOUT,
                                   true);
    return numKeysUpdated;
}
```

``` c
typedef struct {
    jint fd;
    jshort events;
} pollfd;

JNIEXPORT jint JNICALL
Java_sun_nio_ch_WindowsSelectorImpl_00024SubSelector_poll0(JNIEnv *env, jobject this,
                                   jlong pollAddress, jint numfds,
                                   jintArray returnReadFds, jintArray returnWriteFds,
                                   jintArray returnExceptFds, jlong timeout)
{
    DWORD result = 0;
    pollfd *fds = (pollfd *) pollAddress;
    int i;
    FD_SET readfds, writefds, exceptfds;
    struct timeval timevalue, *tv;
    static struct timeval zerotime = {0, 0};
    int read_count = 0, write_count = 0, except_count = 0;

	// 初始化 timeval 参数
    if (timeout == 0) {
        tv = &zerotime;
    } else if (timeout < 0) {
        tv = NULL;
    } else {
        tv = &timevalue;
        tv->tv_sec =  (long)(timeout / 1000);
        tv->tv_usec = (long)((timeout % 1000) * 1000);
    }

    /* Set FD_SET structures required for select */
    // 初始化 三个 FD_SET
    for (i = 0; i < numfds; i++) {
        if (fds[i].events & sun_nio_ch_PollArrayWrapper_POLLIN) {
           readfds.fd_array[read_count] = fds[i].fd;
           read_count++;
        }
        if (fds[i].events & (sun_nio_ch_PollArrayWrapper_POLLOUT |
                             sun_nio_ch_PollArrayWrapper_POLLCONN))
        {
           writefds.fd_array[write_count] = fds[i].fd;
           write_count++;
        }
        exceptfds.fd_array[except_count] = fds[i].fd;
        except_count++;
    }

    readfds.fd_count = read_count;
    writefds.fd_count = write_count;
    exceptfds.fd_count = except_count;

    /* 调用 select */
    if ((result = select(0 , &readfds, &writefds, &exceptfds, tv))
			== SOCKET_ERROR) {
   		// handle error
    }

    /* Return selected sockets. */
    /* Each Java array consists of sockets count followed by sockets list */
    // 处理返回值 returnReadFds, returnWriteFds, returnExceptFds
    (*env)->SetIntArrayRegion(env, returnReadFds, 0,
                              readfds.fd_count + 1, (jint *)&readfds);

    (*env)->SetIntArrayRegion(env, returnWriteFds, 0,
                              writefds.fd_count + 1, (jint *)&writefds);
    (*env)->SetIntArrayRegion(env, returnExceptFds, 0,
                              exceptfds.fd_count + 1, (jint *)&exceptfds);
    return 0;
}

```


## 参考
1. [select](https://linux.die.net/man/2/select)
2. [poll](https://linux.die.net/man/2/poll)
3. [epoll](https://linux.die.net/man/4/epoll)
4. [epoll_create](https://linux.die.net/man/2/epoll_create)
5. [epoll_ctl](https://linux.die.net/man/2/epoll_ctl)
6. [epoll_wait](https://linux.die.net/man/2/epoll_wait)
7. [select、poll、epoll之间的区别总结](http://www.cnblogs.com/Anker/p/3265058.html)
8. [IO 多路复用是什么意思](https://www.zhihu.com/question/32163005)
9. [高性能IO模型浅析](http://www.cnblogs.com/fanzhidongyzby/p/4098546.html)
10. [Comparing Two High-Performance I/O Design Patterns: Reactor and Proactor](http://www.artima.com/articles/io_design_patternsP.html)
11. [epoll 或者 kqueue 的原理是什么](https://www.zhihu.com/question/20122137)
12. [The C10K problem](http://www.kegel.com/c10k.html)
13. [Reactor模式详解](http://www.blogjava.net/DLevin/archive/2015/09/02/427045.html)
14. [IO设计模式：Reactor和Proactor对比](https://segmentfault.com/a/1190000002715832)
15. [Java NIO系列教程（六） Selector](http://ifeve.com/selectors/)