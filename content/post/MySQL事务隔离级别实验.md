---
title: "MySQL事务隔离级别实验"
date: 2021-04-01T16:32:25+08:00
draft: true
  - 数据库
  - 隔离级别
tags:
  - 数据库
  - 隔离级别
  - MySQL
---

# 实验环境

* MySQL v5.7 docker Mac

* ```sql
  -- 建表
  create table t2
  (
      id      int         not null primary key,
      content varchar(50) null,
      type    int         null
  );
  -- 初始化数据
  insert into t2 (id, content,type) values (1,'内容1',1);
  insert into t2 (id, content,type) values (2,'内容2',2);
  insert into t2 (id, content,type) values (4,'内容4',4);
  insert into t2 (id, content,type) values (6,'内容6',6);
  ```

# 理论铺垫

## 事务定义

## 事务的并发问题

1）脏读

一个事务读到了另一个未提交事务修改过的数据

2）不可重复读

一个事务只能读到另一个已经提交的事务修改过的数据，并且其他事务每对该数据进行一次修改并提交后，该事务都能查询得到最新值

3）幻读

一个事务先根据某些条件查询出一些记录，之后另一个事务又向表中插入了符合这些条件的记录，原先的事务再次按照该条件查询时，能把另一个事务插入的记录也读出来

## 事务的隔离级别

## 事务的创建/提交/回滚

## 事务的查看

执行命令

```sql
-- 需要用户具有process权限
SELECT * FROM information_schema.INNODB_TRX
```

执行后会出现如下的结果

