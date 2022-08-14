# RocketMQ源码分析之二一定时消息机制

> 本章节，我们将分析 RocketMQ 定时消息的原理。RocketMQ 消息算是设计比较巧妙得，
>让我们一块来学习吧！
---

### 继前N篇文章：

[RocketMQ源码分析之十三延时消息原理分析.md](RocketMQ源码分析之十三延时消息原理分析.md)

[RocketMQ源码分析之十九Broker概述.md](RocketMQ源码分析之十九Broker概述.md)

---

### 定时消息概述

在RocketMQ中，定时消息指消息在构造的时候，指定具体的 <font color='green'>delayLevel</font>，消息到达 Broker 端，
并不会立即被消费者消费，
而是要<font color='green'>等到特定的时间</font>后，才能被消费。但是开源版本RocketMQ是不支持任意时间精度的。
如果想要支持 任意时间精度的，
可以考虑 Redis zset  方式。其实，这个也与当时的业务场景有关系；比如，张三下单后，30分钟提醒他支付或者取消订单。
RocketMQ 的定时消息的模块在 <font color='green'>ScheduleMessageService</font> 中。
---
### 指定定时消息规则

> RocketMQ 定时消息，是通过 <font color='green'>delayLevel</font> 来体现出来的。
>delayLevel 级别是在 Broker 部署的时候指定的。
>delayLevel 从 1 开始。比如 delayLevel=1，表示此消息延迟 1s 后才能消费，依次类推。
> 

```java
// 定时消息相关，延迟消费。重试消费相关
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";

// 延迟队列的 Topic
public static final String SCHEDULE_TOPIC = "SCHEDULE_TOPIC_XXXX";
```

---

### ScheduleMessageService

```java
public class ScheduleMessageService extends ConfigManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.STORE_LOGGER_NAME);

    // 延迟队列的 Topic
    public static final String SCHEDULE_TOPIC = "SCHEDULE_TOPIC_XXXX";
    // 首次延迟 1s
    private static final long FIRST_DELAY_TIME = 1000L;
    // 每次调度 100 ms
    private static final long DELAY_FOR_A_WHILE = 100L;
    // 10s 。如果出现错误了。则 10s 后再投递消费
    private static final long DELAY_FOR_A_PERIOD = 10000L;

    // 每个level对应的延时时间
    private final ConcurrentMap<Integer /* level */, Long/* delay timeMillis */> delayLevelTable =
        new ConcurrentHashMap<Integer, Long>(32);

    // 延时计算到了哪里，每个 level 的 offset，应该是每个 level 对应一个 消息队列
    private final ConcurrentMap<Integer /* level */, Long/* offset */> offsetTable =
        new ConcurrentHashMap<Integer, Long>(32);
    private final DefaultMessageStore defaultMessageStore;
    private final AtomicBoolean started = new AtomicBoolean(false);
    // 定时器,后台程序
    private Timer timer;
    // 存储顶层对象
    private MessageStore writeMessageStore;
    // 获取 messageDelayLevel 的最大值,就是配置延迟级别中的最多的那个
    private int maxDelayLevel;
}
```

---

### 发送端

> RocketMQ 发送定时消息有两个入口：
>
> 1、自己根据业务规则，来发送定时消息。

>2、消费失败后，RocketMQ 自动将消息发送到 定时队列里面，保证下一次继续消费。
> 

```java

// 根据业务规则，发送 定时消息。
public static void main(String[] args) throws MQClientException, InterruptedException {
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");

        producer.start();

        try {
            Message msg = new Message("TopicTest" /* Topic */,
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );

            msg.setDelayTimeLevel(5);

            SendResult sendResult = producer.send(msg);

            System.out.printf("%s%n", sendResult);
        } catch (Exception e) {
            e.printStackTrace();
            Thread.sleep(1000);
        }

    }

// -------------------------------------------------------- 分割线 ----------------------------------------------------------------------

// 指定消费失败，ConsumeConcurrentlyStatus.RECONSUME_LATER 表示消息重新发送到 Broker 再重新消费
public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer defaultMQPushConsumer = new DefaultMQPushConsumer("InstanceTest", "cidTest");
        defaultMQPushConsumer.setNamesrvAddr("127.0.0.1:9876");
        defaultMQPushConsumer.subscribe("topicTest", "*");
        defaultMQPushConsumer.registerMessageListener((MessageListenerConcurrently)(msgs, context) -> {
            msgs.stream().forEach((msg) -> {
                System.out.printf("Msg topic is:%s, MsgId is:%s, reconsumeTimes is:%s%n", msg.getTopic() , msg.getMsgId(), msg.getReconsumeTimes());
            });
						// 表示
            return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        });

        defaultMQPushConsumer.start();
}

// 针对 ConsumeConcurrentlyStatus.RECONSUME_LATER 的处理，这个方法会将消息重新发送到 Broker 处理一遍
org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService#processConsumeResult
```

