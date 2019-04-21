---
title: Channel总结
date: 2016-12-1 10:42:41
---

## 0. Channel 接口

> A nexus for I/O operations. 
>
> 代表 I/O 操作的联结(nexus: 联结，联系，关系)
>
> A channel represents an open connection to an entity such as a hardware device, a file, a network socket, or a program component that is capable of performing one or more distinct I/O operations, for example reading or writing. 
>
> channel 代表和一个entity打开的连接。entity 表示硬件设备，文件，网络socket,或者是一个可以执行I/O操作，例如 read & write 操作的程序组件。
> 
> A channel is either open or closed. A channel is open upon creation, and once closed it remains closed. Once a channel is closed, any attempt to invoke an I/O operation upon it will cause a ClosedChannelException to be thrown. Whether or not a channel is open may be tested by invoking its isOpen method. 
>
> Channel 表示一种连接，所以其状态有打开和关闭两种。当前 channel 被创建的时候处于 Open 状态，一旦关闭之后就处于 closed 状态。
> 
> 一旦 channel 被关闭，所有在其上的 I/O 操作将都会出现 ClosedChannelException
> 
> Channels are, in general, intended to be safe for multithreaded access as described in the specifications of the interfaces and classes that extend and implement this interface.
> 
> channel 通常上来说是线程安全的。
> 
> 具体看实现这个接口的子类。

**关于某个特定的Channel，其对并发访问的支持参考对应的 java doc**

## 1. Channel 的异步关闭

继承自 AbstractInterruptibleChannel 的类都可以进行异步关闭和中断。

其实应该说是可以进行**安全的异步关闭和中断**。

安全体现在两点：

1. Channel占用的系统资源被正常的回收

	AbstractInterruptibleChannel.implCloseChannel 这个方法会被调用，这个方法用来执行资源清理和回收。
	
2. 客户代码会得到正确的通知

	一旦 Channel 被关闭，再次向 Channel 中进行 read & write 操作时，read 和 write 方法会抛出 `ClosedChannelException`。
	
	客户代码可以通过捕获这个异常，从而实现对异步关闭进行合理的业务处理。从而增强代码的健壮性。
	
其实，这两点特性，可以和 java.net.Socket 进行比较。Socket 的IO是，基于 Inputstream & Outputstream 。基于 Stream 的IO，只会抛出 `IOException`,无法得到更多的异常信息。而基于 Channel 的IO，其 read & write 方法都会抛出更加明确的异常信息，便于客户代码进行进一步处理。

## 2. Channel 的并发

这几种 Channel 都支持并发读写。

### 2.1 FileChannel

``` java
public class FileChannelImpl extends FileChannel {
    // Lock for operations involving position and size
    private final Object positionLock = new Object();
    
    public int read(ByteBuffer dst) throws IOException {

        synchronized (positionLock) {
			// ...
        }
    }
    
    public int write(ByteBuffer src) throws IOException {

        synchronized (positionLock) {
			// ...
        }
    }
}
```

读写等会改变Channel状态（文件的offset）的方法，使用 `positionLock` 来进行保护。

### 2.2 SocketChannel

``` java
class SocketChannelImpl extends SocketChannel implements SelChImpl{
    // Lock held by current reading or connecting thread
    private final Object readLock = new Object();

    // Lock held by current writing or connecting thread
    private final Object writeLock = new Object();
    
    // ...
}
```

当执行 `read` 操作的时候使用 `readLock`

当执行 `write` 操作的时候使用 `writeLock`

由此可见 Channel 是支持并发的。

### 2.3 DatagramChannel

和 SocketChannel 类似，在进行读写操作时分别加锁。

## 3. 基于 Channel 和 基于 Stream 的 I/O 的对比