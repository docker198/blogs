

> 在很多RocketMQ 书籍中，基本没有介绍到 RocketMQ 批量发送消息的<font color='green'>底层逻辑</font>。但是这也是一个能再次跟 Kafka 较劲的一个功能！接下来，
>让我们看看 RocketMQ 是如何处理批量发送消息的。当前RocketMQ 版本是 4.6.0。
> 

## **1. 问题**

1. RocketMQ 是如何支持批量发送消息的？
2. RocketMQ 消费批量消息，如果失败了，是怎么处理的呢？

## **2. 例子**

```java
public class SimpleBatchProducer {
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("BatchProducerGroupName");
        producer.start();        
        // If you just send messages of no more than 1MiB at a time, it is easy to use batch        
        // Messages of the same batch should have: same topic, same waitStoreMsgOK and no schedule support
        String topic = "BatchTest";
        List<Message> messages = new ArrayList<>();
        messages.add(new Message(topic, "Tag", "OrderID001", "Hello world 0".getBytes()));
        messages.add(new Message(topic, "Tag", "OrderID002", "Hello world 1".getBytes()));
        messages.add(new Message(topic, "Tag", "OrderID003", "Hello world 2".getBytes()));
        producer.send(messages);
    }
}
```

## **3. Producer 端**

### **3.1 代码入口**

```java
    /**
     * 发送批量消息，broker 端分割。不支持重试。只能是 同一个 Topic
     *
     * @param msgs * @return * @throws MQClientException
     * @throws RemotingException    * @throws MQBrokerException
     * @throws InterruptedException
     */
    @Overridepublic
    SendResult send(Collection<Message> msgs) throws
            MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return this.defaultMQProducerImpl.send(batch(msgs));
    }
```

### **3.2 组装批量消息**

```java
private MessageBatch batch(Collection<Message> msgs) throws MQClientException {
        MessageBatch msgBatch;
        try {
            // 这里会做校验。请看【3.3 检测是否符合条件】            
            msgBatch = MessageBatch.generateFromList(msgs);
            for (Message message : msgBatch) {
                Validators.checkMessage(message, this);
                MessageClientIDSetter.setUniqID(message);
                message.setTopic(withNamespace(message.getTopic()));
            }
            msgBatch.setBody(msgBatch.encode());
        } catch (Exception e) {
            throw new MQClientException("Failed to initiate the MessageBatch", e);
        }
        msgBatch.setTopic(withNamespace(msgBatch.getTopic()));
        return msgBatch;
    }
```

### **3.3 检测是否符合条件**

```java
    public static MessageBatch generateFromList(Collection<Message> messages) {
        assert messages != null;
        assert messages.size() > 0;
        List<Message> messageList = new ArrayList<Message>(messages.size());
        Message first = null;
        for (Message message : messages) {
            // 不支持延迟消息            
            if (message.getDelayTimeLevel() > 0) {
                throw new UnsupportedOperationException("TimeDelayLevel is not supported for batching");
            }            // 不支持重试消息
            if (message.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                throw new UnsupportedOperationException("Retry Group is not supported for batching");
            }            // Topic 需要保持一致
            if (first == null) {
                first = message;
            } else {
                if (!first.getTopic().equals(message.getTopic())) {
                    throw new UnsupportedOperationException("The topic of the messages in one batch should be the same");
                }
                // 保存消息的语义一致：要么同步刷盘，要么异步刷盘。因为RocketMQ 是 APPEND 模式的。
                if (first.isWaitStoreMsgOK() != message.isWaitStoreMsgOK()) {
                    throw new UnsupportedOperationException("The waitStoreMsgOK of the messages in one batch should the same");
                }
            }
            messageList.add(message);
        }
        MessageBatch messageBatch = new MessageBatch(messageList);
        messageBatch.setTopic(first.getTopic());
        messageBatch.setWaitStoreMsgOK(first.isWaitStoreMsgOK());
        return messageBatch;
    }
```

## **4 协议层**

> org.apache.rocketmq.common.protocol.header.SendMessageRequestHeader。
>这个是 RocketMQ Producer 和 Broker 之间转换的协议。协议层，比之前增加了一个 batch 的字段，用来支持批量消息。
> 

```java
 @CFNotNull
    private String producerGroup;
    @CFNotNull
    private String topic;
    @CFNotNull
    private String defaultTopic;
    @CFNotNull
    private Integer defaultTopicQueueNums;
    @CFNotNull
    private Integer queueId;
    @CFNotNull
    private Integer sysFlag;
    @CFNotNull
    private Long bornTimestamp;
    @CFNotNull
    private Integer flag;
    @CFNullable
    private String properties;
    @CFNullable
    private Integer reconsumeTimes;
    @CFNullable
    private boolean unitMode = false;
    // 是否批量  
    @CFNullable
    private boolean batch = false;
    // 最大重试次数    
    private Integer maxReconsumeTimes;
```

