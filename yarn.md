# 介绍
YARN这个名字就很有意思 Yet Another Resource Negotiators

随着Hadoop的大量应用，大家发现有两个主要缺陷：1.有很多将编程模型将资源管理框架紧密结合，使得研发人员不得不放弃MapReduce 2.中心化的任务控制流，会使得调度者的拓展性收到限制。

YARN的作用就是将资源管理框架和变成模型分离开来，同时为每个任务组件建提供许多调度函数（比如任务容错）。



# 

# 附录
这篇paper主要参考的是[Apache Hadoop YARN: Yet Another Resource Negotiator](http://dl.acm.org/citation.cfm?id=2523633) 这篇paper是有雅虎、微软、脸书的工作人员一起编写。