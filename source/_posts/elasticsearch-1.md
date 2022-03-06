---
title: 调试elasticsearch源码笔记
date: 2021-03-23 02:21:43
categories:
- spring cloud alibaba
tags:
- elasticsearch
---

# 环境

window10

# 版本 

7.13.2

# 源码下载

https://github.com/elastic/elasticsearch.git

git@github.com:elastic/elasticsearch.git

# 二进制包下载

https://www.elastic.co/cn/downloads/elasticsearch

# 下载jdk

不同版本的es要求对应的jdk版本不一样，当前版本至少要jdk15，可参考以下编译文档

https://github.com/elastic/elasticsearch/blob/master/CONTRIBUTING.md

# gradle 配置

可以参考网上教程，我偷懒不想配置了，打开源码让他自行下载gradle，包下载嫌慢下载路径可以改到阿里云，都不配的情况下，可能要等1，2小时

# build

打开idea，要求2020.1版本，版本要求同样参考上面的编译文档，我是使用 project from version control,能直接识别是gradle项目

如果是直接git clone,再用idea打开,选择 build.gradle 也能识别是gradle项目，接着等待idea帮我们build

# run

入口函数如下，直接运行

org.elasticsearch.bootstrap.Elasticsearch#main(java.lang.String[])

会直接提示你有错误

```text
"C:\Program Files\Java\jdk-15.0.2\bin\java.exe" ...
ERROR: the system property [es.path.conf] must be set

Process finished with exit code 78
```

添加

-Des.networkaddress.cache.ttl=60
-Des.networkaddress.cache.negative.ttl=10
-Djava.awt.headless=true
-Dfile.encoding=UTF-8
-Djna.nosys=true
-Dio.netty.noUnsafe=true
-Dio.netty.noKeySetOptimization=true
-Dio.netty.recycler.maxCapacityPerThread=0
-Dio.netty.allocator.numDirectArenas=0
-Dlog4j.shutdownHookEnabled=false
-Dlog4j2.disable.jmx=true
-Djava.locale.providers=SPI,COMPAT
--add-opens=java.base/java.io=ALL-UNNAMED
-Delasticsearch
-Des.distribution.flavor=default
-Des.distribution.type=zip
-Des.bundled_jdk=true
-Des.path.home=D:\a\elasticsearch-7.12.0
-Des.path.conf=D:\a\elasticsearch-7.12.0\config
-Djava.io.tmpdir=D:\a\tmp
-Xms512m
-Xms512m
-cp D:\a\elasticsearch-7.12.0\lib\*


"cluster.name" -> "elasticsearch"
"node.name" -> null
"path.home" -> "D:\a\elasticsearch-7.11.2\elasticsearch-7.11.2"
"path.logs" -> "D:\a\elasticsearch-7.11.2\elasticsearch-7.11.2\logs"



https://blog.csdn.net/weixin_36146690/article/details/114086786

https://www.colabug.com/2021/0307/8044055/

interface org.elasticsearch.painless.spi.PainlessExtension


"x-pack-autoscaling" -> {ArrayList@4353}  size = 1
"lang-painless" -> {ArrayList@4050}  size = 10
"x-pack-ql" -> {ArrayList@4377}  size = 2
"x-pack-core" -> {ArrayList@4390}  size = 37


