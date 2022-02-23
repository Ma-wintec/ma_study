# InnoDB的锁

## 1常用命令

```sql
SHOW OPEN TABLES;#查看表上加过的锁
UNLOCK TABLES;#释放表锁
SELECT @@transaction_isolation;#查看数据库的隔离级别
```

## 2数据操作类型划分

### 共享锁和排他锁Shared and Exclusive Locks

> InnoDB实现了标准的行级锁，其中有两种类型的锁，共享锁和排他锁。

**共享锁S**：允许持有该锁的事务读取一行。

**排他锁X**：允许持有该锁的事务更新或删除一行。

#### 兼容性：

|         | 共享锁S  | 排他锁X |
| :-----: | :------: | :-----: |
| 共享锁S | **兼容** | 不兼容  |
| 排他锁X |  不兼容  | 不兼容  |

> 举个例子，分别有事务T1与T2
>
> * 如果事务T1持有行r上的一个共享(S)锁，那么来自不同事务T2的请求对行r上的一个锁的处理如下:
>   * T2对S锁的请求可以立即被授予。因此，T1和T2都对r保持S锁定，即T2可以读r行数据；
>   * T2对X锁的请求不能立即授予，必须要等待T1释放掉该行的S锁后才能请求；
> * 如果事务T1持有行r上的独占(X)锁，那么来自不同事务T2的**对r上任何一种类型的锁的请求都不能立即被授予**。相反，事务T2必须等待事务T1释放其对行r的锁。

#### S锁测试：

```sql
SELECT * FROM terminal_app WHERE sn = 'WINTECCPMTEST4' LOCK IN SHARE MODE;
或
SELECT * FROM terminal_app WHERE sn = 'WINTECCPMTEST4' FOR SHARE; #8.0以后支持
#为sn=WINTECCPMTEST4的行加读锁
```

![image-20220119090250302](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119090250302.png)

此时另一个事务请求这些行的读锁，可以请求，因为S锁与S锁兼容；

![image-20220119090547020](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119090547020.png)

但此时如果请求这些行的写锁，则请求会被一直阻塞，因为T2事务需要等待T1事务释放S锁，因为S锁与X锁不兼容；

![image-20220119090818894](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119090818894.png)

#### X锁测试：

当事务T1请求到了这些行的X锁后，其余事务是无法请求任何这些行的任何锁的，都必须等待T1释放X锁后才能请求；

> **2022年2月8日发现：！！！！select ... for update如果找不到数据，是不加锁的！！！**
>
> **如果未指定主键或索引但是有此数据，则为表级锁**

![image-20220119091613249](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119091613249.png)

#### 8.0新特性

> 在8.0版本后，在SELECT...FOR SHARE 或 SELECT...FOR UPDATE后加入NOWAIT，SKIP LOCKED可跳过锁等待。

```sql
SELECT * FROM terminal_app WHERE sn = 'WINTECCPMTEST4' FOR SHARE NOWAIT;
#如果事务获取不到锁，直接返回报错
SELECT * FROM terminal_app WHERE sn = 'WINTECCPMTEST4' FOR SHARE SKIP LOCKED;
#如果事务获取不到锁，会直接只返回该查询条件中能查询到的数据
#e.g. select ... where sn like 'WINTECCPMTEST%' for share skip locked
#当CPMTEST4被锁定时，立刻返回CPMTEST1,CPMTEST2,CPMTEST3,CPMTEST5...的信息，只是不包含获取不到锁的CPMTEST4
```

#### 写操作

## 3数据操作粒度划分

> 提升数据库的并发度，要求每次锁定的数据范围越小越好，理论上最优选择是每次只锁定当前操作的数据能打到最大并发度。
>
> 由于数据库管理锁很耗费资源（涉及到获取锁，检查锁，释放锁...）,所以数据库系统要在并发相应和系统新能两方面权衡，这就是锁粒度的概念。

### 表级锁

> 顾名思义，锁住整张表，是MySq**l最基本的锁策略**，不依赖存储引擎，开销最小（粒度较大），由于其会一次锁住整张表，可以很好的解决死锁。但并发率较低，出现锁资源争用的概率最高。

#### 表级X锁与S锁

一句话，可以但一般没人用，硬要用：

