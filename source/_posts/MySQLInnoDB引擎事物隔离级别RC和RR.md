---
title: MySQLInnoDB引擎事物隔离级别RC和RR
date: 2018-10-25 16:00:34
tags: [Mysql]
toc: true
---

## 事务

### 概念

事务指的是满足 ACID 特性的一组操作，可以通过 Commit 提交一个事务，也可以使用 Rollback 进行回滚。

### ACID

1. 原子性（Atomicity）

事务被视为不可分割的最小单元，事务的所有操作要么全部提交成功，要么全部失败回滚。

回滚可以用回滚日志来实现，回滚日志记录着事务所执行的修改操作，在回滚时反向执行这些修改操作即可。

2. 一致性（Consistency）

数据库在事务执行前后都保持一致性状态。在一致性状态下，所有事务对一个数据的读取结果都是相同的。

3. 隔离性（Isolation）

一个事务所做的修改在最终提交以前，对其它事务是不可见的。

4. 持久性（Durability）

一旦事务提交，则其所做的修改将会永远保存到数据库中。即使系统发生崩溃，事务执行的结果也不能丢失。

使用重做日志来保证持久性。


事务的 ACID 特性概念简单，但不是很好理解，主要是因为这几个特性不是一种平级关系：

* 只有满足一致性，事务的执行结果才是正确的。

* 在无并发的情况下，事务串行执行，隔离性一定能够满足。此时只要能满足原子性，就一定能满足一致性。

* 在并发的情况下，多个事务并行执行，事务不仅要满足原子性，还需要满足隔离性，才能满足一致性。

* 事务满足持久化是为了能应对数据库崩溃的情况。

![acid](/img/mysql-transaction-1.png)

### 隔离性

其中**隔离性**分为了四种：

#### READ UNCOMMITTED:未提交读

可以读取未提交的数据，未提交的数据称为脏数据，所以又称脏读。此时：幻读，不可重复读和脏读均允许；

#### READ COMMITTED：提交读

只能读取已经提交的数据；此时：允许幻读和不可重复读，但不允许脏读，所以RC隔离级别要求解决脏读；

#### REPEATABLE READ：可重复读

同一个事务中多次执行同一个select，读取到的数据没有发生改变；此时：允许幻读，但不允许不可重复读和脏读，所以RR隔离级别要求解决不可重复读；

#### SERIALIZABLE：可串行化

幻读，不可重复读和脏读都不允许，所以serializable要求解决幻读；


### 几个概念

#### 脏读：

可以读取未提交的数据。RC 要求解决脏读；

#### 不可重复读：

同一个事务中多次执行同一个select， 读取到的数据发生了改变(被其它事务update并且提交)；

#### 可重复读：

同一个事务中多次执行同一个select， 读取到的数据没有发生改变(一般使用MVCC实现)；RR各级级别要求达到可重复读的标准；

#### 幻读：

同一个事务中多次执行同一个select， 读取到的数据行发生改变。也就是行数减少或者增加了(被其它事务delete/insert并且提交)。SERIALIZABLE要求解决幻读问题；

这里一定要区分**不可重复读**和**幻读**：

不可重复读的重点是修改:

同样的条件的select， 你读取过的数据， 再次读取出来发现值不一样了

幻读的重点在于新增或者删除:

同样的条件的select， 第1次和第2次读出来的记录数不一样

对于前者， 在RC下只需要锁住满足条件的记录，就可以避免被其它事务修改，也就是 select for update， select in share mode; RR隔离下使用MVCC实现可重复读；

对于后者， 要锁住满足条件的记录及所有这些记录之间的gap，也就是需要 gap lock。

#### 数据库的默认隔离级别

除了MySQL默认采用RR隔离级别之外，其它几大数据库都是采用RC隔离级别。

#### MySQL 中RC和RR隔离级别的区别

MySQL数据库中默认隔离级别为RR，但是实际情况是使用RC 和 RR隔离级别的都不少。好像淘宝、网易都是使用的 RC 隔离级别。那么在MySQL中 RC 和 RR有什么区别呢。我们该如何选择呢。为什么MySQL将RR作为默认的隔离级别呢。

##### RC 与 RR 在锁方面的区别

1. 显然 RR 支持 gap lock(next-key lock)，而RC则没有gap lock。因为MySQL的RR需要gap lock来解决幻读问题。而RC隔离级别则是允许存在不可重复读和幻读的。所以RC的并发一般要好于RR；

2. RC 隔离级别，通过 where 条件过滤之后，不符合条件的记录上的行锁，会释放掉(虽然这里破坏了“两阶段加锁原则”)；但是RR隔离级别，即使不符合where条件的记录，也不会是否行锁和gap lock；所以从锁方面来看，RC的并发应该要好于RR；

##### RC 与 RR 在复制方面的区别

1. RC 隔离级别不支持 statement 格式的bin log，因为该格式的复制，会导致主从数据的不一致；只能使用 mixed 或者 row 格式的bin log; 这也是为什么MySQL默认使用RR隔离级别的原因。复制时，我们最好使用：binlog_format=row

2. MySQL5.6 的早期版本，RC隔离级别是可以设置成使用statement格式的bin log，后期版本则会直接报错；

##### RC 与 RR 在一致性读方面的区别

RC隔离级别时，事务中的每一条select语句会读取到他自己执行时已经提交了的记录，也就是每一条select都有自己的一致性读ReadView; 而RR隔离级别时，事务中的一致性读的ReadView是以第一条select语句的运行时，作为本事务的一致性读snapshot的建立时间点的。只能读取该时间点之前已经提交的数据。

##### RC 支持半一致性读，RR不支持

RC隔离级别下的update语句，使用的是半一致性读(semi consistent)；而RR隔离级别的update语句使用的是当前读；当前读会发生锁的阻塞。

1. 半一致性读：

> 简单来说，semi-consistent read是read committed与consistent read两者的结合。一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足 update的where条件。若满足(需要更新)，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)。semi-consistent read只会发生在read committed隔离级别下，或者是参数innodb_locks_unsafe_for_binlog被设置为true(该参数即将被废弃)。

对比RR隔离级别，update语句会使用当前读，如果一行被锁定了，那么此时会被阻塞，发生锁等待。而不会读取最新的提交版本，然后来判断是否符合where条件。

半一致性读的优点：

减少了update语句时行锁的冲突；对于不满足update更新条件的记录，可以提前放锁，减少并发冲突的概率。

