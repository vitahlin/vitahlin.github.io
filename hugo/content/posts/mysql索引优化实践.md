---
title: Mysql索引优化实践
<!-- description: 这是一个副标题 -->
date: 2022-04-12
slug: mysql-indexing-optimization-practices
categories:
    - mysql

tags:
    - mysql
---

## 示例表

### 表结构

```sql
DROP TABLE IF EXISTS employees;
CREATE TABLE `employees`
(
    `id`        INT(11)     NOT NULL AUTO_INCREMENT,
    `name`      VARCHAR(24) NOT NULL DEFAULT '' COMMENT '姓名',
    `age`       INT(11)     NOT NULL DEFAULT '0' COMMENT '年龄',
    `position`  VARCHAR(20) NOT NULL DEFAULT '' COMMENT '职位',
    `hire_time` TIMESTAMP   NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
    PRIMARY KEY (`id`),
    KEY `idx_name_age_position` (`name`, `age`, `position`) USING BTREE
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='员工记录表';
```

### 表数据

```sql
INSERT INTO employees(name, age, position, hire_time)
VALUES ('LiLei', 22, 'manager', NOW());
INSERT INTO employees(name, age, position, hire_time)
VALUES ('HanMeimei', 23, 'dev', NOW());
INSERT INTO employees(name, age, position, hire_time)
VALUES ('Lucy', 23, 'dev', NOW());

# 插入一些示例数据
DROP PROCEDURE IF EXISTS insert_emp;
DELIMITER ;;
CREATE PROCEDURE insert_emp()
BEGIN
    DECLARE i INT;
    SET i = 1;
    WHILE(i <= 100000)
        DO
            INSERT INTO employees(name, age, position) VALUES (CONCAT('zhuge', i), i, 'dev');
            SET i = i + 1;
        END WHILE;
END;;
DELIMITER ;
CALL insert_emp();
```

## 联合索引第一个字段用范围查询

### 会不会走索引

**联合索引第一个字段就用范围查找不会走索引，mysql内部可能觉得第一个字段就用范围查找，结果集应该很大，回表效率不高，还不如全表扫描。**

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name > 'LiLei'
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
    "type": "ALL",
    "possible_keys": "idx_name_age_position",
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 99986,
    "filtered": 0.5,
    "Extra": "Using where"
  }
]
```

### 强制走索引效果

使用了强制走索引让联合索引第一个字段范围查找也用索引，扫描的 `rows` 看上去也少了点，但最终但查找效率不一定会比全表扫描高，因为回表效率不高。

```sql
EXPLAIN
SELECT *
FROM employees FORCE INDEX (idx_name_age_position)
WHERE name > 'LiLei'
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
    "type": "range",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "74",
    "ref": null,
    "rows": 49993,
    "filtered": 1,
    "Extra": "Using index condition"
  }
]
```

### 强制走索引和不走索引效率对比

我们先关闭查询缓存，避免影响查询结果。

```json
# 关闭查询缓存
set global query_cache_size = 0;
set global query_cache_type = 0;
```

#### 不走索引结果

```shell
SELECT *
FROM employees
WHERE name > 'LiLei';

vitah> SELECT *
       FROM employees
       WHERE name > 'LiLei'
[2022-03-26 22:11:22] 在 119 ms (execution: 14 ms, fetching: 105 ms) 内检索到从 1 开始的 2,000 行
```

#### 强制走索引结果

```shell
SELECT *
FROM employees force index (idx_name_age_position)
WHERE name > 'LiLei';

vitah> SELECT *
       FROM employees force index (idx_name_age_position)
       WHERE name > 'LiLei'
[2022-03-26 22:11:48] 在 140 ms (execution: 16 ms, fetching: 124 ms) 内检索到从 1 开始的 2,000 行
```

可以看到，强制走索引的效率甚至还不如不走索引的执行效果。

#### 如何优化？

我们可以使用覆盖索引来优化。什么是覆盖索引？
> 创建一个索引，该索引包含查询中用到的所有字段，称为“覆盖索引”。 使用覆盖索引，MySQL 只需要通过索引就可以查找和返回查询所需要的数据，而不必在使用索引处理数据之后再进行回表操作。 覆盖索引可以一次性完成查询工作，有效减少IO，提高查询效率。

```sql
EXPLAIN
SELECT name, age, position
FROM employees
WHERE name > 'LiLei'
  AND age = 22
  AND position = 'mana er';
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
    "key_len": "74",
    "ref": null,
    "rows": 49954,
    "filtered": 1,
    "Extra": "Using where; Using index"
  }
]
```

## in和or查询

### 是否会走索引？

在数据量比较大的情况下会走索引，在表记录不多的情况会选择全表扫描。

#### 大数据量情况

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name in ('LiLei', 'HanMeimei', 'Lucy')
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
    "type": "range",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using index condition"
  }
]
```

