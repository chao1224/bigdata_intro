# Pregel: A System for Large-Scale Graph Processing

Pregel的计算是由seq of interation组成，叫做supersteps。

输入是有向图，其中每一个vertex都有string ID标记。superstep之间是由global sync point分隔开。在每一步都superstep之内，vertex都是进行分布式计算。

vertices之间通过messages进行交流。

实现determinism:
+ partial orering：确定执行的顺序，先removal，再addition。而且addition的时候，先vertex addition，后edge addition。
+ handlers：解决有好几个vertex creating/removal同时发生。Pregel会随机选择一个，就需要user defined-function，来特地确定顺序。

将graph划分到partition，每个partition由vertices和outgoing edges组成。

fault-tolerance：设置checkpoint，当某一个partition的n个worker宕机之后，将这个partition reassign、recompute。

master的活动通过barrier时会选择是否终结。