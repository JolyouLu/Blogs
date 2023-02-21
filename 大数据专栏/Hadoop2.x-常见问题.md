# Hadoop2.x-常见问题

## 生成环境中的常见问题

### MapReduce执行慢

> MapReduce执行慢通过由如下原因造成
>
> 1. 计算机性能
>    * CPU、内存、磁盘健康、网络
> 2. I/O操作
>    * 数据倾斜
>    * Map和Reduce数设置不合理
>    * Map运行时间太长，导致Reduce等待过久
>    * 小文件过多
>    * 大量的不可分块的超大文件
>    * Spill次数过多
>    * Merge次数过多

### MapReduce优化方法

> MapReduce优化方法主要从六个方面考虑：数据输入、Map阶段、Reduce阶段、IO传输、数据倾斜问题和常用的调优参数

#### 数据输入

> 1. 合并小文件：在执行MR任务之前将小文件合并，大量的小文件会产生大量的Map任务，增打Map任务装载次数，而任务的装载比较耗时，从而导致MR运行较慢
> 2. 采用CombineTextInputFormat来作为输入，解决输入端大量小文件场景

#### Map阶段

> 1. 减少溢写(Spill)次数：通过调整io.sort.mb及sort.spill.percent参数值，增大触发Spill的内存上限，减少Spill次数，从而减少磁盘IO
> 2. 减少合并(Merge)次数：通过调整io.sort.factor参数，增大Merge的文件数目，减少Merge的次数，从而缩短MR处理时间
> 3. 在Map之后，不影响业务逻辑前提下，进行Combine，减少I/O

#### Reduce阶段

> 1. 合理设置Map和Reduc数：两个都不能设置太少，也不能设置太多太少，会导致Task等待，延长处理数据；太多，会导致Map、Reduce任务间竞争资源，造成处理超时等错误
> 2. 设置Map、Reduce共存：调整slowstart.completedmaps参数，使Map运行到一定程度后，Reduce也开始运行，减少Reduce的等待时间
> 3. 规避使用Reduce：因为Reduce在用于连接数据集的时候将会产生大量的网络消耗
> 4. 合理设置Reduc端的Buffer：默认情况下，数据达到一个阈值的时候，Buffer中的数据会写入磁盘，然后Reduce会从磁盘中获取所有数据，也就是说，Buffer和Reduce是没有直接关联的，中间多次写磁盘->读磁盘的过程，既然有这个弊端，那么就可以通过参数来配置，使得Buffer中的一部分数据可以直接输送到Reduce，从而减少IO开销，mapred.job.reduce.input.buffer.percent，默认为0.0，当值大于0的时候，会保留指定比例的内容读Buffer中的数据直接拿给Reduce，这样一来设置Buffer需要内存，读取数据需要内存，Reduce计算需要内存，所以要根据作业的运行情况进行调整

#### IO传输

> 1. 采用数据压缩方式，减少网络IO的时间，按照Snappy和LZO压缩编码器
> 2. 使用SequenceFile二进制文件

#### 数据倾斜问题

> 1. 抽样和范围分区：可以通过对原始数据进行抽样得到的结果集来预设分区边界值
> 2. 自定义分区：基于输出键的背景知识进行自定义分区，例如，如果Map输出键的单词来源于一本书，且其中某几个作业词汇比较多，那么就可以自定义分区将这些专业词汇发送给固定的一部分Reduc实例，而将其它的都发送给剩余的Reduce实例
> 3. 使用Combine可以大量的减少数据倾斜，有可能的情况下，Combine的目的就是聚合并精简数据
> 4. 采用MapJoin，尽量避免ReduceJoin

#### 常用参数设置

> 以下参数是在用户自己的MR应用程序中配置就可以生效(mapred-default.xml`https://hadoop.apache.org/docs/r2.7.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml`)

