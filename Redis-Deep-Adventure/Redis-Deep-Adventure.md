# 写在前面

Redis官网：https://redis.io/

Redis源码：https://github.com/redis/redis

Redis在线测试：http://try.redis.io/

Redis命令参考：http://doc.redisfans.com/

Reids教程：https://www.runoob.com/redis/redis-tutorial.html

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

### Redis版本查看

redis-server --version或redis-server -v

```shell
admin ~ » redis-server --version                    1 ↵
Redis server v=6.2.6 sha=00000000:0 malloc=libc bits=64 build=c6f3693d1aced7d9
admin ~ » redis-server -v
Redis server v=6.2.6 sha=00000000:0 malloc=libc bits=64 build=c6f3693d1aced7d9
```


## 5种基础数据结构

Redis有5种基础数据结构，分别为：string（字符串）、list（列表）、hash（字典）、set（集合）和zset（有序集合）。

### string（字符串）

Redis所有的数据结构都以唯一的字符串作为key，然后通过这个唯一的key值来获取相应的value数据。不同类型的数据结构的差异就在于value的结构不一样。

Redis的字符串是动态字符串，是可以修改的字符串，内部结构的实现类似于Java的ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配。当字符串长度小于1MB时，扩容都是加倍现有的空间。如果字符串长度超过1MB，扩容时只会多扩1MB的空间。需要注意的是字符串最大长度为512MB。

