﻿# MySql
MySql是一种RDBMS(关系型)数据库， 由于其是开源的（oracle太贵），市场上的占有率也在不断提高。

1. 开启事务处理的主要两种方式：
- 1）begin 开始一个事务， rollback 回滚事务，commit 提交事务
- 2）SET AOTOCOMMIT=1 开启事务自动提交

补充Mysql事务的4个特性：
- 原子性:一个事务中的所有操作，要么全部完成，要么全部不完成。
- 一致性:事务开始和结束后，数据库完整性没有破坏。
- 隔离性:多个事务并发，数据不会因为并发而不一致。
- 持久性:事务结束后，对数据的修改是持久的。

<br>

2. char和varchar的区别
- char是固定长度，char中的n最大长度是255个字符, 如果是utf8编码方式， 那么char类型占255 * 3个字节，存储时字符数没有达到定义的位数，会后面补齐空格入库; 
- varchar是变长的（0-65535字节），没有达到定义的位数，也不会补齐。 

所以char速度更快，但是会浪费空间。

<br>

3. 存储过程
先编译，再执行。实际就是把sql语句编译后存储在数据库中，这样用户通过指定存储过程名字并给定参数即可执行。明显，存储过程有更快的速度，同时也符合组件化编程的思想。
使用方法详见create_procedure.sql. 
注意：
1）Mysql以";"为分隔符，如果没有声明分隔符，则编译器会将存储过程当作SQL语句处理，编译会报错，所以之前delimiter ;;，结束后再还原回delimiter ; 
2）如果有入参，就SET &p_in=1, CALL 存储过程名（@p_in）

<br>

4. 查看版本，字符集等
show variables like '%version%';
show variables like 'character_set%';

<br>

5. Mysql允许用关键字作为表名（最好不要）。
例如admin作为表明时，查询这张表要select * from `admin`;比较反人类

<br>

6. 连接Mysql: mysql -u 用户名 -p -h Mysql服务器所在主机。下面就会让输入密码。

<br>

7. 什么是STRAIGHT_JOING?
STRAIGHT_JOIN is similar to JOIN, except that the left table is always read before the right table. 
This can be used for those (few) cases for which the join optimizer puts the tables in the wrong order.

例子：
```
select * from t1 straight_join t2 on (t1.a=t2.a);
```
则t1是驱动表，t2是被驱动表，如果直接用inner join则编译器不一定按照这个顺序。

注意：STRAIGHT_JOIN只适用于inner join，并不适用于left join，right join。（因为left join，right join已经代表指定了表的执行顺序）
尽可能让优化器去判断，因为大部分情况下mysql优化器是比人要聪明的。

使用STRAIGHT_JOIN一定要慎重，因为啊部分情况下认为指定的执行顺序并不一定会比优化引擎要靠谱。

<br>

8. explain SQL语句 \G来分析一条SQL语句。 不加\G就是表格形，加了就是列表形式。

<br>

9. MySql中如果设置了主键值达到了上限，再insert语句是就会报主键冲突的错误，
因此建议尽量将主键类型设置为8个字节的 bigint unsigned（2^64-1）防止那么快达到上限.
如果没有设置primary key，那么mysql会自动生成一个自增的看不到的rowId, 其也是bitint unsigned，但是只用了6个字节。达到上限后就会覆盖原来的行，循环。

<br>

10.  update流程还涉及redo log和bin log两个重要的日志模块
MySql里说的WAL(Write-Ahead Logging)，关键点就是先写日志再写磁盘，对应的就是redo log.
- redo log是innodb特有的，而bin log在Service层，MyIAsam和InnoDb都有。
- redo log大小是固定的，从头写到尾又到头，是一个圈；bin log则是追加写，不会覆盖原来的。 
- redo log相当于一个“粉板”， 在执行更新操作的时候，先写到redo log（内存）里，等到空闲的时候再写到磁盘；
- bin log用来恢复数据，可以恢复记录的任意一秒的数据。