| 配置参数                                      | 参数说明                                                     |
| --------------------------------------------- | ------------------------------------------------------------ |
| mapreduce.map.memory.mb                       | 一个MapTask可使用的资源上限(单位MB)，默认1024，如果MapTask实际使用资源超过该值，则会倍强制杀死 |
| mapreduce.reduce.memory.mb                    | 一个ReduceTask可使用的资源上限(单位MB)，默认1024，如果ReduceTask实际使用资源超过该值，则会倍强制杀死 |
| mapreduce.map.cpu.vcores                      | 每个MapTask可使用的最大cpu core数目，默认1                   |
| mapreduce.reduce.cpu.vcores                   | 每个ReduceTask可使用的最大cpu core数目，默认1                |
| mapreduce.reduce.shuffle.parallelcopies       | 每个Reduce去Map中取数据的并行书，默认5                       |
| mapreduce.reduce.shuffle.merge.percent        | Buffer中的数据达到多少比例开始写入磁盘，默认0.66             |
| mapreduce.reduce.shuffle.input.buffer.percent | Buffer大小占Reduce可用内存的比例，默认0.7                    |
| mapreduce.reduce.input.buffer.percent         | 指定多少比例的内存用来存放Buffer中的数据，默认值0.0          |

> 在YARN启动之前就配置在服务器的配置文件中才能生效(yarn-default.xml`https://hadoop.apache.org/docs/r2.7.2/hadoop-yarn/hadoop-yarn-common/yarn-default.xml`)

| 配置参数                                 | 参数说明                                    |
| ---------------------------------------- | ------------------------------------------- |
| yarn.scheduler.minimum-allocation-mb     | 给应用程序Container分配的最小内存，默认1024 |
| yarn.scheduler.maximum-allocation-mb     | 给应用程序Container分配的最大内存，默认8192 |
| yarn.scheduler.minimum-allocation-vcores | 每个Container申请的最小CPU核数，默认1       |
| yarn.scheduler.maximum-allocation-vcores | 每个Container申请的最大CPU核数，默认32      |
| yarn.nodemanager.resource.memory-mb      | 给Container分配的最大物理内存，默认8192     |

> Shuffle性能优化的关键参数，应在YARN启动之前就配置好(mapred-default.xml`https://hadoop.apache.org/docs/r2.7.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml`)

| 配置参数                         | 参数说明                          |
| -------------------------------- | --------------------------------- |
| mapreduce.task.io.sort.mb        | Shuffle的环形缓冲区大小，默认100m |
| mapreduce.map.sort.spill.percent | 环形缓冲区溢出的阈值，默认80%     |

> 如果相关参数(MapReduce性能优化)(mapred-default.xml`https://hadoop.apache.org/docs/r2.7.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml`)

| 配置参数                     | 参数说明                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| mapreduce.map.maxattempts    | 每个Map Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值4 |
| mapreduce.reduce.maxattempts | 每个Reduce Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值4 |
| mapreduce.task.timeout       | Task超时时间，经常需要设置的一个参数，该参数表达的意思为：如果一个Task在一定时间内没有任何进入，即不会读取新的数据，也没有输出数据，则认为该Task处于Block状态，可能是卡住了，也许永远会卡住，为了防止因为用户程序永远Block住不退出，则强制设置了一个该超时时间（单位毫秒），默认是600000。如果你的程序对每条输入数据的处理时间过长（比如会访问数据库，通过网络拉取数据等），建议将该参数调大，该参数过小常出现的错误提示是`AttemptID:attempt_14267829456721_123456_m_000224_0 Timed out after 300 secsContainer killed by the ApplicationMaster.` |

### HDFS小文件优化方法

> HDFS小文件都要在NameNode上建立一个索引，这个索引大小约为150byte，这样当小文件比较多的时候，就会产生好多的索引文件，一方面会大量占用NameNode的存储空间另一方面就是所有文件过大索引数据很慢

#### 小文件解决方案

> 小文件常见的解决方案有
>
> 1. 在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS
> 2. 在业务处理之前，在HDFS上使用MapReduce程序对小文件进行合并
> 3. 在MapReduce处理时，可采用CombineTextInputFormat提高效率

1. Hadoop Archive

   > 是一个高效的将小文件放入HDFS块中的文件归档工具，它能够将多个小文件打包成一个HAR文件，这样就减少了NameNode的内存使用

2. Sequence File

   > Sequence File由一系列的二进制key/value组成，如果key为文件名，value为文件内容，则可以将大批小文件合并成一个大文件

3. CombineFileInputFormat

   > CombineFileInputFormat是一种新的InputFormat，用于将多个文件合并成一个单独的Split另外，它会考虑数据的存储位置

4. 开启JVM重用

   > 对大量小文件Job，可以开启JVM重用会减少45%运行时间
   >
   > 修改参数`mapreduce.job.jvm.numtasks`值在10-20之间

