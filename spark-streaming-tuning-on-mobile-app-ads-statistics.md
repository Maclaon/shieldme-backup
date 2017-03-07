---
title: 移动广告实时统计之Spark Streaming调优
date: 2015-11-30 10:00:00
tags: 移动广告，实时统计，Spark Streaming
author: maclaon
comments: false
---
# 背景
移动广告在App中对特定广告位的进行数据投放的时候，需要及时的获取数据的下发数据，展示数据和点击数据，特别是对交互式广告的数据进行数据统计，以便下发个性化的广告，提高广告转化率和平台的收入。实时统计和实时计算在整个大数据运算中占有重要的位置，本文结合广告业务的场景，来阐述Spark Streaming的实时统计调优的一些技术点。

## 移动广告
移动广告平台主要对下游的SSP下发广告，对上游的广告平台进行对接，同时需要完成对广告的投放的数据统计工作，统计下发、展示和点击的数据，同时还需要对广告点击的行为来进行分析，确定是否反作弊。

## 业务场景
广告下发过后需要对下发的广告，展示曝光和点击的广告的触发行为进行日志的上报，通过`flume`的收集方式一部分写到`HDFS`中，一部分写到RMQ队列中用于实时统计。场景如下:
![](http://oh8mi0yav.bkt.clouddn.com/ad-plantom-log-statistic.png)

<!--more-->
广告下发，展示和点击的量会随着SSP下游的流量高峰的不同导致整个写入到存储媒介是不一致的，但是遵循一定的模式，广告流量高峰期是在上午10点和下午4点以及晚上的10点左右，该阶段的数据量会比较大，直接影响到数据写入的流量，进而对实时统计的入口产生间接的影响。

# Spark Streaming
spark streaming通过结合一个或者多个流式数据源所提供的数据，经过spark的两个阶段`transform`和`action`的操作来进行提交执行。每当数据源产生数据的时候，在给定的批处理间隔时间内(`batch interval`)，收集数据，以`blocks`的方式进行打包，一旦一个批处理时间间隔完成，打包成`blocks`的数据被传递给Spark进行处理。

## 处理时序
+ Spark Streaming constructor通过构造函数确定`batch Interval`的时间.
``` Java
val ssc = new StreamingContext(sc,Seconds(10))
```

![](http://oh8mi0yav.bkt.clouddn.com/spark-streaming-sequence-in-batch-interval.png)

+ 数据以数组的方式存储在`blocks`中，随后被发送到`Block Manager`中，这些`blocks`最终形成运行在Spark上的`RDD`分区。
+ 在Spark Streaming的$$t_0$$时刻，`consumers`初始化，并从数据源处手机数据，这里我们移动广告平台采用的`consumer`是RMQ消息队列。

![](http://oh8mi0yav.bkt.clouddn.com/spark-streaming-sequence-in-batch-interval_t1.png)

+ 接下来的$$t_1$$时刻，收集于$$t_0$$时刻的数据被提交给Spark进行处理，Spark Streaming开始填充$$t_1$$时刻中`consumer`中提供的数据。

![](http://oh8mi0yav.bkt.clouddn.com/spark-streaming-sequence-in-batch-interval_t2.png)

+ 任何时刻，Spark都在处理前一个时间段的收集的数据，Streaming收集当前的数据集，上图中可以看出$$t_2$$时间段的数据被收集的时候，Spark处理$$t_1$$时间段的数据。
+ 一旦Spark处理完一个批处理时间间隔的数据比如$$t_0$$时间段的，就可以被Spark进行清除，什么时候被清除决定于`spark.cleaner.ttl`设置的时间。

总结一下Spark Streaming处理时间的组成阶段:
> + 由Spark Streaming获取数据
> + 由Spark处理数据

上述两个过程在时间上是连续的，Spark Streaming收集数据提交给Spark进行处理，由上述的运行原理可以得出如下结论：
> **处理`batch interval`时间内收集的数据所消耗的时间一定要小于`batch interval`时间。**

因为移动广告平台的业务场景是用RMQ这样的高可靠的队列来收集数据的，为了达到在`batch interval`时间内能够快速的处理收集的数据，把主要的性能调优放在Spark Job这块。

### 处理机制
> + 一个Spark的job处理过程包括：transformations和actions，可以被分解为有限范围内的stages。
> + 一个作用域分布式上收集的`RDD`，被分解成多个分区`partitions`存活于各个节点上。
> + 一个任务就是在`executor`执行数据的一个stage阶段。提交任务的话会消耗一定固定的时间。

根据以上可以得出`batch processing`的时间因素为:

$$T_{processing\;time} \approx N_{tasks} * T_{scheduling\;cost} + N_{task} * T_{time-complexity\;per\;task}$$

上述可以得出$$N_{tasks}$$的取决因子是任务的$$stages$$和$$partition$$的个数，所以可以得出:

$$N_{tasks}=N_{stages} * N_{partition}$$

由上述的公式可以得出，缩短一个`batch interval`的处理时间主要取决于上述的$$N_{stages}$$和$$N_{partition}$$，并且增加并发性

上述的简单公式提供了流式计算在Spark Streaming中具体和那些因素有关，抽象为:tasks, stages, partitions 和 executors中。

# 调优
## 增大`Consumers`
为了增大吞吐量，可以创建多个`consumers`并行的去获取数据。每个`consumer`在`executor`上分配一个`core`，一个`executor`拥有一定数量的`cores`。

## 并发性
先前所述，Spark Streaming主要有两个并发的因素: `consumers`和`Spark processing`。`consumers`由先前的介绍可以知道，创建更多的`consumers`来达到并发性。而Spark集群处理的并发性是由配置的给`Executor`的`cores`决定的，这里的`cores`需要减去分配给`consumer`的个数。

> 假设由`spark.cores.max`配置了集群需要处理的的总核数。可以得出
> + Consumer parallelism: $$N_{consumers}$$
> + Spark parallelism: $$N_{spark.cores.max} - N_{consumers}$$

## Spark.cores.max
为了增大数据本地化和并发性，减少数据的网络传输而带来的性能消耗，$$N_{spark.cores.max}$$应该是$$N_{consumers}$$的个数，比如：移动广告平台的时候，创建了8个`consumers`来进行数据收集，那么$$N_{spark.cores.max}=16$$或者$$N_{spark.cores.max}=24$$

## Partitions
先前说过，减少分区的数量对降低整个处理的时间是非常重要的，因为会导致更少的任务量，所以一次处理更多的数据变的非常重要。

> 那么在`DStream`中我们设置多少个`partitions`给一个`RDD`合适呢？

首先看下`partitions`是如何产生的。每一次`receiver`获取数据，数据有`Receiver`提供给`executor`，`ReceiverSupervisor`用来管理`blocks`，在`batch interval`时间间隔内每一个`block`成为`RDD`的一个`partition`的一部分。这些`blocks`的大小是跟时间有关的并且是由以下参数来设置的(200毫秒是默认的值).

$$T_{spark.streaming.blockInterval} = 200$$

假设每个`consumer`产生的数据形成的`blocks`数目的能力是一致的，由此可以得出一个`RDD`中的`partitions`数目为:

$$N_{partitions}=N_{consumers} * T_{batchIntervalMillis}\;/\; T_{blockInterval}$$

> 那么如何设置$$T_{spark.streaming.blockInterval}$$ ?

增大$$T_{spark.streaming.blockInterval}$$将会减少一个`RDD`上的分区`partitions`的个数以及一个`batch`的`tasks`数目。$$T_{spark.streaming.blockInterval}$$必须是$$T_{batchIntervalMillis}$$的倍数。根据Spark的官方建议每一个`partition`需要$$2,3$$倍可用的`cores`，可以得出。
> + $$T_{bacthIntervalMillis}$$: `batch Interval`毫秒范围的值
> + $$T_{spark.cores.max}$$: 作业提交的时候设置的最大核数
> + $$N_{consumers}$$: 创建的`consumers`的个数
> + 那么`available cores`为: $$N_{sparkCores}\;=\;T_{spark.cores.max} - N_{consumers}$$
> + 一个核数可以处理多少个`partitions`: $$N_{partitionFactor}\;=\;N_{partitions}\;/\;N_{core}$$，如官网建议的一个`partition`可以有$$2,3$$倍，可以设置$$N_{partitionFactor}\;=\;2$$

$$T_{spark.streaming.blockInterval}=T_{bacthIntervalMillis}*N_{consumers}\;/\;(N_{partitionFactor} * N_{sparkCores})$$

例如：$$T_{spark.cores.max}\;=12\;$$，设置$$4$$个consumers，这样$$N_{sparkCores}=\;12\;-\;4\;=\;8$$。同时设置$$T_{bacthIntervalMillis}=6s$$，并且认为$$N_{partitionFactor}=2$$，可以得出:

$$T_{spark.streaming.blockInterval}\;=\;6000\;*\;4\;/\;(2\;*\;8)\;=1500\;ms$$


# 移动广告平台实时调优







# 结果

![](http://oh8mi0yav.bkt.clouddn.com/spark-streaming-processing-on-app-ads-statistic)