可以看到，`key` 为 `idx_name_age_position` 用到了索引。

#### 小数据量情况

复制同样结构的表，往里面插入3条数据：

```shell
CREATE TABLE `employees_copy`
(
    `id`        int(11)     NOT NULL AUTO_INCREMENT,
    `name`      varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
    `age`       int(11)     NOT NULL DEFAULT '0' COMMENT '年龄',
    `position`  varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
    `hire_time` timestamp   NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
    PRIMARY KEY (`id`),
    KEY `idx_name_age_position` (`name`, `age`, `position`) USING BTREE
) ENGINE = InnoDB
  AUTO_INCREMENT = 100004
  DEFAULT CHARSET = utf8 COMMENT ='员工记录表'
  
NSERT INTO employees_copy(name, age, position, hire_time)
VALUES ('LiLei', 22, 'manager', NOW());
INSERT INTO employees_copy(name, age, position, hire_time)
VALUES ('HanMeimei', 23, 'dev', NOW());
INSERT INTO employees_copy(name, age, position, hire_time)
VALUES ('Lucy', 23, 'dev', NOW());
```

查询语句：

```sql
EXPLAIN
SELECT *
FROM employees_copy
WHERE name in ('LiLei', 'HanMeimei', 'Lucy')
  AND age = 22
  AND position = 'manager';
```

执行结果：

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees_copy",
    "partitions": null,
    "type": "ALL",
    "possible_keys": "idx_name_age_position",
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 3,
    "filtered": 100,
    "Extra": "Using where"
  }
]
```

`type` 为 `ALL`，走的是全表扫描。

## like `kk%` 查询

### 会不会走索引？

#### 大数据量查询

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name like 'LiLei%'
  AND age = 22
  AND position = 'manager';
```

分析结果：

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
    "key_len": "140",
    "ref": null,
    "rows": 1,
    "filtered": 5,
    "Extra": "Using index condition"
  }
]
```

#### 小数据量查询

```shell
EXPLAIN
SELECT *
FROM employees_copy
WHERE name LIKE 'LiLei%'
  AND age = 22
  AND position = 'manager';
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees_copy",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_name_age_position",
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": null,
    "rows": 1,
    "filtered": 33.33,
    "Extra": "Using index condition"
  }
]
```

所以，`like KK%` 语句一般情况下都会走索引，为什么呢？这里其实是用到了**索引下推优化**。

### 索引下推

对于辅助的联合索引 `(name,age,posisiton)`，正常情况按照最左前缀原则，查询语句只会走 `name` 字段索引，
因为 `name` 字段过滤完，得到的索引行里的 `age` 和 `position` 是无序的，无法很好的利用索引。
在Mysql5.6之前的版本，这个查询只能在联合索引里匹配到名字是 `LiLei` 开头的索引，
然后拿这些索引对应的主键逐个回表，到主键索引上找出对应记录，再比较 `age` 和 `position` 两个字段的值是否符合。

Mysql5.6 引入了索引下推优化，可以在索引遍历过程中，对索引包含的所有字段做判断，过滤掉不符合条件的记录再回表，可以有效的减少回表次数。
使用了索引下推之后，上面那个查询在联合索引里匹配到名字符合的索引之后，同时还会在索引里面过滤 `age` 和 `position` 这两个字段，
拿着过滤完剩下的索引的主键ID再回表。索引下推会减少回表次数，对于InnoDB引擎的表索引下推只能用于二级索引，
InnoDB的主键索引树叶子节点上保存的是全行数据，所有这个时候索引下推并不会减少查询全行数据的效果。

#### 范围查找是否会用到索引下推？

没有。猜测原因：估计应该是Mysql认为范围查找过滤的结果集过大，`like kk%` 在绝大多数情况下过滤后的结果集小，所以mysql选择给 `like kk%` 用了索引下推优化，
当然这也不是绝对的，有时 `like kk%` 也可能不用索引下推。

## order by 与group by

### case1: order by用到了联合索引的中间字段

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
  AND position = 'dev'
ORDER BY age;
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
    "key_len": "74",
    "ref": "const",
    "rows": 1,
    "filtered": 10,
    "Extra": "Using index condition"
  }
]
```

最左前缀法则，中间字段不能断，因此查询用到了 `name` 索引，从 `key_len=74` 也能看出。
`age` 列用在了排序之中，因为 `extra` 里面没有 `using filesort`。