## **5、Broker 端**

### **5.1 批量消息接收**

> 代码入口：org.apache.rocketmq.broker.processor.SendMessageProcessor#processRequest
> 

```java
    /**
     * 处理请求过程
     * 1、正常发送消息
     * 2、消费端消费不了的消息，发送到 broker 端，然后放进延时队列里面     *
     *
     * @param ctx
     * @param request
     * @return
     * @throws RemotingCommandException
     */
    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
        SendMessageContext mqtraceContext;
        switch (request.getCode()) {

            //  Consumer将处理不了的消息发回服务器         
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.consumerSendMsgBack(ctx, request);
            default:
                // 转换请求头部                
                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
                if (requestHeader == null) {
                    return null;
                }
                // 消息轨迹：记录到达 broker 的消息               
                mqtraceContext = buildMsgContext(ctx, requestHeader);
                this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
                // 构造相应头部，发送消息                
                RemotingCommand response;                
                //如果是批量消息，在 broker 端将消息切割。一条一条存放。返回多个 MessageId             
                if (requestHeader.isBatch()) {
                    response = this.sendBatchMessage(ctx, request, mqtraceContext, requestHeader);
                } else {
                    response = this.sendMessage(ctx, request, mqtraceContext, requestHeader);
                }
                // 消息轨迹：记录发送成功的消息              
                this.executeSendMessageHookAfter(response, mqtraceContext);
                return response;
        }
    }
```

### **5.2 存放批量消息**

> 代码入口：org.apache.rocketmq.broker.processor.SendMessageProcessor#sendBatchMessage。
>我只挑核心代码。大家可看到。RocketMQ 对于批量消息；都是认为一条消息来存储的。下面只摘抄周昂要的内容。

> 

```java
   // 设定重试次数
   messageExtBatch.setReconsumeTimes(requestHeader.getReconsumeTimes() == null ? 0 : requestHeader.getReconsumeTimes());
   String clusterName = this.brokerController.getBrokerConfig().getBrokerClusterName();
   MessageAccessor.putProperty(messageExtBatch, MessageConst.PROPERTY_CLUSTER, clusterName);
   PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessages(messageExtBatch);
   return handlePutMessageResult(putMessageResult, response, request, messageExtBatch, responseHeader, sendMessageContext, ctx, queueIdInt);
```

### 5.3 转换批量消息为 ByteBuffer

> 为后续的一条、一条写入消息，做好准备。
> 

```java
org.apache.rocketmq.store.CommitLog.MessageExtBatchEncoder#encode
```

### 5.4 append 批量消息

> 最后存放消息入口：org.apache.rocketmq.store.CommitLog.DefaultAppendMessageCallback#doAppend(long, java.nio.ByteBuffer, int,org.apache.rocketmq.common.message.MessageExtBatch)；
>通过下面的代码片段，可以看到，RocketMQ 针对批量消息，是拆开后，一条、一条存放的。后面就是刷盘策略了，留到下一期再讲。
> 

```java
   // 拼接 msgId
   if (msgIdBuilder.length() > 0) {
     msgIdBuilder.append(',').append(msgId);
   } else {
     msgIdBuilder.append(msgId);
   }
   // 返回了多个 MsgId。表示多条消息。
```

## **6、批量消息限制**

1. 同一个批次的消息。Topic 必须保持一致。为啥呢？
- 由于 RocketMQ 的 Topic 是可以任意分布在多个 Broker 上的。如果是相同 Broker，那么，就一次请求即可，不需要多次请求。
2. 同一个批次的消息。刷怕的策略保持一致。为啥呢？
- 因为 RocketMQ 消息是 APPEND 机制，没办法做到不同消息采取不同的刷盘策略。
5. 同一个批次的消息，不支持延迟消息。为啥呢？
- 因为延迟消息，在 RocketMQ 也是一种 Topic，分布在不同的 Broker 上。如果个别消息使用了延时消息，也是不行的。

## **7 总结**

### **7.1 问题回答：**

1. RocketMQ 是如何支持批量发送消息的？
- RocketMQ 在客户端讲多条消息统一转换为：MessageBatch ，在 Broker 统一拆开；一条一条的保存。
3. RocketMQ 消费批量消息，如果失败了，是怎么处理的呢？
- 由于 Broker 是拆开一条、一条保存的。所以，Consumer 端获取到之前Producer发送的批量消息啦！