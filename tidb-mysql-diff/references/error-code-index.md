# TiDB v8.5 常见错误码速查（8000-9000 段）

## 兼容性错误

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| 8108 | FUNCTION xxx is not supported | 使用了 TiDB 不支持的函数/语法 | 查找替代方案或改写 SQL |
| 1064 | SQL syntax error | 语法错误（可能是兼容性问题） | 检查 TiDB 语法支持 |
| 1175 | UPDATE/DELETE without WHERE | 安全模式限制 | SET sql_safe_updates=0 或加 WHERE |

## 事务错误

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| 8028 | Information schema is changed | 乐观事务冲突，重试失败 | 改用悲观事务或应用层重试 |
| 8003 | Write conflict | 并发写入同一 Key | 悲观事务或降低并发 |
| 9007 | Write conflict during pessimistic transaction | 悲观事务写入冲突 | 检查事务逻辑，减少冲突 |

## TiKV/Region 错误

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| 9005 | Region is unavailable | TiKV 节点异常或 Region 未选举 | 检查 TiKV 状态，等待恢复 |
| 9006 | GC life time is shorter than transaction duration | 事务执行时间超过 GC TTL | 增大 tidb_gc_life_time 或优化事务 |
| 9012 | Empty response | TiKV 返回空响应 | 检查 TiKV 是否 OOM 或宕机 |

## 连接错误

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| 1040 | Too many connections | 连接数超过限制 | 增大 max_connections 或使用连接池 |
| 1045 | Access denied | 用户名/密码错误 | 检查凭据 |
| 2003 | Can't connect to MySQL server | TiDB 节点不可达 | 检查网络和 TiDB 进程 |

## 数据错误

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| 1062 | Duplicate entry | 主键/唯一索引冲突 | 处理重复数据 |
| 1406 | Data too long for column | 数据长度超过列定义 | 修改列定义或截断数据 |
| 1265 | Data truncated for column | 数据类型不匹配 | 检查数据类型 |

## 资源限制错误

| 错误码 | 错误信息 | 原因 | 解决方案 |
|--------|---------|------|---------|
| 1105 | Runaway query exceeds resource group quota | 资源组限制触发 | 调整资源组配额或优化 SQL |
| 8004 | Transaction is too large | 事务大小超过限制 | 拆分事务（默认限制 10GB） |
