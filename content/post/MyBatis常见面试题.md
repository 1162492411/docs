---
title: "MyBatis常见面试题"
date: 2020-12-27T15:30:50+08:00
draft: false
categories:
  - 面试题
  - 框架
  - MyBatis
tags:
  - 面试题
  - 框架
  - MyBatis
---



# 基础篇

##  **#** 和**$**的区别

{{< spoiler >}} 

- `${}`是 Properties 文件中的变量占位符，它可以用于标签属性值和 sql 内部，属于静态文本替换，比如${driver}会被静态替换为`com.mysql.jdbc.Driver`。
- `#{}`是 sql 的参数占位符，MyBatis 会将 sql 中的`#{}`替换为?号，在 sql 执行前会使用 PreparedStatement 的参数设置方法，按序给 sql 的?号占位符设置参数值，比如 ps.setInt(0, parameterValue)，`#{item.name}` 的取值方式为使用反射从参数对象中获取 item 对象的 name 属性值，相当于 `param.getItem().getName()`。

{{< / spoiler >}}

## Xml映射文件中，除了常见的select|insert|updae|delete标签之外，还有哪些标签

{{< spoiler >}} 

<resultMap>、<parameterMap>、<sql>、<include>、<selectKey>

{{< / spoiler >}}



## MyBatis的动态SQL是什么

{{< spoiler >}} 

Mybatis动态sql可以让我们在Xml映射文件内，以标签的形式编写动态sql，完成逻辑判断和动态拼接sql的功能，Mybatis提供了9种动态sql标签trim|where|set|foreach|if|choose|when|otherwise|bind。

其执行原理为，使用OGNL从sql参数对象中计算表达式的值，根据表达式的值动态拼接sql，以此来完成动态sql的功能

{{< / spoiler >}}

## Mybatis是如何将sql执行结果封装为目标对象并返回的？都有哪些映射形式

{{< spoiler >}} 

第一种是使用<resultMap>标签，逐一定义列名和对象属性名之间的映射关系。第二种是使用sql列的别名功能，这种方式原理是反射。

{{< / spoiler >}}

## 为什么说Mybatis是半自动ORM映射工具？它与全自动的区别在哪里

{{< spoiler >}} 

Hibernate属于全自动ORM映射工具，使用Hibernate查询关联对象或者关联集合对象时，可以根据对象关系模型直接获取，所以它是全自动的。而Mybatis在查询关联对象或关联集合对象时，需要手动编写sql来完成，所以，称之为半自动ORM映射工具

{{< / spoiler >}}

## TypeHandler的作用有哪些

{{< spoiler >}} 

* 完成javaType至jdbcType的转换
* 完成javaType至jdbcType的转换

{{< / spoiler >}}





# 核心原理篇

## MyBatis有哪些主要组件

{{< spoiler >}} 

* Transaction ：事务接口，所有操作最终由该接口
* TransactionFactory ： 负责Transaction的创建、销毁

* SqlSessionFactory：负责SqlSession的创建、销毁

* SqlSession ： 负责提供给用户可以操作的api，如insert(),insertBatch()等，它的生命周期限定在线程之内

* Executor ： 负责执行对数据库的操作

* StatementHandler ：Executor将工作委托给StatementHandler执行（实际干活的老实人）

{{< / spoiler >}}

## Mybatis的执行流程