为什么用bin log恢复数据？因为前面已经说了，redo log是innnodb用来优化性能和保证crash-safe（即数据库异常重启，之前的提交也不会丢失），而且会循环写覆盖，而bin log是追加写的。
Mysql innodb在update时，会先记录redo log， 再记录bin log,最后提交事务，是一个典型的两阶段提交。

innodb_flush_log_at_trx_commit设置为1，表示redo log持久化到磁盘，这样即使MySql异常重启后数据也不会丢失；
```
mysql> show variables like 'innodb_flush_log_at_trx_commit'
    -> ;
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |
+--------------------------------+-------+
1 row in set (0.00 sec)
```
可以看到，在MySql 5.7中，默认就是1.

sync_binlog 这个参数设置成 1 表示每次事务的bin log都写到磁盘，则异常重启后bin log不会丢失。
```
mysql> show variables like 'sync_binlog';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| sync_binlog   | 1     |
+---------------+-------+
1 row in set (0.00 sec)
```

<strong>bin log是逻辑日志，redo log是物理日志</strong>。
逻辑日志可以给别的数据库，别的引擎使用，已经大家都讲得通这个“逻辑”；
物理日志就只有“我”自己能用，别人没有共享我的“物理格式”

innodb_flush_log_at_trx_commit取值补充：

innodb_flush_log_at_trx_commit = 0

Innodb 中的Log Thread 每隔1 秒钟会将log buffer中的数据写入到文件，同时还会通知文件系统进行文件同步的flush
操作，保证数据确实已经写入到磁盘上面的物理文件。但是，每次事务的结束（commit 或者是rollback）并不会触发Log Thread
将log buffer 中的数据写入文件。所以，当设置为0 的时候，当MySQL Crash 和OS Crash
或者主机断电之后，最极端的情况是丢失1 秒时间的数据变更。

innodb_flush_log_at_trx_commit = 1

这也是Innodb 的默认设置。我们每次事务的结束都会触发Log Thread 将log buffer
中的数据写入文件并通知文件系统同步文件。**这个设置是最安全的设置，能够保证不论是MySQL Crash 还是OS Crash
或者是主机断电都不会丢失任何已经提交的数据。**

innodb_flush_log_at_trx_commit = 2

当我们设置为2 的时候，Log Thread
会在我们每次事务结束的时候将数据写入事务日志，但是这里的写入仅仅是调用了文件系统的文件写入操作。
而我们的文件系统都是有缓存机制的，所以Log Thread的这个写入并不能保证内容真的已经写入到物理磁盘上面完成持久化的动作。
文件系统什么时候会将缓存中的这个数据同步到物理磁盘文件Log
Thread 就完全不知道了。所以，当设置为2 的时候，MySQL Crash 并不会造成数据的丢失，但是OS Crash
或者是主机断电后可能丢失的数据量就完全控制在文件系统上了。
各种文件系统对于自己缓存的刷新机制各不一样，大家可以自行参阅相关的手册。

根据上面三种参数值的说明，0的时候，如果mysql crash可能会丢失数据，可靠性不高。
我们着重测试1和2两种情况。1的时候会影响数据库写入性能，相对2而言写入速度会慢，这只能根据实际情况来决定吧。

<br>

11.  count(*)
MyISAM把一张表的记录总数记到了磁盘上，所以count(*) 查询时直接读取总数的记录就OK了，优点点是速度快，但是MyISAM不支持事务的。 
而InnoDB则是一条一条记录取数，目的是为了保证多版本控制（MVVC）时数据的正确性。

InnoDB count(*)性能优化：
- 1）可以单独一张表存放总数，增加时+1，删除时-1；如果使用redis等内存数据库，可能会统计不准确，要求不高时可以使用。
- 2）应该尽量使用count(*), Mysql Innodb做了优化，速度count(*)约等于count(1)>count(id)>count(字段) 3）. 
- 3）根据业务场景优化，比如统计A+B+C三次查询记录的总数，可以在查询记录时把数量一起返回给前端，前端再加起来。

12.  Mysql查看，修改隔离界别： select @@transaction_isolation; 默认值时可重复读。(Mariadb是 select @@session.tx_isolation;)