- 设置一个string：set key value [EX seconds|PX milliseconds|EXAT timestamp|PXAT mi

- 判断string是否存在：exists key [key ...]

- 删除一个key：del key [key ...]

- 获取key：get key

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

- 批量获取string：mget key [key ...]

- 批量设置string：mset key value [key value ...]

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

- 为key设置超时时间：expire key seconds

- 如果不存在则设置value：setnx key value

- 设置的时同时设置超时时间：setex key seconds value

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

- 数值加1：incr key

- 数值加指定的值：incrby key increment



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

利用Redis的list数据结构可以实现队列的效果。从一个方向添加元素，再从另一个方向上获取元素，就可以实现队列。比如下面这个右进左出的list。

- 左侧添加元素：lpush key element [element ...]

- 左侧弹出元素：lpop key [count]

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

和队列的原理类似，使用Redis的list数据结构在同一个方向上添加和获取元素，就可以实现栈这样的数据结构了。

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

- lindex：相当于Java链表的get(int index)方法，他需要对链表进行遍历，性能随着参数index增大而变差。lindex key index

- ltrim：ltrim的两个参数start_index和end_index定义了一个区间，在这个区间内的值保留，区间之外的则统统砍掉。可以通过ltrim来实现一个定长的链表。ltrim key start stop

- lrange：获取start_index和end_index区间的所有值。这三个函数的index都可以为-1，-1表示最后一个元素的位置。lrange key start stop

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

- 设置hash：hset key field value [field value ...]

- 获取hash：hget key field

- 获取hash中的所有：hgetall key

- 批量设置hash：hmset key field value [field value ...]

- 批量获取hash：hmget key field [field ...]

- 删除hash中的键：hdel key field [field ...]


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
127.0.0.1:6379> hdel books java
(integer) 1
127.0.0.1:6379> hdel books python
(integer) 1
127.0.0.1:6379> hdel books golang
(integer) 1
127.0.0.1:6379> hgetall books
(empty array)
```

同字符串一样，hash结构中的单个子key也可以进行计数，它对应的指令是hincrby。比如给用户的年龄加1。

```shell
127.0.0.1:6379> hset user-jolan age 29
(integer) 1
127.0.0.1:6379> hincrby user-jolan age 1
(integer) 30
```

#### set（集合）

Redis的集合相当于Java语言中的HashSet，它内部的键值对是无序的、唯一的。

- 向集合中增加元素：sadd key member [member ...]

- 获取set中所有元素：smembers key

- 判断是否集合中的元素：sismember key member

- 元素的个数：scard key

- 弹出一个元素：spop key [count]

- 删除指定value的元素：srem key member [member ...]


```shell
127.0.0.1:6379> sadd books python
(integer) 1
127.0.0.1:6379> sadd books python
(integer) 0
127.0.0.1:6379> sadd books java golang
(integer) 2
127.0.0.1:6379> smembers books  #和插入顺序不一样
1) "java"
2) "golang"
3) "python"
127.0.0.1:6379> sismember books java  #查询某个value是否存在
(integer) 1
127.0.0.1:6379> sismember books rust
(integer) 0
127.0.0.1:6379> scard books  #获取长度
(integer) 3
127.0.0.1:6379> spop books  #弹出一个元素
"python"
127.0.0.1:6379> scard books
(integer) 2
127.0.0.1:6379> srem books golang  #删除value位golang的元素
(integer) 1
127.0.0.1:6379> smembers books
1) "java"
127.0.0.1:6379> del books  #删除set
(integer) 1
127.0.0.1:6379> scard books
(integer) 0
127.0.0.1:6379> smembers books
(empty array)
```

#### zset（有序列表）

zset一方面是一个set，保证了内部value的唯一性，另一方面它可以给每个value赋予一个score，代表这个value的排序权重。

- 列表中增加元素：zadd key [NX|XX] [GT|LT] [CH] [INCR] score member [score member

- 按score排序列出：zrange key min max [BYSCORE|BYLEX] [REV] [LIMIT offset count] [W

- 按score的逆序列出：zrevrange key start stop [WITHSCORES] 

- 获取列表长度：zcard key

- 获取指定value的score：zscore key member

- 获取指定元素的排名：zrank key member

- 根据分值区间遍历zset：zrangebyscore key min max [WITHSCORES] [LIMIT offset count]

- 删除指定value的元素：zrem key member [member ...]

```shell
zadd books 9.0 "think in java"
(integer) 1
127.0.0.1:6379> zadd books 8.9 "java concurrency"
(integer) 1
127.0.0.1:6379> zadd books 8.6 "java cookbook"
(integer) 1
127.0.0.1:6379> zrange books 0 -1  #按score排序列出，参数区间为排名范围
1) "java cookbook"
2) "java concurrency"
3) "think in java"
127.0.0.1:6379> zrevrange books 0 -1  #按score逆序列出
1) "think in java"
2) "java concurrency"
3) "java cookbook"
127.0.0.1:6379> zcard books
(integer) 3
127.0.0.1:6379> zscore books "java concurrency"  #获取指定value的score，内部score使用double类型进行存储，所以存在小数点精度问题
"8.9000000000000004"
127.0.0.1:6379> zrank books "java concurrency"  #指定value的排名
(integer) 1
127.0.0.1:6379> zrangebyscore books 0 8.91  #根据分值区间遍历zset
1) "java cookbook"
2) "java concurrency"
127.0.0.1:6379> zrangebyscore books -inf 8.91 withscores  #根据分支区间遍历set，同时返回分数。其中-inf表示负无穷
1) "java cookbook"
2) "8.5999999999999996"
3) "java concurrency"
4) "8.9000000000000004"
127.0.0.1:6379> zrem books "java concurrency"  #删除指定value
(integer) 1
127.0.0.1:6379> zrange books 0 -1
1) "java cookbook"
2) "think in java"
```

zset使用跳跃链表的数据结构。

### 容器型数据结构的通用规则

list、set、hash、zset这四种数据结构是容器型数据结构，它们共享下面两条通用规则。

- create if not exists:如果容器不存在则先创建容器，再进行操作。

- drop if no elements：如果容器里已没有元素，那么立即删除容器，释放内存。如list在lpop操作到最后一个元素，列表就消失了。

### 过期时间

Redis所有的数据结构都可以设置过期时间，时间一到，Redis会自动删除相应的对象。需要注意的是，过期是以对象为单位的，而不是以其中某个子元素为单位的。

还有一个需要特别注意的地方，如果一个字符串已经设置了过期时间，然后调用set方法修改了它，它的过期时间会消失。

```shell
127.0.0.1:6379> set name yoyo
OK
127.0.0.1:6379> expire name 600
(integer) 1
127.0.0.1:6379> ttl name
(integer) 598
127.0.0.1:6379> set name yoyo
OK
127.0.0.1:6379> ttl name
(integer) -1
```

## 分布式锁

我们常常使用Redis的setnx命令来判断是否可以获取到分布式锁，为了防止线程出现异常无法释放锁，还会使用expire命令给锁加一个时间。但是setnx和expire是两个命令，无法保证原子性。如果在两个命令中间线程出错了，仍然会造成无法释放锁的情况。

为了解决这个问题，在Redis2.8版本中，作者加入了set指令的扩展参数，使setnx和expire指令可以一起执行，彻底解决了这个问题。

```shell
127.0.0.1:6379> set lock_name true ex 10 nx
OK
127.0.0.1:6379> ttl lock_name
(integer) 8
127.0.0.1:6379> ttl lock_name
(integer) 6
127.0.0.1:6379> del lock_name
(integer) 1
127.0.0.1:6379> get lock_name
(nil)
```

## 延时队列

### 异步消息队列

并非传统的队列不好用，这里只是探讨使用Redis实现消息队列的一种方案或可能性。聪明的你可能已经想到了，使用Redis的list数据结构就能实现一个队列，这种方案我们前面已经探讨了。使用Redis实现消息队列，它的有优点是速度快、构建方便，缺点是没有传统队列的那些高级特性，没有ack保证，做技术选型时一定要慎重。

### 队列空了怎么办

和传统的消息队列不同，使用Redis实现的消息队列，消费者需要pop操作来获取消息，可是如果一旦Redis的队列是空的，消费者就会陷入pop的死循环。空的轮询不但拉高了消费者所在服务器的CPU，Redis的QPS也会被拉高。这样的消费者如果有几十个，Redis慢查询可能会显著增多。通常，当消费者获取不到新的数据时，让消费者线程适当睡眠来解决这个问题。如Thread.sleep(1)。

### 阻塞读

通过线程睡眠可以解决队列为空的问题了，但是这会引入一个新的问题，那就是睡眠会导致消费者的延迟增大。

针对这个问题，我们可以使用阻塞读来解决。阻塞读在队列没有数据的时候，会立即进入休眠状态，一旦数据到来，则立刻醒过来。消息的延迟几乎为另。阻塞读对应Redis中的命令是blpop/brpop，其中b代表blocking，也就是阻塞的意思。

- brpop key [key ...] timeout

```shell
#client1
127.0.0.1:6379> brpop blocking_key 60 
1) "blocking_key"
2) "message"
(35.60s)

