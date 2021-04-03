---
title: GeekTime -《MySQL实战45讲》笔记
date: 2019-09-11 15:28:21
tags:
- 笔记
- 极客时间
- 数据库
- MySQL
categories:
- 极客时间
---

# MySQL实战45讲

### 01_基础架构：一条SQL查询语句是如何执行的？

- Mysql的逻辑架构图

  - 连接器

    负责跟客户端建立连接，获取权限，维持和管理连接

    ```
    mysql -h ip -P port -u username -p password
    ```

  - 查看数据库当前的连接

    ```
    mysql> show processlist;
    ```

  - 长连接，指连接成功后，客户端持续请求会使用同一个连接

  - 短连接，每次执行完很少的几次查询就断开连接，下次查询再重新建立一个

  - 分析器

    - 词法分析
    - 语法分析

  - 查询缓存

    - 执行过的语句及其结果直接缓存再内存中，查询命中则直接返回结果
    - 建议不要使用查询缓存，往往弊大于利
    - 使用SQL_CACHE显式指定使用查询缓存

    ```
    mysql> select SQL_CACHE * from T where ID = 1;
    ```

    - MySQL8.0版本将查询缓存整块功能删掉了

  - 优化器

    执行计划生成，索引选择

  - 执行器

    - 操作引擎，返回结果
    - rows_examined，表示语句执行过程中扫描了多少行
    - 引擎扫描行数跟rows_examined并不少完全相同的

  - 存储引擎

    负责数据的存储和提取，架构模式是插件式的，支持InnoDB，MyISAM，Memory等

### 02_日志系统：一条SQL更新语句是如何执行的？

- WAL技术

  Write-Ahead Logging，关键点就是先写日志，再写磁盘

- redo log 重做日志

  - InnoDB特有的日志
  - 固定大小，可以配置为一组4个文件，每个文件大小是1GB
  - 从头开始写，写到末尾就回到开头循环写
  - write pos是当前记录的位置
  - checkpoint是当前要擦除的位置，擦除前要把记录更新到数据文件
  - 有了redo log，InnoDB可以保证数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为crash-safe

- binlog 归档日志

- server层自有的日志

- 三点不同

  - redo log是InnoDB引擎特有的，binlog是Server层实现，所有引擎都可以使用
  - redo log是物理日志，binlog是逻辑日志
  - redo log是循环写，binlog是可以追加写的

- 两阶段提交

  - redo log的写入拆成了两个步骤，prepare和commit
  - 作用：为了让redo log和binlog的状态保持逻辑上的一致
  - 两阶段提交是跨系统维持数据逻辑一致性时常用的一个方案

### 03_事务隔离

- ACID

  - Atomicity，原子性
  - Consistency，一致性
  - Isolation，隔离性
  - Durability，持久性

- 事务隔离级别

  - 读未提交，read uncommitted

    一个事务还没提交时，它做的变更就能别别的事务看到

  - 读提交，read committed

    一个事务提交之后，它做的变更才会背其他事务看到

  - 可重复读，repeatable read

    一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的

  - 串行化，serializable

    后访问的事务必须等前一个事务执行完成，才能继续执行

- MVCC，数据库的多版本并发控制

- 不同时刻启动的事务会有不同的read-view

- 长事务的风险，要避免长事务的使用

- 事务的启动方式

  - begin 或者 start transaction
  - set autocommit=0

- information_schema.innodb_trx表可以查询长事务

### 04_深入浅出索引（上）

- 索引的出现其实就是为了提高数据查询的效率，就像书的目录一样

- 索引的常见模型

  - 哈希表

    哈希表这种结构适用于只有等值查询的场景

  - 有序数组

    在等值查询和范围查询场景中的性能就都非常优秀，但是更新数据的操作成本太高，只适用于静态存储引擎

  - 搜索树

- InnoDB的索引模型

  - 每一个索引在InnoDB里面对应一棵B+树
  - 主键索引的叶子节点存的是整行数据
  - 在InnoDB中主键索引也被称为聚簇索引，clustered index
  - 非主键索引的叶子节点内容是主键的值，在InnoDB中也被称为二级索引，secondary index

- 普通的索引查询方式：先搜索非主键索引树，再根据获取到的主键值到主键索引树搜索一次，称为回表

