# 分区操作
## 添加分区
#### SQL代码格式
```sql
alter table <table_name> add [if not exists] partition <pt_spec> [partition <pt_spec> partition <pt_spec>...];
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |带新增分区的分区表名
|if not exists |可选| 如果未指定if not exists而同名的分区已存在，会执行失败并返回报错|
|pt_spec| 必填 | 新增的分区，格式为(partition_col1 = partition_col_value1, partition_col2 = partition_col_value2, ...)。**partition_col**是分区字段，**partition_col_value**是分区值。分区字段不区分大小写，分区值区分大小写|
#### SQL代码示例
* 给表添加一个新的分区
```sql
alter table sale_detail add if not exists partition (sale_date='201312', region='hangzhou');
```
## 删除分区
### 不指定筛选条件
#### SQL代码格式
```sql
--一次删除一个分区。
alter table <table_name> drop [if exists] partition <pt_spec>;
--一次删除多个分区。
alter table <table_name> drop [if exists] partition <pt_spec>,partition <pt_spec>[,partition <pt_spec>....];
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |待删除分区的分区表名|
|if exists |可选| 如果未指定if exists且分区不存在，则返回报错|
|pt_spec| 必填 | 删除的分区，格式为(partition_col1 = partition_col_value1, partition_col2 = partition_col_value2, ...)。**partition_col**是分区字段，**partition_col_value**是分区值。分区字段不区分大小写，分区值区分大小写|
#### SQL代码示例
```sql
--从表sale_detail中删除一个分区，2013年12月杭州分区的销售记录。
alter table sale_detail drop if exists partition(sale_date='201312',region='hangzhou'); 
--从表sale_detail中同时删除两个分区，2013年12月杭州和上海分区的销售记录。
alter table sale_detail drop if exists partition(sale_date='201312',region='hangzhou'),partition(sale_date='201312',region='shanghai');
```

### 指定筛选条件
#### SQL代码格式
```sql
alter table <table_name> drop [if exists] partition <partition_filtercondition>;
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |待删除分区的分区表名|
|if exists |可选| 如果未指定if exists且分区不存在，则返回报错|
|pt_spec| 必填 | 删除的分区，格式为(partition_col1 = partition_col_value1, partition_col2 = partition_col_value2, ...)。**partition_col**是分区字段，**partition_col_value**是分区值。分区字段不区分大小写，分区值区分大小写|
|partition_filtercondition |必填 | 分区筛选条件，不区分大小写|

#### partition_filtercondition格式
```sql
partition_filtercondition
    : partition (<partition_col> <relational_operators> <partition_col_value>)
    | partition (<partition_filtercondition1> AND|OR <partition_filtercondition2>)
    | partition (NOT <partition_filtercondition>)
    | partition (<partition_filtercondition1>)[,partition (<partition_filtercondition2>), ...]
