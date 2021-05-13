[TOC]
## 1、RocketMq架构简介

### 1.1、逻辑部署图

![](https://raw.githubusercontent.com/yewenhaocn/doc/master/rocketmq/assets/images/rocketmq%E9%80%BB%E8%BE%91%E9%83%A8%E7%BD%B2%E5%9B%BE.jpg)

### 1.2、核心组件说明

通过上图可以看到，rocketmq的核心组件主要包括4个，分别是nameserver、Broker、Producer和Consumer，下面我们先依次简单说明下这四个核心组件：

+ **nameserver**：nameserver充当路由信息的提供者。生产者或消费者能够通过nameserver查找各topic相应的Broker IP列表。多个Namesrver实例组成集群，但相互独立，没有信息交换。
+ **Broker**：消息中转角色，负责存储消息、转发消息。Broker服务器在RocketMQ系统中负责接收从生产者发送来的消息并存储、同时为消费者的拉取请求作准备。Broker服务器也存储消息相关的元数据，包括消费者组、消费进度偏移和主题和队列消息等。
+ **Producer**：负责生产消息，一般由业务系统负责生产消息。一个消息生产者会把业务应用系统里产生的消息发送到broker服务器。RocketMQ提供多种发送方式，同步发送、异步发送、顺序发送、单向发送。同步和异步方式均需要Broker返回确认信息，单向发送不需要。
+ **Consumer**：负责消费消息，一般是后台系统负责异步消费。一个消息消费者会从Broker服务器拉取消息、并将其提供给应用程序。从用户应用的角度而言提供了两种消费形式：拉取式消费、推动式消费。

除了上面说的三个核心组件外，还有Topic这个概念下面也会多次提到：

+ **Topic**：表示一类消息的集合，每个Topic包含若干条消息，每条消息只能属于一个Topic，是RocketMQ进行消息订阅的基本单位。一个Topic可以分片在多个broker集群上，每一个Topic分片包含多个queue，具体结构可以参考下图：

![](https://raw.githubusercontent.com/yewenhaocn/doc/master/rocketmq/assets/images/topic%26queue%E7%BB%93%E6%9E%84.jpg)

### 1.3、设计理念

RocketMq是基于主题的发布与订阅模式，核心功能包括消息发送、消息存储、消息消费，整体设计追求简单与性能第一，归纳来说主要是下面三种：

+ nameserver取代ZK充当注册中心，nameserver集群间互不通信，容忍路由信息在集群内分钟级不一致，更加轻量级
+ 使用内存映射机制实现高效的IO存储，达到高吞吐量
+ 容忍设计缺陷，通过ACK确保消息至少消费一次，但是如果ACK丢失，可能消息重复消费，这种情况设计上允许，交给使用者自己保证。

本文重点介绍的就是nameserver，我们下面一起来看下nameserver是如何启动以及如何进行路由管理的。

## 2、nameserver架构设计

在第一章已经简单介绍了nameserver取代zk作为一种更轻量级的注册中心充当路由信息的提供者。那么具体是如何来实现路由信息管理的呢？我们先看下图：

![](https://raw.githubusercontent.com/yewenhaocn/doc/master/rocketmq/assets/images/namerserver%E8%B7%AF%E7%94%B1%E7%AE%A1%E7%90%86%E5%8E%9F%E7%90%86%E5%9B%BE-202105101719.jpg)

上面的图描述了nameserver进行路由注册、路由剔除和路由发现的核心原理。

+ **路由注册**：Broker服务器在启动的时候会想nameserver集群中所有的nameserver发送心跳信号进行注册，并会每隔30秒向nameserver发送心跳，告诉nameserver自己活着。nameserver接收到broker发送的心跳包之后，会记录该broker信息，并保存最近一次收到心跳包的时间。
+ **路由剔除**：nameserver和每个Broker保持长连接，每隔30秒接收broker发送的心跳包，同时自身每个10秒扫描brokerLiveTable，比较上次收到心跳时间和当前时间比较是否大于120秒，如果超过，那么认为broker不可用，剔除路由表中该broker相关信息
+ **路由发现**：路由发现不是实时的，路由变化后，nameserver不主动推给客户端，等待producer定期拉取最新路由信息。这样的设计方式降低了nameserver实现的复杂性，当路由发生变化时通过在消息发送端的容错机制来保证消息发送的高可用（这块内容会在后续介绍producer消息发送时介绍，本文不展开讲解）。
+ **高可用**：nameserver通过部署多台nameserver服务器来保证自身的高可用，同时多个nameserver服务器之间不进行通信，这样路由信息发生变化时，各个nameserver服务器之间数据可能不是完全相同的，但是通过发送端的容错机制保证消息发送的高可用。这个也正是nameserver追求简单高效的目的所在。

## 3、启动流程

在整理了解了nameserver的架构设计之后，我们先来看下nameserver到底是如何启动的呢？

既然是源码解读，那么我们先来看下代码入口：`org.apache.rocketmq.namesrv.NamesrvStartup#main(String[] args)`，实际调用的是`main0()`方法，代码如下：

```java
public static NamesrvController main0(String[] args) {

    try {
        //创建namesrvController
        NamesrvController controller = createNamesrvController(args);
        //初始化并启动NamesrvController
        start(controller);
        String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

通过main方法启动nameserver，主要分为两大步，先创建namesrvController，然后再初始化并启动NamesrvController。我们分别展开来分析。

### 3.1、时序图

具体展开阅读代码之前，我们先通过一个序列图对整体流程有个了解，如下图：

![](https://raw.githubusercontent.com/yewenhaocn/doc/master/rocketmq/assets/images/Nameserver%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### 3.2、创建namesrvController

先来看核心代码，如下：

```java
public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
    // 设置版本号为当前版本号
    System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
    //PackageConflictDetect.detectFastjson();
	//构造org.apache.commons.cli.Options,并添加-h -n参数，-h参数是打印帮助信息，-n参数是指定namesrvAddr
    Options options = ServerUtil.buildCommandlineOptions(new Options());
    //初始化commandLine，并在options中添加-c -p参数，-c指定nameserver的配置文件路径，-p标识打印配置信息
    commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
    if (null == commandLine) {
        System.exit(-1);
        return null;
    }
	//nameserver配置类，业务参数
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    //netty服务器配置类，网络参数
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    //设置nameserver的端口号
    nettyServerConfig.setListenPort(9876);
    //命令带有-c参数，说明指定配置文件，需要根据配置文件路径读取配置文件内容，并将文件中配置信息赋值给NamesrvConfig和NettyServerConfig
    if (commandLine.hasOption('c')) {
        String file = commandLine.getOptionValue('c');
        if (file != null) {
            InputStream in = new BufferedInputStream(new FileInputStream(file));
            properties = new Properties();
            properties.load(in);
            //反射的方式
            MixAll.properties2Object(properties, namesrvConfig);
            MixAll.properties2Object(properties, nettyServerConfig);
			//设置配置文件路径
            namesrvConfig.setConfigStorePath(file);

            System.out.printf("load config properties file OK, %s%n", file);
            in.close();
        }
    }
	//命令行带有-p，说明是打印参数的命令，那么就打印出NamesrvConfig和NettyServerConfig的属性。在启动NameServer时可以先使用./mqnameserver -c configFile -p打印当前加载的配置属性 
    if (commandLine.hasOption('p')) {
        InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
        MixAll.printObjectProperties(console, namesrvConfig);
        MixAll.printObjectProperties(console, nettyServerConfig);
        //打印参数命令不需要启动nameserver服务，只需要打印参数即可
        System.exit(0);
    }
	//解析命令行参数，并加载到namesrvConfig中
    MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);
	//检查ROCKETMQ_HOME，不能为空
    if (null == namesrvConfig.getRocketmqHome()) {
        System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
        System.exit(-2);
    }
	//初始化logback日志工厂，rocketmq默认使用logback作为日志输出
    LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
    JoranConfigurator configurator = new JoranConfigurator();
    configurator.setContext(lc);
    lc.reset();
    configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

    log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    MixAll.printObjectProperties(log, namesrvConfig);
    MixAll.printObjectProperties(log, nettyServerConfig);
	//创建NamesrvController
    final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

    //将全局Properties的内容复制到NamesrvController.Configuration.allConfigs中
    // remember all configs to prevent discard
    controller.getConfiguration().registerConfig(properties);

    return controller;
}
```

通过上面对每一行代码的注释，可以看出来，创建`NamesrvController`的过程主要分为两步：

+ **Step1**：通过命令行中获取配置。赋值给`NamesrvConfig`和`NettyServerConfig`类
+ **Step2**：根据配置类`NamesrvConfig`和`NettyServerConfig`构造一个`NamesrvController`实例

可见`NamesrvConfig`和`NettyServerConfig`是想当重要的，这两个类分别是nameserver的业务参数和网络参数，我们分别看下这两个类里面有哪些属性：

+ **NamesrvConfig**

| 属性名             | 含义                                                         |
| ------------------ | ------------------------------------------------------------ |
| rocketmqHome       | RocketMQ 主目录，通过 Drocketmq. home.dir=path 或通过设置环境变量 ROCKETMQ_HOME 来配置 RocketMQ 的主目录 |
| kvConfigPath       | kv 配置文件路径，包含顺序消息主题的配置信息，默认值：$user.home/namesrv/kvConfig.json |
| configStorePath    | NameServer 配置文件路径，默认值：$user.home/namesrv/namesrv.properties |
| clusterTest        | 是否开启集群测试，默认值：false                              |
| orderMessageEnable | 是否支持顺序消息，默认值：false                              |

+ **NettyServerConfig**

| 属性名                             | 含义                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| listenPort                         | NameServer监听端口，该值默认会被初始化为 9876                |
| serverWorkerThreads                | netty业务线程池线程个数                                      |
| serverCallbackExecutorThreads      | Netty public任务线程池线程个数，Netty网络设计，根据业务类型会创建不同的线程池，比如处理消息发送、消息消费、心跳检测等，如果该业务类型（ RequestCode ）未注册线程池，则由public线程池执行 |
| serverSelectorThreads              | IO线程池线程个数，主要是NameServer、Brok端解析请求、返回相应的线程个数，这类线程主要是处理网络请求的，解析请求包，然后转发到各个业务线程池完成具体的业务操作，然后将结果再返回调用方 |
| serverOnewaySemaphoreValue         | send oneway 消息请求井发度（ Broker 端参数）                 |
| serverAsyncSemaphoreValue          | 异步消息发送最大并发度（ Broker 端参数）                     |
| serverChannelMaxIdleTimeSeconds    | 网络连接最大空闲时间，默认120s，如果连接空闲时间超过该参数设置的值，连接将被关闭 |
| serverSocketSndBufSize             | 网络socket发送缓存区大小，默认64k                            |
| serverSocketRcvBufSize             | 网络socket接收缓存区大小，默认64k                            |
| serverPooledByteBufAllocatorEnable | ByteBuffer是否开启缓存，建议开启                             |
| useEpollNativeSelector             | 是否弃用Epoll IO模型，Linux环境建议开启                      |

> ***注：Apache Commons CLI是开源的命令行解析工具，它可以帮助开发者快速构建启动命令，并且帮助你组织命令的参数、以及输出列表等***

### 3.3、初始化并启动

创建了`NamesrvController`实例之后，开始初始化并启动NameServer。

1. 首先进行初始化，代码入口是`NamesrvController#initialize`。

```java
public boolean initialize() {
	//加载kvConfigPath下kvConfig.json配置文件里的KV配置，然后将这些配置放到KVConfigManager#configTable属性中
    this.kvConfigManager.load();
	//根据nettyServerConfig初始化一个netty服务器。
    //brokerHousekeepingService是在NamesrvController实例化时构造函数里实例化的，该类负责Broker连接事件的处理，实现了ChannelEventListener，主要用来管理RouteInfoManager的brokerLiveTable
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
	//初始化负责处理Netty网络交互数据的线程池，默认线程数是8个
    this.remotingExecutor =
        Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
	//注册Netty服务端业务处理逻辑，如果开启了clusterTest，那么注册的请求处理类是ClusterTestRequestProcessor，否则请求处理类是DefaultRequestProcessor
    this.registerProcessor();
	//注册心跳机制线程池，延迟5秒启动，每隔10秒遍历RouteInfoManager#brokerLiveTable这个属性，用来扫描不存活的broker
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);
	//注册打印KV配置线程池，延迟1分钟启动、每10分钟打印出kvConfig配置
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);
	//rocketmq可以通过开启TLS来提高数据传输的安全性，如果开启了，那么需要注册一个监听器来重新加载SslContext
    if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
        // Register a listener to reload SslContext
        try {
            fileWatchService = new FileWatchService(
                new String[] {
                    TlsSystemConfig.tlsServerCertPath,
                    TlsSystemConfig.tlsServerKeyPath,
                    TlsSystemConfig.tlsServerTrustCertPath
                },
                new FileWatchService.Listener() {
                    boolean certChanged, keyChanged = false;
                    @Override
                    public void onChanged(String path) {
                        if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                            log.info("The trust certificate changed, reload the ssl context");
                            reloadServerSslContext();
                        }
                        if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                            certChanged = true;
                        }
                        if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                            keyChanged = true;
                        }
                        if (certChanged && keyChanged) {
                            log.info("The certificate and private key changed, reload the ssl context");
                            certChanged = keyChanged = false;
                            reloadServerSslContext();
                        }
                    }
                    private void reloadServerSslContext() {
                        ((NettyRemotingServer) remotingServer).loadSslContext();
                    }
                });
        } catch (Exception e) {
            log.warn("FileWatchService created error, can't load the certificate dynamically");
        }
    }

    return true;
}
```

上面的代码是NameServer初始化流程，通过每行代码的注释，我们可以看出来，主要有5步骤操作：

+ **Step1**：加载KV配置，并写入到`KVConfigManager`的`configTable`属性中
+ **Step2**：初始化netty服务器
+ **Step3**：初始化处理netty网络交互数据的线程池
+ **Step4**：注册心跳机制线程池，启动5秒后每隔10秒检测一次broker的存活情况
+ **Step5**：注册打印KV配置的线程池，启动1分钟后，每隔10分钟打印一次KV配置

2. rocketmq的开发团队还使用了一个常用的编程技巧，就是使用JVM钩子函数对nameserver进行优雅停机。这样在JVM进程关闭前，会先执行shutdown操作。

```java
Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
    @Override
    public Void call() throws Exception {
        controller.shutdown();
        return null;
    }
}));
```

3. 执行start函数，启动nameserver。代码比较简单，就是将第一步中创建的netty server进行启动。其中`remotingServer.start()`方法不展开详细说明了，需要对netty比较熟悉，不是本篇文章重点，有兴趣的同学可以自行下载源码阅读。

```java
public void start() throws Exception {
    //启动netty服务
    this.remotingServer.start();
	//如果开启了TLS
    if (this.fileWatchService != null) {
        this.fileWatchService.start();
    }
}
```

## 4、路由管理

我们在第二章开篇有了解到nameserver作为一个轻量级的注册中心，主要是为消息生产者和消费者提供Topic的路由信息，并对这些路由信息和broker节点进行管理，主要包括路由注册、路由剔除和路由发现。本章将会通过源码的角度来具体分析下nameserver是如果进行路由信息管理的。核心代码主要都在`org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager`中实现。

### 4.1、路由元信息

在了解路由信息管理之前，我们首先需要了解NameServer到底存储了哪些路由元信息，数据结构分别是什么样的。

查看代码我们可以看到主要通过5个属性来维护路由元信息，如下：

```java
private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

我们依次对这5个属性进行展开说明。

#### 4.1.1、topicQueueTable

+ **说明**：Topic消息队列路由信息，消息发送时根据路由表进行负载均衡

+ **数据结构**：`HashMap`结构，key是topic名字，value是一个类型是<span id="1">`QueueData`</span>的队列集合。在第一章就讲过，一个topic中有多个队列。`QueueData`的数据结构如下：

  | 字段名         | 类型   | 说明                              |
    | -------------- | ------ | --------------------------------- |
  | brokerName     | String | broker名字                        |
  | readQueueNums  | int    | 读队列数目                        |
  | writeQueueNums | int    | 写队列数目                        |
  | perm           | int    | 读写权限，参考`PermName`          |
  | topicSynFlag   | int    | topic同步标记，参考`TopicSysFlag` |

+ **数据结构示例**：

```json
topicQueueTable:{
    "topic1": [
        {
            "brokerName": "broker-a",
            "readQueueNums":4,
            "writeQueueNums":4,
            "perm":6,
            "topicSynFlag":0,
        },
        {
            "brokerName": "broker-b",
            "readQueueNums":4,
            "writeQueueNums":4,
            "perm":6,
            "topicSynFlag":0,
        }
    ]
}
```

#### 4.1.2、brokerAddrTable

+ **说明**：Broker基础信息，包含brokerName、所属集群名称、主备Broker地址

+ **数据结构**：`HashMap`结构，key是brokerName，value是一个类型是<span id="2">`BrokerData`</span>的对象。`BrokerData`的数据结构如下(可以结合下面broker主从结构逻辑图来理解)：

  | 字段名      | 类型                                                    | 说明                                                         |
    | ----------- | ------------------------------------------------------- | ------------------------------------------------------------ |
  | cluster     | String                                                  | broker集群名                                                 |
  | brokerName  | String                                                  | broker名                                                     |
  | brokerAddrs | HashMap<Long/* brokerId */, String/* broker address */> | 该broker的ip集合（每个broker主从结构），key是brokerId，value是ip地址 |

+  **broker主从结构逻辑图**：

![](https://raw.githubusercontent.com/yewenhaocn/doc/master/rocketmq/assets/images/broker%E4%B8%BB%E4%BB%8E%E7%BB%93%E6%9E%84.jpg)

+ **数据结构示例**：

```json
brokerAddrTable:{
    "broker-a": {
        "cluster": "c1",
        "brokerName": "broker-a",
        "brokerAddrs": {
            0: "192.168.1.1:10000",
            1: "192.168.1.2:10000"
        }
    },
    "broker-b": {
        "cluster": "c1",
        "brokerName": "broker-b",
        "brokerAddrs": {
            0: "192.168.1.3:10000",
            1: "192.168.1.4:10000"
        }
    }
}
```

#### 4.1.3、clusterAddrTable

+ **说明**：Broker集群信息，存储集群中所有Broker名称
+ **数据结构**：`HashMap`结构，key是clusterName，value是存储brokerName的`Set`结构。
+ **数据结构示例**：

```json
clusterAddrTable:{
    "c1": ["broker-a","broker-b"]
}
```

#### 4.1.4、brokerLiveTable

+ **说明**：Broker状态信息。NameServer每次收到心跳包时会替换该信息

+ **数据结构**：`HashMap`结构，key是broker的地址，value是`BrokerLiveInfo`结构的该broker信息对象。`BrokerLiveInfo`的数据结构如下：

  | 字段名              | 类型        | 说明                                                         |
    | ------------------- | ----------- | ------------------------------------------------------------ |
  | lastUpdateTimestamp | long        | 最近一次收到心跳包的时间戳                                   |
  | dataVersion         | DataVersion | 数据版本号对象                                               |
  | channel             | Channel     | netty的Channel，IO数据交互的媒介                             |
  | haServerAddr        | String      | master地址，初次请求的时候该值为空，slave向Nameserver注册后返回 |

+ **数据结构示例**：

```json
brokerLiveTable:{
    "192.168.1.1:10000": {
            "lastUpdateTimestamp": 1518270318980,
            "dataVersion":versionObj1,
            "channel":channelObj,
            "haServerAddr":""
    },
    "192.168.1.2:10000": {
            "lastUpdateTimestamp": 1518270318980,
            "dataVersion":versionObj1,
            "channel":channelObj,
            "haServerAddr":"192.168.1.1:10000"
     },
    "192.168.1.3:10000": {
            "lastUpdateTimestamp": 1518270318980,
            "dataVersion":versionObj1,
            "channel":channelObj,
            "haServerAddr":""
     },
    "192.168.1.4:10000": {
            "lastUpdateTimestamp": 1518270318980,
            "dataVersion":versionObj1,
            "channel":channelObj,
            "haServerAddr":"192.168.1.3:10000"
     }
}
```

#### 4.1.5、<span id="3">filterServerTable</span>

+ **说明**：Broker上的FilterServer列表，消息过滤服务器列表，后续介绍Consumer时会介绍，consumer拉取数据是通过filterServer拉取，consumer向broker注册
+ **数据结构**：`HashMap`结构，key是broker地址，value是记录了filterServer地址的`List`集合

### 4.2、路由注册

路由注册是通过Broker和NameServer之间的心跳功能来实现的。主要分为两步：

+ **Step1**：Broker启动时向集群中所有NameServer发送心跳语句，每隔30秒（默认30s，时间间隔在10秒到60秒之间）再发一次。
+ **Step2**：NameServer收到心跳包更新`topicQueueTable`,`brokerAddrTable`,`brokerLiveTable`，`clusterAddrTable`，`filterServerTable`。

我们分别展开分析这两步。

#### 4.2.1、Broker发送心跳包

发送心跳包的核心逻辑是在Broker启动逻辑里,代码入口是`org.apache.rocketmq.broker.BrokerController#start`，本篇文章重点关注的是发送心跳包的逻辑实现，只列出发送心跳包的核心代码，如下：

1. 创建了一个线程池注册Broker，程序启动10秒后执行，每隔30秒（默认30s，时间间隔在10秒到60秒之间，`brokerConfig.getRegisterNameServerPeriod()`的默认值是30秒）执行一次。

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        try {
            BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
        } catch (Throwable e) {
            log.error("registerBrokerAll Exception", e);
        }
    }
}, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

