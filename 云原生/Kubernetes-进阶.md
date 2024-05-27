# Kubernetes-进阶

## Pod详解

> 每个Pod中都可以包含一个或多个容器，这些容器可以分两类
>
> * 用户程序所在容器，数量用户决定
> * Pause容器，这是每个Pod都会有的一个根容器，它的作用有两个
>   1. 可以以它为依据，评估整个Pod的健康状态
>   2. 可以在根容器上设置Ip地址，其它容器都可以通过此IP(Pod IP)，就行Pod内部的网络通信`Pod与Pod之间的网络通信是通过虚拟二层网络技术实现的，我们当前的环境用的是Flannel`

![image-20230623154245358](./images/image-20230623154245358-16875061675249.png)

### Pod定义

> 以下是Pod的yaml中可配置的信息
>
> 使用命令`kubectl explain pod`可以看到pod的可配置的属性<br>使用命令`kubectl explain pod.metadata`可以看到pod.metadata的可配置的属性

~~~yaml
apiVersion: v1     #必选，版本号，例如v1
kind: Pod       　 #必选，资源类型，例如 Pod
metadata:       　 #必选，元数据
  name: string     #必选，Pod名称
  namespace: string  #Pod所属的命名空间,默认为"default"
  labels:       　　  #自定义标签列表
    - name: string      　          
spec:  #必选，Pod中容器的详细定义
  containers:  #必选，Pod中容器列表
  - name: string   #必选，容器名称
    image: string  #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
    command: [string]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      #容器的启动命令参数列表
    workingDir: string  #容器的工作目录
    volumeMounts:       #挂载到容器内部的存储卷配置
    - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean #是否为只读模式
    ports: #需要暴露的端口库号列表
    - name: string        #端口的名称
      containerPort: int  #容器需要监听的端口号
      hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string    #端口协议，支持TCP和UDP，默认TCP
    env:   #容器运行前需设置的环境变量列表
    - name: string  #环境变量名称
      value: string #环境变量的值
    resources: #资源限制和请求的设置
      limits:  #资源限制的设置
        cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests: #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string #内存请求,容器启动的初始可用数量
    lifecycle: #生命周期钩子
        postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
        preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
    livenessProbe:  #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
      exec:       　 #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
  restartPolicy: [Always | Never | OnFailure]  #Pod的重启策略
  nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
  nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
  - name: string
  hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes:   #在该pod上定义共享存储卷列表
  - name: string    #共享存储卷名称 （volumes类型有很多种）
    emptyDir: {}       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
    hostPath: string   #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
      path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
    secret:       　　　#类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
      scretname: string  
      items:     
      - key: string
        path: string
    configMap:         #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
      name: string
      items:
      - key: string
        path: string
~~~

### Pod配置

**Pod一级属性**

| 属性名              | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| apiVersion <string> | 版本，由kubernetes内部定义，版本号必须可以用 kubectl api-versions 查询到 |
| kind <string>       | 类型，由kubernetes内部定义，版本号必须可以用 kubectl api-resources 查询到 |
| metadata <Object>   | 元数据，主要是资源标识和说明，常用的有name、namespace、labels等 |
| spec <Object>       | 描述，这是配置中最重要的一部分，里面是对各种资源配置的详细描述 |
| status <Object>     | 状态信息，里面的内容不需要定义，由kubernetes自动生成         |

**spec属性(重点)**

| 属性名                 | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| containers <[]Object>  | 容器列表，用于定义容器的详细信息                             |
| nodeName <String>      | 根据nodeName的值将pod调度到指定的Node节点上                  |
| nodeSelector <map[]>   | 根据NodeSelector中定义的信息选择将该Pod调度到包含这些label的Node 上 |
| hostNetwork <boolean>  | 是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络 |
| volumes <[]Object>     | 存储卷，用于定义Pod上面挂在的存储信息                        |
| restartPolicy <string> | 重启策略，表示Pod在遇到故障的时候的处理策略                  |

#### 基本配置

> 使用`kubectl explain pod.spec.containers`命令查看可配置项

```shell
KIND:     Pod
VERSION:  v1
RESOURCE: containers <[]Object>   # 数组，代表可以有多个容器
FIELDS:
   name  <string>     # 容器名称
   image <string>     # 容器需要的镜像地址
   imagePullPolicy  <string> # 镜像拉取策略 
   command  <[]string> # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
   args     <[]string> # 容器的启动命令需要的参数列表
   env      <[]Object> # 容器环境变量的配置
   ports    <[]Object>     # 容器需要暴露的端口号列表
   resources <Object>      # 资源限制和资源请求的设置
```

**案例**

> 创建pod-base.yaml文件，内容如下：
>
> 使用`kubectl apply/delete`启动(更新)/删除 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: dev
  labels:
    user: test
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
```

> 上面定义了一个比较简单Pod的配置，里面有两个容器：
>- nginx：用1.17.1版本的nginx镜像创建，（nginx是一个轻量级web容器）
>- busybox：用1.30版本的busybox镜像创建，（busybox是一个小巧的linux命令集合）

#### 镜像拉取

> 创建pod-imagepullpolicy.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-imagepullpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    imagePullPolicy: Never # 用于设置镜像拉取策略
  - name: busybox
    image: busybox:1.30
```

imagePullPolicy，用于设置镜像拉取策略，kubernetes支持配置三种拉取策略：

- Always：总是从远程仓库拉取镜像（一直远程下载）
- IfNotPresent：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像（本地有就本地 本地没远程下载）
- Never：只使用本地镜像，从不去远程仓库拉取，本地没有就报错 （一直使用本地）

> 默认值说明：
>
> 如果镜像tag为具体版本号， 默认策略是：IfNotPresent
>
> 如果镜像tag为：latest（最终版本） ，默认策略是always

#### 启动命令

在说启动命令之前，先说说在前面执行的2个脚本可以冲READY中看到只启动了一个，通过使用`kubectl describe pod <pod命令> -n dev`命令可以看到busybox启动不了，这是因为busybox启动后没有进程在运行所以启动后就自动关闭了，那么如何让容器启动后不会自动关闭呢，写过Dockerfile的小伙伴应该就清楚只需要在后面执行一个`CMD`命令即可

![image-20230623170451553](./images/image-20230623170451553-16875110933441.png)

> 创建pod-command.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done;"] #执行cmd命令使得容器有进程在运行
```

> command，用于在pod中的容器初始化完毕之后运行一个命令。
>
> "/bin/sh","-c", 使用sh执行命令
>
> touch /tmp/hello.txt; 创建一个/tmp/hello.txt 文件
>
> while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done; 每隔3秒向文件中写入当前时间

![image-20230623171325833](./images/image-20230623171325833-16875116072093.png)

**进入容器测试**

~~~shell
# 进入pod中的busybox容器，查看文件内容
# 补充一个命令: kubectl exec  pod名称 -n 命名空间 -it -c 容器名称 /bin/sh  在容器内部执行命令
# 使用这个命令就可以进入某个容器的内部，然后进行相关操作了
# 比如，可以查看txt文件的内容
[root@k8s-master01 pod]# kubectl exec pod-command -n dev -it -c busybox /bin/sh
#进入容器后执行
tail -f /tmp/hello.txt
~~~

> 特别说明：
>     通过上面发现command已经可以完成启动命令和传递参数的功能，为什么这里还要提供一个args选项，用于传递参数呢?这其实跟docker有点关系，kubernetes中的command、args两项其实是实现覆盖Dockerfile中ENTRYPOINT的功能。
>  1 如果command和args均没有写，那么用Dockerfile的配置。
>  2 如果command写了，但args没有写，那么Dockerfile默认的配置会被忽略，执行输入的command
>  3 如果command没写，但args写了，那么Dockerfile中配置的ENTRYPOINT的命令会被执行，使用当前args的参数
>  4 如果command和args都写了，那么Dockerfile的配置被忽略，执行command并追加上args参数

#### 环境变量

> 创建pod-env.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do /bin/echo $(date +%T);sleep 60; done;"]
    env: # 设置环境变量列表
    - name: "username"
      value: "admin"
    - name: "password"
      value: "123456"
```

**进入容器测试**

```shell
# 进入容器，输出环境变量
kubectl exec pod-env -n dev -c busybox -it /bin/sh
#进入容器后执行
echo $username
echo $password
```

#### 端口设置

> 使用`kubectl explain pod.spec.containers.ports`命令查看可配置项

~~~shell
KIND:     Pod
VERSION:  v1
RESOURCE: ports <[]Object>
FIELDS:
   name         <string>  # 端口名称，如果指定，必须保证name在pod中是唯一的		
   containerPort<integer> # 容器要监听的端口(0<x<65536)
   hostPort     <integer> # 容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本(一般省略) 
   hostIP       <string>  # 要将外部端口绑定到的主机IP(一般省略)
   protocol     <string>  # 端口协议。必须是UDP、TCP或SCTP。默认为“TCP”。
~~~

> 创建pod-ports.yaml，内容如下：

~~~yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: # 设置容器暴露的端口列表
    - name: nginx-port
      containerPort: 80
      protocol: TCP
~~~

> 访问容器中的程序需要使用的是`Podip:containerPort`

#### 资源配额

容器中的程序要运行，肯定是要占用一定资源的，比如cpu和内存等，如果不对某个容器的资源做限制，那么它就可能吃掉大量资源，导致其它容器无法运行。针对这种情况，kubernetes提供了对内存和cpu的资源进行配额的机制，这种机制主要通过resources选项实现，他有两个子选项：

- limits：用于限制运行时容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启
- requests ：用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动

> 创建pod-resources.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    resources: # 资源配额
      limits:  # 限制资源（上限）
        cpu: "2" # CPU限制，单位是core数
        memory: "10Gi" # 内存限制
      requests: # 请求资源（下限）
        cpu: "1"  # CPU限制，单位是core数
        memory: "10Mi"  # 内存限制
