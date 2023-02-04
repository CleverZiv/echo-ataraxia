---
title: 精读 RocketMQ 源码系列（1）--- NameServer
tags:
  - RocketMQ
  - 源码
categories:
  - RocketMQ
date: 2021-07-10 00:00:00
---

# 一、前言
相信看了 https://juejin.cn/post/6982206285462634510 中推荐的官方中文文档的同学一定已经对 NameServer 有了初步的了解。这里我们总结一下：

1. NameServer 是路由注册中心
2. NameServer 的主要功能是：注册发现和路由删除

在阅读源码时，我希望自己是带着问题去看源码。我也认为，学习的第一步就是要学会提问，有时候一个好的问题比一个精彩的答案更重要。

在了解 NameServer 的技术架构、主要功能之后，可能会提出这么几个问题：

1. NameServer 是怎么实现注册发现和路由删除功能的？
2. NameServer 是集群部署的，但彼此之间不通信，不同 NameServer 之间产生的数据不一致问题，怎么解决？
3. 为什么不选择使用 zookeeper 作为注册中心，而选择自研 NameServer？

针对这三个问题，尝试去源码中寻找答案吧......

<!-- more -->

# 二、启动流程

面对一个东西，我们的第一个问题往往是：它从何而来，生命周期的源头在哪？
对于一些组件，这个问题就是：它是如何启动的？

接下来就先来看看 NameServer 是如何启动的。

启动类：`org/apache/rocketmq/namesrv/NamesrvStartup.java`

![nameserver.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c3d5b571cd045f99da2d07f1b91a5f5~tplv-k3u1fbpfcp-watermark.image)

可以参考上面这张流程图，自己将这部分源码过一遍。

## 2.1 `NameSrvStartup#main0`

首先看到 `main0`方法：

```java
    public static NamesrvController main0(String[] args) {

        try {
            NamesrvController controller = createNamesrvController(args);
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

这个方法就做了两件事：

1. 创建一个 `NamesrvController` 实例
2. 启动该实例

从名字上可以看出 `NamesrvController` 是 `NameServer` 的核心控制器，因此 `NamesrvController` 的启动，主要也是在启动它。

## 2.2 `NameSrvStartup#createNamesrvController`

**填充配置对象的属性**

该方法首先是对两个配置对象进行属性填充，填充方式有两种：

- `-c`：后跟配置文件路径
- `-p`：表示通过`--属性名 属性值`的形式配置

相关属性如下：

```java
public class NamesrvConfig {
   private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    // rocketmq 主目录，通过设置 ROCKETMQ_HOME 配置主目录，在源码环境搭建的过程中有这一步
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    // 用于存储 KV配置的持久化路径
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
    // 默认配置文件路径，不生效。若需要在启动时配置 NameServer 启动属性，使用 -c 配置文件路径 的方法
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
    private String productEnvName = "center";
    private boolean clusterTest = false;
    // 是否支持顺序消息，默认不支持，没大用，就是在处理客户端获取路由数据时标识下，如果支持顺序消息，需要返回对应 topic 的顺序消息配置属性
    private boolean orderMessageEnable = false;
  
  // getter and setter
  ...
}
```

```java
public class NettyServerConfig implements Cloneable {
    private int listenPort = 8888; // NameServer 监听端口，默认会被初始化为 9876
    private int serverWorkerThreads = 8; // Netty 业务线程池线程个数
    private int serverCallbackExecutorThreads = 0; // Netty public 任务线程池线程个数，Netty 网络设计，根据业务类型会创建不同的线程池
    // 比如处理消息发送，消息消费、心跳检测等。如果该业务类型未注册线程池，则由public线程池执行
    private int serverSelectorThreads = 3; // IO 线程池个数，主要是 NameServer、Broker 端解析请求、返回相的线程个数，这类线程主要是处理
    // 网络请求的，解析请求包，然后转发到各个业务线程池完成具体的操作，然后将结果返回给调用方
    private int serverOnewaySemaphoreValue = 256; // send oneway 消息请求并发度（broker 端参数）
    private int serverAsyncSemaphoreValue = 64; // 异步消息发送最大并发度
    private int serverChannelMaxIdleTimeSeconds = 120; // 网络连接最大空闲时间

    private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize; // 网络 socket 发送缓冲区大小
    private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize; // 网络接收端缓存区大小
    private boolean serverPooledByteBufAllocatorEnable = true; // bytebuffer是否开启缓存

    /**
     * make make install
     *
     *
     * ../glibc-2.10.1/configure \ --prefix=/usr \ --with-headers=/usr/include \
     * --host=x86_64-linux-gnu \ --build=x86_64-pc-linux-gnu \ --without-gd
     */
    private boolean useEpollNativeSelector = false; // 是否启用 Epoll IO 模型
```

