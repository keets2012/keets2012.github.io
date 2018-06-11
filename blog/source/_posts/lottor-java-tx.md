---
title: 基于可靠消息方案的分布式事务（二）：Java中的事务
date: 2018-5-31 
categories: 分布式事务
tags:
- 分布式
- 事务
- micro-service
---
前言：在上一篇文章 [基于可靠消息方案的分布式事务：Lottor介绍](http://blueskykong.com/2018/05/04/lottor-intro/) 中介绍了常见的分布式事务的解决方案以及笔者基于可靠消息方案实现的分布式事务组件Lottor的原理，并展示了应用的控制台管理。在正式介绍Lottor的具体实现之前，本文首先将会介绍Java中的事务管理，具体来说是Spring的事务管理。PS：有很多读者提问Lottor是否开源，这里统一回答：是开源的，Lottor目前在笔者所在公司的内部项目应用，并且笔者在将耦合的业务代码重构，将会在下一篇文章同步更新到GitHub，敬请期待。本文较长，适合电脑阅读。如果已对Java中的事务了解，可略过本文，欢迎关注本系列文章。

## Java 事务的类型
事务管理是应用系统开发中必不可少的一部分。Java事务的类型有三种：JDBC事务、JTA(Java Transaction API)事务、容器事务。 常见的容器事务如Spring事务，容器事务主要是J2EE应用服务器提供的，容器事务大多是基于JTA完成，这是一个基于JNDI的，相当复杂的API实现。首先介绍J2EE开发中的两个事务：JDBC事务和JTA事务。
### JDBC 事务
JDBC的一切行为包括事务是基于一个Connection的，在JDBC中是通过Connection对象进行事务管理。在JDBC中，常用的和事务相关的方法是： setAutoCommit、commit、rollback等。

默认情况下，当我们创建一个数据库连接时，会运行在自动提交模式（Auto-commit）下。这意味着，任何时候我们执行一条SQL完成之后，事务都会自动提交。所以我们执行的每一条SQL都是一个事务，并且如果正在运行DML或者DDL语句，这些改变会在每一条SQL语句结束的时存入数据库。有时候我们想让一组SQL语句成为事务的一部分，那样我们就可以在所有语句运行成功的时候提交，并且如果出现任何异常，这些语句作为事务的一部分，我们可以选择将其全部回滚。

```java
public static void main(String[] args) {
        Connection con = null;
        try {
            con = DBConnection.getConnection();
            con.setAutoCommit(false);
            insertAccountData(con, 1, "Pankaj");
            insertAddressData(con, 1, "Albany Dr", "San Jose", "USA");
        }
        catch (SQLException | ClassNotFoundException e) {
            e.printStackTrace();
            try {
                con.rollback();
                System.out.println("JDBC Transaction rolled back successfully");
            }
            catch (SQLException e1) {
                System.out.println("SQLException in rollback" + e.getMessage());
            }
        }
        finally {
            try {
                if (con != null)
                    con.close();
            }
            catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
```
上面的代码实现了一个简单的插入账号和地址信息的功能，通过事务来控制插入操作，要么都提交，要么都回滚。

JDBC为使用Java进行数据库的事务操作提供了最基本的支持。通过JDBC事务，我们可以将多个SQL语句放到同一个事务中，保证一个 JDBC 事务不能跨越多个数据库！所以，如果涉及到多数据库的操作或者分布式场景，JDBC事务就无能为力了。

### JTA 事务
通常，JDBC事务就可以解决数据的一致性等问题，鉴于他用法相对简单，所以很多人关于Java中的事务只知道有JDBC事务，或者有人知道框架中的事务（比如Hibernate、Spring）等。但是，由于JDBC无法实现分布式事务，而如今的分布式场景越来越多，所以，JTA事务就应运而生。

JTA(Java Transaction API)，Java事务API允许应用程序执行分布式事务，也就是说事务可以访问或更新两个或更多网络上的计算机资源。JTA指定事务管理器和分布式事务系统中涉及的各方之间的标准Java接口：应用程序，应用程序服务器和控制对受事务影响的共享资源的访问的资源管理器。一个事务定义了完全成功或根本不产生结果的逻辑工作单元。

> Java 事务编程接口（JTA：Java Transaction API）和 Java 事务服务 (JTS；Java Transaction Service) 为 J2EE 平台提供了分布式事务服务。分布式事务（Distributed Transaction）包括事务管理器（Transaction Manager）和一个或多个支持 XA 协议的资源管理器 ( Resource Manager )。我们可以将资源管理器看做任意类型的持久化数据存储；事务管理器承担着所有事务参与单元的协调与控制。JTA 事务有效的屏蔽了底层事务资源，使应用可以以透明的方式参入到事务处理中；但是与本地事务相比，XA 协议的系统开销大，在系统开发过程中应慎重考虑是否确实需要分布式事务。若确实需要分布式事务以协调多个事务资源，则应实现和配置所支持 XA 协议的事务资源，如 JMS、JDBC 数据库连接池等。

JTA里面提供了 java.transaction.UserTransaction ，里面定义了下面几个方法：

- begin()- 开始一个分布式事务，（在后台 TransactionManager 会创建一个 Transaction 事务对象并把此对象通过 ThreadLocale 关联到当前线程上 )
- commit()- 提交事务（在后台 TransactionManager 会从当前线程下取出事务对象并把此对象所代表的事务提交）
- rollback()- 回滚事务（在后台 TransactionManager 会从当前线程下取出事务对象并把此对象所代表的事务回滚）
- getStatus()- 返回关联到当前线程的分布式事务的状态 (Status 对象里边定义了所有的事务状态)
- setRollbackOnly()- 标识关联到当前线程的分布式事务将被回滚

