---
title: network-URLConnection
date: 2016-12-28 16:38:49
---

## 1. URLConnection 对象的创建过程

摘自 rfc7230

> If the port subcomponent is empty or not given, TCP port 80 (the reserved port for WWW services) is the default.

Http 协议默认 80 端口

``` java
// URLConnection 的使用。
URL url = new URL("http://www.cnblogs.com");
// 依据 URL String 中的 schema
// 选择适当的 URLConnection 的子类，进行实例化
// 下面的 connection 实例，自然是 HttpURLConnection 对象
URLConnection connection = url.openConnection();

// 执行下面的命令的时候，会进行 DNS 查询
// 将 www.cnblogs.com 这个域名对应的 IP 查询出来
// 这个ip和http协议默认的端口（80）,将共同组成
// server 端的 Socket。有了这个 Socket，
// 客户端就可以和服务端进行通信了。
// connect 方法的作用：
// 1. 查询 DNS 将 域名解析成 IP
// 2. 创建 <serverIP, 80> 的TCP 连接 
connection.connect();

// 从创建连接到下面的获得数据，可能
// 出现过时的情况，例如 server 端默认 60 秒，后超时
// 则 server 端就会主动关闭 http 连接.
// 而当下面的请求，用来用来请求数据的时候，
// 上面创建的Socket已经连接超时了，则 HttpURLConnection
// 就会创建新的 socket 进行连接。获取数据。
InputStream inputStream = connection.getInputStream();

// 由于 http 协议，规定，连接的释放在
// 一次请求完就需要主动释放，所以
// 下面的 close 方法，调用时 tcp 连接可能已经被释放了。
inputStream.close();
```

可以使用 wireshark 工具来分析上面代码中出现的网络请求。

``` js
// 过滤 http 协议中主机是 "cnblogs.com" 的流量
http.host contains "cnblogs.com"

// 过滤 ip 42.121.252.58 的流量
ip.addr == 42.121.252.58

// 过滤 域名为 www.cnblogs.com 的 DNS 查询请求
dns.qry.name==www.cnblogs.com

dns.qry.name==www.cnblogs.com or ip.addr == 42.121.252.58
```

## 2. 实现

### openConnection

URL 的 openConnection 方法，最终是创建了一个 `sun.net.www.protocol.http.HttpURLConnection` 类，这个类继承自 `java.net.HttpURLConnection`

### connect

HttpURLConnection 的 `connect` 方法则负责创建一个 `sun.net.www.http.HttpClient` 对象。

然后调用这个 HttpClient 对象的 openServer 方法。

openServer方法，将创建一个 `Socket` 对象。并调用 `Socket` 的 `connect` 方法和 URL 中指定的 <server, port> 建立 socket 连接。

``` java
// HttpClient 创建过程中，最终调用 openServer 方法
public void openServer(String server, int port) throws IOException {
	// doConnect 方法做两件事情
	// 1. 创建一个 <server, port> 的 socket
	// 2. 调用这个 socket 的 connect 方法，创建 tcp 连接。
    serverSocket = doConnect(server, port);
    try {
        OutputStream out = serverSocket.getOutputStream();
        if (capture != null) {
            out = new HttpCaptureOutputStream(out, capture);
        }
        
        // 初始化 serverOutput 方法。
        /** Stream for printing to the server. */
    	//  public PrintStream  serverOutput;
    	// 向 serverSocket 传输数据就使用到它
        serverOutput = new PrintStream(
            new BufferedOutputStream(out),
                                     false, encoding);
    } catch (UnsupportedEncodingException e) {
        throw new InternalError(encoding+" encoding not found");
    }
    
    // 设置 tcp 参数。
    serverSocket.setTcpNoDelay(true);
}
```

### getInputStream

下面的整个过程是阻塞的，就是使用 socket 的 InputStream 的 read 方法进行数据的读取。

#### 1. 初始化 HTTP 请求头 

将请求参数封装到 MessageHeader 对象中。

#### 2. 发送 HTTP 请求 

使用上面的 serverOutput 将 MessageHeader 输出

#### 3. 等待 HTTP 响应

将响应数据解析出来，封装到 `HttpInputStream` 对象中和 `MessageHeader` 对象中

MessageHeader 存储 Http 响应头信息

HttpInputStream 存储传输过来的数据

#### 4. 返回 InputStream

返回上面解析好的 HttpInputStream

### Http 关闭连接


### Http 的超时

URLConnection 有下面两个方法。用来设置连接超时时间和，socket读取超时时间。

* setConnectTimeout
* setReadTimeout

这两个方法最终调用 HttpClient 类所持有的 Socket 方法的 setSoTimeout 来设置超时。

而 setSoTimeout 方法的实现，则是调用系统的 socket API `setsockopt` 来设置 `SO_RCVTIMEO` 和 `SO_SNDTIMEO` 进行超时设置。

> If the timeout is set to zero (the default) then the operation will never timeout.

对于 URLConnection 的实现来说，其默认的超时值就是 0, 所以也就是不会超时。

Keep-Alive 保持连接。可以让一个请求连接，被复用多次。当创建一个 HTTP 请求的时候，其底层对应的是一个 TCP 连接。在 http 1.0 时，一个 http 响应完成之后其所对应的 tcp 连接就立即释放了。

但是，对于一个页面，可能出现大量的资源需要请求，例如图片，js, css 等等文件。所以创建的 tcp 可以被复用的。因为创建一个tcp连接也是非常的耗时和耗费资源的。所以使用 keep-alive 来保持一个连接。

## $. 参考
1. [结合Wireshark捕获分组深入理解DNS协议](http://blog.csdn.net/zhaqiwen/article/details/18048791)
2. [DNS原理及其解析过程](http://369369.blog.51cto.com/319630/812889/)
3. [rfc2616-Hypertext Transfer Protocol](https://tools.ietf.org/html/rfc2616)
4. [Uniform Resource Identifier (URI): Generic Syntax](https://tools.ietf.org/html/rfc3986)
5. [Uniform Resource Identifiers (URI): Generic Syntax](https://tools.ietf.org/html/rfc2396)
6. [Uri Class(c#)](https://msdn.microsoft.com/en-us/library/system.uri)
7. [HTTP - Hypertext Transfer Protocol](https://www.w3.org/Protocols/)