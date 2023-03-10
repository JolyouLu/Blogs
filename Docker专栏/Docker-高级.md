# Docker高级

## Docker网络认识

### 计算机网络

> 在了解Dcoker我们先了解一下什么是计算机网络，就拿我平时用的笔记本win10系统来说，在控制面板=>所有控制面板项=>网络连接，可以看到我们计算机中是拥有很多的络适配器，wifi、蓝牙、网口等模块，它们都有自己的网络适配器，那么这些网络适配器到底有什么用呢

![image-20210522121558931](./images/image-20210522121558931.png)

>在日常生活中我们无时无刻的都在运用着计算机网络，在每一个有宽带的家庭中都有一个属于自己的局域网，看如图组成这个局域网有几个主要的东西，路由器（理解为中国电信提供给你上网的”光猫“）、交换机（理解为你家中的WIFI），当我们所有的电脑都接入同一个WiFi时那么我们的电脑就可以直接相互访问了，因为它们都处于同一个交换机管理所以它们的IP地址网段都是同一个网段，什么是IP地址网段，这里就占时不做过多解释了，由于计算机网络展开来讲可以写成一个专栏，所以只是简单描述，如需了解请自行百度

![image-20210522124835372](./images/image-20210522123632105.png)

> 交换机：交换机主要是负责管理计算机之间的通讯，比如A开启了文件共享，B想去A获取共享的文件那么B会通过交换机去连接到A计算机并且访问它的共享文件
>
> 路由器：路由器主要负责帮助计算机访问外部网络，有了交换机那么我们家中的计算机都可以相互通讯了，但这不足已满足我们的需求，我们需要访问百度、淘宝网上冲浪，那么这时我们就去运营商办理了宽带，办理成功后师傅会上门安装宽带，安装时会有一个叫“光猫”的东西，师傅会将光猫也一同接入到交换机中，那么路由器也是局域网中的一个成员了，他可以与计算机相互通讯了，这是A计算机需要访问百度，A计算机会经过交换机，再经过路由器，完成访问百度服务器

### Docker0

> 经过简单的描述了计算机网络的特点，我们接下来我们来说是Docker的网络，通过上面的理解我们知道组成一个计算机网络最重要的东西是什么，那就是交换机，交换机可以让我们计算机相互通讯，Docker的网络也是如此
>
> 通过`ip addr`可以查看到Linux里网卡信息，可以看到当前Linux下网卡有一个叫`docker0`这个网卡这个就是docker的网络，可以理解为一个交换机，每当一个容器启动时，docker0就会分配一个同一个网段的IP地址给容器，这样容器之间就可以相互通讯了

![image-20210511202952099](./images/image-20210511202952099.png)

### 主机与容器通讯测试

> 执行如下命令

~~~shell
#启动一个tomcat容器
docker run -d -P --name tomcat01 tomcat
#查看启动了的容器ip地址
docker exec -it tomcat01 ip addr
~~~

![image-20210522130710269](./images/image-20210522130710269.png)

> 本机Linux直接使用`ping`命令`ping`刚刚启动了的tomcat容器发现通讯正常，那么标识我们Linux可以访问docker中的容器

![image-20210522130838153](./images/image-20210522130838153.png)

### 容器与容器通讯测试
> 执行如下命令

~~~shell
#启动一个tomcat容器
docker run -d -P --name tomcat02 tomcat
#利用tomcat02 ping tomcat01地址
docker exec -it tomcat02 ping 172.17.0.2
~~~

> 可以发现容器与容器之间是可以相互通讯的

![image-20210522134220268](./images/image-20210522134220268.png)

### Docker通讯原理

> 经过上面测试，我们知道我们的Linux主机与docker的容器之间，以及容器与容器之间都是相互可以通讯的，那么它们是通过什么技术相互通讯的呢，我们通过`ip addr`命令并且查看容器与主机的网卡信息
>
> 网络桥接特点，我们通过查看容器的网卡与主机网卡，我们可以发现容器`166: eth0@if167`网卡，在主机中也有一个`167: vetha2969b6@if166`与容器相反的网卡，这个就是“桥接模式”，可以理解为把容器中的网卡桥接到主机的docker0上，这样所有的容器都可以相互通讯了，并且让主机通过docker0可以访问所有容器

![image-20210522135603656](./images/image-20210522135603656.png)

> 大致网络图

