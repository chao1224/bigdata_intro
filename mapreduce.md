# Introduction

MapReduce作为一种编程模型Programming Model，同时支持了处理和生成大数据集合。用户定义map函数和reduce函数。map函数会处理一个键值对key-value pair，来生成一系列的中间键值对intermediate key-value pair；reduce函数将所有的intermediate key-value pair根据intermediate key融合。而现实生活中的许多问题都能映射到这两个操作上。其实这个就像以前我在写算法题一样，基本的算法题都可以归纳为搜索状态，而搜索的方式不外乎分割处理（自上到下）或者合并处理（自下到上）。

这个系统会帮助用户处理比如分割输入数据、安排程序执行、处理宕机、和机器间交互，从而使程序员不用关注分布式和并行处理。

Jeff和其他谷歌的程序员已经实现了很多不同的带特殊目的地计算，已至此处理大量raw data，比如抓取的文本、网页日志，来计算不同的数据，比如inverted indices，不同网页文件的图结构表示，每天最常用的query等。这些计算都很直白，但是输入的数据量非常大，并且输入的数据都分布在不同的机器上。如何并行计算、分布数据并处理失败，是需要解决的关键问题。

解决的方案就是设计了一个新的抽象，这个抽象允许我们能运行程序、但是同时将技术细节隐藏在了库里面，技术细节比如上面提到的并行计算、数据分布、容错能力和负载平衡。抽象的描述受到了Lisp和其他面向函数语言中的map和reduce原始操作的启发，很多甚至大部分操作都可以由map和reduce完成。这种处理方案也能更好地支持大量数据的并行处理，使用重复运行作为解决容错的主要机制。

# Programming Model

输入是一系列键值对，输出也是键值对。

Map的输入是键值对，产生中间键，intermediate key，和指向这个key的数值集合。MapReduce库会把同一个key的库的value集合合并，传递给Reduce函数。



# Implementation

描述了MapReduce实现接口，以及如何如何适用到谷歌内。

# Refinements 

描述了几个有助于编程模型的提升refinement。

# Performance 性能

# Appendix
每次提到MapReduce，大家都会想到Jeff Dean，这里主要也是参考了他和Sanjay Ghemawat的paper。这篇发表在2008年[Communication of the ACM(CACM)](http://cacm.acm.org/)上的文章，引用量在我写这些文字的时候超过了18k [MapReduce: Simplified Data Processing on Large Clusters](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)
