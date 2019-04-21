---
title: IO
date: 2016-8-22 11:42:55
tags: []
---

## openjdk 中关于 File io 的实现

[native 方法的实现](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/native/java/io/FileInputStream.c) 这是JNI 接口层。所有平台的公共抽象。

上面的实现是不同平台共有的实现，针对不同平台的实现，
[windows 平台](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/windows/native/java/io/io_util_md.c) 这是 windows 平台上的具体实现。

[有关 JNI 介绍](http://blog.csdn.net/jiangwei0910410003/article/details/17465457)

## 参考

[Java IO Overview](http://tutorials.jenkov.com/java-io/overview.html)

[Java NIO Tutorial](http://tutorials.jenkov.com/java-nio/index.html)

[深入分析 Java I/O 的工作机制](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/)

[openjdk-mirror/jdk7u-hotspot](https://github.com/openjdk-mirror/jdk7u-hotspot)

[openjdk-mirror/jdk7u-jdk](https://github.com/openjdk-mirror/jdk7u-jdk)

[jdk IO 实现](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/native/java/io/FileInputStream.c)

[Java™ Native Interface](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/)