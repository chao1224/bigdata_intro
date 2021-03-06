# 介绍

YARN这个名字就很有意思 Yet Another Resource Negotiators

随着Hadoop的大量应用，大家发现有两个主要缺陷：1.有很多将编程模型将资源管理框架紧密结合，使得研发人员不得不放弃MapReduce 2.中心化的任务控制流，会使得调度者的拓展性收到限制。

YARN的作用就是将资源管理框架和编程模型分离开来，同时为每个任务组件建提供许多调度函数（比如任务容错）。

许多网络公司都用Hadoop来存储、运行数据。而且工程人员和研究人员都能能够在这个平台上同时使用，这也是Hadoop能够成功的原因之一，这也有一个弊端，就是developer经常会将这编程模型拓展到超出集群管理。一种通常模式是提交"map-only"只有map的任务，在集群中随机产生一些任务。这种滥用的例子包括forking web server和gang-schedule计算？？

这些MapReduce的限制就使得大家开始讲Hadoop作为baseline，开始研发新的环境。在这里（YARN）就提供了一个在社区community推动下的解决方案。核心思路就是上面所说的，将资源管理框架和编程模型分开。而此时，MapReduce仅仅是一个运行在YARN上面的应用——为编程模型提供了便利性。其他能运行在YARN上面的编程模型也有很多，比如Dryad，Giraph，Spark和Tez。

# 历史和基本原理 History and Rationale

先说一下YARN的历史，是伴随什么需求发生的。

Yahoo!在2006年用Apach Hadoop来代替原来的WebMap应用框架，WebMap是在已知的网站上建立一个图，来提高搜索性能。这个网络图web graph有1k亿的节点和1万亿的连边。它的前身，"Dreadnaught"的极限是800台机器，并且需要整个框架于web相适应。Dreadnought已经有类似MapReduce的应用，所以如果要沿用MapReduce框架，很多重要的pipeline可以直接移植。这也是MapReduce前期能发展起来的原因：可拓展性。

除了Yahoo! Search需要这么大规模的pipeline，优化广告分析、垃圾邮件过滤和内容优化推动了前期的需求。当Apache Hadoop社区将这个平台拓展到了更大的MapReduce任务，multi-tenancy的需求就提上来了。[multi-tenancy](https://en.wikipedia.org/wiki/Multitenancy)指的就是一个软件跑在一个server上同时服务于多个tenant，一个tenant指的是一群优先级一样的用户。

# 结构

YARN提供了一系列函数，来提供这个平台需要的资源管理。特别地，每一个集群的ResourceManager（RM）追踪资源使用和节点状态、执行分配、和分配租户tenant之间的连接。通过将这些职能相互独立，中心分配者能够用tenant的抽象需求描述，而且同时不需要理解每个分配的语义。理解语义这个任务就交给了ApplicationManager（AM），它会协调一个job内的逻辑方案，通过从RM那里请求资源，从RM那里接收到的资源生成物理方案，并协调方案的执行（主要是容错）。

## 概念 Overview

RM在某一个机器上跑后台程序，作用类似于一个中央权威，在不同应用中调度资源。当有了这么一个可以中心、全局的集群资源视角，RM就可以执行很多功能，比如tenant间的fairness、capacity和locality。依据应用需求、调度优先级和资源可用性，RM动态分配租约lease——叫container——给应用，来跑在专门的节点上。container在逻辑上是一些资源集合，比如`<2FB RAM,1 CPU>`，这些资源绑定到了某个节点。为了执行和追踪这些分配，RM和每个节点上叫NodeManager（NM）的系统后台程序进行交互。RM和NM的交流是给予hearbeat。NM负责检测资源可用性、报告错误和container生命周期的管理（开始、结束等）RM就将NM状态的快照snapshot集合起来，形成一个全局的视图。

作业job通过公共提交协议提交给RM。job会经历准入控制阶段admission control phase，在此期间会验证安全证书，进行运行和管理的检测。合格的job会传递给调度器去跑。当调度器有了足够的资源，应用的状态就从accepted变成running。同时也给AM分配container病在node里面产生一个container。合格的app会被写到硬盘上，当RM重启或者宕机的时候就能恢复。

每个作业的ApplicationMaster是核心：他通过动态增加或者减少资源消耗来控制生命周期、管理执行流程、处理错误和计算事务，处理其他本地优化。AM可以跑任何的用户代码，能用任意的程序语言编写，因为所有和RM、NM的交流都封装到了可拓展的协议里。

一般情况下，AM会利用好几个节点的资源来完成一个作业，为了得到container，AM向RM发出资源请求。这些请求的个数包括了container的本地偏好，而RM会尽量满足这些来自各个应用的请求，根据他们的可用性和调度策略。当资源以AM的名义被分配之后，RM会为这些资源产生一个lease，AM通过heartbeat获取这个lease。当AM发现某一个container已经可用了，它会应用特定的发布请求和lease一起编码。在MapReduce里面，每一个container里面跑到不是map任务就是task任务。如果需要，运行的container可以直接和AM通过应用特定的协议进行交流，包括报告状态和接受框架特定的命令。总的来说，YARN的部署提供了一个基本但同时牢固的生命周期管理和container监控架构infrastructure，而每个应用特定的语法都是由框架framework自行管理。

## Resource Manager(RM)

RM由一个调度器pluggable scheduler和一个ApplicationMaster组成。

RM提供了两个公共接口给 1)client提交应用，2)AM动态协商资源；并提供了一个内部接口给NM，为了集群监察和资源访问管理。下面关注的是RM和AM之间的公共协议，因为这个最好的展示了YARN平台和其他跑在上面的应用/框架的边界。

