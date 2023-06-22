# Maven-下载依赖失败



> Maven在下载仓库中找不到相应资源时，会生成一个.lastUpdated为后缀的文件。这个文件的存在导致了无法更新获取jar，我们只需要进入jar包目录删除所有.lastUpdated结尾文件即可

Lunix下解决办法

~~~shell
find ~/.m2 -name “*.lastUpdated” -exec grep -q “Could not transfer” {} ; -print -exec rm {} ;
~~~

Win下解决办法

~~~shll
cd %userprofile%.m2\repository
for /r %i in (*.lastUpdated) do del %i
~~~

