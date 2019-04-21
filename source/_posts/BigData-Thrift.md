---
title: BigData-Thrift
date: 2017-4-28 14:33:32
---



## 定义服务

Thrifty 提供了一套快速发布服务的编程抽象。使用其所定义的 IDL 编写 <service>.thrift ，通过编译生成目标语言的运行代码。其主要的核心是生成 TProcessor 的子类，以及提供一个 IFace 接口。

所以定义服务，其实使用 Thrift IDL 编写 <service>.thrift，然后使用工具生成目标语言的代码。

## Thrift 发布服务

对于发布服务方来说，所要做的就是：

* 实现 IFace 接口，完成服务的功能。
* 依据业务场景，并发，服务的吞吐量，从 Thtift 提供的 TServer 中选择合适的 Server，启动 Server，从而发布服务。

**Thrift 已经提供了多种 Server 模型，不同的 Server 适用于
不同的场景，首先需要根据具体的业务场景选择合适的 Server, Server 的选择关系到，服务运行起来之后的，并发量和吞吐量等，所以需要具体分析，具体选择，最终实现，既可以完成业务功能，又不至于浪费系统资源（主要是 socket 和 线程资源）**

**关于 Thrift 已经提供的多个 Server 的比较，可以参考：[Thrift Java Servers Compared](https://github.com/m1ch1/mapkeeper/wiki/Thrift-Java-Servers-Compared)**

如果 Thrift 提供的 Server 不能满足需求，可以通过继承 TServer 的方式，实现自己的 Server。


``` java
// Thrift 服务发布流程
// 1. 实现 IFace 接口
// 注意 IFace 接口在实现的时候，需要考虑，并发模型
// IFace 最终的并发模型在下面选择适当的 Server 的时候
// 也具有重要的参考意义

// 2. 创建 TProcessor 对象

// 3. !!! 选择一个适当的 Server !!!
TServer _server = new THsHaServer(server_args);

// 4. 启动 Server
_server.serve();
```

## Thrift 使用服务

发布服务生成源码的时候也会生成对应的客户端。

`public static class Client extends org.apache.thrift.TServiceClient implements Iface{...}`

选择适当的数据传输协议 TProtocol，就可以创建一个客户端。使用这个客户端，连接到服务。就可以像使用本地方法一样，使用远程服务。

Thrift 框架将参数的打包，网络传输，服务端的解析，等过程全部封装好了。有使用 Client 和实现 IFace 的时候都不需要关注这些，只需要实现核心业务。

``` java
// Thrift Client 使用流程。
// 1. 创建 TSocket
TSocket socket = new TSocket(_host, _port);

// 2. 创建 TTransport
TTransport _transport = new TFramedTransport(socket, maxBufferSize);

// 3. 创建 TProtocol
// Thrift 提供了多种 传输协议，Binary， JSON 等等，
// 这里的选择需要和发布的服务选择的协议相同。
TProtocol _protocol = new  TBinaryProtocol(_transport);

// 4. 创建 Client
Nimbus.Client _client = new Nimbus.Client(_protocol);

// 5. 使用 _client 调用服务
_client.submitTopology();
```

## Thrif 架构设计

### 功能分层

分层的设计，将复杂的功能，通过抽象以接口和抽象类的形式表达。各层之间的依赖通过，对象组合的方法完成，最终完成整个功能。

``` java
// 对于一个 Server 而言，只需要一个 TServerTransport
// 所以 TServerTransport 这个依赖通过，参数在 构造 Server 的时候传递进来
// 其 生命周期和 Server 是一样的。
TTransport client_ = serverTransport_.accept();

// 下面的这几个变量都是和处理特定请求有关，
// 就是说，它们的创建是需要 client_ 对象的。
// 而当创建 Server 的时候 client_ 对象并存在，所以
// 使用工厂方法(Factory Method)，来创建对象。
// 同时，使用工厂方法，可以方便地配置不同的实现。
processor = processorFactory_.getProcessor(client_);
inputTransport = inputTransportFactory_.getTransport(client_);
outputTransport = outputTransportFactory_.getTransport(client_);
inputProtocol = inputProtocolFactory_.getProtocol(inputTransport);
outputProtocol = outputProtocolFactory_.getProtocol(outputTransport);	 
```

### 设计模式的使用 --- 工厂方法

org.apache.thrift.server.TServer 及其子类的设计抽象，非常具有代表性。其 serve 方法的实现，也非常值得学习。

设计模式的使用通常在框架（Framework）和大型应用中，在这种场景下和我们编写一个普通的程序，完成特定的功能，有着非常在的不同。

当我们只是写一段代码来完成特定的功能的时候，我们需要什么对象，就可以直接 new (创建)这个对象。

然而在大型应用，尤其是框架中，我们需要这个对象，但是这个对象的实现，却应该由框架的使用者来选择或者实现，此时，为了完成框架的功能，我们有必须需要这个对象。这就是一个变化点，此时就可以使用 创建型模式（Creational Design Pattern），将对象创建的过程封装起来。

例如，使用工厂方法的模式，将 inputTransport 和 outputTransport 对象的创建过程封装到 TTransportFactory 工厂类中。然后，将工厂类作为一个依赖。工厂类在创建 Server 时候就可以依据具体的应用场景确定好的。同时也可以提供一些配置。

``` java
// 下面的代码摘自：
// org.apache.thrift.server.TThreadPoolServer.WorkerProcess.run()
// 是实现 org.apache.thrift.server.TThreadPoolServer.serve 方法的核心
public void run() {
  TProcessor processor = null;
  TTransport inputTransport = null;
  TTransport outputTransport = null;
  TProtocol inputProtocol = null;
  TProtocol outputProtocol = null;

  TServerEventHandler eventHandler = null;
  ServerContext connectionContext = null;

  try {
	// 为了实现框架（framework）的功能，需要下面的 5 个变量
	// 但是这 5 个变量能不能直接在这里 new 出来呢？
	// 答案是可以，可以直接 new 出来，而不是使用工厂方法，来获取这些对象
	// 但是从框架的扩展性上考虑，例如以下场景：
	// 我们要使用 TThreadPoolServer 类，创建一个 Thrift 服务
	// 但是，需要使用 protobuf 格式来进行数据编码
	// 也就是重新定义一种 TProtocol
	// 所以此时，希望这里直接硬编码的 inputProtocol
	// 能够被替换成新的，我们自己定义的 TProtocol
	// 如果是硬编码，则为了使用新定义的 TProtocol，就要 checkout 出
	// Thrift 的源码，修改下面直接硬编码的 inputProtocol
	// new 成我们新创建的对象。显然这样使用框架，简直是非常麻烦
	// 并且，需要重新编译我们所使用的框架，为后续框架的升级和兼容带来麻烦
	// 所以其实，现在的问题就是我们在框架核心逻辑实现中需要创建这个对象
	// 但是这个对象，但是这个对象不能由框架创建（硬编码，导致无法使用新的TProtocol）
	// 而且，这个对象，也不能由客户代码（使用框架的人）直接创建
	// 因为，创建这个对象时，需要框架代码中变量参数 client_，而这个变量参数
	// 是由框架的逻辑，动态产生的，例如。
	// TTransport client_ = serverTransport_.accept();
	// 这个 client_ 对象，是框架在接收到用户请求时产生的。
	// 所以创建这个对象的参数，客户代码也无法得到和创建。
	// 创建客户代码也不能直接创建这个对象。
	// 现在问题来了，服务器，由于不能硬编码，破坏扩展性来创建所需要的对象
	// 客户代码，由于无法得到创建这个对象时所需要的参数（client_）
	// ！！！ 双方都不能创建这个对象。！！！
	// 好现在要解决这个问题。从客户代码的扩展性需求出发
	// 客户代码所需要的扩展性其实就是客户代码能够指定创建对象的类型
	// 那么，我们可以将这个类型作为参数，传递给 TThreadPoolServer 对象
	// 此时 下面就可通过，反射的方式创建对象。问题，似乎解决了
	// 客户代码可以将类型作为参数传递给 TThreadPoolServer
	// TThreadPoolServer 类，有具有创建这个类型的对象所需要的参数
	// 此时就可以创建对象了。
	// 其实上面的方法已经将问题解决了，但是如果，我们要在对象创建的时候
	// 做一些特定的处理，例如对于 同一个 <host, ip> 的 Client
	// 始终使用同一个 inputProtocol 对象，而不再每次都创建一个新的对象
	// 此时，就需要在对象创建的时候，添加一定的逻辑，而是不能直接使用
	// Class.newInstance 方法来创建对象。
	// 可以这样想，我们其实需要的是客户代码给我们一个创建对象的方法（对象创建的逻辑封装在里面，自然客户代码可以指定对象的类型）
	// 然后，在这里将参数传递给这个方法，让这个方法返回一个我们需要的对象。
	// 但是 java 中不能直接传递方法（函数）作为参数，所以需要将这个方法封装到
	// 一个类中（工厂类），然后将这个工厂类作为参数传递给 TThreadPoolServer 类
	// 下面可以将创建对象需要的参数，传递这个类中的可以获取到我们所需要的对象的
	// 那个方法（工厂方法），从而获取到对象。
	// 由此，工厂方法就诞生了！！！
	// 工厂方法的设计模式使得，下面的代码的在不修改的情况下，就是可以实现
	// 水平扩展。
    processor = processorFactory_.getProcessor(client_);
    inputTransport = inputTransportFactory_.getTransport(client_);
    outputTransport = outputTransportFactory_.getTransport(client_);
    inputProtocol = inputProtocolFactory_.getProtocol(inputTransport);
    outputProtocol = outputProtocolFactory_.getProtocol(outputTransport);

	// TServer 实现的调度核心
	// 这也是一个框架的基本作用，封装特定的功能
    while (true) {
		
		// 这里使用到了局部变量 inputTransport， outputTransport
        eventHandler.processContext(connectionContext, inputTransport, outputTransport);

		// 这里使用到了局部变量 processor，inputProtocol， outputProtocol
        if(stopped_ || !processor.process(inputProtocol, outputProtocol)) {
          break;
        }
    }
  } catch (...) {
		// ......
  } finally {
		// ......
  }
}
```

**工厂方法的 Scope 是 Class ， 就是因为工厂方法的目地就是可以动态指定创建对象的类型（Class）**

## $.参考
1. [Thrift源码分析](http://www.kancloud.cn/digest/thrift/118983)
2. [Thrift Java Servers Compared](https://github.com/m1ch1/mapkeeper/wiki/Thrift-Java-Servers-Compared)