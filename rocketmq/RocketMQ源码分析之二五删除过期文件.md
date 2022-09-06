> 本章节中，我们将分析 RocketMQ 是如何删除过期文件的。
> 

### 为什么需要删除文件

RocketMQ 中主要保存了 <font color='green'>CommitLog、Consume Queue、Index File</font> 这三种数据文件。
因为一般机器的磁盘空间是有限的，所以需要定时去删除一些过期的文件来保持磁盘有空余的空间。
RocketMQ 通过以下的配置来控制是如何删除文件的。具体配置如下

```java
// 删除文件的时间点，默认是 凌晨4点
deleteWhen=04

// 磁盘使用率，超过此使用率，也会进行强制删除
diskMaxUsedSpaceRatio=75

// 文件保留时间，默认是72小时
fileReservedTime=72
```

### 初始化删除文件服务

> 代码入口：org.apache.rocketmq.store.DefaultMessageStore#DefaultMessageStore
> 

```java
// 清理 commitlog 文件
this.cleanCommitLogService = new CleanCommitLogService();
```

### 启动定时任务删除过期文件

> 代码入口：org.apache.rocketmq.store.DefaultMessageStore#addScheduleTask
> 

```java
// 定时删除过期文件。consumequeue、commitLog、index 文件, 1s 才进行处理
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                DefaultMessageStore.this.cleanFilesPeriodically();
            }
        }, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);

/**
     * 周期性清除文件
     * 1、会不会把 pageCache 里面的信息刷到硬盘中
     */
    private void cleanFilesPeriodically() {
        // 清除 Commitlog、Maped file 文件
        this.cleanCommitLogService.run();

        // 清除 ConsumeQueue、index 文件
        this.cleanConsumeQueueService.run();
    }
```

### 删除 commitLog

> CommitLog 删除的策略，是根据最后的修改时间来判断的。逻辑比较简单。下面删除文件有 3 个约束条件
>
1、当前已经等于配置删除时间。
>
2、磁盘使用率是否草果了约定的阈值。
>
3、手动删除文件（开源版本不支持）
计算公式如下：
`long liveMaxTimestamp = mappedFile.getLastModifiedTimestamp() + expiredTime;`
> 

```java
/**
         * 清除过期的 commitlog
         */
        private void deleteExpiredFiles() {
            // 删除文件个数
            int deleteCount = 0;

            // 文件保留是时长,72小时
            long fileReservedTime = DefaultMessageStore.this.getMessageStoreConfig().getFileReservedTime();

            // 删除多个CommitLog文件的间隔时间（单位毫秒），100ms
            int deletePhysicFilesInterval = DefaultMessageStore.this.getMessageStoreConfig().getDeleteCommitLogFilesInterval();

            // 强制删除文件间隔时间（单位毫秒）,120s
            int destroyMapedFileIntervalForcibly = DefaultMessageStore.this.getMessageStoreConfig().getDestroyMapedFileIntervalForcibly();

            // 是否到达时间啦
            boolean timeup = this.isTimeToDelete();

            // 是否越过安全阈值
            boolean spacefull = this.isSpaceToDelete();

            // 是否手工触发删除文件
            boolean manualDelete = this.manualDeleteFileSeveralTimes > 0;

            // 删除物理队列文件；时间达到、空间满了、手工删除
            if (timeup || spacefull || manualDelete) {

                if (manualDelete)
                    this.manualDeleteFileSeveralTimes--;

                // 是否立刻强制删除文件
                boolean cleanAtOnce = DefaultMessageStore.this.getMessageStoreConfig().isCleanFileForciblyEnable() && this.cleanImmediately;

                // 小时转化成毫秒
                fileReservedTime *= 60 * 60 * 1000;

                // 删除文件的总数，根据 CommitLog 最后的更改时间+fileReservedTime
                deleteCount = DefaultMessageStore.this.commitLog.deleteExpiredFile(fileReservedTime, deletePhysicFilesInterval,
                    destroyMapedFileIntervalForcibly, cleanAtOnce);
            }
        }
```

