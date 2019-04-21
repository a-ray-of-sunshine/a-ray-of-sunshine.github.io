---
title: IO-FilterInputStream
date: 2016-9-12 12:22:48
---

> A FilterInputStream contains some other input stream, which it uses as its basic source of data, possibly transforming the data along the way or providing additional functionality. 
>
> Subclasses of FilterInputStream may further override some of these methods and may also provide additional methods and fields.

``` java
public
class FilterInputStream extends InputStream {

    protected volatile InputStream in;

    protected FilterInputStream(InputStream in) {
        this.in = in;
    }
}
```

子类将使用 in 作为底层的流，并实现 InputStream 的方法，实现的过程中，就可以在 底层 in 这个流调用前后添加装饰功能。

实际上，FilterInputStream 就是使用了装饰模式的典型。

## PushbackInputStream

PushbackInputStream 内部持有一个 push back buffer, 调用 unread 方法，将字节 push back 到这个 buffer中，此后，**read 操作 和 skip 操作，都优先 read 和 skip 这个 buffer 的数据**，如果，buffer 中的数据个数，满足了 read 和 skip 操作，则返回。否则，调用底层的流的 read 和 skip 将剩余的还需要 read 和 skip 的数据进行 read 和 skip.

```
// buf 的 length 由 PushbackInputStream 的构造函数
// size 参数决定。
public PushbackInputStream(InputStream in, int size) {
    super(in);
    if (size <= 0) {
        throw new IllegalArgumentException("size <= 0");
    }
    this.buf = new byte[size];
    this.pos = size;
}

 buf
  ↓
+----+----+----+----+----+----+----+----+----+----+----+----+
|    |    |    |    |    |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+----+----+----+----+
                                                          ↑
                                                         pos


                                           push back bytes
 buf                                             ↓
  ↓                                     ↓___________________↓ 
+----+----+----+----+----+----+----+----+----+----+----+----+
|    |    |    |    |    |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+----+----+----+----+
                                      ↑  
                                     pos

push back bytes: 表示从 unread 方法 push back 过来的字节数据。
在 buf 中：
0 - pos 表示 buffer 目前的存储容量，是空闲位置
pos - buf.length 的字节是 push back bytes.
```

这个类的核心在三个 push back 方法，unread.

``` java
public void unread(int b) throws IOException {
    ensureOpen();
	// 初始状态下 pos 是 buf 的 长度，代表 buffer 目前的容量
	// 随着 unread, pos 向右移动，直到 pos = 0
	// 表示 buf 中已经写满的数据，所以
	// push back buffer is full.
    if (pos == 0) {
        throw new IOException("Push back buffer is full");
    }

	// 写入数据
    buf[--pos] = (byte)b;
}

public void unread(byte[] b, int off, int len) throws IOException {
    ensureOpen();
	// pos 表示 push back buffer 当前的容量，
	// len > pos 表示待写入的数据的长度，已经超过，buffer 可以接受的容量
	// 所以，直接抛出异常，但是这个异常描述，准确：Push back buffer is full
	// 可能此时 pos = 3, 而 len = 5, 所以对于当前这个 unread 调用来说
	// 是没法写入数据的，因为空间不够，但不是 buffer is full
	// 事实上，buffer 还可以容纳 3 个字节。所以这个异常描述并不准确
    if (len > pos) {
        throw new IOException("Push back buffer is full");
    }

	// 写入数据
    pos -= len;
    System.arraycopy(b, off, buf, pos, len);
}

public void unread(byte[] b) throws IOException {
    unread(b, 0, b.length);
}
```

由于这个 push back buffer 的存在，对于这个类的 read 方法的实现来说，优先读取 push back buffer 中的数据，如果 push back 中没有数据，或者数据不够，则才从，底层的流 in 中读取数据。


### read

