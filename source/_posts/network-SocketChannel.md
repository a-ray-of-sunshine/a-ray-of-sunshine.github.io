---
title: SocketChannel
date: 2016-11-26 14:15:04
---

## 1. ServerSocketChannel

### 1.1 ServerSocketChannel 的创建

#### 1.1.1 获取 SelectorProvider 对象

SelectorProvider 这个类用来获取 selectors and selectable channels

> Service-provider class for selectors and selectable channels. 

其 provider 方法用来查找当前系统中 SelectorProvider 的实现，其查找过程如下：

> 1. If the system property java.nio.channels.spi.SelectorProvider is defined then it is taken to be the fully-qualified name of a concrete provider class. The class is loaded and instantiated; if this process fails then an unspecified error is thrown. 
> 
> 2. If a provider class has been installed in a jar file that is visible to the system class loader, and that jar file contains a provider-configuration file named java.nio.channels.spi.SelectorProvider in the resource directory META-INF/services, then the first class name specified in that file is taken. The class is loaded and instantiated; if this process fails then an unspecified error is thrown. 
> 
> 3. Finally, if no provider has been specified by any of the above means then the system-default provider class is instantiated and the result is returned. 

上面对应是实现：

1. 获得系统属性

	``` java
	String cn = System.getProperty("java.nio.channels.spi.SelectorProvider");
    Class<?> c = Class.forName(cn, true,ClassLoader.getSystemClassLoader());
	provider = (SelectorProvider)c.newInstance();
	```

2. 通过 ServiceLoader 查找 SelectorProvider

	``` java
	ServiceLoader<SelectorProvider> sl =
            ServiceLoader.load(SelectorProvider.class,
                               ClassLoader.getSystemClassLoader());
	```

3. 提供JDK默认的实现

	``` java
	provider = sun.nio.ch.DefaultSelectorProvider.create();
	```

而这个 DefaultSelectorProvider 类，在不同的操作系统平台上有不同的实现。注意 DefaultSelectorProvider 这个类本身和 SelectorProvider 没有任何关系（例如：继承，实现等）。它相当于一个JDK到不同平台的一个中间层，不同的平台通过提供这个中间层来向上层提供服务。这个中间层在编译连接JDK时，就会根据不同的平台连接相对应的源码实现文件。

* windows -- WindowsSelectorProvider

	``` java
	/**
	 * Returns the default SelectorProvider.
	 */
	public static SelectorProvider create() {
	    return new sun.nio.ch.WindowsSelectorProvider();
	}
	```

* linux -- EPollSelectorProvider

	``` java
    /**
     * Returns the default SelectorProvider.
     */
    public static SelectorProvider create() {
        String osname = AccessController.doPrivileged(
            new GetPropertyAction("os.name"));
        if ("SunOS".equals(osname)) {
            return new sun.nio.ch.DevPollSelectorProvider();
        }

        // use EPollSelectorProvider for Linux kernels >= 2.6
        if ("Linux".equals(osname)) {
            String osversion = AccessController.doPrivileged(
                new GetPropertyAction("os.version"));
            String[] vers = osversion.split("\\.", 0);
            if (vers.length >= 2) {
                try {
                    int major = Integer.parseInt(vers[0]);
                    int minor = Integer.parseInt(vers[1]);
                    if (major > 2 || (major == 2 && minor >= 6)) {
                        return new sun.nio.ch.EPollSelectorProvider();
                    }
                } catch (NumberFormatException x) {
                    // format not recognized
                }
            }
        }

        return new sun.nio.ch.PollSelectorProvider();
    }
	```

由此可知：SelectorProvider 在 windows 平台上的实现是：`sun.nio.ch.WindowsSelectorProvider`，在 linux 平台上的实现是：`sun.nio.ch.EPollSelectorProvider`

* WindowsSelectorProvider

	``` java
	public class WindowsSelectorProvider extends SelectorProviderImpl {
	    public AbstractSelector openSelector() throws IOException {
	        return new WindowsSelectorImpl(this);
	    }
	}
	```

