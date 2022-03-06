---
title: elasticsearch数据写入
date: 2021-05-04 19:42:00
categories:
- spring cloud alibaba
tags:
- elasticsearch
---

https://elasticsearch.cn/article/13533

# 写入入口 RestIndexAction


<table>
    <tr>
        <td>参数</td> 
        <td>简介</td> 
   </tr>
    <tr>
        <td>version</td> 
        <td>设置文档版本号。主要用于实现乐观锁</td> 
   </tr>
    <tr>
        <td>version type</td> 
        <td>默认为internal，请求参数指定的版本号与存储的文档版本号相同则写入。其他可选值有external等类型，为external类型时，如果当前存储的文档版本号小于请求参数指定的版本号，则写入数据。version type主要控制版本号的比较机制，用于对文档进行并发更新操作时同步数据</td> 
   </tr>
    <tr>
        <td>op type</td> 
        <td>可设置为create，代表仅在文档不存在时才写入。如果文档已存在，则写请求将失败</td> 
   </tr>
    <tr>
        <td>routing</td> 
        <td>ES默认使用文档ID进行路由，指定routing可使用routing值进行路由</td> 
    </tr>
    <tr>
        <td>wait for active shards</td> 
        <td>用于控制写一致性，当指定数量的分片副本可用时才执行写入，否则重试直至超时。默认为1，主分片可用即执行写入</td> 
    </tr>
    <tr>
        <td>refresh</td> 
        <td>写入完毕后执行refresh,使其对搜索可见</td> 
    </tr>
    <tr>
        <td>timeout</td> 
        <td>请求超时时间，默认为1分钟</td> 
    </tr>
    <tr>
        <td>pipeline</td> 
        <td>指定事先创建好的pipeline名称</td> 
   </tr>
</table>


```java
public class RestIndexAction extends BaseRestHandler {
    public RestIndexAction(Settings settings, RestController controller) {
        super(settings);
        //1 写入文档，自动生成id, 注册 post 请求 /{index}/{type} 对应的处理函数 this
        controller.registerHandler(POST, "/{index}/{type}", this); // auto id creation
        controller.registerHandler(PUT, "/{index}/{type}/{id}", this);
        controller.registerHandler(POST, "/{index}/{type}/{id}", this);
        CreateHandler createHandler = new CreateHandler(settings);
        controller.registerHandler(PUT, "/{index}/{type}/{id}/_create", createHandler);
        controller.registerHandler(POST, "/{index}/{type}/{id}/_create", createHandler);
    }

    @Override
    public String getName() {
        return "document_index_action";
    }

    final class CreateHandler extends BaseRestHandler {
        protected CreateHandler(Settings settings) {
            super(settings);
        }

        @Override
        public String getName() {
            return "document_create_action";
        }

        @Override
        public RestChannelConsumer prepareRequest(RestRequest request, final NodeClient client) throws IOException {
            request.params().put("op_type", "create");
            return RestIndexAction.this.prepareRequest(request, client);
        }
    }

    //2 对应处理器方法执行
    @Override
    public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
        //解析各种请求参数,请求体
        IndexRequest indexRequest = new IndexRequest(request.param("index"), request.param("type"), request.param("id"));
        //默认id路由
        indexRequest.routing(request.param("routing"));
        indexRequest.parent(request.param("parent"));
        indexRequest.setPipeline(request.param("pipeline"));
        //请求体，即文档内容
        indexRequest.source(request.requiredContent(), request.getXContentType());
        //默认1分钟超时
        indexRequest.timeout(request.paramAsTime("timeout", IndexRequest.DEFAULT_TIMEOUT));
        //默认不刷新
        indexRequest.setRefreshPolicy(request.param("refresh"));
        //设置版本号，用于乐观锁
        indexRequest.version(RestActions.parseVersion(request));
        indexRequest.versionType(VersionType.fromString(request.param("version_type"), indexRequest.versionType()));
        String sOpType = request.param("op_type");
        String waitForActiveShards = request.param("wait_for_active_shards");
        if (waitForActiveShards != null) {
            indexRequest.waitForActiveShards(ActiveShardCount.parseString(waitForActiveShards));
        }
        if (sOpType != null) {
            indexRequest.opType(sOpType);
        }

        //3 执行这个方法
        return channel ->
                client.index(indexRequest, new RestStatusToXContentListener<>(channel, r -> r.getLocation(indexRequest.routing())));
    }
}
```

