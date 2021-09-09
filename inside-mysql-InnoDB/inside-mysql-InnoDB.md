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