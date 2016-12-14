# GraphX: Graph Processing in a Distributed Dataflow Framework

Joseph E. Gonzalez, REynold S. Xin, Ankur Dave, Daniel Crankshaw, Michael J. Franklin, Ion Stoica

UCB amplab

# 1 Introduction

许多基于graph数据的专门应用，包括Pregel，PowerGraph。通过将有graph-specific操作支持的专门的抽象，可以很自然地表示类似PageRank的图迭代算法。因此graph processing系统比传统的一般化分布式数据流框架数量级上提升了性能，比如hadoop mapreduce。

这些已有的系统尽管性能上有所提升，但都是有一定的代价，graph只是更大的分析过程中的一部分，也就经常会将graph和unstructured data结合，最终这种pipeline会被迫包含许多不同的系统，并带来没有必要的数据移动和复制。而且这些系统通常没有很好的容错机制。

相比之下，general-purpose分布式数据流框架，比如Map Reduce和Spark，提供了丰富的数据流操作，对于unstructured and tabular数据支持比较好。但是直接实现iterative graph algorithm比较有挑战性，而且join和aggregation策略没有充分利用迭代图算法中一些共有的模式和结构，因此没有进行充分的优化。

graph processing的发展和分布式流处理框架是分离开发展的。分布式流出来框架distributed dataflow framework，早期s重点在每一个stage的计算，使得对于图算法中迭代步骤之间充数访问数据，随机访问subgraph的应用支持力度不够。而且他们没有提供细粒度的操作控制data parttitioning。但最新的spark，naiad提供了数据的分割，一定程度缓解了这个问题。

因此我们提供了一个模型，将这两者结合起来。我们在general purpose的分布式流处理系统上提供了graph processing系统的设计。GraphX，基于Spark的设计。

# 2 Background

graph processing system的计算都是在节点和他们邻居这里粒度上进行的，而general purpose的分布式流处理则是基于每一个item的粒度，或者是整个集合。

## 2.1 The Property Graph Data Model

graph processing system将图结果的数据当作property graph。包含了meta-data（用户信息和时间戳）和program state（节点的PageRank或者推断的关系）。这些图通常是高度skewed，边数比节点数大很多。这使得优化和分析变得更加困难。

## 2.2 The Graph Parallel Abstraction

类似PageRank和latent factor analysis的算法，会迭代的将节点属性进行传输给邻居，这种迭代式的本地传输是graph-parallel抽象的基础，用户定义的vertex program会同时为每一个节点实例化，并且和邻接节点进行交互。但所有节点都“投票”暂停时，程序结束。

每一个节点同时运行的同步性程度，有同步和异步两种。因为异步操作带来的便利会被额外的复杂性所平衡，所以我们先只考虑同步。

graph-paralle抽象对于迭代算法很帮助，但是对于没有连接或者改变图结构情况有局限，比如将图从文本中进行构建。

## 2.3 Graph System Optimizations

系统优化：

GAS Decomposition：大多数节点程序都和邻居交互，然后广播新的信息，他们提出了GAS decomposition，将节点程序分为三个data-parallel stages：Gather，Apply，和Scatter。

者带来了一个pull-based的信息处理模型。这样用户就不用专门给vertex发信息，但是没有连接的vertex直接也没法直接通信。几种优化方案。

Graph Partitioning：vertex-cut partitioning

Mirror Vertices：同一台机器上可能会有好几个互为neighbour的节点，那么就不需要发送多份message，只需要发送一份给一个mirror节点，然后mirror节点再把这些信息一起返回。

Active Vertices：算法会converge，有点节点比较稳定，所以每次更新的data set变少。一些系统是用来packed data-structures，比如compressed sparse row（CSR），使得访问周围active节点变快。

## 2.4 Distributed Dataflow Frameworks

用来指类似MapReduce的cluster compute frameworks。

# 3 The GraphX Programming Abstraction

把property graph data model当作collection，graph parallel asbtraction当作数据流操作的模式。

## 3.1 Property Graphs as Collections

vertex collection包含了vertex info，有vertex id区分

edge collection包含了edge info，由source和destination vertex id区分。

这样增加节点就好比增加一个source和destination的连边。分享数据只要将对于的collection进行join。这个是抽象层次上合并了。

## 3.2 Graph Computation as Dataflow Ops.

