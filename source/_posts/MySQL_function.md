---
title: Mysql function应用
categories:
  - [DB,MySQL]
---



# 自连接应用

类似排名这种逻辑，可以用自连接来做。比如   `某学生在班级中某科目的分数排名`   ，自连接的时候用count()看比自己分高的同学的数量即可得到。要注意   **distinct** 去重。

> 应用：leetcode [185. 部门工资前三高的所有员工](https://leetcode-cn.com/problems/department-top-three-salaries/)

```mysql
select d.Name Department, e.Name Employee, e.Salary Salary
from Employee e left join Department d
on e.DepartmentId = d.Id
where e.id in (
    select e1.Id from Employee e1 left join Employee e2 
    on e1.DepartmentId = e2.DepartmentId and e1.Salary < e2.Salary
    group by e1.id
    having count(distinct e2.Salary) < 3
)
and e.DepartmentId in (select DepartmentId from Department)

-- 你测试用例来个Department表是空的可太秀了  淦
```

`看第一个评论学到的`



# 数字相关函数

`trancate(表达式，保留n位小数)`    不会四舍五入

`round(表达式，保留n位小数)`    会四舍五入

> 应用：leetcode [262. 行程和用](https://leetcode-cn.com/problems/trips-and-users/)

```mysql
select Request_at as Day, 
	round(count(if(Status != 'completed', Status, null)) / count(Status) , 2) as "Cancellation Rate"
from Trips t
	inner join Users u on t.Client_id = u.Users_id and u.Banned = "No"
	inner join Users u2 on t.Driver_id = u2.Users_id and u2.Banned = "No"
and Request_at < "2013-10-04" and Request_at >= "2013-10-01"
group by Request_at
```





# 总结MySQL查询的一般性思路：

- 能用单表优先用单表，即便是需要用group by、order by、limit等，效率一般也比多表高

- 不能用单表时优先用连接，连接是SQL中非常强大的用法，小表驱动大表+建立合适索引+合理运用连接条件，基本上连接可以解决绝大部分问题。但join级数不宜过多，毕竟是一个接近指数级增长的关联效果

- 能不用子查询、笛卡尔积尽量不用，虽然很多情况下MySQL优化器会将其优化成连接方式的执行过程，但效率仍然难以保证

- 自定义变量在复杂SQL实现中会很有用，例如LeetCode中困难级别的数据库题目很多都需要借助自定义变量实现

- 如果MySQL版本允许，某些带聚合功能的查询需求应用窗口函数是一个最优选择。除了经典的获取3种排名信息，还有聚合函数、向前向后取值、百分位等，具体可参考官方指南。以下是官方给出的几个窗口函数的介绍：

![image.png](mysql8.0-window%20functions.png)


最后的最后再补充一点，本题将查询语句封装成一个自定义函数并给出了模板，实际上是降低了对函数语法的书写要求和难度，而且提供的函数写法也较为精简。然而，自定义函数更一般化和常用的写法应该是分三步：

定义变量接收返回值
执行查询条件，并赋值给相应变量
返回结果