---
title: 精读 RocketMQ 源码系列（2）--- Producer
tags:
  - RocketMQ
  - 源码
categories:
  - RocketMQ
date: 2021-07-12 00:00:00
---
# 一、前言

在上篇 [精读 RocketMQ 源码系列（1）--- NameServer](https://juejin.cn/post/6983267530181705742) 中我们有一个遗留问题：

由于从 broker 宕机到 NameServer 路由删除有120秒的间隔，会导致生产者可能会向一个已经宕机的 broker 发送消息，这种情况 RocketMQ 是如何处理的呢？

本文会对这个问题做一个解释。同时本文会从源码角度着重分析两个问题：

1. Producer 的启动流程是怎样的？
2. Producer 是如何把消息发送到 broker 上的？

需要强调的是，本文不会详细讲和 Producer、消息相关的一些概念，对这一块不太熟悉的同学可以在 [精读 RocketMQ 源码系列（0）---开篇词](https://juejin.cn/post/6982206285462634510) 这篇中找到官方的中文文档，进行了解。

# 二、启动流程

RocketMQ 中生产者的核心类是 `DefaultMQProducer`，启动流程的源码入口是 `DefaultMQProducer#start()`。流程图如下：

![producer_startup.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/daf0157149c14e2099f384ff2ac94714~tplv-k3u1fbpfcp-watermark.image)


大家可以对照流程图阅读源码，这里做个简单总结，启动流程步骤大致如下：

1. 检查生产者组是否合法
2. 获取 MQClientInstance
3. 将当前生产者注册到 MQClientInstance（注册可以理解为将 producer set 到 MQClientInstance）
4. 启动 MQClientInstance 客户端

接下来对一些小细节分析下

## 2.1 MQClientManager

字面意思上理解，MQ 客户端管理者。

整个 JVM 中只存在一个 MQClientManager 实例，为什么呢？

```java
public class MQClientManager {
    private final static InternalLogger log = ClientLogger.getLog();
    // MQClientManager 实例，对外暴露
    private static MQClientManager instance = new MQClientManager();
    private AtomicInteger factoryIndexGenerator = new AtomicInteger();
    // MQClientInstance 缓存表
    private ConcurrentMap<String/* clientId */, MQClientInstance> factoryTable =
        new ConcurrentHashMap<String, MQClientInstance>();

    private MQClientManager() {

    }

    public static MQClientManager getInstance() {
        return instance;
    }
...
}
```

对外暴露的`instance`是一个静态变量，只有在类初次加载的时候会被初始化，相关信息存储在 JVM 中，具有唯一性。

它维护一个 MQClientInstance 缓存表：`ConcurrentMap<String/* clientId */, MQClientInstance> factoryTable`。同一个 clientId 只会创建一个 MQClientInstance。所以总结来说，在一个 JVM 实例中，只会有一个 MQClientManager 存在，但如果运行了多个应用程序（客户端），就会存在多个 MQClientInstance。

我们可以看一下 clientId 是怎么生成的：

```java
    public String buildMQClientId() {
        StringBuilder sb = new StringBuilder();
        sb.append(this.getClientIP());

        sb.append("@");
        sb.append(this.getInstanceName());
        if (!UtilAll.isBlank(this.unitName)) {
            sb.append("@");
            sb.append(this.unitName);
        }

        return sb.toString();
    }
```

`clientId`是由：ip地址、实例名、unitName（可选）拼接而成的。那这就有问题了，同一实例中 ip 地址和实例名都一样啊。

其实这里实例名已经被修改了，可以看到这里：已经把实例名改成了进程id

```java
   if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
      }
```

## 2.2 MQClientInstance

MQClientInstance 中封装了 RocketMQ 网络处理的 API，是消费者和生产者与 NameServer、Broker 通信的网络通道。

```java
  if (consumerEmpty) {
      if (id != MixAll.MASTER_ID)
         continue;
  }
```

以上代码片段位于`MQClientInstance`的`sendHeartbeatToAllBroker()`方法，表明了生产者只会向 Master 的 broker 发送心跳

创建 MQClientInstance 的源码：

```java
    public MQClientInstance getOrCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
        String clientId = clientConfig.buildMQClientId();
        MQClientInstance instance = this.factoryTable.get(clientId);
        if (null == instance) {
            instance =
                new MQClientInstance(clientConfig.cloneClientConfig(),
                    this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);
            MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);
            if (prev != null) {
                instance = prev;
                log.warn("Returned Previous MQClientInstance for clientId:[{}]", clientId);
            } else {
                log.info("Created new MQClientInstance for clientId:[{}]", clientId);
            }
        }

        return instance;
    }
```

利用 ConcurrentMap 来保证并发时不会出错，同时通过双重校验，保证多线程场景下，返回的实例是同一个。

## 2.3 心跳机制

在 MQClientInstance 启动之后，还有一行代码很重要：

```java
this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
```

注意，这里的 AllBroker 当然不是集群中所有的 Broker，而是与当前客户端相关的 Broker。

# 三、发送消息

启动完成之后，Producer 就可以开始发送消息了。可以看到在 `DefaultMQProducer` 中发送消息的方法非常多，大致可进行如下分类：

**根据消息类型：**

- 普通消息：没有什么特殊的地方，就是普通消息
- 延迟消息：延时消息在投递时，需要设置指定的延时级别，即等到特定的时间间隔后消息才会被消费者消费。mq服务端 ScheduleMessageService中，为每一个延迟级别单独设置一个定时器，定时(每隔1秒)拉取对应延迟级别的消费队列。目前RocketMQ不支持任意时间间隔的延时消息，只支持特定级别的延时消息，即 "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"
- 顺序消息：对于指定的一个 Topic，Producer保证消息顺序的发到一个队列中，消费的时候需要保证队列的数据只有一个线程消费。
- 事务消息：通过两阶段提交、状态定时回查来保证消息一定发到broker。具体流程见下图


![image-20210712140144270.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6007effbb485405cb3b6df30df31b635~tplv-k3u1fbpfcp-watermark.image)


**根据发送方式：**

- 可靠同步发送：同步发送是指消息发送方发出数据后，会在收到接收方发回响应之后才发下一个数据包的通讯方式。
- 可靠异步发送：异步发送是指发送方发出数据后，不等接收方发回响应，接着发送下个数据包的通讯方式。 MQ 的异步发送，需要用户实现异步发送回调接口（SendCallback）。消息发送方在发送了一条消息后，不需要等待服务器响应即可返回，进行第二条消息发送。发送方通过回调接口接收服务器响应，并对响应结果进行处理。
- 单向（Oneway）发送：特点为发送方只负责发送消息，不等待服务器回应且没有回调函数触发，即只发送请求不等待应答。 此方式发送消息的过程耗时非常短，一般在微秒级别。



这里，我们选择分析发送同步消息，发送消息的流程图如下：

![send_msg.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a2cf2e3b74a40c79f8a04676ff1ffe6~tplv-k3u1fbpfcp-watermark.image)

简单总结为以下几个主要步骤：

1. 校验消息：主题和消息体的校验
2. 查找主题路由信息：注意这里查找出来的信息是以 MessageQueue 维度存储的
3. 选择消息队列：第2步中会返回待发送消息对应主题下的所有 Broker 的 MessageQueue 的信息，这一步就是在这些 MessageQueue 中选择一个进行发送
4. 执行具体发送消息的动作

## 3.1 选择 MessageQueue --- 默认方案

`MQFaultStrategy#selectOneMessageQueue`

```java
    public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
        if (this.sendLatencyFaultEnable) {
            ...... // 业务逻辑
            return tpInfo.selectOneMessageQueue();
        }

        return tpInfo.selectOneMessageQueue(lastBrokerName);
    }
```

根据参数 `sendLatencyFaultEnable` ，我们有两种方案，一种称之为默认选择方案，另一种为启用故障延迟后的方案。可以看到启用故障延迟后的方案实际调用了默认的方案，我们先看看默认方案是如何做的？

`TopicPublishInfo#selectOneMessageQueue`

```java
    public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
      // lastBrokerName 表示上次发送消息给了哪个 broker
        if (lastBrokerName == null) {
            return selectOneMessageQueue();
        } else {
            for (int i = 0; i < this.messageQueueList.size(); i++) {
              // sendWhichQueue 记录了一个 index，可自增，使用到了 ThreadLocal
                int index = this.sendWhichQueue.getAndIncrement();
                int pos = Math.abs(index) % this.messageQueueList.size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = this.messageQueueList.get(pos);
                if (!mq.getBrokerName().equals(lastBrokerName)) {
                    return mq;
                }
            }
            return selectOneMessageQueue();
        }
    }
```

可以看到，选择 MessageQueue 的方案其实很简单：

- 维护一个可自增的值 `sendWhichQueue` 每次将其与总的 MessageQueue 的数量取模获得新的 MessageQueue 的下标；

- 当选择的新的 MessageQueue 属于上次的 Broker 时，重新选择。

  这么做可以使得相邻两次发送的消息不会发送到同一个 broker 上，实现负载均衡；同时当其中一个 broker 宕机时，可以最大限度减少消息发送到宕机的 broker 上。

## 3.2 选择 MessageQueue --- 故障延迟方案

当 `sendLatencyFaultEnable` 开启时，我们会执行以下逻辑分支：

```java
    if (this.sendLatencyFaultEnable) {
            try {
                int index = tpInfo.getSendWhichQueue().getAndIncrement();
                for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                    int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                    if (pos < 0)
                        pos = 0;
                    MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                    if (latencyFaultTolerance.isAvailable(mq.getBrokerName()))
                        return mq;
                }

                final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
                int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
                if (writeQueueNums > 0) {
                    final MessageQueue mq = tpInfo.selectOneMessageQueue();
                    if (notBestBroker != null) {
                        mq.setBrokerName(notBestBroker);
                        mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                    }
                    return mq;
                } else {
                    latencyFaultTolerance.remove(notBestBroker);
                }
            } catch (Exception e) {
                log.error("Error occurred when selecting message queue", e);
            }

            return tpInfo.selectOneMessageQueue();
        }
```

整体逻辑，总结如下：

1. 选择出一个 MessageQueue，方法与默认方案的方法相同
2. 校验该 MessageQueue 是否可用，可用则直接返回
3. 不可用则在尝试从规避的 Broker 中选择一个可用的 broker，如果选出来的 broker 有写队列则返回
4. 如果无可写队列则最后再用默认方案选出一个队列返回

故障延迟机制的核心是使用了

```java
ConcurrentHashMap<String, FaultItem> faultItemTable = new ConcurrentHashMap<String, FaultItem>(16);    class FaultItem implements Comparable<FaultItem> {        private final String name;  // broker name        private volatile long currentLatency;  // 消息发送故障延迟时间        private volatile long startTimestamp;  // 不可用时间戳，当前时间不超过这个时间戳表示该需要规避该 broker				...    }
```

每次选择出一个队列之后，需要通过内存的一张表`faultItemTable`去判断当前这个Broker是否在其中，如果不在证明可用，直接返回即可；如果在，证明可能不可用，需要再判断一下

```go
   public boolean isAvailable() {            
       return (System.currentTimeMillis() - startTimestamp) >= 0;   
       }
```

该表是每次发送消息的时候都会更新。



**再之后就是调用发送消息的核心方案 `sendKernelImpl`，进行消息的组装和发送。感兴趣的同学可对照流程图读一下源码**



# 四、前言中提到的几个问题

启动流程和消息发送分别都已经在第二和第三节中叙述了。现在来看下生产者是如何应对 broker 宕机的问题的，同时这也是上一篇文章中遗留的一个问题。

## 4.1 生产者是如何应对 broker 宕机

我们来看看发送消息的方法 `sendDefaultImpl` 中可以看到有这么一段 `for `循环

```java
  for (; times < timesTotal; times++) {    ...// 消息发送             sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);       endTimestamp = System.currentTimeMillis();       this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);       switch (communicationMode) {           case ASYNC:           return null;           case ONEWAY:           return null;           case SYNC:           if (sendResult.getSendStatus() != SendStatus.SEND_OK) {               // 开启了消息重试开关，则进行消息重试               if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {                   continue;                }           }           // 未开启，则直接返回结果，结束本次消息发送            return sendResult;            default:                break;        }  }
```

`timesTotal` 为重试次数+1，也就是说，开启了消息重试开关，生产者会进行消息重试。

结合刚刚我们讲的选择 MessageQueue 的方案，不论是默认方案还是故障延迟方案，在重新选择时，都会规避上一次的 broker。因此消息重试时是不会再选择到导致本次消息发送失败的 broker 的。

总结来说，RocketMQ 通过 消息重试+broker规避 实现了消息发送的高可用

## 4.2 生产环境下为什么不能自动创建Topic？

很多时候我们会被告知，生产环境下不要将 `autoCreateTopicEnable` 设置为 true，因为这会使得：自动新建的Topic只会存在于一台Broker上，后续所有对该Topic的请求都会局限在单台Broker上，造成单点压力。

但是为什么会这样呢？我们接下来来分析下

**1、broker 启动时，会加载在本地创建的 topic**

```java
 public BrokerController(
        final BrokerConfig brokerConfig,
        final NettyServerConfig nettyServerConfig,
        final NettyClientConfig nettyClientConfig,
        final MessageStoreConfig messageStoreConfig
    ) {
        this.brokerConfig = brokerConfig;
        this.nettyServerConfig = nettyServerConfig;
        this.nettyClientConfig = nettyClientConfig;
        this.messageStoreConfig = messageStoreConfig;
        this.consumerOffsetManager = new ConsumerOffsetManager(this);
   			// 加载 topic 配置
        this.topicConfigManager = new TopicConfigManager(this);
        this.pullMessageProcessor = new PullMessageProcessor(this);
   			...
 }
```

在 `TopicConfigManager`的构造函数中，会判断 `autoCreateTopicEnable` ，然后对默认主题进行加载：

```java
    if (this.brokerController.getBrokerConfig().isAutoCreateTopicEnable()) {
                String topic = TopicValidator.AUTO_CREATE_TOPIC_KEY_TOPIC;
                TopicConfig topicConfig = new TopicConfig(topic);
                TopicValidator.addSystemTopic(topic);
                topicConfig.setReadQueueNums(this.brokerController.getBrokerConfig()
                    .getDefaultTopicQueueNums());
                topicConfig.setWriteQueueNums(this.brokerController.getBrokerConfig()
                    .getDefaultTopicQueueNums());
                int perm = PermName.PERM_INHERIT | PermName.PERM_READ | PermName.PERM_WRITE;
                topicConfig.setPerm(perm);
                this.topicConfigTable.put(topicConfig.getTopicName(), topicConfig);
            }
```

可以看到这里创建了一个名为 `AUTO_CREATE_TOPIC_KEY_TOPIC`，读写队列都为 8 的主题信息。

接着该信息会被同步到 NameServer 上。

这里要注意的是，每一个开启了`autoCreateTopicEnable` 的broker 都会在启动时去加载默认主题信息并上报至 NameServer。那么在 NameServer 处存储的关于默认主题就会有多个 broker 信息

**2、生产者发送消息，查询 topic 信息**

生产者发送消息时，首先会使用 `tryToFindTopicPublishInfo` 去查询主题信息：

```java
    private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
        TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
        if (null == topicPublishInfo || !topicPublishInfo.ok()) {
            this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
        }

        if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
            return topicPublishInfo;
        } else {
            // 新创建的主题走这个分支
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
            topicPublishInfo = this.topicPublishInfoTable.get(topic);
            return topicPublishInfo;
        }
    }
```

当然，现在主题的消息只在生产者这端存在，所以肯定找不到，最后只能走最下面的这个分支，来到 `updateTopicRouteInfoFromNameServer` 并执行以下逻辑：

```java
TopicRouteData topicRouteData;
if (isDefault && defaultMQProducer != null) {
   // 获取到 broker 创建的默认的主题信息
		topicRouteData = 				  this.mQClientAPIImpl.getDefaultTopicRouteInfoFromNameServer(defaultMQProducer.getCreateTopicKey(),
                            1000 * 3);
		if (topicRouteData != null) {
      // 将默认主题信息的读写队列数目更改为4
    		for (QueueData data : topicRouteData.getQueueDatas()) {
        		int queueNums = Math.min(defaultMQProducer.getDefaultTopicQueueNums(), data.getReadQueueNums());
            data.setReadQueueNums(queueNums);
            data.setWriteQueueNums(queueNums);
         }
     }
} 
....
TopicRouteData old = this.topicRouteTable.get(topic);
boolean changed = topicRouteDataIsChange(old, topicRouteData);
```

接着生产者会选择一个 MessageQueue，并将消息封装进行发送。注意，这里发送出去的消息，并不是默认主题的，而是消息本身的主题：

```java
// 代码位置：sendKernelImpl requestHeader.setTopic(msg.getTopic());
```

然后 broker 接收到消息后首先会对消息进行校验：`AbstractSendMessageProcessor#msgCheck`

```java
// 查询消息头的 topic 是否存在
        TopicConfig topicConfig =
            this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
        if (null == topicConfig) {
            int topicSysFlag = 0;
            if (requestHeader.isUnitMode()) {
                if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
                } else {
                    topicSysFlag = TopicSysFlag.buildSysFlag(true, false);
                }
            }

            log.warn("the topic {} not exist, producer: {}", requestHeader.getTopic(), ctx.channel().remoteAddress());
            // 不存在则根据消息中的相关信息进行主题的创建
            topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageMethod(
                requestHeader.getTopic(),
                requestHeader.getDefaultTopic(),
                RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                requestHeader.getDefaultTopicQueueNums(), topicSysFlag);
```

那么此时，新主题的信息就在这台 broker 上有了。

**3、接下来的步骤**

接下来事情的走向就是：

- broker 通过心跳机制上报主题消息，包括刚刚新创建的主题
- NameServer 接收来自 broker 的主题信息并更新路由信息
- 生产者再次往刚创建的新主题发消息时，发现新主题在 NameServer 端有路由，那就取到路由信息，按照路由信息进行发送

发现问题没有？到目前为止，虽然新主题的路由信息已经在 NameServer 存在了，但是只有一个 broker，并且不会再有更新。

以上。

**那有没有方法解决这个问题呢？有！**

方法一：

`autoCreateTopicEnable` 置为 false，所以生产环境需要用命令行工具手动创建Topic，可以用集群模式去创建（这样集群里面每个broker的queue的数量相同），也可以用单个broker模式去创建（这样每个broker的queue数量可以不一致）。

方法二：

连续快速地发送9条消息以上（单个 broker 的写队列默认是8）。

因为上面的关键点在于，当第一条消息发送出去之后，接收到消息的 Broker 便会在本地创建 topic，然后通过心跳机制同步到 NameServer，这个时间间隔最多只有30s。如果我们在最快时间内发送9条消息以上，那么消息就会被多个 broker 接收到，最后 NameServer 上的路由信息也将是多个 broker。

但这个方式太不可控，因此生产上我们还是使用方法一。

# 参考资料：
- RocketMQ 4.8.0 源码
- https://github.com/apache/rocketmq/tree/master/docs/cn
- https://github.com/DillonDong/notes/blob/master/RocketMQ/RocketMQ-03.md
- 《RocketMQ 技术内幕》

**最后**
- 文章若有错误，欢迎评论留言指出，也欢迎转载，转载请注明出处；
- 文章源码 github 地址：https://github.com/CleverZiv/rocketmq-master （带中文注释）



