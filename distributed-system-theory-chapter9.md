---
title: 分布式理论（九）
date: 2015-06-31 10:00:00
tags: 分布式理论，Quorum机制
author: maclaon
comments: false
---
Quorum机制
> 某系统有$$5$$个副本，$$W=3、R=3$$，最初5个副本的数据一致，都是$$v_1$$，某次更新操作$$w_2$$在前$$3$$副本上成功，副本情况变成$$(v2 v2 v2 v1 v1)$$。此时,任意$$3$$个副本组成的集合中一定包括$$v_2$$。