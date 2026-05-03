# TiDB Docs Skills

基于 TiDB v8.5 LTS 官方文档构建的 AI Agent Skill 集合，专为 TiDB 运维、开发、迁移场景设计。

## 作者信息

| | |
|---|---|
| **作者** | Shawn Yan |
| **身份** | TiDB 社区版主 / Oracle ACE / 公众号「少安事务所」主笔 |
| **主页** | [https://shawnyan.cn](https://shawnyan.cn) |
| **文档版本** | TiDB v8.5 LTS（release-8.5） |
| **更新时间** | 2026-05-03 |

## Skill 总览

本项目包含 9 个 Skill，其中 1 个路由入口 + 8 个专项 Skill：

| Skill | 目录 | 覆盖范围 |
|-------|------|---------|
| **tidb-index** | `tidb-index/` | 总入口，意图识别与路由分发 |
| **tidb-deploy-ops** | `tidb-deploy-ops/` | 集群部署、升级、扩缩容、日常运维 |
| **tidb-migrate-sync** | `tidb-migrate-sync/` | 数据迁移工具链（Lightning/DM/TiCDC/Dumpling） |
| **tidb-sql-tuning** | `tidb-sql-tuning/` | 慢查询分析、执行计划、Optimizer Hints |
| **tidb-disaster-recovery** | `tidb-disaster-recovery/` | 备份恢复（BR）、PITR、容灾方案 |
| **tidb-troubleshoot** | `tidb-troubleshoot/` | 集群故障诊断、OOM、热点、锁冲突 |
| **tidb-security** | `tidb-security/` | TLS 配置、权限管理、加密、日志脱敏 |
| **tidb-htap** | `tidb-htap/` | TiFlash HTAP、MPP 查询优化、存算分离 |
| **tidb-mysql-diff** | `tidb-mysql-diff/` | TiDB 与 MySQL 兼容性差异、迁移评估 |

## 项目结构

```
tidb-docs-skills/
├── README.md                          # 本文件
├── tidb-index/
│   └── SKILL.md                       # 路由入口 Skill
├── tidb-deploy-ops/
│   ├── SKILL.md
│   └── references/
│       ├── port-list.md               # 端口列表速查
│       └── tiup-topology-full-example.yaml  # 完整拓扑模板
├── tidb-disaster-recovery/
│   ├── SKILL.md
│   └── references/
│       └── br-cli-commands.md         # BR 命令手册
├── tidb-htap/
│   ├── SKILL.md
│   └── references/
│       └── tiflash-mpp-mode-details.md
├── tidb-migrate-sync/
│   ├── SKILL.md
│   └── references/
│       └── dm-task-config-template.md
├── tidb-mysql-diff/
│   ├── SKILL.md
│   └── references/
│       └── error-code-index.md
├── tidb-security/
│   ├── SKILL.md
│   └── references/
│       └── tls-configuration-full.md
├── tidb-sql-tuning/
│   ├── SKILL.md
│   └── references/
│       └── optimizer-hints-full-list.md
└── tidb-troubleshoot/
    ├── SKILL.md
    └── references/
        └── troubleshooting-map.md
```

## 使用说明

### 安装到 WorkBuddy

将各 Skill 目录拷贝至 WorkBuddy 的 Skill 目录（用户级或项目级均可）：

```bash
# 用户级（推荐，跨项目可用）
cp -r tidb-index ~/.workbuddy/skills/
cp -r tidb-deploy-ops ~/.workbuddy/skills/
cp -r tidb-migrate-sync ~/.workbuddy/skills/
# ... 其他 Skill 同理
```

### 触发方式

推荐优先加载总入口 **tidb-index**，由其根据用户问题自动路由到合适的子 Skill。

也可直接加载某个专项 Skill，适合已知问题场景：

- 部署/升级问题 → `tidb-deploy-ops`
- 数据迁移/同步问题 → `tidb-migrate-sync`
- SQL 慢查询/执行计划 → `tidb-sql-tuning`
- 备份/容灾 → `tidb-disaster-recovery`
- 集群故障排查 → `tidb-troubleshoot`
- 安全加固/权限 → `tidb-security`
- HTAP/TiFlash → `tidb-htap`
- MySQL 迁移兼容性 → `tidb-mysql-diff`

### 注意事项

- 所有 Skill 默认面向 **TiDB v8.5 LTS**（release-8.5），若用户版本不同，请在问题中注明。
- 涉及**高危操作**（删库、缩容 TiKV、升级等）时，Skill 会要求二次确认，请勿跳过。
- 部分 references 文件为延迟加载（按需），Skill 内会明确指引何时加载。

## 贡献

如发现内容错误或需要更新，欢迎通过 [https://shawnyan.cn](https://shawnyan.cn) 联系作者。

---

> 本 Skill 集基于 TiDB 官方文档整理，遵循 [TiDB Documentation License](https://github.com/pingcap/docs-cn/blob/master/LICENSE)。
