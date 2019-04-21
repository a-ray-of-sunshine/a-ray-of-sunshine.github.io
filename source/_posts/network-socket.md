---
title: socket
date: 2016-10-24 21:07:11
---


java.net 包中的分类

* Address
* Socket
* DatagramSocket
* HTTP	

## Address

1. InetAddress 

	* InetAddress 
		* Inet4Address
		* Inet6Address

2. SocketAddress

	* SocketAddress
		* InetSocketAddress

这两个不同的地址类，其实是对于不同领域（domain）的地址抽象，对于 InetAddress 它代表是 基于 IP 协议的地址：

> This class(InetAddress) represents an Internet Protocol (IP) address. 

所以按照IPv4和IPv6分别有不同的协议的地址类型。所以 InetAddress 是对 IP 网络通信相关协议的地址。

SocketAddress 则是 Socket 编程模型中的地址。Socket 模型是操作系统所提供的对TCP/IP协议栈的编程接口。通过 Socket 编程接口，可以使用 TCP/IP 协议栈所提供的基于 TCP 和 UDP 的两种方式进行主机间通信。所以 Socket 编程使用的地址就是 TCP/UDP 通信时，所要使用的地址。所以就有了 SocketAddress 的抽象。

所以 SocketAddress 表示的是 网络通信接口--Socket编程模型 的地址抽象。它自身并没有和任何通信协议相关联。

> This class(SocketAddress) represents a Socket Address with no protocol attachment. As an abstract class, it is meant to be subclassed with a specific, protocol dependent, implementation. 

从这两个类的设计来看，虽然最终是表示网络通信时的地址，但是还是按照不同领域进行抽象。这样有利于扩展和使用。

例如：假设 现在 Socket 支持 基于 ATM 网络 的通信机制，而在创建 Socket 的时候使用了 SocketAddress 来表示，Socket 地址。此时就可以直接 抽象出一个 ATMAddress 类，然后再子类化 SocketAddress 成 ATMSocketAddress。其中 ATMSocketAddress 持有一个 ATMAddress 对象。这样 ATMSocketAddress 也是 SocketAddress  类型的，所以对于 Socket这边的抽象，就不需要做任何变化就可以支持 ATM 网络的地址了。

## socket 实现

java 实现的 socket 是对系统所提供的 socket 功能的封装。

AbstractPlainSocketImpl 在不同平台上，实现了子类：PlainSocketImpl。 下面分析 PlainSocketImpl 类在不同平台上的实现。

### linux 中的实现

最终实现由JVM提供。

openjdk\hotspot\src\share\vm\prims\jvm.cpp

和 openjdk\hotspot\src\os\linux\vm\os_linux.inline.hpp 中提供了实现源码。

这两层实现的功能就是直接调用系统提供的 socket API 进行。

### windows 中的实现

由 JDK 提供实现。

windos 实现 PlainSocketImpl 是并不是由JVM来实现，而是由 TwoStacksPlainSocketImpl 这个类来实现。jdk\src\windows\native\java\net\TwoStacksPlainSocketImpl.c 中实现了 socket API的包装调用。

而 java 这边主要的逻辑是在 AbstractPlainSocketImpl 类中实现。

当创建一个Socket的时候，其内部持有一个 AbstractPlainSocketImpl 对象，socket 实现相关的功能将都使用这个对象来实现。同时创建一个 Socket 对象，其内部会调用 `socket` 这个方法，创建系统中的socket. 最终 socket 将的描述符，被保存在。AbstractPlainSocketImpl的父类 SocketImpl 类的 fd 字段中。后续的 socket 调用将使用这个 fd 来标识这个 socket.

### ServerSocket & Socket

ServerSocket 中提供 port 的构造函数，默认执行 bind 和 listen 操作。

Socket 中提供了 SocketAddress 的构造函数，默认会进行 conntion 操作。

**对于已经 bound 或者 connection 的 socket, 再次调用 bind 和 connect 将导致异常。**

## Socket I/O

通过 SocketOutputStream 和 SocketInputStream 这两个类来实现，这两个类都继承自 java.io.FileInputStream 。所以直接使用文件读写的API来进行 socket 的读写。

相当于 Socket I/O 其实就是对应于文件I/O。

**I/O 流的关闭，将导致socket的关闭。**

``` java
public void close() throws IOException {
    // Prevent recursion. See BugId 4484411
    if (closing)
        return;
    closing = true;
    if (socket != null) {
        if (!socket.isClosed())
            socket.close();
    } else
        impl.close();
    closing = false;
}
```

## DatagramSocket

### linux 上的实现

由 java.net.PlainDatagramSocketImpl 来实现

### windows 上的实现

由 java.net.TwoStacksPlainDatagramSocketImpl 来实现。

### DatagramSocket 的创建过程

下面给出创建过程的核心流程。

1. Creates a datagram socket

	``` java
    // AbstractPlainDatagramSocketImpl
    // fd 对象是 AbstractPlainDatagramSocketImpl 父类的一个对象
    // 用来存储 socket 文件描述符
    protected synchronized void create() throws SocketException {
        fd = new FileDescriptor();
        ResourceManager.beforeUdpCreate();
        try {
            datagramSocketCreate();
        } catch (SocketException ioe) {
            ResourceManager.afterUdpClose();
            fd = null;
            throw ioe;
        }
    }
	```

