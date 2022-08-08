> 上一篇文章中，我们提到了 Reader 来读取数据，放到 buffer 或者 channel ，那接下来，我们看看 Writer 是如何读取消息的。
> 

继上一篇：

[DataX源码分析十二之Reader读取数据.md](DataX源码分析十二之Reader读取数据.md) 


## **代码入口**

```java
// 代码入口com.alibaba.datax.core.taskgroup.runner.WriterRunner#run
//启动 Writer 写入数据到目标元
taskWriter.startWrite(recordReceiver);

```

## **Reader 推送数据到 channel 里面**

```java
    @Override
    public void flush() {
        if (shutdown) {
            throw DataXException.asDataXException(CommonErrorCode.SHUT_DOWN_TASK, "");
        }
        // 这里会将数据写入到 channel 里面，并且还会简单的限流以下。
        this.channel.pushAll(this.buffer);
        this.buffer.clear();
        this.bufferIndex = 0;
        this.memoryBytes.set(0);
    }

    @Override
    protected void doPushAll(Collection<Record> rs) {
        try {
            long startTime = System.nanoTime();
            lock.lockInterruptibly();
            int bytes = getRecordBytes(rs);
            while (memoryBytes.get() + bytes > this.byteCapacity || rs.size() > this.queue.remainingCapacity()) {
                // 等待      
                notInsufficient.await(200L, TimeUnit.MILLISECONDS);
            }
            // MemoryChannel ，添加数据
            this.queue.addAll(rs);
            waitWriterTime += System.nanoTime() - startTime;
            memoryBytes.addAndGet(bytes);
            // 重入锁
            notEmpty.signalAll();
        } catch (InterruptedException e) {
            throw DataXException.asDataXException(FrameworkErrorCode.RUNTIME_ERROR, e);
        } finally {
            lock.unlock();
        }
    }
```

## **Writer 读取数据**

> 从 Reader 读取数据
> 

```java
    @Override
    public Record getFromReader() {
        if (shutdown) {
            throw DataXException.asDataXException(CommonErrorCode.SHUT_DOWN_TASK, "");
        }
        boolean isEmpty = (this.bufferIndex >= this.buffer.size());
        if (isEmpty) {
            receive();
        }
        Record record = this.buffer.get(this.bufferIndex++);
        if (record instanceof TerminateRecord) {
            record = null;
        }
        return record;
    }
```

- 注意 receive() 这个方法。如果 buffuer 是空的话，那么，将 pull 数据放到 buffer 里面，然后再从 buffer 里面 GET 一次数据。

## **Writer 拉取数据**

> DataX 的 Reader 和 Writer 之间，是生产者和消费者模型。
> 

```java
    private void receive() {
        this.channel.pullAll(this.buffer);
        this.bufferIndex = 0;
        this.bufferSize = this.buffer.size();
    }

    @Overrideprotected
    void doPullAll(Collection<Record> rs) {
        assert rs != null;
        rs.clear();
        try {
            long startTime = System.nanoTime();
            lock.lockInterruptibly();
            // 批量获取，为空不阻塞,返回获取条数
            while (this.queue.drainTo(rs, bufferSize) <= 0) {
                // 重入锁
                notEmpty.await(200L, TimeUnit.MILLISECONDS);
            }
            waitReaderTime += System.nanoTime() - startTime;
            int bytes = getRecordBytes(rs);
            memoryBytes.addAndGet(-bytes);
            // 还有空间
            notInsufficient.signalAll();
        } catch (InterruptedException e) {
            throw DataXException.asDataXException(FrameworkErrorCode.RUNTIME_ERROR, e);
        } finally {
            lock.unlock();
        }
    }
```

## **结论**

1. DataX 通过 Channel ，达到了 Reader、Writer 解耦。
2. DataX 通过 buffer 缓存数据，通过 channel 来达到批量推送数据到 Writer里面。
3. DataX 中的 Reaer、Writer 通过生产者、消费者模型，达到了解耦的目的。