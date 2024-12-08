# 交集（INTERSECT）、并集（UNION）和补集（EXCEPT）
## 交集
#### SQL代码格式
```sql
--取交集不去重。
<select_statement1> intersect all <select_statement2>;
--取交集并去重。intersect效果等同于intersect distinct。
<select_statement1> intersect [distinct] <select_statement2>;
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_statement1、select_statement2 |必填 | select语句|
|distinct |可选| 对两个数据集取交集的结果去重 |
#### 使用示例
```sql
--对两个数据集取交集，不去重
select * from values (1, 2), (1, 2), (3, 4), (5, 6) t(a, b) 
intersect all 
select * from values (1, 2), (1, 2), (3, 4), (5, 7) t(a, b);

--返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 1          | 2          |
| 3          | 4          |
+------------+------------+
```
```sql
--对两个查询结果取交集并去重
select * from values (1, 2), (1, 2), (3, 4), (5, 6) t(a, b) 
intersect distinct 
select * from values (1, 2), (1, 2), (3, 4), (5, 7) t(a, b);
--等效于如下语句。
select distinct * from 
(select * from values (1, 2), (1, 2), (3, 4), (5, 6) t(a, b) 
intersect all 
select * from values (1, 2), (1, 2), (3, 4), (5, 7) t(a, b)) t;

-返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 3          | 4          |
+------------+------------+
```

## 并集
#### SQL代码格式
```sql
--取并集不去重。
<select_statement1> union all <select_statement2>;
--取并集并去重。
<select_statement1> union [distinct] <select_statement2>;
```
#### SQL注意事项
* 存在多个`union all`，支持通过括号指定`union all`的优先级
* `union`后如果有`cluster by`、`distribute by`、`sort by`、`order by`或`limit`子句时，如果设置`set odps.sql.type.system.odps2=false`;，其作用于`union`的最后一个`select_statement`；如果设置`set odps.sql.type.system.odps2=true`;时，作用于前面所有`union`的结果。

|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_statement1、select_statement2 |必填 | select语句|
|distinct |可选| 对两个数据集取并集的结果去重 |
#### SQL使用示例
```sql
--对两个数据集取并集、不去重
select * from values (1, 2), (1, 2), (3, 4) t(a, b) 
union all 
select * from values (1, 2), (1, 4) t(a, b);

--返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 1          | 2          |
| 3          | 4          |
| 1          | 2          |
| 1          | 4          |
+------------+------------+
```
```sql
--对两个数据集取并集并去重
select * from values (1, 2), (1, 2), (3, 4) t(a, b)
union distinct 
select * from values (1, 2), (1, 4) t(a, b);
--等效于如下语句。
select distinct * from (
select * from values (1, 2), (1, 2), (3, 4) t(a, b) 
union all 
select * from values (1, 2), (1, 4) t(a, b));

--返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 1          | 4          |
| 3          | 4          |
+------------+------------+
```
```sql
--通过括号指定union all的优先级
select * from values (1, 2), (1, 2), (5, 6) t(a, b)
union all 
(select * from values (1, 2), (1, 2), (3, 4) t(a, b)
union all 
select * from values (1, 2), (1, 4) t(a, b));

--返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 1          | 2          |
| 5          | 6          |
| 1          | 2          |
| 1          | 2          |
| 3          | 4          |
| 1          | 2          |
| 1          | 4          |
+------------+------------+
```
```sql
--union后有cluster by、distribute by、sort by、order by或limit子句，设置set odps.sql.type.system.odps2=true;属性
set odps.sql.type.system.odps2=true;
select explode(array(3, 1)) as (a) union all select explode(array(0, 4, 2)) as (a) order by a limit 3;

--返回结果
+------------+
| a          |
+------------+
| 0          |
| 1          |
| 2          |
+------------+
```
```sql
--union后有cluster by、distribute by、sort by、order by或limit子句，设置set odps.sql.type.system.odps2=false;属性。