- 索引维护

  - 页分裂：数据页满，根据B+树的算法，需要申请一个新的数据页，然后挪动部分数据过去
  - 页合并：当相邻两个页由于删除了数据，利用率很低之后，会将数据页做合并

- 主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小

- 从性能和存储空间方面考量，自增主键往往是更合理的选择

### 05_深入浅出索引（下）

- 回到主键索引树搜索的过程，我们称为回表

- 覆盖索引

  由于覆盖所有可以减少树的搜索次数，显著提升查询性能，所以使用覆盖所有是一个常用的性能优化手段

- 最左前缀原则

- 索引下推

  Mysql5.6引入的索引下推优化，index condition pushdown，可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数

### 06 全局锁和表锁

- 根据加锁的范围，Mysql里面的锁大致可以分成全局锁，表级锁，和行锁三类
- 全局锁的典型使用场景是，做全库逻辑备份，也就是把整库每个表都select出来存成文本
- 官方自带的逻辑备份工具是mysqldump
- mysqldump -single-transaction 方法只适用于所有的表使用事务引擎的库
- 表级别的锁有两种，一种是表锁，一种是元数据锁，meta data lock，MDL
- 表锁的语法是 lock tables … read/write
- 读锁之间不互斥，读写锁之间，写锁之间是互斥的

### 07 行锁功过

- MyISAM引擎不支持行锁

- InnoDB支持行锁，这也是MyISAM被InnoDB替代的重要原因之一

- 两阶段锁协议

  在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放

- 如果你的事务中需要锁多个行，要把最可能造成锁冲突，最可能影响并发度的锁尽量往后放

- 死锁和死锁检测

  - 策略1，等待超时，超时时间通过参数innodb_lock_wait_timeout来设置
  - 策略2，发起死锁检测，参数innodb_deadlock_detect设置为on

### 08 事务到底是隔离的还是不隔离的？

- 视图的两个概念
  - 一个是view，创建视图的语法是create view，查询方法和表一样
  - 另一个是InnoDB在实现MVCC用到的一致性读视图，consistent read view，用于支持RC，RR隔离级别的实现
- 每个事务有一个唯一的事务ID，叫做transaction id，按申请顺序严格递增的
- 数据表中的一行记录，其实可能有多个版本，每个版本有自己的row trx_id
- up_limit_id，所有已提交的事务ID的最大值
- InnoDB利用了所有数据都有多个版本的这个特性，实现了秒级创建快照的能力
- 更新数据都是先读后写的，而这个读，只能读当前的值，称为当前读，current read
- RR和RC隔离级别的区别
  - 在RR下，只需要在事务开始的时候拿到up_limit_id，之后事务里的查询都共用这个up_limit_id
  - 在RC下，每一个语句在执行前都会重新算一次up_limit_id的值

### 09 普通索引和唯一索引，应该怎么选择？

- change buffer，当需要更新一个数据页时，如果数据页还没有在内存中，在不影响数据一致性的前提下，InnoDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了
- 将change buffer中的操作应用到原数据页，得到最新结果的过程称为purge
- 唯一索引的更新不能使用change buffer，实际上也只有普通索引可以使用
- 数据页在purge之前，change buffer记录的变更越多，收益就越大
- redo log主要节省的是随机写磁盘的IO消耗（转为顺序写），而change buffer主要节省的是随机读磁盘的IO消耗

### 10 Mysql为什么有时候会选错索引？

- 优化器的逻辑
  - 扫描行数是影响优化器选择索引的因素之一，扫描的行数越少，意味着访问磁盘数据的次数越少，消耗的CPU资源越少
- 索引的基数
  - 一个索引上的不同值越多，这个索引的区分度越好，而一个索引上不同的值的个数，我们称之为基数，cardinality
  - 基数越大，索引的区分度越好
  - InnoDB采用采样统计的方法得到索引的基数
  - 重新统计索引信息，analyze table t
  - 可以使用force index强行指定索引，也可以通过修改sql语句引导优化器

### 11 怎么给字符串家索引

- 使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本
- 建立索引时关注的是区分度，区分度越高越好
- 使用覆盖所有就用不上覆盖索引对查询性能的优化了，这是在选择是否使用前缀索引时需要考虑的一个因素
- 前缀索引区分度不够好的其他方式
  - 倒序存储
  - 使用hash字段
- 字符串创建索引的方式：
  - 完整索引
  - 前缀索引
  - 倒序存储
  - hash字段索引

