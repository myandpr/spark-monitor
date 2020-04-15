# spark-monitor
## spark常用的监控告警系统实现方式
  分享一下关于在spark on yarn任务我们应该做哪些监控，如何监控。
  一般来说，对于spark的任务关心的指标主要是：app存货，spark streaming的job堆积情况，job运行状态及进度，stage运行进度，rdd缓存监控，内存监控等。
  其中，最重要的是App存活，因为峰值容易导致应用挂掉
### App存活监控
  在yarn上运行的spark，可以通过yarn客户端获取任务。
```
Configuration conf = new YarnConfiguration();
YarnClient yarnClient = YarnClient.createYarnClient();
yarnClient.init(conf);
yarnClient.start();
try{
   List<ApplicationReport> applications = yarnClient.getApplications(EnumSet.of(YarnApplicationState.RUNNING, YarnApplicationState.FINISHED));
   System.out.println("ApplicationId ============> "+applications.get(0).getApplicationId());
   System.out.println("name ============> "+applications.get(0).getName());
   System.out.println("queue ============> "+applications.get(0).getQueue());
   System.out.println("queue ============> "+applications.get(0).getUser());
} catch(YarnException e) {
   e.printStackTrace();
} catch(IOException e) {
   e.printStackTrace();
}
       yarnClient.stop();
```
  还可以有更加细节监控：executor数量，内存，cup等。以下就是获取这些指标的方法：
#### ApplicationInfo
  调用SparkContext的AppStatusStore（新版本的spark有，应该是从2.X的某个版本开始的，具体忘了）
```
val statusStore = sparkContext.statusStore
statusStore.applicationinfo()
```
  以下是获取到的ApplicationInfo的结构
```
case class ApplicationInfo private[spark](
    id: String,
    name: String,
    coresGranted: Option[Int],
    maxCores: Option[Int],
    coresPerExecutor: Option[Int],
    memoryPerExecutorMB: Option[Int],
    attempts: Seq[ApplicationAttemptInfo]) 
```
#### AppSummary
  调用SparkContext的AppStatusStore对象获取AppSummary汇总
```
val statusStore = sparkContext.statusStore
statusStore.appSummary()

statusStore.appSummary().numCompletedJobs
statusStore.appSummary().numCompletedStages
```
### Job监控
  job的运行状态信息，spark streaming的job堆积情况。这种监控主要是通过继承一个StreamingListener，然后SparkContext.addStreamingListener注册监听器。
  下面例子是spark streaming 数据量过大，导致batch不能及时处理而使得batch堆积的情况。
```
val waitingBatchUIData = new HashMap[Time, BatchUIData]
ssc.addStreamingListener(new StreamingListener {
  override def onStreamingStarted(streamingStarted: StreamingListenerStreamingStarted): Unit = println("started")

  override def onReceiverStarted(receiverStarted: StreamingListenerReceiverStarted): Unit = super.onReceiverStarted(receiverStarted)

  override def onReceiverError(receiverError: StreamingListenerReceiverError): Unit = super.onReceiverError(receiverError)

  override def onReceiverStopped(receiverStopped: StreamingListenerReceiverStopped): Unit = super.onReceiverStopped(receiverStopped)

  override def onBatchSubmitted(batchSubmitted: StreamingListenerBatchSubmitted): Unit = {
    synchronized {
      waitingBatchUIData(batchSubmitted.batchInfo.batchTime) =
        BatchUIData(batchSubmitted.batchInfo)
    }
  }

  override def onBatchStarted(batchStarted: StreamingListenerBatchStarted): Unit =     waitingBatchUIData.remove(batchStarted.batchInfo.batchTime)
  
  override def onBatchCompleted(batchCompleted: StreamingListenerBatchCompleted): Unit = super.onBatchCompleted(batchCompleted)

  override def onOutputOperationStarted(outputOperationStarted: StreamingListenerOutputOperationStarted): Unit = super.onOutputOperationStarted(outputOperationStarted)

  override def onOutputOperationCompleted(outputOperationCompleted: StreamingListenerOutputOperationCompleted): Unit = super.onOutputOperationCompleted(outputOperationCompleted)
})
```
  最终，我们使用waitingBatchUIData的大小，代表待处理的batch大小，比如待处理批次大于10，就告警，这个可以按照任务的重要程度和持续时间来设置一定的告警规则，避免误操作。
