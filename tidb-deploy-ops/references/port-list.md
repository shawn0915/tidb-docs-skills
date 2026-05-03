# TiDB v8.5 默认端口列表

## 服务端口

| 组件 | 默认端口 | 协议 | 用途 |
|------|---------|------|------|
| TiDB | 4000 | MySQL 协议 | SQL 客户端连接 |
| TiDB | 10080 | HTTP | TiDB HTTP API / Status |
| TiKV | 20160 | gRPC | TiKV 服务端口 |
| TiKV | 20180 | HTTP | TiKV Status / PD 通信 |
| PD | 2379 | HTTP/gRPC | PD 客户端服务 |
| PD | 2380 | HTTP/gRPC | PD 节点间通信 |
| TiFlash | 9000 | TCP | TiFlash TCP 服务 |
| TiFlash | 8123 | HTTP | TiFlash HTTP 服务 |
| TiFlash | 3930 | gRPC | TiFlash 内部服务 |
| TiFlash | 20170 | gRPC | TiFlash Proxy (Raft) |
| TiFlash | 20292 | HTTP | TiFlash Proxy Status |
| TiFlash | 8234 | HTTP | TiFlash Metrics |
| TiProxy | 3080 | MySQL 协议 | SQL 负载均衡 |
| TiProxy | 3081 | HTTP | TiProxy Status |
| TiCDC | 8300 | HTTP/gRPC | TiCDC 服务 |
| TiCDC | 8301 | HTTP | TiCDC Status |
| Pump | 8250 | HTTP | Binlog Pump |
| Drainer | 8249 | HTTP | Binlog Drainer |
| DM-master | 8261 | HTTP/gRPC | DM 主控 |
| DM-worker | 8262 | HTTP | DM Worker |
| Lightning | 8289 | HTTP | Lightning Status |

## 监控端口

| 组件 | 默认端口 | 用途 |
|------|---------|------|
| Prometheus | 9090 | 指标采集 / 查询 |
| Grafana | 3000 | 可视化监控面板 |
| Alertmanager | 9093 | 告警管理 Web |
| Alertmanager | 9094 | 告警管理集群通信 |
| Node Exporter | 9100 | 主机指标采集 |
| Blackbox Exporter | 9115 | 网络探测 |

## 防火墙配置建议

```bash
# TiDB 节点需要开放的端口
-A INPUT -p tcp -m state --state NEW -m tcp --dport 4000 -j ACCEPT     # SQL
-A INPUT -p tcp -m state --state NEW -m tcp --dport 10080 -j ACCEPT    # HTTP API

# TiKV 节点需要开放的端口
-A INPUT -p tcp -m state --state NEW -m tcp --dport 20160 -j ACCEPT    # gRPC
-A INPUT -p tcp -m state --state NEW -m tcp --dport 20180 -j ACCEPT    # Status

# PD 节点需要开放的端口
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2379 -j ACCEPT     # Client
-A INPUT -p tcp -m state --state NEW -m tcp --dport 2380 -j ACCEPT     # Peer

# 监控节点
-A INPUT -p tcp -m state --state NEW -m tcp --dport 9090 -j ACCEPT     # Prometheus
-A INPUT -p tcp -m state --state NEW -m tcp --dport 3000 -j ACCEPT     # Grafana
```
