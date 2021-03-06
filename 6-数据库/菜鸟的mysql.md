## mysql引擎，InnoDB和MyISAM的区别
MyISAM是MySQL的默认数据库引擎（5.5版之前）。虽然性能极佳，而且提供了大量的特性，包括全文索引、压缩、空间函数等，但MyISAM不支持事务和行级锁，而且最大的缺陷就是崩溃后无法安全恢复。不过，5.5版本之后，MySQL引入了InnoDB（事务型数据库引擎），MySQL5.5版本后默认的存储引擎为InnoDB。  
大多数时候我们使用的都是InnoDB存储引擎，但是在某些情况下使用MyISAM也是合适的比如读密集的情况下。  
两者的对比：  
1、是否支持行级锁：MyISAM只有表级锁（table-level locking），而InnoDB支持行级锁（row-level locking）和表级锁，默认为行级锁。  
2、是否支持事务和崩溃后的安全恢复：MyISAM强调的是性能，每次查询具有原子性，其执行速度比InnoDB类型更快，但是不提供事务支持。但是InnoDB提供事务支持事务，外部键等高级数据库功能。具有事务（commit）、回滚（rollback）和崩溃修复能力（crash recovery capabilities）的事务安全（transaction-safe ACID compliant）型表。  
3、是否支持外键：MyISAM不支持，InnoDB支持。  
4、是否支持MVCC：仅InnoDB支持。应对高并发事务，MVCC比单纯的加锁更高效；MVCC只在READ COMMITTED和REPEATABLE READ两个隔离级别下工作，MVCC可以使用乐观锁和悲观锁来实现，各数据库中MVCC实现并不统一。  

## mysql索引
mysql索引使用的数据结构主要有BTree索引和哈希索引。对于哈希索引来说，底层的数据结构就是哈希表，因此在绝大多数需求为单条记录查询的时候，可以选择哈希索引，查询性能最快；其余大部分场景，建议选择BTree索引。  
mysql的BTree索引使用的是B树中的B+Tree，但对于主要的两种存储引擎的实现方式是不同的。  
MyISAM：B+Tree叶节点的data域存放的是数据记录的地址。在索引检索的时候，首先按照B+Tree搜索算法搜索索引，如果指定的key存在，则取出其data域的值，然后以data域的值为地址读取相应的数据记录。这被称为“非聚簇索引”。  
InnoDB：其数据文件本身就是索引文件。相比MyISAM，索引文件和数据文件是分离的，其表数据文件本身就是按B+Tree组织的一个索引结构，树的叶节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。这被称为聚簇索引。而其余的索引都作为辅助索引，辅助索引的data域存储相应记录主键的值而不是地址，这也是和MyISAM不同的地方。在根据主索引搜索时，直接找到key所在的节点即可取出数据；在根据辅助索引查找时，则需要先取出主键的值，再走一遍主索引。因此在设计表的时候，不建议使用过长的字段作为主键，也不建议使用非单调的字段作为主键，这样会造成主索引频繁分裂。  

## 什么是事务？
事务是逻辑上的一组操作，要么都执行，要么都不执行。  
事物的四大特性（ACID）  
1、原子性（Atomicity）：事务是最小的执行单位，不允许分割，事物的原子性确保动作要么全部完成，要么完全不起作用；  
2、一致性（Consistency）：执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；  
3、隔离性（Isolation）：并发访问数据库时，一个用户的事务不被其他事务所干扰，个并发事务之间数据库时独立的；  
4、持久性（Durability）：一个事务被提交之后。它对数据库总数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。  


## 并发事务带来哪些问题？
在典型的应用程序中，多个事务并发运行，经常会操作相同的数据来完成各自的任务（多个用户对同一数据进行操作）。并发虽然是必须的，但可能会导致以下的问题。  
脏读（Dirty read）：当一个事务正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。  
丢失修改（Lost to modify）：指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。  
不可重复读（Unrepeatable）：指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。  
幻读（Phantom read）：幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。  
不可重复读和幻读区别:  
不可重复读的重点是修改比如多次读取一条记录发现其中某些列的值被修改，幻读的重点在于新增或者删除比如多次读取一条记录发现记录增多或减少了。  


## 事务隔离级别有哪些？MySQL的默认隔离级别是？
SQL标准定义了四个隔离级别：  
READ-UNCOMMITTED（读取未提交）：最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。  
READ-COMMITTED（读取已提交）：允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生。  
REPEATABLE-READ（可重复读）：对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。  
SERIALIZABLE（可串行化）：最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读，不可重复读以及幻读。  
MySQL InnoDB存储引擎的默认支持的隔离级别是REPEATABLE-READ（可重读）。  
与SQL标准不同的地方在于InnoDB存储引擎在REPEATABLE-READ（可重读）事务隔离级别下使用的是Next-Key Lock锁算法，因此可以避免幻读的产生，这与其他数据库系统不同。所以说InnoDB存储引擎的默认支持的隔离级别是REPEATABLE-READ（可重读）已经可以完全保证事务的隔离性要求，即达到了SQL标准的SERIALIZABLE（可串行化）隔离级别。因为隔离级别越低，事务请求的锁越少，所以大部分数据库系统的隔离级别都是READ-COMMITTED（读取提交内容），但是你要知道的是InnoDB存储引擎默认使用REPEATABLE-READ（可重读）并不会有任何性能损失。  
InnoDB存储引擎在分布式事务的情况下一般会用到SERIALIZABLE（可串行化）隔离级别。  