进入这个方法

## org.elasticsearch.action.bulk.TransportBulkAction#doExecute(org.elasticsearch.tasks.Task, org.elasticsearch.action.bulk.BulkRequest, org.elasticsearch.action.ActionListener<org.elasticsearch.action.bulk.BulkResponse>)

这个方法做了以下事情

### 1. ingest pipeline

查看该请求是否符合某个ingest pipeline的pattern, 如果符合则执行pipeline中的逻辑，一般是对文档进行各种预处理，如格式调整，增加字段等。如果当前节点没有ingest角色，则需要将请求转发给有ingest角色的节点执行。

### 2. 自动创建索引

判断索引是否存在，如果开启了自动创建则交给主节点自动创建，否则报错

### 3. 设置routing

获取请求URL或mapping中的_routing，如果没有则使用_id, 如果没有指定_id则ES会自动生成一个全局唯一ID。该_routing字段用于决定文档分配在索引的哪个shard上。

### 4. 构建BulkShardRequest

由于Bulk Request中包含多种(Index/Update/Delete)请求，这些请求分别需要到不同的shard上执行，因此协调节点，会将请求按照shard分开，同一个shard上的请求聚合到一起，构建BulkShardRequest

### 5. 将请求发送给primary shard

因为当前执行的是写操作，因此只能在primary上完成，所以需要把请求路由到primary shard所在节点

### 6. 等待primary shard返回

