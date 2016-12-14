# BlinkDB: Queries with Bounded Errors and Bounded Response Times on Very Large Data

有四种可能的priori，其中这篇paper使用的是predictable QCSs，query column sets。并且通过数据验证了predictable QCSs的稳定性。

基于Hive框架，并增加了两个组件

1. offline sampling module
2. run-time sampling selection module：对每一条query建立了一个Error-Latency Profile(ELP)，而ELP决定了对于sample size的大小

不支持任意join，但join不会成为阻碍，因为都可以平面化到对应的数据上。

但也还是支持distributed data warehouses。join一个大table和一个小table，小的table可以放在内存中。

stratified samples： 有用，但是需要额外的存储

uniform distribution不好：会增加larger group的比重；会misssmaller group。而是用stratified sampling，subgroups are sufficiently represented。
 
optimization考虑三个要素

1. sparsity of the data：
2. workload：某一个QCS的过去发生的频率可以用来评估
3. storage cost：

## Runtime

模拟一下给定一条query，目标是在运行时刻选择一个或多个sample，并且满足规定的时间和容错率。

首先在一个很小的subsample里面跑，然后收集query's selectivity, complexity and underlying distribution。根据这些BlinkDB在ELP上面找到对应的时间和错误率。

## Appendix

Best paper