![image-20210522141121155](./images/image-20210522141121155.png)

### --Link命令

> 经过上面的学习，相信大家对Docker网络以及了解了，也知道容器之间是如何通讯了，但是这引出了一个新问题，由于每次创建新容器时IP会被重新分配，那么比如我们部署的微服务项目，项目需要进行更新创建新容器时那么容器分配了新的IP给这个容器那么意味着如果其它容器需要与新容器通讯配置文件都要修改成为新容器的IP，利用--link通过容器名称去与其它容器通讯，而不是通过IP地址去与其它容器通讯就可以解决这个问题
>
> 我们再启动一个tomcat但是注意这次启动时添加了`--link`参数

~~~shell
#启动一个tomcat并且让当前tomcat与tomcat01连接
docker run -d -P --name tomcat03 --link tomcat01 tomcat
~~~

> tomcat03使用命令ping tomcat01，注意这次使用的不是ping+IP地址了，而是直接ping+容器名称，我们发现也可以通讯了

![image-20210522142336689](./images/image-20210522142336689.png)

**实现原理**

> 其实--Link实现原理就是修改，本地的hosts文件，我们查看tomcat03的hosts文件可以发现，tomcat01直接被写死只要访问tomcat01就是访问172.17.0.2的ip地址，所以这里还是有坑，如果我容器更新了tomcat01的ip地址并且不会改变

![image-20210522143425284](./images/image-20210522143425284.png)

## Docker自定义网络

> 由于Docker0是默认提供的网卡，所以局限性很多，最大的问题就是不能通过容器名去与其它容器通讯，而且使用`--link`看似能解决问题但是只是治标不治本，那么接下来说的就是Docker高级用法，自定义网络

### 查看Docker网络

> 配置docker网络需要使用到`docker network`命令，使用`docker network ls`可以查看到当前 

![image-20210522144043560](./images/image-20210522144043560.png)

### Docker网络模式

1. bridge（桥接模式，docker默认，自定义网络也用这种）
2. none（不配置网络）
3. host（与宿主机共享网络）
4. container（容器之间直接通讯，局限性较大）

### 创建一个网络

~~~shell
#driver:网络模式桥接模式
#subnet:子网（宿主机所在网段）192.168.0.0/16，可以分配192.168网段的255*255个IP地址
#gateway:网关192.168.0.1，就是你家路由器的地址，这个没写对容器无法访问互联网
#名字：mynet
docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet
~~~

> 使用`docker network ls`可以查看到刚刚创建的网络

![image-20210522145330766](./images/image-20210522145330766.png)

> 使用` docker network inspect mynet`可以查看创建的网络详细配置

![image-20210522145549561](./images/image-20210522145549561.png)

### 指定网络启动容器

> 分别启动2个tomcat并且指定使用刚刚创建的自定义网络启动

~~~shell
#启动一个tomcat指定网络是mynet网络
docker run -d -P --name tomcat01 --net mynet tomcat
docker run -d -P --name tomcat02 --net mynet tomcat
~~~

### 查看网络信息

> 通过使用`docker network inspect mynet`查看网络信息，可以看到刚刚启动的2个tomcat都在配置好的网络中，可以注意到IP地址就是我们自己分配的

![image-20210522150006387](./images/image-20210522150006387.png)

### 通过容器名相互通讯

> 使用了自定义的网络，我们不用使用`--link`命令直接使用`ping`命令通过容器名相互访问容器，自定义的网络可以解决Docker0存在的问题

![image-20210522150222705](./images/image-20210522150222705.png)

### 不同网络直接容器通讯

> 如图，我们现在docker拥有2个网络，一个是docker0一个的mynet，那么在mynet中有2个容器tomcat01与tomcat02，在docker0上有1个容器tomcat-docker0，在实际情况下我们可能会遇到一个问题，如何让docker0中的容器与mynet容器互通

![image-20210522151152123](./images/image-20210522151152123.png)

> 首先我们尝试直接通讯看看，可以看出直接通讯是无法通讯的

![image-20210522151730567](./images/image-20210522151730567.png)

> 由于docker0与mynet都是不一样的网络配置，所以容器直接是无法直接通讯的，使用`docker network connect`就能解决这个问题

~~~shell
#将tomcat-docker0容器接入到mynet网络中
docker network connect mynet tomcat-docker0
~~~

![image-20210522151855068](./images/image-20210522151855068.png)