创建 `NameServerController` 对象

根据以上填充好的配置创建对象，并将配置备份在 `NameServerController` 中

```java
final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

// remember all configs to prevent discard
controller.getConfiguration().registerConfig(properties);
```

## 2.3 `NameSrvStartup#start`

首先执行 `initialize` 方法，这个方法还是做了很多事：

```java
public boolean initialize() {

        // 1. 加载 kv 配置
        this.kvConfigManager.load();

        // 2. 创建 netty 服务端
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);

        // 3. 创建接收客户端请求的线程池
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));

        this.registerProcessor();

        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                // 4. 创建一个定时任务，每隔 10 秒扫描不活跃的 Broker
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);
  ...
}
```

然后做了一件很重要的事：注册了一个钩子方法，监听 JVM 退出事件，在退出时进行 controller 的资源释放。

然后启动 `controller`

```java
// 注册了一个钩子方法，监听 JVM 退出事件，在退出时进行 controller 的资源释放
Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
     @Override
     public Void call() throws Exception {
         controller.shutdown();
         return null;
     }
 }));

controller.start();
```

>“如果代码中使用了线程池，一种优雅停机的方式就是注册一个 JVM 钩子函数，在 JVM 关闭之前，先将线程池关闭，及时释放资源”
>
>——《RocketMQ 技术内幕》

到这还没结束，我们继续往下看，`controller.start()`

```java
    public void start() throws Exception {
      	// 启动 netty 服务端，用于接收客户端请求
        this.remotingServer.start();

        if (this.fileWatchService != null) {
          	// 启动监听TLS配置文件的线程
            this.fileWatchService.start();
        }
    }
```

所以最后可以看到，启动 `NameServer` 就是为了启动这个 netty 服务端，然后就可以接收来自 broker 的注册请求和客户端的路由发现请求。

对于`fileWatchService`，它监听的是 TLS（Transport Layer Security，传输层安全协议） 配置文件，并不涉及业务逻辑。因此就不详细深入了，对网络安全协议感兴趣的同学可以上网学习。

至此，启动流程分析完毕。最后来小结下：

1. 首先通过命令行参数、配置文件、默认配置填充`NamesrvConfig`、`NettyServerConfig`
2. 根据以上两个配置对象创建`NamesrvController`，并备份配置信息到`NamesrvController`中
3. 启动`NamesrvController`实例，实际上启动的是 netty 服务端

怎么样，其实并不是很难对不对，接下来我们继续问问题。

启动之后，NameServer 又做了哪些事呢？本文一开始就已经给出了答案：**注册发现和路由删除**，紧接着文章开头其实就提出了一个问题：

> NameServer 是怎么实现注册发现和路由删除功能的？

继续从源码中寻找答案......

# 三、注册发现

注册和发现其实是两个动作，但针对的都是 broker。注册是指 broker 注册到 NameServer，发现是指 producer 和 consumer 通过 NameServer 发现 broker。

## 3.1 注册

既然是“broker 注册到 NameServer”，那当然要先去 broker 的源码中寻找答案。问题是 broker 源码那么多，我该从哪下手呢？这时候可以停下来想想，如果是我自己去设计一个系统，这个系统需要将自己的存活状态上报至一个注册中心，我会选择在什么时候去注册呢？

应该容易想到，最好是启动成功的时候就马上去注册，然后与注册中心建立一个心跳机制，一直不停的告诉注册中心：我还活着！

不管这个想法对不对，但至少这个时候有方向了，我知道应该去 broker 的一大堆源码中，先找它的启动流程源码。

> 这就是一直强调的带着问题去读源码，通过问问题，让自己阅读源码时更具有目的性；通过对问题的思考，可以提出自己的猜想，如果猜想是对的，恭喜你，你将会收获成就感；如果猜想是错的，更要恭喜你，你可以对比自己的猜想和源码的实现，看看差在哪个地方，这个地方就是你提升的空间。