org.elasticsearch.search.aggregations.matrix.MatrixAggregationPlugin
org.elasticsearch.analysis.common.CommonAnalysisPlugin
org.elasticsearch.xpack.constantkeyword.ConstantKeywordMapperPlugin 1
org.elasticsearch.xpack.flattened.FlattenedMapperPlugin
org.elasticsearch.xpack.frozen.FrozenIndices
org.elasticsearch.ingest.common.IngestCommonPlugin
org.elasticsearch.ingest.geoip.IngestGeoIpPlugin
org.elasticsearch.ingest.useragent.IngestUserAgentPlugin
org.elasticsearch.kibana.KibanaPlugin
org.elasticsearch.script.expression.ExpressionPlugin
org.elasticsearch.script.mustache.MustachePlugin
org.elasticsearch.painless.PainlessPlugin 1
org.elasticsearch.index.mapper.MapperExtrasPlugin
org.elasticsearch.xpack.versionfield.VersionFieldPlugin
org.elasticsearch.join.ParentJoinPlugin
org.elasticsearch.percolator.PercolatorPlugin
org.elasticsearch.index.rankeval.RankEvalPlugin
org.elasticsearch.index.reindex.ReindexPlugin
org.elasticsearch.xpack.repositories.metering.RepositoriesMeteringPlugin
org.elasticsearch.repositories.encrypted.EncryptedRepositoryPlugin 1
org.elasticsearch.plugin.repository.url.URLRepositoryPlugin
org.elasticsearch.xpack.searchablesnapshots.SearchableSnapshots 1
org.elasticsearch.xpack.searchbusinessrules.SearchBusinessRules
org.elasticsearch.repositories.blobstore.testkit.SnapshotRepositoryTestKit
org.elasticsearch.xpack.spatial.SpatialPlugin
org.elasticsearch.xpack.transform.Transform 1
org.elasticsearch.transport.Netty4Plugin
org.elasticsearch.xpack.unsignedlong.UnsignedLongMapperPlugin
org.elasticsearch.xpack.vectors.Vectors
org.elasticsearch.xpack.wildcard.Wildcard
org.elasticsearch.xpack.aggregatemetric.AggregateMetricMapperPlugin 1
org.elasticsearch.xpack.analytics.AnalyticsPlugin 1
org.elasticsearch.xpack.async.AsyncResultsIndexPlugin 1
org.elasticsearch.xpack.search.AsyncSearch
org.elasticsearch.xpack.autoscaling.Autoscaling 1
org.elasticsearch.xpack.ccr.Ccr  1
org.elasticsearch.xpack.core.XPackPlugin 1
org.elasticsearch.xpack.datastreams.DataStreamsPlugin 1
org.elasticsearch.xpack.deprecation.Deprecation
org.elasticsearch.xpack.enrich.EnrichPlugin 1
org.elasticsearch.xpack.eql.plugin.EqlPlugin
org.elasticsearch.xpack.fleet.Fleet
org.elasticsearch.xpack.graph.Graph 1
org.elasticsearch.xpack.idp.IdentityProviderPlugin
org.elasticsearch.xpack.ilm.IndexLifecycle 1
org.elasticsearch.xpack.ingest.IngestPlugin
org.elasticsearch.xpack.logstash.Logstash
org.elasticsearch.xpack.ml.MachineLearning 1
org.elasticsearch.xpack.monitoring.Monitoring 1
org.elasticsearch.xpack.ql.plugin.QlPlugin
org.elasticsearch.xpack.rollup.Rollup 1
org.elasticsearch.xpack.runtimefields.RuntimeFields 1
org.elasticsearch.xpack.security.Security 1 
org.elasticsearch.xpack.sql.plugin.SqlPlugin
org.elasticsearch.xpack.stack.StackPlugin
org.elasticsearch.xpack.textstructure.TextStructurePlugin
org.elasticsearch.cluster.coordination.VotingOnlyNodePlugin 1
org.elasticsearch.xpack.watcher.Watcher 1

"client.type" -> "node"
"cluster.name" -> "elasticsearch"
"node.name" -> null
"path.home" -> "D:\a\elasticsearch-7.12.0"
"path.logs" -> "D:\a\elasticsearch-7.12.0\logs"



org.elasticsearch.xpack.core.XPackPlugin

调度任务

lowFuture = threadPool.scheduleWithFixedDelay(lowMonitor, lowMonitor.interval, Names.SAME);
mediumFuture = threadPool.scheduleWithFixedDelay(mediumMonitor, mediumMonitor.interval, Names.SAME);
highFuture = threadPool.scheduleWithFixedDelay(highMonitor, highMonitor.interval, Names.SAME);

线程
this.cachedTimeThread = new CachedTimeThread(EsExecutors.threadName(settings, "[timer]"), estimatedTimeInterval.millis());
this.cachedTimeThread.start();



-Xms256m
-Xmx256m
-Djava.awt.headless=true
-Dfile.encoding=UTF-8
-Djna.nosys=true
-Dio.netty.noUnsafe=true
-Dio.netty.noKeySetOptimization=true
-Dio.netty.recycler.maxCapacityPerThread=0
-Dlog4j.shutdownHookEnabled=false
-Dlog4j2.disable.jmx=true
-Delasticsearch
-Des.path.home=D:\a\612\elasticsearch-6.1.2\elasticsearch-6.1.2
-Des.path.conf=D:\a\612\elasticsearch-6.1.2\elasticsearch-6.1.2\config
-cp D:\a\612\elasticsearch-6.1.2\elasticsearch-6.1.2\lib\*