2. 封装topic配置和版本号之后，进行实际的路由注册（*注：封装topic配置不是本篇重点，会在介绍broker源码时重点讲解*）。实际路由注册是在`org.apache.rocketmq.broker.out.BrokerOuterAPI#registerBrokerAll`中实现，核心代码如下：

```java
public List<RegisterBrokerResult> registerBrokerAll(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final boolean oneway,
    final int timeoutMills,
    final boolean compressed) {

    final List<RegisterBrokerResult> registerBrokerResultList = new CopyOnWriteArrayList<>();
    //获取nameserver地址列表
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {
		/**
		  *封装请求包头start
		  *封装请求包头，主要封装broker相关信息
		**/
        final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
        requestHeader.setBrokerAddr(brokerAddr);
        requestHeader.setBrokerId(brokerId);
        requestHeader.setBrokerName(brokerName);
        requestHeader.setClusterName(clusterName);
        requestHeader.setHaServerAddr(haServerAddr);
        requestHeader.setCompressed(compressed);
		//封装requestBody，包括topic和filterServerList相关信息
        RegisterBrokerBody requestBody = new RegisterBrokerBody();
        requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
        requestBody.setFilterServerList(filterServerList);
        final byte[] body = requestBody.encode(compressed);
        final int bodyCrc32 = UtilAll.crc32(body);
        requestHeader.setBodyCrc32(bodyCrc32);
        /**
		  *封装请求包头end
		**/
        //开启多线程到每个nameserver进行注册
        final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
        for (final String namesrvAddr : nameServerAddressList) {
            brokerOuterExecutor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        //实际进行注册方法
                        RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                        if (result != null) {
                            //封装nameserver返回的信息
                            registerBrokerResultList.add(result);
                        }

                        log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                    } catch (Exception e) {
                        log.warn("registerBroker Exception, {}", namesrvAddr, e);
                    } finally {
                        countDownLatch.countDown();
                    }
                }
            });
        }

        try {
            countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
        }
    }

    return registerBrokerResultList;
}
```