> 值得注意的是，不是使用了UserTransaction就能把普通的JDBC操作直接转成JTA操作，JTA对DataSource、Connection和Resource 都是有要求的，只有符合XA规范，并且实现了XA规范的相关接口的类才能参与到JTA事务中来。

关于XA规范，读者可以根据前一篇文章进行补充，目前主流的数据库都支持XA规范。使用的流程如下:

> 要想使用用 JTA 事务，那么就需要有一个实现 javax.sql.XADataSource 、 javax.sql.XAConnection 和 javax.sql.XAResource 接口的 JDBC 驱动程序。一个实现了这些接口的驱动程序将可以参与 JTA 事务。一个 XADataSource 对象就是一个 XAConnection 对象的工厂。XAConnection 是参与 JTA 事务的 JDBC 连接。

> 要使用JTA事务，必须使用XADataSource来产生数据库连接，产生的连接为一个XA连接。

> XA连接（javax.sql.XAConnection）和非XA（java.sql.Connection）连接的区别在于：XA可以参与JTA的事务，而且不支持自动提交。

除了数据库，还有JBoss、JMS的消息中间件ActiveMQ等很多组件都是遵守XA协议，2PC和3PC也是符合XA规范的。具体JTA更多的内容，本文不再展开，后面有机会专门深入介绍JTA事务。

## Spring 事务管理
Spring支持编程式事务管理和声明式事务管理两种方式。Spring所有的事务管理策略类都继承自`PlatformTransactionManager`接口，类图如下：

