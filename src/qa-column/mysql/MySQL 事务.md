---
title: MySQL 事务
---

## 问题清单

::: tip Questions
1.   **事务有哪些特性？**

2.   **并发事务会引发什么问题？**

3.   **事务隔离级别有哪些？**

4.   **InnoDB 引擎如何解决幻读问题？**

5.   **事务隔离级别具体的实现方式是什么？**

6.   **开启事务的命令是什么？**

7.   **Read View 在 MVCC 中是如何工作的？**

8.   **可重复读是如何工作的？**

9.   **读提交是如何工作的？**

10.   **快照读是如何避免幻读的？**

11.   **当前读是如何避免幻读的？**

12.   **可重复读完全避免幻读现象了吗？**
::: 

## 问题回答

1.   **事务有哪些特性？**
::: info Answer
  -   原子性：一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。执行事务过程中发生错误，事务则会回滚。

  -   一致性：是指事务操作前和操作后，数据满足完整性约束，数据库保持一致性状态。

  -   隔离性：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致，因为多个事务同时使用相同的数据时，不会相互干扰，每个事务都有一个完整的数据空间，对其他并发事务是隔离的。

  -   持久性：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

  持久性通过redo log重做日志保证的；原子性通过undo log回滚日志保证的；隔离性通过MVCC多版本并发控制保证的；一致性是前三者共同保证的
:::

2.   **并发事务会引发什么问题？**
::: info Answer
  -   脏读：一个事务读到了另一个未提交事务修改过的数据

  -   不可重复读：一个事务多次读取同一个数据，出现前后两次读到的数据不一致，由于另一个事务在这过程中修改数据并提交了事务

  -   幻读：在一个事务内多次查询某个符合条件的记录数量，出现前后两次查询到的记录数量不一致的情况
:::

3.   **事务隔离级别有哪些？**
::: info Answer
  -   读未提交：指一个事务还没提交，它做的数据变更就能被其他事务看到

      -   可能出现脏读、不可重复读、幻读问题

  -   读已提交：指一个事务提交之后，它做的数据变更才能被其他事务看到

      -   可能出现不可重复读、幻读问题

  -   可重复读：指一个事务在执行过程中看到的数据始终和事务刚启动时看到的数据一致

      -   可能出现幻读问题

  -   串行化：会对记录加上读写锁，在多个事务对这条记录进行读写操作时，如果发生了读写冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行；
:::

4.   **InnoDB 引擎如何解决幻读问题？**
::: info Answer
  虽然MySQL默认的事务隔离级别为可重复读，但是它在很大程度上解决了幻读问题

  -   针对快照读（普通select语句），通过MVCC方式解决了幻读

      可重复读隔离级别下，事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的，即使中途有其他事务插入了一条数据，是查询不出来这条数据的，所以就很好了避免幻读问题。

  -   针对当前读（select for update语句），通过next-key lock记录锁和间隙锁解决了幻读

      如果有其他事务在 next-key lock 锁范围内插入了一条记录，那么这个插入语句就会被阻塞，无法成功插入
:::

5.   **事务隔离级别具体的实现方式是什么？**
::: info Answer
  -   读未提交：直接读，没有额外限制

  -   串行化：通过读写锁避免并行访问

  -   读已提交、可重复读：通过read view实现，read view相当于数据快照

      -   「读提交」隔离级别是在「每个语句执行前」都会重新生成一个 Read View

      -   「可重复读」隔离级别是「启动事务时」生成一个 Read View，然后整个事务期间都在用这个 Read View
:::

6.   **开启事务的命令是什么？**
::: info Answer
  -   执行了 begin/start transaction 命令后，并不代表事务启动了。只有在执行这个命令后，执行了增删查改操作的 SQL 语句，才是事务真正启动的时机；

  -   执行了 start transaction with consistent snapshot 命令，就会马上启动事务。
:::