### 12 为什么我的Mysql会”抖“一下？

- 当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内容页为”脏页“，内存数据写入到磁盘后，内存和磁盘上的数据页的内容就一致了，称为”干净页“
- Mysql偶尔抖一下的那个瞬间，可能就是在刷脏页
- 什么情况会引发数据库的flush过程
  - redo log写满了
  - 系统内存不够，淘汰一些数据页
  - Mysql认为系统空闲时
  - Mysql正常关闭服务时
- InnoDB用缓存池 buffer pool 管理内存
- InnoDB刷脏页的控制策略
  - innodb_io_capacity，告诉InnoDB你的磁盘能力
- 平时多关注脏页比例，不要让它经常接近75%
- innodb_flush_neighbors

### 13 为什么表数据删掉一半，表文件大小不变？

- 参数innodb_file_per_table
- innodb删除行记录，只是把记录标记为删除，之后会复用这个位置，但磁盘大小不会缩小
- innodb删除数据页上的所有记录，整个数据页就可以被复用了
- 用delete命令，删除整个表的数据，所有数据页都会被标记为可复用，但是磁盘上文件大小不会变小
- 重建表
  - 使用alter table A engine=InnoDB来重建表
- Mysql5.6版本开始引入的Online DDL，对重建表流程做了优化
- Online 和 inplace
  - DDL过程如果是Online的，就一定是inplace的
  - 反过来未必，也就是说inplace的DLL，有可能不是DDL的，如，加全文索引FULLTEXT index和空间索引 SPATIAL index
- 三种方式重建表
  - alter table t engine = InnoDB，也即recreate
  - analyze table t
  - optimize table t 等于recreate+analyze

### 14 count（*）这么慢，我该怎么办？

- count（*）的实现方式

  - MyISAM引擎把一个表的总行数存在了磁盘上，执行count（*）的时候会直接返回这个数，效率很高；
  - InnoDB执行时，需要把数据一行一行从引擎里面读出来，然后累计计数

- 在保证正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一

- 不同的count用法

  - count（主键id）
  - count（1）
  - count（字段）
  - count（*），不会把全部字段取出来，专门做了优化，不取值

- 按效率排序

  - count（字段）<count（主键id）<count（1）约等于count（*）

  ### 16 order by是怎么工作的？

  - 全字段排序

    ‘using filesort’表锁的需要排序，Mysql会给每个线程分配一块内存用于排序，称为sort_buffer

    - sort_buffer_size，表示的是Mysql为排序开辟的内存大小，如果排序数据大<sort_buffer_size，则在内存中完成，反之则利用磁盘临时文件辅助排序
    - number_of_tmp_files，表示的是，排序过程中使用的临时文件数

  - rowid排序

    - max_length_for_sort_data，是Mysql专门控制用于排序的行数据的长度的一个参数，如果单行出超过这个值，则Mysql认为要换一个排序算法

  - Mysql认为内存太小，才会采用rowid排序算法，如果认为内存足够大，会优先选择全字段排序，直接把需要的字段放到sort_buffer中

  - 覆盖索引，索引上的信息足够满足查询请求，不需要回到主键索引上取数据

  - 索引是有维护代价的，是不是需要用上覆盖索引，是需要权衡的决定

  ### 17 如何正确地显示随机消息

  - 使用order by rand()实现随机获取，使用了内存临时表，内存临时表排序的时候使用了rowid排序方法，执行代价比较大，在设计的时候要尽量避开这中写法

    ```sql
    select word from words order by rand() limit 3;
    ```

    - Extra字段中，Using temporary，表示需要使用临时表；Using filesort，表示需要执行排序操作

- 对于InnoDB表来说，相对于rowid排序，执行全字段排序会减少磁盘访问，因此会被优先选择

- Mysql的表是用什么方法定位一行数据的？表的主键，或者是InnoDB会自己生成一个长度为6字节的rowid作为主键

- tmp_table_size配置限制了内存临时表的大小，如果临时表超过了，则内存临时表会转成磁盘临时表

- 磁盘临时表引擎默认是InnoDB，由参数internal_disk_storage_engine控制

- Mysql5.6引入了一个新的排序算法，即优先队列排序算法

- 如何正确的随机排序？

### 18 为什么这些SQL语句逻辑相同，性能却差异巨大？

