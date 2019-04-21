---
title: java字符集
date: 2016-09-16 15:37:45
---

我们在java文件中写的代码的所有字符都是字符串。例如：

``` java
int ia = 123;
String str = new String("你好");
```

上面的代码，使用二进制查看如下：

```
00000000: 696e 7420 6961 203d 2031 3233 3b0a 5374  int ia = 123;.St
00000010: 7269 6e67 2073 7472 203d 206e 6577 2053  ring str = new S
00000020: 7472 696e 6728 22e4 bda0 e5a5 bd22 293b  tring("......");
00000030: 0a                                       .
```

可以看到，空格： 0x20, 分号： 0x3b, 换行：0x0a, 引号：0x22

其实对于，计算机来说，它根本就不知道，任何所谓，字符串，汉字，英文字母，数字等等。其实在计算机上面所有我们所能看到的所谓汉字，英文字母，数字都是图片（可以简单认为是图片，或者说是 图形 graph）。

那么，如何存储这些图形呢？ 例如：在　Hello.java 文件中有这样一句代码片断：

``` java
String str = new String("你好");
```

现在，已经知道 `String` 其实是6个图形。那么 Hello.java 文件中是要存储这 6 个图形数据吗？图形数据如何表示呢？由谁来定 S 个图形的数据具体是什么呢？ 假如由当前的使用的编辑器来定，则 上面的代码是在 eclipse 中编辑的，所以对于  eclipse 它可以定 S 这个字符的图形数据。eclipse 知道如何解析，并显示这个字符。但是，问题来了，我要使用 notepad.exe 来查看 Hello.java 文件，但是对于 notepad 来说，它并不清楚 eclipse 到底是存储的 S 这个图形的数据结构。则 notepad 肯定是无法正常显示出 S 这个字符了。

但是，事实上，使用 eclipse 编写的代码，notepad 等，任何文本编辑器都可以正常打开。并显示出 String 这些字符。这是怎么回事呢？难道所有文本编辑器都相互将如何保存和显示文本的格式都公开了，当 notepad 打开 eclipse 编辑的 Hello.java 时，就使用 eclipse 的图形解析程序，将 Hello.java 中存储的图形数据。进行解析展示。

但是，如果是这样，notepad 就需要知道 Hello.java 这个文件是 eclipse 编辑的，而不是 notepad++ 或者 ue 等编辑器编辑的。所以 对于每一个文本编辑器，都必须要在编辑的文件中标记清楚，这个文件是由什么工具编辑的。

显然，对于字符的保存和显示所以按照上面的思路去实现，每个文本编辑器要打开显示一个文件将变得非常复杂。

上面的思路是：Hello.java 中就存储，字符的图形数据，这个图形数据可能非常复杂。不同的字符，要表达清楚这个字符，可能图形数据完全不同，如果按照这个思路实现，文本存储，将使得文本的显示变得异常复杂（需要解析每一个图形数据）。并且，常用的字符，就那么多。例如一本 100 万字的小说，其中常用字可能就只有 2000 个，所以，如果使用上面的实现，则同一个字符，将要在这个小说文件中存储 1000000 / 2000 = 500 个完全相同的字符数据。例如 “我” 这个常用字，其图形数据需要 20 字节（假设的，真实需要描述这个字符肯定比较复杂）。这样的话，20 * 500 = 10000, 大约需要 10kb. 而对于每一个字符，其图形数据肯定是完全一样的，那么每出现一次，就存储一次，完全是没有必要的，这样做非常浪费存储空间。

考虑下面的思路：给每一个字符编号，每一个字符对应惟一的一个编号，然后在文件中存储这个编号就可以了。然后，另外创建一个文件 CharsetMapFile（字符编号映射文件），这个文件中存储所有的字符的图形数据和编号。当向一个文本文件中写入一个字符时，就将其编号保存到这个文件中。

当 notepad 打开这个文件时，所读到的全部是字符编号，就到 CharsetMapFile 找到，编号对应的图形数据，然后，就可以正常显示出文件了。