![PlatformTransactionManager类图](http://ovci1ihdy.bkt.clouddn.com/PlatformTransactionManager.jpg)

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition var1) throws TransactionException;

    void commit(TransactionStatus var1) throws TransactionException;

    void rollback(TransactionStatus var1) throws TransactionException;
}
```
其中，`#getTransaction(TransactionDefinition)`方法根据指定的传播行为返回当前活动的事务或创建一个新的事务，这个方法里面的参数是`TransactionDefinition`类，这个类定义了一些基本的事务属性。 其他两个方法，分别是提交和回滚，传入`TransactionStatus`事务状态值。
### 事务属性

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";
	// 定义事务的传播规则
    Propagation propagation() default Propagation.REQUIRED;
	// 指定事务的隔离级别
    Isolation isolation() default Isolation.DEFAULT;
	// 对于长时间运行的事务定义超时时间
    int timeout() default -1;
	// 指定事务为只读
    boolean readOnly() default false;
	// rollback-for指定事务对哪些检查型异常应当回滚而不提交
    Class<? extends Throwable>[] rollbackFor() default {};
	// 导致事务回滚的异常类名字数组
    String[] rollbackForClassName() default {};
	// no-rollback-for指定事务对哪些异常应当继续执行而不回滚
    Class<? extends Throwable>[] noRollbackFor() default {};
	// 不会导致事务回滚的异常类名字数组
    String[] noRollbackForClassName() default {};
}
```

其实通过 `@Transactional`可以了解事务属性的定义。事务属性描述了事务策略如何应用到方法上，事务属性包含5个方面：

- 传播行为
- 隔离级别
- 回滚规则
- 事务超时
- 是否只读

### Spring 事务传播属性
传播行为定义了客户端与被调用方法之间的事务边界，即传播规则回答了这样的一个问题，新的事务应该被启动还是挂起，或者方法是否要在事务环境中运行。Spring定义了7个以`PROPAGATION_`开头的常量表示它的传播属性。在`TransactionDefinition`中定义了如下的常量：

| 名称        | 值    |  解释  |
| --------   | -----:   | :----: |
| PROPAGATION_REQUIRED        |   0    |   支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择，也是Spring默认的事务的传播。    |
| PROPAGATION_SUPPORTS        | 1      |   支持当前事务，如果当前没有事务，就以非事务方式执行。    |
| PROPAGATION_MANDATORY        | 2      |   支持当前事务，如果当前没有事务，就抛出异常。    |
| PROPAGATION_REQUIRES_NEW        | 3      |   新建事务，如果当前存在事务，把当前事务挂起。    |
| PROPAGATION_NOT_SUPPORTED        | 4      |   以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。    |
| PROPAGATION_NEVER        | 5      |   以非事务方式执行，如果当前存在事务，则抛出异常。    |
| PROPAGATION_NESTED        | 6      |   如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。|

### Spring 隔离级别
隔离级别定义了一个事务可能受其他并发事务影响的程度。多事务并发可能会导致脏读、幻读、不可重复读等各种读现象。在`TransactionDefinition`中定义了如下的常量：

| 水果        | 价格    |  数量  |
| --------   | -----:   | :----: |
| ISOLATION_DEFAULT	| -1| 	这是一个PlatfromTransactionManager默认的隔离级别，使用数据库默认的事务隔离级别。另外四个与JDBC的隔离级别相对应| 
| ISOLATION_READ_UNCOMMITTED| 	1	| 这是事务最低的隔离级别，它充许另外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻读。| 
| ISOLATION_READ_COMMITTED| 	2	| 保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。| 
| ISOLATION_REPEATABLE_READ	| 4	| 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻读。| 
| ISOLATION_SERIALIZABLE	| 8	| 这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻读。| 


### 是否只读
如果事务只对后端的数据库进行读操作，数据库可以利用事务ID只读特性来进行一些特定的优化。通过将事务设置为只读，你就可以给数据库一个机会，让他应用它认为合适的优化措施。因为是否只读是在事务启动的时候由数据库实施的，所以只有对那些具备可能启动一个新事务的传播行为（`PROPAGATION_REQUIRED`,`PROPAGATION_REQUIRED_NEW`,`PROPAGATION_NESTED`）的方法来说，才有意义。

### 事务超时
为了使应用程序很好地运行，事务不能运行太长时间。因为超时时钟会在事务开始时启动，所以只有对那些具备可能启动一个新事务的传播行为（同上几种）的方法来说，才有意义。默认设置为底层事务系统的超时值，如果底层数据库事务系统没有设置超时值，那么就是none，没有超时限制。

### 事务回滚
事务回滚规则定义了哪些异常会导致事务回滚而哪些不会。默认情况下，事务只有在遇到运行时期异常才回滚，而在遇到检查型异常时不会回滚。就是抛出的异常为`RuntimeException`的子类(Errors也会导致事务回滚)，而抛出checked异常则不会导致事务回滚。可以明确的配置在抛出那些异常时回滚事务，包括checked异常。也可以明确定义那些异常抛出时不回滚事务。还可以通过编程的`setRollbackOnly()`方法来指示一个事务必须回滚，在调用完`setRollbackOnly()`后所能执行的唯一操作就是回滚。

## 配置 Spring 事务管理器

Spring中使用xml进行如下的配置即可：

```xml
 <!-- 配置事务管理器 -->  
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">  
        <property name="dataSource" ref="dataSource" />  
    </bean>  
```

如果是Spring Boot，我们只需要设置好`DataSource`即可，`DataSourceTransactionManagerAutoConfiguration`会自动配置好`DataSourceTransactionManager`。

```java
		@Bean
		@ConditionalOnMissingBean(PlatformTransactionManager.class)
		public DataSourceTransactionManager transactionManager(
				DataSourceProperties properties) {
			DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(
					this.dataSource);
			if (this.transactionManagerCustomizers != null) {
				this.transactionManagerCustomizers.customize(transactionManager);
			}
			return transactionManager;
		}
```
配置了事务管理器后，事务当然还是得我们自己去操作，Spring提供了两种事务管理的方式：编程式事务管理和声明式事务管理，下面分别看一下如何应用。
### 编程式事务管理
编程式事务管理我们可以通过`PlatformTransactionManager`实现来进行事务管理，同样的Spring也为我们提供了模板类`TransactionTemplate`进行事务管理，下面主要介绍模板类。

#### 模板类
我们需要在配置文件中配置：

```xml
    <!--配置事务管理的模板-->
    <bean id="transactionTemplate" class="org.springframework.transaction.support.TransactionTemplate">
        <property name="transactionManager" ref="transactionManager"></property>
        <!--定义事务隔离级别,-1表示使用数据库默认级别-->
        <property name="isolationLevelName" value="ISOLATION_DEFAULT"></property>
        <property name="propagationBehaviorName" value="PROPAGATION_REQUIRED"></property>
    </bean>