---

### RequestCode

```java
// 消费失败后，发布重试消息
public static final int CONSUMER_SEND_MSG_BACK = 36;

// Broker 发送消息
public static final int SEND_MESSAGE = 10;
```
--- 

### Broker 端处理之消息转换

> 代码入口：
org.apache.rocketmq.store.CommitLog#putMessage

>将原来真实的 Topic、tag 等等属性转换为字符串，包装起来。
> 

```java
// 这个表达式有毒。如果是非事务类型，或者事务提交，都可以进来这里的。
        if (tranType == MessageSysFlag.TRANSACTION_NOT_TYPE
            || tranType == MessageSysFlag.TRANSACTION_COMMIT_TYPE) {
            // Delay Delivery
            // 剔除 delevel 属性 MessageAccessor.clearProperty(msgInner, MessageConst.PROPERTY_DELAY_TIME_LEVEL);
            if (msg.getDelayTimeLevel() > 0) {
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
}
```

--- 

### Broker 端处理之ConsumeQueue 构建

> 正常情况下，RocketMQ 的消息有  topic、tags，而<font color='green'>定时消息将最终的投递时间转换为 tags</font>。
>算是一种比较巧妙的设计。因为读取消息的时候，先根据 consumequeue 再来读取具体的消息。
> 

```java

// 构建 tagsCode。
tagsCode = this.defaultMessageStore.getScheduleMessageService().computeDeliverTimestamp(delayLevel,storeTimestamp);

/**
     * 根据延迟级别，判断真正投递的时间
     * @param delayLevel
     * @param storeTimestamp
     * @return
     */
    public long computeDeliverTimestamp(final int delayLevel, final long storeTimestamp) {
        Long time = this.delayLevelTable.get(delayLevel);
        if (time != null) {
            return time + storeTimestamp;
        }

        return storeTimestamp + 1000;
    }
```
--- 

### Broker 端之初始化定时消息模块

> 代码入口：org.apache.rocketmq.store.DefaultMessageStore#DefaultMessageStore
> 

```java
// 延迟队列服务
this.scheduleMessageService = new ScheduleMessageService(this);
```

--- 

### Broker 端之加载定时消息模块

> 代码入口：org.apache.rocketmq.store.DefaultMessageStore#load
> 

```java
// load 定时进度
// 这个步骤要放置到最前面，从CommitLog里Recover定时消息需要依赖加载的定时级别参数
// slave依赖scheduleMessageService做定时消息的恢复
if (null != scheduleMessageService) {
    result = result && this.scheduleMessageService.load();
}
```

--- 

### 定时模块加载 load

> 代码入口：
org.apache.rocketmq.store.schedule.ScheduleMessageService#load
> 

```java
/**
     * 延迟队列加载，以及延迟队列的时间进行转换
     * @return
     */
    public boolean load() {
        // 延迟队列等等文件相关的
        boolean result = super.load();
        // 延迟队列的级别转换
        result = result && this.parseDelayLevel();
        return result;
    }

/**
     * 延迟队列的级别与时间进行转换
     * messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h";
     * @return
     */
    public boolean parseDelayLevel() {
        HashMap<String, Long> timeUnitTable = new HashMap<String, Long>();
        timeUnitTable.put("s", 1000L);
        timeUnitTable.put("m", 1000L * 60);
        timeUnitTable.put("h", 1000L * 60 * 60);
        timeUnitTable.put("d", 1000L * 60 * 60 * 24);

        // 默认就已经配置啦
        String levelString = this.defaultMessageStore.getMessageStoreConfig().getMessageDelayLevel();
        try {
            String[] levelArray = levelString.split(" ");
            for (int i = 0; i < levelArray.length; i++) {
                String value = levelArray[i];
                // 单位
                String ch = value.substring(value.length() - 1);
                // 单位对应的值
                Long tu = timeUnitTable.get(ch);

                int level = i + 1;

                // 找到最大的级别。不用 配置文件进行赋值
                if (level > this.maxDelayLevel) {
                    this.maxDelayLevel = level;
                }
                long num = Long.parseLong(value.substring(0, value.length() - 1));
                // 延迟几秒。
                long delayTimeMillis = tu * num;
                this.delayLevelTable.put(level, delayTimeMillis);
            }
        } catch (Exception e) {
            log.error("parseDelayLevel exception", e);
            log.info("levelString String = {}", levelString);
            return false;
        }

        return true;
    }
```

