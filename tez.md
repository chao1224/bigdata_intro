# Apache Tez: A Unifying Framework for Modeling and Building Data Processing Applications

Bikas Saha, Hitesh Shah, Siddharth Seth, Gopal Vijayaraghavan, Arun Murthy, Carlo Curino

Hortonworks, Microsoft

# Abstraction

开源的MapReduce正在渐渐被一些针对特定的引擎。这导致很多重复的工作——每一个新的引擎都会实现一些重要功能，比如容错、安全、straggler。

这里介绍了开源框架Apache Tez，用于建立data-flow driven processing runtimes。Tez提供了组件来快速搭建可移植的data-flow centric 引擎。我们设计的河西那就是加强component re-use，同时不会阻碍和性能相关的data plane。这实际上是和前面几代系统最主要的差别。而且Tez还提供了本地支持，进行运行时优化，比如Hive的动态划分调整。

Tez在Yahoo!，Azure，LinkedIn和许多Hortonworks上部署，还有许多引擎正在整合到上面。

# 1 Introduction

大规模数据分析越来越重要。

Hadoop最开始只是为了服务MapReduce系统，现在已经完整发展成了一套包罗万象的数据分析平台。第一阶段是提供了一些列建立于MapReduce的更高层次的抽象，比如Hive，Pig和Cascading。这加快了Hadoop的传播使用，但也逐渐发现了Hadoop的不足。这些限制让大家逐渐研究出来更加灵活、高效的改进，将Hadoop发展成了更一般功能、类似OS资源管理器的layer，也就是YARN，和一个适用于任意执行引擎的应用框架层。这也使得MapReduce仅仅是一个建立在YARN上的应用，其他的类似Spark，Impala，和Flink。提供了方便，但是也导致了效率低，因为不同的应用，比如MapReduce和Spark都会独立的实现delay scheduling机制。

这篇paper提出了Tez，是一个支持任意数据流导向框架，同事避免了重复性功能。Tez API允许框架根据数据流建立逻辑和物理语义。Tez有以下贡献：

1. 允许用户讲计算建模为DAG。这种方法比传统的节点和边，提供了更细粒度的讲解，deliver可以更细致的控制、定制data plane。
2. 将APIs提供动态的修改DAG。这允许更复杂的优化查询，比如基于在线信息的修改数据划分。
3. 提供了一个可移植和有效的实现。比如YARN安全性、数据本地性、资源复用、容错。
4. 提供了工程师和研究院一起讨论的机会。（这个我真的很想吐槽）

# 2 Background and Rationale

一些term。

DAG：有向无环图，表示数据留图，数据在边的方向流动。

Vertex：代表logical step。一个processing step通过应用的代码进行数据转换，比如filter或者modify。

Logical DAG：由一个vertices集合组成，每一个vertex表示一个logical step。在分布式运行中，有一个节点表示的logical work物理上

Task：代表了一个vertex的工作单元。在分布式系统中，一个vertex的logical work物理上是在集群中好几台机器上跑的一个task集合。每一个task都是vertex的实力，运行处理了这个vertex代表的部分数据。

Physical DAG：物理DAG是有tasks集合组成，讲logical DAG拓展到相对应的组成的tasks上。

Edge：表示数据流动的方向，在producer和consumer之间。Logical DAG的一条边表示数据的依赖。physical DAG的一条边表示数据的转移。

**Expressiveness：**

MapReduce对于应用进行了比较大的限制，必须用map或者reduce函数。一个基于DAG的模型可以更好地控制计算。Tez的核心模型也是围绕DAG。MapReduce还提供了内置的语义，在逻辑上运行map/reduce，在map和reduce之间强加了stored & partitioned操作，对应一些应用是额外的。所有最后在DAG中，能有描述任意DAG的API，而不用强加没有联系的语义。

**Data-plane Customizability：数据图定制**

一旦一个分布式计算的结构确定了，有许多种将这个逻辑实现的方式。可以是不同划分数据方式的算法，或者是用到了不同的硬件，比如remote memory access（RDMA）。在MapReduce中，引擎内置的语义使得这个变得比较困哪，是因为这会使得改变引擎内部实现。还有的原因是对于执行MapReduce工作的任务，它们的整体结构使得插入可替换的实现比较困难。这就会让定义data plane的数据转移完全的定制化。

