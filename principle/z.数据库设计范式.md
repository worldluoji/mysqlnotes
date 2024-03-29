# 数据库设计范式
数据库设计范式，简称“数据库范式”，也经常简称“范式”，英文为 Normal Form，简称 NF，
大部分情况是面向“关系型数据库”的设计规范。

## 三个范式

但实际开发工作中常用的就是前三个范式，第一范式、第二范式和第三范式。

第一范式，是指在关系型数据库的表中，每一个字段都是原子性的，也就是最小原子，不能再拆分。
比如，email和phone应该作为两个字段保存，而不应该合并到一个contact字段中。

第二范式，在第一范式的基础上新增了一个规则，“不存在非关键字段对任意一候选字段的部分函数依赖”
第二范式可以理解是“数据的唯一性和一致性”，通过“主键”，来关联对应详细数据，避免造成数据的冗余和不一致。

比如page表，通过user_id关联user表，page表中就不应该再出现email、phone等用户信息，
因为这时候，如果用户表更新了，就必须同步更新page表中的用户信息,造成冗余。

总之，有了冗余，就不满足第二范式。但是，一些特殊情况下，有少量的冗余也是合理的。

第三范式，在第二范式的基础上再新增了一个规则，“任何非主属性不依赖于其他非主属性”。
换句话讲，就是要“消除传递依赖”。比如user表：
```
id username department manager
1  张三      技术部      王麻子
2  王五      财务部      孙麻子
```
添加了员工部门“department”和员工主管“manager”的数据字段，
这个两个字段就是“非主属性”或“非主键”，存在依赖关系。应该改为：
```
id username department
1  张三      技术部
2  王五      财务部

id department manager
1  技术部      王麻子
2  财务部      孙麻子
```