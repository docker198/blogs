> 上一章中。我们分析了 RocketMQ 获取 Topic 的命令过程。本章，我们开始分析 RocketMQ 更新 
><font color='green'>Topic 权限</font>的源码过程。

#继上一篇文章：

[RocketMQ源码分析二之更新Topic命令.md](RocketMQ源码分析二之更新Topic命令.md)

[RocketMQ源码分析四之获取Topic信息.md](RocketMQ源码分析四之获取Topic信息.md)

# 更新Topic权限用处

通常情况下，我们会做 <font color='green'>RocketMQ 集群拆分</font>。先把新的 Broker 集群都挂在 两个 Name server 集群上面。然后，我们再设置老 Broker 上面的 Topic 权限。就可以达到 集群拆分目的了。

# 命令参数

> 用法：sh mqadmin updateTopicPerm -n 192.168.1.100:9876 -t shg
>
> 指令：updateTopicPerm
>
> 代码入口：
> org.apache.rocketmq.tools.command.topic.UpdateTopicPermSubCommand

| 参数 | 是否必填 | 说明                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| -h   | 否       | 打印帮助                                                     |
| -n   | 是       | nameserve 服务地址列表，格式ip:port;ip:port;…                |
| -t   | 是       | Topic 名字                                                   |
| -b   | 否       | broker 地址，表示topic 建在该broker                          |
| -c   | 否       | cluster 名称，表示topic 建在该集群（集群可通过clusterList 查询） |
| -p   | 是       | 指定新topic 的权限限制；2:[读]; 4:[写]; 6:[可读可写]         |

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

# RequestCode

```java
// Broker 更新或者增加一个Topic
public static final int UPDATE_AND_CREATE_TOPIC = 17;
// Namesrv 根据Topic获取Broker Name、队列数(包含读队列与写队列)
public static final int GET_ROUTEINTO_BY_TOPIC = 105;
```

# 整体流程

``` java
// 第一步：从 Name Server 获取此 Topic 路由信息。
TopicRouteData topicRouteData = defaultMQAdminExt.examineTopicRouteInfo(topic);
// 第二步：获取此 Topic 的 Queue 信息
List<QueueData> queueDatas = topicRouteData.getQueueDatas();
// 第三步：设置Topic 的权限
topicConfig.setPerm(perm);
// 第四步：获取 此 Topic 分布在哪些 Broker 上
List<BrokerData> brokerDatas = topicRouteData.getBrokerDatas();
// 第四步：获取此集群的 master broker 地址
Set<String> masterSet = CommandUtil.fetchMasterAddrByClusterName(defaultMQAdminExt, clusterName);
// 第五步：更新 此 Broker 上的 此 Topic 权限
defaultMQAdminExt.createAndUpdateTopicConfig(brokerAddr, topicConfig);
```

# 总结

- RocketMQ 更新 Topic 权限，跟创建 Topic 的流程差不多。
- 其实，在创建 Topic 的时候，我们没有指定权限，<font color='green'>默认的权限是 可读&可写</font>。