```java
public class TransportBulkAction extends HandledTransportAction<BulkRequest, BulkResponse> {
    @Override
    protected void doExecute(Task task, BulkRequest bulkRequest, ActionListener<BulkResponse> listener) {
        //1 处理pipeline请求
        if (bulkRequest.hasIndexRequestsWithPipelines()) {
            if (clusterService.localNode().isIngestNode()) {
                processBulkIndexIngestRequest(task, bulkRequest, listener);
            } else {
                ingestForwarder.forwardIngestRequest(BulkAction.INSTANCE, bulkRequest, listener);
            }
            return;
        }

        final long startTime = relativeTime();
        final AtomicArray<BulkItemResponse> responses = new AtomicArray<>(bulkRequest.requests.size());

        //默认为true
        if (needToCheck()) {
            // Attempt to create all the indices that we're going to need during the bulk before we start.
            // Step 1: collect all the indices in the request
            //获取索引名字
            final Set<String> indices = bulkRequest.requests.stream()
                    // delete requests should not attempt to create the index (if the index does not
                    // exists), unless an external versioning is used
                    .filter(request -> request.opType() != DocWriteRequest.OpType.DELETE
                            || request.versionType() == VersionType.EXTERNAL
                            || request.versionType() == VersionType.EXTERNAL_GTE)
                    .map(DocWriteRequest::index)
                    .collect(Collectors.toSet());
            /* Step 2: filter that to indices that don't exist and we can create. At the same time build a map of indices we can't create
             * that we'll use when we try to run the requests. */
            final Map<String, IndexNotFoundException> indicesThatCannotBeCreated = new HashMap<>();
            Set<String> autoCreateIndices = new HashSet<>();
            ClusterState state = clusterService.state();
            for (String index : indices) {
                boolean shouldAutoCreate;
                try {
                    //是否自动创建索引,从路由表查询索引是否存在，如果不存在那么就自动创建索引,路由表在集群状态中保存
                    shouldAutoCreate = shouldAutoCreate(index, state);
                } catch (IndexNotFoundException e) {
                    shouldAutoCreate = false;
                    indicesThatCannotBeCreated.put(index, e);
                }
                if (shouldAutoCreate) {
                    autoCreateIndices.add(index);
                }
            }
            // Step 3: create all the indices that are missing, if there are any missing. start the bulk after all the creates come back.
            if (autoCreateIndices.isEmpty()) {
                executeBulk(task, bulkRequest, startTime, listener, responses, indicesThatCannotBeCreated);
            } else {
                //需要自动创建索引
                final AtomicInteger counter = new AtomicInteger(autoCreateIndices.size());
                for (String index : autoCreateIndices) {
                    //2 这个方法就是创建索引
                    createIndex(index, bulkRequest.timeout(), new ActionListener<CreateIndexResponse>() {
                        @Override
                        public void onResponse(CreateIndexResponse result) {
                            if (counter.decrementAndGet() == 0) {
                                //创建完所有索引后执行
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
                }
            }
        } else {
            executeBulk(task, bulkRequest, startTime, listener, responses, emptyMap());
        }
    }

    void executeBulk(Task task, final BulkRequest bulkRequest, final long startTimeNanos, final ActionListener<BulkResponse> listener,
                     final AtomicArray<BulkItemResponse> responses, Map<String, IndexNotFoundException> indicesThatCannotBeCreated) {
        new BulkOperation(task, bulkRequest, listener, responses, startTimeNanos, indicesThatCannotBeCreated).run();
    }

    private final class BulkOperation extends AbstractRunnable {
        @Override
        protected void doRun() throws Exception {
            final ClusterState clusterState = observer.setAndGetObservedState();
            if (handleBlockExceptions(clusterState)) {
                return;
            }
            final ConcreteIndices concreteIndices = new ConcreteIndices(clusterState, indexNameExpressionResolver);
            MetaData metaData = clusterState.metaData();
            //遍历所有请求
            for (int i = 0; i < bulkRequest.requests.size(); i++) {
                DocWriteRequest docWriteRequest = bulkRequest.requests.get(i);
                //the request can only be null because we set it to null in the previous step, so it gets ignored
                if (docWriteRequest == null) {
                    continue;
                }
                //检查索引状态是否可用
                if (addFailureIfIndexIsUnavailable(docWriteRequest, i, concreteIndices, metaData)) {
                    continue;
                }
                Index concreteIndex = concreteIndices.resolveIfAbsent(docWriteRequest);
                try {
                    //opType 默认 INDEX
                    switch (docWriteRequest.opType()) {
                        case CREATE:
                        case INDEX:
                            IndexRequest indexRequest = (IndexRequest) docWriteRequest;
                            final IndexMetaData indexMetaData = metaData.index(concreteIndex);
                            MappingMetaData mappingMd = indexMetaData.mappingOrDefault(indexRequest.type());
                            Version indexCreated = indexMetaData.getCreationVersion();
                            //3 设置 routing，获取请求URL或mapping中的_routing，如果没有则使用_id
                            indexRequest.resolveRouting(metaData);
                            //如果id为空，生成id
                            indexRequest.process(indexCreated, mappingMd, concreteIndex.getName());
                            break;
                        case UPDATE:
                            TransportUpdateAction.resolveAndValidateRouting(metaData, concreteIndex.getName(), (UpdateRequest) docWriteRequest);
                            break;
                        case DELETE:
                            docWriteRequest.routing(metaData.resolveIndexRouting(docWriteRequest.parent(), docWriteRequest.routing(), docWriteRequest.index()));
                            // check if routing is required, if so, throw error if routing wasn't specified
                            if (docWriteRequest.routing() == null && metaData.routingRequired(concreteIndex.getName(), docWriteRequest.type())) {
                                throw new RoutingMissingException(concreteIndex.getName(), docWriteRequest.type(), docWriteRequest.id());
                            }
                            break;
                        default: throw new AssertionError("request type not supported: [" + docWriteRequest.opType() + "]");
                    }
                } catch (ElasticsearchParseException | IllegalArgumentException | RoutingMissingException e) {
                    BulkItemResponse.Failure failure = new BulkItemResponse.Failure(concreteIndex.getName(), docWriteRequest.type(), docWriteRequest.id(), e);
                    BulkItemResponse bulkItemResponse = new BulkItemResponse(i, docWriteRequest.opType(), failure);
                    responses.set(i, bulkItemResponse);
                    // make sure the request gets never processed again
                    bulkRequest.requests.set(i, null);
                }
            }

            // first, go over all the requests and create a ShardId -> Operations mapping
            Map<ShardId, List<BulkItemRequest>> requestsByShard = new HashMap<>();
            for (int i = 0; i < bulkRequest.requests.size(); i++) {
                DocWriteRequest request = bulkRequest.requests.get(i);
                if (request == null) {
                    continue;
                }
                //获取索引名
                String concreteIndex = concreteIndices.getConcreteIndex(request.index()).getName();
                //根据id 或 _routing 算出要写入到的分片id
                ShardId shardId = clusterService.operationRouting().indexShards(clusterState, concreteIndex, request.id(), request.routing()).shardId();
                List<BulkItemRequest> shardRequests = requestsByShard.computeIfAbsent(shardId, shard -> new ArrayList<>());
                //把同一分片的请求合并一起
                shardRequests.add(new BulkItemRequest(i, request));
            }

            if (requestsByShard.isEmpty()) {
                listener.onResponse(new BulkResponse(responses.toArray(new BulkItemResponse[responses.length()]), buildTookInMillis(startTimeNanos)));
                return;
            }

            final AtomicInteger counter = new AtomicInteger(requestsByShard.size());
            String nodeId = clusterService.localNode().getId();
            //遍历每个分片请求
            for (Map.Entry<ShardId, List<BulkItemRequest>> entry : requestsByShard.entrySet()) {
                final ShardId shardId = entry.getKey();
                final List<BulkItemRequest> requests = entry.getValue();
                //4 构建BulkShardRequest
                BulkShardRequest bulkShardRequest = new BulkShardRequest(shardId, bulkRequest.getRefreshPolicy(),
                        requests.toArray(new BulkItemRequest[requests.size()]));
                bulkShardRequest.waitForActiveShards(bulkRequest.waitForActiveShards());
                bulkShardRequest.timeout(bulkRequest.timeout());
                if (task != null) {
                    bulkShardRequest.setParentTask(nodeId, task.getId());
                }
                shardBulkAction.execute(bulkShardRequest, new ActionListener<BulkShardResponse>() {
                    @Override
                    public void onResponse(BulkShardResponse bulkShardResponse) {
                        for (BulkItemResponse bulkItemResponse : bulkShardResponse.getResponses()) {
                            // we may have no response if item failed
                            if (bulkItemResponse.getResponse() != null) {
                                bulkItemResponse.getResponse().setShardInfo(bulkShardResponse.getShardInfo());
                            }
                            responses.set(bulkItemResponse.getItemId(), bulkItemResponse);
                        }
                        if (counter.decrementAndGet() == 0) {
                            finishHim();
                        }
                    }

                    @Override
                    public void onFailure(Exception e) {
                        // create failures for all relevant requests
                        for (BulkItemRequest request : requests) {
                            final String indexName = concreteIndices.getConcreteIndex(request.index()).getName();
                            DocWriteRequest docWriteRequest = request.request();
                            responses.set(request.id(), new BulkItemResponse(request.id(), docWriteRequest.opType(),
                                    new BulkItemResponse.Failure(indexName, docWriteRequest.type(), docWriteRequest.id(), e)));
                        }
                        if (counter.decrementAndGet() == 0) {
                            finishHim();
                        }
                    }

                    private void finishHim() {
                        listener.onResponse(new BulkResponse(responses.toArray(new BulkItemResponse[responses.length()]), buildTookInMillis(startTimeNanos)));
                    }
                });
            }
        }
    }
}

```