### case2: order by用到了联合索引的最后一个字段

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
ORDER BY position;
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
    "key_len": "74",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition; Using filesort"
  }
]
```

`key_len=74`，查询使用了 `name` 索引。由于用了 `position` 进行排序，跳过了 `age`，出现了 `Using filesort`。

### case3

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
ORDER BY age, position;
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
    "key_len": "74",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition"
  }
]
```

查询只用到了索引 `name`，`age` 和 `position` 用于排序，无 `Using filesort`。

### case4

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
ORDER BY position, age;
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
    "key_len": "74",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition; Using filesort"
  }
]
```

和case2 中结果一样。

### case5

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
  AND age = 10
ORDER BY position;
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
    "key_len": "78",
    "ref": "const,const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition"
  }
]
```

与 case4 对比，因为 `age` 为常量，在排序中被优化，所以索引未颠倒，不会出现 `Using filesort`。

### case6

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
ORDER BY age ASC, position DESC;
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
    "key_len": "74",
    "ref": "const",
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition; Using filesort"
  }
]
```

虽然排序的字段与索引顺序一样，且 **`order by` 默认升序**，这里 `position desc` 变成了降序，导致与索引的排序方式不同，
从而产生了 `Using filesort`。Mysql8以上版本有降序索引可以支持该查询方式。

### case7

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name IN ('LiLei', 'Hanmeimei')
ORDER BY age, position;
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
    "key_len": "74",
    "ref": null,
    "rows": 2,
    "filtered": 100,
    "Extra": "Using index condition; Using filesort"
  }
]
```

**对于排序来说，多个相等条件等于范围查询。**

### case8

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name > 'a'
ORDER BY name;
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
    "rows": 100269,
    "filtered": 50,
    "Extra": "Using where; Using filesort"
  }
]
```

可以看到，上述查询并不会用到索引。但是可以使用覆盖索引进行优化。

```sql
EXPLAIN
SELECT name, age, position
FROM employees
WHERE name > 'a'
ORDER BY name;
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
    "key_len": "74",
    "ref": null,
    "rows": 50134,
    "filtered": 100,
    "Extra": "Using where; Using index"
  }
]
```

### 优化总结

- mysql支持两种方式的排序 `filesort` 和 `index`，`Using index` 指扫描索引本身完成排序。`index`效率高，`filesort`效率低。
- order by 满足两种情况会使用 Using index
  - order by 语句使用索引最左前列
  - 使用 where 子句与 order by 子句条件列组合满足索引最左前列
- 尽量在索引列上完成排序，遵循索引建立（索引创建的顺序）时的最左前缀法则
- 如果 order by 的条件不在索引列上，就会产生 Using filesort
- 能用覆盖索引尽量用覆盖索引
- group by 与 order by 很类似，实质是先排序后分组，遵循索引创建顺序的最左前缀法则。对于 group by 的优化如果不需要排序可以加上 order by null禁止排序。注意，where 高于 having，能写在 where 中的限定条件就不要去having 限定了。

## Using filesort文件排序原理

### filesort 文件排序方式

#### 单路排序

一次性取出满足条件所有字段，然后在 sort buffer 中进行排序；用 trace 工具可以看到 sort_model 信息里面显示 <sort_key, additional_fields> 或者 <sort_key, packed_additional_fields>

#### 双路排序

又叫回表排序方式。先根据对于条件取出排序字段和可以直接定位行数据的行ID，然后在 sort buffer 中进行排序，排序完成后需要再次取回其他需要的字段；用trace工具可以看到 sort_model 信息里面显示 <sort_key, rowid>

#### 什么时候用单路排序什么时候用双路排序？

mysql 通过比较系统变量 max_length_for_sort_data(默认1024字节)的大小和需要查询的字段总大小来判断使用哪种排序模式

- 如果字段总长度小于max_length_for_sort_data，使用单路排序
- 字段总长度大于max_length_for_sort_data，使用双路排序

### filesort文件排序示例

#### 单路排序示例

```sql
SET SESSION OPTIMIZER_TRACE = 'enabled=on',END_MARKERS_IN_JSON = ON;

SELECT *
FROM employees
WHERE name = 'lilei'
ORDER BY position;