从上面代码来看，也比较简单，首先需要封装请求包头和requestBody，然后开启多线程到每个nameserver服务器去注册。

+ 请求包头类型为`RegisterBrokerRequestHeader`，主要包括如下字段：

| 字段名       | 类型    | 说明                                                         |
| ------------ | ------- | ------------------------------------------------------------ |
| brokerAddr   | String  | broker地址                                                   |
| brokerId     | Long    | broker的id                                                   |
| brokerName   | String  | broker名字                                                   |
| clusterName  | String  | 集群名                                                       |
| haServerAddr | String  | master地址，主从架构，实现高可用需要                         |
| compressed   | boolean | 是否需要压缩，RegisterBrokerBody是否需要压缩后序列化         |
| bodyCrc32    | int     | 将requestBody通过CRC32算法计算出CRC值，用于在nameserver端校验数据的正确性 |

+ requestBody类型是`RegisterBrokerBody`，主要包括如下字段：

| 字段名                      | 类型                        | 说明                                                         |
| --------------------------- | --------------------------- | ------------------------------------------------------------ |
| topicConfigSerializeWrapper | TopicConfigSerializeWrapper | topic配置相关信息                                            |
| filterServerList            | List<String>                | 消息过滤服务器列表，后续介绍Consumer时会介绍，consumer拉取数据是通过filterServer拉取，consumer向broker注册 |

