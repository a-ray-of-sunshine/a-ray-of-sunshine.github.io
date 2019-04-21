---
title: IO-InputStream
date: 2016-9-10 9:58:08
---

## read

抽象类，提供了一个 read 方法来读取字节。这个方法是一个抽象方法
交由具体的子类来实现。

``` java
public abstract class InputStream implements Closeable {
	// 对象要实现 InputStream 的类，
	// 需要实现下面的方法。
	public abstract int read() throws IOException
}

```

>Reads the **next byte of data** from the input stream. The value byte is returned as an int in the range 0 to 255. 
>
>If no byte is available because the end of the stream has been reached, the value -1 is returned. 
>
>This method blocks until input data is available, the end of the stream is detected, or an exception is thrown.

对于 read 方法来说，每调用一次，其返回数据中的下一个字节，所以就像流水一样。数据从某个地方（数据的源头，例如：磁盘文件，内存，java序列化后的对象）不断地调用 read 方法，将数据取出来：

```
+---------+
|         |           
|  data   |==========>>(read)====> 1bye
|         |
+---------+
```

这个函数会一直阻塞，直到：

* input data is available
* the end of the stream is detected
* an exception is thrown

>If no byte is available because the end of the stream has been reached, the value -1 is returned.

当 read 到数据源的 EOF 时，这个函数将返回 -1.

由于 read 方法并没有任何参数，所以，对于数据源的数据读取，只能是顺序的读取，其数据源内部的下一个返回字节的offset则由具体的实现来维护。例如 FileInputStream中，打开的文件，其默认 offset 就是第一个字节，随着不断 read , offset 由 ReadFile （win32） 和 read (linux) 的实现来维护。

同时，从 InputStream 这个抽象来说，其设计的目的就是提供数据的流式访问，而不是随机访问。所以，InputStream 中并没有提供类似 lseek 等，可以直接设置，下次读取数据位置的方法。

## mark

虽然，InputStream 是流式（顺序）读取数据，但是其仍然，提供了一些可以跳跃式读取数据的方法

``` java

public long skip(long n) throws IOException{
	//...
	//...
}

public boolean markSupported() {
    return false;
}

public synchronized void mark(int readlimit) {}

public synchronized void reset() throws IOException {
    throw new IOException("mark/reset not supported");
}
```

* long skip(long n)

	Skips over and discards n bytes of data from this input stream. 从 stream 的当前位置起 skip n 个字节，所以在 skip 调用之后，stream 的 offset 自然也就偏移了 n 个字节，所以 read 调用，将返回 skip 之后的字节。

* boolean markSupported()

	Tests if this input stream supports the mark and reset methods. 如果 stream 支持 mark 和 reset 操作，则这个方法返回 true, 否则返回 false.

* void mark(int readlimit)
	
	Marks the current position in this input stream.当这个方法被调用的时候，mark 会将当前的 stream 的读取的 position 记录下来，注意，**mark的stream的位置和参数 readlimit 没有任何关系。**

	mark 下来的当前位置是有一个有效期的，在这个有效期内，调用 reset 方法可以恢复到 mark 的 position,否则，reset 将抛出异常。这个有效期就是 mark 的参数 readlimit:

	The readlimit arguments tells this input stream to allow that many bytes to be read before the mark position gets invalidated. 

	也就是说在 mark 之后，如果有 readlimit 个字节，已经被读取，那么 mark 调用记录下来的那个 position 将变成 invalidated. 所以 reset 可能抛出 IOException

* void reset()

	Repositions this stream to the position at the time the mark method was last called on this input stream.

注意上面的描述是来自 InputStream 的文档，也就是说这相当于接口的约定。而具体的行为，则需要参考具体的实现。

例如：InputStream 的一个实现类 ByteArrayInputStream 是支持 mark & reset 操作的，但是其在实现 mark 方法时，并没有使用参数 readlimit. 也就是说这个参数并没有干预到 reset 方法的实现。而在 InputStream 对接口的 reset 的描述中，readlimit 是起作用的。

所以，对一个 stream 的 mark & reset 操作，一定要参考其具体的实现的方式。而不是 InputStream 中的描述。

InputStream 对于 mark & reset 的实现是：

* markSupported return false
* mark  does nothing
* reset does nothing except throw an IOException.

所以，如果InputStream的子类没有 override 这几个方法，则这个stream 就不支持 mark & reset.