```

### Pod生命周期

现在以一般的Pod对象从创建至终的这段实际范围称为pod的生命周期，它主要包含以下的过程

1. pod创建过程
2. 运行初始化容器(init container)过程
3. 运行主容器(main container)过程
   * 容器启动后钩子(post start)、容器终止前钩子(pre stop)
   * 容器的存活性探测(livenes porbe)、就绪性探测(readiness probe
4. pod终止

![image-20230630132854744](./images/image-20230630132854744-16881029360433.png)

在整个的生命周期中pod会出现5种状态

| 状态              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| 挂起（Pending）   | apiService已经创建了pod资源对象，但它尚未被调度卧槽或者仍处于下载镜像的过程中 |
| 运行种（Running） | pod已经被调度至某个节点，并且所有容器都已经被kubelet创建完成 |
| 成功（Succeeded） | pod中的所有容器都已经成功终止并且不会被重启                  |
| 失败（Failed）    | 所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态 |
| 未知（Unknown）   | apiserver无法正常获取到pod对象的状态信息，通常由网络通讯失败所导致 |

#### 创建过程

1. 用户通过kubectl或者其它api客户端提交需要创建的pod信息给apiServer
2. apiServer开始生成pod对象的信息，并将信息存入etcd，然后返回确认信息到客户端
3. apiServer开始反映etcd中的pod对象变化，其它组件使用watch机制来跟踪检查apiServer上变动
4. cheduler发现有新的pod对象要创建，开始为Pod分配主机并将结果更新至apiServer
5. node节点上的kubelet发现有pod调度过来，尝试调用docker启动容器，并将结果送会至apiServer
6. apiServer将接收到的pod状态信息存入etcd中

![image-20230630140726557](./images/image-20230630140726557-16881052479495.png)

#### 终止过程

1. 用户向apiServer发生删除pod对象命令
2. apiServer中的pod对象信息会随时间的推移而更新，宽限期内(默认30s)，pod被视为dead
3. 将pod标记为terminating状态
4. kubelet在监控到pod对象转为terminating状态的同时启动pod关闭过程
5. 端点控制器监控到pod对象的关闭行为是将其所有匹配到此端点的service资源从端点列表中移除
6. 如果当前pod对象定义了preStop钩子处理器，则在标记为terminating后即同步的方式启动执行
7. pod对象中的容器进程收到停止信号
8. 宽限期结束后，若pod中还存在运行的进程，那么pod对象会收到立刻终止的信号
9. kubelet请求apiserver将此pod资源宽限期设置为0从而完成删除操作，此时pod对应用户已经不可见

#### 初始化容器

初始化容器是在pod的主容器启动之前要运行的容器，主要是做一些主容器的前置工作，它有2大特性

1. 初始化容器必须完成直到结束，若某个初始化容器运行失败，那么kubernetes需要重启它直到成功完成
2. 初始化容器必须按照定义的顺序执行，当且仅当前一个成功后，后面的一个才能运行

初始化容器有很多应用场景，下面是常见的场景

1. 提供主容器镜像中不具备的工具程序或自定义代码
2. 初始化容器要先于运用容器串行启动并运行完成，因此可以作为后续容器启动的依赖条件是否满足判断

**案例**

> 以下设计一个案例，假设主容器运行nginx，但是在nginx运行之前需要确保能够先连接上mysql和redis服务器
>
> 假如：mysql 192.168.10.101 redis 192.168.10.102
>
> 创建pod-initcontainer.yaml，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-initcontainer
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
  initContainers:
  - name: test-mysql
    image: busybox:1.30
    command: ['sh', '-c', 'until ping 192.168.10.101 -c 1 ; do echo waiting for mysql...; sleep 2; done;']
  - name: test-redis
    image: busybox:1.30
    command: ['sh', '-c', 'until ping 192.168.10.102 -c 1 ; do echo waiting for reids...; sleep 2; done;']
```

> 执行命令测试

```shell
# 创建pod
kubectl create -f pod-initcontainer.yaml
# 查看pod状态
# 发现pod卡在启动第一个初始化容器过程中，后面的容器不会运行
kubectl describe pod  pod-initcontainer -n dev
# 动态查看pod
kubectl get pods pod-initcontainer -n dev -w
# 接下来新开一个shell，为当前服务器新增两个ip，观察pod的变化
[root@k8s-master01 ~]# ifconfig ens33:1 192.168.5.14 netmask 255.255.255.0 up
[root@k8s-master01 ~]# ifconfig ens33:2 192.168.5.15 netmask 255.255.255.0 up
```

![image-20230630144527228](./images/image-20230630144527228-16881075285497.png)

#### 钩子函数

Kubernetes在主容器的启动之后和停止之前提供两个钩子函数

* post start：容器创建之后执行，如果失败了会重启容器
* pre stop：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作

**钩子处理支持以下3种方式**

##### Exec命令

> 在容器内执行一次命令

  ```yaml
    lifecycle:
      postStart: 
        exec:
          command:
          - cat
          - /tmp/healthy
  ```

##### TCPSocket

> 在当前容器尝试访问指定的socket

  ```yaml
    lifecycle:
      postStart:
        tcpSocket:
          port: 8080
  ```

##### HTTPGet

> 在当前容器中向某url发起http请求

  ```yaml
    lifecycle:
      postStart:
        httpGet:
          path: / #URI地址
          port: 80 #端口号
          host: 192.168.5.3 #主机地址
          scheme: HTTP #支持的协议，http或者https
  ```

**案例**

> 创建pod-hook-exec.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hook-exec
  namespace: dev
spec:
  containers:
  - name: main-container
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    lifecycle:
      postStart: 
        exec: # 在容器启动的时候执行一个命令，修改掉nginx的默认首页内容
          command: ["/bin/sh", "-c", "echo postStart... > /usr/share/nginx/html/index.html"]
      preStop:
        exec: # 在容器停止之前停止nginx服务
          command: ["/usr/sbin/nginx","-s","quit"]
```

> 执行命令测试

```shell
# 创建pod
kubectl create -f pod-hook-exec.yaml
# 查看pod
kubectl get pods  pod-hook-exec -n dev -o wide  
# 访问
curl 10.244.22.48
```

#### 容器探测

容器探测用于检测容器中的应用是否正常工作，是保障业务可用性的一种传统机制，如果结果探测实例的状态不符合预期，那么kubernetes就会把该问题实例"摘除"，不承担业务流量kubernetes提供了两种探针来实现容器探测

1. liveness probes：存活性探针，用于检测应用实例当前是否处于正常运行状态，如果不是，k8s会重启容器

2. readiness probes：就绪性探针，用于检测应用实例当前是否可以接收请求，如果不能，k8s不会转发流量

> livenessProbe 决定是否重启容器，readinessProbe 决定是否将请求转发给容器

**探针3种方式**

```shell
#查看探测可配置参数
kubectl explain pod.spec.containers.livenessProbe
FIELDS:
   exec <Object>  
   tcpSocket    <Object>
   httpGet      <Object>
   initialDelaySeconds  <integer>  # 容器启动后等待多少秒执行第一次探测
   timeoutSeconds       <integer>  # 探测超时时间。默认1秒，最小1秒
   periodSeconds        <integer>  # 执行探测的频率。默认是10秒，最小1秒
   failureThreshold     <integer>  # 连续探测失败多少次才被认定为失败。默认是3。最小值是1
   successThreshold     <integer>  # 连续探测成功多少次才被认定为成功。默认是1
```

##### Exec命令

> 在容器内执行一次命令，如果命令执行的退出码为0，则认为程序正常，否则不正常

  ```yaml
    livenessProbe:
      postStart: 
        exec:
          command:
          - cat
          - /tmp/healthy
  ```

##### TCPSocket

> 将会尝试访问一个用户容器的端口，如果能够建立这条连接，则认为程序正常，否则不正常

  ```yaml
    livenessProbe:
      postStart:
        tcpSocket:
          port: 8080
  ```

##### HTTPGet

> 调用容器内Web应用的URL，如果返回的状态码在200和399之间，则认为程序正常，否则不正常

  ```yaml
    livenessProbe:
      postStart:
        httpGet:
          path: / #URI地址
          port: 80 #端口号
          host: 192.168.5.3 #主机地址
          scheme: HTTP #支持的协议，http或者https
  ```

**案例**

> 创建pod-liveness-exec.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports: 
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      exec:
        command: ["/bin/cat","/tmp/hello.txt"] # 执行一个查看文件的命令
```

> 执行命令测试

```shell
# 创建Pod
kubectl create -f pod-liveness-exec.yaml
# 查看Pod详情
kubectl describe pods pod-liveness-exec -n dev
# 观察上面的信息就会发现nginx容器启动之后就进行了健康检查
# 检查失败之后，容器被kill掉，然后尝试进行重启（这是重启策略的作用，后面讲解）
# 稍等一会之后，再观察pod信息，就可以看到RESTARTS不再是0，而是一直增长
kubectl get pods pod-liveness-exec -n dev
```

> 由于nginx不存在hello.txt文件可以看到pod在不断重启

![image-20230630151734131](./images/image-20230630151734131-16881094553659-168810945655611.png)

#### 重启策略

在上一节中，一旦容器探测出现了问题，kubernetes就会对容器所在的Pod进行重启，其实这是由pod的重启策略决定的，pod的重启策略有 3 种，分别如下：

- Always ：容器失效时，自动重启该容器，这也是默认值。
- OnFailure ： 容器终止运行且退出码不为0时重启
- Never ： 不论状态为何，都不重启该容器

重启策略适用于pod对象中的所有容器，首次需要重启的容器，将在其需要时立即进行重启，随后再次需要重启的操作将由kubelet延迟一段时间后进行，且反复的重启操作的延迟时长以此为10s、20s、40s、80s、160s和300s，300s是最大延迟时长。

> 创建pod-restartpolicy.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - name: nginx-port
      containerPort: 80
    livenessProbe:
      httpGet:
        scheme: HTTP
        port: 80
        path: /hello
  restartPolicy: Never # 设置重启策略为Never
```

```shell
# 创建Pod
kubectl create -f pod-restartpolicy.yaml
# 查看Pod详情，发现nginx容器失败
kubectl  describe pods pod-restartpolicy  -n dev
# 多等一会，再观察pod的重启次数，发现一直是0，并未重启   
kubectl  get pods pod-restartpolicy -n dev
```

### Pod调度

默认情况下一个Pod在那个Node节点上运行是由Scheduler组件采用相应的算法计算出来的，这个过程是不受人工控制的但是在实际使用中这并不满足需求，因为很多情况下我们想控制Pod部署在某个节点上，这就需要了解和使用Pod的调度规则，Kubernetes提供四大类的调度方式

1. 自动调度(默认)：运行在哪个节点上完全由Scheduler经过计算得出
2. 定向调度：NodeName、NodeSelector
3. 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity
4. 污点(容忍)调度：Taints、Toleration

#### 定向调度

定向调度是一种强制性调度，指的是利用在pod上声明nodeName或者nodeSelector，以此将Pod调度到期望的node节点上

> 注意，由于调度是强制性的，意味着即使要调度的目标Node不存在，也会进行调度，只不过pod运行失败

##### nodeName

NodeName用于强制约束将Pod调度到指定的Name的Node节点上，这种方式直接通过Scheduler的调度逻辑，直接将Pod调度到指定名称的节点

> 创建一个pod-nodename.yaml文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: node1 # 指定调度到node1节点上
```

> 执行命令测试

```shell
#创建Pod
kubectl create -f pod-nodename.yaml
#查看Pod调度到NODE属性，确实是调度到了node1节点上
kubectl get pods pod-nodename -n dev -o wide   
```

##### nodeSelector

NodeSelector用于强制约束将Pod调度到指定标签的node节点上，它是通过kubernetes的label-selector机制实现的，在pod创建之前会由scheduler使用MatchNodeSelector调度策略进行label匹配，找到目标node然后将pod调度到目标node节点上

> 首先需要为node节点添加标签

~~~shell
kubectl label nodes node1 nodeenv=pro
kubectl label nodes node2 nodeenv=test
~~~

> 创建一个pod-nodeselector.yaml文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeSelector: 
    nodeenv: pro # 指定调度到具有nodeenv=pro标签的节点上
```

> 执行命令测试

~~~shell
#创建Pod
kubectl create -f pod-nodeselector.yaml
#查看Pod调度到NODE属性，确实是调度到了node1节点上
kubectl get pods pod-nodeselector -n dev -o wide
~~~

#### 亲和性调度

在上面的定向调度使用过程中发现存在一个缺点，假如调度的目标不存在那么pod就不会启动成功，基于这个问题Kubernetes还提供了亲和性调度(Affinity)，它在NodeSelector的基础之上进行了扩展，可以通过配置的形式实现优先满足条件的Node进行调度，如果没有也可以调度到不满足条件的Node上，使得调度更加灵活

Affinit主要分3类

1. nodeAffinity(node亲和性)：以node为目标，解决pod可以调度到那些node的问题
2. podAffinity(pod亲和性)：以pod为目标，解决pod可以和哪些已存在的pod部署在同一个拓扑域中的问题
3. podAntiAffinity(pod反亲和性)：以pod为目标，解决pod不能和那些已存在pod部署在同一个拓扑域种的问题

> 关于亲和性(反亲和性)使用场景的说明：
>
> **亲和性**：如果两个应用频繁交互，那就有必要利用亲和性让两个应用的尽可能的靠近，这样可以减少因网络通信而带来的性能损耗
>
> **反亲和性**：当应用的采用多副本部署时，有必要采用反亲和性让各个应用实例打散分布在各个node上，这样可以提高服务的高可用性

##### nodeAffinity

> 首先来看一下`NodeAffinity`的可配置项

```shell
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  节点选择列表
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution 优先调度到满足指定的规则的Node，相当于软限制 (倾向)
    preference   一个节点选择器项，与相应的权重相关联
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt
	weight 倾向权重，在范围1-100。
