# Resilient Distributed Datasets: A Fault-Tolerance Abstraction for In-Memory Cluster Computing

Matei Zaharia, Mosharaf Chowdhury, Tathagata Das, Ankur Dave, Justin Ma, Murphy McCauley, Michael J. Franklin, Scott Shenker, Ion Stoica
UC Berkeley

提出了RDD(Resilient Distributed Datasets)，一个分布式的内存抽象，使得程序员能够将在内存中进行计算。RDD是由两种应用激发的，因为当前的计算框架有两个无法解决的问题：iterative algorithm和交互式数据挖掘工具interactive data mining tools。这两种情况下，将数据保存在内存可以数量级的优势提高性能。

# 1 介绍

类似MapReduce的计算框架，让用户用high-level的操作写并行计算，而不用担心工作分布和容错。但这就使得对于一些需要复用中间结果的应用不是特别有效。在许多iterative机器学习和图算法中，数据复用是非常多的。比如PageRank，K-means clustering，和logistic regression。另外一个就是交互式的数据挖掘，用户要在同一个子数据集上跑多组查询。但目前的框架，大部分都是通过磁盘的读写来实现数据的复用，带来了大量的额外消耗。

然后有人开始研究特殊的可以进行数据复用的框架。比如Pregel是一个交互式图计算的系统，讲中间数据保存在内存中；HaLoop是一个iterative MapReduce接口。但这些方法都是支持特殊的几个计算模型，不够general。

这篇paper提出了一个的通用的变成抽象：resilient distributed datasets(RDD)。它允许在一个应用内实现高效的数据复用。现有的内存存储抽象，比如分布式共享内存、键值存储、数据库和Piccolo，提供了基于细粒度更新（比如table里的一个cell）的接口。用这个接口的话，保证容错性的方法就是将数据复制或者更新日志。两种方法都会带来额外的消耗，因为需要复制大量的数据。

和这些现有框架不一样的是，RDD提供了给予粗粒度数据转移(map,filter, and join)的接口，这些操作可以应用于广泛的数据上。如果一个RDD有部分数据丢失了，它可以从其他RDD那里获取数据并且重新计算。

RDD因为用类似的操作，所有很容易支持cluster应用，比如MapReduce、SQL、DryadLINQ、Pregel和HaLoop。当然也包括这些系统不支持的交互式数据挖掘interactive data mining。

# 2 Resilient Distributed Dataset(RDD)

## 2.1 RDD Abstraction

RDD是只读、划分好的记录集合，只能被确定的操作创建：1)从stable storage里面读 或者2)从另外的RDD里面读。这个操作叫做transformation。

RDD不需要一直被实体化（永久存储）。反而RDD会保持这个数据是如何获取的。这是一个很有效的性质，因为一个RDD如果失败之后不会重建，那么程序是不能引用它的。

最后用户能控制RDD另外两个特征：persistence和partitioning。用户会指出他们会用哪一个RDD，并选择一个存储方式(比如in-memory)。用户也会让RDD的元素根据每条记录的key在不同机器上进行划分。

## 2.2 Spark Programming Interface

Spark将RDD通过整合到语言中的API提供给用户。每一个dataset都是一个对象，transformation就是这些对象调用的函数。

开始的时候，通过从stable storage(比如map,filter)里面创建RDDs。然后它们会调用一些actions，将数值返回给应用或者给storage的操作。比如count 返回dataset里面有几个元素，collect 返回元素，save 讲dataset输出到storage。

另外，还可以用persist方法，来表示这个RDD将来会被复用。Spark将persis RDD保存在内存中，但如果内存不够，就会放到硬盘上。

### 2.2.1 Example: Console Log Mining

假设一个web service报错，然后一个操作算子正在TB的日志中找哪里出错了。用Spark，操作算子可以只价值日志里的错误信息，到内存中，然后交互式的查询。可以这么敲Scala代码：

```
lines = spark.textFile("hdfs://...")
erros = line.filter(_.startsWith("ERROR"))
erros.persist()
```

