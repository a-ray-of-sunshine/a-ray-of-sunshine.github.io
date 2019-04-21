---
title: pinpoint-插件机制
date: 2016-12-12 14:17:15
---

## Agent 的启动

在 java 启动一个 JVM 时添加如下的命令行参数，即可将一个 jar 作为 agent 启动。

``` bash
-javaagent:jarpath[=options] 
```

对于一个 agent jar 包在其 manifest 文件，必须包含一个 Premain-Class 的属性，用来标志 Agent 的主类，

> The manifest of the agent JAR file must contain the attribute Premain-Class. The value of this attribute is the name of the agent class. The agent class must implement a public static premain method similar in principle to the main application entry point. **After the Java Virtual Machine (JVM) has initialized, each premain method will be called in the order the agents were specified, then the real application main method will be called.**

Premain-Class 类的 premain 方法被调用完成之后，应用的 main 方法才会被调用。

在 pinpoint 中，agent 是由 pinpoint/bootstrap 这个项目生成的 Jar 包。

查看 pinpoint/bootstrap 项目的 pom 配置：

``` xml
<!-- 摘自 pinpoint-bootstrap/pom.xml -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <executions>
        <execution>
            <phase>package</phase>
        </execution>
    </executions>
    <configuration>
        <archive>
            <manifestEntries>
                <Premain-Class>com.navercorp.pinpoint.bootstrap.PinpointBootStrap</Premain-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
                <Pinpoint-Version>${project.version}</Pinpoint-Version>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

由此可以 pinpoint agent 的入口就是 `com.navercorp.pinpoint.bootstrap.PinpointBootStrap` 类的 `premain` 方法

### 加载 boot/pinpoint-bootstrap-core-x.x.x.jar

通过 agent jar 的名字及 classpath 定位到 agent 所在的目录

然后定位 agent/boot/ 目录下的 pinpoint-bootstrap-core-x.x.x.jar 包的路径

最后，通过 `Instrumentation.appendToBootstrapClassLoaderSearch` 方法，将上面的路径添加到 classpath 中。

### 调用 PinpointStarter 的 start 方法

在 PinpointStarter 的 start 方法，中进行 agent 执行环境的初始化。

注意到 PinpointStarter 类中已经使用到了 pinpoint/bootstrap 和 pinpoint/commons 项目中的类，而在上一步中，已经将 pinpoint-bootstrap-core-x.x.x.jar 添加到 classpath 中了，所以可以正常使用 pinpoint/bootstrap 中的类，但是并没有将 pinpoint-commons-x.x.x.jar 添加进行，为什么也可以直接使用。

这和 pinpoint/bootstrap 项目的打包有关，在 pinpoint/bootstrap 最终的生成物 pinpoint-bootstrap-core-x.x.x.jar 已经将 pinpoint/commons 项目编译的文件全部打包到其中了，所以可以在  PinpointStarter 类可以正常使用 pinpoint/commons 项目中的类。

``` xml
<!-- 摘自 pinpoint-bootstrap-core/pom.xml -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>1.7.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <dependencyReducedPomLocation>${basedir}/target/dependency-reduced-pom.xml</dependencyReducedPomLocation>
                <filters>
                    <filter>
                        <artifact>com.navercorp.pinpoint:pinpoint-commons</artifact>
                    </filter>
                </filters>
            </configuration>
        </execution>
    </executions>
</plugin>
```

`maven-shade-plugin` 插件将 pinpoint-commons 项目编译生成的 class 文件也打包到 pinpoint-bootstrap-core 包中。


### 加载 plugin 中的 TraceMetadataProvider 服务

同样地， 获得 agent/plugin 目录，遍历其中的 jar， 使用

` ServiceLoader<T> serviceLoader = ServiceLoader.load(serviceType, classLoader);`

加载 插件 jar 包中的实现了 `com.navercorp.pinpoint.common.trace.TraceMetadataProvider` 接口的类

然后，调用这些类的 `setup` 方法。

``` java
// 通过  ServiceLoader.load 加载 plugin 目录下的 TraceMetadataProvider
for (TraceMetadataProvider provider : providers) {
    if (logger.isLoggable(Level.INFO)) {
        logger.info("Loading TraceMetadataProvider: " + provider.getClass().getName() + " name:" + provider.toString());
    }

    TraceMetadataSetupContextImpl context = new TraceMetadataSetupContextImpl(provider.getClass());
    // 这个 setup 方法最终会向 TraceMetadataLoader 类的
    // serviceTypeInfos 字段中注入，Plugin 所能提供的服务名及代码等信息 
    provider.setup(context);
}
```

### 启动 Agent

在 agent 启动的时候，可以传递一个命令行参数 `bootClass`，用来标识 Agent （一个实现了Agent接口的类），如果没有传递这个参数，则使用默认的实现：`com.navercorp.pinpoint.profiler.DefaultAgent`

启动 Agent 就是，调用 DefaultAgent 类的 `start` 方法。

``` java
public void start() {
    synchronized (this) {
        if (this.agentStatus == AgentStatus.INITIALIZING) {
            changeStatus(AgentStatus.RUNNING);
        } else {
            logger.warn("Agent already started.");
            return;
        }
    }
    logger.info("Starting {} Agent.", ProductInfo.NAME);
    
    // 启动 agent 信息发送线程
    this.agentInfoSender.start();
    this.agentStatMonitor.start();
}
````
