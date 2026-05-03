---
name: tidb-disaster-recovery
description: >
  TiDB v8.5 备份恢复（BR）、PITR 及容灾方案设计与执行。
  当用户涉及：1) 全量/增量备份策略；2) 时间点恢复（PITR）；
  3) 跨地域容灾架构；4) BR 命令使用；5) 备份存储配置；
  6) 日志备份开启与管理；7) 容灾切换演练时触发。
author: "Shawn Yan"
author_url: "https://shawnyan.cn"
author_role:
  - "TiDB 社区版主"
  - "Oracle ACE"
  - "公众号「少安事务所」主笔"
doc_version: "TiDB v8.5 LTS"

# TiDB v8.5 备份恢复与容灾

## 容灾方案选型

| 方案 | RTO | RPO | 成本 | 适用场景 |
|------|-----|-----|------|---------|
| 主备集群（TiCDC 同步） | 分钟级 | 秒级 | 高（双集群） | 金融级核心业务 |
| 多副本单集群（跨 AZ） | 秒级 | 0 | 中（跨 AZ 部署） | 同城容灾 |
| 备份恢复（BR + PITR） | 小时级 | 分钟级 | 低（对象存储） | 中小业务、合规备份 |

### 选择建议

- **同城容灾**：多副本单集群 + Placement Rules（Label 感知调度）
- **异地容灾**：主备集群（TiCDC 同步）+ BR 兜底
- **备份合规**：BR 快照备份 + 日志备份（PITR）

## BR 快照备份（最常用）

### 全量备份

```bash
# 备份到 S3
br backup full \
  --pd <pd-addr>:2379 \
  --storage "s3://bucket/backup-folder?region=us-west-2" \
  --send-credentials-to-tikv=false \
  --ratelimit 128 \
  --log-file backup.log

# 备份到本地 NFS
br backup full \
  --pd <pd-addr>:2379 \
  --storage "local:///nfs/backup" \
  --log-file backup.log
```

### 指定库表备份

```bash
# 备份指定数据库
br backup db --db test \
  --pd <pd-addr>:2379 \
  --storage "s3://bucket/backup"

# 备份指定表
br backup table --db test --table t1 \
  --pd <pd-addr>:2379 \
  --storage "s3://bucket/backup"
```

### 备份性能影响

- 推荐在**业务低峰期**执行快照备份
- 备份会占用 TiKV 磁盘 IO 和 CPU（约 10-30%）
- **不推荐同时运行多个备份任务**
- 可通过 `--ratelimit` 限制备份速度

## 日志备份与 PITR（v8.5 增强）

### 开启日志备份

```bash
# 启动日志备份（后台持续运行）
br log start \
  --task-name=pitr \
  --pd <pd-addr>:2379 \
  --storage "s3://log-bucket/log-folder?region=us-west-2"

# 查看日志备份状态
br log status --task-name=pitr --pd <pd-addr>:2379
```

### 快照恢复

```bash
# 恢复到全新空集群
br restore full \
  --pd <pd-addr>:2379 \
  --storage "s3://bucket/backup-folder"

# 恢复指定库表
br restore table --db test --table t1 \
  --pd <pd-addr>:2379 \
  --storage "s3://bucket/backup-folder"
```

### PITR 时间点恢复

```bash
# 恢复到指定时间点
br restore point \
  --pd <pd-addr>:2379 \
  --storage "s3://log-bucket/log-folder" \
  --full-backup-storage "s3://bucket/backup-folder" \
  --restored-ts '2026-05-01 12:00:00+0800'

# 获取可用恢复范围
br log metadata \
  --storage "s3://log-bucket/log-folder"
```

### 日志压缩（v8.5 增强）

```bash
# 压缩历史日志备份，减少存储成本
br compact \
  --pd <pd-addr>:2379 \
  --storage "s3://log-bucket/log-folder"
```

### 断点备份/恢复

- v8.5 支持**断点续传**
- 大集群备份中断后可从中断点继续
- 无需从头开始

## 关键限制

| 限制 | 说明 |
|------|------|
| 恢复目标 | BR 恢复时**目标集群必须为空**（或指定库表恢复） |
| 系统表 | PITR **不支持恢复系统表中用户表和权限表**的数据 |
| 并发备份 | **不支持**在一个集群上同时运行多个数据备份任务 |
| 备份恢复冲突 | 不建议备份正在恢复的表，数据可能异常 |
| PITR 冲突 | PITR 恢复期间不支持同时运行日志备份和 TiCDC 同步 |
| TiFlash | BR **默认不备份 TiFlash 副本**，恢复后需重新添加 |
| Lightning | 物理导入模式期间**不能同时运行日志备份** |

## 容灾演练检查清单

### 备份有效性验证

- [ ] 定期执行恢复演练（每季度至少一次）
- [ ] 验证备份数据完整性
- [ ] 确认恢复时间（RTO）满足 SLA
- [ ] 验证 PITR 恢复精度

### 监控告警

```yaml
# Prometheus 告警规则建议
- alert: BackupTaskFailed
  expr: increase(tidb_backup_restore_result{status="fail"}[1h]) > 0
  for: 5m
  
- alert: LogBackupCheckpointStale
  expr: time() - tidb_log_backup_checkpoint_ts > 3600
  for: 5m
```

## 参考资料

- 当用户需要 BR 完整命令手册时，加载 `references/br-cli-commands.md`
- 当用户需要版本兼容性矩阵时，加载 `references/br-version-compatibility.md`
- 当用户需要容灾方案详细设计时，加载 `references/dr-solution-design.md`