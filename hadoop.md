# HDFS

首先先介绍HDFS，这个是整个Hadoop系统的基础组成部分。

## 介绍

HDFS的设计就是为了解决大数据集合存储的可靠性，并且使这些数据能够以高带宽提供给读者。在一个很大的cluster内部，上千台机器不仅自己存储数据并且执行任务task。用分布式系统的好处之一就是，资源会随着需求在一个合理的范围内增长。

Hadoop提供了分布式文件系统和为MapReduce模型使用大分析和转移大数据集框架。Hadoop的一个特征就是数据大分布(data partition)和搭建在不同主机上大计算，以及将应用的并行计算执行尽量靠近数据。(这里我猜测是物理上的靠近，以次减少

## 框架

## 文件读写操作和复制replica管理

## Appendix
大部分内容都是参考这篇由Yahoo!的Konstantin,Hairong,Sanjay,和Rober发的 [The Hadoop Distributed File System](http://ieeexplore.ieee.org/document/5496972/?arnumber=5496972&tag=1)


