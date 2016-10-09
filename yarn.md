# 介绍

YARN这个名字就很有意思 Yet Another Resource Negotiators

随着Hadoop的大量应用，大家发现有两个主要缺陷：1.有很多将编程模型将资源管理框架紧密结合，使得研发人员不得不放弃MapReduce 2.中心化的任务控制流，会使得调度者的拓展性收到限制。

YARN的作用就是将资源管理框架和编程模型分离开来，同时为每个任务组件建提供许多调度函数（比如任务容错）。

这些MapReduce的限制就使得大家开始讲Hadoop作为baseline，开始研发新的环境。在这里（YARN）就提供了一个在社区community推动下的解决方案。核心思路就是上面所说的，将资源管理框架和编程模型分开。而此时，MapReduce仅仅是一个运行在YARN上面的应用——为编程模型提供了便利性。其他能运行在YARN上面的编程模型也有很多，比如Dryad，Giraph，Spark和Tez。

# 历史和基本原理 History and Rationale

先说一下YARN的历史，是伴随什么需求发生的。

Yahoo!在2006年用Apach Hadoop来代替原来的WebMap应用框架，WebMap是在已知的网站上建立一个图，来提高搜索性能。这个网络图web graph有1k亿的节点和1万亿的连边。它的前身，"Dreadnaught"的极限是800台机器，并且需要整个框架于web相适应。Dreadnought已经有类似MapReduce的应用，所以如果要沿用MapReduce框架，很多重要的pipeline可以直接移植。这也是MapReduce前期能发展起来的原因：可拓展性。

除了Yahoo! Search需要这么大规模的pipeline，优化广告分析、垃圾邮件过滤和内容优化推动了前期的需求。当Apache Hadoop社区将这个平台拓展到了更大的MapReduce任务，multi-tenancy的需求就提上来了。[multi-tenancy](https://en.wikipedia.org/wiki/Multitenancy)指的就是一个软件跑在一个server上同时服务于多个tenant，一个tenant指的是一群优先级一样的用户。

# 结构

YARN提供了一系列函数，来提供这个平台需要的资源管理。特别地，每一个集群的ResourceManager（RM）追踪资源使用和节点状态、执行分配、和分配租户tenant之间的连接。通过将这些职能相互独立，中心分配者能够用tenant的抽象需求描述，而且同时不需要理解每个分配的语义。理解语义这个任务就交给了ApplicationManager（AM），它会协调一个job内的逻辑方案，通过从RM哪里请求资源，从RM那里接收到的资源生成物理方案，并协调方案的执行（主要是容错）。

## 概念 Overview

RM在某一个机器上跑后台程序，作用类似于一个中央权威，在不同应用中调度资源。当有了这么一个可以中心、全局的集群资源视角，RM就可以执行很多功能，比如tenant间的fairness、capacity和locality。依据应用需求、调度优先级和资源可用性，RM动态分配租约lease——叫container——给应用，来跑在专门的节点上。container在逻辑上是一些资源集合，比如<2FB RAM,1 CPU>，这些资源绑定到了某个节点。为了执行和追踪这些分配，RM和每个节点上叫NodeManager（NM）的系统后台程序进行交互。RM和NM的交流是给予hearbeat。NM负责检测资源可用性、报告错误和container生命周期的管理（开始、结束等）RM就将NM状态的快照snapshot集合起来，形成一个全局的视图。

作业job通过公共提交协议提交给RM。job会经历准入控制阶段admission control phase，在此期间会验证安全证书，进行运行和管理的检测。合格的job会传递给调度器去跑。当调度器有了足够的资源，应用的状态就从accepted变成running。同时也给AM分配container病在node里面产生一个container。合格的app会被写到硬盘上，当RM重启或者宕机的时候就能恢复。

每个作业的ApplicationMaster是核心：他通过动态增加或者减少资源消耗来控制生命周期、管理执行流程、处理错误和计算事务，处理其他本地优化。AM可以跑任何的用户代码，能用任意的程序语言编写，因为所有和RM、NM的交流都封装到了可拓展的协议里。

一般情况下，AM会利用好几个节点的资源来完成一个作业，为了得到container，AM向RM发出资源请求。这些请求的个数包括了container的本地偏好，而RM会尽量满足这些来自各个应用的请求，根据他们的可用性和调度策略。当资源以AM的名义被分配之后，RM会为这些资源产生一个lease，AM通过heartbeat获取这个lease。当AM发现某一个container已经可用了，它会应用特定的发布请求和lease一起编码。在MapReduce里面，每一个container里面跑到不是map任务就是task任务。如果需要，运行的container可以直接和AM通过应用特定的协议进行交流，包括报告状态和接受框架特定的命令。总的来说，YARN的部署提供了一个基本但同时牢固的生命周期管理和container监控架构infrastructure，而每个应用特定的语法都是由框架framework自行管理。

## Resource Manager(RM)



## Application Master(AM)

## Node Manager(NM)

## YARN框架

## 容错能力和

# 附录

这篇paper主要参考的是[Apache Hadoop YARN: Yet Another Resource Negotiator](http://dl.acm.org/citation.cfm?id=2523633) 这篇paper是有雅虎、微软、脸书的工作人员一起编写。

