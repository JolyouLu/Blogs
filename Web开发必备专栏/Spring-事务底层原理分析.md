

# Spring 事务底层原理分析

## 数据库的事务的基本特性

事务是区分文件存储系统与Nosql数据库重要特性之一，其存在的意义是为了保证即使在并发情况下也能正确的执行crud操作。怎样才算正确的呢?这时提出了事务需要保证的算个特性即ACID

### A：原子性（atomicity）

事务中各项的操作，要么全做要么全不做，任何一项操作的失败都会导致整个事务的失败

### C：一致性（consistency）

事务结束后与系统状态是一致的

### I：隔离性（isolation）

并发执行的事务彼此无法看到对方的中间状态

### D：持久性（durability）

事务完成后做的改动都会被持久化，即使发生灾难性的失败

在高并发的情况下，要完全保证其ACID特性是非常困难的，除非把所有的事务串行化执行，但带来的负面的影响将是性能大打折扣。很多时候我们有些业务对事务的要求是不一样的，所以数据库中设计了四种隔离级别，供用户基于业务进行选择。

| **隔离级别**                 | **脏读（Dirty Read）** | **不可重复读（NonRepeatable Read）** | **幻读（Phantom Read）** |
| ---------------------------- | ---------------------- | ------------------------------------ | ------------------------ |
| 未提交读（Read uncommitted） | 可能                   | 可能                                 | 可能                     |
| 已提交读（Read committed）   | 不可能                 | 可能                                 | 可能                     |
| 可重复读（Repeatable read）  | 不可能                 | 不可能                               | 可能                     |
| 可串行化（SERIALIZABLE）     | 不可能                 | 不可能                               | 不可能                   |

**脏读 :**

对应的演示类com.spring.tx.ReadCommittedExample

一个事务读取到另一事务未提交的更新数据

**不可重复读 :** 

对应的演示类com.spring.tx.ReadUncommittedExample

在同一事务中,多次读取同一数据返回的结果有所不同, 换句话说, 后续读取可以读到另一事务已提交的更新数据. 相反, “可重复读”在同一事务中多次读取数据时, 能够保证所读数据一样, 也就是后续读取不能读到另一事务已提交的更新数据。

**幻读 :**

对应的演示类com.spring.tx.SerializableExample

查询表中一条数据如果不存在就插入一条，并发的时候却发现，里面居然有两条相同的数据。这就幻读的问题。 

### 小知识

我们几个几个类的演示我们可以发现其实我们执行了insert语句后没有提交事务我们的表已经插入了这条语句了，但是我们不对事务commit这个语句又会被删除，其实我们每一个事务他都有一条回滚的语句，如一个insert事务中那就会有insert语句对应的delete语句，如果事务的在途有什么问题他就会执行delete语句把原来已经插入的数据delete

Oracle中默认级别是 Read committed

mysql 中默认级别 Repeatable read

~~~sql
-- 查看mysql 的默认隔离级别
SELECT @@tx_isolation
~~~

## Spring 对事务的支持与使用

### spring事务相关API说明

spring 事务是在数据库事务的基础上进行封装扩展 其主要特性如下

1.  支持原有的数据事务的隔离级别

2. 加入了事务传播的概念 提供多个事务的和并或隔离的功能

3. 提供声明式事务，让业务代码与事务分离，事务变得更易用

怎么样去使用Spring事务呢？spring 提供了三个接口供使用事务。分别是：

* TransactionDefinition
  * 事务定义

* PlatformTransactionManager
  * 事务管理

* TransactionStatus
  * 事务运行时状态

#### spring事务API例子

~~~java
public class SpringTransaction {
    private static String url = "jdbc:mysql://localhost:3306/study?characterEncoding=utf8&useSSL=true";
    private static String user = "root";
    private static String password = "123456";

    public static Connection openConnection() throws SQLException, ClassNotFoundException {
        Class.forName("com.mysql.jdbc.Driver");
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/study?characterEncoding=utf8&useSSL=true", "root", "123456");
        return conn;
    }

