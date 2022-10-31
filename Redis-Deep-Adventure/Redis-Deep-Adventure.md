# 第1篇 基础和应用篇

## Redis的安装

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

Redis的字符串是动态字符串，是可以修改的字符串，内部结构的实现类似于Java的ArrayList，采用预分配冗余空间的方式来减少内存的频繁分配。当字符串长度小于1MB时，扩容都是加倍现有的空间。如果字符串长度超过1MB，扩容时只会多扩1MB的空间。需要注意的是字符串最大长度为512MB。

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

