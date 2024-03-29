# 存储引擎介绍

* MySQL数据库在5.5.8版本开始，InnoDB存储引擎是默认的存储引擎

* InnoDB提供4种数据库隔离级别，默认为REPEATABLE级别

* 如果没有指定主键，InnoDB存储引擎会为每一行生成一个6字节的ROWID，并以此作为主键

* MyISAM存储引擎的缓冲池只缓存索引文件，而不缓冲数据文件

* NDB存储引擎的特点是把数据全部放在内存中

* Memory将表中数据全部放在内存中，如果数据库重启或崩溃，表中数据都将消失。使用哈希索引。只支持表级锁，并发性能差。MySQL数据库使用Memory存储引擎作为临时表存放查询的中间结果集，如果中间结果集大于Memory存储引擎的表容量设置，有或者中间结果含有TEXT或BLOB列类型字段，则MySQL数据库会把其转换到MyISAM存储引擎表存放到磁盘中，MyISAM引擎由于不缓存数据文件，因此会对查询性能有所影响

* Archive存储引擎只支持INSERT和SELECT操作，使用zlib算法将数据行进行压缩后存储，不是事物安全的，主要提供高速插入和压缩功能

* Federated存储引擎并不存放数据，他只指向一台远程MySQL数据库服务器上的表

* Maria支持缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事物和非事物安全的选项，以及更好的BLOB字符类型的处理性能

# InnoDB存储引擎

* 通过SHOW ENGINES语句查看当前使用么MySQL数据库所支持的存储引擎

* 连接MySQL

  * TCP/IP：MySQL数据库在任何平台都支持的连接方式，也是使用最多的方式。如：mysql -h192.168.0.1 -u david -p

  * 命令管道和共享内存：如果两个需要进程通信的进程在同一台服务器上，可以使用命令管道

  * UNIX域套接字。只能在MySQL客户端和数据库实例在一台服务器的情况下使用

+ 后台线程：后台线程的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下InnoDB能恢复到正常运行状态。
  - Master Thread是非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性
  - IO Thread。InnoDB打量使用AIO处理IO请求来提高数据库性能，IO Thread负责这些IO的回调处理。IO Thread包括write、read、insert buffer和log IO Thread
  - PurgeThread回收已经使用并分配的undo页。
  - Page Cleaner Thread。将之前版本中的脏页的刷新都放入到单独的线程中来完成

- 缓冲池：MySQL是基于磁盘存储的，但是CPU和磁盘之间存在巨大的速度鸿沟，所以内存来弥补磁盘速度慢的影响。
  - 缓冲池配置：SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
  - 缓冲池的缓存的数据类型有：索引页、数据页、undo页、插入缓冲、自适应哈希索引、InnoDB存储的锁信息、数据字典信息等
  - InnoDB引擎缓冲池页的大小默认是16KB
  - LRU List：（从磁盘）新读取到的页，虽然是最新访问的页，但并不直接放入到LRU列表的首部，而是放入到LRU列表的midpoint位置。该位置默认是在LRU列表的5/8处（SHOW VARIABLES  LIKE 'innodb_old_blocks_pct'）。把midpoint之后的列表成为old列表，之前的列表成为new列表。之所以不直接采用最朴素的LRU算法，是因为有可能查询的页虽然是最新的，但是并非热点数据，那么可能将真正需要的热点数据从LRU列表中移除。引入另一个参数进一步管理LRU列表（SET GLOBAL innodb_old_blocks_time = 1000），用来表示页读取到mid位置后需要等待多久才会被加入到LRU列表热端。
  - Free列表：数据库刚刚启动时，LRU列表是空的，这时页都存放在Free列表中，当需要从缓冲池分页时，先从Free列表中查找是否有可用的空闲也，如果有则将该页从Free列表删除，放入到LRU列表中。否则根据LRU算法淘汰末尾页（SHOW ENGINE INNODB STATUS）。其中Buffer pool hit rate用来表示命中率，若小于95%，用户需要观察是否是由于全表扫描引起LRU列表被污染。
  - 脏页：LRU列表被修改后被称为脏页，即缓冲池中的页和磁盘上的页的数据不一致，这是会用CHECKPOINT机制将脏页刷新回磁盘，Flush列表中的页即为脏页。脏页既存在于LRU列表中，也存在于Flush列表中。

- 重做日志缓冲：重做日志缓冲不用太大，一般设置8M即可（SHOW VARIABLES LIKE 'innodb_log_buffer_size';），因为一般每1秒就会把重做日志缓冲中的内容刷新到磁盘。
  - Master Thread每1秒将重做日志刷新到重做日志文件
  - 每个事物提交会把重做日志缓冲刷新到重做日志文件
  - 当重做日志缓冲池剩余空间小于1/2会把重做日志缓冲刷新到重做日志文件

- checkpoint：为避免发生数据丢失，事物提交时会先写重做日志，再修改页。Checkpoint技术解决一下几个问题。
  - 缩短数据库的恢复时间
  - 缓冲池不够用时，将脏页刷新到磁盘
  - 重做日志不可用时，刷新脏页

​        InnoDB存储引擎有两种Checkpoint：

