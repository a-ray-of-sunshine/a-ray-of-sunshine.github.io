---
title: BigData-storm-任务分配及调度
date: 2017-5-21 14:39:16
---

## 构建拓扑

拓扑构建的核心是


``` java
private Map<String, IRichBolt> _bolts = new HashMap<>();
private Map<String, IRichSpout> _spouts = new HashMap<>();

// 添加一个 Spout
public SpoutDeclarer setSpout(String id, IRichSpout spout, Number parallelism_hint) throws IllegalArgumentException {
	// spout 和 bolt 的 id 不能相同
	// 因为 spout 和 bolt 最终要使用 id 作为 key 放置到
	// 同一个 Map 中。
    validateUnusedId(id);
    initCommon(id, spout, parallelism_hint);
    _spouts.put(id, spout);
    return new SpoutGetter(id);
}

// 添加一个 Bolt
public BoltDeclarer setBolt(String id, IRichBolt bolt, Number parallelism_hint) throws IllegalArgumentException {
    validateUnusedId(id);
    initCommon(id, bolt, parallelism_hint);
    _bolts.put(id, bolt);
    return new BoltGetter(id);
}

// 初始化 ComponentCommon 对象
private void initCommon(String id, IComponent component, Number parallelism) throws IllegalArgumentException {
    ComponentCommon common = new ComponentCommon();
	// 1. 设置 inputs 字段
    common.set_inputs(new HashMap<GlobalStreamId, Grouping>());
    if(parallelism!=null) {
        int dop = parallelism.intValue();
        if(dop < 1) {
            throw new IllegalArgumentException("Parallelism must be positive.");
        }

		// 2. 设置 parallelism_hint 字段
        common.set_parallelism_hint(dop);
    }

	// 3. 设置 json_conf 字段
    Map conf = component.getComponentConfiguration();
    if(conf!=null) common.set_json_conf(JSONValue.toJSONString(conf));
    _commons.put(id, common);
}

// SpoutGetter 和 BoltGetter
// 用来设置有关 Spout 和 Bolt 的配置信息
// 所有的配置信息，最终都被添加到 ComponentCommon 的 json_conf 字段中，
// 所以在 SpoutDeclarer 和 BoltDeclarer 的所有配置，最终被设置到
String currConf = _commons.get(_id).get_json_conf();
_commons.get(_id).set_json_conf(mergeIntoJson(parseJson(currConf), conf));
```

如果要对一个 Spout 或者 Bolt 进行配置，则可以有两种方法，一种方法是，override Spout 或者 Bolt 的 getComponentConfiguration 方法，返回配置信息的 Map 对象。第二种方法是，通过 SpoutDeclarer 和 BoltDeclarer 的 系列 set 方法进行设置，注意，当属性名相同的时候，第二种方法设置的属性将覆盖前面的。

对于 Spout 而言返回的 SpoutDeclarer 通常就是使用 set 方法设置属性。

Bolt 可以通过 BoltDeclarer 进行分组设置，BoltDeclarer 继承自 InputDeclarer，


``` java
// 4. 通过分组函数来初始化 inputs 字段
// org.apache.storm.topology.TopologyBuilder.BoltGetter.grouping
private BoltDeclarer grouping(String componentId, String streamId, Grouping grouping) {
	// 设置 _boltId 所对应的 ComponentCommon 的 inputs 属性
	// inputs 的类型是 HashMap<GlobalStreamId,Grouping>
	// 所以如果 对于同一个 Component 的同一个 stream 设置多次
	// 则以最后一次设置的为准
    _commons.get(_boltId).put_to_inputs(new GlobalStreamId(componentId, streamId), grouping);
    return this;
}
```

ComponentCommon 还有一个字段未初始化就是这个Component 可以发射出哪些流，这些流具有哪些字段。这些信息被存储在 streams 中。是在调用 TopologyBuilder.createTopology 的时候进行初始化的。

``` java
// org.apache.storm.topology.TopologyBuilder.getComponentCommon
private ComponentCommon getComponentCommon(String id, IComponent component) {
    ComponentCommon ret = new ComponentCommon(_commons.get(id));
    OutputFieldsGetter getter = new OutputFieldsGetter();
	// 调用 component 的 declareOutputFields 方法
	// 获得 component 可以发射的流的信息。
    component.declareOutputFields(getter);
    ret.set_streams(getter.getFieldsDeclaration());
    return ret;
}
```

StormTopology 的创建

``` java
public StormTopology createTopology() {
    Map<String, Bolt> boltSpecs = new HashMap<>();
    Map<String, SpoutSpec> spoutSpecs = new HashMap<>();
    maybeAddCheckpointSpout();
    for(String boltId: _bolts.keySet()) {
        IRichBolt bolt = _bolts.get(boltId);
        bolt = maybeAddCheckpointTupleForwarder(bolt);
        ComponentCommon common = getComponentCommon(boltId, bolt);
		// bolt 需要实现序列化
        maybeAddCheckpointInputs(common);
        boltSpecs.put(boltId, new Bolt(ComponentObject.serialized_java(Utils.javaSerialize(bolt)), common));
    }

    for(String spoutId: _spouts.keySet()) {
        IRichSpout spout = _spouts.get(spoutId);
        ComponentCommon common = getComponentCommon(spoutId, spout);
        // spout 需要实现序列化
        spoutSpecs.put(spoutId, new SpoutSpec(ComponentObject.serialized_java(Utils.javaSerialize(spout)), common));
    }

	// 创建 StormTopology 对象。
    StormTopology stormTopology = new StormTopology(spoutSpecs,
            boltSpecs,
            new HashMap<String, StateSpoutSpec>());

    stormTopology.set_worker_hooks(_workerHooks);

    return stormTopology;
}
```

### 拓扑的编译时结构 --- 静态

<table>
<caption>拓扑静态结构</caption>
<tr>
<th colspan=3></th><th></th><th></th><th></th><th></th>
</tr>
<tr>
<td>serialized_java:ByteBuffer</td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
</table>

### 拓扑的运行时结构 --- 动态

依据拓扑的编译时结构，再结合当前 storm 集群的实际资源进行 storm 的资源分配和调度

## Topology的配置

假设目前有一个 storm 集群， 其结构如下：

nimbus1, nimbus2, supervisor1, supervisor2, supervisor3

现在有一个拓扑 topology 在 supervisor1 上进行了提交，则当 topology 提交到 nimbus 之前，会收集如下的配置。

0. 调用 submitTopology 方法时传的 Map 中的配置
1. 在提交拓扑的时候的传入的 `-Dstorm.options` 参数中的配置
2. 添加一个 storm.zookeeper.topology.auth.payload 配置，默认随机生成
3. 添加 storm.zookeeper.topology.auth.scheme 配置，默认为 "digest"

由此可知，如果需要添加一些针对于 topology 的特定配置，则可以在创建拓扑的时候就在Map 对象中添加配置。同时如果需要传递大量的参数，可以将配置文件放置到 <storm>/conf/*.yaml 中 然后调用 `Utils.findAndReadConfigFile` 方法，但是这种方法要求配置文件必须是 yaml 格式的。

## Components 的配置

每一个 Bolt 和 Spout 都属于 Component, 可以通过 override `getComponentConfiguration` 方法返回一个 Component 级的配置。在创建拓扑的时候调用 setXXX 方法进行配置。