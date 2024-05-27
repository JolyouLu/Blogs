# Kind-基础

## 介绍

kind是一个使用 Docker 容器“节点”运行本地 Kubernetes 集群的工具。它主要用于测试 Kubernetes 本身，但也可用于本地开发或CI。顾名思义，就是将 Kubernetes 所需要的所有组件，全部部署在一个 Docker 容器中，可以很方便的搭建 Kubernetes 集群。

将 docker 容器作为一个 kubernetes 的 "node"，并在该 "node" 中安装 kubernetes 组件

Kind 使用一个 container 来模拟一个 node，在 container 里面跑了 systemd ，并用 systemd 托管了 kubelet 以及 containerd，然后容器内部的 kubelet 把其他 Kubernetes 组件，比如 kube-apiserver，etcd，cni 等组件跑起来。
可以通过配置文件的方式，来通过创建多个 container 的方式，来模拟创建多个 Node，并以这些 Node 来构建一个多节点的 Kubernetes 集群。

## 常用操作

### 创建集群

以下命令是使用默认配置创建一个单节点集群

~~~shell
#创建一个默认集群
kind create cluster
#指定配置文件创建集群
kind create cluster --config kind-config.yaml
#创建成功后，假如有多个kind集群可以使用kubectl切换k8s集群
kubectl cluster-info --context kind-你的集群名称
~~~

### 切换集群

~~~shell
#获取当前的集群信息
kind get clusters
#切换集群
kubectl cluster-info --context kind-集群名称
~~~
### 删除集群

~~~shell
#获取当前的集群信息
kind get clusters
#删除指定集群名称
kind delete cluser --name 集群名称
~~~

### 进入集群内部

~~~shell
#查看容器
docker ps
#进入到kind集群内部
docker exec -it 集群名称-control-plane /bin/bash
#集群内部没有docker使用crictl命令
crictl ps
~~~

### 将docker镜像load到集群

~~~shell
#获取当前的集群信息
kind get clusters
#将镜像加载到集群中
kind load docker-image 镜像名称:版本 --name 集群名称
#查看集群内部镜像文件（集群内部使用crictl命令）
docker exec -it 集群名称-control-plane crictl images
~~~

## 案例

环境说明：本人笔记本内安装了VMware，在Vmware中安装了Linux Centos7，并且在Linux中搭建Kind机器，本人笔记本与Vmware的Linux使用NAT网络[Win10-VMware网络配置NAT模式](../Win10-VMware网络配置NAT模式)

准备一个kind集群，并且部署nginx开放端口30000，并且本人笔记本可以访问

### 集群配置文件

~~~yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: my-cluster                                                            # 集群名称
nodes:
  - role: control-plane                                                     # 单节点部署
    extraPortMappings:                                                      # 端口映射
      - containerPort: 30000                                                # 容器端口：30000（Nginx）
        hostPort: 30000                                                     # 宿主机端口：30000（Nginx）
        protocol: TCP
    extraMounts:                                                            # 挂载目录
      - hostPath: /data                                                     # 宿主机：数据文件路径
        containerPath: /data                                                # 容器：数据文件路径
      - hostPath: /data/k8s-data/kind-config/etc/containerd/config.toml     # 宿主机：crictl配置
        containerPath: /etc/containerd/config.toml                          # 容器：crictl配置
~~~

> /etc/containerd/config.toml 映射是用于镜像加速的，文件内容可以看`常见问题=>镜像下载失败`

### Nginx配置文件

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: dev
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30000
  type: NodePort
~~~

### 执行命令

~~~shell
#创建kind集群
kind create cluster --config kind-config/kind-config.yaml
#kubectl切换k8s集群环境， 如果只有一个不用切换
kubectl cluster-info --context kind-集群名称
#创建命名空间dev
kubectl create ns dev
#创建nginx pod
kubectl apply -f nginx.yaml
#查看状态
kubectl get deploy,pods,svc -n dev
~~~

> 测试访问nginx

![image-20240522174123869](E:\one-drive-data\OneDrive\CSDN\Docker专栏\images\image-20240522174123869.png)

## 常见问题

### Ensuring node image (kindest/node:v1.25.3) 提示下载失败

安装成功会看到如下内容，首次搭建可能会卡在`Ensuring node image (kindest/node:v1.25.3)`提示网络无法访问

解决方案：多次尝试就可以下载成功了

![image-20240522011220622](E:\one-drive-data\OneDrive\CSDN\Docker专栏\images\image-20240522011220622.png)

### Preparing nodes步骤提示Error

假如在安装过程中出现以下错误表示kind包与Linux内核有关，本人在Centos7安装Kind最新版发生过这个问题，好像和RedHat7中的cgroups有关

解决方案：切换到v0.17.0的v0.17.0就可以成功安装了

![image-20240522164359808](E:\one-drive-data\OneDrive\CSDN\Docker专栏\images\image-20240522164359808.png)

### 镜像下载失败

在kind搭建的k8s集群中拉取镜像使用的是crictl拉取镜像，并且默认的拉取地址是`https://registry.k8s.io, https://k8s.gcr.io`国内有时无法访问导致拉取失败

![image-20240522165301037](E:\one-drive-data\OneDrive\CSDN\Docker专栏\images\image-20240522165301037.png)

> 将config.toml从kin集群中拷贝出来

~~~shell
#查看部署的kind容器，name通常是，集群名称-control-plane
docker ps
#将容器内config.toml拷贝到当前目录中
docker cp 集群名称-control-plane:/etc/containerd/config.toml config.toml
~~~

> 使用vim命令在最后面添加上阿里云镜像地址，内容如下

~~~to
# 阿里云镜像地址
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["https://pztjnors.mirror.aliyuncs.com"]
~~~

解决方案1：

~~~shell
#查看部署的kind容器，name通常是，集群名称-control-plane
docker ps
#使用dockercp命令将config.toml替换进去
docker cp config.toml 集群名称-control-plane:/etc/containerd/config.toml
#退出到外面，重启容器
docker restart 集群名称-control-plane
~~~

解决方案2：

集群创建时将准备好config.toml映射进去

![image-20240522172347216](E:\one-drive-data\OneDrive\CSDN\Docker专栏\images\image-20240522172347216.png)