## InputStream 子类

InputStream 子类， 以数据源为分类有

字节数组，文件，java对象序列化文件流，管道，等等。

### ByteArrayInputStream

#### 构造过程

``` java
public
class ByteArrayInputStream extends InputStream {
	protected byte buf[];
    protected int pos;
    protected int mark = 0;
	
	// The index one greater than the last valid 
	// character in the input stream buffer.
	// count 并不是流的大小（size）或者说长度（length）
	// 而是 buf 数组中最后一个属于当前流的字节的索引+1.
    protected int count;
```

```
            pos, mark                             count 
                ↓                                   ↓
          +---+---+---+---+---+---+---+---+---+---+---+---+
bytes ---→| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |10 |11 |
          +---+---+---+---+---+---+---+---+---+---+---+---+
              ↑                                   ↑
              |___________________________________|
                                  ↑
                                 buf
```

``` java
// 当使用下面的构造函数来构造，ByteArrayInputStream 时
// 其生成的内部结构如上图所示

InputStream bais = new ByteArrayInputStream(bytes, 1, 9);

// 上面的调用使用 bytes[1] <---> bytes[9] 作为 
// ByteArrayInputStream 内部的 buf. read 方法的实现，就是以 buf 作为
// 数据源。由此可知，bytes[0],bytes[10],bytes[11] 这三个字节，对于
// bais 这个 stream 来说是不可见的。

public ByteArrayInputStream(byte buf[], int offset, int length) {
    this.buf = buf;
    this.pos = offset;
    this.count = Math.min(offset + length, buf.length);
    this.mark = offset;
}
```

#### mark & reset

这个类支持 mark & reset

``` java
public boolean markSupported() {
    return true;
}

public void mark(int readAheadLimit) {
	// 记录当前位置。
    mark = pos;

	// readAheadLimit 参数并未使用，所以不存在
	// mark 失效的情况。
}

public synchronized void reset() {
	// 将 stream 位置重置为最近一次 mark 调用时
	// 设置的 mark.
    pos = mark;
}
```

#### notes

1. close

	调用 close 方法并没有任何作用。close 方法是空实现。

2. 内部直接引用的，参数传递过来的数组。如果这个字节数组在其它被修改了，对应 的流内部读取到的数据，也是被修改后的数据。

3. mark的位置，不会因为 readlimt 参数而失效，其内部没有使用 readlimt.

### SequenceInputStream

``` java
public class SequenceInputStream extends InputStream {
    
    // 存储串连起来的流
	Enumeration e;

	// 表示当前将要被使用或正在被使用的流
    InputStream in;

	// ...
}
```

这个类的作用是：将流串连起来。

```
               InputStream in;
                     ↓
+-------------+   +-------------+   +-------------+
|   stream1   |--→|   stream2   |--→|   stream3   |--→ ...
+-------------+   +-------------+   +-------------+
↑                                                        ↑
|________________________________________________________|
                          ↑
				   Enumeration e
```

内部使用 Enumeration 来存储这些流的引用。

将流串连起来的核心方法：

``` java
/**
 *  Continues reading in the next stream if an EOF is reached.
 *  将流串连起来
 */
final void nextStream() throws IOException {
    if (in != null) {
        in.close();
    }

    if (e.hasMoreElements()) {
        // 将流设置为下一个 elements.
        in = (InputStream) e.nextElement();
        if (in == null)
            throw new NullPointerException();
    } // 当 enumeration 中没有元素时，当前流自然就是 null 为了。
    else in = null;

}

// 以递归的形式来调用串连起来的流。
// 直到所有的流都读取完毕，直接返回 -1
public int read() throws IOException {
    if (in == null) {
		// 所有的流都已经读完
        return -1;
    }
    int c = in.read();
	// 说当前流已经读完，调整当前流。
    if (c == -1) {
		// 调整到下一个流
        nextStream();
		// 继续读取。
        return read();
    }
    return c;
}
```

### PipedInputStream

这个类和 PipedOutputStream 在不同的线程中配合使用。表示管道。

流从 PipedOutputStream 中流入，从 PipedInputStream 流出。

```
PipedInputStream 将在 out 处读取数据
	       out
	        ↓		
         +-----+-------------------------+-----+
     -1  |  0  |   buffer: 1024 bytes    | 1023|
         +-----+-------------------------+-----+
      ↑
     in
PipedOutputStream 将在 in 处写入数据

```