调用 TransportShardBulkAction.doExecute

```java

public abstract class TransportReplicationAction<
        Request extends ReplicationRequest<Request>,
        ReplicaRequest extends ReplicationRequest<ReplicaRequest>,
        Response extends ReplicationResponse
        > extends TransportAction<Request, Response> {

    @Override
    protected void doExecute(Task task, Request request, ActionListener<Response> listener) {
        new ReroutePhase((ReplicationTask) task, request, listener).run();
    }

    final class ReroutePhase extends AbstractRunnable {
        private final ActionListener<Response> listener;
        private final Request request;
        private final ReplicationTask task;
        private final ClusterStateObserver observer;
        private final AtomicBoolean finished = new AtomicBoolean();

        @Override
        public void onFailure(Exception e) {
            finishWithUnexpectedFailure(e);
        }

        @Override
        protected void doRun() {
            setPhase(task, "routing");
            final ClusterState state = observer.setAndGetObservedState();
            if (handleBlockExceptions(state)) {
                return;
            }

            // request does not have a shardId yet, we need to pass the concrete index to resolve shardId
            final String concreteIndex = concreteIndex(state);
            final IndexMetaData indexMetaData = state.metaData().index(concreteIndex);
            if (indexMetaData == null) {
                retry(new IndexNotFoundException(concreteIndex));
                return;
            }
            if (indexMetaData.getState() == IndexMetaData.State.CLOSE) {
                throw new IndexClosedException(indexMetaData.getIndex());
            }

            // resolve all derived request fields, so we can route and apply it
            resolveRequest(indexMetaData, request);
            assert request.shardId() != null : "request shardId must be set in resolveRequest";
            assert request.waitForActiveShards() != ActiveShardCount.DEFAULT : "request waitForActiveShards must be set in resolveRequest";

            //根据请求分片id获取主分片
            final ShardRouting primary = primary(state);
            if (retryIfUnavailable(state, primary)) {
                return;
            }
            //获取主分片所在的节点
            final DiscoveryNode node = state.nodes().get(primary.currentNodeId());
            //5 因为当前执行的是写操作，因此只能在primary上完成，所以需要把请求路由到primary shard所在节点
            if (primary.currentNodeId().equals(state.nodes().getLocalNodeId())) {
                //主分片在当前节点
                performLocalAction(state, primary, node, indexMetaData);
            } else {
                //在其他节点
                performRemoteAction(state, primary, node);
            }
        }

        private void performLocalAction(ClusterState state, ShardRouting primary, DiscoveryNode node, IndexMetaData indexMetaData) {
            setPhase(task, "waiting_on_primary");
            if (logger.isTraceEnabled()) {
                logger.trace("send action [{}] to local primary [{}] for request [{}] with cluster state version [{}] to [{}] ",
                        transportPrimaryAction, request.shardId(), request, state.version(), primary.currentNodeId());
            }
            performAction(node, transportPrimaryAction, true,
                    new ConcreteShardRequest<>(request, primary.allocationId().getId(), indexMetaData.primaryTerm(primary.id())));
        }

        private void performAction(final DiscoveryNode node, final String action, final boolean isPrimaryAction,
                                   final TransportRequest requestToPerform) {
            transportService.sendRequest(node, action, requestToPerform, transportOptions, new TransportResponseHandler<Response>() {

                @Override
                public Response newInstance() {
                    return newResponseInstance();
                }

                @Override
                public String executor() {
                    return ThreadPool.Names.SAME;
                }

                @Override
                public void handleResponse(Response response) {
                    finishOnSuccess(response);
                }

                @Override
                public void handleException(TransportException exp) {
                    try {
                        // if we got disconnected from the node, or the node / shard is not in the right state (being closed)
                        final Throwable cause = exp.unwrapCause();
                        if (cause instanceof ConnectTransportException || cause instanceof NodeClosedException ||
                                (isPrimaryAction && retryPrimaryException(cause))) {
                            logger.trace(
                                    (org.apache.logging.log4j.util.Supplier<?>) () -> new ParameterizedMessage(
                                            "received an error from node [{}] for request [{}], scheduling a retry",
                                            node.getId(),
                                            requestToPerform),
                                    exp);
                            retry(exp);
                        } else {
                            finishAsFailed(exp);
                        }
                    } catch (Exception e) {
                        e.addSuppressed(exp);
                        finishWithUnexpectedFailure(e);
                    }
                }
            });
        }
    }
}

```
TransportShardBulkAction
## primary shard