```

```shell
关系符的使用说明:

- matchExpressions:
  - key: nodeenv              # 匹配存在标签的key为nodeenv的节点
    operator: Exists
  - key: nodeenv              # 匹配标签的key为nodeenv,且value是"xxx"或"yyy"的节点
    operator: In
    values: ["xxx","yyy"]
  - key: nodeenv              # 匹配标签的key为nodeenv,且value大于"xxx"的节点
    operator: Gt
    values: "xxx"
```

###### 硬限制

> 使用`requiredDuringSchedulingIgnoredDuringExecution`
>
> 创建pod-nodeaffinity-required.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        nodeSelectorTerms:
        - matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

> 执行命令测试

```shell
# 创建pod
kubectl create -f pod-nodeaffinity-required.yaml
# 查看pod状态 （运行失败）
kubectl get pods pod-nodeaffinity-required -n dev -o wide
# 查看Pod的详情
# 发现调度失败，提示node选择失败
kubectl describe pod pod-nodeaffinity-required -n dev

#接下来，停止pod
kubectl delete -f pod-nodeaffinity-required.yaml
# 修改文件，将values: ["xxx","yyy"]------> ["pro","yyy"]
vim pod-nodeaffinity-required.yaml
# 再次启动
kubectl create -f pod-nodeaffinity-required.yaml
# 此时查看，发现调度成功，已经将pod调度到了node1上
kubectl get pods pod-nodeaffinity-required -n dev -o wide
```

###### 软限制

> 使用`preferredDuringSchedulingIgnoredDuringExecution`
>
> 创建pod-nodeaffinity-preferred.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-preferred
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    nodeAffinity: #设置node亲和性
      preferredDuringSchedulingIgnoredDuringExecution: # 软限制
      - weight: 1
        preference:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签(当前环境没有)
          - key: nodeenv
            operator: In
            values: ["xxx","yyy"]
```

> 执行命令测试

```shell
# 创建pod
kubectl create -f pod-nodeaffinity-preferred.yaml
# 查看pod状态 （运行成功）
kubectl get pod pod-nodeaffinity-preferred -n dev
```

> NodeAffinity规则设置的注意事项：
>     1 如果同时定义了nodeSelector和nodeAffinity，那么必须两个条件都得到满足，Pod才能运行在指定的Node上
>     2 如果nodeAffinity指定了多个nodeSelectorTerms，那么只需要其中一个能够匹配成功即可
>     3 如果一个nodeSelectorTerms中有多个matchExpressions ，则一个节点必须满足所有的才能匹配成功
>     4 如果一个pod所在的Node在Pod运行期间其标签发生了改变，不再符合该Pod的节点亲和性需求，则系统将忽略此变化

##### podAffinity

> 首先来看一下`podAffinity`的可配置项

```shell
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  硬限制
    namespaces       指定参照pod的namespace
    topologyKey      指定调度作用域
    labelSelector    标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels    指多个matchExpressions映射的内容
  preferredDuringSchedulingIgnoredDuringExecution 软限制
    podAffinityTerm  选项
      namespaces      
      topologyKey
      labelSelector
        matchExpressions  
          key    键
          values 值
          operator
        matchLabels 
    weight 倾向权重，在范围1-100
```

> topologyKey用于指定调度时作用域,例如:
>     如果指定为kubernetes.io/hostname，那就是以Node节点为区分范围
> 	如果指定为beta.kubernetes.io/os,则以Node节点的操作系统类型来区分

###### 硬限制

> 首先创建一个参照Pod，pod-podaffinity-target.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-target
  namespace: dev
  labels:
    podenv: pro #设置标签
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: node1 # 将目标pod名确指定到node1上
```

> 执行命令启动

```shell
# 启动目标pod
kubectl create -f pod-podaffinity-target.yaml
# 查看pod状况
kubectl get pods  pod-podaffinity-target -n dev
```

> 创建pod-podaffinity-required.yaml
>
> 新Pod必须要与拥有标签nodeenv=xxx或者nodeenv=yyy的pod在同一Node上，显然现在没有这样pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    podAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配env的值在["xxx","yyy"]中的标签
          - key: podenv
            operator: In
            values: ["xxx","yyy"]
        topologyKey: kubernetes.io/hostname
```

> 执行命令测试

```shell
# 启动pod
kubectl create -f pod-podaffinity-required.yaml
# 查看pod状态，发现未运行
kubectl get pods pod-podaffinity-required -n dev
# 查看详细信息
kubectl describe pods pod-podaffinity-required  -n dev

# 接下来修改  values: ["xxx","yyy"]----->values:["pro","yyy"]
# 意思是：新Pod必须要与拥有标签nodeenv=xxx或者nodeenv=yyy的pod在同一Node上
vim pod-podaffinity-required.yaml
# 然后重新创建pod，查看效果
kubectl delete -f  pod-podaffinity-required.yaml
kubectl create -f pod-podaffinity-required.yaml
# 发现此时Pod运行正常
kubectl get pods pod-podaffinity-required -n dev
```

###### 软限制

> 软限制就不进行演示了，在使用方法大同小异

##### podAntiAffinity

PodAntiAffinity主要实现以运行的Pod为参照，让新创建的Pod跟参照pod不在一个区域中的功能，它的配置方式和选项跟PodAffinty是一样的，这里不再做详细解释，直接做一个测试案例

###### 硬限制

> 继续使用上个案例中目标pod

```shell
#可以看到pod-podaffinity-target有一个podenv标签，并且当前在node1节点上
kubectl get pods -n dev -o wide --show-labels
```

> 创建pod-podantiaffinity-required.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podantiaffinity-required
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  affinity:  #亲和性设置
    podAntiAffinity: #设置pod亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
      - labelSelector:
          matchExpressions: # 匹配podenv的值在["pro"]中的标签
          - key: podenv
            operator: In
            values: ["pro"]
        topologyKey: kubernetes.io/hostname
```

> 执行命令测试

```shell
# 创建pod
kubectl create -f pod-podantiaffinity-required.yaml
# 查看pod
# 发现调度到了node2上
kubectl get pods pod-podantiaffinity-required -n dev -o wide
```

###### 软限制

> 软限制就不进行演示了，在使用方法大同小异

#### 污点(容忍)调度

##### 污点(Taints)

前面的调度方式都是站在Pod的角度上，通过在Pod上添加属性，来确定Pod是否要调度到指定的Node上，其实我们也可以站在Node的角度上，通过在Node上添加**污点**属性，来决定是否允许Pod调度过来。

Node被设置上污点之后就和Pod之间存在了一种相斥的关系，进而拒绝Pod调度进来，甚至可以将已经存在的Pod驱逐出去。

污点的格式为：`key=value:effect`, key和value是污点的标签，effect描述污点的作用，支持如下三个选项：

- PreferNoSchedule：kubernetes将尽量避免把Pod调度到具有该污点的Node上，除非没有其他节点可调度
- NoSchedule：kubernetes将不会把Pod调度到具有该污点的Node上，但不会影响当前Node上已存在的Pod
- NoExecute：kubernetes将不会把Pod调度到具有该污点的Node上，同时也会将Node上已存在的Pod驱离

使用kubectl设置和去除污点的命令示例如下：

```shell
# 设置污点
kubectl taint nodes node1 key=value:effect
# 去除污点
kubectl taint nodes node1 key:effect-
# 去除所有污点
kubectl taint nodes node1 key-
```

> 使用kubeadm搭建的集群，默认就会给master节点添加一个污点标记,所以pod就不会调度到master节点上.

##### 容忍(Toleration)

上面介绍了污点的作用，我们可以在node上添加污点用于拒绝pod调度上来，但是如果就是想将一个pod调度到一个有污点的node上去，这时候应该怎么做呢？这就要使用到**容忍**

> 下面看一下容忍的详细配置

```shell
FIELDS:
   key       # 对应着要容忍的污点的键，空意味着匹配所有的键
   value     # 对应着要容忍的污点的值
   operator  # key-value的运算符，支持Equal和Exists（默认）
   effect    # 对应污点的effect，空意味着匹配所有影响
   tolerationSeconds   # 容忍时间, 当effect为NoExecute时生效，表示pod在Node上的停留时间
```

> 容忍某个污点yaml配置如下

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  tolerations:      # 添加容忍
  - key: "tag"        # 要容忍的污点的key
    operator: "Equal" # 操作符
    value: "test"    # 容忍的污点的value
    effect: "NoExecute"   # 添加容忍的规则，这里必须和标记的污点规则相同
```

### Pod控制器

Pod控制器是管理pod的中间层，使用了pod控制器之后，我们只需要告诉pod控制器想要多少个什么样的pod就可以了，塌就会创建出满足条件的pod并确保每一个pod处于用户期望的状态，如果pod在运行中出现故障，控制器会基于策略重建pod

在Kubernetes中，pod控制器的创建方式可发两类

1. 自主式pod：Kubernetes直接创建出来的pod，这种pod删除后就没有了，也不会重建
2. 控制器创建pod：通过控制器创建的pod，这种pod删除了之后还会自动重建

在Kubernets种，有很多类型的pod控制器，每种都有自己合适的场景，常见有如下

| 控制器                    | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| ReplicationController     | 比较原始的pod控制器，已经被废弃，由ReplicaSet替代            |
| ReplicaSet                | 保证副本数量一直维持在期望值，并支持pod数量扩缩容，镜像版本升级 |
| Deployment                | 通过控制ReplicaSet来控制Pod，并支持滚动升级、回退版本        |
| Horizontal Pod Autoscaler | 可以根据集群负载自动水平调整Pod的数量，实现削峰填谷          |
| DaemonSet                 | 在集群中的指定Node上运行且仅运行一个副本，一般用于守护进程类的任务 |
| Job                       | 它创建出来的pod只要完成任务就立即退出，不需要重启或重建，用于执行一次性任务 |
| Cronjob                   | 它创建的Pod负责周期性任务控制，不需要持续后台运行            |
| StatefulSet               | 管理有状态应用                                               |

#### ReplicaSet(RS)

ReplicaSet的主要作用是**保证一定数量的pod正常运行**，它会持续监听这些Pod的运行状态，一旦Pod发生故障，就会重启或重建。同时它还支持对pod数量的扩缩容和镜像版本的升降级

> yaml配置模板

```yaml
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: rs
spec: # 详情描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

> 在这里面，需要新了解的配置项就是`spec`下面几个选项：
>
> - replicas：指定副本数量，其实就是当前rs创建出来的pod的数量，默认为1
>
> - selector：选择器，它的作用是建立pod控制器和pod之间的关联关系，采用的Label Selector机制
>
>   在pod模板上定义label，在控制器上定义选择器，就可以表明当前控制器能管理哪些pod了
>
> - template：模板，就是当前控制器创建pod所使用的模板板，里面其实就是前一章学过的pod的定义

##### 创建

> 创建pc-replicaset.yaml文件

```yaml
apiVersion: apps/v1
kind: ReplicaSet   
metadata:
  name: pc-replicaset
  namespace: dev
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

