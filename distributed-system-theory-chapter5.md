---
title: 分布式理论（五）
date: 2015-03-07 10:00:00
tags: 分布式理论，Lease机制
author: maclaon
comments: false
---
# Lease机制
## 背景
基本的问题背景如下: 在一个分布式系统中，有一个中心服务器节点，中心服务器存储、维护着一些数据，这些数据是系统的元数据。系统中其他的节点通过访问中心服务器节点读取、修改其上的元数据。由于系统中各种操作都依赖于元数据，如果每次读取元数据的操作都访问中心服务器节点，那么中心服务器节点的性能成为系统的瓶颈。为此，设计一种元数据cache，在各个节点上cache元数据信息，从而减少对中心服务器节点的访问，提高高性能。另一方面，系统的正确运行严格依赖于元数据的正确，这就要求各个节点上cache的数据始终与中心服务器上的数据一致，cache中的数据不能是旧的脏数据。最后，设计的cache系统要能最大可能的处理节点宕机、网络中断等异常，最大程度的高系统的可用性。

## Lease
中心服务器在向各节点发送数据时同时向节点颁发一个lease。每个lease具有一个有效期，和信用卡上的有效期类似，lease上的有效期通常是一个**明确的时间点**，例如12:00:10，一旦真实时间超过这个时间点，则lease过期失效。 这样lease的有效期与节点收到lease的时间无关，节点可能收到lease时该lease就已经过期失效。 这里首先假设中心服务器与各节点的时钟是同步的，下节中讨论时钟不同步对lease的影响。中心服务器发出的 lease的含义为: **在lease的有效期内，中心服务器保证不会修改对应数据的值**。因此，**节点收到数据和lease后，将数据加入本地Cache，一旦对应的lease超时，节点将对应的本地cache数据删除。中心服务器在修改数据时，首先阻塞所有新的读请求，并等待之前为该数据发出的所有lease超时过期，然后修改数据的值**。

<!--more-->### 基于lease机制的Cache的读写流程
#### 客户端读取元数据
![](http://oh8mi0yav.bkt.clouddn.com/read-flow-on-lease-theory-in-distribute-system.png)
#### 客户端更新元数据
+ 节点向服务器发起修改元数据请求。
+ 服务器收到修改请求后，**阻塞**所有新的读数据请求，即接收读请求，但不返回数据（**防止发出新的 lease 从而引起 不断有新客户端节点持有 lease 并缓存着数据,形成“活锁”**）。
+ 服务器等待所有与该元数据相关的lease超时。
+ 服务器修改元数据并向客户端节点返回修改成功。

#### 优化点
上述机制在性能和可用性的基础上有些问题，服务器在修改元数据时首先要阻塞所有新的读请求，造成没有读服务。这是为了防止发出新的lease从而引起不断有新客户端节点持有lease并缓存着数据，形成“活锁”。优化点如下:
> + 服务器在进入修改数据流程后，一旦收到读请求则只返回数据但不颁发lease。从而造成在修改流程执行的过程中，客户端可以读到元数据，但是不能缓存元数据。
> + 进一步优化，进入修改流程，服务器颁发的lease有效期限选择为已发出的lease的最大有效期限。这样的话，客户端可以继续在服务器进入修改流程后继续缓存元数据，但服务器的等待所有lease过期的时间也不会因为颁发新的lease而不断延长。

上述机制可以保证各个节点上的cache与中心服务器上的中心始终一致，中心服务器节点在发送数据的同时授予了节点对应的lease，在lease有效期内，服务器不会修改数据，从而客户端节点可以放心的在lease有效期内 cache数据。上述lease机制可以容错的关键是: **服务器一旦发出数据及lease，无论客户端是否收到，也无论后续客户端是否宕机，也无论后续网络是否正常，服务器只要等待lease超时，就可以保证对应的客户端节点不会再继续cache数据，从而可以放心的修改数据而不会破坏cache的一致性。**