``` java
public int read() throws IOException {
    ensureOpen();
	// pos < buf.length 说明  push back buffer 中有数据
	// 则优先从，buffer 中读取数据
    if (pos < buf.length) {
        return buf[pos++] & 0xff;
    }

	// push back buffer 中没有数据，从底层的流中读取
    return super.read();
}

public int read(byte[] b, int off, int len) throws IOException {
    ensureOpen();
    if (b == null) {
        throw new NullPointerException();
    } else if (off < 0 || len < 0 || len > b.length - off) {
        throw new IndexOutOfBoundsException();
    } else if (len == 0) {
        return 0;
    }

	// 表示当前 push back buffer 中存储的字节的个数。
    int avail = buf.length - pos;
    if (avail > 0) {
		// len < avail 表示 buffer 中数据就满足此次的读取，
		// 如果 len >= avail 表示 buffer 中的数据还不够此次读取，
		// 所以先将 avail, 也就是 buffer 中所有数据，全部读完。
        if (len < avail) {
			// 调整 avail
            avail = len;
        }
		// 将 avail 个数据从，buf 读到 b 中
        System.arraycopy(buf, pos, b, off, avail);
		// buffer 中的 avail 个字节，已经读完，
		// 所以调整，pos
        pos += avail;
		// b 中已经有  avail 个字节，所以调整
        off += avail;
		// 已经向 b 中写入了 avail 个字节，所以
		// 还需要向 b 写入 len = len - avail 个字节。
        len -= avail;
    }

	// len 表示，还需要向 b 中写入的字节的个数
	// len > 0, 说明，上面从 buffer 中读取的数据并没有满足
	// b 需要的字节个数，所以，还需要从底层的 in 流中向 b 写入数据
    if (len > 0) {
		// 此时 len 表示 read 调用，返回的实际数据个数
        len = super.read(b, off, len);
        if (len == -1) {
			// len == -1 表示，没有底层流中获取到数据
			// 同时 avail 表示，从 buffer 中获取到的数据的个数
			// len == -1 && avail == 0 自然，表示
			// 没有获取到任何数据，所以按照 read 的语义约定
			// 返回 -1, 
			// 若 avail != 0,则说明从 buffer 中获取到数据的，所以直接返回
			// 返回 avail, 表示此次读取操作，实际取到的数据个数
            return avail == 0 ? -1 : avail;
        }
		// len != -1, 说明从底层的流中读到了数据
		// 所以总的数据个数是 avail + len
        return avail + len;
    }

	// 自然 len <= 0, 说明 buffer 中读取的 avail 个数据，已经满足此次
	// read 的需求，所以返回 avail, 表示此次读取的个数。
    return avail;
}
```

### skip

同 read 一样，skip 操作也是优先，skip 当前 buffer 中的字节，如果 当前 push back buffer 中数据，足够，skip, 则 返回。否则，调用，剩余的需要的skip的字节个数，调用底层流的 skip 方法。

### mark & reset

不支持： `throw new IOException("mark/reset not supported");`

### 应用场景

## BufferedInputStream

默认分配，8192（8KB）个字节的缓冲区，读取数据就从缓冲区中读取。

```

         markpos               pos
            ↓                   ↓
+----+----+----+----+----+----+----+----+----+----+----+----+
|    |    |    |    |    |    |    |    |    |    | NA | NA |
+----+----+----+----+----+----+----+----+----+----+----+----+
          ↑___________________↑___________________↑
               reset zone         validte zone
          ↑_______________________________________↑
                              ↑
                            count

buf: 存储缓存的字节
markpos: 指向最近一次调用mark方法时，pos所指向位置
		 按照 mark & reset 接口约定，从 markpos 至 pos 位置的字节是可以通过
         reset 方法进行重置，然后可以重新读取的。
pos: 指向下次调用 read 方法时，将要读取的第一个字节。

reset zone 区域的数据已经被读取过了，但是如果调用reset 方法，pos 将被重置成 markpos, 所以这部分区域的数据可以被重新读取。

validte zone 区域的数据表示，buffer 中还未被读取的有效的字节。

count 表示：当前 buf 中可以被reset的字节，以及在buff中还未被读取的有效的字节的个数的和

所以 count - pos : 表示当前 buf 可以被直接读取的有效字节数据，（markpos 至 pos的字节只有当前调用 reset 方法之后，才可以被读取）

```

fill 向缓冲区中读取字节，当 pos >= count 时，也就是缓冲区中没有可用的数据时，就会调用 fill 方法，各缓冲区中补充数据。

