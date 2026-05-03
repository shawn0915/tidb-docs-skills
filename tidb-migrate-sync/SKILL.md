---
name: tidb-migrate-sync
description: >
  TiDB v8.5 数据迁移工具链选型、配置及问题处理。
  当用户涉及：1) 从 MySQL/Aurora/MariaDB 迁移数据到 TiDB；
  2) TiDB 之间的数据同步；3) TiCDC 配置与 Changefeed 管理；
  4) Lightning 导入 / Dumpling 导出；5) 数据校验 (sync-diff-inspector)；
  6) 分库分表合并迁移时触发。
  不回答通用数据库迁移理论。
---

# TiDB v8.5 数据迁移与同步

## 工具选型决策树

```
用户场景
├── 一次性全量迁移（TB 级）
│   ├── 物理导入（最快）→ TiDB Lightning 物理模式
│   └── 逻辑导入（兼容性好）→ Dumpling + Lightning 逻辑模式
├── 持续增量同步（MySQL → TiDB）
│   └── DM（TiDB Data Migration）支持断点续传、Online DDL 兼容
├── TiDB → 下游系统（Kafka/MySQL/Pulsar/存储）
│   └── TiCDC Changefeed
├── 分库分表合并迁移
│   └── DM 分库分表合并（悲观/乐观模式）
└── 数据一致性校验
    └── sync-diff-inspector
```

### 快速选型表

| 场景 | 推荐工具 | 不支持的场景 |
|------|---------|-------------|
| MySQL → TiDB 全量+增量 | DM | 非 MySQL 协议源 |
| 仅全量导入（TB 级） | Lightning 物理模式 | 目标集群已有数据 |
| 仅全量导出 | Dumpling | 非 SQL 格式导出 |
| TiDB → Kafka | TiCDC | 需要事务保证的批量写入 |
| TiDB → MySQL | TiCDC + MySQL sink | 双向同步（需 BDR 模式） |
| 分库分表合并 | DM + route 规则 | 异构分表 |
| 数据校验 | sync-diff-inspector | 实时校验 |

## TiDB Lightning

### 物理导入模式（最快，有限制）

**必要条件**：
- 目标集群**必须为空**（无数据）
- TiKV 版本 >= v6.1
- 物理导入期间**不能同时运行日志备份 (PITR)**

```bash
# lightning.toml 关键配置
[lightning]
# 并行导入支持多 Lightning 实例同时导入不同数据源
index-concurrency = 2
table-concurrency = 6

[tikv-importer]
backend = "local"  # 物理模式
duplicate-resolution = "replace"  # replace | ignore | error
sorted-kv-dir = "/tmp/sorted-kvs"
```

### 逻辑导入模式

```bash
[tikv-importer]
backend = "tidb"  # 逻辑模式，通过 SQL 写入
# 支持导入到已有数据的集群
# 速度较慢，但兼容性最好
```

### 冲突检测策略

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `replace` | 新数据覆盖旧数据 | 允许覆盖 |
| `ignore` | 保留旧数据，忽略新数据 | 保留旧数据 |
| `error` | 报错停止 | 要求严格一致 |

## DM (TiDB Data Migration)

### v8.5 关键配置

**数据源配置**：
```yaml
# source.yaml
source-id: "mysql-01"
enable-gtid: true
# v8.5.6 新增要求：必须开启 LOCK TABLES 权限
```

**任务配置**：
```yaml
# task.yaml
name: "mysql-to-tidb"
task-mode: all  # all | full | incremental

# 分库分表合并
mysql-instances:
  - source-id: "mysql-01"
    route-rules: ["shard-route"]

routes:
  shard-route:
    schema-pattern: "order_*"
    table-pattern: "t_order"
    target-schema: "db"
    target-table: "order"

# Online DDL 工具兼容（pt-osc / gh-ost）
online-ddl: true
```

### 分库分表合并模式

| 模式 | DDL 冲突处理 | 适用条件 |
|------|-------------|---------|
| 悲观模式 | DDL 冲突时自动暂停任务，人工处理 | 各分表 schema 可能不同 |
| 乐观模式 | DDL 自动兼容，继续同步 | 各分表 schema 必须完全一致 |

### v8.5.6 新特性

- **实验性支持外键同步**：从 MySQL 迁移时可保留外键定义
- **LOCK TABLES 权限**：数据源必须开启此权限

## TiCDC

### v8.5 架构变化

- **默认使用新架构**（基于 TiKV Event Feed）
- 老架构已废弃

### Changefeed 配置

```bash
# 创建同步到 Kafka 的 Changefeed
cdc cli changefeed create \
  --server=http://cdc-server:8300 \
  --sink-uri="kafka://broker:9092/topic-name?protocol=canal-json" \
  --changefeed-id="kafka-sync"

# 支持的协议：canal-json / avro / debezium / open-protocol

# 同步到存储服务（CSV / Parquet）
cdc cli changefeed create \
  --sink-uri="s3://bucket/path?protocol=csv"
```

### 双向复制 (BDR)

```sql
-- 主集群设置
ALTER PRIMARY CLUSTER;

-- 从集群设置
ALTER DR CLUSTER;
```

- 需开启 BDR 模式
- 设置主从集群角色
- 避免双向写入冲突

### TiCDC 常见问题

| 症状 | 原因 | 解决方案 |
|------|------|---------|
| Changefeed failed | 下游 MySQL 磁盘满 | 清理下游磁盘空间 |
| 同步延迟增大 | Kafka topic 分区不足 | 增加分区数 |
| checkpoint 不推进 | TiKV GC 时间 < Changefeed checkpoint | 调大 `gc-ttl` |

## sync-diff-inspector 数据校验

```toml
# config.toml
[data-sources.tidb]
host = "127.0.0.1"
port = 4000

[data-sources.mysql]
host = "127.0.0.1"
port = 3306

[task]
source-instances = ["mysql"]
target-instance = "tidb"

# 忽略某些已知不一致的列
ignore-columns = ["created_at", "updated_at"]
```

## 参考资料

- 当用户需要 DM 任务配置模板时，加载 `references/dm-task-config-template.md`
- 当用户需要 TiCDC Changefeed 完整配置时，加载 `references/ticdc-changefeed-config.md`
- 当用户询问 Lightning 物理 vs 逻辑模式时，加载 `references/lightning-physical-vs-logical.md`