---
title: BigData-storm(2)
date: 2017-4-23 11:05:53
---

## storm 命令的使用

`bin/strom nimbus` 命令的执行过程

1. 执行 storm shell 脚本

	``` bash
	exec "$PYTHON" "${STORM_BIN_DIR}/storm.py" "$@"
	```

2. 调用 storm.py 脚本

	``` python
	java org.apache.storm.daemon.nimbus ## other args
	```

3. 执行 org.apache.storm.daemon.nimbus 类的 main 方法


## nimbus 的启动

nimbus 的核心实现在 src/clj/org/apache/storm/daemon/nimbus.clj

### 实现 org.apache.storm.scheduler.INimbus 接口

``` clojure
(defn standalone-nimbus []
  (reify INimbus
    (prepare [this conf local-dir]
      )

    (allSlotsAvailableForScheduling [this supervisors topologies topologies-missing-assignments]
      (->> supervisors
           (mapcat (fn [^SupervisorDetails s]
                     (for [p (.getMeta s)]
                       (WorkerSlot. (.getId s) p))))
           set ))

    (assignSlots [this topology slots]
      )

    (getForcedScheduler [this]
      nil )

    (getHostName [this supervisors node-id]
      (if-let [^SupervisorDetails supervisor (get supervisors node-id)]
        (.getHost supervisor)))
    ))
```

### 读取配置文件

读取 defaults.yaml, storm.yaml 及 storm-cluster-auth.yaml 配置文件，返回一个Map对象 conf

``` clojure
(defn -launch [nimbus]
  (let [conf (merge
               (read-storm-config)
               (read-yaml-config "storm-cluster-auth.yaml" false))]
  (launch-server! conf nimbus)))
```

* read-storm-config

	读取 defaults.yaml 和 storm.yaml 文件的配置，并使用 storm.yaml 覆盖 defaults.yaml 的配置。返回一个 Map 对象。

* read-yaml-config

	读取 storm-cluster-auth.yaml 配置文件。

### 启动 Server

``` clojure
(defn launch-server! [conf nimbus]
  ;; 验证 storm.cluster.mode: "distributed"
  ;; 而不是 "local"
  (validate-distributed-mode! conf)
  ;; 验证 nimbus.thrift.port 端口(默认是6627)是否被占用
  (validate-port-available conf)

  ;; service-handler 函数执行一系列初始化，然后返回一个
  ;; 实现了 Nimbus$Iface 接口的对象。
  (let [service-handler (service-handler conf nimbus)
		;; 创建 ThriftServer 对象。
        server (ThriftServer. conf (Nimbus$Processor. service-handler)
                              ThriftConnectionType/NIMBUS)]
    (add-shutdown-hook-with-force-kill-in-1-sec (fn []
                                                  (.shutdown service-handler)
                                                  (.stop server)))
    (log-message "Starting nimbus server for storm version '"
                 STORM-VERSION
                 "'")
	;; 启动 ThriftServer.
	;; 此时 nimbus 就可以对外提供服务了。
    (.serve server)
    service-handler))
```

上面的 ThriftServer 对象的过程使用 Java 代码实现就是。

``` java
// 实现 iface 接口。
Nimbus.Iface iface = service-handler(conf, nimbus);
// 创建一个 TServer 用来接收和响应请求。
ThriftServer server = new ThriftServer(conf, new Nimbus.Processor(iface), ThriftConnectionType.NIMBUS);
// 启动 TServer
server.serve();
```

TServer 的创建可以参考 org.apache.storm.security.auth.SimpleTransportPlugin.getServer 方法。

1. 创建一个在 TNonblockingServerSocket 对象。

	这个对象持有 nimbus.thrift.port , 当 Server 启动后，会在这个端口上进行监听。

2. 创建一个 ThreadPoolExecutor

	``` java
	// numWorkerThreads 为 nimbus.thrift.threads 默认 64
	// queueSize 为 nimbus.queue.size 默认 100000
	new ThreadPoolExecutor(numWorkerThreads, numWorkerThreads, 
                                   60, TimeUnit.SECONDS, new ArrayBlockingQueue(queueSize))
	```

由此可知，ThriftServer 的 serve 方法将在 nimbus.thrift.port 端口进行监听，一旦有请求过来。则 submit 到线程池 ThreadPoolExecutor 中进行执行。

