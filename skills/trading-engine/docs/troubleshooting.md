# 撮合引擎问题排查手册

## 快速诊断

### 健康检查

```bash
# 1. 检查进程状态
ps aux | grep matching-engine

# 2. 检查端口监听
netstat -tlnp | grep 8080

# 3. 检查日志
tail -f /var/log/matching-engine/engine.log

# 4. 检查指标
curl http://localhost:9090/metrics
```

### 常见故障现象

| 现象 | 可能原因 | 排查命令 |
|------|---------|---------|
| 撮合延迟升高 | CPU 瓶颈、订单积压 | `top -H -p $(pgrep matching)` |
| 订单无响应 | 网关连接断开 | `netstat -an \| grep 8080` |
| 内存持续增长 | 订单簿未清理 | `pmap -x $(pgrep matching)` |
| WAL 写入失败 | 磁盘满、权限问题 | `df -h && ls -la /data/wal` |

## 性能问题

### 延迟异常

**症状**：P99 延迟 > 5ms

**排查步骤**：

```bash
# 1. 查看实时延迟分布
./scripts/latency_histogram.sh

# 2. 检查是否有慢订单
grep "SLOW_ORDER" /var/log/matching-engine/engine.log | tail -20

# 3. 分析火焰图
perf record -p $(pgrep matching) -g -- sleep 10
perf script | flamegraph.pl > flame.svg

# 4. 检查锁竞争
perf record -e sched:sched_switch -p $(pgrep matching) -- sleep 5
```

**常见原因与解决方案**：

1. **订单簿深度过大**
   - 现象：单价格档位订单 > 1000
   - 解决：清理僵尸订单，增加订单过期时间
   ```cpp
   void cleanupStaleOrders() {
       auto now = getCurrentTime();
       for (auto& [price, level] : orderbook.bids) {
           level.orders.erase(
               std::remove_if(level.orders.begin(), level.orders.end(),
                   [now](Order* o) { return now - o->timestamp > 3600; }),
               level.orders.end()
           );
       }
   }
   ```

2. **WAL 同步阻塞**
   - 现象：WAL 写入延迟 > 10ms
   - 解决：切换到异步 WAL 模式（牺牲部分一致性）
   ```cpp
   wal_logger.setSyncMode(WALSyncMode::ASYNC);
   wal_logger.setFlushInterval(std::chrono::milliseconds(100));
   ```

3. **CPU 争抢**
   - 现象：CPU 使用率 > 90%
   - 解决：增加撮合线程，启用批量处理
   ```yaml
   engine:
     thread_count: 8
     batch_size: 200
   ```

### 吞吐量下降

**症状**：TPS < 5 万

**排查步骤**：

```bash
# 1. 查看当前 TPS
watch -n 1 'curl -s http://localhost:9090/metrics | grep order_processing_rate'

# 2. 检查订单队列积压
curl http://localhost:8080/admin/queue_stats

# 3. 分析慢查询
grep "SLOW_MATCH" /var/log/matching-engine/engine.log
```

**优化方案**：

```cpp
// 批量撮合优化
void batchMatch(std::vector<Order*>& orders) {
    // 按 symbol 分组
    std::unordered_map<std::string, std::vector<Order*>> grouped;
    for (auto* order : orders) {
        grouped[order->symbol].push_back(order);
    }

    // 并行撮合
    #pragma omp parallel for
    for (auto& [symbol, symbol_orders] : grouped) {
        for (auto* order : symbol_orders) {
            matchOrder(order);
        }
    }
}
```

## 数据一致性问题

### 订单状态不一致

**症状**：用户看到的订单状态与数据库不符

**排查步骤**：

```bash
# 1. 查询订单生命周期
./scripts/trace_order.sh <order_id>

# 2. 对比内存与数据库
redis-cli GET order:<order_id>
psql -c "SELECT * FROM orders WHERE order_id='<order_id>'"

# 3. 检查事件流
kafka-console-consumer --topic order-events \
  --from-beginning --max-messages 100 | grep <order_id>
```

**修复方案**：

```cpp
// 数据一致性校验
void reconcileOrder(const std::string& order_id) {
    auto mem_order = orderbook.getOrder(order_id);
    auto db_order = database.queryOrder(order_id);

    if (!mem_order && db_order) {
        LOG(WARNING) << "Order exists in DB but not in memory: " << order_id;
        // 从 WAL 恢复
        replayOrderFromWAL(order_id);
    } else if (mem_order && !db_order) {
        LOG(WARNING) << "Order exists in memory but not in DB: " << order_id;
        // 强制持久化
        database.insertOrder(*mem_order);
    } else if (mem_order->status != db_order->status) {
        LOG(ERROR) << "Order status mismatch: " << order_id;
        // 以内存状态为准
        database.updateOrderStatus(order_id, mem_order->status);
    }
}
```

### 成交金额对不上

**症状**：用户账户余额与成交流计算结果不符

**排查步骤**：

```sql
-- 1. 查询用户成交记录
SELECT trade_id, price, quantity, fee, create_time
FROM trades
WHERE user_id = '<user_id>'
  AND create_time > NOW() - INTERVAL '1 hour'
ORDER BY create_time;

-- 2. 计算期望余额
SELECT
    user_id,
    SUM(CASE WHEN side='BUY' THEN -price*quantity-fee ELSE price*quantity-fee END) as net_change
FROM trades
WHERE user_id = '<user_id>'
GROUP BY user_id;

-- 3. 对比实际余额
SELECT balance FROM accounts WHERE user_id = '<user_id>';
```

**修复脚本**：

