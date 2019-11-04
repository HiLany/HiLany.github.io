---
layout:     post
title:      Greenplum优化
subtitle:   Greenplum常见性能问题分析
date:       2019-11-02
author:     LANY
catalog: true
tags:
    - Pivotal
    - Greenplum
    - MPP
    - 大数据
    - 性能调优
---

# 性能问题的常见原因

这篇文章解释了常见性能问题的诊断过程以及这些问题的潜在解决方法。

## 区分硬件以及Segment故障

Greenplum数据库的性能取决于它所运行的硬件以及IT基础设施。Greenplum有多个服务器（主机）组成，它们共同充当一个内聚系统（阵列）。诊断性能问题的第一步就是确保GreenplumDatabase的所有Segments都在线。GreenplumDatabase的性能的取决于所有主机中最慢的那台（短板效应）。CPU利用率、内存管理、IO处理以及网络负载的问题都会影响其性能。常见的与硬件相关的问题如下：

- **磁盘故障**

如果你的磁盘做了RAID，那么即使是当个磁盘的故障也不会显著影响数据库的性能。但是在有磁盘故障的主机上进行磁盘重新同步也会消耗资源。通过`gpcheckperf`工具可以帮助你分辨哪台主机上有磁盘IO问题。

- **主机故障**

当一台服务器宕掉了之后，那么在这台主机上的所有segments都会变得不可操作。这就意味这在这组主机中的剩余主机必须承受它们平常工作的两倍负载，因为那些主机上也运行了PrimarySegments以及多个mirrors。如果mirror(镜像)变得不可用，那么服务会暂时被中断直到恢复了存在故障的segments。通过`gpstate`工具可以帮助你分辨哪些segments发生了故障

- **网络故障**

网卡、交换机以及DNS服务器的故障也会使segments宕掉。如果主机名或者IP地址在Greenplum的主机之间不能被解析。那么在数据库中会显示interconnect错误。通过`gpcheckperf`工具可以帮助你分辨哪台主机上有网络故障问题。

- **磁盘容量**

每台主机上的磁盘使用量不应该超过该主机总磁盘容量的70%。因为GreenplumDatabase中的运行时进程需要一些空间。要回收删除行所占用的空间，你可以在数据加载或者更新之后运行`VACUUM`命令回收。你可以通过数据库中的`gp_toolkit`管理模式里面的视图检查分布式数据库对象的大小。

## 管理工作负载

数据库系统具有有限的CPU能力、内存以及磁盘IO资源。当多个工作负载同时争用这些资源时数据库的性能会受到影响。当系统遇到各种业务需求时资源管理会最大化其吞吐量。GreenplumDatabase系统了资源队列和资源组帮助你管理这些系统资源。

资源队列和资源组限制了资源的使用量以及在特定的队列或者组中总的并发查询执行数量。 将数据库角色分配到合适的队列或者组中，管理员可以控制并发用户查询并阻止系统过载。

管理员应该将维护工作比如`VACUUM ANALYZE`操作放到数据库压力较小的时候执行，比如每天的凌晨。

## 避免竞争

当多个用户或者工作负载在共同使用系统的资源时会产生竞争。举个例子，当两个事务尝试更新同一张表的时候会争用这个表资源。寻求表记锁或者行级锁的事务会长时间等待直到冲突锁被释放。应用程序不应该长时间保持事务打开。比如说在等待用户输入时。

## 维护数据库统计信息

GreenplumDatabase使用依赖于数据库统计信息的基于cost的查询优化器。精准的统计信息会让查询优化器更好的预估查询所需要检索的行数从而选择更高效的查询计划。如果没有数据库统计信息，查询优化器不能预估有多少记录会被返回。优化器并不假设它有足够的内存来执行某些操作，比如聚合，因此它采取最保守的操作，并通过从磁盘读写来执行这些操作。这相对于在内存中做这些事情来说时非常的慢。`ANALYZE`收集统计了查询优化器所需要的有关数据库统计信息。

> 当通过GPORCA优化器执行一个SQL命令时，如果该命令的性能可以通过手机字段或者SQL中所涉及到的字段集合中的信息而得到提升的话，GreenplumDatabase会提出一个警告。这个警告会在命令行中被提出并记录到日志中。

## 区分查询计划中的统计信息问题

在你使用`EXPLAIN`或者`EXPLAIN ANALYZE`解析查询计划前，你需要熟悉一下你的数据来帮助你区分可能的统计信息问题。

参考[如何解读Greenplum执行计划](https://lanyang.io/2019/02/19/%E5%A6%82%E4%BD%95%E8%A7%A3%E8%AF%BBGreenplum%E8%A7%A3%E6%9E%90%E8%AE%A1%E5%88%92/)

## 统计信息收集调优

下面的两个配置参数控制着统计信息收集样本的数据量

- default_statistics_target
- gp_analyze_relative_error

这两个参数控制系统级的统计信息样本。最好只对查询谓词中使用最频繁的列进行取样。调整取样的命令：

```SQL
ALTER TABLE ... SET STATISTICS
```

举个例子：

```SQL
ALTER TABLE sales ALTER COLUMN region SET STATISTICS 50;
```
这条命令等效于为特定的列改变`default_statistics_target`。后续的`ANALYZE`操作将会为这列生成更多的统计信息并生成更好的查询计划作为结果。


## 优化数据分布

当你在GreenplumDatabase中创建一张表的时候，你必须为这张表声明一张分布键来允许跨系统中的所有segments进行数据的均匀分布。由于一个query是并行工作在segment上的，数据库的快慢取决与系统中最慢的那个segment。如果数据分布的不均匀，一个segment有很多的数据那么返回的数据会非常的慢，因此整个系统也会变得缓慢。

## 优化你的数据库设计

大部分的性能问题可以通过数据的设计得到提升。检查你的数据库设计并考虑如下几条：

- 模式是否反应了数据的访问方式？
- 大表是否可以分区？
- 你是否是使用最小的数据类型去存储列的值？
- 关联表的时候，所关联的字段是否是同一类型？
- 是否用到了索引？

## GreenplumDatabase最大限制

| 维度| 限制|
|:---:|:---:|
|Database Size|Unlimited|
|Table Size| Unlimited,每个段每个分区128TB|
|Row Size| 1.6TB(1600 columns * 1GB)|
|Field Size| 1GB|
|Rows per Table| 2^48|
|Columns per Table/View| 1600|
|Indexes per Table| Unlimited|
|Columns per Index| 32|
|Table-level Constraints per Table|Unlimited|
|Table Name Length| 63Bytes(Limited by name data type)|

在Greenplum数据库中，无限维度并没有本质上的限制。但是，实际上它们仅限于可用的磁盘空间和内存/交换空间。当这些值异常大时，性能可能会受到影响。