尽管已经有框架实现了一般化概念的DAG，但是还是有限制，如内置的语义和实现。Tez提供了稍微低一级层次的抽象，使得这些语义和特殊的实现可以在一些组件上完成。

**Late-binding Runtime Optimization**

应用需要进行late-binding决策。有些算法（比如join的策略）会因为观测到的数据而动态改变。分区基数和工作划分会因为应用对数据和环境的深入理解而改变。Hadoop集群会因为它的用法和负载特征而动态改变。这些要求应用根据当时集群的状态再确定执行的特征。

# 3 Architecture

总结一下，Tez解决了三个问题：1)对底层模型的表达 2)data plane的定制 3)加强运行时优化。与搭建一个一般化的引擎不同，Tez提供的是一个统一框架，服务于为了实现各自功能的引擎（每一个引擎都将数据处理按照需求进行定制）。Tez处理了最常见也是比较困难的问题，在Hadoop上安排并且运行分布式数据的应用，从而让应用只需要关注提供特殊的语义和优化。Tez library层和上一层应用层的关注点有明显的区别。Tez提供了集群资源协商、容错、资源弹性、安全性、内置性能优化和共享库。每个应用有定制的逻辑、data plane和专门的优化。

带来了三个优势：1)分期还研发消耗 2)提高性能 3)将来可以充分利用不同的引擎，因为共享底层

Tez是由一系列APi组成，定义了数据处理。可以把它理解为提供了数据流框架的组件，应用程序就使用定制的逻辑（operator）和数据转移code（比如从远程机器读取数据）。具有战术性和战略性。

## 3.1 DAG API

Tez用DAG来描述计算的框架。

**Vertex**：DAG API的一个节点表示了数据转换，是处理数据的一部分。核心logic运用到数据的地方就在这里。因此，一个vertex要配置使用用户定义的`processor`类，这个类定义了每一个task的逻辑。DAG里的一个vertex经常在几个任务并行执行。

**Edge**：DAG里的edge从逻辑和物理层次表示了数据运动的方向。（在producer和consumer节点之间）

+ Connection Pattern：edge的逻辑角度就是producer和consumer vertex tasks之间的connection pattern和调度依赖性。三种常见的connection pattern：one-to-one, broadcast, scatter-gather。
+ Transport Mechanism传输机制：edge的物理角度就是存储或者传输机制。可以是本地硬盘，或者本地/远程内存，等。实际数据传输是通过兼容的输入和输出类定义的。兼容性基于一样的数据格式和物理传输机制。比如都是用键值对和硬盘上操作，或者都操作在字节流上并用内存。

DAG通过创建vertices并通过edge连接它们定义。会从data source读取数据作为初始，最终写到data sinks。data source和`DataSourceInitializer`有所联系，可以在运行时被调用，来确定最优读取模式。data sink和`DataSinkCommiter`联系，运行时输出。

## 3.2 Runtime API

DAG API定义了数据处理的组件结构。runtime API就用了将实际应用代码插入到这些组件结构中。rumtime API定义了创建processor处理器和DAG中输入输出类的接口。

Input，Processor，Output（IPO）：逻辑的一个vertex表示数据转移。实际专一要在这个vertex上运行task。Tez将每一个task定义为一个集合的输入、一个处理器和一个集合的输出（IPO）。processor由这个task对应的vertex定义。input由连接到这个vertex的edge的输出类定义。output是edge的输入定义。这使得processor能对整个处理有一个逻辑视图，维护一个简化的编程模型。当IPO被创建之后，它们就被配置。

IPO Configuration：IPO的配置是在DAG创建时，有一个不透明的二进制有效负载完成。二进制有效载荷的方法使得应用可以按照自己机制实例化代码。处理简单的配置，也可以进行代码注入。配置之后，processor会被提供input、output，然后被要求进行计算。IPO合作完成计算。框架和这三个交流。

