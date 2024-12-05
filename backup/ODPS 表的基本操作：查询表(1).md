# 查询表基本语法：SELECT语法
## 总体格式
#### SQL代码格式
```sql
[with <cte>[, ...] ]
SELECT [all | distinct] <SELECT_expr>[, <except_expr>][, <replace_expr>] ...
       from <table_reference>
       [where <where_condition>]
       [group by {<col_list>|rollup(<col_list>)}]
       [having <having_condition>]
       [window <window_clause>]
       [order by <order_condition>]
       [distribute by <distribute_condition> [sort by <sort_condition>]|[ cluster by <cluster_condition>] ]
       [limit <number>];
```
#### 源数据示例
```sql
--创建一张分区表sale_detail。
create table if not exists sale_detail
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

--向源表增加分区。
alter table sale_detail add partition (sale_date='2013', region='china');

--向源表追加数据。
insert into sale_detail partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s3','c3',100.3);

--查询分区表sale_detail中的数据
SELECT * from sale_detail;
--返回结果
+------------+------------+------------+------------+------------+
| shop_name  | price      | customer   | sale_date  | region     |
+------------+------------+------------+------------+------------+
| s1         | 100.1      | c1         | 2013       | china      |
| s2         | 100.2      | c2         | 2013       | china      |
| s3         | 100.3      | c3         | 2013       | china      |
+------------+------------+------------+------------+------------+
```

## WITH子句（Common Table Expression, cte）：可选
WITH子句包含一个或多个常用的表达式CTE，这些CTE充当当前运行环境中的临时表，可以在后续查询中引用。
#### CTE使用规则
* 在同一WITH子句中的CTE必须具有唯一的名字
* 在WITH子句中定义的CTE仅对在同一WITH子句中的其他CTE可以使用
#### 错误命令示例
```sql
--A引用A，无效
with 
A as (SELECT 1 from A) 
SELECT * from A;
```
```sql
--A引用B，B引用A，无效
with 
A as (SELECT * from B ), 
B as (SELECT * from A ) 
SELECT * from B;
```
#### 正确命令示例
```sql
with 
A as (SELECT 1 as C),
B as (SELECT * from A) 
SELECT * from B;
--返回结果
+---+
| c |
+---+
| 1 |
+---+
```

## 列表达式（SELECT_expr）：必填
`SELECT_expr`格式为`col1_name, col2_name, 列表达式, ...`，表示待查询的普通列、分区列或正则表达式。
#### 列表达式使用规则
* 用列名指定要读取的列
```sql
--读取表的列
SELECT shop_name from sale_detail;

--返回结果
+------------+
| shop_name  |
+------------+
| s1         |
| s2         |
| s3         |
+------------+
```
* 用星号（\*）代表查询所有的列，可以配合where子句指定过滤条件
```sql
--读取表中所有列
SELECT * from sale_detail;

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s3         | c3          | 100.3       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```
```sql
--在where子句中指定过滤条件
SELECT * from sale_detail where shop_name='s1';

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```
* 可以使用正则表达式
```sql
--选出表中所有列名以sh开头的列
SELECT `sh.*` from sale_detail;

--返回结果
+------------+
| shop_name  |
+------------+
| s1         |
| s2         |
| s3         |
+------------+
```
```sql
--选出表中所有列名不为shop_name的所有列
SELECT `(shop_name)?+.+` from sale_detail;

--返回结果
+-------------+-------------+------------+------------+
| customer_id | total_price | sale_date  | region     |
+-------------+-------------+------------+------------+
| c1          | 100.1       | 2013       | china      |
| c2          | 100.2       | 2013       | china      |
| c3          | 100.3       | 2013       | china      |
+-------------+-------------+------------+------------+
```
```sql
--选出表中排除shop_name和customer_id两列的其他列
SELECT `(shop_name|customer_id)?+.+` from sale_detail;

--返回结果
+-------------+------------+------------+
| total_price | sale_date  | region     |
+-------------+------------+------------+
| 100.1       | 2013       | china      |
| 100.2       | 2013       | china      |
| 100.3       | 2013       | china      |
+-------------+------------+------------+
```
```sql
--选出表中排除列名以t开头的其他列
SELECT `(t.*)?+.+` from sale_detail;

--返回结果
+------------+-------------+------------+------------+
| shop_name  | customer_id | sale_date  | region     |
+------------+-------------+------------+------------+
| s1         | c1          | 2013       | china      |
| s2         | c2          | 2013       | china      |
| s3         | c3          | 2013       | china      |
+------------+-------------+------------+------------+
```
* 选取列名前可以使用distinct去掉重复字段，只返回去重后的值
```sql
--查询表中region列数据，如果有重复值时仅显示一条
SELECT distinct region from sale_detail;

--返回结果
+------------+
| region     |
+------------+
| china      |
+------------+
```
```sql
--去重多列时，distinct的作用域是SELECT的列集合，而非单个列
SELECT distinct region, sale_date from sale_detail;

--返回结果
+------------+------------+
| region     | sale_date  |
+------------+------------+
| china      | 2013       |
+------------+------------+
```
```sql
--distinct可以对窗口函数的计算结果进行去重，即distinct可以配合窗口函数使用
SELECT distinct sale_date, row_number() over (partition by customer_id order by total_price) as rn from sale_detail;

--返回结果
+-----------+------------+
| sale_date | rn         |
+-----------+------------+
| 2013      | 1          |
+-----------+------------+
```
```sql
--distinct不支持与group by联合使用
--错误示例
SELECT distinct shop_name from sale_detail group by shop_name;
--报错信息： GROUP BY cannot be used with SELECT DISTINCT
```

