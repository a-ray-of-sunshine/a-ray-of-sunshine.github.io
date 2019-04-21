---
title: 可打印流
date: 2016-10-3 14:02:17
---

## PrintStream

这个流类提供了大量 print 类方法，用来将数据以 字符串 形式打印出来。注意这里的打印:

* 输出到屏幕上

	例如 使用 System.out.println(); 其中的 out 变量就是 PrintStream 的一个实例。
	
	``` java
	// 注意到这几个变量都是 final 类型的
	// 并且，其初始化值都被设置成 null
	// 所以为了不违反 java 的 final 关键词的语义
	// 对于这三个变量的 set 方法，提供了 native 的实现，
	// 即 setIn0, setOut0, setErr0
	public final static InputStream in = null;
	public final static PrintStream out = null;
	public final static PrintStream err = null
	
	// FileDescriptor.in, FileDescriptor.out 为系统预定义的文件描述符
	// 表示标准输出，标准输入。
	FileInputStream fdIn = new FileInputStream(FileDescriptor.in);
	FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
	FileOutputStream fdErr = new FileOutputStream(FileDescriptor.err);
	
	// 创建对应的 PrintStream 流。
	setIn0(new BufferedInputStream(fdIn));
	setOut0(new PrintStream(new BufferedOutputStream(fdOut, 128), true));
	setErr0(new PrintStream(new BufferedOutputStream(fdErr, 128), true));
	```
	
	setOut0 的实现如下：
	
	``` c
	// System.c
	JNIEXPORT void JNICALL
	Java_java_lang_System_setOut0(JNIEnv *env, jclass cla, jobject stream)
	{
		// 获得 System.out 的 fieldId
	    jfieldID fid =
	        (*env)->GetStaticFieldID(env,cla,"out","Ljava/io/PrintStream;");
	    if (fid == 0)
	        return;
	
		// 设置 out 字段的值为 stream
	    (*env)->SetStaticObjectField(env,cla,fid,stream);
	}
	```

* 输出到文件中

	``` java
	FileOutputStream fos = new FileOutputStream(filename);
	PrintStream ps = new PrintStream(new BufferedOutputStream(fos), true);
	```
	
	这里 ps 的 print 方法调用，会将数据输出到 filename 命名的文件中去。

* 输出到内存中

	``` java
	ByteArrayOutputStream bos = new ByteArrayOutputStream();
	PrintStream ps = new PrintStream(bos);
	ps.println("你好");
	ps.println(7892);
	ps.print(434.22);
	System.out.println(bos.toString());
	```

	这里的 ps 将数据输出到 bos 所拥有的字节数组中。


### 构造函数

PrintStream 提供了许多构造函数，最终都是使用下面的两个，来完成构造过程。

``` java

private BufferedWriter textOut;
private OutputStreamWriter charOut;

/* Private constructors */
private PrintStream(boolean autoFlush, OutputStream out) {
    super(out);
    this.autoFlush = autoFlush;
    this.charOut = new OutputStreamWriter(this);
    this.textOut = new BufferedWriter(charOut);
}

private PrintStream(boolean autoFlush, OutputStream out, Charset charset) {
    super(out);
    this.autoFlush = autoFlush;
    this.charOut = new OutputStreamWriter(this, charset);
    this.textOut = new BufferedWriter(charOut);
}
```

这个类的实现过程，如下：

1. PrintStream 类的 print 系列方法，将原生数据类型转换成 String 类型。

	``` java
	public void print(long l) {
        write(String.valueOf(l));
    }
	```

2. PrintStream 的 print 方法将调用其 write 方法

	``` java
    private void write(String s) {
        try {
            synchronized (this) {
                ensureOpen();
                textOut.write(s);
                textOut.flushBuffer();
                charOut.flushBuffer();
                if (autoFlush && (s.indexOf('\n') >= 0))
                    out.flush();
            }
        }
        catch (InterruptedIOException x) {
            Thread.currentThread().interrupt();
        }
        catch (IOException x) {
            trouble = true;
        }
    }
	```

3. BufferedWriter 调用 write, 使写入的过程带有缓冲功能，同时将 String 转换成 char 数组

	``` java
    void flushBuffer() throws IOException {
    synchronized (lock) {
        ensureOpen();
        if (nextChar == 0)
            return;
        out.write(cb, 0, nextChar);
        nextChar = 0;
    	}
	}
	```

