# 第1篇 基础和应用篇

## Redis的安装

- 使用Docker安装

- 通过GitHub源码编译

- 直接安装apt-get install(Ubuntu)、yum install(RedHat)或者brew install（Mac）。

### 直接安装方式（Mac）

```
brew install redis
./redis-server
redis-cli
```

![avatar](img/1.png)


## 5种基础数据结构

Redis有5种基础数据结构，分别为：string（字符串）、list（列表）、hash（字典）、set（集合）和zset（有序集合）。

### string（字符串）

Redis所有的数据结构都以唯一的字符串作为key，然后通过这个唯一的key值来获取相应的value数据。不同类型的数据结构的差异就在于value的结构不一样。

Redis的字符串是动态字符串，是可以修改的字符串，内部结构的实现类似于Java的ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配。当字符串长度小于1MB时，扩容都是加倍现有的空间。如果字符串长度超过1MB，扩容时只会多扩1MB的空间。需要注意的是字符串最大长度为512MB。

#### 键值对

相当于字典的key和value。

```shell
127.0.0.1:6379> set name jolan
OK
127.0.0.1:6379> get name
"jolan"
127.0.0.1:6379> exists name
(integer) 1
127.0.0.1:6379> del name
(integer) 1
127.0.0.1:6379> get name
(nil)
```

#### 批量键值对

```shell
127.0.0.1:6379> set name1 jolan
OK
127.0.0.1:6379> set name2 joy
OK
127.0.0.1:6379> mget name1 name2 name3
1) "jolan"
2) "joy"
3) (nil)
127.0.0.1:6379> mset name1 boy name2 girl name3 unknown
OK
127.0.0.1:6379> mget name1 name2 name3
1) "boy"
2) "girl"
3) "unknown"
```

#### 过期和set命令扩展

可以对key设置过期时间，到时间会被自动删除。

```shell
127.0.0.1:6379> set name jolan
OK
127.0.0.1:6379> get name
"jolan"
127.0.0.1:6379> expire name 5  #5s后过期
(integer) 1
127.0.0.1:6379> get name  #5s后执行
(nil)
127.0.0.1:6379> setex name 5 jolan  #5s后过期，等价于set+expire
OK
127.0.0.1:6379> get name  #5s后执行
(nil)
127.0.0.1:6379> setnx name jolan  #如果name不存在就执行set创建
(integer) 1
127.0.0.1:6379> setnx name jolan  #因为name已存在所以set创建不成功
(integer) 0
127.0.0.1:6379>
```

#### 计数

如果value值是一个整数，还可以对它进行自增操作。自增是有范围的，他的范围在signed long的最大值和最小值之间，超出这个范围，Redis会报错。

```shell
127.0.0.1:6379> set age 30
OK
127.0.0.1:6379> incr age
(integer) 31
127.0.0.1:6379> incrby age 5
(integer) 36
127.0.0.1:6379> incrby age -5
(integer) 31
127.0.0.1:6379> set errorcode 9223372036854775807  #Long.Max
OK
127.0.0.1:6379> incr errorcode
(error) ERR increment or decrement would overflow
```

### list(列表)

Redis的列表相当于Java语言里面的LinkedList，注意他是链表而不是数组。所以list的插入和删除很快，但是索引定位很慢，时间复杂度分贝是O(1)和O(n)。

Redis的列表结构常用来做异步队列使用。将需要延后处理的任务结构体序列化成字符串，塞进Redis列表，另外一个线程从这个列表中轮询数据进行处理。

#### 队列

利用Redis的list数据结构可以实现队列的效果。list支持从左侧添加院所和从右侧加入元素，从一个方向添加元素，再从另一个方向上获取元素，就可以实现对了。比如下面这个右进左出的list。

```shell
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> llen books
(integer) 3
127.0.0.1:6379> lpop books
"python"
127.0.0.1:6379> lpop books
"java"
127.0.0.1:6379> lpop books
"golang"
127.0.0.1:6379> lpop books
(nil)
```

#### 栈

和队列的远离类似，使用Redis的list数据结构在一个方向上添加和获取元素，就可以实现栈的数据结构了。

```shell
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> rpop books
"golang"
127.0.0.1:6379> rpop books
"java"
127.0.0.1:6379> rpop books
"python"
127.0.0.1:6379> rpop books
(nil)
```

#### list的其他操作

- lindex：相当于Java链表的get(int index)方法，他需要对链表进行遍历，性能随着参数index增大而变差。

- ltrim：ltrim的两个参数start_index和end_index定义了一个区间，在这个区间内的值保留，区间之外的则统统砍掉。可以通过ltrim来实现一个定长的链表。

- lrange：获取start_index和end_index区间的所有值。这三个函数的index都可以为-1，-1表示最后一个元素的位置。

```shell
127.0.0.1:6379> rpush books python java golang
(integer) 3
127.0.0.1:6379> lindex books 1  #效率为O(n)，慎用
"java"
127.0.0.1:6379> lrange books 0 -1  #获取所有元素，慎用
1) "python"
2) "java"
3) "golang"
127.0.0.1:6379> ltrim books 1 -1
OK
127.0.0.1:6379> lrange books 0 -1
1) "java"
2) "golang"
127.0.0.1:6379> ltrim books 1 0  #这实际清空了整个列表， 因为区间范围长度为负
OK
127.0.0.1:6379> llen books
(integer) 0
```

### hash(字典)

Redis的字典相当于Java语言里面的HashMap，它是无序的，实现结构上是“数组+链表”的二位结构。

Redis的字典值只能是字符串。Redis的rehash是采用的是渐进式rehash策略。渐进式rehash会在rehash的同时，保留新旧两个hash结构，然后在后续的定时任务以及hash操作指令中，循序渐进低将旧hash的内容一点点地迁移到新的hash结构中。当搬迁完成了，就会使用心得hash结构取而代之。

hash结构的存储消耗要高于单个的字符串。

```shell
127.0.0.1:6379> hset books java "think in java"  #命令行的字符串如果包含空格，要用引号扩起来
(integer) 1
127.0.0.1:6379> hset books golang "concurrency in go"
(integer) 1
127.0.0.1:6379> hset books python "python cookbook"
(integer) 1
127.0.0.1:6379> hgetall books  #entries()，key和value间隔出现
1) "java"
2) "think in java"
3) "golang"
4) "concurrency in go"
5) "python"
6) "python cookbook"
127.0.0.1:6379> hlen books
(integer) 3
127.0.0.1:6379> hget books java
"think in java"
127.0.0.1:6379> hset books golang "learning go prpgramming"
(integer) 0
127.0.0.1:6379> hget books golang
"learning go prpgramming"
127.0.0.1:6379> hmset books java "effective java" python "learning python" golang "modern gplang programming"
OK
```

同字符串一样，hash结构中的单个子key也可以进行计数，它对应的指令是hincrby。比如给用户的年龄加1。

```shell
127.0.0.1:6379> hset user-jolan age 29
(integer) 1
127.0.0.1:6379> hincrby user-jolan age 1
(integer) 30
```

