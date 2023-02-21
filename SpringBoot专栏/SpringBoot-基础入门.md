# SpringBoot基础入门

> SpringBoot是一个强大，并快捷的框架，能够快速搭建一个生产级别基于Spring的应用程序，相较于传统的SSM、SSH项目SpringBoot遵循约定大于配置使得构建项目时省去了繁琐的配置

## 项目构建

### 基于Maven构建

#### 快速构建

> 以下构建使用的是Idea构建构建，点击文件创建一个新的项目，使用spring官方构建工具进行构建

![image-20211101112320151](./images/image-20211101112320151.png)

> 填写项目名称，选择jdk版本号等信息

![image-20211101112550102](./images/image-20211101112550102.png)

> 下一步后会接入一个依赖选择界面，勾上对应依赖其实就是在`pom.xml`中加入该依赖，这里我选择不勾选，后期自己在`pom.xml`添加

![image-20211101112628011](./images/image-20211101112628011.png)

> 下一步后确定模块名称与项目根目录，点击Finish后完成构建

![image-20211101112934090](./images/image-20211101112934090.png)

#### 引入依赖

> 进入到pom文件中，引入web依赖

![image-20211101113144555](./images/image-20211101113144555.png)

#### 启动项目

> 在java包下可以看到一个xxxApplication的启动类，启动main方法即可

![image-20211101113306889](./images/image-20211101113306889.png)

> 当看到`Tomcat started`表示启动成功

![image-20211101113527386](./images/image-20211101113527386.png)

## 项目打包

> 项目打包有2钟方式，1是打jar包，2是打war包区别如下
>
> jar包：内置tomcat，轻量级启动方式只需要使用`jar -jar xxx.jar`即可，推荐打包方式
>
> war包：不内置tomcat，启动方式需要依赖web容器(如tomcat)，比较传统的打包方式

### jar包

#### 修改pom文件

> 在pom.xml钟的build中添加如下内容

![image-20211101114809755](./images/image-20211101114809755.png)

#### 执行打包命令

> 在idea的右侧，maven侧边栏中找到该项目，点击package即可打包

![image-20211101114937722](./images/image-20211101114937722.png)

> 打包成功后可以看到在target目录下生成了一个jar包

![image-20211101134224686](./images/image-20211101134224686.png)

### war包

#### 修改pom文件

> 需要完成如下3个操作
>
> 1. 在pom文件添加打包插件
> 2. 并且修改打包方式为war
> 3. 设置打包时排查内置tomcat依赖

![image-20211101134343042](./images/image-20211101134343042.png)

![image-20211101134446029](./images/image-20211101134446029.png)

![image-20211101134813221](./images/image-20211101134813221.png)

#### 修改主启动类

> 继承`SpringBootServletInitializer`类，实现configure方法

![image-20211101135007482](./images/image-20211101135007482.png)

#### 执行打包命令

> 在idea的右侧，maven侧边栏中找到该项目，点击package即可打包

![image-20211101135038965](./images/image-20211101135038965.png)

> 打包成功后可以看到在target目录下生成了一个war包

![image-20211101135118107](./images/image-20211101135118107.png)

### 特殊情况说明

#### 存在多个启动类

> 若项目存在多个启动类那么需要在pom中指定打包时的启动类

![image-20211101141842744](./images/image-20211101141842744.png)

## 配置切换

> 在开发过程中，通常一个项目会使用很多套不同的配置，如开发、测速、生产等不同环境下，连接的数据库，redis，端口等都不一样，以下就叫大家如何通过profile切换不同的配置

### yml或properties文件环境切换

> 该方式yml或properties文件格式都适用，也是现在使用最普遍的方法，将原`application.yml`拷贝出2分，注意命名规则需要遵守`application-xx.yml`

![image-20211101135732183](./images/image-20211101135732183.png)

#### dev与pro环境配置

> 配置好dev与pro环境中服务开发的端口分别是不同的端口

![image-20211101140009572](./images/image-20211101140009572.png)

#### 配置切换方法

> 切换配置的方式有3种

 **直接修改`application.yml`使用`spring.profiles.active=dev|pro`来指定**