```python
#!/usr/bin/env python3
import psycopg2

def fix_balance_mismatch(user_id):
    conn = psycopg2.connect("dbname=trading")
    cur = conn.cursor()

    # 计算正确余额
    cur.execute("""
        SELECT COALESCE(SUM(amount), 0)
        FROM transactions
        WHERE user_id = %s
    """, (user_id,))
    expected_balance = cur.fetchone()[0]

    # 更新余额
    cur.execute("""
        UPDATE accounts
        SET balance = %s,
            updated_at = NOW()
        WHERE user_id = %s
    """, (expected_balance, user_id))

    conn.commit()
    print(f"Fixed balance for user {user_id}: {expected_balance}")
```

## 高可用问题

### 主备切换失败

**症状**：主节点宕机后备节点未接管

**排查步骤**：

```bash
# 1. 检查 Raft 集群状态
./bin/raft-cli status

# 2. 查看选举日志
grep "ELECTION" /var/log/matching-engine/raft.log

# 3. 检查网络连通性
ping <backup_node_ip>
telnet <backup_node_ip> 8081
```

**手动切换**：

```bash
# 1. 停止主节点
ssh master "systemctl stop matching-engine"

# 2. 提升备节点为主
ssh backup "curl -X POST http://localhost:8080/admin/promote"

# 3. 验证流量切换
curl http://backup:8080/health
```

### 数据同步延迟

**症状**：备节点数据落后主节点 > 1s

**排查步骤**：

```bash
# 1. 查看同步延迟
curl http://master:8080/admin/replication_lag

# 2. 检查网络带宽
iftop -i eth0

# 3. 查看 WAL 发送速率
grep "WAL_SEND" /var/log/matching-engine/replication.log | tail -100
```

**优化方案**：

```cpp
// 批量同步优化
class ReplicationManager {
public:
    void syncToReplica() {
        std::vector<WALEntry> batch;
        while (batch.size() < 1000 && wal_queue.tryPop(entry)) {
            batch.push_back(entry);
        }

        if (!batch.empty()) {
            // 批量发送
            sendBatch(replica_connection, batch);
        }
    }
};
```

## 监控告警

### 告警处理流程

```
告警触发 → 值班人员响应 → 初步排查 →
  ├─ 能自行解决 → 执行修复 → 记录问题 → 结束
  └─ 无法解决 → 升级专家 → 协同处理 → 记录问题 → 结束
```

### 关键告警及处理

| 告警名称 | 触发条件 | 处理 SOP |
|---------|---------|---------|
| HighMatchingLatency | P99 > 5ms for 1min | 1. 检查订单队列<br>2. 重启实例<br>3. 切换备用 |
| WALWriteFailure | 写入失败 > 0 | 1. 检查磁盘空间<br>2. 检查权限<br>3. 手动轮转日志 |
| OrderbookMemoryHigh | 内存 > 8GB | 1. 清理过期订单<br>2. 触发快照<br>3. 重启实例 |
| ReplicationLag | 延迟 > 5s | 1. 检查网络<br>2. 增加带宽<br>3. 临时停止备份 |

### 监控大盘

**Grafana 面板**：http://grafana.company.com/d/matching-engine

**关键指标**：
- 撮合延迟（P50 / P99 / P999）
- TPS（按交易对）
- 订单簿深度（买卖盘档位数）
- 内存占用趋势
- WAL 写入速率
- 网络 I/O

## 应急预案

### 场景 1：撮合引擎宕机

```bash
# 1. 立即切换到备用实例
./scripts/failover.sh --to backup-01

# 2. 通知业务方（暂停交易）
curl -X POST http://gateway/admin/pause_trading

# 3. 排查宕机原因
journalctl -u matching-engine -n 500

# 4. 修复后重新上线
systemctl start matching-engine
./scripts/health_check.sh
```

### 场景 2：数据库连接断开

```bash
# 1. 检查数据库状态
pg_isready -h db-master -p 5432

# 2. 切换到只读模式（停止持久化）
curl -X POST http://localhost:8080/admin/readonly_mode

# 3. 等待数据库恢复后重新连接
systemctl restart matching-engine
```

### 场景 3：消息队列积压

```bash
# 1. 检查 Kafka 消费 lag
kafka-consumer-groups --describe --group matching-engine-group

# 2. 临时增加消费者
./scripts/scale_consumers.sh --count 10

# 3. 清理旧消息（谨慎操作）
kafka-delete-records --topic trades --offset-json-file clear.json
```

## 日志分析

### 日志格式

```
[2024-05-21 10:23:45.123] [INFO] [MatchingEngine] Order received: order_id=xxx, symbol=BTC/USDT
[2024-05-21 10:23:45.125] [DEBUG] [OrderBook] Added to book: price=50000, qty=0.1
[2024-05-21 10:23:45.128] [INFO] [MatchingEngine] Trade executed: trade_id=yyy, price=50000, qty=0.1
[2024-05-21 10:23:45.130] [WARN] [WALLogger] Slow write detected: 15ms
[2024-05-21 10:23:45.135] [ERROR] [Kafka] Failed to publish trade: connection timeout
```

### 常用日志查询

```bash
# 查找错误日志
grep ERROR /var/log/matching-engine/engine.log | tail -50

# 查找特定订单的日志
grep "order_id=12345" /var/log/matching-engine/*.log

# 统计每分钟订单量
awk '/Order received/ {print $1" "$2}' engine.log | cut -d: -f1,2 | uniq -c

# 查找慢撮合
awk '/Match took/ && $NF > 5 {print}' engine.log
```

## 联系支持

- **紧急联系人**：张三 (13800138000)
- **值班电话**：400-123-4567
- **钉钉群**：「交易核心研发」
- **工单系统**：https://jira.company.com/projects/TRADING
