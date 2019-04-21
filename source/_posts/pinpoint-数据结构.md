---
title: pinpoint-数据结构
date: 2016-12-19 16:00:06
---

pinpoint 的业务对象（BO）在 pinpoint-commons-server 工程中。hbase 中每一个表都对应其中的一个类。

例如：AgentEvent 表，对应于 AgentEventBo 类。

pinpoint-commons-server 中的 BO 类，是被 web 和 collector 所共享的，对于 collector 来说，它执行插入操作，所以其 dao 层的 `AgentEventDao` 提供了 `void insert(AgentEventBo agentEventBo);` 方法。

而对于 web 层来说，其所主要功能是进行查询，所以其所提供的是 dao 层 `AgentEventDao` 主要是 `get` 方法，例如：查询全部数据，查询某个时间范围内的数据，查询某条记录等。

**com.navercorp.pinpoint.collector.handler 包就是 collector 用来处理 agent 发送过来的数据。进行数据入库操作**

## application list

web端在 com.navercorp.pinpoint.web.controller 包中。

* 查询应用列表

	url: /applications

	control: com.navercorp.pinpoint.web.controller.MainController

应用列表：

ApplicationIndex 表

该表相关的DAO类：com.navercorp.pinpoint.web.dao.hbase.HbaseApplicationIndexDao


com.navercorp.pinpoint.web.mapper.ApplicationNameMapper

有三个字段

1. name

	存储的agent启动时，指定的 applicationName 这个参数，代表监控的应用的名称
	
2. agentid

	agent启动时，指定的 agentID 参数，用来惟一标识一个agent.
	
3. timestamp

	agetn 启动的时间。
	
4. value

	这个参数表示表示 Application 的类型编码，在 com.navercorp.pinpoint.common.trace.ServiceType 接口有相关的描述。
	
	例如： 1010 表示的 tomcat 服务器， 1030 表示 jetty 容器。

## AgentEvent

表示 Agent 当前的运行状态。

`com.navercorp.pinpoint.web.vo.AgentEvent`

1. agentId

	表示 agent id
	
2. eventTimestamp

	事件发生的时间戳
	
3. eventTypeCode

	事件类型编码。这个编码是在 com.navercorp.pinpoint.common.util.AgentEventType 类中。
	
4. eventTypeDesc

	事件类型描述

5. hasEventMessage

	是否存在 eventMessage 事件消息
	
6. eventMessage

	事件消息。Object类型，依照 eventTypeCode 的不同，其中存储的数据也不同。

## AgentInfo

`com.navercorp.pinpoint.common.bo.AgentInfoBo`

`com.navercorp.pinpoint.web.vo.AgentInfo`

对应 agent 的详细信息，例如 agent 所在的主机名，ip, 还有 JVM 信息，Agent的状态，Agent的版本等等。

## AgentLifeCycle

表示 agent 的生命周期。

与这相关的类。

`com.navercorp.pinpoint.common.bo.AgentLifeCycleBo`

相关字段信息：

* version

	agent 版本
	
* agentId

	agent id
	
* startTimestamp

	agent 启动时间
	
* eventTimestamp

	事件发生的时间

* eventIdentifier

	事件标识
	
* agentLifeCycleState

	有下面几种状态。

	``` java
	RUNNING((short)100, "Running"),
    SHUTDOWN((short)200, "Shutdown"),
    UNEXPECTED_SHUTDOWN((short)201, "Unexpected Shutdown"),
    DISCONNECTED((short)300, "Disconnected"),
    UNKNOWN((short)-1, "Unknown");
	```

## AgentStat

表示 Agent 的基本情况。例如： 堆使用率，堆大小，jvm CPU使用率等等。

`com.navercorp.pinpoint.web.vo.AgentStat`

## ApiMetaData

`com.navercorp.pinpoint.common.bo.ApiMetaDataBo`

* agentId
* startTime
* apiId
* apiInfo
* lineNumber
* type

## ApplicationMapStatisticsCallee_Ver2

## ApplicationMapStatisticsCaller_Ver2

## ApplicationMapStatisticsSelf_Ver2

## ApplicationTraceIndex

在 collector 的 dao 层，可以看到在这个表中插入的数据是： TSpan 类型的。

``` java
// com.navercorp.pinpoint.collector.dao
public interface ApplicationTraceIndexDao {
    void insert(TSpan span);
}
```

## HostApplicationMap_Ver2

host, bindApplicationName, bindServiceType, statisticsRowSlot, parentApplicationName, parentServiceType

``` java
public class AcceptApplication {
    private final String host;
    private final Application application;

	// ...
}
```

## SqlMetaData_Ver2

可以通过分析 ` com.navercorp.pinpoint.collector.dao.hbase.HbaseSqlMetaDataDao.insert` 方法获得插入的数据。

sql 元数据

`com.navercorp.pinpoint.common.bo.SqlMetaDataBo`

## StringMetaData

存储常用字符串常量

* agentId

* stringId

* startTime

* stringValue

	需要存储的字符串值

## Traces

`com.navercorp.pinpoint.common.bo.SpanBo`

``` java
public interface TracesDao {
    void insert(TSpan span);

    void insertSpanChunk(TSpanChunk spanChunk);
}
```


## 关键url

### 主界面应用拓扑

url: getServerMapData

controller: MapController

### 主界面，右侧图表

url: getScatterData

controller: ScatterChartController

### 事务

url: transactionInfo

controller: BusinessTransactionController

