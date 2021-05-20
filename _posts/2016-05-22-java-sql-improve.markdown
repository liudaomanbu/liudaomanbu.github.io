---
layout:     post
title:      "项目性能优化实践和SQL优化总结"
subtitle:   "项目性能优化实践和SQL优化总结"
date:       2016-5-22
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - sql
    - Sql
    - database
    - Database
    - data base
    - Data Base
    - improve
    - Improve
    - optimize
    - Optimize
    - mysql
    - Mysql
    - oracle
    - Oracle
    - db2
    - Db2
    - sqlserver
    - Sqlserver
    - dba
    - Dba
    - 优化
    - limit
    - Limit
    - having
    - HAVING
    - in
    - IN
    - not in
    - not in
    - exists
    - EXISTS
    - not exists
    - NOT EXISTS
    - insert
    - INSERT
    - index
    - Index
    - composite index
    - Composite Index
    - 索引
    - 复合索引
    - like
    - LIKE
---


最近,由于项目某些功能被客户投诉速度太慢,我被分配任务负责优化这些功能的性能,并且主导进行一次项目整体的性能优化.

我接到任务后,**首先**要做的就是先**找出**这些功能中**主要耗时的部分**.

为了达到这个目的,自然要上性能分析软件,这里我采用的是YourKit Java Profiler.

在性能分析软件中,展示了各个层级各个方法的耗时,自然是比较容易的找出了主要耗时部分.

在找到主要耗时部分后,接下来要做的自然是着手优化了.

**各种典型的错误做法以及优化方法如下:**

**1.在循环中发起sql.通常同样存在没有在方法级上统一事务的问题(通常service中的方法所有sql都应该在一个事务中).**

**错误原因:**这自然是典型的错误做法,**将O(1)的时间复杂度变成了O(N)的时间复杂度**,99%的场景下,这种N次的sql都可以直接被简单的合并为循环外的一条sql,并且不增加其他方面的任何负担.

**如果这个sql不是select而是update、insert、delete这种操作数据的sql,那么没有在方法级上统一事务的问题不但会造成每句sql都执行事务,而且还会在出错时造成事务不统一的严重问题.**

这已经不是性能问题,而是功能上就有问题了.

**优化方式:将循环的多条sql用eq改为in等方式合并为一条sql.**

**2.为了开发效率,使用已经写好的处理集合的函数.**

**错误原因:需要进行多个操作时,实际是调用一次函数就对集合循环了一次,将O(N)的时间复杂度变成了O(2N)的时间复杂度,如果循环还有嵌套的话,时间复杂度可能是O(N²)、O(N³)等**

**优化方式:**

**(1)Java8**终于根据函数式编程思想加入的**stream**式操作方法就是为了解决这个问题而诞生的.

**(2)如果项目的环境还没升级到Java8,并且也不允许在短期内升级到Java8**,那也没关系.Guava这个谷歌开源的Java常用工具库早就在Java8诞生之前就为这个问题而努力,并且完成了替代方案.
**Guava的FluentIterable完全可以作为stream的替代品,如果项目没有使用Guava也不允许引入Guava,那么apache开源的common-lang常用工具库的FluentIterable也可以作为替代.**

(3)如果项目以上条件都不能满足,那就没办法了,要么忍受多次循环,要么每次都要重新实现方法插入循环中.

**3.sql存在优化空间,由于项目使用的ORM框架为mybatis,所以sql都在xml文件中,因此我写了一个读取xml文件的工具就能方便找出可以优化的sql**,下面是常见的有优化空间的sql,包括我的总结以及网上一些博客的总结.

**3.1 使用了连接词OR.**
``` sql
select col1,col2 from t1 where col1 =val or col2=val
```

**错误原因:除非or条件中的所有列都加入了索引,否则使用or会导致不使用索引**

**优化方式:**

**(1)保证or条件中的所有列都加入了索引.**