SELECT *
FROM information_schema.optimizer_trace;
```

```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`employees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from `employees` where (`employees`.`name` = 'zhuge') order by `employees`.`position`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`employees`.`name` = 'zhuge')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`employees`.`name` = 'zhuge')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`employees`.`name` = 'zhuge')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`employees`.`name` = 'zhuge')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`employees`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`employees`",
                "field": "name",
                "equals": "'zhuge'",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`employees`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 100269,
                    "cost": 20345
                  } /* table_scan */,
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_name_age_position",
                      "usable": true,
                      "key_parts": [
                        "name",
                        "age",
                        "position",
                        "id"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_name_age_position",
                        "ranges": [
                          "zhuge <= name <= zhuge"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1,
                        "cost": 2.21,
                        "chosen": true
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "idx_name_age_position",
                      "rows": 1,
                      "ranges": [
                        "zhuge <= name <= zhuge"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,
                    "cost_for_plan": 2.21,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`employees`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "idx_name_age_position",
                      "rows": 1,
                      "cost": 1.2,
                      "chosen": true
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "idx_name_age_position"
                      } /* range_details */,
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 1.2,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`employees`.`name` = 'zhuge')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`employees`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`employees`.`position`",
              "items": [
                {
                  "item": "`employees`.`position`"
                }
              ] /* items */,
              "resulting_clause_is_simple": true,
              "resulting_clause": "`employees`.`position`"
            } /* clause_processing */
          },
          {
            "added_back_ref_condition": "((`employees`.`name` <=> 'zhuge'))"
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`employees`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "idx_name_age_position",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`employees`",
                "pushed_index_condition": "(`employees`.`name` <=> 'zhuge')",
                "table_condition_attached": null
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "filesort_information": [
              {
                "direction": "asc",
                "table": "`employees`",
                "field": "position"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "usable": false,
              "cause": "not applicable (no LIMIT)"
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            
            /* 文件排序信息 */
            "filesort_summary": {
              /* 预扫描行数 */
              "rows": 0,
              /* 参与排序的行 */
              "examined_rows": 0,
              /* 使用临时文件的个数，这个值为0代表全部使用sort_buffer内存排序，否则使用的磁盘文件排序 */
              "number_of_tmp_files": 0,
              /* 排序缓存的大小，单位Byte */
              "sort_buffer_size": 262056,
              /* 排序方式，这里是单路排序 */
              "sort_mode": "<sort_key, packed_additional_fields>"
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

单路排序过程

1. 从索引name找到第一个满足条件name='lilei' 的主键ID
2. 根据主键ID取出整行所有字段的值，存sort_buffer中
3. 从索引name找到下一个满足条件的主键ID
4. 重复2，3步骤知道条件不满足 name='lilei'
5. 对sort_buffer中的数据按照字段position进行排序
6. 返回结果给客户端

#### 双路排序示例

```sql
SET MAX_LENGTH_FOR_SORT_DATA = 10;

SELECT *
FROM employees
WHERE name = 'zhuge'
ORDER BY position;