```shell
# 创建rs
kubectl create -f pc-replicaset.yaml
# 查看rs
# DESIRED:期望副本数量  
# CURRENT:当前副本数量  
# READY:已经准备好提供服务的副本数量
kubectl get rs pc-replicaset -n dev -o wide
# 查看当前控制器创建出来的pod
# 这里发现控制器创建出来的pod的名称是在控制器名称后面拼接了-xxxxx随机码
kubectl get pod -n dev
```

##### 扩缩容

```shell
# 方式一：
# 编辑rs的副本数量，修改spec:replicas: 6即可
kubectl edit rs pc-replicaset -n dev
# 查看pod
kubectl get pods -n dev

# 方式二：
# 当然也可以直接使用命令实现
# 使用scale命令实现扩缩容， 后面--replicas=n直接指定目标数量即可
kubectl scale rs pc-replicaset --replicas=2 -n dev
# 命令运行完毕，立即查看，发现已经有4个开始准备退出了
# 稍等片刻，就只剩下2个了
kubectl get pods -n dev
```

##### 镜像升级

```shell
# 方式一：
# 编辑rs的容器镜像 - image: nginx:1.17.2
kubectl edit rs pc-replicaset -n dev
# 再次查看，发现镜像版本已经变更了
kubectl get rs -n dev -o wide

# 方式二：
# 同样的道理，也可以使用命令完成这个工作
# kubectl set image rs rs名称 容器=镜像版本 -n namespace
kubectl set image rs pc-replicaset nginx=nginx:1.17.1  -n dev
# 再次查看，发现镜像版本已经变更了
kubectl get rs -n dev -o wide
```

##### 删除

```shell
#方式一：
# 使用kubectl delete命令会删除此RS以及它管理的Pod
# 在kubernetes删除RS前，会将RS的replicasclear调整为0，等待所有的Pod被删除后，在执行RS对象的删除
kubectl delete rs pc-replicaset -n dev
kubectl get pod -n dev -o wide

#方式二：
# 如果希望仅仅删除RS对象（保留Pod），可以使用kubectl delete命令时添加--cascade=false选项（不推荐）。
kubectl delete rs pc-replicaset -n dev --cascade=false
kubectl get pods -n dev

#方式三：
# 也可以使用yaml直接删除(推荐)
kubectl delete -f pc-replicaset.yaml
```

#### Deployment(Deploy)

为了更好的解决服务编排的问题，kubernetes在V1.2版本开始，引入了Deployment控制器。值得一提的是，这种控制器并不直接管理pod，而是通过管理ReplicaSet来简介管理Pod，即：Deployment管理ReplicaSet，ReplicaSet管理Pod。所以Deployment比ReplicaSet功能更加强大。

Deployment主要功能有下面几个：

- 支持ReplicaSet的所有功能
- 支持发布的停止、继续
- 支持滚动更新和回滚版本

> yaml配置模板

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留历史版本
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

##### 创建

> 创建pc-deployment.yaml文件

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: dev
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

> 执行命令测试

```shell
# 创建deployment
kubectl create -f pc-deployment.yaml --record=true
# 查看deployment
# UP-TO-DATE 最新版本的pod的数量
# AVAILABLE  当前可用的pod的数量
kubectl get deploy pc-deployment -n dev
# 查看rs
# 发现rs的名称是在原来deployment的名字后面添加了一个10位数的随机串
kubectl get rs -n dev
# 查看pod
kubectl get pods -n dev
```

##### 扩缩容

```shell
#方式一：
# 变更副本数量为5个
kubectl scale deploy pc-deployment --replicas=5  -n dev
# 查看deployment
kubectl get deploy pc-deployment -n dev
# 查看pod
kubectl get pods -n dev

#方式二：
# 编辑deployment的副本数量，修改spec:replicas: 4即可
kubectl edit deploy pc-deployment -n dev
# 查看pod
kubectl get pods -n dev
```

##### 镜像更新

deployment支持两种更新策略:`重建更新`和`滚动更新`,可以通过`strategy`指定策略类型,支持两个属性:

```yaml
strategy：指定新的Pod替换旧的Pod的策略， 支持两个属性：
  type：指定策略类型，支持两种策略
    Recreate：在创建出新的Pod之前会先杀掉所有已存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本Pod
  rollingUpdate：当type为RollingUpdate时生效，用于为RollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

**重建更新**

在更新式，重建更新策略会将原来运行的pod全部删除，然后重新创建新的pod

> 编辑pc-deployment.yaml

```yaml
spec:
  strategy: # 策略
    type: Recreate # 重建更新
```

> 执行命令测试

```shell
# 更新配置
kubectl apply -f pc-deployment.yaml
# 观察升级过程(新开窗口)
kubectl get pods -n dev -w
# 变更镜像
kubectl set image deployment pc-deployment nginx=nginx:1.17.2 -n dev
```

**滚动更新**

在更新式，滚动更新策略会根据配置的`rollingUpdate`先启动对应数量新的成功后停止对应数量的旧的，不断重复直到所有旧的被替换

> 编辑pc-deployment.yaml

```yaml
spec:
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate:
      maxSurge: 25% 
      maxUnavailable: 25%
```

> 执行命令测试

```shell
# 更新配置
kubectl apply -f pc-deployment.yaml
# 观察升级过程(新开窗口)
kubectl get pods -n dev -w
# 变更镜像
kubectl set image deployment pc-deployment nginx=nginx:1.17.3 -n dev
```

> 在镜像更新时通过`kubectl get rs -n dev -w`完成rs可以发现在每次更新时rs都创建一个新的rs并且创建新pod，旧的rs的pod会被删除
>
> 之所以这样设计是为了后续的deployment版本回退，当更新时出现问题只需要切换rs就能够实现版本回退

![image-20230701205514431](./images/image-20230701205514431-168821611575413.png)

##### 版本回退

deployment支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能，下面具体来看.

kubectl rollout： 版本升级相关功能，支持下面的选项：

- status 显示当前升级状态
- history 显示 升级历史记录
- pause 暂停版本升级过程
- resume 继续已经暂停的版本升级过程
- restart 重启版本升级过程
- undo 回滚到上一级版本（可以使用--to-revision回滚到指定版本）

```shell
# 查看当前升级版本的状态
kubectl rollout status deploy pc-deployment -n dev
# 查看升级历史记录
kubectl rollout history deploy pc-deployment -n dev

# 版本回滚
# 这里直接使用--to-revision=1回滚到了1版本， 如果省略这个选项，就是回退到上个版本，就是2版本
kubectl rollout undo deployment pc-deployment --to-revision=1 -n dev
# 查看发现，通过nginx镜像版本可以发现到了第一版
kubectl get deploy -n dev -o wide
# 查看rs，发现第一个rs中有4个pod运行，后面两个版本的rs中pod为运行
# 其实deployment之所以可是实现版本的回滚，就是通过记录下历史rs来实现的，
# 一旦想回滚到哪个版本，只需要将当前版本pod数量降为0，然后将回滚版本的pod提升为目标数量就可以了
kubectl get rs -n dev
```

##### 金丝雀发布

Deployment控制器支持控制更新过程中的控制，如“暂停(pause)”或“继续(resume)”更新操作。

比如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的金丝雀发布。

```shell
# 更新deployment的版本，并配置暂停deployment
kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment  -n dev
#观察更新状态
kubectl rollout status deploy pc-deployment -n dev　
# 监控更新的过程，可以看到已经新增了一个资源，但是并未按照预期的状态去删除一个旧的资源，就是因为使用了pause暂停命令
kubectl get rs -n dev -o wide
kubectl get pods -n dev
# 确保更新的pod没问题了，继续更新
kubectl rollout resume deploy pc-deployment -n dev
# 查看最后的更新情况
kubectl get rs -n dev -o wide
kubectl get pods -n dev
```

##### 删除

```shell
kubectl delete -f pc-deployment.yaml
```

#### Horizontal Pod Autoscaler(HPA)

在前面的课程种，我们可以通过手工执行`kubectl scale`命令实现Pod扩容，但是这显然不够智能，Kubernetes期望可以通过检测Pod的使用情况，实现pod数量的自动调整，于是产生了HPA这种控制器

HPA可以获取每个pod利用率，然后和HPA中设定的指标进行对比，同时计算出需要伸缩的具体值，最后实现pod的数量的调整，其实HPA与之前的Deployment一样，也属于一种Kubernetes资源对象，它通过追踪分析目标pod的负载变化情况，来确认是否需要针对性的调整目标pod的副本数

##### 准备工作

###### 安装metrics-server

```shell
# 安装git
yum install git -y
# 获取metrics-server, 注意使用的版本
git clone -b v0.3.6 https://github.com/kubernetes-incubator/metrics-server
# 修改deployment, 注意修改的是镜像和初始化参数
cd /root/metrics-server/deploy/1.8+/
#添加下面选项
#hostNetwork: true
#image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6
#args:
#- --kubelet-insecure-tls
#- --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
vim metrics-server-deployment.yaml

# 安装metrics-server
kubectl apply -f ./
# 查看pod运行情况
kubectl get pod -n kube-system
# 使用kubectl top node 查看资源使用情况
kubectl top nodes
kubectl top pod -n kube-system
```

![image-20230701221140215](./images/image-20230701221140215-168822070152615.png)

###### 准备deployment和servie

> 执行如下命令

```yaml
# 使用命令模式创建deployment
kubectl run nginx --image=nginx:1.17.1 --requests=cpu=100m -n dev
# 创建service
kubectl expose deployment nginx --type=NodePort --port=80 -n dev
# 查看
kubectl get deployment,pod,svc -n dev
```

##### 部署HPA

> 创建pc-hpa.yaml文件

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: pc-hpa
  namespace: dev
spec:
  minReplicas: 1  #最小pod数量
  maxReplicas: 10 #最大pod数量
  targetCPUUtilizationPercentage: 3 # CPU使用率指标3%
  scaleTargetRef:   # 指定要控制的nginx信息
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
```

> 执行命令测试

```shell
# 创建hpa
kubectl create -f pc-hpa.yaml
# 查看hpa
kubectl get hpa -n dev
```

> 使用Jmeter进行压测，可发现当压力增加后容器也不断增加最多增加10个

![image-20230703231659779](./images/image-20230703231659779-16883974214071-16883974228423.png)

#### DaemonSet(DS)

DaemonSet类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本，一般适用于日志收集、节点监控等场景，也就是说，如果一个Pod提供的功能是节点级别的（每个节点都需要且只需要一个），那么这类Pod就适合使用DaemonSet类型的控制器创建。

> DaemonSet控制器的特点：
>
> - 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
> - 当节点从集群中移除时，Pod 也就被垃圾回收了

**DaemonSet的yaml配置清单**

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

##### 创建

> 创建pc-daemonset.yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet      
metadata:
  name: pc-daemonset
  namespace: dev
