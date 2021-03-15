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

HashCode为什么采用31作为乘数

{{< spoiler >}} 

1. 31 是一个奇质数，如果选择偶数会导致乘积运算时数据溢出
2. 使用 31、33、37、39 和 41 作为乘积，得到的碰撞几率较小

{{< / spoiler >}}


HashMap底层实现

{{< spoiler >}} 

1.7及以前是数组+链表，1.8以后是数组+链表+红黑树

{{< / spoiler >}}



HashMap的put如何实现

{{< spoiler >}} 

1.对key的hashCode()做hash，然后再计算index;
2.校验桶数组是否被初始化 ： 如果未被初始化则进行初始化
3.校验某个桶中是否为空 : 如果为空，将本次待插入的数据放入该桶;如果桶非空
3.1 如果目前是红黑树,调用红黑树的插入方法，并将插入后的数据赋值给临时变量e
3.2 如果不是红黑树，并且当前桶的首个数据等于本次待插入的数据，将本次待插入的数据赋值给临时变量e
3.3 其他情况下遍历整个链表,查找待插入数据是否已存在于该Map，如果存在则赋值给临时变量e，如果不存在也将待插入的数据赋值给临时变量e
3.4 上边三步进行完之后，如果临时变量e非空，将Map中指定位置的值替换为本次待插入的数据，同时执行afterNodeAccess扩展方法
4.键值对数量超过阈值时，则进行扩容
5.执行afterNodeInsertion扩展方法

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

可重入锁是什么，synchronized是不是可重入锁，如果是，那么它是如何实现的

{{< spoiler >}} 
1.允许一个线程二次请求自己持有对象锁的临界资源，
2.synchronized是可重入锁
3.synchronized 锁对象有个计数器，会随着线程获 取锁后 +1 计数，当线程执行完毕后 -1，直到清零释放锁
{{< / spoiler >}}




公平锁和非公平锁的区别，为什么公平锁效率低于非公平锁

同步队列器AQS思想，以及基于AQS实现的lock


偏向锁、轻量级锁、重量级锁三者各自的应用场景

{{< spoiler >}} 
偏向锁：只有一个线程进入临界区；
轻量级锁：多个线程交替进入临界区；
重量级锁：多个线程同时进入临界区
{{< / spoiler >}}



# 并发篇

ConcurrentHashMap为什么比HashMap安全又高效
{{< spoiler >}} 
jdk7分段锁，jdk8cas
{{< / spoiler >}}

为了实现可见性，volatile和synchronized所使用的方法有何不同

{{< spoiler >}} 
volatile通过内存屏障来实现，而synchronized通过系统内核互斥实现，相当于JMM中的lock、unlock，退出代码块时刷新变量到主内存
{{< / spoiler >}}



# 线程篇

## 线程和进程的区别

{{< spoiler >}} 
进程是资源分配的最小单位，线程是CPU调度的最小单位

{{< / spoiler >}}

## 线程有哪几种状态

