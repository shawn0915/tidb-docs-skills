---
name: tidb-sql-tuning
description: >
  TiDB v8.5 SQL 性能调优、执行计划解读与优化器控制。
  当用户涉及：1) 慢查询分析与优化；2) EXPLAIN 执行计划解读；
  3) 索引选择与创建建议；4) Optimizer Hints 使用；
  5) 统计信息管理；6) MPP 模式查询优化；
  7) 热点问题处理；8) Runaway Query 管控时触发。
  不回答通用 SQL 教学。
author: "Shawn Yan"
author_url: "https://shawnyan.cn"
author_role:
  - "TiDB 社区版主"
  - "Oracle ACE"
  - "公众号「少安事务所」主笔"
doc_version: "TiDB v8.5 LTS"

# TiDB v8.5 SQL 性能调优

## 执行计划解读（TiDB 特有算子）

| 算子 | 含义 | 优化建议 |
|------|------|---------|
| `TableFullScan` | 全表扫描 | 检查是否有可用索引、谓词是否能下推 |
| `IndexLookUp` | 先读索引再回表查数据 | 若回表量大，考虑覆盖索引或 TiFlash |
| `IndexMerge` | 多索引合并（v8.5 默认开启） | 关注 `union`/`intersection` 类型，性能可能优于单索引 |
| `HashAgg` | 哈希聚合 | 内存不足时会 spill 到磁盘，关注 `tidb_mem_quota_query` |
| `StreamAgg` | 流式聚合（利用有序数据） | 数据有序时更优，内存占用小 |
| `ExchangeSender/Receiver` | MPP 数据交换 | TiFlash MPP 模式特有，跨节点 shuffle 数据 |
| `TableReader` | TiDB 从 TiKV 读数据 | 关注 `pushdown` 条件是否充分下推到 TiKV |
| `IndexReader` | TiDB 从索引读数据 | 无需回表时性能最优（覆盖索引） |
| `PointGet` / `BatchPointGet` | 点查 / 批量点查 | 最优访问路径，通过主键或唯一索引 |
| `Sort` | 显式排序 | 检查是否可利用索引顺序消除排序 |
| `TopN` | 取前 N 条 | 若 N 小，优先尝试索引顺序 |
| `HashJoin` | 哈希连接 | 大表join常用，关注内存使用 |
| `MergeJoin` | 归并连接 | 双方有序时更优 |
| `IndexHashJoin` | 索引嵌套循环连接 | 外表小、内表可走索引时 |

## v8.5 优化器新特性

### Runtime Filter（默认开启）

- 下推 Bloom Filter 到 TiKV，减少扫描数据量
- 对 Join 查询优化效果明显
- 可通过 `tidb_runtime_filter_mode` 控制

### Index Advisor（实验特性）

```sql
-- 获取索引推荐
ADMIN SHOW INDEX ADVISE FOR DATABASE db_name;
-- 或针对具体查询
ADMIN SHOW INDEX ADVISE FOR SELECT ...;
```

### Optimizer Fix Controls

通过 `tidb_opt_fix_control` 精细控制优化器行为：

```sql
-- 示例：禁用特定优化规则
SET SESSION tidb_opt_fix_control = "disable_rules=predicate_push_down";
```

### 非 Prepare 语句执行计划缓存（v8.5 增强）

- v8.5 支持更多场景的计划复用
- 减少 SQL 解析和优化开销
- 对 ORM 框架（如 MyBatis）友好

### 废弃 stats v1

- `analyze_version=2` 为默认，不再维护 v1
- 迁移后需确认统计信息版本

## 慢查询诊断模板

### 步骤1：定位慢查询

```sql
-- 方法1：查询 SLOW_QUERY 表
SELECT query_time, query, plan_digest
FROM information_schema.slow_query
WHERE time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
ORDER BY query_time DESC
LIMIT 10;

-- 方法2：TiDB Dashboard Slow Query 页面（可视化）
-- http://<pd>:2379/dashboard/#/slow_query
```

### 步骤2：分析慢查询特征

| 特征 | 诊断 | 优化方向 |
|------|------|---------|
| `Query_time` > 1s 且 `Process_keys` 很大 | 扫描量大 | 加索引、优化谓词下推 |
| `Query_time` 大但 `Process_keys` 小 | 锁等待或网络延迟 | 检查事务、网络、连接池 |
| `Cop_time` 占比高 | TiKV 处理慢 | 检查 Region 热点、TiKV 资源 |
| `Wait_time` 占比高 | 锁冲突或资源排队 | 检查 Runaway Query、资源组 |
| `Plan_digest` 变化频繁 | 执行计划不稳定 | 收集统计信息、绑定执行计划 |

### 步骤3：验证优化效果

