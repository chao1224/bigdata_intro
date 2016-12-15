# Strom @Twitter

storm是实时、容错、分布式流数据处理系统。有五个特性

1. scalable
2. resilient
3. extensible
4. efficient
5. easy to administer

topology：是一个有向图，节点表示计算，边表示数据流

architecture：在整个topology中的tuple stream

vertice分为两种:
+ spouts, data source
+ bolts, process incoming tuples

跑在cluster上，用户把topologies提交给master node，这个master node叫做Nimbus。

实际跑则是在worker上。一台机器有多个worker。一个worker只跑部分topology，有一个supervisor，在一个JVM上。一个worker有多个executor。一个executor有多个task。bolt和spout在task上面完成。

topology可以认为是一个logical query plan。

Nimbus是一个thrift服务，Storm topology是Thrift objects。

每一个worker有一个进程叫supervisor，和master的Nimbus交互。

## 错误恢复

Nimbus宕机， worker还能继续，user不能提交新的topology。

worker宕机，supervisor restart worker；对于running topologies，(运行中的topologies)直到Nimbus revive之前，都不能reassign。

## Processing Semantics

提供两种semantic guarantees：至少一次和正好一次。each tuple that is input to the toplogy will be processed at least/exactly once。

Storm实现了at least once。通过增加一个AckerBolt，用XOR来记录输入和输出的data。只有XOR之后为0表示通过。

## Appendix

Storm的缺点
1. 同一个topology或者不同topologies的task会在同一个machine上
2. bolt宕机／slowdown of a task，reads slow donw -> buffer to fill up -> global input buffer fills up -> updtream output buffer to fill up