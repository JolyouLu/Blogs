# Spark3.x-基础(SparkStreaming)

SparkStreaming是准实时（秒，分钟），微批次（时间）的数据处理框架，支持多种数据源如：Kafka、Flume、Twitter、ZeroMQ和简单的TCP套接字等。数据输入后可以用Spark的高度抽象原语如：map、reduce、join等进行运算。而结果也可以保存到很多地方如HDFS数据库等

![image-20240519143738080](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519143738080.png)

和Spark基于RDD的概念很相似，SparkStreaming使用离散化流（discretized stream）作为抽象的表示，叫做DStream。DStream是随着时间推移而收到的数据的序列，在内部，每个时间区间收到的数据都作为 RDD 存在，而 DStream 是由这些 RDD 所组成的序列(因此得名“离散化”)。所以 简单来将，DStream 就是对 RDD 在实时数据处理场景的一种封装。

## SparkStreaming特点

| 特点              | 说明                                       |
| ----------------- | ------------------------------------------ |
| 易用              | 内置很多简单易用的api                      |
| 容错              | 开箱即用，无需额外操作就提高了可恢复的操作 |
| 易整合到Spark体系 |                                            |

**整体架构图**

![image-20240519145814266](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519145814266.png)

![image-20240519145233762](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519145233762.png)

**背压机制**

Spark 1.5 以前版本，用户如果要限制 Receiver 的数据接收速率，可以通过设置静态配制参 数“spark.streaming.receiver.maxRate”的值来实现，此举虽然可以通过限制接收速率，来适配当前 的处理能力，防止内存溢出，但也会引入其它问题。比如：producer 数据生产高于 maxRate，当 前集群处理能力也高于 maxRate，这就会造成资源利用率下降等问题。 为了更好的协调数据接收速率与资源处理能力，1.5 版本开始 Spark Streaming 可以动态控制 数据接收速率来适配集群数据处理能力。背压机制（即 Spark Streaming Backpressure）: 根据 JobScheduler 反馈作业的执行信息来动态调整 Receiver 数据接收率。 通过属性“spark.streaming.backpressure.enabled”来控制是否启用 backpressure 机制，默认值 false，即不启用。

## SparkStreaming核心编程

### WordCount案例

首先通过一个简单的wordCount案例属性SparkStreaming，使用 netcat 工具向 9999 端口不断的发送数据，通过 SparkStreaming 读取端口数据并 统计不同单词出现的次数

> `Linux下使用 yum -y install`安装netcat工具

**编写代码**

**依赖导入**

~~~xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.12</artifactId>
    <version>3.0.0</version>
</dependency>
~~~

**WordCount代码**

![image-20240519160406645](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519160406645.png)

**测试**

在192.168.88.100使用`nc -lk 9999`启动socket并且发送消息，可以看到控制台每隔一段时间会拉取消息并且合计、打印

![image-20240519160602362](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519160602362.png)

### DStream创建

#### RDD队列的使用

在前面WordCount案例中需要使用netcat工具测试比较麻烦，平时测试时可以使用RDD队列来替代netcat

**案例实操**

循环创建RDD写入到队列中，通过SparkStream创建的 Dstream，读取并且计算wordCount

![image-20240519164146965](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519164146965.png)

#### 自定义数据源

SparkStreamin默认提供的采集器非常有限，可以通过实现自己的自定义采集器去定期采集指定目标的数据

![image-20240519175256308](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519175256308.png)

#### Kafka数据源（重点）

##### 版本选项

ReceiverAPI：需要一个专门的Executor去接收数据，然后发送给其他的Executor做计算，存在的问题是接收数据的Executor和计算的Executor速度有所不同，特别在接收数据的Executor速度大于计算的Executor速度时，会导致计算数据的节点内存溢出。早期版本中提供此方式，当前版本不适用

DirectAPI：是由计算机的Executor来主动消费Kafka的数据，速度由自身控制

###### Kafka 0-10 Direct 模式

![image-20240519213130997](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519213130997.png)

使用命令行向Kafka的test主题发送消息，可以看到Spark定期会消费Kafka中的消息

![image-20240519213106548](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519213106548.png)

### DStream转换

DStream 上的操作与 RDD 的类似，分为 Transformations（转换）和 OutputOperations（输 出）两种，此外转换操作中还有一些比较特殊的原语，如：updateStateByKey()、transform()以及 各种 Window 相关的原语。

#### 无状态操作

无状态转化操作就是把简单的 RDD 转化操作应用到每个批次上，也就是转化 DStream 中的每 一个 RDD。部分无状态转化操作列在了下表中。注意，针对键值对的 DStream 转化操作(比如 reduceByKey())要添加 import StreamingContext._才能在 Scala 中使用。

![image-20240519215822666](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240519215822666.png)

