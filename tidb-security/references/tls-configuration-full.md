# TiDB v8.5 TLS 完整配置指南

## 组件间 TLS（一键开启）

### 新集群部署时开启

```bash
tiup cluster deploy <name> v8.5.6 topology.yaml --enable_tls
```

### 已有集群开启 TLS

```bash
# 1. 生成证书
tiup cluster tls <name> generate-certs

# 2. 分发证书并重启集群
tiup cluster tls <name> enable

# 3. 验证 TLS 状态
tiup cluster display <name>
```

### 手动证书管理

```bash
# 使用 cfssl 生成 CA 和组件证书
# 1. 创建 CA
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# 2. 为 TiDB 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=internal \
  tidb-server.json | cfssljson -bare tidb-server

# 3. 为 TiKV 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=internal \
  tikv-server.json | cfssljson -bare tikv-server

# 4. 为 PD 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem \
  -config=ca-config.json -profile=internal \
  pd-server.json | cfssljson -bare pd-server
```

## 客户端 TLS

### 强制客户端使用 TLS

```toml
# tidb.toml
[security]
require-secure-transport = true
ssl-ca = "/path/to/ca.pem"
ssl-cert = "/path/to/tidb-server.pem"
ssl-key = "/path/to/tidb-server-key.pem"
```

### 客户端连接

```bash
# MySQL 客户端
mysql -h tidb-host -P 4000 -u root \
  --ssl-mode=VERIFY_CA \
  --ssl-ca=ca.pem \
  --ssl-cert=client-cert.pem \
  --ssl-key=client-key.pem

# JDBC 连接串
jdbc:mysql://tidb-host:4000/test?
  useSSL=true&
  requireSSL=true&
  verifyServerCertificate=true&
  trustCertificateKeyStoreUrl=file:ca.jks&
  clientCertificateKeyStoreUrl=file:client.jks

# Go MySQL Driver
dsn := "root@tcp(tidb-host:4000)/test?tls=custom&sslca=ca.pem&sslcert=client-cert.pem&sslkey=client-key.pem"
```

## TiFlash TLS

TiFlash 复用集群的证书体系，部署时自动配置。

## TiCDC TLS

```yaml
# changefeed 配置中指定 TLS
cdc cli changefeed create \
  --server=https://cdc-host:8300 \
  --sink-uri="mysql://user:password@downstream:3306/?ssl-ca=ca.pem&ssl-cert=client.pem&ssl-key=client-key.pem"
```

## 证书轮换

```bash
# 1. 生成新证书
cfssl gencert -renewca -ca ca.pem -ca-key ca-key.pem | cfssljson -bare ca

# 2. 滚动更新各组件证书
tiup cluster reload <name> --role tidb
tiup cluster reload <name> --role tikv
tiup cluster reload <name> --role pd

# 注意：证书轮换期间会有短暂连接中断
```

## 证书有效期监控

```bash
# 检查证书过期时间
openssl x509 -in tidb-server.pem -noout -dates

# Prometheus 告警规则
- alert: TiDBTLSCertExpiresSoon
  expr: (tidb_security_certificate_expiration_seconds - time()) / 86400 < 30
  for: 1h
  labels:
    severity: warning
  annotations:
    summary: "TiDB TLS certificate expires in less than 30 days"
```
