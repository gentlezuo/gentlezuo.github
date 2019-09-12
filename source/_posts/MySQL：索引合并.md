---
title: MySQL：索引合并
date: 2019-09-11 15:00:50
tags: 数据库
category: MySQL
---

当单表中使用到了多个索引，优化器就可能使用索引合并(index merge)。mysql官网上只说了是什么，并没有讲为什么。简单分析一下...
<!--more-->

- [索引合并](#%E7%B4%A2%E5%BC%95%E5%90%88%E5%B9%B6)
  - [不足](#%E4%B8%8D%E8%B6%B3)
  - [简介](#%E7%AE%80%E4%BB%8B)
  - [算法](#%E7%AE%97%E6%B3%95)
    - [index merge intersection access algorithm（索引合并交集访问算法）](#index-merge-intersection-access-algorithm%E7%B4%A2%E5%BC%95%E5%90%88%E5%B9%B6%E4%BA%A4%E9%9B%86%E8%AE%BF%E9%97%AE%E7%AE%97%E6%B3%95)
      - [执行流程](#%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B)
      - [必要条件](#%E5%BF%85%E8%A6%81%E6%9D%A1%E4%BB%B6)
    - [index merge union access algorithm（索引合并并集访问算法）](#index-merge-union-access-algorithm%E7%B4%A2%E5%BC%95%E5%90%88%E5%B9%B6%E5%B9%B6%E9%9B%86%E8%AE%BF%E9%97%AE%E7%AE%97%E6%B3%95)
      - [必要条件](#%E5%BF%85%E8%A6%81%E6%9D%A1%E4%BB%B6-1)
      - [执行流程](#%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B-1)
      - [例子](#%E4%BE%8B%E5%AD%90)
    - [index merge sort sort-union access algorithm （索引合并排序并集访问算法）](#index-merge-sort-sort-union-access-algorithm-%E7%B4%A2%E5%BC%95%E5%90%88%E5%B9%B6%E6%8E%92%E5%BA%8F%E5%B9%B6%E9%9B%86%E8%AE%BF%E9%97%AE%E7%AE%97%E6%B3%95)
      - [必要条件](#%E5%BF%85%E8%A6%81%E6%9D%A1%E4%BB%B6-2)
      - [执行流程](#%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B-2)
      - [例子](#%E4%BE%8B%E5%AD%90-1)
  - [实战](#%E5%AE%9E%E6%88%98)
    - [复现 intersect](#%E5%A4%8D%E7%8E%B0-intersect)
    - [复现 union](#%E5%A4%8D%E7%8E%B0-union)
    - [复现 sort-union](#%E5%A4%8D%E7%8E%B0-sort-union)
    - [复现因为组合索引没有完全覆盖而导致没有使用intersect](#%E5%A4%8D%E7%8E%B0%E5%9B%A0%E4%B8%BA%E7%BB%84%E5%90%88%E7%B4%A2%E5%BC%95%E6%B2%A1%E6%9C%89%E5%AE%8C%E5%85%A8%E8%A6%86%E7%9B%96%E8%80%8C%E5%AF%BC%E8%87%B4%E6%B2%A1%E6%9C%89%E4%BD%BF%E7%94%A8intersect)
    - [复现因为二级索引不是等值查询而导致没有使用intersect](#%E5%A4%8D%E7%8E%B0%E5%9B%A0%E4%B8%BA%E4%BA%8C%E7%BA%A7%E7%B4%A2%E5%BC%95%E4%B8%8D%E6%98%AF%E7%AD%89%E5%80%BC%E6%9F%A5%E8%AF%A2%E8%80%8C%E5%AF%BC%E8%87%B4%E6%B2%A1%E6%9C%89%E4%BD%BF%E7%94%A8intersect)
  - [参考](#%E5%8F%82%E8%80%83)

# 索引合并
当单表使用了多个索引，每个索引都可能返回一个结果集，mysql会将其求交集或者并集，或者是交集和并集的组合。也就是说一次查询中可以使用多个索引。

比如以下sql语句
```sql
SELECT * FROM tbl_name WHERE key1 = 10 OR key2 = 20;

SELECT * FROM tbl_name
  WHERE (key1 = 10 OR key2 = 20) AND non_key = 30;

SELECT * FROM t1, t2
  WHERE (t1.key1 IN (1,2) OR t1.key2 LIKE 'value%')
  AND t2.key1 = t1.some_col;

```

解释：
1. 对于第一条语句：使用索引并集访问算法，得到key1=10的主键有序集合，得到key2=20的主键有序集合，再进行求并集；最后回表
2. 对于第二条语句：先丢弃`non_key=30`,因为它使用不到索引，where语句就变成了`where key10 or key2=20`,使用索引先根据索引合并并集访问算法
3. 对于第三条语句：索引合并并集访问算法求出`(t1.key1 IN (1,2) OR t1.key2 LIKE 'value%'`；然后是join操作

## 不足
- 对于复杂的where子语句，包含了很深的and/or 嵌套，那么不会使用到索引合并技术
- 不适用于全文索引

## 简介
在使用explain 时，在type那一列会显示`index_merge`,key那一列是所有使用到的索引。

索引合并又包含三个算法，在explain中显示：
- using intersect
- using union
- using sort_union

接下来详细的看一下
## 算法

### index merge intersection access algorithm（索引合并交集访问算法）

#### 执行流程

它的工作流程是：
对于每一个使用到的索引进行查询，查询主键值集合，然后进行合并，求交集，也就是and运算。下面是使用到该算法的两种必要条件：

#### 必要条件
1. 二级索引是等值查询；如果是组合索引，组合索引的每一位都必须覆盖到，不能只是部分
   ```sql
   key_part1 = const1 AND key_part2 = const2 ... AND key_partN = constN
   ```
2. InnoDB表上的主键范围查询条件

例子：
```sql

-- 主键可以使范围查询，二级索引只能是等值查询
SELECT * FROM innodb_table
  WHERE primary_key < 10 AND key_col1 = 20;

-- 没有主键的情况
SELECT * FROM tbl_name
  WHERE key1_part1 = 1 AND key1_part2 = 2 AND key2 = 2;
```

会有这两条条件的原因：    
1. 它是会根据不同的索引条件查询出**主键集合**，然后进行归并（并集）
2. 当二级索引的值相同时，其叶子节点上的数据也是**根据主键排序**的
3. 如果存在主键和二级索引同时存在的情况，主键不是用来查询数据的，而是用来过滤数据。

1. 针对一个必要条件：如果二级索引上的条件是范围条件，那么得到的数据就没有按照主键顺序排序，合并之前需要排序。（根据求并集的算法）。
2. 针对组合索引：同理，组合索引也必须满足**等值（即所有部分都等值查询）**
3. 针对主键：主键可以是范围查询而不是等值查询，是因为主键就是根据主键值排好序的

求交集的算法：    
针对两个升序排序的数组，进行归并：逐个取出两个数组中的最小的值，如果相等，就放入结果集，否则将较小的数指针向后移动。时间复杂度O（N）

### index merge union access algorithm（索引合并并集访问算法）
容易看出，与上述的算法类似，不过是使用了or连接条件，求并集

#### 必要条件
使用该算法的必要条件：
- 二级索引是等值查询；如果是组合索引，组合索引的每一位都必须覆盖到，不能只是部分
- InnoBD表上的主键范围查询
- 符合 index merge intersect 的条件

#### 执行流程
执行流程与index merge intersect 类似，依旧是查询了有序的主键集合，然后进行求并集。

#### 例子
```sql
-- 无主键，or 连接
SELECT * FROM t1
  WHERE key1 = 1 OR key2 = 2 OR key3 = 3;

-- 既有and 也有or
SELECT * FROM innodb_table
  WHERE (key1 = 1 AND key2 = 2)
     OR (key3 = 'foo' AND key4 = 'bar') AND key5 = 5;
```

第一个例子中：就是简单的求并集   
第二个例子中：现根据`key1 = 1 AND key2 = 2`和`key3 = 'foo' AND key4 = 'bar'`使用了两次index merge interset 得到两个主键集合进行合并；然后再根据`key5=5`得到主键集合，然后进行两个主键集合的合并；然后根据有序的主键序列进行回表。

求并集的算法：  
两个有序的升序数组，从两个数组中取出最小的数，然后比较，如果相等并且等于目前最小值，就放入结果集中，向后移动两个指针；否则，将小的数添加到结果集中。

### index merge sort sort-union access algorithm （索引合并排序并集访问算法）
从上可以看到使用 index merge union 算法的条件太苛刻，更多的时候，会对二级索引适用范围查询而不是等值查询。因此出现了新的算法。

#### 必要条件
- 二级索引不必等值查询，联合索引也不必匹配所有的索引项

#### 执行流程  
根据索引查询得到主键集合，对于每个主键集合进行排序，然后求并集。

这样做的好处是扩展了使用条件，增加了使用的范围；缺点就是消耗更大了。

#### 例子
```sql
SELECT * FROM tbl_name
  WHERE key_col1 < 10 OR key_col2 < 20;

SELECT * FROM tbl_name
  WHERE (key_col1 > 10 OR key_col2 = 20) AND nonkey_col = 30;
```

## 实战

实践出真知，通过实践验证上述分析是否正确。

先创建一个一个表，包含主键，联合索引，单列索引，然后插入10000行数据。

```sql
create table t2(id int primary key, a int not null, b int not null,c int not null, d int not null,f int not null ,index idx_abc(a,b,c), index idx_d(d),index idx_f(f));

DELIMITER $$
CREATE PROCEDURE t2_copy()
BEGIN
SET @i=1;
WHILE @i<=10000 DO
INSERT INTO t2 VALUES(@i,@i,@i,@i,@i,@i);
SET @i=@i+1;
END WHILE;
END $$
DELIMITER ;

call tw_copy;

```

### 复现 intersect
执行sql语句：
```sql
mysql> explain select * from t2 where id>1000 and f=1000;
+----+-------------+-------+------------+-------------+---------------+---------------+---------+------+------+----------+---------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys | key           | key_len | ref  | rows | filtered | Extra                                       |
+----+-------------+-------+------------+-------------+---------------+---------------+---------+------+------+----------+---------------------------------------------+
|  1 | SIMPLE      | t2    | NULL       | index_merge | PRIMARY,idx_f | idx_f,PRIMARY | 8,4     | NULL |    1 |   100.00 | Using intersect(idx_f,PRIMARY); Using where |
+----+-------------+-------+------------+-------------+---------------+---------------+---------+------+------+----------+---------------------------------------------+

```
type列是`index_merge`,extra列是`using intersect`。   
分析：使用了主键范围产找，二级索引等值查询


### 复现 union

执行sql语句
```sql
mysql> explain select * from t2 where f=1000 or d=1000;

+----+-------------+-------+------------+-------------+---------------+-------------+---------+------+------+----------+---------------------------------------+
| id | select_type | table | partitions | type        | possible_keys | key         | key_len | ref  | rows | filtered | Extra                                 |
+----+-------------+-------+------------+-------------+---------------+-------------+---------+------+------+----------+---------------------------------------+
|  1 | SIMPLE      | t2    | NULL       | index_merge | idx_d,idx_f   | idx_f,idx_d | 4,4     | NULL |    2 |   100.00 | Using union(idx_f,idx_d); Using where |
+----+-------------+-------+------------+-------------+---------------+-------------+---------+------+------+----------+---------------------------------------+

```

### 复现 sort-union

可以看见tpye 是`index_merge` ,并且extra里实现`using union(idx_f,idx_d)`，也就是使用了 index merge union access algorithm。   
分析sql语句：使用了二级索引等值查询。

再执行sql 语句：
```sql
mysql> explain select * from t2 where id >1000 or a=2;
+----+-------------+-------+------------+-------------+-----------------+-----------------+---------+------+------+----------+------------------------------------------------+
| id | select_type | table | partitions | type        | possible_keys   | key             | key_len | ref  | rows | filtered | Extra                                          |
+----+-------------+-------+------------+-------------+-----------------+-----------------+---------+------+------+----------+------------------------------------------------+
|  1 | SIMPLE      | t2    | NULL       | index_merge | PRIMARY,idx_abc | idx_abc,PRIMARY | 4,4     | NULL | 4974 |   100.00 | Using sort_union(idx_abc,PRIMARY); Using where |
+----+-------------+-------+------------+-------------+-----------------+-----------------+---------+------+------+----------+------------------------------------------------+
```

type是`index_merge`, extra 是`using sort_union(idx_abc,primary)`,使用了 index merge sort-union access algorithm算法。   
分析sql 语句：使用了组合索引，但是没有使用联合索引中的所有项，使用了主键范围查询。

### 复现因为组合索引没有完全覆盖而导致没有使用intersect

sql语句：
```sql
mysql> explain select * from t2 where id>1000 and a=1000;
+----+-------------+-------+------------+------+-----------------+---------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys   | key     | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+-----------------+---------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | t2    | NULL       | ref  | PRIMARY,idx_abc | idx_abc | 4       | const |    1 |    49.99 | Using index condition |
+----+-------------+-------+------------+------+-----------------+---------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```

### 复现因为二级索引不是等值查询而导致没有使用intersect

sql语句：
```
mysql> explain select * from t2 where id>1000 and d<1000;
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t2    | NULL       | range | PRIMARY,idx_d | PRIMARY | 4       | NULL | 4973 |    10.04 | Using where |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+-------------+

```

## 参考
[https://dev.mysql.com/doc/refman/8.0/en/index-merge-optimization.html](https://dev.mysql.com/doc/refman/8.0/en/index-merge-optimization.html)   
[https://www.ywnds.com/?p=14468](https://www.ywnds.com/?p=14468)

