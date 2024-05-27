# Win10-Hadoop环境准备

在日常的大数据开发调试中通常首先在本地电脑上运行少量数据测试在提交到生产环境上运行，以下简介如何在Win上安装Hadoop的本地环境

## 安装包下载

Win上安装Hadoop需要下载Hadoop以及winutils

Hadoop：https://hadoop.apache.org/releases.html

winutils：https://github.com/steveloughran/winutils

## Hadoop环境变量配置

将下载好的Hadoop解压到目录后配置相应的环境变量

![image-20240517142354758](E:\one-drive-data\OneDrive\CSDN\Windos\images\image-20240517142354758.png)

创建环境变量后找到path，把创建的环境变量添加到全局

![image-20240517142449851](E:\one-drive-data\OneDrive\CSDN\Windos\images\image-20240517142449851.png)

## winutils配置

由于Hadoop没有Win版本的，所以需要winutils配合才能使得Hadoop在win上运行

进入到winutils文件夹中找到对应Hadoop版本我是3.0.0以后的所以我进去的是`winutils\hadoop-3.0.0\bin`，然后将其中的`hadoop.dll`和`winutils.exe`拷贝到下载的Hadoop3.1.1的bin目录中

然后双机执行以下`winutils.exe`看到一个黑色窗口一闪而过表示执行成功

![image-20240517142726779](E:\one-drive-data\OneDrive\CSDN\Windos\images\image-20240517142726779.png)
