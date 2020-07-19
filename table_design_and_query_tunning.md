# 表设计和查询优化
--

在本实验中，您将分析压缩，反范式，分布和排序对Redshift查询性能的影响。

## 结果集缓存和执行计划重用
当知道基础表中的数据未更改时，Redshift使结果集缓存可以加快数据检索速度。当仅查询的谓词已更改时，它还可以重新使用已编译的查询计划。

- 
执行以下查询，并记下查询执行时间。由于这是此查询的首次执行，因此Redshift将需要编译查询以及缓存结果集。

```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;

```

- 第二次执行相同的查询，并记下查询执行时间。在第二次执行中，redshift将利用结果集缓存并立即返回。

```
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;

```

- 更新表中的数据，然后再次运行查询。当基础表中的数据发生更改时，Redshift将知道更改，并使与查询关联的结果集缓存无效。请注意，执行时间不会比第2步快，而是比第1步快，因为它虽然无法重用缓存，但可以重用已编译的计划。

```
UPDATE customer
SET c_mktsegment = c_mktsegment
WHERE c_mktsegment = 'MACHINERY';

VACUUM DELETE ONLY customer;

SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer c
JOIN orders o on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;

```

使用`PREDICATE`执行新查询，并记下查询执行时间。由于这是此查询的首次执行，因此Redshift将需要编译查询以及缓存结果集。

```
SELECT c_mktsegment, count(1)
FROM Customer c
WHERE c_mktsegment = 'MACHINERY'
GROUP BY c_mktsegment;

```

使用稍有不同的`PREDICATE`执行查询，请注意，即使扫描和聚合的数据量非常相似，执行时间也比之前的执行时间要快。此行为是由于仅谓词已更改，因此重新使用了编译缓存。这种类型的模式通常用于BI报表，其中SQL模式与不同用户检索与不同谓词关联的数据保持一致。

```
SELECT c_mktsegment, count(1)
FROM customer c
WHERE c_mktsegment = 'BUILDING'
GROUP BY c_mktsegment;

```

## 选择性过滤

Redshift利用了区域映射的优势，它允许优化器在知道过滤器标准不匹配时跳过读取数据块。对于`orders_v3`表，由于我们在`o_order_date`上定义了一个排序键，因此利用该字段作为`PREDICATE`的查询将更快地返回。

- 
两次执行以下查询，注意第二次执行的执行时间。第一步是确保计划被编译。第二点更能代表最终用户的体验。

`注意`：将`enable_result_cache_for_session`设置为`false`可以确保不使用结果集缓存。

```
set enable_result_cache_for_session to false;

SELECT count(1), sum(o_totalprice)
FROM orders
WHERE o_orderdate between '1992-07-05' and '1992-07-07';

SELECT count(1), sum(o_totalprice)
FROM orders
WHERE o_orderdate between '1992-07-05' and '1992-07-07';

```


执行以下查询，注意第二次执行的执行时间。同样，第一个查询是确保计划已编译。第二个过滤条件稍有不同，以确保无法使用结果缓存。

```
set enable_result_cache_for_session to false;

SELECT count(1), sum(o_totalprice)
FROM orders
where o_orderkey < 600001;

SELECT count(1), sum(o_totalprice)
FROM orders
where o_orderkey < 600001;

```

执行以下操作以捕获每个查询的执行时间。您将注意到第二个查询所花费的时间明显长于上一步中的第二个查询，即使所汇总的行数相似。这是由于第一个查询具有利用表上定义的排序键的能力。

```
SELECT query, TRIM(querytxt) as SQL, starttime, endtime, DATEDIFF(microsecs, starttime, endtime) AS duration
FROM STL_QUERY
WHERE TRIM(querytxt) like '%orders%'
ORDER BY starttime DESC
LIMIT 4;

```

## Join 策略	

因为或Redshift的分布式体系结构，为了处理连接在一起的数据，可能必须将数据从一个节点广播到另一个节点。重要的是要分析查询的解释计划，以识别正在使用的联接策略以及如何对其进行改进。


对以下查询执行EXPLAIN。将这些表加载到	`LAB 2-数据导入`中后，将`Customer`表的`DISTSTYLE ALL`设置为`ALL`，为`order`表设置`DISTKEY（o_orderkey）`。对于较小的维度表，ALL分布是一个好习惯, 可应用到`Hash Join DS_DIST_ALL_NONE`的连接策略和相对较低的成本。

```
EXPLAIN
SELECT c_mktsegment, o_orderpriority, sum(o_totalprice)
FROM customer
JOIN orders on c_custkey = o_custkey
GROUP BY c_mktsegment, o_orderpriority;

```

创建新版本的客户表，该版本分发在客户表的标识符上。执行EXPLAIN并注意，这会导致以较高的成本生成“ DS_BCAST_INNER”联接策略。这是由于以下事实：两个表都不位于同一位置，并且必须广播内部表customer_v1中的数据才能完成连接。

现在执行两次查询，注意第二次执行的执行时间。第一步是确保计划被编译。第二点更能代表最终用户的体验。

```
set enable_result_cache_for_session to false;

SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;
```

创建一个新版本的客户表，该客户表使用客户密钥分发。执行EXPLAIN并注意，这将导致成本较高的`DS_BCAST_INNER`联接策略。这是由于以下原因：客户表或订单表都不位于同一位置，并且必须广播内部表中的数据才能完成连接。

```
DROP TABLE IF EXISTS customer_v1;
CREATE TABLE customer_v1
DISTKEY (c_custkey) as
SELECT * FROM customer;

EXPLAIN
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer_v1 c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;

```

现在执行两次查询，注意第二次执行的执行时间。第一步是确保计划被编译。第二点更能代表最终用户的体验。

```
set enable_result_cache_for_session to false;

SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders on l_orderkey = o_orderkey
JOIN customer_v1 c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;

```

最后，创建新版本的订单表，该表使用EVEN发行版进行分发。执行一个EXPLAIN并注意，当将大的lineitem表连接到orders表时，这导致连接策略为`DS_DIST_INNER`，因为它们没有分布在同一键上。另外，将这些结果加入到客户表中时，数据需要广播到节点，如`DS_BCAST_INNER`加入策略所证明的。

```
DROP TABLE IF EXISTS order_v1;
CREATE TABLE order_v1
DISTSTYLE EVEN as
SELECT * FROM order;

EXPLAIN
SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem
JOIN orders_v1 on l_orderkey = o_orderkey
JOIN customer_v1 c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;

```

现在执行两次查询，注意第二次执行的执行时间。第一步是确保计划被编译。第二点更能代表最终用户的体验。

```
set enable_result_cache_for_session to false;

SELECT c_mktsegment,COUNT(o_orderkey) AS orders_count, sum(l_quantity) as quantity, sum (l_extendedprice) as extendedprice
FROM lineitem_v2
JOIN orders_v1 on l_orderkey = o_orderkey
JOIN customer c on o_custkey = c_custkey
WHERE l_commitdate between '1992-01-01T00:00:00Z' and '1992-12-31T00:00:00Z'
GROUP BY c_mktsegment;

```
