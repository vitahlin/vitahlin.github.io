---
title: Mysql如何选择合适的索引
<!-- description: 这是一个副标题 -->
date: 2022-04-13
slug: mysql-how-to-choose-the-right-index
categories:
    - mysql

tags:
    - mysql
    - database
---

## 示例表

### 表结构

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
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='员工记录表';
```

### 测试数据

```sql
INSERT INTO employees(name, age, position, hire_time)
VALUES ('LiLei', 22, 'manager', NOW());
INSERT INTO employees(name, age, position, hire_time)
VALUES ('HanMeimei', 23, 'dev', NOW());
INSERT INTO employees(name, age, position, hire_time)
VALUES ('Lucy', 23, 'dev', NOW());

# 插入一些示例数据
drop procedure if exists insert_emp;
delimiter ;;
create procedure insert_emp()
begin
    declare i int;
    set i = 1;
    while(i <= 100000)
        do
            insert into employees(name, age, position) values (CONCAT('zhuge', i), i, 'dev');
            set i = i + 1;
        end while;
end;;
delimiter ;
call insert_emp();
```

## 不同的索引选择

我们用一个查询语句来做示例，同样的查询条件，仅仅是查询参数的不同。

### 全表扫描示例

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name > 'a';
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
    "Extra": "Using where"
  }
]
```

### 覆盖索引示例

```sql
EXPLAIN
SELECT name, age, position
FROM employees
WHERE name > 'a';
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

### 走索引示例

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name > 'zzz';
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
    "rows": 1,
    "filtered": 100,
    "Extra": "Using index condition"
  }
]
```

可以看到，上述3种情况，mysql选择了不同的查询方案。
mysql最终选择是否走索引或者一张表涉及多个索引最终选择哪个索引，我们可以用trace工具来一查究竟。

## trace介绍

### 用法

开启 trace 工具会影响 mysql 性能，所以只能临时分析 sql 使用，使用后应该立即关闭。

```sql
# 开启trace
SET SESSION optimizer_trace = "enabled=on",end_markers_in_json = ON;

# 关闭 trace
SET SESSION optimizer_trace = "enabled=off";
```

使用trace工具查看语句：

```sql
SELECT *
FROM employees
WHERE name > 'a'
ORDER BY position;

SELECT *
FROM information_schema.optimizer_trace;
```

### 结果分析

```json
{
  "steps": [
    {
      /* 第一阶段，sql准备阶段，格式化sql */
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `employees`.`id` AS `id`,`employees`.`name` AS `name`,`employees`.`age` AS `age`,`employees`.`position` AS `position`,`employees`.`hire_time` AS `hire_time` from `employees` where (`employees`.`name` > 'a') order by `employees`.`position`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      /* 第二阶段，sql优化阶段 */
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            /* 条件处理 */
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`employees`.`name` > 'a')",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "(`employees`.`name` > 'a')"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            /* 表依赖详情 */
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
            ] /* ref_optimizer_key_uses */
          },
          {
            /* 预估表的访问成本 */
            "rows_estimation": [
              {
                "table": "`employees`",
                "range_analysis": {
                  /* 全表扫描情况 */
                  "table_scan": {
                    /* 扫描行数 */
                    "rows": 100269, 
                    
                    /* 扫描成本 */
                    "cost": 20345
                  } /* table_scan */,
                  
                  /* 查询可能使用的索引 */
                  "potential_range_indexes": [
                    {
                      /* 主键索引 */
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      /* 辅助索引 */
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
                  
                  /* 分析各个索引使用成本 */
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_name_age_position",
                        
                        /* 索引使用范围 */
                        "ranges": [
                          "a < name"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        
                        /* 使用该索引获取的记录是否按照主键排序 */
                        "rowid_ordered": false,
                        "using_mrr": false,
                        
                        /* 是否使用覆盖索引 */
                        "index_only": false,
                        /* 索引扫描行数 */
                        "rows": 50134,
                        /* 索引扫描成本 */
                        "cost": 60162,
                        /* 是否选择该索引 */
                        "chosen": false,
                        "cause": "cost"
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */
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
                /* 最优访问路径 */
                "best_access_path": {
                  /* 最终选择的访问路径 */
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 100269,
                      /* 访问类型，scan为全表扫描 */
                      "access_type": "scan",
                      "resulting_rows": 100269,
                      "cost": 20343,
                      /* 是否选择：是 */
                      "chosen": true,
                      "use_tmp_table": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 100269,
                "cost_for_plan": 20343,
                "sort_cost": 100269,
                "new_cost_for_plan": 120612,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`employees`.`name` > 'a')",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`employees`",
                  "attached": "(`employees`.`name` > 'a')"
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
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`employees`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "unknown",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "refine_plan": [
              {
                "table": "`employees`"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      /* 第三阶段，sql执行阶段 */
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
              "rows": 100003,
              "examined_rows": 100003,
              "number_of_tmp_files": 30,
              "sort_buffer_size": 262056,
              "sort_mode": "<sort_key, packed_additional_fields>"
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

上述 json 结果中给出大部分关键字段的注释，可以看到当前执行语句，mysql分析全表扫描比走索引查询的成本 cost 更低，所以选择了全表扫描。
