---
layout: post
title: mysql summary
categories: [mysql]
description: mysql
keywords: mysql
---

## Leetcode 题目
测试可以用 docker 里面的 mysql

### combine two table
 
[链接](https://leetcode.com/problems/combine-two-tables/)

```sql
select FirstName, LastName, City, State
from Person left join Address
on Person.PersonId = Address.PersonId
```

### second highest salary

[题目](https://leetcode.com/problems/second-highest-salary/)

```sql
select max(Salary)
from Employee
where Salary < (
    select max(Salary)
    from Employee
)
```

### Nth highest salary

count = N-1 不太了解, 留下来后面分析吧

[题目](https://leetcode.com/problems/nth-highest-salary/)

```
SELECT e1.Salary
FROM (SELECT DISTINCT Salary FROM Employee) e1
WHERE (SELECT COUNT(*) FROM (SELECT DISTINCT Salary FROM Employee) e2 WHERE e2.Salary > e1.Salary) = N - 1      
      
LIMIT 1
```

### Rank Score

[题目](https://leetcode.com/problems/rank-scores/)

这道题可以出一些变形题, 比如允许有 hole 

考虑 sql 的执行顺序

```sql
select Score, 
	(select count(distinct Score) From Scores where s.`Score` <= Scores.score) as Rank
from Scores as s
order by Score desc;
```

### Consecutive Numbers

[question](https://leetcode.com/problems/consecutive-numbers/)

```
Select DISTINCT l1.Num from Logs l1, Logs l2, Logs l3 
where l1.Id=l2.Id-1 and l2.Id=l3.Id-1 
and l1.Num=l2.Num and l2.Num=l3.Num
```

很奇特的一道题

### Employees Earning More Than Their Managers

[question](https://leetcode.com/problems/employees-earning-more-than-their-managers/)

```sql
select e1.Name as Employee
from Employee as e1 left join Employee as e2 on e1.ManagerId = e2.Id
where e1.Salary > e2.Salary
```

### Duplicate Emails

[question](https://leetcode.com/problems/duplicate-emails/)

```sql
select Email
from Person
group by Email
having count(Email) > 1;
```

### Customer who never order

[question](https://leetcode.com/problems/customers-who-never-order/)

```sql
select Name as Customers
from Customers
where Id not in (
	select Customers.`Id`
	from Customers join Orders
	where Customers.`Id` = Orders.`CustomerId`
);
```

### Department Highest Salary

[question](https://leetcode.com/problems/department-highest-salary/)

三种解法

```sql
SELECT D.Name AS Department ,E.Name AS Employee ,E.Salary 
FROM
	Employee E,
	(SELECT DepartmentId,max(Salary) as max FROM Employee GROUP BY DepartmentId) T,
	Department D
WHERE E.DepartmentId = T.DepartmentId 
  AND E.Salary = T.max
  AND E.DepartmentId = D.id

SELECT D.Name,A.Name,A.Salary 
FROM 
	Employee A,
	Department D   
WHERE A.DepartmentId = D.Id 
  AND NOT EXISTS 
  (SELECT 1 FROM Employee B WHERE B.Salary > A.Salary AND A.DepartmentId = B.DepartmentId) 

SELECT D.Name AS Department ,E.Name AS Employee ,E.Salary 
from 
	Employee E,
	Department D 
WHERE E.DepartmentId = D.id 
  AND (DepartmentId,Salary) in 
  (SELECT DepartmentId,max(Salary) as max FROM Employee GROUP BY DepartmentId)
```

### Department Top Three Salaries
[question](https://leetcode.com/problems/department-top-three-salaries/)

这道题没怎么看明白, 感觉没有答案这么复杂

```sql
select D.Name as Department, t.Name as Employee, t.Salary as Salary
from
(
    select *,
           (
            select count(distinct DepartmentId, Salary) + 1
            from Employee E2
            where E2.DepartmentId = E1.DepartmentId and E2.Salary > E1.Salary
           ) as rank
    from Employee E1
) t, Department D
where t.rank in (1, 2, 3) and t.DepartmentId = D.Id
```

### Delete Duplicate Emails

[question](https://leetcode.com/problems/delete-duplicate-emails/)

```sql
DELETE p1
FROM Person p1, Person p2
WHERE p1.Email = p2.Email AND
p1.Id > p2.Id
```

### Rising Temperature

[question](https://leetcode.com/problems/rising-temperature/)

```sql
select a.Id as Id
from Weather a, Weather b
where to_days(a.Date) = to_days(b.Date) + 1 and a.Temperature > b.Temperature;
```

### leetcode 总结

> 有 ID 的表, 首先考虑能否用两张表完成工作, 因为他们之间可用 ID 比较

> groupby 不能和 where 一起用, having 的作用很有限。但是 groupby 之后可以用 max 作为其他操作的参考

## SQL 语句的执行顺序

## 和名次相关的 query

## 知识点

### join, left join, inner join

### 索引

```sql
ALTER TABLE article ADD INDEX index_article_title ON title(200);
```

索引是一种特殊的文件(InnoDB数据表上的索引是表空间的一个组成部分)，它们包含着对数据表里所有记录的引用指针。
更通俗的说，数据库索引好比是一本书前面的目录，能加快数据库的查询速度。上述SQL语句，在没有索引的情况下，数据库会
遍历全部200条数据后选择符合条件的；而有了相应的索引之后，数据库会直接在索引中查找符合条件的选项。如果我们把
SQL语句换成“SELECT * FROM article WHERE id=2000000”，那么你是希望数据库按照顺序读取完200万行数据以后给
你结果还是直接在索引中定位呢？上面的两个图片鲜明的用时对比已经给出了答案（注：一般数据库默认都会为主键生成索引）。

```sql
– 直接创建索引
CREATE INDEX index_name ON table(column(length))

–修改表结构的方式添加索引
ALTER TABLE table_name ADD INDEX index_name ON (column(length))

–创建表的时候同时创建索引
CREATE TABLE `table` (
`id` int(11) NOT NULL AUTO_INCREMENT ,
`title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
`content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
`time` int(10) NULL DEFAULT NULL ,
PRIMARY KEY (`id`),
INDEX index_name (title(length))

–删除索引
DROP INDEX index_name ON table
```

### 唯一索引

与普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值（注意和主键不同）。如果是组合索引，则列值的组合必须唯一，创建方法和普通索引类似。

```sql
–创建唯一索引
CREATE UNIQUE INDEX indexName ON table(column(length))

–修改表结构
ALTER TABLE table_name ADD UNIQUE indexName ON (column(length))

–创建表的时候直接指定
CREATE TABLE `table` (
`id` int(11) NOT NULL AUTO_INCREMENT ,
`title` char(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL ,
`content` text CHARACTER SET utf8 COLLATE utf8_general_ci NULL ,
`time` int(10) NULL DEFAULT NULL ,
PRIMARY KEY (`id`),
UNIQUE indexName (title(length))
);
```

### 单列索引、多列索引

多个单列索引与单个多列索引的查询效果不同，因为执行查询时，MySQL只能使用一个索引，会从多个索引中选择一个限制最为严格的索引。

### 组合索引（最左前缀）

平时用的SQL查询语句一般都有比较多的限制条件，所以为了进一步榨取MySQL的效率，就要考虑建立组合索引。
例如上表中针对title和time建立一个组合索引：ALTER TABLE article ADD INDEX index_titme_time (title(50),time(10))。
建立这样的组合索引，其实是相当于分别建立了下面两组组合索引:

–title,time

–title

为什么没有time这样的组合索引呢？这是因为MySQL组合索引“最左前缀”的结果。简单的理解就是只从最左面的开始组合。并不是只要包含这两列的查询都会用到该组合索引，如下面的几个SQL所示：

```
–使用到上面的索引
SELECT * FROM article WHREE title='测试' AND time=1234567890;
SELECT * FROM article WHREE utitle='测试';

–不使用上面的索引
SELECT * FROM article WHREE time=1234567890;
```

上面都在说使用索引的好处，但过多的使用索引将会造成滥用。因此索引也会有它的缺点：虽然索引大大提高了查询速度，同时却会降低更新表的速度，
如对表进行INSERT、UPDATE和DELETE。因为更新表时，MySQL不仅要保存数据，还要保存一下索引文件。建立索引会占用磁盘空间的索引文件。
一般情况这个问题不太严重，但如果你在一个大表上创建了多种组合索引，索引文件的会膨胀很快。索引只是提高效率的一个因素，如果你的MySQL有
大数据量的表，就需要花时间研究建立最优秀的索引，或优化查询语句。下面是一些总结以及收藏的MySQL索引的注意事项和优化方法。

:复合索引的使用原则是第一个条件应该是复合索引的第一列,依次类推,否则复合索引不会被使用 

所以,正常情况下复合索引不能替代多个单一索引 

:如果一个表中的数据在查询时有多个字段总是同时出现则这些字段就可以作为复合索引,形成索引覆盖可以提高查询的效率 

:建立索引的目的就是帮助查询,如果查寻用不到则索引就没有必要建立,另外如果数据表过大(5w以上)则有些字段(字符型长度超过(40))不适合作为索引,另外如果表是经常需要更新的也不适合做索引

### 何时使用聚集索引或非聚集索引

事实上，我们可以通过前面聚集索引和非聚集索引的定义的例子来理解上表。如：返回某范围内的数据一项。比如
您的某个表有一个时间列，恰好您把聚合索引建立在了该列，这时您查询2004年1月1日至2004年10月1日之
间的全部数据时，这个速度就将是很快的，因为您的这本字典正文是按日期进行排序的，聚类索引只需要找到要检索的
所有数据中的开头和结尾数据即可；而不像非聚集索引，必须先查到目录中查到每一项数据对应的页码，然后再根据页
码查到具体内容。其实这个具体用法我还不是很理解，只能等待后期的项目开发中慢慢学学了。

### 索引不会包含有NULL值的列

只要列中包含有NULL值都将不会被包含在索引中，复合索引中只要有一列含有NULL值，那么这一列对于此复合索引就是无效的。所以我们在数据库设计时不要让字段的默认值为NULL。

### 使用短索引

对串列进行索引，如果可能应该指定一个前缀长度。例如，如果有一个CHAR(255)的列，如果在前10个或20个字符内，多数值是惟一的，那么就不要对整个列进行索引。短索引不仅可以提高查询速度而且可以节省磁盘空间和I/O操作。

### 索引列排序

MySQL查询只使用一个索引，因此如果where子句中已经使用了索引的话，那么order by中的列是不会使用索引的。因此数据库默认排序可以符合要求的情况下不要使用排序操作；尽量不要包含多个列的排序，如果需要最好给这些列创建复合索引。

### like语句操作

一般情况下不鼓励使用like操作，如果非使用不可，如何使用也是一个问题。like “%aaa%” 不会使用索引而like “aaa%”可以使用索引。

### 不要在列上进行运算

例如：select * from users where YEAR(adddate)<2007，将在每个行上进行运算，这将导致索引失效而进行全表扫描，
因此我们可以改成：select * from users where adddate<’2007-01-01′。关于这一点可以围观：一个单引号引发的MYSQL性能损失。

最后总结一下，MySQL只对一下操作符才使用索引：<,<=,=,>,>=,between,in,以及某些时候的like(不以通配符%或_开头的情形)。而理论上每张表里面最多可创建16个索引，不过除非是数据量真的很多，否则过多的使用索引也不是那么好玩的，比如我刚才针对text类型的字段创建索引的时候，系统差点就卡死了。

 

例如：select * from users where YEAR(adddate)<2007，将在每个行上进行运算，这将导致索引失效而进行全表扫描，因此我们
可以改成：select * from users where adddate<’2007-01-01′。关于这一点可以围观：一个单引号引发的MYSQL性能损失。

## 基本操作

```sql
create table messages (
     `id` INT(11) unsigned NOT NULL AUTO_INCREMENT,
     `message` TEXT,
     `create_time` datetime DEFAULT NULL,
     PRIMARY KEY (id)
) engine = InnoDB CHARSET = utf8

INSERT INTO `messages` (message, created_time)
```

```java
  private def rowToAlarmEscalteProcess(row: RowData): AlarmEscalateProcess = {
    new AlarmEscalateProcess(
      id = row("id").asInstanceOf[String],
      timestamp = row("timestamp").asInstanceOf[Timestamp],
      `type` = row("type").asInstanceOf[String],
      comment = row("comment").asInstanceOf[String],
```

[参考资料](http://blog.jobbole.com/86594/)