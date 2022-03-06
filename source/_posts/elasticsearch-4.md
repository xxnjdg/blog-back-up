---
title: elasticsearch阅读笔记4
date: 2021-07-08 09:42:05
categories:
- spring cloud alibaba
tags:
- elasticsearch
---


Elasticsearch源码解析与优化实战 

第15章 Transport模块分析

elasticsearch版本 7.13.2

Node 构造方法先执行，后执行start方法

```java
public class Node implements Closeable {
    protected Node(final Environment initialEnvironment,
                   Collection<Class<? extends Plugin>> classpathPlugins, boolean forbidPrivateIndexSettings) {

        //构建节点级别的 Settings
        final Settings settings = pluginsService.updatedSettings();
        
        //1 加载插件初始化相应成员
        final NetworkModule networkModule = new NetworkModule(settings, false, pluginsService.filterPlugins(NetworkPlugin.class),
                threadPool, bigArrays, pageCacheRecycler, circuitBreakerService, namedWriteableRegistry, xContentRegistry,
                networkService, restController, clusterService.getClusterSettings());

        //2 生成负责内部节点的rpc请求服务模块
        final Transport transport = networkModule.getTransportSupplier().get();

        //3 基于 Transport 构建 TransportService 服务层
        final TransportService transportService = newTransportService(settings, transport, threadPool,
                networkModule.getTransportInterceptor(), localNodeFactory, settingsModule.getClusterSettings(), taskHeaders);

        //4 生成负责对用户的rest接口服务模块
        final HttpServerTransport httpServerTransport = newHttpTransport(networkModule);
    }
}
```

# NetworkModule 初始化

```java
public final class NetworkModule {

    //value = Transport工厂方法 = 生成负责内部节点的rpc请求服务模块
    private final Map<String, Supplier<Transport>> transportFactories = new HashMap<>();
    //value = HttpServerTransport工厂方法 = 生成负责对用户的rest接口服务模块
    private final Map<String, Supplier<HttpServerTransport>> transportHttpFactories = new HashMap<>();
    //传输拦截器
    private final List<TransportInterceptor> transportInterceptors = new ArrayList<>();
    
}
```

上述三个成员在NetworkModule的构造函数(节点启动时调用)中通过插件方式加载

# Transport 

## 初始化

```java
public final class NetworkModule {

    public Supplier<Transport> getTransportSupplier() {
        final String name;
        //settings 是否存在 transport.type 这个setting
        if (TRANSPORT_TYPE_SETTING.exists(settings)) {
            //name = SecurityField.NAME4 如果存在这个setting，获取对应值
            name = TRANSPORT_TYPE_SETTING.get(settings);
        } else {
            //如果没有 transport.type 这个setting 从 transport.type.default 获取 
            name = TRANSPORT_DEFAULT_TYPE_SETTING.get(settings);
        }
        final Supplier<Transport> factory = transportFactories.get(name);
        if (factory == null) {
            throw new IllegalStateException("Unsupported transport.type [" + name + "]");
        }
        return factory;
    }
}

```



如果配置文件 (二进制包\config\elasticsearch.yml) 不修改 transport.type ,节点级settings构建时会读取插件模块初始化 transport.type 这个 setting，默认值为 SecurityField.NAME4

NetworkModule 构造方法回调 Security 插件 getTransports 方法时，会加载 SecurityField.NAME4

```java

public class Security extends Plugin implements SystemIndexPlugin, IngestPlugin, NetworkPlugin, ClusterPlugin,
        DiscoveryPlugin, MapperPlugin, ExtensiblePlugin {
    @Override
    public Map<String, Supplier<Transport>> getTransports(Settings settings, ThreadPool threadPool, PageCacheRecycler pageCacheRecycler,
                                                          CircuitBreakerService circuitBreakerService,
                                                          NamedWriteableRegistry namedWriteableRegistry, NetworkService networkService) {
        if (transportClientMode || enabled == false) { // don't register anything if we are not enabled, or in transport client mode
            return Collections.emptyMap();
        }

        IPFilter ipFilter = this.ipFilter.get();

        Map<String, Supplier<Transport>> transports = new HashMap<>();
        //
        transports.put(SecurityField.NAME4, () -> {
            transportReference.set(new SecurityNetty4ServerTransport(settings, Version.CURRENT, threadPool,
                    networkService, pageCacheRecycler, namedWriteableRegistry, circuitBreakerService, ipFilter, getSslService(),
                    getNettySharedGroupFactory(settings)));
            return transportReference.get();
        });
        transports.put(SecurityField.NIO, () -> {
            transportReference.set(new SecurityNioTransport(settings, Version.CURRENT, threadPool, networkService,
                    pageCacheRecycler, namedWriteableRegistry, circuitBreakerService, ipFilter, getSslService(),
                    getNioGroupFactory(settings)));
            return transportReference.get();
        });

        return Collections.unmodifiableMap(transports);
    }
}
```

