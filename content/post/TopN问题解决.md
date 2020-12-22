---
title: "TopN问题解决"
date: 2020-11-24T17:24:56+08:00
description: "Article description."
draft: false
toc: TRUE
categories:
  - 数据库
tags:
  - MySQL
  - TopN
---

# 需求

将数据分组,每组内取前n条.最常见的需求是取每组内第一条,例如以imei分组,组内取time最新的一条

# 表结构

```
create table com(
  n_id int auto_increment primary key,
  c_imei varchar(10) null,
  c_time bigint null,
  c_name varchar(10) null
);

create index com_c_imei_index on com (c_imei);
```

# 表数据

```
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (1, 'a', 8, '010101');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (2, 'e', 2, '020202');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (3, 'c', 9, '030303');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (4, 'b', 4, '040404');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (5, 'd', 5, '050505');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (6, 'a', 6, '060606');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (7, 'e', 4, '070707');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (8, 'c', 3, '0808080');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (9, 'b', 5, '090909');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (10, 'd', 8, '101010');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (11, 'a', 5, '111111');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (12, 'e', 7, '121212');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (13, 'c', 2, '131313');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (14, 'b', 6, '141414');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (15, 'd', 9, '151515');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (16, 'a', 2, '161616');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (17, 'e', 1, '171717');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (18, 'c', 5, '181818');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (19, 'b', 8, '191919');
INSERT INTO com (n_id, c_imei, c_time, c_name) VALUES (20, 'd', 7, '202020');
```

# SQL

```
#方法一，自连接
SELECT a.c_imei, a.n_id, a.c_time
FROM com a
 LEFT JOIN com b
 ON a.c_imei = b.c_imei
 AND a.c_time < b.c_time
WHERE b.c_time IS NULL
ORDER BY a.c_imei;

#方法一的另一种形式,如果要取每组内前n条，那么将1改成n即可
SELECT n_id, c_imei, c_time, c_name
FROM com a
WHERE (SELECT count(*) FROM com b WHERE a.c_imei = b.c_imei AND a.c_time < b.c_time) < 1
order by c_imei;


#方法二，派生表排序后分组，注意limit必须加不然没用
select n_id, c_imei, c_time, c_name
from (select n_id, c_imei, c_time, c_name from com order by c_time desc limit 999999) a
group by a.c_imei;

#方法三,相关子查询，注意GROUP_CONCAT结果的长度受限于group_concat_max_len，默认1024
SELECT n_id, c_imei, c_time, c_name
FROM com
WHERE n_id IN (SELECT SUBSTRING_INDEX(GROUP_CONCAT(n_id ORDER BY c_time DESC), ',', 1) FROM com GROUP BY c_imei)
ORDER BY c_imei;

#方法四,派生表关联查询
select distinct com.n_id, com.c_imei, com.c_time, com.c_name
from com
 join (select c_imei, max(c_time) as ct from com group by c_imei) tmp
 on com.c_imei = tmp.c_imei and com.c_time = tmp.ct
order by com.c_imei;

##方法四优化
select distinct com.n_id, com.c_imei, com.c_time, com.c_name
from com
 right join (select c_imei, max(c_time) as ct from com group by c_imei) tmp
 on com.c_imei = tmp.c_imei and com.c_time = tmp.ct
order by com.c_imei;
```

## 其他方法

MySQL8及以上的row_number、rank、dense_rank、over函数