* EPollSelectorProvider

	``` java
	public class EPollSelectorProvider extends SelectorProviderImpl {
	    public AbstractSelector openSelector() throws IOException {
	        return new EPollSelectorImpl(this);
	    }
	
	    public Channel inheritedChannel() throws IOException {
	        return InheritedChannel.getChannel();
	    }
	}
	```

可以看到这两个类都继承自 SelectorProviderImpl 类，SelectorProviderImpl 这个是不同平台所共享的。

不同的一点就是这两个类重写了 openSelector 方法，提供了不同平台上 java.nio.channels.Selector 的实现。由此可知 Selector 在 windows 平台上的实现是： WindowsSelectorImpl，而在 linux 平台上的实现则是： EPollSelectorImpl

#### 1.1.2 SelectorProviderImpl --- SelectorProvider的实现

查看 SelectorProviderImpl 实现，总结 SelectorProvider 的各个 open 方法的实现如下：

* openDatagramChannel -- **DatagramChannel**

	`new DatagramChannelImpl(this);`

* openPipe -- **Pipe**

	`new PipeImpl(this);`

* openServerSocketChannel -- **ServerSocketChannel**

	`new ServerSocketChannelImpl(this);`

* openSocketChannel -- **SocketChannel**

	`new SocketChannelImpl(this);`

* openSelector -- **AbstractSelector**

	* windows

		`new WindowsSelectorImpl(this);`

	* linux

		`new EPollSelectorImpl(this);`

## 1.2 ServerSocketChannel 的实现 -- ServerSocketChannelImpl

由上面的分析可知 SelectorProviderImpl 通过 openServerSocketChannel 方法创建一个 ServerSocketChannelImpl 对象来实现 ServerSocketChannel。

``` java
class ServerSocketChannelImpl
    extends ServerSocketChannel
    implements SelChImpl{
	// ...
    // Used to make native close and configure calls
    private static NativeDispatcher nd;

    private native int accept0(FileDescriptor ssfd, FileDescriptor newfd, InetSocketAddress[] isaa) throws IOException;

    private static native void initIDs();

    static {
		// 这个调用的作用是加载 net.dll 和 nio.dll 文件到 JVM 中。
		// 也就是 nio 和 net 中的 natvie 实现。
        Util.load();

		// 初始化 ServerSocketChannelImpl 中 native 代码中
		// 所使用的到相关数据结构,主要是JNI调用中使用的一些ID
        initIDs();

		// 初始化 NativeDispatcher 对象。
        nd = new SocketDispatcher();
    }

	// 这个字段在父类 AbstractSelectableChannel 中。
    // The provider that created this channel
    private final SelectorProvider provider;

	// fd 存储 socket 的句柄（文件描述符）
    // Our file descriptor
    private final FileDescriptor fd;

	// 直接存储 socket 的句柄值。
    // fd value needed for dev/poll.
    private int fdVal;

	// Channel 的状态，由于这个对象继承 AbstractInterruptibleChannel
	// 所以 Channel 需要支持异步关闭（close）,要想正确实现异步关闭
	// 就必须有一个字段存储当前 Channel 的状态，当执行异步 close 操作时
	// 就可以根据这个状态，执行相应的资源回收等操作。
    // Channel state, increases monotonically
    private static final int ST_UNINITIALIZED = -1; // 还未初始化
    private static final int ST_INUSE = 0;          // 正在使用中
    private static final int ST_KILLED = 1;         // 已经被关闭
    private int state = ST_UNINITIALIZED; // 默认状态：未初始化

	// 构造函数
    ServerSocketChannelImpl(SelectorProvider sp) throws IOException {
		// 将 sp 存储到 父类 AbstractSelectableChannel 的 provider 字段中
        super(sp);

		// 通过 native 方法 socket0 最终调用系统提供的 socket
		// 方法创建一个 socket, 将返回的句柄（文件描述符）封装到
		// FileDescriptor 对象中返回。
        this.fd =  Net.serverSocket(true);

		// 将 fd 中的存储的句柄的值直接取出来
		// 存储到 int 类型的变量 fdVal 中。
        this.fdVal = IOUtil.fdVal(fd);

		// 设置 Channel 当前的状态，socket 已经完全初始化好了
		// 可以正常使用了
        this.state = ST_INUSE;
    }

	// ...
}
```