``` java
private void fill() throws IOException {
    byte[] buffer = getBufIfOpen();
    if (markpos < 0)
		// markpos < 0 表示 markpos 此时是无效的
		// 也就是说，缓冲区中没有 can reset bytes
		// 所以此时 buf 中所有的数据都是无效的或者已经被读取过的
		// 所以，直接将 pos 置为0，
		// 然后在下面通过 read(buffer, pos, buffer.length - pos);
		// 相当于 read(buffer, 0, buffer.length);
		// 表示把 buffer 缓冲区读满数据。
        pos = 0;            /* no mark: throw away the buffer */
    else if (pos >= buffer.length)  /* no room left in buffer */
		// 代码执行到这里则： markpos >= 0 && pos >= buffer.length
		// markpos >= 0 表示 mark方法 已经被调用过了。所以在下面的处理过程中
		// 就必须考虑 markpos 至 pos 这段，可能会被 reset 的字节区。这段
		// 字节区必须被保存下来。
        if (markpos > 0) {  /* can throw away early part of the buffer */
            // markpos > 0 && pos >= buffer.length
			// sz 是 markpos 至 pos 处的字节个数，
			int sz = pos - markpos;
			// 保留 reset zone
            System.arraycopy(buffer, markpos, buffer, 0, sz);
			// reset zone 已经被移到 buffer 的开头
			// 所以将 markpos 置为0， pos 置为 sz.
            pos = sz;
            markpos = 0;
        } else if (buffer.length >= marklimit) {
            // markpos == 0 && pos >= buffer.length && buffer.length >= marklimit
			// 注意上面的条件：pos >= buffer.length && buffer.length >= marklimit
			// <==> pos >= marklimit 表示，pos 的位置，已经超过预定的
			// marklimit的限制了，所以当前的 markpos 无效的，所以
			// 直接重置 markpos 和 pos
            markpos = -1;   /* buffer got too big, invalidate mark */
            pos = 0;        /* drop buffer contents */
        } else {            /* grow buffer */
			// markpos == 0 && pos >= buffer.length && buffer.length < marklimit
			// 上面的三个条件是代码执行到这里的必要条件。
			// 其含义是，当前 buffer 中所有的数据已经被 read过了，but 
			// 由于 markpos == 0 && buffer.length < marklimit,所以整个buffer 的 bytes 全部是允许 reset 的，所以整个buffer 就需要被保留
			// 这也意味着 此时的 buffer 将没有空间，存放 validate zone的数据了
			// 所以必须扩容。
			// 为 buffer 扩容
			// 计算新 buffer 的 size.
            int nsz = pos * 2;
            if (nsz > marklimit)
                nsz = marklimit;

			// 将整个 buffer 中的字节拷贝到新的 buffer 中。
            byte nbuf[] = new byte[nsz];
            System.arraycopy(buffer, 0, nbuf, 0, pos);
			// 更新 buf 字段
            if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                // Can't replace buf if there was an async close.
                // Note: This would need to be changed if fill()
                // is ever made accessible to multiple threads.
                // But for now, the only way CAS can fail is via close.
                // assert buf == null;
                throw new IOException("Stream closed");
            }
            buffer = nbuf;
			// 新的buffer ，相当于原来buffer的后面增加的新的空间
			// 所以 markpos 和 pos 都没有变化，所以不需要进行调整。
        }
	// 将 reset zone 的字节个数 赋给 count
    count = pos;
	// 向 buffer fill 数据
    int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
    if (n > 0)
		// count = validate zone + reset zone 
		// n 是 validate zone 的数据
		// pos 是 reset zone 的数据。
        count = n + pos;
}
```

