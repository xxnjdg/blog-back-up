---
title: elasticsearch-3
date: 2021-05-07 13:20:36
categories:
- spring cloud alibaba
tags:
- elasticsearch
---

# org.elasticsearch.action.bulk.TransportBulkAction#createIndex

```java
public class a{
//创建索引
    void createIndex(String index, TimeValue timeout, ActionListener<CreateIndexResponse> listener) {
        CreateIndexRequest createIndexRequest = new CreateIndexRequest();
        createIndexRequest.index(index);
        createIndexRequest.cause("auto(bulk api)");
        createIndexRequest.masterNodeTimeout(timeout);
        createIndexAction.execute(createIndexRequest, listener);
    }
}
```

索引创建请求发送到主节点处理

# org.elasticsearch.action.admin.indices.create.TransportCreateIndexAction#masterOperation

```java
public class TransportCreateIndexAction extends TransportMasterNodeAction<CreateIndexRequest, CreateIndexResponse> {
    @Override
    protected void masterOperation(final CreateIndexRequest request, final ClusterState state, final ActionListener<CreateIndexResponse> listener) {
        String cause = request.cause();
        if (cause.length() == 0) {
            cause = "api";
        }

        //索引名
        final String indexName = indexNameExpressionResolver.resolveDateMathExpression(request.index());
        final CreateIndexClusterStateUpdateRequest updateRequest = new CreateIndexClusterStateUpdateRequest(request, cause, indexName, request.index(), request.updateAllTypes())
                .ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
                .settings(request.settings()).mappings(request.mappings())
                .aliases(request.aliases()).customs(request.customs())
                .waitForActiveShards(request.waitForActiveShards());

        createIndexService.createIndex(updateRequest, ActionListener.wrap(response ->
                        listener.onResponse(new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcked(), indexName)),
                listener::onFailure));
    }
}

public class MetaDataCreateIndexService extends AbstractComponent {
    public void createIndex(final CreateIndexClusterStateUpdateRequest request,
                            final ActionListener<CreateIndexClusterStateUpdateResponse> listener) {
        onlyCreateIndex(request, ActionListener.wrap(response -> {
            if (response.isAcknowledged()) {
                activeShardsObserver.waitForActiveShards(new String[]{request.index()}, request.waitForActiveShards(), request.ackTimeout(),
                        shardsAcked -> {
                            if (shardsAcked == false) {
                                logger.debug("[{}] index created, but the operation timed out while waiting for " +
                                        "enough shards to be started.", request.index());
                            }
                            listener.onResponse(new CreateIndexClusterStateUpdateResponse(response.isAcknowledged(), shardsAcked));
                        }, listener::onFailure);
            } else {
                listener.onResponse(new CreateIndexClusterStateUpdateResponse(false, false));
            }
        }, listener::onFailure));
    }

    private void onlyCreateIndex(final CreateIndexClusterStateUpdateRequest request,
                                 final ActionListener<ClusterStateUpdateResponse> listener) {
        Settings.Builder updatedSettingsBuilder = Settings.builder();
        Settings build = updatedSettingsBuilder.put(request.settings()).normalizePrefix(IndexMetaData.INDEX_SETTING_PREFIX).build();
        indexScopedSettings.validate(build, true); // we do validate here - index setting must be consistent
        request.settings(build);
        //提交任务
        clusterService.submitStateUpdateTask("create-index [" + request.index() + "], cause [" + request.cause() + "]",
                new IndexCreationTask(logger, allocationService, request, listener, indicesService, aliasValidator, xContentRegistry, settings,
                        this::validate));
    }
}
```

IndexCreationTask

SafeAckedClusterStateTaskListener
TieBreakingPrioritizedRunnable

Index 相关 API