request header 长度 int
request header 内容
response header 长度 int 
response header 内容
action 长度 int
action 内容
一个字符站一个字节
nodeId 长度 

output.writeByte((byte)'E');
output.writeByte((byte)'S');
// write the size, the size indicates the remaining message size, not including the size int
output.writeInt(messageSize + REQUEST_ID_SIZE + STATUS_SIZE + VERSION_ID_SIZE);
output.writeLong(requestId);
output.writeByte(status);
output.writeInt(version.id);

transportService.registerRequestHandler(ACTION_NAME, UnicastPingRequest::new, ThreadPool.Names.SAME,
new UnicastPingRequestHandler());

transportService.sendRequest(node, SEND_ACTION_NAME,
new BytesTransportRequest(bytes, node.getVersion()),
options,
new EmptyTransportResponseHandler(ThreadPool.Names.SAME) {

                        @Override
                        public void handleResponse(TransportResponse.Empty response) {
                            if (sendingController.getPublishingTimedOut()) {
                                logger.debug("node {} responded for cluster state [{}] (took longer than [{}])", node,
                                    clusterState.version(), publishTimeout);
                            }
                            sendingController.onNodeSendAck(node);
                        }
    
                        @Override
                        public void handleException(TransportException exp) {
                            if (sendDiffs && exp.unwrapCause() instanceof IncompatibleClusterStateVersionException) {
                                logger.debug("resending full cluster state to node {} reason {}", node, exp.getDetailedMessage());
                                sendFullClusterState(clusterState, serializedStates, node, publishTimeout, sendingController);
                            } else {
                                logger.debug((org.apache.logging.log4j.util.Supplier<?>) () ->
                                    new ParameterizedMessage("failed to send cluster state to {}", node), exp);
                                sendingController.onNodeSendFailed(node, exp);
                            }
                        }
                    });

transportService.registerRequestHandler(SEND_ACTION_NAME, BytesTransportRequest::new, ThreadPool.Names.SAME, false, false,
new SendClusterStateRequestHandler());

tasks.put(BECOME_MASTER_TASK, (source1, e) -> {}); // noop listener, the election finished listener determines result
tasks.put(FINISH_ELECTION_TASK, electionFinishedListener);


STATE_NOT_RECOVERED_BLOCK

customSupplier.put(SnapshotDeletionsInProgress.TYPE, SnapshotDeletionsInProgress::new);
customSupplier.put(RestoreInProgress.TYPE, RestoreInProgress::new);
customSupplier.put(SnapshotsInProgress.TYPE, SnapshotsInProgress::new);

IndexGraveyard.TYPE, new IndexGraveyard(tombstones)


httpRequest = new Netty4HttpRequest(serverTransport.xContentRegistry, copy, ctx.channel());

final Netty4HttpChannel channel =
new Netty4HttpChannel(serverTransport, httpRequest, pipelinedRequest, detailedErrorsEnabled, threadContext);

return channel ->
client.index(indexRequest, new RestStatusToXContentListener<>(channel, r -> r.getLocation(indexRequest.routing())));

execute(IndexAction.INSTANCE, request, listener);

ActionHandler

actions.register(IndexAction.INSTANCE, TransportIndexAction.class);

execute(task, request, new ActionListener<Response>() {
@Override
public void onResponse(Response response) {
taskManager.unregister(task);
listener.onResponse(response);
}

                @Override
                public void onFailure(Exception e) {
                    taskManager.unregister(task);
                    listener.onFailure(e);
                }
            });

doExecute

protected abstract void doExecute(Request request, ActionListener<Response> listener);

this.action.doExecute(task, request, listener);









return channel ->
client.index(indexRequest, new RestStatusToXContentListener<>(channel, r -> r.getLocation(indexRequest.routing())));
IndexRequest
TransportIndexAction

execute(task, request, new ActionListener<Response>() {
@Override
public void onResponse(Response response) {
taskManager.unregister(task);
listener.onResponse(response);
}

                @Override
                public void onFailure(Exception e) {
                    taskManager.unregister(task);
                    listener.onFailure(e);
                }
            });

