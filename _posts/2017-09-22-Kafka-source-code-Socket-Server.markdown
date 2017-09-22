---
layout: post
title:  "Kafka Source code: Socket Server"
date: 2017-09-22 15:10:00 +0000
categories: SourceCode, Kafka
---

### why kafka made there own socket server instead of using Netty 
From Quora I got somehow an answer which seem accetpable  

https://www.quora.com/Why-did-Kafka-developers-prefer-to-implement-their-own-socket-server-instead-of-using-Netty-Does-that-help-with-performance-Does-Kafka-implement-such-features-already

Existing answers say that performance drove the decision - which is absolutely correct, but is only part of the reason.

The rest of the reason is dependencies.

In the old days, before Kafka 0.8.2. came out, Kafka server and clients were one big library - in order for your app to consume or produce anything from Kafka, you needed to include the whole thing as a dependency. Which means that we had to be very very careful not to make Kafka depend on any library that an application acting as Kafka client might depend on - because if we do, we may each depend on a different version and the road to compatibility hell is very short from there.

So we stayed the hell away from any library that can possibly cause conflicts and implemented way too much ourselves. Kafka’s very own metric’s library, avoiding the popular Curator library for ZK, etc.

These days we are still super strict on dependencies that go into client and common package, and a bit more lenient about the server.

Was it the right choice? It’s hard to know for sure. Performance and no dependency-hell is great! But we are paying the price over and over - our own security layers (which Netty would have provided), very odd bugs caused by changes in low level networking code in different operating systems that take forever to debug (and we assume Netty already solved…).


#### NIO 框架架构
虽然这部分感觉和 netty 应该基本差不多，但是但是我还是想分析一下看一下。 首先看一下传统的nio 架构 netty 以及 mina. 

#### 先介绍一下 reactor 模式 
The Reactor Pattern

The reactor pattern is one implementation technique of event-driven architecture. In simple terms, it uses a single threaded event loop blocking on resource-emitting events and dispatches them to corresponding handlers and callbacks.

reactor 模式是一种 event-driven 架构的实现。 简单地说， 它使用一个单线程来循环监听由 resource 发出的事件，并且将这些事件交给相对应的 handler 以及 callback。

There is no need to block on I/O, as long as handlers and callbacks for events are registered to take care of them. Events refer to instances like a new incoming connection, ready for read, ready for write, etc. Those handlers/callbacks may utilize a thread pool in multi-core environments.

注意这里是不需要block I/O的， 因为 handler 以及callback 会处理这些。 此外所谓的event 在现实中可以是  new incoming connection, ready for read , ready for write 等等。 handler/callback 一般是会运行在多线程环境。

This pattern decouples the modular application-level code from reusable reactor implementation.

There are two important participants in the architecture of Reactor Pattern:

这种模式有一个很大的好处是 将 应用层 的代码 和具体的 reactor 的时间分离开来。 这种模式下 主要功能分为两部分。

1. Reactor

A Reactor runs in a separate thread, and its job is to react to IO events by dispatching the work to the appropriate handler. It’s like a telephone operator in a company who answers calls from clients and transfers the line to the appropriate contact.
2. Handlers

A Handler performs the actual work to be done with an I/O event, similar to the actual officer in the company the client wants to speak to.

A reactor responds to I/O events by dispatching the appropriate handler. Handlers perform non-blocking actions.


reactor 是一个独立的线程， 它除妖的工作就是针对 IO 的时间做出反应，并将其发送给对应的 handler。 可以想象成一个公司的接线员。

handler 则是做附体工作的地方。 

总的来说就是 reactor 负责 分发， handler
负责做 non-blocking的事件。


The Intent of the Reactor Pattern

The Reactor architectural pattern allows event-driven applications to demultiplex and dispatch service requests that are delivered to an application from one or more clients.

Reactor 架构让application可以多路复用的处理客户端发送过来的application 

One reactor will keep looking for events and will inform the corresponding event handler to handle it once the event gets triggered.

reactor会一直监控 event， 并且同事handler 去处理。