```
#查看索引相关信息
GET kibana_sample_data_ecommerce

#查看索引的文档总数
GET kibana_sample_data_ecommerce/_count

#查看前10条文档，了解文档格式
POST kibana_sample_data_ecommerce/_search
{
}

#_cat indices API
#查看indices
GET /_cat/indices/kibana*?v&s=index

#查看状态为绿的索引
GET /_cat/indices?v&health=green

#按照文档个数排序
GET /_cat/indices?v&s=docs.count:desc

#查看具体的字段
GET /_cat/indices/kibana*?pri&v&h=health,index,pri,rep,docs.count,mt

#How much memory is used per index?
GET /_cat/indices?v&h=i,tm&s=tm:desc


```



```
get _cat/nodes?v
GET /_nodes/es7_01,es7_02
GET /_cat/nodes?v
GET /_cat/nodes?v&h=id,ip,port,v,m


GET _cluster/health
GET _cluster/health?level=shards
GET /_cluster/health/kibana_sample_data_ecommerce,kibana_sample_data_flights
GET /_cluster/health/kibana_sample_data_flights?level=shards

#### cluster state
The cluster state API allows access to metadata representing the state of the whole cluster. This includes information such as
GET /_cluster/state

#cluster get settings
GET /_cluster/settings
GET /_cluster/settings?include_defaults=true

GET _cat/shards
GET _cat/shards?h=index,shard,prirep,state,unassigned.reason
```

final CreateIndexClusterStateUpdateRequest updateRequest = new CreateIndexClusterStateUpdateRequest(request, cause, indexName, request.index(), request.updateAllTypes())
.ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
.settings(request.settings()).mappings(request.mappings())
.aliases(request.aliases()).customs(request.customs())
.waitForActiveShards(request.waitForActiveShards());

        createIndexService.createIndex(updateRequest, ActionListener.wrap(response ->
            listener.onResponse(new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcked(), indexName)),
            listener::onFailure));

new IndexCreationTask(logger, allocationService, request, listener, indicesService, aliasValidator, xContentRegistry, settings,
this::validate))


1 从集群状态元数据找出 templates
2 从请求中解析出mappings
3 设置索引setting,优先使用templates，如果没有使用默认setting
4 构造了新 indexMetaData
5 把 indexMetaData 设置进 MetaData，生成新 MetaData
6 把 MetaData 设置进 ClusterState，生成新 ClusterState
7 重新设置路由表，把 RoutingTable 设置进 ClusterState
8 返回新状态，得到新集群状态后，开始给节点分配分片，刚开始只会把主分片分给节点，副分片不动，主分片状态由UNASSIGNED 转成 INITIALIZING
9 开始把新的集群状态发送给集群所有节点
10 org.elasticsearch.cluster.service.ClusterApplierService#callClusterStateAppliers 方法开始对新集群状态设置
org.elasticsearch.indices.cluster.IndicesClusterStateService#applyClusterState

11 接受到状态后，查找该节点分配到的分片，创建分片IndexShard
创建_state和index文件夹，_state下写入state-0.st文件的是否为主分片，索引id,分片id
translog translog.ckp translog-1.tlog


org.elasticsearch.index.engine.InternalEngine#getIndexWriterConfig

创建 IndexWriterConfig

org.elasticsearch.index.engine.InternalEngine#createWriter(org.apache.lucene.store.Directory, org.apache.lucene.index.IndexWriterConfig)

//创建 IndexWriter

seqNoStats = new SeqNoStats(
SequenceNumbers.NO_OPS_PERFORMED,
SequenceNumbers.NO_OPS_PERFORMED,
SequenceNumbers.UNASSIGNED_SEQ_NO);

seqNoService.getGlobalCheckpoint()

final long minSeqNo = SequenceNumbers.NO_OPS_PERFORMED;
final long maxSeqNo = SequenceNumbers.NO_OPS_PERFORMED;

Translog.ckp

return new Checkpoint(offset = 0, numOps=0, generation=1, minSeqNo=-1, maxSeqNo=-1, globalCheckpoint=-2, minTranslogGeneration=1);

Checkpoint数据写入Translog.ckp文件

translog-1.tlog

