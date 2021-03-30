---
title: "Spring常见面试题"
date: 2020-12-23T21:04:04+08:00
draft: false
categories:
  - 面试题
  - Spring
tags:
  - 面试题
  - Spring
---

# IOC

## 创建IOC容器的过程

{{< spoiler >}} 

以最原始的XmlBeanFactory为例讲解,

1.创建Ioc配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息 ；

 2.创建一个BeanFactory，这里使用了DefaultListableBeanFactory   ； 

3.创建一个载入BeanDefinition的读取器，这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition  ； 

4.然后将上面定位好的Resource，通过一个回调配置给BeanFactory  ； 

 5.从定位好的资源位置读入配置信息，具体的解析过程由XmlBeanDefinitionReader完成 ；  

 6.完成整个载入和注册Bean定义之后，需要的Ioc容器就初步建立起来了

{{< / spoiler >}}

## 配置Bean的方法

{{< spoiler >}} 

- 基于XML文件进行配置
- 基于注解进行配置
- 基于Java程序进行配置

{{< / spoiler >}}

## Spring bean的初始化顺序

{{< spoiler >}} 

* Constructor 
* BeanPostProsser的before
* @PostConstruct(其实也是一个BeanPostProsser的before)
* InitializingBean
* init-method
* BeanPostProsser的after

{{< / spoiler >}}

## Spring Bean的销毁顺序

{{< spoiler >}} 

* destory
* destroy-method

{{< / spoiler >}}

## Spring中Bean的生命周期