![java-线程状态图](https://gitee.com/1162492411/pic/raw/master/java-线程状态图.jpeg)



## 线程的实现方式有哪些，这些方式之间有什么区别

{{< spoiler >}} 

1）继承Thread类
2）实现Runnable接口再调用start方法 
3）Thread 是类，而Runnable是接口，并且Runnable可以实现资源共享，例如卖票的场景
{{< / spoiler >}}

## Thread类包含start()和run()方法，它们的区别是什么

{{< spoiler >}} 
start() : 它的作用是启动一个新线程，新线程会执行相应的run()方法。start()不能被重复调用。

run()   : run()就和普通的成员方法一样，可以被重复调用。单独调用run()的话，会在当前线程中执行run()，而并不会启动新线程
{{< / spoiler >}}

## 为什么notify(), wait()等函数定义在Object中，而不是Thread中

{{< spoiler >}} 
notify(), wait()依赖于“同步锁”，而“同步锁”是对象锁持有，并且每个对象有且仅有一个
{{< / spoiler >}}

## sleep() 与 wait()的比较

{{< spoiler >}} 
wait()的作用是让当前线程由“运行状态”进入“等待(阻塞)状态”的同时，也会释放同步锁。
而sleep()的作用是也是让当前线程由“运行状态”进入到“休眠(阻塞)状态”。
但是，wait()会释放对象的同步锁，而sleep()则不会释放锁
{{< / spoiler >}}

## join()方法的作用和原理

{{< spoiler >}} 
作用是让“主线程”等待“子线程”结束之后才能继续运行，原理就是对应的native方法中先是主线程调用了wait然后在子线程threadA执行完毕之后，JVM会调用lock.notify_all(thread)来唤醒就是主线程
{{< / spoiler >}}

## nofity和nofityAll的区别

{{< spoiler >}} 
notify()方法只随机唤醒一个 wait 线程，而notifyAll()方法唤醒所有 wait 线程
{{< / spoiler >}}

## 如何实现线程安全，各个实现方法有什么区别

{{< spoiler >}} 

{{< / spoiler >}}

## JDK自带的有哪几种线程池

{{< spoiler >}} 



{{< / spoiler >}}

## 如何设计一个线程池

{{< spoiler >}} 



{{< / spoiler >}}

## 线程池的参数有哪些，各自作用是什么





## execute和submit的区别与联系

{{< spoiler >}} 

* 任务类型 ：execute只能提交Runnable类型的任务，而submit既能提交Runnable类型任务也能提交Callable类型任务
* 异常 ： execute直接抛出异常，submit会吃掉异常，可用future的get捕获
* 顶层接口 ：execute所属顶层接口是Executor,submit所属顶层接口是ExecutorService

{{< / spoiler >}}



并发编程三要素？
实现可见性的方法有哪些？
多线程的价值？
创建线程的有哪些方式？
创建线程的三种方式的对比？
常用的并发工具类有哪些？
CyclicBarrier 和 CountDownLatch 的区别
synchronized 的作用？
volatile 关键字的作用
sleep 方法和 wait 方法有什么区别?
什么是 CAS
CAS 的问题
什么是 Future？
什么是 AQS
AQS 支持两种同步方式
ReadWriteLock 是什么
FutureTask 是什么
synchronized 和 ReentrantLock 的区别
什么是乐观锁和悲观锁
线程 B 怎么知道线程 A 修改了变量
synchronized、volatile、CAS 比较
为什么 wait()方法和 notify()/notifyAll()方法要在同步块中被调用
多线程同步有哪几种方法？
线程的调度策略
ConcurrentHashMap 的并发度是什么？
Linux 环境下如何查找哪个线程使用 CPU 最长
死锁的原因？
Java 死锁以及如何避免？
怎么唤醒一个阻塞的线程？
不可变对象对多线程有什么帮助？
什么是多线程的上下文切换？
如果你提交任务时，线程池队列已满，这时会发生什么？
Java 中用到的线程调度算法是什么？
什么是线程调度器(Thread Scheduler)和时间分片(TimeSlicing)？
什么是自旋？
Java Concurrency API 中的 Lock 接口(Lock interface)是什么？对比同步它有什么优势？



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








# 设计模式篇



简要介绍各设计模式中的关键点

{{< spoiler >}} 

单例模式：某个类只能有一个实例，提供一个全局的访问点。

简单工厂：一个工厂类根据传入的参量决定创建出那一种产品类的实例。

工厂方法：定义一个创建对象的接口，让子类决定实例化那个类。

抽象工厂：创建相关或依赖对象的家族，而无需明确指定具体类。

建造者模式：封装一个复杂对象的构建过程，并可以按步骤构造。

原型模式：通过复制现有的实例来创建新的实例。

 

适配器模式：将一个类的方法接口转换成客户希望的另外一个接口。

组合模式：将对象组合成树形结构以表示“”部分-整体“”的层次结构。

装饰模式：动态的给对象添加新的功能。

代理模式：为其他对象提供一个代理以便控制这个对象的访问。

亨元（蝇量）模式：通过共享技术来有效的支持大量细粒度的对象。

外观模式：对外提供一个统一的方法，来访问子系统中的一群接口。

桥接模式：将抽象部分和它的实现部分分离，使它们都可以独立的变化。

 

模板模式：定义一个算法结构，而将一些步骤延迟到子类实现。

解释器模式：给定一个语言，定义它的文法的一种表示，并定义一个解释器。

策略模式：定义一系列算法，把他们封装起来，并且使它们可以相互替换。

状态模式：允许一个对象在其对象内部状态改变时改变它的行为。

观察者模式：对象间的一对多的依赖关系。

备忘录模式：在不破坏封装的前提下，保持对象的内部状态。

中介者模式：用一个中介对象来封装一系列的对象交互。

命令模式：将命令请求封装为一个对象，使得可以用不同的请求来进行参数化。

访问者模式：在不改变数据结构的前提下，增加作用于一组对象元素的新功能。

责任链模式：将请求的发送者和接收者解耦，使的多个对象都有处理这个请求的机会。

迭代器模式：一种遍历访问聚合对象中各个元素的方法，不暴露该对象的内部结构。

{{< / spoiler >}}

# 实战排查



如何排查线上出现的JVM问题