--返回结果
+------------+
| a          |
+------------+
| 3          |
| 1          |
| 0          |
| 2          |
| 4          |
+------------+
```
## 补集
#### SQL命令格式
```sql
--取补集不去重。
<select_statement1> except all <select_statement2>;
<select_statement1> minus all <select_statement2>;
--取补集并去重。
<select_statement1> except [distinct] <select_statement2>;
<select_statement1> minus [distinct] <select_statement2>;
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_statement1、select_statement2 |必填 | select语句|
|distinct |可选| 对取补集的结果去重 |
#### SQL使用示例
```sql
--求数据集的补集，不去重
select * from values (1, 2), (1, 2), (3, 4), (3, 4), (5, 6), (7, 8) t(a, b)
except all 
select * from values (3, 4), (5, 6), (5, 6), (9, 10) t(a, b);
--等效于如下语句。
select * from values (1, 2), (1, 2), (3, 4), (3, 4), (5, 6), (7, 8) t(a, b)
minus all 
select * from values (3, 4), (5, 6), (5, 6), (9, 10) t(a, b);

--返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 1          | 2          |
| 3          | 4          |
| 7          | 8          |
+------------+------------+
```
```sql
--求数据集的补集并去重
select * from values (1, 2), (1, 2), (3, 4), (3, 4), (5, 6), (7, 8) t(a, b)
except distinct 
select * from values (3, 4), (5, 6), (5, 6), (9, 10) t(a, b);
--等效于如下语句。
select * from values (1, 2), (1, 2), (3, 4), (3, 4), (5, 6), (7, 8) t(a, b)
minus distinct 
select * from values (3, 4), (5, 6), (5, 6), (9, 10) t(a, b);
--等效于如下语句。
select distinct * from values (1, 2), (1, 2), (3, 4), (3, 4), (5, 6), (7, 8) t(a, b) except all select * from values (3, 4), (5, 6), (5, 6), (9, 10) t(a, b);

--返回结果
+------------+------------+
| a          | b          |
+------------+------------+
| 1          | 2          |
| 7          | 8          |
+------------+------------+
```

# JOIN
## JOIN语法注意事项
* 使用`JOIN`时，会在计算中自动加入`JOIN`的`key is not null`的过滤条件，去除关联键为`NULL`的值所在行
* `join`操作的使用限制有两个：一是不支持无`on`条件的连接，二是只允许出现`and`连接的等值条件，但可以通过mapjoin操作使用不等值连接或or连接多个条件
## SQL命令格式
```sql
<table_reference> join <table_factor> [<join_condition>]
| <table_reference> {left outer|right outer|full outer|inner|natural} join <table_reference> <join_condition>
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_reference |必填 | 待执行`join`操作的左表查询语句。格式为`table_name [alias] | table_query [alias] |... `|
|table_factor |必填| 待执行`join`操作的右表或表查询语句。格式为`table_name [alias] | table_subquery [alias] |... ` |
|join_condition |可选| `join`连接条件，是一个或多个等式表达式组合。格式为`on equality_expression [and equality_expression]...，equality_expression`为等式表达式。 |
## SQL示例数据
```sql
--创建分区表sale_detail和sale_detail_jt。
create table if not exists sale_detail
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);
create table if not exists sale_detail_jt
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

--向源表增加分区。
alter table sale_detail add partition (sale_date='2013', region='china') partition (sale_date='2014', region='shanghai');
alter table sale_detail_jt add partition (sale_date='2013', region='china');

--向源表追加数据。
insert into sale_detail partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s3','c3',100.3);
insert into sale_detail partition (sale_date='2014', region='shanghai') values ('null','c5',null),('s6','c6',100.4),('s7','c7',100.5);
insert into sale_detail_jt partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s5','c2',100.2);

查询表sale_detail和sale_detail_jt中的数据，命令示例如下：
select * from sale_detail;
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s3         | c3          | 100.3       | 2013       | china      |
| null       | c5          | NULL        | 2014       | shanghai   |
| s6         | c6          | 100.4       | 2014       | shanghai   |
| s7         | c7          | 100.5       | 2014       | shanghai   |
+------------+-------------+-------------+------------+------------+
select * from sale_detail_jt;
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s5         | c2          | 100.2       | 2013       | china      |
+------------+-------------+-------------+------------+------------+

--创建做关联的表。
create table shop as select shop_name, customer_id, total_price from sale_detail;
```
#### SQL使用示例
* 左连接
```sql
--由于表sale_detail_jt及sale_detail中都有shop_name列，因此需要在select子句中使用别名进行区分。
select a.shop_name as ashop, b.shop_name as bshop from sale_detail_jt a 
       left outer join sale_detail b on a.shop_name=b.shop_name;

--返回结果
+------------+------------+
| ashop      | bshop      |
+------------+------------+
| s2         | s2         |
| s1         | s1         |
| s5         | NULL       |
+------------+------------+
```
* 右连接
```sql
--由于表sale_detail_jt及sale_detail中都有shop_name列，因此需要在select子句中使用别名进行区分。
select a.shop_name as ashop, b.shop_name as bshop from sale_detail_jt a 
       right outer join sale_detail b on a.shop_name=b.shop_name;
--返回结果
+------------+------------+
| ashop      | bshop      |
+------------+------------+
| NULL       | s3         |
| NULL       | s6         |
| NULL       | null       |
| s2         | s2         |
| NULL       | s7         |
| s1         | s1         |
+------------+------------+
```
* 全连接
```sql
--由于表sale_detail_jt及sale_detail中都有shop_name列，因此需要在select子句中使用别名进行区分。
select a.shop_name as ashop, b.shop_name as bshop from sale_detail_jt a 
       full outer join sale_detail b on a.shop_name=b.shop_name;
