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

{{< spoiler >}} 

free

{{< / spoiler >}}

如何清理系统内存

{{< spoiler >}} 

```shell
# 仅清除页面缓存（PageCache） 
echo 1 > /proc/sys/vm/drop_caches
# 清除目录项和inode
echo 2 > /proc/sys/vm/drop_caches
# 清除页面缓存，目录项和inode
echo 3 > /proc/sys/vm/drop_caches
```

{{< / spoiler >}}


# 内核

用户态如何切换到内核态
{{< spoiler >}} 
1. 系统调用，例如fork() 

2. 异常

3. 外围设备的中断

  {{< / spoiler >}}


# 系统函数

select和epoll的区别





# CPU

cpu的load值不同时，cpu有哪几种模式(空闲，轻度负载，高负载)

# 文件系统

 Linux，查找磁盘上最大的文件的命令

# 常用命令

 ## 如何查看系统负载

{{< spoiler >}} 

使用top命令 ，执行后效果如下

```
top - 01:06:48 up  1:22,  1 user,  load average: 0.06, 0.60, 0.48
Tasks:  29 total,   1 running,  28 sleeping,   0 stopped,   0 zombie
Cpu(s):  0.3% us,  1.0% sy,  0.0% ni, 98.7% id,  0.0% wa,  0.0% hi,  0.0% si
Mem:    191272k total,   173656k used,    17616k free,    22052k buffers
Swap:   192772k total,        0k used,   192772k free,   123988k cached

PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
1379 root      16   0  7976 2456 1980 S  0.7  1.3   0:11.03 sshd
14704 root      16   0  2128  980  796 R  0.7  0.5   0:02.72 top
1 root      16   0  1992  632  544 S  0.0  0.3   0:00.90 init
2 root      34  19     0    0    0 S  0.0  0.0   0:00.00 ksoftirqd/0
3 root      RT   0     0    0    0 S  0.0  0.0   0:00.00 watchdog/0
```

第一行是任务队列信息 ：分别表示系统当前时间、系统运行时间、当前登陆用户数量、系统负载(即任务队列的平均长度,三个数值分别为 1分钟、5分钟、15分钟前到现在的平均值)

第二、三行是进程和CPU的信息 ： 

```
total 进程总数
running 正在运行的进程数
sleeping 睡眠的进程数
stopped 停止的进程数
zombie 僵尸进程数
Cpu(s): 
0.3% us 用户空间占用CPU百分比
1.0% sy 内核空间占用CPU百分比
0.0% ni 用户进程空间内改变过优先级的进程占用CPU百分比
98.7% id 空闲CPU百分比
0.0% wa 等待输入输出的CPU时间百分比
0.0%hi：硬件CPU中断占用百分比
0.0%si：软中断占用百分比
0.0%st：虚拟机占用百分比
```

第四、五行是内存信息

```
Mem:
191272k total    物理内存总量
173656k used    使用的物理内存总量
17616k free    空闲内存总量
22052k buffers    用作内核缓存的内存量
Swap: 
192772k total    交换区总量
0k used    使用的交换区总量
192772k free    空闲交换区总量
123988k cached    缓冲的交换区总量
```

剩下的是进程信息区统计信息区域，它显示了各个进程的详细信息，各列的含义如下

```
序号  列名    含义
a    PID     进程id
b    PPID    父进程id
c    RUSER   Real user name
d    UID     进程所有者的用户id
e    USER    进程所有者的用户名
f    GROUP   进程所有者的组名
g    TTY     启动进程的终端名。不是从终端启动的进程则显示为 ?
h    PR      优先级
i    NI      nice值。负值表示高优先级，正值表示低优先级
j    P       最后使用的CPU，仅在多CPU环境下有意义
k    %CPU    上次更新到现在的CPU时间占用百分比
l    TIME    进程使用的CPU时间总计，单位秒
m    TIME+   进程使用的CPU时间总计，单位1/100秒
n    %MEM    进程使用的物理内存百分比
o    VIRT    进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
p    SWAP    进程使用的虚拟内存中，被换出的大小，单位kb。
q    RES     进程使用的、未被换出的物理内存大小，单位kb。RES=CODE+DATA
r    CODE    可执行代码占用的物理内存大小，单位kb
s    DATA    可执行代码以外的部分(数据段+栈)占用的物理内存大小，单位kb
t    SHR     共享内存大小，单位kb
u    nFLT    页面错误次数
v    nDRT    最后一次写入到现在，被修改过的页面数。
w    S       进程状态(D=不可中断的睡眠状态,R=运行,S=睡眠,T=跟踪/停止,Z=僵尸进程)
x    COMMAND 命令名/命令行
y    WCHAN   若该进程在睡眠，则显示睡眠中的系统函数名
z    Flags   任务标志，参考 sched.h
```

{{< / spoiler >}}



## vmstat命令各字段含义

{{< spoiler >}}

- r: 运行队列中进程数量（当数量大于CPU核数表示有阻塞的线程）
- b: 等待IO的进程数量
- swpd: 使用虚拟内存大小
- free: 空闲物理内存大小
- buff: 用作缓冲的内存大小(内存和硬盘的缓冲区)
- cache: 用作缓存的内存大小（CPU和内存之间的缓冲区）
- si: 每秒从交换区写到内存的大小，由磁盘调入内存
- so: 每秒写入交换区的内存大小，由内存调入磁盘
- bi: 每秒读取的块数
- bo: 每秒写入的块数
- in: 每秒中断数，包括时钟中断。
- cs: 每秒上下文切换数。
- us: 用户进程执行时间百分比(user time)
- sy: 内核系统进程执行时间百分比(system time)
- wa: IO等待时间百分比
- id: 空闲时间百分比

{{< / spoiler >}}

