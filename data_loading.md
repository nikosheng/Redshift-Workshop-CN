#数据导入
--

在本实验中，您将基于TPC Benchmark数据模型使用一组八个表。您可以在Redshift集群中创建这些表，并使用存储在S3中的样本数据加载这些表。

##创建表结构
复制以下创建表语句以在数据库中创建模拟TPC Benchmark数据模型的表。

```
DROP TABLE IF EXISTS partsupp;
DROP TABLE IF EXISTS lineitem;
DROP TABLE IF EXISTS supplier;
DROP TABLE IF EXISTS part;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS customer;
DROP TABLE IF EXISTS nation;
DROP TABLE IF EXISTS region;

CREATE TABLE region (
  R_REGIONKEY bigint NOT NULL PRIMARY KEY,
  R_NAME varchar(25),
  R_COMMENT varchar(152))
diststyle all;

CREATE TABLE nation (
  N_NATIONKEY bigint NOT NULL PRIMARY KEY,
  N_NAME varchar(25),
  N_REGIONKEY bigint REFERENCES region(R_REGIONKEY),
  N_COMMENT varchar(152))
diststyle all;

create table customer (
  C_CUSTKEY bigint NOT NULL PRIMARY KEY,
  C_NAME varchar(25),
  C_ADDRESS varchar(40),
  C_NATIONKEY bigint REFERENCES nation(N_NATIONKEY),
  C_PHONE varchar(15),
  C_ACCTBAL decimal(18,4),
  C_MKTSEGMENT varchar(10),
  C_COMMENT varchar(117))
diststyle all;

create table orders (
  O_ORDERKEY bigint NOT NULL PRIMARY KEY,
  O_CUSTKEY bigint REFERENCES customer(C_CUSTKEY),
  O_ORDERSTATUS varchar(1),
  O_TOTALPRICE decimal(18,4),
  O_ORDERDATE Date,
  O_ORDERPRIORITY varchar(15),
  O_CLERK varchar(15),
  O_SHIPPRIORITY Integer,
  O_COMMENT varchar(79))
distkey (O_ORDERKEY)
sortkey (O_ORDERDATE);

create table part (
  P_PARTKEY bigint NOT NULL PRIMARY KEY,
  P_NAME varchar(55),
  P_MFGR  varchar(25),
  P_BRAND varchar(10),
  P_TYPE varchar(25),
  P_SIZE integer,
  P_CONTAINER varchar(10),
  P_RETAILPRICE decimal(18,4),
  P_COMMENT varchar(23))
diststyle all;

create table supplier (
  S_SUPPKEY bigint NOT NULL PRIMARY KEY,
  S_NAME varchar(25),
  S_ADDRESS varchar(40),
  S_NATIONKEY bigint REFERENCES nation(n_nationkey),
  S_PHONE varchar(15),
  S_ACCTBAL decimal(18,4),
  S_COMMENT varchar(101))
diststyle all;                                                              

create table lineitem (
  L_ORDERKEY bigint NOT NULL REFERENCES orders(O_ORDERKEY),
  L_PARTKEY bigint REFERENCES part(P_PARTKEY),
  L_SUPPKEY bigint REFERENCES supplier(S_SUPPKEY),
  L_LINENUMBER integer NOT NULL,
  L_QUANTITY decimal(18,4),
  L_EXTENDEDPRICE decimal(18,4),
  L_DISCOUNT decimal(18,4),
  L_TAX decimal(18,4),
  L_RETURNFLAG varchar(1),
  L_LINESTATUS varchar(1),
  L_SHIPDATE date,
  L_COMMITDATE date,
  L_RECEIPTDATE date,
  L_SHIPINSTRUCT varchar(25),
  L_SHIPMODE varchar(10),
  L_COMMENT varchar(44),
PRIMARY KEY (L_ORDERKEY, L_LINENUMBER))
distkey (L_ORDERKEY)
sortkey (L_RECEIPTDATE);

create table partsupp (
  PS_PARTKEY bigint NOT NULL REFERENCES part(P_PARTKEY),
  PS_SUPPKEY bigint NOT NULL REFERENCES supplier(S_SUPPKEY),
  PS_AVAILQTY integer,
  PS_SUPPLYCOST decimal(18,4),
  PS_COMMENT varchar(199),
PRIMARY KEY (PS_PARTKEY, PS_SUPPKEY))
diststyle even;

```

##数据导入
COPY命令比使用INSERT语句更有效地加载大量数据，并且也更有效地存储数据。使用单个COPY命令从多个文件中加载一个表的数据。然后，Amazon Redshift自动并行加载数据。为了方便起见，将在公共Amazon S3存储桶中提供您将使用的样本数据。为确保Redshift执行压缩分析，请在COPY命令中将COMPUPDATE参数设置为ON。要复制此数据，您将需要替换以下脚本中的`[Your-AWS_Account_Id]`和`[Your-Redshift_Role]`值。

```
COPY region FROM 's3://redshift-immersionday-labs/data/region/region.tbl.lzo'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

COPY nation FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy customer from 's3://redshift-immersionday-labs/data/customer/customer.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy orders from 's3://redshift-immersionday-labs/data/orders/orders.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy part from 's3://redshift-immersionday-labs/data/part/part.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy supplier from 's3://redshift-immersionday-labs/data/supplier/supplier.json' manifest
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy lineitem from 's3://redshift-immersionday-labs/data/lineitem/lineitem.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

copy partsupp from 's3://redshift-immersionday-labs/data/partsupp/partsupp.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' COMPUPDATE PRESET;

```
如果您使用4个`dc2.large`集群节点，则估计的数据加载时间如下，请注意，您可以在redshift控制台的`Performance`和`Query`选项卡中检查有关操作的计时信息：

