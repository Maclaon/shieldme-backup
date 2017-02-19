---
title: Elasticsearch中分布式文档的存储方式
date: 2015-07-18 10:00:00
tags: document store,elasticsearch,分布式
author: maclaon
comments: false
---
# 分布式Document存储
## 分片上的Document路由
> 建立一个索引的时候，Elasticsearch是如何知道该索引是放在集群中哪个`shard`分片上的？

事实上，Elasticsearch在进行数据在分片上存储的时候，是基于固定的路由方程式来做的：

$$D_{shard} = hash(\;V_{routing\;})\; \% \; N_{number\_of\_primary\_shards}$$

上述的$$V_{routing}$$是一个任意的字符串，默认的话是document中对应`_id`的值，但是可以根据需要设置成业务相关的值。通过对$$V_{routing}$$进行$$hash$$函数产生一个数字（一个字符串的映射到整数的功能性），并且通过求余数的方式将该document具体落到哪个分片上进行确定，上述的求余数的值域在$$0$$到$$N_{number\_of\_primary\_shards}\;-\;1$$，这样就可以从这样的一个路由函数的基础上对数据进行查询。

上述的路由公式就解决了为什么主分片的数量在设置以后就不允许在改变的原因，如果主分片的数量被改变了，那么久无法通过上述的路由公式去准确的定位一个document在哪个分片上存储。

<!--more-->

### 整体性学习
> 上述的通过一个字符串进行hash的思路，可以准确的应用到数据分布的基础上，比如面试题:50亿的url，如何找出其中不同的url数据，内存大小只有$$4G$$，那么就可以采用上述思路对url进行hash，然后除以1000，这样判断不同的数据只要在1000的数据范围来进行即可，涉及到分段加载，数据分隔的问题。

## Primary和Replica分片的交互
假设集群中有三个节点，包含blogs的索引和$$2$$个primary分片，每个主分片有$$2$$个replica分片，主分片和副本分片不会同时在一个节点上。如下图所示:

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0401.png)

### 写操作(Creating,Indexing,Deleting)
因Elasticsearch采用的是中心化副本协议primary-secondary，所以所有的写操作必须在primary分片上完成后，才能写对应的replica分片，写replica副本分片的时候采用[Quorum](http://shieldme.cn/2015/07/01/distributed-system-theory-chapter9/)，进行写操作的过程的相关操作步骤如下：

![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0402.png)

步骤如下:
> + 客户端发送写请求到节点1上
> + 节点1通过document的`_id`决定该document的值属于$$shard 0$$，他会将请求转发到$$shard 0$$主分片primary-shard所在的节点3上。
> + 节点3执行该请求，如果成功，会并行的将请求转发到存在于节点1和节点2的副本上执行，当节点1和节点2都执行成功后，节点3会将写成功的信息转发到请求节点1上，请求节点1会将结果返回给客户端。

上述的步骤是最原始的执行，因采用了primary-secondary的机制，可以在请求的中设置类似于[Quorum](http://shieldme.cn/2015/07/01/distributed-system-theory-chapter9/)透出的相关数据。

### 获取分片
![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0403.png)

获取分片的过程如下:
> + 客户端发送一个查询请求到节点1上
> + 节点1使用document的`_id`去决定该document是数据域$$shard 0$$上。副本$$shard 0$$存在于所有的集群节点上，在这种情况下，请求将会被转发到节点2进行处理，采用轮训的方式去处理，不在主分片节点上去执行的另外一个原因是可以，提高性能。
> + 节点2将document的结果返回给节点1，节点1再将数据集返回给客户端。

### 增量更新
![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0404.png)

增量更新的过程如下:
> + 客户端发送请求到节点1
> + 节点1将请求转发到节点3上，因为主分片是在节点3上
> + 节点3从主分片上获取对应的document，改变Json中的`_source`字段，并且在主分片上重新建立索引，如果此时发生[数据冲突](http://shieldme.cn/2015/07/17/crud-conflict-dealing-of-the-definitive-guide-of-elasticsearch/)，那么会根据`retry_on_conflict`的值重试该步骤。
> + 如果节点3更新该document成功，会带着新的版本`_version`到节点1和节点2上进行重建，一旦所有的副本都执行成功，节点3将成功结果返回该请求节点，请求节点在将数据结果返回给客户端。

### 批量操作
批量操作`mget`和`bulk`的解开操作类似于上述的单个操作，不同之处是请求节点明确的知道每个document存在，将多个请求打散在每个分片上，并用分片的并行的请求参与的节点。

#### mget
![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0405.png)

操作流程如下:
> + 客户端发送`mget`请求到节点1上
> + 节点1在每个分片上构建`mget`请求，并且将这些请求并行的发送给哪些需要进行相关查询操作的主分片和副本分片，一旦所有的请求都收到了，节点1构造返回结果，并将结果返回给客户端。

#### bulk
![](https://www.elastic.co/guide/en/elasticsearch/guide/current/images/elas_0406.png)

操作流程如下:
> + 客户端发送一个`bulk`请求到节点1上
> + 节点1用每个分片构建一个`bulk`请求，并将这些请求并行的发送到与之有关的主分片上。
> + 主分片一个接着一个执行对应的动作，每一个动作执行成功后，主分片都会将新的副本分片并行的发送到副本分片上，接着执行下一个动作。一旦所有的副本分片对所有的执行结果报告成功，节点将成功的返回发送到请求节点，请求节点整理所有的返回结果，并将它们返回给客户端。