spec: 
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
```

> 执行命令测试

```shell
# 创建daemonset
kubectl create -f  pc-daemonset.yaml
# 查看daemonset
kubectl get ds -n dev -o wide
# 查看pod,发现在每个Node上都运行一个pod
kubectl get pods -n dev -o wide
# 删除daemonset
kubectl delete -f pc-daemonset.yaml
```

#### Job

Job，主要用于负责**批量处理(一次要处理指定数量任务)**短暂的**一次性(每个任务仅运行一次就结束)**任务。Job特点如下：

- 当Job创建的pod执行成功结束时，Job将记录成功结束的pod数量
- 当成功结束的pod达到指定的数量时，Job将完成执行

**Job的yaml配置清单**

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定job需要成功运行Pods的次数。默认值: 1
  parallelism: 1 # 指定job在任一时刻应该并发运行Pods的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定job可运行的时间期限，超过时间还未结束，系统将会尝试进行终止。
  backoffLimit: 6 # 指定job失败后进行重试的次数。默认是6
  manualSelector: true # 是否可以使用selector选择器选择pod，默认是false
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或者OnFailure
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```

> restartPolicy关于重启策略设置的说明：
>     如果指定为OnFailure，则job会在pod出现故障时重启容器，而不是创建pod，failed次数不变
>     如果指定为Never，则job会在pod出现故障时创建新的pod，并且故障pod不会消失，也不会重启，failed次数加1
>     如果指定为Always的话，就意味着一直重启，意味着job任务会重复去执行了，当然不对，所以不能设置为Always

##### 创建

> 创建pc-job.yaml

```yaml
apiVersion: batch/v1
kind: Job      
metadata:
  name: pc-job
  namespace: dev
spec:
  manualSelector: true
  selector:
    matchLabels:
      app: counter-pod
  template:
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never
      containers:
      - name: counter
        image: busybox:1.30
        command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

> 执行命令测试

```shell
# 创建job
kubectl create -f pc-job.yaml
# 查看job
kubectl get job -n dev -o wide  -w
# 通过观察pod状态可以看到，pod在运行完毕任务后，就会变成Completed状态
kubectl get pods -n dev -w
# 接下来，调整下pod运行的总数量和并行数量 即：在spec下设置下面两个选项
#  completions: 6 # 指定job需要成功运行Pods的次数为6
#  parallelism: 3 # 指定job并发运行Pods的数量为3
#  然后重新运行job，观察效果，此时会发现，job会每次运行3个pod，总共执行了6个pod
kubectl get pods -n dev -w
# 删除job
kubectl delete -f pc-job.yaml
```

#### Cronjob

CronJob控制器以Job控制器资源为其管控对象，并借助它管理pod资源对象，Job控制器定义的作业任务在其控制器资源创建之后便会立即执行，但CronJob可以以类似于Linux操作系统的周期性任务作业计划的方式控制其运行**时间点**及**重复运行**的方式。也就是说，**CronJob可以在特定的时间点(反复的)去运行job任务**。

**CronJob的yaml配置清单**

```yaml
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型       
metadata: # 元数据
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运行时间点,用于控制任务在什么时间执行
  concurrencyPolicy: # 并发执行策略，用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业
  failedJobHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
  startingDeadlineSeconds: # 启动作业错误的超时时长
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象;下面其实就是job的定义
    metadata:
    spec:
      completions: 1
      parallelism: 1
      activeDeadlineSeconds: 30
      backoffLimit: 6
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
        matchExpressions: 规则
          - {key: app, operator: In, values: [counter-pod]}
      template:
        metadata:
          labels:
            app: counter-pod
        spec:
          restartPolicy: Never 
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```

> 需要重点解释的几个选项：
> schedule: cron表达式，用于指定任务的执行时间
>
> concurrencyPolicy:
>     Allow:   允许Jobs并发运行(默认)
>     Forbid:  禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行
>     Replace: 替换，取消当前正在运行的作业并用新作业替换它

##### 创建

> 创建pc-cronjob.yaml

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pc-cronjob
  namespace: dev
  labels:
    controller: cronjob
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: counter
            image: busybox:1.30
            command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 3;done"]
```

> 执行命令测试

```shell
# 创建cronjob
kubectl create -f pc-cronjob.yaml
# 查看cronjob
kubectl get cronjobs -n dev
# 查看job
kubectl get jobs -n dev
# 查看pod
kubectl get pods -n dev
# 删除cronjob
kubectl  delete -f pc-cronjob.yaml
```

## Service详解

Kubernetes流量负载组件：Service(4层路由的负载)和Ingress(7层路由的负载)

在Kubernetes中，pod是应用程序的载体，通过访问pod的ip来访问程序，但是pod的ip地址都是随机生成的，为了解决这个问题Kubernetes提供了Service资源，Service资源会对相同一个服务多个pod进行聚合，并提供一个统一的入口地址，通过Service的入口地址就能访问到后面的pod服务

![image-20230708143034706](./images/image-20230708143034706-16887978356741-16887978365653.png)

Service在很多情况下只是一个概念，真正起作用的其实是kube-proxy服务进程，每个Node节点上都运行着一个Kube-proxy服务，当创建Service的时候会通过api-Servee想etcd写入创建Service信息，而kube-proxy会基于监听的机制发现这种Service的变动，然后它会将最新的Service信息**转换成对应的访问规则**

![image-20230708144202335](./images/image-20230708144202335-16887985235185.png)

> 10.97.97.97:80 是service提供的访问入口
> 当访问这个入口的时候，可以发现后面有三个pod的服务在等待调用，
> kube-proxy会基于rr（轮询）的策略，将请求分发到其中一个pod上去
> 这个规则会同时在集群内的所有节点上都生成，所以在任何一个节点上访问都可以。
> `ipvsadm -Ln`

![image-20230708151542713](./images/image-20230708151542713-16888005442837-16888005454149.png)

### kube-proxy工作模式

#### userspace

> userspace模式下，kube-proxy会为每一个Service创建一个监听端口，发向Cluster IP的请求被Iptables规则重定向到kube-proxy监听的端口上，kube-proxy根据LB算法选择一个提供服务的Pod并和其建立链接，以将请求转发到Pod上。  该模式下，kube-proxy充当了一个四层负责均衡器的角色。由于kube-proxy运行在userspace中，在进行转发处理时会增加内核和用户空间之间的数据拷贝，虽然比较稳定，但是效率比较低

![image-20230708151943064](./images/image-20230708151943064-168880078418611-168880078559413.png)

#### iptables

> iptables模式下，kube-proxy为service后端的每个Pod创建对应的iptables规则，直接将发向Cluster IP的请求重定向到一个Pod IP。  该模式下kube-proxy不承担四层负责均衡器的角色，只负责创建iptables规则。该模式的优点是较userspace模式效率更高，但不能提供灵活的LB策略，当后端Pod不可用时也无法进行重试。

![image-20230708152237879](./images/image-20230708152237879-168880095928015.png)

#### ipvs

> ipvs模式和iptables类似，kube-proxy监控Pod的变化并创建相应的ipvs规则。ipvs相对iptables转发效率更高。除此以外，ipvs支持更多的LB算法

![image-20230708153419636](./images/image-20230708153419636-168880166057217-168880166139919.png)

```shell
# 此模式必须安装ipvs内核模块，否则会降级为iptables
# 开启ipvs
kubectl edit cm kube-proxy -n kube-system
# 修改mode: "ipvs"
kubectl delete pod -l k8s-app=kube-proxy -n kube-system
# 查看ipvs信息
ipvsadm -Ln
```

### Service类型

**Service的yaml配置清单**

```yaml
kind: Service  # 资源类型
apiVersion: v1  # 资源版本
metadata: # 元数据
  name: service # 资源名称
  namespace: dev # 命名空间
spec: # 描述
  selector: # 标签选择器，用于确定当前service代理哪些pod
    app: nginx
  type: # Service类型，指定service的访问方式
  clusterIP:  # 虚拟服务的ip地址
  sessionAffinity: # session亲和性，支持ClientIP、None两个选项
  ports: # 端口信息
    - protocol: TCP 
      port: 3017  # service端口
      targetPort: 5003 # pod端口
      nodePort: 31122 # 主机端口
```

> - ClusterIP：默认值，它是Kubernetes系统自动分配的虚拟IP，只能在集群内部访问
> - NodePort：将Service通过指定的Node上的端口暴露给外部，通过此方法，就可以在集群外部访问服务
> - LoadBalancer：使用外接负载均衡器完成到服务的负载分发，注意此模式需要外部云环境支持
> - ExternalName： 把集群外部的服务引入集群内部，直接使用

### Service使用

#### 环境准备

> 首先我们需要安装如下配置准备3个pod，通过Service访问pod服务再通过切换不同类型的Service来测试不同类型的Service的区别

| pod程序 | pod控制器 | pod暴露端口 |
| ------- | --------- | ----------- |
| nignx   | deploy    | 80          |
| nignx   | deploy    | 80          |
| nignx   | deploy    | 80          |

> 创建deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment      
metadata:
  name: pc-deployment
  namespace: dev
spec: 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

> 执行命令

```shell
# 创建pod
kubectl create -f deployment.yaml
# 查看pod详情
kubectl get pods -n dev -o wide --show-labels
```
> 在不同的机器上使用命令进入容器，修改inde.heml页面内容，方便后期测试时知道访问的是那个nginx

```shell
# 为了方便后面的测试，修改下三台nginx的index.html页面（三台修改的IP地址不一致）
kubectl exec -it pc-deployment-6696798b78-7lms6 -n dev /bin/sh 
echo "nginx IP => 10.244.2.43" > /usr/share/nginx/html/index.html
exit

kubectl exec -it pc-deployment-6696798b78-f7w2g -n dev /bin/sh
echo "nginx IP => 10.244.1.142" > /usr/share/nginx/html/index.html
exit

kubectl exec -it pc-deployment-6696798b78-q9dqw -n dev /bin/sh
echo "nginx IP => 10.244.1.141" > /usr/share/nginx/html/index.html
exit
```
![image-20230708161905939](./images/image-20230708161905939-168880434825329-168880434933431.png)

> 测试是否能够正常访问

```shell
curl 10.244.2.43
curl 10.244.1.142
curl 10.244.1.141
```

![image-20230708161941417](./images/image-20230708161941417-168880438293333-168880438395035.png)

#### ClusterIP类型Service

> 创建service-clusterip.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-clusterip
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: 10.97.97.97 # service的ip地址，如果不写，默认会生成一个
  type: ClusterIP
  ports:
  - port: 80  # Service端口       
    targetPort: 80 # pod端口
```

> 执行如下命令

```shell
# 创建service
kubectl create -f service-clusterip.yaml
# 查看service
kubectl get svc -n dev -o wide
# 查看service的详细信息
# 在这里有一个Endpoints列表，里面就是当前service可以负载到的服务入口
kubectl describe svc service-clusterip -n dev
```

![image-20230708162658348](./images/image-20230708162658348-168880481946837-168880482032539.png)

> 测试，查看kube-poxy是否创建了相应的ipvs规则

~~~shell
# 查看ipvs的映射规则
ipvsadm -Ln | grep 10.97.97.97 -A 3
# 访问10.97.97.97:80观察效果
curl 10.97.97.97:80
~~~

![image-20230708162910403](./images/image-20230708162910403-168880495172441-168880495244743.png)

**负载分发策略**

对Service的访问被分发到了后端的Pod上去，目前kubernetes提供了两种负载分发策略：

- 如果不定义，默认使用kube-proxy的策略，比如随机、轮询（上面的rr就是轮询）

- 基于客户端地址的会话保持模式，即来自同一个客户端发起的所有请求都会转发到固定的一个Pod上

  此模式可以使在spec中添加`sessionAffinity: ClientIP`选项

~~~shell
#修改原有的配置文件
#添加sessionAffinity: ClientIP
vim service-clusterip.yaml
#应用配置文件
kubectl apply -f service-clusterip.yaml
# 查看ipvs的映射规则
ipvsadm -Ln | grep 10.97.97.97 -A 3
# 访问10.97.97.97:80观察效果
curl 10.97.97.97:80
~~~

> ipvsadm可以看到在rr后面多了persistent 10800(持久化10800秒)
>
> 可以发现亲和性已改为ClientIP
>
> 通过curl发全球可以发现请求转发都是基于session的，在有效期内都是访问的相同的nginx

![image-20230708164514621](images/image-20230708164514621.png)

![image-20230708164237005](./images/image-20230708164237005-168880575857545-168880575949347.png)

#### HeadLiness类型的Service

在某些场景中，开发人员可能不想使用Service提供的负载均衡功能，而希望自己来控制负载均衡策略，针对这种情况，kubernetes提供了HeadLiness Service，这类Service不会分配Cluster IP，如果想要访问service，只能通过service的域名进行查询。

> 创建service-headliness.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
  - port: 80    
    targetPort: 80
```