```sql
LOCK TABLE t READ;#S锁
LOCK TABLE t WRITE;#X锁
```

#### 意向锁Intention Locks

> InnoDB支持多粒度锁，允许行锁和表锁共存。例如，一个语句，如LOCK TABLES…WRITE接受指定表上的一个排它锁(X锁)。为了实现多粒度级别的锁，InnoDB使用了意图锁。意图锁是**表级别**的锁，它指示事务稍后需要表中的一行使用哪种类型的锁(共享或排他)。有两种类型的意图锁:**意图共享锁(IS)**和**意图排它锁(IX)**。

##### 存在意义

> * 协调行锁和表锁的关系，支持多粒度锁并存（行锁和表锁）
> * 意向锁是一种**不与行锁冲突**的表级锁
> * 表明某个事务正在某些行持有了锁或准备持有锁

说人话，就是告诉别的事务，本事务已经或准备持有某些行的锁，如果你想要请求**表级锁**，你要掂量掂量能不能请求。

举个例子：

现在事务T1要在terminal_app表上锁住WINTECCPMTEST4，锁为X锁。

```sql
SELECT * FROM terminal_app where sn = 'WINTECCPMTEST4' FOR UPDATE;
```

那么此时事务T2想要获取terminal_app的表级锁，就会阻塞

![image-20220119102458097](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119102458097.png)

原因是因为在T1插入X锁时，数据库系统同时为terminal_app插入了一条IX意向排他锁，导致其后请求的S锁与IX锁冲突。

* 如果没有意向排他锁，由于T1已经占用CPMTEST4的X锁，导致T2不能请求表级S锁，但T2需要从头遍历这个表，看是否有行已经被锁住导致冲突，效率很低。
* 而当引入意向锁后，T2在请求S锁前看到有意向锁后，就会被阻塞，无需遍历整个表，从而提升了效率。

**意图共享锁IS**：表示事务意图在表中的各个行上设置共享锁。

**意图排他锁IX**：表示事务打算在表中的各个行上设置一个排它锁。

**意向锁是存储引擎自己维护的，用户无法手动操作意向锁。在为数据行添加共享或排他锁前，存储引擎InnoDB会首先获取该数据行对应数据表的意向锁。**

例如,[`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)设置`IS`锁，和[`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)设置`IX`锁上。

> 意图锁定协议如下:
>
> 事务在获取表中某一行的共享锁之前，必须首先获取表上的IS锁或更强的锁。
>
> 事务在获取表中某一行上的排它锁之前，必须首先获取表上的IX锁。

#### 表级锁类型兼容性：

表级锁类型兼容性总结在以下矩阵中。

|            | 排他锁 | 意图排他锁 | 共享锁 | 意图共享锁 |
| :--------: | :----: | :--------: | :----: | :--------: |
|   排他锁   |  冲突  |    冲突    |  冲突  |    冲突    |
| 意图排他锁 |  冲突  |    兼容    |  冲突  |    兼容    |
|   共享锁   |  冲突  |    冲突    |  兼容  |    兼容    |
| 意图共享锁 |  冲突  |    兼容    |  兼容  |    兼容    |

> 如果一个锁与现有的锁兼容，那么它就会被授予给一个请求的事务，但如果它与现有的锁冲突，就不会被授予。事务一直等待，直到冲突的现有锁被释放。如果锁请求与现有锁冲突，并且由于会导致死锁而不能被授予，则会发生错误。

说人话，如果该行被排他锁锁定，而你也想要为该行加上排他锁（请求一个锁与现有锁冲突），就需要等其他事务释放掉该行的排他锁，然后你再将排他锁加到该行上。同理，如果该行被一个事务加上了排他锁，那当你想申请整个表的写锁（意向排他锁）时，就需要等待该事务先释放该行的排他锁，然后你再加上IX到整个表上（冲突）。

#### 自增锁（了解）

> 建表时使用AUTO_INCREMENT，可以在插入语句时不为该列赋值。
>
> ![image-20220119111827751](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119111827751.png)

插入分三类：

* 简单插入，即可以**预先确定插入行数**；

  * ```sql
    INSERT ... VALUES()
    ```

* 批量插入，事先不知道要插入行数；

  * ```sql
    INSERT ... SELECT,REPLACE ... SELECT
    ```

