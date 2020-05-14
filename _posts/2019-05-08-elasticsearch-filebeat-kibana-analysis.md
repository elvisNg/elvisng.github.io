---
layout: post
title: "Elasticsearch-Filebeat-Kibana-Analysis"
subtitle: '主流日志框架应用'
author: "Elvis"
header-style: text
tags:
  - ELK
  - 技术分享
---

# Elasticsearch Analysis

#### ES集群构成
一个Elasticsearch集群(下面简称ES集群)是由许多节点(Node)构成。

Node可以有不同的类型,四种不同类型的Node是一个node.master和node.data的true/false的两两组合。

当node.master为true时，其表示这个node是一个master的候选节点，可以参与选举，在ES的文档中常被称作master-eligible node，类似于MasterCandidate。ES正常运行时只能有一个master(即leader)，多于1个时会发生脑裂。

当node.data为true时，这个节点作为一个数据节点，会存储分配在该node上的shard的数据并负责这些shard的写入、查询等。

此外，任何一个集群内的node都可以执行任何请求，其会负责将请求转发给对应的node进行处理，所以当node.master和node.data都为false时，这个节点可以作为一个类似proxy的节点，接受请求并进行转发、结果聚合等。


![](/img/in-post/cluster-demo.png)

图1-1


上图是一个ES集群的示意图，其中Node_A是当前集群的Master，Node_B和Node_C是Master的候选节点，其中Node_A和Node_B同时也是数据节点(DataNode)，此外，Node_D是一个单纯的数据节点，Node_E是一个proxy节点。

#### 节点发现

Node启动后，首先要通过节点发现功能加入集群。ZenDiscovery是ES自己实现的一套用于节点发现和选主等功能的模块，没有依赖Zookeeper等工具，

[官方ZenDiscovery](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/modules-discovery-zen.html)


简单来说，节点发现依赖以下配置：

conf/elasticsearch.yml:

>   discovery.zen.ping.unicast.hosts: [1.1.1.1, 1.1.1.2, 1.1.1.3]

这个配置可以看作是，在本节点到每个hosts中的节点建立一条边，当整个集群所有的node形成一个连通图(如图1-1)时，所有节点都可以知道集群中有哪些节点，不会形成孤岛。


#### Master选举

**master选举谁发起，什么时候发起**

ES采用了常见的分布式系统思路，保证选举出的master被多数派(quorum)的master-eligible node认可，以此来保证只有一个master。这个quorum通过以下配置进行配置：

conf/elasticsearch.yml:
>   discovery.zen.minimum_master_nodes: 2

master选举是由master-eligible节点发起，当一个master-eligible节点发现满足以下条件时发起选举：

* 该master-eligible节点的当前状态不是master。
* 该master-eligible节点通过ZenDiscovery模块的ping操作询问其已知的集群其他节点，没有任何节点连接到master。
* 包括本节点在内，当前已有超过minimum_master_nodes个节点没有连接到master。

*总结一句话，即当一个节点发现包括自己在内的多数派的master-eligible节点认为集群没有master时，就可以发起master选举。*


**当需要选举master时，选举谁**

选举的是排序后的第一个MasterCandidate(即master-eligible node)。

```
  public MasterCandidate electMaster(Collection<MasterCandidate> candidates) {
    assert hasEnoughCandidates(candidates);
    List<MasterCandidate> sortedCandidates = new ArrayList<>(candidates);
    sortedCandidates.sort(MasterCandidate::compare);
    return sortedCandidates.get(0);
}
```
排序方法如下源码所示

```
public static int compare(MasterCandidate c1, MasterCandidate c2) {

// we explicitly swap c1 and c2 here. the code expects "better" is lower in a sorted
// list, so if c2 has a higher cluster state version, it needs to come first.

int ret = Long.compare(c2.clusterStateVersion, c1.clusterStateVersion);

if (ret == 0) {
    ret = compareNodes(c1.getNode(), c2.getNode());
}
return ret;
}
```
注：根据节点的clusterStateVersion比较，clusterStateVersion越大，优先级越高。(clusterStateVersion高代表节点越早加入)。clusterStateVersion相同时，进入compareNodes，其内部按照节点的Id比较(Id为节点第一次启动时随机生成)。




## ES分布式的数据一致性问题

ES 数据并发冲突控制基于的乐观锁和版本号的机制
一个document第一次创建的时候，它的_version内部版本号就是1；以后，每次对这个document执行修改或者删除操作，都会对这个_version版本号自动加1；
哪怕是删除，也会对这条数据的版本号加1(假删除)。
客户端对es数据做更新的时候，如果带上了版本号，那带的版本号与es中文档的版本号一致才能修改成功，否则抛出异常。如果客户端没有带上版本号，首先会
读取最新版本号才做更新尝试，这个尝试类似于CAS操作，可能需要尝试很多次才能成功。

es节点更新之后会向副本节点同步更新数据(同步写入)，直到所有副本都更新了才返回成功。


