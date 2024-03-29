---
categories:
- k8s
date: 2023-05-14 21:35:12
tags:
- 思考
title: cncf x aliyun 云原生公开课笔记
---

## K8s 核心概念

- 调度
- 自动修复
- 水平伸缩

### k8s 架构

典型的两层 client-server 架构

### master 核心组件

- API Server
- Controller
- Scheduler
- etcd

<!--more-->

#### Pod

容器的本质实际上是一个进程，是一个视图被隔离，资源受限的进程

- k8s -> 云操作系统
- 容器镜像 -> 云操作系统安装包

#### 进程组

容器是单进程模型，管理容器就是管理进程，一个程序有多个进程

pod = "进程组"

一个应用的多个进程定义为多个容器启动起来，定义在一个 pod 里面

Pod 在 Kubernetes 里面只有一个逻辑单位，没有一个真实的东西对应说这个就是 Pod，真正起来在物理上存在的东西，就是四个容器。

### Pod -> K8s 原子调度单位 Why?

- 容器之间紧密协作，有依赖 Task co-scheduling 关系

### 超亲密关系

- 亲密关系：调度解决
  - 两个应用需要运行在同一个宿主机上
- 超亲密关系: pod 解决
  _ 会发生直接的文件交换
  _ localhost 或者 socket 本地通信
  _ 频繁的 RPC 调用
  _ 共享 linux namespace(etc,一个容器要加入另一个容器的 network namespace) \* ...

### Pod 实现

- 共享网络 \* infra container，整个 pod 内的 container 共享同一个网络，其他所有容器加入 infra container 的 network namespace，启动时候 infra container 先启动，整个 Pod 的生命周期是等同于 Infra container 的生命周期
- 共享存储 \* pod 内多个容器挂载同一个 volume

### 容器设计模式

Init Container 按照定义顺序做初始化工作

### Sidecar

在 Pod 里面，可以定义一些专门的容器，来执行主业务容器所需要的一些辅助工作

- ssh
- 日志收集
- debug
  辅助功能解耦业务，可以独立发布 sidecar 容器

### Sidecar 代理容器

- 代理容器对业务容器屏蔽被代理的服务集群，简化业务代码的实现逻辑
  _ 容器之间通过 localhost 直接访问
  _ 代理容器的代码可以对全公司复用

#### Sidecar 适配器容器

### 设计模式本质 : 解耦和复用

## 应用编排管理

1. k8s 资源元信息
   - Spec: 期望状态
   - Status: 观测到的状态
   - Labels: 资源注解
   - Annotations: 多个资源之间相互关系的 OwnerReference
2. Labels
   - key: value 元数据
   - 筛选资源，唯一的组合资源方法
   - 集合型 selector
3. Annotations
   - 系统或者工具用来存储资源的非标示性信息
   - 扩展资源的 spec/status 的描述
   - 特点
     _ 一般比 label 更大
     _ 可以包含特殊字符 \* 可以结构化也可以非结构化
4. Ownerreference
   - 所有者，一般就是指集合类的资源
   - 集合类资源的控制器会创建对应的归属资源。比如：replicaset 控制器在操作中会创建 Pod，被创建 Pod 的 Ownereference 就指向了创建 Pod 的 replicaset，Ownereference 使得用户可以方便地查找一个创建资源的对象，另外，还可以用来实现级联删除的效果。

### 控制器模式

1. 控制循环

   - 控制器
   - 被控制的系统
   - 能跟观测系统的传感器

   外界通过修改资源 spec 来控制资源，控制器比较资源 spec 和 status，从而计算一个 diff，diff 最后会用来决定执行对系统进行什么样的控制操作，控制操作会使得系统产生新的输出，并被传感器以资源 status 形式上报，控制器的各个组件将都会是独立自主地运行，不断使系统向 spec 表示终态趋近。

   各组件独立自主运行，不断使系统向终态趋近 status -> spec

2. Sensor
   逻辑传感器

- Reflector
- Informer
- Indexer

Reflector 通过 List 和 Watch K8s server 来获取资源的数据。List 用来在 Controller 重启以及 Watch 中断的情况下，进行系统资源的全量更新；而 Watch 则在多次 List 之间进行增量的资源更新；Reflector 在获取新的资源数据后，会在 Delta 队列中塞入一个包括资源对象信息本身以及资源对象事件类型的 Delta 记录，Delta 队列中可以保证同一个对象在队列中仅有一条记录，从而避免 Reflector 重新 List 和 Watch 的时候产生重复的记录。

Informer 组件不断地从 Delta 队列中弹出 delta 记录，然后把资源对象交给 indexer，让 indexer 把资源记录在一个缓存中，缓存在默认设置下是用资源的命名空间来做索引的，并且可以被 Controller Manager 或多个 Controller 所共享。之后，再把这个事件交给事件的回调函数

3. 控制循环例子 - 扩容
   ReplicaSet 是一个用来描述无状态应用的扩缩容行为的资源， ReplicaSet controler 通过监听 ReplicaSet 资源来维持应用希望的状态数量，ReplicaSet 中通过 selector 来匹配所关联的 Pod，在这里考虑 ReplicaSet rsA 的，replicas 从 2 被改到 3 的场景。

首先，Reflector 会 watch 到 ReplicaSet 和 Pod 两种资源的变化，为什么我们还会 watch pod 资源的变化稍后会讲到。发现 ReplicaSet 发生变化后，在 delta 队列中塞入了对象是 rsA，而且类型是更新的记录。

Informer 一方面把新的 ReplicaSet 更新到缓存中，并与 Namespace nsA 作为索引。另外一方面，调用 Update 的回调函数，ReplicaSet 控制器发现 ReplicaSet 发生变化后会把字符串的 nsA/rsA 字符串塞入到工作队列中，工作队列后的一个 Worker 从工作队列中取到了 nsA/rsA 这个字符串的 key，并且从缓存中取到了最新的 ReplicaSet 数据。

Worker 通过比较 ReplicaSet 中 spec 和 status 里的数值，发现需要对这个 ReplicaSet 进行扩容，因此 ReplicaSet 的 Worker 创建了一个 Pod，这个 pod 中的 Ownereference 取向了 ReplicaSet rsA。

然后 Reflector Watch 到的 Pod 新增事件，在 delta 队列中额外加入了 Add 类型的 deta 记录，一方面把新的 Pod 记录通过 Indexer 存储到了缓存中，另一方面调用了 ReplicaSet 控制器的 Add 回调函数，Add 回调函数通过检查 pod ownerReferences 找到了对应的 ReplicaSet，并把包括 ReplicaSet 命名空间和字符串塞入到了工作队列中。

ReplicaSet 的 Woker 在得到新的工作项之后，从缓存中取到了新的 ReplicaSet 记录，并得到了其所有创建的 Pod，因为 ReplicaSet 的状态不是最新的，也就是所有创建 Pod 的数量不是最新的。因此在此时 ReplicaSet 更新 status 使得 spec 和 status 达成一致。

### 控制器模式总结

1. 两种 API 设计方法
   Kubernetes 控制器模式依赖声明式的 API
2. 命令式 API 的问题
   命令 API 最大的一个问题在于错误处理；
   在大规模的分布式系统中，错误是无处不在的。一旦发出的命令没有响应，调用方只能通过反复重试的方式来试图恢复错误，然而盲目的重试可能会带来更大的问题。

假设原来的命令，后台实际上已经执行完成了，重试后又多执行了一个重试的命令操作。为了避免重试的问题，系统往往还需要在执行命令前，先记录一下需要执行的命令，并且在重启等场景下，重做待执行的命令，而且在执行的过程中，还需要考虑多个命令的先后顺序、覆盖关系等等一些复杂的逻辑情况。

实际上许多命令式的交互系统后台往往还会做一个巡检的系统，用来修正命令处理超时、重试等一些场景造成数据不一致的问题；

然而，因为巡检逻辑和日常操作逻辑是不一样的，往往在测试上覆盖不够，在错误处理上不够严谨，具有很大的操作风险，因此往往很多巡检系统都是人工来触发的。

最后，命令式 API 在处理多并发访问时，也很容易出现问题；
假如有多方并发的对一个资源请求进行操作，并且一旦其中有操作出现了错误，就需要重试。那么最后哪一个操作生效了，就很难确认，也无法保证。很多命令式系统往往在操作前会对系统进行加锁，从而保证整个系统最后生效行为的可预见性，但是加锁行为会降低整个系统的操作执行效率。

相对的，声明式 API 系统里天然地记录了系统现在和最终的状态。
不需要额外的操作数据。另外因为状态的幂等性，可以在任意时刻反复操作。在声明式系统运行的方式里，正常的操作实际上就是对资源状态的巡检，不需要额外开发巡检系统，系统的运行逻辑也能够在日常的运行中得到测试和锤炼，因此整个操作的稳定性能够得到保证。

3. 控制器模式总结
   - Kubernetes 所采用的控制器模式，是由声明式 API 驱动的。确切来说，是基于对 Kubernetes 资源对象的修改来驱动的；
   - Kubernetes 资源之后，是关注该资源的控制器。这些控制器将异步的控制系统向设置的终态驱近；这些控制器是自主运行的，使得系统的自动化和无人值守成为可能；
   - 因为 Kubernetes 的控制器和资源都是可以自定义的，因此可以方便的扩展控制器模式。特别是对于有状态应用，我们往往通过自定义资源和控制器的方式，来自动化运维操作。这个也就是后续会介绍的 operator 的场景。

## 应用编排与管理

### Deployment: 管理部署发布的控制器

- deployment 定义一种 Pod 期望数量，controller 会持续维持 Pod 数量为期望的数量，当出现问题时候，controller 会帮我们恢复，新扩出来对应的 Pod，保证可用的 Pod 数量与期望数量一致
- 配置 Pod 发布方式。 controller 会按照用户给定的策略来更新 Pod，更新过程中，也可以设定不可用 Pod 数量在多少范围内。

### 用例解读

#### Deployment Status

- Processing
- Complete
- Failed

### 架构设计

#### 管理模式

- Deployment 只负责管理不同版本的 Replicaset，由 Replicaset 管理 Pod 副本数。
- 每个 ReplicaSet 对应了 Deployment template 的一个版本
- 一个 ReplicaSet 下的 Pod 都是相同的版本

#### Deployment 控制器

所有的控制器都是通过 Informer 中的 Event 做一些 Handler 和 Watch。这个地方 Deployment 控制器，其实是关注 Deployment 和 ReplicaSet 中的 event，收到事件后会加入到队列中。而 Deployment controller 从队列中取出来之后，它的逻辑会判断 Check Paused，这个 Paused 其实是 Deployment 是否需要新的发布，如果 Paused 设置为 true 的话，就表示这个 Deployment 只会做一个数量上的维持，不会做新的发布。

如果 Check paused 为 Yes 也就是 true 的话，那么只会做 Sync replicas。也就是说把 replicas sync 同步到对应的 ReplicaSet 中，最后再 Update Deployment status，那么 controller 这一次的 ReplicaSet 就结束了。

那么如果 paused 为 false 的话，它就会做 Rollout，也就是通过 Create 或者是 Rolling 的方式来做更新，更新的方式其实也是通过 Create/Update/Delete 这种 ReplicaSet 来做实现的。

#### ReplicaSet 控制器

当 Deployment 分配 ReplicaSet 之后，ReplicaSet 控制器本身也是从 Informer 中 watch 一些事件，这些事件包含了 ReplicaSet 和 Pod 的事件。从队列中取出之后，ReplicaSet controller 的逻辑很简单，就只管理副本数。也就是说如果 controller 发现 replicas 比 Pod 数量大的话，就会扩容，而如果发现实际数量超过期望数量的话，就会删除 Pod。

上面 Deployment 控制器的图中可以看到，Deployment 控制器其实做了更复杂的事情，包含了版本管理，而它把每一个版本下的数量维持工作交给 ReplicaSet 来做。

#### 扩/缩容模拟

下面来看一些操作模拟，比如说扩容模拟。这里有一个 Deployment，它的副本数是 2，对应的 ReplicaSet 有 Pod1 和 Pod2。这时如果我们修改 Deployment replicas， controller 就会把 replicas 同步到当前版本的 ReplicaSet 中，这个 ReplicaSet 发现当前有 2 个 Pod，不满足当前期望 3 个，就会创建一个新的 Pod3。

#### 发布模拟

我们再模拟一下发布，发布的情况会稍微复杂一点。这里可以看到 Deployment 当前初始的 template，比如说 template1 这个版本。template1 这个 ReplicaSet 对应的版本下有三个 Pod：Pod1，Pod2，Pod3。

这时修改 template 中一个容器的 image， Deployment controller 就会新建一个对应 template2 的 ReplicaSet。创建出来之后 ReplicaSet 会逐渐修改两个 ReplicaSet 的数量，比如它会逐渐增加 ReplicaSet2 中 replicas 的期望数量，而逐渐减少 ReplicaSet1 中的 Pod 数量。

那么最终达到的效果是：新版本的 Pod 为 Pod4、Pod5 和 Pod6，旧版本的 Pod 已经被删除了，这里就完成了一次发布。

#### 回滚模拟

来看一下回滚模拟，根据上面的发布模拟可以知道 Pod4、Pod5、Pod6 已经发布完成。这时发现当前的业务版本是有问题的，如果做回滚的话，不管是通过 rollout 命令还是通过回滚修改 template，它其实都是把 template 回滚为旧版本的 template1。

这个时候 Deployment 会重新修改 ReplicaSet1 中 Pod 的期望数量，把期望数量修改为 3 个，且会逐渐减少新版本也就是 ReplicaSet2 中的 replica 数量，最终的效果就是把 Pod 从旧版本重新创建出来。

#### spec 字段解析

最后再来简单看一些 Deployment 中的字段解析。首先看一下 Deployment 中其他的 spec 字段：

- MinReadySeconds：Deployment 会根据 Pod ready 来看 Pod 是否可用，但是如果我们设置了 MinReadySeconds 之后，比如设置为 30 秒，那 Deployment 就一定会等到 Pod ready 超过 30 秒之后才认为 Pod 是 available 的。Pod available 的前提条件是 Pod ready，但是 ready 的 Pod 不一定是 available 的，它一定要超过 MinReadySeconds 之后，才会判断为 available；
- revisionHistoryLimit：保留历史 revision，即保留历史 ReplicaSet 的数量，默认值为 10 个。这里可以设置为一个或两个，如果回滚可能性比较大的话，可以设置数量超过 10；
- paused：paused 是标识，Deployment 只做数量维持，不做新的发布，这里在 Debug 场景可能会用到；
- progressDeadlineSeconds：前面提到当 Deployment 处于扩容或者发布状态时，它的 condition 会处于一个 processing 的状态，processing 可以设置一个超时时间。如果超过超时时间还处于 processing，那么 controller 将认为这个 Pod 会进入 failed 的状态。

#### 升级策略字段解析

Deployment 在 RollingUpdate 中主要提供了两个策略，一个是 MaxUnavailable，另一个是 MaxSurge。这两个字段解析的意思，可以看下图中详细的 comment，或者简单解释一下：

- MaxUnavailable：滚动过程中最多有多少个 Pod 不可用；
- MaxSurge：滚动过程中最多存在多少个 Pod 超过预期 replicas 数量。
  上文提到，ReplicaSet 为 3 的 Deployment 在发布的时候可能存在一种情况：新版本的 ReplicaSet 和旧版本的 ReplicaSet 都可能有两个 replicas，加在一起就是 4 个，超过了我们期望的数量三个。这是因为我们默认的 MaxUnavailable 和 MaxSurge 都是 25%，默认 Deployment 在发布的过程中，可能有 25% 的 replica 是不可用的，也可能超过 replica 数量 25% 是可用的，最高可以达到 125% 的 replica 数量。
  这里其实可以根据用户实际场景来做设置。比如当用户的资源足够，且更注重发布过程中的可用性，可设置 MaxUnavailable 较小、MaxSurge 较大。但如果用户的资源比较紧张，可以设置 MaxSurge 较小，甚至设置为 0，这里要注意的是 MaxSurge 和 MaxUnavailable 不能同时为 0。

理由不难理解，当 MaxSurge 为 0 的时候，必须要删除 Pod，才能扩容 Pod；如果不删除 Pod 是不能新扩 Pod 的，因为新扩出来的话，总共的 Pod 数量就会超过期望数量。而两者同时为 0 的话，MaxSurge 保证不能新扩 Pod，而 MaxUnavailable 不能保证 ReplicaSet 中有 Pod 是 available 的，这样就会产生问题。所以说这两个值不能同时为 0。用户可以根据自己的实际场景来设置对应的、合适的值。

### Summary

- Deployment 是 Kubernetes 中常见的一种 Workload，支持部署管理多版本的 Pod；
- Deployment 管理多版本的方式，是针对每个版本的 template 创建一个 ReplicaSet，由 \* ReplicaSet 维护一定数量的 Pod 副本，而 Deployment 只需要关心不同版本的 ReplicaSet 里要指定多少数量的 Pod；
- Deployment 发布部署的根本原理，就是 Deployment 调整不同版本 ReplicaSet 里的终态副本数，以此来达到多版本 Pod 的升级和回滚。

### Job

- kubernetes 的 Job 是一个管理任务的控制器，它可以创建一个或多个 Pod 来指定 Pod 的数量，并可以监控它是否成功地运行或终止；
- 可以根据 Pod 的状态来给 Job 设置重置的方式及重试的次数；
- 还可以根据依赖关系，保证上一个任务运行完成之后再运行下一个任务；
- 同时还可以控制任务的并行度，根据并行度来确保 Pod 运行过程中的并行次数和总体完成大小。

#### 并行运行 Job

有时候有些需求：希望 Job 运行的时候可以最大化的并行，并行出 n 个 Pod 去快速地执行。同时，由于我们的节点数有限制，可能也不希望同时并行的 Pod 数过多，有那么一个管道的概念，我们可以希望最大的并行度是多少，Job 控制器都可以帮我们来做到。

这里主要看两个参数: 一个是 completions，一个是 parallelism。

- completions: 是用来指定本 Pod 队列执行次数。可能这个不是很好理解，其实可以把它认为是这个 Job 指定的可以运行的总次数。比如这里设置成 8，即这个任务一共会被执行 8 次；
- parallelism: 代表这个并行执行的个数。所谓并行执行的次数，其实就是一个管道或者缓冲器中缓冲队列的大小，把它设置成 2，也就是说这个 Job 一定要执行 8 次，每次并行 2 个 Pod，这样的话，一共会执行 4 个批次。

### CronJob

### 架构设计

#### Job 管理模式

1. Job Controller 负责根据配置创建 Pod。
2. Job Controller 跟踪 Job 状态，根据配置及时重试或者继续创建。
3. Job Controller 会自动添加 label 来跟踪对应的 pod，并根据配置并行或者串行创建 pod。

### DaemonSet 守护进程控制器

DaemonSet 也是 Kubernetes 提供的一个 default controller，它实际是做一个守护进程的控制器，它能帮我们做到以下几件事情：

- 首先能保证集群内的每一个节点都运行一组相同的 pod；
- 同时还能根据节点的状态保证新加入的节点自动创建对应的 pod；
- 在移除节点的时候，能删除对应的 pod；
- 而且它会跟踪每个 pod 的状态，当这个 pod 出现异常、Crash 掉了，会及时地去 recovery 这个状态。

#### DaemonSet 语法

适用场景

1. 集群存储进程: glusterd、ceph
2. 日志收集进程: fluentd、logstash
3. 需要在每个节点运行的监控收集器

#### 更新 DaemonSet

DaemonSet 和 deployment 特别像，它也有两种更新策略：一个是 RollingUpdate，另一个是 OnDelete。

- RollingUpdate 其实比较好理解，就是会一个一个的更新。先更新第一个 pod，然后老的 pod 被移除，通过健康检查之后再去见第二个 pod，这样对于业务上来说会比较平滑地升级，不会中断；
- OnDelete 其实也是一个很好的更新策略，就是模板更新之后，pod 不会有任何变化，需要我们手动控制。我们去删除某一个节点对应的 pod，它就会重建，不删除的话它就不会重建，这样的话对于一些我们需要手动控制的特殊需求也会有特别好的作用。

#### DaemonSet 管理模式

1. DaemonSet Controller 负责根据配置创建 Pod
2. DaemonSet Controller 跟踪 Job 状态，根据配置及时重试 Pod 或者继续创建
3. DaemonSet Controller 会自动添加 affinity & label 来跟踪对应的 Pod，并根据配置在每个节点或者适合的部分节点创建 Pod

#### DaemonSet 的控制器

DaemonSet 其实和 Job controller 做的差不多：两者都需要根据 watch 这个 API Server 的状态。现在 DaemonSet 和 Job controller 唯一的不同点在于，DaemonsetSet Controller 需要去 watch node 的状态，但其实这个 node 的状态还是通过 API Server 传递到 ETCD 上。

当有 node 状态节点发生变化时，它会通过一个内存消息队列发进来，然后 DaemonSet controller 会去 watch 这个状态，看一下各个节点上是都有对应的 Pod，如果没有的话就去创建。当然它会去做一个对比，如果有的话，它会比较一下版本，然后加上刚才提到的是否去做 RollingUpdate？如果没有的话就会重新创建，Ondelete 删除 pod 的时候也会去做 check 它做一遍检查，是否去更新，或者去创建对应的 pod。

当然最后的时候，如果全部更新完了之后，它会把整个 DaemonSet 的状态去更新到 API Server 上，完成最后全部的更新。

## 应用配置管理

### Pod 配置管理

- 可变配置: COnfigMap
- 敏感信息: Secret
- 身份认证: ServiceAccount
- 资源配置: Resources
- 安全管控: SecurityContext
- 前置校验: InitContainers

### ConfigMap

管理容器运行所需的配置文件、环境变量、命令行参数等可变配置，解耦容器镜像和可变配置，保证工作负载(Pod)的可移植性

### ConfigMap 注意要点

1. ConfigMap 文件大小: etcd 有写入 1MB 限制
2. Pod 引入 ConfigMap 时必须是相同 namespace 的 ConfigMap
3. Pod 引用的 ConfigMap 如何不存在，pod 会创建失败，pod 需要先创建好要引用的 configmap
4. 使用 envFrom 的方式，把 ConfigMap 里面所有的信息导入成环境变量时，如果 ConfigMap 里有些 key 是无效的，比如 key 的名字里面带有数字，这个环境变量是不会注入容器的，它会被忽略。但是这个 pod 本身是可以创建的。这个和第三点是不一样的方式，是 ConfigMap 文件存在基础上，整体导入成环境变量的一种形式。
5. 什么样的 pod 才能使用 ConfigMap？这里只有通过 K8s api 创建的 pod 才能使用 ConfigMap，比如说通过用命令行 kubectl 来创建的 pod，肯定是可以使用 ConfigMap 的，但其他方式创建的 pod，比如说 kubelet 通过 manifest 创建的 static pod，它是不能使用 ConfigMap 的。

### Secret

Secret 是一个主要用来存储密码 token 等一些敏感信息的资源对象。其中，敏感信息是采用 base-64 编码保存起来的,元数据的话，里面主要是 name、namespace 两个字段；接下来是 type，它是非常重要的一个字段，是指 Secret 的一个类型。Secret 类型种类比较多，下面列了常用的四种类型：

- 第一种是 Opaque，它是普通的 Secret 文件；
- 第二种是 service-account-token，是用于 service-account 身份认证用的 Secret；
- 第三种是 dockerconfigjson，这是拉取私有仓库镜像的用的一种 Secret；
- 第四种是 bootstrap.token，是用于节点接入集群校验用的 Secret。
  Secret 使用注意要点

#### Secret 使用注意点：

1. Secret 文件大小限制和 ConfigMap 一样，也是 1MB
2. Secret 采用了 base-64 编码，但是它跟明文也没有太大区别。所以说，如果有一些机密信息要用 Secret 来存储的话，还是要很慎重考虑。也就是说谁会来访问你这个集群，谁会来用你这个 Secret，还是要慎重考虑，因为它如果能够访问这个集群，就能拿到这个 Secret。如果是对 Secret 敏感信息要求很高，对加密这块有很强的需求，推荐可以使用 Kubernetes 和开源的 vault 做一个解决方案，来解决敏感信息的加密和权限管理。
3. Secret 读取的最佳实践，建议不要用 list/watch，如果用 list/watch 操作的话，会把 namespace 下的所有 Secret 全部拉取下来，这样其实暴露了更多的信息。推荐使用 GET 的方法，这样只获取你自己需要的那个 Secret。

### ServiceAccount

ServiceAccount 首先是用于解决 pod 在集群里面的身份认证问题，身份认证信息是存在于 Secret 里面

#### Pod 服务质量 (QoS) 配置

根据 CPU 对容器内存资源的需求，我们对 pod 的服务质量进行一个分类，分别是 Guaranteed、Burstable 和 BestEffort。

- Guaranteed ：pod 里面每个容器都必须有内存和 CPU 的 request 以及 limit 的一个声明，且 request 和 limit 必须是一样的，这就是 Guaranteed；
- Burstable：Burstable 至少有一个容器存在内存和 CPU 的一个 request；BestEffort：只要不是 Guaranteed 和 Burstable，那就是 BestEffort。
  那么这个服务质量是什么样的呢？资源配置好后，当这个节点上 pod 容器运行，比如说节点上 memory 配额资源不足，kubelet 会把一些低优先级的，或者说服务质量要求不高的（如：BestEffort、Burstable）pod 驱逐掉。它们是按照先去除 BestEffort，再去除 Burstable 的一个顺序来驱逐 pod 的。

### SecurityContext

SecurityContext 主要是用于限制容器的一个行为，它能保证系统和其他容器的安全。这一块的能力不是 Kubernetes 或者容器 runtime 本身的能力，而是 Kubernetes 和 runtime 通过用户的配置，最后下传到内核里，再通过内核的机制让 SecurityContext 来生效。所以这里讲的内容，会比较简单或者说比较抽象一点。

SecurityContext 主要分为三个级别：

第一个是容器级别，仅对容器生效；
第二个是 pod 级别，对 pod 里所有容器生效；
第三个是集群级别，就是 PSP，对集群内所有 pod 生效。
权限和访问控制设置项，现在一共列有七项（这个数量后续可能会变化）：

第一个就是通过用户 ID 和组 ID 来控制文件访问权限；
第二个是 SELinux，它是通过策略配置来控制用户或者进程对文件的访问控制；
第三个是特权容器；
第四个是 Capabilities，它也是给特定进程来配置一个 privileged 能力；
第五个是 AppArmor，它也是通过一些配置文件来控制可执行文件的一个访问控制权限，比如说一些端口的读写；
第六个是一个对系统调用的控制；
第七个是对子进程能否获取比父亲更多的权限的一个限制。

### InitContainer

InitContainer 和普通 Container 区别

- InitContainer 首先会比普通 container 先启动，并且直到所有的 InitContainer 执行成功后，普通 container 才会被启动；
- InitContainer 之间是按定义的次序去启动执行的，执行成功一个之后再执行第二个，而普通的 container 是并发启动的；
- InitContainer 执行成功后就结束退出，而普通容器可能会一直在执行。它可能是一个 longtime 的，或者说失败了会重启，这个也是 InitContainer 和普通 container 不同的地方。
  InitContainer 用途主要为普通 container 服务，比如说它可以为普通 container 启动之前做一个初始化，或者为它准备一些配置文件， 配置文件可能是一些变化的东西。再比如做一些前置条件的校验，如网络是否联通。

### Volumes

Pod Volumes
使用场景：

1. 如果 pod 中的某一个容器在运行时异常退出，被 kubelet 重新拉起之后，如何保证之前容器产生的重要数据没有丢失？
2. 如果同一个 pod 中的多个容器想要共享数据，应该如何去做？
   Pod Volumes 的常见类型：

- 本地存储，常用的有 emptydir/hostpath；
- 网络存储：网络存储当前的实现方式有两种，一种是 in-tree，它的实现的代码是放在 K8s 代码仓库中的，随着 k8s 对存储类型支持的增多，这种方式会给 k8s 本身的维护和发展带来很大的负担；而第二种实现方式是 out-of-tree，它的实现其实是给 K8s 本身解耦的，通过抽象接口将不同存储的 driver 实现从 k8s 代码仓库中剥离，因此 out-of-tree 是后面社区主推的一种实现网络存储插件的方式；
- Projected Volumes：它其实是将一些配置信息，如 secret/configmap 用卷的形式挂载在容器中，让容器中的程序可以通过 POSIX 接口来访问配置数据

#### Persistent Volumes

PV 使用场景

![](/img/pv.png)

pod 中声明的 volume 生命周期与 pod 是相同的，以下有几种常见的场景：

- 场景一：pod 重建销毁，如用 Deployment 管理的 pod，在做镜像升级的过程中，会产生新的 pod 并且删除旧的 pod ，那新旧 pod 之间如何复用数据？
- 场景二：宿主机宕机的时候，要把上面的 pod 迁移，这个时候 StatefulSet 管理的 pod，其实已经实现了带卷迁移的语义。这时通过 Pod Volumes 显然是做不到的；
- 场景三：多个 pod 之间，如果想要共享数据，应该如何去声明呢？我们知道，同一个 pod 中多个容器想共享数据，可以借助 Pod Volumes 来解决；当多个 pod 想共享数据时，Pod Volumes 就很难去表达这种语义；
- 场景四：如果要想对数据卷做一些功能扩展性，如：snapshot、resize 这些功能，又应该如何去做呢？

Persistent Volumes 可以将存储和计算分离，通过不同的组件来管理存储资源和计算资源，然后解耦 pod 和 Volume 之间生命周期的关联。这样，当把 pod 删除之后，它使用的 PV 仍然存在，还可以被新建的 pod 复用。

#### PVC 设计意图

![](/img/pv-design.png)

用户在使用 PV 时其实是通过 PVC，为什么有了 PV 又设计了 PVC 呢？主要原因是为了简化 K8s 用户对存储的使用方式，做到职责分离。通常用户在使用存储的时候，只用声明所需的存储大小以及访问模式。

访问模式是什么？其实就是：我要使用的存储是可以被多个 node 共享还是只能单 node 独占访问(注意是 node level 而不是 pod level)？只读还是读写访问？用户只用关心这些东西，与存储相关的实现细节是不需要关心的。

通过 PVC 和 PV 的概念，将用户需求和实现细节解耦开，用户只用通过 PVC 声明自己的存储需求。PV 是有集群管理员和存储相关团队来统一运维和管控，这样的话，就简化了用户使用存储的方式。可以看到，PV 和 PVC 的设计其实有点像面向对象的接口与实现的关系。用户在使用功能时，只需关心用户接口，不需关心它内部复杂的实现细节。

#### Static Volume Provisioning

![](/img/svp.png)

静态 Provisioning：由集群管理员事先去规划这个集群中的用户会怎样使用存储，它会先预分配一些存储，也就是预先创建一些 PV；然后用户在提交自己的存储需求（也就是 PVC）的时候，K8s 内部相关组件会帮助它把 PVC 和 PV 做绑定；之后用户再通过 pod 去使用存储的时候，就可以通过 PVC 找到相应的 PV，它就可以使用了。

静态产生方式不足

- 需要集群管理员预分配，预分配其实是很难预测用户真实需求的。举一个最简单的例子：如果用户需要的是 20G，然而集群管理员在分配的时候可能有 80G 、100G 的，但没有 20G 的，这样就很难满足用户的真实需求，也会造成资源浪费。

#### Dynamic Volume Provisioning

![](/img/dvp.png)

动态供给: 集群管理员不预分配 PV，他写了一个模板文件，这个模板文件是用来表示创建某一类型存储（块存储，文件存储等）所需的一些参数，这些参数是用户不关心的，给存储本身实现有关的参数。用户只需要提交自身的存储需求，也就是 PVC 文件，并在 PVC 中指定使用的存储模板（StorageClass）。

K8s 集群中的管控组件，会结合 PVC 和 StorageClass 的信息动态，生成用户所需要的存储（PV），将 PVC 和 PV 进行绑定后，pod 就可以使用 PV 了。通过 StorageClass 配置生成存储所需要的存储模板，再结合用户的需求动态创建 PV 对象，做到按需分配，在没有增加用户使用难度的同时也解放了集群管理员的运维工作。

![](/img/pv-usage.png)

首先来看一下 Pod Volumes 的使用。如上图左侧所示，我们可以在 pod yaml 文件中的 Volumes 字段中，声明我们卷的名字以及卷的类型。声明的两个卷，一个是用的是 emptyDir，另外一个用的是 hostPath，这两种都是本地卷。在容器中应该怎么去使用这个卷呢？它其实可以通过 volumeMounts 这个字段，volumeMounts 字段里面指定的 name 其实就是它使用的哪个卷，mountPath 就是容器中的挂载路径。

这里还有个 subPath，subPath 是什么？

先看一下，这两个容器都指定使用了同一个卷，就是这个 cache-volume。那么，在多个容器共享同一个卷的时候，为了隔离数据，我们可以通过 subPath 来完成这个操作。它会在卷里面建立两个子目录，然后容器 1 往 cache 下面写的数据其实都写在子目录 cache1 了，容器 2 往 cache 写的目录，其数据最终会落在这个卷里子目录下面的 cache2 下。

还有一个 readOnly 字段，readOnly 的意思其实就是只读挂载，这个挂载你往挂载点下面实际上是没有办法去写数据的。

另外 emptyDir、hostPath 都是本地存储，它们之间有什么细微的差别呢？emptyDir 其实是在 pod 创建的过程中会临时创建的一个目录，这个目录随着 pod 删除也会被删除，里面的数据会被清空掉；hostPath 顾名思义，其实就是宿主机上的一个路径，在 pod 删除之后，这个目录还是存在的，它的数据也不会被丢失。这就是它们两者之间一个细微的差别。

### 静态 PV 使用

![](/img/svp-usage.png)
![](/img/svp-usage2.png)

#### 动态 PV 使用

![](/img/dvp-usage.png)

#### PV Spec 重要字段解析

![](/img/pvc-spec.png)

Capacity：这个很好理解，就是存储对象的大小；
AccessModes: 也是用户需要关心的，就是说我使用这个 PV 的方式。它有三种使用方式。
一种是单 node 读写访问；
第二种是多个 node 只读访问，是常见的一种数据的共享方式；
第三种是多个 node 上读写访问。
用户在提交 PVC 的时候，最重要的两个字段 —— Capacity 和 AccessModes。在提交 PVC 后，k8s 集群中的相关组件是如何去找到合适的 PV 呢？首先它是通过为 PV 建立的 AccessModes 索引找到所有能够满足用户的 PVC 里面的 AccessModes 要求的 PV list，然后根据 PVC 的 Capacity，StorageClassName, Label Selector 进一步筛选 PV，如果满足条件的 PV 有多个，选择 PV 的 size 最小的，accessmodes 列表最短的 PV，也即最小适合原则。

ReclaimPolicy：这个就是刚才提到的，我的用户方 PV 的 PVC 在删除之后，我的 PV 应该做如何处理？常见的有三种方式。
第一种方式我们就不说了，现在 K8s 中已经不推荐使用了；
第二种方式 delete，也就是说 PVC 被删除之后，PV 也会被删除；
第三种方式 Retain，就是保留，保留之后，后面这个 PV 需要管理员来手动处理。
StorageClassName：StorageClassName 这个我们刚才说了，我们动态 Provisioning 时必须指定的一个字段，就是说我们要指定到底用哪一个模板文件来生成 PV ；
NodeAffinity：就是说我创建出来的 PV，它能被哪些 node 去挂载使用，其实是有限制的。然后通过 NodeAffinity 来声明对 node 的限制，这样其实对 使用该 PV 的 pod 调度也有限制，就是说 pod 必须要调度到这些能访问 PV 的 node 上，才能使用这块 PV，这个字段在我们下一讲讲解存储拓扑调度时在细说。

#### PV 状态流转

![](/img/pv-state.png)

接下来我们看一下 PV 的状态流转。首先在创建 PV 对象后，它会处在短暂的 pending 状态；等真正的 PV 创建好之后，它就处在 available 状态。

available 状态意思就是可以使用的状态，用户在提交 PVC 之后，被 K8s 相关组件做完 bound（即：找到相应的 PV），这个时候 PV 和 PVC 就结合到一起了，此时两者都处在 bound 状态。当用户在使用完 PVC，将其删除后，这个 PV 就处在 released 状态，之后它应该被删除还是被保留呢？这个就会依赖我们刚才说的 ReclaimPolicy。

这里有一个点需要特别说明一下：当 PV 已经处在 released 状态下，它是没有办法直接回到 available 状态，也就是说接下来无法被一个新的 PVC 去做绑定。如果我们想把已经 released 的 PV 复用，我们这个时候通常应该怎么去做呢？ 第一种方式：我们可以新建一个 PV 对象，然后把之前的 released 的 PV 的相关字段的信息填到新的 PV 对象里面，这样的话，这个 PV 就可以结合新的 PVC 了；第二种是我们在删除 pod 之后，不要去删除 PVC 对象，这样给 PV 绑定的 PVC 还是存在的，下次 pod 使用的时候，就可以直接通过 PVC 去复用。K8s 中的 StatefulSet 管理的 Pod 带存储的迁移就是通过这种方式。

### PV 架构设计

![](/img/pv-arch.png)
csi 是什么？csi 的全称是 container storage interface，它是 K8s 社区后面对存储插件实现(out of tree)的官方推荐方式。csi 的实现大体可以分为两部分：

第一部分是由 k8s 社区驱动实现的通用的部分，像我们这张图中的 csi-provisioner 和 csi-attacher controller；
另外一种是由云存储厂商实践的，对接云存储厂商的 OpenApi，主要是实现真正的 create/delete/mount/unmount 存储的相关操作，对应到上图中的 csi-controller-server 和 csi-node-server。
接下来看一下，当用户提交 yaml 之后，k8s 内部的处理流程。用户在提交 PVCyaml 的时候，首先会在集群中生成一个 PVC 对象，然后 PVC 对象会被 csi-provisioner controller watch 到，csi-provisioner 会结合 PVC 对象以及 PVC 对象中声明的 storageClass，通过 GRPC 调用 csi-controller-server，然后，到云存储服务这边去创建真正的存储，并最终创建出来 PV 对象。最后，由集群中的 PV controller 将 PVC 和 PV 对象做 bound 之后，这个 PV 就可以被使用了。

用户在提交 pod 之后，首先会被调度器调度选中某一个合适的 node，之后该 node 上面的 kubelet 在创建 pod 流程中会通过首先 csi-node-server 将我们之前创建的 PV 挂载到我们 pod 可以使用的路径，然后 kubelet 开始 create && start pod 中的所有 container。

#### PV、PVC 以及通过 csi 使用存储流程

![](/img/pv-pvc-process.png)
主要分为三个阶段：

1. (Create 阶段)是用户提交完 PVC，由 csi-provisioner 创建存储，并生成 PV 对象，之后 PV controller 将 PVC 及生成的 PV 对象做 bound，bound 之后，create 阶段就完成了；
2. 之后用户在提交 pod yaml 的时候，首先会被调度选中某一个 合适的 node，等 pod 的运行 node 被选出来之后，会被 AD Controller watch 到 pod 选中的 node，它会去查找 pod 中使用了哪些 PV。然后它会生成一个内部的对象叫 VolumeAttachment 对象，从而去触发 csi-attacher 去调用 csi-controller-server 去做真正的 attache 操作，attach 操作调到云存储厂商 OpenAPI。这个 attach 操作就是将存储 attach 到 pod 将会运行的 node 上面。第二个阶段 —— attach 阶段完成；
   然后我们接下来看第三个阶段。
3. 第三个阶段 发生在 kubelet 创建 pod 的过程中，它在创建 pod 的过程中，首先要去做一个 mount，这里的 mount 操作是为了将已经 attach 到这个 node 上面那块盘，进一步 mount 到 pod 可以使用的一个具体路径，之后 kubelet 才开始创建并启动容器。这就是 PV 加 PVC 创建存储以及使用存储的第三个阶段 —— mount 阶段。

总的来说，有三个阶段：第一个 create 阶段，主要是创建存储；第二个 attach 阶段，就是将那块存储挂载到 node 上面(通常为将存储 load 到 node 的/dev 下面)；第三个 mount 阶段，将对应的存储进一步挂载到 pod 可以使用的路径。这就是我们的 PVC、PV、已经通过 CSI 实现的卷从创建到使用的完整流程。

## 应用存储和持久化数据卷：存储快照与拓扑调度(至天)

### 存储快照产生背景

在使用存储时，为了提高数据操作的容错性，我们通常有需要对线上数据进行 snapshot，以及能快速 restore 的能力。另外，当需要对线上数据进行快速的复制以及迁移等动作，如进行环境的复制、数据开发等功能时，都可以通过存储快照来满足需求，而 K8s 中通过 CSI Snapshotter controller 来实现存储快照的功能。

### 存储快照用户接口-Snapshot

我们知道，K8s 中通过 pvc 以及 pv 的设计体系来简化用户对存储的使用，而存储快照的设计其实是仿照 pvc & pv 体系的设计思想。当用户需要存储快照的功能时，可以通过 VolumeSnapshot 对象来声明，并指定相应的 VolumeSnapshotClass 对象，之后由集群中的相关组件动态生成存储快照以及存储快照对应的对象 VolumeSnapshotContent。如下对比图所示，动态生成 VolumeSnapshotContent 和动态生成 pv 的流程是非常相似的。
![](/img/snapshot.png)

### 存储快照用户接口-Restore

有了存储快照之后，如何将快照数据快速恢复过来呢？如下图所示：
![](/img/pvc-recover.png)
如上所示的流程，可以借助 PVC 对象将其的 dataSource 字段指定为 VolumeSnapshot 对象。这样当 PVC 提交之后，会由集群中的相关组件找到 dataSource 所指向的存储快照数据，然后新创建对应的存储以及 pv 对象，将存储快照数据恢复到新的 pv 中，这样数据就恢复回来了，这就是存储快照的 restore 用法。

### Topolopy-含义

首先了解一下拓扑是什么意思：这里所说的拓扑是 K8s 集群中为管理的 nodes 划分的一种“位置”关系，意思为：可以通过在 node 的 labels 信息里面填写某一个 node 属于某一个拓扑。

常见的有三种，这三种在使用时经常会遇到的：

1. 在使用云存储服务的时候，经常会遇到 region，也就是地区的概念，在 K8s 中常通过 label failure-domain.beta.kubernetes.io/region 来标识。这个是为了标识单个 K8s 集群管理的跨 region 的 nodes 到底属于哪个地区；
2. 比较常用的是可用区，也就是 available zone，在 K8s 中常通过 label failure-domain.beta.kubernetes.io/zone 来标识。这个是为了标识单个 K8s 集群管理的跨 zone 的 nodes 到底属于哪个可用区；
3. 是 hostname，就是单机维度，是拓扑域为 node 范围，在 K8s 中常通过 label kubernetes.io/hostname 来标识，这个在文章的最后讲 local pv 的时候，会再详细描述。
   上面讲到的三个拓扑是比较常用的，而拓扑其实是可以自己定义的。可以定义一个字符串来表示一个拓扑域，这个 key 所对应的值其实就是拓扑域下不同的拓扑位置。

举个例子：可以用 rack 也就是机房中的机架这个纬度来做一个拓扑域。这样就可以将不同机架 (rack) 上面的机器标记为不同的拓扑位置，也就是说可以将不同机架上机器的位置关系通过 rack 这个纬度来标识。属于 rack1 上的机器，node label 中都添加 rack 的标识，它的 value 就标识成 rack1，即 rack=rack1；另外一组机架上的机器可以标识为 rack=rack2，这样就可以通过机架的纬度就来区分来 K8s 中的 node 所处的位置。

### 存储拓扑调度产生背景

上一节课我们说过，K8s 中通过 PV 的 PVC 体系将存储资源和计算资源分开管理了。如果创建出来的 PV 有"访问位置"的限制，也就是说，它通过 nodeAffinity 来指定哪些 node 可以访问这个 PV。为什么会有这个访问位置的限制？

