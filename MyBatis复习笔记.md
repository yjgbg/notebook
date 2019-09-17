# MyBatis复习笔记

## 入门

### 引入依赖

```groovy
// https://mvnrepository.com/artifact/org.mybatis/mybatis MYBATIS
compile group: 'org.mybatis', name: 'mybatis', version: '3.4.6'
// https://mvnrepository.com/artifact/mysql/mysql-connector-java MYSQL DRIVER
compile group: 'mysql', name: 'mysql-connector-java', version: '8.0.13'
// https://mvnrepository.com/artifact/com.alibaba/druid DRUID POOL
compile group: 'com.alibaba', name: 'druid', version: '1.1.12'
```

### 使用XML配置文件

mybatis-config.xml文件:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
  <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="UserMapper.xml"/><!--引入了mapper文件-->
  </mappers>
</configuration>
```

UserMapper.xml文件:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="me.yjgbg.dao.UserMapper"><!--namespace定位到类-->
  <select id="selectOne" resultType="UserEntity"><!--id+namespace定位到具体方法-->
    select * from USER where id = #{id}
  </select>
</mapper>
```

mapper文件对应的接口：

```java
package me.yjgbg.dao;

public class UserMapper{
    public UserEntity selectOne(int id);
}
```

主程序Application类：

```java
public class Application{
public static void main(String... args){
        String  resource = "mybatis-config.xml";
        InputStream stream = Resource.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resource);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        userMapper.selectOne(10);
        sqlSession.close();
    }
}
```

### 不使用XML配置文件

接口：

```java
package me.yjgbg.dao

public class UserMapper{
    @Select("SELECT * FROM USER WHERE ID = #{id}")
    public UserEntity selectOne(int id);
    @Update("UPDATE USER SET name = #{user.name},age = #{user.age}WHERE id = #{id}")
    public void updateOne(@Param("id")int id, @Param("user")UserEntity user)
}
```

Application主程序:

```java
public class Application{
    public static void main(String... args){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/Demo2?characterEncoding=utf-8&amp;useSSL=false");
        dataSource.setUsername("yjgbg");
        dataSource.setPassword("1314");
        //不用设置驱动，因为url中包含了数据库是哪种数据库，所以可以直接加载驱动程序
        TransactionFactory factory = new JdbcTransactionFactory();
        Environment env = new Environment("envId",factory,dataSource);
        Configuration config = new Configuration();
        config.setEnvironment(env);
        config.addMapper(userMapper.class);
        SqlSessionFactory fac = new SqlSessionFactoryBuilder().build(config);
        SqlSession sqlSession = fac.openSession(autocommit:true);
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        mapper.selectOne(10);
        UserEntity user = new UserEntity();
        user.setName("Alice");
        user.setAge(23);
        mapper.updateOne(10,user);
       sqlSession.close();
    }
}
```

## 作用域（Scope）和生命周期

作用域与声明周期很重要，错误的使用会导致并发问题。

对象依赖注入框架可以创建线程安全，基于事务的SqlSession和Mapper映射器，并将他们注入到bean中，因此可以直接忽略它们的声明周期。with spring，详见mybatis-spring或mybatis-guide两个子项目。

### SqlSessionFactoryBuilder

这个类用来构建SqlSessionFactory,构建完就没用了。也就是说作用域应尽可能小（方法作用域，匿名对象）

### SqlSessionFactory

一旦被创建，整个运行期间都会存在，不应被清除。最佳策略是，单例。且尽量全局可访问。

### SqlSession

每个线程都有一个SqlSession实例。SqlSession非线程安全，于是不可用来多线程共享。最佳作用域是方法作用域或者http请求的整个处理过程。注：不可以是类的成员变量。

```java
SqlSession session = SqlSessionFactory.openSession(autocommit:true);
try{
    //TODO
}finally{
    session.close();
}
```

或者使用try-with-resource：

```java
try(SqlSession session = sqlSessionFactory.openSession(true)){
    //TODO
}
```