Data Plane Agnostic数据平面无关：Tez实际上并没有要求特定的数据结构。Tez进行的数据转移只有在连接producer和consumer时候会产生。当一个producer任务产生输出数据，（比如URL，size）会将元数据通过Tez发送给consumer任务的输入。Tez将这个元数据通过connection pattern路由给comsumer，通过edge。所有对网络的小号不打。这个也是的Tez的数据格式不用知道。input、processor和output可以选择自己的数据格式。

IPO这种任务组成方法使得系统pulggable。

## 3.3 Event Based Control Plane

Tez框架使用了解耦合控制，允许许多实体相互之间交流控制信息。为了实现这个功能，Tez提供了event based control plane。软件部分产生事件，事件路由到接收者，异步、非阻塞式。如图5，`DataEvent`由一个producer任务输出产生，包含了输出的元数据，来给consumer任务读取。这个时间由framework接收，路由给consumer的输入，通过edge的连接信息。如果consumer任务输入有错误，那么会发送一个`ErrorEvent`给framework；Tez会让producer task重新产生数据。event based通信的优势是提供了更大的便利性、可以增加更多的实例。每一个event有一个opaque load payload，有sender解释，然后receiver交换元数据。event传输的延迟依赖心跳的延迟和orchestrator处理的延迟。这个延迟和job大小成正相关，如果control plane events会占用data plane，那么会对整个应用的延迟负面影响；但如果只是在data plane建立的时候用到，那么影响有限。

## 3.4 Vertex Manager: dynamically adapting the execution

数据依赖的action比如sample based range partition或者partition pruning优化，都需要改变DAG。不可能将所有这样的改变前后的重新配置的图编码。所以Tez需要让application将这种决策自己在运行时定义，冰河Tez合作，动态改变DAG。这个就通过`VertexManager`抽象实现。

Runtime Graph Re-configuration：当监理一个DAG，每一个vertex都和`VertexManager`联系，`VertexManager`负责每一个vertex在DAG运行时的重配置。业务力促恒框架包含了不同状态的机器（控制vertex和task的生命周期）。vertex state machine负责在状态转换时和`VertexManager`交流。`VertexManager`会被提供对象context object，context objexc告诉他比如一个task完成了，状态转变了；这样`VertexManager`就会在自己vertex状态里面改变相对应的状态。`VertexManager`还可以控制vertex并行化、IPO payload的配置、edge属性、task调度。

Automatic Partition Cardinality Estimation自动划分基数估计：作为运行时优化的一个例子，展示了一些MapReduce关于确定reduce阶段几个tasks的一种解决方案。这个数字通常依赖于reduce阶段数据的大小。shuffle指的是在reduce操作前，跨网络读和将划分的输入聚合起来。在Tez中，`ShuffleVertexManager`可以用来控制读取shuffle data的vertex：任务产生了需要shuffle的data，用`VertexManager` events给`ShuffleVertexManager`发送数据统计来计算整个数据大小和估计reducer数目（用per-reducer desired data size的启发式方法）。因为reducer的个数表示了partition cardinality划分基数，这个方法可以用来估计最有划分数。

Scheduling Optimization：`VertexManager`夜空中每一个vertex的task调度。task应该在输入数据准备完成之后开始。但如果一个任务可以在部分输入数据获取的情况下有意义进行，那么不按照顺序开始。shuffle操作就是这个例子：当一部分输入数据都区之后，可以先开始运行。shuffle是一个很“贵”的操作，提早开始可以减小延迟。无序调度会导致资源的死锁，Tez有内置死锁检测和抢占机制预防死锁。它使用DAG依赖来检测无序任务并通过抢占解决死锁。

## 3.5 Data Source Initializer

DAG中的data source关联到了`DataSourceInitializer`类，会在框架运行任务之前调用，让vertex读取data source。和`VertexManager`一样，initializer也会接收和发送event给其他的事件。会配置任务读取数据或者告诉vertex manager关于vertex的重配置。

比如Hive动态划分修剪。data source经常会读取后在某些key上join操作。如果join key空间已知，那么我们可以只读取和join相关的子数据。有时候这种元数据只有在DAG里面运行时才知道，Hive用`InputInitializer`event发送这个元数据，从其他vertices的tasks发送给data source的initializer，initializer用这个元数据来决定读取先关的子数据。