bulkAction.execute(task, toSingleItemBulkRequest(request), wrapBulkResponse(listener));

this.action.doExecute(task, request, listener);



createIndex(index, bulkRequest.timeout(), new ActionListener<CreateIndexResponse>() {
@Override
public void onResponse(CreateIndexResponse result) {
if (counter.decrementAndGet() == 0) {
executeBulk(task, bulkRequest, startTime, listener, responses, indicesThatCannotBeCreated);
}
}

                        @Override
                        public void onFailure(Exception e) {
                            if (!(ExceptionsHelper.unwrapCause(e) instanceof ResourceAlreadyExistsException)) {
                                // fail all requests involving this index, if create didn't work
                                for (int i = 0; i < bulkRequest.requests.size(); i++) {
                                    DocWriteRequest request = bulkRequest.requests.get(i);
                                    if (request != null && setResponseFailureIfIndexMatches(responses, i, request, index, e)) {
                                        bulkRequest.requests.set(i, null);
                                    }
                                }
                            }
                            if (counter.decrementAndGet() == 0) {
                                executeBulk(task, bulkRequest, startTime, ActionListener.wrap(listener::onResponse, inner -> {
                                    inner.addSuppressed(e);
                                    listener.onFailure(inner);
                                }), responses, indicesThatCannotBeCreated);
                            }
                        }
                    });

CreateIndexRequest createIndexRequest = new CreateIndexRequest();
createIndexRequest.index(index);
createIndexRequest.cause("auto(bulk api)");
createIndexRequest.masterNodeTimeout(timeout);
createIndexAction.execute(createIndexRequest, listener);


execute(task, request, new ActionListener<Response>() {
@Override
public void onResponse(Response response) {
taskManager.unregister(task);
listener.onResponse(response);
}

                @Override
                public void onFailure(Exception e) {
                    taskManager.unregister(task);
                    listener.onFailure(e);
                }
            });

TransportCreateIndexAction
doExecute

ActionListener<Response> delegate = new ActionListener<Response>() {
@Override
public void onResponse(Response response) {
listener.onResponse(response);
}

                        @Override
                        public void onFailure(Exception t) {
                            if (t instanceof Discovery.FailedToCommitClusterStateException
                                    || (t instanceof NotMasterException)) {
                                logger.debug((org.apache.logging.log4j.util.Supplier<?>) () -> new ParameterizedMessage("master could not publish cluster state or stepped down before publishing action [{}], scheduling a retry", actionName), t);
                                retry(t, masterChangePredicate);
                            } else {
                                listener.onFailure(t);
                            }
                        }
                    };

masterOperation(task, request, clusterState, delegate);
AckedClusterStateTaskListener
new IndexCreationTask(logger, allocationService, request, listener, indicesService, aliasValidator, xContentRegistry, settings,
this::validate)

finalListeners.add(onStoreClose);
finalListeners.add(oldShardsStats);

PreConfiguredCharFilter

PreConfiguredTokenFilter

PreConfiguredTokenizer


