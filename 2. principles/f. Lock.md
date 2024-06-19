# MySql Lock
根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类。

1. 全局锁

全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，
命令是 
```
Flush tables with read lock (FTWRL)。
```
当你需要让整个库处于只读状态的时候，可以使用这个命令，之后其他线程的以下语句会被阻塞：
数据更新语句（数据的增删改）、数据定义语句（包括建表、修改表结构等）和更新类事务的提交语句。

全局锁的典型使用场景是，做全库逻辑备份。也就是把整库每个表都 select 出来存成文本。

但是让整库都只读，听上去就很危险：
如果你在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆；
如果你在从库上备份，那么备份期间从库不能执行主库同步过来的 binlog，会导致主从延迟。

官方自带的逻辑备份工具是 mysqldump。当 mysqldump 使用参数–single-transaction 的时候，
导数据之前就会启动一个事务，来确保拿到一致性视图。而由于 MVCC 的支持，这个过程中数据是可以正常更新的。

为什么还需要 FTWRL 呢？一致性读是好，但前提是引擎要支持这个隔离级别。
比如，对于 MyISAM 这种不支持事务的引擎，如果备份过程中有更新，总是只能取到最新的数据，那么就破坏了备份的一致性。
这时，我们就需要使用 FTWRL 

single-transaction 方法只适用于所有的表使用事务引擎的库。如果有的表使用了不支持事务的引擎，那么备份就只能通过 FTWRL 方法。
这往往是 DBA 要求业务开发人员使用 InnoDB 替代 MyISAM 的原因之一。

既然要全库只读，为什么不使用 set global readonly=true 的方式呢？
确实 readonly 方式也可以让全库进入只读状态，但我还是会建议你用 FTWRL 方式，主要有两个原因：
一是，在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。
因此，修改 global 变量的方式影响面更大，我不建议你使用。

二是，在异常处理机制上有差异。如果执行 FTWRL 命令之后由于客户端发生异常断开，
那么 MySQL 会自动释放这个全局锁，整个库回到可以正常更新的状态。
而将整个库设置为 readonly 之后，如果客户端发生异常，则数据库就会一直保持 readonly 状态，
这样会导致整个库长时间处于不可写状态，风险较高。

<br>

2. 表级锁

MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。
举个例子, 如果在某个线程 A 中执行 
```
lock tables t1 read, t2 write; 
```
这个语句，则其他线程写 t1、读写 t2 的语句都会被阻塞。
同时，线程 A 在执行 unlock tables 之前，也只能执行读 t1、读写 t2 的操作。
连写 t1 都不允许，自然也不能访问其他表。

在还没有出现更细粒度的锁的时候，表锁是最常用的处理并发的方式。
而对于 InnoDB 这种支持行锁的引擎，一般不使用 lock tables 命令来控制并发，毕竟锁住整个表的影响面还是太大。

另一类表级的锁是 MDL（metadata lock)。MDL 不需要显式使用，在访问一个表的时候会被自动加上。
MDL 的作用是，保证读写的正确性。
你可以想象一下，如果一个查询正在遍历一个表中的数据，而执行期间另一个线程对这个表结构做变更，
删了一列，那么查询线程拿到的结果跟表结构对不上，肯定是不行的。

因此，在 MySQL 5.5 版本中引入了 MDL，
当对一个表做增删改查操作的时候，加 MDL 读锁；
当要对表做结构变更操作的时候，加 MDL 写锁。

读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。
因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。

如何安全地给小表加字段？
首先我们要解决长事务，事务不提交，就会一直占着 MDL 锁。
在 MySQL 的 information_schema 库的 innodb_trx 表中，你可以查到当前执行中的事务。
如果你要做 DDL 变更的表刚好有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务。

如果你要变更的表是一个热点表，虽然数据量不大，但是上面的请求很频繁（kill已经不好用，请求来得快），
而你不得不加个字段，你该怎么做呢？
比较理想的机制是，在 alter table 语句里面设定等待时间，如果在这个指定的等待时间里面能够拿到 MDL 写锁最好，
拿不到也不要阻塞后面的业务语句，先放弃。之后开发人员或者 DBA 再通过重试命令重复这个过程。

MariaDB 已经合并了 AliSQL 的这个功能，所以这两个开源分支目前都支持 DDL NOWAIT/WAIT n 这个语法：
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ... 

<br>

3. 行锁