也就是说，随着请求的到来，TServer 会创建 64 个线程，当 64 个线程都在运行的时候，新的请求将被添加到队列当前。直到有空闲线程来处理这个请求。

Nimbus 最终所对外提供的服务就是由 nimbus.clj 中的 service-handler 函数中实现的 Nimbus$Iface 接口。这个接口，最终将以 ThriftServer 的形式向外提供服务。

### Nimbus 对外提供的服务

nimbus 对外提供的服务定义在 storm.thrift 文件中。

``` thrift
service Nimbus {
  void submitTopology(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology) throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite, 3: AuthorizationException aze);
  void submitTopologyWithOpts(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology, 5: SubmitOptions options) throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite, 3: AuthorizationException aze);
  void killTopology(1: string name) throws (1: NotAliveException e, 2: AuthorizationException aze);
  void killTopologyWithOpts(1: string name, 2: KillOptions options) throws (1: NotAliveException e, 2: AuthorizationException aze);
  void activate(1: string name) throws (1: NotAliveException e, 2: AuthorizationException aze);
  void deactivate(1: string name) throws (1: NotAliveException e, 2: AuthorizationException aze);
  void rebalance(1: string name, 2: RebalanceOptions options) throws (1: NotAliveException e, 2: InvalidTopologyException ite, 3: AuthorizationException aze);

  // dynamic log levels
  void setLogConfig(1: string name, 2: LogConfig config);
  LogConfig getLogConfig(1: string name);

  /**
  * Enable/disable logging the tuples generated in topology via an internal EventLogger bolt. The component name is optional
  * and if null or empty, the debug flag will apply to the entire topology.
  *
  * The 'samplingPercentage' will limit loggging to a percentage of generated tuples.
  **/
  void debug(1: string name, 2: string component, 3: bool enable, 4: double samplingPercentage) throws (1: NotAliveException e, 2: AuthorizationException aze);

  // dynamic profile actions
  void setWorkerProfiler(1: string id, 2: ProfileRequest  profileRequest);
  list<ProfileRequest> getComponentPendingProfileActions(1: string id, 2: string component_id, 3: ProfileAction action);

  void uploadNewCredentials(1: string name, 2: Credentials creds) throws (1: NotAliveException e, 2: InvalidTopologyException ite, 3: AuthorizationException aze);

  string beginCreateBlob(1: string key, 2: SettableBlobMeta meta) throws (1: AuthorizationException aze, 2: KeyAlreadyExistsException kae);
  string beginUpdateBlob(1: string key) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  void uploadBlobChunk(1: string session, 2: binary chunk) throws (1: AuthorizationException aze);
  void finishBlobUpload(1: string session) throws (1: AuthorizationException aze);
  void cancelBlobUpload(1: string session) throws (1: AuthorizationException aze);
  ReadableBlobMeta getBlobMeta(1: string key) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  void setBlobMeta(1: string key, 2: SettableBlobMeta meta) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  BeginDownloadResult beginBlobDownload(1: string key) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  binary downloadBlobChunk(1: string session) throws (1: AuthorizationException aze);
  void deleteBlob(1: string key) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  ListBlobsResult listBlobs(1: string session); //empty string "" means start at the beginning
  i32 getBlobReplication(1: string key) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  i32 updateBlobReplication(1: string key, 2: i32 replication) throws (1: AuthorizationException aze, 2: KeyNotFoundException knf);
  void createStateInZookeeper(1: string key); // creates state in zookeeper when blob is uploaded through command line

  // need to add functions for asking about status of storms, what nodes they're running on, looking at task logs

  string beginFileUpload() throws (1: AuthorizationException aze);
  void uploadChunk(1: string location, 2: binary chunk) throws (1: AuthorizationException aze);
  void finishFileUpload(1: string location) throws (1: AuthorizationException aze);

  //@deprecated beginBlobDownload does that
  string beginFileDownload(1: string file) throws (1: AuthorizationException aze);
  //can stop downloading chunks when receive 0-length byte array back
  binary downloadChunk(1: string id) throws (1: AuthorizationException aze);

  // returns json
  string getNimbusConf() throws (1: AuthorizationException aze);
  // stats functions
  ClusterSummary getClusterInfo() throws (1: AuthorizationException aze);
  TopologyInfo getTopologyInfo(1: string id) throws (1: NotAliveException e, 2: AuthorizationException aze);
  TopologyInfo getTopologyInfoWithOpts(1: string id, 2: GetInfoOptions options) throws (1: NotAliveException e, 2: AuthorizationException aze);
  TopologyPageInfo getTopologyPageInfo(1: string id, 2: string window, 3: bool is_include_sys) throws (1: NotAliveException e, 2: AuthorizationException aze);
  ComponentPageInfo getComponentPageInfo(1: string topology_id, 2: string component_id, 3: string window, 4: bool is_include_sys) throws (1: NotAliveException e, 2: AuthorizationException aze);
  //returns json
  string getTopologyConf(1: string id) throws (1: NotAliveException e, 2: AuthorizationException aze);
  /**
   * Returns the compiled topology that contains ackers and metrics consumsers. Compare {@link #getUserTopology(String id)}.
   */
  StormTopology getTopology(1: string id) throws (1: NotAliveException e, 2: AuthorizationException aze);
  /**
   * Returns the user specified topology as submitted originally. Compare {@link #getTopology(String id)}.
   */
  StormTopology getUserTopology(1: string id) throws (1: NotAliveException e, 2: AuthorizationException aze);
  TopologyHistoryInfo getTopologyHistory(1: string user) throws (1: AuthorizationException aze);
}
```