PreBuiltAnalyzers
analyzerProviderFactories.put(name, new PreBuiltAnalyzerProviderFactory(name, AnalyzerScope.INDICES, preBuiltAnalyzerEnum.getAnalyzer(Version.CURRENT)));
final Analyzer a = new StandardAnalyzer(CharArraySet.EMPTY_SET);
PreBuiltAnalyzerProvider
analyzers.register("default", StandardAnalyzerProvider::new);
analyzers.register("standard", StandardAnalyzerProvider::new);
analyzers.register("standard_html_strip", StandardHtmlStripAnalyzerProvider::new);
analyzers.register("simple", SimpleAnalyzerProvider::new);
analyzers.register("stop", StopAnalyzerProvider::new);
analyzers.register("whitespace", WhitespaceAnalyzerProvider::new);
analyzers.register("keyword", KeywordAnalyzerProvider::new);
analyzers.register("pattern", PatternAnalyzerProvider::new);
analyzers.register("snowball", SnowballAnalyzerProvider::new);
analyzers.register("arabic", ArabicAnalyzerProvider::new);
analyzers.register("armenian", ArmenianAnalyzerProvider::new);
analyzers.register("basque", BasqueAnalyzerProvider::new);
analyzers.register("bengali", BengaliAnalyzerProvider::new);
analyzers.register("brazilian", BrazilianAnalyzerProvider::new);
analyzers.register("bulgarian", BulgarianAnalyzerProvider::new);
analyzers.register("catalan", CatalanAnalyzerProvider::new);
analyzers.register("chinese", ChineseAnalyzerProvider::new);
analyzers.register("cjk", CjkAnalyzerProvider::new);
analyzers.register("czech", CzechAnalyzerProvider::new);
analyzers.register("danish", DanishAnalyzerProvider::new);
analyzers.register("dutch", DutchAnalyzerProvider::new);
analyzers.register("english", EnglishAnalyzerProvider::new);
analyzers.register("finnish", FinnishAnalyzerProvider::new);
analyzers.register("french", FrenchAnalyzerProvider::new);
analyzers.register("galician", GalicianAnalyzerProvider::new);
analyzers.register("german", GermanAnalyzerProvider::new);
analyzers.register("greek", GreekAnalyzerProvider::new);
analyzers.register("hindi", HindiAnalyzerProvider::new);
analyzers.register("hungarian", HungarianAnalyzerProvider::new);
analyzers.register("indonesian", IndonesianAnalyzerProvider::new);
analyzers.register("irish", IrishAnalyzerProvider::new);
analyzers.register("italian", ItalianAnalyzerProvider::new);
analyzers.register("latvian", LatvianAnalyzerProvider::new);
analyzers.register("lithuanian", LithuanianAnalyzerProvider::new);
analyzers.register("norwegian", NorwegianAnalyzerProvider::new);
analyzers.register("persian", PersianAnalyzerProvider::new);
analyzers.register("portuguese", PortugueseAnalyzerProvider::new);
analyzers.register("romanian", RomanianAnalyzerProvider::new);
analyzers.register("russian", RussianAnalyzerProvider::new);
analyzers.register("sorani", SoraniAnalyzerProvider::new);
analyzers.register("spanish", SpanishAnalyzerProvider::new);
analyzers.register("swedish", SwedishAnalyzerProvider::new);
analyzers.register("turkish", TurkishAnalyzerProvider::new);
analyzers.register("thai", ThaiAnalyzerProvider::new);
analyzers.register("fingerprint", FingerprintAnalyzerProvider::new);
analyzers.extractAndRegister(plugins, AnalysisPlugin::getAnalyzers);

canAllocate

addAllocationDecider(deciders, new MaxRetryAllocationDecider(settings));
addAllocationDecider(deciders, new ResizeAllocationDecider(settings));
addAllocationDecider(deciders, new ReplicaAfterPrimaryActiveAllocationDecider(settings));
addAllocationDecider(deciders, new RebalanceOnlyWhenActiveAllocationDecider(settings));
addAllocationDecider(deciders, new ClusterRebalanceAllocationDecider(settings, clusterSettings));
addAllocationDecider(deciders, new ConcurrentRebalanceAllocationDecider(settings, clusterSettings));
addAllocationDecider(deciders, new EnableAllocationDecider(settings, clusterSettings)); 3
addAllocationDecider(deciders, new NodeVersionAllocationDecider(settings));
addAllocationDecider(deciders, new SnapshotInProgressAllocationDecider(settings));
addAllocationDecider(deciders, new RestoreInProgressAllocationDecider(settings));
addAllocationDecider(deciders, new FilterAllocationDecider(settings, clusterSettings));
addAllocationDecider(deciders, new SameShardAllocationDecider(settings, clusterSettings));
addAllocationDecider(deciders, new DiskThresholdDecider(settings, clusterSettings));
addAllocationDecider(deciders, new ThrottlingAllocationDecider(settings, clusterSettings));
addAllocationDecider(deciders, new ShardsLimitAllocationDecider(settings, clusterSettings));
addAllocationDecider(deciders, new AwarenessAllocationDecider(settings, clusterSettings));

private final IndexMetaDataUpdater indexMetaDataUpdater = new IndexMetaDataUpdater();
private final RoutingNodesChangedObserver nodesChangedObserver = new RoutingNodesChangedObserver();
private final RestoreInProgressUpdater restoreInProgressUpdater = new RestoreInProgressUpdater();

cachedDecisions.put(AllocationStatus.DECIDERS_NO,
new AllocateUnassignedDecision(AllocationStatus.DECIDERS_NO, null, null, null, false, 0L, 0L));

