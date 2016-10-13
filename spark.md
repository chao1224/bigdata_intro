# 介绍

提出了RDD(Resilient Distributed Datasets)，一个分布式的内存抽象，使得程序员能够将在内存中进行计算。RDD是由两种应用激发的，因为当前的计算框架有两个无法解决的问题：iterative algorithm和交互式数据挖掘工具interactive data mining tools。这两种情况下，将数据保存在内存可以数量级的优势提高性能。

类似MapReduce的计算框架，让用户用high-level的操作写并行计算，而不用担心工作分布和容错。但这就使得对于一些需要复用中间结果的应用不是特别有效。在许多iterative机器学习和图算法中，数据复用是非常多的。比如PageRank，K-means clustering，和logistic regression。另外一个就是交互式的数据挖掘，用户要在同一个子数据集上跑多组查询。

# Appendix

Resilient Distributed Datasets: A Fault-Tolerance Abstraction for In-Memory Cluster Computing