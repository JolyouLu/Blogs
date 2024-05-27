# Spark3.x-基础(SparkSQL)

## Hive与SparkSQL

SparkSQL的前身是Shark（一个把Hive与Spark整合提供SQL查询的框架），用于给熟悉RDBMS但又不理解MapReduce的技术人员提供快速上手的根据。

Hive是早期唯一运行在Hadoop上的SQL-on-Hadoop工具。但是查询过程是通过MapReduce中间大量的磁盘落盘消耗大量的I/O，降低了计算的效率，为了提高SQL-on-Hadoop的效率，一些工具诞生了其中代表有：Drill、Impala、Shark

其中 Shark 是伯克利实验室 Spark 生态环境的组件之一，是基于Hive 所开发的工具，它修改了下图所示的右下角的内存管理、物理计划、执行三个模块，并使之能运行在 Spark 引擎上。

![image-20240516115757699](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516115757699.png)

随着发展由于原先Shark对Hive依赖性过于强（如采用 Hive 的语法解析器、查询优化器等等），制约了 Spark 的迭代速度以及性能等各方面，所以提出了SparkSQL项目，SparkSQL项目抛弃了原有的Shark代码取其精华，重新开发，完全摆脱Hive的依赖，并且保留对Hive的兼容性，SparkSQL特点

1. 数据兼容性：SparkSQL 不但兼容Hive，还可以从RDD、parquet 文件、JSON 文件中获取数据，未来版本甚至支持获取RDBMS 数据以及 cassandra 等NOSQL 数据。
2. 性能：除了采取 In-Memory Columnar Storage、byte-code generation 等优化技术外、将会引进Cost Model 对查询进行动态评估、获取最佳物理计划等等。
3. 组件扩展：无论是 SQL 的语法解析器、分析器还是优化器都可以重新定义，进行扩展。

> 2014年6月1日Shark项目和SparkSQL项目的主持人Reynold Xin 宣布：停止对 Shark 的开发，团队将所有资源放SparkSQL 项目上，至此，Shark 的发展画上了句话，但也因此发展出两个支线：SparkSQL 和 Hive on Spark。
>
> SparkSQL：作为Spark生态的一员继续发展，而不再受限于 Hive，只是兼容 Hive。
>
> Hive on Spark：是一个Hive 的发展计划，该计划将 Spark 作为Hive 的底层引擎之一，也就是说，Hive 将不再受限于一个引擎，可以采用 Map-Reduce、Tez、Spark 等引擎。

## SparkSQL特点

| 特点           | 说明                                   |
| -------------- | -------------------------------------- |
| 易整合         | 无缝的整合了SQL查询和Spark编程         |
| 统一的数据访问 | 使用相同的方式连接不同的数据源         |
| 兼容Hive       | 在已有的仓库上直接运行 SQL 或者 HiveQL |
| 标准数据连接   | 通过 JDBC 或者 ODBC 来连接             |

## SparkSQL核心编程

### 环境准备

使用SparkSQL需要具备Spark环境，环境的搭建查看[Spark3.x-基础(部署)](./Spark3.x-基础(部署))具备Local模式即刻，接下来先用Local模型做一些简单的操作用于对SparkSQL有基本的了解

> 进入到spark的bin目录使用`spark-shell`命令看到如下内容打印表示环境已经准备完毕

![image-20240516180132595](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516180132595.png)

### DataFrame

在Spark中，DataFrame是以RDD为基础的分布式数据集，类似于关系型数据库的二维表，它与RDD的区别在于RDD中显示的每个数据都只是数据本身，而DataFrame着会带着schema元信息每一列都带有名称和类型，这样使得SparkSQL得以洞察更多的结构信息，从而做出针对性的优化。

DataFrame 是为数据提供了 Schema 的视图。可以把它当做数据库中的一张表来对待DataFrame 也是懒执行的，但性能上比 RDD 要高，主要原因：优化的执行计划，即查询计划通过 Spark catalyst optimiser 进行优化

#### 创建DataFrame

创建DataFrame有两种方式：通过Spark的数据源创建，从存在的RDD进行转换

##### 1.通过Spark数据源创建