> 执行如下命令

```shell
# 创建service
kubectl  create -f service-headliness.yaml
```

> 创建成功后因为通过域名访问就需要域名解析器，随便进入一台pod查看以下域名解析器的地址

![image-20230708170248258](./images/image-20230708170248258-168880696934649-168880697043251.png)

> 使用dig命令查看域名解析`dig @10.96.0.10 service-headliness.dev.svc.cluster.local`

![image-20230708171601386](./images/image-20230708171601386.png)

#### NodePort类型的Service

在之前的样例中，创建的Service的ip地址只有集群内部才可以访问，如果希望将Service暴露给集群外部使用，那么就要使用到另外一种类型的Service，称为NodePort类型。NodePort的工作原理其实就是**将service的端口映射到Node的一个端口上**，然后就可以通过`NodeIp:NodePort`来访问service了。

> 创建service-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-nodeport
  namespace: dev
spec:
  selector:
    app: nginx-pod
  type: NodePort # service类型
  ports:
  - port: 80
    nodePort: 30002 # 指定绑定的node的端口(默认的取值范围是：30000-32767), 如果不指定，会默认分配
    targetPort: 80
```

> 执行如下命令

```shell
# 创建service
kubectl create -f service-nodeport.yaml
# 查看service
kubectl get svc -n dev -o wide
```

![image-20230708200607893](./images/image-20230708200607893-168881797059053.png)

#### LoadBalancer类型的Service

LoadBalancer和NodePort很相似，目的都是向外部暴露一个端口，区别在于LoadBalancer会在集群的外部再来做一个负载均衡设备，而这个设备需要外部环境支持的，外部服务发送到这个设备上的请求，会被设备负载之后转发到集群中。

#### ExternalName类型的Service

ExternalName类型的Service用于引入集群外部的服务，它通过`externalName`属性指定外部一个服务的地址，然后在集群内部访问此service就可以访问到外部的服务了。

```shell
apiVersion: v1
kind: Service
metadata:
  name: service-externalname
  namespace: dev
spec:
  type: ExternalName # service类型
  externalName: www.baidu.com  #改成ip地址也可以
```

```shell
# 创建service
kubectl  create -f service-externalname.yaml
# 域名解析
dig @10.96.0.10 service-externalname.dev.svc.cluster.local
service-externalname.dev.svc.cluster.local. 30 IN CNAME www.baidu.com.
```

### Ingress介绍

在前面课程中已经提到，Service对集群之外暴露服务的主要方式有两种：NotePort和LoadBalancer，但是这两种方式，都有一定的缺点：

- NodePort方式的缺点是会占用很多集群机器的端口，那么当集群服务变多的时候，这个缺点就愈发明显
- LB方式的缺点是每个service需要一个LB，浪费、麻烦，并且需要kubernetes之外设备的支持

基于这种现状，kubernetes提供了Ingress资源对象，Ingress只需要一个NodePort或者一个LB就可以满足暴露多个Service的需求。工作机制大致如下图表示：

![image-20230715172604458](./images/image-20230715172604458-16894131660481-16894131674623.png)

实际上，Ingress相当于一个7层的负载均衡器，是kubernetes对反向代理的一个抽象，它的工作原理类似于Nginx，可以理解成在**Ingress里建立诸多映射规则，Ingress Controller通过监听这些配置规则并转化成Nginx的反向代理配置 , 然后对外部提供服务**。在这里有两个核心概念：

- ingress：kubernetes中的一个对象，作用是定义请求如何转发到service的规则
- ingress controller：具体实现反向代理及负载均衡的程序，对ingress定义的规则进行解析，根据配置的规则来实现请求转发，实现方式有很多，比如Nginx, Contour, Haproxy等等

Ingress（以Nginx为例）的工作原理如下：

1. 用户编写Ingress规则，说明哪个域名对应kubernetes集群中的哪个Service
2. Ingress控制器动态感知Ingress服务规则的变化，然后生成一段对应的Nginx反向代理配置
3. Ingress控制器会将生成的Nginx配置写入到一个运行着的Nginx服务中，并动态更新
4. 到此为止，其实真正在工作的就是一个Nginx了，内部配置了用户定义的请求转发规则

![image-20230715173336627](./images/image-20230715173336627.png)

#### 环境准备

**搭建ingress环境**

```shell
# 创建文件夹
mkdir ingress-controller
cd ingress-controller/
# 获取ingress-nginx，本次案例使用的是0.30版本
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
# 创建ingress-nginx
kubectl apply -f ./
# 查看ingress-nginx
kubectl get pod -n ingress-nginx
# 查看service
kubectl get svc -n ingress-nginx
```

**准备service和pod**

> 为了接下来的实验我们准备如下实现模型

![image-20230715174553825](./images/image-20230715174553825.png)

> 创建nginx-tomcat.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

> 执行如下命令

```shell
# 创建
kubectl create -f nginx-tomcat.yaml
# 查看
kubectl get svc -n dev
```

#### Http代理

> 创建ingress-http.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
  - host: nginx.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

> 执行如下命令

```shell
# 创建
kubectl create -f ingress-http.yaml
# 查看
kubectl get ing ingress-http -n dev
# 查看详情
kubectl describe ing ingress-http  -n dev
# 接下来,在本地电脑上配置host文件,解析上面的两个域名到192.168.10.100(master)上
# 然后,就可以分别访问tomcat.test.com:32519  和  nginx.test.com:32519 查看效果了
```

![image-20230715182641135](./images/image-20230715182641135.png)

#### Https代理

>  创建证书

```shell
# 生成证书
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=test.com"
# 创建密钥
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```

> 创建ingress-https.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
  namespace: dev
spec:
  tls:
    - hosts:
      - nginx.test.com
      - tomcat.test.com
      secretName: tls-secret # 指定秘钥
  rules:
  - host: nginx.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

> 执行如下命令

```shell
# 创建
kubectl create -f ingress-https.yaml
# 查看
kubectl get ing ingress-https -n dev
# 查看详情
kubectl describe ing ingress-https -n dev
```

![image-20230715183302039](./images/image-20230715183302039.png)

## 数据存储

在前面已经提到，容器的生命周期可能很短，会被频繁地创建和销毁。那么容器在销毁时，保存在容器中的数据也会被清除。这种结果对用户来说，在某些情况下是不乐意看到的。为了持久化保存容器的数据，kubernetes引入了Volume的概念。

Volume是Pod中能够被多个容器访问的共享目录，它被定义在Pod上，然后被一个Pod里的多个容器挂载到具体的文件目录下，kubernetes通过Volume实现同一个Pod中不同容器之间的数据共享以及数据的持久化存储。Volume的生命容器不与Pod中单个容器的生命周期相关，当容器终止或者重启时，Volume中的数据也不会丢失。

kubernetes的Volume支持多种类型，比较常见的有下面几个：

- 简单存储：EmptyDir、HostPath、NFS
- 高级存储：PV、PVC
- 配置存储：ConfigMap、Secret

### 基础存储

#### EmptyDir

EmptyDir是最基础的Volume类型，一个EmptyDir就是Host上的一个空目录，EmptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且无须指定宿主机上对应的目录文件，因为Kubernetes会自动分配一个目录，`当Pod销毁时EmptyDir中的数据也会被永久删除`

EmptyDir用途如下

- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）

**案例演示**

> 接下来，通过一个容器之间文件共享的案例来使用一下EmptyDir。
>
> 在一个Pod中准备两个容器nginx和busybox，然后声明一个Volume分别挂在到两个容器的目录中，然后nginx容器负责向Volume中写日志，busybox中通过命令将日志内容读到控制台。

![image-20230716171304249](./images/image-20230716171304249.png)

> 创建一个volume-emptydir.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:  # 将logs-volume挂在到nginx容器中，对应的目录为 /var/log/nginx
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件中内容
    volumeMounts:  # 将logs-volume 挂在到busybox容器中，对应的目录为 /logs
    - name: logs-volume
      mountPath: /logs
  volumes: # 声明volume， name为logs-volume，类型为emptyDir
  - name: logs-volume
    emptyDir: {}
```

> 执行如下命令

```shell
# 创建Pod
kubectl create -f volume-emptydir.yaml
# 查看pod
kubectl get pods volume-emptydir -n dev -o wide
# 通过podIp访问nginx
curl 10.244.1.150
# 通过kubectl logs命令查看指定容器的标准输出
kubectl logs -f volume-emptydir -n dev -c busybox
```

![image-20230716172144625](./images/image-20230716172144625.png)

#### HostPath

前面提到的EmptyDir中数据不会被持久化，它会随着Pod的结束而销毁，如果想简单的将数据持久化到主机中，可以选择HostPath。

HostPath就是将Node主机中一个实际目录挂在到Pod中，以供容器使用，这样的设计就可以保证Pod销毁了，但是数据依据可以存在于Node主机上。

![image-20230716172458247](./images/image-20230716172458247.png)

> 创建一个volume-hostpath.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"]
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    hostPath: 
      path: /root/logs
      type: DirectoryOrCreate  # 目录存在就使用，不存在就先创建后使用
```

> 关于type的值的一点说明：

| 类型              | 说明                                 |
| ----------------- | ------------------------------------ |
| DirectoryOrCreate | 目录存在就使用，不存在就先创建后使用 |
| Directory         | 目录必须存在                         |
| FileOrCreate      | 文件存在就使用，不存在就先创建后使用 |
| File              | 文件必须存在                         |
| Socket            | unix套接字必须存在                   |
| CharDevice        | 字符设备必须存在                     |
| BlockDevice       | 块设备必须存在                       |

> 执行如下代码

```shell
# 创建Pod
kubectl create -f volume-hostpath.yaml
# 查看Pod
kubectl get pods volume-hostpath -n dev -o wide
#访问nginx
curl 10.244.2.51
# 接下来就可以去host的/root/logs目录下查看存储的文件了
###  注意: 下面的操作需要到Pod所在的节点运行（案例中是node1）
ls /root/logs/
# 同样的道理，如果在此目录下创建一个文件，到容器中也是可以看到的
```

![image-20230716173203894](./images/image-20230716173203894.png)

#### NFS

HostPath可以解决数据持久化的问题，但是一旦Node节点故障了，Pod如果转移到了别的节点，又会出现问题了，此时需要准备单独的网络存储系统，比较常用的用NFS、CIFS。

NFS是一个网络文件存储系统，可以搭建一台NFS服务器，然后将Pod中的存储直接连接到NFS系统上，这样的话，无论Pod在节点上怎么转移，只要Node跟NFS的对接没问题，数据就可以成功访问。

![image-20230716173607195](./images/image-20230716173607195.png)

**准备nsf服务器**

> 这里为了方便测试直接将master作为nfs服务器

```shell
# 在nfs上安装nfs服务
yum install nfs-utils -y
# 准备一个共享目录
mkdir /root/data/nfs -pv
# 编辑/etc/exports文件(nfs默认读取的一个配置文件)
# 添加内容如下 /root/data/nfs     192.168.10.0/24(rw,no_root_squash)
# 将共享目录以读写权限暴露给192.168.10.0/24网段中的所有主机
vim /etc/exports
more /etc/exports
# 启动nfs服务
systemctl restart nfs
```

**在node上安装nfs工具**

```shell
# 在node上安装nfs服务，注意不需要启动
yum install nfs-utils -y
```

**创建pod配置文件**

> 创建volume-nfs.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-nfs
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs-volume
      mountPath: /var/log/nginx
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","tail -f /logs/access.log"] 
    volumeMounts:
    - name: logs-volume
      mountPath: /logs
  volumes:
  - name: logs-volume
    nfs:
      server: 192.168.10.100  #nfs服务器地址
      path: /root/data/nfs #共享文件路径
```

