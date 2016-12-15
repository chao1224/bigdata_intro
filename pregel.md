# Pregel: A System for Large-Scale Graph Processing

Pregel的计算是由seq of interation组成，叫做supersteps。

是vertex－centric program model。

输入是有向图，其中每一个vertex都有string ID标记。superstep之间是由global sync point分隔开。在每一步都superstep之内，vertex都是进行分布式计算。

vertices之间通过messages进行交流。

combiner: local aggregation of computation results

aggregator: ways of tracking global vertex state

实现determinism:
+ partial orering：对于mutation within superstep的情况，确定执行的顺序。先removal，再addition。而且addition的时候，先vertex addition，后edge addition。
+ handlers：对于这种情况：multiple requests create/remove same vertex/edge in the same superstep。Pregel会随机选择一个，就需要user defined-function，来特地确定顺序。

将graph划分到partition，每个partition由vertices和outgoing edges组成。

fault-tolerance：设置checkpoint，当某一个partition的n个worker宕机之后，将这个partition reassign、recompute。

master的活动通过barrier时会选择是否终结。

# Appendix

Pregel的缺点：
1. sync，不一定能converge。比如染色问题。
2. 类似Page Rank的算法，unpopular点很快converge，然而popular点会不断向unpopular点要数据。
3. Random Partition有潜在的效率问题
4. BSP模型，sync

比较Pregel和GraphX

|Pregel|GraphX|
|---|---|
|vertex-centric|RDD-centric|
|message passing|RDD|
|sync|sync/async|
|random partition|m-way balanced edge/vertex partition|