上面静态代码段和构造函数就是整个 ServerSocketChannel 创建时所执行的操作。

## 1.3 ServerSocketChannel 的使用

ServerSocketChannel 创建成功之后，并不能立即使用，此时的 socket 还是 unbound 的。

> The new channel's socket is initially unbound; it must be bound to a specific address via one of its socket's bind methods before connections can be accepted. 

所以在使用 accept 方法之前需要调用  bind 方法。

### 1.3.1 bind

使用就是调用 socket 的 bind 和 listen 方法，使得 socket 处于监听状态。可以接受外部请求。

``` java
// ServerSocketChannelImpl 实现的 bind 方法
public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
    synchronized (lock) {
        if (!isOpen())
            throw new ClosedChannelException();

		// 不能多次 bind.
        if (isBound())
            throw new AlreadyBoundException();
        InetSocketAddress isa = (local == null) ? new InetSocketAddress(0) :
            Net.checkAddress(local);
        SecurityManager sm = System.getSecurityManager();
        if (sm != null)
            sm.checkListen(isa.getPort());
        NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());

		// 调用 底层的 bind 方法将 socket 进行 bind
        Net.bind(fd, isa.getAddress(), isa.getPort());

		// 进行 listen 操作。同样也是最终调用底层的 listen 方法。
		// 此时 socket 就处于 监听状态，可以接受外部的 tcp 连接请求了。
		// 注意 backlog 的大小默认为 50.
        Net.listen(fd, backlog < 1 ? 50 : backlog);
        synchronized (stateLock) {
            localAddress = Net.localAddress(fd);
        }
    }
    return this;
}
```

### 1.3.2 accept

调用底层的 accept 方法返回一个新的 socket 连接。

``` java
//  ServerSocketChannelImpl 实现的 accept 方法

// ID of native thread currently blocked in this channel, for signalling
private volatile long thread = 0;

public SocketChannel accept() throws IOException {

    int n = 0;
    FileDescriptor newfd = new FileDescriptor();
    InetSocketAddress[] isaa = new InetSocketAddress[1];

	// 记录当前进行accept方法调用时的线程的ID
	thread = NativeThread.current();

	// 调用底层的 accept 方法，接受一个 tcp 请求，创建一个 socket
	// 新创建的 socket 将被放置到 newfd 中。
    n = accept0(this.fd, newfd, isaa);

    IOUtil.configureBlocking(newfd, true);
    InetSocketAddress isa = isaa[0];
	// 创建一个 SocketChannel，表示一个客户请求。
    sc = new SocketChannelImpl(provider(), newfd, isa);

    return sc;
}

```


## 2. SocketChannel 

SocketChannel 对应于 Socket 的实现，表示通过的一端（peer）. 对于 Socket 的实现默认是阻塞的（block），并且也不能够配置其是否阻塞。但是对于 SocketChannel 是可以配置其是否阻塞的。

使用 AbstractSelectableChannel.configureBlocking 方法，用来 Adjusts this channel's blocking mode. 

### 2.1 SocketChannel 的实现 -- SocketChannelImpl