    public static void main(String[] args) {
        //获取一个数据源
        final DriverManagerDataSource ds = new DriverManagerDataSource(url, user, password);
        //创建一个事务模板
        final TransactionTemplate template = new TransactionTemplate();
        //为该模板设置一个事务管理器
        template.setTransactionManager(new DataSourceTransactionManager(ds));

        template.execute(new TransactionCallback<Object>() {
            @Override
            public Object doInTransaction(TransactionStatus transactionStatus) {
                Connection conn = DataSourceUtils.getConnection(ds);
                Object savePoint = null;
                try {
                    {
                        //插入
                        PreparedStatement prepare = conn.prepareStatement("insert INTO account (accountName,user,money) VALUES (?,?,?)");
                        prepare.setString(1, "111");
                        prepare.setString(2, "aaaa");
                        prepare.setInt(3, 10000);
                        prepare.executeUpdate();
                    }
                    // 设置保存点 如果后面有错误事务会 回滚到保存点 提交保存到中的内容
                    savePoint = transactionStatus.createSavepoint();
                    {
                        // 插入
                        PreparedStatement prepare = conn.prepareStatement("insert INTO account (accountName,user,money) VALUES (?,?,?)");
                        prepare.setString(1, "222");
                        prepare.setString(2, "bbb");
                        prepare.setInt(3, 10000);
                        prepare.executeUpdate();
                    }
                    {
                        // 更新
                        PreparedStatement prepare = conn.
                                prepareStatement("UPDATE account SET money= money+1 where user=?");
                        prepare.setString(1, "asdflkjaf");
                        //让程序执行到此处报错，演示事务回滚
                        int i=1/0;
                    }
                } catch (SQLException e) {
                    e.printStackTrace();
                }catch (Exception e) {
                    if (savePoint != null) {
                        System.out.println("更新失败,回滚到保存点提交事务");
                        transactionStatus.rollbackToSavepoint(savePoint);
                    } else {
                        System.out.println("更新失败");
                        transactionStatus.setRollbackOnly();
                    }
                }
                return null;
            }
        });
    }
}
~~~

### 声明式事务

我们前面是通过调用API来实现对事务的控制，这非常的繁琐，与直接操作JDBC事务并没有太多的改善，所以Spring提出了声明式事务，使我们对事务的操作变得非常简单，甚至不需要关心它

#### 配置文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        http://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <context:annotation-config/>
    <context:component-scan base-package="com.spring.service.*"/>

    <aop:aspectj-autoproxy expose-proxy="true"/>

    <!--把数据源配入jdbc模板-->
    <bean class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!--注入TransactionManager -->
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!--配置数据源-->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <constructor-arg name="url" value="jdbc:mysql://localhost:3306/study?useSSL=true"/>
        <constructor-arg name="username" value="root"/>
        <constructor-arg name="password" value="123456"/>
    </bean>

    <!--最重要的一步：把TransactionManager注入到事务的驱动中-->
    <tx:annotation-driven transaction-manager="txManager"></tx:annotation-driven>

</beans>
~~~

只需要在你想开启事务的地方加上@Transactional表示开启事务

~~~java
@Override
@Transactional //声明开启事务
public void createUser(String name) {
    // 插入user 记录
    jdbcTemplate.update("INSERT INTO `user` (name) VALUES(?)", name);
    // 调用 accountService 添加帐户
    accountService.addAccount(name, 10000);
}
~~~

### 事务传播机制

| **事物传播类型**                    | **说明**                                                     |
| ----------------------------------- | ------------------------------------------------------------ |
| PROPAGATION_REQUIRED  （默认的）    | 如果当前没有事物，就新建一个事物，如果已经存在一个事物中，加入到这个事物中。这是最常见的选择。 |
| PROPAGATION_SUPPORTS  （支持）      | 支持当前事物，如果当前没有事物，就以非事物方式执行。         |
| PROPAGATION_MANDATORY  （强制）     | 使用当前的事物，如果当前没有事物，就抛出异常。               |
| PROPAGATION_REQUIRES_NEW  (隔离)    | 新建事物，如果当前存在事物，把当前事物挂起。                 |
| PROPAGATION_NOT_SUPPORTED  (不支持) | 以非事物方式执行操作，如果当前存在事物，就把当前事物挂起。   |
| PROPAGATION_NEVER  (强制非事物)     | 以非事物方式执行，如果当前存在事物，则抛出异常。             |
| PROPAGATION_NESTED  （嵌套事物）    | 如果当前存在事物，则在嵌套事物内执行。如果当前没有事物，则执行与PROPAGATION_REQUIRED类似的操作。 |

