---
title: 谈谈 RocketMQ NameServer 的设计与实现
---

NameServer 作为消息中间件 RocketMQ 的核心组件之一，承担着路由注册中心的作用。本文试图以结合源码的方式解答三个问题：一是作为路由注册中心，有哪些路由信息注册到了 NameServer？二是 NameServer 通常以集群部署，且集群中的各个节点互相不通信，当 NameServer 集群中各节点路由信息不一致时，RocketMQ 如何保证可用性？三是当 Broker 不可用后，NameServer 并不会立即将变更后的注册信息推送至 Client（Producer/Consumer），此时 RocketMQ 如何保证 Client 正常发送/消费消息？

### NameServer 启动流程

本文首先讨论 NameServer 启动流程，重点关注两部分：路由信息维护和网络通信（包括心跳），涉及到的核心类如下图所示。NameServerStartup 是 NameServer 的启动类，负责解析配置文件、加载运行时参数信息和初始化并启动 NameServerController。NameServerController（下文简称 controller ） 是 NameServer 的核心控制器，其通过 RouteInfoManager 管理路由信息，通过 RemotingServer 与 RocketMQ 其他组件（Broker、Producer 和 Consumer）通信。

![](/img/2019-06-18-NameServer1.jpg)

NameServer 的配置参数包括 namesrvConfig 和 nettyServerConfig：namesrvConfig 为 NameServer 业务参数，如 RocketMQ 主目录路径、KV 配置属性持久化路径、是否支持顺序消息等；nettyServerConfig 为 NameServer 网络参数，如监听端口、IO 线程池线程个数、业务线程池线程个数、是否开启 epoll 等。在省略加载和解析配置文件的逻辑后，NameServer 的启动流程精简如下。

```java
// org.apache.rocketmq.namesrv.NamesrvStartup#main0
public static NamesrvController main0(String[] args) {
	try {
		...
		final NamesrvConfig namesrvConfig = new NamesrvConfig();
		final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        // 加载配置信息至 namesrvConfig 和 nettyServerConfig
        ...
		final NamesrvController controller = 
            new NamesrvController(namesrvConfig, nettyServerConfig);
		...
        boolean initResult = controller.initialize();
        if (!initResult) {
        	controller.shutdown();
            System.exit(-3);
        }
		Runtime.getRuntime().addShutdownHook(
            new ShutdownHookThread(log, new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                controller.shutdown();
                return null;
            }
		}));
        controller.start();
        ...
        return controller;
	} catch (Throwable e) {
		e.printStackTrace();
		System.exit(-1);
	}

	return null;
}
```

基于上述流程，NameServer 的启动流程分为 controller 的初始化（initialize）和启动（start）两阶段。初始化包括：初始化 KV 配置信息、初始化网络模块（基于 Netty）、初始化线程池、注册网络请求处理器（DefaultRequestProcessor）和启动两个定时任务（定时扫描 Broker 和定时打印 KV 配置信息）。controller 初始化逻辑如下。

```java
// org.apache.rocketmq.namesrv.NamesrvController#initialize
public boolean initialize() {
	this.kvConfigManager.load();
	this.remotingServer = 
        new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
	this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), 
                                         new ThreadFactoryImpl("RemotingExecutorThread_"));
	this.registerProcessor();
	this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
		@Override
		public void run() {
			NamesrvController.this.routeInfoManager.scanNotActiveBroker();
		}
	}, 5, 10, TimeUnit.SECONDS);
	this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
		@Override
		public void run() {
        	NamesrvController.this.kvConfigManager.printAllPeriodically();
        }
    }, 1, 10, TimeUnit.MINUTES);
}
```

完成 controller 初始化后，注册钩子函数，在 JVM 关闭时执行 controller 的清理工作。接着启动 controller。核心逻辑精简如下。