# 4 Implementation and Practical Considerations

下面考虑怎么实例化到YARN上面。这一部分工业上最耗时。

## 4.1 Implementation in YARN

Apache Tez主要由三个project组成

+ API library API库:提供了DAG APIs和runtime APIs，和其他用户端库，来建立应用。
+ Orchestration framework业务框架流程：这被实现为了一个YARN Master Application，来执行DAG。
+ Runtime library 运行库：提供了不同out of box的输入和输出的实现。

Typical Tez Application Lifecycle：一个基于Tez的应用用API库构建DAG来表示应用的逻辑。通常，更高层的应用，比如Pig会运行时建立DAG通过用自己的语言写，然后encode到Tez DAG。既然Tez是在Hadoop集群上的，也提供了到Hadoop现有的storage services输入和输出的操作——HDFS用于可靠的数据存储和YARN Shuffle Service用于临时数据存储。只用这些服务的应用只需要实现自己的processor来启动和运行。通常应用会创建一个一般化的processor host，用于配置之后执行DAG相关操作。Tez的输入和输出都是基于键值对的数据格式，因为Hadoop的生态环境比如Hive和Pig都是键值对，也可以转变成其他数据格式。DAG然后就通过Tez client API被提交给YARN。YARN发布Tez AM（AM是per-application controller）AM执行的DAG是逻辑DAG，描述了数据流图。AM将这个图扩大，将任务并行化到每一个节点。通过DAG中描述的input initializer和vertex manager实现这个并行化。AM然后要求YARN给自由来在不同的vertex上面跑任务。YARN的回应就是给container，一个container是某一个节点上的资源的合集。任务按照依赖性关系执行，当所有任务完成，DAG就完成。AM日志追踪、元数据信息用于检测和debug。

通过利用YARN和MapReduce现有的库和服务，能快速在现有产品上建立大量网络数据的shuffling；并且和现有的资源共享和multi-tenancy模型整合。

## 4.2 Execution Efficiency

业务流程框架的YARN实现结合了许多现有的想法。

Locality Aware Scheduling：将执行靠近数据的调度对于大数据执行很有效。Tez试着将tasks靠近他们的输入数据为准。本地化在DAG创建的时候会说明，但最终确定是在运行时。tasks在最初读取数据源的时候会读取位置信息，而intermediate task的位置是根据源task和连边推测的。比如scatter-gather的task没有特定的地址，但会比较喜欢靠近更大的输入数据集。1-1 edge明确了严格的本地化关系，在source和target任务之间。因为完美的本地化会可能会导致集群过于忙，所以框架会自动的放宽本地化策略，从node基本放宽到rack级别等等，通过delay scheduling。

Speculation：大集群可以有不同的硬件和不同的负载。这会带来环境诱发的任务变慢。这种慢的任务叫straggler。Tez检测任务进程，并试着检测straggler任务，比同一个vertex其他任务运行慢的任务。一旦检测到这种任务，就会推测性的把源程序再跑一次。如果推测性的运行有效，那就缩短了运行时间。

Container Reuse：Tez AM在YARN提供的container里面跑。当一个人玩完成，AM就会选择将container返回给YARN并请求另一个额度的container。但是每一个container分配都会因为和YARN协商资源而产生额外的联系，同样还有发布container的过程。这种额外使用可以通过re-use container来减少。re-use就是把这个container在其他等待中的任务进行资源和locality的匹配。如果没有匹配的tasks，Tez AM会释放空闲的container给YARN，然后请求下一个container，但是资源容量不一样。在Java里面，这种做法会给JVM优化器更长时间来观察和优化hot code path，更进一步提升性能。

Session：session是container复用更进一步的概念。一个Tez AM可以在session mode中跑，可以跑一系列有client提交的DAGs，这就允许从不同DAGs的tasks复用containers。另外，一个session可以预热，通过要求AM设置container在第一个DAG准备执行之前。这些预热可以执行一些预设的代码》（JVM优化器在这里起作用）比如Hive和Pig用session模式来在同一个session运行多个drill-down的query，提升性能。Tez session也使得迭代行的过程更加有效的执行。每一个迭代都能用一个新的DAG表示，并发送到一个共享的session，来预热所需要的资源。