set session transaction isolation level （level取值如下）
- 1）read uncommitted : 能读取到尚未提交事务的数据 ：哪个问题都不能解决
- 2）read committed：能读取已经提交事务的数据 ：可以解决脏读 ---- oracle默认的
- 3）repeatable read：可重复读：可以解决脏读 和 不可重复读 ---mysql默认的，即与事务启动时看到的一致。
假设你在管理一个个人银行账户表。一个表存了账户余额，一个表存了账单明细。
到了月底你要做数据校对，也就是判断上个月的余额和当前余额的差额，是否与本月的账单明细一致。
你一定希望在校对过程中，即使有用户发生了一笔新的交易，也不影响你的校对结果。
这时候使用“可重复读”隔离级别就很方便。事务启动时的视图可以认为是静态的，不受其他事务更新的影响。
- 4）serializable：串行化：可以解决 脏读 不可重复读 和 幻读，对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。
当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

<br>

13. - 脏读：session A 看到了session B提交的数据(X->Y)，结果 B roll back了， A看到的Y就是脏数据。
    - 不可重复读：session A 看到session B提交事务（X->Y）前后的值不一样。
    - 幻读：同一个事务前后两次查询同一张表， 前后两次查询中间有插入操作，后一次查询出现了前一次查询没有的行。
    见幻读产生例子.png中的Q3，这里要说明一下，在默认可重复读级别下，普通查询时快照读时看不到别的事务下的新增的记录，
    只有在当前读（for update，读取到已经提交的记录的最新值）的情况下，才会出现幻读.
   
14.  select for update例子
```
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
  ) ENGINE=InnoDB;
insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);

   begin;
   select * from t where d=5 for update;
   commit;
```

select for update仅适用于InnoDB，且必须在事务区块(start sta/COMMIT)中才能生效。
select for update会加一个写锁， 写锁在commit提交事务后释放。

15. 理解索引
1）为什么使用索引

使用索引其实就是为了加速查询，就像书的目录一样，能够快速定位到我们想要的内容。

2）什么是聚簇索引、非聚簇索引和回表？
```
create table T(
id int not null primary key auto_increment, 
k int not null, 
name varchar(16),
index (k))engine=InnoDB;
```
id是主键，默认有主键索引，主键索引是“聚簇”的，就是说select * from T where id=5, 直接遍历主键id的索引树，就能把id,k,name都查询出来，就像id把k,name都聚集在一起了一样
k是普通索引，普通索引是非聚簇的，就是说select * from T where k=5那么会遍历k的索引树，找到k=5对应的id集合，再通过id索引树搜索一次找到name等其它信息，这个过程称为回表（或回行）

3）什么是页分裂，为什么说要尽量使用自增主键？

继续上面的例子，假设ID没有设置auto_increment且ID不连续,ids=(300,500,800)在一个数据页。
这时候插入一个ID为600的数据，如果该数据页未满，需要把800的数据往后移，再插入600的数据；如果该数据已满，根据B+树算法，就会发生页分裂，800会被移动到新申请的数据页上。
显然，页分裂会严重影响性能和空间利用率。其逆过程就是页合并。
设置了auto_increment之后，每次id都是自增1，就不会出现上述需要挪动位置的情况，不会触发叶子节点的分裂，因为每次插入一条数据都是追加操作。
另外，如果主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也越小。
当然也有适合其它业务字段做主键的情形，那就是典型的KV场景，即只有主键上有索引（且该索引是唯一索引）的情况。

4）为什么要重建普通索引？
```
alter table T drop index k;
alter table T add index(k);
```
因为重建普通索引可以节省空间。前面说过索引可能因为删除和页分裂等原因导致数据空洞。重建索引时会将数据按顺序插入，这样页的利用率高，数据更加紧凑。
反之如果是主键索引，重建就不合理，因为下面两个语句都会导致数据表的重建。替代语句是alter table T engine=InnoDB.
```
alter table T drop primary key;
alter table T add primary key(id);
```

