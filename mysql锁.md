# Mysql锁
**什么是锁**
- 锁是计算机协调锁哥进程或者线程并发访问某一资源的机制
    - 在数据库中，出传统的计算资源（如CPU,RAM,IO）的争用以外，数据也是一种供许多用户共享的资源。
        - 如何保证数据并发访问的一致性、有效性是所有数据库必须解决的一个问题
        - 锁冲突也是影响数据库并发访问性能的一个重要因素。从这个角度来说，锁对数据库而言显得尤为重要，也更加复杂
    - 事务控制
    - 使用锁对有限的资源进行保护，解决隔离和并发的矛盾

## 锁的分类
- 从对数据操作的类型来分（读\写）
    - 读锁（**共享锁**）
        - 针对同一份数据，多个读操作可以同时进行而不会互相影响
    - 写锁（**排它锁**）
        - 当前写操作没有完成前，它会阻断其他写锁和读锁
- 从对数据的操作粒度来分
    - 表锁
    - 行锁

## 三锁
### 表锁（偏读）
> 偏向MyISAM存储引擎，开销小，加锁快；无死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低

**案例分析**
- 建表sql
    ```
    create table mylock(
     id int not null primary key auto_increment,
     name varchar(20)
    )engine myisam;
    
    insert into mylock(name) values ('a');
    insert into mylock(name) values ('b');
    insert into mylock(name) values ('c');
    insert into mylock(name) values ('d');
    insert into mylock(name) values ('e');
    
    select * from mylock;
    ```

**表锁之加读锁**
- 手动增加一个表锁
    ```
    lock table mylock read,book write;
    -- 解释：给 mylock 表上读锁，给book表上写锁
    ```
- 查看命令
    - `show open tables`
        - 会发现 `In_use` 变成 `1`
- 解锁
    - unlock tables;
    
- 操作：
    - 在第1个session里执行
        ```
        lock table mylock read;
        select * from mylock;
        ```
    - 在第2个session里执行
        ```
        select * from mylock;
        ```
      - 小结：**读锁不会影响另外一个session的读操作**

- 实践并思考1：
    - 思考一下 第1个session 能不能修改表？
        - **不能**
    - 思考：第1个session 在对`mylock`表加锁的情况下能不能读别的表
        - **不能**
        > 错误提示：ERROR 1100 (HY000): Table 'emp' was not locked with LOCK TABLES
    - 思考：第2个session 能不能**更新** `mylock` 表
        - **不能,而且会阻塞**
        - 只要一解锁，session2 里面的更新语句就会立即执行了
        
**表锁之加写锁**
- 加写锁
    ```
    lock table mylock write;
    ```
- 实现并思考
    - 自己这个session能不能读自己加了锁的表
        - **能**
    - 自己这个session能不能修改自己加了锁的表
        - **能**
    - 自己这个session能不能读其他的表
        - **不能**
        > 错误提示：ERROR 1100 (HY000): Table 'emp' was not locked with LOCK TABLES
    - 其他session能不能读这个表
            - **不能，会阻塞**
    - 其他session能不能写这个表
        - **不能，会阻塞**
        

**小结论**
1. MyISAM 在执行查询语句（SELECT） 前，会自动给涉及的所有表加读锁，在执行增删改操作前，会自动给涉及的表加写锁
2. Mysql的表级锁有两种
    - 表共享读锁
    - 表独占写锁

    |锁类型|可否兼容|读锁|写锁
    |:---|:---|:---|:---
    |读锁|是|是|否
    |写锁|是|否|否
3. 结论，结合上表，所有对MyISAM表进行操作的时候，会有以下情况
    - 对 MyISAM 表的读操作（加读锁），不会阻塞其他进程对同一表的读请求，但会阻塞对同一表的写请求。只有当读锁释放后，才会执行其他进程的写操作
    - 对 MyISAM 表的写操作（加写锁），会阻塞其他进程对同一表的读和写操作，只有当写锁释放后，才会执行其他进程的读写操作
    - 简言之
        - 读锁会阻塞写
        - 写锁会阻塞读写

**表锁分析**
1. 看看那些表被加了锁
    - `show open tables;`
    - `show open tables where In_use = 1;`
2. 如何分析表锁定
    - 可以通过检查 `table_locks_waited` 和 `table_locks_immediate` 状态变量来分析系统上表的锁定
        - `table_locks_waited`
        > 产生表级锁定的次数，表示可以立即获取锁的查询次数，没立即获取锁值加1
        - `table_locks_immediate`
        > 出现表级锁定争用而发生等待的次数（不能立即获取锁的次数，每等待一次锁值加1），此值较高则说明存在着较为严重的表级锁争用情况
    - `show status like 'table%'`
