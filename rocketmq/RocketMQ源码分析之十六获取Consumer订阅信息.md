> 上一章中，我们分析了 RocketMQ Consumer、Producer 心跳处理的原理。本章节，我们将分析通过命令来获取 Consumer 连接信息的源码逻辑。

继前几篇文章：

[RocketMQ源码分析十四之心跳处理.md](RocketMQ源码分析之十四心跳处理.md)

[RocketMQ源码分析十五之获取Producer连接信息.md](RocketMQ源码分析之十五获取Producer连接信息.md)



# 命令参数

> 用法：sh mqadmin consumerConnection -n 192.168.1.100:9876
>
> 指令：consumerConnection
>
> 代码入口：
> org.apache.rocketmq.tools.command.connection.ConsumerConnectionSubCommand

| 参数 | 是否必填 | 说明                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| -c   | 是       | cluster 名称，表示topic 建在该集群（集群可通过clusterList 查询） |
| -h   | 否       | 打印帮助                                                     |
| -n   | 是       | NameServer 服务地址列表，格式ip:port;ip:port;…               |
| -g   | 是       | Consumer group的名字                                         |



# 解析命令行参数入口

```java
        // RocketMQ 配置了 命令行的执行 shell 脚本入口。就是下面的 mqadmin.sh 这个文件mqadmin.sh
        //解析命令行入口org.apache.rocketmq.tools.command.MQAdminStartup#main0
        //设置 namesrvAddr 为全局变量。
        if (commandLine.hasOption('n')) {
            String namesrvAddr = commandLine.getOptionValue('n');
            System.setProperty(MixAll.NAMESRV_ADDR_PROPERTY, namesrvAddr);
        }
```



# 运维端处理整体流程

```java
// 构建重试队列 Topic
        String topic = MixAll.getRetryTopic(consumerGroup);
// 获取 broker 地址
        List<BrokerData> brokers = this.examineTopicRouteInfo(topic).getBrokerDatas();
// 随机获取master 地址
        String addr = brokerData.selectBrokerAddr();

```



# RequestCode

```java
// 获取 Consumer 连接信息
public static final int GET_CONSUMER_CONNECTION_LIST = 203;
```



# Broker 端处理

```java
        // Consumer 通过 HeartBeat 来上报心跳数据，Broker 缓存在内存中。
        ConsumerGroupInfo consumerGroupInfo = this.brokerController.getConsumerManager().getConsumerGroupInfo(requestHeader.getConsumerGroup());
```



# 数据结构

```java
    public class ConsumerConnection extends RemotingSerializable {
        private HashSet<Connection> connectionSet = new HashSet<Connection>();
        private ConcurrentMap<String/* Topic */, SubscriptionData> subscriptionTable = new ConcurrentHashMap<String, SubscriptionData>();
        private ConsumeType consumeType;
        private MessageModel messageModel;
        private ConsumeFromWhere consumeFromWhere;
    }

    public class SubscriptionData implements Comparable<SubscriptionData> {
        // 默认是订阅 该 Topic 下的 所有 tag   
        public final static String SUB_ALL = "*";
        // 是否是 过滤类的方式   
        private boolean classFilterMode = false;
        // Topic   
        private String topic;
        // tags   
        private String subString;
        // Tags 集合   
        private Set<String> tagsSet = new HashSet<String>();
        private Set<Integer> codeSet = new HashSet<Integer>();
        private long subVersion = System.currentTimeMillis();
        // 过滤的类型    
        private String expressionType = ExpressionType.TAG;
        /**
         * Java过滤类，通过专有的上传接口上传到Filter Server，可以单独部署,除非压力特别大
         */
        @JSONField(serialize = false)
        private String filterClassSource;
    }
```



# 为啥这样就可以了拿到 Consumer 连接信息呢？

> 由 Topic 可得到 Broker 地址，所以 Consumer 往此 broker 上发送心跳信息。进而就可以知道 此 Topic 关联的 Consumer 信息了

```java
    // 第一次消费消息时候，获取 此 Topic 的队列信息
    // org.apache.rocketmq.client.impl.factory.MQClientInstance#findConsumerIdList

    /**
     * 找到 consumerId 列表
     *
     * @param topic topic
     * @param group 消费组
     * @return
     */
    public List<String> findConsumerIdList(final String topic, final String group) {
        // 从缓存中获取 Topic     
        String brokerAddr = this.findBrokerAddrByTopic(topic);
        // 如果没有，更新 远程路由信息     
        if (null == brokerAddr) {
            this.updateTopicRouteInfoFromNameServer(topic);
            brokerAddr = this.findBrokerAddrByTopic(topic);
        }
        if (null != brokerAddr) {
            try {
                return this.mQClientAPIImpl.getConsumerIdListByGroup(brokerAddr, group, 3000);
            } catch (Exception e) {
                log.warn("getConsumerIdListByGroup exception, " + brokerAddr + " " + group, e);
            }
        }
        return null;
    }

    // 后续定时发送 心跳信息到 Broker 端
    //org.apache.rocketmq.client.impl.factory.MQClientInstance#prepareHeartbeatData
    private HeartbeatData prepareHeartbeatData() {
        // 初始化心跳数据包       
        HeartbeatData heartbeatData = new HeartbeatData();
        // clientID       
        heartbeatData.setClientID(this.clientId);
        // Consumer 心跳数据包        
        for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
            MQConsumerInner impl = entry.getValue();
            if (impl != null) {
                ConsumerData consumerData = new ConsumerData();
                consumerData.setGroupName(impl.groupName());
                consumerData.setConsumeType(impl.consumeType());
                consumerData.setMessageModel(impl.messageModel());
                consumerData.setConsumeFromWhere(impl.consumeFromWhere());
                consumerData.getSubscriptionDataSet().addAll(impl.subscriptions());
                consumerData.setUnitMode(impl.isUnitMode());
                heartbeatData.getConsumerDataSet().add(consumerData);
            }
        }        // 后面还有 Producer 的代码片段。就不贴出来了
    }
```



# 关于监控设计

- 开源的 RocketMQ 没有把 最近的心跳时间 打印出来，稍微增加排查问题的难度。
建议从 Broker端透传过来。因为有一些机房的网络问题，导致抖动。也可以通过<font color='green'>心跳监控大盘</font>看的出来的。



# 总结

- RocketMQ 通过 命令获取 Consumer 来获取连接信息，存在一定的<font color='green'>时间差</font>。
因为数据的来源依靠 Consumer 定时Heartbeat。
- 通过此命令定时订阅数据，注意在前端<font color='green'>对比标红</font>订阅关系不一致的数据。