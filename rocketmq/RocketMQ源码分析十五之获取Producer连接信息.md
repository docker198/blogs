> 上一章中，我们分析了 RocketMQ Consumer、Producer 心跳处理的原理。本章节，我们将分析通过命令来获取 Producer 连接信息的源码逻辑。

继前几篇文章：

[RocketMQ源码分析十四之心跳处理.md](RocketMQ源码分析十四之心跳处理.md)



# 命令参数

> 用法：sh mqadmin producerConnection -n 192.168.1.100:9876
>
> 指令：producerConnection
>
> 代码入口：
> org.apache.rocketmq.tools.command.connection.ProducerConnectionSubCommand

| 参数 | 是否必填 | 说明                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| -c   | 是       | cluster 名称，表示topic 建在该集群（集群可通过clusterList 查询） |
| -h   | 否       | 打印帮助                                                     |
| -n   | 是       | NameServer 服务地址列表，格式ip:port;ip:port;…               |
| -t   | 是       | Topic 名字                                                   |
| -g   | 是       | Producer group的名字                                         |



# 解析命令行参数入口

``` java
    // RocketMQ 配置了 命令行的执行 shell 脚本入口。就是下面的 mqadmin.sh 这个文件mqadmin.sh
    //解析命令行入口org.apache.rocketmq.tools.command.MQAdminStartup#main0
    //设置 namesrvAddr 为全局变量。
    if (commandLine.hasOption('n')) {
       String namesrvAddr = commandLine.getOptionValue('n');
       System.setProperty(MixAll.NAMESRV_ADDR_PROPERTY, namesrvAddr);
    }

```



# RocketMQ Admin端处理整体流程

```java
// 获取 Topic 路由信息
examineTopicRouteInfo(topic);
// 获取 broker 地址
List<BrokerData> brokers = this.examineTopicRouteInfo(topic).getBrokerDatas();
// 随机获取master 地址
String addr = brokerData.selectBrokerAddr();
```

# RequestCode

```java
// 获取 Producer 连接信息
public static final int GET_PRODUCER_CONNECTION_LIST = 204;
```

# Broker 端处理

```java
// Producer 通过 HeartBeat 来上报心跳数据，Broker 缓存在内存中。
this.brokerController.getProducerManager().getGroupChannelTable().get(requestHeader.getProducerGroup());

```

# 数据结构

```java
    public class ClientChannelInfo {

        // 进行通信的 channel   
        private final Channel channel;
        //  客户端ID    
        private final String clientId;
        private final LanguageCode language;
        // 客户端版本号   
        private final int version;
        // 最后更新时间设定为当前时间,用于 channel 是否处于活跃来判断   
        private volatile long lastUpdateTimestamp = System.currentTimeMillis();
    }

    public class ProducerManager {
        private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
        private static final long LOCK_TIMEOUT_MILLIS = 3000;
        private static final long CHANNEL_EXPIRED_TIMEOUT = 1000 * 120;
        private static final int GET_AVALIABLE_CHANNEL_RETRY_COUNT = 3;
        private final Lock groupChannelLock = new ReentrantLock();
        // 生产者 组的管理    
        private final HashMap<String /* group name */, HashMap<Channel, ClientChannelInfo>> groupChannelTable = new HashMap<String, HashMap<Channel, ClientChannelInfo>>();
        private final ConcurrentHashMap<String, Channel> clientChannelTable = new ConcurrentHashMap<>();
        private PositiveAtomicCounter positiveAtomicCounter = new PositiveAtomicCounter();
    }
```



# 为啥这样就可以了拿到 Producer 连接信息呢？

> 由 Topic 可得到 Broker 地址，所以 Producer 往此 broker 上发送心跳信息。
>进而就可以知道 此 Topic 关联的 Producer 信息了

```java
        // 第一次发送消息，先获取 Topic 路由信息// 比如：org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl#sendDefaultImpl
        //如果找不到，那么，将抛出 此 Topic 不存在
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        // 后续定时发送 心跳信息到 Broker 端org.apache.rocketmq.client.impl.factory.MQClientInstance#sendHeartbeatToAllBrokerWithLock
        sendHeartbeatToAllBrokerWithLock();
```



# 关于监控设计

- 开源的 RocketMQ 没有把 最近的心跳时间 打印出来，稍微增加排查问题的难度。
建议从 Broker端透传过来。比如有一些机房的网络问题，导致抖动，通过
<font color='green'>心跳监控大盘</font>展示出来。
- 应当扩展 Producer 的 心跳信息，增加 queueId+发送数量。因为在分布下环境下，
很容易出现数据倾斜，这个是很头疼的事情。



# 总结

- RocketMQ 通过 命令获取 Producer 来获取连接信息，存在一定的时间差。因为数据的来源依靠 Producer 定时Heartbeat。