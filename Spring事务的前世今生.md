# Spring事务 2019-01-24

## 前置知识:脏读&不可重复读&幻读

### 脏读

 读取到尚未其他事务尚未提交的数据

### 不可重复读

 同一个事务内多次读取同一个数据，在这多次读取之间，有其他事务修改了这个数据，导致了多次读取的结果不一致。

### 幻读

 如果事务A要修改表中所有数据，而在事务尚未提交时，事务B添加了一条记录，那么导致B添加的这一行未发生修改，那么对于A而言，这似乎为幻觉，称之为幻读。

## 事务属性

### 隔离级别

#### ISOLATION_DEFAULT

 数据库默认级别，通常等于COMMITTED；

#### ISOLATION_READ_UNCOMMITTED

 可以读取另一个事务修改但尚未提交的数据。

 不可防止脏读和不可重复读。

 约等于不管

#### ISOLATION_READ_COMMITTED

 表示一个事务只能读取令一个事务已经提交的数据。

 防止脏读。

 多数情况下的推荐级别

#### ISOLATION_REPEATABLE_READ

 单个事件中可以多次重复读取一个值而不发生变化。

 防止脏读，不可重复读

#### ISOLATION_SERIALIZABLE

 所有事务依次逐个执行，这样事务之间就完全不可能产生干扰。

 防止脏读，不可重复读，幻读。

### 事务传播类型

#### PROPAGATION_REQUIRED

 如果当前存在事务则加入该事务；如果没有，则创建一个新的事务

 （无论如何使接下来的代码进入一个事务）

#### PROPAGATION_REQUIRED_NEW

 创建一个新的事务，如果当前存在事务，则挂起当前事务。

 （无论如何创建一个事务）

#### PROPAGATION_SUPPORTS

 如果当前存在事务则加入；如果没有，则继续以非事务的方式继续运行。

 （消极，随遇而安，有就加入，没有就算了）

#### PROPAGATION_NOT_SUPPORTED

 以非事务方式运行，如果当前存在事务，则挂起当前事务。

 （无论如何接下来的代码不在事务中运行）

#### PROPAGATION_NEVER

 以非事务方式运行，如果当前存在事务，则抛出异常。

 （当前不应该存在事务）

#### PROPAGATION_MANDATORY

 如果当前存在事务，则加入该事务；如果没有事务，则抛出异常

 （当前应该存在一个事务）

#### PROPAGATION_NESTED（Spring特有的）

 如果当前存在一个事务，则创建一个事务作为当前事务的嵌套事务来运行；如果不存在事务，则创建一个事务。内部事务并不是一个独立的事务，而是依赖于外部事务的存在而存在，只有外部事务的提交才会引起内部事务的提交，这是JDBC中保存点(SavePoint)的一个应用。外部事务的回滚会导致内部事务的回滚。

### 事务超时

 事务超时是指一个事务所能允许的最长时间，超时自动回滚事务。int,单位是秒。

### 事务的只读属性

 事务的只读属性是指对事务性资源进行只读操作或者读写操作。所谓事务性资源是指那些被事务管理的资源，比如数据源（`DataSourceTransactionManager`)，JMS资源(`JmsTransactionManager`)，自定义的事务性资源等。

 在`TransactionDefinition`中定义以`Boolean`类型来定义

### 事务的回滚规则

 默认：发生未检查异常(`RuntimeException`)时回滚事务，其他情况正常提交。

## Spring事务管理的API分析

### `TransactionDefinition`

 用于定义事务，包含了事务的静态属性，包括事件传播行为，超时事件等。Spring中有个默认实现类（`DefaultTransactionDefinition`)

### `PlatformTransactionManager`

 用于执行具体的事务操作(创建事务，提交事务，回滚事务)

 主要实现类有`DataSourceTransactionManager`, `HibernateTransactionManager`, `JpaTransactionManager`,除此之外还有`JtaTransactionManager`, `JdoTransactionManager`, `JmsTransactionManager`等。

 `DataSourceTransactionManager`，在定义时需要提供底层的数据源作为其属性，也就是 `DataSource`。与 `HibernateTransactionManager` 对应的是 `SessionFactory`，与 `JpaTransactionManager` 对应的是 `EntityManagerFactory` 等等。

### `TransactionStatus`

 由`PlatformTransactionManager.getTransaction(TransactionDefinition)`返回。表示一个新的或者已经存在的事务（如果在当前调用堆栈有一个符合条件的事务）（事务池？？？）。

## 编程式事务管理

### 基于底层API

```java
PlatformTransactionManager manager = new DataSourceTransactionManager();
manager.setDatasource(datasource);
TransactionDefinition def = new DefaultTransactionDefinition();
TransactionStatus status = manager.getTransaction(def);
try{
    //TODO 写业务逻辑
    manager.commit(status);
}catch (Exception e){
    manager.rollback(status);
}
```

### 基于`TransactionTemplate`

上面的示例中业务代码和事务管理代码混在一起，不方便管理

```java
public void serviceMethod(Object args){
        PlatformTransactionManager manager = new DataSourceTransactionManager();
        manager.setDatasource(datasource);//manager依赖数据源
        TransactionTemplate transactionTemplate = new TransactionTemplate();
        transactionTemplate.setTransactionManager(manager);//template依赖manager
        transactionTemplate.execute(new  TransactionCallback(){
                @Override
                public Object doInTransaction(TransactionStatus status){
                        //TODO 写业务逻辑
                        setRollbackOnly()//这个方法用来回滚事务
                        return null;
                }
        });
}
```

## 声明式事务管理

声明式事务管理基于AOP。本质是对方法前后进行拦截。，然后在目标方法开始之前创建或者加入一个事务，在执行完目标方法后提交或者回滚事务。

