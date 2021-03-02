---
title: "Java常见面试题"
date: 2020-12-27T14:57:17+08:00
draft: false
categories:
  - 面试题
  - Java
tags:
  - 面试题
  - Java
  - Java基础
---



{{< spoiler >}} 



{{< / spoiler >}}

# 基础篇

Object都有哪些方法，各自作用是什么

{{< spoiler >}} 

对象相等的相关方法：equals()、hashcode();

对象的基本方法 ： toString()、getClass()、clone()、finalize()

锁相关方法 : wait()、nofity()、notifyAll()

{{< / spoiler >}}

Java有哪些修饰符/作用域

{{< spoiler >}} 

private、default、protected、public

{{< / spoiler >}}

# 数据结构篇

HashMap底层实现

{{< spoiler >}} 

1.7及以前是数组+链表，1.8以后是数组+链表+红黑树

{{< / spoiler >}}



HashMap的put如何实现

{{< spoiler >}} 



{{< / spoiler >}}

HashMap扩容策略在1.7和1.8有什么区别

{{< spoiler >}} 



{{< / spoiler >}}

HashMap是否是线程安全的，扩容时的锁在什么情况下会出现

{{< spoiler >}} 



{{< / spoiler >}}

如何实现线程安全的HashMap

{{< spoiler >}} 



{{< / spoiler >}}

# IO篇

IO、BIO、NIO，阻塞与非阻塞的区别



# 锁篇

可重入锁是什么，synchronized是不是可重入锁

{{< spoiler >}} 



{{< / spoiler >}}

wait和sleep的区别

{{< spoiler >}} 





{{< / spoiler >}}

nofity和nofityAll的区别

{{< spoiler >}} 



{{< / spoiler >}}

公平锁和非公平锁的区别，为什么公平锁效率低于非公平锁

同步队列器AQS思想，以及基于AQS实现的lock







# 并发篇

ConcurrentHashMap为什么比HashMap安全又高效(jdk7分段锁，jdk8cas)





# 线程篇

线程和进程的区别

{{< spoiler >}} 



{{< / spoiler >}}

线程的实现方式有哪些

{{< spoiler >}} 

1）继承Thread类2）实现Runnable接口再调用start方法

{{< / spoiler >}}

如何实现线程安全，各个实现方法有什么区别

{{< spoiler >}} 



{{< / spoiler >}}

volatile关键字的使用

{{< spoiler >}} 

线程有哪几种状态



{{< / spoiler >}}

JDK自带的有哪几种线程池

{{< spoiler >}} 



{{< / spoiler >}}

如何设计一个线程池

{{< spoiler >}} 



{{< / spoiler >}}

线程池的参数有哪些，各自作用是什么





# JVM篇

JVM分为哪几块，其中哪几块是线程共享的，每一块存储什么

{{< spoiler >}} 



{{< / spoiler >}}

内存溢出和内存泄露的区别

{{< spoiler >}} 



{{< / spoiler >}}

如何判断哪些对象需要被GC

{{< spoiler >}} 



{{< / spoiler >}}

GC的方法

{{< spoiler >}} 



{{< / spoiler >}}

MinGC与FullGC各自指什么

{{< spoiler >}} 



{{< / spoiler >}}

HotSpot的GC算法以及7种垃圾回收期

{{< spoiler >}} 



{{< / spoiler >}}

类加载的过程

{{< spoiler >}} 



{{< / spoiler >}}



如何排查线上出现的JVM问题





# 设计模式篇



