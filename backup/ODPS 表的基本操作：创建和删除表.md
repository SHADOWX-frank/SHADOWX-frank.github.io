# 创建表
## 直接创建新表
#### SQL代码格式
```sql
CREATE TABLE [IF NOT EXISTS] <table_name> [
  (<col_name <data_type> [NOT NULL] [DEFAULT <default_value>] [comment <col_comment>], ...)
]
[comment <table_comment>]
[PATITIONED BY (<col_name> <data_type> [comment <col_comment>], ...)]
[LIFECYCLE <days>];
```
|通用参数   |是否必填   |说明   |   
|------|:--:|:-----|
| IF NOT EXISTS  |可选   |用于确认库中是否已存在创建表名   |   
|table_name| 必填| 表名|
|col_name| 可选| 表的列名|
|col_comment| 可选| 列名的注释|
|data_type| 可选| 列的数据类型（包含BIGINT、DOUBLE、BOOLEAN、DATETIME、DECIMAL和STRING等多种数据类型）|
|NOT NULL| 可选| 用于禁止该列的值为空|
|default_value| 可选| 列的默认值|
|table_comment| 可选| 表注释|
|LIFECYCLE|可选|表的生命周期| 

|分区表参数 |是否必填 |说明|
|-----|:-----:|:------|
|PARTITIONED BY|可选|指定分区表的分区字段|
|col_name|可选|表的分区列名|
|data_type|可选|分区列的数据类型|
|col_comment|可选|分区列的注释内容|

#### SQL代码示例
* 创建新非分区表
```sql
CREATE TABLE test1 (key STRING);
```
* 创建新分区表
```sql
CREATE TABLE IF NOT EXISTS sale_detail(
 shop_name     STRING,
 customer_id   STRING,
 total_price   DOUBLE)
PARTITIONED BY (sale_date STRING, region STRING);
```
* （进阶扩展）创建新非分区表，并为字段指定默认值
```sql
CREATE TABLE test_default( 
tinyint_name tinyint NOT NULL default 1Y,
smallint_name SMALLINT NOT NULL DEFAULT 1S,
int_name INT NOT NULL DEFAULT 1,
bigint_name BIGINT NOT NULL DEFAULT 1,
binary_name BINARY ,
float_name FLOAT ,
double_name DOUBLE NOT NULL DEFAULT 0.1,
decimal_name DECIMAL(2, 1) NOT NULL DEFAULT 0.0BD,
varchar_name VARCHAR(10) ,
char_name CHAR(2) ,
string_name STRING NOT NULL DEFAULT 'N',
boolean_name BOOLEAN NOT NULL DEFAULT TRUE
);
```

## 基于已有数据表新建表
#### SQL代码格式
* 通过`CREATE TABLE [IF NOT EXISTS] <table_name> [LIFECYCLE <days>] AS <select_statement>;` 语句可以创建一个新表，同时将数据复制到新表之中。
* 通过`CREATE TABLE [IF NOT EXISTS] <table_name> [LIFECYCLE <days>] LIKE <existing_table_name>;` 语句可以新建一个表，该表和源表具有相同的表结构
#### SQL代码示例
* 创建新表，将已有表数据复制到新表，并设置生命周期
```sql
CREATE TABLE sale_detail_ctas1 LIFECYCLE 10 AS 
SELECT * FROM sale_detail;
```
* 创建新表，在SELECT语句中使用常量作为列的值
  - 指定列的名字
    ```sql
    CREATE TABLE sale_detail_ctas2
    AS
    SELECT shop_name, customer_id, total_price, '2013' AS sale_date, 'China' AS region
    FROM sale_detail;  
    ```
  - 不指定列的名字
    ```sql
    CREATE TABLE sale_detail_ctas3
    AS
    SELECT shop_name, customer_id, total_price, '2013', 'China'
    FROM sale_detail;
    ```
* 创建新表，与已有表的结构相同，并设置生命周期
```sql
CREATE TABLE sale_detail_like LIKE sale_detail LIFECYCLE 10;
```

# 删除表
#### SQL代码格式
```sql
DROP TABLE [IF EXISTS] <table_name>; 
```
|参数   |是否必填   |说明   |   
|------|:--:|:-----|
| IF EXISTS  |可选   |如果不指定IF EXISTS且表不存在，则返回异常。如果指定IF EXISTS，无论表是否存在，均返回成功。   |   
|table_name| 必填| 待删除的表名|

#### SQL代码示例
* 删除表，无论表是否存在，均返回成功
```sql
DROP TABLE IF EXISTS sale_detail; 
```