#### 常用事物传播机制

* PROPAGATION_REQUIRED
  * 这个也是默认的传播机制；

* PROPAGATION_NOT_SUPPORTED
  * 可以用于发送提示消息，站内信、短信、邮件提示等。不属于并且不应当影响主体业务逻辑，即使发送失败也不应该对主体业务逻辑回滚。

* PROPAGATION_REQUIRES_NEW
  * 总是新启一个事物，这个传播机制适用于不受父方法事物影响的操作，比如某些业务场景下需要记录业务日志，用于异步反查，那么不管主体业务逻辑是否完成，日志都需要记录下来，不能因为主体业务逻辑报错而丢失日志；

#### 演示事务的传播性

首先我们需要准备2个接口 2个实现Service

AccountService接口

~~~java
@Service
public interface AccountService {
    //添加账户方法
    void addAccount(String name, int initMoney);
}
~~~

UserService接口

~~~java
@Service
public interface UserService {
    //创建user方法
    void createUser(String name);
}
~~~

AccountService接口实现类 添加一个账户 并且设置了一段人为报错

~~~java
@Service
public class AccountServiceImpl implements AccountService {
    @Autowired
    JdbcTemplate jdbcTemplate;

    @Override
    //    @Transactional(propagation = Propagation.REQUIRED) //场景一 开启
    //场景二无事务
    //    @Transactional(propagation = Propagation.NOT_SUPPORTED) //场景三开启
    //    @Transactional(propagation = Propagation.REQUIRES_NEW) //场景四、五开启
    public void addAccount(String name, int initMoney) {
        String accountid = new SimpleDateFormat("yyyyMMddhhmmss").format(new Date());
        jdbcTemplate.update("insert INTO account (accountName,user,money) VALUES (?,?,?)", accountid, name, initMoney);
        // 人为报错 场景五注释
        int i = 1 / 0;
    }
}
~~~

UserService接口实现类 插入一个user 并且同时会添加一个账户

~~~java
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    JdbcTemplate jdbcTemplate;

    @Autowired
    AccountService accountService;

    @Override
    //场景一 无事务
    //    @Transactional(propagation = Propagation.REQUIRED) //场景二、三、四、五 开启
    public void createUser(String name) {
        // 插入user 记录
        jdbcTemplate.update("INSERT INTO `user` (name) VALUES(?)", name);
        // 调用 accountService 添加帐户
        accountService.addAccount(name, 10000);
        // 人为报错 场景五开启
        //        int i = 1 / 0;
    }
}
~~~

测试方法

~~~java
public class UserServiceImplTest {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context =
            new ClassPathXmlApplicationContext("spring-tx.xml");
        UserService service = context.getBean(UserService.class);
        service.createUser("lzj");
    }
}
~~~

每次跑完代码查询表中结果

~~~sql
SELECT * FROM `user`;
SELECT * FROM `account`;

--测试下一个场景时需要执行如下语句把表清空
DELETE FROM `user`;
DELETE FROM `account`;
~~~

场景实验

|        | createUser                   | addAccount                        | 结果                               |
| ------ | ---------------------------- | --------------------------------- | ---------------------------------- |
| 场景一 | 无事务<br />(方法中无异常)   | required<br />(方法中有异常)      | user(成功)<br />Account（不成功)   |
| 场景二 | required<br />(方法中无异常) | 无事物<br />(方法中有异常)        | user(不成功)<br />Account（不成功) |
| 场景三 | required<br />(方法中无异常) | not_supported<br />(方法中有异常) | user(不成功)<br />Account（成功)   |
| 场景四 | required<br />(方法中无异常) | required_new<br />(方法中有异常)  | user(不成功)<br />Account（不成功) |
| 场景五 | required<br />(方法中有异常) | required_new<br />(方法中有异常)  | user(不成功)<br />Account（成功    |

