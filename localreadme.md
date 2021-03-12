
# 版本:
0.8.2.2-local  代表着我只是基于这个版本的分析哈.

# compile problems
``
A problem occurred evaluating root project 'kafka'.
> Failed to apply plugin class 'org.gradle.api.plugins.scala.ScalaBasePlugin'.
   > Could not create task ':core:compileScala'.
      > No such property: useAnt for class: org.gradle.api.tasks.scala.ScalaCompileOptions

``

```


Putting those lines the build.gralde seems to fix the problem in my project:

  ScalaCompileOptions.metaClass.daemonServer = true
  ScalaCompileOptions.metaClass.fork = true
  ScalaCompileOptions.metaClass.useAnt = false
  ScalaCompileOptions.metaClass.useCompileDaemon = false


```


# key entry points
1. server start process apis
``
core/src/main/scala/kafka/server/KafkaApis.scala#handle
``
2. 用到的zk的路径:
`ZkUtils`

```
源码如下:
  val ConsumersPath = "/consumers"
  val BrokerIdsPath = "/brokers/ids"
  val BrokerTopicsPath = "/brokers/topics"
  val TopicConfigPath = "/config/topics"
  val TopicConfigChangesPath = "/config/changes"
  val ControllerPath = "/controller"
  val ControllerEpochPath = "/controller_epoch"
  val ReassignPartitionsPath = "/admin/reassign_partitions"
  val DeleteTopicsPath = "/admin/delete_topics"
  val PreferredReplicaLeaderElectionPath = "/admin/preferred_replica_election"
```

## 发送消息
入口:``kafka.server.KafkaApis.handleProducerOrOffsetCommitRequest``[源码](core/src/main/scala/kafka/server/KafkaApis.scala)
1. 如果当前broker是自己的话 则处理(加入本地的logfile. ``kafka.server.KafkaApis.appendToLocalLog`` ).  否则抛出异常: ``NotLeaderForPartitionException`` <br>
   可能需要更新HIGHWATERMARK.``kafka.cluster.Partition.maybeIncrementLeaderHW`` (根据当前ISR中的offset是否大于当前leader的offset决定)
   
2. 根据request.acks 来决定<br>
   如果是acks=-1, 啥都不干.<br>
   如果是acks=1, 更新本地的offset并提交.<br>
   如果是其他, 新增一个DelayedProduce. 该请求根据: ``kafka.server.DelayedProduce.isSatisfied``方法决定是否已经被复制到足够多的副本. <br>
   (该DeplayedProduce会在 <br>
   **Case1**: 在来自其他Follwer的FetchRequest中 ``kafka.server.KafkaApis.handleFetchRequest`` --> ``kafka.server.KafkaApis.recordFollowerLogEndOffsets``<br> 
   **Case2**: 自己的这个ProduceRequest 会尝试收集已经满足的DelayedRequest)<br>

## 同步其他Leader的消息
处理FetchRequest``kafka.server.KafkaApis.handleFetchRequest``.
1. 从leader当前log中读取``kafka.log.Log.read``[源码](core/src/main/scala/kafka/log/Log.scala)
2. 如果当前request是来自于某个Follower则根据其offset更新可能满足的DeplayedProduce. (因为这个可能已经拷贝了新的数据到follwer)
3. 更新收集可能的DelayedFetch.(比如这个FetchRequest配置了maxwait  那么可能在server端等待, 就会生成DelayedFetch).

## Rebalance
### Kafka如何决定每个Partition的Leader的?
1. 每个Node启动后尝试通过ZK做选举 成为Controller``kafka.controller.KafkaController.startup`` (使用zkpath: /controller). 将当前节点数据尝试写入
```java
"version" -> 1, "brokerid" -> brokerId, "timestamp" -> timestamp
```
2. 如果成为了Controller. 尝试后续的partitions assign: ``kafka.controller.KafkaController.onControllerFailover``
```java
a. 获取所有的topics (读取path), /brokers/topics
b. 获取所有topic的partition相关信息 放入内存. TopicAndPartition 以及Replica个数
c. 对不在线的Parition(或者没有分配的partition) kafka.controller.PartitionStateMachine.triggerOnlinePartitionStateChange
d. 做Leader分配. kafka.controller.PartitionStateMachine.handleStateChange. 可能会从该parttion的已有ISR中选择一个
```

### Kafka如何分配Partition的?

## Exactly once delivery
Producer side:

Consumer side:


