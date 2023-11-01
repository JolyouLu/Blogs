# Spark3.x-基础(Core)

## 运行架构

> Spark 框架的核心是一个计算引擎，整体来说，它采用了标准 master-slave 的结构。 如下图所示，它展示了一个 Spark 执行时的基本结构。图形中的 Driver 表示 master， 负责管理整个集群中的作业任务调度。图形中的 Executor 则是 slave，负责实际执行任务

![image-20231005150323004](./images/image-20231005150323004.png)

## 核心组件

**Driver**

> 从上图可以看到Driver是Spark的核心组件之一，是Spark的驱动节点，在执行Spark任务中的main方法，负责实际代码的执行工作，主要工作职责如下

1. 将用户程序转化为作业(job)
2. 在Executor之间调度任务(task)
3. 跟踪Executor的执行情况
4. 通过UI展示查询运行情况

**Executor**

> 从上图可以看到Driver是Executor的核心组件之一，是集群中工作节点(Worker)中的一个JVM进程，负责在Spark作业中运行具体任务(Task)，任务彼此之间相互独立，Spark应用启动时Executor节点会同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。如果有 Executor 节点发生了 故障或崩溃，Spark 应用也可以继续执行，会将出错节点上的任务调度到其他 Executor 节点 上继续运行

1. 负责运行组成 Spark 应用的任务，并将结果返回给驱动器进程
2. 它们通过自身的块管理器（Block Manager）为用户程序中要求缓存的 RDD 提供内存 式存储。RDD 是直接缓存在 Executor 进程内的，因此任务可以在运行时充分利用缓存 数据加速运算

**Master & Worker**

> Spark 集群的独立部署环境中，不需要依赖其他的资源调度框架，自身就实现了资源调 度的功能，所以环境中还有其他两个核心组件：Master 和 Worker，这里的 Master 是一个进 程，主要负责资源的调度和分配，并进行集群的监控等职责，类似于 Yarn 环境中的 RM, 而 Worker 呢，也是进程，一个 Worker 运行在集群中的一台服务器上，由 Master 分配资源对 数据进行并行的处理和计算，类似于 Yarn 环境中 NM

**ApplicationMaster**

> Hadoop 用户向 YARN 集群提交应用程序时,提交程序中应该包含 ApplicationMaster，用 于向资源调度器申请执行任务的资源容器 Container，运行用户自己的程序任务 job，监控整 个任务的执行，跟踪整个任务的状态，处理任务失败等异常情况。 说的简单点就是，ResourceManager（资源）和 Driver（计算）之间的解耦合靠的就是 ApplicationMaster

## 核心概念

**Executor与Core**

> Spark Executor是集群中运行在工作节点(Worker)中的一个JVM进程，是整个集群中的专门用于计算的节点，在提交应用中可以提供参数指定计算节点的个数，以及对应的资源这里的资源一般指的是工作节点Executor的内存大小和使用的虚拟CPU核(Coure)数量
>
> 在提交任务时通过如下参数调整Executor

| 名称              | 说明                                   |
| ----------------- | -------------------------------------- |
| --num-executors   | 配置 Executor 的数量                   |
| --executor-memory | 配置每个 Executor 的内存大小           |
| --executor-cores  | 配置每个 Executor 的虚拟 CPU core 数量 |

**并行度(Parallelism)**

> 在分布式计算框架中通常是多个任务同时执行的，任务会被分配到不同节点同时执行，所以才能实现真正的并行执行，`并发与并行的区别可以自行百度`在整个集群中可并行执行的任务数量称为并行度，一个作业的并行度是多少取决于框架的配置

**有向无环图(DAG)**

> 在Spark中计算的过程通常都是有先后顺序的，有的任务必须等待另外一些任务执行的限制，就必须对任务进行排队，那么就会形成一个任务对立的集合这个任务集合就是DAG图，每个顶点就是一个任务，每条边代表一种约束(Spark中的依赖关系)

![image-20231005154605863](./images/image-20231005154605863.png)

**提交流程**

> 提交流程就是我们编写好的Spark应用程序通过Spark客户端提交给Spark运行环境执行计算的流程，在不同的部署环境中提交流程基本一致，只有细微的区别，这里只列举Yarn环境的提交流程

![image-20231005155254793](./images/image-20231005155254793.png)

> Spark应用程序提交Yarn环境执行一般有2中部署方式，Client和Cluster

**Yarn Client模式**

> Client 模式将用于监控和调度的 Driver 模块在客户端执行，而不是在 Yarn 中，所以一 般用于测试

1. Driver 在任务提交的本地机器上运行
2. Driver 启动后会和 ResourceManager 通讯申请启动 ApplicationMaster
3. ResourceManager 分配 container，在合适的 NodeManager 上启动 ApplicationMaster，负责向 ResourceManager 申请 Executor 内存
4. ResourceManager 接到 ApplicationMaster 的资源申请后会分配 container，然后 ApplicationMaster 在资源分配指定的 NodeManager 上启动 Executor 进程
5. Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行 main 函数
6. 之后执行到 Action 算子时，触发一个 Job，并根据宽依赖开始划分 stage，每个 stage 生 成对应的 TaskSet,之后将 task 分发到各个 Executor 上执行

**Yarn Cluster 模式**

> Cluster 模式将用于监控和调度的 Driver 模块启动在 Yarn 集群资源中执行。一般应用于 实际生产环境

1. 在 YARN Cluster 模式下，任务提交后会和 ResourceManager 通讯申请启动 ApplicationMaster
2. 随后 ResourceManager 分配 container，在合适的 NodeManager 上启动 ApplicationMaster， 此时的 ApplicationMaster 就是 Driver
3. Driver 启动后向 ResourceManager 申请 Executor 内存，ResourceManager 接到 ApplicationMaster 的资源申请后会分配 container，然后在合适的 NodeManager 上启动 Executor 进程
4. Executor 进程启动后会向 Driver 反向注册，Executor 全部注册完成后 Driver 开始执行 main 函数
5. 之后执行到 Action 算子时，触发一个 Job，并根据宽依赖开始划分 stage，每个 stage 生 成对应的 TaskSet，之后将 task 分发到各个 Executor 上执行

## 核心编程

> Spark 计算框架为了能够进行高并发和高吞吐的数据处理，封装了三大数据结构，用于 处理不同的应用场景。三大数据结构分别是

1. RDD : 弹性分布式数据集
2. 累加器：分布式共享只写变量
3. 广播变量：分布式共享只读变量

### RDD

> RDD（Resilient Distributed Dataset）叫做弹性分布式数据集，是 Spark 中最基本的数据 处理模型。代码中是一个抽象类，它代表一个弹性的、不可变、可分区、里面的元素可并行 计算的集合
>
> RDD是一个最简单的计算单元，在执行过程中RDD会将任务工具Executor数量进行拆分并且发送到各个Executor中完成计算，当多个RDD串连起来就可以组成一个复杂的操作

![image-20231005173609081](./images/image-20231005173609081.png)

#### 特点

1. 弹性

   * 存储的弹性：内存于磁盘自动切换

   * 容错的弹性：数据丢失可以自动恢复

   * 计算的弹性：计算出错有重试机制

   * 分片弹性：可工具需要重新分片

2. 分布式：数据存储在大数据集群不同节点上

3. 数据集：RDD封装了计算逻辑，并不保存数据

4. 数据抽象：RDD是一个抽象类，需要子类具体实现

5. 不可变：RDD 封装了计算逻辑，是不可以改变的，想要改变，只能产生新的 RDD，在 新的 RDD 里面封装计算逻辑

6. 可分区、并行计算