- REGION (5 rows) - 20s 
- NATION (25 rows) - 20s 
- CUSTOMER (15M rows) – 3m 
- ORDERS - (76M rows) - 1m
- PART - (20M rows) - 4m
- SUPPLIER - (1M rows) - 1m
- LINEITEM - (600M rows) - 13m
- PARTSUPPLIER - (80M rows) 3m

`注意：以上COPY语句的一些关键要点`

COMPUPDATE PRESET ON将使用与列的数据类型相关的Amazon Redshift最佳实践分配压缩，但不分析表中的数据。`REGION`表的COPY指向特定文件（`region.tbl.lzo`），而其他表的COPY则指向多个文件的前缀（`lineitem.tbl`）。`SUPPLIER`表的COPY指向清单文件（`supplier.json`）

##表维护 - Analyze
您应该定期更新查询计划者用来构建和选择最佳计划的统计元数据。您可以通过运行ANALYZE命令来显式分析表。使用COPY命令加载数据时，可以通过将STATUPDATE选项设置为ON来自动分析增量加载的数据。当加载到空表中时，默认情况下，COPY命令执行ANALYZE操作。


对`CUSTOMER`表运行`ANALYZE`命令。

```
analyze customer;
```

要查明何时运行`ANALYZE`命令，可以查询系统表并查看`STL_QUERY`和`STV_STATEMENTTEXT`等视图，并包括对`padb_fetch_sample`的限制。例如，要找出上次分析`CUSTOMER`表的时间，请运行以下查询：

```
select query, rtrim(querytxt), starttime
from stl_query
where
querytxt like 'padb_fetch_sample%' and
querytxt like '%customer%'
order by query desc;

```

`注意`：`ANALYZE`的时间时间戳将与执行`COPY`命令的时间相关，并且第二条分析语句将没有任何条目。 `Redshift`知道它不需要运行`ANALYZE`操作，因为表中没有数据已更改。

##表维护 - VACUUM
您应该在进行大量删除或更新后运行`VACUUM`命令。为了执行更新，Amazon Redshift会删除原始行并追加更新后的行，因此每次更新实际上都是删除和插入。虽然Amazon Redshift最近启用了一项自动定期回收空间的功能，但是最好知道如何手动执行此操作。您可以运行完全真空，仅删除真空或仅对真空排序。


捕获`ORDERS`表的初始空间使用情况。

```
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id and
stv_blocklist.slice = stv_tbl_perm.slice and
stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```

| col  |      count     |  
|----------|:-------------:|
| 0 |  352 |
| 1 |  352 |
| 2 |  64 |
| 3 |  448 |
| 4 |  64 |
| 5 |  128 |


从`ORDERS`表中删除行。

```
delete orders where o_orderdate between '1992-01-01' and '1993-01-01';

```

通过再次运行以下查询并注意值没有更改，确认`Redshift`没有自动回收空间。

```
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```

运行`VACUUM`命令

```
vacuum delete only orders;
```

通过再次运行以下查询并注意值已更改，确认`VACUUM`命令已回收空间。

```
select col, count(*)
from stv_blocklist, stv_tbl_perm
where stv_blocklist.tbl = stv_tbl_perm.id
and stv_blocklist.slice = stv_tbl_perm.slice
and stv_tbl_perm.name = 'orders' and
col <= 5
group by col
order by col;
```

| col  |      count     |  
|----------|:-------------:|
| 0 |  320 |
| 1 |  320 |
| 2 |  64 |
| 3 |  416 |
| 4 |  64 |
| 5 |  128 |

`注意`：如果您的表的列数很少，但行数却很大，则三个隐藏的元数据标识列（`INSERT_XID`，`DELETE_XID`，`ROW_ID`）将占用该表不成比例的磁盘空间。为了优化隐藏列的压缩，请尽可能在单个副本事务中加载表。如果使用多个单独的`COPY`命令加载该表，则`INSERT_XID`列将无法很好地压缩，并且多个真空操作将无法改善`INSERT_XID`的压缩。

##调试
有两个Amazon Redshift系统表可帮助您解决数据加载问题：

- `STL_LOAD_ERRORS`
- `STL_FILE_SCAN`

此外，您无需实际加载表即可验证数据。将`NOLOAD`选项与`COPY`命令一起使用，以确保在运行实际数据加载之前，将正确加载数据文件。使用`NOLOAD`选项运行`COPY`比加载数据快得多，因为它仅解析文件。

让我们尝试使用列不匹配的其他数据文件加载`CUSTOMER`表。要复制此数据，您将需要替换以下脚本中的`[Your-AWS_Account_Id]`和`[Your-Redshift_Role]`值。

```
COPY customer FROM 's3://redshift-immersionday-labs/data/nation/nation.tbl.'
iam_role 'arn:aws:iam::[Your-AWS_Account_Id]:role/[Your-Redshift-Role]'
region 'us-west-2' lzop delimiter '|' noload;

```


您将收到以下错误。

```
ERROR: Load into table 'customer' failed.  Check 'stl_load_errors' system table for details. [SQL State=XX000]

```

查询`STL_LOAD_ERROR`系统表以获取详细信息。

```
select * from stl_load_errors;
```

您还可以创建一个视图，以返回有关加载错误的详细信息。下面的示例将`STL_LOAD_ERRORS`表连接到`STV_TBL_PERM`表，以将表ID与实际表名进行匹配。

```
create view loadview as
(select distinct tbl, trim(name) as table_name, query, starttime,
trim(filename) as input, line_number, colname, err_code,
trim(err_reason) as reason
from stl_load_errors sl, stv_tbl_perm sp
where sl.tbl = sp.id);

-- Query the LOADVIEW view to isolate the problem.
select * from loadview where table_name='customer';

```