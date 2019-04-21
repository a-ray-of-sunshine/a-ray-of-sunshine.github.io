---
title: storm概念
date: 2016-7-4 10:53:22
tags: [storm, concepts]
---

## storm 核心概念

* Topologies
* Streams
* Spouts
* Bolts
* Stream groupings
* Reliability
* Tasks
* Workers

## Topologies

topology是对实时应用(realtime application)的逻辑的一种抽象。topology 类似于 MapReduce 中 job 的概念。它们之间一个重要的不同点在于 MapReduce 中的 job 最终会结束，而 topology 则会一直运行，除非显示地kill掉。topology 是一种通过流分组(stream grouping)将 spouts 和 bolts 连接起来的图。（ A topology is a graph of spouts and bolts that are connected with stream groupings. ）

和 topology 相关的类的
[org.apache.storm.topology.TopologyBuilder](http://storm.apache.org/releases/2.0.0-SNAPSHOT/javadocs/org/apache/storm/topology/TopologyBuilder.html)

## Streams

stream 是storm 中的核心抽象。stream 是由 tuple 构成的没有边界的序列，它在分布式环境中被并行的创建和处理。Streams are defined with a schema that names the fields in the stream's tuples. 默认情况下 tuple 可以使用原生数据类型，和实现了 Serializable 接口的类，这些数据类型的数据都可以通过 tuple 在 topology 中流动。

当声明流的时候，通常都会指定一个id。使用OutputFieldsDeclarer 来声明一个流。这个接口有两种类型的方法：declare 和 declareStream。其中 declare 其实是declareStream 的简化版本，不提供参数：streamId，其实使用 declare 方法时就是声明了一个 streamId = "default" 的流。当一个 spout 或 bolt 能够 emit 的流多于一个的时候，可以使用 declareStream 来指定待 emit 的流的id, 这个id 将在 emit 一个流的时候使用到。

backtype.storm.spout.ISpoutOutputCollector 接口：

``` java
List<Integer> emit(String streamId, List<Object> tuple, Object messageId);
```

backtype.storm.task.IOutputCollector 接口：

``` java
List<Integer> emit(String streamId, Collection<Tuple> anchors, List<Object> tuple)
```

这里的 steamId, 就是 OutputFieldsDeclarer 接口中声明的id.

### Tuple
所谓的流其实就是由 tuple 构成的数据流。

由上面的 ISpoutOutputCollector 接口可知，我们发送的数据是 List<Object> 类型的。而，backtype.storm.task.IBolt 也就是处理数据时的类型是：

``` java
void execute(Tuple input);
```

是 backtype.storm.tuple.Tuple 类型的。这个接口继承自
backtype.storm.tuple.ITuple 接口。backtype.storm.tuple.ITuple接口提供了，基于List<Object>的一些快捷的方法来获取 emit 的 List<Object> 中的对象。 而 Tuple 接口则更多地定义了一些关于数据发射源的信息，这些信息有助于Bolt在 execute 方法中进行特定的业务处理。例如：

``` java
		String streamId = tuple.getSourceStreamId();
		if(PageSpout.ACK_ID.equals(streamId)){
			// ...
		}else{
			// ...
		}
```

可以通过 Tuple 接口的 getSourceStreamId 方法来获取数据源流的id,然后，分别处理。所以 Tuple 的抽象其实，是对发射的数据 List<Object>的封装，这样包含了关于流的状态信息，Spout和Bolt发射的每一条 message,都会被包装成 Tuple，完全携带了相关这条消息及消息源的所有状态信息。

关于 Tuple 接口的实现类：backtype.storm.tuple.TupleImpl 这类的字段有：

``` java
    private List<Object> values;
    private int taskId;
    private String streamId;
    private GeneralTopologyContext context;
    private MessageId id;
    private IPersistentMap _meta = null;
```

其中第一个字段就是 emit 的数据 List<Object>

在实践中，emit 数据时通常不是直接使用 List, 而是使用storm 提供的一个 backtype.storm.tuple.Values 类。使用是：A convenience class for making tuple values using new Values("field1", 2, 3) syntax.

``` java
public class Values extends ArrayList<Object>{
    public Values() {
        
    }
    
    public Values(Object... vals) {
    	// 指定底层数组的长度，可以节省内存，减少数组copy的次数。
        super(vals.length);
        for(Object o: vals) {
            add(o);
        }
    }
}
```

## Spout

spout是topology中 stream 的源头。通常情况下 spout 从外部数据源（例如：数据库，kafka消息队列，磁盘文件等等）读取数据，然后，组装成 Values 对象，发射到 topology,topology则将其封装成 Tuple 对象发射到目地 Bolt中去。

spout 可以发射多个 stream:

Spouts can emit more than one stream. To do so, declare multiple streams using the declareStream method of OutputFieldsDeclarer and specify the stream to emit to when using the emit method on SpoutOutputCollector.

spout重要的方法是 nextTuple, 这个方法的作用就是向 topology 发射 tuple,如果没有新的 tuple 需要发射，则这个方法就可以直接返回。**nextTuple 方法的实现必需是非阻塞的,因为storm在调用spout的其它回调方法时都在同一个线程中的**

另外两个关于spout的重要的方法是 ack 和 fail，它们在实现 reliable spouts 时有重要的作用。

## Bolt

在 topology 中对 stream 的处理就是由一个个bolts来完成的。bolt 可以进行数据过滤，业务处理，聚合，连接，读写数据库等等。

bolt 可以发射多个 stream:
Bolts can emit more than one stream. To do so, declare multiple streams using the declareStream method of OutputFieldsDeclarer and specify the stream to emit to when using the emit method on OutputCollector.

bolt 最重要的方法就是 execute :

``` java
void execute(Tuple input);
```

它接受一个 Tuple 对象，经过处理之后，可以 emit Tuple对象。

当处理完成一个 tuple 对象的时候，Bolt就要调用 ack 方法：

 Bolts must call the ack method on the OutputCollector for every tuple they process so that Storm knows when tuples are completed (and can eventually determine that its safe to ack the original spout tuples). 
 
 处理 tuple 的通用方法是，也就是实现 execute 方法的基本过程是：

1. 基于参数 tuple, 处理之后 emit 0个或多个 tuple
2. 调用 ack 方法，来确认 tuple 已经被正确处理了。

对于第2点，storm 提供了一个IBasicBolt接口，实现了这个接口的bolt已经被自动ack过了，所以在实现 execute 方法时不需要 ack 了。

**Its perfectly fine to launch new threads in bolts that do processing asynchronously. OutputCollector is thread-safe and can be called at any time.**

在 bolt 在启动一个新的线程来异步处理tuple是可行的。因为 OutputCollector 是线程安全的。如果一个bolt在处理tuple时比较耗时，可以启动一个新的线程来提高，这个bolt的处理能力，

## stream grouping

如果说 spout 和 bolt 是 topology 中点，则 stream grouping 则是 topology 中连接这些点的线。stream group 其实就是体现了，component(spout和bolt)生成的stream 之间的订阅关系。同时流分组的一个重要的意义就是对流进行分类，聚合。其实这种分类聚合间接地承担了对 stream 进行处理的部分业务功能。

有关流分组的接口：

backtype.storm.topology.InputDeclarer<T extends InputDeclarer>

这个接口定义storm 所支持的所有stream group方式。

有关具体的各种分组方法，可以参考文档。

一种特殊的分组：自定义分组。
A custom stream grouping by implementing the CustomStreamGrouping interface.

``` java
T customGrouping(String componentId,
                 CustomStreamGrouping grouping)
```

you can implement a custom stream grouping by implementing the CustomStreamGrouping interface.
可以通过 实现 CustomStreamGrouping 接口来实现自定义分组。

## 分布式环境下设置 topology 的 classpath

在启动 worker 进程的时候，会将 topology.classpath 添加到 `-cp` 中。

这样的话就可以为拓扑提供不同的 classpath.

具体代码可以参考 [storm-server/src/main/java/org/apache/storm/daemon/supervisor/BasicContainer.java](https://github.com/apache/storm/blob/master/storm-server/src/main/java/org/apache/storm/daemon/supervisor/BasicContainer.java)

``` java
// storm/storm-client/src/jvm/org/apache/storm/Config.java
/**
 * Topology-specific classpath for the worker child process. This is combined to the usual classpath.
 */
@isStringOrStringList
public static final String TOPOLOGY_CLASSPATH="topology.classpath";

/**
 * Topology-specific classpath for the worker child process. This will be *prepended* to
 * the usual classpath, meaning it can override the Storm classpath. This is for debugging
 * purposes, and is disabled by default. To allow topologies to be submitted with user-first
 * classpaths, set the storm.topology.classpath.beginning.enabled config to true.
 */
@isStringOrStringList
public static final String TOPOLOGY_CLASSPATH_BEGINNING="topology.classpath.beginning";
```