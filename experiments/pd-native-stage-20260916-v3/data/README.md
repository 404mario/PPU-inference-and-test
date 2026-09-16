# P/D 原生 STATIC 阶段试验数据

这是 P8＋D8（共 16 张 PPU 卡）和 P8＋D16（共 24 张 PPU 卡）探索性试验的精简数据导出。P 处理输入，D 逐步生成；D16 是跨两个节点的专家组。具体执行到哪些阶段，以 `phase-summaries.json` 的回执、完成状态和缺失字段为准。此数据包不代表已通过 64 卡性能、开放到达率 SLO 或生产服务验收。

`performance.jsonl` 每条记录对应一个去重后的请求。若同一请求同时存在终端流水与独立回执，独立回执优先；矛盾会记入阶段的 `duplicate_conflicts`。保留原记录的失败、未完成、错答、未评分和 token 计数状态，缺失值为 `null`。`correct` 只表示已有评分器通过，不证明广泛语义正确性。

- `usage.prompt_tokens` / `usage.completion_tokens`：服务实际报告的输入/输出 token 数；`input_tokens_expected` 为独立预期输入计数。`usage_valid` 是原客户端的计数校验结论，不是质量判断。
- `first_content_seconds` / `first_reasoning_seconds`：从客户端发起请求到首次内容/思考片段的秒数；二者分别保存，不混作一个首 token 指标。
- `tpot_seconds`：客户端后续内容时段除以服务报告的输出 token 数减一；报告的 token 可含思考，不能当成逐 token 内核耗时。`e2e_seconds` 是客户端端到端秒数。
- 阶段吞吐仅在最终汇总、全部回执和一致计数成立时复算，分母包含停止接收新请求后的排空时间。`reported_usage_valid_output_tokens_per_second` 包含完成但错答/未评分的输出；正确吞吐另列。两者都不是满足延迟约束的 SLO 有效吞吐。
- 单卡吞吐除以 P 和 D 的总卡数 16/24。应先检查实际提交的输入/输出长度分布，再比较拓扑；题库总体配额不能替代实际提交样本。

`metric-samples.jsonl.gz` 保存白名单监控指标的数值和有效标签。匿名节点 `prefill-node-1`、`decode-node-1`、`decode-node-2` 分开采集；D16 第一个 D 节点不会自动汇总第二个节点的所有 rank。快照历史的 rank 并集不代表它们曾同时健康。删除 model/path/endpoint 等标签后序列可能重名，不能盲目相加。

阶段传输计时可能包含排队和完成通知，不能等同纯设备 DMA（直接内存传输）。P 的 `prefill_forward` 与内部 `chunked_prefill` 可能重叠；D 的 `decode_waiting` 与排队指标可能重复，不能全部加总。`fake_output` 是上游预建首 token 阶段名称，本身不表示请求使用虚假 P/D。跨主机单调时钟不能直接相减。

原始日志、请求/响应文本、认证字段、网络地址和机器路径留在本机。公开包保留原件 SHA-256、匿名来源 ID 和行号；私有来源映射不随包发布。错误保留存在性、异常类别、状态码和原始错误文本散列，详细内容留本地复核。模型权重、镜像及第三方输入文档均不随包发布。

查看 `provenance.json` 确认原件/公开文件散列和导出工具版本；查看 `redaction-report.json` 确认脱敏范围。导出是对已保存证据的整理，不会发起新测量、重评分，也不会把缺失数据补成成功结果。

## 压缩存储与读取

指标文件包含 2,036,835 行 JSONL；gzip 压缩后为 20,669,948 字节，解压后为 794,756,030 字节。压缩使用固定时间戳 0，解压散列已与导出原件核对。读取时逐行处理，无需将全部数据载入内存：

```python
import gzip
import json

with gzip.open("metric-samples.jsonl.gz", "rt", encoding="utf-8") as stream:
    for line in stream:
        sample = json.loads(line)
        # 按 sample["stage"]、sample["node"]、sample["name"] 选择需要的指标。
```

`provenance.json` 的 `export_output_files` 保留初始导出清单，`storage_transformations` 记录无损压缩，`documentation_updates` 记录本说明的补充；`output_files` 对应当前 data 目录文件（清单自身除外）。整个实验目录的最终散列见上级 `PUBLICATION-MANIFEST.json`。源码类原件只补充匿名来源 ID、大小和散列，正文不发布。
