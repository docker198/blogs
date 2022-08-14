> 在本章中，我们将分析 RocketMQ 是如何 创建订阅组的。看看 RocketMQ 是如何实现的。



# 如何开启&关闭

在部署 RocketMQ 的 Broker 的时候，我们通常都会把
<font color='green'>autoCreateSubscriptionGroup</font> 设置为 false。一方面防止胡乱订阅，一方面也是为了后面的运维、统计。

# 命令参数

> 用法：sh mqadmin updateSubGroup -n 192.168.1.100:9876 -t shg
>
> 指令：updateSubGroup
>
> 代码入口：
> org.apache.rocketmq.tools.command.consumer.UpdateSubGroupSubCommand

| 参数 | 是否必填 | 说明                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| -h   | 否       | 打印帮助                                                     |
| -n   | 是       | nameserver 服务地址列表，格式ip:port;ip:port;…                |
| -b   | 否       | 如果 -c 为 空，则必填 broker 地址，表示订阅组建立在该 broker 上 |
| -c   | 否       | 如果 -b 为空，则必填 cluster 名称，表示 topic 建在该集群上。（集群可通过 clusterList 命令来查询） |
| -d   | 否       | 是否容许广播方式消费                                         |
| -g   | 是       | 订阅组名称                                                   |
| -i   | 否       | 从哪个broker 开始消费                                        |
| -m   | 否       | 是否容许从队列的最小位置开始消费，默认会设置为 false。       |
| -q   | 否       | 消费失败的消息放到一个重试队列，每个订阅组配置几个重试队列   |
| -r   | 否       | 重试消费最大次数，超过则投递到死信队列，不再投递，并报警     |
| -s   | 否       | 消费功能是否开启                                             |
| -w   | 否       | 发现消息堆积后，将Consumer的消费请求重定向到另外一台Slave机器 |
| -q   | 否       | 重试队列的数量。默认是 1个队列                               |
| -r   | 否       | 最大重试次数                                                 |
| -a   | 否       | 是否通知有消费者实例变化                                     |

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

# RequstCode

```java

// 更新或者创建订阅住
public static final int UPDATE_AND_CREATE_SUBSCRIPTIONGROUP = 200;
// Namesrv 获取注册到Name Server的所有Broker集群信息
public static final int GET_BROKER_CLUSTER_INFO = 106;
```

# 核心代码流程

> 只讨论集群下的。指定 broker 跟这个差不多

```java
// 从 Name Server 获取 此集群下的 master 节点
Set<String> masterSet = CommandUtil.fetchMasterAddrByClusterName(defaultMQAdminExt, clusterName);
// broker 端缓存配置
this.brokerController.getSubscriptionGroupManager().updateSubscriptionGroupConfig(config);
// 需要立马持久化，防止断电等意外情况发生
this.persist();
```

# 核心数据结构

```java
        // 订阅组
        private final ConcurrentMap<String, SubscriptionGroupConfig> subscriptionGroupTable = new ConcurrentHashMap<String, SubscriptionGroupConfig>(1024);
        // 集群信息
        public class ClusterInfo extends RemotingSerializable {
            private HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
            private HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
        }
        // RocketMQ 的broker中的主从关系是通过 brokerName 来绑定的。
        //broker 信息
        public class BrokerData implements Comparable<BrokerData> {
            private String cluster;
            // broker 名字    
            private String brokerName;
            /**     * brokerId 为 0,表示该 broker 为 master     * broker address 这里究竟是什么？是：mq1101.jiandan.com:10911     */
            private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;
            private final Random random = new Random();
        }
```

# 哪里使用到订阅关系了呢

> org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest(io.netty.channel.Channel, org.apache.rocketmq.remoting.protocol.RemotingCommand, boolean)

``` java
        // 确保订阅组存在
        SubscriptionGroupConfig subscriptionGroupConfig = this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getConsumerGroup());
        if (null == subscriptionGroupConfig) {
            response.setCode(ResponseCode.SUBSCRIPTION_GROUP_NOT_EXIST);
            response.setRemark(String.format("subscription group [%s] does not exist, %s", requestHeader.getConsumerGroup(), FAQUrl.suggestTodo(FAQUrl.SUBSCRIPTION_GROUP_NOT_EXIST)));
            return response;
        }
```



# 总结

1. RocketMQ 的订阅关系就是<font color='green'>保存在 broker 并作持久化</font>，
等 Consumer 端消费消息的时候，校验一下。
2. 好几篇 RocketMQ 的源码，我都在一直强调 <font color='green'>RocketMQ数据结构</font>，
这个在 RocketMQ 很重要，把这些数据结构记住了，能方便我们理解 RocketMQ 更快。
3. RocketMQ 通过从 运维端来阅读代码。会更加简单。也让大家了解到 RocketMQ 是多么的简单的。