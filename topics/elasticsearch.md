# Elasticsearch

搜索全部课件整理：
https://www.yuque.com/xiaoliujiangyuanma/cs1x7i/vslxb4zsg03capni?singleDoc# 《搜索课件》 密码：hots

通过以下几点来分析：
1. ES的数据模型 
2. ES module之间的调用
3. ES写入流程 
4. 根据DDIA-5 分区分析 
5. 根据DDIA-6 复制分析

## 1. ES的数据模型

### 1.1 ES文档data-replication对数据模型问题的解释

Elasticsearch的数据复制模型基于leader/follower模型，在PacificA论文中得到了很好的描述。
该模型基于复制组中有一个副本作为主分片。其他副本被称为副本分片。主分片是所有索引操作的主要入口。它负责验证操作并确保其正确性。一旦索引操作被主分片接受，主分片还负责将操作复制到其他副本

本节的目的是对Elasticsearch的复制模型进行高层次的概述，并讨论它对读写操作之间各种交互的影响

#### 写入模型



Bully算法Leader选举的基本算法之一。它假定所有节点都有一个唯一的ID，使用该ID对节点进行排序。任何时候的当前Leader都是参与集群的最高ID节点。

该算法的优点是易于实现。但是，当拥有最大ID的节点处于不稳定状态的场景下会有问题。例如，Master负载过重而假死，集群拥有第二大ID的节点被选为新主，这时原来的Master恢复，再次被选为新主，然后又假死……

ZenDiscovery的选主过程如下：每个节点计算最小的已知节点ID，该节点为临时Master。向该节点发送领导投票。
如果一个节点收到足够多的票数，并且该节点也为自己投票，那么它将扮演领导者的角色，开始发布集群状态。所有节点都会参与选举，并参与投票，但是，只有有资格成为Master的节点（node.master为true）的投票才有效．获得多少选票可以赢得选举胜利，就是所谓的法定人数。
在 ES 中，法定大小是一个可配置的参数。配置项：discovery.zen.minimum_master_nodes。为了避免脑裂，最小值应该是有Master资格的节点数n/2+1。









在看模块的功能之前，先看下ES是如何做模块插件化和内部是如何路由调用module的

## 模块插件化
以NetworkModule为例，所有的Network模块会extends Plugin implements NetworkPlugin，在Node.java初始化的时候，把所有的NetworkPlugin实现传入到NetworkModule的构造方法
```Java
final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),
                threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,
```
然后以key-value的形式保存到map中

```Java
for (NetworkPlugin plugin : plugins) {
    // 获取插件中定义的HttpTransport实现
    Map<String, Supplier<HttpServerTransport>> httpTransportFactory = plugin.getHttpTransports(settings, threadPool, bigArrays,
        pageCacheRecycler, circuitBreakerService, xContentRegistry, networkService, dispatcher, clusterSettings);
    if (transportClient == false) {
        for (Map.Entry<String, Supplier<HttpServerTransport>> entry : httpTransportFactory.entrySet()) {
            registerHttpTransport(entry.getKey(), entry.getValue());
        }
    }
```

使用的时候，根据配置文件的指定对应的实现或者使用默认值

```Java
/**
 * 返回注册的HTTP传输模块
 */
public Supplier<HttpServerTransport> getHttpServerTransportSupplier() {
    final String name;
    // 配置中是否指定了实现
    if (HTTP_TYPE_SETTING.exists(settings)) {
        // 可以配置 nio....
        name = HTTP_TYPE_SETTING.get(settings);
    } else {
        // 默认netty4
        name = HTTP_DEFAULT_TYPE_SETTING.get(settings);
    }
```

## module的调用

```Java
void inboundMessage(TcpChannel channel, InboundMessage message){
    final long startTime = threadPool.relativeTimeInMillis();
    channel.getChannelStats().markAccessed(startTime);
    TransportLogger.logInboundMessage(channel, message);

    if (message.isPing()) {
        keepAlive.receiveKeepAlive(channel);
    } else {
        messageReceived(channel, message, startTime); // 接收到请求
    }
    
    
private void messageReceived(...) {
   。。。。。
   if (header.isRequest()) {
      handleRequest(channel, header, message);
   }
}

private <T extends TransportRequest> 
    void handleRequest(TcpChannel channel, Header header, InboundMessage message){
    final String action = header.getActionName();
    ....
    // 根据action的名字取拿对应的Handlers
    final RequestHandlerRegistry<T> reg = requestHandlers.getHandler(action);
```

这些Handlers主要是两个地方注册:
1. ActionModule->setupActions(..)方法 会向guice IOC框架提供一组需要实例化的Transport*Action的类型, 该类型比较特殊,都是继承自HandledTransportAction类型，该类型的构造方法会负责将本handler注册到这里
2. 代码层使用TransportService->registerRequestHandler(...)方法强制向requestHandlers map中加入请求处理器

```Java
public MembershipAction(...) {
    this.transportService = transportService;
    this.listener = listener;

    transportService.registerRequestHandler(DISCOVERY_JOIN_ACTION_NAME,
       ThreadPool.Names.GENERIC, JoinRequest::new, new JoinRequestRequestHandler());
```
## 选举流程

1. 加入master节点的流程是什么？
2. 如果本节点被选为master，接下去做什么
3. 普通节点如何监控master健康状态
4. master如何监控普通节点健康状态
5. ES如何避免脑裂？它有哪些举措？

ES 7.0之前默认用的是内置的ZenDiscovery，在Node.java中启动
```Java
// start before cluster service so that it can set initial state on ClusterApplierService
discovery.start(); 
discovery.startInitialJoin(); // 选主
```

