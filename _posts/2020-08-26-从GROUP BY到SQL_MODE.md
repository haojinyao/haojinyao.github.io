---
title: 从GROUP BY到SQL_MODE
tags: [SQL,SQL_MODE]
date: 2020-08-26 22:30:04
categories:
- SQL
---


&emsp;&emsp;今天看到一篇讲GROUP BY的文章，引申出了SQL_MODE，让我不禁回想起了当年安装Airflow时被支配的恐惧。看完以后做个复盘叭。

&emsp;&emsp;标准 SQL 规定，在对表进行聚合查询的时候，只能在 SELECT 子句中写下面 3 种内容：通过 GROUP BY 子句指定的聚合键、聚合函数（SUM 、AVG 等）、常量。假如数据表如下：

```sql
DROP TABLE IF EXISTS tbl_student_class;
CREATE TABLE tbl_student_class (
  id int(8) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  sno varchar(12) NOT NULL COMMENT '学号',
  cno varchar(5) NOT NULL COMMENT '班级号',
  cname varchar(20) NOT NULL COMMENT '班级名',
  PRIMARY KEY (id)
) COMMENT='学生班级表';

-- ----------------------------
-- Records of tbl_student_class
-- ----------------------------
INSERT INTO tbl_student_class VALUES ('1', '20190607001', '0607', '影视7班');
INSERT INTO tbl_student_class VALUES ('2', '20190607002', '0607', '影视7班');
INSERT INTO tbl_student_class VALUES ('3', '20190608003', '0608', '影视8班');
INSERT INTO tbl_student_class VALUES ('4', '20190608004', '0608', '影视8班');
INSERT INTO tbl_student_class VALUES ('5', '20190609005', '0609', '影视9班');
INSERT INTO tbl_student_class VALUES ('6', '20190609006', '0609', '影视9班');
```

&emsp;&emsp;此时统计每个班有多少人，最大学号，一般采取的写法是：

```sql
SELECT cno,cname,count(sno),MAX(sno) 
FROM tbl_student_class
GROUP BY cno,cname;
```

&emsp;&emsp;但从表中可以知道，cno是唯一主键，那可不可以仅使用cno作为分组条件呢

```sql
SELECT cno,cname,count(sno),MAX(sno) 
FROM tbl_student_class
GROUP BY cno;
```
&emsp;&emsp;但结果会返回一个错误：SELECT 列表中的第二个表达式（cname）不在 GROUP BY 的子句中，同时它也不是聚合函数；这与 sql 模式：ONLY_FULL_GROUP_BY 不相容。
```
[Err] 1055 - Expression #2 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'test.tbl_student_class.cname' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

&emsp;&emsp;由此进入到我们的正题SQL_MODE。  
&emsp;&emsp;何谓SQL_MODE，即MySQL服务器使用的不同的配置，用来使全局或某些用户的SQL语法和数据验证检查规则不同。SQL 模式主要分两类：语法支持类和数据检查类，常用的如下:

**语法支持类**

*ONLY_FULL_GROUP_BY

对于 GROUP BY 聚合操作，如果在 SELECT 中的列、HAVING 或者 ORDER BY 子句的列，没有在GROUP BY中出现，那么这个SQL是不合法的

*ANSI_QUOTES

启用 ANSI_QUOTES 后，不能用双引号来引用字符串，因为它被解释为识别符，作用与 '一样。设置它以后，update t set f1="" …，会报 Unknown column '' in field list 这样的语法错误

*PIPES_AS_CONCAT

将 || 视为字符串的连接操作符而非 或 运算符，这和Oracle数据库是一样的，也和字符串的拼接函数 CONCAT() 相类似

*NO_TABLE_OPTIONS

使用 SHOW CREATE TABLE 时不会输出MySQL特有的语法部分，如 ENGINE ，这个在使用 mysqldump 跨DB种类迁移的时候需要考虑

*NO_AUTO_CREATE_USER

字面意思不自动创建用户。在给MySQL用户授权时，我们习惯使用 GRANT … ON … TO dbuser 顺道一起创建用户。设置该选项后就与oracle操作类似，授权之前必须先建立用户

**数据检查类**

*NO_ZERO_DATE

认为日期 ‘0000-00-00’ 非法，与是否设置后面的严格模式有关

1、如果设置了严格模式，则 NO_ZERO_DATE 自然满足。但如果是 INSERT IGNORE 或 UPDATE IGNORE，’0000-00-00’依然允许且只显示warning；

2、如果在非严格模式下，设置了NO_ZERO_DATE，效果与上面一样，’0000-00-00’ 允许但显示warning；如果没有设置NO_ZERO_DATE，no warning，当做完全合法的值；

3、NO_ZERO_IN_DATE情况与上面类似，不同的是控制日期和天，是否可为 0 ，即 2010-01-00 是否合法；

*NO_ENGINE_SUBSTITUTION

使用 ALTER TABLE 或 CREATE TABLE 指定 ENGINE 时， 需要的存储引擎被禁用或未编译，该如何处理。启用 NO_ENGINE_SUBSTITUTION 时，那么直接抛出错误；不设置此值时，CREATE用默认的存储引擎替代，ATLER不进行更改，并抛出一个 warning

*STRICT_TRANS_TABLES

设置它，表示启用严格模式。注意 STRICT_TRANS_TABLES 不是几种策略的组合，单独指 INSERT、UPDATE 出现少值或无效值该如何处理：

1、前面提到的把 ‘’ 传给int，严格模式下非法，若启用非严格模式则变成 0，产生一个warning；

2、Out Of Range，变成插入最大边界值；

3、当要插入的新行中，不包含其定义中没有显式DEFAULT子句的非NULL列的值时，该列缺少值；

&emsp;&emsp;每个版本的MySQL，默认的SQL_MODE也不相同，可以通过如下命令查看：

```sql
-- 查看 MySQL 版本
SELECT VERSION();