PipedInputStream 的初始状态。

* in   表示 PipedOutputStream.write 方法写入buffer中下一个字节的索引。
* out  表示 PipedInputStream.read 方法读取buffer中下一个字节的索引。

初始状态下：
buffer 中没有数据，所以 in = -1 表示流是空的。
out = 0, 表示可以向 buffer[0] 处写入数据。

此时，如果从 pipe 中 read 数据，则 reader 线程将进入循环等待状态。直到 in > 0,也就是 pipe 中有数据。如果从 pipe 中 write 数据，则 write 完成之后， out = 0, in = 1. 表示，此时若写入数据则应该写入 buffer[in] == buffer[1], 如果读取数据则应该读取，buffer[out] == buffer[0]。并且此后，将保持， out < in, 直到 out 读到 in 的位置，也就是 out == in, 此时，相当于 buffer 中的数据已经全部读完。所以，重置 in = -1.

#### PipedOutputStream.write

PipedOutputStream.write 调用 PipedInputStream.receive 完成向 Pipe 中写入数据的功能。

``` java
protected synchronized void receive(int b) throws IOException {
    checkStateForReceive();
    writeSide = Thread.currentThread();
	// 如果向 pipe 中不停地写入数据，而不读取数据，则
	// 则，in 直到 >= buffer.length, 此时， 而 buffer 的最后一个字节
	// 被写满之后，in 重置为 0, 则此时由于水消费数据，所以 out 也是 0, 
	// 就有 in == out, 此时，说明 buffer 中全部是有效的数据，且都没有
	// 被消费，所以没有多余的空间存储数据了，所以 进入 wait 状态。
	// 直到 in != out , 也就是 out 被移动了，数据被消费了。
	// write 操作才可以继续。
	// 在 write 过程中，如果出现 in == out, 则表明，写入数据，已经
	// 将 buffer 写满，所以要进行 wait.
    if (in == out)
        awaitSpace();

	// in < 0，表示此时 buffer 中没有数据。buffer 是空的。
    if (in < 0) {
        in = 0;
        out = 0;
    }

	// 向 buffer 的 in 处 写入数据，
	// 并将 in 步进1，表示下次写入数据的位置是 in+1.
    buffer[in++] = (byte)(b & 0xFF);
	// 当写入到 buffer 的最后一个位置的时候，重置写入位置
	// 开始循环写入到 buffer 的第一个字节处。
    if (in >= buffer.length) {
        in = 0;
    }
}

// awaitSpace 进行循环等待。
// 等待 1s ,检测是否有空间，
// 如果没有空间，则继续等待。
private void awaitSpace() throws IOException {
    while (in == out) {
        checkStateForReceive();

        /* full: kick any waiting readers */
		// 如果处在这个循环等待中，则表明当前 pipe 是有数据的
		// 所以调用 notifyAll, 将所有在这个 pipe 上 wait的
		// 的 reader 线程唤醒，使其能够，执行 read 操作。
		// 从而，达到释放空间的目的。
        notifyAll();
        try {
            wait(1000);
        } catch (InterruptedException ex) {
            throw new java.io.InterruptedIOException();
        }
    }
}

```

#### PipedInputStream.read