![](https://gitee.com/1162492411/pic/raw/master/Spring-Bean.png)

## Bean的作用域有哪些

{{< spoiler >}} 

* single ：单例，适用于无状态的bean，默认
* prototype : 每次获取都返回一个新的实例，适用于有状态的bean，默认启动时不加载prototype的bean
* request ：每次http请求会创建一个新的实例
* session ：每次session共享一个新的实例
* globalSession ：全局session共享一个新的实例

{{< / spoiler >}}

## Spring如何解决setter方式的循环依赖

{{< spoiler >}} 

singletonFactories ： 单例对象工厂的cache；
earlySingletonObjects ：提前暴光的单例对象的工厂Cache；
singletonObjects：单例对象的cache；

1. 实例化a，先把beanName放到singletonsCurrentlyInCreation中，然后调用无参构造方法实例化bean,然后构造一个singletonFactory对象放到singletonFactories中，暴露给其它可能的依赖;
2. 最后装配属性时，发现需要注入b,那么就开始构造b,构造b的流程和上一步一致

{{< / spoiler >}}

## Spring无法解决哪些方式的循环依赖

{{< spoiler >}} 

* 构造器 ： a构造器中依赖b
* setter的prototype ： prototype的bean启动时不加载，不在三级缓存中

{{< / spoiler >}}

## 有哪些常见的Resource

{{< spoiler >}} 

- UrlResource：访问网络资源的实现类。
- ClassPathResource：访问类加载路径里资源的实现类。
- FileSystemResource：访问文件系统里资源的实现类。
- ServletContextResource：访问相对于 ServletContext 路径里的资源的实现类：
- InputStreamResource：访问输入流资源的实现类。
- ByteArrayResource：访问字节数组资源的实现类。 这些 Resource 实现类，针对不同的的底层资源，提供了相应的资源访问逻辑，并提供便捷的包装，以利于客户端程序的资源访问

{{< / spoiler >}}

## 有哪些常见的Context

{{< spoiler >}} 

- FileSystemXmlApplicationContext：该容器从 XML 文件中加载已被定义的 bean。在这里，你需要提供给构造器 XML 文件的完整路径
- ClassPathXmlApplicationContext：该容器从 XML 文件中加载已被定义的 bean。在这里，你不需要提供 XML 文件的完整路径，只需正确配置 CLASSPATH 环境变量即可，因为，容器会从 CLASSPATH 中搜索 bean 配置文件。
- WebXmlApplicationContext：该容器会在一个 web 应用程序的范围内加载在 XML 文件中已被定义的 bean

{{< / spoiler >}}

## 有哪些内置的BeanPostProcessor

{{< spoiler >}} 

* ApplicationContextAwareProcessor
  * 作用 ：向组件中注入IOC容器，提供对xxxAware类接口的实现
  * 注册 ：Spring容器的refresh方法内部调用prepareBeanFactory方法，prepareBeanFactory方法会添加ApplicationContextAwareProcessor到BeanFactory中
* CommonAnnotationBeanPostProcessor
  * 作用 ：提供对@PostConstruct、@PreDestroy、@Resource、WebServiceRef注解的支持
  * 注册 ：在AnnotationConfigUtils类的registerAnnotationConfigProcessors方法中被封装成RootBeanDefinition并注册到Spring容器中
* AutowiredAnnotationBeanPostProcessor
  * 作用 ：提供对@Autowired、@Value、@Lookup和@Inject注解的实现
  * 注册 ：在AnnotationConfigUtils类的registerAnnotationConfigProcessors方法被注册到Spring容器中
* RequiredAnnotationBeanPostProcessor
  * 作用 ：提供对@Required注解的实现
  * 注册 ：在AnnotationConfigUtils类的registerAnnotationConfigProcessors方法被注册到Spring容器中
* BeanValidationPostProcessor
  * 作用 ：提供对JSR-303验证的支持
* AbstractAutoProxyCreator
  * 作用 ：提供对AOP的支持
  * 注册 ：默认不注册。在SpringBoot中加入aop-starter之后，会触发AopAutoConfiguration自动化配置，然后将AnnotationAwareAspectJAutoProxyCreator注册到Spring容器中
* MethodValidationPostProcessor
  * 作用 ：支持方法级别的JSR-303规范。需要在类上加上@Validated注解，以及在方法的参数中加上验证注解，比如@Max，@Min，@NotEmpty
  * 注册 ： 默认不注册
* ScheduledAnnotationBeanPostProcessor
  * 作用 ：Spring Scheduling功能对bean中使用了@Scheduled注解的方法进行调度处理
  * 注册 ： 默认不注册，添加@EnableScheduling会被注册
* AsyncAnnotationBeanPostProcessor
  * 作用 ：提供对@Async注解的实现，通过AOP实现
  * 注册 ： 默认不注册，添加@EnableAsync会被注册

{{< / spoiler >}}

# MVC

## MVC的启动流程







# 注解

## 如何开启注解功能

{{< spoiler >}} 

* XML方式 ：在Spring配置文件中配置 context:annotation-config/元素
* 注解方式 ： 使用**@ComponentScan**

{{< / spoiler >}}

## @Autowired和@Resources注解的异同

{{< spoiler >}} 

1.autowired默认按类型查找对象，resources默认按照名称查找对象

2.autowired是spirng提供的注解，resouces是j2ee提供的注解，但是二者都是jsr标准下的注解

3.两个注解都可以用在字段上

{{< / spoiler >}}

## 同样的接口存在多个实现时如何指定某一个实现

{{< spoiler >}} 

1.@Qualifer 2.@Primary

{{< / spoiler >}}

# AOP

aop的底层实现

动态代理有哪些实现方式，有什么区别



# 事务

Spring事务底层如何实现



## 事务的五个隔离级别

{{< spoiler >}} 

**TransactionDefinition 接口中定义了五个表示隔离级别的常量：**

- **TransactionDefinition.ISOLATION_DEFAULT:**  使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
- **TransactionDefinition.ISOLATION_READ_COMMITTED:**   允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
- **TransactionDefinition.ISOLATION_REPEATABLE_READ:**  对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
- **TransactionDefinition.ISOLATION_SERIALIZABLE:**   最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别

{{< / spoiler >}}

## Spring事务的七个传播级别

{{< spoiler >}} 

- PROPAGATION_REQUIRED: 支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。
- PROPAGATION_SUPPORTS: 支持当前事务，如果当前没有事务，就以非事务方式执行。
- PROPAGATION_MANDATORY: 支持当前事务，如果当前没有事务，就抛出异常。
- PROPAGATION_REQUIRES_NEW: 新建事务，如果当前存在事务，把当前事务挂起。
- PROPAGATION_NOT_SUPPORTED: 以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。
- PROPAGATION_NEVER: 以非事务方式执行，如果当前存在事务，则抛出异常。
- PROPAGATION_NESTED:如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。

{{< / spoiler >}}

## 嵌套事务和挂起事务的区别是什么

{{< spoiler >}} 

* 嵌套事务 ：本质上还是同一个事务的不同保存点，如果涉及到外层事务回滚，则内层的也将会被回滚
* 挂起事务 ：对应的是一个新的事务，拿到的是新的资源，所以外层事务回滚时，不影响内层事务

{{< / spoiler >}}

# 拷贝

1. 什么是浅拷贝和深拷贝，有什么区别
2. 常用的实体拷贝有哪几种方式，各自是如何实现的
3. Spring的BeanUtils拷贝存在哪些细节问题

# 设计模式

## Spring中用到了哪些设计模式

{{< spoiler >}} 

（1）工厂模式：BeanFactory就是简单工厂模式的体现，用来创建对象的实例；

（2）单例模式：Bean默认为单例模式。

（3）代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术；

（4）模板方法：用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。

（5）观察者模式：定义对象键一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知被动更新，如Spring中listener的实现–ApplicationListener。

{{< / spoiler >}}

# SpringBoot
## 简述配置文件加载顺序
{{< spoiler >}} 
由高到低依次为：

* 命令行参数。所有的配置都可以在命令行上进行指定；
* 来自java:comp/env的JNDI属性；
* Java系统属性（System.getProperties()）；
* 操作系统环境变量 ；
* jar包外部的application-{profile}.properties或application.yml(带spring.profile)配置文件
* jar包内部的application-{profile}.properties或application.yml(带spring.profile)配置文件 再来加载不带profile
* jar包外部的application.properties或application.yml(不带spring.profile)配置文件
* jar包内部的application.properties或application.yml(不带spring.profile)配置文
* @Configuration注解类上的@PropertySource

{{< / spoiler >}}

## 概述SpringBoot的启动流程

{{< spoiler >}} 

1. 从`spring.factories`配置文件中**加载`EventPublishingRunListener`对象**

2. **准备环境变量**，包括系统变量，环境变量，命令行参数，默认变量，`servlet`相关配置变量，随机值以及配置文件等
3. 控制台**打印SpringBoot的`bannner`标志**
4. **根据不同类型环境创建不同类型的`applicationcontext`容器**
5. 从`spring.factories`配置文件中**加载`FailureAnalyzers`对象**,用来报告SpringBoot启动过程中的异常
6. **为刚创建的容器对象做一些初始化工作**，准备一些容器属性值等，对`ApplicationContext`应用一些相关的后置处理和调用各个`ApplicationContextInitializer`的初始化方法来执行一些初始化逻辑等；
7. **刷新容器**，这一步至关重要。比如调用`bean factory`的后置处理器，注册`BeanPostProcessor`后置处理器，初始化事件广播器且广播事件，初始化剩下的单例`bean`和SpringBoot创建内嵌的`Tomcat`服务器等等重要且复杂的逻辑都在这里实现，主要步骤可见代码的注释，关于这里的逻辑会在以后的spring源码分析专题详细分析；
8. **执行刷新容器后的后置处理逻辑**，注意这里为空方法；
9. **调用`ApplicationRunner`和`CommandLineRunner`的run方法**，我们实现这两个接口可以在spring容器启动后需要的一些东西比如加载一些业务数据等;
10. **报告启动异常**，即若启动过程中抛出异常，此时用`FailureAnalyzers`来报告异常;
11. 最终**返回容器对象**，这里调用方法没有声明对象来接收

{{< / spoiler >}}

## 概述SpringBoot的自动装配原理

{{< spoiler >}} 

* 判断自动装配开关是否打开。默认`spring.boot.enableautoconfiguration=true`，可在 `application.properties` 或 `application.yml` 中设置
* 获取`EnableAutoConfiguration`注解中的 `exclude` 和 `excludeName`
* 获取需要自动装配的所有配置类，读取`META-INF/spring.factories`
* 通过@Conditionalxxx的结果判断需要加载哪些配置类

{{< / spoiler >}}