* 混合模式插入，只制定了部分ID的值

  * ```sql
    INSERT INTO ... VALUES(1,'a'),(NULL,'b'),... 
    ```

锁也分三类：

* ```sql
  innodb_autoinc_lock_mode = 0 传统锁定
  ```

  * 所有insert语句都会获得表级锁AUTO-INC，只有争得此锁的事务才能进行插入，并发情况下效果不好；

* ```sql
  innodb_autoinc_lock_mode = 1 连续锁定
  ```

  * 此模式为8.0前默认模式，该模式仅对简单插入做出优化，由于简单插入可预知插入行数，所以其只在语句被初始处理时保持AUTO-INC锁，而不是直到语句完成，期间其他事务可以请求到锁。

* ```sql
  innodb_autoinc_lock_mode = 2 交错锁定
  ```

  * 8.0以后的默认模式，所有类型的INSERT语句都不会使用AUTO-INC，并且可以执行多个语句，自动递增值可以保证所有并发执行的INSERT语句中ID为唯一且单调递增的，但可能会让批量插入的数据值之间不连续。

#### 元数据锁MDL

> 5.5版本后引入，属于表锁范畴，作用为**保证读写正确性。**
>
> e.g.如果对一个表做增删改查的操作时，另一个事务修改了表结构，会导致查询的结果与表结构对不上。

**所以，当对一个表做增删改查操作时，加MDL读锁；对表结构更改时，加MDL写锁。**

MDL锁中读锁之间不互斥，写锁互斥，读写锁互斥，其用于保证改变表结构操作的安全性，无需显式使用，在访问一个表时会自动加上MDL锁。

##### 测试

在T1事务正在请求CPMTEST4的X锁时，T1自动请求了MDL的读锁，此时T2事务想要修改表结构便会被阻塞。

![image-20220119113413433](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119113413433.png)

```sql
SHOW PROCESSLIST;#查看线程状态
```

![image-20220119113547386](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119113547386.png)

**小问题**

启动三个线程，T1请求S锁（系统为其添加MDL读锁），T2请求修改表结构（系统为其添加MDL写锁），T3此时再次请求S锁（系统为其添加MDL读锁），此时T2，T3线程全部阻塞，原因为：

* T1持有MDL读锁，不释放；
* T2请求MDL写锁，与T1的MDL读锁冲突，阻塞；
* T3请求MDL读锁，本来与T1的MDL读锁不冲突，但由于T2请求MDL写锁，导致T3也一起阻塞。

![image-20220119131849422](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119131849422.png)

**这里猜测是这样，一旦有一个地方发生了Waiting for metadata lock，后续其余的操作都会Waiting for metadata lock**，详见下面记录一次问题的产生。

#### 记录一次问题的产生

在测试MDL锁时，我有一个事务忘记提交了，导致其一直在修改表结构，导致后续操作阻塞。

查询进程信息```SHOW PROCESSLIST;```显示阻塞进程的问题：**Waiting for table metadata lock**

通过查询未提交的事务：

```sql
select trx_state, trx_started, trx_mysql_thread_id, trx_query from `information_schema`.INNODB_TRX trx;
```

找到对应进程，kill掉即可。

**分析**：这个问题就可以说明上述问题，一旦一个地方Waiting for metadata lock，后续无论删除表，获取S锁，X锁，....都会Waiting for metadata lock阻塞。

### 行级锁

> 行锁仅在存储引擎层实现，其锁定粒度小，锁冲突概率低，并发场景下效率好；缺点是锁开销较大，加锁较慢，容易出现死锁。

#### 记录锁Record Locks

**记录锁**：对**索引**记录的锁。

> 例如，SELECT c1 FROM t WHERE c1 = 10 For UPDATE;防止任何其他事务插入、更新或删除t.c1值为10的行，但对其他的数据没影响。

##### 测试

换一张表，user_wives:

![image-20220119134942883](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119134942883.png)

事务T1修改id为1的数据

```sql
BEGIN;
UPDATE user_wives SET NAME = '大美女漂亮' WHERE ID = 1;
#先不提交事务
COMMIT;
```

事务T2查看id为1的数据，并请求该行S锁。

```sql
BEGIN;
SELECT * FROM user_wives WHERE id = 1 FOR SHARE;
#先不提交事务
COMMIT;
```

