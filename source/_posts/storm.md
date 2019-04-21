---
title: storm
date: 2016-6-20 10:38:26
tags: [storm]
---

## 相关文档

[官网](http://storm.apache.org/)

[0.9.6 javadocs](http://storm.apache.org/releases/0.9.6/javadocs/index.html)

[单词计数示例](http://blog.csdn.net/wust__wangfan/article/details/50412695)

[storm-book示例](https://github.com/storm-book)

[storm源码分析之topology提交过程](http://www.aboutyun.com/thread-12372-1-1.html)

storm 用来完成：任务的并发提交，路由，调度，和执行。

## storm 编程中使用的核心 API

* backtype.storm.topology.IComponent
	
	拓扑结构中所有 component 的核心接口

* backtype.storm.spout.ISpout

	spouts 的接口

* backtype.storm.task.IBolt

	bolt 的接口
	
* backtype.storm.task.TopologyContext

	拓扑上下文信息
	
* backtype.storm.spout.SpoutOutputCollector

	This output collector exposes the API for emitting tuples from an backtype.storm.topology.IRichSpout. The main difference between this output collector and OutputCollector for backtype.storm.topology.IRichBolt is that spouts can tag messages with ids so that they can be acked or failed later on. This is the Spout portion of Storm's API to guarantee that each message is fully processed at least once.
	
* backtype.storm.task.OutputCollector

	This output collector exposes the API for emitting tuples from an IRichBolt. This is the core API for emitting tuples. For a simpler API, and a more restricted form of stream processing, see IBasicBolt and BasicOutputCollector.

## storm 并发机制
### 配置 executor 和 task

配置示例代码如下：

``` java
builder.setBolt(COUNT_BOLT_ID, countBolt, 2).setNumTasks(4)
```

其中 setBolt 调用的第3个参数，就是设置并发数，也就是executor(线程)数。

setNumTasks 设置 bolt 个数

相关参考文档：
[Parallelism of a Storm Topology](http://storm.apache.org/releases/1.0.1/Understanding-the-parallelism-of-a-Storm-topology.html)

## 启动 storm 集群

1. 启动 zookeeper

2. 启动 nimbus

2. 启动 supervisor

0. 启动 ui

0. 提交 topology

``` bash
storm jar storm-book-0.0.1-SNAPSHOT.jar {strom-topology-name} {stormname}
```

0. kill topology

``` bash
storm kill {stormname}
```

## storm 日志配置

配置文件在 ${STORM_HOME}/logback/cluster.xml

日志文件默认路径
${STORM_HOME}/logs

## storm componet的生命周期

参考：

[Storm bolt/spout生命周期](http://blog.csdn.net/nana5love/article/details/41524037)

[Twitter Storm源代码分析之Topology的执行过程](http://blog.csdn.net/nana5love/article/details/41524469)

[从storm-jdbc谈谈component的生命周期](http://my.oschina.net/ericquan8/blog/614279)


## 实时计算 相关参考

[实时计算的技术难点](https://segmentfault.com/a/1190000002686611)

[Storm 实现滑动窗口计数](http://f.dataguru.cn/thread-429705-1-1.html)

[Tick tuples within Storm](http://kitmenke.com/blog/2014/08/04/tick-tuples-within-storm/)

[Strom 连续多级批处理实践](http://weyo.me/pages/techs/storm-continuous-batch-process/)

[storm 实时计算](http://weyo.me/tag/shi-shi-ji-suan.html)

[流式统计的几个难点](https://segmentfault.com/a/1190000003048757)

[实时流Streaming大数据：Storm,Spark和Samza](http://www.jdon.com/bigdata/streaming-big-data-storm-spark.html)

[那些storm的坑坑](http://blackwing.iteye.com/blog/2147633)

[Storm核心概念剖析](http://iamzhongyong.iteye.com/blog/2194382)

[Storm Concepts](http://storm.apache.org/releases/current/Concepts.html)