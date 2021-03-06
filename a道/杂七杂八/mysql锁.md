
“由于比较复杂，一开始想要大量画图，但画图的成本实在是太高了，因为划清楚不太容易，所以很多地方呢，我就尽量通俗点的解释，不一定使用官方术语，如果有难理解的地方，我在手动画图”

---
## 核心要点
mysql锁的机制比较复杂，这一点单是从锁的数量上就可以看得出来，因此我们不会介绍所有锁相关的东西，只介绍常用的机制及其原理。实际上，很多业务都通过【分布式锁+业务自实现锁】来处理并发读写的，用不到mysql的锁，所以可能很多人不太熟悉这块，甚至有的公司都不允许使用select for update，因为需要一定的门槛，如果乱用就可能掉坑里了。

主要从以下几个方面来介绍，有些太基础的东西就没有罗列，涉及到的时候有疑问在讲。
1. 基本概念
2. 锁的类型
3. 锁的兼容性
4. 具体锁的讲解
5. 锁定读和非锁定读
6. 案例回顾
7. 业务实践与思考

---
## 基本概念
为了更好的理解，避免混淆，简单的提几个基本概念：
1. 锁的意义：保证并发访问的一致性读写（事务ACID那四个基本原则）。
2. innodb支持多粒度的层级锁，简单理解为支持表锁和行锁，行锁是innodb引擎的一大特色。
3. 锁之间存在兼容性问题，因而每一个锁的申请都要检查是否兼容。
4. 锁都是加在索引上，索引是B+树存储结构。
5. 死锁的特征：互相持有对方需要的锁而进入循环等待的状态。

---
## 锁的类型
InnoDB存储引擎支持多粒度锁定，即表级锁和行级锁“同时”存在。且行级锁的数量不影响开销，因为使用的是位图标记的算法。

- 表级锁（锁定整个表）
- 页级锁（锁定一页）
- 行级锁（锁定一行）
- 共享锁（S锁，MyISAM 叫做读锁）
- 排他锁（X锁，MyISAM 叫做写锁）
- 悲观锁（抽象性，不真实存在这个锁）
- 乐观锁（抽象性，不真实存在这个锁）

1. 表锁：X锁和S锁
2. 行锁：X锁和S锁
3. 意向锁：IX锁和IS锁，为了支持表锁和行锁的兼容性问题，而引入的。
4. 插入意向锁：IGap锁或IIX
5. 元数据锁：MDL锁

由于X锁和S锁是一个泛化的概念，所以后面具体加锁时，使用X型和S型来表示。

【图片】锁的兼容性

---
### 意向锁：Intention Lock
- 意向锁是将锁定的对象分为多个层次。
- 意向锁是表级锁，用来支持行锁和表锁的兼容，因而意向锁只和表锁冲突，不会和行锁冲突。
- 如果没有意向锁，这会带来一个问题：申请表锁时如何检查是否存在了行锁。因为锁之间存在兼容性问题，为了确定是否存在行级锁，需要扫描全表，开销太大。
- 引入意向锁后，每次申请行锁都会先申请意向锁，然后才加行锁，此时其他表锁申请会先判断表上是否有意向锁（意味着有行级锁的存在或者即将有行级锁的存在）然后检查其兼容性判断是否允许申请。

【图片】层级

延伸的问题：意向锁兼容，而行锁不兼容，是否可以申请。
答：是要不同行，可以兼容。（意向锁只和表锁冲突，不会和行锁冲突）

（IX，IS是表级锁，不会和行级的X，S锁发生冲突。只会和表级的X，S发生冲突）

---
### 行锁
1. 共享锁S锁
    
    `select lock in share mode`

    对读取的行记录加一个S锁，其他事务可以向被锁定的行加S锁，但是如果加X锁，则会被阻塞。

2. 排它锁X锁

    `select for update`

3. 两种类型均细分为以下三种：
    - R锁：Record Lock，没有索引则会使用隐式主键
    - GAP锁：间隙锁
    - NK锁：Next-key Lock，R锁和Gap锁的结合。

4. 自增锁：存活时间为sql语句期间

> 注：RC级别下只有R锁


---
#### GAP锁
- 间隙锁，锁定一个范围（前和后），但不包含记录本身。
- 作用：为了阻止多个事务将记录插入到同一范围内，而这会导致幻读的产生。

使用Gap锁的两种情况：
- 对普通索引加锁
- 对唯一索引进行范围查询：如 where id > 10 for update;

【图】示意图

---
#### NK锁：Next-key Lock
锁定一个范围（前gap+本身，但是后gap也存在，那是gap锁的，不属于NK的)，并且锁定记录本身，是结合了Gap Lock和Record Lock的一种锁定算法。
>默认情况下，mysql的事务隔离级别是可重复读，这时默认采用NK锁。
```
例如一个索引有10，11，13和20这四个值，则NK锁区间为
(-∞,10]
(10,11]
(11,13]
(13, 20]
(20,+∞)
真正锁定的是前区间，之所以叫next key，是因为锁定的记录本身是在区间的最后。
```

