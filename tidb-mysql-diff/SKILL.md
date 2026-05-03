---
name: tidb-mysql-diff
description: >
  TiDB v8.5 与 MySQL 的兼容性差异、迁移评估及问题修复。
  当用户涉及：1) MySQL 迁移到 TiDB 的兼容性评估；
  2) "MySQL 能跑但 TiDB 报错"的语法问题；
  3) 字符集/collation 差异；
  4) 自增列、外键、存储过程等特性差异；
  5) MySQL 应用迁移后行为不一致时触发。
  不回答通用 SQL 问题，只解决 TiDB 与 MySQL 的差异。
---

# TiDB v8.5 与 MySQL 兼容性差异

## v8.5 关键差异清单

### 完全不支持（必须改写）

| 功能 | TiDB 状态 | 替代方案 |
|------|----------|---------|
| 存储过程 (Stored Procedures) | 不支持 | CTE / 应用层封装 |
| 自定义函数 (UDF) | 不支持 | 应用层实现 |
| 触发器 (Triggers) | 不支持 | 应用层钩子 / TiCDC 事件捕获 |
| 事件调度器 (Event Scheduler) | 不支持 | Cron + 应用层 / TiDB 定时任务 |
| `SELECT ... INTO OUTFILE` | 不支持 | `Dumpling` 导出 |
| `LOAD DATA ... LOCAL INFILE` | 部分支持 | `Lightning` 物理导入（更快） |
| 空间函数 (GIS/GEOMETRY) | 部分支持 | 不完整，需逐个验证 |
| `CREATE TABLE ... AS SELECT` | 不支持 #4754 | 先 CREATE 再 INSERT ... SELECT |
| 全文索引 (Full-text index) | v8.1+ 支持 | 升级版本或改用搜索引擎 |
| `CHECK TABLE` / `CHECKSUM TABLE` | 不支持 | `ADMIN CHECK TABLE` |
| XA 语法（SQL 接口） | 不支持 | TiDB 内部两阶段提交，不暴露 SQL 接口 |
| `HANDLER` 语句 | 不支持 | 标准 SQL 替代 |

### 行为差异（最容易踩坑）

#### 自增列 (AUTO_INCREMENT)

- **全局唯一但不保证连续**
- TiDB 采用 `auto_id_cache` 机制，每个 TiDB 节点缓存一段 ID，大表可能有 30000 级空洞
- **若业务依赖"连续自增"**：改用应用层发号器或 `AUTO_RANDOM`
- `AUTO_RANDOM`：解决写热点，但 ID 不连续且有特定格式

#### 事务隔离

- 默认 `REPEATABLE READ`
- **实现为快照隔离 (Snapshot Isolation)**，不存在幻读
- MySQL InnoDB 用 Next-Key Lock 实现，存在幻读保护
- **迁移注意**：若应用依赖间隙锁 (Gap Lock) 行为，需调整逻辑

#### 乐观/悲观事务模式

- v4.0+ **默认悲观事务** (`tidb_txn_mode='pessimistic'`)
- 若应用从 MySQL 迁移且依赖 `SELECT FOR UPDATE`，确认当前模式
- 乐观事务冲突时自动重试，但 `ERROR 8028` 表示重试次数耗尽

#### GROUP BY 严格模式

- TiDB 比 MySQL 更严格
- **不允许** `SELECT` 非聚合列未出现在 `GROUP BY` 中
- MySQL 在非严格模式下允许（取不确定值）

#### 字符集与排序规则

| 项目 | TiDB v8.5 | MySQL 常用 |
|------|-----------|-----------|
| 默认字符集 | `utf8mb4` | `utf8mb4` |
| 默认排序规则 | `utf8mb4_bin`（大小写敏感） | `utf8mb4_general_ci`（大小写不敏感） |
| 大小写敏感 | 默认敏感 | 取决于 collation |

- **迁移陷阱**：若 MySQL 用 `utf8mb4_general_ci`，迁移后查询结果可能因大小写敏感而不一致
- 解决：建库时显式指定 `utf8mb4_general_ci`

#### 外键 (Foreign Keys)

- v8.5 **语法解析外键但不强制执行**
- 应用需自行维护参照完整性
- DM v8.5.6 实验性支持外键同步（从 MySQL 迁移时保留外键定义）

### v8.5 新特性注意点

| 特性 | 说明 | 迁移影响 |
|------|------|---------|
| 废弃 stats v1 | `analyze_version=2` 为默认 | 确认统计信息版本 |
| Global Indexes | 分区表支持全局索引 | 行为与 MySQL 分区表索引有差异 |
| LATERAL derived tables | v8.5 新增 | MySQL 8.0.14+ 也有，语法需核对 |
| 列级权限 | `GRANT SELECT(col) ON ...` | MySQL 5.7 不支持，8.0 支持 |
| 列级掩码策略 | 动态数据脱敏 | TiDB 特有 |

## 诊断模板（用户报错时的排查路径）

### 步骤1：确认报错类型

| 错误码 | 含义 | 常见原因 |
|--------|------|---------|
| `ERROR 8108` | 语法不兼容 | 存储过程、特定函数、不支持的语法 |
| `ERROR 9005` | Region 不可用 | TiKV 节点异常、网络分区 |
| `ERROR 8028` | 事务冲突重试失败 | 乐观事务冲突，建议改悲观事务 |
| `ERROR 8003` | 写入冲突 | 并发写入同一 Key |
| `ERROR 1064` | SQL 语法错误 | 兼容性问题 |

### 步骤2：提供替代方案

- **存储过程** → CTE (Common Table Expression) / 应用层封装
- `FIND_IN_SET` / `GROUP_CONCAT` → v8.5 已支持，直接使用
- `LOCK TABLES` → 悲观事务 + `SELECT FOR UPDATE`
- `REPLACE INTO` → `INSERT ... ON DUPLICATE KEY UPDATE`
- `INSERT IGNORE` → 支持，但注意与 MySQL 行为一致

### 步骤3：迁移工具链

| 工具 | 用途 | 命令示例 |
|------|------|---------|
| Dumpling | 导出 schema + 数据（逻辑） | `dumpling -u root -p -P 4000 -o /output` |
| Lightning | 导入数据（物理/逻辑模式） | `tidb-lightning -config lightning.toml` |
| DM | 增量同步 + 分库分表合并 | `tiup dmctl --master-addr :8261 start-task task.yaml` |
| sync-diff-inspector | 数据一致性校验 | `sync_diff_inspector -C config.toml` |

## 迁移评估检查清单

在正式迁移前，确认以下项目：

- [ ] 应用是否使用存储过程/触发器/UDF
- [ ] 是否依赖连续自增 ID
- [ ] 是否依赖大小写不敏感的查询（默认 collation 不同）
- [ ] 是否使用外键约束（TiDB 不强制执行）
- [ ] 是否依赖间隙锁 (Gap Lock)
- [ ] 是否使用 MySQL 8.0 特有函数
- [ ] 大事务大小是否超过 10GB（TiDB 限制）

## 参考资料

- 当用户询问具体函数兼容性时，加载 `references/function-compatibility.md`
- 当用户报具体错误码时，加载 `references/error-code-index.md`
- 当用户需要完整迁移方案时，加载 `references/migration-playbook.md`