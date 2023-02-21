# Liunx-Kafka集群部署

### 前置环境

[Zookeeper-快速入门(服务安装、单机部署、集群部署)]([(7条消息) Zookeeper-快速入门(服务安装、单机部署、集群部署)_JolyouLu的博客-CSDN博客](https://blog.csdn.net/weixin_44642403/article/details/115985332))

#### 安装包下载

> 进入到[Apache Kafka](https://kafka.apache.org/)官方，下载Kafka安装包

![image-20220702161749683](./images/image-20220702161749683.png)

#### 解压压缩包

~~~shell
# 解压压缩包
tar -zxvf kafka_2.12-3.0.0.tgz
~~~

### 集群部署

#### 修改配置文件

~~~shell
#进入到config目录
cd kafka/config
#修改server配置文件
vim server.properties
~~~

**broker.id**

> broker.id是kafka的唯一标识，在集群环境下不能重复

![image-20220702164527040](./images/image-20220702164527040.png)

**listeners**

> listeners修改监听的域名与端口

![image-20220702223320380](./images/image-20220702223320380.png)

**log.dirs**

> log.dirs是Kafka数据文件，默认是保存在temp目录下的会丢失，需要修改路径

![image-20220702164651330](./images/image-20220702164651330.png)

**zookeeper.connect**

> zookeeper.connect修改Kafka连接Zookeeper的地址，`IP,IP,IP/kafka`后面的`/kafka`是指定保存到Zookeeper的kafka目录下，这样方便管理

![image-20220702164823203](./images/image-20220702164823203.png)

#### 启动项目

~~~shell
#进入到kafka的bin目录
cd /data/software/kafka/bin
#启动Kafka server 指定配置文件启动
./bin/kafka-server-start.sh -daemon config/server.properties
~~~

> 使用jps看到Kafka表示启动成功

![image-20220702164922548](./images/image-20220702164922548.png)

> Zookeeper中也能看到3台brokers

![image-20220702165117807](./images/image-20220702165117807.png)