默认负责内部节点的rpc请求服务模块即 SecurityNetty4ServerTransport

内部节点的rpc请求,包括节点选举之间通信，各种需要主节点处理的请求需要转发，例如索引创建，mapping创建，setting创建等

这只是一小部分，还有很多内部节点的rpc请求

服务层指网络模块的上层应用层， 基于网络模块提供的 Transport 来收/发数据。该通信使用 TransportService 类实现 

在网络模块提供的 Transport 基础上， 该类提供连接到节点、 发送数据、 注册事件响应函数等方法

## org.elasticsearch.transport.TransportService#doStart

```java
public class Node implements Closeable {
    public Node start() throws NodeValidationException {
        
        //5 服务层 TransportService 开启
        TransportService transportService = injector.getInstance(TransportService.class);
        transportService.getTaskManager().setTaskResultsService(injector.getInstance(TaskResultsService.class));
        transportService.getTaskManager().setTaskCancellationService(new TaskCancellationService(transportService));
        transportService.start();
    }
}

public class TransportService extends AbstractLifecycleComponent
        implements ReportingService<TransportInfo>, TransportMessageListener, TransportConnectionListener {
    @Override
    protected void doStart() {
        transport.setMessageListener(this);
        connectionManager.addListener(this);
        //创建 Bootstrap，创建 ServerBootstrap，并绑定监听地址
        transport.start();
        if (transport.boundAddress() != null && logger.isInfoEnabled()) {
            logger.info("{}", transport.boundAddress());
            for (Map.Entry<String, BoundTransportAddress> entry : transport.profileBoundAddresses().entrySet()) {
                logger.info("profile [{}]: {}", entry.getKey(), entry.getValue());
            }
        }
        //创建本地节点
        localNode = localNodeFactory.apply(transport.boundAddress());

        if (remoteClusterClient) {
            // here we start to connect to the remote clusters
            remoteClusterService.initializeRemoteClusters();
        }
    }
}

public class SecurityNetty4ServerTransport extends SecurityNetty4Transport {
    @Override
    protected void doStart() {
        super.doStart();
        if (authenticator != null) {
            authenticator.setBoundTransportAddress(boundAddress(), profileBoundAddresses());
        }
    }
}

public class Netty4Transport extends TcpTransport {
    @Override
    protected void doStart() {
        boolean success = false;
        try {
            //获取共享 NioEventLoopGroup
            sharedGroup = sharedGroupFactory.getTransportGroup();
            //创建 Bootstrap
            clientBootstrap = createClientBootstrap(sharedGroup);
            if (NetworkService.NETWORK_SERVER.get(settings)) {
                //遍历 ProfileSettings,配置文件不改默认只有一个 ProfileSettings
                for (ProfileSettings profileSettings : profileSettings) {
                    //创建 ServerBootstrap
                    createServerBootstrap(profileSettings, sharedGroup);
                    //ServerBootstrap 绑定监听地址
                    bindServer(profileSettings);
                }
            }
            super.doStart();
            success = true;
        } finally {
            if (success == false) {
                doStop();
            }
        }
    }
}
    
```

serverBootstrap.childHandler(getServerChannelInitializer(name));


persistedState.setCurrentTerm(startJoinRequest.getTerm());
startedJoinSinceLastReboot = true;


return new PreVoteResponse(getCurrentTerm() = 0, coordinationState.get().getLastAcceptedTerm() = 0,
coordinationState.get().getLastAcceptedState().getVersionOrMetadataVersion() = 0);