--返回结果
+------------+------------+
| ashop      | bshop      |
+------------+------------+
| NULL       | s3         |
| NULL       | s6         |
| s2         | s2         |
| NULL       | null       |
| NULL       | s7         |
| s1         | s1         |
| s5         | NULL       |
+------------+------------+
```
* 内连接
```sql
--由于表sale_detail_jt及sale_detail中都有shop_name列，因此需要在select子句中使用别名进行区分。
select a.shop_name as ashop, b.shop_name as bshop from sale_detail_jt a 
       inner join sale_detail b on a.shop_name=b.shop_name;

--返回结果
+------------+------------+
| ashop      | bshop      |
+------------+------------+
| s2         | s2         |
| s1         | s1         |
+------------+------------+
```
* `join`与`where`相结合
```sql
--执行SQL语句。
select a.shop_name
    ,a.customer_id
    ,a.total_price
        ,b.total_price
from  (select * from sale_detail where region = "china") a
left join (select * from sale_detail_jt where region = "china") b
on   a.shop_name = b.shop_name;

--返回结果
+------------+-------------+-------------+--------------+
| shop_name  | customer_id | total_price | total_price2 |
+------------+-------------+-------------+--------------+
| s1         | c1          | 100.1       | 100.1        |
| s2         | c2          | 100.2       | 100.2        |
| s3         | c3          | 100.3       | NULL         |
+------------+-------------+-------------+--------------+
```
* 错误命令示例
```sql
select a.shop_name
    ,a.customer_id
    ,a.total_price
        ,b.total_price
from  sale_detail a
left join sale_detail_jt b
on   a.shop_name = b.shop_name
where  a.region = "china" and b.region = "china";

--返回结果
+------------+-------------+-------------+--------------+
| shop_name  | customer_id | total_price | total_price2 |
+------------+-------------+-------------+--------------+
| s1         | c1          | 100.1       | 100.1        |
| s2         | c2          | 100.2       | 100.2        |
+------------+-------------+-------------+--------------+
```

# MAPJOIN HINT
## mapjoin使用限制
* `mapjoin`在Map阶段会将指定表的数据全部加载在内存中，因此指定的表仅能为小表，且表被加载到内存后占用的总内存不得超过512 MB。由于MaxCompute是压缩存储，因此小表在被加载到内存后，数据大小会急剧膨胀。此处的512 MB是指加载到内存后的空间大小。
* `mapjoin`中`join`操作的限制：
  - `left outer join`的左表必须是大表
  - `right outer join`的右表必须是大表
  - 不支持`full outer join`
  - `inner join`的左表或右表均可以是大表

## mapjoin使用方法
您需要在`select`语句中使用Hint提示`/*+ mapjoin(<table_name>) */`才会执行`mapjoin`。
* 引用小表或子查询时，需要引用别名
* `mapjoin`支持小表为子查询
* 在`mapjoin`中，可以使用不等值连接或`or`连接多个条件
* `mapjoin`中多个小表用英文逗号（,）分隔，例如`/*+ mapjoin(a,b,c)*/`

##  SQL示例数据
```sql
--创建一张分区表sale_detail。
create table if not exists sale_detail
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

create table if not exists sale_detail_sj
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

--向源表增加分区。
alter table sale_detail add partition (sale_date='2013', region='china');
alter table sale_detail_sj add partition (sale_date='2013', region='china');

--向源表追加数据。
insert into sale_detail partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s3','c3',100.3);
insert into sale_detail_sj partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s5','c2',100.2),('s2','c2',100.2);
```

## SQL使用示例
```sql
-- 使用mapjoin查询
select /*+ mapjoin(a) */
        a.shop_name,
        a.total_price,
        b.total_price
from sale_detail_sj a join sale_detail b
on a.total_price < b.total_price or a.total_price + b.total_price < 500;

--返回结果
+-----------+-------------+--------------+
| shop_name | total_price | total_price2 |
+-----------+-------------+--------------+
| s1        | 100.1       | 100.1        |
| s2        | 100.2       | 100.1        |
| s5        | 100.2       | 100.1        |
| s2        | 100.2       | 100.1        |
| s1        | 100.1       | 100.2        |
| s2        | 100.2       | 100.2        |
| s5        | 100.2       | 100.2        |
| s2        | 100.2       | 100.2        |
| s1        | 100.1       | 100.3        |
| s2        | 100.2       | 100.3        |
| s5        | 100.2       | 100.3        |
| s2        | 100.2       | 100.3        |
+-----------+-------------+--------------+
```




