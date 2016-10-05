# HDFS

首先先介绍HDFS，这个是整个Hadoop系统的基础组成部分。

## 介绍

HDFS的设计就是为了解决大数据集合存储的可靠性，并且使这些数据能够以高带宽提供给读者。在一个很大的cluster内部，上千台机器不仅自己存储数据并且执行任务task。用分布式系统的好处之一就是，资源会随着需求在一个合理的范围内增长。

Hadoop提供了分布式文件系统和为MapReduce模型使用大分析和转移大数据集框架。Hadoop的一个特征就是数据大分布\(data partition\)和搭建在不同主机上大计算，以及将应用的并行计算执行尽量靠近数据。\(这里我猜测是物理上的靠近，以次减少传输数据所花费大时间)。

Hadoop是一个Apache开源项目，Yahoo！贡献了Pig、ZooKeeper和Chukwa，以及超过80%的HDFS、MapReduce。Powerset(被微软收购为一个部门)开发了HBase，FB开发了Hive。

HDFS是Hadoop的文件系统。它将元数据metadata和应用数据application data分开存储。HDFS将元数据存储在一个特殊的服务器NameNode上，而应用数据存储在DataNode的服务器上。所有的服务器都是完全连接。

HDFS的DataNode并没有用类似于RAID的方式来保证数据可靠性，而是用了类似GFS的技术，将一份文件的内容存储在多个DataNode上面来保证耐用性。而且这个机制除了保证耐用性，还能数据传输的带宽（？？？），每个计算也更有可能靠近数据。

不同的文件系统都有实现namespace。Ceph有namespace server(MDS)的集群。GFS有几百个namespace servers(master)，而每个master都会有1亿的文件。

## 框架

### NameNode

HDFS的namespace是文件和目录的层级结构。文件和目录都以索引节点inode的形式存储，记录比如permissio,修改和访问时间，namespace和硬盘分配的内容。文件被分成了block(比如129MB，可以自定义)。NameNode维护一个namespace树和file block到NameNode的映射。当client读一个文件的时候，首先通过NameNode找到这个file所有block的位置，然后从距离client比较近的DataNode读取数据。当写数据的时候，client会要求NameNode置顶三个DataNode来存储数据（存三份）。cluster可以有一个NameNode，几千个DataNode和上万个HDFS client，而每个DataNode可以同时执行多个任务。

HDFS将整个namespace存放在RAM中。索引节点数据和每个文件由哪些block组成的list，这些命名系统的元数据都叫做image。image存储在本机文件系统的永久数据叫做checkpoint。

## 文件读写操作和复制replica管理

## Appendix

大部分内容都是参考这篇由Yahoo!的Konstantin,Hairong,Sanjay,和Rober发的 [The Hadoop Distributed File System](http://ieeexplore.ieee.org/document/5496972/?arnumber=5496972&tag=1)

