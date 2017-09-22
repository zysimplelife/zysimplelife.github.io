---
layout: post
title:  "Kafka Source code: How to start a server in Java or Scala"
date: 2017-09-22 15:10:00 +0000
categories: SourceCode, Kafka
---

Nowdays more and more software follow the princpal of microservice, which means each appliation will be a very dedicate service rather than a complexity, monolithic one. Some heave framwork like spring or tomcat become less and less popular.

But, if I am going to design a application in JAVA or scala, how would design the method "main" which is the entrypoint of a process. Those days, I have read the code of Kafka which can give me some idea on it.

What is the most important for a process in main? I think it should be startup and stop. let have a look on how kafka how would a very simple process do.

```java 
package kafka.examples;


import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.Scanner;

public class ChatServer {

    public static void main(String[] args) throws IOException {

        /**
         * Start a socket and bind to a port
         */
        ServerSocket ss = null;
        try {
            ss = new ServerSocket(8000);
        } catch (IOException ioe) {
            System.out.println("Can't connect to port.");
        }

        /**
         * Accept a connection, this example only accept one connection
         */
        Socket sock = null;
        try {
            sock = ss.accept();
        } catch (IOException ioe) {
            System.out.println("Can't connect to client.");
        }
        System.out.println("Connection successful.");


        /**
         * Prepare input and output object
         */

        PrintWriter out = null;
        Scanner in = null;
        try {
            out = new PrintWriter(sock.getOutputStream(), true);
            in = new Scanner(sock.getInputStream());
        } catch (Exception e) {
            System.out.println("Error.");
        }

        String line;
        String lines = "";

        /**
         * Interact with client
         */
        while ((line = in.nextLine()) != null) {
            switch (line) {
                case "list":
                    out.println("1. bla  |  2. ble  |  3. bli");
                    break;
                case "group":
                    out.println("Here group will be chosen.");
                    break;
                case "header":
                    out.println("Returns header.");
                    break;
                case "loop":
                    FileReader fr = new FileReader("D:\\Eclipse\\TestFiles\\text2.txt");
                    BufferedReader br = new BufferedReader(fr);
                    while ((lines = br.readLine()) != null) {
                        out.println(lines);
                    }
                    break;
                default:
                    if (!(line.equals("end"))) {
                        out.println("Unknown command.Try again.");
                        break;
                    }
            }
            if (line.equals("end")) {
                out.println("Connection over.");
                break;
            }
        }

        /**
         * close all socket
         */
        try {
            in.close();
            out.close();
            sock.close();
            ss.close();
        } catch (IOException ioe) {
        }

    }
}

```

As you can see, it use a while loop to keep server runing and do some close or clean task when client close the server.  That is basicly how a server keep working in JAVA world.  But it is not good for product. The major problem is it is very hard to extend for multi thread environment which is used for most of Server side application. 

Let us see how Kafka did. In brief, we can find no any look in this method. It is very clear that this application is composed of a bunch of different service, including log, socket server, controller and so on.

```scala
 def main(args: Array[String]): Unit = {
    try {
      val serverProps = getPropsFromArgs(args)
      val kafkaServerStartable = KafkaServerStartable.fromProps(serverProps)

      // attach shutdown handler to catch control-c, this hook will be run in a new Thread
      Runtime.getRuntime().addShutdownHook(new Thread("kafka-shutdown-hook") {
        override def run(): Unit = kafkaServerStartable.shutdown()
      })

      kafkaServerStartable.startup()
      kafkaServerStartable.awaitShutdown()
    }
    catch {
      case e: Throwable =>
        fatal(e)
        Exit.exit(1)
    }
    Exit.exit(0)
  }
```

How it work without while loop or simular,  the secret is in method "awaitShutdown()".  It is a very simple method of waiting for a Latch to be zero. 

```
  /**
   * After calling shutdown(), use this API to wait until the shutdown is complete
   */
  def awaitShutdown(): Unit = shutdownLatch.await()
```

You can find the this object is actually created in "starup()" method from source code. When some one call the shudown method or send control+c to application, this Latch will be decreas to 0 so the process will exist.