5) 什么是索引覆盖？
```
select ID from T where k=5
```
由于ID已经在索引树上了，这句查询就不需要回表。这就是索引覆盖。

索引覆盖例子:
```
CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `ismale` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `id_card_name` (`id_card`，`name`),
  KEY `name_age` (`name`,`age`)
) ENGINE=InnoDB
select name from tuser where id_card='xxxxxxxxxx'
```
根据身份证号查询姓名时，联合索引就让我们使用上了索引覆盖，通过id_card的索引树就能直接拿到name,不需要回表了

6）什么是索引的最左前缀原则？

实际是利用了B+树的最左前缀原则。上面的name_age联合索引，在select * from tuser where name='xxx'也能使用到索引。 
但是select * from tuser where age=28就使用不到了。这就是左前缀原则。

所以通过建立name_age联合索引，就不用再为name单独建立索引了，可以少维护一个索引。
```
select name from tuser where name like '张%' and age=10
```
这样也能利用到索引，找到名字“张”起头，age=10的记录

7）什么是索引下推？
```
select * from tuser where name like '张 %' and age=10 and ismale=1;
```
还是只有name_age联合索引，在Mysql5.6及以后，引入了索引下推。
这句查询语句会先匹配name以张开头的且age=10的记录，再回表找id，并且找到ismale=1的记录；
5.6之前没有索引下推，在找到name以张开头的就会回表去查询，这样明显age != 10的也被回表查出来了。

8）下面的联合索引有必要吗？
```
CREATE TABLE `geek` (
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  `d` int(11) NOT NULL,
  PRIMARY KEY (`a`,`b`),
  KEY `c` (`c`),
  KEY `ca` (`c`,`a`),
  KEY `cb` (`c`,`b`)
) ENGINE=InnoDB;
```
业务中有查询语句
```
select * from geek where c=N order by a limit 1;
select * from geek where c=N order by b limit 1;
```
答：cb联合索引有必要，但是ca联合索引没有必要。
对于
select ... from geek where c=N order by a 
走ca,cb索引都能定位到满足c=N主键
而且主键的聚簇索引本身就是按order by a,b排序，无需重新排序。所以ca可以去掉
select ... from geek where c=N order by b 
这条sql如果只有c单个字段的索引，定位记录可以走索引，但是order by b的顺序与主键顺序不一致，需要额外排序
cb索引可以把排序优化调

9）分页优化
LIMIT 2 5 : 在原始结果集中，跳过 2 个记录行，并从 第 3 个记录行开始，最多返回 5 个记录行。
LIMIT 5 最多返回 5 个记录行，等效于 LIMIT 0 5
```
select * from ORDER_DMEO order by order_no limit 10000, 20; 速度慢
select * from ORDER_DMEO where order_no > (select order_no from ORDER_DMEO order by order_no limit 10000, 1) limit 20; 速度快
```
默认在order_no上加入了索引
第一个语句要扫描所有记录行，按照order_no排序，还要回表取出其它所有信息，再取出其中20行；
第二个语句先用子查询查出起始的那条记录，由于子查询里只查了order_no ，不需要回表，速度很快；外部查询就只需要查询20条记录了。
可以查看order_demo.sql，里面插入了12000行记录
通过explain分析：
```
explain select * from ORDER_DMEO order by order_no limit 10000, 20 \G;
           id: 1
  select_type: SIMPLE
        table: ORDER_DMEO
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 12222
     filtered: 100.00
        Extra: Using filesort
```
可以看到第一条语句使用到了filesort，即因为扫描行数太多（1222行），无法直接通过sort_buffer排序，所以用到了磁盘，用到了磁盘效率就大大降低了。

第二条语句，子查询用到了索引，而且只扫描了10001行
```
 explain select order_no from ORDER_DMEO order by order_no limit 10000, 1 \G
*************************** 1. row ***************************
           order_no : 1
  select_type: SIMPLE
        table: ORDER_DMEO
   partitions: NULL
         type: index
possible_keys: NULL
          key: order_no_index
      key_len: 147
          ref: NULL
         rows: 10001
     filtered: 100.00
        Extra: Using index
```
 最后外部查询只需要扫描20行即可。
 