### Stage监控
  这个不太重要，statusStore.activeStages()得到的是一个Seq[v1.StageData] 
```
class StageData private[spark](
    val status: StageStatus,
    val stageId: Int,
    val attemptId: Int,
    val numTasks: Int,
    val numActiveTasks: Int,
    val numCompleteTasks: Int,
    val numFailedTasks: Int,
    val numKilledTasks: Int,
    val numCompletedIndices: Int,

    val executorRunTime: Long,
    val executorCpuTime: Long,
    val submissionTime: Option[Date],
    val firstTaskLaunchedTime: Option[Date],
    val completionTime: Option[Date],
    val failureReason: Option[String],

    val inputBytes: Long,
    val inputRecords: Long,
    val outputBytes: Long,
    val outputRecords: Long,
    val shuffleReadBytes: Long,
    val shuffleReadRecords: Long,
    val shuffleWriteBytes: Long,
    val shuffleWriteRecords: Long,
    val memoryBytesSpilled: Long,
    val diskBytesSpilled: Long,

    val name: String,
    val description: Option[String],
    val details: String,
    val schedulingPool: String,

    val rddIds: Seq[Int],
    val accumulatorUpdates: Seq[AccumulableInfo],
    val tasks: Option[Map[Long, TaskData]],
    val executorSummary: Option[Map[String, ExecutorStageSummary]],
    val killedTasksSummary: Map[String, Int])
```

### RDD监控
  最不重要的，SparkContext.AppStatusStore
```
val statusStore = sparkContext.statusStore
statusStore.rddList()
```
```
class RDDStorageInfo private[spark](
    val id: Int,
    val name: String,
    val numPartitions: Int,
    val numCachedPartitions: Int,
    val storageLevel: String,
    val memoryUsed: Long,
    val diskUsed: Long,
    val dataDistribution: Option[Seq[RDDDataDistribution]],
    val partitions: Option[Seq[RDDPartitionInfo]])

class RDDDataDistribution private[spark](
    val address: String,
    val memoryUsed: Long,
    val memoryRemaining: Long,
    val diskUsed: Long,
    @JsonDeserialize(contentAs = classOf[JLong])
    val onHeapMemoryUsed: Option[Long],
    @JsonDeserialize(contentAs = classOf[JLong])
    val offHeapMemoryUsed: Option[Long],
    @JsonDeserialize(contentAs = classOf[JLong])
    val onHeapMemoryRemaining: Option[Long],
    @JsonDeserialize(contentAs = classOf[JLong])
    val offHeapMemoryRemaining: Option[Long])

class RDDPartitionInfo private[spark](
    val blockName: String,
    val storageLevel: String,
    val memoryUsed: Long,
    val diskUsed: Long,
    val executors: Seq[String])
```

### RDD内存及缓存监控
### Executor监控
  Executor的注册，启动，挂掉都可以通过SparkListener来获取到，而单个executor内部的细节获取也还是通过SparkContext的一个内部变量，叫做SparkStatusTracker。
  ```
  sc.statusTracker.getExecutorInfos
  ```
  得到的是一个Array[SparkExecutorInfo]
```
private class SparkExecutorInfoImpl(
    val host: String,
    val port: Int,
    val cacheSize: Long,
    val numRunningTasks: Int,
    val usedOnHeapStorageMemory: Long,
    val usedOffHeapStorageMemory: Long,
    val totalOnHeapStorageMemory: Long,
    val totalOffHeapStorageMemory: Long)
  extends SparkExecutorInfo
```

## 总结
  其实最重要的监控就两种：
    1、App存活监控
    2、自定义SparkListener监听器，实现检测spark Streaming队列积压
