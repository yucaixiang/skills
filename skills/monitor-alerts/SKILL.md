# 监控告警配置

## 监控架构
- 采集: Prometheus + Node Exporter + JMX Exporter
- 存储: Thanos (长期存储)
- 展示: Grafana
- 告警: Alertmanager → 企业微信/电话

## 核心告警规则

### 基础设施
- CPU 使用率 > 80% 持续 5min → P2
- 内存使用率 > 90% → P1
- 磁盘使用率 > 85% → P2
- Pod 重启次数 > 3/5min → P1

### 业务指标
- QPS 下降 > 50% → P0
- 错误率 > 5% → P1
- P99 延迟 > 3s → P2
- 订单成功率 < 95% → P1

## 配置示例

```yaml
groups:
  - name: business
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: P1
        annotations:
          summary: "服务错误率超过5%"
```

## 值班安排
- 每周轮换，周一 10:00 交接
- OnCall 手机保持畅通
- 升级路径: OnCall → TL → VP