<br>

16.  普通索引和唯一索引区别
1). 唯一索引要求字段不能重复，普通索引则没有这个限制。

2). 查询区别：
```
select id from T where k=5. 
```
- 普通索引，查找到下一个不满足k=5的才完成查询；
- 唯一索引：因为是唯一的，查到到k=5的后就退出

可以看到对于查询来说，普通索引查询会多一次判断，但性能影响不大。

3). 更新。对于更新来说，当更新一个数据页时，普通索引会使用change buffer。
即当这个数据页在内存中时就直接修改，否则先写到change buffer中（写的是操作），这样就暂时不用先操作磁盘了。
这样下次读取数据时，再执行change buffer中的这个操作，这样就保证了数据的一致性。
如果加了唯一索引，每次更新数据时就会校验唯一性，就会将数据从磁盘读取到内存，所以change buffer就使用不上了。
可见change buffer可以提升“写多读少”时候的性能。如果读多写少，由于加了change buffer, 性能反而可能下降。
 
<br>

17.  尽量不要用业务字段作为主键，但有一种情况例外，即典型的K-V场景。

<br>

18.  MySQL 中 sum 函数没统计到任何记录时，会返回 null 而不是 0，可以使用 IFNULL 函数把 null 转换为 0；
- MySQL 中 count 字段不统计 null 值，COUNT(*) 才是统计所有记录数量的正确方式。
- MySQL 中 =NULL 并不是判断条件而是赋值，对 NULL 进行判断只能使用 IS NULL 或者 IS NOT NULL。

<br>

19.  共享锁和排他锁
SELECT ... LOCK IN SHARE MODE走的是IS锁(意向共享锁)，即在符合条件的rows上都加了共享锁，
这样的话，其他session可以读取这些记录，也可以继续添加IS锁，
但是无法修改这些记录直到你这个加锁的session执行完成(否则直接锁等待超时)。 

SELECT ... FOR UPDATE 走的是IX锁(意向排它锁)，即在符合条件的rows上都加了排它锁，
其他session也就无法在这些记录上添加任何的S锁或X锁。
但是innodb有非锁定读(快照读并不需要加锁)，for update之后并不会阻塞其他session的快照读取操作，
除了select ...lock in share mode和select ... for update这种显示加锁的查询操作。 

通过对比，发现for update的加锁方式无非是比lock in share mode的方式
多阻塞了select...lock in share mode的查询方式，并不会阻塞快照读。
但是，for update 会加上一个写锁。

example:
```
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
 
begin;
select * from t where d=5 for update;
commit;
```
这个语句会命中 d=5 的这一行，对应的主键 id=5，
因此在 select 语句执行完成后，id=5 这一行会加一个写锁，而且由于两阶段锁协议，
这个写锁会在执行 commit 语句的时候释放。

<br>

20. 表数据量大，增加字段很慢的解决办法
新建一张表，将原表的数据全部插入进入，将新表重命名。

<br>

21.  其它常用语句
增加一个age字段，类型是INT(4)
```
ALTER TABLE student ADD age INT(4); 
```
<br>

22.  导出全部数据库
```
mysqldump -uroot -p --all-databases > sqlfile.sql
```

<br>

23.   关于外键约束
在Mysql数据库中，有时会用到外键约束FOREIGN KEY。这些外键约束也可以使用触发器TRIGGER来实现。当然，外键约束和触发器都是不提倡使用的。
因为外键约束和触发器容易给数据库服务器增加额外的负担，造成性能下降。甚至可能造成频发的锁等待或者死锁。

<br>

24.  MySQL FIND_IN_SET
FIND_IN_SET内置函数，允许您在逗号分隔的字符串列表中查找指定字符串的位置。
例子：
```
mysql> SELECT FIND_IN_SET('y','x,y,z');
+--------------------------+
| FIND_IN_SET('y','x,y,z') |
+--------------------------+
|                        2 |
+--------------------------+
1 row in set
```
如没有查询到则返回0