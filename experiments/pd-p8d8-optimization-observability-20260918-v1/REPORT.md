# DeepSeek-V4.1-Flash：P8+D8 DSpark 可观测优化实验

本包发布 2026-09-18 在真武 PPU 上完成的 P8+D8、EP8、原生 STATIC DSpark 优化实验的可公开复核数据。所有完成实验均为 72/72 精确正确的请求检查；这些结果用于定位性能机制，不构成 64 卡生产验收或最终拓扑结论。

P（prefill）负责输入处理，D（decode）负责逐 token 生成，EP（专家并行）将专家分布在实例的 8 张卡上。每一正式轮并发 8 个约 6.5k-token 输入的确定性精确提取请求，输出合计 2,697 或 2,702 token；各候选均有 9 个正式轮次，且使用相同 P8+D8、模型精度、STATIC DSpark、网关策略和请求结构。

## 结果

| 候选与唯一改动 | 正确性 | 中位 makespan | 中位有效输出吞吐 | 相对对应基线 | 结论 |
| --- | ---: | ---: | ---: | ---: | --- |
| D 调度 metadata gather：设备/PCCL 基线 | 72/72 | 32.07 s | 84.11 token/s | — | 对照组 |
| D 调度 metadata gather：CPU/Gloo | 72/72 | 34.81 s | 77.47 token/s | makespan +8.6%，吞吐 −7.9% | 否决 |
| PyTorch kernel cache：未预建目录 | 72/72 | 32.07 s | 84.11 token/s | — | 对照组 |
| PyTorch kernel cache：预建可写目录 | 72/72 | 34.91 s | 77.27 token/s | makespan +8.9%，吞吐 −8.1% | 修复运维告警，不作为性能优化采用 |
| tokenizer 微批：10 ms 窗口 | 72/72 | 33.98 s | 79.53 token/s | 相对 32.07 s 基线，makespan +6.0% | 否决 |

CPU/Gloo 实验未能改善 D forward：其 D forward 中位数为 10.75 s，设备/PCCL 对照为 10.70 s。说明小 metadata collective 的显著 profile 占比主要是 rank 到达不同步造成的等待，改通信组本身不能消除等待。

kernel cache 实验消除了“kernel cache directory could not be created”告警，并观察到实际 cache 文件写入；但在该负载和测量窗口中没有可复现的性能收益。保留目录修复作为部署卫生项，不将其计入吞吐改进。

10 ms tokenizer 微批只使首批少量请求共同编码，没有消除 P 端 rank 到达错位：入口时间差中位数仍为 0.343 s（p95 0.358 s），9 轮中 7 轮仍发生“最后一个 512-token chunk 延迟”。同一轮请求完成时间差中位数为 11.32 s，说明该候选不足以解除 P/D 状态交接前的串行波次。

## P 侧机制观测

P 端实际 `chunked_prefill_size` 在 attention-DP=8 后为每 rank 512 token。约 6.5k-token 的请求需要约 13 个 chunk。基线中，8 个请求的 P scheduler 入口时间差中位数为 0.349 s、p95 为 0.392 s，却使完成时间差中位数扩大至 10.87 s；9 个正式轮中 7 轮有最后一个 chunk 延迟至少 5 s。

这解释了 7 s、约 21–23 s、约 35–41 s 三类 P forward 档：一个先到 rank 先完成第 13 个 chunk，进入 D；剩余 rank 的最后一 chunk 受同步执行顺序影响，等待 D 的一段执行后才推进。该结论来自日志与请求时间对齐，尚未证明所有负载都会出现同一模式。

## 未完成状态

`pd-tokenizer-coalesce-ab-20260918-v3` 使用 500 ms tokenizer 聚齐窗口，旨在覆盖观测到的 0.40 s 到达错位。发布时服务仍在权重加载和健康等待阶段，未进入请求执行；本包只保留其脱敏配置、运行状态和源文件散列，不发布性能数字。它不能与已完成 A/B 结论混用。

## 数据、复核与边界

- [data/results.json](data/results.json) 保存每项已完成候选的脱敏汇总观测、机制计数与摘要；逐轮原始时间序列不公开，以免恢复私有请求与运行环境。
- [data/configuration.json](data/configuration.json) 保存各候选的拓扑与唯一变量；内部地址、路径、容器命令和凭据已删除。
- [data/provenance.json](data/provenance.json) 记录本地运行包的 SHA-256、私有归档原件名称及脱敏规则；远端路径与原件本身不公开。
- [PUBLICATION-MANIFEST.json](PUBLICATION-MANIFEST.json) 列出所有公开文件和完整性散列。
- 表内五个已完成条件均通过精确正确性检查。500 ms 聚齐实验仅完成配置解析与启动，不表示运行通过、性能成立或服务验收。
- 本包不含内网地址、凭据、权重、镜像、服务日志、原始请求/响应、原始 metrics 或可用于恢复请求内容的标识符。