--- 

### 定时消息模块启动

> delayLevelTable 是将 messageDelayLevel 这个配置内容转换为指定的 Map 结构，
>
>具体为：delayLevelTable.put(level, delayTimeMillis)
>
><font color='green'>每个 delayLevel 都是一个 Timer 来执行</font>。
> 

```java
public void start() {
        if (started.compareAndSet(false, true)) {
            this.timer = new Timer("ScheduleMessageTimerThread", true);
            for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
                Integer level = entry.getKey();
                Long timeDelay = entry.getValue();
                Long offset = this.offsetTable.get(level);
                if (null == offset) {
                    offset = 0L;
                }

                if (timeDelay != null) {
                    // 启动任务，每个 level 都作为一个 TimerTask 来进行调度
                    this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), FIRST_DELAY_TIME);
                }
            }

            // 定时将延时消息消费进度，进度刷盘持久化。
            this.timer.scheduleAtFixedRate(new TimerTask() {

                @Override
                public void run() {
                    try {
                        if (started.get()) ScheduleMessageService.this.persist();
                    } catch (Throwable e) {
                        log.error("scheduleAtFixedRate flush exception", e);
                    }
                }
            }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
        }
    }
```
--- 

### QueueId 与 DelayLevel 映射关系

```java
public static int queueId2DelayLevel(final int queueId) {
        return queueId + 1;
}

/**
  * 将 延迟等级转换为 queueId
  * @param delayLevel
  * @return
*/
public static int delayLevel2QueueId(final int delayLevel) {
    return delayLevel - 1;
}
```
---

### 定时消息之消息探测任务

> 代码入口：
org.apache.rocketmq.store.schedule.ScheduleMessageService.DeliverDelayedMessageTimerTask
> 

```java
// 第一步：
// 重试消息的队列。delayLevel 在启动的时候，就已经初始化了。创建了 consumequeue
ConsumeQueue cq =ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(SCHEDULE_TOPIC,delayLevel2QueueId(delayLevel));

// 第二步：根据 offset ，找到当前队列中所有有效的消息。如果没有找到，则
// 找到映射关系；（当前写的位置-请求的位置）这么多内容
SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);

// 第三步：获取任务这个定时队列里面的第一条消息的时间
ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
     long offsetPy = bufferCQ.getByteBuffer().getLong();
     int sizePy = bufferCQ.getByteBuffer().getInt();

      // computeDeliverTimestamp 参看这个方法
     long tagsCode = bufferCQ.getByteBuffer().getLong();
		// 队列里存储的tagsCode实际是一个时间点
    long now = System.currentTimeMillis();
    long deliverTimestamp = this.correctDeliverTimestamp(now, tagsCode);

		// 根据 offset 获取具体的消息
		MessageExt msgExt =ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(offsetPy, sizePy);

		// 转换消息为正常的Topic，tag，这里会把 delayLevel 这个属性剔除掉的
    MessageExtBrokerInner msgInner = this.messageTimeup(msgExt);
    // 按照正常逻辑来投递消息。之后可以构建 consumer queue、index
    // 这里的 msgId 会发生变化的
    PutMessageResult putMessageResult =ScheduleMessageService.this.writeMessageStore.putMessage(msgInner);

}

// 第四步：进入下一个周期的调度任务，并且更新 offset
nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);

// 100ms 在重新观察一下
ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset), DELAY_FOR_A_WHILE);

// 更新 offset，并定时持久化到磁盘中。
ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
				

// 如果没有找到有效消息，则进入下一个调度任务。
ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel,
                failScheduleOffset), DELAY_FOR_A_WHILE);
```

