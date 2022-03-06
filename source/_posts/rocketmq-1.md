---
title: namesrv笔记
date: 2021-02-15 15:27:27
categories:
- mq
tags:
- rocketmq
---

# org.apache.rocketmq.namesrv.NamesrvStartup

程序入口

## 定时任务

```text
//定时扫描不可用的broker，同时删除不可用的broker，同时打印相关日志
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        //定时将kv的配置信息输出到info日志中
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);
```

requestHeader.setBrokerAddr(brokerAddr);
requestHeader.setBrokerId(brokerId);
requestHeader.setBrokerName(brokerName);
requestHeader.setClusterName(clusterName);
requestHeader.setHaServerAddr(haServerAddr);
requestHeader.setCompressed(compressed);
requestHeader.setBodyCrc32(bodyCrc32);

requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
requestBody.setFilterServerList(filterServerList);



# Broker

## 定时任务

```text
//每30秒循环 Broker 发送心跳包
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
}, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

消息队列如何进行负载 ？
消息发送如何实现高可用 ？
批量消息发送如何实现一致性？
2 ）消息队列负载机制
消息生产者在发送消息时，如果本地路由表中未缓存 topic 的路由信息，向 NameServer 发送获取路由信息请求，更新本地路由信息表，并且消息生产者每隔 30s 从 NameServer 更新路由表 。
3 ）消息发送异常机制
消息发送高可用主要通过两个手段 ： 重试与 Broker 规避 。 Brok巳r 规避就是在一次消息
发送过程中发现错误，在某一时间段内，消息生产者不会选择该 Broker（消息服务器）上的
消息队列，提高发送消息的成功率 。
4 ）批量消息发送
RocketMQ 支持将 同一主题下 的多条消息一次性发送到消息服务端 。

SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
//消息发送组
requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
//消息的topic
requestHeader.setTopic(msg.getTopic());
//消息的默认topic
requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
//消息的默认queue数量
requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
//消息发送的queueid
requestHeader.setQueueId(mq.getQueueId());
//特殊标识
requestHeader.setSysFlag(sysFlag);
//消息创建的时间戳
requestHeader.setBornTimestamp(System.currentTimeMillis());
//标识
requestHeader.setFlag(msg.getFlag());
//消息的扩展属性
requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
//消息的消费
requestHeader.setReconsumeTimes(0);
//模型
requestHeader.setUnitMode(this.isUnitMode());
//批量
requestHeader.setBatch(msg instanceof MessageBatch);

response.setOpaque(request.getOpaque());
response.addExtField(MessageConst.PROPERTY_MSG_REGION, this.brokerController.getBrokerConfig().getRegionId());
response.addExtField(MessageConst.PROPERTY_TRACE_SWITCH, String.valueOf(this.brokerController.getBrokerConfig().isTraceOn()));
response.setCode(ResponseCode.SUCCESS);
response.setRemark(null);
responseHeader.setMsgId(putMessageResult.getAppendMessageResult().getMsgId());
responseHeader.setQueueId(queueIdInt);
responseHeader.setQueueOffset(putMessageResult.getAppendMessageResult().getLogicsOffset());

MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_CLUSTER, clusterName);

MessageAccessor.putProperty(msgExt, MessageConst.PROPERTY_RETRY_TOPIC, msgExt.getTopic());
this.putProperty(MessageConst.PROPERTY_WAIT_STORE_MSG_OK, false);
this.putProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL, 3);
putProperty(msg, MessageConst.PROPERTY_ORIGIN_MESSAGE_ID, msgExt.getMsgId());
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

1 ) TOTALSIZE ： 该消息条目总长度 ， 4 字节 。
2 ) MAGICCODE ： 魔数， 4 字节 。 固定值 Oxdaa320a7 。
3 ) BODYCRC ： 消息体 ere 校验码， 4 字节 。
4 ) QUEUEID ： 消息消费队列 ID , 4 字节 。
5 ) FLAG ： 消息 FLAG , RocketMQ 不做处理 ， 供应用程序使用，默认 4 字节 。
6 ) QUEUEOFFSET ：消息在消息消费队列的偏移量 ， 8 字节 。
7 ) PHYSICALOFFSET ： 消息在 CommitLog 文件中的偏移量 ， 8 字节 。
8 ) SYSFLAG ： 消息系统 Flag ，例如是否压缩 、 是否是事务消息等 ， 4 字节 。
9 ) BORNTIMESTAMP ： 消息生产者调用消息发送 API 的时间戳， 8 字节 。
10 ) BORNHOST ：消息发送者 IP 、端 口 号， 8 字节 。
11 ) STORETIMESTAMP ： 消息存储时间戳， 8 字节 。
12 ) STOREHOSTADDRESS: Broker 服务器 IP＋ 端 口 号， 8 字节 。
13 ） 阻CONSUMETIMES ： 消息重试次数， 4 字节 。
14 ) Prepared Transaction Offset ： 事务消息物理偏移量 ， 8 字节 。
15 ) BodyLength ：消息体长度， 4 字节 。
16 ) Body ： 消息体内容，长度为 bodyLenth 中存储的值。
17 ) TopieLength ： 主题存储长度， 1 字节 ，表示主题名称不能超过 255 个字符 。
18) Topie ： 主题，长度为 TopieLength 中存储的值。
    19 ) PropertiesLength ： 消息属性长度 ， 2 字节 ， 表示消息属性长度不能超过 6 553 6 个
    字符 。
    20 ) Properties ： 消息属性，长度为 PropertiesLength 中存储的值 。


DLedgerMmapFileStore.AppendHook appendHook = (entry, buffer, bodyOffset) -> {
assert bodyOffset == DLedgerEntry.BODY_OFFSET;
buffer.position(buffer.position() + bodyOffset + MessageDecoder.PHY_POS_POSITION);
buffer.putLong(entry.getPos() + bodyOffset);
};
dLedgerFileStore.addAppendHook(appendHook);

# org.apache.rocketmq.common.namesrv.NamesrvConfig
<table>
    <tr>
        <td>配置项</td> 
        <td>key</td>
        <td>默认值</td> 
        <td>说明</td> 
    </tr>
    <tr>
    	<td>rocketmq 主目录</td>
    	<td>rocketmqHome</td>
    	<td>System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV))</td>
    	<td>可以通过 -Drocketmq.home.dir=path 或通过设置环境变量 ROCKETMQ_HOME 来配置 RocketMQ 的主目录</td>
    </tr>
    <tr>
    	<td>NameServer存储 KV 配置属性的持久化路径</td>
    	<td>kvConfigPath</td>
    	<td>System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json"</td>
    	<td>NameServer 存储 KV 配置属性的持久化路径</td>
    </tr>
    <tr>
    	<td>NameServer 默认配置文件路径</td>
    	<td>configStorePath</td>
    	<td>System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties"</td>
    	<td>NameServer 默认配置文件路径,不生效. NameServer 启动时如果要通过配置文件配置 NameServer 启动属性的话，请使用 -c 选项</td>
    </tr>
    <tr>
    	<td></td>
    	<td>productEnvName</td>
    	<td>center</td>
    	<td></td>
    </tr>
    <tr>
    	<td>集群测试</td>
    	<td>clusterTest</td>
    	<td>false</td>
    	<td>是否开启集群测试</td>
    </tr>
    <tr>
    	<td>顺序消息</td>
    	<td>orderMessageEnable</td>
    	<td>false</td>
    	<td>是否支持顺序消息，默认是不支持</td>
    </tr>
</table>

# org.apache.rocketmq.remoting.netty.NettyServerConfig

<table>
    <tr>
        <td>配置项</td> 
        <td>默认值类型</td>
        <td>默认值</td> 
        <td>说明</td> 
    </tr>
    <tr>
    	<td>listenPort</td>
    	<td>int</td>
    	<td>8888</td>
    	<td>服务端监听端口，NameServer 监昕端口，该值默认会被初始化为 9876</td>
    </tr>
    <tr>
    	<td>serverWorkerThreads</td>
    	<td>int</td>
    	<td>8</td>
    	<td> Netty 业务线程池线程个数</td>
    </tr>
    <tr>
    	<td>serverCallbackExecutorThreads</td>
    	<td>int</td>
    	<td>0</td>
    	<td>Netty public 任务线程池线程个数，Netty 网络设计，根据业务类型会创建不同的线程池，比如处理消息发送、消息消费、心跳检测等。如果该业务类型（RequestCode）未注册线程池，则由 public 线程池执行</td>
    </tr>
    <tr>
    	<td>serverSelectorThreads</td>
    	<td>int</td>
    	<td>3</td>
    	<td>IO线程池线程个数，主要是 NameServer 、 Broker 端解析请求、返回相应的线程个数，这类线程主要是处理网络请求的，解析请求包，然后转发到各个业务线程池完成具体的业务操作，然后将结果再返回调用方</td>
    </tr>
    <tr>
    	<td>serverOnewaySemaphoreValue</td>
    	<td>int</td>
    	<td>256</td>
    	<td>send oneway 消息请求井发度（ Broker 端参数）</td>
    </tr>
    <tr>
    	<td>serverAsyncSemaphoreValue</td>
    	<td>int</td>
    	<td>64</td>
    	<td>异步消息发送最大并发度（ Broker 端参数）</td>
    </tr>
    <tr>
    	<td>serverChannelMaxIdleTimeSeconds</td>
    	<td>int</td>
    	<td>120</td>
    	<td>网络连接最大空闲时间，默认120s。如果连接空闲时间超过该参数设置的值，连接将被关闭</td>
    </tr>
    <tr>
    	<td>serverSocketSndBufSize</td>
    	<td>int</td>
    	<td>65535</td>
    	<td>网络 socket 发送缓存区大小，默认64k</td>
    </tr>
    <tr>
    	<td>serverSocketRcvBufSize</td>
    	<td>int</td>
    	<td>65535</td>
    	<td>网络 socket 接收缓存区大小，默认64k</td>
    </tr>
    <tr>
    	<td>serverPooledByteBufAllocatorEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td>ByteBuffer 是否开启缓存，建议开启</td>
    </tr>
    <tr>
    	<td>useEpollNativeSelector</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td>是否启用 EpollIO 模型，Linux 环境建议开启</td>
    </tr>
</table>


<table>
    <tr>
    	<td>rocketmqHome</td>
    	<td>class java.lang.String</td>
    	<td>System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV))</td>
    	<td></td>
    </tr>
    <tr>
    	<td>namesrvAddr</td>
    	<td>class java.lang.String</td>
    	<td>System.getProperty(MixAll.NAMESRV_ADDR_PROPERTY, System.getenv(MixAll.NAMESRV_ADDR_ENV))</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerIP1</td>
    	<td>class java.lang.String</td>
    	<td>RemotingUtil.getLocalAddress()</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerIP2</td>
    	<td>class java.lang.String</td>
    	<td>RemotingUtil.getLocalAddress()</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerName</td>
    	<td>class java.lang.String</td>
    	<td>localHostName()</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerClusterName</td>
    	<td>class java.lang.String</td>
    	<td>DefaultCluster</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerId</td>
    	<td>long</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerPermission</td>
    	<td>int</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>defaultTopicQueueNums</td>
    	<td>int</td>
    	<td>8</td>
    	<td></td>
    </tr>
    <tr>
    	<td>autoCreateTopicEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>clusterTopicEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerTopicEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>autoCreateSubscriptionGroup</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>messageStorePlugIn</td>
    	<td>class java.lang.String</td>
    	<td></td>
    	<td></td>
    </tr>
    <tr>
    	<td>msgTraceTopicName</td>
    	<td>class java.lang.String</td>
    	<td>TopicValidator.RMQ_SYS_TRACE_TOPIC</td>
    	<td></td>
    </tr>
    <tr>
    	<td>traceTopicEnable</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>sendMessageThreadPoolNums</td>
    	<td>int</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>pullMessageThreadPoolNums</td>
    	<td>int</td>
    	<td>32</td>
    	<td></td>
    </tr>
    <tr>
    	<td>processReplyMessageThreadPoolNums</td>
    	<td>int</td>
    	<td>32</td>
    	<td></td>
    </tr>
    <tr>
    	<td>queryMessageThreadPoolNums</td>
    	<td>int</td>
    	<td>16</td>
    	<td></td>
    </tr>
    <tr>
    	<td>adminBrokerThreadPoolNums</td>
    	<td>int</td>
    	<td>16</td>
    	<td></td>
    </tr>
    <tr>
    	<td>clientManageThreadPoolNums</td>
    	<td>int</td>
    	<td>32</td>
    	<td></td>
    </tr>
    <tr>
    	<td>consumerManageThreadPoolNums</td>
    	<td>int</td>
    	<td>32</td>
    	<td></td>
    </tr>
    <tr>
    	<td>heartbeatThreadPoolNums</td>
    	<td>int</td>
    	<td>8</td>
    	<td></td>
    </tr>
    <tr>
    	<td>endTransactionThreadPoolNums</td>
    	<td>int</td>
    	<td>24</td>
    	<td></td>
    </tr>
    <tr>
    	<td>flushConsumerOffsetInterval</td>
    	<td>int</td>
    	<td>5000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>flushConsumerOffsetHistoryInterval</td>
    	<td>int</td>
    	<td>60000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>rejectTransactionMessage</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>fetchNamesrvAddrByAddressServer</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>sendThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>10000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>pullThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>100000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>replyThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>10000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>queryThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>20000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>clientManagerThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>1000000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>consumerManagerThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>1000000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>heartbeatThreadPoolQueueCapacity</td>
    	<td>int</td>
    	<td>50000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>endTransactionPoolQueueCapacity</td>
    	<td>int</td>
    	<td>100000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>filterServerNums</td>
    	<td>int</td>
    	<td>0</td>
    	<td></td>
    </tr>
    <tr>
    	<td>longPollingEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>shortPollingTimeMills</td>
    	<td>long</td>
    	<td>1000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>notifyConsumerIdsChangedEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>highSpeedMode</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>commercialEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>commercialTimerCount</td>
    	<td>int</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>commercialTransCount</td>
    	<td>int</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>commercialBigCount</td>
    	<td>int</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>commercialBaseCount</td>
    	<td>int</td>
    	<td>1</td>
    	<td></td>
    </tr>
    <tr>
    	<td>transferMsgByHeap</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>maxDelayTime</td>
    	<td>int</td>
    	<td>40</td>
    	<td></td>
    </tr>
    <tr>
    	<td>regionId</td>
    	<td>class java.lang.String</td>
    	<td>MixAll.DEFAULT_TRACE_REGION_ID</td>
    	<td></td>
    </tr>
    <tr>
    	<td>registerBrokerTimeoutMills</td>
    	<td>int</td>
    	<td>6000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>slaveReadEnable</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>disableConsumeIfConsumerReadSlowly</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>consumerFallbehindThreshold</td>
    	<td>long</td>
    	<td>17179869184</td>
    	<td></td>
    </tr>
    <tr>
    	<td>brokerFastFailureEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>waitTimeMillsInSendQueue</td>
    	<td>long</td>
    	<td>200</td>
    	<td></td>
    </tr>
    <tr>
    	<td>waitTimeMillsInPullQueue</td>
    	<td>long</td>
    	<td>5000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>waitTimeMillsInHeartbeatQueue</td>
    	<td>long</td>
    	<td>31000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>waitTimeMillsInTransactionQueue</td>
    	<td>long</td>
    	<td>3000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>startAcceptSendRequestTimeStamp</td>
    	<td>long</td>
    	<td>0</td>
    	<td></td>
    </tr>
    <tr>
    	<td>traceOn</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>enableCalcFilterBitMap</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>expectConsumerNumUseFilter</td>
    	<td>int</td>
    	<td>32</td>
    	<td></td>
    </tr>
    <tr>
    	<td>maxErrorRateOfBloomFilter</td>
    	<td>int</td>
    	<td>20</td>
    	<td></td>
    </tr>
    <tr>
    	<td>filterDataCleanTimeSpan</td>
    	<td>long</td>
    	<td>86400000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>filterSupportRetry</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>enablePropertyFilter</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>compressedRegister</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>forceRegister</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>registerNameServerPeriod</td>
    	<td>int</td>
    	<td>30000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>transactionTimeOut</td>
    	<td>long</td>
    	<td>6000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>transactionCheckMax</td>
    	<td>int</td>
    	<td>15</td>
    	<td></td>
    </tr>
    <tr>
    	<td>transactionCheckInterval</td>
    	<td>long</td>
    	<td>60000</td>
    	<td></td>
    </tr>
    <tr>
    	<td>aclEnable</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
    <tr>
    	<td>storeReplyMessageEnable</td>
    	<td>boolean</td>
    	<td>true</td>
    	<td></td>
    </tr>
    <tr>
    	<td>autoDeleteUnusedStats</td>
    	<td>boolean</td>
    	<td>false</td>
    	<td></td>
    </tr>
</table>