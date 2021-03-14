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



# 基础篇

##  **#** 和**$**的区别

{{< spoiler >}} 

- `${}`是 Properties 文件中的变量占位符，它可以用于标签属性值和 sql 内部，属于静态文本替换，比如${driver}会被静态替换为`com.mysql.jdbc.Driver`。
- `#{}`是 sql 的参数占位符，MyBatis 会将 sql 中的`#{}`替换为?号，在 sql 执行前会使用 PreparedStatement 的参数设置方法，按序给 sql 的?号占位符设置参数值，比如 ps.setInt(0, parameterValue)，`#{item.name}` 的取值方式为使用反射从参数对象中获取 item 对象的 name 属性值，相当于 `param.getItem().getName()`。

{{< / spoiler >}}





# 核心原理篇

## MyBatis有哪些主要组件

{{< spoiler >}} 

1）SqlSessionFactory：负责sqlSession的创建、销毁

2）SqlSession ： 负责提供给用户可以操作的api，如insert(),insertBatch()等

3）Executor ： 负责执行对数据库的操作

4）StatementHandler ：Executor将工作委托给StatementHandler执行

{{< / spoiler >}}



## Mybatis的工作流程

{{< spoiler >}} 



{{< / spoiler >}}



## 接口如何绑定





## MyBatis的一级缓存和二级缓存





