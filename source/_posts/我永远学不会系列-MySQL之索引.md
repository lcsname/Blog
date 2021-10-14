---
layout: posts
title: 我永远学不会系列-MySQL之索引
date: 2018-08-19 00:38:36
categories: MySQL
tags: [MySQL,数据库]
top: false
---
一般的应用系统，读写比例在10:1左右，而且插入操作和一般的更新操作很少出现性能问题，遇到最多的，也是最容易出问题的，还是一些复杂的查询操作，所以查询语句的优化显然是重中之重，在数据量和访问量不大的情况下，MySQL访问是非常快的，是否加索引对访问影响不大。但是当数据量和访问量剧增的时候，就会发现MySQL变慢，甚至down掉，这就必须要考虑优化SQL了，给数据库建立正确合理的索引，是MySQL优化的一个重要手段。

<!--more--> 

#### 一、有关MySQL的索引的作用

索引的目的在于提高查询效率，可以类比字典，如果要查“mysql”这个单词，我们肯定需要定位到m字母，然后从上往下找到y字母，再找到剩下的sql。如果没有索引，那么你可能需要把所有单词看一遍才能找到你想要的。除了词典，生活中随处可见索引的例子，如火车站的车次表、图书的目录等。它们的原理都是一样的，通过不断的缩小想要获得数据的范围来筛选出最终想要的结果，同时把随机的事件变成顺序的事件，也就是我们总是通过同一种查找方式来锁定数据。

在创建索引时，需要考虑哪些列会用于 SQL 查询，然后为这些列创建一个或多个索引。事实上，索引也是一种表，保存着主键或索引字段，以及一个能将每个记录指向实际表的指针。数据库用户是看不到索引的，它们只是用来加速查询的。数据库搜索引擎使用索引来快速定位记录。

INSERT 与 UPDATE 语句在拥有索引的表中执行会花费更多的时间，而SELECT 语句却会执行得更快。这是因为，在进行插入或更新时，数据库也需要插入或更新索引值。

#### 二、索引的创建、删除

索引的类型：

- `UNIQUE(唯一索引)`：不可以出现相同的值，可以有NULL值
- `INDEX(普通索引)`：允许出现相同的索引内容
- `PROMARY KEY(主键索引)`：不允许出现相同的值
- `fulltext index(全文索引)`：可以针对值中的某个单词，但效率确实不敢恭维
- `组合索引`：实质上是将多个字段建到一个索引里，列值的组合必须唯一

##### A.使用ALTER TABLE语句创建索性

应用于表创建完毕之后再添加

```
ALTER TABLE 表名 ADD 索引类型 （unique,primary key,fulltext,index）[索引名]（字段名）
--普通索引
alter table table_name add index index_name (column_list) ;
--唯一索引
alter table table_name add unique (column_list) ;
--主键索引
alter table table_name add primary key (column_list) ;
```

##### B.使用CREATE INDEX语句对表增加索引

CREATE INDEX可用于对表增加**普通索引或UNIQUE索引**，可用于建表时创建索引。

```
CREATE INDEX index_name ON table_name(username(length));
```

如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length。

```
--create只能添加这两种索引;
CREATE INDEX index_name ON table_name (column_list)
CREATE UNIQUE INDEX index_name ON table_name (column_list)
```

> **不能用CREATE INDEX语句创建PRIMARY KEY索引**。

##### C.删除索引

删除索引可以使用ALTER TABLE或DROP INDEX语句来实现。DROP INDEX可以在ALTER TABLE内部作为一条语句处理，其格式如下：

```
drop index index_name on table_name ;
alter table table_name drop index index_name ;
alter table table_name drop primary key ;
```

其中，在前面的两条语句中，都删除了table_name中的索引index_name。而在最后一条语句中，只在删除PRIMARY KEY索引中使用，因为**一个表只可能有一个PRIMARY KEY索引**，因此不需要指定索引名。如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引。

##### D.组合索引与前缀索引

组合索引和前缀索引是对建立索引技巧的一种称呼，并不是索引的类型

```
create table USER_DEMO
(
   ID                   int not null auto_increment comment '主键',
   LOGIN_NAME           varchar(100) not null comment '登录名',
   PASSWORD             varchar(100) not null comment '密码',
   CITY                 varchar(30) not null comment '城市',
   AGE                  int not null comment '年龄',
   SEX                  int not null comment '性别(0:女 1：男)',
   primary key (ID)
);
```

