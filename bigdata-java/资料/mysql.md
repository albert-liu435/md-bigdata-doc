## Mysql索引

### Mysql聚集索引和非聚集索引

https://www.cnblogs.com/starcrm/p/12971702.html

MySQL的Innodb存储引擎的索引分为聚集索引和非聚集索引两大类，理解聚集索引和非聚集索引可通过对比汉语字典的索引。汉语字典提供了两类检索汉字的方式，第一类是拼音检索（前提是知道该汉字读音），比如拼音为cheng的汉字排在拼音chang的汉字后面，根据拼音找到对应汉字的页码（因为按拼音排序，二分查找很快就能定位），这就是我们通常所说的字典序；第二类是部首笔画检索，根据笔画找到对应汉字，查到汉字对应的页码。拼音检索就是聚集索引，因为存储的记录（数据库中是行数据、字典中是汉字的详情记录）是按照该索引排序的；笔画索引，虽然笔画相同的字在笔画索引中相邻，但是实际存储页码却不相邻，这是非聚集索引。

#### 聚集索引

索引中键值的逻辑顺序决定了表中相应行的物理顺序。
　　聚集索引确定表中数据的物理顺序。聚集索引类似于电话簿，后者按姓氏排列数据。 聚集索引对于那些经常要搜索范围值的列特别有效。使用聚集索引找到包含第一个值的行后，便可以确保包含后续索引值的行在物理相邻。例如，如果应用程序执行 的一个查询经常检索某一日期范围内的记录，则使用聚集索引可以迅速找到包含开始日期的行，然后检索表中所有相邻的行，直到到达结束日期。这样有助于提高此 类查询的性能。同样，如果对从表中检索的数据进行排序时经常要用到某一列，则可以将该表在该列上聚集（物理排序），避免每次查询该列时都进行排序，从而节 省成本。

#### 非聚集索引

索引中索引的逻辑顺序与磁盘上行的物理存储顺序不同。

　　其实按照定义，除了聚集索引以外的索引都是非聚集索引，只是人们想细分一下非聚集索引，分成普通索引，唯一索引，全文索引。如果非要把非聚集索引类比成现实生活中的东西，那么非聚集索引就像新华字典的偏旁字典，他结构顺序与实际存放顺序不一定一致。

非聚集索引的存储结构与前面是一样的，不同的是在叶子结点的数据部分存的不再是具体的数据，而数据的聚集索引的key。所以通过非聚集索引查找的过程是先找到该索引key对应的聚集索引的key，然后再拿聚集索引的key到主键索引树上查找对应的数据，这个过程称为**回表**！

如何解决非聚集索引的二次查询的问题

建立两列以上的索引，即可查询复合索引里的列的数据而不需要进行回表二次查询，如index(col1, col2)，执行下面的语句

要注意使用复合索引需要满足最左侧索引的原则，也就是查询的时候如果where条件里面没有最左边的一到多列，索引就不会起作用。

#### 最左前缀原则

我们都知道索引的底层是一颗 B+ 树，那么联合索引当然还是一颗 B+ 树，只不过联合索引的健值数量不是一个，而是多个。**构建一颗 B+ 树只能根据一个值来构建**，因此数据库依据联合索引最左的字段来构建 B+ 树。

就是要考虑查询字段的字段顺序，只有遵守这个原则才能最大的提高使用效率

- mysql会从左到右匹配，直到遇到范围查询（>,<,between, like）就停止匹配，比如联合索引（a,b,c,d）匹配a=1 and b=2 and c>2 and d=1,此时d字段是使用不到索引的功能的，如果这时索引的字段顺序是（abdc），此时d就会应用联合索引

- 如果判断条件“=”，那么不会强调字段的顺序，mysql的查询优化器会自动识别形式：a=1 and c=2 and b=2字段顺序a,b,c不做区别

#### 为什么使用联合索引

索引下推

1、减少开销。建一个联合索引(col1,col2,col3)，实际相当于建了(col1),(col1,col2),(col1,col2,col3)三个索引。每多一个索引，都会增加写操作的开销和磁盘空间的开销。对于大量数据的表，使用联合索引会大大的减少开销！

2、覆盖索引。对联合索引(col1,col2,col3)，如果有如下的sql: select col1,col2,col3 from test where col1=1 and col2=2。那么MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机io操作。减少io操作，特别的随机io其实是dba主要的优化策略。所以，在真正的实际应用中，覆盖索引是主要的提升性能的优化手段之一。

3、效率高。索引列越多，通过索引筛选出的数据越少。有1000W条数据的表，有如下sql:select from table where col1=1 and col2=2 and col3=3,假设假设每个条件可以筛选出10%的数据，如果只有单值索引，那么通过该索引能筛选出1000W10%=100w条数据，然后再回表从100w条数据中找到符合col2=2 and col3= 3的数据，然后再排序，再分页；如果是联合索引，通过索引筛选出1000w10% 10% *10%=1w，效率提升可想而知
原文链接：https://blog.csdn.net/weixin_39631017/article/details/113440601

## Mysql的执行流程

[参考](https://blog.csdn.net/yajie_12/article/details/89314039)

![mysql基础架构](.\mysql\mysql基础架构.png)

## Mysql中的锁

### 共享锁和排它锁(Shared and Exclusivve Locks)

[参考](https://www.cnblogs.com/boblogsbo/p/5602122.html)

　　共享锁和排它锁是行级锁，有两种类型的行级锁
　　　　共享锁(s lock)又称为读锁，简称S锁，允许持有锁的事务对行进行读取操作
　　　　排它锁(x lock)又称写锁，简称X锁，允许持有锁的事务对行进行更新和删除操作

　　事务a在行r上拥有共享锁，则其他事务可以获得r的共享锁，无法获得r的排它锁，即可读不可写
　　事务a在行r上拥有排它锁，则其他事务既不能获得共享锁，也不能获得排它锁，即不可读也不可写而必须等待当前事务完成

排他锁可以使用select ...for update语句，加共享锁可以使用select ... lock in share mode语句

### 意向锁(Intention Locks)

[参考](https://baijiahao.baidu.com/s?id=1698458787310931243&wfr=spider&for=pc)

　　意向锁是表级锁，用来表示将会有哪种类型的行级锁即将被使用，有两种类型的意向锁
　　　　意向共享锁(IS Locks)：表明即将在表中的某一行上设置共享锁(s lock)
　　　　意向排它锁(Ix Locks): 表明即将在表中的某一行上设置排它锁(x lock)

　　一个事务如果要获得一个共享锁(s lock)必须首先获得一个意向共享锁(is lock)，一个事务如果要获得一个排它锁(x lock)则必须先获得一个意向排它锁(ix lock)

## Mysql MVCC

[参考](https://blog.csdn.net/SnailMann/article/details/94724197)

