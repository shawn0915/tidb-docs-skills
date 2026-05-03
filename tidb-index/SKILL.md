---
name: tidb-index
description: >
  TiDB v8.5 LTS 官方文档知识库总入口。
  当用户询问任何与 TiDB 相关的问题时，先调用此 Skill 进行意图识别和子 Skill 路由。
  负责确认用户 TiDB 版本、部署环境（TiUP/K8s/Cloud），并分发给下游专业 Skill。
  不直接回答技术问题，只做路由分发和版本锚定。
author: "Shawn Yan"
author_url: "https://shawnyan.cn"
author_role:
  - "TiDB 社区版主"
  - "Oracle ACE"
  - "公众号「少安事务所」主笔"
doc_version: "TiDB v8.5 LTS"

# TiDB v8.5 文档知识库总入口

## 版本锚定规则

- **默认版本**：TiDB v8.5 LTS（release-8.5）
- **版本阻断**：若用户版本 < v6.2.0，必须先阻断 —— v8.5 不再支持从 v6.2.0 以下直接升级
- **版本确认**：首次交互时确认用户当前 TiDB 版本，若未声明则默认按 v8.5 回答

## 子 Skill 路由表

| 子 Skill | 触发场景 | 关键词 |
|---------|---------|--------|
| `tidb-mysql-diff` | MySQL 迁移兼容性、语法差异、字符集问题 | "从 MySQL 迁移","MySQL 能跑 TiDB 报错","外键","自增列","存储过程" |
| `tidb-deploy-ops` | 部署、升级、扩缩容、TiUP 拓扑配置 | "部署集群","升级版本","扩容 TiKV","缩容","TiUP" |
| `tidb-migrate-sync` | 数据迁移工具选型、DM、TiCDC、Lightning、Dumpling | "数据迁移","DM","TiCDC","同步到 Kafka","Lightning 导入" |
| `tidb-sql-tuning` | 慢查询、执行计划、索引优化、Optimizer Hints | "SQL 慢","执行计划","EXPLAIN","索引优化","慢查询" |
| `tidb-disaster-recovery` | 备份恢复、BR、PITR、容灾方案 | "备份","恢复","PITR","容灾","BR" |
| `tidb-troubleshoot` | 集群故障、节点宕机、OOM、热点、锁冲突 | "OOM","节点挂了","延迟高","锁冲突","报错" |
| `tidb-security` | TLS、加密、权限管理、审计 | "TLS","加密","权限","密码策略","列级权限" |
| `tidb-htap` | TiFlash、MPP、实时分析、存算分离 | "TiFlash","HTAP","MPP","列存","实时分析" |

## 路由决策流程

1. 识别用户意图中的关键词（上表）
2. 若匹配多个 Skill，选择最具体的那个
3. 若无法确定，先询问用户具体问题场景

## 全局禁忌

- **不提供** 通用 SQL 教学（如"什么是 JOIN"、"怎么写子查询"）
- **不提供** Linux 基础运维指导（如"怎么配 SSH 免密"、"怎么装 CentOS"）
- **必须二次确认** 涉及以下高风险操作：
  - 删库（DROP DATABASE/TABLE）
  - 销毁集群（tiup cluster destroy）
  - 强制降级（ downgrade ）
  - 缩容 TiKV（数据迁移风险）

## 环境确认清单

首次处理运维类问题时，确认以下信息：

1. TiDB 版本：`SELECT tidb_version()`
2. 部署方式：TiUP / TiDB Operator (K8s) / TiDB Cloud
3. 集群规模：TiDB/TiKV/PD/TiFlash 节点数
4. 是否有正在执行的 DDL：`ADMIN SHOW DDL JOBS`

## 元数据

```yaml
metadata:
  doc_version: "TiDB v8.5 LTS"
  doc_branch: "release-8.5"
  last_sync: "2026-05-02"
  skill_count: 8
  coverage: "部署/迁移/调优/备份/故障/安全/HTAP/MySQL兼容"
```