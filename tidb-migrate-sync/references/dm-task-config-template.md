# DM v8.5 任务配置模板

## 全量+增量迁移任务

```yaml
# task.yaml
name: "mysql-to-tidb"
task-mode: all  # all: 全量+增量 | full: 仅全量 | incremental: 仅增量

# 目标 TiDB 配置
target-database:
  host: "10.0.1.11"
  port: 4000
  user: "root"
  password: "password"

# MySQL 数据源
mysql-instances:
  - source-id: "mysql-01"
    block-allow-list: "ba-list"
    route-rules: ["shard-route-1", "shard-route-2"]
    filter-rules: ["filter-event"]
    mydumper-config-name: "global-dump"
    loader-config-name: "global-load"
    syncer-config-name: "global-sync"
    meta:                          # 增量起始位置（不填则从当前 binlog 开始）
      binlog-name: "mysql-bin.000001"
      binlog-pos: 154

  - source-id: "mysql-02"
    block-allow-list: "ba-list"
    route-rules: ["shard-route-1", "shard-route-2"]
    mydumper-config-name: "global-dump"
    loader-config-name: "global-load"
    syncer-config-name: "global-sync"

# 数据源配置（通过 dmctl operate-source 加载）
# source-1.yaml:
#   source-id: "mysql-01"
#   enable-gtid: true
#   from:
#     host: "10.0.1.101"
#     port: 3306
#     user: "dm_user"
#     password: "password"

# 黑白名单
block-allow-list:
  ba-list:
    do-dbs: ["order_db", "user_db"]
    ignore-dbs: ["test", "mysql"]
    do-tables:
      - db-name: "order_db"
        tbl-name: "t_order"

# 分库分表合并规则
routes:
  shard-route-1:
    schema-pattern: "order_db_*"
    table-pattern: "t_order"
    target-schema: "order_db"
    target-table: "t_order"
  shard-route-2:
    schema-pattern: "user_db_*"
    table-pattern: "t_user"
    target-schema: "user_db"
    target-table: "t_user"

# 事件过滤
filters:
  filter-event:
    schema-pattern: "order_db"
    table-pattern: "t_order"
    events: ["drop database", "truncate table", "drop table"]
    action: Ignore

# Online DDL 工具兼容（pt-osc / gh-ost）
online-ddl: true

# mydumper 配置
mydumpers:
  global-dump:
    threads: 4
    chunk-filesize: 64
    skip-tz-utc: true
    extra-args: "--consistency none"

# loader 配置
loaders:
  global-load:
    pool-size: 16
    dir: "./dumped_data"

# syncer 配置
syncers:
  global-sync:
    worker-count: 16
    batch: 100
    queue-size: 1024
    checkpoint-flush-interval: 30
```

## 常用命令

```bash
# 1. 加载数据源
tiup dmctl --master-addr :8261 operate-source create source-1.yaml

# 2. 启动迁移任务
tiup dmctl --master-addr :8261 start-task task.yaml

# 3. 查看任务状态
tiup dmctl --master-addr :8261 query-status task-name

# 4. 暂停/恢复任务
tiup dmctl --master-addr :8261 pause-task task-name
tiup dmctl --master-addr :8261 resume-task task-name

# 5. 停止任务（保留 checkpoint）
tiup dmctl --master-addr :8261 stop-task task-name

# 6. 查看迁移延迟
tiup dmctl --master-addr :8261 query-status task-name | grep seconds
```