> 执行如下命令

```shell
# 创建pod
kubectl create -f volume-nfs.yaml
# 查看pod
kubectl get pods volume-nfs -n dev
# 查看nfs服务器上的共享目录，发现已经有文件了
ls /root/data/nfs
```

### 高级存储

前面已经学习了使用NFS提供存储，此时就要求用户会搭建NFS系统，并且会在yaml配置nfs。由于kubernetes支持的存储系统有很多，要求客户全都掌握，显然不现实。为了能够屏蔽底层存储实现的细节，方便用户使用， kubernetes引入PV和PVC两种资源对象。

PV（Persistent Volume）是持久化卷的意思，是对底层的共享存储的一种抽象。一般情况下PV由kubernetes管理员进行创建和配置，它与底层具体的共享存储技术有关，并通过插件完成与共享存储的对接。

PVC（Persistent Volume Claim）是持久卷声明的意思，是用户对于存储需求的一种声明。换句话说，PVC其实就是用户向kubernetes系统发出的一种资源需求申请。

![image-20230716180751289](./images/image-20230716180751289.png)

> 使用了PV和PVC之后，工作可以得到进一步的细分：
>
> - 存储：存储工程师维护
> - PV： kubernetes管理员维护
> - PVC：kubernetes用户维护

#### PV

**PV的yaml配置清单**

```yaml
apiVersion: v1  
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，与底层真正存储对应
  capacity:  # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes:  # 访问模式
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
```

##### 参数说明

- **存储类型**

  底层实际存储的类型，kubernetes支持多种存储类型，每种存储类型的配置都有所差异

- **存储能力（capacity）**

 	   目前只支持存储空间的设置( storage=1Gi )，不过未来可能会加入IOPS、吞吐量等指标的配置

- **访问模式（accessModes）**

  用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：

  - ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
  - ReadOnlyMany（ROX）： 只读权限，可以被多个节点挂载
  - ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

  `需要注意的是，底层不同的存储类型可能支持的访问模式不同`

