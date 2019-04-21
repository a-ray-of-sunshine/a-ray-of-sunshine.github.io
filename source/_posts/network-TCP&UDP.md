---
title: TCP&UDP
date: 2016-10-24 17:22:39
---

## TCP&UDP

### 格式

```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |Version|  IHL  |Type of Service|          Total Length         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |         Identification        |Flags|      Fragment Offset    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Time to Live |    Protocol   |         Header Checksum       |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                       Source Address                          |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Destination Address                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                    Example Internet Datagram Header
                    
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |          Source Port          |       Destination Port        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                        Sequence Number                        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Acknowledgment Number                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |  Data |           |U|A|P|R|S|F|                               |
   | Offset| Reserved  |R|C|S|S|Y|I|            Window             |
   |       |           |G|K|H|T|N|N|                               |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           Checksum            |         Urgent Pointer        |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                    Options                    |    Padding    |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                             data                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 						TCP Header Format
```

## socket 编程

### TCP 建立链接

|source|destination|protocol|length|info|
|-----:|:---------:|:------:|:----:|:--:|
|127.0.0.1|127.0.0.1|TCP|108|53226→1234 [SYN] Seq=0 Win=8192 Len=0 MSS=65495 WS=256 SACK_PERM=1
|127.0.0.1|127.0.0.1|TCP|108|1234→53226 [SYN, ACK] Seq=0 Ack=1 Win=8192 Len=0 MSS=65495 WS=256 SACK_PERM=1
|127.0.0.1|127.0.0.1|TCP|84|53226→1234 [ACK] Seq=1 Ack=1 Win=8192 Len=0

建立TCP接连，其实就是进行上面的三次握手，也就是序号同步。

### 关闭 TCP 链接

|source|destination|protocol|length|info|
|-----:|:---------:|:------:|:----:|:--:|
|127.0.0.1|127.0.0.1|TCP|84|50514→1234 [FIN, ACK] Seq=3428547810 Ack=490876652 Win=8192 Len=0|
|127.0.0.1|127.0.0.1|TCP|84|1234→50514 [ACK] Seq=490876652 Ack=3428547811 Win=7936 Len=0|
|127.0.0.1|127.0.0.1|TCP|84|1234→50514 [FIN, ACK] Seq=490876652 Ack=3428547811 Win=7936 Len=0|
|127.0.0.1|127.0.0.1|TCP|84|50514→1234 [ACK] Seq=3428547811 Ack=490876653 Win=8192 Len=0|

### TCP 链接

|NO.|src port|dest port|Control Bits|Seq|Ack|Len|Data|
|:-:|:------:|:-------:|:----------:|:-:|:-:|:-:|:--:|
|0|50514|1234|[SYN]|x|0|0|N/A|
|1|1234|50514|[SYN, ACK]|y|x+1|0|N/A|
|2|50514|1234|[ACK]|x+1|y+1|0|N/A|
|3|50514|1234|[PSH, ACK]|x+1|y+1|6|abc123|
|4|1234|50514|[ACK]|y+1|x+1+6|0|N/A|
|5|50514|1234|[PSH, ACK]|x+1+6|y+1|4|ef45|
|6|1234|50514|[ACK]|y+1|x+1+6+4|0|N/A|
|7|50514|1234|[FIN, ACK]|x+1+6+4|y+1|0|N/A|
|8|1234|50514|[ACK]|y+1|x+1+6+4+1|0|N/A|
|9|1234|50514|[FIN, ACK]|y+1|x+1+6+4+1|0|N/A|
|10|50514|1234|[ACK]|x+1+6+4+1|y+1+1|0|N/A|

* 建立链接

	[0-2]的过程，就是建立一个链接的过程。
	
* 传输数据

	[3-4] 和 [5-6] 是两个传输数据的过程

* 断开链接

	[7-10]的过程，就是断开链接的过程
	
在传输数据的过程中，如果出现PSH标志，则表示应该立即将数据从TCP的缓存传输到数据接收方的用户空间中，这使得 `recv` 函数可以返回。

