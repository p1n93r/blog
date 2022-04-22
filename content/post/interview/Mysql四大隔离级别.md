---
typora-root-url: ../../../static
title: "Mysql四大隔离级别"
date: 2020-06-25T16:04:36+08:00
categories: ["Mysql"]
draft: false
---

## 事务
### 什么是事务
事务是由一个有限的数据库操作序列构成，这些操作要么全部执行，要么全部不执行。是一个不可分割的工作单位。

### 事务的四大特性
- 原子性(Atomicity)：事物是一个不可分割的原子项，要么全都执行，要么全都不执行。
- 一致性(Consistency)：事物开始前和事物结束后，数据的完整性没有被破坏。
- 隔离性(Isolation)：多个事物并发访问时，事物与事物之间是互不干扰的。
- 持久性(Durabilily)：事物执行完后对数据的修改是持久化的。

## 事务并发存在的问题
假设现在存在如下表：

	CREATE TABLE `account` (
		`id` int(11) NOT NULL,
		`name` varchar(255) DEFAULT NULL,
		`balance` int(11) DEFAULT NULL, 
		PRIMARY KEY (`id`), 
		UNIQUE KEY `un_name_idx` (`name`) USING BTREE 
	) ENGINE=InnoDB DEFAULT CHARSET=UTF8; 

表中有如下数据：

|id|name|balance|
|---|----|-------|
|1|Jay|100|
|2|Eason|100|
|3|Lin|100|

### 脏读
假设现在存在两个事务A、B，事务A正准备查询Jay的余额，这时候B先扣除Jay的余额，扣了10，最后A读取到的是扣减后的余额：

|时间编号|事务A|事务B|
|-------|-----|-----|
|①|Begin||
|②||Begin|
|③||update account set balance=balance-10 where name='Jay'|
|④|select balance from account where name='Jay'||
|⑤|读取到Jay的balance为100-10=90||

事务A、B交替执行，事务A被事务B干扰了，因为事务A读取到事务B未提交的数据，这就是 **脏读** 。

### 不可重复读
假设现在有两个事务A和B，事务A先查询Jay的余额，查询结果是100，这时候事务B对Jay的账户余额进行减扣，扣去10，提交事务，事务A再去查询Jay的余额发现变成了90：

|时间编号|事务A|事务B|
|-------|-----|-----|
|①|Begin||
|②|select balance from account where name='Jay';||
|③|读取到Jay的balance为100||
|④||Begin|
|⑤||update accout set balance=balance-10 where name='Jay';|
|⑥||Commit|
|⑦|select balance from account where name='Jay';||
|⑧|读取到Jay的balance为100-10=90||
|⑨|Commit||

事务A被事务B干扰了，在事务A范围内，两个相同的查询，读取用一条记录，却返回了不同的数据，这就是 **不可重复读** 。

### 幻读
假设存在两个事务A、B，事务A先查询id大于等于2的记录，得到id=2和id=3的两条记录，这时候事务B开启，插入一条id=4的记录，并且提交了，事务A再去执行相同的查询，却得到了id=2，3，4的三条记录：

|时间编号|事务A|事务B|
|-------|-----|-----|
|①|Begin||
|②|select * from account where id>=2;||
|③|返回id=2，3的两条记录||
|④||Begin|
|⑤||insert into account value(4,'Yan',100);|
|⑥||Commit|
|⑦|select * from account where id>=2;;||
|⑧|返回id=2，3，4的三条记录||

事务A查询一个 **范围结果集** ，另一个并发事务B往这个范围中插入/删除了数据，并提交了，然后事务A再去查询相同的范围，两个读取到的结果集不一样了，这就是 **幻读** 。

## 事务的四大隔离级别实践
并发事务存在 **脏读、幻读、不可重复读** 等问题，InnoDB实现了： **读未提交（Read-Uncommitted）、读已提交（Read-Committed）、可重复读（Repeatable-Read）和串行化（Serializable）** 。

### 读未提交
在读未提交的事务隔离级别下，一个事务会读取到其他事务未提交的数据，存在 **脏读** 的问题，是隔离级别最低的一种。下面进行测试，首先设置两个窗口的事务隔离级别未读未提交，然后进行测试：

![设置读未提交的事务隔离级别][p0]

<center><strong>设置读未提交的事务隔离级别</strong></center>

![测试读未提交的问题][p1]

<center><strong>测试读未提交的问题</strong></center>

### 已提交读
为了避免脏读，出现了已提交读（但还是会存在幻读的问题）。下面将事务隔离级别设置为已提交读，再次进行测试：

![已提交读隔离级别下解决脏读][p2]

<center><strong>已提交读隔离级别下解决脏读</strong></center>

### 可重复读
可重复读指的是同一个事务内多次相同条件的查询，数据的结果不变。根据上面的测试，已提交读可以读取到其他事务已提交的数据，则同个事务内，相同的查询条件得到了不同的查询结果，由此造成了不可重读，为了解决这个问题，出现了可重复读的事务隔离级别。测试如下：

![可重读][p3]

<center><strong>可重读隔离级别解决不可重读的问题</strong></center>

可重读可以解决不可重读的问题，那么是否可以解决幻读呢？先说结论： **如果同一个事务当中使用了update语句（update添加的记录），那么还是会出现幻读的问题，反之不会** 。下面进行测试：

![不可重读隔离级别下的幻读现象][p4]

<center><strong>不可重读隔离级别下的幻读现象</strong></center>

***Notice:*** mysql可重复读隔离级别并不完全解决了幻读的问题，而是解决了读数据情况下的幻读问题，而对于修改操作依然存在幻读问题。

### 串行化
串行化是隔离级别中最严格的，它保证了最好的安全性。它要求事务序列化执行，事务只能一个接着一个地执行，不能并发执行。下面进行的测试当中，右边的事务进行写入数据时候，会等待左边的事务提交：

![串行化隔离级别效果演示][p5]

<center><strong>串行化隔离级别效果演示</strong></center>

















[p0]:/media/2020-07-08-1.png
[p1]:/media/2020-07-08-2.png
[p2]:/media/2020-07-08-3.png
[p3]:/media/2020-07-08-4.png
[p4]:/media/2020-07-08-5.png
[p5]:/media/2020-07-08-6.png