**(2)使用UNION ALL或UNION代替.**
``` sql
select  col1,col2…  from  t1 where col1 =val
union all
select  col1,col2…  from  t1 where col2=val
```

**提升级别:强**
  
**3.2 无条件使用in子查询**

**错误原因:in是把外表和内表作hash连接,而exists是对外表作loop循环,每次loop循环再对内表进行查询.如果查询的两个表大小相当,那么用in和exists差别不大.反之,子查询大表应该用exists,子查询小表应该用in.**

**优化方式:子查询大表应该用exists,子查询小表应该用in.**
``` sql
select * from t_small where col in(select col from t_big)　　               -->效率低,用到了T_small表上col列的索引；
select * from t_small where exists(select col from t_big where col=t_small.col)　　-->效率高,用到了t_big表上col列的索引;
select * from t_big where col in(select col from t_small)　　               -->效率高,用到了t_big表上col列的索引;
select * from t_big where exists(select col from t_small where col=t_big.col)　　-->效率低,用到了t_small表上col列的索引;
``` 

**提升级别:中**

**3.3 在不能保证表之间的关联字段为非空时使用not in子查询**
``` sql
select *  from t1  where col not in (select  col  from t2)
```

**错误原因:因为null无法放入hash桶中,所以在不能保证表之间的关联字段为非空时使用了not in,那么为了保证正确性,肯定不会使用索引,而是对内外表都进行全表扫描,时间复杂度从O(N)变成了O(N²)**

**优化方式:**

**(1)在表结构中,将内表与外表的关联字段设定为非空的.**

(2)查询时在内表与外表中过滤空值,where条件中加上is not null的条件(不具有实用性,需要逐句修改,没有统一性,且复杂sql修改较麻烦).

**(3)使用NOT EXISTS代替.**
``` sql
select * from t1 where exists(select col from t2 where a.col=b.col)
```
**注意:not in逻辑上不完全等同于not exists.**
``` sql
select * from t where col not in (val1,val2,...null)
```
上面的sql等价于
``` sql
select * from t where col <> val1 and col <> val2... and col <> null
```
而null这个值是特殊的,任何值和null进行任何比较结果都为false,col <> null的结果永远是false,所以如果in的条件集合中存在null(即使in后面是子查询语句而不是集合值,实际也是先执行子查询,得到结果集后填充会条件),结果集将会是空的.

同理,如果上面例句中的外表t1的col列也存在null值,那么结果集将会过滤这条数据,因为null <> val1也是永远返回false的.

而not exists的执行计划为hash join,以上两种情况对not exists的查询结果都没有任何影响.

**对于not exists查询,内表存在空值对查询结果没有影响；对于not in查询,内表存在空值将导致最终的查询结果为空.**
 
**对于not exists查询,外表存在空值,存在空值的那条记录最终会输出；对于not in查询,外表存在空值,存在空值的那条记录最终将被过滤,其他数据不受影响.**

**考虑到99%的情况下,业务上要求的应该都是not exists的逻辑,因此认为三种优化方式是等价的.**

**提升级别:强**

**3.4 使用了不等号<>或!=**
``` sql
select * from t where col<>val
```

**错误原因:除非sql查询的所有列都在索引中,否则即使条件字段有索引,使用了不等号<>或!=也会导致全表扫描(不太能理解其中原因,知道的人请指教)**

**优化方式:拆成<和>后用or或union all连接.**
``` sql
select * from t  where col<val
union all
select * from t where col>val
``` 

**提升级别:强**

**3.5 列字段上使用计算、使用函数、进行隐式类型转换**
``` sql
SELECT col FROM t WHERE col+10=30;
SELECT col1 FROM t WHERE concat(col1,col2) ='Jaskeyabc';
SELECT * FROM t WHERE col(列数据类型为字符串)=1
```

**错误原因:应该很好理解,索引中只存了数据的原始值,对数据做任何操作都只能进行转换**

**优化方式:**

**(1)将计算、函数移到右边或者逻辑转为代码处理**

