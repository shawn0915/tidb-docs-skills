---
name: tidb-htap
description: >
  TiDB v8.5 TiFlash HTAP 架构、MPP 查询优化与实时分析。
  当用户涉及：1) TiFlash 副本创建与管理；2) MPP 模式查询；
  3) TiFlash 存算分离/S3 架构；4) FastScan / 延迟物化；
  5) 查询结果物化；6) HTAP 场景设计与性能调优；
  7) TiFlash 与 TiKV 的选择时触发。
author: "Shawn Yan"
author_url: "https://shawnyan.cn"
author_role:
  - "TiDB 社区版主"
  - "Oracle ACE"
  - "公众号「少安事务所」主笔"
doc_version: "TiDB v8.5 LTS"

# TiDB v8.5 TiFlash HTAP 与实时分析

## TiFlash 核心概念

- **列存引擎**：基于 ClickHouse 改进，支持向量化执行
- **MPP 模式**：大规模并行处理，多 TiFlash 节点协同计算
- **存算分离**：v7.0+ 支持，TiFlash 计算节点 + S3 存储（降低成本）
- **强一致性读取**：通过 Raft 校对索引 + MVCC 实现 Snapshot Isolation
- **工作负载隔离**：TiFlash 推荐独立节点，与 TiKV 物理隔离

### 架构特点

- TiFlash 以 **Raft Learner** 角色接入集群
- 数据从 TiKV 异步复制到 TiFlash
- 读取时通过 Raft 校对索引保证一致性
- 任何数据必须先写入 TiKV，再同步到 TiFlash

### 硬件要求

```bash
# x86_64：必须支持 AVX2
grep avx2 /proc/cpuinfo

# ARM64：必须支持 ARMv8 + crc32 + asimd
grep 'crc32' /proc/cpuinfo | grep 'asimd'
```

## 副本管理

### 创建 TiFlash 副本

```sql
-- 为表创建 1 个 TiFlash 副本
ALTER TABLE orders SET TIFLASH REPLICA 1;

-- 为表创建 2 个 TiFlash 副本（多 TiFlash 节点时）
ALTER TABLE large_table SET TIFLASH REPLICA 2;

-- 查看同步进度
SELECT * FROM information_schema.tiflash_replica 
WHERE table_schema = 'db_name';

-- 关键字段：
-- REPLICA_COUNT: 目标副本数
-- LOCATION_LABELS: 放置策略
-- AVAILABLE: 1 表示可用，0 表示同步中
-- PROGRESS: 同步进度 (0~1)
```

### 删除 TiFlash 副本

```sql
-- 删除副本（释放 TiFlash 空间）
ALTER TABLE orders SET TIFLASH REPLICA 0;
```

### 副本放置策略

```sql
-- 使用 Placement Rules 控制副本分布
ALTER TABLE orders SET TIFLASH REPLICA 1
PLACEMENT POLICY zone1;

-- 跨 AZ 部署
CREATE PLACEMENT POLICY zone1 
PRIMARY_ZONE = "zone1" 
FOLLOWER_ZONES = "zone2,zone3";
```

## MPP 查询优化

### 自动选择

TiDB 优化器自动判断走 TiKV（行存）还是 TiFlash（列存）：

```sql
-- 优化器自动选择
SELECT COUNT(*), SUM(amount) FROM orders WHERE order_date > '2026-01-01';
-- 通常大表聚合查询会选择 TiFlash
```

### 强制走 TiFlash

```sql
-- 使用 Optimizer Hint 强制 TiFlash
SELECT /*+ read_from_storage(tiflash[orders]) */ 
  COUNT(*), SUM(amount) 
FROM orders 
WHERE order_date > '2026-01-01';
```

### 强制走 TiKV

```sql
-- 使用 Optimizer Hint 强制 TiKV
SELECT /*+ read_from_storage(tikv[orders]) */ 
  * FROM orders WHERE id = 123;
-- 点查通常走 TiKV 更优
```

### MPP Join 策略

