---
title: 分布式任务调度框架Saturn研究（二）
date: 2016-12-17 11:39:37
tags: 分布式，Zookeeper，任务调度，唯品会
comments: false
---
针对于前一章节的[分布式任务调度框架Saturn研究（一）](http://shieldme.cn/2016/12/10/saturn-source-code-reading-chapter1/)的阐述，Saturn主要的功能，架构和业务逻辑都已经明确，这一章主要从源码的角度来进行分析和明确，初步以流程图，类图和代码片段的方式来进行阐述，力求做到为什么这样设计，这样设计有什么好处，有哪些改进点。
<!--more-->
## 系统划分
| 源码子系统| 名称 | 功能概要  |
| :----: |:----:|:----:|
| 1      | saturn-core | 公共模块，定义公共类，方法及实现 |
| 2      | saturn-console-core |  saturn控制台公共模块 |
| 3 	 | saturn-console      |    saturn控制台 |
|4|saturn-executor|saturn执行结点|
|5|saturn-job-sharding|saturn作业分片调度器|
|6|saturn-job-embed|嵌入方式使用saturn的转换模块（把saturn嵌入其它系统中运行，比如tomcat)|
|7|saturn-plugin|saturn maven插件|
|8|saturn-it|saturn 集成测试|
|9|saturn-job-api|暴露API|

## Saturn-core(公共模块)
> com.vip.saturn.job.basic

