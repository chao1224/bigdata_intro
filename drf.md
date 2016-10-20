# Donimant Resource Fairness: Fair Allocation of Multiple Resource Types

Ali Ghodsi, Matei Zaharia, Benjamin Hindman, Andy Konwinski, Scott Shenker, Ion Stocia

UC Berkeley

为了解决resource allocation问题，提出了 Dominant Resource Fairness（DRF），是不同资源的max-min fairness实现。特性：

1. DRF保证所有用户都能公平得分配到资源
2. DRF是策略上得到证明的，保证不会有用户通过说谎而得到更多的资源
3. DRF是envy-free，没有用户能和其他用户进行资源交换
4. DRF是pareto efficient，没次分配，都是针对于某一个用户，在给定的资源数情况下的最优

# Introduction
最大化系统中用户接受到的最小分配。max-min fairness一般化到了加权max-min fairness，可以支持很多资源分配策略，比如优先级、预存、和基于deadline的分配。

给定这些特征，那么就不会奇怪有大量的（加强）max-min fairness，比如round-roubin，proportional resource sharing，和weighted fair queueing。

Hadoop和Dryad的fair schedulers，将资源分配在固定大小的partition，叫做slots。这篇paper提出了fair allocation的解决方案。尤其是提出来DRF，一种一般化的max-min fairness。DRF的想法是用户的资源分配由dominant share定义，也就是用户能被分配的任意资源的最大share。DRF就是为了最大化所有用户的dominant share。

已经在Mesos中实现了DRF，Mesos是一个resource manager，把DRF和slot-based调度比较，DRF占优。

# 2 Motivation

multi-resources指的是不同类型的resources，而不是同一个资源的不同实例。

在图1中，大部分任务是CPU-heavy，许多reduce tasks是memory-heavy。

现有的cluster中fair scheduler，比如Quincy和Hadoop Fair Scheduler，都忽略了用户需求的异质性，并按照slots分配资源，而每一个slot都是每个节点上固定的。因为slot会导致poor match，效率低。图1中，ratio为1表示刚好match，大部分都是over-和under-utilization。

# 3 Allocation Properties

现在考虑设计一个max-min fair allocation策略。考虑有9个CPU和18G RAM，两个用户：用户A的tasks需要(1CPU,4GB)，而用户B(3CPU,1GB)。一种分配策略是每个人都给一半的资源。另一种可能性是将整合后的资源（CPU+memory）平均化。尽管能想到很多策略，但是如何比较也是一个问题。

先考虑用fair allocation解决这类问题下应该满足的性质：

1. sharing incentive：每个用户都应该通过分享资源获得更好的性能，与独立占用一部分资源相比。如果一个机器有一样的节点和n各用户，那么用户不能改被分配到所有资源的1/n。
2. stretegy-proofness：用户不能改通过说谎而获取更多的资源。
3. envy-freeness：一个用户不能偏好其他用户的分配。
4. pareto efficiency：如果要增加一个用户的分配，另外至少一个用户的分配会减少。

strategy-proofness和sharing incentive这两个性质更重要一点。前面已经讨论过的cloud operator（map和reduce），显示strategy-proofness很重要。比如Yahoo！的Hadoop MapReduce系统有不同slots的map和reduce，有一个用户发现map slots符合，然后只要有reduce slots就把所有的任务启动。另外一个大公司系统提供了只有用户能够保证高利用率才给分配机器，这导致了用户会用无限循环来人工提高使用率。

满足了sharing incentive性质的系统都能保证系能独立性，因为保证了每一个用户分配到的最小额度（不会低于1/n），而不用管其他用户的请求。

对于单个资源，很容易证明max-min满足以上性质，但是多资源就不那么容易了。

考虑四个额外的性质：

1. single resource fairness：
2. bottleneck fairness：
3. population monotonicity：一个用户离开系统，释放资源，其他的用户的资源不会减少。
4. resource monotonicity：如果系统增加了新的资源，每个用户的资源不会减少。

# 4 Dominant Resource Fairness(DRF)

