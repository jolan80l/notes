* MySQL数据库在5.5.8版本开始，InnoDB存储引擎是默认的存储引擎

* InnoDB提供4种数据库隔离级别，默认为REPEATABLE级别

* 如果没有指定主键，InnoDB存储引擎会为每一行生成一个6字节的ROWID，并以此作为主键

* MyISAM存储引擎的缓冲池只缓存索引文件，而不缓冲数据文件

* NDB存储引擎的特点是把数据全部放在内存中

* Memory将表中数据全部放在内存中，如果数据库重启或崩溃，表中数据都将消失。使用哈希索引。只支持表级锁，并发性能差。MySQL数据库使用Memory存储引擎作为临时表存放查询的中间结果集，如果中间结果集大于Memory存储引擎的表容量设置，有或者中间结果含有TEXT或BLOB列类型字段，则MySQL数据库会把其转换到MyISAM存储引擎表存放到磁盘中，MyISAM引擎由于不缓存数据文件，因此会对查询性能有所影响

* Archive存储引擎只支持INSERT和SELECT操作，使用zlib算法将数据行进行压缩后存储，不是事物安全的，主要提供高速插入和压缩功能

* Federated存储引擎并不存放数据，他只指向一台远程MySQL数据库服务器上的表

* Maria支持缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事物和非事物安全的选项，以及更好的BLOB字符类型的处理性能

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

- 参数文件：MySQL启动时指定的参数的文件。mysql--help | grep my.cnf。select * from GLOBAL_VARIABLES WHERE VARIABLE_NAME LIKE 'innodb_buffer%';参数类型包括动态参数和静态参数。动态参数可以用过set命令动态指定，静态参数只能在数据库启动时生效。

- 错误日志文件：SHOW VARIBLES LIKE 'log_error';

- 慢查询日志：SHOW VARIBLES LIKE 'long_query_time'; SHOW VARIBLES LIKE 'log_show_queries';  得到执行时间最长的10条SQL语句：mysqldumpslow -s al -n 10 jolan.log;   SELECT * FROM mysql.slow_log;

- 二进制日志（binary log）：记录了MySQL数据库执行更改的所有操作，但是不包括SELECT和SHOW这类操作。二进制日志的作用：恢复、复制、审计。

- InnoDB所有数据都被逻辑地存放在一个空间中，称之为表空间。表空间又有段（segment）、区（extent）、页（page）组成

- 区是由连续的页组成的空间，在任何情况下每个区的大小都为1MB。InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。在InnoDB1.0.X之后引入了压缩页，所以一个区也可能有512、256、128个页。

- InnoDB的行记录格式：略

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

- 