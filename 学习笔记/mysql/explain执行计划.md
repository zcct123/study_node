# SQL 查询优化
## 1.前言
#### 1.SQL Select 语句完整的执行顺序
1. from 子句组装来自不同数据源的数据
2. where 子句基于指定的条件对记录行进行筛选
3. group by 子句将数据划分为多个分组； 
4. 使用聚集函数进行计算； 
5. 使用 having 子句筛选分组； 
6. 计算所有的表达式； 
7. select 的字段； 
8. 使用 order by 对结果集进行排序;

``` java
(1) FROM <left_table> 
(2) <join_type> JOIN <right_table> 
(3) ON <join_condition> 
(4) WHERE <where_condition> 
(5) GROUP BY <group_by_list>
(6) WITH {CUBE | ROLLUP} 
(7) HAVING <having_condition> 
(8) SELECT (9) DISTINCT 
(9) ORDER BY <order_by_list> 
(10) <TOP_specification> <select_list>
//以上每个步骤都会产生一个虚拟表，该虚拟表被用作下一个步骤的输入。
//这些虚拟表对调用者(客户端应 用程序或者外部查询)不可用。
//只有最后一步生成的表才会会给调用者。如果没有在查询中指定某一个子句， 将跳过相应的步骤。
```
## 2. explain 介绍
 **EXPLAIN 用来获取关于查询额执行计划信息**
 ``` sql
 EXPLAIN  SELECT * FROM  config_info
 ```
 ![](/img/2022-02-15-18-00-57.png)
 > mysql explain 只能解释select查询，并不会对存储过程调用，insert、update、delete或其他语句做解释，可以重写非select查询以利用explain1
### 1. explain中的列
* id列
    **标识select所属的行**
    > MySql 将 select 查询分为简单和复杂查询，复杂查询可分为三大类 ：简单子查询、from中的子查询、UNION 查询
     ``` sql
     # 简单子查询  table 来源于两个基本表
    EXPLAIN SELECT  (SELECT 1 FROM config_info LIMIT 1 ) FROM config_info_tag
    ```
    ![](/img/屏幕截图%202022-02-15%20181600.png)
    ``` sql
    # 基本子查询  查询时产生一个匿名临时表，通过别名在外层查询中引用这个临时表
    EXPLAIN SELECT id FROM  (SELECT id  FROM config_info LIMIT 1 ) as tem
    ```
    ![](/img/屏幕截图%202022-02-15%20181920.png)
    ``` sql
    # UNION查询  UNION 结果总是放在一个匿名临时表中，临时表不在原sql中出现，因此他的id列是null，与之前相比，这个查询产生的零食表放在最后一行
    EXPLAIN SELECT 1 UNION SELECT 1
    ```
    ![](/img/屏幕截图%202022-02-15%20182224.png)
* select_type列
    **这一列显示了对应行是简单还是复杂 select ，如果是复杂类型的查询，则会标记出是三种复杂查询的哪一种**    
    1. SIMPLE 意味着查询不包括子查询和UNION
    2. PRIMARY (主要的) 如果查询有任何复杂的子部分，泽最外层部分标记为  PRIMARY
    3. SUBQUERY (子查询) 子查询不在 from 中标记为SUBQUERY
    4. DERIVED (衍生) 表示包含在 from 中的子查询中的 select，Mysql 会递归执行并京结果放到一个临时表中，也叫做派生表，因为该临时表是从子查询中派生来的
    5. UNION 在 union 中的第二个和随后的 SELECT 被标记为 union，第一个 select 被标记就好像他以部分外查询来之陷阱
    6. UNION RESULT 用来从 union 的匿名临时表检索结果的 select 被标记为 union result 

** 除了这些值， SHUBQUERY 和  UNION 还可以被标记为 dependent 和 uncacheable，dependent意味着 select 依赖于外层查询中发现的数据； uncacheable 意味着 select 中的默写特性组织结果被缓存与一个 item_cache中。**
* table 列
**这一列对应行正在访问的拿一个表，或者是该表的别名**
**派生表和联合：** 
  1. 当 from 语句中有子查询或有 union 时， table的列就会变多，没有一个表可以被参考到
  2. 当 from 语句中有子查询时，table 列是 < derivedN >的形式，其中 N 是子查询的 id，也就是 对子查询结果的引用。(向前引用)
   ```sql
    EXPLAIN SELECT id FROM  (SELECT id  FROM config_info LIMIT 1 ) as a
   ```
    ![](/img/屏幕截图%202022-02-15%20192427.png)
  3. 当有 union 时，UNION RESULT 的 table 列包含一个参与 UNION 的 id 列表。 这总是 向后引用，因为UNION RESULT 出现在 union 中所参与的行之后
   ```sql
    EXPLAIN SELECT id  FROM config_info  UNION SELECT id FROM config_info_tag UNION SELECT id FROM config_info_beta
   ```
   ![](/img/屏幕截图%202022-02-15%20192903.png)