因为在 K8s 中创建 pod 的流程和创建 PV 的流程，其实可以认为是并行进行的，这样的话，就没有办法来保证 pod 最终运行的 node 是能访问到 有位置限制的 PV 对应的存储，最终导致 pod 没法正常运行。这里来举两个经典的例子：

首先来看一下 Local PV 的例子，Local PV 是将一个 node 上的本地存储封装为 PV，通过使用 PV 的方式来访问本地存储。为什么会有 Local PV 的需求呢？简单来说，刚开始使用 PV 或 PVC 体系的时候，主要是用来针对分布式存储的，分布式存储依赖于网络，如果某些业务对 I/O 的性能要求非常高，可能通过网络访问分布式存储没办法满足它的性能需求。这个时候需要使用本地存储，刨除了网络的 overhead，性能往往会比较高。但是用本地存储也是有坏处的！分布式存储可以通过多副本来保证高可用，但本地存储就需要业务自己用类似 Raft 协议来实现多副本高可用。

接下来看一下 Local PV 场景可能如果没有对 PV 做“访问位置”的限制会遇到什么问题？
![](/img/localpv.png)
当用户在提交完 PVC 的时候，K8s PV controller 可能绑定的是 node2 上面的 PV。但是，真正使用这个 PV 的 pod，在被调度的时候，有可能调度在 node1 上，最终导致这个 pod 在起来的时候没办法去使用这块存储，因为 pod 真实情况是要使用 node2 上面的存储。

第二个(如果不对 PV 做“访问位置”的限制会出问题的)场景：
![](/img/localpv2.png)
如果搭建的 K8s 集群管理的 nodes 分布在单个区域多个可用区内。在创建动态存储的时候，创建出来的存储属于可用区 2，但之后在提交使用该存储的 pod，它可能会被调度到可用区 1 了，那就可能没办法使用这块存储。因此像阿里云的云盘，也就是块存储，当前不能跨可用区使用，如果创建的存储其实属于可用区 2，但是 pod 运行在可用区 1，就没办法使用这块存储，这是第二个常见的问题场景。

### 存储拓扑调度

首先总结一下之前的两个问题，它们都是 PV 在给 PVC 绑定或者动态生成 PV 的时候，我并不知道后面将使用它的 pod 将调度在哪些 node 上。但 PV 本身的使用，是对 pod 所在的 node 有拓扑位置的限制的，如 Local PV 场景是我要调度在指定的 node 上我才能使用那块 PV，而对第二个问题场景就是说跨可用区的话，必须要在将使用该 PV 的 pod 调度到同一个可用区的 node 上才能使用阿里云云盘服务，那 K8s 中怎样去解决这个问题呢？

简单来说，在 K8s 中将 PV 和 PVC 的 binding 操作和动态创建 PV 的操作做了 delay，delay 到 pod 调度结果出来之后，再去做这两个操作。这样的话有什么好处？

首先，如果要是所要使用的 PV 是预分配的，如 Local PV，其实使用这块 PV 的 pod 它对应的 PVC 其实还没有做绑定，就可以通过调度器在调度的过程中，结合 pod 的计算资源需求(如 cpu/mem) 以及 pod 的 PVC 需求，选择的 node 既要满足计算资源的需求又要 pod 使用的 pvc 要能 binding 的 pv 的 nodeaffinity 限制;
其次对动态生成 PV 的场景其实就相当于是如果知道 pod 运行的 node 之后，就可以根据 node 上记录的拓扑信息来动态的创建这个 PV，也就是保证新创建出来的 PV 的拓扑位置与运行的 node 所在的拓扑位置是一致的，如上面所述的阿里云云盘的例子，既然知道 pod 要运行到可用区 1，那之后创建存储时指定在可用区 1 创建即可。
为了实现上面所说的延迟绑定和延迟创建 PV，需要在 K8s 中的改动涉及到的相关组件有三个：

PV Controller 也就是 persistent volume controller，它需要支持延迟 Binding 这个操作。
另一个是动态生成 PV 的组件，如果 pod 调度结果出来之后，它要根据 pod 的拓扑信息来去动态的创建 PV。
第三组件，也是最重要的一个改动点就是 kube-scheduler。在为 pod 选择 node 节点的时候，它不仅要考虑 pod 对 CPU/MEM 的计算资源的需求，它还要考虑这个 pod 对存储的需求，也就是根据它的 PVC，它要先去看一下当前要选择的 node，能否满足能和这个 PVC 能匹配的 PV 的 nodeAffinity；或者是动态生成 PV 的过程，它要根据 StorageClass 中指定的拓扑限制来 check 当前的 node 是不是满足这个拓扑限制，这样就能保证调度器最终选择出来的 node 就能满足存储本身对拓扑的限制。

#### Volume Snapshot/Restore 示例

![](/img/volume-snapshot.png)
下面来看一下存储快照如何使用：首先需要集群管理员，在集群中创建 VolumeSnapshotClass 对象，VolumeSnapshotClass 中一个重要字段就是 Snapshot，它是指定真正创建存储快照所使用的卷插件，这个卷插件是需要提前部署的，稍后再说这个卷插件。

接下来用户他如果要做真正的存储快照，需要声明一个 VolumeSnapshotClass，VolumeSnapshotClass 首先它要指定的是 VolumeSnapshotClassName，接着它要指定的一个非常重要的字段就是 source，这个 source 其实就是指定快照的数据源是啥。这个地方指定 name 为 disk-pvc，也就是说通过这个 pvc 对象来创建存储快照。提交这个 VolumeSnapshot 对象之后，集群中的相关组件它会找到这个 PVC 对应的 PV 存储，对这个 PV 存储做一次快照。

有了存储快照之后，那接下来怎么去用存储快照恢复数据呢？这个其实也很简单，通过声明一个新的 PVC 对象并在它的 spec 下面的 DataSource 中来声明我的数据源来自于哪个 VolumeSnapshot，这里指定的是 disk-snapshot 对象，当我这个 PVC 提交之后，集群中的相关组件，它会动态生成新的 PV 存储，这个新的 PV 存储中的数据就来源于这个 Snapshot 之前做的存储快照。

#### 限制 Dynamic Provisioning PV 拓扑示例

![](/img/limit-dpv.png)
动态就是指动态创建 PV 就有拓扑位置的限制，那怎么去指定？

首先在 storageclass 还是需要指定 BindingMode，就是 WaitForFirstConsumer，就是延迟绑定。

其次特别重要的一个字段就是 allowedTopologies，限制就在这个地方。上图中可以看到拓扑限制是可用区的级别，这里其实有两层意思：

第一层意思就是说我在动态创建 PV 的时候，创建出来的 PV 必须是在这个可用区可以访问的;
第二层含义是因为声明的是延迟绑定，那调度器在发现使用它的 PVC 正好对应的是该 storageclass 的时候，调度 pod 就要选择位于该可用区的 nodes。
总之，就是要从两方面保证，一是动态创建出来的存储时要能被这个可用区访问的，二是我调度器在选择 node 的时候，要落在这个可用区内的，这样的话就保证我的存储和我要使用存储的这个 pod 它所对应的 node，它们之间的拓扑域是在同一个拓扑域，用户在写 PVC 文件的时候，写法是跟以前的写法是一样的，主要是在 storageclass 中要做一些拓扑限制。

#### 处理流程

kubernetes 对 Volume Snapshot/Restore 处理流程
![](/img/process-pv.png)
首先来看一下存储快照的处理流程，这里来首先解释一下 csi 部分。K8s 中对存储的扩展功能都是推荐通过 csi out-of-tree 的方式来实现的。

csi 实现存储扩展主要包含两部分：

第一部分是由 K8s 社区推动实现的 csi controller 部分，也就是这里的 csi-snapshottor controller 以及 csi-provisioner controller，这些主要是通用的 controller 部分;
另外一部分是由特定的云存储厂商用自身 OpenAPI 实现的不同的 csi-plugin 部分，也叫存储的 driver 部分。
两部分部件通过 unix domain socket 通信连接到一起。有这两部分，才能形成一个真正的存储扩展功能。

如上图所示，当用户提交 VolumeSnapshot 对象之后，会被 csi-snapshottor controller watch 到。之后它会通过 GPPC 调用到 csi-plugin，csi-plugin 通过 OpenAPI 来真正实现存储快照的动作，等存储快照已经生成之后，会返回到 csi-snapshottor controller 中，csi-snapshottor controller 会将存储快照生成的相关信息放到 VolumeSnapshotContent 对象中并将用户提交的 VolumeSnapshot 做 bound。这个 bound 其实就有点类似 PV 和 PVC 的 bound 一样。

有了存储快照，如何去使用存储快照恢复之前的数据呢？前面也说过，通过声明一个新的 PVC 对象，并且指定他的 dataSource 为 Snapshot 对象，当提交 PVC 的时候会被 csi-provisioner watch 到，之后会通过 GRPC 去创建存储。这里创建存储跟之前讲解的 csi-provisioner 有一个不太一样的地方，就是它里面还指定了 Snapshot 的 ID，当去云厂商创建存储时，需要多做一步操作，即将之前的快照数据恢复到新创建的存储中。之后流程返回到 csi-provisioner，它会将新创建的存储的相关信息写到一个新的 PV 对象中，新的 PV 对象被 PV controller watch 到它会将用户提交的 PVC 与 PV 做一个 bound，之后 pod 就可以通过 PVC 来使用 Restore 出来的数据了。这是 K8s 中对存储快照的处理流程。

#### kubernetes 对 Volume Topology-aware Scheduling 处理流程

存储拓扑调度处理流程：
![](/img/dpv-process.png)
第一个步骤其实就是要去声明延迟绑定，这个通过 StorageClass 来做的，上面已经阐述过，这里就不做详细描述了。

接下来看一下调度器，上图中红色部分就是调度器新加的存储拓扑调度逻辑，我们先来看一下不加红色部分时调度器的为一个 pod 选择 node 时，它的大概流程：

首先用户提交完 pod 之后，会被调度器 watch 到，它就会去首先做预选，预选就是说它会将集群中的所有 node 都来与这个 pod 它需要的资源做匹配；
如果匹配上，就相当于这个 node 可以使用，当然可能不止一个 node 可以使用，最终会选出来一批 node；
然后再经过第二个阶段优选，优选就相当于我要对这些 node 做一个打分的过程，通过打分找到最匹配的一个 node；
之后调度器将调度结果写到 pod 里面的 spec.nodeName 字段里面，然后会被相应的 node 上面的 kubelet watch 到，最后就开始创建 pod 的整个流程。
那现在看一下加上卷相关的调度的时候，筛选 node(第二个步骤)又是怎么做的？

先就要找到 pod 中使用的所有 PVC，找到已经 bound 的 PVC，以及需要延迟绑定的这些 PVC；
对于已经 bound 的 PVC，要 check 一下它对应的 PV 里面的 nodeAffinity 与当前 node 的拓扑是否匹配 。如果不匹配， 就说明这个 node 不能被调度。如果匹配，继续往下走，就要去看一下需要延迟绑定的 PVC；
对于需要延迟绑定的 PVC。先去获取集群中存量的 PV，满足 PVC 需求的，先把它全部捞出来，然后再将它们一一与当前的 node labels 上的拓扑做匹配，如果它们(存量的 PV)都不匹配，那就说明当前的存量的 PV 不能满足需求，就要进一步去看一下如果要动态创建 PV 当前 node 是否满足拓扑限制，也就是还要进一步去 check StorageClass 中的拓扑限制，如果 StorageClass 中声明的拓扑限制与当前的 node 上面已经有的 labels 里面的拓扑是相匹配的，那其实这个 node 就可以使用，如果不匹配，说明该 node 就不能被调度。
经过这上面步骤之后，就找到了所有即满足 pod 计算资源需求又满足 pod 存储资源需求的所有 nodes。

当 node 选出来之后，第三个步骤就是调度器内部做的一个优化。这里简单过一下，就是更新经过预选和优选之后，pod 的 node 信息，以及 PV 和 PVC 在 scheduler 中做的一些 cache 信息。

第四个步骤也是重要的一步，已经选择出来 node 的 Pod，不管其使用的 PVC 是要 binding 已经存在的 PV，还是要做动态创建 PV，这时就可以开始做。由调度器来触发，调度器它就会去更新 PVC 对象和 PV 对象里面的相关信息，然后去触发 PV controller 去做 binding 操作，或者是由 csi-provisioner 去做动态创建流程。

## 可观测性: 你的应用健康吗?

当把应用迁移到 Kubernetes 之后，要如何去保障应用的健康与稳定?
首先是提高应用的可观测性；
第二是提高应用的可恢复能力。
从可观测性上来讲，可以在三个方面来去做增强：
首先是应用的健康状态上面，可以实时地进行观测；
第二个是可以获取应用的资源使用情况；
第三个是可以拿到应用的实时日志，进行问题的诊断与分析。

当出现了问题之后，首先要做的事情是要降低影响的范围，进行问题的调试与诊断。最后当出现问题的时候，理想的状况是：可以通过和 K8s 集成的自愈机制进行完整的恢复。

### Liveness 与 Readiness

Liveness probe 也叫就绪指针，用来判断一个 pod 是否处在就绪状态。当一个 pod 处在就绪状态的时候，它才能够对外提供相应的服务，也就是说接入层的流量才能打到相应的 pod。当这个 pod 不处在就绪状态的时候，接入层会把相应的流量从这个 pod 上面进行摘除。

来看一下简单的一个例子：

如下图其实就是一个 Readiness 就绪的一个例子：
![](/img/readiness.png)

当这个 pod 指针判断一直处在失败状态的时候，其实接入层的流量不会打到现在这个 pod 上。
![](/img/readiness2.png)

当这个 pod 的状态从 FAIL 的状态转换成 success 的状态时，它才能够真实地承载这个流量。 Liveness 指针也是类似的，它是存活指针，用来判断一个 pod 是否处在存活状态。当一个 pod 处在不存活状态的时候，会出现什么事情呢？

![](/img/readiness3.png)

这个时候会由上层的判断机制来判断这个 pod 是否需要被重新拉起。那如果上层配置的重启策略是 restart always 的话，那么此时这个 pod 会直接被重新拉起。
应用健康状态-使用方式
接下来看一下 Liveness 指针和 Readiness 指针的具体的用法。

探测方式
Liveness 指针和 Readiness 指针支持三种不同的探测方式：

第一种是 httpGet。它是通过发送 http Get 请求来进行判断的，当返回码是 200-399 之间的状态码时，标识这个应用是健康的；
第二种探测方式是 Exec。它是通过执行容器中的一个命令来判断当前的服务是否是正常的，当命令行的返回结果是 0，则标识容器是健康的；
第三种探测方式是 tcpSocket。它是通过探测容器的 IP 和 Port 进行 TCP 健康检查，如果这个 TCP 的链接能够正常被建立，那么标识当前这个容器是健康的。
探测结果
从探测结果来讲主要分为三种：