### 防止删除CommitLog 失败

> RocketMQ在删除文件的时候，有可能出现失败、引用的资源还没释放等等情况。需要再次检测一下。
> 

```java
/**
         * 最前面的文件有可能Hang住，定期检查一下
         * Hang 表示线程被挂起了。需要重新删除一下文件
         */
        private void redeleteHangedFile() {
            // 定期检查Hanged文件间隔时间（单位毫秒）
            int interval = DefaultMessageStore.this.getMessageStoreConfig().getRedeleteHangedFileInterval();
            long currentTimestamp = System.currentTimeMillis();
            if ((currentTimestamp - this.lastRedeleteTimestamp) > interval) {
                this.lastRedeleteTimestamp = currentTimestamp;

                // 强制删除文件间隔时间（单位毫秒）
                int destroyMapedFileIntervalForcibly =
                    DefaultMessageStore.this.getMessageStoreConfig().getDestroyMapedFileIntervalForcibly();

                // 重新删除文件
                if (DefaultMessageStore.this.commitLog.retryDeleteFirstFile(destroyMapedFileIntervalForcibly)) {
                }
            }
        }

// 再次尝试删除的逻辑
public boolean retryDeleteFirstFile(final long intervalForcibly) {
        MappedFile mappedFile = this.getFirstMappedFile();
        if (mappedFile != null) {
            // 不可用
            if (!mappedFile.isAvailable()) {
                log.warn("the mappedFile was destroyed once, but still alive, " + mappedFile.getFileName());
                boolean result = mappedFile.destroy(intervalForcibly);
                if (result) {
                    log.info("the mappedFile re delete OK, " + mappedFile.getFileName());
                    List<MappedFile> tmpFiles = new ArrayList<MappedFile>();
                    tmpFiles.add(mappedFile);
                    this.deleteExpiredFile(tmpFiles);
                } else {
                    log.warn("the mappedFile re delete failed, " + mappedFile.getFileName());
                }

                return result;
            }
        }

        return false;
}
```

### 删除Consume Queue和 Index 文件

> 代码入口：org.apache.rocketmq.store.DefaultMessageStore.CleanConsumeQueueService#deleteExpiredFiles
总结一下：根据 commitLog 最小的 offset  来删除的。
> 

```java
/**
         * 删除过期的文件
         */
        private void deleteExpiredFiles() {
            int deleteLogicsFilesInterval = DefaultMessageStore.this.getMessageStoreConfig().getDeleteConsumeQueueFilesInterval();

            // 获取 CommitLog 最小的 offset 
            long minOffset = DefaultMessageStore.this.commitLog.getMinOffset();
            if (minOffset > this.lastPhysicalMinOffset) {
								// 上一次记录的删除位点的信息
                this.lastPhysicalMinOffset = minOffset;

                ConcurrentMap<String, ConcurrentMap<Integer, ConsumeQueue>> tables = DefaultMessageStore.this.consumeQueueTable;

                for (ConcurrentMap<Integer, ConsumeQueue> maps : tables.values()) {
                    for (ConsumeQueue logic : maps.values()) {

                        // 删除 consumequeue
                        int deleteCount = logic.deleteExpiredFile(minOffset);

                        if (deleteCount > 0 && deleteLogicsFilesInterval > 0) {
                            try {
                                Thread.sleep(deleteLogicsFilesInterval);
                            } catch (InterruptedException ignored) {
                            }
                        }
                    }
                }

                // 删除索引
                DefaultMessageStore.this.indexService.deleteExpiredFile(minOffset);
            }
        }
```

### Consume queue 删除

> 下面我只摘抄核心代码
> 