~~~scala
//读取json文件转换为DataFrame
val df = spark.read.json("C:/Users/JolyouLu/Downloads/user.json")
//打印内容
df.show
~~~

![image-20240516181420608](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516181420608.png)

##### 2.从存在的RDD进行转换

~~~scala
//构建RDD
val rdd = sc.makeRDD(List(("张三",30),("李四",25),("王五",30)))
//转DataFrame
val df = rdd.toDF("username","age")
//显示
df.show
~~~

![image-20240516181903760](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516181903760.png)

#### DataFrame转换RDD

DataFrame是对RDD的封装，所以可以直接获取内部rdd

~~~scala
//DataFrame转换RDD
df.rdd
~~~

![image-20240516183004206](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516183004206.png)

### DataSet

DataSet 是分布式数据集合。DataSet 是Spark 1.6 中添加的一个新抽象，是DataFrame的一个扩展。它提供了RDD 的优势（强类型，使用强大的 lambda 函数的能力）以及SparkSQL 优化执行引擎的优点。DataSet 也可以使用功能性的转换（操作 map，flatMap，filter等等）

#### 创建DataSet

创建DataSet有两种方式：从存在的DataFrame进行转换，从存在的RDD进行转换

##### 1.从存在的DataFrame进行转换

DataFramez转换为DataSet时需要具备一个样例类

~~~scala
//读取json文件转换为DataFrame
val df = spark.read.json("C:/Users/JolyouLu/Downloads/user.json")
//创建样例类
case class User(age:Long,username:String)
//转换为DataSet
val ds = df.as[User]
//打印内容
ds.show
~~~

![image-20240516183532972](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516183532972.png)

##### 2.从存在的RDD进行转换

~~~scala
//样例类
case class User(username:String,age:Int)
//构建RDD，使用map转换成样例类的格式
val rdd = sc.makeRDD(List(("张三",30),("李四",25),("王五",30))).map(t => User(t._1,t._2))
//转换为DataSet
val ds = rdd.toDS
//打印内容
ds.show
~~~

![image-20240516184625880](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516184625880.png)

#### DataSet转为RDD

DataSet是对RDD的封装，所以可以直接获取内部rdd

~~~scala
//DataSet转RDD
ds.rdd
~~~

![image-20240516185001590](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516185001590.png)

#### DataSet转为DataFrame

~~~scala
//DataSet转换为DataFrame
ds.toDF
~~~

![image-20240516185131533](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516185131533.png)

### RDD、DF、DS三者关系

#### 共同点

1. RDD、DataFrame、DataSet 全都是 spark 平台下的分布式弹性数据集，为处理超大型数据提供便利
2. 三者都有惰性机制，在进行创建、转换，如 map 方法时，不会立即执行，只有在遇到Action 如 foreach 时，三者才会开始遍历运算
3. 三者有许多共同的函数，如 filter，排序等
4.  在对DataFrame 和Dataset 进行操作许多操作都需要这个包:import spark.implicits.（在创建好 SparkSession 对象后尽量直接导入）
5. 三者都会根据 Spark 的内存情况自动缓存运算，这样即使数据量很大，也不用担心会内存溢出
6.  三者都有 partition 的概念
7. DataFrame 和DataSet 均可使用模式匹配获取各个字段的值和类型

#### 区别

**RDD**

* 诞生于Spark1.0版本

* 一般和sparkMLlib同时使用
* RDD不支持sparkSQL操作

**DataFrame**

* 诞生于Spark1.3版本
* 与RDD和DataSet不同，DataFrame每行的类型固定为Row，每列的值无法直接访问，只能通过解析获取各列值
* DataFrame与DataSet一般不与sparkMLlib同时使用
* DataFrame与DataSet均支持SparkSQL的操作，比如select、groupBy之类，还能注册临时表/视图，进行sql语句操作
* DataFrame与DataSet支持一些特别方便的保存方式，比如保存成 csv，可以带上表头

**DataSet**

