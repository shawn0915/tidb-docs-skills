# TiDB v8.5 问题排查导图

## 集群不可用

```
集群不可用
├── 所有 TiDB 节点无法连接
│   ├── 检查 TiDB 进程：ps aux | grep tidb-server
│   ├── 检查端口监听：ss -tlnp | grep 4000
│   ├── 检查 TiDB 日志：tiup cluster log <name> -N <tidb-node>
│   └── 检查资源：top, free -h, df -h
├── 部分 TiDB 节点异常
│   ├── tiup cluster display <name>
│   ├── 检查网络连通性：ping, telnet
│   └── 检查负载均衡器状态
├── TiKV 多数节点异常
│   ├── 检查 TiKV 进程和磁盘
│   ├── 检查 Region 状态：pd-ctl region check
│   └── 若多数节点永久丢失：需从备份恢复
└── PD 多数节点异常
    ├── PD 必须保证多数派（2n+1 中至少 n+1 存活）
    ├── 检查 PD 进程和网络
    └── 若 Leader 异常：自动选举新 Leader
```

## 读写延迟高

```
读写延迟高
├── TiDB 层
│   ├── 解析/编译慢 → 统计信息过期 → ANALYZE TABLE
│   ├── 执行慢 → 执行计划差 → EXPLAIN ANALYZE 优化
│   ├── 等待时间长 → 锁冲突 → DATA_LOCK_WAITS
│   └── Coprocessor 等待 → TiKV 压力大
├── TiKV 层
│   ├── Store duration 高 → 磁盘 IO 瓶颈 → iostat
│   ├── Apply duration 高 → Region 过多 → merge/split
│   └── Coprocessor 慢 → 大范围扫描 → 加索引/TiFlash
└── 网络层
    ├── TiDB-TiKV 网络延迟 → ping, tracepath
    └── 跨区域部署 → 考虑就近访问
```

## OOM 问题

```
OOM (Out of Memory)
├── TiDB OOM
│   ├── tidb_mem_quota_query 太小
│   ├── 大查询未走索引 → EXPLAIN 检查
│   ├── 多表 Join 中间结果大
│   └── 并发查询过多 → 资源组限制
├── TiKV OOM
│   ├── block-cache 配置过大
│   ├── 写入太快 → Raft 日志堆积
│   └── Region 过多 → 调大 region-split-size
└── TiFlash OOM
    ├── 大查询内存占用高
    └── MPP 查询并发太高
```

## 数据不一致

```
数据不一致
├── 主从同步延迟
│   ├── DM：query-status 查看 secondsBehindMaster
│   ├── TiCDC：changefeed list 查看 checkpoint
│   └── 网络带宽不足
├── 数据丢失
│   ├── 检查是否有 DROP/TRUNCATE 误操作
│   ├── 检查 GC TTL 是否过短
│   └── 检查是否有定时任务清理数据
└── 数据重复
    ├── 主键/唯一索引冲突 → 检查应用写入逻辑
    ├── 重复导入数据 → Lightning duplicate-resolution
    └── 双向同步冲突 → BDR 模式配置
```

## 存储空间满

```
存储空间满
├── TiKV 数据目录
│   ├── 检查数据分布：pd-ctl store
│   ├── 清理旧版本数据（GC）
│   ├── 扩容 TiKV 节点
│   └── 删除不必要的历史数据
├── TiFlash 数据目录
│   ├── 减少 TiFlash 副本数
│   ├── 删除不必要的列存副本
│   └── 启用存算分离到 S3
├── 备份存储
│   ├── 删除过期备份
│   ├── 开启日志压缩
│   └── 调整备份保留策略
└── 日志文件
    ├── 清理旧日志
    ├── 调整日志保留天数
    └── 配置日志轮转
```