``` java
public synchronized int read(byte b[], int off, int len)
    throws IOException
{
    getBufIfOpen(); // Check for closed stream
    if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
        throw new IndexOutOfBoundsException();
    } else if (len == 0) {
        return 0;
    }

	// n 表示目前已经向 b 中读取到的字节的个数。
	// 下面通过循环不断向 b 中读取数据
	// 所以初始情况下 n = 0, 表示已经向 b 中读取了 0 个字节。
    int n = 0;
    for (;;) {
		// 已经向 b 中读取了 n 个字节
		// 所以将 b 的下次读取的 offset: 设置成 off + n
		// 读取的 length： 设置成 len - n
		// nread 表示 read1 实际读取的字节个数
        int nread = read1(b, off + n, len - n);

		// nread <= 0 表明，此次 read1 调用
		// 没有读取到数据。 n==0, 表明是循环中的第一次
		// 所以返回 nread 表示读取的字节个数，
		// 如果 n != 0, 则说明已经循环多次了，
		// n 就是 当前 b 中的字节个数。所以直接返回 n
        if (nread <= 0)
            return (n == 0) ? nread : n;

		// 此时 nread > 0 , 表明 read1 向 b 中读取到了
		// 有效的数据，所以将 读取到的字节个数添加到 n 
        n += nread;

		// n >= len 表明 已经向 b 中读取了至少 len 个字节了 所以应该返回了。
        if (n >= len)
            return n;

		// 此时应该是 n < len, 就是向 b 中还未读取到 len
		// 个字节，应该进行循环读取，
		// 但是在循环读取前先判断一下，底层的流中是否
		// 还有数据可供读取，如果没有自然，就没有必要继续
		// 循环读取了，所以直接返回.
        // if not closed but no bytes available, return
        InputStream input = in;
        if (input != null && input.available() <= 0)
            return n;
    }
}

// read1 实现的原则是，至多直接向底层流直接读取数据一次，
// 否则，总是，先从 buf 中读取，如果 buf 读完，则调用 fill
// 将 buf 填充，然后再从 buf 中读取。
private int read1(byte[] b, int off, int len) throws IOException {
    int avail = count - pos;

	// buf 中没有可以直接读取的数据。
    if (avail <= 0) {
        /* If the requested length is at least as large as the buffer, and
           if there is no mark/reset activity, do not bother to copy the
           bytes into the local buffer.  In this way buffered streams will
           cascade harmlessly. */
        if (len >= getBufIfOpen().length && markpos < 0) {
			// len >= getBufIfOpen().length 的情况下，如果直接调用
			// fill 进行数据填充，很可能造成 buf 的扩容操作。
			// 所以在 mark 无效的情况的下，可能直接从底层的 in 流中读取数据
			// 而不是 调用 fill， 这样就避免了 fill 中的对 buf 的扩容
			// 从而避免的空间浪费。还有一个作用就是，如果 mark 无效，
			// 则没有必要维护当前的buff了（因为 len > buf.length）
			// 所以直接将数据读取到 b 中，而不是读取到 buf 中，再从 buf 读取到 b 中
			// 减少数组之间的拷贝，从而提升的读取的效率。
			// 当然如果 mark 是有效的（markpos >= 0）
			// 为了维护 markpos，也就是保留 reset zone 的字节区的数据，
			// 则必须得调用 fill, 此时扩容也是有必要的。
            return getInIfOpen().read(b, off, len);
        }

		// 填充 buf.
        fill();
        avail = count - pos;
        if (avail <= 0) return -1;
    }

	// 从 buf 中读取 cnt 个字节。
    int cnt = (avail < len) ? avail : len;
    System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
    pos += cnt;
    return cnt;
}

```

### mark & reset

支持 mark & reset

``` java
public synchronized void mark(int readlimit) {
    marklimit = readlimit;
    markpos = pos;
}

public synchronized void reset() throws IOException {
    getBufIfOpen(); // Cause exception if closed
	// 值得注意的是，在这里并没有直接使用这样的判断条件
	// int haveread = pos - markpos;
	// if (haveread > readlimit){
	//		throw new IOException("Resetting to invalid mark");
	// }
	// 按照 mark & reset 的语义来说，应该是上面的判断，而
	// BufferedInputStream 却使用下面的方法来判断，mark 是否无效
	// markpos 被重置成 -1 是在 fill 方法中，由此可见，
	// markpos 是否无效有判定，不仅仅由 readlimt 来决定，还
	// 会受到 缓冲区 大小的影响，因为缓冲区大小会决定，fill 方法
	// 何时被调用。
    if (markpos < 0)
        throw new IOException("Resetting to invalid mark");
    pos = markpos;
}

public boolean markSupported() {
    return true;
}
```