### 3.1.1 broker 启动流程

代码位置：`BrokerStartup#main`

流程图如下：

![broker_startup.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/399263d9dc04479aa1f3ce6bc3c8fb21~tplv-k3u1fbpfcp-watermark.image)

以上流程图的大致步骤是：

1. 创建 broker 的核心控制器 `BrokerController`
2. 启动 `BrokerController`：这一步会启动很多服务，如消息存储服务、netty 服务端、`fileWatchService`、以及我们重点关心的给 NameServer 发送心跳的服务

这里我们重点分析图中的第 18~20：

```java
        // 1. 在 broker 启动时，先做一次注册
        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            startProcessorByHa(messageStoreConfig.getBrokerRole());
            handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
            this.registerBrokerAll(true, false, true);
        }

        // 2. 接下来先延迟 10s，然后每隔30s（brokerConfig.getRegisterNameServerPeriod()我配置的是30s）进行一次注册
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

为什么要先延迟 10s？

> 因为 broker 刚刚已经发送了注册请求，没有必要立马再进行注册，所以定时任务线程池先延迟了10s。这种设计很细节，但在业务上是有效的，避免不必要的资源浪费。

要正确理解 `registerBrokerAll`方法的意思：这个方法并不是把“所有的broker”都注册，而是把该 broker 注册到所有的 NameServer 上，这一点在后面的源码中可以得到验证。

### 3.1.2 `registerBrokerAll`

这个方法有三个参数：

```java
boolean checkOrderConfig, // 是否校验 顺序消息配置
boolean oneway,  // 是否是 单向发送，单向发送不接收返回值
boolean forceRegister // 是否强制注册
```

```java
    public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway, boolean forceRegister) {
        // topicConfigWrapper 中封装了 该broker 上的 topic 信息和 dataVersion
        TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();

        // 这块代码的作用是将 topicConfigWrapper 中的值取出来重新封装一遍，又再塞回 topicConfigWrapper，我理解是为了将 this.brokerConfig.getBrokerPermission()
        // 的属性值 set 进去。不过这并不是很重要的细节，我们只要知道 topicConfigWrapper 至少包含了该broker 上的 topic 信息和 dataVersion 即可
        if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission())
            || !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {
            ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>();
            for (TopicConfig topicConfig : topicConfigWrapper.getTopicConfigTable().values()) {
                TopicConfig tmp =
                    new TopicConfig(topicConfig.getTopicName(), topicConfig.getReadQueueNums(), topicConfig.getWriteQueueNums(),
                        this.brokerConfig.getBrokerPermission());
                topicConfigTable.put(topicConfig.getTopicName(), tmp);
            }
            topicConfigWrapper.setTopicConfigTable(topicConfigTable);
        }

        if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
            this.getBrokerAddr(),
            this.brokerConfig.getBrokerName(),
            this.brokerConfig.getBrokerId(),
            this.brokerConfig.getRegisterBrokerTimeoutMills())) {
            doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
        }
    }
```

重点看一下最下面的这块`if`语句的内容，由于`forceRigister == true`，所以后面的 `needRegister`方法的逻辑并没有机会执行。但这里还是简单讲一下这个方法：

1. broker 请求 NameServer，查询 broker 相关的配置在 NameServer 端的数据版本，请求类型：`QUERY_DATA_VERSION = 322`
2. NameServer 接收请求，`DefaultRequestProcessor#processRequest`，会将从 broker 发送过来的 dataVersion 和 NameServer 存储的进行比较，如果不相等，则需要在 NameServer 端更新 broker 的心跳更新时间`lastUpdateTimestamp`；如果相等返回`changed == false`的结果
3. Broker 端处理所有 NameServer 返回的结果，只要有一个 `changed == true`，那么`needRegister == true`

也就是需要执行`doRegisterBrokerAll`

当然在当前 broker 启动过程中，是一定会执行 `doRegisterBrokerAll`的。

### 3.1.3 `doRegisterBrokerAll`

这个方法主要做了两件事：

1. 调用 `brokerOuterAPI.registerBrokerAll`进行注册
2. 处理注册结果 `registerBrokerResultList`：进行 master 地址的更新、顺序消息Topic的配置更新