CodecUtil.writeHeader(out, TRANSLOG_CODEC, VERSION);
translogUUID

ref=translogUUID
return new Checkpoint(offset = getHeaderLength(ref.length), numOps=0, generation=1, minSeqNo=-1, maxSeqNo=-1, globalCheckpoint=-2, minTranslogGeneration=1);

段提交

开始生成 StandardDirectoryReader,并生成IndexSearcher
internalSearcherManager.addListener(versionMap);
//new RefreshMetricUpdater(refreshMetric)
this.internalSearcherManager.addListener(listener);
//buildRefreshListeners()
this.externalSearcherManager.addListener(listener);

refresh * 2


主分片状态由INITIALIZING 转成 STARTED
给主节点发送请求开始为副分片分配节点
从新得到新的集群状态，发布新状态，得到新的状态节点开始更新

主分片更新

IndexNotFoundException FileNotFoundException IOException    


副分片创建IndexShared,后开始索引恢复，发送请求给主分片节点

主分片节点接受到请求后，传送主分片给副分片，副分片创建 Engine,副分片恢复成功

副分片更新，只是换了状态




nonFailedTasks.forEach(task -> task.listener.clusterStateProcessed(task.source(), previousClusterState, newClusterState));
taskInputs.executor.clusterStatePublished(clusterChangedEvent);


SubReaderWrapper

onTragicEvent

tragicEvent

@Override
public IndexSearcher newSearcher(IndexReader reader, IndexReader previousReader) throws IOException {
IndexSearcher searcher = super.newSearcher(reader, previousReader);
searcher.setQueryCache(engineConfig.getQueryCache());
searcher.setQueryCachingPolicy(engineConfig.getQueryCachingPolicy());
searcher.setSimilarity(engineConfig.getSimilarity());
return searcher;
}

(searcher) -> { IndexShard shard =  getShardOrNull(shardId.getId()); if (shard != null) { warmer.warm(searcher, shard, IndexService.this.indexSettings);}}

org.elasticsearch.index.shard.IndexShard#newEngineConfig

DirectoryReader.openIfChanged((DirectoryReader) r);