由上面的分析，可知 对于 BufferedInputStream 实现的 mark & reset, 其 reset 提供的参数 readlimt 有可能并不会立即生效。也就是说例如： 

``` java

mark(2);
read();
read();
read();
// 此时 已经读取了 3 个字节了，而 readlimit 是2
// 那么下面的 reset 会不会抛出异常，就由 当前 buf 的大小决定了
// 如果 buf 足够大， 不会导致 fill 方法被调用，则 markpos 就不会被置为 -1
// 则 reset 方法就 markpos < 0 的判断就失效了，所以不会抛出 IOException
// 同样，如果 上面的第三个 read 方法引发了 fill 方法调用，则 reset 就会抛出异常
reset();
```

摘自：java.io.InputStream.reset 方法文档

>* If the method markSupported returns true, then: 
> 
>  * If the method mark has not been called since the stream was created, or the number of bytes read from the stream since mark was last called is larger than the argument to mark at that last call, then an IOException might be thrown. 
>
>  * If such an IOException is not thrown, then the stream is reset to a state such that all the bytes read since the most recent call to mark (or since the start of the file, if mark has not been called) will be resupplied to subsequent callers of the read method, followed by any bytes that otherwise would have been the next input data as of the time of the call to reset. 

其中就提到，markSupported returns true 并且 mark 方法也被调用了：then an IOException might be thrown， 
可能会抛出 IOException ，所以说对于 mark & reset 接口来说，并不是说 mark 的参数 readlimit 就一定决定 reset 方法在读取超过 readlimit 个字节被调用时会一定抛出异常。

## DataInputStream

``` java
public class DataInputStream extends FilterInputStream implements DataInput
```

这个类实现的 DataInput 接口。这个类的实现和 DataOutputStream 密切相关。通常是由 DataOutputStream 写入数据，然后由 DataInputStream 读取数据。

``` java
public class DataOutputStream extends FilterOutputStream implements DataOutput
```

这两个类的实现都比较简单，需要注意的是 DataOutput 接口的几个write 方法。

``` java
writeByte(int v)
writeChar(int v)
writeShort(int v) 
```

这三个方法功能正如其名，但是其所接受的参数，都是 int 类型。DataOutputStream 类在实现的时候，会其对应的低字节，也就是对int参数进行截断。例如：

``` java
// DataOutputStream 的 writeShort 方法。
public final void writeShort(int v) throws IOException {
    out.write((v >>> 8) & 0xFF);
    out.write((v >>> 0) & 0xFF);
    incCount(2);
}
```

### 关于 String 的 write 和 read

DataInput 和 DataOuput 接口提供了两种读写 String 的方法。

``` java
// DataOutputStream.writeChars
public final void writeChars(String s) throws IOException {
    int len = s.length();
    for (int i = 0 ; i < len ; i++) {
        int v = s.charAt(i);
        out.write((v >>> 8) & 0xFF);
        out.write((v >>> 0) & 0xFF);
    }
    incCount(len * 2);
}

// DataInput.readLine
String readLine() throws IOException;

// DataOutputStream.writeUTF
public final void writeUTF(String str) throws IOException {
    writeUTF(str, this);
}

// DataInputStream.readUTF
public final String readUTF() throws IOException {
    return readUTF(this);
}
```

由 DataOutputStream 的 writeChars 的实现方法，可知，其并没有对于一个String的结束做任何标记，所以如果通过 writeChars 方法写入 String 则由于无法判断这一系列chars的字节的结束位置，所以在 DataInput 在也就不存在，readChars 方法。

但是 DataInput 提供了一个 readLine 方法。but

> Reads the next line of text from the input stream. It reads successive bytes, **converting each byte separately into a character**, until it encounters a line terminator or end of file; the characters read are then returned as a String. Note that because this method processes bytes, it does not support input of the full Unicode character set. 

注意这个方法是将每一个字节转换成一个字符。所以即使使用 DataOutput 的 writeChar 方法写入字节，这个 readLine 方法也不能正常读取出字符数据来。因为对 writeChar 方法来说一个 char(字符) 使用两个字节表示，所以当使用 readLine 方法时会把 char 分解成两个字符，所以这个方法其实，并不能使用。在 DataInputStream 类实现的 readLine 方法已经将其标记为 @Deprecated 的。