![image-20210401171514608](https://gitee.com/1162492411/pic/raw/master/数据库-MySQL-事务-事务列表示例.png)

对于结果中各项属性的作用解释如下

```
trx_id：唯一事务id号，只读事务和非锁事务是不会创建id的。
TRX_WEIGHT：事务的高度，代表修改的行数（不一定准确）和被事务锁住的行数。为了解决死锁，innodb会选择一个高度最小的事务来当做牺牲品进行回滚。已经被更改的非交易型表的事务权重比其他事务高，即使改变的行和锁住的行比其他事务低。
TRX_STATE：事务的执行状态，值一般分为：RUNNING, LOCK WAIT, ROLLING BACK, and COMMITTING.
TRX_STARTED：事务的开始时间
TRX_REQUESTED_LOCK_ID:如果trx_state是lockwait,显示事务当前等待锁的id，不是则为空。想要获取锁的信息，根据该lock_id，以innodb_locks表中lock_id列匹配条件进行查询，获取相关信息。
TRX_WAIT_STARTED：如果trx_state是lockwait,该值代表事务开始等待锁的时间；否则为空。
TRX_MYSQL_THREAD_ID：mysql线程id。想要获取该线程的信息，根据该thread_id，以INFORMATION_SCHEMA.PROCESSLIST表的id列为匹配条件进行查询。
TRX_QUERY：事务正在执行的sql语句。
TRX_OPERATION_STATE：事务当前的操作状态，没有则为空。
TRX_TABLES_IN_USE：事务在处理当前sql语句使用innodb引擎表的数量。
TRX_TABLES_LOCKED：当前sql语句有行锁的innodb表的数量。（因为只是行锁，不是表锁，表仍然可以被多个事务读和写）
TRX_LOCK_STRUCTS：事务保留锁的数量。
TRX_LOCK_MEMORY_BYTES：在内存中事务索结构占得空间大小。
TRX_ROWS_LOCKED：事务行锁最准确的数量。这个值可能包括对于事务在物理上存在，实际不可见的删除标记的行。
TRX_ROWS_MODIFIED：事务修改和插入的行数
TRX_CONCURRENCY_TICKETS：该值代表当前事务在被清掉之前可以多少工作，由 innodb_concurrency_tickets系统变量值指定。
TRX_ISOLATION_LEVEL：事务隔离等级。
TRX_UNIQUE_CHECKS：当前事务唯一性检查启用还是禁用。当批量数据导入时，这个参数是关闭的。
TRX_FOREIGN_KEY_CHECKS：当前事务的外键坚持是启用还是禁用。当批量数据导入时，这个参数是关闭的。
TRX_LAST_FOREIGN_KEY_ERROR：最新一个外键错误信息，没有则为空。
TRX_ADAPTIVE_HASH_LATCHED：自适应哈希索引是否被当前事务阻塞。当自适应哈希索引查找系统分区，一个单独的事务不会阻塞全部的自适应hash索引。自适应hash索引分区通过 innodb_adaptive_hash_index_parts参数控制，默认值为8。
TRX_ADAPTIVE_HASH_TIMEOUT：是否为了自适应hash索引立即放弃查询锁，或者通过调用mysql函数保留它。当没有自适应hash索引冲突，该值为0并且语句保持锁直到结束。在冲突过程中，该值被计数为0，每句查询完之后立即释放门闩。当自适应hash索引查询系统被分区（由 innodb_adaptive_hash_index_parts参数控制），值保持为0。
TRX_IS_READ_ONLY：值为1表示事务是read only。
TRX_AUTOCOMMIT_NON_LOCKING：值为1表示事务是一个select语句，该语句没有使用for update或者shared mode锁，并且执行开启了autocommit，因此事务只包含一个语句。当TRX_AUTOCOMMIT_NON_LOCKING和TRX_IS_READ_ONLY同时为1，innodb通过降低事务开销和改变表数据库来优化事务。
```

也可以直接参考[官方链接](https://dev.mysql.com/doc/refman/5.7/en/information-schema-innodb-trx-table.html)

## 快照读和当前读



## 隔离级别能解决哪些并发问题

| 隔离级别     | 脏读 | 不可重复读 | 幻影读 |
| ------------ | ---- | ---------- | ------ |
| RU           | √    | √          | √      |
| RC           | ×    | √          | √      |
| RR           | ×    | ×          | √      |
| SERIALIZABLE | ×    | ×          | ×      |

# 并发问题验证

## RU实验

### 实验一 ： 验证RU下的脏读问题

* 事务A ：Autocommit = 0，Isolation = RU
* 事务B ：Autocommit = 0，Isolation = RU

| 时间 | 事务A                         | 事务A结果      | 事务B                               | 事务B结果 | 备注 |
| ---- | ----------------------------- | -------------- | ----------------------------------- | --------- | ---- |
| T1   |                               |                | begin                               |           |      |
| T2   |                               |                | update t2 set type=2222 where id=1; |           |      |
| T3   | begin                         |                |                                     |           |      |
| T4   | select * from t2 where id=1 ; | type的值为2222 |                                     |           |      |

分析： 

* T2时刻，事务B修改了数据但没有提交事务
* T4时刻，事务A查询到了事务B未提交的修改，发生了脏读

### 实验二 ： 验证RU下的不可重复读问题

* 事务A ：Autocommit = 0，Isolation = RU
* 事务B ：Autocommit = 0，Isolation = RU

| 时间 | 事务A                         | 事务A结果      | 事务B                               | 事务B结果 | 备注 |
| ---- | ----------------------------- | -------------- | ----------------------------------- | --------- | ---- |
| T1   | begin                         |                |                                     |           |      |
| T2   | select * from t2 where id=1 ; | type的值为1    |                                     |           |      |
| T3   |                               |                | begin                               |           |      |
| T4   |                               |                | update t2 set type=2222 where id=1; |           |      |
| T5   | select * from t2 where id=1 ; | type的值为2222 |                                     |           |      |

分析： 

* T1时刻，事务A正常读取到type的值为1
* T4时刻，事务B将type的值修改为2222但并没有提交
* T5时刻，事务A读取type的值为2222，与T1时刻相比，同一个事务中两次读取的值不同

### 实验三 ： 验证RU下的幻读问题

* 事务A ：Autocommit = 0，Isolation = RU
* 事务B ：Autocommit = 0，Isolation = RU

| 时间 | 事务A                                        | 事务A结果                    | 事务B                                                  | 事务B结果 | 备注 |
| ---- | -------------------------------------------- | ---------------------------- | ------------------------------------------------------ | --------- | ---- |
| T1   | begin                                        |                              |                                                        |           |      |
| T2   | select * from t2 where type between 1 and 3; | 数据1<br />数据2             |                                                        |           |      |
| T3   |                                              |                              | begin                                                  |           |      |
| T4   |                                              |                              | insert into t2 (id, content, type) value (11,'ccc',2); |           |      |
| T5   | select * from t2 where type between 1 and 3; | 数据1<br />数据2<br />数据11 |                                                        |           |      |
| T6   |                                              |                              | rollback                                               |           |      |
| T7   | select * from t2 where type between 1 and 3; | 数据1<br />数据2             |                                                        |           |      |

分析： 

* T4时刻事务B插入了数据11但是并没有提交事务
* T5时刻事务A查询到了数据11
* T6时刻事务B回滚了事务
* T7时刻事务A又查不到数据11，与T5时刻相比，数据11消失不见了
* (也可以拿T1～T5时刻进行比较，T5时刻的结果相比T2时刻竟然多了数据11这条记录)

## RC实验

### 实验一 ： 验证RC下的脏读问题（不存在）

* 事务A ：Autocommit = 0，Isolation = RC
* 事务B ：Autocommit = 0，Isolation = RC

| 时间 | 事务A                         | 事务A结果   | 事务B                               | 事务B结果 | 备注 |
| ---- | ----------------------------- | ----------- | ----------------------------------- | --------- | ---- |
| T1   |                               |             | begin                               |           |      |
| T2   |                               |             | update t2 set type=2222 where id=1; |           |      |
| T3   | begin                         |             |                                     |           |      |
| T4   | select * from t2 where id=1 ; | type的值为1 |                                     |           |      |

分析： 

* T2时刻，事务B修改了数据但没有提交事务
* T4时刻，事务A读取数据，type为1，并没有受到事务B修改为2222的影响

### 实验二 ：验证RC下的不可重复读问题

* 事务A ：Autocommit = 0，Isolation = RC
* 事务B ：Autocommit = 0，Isolation = RC

| 时间 | 事务A                         | 事务A结果      | 事务B                               | 事务B结果 | 备注 |
| ---- | ----------------------------- | -------------- | ----------------------------------- | --------- | ---- |
| T1   | begin                         |                |                                     |           |      |
| T2   | select * from t2 where id=1 ; | type的值为1    |                                     |           |      |
| T3   |                               |                | begin                               |           |      |
| T4   |                               |                | update t2 set type=2222 where id=1; |           |      |
| T5   |                               |                | commit                              |           |      |
| T6   | select * from t2 where id=1 ; | type的值为2222 |                                     |           |      |

分析： 

* T1时刻，事务A正常读取到type的值为1
* T4时刻，事务B将type的值修改为2222并在T5时刻提交事务
* T6时刻，事务A读取type的值为2222，与T1时刻相比，同一个事务中两次读取的值不同

### 实验三 ： 验证RC下的幻读问题

* 事务A ：Autocommit = 0，Isolation = RC
* 事务B ：Autocommit = 0，Isolation = RC

| 时间 | 事务A                                        | 事务A结果                    | 事务B                                                  | 事务B结果 | 备注 |
| ---- | -------------------------------------------- | ---------------------------- | ------------------------------------------------------ | --------- | ---- |
| T1   | begin                                        |                              |                                                        |           |      |
| T2   | select * from t2 where type between 1 and 3; | 数据1<br />数据2             |                                                        |           |      |
| T3   |                                              |                              | begin                                                  |           |      |
| T4   |                                              |                              | insert into t2 (id, content, type) value (11,'ccc',2); |           |      |
| T5   |                                              |                              | commit                                                 |           |      |
| T6   | select * from t2 where type between 1 and 3; | 数据1<br />数据2<br />数据11 |                                                        |           |      |

分析 ：

* T2时刻事务A可以看到两条数据
* T3～T5时刻事务B插入了数据11并提交事务
* T6时刻，事务A看到了三条数据，相比T2时刻多了一条数据

## RR实验

### 实验一 ：验证RR下的脏读问题(不存在)

* 事务A ：Autocommit = 0，Isolation = RR
* 事务B ：Autocommit = 0，Isolation = RR

| 时间 | 事务A                         | 事务A结果   | 事务B                               | 事务B结果 | 备注 |
| ---- | ----------------------------- | ----------- | ----------------------------------- | --------- | ---- |
| T1   |                               |             | begin                               |           |      |
| T2   |                               |             | update t2 set type=2222 where id=1; |           |      |
| T3   | begin                         |             |                                     |           |      |
| T4   | select * from t2 where id=1 ; | type的值为1 |                                     |           |      |

分析： 

* T2时刻，事务B修改了数据但没有提交事务
* T4时刻，事务A读取数据，type为1，并没有受到事务B修改为2222的影响

### 实验二 ：验证RR下的不可重复读问题(不存在)

* 事务A ：Autocommit = 0，Isolation = RR
* 事务B ：Autocommit = 0，Isolation = RR

| 时间 | 事务A                         | 事务A结果   | 事务B                               | 事务B结果 | 备注 |
| ---- | ----------------------------- | ----------- | ----------------------------------- | --------- | ---- |
| T1   | begin                         |             |                                     |           |      |
| T2   | select * from t2 where id=1 ; | type的值为1 |                                     |           |      |
| T3   |                               |             | begin                               |           |      |
| T4   |                               |             | update t2 set type=2222 where id=1; |           |      |
| T5   |                               |             | commit                              |           |      |
| T6   | select * from t2 where id=1 ; | type的值为1 |                                     |           |      |

分析： 

* T1时刻，事务A正常读取到type的值为1
* T4时刻，事务B将type的值修改为2222并在T5时刻提交事务
* T6时刻，事务A读取type的值为1，与T1时刻相比，同一个事务中两次读取的值相同

### 实验三 ：验证RR下的幻读问题

* 事务A ：Autocommit = 0，Isolation = RR
* 事务B ：Autocommit = 0，Isolation = RR

| 时间 | 事务A                                                   | 事务A结果                          | 事务B                                                        | 事务B结果 | 备注              |
| ---- | ------------------------------------------------------- | ---------------------------------- | ------------------------------------------------------------ | --------- | ----------------- |
| T1   | begin                                                   |                                    |                                                              |           |                   |
| T2   | select * from t2 where type between 1 and 3;            | 数据1<br />数据2                   |                                                              |           |                   |
| T3   |                                                         |                                    | begin                                                        |           |                   |
| T4   |                                                         |                                    | SQL1 : insert into t2 (id, content, type) value (11,'ccc',2);<br />SQL2 : update t2 set type=2 where id=4; |           | 执行SQL1/SQL2均可 |
| T5   |                                                         |                                    | commit                                                       |           |                   |
| T6   | select * from t2 where type between 1 and 3 for update; | 数据1<br />数据2<br />数据11/数据4 |                                                              |           |                   |

分析 ：

* T2时刻事务A可以看到两条数据
* T3～T5时刻事务B插入了数据11/修改了数据4并提交事务
* T6时刻，事务A看到了三条数据，相比T2时刻多了一条数据

## 序列化实验

### Todo实验一 ：验证序列化下的脏读问题(不存在)

### todo实验二 ：验证序列化下的不可重复读问题(不存在)

### todo实验三 ：验证序列化下的幻读问题(不存在)





# 实验

## 实验一 ：验证当前读和快照读

* 事务A ：Autocommit = 0，Isolation = RR
* 事务B ：Autocommit = 0，Isolation = RR

| 时间 | 事务A                                                    | 事务A结果               | 事务B                                                   | 事务B结果 | 备注 |
| ---- | -------------------------------------------------------- | ----------------------- | ------------------------------------------------------- | --------- | ---- |
| T1   | Begin                                                    |                         |                                                         |           |      |
| T2   | select * from t2 where type between 5 and 10;            | 6,文字6,6               |                                                         |           |      |
| T3   |                                                          |                         | begin                                                   |           |      |
| T4   |                                                          |                         | insert into t2 (id, content,type) values (7,'内容7',7); |           |      |
| T5   | select * from t2 where type between 5 and 10;            | 6,文字6,6               |                                                         |           |      |
| T6   |                                                          |                         | commit                                                  |           |      |
| T7   | select * from t2 where type between 5 and 10;            | 6,文字6,6               |                                                         |           |      |
| T8   | select * from t2 where type between 5 and 10 for update; | 6,文字6,6<br/>7,内容7,7 |                                                         |           |      |
| T9   | commit                                                   |                         |                                                         |           |      |

分析： 

* T5时刻执行了查询语句，但是事务A并没有看到数据7，因为事务B此时没有提交事务
* T7时刻执行了查询语句，事务B已经提交事务，事务A仍然没有看到数据7，在T8时刻才能看到数据7，因为select是快照读，而select for update是当前读

## 实验二 ： select for update排它锁的验证

* 事务A ：Autocommit = 0，Isolation = RR
* 事务B ：Autocommit = 0，Isolation = RR

| 时间 | 事务A                                                        | 事务A结果                                                    | 事务B                                                        | 事务B结果 | 备注 |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | --------- | ---- |
| T1   | Begin                                                        |                                                              |                                                              |           |      |
| T2   | select * from t2 where type between 5 and 10;                | 数据6                                                        |                                                              |           |      |
| T3   |                                                              |                                                              | begin                                                        |           |      |
| T4   |                                                              |                                                              | insert into t2 (id, content,type) values (7,'内容7',7);      |           |      |
| T5   | SQL1 ： select * from t2 where type between 5 and 10 for update ;<br />SQL2 ：select * from t2 where type = 2 for update ;<br />SQL3 ：select * from t2 where type between 5 and 10;<br />SQL4 ：select * from t2 where type = 2 | 若执行SQL1会阻塞直到超时；若type字段无索引，执行SQL2会阻塞，若type字段又索引，执行SQL会查询出数据2；若执行SQL3会查询出数据6;若执行SQL4会查询出数据2 | SQL5 : select * from t2 where type = 2 for update ;<br />SQL6 : select * from t2 where id = 2 for update ; |           |      |
| T6   |                                                              |                                                              | Commit                                                       |           |      |
| T7   | select * from t2 where type between 5 and 10  ;              | 数据6                                                        |                                                              |           |      |
| T8   | select * from t2 where type between 5 and 10  for update;    | 数据6<br />数据7                                             |                                                              |           |      |
| T9   | commit                                                       |                                                              |                                                              |           |      |

分析： 

* T5时刻，四条SQL中，若执行SQL1会阻塞，因为此刻事务B正在插入数据，若执行SQL3/SQL4不会阻塞，因为这两条SQL是当前读；若type字段无索引时，执行SQL1/SQL2时会锁表，无法修改表的数据但可以快照读，若type字段有索引并且命中索引时，执行SQL1/SQL2时仅会锁住范围内的数据(关于行锁和表锁以及与索引的实际情况比较复杂)
* T7时刻，执行SQL无法查询出数据7，因为select是当前读，事务A开始时就看不到数据7，此刻也应该看不到数据7
* T8时刻，执行SQL可以查询出数据7，因为此刻没有其他事务在修改/插入数据，所以不会阻塞，因为select for update是当前读，因此可以看到数据7

## 实验三 ： 验证排它锁的行锁和表锁(todo)



## 实验四 ：验证两个写事务

* 事务A ：Autocommit = 0，Isolation = RR
* 事务B ：Autocommit = 0，Isolation = RR

| 时间 | 事务A                                                   | 事务A结果 | 事务B                                                        | 事务B结果 | 备注 |
| ---- | ------------------------------------------------------- | --------- | ------------------------------------------------------------ | --------- | ---- |
| T1   | Begin                                                   |           |                                                              |           |      |
| T2   | insert into t2 (id, content,type) values (9,'内容9',9); |           |                                                              |           |      |
| T3   |                                                         |           | begin                                                        |           |      |
| T4   |                                                         |           | SQL1 : insert into t2 (id, content,type) values (7,'内容7',7);<br />SQL2 : insert into t2 (id, content,type) values (9,'内容7',9); |           |      |
| T5   | commit                                                  |           |                                                              |           |      |
| T6   |                                                         |           | commit                                                       |           |      |

分析： 

* T4时刻，执行SQL1，可以正常执行，并且T5、T6可以正常提交
* T4时刻，执行SQL2，事务B阻塞等待，T5时刻事务A提交后，事务B会报错主键冲突

## 实验五 ：select 常量 for update

例如 select 'xx' for update，在这种场景中，即便手动begin事务，仍然不会在mysql的事务列表中观察到事务

## 需要继续研究 ： 意向锁、意向锁之间的兼容情况、select for update和select in share mode之间的异同点