2. 实际的路由注册是通过`registerBroker`方法实现，核心代码如下：

```java
private RegisterBrokerResult registerBroker(
    final String namesrvAddr,
    final boolean oneway,
    final int timeoutMills,
    final RegisterBrokerRequestHeader requestHeader,
    final byte[] body
) throws RemotingCommandException, MQBrokerException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException,
InterruptedException {
    //创建请求指令，需要注意RequestCode.REGISTER_BROKER，nameserver端的网络处理器会根据requestCode进行相应的业务处理
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.REGISTER_BROKER, requestHeader);
    request.setBody(body);
	//基于netty进行网络传输
    if (oneway) {
        //如果是单向调用，没有返回值，不返回nameserver返回结果
        try {
            this.remotingClient.invokeOneway(namesrvAddr, request, timeoutMills);
        } catch (RemotingTooMuchRequestException e) {
            // Ignore
        }
        return null;
    }
	//异步调用向nameserver发起注册，获取nameserver的返回信息
    RemotingCommand response = this.remotingClient.invokeSync(namesrvAddr, request, timeoutMills);
    assert response != null;
    switch (response.getCode()) {
        case ResponseCode.SUCCESS: {
            //获取返回的reponseHeader
            RegisterBrokerResponseHeader responseHeader =
                (RegisterBrokerResponseHeader) response.decodeCommandCustomHeader(RegisterBrokerResponseHeader.class);
            //重新封装返回结果，更新masterAddr和haServerAddr
            RegisterBrokerResult result = new RegisterBrokerResult();
            result.setMasterAddr(responseHeader.getMasterAddr());
            result.setHaServerAddr(responseHeader.getHaServerAddr());
            if (response.getBody() != null) {
                result.setKvTable(KVTable.decode(response.getBody(), KVTable.class));
            }
            return result;
        }
        default:
            break;
    }

    throw new MQBrokerException(response.getCode(), response.getRemark(), requestHeader == null ? null : requestHeader.getBrokerAddr());
}
```

