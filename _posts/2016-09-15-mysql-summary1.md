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