- 对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能
- 案例：
  - 条件字段函数操作
  - 隐式类型转换，如字符串和数字做比较时，字符串转换为整数
  - 隐式字符编码转换

### 19 为什么我只插一行的语句，也执行这么慢？

- 常见的情况：
  - 查询长时间不返回。一般大概率是查询表被锁住了
    - 等MDL锁，waiting for table metadata lock，表示有一个线程正在表上请求或持有MDL写锁，把select语句堵住了。查询sys.schema_table_lock_waits可以找到造成阻塞的process id
    - 等flush
    - 等行锁
  - 查询慢

### 20 幻读是什么，幻读有什么问题？

- 幻读，指的是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行
- 幻读的问题
  - 语义问题
  - 数据一致性问题
- 即使那所有的记录都加上锁，还是阻止不了新插入的记录
- InnoDB通过引入新的锁，间隙锁，gap lock，来解决幻读问题
- 间隙锁，锁的是两个值之间的空隙
- 行级锁，是不同类型行级锁之间有冲突，而间隙锁则是，和“往这个间隙插入一个记录”这个操作存在冲突关系，间隙锁之间不存在冲突关系
- 间隙锁和行级锁合称next-key lock，每个lock都是前开后闭区间，而间隙锁是开区间
- 间隙锁只在可重复读隔离级别下才有效

### 21 为什么我只改一行的语句，锁这么多？

### 22 Mysql有哪些饮鸩止渴提高性能的方法？

- 短连接风暴导致的性能问题，max_connections参数，用来控制一个Mysql实例同时存在的连接数的上限，超过这个值，系统会拒绝接下来的连接请求，提示，too many connections
- 可能的解决方案
  - 先处理掉那些占着连接但是不工作的线程
  - 减少连接过程的消耗
- 慢查询性能问题
  - 索引没有设计好
    - 变更索引，使用alter table
  - SQL语句没写好
    - 使用query_write功能，把输入的语句改写成另外一种模式
  - Mysql选错了索引
    - 应急方案是给语句加force index
- QPS突增问题
  - 修改白名单
  - 将用户删除，断开当前的连接
  - 可以使用query_write功能，先将压力最大的语句直接改写为select 1返回

### 23 Mysql是怎么保证数据不丢的？

- binlog的写入机制
  - 写入流程：binlog cache -> binlog files -> disk
  - 每个线程有自己的binlog cache，write到binlog files指的是写入到文件系统的page cache，并没有持久化
  - fsync到disk才是将数据持久化到磁盘的操作
- write和fsync的时机，是由参数sync_binlog控制的
- 在IO瓶颈的场景里，将sync_binlog设置成一个比较大的值，可以提升性能，但对应的风险：如果主机发生异常重启，会丢失最近N个事务的binlog日志
- redo log的写入机制
  - redo log的三种状态
    - 存在redo log buffer中，即Mysql进程的内存中
    - 已write，未fsync，存在文件系统的page cacge中
    - 已fsync，存在磁盘中
  - 未提交事务的redo log在以下三种场景中有可能会被写入到磁盘中：
    - 后台线程每秒一次的轮询操作
    - redo log buffer占用的空间即将达到innodb_log_buffer_size一半时，后台线程会主动写盘
    - 并行的其他事务提交时，顺带将这个事务的redo log buffer持久化到磁盘

### 24 Mysql是怎么保证主备一致的？

- readonly对超级权限用户是无效的，而主备同步更新的线程就拥有超级权限

- binlog的三种格式

  - statement

    原封不动地记录执行的sql语句，但是因为索引原因，可能导致主从执行结果不一致

  - row

    记录了执行的行记录，占用大量空间，耗费IO资源，影响执行速度

  - mixed（前两种格式的混合）

    Mysql会判断执行SQL使用哪种格式保存binlog，既利用了statement格式的优点，也避免了数据不一致的风险

- 如果线上Mysql设置的binlog格式是statement，基本上就是不合理的设置，至少要设置为mixed

- 用row的好处：恢复数据，delete语句可以改为insert语句，insert语句改为delete语句，而update语句可以将event前后的信息对调一下，重新执行即可

- 双master模式下，如何解决两个节点间的循环复制问题？

  - 1，规定两个库的server_id不同，如果相同，则不能互为主备关系
  - 2，备库拿到主库的binlog执行并生成与原binlog的server_id相同的新的binlog
  - 3，每个库收到日志后，先判断server_id，如果和自己的相同，表示是自己生成的，直接丢弃这个日志