Primary请求的入口是PrimaryOperationTransportHandler的MessageReceived, 当接收到请求时，执行的逻辑如下

### 1. 判断操作类型

遍历bulk请求中的各子请求，根据不同的操作类型跳转到不同的处理逻辑

### 2. 将update操作转换为Index和Delete操作

获取文档的当前内容，与update内容合并生成新文档，然后将update请求转换成index请求，此处文档设置一个version v1

### 3. Parse Doc

解析文档的各字段，并添加如_uid等ES相关的一些系统字段

### 4. 更新mapping

对于新增字段会根据dynamic mapping或dynamic template生成对应的mapping，如果mapping中有dynamic mapping相关设置则按设置处理，如忽略或抛出异常

### 5. 获取sequence Id和Version

从SequcenceNumberService获取一个sequenceID和Version。SequcenID用于初始化LocalCheckPoint， verion是根据当前Versoin+1用于防止并发写导致数据不一致。

### 6. 写入lucene

这一步开始会对文档uid加锁，然后判断uid对应的version v2和之前update转换时的versoin v1是否一致，不一致则返回第二步重新执行。 如果version一致，如果同id的doc已经存在，则调用lucene的updateDocument接口，如果是新文档则调用lucene的addDoucument. 这里有个问题，如何保证Delete-Then-Add的原子性，ES是通过在Delete之前会加上已refresh锁，禁止被refresh，只有等待Add完成后释放了Refresh Lock, 这样就保证了这个操作的原子性。

### 7. 写入translog

写入Lucene的Segment后，会以key value的形式写Translog， Key是Id, Value是Doc的内容。当查询的时候，如果请求的是GetDocById则可以直接根据_id从translog中获取。满足nosql场景的实时性。

### 8. 重构bulk request

