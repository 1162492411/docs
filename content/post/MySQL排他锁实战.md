---
title: "MySQL排他锁实战"
date: 2020-12-21T21:52:18+08:00
draft: false
toc: true
categories:
  - MySQL
  - 锁
tags:
  - MySQL
  - 分布式锁
---
# 1. 需求背景

​    基于MySQL/Oracle数据库实现分布式锁，保证一个项目中的定时任务代码在多台机器中同时执行时最多有一个任务可以成功获取锁，其他任务获取锁失败



# 2. 排他锁介绍

## 2.1 概念

​    如果事务T对数据A加上排他锁(exclusive lock，即X锁)后，则其他事务不能再对A加任任何类型的锁，将会等待事务T结束。获准排他锁的事务既能读数据，又能修改数据.

​    在MySQL中，X锁仅适用于InnoDB引擎，而且必须在事务中才能生效，根据where条件是否通过索引命中数据，MySQL中的X锁分为行锁与表锁 ：命中数据时采用行锁，本质是对索引加锁；其他情况下均为表锁（例如没有where条件对应的数据，where后的字段没有索引）；特殊地，如果表数据过少，InnoDB引擎也可能将SQL优化为表锁，这种情况下可以通过force index来强制使用索引。

## 2.2 用法示例   

### 2.2.1 基本用法

 select … for update;

例如：select * from goods where id = 1 for update;

### 2.2.2 进阶用法

```
# nowait --> 不再等待事务而是立即返回结果，如果发现where条件的结果集被其他事务锁定则立即返回失败，该语法自MySQL的8.0版本开始支持，Oracle支持
select ... for update no wait;

# wait --> 最多等待指定的时间x秒之后返回结果，该语法在Orale中支持
select ... for update wait x;

# skip locked --> 如果数据锁定时跳过锁定的数据,该语法自MySQL的8.0版本开始支持，Oracle支持
select ... for update skip locked;
```



# 3.准备数据

## 3.1 准备表

```
create table t_gdts_sync_flag
(
    n_id         bigint auto_increment comment '流水id'
        primary key,
    c_company_id varchar(20) null comment '集团id,对应t_gdts_company_rel的n_id',
    n_type       tinyint(2)  null comment '同步标识类型,1集团,2部门,3人员',
    c_status     varchar(10) null comment '同步状态,sync/idle'
) comment '同步标识表';
```

## 3.2 准备数据

```
INSERT INTO gropt.t_gdts_sync_flag (n_id, c_company_id, n_type, c_status) VALUES (75, '1326009432085127169', 1, 'idle');
INSERT INTO gropt.t_gdts_sync_flag (n_id, c_company_id, n_type, c_status) VALUES (76, '1326009432085127169', 2, 'idle');
INSERT INTO gropt.t_gdts_sync_flag (n_id, c_company_id, n_type, c_status) VALUES (77, '1326009432085127169', 3, 'idle');
```

# 4. 实战

 ## 4.1 定义用于获取锁的线程池

```

```

## 4.2 获取锁的SQL语句

```

```

## 4.3 获取锁的Service代码

```

```

## 4.4 定时任务代码

```

```
