## Redis.conf详解

### 配置说明

> redis对配置文件对大小写不敏感

![image-20201205093302544](./images/image-20201205093302544.png)

### 引入配置文件（INCLUDES）

> redis可以通过引入的方式引入多个配置文件

![image-20201205093356495](./images/image-20201205093356495.png)

### 网络配置（NETWORK）

> 绑定的IP地址

![image-20201205093611602](./images/image-20201205093611602.png)

> 保护模式（默认开启）

![image-20201205093702919](./images/image-20201205093702919.png)

> Redis端口

![image-20201205093737381](./images/image-20201205093737381.png)

### 通用配置（GENERAL）

> 以守护进程方式运行（默认是no）

![image-20201205093841676](./images/image-20201205093841676.png)

> 如果以守护进程方式运行，需要指定一个PID进程文件

![image-20201205093956602](./images/image-20201205093956602.png)

> 日志 debug测试环境使用 notice生产环境使用

![image-20201205094018071](./images/image-20201205094018071.png)

> 日志生成的文件位置名，如果空只是控制台输出

![image-20201205094216399](./images/image-20201205094216399.png)

> 数据库数量 默认16

![image-20201205094259698](./images/image-20201205094259698.png)

> 是否显示Redis LOGO

![image-20201205094345585](./images/image-20201205094345585.png)

### 快照设置（SNAPSHOTTING）

> 持久化时需要设置在规定时间内有多少次操作会持久化，保存到持久化文件分.rdb和.aof

~~~shell
#如果900秒内至少有1个key修改，就进行持久化操作
save 900 1 
#如果300秒内至少有10个key修改，就进行持久化操作
save 300 10 
#如果60秒内至少有1万个key修改（高并发），就进行持久化操作
save 60 10000 
~~~

![image-20201205094547722](./images/image-20201205094547722.png)

> 持久化出错是否继续工作

![image-20201205094921663](./images/image-20201205094921663.png)

> 是否压缩rdb文件，即持久化文件，会消耗CPU资源

![image-20201205094950811](./images/image-20201205094950811.png)

> 保存rdb文件的时候继续错误的检测，校验

![image-20201205095041368](./images/image-20201205095041368.png)

> rdb文件保存的目录

![image-20201205095113835](./images/image-20201205095113835.png)

### 主从复制（REPLICATION）

> 指定主机的IP与端口号

![image-20201205133627357](./images/image-20201205133627357.png)

### 安全配置（SECURITY）

> 连接密码，默认是不需要密码的，可以通过“requirepass 123456”设置连接密码为”123456“

~~~shell
#auth命令登录redis
auth [password]
~~~

![image-20201205095341298](./images/image-20201205095341298.png)

### 客户端限制（CLIENTS）

> 最大客户端连接数

![image-20201205095807271](./images/image-20201205095807271.png)

### 内存配置（MEMORY MANAGEMENT）

> redis最大内存容量

![image-20201205095858245](./images/image-20201205095858245.png)

> 内存达到上限的处理策略
>
> 1、volatile-lru：只对设置了过期时间的key进行LRU（默认值）
> 2、allkeys-lru ： 删除lru算法的key
> 3、volatile-random：随机删除即将过期key
> 4、allkeys-random：随机删除
> 5、volatile-ttl ： 删除即将过期的
> 6、noeviction ： 永不过期，返回错误

![image-20201205095956782](./images/image-20201205095956782.png)

### AOF配置（APPEND ONLY MODE）

> 默认不开启aof模式，默认使用rdb方式持久化，大部分情况下rdb够用

![image-20201205100350997](./images/image-20201205100350997.png)

> aof持久化文件名字

![image-20201205100442121](./images/image-20201205100442121.png)

> 同步设置

~~~shell
#每次修改都会 sync，消耗性能
appendfsync always 
#每次执行一次 sync，可能会丢失这ls的数据
appendfsync everysec
#不执行 sync，操作系统自己同步数据
appendfsync no
~~~

![image-20201205100513711](./images/image-20201205100513711.png)