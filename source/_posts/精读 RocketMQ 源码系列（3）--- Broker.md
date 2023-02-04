---
title: 精读RocketMQ源码系列（0）-开篇
tags:
  - RocketMQ
  - 源码
categories:
  - RocketMQ
date: 2021-07-07 00:00:00
---
## 一、前言

开始之前，我们先看一张来自 RocketMQ 官方文档中的消息数据流图：

![image-20210714230821260.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21e07125a09948168f05b68ee4d94836~tplv-k3u1fbpfcp-watermark.image)

可以简单总结为：

0.  Producer 通过 netty 客户端将消息发送给 Broker
0.  Broker 通过 netty 服务端接收到消息，解析后存储在 CommitLog 中
0.  消息存储到 CommitLog 后会被分发到 ConsumerQueue 和 IndexFile 文件
0.  消费者拉取 ConsumerQueue 的消息完成消费

注意：MessageQueue 和 ConsumerQueue 逻辑上是一一对应的，因为两者使用同一个 `queueId`。

上一篇 [精读 RocketMQ 源码系列（2）--- Producer](https://juejin.cn/post/6983960352283164686) 中我们讲了消息是怎么发送的，也就是第1步。本篇主要围绕 Broker 在接收到消息之后是如何进行存储的这个问题展开，也就是第2、3步展开。

<!-- more -->

思考几个问题：

0.  从接收到存储的完整处理流程是怎样的？
1.  为什么消息存储到了 CommitLog 之后还需要分发到 ConsumerQueue 和 IndexFile 文件中
2.  CommitLog、ConsumerQueue、IndexFile 分别存储了什么消息的什么内容，作用是什么？

## 二、消息接收和存储流程

看了很多资料发现都是直接从 Broker 的存储的时候就开始讲了，没有提消息是如何接收的。为了保持逻辑的连贯性，我还是从消息接收说起。

### 2.1 消息接收

我们知道 RocketMQ 底层是使用 netty 进行通信的，在[精读 RocketMQ 源码系列（1）--- NameServer](https://juejin.cn/post/6983267530181705742) 的 `3.1.1` 中讲 Broker 启动流程的时候，提到过 Broker 启动时还会启动 netty 客户端，源码位置：`BrokerController#start`

```
if (this.remotingServer != null) {
    this.remotingServer.start();
}
```

跟进这个 `remotingServer#start`方法，实际是 `NettyRemotingServer#start`，主要关注下面这段代码：

```
// ServerBootstrap：netty服务启动的辅助类
ServerBootstrap childHandler =
this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
  .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
  .option(ChannelOption.SO_BACKLOG, 1024)
  .option(ChannelOption.SO_REUSEADDR, true)
  .option(ChannelOption.SO_KEEPALIVE, false)
  .childOption(ChannelOption.TCP_NODELAY, true)
  .childOption(ChannelOption.SO_SNDBUF, nettyServerConfig.getServerSocketSndBufSize())
  .childOption(ChannelOption.SO_RCVBUF, nettyServerConfig.getServerSocketRcvBufSize())
  .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
  .childHandler(new ChannelInitializer<SocketChannel>() { // Handler：理解为业务逻辑处理器
@Override
public void initChannel(SocketChannel ch) throws Exception {
ch.pipeline() // pipeline：一组 Handler 的链条
  .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
  .addLast(defaultEventExecutorGroup,
encoder,
new NettyDecoder(),
new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
connectionManageHandler,
serverHandler // ☆☆ 处理发送过来的消息的核心处理逻辑
  );
  }
  });
```

上面这段代码看起来很多，但其实没什么东西。这就是使用 netty 进行网络编程时经常写的一段模板代码，可以对照注释看下（我也是刚接触 netty，不是太懂，计划后期也写一个 netty 的系列）。现在可以这么简单理解这段代码：

> netty 服务端启动时，会进行很多配置，同时会绑定一些 Handler，这些 Handler 就是服务端在接收到客户端请求后需要执行的业务逻辑。这里面，我们要找的关键逻辑在 `serverHandler` 里。

再进到 `serverHandler` 中，发现它是一个内部类：

```
@ChannelHandler.Sharable
class NettyServerHandler extends SimpleChannelInboundHandler<RemotingCommand> {

@Override
protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
processMessageReceived(ctx, msg);
  }
  }
```

继续下去，依次会进到：

0.  `NettyRemotingAbstract#processRequestCommand`
0.  `SendMessageProcessor#asyncProcessRequest`，不论是`AsyncNettyRequestProcessor`做处理还是`NettyRequestProcessor`都会进到这个方法中
0.  `SendMessageProcessor#asyncSendMessage`：这个方法会对消息 `requestHeader`做解析，获取消息的 topic、queueId等信息，并将消息封装到Broker端的消息内置类型 `MessageExtBrokerInner` 中
0.  `DefaultMessageStore#asyncPutMessage`：最后来到了消息存储核心类 `DefaultMessageStore` 这里

以上流程总结为流程图如下：

![msg_receive.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5feadae35af54b50b47e0484a752ec2f~tplv-k3u1fbpfcp-watermark.image)

好了，消息接收的部分就结束了，下一节我们开始关注本篇的重头戏：消息存储！
### 2.2 消息存储

消息存储我这里选的入口是 `DefaultMessageStore#putMessage`，`DefaultMessageStore#asyncPutMessage` 与之类似

接收到消息之后的消息存储流程如下：

![msg_store.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3753a5f794e346a39bc279f308cc3d8a~tplv-k3u1fbpfcp-watermark.image)

上图看着流程很多，但其实并不复杂（像这种时序图都是一边看源码一遍记录的，所以一些细枝末节可能也被记录下来）。对于也想直接看源码的读者是很好的对照图。

这里我以消息为主体，以消息所在位置为观察重心将以上流程总结为以下几步：

0.  `SendMessageProcessor` 接收到的消息是封装在 `RemotingCommand`类中的，然后`SendMessageProcessor`会将 header 的内容解析到`SendMessageRequestHeader`中：

    ```
     SendMessageRequestHeader requestHeader = parseRequestHeader(request);
    ```

0.  `SendMessageProcessor`将消息的 header 和 body 封装到 `MessageExtBrokerInner`

0.  按顺序将消息内容通过 `ByteBuffer`逐一写入 `MappedFile`

0.  `MappedFile` 刷入磁盘，进行持久化

我们知道，消息在 Broker 的文件存储形式是 CommitLog，这里我们刷入磁盘的是 `MappedFile`。这两者之间的关系是一一对应的：

![MappedFile.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/60b6a68e9a2748cd9d249d00fd5c58b1~tplv-k3u1fbpfcp-watermark.image)

现在我们知道了，MappedFile 刷入磁盘之后，即是将消息写到了 CommitLog 文件中。

但它是怎么刷盘的呢？我们回到`CommitLog#asyncPutMessage` 的方法，往下看，看到`CommitLog#submitFlushRequest`

> 在 `putMessage` 方法中会调用 `handleDiskFlush` 方法，在 `asyncPutMessage`中会调用`submitFlushRequest`方法，这两个方法差别不大。我在上面的流程图中写的是 `handleDiskFlush`。这里我们分析 `submitFlushRequest` 即可

这个方法的主逻辑非常重要，所以我这里把它整体贴出来

```
    public CompletableFuture<PutMessageStatus> submitFlushRequest(AppendMessageResult result, MessageExt messageExt) {
        // Synchronization flush   同步刷盘
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            if (messageExt.isWaitStoreMsgOK()) { // 是否需要等待消息存储完成才返回给生产者发送成功的响应
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes(),
                        this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                service.putRequest(request);
                return request.future();
            } else {
                service.wakeup(); // 不需要则唤醒服务并立即返回给生产者发送成功的响应
                return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
            }
        }
        // Asynchronous flush    异步刷盘
        else {
            // 判断是否启动堆外内存 transientStorePoolEnable 默认是 false
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                // 不开启时，采用 MappedByteBuffer 刷盘
                flushCommitLogService.wakeup();
            } else  {
                // 开启时，通过 FileChannel 刷盘
                commitLogService.wakeup();
            }
            return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
        }
    }
```

关键逻辑见注释。这里我们小结一下：

0.  有同步刷盘和异步刷盘两种方式，同步刷盘还是异步刷盘是由 MessageStoreConfig 中的参数 `flushDiskType` 决定的，默认是异步

0.  对于同步刷盘，有两种情况：

    0.  等待消息存储完成再返回生产者发送结果
    0.  直接返回生产者发送结果

0.  对于异步刷盘，有两种方式：

    0.  `transientStorePoolEnable = true` 时，表示开启堆外内存，消息会先放在 `writeBuffer`，然后通过 `FileChannle` 映射到虚拟内存，最后再 `flush` 到磁盘
    0.  `transientStorePoolEnable=false`（默认），消息追加时，直接存入 MappedByteBuffer(pageCache) 中，然后定时 flush

完成消息到 CommitLog 刷盘之后，消息就算是已经完成持久化了。

贴一张 CommitLog 中一条消息的组成结构图：

![image-20210721005237968.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a9c220119d14f8e988f92b846571a77~tplv-k3u1fbpfcp-watermark.image)

但为了消费者更好地消费消息，还需要将 CommitLog 分发到 ConsumeQueue 中；为了实现根据某些关键字查询的功能那个，又需要将 CommitLog 分发到 IndexFile 中。
### 2.3 indexFile 和 ConsumeQueue 是如何更新的

启动 broker 的时候会启动 `DefaultMessageStore`，在其`start`方法中又启动了存储相关的服务，其中就包括了将 CommitLog 分发到 indexFile 和 comsumeQueue 的服务

```
//设置CommitLog内存中最大偏移量
this.reputMessageService.setReputFromOffset(maxPhysicalPosInLogicQueue);
//启动
this.reputMessageService.start();
```

看看这个方法的执行时序图：


![reput_msg.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35e44d5c824c42dbb3043d9aae8cc154~tplv-k3u1fbpfcp-watermark.image)

接着会分别调用不同的实现类，去分别构建 ConsumeQueue 和 Index

**构建 ConsumeQueue**


![dispatch_consume.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/322c920b1f89473194b38130bf9c24ab~tplv-k3u1fbpfcp-watermark.image)

**构建 IndexFile**


![dispatch_index.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1a6b199769348b095b9eb48505c01b9~tplv-k3u1fbpfcp-watermark.image)

最后我们来分别看一下这两个文件的结构。

#### ConsumeQueue

ConsumeQueue 在逻辑上与生产者发送时 MessageQueue 是一一对应的。如果有4个消息写队列，那么 CommitLog 也会被分发到4个相应的 ConsumeQueue（同一个consumeQueue有多个文件，因为单独一个文件的限制是30万*20B）。

ConsumeQueue 由30万个固定大小为20byte的数据块组成，数据块的内容如下：


![ConsumeQueue.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f46f608797ca4500b3fd59cdaa02a058~tplv-k3u1fbpfcp-watermark.image)

`msgPhyOffset`: 消息在 CommitLog 文件中的起始位置

`msgSize`: 消息在文件中占的长度

`msgTagCode`: 消息 tag，用于标识业务相关

**如何构建：**

0.  broker启动时会启动ReputMessageService任务，1ms执行1次
0.  ReputMessageService记录分发的reputFromOffset，每次将对应的消息追加到 consumeQueue文件中，完成一条消息的分发
0.  设置reputFromOffset = reputFromOffset + 读取到的消息.size，等待下次任务继续构建下一条消息

**如何查询：**

消费者消费时，只要知道自己要消费的是第几条消息（称之为消费位点），就可以通过消费位点对30万取余的方式，定位到指定的 consumeQueue文件的指定数据块。如果超过了30万，那就是下一个文件的数据块。例如，我需要读取第31万的消息，我可以计算出这条消息的索引内容在第2个consumequeue的第1万个数据块。然后读取这个数据块的内容，再去commitLog获取真正的消息。

#### IndexFile

indexFile 也是一个索引文件，不过它的定位是提供根据msgId或生产者指定的消息key作为索引key。整个indexFile的文件物理存储结构如下：


![index.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b95cb0ec28dc419a8f30470b2ebf0155~tplv-k3u1fbpfcp-watermark.image)

0.  Header：固定大小 40B

    0.  beginTimestamp: 该indexFile对应的第一条消息的存储时间
    0.  endTimestamp: 该indexFile对应的最后一条消息的存储时间
    0.  beginPhyOffset: 该indexFile对应的第一条消息在CommitLog中的偏移量
    0.  endPhyOffset: 该indexFile对应的最后一条消息在CommitLog中的偏移量
    0.  hashSlotCount: 已填充值的slot数量
    0.  indexCount: 该indexFile包含的索引个数

0.  Slot: 500万个，每个4B，slot 中存储一个int值，保存当前slot下最新的index的序号（链表头），可以算出来该Index的位置

0.  Index：结构如图灰色部分，共2000万个，每个20B

    0.  Key Hash: 索引 key 的hash值
    0.  CommitLog Offset: 索引对应的消息在 CommitLog 的偏移量
    0.  Timestamp：记录的是该条消息与当前索引文件第一条消息的存储时间的时间差，并不是绝对时间
    0.  next index offset: 当前slot下，当前index的前一个index的slotValue(可以简单理解是一个指向前一个Index的指针)，这也就是为什么 slot 总是存最新的index，因为最新的index是链表头，持有前一个index的序号。

其实讲到这里，应该很容易想到，逻辑上，indexFile 的构造很像 Java 中的 HashMap

**如何构建：**

0.  获取到消息的msgId，进行hash值计算
0.  对500万取余获得对应的slot号n
0.  根据40+(n-1)*4算出该slot文件的位置，并读取slotValue
0.  追加写入一条index数据，next index offset写第3步中获取到的slotValue，即相同slot下前一个Index的序号
0.  更新当前slot值为新插入的index的序号
0.  更新Header中的endTimestamp、endPhyOffset、indexCount、hashSlotCount（可能不更新）

**如何查询：**

查询需要传入的参数有：key、beginTimestamp、endTimestamp

为什么要传时间呢？

因为 indexFile 文件有多个，而key有可能在不同的indexFile中重复，所以要先根据时间范围确定唯一的indexFile。而indexFile的文件命名就是一个起始时间戳，同时Header中有截止时间戳，根据这些信息就可以确定indexFile。

确定了indexFile之后是怎么查询的？

0.  根据key计算hash值，hash值对500万取余得出slot的序号n
0.  根据公式：40+(n-1)*4 即可获得slot在文件中的位置
0.  读取slot中的值，也即最新的index在文件中的序号s
0.  根据`40+500万*4+(s-1)*20`可以得到最新的index在文件中的位置
0.  读取该Index，将该index的hash值、timestamp和传入参数作比对
0.  不符合则找到下一个index，找到后得到了偏移量，就可以去commitLog中拿到具体的消息

为什么比对的时候也要比对时间范围？

因为key可能会重复，producer在消息生产时可以指定消息的key，这个key显然无法保证唯一性。而自动生成的msgId也不能保证唯一。

> msgId生成规则: 前机器IP+进程号+MessageClientIDSetter.class.getClassLoader()的hashCode值+消息生产时间与broker启动时间的差值+broker启动后从0开始单调自增的int值，前面三项很明显可能重复，后面两项一个是时间差，一个是重启归零，也可能重复

## 三、小结

到这里我们小结下，本篇文章讲了哪些东西：

1.  消息接收：通过 nettey server 接收请求，判断是消息发送请求，则会调用相应的消息接收方法
1.  消息存储：消息先会存储 CommitLog 中，然后会通过同步/异步的方式刷盘到 ConsumerQueue 和 IndexFile 中

接着回答下文章开头的第二个问题：为什么消息还要被分发到 ConsumeQueue 和 IndexFile？

1.  ConsumeQueue 可以认为是逻辑分区，类似于 Kafka 中的 partition，通过将 CommitLog 分为多个 ConsumeQueue，使得同一个 Topic 的消息可以被多个消费者同时消费，增加吞吐量。同时也能实现单个 ConsumeQueue 的消息的顺序性，满足一些业务场景。
1.  分发到 IndexFile，是因为在客户端（生产者和消费者）和admin接口提供了根据 key 查询消息的实现。为了方便用户查询具体某条消息。

## 四、参考文档

<https://blog.csdn.net/wb_snail/article/details/106236898>

<https://blog.csdn.net/wb_snail/article/details/106234406>

<https://blog.csdn.net/meilong_whpu/article/details/76919267>

<https://github.com/DillonDong/notes/blob/master/RocketMQ/RocketMQ-03.md>

**最后**
-   文章若有错误，欢迎评论留言指出，也欢迎转载，转载请注明出处