#client2
lpush blocking_key message
(integer) 1
```

### 空闲连接自动断开

如果线程一直阻塞在那里，Redis的连接就成了闲置连接，闲置时间过久，Redis一般会主动断开连接，减少闲置资源的占用。这个时候blpop和brpop会抛出异常。所以针对上面的解决方案，消费者断需要注意捕获这个异常并重试。

### 延时队列的实现

延时队列可以使用Redis的zset来实现。我们将消息序列号成一个字符串作为zset的value，这个消息的到期处理时间作为score。然后多个线程轮询zset获取到期的任务进行处理。zrem方法是多进程争抢任务的关键，它的返回值决定了当前实例有没有抢到任务。

```java

//塞入延时队列，5s后消费
jedis.zadd(queueKey, System.currentTimeMills() + 5000, s);


while(true){
    //获取一条任务
    Set<String> values = jedis.zrangeByScore(queueKey, 0, System.currentTimeMills(), 0, 1);

    String s = values.iterator().next();
    //判断是否抢到任务
    if(jedis.zrem(queueKey, s) > 0){

    }

}


```

## 位图

位图并非特殊的数据结构，它的内容其实就是普通的字符串。可以使用普通的get/set直接获取和设置整个位图的内容，也可以使用位图操作getbit/setbit等将byte数组看成“位数组”来处理。

### 基本用法

Redis的位数组是自动扩展的，如果设置了某个偏移位置超出了现有的内容范围，就会自动将位数组进行零扩充。接下来我们使用位操作将字符串设置为hello。首先需要得到hello的ASCII码。可在线查询或使用python命令行查询。hello去重后共四个字母，从前到后他们ASCII码的二进制分别是1101000，1100101，1101100，1101111。

接下来设置第一个字符，也就是位数组的前8位。我们只需要设置值为1的位，其中h只有1、2、4位需要设置，e字符只有9、10、13、15位需要设置。

- 获取指定位：getbit key offset

- 设置指定位的值：setbit key offset value

```shell
setbit s 1 1
0
setbit s 2 1
0
setbit s 4 1
0
setbit s 9 1
0
setbit s 10 1
0
setbit s 13 1
0
setbit s 15 1
0
get s
"he"
setbit s 16 x
(error) ERR bit is not an integer or out of range
```

上面这个例子可以理解为“零存整取”，同样也可以“零村零取”或者“整存零取”。

### 另存零取

使用单个位操作设置位值，使用单个位操作获取具体位值。

```shell
setbit w 1 1
0
setbit w 2 1
0
setbit w 4 1
0
getbit w 1
1
getbit w 2
1
getbit w 4
1
getbit w 5
0
```

### 整存零取

```shell
127.0.0.1:6379> set w h
OK
127.0.0.1:6379> getbit w 1
(integer) 1
127.0.0.1:6379> getbit w 2
(integer) 1
127.0.0.1:6379> getbit w 4
(integer) 1
127.0.0.1:6379> getbit w 5
(integer) 0
127.0.0.1:6379> setbit x 0 1
(integer) 0
127.0.0.1:6379> setbit x 1 1
(integer) 0
127.0.0.1:6379> get x  # 如果对应位的字节是不可打印字符，redis-cli会显示该字符的十六进制形式
"\xc0"
```

### 位图的统计和查找

Redis提供了位图统计指令bitcount和位图查找指令bitpos。bitcount用来统计指定位置范围内1的个数，bitpos用来查找指定范围内出现的第一个0或1。比如我们可以用bitcount统计用户一共签到了多少天，通过bitpos指令查找用户从哪天开始第一次签到。如果指定了范围参数[start,end]，就可以统计再某个时间范围内用户签到了多少天，用户自某天以后的哪天开始签到。需要注意的是，start和end参数是字节索引，也就是说指定的范围必须是8的倍数。

- 统计指定范围内1的个数：bitcount key [start end]

- 统计指定范围内的第一个0或1的位置：bitpos key bit [start] [end]

```shell
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitcount w
(integer) 21
127.0.0.1:6379> bitcount w 0 0  # 第一个字符中1的位数（第一个0表示起始位是0， 第二个0表示截止位是0）
(integer) 3
127.0.0.1:6379> bitcount w 0 1  # 前两个字符中1的位数
(integer) 7
127.0.0.1:6379> bitpos w 0  # 第一个0位
(integer) 0
127.0.0.1:6379> bitpos w 1 # 第一个1位
(integer) 1
127.0.0.1:6379> bitpos w 1 1 1  # 从第二个字符算起，第一个1位（下标从0开始）
(integer) 9
127.0.0.1:6379> bitpos w 1 2 2  # 从第三个字符算起，第一个1位
(integer) 17
```

### 魔术指令bitfield

Redis在在3.2版本以后新增了一个功能强大的指令bitfield，有了这条指令，可以一次进行多个位操作。

bitfield有三个子指令，分别是get、set、incrby。它们都可以对指定位片段进行读写，但是最多只能处理64个连续的位。

- bitfield key [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]

```shell
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w get u4 0  # 从第一位开始取4个位，结果是无符号数。其中u代表无符号数
1) (integer) 6
127.0.0.1:6379> bitfield w get u3 2  # 从第三位开始取3个位，结果是无符号数。其中u代表无符号数
1) (integer) 5
127.0.0.1:6379> bitfield w get i4 0  # 从第一位开始取4个位，结果是有符号数。其中i代表有符号数
1) (integer) 6
127.0.0.1:6379> bitfield w get i3 2  # 从第三位开始取3个位，结果是有符号数。其中i代表有符号数
1) (integer) -3
127.0.0.1:6379> bitfield w get u64 2
(error) ERR Invalid bitfield type. Use something like i16 u8. Note that u64 is not supported but i64 is.
127.0.0.1:6379> bitfield w get i64 2
1) (integer) -6803336285151821824
```

所谓有符号数是指获取位数组中第一个位是符号位，剩下的才是值。如果第一位是1，那就是负数。无符号数表示非负数，没有符号位，获取的位数组全部都是值。有符号数最多可以获取64位，无符号数只能获取63位。

```shell
127.0.0.1:6379> bitfield w get u4 0 get u3 2 get i4 0 get i3 2  # 一次执行多个指令
1) (integer) 6
2) (integer) 5
3) (integer) 6
4) (integer) -3