重点在第一步。

`BrokerOuterAPI` 这个类是 broker 对外交互的类，其中封装了 `RemotingClient remotingClient` ，在这里是作为客户端向 NameServer 发送真正的注册请求。

看一下`registerBrokerAll` 中核心的一块代码

```java
 // CountDownLatch 使得只有所有 nameServer 的响应结果都返回时才会继续执行后续的逻辑
            final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
            // 遍历所有的 NameServer，并将注册任务 registerBroker 丢进 brokerOuterExecutor 线程池中执行
            for (final String namesrvAddr : nameServerAddressList) {
                brokerOuterExecutor.execute(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            RegisterBrokerResult result = registerBroker(namesrvAddr,oneway, timeoutMills,requestHeader,body);
                            if (result != null) {
                                registerBrokerResultList.add(result);
                            }

                            log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                        } catch (Exception e) {
                            log.warn("registerBroker Exception, {}", namesrvAddr, e);
                        } finally {
                            // 每返回一个结果，减1
                            countDownLatch.countDown();
                        }
                    }
                });
            }

            try {
                // 主线程阻塞在此，直到所有的 countDownLatch 减为0
                countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
            }
```

至于`registerBroker`，里面的逻辑就是去调用 netty 客户端的 `invokeSync`或`invokeOneway`，去向 NameServer 发送请求。具体的通信过程和原理涉及到 netty，不是本文的重点。笔者也计划后续会进行 netty 源码的解析，敬请期待。

值得一提的是，上面这段代码采用多线程的方式，使用到了：

- `CountDownLatch`
- `BrokerFixedThreadPoolExecutor` ：父类是 `ThreadPoolExecutor`
- `CopyOnWriteArrayList`：存储线程执行结果，因为存在多线程的写操作，所以需要使用并发安全的容器

对并发编程感兴趣的同学可以学习下这里的用法，同时有兴趣了解其原理的，可以查看 JDK 源码。

到这一步，对于 **注册** 这件事来说，broker 端算是完成了它的工作，后续就是 NameServer 接收到请求去处理的事了。

### 3.1.4 NameServer 处理注册请求

我们再回到 NameServer ，看看是怎么处理注册请求的。我们稍微思考下，这个处理请求的代码入口在哪呢？（实际过程中是通过代码调试得知入口的，但代码调试得到的结果实际上有点像翻答案，我们可以尝试自己先思考下）

当然首先是 netty 服务端先接收到请求，因此我们先去看一下 `NettyRemotingServer`，看了一圈发现这个类里并没有类似处理请求的方法。但是这个类集成了 `NettyRemotingAbstract`，我们继续在这里找一下，发现了这个类里有个方法叫 `processRequestCommand`。

这个类最后会调用到 NameServer 的 `DefaultRequestProcessor#processRequest`，这个方法中帮助 NameServer 处理来自客户端和 Broker 的各种请求。

流程图：

![namesrv_process_request.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/078b094f7695425da622348cc0432e45~tplv-k3u1fbpfcp-watermark.image)


首先方法一进来就是一个 `switch case`，找到 `REGISTER_BROKER`

```java
switch (request.getCode()) {
            ...
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
    						// 根据版本区分，实际两个方法没有很大的区别
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
}
```

中间的请求crc校验、请求参数的处理过程我们不详细看了。值得一提的是，NameServer 中管理路由信息的类是 `RouteInfoManager`，其维护了五张表：

```java
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;// Topic 消息队列的路由信息，消息发送时根据此表进行负载均衡
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;// broker 地址信息
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;// 集群信息
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;// 存活的broker
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

看一下更新 `brokerLiveTable` 的逻辑，注意：其它几张表也会被更新

```java
  BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                    new BrokerLiveInfo(
                        System.currentTimeMillis(),
                        topicConfigWrapper.getDataVersion(),
                        channel,
                        haServerAddr));
