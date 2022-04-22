---
typora-root-url: ../../../static
title: "LeetCode数据库(第二天)"
date: 2020-09-16T23:06:36+08:00
draft: false
categories: ["sql"]
---

## [178]分数排名
题目内容如下：

	//编写一个 SQL 查询来实现分数排名。
	//
	// 如果两个分数相同，则两个分数排名（Rank）相同。请注意，平分后的下一个名次应该是下一个连续的整数值。换句话说，名次之间不应该有“间隔”。
	//
	// +----+-------+
	//| Id | Score |
	//+----+-------+
	//| 1  | 3.50  |
	//| 2  | 3.65  |
	//| 3  | 4.00  |
	//| 4  | 3.85  |
	//| 5  | 4.00  |
	//| 6  | 3.65  |
	//+----+-------+
	//
	//
	// 例如，根据上述给定的 Scores 表，你的查询应该返回（按分数从高到低排列）：
	//
	// +-------+------+
	//| Score | Rank |
	//+-------+------+
	//| 4.00  | 1    |
	//| 4.00  | 1    |
	//| 3.85  | 2    |
	//| 3.65  | 3    |
	//| 3.65  | 3    |
	//| 3.50  | 4    |
	//+-------+------+
	//
	//
	// 重要提示：对于 MySQL 解决方案，如果要转义用作列名的保留字，可以在关键字之前和之后使用撇号。例如 `Rank`
	// 👍 600 👎 0

