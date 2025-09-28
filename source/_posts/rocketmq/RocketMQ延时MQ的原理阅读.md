---
title: RocketMQ延时MQ的原理阅读
date: 2025-09-19 11:44:41
tags: [Java, RocketMQ, 分布式, 消息队列, 架构, 源码] 
categories: [技术分享, 学习记录]
---
status: draft
createDate: 2025-09-19 11:44:41
endDate:

## 目录
- [RocketMQ 4.X的延时MQ现状](#rocketmq-4.x-now)<a id="rocketmq-4.x-now-title"></a>☑️
- [RocketMQ 4.X的延时MQ原理解析](#rocketmq-4.x-principle)<a id="rocketmq-4.x-principle-title"></a>☑️
- [RocketMQ 4.X的延时MQ源码解析](#rocketmq-4.x-code)<a id="rocketmq-4.x-code-title"></a>☑️
- [RocketMQ 5.X的延时MQ现状](#rocketmq-5.x-now)<a id="rocketmq-5.x-now-title"></a>☑️
- [RocketMQ 5.X的延时MQ原理解析](#rocketmq-5.x-principle)<a id="rocketmq-5.x-principle-title"></a>⬜
- [RocketMQ 5.X的延时MQ源码解析](#rocketmq-5.x-code)<a id="rocketmq-5.x-code-title"></a>⬜
- [写在最后](#end)<a id="end-title"></a>

## [RocketMQ 4.X的延时MQ现状](#rocketmq-4.x-now-title)<a id="rocketmq-4.x-now"></a> [官方文档](https://rocketmq.apache.org/zh/docs/4.x/producer/04message3/)
- 字段message.delayTimeLevel
- 枚举为数字1-18，只支持18个固定延时等级（1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h）

## [RocketMQ 4.X的延时MQ原理解析](#rocketmq-4.x-principle-title)<a id="rocketmq-4.x-principle"></a>
底层实现主要是基于Broke端，主要涉及以下组件：

**1. CommitLog 写入处**
当消息带有delayTimeLevel > 0时，在CommitLog#asyncPutMessage方法中会进行特殊处理(老版本是CommitLog#putMessage)
- 将原始的topic和queueId保存到消息属性中(REAL_TOPIC、REAL_QUEUE_ID)
- 修改消息的topic为系统预设的延迟topic(SCHEDULE_TOPIC_XXXX)
- 根据延迟级别计算出对应的queueId [ScheduleMessageService#delayLevel2QueueId](https://github.com/apache/rocketmq/blob/4.9.x/store/src/main/java/org/apache/rocketmq/store/schedule/ScheduleMessageService.java#L91)并投放到对应的queue中

**2. 定时调度服务ScheduleMessageService**
该服务负责扫描延迟队列中的消息，并在到期后将其重新投递到原始的topic
- 启动时为每一个延迟级别的queue创建一个定时任务
- 定期检查每个延迟队列的消息是否到期(上一步会计算当前时间，并根据延迟等级计算出到期时间，注意超出18会默认18)
- 到期后，恢复原始topic和queueId，重新构建消息并写入CommitLog，进入正常消费流程。

**3.关键源码**
- [org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncProcessRequest](#rocketmSendMessageProcessor#asyncProcessRequest)<a id="rocketmSendMessageProcessor#asyncProcessRequest-title"></a>
- [org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncSendMessage](#SendMessageProcessor#asyncSendMessage)<a id="SendMessageProcessor#asyncSendMessage-title"></a>
- [org.apache.rocketmq.store.CommitLog#asyncPutMessage](#CommitLog#asyncPutMessage)<a id="CommitLog#asyncPutMessage-title"></a>
- [org.apache.rocketmq.store.schedule.ScheduleMessageService#start](#ScheduleMessageService#start)<a id="ScheduleMessageService#start-title"></a>
- [org.apache.rocketmq.store.schedule.ScheduleMessageService.DeliverDelayedMessageTimerTask](#ScheduleMessageService.DeliverDelayedMessageTimerTask)<a id="ScheduleMessageService.DeliverDelayedMessageTimerTask-title"></a>

## [RocketMQ 4.X的延时MQ源码解析](#rocketmq-4.x-code-title)<a id="rocketmq-4.x-code"></a>
### [org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncProcessRequest](#rocketmSendMessageProcessor#asyncProcessRequest-title) <a id="rocketmSendMessageProcessor#asyncProcessRequest"></a>
首先是收到的消息先通过这个方法进行转发处理，主要是第17行代码提供入口。
``` Java
public CompletableFuture<RemotingCommand> asyncProcessRequest(ChannelHandlerContext ctx,
                                                                  RemotingCommand request) throws RemotingCommandException {
        final SendMessageContext mqtraceContext;
        switch (request.getCode()) {
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.asyncConsumerSendMsgBack(ctx, request);
            default:
                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
                if (requestHeader == null) {
                    return CompletableFuture.completedFuture(null);
                }
                mqtraceContext = buildMsgContext(ctx, requestHeader);
                this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
                if (requestHeader.isBatch()) {
                    return this.asyncSendBatchMessage(ctx, request, mqtraceContext, requestHeader);
                } else {
                    return this.asyncSendMessage(ctx, request, mqtraceContext, requestHeader);
                }
        }
    }
```
### [org.apache.rocketmq.broker.processor.SendMessageProcessor#asyncSendMessage](#SendMessageProcessor#asyncSendMessage-title) <a id="SendMessageProcessor#asyncSendMessage"></a>
这里主要的是关注第61行代码，进入到Strore，运行asyncPutMessage。
``` Java
private CompletableFuture<RemotingCommand> asyncSendMessage(ChannelHandlerContext ctx, RemotingCommand request,
                                                                SendMessageContext mqtraceContext,
                                                                SendMessageRequestHeader requestHeader) {
        final RemotingCommand response = preSend(ctx, request, requestHeader);
        final SendMessageResponseHeader responseHeader = (SendMessageResponseHeader)response.readCustomHeader();

        if (response.getCode() != -1) {
            return CompletableFuture.completedFuture(response);
        }

        final byte[] body = request.getBody();

        int queueIdInt = requestHeader.getQueueId();
        TopicConfig topicConfig = this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());

        if (queueIdInt < 0) {
            queueIdInt = randomQueueId(topicConfig.getWriteQueueNums());
        }

        MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
        msgInner.setTopic(requestHeader.getTopic());
        msgInner.setQueueId(queueIdInt);

        if (!handleRetryAndDLQ(requestHeader, response, request, msgInner, topicConfig)) {
            return CompletableFuture.completedFuture(response);
        }

        msgInner.setBody(body);
        msgInner.setFlag(requestHeader.getFlag());
        Map<String, String> origProps = MessageDecoder.string2messageProperties(requestHeader.getProperties());
        MessageAccessor.setProperties(msgInner, origProps);
        msgInner.setBornTimestamp(requestHeader.getBornTimestamp());
        msgInner.setBornHost(ctx.channel().remoteAddress());
        msgInner.setStoreHost(this.getStoreHost());
        msgInner.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
        String clusterName = this.brokerController.getBrokerConfig().getBrokerClusterName();
        MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_CLUSTER, clusterName);
        if (origProps.containsKey(MessageConst.PROPERTY_WAIT_STORE_MSG_OK)) {
            // There is no need to store "WAIT=true", remove it from propertiesString to save 9 bytes for each message.
            // It works for most case. In some cases msgInner.setPropertiesString invoked later and replace it.
            String waitStoreMsgOKValue = origProps.remove(MessageConst.PROPERTY_WAIT_STORE_MSG_OK);
            msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
            // Reput to properties, since msgInner.isWaitStoreMsgOK() will be invoked later
            origProps.put(MessageConst.PROPERTY_WAIT_STORE_MSG_OK, waitStoreMsgOKValue);
        } else {
            msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
        }

        CompletableFuture<PutMessageResult> putMessageResult = null;
        String transFlag = origProps.get(MessageConst.PROPERTY_TRANSACTION_PREPARED);
        if (Boolean.parseBoolean(transFlag)) {
            if (this.brokerController.getBrokerConfig().isRejectTransactionMessage()) {
                response.setCode(ResponseCode.NO_PERMISSION);
                response.setRemark(
                        "the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
                                + "] sending transaction message is forbidden");
                return CompletableFuture.completedFuture(response);
            }
            putMessageResult = this.brokerController.getTransactionalMessageService().asyncPrepareMessage(msgInner);
        } else {
            putMessageResult = this.brokerController.getMessageStore().asyncPutMessage(msgInner);
        }
        return handlePutMessageResultFuture(putMessageResult, response, request, msgInner, responseHeader, mqtraceContext, ctx, queueIdInt);
    }
```
### [org.apache.rocketmq.store.CommitLog#asyncPutMessage](#CommitLog#asyncPutMessage-title) <a id="CommitLog#asyncPutMessage"></a>
putMessage的过程中会进行判断，主要看第十八行开始，如果是延迟mq，则会将消息放入到topic=SCHEDULE_TOPIC_XXXX,queueId=delayLevel-1中
``` java
public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
        // Set the storage time
        msg.setStoreTimestamp(System.currentTimeMillis());
        // Set the message body BODY CRC (consider the most appropriate setting
        // on the client)
        msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
        // Back to Results
        AppendMessageResult result = null;

        StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();

        String topic = msg.getTopic();
//        int queueId msg.getQueueId();
        final int tranType = MessageSysFlag.getTransactionValue(msg.getSysFlag());
        if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
                || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
            // Delay Delivery
            if (msg.getDelayTimeLevel() > 0) {
                if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                    msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
                }

                topic = TopicValidator.RMQ_SYS_SCHEDULE_TOPIC;
                int queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

                // Backup real topic, queueId
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
                msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

                msg.setTopic(topic);
                msg.setQueueId(queueId);
            }
        }

        InetSocketAddress bornSocketAddress = (InetSocketAddress) msg.getBornHost();
        if (bornSocketAddress.getAddress() instanceof Inet6Address) {
            msg.setBornHostV6Flag();
        }

        InetSocketAddress storeSocketAddress = (InetSocketAddress) msg.getStoreHost();
        if (storeSocketAddress.getAddress() instanceof Inet6Address) {
            msg.setStoreHostAddressV6Flag();
        }

        PutMessageThreadLocal putMessageThreadLocal = this.putMessageThreadLocal.get();
        updateMaxMessageSize(putMessageThreadLocal);
        if (!multiDispatch.isMultiDispatchMsg(msg)) {
            PutMessageResult encodeResult = putMessageThreadLocal.getEncoder().encode(msg);
            if (encodeResult != null) {
                return CompletableFuture.completedFuture(encodeResult);
            }
            msg.setEncodedBuff(putMessageThreadLocal.getEncoder().getEncoderBuffer());
        }
        PutMessageContext putMessageContext = new PutMessageContext(generateKey(putMessageThreadLocal.getKeyBuilder(), msg));

        long elapsedTimeInLock = 0;
        MappedFile unlockMappedFile = null;

        putMessageLock.lock(); //spin or ReentrantLock ,depending on store config
        try {
            MappedFile mappedFile = this.mappedFileQueue.getLastMappedFile();
            long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
            this.beginTimeInLock = beginLockTimestamp;

            // Here settings are stored timestamp, in order to ensure an orderly
            // global
            msg.setStoreTimestamp(beginLockTimestamp);

            if (null == mappedFile || mappedFile.isFull()) {
                mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
            }
            if (null == mappedFile) {
                log.error("create mapped file1 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, null));
            }

            result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
            switch (result.getStatus()) {
                case PUT_OK:
                    break;
                case END_OF_FILE:
                    unlockMappedFile = mappedFile;
                    // Create a new file, re-write the message
                    mappedFile = this.mappedFileQueue.getLastMappedFile(0);
                    if (null == mappedFile) {
                        // XXX: warn and notify me
                        log.error("create mapped file2 error, topic: " + msg.getTopic() + " clientAddr: " + msg.getBornHostString());
                        return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.CREATE_MAPEDFILE_FAILED, result));
                    }
                    result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
                    break;
                case MESSAGE_SIZE_EXCEEDED:
                case PROPERTIES_SIZE_EXCEEDED:
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.MESSAGE_ILLEGAL, result));
                case UNKNOWN_ERROR:
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
                default:
                    return CompletableFuture.completedFuture(new PutMessageResult(PutMessageStatus.UNKNOWN_ERROR, result));
            }

            elapsedTimeInLock = this.defaultMessageStore.getSystemClock().now() - beginLockTimestamp;
        } finally {
            beginTimeInLock = 0;
            putMessageLock.unlock();
        }

        if (elapsedTimeInLock > 500) {
            log.warn("[NOTIFYME]putMessage in lock cost time(ms)={}, bodyLength={} AppendMessageResult={}", elapsedTimeInLock, msg.getBody().length, result);
        }

        if (null != unlockMappedFile && this.defaultMessageStore.getMessageStoreConfig().isWarmMapedFileEnable()) {
            this.defaultMessageStore.unlockMappedFile(unlockMappedFile);
        }

        PutMessageResult putMessageResult = new PutMessageResult(PutMessageStatus.PUT_OK, result);

        // Statistics
        storeStatsService.getSinglePutMessageTopicTimesTotal(msg.getTopic()).add(1);
        storeStatsService.getSinglePutMessageTopicSizeTotal(topic).add(result.getWroteBytes());

        CompletableFuture<PutMessageStatus> flushResultFuture = submitFlushRequest(result, msg);
        CompletableFuture<PutMessageStatus> replicaResultFuture = submitReplicaRequest(result, msg);
        return flushResultFuture.thenCombine(replicaResultFuture, (flushStatus, replicaStatus) -> {
            if (flushStatus != PutMessageStatus.PUT_OK) {
                putMessageResult.setPutMessageStatus(flushStatus);
            }
            if (replicaStatus != PutMessageStatus.PUT_OK) {
                putMessageResult.setPutMessageStatus(replicaStatus);
            }
            return putMessageResult;
        });
    }
```
### [org.apache.rocketmq.store.schedule.ScheduleMessageService#start](#ScheduleMessageService#start-title) <a id="ScheduleMessageService#start"></a>
这里主要是解释schedule对每一个delayLevel都启动了一个定时任务线程，执行频率根据不同delayLevel对应的deladeliveryTime决定。
``` java
public void start() {
        if (started.compareAndSet(false, true)) {
            this.load();
            this.deliverExecutorService = new ScheduledThreadPoolExecutor(this.maxDelayLevel, new ThreadFactoryImpl("ScheduleMessageTimerThread_"));
            if (this.enableAsyncDeliver) {
                this.handleExecutorService = new ScheduledThreadPoolExecutor(this.maxDelayLevel, new ThreadFactoryImpl("ScheduleMessageExecutorHandleThread_"));
            }
            for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
                Integer level = entry.getKey();
                Long timeDelay = entry.getValue();
                Long offset = this.offsetTable.get(level);
                if (null == offset) {
                    offset = 0L;
                }

                if (timeDelay != null) {
                    if (this.enableAsyncDeliver) {
                        this.handleExecutorService.schedule(new HandlePutResultTask(level), FIRST_DELAY_TIME, TimeUnit.MILLISECONDS);
                    }
                    this.deliverExecutorService.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME, TimeUnit.MILLISECONDS);
                }
            }

            this.deliverExecutorService.scheduleAtFixedRate(new Runnable() {

                @Override
                public void run() {
                    try {
                        if (started.get()) {
                            ScheduleMessageService.this.persist();
                        }
                    } catch (Throwable e) {
                        log.error("scheduleAtFixedRate flush exception", e);
                    }
                }
            }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval(), TimeUnit.MILLISECONDS);
        }
    }
```
### [org.apache.rocketmq.store.schedule.ScheduleMessageService.DeliverDelayedMessageTimerTask](#ScheduleMessageService.DeliverDelayedMessageTimerTask-title) <a id="ScheduleMessageService.DeliverDelayedMessageTimerTask"></a>
task主要的任务就是101行，调用org.apache.rocketmq.store.schedule.ScheduleMessageService#messageTimeup，将对象转换成MessageExtBrokerInner对象（并在此时将topic和queueId重置成原来的），然后调用org.apache.rocketmq.store.DefaultMessageStore#putMessage，将消息放入到commitLog中。
``` java
class DeliverDelayedMessageTimerTask implements Runnable {
        private final int delayLevel;
        private final long offset;

        public DeliverDelayedMessageTimerTask(int delayLevel, long offset) {
            this.delayLevel = delayLevel;
            this.offset = offset;
        }

        @Override
        public void run() {
            try {
                if (isStarted()) {
                    this.executeOnTimeup();
                }
            } catch (Exception e) {
                // XXX: warn and notify me
                log.error("ScheduleMessageService, executeOnTimeup exception", e);
                this.scheduleNextTimerTask(this.offset, DELAY_FOR_A_PERIOD);
            }
        }

        /**
         * @return
         */
        private long correctDeliverTimestamp(final long now, final long deliverTimestamp) {

            long result = deliverTimestamp;

            long maxTimestamp = now + ScheduleMessageService.this.delayLevelTable.get(this.delayLevel);
            if (deliverTimestamp > maxTimestamp) {
                result = now;
            }

            return result;
        }

        public void executeOnTimeup() {
            ConsumeQueue cq =
                ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(TopicValidator.RMQ_SYS_SCHEDULE_TOPIC,
                    delayLevel2QueueId(delayLevel));

            if (cq == null) {
                this.scheduleNextTimerTask(this.offset, DELAY_FOR_A_WHILE);
                return;
            }

            SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
            if (bufferCQ == null) {
                long resetOffset;
                if ((resetOffset = cq.getMinOffsetInQueue()) > this.offset) {
                    log.error("schedule CQ offset invalid. offset={}, cqMinOffset={}, queueId={}",
                        this.offset, resetOffset, cq.getQueueId());
                } else if ((resetOffset = cq.getMaxOffsetInQueue()) < this.offset) {
                    log.error("schedule CQ offset invalid. offset={}, cqMaxOffset={}, queueId={}",
                        this.offset, resetOffset, cq.getQueueId());
                } else {
                    resetOffset = this.offset;
                }

                this.scheduleNextTimerTask(resetOffset, DELAY_FOR_A_WHILE);
                return;
            }

            long nextOffset = this.offset;
            try {
                int i = 0;
                ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
                for (; i < bufferCQ.getSize() && isStarted(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                    long offsetPy = bufferCQ.getByteBuffer().getLong();
                    int sizePy = bufferCQ.getByteBuffer().getInt();
                    long tagsCode = bufferCQ.getByteBuffer().getLong();

                    if (cq.isExtAddr(tagsCode)) {
                        if (cq.getExt(tagsCode, cqExtUnit)) {
                            tagsCode = cqExtUnit.getTagsCode();
                        } else {
                            //can't find ext content.So re compute tags code.
                            log.error("[BUG] can't find consume queue extend file content!addr={}, offsetPy={}, sizePy={}",
                                tagsCode, offsetPy, sizePy);
                            long msgStoreTime = defaultMessageStore.getCommitLog().pickupStoreTimestamp(offsetPy, sizePy);
                            tagsCode = computeDeliverTimestamp(delayLevel, msgStoreTime);
                        }
                    }

                    long now = System.currentTimeMillis();
                    long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode);
                    nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

                    long countdown = deliverTimestamp - now;
                    if (countdown > 0) {
                        this.scheduleNextTimerTask(nextOffset, DELAY_FOR_A_WHILE);
                        return;
                    }

                    MessageExt msgExt = ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(offsetPy, sizePy);
                    if (msgExt == null) {
                        continue;
                    }

                    MessageExtBrokerInner msgInner = ScheduleMessageService.this.messageTimeup(msgExt);
                    if (TopicValidator.RMQ_SYS_TRANS_HALF_TOPIC.equals(msgInner.getTopic())) {
                        log.error("[BUG] the real topic of schedule msg is {}, discard the msg. msg={}",
                            msgInner.getTopic(), msgInner);
                        continue;
                    }

                    boolean deliverSuc;
                    if (ScheduleMessageService.this.enableAsyncDeliver) {
                        deliverSuc = this.asyncDeliver(msgInner, msgExt.getMsgId(), nextOffset, offsetPy, sizePy);
                    } else {
                        deliverSuc = this.syncDeliver(msgInner, msgExt.getMsgId(), nextOffset, offsetPy, sizePy);
                    }

                    if (!deliverSuc) {
                        this.scheduleNextTimerTask(nextOffset, DELAY_FOR_A_WHILE);
                        return;
                    }
                }

                nextOffset = this.offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
            } catch (Exception e) {
                log.error("ScheduleMessageService, messageTimeup execute error, offset = {}", nextOffset, e);
            } finally {
                bufferCQ.release();
            }

            this.scheduleNextTimerTask(nextOffset, DELAY_FOR_A_WHILE);
        }

        public void scheduleNextTimerTask(long offset, long delay) {
            ScheduleMessageService.this.deliverExecutorService.schedule(new DeliverDelayedMessageTimerTask(
                this.delayLevel, offset), delay, TimeUnit.MILLISECONDS);
        }

        private boolean syncDeliver(MessageExtBrokerInner msgInner, String msgId, long offset, long offsetPy,
            int sizePy) {
            PutResultProcess resultProcess = deliverMessage(msgInner, msgId, offset, offsetPy, sizePy, false);
            PutMessageResult result = resultProcess.get();
            boolean sendStatus = result != null && result.getPutMessageStatus() == PutMessageStatus.PUT_OK;
            if (sendStatus) {
                ScheduleMessageService.this.updateOffset(this.delayLevel, resultProcess.getNextOffset());
            }
            return sendStatus;
        }

        private boolean asyncDeliver(MessageExtBrokerInner msgInner, String msgId, long offset, long offsetPy,
            int sizePy) {
            Queue<PutResultProcess> processesQueue = ScheduleMessageService.this.deliverPendingTable.get(this.delayLevel);

            //Flow Control
            int currentPendingNum = processesQueue.size();
            int maxPendingLimit = ScheduleMessageService.this.defaultMessageStore.getMessageStoreConfig()
                .getScheduleAsyncDeliverMaxPendingLimit();
            if (currentPendingNum > maxPendingLimit) {
                log.warn("Asynchronous deliver triggers flow control, " +
                    "currentPendingNum={}, maxPendingLimit={}", currentPendingNum, maxPendingLimit);
                return false;
            }

            //Blocked
            PutResultProcess firstProcess = processesQueue.peek();
            if (firstProcess != null && firstProcess.need2Blocked()) {
                log.warn("Asynchronous deliver block. info={}", firstProcess.toString());
                return false;
            }

            PutResultProcess resultProcess = deliverMessage(msgInner, msgId, offset, offsetPy, sizePy, true);
            processesQueue.add(resultProcess);
            return true;
        }

        private PutResultProcess deliverMessage(MessageExtBrokerInner msgInner, String msgId, long offset,
            long offsetPy, int sizePy, boolean autoResend) {
            CompletableFuture<PutMessageResult> future =
                ScheduleMessageService.this.writeMessageStore.asyncPutMessage(msgInner);
            return new PutResultProcess()
                .setTopic(msgInner.getTopic())
                .setDelayLevel(this.delayLevel)
                .setOffset(offset)
                .setPhysicOffset(offsetPy)
                .setPhysicSize(sizePy)
                .setMsgId(msgId)
                .setAutoResend(autoResend)
                .setFuture(future)
                .thenProcess();
        }
    }
```
## [RocketMQ 5.X的延时MQ现状](#rocketmq-5.x-now-title)<a id="rocketmq-5.x-now"></a> [官方文档](https://rocketmq.apache.org/zh/docs/featureBehavior/02delaymessage/)
- 字段message.deliveryTimestamp
- 可以预定触发的时间戳，而不是延时时长
- 支持以格式为毫秒级unix时间戳（默认1000毫秒的精度也就是1秒）
- 支持最长24小时，不支持自定义修改，超过24小时延时不生效，服务端会立即投递
- 定时任务仅支持发送到MessageType为Delay的Topic中
## [RocketMQ 5.X的延时MQ原理解析](#rocketmq-5.x-principle-title)<a id="rocketmq-5.x-principle"></a>
## [RocketMQ 5.X的延时MQ源码解析](#rocketmq-5.x-code-title)<a id="rocketmq-5.x-code"></a>
## [写在最后](#end-title)<a id="end"></a>
&nbsp;&nbsp;&nbsp;&nbsp;在2025年09月18日，心血来潮在github上搜索rocketMQ的延迟mq，发现了2018年的一些点点滴滴。
&nbsp;&nbsp;&nbsp;&nbsp;有自己为了学习ai并作为毕设的仓库，有同学带我玩的hexo + github实现的简单个人博客。
&nbsp;&nbsp;&nbsp;&nbsp;在看完文档和代码之后百感交集，终于打算重新开始写博客。不知道这次还能坚持多久，但是既然自己能一口气减肥掉了43斤（截至目前），那么还有什么不能的呢？
&nbsp;&nbsp;&nbsp;&nbsp;加油吧自己，在没有目标的时候好好积累，厚积薄发。