## 排除列（except_expr）：可选
`except_expr`格式为`except(col1_name, col2_name, ...)`，表示读取表数据时会排除指定列（col1、col2）的数据。
```sql
--读取sale_detail表的数据，并排除region列的数据。
SELECT * except(region) from sale_detail;

--返回结果
+-----------+-------------+-------------+-----------+
| shop_name | customer_id | total_price | sale_date |
+-----------+-------------+-------------+-----------+
| s1        | c1          | 100.1       | 2013      |
| s2        | c2          | 100.2       | 2013      |
| s3        | c3          | 100.3       | 2013      |
+-----------+-------------+-------------+-----------+
```

## 修改列（replace_expr）：可选
`replace_expr`格式为`replace(exp1 [as] col1_name, exp2 [as] col2_name, ...)`，表示读取表数据时会将col1的数据修改为exp1，将col2的数据修改为exp2。
```sql
--读取sale_detail表的数据，并修改total_price、region两列的数据。
SELECT * replace(total_price+100 as total_price, 'shanghai' as region) from sale_detail;

--返回结果
+-----------+-------------+-------------+-----------+--------+
| shop_name | customer_id | total_price | sale_date | region |
+-----------+-------------+-------------+-----------+--------+
| s1        | c1          | 200.1       | 2013      | shanghai |
| s2        | c2          | 200.2       | 2013      | shanghai |
| s3        | c3          | 200.3       | 2013      | shanghai |
+-----------+-------------+-------------+-----------+--------+
```

## 目标表信息（table_reference）：必填
`table_reference`表示查询的目标表信息
* 直接指定目标表名
```sql
SELECT customer_id from sale_detail;

--返回结果
+-------------+
| customer_id |
+-------------+
| c1          |
| c2          |
| c3          |
+-------------+
```
* 嵌套子查询
```sql
SELECT * from (SELECT region,sale_date from sale_detail) t where region = 'china';

--返回结果
+------------+------------+
| region     | sale_date  |
+------------+------------+
| china      | 2013       |
| china      | 2013       |
| china      | 2013       |
+------------+------------+
```

## WHERE子句（where_condition）：可选
`where`子句为过滤条件，如果表是分区表，可以实现列裁剪
* 配合关系运算符，筛选满足指定条件的数据。关系运算符包括：
  - `>`、`<`、`=`、`>=`、`<=` 、`<>`
  - `like`、`rlike`
  - `in`、`not in`
  -  `between...and`