### Mapper(映射器实例)

Mapper是从SqlSession中获得的，于是Mapper的作用域肯定不大于SqlSession的作用域。其最佳作用域为方法作用域。映射器实例应该在调用它们的方法中被申请，并且用完之后立即废弃。并不需要显式地关闭。

如果在整个请求域（一次Request）中保持mapper，会难以控制。所以保持简单：将mapper放在方法作用域。

```java
SqlSession session = SqlSessionFactory.openSession(true);
try{
    UserMapper mapper = session.getMapper(UserMapper.class);
    //TODO
}finally{
    session.close();
}
```

## XML配置文件（mybatis-config.xml)

### properties

mybatisconfig.xml

```xml
......
<properties resource="config.properties">
<property name="org.apache.ibatis.parsing.PropertyParser.enable-default-value" value="true"/>
</properties>
......
<dataSource class="xxx.xxx.xxx">
    <property name="driver" value="${jdbc.driver}" />
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="{jdbc.username:yjgbg}"/><!--为属性提供了一个默认值，如果找不到jdbc.username属性，则填充为yjgbg。但是这个Feature是默认关闭的，需要设置本段代码第三行的内容用以开启这个Feature-->
    <property name="password" value="{jdbc.password}"/>
</dataSource>
```

config.properties

```properties
jdbc.url=jdbc:mysql://localhost:3306/Demo2?characterEncoding=utf-8&amp;amp;useSSL=false
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.username=yjgbg
jdbc.password=1314
```

备注：不支持yml文件进行配置

或者：

```java
Properties props = new Properties();
props.setProperty("jdbc.username","yjgbg");
//TODO 设置props其他jdbc属性
SqlSessionFactory fac = new SqlSessionFactoryBuilder().build(reader,props);
```

优先级:

代码中>properties的resource单独文件中>properties元素体内

先读取properties元素体内的，再读取resource单独文件中的（有重复则覆盖），最后读取代码中设置的（有重复则覆盖），最终共同形成一个properties对象，用来生成factory。

### Settings

常用属性:

| 属性               | 描述                             | 有效值     | 默认值 |
| ------------------ | -------------------------------- | ---------- | ------ |
| cacheEnabled       | 缓存                             | true/false | true   |
| LazyloadingEnabled | 开启时所有的对象都会被延迟加载。 | true/false | false  |
| useGeneratedKeys   | 允许jdbc支持自动生成主键         | true/false | false  |

### TypeHandler

在预处理语句或者结果集处理中，都需要JDBC类型与JAVA类型之间的转化，完成转化的就是TypeHandler（类型处理器）

MyBatis自带了很多TypeHandler，包括8中JAVA基本类型对应的类型处理器，以及部分日期的类型处理器。

自己写TypeHandler需要继承org.apache.ibatis.type.BaseTypeHandler类或实现org.apache.ibatis.type.TypeHandler接口,然后选择性地映射到一个JDBC类型。

```java
@MappedJdbcTypes(JdbcType.VARCHAR)
public ExampleTypeHandler extends BaseTypeHandler<Obj>{
    @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Obj parameter, JdbcType jdbcType) throws SQLException {
    ps.setString(i, Obj.toString());
  }

  @Override
  public Obj getNullableResult(ResultSet rs, Obj columnName) throws SQLException {
    return new Obj(rs.getString(columnName));
  }

  @Override
  public Obj getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    return new Obj(rs.getString(columnName));
  }

  @Override
  public Obj getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    return new Obj(rs.getString(columnName));
  }
}
```

```
<typeHandlers>
    <typeHandler handler="xxx.xxx.ExampleTypeHandler" />
</typeHandlers>
```
MyBatis不会窥探数据库元信息来确定映射到哪种JDBC类型，因此需要用在TypeHandler类上用@MappedJdbcTypes(Jdbc.VARCHAR)注解来标注需要目标JDBC类型；或者在xml文件中的typeHandler标签上注明：jdbcType=”VARCHAR”

