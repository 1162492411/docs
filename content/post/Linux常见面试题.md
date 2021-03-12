---
title: "Linux常见面试题"
date: 2020-12-23T08:59:41+08:00
draft: false
toc: TRUE
categories:
  - Linux
  - 面试题
tags:
  - Linux
  - 面试题
---


{{< spoiler >}} 

{{< / spoiler >}}

# 内存

如何查看系统内存

如何清除系统内存


# 内核

用户态如何切换到内核态
{{< spoiler >}} 
1. 系统调用，例如foerk() 
2. 异常
3. 外围设备的中断
{{< / spoiler >}}


# 系统函数

select和epoll的区别




如何查看系统负载

# CPU

cpu的load值不同时，cpu有哪几种模式(空闲，轻度负载，高负载这个吗)

# 文件系统

 Linux，查找磁盘上最大的文件的命令