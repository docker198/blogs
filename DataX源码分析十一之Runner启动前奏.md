> 在前面几章中，我们提到了。DataX将 Reader、Writer 包装成 Runner 来执行？为啥搞这么复杂呢？
> 

继上N篇文章：

[DataX源码分享八之TaskGroupContainer.md](DataX源码分享八之TaskGroupContainer.md) 

[DataX源码分析九至Task初始化.md](DataX源码分析九至Task初始化.md) 




## **起码启动入口**

> 启动代码入口片段：<font color='green'>com.alibaba.datax.core.taskgroup.TaskGroupContainer.TaskExecutor#doStart</font> 
>其实，逻辑都是很简单的。
> 

```java
// 启动 Writer 组件
this.writerThread.start();
// 启动 Reader 组件
this.readerThread.start();
```

## **开始导数据了嘛？**

不！上述的代码片段中的 writerThread、readerThread 其实是 AbstractRunner 的继承类。那它里面都干了些啥呢？让我们一块瞧瞧。

## **AbstractRunner 概述**

> AbstractRunner 是 ReaderRuner、WriterRunner 的 abstract 类。类的继承图如下所示。
> 

![Alt text](images/datax_11_01.png)
AbstractRunner 的继承关系图谱

## **Runner 启动**

> Runner 启动，都是通过调用 <font color='green'>abstractRunner</font> 类中的 start 方法来启动的。下面的代码片段我是摘抄的。
> 

```java
// 初始化探针 
PerfRecordLOG.debug("task writer starts to do init ...");
PerfRecord initPerfRecord = new PerfRecord(getTaskGroupId(), getTaskId(), PerfRecord.PHASE.WRITE_TASK_INIT);
// 探针启动
initPerfRecord.start();
// Writer 插件启动
taskWriter.init();
// 探针结束
initPerfRecord.end();
```

还是一样的套路。Runner 启动时，还是按照顺序来调用<font color='green'>init()、prepare()、startWrite()、post</font>类似的方法。和在 Job 里面的方法逻辑差不多。

## **总结**

- AbstractRunner 所做的事情也是比较简单的。没有很高深的代码逻辑在里面。
- AbstractRunner 更多的是提供了<font color='green'>模板方法</font>。