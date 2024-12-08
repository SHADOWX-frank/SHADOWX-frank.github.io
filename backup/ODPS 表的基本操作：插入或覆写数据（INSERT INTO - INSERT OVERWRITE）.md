# 插入数据&覆写数据
#### SQL代码格式
```sql
insert {into|overwrite} table <table_name> [partition (<pt_spec>)] [(<col_name> [,<col_name> ...)]]
<select_statement>
from <from_statement>
[zorder by <zcol_name> [, <zcol_name> ...]];
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |需要插入数据的目标表名称|
|pt_spec|可选| 需要插入数据的分区信息，不允许使用函数等表达式，只能是常量|
|col_name |可选| 需要插入数据的目标表的列名称 `insert overwrite`不支持指定`[(<col_name> [,<col_name> ...)]`|
|select_statement|必填| `select`子句，从源表中查询需要插入目标表的数据|
|from_statement|必填| `from`子句，从源表中查询需要插入目标表的数据|

|进阶参数| 是否必填 |说明|
|-----|:----:|----------|
|zorder by <zcol_name> [, <zcol_name> ...]|可选|向表或分区写入数据时，支持根据指定的一列或多列（select_statement对应表中的列），把排序列数据相近的行排列在一起，提升查询时的过滤性能，在一定程度上降低存储成本。需要注意的是，order by x, y会严格地按照先x后y的顺序对数据进行排序，zorder by x, y会把相近的<x, y>尽量排列在一起|

#### SQL代码示例
* 执行`insert into`命令向非分区表中追加数据
```sql
--创建一张非分区表websites。
create table if not exists websites
(id int,
 name string,
 url string
);
--创建一张非分区表apps
create table if not exists apps
(id int,
 app_name string,
 url string
);
--向表apps追加数据。其中：insert into table table_name可以简写为insert into table_name
insert into apps (id,app_name,url) values 
(1,'Aliyun','https://www.aliyun.com');
--复制apps的表数据追加至websites表
insert into websites (id,name,url) select id,app_name,url
from  apps;
--执行select语句查看表websites中的数据。
select * from websites;
--返回结果。
+------------+------------+------------+
| id         | name       | url        |
+------------+------------+------------+
| 1          | Aliyun     | https://www.aliyun.com |
+------------+------------+------------+
```
* 执行`insert into`命令向分区表中追加数据
```sql
--创建一张分区表sale_detail。
create table if not exists sale_detail
(
shop_name     string,
customer_id   string,
total_price   double
)
partitioned by (sale_date string, region string);

--向源表增加分区。非必需操作，如果不提前创建，写入时会自动创建该分区。
alter table sale_detail add partition (sale_date='2013', region='china');

--向源表追加数据。其中：insert into table table_name可以简写为insert into table_name，但insert overwrite table table_name不可以省略table关键字。
insert into sale_detail partition (sale_date='2013', region='china') values ('s1','c1',100.1),('s2','c2',100.2),('s3','c3',100.3);

--开启全表扫描，仅此Session有效。执行select语句查看表sale_detail中的数据。
set odps.sql.allow.fullscan=true; 
select * from sale_detail;

--返回结果。
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s3         | c3          | 100.3       | 2013       | china      |
+------------+-------------+-------------+------------+------------+
```
* 执行`insert overwrite`命令覆写数据
```sql
--创建目标表sale_detail_insert，与sale_detail有相同的结构。
create table sale_detail_insert like sale_detail;

--给目标表增加分区。非必需操作，如果不提前创建，写入时会自动创建该分区。
alter table sale_detail_insert add partition (sale_date='2013', region='china');

--从源表sale_detail中取出数据插入目标表sale_detail_insert。注意不需要声明目标表字段，也不支持重排目标表字段顺序。
--对于静态分区目标表，分区字段赋值已经在partition()部分声明，不需要在select_statement中包含，只要按照目标表普通列顺序查出对应字段，按顺序映射到目标表即可。动态分区表则需要在select中包含分区字段，详情请参见插入或覆写动态分区数据（DYNAMIC PARTITION）。
insert overwrite table sale_detail_insert partition (sale_date='2013', region='china')
  select 
  shop_name, 
  customer_id,
  total_price 
  from sale_detail
  zorder by customer_id, total_price;

--开启全表扫描，仅此Session有效。执行select语句查看表sale_detail_insert中的数据。
set odps.sql.allow.fullscan=true;
select * from sale_detail_insert;

--返回结果。
+------------+-------------+-------------+------------+------------+
| shop_name  | customer_id | total_price | sale_date  | region     |
+------------+-------------+-------------+------------+------------+
| s1         | c1          | 100.1       | 2013       | china      |
| s2         | c2          | 100.2       | 2013       | china      |
| s3         | c3          | 100.3       | 2013       | china      |
```
* 执行`insert overwrite`命令覆写数据，调整`select`子句中列的顺序。（注意：源表与目标表的对应关系依赖于select子句中列的顺序，而不是表与表之间列名的对应关系）
```sql
insert overwrite table sale_detail_insert partition (sale_date='2013', region='china')
    select customer_id, shop_name, total_price from sale_detail;    

set odps.sql.allow.fullscan=true;
select * from sale_detail_insert;
```
#### 错误SQL示例
* 向某个分区插入数据时，分区列不能出现在select子句
```sql
insert overwrite table sale_detail_insert partition (sale_date='2013', region='china')
   select shop_name, customer_id, total_price, sale_date, region from sale_detail;
```
* `partition`的值只能是常量，不可以为表达式
```sql
insert overwrite table sale_detail_insert partition (sale_date=datepart('2016-09-18 01:10:00', 'yyyy') , region='china')
   select shop_name, customer_id, total_price from sale_detail;
```