而通过mapper文件中namespace和id定位到方法，可以通过方法的签名信息确定输入或输出的JAVA类型。

配置类型处理器也可以直接写包名，然后自动查找：

```
<typeHandlers>
    <package>me.yjgbg.typeHandlers</package>
</typeHandlers>
```

在使用自动索引(autodiscovery)时，只能在类中用注解注明JDBCType。

#### ENUM处理器

枚举处理器是一个特殊的存在，其他类型处理器只处理某种类型，而枚举处理器处理所有的枚举类型。

### 对象工厂

MyBatis在创建结果对象的实例时会用ObjectFactory实例来完成。默认的对象工厂做的操作只是创建对象，于是当我们需要在创建对象是执行一定的操作时就需要创建自己的对象工厂实例。

例如:

```java
public class ExampleObjectFactory extends DefaultObjectFactory{
    public Object create(Class type) {
    return super.create(type);
  }
  public Object create(Class type, List<Class> constructorArgTypes, List<Object> constructorArgs) {
    return super.create(type, constructorArgTypes, constructorArgs);
  }
  public void setProperties(Properties properties) {
    super.setProperties(properties);
  }
  public <T> boolean isCollection(Class<T> type) {
    return Collection.class.isAssignableFrom(type);
  }
}
```

```xml
<objectFactory type="xxx.xxx.ExampleObjectFactory">
        <property name="someProperty" value="val" />
</objectFactory>
```



### 插件(plugins)

MyBatis允许使用插件来拦截的方法有:

```java
Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)
```

使用方式:

```java
@Intercepts({@Signature(
    type=Executor.class,method="update",
    args={MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interecptor{
    public Object intercept(Invocation invocation) throws Throwable{
        return invocation.proceed();
    }
    public Object plugin(Object target){
        return Plugin.wrap(target,this);
    }
    public void setProperties(Properties properties){

    }
}

```



```xml
<plugins>
  <plugin interceptor="org.mybatis.example.ExamplePlugin">
    <property name="someProperty" value="100"/><!--此处传递的properties将会调用上面的setProperties方法-->
  </plugin>
</plugins>
```





==注：这可能会严重影响 MyBatis 的行为，慎重。==

### 配置环境

尽管每个config类或者xml文件中可以配置多个环境，但是在生成SqlSessionFactory时只能选择其一。如果是3个数据库则需要对应三个SqlSessionFactory.

每个数据库对应一个SqlSessionFactory实例。

如果在创建SqlSessionFactory时没有指定环境，那么就会加载默认环境。

```xml
<environments default="development">
  <environment id="development">
    <transactionManager type="JDBC">
      <property name="..." value="..."/>
    </transactionManager>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
</environments>
```

文件详解：

- 默认的环境ID： default=”development”
- 每个环境有其对应的ID
- 事务管理器的配置
- 数据源的设置

#### 事务管理器

在MyBatis中有两种事务管理器

- JDBC - 直接使用JDBC的提交和回滚，依赖于从数据源获得的连接来管理事务作用域。
- MANAGED 什么都不做，从不提交或回滚，而是让容器来管理整个事务的声明周期。

如果使用Spring，那么在MyBatis中不用设置事务管理器，因为Spring会覆盖这个设置。

## 返回自增主键的值

```xml
<insert id="insertAuthor" useGeneratedKeys="true" keyColumn="idInDB"
    keyProperty="idInApp">
  insert into Author (username,password,email,bio)
  values (#{username},#{password},#{email},#{bio})
</insert>
```

于是数据库中自增的idInDB列数据会被返回到调用的参数对象的idInApp字段

### SQL标签

```xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
```

这个元素被用来定义可重用的SQL代码段。

可以被包含到其他SQL语句中：

```xml
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

于是整句被拼接成了：

```sql
select t1.id, t1.username, t1.password,
t2.id, t2.username, t2.password from some_table t1 cross join some_table t2
```