new Join(node, localNode, preVoteResponse.getCurrentTerm(),
preVoteResponse.getLastAcceptedTerm(), preVoteResponse.getLastAcceptedVersion()))



final StartJoinRequest startJoinRequest
= new StartJoinRequest(getLocalNode(), Math.max(getCurrentTerm(), maxTermSeen) + 1);

node2
0
return new Join(node2, startJoinRequest.getSourceNode() = node1, getCurrentTerm() = 1, getLastAcceptedTerm() =  0,
getLastAcceptedVersionOrMetadataVersion() = 0) ;


return new Join(node1, node1, getCurrentTerm() = 1, getLastAcceptedTerm(),
getLastAcceptedVersionOrMetadataVersion());



19.1 为文件系统cache预留足够的内存
19.2 使用更快的硬件
19.3 文档模型
为了让搜索时的成本更低， 文档应该合理建模。 

特别是应该避免 join操作， 嵌套（ nested） 会使查询慢几倍， 父子（ parent-child） 关系可能使查询慢数百倍， 

因此， 如果可以通过非规范化（ denormalizing） 文 档来回答相同的问题， 则可以显著地提高搜索速度


19.4 预索引数据
还可以针对某些查询的模式来优化数据的索引方式。 例如， 如果所
有文档都有一个 price字段， 并且大多数查询在一个固定的范围上运行
range聚合， 那么可以通过将范围“pre-indexing”到索引中并使用terms聚
合来加快聚合速度

19.5 字段映射
有些字段的内容是数值， 但并不意味着其总是应该被映射为数值类
型， 例如， 一些标识符， 将它们映射为keyword可能会比integer或long更
好。


19.6 避免使用脚本
一般来说， 应该避免使用脚本。 如果一定要用， 则应该优先考虑
painless和expressions

19.7 优化日期搜索
在使用日期范围检索时， 使用now的查询通常不能缓存， 因为匹配
到的范围一直在变化。 但是， 从用户体验的角度来看， 切换到一个完整
的日期通常是可以接受的， 这样可以更好地利用查询缓存



localNodeConnection


START_JOIN_ACTION_NAME

ContextRestoreResponseHandler<T> responseHandler = new ContextRestoreResponseHandler<>(storedContextSupplier, handler);


startElectionScheduler


return new Join(localNode = node1, startJoinRequest.getSourceNode() = node1, getCurrentTerm() = 1, getLastAcceptedTerm(),
getLastAcceptedVersionOrMetadataVersion());

final JoinRequest joinRequest = new JoinRequest(transportService.getLocalNode() = node1, term = 0, optionalJoin);



return new Join(localNode = node2, startJoinRequest.getSourceNode() = node1, getCurrentTerm() = 1, getLastAcceptedTerm(),
getLastAcceptedVersionOrMetadataVersion());

final JoinRequest joinRequest = new JoinRequest(transportService.getLocalNode() = node2, term = 0, optionalJoin);

becomeLeader("handleJoinRequest");

return new Task(null, Task.BECOME_MASTER_TASK_REASON);
return new Task(null, Task.FINISH_ELECTION_TASK_REASON);




```java
class A{
    public void B(){
        new JoinTaskExecutor(settings, allocationService, logger, rerouteService) {

            private final long term = currentTermSupplier.getAsLong();

            @Override
            public ClusterTasksResult<JoinTaskExecutor.Task> execute(ClusterState currentState, List<JoinTaskExecutor.Task> joiningTasks)
                    throws Exception {
                // The current state that MasterService uses might have been updated by a (different) master in a higher term already
                // Stop processing the current cluster state update, as there's no point in continuing to compute it as
                // it will later be rejected by Coordinator.publish(...) anyhow
                if (currentState.term() > term) {
                    logger.trace("encountered higher term {} than current {}, there is a newer master", currentState.term(), term);
                    throw new NotMasterException("Higher term encountered (current: " + currentState.term() + " > used: " +
                            term + "), there is a newer master");
                } else if (currentState.nodes().getMasterNodeId() == null && joiningTasks.stream().anyMatch(Task::isBecomeMasterTask)) {
                    assert currentState.term() < term : "there should be at most one become master task per election (= by term)";
                    final CoordinationMetadata coordinationMetadata =
                            CoordinationMetadata.builder(currentState.coordinationMetadata()).term(term).build();
                    final Metadata metadata = Metadata.builder(currentState.metadata()).coordinationMetadata(coordinationMetadata).build();
                    currentState = ClusterState.builder(currentState).metadata(metadata).build();
                } else if (currentState.nodes().isLocalNodeElectedMaster()) {
                    assert currentState.term() == term : "term should be stable for the same master";
                }
                return super.execute(currentState, joiningTasks);
            }

        };        
    }
}

```






