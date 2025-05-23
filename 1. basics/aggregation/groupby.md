﻿# groupby

数据准备：
```sql
CREATE TABLE `student` (
  `ID` int(11) NOT NULL auto_increment,
  `STU_NUM` int(11) DEFAULT NULL,
  `GRADE` VARCHAR(30),
  `CLASS` VARCHAR(30),
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;

insert into student(STU_NUM, GRADE, CLASS) values(10001, 'gradeA', 'classA');
insert into student(STU_NUM, GRADE, CLASS) values(10002, 'gradeA', 'classA');
insert into student(STU_NUM, GRADE, CLASS) values(10003, 'gradeA', 'classC');
insert into student(STU_NUM, GRADE, CLASS) values(10004, 'gradeA', 'classA');
insert into student(STU_NUM, GRADE, CLASS) values(10005, 'gradeA', 'classB');
insert into student(STU_NUM, GRADE, CLASS) values(10006, 'gradeA', 'classB');
insert into student(STU_NUM, GRADE, CLASS) values(10007, 'gradeA', 'classB');
insert into student(STU_NUM, GRADE, CLASS) values(10007, 'gradeB', 'classA');
```

group by的常规用法是配合聚合函数，利用分组信息进行统计，常见的是配合max等聚合函数筛选数据后分析，以及配合having进行筛选后过滤。
```
select min(stu_num),class from student group by class;
+--------------+--------+
| min(stu_num) | class  |
+--------------+--------+
|        10001 | classA |
|        10003 | classC |
|        10005 | classB |
+--------------+--------+
```
即查询每种class中学号最小的那个，分类输出。

使用having则可以加入过滤条件
```
select min(stu_num),class from student group by class having class > 'classA'
+--------------+--------+
| min(stu_num) | class  |
+--------------+--------+
|        10003 | classC |
|        10005 | classB |
+--------------+--------+
```
结论:
- 当group by 与聚合函数配合使用时，功能为分组后计算
- 当group by 与having配合使用时，功能为分组后过滤
- 当group by 与聚合函数，同时非聚合字段同时使用时，非聚合字段的取值是第一个匹配到的字段内容，即id小的条目对应的字段内容。

---

## 实战：
1. leetcode586 订单最多的用户：

在表 orders 中找到订单数最多客户对应的 customer_number 。

数据保证订单数最多的顾客恰好只有一位。

表 orders 定义如下：
```
| Column            | Type      |
|-------------------|-----------|
| order_number (PK) | int       |
| customer_number   | int       |
| order_date        | date      |
| required_date     | date      |
| shipped_date      | date      |
| status            | char(15)  |
| comment           | char(200) |
```
实际上就是看那个customer_number的记录数目最多。

解答：
```sql
select customer_number from (
select customer_number,count(*) as counts from orders group by customer_number order by counts desc
) as temp limit 0,1
```
Mysql中产生的新的表必须用as写别名，否则会报错：
```
Every derived table must have its own alias
```

---

2. leetcode597
```sql
select IFNULL( CONVERT(
(select count(*) from 
(select requester_id,accepter_id from request_accepted group by requester_id,accepter_id) as tmp2)
    /
(select count(*) from 
(select sender_id,send_to_id from friend_request group by sender_id,send_to_id) as tmp1),DECIMAL(10,2))
,0.00) as accept_rate
```
IFNULL() 函数用于**判断第一个表达式是否为 NULL，如果为 NULL 则返回第二个参数的值，如果不为 NULL 则返回第一个参数的值**。

保留小数位：

这个是保留整数位
```sql
SELECT CONVERT(4545.1366,DECIMAL);
```

这个是保留两位小数    
```sql
SELECT CONVERT(4545.1366,DECIMAL(10,2));
```

这个是截取两位，并不会四舍五入保留两位小数
```
SELECT TRUNCATE(4545.1366,2);
```

实际使用distinct速度更快
```sql
select IFNULL( CONVERT(
(select count(*) from 
(select distinct requester_id,accepter_id from request_accepted) as tmp2)
    /
(select count(*) from 
(select distinct sender_id,send_to_id from friend_request) as tmp1),DECIMAL(10,2))
,0.00) as accept_rate
```

---

练习：
leetcode 511 
```sql
select player_id, min(event_date) as first_login
from Activity group by player_id
```

leetcode 1327
```sql
select p.product_name,o.unit 
from Products p 
left join 
(select t.product_id,t.order_date, sum(t.unit) as unit from Orders t where Year(t.order_date) = 2020 and Month(t.order_date) = 2 group by t.product_id ) o
on p.product_id=o.product_id
where o.unit >= 100
```

注意： 
- 1）where要在group by之前； **where是先过滤再分组；having是先分组再过滤**
- 2）日期的处理

如果你要查询2013年1月份加入的产品呢？代码如下:
```sql
select * from product where Date(add_time) between '2013-01-01' and '2013-01-31'
```
你还可以这样写：
```sql
select * from product where Year(add_time) = 2013 and Month(add_time) = 1     
```