* 诞生于Spark1.6版本
* DataSet和DataFrame 拥有完全相同的成员函数，区别只是每一行的数据类型不同，DataFrame 其实就是DataSet 的一个特例 `type DataFrame = Dataset[Row]`
* DataFrame 也可以叫DataSet[Row],每一行的类型是 Row，不解析，每一行究竟有哪些字段，各个字段又是什么类型都无从得知，只能用上面提到的 getAS 方法或者共性中的第七条提到的模式匹配拿出特定字段。而DataSet中，每一行是什么类型是不一定的，在自定义了 case class 之后可以很自由的获得每一行的信息

![image-20240516175550385](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516175550385.png)

### SQL语法

SQL 语法风格是指我们查询数据的时候使用 SQL 语句来查询，这种风格的查询必须要有临时视图或者全局视图来辅助

~~~scala
//创建临时视图
df.createOrReplaceTempView("表名")
//执行SQL查询视图信息
val sqlRes = spark.sql("SQL语句")
~~~

![image-20240516185859591](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516185859591.png)

### DSL语法

DataFrame 提供一个特定领域语言(domain-specific language, DSL)去管理结构化的数据。可以在 Scala, Java, Python 和 R 中使用 DSL，使用 DSL 语法风格不必去创建临时视图了

**查看Schema信息**

~~~scala
//打印表结构
df.printSchema
~~~

![image-20240516190212382](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516190212382.png)

**只查询username列**

![image-20240516190424062](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516190424062.png)

**只查询对age列+1**

![image-20240516190600849](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516190600849.png)

**按age分组合计**

![image-20240516190808022](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240516190808022.png)

### IDE中使用SparkSQL

#### 三者互转

~~~scala
object Spark01_SparkSQL_Transform {
  def main(args: Array[String]): Unit = {
    //创建SparkSQL的运行环境
    val sparkSQL: SparkConf = new SparkConf().setMaster("local[*]").setAppName("SparkSQL")
    val spark = SparkSession.builder().config(sparkSQL).getOrCreate()
    val sc: SparkContext = spark.sparkContext
    import spark.implicits._ //将spark转换规则引入
    //DataFrame转换
    val df: DataFrame = spark.read.json("datas/user.json")
    println("======================RDD => DataFrame======================")
    val rdd2df: DataFrame = sc.makeRDD(List(("张三", 30), ("李四", 25), ("王五", 30))).toDF()
    rdd2df.show()
    println("======================DataFrame => RDD======================")
    val df2rdd: RDD[Row] = df.rdd
    df2rdd.foreach(println)
    println("======================DataFrame => DataSet======================")
    val df2ds: Dataset[User] = df.as[User]
    df2ds.show()

    //DataSet转换
    val ds: Dataset[User] = df.as[User]
    println("======================RDD => DataSet======================")
    val rdd2ds: Dataset[User] = sc.makeRDD(List(("张三", 30), ("李四", 25), ("王五", 30)))
      .map(item => User(item._2, item._1)).toDS()
    rdd2ds.show()
    println("======================DataSet => RDD======================")
    val ds2rdd: RDD[User] = ds.rdd
    ds2rdd.foreach(println)
    println("======================DataSet => DataFrame======================")
    val ds2df: DataFrame = ds.toDF()
    ds2df.show()
    
    //关闭环境
    spark.close()
  }
  case class User(age:Long,username:String)
}
~~~

#### 用户自定义函数（UDF）

Spark提供了UDF可以用户自定义函数并且注册到Spark中，在使用SQL查询时使用自定义的UDF

![image-20240517120057854](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517120057854.png)

#### 用户自定义聚合函数（UDAF）

通过一个案例来解释如何在Spark种使用UDAF，需求内容：需要对一组数据利用自定义符合函数的方式计算年龄的平均值

![image-20240517132430044](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517132430044.png)

##### 弱类型的UDAF（不推荐）

无类型通过继承`UserDefinedAggregateFunction`实现相应的方法，由于无类型约束容易导致编写过程中出错所以官方不推荐使用

![image-20240517132011683](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517132011683.png)

##### 强类型的UDAF（Spark3之前 推荐）

Spark3.x以前通过ds+DSL语法的方式实现强类型的控制

![image-20240517132628954](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517132628954.png)

