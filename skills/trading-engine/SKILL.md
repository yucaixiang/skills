# 交易撮合引擎

## 概述
高性能交易撮合引擎，支持限价单、市价单的实时撮合，日均处理量 100万+ 笔。

## 技术架构
- 核心引擎: Java 17 + Disruptor 无锁队列
- 消息通信: Kafka / RocketMQ
- 持久化: TiDB + Redis
- 监控: Prometheus + Grafana

## 撮合规则
1. 价格优先：买入价高者优先，卖出价低者优先
2. 时间优先：同等价格按委托时间排序
3. 部分成交：支持部分成交，剩余挂单

## 关键配置
- 撮合线程数: 按交易对分配独立线程
- 订单簿深度: 默认保留 200 档
- 快照间隔: 每 1000 笔成交一次快照



## Recent Update (Auto-synced)
### Rate Limiter Enhancement
- Added per-market-maker rate limiting in OrderGateway
- New config: `gateway.rate-limit.per-market-maker = 500/s`
- Code: `Gateway.java:89-102`
- PR #2089 by zhangsan (2026-05-18)