TransportShardBulkAction



```json

```

文档元素据

元数据，用于标注文档的相关信息
_index一文档所属的索引名
_type一文档所属的类型名
_id一文档唯一Id
_source:文档的原始Json数据
_all:整合所有字段内容到该字段，已被废除
_version:文档的版本信息
_score:相关性打分

索引

Index一索引是文档的容器，是一类文档的结合
      Index体现了逻辑空间的概念:每个索引都有
      自己的Mapping定义，用于定义包含的文档
      的字段名和字段类型
      Shard体现了物理空间的概念:索引中的数据
      分散在Shard上
索引的Mapping与Settings
Mapping定义文档字段的类型
Setting定义不同的数据分布

Master-eligible nodes和Master Node

每个节点启动后，默认就是一个Master eligible节点
      可以设置node.master: false禁止
Master-eligible节点可以参加选主流程，成为Master节点
当第一个节点启动时候，它会将自己选举成Master节点
每个节点上都保存了集群的状态，只有Master节点才能修改集群的状态信息
    集群状态(Cluster State)，维护了一个集群中，必要的信息
          所有的节点信息
          所有的索引和其相关的Mapping与Setti ng信息
            分片的路由信息
      任意节点都能修改信息会导致数据的不一致性

Data Node&Coordinating Node
Data Node
    可以保存数据的节点，叫做Data Node。负责保存分片数据。在数据扩展上起到了至关重要的作用
Coordinating Node
负责接受Client的请求，将请求分发到合适的节点，最终把结果汇集到一起
每个节点默认都起到了Coordinating Node的职责

.Hot&Warm Node
不同硬件配置的Data Node，用来实现Hot&Warm架构，降低集群部署的成本
.Machine Learning Node
负责跑机器学习的Job，用来做异常检测
Tribe Node
(5.3开始使用Cross Cluster Serarch)tribe Node连接到不同的elasticsearch集群
并且支持将这些集群当成一个单独的集群处理

配置节点类型
开发环境中一个节点可以承担多种角色
生产环境中，应该设置单一的角色的节点(dedicated node)

分片(Primary Shard&Replica Shard)

主分片，用以解决数据水平扩展的问题。通过主分片，可以将数据分布到集群内的所有节点之上
一个分片是一个运行的Lucene的实例
主分片数在索引创建时指定，后续不允许修改，除非Reindex
副本，用以解决数据高可用的问题。分片是主分片的拷贝
副本分片数，可以动态题调整
增加副本数，还可以在一定程度上提高服务的可用性(读取的吞吐)

对于生产环境中分片的设定，需要提前做好容量规划
分片数设置过小
导致后续无法增加节点实现水品扩展
单个分片的数据量太大，导致数据重新分配耗时
分片数设置过大，7.0开始，默认主分片设置成1，解决了over-sha rd i ng的问题
影响搜索结果的相关’}生打分，影响统计结果的准确’}生
单个节点上过多的分片，会导致资源浪费，同时也会影响’}生能

Green一主分片与副本都正常分配
yellow一主分片全部正常分配，有副本分片未能正常分配
Red一有主分片未能分配
    例如，当服务器的磁盘容量超过85%时
    去创建了一个新的索引

type名，约定都用_doc
Create一如果ID已经存在，会失败
Index一如果ID不存在，创建新的文档。否则，先删除现有的文档再创建新的文档，版本会增加
Update一文档必须已经存在，更新只会对相应字段做增量修改

具体操作看 3.3-文档的基本CRUD与批量操作

Bulk API

支持在一次API调用中，对不同的索引进行操作
支持四种类型操作
Index
Create
Update
Delete
可以再URI中指定Index，也可以在请求的Payload中进行
操作中单条操作失败，并不会影响其他操作
返回结果包括了每一条操作执行的结果

具体操作看 3.3-文档的基本CRUD与批量操作

批量读取文档一mget
批量操作，可以减少网络连接所产生的开销，提高性能

具体操作看 3.3-文档的基本CRUD与批量操作

批量查询一msearch

具体操作看 3.3-文档的基本CRUD与批量操作

倒排索引的核心组成