ZenDiscovery核心属性:
```Text
TransportService: 通信服务
ZenPing：UnicastZenPing，Ping工具
MasterFaultDetection: Node监控Master节点状态服务
JoinThreadControl: 加入集群线程控制器
NodeJoinController: 被选中节点，控制普通节点连接的逻辑
ClusterApplier: 集群状态“应用”服务
ThreadPool: es封装的线程池，以后课程分析

MasterService: 主节点服务
publishClusterState: 发布集群状态服务
NodesFaultDetection: Master监控普通节点状态服务
MembershipAction: 处理成员请求事件
PendingClusterStatesQueue: 集群状态Pending队列
AtomicReference<ClusterState> committedState: 最后一次提交的状态
ElectMasterService: 选举主节点服务
```

discovery.start()调用，只是初始化一些默认值
```Java
protected void doStart() {
    // 获取表示本地节点信息的node对象
    DiscoveryNode localNode = transportService.getLocalNode();
    assert localNode != null;
    synchronized (stateMutex) {
        // set initial state
        assert committedState.get() == null;
        assert localNode != null;

        // 集群状态构造器
        ClusterState.Builder builder = ClusterState.builder(clusterName);
        ClusterState initialState = builder
            .blocks(ClusterBlocks.builder()
                .addGlobalBlock(STATE_NOT_RECOVERED_BLOCK) // 集群未恢复
                .addGlobalBlock(noMasterBlockService.getNoMasterBlock())) // 集群无主状态
            .nodes(DiscoveryNodes.builder().add(localNode).localNodeId(localNode.getId()))
            .build();

        // 将初始的集群信息写给committedState。因为committedState要始终保持最新的集群状态
        committedState.set(initialState);

        // clusterApplier内部封装了应用集群状态的逻辑
        // 内部注册了一些Listener，Listener收到newState之后，做什么事情，由它决定。
        clusterApplier.setInitialState(initialState);

        // nodesFD: 普通node节点join到master之后，启动，会定时1s向master发起ping，用于监控master存活
        // 这里设置localNode，是因为ping的request中包含了local信息
        nodesFD.setLocalNode(localNode);

        // joinThreadControl：该Control用于当前节点控制joinThread，避免本地同时有多个joinThread去工作。确保
        // 在需要joinThread工作的时候，仅仅只有一个该线程。内部封装了启动 joinThread 的逻辑。
        // start()仅仅是设置running开关为true，并不会启动线程。
        joinThreadControl.start();
    }
    zenPing.start();
}
```

discovery.startInitialJoin()则是启动一个异步任务，放到generic线程池中

找不到则不停地循环，如果选举失败，node会开启一个新的joinThread去顶替当前线程的工作
```Java
 while (masterNode == null && joinThreadControl.joinThreadActive(currentThread)) {
    masterNode = findMaster();
}
```

选举失败的流程在innerJoinCluster方法的最后, 将joinThreadControl->currentJoinThread字段设置为null, 
让zenDiscovery->executePool重新提交runnable任务，会再次开启一个新线程去做join的事情. 最终形成了一个闭环
```Java
  synchronized (stateMutex) {
    if (success) {
        ......
    } else {
        // failed to join. Try again...
        joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
    }
```

findMaster会向discovery.zen.ping.unicast.hosts配置项的节点发送ping请求，并且线程等待对端响应，返回的是PingResponse对象，包含对端节点和集群的信息
```Java
// 内部还封装了ping失败后，重试等等逻辑...
List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();

class PingResponse implements Writeable {
    // 返回的PingResponse对象
    private final long id;
    // 对方节点的集群名
    private final ClusterName clusterName;
    // 对方节点的基本信息
    private final DiscoveryNode node;
    // 对方所在master的节点的基本信息
    private final DiscoveryNode master;
    // 集群状态版本号
    private final long clusterStateVersion;
    
// 本地节点加入到PingResponse
fullPingResponses.add(new ZenPing.PingResponse(localNode, null, this.clusterState()));

// 默认情况下，全部节点都是默认投票权，如果配置ignore_non_master_pings=true的配置的话，则会过滤掉非master权限
final List<ZenPing.PingResponse> pingResponses = filterPingResponses(fullPingResponses, masterElectionIgnoreNonMasters, logger);

// masterCandidates: 该列表保存所有可能成为master节点的信息
List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
for (ZenPing.PingResponse pingResponse : pingResponses) {
    if (pingResponse.node().isMasterNode()) {
        masterCandidates.add(new ElectMasterService.MasterCandidate(pingResponse.node(), pingResponse.getClusterStateVersion()));
    }
}

// CASE1: 当前集群无主
if (activeMasters.isEmpty()) {

    // 判断是否有足够数量的候选者
    if (electMaster.hasEnoughCandidates(masterCandidates)) {

        // 从候选者列表中，选举winner：对masterCandidates进行sort，先按照集群版本大小排序，大的靠前，如果
        // 集群版本号一致，则再按照 masterCandidate->node->id进行排序，小的靠前..
        // 排序完之后，选择 list.get(0)为胜出者
        final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
        logger.trace("candidate {} won election", winner);
        return winner.getNode();
    } else {
        // if we don't have enough master nodes, we bail, because there are not enough master to elect from
        logger.warn("not enough master nodes discovered during pinging (found [{}], but needed [{}]), pinging again",
                    masterCandidates, electMaster.minimumMasterNodes());
        return null;
    }
} else {
    // CASE2: 当前集群有主
    assert !activeMasters.contains(localNode) :
        "local node should never be elected as master when other nodes indicate an active master";
    // lets tie break between discovered nodes
    return electMaster.tieBreakActiveMasters(activeMasters);
}
```

