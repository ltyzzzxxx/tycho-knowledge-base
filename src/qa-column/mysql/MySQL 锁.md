---
title: MySQL 锁
---

## 问题清单

::: tip Questions
1.   **MySQL 有哪些锁？**

2.   **MySQL 是如何加锁的？**

3.   **Update 语句不加索引会锁住整张表吗？**

4.   **Insert 语句是如何加锁的？**

5.   **MySQL 如何避免死锁？**

6.   **MySQL 间隙锁之间会冲突吗？**
::: 

## 问题回答

1.   **MySQL 有哪些锁？**
::: info Answer
  -   全局锁：主要应用于数据库备份

      -   对于不支持可重复读的存储引擎MyISAM来说：在备份数据库时需要使用全局锁

      -   对于支持可重复读的存储引擎InnoDB来说：可以在备份数据库之前开启事务，会创建一份Read View，之后整个事务执行期间都会用这个Read View

  -   表级锁

      -   表锁：限制了所有线程对当前表的读写操作

      -   元数据锁：不需要显式使用，对数据库表进行操作，会自动加锁

          -   对一张表进行 CRUD 操作时，加的是 **MDL 读锁**；

          -   对一张表做结构变更操作的时候，加的是 **MDL 写锁**；

          -   MDL是在事务提交之后才会被释放，这意味着长事务有可能阻塞对于表结构的更改操作

              -   申请MDL的锁会形成一个队列，且写锁优先级高于读锁，

                  如果表结构的更改操作被阻塞，则会导致之后CRUD操作也会被阻塞

              -   因此为了保证表结构能够正常地被修改，需要在执行操作前，检查数据库当中正在执行的长事务

      -   意向锁

          -   在使用 InnoDB 引擎的表里对某些记录加上「共享锁」之前，需要先在表级别加上一个「意向共享锁」；

          -   在使用 InnoDB 引擎的表里对某些纪录加上「独占锁」之前，需要先在表级别加上一个「意向独占锁」；

          意向锁目的是为了快速判断当前表中有没有行独占锁，因此只会和共享表锁、独占表锁发生冲突

          -   如果没有「意向锁」，那么加「独占表锁」时，就需要遍历表里所有记录，查看是否有记录存在独占锁，这样效率会很慢。

          -   那么有了「意向锁」，由于在对记录加独占锁前，先会加上表级别的意向独占锁，那么在加「独占表锁」时，直接查该表是否有意向独占锁，如果有就意味着表里已经有记录被加了独占锁，这样就不用去遍历表里的记录。

          -   **意向共享锁和意向独占锁是表级锁，不会和行级的共享锁和独占锁发生冲突，而且意向锁之间也不会发生冲突，只会和共享表锁（\*lock tables ... read\*）和独占表锁（\*lock tables ... write\*）发生冲突。**

      -   AUTO-INC 锁：特殊的表锁机制，锁**不是再一个事务提交后才释放，而是再执行完插入语句后就会立即释放**

          -    在 MySQL 5.1.22 版本开始，InnoDB 存储引擎提供了一种**轻量级的锁**来实现自增。

          -    一样也是在插入数据的时候，会为被 `AUTO_INCREMENT` 修饰的字段加上轻量级锁，**然后给该字段赋值一个自增的值，就把这个轻量级锁释放了，而不需要等待整个插入语句执行完后才释放锁**。

      -   行级锁

          普通select语句是不会给记录加锁的，属于快照读，是通过MVCC多版本并发控制实现的

          以下是加锁的select语句

          ```sql
          //对读取的记录加共享锁
          select ... lock in share mode;
          
          //对读取的记录加独占锁
          select ... for update;
          ```

          -   Record Lock：记录锁，锁住的是一条记录，有读写锁之分

          -   Gap Lock：间隙锁，只存在于可重复读隔离级别，目的是为了解决可重复读隔离级别下幻读的现象

          -   Next-Key Lock：临键锁，是 Record Lock + Gap Lock 的组合，锁定一个范围，并且锁定记录本身

          -   插入意向锁

              一个事务在插入一条记录的时候，需要判断插入位置是否已被其他事务加了间隙锁（next-key lock 也包含间隙锁）。

              如果有的话，插入操作就会发生**阻塞**，直到拥有间隙锁的那个事务提交为止（释放间隙锁的时刻），在此期间会生成一个**插入意向锁**，表明有事务想在某个区间插入新记录，但是现在处于等待状态。
:::

