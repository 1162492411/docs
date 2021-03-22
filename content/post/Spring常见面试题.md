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



{{< spoiler >}} 



{{< / spoiler >}}



# IOC

Spring如何解决setter方式的循环依赖

{{< spoiler >}} 

singletonFactories ： 单例对象工厂的cache；
earlySingletonObjects ：提前暴光的单例对象的Cache；
singletonObjects：单例对象的cache；

1. 实例化a，先把beanName放到singletonsCurrentlyInCreation中，然后调用无参构造方法实例化bean,然后构造一个singletonFactory对象放到singletonFactories中，暴露给其它可能的依赖;
2. 最后装配属性时，发现需要注入b,那么就开始构造b,构造b的流程和上一步一致

{{< / spoiler >}}

Spring中Bean的生命周期

![](https://gitee.com/1162492411/pic/raw/master/Spring-Bean.png)

@Autowired和@Resources注解的异同



{{< spoiler >}} 

1.autowired默认按类型查找对象，resources默认按照名称查找对象

2.autowired是spirng提供的注解，resouces是j2ee提供的注解，但是二者都是jsr标准下的注解

3.两个注解都可以用在字段上

{{< / spoiler >}}



同样的接口存在多个实现时如何指定某一个实现



{{< spoiler >}} 

1.@Qualifer 2.@Primary

{{< / spoiler >}}



创建IOC容器的过程

{{< spoiler >}} 

以最原始的XmlBeanFactory为例讲解,

1.创建Ioc配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息 ；  2.创建一个BeanFactory，这里使用了DefaultListableBeanFactory   ； 3.创建一个载入BeanDefinition的读取器，这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition  ；  4.然后将上面定位好的Resource，通过一个回调配置给BeanFactory  ；  5.从定位好的资源位置读入配置信息，具体的解析过程由XmlBeanDefinitionReader完成 ；   6.完成整个载入和注册Bean定义之后，需要的Ioc容器就初步建立起来了

{{< / spoiler >}}



Spring bean的初始化顺序

{{< spoiler >}} 

1. Constructor; 2. @PostConstruct; 3. InitializingBean; 4. init-method

{{< / spoiler >}}



# AOP

aop的底层实现

动态代理有哪些实现方式，有什么区别



# 事务

Spring事务底层如何实现

Spring事务的七个传播级别，默认是哪个

{{< spoiler >}} 



{{< / spoiler >}}

# 拷贝

1. 什么是浅拷贝和深拷贝，有什么区别
2. 常用的实体拷贝有哪几种方式，各自是如何实现的
3. Spring的BeanUtils拷贝存在哪些细节问题

# SpringBoot
## 简述SpringBoot的配置文件加载顺序
{{< spoiler >}} 
由高到低依次为：
命令行参数。所有的配置都可以在命令行上进行指定；
来自java:comp/env的JNDI属性；
Java系统属性（System.getProperties()）；
操作系统环境变量 ；
jar包外部的application-{profile}.properties或application.yml(带spring.profile)配置文件
jar包内部的application-{profile}.properties或application.yml(带spring.profile)配置文件 再来加载不带profile
jar包外部的application.properties或application.yml(不带spring.profile)配置文件
jar包内部的application.properties或application.yml(不带spring.profile)配置文件
@Configuration注解类上的@PropertySource
{{< / spoiler >}}






