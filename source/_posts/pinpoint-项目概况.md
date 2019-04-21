---
title: pinpoint-项目概况 
date: 2016-12-2 13:36:16
---

## 0. 项目概况

pinpoint 由 maven 进行管理

整个项目分为以下几个子项目：

* agent

	项目类型： jar
	
* bootstrap-core

	项目类型： jar

* bootstrap

	项目类型： jar

* collector

	项目类型： war

* commons

	项目类型： jar

* commons-hbase

	项目类型： jar

* commons-server

	项目类型： jar

* plugins

	项目类型： pom, 其下包含许多 plugin 项目

* profiler

	项目类型： jar

* profiler-optional

	项目类型： jar
	
* rpc

	项目类型： jar
	
* thrift

	项目类型： jar
	
* test

	项目类型： jar
	
* web

	项目类型： war
	
* java8-test

	项目类型： jar, 用来做集成测试

* hbase

	项目类型： pom


在项目根目录下面除了上面的项目，还下列文件夹：

1. doc

	doc 目录用来存放文档
	
2. quickstart

	> Pinpoint QuickStart provides a sample TestApp for the Agent to attach itself to, and launches all three components using Tomcat Maven Plugin.

### 存储

pinpoint 1.5.2 使用 hbase1.0.1

## 项目构建

1. 安装java

	java6, java7, java8 并设置 JAVA_6_HOME, JAVA_7_HOME, JAVA_8_HOME

2. 创建稳定版本的分支--用于开发

	`git checkout -b 1.5.2-dev 1.5.2`
	
	上面的命令，从当前的 tag 1.5.2, 创建一个分支，分支名称叫 1.5.2-dev

3. 编译

	`mvn install -Dmaven.test.skip=true`

## 项目部署

pinpoint 1.5.2 将配置信息存储在 zookeeper 中，所以需要安装 zookeeper

### 安装 zookeeper

### 安装 hbase

pinpoint 1.5.2 依赖数据存储使用 hbase-1.0.0, 所以首先，需要安装 hbase-1.0.0。

[hbase下载地址](http://archive.apache.org/dist/hbase/hbase-1.0.0/)

下载成功之后，配置 hbase

参考: [Hbase禁用自带ZooKeeper，使用已经安装的ZooKeeper](http://www.aboutyun.com/thread-7451-1-1.html)

### 项目配置

参考 doc/installation.md 这个文档，进行配置。需要配置的有三个组件

#### Collector

配置文件在 src/main/resources

1. pinpoint-collector.properties

	将 cluster.zookeeper.address 设置成 zookeeper 安装的机器的IP

2. hbase.properties

	设置连接 hbase 的地址。就是 hbase 连接的 zookeeper 的地址。
	
	hbase.client.host 设置成 zookeeper 的IP
	
	hbase.client.port 设置成 zookeeper 监听的端口，默认 2181

#### Web

配置文件在 src/main/resources

1. pinpoint-web.properties

	将 cluster.zookeeper.address 设置成 zookeeper 安装的机器的IP

2. hbase.properties

	设置连接 hbase 的地址。就是 hbase 连接的 zookeeper 的地址。
	
	hbase.client.host 设置成 zookeeper 的IP
	
	hbase.client.port 设置成 zookeeper 监听的端口，默认 2181

#### Agent

1. pinpoint.config

	需要配置 profiler.collector.ip ，表示 Collector 部署的机器的IP

##### agent 的使用

在启动jvm的时候，指定参数 javaagent, 这里 javaagent 就是就是 Agent项目编译之后的 pinpoint-bootstrap-1.5.2.jar 。

例如，需要监控 tomcat. 则可以使用下面的方法。

在tomcat的 bin/catalina.bat 文件中添加以下脚本：

``` shell
rem ============== pinpoint Agent
rem agent 解压之后的路径
set "AGENT_PATH=D:\deve\project\pinpoint\agent\target\pinpoint-agent-1.5.2"
rem agent的id, 自定义，只要不和已经存在的 agent 重复即可
set "AGENT_ID=99999"
rem agent监控的项目的名称，这个名称会出现在 web 界面的 Application List中
set "APPLICATION_NAME=testpin"

rem CATALINA_OPTS 参数最终会在 tomcat 启动的时候添加下面的参数。
set "CATALINA_OPTS="

set "CATALINA_OPTS=%CATALINA_OPTS% -javaagent:%AGENT_PATH%/pinpoint-bootstrap-1.5.2.jar"
set "CATALINA_OPTS=%CATALINA_OPTS% -Dpinpoint.agentId=%AGENT_ID%"
set "CATALINA_OPTS=%CATALINA_OPTS% -Dpinpoint.applicationName=%APPLICATION_NAME%"
echo %CATALINA_OPTS%
rem ==============
```

在 linux 下面部署agent时的，添加如下的配置：

``` bash
# ----- config agent -----
## agent 的安装位置
AGENT_PATH="/opt/pinpoint-agent"
## agent id 自定义
AGENT_ID="1188088"
## 监控的应用名称，自定义
APPLICATION_NAME="apm"
## agent的版本
AGENT_VERSION="1.5.2"

CATALINA_OPTS="$CATALINA_OPTS -javaagent:$AGENT_PATH/pinpoint-bootstrap-$AGENT_VERSION.jar"
CATALINA_OPTS="$CATALINA_OPTS -Dpinpoint.agentId=$AGENT_ID"
CATALINA_OPTS="$CATALINA_OPTS -Dpinpoint.applicationName=$APPLICATION_NAME"
# ----- config agent -----
```

### 项目编译

上面，所有的配置完成之后，开始进行项目的编译。

编译之前，需要设置，JAVA_6_HOME, JAVA_7_HOME, JAVA_8_HOME 3个环境变量，分别对应，安装的 jdk6, jdk7, jdk8

使用下面的命令进行编译。

`mvn install -Dmaven.test.skip=true`

### 项目启动

1. 启动 hbase

	`bin/start-hbase.sh`
	
2. 导入表结构

	`bin/hbase shell pinpoint/hbase/scripts/hbase-create.hbase`
	
3. 将项目部署到 tomcat

	将 pinpoint-collector 和 pinpoint-web 两个项目部署到 tomcat中。如果是使用 eclipse ，则需要将 pinpoint- 项目的Path设置成根路径。
	
	设置方法：在 pinpoint-web 右键 Properties --> Web Project Settings 将 Context root 设置在 "/" 即可。
	
4. 启动项目

	启动 tomcat , Collector 和 web 都会启动。
	
	启动完成之后，访问：`http://localhost:8080/`
	就可以看到 pinpoint 的 web 界面。

5. 启动 agent

	按照上面的方法，将 Agent 启动。
	
6. 查看监控信息
	
	刷新 web 界面，选择 Application List中，就出现上面启动 Agent 时指定的ApplicationName。选择这个应用，就可以看到，监控信息了。

## 日志

web 和 collector 的日志目录在 ${catalina.base}/logs/pinpoint/ 中分别对应 web.log 和 collector.log

agent 的日志目录在： to-agent-path/log/${pinpoint.agentId}-pinpoint.log


## $. 参考
1. [项目地址](https://github.com/naver/pinpoint)
2. [pinpoint wiki](https://github.com/naver/pinpoint/wiki)
3. [Technical Overview Of Pinpoint](https://github.com/naver/pinpoint/wiki/Technical-Overview-Of-Pinpoint)
4. [Maven仓库管理](https://my.oschina.net/aiguozhe/blog/101537)
5. [HBase 常用Shell命令](http://www.cnblogs.com/nexiyi/p/hbase_shell.html)