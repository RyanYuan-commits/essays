从字段特性的角度来看，索引分为主键索引、唯一索引、普通索引、前缀索引。

# 主键索引
主键索引就是建立在主键字段上的索引，通常在创建表的时候一起创建，一张表最多只有一个主键索引，索引列的值不允许有空值。
在创建表时，创建主键索引的方式如下：
```sql
CREATE TABLE table_name  (
  ....
  PRIMARY KEY (index_column_1) USING BTREE
);
```
# 唯一索引
唯一索引建立在 UNIQUE 字段上的索引，一张表可以有多个唯一索引，索引列的值必须唯一，但是允许有空值。
在创建表时，创建唯一索引的方式如下：
```sql
CREATE TABLE table_name  (
  ....
  UNIQUE KEY(index_column_1,index_column_2,...) 
);
```
建表后，如果要创建唯一索引，可以使用这面这条命令：
```sql
CREATE UNIQUE INDEX index_name
ON table_name(index_column_1,index_column_2,...); 
```
# 普通索引
普通索引就是建立在普通字段上的索引，既不要求字段为主键，也不要求字段为 UNIQUE。
在创建表时，创建普通索引的方式如下：
```sql
CREATE TABLE table_name  (
  ....
  INDEX(index_column_1,index_column_2,...) 
);
```
建表后，如果要创建普通索引，可以使用这面这条命令：
```sql
CREATE INDEX index_name
ON table_name(index_column_1,index_column_2,...); 
```
# 前缀索引
前缀索引是指对字符类型字段的前几个字符建立的索引，而不是在整个字段上建立的索引，前缀索引可以建立在字段类型为 char、 varchar、binary、varbinary 的列上。
使用前缀索引的目的是为了减少索引占用的存储空间，提升查询效率。
在创建表时，创建前缀索引的方式如下：
```sql
CREATE TABLE table_name(
    column_list,
    INDEX(column_name(length))
); 
```
建表后，如果要创建前缀索引，可以使用这面这条命令：
```sql
CREATE INDEX index_name
ON table_name(column_name(length)); 
```