这样做的，图形数据只需要在 CharsetMapFile 存储一份就可以了。所有的文本文件都存储字符编号。

假设，使用 2 个字节进行编号，则一共可以有 2 ^ ( 2 * 8 ) = 2 ^ 16 = 65536 个编号。所以可以有 65536 个字符，可以使用 2 个字节来表示。 常用汉字，英文字母，数字，标点符号也就 3，4千个字符。可见使用 2 个字节来对字符进行编号完全可行。

上面的假设成为了现实，CharsetMapFile 其实就是所谓 字符集（Charset）。字符集就是对字符进行编号。所谓不同的字符集其实就是相同的字符由于编码规则不同，从而造成同一个字符其所对应的编号不同，则这两种编码规则，将产生不同的字符集。

例如： UNICODE 字符集，其中按字形顺序进行编码：

``` 
仨： 0x4EE8 
̩仩： 0x4EE9
仪： 0x4EEA
仫： 0x4EEB
们： 0x4EEC
```

这些字符都是 亻偏旁，进行编码。


对于 gb2312 使用将汉字，按照拼音进行排列，然后，开始逐个编码。

```
鞍：0xB0B0 （0x978D）
氨：0xB0B1 （0x6C28）
安：0xB0B2 （0x5B89）
俺：0xB0B3 （0x4FFA）
按：0xB0B4 （0x6309）
暗：0xB0B5 （0x6697）
```
这些字符都是按拼音 an 来进行编码。其中括号内的是该字符的 unicode 编码。

有了字符集的概念，就可以重新考虑，Hello.java 如何存储了。

```
// 使用 unicode 表示

	0x0074  0x0067              0x4F60
	  ↓       ↓                    ↓
	S t r i n g str = new String("你好");
	↑     ↑                         ↑
 0x0053 0x0069                   0x597D
```

上面的代码中的每一个字符都有一个2个字节的unicde码。那么，文件就可以直接将这些 unicode 代码保存起来。

```
S:0x0053 t:0x0074 r:0x0072 i:0x0069 n:0x006e g:0x0067 (空格):0x0020 
s::0x0073 t:0x0074 r:0x0072 (空格):0x0020 =:0x003d (空格):0x0020 
n:0x006e e:0x0065 w:0x0077 (空格):0x0020 S:0x0053 t:0x0074 r:0x0072 
i:0x0069 n:0x006e g:0x0067 (:0x0028 ":0x0022 
你:0x4F60 好:0x597D
":0x0022 ):0x0029 ;:0x003b

===>
unicode: 对应  UTF-16BE 存储。
0053 0074 0072 0069 006e 0067 0020 0073 0074 0072 0020 003d 0020 006e 0065 
0077 0020 0053 0074 0072 0069 006e 0067 0028 0022 4F60 597D 0022 0029 003b

utf-8:
5374 7269 6e67 2073 7472 203d 206e 6577 2053 7472 696e 6728 22e4 bda0 e5a5 bd22 293b
```

`String str = new String("你好");` 30个字符，所以，上面使用 unicode 编码存储，一共是 30 * 2 = 60 个字节。

但是，实际 Hello.java 中，存储却不是这样。Hello.java 文件使用的文件编码是 utf-8, 这是一种对 unicode 编码的线性映射编码。可以说 unicode 是一种编码规范，这种规范，将所有的字符进行了编码，分配了惟一的 UNICODE。 但是呢，由于存储空间的问题，例如，一部 100 万字的英文小说，如果使用 unicode 编码来存储，将浪费一半的空间。因为，英文字母和常用的标点符号的 unicode 编码中，前一个字节，总是 0x00,所以可以考虑，将其去掉，这样，存储空间将节省一半。所以就出现了，所谓 utf-8 编码，其实 utf-8,使用的是 unicode 的编码，然后，对编码进行了线性变换，这种线性变换后的存储将更加适合网络传输，更加安全。所以 utf-8 算是对 unicode 的一种具体实现。

