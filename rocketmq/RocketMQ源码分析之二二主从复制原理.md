# RocketMQ源码分析至二三主从同步机制

> 本章节，我们将分析 RocketMQ 中的 Broker 主从同步机制
> 

### 概述

我们知道 RocketMQ 的 Broker 是分为 master 和 slave 。Master 主要用于处理 Producer、Consumer 的请求和存储数据。而Slave 更像是备份，
从 Master 同步数据保存至本地。为啥需要一个 slave 角色存在呢？下面我们展开来说：

- **Broker 服务服务高可用**。在分布式系统环境中，我们追求高可用、备份。假设在生产环境中，master 挂掉了或者服务 Hang 住了，那么，
我们可以通过 slave 来读取数据，减少 master 的压力，同时也能让 master 快速的恢复，不至于处于高压状态。
- **提高Broker 吞吐量**。如果 Consumer 从 Broker 拉取消息，Broker 端计算发现 与 commitLog 相差很大，则让其下一次直接请求到
 slave 拉取消息。这种也是类时 <font color='green'>82 原则</font>，master 处理热点数据（尽量保证数据在内存中），slave 处理冷却数据。

### Broker 同步模式

- 同步复制：生产者发送消息到 master 后，由 master notify slave ，等待 slave 写入后，再响应 Producer 是否写入成功。写入 RT 略有下降，常见于支付等场景。
    - 配置：<font color='green'>brokerRole = SYNC_MASTER</font>
    - 性能：这种性能略低，但是可靠性很强；可通过部署多台机器满足性能，或者使用  SSD，适用于金融，支付交易等场景。
- 异步复制：生产者发送消息到 master 后，就可以响应 client 了。后续由 slave 主动来复制同步最新消息。常见于写入 RT敏感或者吞吐量较大的系统。
    - 配置：<font color='green'>brokerRole = ASYNC_MASTER </font>
    - 性能：适用于大部分场景。

### RocketMQ 主从同步关系图谱

> RocketMQ 主从复制的代码结构不太好，有点绕，在这里，我先按照表格的方式整理出来。
> 

| 服务名 | 职责 |
| --- | --- |
| SlaveSynchronize | 从 Master 同步配置，同步 Topic、offset 、delayOffset、订阅关系。都是从 Broker 端同步数据的。 |
| HAService | HA服务，负责同步双写，异步复制功能 |
| HAService.GroupTransferService | 主从复制通知服务。这里是一个线程哦。 |
| HAService.HAClient | 读取 master 传过来的值，master & slave 沟通的桥梁，接受 master 推送过来的数据，并向 master 汇报消费进度 |
| HAService.AcceptSocketService | 接收新的Socket连接。HAConnection 将拥有 读 slave 上传的数据、往 slave 写入数据的能力。 |
| HAConnection | HA服务，Master用来向Slave <font color='green'>Push数据</font>，并接收Slave应答 |
| HAConnection.ReadSocketService | 读取Slave请求，一般为push ack.master 所拥有。得到 slave 的消费进度 |
| HAConnection.WriteSocketService | 向Slave写入数据,Slave 得到的 传输数据协议 <Phy Offset> <Body Size> <Body Data><br> |

### 同步配置入口（`SlaveSynchronize`）

> 启动入口：org.apache.rocketmq.broker.BrokerController#handleSlaveSynchronize
通过定时任务，同步的配置有：
>
>1、topic 信息 ；
>
>2、同步consumerOffset；
>
>3、同步 delayOffset；
>
>4、同步 subConfig 信息
>
> 

