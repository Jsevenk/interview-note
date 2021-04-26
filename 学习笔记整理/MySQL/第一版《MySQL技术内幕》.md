# 第一版《MySQL技术内幕》
## 《MySQL技术内幕》笔记

### 第一章
#### MySQL体系结构
操作 -> MySQL实例 -> MySQL数据库

MySQL组成部分
1. 连接池组件
2. 管理服务和工具组件
3. SQL接口组件
4. 查询分析器组件
5. 优化器组件
6. 缓存组件
7. 插件式存储引擎
8. 物理文件

#### MySQL存储引擎
**InnoDB**
支持事务
设计目标：主要面向在线事务处理的应用
特点：行锁、外键、默认读取操作不会产生锁
5.5.8 默认引擎

1. 表空间
2. 多版本并发控制（MVCC） ->高并发性、4种隔离级别、next-key locking
3. 插入缓冲、二次写、自适应哈希所以、预读
4. 表中数据的存储：聚集方式。主键 顺序存放。ROWID


MyISAM存储引擎
1. 不支持事务、表锁
2. 支持全文索引
3. 缓存池只缓存索引文件，不缓存数据文件

主要面向OLAP数据库应用


#### 连接MySQl
TCP/IP
权限视图：限制host（ip）、user、password

命名管道 && 共享内存
//todo 了解下


### 第二章 InnoDB存储引擎
#### 版本
5.5 -> innoDB 1.1.x (增Linux AIO、多回滚段[不支持则最大并发事务数量限制在1023])
5.6 -> innoDB 1.2.x

#### 体系架构
后台线程*N
innoDB存储引擎内存池
数据文件

**后台线程**
1. Master Thread
2. IO Thread
3. Purge Thread
4. Page Cleaner Thread

Master Thread：
1.异步将缓存池数据刷新到磁盘
2.保证数据的一致性 ->脏页的刷新、合并插入缓冲（INSERT BUFFER）、UNDO页的回收

IO Thread：
原因：在InnoDB中大量使用了AIO（Async IO）来处理写IO请求
1.负责这些IO请求的回调（call back）处理
2.write、read、insert buffer、log IO thread
（1.0.x之前4个，之后4+4+1+1，可调整）

Purge Thread：
1.事务被提交后，其所使用的undolog可能不再需要，使用Purge Thread来回收已经使用并分配的undo页
2.[1.1.x]之前,purge操作仅在Master Thread完成
3.[1.1.x]开始，purge可以独立到单独线程中
（独立到Purge Thread中，提升CPU使用率以及提升存储引擎性能）
4.[1.2.X]支持多个Purge Thread，目的：加快undo页的回收 & 需要离散地读取undo页->进一步利用此版的随机读取性能

Page Cleaner Thread：
[1.2.x]引入，将脏页的刷新操作都放入到单独的线程完成 -> 减轻Master Thread工作,减轻对于用户查询的阻塞

**内存**
1. 缓冲池
2. LRU List、Free List、Flush List
3. 重做日志缓冲
4. 额外的内存池

缓冲池
innoDB基于磁盘存储的，按照页的方式管理数据
利用缓冲池技术来提高性能
1.读取页 -> 存放在缓冲池 -> 下一次读取相同页,判断是否在缓冲池，有读取，无读取磁盘
2.修改操作 -> 先修改缓冲池 -> 利用Checkpoint的机制刷新回磁盘
3.数据页类型：索引页、数据页、undo页、插入缓冲（insert buffer）、自适应哈希索引、InnoDb存储的锁信息(lock info)、数据字典信息 -> 其中主要是数据页和索引页
4.[1.0.x]开始允许有多个缓冲池实例 -> 每个页根据哈希值平均分配到不同的缓冲池中 -> 减少内部资源竞争、增加并发处理能力
5.[5.6]开始可以观察缓冲的状态（INNODB_BUFFER_POOL_STATS）,可以看到各个缓冲池的使用情况（POOL_SIZE/FREE_BUFFERS/DATABASE_PAGES）


LRU List、Free List、Flush List
如何管理缓冲池的内存区域

通常：
1.通过LRU算法管理缓冲池的
（LRU算法：频繁使用的页放在LRU列表前端，最少使用的页放在LRU列表的尾端，缓冲池不能放读取到的新页时，释放LRU尾端的页面）

