# RocketMQ 源码分析二四之根据MsgId查询消息

> 本开始分析 RocketMQ 根据 MsgId 来查询消息。
> 

### 命令参数

> 用法：sh mqadmin `queryMsgById`  -i afcb
指令：`queryMsgById`
代码入口：org.apache.rocketmq.tools.command.message.QueryMsgByIdSubCommand
> 

| 参数 | 是否必填 | 说明 |
| --- | --- | --- |
| -i | 是 | msgId |
| -g | false | 消费分组 |
| -d | 否 | clinetId |
| -s | 否 | 重新发送消息 |
| -u | 否 | unit name |

解析命令行参数入口

```java
// RocketMQ 配置了 命令行的执行 shell 脚本入口。就是下面的 mqadmin.sh 这个文件
mqadmin.sh

// 解析命令行入口
org.apache.rocketmq.tools.command.MQAdminStartup#main0

// 设置 namesrvAddr 为全局变量。
if (commandLine.hasOption('n')) {
    String namesrvAddr = commandLine.getOptionValue('n');
    System.setProperty(MixAll.NAMESRV_ADDR_PROPERTY, namesrvAddr);
}
```

### 模块介绍

- 查询消息<font color='green'>（这个是本文分析的重点）</font>
- 投递至某个 clientId消息，还是基于 <font color='green'>查询消息</font> 功能再 丰富一下。
- 重新发送消息，还是基于 <font color='green'>查询消息</font> 功能再 丰富一下。

### 投递消息给某个 ClientId

```java
@Override
    public ConsumeMessageDirectlyResult consumeMessageDirectly(String consumerGroup, String clientId, String msgId)
        throws RemotingException, MQClientException, InterruptedException, MQBrokerException {
        MessageExt msg = this.viewMessage(msgId);

        return this.mqClientInstance.getMQClientAPIImpl().consumeMessageDirectly(RemotingUtil.socketAddress2String(msg.getStoreHost()),
            consumerGroup, clientId, msgId, timeoutMillis * 3);
    }
```

### 重新发送消息

```java
private void sendMsg(final DefaultMQAdminExt defaultMQAdminExt, final DefaultMQProducer defaultMQProducer,
        final String msgId) throws RemotingException, MQBrokerException, InterruptedException, MQClientException {
        try {
            MessageExt msg = defaultMQAdminExt.viewMessage(msgId);
            if (msg != null) {
                // resend msg by id
                System.out.printf("prepare resend msg. originalMsgId=%s", msgId);
                SendResult result = defaultMQProducer.send(msg);
                System.out.printf("%s", result);
            } else {
                System.out.printf("no message. msgId=%s", msgId);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

### 根据MsgId查询消息

```java
/**
     * 查看消息，通过 MsgId 来查询
     * @param offsetMsgId message id
     * @return
     * @throws RemotingException
     * @throws MQBrokerException
     * @throws InterruptedException
     * @throws MQClientException
     */
    @Override
    public MessageExt viewMessage(
        String offsetMsgId) throws RemotingException, MQBrokerException, InterruptedException, MQClientException {
        return defaultMQAdminExtImpl.viewMessage(offsetMsgId);
    }
```

### 根据 MsgId 获取到 Broker地址与 offset 地址

```java
/**
     * 构建 msgID
     * @param input
     * @param addr
     * @param offset
     * @return
     */
    public static String createMessageId(final ByteBuffer input, final ByteBuffer addr, final long offset) {
        // limit=position;position=0;
        input.flip();
        int msgIDLength = addr.limit() == 8 ? 16 : 28;
        input.limit(msgIDLength);

        input.put(addr);
        input.putLong(offset);

        return UtilAll.bytes2string(input.array());
    }

/**
     * 解码 MsgId
     *
     * @param msgId
     * @return
     * @throws UnknownHostException
     */
    public static MessageId decodeMessageId(final String msgId) throws UnknownHostException {
        SocketAddress address;
        // 在 commit log 中的物理位置
        long offset;
        int ipLength = msgId.length() == 32 ? 4 * 2 : 16 * 2;

        byte[] ip = UtilAll.string2bytes(msgId.substring(0, ipLength));
        byte[] port = UtilAll.string2bytes(msgId.substring(ipLength, ipLength + 8));
        ByteBuffer bb = ByteBuffer.wrap(port);
        int portInt = bb.getInt(0);
        address = new InetSocketAddress(InetAddress.getByAddress(ip), portInt);

        // offset
        byte[] data = UtilAll.string2bytes(msgId.substring(ipLength + 8, ipLength + 8 + 16));
        bb = ByteBuffer.wrap(data);
        offset = bb.getLong(0);

        return new MessageId(address, offset);
    }
```

### RequestCode

```java
// Broker 根据消息ID来查询消息
public static final int VIEW_MESSAGE_BY_ID = 33;
```

### Broker 端处理

```java
// 先获取消息体的大小
SelectMappedBufferResult sbr = this.commitLog.getMessage(commitLogOffset, 4);
 

// 获取每个 MappedFile 大小
int mappedFileSize = this.defaultMessageStore.getMessageStoreConfig().getMappedFileSizeCommitLog();

// 根据 最小的 MappedFile 和 LastMappedFile 找到具体的 MappedFile ，就是一个二分法
MappedFile firstMappedFile = this.getFirstMappedFile();
MappedFile lastMappedFile = this.getLastMappedFile();

// 根据找到的 MappedFile 查找消息
int pos = (int) (offset % mappedFileSize);

// 读取具体内容
public SelectMappedBufferResult selectMappedBuffer(int pos, int size) {
        // return this.writeBuffer == null ? this.wrotePosition.get() : this.committedPosition.get();
        int readPosition = getReadPosition();
        if ((pos + size) <= readPosition) {
            if (this.hold()) {
                ByteBuffer byteBuffer = this.mappedByteBuffer.slice();
                byteBuffer.position(pos);
                ByteBuffer byteBufferNew = byteBuffer.slice();
                byteBufferNew.limit(size);
                return new SelectMappedBufferResult(this.fileFromOffset + pos, byteBufferNew, size, this);
            } else {
                log.warn("matched, but hold failed, request pos: " + pos + ", fileFromOffset: "
                    + this.fileFromOffset);
            }
        } else {
            log.warn("selectMappedBuffer request pos invalid, request pos: " + pos + ", size: " + size
                + ", fileFromOffset: " + this.fileFromOffset);
        }

        return null;
    }
```

### 查询消息消费轨迹

下章节继续分享。

### 总结

- RocketMQ msgId 是组成是有规则的，根据规则能拿到具体的 <font color='green'>Broker地址和 offset</font>。
- 根据 此 offset 来找到对应的 MappedFile，再读取指定的消息内容。