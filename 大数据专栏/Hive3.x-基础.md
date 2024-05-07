# Hive3.x-基础

## Hive简介

Hive是由Feacebook开源，基于Hadoop的数据仓库根据，它可以将结构化的数据文件映射为一张表，并提供类似SQL查询功能，Hive是为了简化Hadoop数据统计时需要编写MapReduce问题，Hive可以通过编写SQL的形式然后直接在Hadoo上执行，Hive的本质上其实是一个Hadoop的客户端，它会将用户编写好的SQL转换成为MapReduce程序提交给Yarn执行，也可以集成Spark或者Tez这样的计算框架

![image-20240407103535789](E:\one-drive-data\OneDrive\CSDN\大数据专栏\images\image-20240407103535789.png)

**主要组件**

HiveClient：hive客户端可以通过CLI(本地机器)和JDBC协议操作，通常情况远程连接通过JDBC方式

Metastore：通过hiveClien创建的表结构保存在Metastore中，表结构信息是用于描述hdfs中文件的名称、行、列信息，使得hdfs中的文件可以有类似关系型数据库的ddl文件

Drive：接受hive客户端提交的任务，并将sql翻译成MapReduce程序提交给Yarn执行

1. 解析器（SQLParser）：将SQL字符串转换成抽象的语法树（AST）
2. 语义分析（Semantic Analyzer）：将AST进一步划分为QeuryBlock
3. 逻辑计划生成器（Logical Plan Gen）：将语法树生成逻辑计划
4. 逻辑优化器（Logical Optimizer）：对逻辑计划进行优化
5. 物理计划生成器（Physical Plan Gen）：根据优化后的逻辑计划生成物理计划
6. 物理优化器（Physical Optimizer）：对物理计划进行优化
7. 执行器（Execution）：执行该计划，得到查询结果并返回给客户端

## Hive安装

Hive是基于Hadoop基础上的数据仓库，所以需要具备Hadoop环境才能安装，[Haddop3.x-基础(部署)](Haddop3.x-基础(部署))