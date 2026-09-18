# PPU inference and test

真武 PPU 上的 DeepSeek-V4.1-Flash 推理适配与实验数据。当前包含 16 / 24 卡阶段实验，以及 2026-09-17 完成的 64 卡 P/D 拓扑筛选。项目尚未完成生产服务验收。

| 实验 | 已执行范围 | 结果入口 |
| --- | --- | --- |
| `pd-native-stage-20260916-v3` | P8＋D8、P8＋D16；P/D 分离、原生 STATIC DSpark；D16 跨两个节点 | [实验报告](experiments/pd-native-stage-20260916-v3/REPORT.md) · [数据说明](experiments/pd-native-stage-20260916-v3/data/README.md) |
| `pd64-static-topology-screen-20260917-v1` | 四套 64 卡 P/D 配比与 EP8/EP16 筛选；服务阶段 profile | [实验报告](experiments/pd64-static-topology-screen-20260917-v1/REPORT.md) · [数据说明](experiments/pd64-static-topology-screen-20260917-v1/data/README.md) |
| `pd-p8d8-optimization-observability-20260918-v1` | P8+D8、EP8、STATIC DSpark 的调度通信、kernel cache 与 tokenizer 聚齐可观测优化；含进行中 500 ms 候选状态 | [实验报告](experiments/pd-p8d8-optimization-observability-20260918-v1/REPORT.md) · [数据说明](experiments/pd-p8d8-optimization-observability-20260918-v1/data/README.md) |

P（prefill）处理输入，D（decode）生成输出；EP（专家并行）将模型专家分布到多张卡。64 卡筛选的领先候选是 2P8+6D8，但仍缺同拓扑普通解码、重复稳态和自然业务质量，不能据此称为生产最优。详细比较应同时检查正确性、请求速度、输出长度、延迟和样本量。

公开包包含逐请求性能、阶段汇总、监控指标、部署参数、语料方法和校验清单。原始请求/响应、机器地址、凭据、原始日志、模型权重及镜像不随包发布。此交付是实验数据与证据说明，完整运行源码与部署包不在本次发布范围。