第一种是 success，当状态是 success 的时候，表示 container 通过了健康检查，也就是 Liveness probe 或 Readiness probe 是正常的一个状态；
第二种是 Failure，Failure 表示的是这个 container 没有通过健康检查，如果没有通过健康检查的话，那么此时就会进行相应的一个处理，那在 Readiness 处理的一个方式就是通过 service。service 层将没有通过 Readiness 的 pod 进行摘除，而 Liveness 就是将这个 pod 进行重新拉起，或者是删除。
第三种状态是 Unknown，Unknown 是表示说当前的执行的机制没有进行完整的一个执行，可能是因为类似像超时或者像一些脚本没有及时返回，那么此时 Readiness-probe 或 Liveness-probe 会不做任何的一个操作，会等待下一次的机制来进行检验。
那在 kubelet 里面有一个叫 ProbeManager 的组件，这个组件里面会包含 Liveness-probe 或 Readiness-probe，这两个 probe 会将相应的 Liveness 诊断和 Readiness 诊断作用在 pod 之上，来实现一个具体的判断。

#### 应用健康状态-Pod Probe Spec

下面介绍这三种方式不同的检测方式的一个 yaml 文件的使用。

首先先看一下 exec，exec 的使用其实非常简单。如下图所示，大家可以看到这是一个 Liveness probe，它里面配置了一个 exec 的一个诊断。接下来，它又配置了一个 command 的字段，这个 command 字段里面通过 cat 一个具体的文件来判断当前 Liveness probe 的状态，当这个文件里面返回的结果是 0 时，或者说这个命令返回是 0 时，它会认为此时这个 pod 是处在健康的一个状态。

![](/img/exec.png)
那再来看一下这个 httpGet，httpGet 里面有一个字段是路径，第二个字段是 port，第三个是 headers。这个地方有时需要通过类似像 header 头的一个机制做 health 的一个判断时，需要配置这个 header，通常情况下，可能只需要通过 health 和 port 的方式就可以了。

![](/img/httpget.png)
第三种是 tcpSocket，tcpSocket 的使用方式其实也比较简单，你只需要设置一个检测的端口，像这个例子里面使用的是 8080 端口，当这个 8080 端口 tcp connect 审核正常被建立的时候，那 tecSocket，Probe 会认为是健康的一个状态。
![](/img/tcpsocket.png)
此外还有如下的五个参数，是 Global 的参数。

第一个参数叫 initialDelaySeconds，它表示的是说这个 pod 启动延迟多久进行一次检查，比如说现在有一个 Java 的应用，它启动的时间可能会比较长，因为涉及到 jvm 的启动，包括 Java 自身 jar 的加载。所以前期，可能有一段时间是没有办法被检测的，而这个时间又是可预期的，那这时可能要设置一下 initialDelaySeconds；
第二个是 periodSeconds，它表示的是检测的时间间隔，正常默认的这个值是 10 秒；
第三个字段是 timeoutSeconds，它表示的是检测的超时时间，当超时时间之内没有检测成功，那它会认为是失败的一个状态；
第四个是 successThreshold，它表示的是：当这个 pod 从探测失败到再一次判断探测成功，所需要的阈值次数，默认情况下是 1 次，表示原本是失败的，那接下来探测这一次成功了，就会认为这个 pod 是处在一个探针状态正常的一个状态；
最后一个参数是 failureThreshold，它表示的是探测失败的重试次数，默认值是 3，表示的是当从一个健康的状态连续探测 3 次失败，那此时会判断当前这个 pod 的状态处在一个失败的状态。

#### 应用健康状态-Liveness 与 Readiness 总结

接下来对 Liveness 指针和 Readiness 指针进行一个简单的总结
![](/img/app-health-summary.png)

#### 介绍

Liveness 指针是存活指针，它用来判断容器是否存活、判断 pod 是否 running。如果 Liveness 指针判断容器不健康，此时会通过 kubelet 杀掉相应的 pod，并根据重启策略来判断是否重启这个容器。如果默认不配置 Liveness 指针，则默认情况下认为它这个探测默认返回是成功的。

Readiness 指针用来判断这个容器是否启动完成，即 pod 的 condition 是否 ready。如果探测的一个结果是不成功，那么此时它会从 pod 上 Endpoint 上移除，也就是说从接入层上面会把前一个 pod 进行摘除，直到下一次判断成功，这个 pod 才会再次挂到相应的 endpoint 之上。

#### 检测失败

对于检测失败上面来讲 Liveness 指针是直接杀掉这个 pod，而 Readiness 指针是切掉 endpoint 到这个 pod 之间的关联关系，也就是说它把这个流量从这个 pod 上面进行切掉。

#### 适用场景

Liveness 指针适用场景是支持那些可以重新拉起的应用，而 Readiness 指针主要应对的是启动之后无法立即对外提供服务的这些应用。

#### 注意事项

在使用 Liveness 指针和 Readiness 指针的时候有一些注意事项。因为不论是 Liveness 指针还是 Readiness 指针都需要配置合适的探测方式，以免被误操作。

第一个是调大超时的阈值，因为在容器里面执行一个 shell 脚本，它的执行时长是非常长的，平时在一台 ecs 或者在一台 vm 上执行，可能 3 秒钟返回的一个脚本在容器里面需要 30 秒钟。所以这个时间是需要在容器里面事先进行一个判断的，那如果可以调大超时阈值的方式，来防止由于容器压力比较大的时候出现偶发的超时；
第二个是调整判断的一个次数，3 次的默认值其实在比较短周期的判断周期之下，不一定是最佳实践，适当调整一下判断的次数也是一个比较好的方式；
第三个是 exec，如果是使用 shell 脚本的这个判断，调用时间会比较长，比较建议大家可以使用类似像一些编译性的脚本 Golang 或者一些 C 语言、C++ 编译出来的这个二进制的 binary 进行判断，那这种通常会比 shell 脚本的执行效率高 30% 到 50%；
第四个是如果使用 tcpSocket 方式进行判断的时候，如果遇到了 TLS 的服务，那可能会造成后边 TLS 里面有很多这种未健全的 tcp connection，那这个时候需要自己对业务场景上来判断，这种的链接是否会对业务造成影响。

#### 问题诊断

接下来给大家讲解一下在 K8s 中常见的问题诊断。

#### 应用故障排查-了解状态机制

首先要了解一下 K8s 中的一个设计理念，就是这个状态机制。因为 K8s 是整个的一个设计是面向状态机的，它里面通过 yaml 的方式来定义的是一个期望到达的一个状态，而真正这个 yaml 在执行过程中会由各种各样的 controller 来负责整体的状态之间的一个转换。
![](/img/app-health-status.png)
![](/img/app-status-desc.png)
比如说上面的图，实际上是一个 Pod 的一个生命周期。刚开始它处在一个 pending 的状态，那接下来可能会转换到类似像 running，也可能转换到 Unknown，甚至可以转换到 failed。然后，当 running 执行了一段时间之后，它可以转换到类似像 successded 或者是 failed，然后当出现在 unknown 这个状态时，可能由于一些状态的恢复，它会重新恢复到 running 或者 successded 或者是 failed。

其实 K8s 整体的一个状态就是基于这种类似像状态机的一个机制进行转换的，而不同状态之间的转化都会在相应的 K8s 对象上面留下来类似像 Status 或者像 Conditions 的一些字段来进行表示。

像下面这张图其实表示的就是说在一个 Pod 上面一些状态位的一些展现。

![](/img/pod-status.png)

比如说在 Pod 上面有一个字段叫 Status，这个 Status 表示的是 Pod 的一个聚合状态，在这个里面，这个聚合状态处在一个 pending 状态。

然后再往下看，因为一个 pod 里面有多个 container，每个 container 上面又会有一个字段叫 State，然后 State 的状态表示当前这个 container 的一个聚合状态。那在这个例子里面，这个聚合状态处在的是 waiting 的状态，那具体的原因是因为什么呢？是因为它的镜像没有拉下来，所以处在 waiting 的状态，是在等待这个镜像拉取。然后这个 ready 的部分呢，目前是 false，因为它这个进行目前没有拉取下来，所以这个 pod 不能够正常对外服务，所以此时 ready 的状态是未知的，定义为 false。如果上层的 endpoint 发现底层这个 ready 不是 true 的话，那么此时这个服务是没有办法对外服务的。

再往下是 condition，condition 这个机制表示是说：在 K8s 里面有很多这种比较小的这个状态，而这个状态之间的聚合会变成上层的这个 Status。那在这个例子里面有几个状态，第一个是 Initialized，表示是不是已经初始化完成？那在这个例子里面已经是初始化完成的，那它走的是第二个阶段，是在这个 ready 的状态。因为上面几个 container 没有拉取下来相应的镜像，所以 ready 的状态是 false。

然后再往下可以看到这个 container 是否 ready，这里可以看到是 false，而这个状态是 PodScheduled，表示说当前这个 pod 是否是处在一个已经被调度的状态，它已经 bound 在现在这个 node 之上了，所以这个状态也是 true。

那可以通过相应的 condition 是 true 还是 false 来判断整体上方的这个状态是否是正常的一个状态。而在 K8s 里面不同的状态之间的这个转换都会发生相应的事件，而事件分为两种： 一种叫做 normal 的事件，一种是 warning 事件。大家可以看见在这第一条的事件是有个 normal 事件，然后它相应的 reason 是 scheduler，表示说这个 pod 已经被默认的调度器调度到相应的一个节点之上，然后这个节点是 cn-beijing192.168.3.167 这个节点之上。

再接下来，又是一个 normal 的事件，表示说当前的这个镜像在 pull 相应的这个 image。然后再往下是一个 warning 事件，这个 warning 事件表示说 pull 这个镜像失败了。

以此类推，这个地方表示的一个状态就是说在 K8s 里面这个状态机制之间这个状态转换会产生相应的事件，而这个事件又通过类似像 normal 或者是 warning 的方式进行暴露。开发者可以通过类似像通过这个事件的机制，可以通过上层 condition Status 相应的一系列的这个字段来判断当前这个应用的具体的状态以及进行一系列的诊断。

#### 应用故障排查-常见应用异常

Pod 停留在 Pending
第一个就是 pending 状态，pending 表示调度器没有进行介入。此时可以通过 kubectl describe pod 来查看相应的事件，如果由于资源或者说端口占用，或者是由于 node selector 造成 pod 无法调度的时候，可以在相应的事件里面看到相应的结果，这个结果里面会表示说有多少个不满足的 node，有多少是因为 CPU 不满足，有多少是由于 node 不满足，有多少是由于 tag 打标造成的不满足。

Pod 停留在 waiting
那第二个状态就是 pod 可能会停留在 waiting 的状态，pod 的 states 处在 waiting 的时候，通常表示说这个 pod 的镜像没有正常拉取，原因可能是由于这个镜像是私有镜像，但是没有配置 Pod secret；那第二种是说可能由于这个镜像地址是不存在的，造成这个镜像拉取不下来；还有一个是说这个镜像可能是一个公网的镜像，造成镜像的拉取失败。

Pod 不断被拉取并且可以看到 crashing
第三种是 pod 不断被拉起，而且可以看到类似像 backoff。这个通常表示说 pod 已经被调度完成了，但是启动失败，那这个时候通常要关注的应该是这个应用自身的一个状态，并不是说配置是否正确、权限是否正确，此时需要查看的应该是 pod 的具体日志。

Pod 处在 Runing 但是没有正常工作
第四种 pod 处在 running 状态，但是没有正常对外服务。那此时比较常见的一个点就可能是由于一些非常细碎的配置，类似像有一些字段可能拼写错误，造成了 yaml 下发下去了，但是有一段没有正常地生效，从而使得这个 pod 处在 running 的状态没有对外服务，那此时可以通过 apply-validate-f pod.yaml 的方式来进行判断当前 yaml 是否是正常的，如果 yaml 没有问题，那么接下来可能要诊断配置的端口是否是正常的，以及 Liveness 或 Readiness 是否已经配置正确。

Service 无法正常的工作
最后一种就是 service 无法正常工作的时候，该怎么去判断呢？那比较常见的 service 出现问题的时候，是自己的使用上面出现了问题。因为 service 和底层的 pod 之间的关联关系是通过 selector 的方式来匹配的，也就是说 pod 上面配置了一些 label，然后 service 通过 match label 的方式和这个 pod 进行相互关联。如果这个 label 配置的有问题，可能会造成这个 service 无法找到后面的 endpoint，从而造成相应的 service 没有办法对外提供服务，那如果 service 出现异常的时候，第一个要看的是这个 service 后面是不是有一个真正的 endpoint，其次来看这个 endpoint 是否可以对外提供正常的服务。

#### 应用远程调试

本节讲解的是在 K8s 里面如何进行应用的远程调试，远程调试主要分为 pod 的远程调试以及 service 的远程调试。还有就是针对一些性能优化的远程调试。

应用远程调试 - Pod 远程调试
首先把一个应用部署到集群里面的时候，发现问题的时候，需要进行快速验证，或者说修改的时候，可能需要类似像登陆进这个容器来进行一些诊断。
![](/img/pod-remote-debug.png)

比如说可以通过 exec 的方式进入一个 pod。像这条命令里面，通过 kubectl exec-it pod-name 后面再填写一个相应的命令，比如说 /bin/bash，表示希望到这个 pod 里面进入一个交互式的一个 bash。然后在 bash 里面可以做一些相应的命令，比如说修改一些配置，通过 supervisor 去重新拉起这个应用，都是可以的。

那如果指定这一个 pod 里面可能包含着多个 container，这个时候该怎么办呢？怎么通过 pod 来指定 container 呢？其实这个时候有一个参数叫做 -c，如上图下方的命令所示。-c 后面是一个 container-name，可以通过 pod 在指定 -c 到这个 container-name，具体指定要进入哪个 container，后面再跟上相应的具体的命令，通过这种方式来实现一个多容器的命令的一个进入，从而实现多容器的一个远程调试。

应用远程调试 - Servic 远程调试
那么 service 的远程调试该怎么做呢？service 的远程调试其实分为两个部分：

第一个部分是说我想将一个服务暴露到远程的一个集群之内，让远程集群内的一些应用来去调用本地的一个服务，这是一条反向的一个链路；
还有一种方式是我想让这个本地服务能够去调远程的服务，那么这是一条正向的链路。
在反向列入上面有这样一个开源组件，叫做 Telepresence，它可以将本地的应用代理到远程集群中的一个 service 上面，使用它的方式非常简单。

![](/img/pod-debug2.png)
首先先将 Telepresence 的一个 Proxy 应用部署到远程的 K8s 集群里面。然后将远程单一个 deployment swap 到本地的一个 application，使用的命令就是 Telepresence-swap-deployment 然后以及远程的 DEPLOYMENT_NAME。通过这种方式就可以将本地一个 application 代理到远程的 service 之上、可以将应用在远程集群里面进行本地调试，这个有兴趣的同学可以到 GitHub 上面来看一下这个插件的使用的方式。

