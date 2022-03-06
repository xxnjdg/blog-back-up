---
title: elasticsearch官方文档mapping笔记
date: 2021-07-23 17:53:06
categories:
- spring cloud alibaba
tags:
- elasticsearch
---


# 参考文档 

https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html

映射是定义文档及其包含的字段如何存储和索引的过程。

dynamic mapping，动态映射，es自动构建mapping

explicit mapping，自定义映射

Elasticsearch 最重要的功能之一是它会尽量避开您的视线，让您尽快开始探索数据。

要索引文档，您不必先创建索引、定义映射类型和定义字段 — 您只需索引文档，索引、类型和字段将自动显示：


indexRequest.routing(request.param("routing"));
indexRequest.setPipeline(request.param("pipeline"));
indexRequest.source(request.requiredContent(), request.getXContentType());
indexRequest.timeout(request.paramAsTime("timeout", IndexRequest.DEFAULT_TIMEOUT));
indexRequest.setRefreshPolicy(request.param("refresh"));
indexRequest.version(RestActions.parseVersion(request));
indexRequest.versionType(VersionType.fromString(request.param("version_type"), indexRequest.versionType()));
indexRequest.setIfSeqNo(request.paramAsLong("if_seq_no", indexRequest.ifSeqNo()));
indexRequest.setIfPrimaryTerm(request.paramAsLong("if_primary_term", indexRequest.ifPrimaryTerm()));
indexRequest.setRequireAlias(request.paramAsBoolean(DocWriteRequest.REQUIRE_ALIAS, indexRequest.isRequireAlias()));
String sOpType = request.param("op_type");
String waitForActiveShards = request.param("wait_for_active_shards");
if (waitForActiveShards != null) {
indexRequest.waitForActiveShards(ActiveShardCount.parseString(waitForActiveShards));
}
if (sOpType != null) {
indexRequest.opType(sOpType);
}

CreateIndexRequest createIndexRequest = new CreateIndexRequest();
createIndexRequest.index(index);
createIndexRequest.cause("auto(bulk api)");
createIndexRequest.masterNodeTimeout(1m);
