> 前面几篇文章中，我们都涉及到 Topic 的创建流程、更新 Topic 权限等等。本章我们将要分析 获取Topic 所在的集群。



# 继上N篇文章：
[RocketMQ源码分析之一概念理解](RocketMQ源码分析之一概念理解.md)

[RocketMQ源码分析二之更新Topic命令.md](RocketMQ源码分析二之更新Topic命令.md)



# 这个命令用在哪里？

- 比如，我们需要做<font color='green'>集群切换</font>的时候，需要再次确认老集群的 Topic 是否还存在。
- 因为 RocketMQ 的 Name Server 可以挂在很多 集群。这个命令也是确认 Topic 在哪个集群。



# 命令参数

> 用法：sh mqadmin topicClusterList -n 192.168.1.100:9876 -t shg
>
> 指令：topicClusterList
>
> 代码入口：
> org.apache.rocketmq.tools.command.topic.TopicClusterSubCommand

| 参数 | 是否必填 | 说明                                          |
| ---- | -------- | --------------------------------------------- |
| -h   | 否       | 打印帮助                                      |
| -n   | 是       | nameserver 服务地址列表，格式ip:port;ip:port;… |
| -t   | 是       | Topic 名字                                    |

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

# RequestCode

```java
// Namesrv 获取注册到Name Server的所有Broker集群信息
public static final int GET_BROKER_CLUSTER_INFO = 106;
// Namesrv 根据Topic获取Broker Name、队列数(包含读队列与写队列)
public static final int GET_ROUTEINTO_BY_TOPIC = 105;

```

# 整体流程

```java
        // 第一步：从 Name Server 获取集群信息
        ClusterInfo clusterInfo = examineBrokerClusterInfo();
        // 第二步：从 Name Server 获取 Topic 在哪些 Broker 上面
        TopicRouteData topicRouteData = examineTopicRouteInfo(topic);
        // 第三步：上面两个信息去交集
        BrokerData brokerData = topicRouteData.getBrokerDatas().get(0);
        String brokerName = brokerData.getBrokerName();
        private HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
        Iterator<Map.Entry<String, Set<String>>> it = clusterInfo.getClusterAddrTable().entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<String, Set<String>> next = it.next();
            if (next.getValue().contains(brokerName)) {
                clusterSet.add(next.getKey());
            }
        }
```

# 核心数据结构

```java
    public class RouteInfoManager {
        private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
        private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
        private final ReadWriteLock lock = new ReentrantReadWriteLock();
        // topic & queue 的信息    
        private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
        //说明 master 与 slave 是通过 brokerName 进行配对    
        private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
        // 将 broker 按照 clusterName 分组    
        private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
        // 代表一个活的 broker 链接由最后更新时间，一个链接 channel，数据版本和 Ha 地址组成    
        //Broker 定时向 namesrv 注册并更新 BrokerLiveInfo 的时间戳    
        private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
        private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;

        // 获取所有的 cluster Info    
        public byte[] getAllClusterInfo() {
            ClusterInfo clusterInfoSerializeWrapper = new ClusterInfo();
            clusterInfoSerializeWrapper.setBrokerAddrTable(this.brokerAddrTable);
            clusterInfoSerializeWrapper.setClusterAddrTable(this.clusterAddrTable);
            return clusterInfoSerializeWrapper.encode();
        }
    }

    public class ClusterInfo extends RemotingSerializable {
        private HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
        private HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    }
```

# 总结

- RocketMQ 创建 Topic 的时候，在 Broker 留一份，Name Server 留一份，<font color='green'>主要是 Name Server 为主</font>。
- RoccketMQ Broker 启动后，会<font color='green'>定时<font>向 Name Server 注册心跳。
- 所以，获取 Topic 在哪个集群上，我们可以取交集即可。