## 锁机制与InnoDB锁算法
MyISAM和InnoDB存储引擎使用的锁：  
MyISAM采用表级锁（table-level locking）  
InnoDB支持行级锁（row-level locking）和表级锁，默认为行级锁  

表级锁和行级锁对比：  
表级锁：MySQL中锁定粒度最大的一种锁，对当前操作的整张表加锁，实现简单，资源消耗也比较少，加锁快，不会出现死锁。其锁定粒度最大，触发锁冲突的概率最高，并发度最低，MyISAM和InnoDB引擎都支持表级锁。  
行级锁：MySQL中锁定粒度最小的一种锁，只针对当前操作的行进行加锁。行级锁能大大减少数据库操作的冲突。其加锁粒度最小，并发度高，但加锁的开销也最大，加锁慢，会出现死锁。  

InnoDB存储引擎的锁的算法有三种：  
record lock：单个行记录上的锁  
gap lock：间隙锁，锁定一个范围，不包括记录本身  
next-key lock：record+gap锁定一个范围，包含记录本身  

## 大表优化
当MySQL单表记录数过大时，数据库的CRUD性能会明显下降，一些常见的优化措施如下：  
1、限定数据的范围  
务必禁止不带任何限制数据范围条件的查询语句。比如：我们当用户在查询订单历史的时候，我们可以控制在一个月的范围内；  
2、读写分离  
经典的数据库拆分方案，主库负责写，从库负责读；  
3、垂直分区  
根据数据库里面数据表的相关性进行拆分。例如，用户表中既有用户的登录信息又有用户的基本信息，可以将用户表拆分成两个单独的表，甚至放到单独的库做分库。  
简单来说垂直拆分是指数据表列的拆分，把一张列比较多的表拆分为多张表。  
垂直拆分的优点：可以使得列数据变小，在查询时减少读取的Block数，减少IO次数。此外，垂直分区可以简化表的结构，易于维护。  
垂直拆分的缺点：主键会出现冗余，需要管理冗余列，并会引起join操作，可以通过在应用层进行join来解决。此外，垂直分区会让事务变得更加复杂。  
4、水平分区
保持数据表结构不变，通过某种策略存储数据分片。这样每一片数据分散到不同的表或者库中，达到了分布式的目的。水平拆分可以支撑非常大的数据量。  
水平拆分是指数据表行的拆分，表的行数超过200万行时，就会变慢，这时可以把一张表的数据拆分成多张表来存放。举个例子，我们可以将用户信息拆分成多个用户信息表，这样就可以避免单一表数据量过大对性能造成影响。  
水平拆分可以支持非常大的数据量。需要注意的一点是:分表仅仅是解决了单一表数据过大的问题，但由于表的数据还是在同一台机器上，其实对于提升MySQL并发能力没有什么意义，所以水平拆分最好分库。  
水平拆分能够支持非常大的数据量存储，应用端改造也少，但分片事务难以解决，跨节点join性能较差，逻辑复杂。

## 解释一下什么是池化设计思想。什么是数据库连接池？为什么需要数据库连接池？
池化设计应该不是一个新名词。我们常见的如java线程池，jdbc连接池，redis连接池等就是这类设计的代表实现。这种设计会初始预设资源，解决的问题就是抵消每次获取资源的消耗，如创建线程的开销，获取远程连接的开销等。  
数据库连接本质就是一个socket的连接，数据库服务端还要维护一些缓存和用户权限信息之类的，我们可以把数据库连接池看作是是维护的数据库连接的缓存，以便将来需要对数据库的请求时可以重用这些连接。  

## 分库分表之后，id主键如何处理？
因为要是分成多个表之后，每个表都是从1开始累加，这样是不对的，我们需要一个全局唯一的id来支持。  
生成全局id有下面几种方式：  
UUID：不适合作为主键，因为太长了，并且无序不可读，查询效率低。比较适合用于生成唯一的名字的标示比如文件的名字。  
数据库自增id：两台数据库分别设置不同步长，生成不重复ID的策略来实现高可用。这种方式生成的id有序，但是需要独立部署数据库实例，成本高，还会有性能瓶颈。  
利用redis生成id：性能比较好，灵活方便，不依赖于数据库。但是，引入了新的组件造成系统更加复杂，可用性降低，编码更加复杂，增加了系统成本。  
Twitter的snowflake算法  
美团的Leaf分布式ID生成系统  