3. **此外，MyISAM的读写锁调度是写优先，这也是MyISAM不是和做写为主表的引擎。因为写锁后，其他线程不能做任何操作，大量的更新会使查询很难得到锁，从而造成永久阻塞**
### 行锁
> 偏向 InnoDB 引擎，开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高
><br>InnoDB 与 MyISAM最大的不同有两点：一是支持事务；二是采用了行级锁

**由于行锁支持事务，事务的东西**
1. 事务（ACID属性）
    - 原子
    - 一致
    - 持久
    - 隔离
2. 并发事务处理带来的问题
    - 更新丢失 （Lost Update）
        - 当两个或者多个事务选择同一行，然后基于最初选定的值更新改行时，由于每个事务都不知道其他事务的存在，就会发生更新丢失问题
        - 最后的更新覆盖了由其他事务所做的跟新
        - 例如，两个程序员修改同一java文件。没个程序员独立地更改其副本，这样就覆盖了原始文档。最后保存其更改副本的编辑人员覆盖前一个程序员所做的更改
        - 如果一个程序员完成并提交事务之前，另一个程序员不能访问同一文件，则可避免此问题
    - 脏读 (Dirty Reads)
        - 一个事务正在对一条记录做修改，在这个事务完成并**提交前**，这条记录的**数据就处于不一致状态**；这时，另外一个事务也来读取同一条记录，如果不加控制，第二个事务读取了这些“脏”数据，并据此做进一步的处理，就会产生未提交的数据依赖关系。这种现象称之为“脏读”
        - 事务A读取到事务B**已修改但未提交的数据**，还在这个数据的基础上进行了操作。此时，如果事务B回滚，A读取的数据就是无效的
            - 不符合**一致性要求**
    - 不可重复读 (Non-Repeatable Reads)
        - 一个事务在读取了某些数据后的某个时间，再次读取以前读过的数据，却发现其读出的数据已经发生了改变、或某些记录已经被删除！ 这种现象称之为不可重复读
        - 事务A读取到了事务B已经提交的修改的数据，不符合隔离性
    - 幻读 (Phantom Reads)
        - 一个事务按照相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象称之为幻读
            - 幻读和脏读有点类似
            - 脏读是读到了其他事务修改的数据
            - 幻读是读到了其他事务新增的数据
3. 事务隔离级别

    |读数据一致性以及允许的并发副作用<br>隔离级别|读数据一致性|脏读|不可重复读|幻读
    |:---|:---|:---|:---|:---
    |未提交读 `Read uncommitted` |最低级别，只能保证不读取物理上损坏的数据|是|是|是
    |已提交读 `Read committed` |语句级|否|是|是
    |可重复读 `Repeatable read`|事务级|否|否|是
    |可序列化  `Serializable`|最高级别，事务级|否|否|否

    - 数据库事务的隔离级别越严格，并发副作用越小，但付出的代价就越大，因为事务隔离实质上就是使事务在一定程度上“串行化”进行，这显然与“并发”是矛盾的。
    - 同时，不同的应用对读一致性和事务隔离程度的要求也是不同的，比如许多应用对不可重复读和幻读并不敏感，可能更关心数据并发访问的能力
    - 查看数据库事务隔离级别
        - `show variables like 'tx_isolation'`
            - 适用 Mysql5.7
        - `show variables like 'transaction_isolation'`;
            - 使用 Mysql8
            
**演示**
1. 建表sql
    ```
    drop table if exists test_innodb_lock;
    
    create table test_innodb_lock(
        a int(10),
        b varchar(16)
    )engine=innodb;
    
    insert into test_innodb_lock values (1,'b2');
    insert into test_innodb_lock values (3,'3');
    insert into test_innodb_lock values (4,'4000');
    insert into test_innodb_lock values (5,'5000');
    insert into test_innodb_lock values (6,'6000');
    insert into test_innodb_lock values (7,'7000');
    insert into test_innodb_lock values (8,'8000');
    insert into test_innodb_lock values (9,'9000');
    insert into test_innodb_lock values (1,'b1');
    
    create index test_innodb_lock_a on test_innodb_lock(a);
    create index test_innodb_lock_b on test_innodb_lock(b);
    
    select * from test_innodb_lock;
    ```