![执行流程](https://gitee.com/1162492411/pic/raw/master/组件-MyBatis-执行流程.jpg)

## 简述SQL在Myabtis中的执行流程

![SQL执行流程](https://gitee.com/1162492411/pic/raw/master/框架-MyBatis-Sql执行流程.jpg)

## 简述各Executor子类的作用

![Executor类结构图](https://gitee.com/1162492411/pic/raw/master/组件-MyBatis-Executor类结构图.png)

{{< spoiler >}} 

* SimpleExecutor ：简单Executor，每执行一次update或select，就开启一个Statement对象，用完立刻关闭Statement对象
* ReuseExecutor ： 重用Executor，执行update或select，以sql作为key查找Statement对象，存在就使用，不存在就创建，用完后，不关闭Statement对象，而是放置于Map<String, Statement>内，供下一次使用。在执行commit、rollback等动作前，将会执行flushStatements()方法，将Statement对象逐一关闭
* BatchExecutor ：批量Executor，将所有sql都添加到批处理中（addBatch()），缓存了多个Statement对象，sql添加完成后统一执行（executeBatch()）
* CachingExecutor ： 缓存Executor，装饰模式的应用，先从缓存中获取查询结果，存在就返回，不存在，再委托给Executor delegate去数据库取，delegate可以是上面任一的SimpleExecutor、ReuseExecutor、BatchExecutor

{{< / spoiler >}}

## 简述各StatementHandler子类的作用

![StatementHandler类结构图](https://gitee.com/1162492411/pic/raw/master/组件-MyBatis-StatementHandler类结构图.png)

{{< spoiler >}} 

* SimpleStatementHandler：用于处理**Statement**对象的数据库操作
* PreparedStatementHandler：用于处理**PreparedStatement**对象的数据库操作。
* CallableStatementHandler：用于处理存储过程
* RoutingStatementHandler ： 根据statementType来创建其他三个StatementHandler对象

{{< / spoiler >}}

## 简述KeyGenerator各子类的作用

{{< spoiler >}} 

* NoKeyGenerator : 空实现，不需要处理主键
* Jdbc3KeyGenerator : 用于处理数据库支持自增主键的情况，如MySQL的auto_increment
* SelectKeyGenerator : 用于处理数据库不支持自增主键的情况，比如Oracle的sequence序列

{{< / spoiler >}}



## 简述设计模式在MyBatis中的运用

{{< spoiler >}} 

* 工厂模式 ： TransactionFactory负责Transaction的生产、销毁；SqlSessionFactory负责SqlSession的生产、销毁

* 装饰器模式 ： CachingExecutor在SimpleExecutor/ReuseExecutor/BatchExecutor的基础上提供了缓存功能

* 模版模式 ： Executor通过调用StatementHandler的模版方法来完成对数据库的操作
* 适配器模式 ： BaseStatementHandler抽象类分别有三个实现类：SimpleStatementHandler、PreparedStatementHandler、CallableStatementHandler
* 策略模式 ： RoutingStatementHandler根据statementType的不同来创建不同的StatementHandler
* 责任链模式 ： MyBatis中存在一些插件，它们都会以责任链的方式逐一执行

{{< / spoiler >}}

## 简述一级缓存和二级缓存的原理

{{< spoiler >}} 

* 一级缓存 ： 在CachingExecutor中针对query操作的结果，将其放置在HashMap中，有效范围为同一个SqlSession(默认)/Statement
* 二级缓存 ： 在CachingExecutor中实现，有效范围为全局Configuration，在所有SqlSession中均有效

{{< / spoiler >}}

## 一级缓存在哪些情况下失效

{{< spoiler >}} 

* sqlsession变了 缓存失效
* sqlsession不变,查询条件不同，一级缓存失效
* sqlsession不变,中间发生了增删改操作，一级缓存失败
* sqlsession不变,手动清除缓存，一级缓存失败
{{< / spoiler >}}


## 缓存清空策略有哪些

{{< spoiler >}} 

* LRU ：最近最少使用算法，即如果缓存中容量已经满了，会将缓存中最近做少被使用的缓存记录清除掉，然后添加新的记录
* FIFO ：先进先出算法，如果缓存中的容量已经满了，那么会将最先进入缓存中的数据清除掉
* Scheduled ：指定时间间隔清空算法，该算法会以指定的某一个时间间隔将Cache缓存中的数据清空

{{< / spoiler >}}

## Mybatis是否支持延迟加载？如果支持，它的实现原理是什么

{{< spoiler >}} 

Mybatis仅支持association关联对象和collection关联集合对象的延迟加载，association指的就是一对一，collection指的就是一对多查询。在Mybatis配置文件中，可以配置是否启用延迟加载lazyLoadingEnabled=true|false。

它的原理是，使用CGLIB创建目标对象的代理对象，当调用目标方法时，进入拦截器方法，比如调用a.getB().getName()，拦截器invoke()方法发现a.getB()是null值，那么就会单独发送事先保存好的查询关联B对象的sql，把B查询上来，然后调用a.setB(b)，于是a的对象b属性就有值了，接着完成a.getB().getName()方法的调用。这就是延迟加载的基本原理

{{< / spoiler >}}

# 实战篇





## 通常一个Xml映射文件都有一个Dao接口与之对应，请问，这个Dao接口的工作原理是什么？Dao接口里的方法，参数不同时，方法能重载吗

{{< spoiler >}} 

* Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为key值，可唯一定位一个MappedStatement，举例：com.mybatis3.mappers.StudentDao.findStudentById，可以唯一找到namespace为com.mybatis3.mappers.StudentDao下面id = findStudentById的MappedStatement
* Dao接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Dao接口生成代理proxy对象，代理对象proxy会拦截接口方法，转而执行MappedStatement所代表的sql，然后将sql执行结果返回
* 因此，Dao接口里的方法，是不能重载的，因为是全限名+方法名的保存和寻找策略

{{< / spoiler >}}



## Mybatis的Xml映射文件中，不同的Xml映射文件，id是否可以重复

{{< spoiler >}} 

不同的Xml映射文件，如果配置了namespace，那么id可以重复；如果没有配置namespace，那么id不能重复；毕竟namespace不是必须的

{{< / spoiler >}}

## Mybatis是如何进行分页的？分页插件的原理是什么

{{< spoiler >}} 

Mybatis使用RowBounds对象进行分页，它是针对ResultSet结果集执行的内存分页，而非物理分页，可以在sql内直接书写带有物理分页的参数来完成物理分页功能，也可以使用分页插件来完成物理分页。

分页插件的基本原理是使用Mybatis提供的插件接口，实现自定义插件，在插件的拦截方法内拦截待执行的sql，然后重写sql，根据dialect方言，添加对应的物理分页语句和物理分页参数。

举例：select * from student，拦截sql后重写为：select t.* from （select * from student）t limit 0，10

{{< / spoiler >}}

## MyBatis单条数据插入如何返回主键

{{< spoiler >}} 

* MySQL ： 在<insert>中将useGeneratedKeys属性设置为true，并制定keyProperty为实体对象的id

* Oracle ： 

  ```xml
  <insert id="add" parameterType="vo.Category">
  <selectKey resultType="java.lang.Short" order="BEFORE" keyProperty="id">
  SELECT SEQ_TEST.NEXTVAL FROM DUAL
  </selectKey>
  insert into category (name_zh, parent_id,
  show_order, delete_status, description
  ) values xxxx
  </insert>
  ```

{{< / spoiler >}}

## MyBatis批量插入如何返回主键列表

{{< spoiler >}} 

Xml

```xml
<insert id="batchInsert" parameterType="java.util.List" useGeneratedKeys="true" keyProperty="id" >
		INSERT INTO
		<include refid="t_shop_resource" />
		(relation_id, summary_id, relation_type)
		VALUES
		<foreach collection="list" index="index" item="shopResource" separator=",">
			(
			    #{shopResource.relationId}, #{shopResource.summaryId}, #{shopResource.relationType}
			)
		</foreach>
</insert>
```

Dao

```java
public List<ShopResource> batchinsertCallId(List<ShopResource> shopResourceList)
	{
		this.getSqlSession().insert(getStatement(SQL_BATCH_INSERT_CALL_ID), shopResourceList);
		return shopResourceList;// 重点介绍
}
```

MyBatis需要在3.3.1以上，如果在Dao中使用@Param注解，需要MyBatis3.5以上

{{< / spoiler >}}

## 插件篇

## 插件存储于哪里

{{< spoiler >}} 

初始化时，会读取插件，保存于Configuration对象的InterceptorChain中

{{< / spoiler >}}

## 如何编写插件

{{< spoiler >}} 

* 实现org.apache.ibatis.plugin.Interceptor接口
* 配置@Intercepts注解 ：在该注解中配置对哪些Mapper的哪些方法进行拦截
* 重写setProperties()方法：给自定义的拦截器传递xml配置的属性参数
* 重写plugin()方法：决定是否触发intercept()方法
* 重写intercept()方法：执行拦截内容的地方

{{< / spoiler >}}

## 插件可以拦截哪些MyBatis核心对象

{{< spoiler >}} 

Executor、StatementHandler、ParameterHandler、ResultSetHandler

{{< / spoiler >}}

## RowBounds分页插件的原理

{{< spoiler >}} 

org.apache.ibatis.executor.resultset.DefaultResultSetHandler.handleRowValuesForSimpleResultMap()方法 ： 对取到的结果集在内存中分页

{{< / spoiler >}}

## PageHelper分页插件的原理

{{< spoiler >}} 

com.github.pagehelper.PageInterceptor ：

1.将分页参数等绑定在ThreadLocal中

2.查询总数

3.改写分页sql，添加limit或者rownum等

{{< / spoiler >}}





## 占

{{< spoiler >}} 



{{< / spoiler >}}

