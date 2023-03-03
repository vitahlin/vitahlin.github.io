---
title: MySQL索引优化规则
<!-- description: 这是一个副标题 -->
date: 2022-04-10
slug: 01GTJYEDXV0QJBXKPMWWKQMP66
categories:
    - MySQL

tags:
    - MySQL
---

## 示例表

```sql
DROP TABLE IF EXISTS employees;
CREATE TABLE `employees`
(
    `id`        int(11)     NOT NULL AUTO_INCREMENT,
    `name`      varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
    `age`       int(11)     NOT NULL DEFAULT '0' COMMENT '年龄',
    `position`  varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
    `hire_time` timestamp   NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
    PRIMARY KEY (`id`),
    KEY `idx_name_age_position` (`name`, `age`, `position`) USING BTREE
) ENGINE = InnoDB
  AUTO_INCREMENT = 4
  DEFAULT CHARSET = utf8 COMMENT ='员工记录表';

INSERT INTO employees(name, age, position, hire_time)
VALUES ('LiLei', 22, 'manager', NOW());
INSERT INTO employees(name, age, position, hire_time)
VALUES ('HanMeimei', 23, 'dev', NOW());
INSERT INTO employees(name, age, position, hire_time)
VALUES ('Lucy', 23, 'dev', NOW());
```

## 索引优化规则

### 1. 最左匹配原则

如果索引了多列，要遵守最左匹配原则。查询从索引最左前列开始并且不跳过索引中间的列。

### 2. 不在索引列上做操作

不在索引列上做任何操作(计算、函数、类型转换(自动/手动))，会导致索引失效而转向全表扫描。

```sql
EXPLAIN SELECT * FROM employees WHERE left(name, 3) = 'LiLei';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 99909,
    "filtered": 100,
    "Extra": "Using where"
  }
]
```

### 3. 存储引擎不能使用索引中范围条件右边的列

#### 走索引示例

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
  AND age = 22
  AND position = 'manager';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": "const,const,const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

#### 不走索引示例

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
  AND age > 22
  AND position = 'manager';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "78",
    "ref": null,
    "rows": 1,
    "filtered": 10,
    "Extra": "Using index condition"
  }
]
```

### 4. 尽量使用覆盖索引，减少 select *

#### 覆盖索引示例

```sql
EXPLAIN
SELECT name, age
FROM employees
WHERE name = 'LiLei'
  AND age = 23
  AND position = 'manager';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": "const,const,const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

#### 非覆盖索引select *

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
  AND age = 23
  AND position = 'manager';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": "const,const,const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

### 5. 使用!=, <>, not in, not exist的时候不会用到索引

< 小于、 > 大于、 <=、>= 这些，mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name != 'LiLei';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ALL",
    "possible_keys": "idx_name_age_position",
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 99909,
    "filtered": 50,
    "Extra": "Using where"
  }
]
```

### 6. is null, is not null一般情况下也无法用到索引

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name IS NULL
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": null,
    "partitions": null,
    "type": null,
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": null,
    "filtered": null,
    "Extra": "Impossible WHERE"
  }
]
```

### 7. like以通配符开头('$abc...')索引会失效

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name LIKE '%Lei'
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 99909,
    "filtered": 11.11,
    "Extra": "Using where"
  }
]
```

#### 如何解决 `like '%abc%'`索引失效问题？

##### 覆盖索引

使用覆盖索引，查询字段必须是建立覆盖索引字段，例如

```sql
EXPLAIN
SELECT name, age, position
FROM employees
WHERE name LIKE '%lei%';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "index",
    "possible_keys": null,
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": null,
    "rows": 99909,
    "filtered": 11.11,
    "Extra": "Using where; Using index"
  }
]
```

##### 搜索引擎

如果不能使用覆盖索引，则可借助搜索引擎，如ES。

### 8. 字符串不加单引号会导致索引失效

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 1000;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ALL",
    "possible_keys": "idx_name_age_position",
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 99909,
    "filtered": 10,
    "Extra": "Using where"
  }
]
```

这其实是类型转换的问题

### 9. 少用or或者in