### 25 Mysql是怎么保证高可用的？

- 主备延迟的最直接表现是，备库消费中转日志relay log的速度，比主库生产binlog的速度要慢
- 主备延迟的来源
  - 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差
  - 备库压力大，备库的查询消耗大量的CPU资源，影响同步资源，造成主备延迟
  - 大事务，一次性操作的数据太多
  - 备库的并行复制能力
- 可靠性优先策略，主备延迟为0时才执行主从切换
- 可用性优先策略，不等主备数据同步完成，直接把连接切到备库

### 26 备库为什么会延迟好几个小时？

- 如果备库执行日志的速度持续低于主库生成日志的速度，将造成主备延迟过高的问题
- 在mysql5.6之前只支持单线程复制，为了解决主备延迟问题，进而演进为多线程复制
- 并行复制策略
  - 按表分发策略
  - 按行分发策略
    - 相比于按表并行分发策略，按行并行策略在决定分发的时候，需要消耗更多的技术资源
  - mysql5.6版本的并行复制策略，支持的粒度是按库并行
  - mariaDB的并行复制策略，基于redo log的组提交特性
  - mysql5.7的并行复制策略
  - mysql5.7.22的并行复制策略
    - binlog-transcation-dependency-tracking参数，用于控制使用哪个策略
      - COMIT_ORDER
      - WRITESET
      - WRITESET_SESSION

### 27 主库出问题了，从库怎么办？

- 基于位点的主备切换

  - 主备切换时，位点很难精确定位到，故只能找一个稍微往前的，然后通过判断跳过那些在从库上已经执行过的事务

  - 在主备切换时，要主动跳过错误，有两种常用的方法：

    - 主动跳过事务

      ```sql
      set global sql_salve_skip_counter = 1;
      start slave;
      ```

    - 设置slave_skip_errors参数，直接设置跳过指定的错误，常见的有以下两类错误：

      - 1062，插入数据时的唯一键冲突
      - 1032，删除数据时找不到行，

- GTID，global transaction identifier，全局事务ID，mysql5.6版本引入的，是一个事务在提交的时候生成的，是这个事务的唯一标识

  ```sql
  GTID=server_uuid:gno
  ```

  - 基于GTID的主备切换

    ```sql
    master_auto_position=1
    ```

### 28 读写分离有哪些坑？

- 由于主从可能存在延迟，客户端执行完一个更新事务后马上发起查询，如果查询的是从库，就可能读到刚刚的事务更新之前的状态，称为过期读
- 处理过期读的方案：
  - 强制走主库方案
  - sleep方案
  - 判断主备无延迟方案
  - 配合semo-sync方案
  - 等主库位点方案
  - 等GTID方案

### 29 如何判断一个数据库是不是出问题了？

- 判断主库是否出问题？

  - select 1

    - 成功返回，只能说明这个库的进程还在，并不能说明主库没问题
    - 并发连接，show processlist的结果中的连接指的是并发连接
    - 并发查询，当前正在执行的语句，是指的并发查询（并发线程）
    - 查询线程加入锁等待之后，并发线程的计数会减一，也就是说等行锁的线程不算在并发线程限制数中

  - 查表判断

    - 一般做法是，在系统库中创建一个表，如health_check，然后定期执行select * from mysql.health_check
    - 如果数据库空间满了，这个方法就不好使

  - 更新判断

    - 在表中存入多行数据，并用主库，备库的server_id做主键

      ```sql
      CREATE TABLE `health_check` (
       `id` int(11) NOT NULL,
       `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
       PRIMARY KEY (`id`)
      ) ENGINE=InnoDB;
      ```

    - 然后更新语句，update mysql.health_check set t_modified=now()；

    - 相对比较常用，但是存在服务器IO资源分配导致的update检测判断错误的问题

  - 内部统计

    - 读取performance_shcema库的file_summary_by_event_name表，统计了每次IO请求的时间
    - 打开所有performance_schema项，系统性能大概会下降10%，建议只打开自己需要的项进行统计
    - 通过判断MAX_TIMER的值来判断数据库是否出问题，通过设定阈值，检测超过阈值则数据库当前异常

  - 优先考虑update系统表方法，再配合增加检测performance_schema的统计项