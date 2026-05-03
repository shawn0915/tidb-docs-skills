# BR v8.5 命令行完整手册

## 备份命令

### 全量备份

```bash
br backup full \
  --pd "10.0.1.31:2379" \
  --storage "s3://backup-bucket/tidb/full?region=us-west-2" \
  --send-credentials-to-tikv=false \
  --s3.provider="aws" \
  --ratelimit 128 \
  --checksum=true \
  --log-file backup-full.log
```

### 库级备份

```bash
br backup db \
  --db order_db \
  --pd "10.0.1.31:2379" \
  --storage "s3://backup-bucket/tidb/order_db" \
  --send-credentials-to-tikv=false
```

### 表级备份

```bash
br backup table \
  --db order_db \
  --table t_order \
  --pd "10.0.1.31:2379" \
  --storage "s3://backup-bucket/tidb/order_db/t_order"
```

## 恢复命令

### 全量恢复

```bash
br restore full \
  --pd "10.0.1.31:2379" \
  --storage "s3://backup-bucket/tidb/full" \
  --send-credentials-to-tikv=false \
  --checksum=true \
  --log-file restore-full.log
```

### 库级恢复

```bash
br restore db \
  --db order_db \
  --pd "10.0.1.31:2379" \
  --storage "s3://backup-bucket/tidb/order_db"
```

### 表级恢复

```bash
br restore table \
  --db order_db \
  --table t_order \
  --pd "10.0.1.31:2379" \
  --storage "s3://backup-bucket/tidb/order_db/t_order"
```

## 日志备份（PITR）

### 启动日志备份

```bash
br log start \
  --task-name=pitr \
  --pd "10.0.1.31:2379" \
  --storage "s3://log-bucket/tidb/logs?region=us-west-2" \
  --send-credentials-to-tikv=false
```

### 查看日志备份状态

```bash
br log status \
  --task-name=pitr \
  --pd "10.0.1.31:2379"
```

### 停止日志备份

```bash
br log stop \
  --task-name=pitr \
  --pd "10.0.1.31:2379"
```

### PITR 恢复

```bash
br restore point \
  --pd "10.0.1.31:2379" \
  --storage "s3://log-bucket/tidb/logs" \
  --full-backup-storage "s3://backup-bucket/tidb/full" \
  --restored-ts '2026-05-01 12:00:00+0800' \
  --send-credentials-to-tikv=false
```

### 日志压缩

```bash
br log compact \
  --pd "10.0.1.31:2379" \
  --storage "s3://log-bucket/tidb/logs"
```

### 查看可恢复范围

```bash
br log metadata \
  --storage "s3://log-bucket/tidb/logs"
```

## 参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--pd` | PD 地址 | 必填 |
| `--storage` | 存储路径（s3/local/nfs/gcs/azure） | 必填 |
| `--send-credentials-to-tikv` | 是否将凭证发送到 TiKV | false |
| `--ratelimit` | 限速(MB/s) | 0 (不限速) |
| `--checksum` | 是否校验 | true |
| `--concurrency` | 并发数 | 4 |
| `--log-file` | 日志文件路径 | - |
| `--gcttl` | GC TTL（分钟） | 自动计算 |
