---
name: tidb-deploy-ops
description: >
  TiDB v8.5 集群部署、升级、扩缩容及日常运维操作。
  当用户涉及：1) TiUP 部署或拓扑配置；2) 版本升级路径规划；
  3) 扩容/缩容 TiKV/TiDB/TiFlash/PD；4) 集群配置变更；
  5) TiProxy/TiCDC 组件部署；6) 集群启停、重启、销毁时触发。
  不回答 Linux 基础问题。
---

# TiDB v8.5 部署与运维操作

## 部署工作流

### 环境检查清单（部署前必须完成）

| 检查项 | 要求 | 验证命令 |
|--------|------|---------|
| OS 版本 | CentOS 7.9+ / Rocky Linux 8 / Ubuntu 20.04+ | `cat /etc/os-release` |
| CPU | x86_64 支持 AVX2；ARM64 支持 ARMv8+crc32+asimd | `grep avx2 /proc/cpuinfo` |
| 内存 | TiKV 节点最低 32GB，推荐 64GB+ | `free -h` |
| 磁盘 | TiKV 数据目录必须是**独立 SSD/NVMe**，禁止与系统盘混用 | `lsblk` |
| 网络 | 万兆以太网（10GbE）最低要求 | `ethtool eth0` |
| NTP | 所有节点时间同步误差 < 50ms | `chronyc sources` |
| 透明大页 | 必须关闭 | `cat /sys/kernel/mm/transparent_hugepage/enabled` → `never` |
| 防火墙 | 端口 2379/2380/20160/20161/4000/10080/9090 等需互通 | `ss -tlnp` |
| 文件描述符 | >= 100000 | `ulimit -n` |

### 部署方式选择

| 场景 | 方式 | 命令 |
|------|------|------|
| 生产环境（裸金属/VM） | TiUP cluster 部署 | `tiup cluster deploy <name> v8.5.0 topology.yaml` |
| K8s 环境 | TiDB Operator | 本 Skill 不深入，引导至官方文档 |
| 测试环境（本地快速体验） | TiUP playground | `tiup playground v8.5.0` |
| 最小生产拓扑 | TiUP cluster（2 TiDB + 3 TiKV + 3 PD） | 参考 topology 模板 |

### 部署后验证

```bash
# 1. 查看集群状态，确认所有节点 Status 为 Up
tiup cluster display <name>

# 2. 集群健康检查
tiup cluster check <name> --cluster

# 3. SQL 层确认版本一致
mysql -u root -P 4000 -h <tidb-host> -e "SELECT * FROM information_schema.cluster_info"

# 4. 确认无异常告警
# 访问 Grafana http://<prometheus>:9090，查看 Overview 面板
```

## 升级工作流（高风险操作）

### 前置检查（v8.5 强制要求）

- [ ] **版本路径检查**：v8.5 不再支持从 < v6.2.0 直接升级
  - 当前 < v6.2.0 → 必须先升级到 [v6.2.0, v8.5.0) 区间版本
- [ ] **无正在执行的 DDL**：`ADMIN SHOW DDL JOBS` 中无 running/pausing 状态
- [ ] **tiup cluster check <name> --cluster** 通过
- [ ] **确认初始化参数兼容**：特别是 `new_collations_enabled_on_first_bootstrap`

### 强制备份（升级前必须执行）

```bash
# 全量备份
br backup full --pd <pd-addr> --storage "s3://bucket/backup"

# 导出拓扑
tiup cluster edit-config <name> > topology-backup.yaml

# 记录当前状态
tiup cluster display <name> > status-before.txt
```

### 滚动升级命令

```bash
# 在线滚动升级（推荐，业务不中断）
tiup cluster upgrade <name> v8.5.6

# 默认顺序：PD → TiKV → TiDB → TiFlash，每节点等待 Region Leader 迁移

# 维护窗口内可加 --offline 加速（会中断业务）
tiup cluster upgrade <name> v8.5.6 --offline

# 仅升级指定组件
tiup cluster upgrade <name> v8.5.6 --tiflash only
```