第二个是如果本地应用需要调用远程集群的服务时候，可以通过 port-forward 的方式将远程的应用调用到本地的端口之上。比如说现在远程的里面有一个 API server，这个 API server 提供了一些端口，本地在调试 Code 时候，想要直接调用这个 API server，那么这时，比较简单的一个方式就是通过 port-forward 的方式。
![](/img/kubectl-port-forward.png)

它的使用方式是 kubectl port-forward，然后 service 加上远程的 service name，再加上相应的 namespace，后面还可以加上一些额外的参数，比如说端口的一个映射，通过这种机制就可以把远程的一个应用代理到本地的端口之上，此时通过访问本地端口就可以访问远程的服务。

开源的调试工具 - kubectl-debug
最后再给大家介绍一个开源的调试工具，它也是 kubectl 的一个插件，叫 kubectl-debug。我们知道在 K8s 里面，底层的容器 runtime 比较常见的就是类似像 docker 或者是 containerd，不论是 docker 还是 containerd，它们使用的一个机制都是基于 Linux namespace 的一个方式进行虚拟化和隔离的。

通常情况下 ，并不会在镜像里面带特别多的调试工具，类似像 netstat telnet 等等这些 ，因为这个会造成应用整体非常冗余。那么如果想要调试的时候该怎么做呢？其实这个时候就可以依赖类似于像 kubectl-debug 这样一个工具。

kubectl-debug 这个工具是依赖于 Linux namespace 的方式来去做的，它可以 datash 一个 Linux namespace 到一个额外的 container，然后在这个 container 里面执行任何的 debug 动作，其实和直接去 debug 这个 Linux namespace 是一致的。这里有一个简单的操作，给大家来介绍一下：

这个地方其实已经安装好了 kubectl-debug，它是 kubectl 的一个插件。所以这个时候，你可以直接通过 kubectl-debug 这条命令来去诊断远程的一个 pod。像这个例子里面，当执行 debug 的时候，实际上它首先会先拉取一些镜像，这个镜像里面实际上会默认带一些诊断的工具。当这个镜像启用的时候，它会把这个 debug container 进行启动。与此同时会把这个 container 和相应的你要诊断的这个 container 的 namespace 进行挂靠，也就说此时这个 container 和你是同 namespace 的，类似像网络站，或者是类似像内核的一些参数，其实都可以在这个 debug container 里面实时地进行查看。

![](/img/pod-debug-3.png)
像这个例子里面，去查看类似像 hostname、进程、netstat 等等，这些其实都是和这个需要 debug 的 pod 是在同一个环境里面的，所以你之前这三条命令可以看到里面相关的信息。
![](/img/pod-debug-4.png)
如果此时进行 logout 的话，相当于会把相应的这个 debug pod 杀掉，然后进行退出，此时对应用实际上是没有任何的影响的。那么通过这种方式可以不介入到容器里面，就可以实现相应的一个诊断。
![](/img/pod-debug5.png)
此外它还支持额外的一些机制，比如说我给设定一些 image，然后类似像这里面安装了的是 htop，然后开发者可以通过这个机制来定义自己需要的这个命令行的工具，并且通过这种 image 的方式设置进来。那么这个时候就可以通过这种机制来调试远程的一个 pod。

## 可观测性 - 监控与日志

### 监控
### 监控类型
#### 资源监控
比较常见的像 CPU、内存、网络这种资源类的一个指标，通常这些指标会以数值、百分比的单位进行统计，是最常见的一个监控方式。这种监控方式在常规的监控里面，类似项目 zabbix telegraph，这些系统都是可以做到的。

#### 性能监控
性能监控指的就是 APM 监控，也就是说常见的一些应用性能类的监控指标的检查。通常是通过一些 Hook 的机制在虚拟机层、字节码执行层通过隐式调用，或者是在应用层显示注入，获取更深层次的一个监控指标，一般是用来应用的调优和诊断的。比较常见的类似像 jvm 或者 php 的 Zend Engine，通过一些常见的 Hook 机制，拿到类似像 jvm 里面的 GC 的次数，各种内存代的一个分布以及网络连接数的一些指标，通过这种方式来进行应用的性能诊断和调优。

#### 安全监控
安全监控主要是对安全进行的一系列的监控策略，类似像越权管理、安全漏洞扫描等等。

#### 事件监控
事件监控是 K8s 中比较另类的一种监控方式。在上一节课中给大家介绍了在 K8s 中的一个设计理念，就是基于状态机的一个状态转换。从正常的状态转换成另一个正常的状态的时候，会发生一个 normal 的事件，而从一个正常状态转换成一个异常状态的时候，会发生一个 warning 的事件。通常情况下，warning 的事件是我们比较关心的，而事件监控就是可以把 normal 的事件或者是 warning 事件离线到一个数据中心，然后通过数据中心的分析以及报警，把相应的一些异常通过像钉钉或者是短信、邮件的方式进行暴露，弥补常规监控的一些缺陷和弊端。
### Kubernetes 的监控演进
![](/img/heapster.png)
首先，我们在每一个 Kubernetes 上面有一个包裹好的 cadvisor，这个 cadvisor 是负责数据采集的组件。当 cadvisor 把数据采集完成，Kubernetes 会把 cadvisor 采集到的数据进行包裹，暴露成相应的 API。在早期的时候，实际上是有三种不同的 API：

* summary 接口；
* kubelet 接口；
* Prometheus 接口。
这三种接口，其实对应的数据源都是 cadvisor，只是数据格式有所不同。而在 Heapster 里面，其实支持了 summary 接口和 kubelet 两种数据采集接口，Heapster 会定期去每一个节点拉取数据，在自己的内存里面进行聚合，然后再暴露相应的 service，供上层的消费者进行使用。在 K8s 中比较常见的消费者，类似像 dashboard，或者是 HPA-Controller，它通过调用 service 获取相应的监控数据，来实现相应的弹性伸缩，以及监控数据的一个展示。

这个是以前的一个数据消费链路，这条消费链路看上去很清晰，也没有太多的一个问题，那为什么 Kubernetes 会将 Heapster 放弃掉而转换到 metrics-service 呢？其实这个主要的一个动力来源是由于 Heapster 在做监控数据接口的标准化。为什么要做监控数据接口标准化呢？

第一点在于客户的需求是千变万化的，比如说今天用 Heapster 进行了基础数据的一个资源采集，那明天的时候，我想在应用里面暴露在线人数的一个数据接口，放到自己的接口系统里进行数据的一个展现，以及类似像 HPA 的一个数据消费。那这个场景在 Heapster 下能不能做呢？答案是不可以的，所以这就是 Heapster 自身扩展性的弊端；
第二点是 Heapster 里面为了保证数据的离线能力，提供了很多的 sink，而这个 sink 包含了类似像 influxdb、sls、钉钉等等一系列 sink。这个 sink 主要做的是把数据采集下来，并且把这个数据离线走，然后很多客户会用 influxdb 做这个数据离线，在 influxdb 上去接入类似像 grafana 监控数据的一个可视化的软件，来实践监控数据的可视化。
但是后来社区发现，这些 sink 很多时候都是没有人来维护的。这也导致整个 Heapster 的项目有很多的 bug，这个 bug 一直存留在社区里面，是没有人修复的，这个也是会给社区的项目的活跃度包括项目的稳定性带来了很多的挑战。

基于这两点原因，K8s 把 Heapster 进行了 break 掉，然后做了一个精简版的监控采集组件，叫做 metrics-server。
![](/img/heapster-2.png)
上图是 Heapster 内部的一个架构。大家可以发现它分为几个部分，第一个部分是 core 部分，然后上层是有一个通过标准的 http 或者 https 暴露的这个 API。然后中间是 source 的部分，source 部分相当于是采集数据暴露的不同的接口，然后 processor 的部分是进行数据转换以及数据聚合的部分。最后是 sink 部分，sink 部分是负责数据离线的，这个是早期的 Heapster 的一个应用的架构。那到后期的时候呢，K8s 做了这个监控接口的一个标准化，逐渐就把 Heapster 进行了裁剪，转化成了 metrics-server。
![](/img/metrics-server.png)
目前 0.3.1 版本的 metrics-server 大致的一个结构就变成了上图这样，是非常简单的：有一个 core 层、中间的 source 层，以及简单的 API 层，额外增加了 API Registration 这层。这层的作用就是它可以把相应的数据接口注册到 K8s 的 API server 之上，以后客户不再需要通过这个 API 层去访问 metrics-server，而是可以通过这个 API 注册层，通过 API server 访问 API 注册层，再到 metrics-server。这样的话，真正的数据消费方可能感知到的并不是一个 metrics-server，而是说感知到的是实现了这样一个 API 的具体的实现，而这个实现是 metrics-server。这个就是 metrics-server 改动最大的一个地方。
### Kubernetes 的监控接口标准
在 K8s 里面针对于监控，有三种不同的接口标准。它将监控的数据消费能力进行了标准化和解耦，实现了一个与社区的融合，社区里面主要分为三类。

第一类 Resource Metrice
对应的接口是 metrics.k8s.io，主要的实现就是 metrics-server，它提供的是资源的监控，比较常见的是节点级别、pod 级别、namespace 级别、class 级别。这类的监控指标都可以通过 metrics.k8s.io 这个接口获取到。

第二类 Custom Metrics
对应的 API 是 custom.metrics.k8s.io，主要的实现是 Prometheus。它提供的是资源监控和自定义监控，资源监控和上面的资源监控其实是有覆盖关系的，而这个自定义监控指的是：比如应用上面想暴露一个类似像在线人数，或者说调用后面的这个数据库的 MySQL 的慢查询。这些其实都是可以在应用层做自己的定义的，然后并通过标准的 Prometheus 的 client，暴露出相应的 metrics，然后再被 Prometheus 进行采集。

而这类的接口一旦采集上来也是可以通过类似像 custom.metrics.k8s.io 这样一个接口的标准来进行数据消费的，也就是说现在如果以这种方式接入的 Prometheus，那你就可以通过 custom.metrics.k8s.io 这个接口来进行 HPA，进行数据消费。

第三类 External Metrics
External Metrics 其实是比较特殊的一类，因为我们知道 K8s 现在已经成为了云原生接口的一个实现标准。很多时候在云上打交道的是云服务，比如说在一个应用里面用到了前面的是消息队列，后面的是 RBS 数据库。那有时在进行数据消费的时候，同时需要去消费一些云产品的监控指标，类似像消息队列中消息的数目，或者是接入层 SLB 的 connection 数目，SLB 上层的 200 个请求数目等等，这些监控指标。

那怎么去消费呢？也是在 K8s 里面实现了一个标准，就是 external.metrics.k8s.io。主要的实现厂商就是各个云厂商的 provider，通过这个 provider 可以通过云资源的监控指标。在阿里云上面也实现了阿里巴巴 cloud metrics adapter 用来提供这个标准的 external.metrics.k8s.io 的一个实现。

Promethues - 开源社区的监控“标准”
接下来我们来看一个比较常见的开源社区里面的监控方案，就是 Prometheus。Prometheus 为什么说是开源社区的监控标准呢？

一是因为首先 Prometheus 是 CNCF 云原生社区的一个毕业项目。然后第二个是现在有越来越多的开源项目都以 Prometheus 作为监控标准，类似说我们比较常见的 Spark、Tensorflow、Flink 这些项目，其实它都有标准的 Prometheus 的采集接口。
第二个是对于类似像比较常见的一些数据库、中间件这类的项目，它都有相应的 Prometheus 采集客户端。类似像 ETCD、zookeeper、MySQL 或者说 PostgreSQL，这些其实都有相应的这个 Prometheus 的接口，如果没有的，社区里面也会有相应的 exporter 进行接口的一个实现。
那我们先来看一下 Prometheus 整个的大致一个结构。

![](/img/prometheus.png)
上图是 Prometheus 采集的数据链路，它主要可以分为三种不同的数据采集链路。

第一种，是这个 push 的方式，就是通过 pushgateway 进行数据采集，然后数据线到 pushgateway，然后 Prometheus 再通过 pull 的方式去 pushgateway 去拉数据。这种采集方式主要应对的场景就是你的这个任务可能是比较短暂的，比如说我们知道 Prometheus，最常见的采集方式是拉模式，那带来一个问题就是，一旦你的数据声明周期短于数据的采集周期，比如我采集周期是 30s，而我这个任务可能运行 15s 就完了。这种场景之下，可能会造成有些数据漏采。对于这种场景最简单的一个做法就是先通过 pushgateway，先把你的 metrics push下来，然后再通过 pull 的方式从 pushgateway 去拉数据，通过这种方式可以做到，短时间的不丢作业任务。
第二种是标准的 pull 模式，它是直接通过拉模式去对应的数据的任务上面去拉取数据。
第三种是 Prometheus on Prometheus，就是可以通过另一个 Prometheus 来去同步数据到这个 Prometheus。
这是三种 Prometheus 中的采集方式。那从数据源上面，除了标准的静态配置，Prometheus 也支持 service discovery。也就是说可以通过一些服务发现的机制，动态地去发现一些采集对象。在 K8s 里面比较常见的是可以有 Kubernetes 的这种动态发现机制，只需要配置一些 annotation，它就可以自动地来配置采集任务来进行数据采集，是非常方便的。

在报警上面，Prometheus 提供了一个外置组件叫 Alentmanager，它可以将相应的报警信息通过邮件或者短信的方式进行数据的一个告警。在数据消费上面，可以通过上层的 API clients，可以通过 web UI，可以通过 Grafana 进行数据的展现和数据的消费。

总结起来 Prometheus 有如下五个特点：
1. 简洁强大的接入标准，开发者只需要实现 Prometheus Client 这样一个接口标准，就可以直接实现数据的一个采集；
2. 多种的数据采集、离线的方式。可以通过 push 的方式、 pull 的方式、Prometheus on Prometheus的方式来进行数据的采集和离线；
3.兼容k8s；
4. 丰富的插件机制与生态；
5. Prometheus Operator 的一个助力，Prometheus Operator 可能是目前我们见到的所有 Operator 里面做的最复杂的，但是它里面也是把 Prometheus 这种动态能力做到淋漓尽致的一个 Operator，如果在 K8s 里面使用 Prometheus，比较推荐大家使用 Prometheus Operator 的方式来去进行部署和运维。

#### kube-eventer - Kubernetes 事件离线工具
最后，我们给大家介绍一个 K8s 中的事件离线工具叫做 kube-eventer。kube-eventer 是阿里云容器服务开源出的一个组件，它可以将 K8s 里面，类似像 pod eventer、node eventer、核心组件的 eventer、crd 的 eventer 等等一系列的 eventer，通过 API sever 的这个 watch 机制离线到类似像 SLS、Dingtalk、kafka、InfluxDB，然后通过这种离线的机制进行一个时间的审计、监控和告警，我们现在已经把这个项目开源到 GitHub 上了，大家有兴趣的话可以来看一下这个项目。
![](/img/kube-eventer.png)
### 日志
k8s日志四大场景