顾名思义，行锁就是针对数据表中行记录的锁。
这很好理解，比如事务 A 更新了一行，而这时候事务 B 也要更新同一行，则必须等事务 A 的操作完成后才能进行更新。
你可以验证一下：事务A和事务B中如果会update同一行，那么
事务 B 的 update 语句会被阻塞，直到事务 A 执行 commit 之后，事务 B 才能继续执行。

在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。
这个就是两阶段锁协议。

如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

假设你负责实现一个电影票在线交易业务，顾客 A 要在影院 B 购买电影票。
我们简化一点，这个业务需要涉及到以下操作：
- 1、从顾客 A 账户余额中扣除电影票价；
- 2、给影院 B 的账户余额增加这张电影票价；
- 3、记录一条交易日志。

试想如果同时有另外一个顾客 C 要在影院 B 买票，那么这两个事务冲突的部分就是语句 2 了。
因为它们要更新同一个影院账户的余额，需要修改同一行数据。

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。
所以，如果你把语句 2 安排在最后，比如按照 3、1、2 这样的顺序，那么影院账户余额这一行的锁时间就最少。
这就最大程度地减少了事务之间的锁等待，提升了并发度。

<br>

## 死锁和死锁检测
当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，
就会导致这几个线程都进入无限等待的状态，称为死锁，例子如下：
```
sessionA                          sessionB
begin
update t set k=k+1 where id =1    begin
                                  update t set k=k+1 where id =2
update t set k=k+1 where id =2
                                  update t set k=k+1 where id =1
```
这时候，事务 A 在等待事务 B 释放 id=2 的行锁，而事务 B 在等待事务 A 释放 id=1 的行锁。 
事务 A 和事务 B 在互相等待对方的资源释放，就是进入了死锁状态。

解决方式有两种：
一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。
另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。
将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑。

- 第一种策略，默认为50s,显然太长，死锁会占用太多资源，但是如果设置为1s，可能会造成误杀。
- 第二种策略，主动死锁检测在发生死锁的时候，是能够快速发现并进行处理的，但是它也是有额外负担的。
每个新来的被堵住的线程，都要判断会不会由于自己的加入导致了死锁，这是一个时间复杂度是 O(n) 的操作。
假设有 1000 个并发线程要同时更新同一行，那么死锁检测操作就是 100 万这个量级的。
虽然最终检测的结果是没有死锁，但是这期间要消耗大量的 CPU 资源。
因此，你就会看到 CPU 利用率很高，但是每秒却执行不了几个事务。

怎么解决由这种热点行更新导致的性能问题呢？问题的症结在于，死锁检测要耗费大量的 CPU 资源。

1）一种头痛医头的方法，就是如果你能确保这个业务一定不会出现死锁，可以临时把死锁检测关掉。
但是这种操作本身带有一定的风险，因为业务设计的时候一般不会把死锁当做一个严重错误，
毕竟出现死锁了，就回滚，然后通过业务重试一般就没问题了，这是业务无损的。
而关掉死锁检测意味着可能会出现大量的超时，这是业务有损的。

2）另一个思路是控制并发度。根据上面的分析，你会发现如果并发能够控制住，比如同一行同时最多只有 10 个线程在更新，
那么死锁检测的成本很低，就不会出现这个问题。一个直接的想法就是，在客户端做并发控制。
但是，你会很快发现这个方法不太可行，因为客户端很多。我见过一个应用，有 600 个客户端，
这样即使每个客户端控制到只有 5 个并发线程，汇总到数据库服务端以后，峰值并发数也可能要达到 3000。
因此，这个并发控制要做在数据库服务端。如果你有中间件，可以考虑在中间件实现；
如果你的团队有能修改 MySQL 源码的人，也可以做在 MySQL 里面。
基本思路就是，对于相同行的更新，在进入引擎之前排队。这样在 InnoDB 内部就不会有大量的死锁检测工作了。

3）从设计上解决
你可以考虑通过将一行改成逻辑上的多行来减少锁冲突。
还是以影院账户为例，可以考虑放在多条记录上，比如 10 个记录，影院的账户总额等于这 10 个记录的值的总和。
这样每次要给影院账户加金额的时候，随机选其中一条记录来加。
这样每次冲突概率变成原来的 1/10，可以减少锁等待个数，也就减少了死锁检测的 CPU 消耗。

这个方案看上去是无损的，但其实这类方案需要根据业务逻辑做详细设计。
如果账户余额可能会减少，比如退票逻辑，那么这时候就需要考虑当一部分行记录变成 0 的时候，代码要有特殊处理。