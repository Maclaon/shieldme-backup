---
title: 分布式理论（四）
date: 2015-02-20 10:00:00
tags: 分布式理论，数据副本
author: maclaon
comments: false
---
# 基本副本协议
副本控制协议指按特定的协议流程控制副本数据的读写行为，使得副本满足一定的可用性和一致性要求的分布式协议。副本控制协议要具有一定的对抗异常状态的容错能力，从而使得系统具有一定的可用性，同时副本控制协议要能供一定一致性级别。由CAP原理可知，**要设计一种满足强一致性，且在出现任何网络异常时都可用的副本协议是不可能的**。

> 副本控制协议分为两大类: 中心化副本控制协议和去中心化副本控制协议

## 中心化副本控制协议
> 中心化副本控制协议的基本思路是由一个中心节点协调副本数据的更新、维护副本之间的一致性。

中心化副本控制协议的优点是协议相对较为简单，所有的副本相关的控制交由中心节点完成，并发控制将由中心节点完成，从而使得一个分布式并发控制问题，简化为一个单机并发控制问题。所谓并发控制，即多个节点同时需要修改副本数据时，需要解决“写写”、“读写”等并发冲突。单机系统上常用加锁等方式进行并发控制。对于分布式并 发控制，加锁也是一个常用的方法，但如果没有中心节点统一进行锁管理，就需要完全分布式化的锁系统，会使得协议非常复杂。如下图所示:

![](http://oh8mi0yav.bkt.clouddn.com/center-data-replic-control-protocal-in-distribute-system.png)

<!--more-->

### 优缺点
优点:
+ 副本一致性有单机的性质来控制，便于管理和协调

缺点:
+ 可用性依赖于中心化节点，当中心节点异常或与中心节点通信中断时，系统将失去某些服务(通常至少失去更新服务)，所以中心化副本控制协议的缺点正是存在一定的停服务时间。

### primary-secondary协议

在primary-secondary类型的协议中，副本被分为两大类，其中有且仅有一个副本作为primary副本，除 primary以外的副本都作为secondary副本。维护primary副本的节点作为中心节点，中心节点负责维护数据的更新、并发控制、协调副本的一致性。

> primary-secondary协议主要解决四大类问题: 数据更新流程、数据读取方式、Primary副本的确定和切换、数据同步。

#### 数据更新流程
primary-secondary协议的数据更新流程:
> + 数据更新都由primary节点协调完成
> + 外部节点将更新操作发给primary节点
> + primary节点进行并发控制即确定并发更新操作的先后顺序
> + primary节点将更新操作发送给secondary节点
> + primary根据secondary节点的完成情况决定更新是否成功并将结果返回外部节点。

![](http://oh8mi0yav.bkt.clouddn.com/primary-secondary-data-replica-protocal.png)

#### 数据读取方式
#### Primary副本的确定和切换


## 去中心化副本控制协议