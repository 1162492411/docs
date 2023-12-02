---
title: "MySQL死锁和锁等待的几种常见场景"
date: 2023-12-02T21:25:53+08:00
draft: false
toc: true
categories:
  - MySQL
tags:
  - MySQL
  - 数据库
  - MYSQL锁锁
comment: false
---

  本次分享偏重场景罗列，锁原因分析会薄弱一些，对基础知识会走马观花式的一带而过，仅做查漏补缺。
# 事务的知识点

  作为后端开发，使用不同数据库的时候都会接触到事务的概念。MySQL也不例外，它的Innodb引擎也支持事务。
  在连接MySQL的时候，会出现一台MySQL同时处理多个事务的情况，事务并发了，就可能出现脏读（dirty read）、不可重复读（non-repeatable read）、幻读（phantom read）的问题。
  为了解决这些问题，Innodb实现了不同的隔离级别。

| 隔离级别                         | 定义                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| 读未提交（read uncommitted，RU） | 一个事务还未提交时，它做的变更就可以被别的事务看到。         |
| 读提交（read committed，RC）     | 事务提交以后，它做的变更才能被其它事务看到。但是在这个事务未提交之前，数据库中发生的变更，这个事务也能看见。 |
| 可重复读（repeatable read，RR）  | 事务总是只能看见在启动的那个时刻，数据库的状态。事务未提交之前做的变更，其它事务看不见。事务执行期间，数据库中已经发生的变更，这个事务也看不见。只能看见事务刚启动时刻，数据库的状态。 |
| 串行化（serializable）           | 事务对某一行的操作会加锁，“写”会加“写锁”，“读”会加“读锁”，在锁释放掉之前，其它的事务都无法都这一行的记录进行操作。必须等之前的事务执行完毕，释放锁。后面的事务又会重新加锁。 |

  不同的隔离级别能够解决不同的问题
|          | 脏读 | 不可重复读 | 幻读 |
| -------- | ---- | ---------- | ---- |
| 读未提交 | 存在 | 存在       | 存在 |
| 读已提交 |      | 存在       | 存在 |
| 可重复读 |      |            | 存在 |
| 串行化   |      |            |      |

  这四种隔离级别具体是如何实现的呢？(该段来自MySQL45讲)
| 隔离级别   | 实现方式                                                     |
| ---------- | ------------------------------------------------------------ |
| 读未提交   | 因为可以读到未提交事务修改的数据，所以直接读取最新的数据     |
| 读提交RC   | 通过 Read View （MVCC多版本控制），在「每个语句执行前」都会重新生成一个 Read View |
| 可重复读RR | 通过 Read View （MVCC多版本控制），「启动事务时」生成一个 Read View， |
| 串行化     | 通过加读写锁的方式来避免并行访问                             |

  至于ReadView的实现方式，本次不详细展开，感兴趣的自行搜索，大致是
- 在数据中追加trx_id+undo log，

- 保存一个全局的事务id列表，

- 通过比较trx_id的大小来决策出某个事务能看到哪个版本的数据