其实，我们上面设计的 unicode 存储，就是 UTF-16BE (unicode 大端存储)存储方案。当我们，使用 windows 自带的 记事本（notepad.exe） 程序，保存 `String str = new String("你好");` 然后选择注意在保存时选择，UNICODE 格式。将保存成上面 unicode 格式的数据。

notepad 支持 4 种编码格式:

ANSI: 和系统设置的区域有关，简体中文环境下 ANSI 就是 GBK 编码
Unicode:            UTF-16LE unicode 小端
Unicode big endian: UTF-16BE unicode 大端
UTF-8: utf-8 编码

UTF-16LE 和 UTF-16BE 完全使用字符的 unicode 码，区别就是表示字符的 2 个字节的高位字节和底位字节的顺序。高位字节先存储为 big endian, 低位字节先存储为 little endian. 例如：'你' unicode 编码是： 0x4F60， 其高位字节是： 0x4F, 低位字节是：0x60, 则 0x4F60 就是大端存储，而 0x604F, 则是小端存储。

对于 java 语言来说，其 数值和字符串 都使用 Big Endian 的存储方式。

String 类的几个方法：

``` java
// 返回指定索引位置的字符。
public char charAt(int index) 
// 返回指定索引位置的 codepoint， codepoint 就是字符的 unicode 编码。
public int codePointAt(int index)
public byte[] getBytes(Charset charset) 
```

java 中的 char 表示的就是字符，'a' , '你', '!', '界' ，都是字符。那么，char 如何存储呢。 char == 字符，由上面的分析，可以字符可以由，字符集中的编码来表示，例如： '你': unicode: 0x4F60  gbk: 0xC4E3 。不同的字符集有不同的编码。在 java 在使用 Unicode big endian 来表示一个字符 <==> UTF-16 编码，也就是直接使用字符的 unicode 编码来代表一个字符。

也就说，在代码中：

``` java
// \u0030 表示 unicode 字符(char) 0,
// 所以下面的语句相当于
// int iii = (char)\u0030; <==> int iii = (char)0;
// 注意，下面的用法是推荐的，因为，这种写法其实是已经
// 和 char 和 unicode 没有关系也，
// 之所以下面的赋值，可以通过编译，是因为 \u0030 对于编译器来说
// \u0030 和 直接写 0 是完全等价的。
/ 字母 i: \u0069, n: \u006e, t: \u0074
// 所以 int <==> \u0069\u006e\u0074
// 所以下面的代码： \u0069\u006e\u0074 some = 123;
// 就相当于 int some = 123;
// 并且可以正常这样定义的变量，可以正常编码通过并运行。
int iii = \u0030;
// 将打印出 0
System.out.println(iii);

// 当 \u0030 使用 ' 包裹，表示 '\u0030' 是一个 char, 
// '\u0030' 是一个字符 '0', 而字符 '0'，使用 unicode来编码，
// 其对应的 unicode 码是 0x0030，也就是说 '0', 内部存储的字节是 0x0030
// 所以下面的赋值操作相当于： int qqq = (int)0x0030 = (int)48
// 也就是对 char 内部存储的数值进行了类型转换
// 所以自然，将其转换为一个整数 qqq = 48
// 但是，如果把 '\u0030' 赋值给一个 char 变量，
// 则打印出来的就是 0, 对于 char 变量来说 \u0030(也即：0x0030)
// 就是 0 的编码，而打印字符，自然打印的是
// 编码所对应的字符 '0', 而不是编码本身的值。
int qqq = '\u0030';
// 打印出 48
System.out.println(qqq);

// 也可以使用 unicode 来作变量的名称
// 使用 '你好' 的 unicode 编码
int \u4f60\u597d = 12;
System.out.println(\u4f60\u597d);

// 或者直接使用 中文 都是可以的
int 你好2 = 12;
System.out.println(你好2);
```

从变量的命名，也可以看到，java 最终将这些字面量（变量名）和 字符串，都以 UTF-16 编码方式存储。

所以，我们甚至可以完全使用 unicode 字符来编码：