![image-20211101140124039](./images/image-20211101140124039.png)

 **设置虚拟机参数`-Dspring.profiles.active=dev|pro来指定`**

![image-20211101140849719](./images/image-20211101140849719.png)

**使用jar命令启动是指定`java -jar xxxx.jar --spring.profiles.active=dev|pro`**

![image-20211101141257158](./images/image-20211101141257158.png)

### 配置读取

> 再开发过程中，有时会需要读取yml内容，来运行我们的程序

#### 修改yml

> 首先在`yml`中增加参数对象

![image-20211101144130163](./images/image-20211101144130163.png)

#### 编写读取配置类

> `@ConfigurationProperties(prefix = "person")`将指定前缀的参数映射带当前类的属性中

![image-20211101144225379](./images/image-20211101144225379.png)

#### 修改主启动类

> 在主启动类中将当前写好的配置类转载进入到容器中

![image-20211101144353125](./images/image-20211101144353125.png)

#### 测速

> 编写一个controller进行测试，通过`Autowired`将Person注入到Controller中

![image-20211101144442542](./images/image-20211101144442542.png)

> 从测试中可以看到在yml中的信息被成功读取，并且装入搭配Person对象中

![image-20211101144547444](./images/image-20211101144547444.png)

## Web开发

### webJars

> 官网：https://www.webjars.org/
>
> webJars让前端资源引入，可以让前端组件管理就像引入jar包一样容易只需要通过修改pom文件来引入前端资源

#### 引入资源

> 在非前后端分离项目时，开发过程中常常需要使用到前端的一些js组件如jquery，bootstrap，这些组件都是可以 通过webJar的形式引入的

![image-20211101150110537](./images/image-20211101150110537.png)

#### 测试

> 引入资源后重启项目后，可在webjars目录下访问到导入的前端资源

![image-20211101151345619](./images/image-20211101151345619.png)

#### 实现原理

> 实现原理其实是在MVC层对所有的`webjars`目录请求做拦截，返回拼接好的目录

![image-20211101150308277](./images/image-20211101150308277.png)

### 过滤器

> servlet三大利器之一过滤器的在SpringBoot的使用

#### 编写过滤器

> 编写一个类，继承Filter对象，实现doFilter方法

![image-20211101160539531](./images/image-20211101160539531.png)

#### 编写配置类

> 编写一个配置类，需要继承WebMvcConfigurerAdapter对象，将过滤器注入到容器当中

![image-20211101160731621](./images/image-20211101160731621.png)

#### 测试

> 只要访问任何请求都会执行过滤器中的doFilter方法

![image-20211101160749535](./images/image-20211101160749535.png)

### 拦截器

#### 编写拦截器

> 实现HandlerInterceptor接口，并且实现该接口下的3个方法

![image-20211102140314035](./images/image-20211102140314035.png)

#### 编写配置类型

> 配置类继承`WebMvcConfigurerAdapter`重写`addInterceptors`方法，从容器中获取到构建的拦截器传入到`registry`中

![image-20211102140359192](./images/image-20211102140359192.png)

#### 测试

![image-20211102140558961](./images/image-20211102140558961.png)

### 全局异常处理

> 全局统一异常处理是在web开发中一个很重要的功能，在前后端分离的项目下如果接口执行时发生异常那么不能直接返回保存内容给前端，需要自定义返回固定格式的Json数据，这就需要使用到全局统一异常处理

#### 编写自定义异常

> 编写一个自定义异常类，继承RuntimeException对父类构造方法进行增强

![image-20211102143841403](./images/image-20211102143841403.png)

#### 编写异常处理类

> `@ControllerAdvice`指定一个全局异常处理类，使用`@ExceptionHandler`指定处理那些异常类，返回自定义格式的json数据

![image-20211102143944504](./images/image-20211102143944504.png)

#### 测试

> 编写一个测试的Controller，抛出自定义的异常类

![image-20211102144130110](./images/image-20211102144130110.png)

> 发送请求时，可以看到返回结果被全局异常类捕获并处理了

![image-20211102144217327](./images/image-20211102144217327.png)