```java
// org.apache.rocketmq.namesrv.NamesrvController#start
public void start() throws Exception {
	this.remotingServer.start();
}

// org.apache.rocketmq.remoting.netty.NettyRemotingServer#start
@Override
public void start() {
    this.defaultEventExecutorGroup = new DefaultEventExecutorGroup(
        nettyServerConfig.getServerWorkerThreads(),
        new ThreadFactory() {...});

    ServerBootstrap childHandler =
        this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
            .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
            .option(...)
            .childOption(...)
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline()
                    	.addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME,
                        	new HandshakeHandler(TlsSystemConfig.tlsMode))
                        .addLast(defaultEventExecutorGroup,
                        	new NettyEncoder(),
                            new NettyDecoder(),
                            new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                            new NettyConnectManageHandler(),
                            new NettyServerHandler()
                         );
                }
            });
	...
    try {
        ChannelFuture sync = this.serverBootstrap.bind().sync();
        InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
        this.port = addr.getPort();
    } catch (InterruptedException e1) {
    	...
    }

    if (this.channelEventListener != null) {
        this.nettyEventExecutor.start();
    }

    this.timer.scheduleAtFixedRate(new TimerTask() {
        @Override
        public void run() {
            try {
                NettyRemotingServer.this.scanResponseTable();
            } catch (Throwable e) {
                log.error("scanResponseTable exception", e);
            }
        }
    }, 1000 * 3, 1000);
}
```

启动流程很常规，NameServer 开始监听端口，网络请求经过一系列 ChannelHandler 交由 NettyServerHandler：NettyServerHandler 是 NettyRemotingServer 的内部类，且 NettyRemotingServer 继承自NettyRemotingAbstract，具体的业务逻辑在 NettyRemotingAbstract.processMessageReceived 方法实现。

至此，NameServer 启动所需工作已经完成。

### NameServer 路由信息

NameServer 作为路由注册中心，其核心作用是为 Client 提供消息发送/消费的路由信息。Master Broker 节点和所有 NameServer 建立长连接，每个 NameServer 节点拥有所有 Topic 对应 Queue 以及 Broker 的映射关系，具体由 RouteInfoManager 负责管理，核心数据结构精简如下。

```java
// org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager
public class RouteInfoManager {
    ...
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    ...
}
```

* topicQueueTable 维护了 Topic 和其对应消息队列的映射关系，QueueData 记录了一条队列的元信息：所在 Broker、读队列数量、写队列数量等。
* brokerAddrTable 维护了 Broker Name 和 Broker 元信息的映射关系，Broker 通常以 Master-Slave 架构部署，BrokerData 记录了同一个 Broker Name 下所有节点的地址信息。
* clusterAddrTable 维护了 Broker 的集群信息。
* brokerLiveTable 维护了 Broker 的存活信息。NameServer 在收到来自 Broker 的心跳消息后，更新 BrokerLiveInfo 中的 lastUpdateTimestamp，如果 NameServer 长时间未收到 Broker 的心跳信息，NameServer 就会将其移除。
* filterServerTable 用于消息过滤，本文不做详细介绍。

![](/img/2019-06-18-NameServer2.jpg)

在给定 Topic 的情况下，Client 根据负载均衡策略选择合适的消息队列，进一步获取到对应的 Broker 地址信息，整体流程如上。至此，本文第一个问题已经得到解答。

#### NameServer 请求处理

NettyRequestProcessor 定义了 RocketMQ 处理网络请求的接口，DefaultRequestProcessor 实现了该接口并负责处理 NameServer 收到的网络请求。NettyRequestProcessor 定义如下。

```java
// org.apache.rocketmq.remoting.netty.NettyRequestProcessor
public interface NettyRequestProcessor {
    RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request)
        throws Exception;

    boolean rejectRequest();
}
```

RemotingCommand 包含请求编码（code），针对不同的请求编码执行对应的请求处理逻辑。请求编码定义在 RequestCode 常量类中，本节以 REGISTER_BROKER（Broker 注册 & 心跳）为例，结合 Broker 的启动流程阐述请求从被 Broker 发出到被 NameServer 处理的流程。

