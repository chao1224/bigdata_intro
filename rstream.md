# StreamScope: Continuous Reliable Distributed Processing of Big Data Streams

主要是两个数据结构：rStream和rVertex。

rVertex就是节点，rStream就是连边。

snapshot (# of input stream, # of output stream, current state)

failure recovery：宕机后，调用load(snapshot)

生成DAG过程：

1. 把program转换成logical plan（DAG）
2. optimizer然后评估不同的plan，选择代价最小的
3. 最终建立physical plan，通过将每一个logical vertex映射到数量合适的physical vertices

Determinism:

1. function determinism
2. input determinism

用CTI保证。multiple input streams：在每个vertex开始之前插入MERGE操作，产生deterministic(确定的)event序列。

recomputation会级联影响到upstream。

三种recovery的方法。
1. checkpoint-based recovery:vertex每隔一段时间把它的snapshot进行persistent存储。
2. relay-based recovery:很多时候，现在的状态值取决于前面的一小段时间，比如五分钟，所以只要recover这么一个small window即可。
3. replication-based recover:每一个vertex run同时跑好几个实例。