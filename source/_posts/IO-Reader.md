---
title: Reader & Writer
date: 2016-10-6 17:10:05
---

## Reader & Writer

### Reader

* BufferedReader
	* LineNumberReader
* CharArrayReader
* FilterReader
	* PushbackReader
* InputStreamReader
	* FileReader
* PipedReader
* StringReader

### Writer

* BufferedWriter
* CharArrayWriter
* FilterWriter
* OutputStreamWriter
	* FileWriter
* PipedWriter
* PrintWriter
* StringWriter

Reader & Writer 也是流类 API， 不过和 InputStream & OutputStream 不同的是它们操作的数据对象是字符流，而不是字节流。**字符流操作的对象就是 char , 也就是两个字节的 UTF-16 编码的数据。**

同样 Reader 系列的流的前缀表示的流的来源，表示流从什么地方读取数据。例如： CharArrayReader，这个字符流的数据来源就是 CharArray 也即字符数组。

Writer 系列流的前缀表示流的去向，表示流将被写入到哪里。例如：FileWriter 数据将被写入到 File（文件）中去。

基本上，Reader 和 Writer 都有互相对应的流，这也是 IO 的一个特点：流可以写入到 A 中，自然也应该可以从 A 中读取出来。反之亦然。

## BufferedReader & BufferedWriter

使得 Reader 和 Writer 的过程带有缓冲区功能。默认的缓冲区大小 8KB。 使用缓冲区使得低层流的I/O次数减少，从提高效率。

注意： 在字节流中，BufferedInputStream 和 BufferedOutputStream 是 Filter 流，而这里却直接继承自 Reader 和 Writer，而不是继承自 FilterReader 和 FilterWriter。

### 按行读写数据

* BufferedReader.readLine

	> Reads a line of text. A line is considered to be terminated by any one of a line feed ('\n'), a carriage return ('\r'), or a carriage return followed immediately by a linefeed.

	从字符流中读取一行数据。注意行结束符，可以是 '\r', '\n', '\r\n'. 一旦遇到这三个字符中的任意一个，都会认为是一行数据。

	> A String containing the contents of the line, not including any line-termination characters.

	返回的数据中，并不会包含行结束符。

* BufferedWriter.newLine

	> Writes a line separator. The line separator string is defined by the system property line.separator, and is not necessarily a single newline ('\n') character.

	向字符流中写入一个行结束符。这里的行结束符来自系统属性 line.separator。一般在 windows 平台是 '\r\n', 在 linux 平台是 '\n'

### mark & reset

BufferedReader 支持 mark & reset 操作。

### flush

将缓冲区的数据写入到底层的流中。

## CharArrayReader & CharArrayWriter

char[] 成为，数据的源头或者目的地。

``` java
/** The character buffer. */
protected char buf[];
```

内部均有一个这样在的字段，用来存储待操作的字符数组。

* 对于 CharArrayReader 来说，read 方法将从这个数据中读取数据。

	** CharArrayReader 共享原始的外部字符数组 **

	这个字段指向的是从构造函数中传入的待读取的 buf 引用，注意它不会将外部的 char 数组进行复制，而是直接使用。

* 对于 CharArrayWriter 来说，writer 方法将向这个数组中写入数据。 

	** CharArrayWriter 通过扩容操作保证数据可以写入 **

	默认的容量是 32。但是其的 write 方法会在写入之前检测是否可以存储，如果无法存储，将会进行扩容操作。所有可以认为 CharArrayWriter 可以写入无限多个字符。其内部的扩容策略如下：

	``` java
	// count 表示当前 buf 中已经写入的字符个数，len 表示等待写入字符的个数
    int newcount = count + len;
    if (newcount > buf.length) {
        buf = Arrays.copyOf(buf, Math.max(buf.length << 1, newcount));
    }
	```

	如果当前 buf 的长度 * 2 大于，需求的个数，则使用这个长度去扩容，否则，使用需要的长度去扩容。