Broker 启动类 BrokerStartup 在初始化核心控制器 BrokerController 阶段注册定时任务，定时发送 HTTP 请求获取 NameServer 地址列表并存储于 remotingClient。此处考虑一个问题：在弃用 ZooKeeper 后，RocketMQ 不存在注册中心供 NameServer 注册，那么 NameServer 地址列表是如何维护的？在获取过时的地址列表后，RocketMQ 如何持续保证可用性？BrokerController 定时获取地址列表核心逻辑精简如下。

```java
// org.apache.rocketmq.broker.BrokerController#initialize
public boolean initialize() throws CloneNotSupportedException {
    ...
    if (this.brokerConfig.getNamesrvAddr() != null) {
        this.brokerOuterAPI.updateNameServerAddressList(this.brokerConfig.getNamesrvAddr());
        ...
    } else if (this.brokerConfig.isFetchNamesrvAddrByAddressServer()) {
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                try {
                    BrokerController.this.brokerOuterAPI.fetchNameServerAddr();
                } catch (Throwable e) {
                    log.error("ScheduledTask fetchNameServerAddr exception", e);
                }
            }
        }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
    }
    ...    
}

// org.apache.rocketmq.broker.out.BrokerOuterAPI#fetchNameServerAddr
public String fetchNameServerAddr() {
    try {
        String addrs = this.topAddressing.fetchNSAddr();
        if (addrs != null) {
            if (!addrs.equals(this.nameSrvAddr)) {
                ...
                this.updateNameServerAddressList(addrs);
                this.nameSrvAddr = addrs;
                return nameSrvAddr;
            }
        }
    } catch (Exception e) {
        log.error("fetchNameServerAddr Exception", e);
    }
    return nameSrvAddr;
}

// org.apache.rocketmq.common.namesrv.TopAddressing#fetchNSAddr(boolean, long)
public final String fetchNSAddr(boolean verbose, long timeoutMills) {
    ...
    HttpTinyClient.HttpResult result = HttpTinyClient.httpGet(url, null, null, "UTF-8", timeoutMills);
    ...   
}
```

Broker 在获取到 NameServer 地址列表后，针对每个地址开启一个线程，将自身信息同步（默认）注册至 NameServer。Broker 利用 CountDownLatch 等待所有线程注册工作完成后，继续执行后续的工作。核心流程精简如下。

```java
// org.apache.rocketmq.broker.BrokerController#start
public void start() throws Exception {
 this.registerBrokerAll(true, false, true);
}

// org.apache.rocketmq.broker.BrokerController#registerBrokerAll
public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway,
                                           boolean forceRegister) {
    ...
    if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId(),
        this.brokerConfig.getRegisterBrokerTimeoutMills())) {
        doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
    }
}

// org.apache.rocketmq.broker.BrokerController#doRegisterBrokerAll
private void doRegisterBrokerAll(boolean checkOrderConfig, boolean oneway,
    TopicConfigSerializeWrapper topicConfigWrapper) {
    List<RegisterBrokerResult> registerBrokerResultList = this.brokerOuterAPI.registerBrokerAll(
        this.brokerConfig.getBrokerClusterName(),
        this.getBrokerAddr(),
        this.brokerConfig.getBrokerName(),
        this.brokerConfig.getBrokerId(),
        this.getHAServerAddr(),
        topicConfigWrapper,
        this.filterServerManager.buildNewFilterServerList(),
        oneway,
        this.brokerConfig.getRegisterBrokerTimeoutMills(),
        this.brokerConfig.isCompressedRegister());
    ...
}

// org.apache.rocketmq.broker.out.BrokerOuterAPI#registerBrokerAll
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
    final List<RegisterBrokerResult> registerBrokerResultList = Lists.newArrayList();
    List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
    if (nameServerAddressList != null && nameServerAddressList.size() > 0) {
        final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
        ...
        RegisterBrokerBody requestBody = new RegisterBrokerBody();
        ...
        final byte[] body = requestBody.encode(compressed);
        ...
        final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
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

// org.apache.rocketmq.broker.out.BrokerOuterAPI#registerBroker
private RegisterBrokerResult registerBroker(
    final String namesrvAddr,
    final boolean oneway,
    final int timeoutMills,
    final RegisterBrokerRequestHeader requestHeader,
    final byte[] body
) throws ... {
    RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.REGISTER_BROKER, requestHeader);
    request.setBody(body);
    ...
    RemotingCommand response = this.remotingClient.invokeSync(namesrvAddr, request, timeoutMills);
    switch (response.getCode()) {
        case ResponseCode.SUCCESS: { ... }
        default: break;
    }
    throw new MQBrokerException(response.getCode(), response.getRemark());
}
```