```sql
-- 对比 estRows（估算）与 actRows（实际）
EXPLAIN ANALYZE FORMAT='verbose' SELECT ...;

-- 若 estRows 与 actRows 偏差 > 10x，说明统计信息不准确
ANALYZE TABLE table_name;
```

## 统计信息管理

```sql
-- 收集统计信息（v8.5 默认 analyze_version=2）
ANALYZE TABLE table_name;

-- 查看统计信息健康度（< 80 表示需要更新）
SHOW STATS_HEALTHY WHERE table_name = 'xxx';

-- 自动收集配置
SHOW VARIABLES LIKE 'tidb_auto_analyze%';
-- tidb_auto_analyze_ratio = 0.5  （修改行数超过50%触发）
-- tidb_auto_analyze_start_time = '00:00 +0000'
-- tidb_auto_analyze_end_time = '23:59 +0000'
```

## 热点问题处理

### 读热点

- 症状：某些 Region 的读 QPS 明显高于其他
- 诊断：TiDB Dashboard → Key Visualizer
- 解决方案：
  ```sql
  -- 1. 建表时设置预分区
  CREATE TABLE t (id INT PRIMARY KEY) SHARD_ROW_ID_BITS = 4 PRE_SPLIT_REGIONS = 4;

  -- 2. 已有表手动分裂 Region
  SPLIT TABLE table_name BETWEEN (0) AND (1000000000) REGIONS 16;
  ```

### 写热点

- 症状：某些 Region 的写入量明显高于其他
- 常见原因：`AUTO_INCREMENT` 主键导致单调递增写入
- 解决方案：
  ```sql
  -- 1. 用 AUTO_RANDOM 替代 AUTO_INCREMENT
  CREATE TABLE t (id BIGINT PRIMARY KEY AUTO_RANDOM, ...);

  -- 2. 手动 split Region
  SPLIT TABLE table_name INDEX idx_name BETWEEN ("a") AND ("z") REGIONS 26;
  ```

### TiKV 热点 Region 定位

```bash
# 使用 pd-ctl 查看 hot regions
tiup ctl:v8.5.6 pd -u http://<pd>:2379 hot read
tiup ctl:v8.5.6 pd -u http://<pd>:2379 hot write

# TiDB Dashboard Key Visualizer（最直观）
# http://<pd>:2379/dashboard/#/keyvis
```

## Optimizer Hints 速查

| Hint | 用途 | 示例 |
|------|------|------|
| `/*+ use_index(t, idx) */` | 强制使用指定索引 | `SELECT /*+ use_index(t, idx_a) */ * FROM t` |
| `/*+ ignore_index(t, idx) */` | 忽略指定索引 | `SELECT /*+ ignore_index(t, idx_a) */ * FROM t` |
| `/*+ read_from_storage(tikv[t]) */` | 强制走 TiKV | `SELECT /*+ read_from_storage(tikv[t]) */ * FROM t` |
| `/*+ read_from_storage(tiflash[t]) */` | 强制走 TiFlash | `SELECT /*+ read_from_storage(tiflash[t]) */ * FROM t` |
| `/*+ hash_join(t1, t2) */` | 强制 Hash Join | `SELECT /*+ hash_join(t1, t2) */ * FROM t1, t2` |
| `/*+ merge_join(t1, t2) */` | 强制 Merge Join | `SELECT /*+ merge_join(t1, t2) */ * FROM t1, t2` |
| `/*+ stream_agg() */` | 强制 Stream Aggregate | `SELECT /*+ stream_agg() */ COUNT(*) FROM t` |
| `/*+ max_execution_time(1000) */` | 限制执行时间(ms) | `SELECT /*+ max_execution_time(1000) */ * FROM t` |
| `/*+ memory_quota(1 GB) */` | 限制内存使用 | `SELECT /*+ memory_quota(1 GB) */ * FROM t` |

## 内存与资源管控

```sql
-- 单条查询内存限制（默认 1GB）
SET GLOBAL tidb_mem_quota_query = 1073741824;

-- 磁盘 spill 限制
SET GLOBAL tidb_tmp_storage_quota = 64424509440;  -- 60GB

-- 创建资源组（Runaway Query 管控）
CREATE RESOURCE GROUP batch
  RU_PER_SEC = 1000
  BURSTABLE
  QUERY_LIMIT=(EXEC_ELAPSED='1m', ACTION=KILL);

-- 将用户绑定到资源组
ALTER USER 'batch_user' RESOURCE GROUP batch;
```

## 参考资料

- 当用户需要完整 Hint 列表时，加载 `references/optimizer-hints-full-list.md`
- 当用户需要系统变量调优指南时，加载 `references/system-variables-tuning.md`
- 当用户报 8000-9000 错误码时，加载 `references/error-code-8000-9000.md`