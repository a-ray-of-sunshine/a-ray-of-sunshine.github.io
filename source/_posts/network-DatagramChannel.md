---
title: DatagramChannel
date: 2016-11-29 16:38:52
---

## 1. DatagramChannel的初始化

DatagramChannel 的 open 方法创建一个 DatagramChannelImpl 对象的一个实例。

所以创建一个 DatagramChannel 就是创建 DatagramChannelImpl 类的一个对象。

下面，来梳理创建一个 DatagramChannelImpl  对象的时候，创建了什么

``` java
class DatagramChannelImpl
    extends DatagramChannel
    implements SelChImpl
{
    // Used to make native read and write calls
    private static NativeDispatcher nd = new DatagramDispatcher();

    // Our file descriptor
    private final FileDescriptor fd;

    // fd value needed for dev/poll.
    private final int fdVal;
    
    // The protocol family of the socket
    private final ProtocolFamily family;
    
    // State (does not necessarily increase monotonically)
    private static final int ST_UNINITIALIZED = -1;
    private static final int ST_UNCONNECTED = 0;
    private static final int ST_CONNECTED = 1;
    private static final int ST_KILLED = 2;
    private int state = ST_UNINITIALIZED;
    
    public DatagramChannelImpl(SelectorProvider sp)
        throws IOException
    {
        super(sp);
        ResourceManager.beforeUdpCreate();
        try {
        	// 指定使用的协议簇
            this.family = Net.isIPv6Available() ?
                StandardProtocolFamily.INET6 : StandardProtocolFamily.INET;
                
            // 创建 socket
            this.fd = Net.socket(family, false);
            
            // 将 fd 中存储的 socket 的文件描述符，提取出来
            this.fdVal = IOUtil.fdVal(fd);
            
            // 设置当前 Channel 的状态
            this.state = ST_UNCONNECTED;
        } catch (IOException ioe) {
            ResourceManager.afterUdpClose();
            throw ioe;
        }
    }

    static {
    	// 载入，net, nio 动态库。
    	// 在 windows 平台是加载 %JAVA_HOME%/jre/bin
    	// 目录中的 net.dll 和 nio.dll 文件
    	// 在 linux 平台是加载 %JAVA_HOME%/jre/lib/amd64
    	// 这个文件夹中的 libnet.so 和 libnio.so
        Util.load();
        
        // 初始化 native 代码需要使用到的数据结构
        initIDs();
    }
}
```

创建的 socket.

``` c
// 可见在这里创建了一个 SOCK_DGRAM 数据报 Socket
fd = socket(AF_INET, SOCK_DGRAM, 0);
```

由此，可见创建一个 DatagramChannel , 其实就是创建一个 SOCK_DGRAM socket,然后将其文件描述符保存到 fd 对象中。

## 2. bind 的实现

对于 DatagramChannel 的 server 来说，其需要 bind 到一个本地IP和端口上，所以，需要调用 bind 方法。

bind 方法的作用：

> When a socket is created with socket, it exists in a name space (address family) but has no address assigned to it. bind() assigns the address specified by addr to the socket referred to by the file descriptor sockfd.
> 
> Traditionally, this operation is called "assigning a name to a socket".

DatagramChannel 实现的 bind 方法，除了调用上面的 bind 系统调用，还会初始化 DatagramChannelImpl 类的

``` java
// Binding
private SocketAddress localAddress;
private SocketAddress remoteAddress;

// DatagramChannelImpl.bind 方法
public DatagramChannel bind(SocketAddress local) throws IOException {
	// ...
    Net.bind(family, fd, isa.getAddress(), isa.getPort());
    // Net.localAddress 方法调用 getsockname 方法。
    // 获得 socket fd 最终系统分配给的 Addr
    localAddress = Net.localAddress(fd);
    // ...
    return this;
}

```

## 3. connect 的实现

DatagramChannel 调用 connect 方法之后，可以使用 write 方法，进行 channel 的写入了。否则使用 write 方法，将收到 `NotYetConnectedException`

``` java
public DatagramChannel connect(SocketAddress sa) throws IOException {

	// Net.connect 方法调用系统的 connect 方法
    int n = Net.connect(family,
                        fd,
                        isa.getAddress(),
                        isa.getPort());

    // Connection succeeded; disallow further invocation
    // 此时 channel 的状态，变成： ST_CONNECTED 表示已经连接状态。
    state = ST_CONNECTED;

    return this;
}
```

UDP socket 进行 connect 的作用：

> If the socket sockfd is of type SOCK_DGRAM then addr is the address to which datagrams are sent by default, and the only address from which datagrams are received.

所以 connect 只是记录地址，这个地址将进行默认的收发操作。而不会像 SOCK_STREAM（TCP）类型的 socket , 会向这个 connect 的地址进行握手连接。

**connect 方法可以被多次调用:**

> Generally, connection-based protocol sockets may successfully connect() only once; **connectionless protocol sockets may use connect() multiple times to change their association.**

`getRemoteAddress` 方法就可以获取到通过connect方法调用中提供的远程通信的socket的地址。

## 4. DatagramChannel 的 IO

### send & receive

BSD Socket 提供的接口。

* send

	如果，当前 socket 已经处于 ST_CONNECTED 状态，则 send 方法调用 write 方法来实现。
	
	如果 socket 没有 connect, 则最终调用系统的 `sendto` 方法进行数据发送。

* receive

	调用系统的 `recvfrom` 方法获取数据。

### write & read

channel 抽象中提供的接口。调用 write 和 read API 的时候，通道必须处于连接状态，否则，会出现 `NotYetConnectedException` 异常。

在 windows 平台上， write 和 read 最终使用 `WSASend` 和 `WSARecv` 来实现

在 linux 平台上，write 和 read 最终使用 `recv` 和 `send` 来实现。

### 两种IO方式的比较

使用 write 和 read 进行 channel IO 时，chennel 自身必须是已经 connected 过的。否则，会出现 `NotYetConnectedException` 异常。所以，即使是Server端的 DatagramChannel 也需要进行 connect.

## 杂项

### jdk在不同平台的 native 方法的共享库的位置

在 windows 平台下其，位置是 %JAVA_HOME%/jre/bin，其中有 net.dll , nio.dll

在 linxu 平台下，其位置是 %JAVA_HOME%/jre/lib/amd64，其中有 libnet.so libnio.so 文件。

### 如何查看 so 文件中导出的符号

``` bash
nm -D libnet.so
objdump -tT libnet.so
```

### 如果查看一个进程加载了哪些共享库

``` bash
## 获得进程IO
ps -ef |  grep java

## 查看进程ID 为 pid 加载的共享库
pmap <pid>

pmap <pid> | grep amd
```