``` java
public synchronized int read()  throws IOException {
    if (!connected) {
        throw new IOException("Pipe not connected");
    } else if (closedByReader) {
        throw new IOException("Pipe closed");
    } else if (writeSide != null && !writeSide.isAlive()
               && !closedByWriter && (in < 0)) {
        throw new IOException("Write end dead");
    }

    readSide = Thread.currentThread();
    int trials = 2;

	// in < 0 表明，未曾有数据写入，此时 pipe 是空的
	// 所以进入循环等待中，直到 in >= 0 表明，已经有数据写入了
	// 可以读取了。
    while (in < 0) {
        if (closedByWriter) {
            /* closed by writer, return EOF */
            return -1;
        }
        if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
            throw new IOException("Pipe broken");
        }
        /* might be a writer waiting */
        // 在这个循环等待中，说明，目前 pipe 是空的，所以 writer 可以
	    // 执行，写入操作，所以 notifyAll， 将所所有在这个 pipe 上
		// wait 的 writer 唤醒。让其能够进行写入操作。
		// 从而，使得 pipe 中有了可以消费的数据。
        notifyAll();
        try {
            wait(1000);
        } catch (InterruptedException ex) {
            throw new java.io.InterruptedIOException();
        }
    }

	// 代码，执行到这里，表明， pipe中有数据了
	// 直接获取数据。
    int ret = buffer[out++] & 0xFF;

	// 当获取数据的位置，到达 buffer 的最后一个位置的时候
	// 重置 out 到 0, 表明，下次读取数据将从 buffer[0] 处开始。
    if (out >= buffer.length) {
        out = 0;
    }

	// 如果 pipe 中没有数据了 则 in == out
	// 表明，下次读和写的位置相同了，也就是说当前 pipe
	// 空了，所以：
	// 此时，重置 in = -1, 表示，buffer 已经空了。
	// 其实，这里也应该将 out 重置成 0 的。但是
	// 由于 write 处，有:
	//   if (in < 0) {
    //        in = 0;
    //        out = 0;
    //   }
	// 所以这里，只是将 in 重置。
    if (in == out) {
	// 在 read 过程中，如果出现: in == out 则表明，
	// 当前 buffer 已经读完，所以 buffer 是空的，则
	// 重置 in = -1
        /* now empty */
        in = -1;
    }

    return ret;
}

```

#### connect

PipedInputStream 和 PipedOutputStream 如果连接(connect)起来。

``` java
// PipedInputStream.connect
public void connect(PipedOutputStream src) throws IOException {
    src.connect(this);
}

// PipedOutputStream.connect
public synchronized void connect(PipedInputStream snk) throws IOException {
    if (snk == null) {
        throw new NullPointerException();
    } else if (sink != null || snk.connected) {
		// 如果当前 PipedOutputStream 已经和其它 
		// PipedInputStream 连接了(sink != null) 
		// 或者 PipedInputStream 已经被其它 PipedOutputStream
		// 连接了(snk.connected)，
		// 则直接抛出 IOException("Already connected")
		// 表明 PipedInputStream 和 PipedOutputStream 已经被
		// connect 过了。
        throw new IOException("Already connected");
    }
    sink = snk;
    snk.in = -1;
    snk.out = 0;
    snk.connected = true;
}
```

对于 每一个 PipedOutputStream 对象，内部持有一个 PipedInputStream 对象

``` java
private PipedInputStream sink;
```

表示这个 OutputStream 的输出的目的地。

由上面的 connect 的实现，可知，PipedOutputStream 和 PipedInputStream 永远是 1:1 的关系，一个 PipedOutputStream 对象只能和一个 PipedInputStream 进行 connect，反之亦然。否则，直接 IOException("Already connected");

所以，这里的 pipe 就像管道一样，一个出口，一个入口。而不是像消息系统，那样，可以有多个消费者（PipedInputStream），多个生产者（PipedOutputStream）。

#### 使用场景

>Typically, data is written to a PipedOutputStream object by one thread and data is read from the connected PipedInputStream by some other thread
>
>Typically, data is read from a PipedInputStream object by one thread and data is written to the corresponding PipedOutputStream by some other thread.

所以通常是使用两个线程来使用 PipedOutputStream 和 PipedInputStream。对于使用 PipedOutputStream 向 pipe 中写入数据的线程可以认为是生产者，使用 PipedInputStream 读取数据的线程可以认为是 消费者。

pipe 是线程安全的：
尽管 pipe 是 1：1 的 PipedOutputStream 和 PipedInputStream 互相连接，但是 PipedOutputStream 和 PipedInputStream 的对象，可以由多个线程，同时使用。也就是说 PipedOutputStream 和 PipedInputStream 两个对象，构成了一个线程安全的 pipe , 然后多个线程可以通过这个 pipe 来生产和消费数据。

```
+-----------------------------------------------------+
|                                                     |
| ThreadA                                             |
|         ↘                                Thread1   |
| ThreadB →    +----------------------+  ↗           |
|              |  ===== pipe =======> |  → Thread2    |
| ThreadC →    +----------------------+  ↘           |
|                                          Thread3    |
| ThreadD ↗                                          |
|                                                     |
+-----------------------------------------------------+
```

如上图所示，ThreadA， ThreadB，ThreadC，ThreadD 持有PipedOutputStream 可以向 pipe 中写入数据，而 Thread1，Thread2，Thread3 持有 PipedInputStream 可以从 pipe 中读取数据。

