---
title: Borg 论文笔记
date: 2021-03-27 16:52:56
tags:
---


## Borg 论文笔记

### Google的Borg系统是一个集群管理器。它在多个万台机器规模的集群上运行着来自几千个不同应用的几十万个作业。
Borg通过准入控制、高效的任务装箱、超售、机器共享、以及进程级别的性能隔离，实现了高利用率。它为高可用应用提供了可以减少故障恢复时间的运行时特性，以及降低关联故障概率的调度策略。Borg提供了声明式的作业描述语言、域名服务集成、实时作业监控、分析和模拟系统行为的工具。这些简化了用户的使用。
本文介绍了Borg系统架构和特性，重要的设计决策，对某些策略选择的定量分析，以及十年来的运营经验和教训。
#### Borg 集群管理系统负责接收、调度、启动、重启和监控Google所有应用。
Borg 提供三个主要好处
- 隐藏资源管理和故障处理细节，使用户可以专注于应用开发
- 高可靠和高可用运维，并支持应用程序也这样。
- 让我们可以在几万台机器上高效运行负载。

![](borg.png)
<!-- more -->
## Borg 的用户是google开发人员和站点可靠性工程师(SRE),用户以作业(Job)的方式将工作提交给Borg，作业由一个或多个任务(Task)组成，每个任务执行相同的二进制程序，每个作业只运行在一个Borg单元(Cell)里，是一组机器的管理单元。
SRE职责比系统管理员多很多: 他们负责 Google 生产服务的工程师，他们也设计和实现包括自动化系统等软件，管理应用，服务基础设施和平台，以保证在 Google 如此大规模下的高性能和高可靠性。
### 工作负载
Borg Cell 主要运行2种异构的工作负载
1. 永不停止的长期运行的服务，处理持续时间较短但对延迟敏感的请求(从几毫秒到几百毫秒)，这些服务用于面对最终用户的产品，如Gmail、Google Docs、网页搜索，以及内部基础设施服务(例如Bigtable)
2. 批处理作业，执行时间从几秒到几天，对短期性能波动不敏感，这2种负载在不同Cell中比例不同，取决于其主要租户(例如，有些Cell以批处理作业为主)，工作负载也随着时间变化: 批处理作业不断提交或者技术，而很多面向终端用户的服务表现出昼夜周期性的使用模式，Borg 需要都处理好这些情况。
Borg 的代表性负载是一个公开的2011年5月正月的记录数据集【80】这个数据集已经得到了广泛的分析[1,26,27,57,68]
最近几年，以Borg为基础构建了很多应用框架，包括MapReduce系统、FlumeJava、Millwheel和Prege，这些框架大多有一个控制器来提交Master Job，还有多个Worker Jon，前两个框架类似于YARN的应用管理器，我们的分布式存储系统，例如GFS和它的后继者CFS、Bigtable、以及Megastore，都是运行在Borg上的。
我们把高优先级的 Borg 作业称为生产作业(prod),其他的则是非生产的(non-prod)，大多数长期服务是prod的，大部分批处理作业是non-prod的，一个典型cell里，prod作业分配了约70%的总CPU资源，占总CPU使用量约60%,分配了约55%的总内存资源，占总内存使用量约85%，5.5节表名分配量和使用量的差异是值得注意的。