少用or或in，用它查询时，mysql**不一定使用索引**，mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引。例如：

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
   OR name = 'HanMeimei';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ALL",
    "possible_keys": "idx_name_age_position",
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 66.67,
    "Extra": "Using where"
  }
]
```

### 10. 范围查询

#### 是否会走索引？

先给年龄字段添加单值索引

```sql
ALTER TABLE `employees`
    ADD INDEX `idx_age` (`age`) USING BTREE;
```

```sql
EXPLAIN
SELECT *
FROM employees
WHERE age >= 1
  AND age <= 2000;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 99909,
    "filtered": 11.11,
    "Extra": "Using where"
  }
]
```

**没走索引原因：mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引。比如这个例子，可能是由于单次数据量查询过大导致优化器最终选择不走索引。**

#### 如何优化范围查询

优化方法：将大的范围拆分成多个小范围。

```sql
EXPLAIN
SELECT *
FROM employees
WHERE age >= 1001
  AND age <= 2000;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_age",
    "key": "idx_age",
    "key_len": "4",
    "ref": null,
    "rows": 1000,
    "filtered": 100,
    "Extra": "Using index condition"
  }
]
```

### 11. 不要在小基数字段上建立索引

**索引基数是指这个字段在表里总共有多少个不同的值**，比如性别字段，值不是男就是女，该字段基数就是2。
如果对这种小基数字段建立索引的话，还不如全表扫描来，因为你的索引树里面就包含男和女两种值，根本没办法快速的二分查找，那用索引就没有意义了。
一般建立索引，尽量使用那些基数比较大的字段，就是值比较多的字段，才能发挥出B+树快速二分查找的优势。

### 12. 长字符串可以采用前缀索引

尽量对字段类型比较小的列设计索引，比如说 tinyint之类的，因为字段类型比较小的话，占用磁盘空间也会比较小。那么当我们需要对 varhcar(255) 这种字段建立索引怎么办？
我们可以针对这个字段对前20个字符建立索引，就是说把这个字段里对每个值对前20个字符放在索引树里，如 index(name(20), age, position)。
此时在where条件搜索的时候，如果是根据name字段来搜索，就会先到索引树里根据name字段的前20个字符去搜索，定位到前20个字符的前缀匹配的部分数据之后，再回到聚簇索引提前出来完整的name字段值进行比对。
但是如果要是order by name，那么此时name在索引树里面仅仅包含前20个字符，所以这个排序是没法用上索引的，group by也是同理。

### 13. where与order by冲突时优先where

where与order by出现索引设计冲突时，到底是根据where去设计，还是针对order by去设计索引？
一般都是让where条件去使用索引来快速筛选一部分指定顺序，接着再进行排序。
大多数情况基于索引进行where筛选可以更快速度筛选你要的少部分数据，然后做排序的成本会小很多。

### 14. 基于慢查询sql做优化

什么是慢查询？
> MySQL的慢查询，全名是慢查询日志，是MySQL提供的一种日志记录，用来记录在MySQL中响应时间超过阀值的语句。
> 具体环境中，运行时间超过long_query_time值的SQL语句，则会被记录到慢查询日志中。
> long_query_time的默认值为10，意思是记录运行10秒以上的语句。
> 默认情况下，MySQL数据库并不启动慢查询日志，需要手动来设置这个参数。
> 当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。

## 索引使用总结

假设索引 `index(a,b,c)`

| where 语句 | 索引是否被使用 |
| --- | --- |
| where a=3 | 有，使用到了a |
| where a=3 and b=5 | 有，使用到了a, b |
| where a=3 and b=4 and c=5 | 有，使用到了a, b, c |
| where b=3 | 没有 |
| where b=3 and c=4 | 没有 |
| where c=4 | 没有 |
| where a=3 and c=5 | 有，但只使用了a |
| where a=3 and b>4 and c=5 | 有，使用了a,b，c不能用在范围之后，b之后断了 |
| where a=3 and b like 'kk%' and c=4 | 有，使用到了a,b,c |
| where a=3 and b like '%kk' and c=4 | 有，只用到了a |
| where a=3 and b like '%kk%' and c=4 | 有，只用到了a |
| where a=3 and b like 'k%kk%' and c=4 | 有，使用到了a,b,c |

`like 'kk%'`相当于等于常量，`%kk` 和 `%kk%` 相当于范围。
