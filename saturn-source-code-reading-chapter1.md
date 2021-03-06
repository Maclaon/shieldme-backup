---
title: 分布式任务调度框架Saturn研究（一）
date: 2016-12-10 11:39:37
tags: 分布式，Zookeeper，任务调度，唯品会
comments: false
---
## 题记
![](https://raw.githubusercontent.com/vipshop/Saturn/doc/saturn-logo.jpg)
本文主要分析唯品会基于当当网elastic-job进行改造的分布式任务调度系统Saturn来进行源码阅读，用于**任务调度，定时任务和半自动化运营以及爬虫等方面，旨在增加运营效率和数据产出！**在文档书写的过程中沿用原有的term，并在此基础上进行扩展和延伸。
<!--more-->
## 术语
> Job（作业）: 可以独立进行脚本运行的，不局限于各种语言或者具备某项功能的函数实现Java，C++等功能实现。作业可以并发执行在多个执行节点上（Executor），执行节点必须配备需要执行的环境Python，Phantomjs等，作业的分片和目标执行机器定义一个作业并发执行的数量。如：作业A有5个分片（0，1，2，3，4），一共有两个执行节点E0和E1，那么可以得出：0，1，2指派给E0；3，4指派给E1。其中的分片算法可以自我调整，上述的模型可以认同为，**一个程序在单机情况下的多线程并发的执行问题**。   
> Namespace（域）：一组特定的执行节点和作业。便于对不同执行节点功能的，比如说：定时报表，爬虫，数据库导入等特定机器，可以在设计上在客户端执行节点的时候加入不同的域来操作。执行节点必须隶属于某一个特定的域。域下的任何一个节点都有能力执行域下的全部作业，换言之，作业可以在域下任何一个执行节点执行。   
> Saturn: 定时任务调度系统。   
> 执行节点（Executor）：调用并执行作业的程序。通过定时（Quartz）驱动出发调用时间，并最终调用作业的执行入口，执行节点始终执行作业分片，当前作业无法执行时，调用域下的其他执行节点进行执行，当全部域下的执行节点都不可用的时候，放入等待队列，等待域的执行。**业务方在使用的过程中也可以不基于当前的执行节点来进行操作，可以接入定时调度的策略，在自己应用的服务器上进行定时任务的执行操作。**
> 控制台（Console）：后台管理平台，可以使用控制台进行的数据查看和作业状态以及执行日志等。

## 基本原理
![](https://raw.githubusercontent.com/vipshop/Saturn/doc/architect/overall.png)
Saturn的基本原理是将作业在逻辑上划分为若干个作业分片，通过**作业分片调度器**将作业分片指派给特定域的执行结点。执行结点通过quartz触发执行作业的具体实现（以shell为例，则为shell脚本)，在执行的时候，会将分片序号和参数作为参数传入(见上图)。 作业的实现逻辑需分析分片序号和分片参数，并以此为依据来调用具体的实现（比如一个批量处理数据库的作业，可以划分0号分片处理1-10号数据库，1号分片处理11-20号数据库）。

## 系统逻辑架构
![系统逻辑架构图](https://raw.githubusercontent.com/vipshop/Saturn/doc/architect/logic.png)
## 组件说明
### 执行节点
执行节点主要负责分配分片作业的定时执行，结果上报，日志记录，告警上报，监控记录等功能，属于可执行的Client节点，可以单独的进行部署，也可以进行集群的管理，执行节点部署后会向zookeeper的集群管理中心进行注册。执行节点的执行方式可以是进群机器上单独执行，jar包方式执行，业务方主动调用执行和特殊节点client环境下如：爬虫需要PhantomJs等依赖工具的CS架构的执行。**总之，执行节点的地方多种多样，Saturn主要负责任务分片的分发与下发，至于如何执行需要根据业务方自己的业务场景来定义**。   

![Saturn Executor](https://raw.githubusercontent.com/vipshop/Saturn/doc/architect/executor.png)

1.  **JobScheduler**   
执行节点负责从Zk中读取作业配置信息，资源信息（比如：作业类型【Java或者Python或者爬虫或者报表或者消息作业】，作业cron表达式，分片参数，用户账号，IP等），执行节点会根据不同的作业类型或者加载所需要的资源信息启动处理不同的业务逻辑。

2.  **分片加载（重新加载）**    
检查有没有收到作业分片调度器发出的重新加载指令(通过ZK结点)，如果有，则全部执行结点都须等待全部分片执行完成，然后由其中一个执行结点从任务调度器的调结果中读取本作业的分片指派结果， 并将结果广播（使用ZK的监听机制）给本作业的全部执行结点。  
3. **作业触发**   
作业根据cron表达式定义的时间规则进行周期性触发，每次触发周期到达时，先进行分片加载（重新加载），之后根据作业类型调用Job Runner。 Job Runner取出分片序列，解析分片参数（分片参数可通过控制台配置)； 比如java作业，启动线程池，将分片序列和分片参数作为调用参数构造作业执行线程，将作业执行线程提交到线程池执行；Job Runner启动作业之后，如果作业配置了超时时间，则需进行超时检测，一旦检测到超时，需要做超时处理（作业中止，状态上报）。 

### 作业分片调度器
![作业分片调度器](https://raw.githubusercontent.com/vipshop/Saturn/doc/architect/scheduler.png)   

Saturn的”大脑“，其基本功能是将作业分片指派到执行结点。通过调整分配算法和分配策略，**策略可支持业务方自己定制**，可以将作业合理地安排到合适的执行结点，从而实现HA，负载均衡，动态扩容，作业隔离，资源隔离等治理功能。 作业分片调度器为后台程序，单独部署；它是公共资源，所有域共用同一套作业分片调度器。 接入作业后，会自动接受作业分片调度器的调度。调度器遍历全部域，为每个域都生成域作业分片调度器。域作业分片调度器的处理步骤为：ZK事件监听，事件分发和处理，结果下发。   
  
1. **ZK事件监听**   
执行结点上线后会在ZK注册一个临时结点，下线后ZK会话超时临时结点会被清除。 作业启用和禁用状态为持久化结点，由管理员通过控制台修改（从启用改为禁用，或者从禁用改为启用）。 Saturn作业分片调度器监听这些结点的变化，根据结点路径和结点的值判断出事件类型进行事件分发。
2. **事件分发**   
Saturn作业分片调度器处理以下4类事件：结点上线，结点下线，作业启用，作业禁用。 分别对应结点上线处理类，结点下线处理类，作业启用处理类，作业禁用处理类四个模块。 事件处理类的作用是进行作业分片调度，将作业分片指派给执行结点。并将指派结果保存到ZK。
3. **结果下发**

### Zookeeper
集群管理员，分布式系统中必备的组件，解决分布式系统中一致性的问题，Zookeeper可以相当于分布式锁，解决分布式机器中数据、文件同步的问题。Zookeeper在这里的主要作用是对不同执行节点或者不同业务使用方的任务分片，调度策略，节点执行结果，监控数据和容错情况下的各种数据进行登记，所有的数据交互都需经过Zookeeper来进行注册和下发。

## 总结
Saturn在实现的基础上通过Zookeeper来进行全局作业的配置和管理，执行节点目前主要支持Java和Shell的执行方式，提供独立启动的以及jar包业务方集成的方式，其执行节点运行主要是Java和脚本的方式，目前支持大部分的任务类型。Saturn在设计上沿用了分布式的思想在里面，在执行节点上没有做更多的扩充和补充，后续可以根据业务场景的需要定制化执行节点，脱离业务的同时，又很好的支持业务。