```

把当前系统时间作为 `lastUpdateTimestamp` broker 上报心跳的时间。

后续会将处理结果封装并返回到请求响应中，通过 netty 再返回给 broker，与上面分析的 broker 端的流程形成闭环。

至此，关于注册的源码分析全部完成，最后来小结一下

### 3.1.5 小结

1. broker 服务器在启动时会向**所有的 NameServer **注册，并建立长连接，之后每隔30s发送一次心跳，内容包括 brokerId，broker 地址，名称和集群信息
2. NameServer 接收到心跳包后，会将整个消息集群的数据存入到 RouteInfoManager 的几个HashMap中，并更新 `lastUpdateTimestamp`

其中涉及到的一些可深入的技术点：

1. 并发编程，包括线程池的使用、并发组件（CountDownLatch、CopyOnWriteArrayList）、锁的使用（NameServer 更新几张路由表的时候）
2. netty 相关的网络编程知识

## 3.2 发现

客户端在从 NameServer 中获取 broker 相关信息，这个过程就是路由发现。我们以生产者为例分析路由发现。

路由信息在生产者中存放在 `DefaultMQProducerImpl.topicPublishInfoTable`中

```java
   private final ConcurrentMap<String/* topic */, TopicPublishInfo> topicPublishInfoTable =
        new ConcurrentHashMap<String, TopicPublishInfo>();
```

是一个并发安全的容器，为什么要使用`ConcurrentMap`呢，因为它的写入口实际上有两个，也就是生产者路由发现的时机有两个：

1. 发送消息时会去检查 `topicPublishInfoTable` 是否为空或可用，不符合条件则去 NameServer 中查询
    - `DefaultMQProducer#send`，实际调用的是 `DefaultMQProducerImpl#tryToFindTopicPublishInfo`
2. 生产者启动时也会启动一个定时任务，定时从 NameServer 上拉取 topic 信息
    - 这个代码路径是：`DefaultMQProducerImpl#start` -> `mQClientFactory.start()` -> `MQClientInstance#this.startScheduledTask()`-> `MQClientInstance#this.updateTopicRouteInfoFromNameServer();`

为什么要有两个入口呢？

- 这是因为路由信息变更时，nameserver不会主动推送，需要客户端主动拉取路由信息才能将客户端上路由信息进行更新。请求类型 `GET_ROUTEINFO_BY_TOPIC`，调用`RouteInfoManager`的`pickupTopicRouteData`方法，这样设计的目的是降低 NameServer 的复杂度。因此第2种方式是必不可少的。
- 而第一种方式，就更好理解了，按需更新，这里的需是指发消息的需求。

两个方法都会调用下面这个方法`MQClientInstance`：

```java
public boolean updateTopicRouteInfoFromNameServer(final String topic, boolean isDefault,
    DefaultMQProducer defaultMQProducer)
```

最终调用 `MQClientAPIImpl#getTopicRouteInfoFromNameServer`

```java
    public TopicRouteData getTopicRouteInfoFromNameServer(final String topic, final long timeoutMillis,
        boolean allowTopicNotExist) throws MQClientException, InterruptedException, RemotingTimeoutException, RemotingSendRequestException, RemotingConnectException {
        GetRouteInfoRequestHeader requestHeader = new GetRouteInfoRequestHeader();
        requestHeader.setTopic(topic);

      // 请求code: GET_ROUTEINFO_BY_TOPIC，最终又会前面提到的 processRequest 方法中，根据code找到处理逻辑
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_ROUTEINFO_BY_TOPIC, requestHeader);

        RemotingCommand response = this.remotingClient.invokeSync(null, request, timeoutMillis);
        assert response != null;
        switch (response.getCode()) {
            case ResponseCode.TOPIC_NOT_EXIST: {
                if (allowTopicNotExist) {
                    log.warn("get Topic [{}] RouteInfoFromNameServer is not exist value", topic);
                }

                break;
            }
            case ResponseCode.SUCCESS: {
                byte[] body = response.getBody();
                if (body != null) {
                    return TopicRouteData.decode(body, TopicRouteData.class);
                }
            }
            default:
                break;
        }

        throw new MQClientException(response.getCode(), response.getRemark());
    }
```

NameServer 端处理请求的逻辑比较简单，就是查询一下路由信息然后返回，这里就不再赘述。

至此，NameServer 的注册发现分析完毕

# 四、路由删除

路由删除有两个触发点：

1. broker 非正常关闭，NameServer 发现 broker 无响应，将其删除。详细过程：
    - Broker 每隔 30s 向 nameserver 发送心跳包，并更新 brokerLiveTable 中的信息，尤其是 lastUpdateTimestamp。
    - namaserver 每隔10s会扫描 brokerLiveTable，如果发现 lastUpdateTimestamp 距离当前时间已经超过了120s，则认为 Broker 宕机，会进行路由删除操作。
