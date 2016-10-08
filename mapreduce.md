# Introduction

MapReduce作为一种编程模型Programming Model，同时支持了处理和生成大数据集合。用户定义map函数和reduce函数。map函数会处理一个键值对key-value pair，来生成一系列的中间键值对intermediate key-value pair；reduce函数将所有的intermediate key-value pair根据intermediate key融合。而现实生活中的许多问题都能映射到这两个操作上。其实这个就像以前我在写算法题一样，基本的算法题都可以归纳为搜索状态，而搜索的方式不外乎分割处理（自上到下）或者合并处理（自下到上）。

这个系统会帮助用户处理比如分割输入数据、安排程序执行、处理宕机、和机器间交互，从而使程序员不用关注分布式和并行处理。

Jeff和其他谷歌的程序员已经实现了很多不同的带特殊目的地计算，已至此处理大量raw data，比如抓取的文本、网页日志，来计算不同的数据，比如inverted indices，不同网页文件的图结构表示，每天最常用的query等。这些计算都很直白，但是输入的数据量非常大，并且输入的数据都分布在不同的机器上。如何并行计算、分布数据并处理失败，是需要解决的关键问题。

解决的方案就是设计了一个新的抽象，这个抽象允许我们能运行程序、但是同时将技术细节隐藏在了库里面，技术细节比如上面提到的并行计算、数据分布、容错能力和负载平衡。抽象的描述受到了Lisp和其他面向函数语言中的map和reduce原始操作的启发，很多甚至大部分操作都可以由map和reduce完成。这种处理方案也能更好地支持大量数据的并行处理，使用重复运行作为解决容错的主要机制。

# Programming Model

输入是一系列键值对，输出也是键值对。

Map的输入是键值对，产生中间键，intermediate key，和指向这个key的数值集合。MapReduce库会把同一个key的库的value集合合并，传递给Reduce函数。

Reduce函数接受中间键以及这个键指向的数值。它将这些数值压缩到一个更小的数据集，通常每个reduce操作没有或者只有一个输出。

## 例子

经典例子，计算一个大文档集合中每个单词出现的次数。用户会用如下的伪代码：

```
map(String key, String value):
    // key: 文档名字
    // value：文档内容
    for word in value:
        EmitIntermediate(word,'1')
        // 对某个文档中的每个单词遍历，并将这个单词以<word,1>作为中间键值对。

reduce(String key, String value):
    // key：单词
    // value： 一系列数数的内容（在这里就是一堆1）
    result = 0
    for v in values:
        result += v
        // 这是MapReduce最naive的做法
        // 如果某个单词出现了k次，那么就循环k次，每次+1
    Emit(result)
```

注：这里的reduce函数每次执行只是针对于一个key值。

然后用户就会写一个MapReduce specification，包括输入输出文件名，然后调用上面我们写的map和reduce函数。MapReduce specification类似如下：

```
    Configuration conf = new Configuration();
    String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
    Job job = new Job(conf, "separate anagram");

    job.setJarByClass(anagram.class);
    job.setMapperClass(FirstMapper.class);
    job.setReducerClass(FirstReducer.class);

    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(Text.class);
    job.setOutputKeyClass(NullWritable.class);
    job.setOutputValueClass(Text.class);
```

这里类似`setMapOutputKeyClass`的四个类即为设置输入输出类型。

## 类型

通过上面，我们不难发现map和reduce的输入输出格式有某些关系：

```
    map       (k1, v1)        --->list(k2,v2)
    reduce    (ke, list(v2))  --->v2
```

不难发现输入和输出的键值对的domain都不一样，并且中间键值对和输出的键值对domain一样。

## 跟多例子

+ 分布式查找distributed grep：每一个map函数，如果某一行和某种模式符合的时候，就将这一行字符串输出。然后reduce函数只是将输入复制到输出。

+ URL访问计数：map将每个网页请求的日志作为输出，intermediate是<URL,1>，而reduce函数讲对每个URL进行计数，输出<URL,total count>。

+ 逆网络连接图reverse web-link graph：map函数的输出是<target,source>，表示对source网页的每个target URL。reduce函数则是对于同一个target URL，找到所有包含这个URL的页面list，<target,list<source>>。

+ 每个主机的术语向量term-vector per host：术语向量term vector总结了某个文档或者文档集合中出现的最重要的单词，作为<word,frequency>对。map函数输入是文档，输出是<hostname,term vector>，hostname可以从文档的URL中获取。reduce函数则是对于某一个hostname，集合了所有的term vector，去掉频繁率比较低的项，然后输出<hostname,term vector>。

+ 倒排索引inverted index：map函数对于每一个输入的文档，输出<word,document ID>作为intermediate键值对。reduce函数接收对于某一个单词的所有键值对，按照文档ID排序，输出<word,list<document ID>>。

+ 分布式排序distributed sort：map函数对于每一条记录取出键值，输出<key,record>。reduce函数？？？

# Implementation

描述了MapReduce实现接口，以及如何如何适用到谷歌内部。

Map调用是通过将输入数据分成M份而分散在不同的机器上，而数据的分割可以并行处理。reduce调用的分布是根据将中间键空间分割成R份。

1. 用户程序中的MapReduce库会先将输入数据分割成M份，每份通常是16-64MB。然后开始很多份程序。

2. 其中一份叫master的程序比较特殊。其他程序都是被master分到了worker上。有M个Map和R个Reduce任务需要分配。

3. 被分配到map任务的worker会读取相对应的数据。它将输入数据中的键值对输出到用户定义的map函数中，而生成的中间键值对就暂存在内存里。

4. 间隔性的，缓冲的键值对就会写到本地磁盘，按照分配函数分到R个区域。这些键值对所在的本地磁盘信息会返回给master节点，而master节点则会将这些位置传递给reduce节点。

5. 当一个执行reduce的worker从master那里接收到intermediate的位置信息，它就调用RPC来从map worker的本地磁盘读取数据。当一个reduce worker读完所有的intermediate数据，它就会根据intermediate key进行排序，因此同样key的键值对会分到一组。排序的原因是这样可以让同一个key的键值对分到一起。如果数据量太大，那么就需要external sort（即当内存不够时，利用硬盘存储）。

6. reduce worker会遍历排序后的中间键值，将键值和数据传递给reduce函数。reduce函数的output会添加append到最终输出文件。

7. 当所有的map和reduce函数都完成，master会唤醒用户程序。然后用户程序里的MapReduce调用就会返回给用户代码？？？

执行完成之后，output数据在R个输出文件内，每一个reduce任务针对一个输出结果。用户不需要将这R个输出文件合并到一个，因为他们也经常是另外一个MapReduce调用的输入（n个连在一起），或者另外一个分布式应用会直接使用这些分布式的数据。（后面会提到，现在看的话，不确定类似Hive、Pig的程序是否会这么调用它）

## Master数据结构

master节点保持好几个数据结构。对每一个map和reduce任务，它保存了这个任务的状态和worker机器的识别（仅针对与非空闲的任务）。



# Refinements

描述了几个有助于编程模型的提升refinement。

# Performance 性能

# Appendix

每次提到MapReduce，大家都会想到Jeff Dean，这里主要也是参考了他和Sanjay Ghemawat的paper。这篇发表在2008年[Communication of the ACM\(CACM\)](http://cacm.acm.org/)上的文章，引用量在我写这些文字的时候超过了18k [MapReduce: Simplified Data Processing on Large Clusters](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)