至此，Broker 发送 REGISTER_BROKER 请求所需工作已经完成。NameServer 节点在收到 REGISTER_BROKER 请求后，交由 DefaultRequestProcessor 处理，请求处理入口精简如下，每收到 REGISTER_BROKER 请求，brokerLiveTable 中对应 Broker 的活跃状态，以及 topicQueueTable、brokerAddrTable 和 filterServerTable 路由信息将会被同步更新，registerBroker 详细逻辑不再赘述。NameServer 引入读写锁 ReentrantReadWriteLock 控制并发，允许并发读，满足消息发送时的高并发需求。

```java
// org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor#processRequest
@Override
public RemotingCommand processRequest(ChannelHandlerContext ctx,
    RemotingCommand request) throws RemotingCommandException {
    switch (request.getCode()) {
        ...
        case RequestCode.REGISTER_BROKER:
            return this.registerBroker(ctx, request);
        default: break;
    }
    return null;
}
```

### NameServer 一致性

讨论本文提出的第二个问题，当 NameServer 路由信息不一致时，RocketMQ 如何保证可用性。什么时候路由信息会出现不一致？举一个例子，如下图所示，NameServer 集群和 Broker 集群均包含 2 个节点，假设此时发生网络分区（Network Partition），各个 NameServer 仅维护了分区内的 Broker 路由信息，Client 也仅能根据分区内 NameServer 提供的路由信息发送/消费消息（暂不考虑 Client 端路由信息缓存）。

![](/img/2019-06-18-NameServer3.jpg)

进一步思考，在什么场景下需要保证强一致性（Strong Consistency）？结合 CAP，P 一定会发生，CA 二选一，那么 NameServer 是否需要提供强一致性呢？结合上图，此时分区 a 中的 Client 只能将消息投递到分区内的 Broker，也只能消费存储在分区内 Broker 的消息。如果在网络分区前已经有消息被投递到分区 b，那么当网络分区恢复后，分区 b 中的 Broker 重新将路由信息注册至分区 a 中的 NameServer，Client 可以继续消费消息。

如果考虑顺序消息，此时条件更加苛刻：为保证消息的顺序性，需要满足从 Provider 到 Broker 到 Consumer 都是单点单线程。所谓单点，自然满足强一致性。因此，在这种场景下可用性是无法得到保证的。

至此，本文提出的第二个问题得到解答：NameServer 在设计上做了 trade-off，在乱序消息的场景下，NameServer 是一个 CP 系统，并不需要强一致性的保证。

### Broker 失效

Broker 启动后每间隔 30s 向 NameServer 集群所有节点广播发送心跳消息。NameServer 启动后每间隔 10s 扫描自身维护的 Broker 活跃信息。如果超过 120s 未收到 Broker 的心跳消息，NameServer 将对应的 Broker 从路由信息中移除。上述逻辑精简后如下。

