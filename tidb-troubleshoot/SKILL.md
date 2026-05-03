---
name: tidb-troubleshoot
description: >
  TiDB v8.5 集群故障定位、诊断方法与常见错误处理。
  当用户涉及：1) 集群异常、节点宕机；2) 慢查询、OOM、热点；
  3) 锁冲突、TTL 超时；4) TiCDC/TiKV/TiFlash 异常；
  5) 报警响应与处理；6) 读写延迟高；7) 节点状态异常时触发。
  不提供 Linux 基础运维指导。
author: "Shawn Yan"
author_url: "https://shawnyan.cn"
author_role:
  - "TiDB 社区版主"
  - "Oracle ACE"
  - "公众号「少安事务所」主笔"
doc_version: "TiDB v8.5 LTS"

# TiDB v8.5 故障诊断与排查

## 诊断工具速查

| 工具 | 用途 | 访问方式 |
|------|------|---------|
| TiDB Dashboard | 可视化诊断中心 | `http://<pd>:2379/dashboard` |
| Top SQL | 定位资源消耗大户 | TiDB Dashboard → Top SQL |
| Key Visualizer | 热点 Region 定位 | TiDB Dashboard → Traffic |
| Slow Query | 慢查询分析 | TiDB Dashboard → Slow Query |
| SQL Plan Replayer | 保存现场执行计划 | `PLAN REPLAYER DUMP EXPLAIN ANALYZE ...` |
| Clinic | 远程诊断支持 | `tiup clinic diag collect <cluster>` |
| Grafana | 监控面板 | `http://<grafana>:3000` |
| Prometheus | 指标查询 | `http://<prometheus>:9090` |

## 按症状诊断

### 症状：TiDB OOM (Out of Memory)

1. 检查 `tidb_mem_quota_query` 是否过小（默认 1GB）
   ```sql
   SHOW VARIABLES LIKE 'tidb_mem_quota_query';
   ```

2. 检查是否有大查询未走索引
   ```sql
   -- 查看最近慢查询的执行计划
   SELECT query, plan FROM information_schema.slow_query 
   WHERE plan_digest != '' ORDER BY query_time DESC LIMIT 5;
   ```

3. 检查磁盘 spill 限制
   ```sql
   SHOW VARIABLES LIKE 'tidb_tmp_storage_quota';
   ```

4. 检查是否有 Runaway Query 被资源管控 kill
   ```sql
   -- 查看资源组管控记录
   SELECT * FROM information_schema.runaway_transactions;
   ```

5. **解决方案**：
   - 调大 `tidb_mem_quota_query`
   - 优化 SQL（加索引、减少扫描量）
   - 启用磁盘 spill（`tidb_enable_tmp_storage_on_oom = ON`）
   - 设置资源组限制（`QUERY_LIMIT`）

### 症状：读写延迟高

#### TiKV 侧排查

```
Grafana → TiKV-Details → gRPC 消息耗时
```

| 指标 | 含义 | 排查方向 |
|------|------|---------|
| `store duration` 高 | 存储层处理慢 | 检查磁盘 IO（`iostat -x 1`）、RocksDB 压缩 |
| `apply duration` 高 | Raft apply 慢 | 检查 Region 数量是否过多（> 3万/节点需优化） |
| `coprocessor duration` 高 | 计算层处理慢 | 检查是否有大范围扫描、热点 |

#### TiDB 侧排查

```
Grafana → TiDB → 解析/编译/执行耗时
```

| 指标 | 含义 | 排查方向 |
|------|------|---------|
| `compile` 耗时高 | 解析/优化慢 | 检查统计信息是否过期（`SHOW STATS_HEALTHY`） |
| `execution` 耗时高 | 执行慢 | 检查 Coprocessor 任务等待、数据量 |
| `wait_duration` 高 | 等待时间长 | 检查锁冲突、资源排队 |

### 症状：锁冲突 / TTL 超时

#### 悲观事务锁冲突

```sql
-- 查看当前锁等待链
SELECT * FROM information_schema.DATA_LOCK_WAITS;

-- 查看事务信息
SELECT * FROM information_schema.cluster_tidb_trx;
```

#### 乐观事务冲突

- `ERROR 8028`：乐观事务重试次数耗尽
- **建议**：改为悲观事务（`SET GLOBAL tidb_txn_mode = 'pessimistic'`）
- 或在应用层增加重试逻辑

#### 大事务限制

- TiDB 默认限制单事务大小 **10GB**（`txn-total_size-limit`）
- 单事务影响行数建议不超过 300万
- 超大事务可能导致：
  - TiKV OOM
  - 提交耗时过长
  - 锁冲突概率增加

### 症状：TiCDC 同步中断

```bash
# 1. 查看 Changefeed 状态
cdc cli changefeed list --server=http://<cdc>:8300

# 2. 若状态为 failed，查看错误日志
cdc cli changefeed query --server=http://<cdc>:8300 --changefeed-id=<id>
```

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `failed` - 下游磁盘满 | MySQL/Kafka 磁盘空间不足 | 清理下游磁盘 |
| `failed` - GC TTL | TiKV GC 时间 < Changefeed checkpoint | 调大 `gc-ttl`（`tiup cluster edit-config`） |
| `failed` - Kafka 分区不足 | 生产速度 > 消费速度 | 增加 Kafka topic 分区数 |
| `stopped` | 手动停止或异常 | `cdc cli changefeed resume --server=... --changefeed-id=<id>` |
| `error` - 表不存在 | 下游表被删除 | 下游重建表后重启 changefeed |

### 症状：TiKV 节点异常

```bash
# 1. 查看 TiKV 状态
tiup cluster display <name>

# 2. 检查 TiKV 日志
tiup cluster log <name> -N <tikv-node>

# 3. 查看 TiKV 监控指标
# Grafana → TiKV-Details
```

| 状态 | 含义 | 处理 |
|------|------|------|
| `Down` | 节点失联 | 检查网络、进程是否存活 |
| `Disconnected` | 临时断开 | 等待自动恢复或重启 TiKV |
| `Offline` | 正在下线 | 等待 Region 迁移完成 |
| `Tombstone` | 已下线 | 可安全移除 |

### 症状：PD 节点异常

- PD 是元数据管理层，**必须保证多数派存活**
- 3 节点 PD 集群最多容忍 1 个节点故障
- 5 节点 PD 集群最多容忍 2 个节点故障

```bash
# 查看 PD 成员状态
tiup ctl:v8.5.6 pd -u http://<pd>:2379 member list

# 查看 PD Leader
tiup ctl:v8.5.6 pd -u http://<pd>:2379 member leader_show

# 若 Leader 异常，手动迁移
# （集群会自动选举新 Leader，通常无需手动干预）
```

## SQL Plan Replayer 保存现场

```sql
-- 保存问题 SQL 的执行计划和环境信息
PLAN REPLAYER DUMP EXPLAIN ANALYZE 
SELECT * FROM t WHERE ...;

-- 导出为文件后发给支持团队分析
-- 或导入到其他 TiDB 集群复现
PLAN REPLAYER LOAD 'replayer_zip_file';
```

## Clinic 远程诊断

```bash
# 收集集群诊断数据
tiup clinic diag collect <cluster-name>

# 上传数据到 PingCAP 支持平台（可选）
tiup clinic diag upload <diag-folder>
```

## 参考资料

- 当用户需要问题排查导图时，加载 `references/troubleshooting-map.md`
- 当用户需要错误码速查时，加载 `references/error-code-index.md`
- 当用户需要告警处理指南时，加载 `references/alert-handling-guide.md`