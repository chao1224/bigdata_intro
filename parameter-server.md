# Scaling Distributed Machine Learning with the Parameter Server

<p align="center">Mu Li, David G. Andersen, Jun Woo Park, ALexander J. Smola, Amr Ahmed, Vanja Josifovski, James Long, Eugene J. Sita， Bor-Yiing Su</p>

# 1 介绍

实际数据都是TB和PB级别，复杂模型的参数可以到达$$10^9-10^{12}$$，全局共享的参数如果在所以worker机器之间传递，频繁的IO会带来的问题：占用大量带宽、sequential的算法会受到barrier的影响、容错性要求高。

## 1.1 贡献

除了第2点，其余都已经在2013年NIPS workshop那篇paper里面提到了，可以转到我[那篇的分析](https://lschacker.gitbooks.io/running-paper/content/parameter-server-for-distributed-machine-learning.html)。

1. Efficient communication：节点间的通信是异步的，同步（迭代的操作）也不会阻塞运算。
2. Flexible consistency models：一致性模型，放宽的一致性更好的因此了同步的代价和延迟。允许算法设计者可以在convergence rate和system efficiency之间做一个trade-off。
3. Elastic Scalability：新增加一个节点不用重启整个系统，用分布式的哈希表来使得节点可以动态加入。
4. Fault Tolerance and Durability:宕机是不可避免的，调度器的强制回收会加剧这个现象。用优化的数据复制架构，将数据copy保存在不同的服务器节点来提供快速恢复。（小于1s）而且client是相互独立的，新的client可以在某一个宕机之后自动启动，和MapReduce重新安排mapper一样。
5. Ease of use：全剧共享参数是以向量和矩阵的形式标记的，对ml应用比以键值对或者表格形式存储的数据结构好用。高性能的多线程操作，比如向量-矩阵乘法，都能用来进行很好的支持。

## 1.2 工程上的难度

（稍微有点疑惑）每一个parameter server node保存部分的parameter，而每一个worker node只需要这些parameters的subset。

几个主要问题：

**Communication通信**：首先数值一般比较小，其次用键值对传输数据开销比较大。改进的方法就是用vector和matrix，或者tensor。
**Fault tolerance容错**：有一张图展示了在不同系统上跑不同算法的数量级，parameter server性能最好，比其他系统以数量级的优势超过。

## 1.3 相关工作

大公司都有在做parameter server框架，Amazon，Baidu，Facebook，Google，Microsoft和Yahoo!开源的项目则有YahooLDA，Petuum（Eric Xing）。Graphlab支持的在一个最佳模型上进行parameter synchronization。

第一代parameter server：提出来memcached的分布式同步存储机制。缺少灵活性和性能。仅使用键值对存储作为同步机制。YahooLDA增加了一个专门的服务器来改进这个机制，是用户能自定义地更新操作。<br/>
第二代parameter server：第二代则特定针对了应用application specific parameter server。Pentuum是第一个将这个平台一般化的。Petuum用了bounded delay model改进YahooLDA，但在线程模型上有了更多的限制。<br/>
第三代parameter server：性能、灵活性、容错性都能够做到。

已经有了很多解决方案，比如Mahout（基于Hadoop）MLI（基于Spark）两个都采纳了迭代的MapReduce框架，也都使用了同步的迭代模式，使得两个方法在迭代的机器学习算法中都会产生不一直的性能分布。为了克服这个缺点，distributed GraphLab通过图抽象的方法用异步通信的调度。但是这个方法缺少MapReduce框架弹性的拓展性，并且依赖于粗粒度coarse-grained的快照来恢复。

# 2 机器学习

training data可以分作三个部分：feature extraction，（工业界有专门叫feature engineering的东西）objective function和learning。

## 2.1 Goal

许多机器学习的算法都能通过objective function来表示，SVM、logistic regression、linear regression等等（Deep Learning有没有我还不知道，看到的小伙伴有知道的不？）

下面主要通过两个机器学习技术来展示parameter server的高效性。

## 2.2 Risk Minimization

其实一个式子就能表示清楚：

$$F(w) = \sum_{i=1}^n l(x_i,y_i,w) + \Omega(w) $$

期中$$l$$是loss function，$$w$$是参数，$$\Omega(w)$$是regularizer。然后Algorithm 1详细描述了Task Scheduler、Worker和Server之间是如何合作的。

每一个worker用自己分配的参数求解gradient，然后传给server来更新。

## 2.3 Generative Model

机器学习的第二大类算法中，训练样例的标签是未知的，即unsupervised learning。比如topic modeling：给定一个文档集合，推断每一个文档包含的topic。

实际问题中，数据量会非常大：上亿的用户和上十亿的文档。因为数据量大，这些算法在第一代parameter server上经济性比较能接受。topic model的一个挑战是：描述当前文档生成的estimate必须要共享。

一个topic modeling的解决方案是LDA。尽管统计模型会不一样，但是算法的描述还是和Algorithm 1一样。主要区别是，跟新的步骤不是gradient computation，而是文档可以被当前的模型多大程度描述的估计。这个计算需要访问辅助的元数据，这个元数据是在文档更新的时候会写到硬盘上。

辅助性的元数据里面是topic集合，是这个文档的每个单词对于的topic集合。参数$$w$$是单词出现的相对频率。然后和前面描述的一样，每一个worker只需要存储它处理的文档里面所包含的单词的参数。

# 3 架构

parameter server由一个server group和多个worker group组成。每一个server node都保存部分的全局共享参数，然后server node相互之间会通信，来复制和（或）转移参数，一个server manager node监察每个server node。

每一个worker group跑一个应用，一个worker存储部分的训练数据，只和server node交流。每一个worker group有一个scheduler，负责分配任务和监察进程。

parameter server支持独立的参数命名空间。这使得一个worker group可以将他们的共享参数和其他group独立开来。几个worker groups可能有一样的namespace，这就表示他们几个group一起出来一个应用，比如deep learning来增加并行性。另外有可能是一个模型是online services consuming，然后新的数据加进来之后，就在其他worker group上进行训练。

## 3.1 (key,value)向量

可以用(key,value)向量来表示一个模型的参数。比如loss minimization问题，这个pair就是(feature ID, weight)；对LDA，就是word ID、topic ID和count的结合。每一个条目都可以通过key本地、远程地读写。

parameter server通过这个提升：机器学习通常将模型当作一个线性代数对象，通过将这些对象当作稀疏的，parameter server可以提供和(key,value)一样的抽象函数，但是允许重要的优化操作，比如矩阵和向量的加法、乘法、求2范数等。

## 3.2 Range Push and Pull

数据是通过push和pull来传递的。在Algorithm 1中，每一个worker都push gradient到server上，然后从server那里pull更新后的数据。更高级的算法描述在Algorithm 3中，模式几乎一样，除了每次只通信一个范围内的key。

`w.push(R,dest)`是将在key range $$R$$ 内，把数据push到dest的node或者group。`w.pull(R,dest)`也类似。但是为什么range-based的方法能提速没有介绍。猜测是类似于batch的概念。

## 3.3 User-Defined Functions on the Server

server节点可以执行用户自定义的函数，比如proximal gradient method。

## 3.4 Asynchronous Tasks and Dependency

任务是由RPC发出的，可以是worker想server发出的push或者pull，也可以是scheduler向任意节点发送的用户自定义函数。task由subtask组成，比如Algorithm 1中的`WorkerIterate`就是包含了一个push和一个pull。

task的异步执行：caller可以在发出任务之后立即进行计算。caller在收到callee的回复后一个任务标记为完成。一个回复可以是用户自定义的函数回复、由pull请求的键值对、或者一个空的确认。callee会在subtask完成之后，task的call返回，标记为完成。

callee会并行计算tasks。caller则在task之间加入_execute-after-finished_的依赖。paper中的Figure 5给了一个很好的例子。

## 3.5 Flexible Consistency

上面提到的依赖关系图可以用于放宽一致性要求。独立的任务通过并行（CPU，disk，network）来提高效率，但也会带来节点间的数据不一致性。还是图中的例子。图中的例子会使得两次迭代的结果一样，但是有的算法会对数据不一致性不那么敏感，比如每次迭代只使用one block of data，那么只有一部分收到了影响。

算法的效率和系统的性能依赖于许多因素，比如特征关系、硬件容量、数据中心负载等等。parameter server提供很高的灵活性，通过给不同的一致性模型建立不同的任务依赖图，这个依赖图是有tasks之间的依赖性定义的DAG。这么三个例子：

+ sequential consistency：其实就是同步的任务，只有上一个完成才能进行下一个
+ eventual consistency：所有任务相互独立，那么就可以同时开始
+ bounded delay：介于sequential和eventual中间，设置了一个最大延迟时间$$\tau$$，只有之前的任务在$$\tau$$时间完成了，才能开始新的任务。section 4会介绍这个方法的应用。这个$$\tau$$是可以动态改变的。

## 3.6 用户自定义过滤器实现的一致性模型

对优化问题，比如梯度下降，并不是每次计算的梯度对最优化都是有价值的，可以通过自定义的filter过滤掉一些不必要的传输，压缩带宽cost。

用户自定义的过滤器可以更加细粒度地控制一致性。一个filter可以改变和有选择性的同步键值对。几个可以合起来提升data compression的filter：

+ significantly modified filter：只有改变超过了阈值，才push
+ random skip filter：随机选择
+ KKT filter：采用了optimality condition，只在非最优情况下发送
+ key chaching filter
+ compressing filter：因为大部分传送的数据都是0、small int、和float，可以进行无损压缩

# 4 实现

## 4.1 Vector Clock

给定一个任务，它的依赖图很复杂，并且需要快速恢复，每一个键值对与一个vector clock联系。vector clock记录了每一个每一个节点包含这个键值对的时间。他用了追踪聚合状态或者拒接重复发送数据比较方便。但一个比较直接的实现方式是用`O(nm)`的空间，n个节点，m个参数，当n和m特别大的时候，十分消耗空间和带宽。

幸运的是，因为用来range-based通信，许多参数都有一样的时间戳：如果一个节点push了某个range内的参数给其他节点，那么这些range内的数据时间戳是一样的，也就可以将他们压缩到一个vector clock。比如$$vc_i(R) = t$$，就表示$$\forall k \in R$$, $$vc_i(k)=t$$，下标i表示第i个节点。

最初每一个节点只有一个range vector clock，所有在这个key space的参数都以0作为初始timesstamp。每一个range set最多将range划分成三份，可以参考Algorithm 2。

## 4.2 Messages

节点会发送信息给其他节点或者群组。一份message包含一个链表的键值对和相对应的range vector block:

$$[vc(R), (k_1,v_1), ..., (k_p,v_p)] \text{ } k_j \in R \text{ } and\text{ } j \in \{1,...p\}$$

因为有很多0的参数是可以不发送的（尤其是sparse情况），加上可以发送哈希后的list，而不是list本身，以此来进行压缩。用了snappy压缩库来压缩message。

## 4.3 Consistent Hashing

parameter server将key划分的范数和传统的分布式哈希表一样：key和server ID都被插入到了哈希环内。一个物理节点在环内通常有好几个“虚拟”服务器，来提高负载均衡和回复。（为毛？）

server manager负责ring management，所以其他的节点在本地高速缓存key划分，这样就能相互之间知道server的状态以及数据传给谁。

## 4.4 Replication and Consistency

每一个server节点都会保持它顺时针方向的k个邻居的key range。每次server从worker那里拿到数据并进行更新的时候，（大部分是aggrega）不妨更新完之后再replication，从而减少带宽使用。

注：这里有一个master和slave概念，对于任意一个server，保存的属于自己的数据就是master，保存的k分邻居的数据，那对于那几个邻居来说它就是slave。

## 4.5 Server Management

为了增加容错性，要支持节点的增加和删除功能。

从其他节点S将某一个范围$$R$$内的数据取出需要两个步骤：首先S将所有在这个范围$$R$$的键值对和对应的vector clock都复制；然后第二阶段，S再也不接受范围$$R$$内的message，然后同时，将第一步发生期间的更新发送给这个新的节点。（有点类似于分布式数据库更新的base和delta表）

节点离开也类似，server manager将这个离开节点的数据复制到另外一个节点上。有意思的是，当前还没有将整个管理部分和YARN或者Mesos类似的资源管理器结合，如果结合起来很期待。这里应该是更加大量的工程性问题。

## 4.6 Worker Management

增加一个worker节点类似

1. task scheduler给新的worker分配一个范围的数据
2. 然后这个节点从文件系统或者其他worker那里读取数据（training data），并从server那里pull共享参数
3. task scheduler会将新加的worker广播给其他worker，使他们释放一些数据

当一个worker离开的时候，task scheduler可能会用replacement。首先training instances特别多的时候，恢复一个worker比server复杂多，而且一个worker的一点数据对于最终机器学习的结果没有影响。因此算法设计者可能有时会倾向于继续而不管这个离开的worker。

# 5 Evaluation

# Appendix

这一篇既然是发在OSDI上的，和NIPS相比，更偏向于系统和框架。