``` java
// 注意下面的代码并不是乱码，只是使用 unicode 字符来编码
// 其含义和 int some = 123; 
// 每一个 unicode 编码都和上面的字符一一对应。
// 并且这个代码是完全可以编码通过的。正常运行的
// 并且如果再定义一个名称为 some 的变量，编译器会报出 重复的变量 异常。
\u0069\u006e\u0074\u0020\u0073\u006f\u006d\u0065\u0020\u003d\u0020\u0031\u0032\u0033\u003b
System.out.println(\u0073\u006f\u006d\u0065);
```

同时，对于字符串来说：

String a = "你好world世界";
<==>
a = new String({'你', '好', 'w', 'o', 'r', 'l', 'd', '世', '界'});
<==>
a = new String({'\u4f60', '\u597d', '\u0077', 'o', 'r', 'l', 'd', '世', '界'});

``` java
String ss = "你好";
System.out.println(ss);

char[] c0 = {'你', '好'};
String s0 = new String(c0);
System.out.println(s0);

// java 中的 char 默认使用 UTF-16 来表示字符，而 utf-16 编码完全和 unicode 对应
// 所以 char a = '\u4f60' ,代表 utf-16 的字符：你
char[] c1 = {'\u4f60', '\u597d', '\u0077', '\u006f', '\u0072', '\u006c', '\u0064',
		'\u4e16', '\u754c'};
String s1 = new String(c1);
System.out.println(s1);

// 对于 byte 数组，byte 本身和字符没有关系，只有指定的字符集，
// byte 才能够，通过字符集，byte的值才能够映射成字符集中某个字符的索引（当作这个字符集的索引使用）。
// 然后这个索引，可以通过字符集找到，其所对应的字符，
// 0x4f, 0x60, 0x59, 0x7d 这个四个字节，如果代表数值，则分别是：79， 96，89，125
// 现在指定了，b1 可以使用 字符集 "utf-16" 来进行解码（decode）
// 由 utf-16 的规则，2个字节对应一个字符，并且使用大端存储，所以
// b1[0], b1[1] 按大端形成一个索引： 0x4f60  这个索引在 utf-16 中对应字符： 你
// b1[2], b1[3] 按大端形成一个索引： 0x597d 这个索引在 utf-16 中对应字符： 好
byte[] b1 = {0x4f, 0x60, 0x59, 0x7d};
String s2 = new String(b1, Charset.forName("utf-16"));
System.out.println(s2);

byte[] b2 = {0x60, 0x4f, 0x7d, 0x59};
String s3 = new String(b2, Charset.forName("utf-16le"));
System.out.println(s3);

byte[] b3 = {(byte) 0xe4, (byte) 0xbd, (byte) 0xa0, (byte) 0xe5, (byte) 0xa5, (byte) 0xbd};
String s4 = new String(b3, Charset.forName("utf-8"));
System.out.println(s4);

// 同理 你 GBK 编码： 0xc4e3 
//      好 GBK 编码： 0xbac3
byte[] b4 = {(byte) 0xc4, (byte) 0xe3, (byte) 0xba, (byte) 0xc3};
String s5 = new String(b4, Charset.forName("gbk"));
System.out.println(s5);

// big5 编码：
byte[] b5 = {(byte) 0xa7, (byte) 0x41, (byte) 0xa6, (byte) 0x6e};
String s6 = new String(b5, Charset.forName("big5"));
System.out.println(s6);
```

Charset.defaultCharset() 方法可以获得默认字符集。

``` java
// String.getChars
// 这个方法，将 String 内部持有数据的 char数组，拷贝到 dst 参数中。
// 并没有对 value 做任何处理，这样就可以 反推出 value 默认的存储字符集。
// 
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}

// ch 中持有 String 内部 value 字符数组的数据
// 将 ch 打印出来
// 4f60 597d 77 6f 72 6c 64 4e16 754c
// '你' 的 utf16 编码是： 4f60,
// '界' 的 utf16 编码是： 754c
// 所以，这进一步印证了，Java 中的 char 使用是 utf-16编码存储。
String str = "你好world世界";
char[] ch = new char[str.length()];
str.getChars(0, str.length(), ch, 0);

```