```sql
--where子句可以指定分区范围，只扫描表的指定部分
SELECT * 
from sale_detail
where sale_date >= '2008' and sale_date <= '2014';
--等价于如下语句。
SELECT * 
from sale_detail 
where sale_date between '2008' and '2014';

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s3         | c3          | 100.3       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```
* 在列表达式中，如果被重命名的列字段（赋予列别名）使用了函数，则不能在where子句中引用列别名
```sql
--错误命令示例
SELECT  task_name
        ,inst_id
        ,settings
        ,GET_JSON_OBJECT(settings, '$.SKYNET_ID') as skynet_id
        ,GET_JSON_OBJECT(settings, '$.SKYNET_NODENAME') as user_agent
from    Information_Schema.TASKS_HISTORY
where   ds = '20211215' and skynet_id is not null
limit 10;
```
## GROUP BY分组查询（col_list）：可选
`group by`和聚合函数配合使用，根据指定的普通列、分区列或正则表达式进行分组。
#### `group by` 使用规则
* `group by`取值为正则表达式，必须使用列的完整表达式
* `SELECT`语句中没有使用聚合函数的列必须出现在`GROUP BY`中
#### `group by`使用示例
```sql
--直接使用输入表列名作为group by的列
SELECT region from sale_detail group by region;

--返回结果
+------------+
| region     |
+------------+
| china      |
+------------+
```
```sql
--以某一列的值分组，返回计算结果
SELECT sum(total_price) from sale_detail group by region;

--返回结果
+------------+
| _c0        |
+------------+
| 300.6      |
+------------+
```
```sql
--同时获取列信息和计算结果
SELECT region, sum (total_price) from sale_detail group by region;

--返回结果
+------------+------------+
| region     | _c1        |
+------------+------------+
| china      | 300.6      |
+------------+------------+
```
```sql
--以`SELECT`列的别名分组
SELECT region as r from sale_detail group by r;
--等效于如下语句。
SELECT region as r from sale_detail group by region;

--返回结果
+------------+
| r          |
+------------+
| china      |
+------------+
```
```sql
--以列表达式分组
SELECT 2 + total_price as r from sale_detail group by 2 + total_price;

--返回结果
+------------+
| r          |
+------------+
| 102.1      |
| 102.2      |
| 102.3      |
+------------+
```
```sql
--SELECT的所有列中没有使用聚合函数的列，必须出现在GROUP BY中，否则会报错
--错误示例
SELECT region, total_price from sale_detail group by region;

--正确示例
SELECT region, total_price from sale_detail group by region, total_price;

--返回结果
+------------+-------------+
| region     | total_price |
+------------+-------------+
| china      | 100.1       |
| china      | 100.2       |
| china      | 100.3       |
+------------+-------------+
```

## HAVING子句（having_condition）：可选
通常`HAVING`子句与聚合函数一起使用，实现过滤
```sql
--为直观展示数据呈现效果，向sale_detail表中追加数据。
insert into sale_detail partition (sale_date='2014', region='shanghai') values ('null','c5',null),('s6','c6',100.4),('s7','c7',100.5);
--使用having子句配合聚合函数实现过滤。
SELECT region,sum(total_price) from sale_detail 
group by region 
having sum(total_price)<305;

--返回结果
+------------+------------+
| region     | _c1        |
+------------+------------+
| china      | 300.6      |
| shanghai   | 200.9      |
+------------+------------+
```

## ORDER BY全局排序（order_condition）：可选
`order by`用于对所有数据按照指定普通列、分区列或指定常量进行全局排序
#### `order by`使用规则
* 默认对数据进行升序排序，如果降序 排序，需要使用`desc`关键字
* `order by`默认要求带limit数据行数限制，没有`limit`会返回错误
#### `order by`使用示例
```sql
--查询表的信息，并按照total_price升序排列前2条
SELECT * from sale_detail order by total_price limit 2;

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```
```sql
--查询表的信息，并按照total_price降序排列前2条
SELECT * from sale_detail order by total_price desc limit 2;

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s3         | c3          | 100.3       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```

## DISTRIBUTE BY哈希分片（distribute_condition）：可选
`distribute by`用于对数据按照某几列的值做Hash分片。`distribute by`控制Map的输出在Reducer是如何划分的，如果不希望Reducer的内容存在重叠，或需要对同一分组的数据一起处理，可以使用`distribute by`来保证同组数据分发到同一个Reducer中。
```sql
--查询表sale_detail中的列region值并按照region值进行哈希分片。
SELECT region from sale_detail distribute by region;
--等价于如下语句。
SELECT region as r from sale_detail distribute by region;
SELECT region as r from sale_detail distribute by r;
```