InnoDB中
1.缓冲池中页的大小默认16K
2.优化LRU ->增加了midpoint位置
（新读取的页放入LRU列表的midpoint位置[默认在5/8处]，分割old列表和new列表）
3.不采用的原因：某些SQL操作（常见的如：索引或数据的扫描）可能会访问许多页或者全部页（却又不是热点数据），导致原本的缓冲池数据被刷掉，影响效率。
4.innodb_old_blocks_pct/innodb_old_blocks_times
(LRU列表、FREE列表、old部分到new部分、page made young、page not made young)
5.[1.0.x]开始，压缩页 unzip_LRU列表管理
6.脏页（LRU列表中的页被修改后），即缓冲池中的页和磁盘上的页数据产生了不一致。CHECKPOINT机制刷新会磁盘，Flush列表则为脏页列表。（脏页列表既存在于LRU列表，又存在于Flush列表）


重做日志缓冲（redo log buffer）
1.重做日志信息先放入缓冲池，再一定频率刷新到重做日志文件。
2.一般不要设置很大，一般是每一秒刷新的。只要保证每秒产生的事务在这个缓冲之中即可（默认8M）。
3.3种情况刷新到重做日志文件中（1.Master Thread每秒刷新2.每个事务提交时3.当重做日志缓冲池剩余空间小于1/2时）


额外的内存池

//todo需要理解
缓冲池（innodb_buffer_pool）
帧缓冲（frame buffer）
缓冲控制对象（buffer control block）

缓冲控制对象 记录了LRU、锁等信息，这个对象的内存是从额外内存池中申请的，所以InnoDB的缓冲池和额外内存池有相关性，一个增加另一个也要同步增加。

#### Checkpoint技术

避免发生数据丢失问题，采用WriteAheahLog，即当事务提交时，先写重做日志（redo log），再修改页。
[宕机后通过重做日志来恢复数据]
事务的持久性的要求（ACID中的D）。

checkpoint技术解决的问题：
1.缩短数据库的恢复时间
2.缓冲池不够用时，将脏页刷新到磁盘
3.重做日志不够用时，刷新脏页

[宕机后，只需要对checkpoint之后的重做日志进行恢复即可]

LSN？

脏页刷回磁盘：
每次刷新多少页到磁盘？每次从哪里取脏页？什么时候触发？

两种checkpoint：
1.Sharp Checkpoint
2.Fuzzy Checkpoint

Sharp Checkpoint：
数据库关闭时，将所有的脏页刷新回磁盘
(作用：刷新所有脏页，时间：数据库关闭时)

Fuzzy Checkpoint：
数据库运行中，刷新一部分脏页数据回磁盘
场景：
1.Master Thread Checkpoint
2.FLUSH_LRU_LIST Checkpoint
3.Async/Sync Flush Checkpoint
4.Dirty Page too much Checkpoint


1.Master Thread中发生的，每秒or每10秒的速度刷新一定比列，异步的，用户查询不会阻塞

2.保证LRU列表有差不多100个空闲页可使用
[5.6 Purge单独线程之前，必须有100个空闲页可使用，否则会阻塞用户查询]

3.重做日志文件不可用的情况


4.脏页太多，强制刷新
脏页占缓冲池的比列超过设定值时，刷新
参数：innodb_max_dirty_pages_pct
[1.0.x]之前默认90
[1.0.x]之后默认75

#### Master Thread 工作方式

InnoDB存储引擎的主要工作都在Master Thread（后台的单独线程）中进行
具体如何实现？可能存在的问题？

按照版本区分
1.[1.0.x]之前
2.[1.2.x]之前
3.[1.2.x]开始
<!--这块也可以是之前的总结-->

1.[1.0.x]之前
内部由多个循环组成（loop、background loop、flush loop、suspend loop）

一、主循环（loop）分为两个操作
[每秒钟的操作、每10秒钟的操作] -> 尽量保持这个频率
每秒操作
1.固定的redo log刷新到磁盘[即使事务还没有提交] ->解释了再大的事务提交的时间也是很短的
2.合并插入缓冲[一秒内IO次数小于5次，认为IO压力小，执行合并插入缓冲（INSERT BUFFER）]
3.100个脏页刷新[脏页比例达到参数设定值，将100个脏页写入]
4.当前没有用户活动，切换到background loop

每10秒钟操作
1.10s内IO操作小于200（认为有足够的磁盘IO操作能力），将100个脏页刷新到磁盘
2.总会进行，合并插入缓冲
3.同每秒，redo log刷入磁盘
4.进行一次full purge（删除无用的undo页），执行时，每次最多尝试回收20个undo页。 -> 1.update、delete，原先行标记删除，一致性读，行版本信息 2.full purge过程判断是否可以删除，读取之前的undo信息。
5.判断脏页再缓冲池的比例，超过70%刷新100个脏页，没超过，刷新10%的脏页数据。