7.   **Read View 在 MVCC 中是如何工作的？**
::: info Answer
  Read View的组成：

  -   m_ids：活跃事务的事务ID列表。活跃事务指的是启动了但是还没提交的事务。

  -   min_trx_id：活跃事务中ID最小的事务。

  -   max_trx_id：全局事务中最大事务ID+1

  -   create_trx_id：创建Read View事务的事务ID

  每一个聚簇索引记录中都包含下面两个隐藏列：

  -   trx_id，当一个事务对某条聚簇索引记录进行改动时，就会**把该事务的事务 id 记录在 trx_id 隐藏列里**；

  -   roll_pointer，每次对某条聚簇索引记录进行改动时，都会把旧版本的记录写入到 undo 日志中，然后**这个隐藏列是个指针，指向每一个旧版本记录**，于是就可以通过它找到修改前的记录。

  所以，当一个事务去访问记录时，分为以下情况：

  -   如果记录的 trx_id 值小于 Read View 中的 `min_trx_id` 值，表示这个版本的记录是在创建 Read View **前**已经提交的事务生成的，所以该版本的记录对当前事务**可见**。

  -   如果记录的 trx_id 值大于等于 Read View 中的 `max_trx_id` 值，表示这个版本的记录是在创建 Read View **后**才启动的事务生成的，所以该版本的记录对当前事务**不可见**。

  -   如果记录的 trx_id 值在 Read View 的 `min_trx_id` 和 `max_trx_id` 之间，需要判断 trx_id 是否在 m_ids 列表中：

      -   如果记录的 trx_id **在** `m_ids` 列表中，表示生成该版本记录的活跃事务依然活跃着（还没提交事务），所以该版本的记录对当前事务**不可见**。

      -   如果记录的 trx_id **不在** `m_ids`列表中，表示生成该版本记录的活跃事务已经被提交，所以该版本的记录对当前事务**可见**。

  **这种通过「版本链」来控制并发事务访问同一个记录时的行为就叫 MVCC（多版本并发控制）。**
:::

8.   **可重复读是如何工作的？**
::: info Answer
  依靠记录字段中的roll_pointer指针

  -   沿着 undo log 链条往下找旧版本的记录，直到找到 trx_id 「小于」事务 的 Read View 中的 min_trx_id 值的第一条记录

  由于隔离级别时「可重复读」，所以事务再次读取记录时，还是基于启动事务时创建的 Read View 来判断当前版本的记录是否可见，

  所以读到的还是一样的记录
:::

9.   **读提交是如何工作的？**
::: info Answer
  事务每次读数据时都重新创建 Read View，其中活跃事务m_ids有可能随着其他事务的提交而变化，就可能导致读取的数据发生变化

  （若其他事务更新了数据）
:::

10.   **快照读是如何避免幻读的？**
::: info Answer
  可重复读隔离级是由 MVCC（多版本并发控制）实现的，实现的方式是开始事务后（执行 begin 语句后），在执行第一个查询语句后，会创建一个 Read View，**后续的查询语句利用这个 Read View，通过这个 Read View 就可以在 undo log 版本链找到事务开始时的数据，所以事务过程中每次查询的数据都是一样的**，即使中途有其他事务插入了新纪录，是查询不出来这条数据的，所以就很好了避免幻读问题。
:::

11.   **当前读是如何避免幻读的？**
::: info Answer
  举例：事务A执行如下语句，添加id范围为(2, +∞]的next-key lock，因此无法在事务A活跃时在此区间内插入数据，避免了幻读现象

  ```
  select name from t_stu where id > 2 for update;
  ```
:::

12.   **可重复读完全避免幻读现象了吗？**
::: info Answer
  没有。

  -   对于快照读， MVCC 并不能完全避免幻读现象。因为当事务 A 更新了一条事务 B 插入的记录，那么事务 A 前后两次查询的记录条目就不一样了，所以就发生幻读。

  -   对于当前读，如果事务开启后，并没有执行当前读，而是先快照读，然后这期间如果其他事务插入了一条记录，那么事务后续使用当前读进行查询的时候，就会发现两次查询的记录条目就不一样了，所以就发生幻读。
:::