``` java
class SocketChannelImpl
    extends SocketChannel
    implements SelChImpl
{
	// ...

    // Used to make native read and write calls
    private static NativeDispatcher nd;

    static {
		// 载入 net & nio liberary
        Util.load();
		// 创建 NativeDispatcher 对象。
        nd = new SocketDispatcher();
    }
	
	// 存储 socket 的文件描述符。
    // Our file descriptor object
    private final FileDescriptor fd;

    // fd value needed for dev/poll.
    private final int fdVal;

	// socket的状态
    // State, increases monotonically
    private static final int ST_UNINITIALIZED = -1;
    private static final int ST_UNCONNECTED = 0;
    private static final int ST_PENDING = 1;
    private static final int ST_CONNECTED = 2;
    private static final int ST_KILLPENDING = 3;
    private static final int ST_KILLED = 4;
    private int state = ST_UNINITIALIZED;

	// Constructor for normal connecting sockets
    SocketChannelImpl(SelectorProvider sp) throws IOException {
        super(sp);
		// 调用系统的 socket 函数创建一个 Socket
        this.fd = Net.socket(true);
        this.fdVal = IOUtil.fdVal(fd);
		// 初始化 Channel 的状态。
        this.state = ST_UNCONNECTED;
    }

	// 当 ServerSocketChannel 调用 accept 返回一个 newFd 时
	// 调用下面的构造函数初始化一个 SocketChannel
    // Constructor for sockets obtained from server sockets
    SocketChannelImpl(SelectorProvider sp,
                      FileDescriptor fd, InetSocketAddress remote)
        throws IOException
    {
        super(sp);
        this.fd = fd;
        this.fdVal = IOUtil.fdVal(fd);
		// 由于是通过 accpet 返回的socket,所以已经是建立
		// 了连接的 socket
        this.state = ST_CONNECTED;
        this.localAddress = Net.localAddress(fd);
        this.remoteAddress = remote;
    }

	// ...
}
```

所以创建一个 SocketChannel 其实就是调用 socket 函数创建系统对应的 Socket。

### 2.2 connect

通过上面的方法创建一个 socket 之后并不能立即使用，而是需要使用 connect 函数和ServerSocket建立连接之后才可以进行 read 和 write 操作。

这个方法实现，同样依赖于底层的 connect 方法。关于 connect 方法的描述：

> If the connection or binding succeeds, zero is returned. On error, -1 is returned, and errno is set appropriately.

如果 connect 方法连接成功，则返回0，否则返回 -1，其中 errno 被设置成失败原因。

其中 errno 有这样一项：

> EINPROGRESS: The socket is non-blocking and the connection cannot be completed immediately. use getsockopt(2) to read the SO_ERROR option at level SOL_SOCKET to determine whether connect() completed successfully (SO_ERROR is zero) or unsuccessfully (SO_ERROR is one of the usual error codes listed here, explaining the reason for the failure). 

可见，当 connect 函数在 non-blocking 模式下，如果连接还未建立成功，则返回 EINPROGRESS。可以使用 getsockopt 来查询一个 socket 是否建立成功连接。

### 2.3 configureBlocking

设置 socket 的阻塞模式

* windows

	使用 ioctlsocket 来实现
	
	``` java
	#define SET_BLOCKING 0
	#define SET_NONBLOCKING 1

    if (blocking == JNI_FALSE) {
        argp = SET_NONBLOCKING;
    } else {
        argp = SET_BLOCKING;
        /* Blocking fd cannot be registered with EventSelect */
        WSAEventSelect(fd, NULL, 0);
    }
    result = ioctlsocket(fd, FIONBIO, &argp);
	```
	The ioctlsocket function controls the I/O mode of a socket.

	msdn中关于 ioctlsocket 的使用：
	当这个函数的第二个参数是 FIONBIO，其第三个参数的意义如下：
	> Set *argp to a nonzero value if the nonblocking mode should be enabled, or zero if the nonblocking mode should be disabled. When a socket is created, it operates in blocking mode by default

* linux

	使用 fcntl 函数来实现。

	``` java
    int flags = fcntl(fd, F_GETFL);
    int newflags = blocking ? (flags & ~O_NONBLOCK) : (flags | O_NONBLOCK);

    return (flags == newflags) ? 0 : fcntl(fd, F_SETFL, newflags);
	```

	fcntl 调用设置 socket 的属性
	> fcntl -- manipulate file descriptor 
	第二个参数是F_SETFL： Set the file status flags to the value specified by arg. 
	O_NONBLOCK： When possible, the file is opened in non-blocking mode. 

