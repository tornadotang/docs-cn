---
title: TiDB-Binlog 监控指标说明
category: tools
---

# TiDB-Binlog 监控指标说明

目前对 TiDB-Binlog 中一些比较重要的方面配置了监控，根据指标的重要程度分为 Emergency、Critical 和 Warning 三种级别。

## Emergency

### binlog_pump_storage_error_count

- 含义：Pump 写 binlog 到本地存储时失败
- 监控规则：changes(binlog_pump_storage_error_count[1m]) > 0
- 处理方法：先确认 pump_storage_error 监控是否存在错误，查看 Pump 日志确认原因

## Critical

### binlog_drainer_checkpoint_high_delay

- 含义：Drainer 同步落后延迟超过 1 个小时
- 监控规则：(time() - binlog_drainer_checkpoint_tso / 1000) > 3600
- 处理方法：

    - 判断从 Pump 获取数据是否太慢：

        监控 Pump handle tso 可以看每个 Pump 最近一条消息的时间，是不是有延迟特别大的 Pump，确认对应 Pump 正常运行
        
    - 根据 Drainer event 和 Drainer execute latency 来判断是否下游同步太慢：
        
        - 如果 Drainer execute time 过大，则检查到目标库网络带宽和延迟，以及目标库状态
        - 如果 Drainer execute time 不大，Drainer event 过小，则增加 work count 和 batch 进行重试
            
    - 上面都不满足或者操作后没有改观，则报备开发 support@pingcap.com 进行处理

## Warning

### binlog_pump_write_binlog_rpc_duration_seconds_bucket

- 含义：Pump 处理 TiDB 写 Binlog 请求耗时过大
- 监控规则：histogram_quantile(0.9, rate(binlog_pump_rpc_duration_seconds_bucket{method="WriteBinlog"}[5m])) > 1
- 处理方法：

    - 确认磁盘性能压力，通过 node exported 查看 disk performance 监控
    - 如果 disk latency 和 util 都很低，那么报备研发 support@pingcap.com 处理

### binlog_pump_storage_write_binlog_duration_time_bucket

- 含义：Pump 写本地 binlog 到本地盘的耗时
- 监控规则：histogram_quantile(0.9, rate(binlog_pump_storage_write_binlog_duration_time_bucket{type="batch"}[5m])) > 1
- 处理方法：确认 Pump 本地盘情况，进行修复

### binlog_pump_storage_available_size_less_than_20G

- 含义：Pump 剩余可用磁盘空间不足 20G
- 监控规则：binlog_pump_storage_storage_size_bytes{type="available"} < 20 * 1024 * 1024 * 1024
- 处理方法：监控确认 Pump gc_tso 正常，需要的话调整 Pump gc 时间配置或者下线对应 Pump

### binlog_drainer_checkpoint_tso_no_change_for_1m

- 含义：Drainer checkpoint 一分钟没有更新
- 监控规则：changes(binlog_drainer_checkpoint_tso[1m]) < 1
- 处理方法：确认是否所有非下线 Pump 正常运行

### binlog_drainer_execute_duration_time_more_than_10s

- 含义：Drainer 同步到 TiDB 的 transaction 耗时；如果过大则影响 Drainer 同步
- 监控规则：histogram_quantile(0.9, rate(binlog_drainer_execute_duration_time_bucket[1m])) > 10
- 处理方法：

    - 查看 TiDB cluster 状态情况
    - 查看 Drainer 日志或监控，如果是 DDL 则忽略