### mark & reset

CharArrayReader 支持 mark & reset 方法。

### flush

CharArrayWriter 因为操作的是字符数组，其实就是内存操作，并没有占用外部设备，资源等等。所以 flush 操作是空实现，donothing.

### close

CharArrayReader 的将关闭操作将使其内部引用的 buf 置为 null, 由于 read 操作会对 buf 进行判断空，当 close 调用之后，再次调用 read 将触发异常： throw new IOException("Stream closed");

CharArrayWriter 的close 是空实现，也就是说当调用 close 之后，不会对后续的 write 操作产生任何影响。之所以这样实现的原因是：

> This method does not release the buffer, since its contents might still be required. 

**但是从接口约定的语义规则上来说，当调用 close 之后，就表示这个字符流已经被关闭了，所以不能也不应该进行调用 write 来使用这个流了。**

CharArrayWriter 将写入字符存储在 buf 中，可以通过 `toCharArray` 方法获得 buf 中的chars.

## PipedReader & PipedWriter

这两个类构成了一个字符流管道，PipedReader 和 PipedWriter 可以相互连接在一起，然后，由 PipedWriter 向 pipe 中写入字符数据，PipedReader 将从 pipe 中读取数据。

它们两者之间都可以相互发起 connect, 但是只需要一方进行 connect 管道就可以正常使用了，一旦再次调用 connect，就会发生 throw new IOException("Already connected"); 已经连接异常。

管道的内部结构：

``` java

/**
* The size of the pipe's circular input buffer.
*/
private static final int DEFAULT_PIPE_SIZE = 1024;

/**
 * The circular buffer into which incoming data is placed.
 */
char buffer[];

// PipedReader 内部有一个 1KB 的缓冲区。这个缓冲区其实就充当的管道的角色
// PipedWriter 将字符流 write 到 buffer 中， PipedReader 从 buffer 中读取字符流

						 +-----------------+
PipedWriter ---chars---> |  pipe(buffer)   |---chars---> PipedReader
						 +-----------------+
```

通常 PipedWriter 在线程 A 中向管道中写入数据，PipedReader 在线程 B 中从管道中读取数据。如果 线程 A 和 线程 B 在数据生产(write)和消费(read)的速率不同，会出现，pipe 为空，或者 pipe 被占满。

* 当 PipedWriter 的速率大于 PipedReader 
	
	管道可能被占满，此时 PipedWriter 再调用 write 方法线程将进行 wait 状态。直到 PipedReader 从管道中消费了数据，pipe 有了存储空间，write 方法就可以继续使用了。

* 当 PipedReader 的速率大于 PipedWriter 

	管道可能为空，此时 PipedReader 再调用 read 方法线程将进行 wait 状态。直到 PipedWriter 向管道中写入到数据，PipedReader 就可以消费了。

### close

管道的任何一方调用 close 方法将使得管道被关闭，继续使用 write 和 read 将出现。throw new IOException("Pipe closed");

## StringReader & StringWriter

### StringReader

将使用 String 作为数据源头。read 方法将读取构成一个 String 的一个个字符，通过 String 类的

* String.charAt
* String.getChars

方法来实现，charAt 获得指定索引处的字符，getChars 获得指定长度的chars.

支持 mark & reset 操作。

close 方法将使用引用的 str 置为 null. 后续的 read 调用将引发 IOException("Stream closed")。

### StringWriter

内部持有一个 StringBuffer， StringWriter 的 write 操作调用 buf 的 apped 方法将数据写入到 StringBuffer 中。

```
private StringBuffer buf;
```

支持 mark & reset

close 方法 donothing. 和 CharArrayWriter 类似，方便 buf 中的数据被再次使用。

调用 ` getBuffer ` 方法用来获得底层的 buf.

## LineNumberReader

行结束符： '\r'(0x0D), '\n'(0x0A), '\r\n'(0x0D0A) 读取的字符流中如果遇到这三个字符中的任意一个，就认为是一行数据。