Shared Object Registry：Tes将container复用的好处扩大到应用，通过提供一个内存中cache的对象，这个对象被填充，然后复用到给下一个任务填充。这个对象的生命周期可以先知道每一个vertex，一个DAG，或者一个session，有框架管理。可以用于减少重复计算结果。比如Hive会把（Hive parlance中）map join结果中较小端的哈希表填充起来，一旦一个join任务构建了这个哈希表，那么其他任务就不需要从重新计算。

## 4.3 Production Readiness

尽管性能和效率对于类似Tez的框架很重要，我们不能忽视为production做准备的标准库和大数据处理所依赖的框架。过多的实体都用了类似Apache Hive、Pig（开源的）和其他commercial software比如Cascading来跑主要操作。如果他们用了Tez，那么容错、安全性、multi-tenancy就变成了必要需求。幸运的是，Tez已经能够在YARN上面运行，并且利用它们实现了这些功能。YARN整合例证了实现了特殊功能的Tez可以在YARN上跑。

Multi-tenancy:几个应用相互之间共享资源。有些引擎（应用）使用的后台程序，会占有很多资源，而且不能和其他应用共享。Tez里，因为每个应用都是跑在task里，更加细粒度地控制，资源弹性。

Security:Tez加密和应用之间交流的metadata。

Fault Tolerance:当一个人玩因为及其原因挂了，那么就在另外一台机器上重跑。已经执行完的consumer task不用重新跑。

Limitations:现在只能在JVM上跑。

# 5 Applications and Adoption

这里列出已经被更新到Tez框架的projects。

## 5.1 Apache MapReduce

MapReduce就是一个2个节点组成的DAG。Tez中，它可以贝表示成一个map vertex和一个reduce vertex通过scatter-gather edge连接。Tez有内置的MapProcessor和ReduceProcessor，提供了一样的功能。

## 5.2 Apache Hive

Hive是最受欢迎的SQL-based描述式查新引擎。Hive将查询翻译成优化查询书。通常这些翻译到MapReduce的查询不高效，因为MapReduce表达能力有限。用了Tez DAG API，这些树可以直接翻译成DAGS，表达更加高效。

## 5.3 Apache Pig

Pig提供了程序脚本式语言PigLatin，用来写复杂的batch processing ELT流。PigLatin支持复杂的DAGs，每一个vertex都有多个output。MapReduce里，应用只能写一个output，因此需要对输出数据加一些额外的标签来区分。Tez就使得Pig的代码干净、可维护。

## 5.4 Apache Spark

Spark提供了分布式数据处理。数据分布的元数据是通过RDD获取的，这个元数据也在构建tasks DAG的编译时用到。我们可以将已经编译完成的Spark DAG转换成Tez DAG，并在YARN集群上面跑。用户定义的Spark code被序列化成Tez processor payload，并被注入到通用的Spark processor，让这个processor反序列化并执行用户代码。

## 5.5 Apache Flink

Flink原来是Stratosphere research project of the TU Berlin data management community。是一个并行处理的引擎，提供Java和Scala的API，基于成本的优化。Flink优化后的DAG可以转换到Tez DAG。Hive和Pig都是用的键值对，Flink用的是二进制数据格式，这种格式可以用来处理类似group by的操作而不用任何的反序列化消耗。因为Tez task model可组合性，Flink可以将运行时操作算子和二进制格式合并，从而使程序不用修改就能跑在YARN上面。

## 5.6 Comercial Adoption

Tez的这种定制化和专注性能的设计被应用到了商业软件项目中。一个分布式数据处理的开源项目，Cascading（Java API）。已经被更新到运行在YARN上面，并使用Tez框架。

## 5.7 Deployments

Yahoo!在Tez上面跑Hive和Pig。LinkedIn讲Hadoop机器移植到了YARN，并且在Tez上跑Hive和Pig。Azure用了Tez。Hortonworks提供TEz作为Hadoop分布式一部分。

# Appendix
