---
layout: post
title: "开发遇到过的MYSQL坑"
date: 2017-09-03 22:49:03 +0800
categories: 基础
tags: mysql
---

* TOC
{:toc}

在Java开发中，数据存储大多使用的是MySQL，这里记录下开发过程中使用MySQL出现的问题，总结教训，避免再次出现问题。

# 金额表示

今天又掉了个坑，是个小坑，从数据库中取出的金额跟实际不符，查看数据表的字段类型的时候发现问题：

~~~
`pay_once` float(10,3) NOT NULL DEFAULT '0.000' COMMENT '接口花费'
~~~

但是在java中，float是不能表示精确的小数的，所以在表示金额的时候应该使用`DECIMAL`，对应到Java中的类型就是`BigDecimal`。

依旧是一个应该形成规范的问题



# NULL

今天发布遇到的问题，在取数据的时候，判断值是否为0来进行筛选。上线之后发现筛选得到的数据数量不对。仔细检查sql没有问题，查看筛选数据的时候才发现这个字段绝大部分都是NULL。在建表语句中，这个字段定义为：`tinyint(4) default 0 comment '..'`，但实际在插入数据的时候直接插入了NULL。而NULL只能通过is来做判断，于是导致了数据漏掉的情况。这个问题其实是Mysql使用不规范，但作为开发，还是要考虑到其他人的各种操作不规范的情况，永远不要指望别人照着自己的标准去做，只有这样才能避免频繁踩到别人的坑。

# CASE-WHEN-THEN

在使用CASE-WHEN做字段的值转换的时候，如果没有else做最后赋值，在条件均不满足的情况下，mysql会以null赋值给选定的字段。如果不希望字段被赋值为null的话，最好加上在最后加上else做一个对未列出条件的补充。

~~~
SELECT CASE WHEN T.column_name='3'  THEN '3' WHEN T.column_name='4' THEN '4' ELSE '2' END AS columnName FROM table T
~~~

# join on

首先看两个sql

`select * from table1 left join table2 on table1.column1=table2.column2 and table2.column3=#{value1} where table1.column1=#{value2}`
`select * from table1 left join table2 on table1.column1=table2.column2 where table1.column1=#{value2} and table2.column3=#{value1}`

在使用外联结时，以左联结为例，在查询时，会依次拿左表中的数据，根据关联字段column1，去table2中找符合条件的数据，即table2.column2和column1相等，并且table2.column3=value1，然后再执行where条件
所以在查询的时候一定要清楚各个关键词的优先级，查询的先后顺序不同，最终查的的结果也不同。

# in

看一个MySQL的查询

`select id from table1 where column1='value'`
`select * from table where id in ();`

对应的java代码就是

`List<Long>ids=queryIdsByCondition();`
`List<Object>items=queryByIds(ids);`

问题就在于当ids为空的时候，在做第二个查询前未判断是否为空，而是直接做查询的时候就会出现sql异常。这个问题排查起来很简单，只是容易疏忽造成问题，所以需要养成习惯，在需要传递一个List到一个MySQL的查询语句的时候，必须加上判断逻辑。

# union，union all

最近做的一个需求，需要根据查询条件的不同查询不同的表，然后把各个表查的的结果组合在一起后分页返回。看了下现有的代码，是写了三个sql语句做分别查询，查询完成之后放在一个List中，然后subList返回查的的结果数据。其实这里完全可以用MySQL自带的union关键字

首先区分下union和union all的区别

这两个都是把查的的结果组合在一起，但区别在于union会合并重复的数据，而union all不会做合并操作

具体用法很简单，并且可以对一个单独的select添加order by和limit，也可以对组合后的所有数据做order by和limit


# charset

今天出现的一个奇怪的问题，MySQL按照一个类型是varchar的字段排序，这个字段有'10001002710030005245'这种纯数字的，也有'ANS201603020283'这种以字母开头的，那么通过orderby正序排列的时候，我期待的是所有纯数字的排在最前边，但是事实是：  

![](/_pic/201709/Screenshot from 2017-09-07 10-38-20.png)

![](/_pic/201709/Screenshot from 2017-09-07 10-38-59.png)

把最后那条诡异的数据的LOAN_NO手工输入，更新后，再次正序排列，这条数据就成为了

![](/_pic/201709/Screenshot from 2017-09-07 10-52-53.png)  

百度后，发现了一个解决方案：`SELECT * FROM {$tableName} order by  CONVERT(${columnName} USING gbk);`  

如此排序问题解决。  

在我的建表语句中已经设定了charset=utf8，不太明白为什么数据库中会存在其他编码的字符。请教了下DBA同学，有一个可能是输入时强制指定了字符编码，且指定的字符编码不为UTF8.  

不管这个问题是怎样产生的，一个通用的解决方案是，所有的数据库使用统一的编码格式，即目前通用的UTF8，或者UTF8MB4就可以避免这些诡异的编码问题  

另外需要注意的是，这种转换字符之后做排序的做法不适用于BigDecimal和Int等类型。

# alias  

今天线上sql查询数据时sql报错，看下报错的sql：

~~~
<select id="queryData">
    SELECT <include refid="columns"></include>
    FROM table1 t1
    <if test="condition1">
        INNER JOIN table2 t2 ON t1.column1 = t2.column1
    </if>
    <if test="condition2">
        INNER JOIN table3 t2 ON t1.column1 = t2.column1
    </if>  
    LEFT JOIN table4 t4 ON t1.column1 = t4.column1   
</select>
~~~

错误日志：

~~~
### Error querying database.  Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Not unique table/alias: 'kr'
### The error may exist in dao.xml
### The error may involve defaultParameterMap
### The error occurred while setting parameters
### Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Not unique table/alias: 'kr'
~~~

在MySQL中，对数据表的别名是不能够重复的，否则就会出现上边的问题，所以在mybatis中使用判断条件的时候，要注意判断条件的情况
