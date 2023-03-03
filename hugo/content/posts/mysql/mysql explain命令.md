---
title: Mysql explain命令介绍
<!-- description: 这是一个副标题 -->
date: 2022-04-01
slug: 01GTJVQBR1YCNSGZP4QFJAXDKV
categories:
    - mysql

tags:
    - mysql
---


## 示例表和数据SQL

```sql
DROP TABLE IF EXISTS `actor`;
CREATE TABLE `actor`
(
    `id`          int(11) NOT NULL,
    `name`        varchar(45) DEFAULT NULL,
    `update_time` datetime    DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

INSERT INTO `actor` (`id`, `name`, `update_time`)
VALUES (1, 'a', '2017-12-29 10:27:18'),
       (2, 'b', '2017-12-29 15:27:18'),
       (3, 'c', '2017-12-29 15:27:18');

DROP TABLE IF EXISTS `film`;
CREATE TABLE `film`
(
    `id`   int(11) NOT NULL AUTO_INCREMENT,
    `name` varchar(10) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_name` (`name`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

INSERT INTO `film` (`id`, `name`)
VALUES (3, 'film0'),
       (1, 'film1'),
       (2, 'film2');

DROP TABLE IF EXISTS `film_actor`;
CREATE TABLE `film_actor`
(
    `id`       int(11) NOT NULL,
    `film_id`  int(11) NOT NULL,
    `actor_id` int(11) NOT NULL,
    `remark`   varchar(255) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_film_actor_id` (`film_id`, `actor_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

INSERT INTO `film_actor` (`id`, `film_id`, `actor_id`)
VALUES (1, 1, 1),
       (2, 1, 2),
       (3, 2, 1);
```

## id列

```sql
explain select * from film where id = 1;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "const",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

其中，`id` 的编号就是 `select` 的序列号，有几个 `select` 就有几个 `id`，并且 id 的顺序是按 `select` 出现的顺序增长的，*`id` 越大执行优先级越高，`id` 相同从上往下执行，`id` 为 `NULL` 最后执行*。

## select_type列

`select_type` 表示对应行是简单还是复杂的查询。

### simple

简单查询，查询不包含子查询和 `union`

```sql
EXPLAIN
SELECT *
FROM film
WHERE id = 1;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "const",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

### primary, subquery, derived

- `primary`，复杂查询中最外层的 `select`
- `subquery`，包含在 `select` 中的子查询(不在 `from` 子句中)
- `derived`，包含在 `from` 子句中的子查询。Mysql 会将结果存放在一个临时表中，也称为派生表。

用下面这个例子来了解 `primary`、`subquery` 和 `derived` 类型

```sql
# 关闭mysql5.7新特性对衍生表的合并优化
SET SESSION optimizer_switch = 'derived_merge=off';
EXPLAIN
SELECT (SELECT 1 FROM actor WHERE id = 1)
FROM (SELECT * FROM film WHERE id = 1) der;
```

```json
[
  {
    "id": 1,
    "select_type": "PRIMARY",
    "table": "<derived3>",
    "partitions": null,
    "type": "system",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 1,
    "filtered": 100,
    "Extra": null
  },
  {
    "id": 3,
    "select_type": "DERIVED",
    "table": "film",
    "partitions": null,
    "type": "const",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  },
  {
    "id": 2,
    "select_type": "SUBQUERY",
    "table": "actor",
    "partitions": null,
    "type": "const",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index"
  }

```

其中，可以看到 id 最大值为3，表示有3个 `select` 语句，`id=3` 的查询对应的子查询是 `(select * from film where id = 1)` 部分，`id=2` 对应的是 `(select 1 from actor where id = 1)` 语句，`id=1` 则是最外层的`select`。

### union

在 `union` 中的第二个和随后的 `select`。

```sql
EXPLAIN
SELECT 1
UNION ALL
SELECT 1;
```

```json
[
  {
    "id": 1,
    "select_type": "PRIMARY",
    "table": null,
    "partitions": null,
    "type": null,
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": null,
    "filtered": null,
    "Extra": "No tables used"
  },
  {
    "id": 2,
    "select_type": "UNION",
    "table": null,
    "partitions": null,
    "type": null,
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": null,
    "filtered": null,
    "Extra": "No tables used"
  }
]
```

## table列

这一列表示 `explain` 的一行正在访问哪个表。它可以是下面值中的一个

- `<union M,N>` 这一行引用了ID值为M和N的表的联合。M 和 N 表示 `union` 的 `select` 行的 `id`。
- `<deriven N>` 这一行引用了ID值为N的表所派生的表。派生的表可能是一个结果集，比如，FROM 子句中的子查询，表示当前查询依赖 `id=N` 的查询，于是先执行 `id=N` 的查询。
- `subquery N` 这一行引用了ID值为N的物化字查询的结果。参考：[https://dev.mysql.com/doc/refman/5.7/en/subquery-materialization.html](https://dev.mysql.com/doc/refman/5.7/en/subquery-materialization.html)

## type列

表示关联类型或者访问类型，即 Mysql 决定如何查找表中的行，查找数据行记录的大概范围。
依次从最优到最差分别为：`system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`，一般来说，要**保证查询达到 range 级别，最好达到 ref**。

### NULL值

mysql 能够在优化阶段分解查询语句，在执行阶段不需要访问表或者索引。例如：在索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表。

```sql
EXPLAIN
SELECT MIN(id)
FROM film;
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
    "Extra": "Select tables optimized away"
  }
]
```

### const, system

mysql 能对查询对某部分进行优化并将其转为一个常量(可以看 show warnings 的结果)。用于 `primary key` 或 `unique key` 的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。`system` 是 `const` 的特例，**表里只有一条元组匹配时为** `system**`。

```sql
SET SESSION optimizer_switch = 'derived_merge=off';
EXPLAIN EXTENDED
SELECT *
FROM (SELECT * FROM film WHERE id = 1) tmp;
SET SESSION optimizer_switch = 'derived_merge=on';
```

```json
[
  {
    "id": 1,
    "select_type": "PRIMARY",
    "table": "<derived2>",
    "partitions": null,
    "type": "system",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 1,
    "filtered": 100,
    "Extra": null
  },
  {
    "id": 2,
    "select_type": "DERIVED",
    "table": "film",
    "partitions": null,
    "type": "const",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

可以看到，`id=2` 的语句对应 (`select * from film where id = 1`) 查询，此时比较主键是否相等，类型为 `const`。`id=1` 的查询 临时表中只有一条数据，`const` 转为 `system`。

### eq_ref

主键或者唯一索引的所有部分被连接使用，最多只会返回一条符合条件的记录。这可能是在 `const` 之外最好的联接类型了，简单的 `select` 查询不会出现这种 `type`。

```sql
EXPLAIN
SELECT *
FROM film_actor
         LEFT JOIN film ON film_actor.film_id = film.id;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film_actor",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": null
  },
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "eq_ref",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "vitah.film_actor.film_id",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

### ref

相比 `eq_ref`，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行。

1. 简单的 `select` 查询，`name` 是普通索引（非唯一索引）

```sql
EXPLAIN
SELECT *
FROM film
WHERE name = 'film1';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_name",
    "key": "idx_name",
    "key_len": "33",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

2. 关联表查询，例如下面语句，`idx_film_actor_id` 是表 `film_actor` 字段 `film_id` 和 `actor_id` 的联合索引，这里使用到 `film_actor` 的左边前缀 `film_id` 部分。

```sql
EXPLAIN
SELECT film_id
FROM film
         LEFT JOIN film_actor ON film.id = film_actor.film_id;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "index",
    "possible_keys": null,
    "key": "idx_name",
    "key_len": "33",
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using index"
  },
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film_actor",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_film_actor_id",
    "key": "idx_film_actor_id",
    "key_len": "4",
    "ref": "vitah.film.id",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

### range

范围扫描通常出现在 `in()`, `between`, `>`, `<`, `≥` 等操作中，使用一个索引来检索给定范围的行

```sql
EXPLAIN
SELECT *
FROM actor
WHERE id > 1;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "actor",
    "partitions": null,
    "type": "range",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": null,
    "rows": 2,
    "filtered": 100,
    "Extra": "Using where"
  }
]
```

### index

扫描全索引就能拿到结果，一般是扫描某个二级索引，这种扫描不会从索引树根结点开始查找，而是直接对二级索引的叶子节点遍历和扫描，速度还是比较慢的，这种查询一般为覆盖索引，二级索引一般比较小，所以这种比 ALL 快一些。

```sql
EXPLAIN
SELECT *
FROM film;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "index",
    "possible_keys": null,
    "key": "idx_name",
    "key_len": "33",
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

### ALL

即全表扫描，扫描你的聚簇索引的所有叶子节点。通常这种情况就需要增加索引来优化了。

```sql
EXPLAIN
SELECT *
FROM actor;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "actor",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": null
  }
]
```

## possible_keys列

这一列显示查询可能使用哪些索引来查找。
`explain` 时可能 **出现** `possible_keys` 有列，而 `key` 显示 `NULL` 的情况，是因为表中数据不多，`mysql` 认为索引对此查询帮助不大，选择了全表查询。
如果该列是 `NULL`，则没有相关的索引。在这种情况下，可以通过检查 `where` 子句看是否可以创造一个适当的索引来提高查询性能，然后用 `explain` 查询效果。

## key列

显示 `mysql` 实际采用哪个索引来优化对该表的访问。
如果没有使用索引，该列值是 `NULL`。如果想强制 `mysql` 使用或者忽视 `possible_keys` 列中的索引，可以使用 `force index`、`ignore index`。

## 列key_len

显示了在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列。

比如表 `film_actor` 的联合索引 `idx_film_actor_id` 由 `film_id` 和 `actor_id` 两个 `int` 列组成，并且每个 `int` 是4字节。通过结果中的 `ken_len=4` 可推断出查询使用列第一个列：`film_id` 列来执行索引查找。

```sql
EXPLAIN
SELECT *
FROM film_actor
WHERE film_id = 2;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film_actor",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_film_actor_id",
    "key": "idx_film_actor_id",
    "key_len": "4",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  }
]
```

`key_len` 的计算规则

- 字符串，`char(n)` 和 `varchar(n)`，`n` 代表字符数，不是字节数，如果是 `utf-8`，一个数字或者字母占1字节，如果是汉字占3字节。`char(n)` 存汉字长度为 `3n` 字节，`varchar(n)` 存汉字长度为 `3n+2` 字节，需要额外2字节用来存储字符串长度。
- 数值类型
  - tinyint 1字节
  - smallint 2字节
  - int 4字节
  - bigint 8字节
- 时间类型
  - date 3字节
  - timestamp 4字节
  - datetime 8字节
- 字段允许为 `NULL`，需要额外1字节记录是否为 `NULL`

**索引的最大长度是768字节，当字符串过长时，mysql 会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索引。**

## ref列

显示了在 `key` 列记录的索引中，表查找值所用到的列或常量，常见的有 `const`(常量)，字段名(例如 film.id)

## rows列

`mysql` 估计要读取并检测的行数，并不是结果集里面的行数，**只是一个估计值**。

## filtered列

表示返回结果的行数占需读取行数(列`rows`值)的百分比，`filtered` 列的值越大越好，`filtered` 列的值依赖于统计信息

## Extra列

展示的是额外信息。常见的重要值如下：

### 1. Using index：使用覆盖索引

覆盖索引的定义：mysql 执行计划 `explain` 结果里的 `key` 列有使用索引，如果 `select` 后面查询的字段都可以从这个索引的树中获取，这种情况一般可以说是用到了覆盖索引，`extra` 里一般都有 `using index`。覆盖索引一般针对的是辅助索引，整个查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其他字段值。

- 覆盖索引

```sql
EXPLAIN
SELECT film_id
FROM film_actor
WHERE film_id = 1;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film_actor",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_film_actor_id",
    "key": "idx_film_actor_id",
    "key_len": "4",
    "ref": "const",
    "rows": 2,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

- 非覆盖索引

```sql
EXPLAIN
SELECT *
FROM film_actor
WHERE film_id = 1;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film_actor",
    "partitions": null,
    "type": "ref",
    "possible_keys": "idx_film_actor_id",
    "key": "idx_film_actor_id",
    "key_len": "4",
    "ref": "const",
    "rows": 2,
    "filtered": 100,
    "Extra": null
  }
]
```

### 2. Using where

使用 `where` 语句来处理结果，并且查询的列未被索引覆盖。

```sql
EXPLAIN
SELECT *
FROM actor
WHERE name = 'a';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "actor",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 33.33,
    "Extra": "Using where"
  }
]
```

### 3. Using index condition

查询的列不完全被索引覆盖，`where` 条件是一个前导列的范围

```sql
EXPLAIN
SELECT *
FROM film_actor
WHERE film_id > 1;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film_actor",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_film_actor_id",
    "key": "idx_film_actor_id",
    "key_len": "4",
    "ref": null,
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition"
  }
]
```

### 4. Using temporary

mysql 需要创建一张临时表来处理查询。出现这种情况一般需要进行优化，首先想到用索引来优化

1. `actor.name` 没有索引，此时创建来临时表来 `distinct`

```sql
EXPLAIN
SELECT DISTINCT name
FROM actor;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "actor",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using temporary"
  }
]
```

2. `film.name` 建立来 `idx_name` 索引，此时查询时 `extra` 是 `using index`，没有用临时表

```sql
EXPLAIN
SELECT DISTINCT name
FROM film;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "index",
    "possible_keys": "idx_name",
    "key": "idx_name",
    "key_len": "33",
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

### 5. Using filesort

将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。这种情况下一般也是要考虑用索引来优化。

- 使用外部排序
`actor.name`未创建索引，会浏览`actor`整个表，保存排序关键字`name`和对应的`id`，然后排序`name`并检索行记录。  

```sql
EXPLAIN
SELECT *
FROM actor
ORDER BY name;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "actor",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using filesort"
  }
]
```

- 利用索引优化

`film.name` 建立了 `idx_name` 索引，此时查询时 `extra` 是 `using index`。

```sql
EXPLAIN
SELECT *
FROM film
ORDER BY name;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "film",
    "partitions": null,
    "type": "index",
    "possible_keys": null,
    "key": "idx_name",
    "key_len": "33",
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```

### 6. Select tables optimized away

使用某些聚合函数(比如 `max`、`min`)来访问存在索引的某个字段时

```sql
EXPLAIN
SELECT MIN(id)
FROM film;
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
    "Extra": "Select tables optimized away"
  }
]
```
