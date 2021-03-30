# 命令行

## 展示所有的数据库

​	SHOW DATABASES;

## 使用某个数据库

​	如，使用crashcourse数据库使用命令：

​	USE crashcourse;

## 展示所有的表

​	SHOW TABLES;

## 展示表的所有列

​	展示customer表的所有列：

​	SHOW COLUMS FROM customer;

## 其他常用指令

​	1） 展示customer表的所有列：DESCRIBE customer;

​	2） 显示服务状态：SHOW STATUS;

​	3） 显示用户的安全权限：SHOW GRANTS;

​	4） SHOW帮助命令：HELP SHOW;

# 易混淆概念

## 关于排序

​	如果没有明确排序查询结果，则返回的数据的顺序没有任何意义。返回数据的顺序可能是数据被添加到表中的顺序，也可能不是。

## 大小写

​	1） SQL语句不区分大小写。常常对SQL关键字使用大写，对其他自定义的表、列等使用小写。

​	2） MYSQL在执行查询匹配时，默认不区分大小写。

## DISTINCT

​	不能部分使用DISTINCT，DISTINCT关键字应用于所有列而不仅仅是它所前置的列。

## LIMIT

​	1） LIMIT 5 , 5 表示返回从第5行开始的5行。

​	2） 使用LIMIT检索出来的第1行为行0而不是行1，因此LIMIT 1 , 1检索出来的是第2行而不是第1行。

​	3） LIMIT 4 OFFSET 3意为从行3开始取4行。等同于LIMIT 3 , 4

## ORDER BY 

​	1） 对多个列进行排序，在ORDER BY 后面只要指定多个列名，列名之前用逗号分开。

​	2） 如果要对每个列进行降序排列，要在每个列后面使用DESC，而不是在最后使用。

​	3） ORDER BY必须是SELECT语句的最后一个子句。

## NULL相关

​	1） 在通过过滤选择出不具有特定值的行时（如 name != 'test'），你可能希望返回具有NULL的行，但是不行。因为未知（也就是NULL）具有特殊的含义，数据库不知道他们是否匹配，所以在匹配（等于）过滤或者不匹配（不等于）过滤是不返回他们。

​	2） AVG(), SUM(), MIN(), MAX()这些函数都忽略值为NULL的行。

​	3） 如果分组中（GROUP BY）中具有NULL值，则NULL将作为一个分组返回。如果列中有多行NULL，它们将作为一个分组。

​	3）NULL值不是空串，如果一列是NOT NULL，但仍然可以是''

## IN

​	1） IN操作具有和OR操作相同的功能

​	2） IN操作符一般比OR操作符更快

## 通配符

​	1） 虽然似乎%通配符可以匹配任何东西，但有一个例外，即NULL

​	2） 不要过度使用通配符。

​	3） 如果确实需要使用通配符，尽量避免把他们放在搜索模式的开始处。

## 测试计算

​	可以使用如SELECT 2 * 3;或者SELECT NOW();来快速测试或者计算，省略后面的表名。

## 时间函数

​	1） 如果仅仅是使用日期，请使用Date()，这样方式表中的值有其他格式，影响SQL的查询结果。如：SELECT * FROM orders WHERE Date(order_date) = '2021-03-30';

​	2） 同样的，如果要使用时间，可以用Time()函数。

​	3） 查询某年某月数据的一种方式：SELECT * FROM orders WHERE Year(order_date) = 2021 AND Month(order_date) = 3;

## WITH GROUP

​	待研究

## UNION

​	1） UNION从查询结果中自动去除了重复的行。

​	2） UNION ALL不取消重复的行。

​	3） 在UNION组合查询时，只能使用一条ORDER BY 子句，他必须出现在最后一条SELECT语句之后。

## INSERT

​	1） 在使用INSERT语句时，即使可以默认不列出表的列名字，但是不建议这样做，因为一旦表结构发生变化，插入到表中的数据将会错乱。

​	2）INSERT语句可能很耗时（特别是有很多索引要处理时），而且它可能降低将要执行的SELECT语句的性能。 

​	3） 可以在INSERT和INTO之间添加关键字LOW PRIORITY，只是MYSQL降低INSERT语句的优先级，如：INSERT LOW PRIORITY INTO，这种方法同样适用于UPDAT和DELETE语句。

​	4）INSERT用单条语句处理多行插入比使用多个INSERT语句插入快。如：INSERT INTO orders(...) VALUES(...), (...);

## UPDATE

​	1）如果使用UPDATE语句更新多行，如果在更新这些行中的一行或者多行出现错误时，整个UPDATE操作将被取消。如果即使是发错错误也要继续进行更新时，可使用IGNORE关键字。如：UPDATE IGNORE customers;

## last_insert_id()

​	SELECT last_insert_id()，此语句返回最后一个AUTO_INCREMENT值。

## 表字段默认值

​	MYSQL不允许使用函数作为字段的默认值。

## 重命名表

​	RENAME TABLE customer01 TO customer02;