### AbstractElasticJob   
![类图](http://oh8mi0yav.bkt.clouddn.com/saturn-core-AbstractElasticJob.png)   
上述是弹性化分布式作业的基类，其中定义了作业的状态的，作业的状态类似于**单机系统下的线程执行模型**，作业状态如下图：
![作业状态](https://raw.githubusercontent.com/vipshop/Saturn/doc/architect/status.png)    
+ `private static Logger log`: 抽象基类`AbstractElasticJob`的日志记录
+ `private ExecutorService executorService`: java.util.concurrent中的的线程管理类，继承于`Executor`，`Executor`的接口很简单，是一个抽象的命令模式，仅有一个方法execute(Runable command),`Executor`的接口虽然简单，但是提供了在设计模式上的抽象基类；而`ExecutorService`提供了除基类`Executor`以外的用于对外部服务的"Service"特性，增加了更多用于对当前线程进行管控或者状态查询的方法(Ps:基于`Executor`的继承实现上述的方法，也是可以的，`ExecutorService`的源码是值得参考的)
+ `private boolean stopped = false`: 分布式作业是否被停止
+ `private boolean forceStopped = false`: 分布式作业是否被强制停止
+ `private boolean aborted = false`: 分布式作业是否被丢弃
+ `protected ConfigurationService configService`: 分布式作业配置服务，继承于`AbstractSaturnService`基类，该基类的源码分析[请点击]()
+ `protected ShardingService shardingService`: 分布式作业分片服务类
+ `protected ExecutionContextService executionContextService`: 分布式作业任务运行时的上下文服务类
+ `protected ExecutionService executionService`: 执行分布式作业的服务
+ `protected FailoverService failoverService`: 分布式作业执行失败转移服务
+ `protected OffsetService offsetService`: 数据处理位置服务
+ `protected ServerService serverService`: 作业服务器节点服务
+ `protected String executorName`: 执行节点的名称
+ `protected String jobName`: 执行job的名称
+ `protected String namespace`: 所属域空间
+ `protected SaturnScheduler saturnScheduler`: Saturn调度器
+ `protected JobScheduler jobScheduler`: 作业调度器
+ `protected SaturnExecutorService saturnExecutorService`: Saturn节点执行服务

(Ps: 按照顺序继续阅读，回来在来补充各个方法)

### AbstractJobExecutionShardingContext
![](http://oh8mi0yav.bkt.clouddn.com/AbstractJobExecutionShardingContext.png)

上述是作业运行时分片上下文的抽象类，该类比较简单，定义相关的私有变量，仅仅作为初始的基类被复用
+ `private String jobName`: 作业名称
+ `private int shardingTotalCount`: 分片总数
+ `private String jobParameter`: 作业自定义参数
+ `private Map<String,String> customContext`: 自定义上下文

### AbstractSaturnJob
![](http://oh8mi0yav.bkt.clouddn.com/AbstractSaturnJob.png)
该类继承于`AbstractElasticJob`，形成了Saturn抽象父类，该类具体实现了分布式作业基类的相关接口，其中每个接口的作用如下：
+ `protected final void executeJob(final JobExecutionMultipleShardingContext shardingContext)`: 实现基类`AbstractElasticJob`中`executeJob`的方法，传入的参数是继承实现了基类`AbstractJobExecutionShardingContext`用于管理分布式作业在运行上做个分片的上下文信息，获取Context中的相关参数后，校验参数，处理对应的分布式作业，处理完后，将作业进行汇总更新状态，其中处理任务是暴露对外的`protected abstract Map<Integer, SaturnJobReturn> handleJob(SaturnExecutionContext shardingContext)`接口，每个分片返回一个`SaturnReturn`，如果为null，标识当前分片的任务执行失败。(Ps:**该部分的源码设计思想和MyBatis Generator的源码类似，主要解决多任务执行的，用一个上下文的Context的类来记录该次任务执行的相关信息，在把具体执行的细节暴露给业务方使用**)
+ `protected void updateExecuteResult(SaturnJobReturn saturnJobReturn, SaturnExecutionContext saturnContext, int item)`: 更新分布式作业分片中任务执行的结果，该类根据结果执行的返回值`SaturnJobReturn`中的各种状态进行处理成功，超时，失败等处理，`item`传入的是需要处理的分片任务，同时把成功执行的结果设置到当前分布式作业的上下文环境中，并对处理的结果进行计数统计，用`public final class ProcessCountStatistics`的静态方法进行统计。
+ `public Properties parseKV(String path)`: 解析自定义参数传递进来的以逗号分隔符分割key=value的形式放入`Properties`中。
+ `public int getTimeoutSeconds()`: 获取作业超时时间
+ `protected String getRealItemValue(String jobParameter, String jobValue)`: 获取替换后的作业分片执行值，作业分片对应的参数值，在具体执行SaturnJavaJob和SaturnScriptJob中用到
+ `public String logBusinessExceptionIfNecessary(String jobName, Exception e)`: 记录任务执行的错误
+ `public void updateJobCron(String jobName, String cron, Map<String, String> customContext)`: 更新cron表达式的执行策略
+ `public abstract SaturnJobReturn`: 母鸡什么功能!

### AbstractSaturnService
![](http://oh8mi0yav.bkt.clouddn.com/AbstractSaturnService.png)

Saturn服务类的抽象基类，主要输作业名称，执行名称，作业配置，zookeeper配置，服务启用和关闭的一些服务

### CrondJob
![](http://oh8mi0yav.bkt.clouddn.com/CrondJob.png)
具体的Crond作业类型，其方法实现了基类`AbstractSaturnJob`和`AbstractElasticJob`的方法，用于执行Crond中表达式定时任务的执行策略。
+ `public void enableJob()`: 获取Cron的配置，然后更新`ExecutionService`中的更行当前作业分片的时间
### JavaShardingItemCallable
![](http://oh8mi0yav.bkt.clouddn.com/JavaShardingItemCallable.png)

Java类型分片执行的上下文环境，在分片的基类定义的基础上，额外定义了Java分片执行的状态结果，分片执行作业采用额外的线程`currentThread`，同时定义了该线程的执行结果的状态，包括：

+ `INIT`: 初始化
+ `TIMEOUT`: 超时
+ `SUCCESS`: 成功
+ `FORCE_STOP`: 强制停止
+ `STOPED`: 主动停止

其中的方法如下：
+ `public static Object cloneObject(Object source, ClassLoader classLoader)`: 复制对象，这里利用反射的方式调用`copyFrom`的方法，具体传入的ClassLoader中必须有该方法，具体作用后续更新，暂时不是太明白。
+ `public Object getContextForJob(ClassLoader jobClassLoader)`: 根据父类的`shardingContent`生成分片的执行的上下文对象
+ `void reset()`: 重新初始化
+ `public void beforeExecution()`:
+ `public void afterExecution()`:
+ `public SaturnJobReturn call()`: 作业真正执行的地方，日志记录，线程开启，并对执行结果通过`protected void checkAndSetSaturnJobReturn`进行`SaturnJobReturn`的返回。
### ShardingItemCallable
![](http://oh8mi0yav.bkt.clouddn.com/ShardingItemCallable.png)

该类定义了分片上下文的基类，主要对当前执行分片的上下文的一些参数，执行服务和执行结果的便利定义，涉及作业名称，分片号，分片参数，超时时间，上下文和具体执行的实际Job类，前面的`JavaShardingItemCallable`就是在基础分片执行的上下文定义类中实现基于Java的分片执行的上下文。

### JobExecutionMultipleShardingContext
![](http://oh8mi0yav.bkt.clouddn.com/JobExecutionMultipleShardingContext.png)