---
title: io包中的常用工具类
date: 2016-10-12 10:25:33
---

StreamTokenizer， RandomAccessFile

## 1. StreamTokenizer & StringTokenizer

两个分词器，StreamTokenizer 通过构建分词表，对流数据进行解析。StringTokenizer 的实现则更为简洁，约定一个分隔符（delimiter），对文本文件进行解析。

### 1.1 StreamTokenizer

#### 1.1.1 语法表 --- 字符类型

``` java
// 语法表
private byte ctype[] = new byte[256];

private static final byte CT_WHITESPACE = 1;
private static final byte CT_DIGIT = 2;
private static final byte CT_ALPHA = 4;
private static final byte CT_QUOTE = 8;
private static final byte CT_COMMENT = 16;
```

> Each byte read from the input stream is regarded as a character in the range '\u0000' through '\u00FF'. The character value is used to look up five possible attributes of the character: white space, alphabetic, numeric, string quote, and comment character. Each character can have zero or more of these attributes. 

#### 1.1.2 token类型

``` java
// 用来存储当前解析到的 token 的类型
public int ttype = TT_NOTHING;

/**
 * A constant indicating that the end of the stream has been read.
 */
public static final int TT_EOF = -1;

/**
 * A constant indicating that the end of the line has been read.
 */
public static final int TT_EOL = '\n';

/**
 * A constant indicating that a number token has been read.
 */
public static final int TT_NUMBER = -2;

/**
 * A constant indicating that a word token has been read.
 */
public static final int TT_WORD = -3;

/* A constant indicating that no token has been read, used for
 * initializing ttype.  FIXME This could be made public and
 * made available as the part of the API in a future release.
 */
private static final int TT_NOTHING = -4;
```

#### 1.1.3 StreamTokenizer 的 API 分类

* 构建符号表

	语法表中有5种字符类型，所以构建符号表的过程就是设置这 5 种类类型的字符的过程:

	* `public void whitespaceChars(int low, int hi)`
	
		在符号表中设置 CT_WHITESPACE 类型的字符，表示从 (0xff & low) 这个字符到 (0xff & hi) 这几个字符全部视为空白字符。
		
	* `public void parseNumbers()`
	
		在符号表中设置 CT_DIGIT 类型的字符，设置数字类型的字符表，数字类型由 0-9,-,. 构成。
		
	* `public void wordChars(int low, int hi)`
	
		设置单词表 CT_ALPHA, 这个可以用来构造语法中的关键字。例如使用这个类去解析 java 文件，可以将 int, double, void, static, final 等设置成一个 word.
		
	* `public void quoteChar(int ch)`
	
		设置字符串的标志 CT_QUOTE
		
	* `public void commentChar(int ch)`
	
		设置注释的标志 CT_COMMENT
		
	* `public void ordinaryChars(int low, int hi) &  public void ordinaryChar(int ch)`
	
		设置成变通字符，ctype 设置成 0.

* 清空语法表

	``` java
	// 由于符号表ctype默认值就全部是0
	// 所以下面的方法相当于将上面所有的设置全部清除
	public void resetSyntax() {
        for (int i = ctype.length; --i >= 0;)
            ctype[i] = 0;
    }
	```
* 设置状态位

	设置nextToken解析过程中使用的一些条件状态。
	
	* eolIsSignificant
	
		> Determines whether or not ends of line are treated as tokens. If the flag argument is true, this tokenizer treats end of lines as tokens; the nextToken method returns TT_EOL and also sets the ttype field to this value when an end of line is read. false indicates that end-of-line characters are white space.
	
	* slashStarComments
	
		> true indicates to recognize and ignore C-style comments.
		
		多行注释： /* ... */
		
	* slashSlashComments
	
		> true indicates to recognize and ignore C++-style comments.
		
		单行注释： // ...
		
	* lowerCaseMode
	
		如果为ture, 表示 nextToken 解析出的字符 sval 将被转换成小写

* 解析 token

	* `public int nextToken()`

	使用符号表ctype去解析流中的数据，直到找到一个 token 为止，返回这个 token 的类型。token 类型使用上面的 ttype 来表示。
	
	* `public void pushBack()`
	
	设置 pushedBack 状态位，使得下次调用 nextToken 时，直接返回当前解析好的数据。相当于把当前 token 退回到流中重新解析。

* 获取 token 信息

	* 获得 当前token 的值
	
	``` java
	public String sval;
	public double nval;
	```
	
	* 获得当前 token 的类型
	
	``` java
	public int ttype = TT_NOTHING;
	```
	
	* 获得当前行号
	
	``` java
	/** The line number of the last token read */
    private int LINENO = 1;
	public int lineno() {
        return LINENO;
    }
    ```

#### 1.1.4 nextToken的实现