#### 主机内核的日志
* 主机内核的日志，主机内核日志可以协助开发者进行一些常见的问题与诊断，比如说网栈的异常，类似像我们的 iptables mark，它可以看到有 controller table 这样的一些 message；
* 是驱动异常，比较常见的是一些网络方案里面有的时候可能会出现驱动异常，或者说是类似 GPU 的一些场景，驱动异常可能是比较常见的一些错误；
* 文件系统异常，在早期 docker 还不是很成熟的场景之下，overlayfs 或者是 AUFS，实际上是会经常出现问题的。在这些出现问题后，开发者是没有太好的办法来去进行监控和诊断的。这一部分，其实是可以主机内核日志里面来查看到一些异常；
* 影响节点的一些异常，比如说内核里面的一些 kernel panic，或者是一些 OOM，这些也会在主机日志里面有相应的一些反映。
#### Runtime 的日志
比较常见的是 Docker 的一些日志，我们可以通过 docker 的日志来排查类似像删除一些 Pod Hang 这一系列的问题。

#### 核心组件的日志
第三个是核心组件的日志，在 K8s 里面核心组件包含了类似像一些外置的中间件，类似像 etcd，或者像一些内置的组件，类似像 API server、kube-scheduler、controller-manger、kubelet 等等这一系列的组件。而这些组件的日志可以帮我们来看到整个 K8s 集群里面管控面的一个资源的使用量，然后以及目前运行的一个状态是否有一些异常。

还有的就是类似像一些核心的中间件，如 Ingress 这种网络中间件，它可以帮我们来看到整个的一个接入层的一个流量，通过 Ingress 的日志，可以做到一个很好的接入层的一个应用分析。

#### 部署应用的日志
最后是部署应用的日志，可以通过应用的日志来查看业务层的一个状态。比如说可以看业务层有没有 500 的请求？有没有一些 panic？有没有一些异常的错误的访问？那这些其实都可以通过应用日志来进行查看的。

#### 日志的采集
首先我们来看一下日志采集，从采集位置是哪个划分，需要支持如下三种
![](/img/log-collect.png)

* 宿主机文件，这种场景比较常见的是说我的这个容器里面，通过类似像 volume，把日志文件写到了宿主机之上。通过宿主机的日志轮转的策略进行日志的轮转，然后再通过我的宿主机上的这个 agent 进行采集；
* 容器内有日志文件，那这种常见方式怎么处理呢，比较常见的一个方式是说我通过一个 Sidecar 的 streaming 的 container，转写到 stdout，通过 stdout 写到相应的 log-file，然后再通过本地的一个日志轮转，然后以及外部的一个 agent 采集；
* 直接写到 stdout，这种比较常见的一个策略，第一种就是直接我拿这个 agent 去采集到远端，第二种我直接通过类似像一些 sls 的标准 API 采集到远端。
* 
社区推荐使用 **Fluentd **的一个采集方案，Fluentd 是在每一个节点上面都会起相应的 agent，然后这个 agent 会把数据汇集到一个 Fluentd 的一个 server，这个 server 里面可以将数据离线到相应的类似像 elasticsearch，然后再通过 kibana 做展现；或者是离线到 influxdb，然后通过 Grafana 做展现。这个其实是社区里目前比较推荐的一个做法。
#### 阿里云容器服务监控体系
* sls
* arms
* ahas
* cloud monitor
阿里云增强的功能
* metrics-server
* 阿里云在 metrics-server 上面做了完整的 Heapster 兼容
* enventer、hpd、Prometheus生态
阿里云容器服务日志体系
在日志上面，阿里云做了哪些增强呢？首先是采集方式上，做到了完整的一个兼容。可以采集 pod log 日志、核心组件日志、docker engine 日志、kernel 日志，以及类似像一些中间件的日志，都收集到 SLS。收集到 SLS 之后，我们可以通过数据离线到 OSS，离线到 Max Compute，做一个数据的离线和归档，以及离线预算。

然后还有是对于一些数据的实时消费，我们可以到 Opensearch、可以到 E-Map、可以到 Flink，来做到一个日志的搜索和上层的一个消费。在日志展现上面，我们可以既对接开源的 Grafana，也可以对接类似像 DataV，去做数据展示，实现一个完整的数据链路的采集和消费。

## Kubernetes Service
### 为什么需要服务发现
在 K8s 集群里面会通过 pod 去部署应用，与传统的应用部署不同，传统应用部署在给定的机器上面去部署，我们知道怎么去调用别的机器的 IP 地址。但是在 K8s 集群里面应用是通过 pod 去部署的， 而 pod 生命周期是短暂的。在 pod 的生命周期过程中，比如它创建或销毁，它的 IP 地址都会发生变化，这样就不能使用传统的部署方式，不能指定 IP 去访问指定的应用。

在 K8s 的应用部署里，之前虽然学习了 deployment 的应用部署模式，但还是需要创建一个 pod 组，然后这些 pod 组需要提供一个统一的访问入口，以及怎么去控制流量负载均衡到这个组里面。比如说测试环境、预发环境和线上环境，其实在部署的过程中需要保持同样的一个部署模板以及访问方式。因为这样就可以用同一套应用的模板在不同的环境中直接发布。

#### Service：Kubernetes 中的服务返现与负载均衡
最后应用服务需要暴露到外部去访问，需要提供给外部的用户去调用的,pod 的网络跟机器不是同一个段的网络，那怎么让 pod 网络暴露到去给外部访问呢？这时就需要服务发现。
![](/img/service.png)
在 K8s 里面，服务发现与负载均衡就是 K8s Service。上图就是在 K8s 里 Service 的架构，K8s Service 向上提供了外部网络以及 pod 网络的访问，即外部网络可以通过 service 去访问，pod 网络也可以通过 K8s Service 去访问。

向下，K8s 对接了另外一组 pod，即可以通过 K8s Service 的方式去负载均衡到一组 pod 上面去，这样相当于解决了前面所说的复发性问题，或者提供了统一的访问入口去做服务发现，然后又可以给外部网络访问，解决不同的 pod 之间的访问，提供统一的访问地址。
#### Service endpoints
![](/img/service-endpoints.png)

实际的架构如上图所示。在 service 创建之后，它会在集群里面创建一个虚拟的 IP 地址以及端口，在集群里，所有的 pod 和 node 都可以通过这样一个 IP 地址和端口去访问到这个 service。这个 service 会把它选择的 pod 及其 IP 地址都挂载到后端。这样通过 service 的 IP 地址访问时，就可以负载均衡到后端这些 pod 上面去。

当 pod 的生命周期有变化时，比如说其中一个 pod 销毁，service 就会自动从后端摘除这个 pod。这样实现了：就算 pod 的生命周期有变化，它访问的端点是不会发生变化的。

### 集群内访问 Service

在集群里面，其他 pod 要怎么访问到我们所创建的这个 service 呢？有三种方式：

- 首先我们可以通过 service 的虚拟 IP 去访问，比如说刚创建的 my-service 这个服务，通过 kubectl get svc 或者 kubectl discribe service 都可以看到它的虚拟 IP 地址是 172.29.3.27，端口是 80，然后就可以通过这个虚拟 IP 及端口在 pod 里面直接访问到这个 service 的地址。
- 第二种方式直接访问服务名，依靠 DNS 解析，就是同一个 namespace 里 pod 可以直接通过 service 的名字去访问到刚才所声明的这个 service。不同的 namespace 里面，我们可以通过 service 名字加“.”，然后加 service 所在的哪个 namespace 去访问这个 service，例如我们直接用 curl 去访问，就是 my-service:80 就可以访问到这个 service。
- 第三种是通过环境变量访问，在同一个 namespace 里的 pod 启动时，K8s 会把 service 的一些 IP 地址、端口，以及一些简单的配置，通过环境变量的方式放到 K8s 的 pod 里面。在 K8s pod 的容器启动之后，通过读取系统的环境变量比读取到 namespace 里面其他 service 配置的一个地址，或者是它的端口号等等。比如在集群的某一个 pod 里面，可以直接通过 curl $ 取到一个环境变量的值，比如取到 MY*SERVICE*SERVICE*HOST 就是它的一个 IP 地址，MY*SERVICE 就是刚才我们声明的 MY*SERVICE，SERVICE*PORT 就是它的端口号，这样也可以请求到集群里面的 MY_SERVICE 这个 service。

### Headless Service

service 有一个特别的形态就是 Headless Service。service 创建的时候可以指定 clusterIP:None，告诉 K8s 说我不需要 clusterIP（就是刚才所说的集群里面的一个虚拟 IP），然后 K8s 就不会分配给这个 service 一个虚拟 IP 地址，它没有虚拟 IP 地址怎么做到负载均衡以及统一的访问入口呢？

它是这样来操作的：pod 可以直接通过 service_name 用 DNS 的方式解析到所有后端 pod 的 IP 地址，通过 DNS 的 A 记录的方式会解析到所有后端的 Pod 的地址，由客户端选择一个后端的 IP 地址，这个 A 记录会随着 pod 的生命周期变化，返回的 A 记录列表也发生变化，这样就要求客户端应用要从 A 记录把所有 DNS 返回到 A 记录的列表里面 IP 地址中，客户端自己去选择一个合适的地址去访问 pod。

![](/img/service-dns.png)

可以从上图看一下跟刚才我们声明的模板的区别，就是在中间加了一个 clusterIP:None，即表明不需要虚拟 IP。实际效果就是集群的 pod 访问 my-service 时，会直接解析到所有的 service 对应 pod 的 IP 地址，返回给 pod，然后 pod 里面自己去选择一个 IP 地址去直接访问。

### 向集群外暴露 Service

前面介绍的都是在集群里面 node 或者 pod 去访问 service，service 怎么去向外暴露呢？怎么把应用实际暴露给公网去访问呢？这里 service 也有两种类型去解决这个问题，一个是 NodePort，一个是 LoadBalancer。

- NodePort 的方式就是在集群的 node 上面（即集群的节点的宿主机上面）去暴露节点上的一个端口，这样相当于在节点的一个端口上面访问到之后就会再去做一层转发，转发到虚拟的 IP 地址上面，就是刚刚宿主机上面 service 虚拟 IP 地址。
- LoadBalancer 类型就是在 NodePort 上面又做了一层转换，刚才所说的 NodePort 其实是集群里面每个节点上面一个端口，LoadBalancer 是在所有的节点前又挂一个负载均衡。比如在阿里云上挂一个 SLB，这个负载均衡会提供一个统一的入口，并把所有它接触到的流量负载均衡到每一个集群节点的 node pod 上面去。然后 node pod 再转化成 ClusterIP，去访问到实际的 pod 上面。

### 访问Service方式

* ClusterIP
* service名(不同的namespace通过namespace.service_name)
* 环境变量

在应用的组件调用的时候可以不用关心 pod 在生命周期的一个变化

#### K8s 架构设计

#### Kubernetes 服务发现架构

![](/img/k8s-arch.png)

K8s 分为 master 节点和 worker 节点：

- master 里面主要是 K8s 管控的内容；
- worker 节点里面是实际跑用户应用的一个地方。

在 K8s master 节点里面有 APIServer，就是统一管理 K8s 所有对象的地方，所有的组件都会注册到 APIServer 上面去监听这个对象的变化，比如说我们刚才的组件 pod 生命周期发生变化，这些事件。

这里面最关键的有三个组件：

- 一个是 Cloud Controller Manager，负责去配置 LoadBalancer 的一个负载均衡器给外部去访问；
- 另外一个就是 Coredns，就是通过 Coredns 去观测 APIServer 里面的 service 后端 pod 的一个变化，去配置 service 的 DNS 解析，实现可以通过 service 的名字直接访问到 service 的虚拟 IP，或者是 Headless 类型的 Service 中的 IP 列表的解析；
- 然后在每个 node 里面会有 kube-proxy 这个组件，它通过监听 service 以及 pod 变化，然后实际去配置集群里面的 node pod 或者是虚拟 IP 地址的一个访问。

实际访问链路是什么样的呢？比如说从集群内部的一个 Client Pod3 去访问 Service，就类似于刚才所演示的一个效果。Client Pod3 首先通过 Coredns 这里去解析出 ServiceIP，Coredns 会返回给它 ServiceName 所对应的 service IP 是什么，这个 Client Pod3 就会拿这个 Service IP 去做请求，它的请求到宿主机的网络之后，就会被 kube-proxy 所配置的 iptables 或者 IPVS 去做一层拦截处理，之后去负载均衡到每一个实际的后端 pod 上面去，这样就实现了一个负载均衡以及服务发现。

对于外部的流量，比如说刚才通过公网访问的一个请求。它是通过外部的一个负载均衡器 Cloud Controller Manager 去监听 service 的变化之后，去配置的一个负载均衡器，然后转发到节点上的一个 NodePort 上面去，NodePort 也会经过 kube-proxy 的一个配置的一个 iptables，把 NodePort 的流量转换成 ClusterIP，紧接着转换成后端的一个 pod 的 IP 地址，去做负载均衡以及服务发现。这就是整个 K8s 服务发现以及 K8s Service 整体的结构。

## 从0开始创作云原生应用

### 云原生应用是什么?

![](/img/cloud-native-app.png)

首先我们要准备好它所需的环境，打包成一个 docker 镜像，把这个镜像放到 deployment 中。部署服务、应用所需要的账户、权限、匿名空间、秘钥信息，还有可持久化存储，这些 Kubernetes 的资源，简而言之就是要把一堆 yaml 配置文件布置在 Kubernetes 上。

![](/img/cloud-native-app2.png)

虽然应用的开发者可以把这些镜像存放在公共的仓库中，然后把部署所需要的 yaml 的资源文件提供给用户，当然用户仍然需要自己去寻找这些资源文件在哪里，并把它们一一部署起来。倘若用户希望修改开发者提供的默认资源，比如说想使用更多的副本数，或者是修改服务端口，那他还需要自己去查：在这些资源文件中，哪些地方需要相应的修改。同时版本更替和维护，也会给开发者和用户造成很大的麻烦，所以可以看到最原始的这种 Kubernetes 的应用形态并不是非常的便利。

### Helm 与Helm Chart

开发者安装 Helm Chart 的格式，将应用所需要的资源文件都包装起来，通过模板化的方法，将一些可变的字段，比如说我们之前提到的要暴露哪一个端口、使用多少副本数量，把这些信息都暴露给用户，最后将封装好的应用包，也就是我们所说的 Helm Chart 集中存放在统一的仓库里面供用户浏览下载。