SELECT *
FROM information_schema.optimizer_trace;
```

```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`employees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from `employees` where (`employees`.`name` = 'zhuge') order by `employees`.`position`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`employees`.`name` = 'zhuge')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`employees`.`name` = 'zhuge')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`employees`.`name` = 'zhuge')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`employees`.`name` = 'zhuge')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`employees`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`employees`",
                "field": "name",
                "equals": "'zhuge'",
                "null_rejecting": false
              }
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`employees`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 100269,
                    "cost": 20345
                  } /* table_scan */,
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_name_age_position",
                      "usable": true,
                      "key_parts": [
                        "name",
                        "age",
                        "position",
                        "id"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_name_age_position",
                        "ranges": [
                          "zhuge <= name <= zhuge"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 1,
                        "cost": 2.21,
                        "chosen": true
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "idx_name_age_position",
                      "rows": 1,
                      "ranges": [
                        "zhuge <= name <= zhuge"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 1,
                    "cost_for_plan": 2.21,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`employees`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "idx_name_age_position",
                      "rows": 1,
                      "cost": 1.2,
                      "chosen": true
                    },
                    {
                      "access_type": "range",
                      "range_details": {
                        "used_index": "idx_name_age_position"
                      } /* range_details */,
                      "chosen": false,
                      "cause": "heuristic_index_cheaper"
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 1.2,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`employees`.`name` = 'zhuge')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`employees`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "`employees`.`position`",
              "items": [
                {
                  "item": "`employees`.`position`"
                }
              ] /* items */,
              "resulting_clause_is_simple": true,
              "resulting_clause": "`employees`.`position`"
            } /* clause_processing */
          },
          {
            "added_back_ref_condition": "((`employees`.`name` <=> 'zhuge'))"
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`employees`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "idx_name_age_position",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`employees`",
                "pushed_index_condition": "(`employees`.`name` <=> 'zhuge')",
                "table_condition_attached": null
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "filesort_information": [
              {
                "direction": "asc",
                "table": "`employees`",
                "field": "position"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "usable": false,
              "cause": "not applicable (no LIMIT)"
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            "filesort_summary": {
              "rows": 0,
              "examined_rows": 0,
              "number_of_tmp_files": 0,
              "sort_buffer_size": 262136,
              "sort_mode": "<sort_key, rowid>"
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

可以看到这里的，sort_mode值为<sort_key, rowid>，这里已经变为双路排序。

双路排序过程

1. 从索引name找到第一个满足条件name='lilei' 的主键ID
2. 根据主键ID取出整行，把排序字段position和主键ID这两个字段放到sort_buffer中
3. 从索引name找到下一个满足条件的主键ID
4. 重复2，3步骤知道条件不满足 name='lilei'
5. 对sort_buffer中的字段position和主键ID按照字段position进行排序
6. 遍历排序好的ID和字段position，按照主键ID值回到原表中取出所有字段的值返回给客户端

#### 两种排序方式对比

单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键 和需要排序的字段放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。
如果 MySQL 排序内存 sort_buffer 配置的比较小并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配置小点，让优化器选择使用双路排序算法，可以在sort_buffer 中一次排序更 多的行，只是需要再根据主键回到原表取数据。
如果 MySQL 排序内存有条件可以配置比较大，可以适当增大max_length_for_sort_data 的值，让优化器 优先选择全字段排序(单路排序)，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查询结果了。
所以，MySQL通过 max_length_for_sort_data 这个参数来控制排序，在不同场景使用不同的排序模式， 从而提升排序效率。

注意，如果全部使用sort_buffer内存排序一般情况下效率会高于磁盘文件排序，但不能因为这个就随便增 大sort_buffer(默认1M)，mysql很多参数设置都是做过优化的，不要轻易调整。

## 分页查询优化

### 普通的分页查询

```sql
EXPLAIN
SELECT *
FROM employees
LIMIT 10000,5;
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
    "rows": 100269,
    "filtered": 100,
    "Extra": null
  }
]
```

上述查询从表中取出从10000开始的5行记录。看似只查询了5条记录，实际上条SQL是先读取10005条记录，然后抛弃前10000条记录读到后面10条想要的记录。因此要查询一张大表比较靠后的数据，执行效率是非常低的。

### 常见的优化

#### 自增且连续的主键排序的分页查询

当ID都是自增且连续主键排序，可以将上述查询换成:

```sql
SELECT *
FROM employees
WHERE id > 10000
LIMIT 5;
```

```json
[
  {
    "id": 1,
    "select_type": "SIMPLE",
    "table": "employees",
    "partitions": null,
    "type": "range",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": null,
    "rows": 50134,
    "filtered": 100,
    "Extra": "Using where"
  }
]
```

缺点：

- 在很多场景不实用，因为表中可能某些记录被删，主键不连续导致结果不一致。
- 如果原来SQL 是order by 非主键的字段，按照这种优化会导致两条SQL结果不一致。

#### 非主键字段排序的分页查询

```sql
EXPLAIN
SELECT *
FROM employees
ORDER BY name
LIMIT 90000,5
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
    "rows": 100269,
    "filtered": 100,
    "Extra": "Using filesort"
  }
]
```

查询并没有使用name字段的索引。具体原因是扫描整个索引并查找到没有索引的行的成本比扫描全表的成本更高，所以优化器放弃使用索引。

那么如何优化呢？
关键是让排序时返回的字段尽可能少，所以可以让排序和分页操作先查出主键，再根据主键查到对应的记录，SQL改写如下：

```sql
EXPLAIN
SELECT *
FROM employees e
         INNER JOIN (SELECT id FROM employees ORDER BY name LIMIT 90000,5) ed ON e.id = ed.id
```

```json
[
  {
    "id": 1,
    "select_type": "PRIMARY",
    "table": "<derived2>",
    "partitions": null,
    "type": "ALL",
    "possible_keys": null,
    "key": null,
    "key_len": null,
    "ref": null,
    "rows": 90005,
    "filtered": 100,
    "Extra": null
  },
  {
    "id": 1,
    "select_type": "PRIMARY",
    "table": "e",
    "partitions": null,
    "type": "eq_ref",
    "possible_keys": "PRIMARY",
    "key": "PRIMARY",
    "key_len": "4",
    "ref": "ed.id",
    "rows": 1,
    "filtered": 100,
    "Extra": null
  },
  {
    "id": 2,
    "select_type": "DERIVED",
    "table": "employees",
    "partitions": null,
    "type": "index",
    "possible_keys": null,
    "key": "idx_name_age_position",
    "key_len": "140",
    "ref": null,
    "rows": 90005,
    "filtered": 100,
    "Extra": "Using index"
  }
]
```