第一行定义了一个有HDFS文件支持的RDD，第二行是得到了一个filtered RDD。第三行就是讲errors persist到内存。现在就可以用RDD action操作。比如要知道一共有多少行message:

```
erros.count()
```

也可以在RDD上更进一步的转换，并且用他们的结果，比如：

```
// Count erros mentioning MySQL
erros.filter(_.contains("MySQL")).count()

// Return the time fields of erros mentioning HDFS as an array
// (assuming time is a field number 3 in a tab-separated format)
errors.filter(_.contains("HDFS")).map(_.split('\t)(3)).collect()
```

第一个`errors`的运行结束后，Spark会将`errors`的划分保存在内存中，以此提高计算。注意基本的RDD，`line`并没有保存到内存中。因为`errors`已经够用，而且足够小。

## 2.3 Advantages of the RDD Model

比较RDD和distributed memory abstraction(DSM)。

DSM中，应用读写的位置都是随机的。DSM是非常一般化的抽象，但这个一般化使得他更难在commodity cluster上以一种高效的方式实现。

RDD和DSM的主要区别是RDD只能通过粗粒度的transformation产生，而DSM的读写是细粒度，可以到每一个内存位置读写。这就使得RDD对应需要大量写入的应用支持的不是很好，但也提供了高效的容错。尤其是RDD不需要大量使用checkpoint，而通过使用lineage来恢复。而且RDD只需要恢复丢失的数据，而不需要将整个程序重新运行。

RDD的第二个好处是他们不可变（只读）的本质使得系统能缓和slow node（straggler）带来的影响。

RDD的另外两个优势：1)运行时通过数据的本地性来提高性能 2)RDD会在没有内存的时候优雅的降级，也就是存储到硬盘上。

## 2.4 Application Not Suitable for RDDs

RDD很适合batch应用，对于一个dataset里的元素执行同样的操作。RDD可以高效的将每一个transformation记作为lineage graph的一个步骤，然后可以不用读入大量数据来恢复lost partition。RDD对于异步细粒度的更新应用不合适，比如web应用的存储或者web增量式抓取。对这些应用，用传统的更新日志和数据检查点会更加有效。

这篇paper的目的是提供对于batch analysis有效的模型。

# 3 Spark Programming Interface

Spark通过集合到语言内的API来提供RDD抽象——Scala，因为它的简洁性和高效性。

开发人员写一个驱动程序，来连接到一个worker组成的cluster。driver定义了一个或多个RDD，并调用action。driver上的Spark代码也追踪RDD的lineage。worker是一直存在的进程，可以存储RDD分配到内存中。

