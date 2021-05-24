## 使用索引的代价

> 在熟悉了B+树索引原理之后，本篇文章的主题是唠叨如何更好的使用索引，虽然索引是个好东西，可不能乱建，在介绍如何更好的使用索引之前先要了解一下使用这玩意儿的代价，它在空间和时间上都会拖后腿：
>
> ------
>
> 空间上的代价:
>
> 这个是显而易见的，每建立一个索引都要为它建立一棵B+树，每一棵B+树的每一个节点都是一个数据页，一个页默认会占用16KB的存储空间，一棵很大的B+树由许多数据页组成
>
> ------
>
> 时间上的代价:
>
> 每次对表中的数据进行增、删、改操作时，都需要去修改各个B+树索引。而且我们讲过，B+树每层节点都是按照索引列的值从小到大的顺序排序而组成了双向链表。不论是叶子节点中的记录，还是内节点中的记录（也就是不论是用户记录还是目录项记录）都是按照索引列的值从小到大的顺序而形成了一个单向链表。而增、删、改操作可能会对节点和记录的排序造成破坏，所以存储引擎需要额外的时间进行一些**记录移位，页面分裂、页面回收**啥的操作来维护好节点和记录的排序。如果我们建了许多索引，每个索引对应的B+树都要进行相关的维护操作

**所以说，一个表上索引建的越多，就会占用越多的存储空间，在增删改记录的时候性能就越差。**

## B+树索引适用的条件

> 这里的内容基于我们对于 b+ 树索引的理解程度. 非常重要. 首先，B+树索引并不是万能的，并不是所有的查询语句都能用到我们建立的索引。

创建一个表:

```sql
CREATE TABLE person_info(
    id INT NOT NULL auto_increment,
    name VARCHAR(100) NOT NULL,
    birthday DATE NOT NULL,
    phone_number CHAR(11) NOT NULL,
    country varchar(100) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_name_birthday_phone_number (name, birthday, phone_number)
);
```

> 1.表中的主键是id列，它存储一个自动递增的整数。所以InnoDB存储引擎会自动为id列建立聚簇索引。
> 2.我们额外定义了一个二级索引idx_name_birthday_phone_number，它是由3个列组成的联合索引。所以在这个索引对应的B+树的叶子节点处存储的用户记录只保留name、birthday、phone_number这三个列的值以及主键id的值，并不会保存country列的值。

`idx_name_birthday_phone_number`的二级索引示意图(简略版):

