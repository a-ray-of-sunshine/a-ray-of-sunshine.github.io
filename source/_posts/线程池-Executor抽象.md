---
title: 线程池-Executor抽象
date: 2016-8-11 16:16:06
tags: [Executor]
---

``` java
public interface Executor {
    void execute(Runnable command);
}
```

Executor抽象的意义在于：将 任务的提交(task submission) 和 任务的执行(task execution) 进行了解耦合。It provides a standard means of decoupling *task submission* from *task execution*

使用 execute 方法来进行任务的提交，而任务的具体如何调度执行则由具体的 Executor 的实现类决定。不同的任务使用 Runnable 接口来描述。

任务执行 衍生出了 执行策略（Execution policies），执行策略对应的其实就是如何实现 execute 方法。不同的实现，代表了不同的执行策略。对于同一个实现类，可以通过传入参数的形式，来配置执行策略，使其能够表达不同的策略。配置策略的参数可以通过实现类的构造函数的参数传入。

例如：ThreadPoolExecutor JUC构架中提供的一个Executor：
``` java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}
```
通过传入的参数，决定了 executor 的不同行为。

这种解耦合的设计的意义在于，可以使得线程执行使用不同的策略，而使用不同的策略常常是具有工程意义的，因为在开发环境下面：机器配置并不高，可以使用相对于当前机器的配置，而到测试环境，生产环境中就可以依据机器的配置来，这样就可以以最大限度的使用机器的资源，而不需要进行代码的改变。只需要更改配置或者替换Executor就可以了。

从 Executor 抽象的设计上来看，对于一个模块的设计，并不是完成功能就可以了，还要从工程的角度去考虑问题，例如对于线程池的使用，开发的时候可能并不特别关注其配置。但是到了生产环境，就必须考虑这些问题了。所有才有了将 task submission 和 task execution 分离的抽象。使得，可以依据不同的环境配置不同的执行策略，而不用对原有的代码进行修改。这也符合OOP的开闭原则：对扩展开放，对修改关闭。通过实现不同的 Executor 来动态调整，线程池的行为。而不是修改当前代码。