- 其实可以看成轻量锁，用低代价解决并发时哪个版本的数据可见的问题

  # 都有哪些锁

  ## 锁的种类
  参考[官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)中关于锁的章节与[小林condig中的锁介绍](https://www.xiaolincoding.com/mysql/lock/mysql_lock.html#%E8%A1%8C%E7%BA%A7%E9%94%81)，简述如下
  InnoDB 实现了标准的行级锁，包括两种：共享锁（简称 s 锁）、排它锁（简称 x 锁）。

- 共享锁（S锁）：允许持锁事务读取一行。

- 排他锁（X锁）：允许持锁事务更新或者删除一行。
  如果事务 T1 持有行 r 的 s 锁，那么另一个事务 T2 请求 r 的锁时，会做如下处理：

- T2 请求 s 锁立即被允许，结果 T1 T2 都持有 r 行的 s 锁

- T2 请求 x 锁不能被立即允许
如果 T1 持有 r 的 x 锁，那么 T2 请求 r 的 x、s 锁都不能被立即允许，T2 必须等待T1释放 x 锁才可以，因为X锁与任何的锁都不兼容。

| 类别                  | 名称                 | 介绍                                                         | 数据库日志示例                                               |
| --------------------- | -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 表级锁                | 表锁                 |                                                              |                                                              |
|                       | 元数据锁             |                                                              |                                                              |
|                       | 意向锁               | MySQL对数据加X锁或S锁之前，先加一个意向X锁或意向S锁，获取意向锁成功则可以对数据加X锁或S锁， 它的目的是为了快速判断表里是否有记录被加锁 | TABLE LOCK table lc_3.a trx id 133588125  lock mode IX       |
|                       | 自增锁               | 表级别的锁，单条SQL执行完就释放 源码实现方面，也有Latch参与该锁 | TABLE LOCK table xx trx id 7498948  lock mode AUTO-INC waiting |
| 行级锁 加锁对象为索引 | 行锁 Record Lock     | 锁住的是一条记录，分为 S 锁和 X 锁 行锁的S和X兼容性：只有S和S兼容，其他冲突 | RECORD LOCKS space id 281 page no 3 n bits 72 index PRIMARY of table lc_3.a trx id 133588125  lock_mode X locks rec but not gap |
|                       | 间隙锁 Gap Lock      | 锁住一个范围,两侧开区间，RR和串行会有 两个间隙锁互相兼容，可存在多个事务锁定同样的范围 | RECORD LOCKS space id 281 page no 5 n bits 72 index idx_c of table lc_3.a trx id 133588125  lock_mode X locks gap before rec |
|                       | 临键锁 Next-Key Lock | 锁住一个范围+一条记录，一侧开区间一侧闭区间，RR和串行会有    | RECORD LOCKS space id 281 page no 5 n bits 72 index idx_c of table lc_3.a trx id 133588125  lock_mode X |

## 锁的兼容性矩阵
  横向是已持有锁，纵向是正在请求的锁

[图片]

[图片]

## 加锁规则
[官方文档-不同SQL的加锁-5.7](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)

| 语句                                                 | 加锁规则                                                     |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| select                                               | 使用快照读                                                   |
| select.. for update 和 select ... lock in share mode | next key lock(视情况会退化)                                  |
| Update where Delete where                            | next key lock(视情况会退化)                                  |
| insert                                               | INSERT 在插入的行上设置排他锁。 该锁是索引记录锁，而不是下一个键锁（即没有间隙锁），并且不会阻止其他会话插入到插入行之前的间隙中。 在插入行之前，会设置一种称为插入意向间隙锁的间隙锁。 此锁表明插入的意图是，插入同一索引间隙的多个事务如果没有插入间隙内的同一位置，则无需互相等待。 假设存在值为 4 和 7 的索引记录。尝试插入值 5 和 6 的单独事务在获得插入行上的排他锁之前，每个事务都使用插入意向锁锁定 4 和 7 之间的间隙，但不这样做 相互阻塞，因为行不冲突。 如果发生重复键错误，则会在重复索引记录上设置共享锁。 如果另一个会话已经拥有排它锁，则如果多个会话尝试插入同一行，则使用共享锁可能会导致死锁。 如果另一个会话删除该行，则可能会发生这种情况 |
| replace                                              |                                                              |

  这里提出了“快照读”和“当前读”的概念(其实还有半一致性读，后文我们通过实际例子来感受它)。
## Next Key Lock加锁规则
  不同版本会有细微差异，且情况比较复杂。
  加锁规则总结为以下几点，不同MySQL版本会有微小的差异
- 查询过程中只要访问的数据都会加锁，加锁的基本单位是next-key lock，左开右闭
- 唯一索引等值查询，next-key lock退化为行锁
- 索引等值查询，需要访问到第一个不满足条件的值，此时的next-key lock会退化为间隙锁
- 索引范围查询需要访问到不满足条件的第一个值为止，如果where条件部分命中(>、<、like等)或者全未命中，则会加附近Gap间隙锁。例如，某表数据如下，非唯一索引2,6,9,9,11,15。如下语句要操作非唯一索引列9的数据，gap锁将会锁定的列是(6,11]，该区间内无法插入数据。
[图片]
[小林coding](https://www.xiaolincoding.com/mysql/lock/how_to_lock.html#%E6%80%BB%E7%BB%93)
[咔咔-8.0.26](https://mdnice.com/writing/4b821fa327f84fb3ae6b98a386674d3e#writing-title)
  下表没补充完，细分情况挺多的

| 方式-RR级别      | 值是否存在 | 加锁                                                         | 示例                                                         |
| ---------------- | ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 没索引           |            | 对每一条记录的索引上都会加 next-key 锁                       | sql：where age=2 b表内数据：1，2，5，9 锁: (-无穷,1]、(1,2]、(2,5]、(5,9] |
| 唯一索引等值查询 | 存在       | 在索引树上定位到这一条记录后，加Record Lok                   | Sql: where age=1 表内数据：1，2，3 锁定：age=1 record        |
|                  | 不存在     | 找到最后一条小于该值的记录，找到第一条大于该值的记录，加GAP Lock | sql:where age=24 表内数据：20，26，28 锁定：gap (20,26)      |
| 唯一索引范围查询 |            |                                                              |                                                              |
| 普通索引等值查询 |            |                                                              |                                                              |
| 普通索引范围查询 |            |                                                              |                                                              |

# 死锁

  死锁是指两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源（我等待你的资源，你却等待我的资源，我们都相互等待，谁也不释放自己占有的资源），从而导致恶性循环的现象：

- 当多个事务试图以不同顺序锁定资源时，就可能会产生死锁

- 多个事务，同时锁定同一个资源时，也会产生死锁
## 定位死锁

  步骤 ： 报错日志、抓取日志、找到代码入口、梳理sql、本地复现、加锁分析、结合业务提出规避方案(加分布式锁、改sql、调整sql顺序、大事务拆成小事务)
## 本地复现

  由于死锁日志中，仅仅是部分信息，仅有冲突时候的SQL和部分锁的情况，因此还需要本地复现来详细分析死锁原因。本地可以通过IDEA中的DataBase插件来复现，连接数据库，新建几个Console，手动调整隔离级别与事务提交方式
  [图片]
  [图片]
  当然也可以采用命令行方式来修改隔离级别和提交方式

  ```sql
  -- 查看隔离级别
  select @@global.tx_isolation,@@tx_isolation;
  -- 修改隔离级别
  -- 全局隔离级别
  set global transaction isolation level read committed;
  -- 当前会话隔离级别
  set session transaction isolation level read committed;
  ```

  执行SQL时，可以查询锁的情况

  ```sql
  -- 5.7 查看锁的情况
  select * from information_schema.innodb_locks;
  -- 5.7 查看等待锁
  select * from information_schema.innodb_lock_waits;
  -- 8.0 查看锁的情况
  select * from performance_schema.data_locks;
  -- 8.0 查看等待锁
  select * from performance_schema.data_lock_waits;
  -- 查看最终的死锁日志
  show engine innodb status;
  ```

  

  ## 锁情况说明

   查看MySQL锁信息(基于5.7,执行select * from information_schema.innodb_locks;,如果是8.0得到的列会稍多一些)，得到如下信息
  [图片]
  (解释：
  INNODB_LOCKS：提供有关InnoDB 事务已请求但尚未获取的每个锁的信息，以及事务持有的阻止另一个事务的每个锁。)
  详见https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-table.html
  
  列名     描述
  LOCK_ID     一个唯一的锁ID号，内部为 InnoDB。
  LOCK_TRX_ID     持有锁的交易的ID
  LOCK_MODE     如何请求锁定。允许锁定模式描述符 S，X， IS，IX， GAP，AUTO_INC，和 UNKNOWN。锁定模式描述符可以组合使用以识别特定的锁定模式。
  LOCK_TYPE     锁的类型
  LOCK_TABLE     已锁定或包含锁定记录的表的名称
  LOCK_INDEX     索引的名称，如果LOCK_TYPE是 RECORD; 否则NULL
  LOCK_SPACE     锁定记录的表空间ID，如果 LOCK_TYPE是RECORD; 否则NULL
  LOCK_PAGE     锁定记录的页码，如果 LOCK_TYPE是RECORD; 否则NULL。
  LOCK_REC     页面内锁定记录的堆号，如果 LOCK_TYPE是RECORD; 否则NULL。
  LOCK_DATA     与锁相关的数据（如果有）。如果 LOCK_TYPE是RECORD，是锁定的记录的主键值，否则NULL。此列包含锁定行中主键列的值，格式为有效的SQL字符串。如果没有主键，LOCK_DATA则是唯一的InnoDB内部行ID号。如果对键值或范围高于索引中的最大值的间隙锁定，则LOCK_DATA 报告supremum pseudo-record。当包含锁定记录的页面不在缓冲池中时（如果在保持锁定时将其分页到磁盘），InnoDB不从磁盘获取页面，以避免不必要的磁盘操作。相反， LOCK_DATA设置为 NULL。

  ## 死锁日志说明

```
2023-10-10 18:22:21 0x7fcb000ec700 INNODB MONITOR OUTPUT
=====================================

Per second averages calculated from the last 19 seconds
-----------------

BACKGROUND THREAD
-----------------

srv_master_thread loops: 31220617 srv_active, 0 srv_shutdown, 152 srv_idle

srv_master_thread log flush and writes: 31220764
----------

SEMAPHORES
----------

OS WAIT ARRAY INFO: reservation count 1506882018
OS WAIT ARRAY INFO: signal count 1446234789
RW-shared spins 0, rounds 2294332596, OS waits 697533435
RW-excl spins 0, rounds 10291686095, OS waits 105277217
RW-sx spins 356384667, rounds 7427459143, OS waits 87734236

Spin rounds per wait: 2294332596.00 RW-shared, 10291686095.00 RW-excl, 20.84 RW-sx
------------------------

LATEST DETECTED DEADLOCK
------------------------

-- 死锁发生的时间
2023-10-10 18:22:15 0x7fd082515700
-- 第一个事务
*** (1) TRANSACTION:
-- 事务id
TRANSACTION 14820253579, ACTIVE 11 sec fetching rows
mysql tables in use 1, locked 1
-- N row lock(s)表示事务持有了多少锁，undo log entries N 表示该事务有多少条undo log
LOCK WAIT 44 lock struct(s), heap size 8400, 4009 row lock(s), undo log entries 2
-- thread id 表示事务所在的thread, 每个连接会对应一个thread_id, 与performance_schema.processlist中的id一致
MySQL thread id 409677283, OS thread handle 140540293539584, query id 31864329702 10.181.61.7 user_dev updating
/* ApplicationName=IntelliJ IDEA 2022.2.4 */ 
-- 死锁时事务内的最后一条sql
update bbb         set left_value = left_value + 2         where field_template_id = 11502           and left_value >= 1           and is_delete = 0
-- 事务发生阻塞时的语句
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
-- 等待的锁的原因，在哪张表的哪个索引上，持有/等待什么样的锁
RECORD LOCKS space id 57560 page no 73 n bits 176 index PRIMARY of table 
`aaa`.`bbb` trx id 14820253579 lock_mode X waiting Record lock,
 heap no 108 PHYSICAL RECORD: n_fields 15; compact format; info bits 0
-- 每行分三部分，n: len m记录长度信息；hex xxxxxxx以16进制记录数据，当锁所在字段类型为数值类型时，
-- 忽略掉最高位，再进行换算，比如"hex 8000001e", 去掉最高位"8", 换算成10进制得到"30"。
-- 最后一部分asc 当锁所在字段类型为字符串类型，才会展示字符串值， 或者next-key lock/间隙锁是最右区间，
-- 会展示"asc supremum"。
 0: len 4; hex 00003aaa; asc   : ;;
 1: len 6; hex 0003735b2589; asc   s[% ;;
...
-- 第二个事务
*** (2) TRANSACTION:
TRANSACTION 14820255113, ACTIVE 8 sec starting index read
mysql tables in use 1, locked 1
3 lock struct(s), heap size 1136, 2 row lock(s), undo log entries 1
MySQL thread id 409677311, OS thread handle 140533516293888, query id 31864337001 10.181.61.7 user_dev updating
/* ApplicationName=IntelliJ IDEA 2022.2.4 */ update bbb         set left_value = left_value + 2         where field_template_id = 11502           and left_value >= 1           and is_delete = 0
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 57560 page no 73 n bits 176 index PRIMARY of table `aaa`.`bbb` trx id 14820255113 lock_mode X locks rec but not gap
Record lock, heap no 108 PHYSICAL RECORD: n_fields 15; compact format; info bits 0
 0: len 4; hex 00003aaa; asc   : ;;
 1: len 6; hex 0003735b2589; asc   s[% ;;
 2: len 7; hex e10000803b0110; asc     ;  ;;
 ...
 *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 57560 page no 5 n bits 216 index PRIMARY of table `aaa`.`bbb` trx id 14820255113 lock_mode X waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 15; compact format; info bits 0
 0: len 4; hex 00000001; asc     ;;
 1: len 6; hex 00034789f60a; asc   G   ;;
 ...
-- 回滚了哪个事务

*** WE ROLL BACK TRANSACTION (2)
------------

TRANSACTIONS
------------

...
```


## 预防规避死锁

- DBA：设置超时时间：默认60s，我司缩短为30s
- DBA：开启死锁检测自动回滚(具体判断哪个事务回滚，可配置回滚策略，是根据事务创建时间还是影响行数，不再展开)
- 开发：保持良好的开发习惯
  - 同顺序：以固定的顺序访问表和行。比如两个更新数据的事务，事务A 更新数据的顺序 为1->2；事务B更新数据的顺序为2->1。这样更可能会造成死锁
  
  - 尽量保持事务简短：大事务更倾向于死锁，如果业务允许，将大事务拆小
  
  - 一次性锁定：在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁概率

  - 降低隔离级别：如果业务允许，将隔离级别调低也是较好的选择，比如将隔离级别从RR调整为RC，可以避免掉很多因为gap锁造成的死锁。但是要充分考虑好各种情况

  - 细粒度锁定（行锁）：为表添加合理的索引。可以看到如果不走索引将会为表的每一行记录添加上锁，死锁的概率大大增大
# 几种典型场景
## 默认的隔离级别
  MySQL默认RR，但是为了提高并发，多改为RC
  
## 顺序相反 - 先A后B与先B后A
  一个先a后b一个先b后a。任意一张表，隔离级别RC
| 时刻 | 事务A-RC                                 | 事务B-RC                                 |
| ---- | ---------------------------------------- | ---------------------------------------- |
| T1   | Begin                                    |                                          |
| T2   | update user set name = 'a-1' where id=1; | Begin                                    |
| T3   |                                          | update user set name = 'b-2' where id=2; |
| T4   | update user set name = 'a-2' where id=2; |                                          |
| T5   | 等待                                     |                                          |
| T6   |                                          | update user set name = 'b-1' where id=1; |
| T7   |                                          | 报错：DeadLock                           |
| T8   | commit                                   |                                          |

  这里就不再分析了，一眼可以看明白，归类于对同样两条数据的先a后b与先b后a。完美符合死锁的定义，且sql简单明了。
  看似不会这么写代码，针对同一张表先a后b与先b后a，但仍然有该情况的变种：两处代码，一处是先表A再表B，另外一处入口是先表B再表A
  举一个业务例子，商家购买某项权益商品，生成订单后支付，钱包回调支付成功时发放给改该商家该权益商品

  ```sql
  CREATE TABLE `order` (
  `id` int NOT NULL COMMENT '主键ID',
  `order_no` varchar(30) DEFAULT NULL COMMENT '订单号',
  `status` int DEFAULT NULL COMMENT '订单状态,待支付,待发货,已收货,售后完成,已关闭',
  `user_id` int DEFAULT NULL COMMENT '用户id',
  `privilege_item_id` int DEFAULT NULL COMMENT '权益商品id',
  ...
  PRIMARY KEY (`id`),
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 comment='订单表'
  CREATE TABLE `wallet_callback_record` (
  `id` int NOT NULL COMMENT '主键ID',
  `wallet_no` varchar(30) DEFAULT NULL COMMENT '钱包流水号',
  `order_no` int DEFAULT NULL COMMENT '订单号',
  `status` int DEFAULT NULL COMMENT '支付状态,待支付(未回调),支付成功,支付失败',
  PRIMARY KEY (`id`),
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 comment='钱包回调表'
  CREATE TABLE `merchant_privilege_record` (
  `id` int NOT NULL COMMENT '主键ID',
  `merchant_id` int DEFAULT NULL COMMENT '商家id',
  `privilege_item_id` int DEFAULT NULL COMMENT '权益商品id',
  `status` int DEFAULT NULL COMMENT '状态',
  `end_dt` timestamp null comment '权益截止期',
  PRIMARY KEY (`id`),
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 comment='商家权益表'
  ```

  


  入口A-下单：

- 插入订单表：insert into order 

- 插入回调记录：insert into wallet_callback_record 

- 插入商家权益表: insert into merchant_privilege_record

  入口B-钱包回调：

- 查询回调记录表：select * from wallet_callback_record where wallet_no='xxx'

- 更新回调记录：update wallet_callback_record where wallet_no='xxx'

- 更新订单表: update order where order_no='xxx'

- 更新商家权益表: update merchant_privilege_record  where merchant_id=111

  入口C-关闭订单：

- 更新商家权益表: update merchant_privilege_record  where merchant_id=111

- 更新订单表：update order set status='取消订单' where order_no='xxx'
  如果入口B和入口C同时发生，那么就会死锁

## 数据重叠 - 表没有辅助索引

  ```sql
  CREATE TABLE `user` (
  `id` int NOT NULL COMMENT '主键ID',
  `name` varchar(30) DEFAULT NULL COMMENT '姓名',
  `age` int DEFAULT NULL COMMENT '年龄',
  `is_delete` tinyint DEFAULT '0',
  PRIMARY KEY (`id`)
  --注意，没有索引哦
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
  -- 插入20条数据,其中id=7、8时is_delete=1，其他数据is_delete=0
  ```

  
| 时刻 | 事务A-RC或RR 例如这是一个刷数据的job                   | 事务B-RC或RR 用户主动改名                       |
| ---- | ------------------------------------------------------ | ----------------------------------------------- |
| T1   | Begin                                                  |                                                 |
| T2   | SQL1： update user set age= age +1  where is_delete=0; | Begin                                           |
| T3   |                                                        | SQL2: update user set name='P7-hh'  where id=7; |
| T4   |                                                        | 等待                                            |
| T5   | commit                                                 |                                                 |

分析
- SQL1通过主键索引锁定了全表每行数据

- SQL2通过主键索引想要锁定id=7的行，由于该锁已被SQL1持有因此等待
## 数据重叠 - 有辅助索引但没使用到
  上一个例子里，如果对sql1中的is_delete加一下索引能否解决问题呢？这里特意把隔离级别调成RR

  ```
  CREATE TABLE `user` (
  `id` int NOT NULL COMMENT '主键ID',
  `name` varchar(30) DEFAULT NULL COMMENT '姓名',
  `age` int DEFAULT NULL COMMENT '年龄',
  `is_delete` tinyint DEFAULT '0',
  PRIMARY KEY (`id`),
  KEY `idx_is_delete` (`is_delete`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
  ```

  如果觉得该场景中的sql不太贴合业务场景，把is_delete换成level商家等级,隔离级别RR，事务A刷数job针对level=1的新手商家统一赠送100积分,表中level=1的商家假设占据一半的比例，事务B为某个高等级商家修改自己的名字。

  分析-RR模式
- 针对SQL1， RR级别下，虽然where条件中的is_delete有索引，但是因为is_delete=0的行数在表中所占比例很大，因此sql最终使用了主键索引，锁住了全表所有数据(可以通过查看执行计划来佐证)
[图片]
- 针对SQL2，where条件中只有id，因此若SQL2执行，通过主键索引会锁定id=7的这条数据。
- RR级别下，事务B 的SQL2执行时，因为想要锁的索引和数据已经被事务A持有，因此会等待事务A释放锁，或者一直等待直至抛错Lock wait timeout exceeded; try restarting transaction。
  
  分析-RC模式
- 针对SQL1，RC级别下，虽然也没有用到is_delete索引，改用了主键索引，但是最终只锁定了符合is_delete=0的那部分数据
- 针对SQL2，与RR模式下的分析一样，若SQL2执行，通过主键索引会锁定id=7的这条数据
- RC级别下，SQL正常执行，它可以正常拿到id=7的这条数据的锁
- 引出疑问：why，rc和rr锁定的不一样，猜想应该是semi-consistent read(半一致性读)在rc级别生效了，把不符合条件的数据的锁给释放掉了(具体方式是先扫描到这些数据，发现不符合where条件后又释放对这些数据的锁定)。semi-consistent read会提前释放一些锁，减少冲突的概率提高并发，具体概念自行查询，这里给出官方的[参考链接](https://dev.mysql.com/doc/refman/5.7/en/innodb-transaction-isolation-levels.html)与中文博客链接）

有索引但是没有使用上的原因很多，日常工作中大家多少都有了解，这里不再举更多例子。
## 数据重叠 - 不同索引方式锁定的数据重叠
  针对上一个例子，如果让事务A中的sql使用上辅助索引，还会出问题吗？
  基于MySQL5.7.24 & 8.0.20
  先说结论：如果多个事务中，当前读的sql锁定的数据中有重叠，也会出现锁等待。

```sql
create table t_field_dict
(
    id                int(11) unsigned auto_increment comment '自动主键' primary key,
    field_template_id int                           not null comment '字段模版id',
    value_name        varchar(255)                  not null comment '枚举值名称',
    parent_id         int          default 0        not null comment '父id',
    left_value        int          default 0        not null comment '左序号',
    right_value       int          default 0        not null comment '右序号',
    is_delete         tinyint(3)   default 0        not null comment '是否删除 0:未删除 1:已删除',
    KEY `idx_p` (`field_id`),
) comment '枚举值表';
insert into a_dict(id,field_template_id,value) values
(1,2,100),(2,2,200),(3,2,300),(4,2,400),(5,8,500);
```

| 时刻 | 事务A-RR或RC                                                 | 事务B-RR或RC                                                 |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | Begin                                                        |                                                              |
| T2   | SQL1 ： select from a_dict where id=3 for update; 或者 delete from a_dict where id=3; 或者update | Begin                                                        |
| T3   |                                                              | SQL2： select from a_dict where field_id=2 for update ; 或者 delete from a_dict where field_template_id=2; 或者update |
| T4   |                                                              | 等待                                                         |
| T5   | commit                                                       |                                                              |

  分析
- sql1通过主键索引锁定id=3的数据

- sql2通过idx_p锁定field_id=2的数据，这之中包含id=1、2、3、4的数据

- sql2执行时锁定的数据与sql1重叠，因此等待
  总结：涉及到当前读的sql时多考虑索引和锁定数据的情况
  至于为什么锁定数据重叠会冲突，我个人猜想是和索引底层实现有关系
  [图片]
  [图片]
  主键索引内存储的值里是实际数据(0001,apple,6.00)，其他索引内存储的值是主键索引的值(1,2,3,4,5)。因此其他索引锁定数据时，也会涉及到相应的主键索引。仅为个人猜测，瓜不保熟。
  
  这个案例看起来代码简单很清晰，后边会举一个业务系统里更加真实复杂的案例

## insert+唯一索引并发

### insert锁规则

  接下来引用[官方链接](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)中的翻译。

  > INSERT 在插入的行上设置排他锁。 该锁是索引记录锁，而不是下一个键锁（即没有间隙锁），并且不会阻止其他会话插入到插入行之前的间隙中。
  > 在插入行之前，会设置一种称为插入意向间隙锁的间隙锁。 此锁表明插入的意图是，插入同一索引间隙的多个事务如果没有插入间隙内的同一位置，则无需互相等待。 假设存在值为 4 和 7 的索引记录。尝试插入值 5 和 6 的单独事务在获得插入行上的排他锁之前，每个事务都使用插入意向锁锁定 4 和 7 之间的间隙，但不这样做 相互阻塞，因为行不冲突。
  > 如果发生重复键错误，则会在重复索引记录上设置共享锁。 如果另一个会话已经拥有排它锁，则如果多个会话尝试插入同一行，则使用共享锁可能会导致死锁。 如果另一个会话删除该行，则可能会发生这种情况

  

  补一下插入意向锁+隐式锁的概念
  这里直接搬运[阿里的链接](http://mysql.taobao.org/monthly/2020/09/06/)，参考其中"Insert语句的加锁流程"小节
  [图片]
  这种场景日常出现的频率很高。

### 场景复现

  版本：5.7.24& 8.0.20

  ```sql
  CREATE TABLE `dl` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `num` int(10) unsigned DEFAULT NULL,
  `val` varchar(30) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `num_index` (`num`)
  ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
  insert into dl(num,val) values (10,'1'),(20,'2'),(30,'30'),
  (50,'50'),(60,'60');假设业务场景是，某个请求重复提交
  ```
| 时刻 | 事务A-RC或RR                                       | 事务B-RC或RR                                                 |
| ---- | -------------------------------------------------- | ------------------------------------------------------------ |
| T1   | begin                                              |                                                              |
| T2   | SQL1: insert into dl(num,val) values(102,'sess1'); | begin                                                        |
| T3   |                                                    | SQL2: insert into dl(num,val) values(102,'sess1');           |
| T4   |                                                    | 等待                                                         |
| T5   | SQL3: insert into dl(num,val) values(101,'sess1'); |                                                              |
| T6   |                                                    | 抛错死锁                                                     |
| T7   |                                                    | -- 轮不到这条sq执行了 insert into dl(num,val) values(101,'sess1'); |
| T8   | commit                                             |                                                              |

  insert这块的逻辑我不是很理解，这里引用别人之前的总结
>insert 操作在RC 隔离级别下
 如果无 unique key 则为：Lock_x + Lock_REC_NOT_RAP 
 有unique key 则为
 唯一性约束检查： Lock_x +LOCK_ORDINARY
 插入位置有GAP锁：LOCK_INSERT_INTENTION
 新数据插入Lock_x + Lock_REC_NOT_GAP

  分析
  在5.27和8.0.20都会死锁，只是死锁日志显示方面8.0.20更全

- sql1:通过num索引获取到X,REC_NOT_GAP，也就是行锁。
- -->另外，由于是插入，因此会加一个gap锁(60,102)-->锁情况中此时没有显示该条，应该是隐式锁转换的原因
- sql2:由于102的值已经重复了，因此会尝试申请num索引上102的S锁，以便读到最新的数据。但是102的X锁被事务A持有了，根据锁的兼容性，只能排队等待事务A释放
- sql3：根据死锁日志来看，多了等待一个gap lock，推测是sql2进行的隐式锁转换，gap范围是(60,102)。因为sql3这是101，位于这个区间内，因此sql3能执行成功
（死锁日志中：hex00000066也就是102）
- sql3执行完之后,事务B与事务A形成死锁，故回滚一个事务（如何导致的呢？存疑，一种推测是事务二看到事务一虽然先拿到了102的行X锁，大家也都持有(下限,102)的gap锁，但是一看到事务一又通过insert101拿到了该gap内又一行数据的行X锁，觉得自己没希望在gap内插入数据了所以自己放弃自己了。另一种推测是事务一insert101的时候看到本事务持有了(下限,102)的gap锁,101的行X本事务也有了，因此觉得自己能够拿到该gap内其他行X锁，于是就成功执行了insert101，并把其他不能在该gap内插入数据的事务给ban掉了）
- -->回滚后，事务A的锁情况中，成功获得了102的 X,GAP,INSERT_INTENTION
[图片]
  解决方式：
- 避免LOCK_ORDINARY(就是Next-Key，这是老版本的叫法)锁，通过主键操作 持有行锁;
- 也可以通过 insert into t3(xx,xx) on duplicate key update `xx`='XX'; mysql特有的语法来解决此问题。因为insert语句对于主键来说，插入的行不管有没有存在，都会只有行锁
- 大事务改小事务，减少锁的持有时间

  继续讨论这个例子，如果调换sql顺序，先insert 101 再insert102又是一种分析结果了。这里不再展开分析了，大家自行实验。

  继续讨论这个例子，如果把唯一索引从数字换成字符串，需要考虑字符串的大小比较规则了（比如'5' > '20'）。但是考虑到实践的情况,如果是单据号一类的字符串唯一索引，一般采用发号器，发号器产生的数据长度固定，且递增增长，因此单据号一类的insert相对不容易出问题。
[图片]
### 变种
  一条insert，未做好并发控制导致重复提交。
| 时刻 | 事务A                                              | 事务B                                              | 事务C                                              |
| ---- | -------------------------------------------------- | -------------------------------------------------- | -------------------------------------------------- |
| T1   | begin                                              |                                                    |                                                    |
| T2   | SQL1: insert into dl(num,val) values(102,'sess1'); | Begin                                              |                                                    |
| T3   |                                                    | SQL2: insert into dl(num,val) values(102,'sess1'); |                                                    |
| T4   |                                                    | 等待                                               | Begin                                              |
| T5   |                                                    |                                                    | SQL3: insert into dl(num,val) values(102,'sess1'); |
| T6   |                                                    |                                                    | 等待                                               |
| T7   | 主动回滚                                           |                                                    |                                                    |
| T8   |                                                    | 报错死锁                                           |                                                    |
| T9   |                                                    |                                                    | sql执行，可commit                                  |

  这个例子不讲解锁分析原因，和官方定义中举得例子一样
分析
- 这句insert本身，会有插入意向锁

- sql1执行后，暂时看不到什么
  [图片]

- sql2执行后，等待。由于隐式锁转换，替事务A加了锁，sql2自身陷入等待。根据锁情况，反向推测45938是事务B，事务A是45933，获得了102的行锁(X,REC NOT GAP)
  [图片]

- sql3执行时，直接开始排队等待，（由于我重新从头开始执行，因此事务id等与之前的图不一样了）。根据锁情况，反向推测45939是事务A，获得了102的行锁(X,REC NOT GAP)；45940是事务B，等待102的S锁；45941是事务C，等待102的S锁
  [图片]

- sql1回滚时，释放锁。这时候剩下两个事务，一个sql执行成功，一个sql执行失败。
  [图片]

- 此时剩下两个事务里，任意回滚一个，锁都会没有了
## insert+update rc或rr死锁
  （基于8.0.20实验,隔离级别RR/RC,案例自身基于5.7.24+rr级别，实测该版本下rc级别也会死锁）
  
  有一张一对多关系中的子表，采用左右序号来存储树形数据

  ```sql
  create table t_field_dict
  (
    id                int(11) unsigned auto_increment comment '自动主键' primary key,
    field_id int                                    not null comment '字段模版id',
    value_name        varchar(255)                  not null comment '枚举值名称',
    parent_id         int          default 0        not null comment '父id',
    left_value        int          default 0        not null comment '左序号',
    right_value       int          default 0        not null comment '右序号',
    is_delete         tinyint(3)   default 0        not null comment '是否删除 0:未删除 1:已删除',
    KEY `idx_p` (`field_id`),
  ) comment '枚举值表';
  ```

  ```java
  @Transactional(isolation = Isolation.REPEATABLE_READ, rollbackFor = Exception.class)
  -- 这里特意改为RR可能原意图是为了防止插入的数据重复
  public void addFirstEnum(FieldDictAddRequest model){
    //1.根据parentId查询父数据是否存在
    FieldDict parentDict = db.queryById(model.getParentId());
    if(Objects.isNull(parentDict)){
        //SQL一：insert父数据(若父数据不存在则插入父数据)
        insert into t_field_dict(field_id, value_name, parent_id,left_value, right_value)
            VALUES (100,'枚举根节点',0,1,2);
    }
    //2.更新其他数据的左右值
    int increase = model.getValueNames().length * 2;
    //SQL二：更新其他数据的左值
    update t_field_dict set left_value = left_value + ${increase} 
    where field_template_id = 100 and left_value >= ${parent.right_value} and is_delete = 0;
    //SQL三:更新其他数据的右值
    update t_field_dict set right_value = right_value + ${increase} 
    where field_template_id = 100 and right_value >= ${parent.right_value} and is_delete = 0;
    //3.插入子数据   
    int value = parentDict.getRightValue();
    for (String name : model.getValueNames()) {
        FieldDict dict = new FieldDict();
        dict.setFieldTemplateId(model.getFieldTemplateId());
        dict.setParentId(parentDict.getId());
        dict.setLeftValue(value);
        dict.setRightValue(value + 1);
        //SQL四：插入子数据
        insert into t_field_dict(field_template_id, value_name, parent_id,left_value, right_value)
            VALUES (11502,${parentId},'xxx',${value},${value+1});
        value = value + 2;
     }
     //插入完毕
  }
  ```

  同样的接口请求，参数重复提交，入口没有通过分布式锁或其他方式来做防重复提交，交替执行两个事务的SQL
  
  假设场景是要在某个父数据下初始化增加苹果,它的父数据'枚举根节点'在数据库中不存在,根数据采用惰性插入，此时表内数据为空,因此会执行insert。
  
| 时间 | 事务A-RR+8.0.20                                              | 事务B-RR+8.0.20                                              |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | START TRANSACTION;                                           |                                                              |
| T2   |                                                              | START TRANSACTION;                                           |
| T3   | insert into t_field_dict(field_id, value_name, parent_id,left_value, right_value)             VALUES (100,'枚举根节点',0,1,2) |                                                              |
| T4   |                                                              | insert into t_field_dict(field_id, value_name, parent_id,left_value, right_value)             VALUES (100,'枚举根节点',0,1,2) |
| T5   | update t_field_dict set ... where field_template_id = 100 and left_value >=2 and is_delete=0; |                                                              |
| T6   | 等待                                                         |                                                              |
| T7   |                                                              | update t_field_dict set ... where field_template_id = 100 and left_value >=2 and is_delete=0; |
| T8   |                                                              | Deadlock found when trying to get lock                       |
| T9   |                                                              | 根据代码配置，异常时回滚                                     |

  简单分析：插入时插入意向锁，update时field_template_id = 100 and left_value >=2的条件锁定的数据中恰好包含前边insert的数据，因此一个事务的update会等待另一个事务的insert，四条sql互相等待构成死锁。
(update锁定的数据：rc下是行锁，field_template_id = 100 and left_value >=2的这部分数据，rr下也许是gap锁了)

  详细分析
- 事务A执行insert。获取意向X锁

- 个人理解：根据文档所述，insert还会加一个record x lock，这里可能是源自官方的优化：隐式锁，
  [图片]

- 事务B执行insert。获取意向IX锁
  [图片]

- 事务A执行update后--RR模式
  [图片]
  倒数第二行,X,REC_NOT_GAP--> 事务A的insert语句，通过主键索引,锁定了19
  100,19 --> 事务A通过idx_p索引，拿到了19的间隙锁+行锁， X REC_NOT_GAP + X GAP
  100,20 -->事务A通过idx_p索引，拿到了100,20的行锁，但是等待X锁（事务B已经拿过了20的主键，于是等待事务B释放）

- 事务A执行update后-RC模式
  [图片]

- 事务B执行update，报错，(看数据库死锁日志)

- 与事务A执行update同样的获取锁情况，拿到了主键的20，等待19的X锁。至此形成了互相等待释放的情况

- rr下的死锁日志
  [图片]

- 然后事务B被回滚-rr
  [图片]
  通过这个例子，我想表达的意思是：当一个方法内逻辑足够复杂时或者希望能够写出通用一些的代码(新增一级枚举、新增非一级枚举共用一套代码)，可能开发人员会将思考重心放在了业务逻辑内，忽略对一些sql死锁的考虑，再加上本例子需要特定分支+并发才能触发，更加加大了在测试环境提前发现的难度。

## RR级别下Record Lock升级Next-Key Lock

  顺着上一个业务案例，我们来看另外一个死锁现象。
  
  该案例来自阿里的[一次InnoDB死锁Bug排查案例](http://mysql.taobao.org/monthly/2022/02/01/)，MySQL8.0.18之前的版本。反馈该案例在8.0.18之前会死锁，但8.0.18及之后的版本可以正常执行。
  

  ```sql
  CREATE TABLE t  (
    id BIGINT UNSIGNED NOT NULL PRIMARY KEY COMMENT 'id, 无实际意义',
    account_id VARCHAR (64) NOT NULL COMMENT '用户id，不同app下的account_id可能重复',
    type TINYINT UNSIGNED NOT NULL COMMENT '余额类型 1:可用余额',
    balance BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '余额',
    state INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '账户状态 1:NORMAL; 2:FROZE',
    UNIQUE KEY uk_account (account_id, type)
  )ENGINE = INNODB DEFAULT CHARSET utf8mb4
  COMMENT '测试';
  insert into t values(1,'1',1,100,1);
  insert into t values(2,'2',1,100,1);
  insert into t values(3,'3',1,100,1);
  insert into t values(4,'4',1,100,1);
  insert into t values(5,'5',1,100,1);
  ```
| 时刻 | 事务A-RR                                                     | 事务B-RR                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T1   | Begin                                                        |                                                              |
| T2   | select * from t where account_id = '1' and type =1 for update; | Begin                                                        |
| T3   |                                                              | select * from t where account_id = '1' and type =1 for update; |
| T4   |                                                              | 等待行锁                                                     |
| T5   | update t set state = 2 where account_id = '1';               |                                                              |
| T6   | 8.0.18之前：抛错死锁回滚 8.0.18及之后：正常执行              |                                                              |

分析
- Select 语句会锁定Record Lock 这条数据(1,'1',1,100,1)

- 事务A执行select获取了行锁

- 事务B执行select，等待事务A释放行锁

- 事务A执行update，需要Next-key Lock
  - 8.0.18之前：事务 A无法立即获得 X record lock，认为可能会发生的死锁原因是因为在整个 lock 的等待关系中存在一个环, 即 A不 commit 提交事务, B 事务也无法获取 X record lock, 从而导致 A 的 UPDATE 语句也无法获得 X record lock 组成 Next-key record lock, 即使 A 已经持有了 X record lock.
  
  - 8.0.18及之后：修改了判断算法。当尝试获取 Next-key record lock 时，先判断当前 trx 是否持有 X record lock, 假如持有就复用这个 X record lock, 从而直接申请 GAP lock

  这里我想表达：不同mysql版本下加锁规则略有变化，需要结合版本进行分析
  

# 附录-一些链接
[Innodb事务子系统介绍-阿里](http://mysql.taobao.org/monthly/2015/12/01/)
[Innodb事务锁系统介绍-阿里](http://mysql.taobao.org/monthly/2016/01/01/)
MySQL45讲-丁奇-极客时间
《高性能MYSQL》
[不同的自增锁模式下锁的规则-官方5.7](https://dev.mysql.com/doc/refman/5.7/en/innodb-auto-increment-handling.html)