![img](https://s1.ax1x.com/2020/04/01/G14H9f.md.png)

从图中可以看出，这个 `idx_name_birthday_phone_number` 索引对应的 B+ 树中页面和记录的排序方式就是这样的：

> 先按照name列的值进行排序。
> 如果name列的值相同，则按照birthday列的值进行排序。
> 如果birthday列的值也相同，则按照phone_number的值进行排序。

这个排序方式是非常非常重要的, 涉及到索引能否为你的查询语句提升效率.



## 全值匹配

如果我们的搜索条件中的列和索引列一致的话，这种情况就称为全值匹配，比方说下边这个查找语句：

> SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday = '1990-09-27' AND phone_number = '15123983239';

我们建立的 `idx_name_birthday_phone_number` 索引包含的 3 个列在这个查询语句中都展现出来了。大家可以想象一下这个查询过程：

> 因为B+树的数据页和记录先是按照name列的值进行排序的，所以先可以很快定位name列的值是Ashburn的记录位置。
> 在name列相同的记录里又是按照birthday列的值进行排序的，所以在name列的值是Ashburn的记录里又可以快速定位birthday列的值是'1990-09-27'的记录。
> 如果很不幸，name和birthday列的值都是相同的，那记录是按照phone_number列的值排序的，所以联合索引中的三个列都可能被用到。

有个疑问，WHERE子句中的几个搜索条件的顺序对查询结果有啥影响么？也就是说如果我们调换name、birthday、phone_number这几个搜索列的顺序对查询的执行过程有影响么？比方说写成下边这样：

> SELECT * FROM person_info WHERE birthday = '1990-09-27' AND phone_number = '15123983239' AND name = 'Ashburn';

答案是：`没影响`, MySQL有一个叫查询优化器，会分析这些搜索条件并且按照可以使用的索引中列的顺序来决定先使用哪个搜索条件，后使用哪个搜索条件。

## 匹配左边的列

其实在我们的搜索语句中也可以不用包含全部联合索引中的列，只包含左边的就行，比方说下边的查询语句：

> SELECT * FROM person_info WHERE name = 'Ashburn';

或者包含多个左边的列也行：

> SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday = '1990-09-27';

那为什么搜索条件中必须出现左边的列才可以使用到这个B+树索引呢？比如下边的语句就用不到这个B+树索引么？

> SELECT * FROM person_info WHERE birthday = '1990-09-27';

答案是: `有影响`, 因为B+树的数据页和记录先是按照name列的值排序的，在name列的值相同的情况下才使用birthday列进行排序，也就是说name列的值不同的记录中birthday的值可能是无序的。所以不能跳过 name 列直接根据 birthday 的值去查找.

所以联合索引的使用技巧就是尽量匹配左边的列, 比方说联合索引idx_name_birthday_phone_number中列的定义顺序是name、birthday、phone_number，如果我们的搜索条件中只有name和phone_number，而没有中间的birthday，比方说这样：

> SELECT * FROM person_info WHERE name = 'Ashburn' AND phone_number = '15123983239';

这样只能用到name列的索引，birthday和phone_number的索引就用不上了，因为name值相同的记录先按照birthday的值进行排序，birthday值相同的记录才按照phone_number值进行排序。

## 匹配列前缀

上个小节说明了使用联合索引时的注意项, 这一小节讲一下当查询时需要进行模糊匹配的情况.

我们联合查询的索引首先按照 name 进行排序, 所以根据 name 列的字符集以及排序规则, 这个索引的页目录记录大概是按照如下进行排序的:

![img](https://s1.ax1x.com/2020/04/01/G1Tcad.png)

所以只匹配 name 的前缀也是可以快速定位记录的, 对于这样的查询语句:

> SELECT * FROM person_info WHERE name LIKE 'As%';

但是需要注意的是，如果只给出后缀或者中间的某个字符串，比如这样：

> SELECT * FROM person_info WHERE name LIKE '%As%';

因为我们的联合索引中并没有这种 `%As%` 排序规则, **所以只能全表扫描了**. 对于这种模糊匹配需要特殊的处理, 也就是考验开发者的查询优化能力.

## 匹配范围值

回头看我们 `idx_name_birthday_phone_number` 索引的B+树示意图，所有记录都是按照索引列的值从小到大的顺序排好序的，所以这极大的方便我们查找索引列的值在某个范围内的记录。比方说下边这个查询语句：

> SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow';

由于B+树中的数据页和记录是先按name列排序的，所以我们上边的查询过程其实是这样的：

> 找到name值为Asa的记录。
> 找到name值为Barlow的记录。
> 由于所有记录都是由链表连起来的（记录之间用单链表，数据页之间用双链表），所以他们之间的记录都可以很容易的取出来
> 找到这些记录的主键值，再到聚簇索引中回表查找完整的记录

不过在使用联合进行范围查找的时候需要注意，如果对多个列同时进行范围查找的话，只有对索引最左边的那个列进行范围查找的时候才能用到B+树索引，比方说这样：

> SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow' AND birthday > '1980-01-01';

上边的查询过程如下:

> 通过条件name > 'Asa' AND name < 'Barlow'来对name进行范围，查找的结果可能有多条name值不同的记录，
> 对这些name值不同的记录继续通过birthday > '1980-01-01'条件继续过滤。
>
> ------
>
> 这样子对于联合索引idx_name_birthday_phone_number来说，只能用到name列的部分，而用不到birthday列的部分，**因为只有name值相同的情况下才能用birthday列的值进行排序**，而这个查询中通过name进行范围查找的记录中可能并不是按照birthday列进行排序的，所以在搜索条件中继续以birthday列进行查找时是用不到这个B+树索引的。

## 精确匹配某一列并范围匹配另外一列

对于同一个联合索引来说，虽然对多个列都进行范围查找时只能用到最左边那个索引列，但是如果左边的列是精确查找，则右边的列可以进行范围查找，比方说这样：

> SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday > '1980-01-01' AND birthday < '2000-12-31' AND phone_number > '15100000000';

这个查询的条件可以分为3个部分：

> 1.name = 'Ashburn'，对name列进行精确查找，当然可以使用B+树索引了。
> \2. birthday > '1980-01-01' AND birthday < '2000-12-31'，由于name列是精确查找，所以通过name = 'Ashburn'条件查找后得到的结果的name值都是相同的，它们会再按照birthday的值进行排序。所以此时对birthday列进行范围查找是可以用到B+树索引的。
> \3. phone_number > '15100000000'，通过birthday的范围查找的记录的birthday的值可能不同，所以这个条件无法再利用B+树索引了，只能遍历上一步查询得到的记录。

同理，下边的查询也是可能用到这个idx_name_birthday_phone_number联合索引的：

> SELECT * FROM person_info WHERE name = 'Ashburn' AND birthday = '1980-01-01' AND phone_number > '15100000000';

## 用于排序

> 我们在写查询语句的时候经常需要对查询出来的记录通过ORDER BY子句按照某种规则进行排序。一般情况下，我们只能把记录都加载到内存中，再用一些排序算法，比如快速排序、归并排序、吧啦吧啦排序等等在内存中对这些记录进行排序，有的时候可能查询的结果集太大以至于不能在内存中进行排序的话，还可能暂时借助磁盘的空间来存放中间结果，排序操作完成后再把排好序的结果集返回到客户端。

> 在MySQL中，把这种在内存中或者磁盘上进行排序的方式统称为文件排序（英文名：filesort），跟文件这个词儿一沾边儿，就显得这些排序操作非常慢了（磁盘和内存的速度比起来，就像是飞机和蜗牛的对比）。但是如果 ORDER BY 子句里使用到了我们的索引列，就有可能省去在内存或文件中排序的步骤，比如下边这个简单的查询语句：

> SELECT * FROM person_info ORDER BY name, birthday, phone_number LIMIT 10;

> 这个查询的结果集需要先按照name值排序，如果记录的name值相同，则需要按照birthday来排序，如果birthday的值相同，则需要按照phone_number排序。大家可以回过头去看我们建立的idx_name_birthday_phone_number索引的示意图，因为这个B+树索引本身就是按照上述规则排好序的，所以直接从索引中提取数据，然后进行回表操作取出该索引中不包含的列就好了。简单吧？是的，索引就是这么牛逼。

> 对于联合索引有个问题需要注意，**ORDER BY的子句后边的列的顺序也必须按照索引列的顺序给出**，如果给出ORDER BY phone_number, birthday, name的顺序，那也是用不了B+树索引，这种颠倒顺序就不能使用索引的原因我们上边详细说过了，这就不赘述了。

> 同理，ORDER BY name、ORDER BY name, birthday这种匹配索引左边的列的形式可以使用部分的B+树索引。**当联合索引左边列的值为常量，也可以使用后边的列进行排序**，比如这样：

> SELECT * FROM person_info WHERE name = 'A' ORDER BY birthday, phone_number LIMIT 10;

### 不可以使用索引进行排序的几种情况

```
ASC、DESC混用
```

对于使用联合索引进行排序的场景，我们要求各个排序列的排序顺序是一致的，也就是要么各个列都是ASC规则排序，要么都是DESC规则排序。

```
排序列包含非同一个索引的列
WHERE子句中出现非排序使用到的索引列
```

## 用于分组

有时候我们为了方便统计表中的一些信息，会把表中的记录按照某些列进行分组。比如下边这个分组查询：

> SELECT name, birthday, phone_number, COUNT(*) FROM person_info GROUP BY name, birthday, phone_number

这个查询语句相当于做了3次分组操作：

> 先把记录按照name值进行分组，所有name值相同的记录划分为一组。
> 将每个name值相同的分组里的记录再按照birthday的值进行分组，将birthday值相同的记录放到一个小分组里，所以看起来就像在一个大分组里又化分了好多小分组。
> 再将上一步中产生的小分组按照phone_number的值分成更小的分组，所以整体上看起来就像是先把记录分成一个大分组，然后把大分组分成若干个小分组，然后把若干个小分组再细分成更多的小小分组。

## 回表的代价

首先看一个查询:

> SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow';

分析一下上面的sql 语句:

> 从索引`idx_name_birthday_phone_number`对应的B+树中取出name值在Asa～Barlow之间的用户记录。
>
> ------
>
> 由于索引idx_name_birthday_phone_number对应的B+树用户记录中只包含name、birthday、phone_number、id这4个字段，而查询列表是*，意味着要查询表中所有字段，也就是还要包括country字段。这时需要把从上一步中获取到的每一条记录的id字段都到聚簇索引对应的B+树中找到完整的用户记录，也就是我们通常所说的
>
> ```
> 回表
> ```
>
> ，然后把完整的用户记录返回给查询用户。
>
> ------
>
> 由于索引idx_name_birthday_phone_number对应的B+树中的记录首先会按照name列的值进行排序，所以值在Asa～Barlow之间的记录在磁盘中的存储是相连的，集中分布在一个或几个数据页中，我们可以很快的把这些连着的记录从磁盘中读出来，这种读取方式我们也可以称为
>
> ```
> 顺序I/O
> ```
>
> ------
>
> 根据第1步中获取到的记录的id字段的值可能并不相连，而在聚簇索引中记录是根据id（也就是主键）的顺序排列的，所以根据这些并不连续的id值到聚簇索引中访问完整的用户记录可能分布在不同的数据页中，这样读取完整的用户记录可能要访问更多的数据页，这种读取方式我们也可以称为
>
> ```
> 随机I/O
> ```
>
> ------
>
> 一般情况下，顺序I/O比随机I/O的性能高很多，所以步骤1的执行可能很快，而步骤2就慢一些。
>
> ```
> 需要回表的记录越多，使用二级索引的性能就越低，甚至让某些查询宁愿使用全表扫描也不使用二级索引
> ```
>
> 。比方说name值在Asa～Barlow之间的用户记录数量占全部记录数量90%以上，那么如果使用idx_name_birthday_phone_number索引的话，有90%多的id值需要回表，这不是吃力不讨好么，还不如直接去扫描聚簇索引（也就是全表扫描）

### 那什么时候采用全表扫描的方式，什么时候使用采用二级索引 + 回表的方式去执行查询呢？

这个就是传说中的查询优化器做的工作，查询优化器会事先对表中的记录计算一些统计数据，然后再利用这些统计数据根据查询的条件来计算一下需要回表的记录数，**需要回表的记录数越多，就越倾向于使用全表扫描**，反之倾向于使用二级索引 + 回表的方式。当然优化器做的分析工作不仅仅是这么简单，但是大致上是个这个过程。一般情况下，限**制查询获取较少的记录数会让优化器更倾向于选择使用二级索引 + 回表的方式进行查询，因为回表的记录越少，性能提升就越高**，比方说上边的查询可以改写成这样：

> SELECT * FROM person_info WHERE name > 'Asa' AND name < 'Barlow' LIMIT 10;

添加了LIMIT 10的查询更容易让优化器采用二级索引 + 回表的方式进行查询.

对于有排序需求的查询，上边讨论的采用全表扫描还是二级索引 + 回表的方式进行查询的条件也是成立的，比方说下边这个查询：

> SELECT * FROM person_info ORDER BY name, birthday, phone_number;

由于查询列表是*，所以如果使用二级索引进行排序的话，需要把排序完的二级索引记录全部进行回表操作，这样操作的成本还不如直接遍历聚簇索引然后再进行文件排序（filesort）低，所以优化器会倾向于使用全表扫描的方式执行查询。如果我们加了LIMIT子句，比如这样：

> SELECT * FROM person_info ORDER BY name, birthday, phone_number LIMIT 10;

这样需要回表的记录特别少，优化器就会倾向于使用二级索引 + 回表的方式执行查询。

## 覆盖索引

为了彻底告别回表操作带来的性能损耗，我们建议：最好在查询列表里只包含索引列，比如这样：

> SELECT name, birthday, phone_number FROM person_info WHERE name > 'Asa' AND name < 'Barlow'

因为我们只查询 name, birthday, phone_number 这三个索引列的值，所以在通过idx_name_birthday_phone_number 索引得到结果后就不必到聚簇索引中再查找记录的剩余列，也就是 country 列的值了，这样就省去了回表操作带来的性能损耗。我们把这种只需要用到索引的查询方式称为 `索引覆盖`。排序操作也优先使用覆盖索引的方式进行查询

## 如何挑选索引

### 只为用于搜索、排序或分组的列创建索引

> 也就是说，只为出现在WHERE子句中的列、连接子句中的连接列，或者出现在ORDER BY或GROUP BY子句中的列创建索引。而出现在查询列表中的列就没必要建立索引了

### 考虑列的基数

`列的基数`指的是某一列中不重复数据的个数，比方说某个列包含值2, 5, 8, 2, 5, 8, 2, 5, 8，虽然有9条记录，但该列的基数却是3。这个列的基数指标非常重要，直接影响我们是否能有效的利用索引。假设某个列的基数为1，也就是所有记录在该列中的值都一样，那为该列建立索引是没有用的，**因为所有值都一样就无法排序**. 而且如果某个建立了二级索引的列的重复值特别多，那么使用这个二级索引查出的记录还可能要做回表操作，这样性能损耗就更大了

### 索引列的类型尽量小

> 数据类型越小，在查询时进行的比较操作越快
> 数据类型越小，索引占用的存储空间就越少，在一个数据页内就可以放下更多的记录，从而减少磁盘I/O带来的性能损耗，也就意味着可以把更多的数据页缓存在内存中，从而加快读写效率

### 让索引列在比较表达式中单独出现

看一下下面两条语句的区别:

> WHERE my_col * 2 < 4
> WHERE my_col < 4/2

第1个WHERE子句中my_col列并不是以单独列的形式出现的，而是以my_col * 2这样的表达式的形式出现的，存储引擎会依次遍历所有的记录，计算这个表达式的值是不是小于4，**所以这种情况下是使用不到为my_col列建立的B+树索引的**。而第2个WHERE子句中my_col列并是以单独列的形式出现的，这样的情况可以直接使用B+树索引

### 主键插入顺序

如果我们插入数据时, 数据的主键忽大忽小, 则可能会引起原来排列好的数据页发生`页分列`, 就意味着性能损耗. 所以我们建议主键一般设置成为自增序列, 让数据库为我们**自己生成主键值, 这样在插入数据的时候, 一般不会产生页分列现象.**

```sql
CREATE TABLE person_info(
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    birthday DATE NOT NULL,
    phone_number CHAR(11) NOT NULL,
    country varchar(100) NOT NULL,
    PRIMARY KEY (id),
    KEY idx_name_birthday_phone_number (name(10), birthday, phone_number)
);   
```

### 不要建立重复索引

```sql
复制CREATE TABLE repeat_index_demo (
    c1 INT PRIMARY KEY,
    c2 INT,
    UNIQUE uidx_c1 (c1),
    INDEX idx_c1 (c1)
); 
```

主键已经是聚簇索引了, 又给 c1 列添加了唯一索引以及普通索引. 这种情况要避免.