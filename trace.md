# trace 分析SQL语句
mysql 5.6提供了对SQL的跟踪trace，通过trace文件能够进一步了解SQL自动优化的过程。（比如真正用到了哪些索引）
首先需要打开trace，设计格式为JSON，设计trace最大能够使用的内存
```SQL
mysql> SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=ON;
Query OK, 0 rows affected (0.00 sec)

mysql> SET OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;
Query OK, 0 rows affected (0.00 sec)

mysql> select count(*) from ucloud.tbl_user;
+----------+
| count(*) |
+----------+
|  3291437 |
+----------+
1 row in set (1.27 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G;
*************************** 1. row ***************************
                            QUERY: select count(*) from ucloud.tbl_user
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select count(0) AS `count(*)` from `ucloud`.`tbl_user`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "table_dependencies": [
              {
                "table": "`ucloud`.`tbl_user`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "rows_estimation": [
              {
                "table": "`ucloud`.`tbl_user`",
                "table_scan": {
                  "rows": 3096932,
                  "cost": 96320
                } /* table_scan */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`ucloud`.`tbl_user`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "scan",
                      "rows": 3.1e6,
                      "cost": 715706,
                      "chosen": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "cost_for_plan": 715706,
                "rows_for_plan": 3.1e6,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": null,
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`ucloud`.`tbl_user`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "refine_plan": [
              {
                "table": "`ucloud`.`tbl_user`",
                "access_type": "index_scan"
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
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
          INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.01 sec)

```

ლ(′◉❥◉｀ლ)