* type列
  **这一列显示 “关联类型” 更准确的说法是访问类型，—— 换言之就是 MySql 决定如何查找表中的行**
  1. ALL 人称 全表扫描，意味着 Mysql 必须扫描整张表，从头到尾找到需要的行（除了 Limit 或者 Extra 列中显示 “Useing distinct/not exists”）
    ``` sql
     SELECT * FROM  config_info name = 'ceshi'
    ```
  2. index 这个和全表扫描一样，只是 mysql 扫描表时按索引次序进行而不是行，他的主要优点时避免了排序；最大的缺点是要承担按索引次序读取整个表的开销，若是随机次序访问行，开销会很大，若是 Extra 列显示 use index 说明使用覆盖索引   
    ``` sql
     # name 为 btree索引
     EXPLAIN  SELECT name FROM  config_info 
    ```
  3. range 范围扫描就是一个有限制的索引扫描，它开始于索引的摸一个顶啊，返回匹配这个值域的所有行，一般是带有 between 或者 where > 的查询， in（） 和 or 列表 也会显示为范围查找。然而两者是不同的访问类型   （in会对列表排序 or 不会）
  4. ref 这是一种索引访问（也叫做索引查找），他返回所有匹配某个单个行的值。然而，他可能找到多个符合条件的行，因此，他是查找和扫面的混合体，此类索引访问当使用**非唯一性索引**或者**唯一性前缀**才会发生。把他叫做 ref 是因为**索引要和某一参考值相比较**，这个参考或者是一个常数，或者自多表查询前一个表里的结果值 。 ref_or_null 是 ref 之上的一个变体，它意味着 mysql 必须在初次查询的结果里进行第二次查找中以找出 null 的条目
  5. eq_ref 使用索引查找， 可以在使用主键或者唯一性索引查找时看到。
  6.  const，system 当mysql 能对查询的某部分进行优化并将其转换成一个常量时，他就会使用这种类型。比如，通过一行的主键进行where查询时，mysql 就能把这个查询转换成一个常量，然后就可以搞笑地将表从联接执行中移除
    ![](/img/屏幕截图%202022-02-15%20205422.png) 
  7. NULL  意味着 Mysql 能在优化阶段分解查询语句，再执行阶段甚至用不着在访问表或者索引
* possible_keys   
    **这一列显示了查询可以使用的索引，这是基于查询访问的列和使用的比较操作符来判定的，这些列表时在优化过程中的早期创建的，因此有些罗列出来的索引可能对后续的优化过程中没有用**
* key 列
  **这一列显示了 Mysql 静定采用马哥索引来优化对该表的访问，如果该索引没有出现在 ssible_keys 中，可能是训责了覆盖索引  换句话说 possible_keys 解释了哪一个索引能有助于高效的进行查找，key 显示的是优化采用哪一个索引可以最小化查询成本**
* key_len 列
    **该列显示了 Mysql 在索引中使用的字节数。**
* ref 列
    **这一列显示了之前的表 key 列记录中的索引中查找值所用的列或者常量 **    
* row 列
    **这一列是 mysql 估计为了查找到所需要的行为要读取的行数 他不是 mysql 认为他最终要从表里读出的行数，而是 mysql 为了找到符合条件的查询的每一个点上标准的那些行为而必须读取的平均行数**
* filtered 列
  **他先是的是针对表里符合条件的记录数的百分比所作的一个悲观估算**
* Extra 列   
  **这一列包含的是不适合在其他列显示的额外信息**
  1. Using index 表示 mysql 将使用覆盖索引
  2. Using where 意味着 mysql 服务器 将在存储引擎检索行后在进行过滤。 许多 where 条件设计索引的列，当他读取索引时，就能被存储引擎检测
  3. Using temprary 意味着 mysql 对结果进行排序时会使用临时表
  4. Using fitersort 意味着 Mysql 会对结果使用一个外部索引排序，为不是按索引次序从表中读取行。
  5. Range checked for each record （index map:N）意味着没有好用的索引, 新的索引将在联接的每一行上进行重新估算。N显示的在 possible_key列中索引的位图，并且是冗余的
