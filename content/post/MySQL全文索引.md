---
title: "MySQL全文索引使用"
date: 2020-10-06T22:45:59+08:00
description: "Article description."
draft: false
toc: true
categories:
  - MySQL
tags:
  - MySQL
  - 数据库
  - 全文索引
comment: false
---
# 1.简介 {#1}

在Web应用中,经常会遇到按照关键字进行模糊搜索的需求,当参数搜索的数据量较少时,我们一般使用like进行搜索,但是当数据量达到一定程度后,like方式的速度就会很慢很慢,这时候我们可以借助一些全文搜索的组件来实现需求.MySQL就提供了全文索引来支持模糊搜索.

# 2.限制条件 {#2}

## 2.1引擎限制 {#2-1}

MySQL 5.6 以前的版本，只有 MyISAM 存储引擎支持全文索引；

MySQL 5.6 及以后的版本，MyISAM 和 InnoDB 存储引擎均支持全文索引

## 2.2版本号限制 {#2-2}

Mysql自v5.6.24版本开始在InnoDB引擎中增加全文索引功能，支持对英文的全文搜索,默认以空格作为分隔符;自v5.7版本开始增加ngram分词器以支持中文

## 2.3字段类型限制 {#2-3}

全文索引支持的字段类型为char、varchar、text等这些基于文本的列

## 2.4连表限制 {#2-4}

全文搜索仅支持在同一张表中进行,不支持对多张表中的关键字进行全文搜索



# 3.准备索引 {#3}

 我们以report表为例

```mysql
-- 准备表
create table report
(
    id      int auto_increment
        primary key,
    content varchar(1000) null
);
-- 在content字段创建普通的全文索引
create fulltext index content_fti on report(content);
-- 在content字段创建支持中文的全文索引
create fulltext index content_fti on report(content) WITH PARSER ngram;
-- 删除索引,方式一
drop index content_fti on report;
-- 删除索引,方式二
alter table report drop index content_fti;

```

# 4.准备配置 {#4}
​    使用全文索引搜索时,搜索引擎受全文搜索的单词长度影响,如果关键词长度小于该配置项,那么将无法搜索出相匹配的结果,通过命令可以查看出相关配置项

```mysql
-- 查看全文搜索配置
show variables like '%ft%';
-- 命令执行结果
// MyISAM:关键词最小长度默认4字符,最大长度84字符
ft_min_word_len = 4;
ft_max_word_len = 84;
// InnoDB:关键词最小长度默认3字符,最大长度84字符
innodb_ft_min_token_size = 3;
innodb_ft_max_token_size = 84;
```

我们以常用的Innodb引擎为例,在MySQL的配置文件中修改配置项

```mysql
[mysqld]
innodb_ft_min_token_size = 1
ft_min_word_len = 1
```

修改后需要重启MySQL,然后修复全文索引(可以删除索引然后重新建立索引,如果是MyIsam引擎,也可以执行repair命令修复)

然而对于使用了ngram的全文索引来讲,它的全文搜索单词长度配置会忽略上述四个配置项,真正生效的为配置项ngram_token_size(默认2),可以通过在MySQL的配置文件中修改以下配置项或启动时追加参数--ngram_token_size=1来实现对该配置项的修改

```
[mysqld]
ngram_token_size=1
```

同样的,修改此项后需要重建全文索引



# 5.准备数据 {#5}

略

# 6.使用索引 {#6}

与like不同,全文索引的搜索需要使用match agnist,示例如下

```
select * from report where match(content) against('测试关键词');
```

match agnist本身还会返回非负浮点数作为搜索的结果行与关键词的相关度.除了match agnist的基础使用,全文搜索还支持以不同的检索模式进行搜索,常用的全文检索模式有两种：
 1、自然语言模式(NATURAL LANGUAGE MODE) ，
 自然语言模式是MySQL 默认的全文检索模式。自然语言模式不能使用操作符，不能指定关键词必须出现或者必须不能出现等复杂查询。当sql中指定了`IN NATURAL LANGUAGE MODE`修饰符或未给出修饰符，则全文搜索是自然语言搜索模式 。
 2、BOOLEAN模式(BOOLEAN MODE)
 BOOLEAN模式可以使用操作符，可以支持指定关键词必须出现或者必须不能出现或者关键词的权重高还是低等复杂查询。

## 6.1自然语言检索模式 {#6-1}



​    在该模式下,可以指定IN NATURAL LANGUAGE MOD,也可以不指定修饰符,下面给出一个按照结果行相关度倒序排列的SQL示例

```mysql
select *,match(content) against('一切') as score from report where match(content) against('一切') order by score desc;
```

## 6.2布尔检索模式 {#6-2}

MySQL可以使用`IN BOOLEAN MODE`修饰符执行布尔型全文本搜索 。在这种模式下,支持通过一些正则来进行高级搜索,布尔模式下支持以下操作符：

