# 实现自己的Mybatis

大家如果不想自己写可以从gitHub下载下来自己看一下

地址：https://github.com/JolyouLu/mybatis01.git 代码在copy.mybatis下

## 前期准备

经过小白架构师成长之路9-Mybatis核心概念与基础配置 大家都基本了解Mybatis运行原理了把现在我们自己来手动写一个mybatsi首先我们需要准备的一些包和类

* copy.mybatis
  * binding      =>存放Mapper相关的类
    * MapperMethod.class                 =>用于存放sql语句和返回值类型的类
    * MapperProxy.class                    =>Mapper的动态代理类
    * MapperRegistry.class                =>用于Mappers注册的类
  * executor    =>存放执行器的类
    * Executor.interface                      =>执行器的接口
    * SimpleExecutor.class                 =>执行器的其中一种实现类
  * result         => 反射相关的类
    * ResultSetHandler.interface         =>反射赋值的接口
    * DefaultResultSetHandler.class    =>实现反射赋值的类
  * session     => 开放给程序员调用的一些接口
    * Configuration.class                      =>用于加载mybatis配置文件的类
    * SqlSession.interface                    =>SqlSession的接口
    * DefaultSqlSession.class               =>SqlSession的接口的默认实现类
    * SqlSessionFactory.class               =>用于返回我们的SqlSession对象
    * SqlSessionFactoryBuilder.class    =>用于返回我们的SqlSessionFactory对象
  * statement  => 具体调用数据库的类
    * StatementHandler.class                 =>StatementHandler数据库操作类
  * util             =>jdbc数据库连接的工具类
    * DbUtil                                              =>连接数据的工具类
  * test            =>我们测试的文件夹
    * mapper
      * UserMapper.interface
    * pojo 
      * User
* resources     =>存放Mapper.xml和配置文件的目录
  * copyMybatis
    * MyUserMapper.xml                          =>对应mapper文件夹下UserMapper接口中的sql
  * myconfig.xml                                           =>我们自己定义的mybatis配置文件

## 测试类的编写

### myconfig.xml

这个配置文件是我们自己写的，因为读取是我们自己的类读的所以按照我们喜欢的格式自己写一个配置文件

~~~xml
<?xml version="1.0" encoding="UTF-8" ?>
<inpitStream>
    <properties>
        <property name="deviceClass" value="com.mysql.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/study?useSSL=true"/>
    </properties>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${deviceClass}"/>
                <property name="url" value="${url}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="copyMybatis/MyUserMapper.xml"/>
    </mappers>
</inpitStream>
~~~

### UserMapper.interface

~~~java
public interface UserMapper {
    public User getUser(Integer id);
}
~~~

### User.class

~~~java
public class User implements Serializable {
    private Integer id;
    private String username;
    private Integer age;
    private String desc;
    //省略get set 方法
}
~~~

### MyUserMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?><mapper namespace="com.lzj.copy.mybatis.test.mapper.UserMapper">    <select id="getUser" resultType="com.lzj.copy.mybatis.test.pojo.User">    select * from user where id = %d  </select></mapper>
```

### 测试接口

~~~java
public static void main(String[] args) throws IOException {
    InputStream inputStream = TestMysqlUtil.class.getClassLoader().getResourceAsStream("myconfig.xml");
    Configuration configuration = new Configuration();
    //把流传入到Configuration对象中
    configuration.setInputStream(inputStream);
    //把配置文件读取配置文件返回sqlSessionFactory
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
    //开启事务，获取加载器，返回sqlSession
    DefaultSqlSession sqlSession = (DefaultSqlSession) sqlSessionFactory.openSession(configuration);
    //调用加载器
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    User user = userMapper.getUser(1);
    System.out.println(user);
}
~~~

## 编写我们自己的Mybatis

### Configuration

我们从测试方法可以看到最先需要创建一个Configuration类用于存放我们的配置文件信息所以我们先实现Configuration类，该类下有一个方法loadConfigurations()方法用于读取配置文件

~~~java
public class Configuration {
    private InputStream inputStream;
    //new一个MapperRegistry 把从Configuration加载的mapper添加到MapperRegistry中
    MapperRegistry mapperRegistry = new MapperRegistry();

    /*通过Dom4j读取配置文件信息*/
    public void loadConfigurations() throws IOException{
        try {
            //读取文件转成dom
            Document document = new SAXReader().read(inputStream);
            //获取根标签
            Element root = document.getRootElement();
            //获取根标签下的mappers中的mapper 返回list
            List<Element> mappers = root.element("mappers").elements("mapper");
            //遍历遍历这些mapper
            for (Element mapper : mappers) {
                //遍历mapper获取如果属性名为resource获取其中的值
                if (mapper.attribute("resource")!=null){
                    mapperRegistry.setKnownMappers(loadXMLConfiguration(mapper.attribute("resource").getText()));
                }
                //遍历mapper获取如果属性名为class获取其中的值
                if (mapper.attribute("class") != null){
                }
            }
        }catch (Exception e){
            System.out.println("读取配置文件错误！");
        }finally {
            inputStream.close();
        }
    }

    /*通过dom4j读取mapper.xml中的信息*/
    private Map<String, MapperMethod> loadXMLConfiguration(String resource) throws DocumentException, IOException {
        Map<String,MapperMethod> map = new HashMap<>();
        InputStream is =null;
        try {
            is = this.getClass().getClassLoader().getResourceAsStream(resource);
            Document document = new SAXReader().read(is);
            Element root = document.getRootElement();
            if (root.getName().equalsIgnoreCase("mapper")){
                //获取namespace中的值
                String namespace = root.attribute("namespace").getText();
                //遍历获取xml下的全部 sql和返回值类型 put到map中
                for (Element select:(List<Element>) root.elements("select")){
                    MapperMethod mapperMethod = new MapperMethod();
                    //获取sql语句
                    mapperMethod.setSql(select.getText().trim());
                    //获取返回值类型
                    mapperMethod.setType(Class.forName(select.attribute("resultType").getText()));
                    //namespace+id+mapperMethod 存入到map中一级缓存的时候 如果有重复的namespace+id+mapperMethod的sql语句不访问数据库直接返回参数
                    map.put(namespace+"."+select.attribute("id").getText(),mapperMethod);
                }
            }
        }catch (ClassNotFoundException e){
            e.printStackTrace();
        }finally {
            is.close();
        }
        return map;
    }
    public InputStream getInputStream() {
        return inputStream;
    }
    public void setInputStream(InputStream inputStream) {
        this.inputStream = inputStream;
    }
    public MapperRegistry getMapperRegistry() {
        return mapperRegistry;
    }
    public void setMapperRegistry(MapperRegistry mapperRegistry) {
        this.mapperRegistry = mapperRegistry;
    }
}

~~~

### MapperMethod

configuration类loadXMLConfiguration方法中我们需要把获取获取到的sql语句返回的类型传入该对象

~~~java
public class MapperMethod<T> {
    //sql语句
    private String sql;
    //返回的类
    private Class<T> type;
    public MapperMethod() {
    }
    public MapperMethod(String sql, Class<T> type) {
        this.sql = sql;
        this.type = type;
    }
    public String getSql() {
        return sql;
    }
    public void setSql(String sql) {
        this.sql = sql;
    }
    public Class<T> getType() {
        return type;
    }
    public void setType(Class<T> type) {
        this.type = type;
    }
}
~~~

### MapperRegistry

该类是为每一个MapperMethod对象添加一个key，以后通过这个key去获取对应的sql和返回值类型

~~~java
public class MapperRegistry {
    private Map<String, MapperMethod> knownMappers = new HashMap<String,MapperMethod>();
    public Map<String, MapperMethod> getKnownMappers() {
        return knownMappers;
    }
    public void setKnownMappers(Map<String, MapperMethod> knownMappers) {
        this.knownMappers = knownMappers;
    }
}
~~~

### SqlSessionFactoryBuilder

对配置文件继续builder的类是这个 在这个类中调用了configuration.loadConfigurations()的方法

~~~java
public class SqlSessionFactoryBuilder {
    //加载配置文件 返回SqlSessionFactory对象
    public SqlSessionFactory build(Configuration configuration) throws IOException {
        //读取xml
        configuration.loadConfigurations();
        return new SqlSessionFactory();
    }
}
~~~

### SqlSessionFactory

得到SqlSessionFactory我们使用openSession来得到SqlSession

~~~java
public class SqlSessionFactory {
    //返回SqlSession 为了简便我们这里直接自己new一个执行器给SqlSession
    public SqlSession openSession(Configuration configuration){
        return new DefaultSqlSession(configuration,new SimpleExecutor(configuration));
    }
}
~~~

### SqlSession

SqlSession接口 里面有selectOne方法

~~~java
public interface SqlSession {
    <T> T selectOne(MapperMethod mapperMethod, Object statement) throws Exception;
}
~~~

### DefaultSqlSession

DefaultSqlSession实现了SqlSession的接口重写了selectOne方法

~~~java
public class DefaultSqlSession implements SqlSession{
    //加载好的配置文件
    private Configuration configuration;
    //执行器
    private Executor executor;
    public Configuration getConfiguration() {
        return configuration;
    }
    public DefaultSqlSession(Configuration configuration, Executor executor) {
        this.configuration = configuration;
        this.executor = executor;
    }
    //使用动态代理 代理给MapperProxy
    public <T> T getMapper(Class<T> type){
        return (T)Proxy.newProxyInstance(type.getClassLoader(),new Class[]{type},new MapperProxy<>(this,type));
    }
    @Override //实现SqlSession接口种的 selectOne方法
    public <T> T selectOne(MapperMethod mapperMethod,Object statement) throws Exception {
        return (T) executor.query(mapperMethod,statement);
    }
}
~~~

### MapperRegistry

在调用DefaultSqlSession中的getMapper方法会被代理给MapperRegistry中的invoke方法类获取对应的MapperMethod，并且把他传回给sqlSession.selectOne方法，这里我们可以发现其实通过gietMapper最后还是调用的selectOne方法

~~~java
public class MapperProxy<T> implements InvocationHandler {

    private final DefaultSqlSession sqlSession;
    private final Class<T> mapperInterface;

    public MapperProxy(DefaultSqlSession sqlSession, Class<T> mapperInterface) {
        this.sqlSession = sqlSession;
        this.mapperInterface = mapperInterface;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //获取Configuration中的MapperRegistry注册中心 通过id到KnownMappers中对于的mapperMethod对象
        MapperMethod mapperMethod = sqlSession.getConfiguration().getMapperRegistry().getKnownMappers().get(method.getDeclaringClass().getName()+"."+method.getName());
        if (null != mapperMethod){
            //调用selectOne方法把mapperMethod传进去和需要的参数
            return sqlSession.selectOne(mapperMethod,String.valueOf(args[0]));
        }
        return method.invoke(proxy,args);
    }
}
~~~

### Executor

执行器接口

~~~java
public interface Executor {
    <T> T query(MapperMethod method, Object parameter) throws Exception;
}
~~~

### SimpleExecutor

selectOne调用的就是Executor中的query方法，实现这个query方法的是SimpleExecutor的query方法

~~~java
public class SimpleExecutor implements Executor {
    private Configuration configuration;
    public SimpleExecutor(Configuration configuration) {
        this.configuration = configuration;
    }
    @Override //实现Executor接口 中的query方法
    public <T> T query(MapperMethod method, Object parameter) throws Exception {
        //创建数据库查询的真正的类
        StatementHandler statementHandler = new StatementHandler(configuration);
        //调用query方法操作数据库
        return statementHandler.query(method,parameter);
    }
}
~~~

### StatementHandler

操作数据库的类

~~~java
public class StatementHandler {
    private Configuration configuration;
    private DefaultResultSetHandler resultSetHandler;
    public StatementHandler() {
    }
    public StatementHandler(Configuration configuration) {
        this.configuration = configuration;
        resultSetHandler = new DefaultResultSetHandler();
    }
    //查询结果
    public <T> T query(MapperMethod method, Object parameter) throws Exception {
        //使用工具类打开数据库连接
        Connection connection = DbUtil.open();
        //传入sql 和 值
        PreparedStatement preparedStatement = connection.prepareStatement(String.format(method.getSql(), Integer.parseInt(String.valueOf(parameter))));
        //执行执行器
        preparedStatement.execute();
        //把结果映射到对象中
        return resultSetHandler.handle(preparedStatement,method);
    }
}
~~~

### ResultSetHandler

反射接口

~~~java
public interface ResultSetHandler {
    public <T> T handle(PreparedStatement pstmt, MapperMethod mapperMethod) throws Exception;
}
~~~

### DefaultResultSetHandler

实现ResultSetHandler的handle方法

~~~java
public class DefaultResultSetHandler implements ResultSetHandler{
    public <T> T handle(PreparedStatement pstmt, MapperMethod mapperMethod) throws Exception {
        Object resultObj = new DefaultObjectFactory().create(mapperMethod.getType());
        ResultSet rs = pstmt.getResultSet();
        if (rs.next()){
            int i = 0;
            for (Field filed : resultObj.getClass().getDeclaredFields()){
                setValue(resultObj,filed,rs,i);
            }
        }
        return (T) resultObj;
    }

    private void setValue(Object resultObj,Field field,ResultSet rs,int i) throws NoSuchMethodException, SQLException, InvocationTargetException, IllegalAccessException {
        Method setMethod = resultObj.getClass().getMethod("set" + upperCapital(field.getName()), field.getType());
        setMethod.invoke(resultObj,getResult(field,rs));
    }

    private String upperCapital(String name){
        String first = name.substring(0,1);
        String tail = name.substring(1);
        return first.toUpperCase()+tail;
    }

    private Object getResult(Field field,ResultSet rs) throws SQLException {
        Class<?> type = field.getType();
        if (Integer.class==type){
            return rs.getInt(field.getName());
        }
        if (String.class==type){
            return rs.getString(field.getName());
        }
        return rs.getString(field.getName());
    }
}
~~~

以上类都编写完毕后就可以运行我们的测试方法就可以看到结果了，大家好好结合前面2篇文章大家好好消化一下，本人也是刚刚入行的小白，把所学的知识分享给大家，在说明过程中可能有一些解释错误的请大家多多指教