## 源文件格式编码与 class 文件 编码

对于 java 源文件来说，可以使用不同类型的编码，例如 gbk, utf-8, 自然，其中的出现的字符串，也是 gbk ,utf-8编码。但是经过 javac 编译之后 生成的class 文件却严格使用 utf-16 编码，所以不同类型的文件，经过编码之后，其编码将发生转换。

but,问题来了, 当我们在 windows 命令行下面调用 javac 执行编译的时候，javac 如何知道当前 Hello.java 文件的编码格式呢？ 对于文本文件 Hello.java 来说，其文件中并没有关于，文件是什么编码格式的说明。所以，必须从外部，获得文件格式。 

javac 将使用当前系统环境的默认编码，ANSI，在 windows 平台上 ANSI 映射为 bgk 字符集。所以如果 Hello.java 使用的是  gbk 字符集，那么下面的编译过程将不会出现问题。

```
// Hello.java
public class Hello{

    public static void main(String[] args){

            System.out.println("源文件");
    }
}

// 当 Hello.java 保存为 gbk 编码格式时
// 下面的编译不会出现问题
javac Hello.java

// 当 Hello.java 保存为 utf-8 编码会出现问题：
$ javac Hello.java
Hello.java:5: 错误: 编码GBK的不可映射字符
            System.out.println("婧愭枃浠?);
                                    ^
Hello.java:5: 错误: 未结束的字符串文字
            System.out.println("婧愭枃浠?);
                               ^
Hello.java:5: 错误: 需要';'
            System.out.println("婧愭枃浠?);
                                       ^
Hello.java:7: 错误: 解析时已到达文件结尾
}
 ^
4 个错误

编译过程中，出现了问题，其实就是 javac 默认将 Hello.java 文件中出现的 

"源文件" 这三个字符当作 gbk 编码处理，然后，将其转换成 utf-16, 但是其实这三个字符是 utf-8 编码，所以误认 gbk 编码，然后按照，gbk 2 个字节一个字符去解析，结果发现，其中的某两个字节构成的字符在 gbk 中并没有可以映射的字符。

这样说吧： gbk 是2个字节构成一个字符，则应该使用 0x0000 - 0xFFFF 当作编码，但是对于 gbk 来说，并不是上面的这 65535 这个编码已经全部被使用了，所以必然存在，某个编码，假设： 0xE6F8(注意只是假设，但肯定存在这种情况) 这个编码在 gbk 中并没有任何字符和其对应。 
而凑巧上面的 utf-8 中包含 0xE6 , 0xF8 , 被误解析成了一个 gbk 字符。此时，由于我们知道，javac 并会把字符转换成 utf-16, 所以需要将它所认为的 gkb 字符 0xE6F8 转换成 utf-16, 而事实是 gbk 中并不存在编码为 0xE6F8 的字符，所以编译程序的字符转换失败了，编码当然不能通过了。所以出现：编码GBK的不可映射字符。
```

其实，上面使用 javac 编码 utf-8 源文件时，之所以出现问题，就是 javac 采用了默认的字符编码。 查看 javac 帮助，可知 javac 支持一个参数: encoding

```
// 直接在命令行 输入 javac , 然后 回车
-encoding <编码> 指定源文件使用的字符编码
```

所以，如果 代码源文件不同平台默认的字符集，则需要手动指定其编码即可，正常编码了。

``` 
// 还是上面的保存成 utf-8 格式的 Hello.java 文件
// 下面的命令可以成功编码 utf-8 源文件。
javac -encoding utf-8 Hello.java
```

类似地，可以将 Hello.java 使用 记事本，保存成 utf-16le 和 utf-16be 格式，然后
使用 javac 编译，同样也是编译不通过，所以需要显示指定，文件编码，