---

### 如果消息未到执行时间

```java
// 时候未到，继续定时，继续启动 Task，本 Task 将结束。
                                // 这里定时任务将向前偏移
                                /**
                                 * 1、如果这条消息的时间还没有到的话，那么，将为其启动一个定时任务，这里的性能损耗很大。
                                 *    那么，这次的 Task 将结束。
                                 * 2、基于第 1 点的成立，那么，定时任务的 offset 不断向前移动。
                                 */
ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset), countdown);
// delay Level 中的 offset 不断偏移。保证了消息不会重复
ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
```
---

### 定时任务之将定时消息转换为正常业务消息

> RocketMQ 定时消息在 Broker 接收到消息后，会将原有消息存入到
><font color='green'>PROPERTY_REAL_TOPIC、PROPERTY_REAL_QUEUE_ID </font> 这两个属性中。
重新消费了几次，还是不会变得
> 

```java
/**
         * 在延迟队列中，如果时间达到了，需要将 Message 转换成原先的。
         * 删除原先 的 delevel 。topic 是 %RETRY%+ConsumerGroup
         * @param msgExt
         * @return
         */
        private MessageExtBrokerInner messageTimeup(MessageExt msgExt) {
            MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
            msgInner.setBody(msgExt.getBody());
            msgInner.setFlag(msgExt.getFlag());
            MessageAccessor.setProperties(msgInner, msgExt.getProperties());

            TopicFilterType topicFilterType = MessageExt.parseTopicFilterType(msgInner.getSysFlag());
            // 将 tag hash 后
            long tagsCodeValue =
                MessageExtBrokerInner.tagsString2tagsCode(topicFilterType, msgInner.getTags());
            msgInner.setTagsCode(tagsCodeValue);
            msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));

            msgInner.setSysFlag(msgExt.getSysFlag());
            msgInner.setBornTimestamp(msgExt.getBornTimestamp());
            msgInner.setBornHost(msgExt.getBornHost());
            msgInner.setStoreHost(msgExt.getStoreHost());
						// 
            msgInner.setReconsumeTimes(msgExt.getReconsumeTimes());

            msgInner.setWaitStoreMsgOK(false);

            // 去掉延迟投递的 properties ,防止再次投递，
            MessageAccessor.clearProperty(msgInner, MessageConst.PROPERTY_DELAY_TIME_LEVEL);

            // 恢复Topic  msg.getTopic()
						// retry Topic；%RETRY%+ConsumerGroup
            msgInner.setTopic(msgInner.getProperty(MessageConst.PROPERTY_REAL_TOPIC));

            // 恢复QueueId msg.getQueueId()
            String queueIdStr = msgInner.getProperty(MessageConst.PROPERTY_REAL_QUEUE_ID);
            int queueId = Integer.parseInt(queueIdStr);
            msgInner.setQueueId(queueId);

            return msgInner;
        }
```
---

### 总结

- RocketMQ 定时消息，在 Broker 端将原有消息的 Topic，tag 替换为 延迟消息的Topic、
<font color='green'>延迟到达时间最为 topic</font>。
- RocketMQ 定时消息，在 Broker 端，模拟一个 consumer 从 consumequeue 里面拉取消息，判断消息是否该重新投递了。
- RocketMQ 定时消息，是按照 delayLevel 来投递到指定的consumequeue 里面的，
所以，针对一个 consume queue 来说，延时消息是绝对有序的。
- RocketMQ 定时消息，在 Producer 端，还是根据原来的 Topic 选择具体的 Broker 来投递消息的，
在 Broker 端再转换消息。所以，RocketMQ 的 <font color='green'>延时消息没有类似的负载均衡策略</font>。
- RocketMQ 定时消息，<font color='green'>如果消息量巨大，那么，可能会造成消息投递有点延迟</font>。
因为在消息到达时，需要根据 offset 去捞消息的内容，再转换，再投递。
- RocketMQ 定时消息，需要做好<font color='green'>消息量监控</font>。
防止业务大规模投递，也适当控制业务得定时消息得梯度，别太密集。