上面的抽象表示使得我们可以将graph放入到分布式数据流框架中。可以将graph-paralle计算表示成join和group-by，map的流。

group-by stage将所有信息收集到同一个节点上，map来apply新的属性，最后join操作来scatter信息给其他节点。

## 3.3 GraphX Operators

基于Spark的抽象，引入了一些特定的graph operators，列在Listing 4中。

# 4 The GraphX System

## 4.1 Distributed Graph Representation

GraphX将图表示成vertex和edge的collection,建立于Spark RDD抽象之上.

**vertex collection**:根据vertex id进行hash-partitioned.为了支持vertex之间的join,每一个partition内部的vertices根据本地哈希值进行排序.

**edge collection**:根据用户定义的函数进行哈希.GraphX允许使用vertex-cut partitioning,减小partition之间的communication.默认情况下,edge的partition是基于分配给input collection的partition,比如本来在HDFS上的partition.

## 4.2 Implementing the Triplets View

**vertex mirroring**: 因为vertex和edge的partition是独立的,每次做join的时候要将数据进行转移.GraphX提供了three-way join.讲vertex属性发送给edge,因此讲edge partition设置为join sites.这是因为edge的数量远大于vertex.

**multicast join**: 因为每次join只需要一小部分数据,所以每一个vertex property只会发送给包含了它的邻居的edge partition.

**Partial Materialization**: 当vertex property改变的时候,vertex replication操作比较频繁,但edge partition的local joins会unmaterialized,来避免冗余.

**incremental view maintenance**: 到后期每次只修改部分数据,使用incremental view maintenance在triplets view,避免没有必要的数据移动.每一次图操作之后,追踪每一个节点是否改变.在triplets view下一次访问时候,只有改变的节点路由到edge partition进行join操作.

## 4.3 Optimization to mrTriplets

### 4.3.1 Filtered Index Scanning

第一步是浏览triplets,给每一个triplet apply用户定义的map函数.但是当一个iterative graph algorithm converge时,working set开始shrink,map funcion会跳过大部分triplet,只需要对那些active vertex进行.每次直接浏览所有的triplet变得浪费.

为了解决这个问题,引入了indexed scan.应用将当前还active的集合通过subgraph操作进行限制.vertex为此会被pushed到edge partition中,对triplets,通过CSR index进行过滤(source vertex id上的).

实验表示,刚开始这种做法会消耗的时间多一点,和sequential scan比,但是到后面converge的时候会迅速下降.Fig 5.

### 4.3.2 Automatic Join Elimination

用了JVM bytecode analyzer来检测用户定义的函数,来检测target vertex property是否会因为用户定义的函数而改变.如果只有一个property被改变,而且triplet没有实体化,GraphX会重写query plan,从three-way变为two-way.如果没有property被改变,那么整个join被取消.

感觉这个就是对自己提出的three-way进行优化,在传统two-way join更好的情况下进行switch.通过JVM bytecode analyzer来判断什么时候two-way更好,什么时候three-way更好.

## 4.4 Additional Optimization

**Memory-based Shuffle**: Spark默认的shuffle会将临时数据materialize到disk.我们进行了修改,materialize map output到内存中,然后用timeout来删除临时数据.

**Batching and Columnar Structure**:join的时候,有一个将vertice shuffle的操作.一种是一个个shuffle,这里将要route同一个target的节点batch进行shuffle.然后用LZF压缩算法进行压缩,然后发送.

**Variable Integer Encoding**:尽管vertex ids有64位,但是远远到不了.经过试验,差不多只要5个bytes进行编码.

# Appendix

比较graph parallel model和general-purpose distributed dataflow framework


|  |Graph Data Model| Distributed Dataflow Framework |
|---|---|---|
|1|使用property graph作为representation|转换为vertex collection和edge collection|
|2|operations are defined with respect to a single property|operations can span multiple collection，比如shuffle|
|3|用户自定义函数被用来实例化每个节点，并且和邻居交流| |
|4|对于某些计算不合适，比如没有直接连的节点之间的交互或者是改变图的结构| |

优缺点比较

优点

1. combine graph + regular processing
2. easier to express computations that change graph structures
3. falut tolerance: with support of RDD, much clearner -set of failing workers

缺点

1. unnatural to fit for general graph parallel algorithm
2. performance loss due to generality