| Join 类型 | 适用场景 | 说明 |
|-----------|---------|------|
| Broadcast Join | 小表 Join 大表 | 小表广播到各 TiFlash 节点 |
| Shuffle Join (Hash Join) | 大表 Join 大表 | 按 Join Key 重新分布数据 |
| Shuffle Hash Join | 双方数据量都大 | 两方都按 Hash 分区 |

```sql
-- 查看执行计划中的 Join 类型
EXPLAIN 
SELECT /*+ read_from_storage(tiflash[t1, t2]) */ *
FROM t1 JOIN t2 ON t1.id = t2.id;
-- 关注 ExchangeSender/ExchangeReceiver 节点
```

## v8.5 新特性

### TiFlash Pipeline Model

- 新的执行模型，减少线程切换开销
- 默认启用
- 对复杂查询性能提升明显

### 延迟物化（Late Materialization）

- 先过滤再读取列数据，减少 IO
- 对高选择率查询（过滤后数据少）效果显著
- 自动启用，无需配置

### 查询结果物化

```sql
-- 将 TiFlash 查询结果持久化，供后续查询复用
-- 减少重复计算
CREATE MATERIALIZED VIEW mv_summary 
AS 
SELECT region, SUM(sales), COUNT(*) 
FROM sales_data 
GROUP BY region;

-- 后续查询直接查物化视图
SELECT * FROM mv_summary WHERE region = 'east';
```

### MinTSO 调度器

- 优化读请求调度，减少等待时间
- 自动工作，无需配置

### FastScan

```sql
-- 开启 FastScan（牺牲强一致性，提升查询速度）
ALTER TABLE t SET TIFLASH MODE FAST;

-- 适用场景：可以接受轻微延迟的分析查询
-- 不适用：要求强一致性的实时查询
```

## 存算分离架构（v7.0+）

```
架构：
  TiFlash Compute Node（计算） ←──→ S3/OSS（存储）
         ↑                                ↑
    向量计算引擎                  列存数据文件
    MPP 查询处理                  低成本对象存储
```

### 优势

- **降低成本**：S3 存储比本地 SSD 便宜 70%+
- **弹性扩展**：计算节点可随时扩缩容
- **冷数据首次读取延迟**：从 S3 加载，约 100ms 级

### 适用场景

- 历史数据分析（冷数据为主）
- 不敏感于亚秒级延迟的报表
- 成本敏感型业务

## 常见陷阱

| 陷阱 | 说明 | 解决方案 |
|------|------|---------|
| 部分函数不支持 | TiFlash 不支持某些 JSON 函数、窗口函数 | 自动回退到 TiDB 执行，可能慢 |
| 副本同步延迟 | `tiflash_replica` 修改后不会立即生效 | 查询 `tiflash_replica` 表确认进度 |
| 冷数据延迟 | 存算分离模式下冷数据首次读取有延迟 | 预热或改用本地存储 |
| 写入必须先写 TiKV | TiFlash 不接受直接写入 | 正常写入流程即可 |
| 小表点查走 TiFlash 反而慢 | 行存点查 TiKV 更优 | 让优化器自动选择，或强制 Hint |

## TiFlash vs TiKV 选择指南

| 场景 | 推荐存储 | 原因 |
|------|---------|------|
| 主键/唯一索引点查 | TiKV | 行存，索引高效 |
| 范围查询（小数据量） | TiKV | 行存，回表快 |
| 聚合查询（COUNT/SUM/AVG） | TiFlash | 列存，向量化执行 |
| 大表 Join | TiFlash (MPP) | 分布式并行处理 |
| 实时报表 | TiFlash | 列存压缩率高，扫描快 |
| 有事务要求的写入 | TiKV | TiFlash 只读 |
| 需要最新数据的查询 | TiKV | TiFlash 有微小同步延迟 |

## 参考资料

- 当用户需要 MPP 模式详细说明时，加载 `references/tiflash-mpp-mode-details.md`
- 当用户需要存算分离配置时，加载 `references/tiflash-disaggregated-config.md`
- 当用户需要 TiFlash 兼容性说明时，加载 `references/tiflash-compatibility.md`