```  scala
  
  /**
     *   整体的启动过程比较符合现在风格， 不同的服务封装成一个一个 service，然后一起启动。
     *   使用的时候看上去是将需要的sevice 传递进去
     */
  /**
   * Start up API for bringing up a single instance of the Kafka server.
   * Instantiates the LogManager, the SocketServer and the request handlers - KafkaRequestHandlers
   */
  def startup() {
    try {
      info("starting")

      if(isShuttingDown.get)
        throw new IllegalStateException("Kafka server is still shutting down, cannot re-start!")

      if(startupComplete.get)
        return

      val canStartup = isStartingUp.compareAndSet(false, true)
      if (canStartup) {
        brokerState.newState(Starting)
        // Create and start different service
        ...
        ...

        brokerState.newState(RunningAsBroker)

        /**
         * Create a new Latch to control process 
         */
        shutdownLatch = new CountDownLatch(1)
        startupComplete.set(true)
        isStartingUp.set(false)
        AppInfoParser.registerAppInfo(jmxPrefix, config.brokerId.toString)
        info("started")
      }
    }
    ...
  }
  
  
  /**
   * Shutdown API for shutting down a single instance of the Kafka server.
   * This method could be trigger by control + c in a new Thread
   * Shuts down the LogManager, the SocketServer and the log cleaner scheduler thread
   */
  def shutdown() {
    try {
      info("shutting down")

      if (isStartingUp.get)
        throw new IllegalStateException("Kafka server is still starting up, cannot shut down!")

      // To ensure correct behavior under concurrent calls, we need to check `shutdownLatch` first since it gets updated
      // last in the `if` block. If the order is reversed, we could shutdown twice or leave `isShuttingDown` set to
      // `true` at the end of this method.
      if (shutdownLatch.getCount > 0 && isShuttingDown.compareAndSet(false, true)) {
        CoreUtils.swallow(controlledShutdown())
        brokerState.newState(BrokerShuttingDown)
        
        // close all the services
        ...
        ...

        brokerState.newState(NotRunning)

        startupComplete.set(false)
        isShuttingDown.set(false)
        CoreUtils.swallow(AppInfoParser.unregisterAppInfo(jmxPrefix, config.brokerId.toString))
        shutdownLatch.countDown()
        info("shut down completed")
      }
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServer shutdown.", e)
        isShuttingDown.set(false)
        throw e
    }
  }

  
```


How about above method, it seem more clear and graceful by removing while loop in main code.  It is a good example to me for new applications.

Attached all the start script with comments from myself

