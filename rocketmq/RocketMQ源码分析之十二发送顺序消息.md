
> 本章节，我们主要分析 RocketMQ Producer 如何发送<font color='green'>顺序消息</font>及其内部原理。


[RocketMQ源码分析十一之发送消息原理.md](RocketMQ源码分析之十一发送消息原理.md)

# 例子

``` java
    public class Producer {
        public static void main(String[] args) {
            try {
                DefaultMQProducer producer = new DefaultMQProducer("PRODUCER_GROUP_PAY_SERVICE");
                producer.setNamesrvAddr("www.hebaodan.com");
                producer.start();
                String orderId = "2022072212581101";
                Message msg = new Message("Topic_ORDER_PAY", "webchat", "KEY#" + orderId, ("Hello RocketMQ ").getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                    // 选择具体的队列
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                        Integer id = (Integer) arg;
                        int index = id % mqs.size();
                        return mqs.get(index);
                    }
                }, orderId);
                producer.shutdown();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```



# 核心代码片段

> 下面的代码比较啰嗦，就是新增加了一个特性，根据 namespace 来获取指定的 queue，
>这种一般都是<font color='green'>资源隔离，比如机房隔离</font>。

``` java
        List<MessageQueue> messageQueueList = mQClientFactory.getMQAdminImpl().parsePublishMessageQueues(topicPublishInfo.getMessageQueueList());
        Message userMessage = MessageAccessor.cloneMessage(msg);
        String userTopic = NamespaceUtil.withoutNamespace(userMessage.getTopic(), mQClientFactory.getClientConfig().getNamespace());
        userMessage.setTopic(userTopic);
        mq = mQClientFactory.getClientConfig().queueWithNamespace(selector.select(messageQueueList, userMessage, arg));
```



# 与普通消息区别

1. 没有<font color='green'>重试次数</font>。普通的还是有重试的。
2. 没有<font color='green'>容错队列</font>。发送普通消息，是有熔断队列，可以选择的。
3. 其他的都跟发送普通消息一样的。



# 总结

- 发送顺序消息，有可能因为某个队列不可靠，导致消息不可用。<font color='green'>业务端需要自己保证可靠性</font>。
- 其实在发送普通消息的时候，也有选择队列的策略。也可以根据自定义策略来选择具体队列。
- RocketMQ 发送局部有序消息的时候，需要注意 broker 端的负载。<font color='green'>防止出现数据不均衡导致抖动</font>。