```
|参数| 说明|
|-----|----------|
|partition_col| 分区名称|
|relational_operators| 关系运算符|
|partition_col_value| 分区列比较值或正则表达式，与分区列数据类型保持一致|

#### SQL代码示例
```sql
--创建分区表。
create table if not exists sale_detail(
shop_name     STRING,
customer_id   STRING,
total_price   DOUBLE)
partitioned by (sale_date STRING);
--添加分区。
alter table sale_detail add if not exists
partition (sale_date= '201910')
partition (sale_date= '201911')
partition (sale_date= '201912')
partition (sale_date= '202001')
partition (sale_date= '202002')
partition (sale_date= '202003')
partition (sale_date= '202004')
partition (sale_date= '202005')
partition (sale_date= '202006')
partition (sale_date= '202007');
--批量删除分区。
alter table sale_detail drop if exists partition(sale_date < '201911');
alter table sale_detail drop if exists partition(sale_date >= '202007');
alter table sale_detail drop if exists partition(sale_date LIKE '20191%');
alter table sale_detail drop if exists partition(sale_date IN ('202002','202004','202006'));
alter table sale_detail drop if exists partition(sale_date BETWEEN '202001' AND '202007');
alter table sale_detail drop if exists partition(substr(sale_date, 1, 4) = '2020');
alter table sale_detail drop if exists partition(sale_date < '201912' OR sale_date >= '202006');
alter table sale_detail drop if exists partition(sale_date > '201912' AND sale_date <= '202004');
alter table sale_detail drop if exists partition(NOT sale_date > '202004');
--支持多个分区过滤表达式，表达式之间是OR的关系。
alter table sale_detail drop if exists partition(sale_date < '201911'), partition(sale_date >= '202007');
--添加其他格式分区。
alter table sale_detail add IF NOT EXISTS
partition (sale_date= '2019-10-05') 
partition (sale_date= '2019-10-06') 
partition (sale_date= '2019-10-07');
--批量删除分区，使用正则表达式匹配分区。
alter table sale_detail drop if exists partition(sale_date RLIKE '2019-\\d+-\\d+');
--创建多级分区表。
create table if not exists region_sale_detail(
shop_name     STRING,
customer_id   STRING,
total_price   DOUBLE)
partitioned by (sale_date STRING , region STRING );
--添加分区。
alter table region_sale_detail add IF NOT EXISTS
partition (sale_date= '201910',region = 'shanghai')
partition (sale_date= '201911',region = 'shanghai')
partition (sale_date= '201912',region = 'shanghai')
partition (sale_date= '202001',region = 'shanghai')
partition (sale_date= '202002',region = 'shanghai')
partition (sale_date= '201910',region = 'beijing')
partition (sale_date= '201911',region = 'beijing')
partition (sale_date= '201912',region = 'beijing')
partition (sale_date= '202001',region = 'beijing')
partition (sale_date= '202002',region = 'beijing');
--执行如下语句批量删除多级分区，两个匹配条件是或的关系，会将sale_date小于201911或region等于beijing的分区都删除掉。
alter table region_sale_detail drop if exists partition(sale_date < '201911'),partition(region = 'beijing');
--如果删除sale_date小于201911且region等于beijing的分区，可以使用如下方法。
alter table region_sale_detail drop if exists partition(sale_date < '201911', region = 'beijing');
```
#### (进阶补充) 错误操作示例
* 批量删除多级分区时，在一个partition过滤子句中，不能根据多个分区列编写组合条件匹配分区
```sql
--分区过滤子句只能访问一个分区列，如下语句报错。
alter table region_sale_detail drop if exists partition(sale_date < '201911' AND region = 'beijing');
```

## 合并分区
#### SQL代码格式
```sql
alter table <table_name> merge [if exists] partition (<predicate>) [, partition(<predicate2>) ...] overwrite partition (<fullpartitionSpec>) ;
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |待合并分区的分区表名称|
|if exists |可选| 如果未指定if exists，且分区不存在，会执行失败并返回报错。如果指定if exists后不存在满足merge条件的分区，则不生成新分区。如果运行过程中出现源数据被并发修改（包括insert、rename或drop）时，即使指定if exists也会报错|
|predicate|必填|筛选待合并分区需要满足的条件 | 
|fullpartitionSpec|必填| 目标分区信息|

#### SQL代码示例
* 合并满足指定条件的分区到目标分区
```sql
--查看分区表的分区。
show partitions intpstringstringstring;

ds=20181101/hh=00/mm=00
ds=20181101/hh=00/mm=10
ds=20181101/hh=10/mm=00
ds=20181101/hh=10/mm=10

--合并所有满足hh='00'的分区到hh='00'，mm='00'中。
alter table intpstringstringstring merge partition(hh='00') overwrite partition(ds='20181101', hh='00', mm='00');

--查看合并后的分区。
show partitions intpstringstringstring;

ds=20181101/hh=00/mm=00
ds=20181101/hh=10/mm=00
ds=20181101/hh=10/mm=10 
```
* 合并指定的多个分区到目标分区
```sql
--合并多个指定分区。
alter table intpstringstringstring merge if exists partition(ds='20181101', hh='00', mm='00'), partition(ds='20181101', hh='10', mm='00'),  partition(ds='20181101', hh='10', mm='10') overwrite partition(ds='20181101', hh='00', mm='00') purge;
--查看分区表的分区。
show partitions intpstringstringstring;

ds=20181101/hh=00/mm=00
```

# 列操作
## 添加列
#### SQL代码格式
```sql
ALTER TABLE <table_name> 
      ADD columns [if not exists]
          (<col_name1> <type1> comment ['<col_comment>']
           [, <col_name2> <type2> comment '<col_comment>'...]
          );
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |待新增列的表名称。添加的新列不支持指定顺序，默认在最后一列|
|col_name |必填| 新增列的名称|
|type|必填|新增列的数据类型 | 
|col_comment|可选| 新增列的注释|
#### SQL代码示例
* 给表添加两个列
```sql
ALTER TABLE sale_detail ADD columns if not exists(customer_name STRING, education BIGINT);
```
* 给表添加两个列同时添加列注释
```sql
ALTER TABLE sale_detail ADD columns (customer_name STRING comment '客户', education BIGINT comment '教育' );
```
## 删除列
#### SQL代码格式
```sql
alter table <table_name> drop columns <col_name1>[, <col_name2>...];
```
|参数| 是否必填 |说明|
|-----|:----:|----------|
|table_name |必填 |待删除列的表名称|
|col_name |必填| 待删除的列名称|
#### SQL代码示例
```sql
--删除表sale_detail的列customer_id。输入yes确认后，即可删除列。
alter table sale_detail drop columns customer_id;

--删除表sale_detail的列shop_name和customer_id。输入yes确认后，即可删除列。
alter table sale_detail drop columns shop_name, customer_id;
```


