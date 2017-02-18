---
title: The Definitive Guide of Elasticsearch
date: 2015-07-17 10:00:00
tags: NoSQL,Elasticsearch, Conflict
author: maclaon
comments: false
---
# 数据冲突
这里主要介绍`Elasticsearch`组件是如何解决类似于数据库中多个写同一份数据的问题，假设有两个或两个以上的操作对`Elasticsearch`中的某一个索引进行修改，如何保障每次修改的数据是的同步性，其实`Elasticsearch`的操作手段类似于Java并发中利用循环CAS来保证数据的可操作性，循环CAS是一种乐观锁。

## 悲观锁和乐观锁
前面在介绍Java的[Java并发机制的底层实现原理](http://shieldme.cn/2015/07/08/the-art-of-java-concurrency-programming-underlying-implemetation/)中，详细介绍了悲观锁和乐观锁的特性，其中乐观锁主要是不阻塞当前操作，如果读写之间出现冲突，那么处理冲突的方式是重试，还是直接返回给客户端。

## Elasticsearch冲突解决
因Elasticsearch是分布式，document经过创建，更新，删除形成一个新的数据版本，最终这个数据会通过副本的方式将数据传递到其他的节点，到达节点的时候可能是无序的了，`Elasticsearch`需要一种方式确定老版本的数据永远不会覆盖新版本的数据。

每当我们创建，更新的数据，我们都会给document带上`_version`的版本号，如果一个老版本的数据低于目前的新版本，那么该操作是被忽略的，一般情况有重试，或者返回conflict状态给用户。

<!--more-->

elasticsearch的`_version`可以由外置的信息来进行设置，可以是数据的timestamp，可以是自己自定义状态的数据。