提出了DRF，多资源分配的一种新策略，满足了前一个section提出的所有性质。对每一个用户，DRF计算了每一个分配到它的资源的share。一个用户分配到的最大的share就是这个用户的dominant share，然后这个被分配到的资源就叫做dominant resource。不同用户有不同的dominant resource。一个用户的任务需要跑大量的计算，那么dominant resource就是CPU；一个需要大量IO的dominant resource就是bandwidth。DRF最大化系统中最小的dominant share。

## 4.1 An Example

考虑一个系统，有9 CPU，18 GB RAM，两个用户：用户A的tasks需要(1CPU,4GB)，而用户B(3CPU,1GB)。

这个例子中，用户A需要所以CPU中的1/9，RAM的2/9，所以dominant resource是RAM。B则是占总共CPU的1/3，RAM的1/18，所以dominant resource是CPU。DRF会按照图三，给A三个tasks，B四个tasks。这样，A和B的dominant share一样，都是2/3。

解决方法就是转化成一个优化问题

$max(x,y)$

st.

$ x + 3y \le 9$

$4x + 4 \le 18$

$ \frac{2x}{9} = \frac{y}{3}$

最后一个是equalize dominant shares，但是不是必要的。当一个用户的total demand满足了，那么就不需要更多的tasks。另外，如果一个资源用尽了，用户不需要这个资源，但是会受到更多其他资源。

## 4.2 DRF Scheduling Algorithm

算法1进行了很好的描述，最主要的一点就是每次从已有的用户中，选择dominant share最小的进行分配资源。

在这个例子中，会在CPU耗尽的时候停止。但是在一般化的场景下，很可能会在某个资源耗尽之后继续执行，因为新的task可能不是用耗尽的资源。

## 4.3 Weighted DRF

实际中，用户之间公平分配资源不是很好的选择。我们会希望给某些用户（跑的任务更重要的）多一些资源，为了实现这个，提出了加权weighted DRF。

每一个用户都给定了一个加权向量，$w_{i,j}$表示对于第i个用户，资源j的权重。那么用户i的dominant share就是$max_j\{u_{ij}/w_{ij}\}$，期中$u_ij$是第i个用户j资源的share。

# 5 Alternative Fair Allocation Policies

另外两种分配策略。asset fairness，尽量平均给每一个用户分配的所有资源和。competitive equilibrium from equal incomes（CEEI），一种在微观经济领域受欢迎d公平分配策略。

## 5.1 Asset Fairness

Asset Fairness的主要想法是不同资源同样的share，同样重要。1%的CPU和1%的bandwidth一样。然后就尽量给每一个用户分配的资源和一样。还是考虑4.1的例子，那么会有最后equivalent share变成

$6x = 7y$， 表示每一个用户都用异样的资源。也就是如下：


$max(x,y)$

st.

$ x + 3y \le 9$

$4x + 4 \le 18$

$ 6x = 7y$

但这个策略违反了sharing incentive性质。会有用户获得少于1/n的资源。

## 5.2 Competitive Equilibrium from Equal Incomes

在微观经济领域，公平划分资源的方式就是Competitive Equilibrium fromEqual Incomes（CEEI）。每个用户最初收到了1/n的资源，然后会和其他用户进行资源交换。CEEi的结果满足envy-free和pareto efficient。CEEI的分配策略是有纳什均衡提出的，选择可行的方案，来最大化$\prod u_i(a_i)$，$u_i(a_i)$是用户i从分配$a_i$中得到的效用。


$max(x*y)$

st.

$ x + 3y \le 9$

$4x + 4 \le 18$

这个方案满足了envy-free和pareto efficient，但是不满足strategy-proof（可以有人通过说谎获取更多的资源）。

## 5.3 Comparison with DRF

图4给出了直观的比较。DRF保证了dominant（面积）一样。Asset Fairness保证了所有资源和一样。CEEI假设是在一个完美的竞争中的市场，每一个资源都被分配。但是用户可以通过说谎来获取更多的资源。 

# Appendix

[pdf](https://people.eecs.berkeley.edu/~alig/papers/drf.pdf)