borker和nameserver之间通过netty进行网络传输，broker向nameserver发起注册时会在请求中添加注册码`RequestCode.REGISTER_BROKER`。这是一种网络跟踪方法，rocketmq的每个请求都会定义一个requestCode，服务端的网络处理器会根据不同的requestCode进行影响的业务处理。

#### 4.2.2、nameserver处理心跳包

broker发出路由注册的心跳包之后，nameserver会根据心跳包中的requestCode进行处理。nameserver的默认网络处理器是`DefaultRequestProcessor`,具体代码如下：

```java
public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
    if (ctx != null) {
        log.debug("receive request, {} {} {}",
                  request.getCode(),
                  RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                  request);
    }
    switch (request.getCode()) {
        ......
        //，如果是RequestCode.REGISTER_BROKER，进行broker注册
        case RequestCode.REGISTER_BROKER:
            Version brokerVersion = MQVersion.value2Version(request.getVersion());
            if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                return this.registerBrokerWithFilterServer(ctx, request);
            } else {
                return this.registerBroker(ctx, request);
            }
        ......
        default:
            break;
    }
    return null;
}
```

判断requestCode，如果是`RequestCode.REGISTER_BROKER`，那么确定业务处理逻辑是注册broker。根据broker版本号选择不同的方法，我们已V3_0_11以上为例，调用`registerBrokerWithFilterServer`方法进行注册主要步骤分为三步：