使用声明式事务管理有效地隔离业务代码和事务控制代码，方便后期的代码维护。

相比于编程式事务管理，声明式不足之处是控制粒度只能到达方法级别，无法像编程式一样作用到代码块级别。

### 基于`TransactionInterecptor`

```xml
<beans>
    ......
    <bean id="transactionInterceptor" class="org.springframework.transaction.interecptor.TransactionInterceptor"><!---定义了插入规则，事务属性的bean--->
        <property name="transactionManager" ref="transactionManager"/>
        <property name="transactionAttributes">
            <props>
                <prop key="transfer">PROPAGATION_REQUIRED</prop><!--key筛选方法，只对符合key的方法进行事务管理-->
            </props>
        </property>
    </bean>

    <bean id="bankServiceTarget" class="xxx.xxx.bankServiceImpl"><!---目标bean--->
        <property name="bankDao" ref="bankDao"/>
    </bean>

    <bean id="bankService" class="org.springframework.aop.framework.ProxyFactoryBean"><!---代理bean--->
        <property name="target" ref="bankServiceTarget"/>
        <property name="interceptorNames">
            <list>
                <idref bean="transactionInterceptor"/>
            </list>
        </property>
    </bean>
    ......
</beans>
```

配置了一个`TransactionInterceptor`用来定义相关的事务规则，两个重要属性：1，`transactionManager`，用来制定一个事务管理器；2，`transactionAttributes`，用来定义事务规则，在该属性的每一个键值对中，键指定的是方法名，值指的是相应方法所应用的事务属性。例：

```xml
<property name="*Service">
                PROPAGATION_REQUIRED, ISOLATION_READ_COMMITTED, TIMEOUT_20, +AbcException, +DefException, -HijException    
</property>
```

针对所有名称以Service结尾的方法。

```xml
<property name="test">PROPAGATION_REQUIRED，readOnly</property>
```

针对所有名字为test的方法。

### 基于`TransactionProxyFactoryBean`

基于`TransactionInteceptor`的方式配置文件太多。必须针对每一个目标对象配置一个`ProxyFactoryBean`,于是加上目标对象本身，每个业务类都会对应三个bean。

于是用`TransactionProxyFactoryBean`将`TransactionInterceptor`和`ProxyFactoryBean`配置合而为一。

```xml
<beans>
    ......
    <bean id = "bankServiceTarget" class="xxx.xxx.BankServiceImpl">
        <property name="bankDao" ref="bankDao"></property>
    </bean>

    <bean id = "bankService" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean"><!---代理bean，定义了插入规则，事务属性--->
        <property name = "target" ref="bankServiceTarget"/>
        <property name="transactionManager" ref="transactionManager"/>
        <property name="transactionAttributes">
            <props>
                <prop key="transfer">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
    </bean>
    ......
</beans>
```

如此以来配置文件每个业务类配置两个bean，简化了1/3的配置文件，但是仍然要显式地为每一个业务类配置。

### 基于`<tx>`命名空间

Spring 2.x引入了`<tx>`命名空间，结合`<aop>`命名空间,使得配置变得更加简单和灵活。

```xml
<beans>
......
    <bean id="bankService" class="xxx.xxx.BankService">
        <property name="bankDao" ref="bankDao"></property>
    </bean>

    <tx:advice id="bankAdvice" transaction-manager="transactionManager"><!--默认为transaction，所以一般可以省略-->
        <tx:attributes>
            <tx:method name="transfer" propagation="REQUIRED"/>
        </tx:attributes>
    </tx:advice>

    <aop:config>
            <aop:pointcut id="bankPointcut" expression="execution (* *.transfer(..) )"/>
            <aop:advisor advice-ref="bankAdvice" pointcut-ref="bankPointcut"/>
    </aop:config>
</beans>
```

通过pointcut切点表达式，一次性可以给多个方法做事务管理，不再需要给每个需要的方法单独创建代理对象。

### 基于@Transactional（Recommended）

Spring 2.x还引入了基于Annotation的方式

```java
@Transactional(propagation = Propagation.REQUIRED)//该方法为一个事件，此处可以配置事件的各种属性。
public boolean transfer(Object args){
    //TODO
}
```

并且需要在beans.xml中配置：

```xml
<tx:annotation-driven transaction-manager="transactionManager"/><!--默认值为transactionManager，所以一般可以省略该属性-->
```

或者在Config.java中配置：

```java
@Configuration//这是一个配置类
@EnableAspectJAutoProxy//开启注解配置AOP
@EnableTransactionManagement//开启事务支持
public class Config{

}
```

@Transactional可以用于接口，接口方法，类，类方法，但是不建议用在接口或接口方法上，因为这样的话只有在使用基于接口的代理时才会生效。

只建议使用tx和@Transactional两种方式；不建议使用TransactionInterceptor和TransactionProxyFactoryBean方式。

这四种方法的后台实现方式一样。

## 总结

TransactionDefinition，PlatformTransactionManager，TransactionStatus编程式事务管理是Spring提供的最原始的方式。

基于TransactionTemplate的编程式事务管理是对上一种方式的封装，使得编码更清晰，简单。

基于TransactionInterceptor的声明式事务是Spring声明式事务的基础，通常也不建议使用

基于TransacttionProxyFactoryBean的声明式事务是上中方式的改进版本，简化了配置文件，早期推荐使用，但是2.x开始不推荐。

基于tx和aop命名空间的声明式事务管理是目前推荐的方式，与AOP结合紧密，可以充分利用切点表达式的强大支持，使事务管理更加灵活。

基于@Transactional的方式将声明式事务管理简化到了极致。配置文件中只需要依据启用相关后处理bean的配置，然后在需要实施事务管理的方法或类上使用@Transactional指定事务规则即可实现事务管理，而且功能上并不比其他方式逊色。