- - Sharp Checkpoint:数据库关闭时，将所有脏页刷新回磁盘
  - Fuzzy Checkpoint:刷新一部分脏页回磁盘
    - Master Thread Checkpoint：以每秒或每十秒将脏页刷新回磁盘
    - FLUSH_LRU_LIST Checkpoint：为保证LRU列表中有一定比例空闲的页，会将LRU列表的尾端页移除，如果这些页有脏页，则将脏页刷新回磁盘
    - Async/Sync Flush Checkpoint:重做日志不可用时，将一些脏页刷新会磁盘
    - Dity Page too much Checkpoint:脏页数量太多，强制进行Checkpoint

- Master Thread（1.0.x前版本）

        主循环（loop）

  - 每秒一次执行的操作：

    - 日志缓冲刷新到磁盘，即使这个事物还没有提交（总是）

    - 合并插入缓冲（可能）：当前IO如果小于每秒5此才执行

    - 最多刷新100个InnoDB的缓冲池中的脏也到磁盘（可能）：根据脏页比例是否大于配置（innodb_max_dirty_pages_pct）
  
    - 如果当前没有用户活动，则切换到background loop(可能)
    
  - 每10秒一次执行的操作：
    
    - 刷新100个脏页到磁盘（可能的情况下）：如果过去10秒的IO操作小于200此，则进行此操作
    
    - 合并之多5个插入缓冲（总是）
    
    - 将日志缓冲刷新到磁盘（总是）
    
    - 删除无用的Undo页（总是）：执行full purge操作，即删除无用的Undo页，每次最多尝试回收20个undo页。undo页用来存放数据修改前的数据，如update和delete的数据。update和delete执行完成后，存储引擎判断是否还需要undo页信息，比如有查询需要读取之前版本的undo信息
    
    - 刷新100个或者10个脏页到磁盘（总是）：脏页比例大于70%则刷新100页，否则刷新10页
    
  
  ​    后台循环（background loop）:当前没有用户活动或数据库关闭，会切换到这个循环
  
  - 删除无用的undo页（总是）
  
  - 合并20个插入缓冲（总是）
  
  - 跳回到主循环（总是）
  
  - 不断刷新100个页知道符合条件（可能，跳转到flush loop中完成）
  
- InnoDB 1.2.x之前的版本优化：增加了几个参数来适应不同服务器内存或IO速度的不同。如前面所述的每次刷新200个脏页等。

  - 每秒刷新到磁盘的脏页比例innodb_max_dirty_pages_pct可配置化，默认是75%
  - innodb_adaptive_flushing（自适应刷新脏页）：根据重做日志的生成速度决定这个参数

- InnoDB 1.2.x版本的优化：对于刷新脏页的操作，从Master Thread线程分离到单独的Page Cleaner Thread，从而减轻了Mater Thread的工作，同时进一步提高了系统的并发性

- Insert Buffer：在进行插入操作时，如果需要插入非聚集索引，就要离散的访问非聚集索引页，所以先判断非聚集索引是否在缓冲池，如果在直接更新，否则先放入Insert Buffer中，再以一定的频率和辅助索引页节点进行合并（merge）。Insert Buffer需要满足两个条件，一个是索引是辅助索引（secondary index），一个是索引不能是唯一（unique）的。Insert Buffer的大小可以用参数调整：IBUF_POOL_SIZE_PER_MAX_SIZE。

- Insert Buffer是一个全局的B+树，存储引擎中所有的辅助索引都在这个B+树中维护。后续有对Insert Buffer进行了升级，delete和update都可以进行缓冲，也就是1.0.x版本开始引入的change Buffer。

- Merge Insert Buffer，发生一下几种情况，会触发存储引擎将Insert Buffer合并到辅助索引页
  - 当辅助索引页被读取到缓冲池中时
  - Insert Buffer Bitmap页追踪到该辅助索引页已无可用空间时
  - Master Thread线程中每秒或每10秒会进行一次Merge Insert Buffer的操作

- 两次写：doublewrite带给InnoDB存储引擎的是数据页的可靠性。如数据在从缓冲池刷新到磁盘时发生宕机，那么数据将会丢失。所以引入两次写，即数据从缓冲池写入磁盘前先写入doublewrite buffer，doublewrite在分成2此，每次1M将数据写入到doublewrite所在的共享表空间，此时的写入是连续的，再从缓冲池写入到磁盘，此时的写入是离散的。如果写入磁盘时候发生宕机，则从doublewrite共享表空间获取一份页的副本，再应用重做日志。

- 自适应哈希索引：数据库根据热点访问的数据，自动简历哈希索引。建立哈希索引必须是一直以某个模式进行访问，如where a = xxx。如果交替使用where a =  xxx和where a = xxx and b = xxx则不会建立哈希索引。

- 异步IO：如果访问的也是连续的，会合并多个IO请求到一次IO请求中去。

- 刷新邻接页

# 文件

- 参数文件：MySQL启动时指定的参数的文件。mysql--help | grep my.cnf。select * from GLOBAL_VARIABLES WHERE VARIABLE_NAME LIKE 'innodb_buffer%';参数类型包括动态参数和静态参数。动态参数可以用过set命令动态指定，静态参数只能在数据库启动时生效。

