> 在前面一章节中，我们分析 创建 ConsumerGroup 整体流程。
> 本章节，我们将分析 <font color='green'>删除 ConsumerGroup</font>

# 需要先预热前面的文章：

[RocketMQ源码分析七之创建ConsumerGroup.md](RocketMQ源码分析之七创建ConsumerGroup.md)

[RocketMQ源码分析三之删除Topic.md](RocketMQ源码分析之三删除Topic.md)

# 命令参数

> 用法：sh mqadmin deleteSubGroup -n 192.168.1.100:9876 -c shg
>
> 指令：deleteSubGroup
>
> 代码入口：
> org.apache.rocketmq.tools.command.consumer.DeleteSubscriptionGroupCommand

| 参数 | 是否必填 | 说明                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| -h   | 否       | 打印帮助                                                     |
| -n   | 是       | nameserver 服务地址列表，格式ip:port;ip:port;…                |
| -g   | 是       | 消费分组                                                     |
| -b   | 否       | 如果 -c 为 空，则必填 broker 地址，表示订阅组建立在该 broker 上 |
| -c   | 是       | 如果 -b 为空，则必填 cluster 名称，表示 topic 建在该集群上。（集群可通过 clusterList 命令来查询） |

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
// 从Broker删除订阅组
public static final int DELETE_SUBSCRIPTIONGROUP = 207;
// Namesrv 获取注册到Name Server的所有Broker集群信息
public static final int GET_BROKER_CLUSTER_INFO = 106;
// 从Namesrv删除Topic配置
public static final int DELETE_TOPIC_IN_NAMESRV = 216;
```

# 核心代码片段

```java
        Set<String> masterSet = CommandUtil.fetchMasterAddrByClusterName(adminExt, clusterName);
        // 从 broker 内存中，移除此 consumerGroup
        this.brokerController.getSubscriptionGroupManager().deleteSubscriptionGroupConfig(requestHeader.getGroupName());
        //删除了，需要立即持久化，因为创建或者更新都会立即持久化的，持久化的目的就是覆盖
        this.persist();
        // 删除%RETRY%GroupName 
        TopicDeleteTopicSubCommand.deleteTopic(adminExt, clusterName, MixAll.RETRY_GROUP_TOPIC_PREFIX + groupName);
        // 删除死信队列，每个消费组都有一个死信队列的
        DeleteTopicSubCommand.deleteTopic(adminExt, clusterName, MixAll.DLQ_GROUP_TOPIC_PREFIX + groupName);
```

# 核心知识点

- 每个 RocketMQ 的 consumerGroup 都有两个特殊的 Topic：
  - %RETRY%${GroupName}：消息消费失败后，会进入此<font color='green'>重试队列</font>，
  之后重试的等级越来越高，每次消费失败的消息等待的时间越久。
  - %DLQ%${GroupName}：这个是重试消息消费到了一定次数后，就进入<font color='green'>死信队列</font>里面呆着，后续可以通过手工的方式重新消费。



# 总结

- RocketMQ 剔除消费分组，其实源码很简单。因为 RocketMQ 创建 <font color='green'>消费分组的数据也是保存在 Broker 端</font>的。