``` java

    /** The current line number */
    private int lineNumber = 0;

```

lineNumber 用来跟踪行数。每遇到一行，就增加1.

### public int read()

每次读取一个字符。

LineNumberReader 的 read 方法在读取过程中如果遇到行结束符，统一返回 '\n'(0x0A)，如果行结束符是 '\r\n' 在 read 读取到 '\r' 时会将 lineNumber++， 然后直接返回 '\n', 等到下次调用 read 时应该是 '\n', 但是会直接跳过，读取下一个有效的字符。相当于三种类型的行结束符都被压缩成了 '\n'

所以文档中说：

> Read a single character. Line terminators are compressed into single newline ('\n') characters. 
> 
> Whenever a line terminator is read the current line number is incremented.

### getLineNumber

可以通过这个方法获得当前读取数据的行号。

## PushbackReader

**内部持有一个固定 size 的 字符 buffer, 可以允许指定 size 的字符被回退到当前字符流，read 方法将优先读取 buffer 中的字符，然后从底层流中读取。**

``` java
public class PushbackReader extends FilterReader {

    /** Pushback buffer */
    private char[] buf;

    /** Current position in buffer */
    private int pos;

	// ...
}
```

内部持有一个 buf, 注意这个 buf 的大小在创建 PushbackReader 时就固定了，如果没有指定，则默认是 1 ，也就是说只允许 1 个字符被 pushback.

unread 方法不会因为没有存储空间，而自动扩容。而是直接抛出异常： new IOException("Pushback buffer overflow");

所以在创建 PushbackReader 的时候，需要考虑好这个 Reader 的使用方式，给 buf 指定适合的 size.

不支持 mark & reset 方法。

## InputStreamReader & OutputStreamWriter

字节流和字符流之间的桥梁。

其内部持有编解码器，用来进行字符和字节之间的编解码。

InputStreamReader 从 InputStream 读取出字节后,使用指定的字符集的 Decoder 解码成 UTF-16的 char 数据，然后被读取。

OutputStreamWriter 将 UTF-16 编码的 char 数据，通过指定字符集的 Encoder 编码成字节数据 写入到底层的 OutputStream 中。

如果没有指定字符集，这两个类使用系统默认的字符集：Charset.defaultCharset()。

所以，如果使用到这两个类的 read 和 write 系列方法，其内部将会进行编解码操作，从效率方面考虑，一般使用下面的形式使用这两个类：

``` java
// 通过增加缓冲区，减少对编码器的的编解码功能的调用，从而提高效率
Reader in = new BufferedReader(new InputStreamReader(inputStream));
Writer out = new BufferedWriter(new OutputStreamWriter(outputStream));
```

### FileReader & FileWriter

分别为 InputStreamReader & OutputStreamWriter 的子类，以字符流的形式读写文件。

虽然文本文件的读写是非常常用的功能，但是这两个类的使用确需要注意，因为它们将使用默认的的字符集对文件进行编解码，而没有参数，可以设置字符集。所以下面的情况下可能会出现问题： 

``` java

// 假设 1. a.txt 是 GBK 编码的文本文件
//      2. 系统默认的字符集是 UTF-8。
FileReader fr = new FileReader("a.txt");

// 这个调用，将引发，底层的解码器按照 UTF-8 对 a.txt 中的字节流进行解码
// 但是，a.txt 却是 GBK 编码的，所以 会出现乱码问题。
int c = read();
```

所以对于文件文本的读写推荐的方法是：

``` java
// 其 fileEncodeName 是 a.txt 这个文件字符数据的编码格式
Reader in = new BufferedReader(new InputStreamReader(new FileInputStream("a.txt"), Charset.forName("fileEncodeName")));

// 其 fileEncodeName 是 以何种编码将字符数据保存到 a.txt 这个文件中
Writer out = new BufferedWriter(new OutputStreamWriter(new FileOutputStream("a.txt"), Charset.forName("fileEncodeName")));
```