专家：返回只读读取器，涵盖对索引的所有已提交和未提交的更改。 这提供了“近乎实时”的搜索，因为在 IndexWriter 会话期间所做的更改可以快速用于搜索，而无需关闭 writer 或调用 {@link #commit}。

<p>请注意，这在功能上等同于调用 {flush} 然后打开一个新的阅读器。
但是这种方法的周转时间应该更快，因为它避免了潜在的代价高昂的 {@link #commit}。

<p>一旦使用完毕，您必须关闭此方法返回的 {@link IndexReader}。

<p>它<i>接近</i>实时，因为没有硬性保证在使用 IndexWriter 进行更改后您能多快获得新阅读器。 您必须根据自己的情况进行试验，以确定它是否足够快。 由于这是一个新的实验性功能，请报告您的发现，以便我们可以学习、改进和迭代。

<p>生成的阅读器支持 {@link DirectoryReader#openIfChanged}，但该调用将简单地转发回此方法（尽管将来可能会改变）。

<p>第一次调用此方法时，此写入器实例将尽一切努力将其打开的读取器池化以进行合并、应用删除等。这意味着额外的资源（RAM、文件描述符、CPU 时间）将被占用 消耗。

<p>为了降低重新打开阅读器的延迟，您应该调用 {@link IndexWriterConfig#setMergedSegmentWarmer} 在新合并的段提交到索引之前对其进行预热。 这对于在大型合并后最小化索引到搜索的延迟很重要。

<p>如果 addIndexes* 调用正在另一个线程中运行，那么这个读取器将只从外部索引中搜索到目前为止已成功复制的那些段。

<p><b>注意</b>：一旦写入器关闭，任何未完成的读取器可能会继续使用。但是，如果您尝试重新打开这些读取器中的任何一个，您将遇到 {@link AlreadyClosedException}。

索引中有多少文档，或者正在添加（保留）的过程中。 例如，像 addIndexes 这样的操作将在实际更改索引之前首先保留添加 N 个文档的权利，就像酒店如何在您的信用卡上放置“授权保留”以确保他们稍后可以在您退房时向您收费。

final IndicesService indicesService = new IndicesService(settings, pluginsService, nodeEnvironment, xContentRegistry,
analysisModule.getAnalysisRegistry(),
clusterModule.getIndexNameExpressionResolver(), indicesModule.getMapperRegistry(), namedWriteableRegistry,
threadPool, settingsModule.getIndexScopedSettings(), circuitBreakerService, bigArrays, scriptModule.getScriptService(),
client, metaStateService);



this.buildInIndexListener =
Arrays.asList(
peerRecoverySourceService,
recoveryTargetService,
searchService,
syncedFlushService,
snapshotShardsService);

shardStartedClusterStateTaskExecutor
threadPool.schedule(activityTimeout, ThreadPool.Names.GENERIC,
new RecoveryMonitor(recoveryTarget.recoveryId(), recoveryTarget.lastAccessTime(), activityTimeout));


org.elasticsearch.indices.recovery.PeerRecoveryTargetService#doRecovery


当节点异常重启时， 写入磁盘的数据先到文件系统的缓冲， 未必来得及刷盘， 如果不通过某种方式将未刷盘的数据找回来， 则会丢失一些数据， 这是保持数据完整性的体现； 
另一方面， 由于写入操作在多个分片副本上没有来得及全部执行， 副分片需要同步成和主分片完全一致， 这是数据副 本一致性的体现

主分片从translog中自我恢复， 尚未执行flush到磁盘的Lucene分段
可以从translog中重建；
· 副分片需要从主分片中拉取Lucene分段和translog进行恢复。 但是
有机会跳过拉取Lucene分段的过程。
索引恢复的触发条件包括从快照备份恢复、 节点加入和离开、 索引
的_open操作等



org.apache.lucene.codecs.lucene90.Lucene90Codec
org.apache.lucene.codecs.lucene90.Lucene90Codec
Lucene90CompressingStoredFieldsWriter

CharTermAttribute
PackedTokenAttributeImpl

this.attributes = input.attributes;
this.attributeImpls = input.attributeImpls;
this.currentState = input.currentState;
this.factory = input.factory;

return new TokenStreamComponents(
r -> {
src.setMaxTokenLength(StandardAnalyzer.this.maxTokenLength);
src.setReader(r);
},
tok);



schema.reset(docId);
invertState.reset();
stream.reset();

Lucene90NormsProducer

final SegmentInfos sis = new SegmentInfos(config.getIndexCreatedVersionMajor());

final SegmentWriteState flushState =
new SegmentWriteState(
infoStream,
directory,
segmentInfo,
fieldInfos.finish(),
pendingUpdates,
new IOContext(new FlushInfo(numDocsInRAM, lastCommittedBytesUsed)));

org.apache.lucene.index.DocumentsWriter#flushAllThreads 
加锁

org.apache.lucene.index.DocumentsWriterFlushControl#markForFullFlush 

加锁换DocumentsWriterDeleteQueue

DeltaPackedLongValues

Lucene90PostingsWriter

new FreqProxTerms(perField)

FreqProxTermsEnum termsEnum = new FreqProxTermsEnum(terms);

posEnum = new FreqProxPostingsEnum(terms, postingsArray);


builder = new TextFieldMapper.Builder(currentFieldName)
.addMultiField(new KeywordFieldMapper.Builder("keyword").ignoreAbove(256));

return new KeywordFieldMapper(
name, fieldType, defaultFieldType, ignoreAbove, includeInAll,
context.indexSettings(), multiFieldsBuilder.build(this, context), copyTo);

return new TextFieldMapper(
name, fieldType, defaultFieldType, positionIncrementGap, includeInAll,
context.indexSettings(), multiFieldsBuilder.build(this, context), copyTo);