倒排索引包含两个部分
单词词典(term Dictionary)，记录所有文档的单词，记录单词到倒排列表的关联关系
单词词典一般比较大，可以通过B+树或哈希拉链法实现，以满足高性能的插入与查询
倒排列表(Posting List)一记录了单词对应的文档结合，由倒排索引项组成
倒排索引项(Posting)
.文档ID
.词频TF一该单词在文档中出现的次数，用于相关性评分
·位置(Position)一单词在文档中分词的位置。用于语句搜索(phrase query)
.偏移(Offset)一记录单词的开始结束位置，实现高亮显示

Elasticsearch的JSON文档中的每个字段，都有自己的倒排索引
可以指定对某些字段不做索引
优点:节省存储空间
缺点:字段无法被搜索

Analysis与Analyzer

Analysis一文本分析是把全文本转换一系列单词(term / token)的过程，也叫分词
Analysis是通过Analyzer来实现的
可使用Elasticsearch内置的分析器/或者按需定制化分析器
除了在数据写入时转换词条，匹配Query语句时候也需要用相同的分析器对查询语句进行分析

分词器是专门处理分词的组件，Analyzer由三部分组成
    Character Filters(针对原始文本处理，例如去除html)/tokenizer(按照规则切分为单词)/token Filter(将切分的的单词进行加工，小写，删除stopwords增加同义词)

具体例子 3.5-通过分析器进行分词

什么是Mapping

Mapping类似数据库中的schema的定义，作用如下
      定义索引中的字段的名称
      定义字段的数据类型，例如字符串，数字，布尔……
    字段，倒排索引的相关配置，(Analyzed or Not Analyzed, Analyzer)
Mapping会把JSON文档映射成Lucene所需要的扁平格式
一个Mapping属于一个索引的type
      每个文档都属于一个type
    一个type有一个Mapping定义
    7.0开始，不需要在Mapping定义中指定type信息

字段的数据类型

简单类型
Text/Keyword
Date
Integer/Floating
Boolean
IPv4&IPv6
复杂类型一对象和嵌套对象
对象类型/嵌套类型
特殊类型
geo_point&geo_ shape/percolator

什么是Dynamic Mapping

在写入文档时候，如果索引不存在会自动创建索引
Dynamic Mapping的机制，使得我们无需手动定义MappingsoElasticsearch会自动根据文档信息，推算出字段的类型
但是有时候会推算的不对，例如地理位置信息
当类型如果设置不对时，会导致一些功能无法正常运行，例如Range查询

类型的自动识别

<table>
    <tr>
        <td>JSON类型</td> 
        <td>Elasticsearch类型</td> 
   </tr>
    <tr>
        <td>字符串</td> 
        <td>匹配日期格式，设置成Date
配置数字设置为float或者Long，该选项默认关闭
设置为text，并且增加keyword子字段
</td> 
   </tr>
    <tr>
        <td>布尔值</td> 
        <td>Boolean</td> 
   </tr>   
    <tr>
        <td>浮点数</td> 
        <td>float</td> 
   </tr>
    <tr>
        <td>整数</td> 
        <td>Long</td> 
   </tr>
    <tr>
        <td>对象</td> 
        <td>Object</td> 
   </tr>    
    <tr>
        <td>数组</td> 
        <td>由第一个非空数值的类型所决定</td> 
   </tr>   
    <tr>
        <td>空值</td> 
        <td>忽略</td> 
   </tr>    
</table>


Term 是表达语意的最⼩小单位。搜索和利利⽤用统计语⾔言模型进⾏行行⾃自然语⾔言处理理都需要处理理 Term  

在 ES 中， Term 查询，对输⼊不做分词。会将输⼊作为一个整体，在倒排索引中查找准确的词项， 并且使⽤用相关度算分公式为每个包含该词项的⽂文档进⾏行行相关度算分 – 例例如“Apple Store”
可以通过 Constant Score 将查询转换成⼀一个 Filtering，避免算分，并利利⽤用缓存，提⾼高性能  

索引和搜索时都会进行分词，查询字符串先传递到⼀个合适的分词器，然后生成一个供查询的词项列表
● 查询时候，先会对输入的查询进⾏分词，然后每个词项逐个进行底层的查询，最终将结果进行合
并。并为每个⽂档⽣生成⼀个算分。 -例例如查 “Matrix reloaded”，会查到包括 Matrix 或者 reload
的所有结果。  