-- 查看 sql_mode
SELECT @@sql_mode;
```

&emsp;&emsp;返回语句中，出现ONLY_FULL_GROUP_BY模式，因此去掉就可以正常运行了。

&emsp;&emsp;但要注意，每一个约束都有其存在道理。如果去掉该模式后，在本文案例下，还能解释的通；但在cno与cname是一对多时，就会出现cname随机的问题，很难排查。因此还是遵守规范为好。

&emsp;&emsp;再说回我在安装Airflow时遇到的问题，问题是在安装成功后，使用`airflow initdb`命令，爆出  
```
sqlalchemy.exc.OperationalError: (_mysql_exceptions.OperationalError) (1292, "Incorrect datetime value: '2019-03-18 13:44:21.317153+00:00' for column 'last_scheduler_run' at row 1") [SQL: u'INSERT INTO dag (dag_id, is_paused, is_subdag, is_active, last_scheduler_run, last_pickled, last_expired, scheduler_lock, pickle_id, fileloc, owners) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)'] [parameters: ('example_skip_dag', 1, 0, 1, datetime.datetime(2019, 9, 18, 13, 44, 21, 317153, tzinfo=<Timezone [UTC]>), None, None, None, None, '/usr/local/lib/python2.7/dist-packages/airflow/example_dags/example_skip_dag.py', u'airflow')] (Background on this error at: http://sqlalche.me/e/e3q8)
```  
的问题，经过stackoverflow的查找，我在`/etc/my.cnf`中的`[mysqld]`下添加了`sql_mode="{除STRICT_TRANS_TABLES外的其他mode}"`，然后重启就解决了。

&emsp;&emsp;回看STRICT_TRANS_TABLES模式的特点，不允许任何未说明可以默认为空的字段插入空，就不难理解了。因为在airflow.cfg中，并未将默认添加example的配置置为false，于是就出现了在initdb时插入example的一幕；加上每个调度在产生时一定不会有last_pickled, last_expired这些值，所以就会出现错误。

&emsp;&emsp;如今airflow也更新到1.10.12了，相比于去年我还在用的1.10.4，只得感叹技术发展好快。

&emsp;&emsp;最后，聊一下原文作者漫谈的SQL设计之道——集合论。

&emsp;&emsp;阶（order）是用来区分集合或谓词的阶数的概念。谓词逻辑中，根据输入值的阶数对谓词进行分类。在SQL中，我们可以把行视为0阶，表（行组成的东西）视为1阶，数据库（表组成的东西）视为2阶。因此，像=这种谓词就是一阶谓词，EXISTS这样输入值为行的就称为二阶谓词。集合论是 SQL 语言的根基，因为它的这个特性，SQL 也被称为面向集合语言。

&emsp;&emsp;再看为什么聚合后不能再引用原表中的列，这是比如说cname是每个学生的属性，我们可以通过WHERE查询到行，也就是0阶，然后得到他的属性——班级。而GROUP BY，是将元素划分成多个子集并聚合，因此它关注的是多行，也可以理解为一个子表，是1阶的。因此，它获得的属性，应当是这么多行的属性，这个组的属性，在例子中就是，这个班级的平均成绩、最大学号等等。

&emsp;&emsp;在这里不得不感叹，SQL的等级分明，让每一个阶的操作得到每一个阶的属性，从概念上杜绝了混乱。

参考文献：
1.<https://mp.weixin.qq.com/s/3uh4K_njPdvkwKvzHxirtw>