---
title: "Redis常见面试题"
date: 2020-12-22T21:29:59+08:00
draft: false
categories:
  - Redis
  - NoSQL
  - 数据库
tags:
  - Redis
  - 中间件
  - NoSQL
  - 数据库
---



# 原理篇

1. 如何理解Redis的通讯协议resp协议
2. 如何理解Redis的cluster bus的gossip协议
3. 为什么早期版本Redis是单线程的
4. Redis为什么速度快
5. Redis的主从复制原理是什么
6. Redis如何划分内存
7. Redis的存储细节
8. Redis如何实现渐进式hash进行扩容
9. Redis事务的CAS



# 基础篇

1. Redis都有哪些基础的数据结构
2. Redis有哪些高级的数据结构，对应的使用场景是什么
3. Redis如何做到数据持久化，这些方式各自有什么优缺点
4. Redis AOF持久化的触发条件有哪些
5. Redis慢查询如何开启
6. Redis的默认内存为多大
7. Redis的淘汰策略有哪些
8. Redis的过期策略有哪些，这些删除策略各自有什么优缺点
9. Redis的Pipeline如何理解
10. Redis如何设置过期时间
11. Redis支持哪些集群模式





# 实战篇



1. Redis有哪些常用场景
2. 如何理解缓存穿透、缓存击穿、缓存雪崩、缓存预热
3. Redis内存使用满会出现什么现象
4. Redis如何实现定时队列
5. Redis如何实现消息队列
6. Redis如何实现异步队列
7. Redis的并发竞争如何解决
8. Redis和数据库如何实现数据一致性
9. 有哪些基于Redis实现的分布式锁
10. Redis中的key过期了是否立即释放内存，为什么
11. 如何保证Redis的高可用和高并发
12. Redis集群模式下，redis的key如何寻址，分布式寻址都有哪些算法
13. 一致性hash算法是什么
14. Redis变慢如何排查
15. 如何从海量key中查找出某一个固定前缀的key
16. 如何为Redis一次增加大批量数据