```java
// 读取 mapped 最后的一个 区域块的数据
SelectMappedBufferResult result = mappedFile.selectMappedBuffer(this.mappedFileSize - unitSize);

// 最后一个区域块的数据 ，是否 小于 commitLog 最小的 offset ，这种情况就得要删除
destroy = maxOffsetInLogicQueue < offset;

// 添加到删除文件的集合中
if (destroy && mappedFile.destroy(1000 * 60)) {
   files.add(mappedFile);
   deleteCount++;
}
```

### Index 文件删除

> 代码入口：org.apache.rocketmq.store.index.IndexService#deleteExpiredFile(long)
> 

```java
// 获取 index 文件第一个文件的最后的 offset 
long endPhyOffset = this.indexFileList.get(0).getEndPhyOffset();
if (endPhyOffset < offset) {
    files = this.indexFileList.toArray();
}

// 删除文件
if (files != null) {
            List<IndexFile> fileList = new ArrayList<IndexFile>();
            for (int i = 0; i < (files.length - 1); i++) {
                IndexFile f = (IndexFile) files[i];
                if (f.getEndPhyOffset() < offset) {
                    fileList.add(f);
                } else {
                    break;
                }
            }

            this.deleteExpiredFile(fileList);
}
```

### 删除文件

```java
/**
     * 禁止资源被访问 shutdown不允许调用多次，最好是由管理线程调用
     */
    public void shutdown(final long intervalForcibly) {
        if (this.available) {
            this.available = false;
            this.firstShutdownTimestamp = System.currentTimeMillis();
            this.release();
        }
        // 强制shutdown
        else if (this.getRefCount() > 0) {
            if ((System.currentTimeMillis() - this.firstShutdownTimestamp) >= intervalForcibly) {
                this.refCount.set(-1000 - this.getRefCount());
                this.release();
            }
        }
    }

/**
     * 资源是否被清理完成，引用的数量&&是否刷盘
     */
    public boolean isCleanupOver() {
        return this.refCount.get() <= 0 && this.cleanupOver;
    }

/**
     * 清理资源，destroy与调用shutdown的线程必须是同一个
     *
     * @return 是否被destory成功，上层调用需要对失败情况处理，失败后尝试重试
     */
    public boolean destroy(final long intervalForcibly) {
        this.shutdown(intervalForcibly);

        if (this.isCleanupOver()) {
            try {
                // channel 关闭
                this.fileChannel.close();
                log.info("close file channel " + this.fileName + " OK");

                long beginTime = System.currentTimeMillis();

                // 删除文件，主要用于预先分配 MappedFile 的情况
                boolean result = this.file.delete();

                log.info("delete file[REF:" + this.getRefCount() + "] " + this.fileName
                    + (result ? " OK, " : " Failed, ") + "W:" + this.getWrotePosition() + " M:"
                    + this.getFlushedPosition() + ", "
                    + UtilAll.computeElapsedTimeMilliseconds(beginTime));
            } catch (Exception e) {
                log.warn("close file channel " + this.fileName + " Failed. ", e);
            }

            return true;
        } else {
            log.warn("destroy mapped file[REF:" + this.getRefCount() + "] " + this.fileName
                + " Failed. cleanupOver: " + this.cleanupOver);
        }

        return false;
    }
```

### 总结

- 本文总结了RocketMQ 删除文件的策略，还是比较简单的。主要<font color='green'>大头是 commitLog</font>
- RocketMQ 删除文件，删除了 CommitLog、consume queue、index File 三种文件。
- RocketMQ 删除 CommitLog 是根据最后的修改时间来判定的。
- RocketMQ 删除 consume queue 、index ，是根据其 第一个文件的 最后一个区块的 offset 是否小于 当前 commitLog 最小的 offset 作为判定的依据。
- 在 RocketMQ 删除 commitLog 的时候，还有一个逻辑是再次检测 commitLog 是否删除成功，比如是否还有资源在关联使用，没有来得及释放，比如写入量巨大，还没来的及刷盘等等。