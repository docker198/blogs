> 上一章中，我们分析了 RocketMQ Broker 端会定时上报数据到 Name Server 端。那么，
>本章节将详细分析如何获取注册到 Name Server 中 RocketMQ Broker 集群的详细信息。

继前几篇文章：

[RocketMQ源码分析十之NameServer.md](RocketMQ源码分析十之NameServer.md)


# 命令参数

> 用法：sh mqadmin clusterList -n 192.168.1.100:9876
>
> 指令：clusterList
>
> 代码入口：
> org.apache.rocketmq.tools.command.cluster.ClusterListSubCommand

| 参数 | 是否必填 | 说明                                            |
| ---- | -------- | ----------------------------------------------- |
|      |          |                                                 |
| -h   | 否       | 打印帮助                                        |
| -n   | 是       | Name Server 服务地址列表，格式ip:port;ip:port;… |
| -m   | 否       | 打印Broker 端更多的采集数据                     |
| -i   | 是       | 打印配置间隔。单位 秒                           |



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

# RocketMQ 运维端处理流程

> 这里只分析打印更多详细信息的！
>
> 只摘抄核心代码片段！

```java
        // 获取注册在 Name Server 上的 集群信息
        ClusterInfo clusterInfoSerializeWrapper = defaultMQAdminExt.examineBrokerClusterInfo();
        // 根据 Broker 信息，获取 Broker 上面统计的数据，是 MAP 结构
        KVTable kvTable = defaultMQAdminExt.fetchBrokerRuntimeStats(brokerAddr);
```



# RequestCode

```java
// 获取注册到Name Server的所有Broker集群信息
public static final int GET_BROKER_CLUSTER_INFO = 106;
// Broker 获取Broker运行时信息
public static final int GET_BROKER_RUNTIME_INFO = 28;
```



# Name Server 端处理

```java
    // 获取集群的配置
    this.namesrvController.getRouteInfoManager().getAllClusterInfo();


    /*** 代码入口：org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#getAllClusterInfo  
     *  得到集群的相关信息：broker 与 cluster 地址的信息 
     *  @return
     **/
    public byte[] getAllClusterInfo() {
        ClusterInfo clusterInfoSerializeWrapper = new ClusterInfo();
        clusterInfoSerializeWrapper.setBrokerAddrTable(this.brokerAddrTable);
        clusterInfoSerializeWrapper.setClusterAddrTable(this.clusterAddrTable);
        return clusterInfoSerializeWrapper.encode();
    }
```

# Name Server 端数据结构

```java
    // 没有很复杂的逻辑在里面，就简单的组合对象罢了
    public class ClusterInfo extends RemotingSerializable {
        private HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
        private HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    }
```

# Name Server 端处理Broker 定时注册

> 代码入口：
>
> org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager#registerBroker Name Server
>
> 核心类：
>
> org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager
> 下面也是摘抄代码块出来

```java
// Broker 定时往 Name Server 上面注册// Namesrv 注册一个Broker，数据都是持久化的，如果存在则覆盖配置
        public static final int REGISTER_BROKER = 103;

        RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().
                registerBroker(requestHeader.getClusterName(), requestHeader.getBrokerAddr(), 
                        requestHeader.getBrokerName(), requestHeader.getBrokerId(), requestHeader.getHaServerAddr(), 
                        registerBrokerBody.getTopicConfigSerializeWrapper(), registerBrokerBody.getFilterServerList(), ctx.channel());
```

# 获取 Broker 运行时统计信息

> 代码入口：
> org.apache.rocketmq.broker.processor.AdminBrokerProcessor#prepareRuntimeInfo
> 下面只摘抄核心代码片段

```java
        HashMap<String, String> runtimeInfo = this.brokerController.getMessageStore().getRuntimeInfo();
        runtimeInfo.put("msgPutTotalYesterdayMorning", String.valueOf(this.brokerController.getBrokerStats().getMsgPutTotalYesterdayMorning()));
        runtimeInfo.put("msgPutTotalTodayMorning", String.valueOf(this.brokerController.getBrokerStats().getMsgPutTotalTodayMorning()));
        runtimeInfo.put("msgPutTotalTodayNow", String.valueOf(this.brokerController.getBrokerStats().getMsgPutTotalTodayNow()));

```

# Broker 端运行数据采集

> Broker 运行时数据采集有两个方向：
> 1、实时获取
> 2、统计昨天的

```java
    // 统计昨天的
    BrokerController.this.getBrokerStats().record();

    // 这个类是管理 RocketMQ Broker 数据采集的。org.apache.rocketmq.store.stats.BrokerStatsManager
    public void incTopicPutNums(final String topic, int num, int times) {
        this.statsTable.get(TOPIC_PUT_NUMS).addValue(topic, num, times);
    }
```



# 关于监控设计

- <font color='green'>定时</font>去统计 RocketMQ 的 Broker stats 的数据，做好监控、告警。
防止在没有预警情况下，大流量进来，直接把 Broker 干挂了；同时提高了排查问题的效率。
- 如果有开发能力的话，建议还是通过 <font color='green'>Prometheus exporter</font> 把 RocketMQ 统计数据直接暴露出去。



# 总结

- 获取 RocketMQ 集群详细信息，先从 Name Sever 读取到集群的数据后，然后，再遍历每个 Broker 的 stats 模块。把数据打印出来。