The Reactor Pattern is a design pattern for synchronous demultiplexing and order of events as they arrive.


It receives messages, requests, and connections coming from multiple concurrent clients and processes these posts sequentially using event handlers. The purpose of the Reactor design pattern is to avoid the common problem of creating a thread for each message, request, and connection. Then it receives events from a set of handlers and distributes them sequentially to the corresponding event handlers.

reactor 接受从多个client来的消息，request，或者链接请求。这种方式可以避免为每一个message，request 或者connection创建独立的线程。而是再一系列的handler之间接受和发送event。


举个例子 

![image](https://lh3.googleusercontent.com/DF3UyCf7C8Y548UJggA5tketo3yL6BTD7WAASGftTJgvS7PHYggXLMLG-A82QlV9kswPvJPoysmHfLHSJwi3pHbzQI21tO-BidgmPWPjY57hbSvg9x2g6L9KwYZ5ekA39hStgKHCOw)



In Summary: Servers have to handle more than 10,000 concurrent clients, and threads cannot scale the connections using Tomcat, Glassfish, JBoss, or HttpClient.


在如上的模式下， server 可能会收到超过 10,000个并行的client，但是由于系统的限制，我们不可能在服务器里面创建 10,000个线程去处理。

![image](https://lh4.googleusercontent.com/2iHLg2EtmznV_RNkO2X_kMWF07_SdZMDIRsTN1nle6YoZrpTZc1ZDfDVTor5keXAPUh1HWxU3OgbHUfEfsUufvgu3w2ay9o-Ae464y74QkalMalXlQqoKVorWpZ09GKWWRmh2vjVfA)

Basically, the standard Reactor allows a lead application with simultaneous events, while maintaining the simplicity of single threading. 

简单的说，用reactor的方法可以让 application只用一个线程但是处理并发的event

A demultiplexer is a circuit that has an input and more than one output. It is a circuit used when you want to send a signal to one of several devices.

demultiplexer 本来是的概念是一个基本电路，它允许一个输入，但是多个输出， 一般用来想将信号发给一个或者多个设备的情况。


This description sounds similar to the description given to a decoder, but is used to select between many devices, while a demultiplexer is used to send a signal among many devices.


听上去很像一个decoder，但是用来在在多个设备中进行选择。 

A Reactor allows multiple tasks which block to be processed efficiently using a single thread. The Reactor also manages a set of event handlers. When called to perform a task, it connects with the handler that is available and makes it as active.

Reacotr 可以用一个线程，让多个会阻塞的task 有效率的执行， 同时reactor 也会管理多个handler。当开始处理一个task的时候， ractor会去调用可以用的handler，并且将他们激活。（注意，handler才会去真正读数据，而不是reactor，这个是和生产者消费者不一样的地方。


The Cycle of Events:

- Find all handlers that are active and unlocked or delegates this for a dispatcher implementation.
- Execute each of these handlers sequentially until complete, or a point is reached where they are blocked. Completed Handlers are deactivated, allowing the event cycle to continue.


处理event的过程分为两部
- 首先是找到空闲的handler。 空闲的意思是，active， unlocked 或者专门做这件事情的。 
- 一次执行这些handler 知道结束或者被block。
 

### NIO的解释




### 实际例子 

实际中 reactor 一般都不是长成上面这个样子。 因为上面这种单线程会有一个很大的问题，就是所有的IO 处理都压在了一个线程上，当负载增加一些以后，就会出现性能问题。 Netty 现在使用的是 主从 reactor的线程模型。 具体可以参考reference 4中关于线程模型的介绍。 简单的说就是有两个 reactor 线程池， 一个叫做boss， 只负责 建立和关闭连接。 还有一个线程池叫做 worker， 负责 IO 操作以及调用 handler 处理逻辑。  


![image](http://wangtianzhi.cn/img/netty-%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B%E5%9B%BE.png)


一开始看netty 的时候始终不明白channel 的概念， 读了源代码发现， 所谓的 channel 其实就是一个 client 连接建立以后创建出来的东西。 一个channel 里面包含了这个连接的一些参数，比如 buffer 大小 ， handler 的数量等。


### Block 

按照上面的解释， 如果一个handler 是block 的，那么肯定会造成所有的这个worker线程里面的所有channel 都被block 住， 理论上这是不可以接受的。 Netty 的解决方法有两个

- DefaultEventExecutorGroup 
- java.util.concurrent.ExecutorService.

一个是使用默认的 executor，也就是netty 共享executor service。 因为handler 是一个双向链表，所以hadler的调用不是通过 woker 的来处理的，这样就可以将context 传给 子线程，由子线程控制下一步的调用，这样就从架构上解决的异步的问题。

所以即使只使用一个线程作为select，理论上也能达到比较高的效率，因为可以让handler放在另一个线程池中执行。


### SelectionKey 

SelectionKey在NIO架构中是一个核心概念，如果理解不正确会造成各种问题。我的简单理解如下是

要点一：不推荐直接写channel，而是通过缓存和attachment传入要写的数据，改变interestOps()来写数据；

要点二：每个channel只对应一个SelectionKey，所以，只能改变interestOps()，不能register()和cancel()。

```
client.register(selector, SelectionKey.OP_WRITE);
//ir
sk.interestOps(SelectionKey.OP_WRITE)
```

执行了这以上任一代码都会无限触发写事件，跟读事件不同，一定注意


nio的select()的时候，只要数据通道允许写，每次select()返回的OP_WRITE都是true。所以在nio的写数据里面，我们在每次需要写数据之前把数据放到缓冲区，并且注册OP_WRITE，对selector进行wakeup()，这样这一轮select()发现有OP_WRITE之后，将缓冲区数据写入channel，清空缓冲区，并且反注册OP_WRITE，写数据完成。

这里面需要注意的是，每个SocketChannel只对应一个SelectionKey，也就是说，在上述的注册和反注册OP_WRITE的时候，不是通过channel.register()和key.cancel()做到的，而是通过key.interestOps()做到的。代码如下：


```
  public void write(MessageSession session, ByteBuffer buffer) throws ClosedChannelException {
    SelectionKey key = session.key();
    if ((key.interestOps() & SelectionKey.OP_WRITE) == 0) {
      key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);
    }
    try {
      writebuf.put(buffer);
    } catch (Exception e) {
      System.out.println("want put:" + buffer.remaining() + ", left:" + writebuf.remaining());
      e.printStackTrace();
    }
    selector.wakeup();
  }
```


### Kafka中的实现

/**
 * An NIO socket server. The thread model is
 *   1 Acceptor thread that handles new connections
 *   N Processor threads that each have their own selectors and handle all requests from their connections synchronously
 *   M Handler threads that handle requests and produce responses back to the processor threads for writing.
 */

一个accpetor 来handle new connection， 多个processor thread， 其实也是一个 主从 selector 的结构。 M个 handler thread 来handle block 的操作，并且将response 返回给 processor 来写。

```scala
  /**
   * 每一个端口都会创建相应的 Acceptor 以及 processor （worker）
   */
  config.listeners.foreach { endpoint =>
    val listenerName = endpoint.listenerName
    val securityProtocol = endpoint.securityProtocol
    val processorEndIndex = processorBeginIndex + numProcessorThreads

    for (i <- processorBeginIndex until processorEndIndex)
      processors(i) = newProcessor(i, connectionQuotas, listenerName, securityProtocol)

    /**
     * 看上去只有一个 acceptor/selector的线程
     */
    val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId,
      processors.slice(processorBeginIndex, processorEndIndex), connectionQuotas)


    acceptors.put(endpoint, acceptor)
    Utils.newThread(s"kafka-socket-acceptor-$listenerName-$securityProtocol-${endpoint.port}", acceptor, false).start()
    acceptor.awaitStartup()

    processorBeginIndex = processorEndIndex
  }
}

```


```
 def accept(key: SelectionKey, processor: Processor) {
    val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
    val socketChannel = serverSocketChannel.accept()
    try {
      /**
       * 创建socket Channel
       */
      connectionQuotas.inc(socketChannel.socket().getInetAddress)
      socketChannel.configureBlocking(false)
      socketChannel.socket().setTcpNoDelay(true)
      socketChannel.socket().setKeepAlive(true)
      if (sendBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socketChannel.socket().setSendBufferSize(sendBufferSize)

      /**
       * 将 scokect channel 交给 processor
       */
      debug("Accepted connection from %s on %s and assigned it to processor %d, sendBufferSize [actual|requested]: [%d|%d] recvBufferSize [actual|requested]: [%d|%d]"
        .format(socketChannel.socket.getRemoteSocketAddress, socketChannel.socket.getLocalSocketAddress, processor.id,
          socketChannel.socket.getSendBufferSize, sendBufferSize,
          socketChannel.socket.getReceiveBufferSize, recvBufferSize))

      processor.accept(socketChannel)
    } catch {
      case e: TooManyConnectionsException =>
        info("Rejected connection from %s, address already has the configured maximum of %d connections.".format(e.ip, e.count))
        close(socketChannel)
    }
  }
  
  
   /**
   * Queue up a new connection for reading
   * connection 不是马上被注册到 processor 的selector 里面
   * 而是放在 newConnection 的queue 里面
   * 这样 process 在进入下一次循环的时候就才会处理这些newconnection
   */
  def accept(socketChannel: SocketChannel) {
    newConnections.add(socketChannel)
    wakeup()
  }
  
    
  /**
   * Register any new connections that have been queued up
   * 将new connection 注册到 processor 的selector 里面， 到这里为止， connection 就已经从 acceptor 线程转交给了 processor 线程
   */
  private def configureNewConnections() {
    while (!newConnections.isEmpty) {
      val channel = newConnections.poll()
      try {
        debug(s"Processor $id listening to new connection from ${channel.socket.getRemoteSocketAddress}")
        val localHost = channel.socket().getLocalAddress.getHostAddress
        val localPort = channel.socket().getLocalPort
        val remoteHost = channel.socket().getInetAddress.getHostAddress
        val remotePort = channel.socket().getPort
        val connectionId = ConnectionId(localHost, localPort, remoteHost, remotePort).toString
        selector.register(connectionId, channel)
      } catch {
        // We explicitly catch all non fatal exceptions and close the socket to avoid a socket leak. The other
        // throwables will be caught in processor and logged as uncaught exceptions.
        case NonFatal(e) =>
          val remoteAddress = channel.getRemoteAddress
          // need to close the channel here to avoid a socket leak.
          close(channel)
          error(s"Processor $id closed connection from $remoteAddress", e)
      }
    }
  }
```

### Processor 线程的逻辑 

这里非常清晰， processor 就是一个selector 的过程，处理读写关闭连接等

```
override def run() {
    startupComplete()
    while (isRunning) {
      try {
        /**
         * 将 read，write 分开处理的，使用了一个线程
         */
        // setup any new connections that have been queued up
        configureNewConnections()
        // register any new responses for writing
        processNewResponses()
        poll()

        /**
         * 处理客户端发过来的 request
         */
        processCompletedReceives()

        /**
         * 处理发送response 以后的清理工作，好像主要是 正在发送对方还没收到的消息
         * 此外，这里会记录数据的perforance 信息
         */
        processCompletedSends()

        /**
         * 顾名思义，处理客户端的断开请求
         */
        processDisconnected()
      } catch {
        // We catch all the throwables here to prevent the processor thread from exiting. We do this because
        // letting a processor exit might cause a bigger impact on the broker. Usually the exceptions thrown would
        // be either associated with a specific socket channel or a bad request. We just ignore the bad socket channel
        // or request. This behavior might need to be reviewed if we see an exception that need the entire broker to stop.
        case e: ControlThrowable => throw e
        case e: Throwable =>
          error("Processor got uncaught exception.", e)
      }
    }

    debug("Closing selector - processor " + id)
    swallowError(closeAll())
    shutdownComplete()
  }
```


### Request 的接收于发送 

在processor 的线程里面，会将进来的数据转成一个 Request 的case class放到生产者消费者的QUEUE里面，这样可以保证socket server 肯定是一个 non-block 的服务。 Queue 会在某种程度上限制发送的速度，这部分应该可以再kafka中进行配置 

```
private def processCompletedReceives() {
    selector.completedReceives.asScala.foreach { receive =>
      try {
        val openChannel = selector.channel(receive.source)
        // Only methods that are safe to call on a disconnected channel should be invoked on 'openOrClosingChannel'.
        val openOrClosingChannel = if (openChannel != null) openChannel else selector.closingChannel(receive.source)

        /**
         * 拿到session 数据
         */
        val session = RequestChannel.Session(new KafkaPrincipal(KafkaPrincipal.USER_TYPE, openOrClosingChannel.principal.getName), openOrClosingChannel.socketAddress)

        /**
         * 核心部分，将数据转换成request 并最终发送出去。
         */
        val req = RequestChannel.Request(processor = id, connectionId = receive.source, session = session,
          buffer = receive.payload, startTimeNanos = time.nanoseconds,
          listenerName = listenerName, securityProtocol = securityProtocol)

        /**
         * 这里是一个生产者消费者模式，数据只是放到 queue 里面，由handler线程读取数据进行处理
         * 我理解由于kafka 不需要像 netty 那样提供一个框架给具体实现使用， 所以不需要用handler接口来抽象业务逻辑
         * handler线程会不断轮训queue里面的数据。 好处是相对来decouple 的会更加好一些
         */
        requestChannel.sendRequest(req)
        selector.mute(receive.source)
      } catch {
        case e@(_: InvalidRequestException | _: SchemaException) =>
          // note that even though we got an exception, we can assume that receive.source is valid. Issues with constructing a valid receive object were handled earlier
          error(s"Closing socket for ${receive.source} because of error", e)
          close(selector, receive.source)
      }
    }
  }
```

Response 的看上去不是用 生产者消费者模式，估计是handler 直接放到bufer里面，然后置标志位。  

```
 /* `protected` for test usage */
  protected[network] def sendResponse(response: RequestChannel.Response, responseSend: Send) {
    val connectionId = response.request.connectionId
    trace(s"Socket server received response to send to $connectionId, registering for write and sending data: $response")
    val channel = selector.channel(connectionId)
    // `channel` can be null if the selector closed the connection because it was idle for too long
    if (channel == null) {
      warn(s"Attempting to send response via channel for which there is no open connection, connection id $connectionId")
      response.request.updateRequestMetrics(0L)
    }
    else {
      selector.send(responseSend)
      inflightResponses += (connectionId -> response)
    }
  }
```

### buffer 处理

TODO

### 总结
估计现在基于 NIO 的架构的 socket 基本都是这样子线程模型了。Netty 号称是 零拷贝， 也就是说从发送进来以后，到最终交给handler，中间不会对于buffer 做任何的处理，这样可以大大提高效率。理论上 Kafka应该也会有类似的东西。 
Selector这种线程模型或者思路其实可以用在很多环境下， 比如传统的生产者消费者模型，如果想要做到大规模扩展，应该也可以借鉴这种方法。 


### Reference

1. [Netty-Mina深入学习与对比]( http://ifeve.com/netty-mina-in-depth-1/)
1. [Understanding Reactor Pattern: Thread-Based and Event-Driven](https://dzone.com/articles/understanding-reactor-pattern-thread-based-and-eve)
1. [netty源码分析](http://wangtianzhi.cn/2016/02/04/netty-source-analysis/)
2. [netty的高性能之道](http://www.infoq.com/cn/articles/netty-high-performance)
3. [netty中的概念]http://blog.onlycatch.com/post/Netty%E4%B8%AD%E7%9A%84%E5%87%A0%E4%B8%AA%E5%85%B3%E9%94%AE%E6%A6%82%E5%BF%B5
4. [Scalable IO in Java] http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf
5. [NIO的解释](http://adblogcat.com/asynchronous-java-nio-for-dummies/)