- 错误日志文件：SHOW VARIBLES LIKE 'log_error';

- 慢查询日志：SHOW VARIBLES LIKE 'long_query_time'; SHOW VARIBLES LIKE 'log_show_queries';  得到执行时间最长的10条SQL语句：mysqldumpslow -s al -n 10 jolan.log;   SELECT * FROM mysql.slow_log;

- 二进制日志（binary log）：记录了MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作。二进制日志的作用：恢复、复制、审计。

- InnoDB所有数据都被逻辑地存放在一个空间中，称之为表空间。表空间又有段（segment）、区（extent）、页（page）组成

- 区是由连续的页组成的空间，在任何情况下每个区的大小都为1MB。InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。在InnoDB1.0.X之后引入了压缩页，所以一个区也可能有512、256、128个页。

- InnoDB的行记录格式：略

# 表

- 创建触发器的命令是CREATE TRIGGER，只有具备Super权限的MySQL数据库用户才可以执行这条命令。最多可以为一个表建立6个触发器，即分别为INSERT、UPDATE、DELETE的BEFORE和AFTER各一个。

- 视图定义中的WITH CHECK OPTION是针对对于可更新的视图的，即更新的值是否需要检查。对于不满足视图条件的，将会抛出一个异常，不允许视图中的数据更新。

