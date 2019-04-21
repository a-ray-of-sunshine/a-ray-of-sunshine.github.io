---
title: BigData-storm
date: 2016-11-17 18:13:35
---

## storm

storm 底层使用 Thrift 进行通信。

## topology 的提交

使用org.apache.storm.StormSubmitter.submitTopologyAs 进行提交。

提交给谁：

使用： NimbusClient 进行提交。

NimbusClient 的 getConfiguredClientAs 方法进行 Nimbus 的查找。找到 nimbus 的 leader. 然后将 topology 提交到 nimbus 中。

topology 提交分两个阶段。

### 提交 jar 文件

### 提交 topology

## Nimbus的实现

org.apache.storm.security.auth.ThriftServer 用来接受请求。

## storm 的配置

**storm 在安装的时候需要配置主机名**
``` bash
## 将 HOSTNAME 改成需要的名称
vi /etc/sysconfig/network
HOSTNAME=node1

## 配置 hosts
vi /ets/hosts
127.0.0.1 node1
```

``` bash
## 启动 nimbus
bin/storm nimbus > /dev/null 2>&1 &
## 启动 supervisor
bin/storm supervisor > /dev/null 2>&1 &
```

配置文件在 $STORM_HOME/conf 目录中。关于 STORM_HOME 这个环境变量并不需要显示配置。当使用 bin/storm 的时候，会自动将当前运行的环境（STORM的安装位置）导出成 STORM_BASE_DIR 这个环境变量。同时还会导出二个环境变量：STORM_CONF_DIR（conf 目录） 和 STORM_CONF_FILE（conf/storm.yaml文件）。

``` bash
# bin/storm 脚本

## 导出 STORM_BASE_DIR 环境变量
STORM_BIN_DIR=`dirname ${PRG}`
export STORM_BASE_DIR=`cd ${STORM_BIN_DIR}/..;pwd`

## 导出 STORM_CONF_DIR 环境变量
export STORM_CONF_DIR="${STORM_CONF_DIR:-$STORM_BASE_DIR/conf}"

## 导出 STORM_CONF_FILE 环境变量
export STORM_CONF_FILE="${STORM_CONF_FILE:-$STORM_BASE_DIR/conf/storm.yaml}"

## 使用 python 执行 bin/storm.py 脚本
exec "$PYTHON" "${STORM_BIN_DIR}/storm.py" "$@"
```

bin/storm.py 脚本实现了 storm 提供的命令。