```
// utf-16le 和 utf-16be 编码的 Hello.java 文件
// 使用下面的命令可以成功编译
javac -encoding utf-16 Hello.java
```

如何查看当前系统所支持的所有字符集？

``` java
SortedMap<String,Charset> characters = Charset.availableCharsets();
for(String name: characters.keySet()){
	Charset charset = characters.get(name);
	// 字符集的名称
	System.out.print(name + " : " + charset + " ");
	// 字符集的别名
	System.out.println(charset.aliases());
}

// 几个常用的字符集
Big5 : Big5 [csBig5]
GB18030 : GB18030 [gb18030-2000]
GB2312 : GB2312 [euc-cn, x-EUC-CN, gb2312-1980, gb2312, gb2312-80, EUC_CN, euccn]
GBK : GBK [windows-936, CP936]
ISO-8859-1 : ISO-8859-1 [csISOLatin1, latin1, IBM-819, iso-ir-100, 8859_1, ISO_8859-1:1987, ISO_8859-1, 819, l1, ISO8859-1, IBM819, ISO_8859_1, ISO8859_1, cp819]
US-ASCII : US-ASCII [cp367, ascii7, ISO646-US, 646, csASCII, us, iso_646.irv:1983, ISO_646.irv:1991, IBM367, ASCII, default, ANSI_X3.4-1986, ANSI_X3.4-1968, iso-ir-6]
UTF-16 : UTF-16 [utf16, UnicodeBig, UTF_16, unicode]
UTF-16BE : UTF-16BE [X-UTF-16BE, UTF_16BE, ISO-10646-UCS-2, UnicodeBigUnmarked]
UTF-16LE : UTF-16LE [UnicodeLittleUnmarked, UTF_16LE, X-UTF-16LE]
UTF-32 : UTF-32 [UTF32, UTF_32]
UTF-32BE : UTF-32BE [X-UTF-32BE, UTF_32BE]
UTF-32LE : UTF-32LE [X-UTF-32LE, UTF_32LE]
UTF-8 : UTF-8 [unicode-1-1-utf-8, UTF8]
```