正如2.2.1中日志的例子一样，用户提供参数给RDD操作，比如`map`。PS.传递的是闭包closure，是函数是编程语言中的[概念](http://www.kancloud.cn/kancloud/functional-programm-for-rest/56927)。Scala将每一个闭包当做是Java对象，这些对象可以序列化并被加载到其他节点上。

尽管在Scala中调用RDD很简单，但是还有一些问题要处理，比如用反射机制处理Scala的闭包。

## 3.1 RDD Operations in Spark

Table 2列出了Spark支持的RDD主要的transformation和actions。

+ transformation是lazy操作，用来生成新的RDD。
+ action通过运算，返回value给程序或者将data写到外存中。

注意有些操作，比如`join`，只有在键值对的RDD上才有效。Spark的函数名也和Scla等面向函数的语言一致，比如`map`是一对一的映设,`flatmap`是把一个输入值映射到一个或多个，类似MapReduce的map。这个描述太抽象，看看一些不错的[解释](https://www.iwwenbo.com/spark-map-flatmap/)，简单的可以理解为`flatmap`比`map`多了一个扁平化操作，就是先`map`，之后序列化/扁平化。

除了这些操作，还可以让RDD persist。而且用户可以得到RDD的partition order，通过Partitioner这个磊表示。

## 3.2 Example Applications

用两个iterativeapplication例子。

### 3.2.1 Logistic Regression

许多函数都是iterative，因为优化问题求解中，都用了gradient。

本身logistic regression的问题是这样：

首先定义$$h_w(x) = g(w^T \cdot x) = \frac{1}{1+e^-{w^T x}}$$, and $$\frac{\partial g(z)}{g(z)} = g(z)g(-z)$$

Minimize loss function $$\frac{1}{2}\sum_{i=1}^n (y_i - h_w(x_i))^2 = \frac{1}{2} \sum_{i=1}^n (y_i - \frac{1}{1+e^-{w^T x_i}})^2 $$

但是这个loss function并不是凸函数，得做一下转化，将问题转化为求最大似然估计：

$$\frac{1}{2} \sum_{i=1}^n ln(1+e^{-y_i\cdot w^T x_i})$$

Proof:

首先根据$$g(w^T x)$$的图，不难得出：

$$ P(y|x) = g(w^Tx)$$, when $$y = 1$$

$$ P(y|x) = 1- g(w^Tx) = g(-w^Tx)$$, when $$y = -1$$

To sum up, we can get $$ P(y|x) = g(y\cdot w^Tx)$$

and $$(x_1,y_1), (x_2,y_2) ... (x_n,y_n)$$ are independent.

Likelihood:

$$P(y_1...y_n|x_1,...x_n)=\prod_{i=1}^{n} P(y_i|x_i)$$

$$\max \prod_{i=1}^{n} P(y_i|x_i)  \Leftrightarrow \max \ln{\prod_{i=1}^{n} P(x_i|x_i)} = \max \sum \ln P(y_i|x_i) \Leftrightarrow \min -\frac{1}{n} \sum \ln P(y_i|x_i) $$

$$=\min \frac{1}{n} \sum ln \frac{1}{g(y_i\cdot w^T x_i)} = \min \frac{1}{n} \sum ln{(1 + e^{-y_i \cdot w^Tx_i})}$$

let $$ L(w) = \frac{1}{n} \sum ln{(1 + e^{-y_i \cdot w^Tx_i})}$$, then gradient is: $$\frac{1}{n} \sum y_i x_i \frac{1}{1+e^{y_i\cdot w^Tx_i}}$$

$$w^{t+1} = w^t - \alpha \cdot  y^T x \cdot \frac{1}{1+e^{y\cdot w^T x}}$$

 应该还有其他推断的方法，但是大致思路就是这样。

### 3.2.2 PageRank

PageRank是一个更加复杂的模式。算法为每一个文档迭代式的更新rank，通过增加连接到这个文档的其他文档的共享度。每一个迭代中，每个文档发送贡献度到$$\frac{r}{n}$$给它的邻居，r是它的rank，n是它的邻居个数。然后更新rank，设置为$$\alpha/N + (1-\alpha)\sum c_i$$，sum是它接收到邻居发来的contribution，而N是一共文档的个数。

`links.join(ranks)`这个就相当于实现了数据库的join操作，返回的是(URL, (outlinks, rank))。

flapmat里面的操作，首先最里面的，`links.map(dest=>(dest, rank/links.size)`是对每一个URL跟新contribution。然后`flatMap{(url, (links,rank)) => links.map(dest => ...) }`就是将我们刚才所得到的contribution和对应的URL组成对。PS.这里面比较难以理解的部分：`links.map(dest=>(dest, rank/links.size)`，这里的`dest`就是`flatMap{(url, (links,rank))`里的`links`。

这个程序的lineage在Figure 3。一个特征是，iteration越往后，消耗的时间就越长。因此，如果一个job有很多iteration，那么就需要复制多分`ranks`数据，来减少错误恢复时间。但是`links`ODD不需要复制，因为它是可以很容易地从输入数据恢复。

还有的优化PageRank的方案是，控制RDD的划分。如果为links确定了划分，比如用哈希函数哈希URL，就可以让同一个rank和links的URL在一个机器上，不需要网络通信。或者可以写一个Partitioner类，将相互链接的links放到一起。

# 4 Representing RDDs

实现RDD抽象的一个难点是如何追踪lineage。理论上，一个实现了RDD的系统能给用户提供尽可能详细的transformation操作，以便用户组合使用。这里提出一个基于图表示的RDD来满足这个需求。

总的来说，我们提出来用一个简单的接口来表示每一个RDD，每个RDD有五条信息：划分集合，是dataset的集合；依赖集合，在parent RDD上的依赖；根据parent计算依赖的函数；关于划分和dataset 放置的元数据。比如，RDD表示的HDFS文件，有这个文件每一个block的划分，以及在哪一台机器上。同时，这个RDD经过map之后的结果有一样的划分，只是在计算这个生成的子RDD时候，会将原（parent）RDD的元素进行map。

如何表示RDD之间的依赖：一种有效的方法是将以来分为两种：

1. narrow dependency：parent RDD的每一个划分最多被一个child RDD划分使用
2. wide dependency：parent RDD的每一个划分都可能被多个child RDD使用

比如`map`就是narrow dependency，而`join`就是wide（除非parent是哈希链接的）。这种区分方法的用处有两个。

+ narrow dependency允许在某一个节点上进行pipelined execution，计算所有的parent partition。比如，一个阶段可以先用`filter`，再用`map`（元素到元素的对应）。与此相反， wide dependency需要所以从parent partition的数据，然后用MapReduce操作进行shuffle。
+ 从节点失败进行回复的时候，narrow dependency效率更高，因为只有丢失的partent partition需要重新计算，而且可以在不同节点并行计算。而一个有wide dependency的lineage graph，一个节点的失败可能是所有的ancestor的几个partition丢失，那就是一个完整的re-execution。

RDD的接口使得在Spark中实现大部分transformation少于20行。下面简单的描述一些RDD实现：

HDFS 文件：input RDDs都是HDFS文件。对这些RDDs，`partitions`返回文件每一个block的划分，`preferredLocations`告诉节点block的位置，和`iterator`来读这个block。

map：调用RDD的`map`函数返回一个MappedRDD对象。这个对象和parent有一样的划分和preferred location，只是当它调用函数会在它的`iterator`方法中，映射调用parent的records。

union：在两个RDD上面调用`union`返回一个RDD，他的划分是两个parent的划分的union。每一个child partition都是narrow dependency。

sample：sampling和mapping类似，除了sampling的RDD会在每一个partition存储一个随机数生成器的种子，来确定性的从parent records进行sample。

join：将两个RDD join会导致或者两个narrow dependencies、或者两个wide dependencies、或者一个narrow一个wide。不管哪种情况，output RDD都会有一个partitioner。

# 5 Implementation

在Scala中用来大约14k行实现了Spark。系统在Mesos上跑，允许和Hadoop、MPI和其他应用共享资源。每一个Spark程序都当作一个独立的Mesos应用跑，有自己的driver（master）和worker；这些Spark应用彼此之间资源共享都是由Mesos控制。

Spark可以从Hadoop的输入资源（HDFS或者HBase）读取数据，并且在一个没有修改unmidified版本的Sala上面跑。下面简单写一下几个在系统中技术上实现很有意思的部分：

## 5.1 Job Scheduling

Spark的scheduler和Dryad很像，但是额外考虑了persistent RDD的哪一个划分在内存中可以用。每当一个用户在RDD上跑一个action，scheduler检查RDD的lineage graph来构造一个stages的DAG，按照stage来执行。每一个stage包含了尽量多的有narrow dependency的pipelined transformations。这些stages的分界线是wide dependency所需的shuffle操作，或者任何已经计算完的划分（将parent RDD的操作短路了）。scheduler然后发布任务来计算丢失的partition，直到计算到目标RDD。

scheduler给机器发布任务是基于数据本地性，并且用delay scheduling。如果一个任务需要用在某个节点内存中的数据，就把任务发给这个节点。如果一个任务处理的是partition，然后包含这个partition的RDD会有preferred locations，就发送到对应的preferred location节点。

对于wide dependencies，比如shuffle dependencies，将中间数据材料化materialize到包含parent partition的节点上，这样可以简化错误恢复，很像MapReduce中将map的输出materialize。

如果一个任务失败了，就将它重新跑在另一个节点上，只要这个stage的parent还在。如果一些stage已经得不到了，
会把这个任务重新提交，以此重新计算丢失的partition。

最后，尽管Spark所有的计算都是作为对driver程序的回应，也尝试了让cluster上的task用`lookup`操作，这个操作能根据key随机读取hash-partitioned RDD。这种情况下，任务需要告诉scheduler计算所需的partition（防止没有这个partition）。

## 5.2 Interpreter Integration

Scala有交互式脚本，类似Ruby和Python。因为用了in-memory数据，延迟会很低，所以希望是用户能够交互式地跑Spark，从解释器到大数据查询。

Scala解释器通常将一个代码的每一行当作一个class编译，加载到JVM，然后调用函数。这个class包括了单例对象，而单例对象包含那一行的变量或者函数，然后在一个initializer中跑这行代码。例子：如果有两行代码
```
var x = 5
println(x)
```
解释器会定义一个class叫`Line1`包含`x`，并让第二行编译`println(Line1.getInstance().x`。在Spark实现中，对Scala中的解释器做了两个调整：
1. class shipping：为了让worker节点获取每一行对应的class的bytecode，我们将通过HTTP提供interpreter操作。
2. modified code generation：通常每一行产生的单例对象通过这个class的静态方法来访问，也就是说当我们序列化闭包指向一个定义在前一行的变量，比如例子中的`Line1.x`，Java不会追踪图来转移`x`周围的实例，因此worker节点不会接收到`x`。我们修改了代码生成逻辑来直接引用每一行对象的实例。

## 5.3 Memory Management

Spark提供了三种persistent RDD的方法：in-memory storage as deserialized Java object, in-memory storge as serialized data, and on-disk storage。第一个最快，因为JVM可以直接访问每一个RDD元素。第二个是在空间有限时，更加高效利用内存的方法，（相比于第一个方法，直接把Java对象存放在JVM中）。第三个方法适合RDD太大而不得不房子disk上。

为了管理有限的内存情况，可以用LRU驱逐策略。当一个新的RDD partition产生，然后没有空间存储它，就将最久未访问的partition释放给新生成的，除非两个partition都是来自同一个RDD；最后这个条件很重要，因为大部分操作都是对应于一整个RDD，所以这个已经在内存中的partition也会用到。

最后，每一个Spark实例都有自己独立的内存空间。将来可能会考虑在不同的Spark实例之间共享RDD，通过一个统一的内存管理。

## 5.4 Support for Checkpointing

尽管可以用lineage来恢复RDD，这个恢复可能会对于lineage很长的任务非常耗时。因此设置checkpoint到stable storage会有帮助。

总的来说，checkpointing对RDD with long lineage graph很有用，比如PageRank。这种个例子中，节点失败会导致从每个parent RDD的部分数据丢失，就需要我们进行一个完整的recomputation。相反，对于在stable storage data的narrow dependency的RDD，比如logistic regression的例子，和PageRank中的link list，checkpoint就没有太大作用。如果一个节点挂了，丢失的数据可以并行重新计算。代价仅仅是复制整个RDD。

Spark提供了checkpointing的API，是否使用看用户。现在在做是否能够自动设置checkpoint，因为scheduler是知道每一个dataset的大小和计算的时间的。

最后，因为RDD的只读性，使得设置checkpoint比传统的shared memory系统方便。因为不需要考虑一致性。

# Appendix
[Resilient Distributed Datasets: A Fault-Tolerance Abstraction for In-Memory Cluster Computing](http://dl.acm.org/citation.cfm?id=2228301)

[关于几个函数的逻辑图](https://github.com/JerryLead/SparkInternals/blob/master/markdown/2-JobLogicalPlan.md)

[怎么还是作业帮系列——这一份介绍Spark API，很不错](https://www.zybuluo.com/jewes/note/35032)