strom 提供了非常多的配置，其中在 storm-core.jar 中的 defaults.yaml 中存储这默认配置，这些默认配置，可以通过 配置 conf/storm.yaml 来覆盖。默认值可以在 [defaults.yaml](http://github.com/apache/storm/blob/v1.0.2/conf/defaults.yaml) 看到。同时其配置项的含义在： [Config](http://storm.apache.org/releases/1.0.2/javadocs/org/apache/storm/Config.html)

nimbus 启动时使用 nimbus.seeds 数据中配置的所有主机，中选取一个 leader。然后使用这个 Leader。启动 Nimbus 服务。

supervisor 启动的时候会启动 supervisor.slots.ports 中配置的端口来启动 相应数目的 worker 进程。这些进程会联系我们配置好的 zookeeper 集群。

zookeeper的作用。zookeeper 集群在启动的时候，会互相联系集群内的所有机器，直到整个集群启动稳定，zookeeper 会进行 Leader 的选举。Leader 选举成功之后，集群内的其它主机将成功 Follwer。Leader 将响应外部的请求，同时进行集群的高可用保证，例如将配置信息同步到其它的 Follwer上。一旦，此时的 Leader 出现问题。zookeeper 集群就会重新进行选举，从而保证了高可用（highly reliable）

Leader选举成功之后，对于 zookeeper 集群内的所有主机。自然都会知道这个 Leader 是谁。所以对于使用 zookeeper 的客户端来说，就可以联系 zookeeper 集群中的任何一个主机，来进行 Leader 的查询，客户端得到 Leader 主机的连接信息就可以进行将配置信息写入到 zookeeper 中。进行协调调度了。

nimbus 集群。storm 在 1.0.0 版本之前是不支持 nimbus 集群的[相关介绍](http://storm.apache.org/2016/04/12/storm100-released.html)，在 strom 1.0.0 之前，配置 nimbus 主机只使用 nimbus.host 指定一台主机，而 strom 1.0.0 之后 使用 nimbus.seeds进行主机配置，这是一个数组，用来提供 nimbus 集群的ip.

对 nimbus 作集群和对 supervisor 作集群的目地是不同的。nimbus 集群的目地是提供高可用，而supervisor 集群则是提供高吞吐量（提供性能）。

所以对于目地是高可用的集群（nimbus集群）在启动的时候需要进行 Leader 选举。而 高吞吐量集群（supervisor集群）则，不需要，高吞吐量是提高应用的响应请求的能力，所以集群内的主机，其作用是一样的，就是进行运算，所以它们是平等的，没有所谓主从关系。而高可用集群，其功能就是对外提供服务或者进行整体的协调调度功能，所以在任意时刻，集群内必须只有一个主机（master）可以对外提供服务（多个supervisor需要使用 nimbus 提供的服务，自然，需要确定一个nimbus来进行连接才可以），所以，为了高可用，需要对 nimbus 进行集群，为了对外服务，需要对 nimbus 集群内的主机进行 Leader 的选举。

## nimbus 启动

启动一个 TServer, 用来响应请求。

## supervisor 启动

## 提交 topology

topology的提交分成二个阶段，提交 Jar 文件 和 提交 topology.

其核心实现在 org.apache.storm.StormSubmitter.submitTopologyAs 方法中

### 提交 Jar 文件

``` clojure
;; nimbus.clj
;; 生成文件保存的路径 fileloc
;; 调用 Channels.newChannel(OutputStream out) 创建一个
;; WritableByteChannel 的 channel.
;; 并将这个 channel 对象 put 到 uploaders 这个 map 中
;; key 使用 fileloc
;; 返回 fileloc
(beginFileUpload [this]
(mark! nimbus:num-beginFileUpload-calls)
(check-authorization! nimbus nil nil "fileUpload")
(let [fileloc (str (inbox nimbus) "/stormjar-" (uuid) ".jar")]
  (.put (:uploaders nimbus)
        fileloc
        (Channels/newChannel (FileOutputStream. fileloc)))
  (log-message "Uploading file from client to " fileloc)
  fileloc
  ))

;; 调用 uploaders 的 get 方法，获得 location 所对应的 channel
;; 使用 channel 的 write 方法将 字节流写入到文件中。
(^void uploadChunk [this ^String location ^ByteBuffer chunk]
(mark! nimbus:num-uploadChunk-calls)
(check-authorization! nimbus nil nil "fileUpload")
(let [uploaders (:uploaders nimbus)
      ^WritableByteChannel channel (.get uploaders location)]
  (when-not channel
    (throw (RuntimeException.
            "File for that location does not exist (or timed out)")))
  (.write channel chunk)
  (.put uploaders location channel)
  ))

;; 文件上传完成，关闭 location 所对应的 channel.
;; 并把 uploaders 这个 map 中维护的 location 移除。
(^void finishFileUpload [this ^String location]
(mark! nimbus:num-finishFileUpload-calls)
(check-authorization! nimbus nil nil "fileUpload")
(let [uploaders (:uploaders nimbus)
      ^WritableByteChannel channel (.get uploaders location)]
  (when-not channel
    (throw (RuntimeException.
            "File for that location does not exist (or timed out)")))
  (.close channel)
  (log-message "Finished uploading file from client: " location)
  (.remove uploaders location)
  ))
```

### 提交 topology

``` clojure
;; 获得 component 的并行度。
;; 如何配置了最大并行度，则使用 max-parallelism 和配置的 任务并行度
;; 中较小的值。
(defn- component-parallelism [storm-conf component]
  (let [storm-conf (merge storm-conf (component-conf component))
        num-tasks (or (storm-conf TOPOLOGY-TASKS) (num-start-executors component))
        max-parallelism (storm-conf TOPOLOGY-MAX-TASK-PARALLELISM)
        ]
    (if max-parallelism
      (min max-parallelism num-tasks)
      num-tasks)))

;; common.clj
(defn num-start-executors [component]
  (thrift/parallelism-hint (.get_common component)))

;; thrift.clj
(defn parallelism-hint
  [^ComponentCommon component-common]
  (let [phint (.get_parallelism_hint component-common)]
    (if-not (.is_set_parallelism_hint component-common) 1 phint)))
```

* common.clj 

	封装一些常用功能

* thrift.clj

	提供 thrift 接口

* zookeeper.clj

	封装对 zookeeper 的常用, 如创建结点等等。

* zookeeper_state_factory.clj

	nimbus 对 zookeeper 的操作，都封装在这里
	使用 zookeeper.clj 完成常用功能

提交 topology 就是向， zookeeper 中创建相应的 node，例如创建 topology node, 等等，其它的 supervior 就可以通过监听这些node 来获得任务，进行执行。

``` clojure
(defserverfn service-handler [conf inimbus]

;; 向 zk 添加一个 data node
;; 并注册一个 ClusterStateListener
;; 一旦集群状态改变，则
;; 在 zk 的 /storm/nimbuses/<storm-id>
;; 添加一个结点。
;; 具体实现在 cluster.clj
add-nimbus-host!

)
```

## ui 启动

## 搭建Strom集群

## 参考

1. [storm download](http://storm.apache.org/downloads.html)
2. [setting up cluster](http://storm.apache.org/releases/1.0.2/Setting-up-a-Storm-cluster.html)
3. [Storm-源码分析汇总](http://www.cnblogs.com/fxjwind/p/3240218.html)
4. [Apache流计算框架详细对比](https://segmentfault.com/a/1190000004593949)
5. [Improving Twitter Search with Real-Time Human Computation](http://blog.echen.me/2013/01/08/improving-twitter-search-with-real-time-human-computation)
6. [实时计算的技术难点](https://segmentfault.com/a/1190000002686611)
7. [Implementing Real-Time Trending Topics With a Distributed Rolling Count Algorithm in Storm](http://www.michael-noll.com/blog/2013/01/18/implementing-real-time-trending-topics-in-storm/)
8. [你了解实时计算吗？](http://www.cnblogs.com/foreach-break/p/what-is-real-time-computing-and-how.html)