writeUTF 方法将 char 使用类 UTF-8 格式存储。同时在存储chars的时候，将 UTF-8 所占的字节流个数也存储了下来。

``` java
// DataOutputStream.writeUTF
static int writeUTF(String str, DataOutput out) throws IOException {
    int strlen = str.length();
    int utflen = 0;
    int c, count = 0;

    /* use charAt instead of copying String to char array */
	// 1. 计算String转换成 utf-8 字节之后的字节个数
    for (int i = 0; i < strlen; i++) {
        c = str.charAt(i);
        if ((c >= 0x0001) && (c <= 0x007F)) {
            utflen++;
        } else if (c > 0x07FF) {
            utflen += 3;
        } else {
            utflen += 2;
        }
    }

	// 如果存储的数据转换成utf-8字节流之后个数大于 65535
	// 则抛出 UTFDataFormatException。
	// 例如，如果String是一个包含：22000 多个汉字，那么，
	// 每个汉字3个字节，自然超出了 65535 会抛出异常。
	// 至于，为什么是 65535， 原因在下面。
    if (utflen > 65535)
        throw new UTFDataFormatException(
            "encoded string too long: " + utflen + " bytes");

	// 2. 分配存储空间。
    byte[] bytearr = null;
    if (out instanceof DataOutputStream) {
        DataOutputStream dos = (DataOutputStream)out;
        if(dos.bytearr == null || (dos.bytearr.length < (utflen+2)))
            dos.bytearr = new byte[(utflen*2) + 2];
        bytearr = dos.bytearr;
    } else {
        bytearr = new byte[utflen+2];
    }

	// 3. 向缓冲区中写入这个 utf-8 流的 长度 utflen
	// 注意这里使用了两个字节来存储长度，所以 utf-8 流的
	// 长度： utflen 必须 <= 65535. 两个字节所能表示的最大的
	// 无符号数。
    bytearr[count++] = (byte) ((utflen >>> 8) & 0xFF);
    bytearr[count++] = (byte) ((utflen >>> 0) & 0xFF);


	// 4. 转换 char 到 utf-8 字节流。
	// 下面的代码很有意思，如果在 char 在 (c >= 0x0001) && (c <= 0x007F)
	// 范围内，则向 bytearr 内写入数据。直到遇见一个不在上面范围的字符,结束
	// 循环。如果String全是 英文，则下面的循环就可以完全将String转换成 utf-8
	// 字节。
    int i=0;
    for (i=0; i<strlen; i++) {
       c = str.charAt(i);
       if (!((c >= 0x0001) && (c <= 0x007F))) break;
       bytearr[count++] = (byte) c;
    }

	// 如果 i < strlen 表明，此时出现了 (c >= 0x0001) && (c <= 0x007F) 之外
	// 的字符。所以需要下面的循环进行判断转换。
    for (;i < strlen; i++){
        c = str.charAt(i);
        if ((c >= 0x0001) && (c <= 0x007F)) {
            bytearr[count++] = (byte) c;

        } else if (c > 0x07FF) {
            bytearr[count++] = (byte) (0xE0 | ((c >> 12) & 0x0F));
            bytearr[count++] = (byte) (0x80 | ((c >>  6) & 0x3F));
            bytearr[count++] = (byte) (0x80 | ((c >>  0) & 0x3F));
        } else {
            bytearr[count++] = (byte) (0xC0 | ((c >>  6) & 0x1F));
            bytearr[count++] = (byte) (0x80 | ((c >>  0) & 0x3F));
        }
    }
    out.write(bytearr, 0, utflen+2);
    return utflen + 2;
}
```

由上面的存储过程，可以看到，utf-8 bytes 的前两字节就是流的长度。所以在 DataInputStream 的 readUTF 的实现中，就可以正确的知道，这个utf-8 bytes 的个数，就能够正常的解析字符串。所以要想正常写入和读取 String 就使用DataOutputStream.writeUTF 和 DataInputStream.readUTF 方法即可。