# Docker概述与安装

## Docker的出现

Dockeer的出现是解决什么问题呢，在传统开发中，一个产品的开发到上线，最少会经历2套环境，1、开发环境，2、应用环境，程序员在自己本地搭建开发环境运行测试，运维拿到测试完毕后的项目部署上应用环境，这从之中就会经常出现一个问题"怎么我本地可以运行，线上无法运行"，而且由于现在项目环境要求越来越多，发布一个项目需要为一个项目搭建很多环境，如发布一个jar包需要（jdk、myslq、redis、es等），这中间就产生一个问题，能不能jar直接带上需要运行的环境一同打包好给运维，直接运行就行了呢，Docker的诞生就是解决这个问题

## Docker的核心思想

通过Docker的图标我们可以看到，一个小鲸鱼承载着很多个集装箱，docker的思想源于就是集装箱，它将来所有东西打包装箱，每一个箱子是相互隔离，这样就充分解决了环境冲突，端口冲突等问题

1. 应用更快速的部署交付
2. 更加便捷的升级和扩缩容
3. 更加简单的系统运维
4. 更搞笑的计算机资源利用

## Docker文档

官方文档：https://docs.docker.com/（超详细的使用说明）

仓库地址：https://hub.docker.com/（与git一样可以发布与下载自己的镜像）

## Docker组成

> 引用官方文档中的一个图片，可以看到一个docker由3部分组成
>
> Client：客户端可以使用docker命令操作docker服务
>
> Docker-Host：docker服务，这也是docker核心里面包含了镜像（Images）、容器（Container）
>
> 1. 镜像（Images）：dockre镜像就好比一个模板，可以通过这个模板来创建容器服务
> 2. 容器（container）：通过镜像创建后的一个一个的容器都是一个独立的微型Liunx系统环境，我们可以控制这些容器，启动、停止、删除等
>
> Registery：docker仓库，用于管理保存我们自己的镜像的仓库，仓库分共用于私有仓库，公有的如阿里云仓库，私有那就是自己的了 

![image-20210502165606217](.\images\image-20210502165606217.png)

## Docker的安装

[Liunx-安装Docker]: https://blog.csdn.net/weixin_44642403/article/details/114265278

## Docker的run流程

![image-20210502182427127](.\images\image-20210502182427127.png)

## Docker是如何工作的

Docker是一个CS结构的系统，Docker是守护进程运行在主机上，通过Socker与客户端交互，DockerServer接收到Docker-Client的指令，就会执行这个命令

![image-20210502183103606](.\images\image-20210502183103606.png)