[标准网](http://www.biaozhuns.com/)

[区位码、gb2312、gbk编码之间的关系](http://blog.csdn.net/lcfeng1982/article/details/11482709)

[UTF-8 GBK GB2312 之间的区别和关系](http://www.66ac.com/Java/JavaSE/20121204/22.html)

[UTF-8, UTF-16, UTF-32 & BOM](http://unicode.org/faq/utf_bom.html)

[UTF-16 编码规范](https://www.ietf.org/rfc/rfc2781.txt)

[UTF-8 编码规范](https://tools.ietf.org/html/rfc3629)

[Character To Glyph Index Mapping Table](https://www.microsoft.com/typography/otspec/cmap.htm) 这篇文章中有讲到，如何将 character 映射到 字体文件的的某个具体的字符上。

[The 'cmap' table](https://developer.apple.com/fonts/TrueType-Reference-Manual/RM06/Chap6cmap.html)

## JDK 对各种字符集的支持，以及编码和解码

jdk提供了三个类用来处理各种字符集。

* 字符集类 Charset

	java.nio.charset.Charset
	
	> A named mapping between sequences of sixteen-bit Unicode code units and sequences of bytes. This class defines methods for creating decoders and encoders and for retrieving the various names associated with a charset.

* 字符编码类 CharsetEncoder

	java.nio.charset.CharsetEncoder
	
	> An engine that can transform a sequence of sixteen-bit Unicode characters into a sequence of bytes in a specific charset. 
	
* 字符解码类 CharsetDecoder

	java.nio.charset.CharsetDecoder
	
	> An engine that can transform a sequence of bytes in a specific charset into a sequence of sixteen-bit Unicode characters. 

这三个类都是抽象类，是各种具体的字符集的实现的共同基类。具体的字符集，需要继承 Charset 类，并实现下面两个方法。

``` java
public abstract CharsetDecoder newDecoder();
public abstract CharsetEncoder newEncoder();
```

newDecoder 获得具体实现字符集的解码类。
newEncoder 获得具体实现字符含棉编码类。

各种字符集的实现在：

sun.nio.cs 和 sun.nio.cs.ext 包中

sun.nio.cs 在 jre\lib\rt.jar 

sun.nio.cs.ext 在 jre\lib\charsets.jar

sun.nio.cs 包中有 UTF-8, UTF-16, UTF-16LE, UTF-16BE 等字符集的实现。

sun.nio.cs.ext包中有 GBK, BIG5 字符集的实现。

### CharsetProvider

java.nio.charset.spi.CharsetProvider 提供了 charsetForName 方法用来查找上面的具体字符集的实现。

### jvm 的默认字符集

使用 System.getProperty("file.encoding") 可以获得 JVM 的默认字符集的名称。对于输入，输出字符流，如果不提供字符集参数，则使用这个默认字符集。

同时 jvm 提供的 Charset.defaultCharset() 方法返回 jvm 默认的 Charset 对象（字符集对象），这个默认的 Charset 对象，就是使用 file.encoding 作为默认的字符集名称创建的。

所以不，file.encoding 设置成 gbk 时， Charset.defaultCharset() 将返回以 gbk 字符集。

默认字符集的作用，就是在涉及到 char 转 byte 的API，如果没有提供字符集，没有提供字符集，则将使用这个默认的字符进行 char 转 byte 的操作。涉及到 char 转 byte 的 API 有：String.getBytes, 还有常用的 System.out.println 方法的类 PrintStream，这个类，如果指定字符集，则 All characters printed by a PrintStream are converted into bytes using the platform's default character encoding. 

还有 FileWriter 类直接使用默认的字符集对字符进行编码：
> Convenience class for writing character files. The constructors of this class assume that the default character encoding and the default byte-buffer size are acceptable. To specify these values yourself, construct an OutputStreamWriter on a FileOutputStream. 

``` java
// 假设，启动 jvm 的时候，传递了参数
// -Dfile.encoding=gbk
// 并且 整个项目的编码是 utf-8, 
// 则 eclipse console 的默认编码也是 utf-8

// 当前是 utf-16 编码
String str = "abc测试字符123";
// 会使用 Charset.defaultCharset() 进行编码
// 将 utf-16 转换以默认的字符集，进行编码
// 如果 file.encoding 属性设置成 gbk 则
// 则  str 字符, 会将字符以 gbk 编码格式，
// 进行编码，使其转换成 gbk 编码字节，
// 所以对于显示设置，也需要改成 gbk 才可以正常显示。
// All characters printed by a PrintStream are converted into bytes 
// using the platform's default character encoding
// 由于使用 defaultCharset str 最终将转换成 gbk 字节流
// 当输出到 eclipse 的 console 时，由于 console 的编码是 utf-8
// 所以下面的输出将出现乱码
System.out.println(str);

// 将 out 流进行 osw 底层使用 System.out, 但是
// 对 osw 指定了转码时使用的默认字符集为 utf-8
// 所以当  file.encoding = gbk 时，
// eclipse console, 才能够正常显示出中文来
// 而不会因为使用默认的字符集，而转成 gbk, 而 eclipse console 所以使用的字符集
// 是在 Run ... Run Configurations ... Common ... Encoding 里面进行设置
// 默认应该是和 项目的默认编码一致。
OutputStreamWriter osw = new OutputStreamWriter(System.out, Charset.forName("utf-8"));
try {
	// 这里在 console 输出时，由于指定了编码 utf-8
	// 所以不会出现乱码。
	osw.write(str);
	osw.write('\n');
	osw.flush();
} catch (IOException e) {
	e.printStackTrace();
}

System.out.println(Charset.defaultCharset());
```


[UTF-8 and Unicode FAQ for Unix/Linux](http://www.cl.cam.ac.uk/~mgk25/unicode.html)

[如何改造 Linux 虚拟终端显示文字](https://www.ibm.com/developerworks/cn/linux/l-cn-termi-hanzi/)