需要记住的是，尽管这些函数看起来像作用在整个流上一样，但事实上每个 DStream 在内部 是由许多 RDD（批次）组成，且无状态转化操作是分别应用到每个 RDD 上的。 例如：reduceByKey()会归约每个时间区间中的数据，但不会归约不同区间之间的数据。

##### Transform

使用SparkStreaming那么所有操作都要通过DStream 完成，但由于DStream功能没有完全覆盖RDD可能会出现有的需求RDD能够实现DStream无法实现，那么我们可以通过Transform获取到DStream底层的RDD进行操作，Transform下对RDD的操作都是周期性执行的

![image-20240520142243501](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240520142243501.png)

##### Join

两个流之间的 join 需要两个流的批次大小一致，这样才能做到同时触发计算。计算过程就是 对当前批次的两个流中各自的 RDD 进行 join，与两个 RDD 的 join 效果相同。

![image-20240520143310822](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240520143310822.png)

#### 有状态操作

##### updateStateByKey

使用reduceByKey仅能处理每批的数据，如果需要将多个批次数据累加那么需要使用有状态的操作updateStateByKey，updateStateByKey利用检查点将每一批处理好的数据保存下来，当取到下一批数据后会将检查点的数据取出合并

![image-20240520000802301](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240520000802301.png)

##### WindowOperations

WindowOperations窗口操作，概念和`TCP协议滑动窗口很像`可以设置窗口的大小和滑动窗口的间隔来动态的获取当前Steaming，可以将多个批次Steaming合并处理，所有基于窗口的操作都需要两个参数，分别为窗口时长以及滑动步长。

![image-20240520170715543](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240520170715543.png)

关于win操作还有如下方法：

| 方法                                                         | 功能                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| window(windowLength, slideInterval)                          | 基于对源 DStream 窗化的批次进行计算返回一个 新的 Dstream     |
| countByWindow(windowLength, slideInterval)                   | 返回一个滑动窗口计数流中的元素个数                           |
| reduceByWindow(func, windowLength, slideInterval)            | 通过使用自定义函数整合滑动区间 流元素来创建一个新的单元素流  |
| reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks]) | 当在一个(K,V) 对的 DStream 上调用此函数，会返回一个新(K,V)对的 DStream，此处通过对滑动窗口中批次数 据使用 reduce 函数来整合每个 key 的 value 值 |
| reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks]) | 这个函 数是上述函数的变化版本，每个窗口的 reduce 值都是通过用前一个窗的 reduce 值来递增计算。当有新数据进入时会触发参数1的fun，当有数据离开窗口时会触发invFun方法 |

### DStream输出

DStream于RDD种的惰性求值类似，如果一个DStream以及派生出的DStream都没有被执行输出操作，那么这些DStream都不会被求值，如果StreamingContext中没有设定输出操作，整个context就都不会启动，输出操作如下：

| 方法                                | 功能                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| print()                             | 在运行流程序的驱动结点上打印 DStream 中每一批次数据的最开始 10 个元素。这 用于开发和调试。在 Python API 中，同样的操作叫 print()。 |
| saveAsTextFiles(prefix, [suffix])   | 以 text 文件形式存储这个 DStream 的内容。每一批次的存 储文件名基于参数中的 prefix 和 suffix。”prefix-Time_IN_MS[.suffix]”。 |
| saveAsObjectFiles(prefix, [suffix]) | 以 Java 对象序列化的方式将 Stream 中的数据保存为 SequenceFiles . 每一批次的存储文件名基于参数中的为"prefix-TIME_IN_MS[.suffix]". Python 中目前不可用。 |
| saveAsHadoopFiles(prefix, [suffix]) | 将 Stream 中的数据保存为 Hadoop files. 每一批次的存 储文件名基于参数中的为"prefix-TIME_IN_MS[.suffix]"。Python API 中目前不可用。 |
| foreachRDD(func)                    | 这是最通用的输出操作，即将函数 func 用于产生于 stream 的每一个 RDD。其中参数传入的函数 func 应该实现将每一个 RDD 中数据推送到外部系统，如将 RDD 存入文件或者通过网络将其写入数据库。 |

> 注意：
>
> 1. print()才会在控制台打印时间戳，foreachRDD(func)不会打印时间戳
> 2. 通用的输出操作 foreachRDD()，它用来对 DStream 中的 RDD 运行任意计算。这和 transform()  有些类似，都可以让我们访问任意 RDD。在 foreachRDD()中，可以重用我们在 Spark 中实现的 所有行动操作。比如，常见的用例之一是把数据写到诸如 MySQL 的外部数据库中。
>    1. 连接不能写在 driver 层面（序列化）
>    2. 如果写在 foreach 则每个 RDD 中的每一条数据都创建，得不偿失；使用 foreachPartition，在分区创建提高效率

### 优雅关闭

创建一个线程不断轮询外部存储如HDFS、Mysql、Redis获取状态判断是否需要关闭程序

![image-20240520174145806](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240520174145806.png)