+ **Step1**：解析requestHeader并验签（基于crc32），判断数据是否正确；
+ **Step2**：解析Topic信息；
+ **Step3**：调用`RouteInfoManager#registerBroker`来进行broker注册；

核心注册逻辑是由`RouteInfoManager#registerBroker`来实现，核心代码如下：

```java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            //加写锁，防止并发写RoutInfoManager中的路由表信息。
            this.lock.writeLock().lockInterruptibly();
			//根据clusterName从clusterAddrTable中获取所有broker名字集合
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            //如果没有获取到，说明broker所属集群还没记录，那么需要创建，并将brokerName加入到集群的broker集合中
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);
			
            boolean registerFirst = false;
			//根据brokerName尝试从brokerAddrTable中获取brokerData
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                //如果没获取到brokerData，新建BrokerData并放入brokerAddrTable，registerFirst设为true；
                registerFirst = true;
                brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            //更新brokerData中的brokerAddrs
            Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
            //考虑到可能出现master挂了，slave变成master的情况，这时候brokerId会变成0，这时候需要把老的brokerAddr给删除
            //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
            //The same IP:PORT must only have one record in brokerAddrTable
            Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
            while (it.hasNext()) {
                Entry<Long, String> item = it.next();
                if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                    it.remove();
                }
            }
			//更新brokerAddrs，根据返回的oldAddr判断是否是第一次注册的broker
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);

            //如过Broker是Master，并且Broker的Topic配置信息发生变化或者是首次注册，需要创建或更新Topic路由元数据，填充topicQueueTable
            if (null != topicConfigWrapper
                && MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            //创建或更新Topic路由元数据
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }
			//更新BrokerLivelnfo,BrokeLivelnfo是执行路由删除的重要依据
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                                                                         new BrokerLiveInfo(
                                                                             System.currentTimeMillis(),
                                                                             topicConfigWrapper.getDataVersion(),
                                                                             channel,
                                                                             haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
            }
			//注册Broker的filterServer地址列表
            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }
			//如果此Broker为从节点，则需要查找Broker Master的节点信息，并更新对应masterAddr属性
            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}
```

通过上面的源码分析，可以分解出一个broker的注册主要分7步：

+ **Step1**：加写锁，防止并发写RoutInfoManager中的路由表信息
+ **Step2**：判断broker所属集群是否存在，不存在需要创建，并将broker名加入到集群broker集合中
+ **Step3**：维护BrokerData
+ **Step4**：如过Broker是Master，并且Broker的Topic配置信息发生变化或者是首次注册，需要创建或更新Topic路由元数据，填充topicQueueTable
+ **Step5**：更新BrokerLivelnfo
+ **Step6**：注册Broker的filterServer地址列表
+ **Step7**：如果此Broker为从节点，则需要查找Broker Master的节点信息，并更新对应masterAddr属性，并返回给broker端

### 4.3、路由剔除

#### 4.3.1、触发条件

路由剔除的触发条件主要有两个：

+ nameserver每隔10s扫描brokerLiveTable，连续120s没收到心跳包，则移除该broker并关闭socket连接
+ broker正常关闭时触发路由删除

#### 4.3.2、源码解析

上面描述的两个触发点最终删除路由的逻辑是一样的，统一在`RouteInfoManager#onChannelDestroy`中实现，核心代码如下：