4. BufferedWriter 内部持有 OutputStreamWriter 

	当 BufferedWriter 调用上面的 flushBuffer 时，将调用 OutputStreamWriter 的 write 方法。

	``` java
    public void write(char cbuf[], int off, int len) throws IOException {
        se.write(cbuf, off, len);
    }
	```

5. 调用 StreamEncoder 对 char[] 进行编码，将编码后的字节写入目的Stream中。

	``` java
	// out 参数是流最终目标，可能是 
	// 文件: FileOutputStream
	// 内存: ByteArrayOutputStream
	// 等等。
	public OutputStreamWriter(OutputStream out, String charsetName)
        throws UnsupportedEncodingException
    {
        super(out);
        if (charsetName == null)
            throw new NullPointerException("charsetName");
		// se 持有 out, 最终 se 将接受 char 数组
		// 将 char 数组，以 charsetName 的字符集进行编码成字节流
		// 写入到 out 中。
        se = StreamEncoder.forOutputStreamWriter(out, this, charsetName);
    }
	```

	这就是一个编码转换的过程 对于待写入的数据 char[] cbuf 来说，其本身是 UTF-16BE 格式的编码，现在需要写入目标流中，因为 utf-16BE 是 JVM 自身用来存储字符数据的编码格式，所以需要将这种编码格式的数据重新进行编码（Encoder）然后输出到输出流 out 中。那么新的编码格式如何获取呢？可以看看 StreamEncoder.forOutputStreamWriter 的实现。

### StreamEncoder 类的实现

注意这个类所在的包是 sun.nio.cs 所以 jdk 文档中并没有这个类的信息，可以参考 openjdk 的源码：

``` java
public class StreamEncoder extends Writer{

    private Charset cs;
	// 将由 Charset.newEncoder() 来初始化
    private CharsetEncoder encoder;

    // Factories for java.io.OutputStreamWriter
    public static StreamEncoder forOutputStreamWriter(OutputStream out,
                                                      Object lock,
                                                      String charsetName)
        throws UnsupportedEncodingException
    {
        String csn = charsetName;
        if (csn == null)
			// 如果没有传递 charsetName 则使用 JVM 默认的 字符集。
            csn = Charset.defaultCharset().name();
        try {
            if (Charset.isSupported(csn))
                return new StreamEncoder(out, lock, Charset.forName(csn));
        } catch (IllegalCharsetNameException x) { }
        throw new UnsupportedEncodingException (csn);
    }

    private StreamEncoder(OutputStream out, Object lock, Charset cs) {
        this(out, lock,
         cs.newEncoder() // 这里调用 cs.newEncoder() 创建encoder
         .onMalformedInput(CodingErrorAction.REPLACE)
         .onUnmappableCharacter(CodingErrorAction.REPLACE));
    }
}
```

StreamEncoder 类继承自 Writer， 所以它有 writer 方法，

``` java

private ByteBuffer bb;

public void write(char cbuf[], int off, int len) throws IOException {
    synchronized (lock) {
        ensureOpen();
		// 参数检查
        if ((off < 0) || (off > cbuf.length) || (len < 0) ||
            ((off + len) > cbuf.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
		// 调用具体的实现
        implWrite(cbuf, off, len);
    }
}

void implWrite(char cbuf[], int off, int len)
    throws IOException
{
    CharBuffer cb = CharBuffer.wrap(cbuf, off, len);

    if (haveLeftoverChar)
    flushLeftoverChar(cb, false);

    while (cb.hasRemaining()) {

	// 实现的关键，使用 encoder 的  encode 方法对 cb 中的 char 进行
	// 编码，编码成 encoder 所代表的字符集的字节数据，放到 bb 中
	// bb 是全局变量 ByteBuffer bb;
    CoderResult cr = encoder.encode(cb, bb, false);

    if (cr.isUnderflow()) {
       assert (cb.remaining() <= 1) : cb.remaining();
       if (cb.remaining() == 1) {
            haveLeftoverChar = true;
            leftoverChar = cb.get();
        }
        break;
    }
    if (cr.isOverflow()) {
        assert bb.position() > 0;
		// 数据已经解码成功，放置到 bb 当中。
		// 调用 writeBytes 将 bb 当中的数据写入到底层的流中
		// 释放 bb 的空间，然后，继续进行编码剩余的数据。
        writeBytes();
        continue;
    }
    cr.throwException();
    }
}

private void writeBytes() throws IOException {
    bb.flip();
    int lim = bb.limit();
    int pos = bb.position();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);

        if (rem > 0) {
    if (ch != null) {
        if (ch.write(bb) != rem)
            assert false : rem;
    } else {
		// 写入到底层的流中，
		// out 可能是 FileOutputStream， 
		// ByteArrayOutputStream 等等。
        out.write(bb.array(), bb.arrayOffset() + pos, rem);
    }
    }
    bb.clear();
}
```

