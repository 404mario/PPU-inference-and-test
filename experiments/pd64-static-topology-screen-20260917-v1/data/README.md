# 数据说明

`results.json` 是四套 64 卡 P/D 拓扑的脱敏阶段汇总。每个正式窗口包含提供到达率、请求状态、精确正确完成速率、输出 token 速率、最大在途请求，以及 short/medium/long 三档的 TTFT、TPOT 和端到端延迟分位数。

`service-profile-*.json` 由每 5 秒 Prometheus 快照按 campaign 的精确窗口计算。gauge 记录均值、p95、最大值和非零比例；counter 使用每个指标源首末快照的差值；`per_stage_event_average` 是阶段累计耗时除以事件数。

`prefill_transfer_kv_cache` 接近实际 KV 传输阶段；`decode_transferred` 还包含等待 P 完成和状态可用，不能当作纯设备 DMA。EP16 的指标端点仅覆盖跨节点实例的部分 rank，因此 decode 事件数量并不完整。

`provenance.json` 记录私有源文件散列和公开转换方法。公开数据不含提示、输出、地址、认证信息、内部路径、命令或日志正文。