当传输的数据大于 tcp 分包的大小，则数据将会分成几个不同的包来传输，直到最后一个数据包传输时会，设置 PSH 标志位。此时 Client 的 TCP 就可以将前面缓存的数据组成一个数据包，给用户空间使用。


## 总结

TCP基于连接的意思是：对于实现TCP通信功能来说，操作系统（内核）会维护那些已经通过三次握手所建立起来的连接。维护连接，说白了就是维护状态，以及分配工作缓存等，状态信息，例如：TCP连接通信时的 SEQ 序号，TCP通信的两端应该都会维护，同时为这个连接，提供一个缓存区，用来存储接受到的数据等。

所以只要不关闭一个TCP连接，则两端可以随时发起通信，就好似，有一个连接的数据通道一样，所以称为连接。

## 网络通信中数据的正确性和完整性

### 正确性

对数据包添加 CRC Checksum 来保证数据的正确性。 

### 完整性

TCP 如何确保数据的完整性，丢包时如何处理。

## TCP 连接中可能会出现的异常

* 连接被重置

	当客户端进行被直接kill掉时，客户端所在的操作系统会给服务器端发生一个含有 [RST] 标志位的 tcp 包，用来通知服务器端，客户端的 tcp 连接被异常关闭了。

* TCP传输延时

	[40毫秒延迟与 TCP_NODELAY](http://jerrypeng.me/2013/08/mythical-40ms-delay-and-tcp-nodelay/)
	
* rfc1122 & rfc1123

	rfc1122 和 rfc1123 这两个rfc对已经存在的tcp等规范进行了进一步的说明和讨论。其中就是关于 PSH 和 ACK 标志的作用及使用的讨论。

## socket 的生命周期

	TCP/IP Socket in java中的第 6.4 节 TCP Socket Life Cycle 讲述了关闭socket的生命周期。

## 网络编程的另一套接口

其实对于 socket 来说，它和 TCP/IP 网络并没有直接的关系，socket 只是一层用户空间的抽象，用来进行进程间通信。而进程间通信的接口，其实并不只是Socket这一种抽象，还有一个 [X/Open Transport Interface](https://en.wikipedia.org/wiki/X/Open_Transport_Interface) 这个接口，也是对通过网络进行进程间的通信做了抽象。使用这个 XTI/TLI 接口，也可以进行网络编程。

如何使用 XTI/TLI 进行编程，参考：[Programming With XTI and TLI](https://docs.oracle.com/cd/E53394_01/html/E54815/tli-33281.html)

[X/Open Transport Interface (XTI)](http://www.openss7.org/xti_man.html)

[Berkeley sockets](https://en.wikipedia.org/wiki/Berkeley_sockets) 这个页面中有如下的描述：

> The STREAMS-based Transport Layer Interface (TLI) API offers an alternative to the socket API. However, recent systems that provide the TLI API also provide the Berkeley socket API.

## UDP

```
  				 0      7 8     15 16    23 24    31
                 +--------+--------+--------+--------+
                 |     Source      |   Destination   |
                 |      Port       |      Port       |
                 +--------+--------+--------+--------+
                 |                 |                 |
                 |     Length      |    Checksum     |
                 +--------+--------+--------+--------+
                 |
                 |          data octets ...
                 +---------------- ...
                      User Datagram Header Format
```

* Source Port

	数据包源端口，是可选字段，如果这个字段未被使用，则插入0代替。如果这个字段被使用，通常，其意义是发送Socket的端口。
	
* Destination Port

	通信的目标端口。
	
* Length

	表示 UDP 头部和发送的数据（data）的总长度（以字节为单位）

* Checksum

	the IP header, the UDP header, and the data 三个数据的检验和。

### UDP的关闭

UDP 本身就是无连接的，所以通信双方，如果有一方断开Socket，另一方并不会得到通知。所以通信双方的 Socket 关闭应该由，实现通信的应用来协商关闭，而不是由Socket来实现关闭。例如关闭 DatagramSocket 的一方，可以在关闭之前显示发送一个 DatagramPacket，这个　Package　中包含一个双方约定好的控制标志位。通信双方只要检测到这个标志位，就可以安全的释放　Socket　资源了。

同样地，由于 UDP 是无连接的，也就是无状态的通信，所以对于通信双方，即使断开，另一方的 receive 或 send 方法的调用也不会出现异常。

### UDP的数据传输

UDP直接发送数据包，没有建立连接的过程（三次握手），也没有接收到数据的一方也没有确认数据的过程（发送ACK包），因为没有建立连接的过程，自然也没有断开的过程（四次挥手）。

DatagramSocket 的 send 方法直接封装数据到一个IP数据包。

如果发送端首先向接受端发送数据，而接收端没有还没有监听那个端口，则 TCP/IP 协议栈会将接收到的数据进行缓存，接收端启动监听，调用 receive 方法就可以接收到之前发送过来的数据。缓存的时间应该是由TCP/IP协议栈的实现决定的。



### UDP的应用场景

## 数据链路层



## 参考

* [rfc793-tcp](https://tools.ietf.org/html/rfc793)
* [rfc768-udp](https://tools.ietf.org/html/rfc768)
* [rfc791-ip](https://tools.ietf.org/html/rfc791)
* [rfc2616-http](https://tools.ietf.org/html/rfc2616)
* [rfc894-IP Datagrams over Ethernet Networks](https://tools.ietf.org/html/rfc894)
* [Ethernet_frame](https://en.wikipedia.org/wiki/Ethernet_frame)
* [EtherType](https://en.wikipedia.org/wiki/EtherType)
* [简析TCP的三次握手与四次分手](http://www.jellythink.com/archives/705)
* [TCP协议可靠性数据传输实现原理分析](http://blog.csdn.net/chexlong/article/details/6123087)
* [TCP 协议如何保证可靠传输](http://www.cnblogs.com/wzyxidian/p/5896456.html)
* [TCP 的那些事儿（上）](http://coolshell.cn/articles/11564.html)
* [TCP 的那些事儿（下）](http://coolshell.cn/articles/11609.html)
* [Windows Socket 最大连接数](http://www.cnblogs.com/zwq194/archive/2012/12/14/2817673.html)
* [单机最大tcp连接数](http://wanshi.iteye.com/blog/1256282)
* [Maximum transmission unit](https://en.wikipedia.org/wiki/Maximum_transmission_unit)
* [Maximum segment size](https://en.wikipedia.org/wiki/Maximum_segment_size)
* [Transmission Control Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
* [TCP分包 粘包 MTU 和MSS之间的关系分析](http://blog.csdn.net/yusiguyuan/article/details/20479369)
* [TCP Immediate Data Transfer: "Push" Function](http://www.tcpipguide.com/free/t_TCPImmediateDataTransferPushFunction-2.htm)
* [Push Bit Interpretation](https://msdn.microsoft.com/en-us/library/aa454066.aspx)
* [Linux 4.4.0内核源码分析——TCP实现](https://github.com/fzyz999/Analysis_TCP_in_Linux)
* [linux内核网络协议栈源码(版本为2.6.35)分析注释](https://github.com/run/kernel-tcp)
* [Nagle's algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm)
* [TCP delayed acknowledgment](https://en.wikipedia.org/wiki/TCP_delayed_acknowledgment)
* [rfc1122-Requirements for Internet Hosts](https://tools.ietf.org/html/rfc1122)
* [深入浅出TCP之listen](http://linuxgp.blog.51cto.com/1708668/1302512)
* [深入剖析 Socket——TCP 套接字的生命周期](http://wiki.jikexueyuan.com/project/java-socket/socket-tcp-bean.html)
* [tcp udp数据包长限制](https://github.com/fmaj7/HoneyBadger/wiki/tcp-udp%E6%95%B0%E6%8D%AE%E5%8C%85%E9%95%BF%E9%99%90%E5%88%B6)