```java
// org.apache.rocketmq.broker.BrokerController#start
public void start() throws Exception {
    ...
    this.registerBrokerAll(true, false);
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                BrokerController.this.registerBrokerAll(true, false);
            } catch (Throwable e) {
                log.error("registerBrokerAll Exception", e);
            }
        }
    }, 1000 * 10, 1000 * 30, TimeUnit.MILLISECONDS);
    ...
}

// org.apache.rocketmq.namesrv.NamesrvController#initialize
public boolean initialize() {
    ...
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 5, 10, TimeUnit.SECONDS);
    ...
    return true;
}

// org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#BROKER_CHANNEL_EXPIRED_TIME
private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;

// org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#scanNotActiveBroker
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```

当某个 Broker 节点不可用后，NameServer 可能需要经过 120s 才能感知，并且 NameServer 并不会主动将更新后的路由信息推送至 Client。如果 Client 在这段时间内发送/消费消息，存在获取到过时路由信息的可能性，即向一个不可用的 Broker 发送/消费消息。本节以 Provider 发送同步消息为例，讨论在获取到过期的 Broker 路由信息的情况下，RocketMQ 如何保证消息发送的可用性？

在消息发送阶段，Provider 基于自身缓存的路由信息结合负载均衡策略选择合适的消息队列作为投递目标，当且仅当缓存为空时，才会向 NameServer 实时同步获取路由信息。同时，Provider 会启动线程定时获取最新的路由信息并更新缓存。上述流程核心逻辑精简如下。

```java
// org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws ... {
    ...
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) { ... }
    ...
}

// org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#tryToFindTopicPublishInfo
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }
 ...
}

// org.apache.rocketmq.client.impl.factory.MQClientInstance#startScheduledTask
private void startScheduledTask() {
    ...
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
        @Override
        public void run() {
            try {
                MQClientInstance.this.updateTopicRouteInfoFromNameServer();
            } catch (Exception e) {
                log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
            }
        }
    }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);
    ...
}
```

当向一个不可用的 Broker 投递消息时，Provider 最终无法收到来自 Broker 的 ACK，RocketMQ 通过两种机制保证消息发送的高可用：重试和无效 Broker 规避。关于重试，当同步消息发送失败后，Provider 默认的重试次数取决于 retryTimesWhenSendFailed，默认等于 2。关于无效 Broker 规避，RocketMQ 通过两个层面实现：在同一个消息重试发送过程中，lastBrokerName 记录了上一次投递失败的 Broker Name，以此避免重试时投递到无效的 Broker；引入故障延迟机制，当 sendLatencyFaultEnable 为开启状态，基于 LatencyFaultToleranceImpl 维护的 faultItemTable 决定 Broker 是否可被加入至轮询队列。上述逻辑精简如下。

```java
// org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws ... {
    ...
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null;
        SendResult sendResult = null;
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                ...
                try {
                    ...
                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout);
                    ...
                } catch (InterruptedException e) {
                    ...
                }
            } else {
                break;
            }
        }
        ...
}

// org.apache.rocketmq.client.latency.MQFaultStrategy#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        try {
            int index = tpInfo.getSendWhichQueue().getAndIncrement();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                        return mq;
                }
            }
            ...
        } catch (Exception e) {
            log.error("Error occurred when selecting message queue", e);
        }

        return tpInfo.selectOneMessageQueue();
    }

    return tpInfo.selectOneMessageQueue(lastBrokerName);
}

// org.apache.rocketmq.client.impl.producer.TopicPublishInfo#selectOneMessageQueue
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    if (lastBrokerName == null) {
        return selectOneMessageQueue();
    } else {
        int index = this.sendWhichQueue.getAndIncrement();
        for (int i = 0; i < this.messageQueueList.size(); i++) {
            int pos = Math.abs(index++) % this.messageQueueList.size();
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

至此，本文提出的第三个问题也得到解答：对于 Broker 无效的场景，RocketMQ 牺牲了 C，选择了 AP，即通过 Client 重试保证可用性，由此产生的重复消息问题，通过 Client 幂等处理逻辑来规避。

### 参考

[1] https://github.com/apache/rocketmq

