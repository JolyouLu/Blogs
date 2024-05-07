# DISM.exe工具
平时在使用Win10时，有时会系统突然故障频繁死机等问题，可能很多小伙伴都想到需要重装，但是重装的代价太大了，软件需要重新下载，资料需要备份，如果在系统遇到频繁死机情况下可以先尝试使用DISM.exe工具进行修复，Windows中自带的DISM.exe工具（Deployment Imaging and Management 部署映像服务和管理）是一个强大的系统映像检查和修复工具
## 进入管理员命令行模式
![在这里插入图片描述](E:\one-drive-data\OneDrive\CSDN\Windos\images\20210328210657199.png)
## 扫描系统文件
> 扫描全部系统文件和系统映像文件是否与官方版一致
~~~shell
Dism /Online /Cleanup-Image /ScanHealth
~~~
![在这里插入图片描述](E:\one-drive-data\OneDrive\CSDN\Windos\images\20210328211136949.png)

## 检查文件的损坏程度
> 如果在扫描系统文件时检测到组件损坏，需要执行如下命令检查文件的损坏程度，结果一般有两种：一般性损坏可以修复和情况严重完全无法修复
~~~shell
Dism /Online /Cleanup-Image /CheckHealth
~~~
## 联网替换损坏文件
>如果是一般性损坏可以修复那么就需要执行如下命令，联网将系统映像文件中与官方文件不相同的文件还原成官方系统的源文件，系统中安装的软件和用户设置保留不变，比重装系统要好得多，而且在修复时系统未损坏部分可以正常运行，不耽误大家手头上的事情
>`如果是严重完全无法修复也可以尝试一下执行如下命令，如果问题没解决，除了重装系统外，没有太好的办法解决`
~~~shell
DISM /Online /Cleanup-image /RestoreHealth
~~~
