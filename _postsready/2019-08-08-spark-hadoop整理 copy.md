---
layout: content
title: spark-hadoop整理
status: 1 
category: java
author:   "yimuren"
tags:
    - java
---

## 一. 前言


## 二. python环境
Hadoop工程包括以下模块：
- Hadoop Common：支持其他Hadoop模块的通用工具。
- Hadoop Distributed File System(HDFS)：提供高吞吐量的访问应用数据的一个分布式文件系统。
- Hadoop YARN：一种作业调度和集群资源管理的框架。
- Hadoop MapReduce：一种基于Yarn来处理大数据集合的系统。

Apache中其他Hadoop相关的项目包括：
- Ambari：一种用于提供、管理和监督Apache Hadoop集群的基于Web UI的且易于使用的Hadoop管理工具。
- Avro：一种数据序列化系统。
- Cassandra：一种无单点故障的可扩展的分布式数据库。
- Chukwa：一种用于管理大型分布式系统的数据收集系统。
- HBase：一种支持存储大型表的结构化存储的可扩展的分布式数据库。
- Hive：一种提供数据汇总和特定查询的数据仓库。
- Mahout：一种可扩展的机器学习和数据挖掘库（Scala语言实现，可结合Spark后端）。
- Pig：一种高级的数据流语言且支持并行计算的执行框架（2017年发布的最新版本0.17.0是添加了Spark上的Pig应用）。
- Spark：一种用于Hadoop数据的快速通用计算引擎。Spark提供一种支持广泛应用的简单而易懂的编程模型，包括ETL（ Extract-Transform-Load）、机器学习、流处理以及图计算。
- Tez：一种建立在Hadoop YARN上数据流编程框架，它提供了一个强大而灵活的引擎来任意构建DAG（Directed-acyclic-graph）任务去处理用于批处理和交互用例的数据。
- ZooKeeper：一种给分布式应用提供高性能的协同服务系统。

选择spark的原因
- 易用性（可以使用Java，Scala，Python，R以及SQL快速的写Spark应用）Spark提供80个以上高级算子便于执行并行应用，并且可以使用Scala、Python、R以及SQL的shell端交互式运行Spark应用。
- 通用性(支持SQL，流数据处理以及复杂分析) Spark拥有一系列库，包括SQL和DataFrame，用于机器学习的MLib,支持图计算GraphX以及流计算模块Streaming。你可以在一个应用中同时组合这些库。
- 支持多种模式运行（平台包括Hadoop,Apache Mesos,Kubernete,standalone或者云上，也可以获取各种数据源上的数据）

spark和mapreduce的区别
- MapReduce是第一代计算引擎，那么Spark就是第二代计算引擎。
- MapReduce的核心是“分而治之”策略。数据在其MapReduce的生命周期中过程中需要经过六大保护神的洗礼，分别是：Input、Split、Map、Shuffule、Reduce和Output。
- MapReduce框架采用Master/Slave架构，一个Master对应多个Slave，Master运行JobTracker，Slave运行TaskTracker；JobTracker充当一个管理者，负责Client端提交的任务能够由手下的TaskTracker执行完成，而TaskTracker充当普通员工执行的Task分为Map Task（Split and Map）和Reduce Task(Shuffle and Reduce)。

MapReduce的缺点
- 抽象层次低，具体的Map和Reduce实现起来代码量大并且对于数据挖掘算法中复杂的分析需要大量的Job来支持，且Job之间的依赖需要开发者定义，导致开发的难度高而代码的可读性不强；
- 中间结果也存放在HDFS文件系统中，导致中间结果不能复用（需要重新从磁盘中读取），不适宜数据挖掘算法中的大量迭代操作，ReduceTask需要等待所有的MapTask执行完毕才可以开始；
- 只适合批处理场景，不支持交互式查询和数据的实时处理。

Spark中的三大概念：
1. RDD（Resilient Distributed Dataset）。实际上对与开发人员而已它是以一种对象的形式作为数据的一种表现形式而存在，可以理解为一种你可以操作的只读的分布式数据集，之所以称之为有弹性，在于：
- RDD可以在内存和磁盘存储间手动或自动切换；
- RDD拥有Lineage（血统）信息，及存储着它的父RDD以及父子之间的关系，当数据丢失时，可通过Lineage关系重新计算并恢复结果集，使其具备高容错性；
- 当血统链太长时，用户可以建立checkpoint将数据存放到磁盘上持久化存储加快容错速度（建议通过saveAsTextFile等方式存储到文件系统），而persist方式可以将数据存储到内存中用于后续计算的复用；
- RDD的数据重新分片可以手动设置。在Spark中执行重新分片操作的方法有repartition和coalesce两个方法，这两个方法都是手动设置RDD的分区数量，repartition只是coalesce接口中参数shuffle=true的实现；是否重新分区对性能影响比较大，如果分区数量大，可以减少每个分区的占存，减少OOM（内存溢出）的风险，但如果分区数量过多同时产生了过多的碎片，消耗过多的线程去处理数据，从而浪费计算资源。

2. Transformations。转换发生在当你将现有的RDD转换成其他的RDD的时候。比如当你打开一个文件然后读取文件内容并通过map方法将字符串类型RDD转换成另外一个数组类型RDD就是一种转换操作，常用的转换操作有map,filer,flatMap,union,distinct,groupByKey等。

3. Actions。动作发生在你需要系统返回一个结果的时候。比如你需要知道RDD的第一行数据是什么样的内容，比如你想知道RDD一共有多少行这样类似的操作就是一种动作，常用的动作操作有reduce,collect,count,first,take(),saveAsTextFile(),foreach()等。

选择spark的原因
- 首先，Spark通过RDD的lineage血统依赖关系提供了一个完备的数据恢复机制；
- 其次，Spark通过使用DAG优化整个计算过程；
- 最后，Spark对RDD进行Transformation和Action的一系列算子操作使得并行计算在粗粒度上就可以简单执行，而且Spark生态系统提供了一系列的开发包使得数据科学家可以执行一系列的SQL、ML、Streaming以及Graph操作，而且还支持很多其他的第三方包括一些交互式框架类似于Apache Zeppelin，地理数据可视化框架GeoSpark以及一些比较流行的深度学习框架Sparking-water,Deeplearning4j,SparkNet等。





参考文章
- https://www.cnblogs.com/wing1995/p/9300120.html