### 升级后验证

```bash
# 1. 版本一致
tiup cluster display <name>

# 2. SQL 层确认
mysql -e "SELECT tidb_version()"

# 3. 核心查询对比执行计划（关注 IndexMerge、MPP 等新特性）
mysql -e "EXPLAIN ANALYZE SELECT ..."

# 4. Grafana 检查是否有异常告警
```

## 扩缩容指令

| 操作 | 命令 | 风险注意 |
|------|------|---------|
| 扩容 TiKV | `tiup cluster scale-out <name> scale-out.yaml` | 新 TiKV 默认无数据，PD 自动调度 Region，期间 IO 增长 |
| 缩容 TiKV | `tiup cluster scale-in <name> -N <node>` | **必须先确认 Region Leader 已迁移**，否则数据不可用 |
| 扩容 TiFlash | `tiup cluster scale-out <name> scale-out.yaml` | TiFlash 需单独配置 `learner_config` |
| 替换 PD | 先 scale-out 新 PD，再 scale-in 旧 PD | 必须保留多数派（2n+1），建议先加后删 |
| 扩容 TiDB | `tiup cluster scale-out <name> scale-out.yaml` | 无状态，可安全扩容 |
| TiProxy 部署 | `tiup cluster scale-out` + tiproxy 拓扑 | TiProxy 是 v7.1+ SQL 层负载均衡组件 |

### 缩容 TiKV 安全检查

```bash
# 1. 确认目标 TiKV 上 Region Leader 数量为 0
tiup ctl:v8.5.6 pd -u http://<pd-addr>:2379 store

# 2. 查看 leader 分布，确认目标节点 leader_count = 0

# 3. 若不为 0，手动迁移 leader
tiup ctl:v8.5.6 pd -u http://<pd-addr>:2379 operator add transfer-leader <region-id> <target-store>

# 4. 确认后再执行 scale-in
tiup cluster scale-in <name> -N <tikv-node>
```

## 配置变更

### 在线修改（无需重启）

```sql
-- 系统变量（全局生效）
SET GLOBAL tidb_distsql_scan_concurrency = 30;

-- TiUP 配置热加载
tiup cluster edit-config <name>  # 修改后执行 reload
tiup cluster reload <name>
```

### 必须重启生效

- TiKV `rocksdb` 相关参数
- `enable-ttl`（且只能在初始化时设置）
- `storage.api-version`（修改会导致数据格式不兼容、启动报错，**严禁修改**）

### 危险操作黑名单

| 操作 | 风险 | 建议 |
|------|------|------|
| 修改 `storage.api-version` | 数据格式不兼容，启动报错 | 严禁在已有数据集群上修改 |
| 修改 TiKV 部署路径 | 数据丢失 | 必须先做完整备份 |
| 直接删除 PD 数据目录 | 元数据丢失 | 使用 tiup cluster 命令操作 |
| 修改 `new_collations_enabled_on_first_bootstrap` | 已有数据集群行为异常 | 仅在新集群初始化时可设置 |

## TiUP 常用运维命令速查

```bash
# 查看集群列表
tiup cluster list

# 查看集群状态
tiup cluster display <name>

# 启动/停止/重启集群
tiup cluster start <name>
tiup cluster stop <name>
tiup cluster restart <name>

# 查看集群配置
tiup cluster edit-config <name>

# 重载配置（不重启）
tiup cluster reload <name>

# 清理日志和临时文件
tiup cluster prune <name>

# 销毁集群（高危！会删除所有数据）
tiup cluster destroy <name>    # 执行前必须二次确认

# 查看操作日志
tiup cluster audit

# SSH 到指定节点
tiup cluster display <name> -N <node>
```

## 参考资料

- 当用户需要完整拓扑模板时，加载 `references/tiup-topology-full-example.yaml`
- 当用户询问端口列表时，加载 `references/port-list.md`
- 当用户询问升级路径时，加载 `references/upgrade-path-matrix.md`