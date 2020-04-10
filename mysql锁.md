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
### 页锁