## org.apache.storm.cluster.StormClusterState 接口

org/apache/storm/cluster.clj 文件中定义了一个接口 StormClusterState 这个接口是用来完成 storm 集群状态维护的核心。

其实现，由 cluster.clj 中的 mk-storm-cluster-state 函数。

mk-storm-cluster-state 的实现则是借助于 org.apache.storm.cluster.ClusterState

``` clojure
(defnk mk-storm-cluster-state
  [cluster-state-spec :acls nil :context (ClusterStateContext.)]
  (let [[solo? cluster-state] (if (instance? ClusterState cluster-state-spec)
                                [false cluster-state-spec]
                                [true (mk-distributed-cluster-state cluster-state-spec :auth-conf cluster-state-spec :acls acls :context context)])
        assignment-info-callback (atom {})

  ;; ...
  ;; 基于 cluster-state 实现 StormClusterState 接口。
)

;; 创建一个由 storm.cluster.state.store 配置指定的工厂类。
;; 这个工厂类需要实现接口 org.apache.storm.cluster.ClusterStateFactory
;; ClusterState mkState(APersistentMap config, APersistentMap auth_conf, List<ACL> acls, ClusterStateContext context);
;; mkState 就是用来返回 ClusterState 对象。
;; storm 默认提供的实现是 org.apache.storm.cluster_state.zookeeper_state_factory
;; 也就是状态信息是基于 zookeeper 来存储的。
;; 通过工厂方法的方式，ClusterState 的实现将可以实现动态可配置。
(defnk mk-distributed-cluster-state
  [conf :auth-conf nil :acls nil :context (ClusterStateContext.)]
  (let [clazz (Class/forName (or (conf STORM-CLUSTER-STATE-STORE)
                                 "org.apache.storm.cluster_state.zookeeper_state_factory"))
        state-instance (.newInstance clazz)]
    (log-debug "Creating cluster state: " (.toString clazz))
    (or (.mkState state-instance conf auth-conf acls context)
        nil)))

;; zookeeper_state_factory 实现的 ClusterState 则依赖于 zookeeper.clj
;; zookeeper.clj 中实现对 zookeeper 的操作。其实现也并不是直接使用 zookeeper 原生的 API，而是使用了一个 CuratorFrameworkd 这个框架来操作 zookeeper.
```

mk-storm-cluster-state 函数的主要功能。

1. 创建 storm 根目录

	创建 storm.zookeeper.root: "/storm" 目录.

2. 创建下面的状态目录

	``` clojure
    (doseq [p [ASSIGNMENTS-SUBTREE STORMS-SUBTREE SUPERVISORS-SUBTREE WORKERBEATS-SUBTREE ERRORS-SUBTREE BLOBSTORE-SUBTREE NIMBUSES-SUBTREE
               LOGCONFIG-SUBTREE BACKPRESSURE-SUBTREE]]
      (.mkdirs cluster-state p acls))
	```