* “+”表示必须包含
* “-”表示必须排除
* “>”表示出现该单词时增加相关性
* “<”表示出现该单词时降低相关性
* “*”表示通配符
* “~”允许出现该单词，但是出现时相关性为负
* “""”表示短语
  下面给出一些示例

```
'apple banana' 
## 无操作符，表示或，要么包含apple，要么包含banana

'+apple +juice'
## 必须同时包含两个词apple和juice

'+apple macintosh'
## 必须包含apple，但是如果也包含macintosh的话，相关性会更高。

'+apple -macintosh'
## 必须包含apple，同时不能包含macintosh。

'+apple ~macintosh'
## 必须包含apple，但是如果也包含macintosh的话，相关性要比不包含macintosh的记录低。

'+apple +(>juice <pie)'
## 查询必须包含apple和juice或者apple和pie的记录，但是apple juice的相关性要比apple pie高。

'apple*'
## 查询包含以apple开头的单词的记录，如apple、apples、applet。

'"some words"'
## 使用双引号把要搜素的词括起来，效果类似于like '%some words%'，
例如“some words of wisdom”会被匹配到，而“some noise words”就不会被匹配。
```

# 7.InnoDB引擎的相关性 {#7}

InnoDB引擎的全文索引基于[Sphinx](http://sphinxsearch.com/),算法基于[BM-25](http://en.wikipedia.org/wiki/Okapi_BM25)和[TF-IDF](http://en.wikipedia.org/wiki/TF-IDF),`InnoDB`使用“术语频率-逆文档频率” （`TF-IDF`）加权系统的变体对给定的全文搜索查询对文档的相关性进行排名,单词出现在文档中的频率越高，单词出现在文档集合中的频率越低，文档的排名就越高。

## 7.1相关性排名的计算方式 {#7-1}

术语频率（`TF`）值是单词在文档中出现的次数。`IDF`单词的逆文档频率（）值是使用以下公式计算的，其中 `total_records`是集合中`matching_records`的记录数，并且是搜索词出现的记录数。

```simple
${IDF} = log10( ${total_records} / ${matching_records} )
```

当文档多次包含一个单词时，IDF值将乘以TF值：

```simple
${TF} * ${IDF}
```

使用`TF`和`IDF` 值，使用以下公式计算文档的相关性等级：

```simple
${rank} = ${TF} * ${IDF} * ${IDF}
```

# 8.停止词 {#8}

可以通过配置停止词来禁止某些词语参与全文索引,详细使用见[全文停用词](https://dev.mysql.com/doc/refman/5.7/en/fulltext-stopwords.html)

# 9.InnoDB分词原理 {#9}

`InnoDB` 全文索引具有倒排索引设计。倒排索引存储一个单词列表，对于每个单词，存储单词出现的文档列表。为了支持邻近搜索，每个单词的位置信息也作为字节偏移量存储。

创建全文索引时,MySQL将创建一组表用于辅助

```
## 查看索引表
SELECT table_id, name, space from INFORMATION_SCHEMA.INNODB_SYS_TABLES
       WHERE name LIKE 'test/%';
## 命令执行结果
424	test/FTS_000000000000006b_0000000000000388_INDEX_1	423
425	test/FTS_000000000000006b_0000000000000388_INDEX_2	424
426	test/FTS_000000000000006b_0000000000000388_INDEX_3	425
427	test/FTS_000000000000006b_0000000000000388_INDEX_4	426
428	test/FTS_000000000000006b_0000000000000388_INDEX_5	427
429	test/FTS_000000000000006b_0000000000000388_INDEX_6	428
430	test/FTS_000000000000006b_BEING_DELETED	429
431	test/FTS_000000000000006b_BEING_DELETED_CACHE	430
432	test/FTS_000000000000006b_CONFIG	431
433	test/FTS_000000000000006b_DELETED	432
434	test/FTS_000000000000006b_DELETED_CACHE	433
107	test/report	93
```

前六个表代表反向索引，并称为辅助索引表。对传入文档进行标记时，各个单词（也称为 “标记”）与位置信息和关联的文档ID（`DOC_ID`）一起插入索引表中。根据单词第一个字符的字符集排序权重，单词在六个索引表中得到完全排序和分区。

倒排索引分为六个辅助索引表，以支持并行索引创建。默认情况下，两个线程对索引表中的单词和相关数据进行标记化，排序和插入。线程数可以使用该[`innodb_ft_sort_pll_degree`](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_ft_sort_pll_degree) 选项配置 。`FULLTEXT`在大型表上创建索引时，请考虑增加线程数 。

辅助索引表名称以前缀 `FTS_`和后缀 `INDEX_*`。每个索引表通过索引表名称中与`table_id`索引表的匹配的十六进制值与索引表相关联。例如，`table_id`所述的 `test/opening_lines`表是 `327`，为此，十六进制值是0x147。如前面的示例所示，十六进制值“ 147 ”出现在与该`test/opening_lines`表关联的索引表的名称中。