```java
private void handleSlaveSynchronize(BrokerRole role) {
        if (role == BrokerRole.SLAVE) {
            if (null != slaveSyncFuture) {
                slaveSyncFuture.cancel(false);
            }
            this.slaveSynchronize.setMasterAddr(null);
            slaveSyncFuture = this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    try {
                        BrokerController.this.slaveSynchronize.syncAll();
                    }
                    catch (Throwable e) {
                        log.error("ScheduledTask SlaveSynchronize syncAll error.", e);
                    }
                }
            }, 1000 * 3, 1000 * 10, TimeUnit.MILLISECONDS);
        } else {
            //handle the slave synchronise
            if (null != slaveSyncFuture) {
                slaveSyncFuture.cancel(false);
            }
            this.slaveSynchronize.setMasterAddr(null);
        }
    }

/**
     * 在 broker initialize() 这个方法中，有启动。每隔 1 分钟，去 master 那边拉取数据
     */
    public void syncAll() {
        // 同步 Topic 信息
        this.syncTopicConfig();

        // 同步消费组的 offset
        this.syncConsumerOffset();

        // 同步延迟队列的 offset
        this.syncDelayOffset();

        // 同步订阅关系
        this.syncSubscriptionGroupConfig();
    }

```

### RequestCode

```java
// Broker 获取所有Topic的配置（Slave和Namesrv都会向Master请求此配置）
public static final int GET_ALL_TOPIC_CONFIG = 21;

// Broker 获取所有Consumer Offset
public static final int GET_ALL_CONSUMER_OFFSET = 43;

// Broker 获取所有延迟进度
public static final int GET_ALL_DELAY_OFFSET = 45;

// 获取所有的订阅关系
public static final int GET_ALL_SUBSCRIPTIONGROUP_CONFIG = 201;
```

### HA 服务启动入口

> org.apache.rocketmq.store.DefaultMessageStore#DefaultMessageStore
> 

```java

// 启动 HA 服务
this.haService = new HAService(this);

		/**
     * 这里构造函数中，几个变量。将在  HAService 中的 start() 中，进行 start()
     * @param defaultMessageStore
     * @throws IOException
     */
    public HAService(final DefaultMessageStore defaultMessageStore) throws IOException {
        this.defaultMessageStore = defaultMessageStore;

        // 接收新的Socket连接。HAConnection 将拥有 读 slave 上传的数据、往 slave 写入数据的能力
        this.acceptSocketService =
            new AcceptSocketService(defaultMessageStore.getMessageStoreConfig().getHaListenPort());

        this.groupTransferService = new GroupTransferService();

        // 这个才是重点，master & slave 沟通的桥梁.接受 master 推送过来的数据，并向 master 汇报消费进度
        this.haClient = new HAClient();
    }

```

### 同步双写 CommitLog

> 处理入口：org.apache.rocketmq.store.CommitLog#handleHA
> 

```java
// HA同步复制， 当msg写入master的commitlog文件后，
    // 判断maser的角色如果是同步双写SYNC_MASTER， 等待master同步到slave在返回结果
    // Synchronous write double 同步多写。也会写 slave 的。会等待 slave 的结果写入了
    public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        // 同步双写
        if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {

            HAService service = this.defaultMessageStore.getHaService();

            // 默认是 true
            if (messageExt.isWaitStoreMsgOK()) {
                // Determine whether to wait
                if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                    GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                    service.putRequest(request);

                    // 那个线程启动呢？groupTransferService 这个将被唤醒
                    service.getWaitNotifyObject().wakeupAll();

                    //wakeupCustomer 这个方法，将计算器的值 - 1 .同步刷盘超时时间 5s
                    boolean flushOK =
                        request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());

                    if (!flushOK) {
                        log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                            + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
                        putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                    }
                }
                // Slave problem
                else {
                    // Tell the producer, slave not available
                    putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
                }
            }
        }

    }
```

### 同步双写流程图

> 这里是同步双写的简单示意图，主要目的是想把 HA 这个模块中。为了方便理解，下面的图，我简化成了 同步的方式。
> 

![rocketmq](../images/rocketmq_22_01.png)
### 异步双写流程

> RocketMQ 异步同步数据，是 slave 定时汇报 offset，WriteSocketService 根据上一次的 offset 继续获取是否拿到 数据。异步的逻辑算是比较简单的。由 Master 的 WriteSocketService 循环去获取数据。
代码入口：org.apache.rocketmq.store.ha.HAService.HAClient#run
> 

```java
// 传输数据,
// selectResult会赋值给this.selectMapedBufferResult，出现异常也会清理掉
SelectMappedBufferResult selectResult =
HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
```

### 总结

- RocketMQ 复制的逻辑源码，确实是有点绕，但是我们需要记住几个类的关系。

### 附录