- 分区是MySQL的功能，不是存储引擎的功能，但是并非所有存储引擎都支持分区。MySQL只支持水平分区（按行），不支持垂直分区（按列）。使用SHOW VARIABLES LIKE '%partition%'查看是否开启了分区功能。

    - RANGE分区：行数据基于属于一个给定连续区间的列值被放入分区。MySQL5.5开始支持。
      - 启用分区后，表不再有一个idb文件组成，而是由建立分区时的各个分区idb文件组成。如t#P##p0.idb，t#P##p1.idb
      - CREATE TABLE t(id INT) ENGINE=INNODB PARTITION BY RANGE (id) (PARTITION p0 VALUES LESS THAN (10), PARTITION p1 LESS THAN (20))
      - 通过SELECT * FROM information_schema.PARTITIONS WHERE table_schema=database() AND table_name = ''来查询分区使用情况。TABLES_ROWS列反映了每个分区中记录的数量。PARTITION_METHOD表示分区类型。
      - 当插入一个不再分区中定义的值时，MySQL数据库会抛出一个异常。可加入一个最大值的分区：ALTER TABLE t ADD PARTITION (partition p2 values less than maxvalue)
      - RANGE分区主要用于日期列的分区
      - PANGE分区的查询，优化器只能对YEAR()，TO_DAYS()，TO_SECONDS()，UNIX_TIMESTAMP()这类函数进行优化选择。如CREATE TABLE sales(money INT UNSIGEND NOT NULL, date DATE) ENGINE=INNODB PARTITION BY RANGE (TO_DAYS(date))
      
    - LIST分区：和RANGE分区类似，只是LIST分区面相的是离散的值
      
      - CREATE TABLE t(id INT) ENGINE=INNODB PARTITION BY LIST(id) (PARTITION p0 VALUES IN(1,3,5,7,9), PARTITION p1 VALUES IN (0,2,4,6,8)
      
    - HASH分区：根据用户自动以的表达式的返回值来进行分区，返回值不能为负数。
      - CREATE TABLE t_hash(a INT, b DATETIME) ENGINE = INNODB PARTITION BY HASH (YEAH(b)) PARTITIONS 4;如果插入一个列b为2010-04-01的记录到t_hash表，那么保存该条记录的分区如下 MOD(YEAH('2010-04-01'), 4)=>MOD(2010, 4)=>2
      - MySQL数据库还支持一种称为LINEAR HASH的分区，它使用一个更加复杂的算法，创建时把 BY HASH修改为 BY LINEAR HASH即可。LINEAR HASH分区的有点在于速度更快，缺点是数据分部可能不太均衡
      
    - KEY分区：根据MySQL数据库提供的哈希函数来进行分区。
    
    - CLOUMNS分区：前面四种分区的条件是数据必须是整形（INTEGER），如果不是整形，那应该通过函数将其转换为整形，如YEAR()，TO_DAYS()等。COLUMNS分区支持所有整形、日期类型、字符串类型。关键字：PARTITION BY RANGE COLUMNS (b)，PARTITION BY LISTCOLUMNS (b)。
    
    - MySQL数据库允许在RANGE和LIST的分区上再进行HASH或KEY子分区
    
    - 分区对于NULL值的处理
    
      - 对于RANGE分区，如果向分区列插入了NULL值，则MySQL数据库会将该值放入最左边的分区
    
      - 在LIST分区下要使用NULL值，则必须显示地指出哪个分区中放入NULL值，否则会报错
    
      - HASH和KEY分区的任何分区函数都会将含有NULL值的记录返回为0
    
- MySQL5.6开始，支持分区或子分区中的数据与另一个非分区表中的数据进行交换：ALTER TABLE ... EXCHANGE PARTITION。ALTER TABLE e EXCHANGE PARTITION p0 WITH TABLE e2;

# 索引与算法

- 索引与算法

  - B+树索引并不能找到一个给定键值的具体行。B+树索引能找到的知识被查找数据行所在的页，然后数据库通过把页读入到内存，再在内存中进行查找，最后得到要查找的数据。

    ``` html
    B+树的新增操作
    ```

  - Leaf Page未满，Index Page未满：直接将记录插入到叶子节点

  - Leaf Page满，Index Page未满

    - 拆分Leaf Page
    - 将（拆分后）中间的节点放入到Index Page中
    - 将小于中间节点的记录放左边
    - 大于或等于中间节点的记录放右边

  - Leaf Page满，Index Page满

    - 拆分Leaf Page

    - 将（拆分后）小于中间节的记录放左边

    - 大于或等于中间节点的记录放右边

    - 拆分Index Page

    - 将（拆分后）小于中间节的记录放左边

    - 大于中间节点的记录放右边

    - 中间节点放入上一层Index Page（比如2层树高变为3层树高）

  - 如果Leaf Page已满，但其他左右兄弟节点没有满的情况下，B+树不会急于去做拆分页，而是会先做旋转，也就是把新增的节点平衡到其他兄弟节点上去

    ```html
    B+树的删除操作：B+树使用填充因子（fill factor）来控制树的删除变化，50%是填充因子可设的最小值。
    ```

  - 叶子节点不小于填充因子，中间节点不小于填充因子：直接将记录从叶子节点删除，如果该节点还是Index Page的节点，用该节点的右节点代替

  - 叶子节点小于填充因子，中间节点不小于填充因子：合并叶子节点和它的兄弟节点，同事更新Index Page

  - 叶子节点小于填充因子，中间节点小于填充因子

    - 合并叶子节点和它的兄弟节点
    - 更新Index Page
    - 合并Index Page和它的兄弟节点

- B+索引在数据库中有个特点是高扇出性，因此在数据库中，B+树的高度一般都在2~4层，这也就是说查找某一个键值的记录时最多只需要2到4次IO，2~4次IO意味着查询时只需要0.02到0.04秒。数据库中的B+树索引可分为聚集索引和辅助索引，但是不管是哪种索引，其内部都是B+树的，即高度平衡的，叶子节点存放着所有的数据。聚集索引与辅助索引不同的是，叶子节点存放的是否是一整行的信息。

- 聚集索引：聚集索引（clustered index）是按照每张表的主键构造一颗B+树，同时叶子节点中存放的即为整张表的行记录数据，也将聚集索引的叶子节点成为数据页。聚集索引的这个特性决定了索引组织表中数据的是索引的一部分。同B+树数据结构一样，每个数据页都通过一个双向链表来进行链接。由于实际的数据页只能按照一颗B+树进行排序，因此每张表只能拥有一个聚集索引。在多数情况下，查询优化器倾向于采用聚集索引。因为聚集索引能够在B+树索引的叶子节点上直接找到数据。此外，由于定义了数据的逻辑顺序，聚集索引能够特别快地访问范围的查询。查询优化器能够快速发现某一段范围的数据页需要扫描。聚集索引的另一个好处是，它对主键的排序查找和范围查找速度非常快。叶子节点的数据就是用户要查询的数据。

- 辅助索引：对于辅助索引（Secondary Index，也成非聚集索引），叶子节点并不包括行记录的全部数据。叶子节点除了包含键值以为，每个叶子节点的索引行中还包含了一个书签（bookmark）。该书签是用来告诉InnoDB存储引擎哪里可以找到与索引想对应的行数据。由于InnoDB存储引擎是索引组织表，因此InnoDB存储引擎的辅助索引的书签就是相应行数据的聚集索引键。举例来说，如果在一颗高度为3的辅助索引树中查找数据，那需要对这颗辅助索引树遍历3遍找到指定的主键，如果聚集索引树的高度也是3，那么还需要对聚集索引树进行3次查找，最终找到一个完整的行数据所在页，因此一共需要6次逻辑IO访问以得到最终的一个数据页。

- 使用show index from 表名;查看索引情况，列的含义如下（主要）

  - Collation：列以什么方式存储在索引中。可以是A或NULL。B+树索引总是A，如果使用了Heap存储引擎，并且建立了Hash所以这里显示NULL
  - Cardinality：非常关键的值，表示索引中唯一值的数目的估计值。Cardinality / 表的行数应尽可能接近1，如果非常小，那么用户需要考虑是否可以删除此索引。优化器会根据这个值来判断是否使用这个索引，但是这个值并不是实时更新的，如果需要更新索引Cardinality的信息，可以使用ANALYZE TABLE命令。
  
- MySQL在5.5版本之前，数据库对索引的添加或者删除的这类DDL操作，MySQL数据库的操作过程为：首先创建一张新表的临时表，表结构为通过ALTER TABLE新定义的结果；然后把原表中的数据导入到临时表；接着删除原表；最后把临时表重命名为原来的表名。若用户对一张大表进行索引的添加和删除操作，那么这会需要很长的时间。更关键的是，若有大量事物需要访问正在被修改的表，这意味着数据库服务不可用。InnoDB存储引擎从1.0.x版本开始支持一种称Fast index Creation的索引创建方式，简称FIC。对于辅助索引的创建，InnoDB存储引擎会对创建索引的表加一个S锁。由于FIC在索引的创建的过程中对表加上了S锁，因此在创建的过程中只能对该表进行读操作，若有大量的事物需要对目标表进行写操作，那么数据库的服务同样不可用。此外，FIC方式只限定于辅助索引，对于主键的创建和删除同样需要重建一张表。相对与FIC，还有Online DDL修改索引的方式（略）。

- B+树索引的使用：OLTP应用这种情况，B+树索引建立后，对该索引的使用应该只是通过该索引取得表中少部分的数据。这时建立B+树索引才是有意义的，否则即使建立了索引，优化器也可能选择不使用索引。在OLAP应用中，都需要访问表中大量的数据，根据这些数据来产生查询结果，这些查询多事面相分析的查询，目的是为决策者提供支持。因此在OLAP中索引的添加根据的应该是宏观的信息，而不是围观，因为最终要得到的结果是提供给决策者的。对于OLAP中的复杂查询，要涉及多张表之间的联结操作，因此索引的添加依然是有意义的。

- 联合索引：对于查询SELECT * FROM TABLE WHERE a = xxx AND b = xxx，显然是可以使用（a，b）这个联合索引的。对于单个a列查询SELECT * FROM TABLE WHERE a = xxx，也可以使用（a，b）索引。但对于b列的查询SELECT * FROM TABLE WHERE b = xxx，则不可以使用这颗B+树索引。联合索引（a，b）其实是根据a、b进行排序的，因此SELECT ... FROM TABLE WHERE a=xxx order by b也是可以直接使用这个联合索引的。但是对于SELECT ... FROM TABLE WHERE a=xxx order by c就不能使用这个联合索引。

- 覆盖索引：查询的所有列都在辅助索引上的索引叫覆盖索引。由于辅助索引叶子节点存储的是索引和聚集索引的书签，所以可能需要回表操作，但是如果需要查询的列全部在辅助索引上，则不需要回表操作。使用explain解释时，Extra字段为Using index表示使用了覆盖索引。

- InnoDB存储引擎会行级别上对表数据上锁

# 锁

- 锁

  - lock的对象是事物，用来锁定的是数据库中的对象，如表、页、行。并且一般lock的对象仅在事物commit或rollback后进行释放（不同事物隔离级别释放的时间可能不同）。![avatar](lock_and_lanch.png "lock与latch区别")
  
  - 对于InnoDB存储引擎中的latch，可以通过命令SHOW ENGINE INNODB MUTEX来进行查看
  
  - lock信息查看：可以通过SHOW ENGINE INNODB STATUS及information_schema架构下的表INNODB_TRX、INNODB_LOCKS、INNODB_LOCK_WAITS来观察锁的信息。
  
  - InnoDB存储引擎实现了如下两种标准的行级锁：
  
    - 共享锁（S Lock），允许事物读一行数据
  
    - 排他锁（X Lock），允许事物删除或更新一行数据
  
      ``` html
      如果一个事物T1已经获得了行r的共享锁，那么另外的事物T2可以立即获取r的共享锁，因为读取并没有改变r的数据，称这种情况为锁兼容（Lock Compatible）。但若有其他的事物T3想获得行r的排他锁，则其必须等待事物T1、T2释放行r的共享锁——这种情况称为锁不兼容。X锁与任何的锁都不兼容，而S锁仅和S锁兼容。需要特别注意的是，S和X都是行锁，兼容是指对统一记录（row）锁的兼容情况。
      ```
  
  - InnoDB存储引擎支持意向锁设计比较简练，其意向锁即为表级别的锁。
  
    - 意向共享锁（IS Lock），事物想要获得一张表中某几行的共享锁
    - 意向排他锁（IX Lock），事物想要获得一张表中某几行的排他锁![avatar](Innodb_lock_compatibility.png "InnDB存储引擎中锁的兼容性")
    
  - 通过SHOW ENGINE INNODB STATUS命令来查看当前锁清秋的信息
  
  - select * from information_schema.INNODB_TRX;
  
    ``` html
    1.trx_id：唯一事务id号，只读事务和非锁事务是不会创建id的。
    2.TRX_WEIGHT：事务的高度，代表修改的行数（不一定准确）和被事务锁住的行数。为了解决死锁，innodb会选择一个高度最小的事务来当做牺牲品进行回滚。已经被更改的非交易型表的事务权重比其他事务高，即使改变的行和锁住的行比其他事务低。
    3.TRX_STATE：事务的执行状态，值一般分为：RUNNING, LOCK WAIT, ROLLING BACK, and COMMITTING.
    4.TRX_STARTED：事务的开始时间
    5.TRX_REQUESTED_LOCK_ID:如果trx_state是lockwait,显示事务当前等待锁的id，不是则为空。想要获取锁的信息，根据该lock_id，以innodb_locks表中lock_id列匹配条件进行查询，获取相关信息。
    6.TRX_WAIT_STARTED：如果trx_state是lockwait,该值代表事务开始等待锁的时间；否则为空。
    7.TRX_MYSQL_THREAD_ID：mysql线程id。想要获取该线程的信息，根据该thread_id，以INFORMATION_SCHEMA.PROCESSLIST表的id列为匹配条件进行查询。
    8.TRX_QUERY：事务正在执行的sql语句。
    9.TRX_OPERATION_STATE：事务当前的操作状态，没有则为空。
    10.TRX_TABLES_IN_USE：事务在处理当前sql语句使用innodb引擎表的数量。
    11.TRX_TABLES_LOCKED：当前sql语句有行锁的innodb表的数量。（因为只是行锁，不是表锁，表仍然可以被多个事务读和写）
    12.TRX_LOCK_STRUCTS：事务保留锁的数量。
    13.TRX_LOCK_MEMORY_BYTES：在内存中事务索结构占得空间大小。
    14.TRX_ROWS_LOCKED：事务行锁最准确的数量。这个值可能包括对于事务在物理上存在，实际不可见的删除标记的行。
    15.TRX_ROWS_MODIFIED：事务修改和插入的行数
    16.TRX_CONCURRENCY_TICKETS：该值代表当前事务在被清掉之前可以多少工作，由 innodb_concurrency_tickets系统变量值指定。
    17.TRX_ISOLATION_LEVEL：事务隔离等级。
    18.TRX_UNIQUE_CHECKS：当前事务唯一性检查启用还是禁用。当批量数据导入时，这个参数是关闭的。
    19.TRX_FOREIGN_KEY_CHECKS：当前事务的外键坚持是启用还是禁用。当批量数据导入时，这个参数是关闭的。
    20.TRX_LAST_FOREIGN_KEY_ERROR：最新一个外键错误信息，没有则为空。
    21.TRX_ADAPTIVE_HASH_LATCHED：自适应哈希索引是否被当前事务阻塞。当自适应哈希索引查找系统分区，一个单独的事务不会阻塞全部的自适应hash索引。自适应hash索引分区通过 innodb_adaptive_hash_index_parts参数控制，默认值为8。
    22.TRX_ADAPTIVE_HASH_TIMEOUT：是否为了自适应hash索引立即放弃查询锁，或者通过调用mysql函数保留它。当没有自适应hash索引冲突，该值为0并且语句保持锁直到结束。在冲突过程中，该值被计数为0，每句查询完之后立即释放门闩。当自适应hash索引查询系统被分区（由 innodb_adaptive_hash_index_parts参数控制），值保持为0。
    23.TRX_IS_READ_ONLY：值为1表示事务是read only。
    24.TRX_AUTOCOMMIT_NON_LOCKING：值为1表示事务是一个select语句，该语句没有使用for update或者shared mode锁，并且执行开启了autocommit，因此事务只包含一个语句。当TRX_AUTOCOMMIT_NON_LOCKING和TRX_IS_READ_ONLY同时为1，innodb通过降低事务开销和改变表数据库来优化事务。
    ```
    
   - select * from information_schema.INNODB_LOCKS;
  
     ``` htm
     1.Lock_id:锁id
     2.Lock_trx_id:拥有锁的事务id，可以和Innodb_trx表join得到事务的详细信息。
     3.Lock_mode:锁的模式
     4.Lock_type:锁的类型。Record代表行级锁，table表示表级锁
     5.lock_table:被锁定的或者包含锁定记录的表的名称
     6.Lock_index:当lock_type=’record’时，表示索引的名称；否则为null
     7.Lock_space:当lock_type=’record’时，表示锁定行的表空间id；否则为null
     8.Lock_page:当lock_type=’record’时，表示锁定行的页号；否则为null
     9.Lock_rec:当lock_type=’record’时，表示一堆页面中锁定行的数量，即被锁定的记录号；否则为null
     10.Lock_data:当lock_type=’record’时，表示锁定行的主键；否则为null
     ```
  
- 一致性非锁定读：一致性的非锁定度（consistent nonlocking read）是指InnoDB存储引擎通过多版本控制（multi versioning）的方式来读取当前执行时间数据库中行的数据。如果读取的行正在执行DELETE或UPDATE操作，这时读取操作不会因此去等待行上锁的释放。相反地，InnoDB存储引擎会去读取行的一个快照数据。该实现是通过undo段来完成。而undo用来在事物中回滚数据，因此快照数据本身没有额外的开销。一个行记录可能有不止一个快照数据，一般称这种技术为行多版本技术。由此带来的并发控制，成为多版本并发控制（Multi Version Concurrency Contorl，MVCC）。
  
- 在事物隔离级别READ COMMITED和REPEATABLE READ（InnoDB存储引擎默认事物隔离级别）下，InnoDB存储引擎使用非锁定的一致性读。在READ COMMITED事物隔离级别下，对于快照数据，非一致性读总是读取被锁定行的最新一份快照数据。而在REPEATABLE READ事物隔离级别下，对于快照数据，非一致性读总是读取事物开始时的行数据版本。

    ```html
    1.线程A开启事物，执行select id from t where id = 1，结果显示为1
    2.线程B开启事物，执行update t set id = 3 where id = 1
    3.在READ COMMITED和REPEATABLE READ两种事物隔离级别下，分别执行select id from t where id = 1，结果显示为1
    4.提交线程B的事物
    5.在READ COMMITED事物隔离级别下，执行select id from t where id = 1，没有结果集（因为此时id已经被修改为3）
    5.在REPEATABLE READ事物隔离级别下，执行select id from t where id = 1，结果显示为1。因为REPEATABLE READ事物隔离级，非一致性读总是读取事物开始时的行数据版本
    ```

- 一致性锁定读：某些情况下，用户需要显示地对数据库读取操作进行加锁以保证数据逻辑的一致性。而这要求数据库支持加锁语句，即时是对SELECT的制度操作。InnoDB存储引擎对于SELECT语句支持两种一致性的锁定读（locking read）的操作。SELECT ... FOR UPDATE对读取的行记录加一个X锁，其他事物不能对已锁定的行加上任何锁。SELECT ... LOCK IN SHARE MODE对读取的行记录加一个S锁，其他事物可以被锁定的行加S锁，但是如果加X锁，则会被阻塞。SELECT ... FOR UPDATE，SELECT ... LOCK IN SHARE MODE必须在一个事物中，当事物提交了，锁也就释放了。

  ```html
  SELECT ... FOR UPDATE
  SELECT ... LOCK IN SHARE MODE
  ```

- 自增长与锁：对于自增长值的列的并发插入性能较差，事物必须等待前一个插入的完成（虽然不用等待事物的完成）。其次，对于INSERT...SELECT大大数据量的插入会影响插入的性能，因为另一个事物中的插入会被阻塞。InnoDB从MySQL 5.1.22版本开始，引入了innodb_autoinc_lock_mode来控制自增长模式。

- 外键和锁：对于一个列的外键，如果没有显示地对这个列加索引，InnoDB存储引擎自动对其加一个索引，因为这样可以避免表锁。对于外键值的插入或更新，使用的是SELECT...LOCK IN SHARE MODE方式，即主动对父表加一个S锁。

- InnoDB有三种行锁的算法：

  - Record Lock：单个行记录上的锁

  - Gap Lock：间隙锁，锁定一个范围，但不包含记录本身

  - Next-Key Lock：Gap Lock+Record Lock，锁定一个范围，并且锁定记录本身

- 在Next-Key Lock算法下，InnoDB对于行的查询都是采用这种锁定算法。例如一个索引有10，11，13，20这四个值，那么该索引可能被Next-key Lock的区间为：(-∞,10]  (10,11]  (11,13]  (13,20]   (20,+∞)。

    - 当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围。
    
      | 时间 | 会话A                                 | 会话B                          |
      | ---- | ------------------------------------- | ------------------------------ |
      | 1    | BEGIN;                                |                                |
      | 2    | SELECT * FROM t WHERE a=5 FOR UPDATE; |                                |
      | 3    |                                       | BEGIN;                         |
      | 4    |                                       | INSERT INTO t SELECT 4;        |
      | 5    |                                       | COMMIT;<br />#成功，不需要等待 |
      | 6    | COMMIT                                |                                |
    
    - 若是辅助索引，则情况完全不同。辅助索引加上的是Next-Key Lock，并且InnoDB存储引擎还会对辅助索引下一个键值加上gap lock。举例如下，先创建测试表z
    
      ```sql
      CREATE TABLE z (a INT, b INT, PRIMARY KEY(a), KEY(b));
      INSERT INTO z SELECT 1,1;
      INSERT INTO z SELECT 3,1;
      INSERT INTO z SELECT 5,3;
      INSERT INTO z SELECT 7,6;
      INSERT INTO z SELECT 10,8;
      ```
    
      在会话A中执行sql语句：SELECT * FROM z WHERE b = 3 FOR UPDATE，此时对辅助索引加上的是Next-key Lock，锁定的范围是(1,3)，并且对下一个范围加间隙锁。所以此时执行如下几个sql都会阻塞：
    
      ```sql
      #在会话A中执行的SQL语句已经对聚集索引中的a=5的值加上X锁，因此会被阻塞
      SELECT * FROM z WHERE a = 5 LOCK IN SHARE MODE;
      #插入主键4没有问题，但是插入的辅助索引2在锁定的范围(1,3)中，因此会被阻塞
      INSERT INTO z SELECT 4,2;
      #插入主键6没有锁定，5也不在范围(1,3)之间，但插入的值5在另一个锁定的范围(3,6)中，因此会被阻塞
      INSERT INTO z SELECT 6,5;
      ```
    
      下面的SQL语句不会被阻塞，可以立即执行
    
      ```SQL
      INSERT INTO z SELECT 8,6;
      INSERT INTO z SELECT 2,0;
      INSERT INTO z SELECT 6,7;
      ```
    
- 在InnoDB存储引擎中，对于INSERT的操作，其会检查插入记录的下一条记录是否被锁定，若已经被锁定，则不允许查询。
  
- 阻塞：因为不同锁之间的兼容性关系，在有些时刻一个事物中的锁需要等待另一个事物中的锁释放它所占用的资源，这就是阻塞。阻塞并不是一件坏事，其是为了确保事物可以并发且正常地运行。在InnoDB存储引擎中，参数innodb_lock_wait_timeout用来控制等待的时间（默认是50秒），innodb_rollback_on_timeout用来设定是否在等待超时对进行中的事物进行回滚操作（默认是OFF，代表不回滚）。参数innodb_lock_wait_timeout是动态的，可以在MySQL数据库运行时进行调整：SET @@innodb_lock_wait_timeout=60;而innodb_rollback_on_timeout是静态的，不可在启动时进行修改。

- 死锁：死锁是指两个或两个以上的事物在执行过程中，因争夺资源而造成的一种互相等待的现象。若无外力作用，事物都将无法推进下去。

    - 解决死锁最简单的一种方法是超时，即当两个事物互相等待时，当一个等待时间超过设置的某一阈值时，其中一个事物进行回滚，另一个等待的事物就能进行。在InnoDB存储引擎中，参数innodb_lock_wait_timeout用来设置超时的时间。

    - 超时机制虽然简单，但是其仅通过超时后对事物进行回滚的方式处理，或者说其是通过FIFO的顺序选择回滚对象。但若超时的事物所占权重比较大，如事物操作更新了很多行，占用了较多的undo log，这是采用FIFO的方式，就显得不合适了，因为回滚了这个事物的时间相对另一个事物所占用的时间可能会很多。因此，除了超时机制，当前数据库还都普遍采用wait-for-graph（等待图）的方式来进行死锁检测。InnoDB存储引擎也采用这种方式。wait-for-graph要求数据库保存一下两种信息：1.锁的信息链表；2.事物等待链表。通过上述链表可以构造一个图，而在这个图中若存在回路，就代表存在死锁，因此资源之间互相等待。在wait-for-graph中，事物为图中的节点，而在图中，事物T1指向T2边的定义为：1.事物T1等待事物T2所占用的资源；2.事物T1最终等待T2所占用的资源，也就是事物之间在等待相同的资源，而事物T1发生在事物T2的后面。

    - wait-for-graph是一种较为主动的死锁检测机制，在每个事物请求并发生等待时都会判断是否存在回路，若存在则有死锁，通常来说InnoDB存储引擎选择回滚undo量最小的事物。

    - 系统中任何一个事物发生死锁的概率 ≈ n<sup>2</sup>r<sup>4</sup>/4R<sup>2</sup>。其中n、r、R的含义是：

      - 系统中事物的数量（n），数量越多发生死锁的概率越大

      - 每个事物操作的数量（r），每个事物操作的数量越多，发生死锁的概率越大

      - 操作数量的几个（R），越小则发生死锁的概率越大

  - 死锁的示例：
  
    | 时间 | 会话A                                            | 会话B                                                        |
    | ---- | ------------------------------------------------ | ------------------------------------------------------------ |
    | 1    | BEGIN;                                           |                                                              |
    | 2    | SELECT * FROM t WHERE a=1 FOR UPDATE;            | BEGIN;                                                       |
    | 3    |                                                  | SE;ECT * FROM t WHERE a=2 FOR UPDATE;                        |
    | 4    | SELECT * FROM t WHERE a=2 FOR UPDATE;<br />#等待 |                                                              |
    | 5    |                                                  | SELECT * FROM t WHERE a=1 FOR UPDATE;<br />ERROR 1213(40001):Deadlock found when trying to get lock;try restarting transaction |
  
  - 发现死锁后，InnoDB存储引擎会马上回滚一个事物，这点是需要注意的。因此如果应用程序中捕获了1213这个错误，其实并不需要对其进行回滚

# 事物

- 事物的分类
  - 扁平事物：在扁平事物中，所有操作都处于同一层次，其由BEGIN WORK开始，由COMMIT WORK或ROLLBACK WORK结束，其间的操作是原子的，要么都执行，要么都回滚
    - BEGIN WORK;Operation 1...Operation n;Commit WORK;
    - BEGIN WORK;Operation 1...Operation n;ROLLBACK WORK;
    - BEGIN WORK;Operation 1...Operation n;因外界原因回滚，如超时等;
  - 带有保存点的扁平事物：除扁平事物，允许在事物执行过程中回滚到同一事物中较早的一个状态。保存点用来通知系统应该记住事物当前的状态，以便当之后发生错误后时，事物能回到保存点当时的状态。
  - 链事物：可视为保存点模式的一种变种。带有保存点的扁平事物，当发生系统崩溃时，所有的保存点豆浆消失。而链式事物的思想是：在提交一个事物时，释放不需要的数据对象，将必要的处理上下文隐式地传给下一个要开始的事物。注意，提交事物操作和开始下一个事物操作将合并为一个原子操作。这意味着下一个事物将看到上一个事物的结果，将好像在一个事物中进行的一样。
  - 嵌套事物：MySQL不支持
  - 分布式事物：通畅是一个在分布式环境下运行的扁平事物，因此需要根据数据库所在位置访问网络中的不同节点。