##### 强类型的UDAF（Spark3之后 推荐）

Spark3.x之后可以直接将强类型UDAF添加到spark的udf中

![image-20240517132102836](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517132102836.png)

### 数据加载与保存

SparkSQL 提供了通用的保存数据和数据加载的方式。这里的通用指的是使用相同的API，根据不同的参数读取和保存不同格式的数据，SparkSQL 默认读取和保存的文件格式为 parquet

#### 数据加载

Spark通用的加载数据spark.read.load("文件路径")`如果文件类型不是parquet那么就会发生错误，不过Spark提供了很多实现可以用于读取常见数据

~~~scala
//spark.read默认提供了以下多种的数据格式读取
//例如读取json如下
spark.read.json("文件路径")

//spark.read.load也能读取其他数据类型，只需要在前面加上format
//format支持的数据类型有csv、jdbc、json、orc、parquet、textFile
spark.read.format("json").load("文件路径")
~~~



#### 保存数据

保存数据也是默认只支持parquet，spark也提供了其他实现

> 如果是win平台检查代码无误无法执行保存操作，请滑下去查看Win平台下保存数据的坑

~~~scala
//spark.write默认提供了以下多种的数据格式写出
//将文件保存位json数据
df.write.json("文件路径")

//spark.write.save也能写出其他数据类型，只需要在前面加上format
//format支持的数据类型有csv、jdbc、json、orc、parquet、textFile
df.write.format("json").save("文件路径")

//spark.write默认的保存模式是error(如果软件已经存在就抛异常)
//使用mode("append")指定为追加模式，跟多模式查看下面SaveMode
df.write.mode("append").json("文件路径")
~~~

**SaveMode**

| Scala/Java                      | Any Language     | Meaning                    |
| ------------------------------- | ---------------- | -------------------------- |
| SaveMode.ErrorIfExists(default) | "error"(default) | 如果文件已经存在则抛出异常 |
| SaveMode.Append                 | "append"         | 如果文件已经存在则追加     |
| SaveMode.Overwrite              | "overwrite"      | 如果文件已经存在则覆盖     |
| SaveMode.Ignore                 | "ignore"         | 如果文件已经存在则忽略     |

**Win平台下保存数据的坑**

假如在Win平台使用Spark保存数据看到存在一个Hadoop的异常现象，请检查本地的Hadoop环境是否正常，Hadoop的`winutils`是否配置完成，具体环境准备步骤查看该文章[Win10-Hadoop环境准备](./Win10-Hadoop环境准备)

![image-20240517145419374](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517145419374.png)

#### 数据加载读取案例

##### Parquet

Parquet是SparkSQL默认的数据源格式，Parquet 是一种能够有效存储嵌套数据的列式存储格式。

~~~scala
spark.read.load("xxx.parquet")
~~~

##### JSON

Spark中Json和传统Json文件是不一样的，Spark中的Json是每一行是Json数据，而不是整个文件是Json文件，整个文件是Json格式Spark是无法读取的

参考格式如下：

~~~json
{"username":"张三","age":30}
{"username":"李四","age":25}
{"username":"王五","age":30}
~~~
读取示例

~~~scala
spark.read.json("xxx.json")
~~~

##### CSV

CSV的读写方式需要设定一些参数

~~~scala
//format("csv") 指定文件类型
//option("sep", ",") 指定分隔符通常是","
//option("inferSchema","true")推导出Schema
//option("header", "true")是否读取表头
spark.read.format("csv").option("sep", ",").option("inferSchema","true").option("header", "true").load("data/user.csv")
~~~

##### Mysql

Spark SQL 可以通过 JDBC 从关系型数据库中读取数据的方式创建DataFrame，通过对 DataFrame 一系列的计算后，还可以将数据再写回关系型数据库中。

**spark-shell方式在启动时指定JDBC**

~~~shell
bin/spark-shell --jars mysql-connector-java-5.1.27-bin.jar
~~~

**IDE方式对Mysql操作**

首先需要引入依赖

~~~xml
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.0.32</version>
</dependency>
~~~

案例代码

![image-20240517152124643](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240517152124643.png)
