# TiFlash MPP 模式详解

## MPP 架构

```
查询流程：
  TiDB (SQL 解析/优化)
    ↓ 生成 MPP 执行计划
  TiFlash MPP Coordinator
    ↓ 分发任务
  ┌─────────────┬─────────────┬─────────────┐
  │ TiFlash-1   │ TiFlash-2   │ TiFlash-3   │
  │ Exchange    │ Exchange    │ Exchange    │
  │ HashAgg     │ HashAgg     │ HashAgg     │
  │ HashJoin    │ HashJoin    │ HashJoin    │
  │ TableScan   │ TableScan   │ TableScan   │
  └─────────────┴─────────────┴─────────────┘
         ↓              ↓            ↓
      ExchangeSender/Receiver (Shuffle/Broadcast)
         ↓
  TiDB (结果汇总)
```

## MPP 执行计划特征

```sql
EXPLAIN SELECT COUNT(*), region FROM sales GROUP BY region;

-- MPP 模式执行计划包含：
-- ExchangeSender: 将数据发送到其他 TiFlash 节点
-- ExchangeReceiver: 接收其他节点数据
-- ExchangeType: Hash/Broadcast/Passthrough
```

## Exchange 类型

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| Hash | 按 Hash 值重新分布数据 | Join/Aggregate 需要相同 Key 的数据 |
| Broadcast | 广播小表到所有节点 | 小表 Join 大表 |
| Passthrough | 直接传递，不重新分布 | 最终汇总 |

## MPP 触发条件

1. 查询涉及 TiFlash 副本的表
2. 优化器判断 MPP 更优（或强制 Hint）
3. 查询类型支持：Aggregate、Join、Sort、Limit

## MPP 性能优化

### 1. 确保数据分布均匀

```sql
-- 检查 Region 分布
SELECT STORE_ID, PEER_ID, ROLE, REGION_ID 
FROM information_schema.tiflash_replica;

-- 不均匀的分布会导致数据倾斜
```

### 2. 选择合适的 Join 策略

```sql
-- Broadcast Join（小表广播）
SELECT /*+ broadcast_join(small_table) */ *
FROM large_table l JOIN small_table s ON l.id = s.id;

-- Shuffle Join（大表重分布）
SELECT /*+ shuffle_join(large_table1, large_table2) */ *
FROM large_table1 t1 JOIN large_table2 t2 ON t1.id = t2.id;
```

### 3. 减少 Shuffle 数据量

- 过滤条件下推：WHERE 条件尽量在 TableScan 层处理
- 列裁剪：只查询需要的列
- 预聚合：利用物化视图减少计算

### 4. 资源管控

```sql
-- 为分析查询创建专用资源组
CREATE RESOURCE GROUP analytics
  RU_PER_SEC = 10000
  BURSTABLE
  QUERY_LIMIT=(EXEC_ELAPSED='10m', ACTION=KILL);
```

## MPP 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 未走 MPP | 表没有 TiFlash 副本 | ALTER TABLE ... SET TIFLASH REPLICA 1 |
| MPP 比 TiKV 慢 | 数据量小，MPP 开销大 | 让优化器自动选择或强制 TiKV |
| Exchange 耗时高 | 网络带宽不足 | 提升网络或减少 Shuffle |
| 数据倾斜 | 某些 Key 数据量大 | 调整分布键或预处理 |
| OOM | 中间结果过大 | 加过滤条件、分批处理 |

## MPP 监控指标

```sql
-- 查询是否走了 MPP
EXPLAIN ANALYZE SELECT ...;
-- 关注是否有 ExchangeSender/Receiver 算子

-- MPP 任务耗时
SELECT * FROM information_schema.tiflash_segments;

-- Grafana 面板
-- TiFlash-Summary → MPP Task Duration
-- TiFlash-Summary → Exchange Duration
```