innodb的优化：

当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围，提高了应用的并发性。

---
### 插入意向锁：Insert Intention Lock
插入意向锁是一种特殊的Gap锁，即意向Gap锁，作用就是当多事务并发插入相同的gap空隙时，只要插入的记录不是gap间隙中的相同位置，则无需等待其他事务就可完成，这样就使得insert操作无须加真正的gap lock，可以提高插入的并发性能。

典型的两种死锁的情况：
- 并发插入相同索引的记录。死锁原因分析：
>首先session1插入一条记录，获得该记录的排它锁，这时session2和session3都检测到了主键冲突错误，但是由于session1并没有提交，所以session1并不算插入成功，于是它并不能直接报错吧，于是session2和session3都申请了该记录的共享锁，这时还没获取到共享锁，处于等待队列中。这时session1 rollback了，也就释放了该行记录的排它锁，那么session2和session3都获取了该行上的共享锁。而session2和session3想要插入记录，必须获取排它锁，但由于他们自己都拥有了共享锁，于是永远无法获取到排它锁，于是死锁就发生了(其中一个由于另一个的死锁回滚，然后成功获得了X锁)。如果这时session1是commit而不是rollback的话，那么session2和session3都直接报错主键冲突错误。如果这时session1是commit而不是rollback的话，那么session2和session3都直接报错主键冲突错误，因为这时候可以明确地确定发生了主键冲突，不必再往下走，之前只是检测到主键冲突，但因为session1还没commit或rollback，不好完全确定（因为session1存在commit和rollback两种情况，如果rollback了就不会主键冲突，因此才不能完全确定）。至于如何检测，可能是通过排它锁x以及insert的意向gap锁来大概推测。
>为什么要先获得S锁？
>所以获取共享锁是为了提高效率？毕竟共享锁的开销小。也就是说当session1大概率是commit的时候，所有获取共享锁的insert都会报错。如果是后面这种方式直接等待X锁，那么，所有Insert都得一个个获取X锁（排他锁，必须一个个获取，不能像共享锁一样所有都获取），这样，正常情况下，session1提交，效率大大提高。是这样吗？（解释：预测大概率会commit，因而大概率会集体主键冲突快速跑错而不必一个个排队失败，所以使用S锁）

- 并发删除和插入相同记录，本质上和第一种一样。死锁原因分析：
>另外一个类似的死锁是session1删除了id=1的记录并未提交，这时session2和session3插入id=1的记录。这时session1 commit了，session2和session3需要insert的话，就需要获取排它锁，那么死锁也就发生了；session1 rollback，则session2和session3报错主键冲突。

---
### 元数据锁：Metadata Lock
- 表级锁，用于保证DDl操作和DML操作的一致性。
- 类似于意向锁，但它是mysql的server层实现的，而意向锁是innodb引擎层实现的。
- 它作用范围比较广，能实现全局锁、库级别的锁、表空间级别的锁，这是InnoDB存储引擎层不能直接实现的锁。
- 事务会占据mdl，知道事务结束才释放。

---
## 锁定读和非锁定读
也叫快照读和当前读。

- 快照读
    
    - 即MVCC，多版本控制。
    - 如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放，而是读取快照数据。
    - 快照读不会加任何的锁。
    - 虽然不会幻读，但记录可能已经失效。
    - 事务的快照时间点是以第一个select来确认的。（未经确认）
    
    原理：事务加锁时会产生快照，快照的原理就是undo，undo用来在事务中回滚数据，因此快照数据本身是没有额外的开销。任何其他事务非锁定读不需要等待锁了，而是直接读快照，提高了并发。

    >READ COMMITTED读提交隔离级别的不同？

【图】快照原理图

- 当前读

    ```sql
    SELECT…FOR UPDATE
    SELECT…LOCK IN SHARE MODE
    insert update delete
    ```

    >对扫描到的索引记录加锁，与是否命中无关。

---
## 案例回顾

- 任何一种情况将从以下两个角度分析：
  - 记录是否存在
  - 索引是【唯一索引】还是【普通索引】，主键（聚集索引）和唯一索引的处理大致相同，主键上加的总是X锁。

- 总结常见的几种操作：
  - select : 快照读，不加锁
  - select lock in share mode : S型NK锁
  - select for update : X型NK锁
  - update 和 delete : X型NK锁
  - insert into : 插入意向锁IIX和X锁

- 死锁案例

---
## 业务实践
- 并发写入不存在的数据，死锁？
- 并发事务修改数据，丢失更新？
- 线上操作Alter，慢事务导致metadata lock？
- 库存方案与分布式锁，或其它方案？