为了进一步榨取mysql的效率，就可以考虑建立组合索引，即将LOGIN_NAME,CITY,AGE建到一个索引里：

```
ALTER TABLE USER_DEMO ADD INDEX name_city_age (LOGIN_NAME(16),CITY,AGE);
```

建表时，LOGIN_NAME长度为100，这里用16，是因为一般情况下名字的长度不会超过16，这样会加快索引查询速度，还会减少索引文件的大小，提高INSERT，UPDATE的更新速度。

如果分别给LOGIN_NAME,CITY,AGE建立单列索引，让该表有3个单列索引，查询时和组合索引的效率是大不一样的，甚至远远低于组合索引。虽然此时有三个索引，但mysql只能用到其中的那个它认为似乎是最有效率的单列索引，另外两个是用不到的，也就是说还是一个全表扫描的过程。

建立这样的组合索引，就相当于分别建立如下三种组合索引：

```
LOGIN_NAME,CITY,AGE
LOGIN_NAME,CITY
LOGIN_NAME
```

为什么没有CITY,AGE等这样的组合索引呢？这是因为mysql组合索引`“最左前缀”`的结果。简单的理解就是只从最左边的开始组合，并不是只要包含这三列的查询都会用到该组合索引。也就是说**name_city_age(LOGIN_NAME(16),CITY,AGE)从左到右进行索引，如果没有左前索引，mysql不会执行索引查询**。

#### 三、索引的使用及注意事项

尽量避免这些不走索引的sql：

```
SELECT `sname` FROM `stu` WHERE `age`+10=30; --不会使用索引,因为所有索引列参与了计算
SELECT `sname` FROM `stu` WHERE LEFT(`date`,4) < 1990; --不会使用索引,因为使用了函数运算,原理与上面相同
SELECT * FROM `houdunwang` WHERE `uname` LIKE '后盾%'; --走索引
SELECT * FROM `houdunwang` WHERE `uname` LIKE "%后盾%"; -- 不走索引
```

索引虽然好处很多，但过多的使用索引可能带来相反的问题，索引也是有缺点的：

- 虽然索引大大提高了查询速度，同时却会降低更新表的速度，如对表进行INSERT,UPDATE和DELETE。因为更新表时，mysql不仅要保存数据，还要保存一下索引文件。
- 建立索引会占用磁盘空间的索引文件。一般情况这个问题不太严重，但如果你在要给大表上建了多种组合索引，索引文件会膨胀。

索引只是提高效率的一个方式，如果mysql有大数据量的表，就要花时间研究建立最优的索引，或优化查询语句。

使用索引时，有一些技巧：

- 索引不会包含有NULL的列。
  - 只要列中包含有NULL值，都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此符合索引就是无效的。
- 使用短索引
  - 对串列进行索引，如果可以就应该指定一个前缀长度。例如，如果有一个char（255）的列，如果在前10个或20个字符内，多数值是唯一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。
- 索引列排序
  - mysql查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作，尽量不要包含多个列的排序，如果需要最好给这些列建复合索引。
- like语句操作
  - 一般情况下不鼓励使用like操作，如果非使用不可，注意正确的使用方式。`like ‘%aaa%’`不会使用索引，而`like ‘aaa%’`可以使用索引。
- 不要在列上进行运算
- 不使用NOT IN 、<>、！=操作，但<,<=，=，>,>=,BETWEEN,IN是可以用到索引的
- 索引要建立在经常进行select操作的字段上。
- 索引要建立在值比较唯一的字段上。
- 对于那些定义为text、image和bit数据类型的列不应该增加索引。因为这些列的数据量要么相当大，要么取值很少。
- 在where和join中出现的列需要建立索引。
- where的查询条件里有不等号(where column != …),mysql将无法使用索引。
- 如果where字句的查询条件里使用了函数(如：where DAY(column)=…),mysql将无法使用索引。
- 在join操作中(需要从多个数据表提取数据时)，mysql只有在**主键和外键的数据类型相同**时才能使用索引，否则即使建立了索引也不会使用。