## PrintWriter

这个类的实现和 PrintStream 非常相似

``` java
public class PrintWriter extends Writer {

	public PrintWriter(OutputStream out, boolean autoFlush) {
	    this(new BufferedWriter(new OutputStreamWriter(out)), autoFlush);
	
	    // save print stream for error propagation
	    if (out instanceof java.io.PrintStream) {
	        psOut = (PrintStream) out;
	    }
	}

    public PrintWriter(File file) throws FileNotFoundException {
        this(new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file))),
             false);
    }

	// ...
}
```

new BufferedWriter(new OutputStreamWriter(new FileOutputStream(file)) 最终使用这个 Writer 入流中写入数据。

需要注意的是： PrintStream类有两个可以直接写入字节数据的方法：

* write(byte[] buf, int off, int len)
* write(int b)

在 PrintStream 的实现中，这两个方法都是直接将原始的字节数据写入到底层的流中，而没有进行编码操作。

在 PrintWriter 中，对应的有

* write(int c)

这个方法会将 c 进行转码操作，然后写入底层流中。


## 多字节字符编码，解码

UTF-16 和 UTF-8, UTF-32 存在直接的映射关系，所以可以直接通过线性计算就可以实现转码。

所谓线性关系就是， U8 = f(U16), 而这里的 f 是一个数学运算函数。

由于 GBK 和 UTF-16 编码并没有线性映射关系（因为相同的字符在 GBK 中的编码和 UTF-16 中的编码的分布是没有规律），所以GBK 和 UTF-16的编码转换必须通过不断的循环查找来实现转换，这样效率非常低。

所以在非线性编码的转码（编码和解码）的过程中创建一个码表将非线性的关系转换成了线性关系。这样转码的效率就是O(1)了，而不用循环查找，这么低效的方法。

所以创建码表就成了关键所在：

jdk中提供的类：sun.nio.cs.ext.GBK 代表 GBK 字符集。 这个字符集中就实现了两个码表：

``` java
public class GBK extends Charset implements HistoricallyNamedCharset {

	// GBK 字节 ====> UTF-16 字符 的码表
	static char[][] b2c = new char[b2cStr.length][];
	static char[] b2cSB;
	private static volatile boolean b2cInitialized = false;

	//  UTF-16 字节 ====> GBK  字符 的码表
	static char[] c2b = new char[28672];
	static char[] c2bIndex = new char[256];
	private static volatile boolean c2bInitialized = false;

}

// 摘自 openjdk sun.nio.cs.ext.DoubleByte
 * Decoding: 解码 GBK ===> UTF16
 *
 *    char[][] b2c;
 *    char[] b2cSB;
 *    int b2Min, b2Max
 *
 *    public char decodeSingle(int b) {
 *        return b2cSB.[b];
 *    }
 *
 *    public char decodeDouble(int b1, int b2) {
 *        if (b2 < b2Min || b2 > b2Max)
 *            return UNMAPPABLE_DECODING;
 *         return b2c[b1][b2 - b2Min];
 *    }
 *
 *    (1)b2Min, b2Max are the corresponding min and max value of the
 *       low-half of the double-byte.
 *    (2)The high 8-bit/b1 of the double-byte are used to indexed into
 *       b2c array.
 *
 * Encoding: 编码： UTF16 ===> GBK
 *
 *    char[] c2b;
 *    char[] c2bIndex;
 *
 *    public int encodeChar(char ch) {
 *        return c2b[c2bIndex[ch >> 8] + (ch & 0xff)];
 *    }

```


```
GBK ===> UTF-16 码表:

 0x  00  01  02  03  ... 40  41  42  43  ... a0  a1  a2  a3  ... f8   f9  fa  fb  fd  fe  ff
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 00 |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 01 |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 02 |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 ...|   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 81 |   |   |   |   |   |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 82 |   |   |   |   |   |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 83 |   |   |   |   |   |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 ...|   |   |   |   |   |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 fd |   |   |   |   |   |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 fe |   |   |   |   |   |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |Ux |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
 ff |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |   |
	+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+

Ux表示： 字符的 UTF-16 编码。

由上面的码表，可以反向，计算出另一张码表: UTF-16 ===> GBK.

假设，上面的码表存储在：

// GBK ===> UTF-16 码表：
char[][] b2c = new char[0x100][0x100];

// UTF-16 ===> GBK
char[][] c2b = new char[0x100][0x100];
for(int b1 = 0x00; b1 < 0x100; b1++){

	for(int b2 = 0x00; b2 < 0x100; b2++){
		
		// 得到 GBK编码为 (b1, b2) 字符的 UTF-16 编码
		char Ux = b2c[b1][b2];
		
		// 得到 UTF-16 编码的高位字节。
		int i1 = Ux >> 8;
		// 得到 UTF-16 编码的低位字节。
		int i2 = Ux && 0xff;

		// 计算 GBK 编码
		char gbk = (char)((b1 << 8) | b2);

		c2b[i1][i2] = gbk
	}

}

这样就制作出的另一张码表：UTF-16 ===> GBK
下面模拟，转换过程：

char[] utf16chars = {'丂', '丄', '丅', '丆', '丏'};
char c1 = utf16chars[0];
int i1 = c1 >> 8;
int i2 = c1 && 0xff;

// 直接使用 c2b 码表即可
int gbk = c2b[i1][i2];

上面的实现，只是为了表达编码转换的过程和思路，sun.nio.cs.ext.DoubleByte 的实现更加的优化，码表占用空间更小，还考虑到了和 ASCII 字符集兼容的问题等等，但其基本思路是和上面的描述是一致的。
```

由上面的码表可知，这其实就是一个二维数组：

``` java
// char 数组用来存储上面的码表
char[][] b2c;
```

对于 二维数组 b2c ，其第一维是 GBK 的高位字节： 0x81 -- 0xfe, 其第二维，构成了 char[] 存放 GBK 低字节 0x40 -- 0xfe 其 191 个字符的 char 数据，由 java 语言可以知道 char 类型其实存储的是 UTF-16BE 编码。所以这张码表就实现了从 GBK 字符集到 UTF-16BE 字符集的编码操作。一旦字节从 GBK 转成 UTF-16BE， 那么 UTF系列编码之间的转换将非常容易，因为它们之间是存在线性关系的。比如，从 UTF-16BE ==> UTF-16LE 按照 UTF-16LE 的编码规范，将 UTF-16BE 字符的高位字节和底位字节对调即可完成转换。UTF-16BE ==> UTF-8 也非常容易，按照 UTF-8 的编码规则进行计算即可。

下面模拟，转换过程：

```
// 丂丄丅丆丏 
// 上面5个字符的 GBK 编码：
byte[] bytes = {81, 40, 81, 41, 81, 42, 81, 43, 81, 44};

// GBK 是双字节字符集，所以使用两个字节来代表一个字符，所以当读取到上面的字节时，每两个字节构成一个字符：

// 先解析 81, 40 按照上面码表的规则，81 将是这个字符的在码表的高位索引，40是低位索引所以对应的 char 就是：

char c = b2c[81][40]  

// 这个 c 就是编码后的字符，其编码是 UTF-16 ，也是两个字节，分别对应：4E, 02
// 所以通过 b2c 就可以将 GBK 字节流转换成 UTF-16 字节流。
// 字符     GBK       UTF-16
//  丂    (81, 40)   (4E, 02)
//  丄    (81, 41)   (4E, 04)
//  丅    (81, 42)   (4E, 05)
//  丆    (81, 43)   (4E, 06)
//  丏    (81, 44)   (4E, 0F)
```

从上面的转换过程中，可以看到，所谓不同编码（或者说，不同字符集）之间的转换其实就是：

**字符A（丂） 在字符集C1（GBK）中的字节（81, 40）  通过映射或者线程运算（b2c[81][40]，其实就是一种映射, b2c 本身就是一个二维数组构成的编码对照表，所以也可以认为就是查表操作） 
转换成 字符集C2(UTF-16)中该字符的字节（4E, 02）**

* 字符是什么？

	字符的本质就是图形

* 字符集是什么？

	字符集的本质就是字符的集合，一个文化领域内的字符构成一个字符集，例如：汉字字符构成的集合：GBK字符集

* 字符编码是什么？

	对于字符和字符集来说，本身是没有所谓编码的概念的。例如: GBK 字符集的字符：汉字，当我们使用的时候，就是按照这个字符的字形进行书写的，所以这个过程并没有涉及到编码。

	但是，对于计算机来说，它是无法理解字符的字形，也就不可能让计算机来书写字符（也即在屏幕上显示字符），但是，换个角度，对于同一个书写方法（例如：宋体书法，隶书，草书）其所对应的字符的字形（glyph）总是固定的，而且一个字符集中所包含的字符总数也是固定的。

	所以，可以将字符集中的所有字符的图形的形式存储起来，然后给每一个图形（字符）进行编号。当我们在计算机中存储字符时，就可以直接存储这些编号。计算机在屏幕上显示的时候，就可以使用这些编号从字符集的图形文件中索引到该字符对应的图形，将这个图形显示出来，就间接地达到也显示字符的效果。

	这个存储 字符集中的所有字符的图形 的文件就是 字体文件。

	对每一个字符需要有一个惟一的编号，使其可以从 字体文件 中被索引到。这个编号 就是 这个字符的编码。

	字符的编码就是一个数字，这个数字在同一个字符集中惟一的索引这个字符。

    由此，也可以知道，其实这个 索引（编码） 就是人为规定的，即使这个规定中存在一定的规律（例如：汉字按拼音排序，高位使用 0x81-0xfe, 低位使用 0x40-0xfe 编码（注意，这只是一个假设的编码的规则，为了便于理解）），它也只不过是一个人为的编号而已。

* 为什么会有不同的字符集？

	由上面的分析可以知道，只要有一个字符集就可以表示字符了，为什么出现多个？
	先出现 GB2312，也就是先编码了最常用的字符，后来计算机普及到各个领域，需要表示的字符需要扩充，就出来了 GBK, GB18030 等等。

* 字符集之间的转换？

	其实就是同一个字符在不同字符集间的编码的转换，就是字节到字节之间的转换。

### java 中的字符集类

* Charset

	> A named mapping between sequences of sixteen-bit Unicode code units and sequences of bytes.

	表示各种字符集： UTF-16, UTF-8, GBK 等等，

	``` java
	// 创建这个字符集的解码器，即 当前字符集 ===> UTF-16
    public abstract CharsetDecoder newDecoder();
	// 创建当前字符集的编码器，即 UTF-16 ===> 当前字符集
    public abstract CharsetEncoder newEncoder()
	```

* CharsetDecoder

	> An engine that can transform a sequence of bytes in a specific charset into a sequence of sixteen-bit Unicode characters

* CharsetEncoder

	> An engine that can transform a sequence of sixteen-bit Unicode characters into a sequence of bytes in a specific charset. 

* sun.nio.cs.StreamEncoder

	StreamEncoder 接受 OutputStream out 和 Charset cs. 这个类还是一个 Writer。这个类的功能就是
	使用 write 方法时，将 char[] 通过 encoder 的编码，转码成 cs 字符集字符，然后，写入到 out 流中。

	``` java
	public class StreamEncoder extends Writer{
		// 这个Stream的 Encoder
    	private CharsetEncoder encoder;

		// 构造函数中将初始化这个 Encoder
		private StreamEncoder(OutputStream out, Object lock, Charset cs) {
        this(out, lock,
         cs.newEncoder()
         .onMalformedInput(CodingErrorAction.REPLACE)
         .onUnmappableCharacter(CodingErrorAction.REPLACE));
    	}

		// write 方法的实现中
		// 将使用 encoder, 对字符进行转码操作。
		// encoder.encode(cb, bb, false);
	 	void implWrite(char cbuf[], int off, int len) throws IOException
	    {
	        CharBuffer cb = CharBuffer.wrap(cbuf, off, len);
	
	        if (haveLeftoverChar)
	        flushLeftoverChar(cb, false);
	
	        while (cb.hasRemaining()) {
	        CoderResult cr = encoder.encode(cb, bb, false);
	        if (cr.isUnderflow()) {
	           assert (cb.remaining() <= 1) : cb.remaining();
	           if (cb.remaining() == 1) {
	                haveLeftoverChar = true;
	                leftoverChar = cb.get();
	            }
	            break;
	        }
	        if (cr.isOverflow()) {
	            assert bb.position() > 0;
	            writeBytes();
	            continue;
	        }
	        cr.throwException();
	        }
	    }
	}
	```


* sun.nio.cs.StreamDecoder

	这个类接受： InputStream in 和 Charset cs ， cs 表示 in 数据流的字符集。可以通过 decoder 将 in 流转码成 java 中的 char 类型，也就是转换成 UTF-16 字节。这样 java 代码就可以处理这些字符数据了。