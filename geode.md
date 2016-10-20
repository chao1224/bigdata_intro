# Global analytics in the face of bandwidth and regulatory constraints

# 1 Introduction

大公司的数据都会分布式地存储在全球各地，因为面向用户的应用，要避免延迟性太高的问题。数据量每天以PB级地增长。这种新的建立在数据中心上的global DB，带来了新的问题。最近的研究有Spanner，Mesa，JetStream。这些应用中，数据都是频繁的访问来挖取信息，用ad-hoc和复发的analytical queries。大公司都运行数十甚至数百的PB级数据。许多操作都是OLAP操作。

现在的公司都通过转移数据，将其集中到某一个data center，然后再data center里面用single-cluster技术，比如relational data warehouse或者Hadoop-based stacks（比如Hive）。但数据量一旦很大，中心化处理的方法会转移大量数据。而整个网络的带宽是有限的。而且与数据量增加相比，网络带宽的增长比较慢。

另外还有privacy的问题。当地政府会进行很多限制。

因此我们的中心议题是：随着数据增长，中心化的解决方案会遇到一个拐点，会将不再有效和可惜。先考虑提供wide area analytics的同时，减小带宽——这种分析叫做WABD，Wide-Area Big Data。（暂时忽略延迟）

以下四个主要贡献：
1. Oprimizer：联系在一起的，优化query执行计划和数据复制，来减小带宽消耗。解决方案同时用了经典中心化query planner和integer program来处理geo-distribuion。
2. Pseudo-distributed measurement：这个技术可以改善query执行，来收集真实数据传输的大小，增加每个data center内部（cheap）计算，但不会使得贵的跨data center传输增加。
3. Subquery Delta：利用单个data center内便宜的存储和计算来进行cache所有的中间结果，利用delta来减少数据传输的冗余。
4. Demonstrated Gains：原型prototype，Geode，是建立在Hive上。在benchmark上测试，达到了250倍地数据传输减少，和360倍的提升（在TPC-CH和Berkeley Big Data测试）

# 2 Approach Overview

通过讨论Berkeley Big Data中的一个例子来开始讨论。

**Running example**

我们有一个数据库存储bach-computed page的元数据和用户访问的日志，包括每次访问带来的收入。

```
ClickLog(sourceIP,destURL,visitDate,adRevenue,...)
PageInfo(pageURL,pageSize,pageRank,...)
```

pages在几个data center中被复制了多份，给用户提供的是最近的page copy。访问日志是自动存储到离用户最近的data center，所以`ClickLog`表分散在几个data center。`PageInfo`表会存储在一个master data center，不定期的通过内部的batch job更新。

现在考虑一个例子，分析产生至少$100广告收入的用户（通过IP地址区分）。

centralized方法会需要把所有`ClickLog`表格的更新到中心data center，在这个data center进行分析。这意味着每天的网络带宽几乎是1.2TB。而Geode提供了location-independent的查询接口，基于分布式数据。Geode接受查询语句，自动划分查询，安排分布式的执行。它构建分布式计划分为两步：

**1. Choose join order and strategies** Geode先为逻辑query Q简历一个physical execution plan，明确table join的顺序和distributed join算法的选择。例子里面只有一个根据`destURL=pageURL`的join。

为了做这些选择，我们用`Calcite++`，基于Apache Calcite中心化SQL query planner。增加了distributed join的算法。

当Calcite++运行在Q上的时候，它会输出一个`JOININT(strategy=right_broadcase)`，表示这个join应该通过吧`PageInfo`表格广播给其他的data ccenter，每一个data center都有部分的`ClickLog`表格，然后在每一个data center内部执行本地join。

假设physical plan转换成DAG图如图2，每条边展示了依赖。任务可以读取base数据划分和/或从其他任务做完输入。所有任务需要的输入都必须是或者已经在或者是从其他data center复制的。尽管在这个例子里面不用考虑，监管限制会组织一些划分复制到某些data center，因此限制了任务调度。

**2. Schedule tasks** Geode需要将人物分配给data center，考虑输入的依赖和基本的数据监管限制。

Geode可以保存base data的几份copy，为了提高性能和容错，以及可以支持将一个任务的多份copy的不同partition同时执行。比如在例子中，可以保存一份`PageInfo`表的同步在每一个data center，然后在每一个data center创建task `q1`的多份copy。复制策略的选择是通过workload optimizer控制的，（改变复制策略）比单独一个查询的执行时间长很多（复制策略通常是每周甚至更久）。

运行时，Geode通过解决一个整数线性优化问题Integer Linear Problem（ILP）来给每一个query分配任务，$$x_{td}=1$$ iff 某一个任务t的一个copy被分配在了data center d。ILP的constraints明确了每一个任务输入的依赖性和每一个data center里数据的监管限制。ILP最小化DAG图中任务之间的数据转移。

假设初始的设置是数据还未被复制。一个自然的想法是把data center里的$$q_1$$持有`PageInfo`表，将$$q_2,q_3,q_4$$push到edge data center，各自持有部分的`ClickLog`数据划分，然后一起定位到$$q_5$$，可以是$$q_2,q_3或q_4$$。如果查询Q每天执行，那么差不多每天要传输14GB的数据。

**Architecture**

Geode在好几个数据中心上进行分析数据，因为和用户的交互，经常更新。用户端的交互并没有在我们系统内实现。假设每一个数据中心里的数据都被放到了标准的single-data-center analytics stack，比如Hive或者RDB。现有的实现就是基于Hive的。

系统的核心就是central command layer。command layer接受SQL查询，将它们变成分布式查询执行计划，执行计划，和收集最后的结果。每一个数据中心中，central command layer都和一个很小的proxy交互。proxy部署在本地的analytical stack，并且帮助数据在data center之间转移、管理本地chache来保存查询的中间结果。

workload optimizer从central command layer获取测量结果，来判断改变query plan或者数据复制策略能提高整体性能。这次性能会通过使用我们的pseudo-distributed执行方法来收集，意味着重写queries。optimizer不会直接使用修改的方案，而会咨询另外一个administrator。

# 3 Subquery deltas: Reducing data transfer

# 4 Workload Optimizer

# 

