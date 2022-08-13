> 上一章中，我们分析了 RocketMQ 的 NameServer 启动的过程，本章我们将分析如何通过命令来获取 Name Server 的配置。



继上一篇：
[RocketMQ源码分析十之NameServer.md](RocketMQ源码分析十之NameServer.md)


# 命令参数

> 用法：sh mqadmin getNamesrvConfig -n 192.168.1.100:9876
>
> 指令：getNamesrvConfig
>
> 代码入口：
> org.apache.rocketmq.tools.command.namesrv.GetNamesrvConfigCommand

| 参数 | 是否必填 | 说明                                                         |
| ---- | -------- | ------------------------------------------------------------ |
| -c   | 是       | cluster 名称，表示topic 建在该集群（集群可通过clusterList 查询） |
| -h   | 否       | 打印帮助                                                     |
| -n   | 是       | Name Server 服务地址列表，格式ip:port;ip:port;…              |

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

```java
    // 获取 Name Server 地址
    String servers = commandLine.getOptionValue('n');

    // 根据 Name Server 列表，分别从每个 Name Server 实例中获取配置。
    public Map<String, Properties> getNameServerConfig(final List<String> nameServers, long timeoutMillis) {
        RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.GET_NAMESRV_CONFIG, null);
        Map<String, Properties> configMap = new HashMap<String, Properties>(4);
        for (String nameServer : invokeNameServers) {
            RemotingCommand response = this.remotingClient.invokeSync(nameServer, request, timeoutMillis);
            assert response != null;
            if (ResponseCode.SUCCESS == response.getCode()) {
                configMap.put(nameServer, MixAll.string2Properties(new String(response.getBody(), MixAll.DEFAULT_CHARSET)));
            } else {
                throw new MQClientException(response.getCode(), response.getRemark());
            }
        }
        return configMap;
    }
```

# RequestCode

```java
// 从 Name Server 读取配置
public static final int GET_NAMESRV_CONFIG = 319;
```

# Name Server 端处理

```java
    // 直接合并配置
    private String getAllConfigsInternal() {
        StringBuilder stringBuilder = new StringBuilder();
        for (Object configObject : this.configObjectList) {
            Properties properties = MixAll.object2Properties(configObject);
            if (properties != null) {
                merge(properties, this.allConfigs);
            } else {
                log.warn("getAllConfigsInternal object2Properties is null, {}", configObject.getClass());
            }
        }
        // 读取所有配置。     
        {
            stringBuilder.append(MixAll.properties2String(this.allConfigs));
        }
        return stringBuilder.toString();
    }


    // configObjectList 数据来源于
    this.configuration = new Configuration(log, this.namesrvConfig, this.nettyServerConfig);
```



# 改造经验

- 这里只是把 Name Server 配置打印出来了。最好还是得要把 <font color='green'>JVM 配置打印出来</font>。
做到查看配置，都有一个统一的入口。
- 虽然打印出来了所有的配置了。最好还得要加上一个<font color='green'>对比的功能</font>。通过肉眼来发现配置不一致容易走眼。

# 总结

- 源码的逻辑比较简单。直接读取 Name Server 配置。然后直接展示出来。