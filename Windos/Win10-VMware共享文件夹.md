# VMware共享文件夹

> 使用VMware共享文件夹可以将宿主机上的一个文件夹与虚拟机中实现共享，可以实现文件的快速传输

**安装VMware Tools**

> 在使用文件夹共享前确保虚拟机已经安装了VMware Tools（通常情况下在安装时会自动安装）

**设置共享文件夹**

![image-20231128232348405](E:\one-drive-data\OneDrive\CSDN\Windos\images\image-20231128232348405.png)

## Linux访问共享文件夹

> 执行以下命令查看当前宿主机有什么文件夹共享了

~~~shell
#查看当前共享的文件夹
vmware-hgfsclient
~~~

![image-20231128232824451](E:\one-drive-data\OneDrive\CSDN\Windos\images\image-202311282328244511.png)

> 修改`/etc/fstab`文件实现自动挂载文件夹

~~~shell
#创建一个目录
mkdir /data/mysql-data
#修改/etc/fstab
vim /etc/fstab
#添加如下内容(这样默认挂载出来的目录和文件都是归属root)
.host:/Mysql-Data /data/mysql-data fuse.vmhgfs-fuse allow_other,defaults 0 0

#指定归属用户挂载(使用uid和gid指定挂载文件归属的用户id和组id)
.host:/Mysql-Data /data/mysql-data fuse.vmhgfs-fuse allow_other,uid=1000,gid=1000,defaults 0 0
#使用如下命令可以查看用户id和组id
id 用户名
~~~

> 设置完成后使用`mount -a`刷新，或者也可以使用`reboot`重启linux，重启完毕后即可看到目录