## SORT BY局部排序（sort_condition）：可选
#### `sort by`使用规则
* `sort by`默认对数据进行升序排序，如果降序排序，需要使用`desc`关键字
* 如果`sort by`语句前有`distribute by`，`sort by`会对`distribute by`的结果按照指定的列进行排序
```sql
--查询表中的列region和total_price的值并按照region值进行哈希分片，然后按照total_price对哈希分片结果进行局部升序排序
--为直观展示数据呈现效果，向sale_detail表中追加数据。
insert into sale_detail partition (sale_date='2014', region='shanghai') values ('null','c5',null),('s6','c6',100.4),('s7','c7',100.5);
SELECT region,total_price from sale_detail distribute by region sort by total_price;

--返回结果
+------------+-------------+
| region     | total_price |
+------------+-------------+
| shanghai   | NULL        |
| china      | 100.1       |
| china      | 100.2       |
| china      | 100.3       |
| shanghai   | 100.4       |
| shanghai   | 100.5       |
+------------+-------------+
```
```sql
--查询表中的列region和total_price的值并按照region值进行哈希分片，然后按照total_price对哈希分片结果进行局部降序排序
SELECT region,total_price from sale_detail distribute by region sort by total_price desc;

--返回结果
+------------+-------------+
| region     | total_price |
+------------+-------------+
| shanghai   | 100.5       |
| shanghai   | 100.4       |
| china      | 100.3       |
| china      | 100.2       |
| china      | 100.1       |
| shanghai   | NULL        |
+------------+-------------+
```
```sql
--如果sort by语句前没有`distribute by`，`sort by`会对每个Reduce中的数据进行局部排序。保证每个Reduce的输出数据都是有序的，从而增加存储压缩率，同时读取时如果有过滤，能够减少真正从磁盘读取的数据量，提高后续全局排序的效率。
SELECT region,total_price from sale_detail sort by total_price desc;

--返回结果
+------------+-------------+
| region     | total_price |
+------------+-------------+
| china      | 100.3       |
| china      | 100.2       |
| china      | 100.1       |
| shanghai   | 100.5       |
| shanghai   | 100.4       |
| shanghai   | NULL        |
+------------+-------------+

--`order by|distribute by|sort by`的取值必须是`SELECT`语句的输出列，即列的别名。列的别名可以为中文。
--在MaxCompute SQL解析中，`order by|distribute by|sort by`执行顺序在`SELECT`操作之后，因此它们的取值只能为`SELECT`语句的输出列。
--`order by`不和`distribute by`、`sort by`同时使用，`group by`也不和`distribute by、sort by`同时使用。
```

## LIMIT限制输出行数（number）：可选
`limit <number>`中的`number`是常数，用于限制输出行数，取值范围为int32位取值范围，即最大值不可超过2,147,483,647。

# 子查询（SUBQUERY）
## 总体格式
```sql
--创建一张分区表sale_detail。
create table if not exists sale_detail
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

--向源表增加分区。
alter table sale_detail add partition (sale_date='2013', region='china') partition (sale_date='2014', region='shanghai');

--向源表追加数据。
insert into sale_detail partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s3','c3',100.3);
insert into sale_detail partition (sale_date='2014', region='shanghai') values ('null','c5',null),('s6','c6',100.4),('s7','c7',100.5);
```
```sql
select * from sale_detail; 
--返回结果。
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
```

## 基础子查询
普通查询的对象是目标表，但是查询的对象也可以是另一个`select`语句，这种查询即是子查询。在`from`子句中，子查询可以被当作一张表，与其他表或子查询进行`join`操作。
####SQL代码格式
```sql
select <select_expr> from (<select_statement>) [<sq_alias_name>];
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_expr |必填 | 格式为`col1_name, col2_name, 正则表达式, ...`，表示待查询的普通列、分区列或正则表达式|
|select_statement |必填| 子查询语句 |
|sq_alias_name |可选| 子查询的别名 |
|table_name |必填| 目标表名称 |
####SQL代码示例
```sql
--子查询语法
select * from (select shop_name from sale_detail) a;

--返回结果
+------------+
| shop_name  |
+------------+
| s1         |
| s2         |
| s3         |
| null       |
| s6         |
| s7         |
+------------+
```
```sql
--在`from`子句中，子查询可以被当作一张表，与其他的表或查询进行`join`操作
--先新建一张表，再执行join操作。
create table shop as select shop_name,customer_id,total_price from sale_detail;
select a.shop_name, a.customer_id, a.total_price from
(select * from shop) a join sale_detail on a.shop_name = sale_detail.shop_name;

