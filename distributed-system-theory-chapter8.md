---
title: 分布式理论（八）
date: 2015-06-10 10:00:00
tags: 分布式理论，Paxos协议
author: maclaon
comments: false
---
# Paxos协议
Paxos协议是少数在工程实践中证实的强一致性，高可用的去中心化分布式协议。流程较为复杂，但其基本思想类似于人类社会的投票过程：有一组完全对等的参与节点，这组节点各自就某一事件做出决议，如果某个决议获得了超过半数节点的批准则生效。Paxos只要有超过一半的节点正常，就可以工作，能很好的对坑宕机，网络分化等异常情况。

## 协议描述
### 节点角色
> **Proposer**: 提案者，可以有一个或者多个，Proposer提出议案(value)，这里的value可以是操作，比如：修改某个变量的值为某个值、进行2Mb数据的传输、设置当前节点为primary节点等等。不同的Proposer可以提出不同的甚至是矛盾的value，例如：某个Proposer提议“将变量X设置为1”,另一个Proposer提议“将变量X设置为2”，但对于同一轮Paxos过程，最多只有一个value被批准。
> **Acceptor**: 批准者，有$$N$$个，Proposer提出的value必须获得超过半数$$(N/2 + 1)$$的批准后才能通过。Acceptor之间完全对等独立。
> **Learner**: 学习者，学习被批准的value，所谓学习就是通过读取各个Proposer对value的选择结果，如果某个value被超过半数Proposer通过，则Learner学习到了这个value。这里类似Quorum机制，某个value需要获得$$W=N/2 + 1$$的Acceptor批准，从而学习者需要至少读取$$N/2+1$$个Accpetor，至多读取$$N$$个Acceptor的结果后,能学习到一个通过的value。上述三个角色在逻辑上的划分，实践中一个节点可以同时充当这三类角色。
<!--more-->

## 竞争及活锁
Paxos协议的过程类似于“占坑”，哪个value把超过半数的“坑”(Acceptor)占住了，哪个value就得到批准了。这个过程也类似于单机系统并行系统的加锁过程。假如有这么单机系统:系统内有$$5$$个锁，有多个线程执行，每个线程需要获得$$5$$个锁中的任意$$3$$个才能执行后续操作，操作完成后释放占用的锁。我们知道，上述单机系统中一定会发生“死锁”。例如: $$3$$个线程并发，第一个线程获得2个锁，第二个线程获得$$2$$个锁，第三个线程获得$$1$$个锁。此时任何一个线程都无法获得$$3$$个锁，也不会主动释放自己占用的锁，从而造成系统死锁。**但在Paxos协议过程中，虽然也存在着并发竞争，不会出现上述死锁。这是因为Paxos 协议引入了轮数的概念，高轮数的Paxos提案可以抢占低轮数的Paxos案。从而避免了死锁的发生，然而这种设计却引入了“活锁”的可能，即Proposer相互不断以更高的轮数提出议案，使得每轮Paxos过程都无法最终完成,从而无法批准任何一个value**。