RM将集群状态的全局模型和运行的应用报告的资源需求匹配。这使得执行全局调度成为可能，但要求调度者能够理解应用的资源需求。交流信息和调度这状态要紧凑、高效，这样RM能对应用需求和集群大小进行延展。资源请求获取的方式是准确度和间接性的一种平衡。幸运的是，调度者对每个应用只处理总体资源文件，而忽视了本地优化和内部应用流。YARN完全没有为map和reduce静态分割资源，它把集群资源当做连续的，从而提高集群利用率。

AM将资源需求根据ResourceRequest编成准则，每一个ResourceRequest都包含如下内容：

1. container的数量
2. 每一个container的资源，比如`<2GB RAM,1 CPU>`
3. 本地偏好locality preference
4. 应用内部的请求优先性

ResourceRequest是为了使得用户知道他们全部的需求，或者是roll-up version（就是上层，比如node级别的需求或者是rack级别和全局的本地偏好）。

调度者追踪、更新、满足这些请求，用NM heartbeat标记的可使用的资源。作为AM发送的请求回应，RM产生container和能够访问资源的标记token。RM将完成的container的突出状态（NM报告的状态），发送给AM。当加入新的NM，AM也会被通知到从而在这些节点上请求新的资源。

新的拓展功能是RM能够从应用那里请求回资源，当集群的资源比较稀缺，调度者准备撤销给定应用的某些资源。我们用类似ResourceRequest的结构来获取本地偏好。填写这些抢占的时候，AM有一定的灵活性，它可以调取那些不太重要的container，检查任务状态，或者讲计算转移到其他运行着对container。整体来说，这保证了应用能够保存工作，而不会强制的关闭某个container。如果应用不同意回收资源，那么RM能够等待一定的时间后，就可以通过强制NM关闭container来获得所需的资源。

## Application Master(AM)

一个应用可以是一系列静态的进程，工作的逻辑描述，甚至是一个长期运行的服务。ApplicationMaster是一个协调应用执行的进程，但他自己和其他container一样运行在集群内。RM的一个组件会和container协商，产生这个引导进程。

AM定期的发送heartbeat给RM来确定活跃度，并且跟新请求。在构建好请求的模型后，AM会将偏好和限制编码到给RM发送的hearbeat信息。作为回应，AM会接受到一个container lease，lease是在一套绑定在某个节点上的资源。根据这个container，AM会调整执行计划。这里应用的分配采用了late binding：产生的进程不是和请求绑定，而是和lease绑定。当AM收到资源的时候，不一定和它请求的保持一致？？？

## Node Manager(NM)

NodeManager也是一个后台程序，它认出container的lease、管理container的依赖性、监察执行、和向container提供一系列的服务。通过向RM注册后，NM发送hearbeat，并接受指令。