```
`TransactionTemplate`帮我们封装了许多代码，节省了我们的工作。专门建了一张account表，用于模拟存钱的一个场景。

```java
    //BaseSeviceImpl.java
    @Override
    public void insert(String sql, boolean flag) throws Exception {
        dao.insertSql(sql);
        // 如果flag 为 true ，抛出异常
        if (flag) {
            throw new Exception("has exception!!!");
        }
    }
    //获取总金额
    @Override
    public Integer sum() {
        return dao.sum();
    }
```


```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:spring-test.xml"})
public class TransactionTest{
    @Resource
    private TransactionTemplate transactionTemplate;
    @Autowired
    private BaseSevice baseSevice;

    @Test
    public void transTest() {
        System.out.println("before transaction");
        Integer sum1 = baseSevice.sum();
        System.out.println("before transaction sum: "+sum1);
        System.out.println("transaction....");
        transactionTemplate.execute(new TransactionCallbackWithoutResult() {
            @Override
            protected void doInTransactionWithoutResult(TransactionStatus status) {
                try {
                    baseSevice.insert("INSERT INTO account VALUES (100);",false);
                    baseSevice.insert("INSERT INTO account VALUES (100);",false);
                } catch (Exception e) {
                    status.setRollbackOnly();
                    e.printStackTrace();
                }
           }
        });
        System.out.println("after transaction");
        Integer sum2 = baseSevice.sum();
        System.out.println("after transaction sum: "+sum2);
    }
}
```

对于抛出Exception类型的异常且需要回滚时，需要捕获异常并通过调用status对象的`setRollbackOnly()`方法告知事务管理器当前事务需要回滚。

### 声明式事务管理
声明式事务管理有两种常用的方式，一种是基于tx和aop命名空间的xml配置文件，一种是基于`@Transactional`注解，随着Spring和Java的版本越来越高，越趋向于使用注解的方式。

#### 基于tx和aop命名空间的xml配置文件 

```xml
<tx:advice id="advice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="insert" propagation="REQUIRED" read-only="false"  rollback-for="Exception"/>
        </tx:attributes>
    </tx:advice>

    <aop:config>
        <aop:pointcut id="pointCut" expression="execution (* com.blueskykong.service.*.*(..))"/>
        <aop:advisor advice-ref="advice" pointcut-ref="pointCut"/>
    </aop:config>
```

```java
    @Test
    public void transTest() {
        System.out.println("before transaction");
        Integer sum1 = baseSevice.sum();
        System.out.println("before transaction sum: "+sum1);
        System.out.println("transaction....");
        try{
            baseSevice.insert("INSERT INTO account VALUES (100);",true);
        } catch (Exception e){
            e.printStackTrace();
        }
        System.out.println("after transaction");
        Integer sum2 = baseSevice.sum();
        System.out.println("after transaction sum: "+sum2);
    }
```
#### 基于@Transactional注解
最常用的方式，也是简单的方式。只需要在配置文件中开启对注解事务管理的支持。

```xml
<tx:annotation-driven transaction-manager="transactionManager"/>
```
而Spring boot启动类必须要开启事务，而开启事务用的注解就是`@EnableTransactionManagement`。

```java
    @Transactional(rollbackFor=Exception.class)
    public void insert(String sql, boolean flag) throws Exception {
        dao.insertSql(sql);
        // 如果flag 为 true ，抛出异常
        if (flag){
            throw new Exception("has exception!!!");
        }
    }
```
指定出现Exception异常的时候回滚，遇到检查性的异常需要回滚，默认情况下非检查性异常，包括error也会自动回滚。 

## 总结
本文主要介绍了Java事务的类型：JDBC事务、JTA(Java Transaction API)事务、容器事务。简要介绍了JDBC事务和JTA事务，详细介绍了Spring的事务管理，Spring对事务管理是通过事务管理器来实现的。Spring提供了许多内置事务管理器实现，如数据源事务管理器`DataSourceTransactionManager `、集成JPA的事务管理`JpaTransactionManager`等。分别介绍了Spring事务属性包含的5个方面，以及Spring提供的两种事务管理的方式：编程式事务管理和声明式事务管理。其中声明式的事务管理，最为便捷、使用的更多。通过本文的介绍，希望读者在接触分布式事务时，首先对Java中的事务能够熟悉。JTA事务时，其实也引出了分布式事务的相关概念，对应2PC和3PC的XA规范。

#### 推荐阅读
[基于可靠消息方案的分布式事务](http://blueskykong.com/categories/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1/)

#### 参考
1. [Spring的事务管理机制](http://www.hollischuang.com/archives/1489)
2. [JTA 深度历险 - 原理与实现](https://www.ibm.com/developerworks/cn/java/j-lo-jta/)
3. [Java中的事务——JDBC事务和JTA事务](http://www.hollischuang.com/archives/1658)
4. [Spring事务管理详解](https://blog.csdn.net/donggua3694857/article/details/69858827)
