---
title: BigData-storm-zookeeper事件注册
date: 2017-5-20 13:53:52
---

## storm集群状态管理

### ClusterState

>  ClusterState provides the API for the pluggable state store used by the Storm daemons. Data is stored in path/value format, and the store supports listing sub-paths at a given path.
>  
> All data should be available across all nodes with eventual consistency.

storm 为其集群状态管理抽象出了 ClusterState 这个类。

也就是说只要是实现了 ClusterState 接口。就可以被 storm 用来进行状态管理。

与此同时，为了便于 storm 可以扩展使用其它的 ClusterState store。storm 进行了工厂方法的抽象，给出了一个 ClusterStateFactory 接口。

``` java
public interface ClusterStateFactory {
    
    ClusterState mkState(APersistentMap config, APersistentMap auth_conf, List<ACL> acls, ClusterStateContext context);

}
```

可以使用 `storm.cluster.state.store` 配置项，对 ClusterStateFactory 进行配置，
这个配置的值是实现了 ClusterStateFactory 接口的类的名称。其默认值是 `org.apache.storm.cluster_state.zookeeper_state_factory`， 这个类是使用 clojure 来实现的。

由 zookeeper-state-factory 类的实现来看，其使用 zookeeper 来实现 ClusterState 接口。可见 storm 默认使用 zookeeper 进行状态配置维护。

### zookeeper_state_factory 的实现

代码参考 zookeeper_state_factory.clj

``` clojure
;; ClusterStateFactory this 代表工厂对象，
;; Map conf 代表 default.yaml 和 storm.yaml 中的配置
;; Map auth-conf 表示认证所需要的参数
;; org.apache.zookeeper.data.ACL acls 访问权限列表
;; ClusterStateContext context 表示集群对象所处的外部环境，nimbus，supervisor, worker 等等。
(defn -mkState [this conf auth-conf acls context]
	;; ...
	;; ...
)

```

1. 创建 storm.zookeeper.root 目录
2. 创建 zookeeper client

	``` clojure
  (let [callbacks (atom {})
        active (atom true) ;; 标识当前 ClusterState 是否是正常激活状态。当 ClusterState 被关闭时 active 设置成 false
        zk-writer (zk/mk-client conf
                         (conf STORM-ZOOKEEPER-SERVERS) ;; storm.zookeeper.servers
                         (conf STORM-ZOOKEEPER-PORT)    ;; storm.zookeeper.port
                         :auth-conf auth-conf
                         :root (conf STORM-ZOOKEEPER-ROOT) ;; storm.zookeeper.root
                         :watcher (fn [state type path]  ;; 实现 callback 回调
                                    (when @active
                                      (when-not (= :connected state)
                                        (log-warn "Received event " state ":" type ":" path " with disconnected Writer Zookeeper."))
                                      (when-not (= :none type)
										;; 调用 callbacks 这个 Map 中存储的所有函数。
                                        (doseq [callback (vals @callbacks)]
                                          (callback type path))))))]

	;; ...
	;; ...
	)
	```

	创建一个 CuratorFramework 对象，并注册 一个回调函数，这个函数的使用就是当有 WATCHED 事件发生时，会调用一个 Map 中存储的所有函数。

	注意，如果创建 ClusterState 的是 nimbus，则再创建一个 zk-reader 进行读写分离。

	ClusterState 有一个 register 函数，就是用来注册，Watch 回调函数。其实现方法就是把回调函数添加到 上面的 callbacks 中，这样当事件发生的时候，回调就可以被调用。

3. 实现 ClusterState 对象

	使用上面创建好的 CuratorFramework （zk-writer） 来实现 ClusterState 的各种 API。

### 创建 CuratorFramework

zookeeper_state_factory 使用 CuratorFramework 这个 zookeeper 的 client library 对 zookeeper 进行操作。

storm 使用 zookeeper.clj 对 CuratorFramework 的操作进行了封装。

``` clojure
;; 用来创建 zookeeper client, CuratorFramework 对象。
;; Map conf 配置
;; String servers   storm.zookeeper.servers
;; String port      storm.zookeeper.port
;; 可选配置：
;; root      设置 根路径
;; watcher   callback function. 回调函数接受 3 个参数 [state type path]
;; auth-conf 认证信息
(defnk mk-client
  [conf servers port
   :root ""
   :watcher default-watcher
   :auth-conf nil]
  ;; 1. 创建 CuratorFramework 对象。
  (let [fk (Utils/newCurator conf servers port root (when auth-conf (ZookeeperAuthInfo. auth-conf)))]
    (.. fk
        (getCuratorListenable)
		;; 2. 添加一个 Listener 
		;; listener 的功能是，当发生 WATCHED 事件时，调用 watcher 回调函数
        (addListener
          (reify CuratorListener
            (^void eventReceived [this ^CuratorFramework _fk ^CuratorEvent e]
                   (when (= (.getType e) CuratorEventType/WATCHED)
                     (let [^WatchedEvent event (.getWatchedEvent e)]
                       (watcher (zk-keeper-states (.getState event))
                                (zk-event-types (.getType event))
                                (.getPath event))))))))
	;; 3. 启动 CuratorFramework
    (.start fk)
	;; 4. 返回创建好的 CuratorFramework 对象。
    fk))
```

## StormClusterState

在 cluster.clj 中定义了一个 StormClusterState 接口，这个接口定义了与 Storm 领域相关的 ClusterState 的一系列操作。其内部持有一个 ClusterState 的引用，用其实现状态维护。

nimbus, supervisor, worker 等等 daemon 进行都将使用 StormClusterState 接口，对 zookeeper 进行操作。

创建一个 StormClusterState 对象时，会注册一个回调函数。

``` clojure
(fn [type path]
                    (let [[subtree & args] (tokenize-path path)] ;; tokenize-path 将 path 按路径分开
                      (condp = subtree
                         ASSIGNMENTS-ROOT (if (empty? args)
											 ;; "/storms/assignments" 结点的回调
                                             (issue-callback! assignments-callback)
                                             (do ;; "/storms/assignments" 的孩子结点发生变化，参数是孩子结点的名称
                                               (issue-map-callback! assignment-info-callback (first args))
                                               (issue-map-callback! assignment-version-callback (first args))
                                               (issue-map-callback! assignment-info-with-version-callback (first args))))
						 ;; "/storms/supervisors" 结点的回调
                         SUPERVISORS-ROOT (issue-callback! supervisors-callback)
						 ;; "/storms/blobstore"
                         BLOBSTORE-ROOT (issue-callback! blobstore-callback) ;; callback register for blobstore
						 ;; "/storms"
                         STORMS-ROOT (issue-map-callback! storm-base-callback (first args))
						 ;; "/storms/credentials"
                         CREDENTIALS-ROOT (issue-map-callback! credentials-callback (first args))
						 ;; "/storms/logconfigs"
                         LOGCONFIG-ROOT (issue-map-callback! log-config-callback (first args))
						 ;; "/storms/backpressure"
                         BACKPRESSURE-ROOT (issue-map-callback! backpressure-callback (first args))
                         ;; this should never happen
						 ;; 直接退出进程
                         (exit-process! 30 "Unknown callback for subtree " subtree args))))
```


`issue-callback!` 和 `issue-map-callback!` 函数将调用回调函数，并且，将当前回调函数移除掉，所以这里注册的回调函数是一次性的。当执行完成之后需要重新注册。