YARN里面所有的container，包括AM，都是通过container launch context（CLC）描述的。包括了环境变量的map、存储在远程访问硬盘的依赖、安全标记security token、NM服务的有效载荷、和创造一个进程的必要指令。验证了lease之后，NM配置container的环境（包括初始化监察子系统，按照lease中的资源上限）。为了发布container，NM复制所有必要的依赖到本地存储上。如果需要，CLC还包含了下载的凭据。container相互之间可以分析依赖，可以是同一个tenant的container，也可以是不同的tenants。最终NM通过跑一个专门的container来进行垃圾回收。

NM会根据RM和AM的指示，杀死container进程。比如当RM报告某个应用已经完成，当调度这决定把某个container分配给另外的tenant，或者当NM检测到这个container使用超过了lease中描述的上限。AM则可以是不需要这个工作的时候要求container被杀死。不管container是否存在，NM会清理本地的工作目录。当一个应用完成的时候，所有它的containers所占有的资源都将被释放，包括还在集群里面跑的进程。

NM也会定期检查物理节点的健康。它会检测本地硬盘的错误，并且跑管理员配置脚本，来判断是硬件还是软件问题。当发现问题的时候，NM就会将它的状态转换为不健康，报告给RM，然后调度器就会杀死container并停止在这个节点上的分配知道问题解决。

NM提供本地服务。比如log aggregation服务，可以上传应用写的数据到stdout和stderr，这两个文件在任务完成的时候就写到HDFS上。

管理员有可能给NM配置了一套可移植、辅助的服务。当一个container的本地存储会在完成后被清空，它可以永久保存输出。这样，一个进程就可以产生数据，哪怕是超过了container的生命周期。一个重要的用例就是Hadoop MapReduce，intermedaite数据会在map和reduce任务之间传输，通过辅助服务。

## YARN框架

总结一下YARN的流程

1. 每一个App有AM，通过AM的CLC将应用提交给RM
2. 当RM开始一个AM的时候，AM通过heartbeat向RM注册并且定期报告他的存活状况和资源需求。
3. 当RM收集到一个container的时候，AM会构造一个CLC来将这个container在对应的NM上发布。此时这第一个container就是AM本身，AM运行在一个node上。AM可以同时监测运行中的container的状态以及在资源回收的时候停止container。在一个container内监测进程是AM的责任。
4. 当一个AM完成工作的时候，它会从RM那里取消注册，并且清空。
5. 可选择的是，框架作战可以加一下控制流在clients之间，来报告作业状态并提供控制平面。

## 容错能力和可用性

Hadoop是设计用来跑在家用硬件上的，通过在每一层构建容错，它能将用户掩盖住监测的复杂性和出错硬件的恢复。YARN也沿用了这个思路，讲容错性分布到了ResourceManager和ApplicationMaster上。

RM可以从初始化时候的永久保存的状态中恢复，当恢复进程完成的时候，它就杀死所有运行中的进程，包括AM。按照正常支持恢复功能的系统，都会保存用户的pipeline。为了让AM能够在RM的重启中存活下来的技术还在进行中。如果有这种方法，那么当RM宕机的时候，AM可以继续执行；然后等RM恢复的时候，AM再重新同步一下即可。

当某个NM宕了，RM通过heartbeat没有回应检测到后，会标记所有在那台node上面的container被杀死，将这个失败告诉所有运行中的AM。而如果这个错误是短暂的，NM会和RM重新同步，清空本地状态然后继续。这两种状态下，AM都会对接电视吧作出回应，很可能是将fault期间在那台node的container里已经完成的任务重新跑一遍。

因为AM只是一个container，所以AM挂了不会影响到任何节点的可用性。RM会将挂了的AM重新启动。重启的AM会和它正在运行的container进行同步。比如，Haddop MapReduce AM会恢复他所有已经跑完的task，但是在AM恢复期间完成的任务会需要重新跑。

最后，container对应出错的方法是完全依赖于framework的。RM从NM收集所有container exit事件，并通过heartbeat将这些时间发送给相对应的AM。MapReduce ApplicationMaster已经收听这些信息，并通过向RM申请新的container来重新做map和reduce任务。

# 附录

这篇paper主要参考的是[Apache Hadoop YARN: Yet Another Resource Negotiator](http://dl.acm.org/citation.cfm?id=2523633) 这篇paper是有雅虎、微软、脸书的工作人员一起编写。

这篇也总结的很好[YARN Architecture](https://www.zybuluo.com/xtccc/note/248181)