2.   **MySQL 是如何加锁的？**
::: info Answer
  -   表级锁：

      -   修改数据库表操作时，会自动添加 `MDL` 写锁

      -   进行 `CRUD` 操作时，会自动添加 `MDL` 读锁

      -   备份数据库时，如果存储引擎不支持可重复读 `MVCC`，会添加表锁

      -   手动的添加独占表锁或共享表锁

      -   当对某些记录添加行锁时，需要先添加表级别意向锁

      -   数据库主键自增时，会自动添加 `AUTO-INCREMENT` 锁

  -   行级锁

      个人理解：记录锁、间隙锁、`next-key` 锁存在的目的是为了保证读写正确性，并实现可重复读，一定程度上防止幻读

      所以根据此，分析是否添加行级锁以及添加何种行级锁，都需要从防止幻读的角度分析

      -   以如下SQL语句进行查询或修改时，会对相关行记录添加共享或独占锁

          -   特别注意的是，前两条读操作SQL语句，需要在事务中执行。因为只执行单行语句，执行完后锁会被释放。

          ```sql
          //对读取的记录加共享锁(S型锁)
          select ... lock in share mode;
          //对读取的记录加独占锁(X型锁)
          select ... for update;
          //对操作的记录加独占锁(X型锁)
          update table .... where id = 1;
          //对操作的记录加独占锁(X型锁)
          delete from table where id = 1;
          ```

      -   对于唯一索引等值查询来说：

          -   若查询记录存在，会为该行记录添加独占记录锁

          -   如果查询记录不存在，会查找到第一条大于该查询记录的记录，添加间隙锁

          -   这两种情况下，仅靠记录锁或间隙锁，便可避免幻读现象

      -   对于唯一索引范围查询来说：

          -   针对大于范围查询，会对大于该记录的索引添加 `next-key` 锁

              ```sql
              select * from user where id > 15 for update;
              ```

          -   针对大于等于范围查询，会对大于该记录的索引添加next-key锁，若存在该记录，添加行记录锁

              ```sql
              select * from user where id >= 15 for update;
              ```


      -   对于不加索引的查询，会对全表进行扫描，会对每一条记录的索引添加next-key锁，相当于锁住全表
:::

3.   **Update 语句不加索引会锁住整张表吗？**
::: info Answer
  会的。因为不加索引，会对全表进行扫描，会对每一条记录的索引添加间隙锁与行记录锁，相当于锁住了整张表

  即使 `update` 语句中的 `where` 条件带上索引，也可能导致锁住整张表，因为索引可能失效，由优化器选择扫描方式

  避免锁住整张表发生：

  -   将 MySQL 里的 `sql_safe_updates` 参数设置为 1，开启安全更新模式。

  -   此时，update 语句必须满足如下条件之一才能执行成功：

      -   使用 where，并且 where 条件中必须有索引列；

      -   使用 limit；

      -   同时使用 where 和 limit，此时 where 条件中可以没有索引列；
:::

4.   **Insert 语句是如何加锁的？**
::: info Answer
  Insert 语句在正常执行时是不会生成锁结构的，它是靠聚簇索引记录自带的 `trx_id` 隐藏列来作为**隐式锁**来保护记录的。

  什么是隐式锁？

  当事务需要加锁的时，如果这个锁不可能发生冲突，`InnoDB` 会跳过加锁环节，这种机制称为隐式锁。隐式锁是 `InnoDB` 实现的一种延迟加锁机制，其特点是只有在可能发生冲突时才加锁，从而减少了锁的数量，提高了系统整体性能。

  `InnoDb` 在插入记录时，是不加锁的。如果事务 A 插入记录且未提交，这时事务 B 尝试对这条记录加锁，事务 B 会先去判断记录上保存的事务 id 是否活跃，如果活跃的话，那么就帮助事务 A 去建立一个锁对象，然后自身进入等待事务 A 状态，这就是所谓的隐式锁转换为显式锁。
:::

5.   **MySQL 如何避免死锁？**
::: info Answer
  -   合理的设计索引，区分度高的列放到组合索引前面，使业务 `SQL` 尽可能通过索引 `定位更少的行，减少锁竞争`。

  -   调整业务逻辑 `SQL` 执行顺序， 避免 `update/delete` 长时间持有锁的 SQL 在事务前面。

  -   避免 `大事务`，尽量将大事务拆成多个小事务来处理，小事务发生锁冲突的几率也更小。

  -   以 `固定的顺序` 访问表和行。比如两个更新数据的事务，事务 A 更新数据的顺序为 1，2;事务 B 更新数据的顺序为 2，1。这样更可能会造成死锁。

  -   在并发比较高的系统中，不要显式加锁，特别是是在事务里显式加锁。如 `select … for update` 语句，如果是在事务里`（运行了 start transaction 或设置了autocommit 等于0）`,那么就会锁定所查找到的记录。

  -   尽量按`主键/索引`去查找记录，范围查找增加了锁冲突的可能性，也不要利用数据库做一些额外额度计算工作。比如有的程序会用到 `select … where … order by rand()` 这样的语句，由于类似这样的语句用不到索引，因此将导致整个表的数据都被锁住。

  -   优化 SQL 和表设计，减少同时占用太多资源的情况。比如说，`减少连接的表`，将复杂 SQL `分解`为多个简单的 SQL。

  -   **设置事务等待锁的超时时间**。当一个事务的等待时间超过该值后，就对这个事务进行回滚，于是锁就释放了，另一个事务就可以继续执行了。在 InnoDB 中，参数 `innodb_lock_wait_timeout` 是用来设置超时时间的，默认值时 50 秒。

  -   **开启主动死锁检测**。主动死锁检测在发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。将参数 `innodb_deadlock_detect` 设置为 on，表示开启这个逻辑，默认就开启
:::

6.   **MySQL 间隙锁之间会冲突吗？**
::: info Answer
  不会。

  **间隙锁的意义只在于阻止区间被插入**，因此是可以共存的。**一个事务获取的间隙锁不会阻止另一个事务获取同一个间隙范围的间隙锁**，共享（S型）和排他（X型）的间隙锁是没有区别的，他们相互不冲突，且功能相同。
:::