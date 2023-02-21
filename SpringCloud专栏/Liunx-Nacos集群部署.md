# Liunx-Nacos集群部署

> 我们安装1nginx+3Nacos+1mysql的方式完成Nacos的集群部署

![image-20220123220533395](./images/image-20220123220533395.png)

## 安装包下载

> 进入到Nacos的git仓库下载`nacos-server-1.3.0.tar.gz`安装包，并且将安装包拷贝到Liunx中，并且使用tar目录将安装包解压
>
> 下载地址：https://github.com/alibaba/nacos/releases/download/1.3.0/nacos-server-1.3.0.tar.gz

![image-20220123225919742](./images/image-20220123225919742.png)

## 配置数据源

### 数据库初始化

> 首先需要将3个Nacos使用相同的mysql数据源，需要准备一台mysql服务器，将nacos/conf文件夹中的`nacos-mysql.sql`脚本放入到mysql中执行

![image-20220123230656807](./images/image-20220123230656807.png)

> 执行完毕后可看到生成如下数据库与表结构

![image-20220123230605385](./images/image-20220123230605385.png)

### 修改配置

> 修改`nacos/conf/application.properties`配置，将如下内容注释去除，并且配置好mysql连接信息

![image-20220123233435338](./images/image-20220123233435338.png)

## 集群配置

> 集群信息配置`cluster.conf`文件中，将拷贝`cluster.conf.example`为`cluster.conf`

![image-20220123231742442](./images/image-20220123231742442.png)

> 将集群中的所有Nacos的IP:端口都写入到`cluster.conf`文件中

![image-20220123232005767](./images/image-20220123232005767.png)

## 修改Nginx

> 修改nginx增加如下配置，进行负载均衡

![image-20220123235255018](./images/image-20220123235255018.png)

## 集群启动

> 进入到nacos的bin目录，执行`startup.sh`脚本启动

![image-20220123232419009](./images/image-20220123232419009.png)

**启动成功后登录到nacos在集群信息中看到nacso列表信息表示成功**

![image-20220124002111598](./images/image-20220124002111598.png)