--返回结果
+------------+-------------+-------------+
| shop_name  | customer_id | total_price |
+------------+-------------+-------------+
| null       | c5          | NULL        |
| s6         | c6          | 100.4       |
| s7         | c7          | 100.5       |
| s1         | c1          | 100.1       |
| s2         | c2          | 100.2       |
| s3         | c3          | 100.3       |
+------------+-------------+-------------+
```
## IN SUBQUERY
`in subquery`与left semi join用法类似
#### SQL命令格式
```sql
--格式1
select<select_expr1>from<table_name1>where<select_expr2>in(select<select_expr3>from<table_name2>);
--等效于leftsemijoin如下语句。
select<select_expr1>from<table_name1><alias_name1>leftsemijoin<table_name2><alias_name2>on<alias_name1>.<select_expr2>=<alias_name2>.<select_expr3>;

--格式二
select<select_expr1>from<table_name1>where<select_expr2>in(select<select_expr3>from<table_name2>where
<table_name1>.<col_name>=<table_name2>.<col_name>);
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_expr1 |必填 | 格式为`col1_name, col2_name, 正则表达式, ...`，表示待查询的普通列、分区列或正则表达式|
|table_name1、table_name2 |必填| 表的名称 |
|select_expr2、select_expr3 |必填| 表示table_name1和table_name2互相映射的列名 |
|col_name |必填| 表的列名 |
#### SQL注意事项
使用`IN`的子查询时，在子查询的返回结果中会自动去除NULL值的记录
#### SQL代码示例
```sql
--使用格式1子查询语法
select * from sale_detail where total_price in (select total_price from shop);

--返回结果
+-----------+-------------+-------------+-----------+--------+
| shop_name | customer_id | total_price | sale_date | region |
+-----------+-------------+-------------+-----------+--------+
| s1        | c1          | 100.1       | 2013      | china  |
| s2        | c2          | 100.2       | 2013      | china  |
| s3        | c3          | 100.3       | 2013      | china  |
| s6        | c6          | 100.4       | 2014      | shanghai |
| s7        | c7          | 100.5       | 2014      | shanghai |
+-----------+-------------+-------------+-----------+--------+
```
```sql
--使用格式2子查询语法
select * from sale_detail where total_price in (select total_price from shop where customer_id = shop.customer_id);

--返回结果
+-----------+-------------+-------------+-----------+--------+
| shop_name | customer_id | total_price | sale_date | region |
+-----------+-------------+-------------+-----------+--------+
| s1        | c1          | 100.1       | 2013      | china  |
| s2        | c2          | 100.2       | 2013      | china  |
| s3        | c3          | 100.3       | 2013      | china  |
| s6        | c6          | 100.4       | 2014      | shanghai |
| s7        | c7          | 100.5       | 2014      | shanghai |
+-----------+-------------+-------------+-----------+--------+
```
## NOT IN SUBQUERY
`not in subquery`与left anti join用法类似，但不完全相同。如果查询目标表的指定列名中有任意一行为NULL，则`not in`表达式值为NULL，导致`where`条件不成立，无数据返回，这点与left anti join不同。
```sql
--格式1
select <select_expr1> from <table_name1> where <select_expr2> not in (select <select_expr2> from <table_name2>);
--等效于left anti join如下语句。
select <select_expr1> from <table_name1> <alias_name1> left anti join <table_name2> <alias_name2> on <alias_name1>.<select_expr1> = <alias_name2>.<select_expr2>;
```
```sql
--MaxCompute不仅支持not in subquery，还支持Correlated条件。子查询中的where <table_name2_colname> = <table_name1>.<colname>即是一个Correlated条件。
select <select_expr1> from <table_name1> where <select_expr2> not in (select <select_expr2> from <table_name2> where <table_name2_colname> = <table_name1>.<colname>);
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_expr1 |必填 | 格式为`col1_name, col2_name, 正则表达式, ...`，表示待查询的普通列、分区列或正则表达式|
|table_name1、table_name2 |必填| 表的名称 |
|select_expr2、select_expr3 |必填| 表示table_name1和table_name2互相映射的列名 |
|col_name |必填| 表的列名 |
#### SQL注意事项
使用`NOT IN`的子查询时，在子查询的返回结果中会自动去除NULL值的记录
#### SQL代码示例
```sql
--使用格式1子查询语法
--创建一张新表shop1并追加数据。
create table shop1 as select shop_name,customer_id,total_price from sale_detail;
insert into shop1 values ('s8','c1',100.1);