### 2.4 channel 的读写

从上面的分析可知，执行网络相关的函数时使用的时 sun.nio.ch.Net 类，这个类封装了操作系统提供的 socket, bind,listen, connect, setsockopt, getsockopt 函数的包装。这几个都是标准的 BSD Socket API, 用来进行网络编程的接口，所以将其封装到 Net 类中。

而对于 Socket 的读写操作，也就是 socket 的 IO 操作其实各个平台有不同的抽象和实现。这类的 API 常常和文件IO的API共用，所以JDK的实现对 socket channel 的读写时，并没有将 read, write 方法封装到 Net 类中，其实对于 read 和 write API在不同的平台上其作用可以和文件IO共用，所以 JDK 在实现的将IO操作封装到 sun.nio.ch.NativeDispatcher 这个抽象中。

``` java
/**
 * Allows different platforms to call different native methods
 * for read and write operations.
 */
abstract class NativeDispatcher{
	// ...

	// ...
}
```

NativeDispatcher 这个类的作用抽象不同平台的 read/write 操作。这个类的继承层次如下：

* NativeDispatcher
	* DatagramDispatcher
	* FileDispatcher
		* FileDispatcherImpl
	* SocketDispatcher

NIO关于 FileChannel, SocketChannel, DatagramChannel 的IO操作就由于上面的对应的抽象来完成。

所以 SocketChannel 读写就是由 SocketDispatcher 来实现的，
在不同的平台下，提供不同的 SocketDispatcher.java/SocketDispatcher.c 文件，

#### 2.4.1 linux平台下的 read/write

由于在 linux 平台下，socket的IO，可以完全当作文件IO来执行，所以这里 SocketChannel 的实现，直接由 FileDispatcherImpl 来实现。

``` java
class SocketDispatcher extends NativeDispatcher
{

    int read(FileDescriptor fd, long address, int len) throws IOException {
        return FileDispatcherImpl.read0(fd, address, len);
    }

    int write(FileDescriptor fd, long address, int len) throws IOException {
        return FileDispatcherImpl.write0(fd, address, len);
    }

	// ...
}
```

可以看到，SocketDispatcher 把 read/write 直接转发给 FileDispatcherImpl。

#### 2.4.2 windows平台下的 read/write

在 windows 平台下，有关于 socket的 Win32 API, 所以 SocketDispatcher 的实现，由这些 win32 API 来实现。

* read

	**`WSARecv`**

	> The WSARecv function receives data from a connected socket.

* wirte

	**`WSASend`**

	> The WSASend function sends data on a connected socket.

* close

	**`WSASendDisconnect`**

	> The WSASendDisconnect function initiates termination of the connection for the socket and sends disconnect data.

## 3. 基于流的Socket API 和基于 chnanel 的 Socket 对比

### 3.1 异常信息

基于 Channel 的 Socket 比 基于流的 Socket 提供的 API 抛出的 异常信息 更加明确。由于网络通信的复杂性，更加明确的异常，有利于开发更加健壮的程序。

### 3.2 读写效率

基于 Channel 的 socket 在进行数据IO时使用的是 `DirectByteBuffer` 这个类的作用是可以直接分配，不由JVM托管的内存。所以内存可以直接在 native 代码中使用，减少了数据缓冲区的复制次数，提高读写效率。

使用 ByteBuffer.allocate 分配的是 JVM 堆内存，使用 ByteBuffer.allocateDirect 分配的则是 native code 可以直接使用的内存。

## 4. 参考
1. [bytebuffer-allocate-vs-bytebuffer-allocatedirect](http://stackoverflow.com/questions/5670862/bytebuffer-allocate-vs-bytebuffer-allocatedirect)