![image-20220119142758513](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119142758513.png)

可以看到请求阻塞，但如果此时请求id=3行的S锁，是可以请求的，并不影响其他事务的其他请求。

![image-20220119142853243](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119142853243.png)

事务T2如果此时更新其他行，同样可以更新。

```sql
UPDATE user_wives SET wife_name = '小美女最漂亮' where id = 3;
```

![image-20220119143255751](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220119143255751.png)

#### 间隙锁Gap Locks

> MySQL在默认隔离级别下（可重复读）可以解决幻读问题，有MVCC和加锁两种方式
>
> 加锁方式中，有一个问题。就是事务第一次执行读取的时候，无法给”幻影“记录加上记录锁。
>
> 所以InnoDB提出间隙锁的概念，LOCK_GAP。
>
> ![image-20220120091852029](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120091852029.png)

##### B+树简单概念

> MySQL采用B+树作为索引的实现，B+树的特征如下：
>
> * 有k个子树的中间节点包含有k个元素（B树中是k-1个元素），每个根节点或中间节点不保存数据，**只用来索引**，**所有数据都保存在叶子节点。**
> * **所有的叶子结点中包含了全部元素的信息**，及指向含这些元素记录的指针，**且叶子结点本身依关键字的大小自小而大顺序链接**。
> * 所有的中间节点元素都同时存在于子节点，在子节点元素中是最大（或最小）元素。
>
> B+树的形式如下：

![image-20220124154943091](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124154943091.png)

> 根节点元素8是子节点2,5,8 的最大元素，也是叶子节点6,8 的最大元素；
> 根节点元素15是子节点11,15 的最大元素，也是叶子节点13,15 的最大元素；
> 根节点的最大元素也就是整个B+树的最大元素，以后无论插入删除多少元素，始终要保持最大的元素在根节点当中。
> 由于父节点的元素都出现在子节点中，因此所有的叶子节点包含了全部元素信息，并且每一个叶子节点都带有指向下一个节点的指针，形成了一个有序链表

###### 单行查询

> 比如我们要查找的元素是3：
>
> ![image-20220124155111622](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124155111622.png)
>
> ![image-20220124155125948](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124155125948.png)

###### 范围查询

> 比如我们要查询3到11的范围数据：
>
> ![image-20220124155214564](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124155214564.png)
>
> ![image-20220124155238576](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124155238576.png)

**测试**

同样用user_wives表测试，启动两个事务

![image-20220120092628730](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120092628730.png)

```sql
#事务T1
begin;
#由于源数据中没有10 所以InnoDB为了防止幻读会给8~15的区间加上间隙锁
select * from user_wives where id = 10 for update; 
COMMIT;
#事务T2
begin;
insert into user_wives values (11,'a','a');
COMMIT;
```

![image-20220120092756067](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120092756067.png)

这里可以看到插入阻塞，说明间隙锁生效，T2插入阻塞（由于插入意向锁于间隙X锁阻塞）。

**说明：如果间隙锁在最后一条数据以下（e.g.select * from user_wives where id = 22 for share），其会将20~无穷大锁定，以防止插入幻读**

##### 间隙锁导致死锁的问题：

![image-20220120095319219](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120095319219.png)