select * from shop1 where shop_name not in (select shop_name from sale_detail);

--返回结果
+------------+-------------+-------------+
| shop_name  | customer_id | total_price |
+------------+-------------+-------------+
| s8         | c1          | 100.1       |
+------------+-------------+-------------+
```
```sql
--使用格式2子查询语法
select * from shop1 where shop_name not in (select shop_name from sale_detail where customer_id = shop1.customer_id);

--返回结果
+------------+-------------+-------------+
| shop_name  | customer_id | total_price |
+------------+-------------+-------------+
| s8         | c1          | 100.1       |
+------------+-------------+-------------+
```
```sql
--not in subquery不作为join条件（where 中包含 and， 所以无法转换为anti join）
select * from shop1 where shop_name not in (select shop_name from sale_detail) and total_price < 100.3;

--返回结果
+------------+-------------+-------------+
| shop_name  | customer_id | total_price |
+------------+-------------+-------------+
| s8         | c1          | 100.1       |
+------------+-------------+-------------+
```
```sql
--假设查询表中有任意一行为NULL，则无数据返回
--创建一张新表sale并追加数据。
create table if not exists sale
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);
alter table sale add partition (sale_date='2013', region='china');
insert into sale partition (sale_date='2013', region='china') values ('null','null',null),('s2','c2',100.2),('s3','c3',100.3),('s8','c8',100.8);

set odps.sql.allow.fullscan=true;
select * from sale where shop_name not in (select shop_name from sale_detail);

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
+------------+-------------+-------------+------------+------------+
```

## EXISTS SUBQUERY
使用`exists subquery`时，当子查询中有至少一行数据时，返回True，否则返回False
MaxCompute只支持含有Correlated条件的where子查询。exists subquery实现的方式是转换为left semi join。
#### SQL代码格式
```sql
select <select_expr> from <table_name1> where exists (select <select_expr> from <table_name2> where <table_name2_colname> = <table_name1>.<colname>);
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_expr |必填 | 格式为`col1_name, col2_name, 正则表达式, ...`，表示待查询的普通列、分区列或正则表达式|
|table_name1、table_name2 |必填| 表的名称 |
|col_name |必填| 表的列名 |
#### SQL注意事项
使用`EXISTS`的子查询时，在子查询的返回结果会自动去除NULL值的记录
#### SQL代码示例
```sql
select * from sale_detail where exists (select * from shop where customer_id = sale_detail.customer_id);
--等效于以下语句。
select * from sale_detail a left semi join shop b on a.customer_id = b.customer_id;

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| null       | c5          | NULL        | 2014       | shanghai   |
| s6         | c6          | 100.4       | 2014       | shanghai   |
| s7         | c7          | 100.5       | 2014       | shanghai   |
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s3         | c3          | 100.3       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```

## NOT EXISTS SUBQUERY
使用`not exists subquery`时，当子查询中无数据时，返回True，否则返回False
MaxCompute只支持含有Correlated条件的`where`子查询。`not exists subquery`实现的方式是转换为`left anti join`。
#### SQL代码格式
```sql
select <select_expr> from <table_name1> where not exists (select <select_expr> from <table_name2> where <table_name2_colname> = <table_name1>.<colname>);
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|select_expr |必填 | 格式为`col1_name, col2_name, 正则表达式, ...`，表示待查询的普通列、分区列或正则表达式|
|table_name1、table_name2 |必填| 表的名称 |
|col_name |必填| 表的列名 |
#### SQL注意事项
使用`NOT EXISTS`的子查询时，在子查询的返回结果中会自动去除NULL值的记录。
#### SQL代码示例
```sql
select * from sale_detail where not exists (select * from shop where shop_name = sale_detail.shop_name);
--等效于以下语句。
select * from sale_detail a left anti join shop b on a.shop_name = b.shop_name;

--返回结果
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
+------------+-------------+-------------+------------+------------+
```