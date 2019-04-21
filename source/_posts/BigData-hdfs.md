---
title: BigData-hdfs
date: 2016-11-23 14:51:46
---

## 大数据相关技术实现

### [spark](https://github.com/apache/spark)

使用 scala 和 Java 混合实现

### [storm](https://github.com/apache/storm)

使用 clojure 和 java 混合实现

### [kafka](https://github.com/apache/kafka)

使用 scala 实现。

### [hadoop](https://github.com/apache/hadoop)

hdfs, mapreduce, yarn 均是 Java 实现

### [hbase](https://github.com/apache/hbase)

使用 java 实现

### [zookeeper](https://github.com/apache/zookeeper)

使用 java 实现

### [netty](https://github.com/netty/netty)

使用 java 实现

### [thrift](https://github.com/apache/thrift)

提供各个语言的实现库，在 jvm 平台上，使用 java 实现。

### [protobuf](https://github.com/google/protobuf)

## 大数据相关的经验

### 消息队列

例如 kafka, 代码的健壮性体现在：

* 网络中断的情况如何处理

	例如，在捕获网络中断异常后，记录日志，保存状态信息进行，例如 读写的 offset , 设置读取尝试次数，在进行数次尝试之后，可以发送邮件和短信提醒。
	
	还可以启动一个定时任务线程，在网络中断的情况下，该定时任务，进行定时周期性调度，例如5分钟，检测网络中断情况。如果可以和 kafka 进行通信了，则重启任务。同时也可以发邮件提醒，任务已经恢复。

### storm

* 时间窗口

	流式数据处理，其实是一个一定时间窗口内的批处理。这里就要能够，正常处理时间窗口，窗口滑动，等等。

	在窗口滑动时，就是数据进行提交的时间。

* tuple 的处理确认

	如何保证，在处理过程中，流中的 tuple 出现异常，如何进行处理。

	storm 提供 ack 机制，可以用来实现 tuple 的确认。

### hdfs

hdfs文件存储的机制，并发的模型，都会在代码编写时，产生影响。

### others

还有一个思考角度，例如，在一个系统中使用了 kafka ,那么，如何进行 topic 的扩展，例如已经创建了 topic 且只有2个分区，如何扩展为4个分区且不影响现有的业务，也就是在这个过程中如何保证 producer 和 consumer 不被中断，不受影响。

同样地在 storm 中，当一个已经提交的 topology ， 如何将其中的 bolt 并发动态增加，

如何动态扩展集群等等。

### 日志信息

能够在代码关键点上，添加日志，便于分析处理。

### 大数据平台任务调度与监控系统


## 参考

1. [Apache Hadoop Main 2.7.3 API](http://hadoop.apache.org/docs/r2.7.3/api/index.html?overview-summary.html)
2. [写给大数据开发初学者的话](http://lxw1234.com/archives/2016/11/779.htm)
3. [How to change the parallelism of a running topology](http://storm.apache.org/releases/1.0.2/Understanding-the-parallelism-of-a-Storm-topology.html)
4. [kafka集群扩展以及重新分布分区](https://www.iteblog.com/archives/1611.html)