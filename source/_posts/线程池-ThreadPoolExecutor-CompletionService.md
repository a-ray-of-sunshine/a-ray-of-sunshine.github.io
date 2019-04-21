---
title: 线程池-ThreadPoolExecutor-CompletionService
date: 2016-8-20 16:23:27
tags: [ExecutorService, CompletionService]
---

## CompletionService

这个接口的主要使用是：将异步任务的生产和已经完成的任务的结果的消费进行了解耦。生产者可以使用 submit 方法来提交任务。而消费者则可以使用 take 方法来获取已经完成的任务。

## ExecutorCompletionService

``` java
public class ExecutorCompletionService<V> implements CompletionService<V> 
```

CompletionService 接口的一个实现。内部
``` java
private final BlockingQueue<Future<V>> completionQueue;
```
使用这个队列存储，Future。

实现的关键，对 task 进行了包装，QueueingFuture 类继承自 FutureTask
``` java
private class QueueingFuture extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task) {
        super(task, null);
        this.task = task;
    }
    protected void done() { completionQueue.add(task); }
    private final Future<V> task;
}
```

这个QueueingFuture类中 override 了 done 方法，这个方法在任务完成之后被调用，可以看到，所有完成的任务都将被添加到 completionQueue。


完成的任务的获取：

``` java
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }
```

直接从队列中获取。

ExecutorCompletionService 类是线程安全的。其中使用到的
``` java
private final Executor executor;
private final AbstractExecutorService aes;
private final BlockingQueue<Future<V>> completionQueue;
```
其实现都是线程安全的，所以这个类也是线程安全的。

ExecutorCompletionService 对 生产者 和 消费者 进行了解耦，使用不会相互依赖，依赖关系转化成了 生产者 和 消费者 都依赖于 ExecutorCompletionService 对象。ExecutorCompletionService 对象 是稳定不变的，所以 生产者 和 消费者可以依据其具体的业务变化而改变，而不会彼此影响。