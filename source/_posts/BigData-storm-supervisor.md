---
title: BigData-storm-supervisor
date: 2017-5-14 15:57:33
---

## storm supervisor 的实现

### 1. 保存 supervisor id

创建 {storm-local}/"isupervisor" 目录。以 "supervisor-id" 作为 key 。以 <long-num>.version 为文件。这个文件中存储着 SupervisorId. 这个 SupervisorId 就是一个 uuid, 使用 UUID.randomUUID 方法生成。

long-num，也就是文件名称的生成使用 当前系统的毫秒数，此后就使用这个毫秒数+1的方法生成 version.

核心参考：org.apache.storm.utils.LocalState。

提供了一种基于版本的状态存储机制。使用文件在存储一个 Map<String, TBase> 的 map 对象。每次向 Map 中添加（当然也包括修改）元素的时候，这个 Map 的版本号就会增加1. 这个文件存储在 {storm-local}/"isupervisor" 目录中。文件的名称是 <timestamep>.version, 此后，增加版本号，则新创建文件 <timestamep+1>.version

``` clojure
(prepare [this conf local-dir]
;; 保存配置到 conf-atom 变量中
(reset! conf-atom conf)

;; 生成 supervisor-id 将其保存到 LocalStore 中
;; 保存位置为 {storm-local}/"isupervisor"
(let [state (LocalState. local-dir)
      curr-id (if-let [id (ls-supervisor-id state)]
                id
                (generate-supervisor-id))]
  (ls-supervisor-id! state curr-id)
  ;; 保存 supervisor-id 到全局变量中。
  (reset! id-atom curr-id))
)
```

### 清理目录

清理 {storm-local}/supervisor/tmp 目录

``` clojure
(FileUtils/cleanDirectory (File. (supervisor-tmp-dir conf)))
```

### 设置 supervisor 的心跳

在 zookeeper 上 创建 "/supervisors/{supervisor-id}" 结点，设置其值为SupervisorInfo

心跳函数每隔 supervisor.heartbeat.frequency.secs: 5 秒，就执行一次。

### 恢复 supervisor

``` clojure
(doseq [storm-id downloaded-storm-ids]
  (add-blob-references (:localizer supervisor) storm-id
    conf))
;; do this after adding the references so we don't try to clean things being used
(.startCleaner (:localizer supervisor))
```

1. 获得 {storm-local}/supervisor/stormdist 目录下面的所有文件的名称


### 启动定时任务

启动两个事件管理器线程。

``` clojure
;; 这两个线程内部持有一个 LinkedBlockingQueue
;; 可以通过 EventManager 的 add 方法向这个 queue 中添加事件
;; 然后事件管理器线程就会执行这些事件。
[event-manager processes-event-manager :as managers] [(event/event-manager false) (event/event-manager false)]

;; 如果  supervisor.enable: true 设置为 true
;; 则 启动以下定时任务。
(when (conf SUPERVISOR-ENABLE)
  ;; This isn't strictly necessary, but it doesn't hurt and ensures that the machine stays up
  ;; to date even if callbacks don't all work exactly right
  ;; 1 & 2 这两个定时任务不是必须的，因为已经在 assignments-snapshot 函数
  ;; 中调用 (.assignments storm-cluster-state callback) 将 mk-synchronize-supervisor 
  ;; 返回的函数注册为回调函数，一旦 zookeeper 上的 /storm/assignments 目录发生变化时 
  ;; mk-synchronize-supervisor 函数就会被调用。 

  ;; 1. 每隔 10 秒执行一次 synchronize-supervisor 方法。
  ;; synchronize-supervisor 的作用：核心实现参考 mk-synchronize-supervisor
  ;;   1.1 下载已经分配给当前 supervisor 的 拓扑代码。
  ;;   1.2 更新一些状态变量
  ;;   1.3 添加一个 sync-processes 事件，这样使得新下载的 topology 能够被立即执行
  (schedule-recurring (:event-timer supervisor) 0 10 (fn [] (.add event-manager synchronize-supervisor)))

  ;; 2. 每隔 supervisor.monitor.frequency.secs: 3 秒 执行 sync-processes
  ;; 管理当前 supervisor 所启动的 worker 进程。如果有进程已经挂掉。则重新启动
  ;; 这个进程
  (schedule-recurring (:event-timer supervisor)
                      0
                      (conf SUPERVISOR-MONITOR-FREQUENCY-SECS)
                      (fn [] (.add processes-event-manager sync-processes)))

  ;; 3. 每隔 30 秒，执行一次 synchronize-blobs-fn
  ;; Blob update thread. Starts with 30 seconds delay, every 30 seconds
  (schedule-recurring (:blob-update-timer supervisor)
                      30
                      30
                      (fn [] (.add event-manager synchronize-blobs-fn)))

  ;; 4. 每隔 5 分钟执行一次健康检查
  (schedule-recurring (:event-timer supervisor)
                      (* 60 5)
                      (* 60 5)
                      (fn [] (let [health-code (healthcheck/health-check conf)
                                   ids (my-worker-ids conf)]
                               (if (not (= health-code 0))
                                 (do
                                   (doseq [id ids]
                                     (shutdown-worker supervisor id))
                                   (throw (RuntimeException. "Supervisor failed health check. Exiting.")))))))

  ;; 5. 每隔 30 秒进行一次 Runs profiler commands
  ;; Launch a thread that Runs profiler commands . Starts with 30 seconds delay, every 30 seconds
  (schedule-recurring (:event-timer supervisor)
                      30
                      30
                      (fn [] (.add event-manager run-profiler-actions-fn))))
```

## 参考
1. [Storm在zookeeper上的目录结构](https://segmentfault.com/a/1190000000653595)