2. datagramSocketCreate

	这个方法用来初始化 fd 对象。核心代码如下：
	
	``` c
	// PlainDatagramSocketImpl.c
	JNIEXPORT void JNICALL Java_java_net_PlainDatagramSocketImpl_datagramSocketCreate(JNIEnv *env, jobject this) {
	// 获得上面创建的 fd 对象
    jobject fdObj = (*env)->GetObjectField(env, this, pdsi_fdID);
    
    // JVM_Socket 调用 系统底层的 socket 函数创建
    // 一个类型为 SOCK_DGRAM 的 socket.
    // JVM_Socket 的实现由 JVM 提供，
    // 其实现就是： socket(domain, type, protocol);
    fd = JVM_Socket(domain, SOCK_DGRAM, 0);
    
    // 将 fdObj 对象初始化为上面创建的系统创建返回的 fd
    (*env)->SetIntField(env, fdObj, IO_fd_fdID, fd);
    
	}
	```
	
	到此为至，DatagramSocket 对象的 fd 字段被初始化成一个 SOCK_DGRAM 类型的 socket 的文件描述符。
	
3. bind

	``` c
	JNIEXPORT void JNICALL
Java_java_net_PlainDatagramSocketImpl_bind0(JNIEnv *env, jobject this, jint localport, jobject iaObj) {
    /* fdObj is the FileDescriptor field on this */
    jobject fdObj = (*env)->GetObjectField(env, this, pdsi_fdID);
    
    // 初始化一个 struct sockaddr_in 结构。使用指定的端口
    SOCKADDR him;
    NET_InetAddressToSockaddr(env, iaObj, localport, (struct sockaddr *)&him, &len, JNI_TRUE);

	// 进行 bind 操作。
	// 其内部调用 bind(fd, him, len); 执行 bind 操作。
    NET_Bind(fd, (struct sockaddr *)&him, len);
    
    // 初始化 localPort 字段
    // DatagramSocketImpl.localPort
    // 表示当前创建的这个socket绑定的端口
    (*env)->SetIntField(env, this, pdsi_localPortID, localport);
    }
	```

### DatagramSocket 数据发送和接收

最终调用 PlainDatagramSocketImpl.c 中的 Java_java_net_PlainDatagramSocketImpl_send 方法来实现。

在这里将调用 `sendto` 方法进行数据的发送，同时如果这个 socket 已经是 connected 过的。则 DatagramPacket 中提供的 InetAddress 和 Port 不会被使用。而是直接使用之前 connect 时提供的 InetAddress 和 Port。

虽然 UDP 是无连接的协议，但是，创建一个UDP socket之后，仍然可以调用 connect 函数（DatagramSocket.connect），不过，调用 connect 和 TCP 调用 connect 是不同的，内核不会进行像TCP那样，进行3次握手。只是记录两个信息。

摘自 connect 的 man-page:

> **If the socket sockfd is of type SOCK_DGRAM, then addr is the address to which datagrams are sent by default, and the only address from which datagrams are received.** If the socket is of type SOCK_STREAM or SOCK_SEQPACKET, this call attempts to make a connection to the socket that is bound to the address specified by addr.
>
> Generally, connection-based protocol sockets may successfully connect() only once; connectionless protocol sockets may use connect() multiple times to change their association.

DatagramSocket.receive 方法的实现调用 `recvfrom`

### DatagramSocket 的 connect 方法

DatagramSocket 可以进行 connect 操作。这个也是由操作系统的socket API所提供的。DatagramSocket 如果执行了 connect 操作，则调用 send 方法的参数DatagramPacket 中就不会指定	InetAddress 和 Port，默认将使用上面的 connect 的参数。

## Socket & TLI/XTI

socket 是操作系统提供的一种访问系统网络功能的接口。操作系统使用网络协议栈中的功能除了 BSD Socket, 还可以使用 TLI(Transport Layer Interface) 进行网络编程，TLI 后来被标准化为 XTI。BSD Socket 被标准化之后是 POSIX socket API。

> Berkeley sockets is an application programming interface (API) for Internet sockets and Unix domain sockets. It originated with the 4.2BSD Unix released in 1983.
>
> the Transport Layer Interface (TLI) was the networking API provided by AT&T UNIX System V Release 3 (SVR3) in 1987.

socket 于 1983 年在 4.2BSD Unix (Open Source) 上实现

TLI 于1987 AT&T UNIX SVR3 (Closed Source) 上实现，这个接口，最终还是被 SVR4-derived operating systems 的操作系统所支持，例如：Solaris。Solaris 同时支持 Socket 和 TLI/XTI 这两种接口进行网络编程。

## 参考

1. [Linux内核Socket实现](http://blog.chinaunix.net/uid-20788636-id-4408261.html)
2. [socket源码分析](http://blog.csdn.net/column/details/socketdive.html)
3. [linux网络协议栈内核分析](http://blog.csdn.net/xaiojiang/article/details/51584533)
4. [Linux网络协议栈-系列文章](http://www.cnblogs.com/hustcat/category/191424.html)
5. [Transport Layer Interface](https://en.wikipedia.org/wiki/Transport_Layer_Interface)
6. [X/Open Transport Interface](https://en.wikipedia.org/wiki/X/Open_Transport_Interface)
7. [Berkeley sockets](https://en.wikipedia.org/wiki/Berkeley_sockets)