---
title: elasticsearch-3
date: 2021-06-20 22:34:24
categories:
- spring cloud alibaba
tags:
- elasticsearch
---

timeout = {TimeValue@8576} "30s"
masterNodeTimeout = {TimeValue@8463} "1m"

```java
public class A{
    public static void main(String[] args) {
        return channel ->
                client.index(indexRequest, new RestStatusToXContentListener<>(channel, r -> r.getLocation(indexRequest.routing())));
    }

    ActionListener<BulkResponse> wrapBulkResponse(ActionListener<Response> listener) {
        return ActionListener.wrap(bulkItemResponses -> {
            assert bulkItemResponses.getItems().length == 1 : "expected only one item in bulk request";
            BulkItemResponse bulkItemResponse = bulkItemResponses.getItems()[0];
            if (bulkItemResponse.isFailed() == false) {
                final DocWriteResponse response = bulkItemResponse.getResponse();
                listener.onResponse((Response) response);
            } else {
                listener.onFailure(bulkItemResponse.getFailure().getCause());
            }
        }, listener::onFailure);
    }
    
    void k(){
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
    }

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
    
    //开启线程
    
    void a(){
        createIndexService.createIndex(updateRequest, ActionListener.wrap(response ->
                        listener.onResponse(new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcked(), indexName)),
                listener::onFailure));


        listenerDJR = onlyCreateIndex(request, ActionListener.wrap(response -> {
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


        clusterService.submitStateUpdateTask("create-index [" + request.index() + "], cause [" + request.cause() + "]",
                new IndexCreationTask(logger, allocationService, request, listenerDJR, indicesService, aliasValidator, xContentRegistry, settings,
                        this::validate));

        
    }

    class UpdateTask extends BatchedTask {
        //SafeAckedClusterStateTaskListener.listener = IndexCreationTask
        final ClusterStateTaskListener listener;

        UpdateTask(Priority priority, String source, Object task, ClusterStateTaskListener listener,
                   ClusterStateTaskExecutor<?> executor) {
            super(priority, source, executor, task);
            this.listener = listener;
        }

        @Override
        public String describeTasks(List<? extends BatchedTask> tasks) {
            return ((ClusterStateTaskExecutor<Object>) batchingKey).describeTasks(
                    tasks.stream().map(BatchedTask::getTask).collect(Collectors.toList()));
        }
    }

    //线程池
    
    //ackListener = DelegetingAckListener

    private static class DelegetingAckListener implements Discovery.AckListener {

        ////ackedListener = (SafeAckedClusterStateTaskListener.listener = IndexCreationTask)
        //ackListeners.add(new AckCountDownListener(ackedListener, newClusterState.version(), newClusterState.nodes(),threadPool));
        private final List<Discovery.AckListener> listeners;

        private DelegetingAckListener(List<Discovery.AckListener> listeners) {
            this.listeners = listeners;
        }

        @Override
        public void onNodeAck(DiscoveryNode node, @Nullable Exception e) {
            for (Discovery.AckListener listener : listeners) {
                listener.onNodeAck(node, e);
            }
        }

        @Override
        public void onTimeout() {
            throw new UnsupportedOperationException("no timeout delegation");
        }
    }
    
    void aa(){
        Supplier<ThreadContext.StoredContext> storedContextSupplier = threadPool.getThreadContext().newRestorableContext(true);
        TransportResponseHandler<T> responseHandler = new ContextRestoreResponseHandler<>(storedContextSupplier, handler);
        clientHandlers.put(requestId, new RequestHolder<>(responseHandler, connection, action, timeoutHandler));
        FailedToCommitClusterStateException

        sendingController.getPublishResponseHandler().onResponse(node);sendingController.getPublishResponseHandler().onResponse(node);
    }
    
    
    void aaaa(){
        ShardStartedClusterStateTaskExecutor;
        clusterService.submitStateUpdateTask(
                "shard-started " + request,
                request,
                ClusterStateTaskConfig.build(Priority.URGENT),
                shardStartedClusterStateTaskExecutor,
                shardStartedClusterStateTaskExecutor);

        request = new StartRecoveryRequest(
                recoveryTarget.shardId(),
                recoveryTarget.indexShard().routingEntry().allocationId().getId(),
                recoveryTarget.sourceNode(),
                clusterService.localNode(),
                Store.MetadataSnapshot.EMPTY,
                recoveryTarget.state().getPrimary() = false,
                recoveryTarget.recoveryId(),
                SequenceNumbers.UNASSIGNED_SEQ_NO);
    }
}
```












return new Checkpoint(0, 0, 1, minSeqNo, maxSeqNo, -2, 1);
return new Checkpoint(offset, 0, 1, minSeqNo, maxSeqNo, -2, 1);



fields.add(new Field(NAME, id, fieldType));

public static SequenceIDFields emptySeqID() {
return new SequenceIDFields(new LongPoint(NAME, SequenceNumbers.UNASSIGNED_SEQ_NO),
new NumericDocValuesField(NAME, SequenceNumbers.UNASSIGNED_SEQ_NO),
new NumericDocValuesField(PRIMARY_TERM_NAME, 0));
context.seqID(seqID);
fields.add(seqID.seqNo);
fields.add(seqID.seqNoDocValue);
fields.add(seqID.primaryTerm);

fields.add(new StoredField(fieldType().name(), ref.bytes, ref.offset, ref.length));

Field field = new Field(fieldType().name(), value, fieldType());

final Field version = new NumericDocValuesField(NAME, -1L);

Field field = new Field(fieldType().name(), binaryValue, fieldType());
fields.add(field);

builder = new TextFieldMapper.Builder(currentFieldName)
.addMultiField(new KeywordFieldMapper.Builder("keyword").ignoreAbove(256));

super(client, action, new PutMappingRequest());

actions.register(PutMappingAction.INSTANCE, TransportPutMappingAction.class);

public void merge(IndexMetaData indexMetaData, MergeReason reason, boolean updateAllTypes) {
internalMerge(indexMetaData, reason, updateAllTypes, false);
}

class PutMappingExecutor implements ClusterStateTaskExecutor<PutMappingClusterStateUpdateRequest> {

```json
{
  "myintdex": {
    "properties": {
      "first": {
        "type": "text",
        "similarity": "bm25",
        "fields": {
          "first.keyword": {
            "type": "keyword",
            "similarity": "bm25"
          }
        }
      },
      "second": {
        "type": "text",
        "similarity": "bm25",
        "fields": {
          "second.keyword": {
            "type": "keyword",
            "similarity": "bm25"
          }
        }
      }
    }
  }
}
```

{ 
mytype:{

    }
}
{first:{}}
}



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
-Djava.io.tmpdir=C:\Users\ddd\AppData\Local\Temp\elasticsearch
-Xms512m 
-Xmx512m
-Delasticsearch
-Des.path.home=D:\a\7132\cluster\node1\elasticsearch-7.13.2
-Des.path.conf=D:\a\7132\cluster\node1\elasticsearch-7.13.2\config
-Des.distribution.flavor=default
-Des.distribution.type=zip
-Des.bundled_jdk=true
-cp D:\a\7132\cluster\node1\elasticsearch-7.13.2\lib\*


SecurityNetty4ServerTransport

SecurityNetty4HttpServerTransport

org.elasticsearch.transport.TransportService#doStart