同时联合主键会导致间隙锁引发另外的问题，详情见[poll接口死锁问题的处理方案](https://gitee.com/ma-zhijian/xiaoma_studyway/blob/master/%E9%A1%B9%E7%9B%AE%E4%B8%AD%E9%81%87%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98/RemoteApi.poll%E6%AD%BB%E9%94%81%E9%97%AE%E9%A2%98.md)。

#### 下键锁Next-Key Lock

这里我不想做太多赘述，下键锁的本质就是**记录锁和gap锁的结合体**，它技能保护该条记录，又能阻止别的事务插入被保护记录前边的间隙。

官方文档如此描述

![image-20220120125451838](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220120125451838.png)

你100%无法领会，所以说人话举一个例子：

| 10   | 11   | 13   | 20   |
| ---- | ---- | ---- | ---- |
| 哈哈 | 嘻嘻 | 啵啵 | 亲亲 |

如果delete 13（delete操作默认插入下键锁），锁住的区域为**(11,13],(13,20)**;

> 这个怎么理解呢，比较好的方式是引入下键锁就是为了**避免幻读**；而如果此时事务首先查询了13的值，显示只有一条”啵啵“，持有（13，啵啵）的S锁（举个例子），此时你想插入（13，小马），事务再一次**以同样查询条件**查询，会发现多了一条小马，这就是幻读现象。
>
> **以下对理解下键锁和间隙锁非常重要！！！！！**
>
> 如何避免幻读呢？MySQL的想法是：**只要不让你在13前后插入数据，就可以避免幻读！**即，锁住(11,13)和(13,20)这两个区域，你插不进来，自然就无法幻读。
>
> 想的很好，但极可能因此带来死锁的问题，这个在另一篇文章中有体现，这里不在赘述，详情可以查看[RemoteApi.poll死锁原因分析](https://gitee.com/ma-zhijian/xiaoma_studyway/blob/master/%E9%A1%B9%E7%9B%AE%E4%B8%AD%E9%81%87%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98/RemoteApi.poll%E6%AD%BB%E9%94%81%E9%97%AE%E9%A2%98.md)查看。
>
> 锁住这个区域后，你可以看出，实际上是两个间隙锁和一个记录锁
>
> * 记录锁锁住13，不让你修改13的值；
> * 间隙锁锁住（11，13），（13，20）【注意，这里不包括13】，防止幻读；
>
> 所以说下键锁是记录锁和间隙锁的结合体，那按照官方定义理解，就是（11，13]，（13，20）
>
> 当然这个解释太过简单，详细内容可以见[什么是间隙锁](https://blog.csdn.net/qq_21729419/article/details/113643359)

#### 插入意向锁

> **一个事务在插入一条记录前要判断插入未知是否有间隙锁，如果有就需要等待释放间隙锁**。InnoDB规定事务在等待插入时也需要在内存中生成一个锁结构，表示有一个事务在等待，等待在一个间隙中插入新记录。这个锁就叫插入意向锁，LOCK_INSERT_INTENTION，是一种特殊的间隙锁。

**插入意向锁**是在insert一条记录之前产生的间隙锁，此锁指示要以这样的方式插入多个事务，即插入到同一索引间隙中的多个事务不需要等待对方。

其为特殊的间隙锁，插入意向锁之间互不排斥

### 总结

MySql中，每隔层级的锁数量都是有限制的，因为锁会占用内存空间，锁的空间大小是有限的。当某个层级锁数量超过这个层级的阈值时，就会进行锁升级。e.g.行锁升级成了表锁，这样可以将锁空间的占用减低，但数据的并发度就下降了。

## 4对待锁态度划分

### 悲观锁

> 对数据被其他事务修改呈现保守态度，通过数据库自身锁机制实现，保证数据良好的排他性。

**悲观锁**总是假设最坏的情况，即每次去拿的数据都被别人修改了，所以每一次拿数据时都加锁，这样别的事务想操作这个数据就会阻塞（即共享资源每一次只给一个线程使用，让其他线程阻塞。用完再将资源给其他线程），e.g.行锁，表锁。Java中synchronization，ReentrantLock等独占锁就是悲观锁的体现。

> 举个例子：
>
> 由两个线程，分别执行秒杀的操作。
>
> ```sql
> #查询id为1001的商品库存
> select quantity from items where id = 1001;
> #如果库存大于0 就买~！
> insert into orders(item_id) values(1001);
> #修改商品库存，num代表购买数量
> update items set quantity = quantity - num where id = 1001;
> ```
>
> 如果不加锁，有可能出现下列情况
>
> | ThreadA                 | ThreadB                 |
> | ----------------------- | ----------------------- |
> | step1 查询还有100个商品 | step1 查询还有100个商品 |
> |                         | step2 生成订单          |
> | step2 生成订单          |                         |
> |                         | step3 减库存60个        |
> | step3 减库存60个        |                         |
>
> 100个商品减少了120次，完成了”超卖“

使用悲观锁解决问题就是，将商品查询后就将当前数据（商品）锁定，可以加写锁，知道减完库存后再解锁，这样就不会有第三者对其修改了。只不过，需要将整个执行的sql语句放在一个事务中，否则无法达到锁定数据的目的。

可以修改sql语句如下：

```sql
begin:
#查询id为1001的商品库存 （加写锁）
select quantity from items where id = 1001 for update;
#如果库存大于0 就买~！
insert into orders(item_id) values(1001);
#修改商品库存，num代表购买数量
update items set quantity = quantity - num where id = 1001;
commit;
```

```select ... for update```时MySql的悲观锁，使用这条语句后，其他事务要执行查询的时候就会被阻塞，因为在1001行的锁并未被释放。

**这里一定要注意，如果要使用悲观锁，一定一定要注意索引！！！一定一定要注意索引！！！一定一定要注意索引！！！**

**因为select ... for update会将执行过程中所有扫描的行都锁上，如果不适用索引则会将很多没必要的行扫描，大大降低效率。**

> **悲观锁不适用的场景很多**，因为悲观锁根本上使用的是数据库的锁机制实现，这样做会给数据库的性能带来影响，特别是长事务而言。

### 乐观锁

## 5InnoDB中不同SQL语句设置的锁

`InnoDB`设置特定类型的锁，如下所示。

- [`SELECT ... FROM`](https://dev.mysql.com/doc/refman/8.0/en/select.html)是一个一致的读取，读取数据库的快照并**设置无锁**，除非事务隔离级别设置为[`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable)。为[`SERIALIZABLE`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable)级别，搜索集将共享它遇到的索引记录上的下一个键锁。但是，对于使用唯一索引锁定行以搜索唯一行的语句，只需要索引记录锁。
- [`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)和[`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)使用唯一索引的语句获取扫描行的锁，并释放不符合结果集中包含条件的行的锁(例如，如果它们不符合`WHERE`条款)。但是，在某些情况下，行可能不会立即解锁，因为结果行与原始源之间的关系在查询执行过程中丢失。例如，在[`UNION`](https://dev.mysql.com/doc/refman/8.0/en/union.html)，在评估表是否符合结果集的条件之前，可能会将表中的扫描行(和锁定行)插入临时表中。在这种情况下，临时表中的行与原始表中的行之间的关系将丢失，而后面的行直到查询执行结束时才会被解锁。
- 为[锁读](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking_read) ([`SELECT`](https://dev.mysql.com/doc/refman/8.0/en/select.html)带着`FOR UPDATE`或`FOR SHARE`), [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html)，和[`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html)语句，所使用的锁取决于语句是使用具有唯一搜索条件的唯一索引，还是使用范围类型搜索条件。
  - 对于具有唯一搜索条件的唯一索引，`InnoDB`只锁定已找到的索引记录，而不锁定[空隙](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_gap)在它之前。
  - 对于其他搜索条件和非唯一索引，`InnoDB`锁定扫描的索引范围，使用[间隙锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_gap_lock)或[下键锁](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_next_key_lock)阻止其他会话插入范围所涵盖的空白。有关间隙锁和下一个键锁的信息，请参阅[第15.7.1节，“InnoDB锁定”](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html).
- 对于索引记录搜索遇到的情况，[`SELECT ... FOR UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)阻止其他会话执行[`SELECT ... FOR SHARE`](https://dev.mysql.com/doc/refman/8.0/en/select.html)或者阅读某些事务隔离级别。一致读取忽略在Read视图中存在的记录上设置的任何锁。
- [`UPDATE ... WHERE ...`](https://dev.mysql.com/doc/refman/8.0/en/update.html)在搜索遇到的每一条记录上设置一个独占的**下键锁**。但是，对于**使用唯一索引锁定行**以搜索唯一行的语句，只需要索引**记录锁。**
- 什么时候[`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html)修改聚集索引记录，对受影响的二级索引记录进行隐式锁定。这个[`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html)在插入新的辅助索引记录之前和插入新的辅助索引记录之前执行重复的检查扫描时，操作还会对受影响的辅助索引记录进行共享锁。
- [`DELETE FROM ... WHERE ...`](https://dev.mysql.com/doc/refman/8.0/en/delete.html)在搜索遇到的每一条记录上设置一个独占的**下键锁**。但是，对于使用**唯一索引锁定行**以搜索唯一行的语句，只需要索引**记录锁**。
- [`INSERT`](https://dev.mysql.com/doc/refman/8.0/en/insert.html)在插入的行上设置独占锁。此锁是**索引记录锁**，而不是下一个键锁(即没有间隙锁)，也**不会阻止其他会话插入插入行之前的间隙**。