2. 行表锁的基本演示 在控制台测试
    - 两个session里面同时 `set autocommit = 0;` 
        - 一个session更新
        - 一个session查询
    - **第一个session**里面执行一个更改语句
        - `update test_innodb_lock set b='4001' where a=4;` 
            - 注意：此时还没有提交
        - 查询一下 `select * from test_innodb_lock;`
    - **第二个session**再查询一下
        - `select * from test_innodb_lock;`
            - 结果，第二个session里面并没有查询到第一个session里面修改的内容
            - 说明没有出现脏读
    - **第一个session** 提交一下，**第二个session**再查询一下 
        - **第二个session**查询到了更新的值
    - ----------------------------------- 分隔符 -----------------------------------
    - 两个session里面同时 `set autocommit = 0;`
        - 两个session同时更新**同一行**
    - **第一个session**里面执行一个更改语句，但是没有提交
        - `update test_innodb_lock set b='4002' where a=4;`
    - **第二个session**里面执行一个更改语句，会发生什么
        - `update test_innodb_lock set b='4003' where a=4;`
        - **阻塞了**
            - 因为 session1 没有提交，占有了 `a=4` 这一行的锁
            - 所以发生了阻塞
    - **第一个session** 提交
        - **第二个session** 从 **阻塞** 转变成 **继续执行**
    - ----------------------------------- 分隔符 -----------------------------------
    - 两个session里面同时 `set autocommit = 0;`
        - 两个session同时更新**不同行**
    - **第一个session**里面执行一个更改语句，但是没有提交
        - `update test_innodb_lock set b='4004' where a=4;`
    - **第二个session**里面执行一个更改语句，会发生什么
        - `update test_innodb_lock set b='9001' where a=9;`
    - 结论：
        - 两个session不会互相影响

**索引失效会导致行锁变成表锁**
- ----------------------------------- 测试一下 -----------------------------------
    - 恢复sql `update test_innodb_lock set b='4000' where a=4;`
- 两个session 都  `set autocommit = 0;` 
- **第一个session**里面执行一个**索引失效的语句**
    - `update test_innodb_lock set a=41 where b=4000;` **不提交**
        - `varchar`类型查询的时候不加索引会导致索引失效
- **第二个session**执行对另外一行的修改
    - `update test_innodb_lock set b='9001' where a=9;`
    - 会出现什么现象
        - **阻塞了**
- **第一个session** commit
    - **第二个session**  从 **阻塞** 转变成 **继续执行**
- **第二个session** commit


**间隙锁的危害**
- 什么是间隙锁
    - 当我们用范围条件而不是相等条件检索数据，并请求共享或排它锁，InnoDB会给符合条件的已有数据记录的索引加锁
    - 对于键值在条件范围内但是并不存在的记录，叫做“间隙（GAP）”
    - InnoDB 也会对这个“间隙” 加锁，这种锁机制就是所谓的间隙锁 `Next-Key 锁`
- 危害
    - 因为query执行过程中**通过范围查找的话**，它会**锁定范围内所有的索引键值，即使这个键值并不存在**
    - 间隙锁有一个比较致命的弱点，就是当锁定一个范围键值之后，即使某些不存在的键值也会被无辜的锁定，而造成在锁定的时候**无法插入锁定键值范围内的任何数据**
    - 在某些场景下可能会对性能造成很大的危害
- 基于上面的案例的测试
    - 案例的特点 没有 `a=2` 的行
    - 两个session 都  `set autocommit = 0;` 
    - **第一个session**执行一个**范围更新**
        - `update test_innodb_lock set b='xxxx' where a>1 and a<6;`
    - **第二个session**插入一个 `a=2` 的值
        - `insert into test_innodb_lock values (2,'20000');`
            - **插入的时候阻塞了!!!**
    - **第一个session** commit
        - **第二个session**  从 **阻塞** 转变成 **继续执行**
    - **第二个session** commit

**如何锁定一行**
- 示例
```
begin;
select * from test_innodb_lock where a=8 for update; // 执行到这里这一行被锁定
... 执行其他操作
commit;
```

- 锁定某一行之后，其他session对这一行的操作会被阻塞
- 只有在 commit 之后，其他session的操作才会继续执行


**行锁分析**
- sql: `show status like 'innodb_row_lock%';`
    - `Innodb_row_lock_current_waits `： 当前正在等待锁定的数量 【重要关注】
    - `Innodb_row_lock_time`: 从系统启动到现在锁定总时间长度
    - `Innodb_row_lock_time_avg`: 每次等待所花的平均时间
    - `Innodb_row_lock_time_max`: 从系统启动到现在等待最长的一次所花费的时间
    - `Innodb_row_lock_time_waits`：系统启动后到现在总共等待的次数 【重要关注】


**优化建议**
- 尽可能让所有的数据检索通过索引来完成，避免无索引行锁升级为表锁
- 合理设计索引，尽量缩小锁的范围
- 尽可能较少检索条件，避免 `间隙锁`
- 尽量控制事务大小，减少锁定资源量和时间长度
- 尽可能低级别事务隔离

### 页锁
- 开销和枷锁时间结余表锁和行锁之间
- 会出现死锁
- 锁定粒度结余表锁和行锁之间
    - 并发度一般