2. Broker 正常关闭时，与 NameServer 断开长连接，会执行 unregisterBroker 指令

路由删除比较简单，大家可以对照源码验证下这里讲的两个触发点。

# 五、前言中提到的几个问题

现在来回顾一下文章开头我们提到的三个问题。第一个问题已经解答完了，重点看一下第二和第三个问题

## 5.1 NameServer 如何解决数据不一致的问题

首先要理解为什么 NameServer 会有数据不一致的问题。因为 NameServer 虽然是集群部署，但各个节点之间是相互独立不进行通信的。那么在进行路由注册、删除时，不同节点之间存在不一样数据的情况是必然存在的

如何解决呢？事实上，RocketMQ 并不认为这是一个需要去解决的问题。因为 Topic 路由信息本身就不需要追求集群中各个节点的强一致性，只需要做到最终一致性。

说白了，NameServer 的各个节点根本不关心自己的数据和别的节点是不是一致。关心这件事的人是生产者和消费者。而客户端关心这件事的本质其实是：**我希望我拿到的路由信息尽量是正确的，可用的**，也就是我根据获取到路由信息选择了一个 broker 去发送消息，这个 broker 是能正常接收到的。

那这就有问题了，因为上面讲路由删除的时候，提到了： NameServer 发现 lastUpdateTimestamp 距离当前时间已经超过了120s，才认为 Broker 宕机，会进行路由删除操作。也就是说，会有 2 分钟的空档，这 2 分钟，很可能生产者会向一个已经宕机的 broker 发送消息。那这种情况怎么办呢？

这个问题先按下不表，因为答案并不在 NameServer，而是在 producer 中。重要的是，现在我又有了一个好问题！

## 5.2 为什么rocketmq选择自己开发一个NameServer，而不是使用zk

事实上，在RocketMQ的早期版本，即MetaQ 1.x和MetaQ 2.x阶段，也是依赖Zookeeper的。但MetaQ 3.x（即RocketMQ）却去掉了ZooKeeper依赖，转而采用自己的NameServer。

因为 RocketMQ 的设计理念是简单高效，并且 RocketMQ 的架构设计决定了它只需要一个轻量级的元数据服务器就足够了，只需要保持最终一致，而不需要Zookeeper这样的强一致性解决方案。这么做的好处很明显：不需要再依赖另一个中间件，从而减少整体维护成本。

这里可以稍微扩展下，其实选择 NameServer 还是选择 Zookeeper 代表了在分布式系统中的设计侧重点。

根据CAP理论， RocketMQ 在注册中心这个模块的设计上选择了 AP 模式的 NameServer，而不是 CP 模式的 Zookeeper

**不使用 zookeeper 的原因是当 RocketMQ 没满足A（可用性）带来的影响比较大，影响稳定性**

**Zookeeper CP 的适用场景：**

- 分布式选主，主备高可用切换等场景下有不可替代的作用，而这些需求往往多集中在大数据、离线任务等相关的业务领域，因为大数据领域，讲究分割数据集，并且大部分时间分任务多进程 / 线程并行处理这些数据集，但是总是有一些点上需要将这些任务和进程统一协调，这时候就是 ZooKeeper 发挥巨大作用的用武之地。
- 但是在交易场景交易链路上，在主业务数据存取，大规模服务发现、大规模健康监测等方面有天然的短板，应该竭力避免在这些场景下引入 ZooKeeper，在生产实践中，应用对 ZooKeeper 申请使用的时候要进行严格的场景、容量、SLA 需求的评估。

**NameServer 的适用场景：**

- NameServer作为一个名称服务，需要提供服务注册、服务剔除、服务发现这些基本功能，但是NameServer节点之间并不通信，容忍在某个时刻各个节点数据可能不一致的情况下 **所以可以使用 CP，也可以使用AP**，但是大数据使用CP，在线服务则AP，分布式协调、选主使用CP，服务发现使用 AP

# 参考资料：
- RocketMQ 4.8.0 源码
- https://github.com/apache/rocketmq/tree/master/docs/cn
- https://github.com/DillonDong/notes/blob/master/RocketMQ/RocketMQ-03.md
- 《RocketMQ 技术内幕》