127.0.0.1:6379> bitfield w set u8 8 97  # 使用set指令将第二个字符e改成a，a的ASCII码是97
1) (integer) 101
127.0.0.1:6379> get w
"hallo"
```

incrby用来对指定范围的位进行自增操作。既然是自增，就可能出现溢出。增加正数会出现上溢出，负数会出现下溢出。Redis默认的处理是折返，即如果出现溢出，就将溢出的符号位丢弃。比如8位无符号数255，加一后就会溢出全部变为0。再比如8位有符号数127，加1就会溢出变成-128。

- bitfield的incrby子指令，对位进行自增操作：bitfield w incrby [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]

```shell
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w incrby u4 2 1  # 从第三个位开始，对接下来的4位无符号数+1
1) (integer) 11
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w incrby u4 2 1  # 折返溢出
1) (integer) 0
```

bitfield指令提供了溢出策略子指令overflow，用户可以选择溢出行为，默认是折返（wrap），还可以选择失败（fail）——报错不执行，以及饱和截断（sat）——超过了范围就停留在最大或最小值。overflow指令只影响接下来的第一条指令，这条指令执行完后溢出策略会变成默认值折返（wrap）。

```shell
127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 11
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w overflow sat incrby u4 2 1  # 保持最大值
1) (integer) 15

