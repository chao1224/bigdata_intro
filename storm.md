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
+ bouts, process incoming tuples

跑在cluster上，用户把topologies提交给master node，这个master node叫做Nimbus。

实际跑则是在worker上。

topology可以认为是一个logical query plan。

Nimbus是一个thrift服务，Storm topology是Thrift objects。

每一个worker有一个进程叫supervisor，和master的Nimbus交互。

错误恢复：

Nimbus宕机
+ worker还能继续，但user不能提交新的topology
+ 对于running topologies，(运行中的topologies)直到Nimbus revive之前，都不能reassign

worker 宕机，则supervisor重启

## Appendix

Storm的缺点
1. 同一个topology或者不同topologies的task会在同一个worker上
2. bolt宕机，queue会溢出，因为包含的内容太多