二、background loop
当前没有用户活动（数据库空闲时）or关闭数据库时，切换到这个循环。
1.删除无用undo页
2.合并20个插入缓冲
3.跳回到主循环
4.跳转到flush loop中（不断刷新100个页直到符合条件）
5.切换到suspend loop，将Master Thread挂起，直到有事件发生，则跳转主循环（loop）

2.[1.2.x]之前
ssd出现后，之前的规定限制了InnoDB对io的性能（写入性能）

磁盘io的吞吐量（innodb_io_capacity）,默认200
1.合并插入缓冲时，吞吐量的5%
2.刷新脏页的数量100->吞吐量的值
（使用ssd时，可以调高innodb_io_capacity的值，直到符合磁盘io的吞吐量）

脏页占缓冲池的90%，默认值的问题
90% -> 75%

自适应的刷新，通过函数（通过redo log的速度）来决定最合适的脏页刷新数量。即，当脏页比例小于参数时，也会刷新一定的脏页。

控制full purge最多回收undo页个数的数值，默认20

3.[1.2.x]版本
刷新脏页操作 分离到单独的Page Cleaner Thread



#### InnoDB关键特性 
1. 插入缓冲（Insert Buffer）
2. 两次写（Double Write）
3. 自适应哈希索引（Adaptive HashIndex）
4. 异步IO（Async IO）
5. 刷新领接页（Flush Neighbor Page）

**插入缓冲（Insert Buffer）**
对于非聚集索引的插入or更新操作
先判断飞聚集索引页是否在缓冲池中，在，直接插入，不在，放入一个Insert Buffer对象中，再以一定的频率和情况进行Insert Buffer和辅助索引页子节点的合并操作。
满足条件：
1.索引是辅助索引
2.索引不是唯一的

一些问题：
1.宕机时（todo）
实际上大量Insert Buffer只是记录，没有真正的合并，因此恢复需要大量时间。
2.写密集时
写密集情况下，插入缓冲会占用大量缓冲池内存，最多达到1/2（可配置）

Change Buffer
[1.0.x]Insert Buffer升级
InnoDB对DML操作-INSERT DELETE UPDATE 都进行缓冲（Insert Buffer、Delete Buffer、Purge Buffer）
使用对象：非唯一的辅助索引
插入->标记删除 ->真正删除
[1.2.x]开始可以控制Change Buffer的最大内存（25，代表1/4，最大50）


Insert Buffer 内部实现
数据结构：B+树
存放在共享表空间中（ibdata1），负责对所用的表的辅助索引进行Insert Buffer。（试图通过独立表空间ibd文件恢复表数据，往往会导致CHECK TABLE失败）
非叶子节点和叶子节点的构造（todo）



Merge Insert Buffer
触发情况
1.辅助索引页被读取到缓冲池
2.Insert Buffer Bitmap页追踪带改辅助索引页已无可用空间时
3.Master Thread

详细内容
1.被读取到时，检查Insert Buffer  Bitmap页，是否有存放在B+树中的记录
2.通过Insert Buffer Bitmap 来追踪每个辅助索引页的可用空间，小于1/32页的空间强制执行
3.每秒or每10秒进行一次（数量会不同），并非按照辅助索引页的排好的顺序，而是随机选择一个页中的space及之后所需要数量的页，目的：在复杂的情况下有更好的公平性。


**两次写**
解决部分写失效问题
在应用重做日志前前，需要一个页的副本，写入失效时，先通过副本还原页，再进行重做。
1.doublewrite  buffer 2M
2.磁盘上共享表空间 2M
对缓冲池脏页刷新时，通过memcpy函数，复制到内存中的doublewrite  buffer，再分两次1M顺序地写入共享表空间的磁盘中，马上调用fsync函数，同步到磁盘上。
1/64


**自适应哈希索引**

访问模式一样
以该模式访问了100次
页通过该模式访问了N次，其中N = 页中记录/16



**异步IO**

IO Merge
[1.1.x]之前代码模拟，之后内核支持，Native AIO，需要libaio库支持（仅支持Linux & windows ，mac暂时不支持）

**刷新领接页** 
当刷新一个脏页，会检测该也所在区的所有页，如果是脏页，则一起刷新
好处：通过AIO将多个IO合并。

固态硬盘有着高效IOPS性能的磁盘建议关闭



#### 启动、关闭和恢复


### 第三章文件