3. 创建 cluster-state 对象


## service-handler 的实现

当启动一个 Nimbus 的时候，会调用 nimbus.clj 的 service-handler 函数，这个函数除了返回一个 实现 IFace 接口的对象。还会执行以下操作.



### 将当前启动的这个 Nimbus 添加到 nimbus 集群中

创建 Storm 相关的状态对象。将 NimbusSummary 序列化到字节数组中。将这些信息保存到 "/nimbuses/<nimbus-id>" 路径下。其中 <nimbus-id> 是 storm.local.hostname:nimbus.thrift.port. 注意这个 node 是一个 ephemeral_node 型的结点。

``` clojure
;add to nimbuses
(.add-nimbus-host! (:storm-cluster-state nimbus) (.toHostPortString (:nimbus-host-port-info nimbus))
  (NimbusSummary.
    (.getHost (:nimbus-host-port-info nimbus))
    (.getPort (:nimbus-host-port-info nimbus))
    (current-time-secs)
    false ;is-leader
    STORM-VERSION))
```

### addToLeaderLockQueue

将当前正在启动的这个 Nimbus 添加到 Leader 队列中。这会导致 Leader 选举。选出新的 nimbus 的 Leader。

### 注册 blobstore 回调

当  zookeeper 的 "/blobstore" 目录发生变化的时候。将新的 blob 同步下来。并且为当前所有的 blob 在 zookeeper 设置状态信息。

### 如果 Nimbus 是 Leader 的话，激活所有的 storms

``` clojure
;; 当前 nimbus 如果是 Leader 的话。
(when (is-leader nimbus :throw-exception false)
  ;; active-storms 方法获得 zookeeper 中的 "/storms/" 目录下面的所有子节点
  (doseq [storm-id (.active-storms (:storm-cluster-state nimbus))]
   ;; 更新子节点
   (transition! nimbus storm-id :startup)))
```

### 启动定时调度任务

1. do-cleanup

	nimbus.monitor.freq.secs （默认 10 秒）

2. clean-inbox

	Deletes jar files in dir older than seconds.

	以 nimbus.cleanup.inbox.freq.secs 为周期（默认 10 分钟），清理已经过期 nimbus.inbox.jar.expiration.secs （默认 1 个小时）的 jar 包。

	获得 storm.home/{storm.local.dir}/nimbus/inbox 目录。

	这个目录下面存储着 jar 文件。进行定时删除。

3. blob-sync

	在 不是 Leader 的 Nimbus 上执行 每隔 nimbus.code.sync.freq.secs: 120 （2分钟） 将 Leader Nimbus 上的代码同步下来。

	每一个 Nimbus 本地都存储着一份 blob. 同步 对于 Leader nimbus 来说，新的 topology 提交之后，会在 zookeeper 的路径上维护这些信息。

	然后 非 Leader 的 nimbus, 每隔 2 分钟，就比较一下本地的存储和 zookeeper 上的存储。然后将需要同步的 jar 下载下来。

	这里同步的文件是 "storm.home/{storm.local.dir}/blobs" 目录下面的文件。

4. clean-topology-history

	Schedule topology history cleaner。

	Deletes topologies from history older than minutes.

5. 启动状态报告线程。

	``` clojure
	(start-metrics-reporters conf)
	```

	可以由下面的属性进行配置。

	storm.daemon.metrics.reporter.plugins:
 	org.apache.storm.daemon.metrics.reporters.JmxPreparableReporter

## 总结

当我们说，我们启动了一个 nimbus 的时候，我们在说什么?

启动一个 nimbus, 首先创建一个 CuratorFramework 的 zookeeper 连接。并尝试去创建 storm 所需要的 zookeeper 相关路径。

然后，会将当前正在启动的这个 nimbus 添加到 nimbus 集群队列中，此时就会进行 Leader 的选举。如果有多个 nimbus，则此后，当前 nimbus 是 leader 还是 flower 已经确定。

启动相关定时任务。例如: 定时 清理 inbox 目录。定时，将 blobs 目录下面的文件从 Leader 上下载下来。

启动 ThriftServer 对外发布服务。