虽然，我们可以这样使用pipe, 但是这里存在一个问题:
例如 对于 Thread2 这个 reader ，如果它没有调用了 read 之后，没有调用 close 方法，然后线程 Thread2 终止了。则此时：

``` java
public synchronized int read()  throws IOException {
	// ...
	// readSide 就是 Thread2, 
	// 但是，之后 Thread2 终止，且没有调用 close
	// 则 closedByReader = false
    readSide = Thread.currentThread();
	// ...
}

// write 之前，进行状态判断
private void checkStateForReceive() throws IOException {
    if (!connected) { // connected = true
        throw new IOException("Pipe not connected");
    } else if (closedByWriter || closedByReader) { // closedByReader = false
        throw new IOException("Pipe closed");
    } else if (readSide != null && !readSide.isAlive()) {
		// 此时进入这个分支，由于 Thread2 已经终止，
		// 所以，当前 write 线程（ThreadD）
		// ThreadD 在 write 时出现 IOException("Read end dead")
		// 而，如果 Thread1， 进行正常的 read 操作，则是可以
		// 正常进行的，而线程 ThreadA 进行 write 操作，也就可以
		// 正常进行了，所以这对 ThreadD 来说，有点莫名其妙
		// ThreadD write 就失败了，而　ThreadA　却成功了。
        throw new IOException("Read end dead");
    }
}
```

其实，造成这种状况的根本，原因，在于:

``` java
public class PipedInputStream extends InputStream {
    boolean closedByWriter = false;
    volatile boolean closedByReader = false;
    boolean connected = false;

    Thread readSide;
    Thread writeSide;
	
	// ...
}
```

PipedInputStream 只跟踪了最近一次在 pipe 上执行 read 和 write 操作的线程。所以，也就无法，在其中一个 reader 或者， writer 出现问题的时候，通知，其它的线程。

再考虑，如果 Thread2 在终止之前调用了 close 方法呢，其实也还是有问题，Thread2 在没有通过其它 reader 的情况下，就直接将 pipe 给关闭了，这是否对其它正在工作的 reader 造成影响。所以，对于在同一个 pipe 上进行 read 和 write 的线程，维护 pipe 的状态，是一个大的问题。

所以，pipe 最好，还是一个线程进行 write 操作，另一个线程进行 read 操作，比较好。或者 一个线程 write , 并执行 close 操作，多个线程 read. 或者 多个线程 write, 然后一个线程 read, 由这个 read 线程来维护 pipe 的 close 操作.

对于 pipe 来说，是有吞吐量的，其吞吐量就是 创建 PipedInputStream 时的参数：

``` java
public PipedInputStream(PipedOutputStream src, int pipeSize) throws IOException {
     initPipe(pipeSize);
     connect(src);
}

private static final int DEFAULT_PIPE_SIZE = 1024;
```

就是 pipeSize， 默认是 1024 个字节。一旦，向这个 pipe 写入 1024 个字节之后，如果不进行消费，则 PipedOutputStream.write 操作将进入 wait 状态，直到，有数据被消费，write 才有空间，继续写入数据。

同理，如果 pipe 中没有数据，没有进行 write 操作，或者 pipe 中的 byte 被消费完毕，则 PipedInputStream.read 操作将进入 wait 状态。直到，有数据被写入，read 才可以读取到数据。

PipedInputStream 不支持，mark & reset.

#### pipe的关闭：

* PipedOutputStream 关闭：

	当 PipedOutputStream 调用 close 时，使得 PipedInputStream.closedByWriter = true.
	表明，pipe 被 writer 线程关闭了，也就是 write 线程调用了 PipedOutputStream.close 关闭了 pipe. 如果是这样，说明流，再不会产生数据了，则当前 PipedInputStream，直到数据读完之后。再次调用 read时, 将得到 -1，说明流已经读完毕了。

	对于 PipedOutputStream 如果在 close 之后，调用，write, 则会产生： IOException("Pipe closed");

* PipedInputStream 关闭

	设置 closedByReader = true; ，表示 pipe 被 read 线程关闭了。此时，如果 PipedInputStream 调用 read 将直接：IOException("Pipe closed"); 而不管流是否被读完。

	当然，对于 PipedOutputStream 写线程来说，其调用
	write 也会是 IOException("Pipe closed");