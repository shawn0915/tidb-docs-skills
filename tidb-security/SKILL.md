---
name: tidb-security
description: >
  TiDB v8.5 安全加固、TLS 配置、权限管理与审计。
  当用户涉及：1) 开启 TLS/加密传输；2) 静态加密（TDE）；
  3) 角色权限配置；4) 密码策略；5) 列级权限/掩码策略；
  6) 证书鉴权；7) 日志脱敏；8) 安全合规加固时触发。
author: "Shawn Yan"
author_url: "https://shawnyan.cn"
author_role:
  - "TiDB 社区版主"
  - "Oracle ACE"
  - "公众号「少安事务所」主笔"
doc_version: "TiDB v8.5 LTS"

# TiDB v8.5 安全加固与权限管理

## TLS 配置层级

| 层级 | 覆盖范围 | 配置方式 | 优先级 |
|------|---------|---------|--------|
| 客户端-TiDB | 应用连接加密 | `ssl-ca/ssl-cert/ssl-key` 连接参数 | 高 |
| 组件间通信 | TiDB-TiKV-PD 内部加密 | `tiup cluster tls` 一键开启 | 中 |
| 备份存储 | BR 备份到 S3 加密 | 服务端加密 SSE-S3 / SSE-KMS | 中 |

### 一键开启组件间 TLS

```bash
# TiUP 部署时自动配置 TLS
tiup cluster deploy <name> v8.5.6 topology.yaml --enable_tls

# 已有集群开启 TLS（需滚动重启）
tiup cluster tls <name> enable
```

### 客户端 TLS 连接

```bash
# mysql 客户端使用 TLS
mysql -u root -h <tidb-host> -P 4000 \
  --ssl-ca=ca.pem \
  --ssl-cert=client-cert.pem \
  --ssl-key=client-key.pem \
  --ssl-mode=VERIFY_CA
```

## 静态加密（TDE）

### TiKV 数据加密

```toml
# tikv.toml
[security.encryption]
method = "aes256-gcm"  # aes256-gcm | sm4-ctr
data-key-rotation-period = "7d"

[security.encryption.master-key]
# 方案1：本地文件（测试环境）
type = "file"
path = "/path/to/master-key"

# 方案2：AWS KMS
# type = "kms"
# key-id = "arn:aws:kms:..."

# 方案3：HashiCorp Vault
# type = "vault"
# ...
```

### TiFlash 数据加密

- 同样支持 RocksDB 层加密
- 配置方式与 TiKV 类似

### 密钥管理方案

| 方案 | 适用场景 | 配置复杂度 |
|------|---------|-----------|
| 本地文件 | 测试/开发环境 | 低 |
| AWS KMS | AWS 部署 | 中 |
| HashiCorp Vault | 企业级密钥管理 | 高 |

## 权限管理（v8.5 新增）

### 列级权限

```sql
-- 授予特定列的查询权限
GRANT SELECT(id, name, email) ON db.users TO 'app_user'@'%';

-- 查看列级权限
SHOW GRANTS FOR 'app_user'@'%';
```

### 列级掩码策略（动态数据脱敏）

```sql
-- 创建掩码策略（v8.5 新增）
CREATE MASKING POLICY phone_mask 
AS (phone VARCHAR(20)) 
RETURN (CASE 
    WHEN CURRENT_USER() = 'admin@%' THEN phone
    ELSE CONCAT(LEFT(phone, 3), '****', RIGHT(phone, 4))
END);

-- 应用策略到列
ALTER TABLE db.users 
ALTER COLUMN phone 
SET MASKING POLICY phone_mask;
```

### 密码管理

```sql
-- 密码过期策略
CREATE USER 'app_user'@'%' 
IDENTIFIED BY 'StrongP@ssw0rd' 
PASSWORD EXPIRE INTERVAL 90 DAY;

-- 密码历史限制（禁止重复使用最近 N 次密码）
SET GLOBAL password_history = 6;

-- 密码强度检查
SET GLOBAL validate_password.enable = ON;
SET GLOBAL validate_password.length = 12;
SET GLOBAL validate_password.mixed_case_count = 1;
SET GLOBAL validate_password.number_count = 1;
SET GLOBAL validate_password.special_char_count = 1;
```

### 基于角色的访问控制 (RBAC)

```sql
-- 创建角色
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- 授予权限
GRANT SELECT ON db.* TO 'app_read';
GRANT SELECT, INSERT, UPDATE ON db.* TO 'app_write';
GRANT ALL ON db.* TO 'app_admin';

-- 角色授权给用户
GRANT 'app_read' TO 'app_user'@'%';
SET DEFAULT ROLE 'app_read' TO 'app_user'@'%';
```

### 证书鉴权（X.509）

```sql
-- 使用证书替代密码
CREATE USER 'secure_user'@'%' 
REQUIRE SUBJECT '/CN=secure_user/O=org'
ISSUER '/CN=TiDB CA'
CIPHER 'ECDHE-RSA-AES128-GCM-SHA256';
```

## 日志脱敏

```sql
-- 开启 SQL 日志脱敏（生产环境建议开启）
SET GLOBAL tidb_redact_log = 1;

-- 效果：SQL 中的敏感数据（密码、身份证号等）在日志中显示为 ?
-- 注意：开启后日志排查难度增加，建议生产开启、测试关闭
```

## 安全配置检查清单

### 生产环境必须项

- [ ] 组件间 TLS 开启
- [ ] 客户端连接 TLS 强制要求
- [ ] 静态加密（TDE）开启
- [ ] 密码强度策略配置
- [ ] 密码过期策略配置
- [ ] 日志脱敏开启
- [ ] 非 root 账户运行 TiDB
- [ ] 防火墙规则配置（最小权限）
- [ ] 审计日志开启
- [ ] 定期安全扫描

### 网络隔离

| 组件 | 需要开放的端口 | 访问来源 |
|------|--------------|---------|
| TiDB | 4000 (SQL), 10080 (HTTP) | 应用服务器 |
| TiKV | 20160 (gRPC), 20161 (HTTP) | TiDB 节点, TiKV 节点 |
| PD | 2379 (客户端), 2380 (peer) | TiDB 节点, TiKV 节点, PD 节点 |
| TiFlash | 9000 (MPP), 8123 (HTTP), 9004 (Raft), 20170, 20292 | TiDB 节点, TiFlash 节点 |
| Grafana | 3000 | 运维人员 |
| Prometheus | 9090 | 运维人员 |

## 参考资料

- 当用户需要完整 TLS 配置指南时，加载 `references/tls-configuration-full.md`
- 当用户需要权限管理详细说明时，加载 `references/privilege-management-details.md`
- 当用户需要审计日志配置时，加载 `references/audit-log-setup.md`