> 本章节，我们主要分析 RocketMQ Producer <font color='green'>发送延迟消息</font>。暂时不涉及到 RocketMQ 的存储端。
>让我们看看 RocketMQ 是如何支持延迟消息的吧！

继续前几篇文章：
[RocketMQ源码分析十一之发送消息原理.md](RocketMQ源码分析十一之发送消息原理.md)

# 例子

```java
    public class Producer {
        public static void main(String[] args) throws MQClientException, InterruptedException {
            // 设置业务上有意义的名称       
            DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
            producer.setNamesrvAddr("www.hebaodan.com");
            producer.start();
            try {
                Message msg = new Message("TopicTest", "TagA", "OrderID188", "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                // 设置延迟级别。
                msg.setDelayTimeLevel(5);
                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
            }
            producer.shutdown();
        }
    }
```



# 延时队列的设置

```java
// org/apache/rocketmq/store/config/MessageStoreConfig.java
//现在RocketMq并不支持任意时间的延时，需要设置几个固定的延时等级，从1s到2h分别对应着等级1到18
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
```

# Producer 端设置延时消息级别

> RocketMQ 的<font color='green'>延时消息</font>在 Producer 端设置非常巧妙，也与其协议设计有关系，大家可以学学。

``` java
    /*** org.apache.rocketmq.common.message.Message* 消息属性，都是系统属性，禁止应用设置，扩展字段，这样的设计非常巧妙*/
    private Map<String, String> properties;

    /**
     * 消息延时投递时间级别，0表示不延时，大于0表示特定延时级别（具体级别在服务器端定义）
     * * 客户端不支持任意时间段的
     * * 最高延迟级别 等于 队列数
     * * @param level
     */
    public void setDelayTimeLevel(int level) {
        this.putProperty(MessageConst.PROPERTY_DELAY_TIME_LEVEL, String.valueOf(level));
    }
```

# Broker 端转换延时消息

```java
        if (msg.getDelayTimeLevel() > 0) {
            // 如果超过了最大延时级别。那么，就设置为最大级别。  
            if (msg.getDelayTimeLevel() > this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel()) {
                msg.setDelayTimeLevel(this.defaultMessageStore.getScheduleMessageService().getMaxDelayLevel());
            }
            // 延迟队列的 Topic，SCHEDULE_TOPIC_XXXX    
            topic = ScheduleMessageService.SCHEDULE_TOPIC;
            // 根据延迟级别获取到执行的 queueId    
            queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());
            // Backup real topic, queueId ，保存原有消息的 topic ，queueId    
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
            MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
            // 这里存放了 tag、key,保存也有 Topic 的 tag,key    
            msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));
            msg.setTopic(topic);
            msg.setQueueId(queueId);
        }
```



# 总结

1. RocketMQ 的延迟级别，是放在 Message 这个对象 可扩展字段里面的。在 broker 端转换进行转化。
2. RocketMQ 延时消息，对于 <font color='green'>MessageBatch 不适用</font>。
3. RocketMQ 延时消息虽然是在 Broker 来校验的，但是还是建议在客户端再一次检验，看看是否跟 Broker 设置一致。
4. RocketMQ 的延迟消息，是每个 broker 端 <font color='green'>默认 18个 queue</font>,即18个级别。