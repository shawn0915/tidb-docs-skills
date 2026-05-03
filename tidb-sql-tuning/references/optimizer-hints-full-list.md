# TiDB v8.5 Optimizer Hints 完整列表

## 索引相关

| Hint | 语法 | 说明 |
|------|------|------|
| USE_INDEX | `/*+ use_index(t, idx1, idx2) */` | 强制使用指定索引 |
| IGNORE_INDEX | `/*+ ignore_index(t, idx1) */` | 忽略指定索引 |
| USE_INDEX_MERGE | `/*+ use_index_merge(t, idx1, idx2) */` | 强制使用 IndexMerge |
| NO_INDEX_MERGE | `/*+ no_index_merge(t) */` | 禁用 IndexMerge |
| ORDER_INDEX | `/*+ order_index(t, idx) */` | 利用索引有序性 |

## 存储引擎选择

| Hint | 语法 | 说明 |
|------|------|------|
| READ_FROM_STORAGE | `/*+ read_from_storage(tikv[t]) */` | 强制走 TiKV |
| READ_FROM_STORAGE | `/*+ read_from_storage(tiflash[t]) */` | 强制走 TiFlash |

## Join 相关

| Hint | 语法 | 说明 |
|------|------|------|
| HASH_JOIN | `/*+ hash_join(t1, t2) */` | 强制 Hash Join |
| MERGE_JOIN | `/*+ merge_join(t1, t2) */` | 强制 Merge Join |
| INL_JOIN | `/*+ inl_join(t1, t2) */` | 强制 Index Nested Loop Join |
| INL_HASH_JOIN | `/*+ inl_hash_join(t1, t2) */` | Index NL + Hash Join |
| NO_INDEX_JOIN | `/*+ no_index_join(t1, t2) */` | 禁用 Index Join |
| BROADCAST_JOIN | `/*+ broadcast_join(t1) */` | MPP Broadcast Join（TiFlash） |
| SHUFFLE_JOIN | `/*+ shuffle_join(t1, t2) */` | MPP Shuffle Join（TiFlash） |

## 聚合相关

| Hint | 语法 | 说明 |
|------|------|------|
| HASH_AGG | `/*+ hash_agg() */` | 强制 Hash Aggregate |
| STREAM_AGG | `/*+ stream_agg() */` | 强制 Stream Aggregate |

## 排序相关

| Hint | 语法 | 说明 |
|------|------|------|
| USE_TOJA | `/*+ use_toja() */` | 将 correlated subquery 转为 join |
| NO_TOJA | `/*+ no_toja() */` | 禁用 subquery 转 join |

## 资源限制

| Hint | 语法 | 说明 |
|------|------|------|
| MAX_EXECUTION_TIME | `/*+ max_execution_time(1000) */` | 最大执行时间(ms) |
| MEMORY_QUOTA | `/*+ memory_quota(1 GB) */` | 内存限制 |
| SET_VAR | `/*+ set_var(var=value) */` | 设置会话变量 |

## 其他

| Hint | 语法 | 说明 |
|------|------|------|
| SEMI_JOIN_REWRITE | `/*+ semi_join_rewrite() */` | 开启 semi join 改写 |
| NO_DECORRELATE | `/*+ no_decorrelate() */` | 禁用 decorrelate |
| MERGE | `/*+ merge() */` | 合并 view |
| TIME_RANGE | `/*+ time_range(t, '2024-01-01', '2024-01-02') */` | 指定查询时间范围（用于 TiFlash） |

## 使用示例

```sql
-- 强制 TiFlash MPP 查询
SELECT /*+ read_from_storage(tiflash[o, i]) broadcast_join(o) */ 
  o.order_id, SUM(i.amount)
FROM orders o JOIN order_items i ON o.order_id = i.order_id
WHERE o.order_date > '2026-01-01'
GROUP BY o.order_id;

-- 强制索引 + 内存限制
SELECT /*+ use_index(t, idx_name) memory_quota(512 MB) max_execution_time(5000) */ 
  * FROM t WHERE name = 'xxx';

-- 设置会话变量
SELECT /*+ set_var(tidb_distsql_scan_concurrency=30) */ 
  * FROM large_table WHERE ...;
```

## 注意事项

1. Hints 优先级高于优化器自动选择
2. 多个 Hints 用空格分隔
3. 表别名需与 Hint 中一致
4. Hint 无法生效时会报 warning
5. TiFlash Hints 仅在查询涉及 TiFlash 副本时有效