- **回收策略（persistentVolumeReclaimPolicy）**

  当PV不再被使用了之后，对其的处理方式。目前支持三种策略：

  - Retain （保留） 保留数据，需要管理员手工清理数据
  - Recycle（回收） 清除 PV 中的数据，效果相当于执行 rm -rf /thevolume/*
  - Delete （删除） 与 PV 相连的后端存储完成 volume 的删除操作，当然这常见于云服务商的存储服务

  `需要注意的是，底层不同的存储类型可能支持的回收策略不同`

- **存储类别**

  PV可以通过storageClassName参数指定一个存储类别

  - 具有特定类别的PV只能与请求了该类别的PVC进行绑定
  - 未设定类别的PV则只能与不请求任何类别的PVC进行绑定

- **状态（status）**

  一个 PV 的生命周期中，可能会处于4中不同的阶段：

  - Available（可用）： 表示可用状态，还未被任何 PVC 绑定
  - Bound（已绑定）： 表示 PV 已经被 PVC 绑定
  - Released（已释放）： 表示 PVC 被删除，但是资源还未被集群重新声明
  - Failed（失败）： 表示该 PV 的自动回收失败

##### 创建PV

> 以下使用nfs作为存储，演示PV的使用，创建3个PV，对应NFS中的3个暴露的路径

**环境准备**

```shell
# 创建目录
mkdir /root/data/{pv1,pv2,pv3} -pv
# 暴露服务,在/etc/exports添加如下内容
#/root/data/pv1     192.168.10.0/24(rw,no_root_squash)
#/root/data/pv2     192.168.10.0/24(rw,no_root_squash)
#/root/data/pv3     192.168.10.0/24(rw,no_root_squash)
vim /etc/exports
# 重启服务
systemctl restart nfs
```

**创建yaml**

> 创建pv.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv1
spec:
  capacity: 
    storage: 1Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv1
    server: 192.168.10.100

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv2
spec:
  capacity: 
    storage: 2Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv2
    server: 192.168.10.100
    
---

apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv3
spec:
  capacity: 
    storage: 3Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /root/data/pv3
    server: 192.168.10.100
```

> 执行如下命令

```shell
# 创建 pv
kubectl create -f pv.yaml
# 查看pv
kubectl get pv -o wide
```

![image-20230716182706642](./images/image-20230716182706642.png)

#### PVC

PVC是对管理员创建好的PV资源的申请，用来声明对存储空间、访问模式、存储类别需求信息

**PVC的yaml配置清单**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  namespace: dev
spec:
  accessModes: # 访问模式
  selector: # 采用标签对PV选择
  storageClassName: # 存储类别
  resources: # 请求空间
    requests:
      storage: 5Gi
```

##### 参数说明

- **访问模式（accessModes）**

用于描述用户应用对存储资源的访问权限

- **选择条件（selector）**

  通过Label Selector的设置，可使PVC对于系统中己存在的PV进行筛选

- **存储类别（storageClassName）**

  PVC在定义时可以设定需要的后端存储的类别，只有设置了该class的pv才能被系统选出

- **资源请求（Resources ）**

  描述对存储资源的请求

##### PV绑定PVC

> 创建pvc.yaml，申请PV

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc3
  namespace: dev
spec:
  accessModes: 
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```shell
# 创建pvc
kubectl create -f pvc.yaml
# 查看pvc
kubectl get pvc  -n dev -o wide
# 查看pv
kubectl get pv -o wide
```

![image-20230716183058803](./images/image-20230716183058803.png)

##### Pod使用PV

> 创建pods.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc1
        readOnly: false
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod2 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc2
        readOnly: false
```

> 执行如下命令

```shell
# 创建pod
kubectl create -f pods.yaml
# 查看pod
kubectl get pods -n dev -o wide
# 查看pvc
kubectl get pvc -n dev -o wide
# 查看pv
kubectl get pv -n dev -o wide
# 查看nfs中的文件存储
more /root/data/pv1/out.txt
more /root/data/pv2/out.txt
```

![image-20230716183708921](./images/image-20230716183708921.png)

#### 生命周期

PVC和PV是一一对应的，PV和PVC之间的相互作用遵循以下生命周期：

- **资源供应**：管理员手动创建底层存储和PV

- **资源绑定**：用户创建PVC，kubernetes负责根据PVC的声明去寻找PV，并绑定

  在用户定义好PVC之后，系统将根据PVC对存储资源的请求在已存在的PV中选择一个满足条件的

  - 一旦找到，就将该PV与用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了
  - 如果找不到，PVC则会无限期处于Pending状态，直到等到系统管理员创建了一个符合其要求的PV

  PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再与其他PVC进行绑定了

- **资源使用**：用户可在pod中像volume一样使用pvc

  Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。

- **资源释放**：用户删除pvc来释放pv

  当存储资源使用完毕后，用户可以删除PVC，与该PVC绑定的PV将会被标记为“已释放”，但还不能立刻与其他PVC进行绑定。通过之前PVC写入的数据可能还被留在存储设备上，只有在清除之后该PV才能再次使用。

- **资源回收**：kubernetes根据pv设置的回收策略进行资源的回收

  对于PV，管理员可以设定回收策略，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用

![image-20230716185120355](./images/image-20230716185120355.png)

### 配置存储

#### ConfigMap

ConfigMap是一种比较特殊的存储卷，它的主要作用是用来存储配置信息的

> 创建configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
  namespace: dev
data:
  info: |
    username:admin
    password:123456
```

> 接下来，使用此配置文件创建configmap

```shell
# 创建configmap
kubectl create -f configmap.yaml
# 查看configmap详情
kubectl describe cm configmap -n dev
```

> 接下来创建一个pod-configmap.yaml，将上面创建的configmap挂载进去

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将configmap挂载到目录
    - name: config
      mountPath: /configmap/config
  volumes: # 引用configmap
  - name: config
    configMap:
      name: configmap
```

> 执行如下命令

```shell
# 创建pod
kubectl create -f pod-configmap.yaml
# 查看pod
kubectl get pod pod-configmap -n dev
#进入容器 执行如下命令 more /configmap/config/info
kubectl exec -it pod-configmap -n dev /bin/sh
# 可以看到映射已经成功，每个configmap都映射成了一个目录
# key--->文件     value---->文件中的内容
# 此时如果更新configmap的内容, 容器中的值也会动态更新
```

#### Secret

在kubernetes中，还存在一种和ConfigMap非常类似的对象，称为Secret对象。它主要用于存储敏感信息，例如密码、秘钥、证书等等

> 首先使用base64对数据进行编码

```shell
echo -n 'admin' | base64 #准备username YWRtaW4=
echo -n '123456' | base64 #准备password MTIzNDU2
```

> 接下来编写secret.yaml，并创建Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

> 执行如下命令

```shell
# 创建secret
kubectl create -f secret.yaml
# 查看secret详情
kubectl describe secret secret -n dev
```

> 创建pod-secret.yaml，将上面创建的secret挂载进去

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
    volumeMounts: # 将secret挂载到目录
    - name: config
      mountPath: /secret/config
  volumes:
  - name: config
    secret:
      secretName: secret
```

> 执行如下命令

```shell
# 创建pod
kubectl create -f pod-secret.yaml
# 查看pod
kubectl get pod pod-secret -n dev
# 进入容器，查看secret信息，发现已经自动解码了
# more /secret/config/username && more /secret/config/password
kubectl exec -it pod-secret /bin/sh -n dev
```

## 安全认证

Kubernetes作为一个分布式集群的管理工具，保证集群的安全性是其一个重要的任务。所谓的安全性其实就是保证对Kubernetes的各种**客户端**进行**认证和鉴权**操作

**客户端**

在Kubernetes集群中，客户端通常有两类：

- **User Account**：一般是独立于kubernetes之外的其他服务管理的用户账号。
- **Service Account**：kubernetes管理的账号，用于为Pod中的服务进程在访问Kubernetes时提供身份标识。

![image-20230716202852531](./images/image-20230716202852531.png)

**认证、授权与准入控制**

ApiServer是访问及管理资源对象的唯一入口。任何一个请求访问ApiServer，都要经过下面三个流程：

- Authentication（认证）：身份鉴别，只有正确的账号才能够通过认证
- Authorization（授权）： 判断用户是否有权限对访问的资源执行特定的动作
- Admission Control（准入控制）：用于补充授权机制以实现更加精细的访问控制功能。

![image-20230716203210242](./images/image-20230716203210242.png)

### 认证管理

Kubernetes集群安全的最关键点在于如何识别并认证客户端身份，它提供了3种客户端身份认证方式：

1. HTTP Base认证：通过用户名+密码的方式认证

   > 这种认证方式是把“用户名:密码”用BASE64算法进行编码后的字符串放在HTTP请求中的Header Authorization域里发送给服务端。服务端收到后进行解码，获取用户名及密码，然后进行用户身份认证的过程。

2. HTTP Token认证：通过一个Token来识别合法用户

   > 这种认证方式是用一个很长的难以被模仿的字符串--Token来表明客户身份的一种方式。每个Token对应一个用户名，当客户端发起API调用请求时，需要在HTTP Header里放入Token，API Server接到Token后会跟服务器中保存的token进行比对，然后进行用户身份认证的过程。

2.  HTTPS证书认证：基于CA根证书签名的双向数字证书认证方式

   > 这种认证方式是安全性最高的一种方式，但是同时也是操作起来最麻烦的一种方式。

**HTTPS认证大体分为3个过程：**

1. 证书申请和下发

   ```
   HTTPS通信双方的服务器向CA机构申请证书，CA机构下发根证书、服务端证书及私钥给申请者
   ```

2. 客户端和服务端的双向认证

   ```
     1> 客户端向服务器端发起请求，服务端下发自己的证书给客户端，
        客户端接收到证书后，通过私钥解密证书，在证书中获得服务端的公钥，
        客户端利用服务器端的公钥认证证书中的信息，如果一致，则认可这个服务器
     2> 客户端发送自己的证书给服务器端，服务端接收到证书后，通过私钥解密证书，
        在证书中获得客户端的公钥，并用该公钥认证证书信息，确认客户端是否合法
   ```

3. 服务器端和客户端进行通信

   ```
     服务器端和客户端协商好加密方案后，客户端会产生一个随机的秘钥并加密，然后发送到服务器端。
     服务器端接收这个秘钥后，双方接下来通信的所有内容都通过该随机秘钥加密
   ```

> 注意: Kubernetes允许同时配置多种认证方式，只要其中任意一个方式认证通过即可

### 授权管理

授权发生在认证成功之后，通过认证就可以知道请求用户是谁， 然后Kubernetes会根据事先定义的授权策略来决定用户是否有权限访问，这个过程就称为授权。

每个发送到ApiServer的请求都带上了用户和资源的信息：比如发送请求的用户、请求的路径、请求的动作等，授权就是根据这些信息和授权策略进行比较，如果符合策略，则认为授权通过，否则会返回错误。

API Server目前支持以下几种授权策略：

- AlwaysDeny：表示拒绝所有请求，一般用于测试
- AlwaysAllow：允许接收所有请求，相当于集群不需要授权流程（Kubernetes默认的策略）
- ABAC：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制
- Webhook：通过调用外部REST服务对用户进行授权
- Node：是一种专用模式，用于对kubelet发出的请求进行访问控制
- RBAC：基于角色的访问控制（kubeadm安装方式下的默认选项）

RBAC(Role-Based Access Control) 基于角色的访问控制，主要是在描述一件事情：**给哪些对象授予了哪些权限**

其中涉及到了下面几个概念：

- 对象：User、Groups、ServiceAccount
- 角色：代表着一组定义在资源上的可操作动作(权限)的集合
- 绑定：将定义好的角色跟用户绑定在一起

![image-20230716205215495](./images/image-20230716205215495.png)

RBAC引入了4个顶级资源对象：

- Role(ns级别)、ClusterRole(集群级别)：角色，用于指定一组权限
- RoleBinding(ns级别)、ClusterRoleBinding(集群级别)：角色绑定，用于将角色（权限）赋予给对象

#### Role、ClusterRole

一个角色就是一组权限的集合，这里的权限都是许可形式的（白名单）

```yaml
# Role只能对命名空间内的资源进行授权，需要指定nameapce
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: dev
  name: authorization-role
rules:
- apiGroups: [""]  # 支持的API组列表,"" 空字符串，表示核心API群
  resources: ["pods"] # 支持的资源对象列表
  verbs: ["get", "watch", "list"] # 允许的对资源对象的操作方法列表
```

```yaml
# ClusterRole可以对集群范围内资源、跨namespaces的范围资源、非资源类型进行授权
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
 name: authorization-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

需要详细说明的是，rules中的参数：

- apiGroups: 支持的API组列表

  ```
  "","apps", "autoscaling", "batch"
  ```

- resources：支持的资源对象列表

  ```
  "services", "endpoints", "pods","secrets","configmaps","crontabs","deployments","jobs",
  "nodes","rolebindings","clusterroles","daemonsets","replicasets","statefulsets",
  "horizontalpodautoscalers","replicationcontrollers","cronjobs"
  ```

- verbs：对资源对象的操作方法列表

  ```
  "get", "list", "watch", "create", "update", "patch", "delete", "exec"
  ```

#### RoleBinding、ClusterRoleBinding

角色绑定用来把一个角色绑定到一个目标对象上，绑定目标可以是User、Group或者ServiceAccount

```yaml
# RoleBinding可以将同一namespace中的subject绑定到某个Role下，则此subject即具有该Role定义的权限
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: authorization-role-binding
  namespace: dev
subjects:
- kind: User
  name: test
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: authorization-role
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ClusterRoleBinding在整个集群级别和所有namespaces将特定的subject与ClusterRole绑定，授予权限
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
 name: authorization-clusterrole-binding
subjects:
- kind: User
  name: test
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: authorization-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

#### RoleBinding引用ClusterRole进行授权

RoleBinding可以引用ClusterRole，对属于同一命名空间内ClusterRole定义的资源主体进行授权。

> 一种很常用的做法就是，集群管理员为集群范围预定义好一组角色（ClusterRole），然后在多个命名空间中重复使用这些ClusterRole。这样可以大幅提高授权管理工作效率，也使得各个命名空间下的基础性授权规则与使用体验保持一致。

```yaml
# 虽然authorization-clusterrole是一个集群角色，但是因为使用了RoleBinding
# 所以test只能读取dev命名空间中的资源
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: authorization-role-binding-ns
  namespace: dev
subjects:
- kind: User
  name: test
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: authorization-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

#### 案例

> 在前面演示时我们都使用的是kubernetes提供的默认账号admin，admin账号权限极高能够操作所有的命名空间，在企业中开发中通常需要缩减权限只需要给开发人员配置只能操作相应的命名空间即可

**创建账号**

```shell
# 1) 创建证书
cd /etc/kubernetes/pki/
(umask 077;openssl genrsa -out devman.key 2048)

# 2) 用apiserver的证书去签署
# 2-1) 签名申请，申请的用户是devman,组是devgroup
openssl req -new -key devman.key -out devman.csr -subj "/CN=devman/O=devgroup"     
# 2-2) 签署证书
openssl x509 -req -in devman.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out devman.crt -days 3650

# 3) 设置集群、用户、上下文信息
kubectl config set-cluster kubernetes --embed-certs=true --certificate-authority=/etc/kubernetes/pki/ca.crt --server=https://192.168.10.100:6443

kubectl config set-credentials devman --embed-certs=true --client-certificate=/etc/kubernetes/pki/devman.crt --client-key=/etc/kubernetes/pki/devman.key

kubectl config set-context devman@kubernetes --cluster=kubernetes --user=devman

# 切换账户到devman
kubectl config use-context devman@kubernetes

# 查看dev下pod，发现没有权限
kubectl get pods -n dev

# 切换到admin账户
kubectl config use-context kubernetes-admin@kubernetes
```

**创建Role和RoleBinding，为devman用户授权**

> 创建dev-role.yaml

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: dev
  name: dev-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  
---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: authorization-role-binding
  namespace: dev
subjects:
- kind: User
  name: devman
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-role
  apiGroup: rbac.authorization.k8s.io
```

> 执行如下命令

```shell
kubectl create -f dev-role.yaml
```

**测试**

```shell
# 切换账户到devman
kubectl config use-context devman@kubernetes
# 再次查看
kubectl get pods -n dev
# 为了不影响后面的学习,切回admin账户
kubectl config use-context kubernetes-admin@kubernetes
```

### 准入控制

通过了前面的认证和授权之后，还需要经过准入控制处理通过之后，apiserver才会处理这个请求。

准入控制是一个可配置的控制器列表，可以通过在Api-Server上通过命令行设置选择执行哪些准入控制器：

```
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,
                      DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds
```

只有当所有的准入控制器都检查通过之后，apiserver才执行该请求，否则返回拒绝。

当前可配置的Admission Control准入控制如下：

- AlwaysAdmit：允许所有请求
- AlwaysDeny：禁止所有请求，一般用于测试
- AlwaysPullImages：在启动容器之前总去下载镜像
- DenyExecOnPrivileged：它会拦截所有想在Privileged Container上执行命令的请求
- ImagePolicyWebhook：这个插件将允许后端的一个Webhook程序来完成admission controller的功能。
- Service Account：实现ServiceAccount实现了自动化
- SecurityContextDeny：这个插件将使用SecurityContext的Pod中的定义全部失效
- ResourceQuota：用于资源配额管理目的，观察所有请求，确保在namespace上的配额不会超标
- LimitRanger：用于资源限制管理，作用于namespace上，确保对Pod进行资源限制
- InitialResources：为未设置资源请求与限制的Pod，根据其镜像的历史资源的使用情况进行设置
- NamespaceLifecycle：如果尝试在一个不存在的namespace中创建资源对象，则该创建请求将被拒绝。当删除一个namespace时，系统将会删除该namespace中所有对象。
- DefaultStorageClass：为了实现共享存储的动态供应，为未指定StorageClass或PV的PVC尝试匹配默认的StorageClass，尽可能减少用户在申请PVC时所需了解的后端存储细节
- DefaultTolerationSeconds：这个插件为那些没有设置forgiveness tolerations并具有notready:NoExecute和unreachable:NoExecute两种taints的Pod设置默认的“容忍”时间，为5min
- PodSecurityPolicy：这个插件用于在创建或修改Pod时决定是否根据Pod的security context和可用的PodSecurityPolicy对Pod的安全策略进行控制

## DashBoard

之前在kubernetes中完成的所有操作都是通过命令行工具kubectl完成的。其实，为了提供更丰富的用户体验，kubernetes还开发了一个基于web的用户界面（Dashboard）。用户可以使用Dashboard部署容器化的应用，还可以监控应用的状态，执行故障排查以及管理kubernetes中各种资源。

### 部署Dashboard

```shell
# 下载yaml
wget  https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
# 修改kubernetes-dashboard的Service类型
#kind: Service
#apiVersion: v1
#metadata:
#  labels:
#    k8s-app: kubernetes-dashboard
#  name: kubernetes-dashboard
#  namespace: kubernetes-dashboard
#spec:
#  type: NodePort  # 新增
#  ports:
#    - port: 443
#      targetPort: 8443
#      nodePort: 30009  # 新增
#  selector:
#    k8s-app: kubernetes-dashboard
# 部署
kubectl create -f recommended.yaml
# 查看namespace下的kubernetes-dashboard下的资源
kubectl get pod,svc -n kubernetes-dashboard
```

> 注意这里使用浏览器要使用火狐，用谷歌浏览器可能会有问题

![image-20230716212841040](./images/image-20230716212841040.png)

>  创建访问账户，获取token

```shell
# 创建账号
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
# 授权
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
# 查看相关密钥
kubectl get secrets -n kubernetes-dashboard | grep dashboard-admin
# 使用describe查看密钥中的token内容
kubectl describe secrets dashboard-admin-token-xtlcw -n kubernetes-dashboard
```

> 登录成功后即可看到kubernetes的信息

![image-20230716215326493](./images/image-20230716215326493.png)