```java
public void onChannelDestroy(String remoteAddr, Channel channel) {
    String brokerAddrFound = null;
    if (channel != null) {
        try {
            try {
                //加读锁
                this.lock.readLock().lockInterruptibly();
                //通过channel从brokerLiveTable中找出对应的Broker地址
                Iterator<Entry<String, BrokerLiveInfo>> itBrokerLiveTable =
                    this.brokerLiveTable.entrySet().iterator();
                while (itBrokerLiveTable.hasNext()) {
                    Entry<String, BrokerLiveInfo> entry = itBrokerLiveTable.next();
                    if (entry.getValue().getChannel() == channel) {
                        brokerAddrFound = entry.getKey();
                        break;
                    }
                }
            } finally {
                //释放读锁
                this.lock.readLock().unlock();
            }
        } catch (Exception e) {
            log.error("onChannelDestroy Exception", e);
        }
    }
	//若该Broker已经从存活的Broker地址列表中被清除，则直接使用remoteAddr
    if (null == brokerAddrFound) {
        brokerAddrFound = remoteAddr;
    } else {
        log.info("the broker's channel destroyed, {}, clean it's data structure at once", brokerAddrFound);
    }

    if (brokerAddrFound != null && brokerAddrFound.length() > 0) {

        try {
            try {
                //申请写锁
                this.lock.writeLock().lockInterruptibly();
                //根据brokerAddress，将这个brokerAddress从brokerLiveTable和filterServerTable中移除
                this.brokerLiveTable.remove(brokerAddrFound);
                this.filterServerTable.remove(brokerAddrFound);
                String brokerNameFound = null;
                boolean removeBrokerName = false;
                Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                    this.brokerAddrTable.entrySet().iterator();
                //遍历brokerAddrTable
                while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
                    BrokerData brokerData = itBrokerAddrTable.next().getValue();

                    Iterator<Entry<Long, String>> it = brokerData.getBrokerAddrs().entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<Long, String> entry = it.next();
                        Long brokerId = entry.getKey();
                        String brokerAddr = entry.getValue();
                        //根据brokerAddress找到对应的brokerData，并将brokerData中对应的brokerAddress移除
                        if (brokerAddr.equals(brokerAddrFound)) {
                            brokerNameFound = brokerData.getBrokerName();
                            it.remove();
                            log.info("remove brokerAddr[{}, {}] from brokerAddrTable, because channel destroyed",
                                     brokerId, brokerAddr);
                            break;
                        }
                    }
					//如果移除后，整个brokerData的brokerAddress空了，那么将整个brokerData移除
                    if (brokerData.getBrokerAddrs().isEmpty()) {
                        removeBrokerName = true;
                        itBrokerAddrTable.remove();
                        log.info("remove brokerName[{}] from brokerAddrTable, because channel destroyed",
                                 brokerData.getBrokerName());
                    }
                }

                if (brokerNameFound != null && removeBrokerName) {
                    //遍历clusterAddrTable
                    Iterator<Entry<String, Set<String>>> it = this.clusterAddrTable.entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<String, Set<String>> entry = it.next();
                        String clusterName = entry.getKey();
                        Set<String> brokerNames = entry.getValue();
                        //根据第三步中获取的需要移除的brokerName，将对应的brokerName移除了
                        boolean removed = brokerNames.remove(brokerNameFound);
                        if (removed) {
                            log.info("remove brokerName[{}], clusterName[{}] from clusterAddrTable, because channel destroyed",
                                     brokerNameFound, clusterName);
							//如果移除后，该集合为空，那么将整个集群从clusterAddrTable中移除
                            if (brokerNames.isEmpty()) {
                                log.info("remove the clusterName[{}] from clusterAddrTable, because channel destroyed and no broker in this cluster",
                                         clusterName);
                                it.remove();
                            }

                            break;
                        }
                    }
                }

                if (removeBrokerName) {
                    Iterator<Entry<String, List<QueueData>>> itTopicQueueTable =
                        this.topicQueueTable.entrySet().iterator();
                    //遍历topicQueueTable
                    while (itTopicQueueTable.hasNext()) {
                        Entry<String, List<QueueData>> entry = itTopicQueueTable.next();
                        String topic = entry.getKey();
                        List<QueueData> queueDataList = entry.getValue();

                        Iterator<QueueData> itQueueData = queueDataList.iterator();
                        while (itQueueData.hasNext()) {
                            QueueData queueData = itQueueData.next();
                            //根据brokerName，将topic下对应的broker移除掉
                            if (queueData.getBrokerName().equals(brokerNameFound)) {
                                itQueueData.remove();
                                log.info("remove topic[{} {}], from topicQueueTable, because channel destroyed",
                                         topic, queueData);
                            }
                        }
						//如果该topic下只有一个待移除的broker，那么该topic也从table中移除
                        if (queueDataList.isEmpty()) {
                            itTopicQueueTable.remove();
                            log.info("remove topic[{}] all queue, from topicQueueTable, because channel destroyed",
                                     topic);
                        }
                    }
                }
            } finally {
                //释放写锁
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("onChannelDestroy Exception", e);
        }
    }
}
```

路由删除整体逻辑主要分为6步：

+ **Step1**：加readlock，通过channel从brokerLiveTable中找出对应的Broker地址，释放readlock，若该Broker已经从存活的Broker地址列表中被清除，则直接使用remoteAddr
+ **Step2**：申请写锁，根据brokerAddress从brokerLiveTable、filterServerTable移除
+ **Step3**：遍历brokerAddrTable，根据brokerAddress找到对应的brokerData，并将brokerData中对应的brokerAddress移除，如果移除后，整个brokerData的brokerAddress空了，那么将整个brokerData移除
+ **Step4**：遍历clusterAddrTable，根据第三步中获取的需要移除的brokerName，将对应的brokerName移除了。如果移除后，该集合为空，那么将整个集群从clusterAddrTable中移除
+ **Step5**：遍历topicQueueTable，根据brokerName，将topic下对应的broker移除掉，如果该topic下只有一个待移除的broker，那么该topic也从table中移除
+ **Step6**：释放写锁

从上面可以看出，路由剔除的整体逻辑比较简单，就是单纯地针对路由元信息的数据结构进行操作。为了大家能够更好地理解这块代码，建议大家对照4.1中介绍的路由元信息的数据结构来进行代码走读。

### 4.4、路由发现

当路由信息发生变化之后，nameserver不会主动推送给客户端，而是等待客户端定期到nameserver主动拉取最新路由信息。这种设计方式降低了nameserver实现的复杂性。

#### 4.4.1、producer主动拉取

producer在启动后会开启一系列定时任务，其中有一个任务就是定期从nameserver获取topic路由信息。代码入口是`MQClientInstance#startScheduledTask()`核心代码如下：

```java
private void startScheduledTask() {
    ......
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

@Override
public void run() {
    try {
    //从nameserver更新最新的topic路由信息
    MQClientInstance.this.updateTopicRouteInfoFromNameServer();
    } catch (Exception e) {
    log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
    }
    }
    }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);

    ......
    }

/**
 * 从nameserver获取topic路由信息
 */
public TopicRouteData getTopicRouteInfoFromNameServer(final String topic, final long timeoutMillis,
    boolean allowTopicNotExist) throws MQClientException, InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException {
    ......
    //向nameserver发送请求包，requestCode为RequestCode.GET_ROUTEINFO_BY_TOPIC
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINFO_BY_TOPIC, requestHeader);
    ......
    }
```

producer和nameserver之间通过netty进行网络传输，producer向nameserver发起的请求中添加注册码`RequestCode.GET_ROUTEINFO_BY_TOPIC`。

#### 4.4.2、nameserver返回路由信息

nameserver收到producer发送的请求后，会根据请求中的requestCode进行处理。处理requestCode同样是在默认的网络处理器`DefaultRequestProcessor`中进行处理，最终通过`RouteInfoManager#pickupTopicRouteData`来实现。

##### TopicRouteData结构

在正式解析源码前，我们先看下nameserver返回给producer的数据结构。通过代码可以看到，返回的是一个`TopicRouteData`对象，具体结构如下：