因为primary shard已经将update操作转换为index操作或delete操作，因此要对之前的bulkrequest进行调整，只包含index或delete操作，不需要再进行update的处理操作。

### 9. flush translog

默认情况下，translog要在此处落盘完成，如果对可靠性要求不高，可以设置translog异步，那么translog的fsync将会异步执行，但是落盘前的数据有丢失风险。

### 10. 发送请求给replicas

将构造好的bulkrequest并发发送给各replicas，等待replica返回，这里需要等待所有的replicas返回，响应请求给协调节点。如果某个shard执行失败，则primary会给master发请求remove该shard。这里会同时把sequenceID， primaryTerm, GlobalCheckPoint等传递给replica。

### 11. 等待replica响应

```java
public abstract class TransportReplicationAction<
        Request extends ReplicationRequest<Request>,
        ReplicaRequest extends ReplicationRequest<ReplicaRequest>,
        Response extends ReplicationResponse
        > extends TransportAction<Request, Response> {
    protected class PrimaryOperationTransportHandler implements TransportRequestHandler<ConcreteShardRequest<Request>> {
        @Override
        public void messageReceived(ConcreteShardRequest<Request> request, TransportChannel channel, Task task) {
            new AsyncPrimaryAction(request.request, request.targetAllocationID, request.primaryTerm, channel, (ReplicationTask) task).run();
        }
    }

    class AsyncPrimaryAction extends AbstractRunnable implements ActionListener<PrimaryShardReference> {

        @Override
        protected void doRun() throws Exception {
            acquirePrimaryShardReference(request.shardId(), targetAllocationID, primaryTerm, this);
        }
        
    }
}
```


当所有的replica返回请求时，更细primary shard的LocalCheckPoint。

Defaults.FIELD_TYPE
FIELD_TYPE.setIndexOptions(IndexOptions.DOCS);
FIELD_TYPE.setTokenized(false);
FIELD_TYPE.setStored(true);
FIELD_TYPE.setOmitNorms(true);
FIELD_TYPE.setIndexAnalyzer(Lucene.KEYWORD_ANALYZER);
FIELD_TYPE.setSearchAnalyzer(Lucene.KEYWORD_ANALYZER);
FIELD_TYPE.setName(NAME);
FIELD_TYPE.freeze();
defaultFieldType.setIndexOptions(IndexOptions.NONE);
defaultFieldType.setStored(false);


org.apache.lucene.index.DefaultIndexingChain#processField


id

        context.seqID(seqID);
        fields.add(seqID.seqNo);
        fields.add(seqID.seqNoDocValue);
        fields.add(seqID.primaryTerm);

source
version
first json
Field field = new Field(fieldType().name(), binaryValue, fieldType());
fields.add(new SortedSetDocValuesField(fieldType().name(), binaryValue));

return new KeywordFieldMapper(
name, fieldType, defaultFieldType, ignoreAbove, includeInAll,
context.indexSettings(), multiFieldsBuilder.build(this, context), copyTo);

            return new TextFieldMapper(
                    name, fieldType, defaultFieldType, positionIncrementGap, includeInAll,
                    context.indexSettings(), multiFieldsBuilder.build(this, context), copyTo);

return new IndexingStrategy(true, false, true, seqNoForIndexing, 1, null);
return new IndexResult(plan.versionForIndexing, plan.seqNoForIndexing, plan.currentNotFoundOrDeleted);

out.writeVInt(SERIALIZATION_FORMAT);
out.writeString(id);
out.writeString(type);
out.writeBytesReference(source);
out.writeOptionalString(routing);
out.writeOptionalString(parent);
out.writeLong(version);

            out.writeByte(versionType.getValue());
            out.writeLong(autoGeneratedIdTimestamp);
            out.writeLong(seqNo);
            out.writeLong(primaryTerm);


BulkItemResponse primaryResponse = new BulkItemResponse(replicaRequest.id(), opType, response);
// set a blank ShardInfo so we can safely send it to the replicas. We won't use it in the real response though.
primaryResponse.getResponse().setShardInfo(new ShardInfo());
return primaryResponse;

return new WritePrimaryResult<>(request, response, location, null, primary, logger);









1
createIndexService.createIndex(updateRequest, ActionListener.wrap(response ->
listener.onResponse(new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcked(), indexName)),
listener::onFailure));

2
ActionListener.wrap(response -> {
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
}, listener::onFailure)



SafeAckedClusterStateTaskListener 任务


AckCountDownListener