**(2)避免隐式类型转换,保证字段类型和条件值类型统一**

**提升级别:强**

**3.6 like进行左匹配**
``` sql
select * from t where col like  “%abc”;
select * from t where col like  “%abc%”
```

**错误原因:数据库索引是遵循左匹配原则的,只有右匹配才走索引**

**优化方式:**

**(1)在业务上避免任何左匹配**

**(2)增加该字段的reverse字段存储reverse后的值,并使用like REVERSE('%XXX')模式来匹配**

``` sql
SELECT * FROM t WHERE `reverse_email` LIKE REVERSE('%.com')
```

**(3)使用搜索引擎来完成左匹配和全匹配功能**

**提升级别:强**

**3.7 在索引列上使用is null和is not null**

**错误原因:索引是使用hash桶的,null值无法加入hash桶中,所以使用is null和is not null无法使用索引**

**优化方式:**

**(1)在逻辑和设计上尽量保证索引列not null,插入数据时一定有有意义的值**

**(2)使用默认值代替null值**

**提升级别:强**

**3.8 错误使用复合索引,Group by、Order by、where等条件字段中没有遵守复合索引的字段顺序**

**错误原因:复合索引(a,b,c)在作用上约等于(a),(a,b)(a,c)(a,b,c)合一.即查询条件中有首字段存在才会使用该索引.**

除非所有记录a字段的值只有很少的几种,数据库才可能会在实质上默认b字段为首字段,跳过a字段进行扫描.
但是一定要避免出现这种情况,这样会令人对复合索引的生效条件产生疑惑,而且为了效率和查询性能考虑,我们应该将能过滤掉最多数据即值最多的字段作为首字段.当然,前提是该字段通常在查询条件中是必须存在的.

**优化方式:合理设计复合索引,并且尽量保证使用复合索引**

**提升级别:强**

**3.9 没有遵守尽量提前过滤中间结果集的原则**

**(1)join时没有先将在能够本表中过滤的数据过滤而是join完之后才对所有结果集过滤.**

**错误原因:where过滤完再join能极大减少需要关联的两表的数据条数,大量减少最终需要处理的结果集**

**优化方式:先将在能够本表中过滤的数据过滤,然后join过滤后的临时表**

**提升级别:中**

**(2)将能放在where中过滤的条件放入having后**

**错误原因:having只会在检索出所有记录之后才对结果集进行过滤.这个处理需要排序等操作.提前过滤能够减少需要这些操作的结果集条数.**

**优化方式:将可以提前过滤的条件放在where语句中**

**提升级别:中**

**3.10 大数据量下使用limit分页(仅限mysql)**

**优化原理:mysql数据库中,limit 10000,20的实现方式为在结果集中读取前10020条,然后再抛弃前面的10000条.**

**优化方式:如果id为自增情况下,可以使用**

``` sql
SELECT * FROM t WHERE id > 9527 ORDER BY id ASC LIMIT 20,20;
```
**之类的方式来提升性能,但是这跟排序方式,id策略等情况有关,较为麻烦**

**提升级别:中**

**3.11 循环进行insert操作**

**优化原理:大部分数据库存在一句insert插入多条数据的写法,效率比多句insert要高不少.但是由于这不是sql规范的内容,数据库并不一定有这个功能,并且各数据库写法不同.也暂时没有框架对这个功能的差异性进行封装.所以考虑到兼容性和通用性,慎用.**

**优化方式:使用一句insert插入多条数据的写法**

**提升级别:中**

**3.12 没有明确指定表名列名**

**优化原理:可以减少解析的时间并减少那些由Column歧义引起的语法错误.**

**优化方式:如不要使用\*代替列名,如使用别名方式保证多表关联时列名的明确性和唯一性**

**提升级别:弱**

**3.13 查询不需要用到的字段**

**优化原理:减少数据库IO开销和网络传输开销,如果产生了覆盖索引的情况,性能将有较大提升.**

**优化方式:只查询需要用到的字段**

**提升级别:弱**