### 解法
一开始想，mysql会不会有什么系统函数可以得到当前行的索引，这样就可以得到排名了，好的，mysql低版本是没有这种函数的，高版本有一个dense_rank()函数可以直接得到排名。但是我本地是低版本的，于是。。。我不会做了，看人家的题解叭=-='''思路就是： **假设当前分数是X，那么查找大于X的不重复个数（平分的人是同个名次）G，那么G+1就是分数X的名次** ，代码如下：

	select Score,(
	        select count(distinct Score)+1 from Scores a where a.Score>b.Score
	) as `Rank` from Scores b order by Score desc;

## [180]连续出现的数字
题目内容如下：

	//编写一个 SQL 查询，查找所有至少连续出现三次的数字。
	//
	// +----+-----+
	//| Id | Num |
	//+----+-----+
	//| 1  |  1  |
	//| 2  |  1  |
	//| 3  |  1  |
	//| 4  |  2  |
	//| 5  |  1  |
	//| 6  |  2  |
	//| 7  |  2  |
	//+----+-----+
	//
	//
	// 例如，给定上面的 Logs 表， 1 是唯一连续出现至少三次的数字。
	//
	// +-----------------+
	//| ConsecutiveNums |
	//+-----------------+
	//| 1               |
	//+-----------------+
	//
	// 👍 305 👎 0

### 解法
至少连续出现三次，那么这就是连续的三行，那么它们的id也是连续的且num相等，根据这个条件，构造查询如下：

	select distinct a.Num as ConsecutiveNums from Logs a,Logs b,Logs c
	        where a.Id+1=b.Id and b.Id+1=c.Id
	            and a.Num=b.Num and b.Num=c.Num;

## [181]超过经理收入的员工
题目内容如下：

	//Employee 表包含所有员工，他们的经理也属于员工。每个员工都有一个 Id，此外还有一列对应员工的经理的 Id。
	//
	// +----+-------+--------+-----------+
	//| Id | Name  | Salary | ManagerId |
	//+----+-------+--------+-----------+
	//| 1  | Joe   | 70000  | 3         |
	//| 2  | Henry | 80000  | 4         |
	//| 3  | Sam   | 60000  | NULL      |
	//| 4  | Max   | 90000  | NULL      |
	//+----+-------+--------+-----------+
	//
	//
	// 给定 Employee 表，编写一个 SQL 查询，该查询可以获取收入超过他们经理的员工的姓名。在上面的表格中，Joe 是唯一一个收入超过他的经理的员工。
	//
	//
	// +----------+
	//| Employee |
	//+----------+
	//| Joe      |
	//+----------+
	//
	// 👍 308 👎 0

### 解法
通过Id和ManagerId连接起来两条记录，然后比较其Salary，代码如下：

	select a.Name as Employee from Employee a,Employee b where a.ManagerId=b.Id and a.Salary>b.Salary;

## [182]查找重复的电子邮箱
题目内容如下：

	//编写一个 SQL 查询，查找 Person 表中所有重复的电子邮箱。
	//
	// 示例：
	//
	// +----+---------+
	//| Id | Email   |
	//+----+---------+
	//| 1  | a@b.com |
	//| 2  | c@d.com |
	//| 3  | a@b.com |
	//+----+---------+
	//
	//
	// 根据以上输入，你的查询应返回以下结果：
	//
	// +---------+
	//| Email   |
	//+---------+
	//| a@b.com |
	//+---------+
	//
	//
	// 说明：所有电子邮箱都是小写字母。
	// 👍 229 👎 0

### 解法
比较简单，就是一个 `group by` ，需要注意的就是聚合函数可以放在 `having` 后面作为条件，代码如下：

	select Email from Person group by Email having count(*)>1;

## [183]从不订购的客户
题目内容如下：

	//某网站包含两个表，Customers 表和 Orders 表。编写一个 SQL 查询，找出所有从不订购任何东西的客户。
	//
	// Customers 表：
	//
	// +----+-------+
	//| Id | Name  |
	//+----+-------+
	//| 1  | Joe   |
	//| 2  | Henry |
	//| 3  | Sam   |
	//| 4  | Max   |
	//+----+-------+
	//
	//
	// Orders 表：
	//
	// +----+------------+
	//| Id | CustomerId |
	//+----+------------+
	//| 1  | 3          |
	//| 2  | 1          |
	//+----+------------+
	//
	//
	// 例如给定上述表格，你的查询应返回：
	//
	// +-----------+
	//| Customers |
	//+-----------+
	//| Henry     |
	//| Max       |
	//+-----------+
	//
	// 👍 172 👎 0

### 解法
比较简单，用 `not in` 就可以啦，代码如下：

	select Name as Customers from Customers where Id not in(
	        select distinct CustomerId from Orders
	);

## [184]部门工资最高的员工
题目内容如下：

	//Employee 表包含所有员工信息，每个员工有其对应的 Id, salary 和 department Id。
	//
	// +----+-------+--------+--------------+
	//| Id | Name  | Salary | DepartmentId |
	//+----+-------+--------+--------------+
	//| 1  | Joe   | 70000  | 1            |
	//| 2  | Jim   | 90000  | 1            |
	//| 3  | Henry | 80000  | 2            |
	//| 4  | Sam   | 60000  | 2            |
	//| 5  | Max   | 90000  | 1            |
	//+----+-------+--------+--------------+
	//
	// Department 表包含公司所有部门的信息。
	//
	// +----+----------+
	//| Id | Name     |
	//+----+----------+
	//| 1  | IT       |
	//| 2  | Sales    |
	//+----+----------+
	//
	// 编写一个 SQL 查询，找出每个部门工资最高的员工。对于上述表，您的 SQL 查询应返回以下行（行的顺序无关紧要）。
	//
	// +------------+----------+--------+
	//| Department | Employee | Salary |
	//+------------+----------+--------+
	//| IT         | Max      | 90000  |
	//| IT         | Jim      | 90000  |
	//| Sales      | Henry    | 80000  |
	//+------------+----------+--------+
	//
	// 解释：
	//
	// Max 和 Jim 在 IT 部门的工资都是最高的，Henry 在销售部的工资最高。
	// 👍 287 👎 0

### 解法
一开始想到了 `IN` ，但是不知道可以 `IN` 多个字段，思路就是： **先查询出部门ID以及对应部门最高工资，然后查询Employee中部门ID和工资IN查询出来的结果（IN的字段要对应）** ，代码如下：

	select D.Name as Department,E.Name as Employee,E.Salary as Salary
	        from Department D,Employee E
	        where D.Id=E.DepartmentId and (E.DepartmentId,E.Salary) in (
	        select F.DepartmentId,max(F.Salary) from Employee F group by F.DepartmentId
	    );

## [175]组合两个表
题目内容如下：

	//表1: Person
	//
	// +-------------+---------+
	//| 列名         | 类型     |
	//+-------------+---------+
	//| PersonId    | int     |
	//| FirstName   | varchar |
	//| LastName    | varchar |
	//+-------------+---------+
	//PersonId 是上表主键
	//
	//
	// 表2: Address
	//
	// +-------------+---------+
	//| 列名         | 类型    |
	//+-------------+---------+
	//| AddressId   | int     |
	//| PersonId    | int     |
	//| City        | varchar |
	//| State       | varchar |
	//+-------------+---------+
	//AddressId 是上表主键
	//
	//
	//
	//
	// 编写一个 SQL 查询，满足条件：无论 person 是否有地址信息，都需要基于上述两表提供 person 的以下信息：
	//
	//
	//
	// FirstName, LastName, City, State
	//
	// 👍 667 👎 0

### 解法
就是一个左连接，代码如下：

	select FirstName,LastName,A.City,A.State from Person left join Address A on Person.PersonId = A.PersonId;

## [176]第二高的薪水
题目内容如下：

	//编写一个 SQL 查询，获取 Employee 表中第二高的薪水（Salary） 。
	//
	// +----+--------+
	//| Id | Salary |
	//+----+--------+
	//| 1  | 100    |
	//| 2  | 200    |
	//| 3  | 300    |
	//+----+--------+
	//
	//
	// 例如上述 Employee 表，SQL查询应该返回 200 作为第二高的薪水。如果不存在第二高的薪水，那么查询应返回 null。
	//
	// +---------------------+
	//| SecondHighestSalary |
	//+---------------------+
	//| 200                 |
	//+---------------------+
	//
	// 👍 637 👎 0

### 解法
返回第二高，想到用 `limit 1,1` ,但是题目多了一个需要返回null的条件，于是可以用 `IFNULL(条件,为null的结果)` 系统函数来处理。代码如下：

	select IFNULL((select distinct Salary from Employee order by Salary desc limit 1,1),null) as SecondHighestSalary;

## [177]第N高的薪水
题目内容如下：

	//编写一个 SQL 查询，获取 Employee 表中第 n 高的薪水（Salary）。
	//
	// +----+--------+
	//| Id | Salary |
	//+----+--------+
	//| 1  | 100    |
	//| 2  | 200    |
	//| 3  | 300    |
	//+----+--------+
	//
	//
	// 例如上述 Employee 表，n = 2 时，应返回第二高的薪水 200。如果不存在第 n 高的薪水，那么查询应返回 null。
	//
	// +------------------------+
	//| getNthHighestSalary(2) |
	//+------------------------+
	//| 200                    |
	//+------------------------+
	//
	// 👍 345 👎 0

### 解法如下：
这个题和上一个题[176]很类似，唯一需要注意的地方就是 `limit offset,count` 中的offset是从0开始的。代码如下：

	CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
	BEGIN
	    SET N=N-1;
	    RETURN (
	        # Write your MySQL query statement below.
	        IFNULL((select distinct Salary from Employee order by Salary desc limit N,1),null)
	    );
	END