那么对于用户而言，使用 Helm 一条简单的命令就可以完成应用的安装、卸载和升级，我们可以在安装完成之后使用 kubectl 来查看一下应用安装后的 pod 的运行状态。需要注意的是，我们证明使用的是 Helm v3 的一个命令，与目前相对较为成熟的 Helm v2 是有一定的区别的。我们推荐大家在进行学习尝鲜的过程中使用这个最新的 v3 版本。

[Helm](https://helm.sh/zh/)

## 深入理解Linux容器

#### 容器

容器是一种轻量级的虚拟化技术，因为它跟虚拟机比起来，它少了一层 hypervisor 层。先看一下下面这张图，这张图简单描述了一个容器的启动过程。

![](/img/container.png)

最下面是一个磁盘，容器的镜像是存储在磁盘上面的。上层是一个容器引擎，容器引擎可以是 docker，也可以是其它的容器引擎。引擎向下发一个请求，比如说创建容器，然后这时候它就把磁盘上面的容器镜像，运行成在宿主机上的一个进程。

对于容器来说，最重要的是怎么保证这个进程所用到的资源是被隔离和被限制住的，在 Linux 内核上面是由 cgroup 和 namespace 这两个技术来保证的。接下来以 docker 为例，来详细介绍一下资源隔离和容器镜像这两部分内容。

### 一、资源隔离和限制

#### namespace

namespace 是用来做资源隔离的，在 Linux 内核上有七种 namespace，docker 中用到了前六种。第七种 cgroup namespace 在 docker 本身并没有用到，但是在 runC 实现中实现了 cgroup namespace。

![](/img/namespace.png)

- 第一个是 mout namespace。mout namespace 就是保证容器看到的文件系统的视图，是容器镜像提供的一个文件系统，也就是说它看不见宿主机上的其他文件，除了通过 -v 参数 bound 的那种模式，是可以把宿主机上面的一些目录和文件，让它在容器里面可见的。
- 第二个是 uts namespace，这个 namespace 主要是隔离了 hostname 和 domain。
- 第三个是 pid namespace，这个 namespace 是保证了容器的 init 进程是以 1 号进程来启动的。
- 第四个是网络 namespace，除了容器用 host 网络这种模式之外，其他所有的网络模式都有一个自己的 network namespace 的文件。
- 第五个是 user namespace，这个 namespace 是控制用户 UID 和 GID 在容器内部和宿主机上的一个映射，不过这个 namespace 用的比较少。
- 第六个是 IPC namespace，这个 namespace 是控制了进程兼通信的一些东西，比方说信号量。
- 第七个是 cgroup namespace，上图右边有两张示意图，分别是表示开启和关闭 cgroup namespace。用 cgroup namespace 带来的一个好处是容器中看到的 cgroup 视图是以根的形式来呈现的，这样的话就和宿主机上面进程看到的 cgroup namespace 的一个视图方式是相同的。另外一个好处是让容器内部使用 cgroup 会变得更安全。

这里我们简单用 unshare 示例一下 namespace 创立的过程。容器中 namespace 的创建其实都是用 unshare 这个系统调用来创建的。

![](/img/namespace-example.png)

上图上半部分是 unshare 使用的一个例子，下半部分是我实际用 unshare 这个命令去创建的一个 pid namespace。可以看到这个 bash 进程已经是在一个新的 pid namespace 里面，然后 ps 看到这个 bash 的 pid 现在是 1，说明它是一个新的 pid namespace。

#### cgroup

#### 两种 cgroup 驱动

cgroup 主要是做资源限制的，docker 容器有两种 cgroup 驱动：一种是 systemd 的，另外一种是 cgroupfs 的。

* systemd cgroup driver
* cgroupfs cgroup driver

- **cgroupfs** 比较好理解。比如说要限制内存是多少，要用 CPU share 为多少，其实直接把 pid 写入对应的一个 cgroup 文件，然后把对应需要限制的资源也写入相应的 memory cgroup 文件和 CPU 的 cgroup 文件就可以了。
- 另外一个是 **systemd** 的一个 cgroup 驱动。这个驱动是因为 systemd 本身可以提供一个 cgroup 管理方式。所以如果用 systemd 做 cgroup 驱动的话，所有的写 cgroup 操作都必须通过 systemd 的接口来完成，不能手动更改 cgroup 的文件。

#### 容器中常用的 cgroup

接下来看一下容器中常用的 cgroup。Linux 内核本身是提供了很多种 cgroup，但是 docker 容器用到的大概只有下面六种：

![](/img/cgroup.png)

- 第一个是 CPU，CPU 一般会去设置 cpu share 和 cupset，控制 CPU 的使用率。
- 第二个是 memory，是控制进程内存的使用量。
- 第三个 device ，device 控制了你可以在容器中看到的 device 设备。
- 第四个 freezer。它和第三个 cgroup（device）都是为了安全的。当你停止容器的时候，freezer 会把当前的进程全部都写入 cgroup，然后把所有的进程都冻结掉，这样做的目的是，防止你在停止的时候，有进程会去做 fork。这样的话就相当于防止进程逃逸到宿主机上面去，是为安全考虑。
- 第五个是 blkio，blkio 主要是限制容器用到的磁盘的一些 IOPS 还有 bps 的速率限制。因为 cgroup 不唯一的话，blkio 只能限制同步 io，docker io 是没办法限制的。
- 第六个是 pid cgroup，pid cgroup 限制的是容器里面可以用到的最大进程数量。

#### 不常用的 cgroup

也有一部分是 docker 容器没有用到的 cgroup。容器中常用的和不常用的，这个区别是对 docker 来说的，因为对于 runC 来说，除了最下面的 rdma，所有的 cgroup 其实都是在 runC 里面支持的，但是 docker 并没有开启这部分支持，所以说 docker 容器是不支持以下这些 cgroup 的。

* net_cls
* net_prod
* hugetlb
* perf_event
* rdma

### 二、容器镜像

#### docker images

接下来我们讲一下容器镜像，以 docker 镜像为例去讲一下容器镜像的构成。

docker 镜像是基于联合文件系统的。简单描述一下联合文件系统：大概的意思就是说，它允许文件是存放在不同的层级上面的，但是最终是可以通过一个统一的视图，看到这些层级上面的所有文件。

![](/img/docker-images.png)

如上图所示，右边是从 docker 官网拿过来的容器存储的一个结构图。这张图非常形象的表明了 docker 的存储，docker 存储也就是基于联合文件系统，是分层的。每一层是一个 Layer，这些 Layer 由不同的文件组成，它是可以被其他镜像所复用的。可以看一下，当镜像被运行成一个容器的时候，最上层就会是一个容器的读写层。这个容器的读写层也可以通过 commit 把它变成一个镜像顶层最新的一层。

docker 镜像的存储，它的底层是基于不同的文件系统的，所以它的存储驱动也是针对不同的文件系统作为定制的，比如 AUFS、btrfs、devicemapper 还有 overlay。docker 对这些文件系统做了一些相对应的一个 graph driver 的驱动，也就是通过这些驱动把镜像存在磁盘上面。

#### 以 overlay 为例

#### 存储流程

接下来我们以 overlay 这个文件系统为例，看一下 docker 镜像是怎么在磁盘上进行存储的。先看一下下面这张图，简单地描述了 overlay 文件系统的工作原理 。

![](/img/overlay.png)

最下层是一个 lower 层，也就是镜像层，它是一个只读层。右上层是一个 upper 层，upper 是容器的读写层，upper 层采用了写实复制的机制，也就是说只有对某些文件需要进行修改的时候才会从 lower 层把这个文件拷贝上来，之后所有的修改操作都会对 upper 层的副本进行修改。

upper 并列的有一个 workdir，它的作用是充当一个中间层的作用。也就是说，当对 upper 层里面的副本进行修改时，会先放到 workdir，然后再从 workdir 移到 upper 里面去，这个是 overlay 的工作机制。

最上面的是 mergedir，是一个统一视图层。从 mergedir 里面可以看到 upper 和 lower 中所有数据的整合，然后我们 docker exec 到容器里面，看到一个文件系统其实就是 mergedir 统一视图层。

#### 文件操作

接下来我们讲一下基于 overlay 这种存储，怎么对容器里面的文件进行操作？

先看一下读操作，容器刚创建出来的时候，upper 其实是空的。这个时候如果去读的话，所有数据都是从 lower 层读来的。

写操作如刚才所提到的，overlay 的 upper 层有一个写实数据的机制，对一些文件需要进行操作的时候，overlay 会去做一个 copy up 的动作，然后会把文件从 lower 层拷贝上来，之后的一些写修改都会对这个部分进行操作。

然后看一下删除操作，overlay 里面其实是没有真正的删除操作的。它所谓的删除其实是通过对文件进行标记，然后从最上层的统一视图层去看，看到这个文件如果做标记，就会让这个文件显示出来，然后就认为这个文件是被删掉的。这个标记有两种方式：

- 一种是 whiteout 的方式。
- 第二个就是通过设置目录的一个扩展权限，通过设置扩展参数来做到目录的删除。

![](/img/container-op.png)

第二张图是 mount，可以看到这个容器 rootfs 的一个挂载，它是一个 overlay 的 type 作为挂载的。里面包括了 upper、lower 还有 workdir 这三个层级。

接下来看一下容器里面新文件的写入。docker exec 去创建一个新文件，diff 这个从上面可以看到，是它的一个 upperdir。再看 upperdir 里面有这个文件，文件里面的内容也是 docker exec 写入的。

最后看一下最下面的是 mergedir，mergedir 里面整合的 upperdir 和 lowerdir 的内容，也可以看到我们写入的数据。

### 三、容器引擎

#### containerd 容器架构详解

接下来讲一下容器引擎，我们基于 CNCF 的一个容器引擎上的 containerd，来讲一下容器引擎大致的构成。下图是从 containerd 官网拿过来的一张架构图，基于这张架构图先简单介绍一下 containerd 的架构。

![](/img/containerd-arch.png)

上图如果把它分成左右两边的话，可以认为 containerd 提供了两大功能。

第一个是对于 runtime，也就是对于容器生命周期的管理，左边 storage 的部分其实是对一个镜像存储的管理。containerd 会负责进行的拉取、镜像的存储。

按照水平层次来看的话:

- 第一层是 GRPC，containerd 对于上层来说是通过 GRPC serve 的形式来对上层提供服务的。Metrics 这个部分主要是提供 cgroup Metrics 的一些内容。
- 下面这层的左边是容器镜像的一个存储，中线 images、containers 下面是 Metadata，这部分 Matadata 是通过 **bootfs **存储在磁盘上面的。右边的 Tasks 是管理容器的容器结构，Events 是对容器的一些操作都会有一个 Event 向上层发出，然后上层可以去订阅这个 Event，由此知道容器状态发生什么变化。
- 最下层是 Runtimes 层，这个 Runtimes 可以从类型区分，比如说 runC 或者是安全容器之类的。

#### shim v1/v2 是什么

接下来讲一下 containerd 在 runtime 这边的大致架构。下面这张图是从 kata 官网拿过来的，上半部分是原图，下半部分加了一些扩展示例，基于这张图我们来看一下 containerd 在 runtime 这层的架构。

![](/img/shim.png)

如图所示：按照从左往右的一个顺序，从上层到最终 runtime 运行起来的一个流程。

我们先看一下最左边，最左边是一个 CRI Client。一般就是 kubelet 通过 CRI 请求，向 containerd 发送请求。containerd 接收到容器的请求之后，会经过一个 containerd shim。containerd shim 是管理容器生命周期的，它主要负责两方面：

- 第一个是它会对 io 进行转发。
- 第二是它会对信号进行传递。

图的上半部分画的是安全容器，也就是 kata 的一个流程，这个就不具体展开了。下半部分，可以看到有各种各样不同的 shim。下面介绍一下 containerd shim 的架构。

一开始在 containerd 中只有一个 shim，也就是蓝色框框起来的 containerd-shim。这个进程的意思是，不管是 kata 容器也好、runc 容器也好、gvisor 容器也好，上面用的 shim 都是 containerd。

后面针对不同类型的 runtime，containerd 去做了一个扩展。这个扩展是通过 shim-v2 这个 interface 去做的，也就是说只要去实现了这个 shim-v2 的 interface，不同的 runtime 就可以定制不同的自己的一个 shim。比如：runC 可以自己做一个 shim，叫 shim-runc；gvisor 可以自己做一个 shim 叫 shim-gvisor；像上面 kata 也可以自己去做一个 shim-kata 的 shim。这些 shim 可以替换掉上面蓝色框的 containerd-shim。

这样做的好处有很多，举一个比较形象的例子。可以看一下 kata 这张图，它上面原先如果用 shim-v1 的话其实有三个组件，之所以有三个组件的原因是因为 kata 自身的一个限制，但是用了 shim-v2 这个架构后，三个组件可以做成一个二进制，也就是原先三个组件，现在可以变成一个 shim-kata 组件，这个可以体现出 shim-v2 的一个好处。

### containerd 容器架构详解 - 容器流程示例

接下来我们以两个示例来详细解释一下容器的流程是怎么工作的，下面的两张图是基于 containerd 的架构画的一个容器的工作流程。

#### start 流程

先看一下容器 start 的流程：

![](/img/containerd-start.png)

这张图由三个部分组成：

- 第一个部分是容器引擎部分，容器引擎可以是 docker，也可以是其它的。
- 两个虚线框框起来的 containerd 和 containerd-shim，它们两个是属于 containerd 架构的部分。
- 最下面就是 container 的部分，这个部分是通过一个 runtime 去拉起的，可以认为是 shim 去操作 runC 命令创建的一个容器。

先看一下这个流程是怎么工作的，图里面也标明了 1、2、3、4。这个 1、2、3、4 就是 containerd 怎么去创建一个容器的流程。

首先它会去创建一个 matadata，然后会去发请求给 task service 说要去创建容器。通过中间一系列的组件，最终把请求下发到一个 shim。containerd 和 shim 的交互其实也是通过 GRPC 来做交互的，containerd 把创建请求发给 shim 之后，shim 会去调用 runtime 创建一个容器出来，以上就是容器 start 的一个示例。

#### exec 流程

接下来看下面这张图，是怎么去 exec 一个容器的。和 start 流程非常相似，结构也大概相同，不同的部分其实就是 containerd 怎么去处理这部分流程。和上面的图一样，我也在图中标明了 1、2、3、4，这些步骤就代表了 containerd 去做 exec 的一个先后顺序。

由上图可以看到，exec 的操作还是发给 containerd-shim 的。对容器来说，去 start 一个容器和去 exec 一个容器，其实并没有本质的区别。

最终的一个区别无非就是，是否对容器中跑的进程做一个 namespace 的创建：

- exec 的时候，需要把这个进程加入到一个已有的 namespace 里面；
- start 的时候，容器进程的 namespace 是需要去专门创建。

## 深入理解etcd - 基本原理解析