NO_MASTER_BLOCK_WRITES
STATE_NOT_RECOVERED_BLOCK

followerChecker.start();



JoinCallback


final JoinTaskExecutor.Task task = new JoinTaskExecutor.Task(key, "elect leader");
pendingAsTasks.put(task, new JoinTaskListener(task, value));

static class JoinTaskListener implements ClusterStateTaskListener {
private final JoinTaskExecutor.Task task;
private final JoinCallback joinCallback;

rerouteService.reroute("post-join reroute", Priority.HIGH, ActionListener.wrap(
r -> logger.trace("post-join reroute completed"),
e -> logger.debug("post-join reroute failed", e)));

final PlainActionFuture<Void> fut = new PlainActionFuture<Void>() {
@Override
protected boolean blockingAllowed() {
return isMasterUpdateThread() || super.blockingAllowed();
}
};

ackListener 空

publishListener 未来

new ListenableFuture<>()

Publication.this.sendPublishRequest(discoveryNode, publishRequest, new PublishResponseHandler());


return new PublishWithJoinResponse(publishResponse,
joinWithDestination(lastJoin, sourceNode, publishRequest.getAcceptedState().term()));


Publication.this.sendApplyCommit(discoveryNode, applyCommitRequest.get(), new ApplyCommitResponseHandler());







+===============================================

localNodeAckEvent = new ListenableFuture<>()

ackListener = DelegatingAckListener

publishListener = final PlainActionFuture<Void> fut = new PlainActionFuture<Void>() {
@Override
protected boolean blockingAllowed() {
return isMasterUpdateThread() || super.blockingAllowed();
}
};



IndicesClusterStateService
RepositoriesService

ScriptService
IndexLifecycleService
RestoreService
IngestService
IngestActionForwarder
TransportCleanupRepositoryAction
TimestampFieldMapperService
TaskManager

SnapshotsService


actions.register(NodesStatsAction.INSTANCE, TransportNodesStatsAction.class);
AsyncAction(Task task, NodesRequest request, ActionListener<NodesResponse> listener) {

AbstractMlAuditor org.elasticsearch.xpack.ml.notifications
InternalClusterInfoService  主节点才有效构造集群信息
InternalSnapshotsInfoService 不知道有啥用
SystemIndexManager 不知道有啥用
AutoscalingMemoryInfoService org.elasticsearch.xpack.autoscaling.capacity.memory
MlUpgradeModeActionFilter org.elasticsearch.xpack.ml
IndexTemplateRegistry  org.elasticsearch.xpack.core.template
AbstractMlAuditor 
AbstractMlAuditor
AbstractMlAuditor
ResultsPersisterService   org.elasticsearch.xpack.ml.utils.persistence
AutodetectProcessManager  org.elasticsearch.xpack.ml.job.process.autodetect
DatafeedManager           org.elasticsearch.xpack.ml.datafeed
TrainedModelStatsService  org.elasticsearch.xpack.ml.inference
ModelLoadingService       org.elasticsearch.xpack.ml.inference.loadingservice
LocalNodeMasterListener   不知道有啥用
MlAssignmentNotifier      org.elasticsearch.xpack.ml
LocalNodeMasterListener
MlInitializationService   org.elasticsearch.xpack.ml
ShardFollowTaskCleaner    org.elasticsearch.xpack.ccr.action
LocalNodeMasterListener
TransformAuditor          org.elasticsearch.xpack.transform.notifications
TransformClusterStateListener  org.elasticsearch.xpack.transform
IndexTemplateRegistry
SecurityIndexManager      org.elasticsearch.xpack.security.support
SecurityIndexManager
TokenService              org.elasticsearch.xpack.security.authc
SecurityServerTransportInterceptor      org.elasticsearch.xpack.security.transport
SearchableSnapshotIndexMetadataUpgrader org.elasticsearch.xpack.searchablesnapshots.upgrade 
IndexTemplateRegistry
WatcherLifeCycleService          org.elasticsearch.xpack.watcher
WatcherIndexingListener          org.elasticsearch.xpack.watcher 
IndexTemplateRegistry
IndexLifecycleService            org.elasticsearch.xpack.ilm 
IndexTemplateRegistry
SnapshotLifecycleService         org.elasticsearch.xpack.slm
LocalNodeMasterListener
IndexTemplateRegistry
SystemIndexMetadataUpgradeService  和索引有关，不知道做了什么
TemplateUpgradeService             和模板有关，不知道做了什么
ResponseCollectorService           和节点有关，不知道做了什么
SnapshotShardsService              不知道做了什么
AbstractMlAuditor
OpenJobPersistentTasksExecutor     org.elasticsearch.xpack.ml.job.task
TransportStartDataFrameAnalyticsAction  org.elasticsearch.xpack.ml.action
AbstractMlAuditor
SnapshotUpgradeTaskExecutor        org.elasticsearch.xpack.ml.job.snapshot.upgrader 
PersistentTasksClusterService      不知道做了什么
DelayedAllocationService           不知道做了什么
IndicesStore                       和路由表有关，不知道做了什么
PersistentTasksNodeService         不知道做了什么
LicenseService                     不知道做了什么
LocalExporter                      org.elasticsearch.xpack.monitoring.exporter.local
AutoFollowCoordinator              org.elasticsearch.xpack.ccr.action
AsyncTaskMaintenanceService        org.elasticsearch.xpack.core.async
GatewayService
PeerRecoverySourceService


```java
public class B{
    private Join joinLeaderInTerm(StartJoinRequest startJoinRequest) {
        synchronized (mutex) {
            logger.debug("joinLeaderInTerm: for [{}] with term {}", startJoinRequest.getSourceNode(), startJoinRequest.getTerm());
            final Join join = coordinationState.get().handleStartJoin(startJoinRequest);//currentTerm + 1
            lastJoin = Optional.of(join);
            peerFinder.setCurrentTerm(getCurrentTerm());
            if (mode != Mode.CANDIDATE) {
                becomeCandidate("joinLeaderInTerm"); // updates followersChecker and preVoteCollector
            } else {
                followersChecker.updateFastResponseState(getCurrentTerm(), mode);
                preVoteCollector.update(getPreVoteResponse(), null);
            }
            return join;
        }
    }
}
```


handleAssociatedJoin

onActiveMasterFound

onMissingJoin

onFollowerCheckRequest

重要

Aim to keep the average shard size between a few GB and a few tens of GB. For use cases with time-based data, it is common to see shards in the 20GB to 40GB range.
Avoid the gazillion shards problem. The number of shards a node can hold is proportional to the available heap space. As a general rule, the number of shards per GB of heap space should be less than 20.


The only reliable and supported way to back up a cluster is by taking a snapshot. You cannot back up an Elasticsearch cluster by making copies of the data directories of its nodes. There are no supported methods to restore any data from a filesystem-level backup. If you try to restore a cluster from such a backup, it may fail with reports of corruption or missing files or other data inconsistencies, or it may appear to have succeeded having silently lost some of your data.


Set Xms and Xmx to no more than 50% of your total memory. Elasticsearch requires memory for purposes other than the JVM heap. For example, Elasticsearch uses off-heap buffers for efficient network communication and relies on the operating system’s filesystem cache for efficient access to files. The JVM itself also requires some memory. It’s normal for Elasticsearch to use more memory than the limit configured with the Xmx setting.

Set Xms and Xmx to no more than the threshold for compressed ordinary object pointers (oops). The exact threshold varies but 26GB is safe on most systems and can be as large as 30GB on some systems. To verify you are under the threshold, check the Elasticsearch log for an entry like this:


Ideally, Elasticsearch should run alone on a server and use all of the resources available to it. In order to do so, you need to configure your operating system to allow the user running Elasticsearch to access more resources than allowed by default.

The following settings must be considered before going to production:

Disable swapping
Increase file descriptors
Ensure sufficient virtual memory
Ensure sufficient threads
JVM DNS cache settings
Temporary directory not mounted with noexec
TCP retransmission timeout