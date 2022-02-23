# RemoteApi.poll死锁问题

## poll业务逻辑

![image-20220120112510548](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120112510548.png)

## 问题描述

日志中出现如下问题：

> 【M】：插入应用出错，错误信息：
> \### Error updating database. Cause: com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
> \### The error may exist in 。。。
> \### The error may involve 。。。
> \### The error occurred while setting parameters
> \### SQL: INSERT INTO 。。。 。。。 VALUES ( ?, ?, ?, ?, ?, ?, ?, ? )
> \### Cause: com.mysql.cj.jdbc.exceptions.MySQLTransactionRollbackException: **Deadlock found when trying to get lock;** try restarting transaction

确定问题原因，由于死锁导致插入设备本地应用信息失败

## 排查

### 查看死锁日志

```sql
SHOW ENGINE INNODB STATUS
```

得到以下信息

> (1) TRANSACTION
>
> INSERT INTO 。。。  ( sn,package_name,... )  
> 
>VALUES  ( 'YT...8013','com.。。。.。。。','。。。','VX1.5.6',156,0,
> '2009-01-01 00:00:00.0','2021-05-14 11:02:27.0' )
> 
>持有锁
> 
>**lock_mode X locks gap before rec** 持有了间隙锁
> YT。。。8014     cn.。。。.。。。;;
> 
>等待锁
> 
>**locks gap before rec insert intention waiting** 请求插入意向锁
> 
>(2) TRANSACTION
> 
>INSERT INTO 。。。  ( sn,package_name,... )  
> 
> VALUES  ( 'YT...8014','android','。。。','。。。',。。,。。,
>'2009-01-01 00:00:00.0','2009-01-01 00:00:00.0' )
> 
> 持有锁
>
> **lock_mode X locks gap before rec** 持有了间隙锁
>YT...8014     cn........;;
> 
> 等待锁
>
> **locks gap before rec insert intention waiting** 请求插入意向锁

### 模拟情景

在自己的数据库中建立termninal_app，省略非必要字段，表结构如下：

![image-20220120104752064](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120104752064.png)

数据内容如下：

![image-20220120104907957](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120104907957.png)

启动两个事务即可，**模拟先删除后插入**

事务1：

```sql
begin;
delete from terminal_app where sn = 'WINTECCPMTEST5';
insert into terminal_app values('WINTECCPMTEST5','a','1',12);
COMMIT;
```

事务2：

```sql
BEGIN;
delete from terminal_app where sn = 'WINTECCPMTEST4';
insert into terminal_app values('WINTECCPMTEST4','z','1',12);
COMMIT;
```

**执行顺序：**事务1启动并删除TEST5的数据，事务2启动并删除TEST4数据，然后事务1执行insert，阻塞；事务二再执行insert，死锁。

![image-20220120105155282](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120105155282.png)

问题复现。

## 问题原理分析

首先明确间隙锁，下键锁，意向插入锁概念，详情见MySql中InnoDB锁机制的笔记。

* 这里提一嘴，由于MySql的默认事务隔离机制是可重复读，其可以解决幻读，但其使用了几乎暴力的下键锁和间隙锁，导致可能出现死锁的问题。

* 同时根据下键锁的特性，对于查询条件为**唯一索引**的记录加锁会退化成记录锁，仅锁住一行，因为唯一索引不会引起幻读问题。

这里首先启动事务1，删除Test5的数据。由于删除默认加下键锁，但查询条件还不是唯一索引（**唯一索引是联合主键，拆开就不是了**），为了解决幻读InnoDB不仅锁住了Test5，还锁住了**（Test5，h）到（Test4，m）**的区间和**（Test5，j）和（Test6，a）**的区间：

![image-20220120105902715](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120105902715.png)

同理对于Test4，导致Test4也会锁住**（Test5，h）到（Test4，m）**的区间：

![image-20220120110958147](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120110958147.png)

所以对于插入的数据，Test4,z请求意向插入锁与Test5的下键锁冲突(**满足死锁条件：互斥**)；Test5，a请求意向插入锁与Test4的下键锁冲突。

```sql
#两者都属于这个区间
insert into terminal_app values('WINTECCPMTEST4','z','1',12);
insert into terminal_app values('WINTECCPMTEST5','a','1',12);
```

同时两个事务都阻塞再insert上，无法commit（**满足死锁条件：不剥夺**），不释放间隙锁，**使其满足死锁条件：请求与保持和环路等待**

![image-20220120111748080](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120111748080.png)

导致死锁，**当然，这个发生的前提是一定并发情况下，导致多个事务（sn连在一起的，例如死锁日志里的情况）在没提交的情况下执行delete**

## 问题解决

1. 客户只要不用桌面管家，不调用poll，啥事没有。（不现实）
2. 修改terminal数据库的事务隔离级别，允许幻读，从而避免下键锁和间隙锁的产生。（不好，不应该因为一个业务修改数据库隔离级别）
3. 修改事务传播机制（学完了，不过我觉得此方案不能解决本问题，事务暂时挂起并不会释放锁，后续的插入仍会被阻塞，[详情请见：](https://gitee.com/ma-zhijian/xiaoma_studyway/blob/master/Spring框架/Spring事务传播机制.md)）
4. 修改表结构，将联合主键改为唯一主键【根据主键删除，避免间隙锁和下键锁，使其退化为记录锁】（理论上可行，但现有业务尤其软件商店大多数接口使用terminalapp，不敢改）
5. 修改事务时长（目前采取的办法）

> 想法：
>
> * 由于删除操作不可避免，所以可以只删除需要删除的，降低删除的时长。
> * 将没改变的数据不进行插入，只插入需要插入的。
> * 将整个的大事务分解成三个小事务，没必要非得全部回滚，进一步降低事务时间，避免同一时刻多个事务执行删除插入

**解决方案：**

* 从代码层面修复，实现上述三个想法，目前再查看plumlog，未发现一条错误日志，证明可行。