| 属性名            | 类型                          | 说明                             |
| ----------------- | ----------------------------- | -------------------------------- |
| orderTopicConf    | String                        | 顺序消息配置内容，来自于kvConfig |
| queueDatas        | List<QueueData>               | topic队列元数据                  |
| brokerDatas       | List<BrokerData>              | topic分布的broker元数据          |
| filterServerTable | HashMap<String, List<String>> | broker上过滤服务器地址列表       |

其中[QueueData](#1)，[BrokerData](#2)，[filterServerTable](#3)在4.1章节介绍路由元信息时有介绍。

##### 源码分析

在了解了返回给producer的`TopicRouteData`结构后，我们进入`RouteInfoManager#pickupTopicRouteData`方法来看下具体如何实现。代码如下：

```java
public TopicRouteData pickupTopicRouteData(final String topic) {
    TopicRouteData topicRouteData = new TopicRouteData();
    boolean foundQueueData = false;
    boolean foundBrokerData = false;
    Set<String> brokerNameSet = new HashSet<String>();
    List<BrokerData> brokerDataList = new LinkedList<BrokerData>();
    topicRouteData.setBrokerDatas(brokerDataList);

    HashMap<String, List<String>> filterServerMap = new HashMap<String, List<String>>();
    topicRouteData.setFilterServerTable(filterServerMap);

    try {
    try {
    //加读锁
    this.lock.readLock().lockInterruptibly();
    //从元数据topicQueueTable中根据topic名字获取队列集合
    List<QueueData> queueDataList = this.topicQueueTable.get(topic);
    if (queueDataList != null) {
    //将获取到的队列集合写入topicRouteData的queueDatas中
    topicRouteData.setQueueDatas(queueDataList);
    foundQueueData = true;

    Iterator<QueueData> it = queueDataList.iterator();
    while (it.hasNext()) {
    QueueData qd = it.next();
    brokerNameSet.add(qd.getBrokerName());
    }
    //遍历从QueueData集合中提取的brokerName
    for (String brokerName : brokerNameSet) {
    //根据brokerName从brokerAddrTable获取brokerData
    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
    if (null != brokerData) {
    //克隆brokerData对象，并写入到topicRouteData的brokerDatas中
    BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData.getBrokerAddrs().clone());
    brokerDataList.add(brokerDataClone);
    foundBrokerData = true;
    //遍历brokerAddrs
    for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
    //根据brokerAddr获取filterServerList，封装后写入到topicRouteData的filterServerTable中
    List<String> filterServerList = this.filterServerTable.get(brokerAddr);
    filterServerMap.put(brokerAddr, filterServerList);
    }
    }
    }
    }
    } finally {
    //释放读锁
    this.lock.readLock().unlock();
    }
    } catch (Exception e) {
    log.error("pickupTopicRouteData Exception", e);
    }

    log.debug("pickupTopicRouteData {} {}", topic, topicRouteData);

    if (foundBrokerData && foundQueueData) {
    return topicRouteData;
    }

    return null;
    }
```

上面代码封装了`topicRouteData`的`queueDatas`、`brokerDatas`和`filterServerTable`，还有`orderTopicConf`字段没封装，我们再看下这个字段是在什么时候封装的，我们向上看`RouteInfoManager#pickupTopicRouteData`的调用方法`DefaultRequestProcessor#getRouteInfoByTopic`，如下：

```java
public RemotingCommand getRouteInfoByTopic(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    ......
    //这块代码就是上面解析的代码，获取到topicRouteData对象
    TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());

    if (topicRouteData != null) {
    //判断nameserver的orderMessageEnable配置是否打开
    if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
    //如果配置打开了，根据namespace和topic名字获取kvConfig配置文件中顺序消息配置内容
    String orderTopicConf =
    this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
    requestHeader.getTopic());
    //封装orderTopicConf
    topicRouteData.setOrderTopicConf(orderTopicConf);
    }

    byte[] content = topicRouteData.encode();
    response.setBody(content);
    response.setCode(ResponseCode.SUCCESS);
    response.setRemark(null);
    return response;
    }
    //如果没有获取到topic路由，那么reponseCode为TOPIC_NOT_EXIST
    response.setCode(ResponseCode.TOPIC_NOT_EXIST);
    response.setRemark("No topic route info in name server for the topic: " + requestHeader.getTopic()
    + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
    return response;
    }
```

结合这两个方法，我们可以总结出查找topic路由主要分为3个步骤：

+ 调用`RouteInfoManager#pickupTopicRouteData`，从`topicQueueTable`,`brokerAddrTabl`，`filterServerTable`中获取信息，分别填充`queueDatas`、`brokerDatas`和`filterServerTable`
+ 如果topic为顺序消息，那么从KVconfig中获取关于顺序消息先关的配置填充到`orderTopicConf`中
+ 如果找不到路由信息，那么返回code为`ResponseCode.TOPIC_NOT_EXIST`

## 5、小结

本篇文章主要是从源码的角度给大家介绍了rocketmq的nameserver，包括nameserver的启动流程、路由注册、路由剔除和路由发现。我们在了解了nameserver的设计原理之后，也可以回过头思考下在设计过程中一些值得我们学习的小技巧，在此我抛砖引玉提出两点：

+ 启动流程注册JVM钩子用于优雅停机。这是一个编程技巧，我们在实际开发过程中，如果有使用线程池或者一些常驻线程任务时，可以考虑通过注册JVM钩子的方式，在JVM关闭前释放资源或者完成一些事情来保证优雅停机。
+ 更新路由表时需要通过加锁防止并发操作，这里使用的是锁粒度较少的读写锁，允许多个消息发送者并发读，保证消息发送时的高并发，但同一时刻nameserver只处理一个Broker心跳包，多个心跳包请求串行执行，这也是读写锁经典使用场景。