```
try {
      info("starting")

      if(isShuttingDown.get)
        throw new IllegalStateException("Kafka server is still shutting down, cannot re-start!")

      if(startupComplete.get)
        return

      val canStartup = isStartingUp.compareAndSet(false, true)
      if (canStartup) {
        brokerState.newState(Starting)

        // ezhayog: scheduler ，封装了java的scheduler，应该主要是做一些后台工作，比如清理过期的message
        /* start scheduler */
        kafkaScheduler.startup()

        //ezhayog:  初始化zookeeper， 这部分有空可以仔细看看有什么特别的地方
        /* setup zookeeper */
        zkUtils = initZk()

        /**
         * ezhayog: cluster id and broker id . 分布式系统的特点，
         * 有机会分析一下除了用来标识和打log，这些ID 还有什么用
         */
        /* Get or create cluster_id */
        _clusterId = getOrGenerateClusterId(zkUtils)
        info(s"Cluster ID = $clusterId")

        /* generate brokerId */
        config.brokerId =  getBrokerId
        this.logIdent = "[Kafka Server " + config.brokerId + "], "


        /**
         *  Performance monitor，一个很重啊哟的系统， 但是不知道他是如何适应分布式系统。
         *  现在看来独立的线程去处理发送 performance 是一个常用的做法
         *  不知道是如何解决顺序和准确度的问题。
         *  此外这里的数据是如何发送出去，是全局性的还是部分的数据也需要有时间看一下。
          */
        /* create and configure metrics */
        val reporters = config.getConfiguredInstances(KafkaConfig.MetricReporterClassesProp, classOf[MetricsReporter],
            Map[String, AnyRef](KafkaConfig.BrokerIdProp -> (config.brokerId.toString)).asJava)
        reporters.add(new JmxReporter(jmxPrefix))
        val metricConfig = KafkaServer.metricConfig(config)
        metrics = new Metrics(metricConfig, reporters, time, true)

        /* register broker metrics */
        _brokerTopicStats = new BrokerTopicStats

        quotaManagers = QuotaFactory.instantiate(config, metrics, time)
        notifyClusterListeners(kafkaMetricsReporters ++ reporters.asScala)


        /**
         * log 也是一个独立的service， 不知道后台是什么地方
         */
        /* start log manager */
        logManager = LogManager(config, zkUtils, brokerState, kafkaScheduler, time, brokerTopicStats)
        logManager.startup()

        /**
         * 应该就是配置文件的cache，这样动态修改的时候应该是修改这个cache
         */
        metadataCache = new MetadataCache(config.brokerId)

        /**
         * ausentication 相关
         * Salted Challenge Response Authentication Mechanis
         * https://en.wikipedia.org/wiki/Salted_Challenge_Response_Authentication_Mechanism
         */
        credentialProvider = new CredentialProvider(config.saslEnabledMechanisms)

        /**
         * 对外的interface ，包括 producter and consumer 连接的端口。
         * 不知道为什么要把 metrics 放进去
         */
        socketServer = new SocketServer(config, metrics, time, credentialProvider)
        socketServer.startup()

        /**
         * replication manager 我理解应该是把一些message 做备份的，但是具体作用目前不清楚
         * kafka在0.8版本前没有提供Partition的Replication机制，一旦Broker宕机，其上的所有Partition就都无法提供服务，
         * 而Partition又没有备份数据，数据的可用性就大大降低了。所以0.8后提供了Replication机制来保证Broker的failover。
         * 由于Partition有多个副本，为了保证多个副本之间的数据同步，有了这个service
         */
        /* start replica manager */
        replicaManager = createReplicaManager(isShuttingDown)
        replicaManager.startup()

        /**
         *Replication的v3版本采用Controller实现，相比v2版本的主要不同点是：

          Leader的变化从监听器改为由Controller管理
          控制器负责检测Broker的失败，并为每个受影响的Partition选举新的Leader
          控制器会将每个Leader的变化事件发送给受影响的每个Broker
          控制器和Broker之间的通信采用直接的RPC，而不是通过ZK队列

        虽然因为引入了Controller，需要实现Controller的failover。但它的优点有：

          因为Leader管理被更加集中地管理，比较容易调试问题
          Leader变化针对ZK的读写可以批量操作，减少在failover过程中端到端的延迟
          更少的ZooKeeper监听器
          使用直接RPC协议相比队列实现的ZK，能够更加高效地在节点之间通信

        从整个集群的所有Brokers中选举出一个Controller，它主要负责：

          Partition的Leader变化事件
          新创建或删除一个topic
          重新分配Partition
          管理分区的状态机和副本的状态机

         */
        /* start kafka controller */
        kafkaController = new KafkaController(config, zkUtils, time, metrics, threadNamePrefix)
        kafkaController.startup()

        /**
         * admini manager 是用来管理topic的
         * admin APIs allow the user to create, update or delete cluster resources:
         * create topics, delete topics, alter topics, alter ACLs and alter configs.
         */
        adminManager = new AdminManager(config, metrics, metadataCache, zkUtils)

        /**
         * 看上去是控制 consumer group 的。 但是consumer group 是什么我还不确定
         *
         * http://www.cnblogs.com/huxi2b/p/6223228.html
          什么是consumer group? 一言以蔽之，consumer group是kafka提供的可扩展且具有容错性的消费者机制。既然是一个组，那么组内必然可以有多个消费者或消费者实例(consumer instance)，
          它们共享一个公共的ID，即group ID。组内的所有消费者协调在一起来消费订阅主题(subscribed topics)的所有分区(partition)。
          当然，每个分区只能由同一个消费组内的一个consumer来消费。（网上文章中说到此处各种炫目多彩的图就会紧跟着抛出来。

          consumer group下可以有一个或多个consumer instance，consumer instance可以是一个进程，也可以是一个线程
          group.id是一个字符串，唯一标识一个consumer group
          consumer group下订阅的topic下的每个分区只能分配给某个group下的一个consumer(当然该分区还可以被分配给其他group)

          以上来自网上， 个人理解就是由多个 消费者一起定来定于一个活这多个主题。 以前是一对一的关系，现在变成了一对多的关系。 这样就可以衍生出不同的用例了。
          不过如何协调就会成为一个问题

         */
        /* start group coordinator */
        // Hardcode Time.SYSTEM for now as some Streams tests fail otherwise, it would be good to fix the underlying issue
        groupCoordinator = GroupCoordinator(config, zkUtils, replicaManager, Time.SYSTEM)
        groupCoordinator.startup()


        /**
         * 不知道kafka 是如何解决transaction的， 有时间研究一下
         */
        /* start transaction coordinator, with a separate background thread scheduler for transaction expiration and log loading */
        // Hardcode Time.SYSTEM for now as some Streams tests fail otherwise, it would be good to fix the underlying issue
        transactionCoordinator = TransactionCoordinator(config, replicaManager, new KafkaScheduler(threads = 1, threadNamePrefix = "transaction-log-manager-"), zkUtils, metrics, metadataCache, Time.SYSTEM)
        transactionCoordinator.startup()

        /* Get the authorizer and initialize it if one is specified.*/
        authorizer = Option(config.authorizerClassName).filter(_.nonEmpty).map { authorizerClassName =>
          val authZ = CoreUtils.createObject[Authorizer](authorizerClassName)
          authZ.configure(config.originals())
          authZ
        }

        /* start processing requests */
        apis = new KafkaApis(socketServer.requestChannel, replicaManager, adminManager, groupCoordinator, transactionCoordinator,
          kafkaController, zkUtils, config.brokerId, config, metadataCache, metrics, authorizer, quotaManagers,
          brokerTopicStats, clusterId, time)

        /**
         * 这里应该是这正做事情的对象
         */
        requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, time,
          config.numIoThreads)

        Mx4jLoader.maybeLoad()

        /* start dynamic config manager */
        dynamicConfigHandlers = Map[String, ConfigHandler](ConfigType.Topic -> new TopicConfigHandler(logManager, config, quotaManagers),
                                                           ConfigType.Client -> new ClientIdConfigHandler(quotaManagers),
                                                           ConfigType.User -> new UserConfigHandler(quotaManagers, credentialProvider),
                                                           ConfigType.Broker -> new BrokerConfigHandler(config, quotaManagers))

        // Create the config manager. start listening to notifications
        dynamicConfigManager = new DynamicConfigManager(zkUtils, dynamicConfigHandlers)
        dynamicConfigManager.startup()

        /* tell everyone we are alive */
        val listeners = config.advertisedListeners.map { endpoint =>
          if (endpoint.port == 0)
            endpoint.copy(port = socketServer.boundPort(endpoint.listenerName))
          else
            endpoint
        }


        /**
         * 居然还有health check， 有意思
         */
        kafkaHealthcheck = new KafkaHealthcheck(config.brokerId, listeners, zkUtils, config.rack,
          config.interBrokerProtocolVersion)
        kafkaHealthcheck.startup()

        // Now that the broker id is successfully registered via KafkaHealthcheck, checkpoint it
        checkpointBrokerId(config.brokerId)

        brokerState.newState(RunningAsBroker)

        /**
         * 用的是 latch 来处理启动关闭的关系，比较形象
         */
        shutdownLatch = new CountDownLatch(1)
        startupComplete.set(true)
        isStartingUp.set(false)
        AppInfoParser.registerAppInfo(jmxPrefix, config.brokerId.toString)
        info("started")
      }
    }
    catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServer startup. Prepare to shutdown", e)
        isStartingUp.set(false)
        shutdown()
        throw e
    }
  }
```