127.0.0.1:6379> set w hello
OK
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 11
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 12
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 13
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 14
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1
1) (integer) 15
127.0.0.1:6379> bitfield w overflow fail incrby u4 2 1  # 不执行
1) (nil)
```

## HyperLogLog

HyperLogLog是Redis的高级数据结构，它提供不精确的去重计数方案，虽然不精确，但是也不是非常离谱，标准误差是0.81%。

HyperLogLog占用的空间大小是12KB

- 加入一个元素：pfadd key element [element ...]

- 计算元素个数：pfcount key [key ...]

- 合并两个HyperLogLog：pfmerge destkey sourcekey [sourcekey ...]

```shell
127.0.0.1:6379> pfadd hll user1
(integer) 1
127.0.0.1:6379> pfcount hll
(integer) 1
127.0.0.1:6379> pfadd hll user2
(integer) 1
127.0.0.1:6379> pfcount hll
(integer) 2
127.0.0.1:6379> pfadd hygg user3
(integer) 1
127.0.0.1:6379> pfadd hygg user4
(integer) 1
127.0.0.1:6379> pfadd hygg user5
(integer) 1
127.0.0.1:6379> pfmerge hll hygg
OK
127.0.0.1:6379> pfcount hygg
(integer) 3
127.0.0.1:6379> pfcount hll
(integer) 5
```

## 布隆过滤器

布隆过滤器（Bloom Filter）也是Redis的高级数据结构。它主要是解决去重问题，虽然不是特别精确，但是在空间上能节省90%以上。

可以把布隆过滤器理解为一个不怎么精确的set结构，当使用它的contains方法判断某个对象是否存在时，它可能会误判。但是布隆过滤器也并非特别不精准，相对来说它有小小的误判概率。当布隆过滤器说某个值存在时，这个值可能不存在；当它说某个值不存在时，这个值肯定不存在。

Redis4.0后官方才以插件的功能提供了布隆过滤器。

### 下载（Mac）

找到下载地址：https://github.com/RedisBloom/RedisBloom（也可以到Redis官网https://redis.io/下载），然后找到发行版本，如下图所示。

![avatar](img/2.png)

根据需要的格式进行下载

![avatar](img/3.jpg)

将文件放到需要的目录进行解压，然后进入目录，执行make命令。执行完成后，在目录下会多一个redisbloom.so文件。

最后使用挂在方式重启redis(或者修改配置文件：loadmodule /xxx/redisbloom.so)。

```shell
redis-server  --loadmodule xxx路径/redisbloom.so 
```

### 布隆过滤器的基本用法

Redis的布隆过滤器有两个基本指令，bf.add和bf.exists。bf.add添加元素，bf.exists查询元素是否存在。他们的用法和set集合的sadd和sismember差不多。bf.add只能一次添加一个元素，bf.madd可以一次添加多个元素。同样的，如果需要一次查询多个元素是否存在，就需要使用bf.mexists指令。

- 向布隆过滤器中添加元素：bf.add key ...options...

- 判断布隆过滤器中是否存在元素：bf.exists key ...options..

```shell
127.0.0.1:6379> bf.add rb user1
(integer) 1
127.0.0.1:6379> bf.add rb user2
(integer) 1
127.0.0.1:6379> bf.add rb user3
(integer) 1
127.0.0.1:6379> bf.exists rb user1
(integer) 1
127.0.0.1:6379> bf.exists rb user2
(integer) 1
127.0.0.1:6379> bf.exists rb user3
(integer) 1
127.0.0.1:6379> bf.exists rb user4
(integer) 0
127.0.0.1:6379> bf.madd rb user4 user5 user6
1) (integer) 1
2) (integer) 1
3) (integer) 1
127.0.0.1:6379> bf.mexists rb user4 user5 user6 user7
1) (integer) 1
2) (integer) 1
3) (integer) 1
4) (integer) 0
```

Redis其实还提供了自定义参数的布隆过滤器，需要我们在add之前使用bf.reserve指令创建。如果对应key已经存在，bf
.reserve会报错。bf.reserve有三个参数，分别是key、error_rate（错误率）和initial_size。

- error_rate越低，需要的空间越大。

- initial_size表示预计放入的元素数量，当实际数量超出这个数值时，误判率会上升，所以需要提前设置一个较大的数值避免超出导致误判率升高.

- 如果不使用bf.reserve，默认的error_rate是0.01，默认的initial_size是100。

- 指令格式：bf.reserve key ...options..

#### 布隆过滤器的原理

每个布隆过滤器对应到Redis的数据结构里面就是一个大型的位数组和几个不一样的无偏hash函数（能够把元素的hash值计算的比较均匀的hash函数）。向布隆过滤器中添加key时，会使用多个hash杉树对key进行hash，酸的一个整数索引值，然后对位数组长度进行取模运算，每个hash函数都会酸的一个不同的位置。再把位数组的这几个位置都置为1，就完成了add操作。判断元素是否在布隆过滤器中的原理和add操作类似。

### 空间占用估计

我们使用n表示预计元素的数量，f表示错误率，k表示hash函数的最佳数量，表示实际元素和预计元素的倍数。

k = 0.7 * (1 / n)  # 约等于
f = 0.6185^(1 / n) # ^表示次方计算
f = (1 - 0.5^t)^k  # 极限接近。当t越大，错误率越大

## 令牌桶限流Redis-cell

Redis4.0提供了限流Redis模块，它叫Redis-Cell。该模块只有一条指令cl.throttle。https://blog.csdn.net/yzf279533105/article/details/111310685

## GeoHash

Redis再3.2版本以后增加了地理位置geo模块。https://blog.csdn.net/usher_ou/article/details/122716877

## scan

Redis提供了简单粗暴的指令keys用来列出所有满足特定正则字符串规则的key。

```shell
127.0.0.1:6379> set demo1 a
OK
127.0.0.1:6379> set demo2 b
OK
127.0.0.1:6379> set demo3 c
OK
127.0.0.1:6379> set demo4 a
OK
127.0.0.1:6379> set demo5 b
OK
127.0.0.1:6379> set demo6 c
OK
127.0.0.1:6379> keys demo*
1) "demo1"
2) "demo4"
3) "demo3"
4) "demo2"
5) "demo5"
6) "demo6"
127.0.0.1:6379> keys d*mo
(empty array)
127.0.0.1:6379> keys d*mo1
1) "demo1"
```