1. 获得第一个字符的语法类型

	`c < 256 ? ct[c] : CT_ALPHA` 用来获得字符c所属的语法类型。可以看到在 0 - 255 这些个字符中直接从语法表获得字符类型，否则默认字符为 CT_ALPHA 类型，例如：如果在字符串，或者注释中出现了中文字符，则中文字符将默认为 CT_ALPHA 类型。

2. 按照第一个字符的语法类型进行分流解析

	* 对于每一种 token 其总是有特定的字符构成，例如数字类型可以这样解析：

	``` java
	// 不断地从流中读取字符，直到读取的字符不是 '.', '0'-'9' 则可以认为，当前
	// 这个数字 token 已经结束。然后把这个 token 转换成 double 类型即可。
	while (true) {
	    if (c == '.' && seendot == 0)
	        seendot = 1;
	    else if ('0' <= c && c <= '9') {
	        v = v * 10 + (c - '0');
	        decexp += seendot;
	    } else
	        break;
	    c = read();
	}
	```

	* 读取单词token 的思路就是，从一个字符开始，直到读取空白字符，这个 token 就可以认为是单词token结束。

	``` java
	// 如果当前字符是一个 CT_ALPHA 类型，则可以认为是单词的开始
	// 然后，不断的循环读取，
	if ((ctype & CT_ALPHA) != 0) {
	    int i = 0;
	    do {
	        if (i >= buf.length) {
	            buf = Arrays.copyOf(buf, buf.length * 2);
	        }
	        buf[i++] = (char) c;
			
			// 循环读取下一个字符，
	        c = read();
			// 如果 c<0,其实是流结束了，则 将 ctype = CT_WHITESPACE 会跳出
			// 这个循环， 否则使用 c < 256 ? ct[c] : CT_ALPHA 获取字符的
			// 的类型。
	        ctype = c < 0 ? CT_WHITESPACE : c < 256 ? ct[c] : CT_ALPHA;
	    } while ((ctype & (CT_ALPHA | CT_DIGIT)) != 0);
	    peekc = c;
	    sval = String.copyValueOf(buf, 0, i);
	    if (forceLower)
	        sval = sval.toLowerCase();
	    return ttype = TT_WORD;
	}
	```

	* 同样地，对于字符串token，就是由 "" 和 '' 包裹的字符序列
	* 注释类的token,就是 /* ... */ 和 // 开头的行。

* 构造 token

	从第一个字符开始，解析直到，遇到不符合这种 token 的字符出现就认为是 token 结束，然后将 token 解析的字符序列，设置到 sval 或者 nval 中。

	数值类型的token，其值被设置到 nval 中， 其它类型的 token 则被设置到 sval.

### 1.2 StringTokenizer

注意：这个类其实已经不推荐使用了：

> **StringTokenizer is a legacy class that is retained for compatibility reasons although its use is discouraged in new code.** It is recommended that anyone seeking this functionality use the split method of String or the java.util.regex package instead. 

这个类的功能和 StreamTokenizer 有点类似，但是其实现却完全不同，这个类仅仅是使用一个 分隔符（delimiter） 对字符串进行 split 操作。

例如： "edit:view:modify" 用来表达权限，现在需要获得具体的权限，需要对这个字符串进行按 ':' 进行分隔处理。就可以使用这个类。

``` java
String priv = "edit:view:modify";
StringTokenizer st = new StringTokenizer(priv, ":");
while(st.hasMoreTokens()){
	System.out.println(st.nextToken());
}
// 将输出：
// edit
// view
// modify
```

值得注意的是这个有一个方法，可以动态的调整分隔符。

``` java
// 参数 delim 将，作为新的分隔符，以后的解析会按照这个分隔符来进行
public String nextToken(String delim) {
    delimiters = delim;

    /* delimiter string specified, so set the appropriate flag. */
    delimsChanged = true;

    setMaxDelimCodePoint();
    return nextToken();
}
```

## 2. RandomAccessFile

RandomAccessFile 内部持有 FileDescriptor fd， 当调用构造函数创建 RandomAccessFile 对象的时候，会初始化 fd 对象。后面的 write 和 read 方法将使用这个 fd 对文件进行操作。

文件打开的时候，可以选择打开模式：

> * "r" Open for reading only. Invoking any of the write methods of the resulting object will cause an IOException to be thrown.  
>
> * "rw" Open for reading and writing. If the file does not already exist then an attempt will be made to create it.  
> 
> * "rws" Open for reading and writing, as with "rw", and also require that every update to the file's content or metadata be written synchronously to the underlying storage device.  
> 
> * "rwd"   Open for reading and writing, as with "rw", and also require that every update to the file's content be written synchronously to the underlying storage device.  

设置文件读写位置：

* seek

> Sets the file-pointer offset, measured from the beginning of this file, at which the next read or write occurs. The offset may be set beyond the end of the file. Setting the offset beyond the end of the file does not change the file length. The file length will change only by writing after the offset has been set beyond the end of the file.
