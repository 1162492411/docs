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



{{< spoiler >}} 



{{< / spoiler >}}



MyBatis有哪些主要组件

{{< spoiler >}} 

1）SqlSessionFactory：负责sqlSession的创建、销毁

2）SqlSession ： 负责提供给用户可以操作的api，如insert(),insertBatch()等

3）Executor ： 负责执行对数据库的操作

4）StatementHandler ：Executor将工作委托给StatementHandler执行



{{< / spoiler >}}



Mybatis的工作流程



 **#** 和**$**的区别



接口如何绑定





MyBatis的一级缓存和二级缓存





