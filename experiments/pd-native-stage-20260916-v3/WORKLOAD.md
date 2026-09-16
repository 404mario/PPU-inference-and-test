# 语料与采样方法

本附件说明 `pd-native-stage-20260916-v3` 的冻结语料、参考答案规则与实际控制程序的采样方式，核对日期为 2026-09-16。依据为该版本冻结的 `workload-manifest.json`、三份自然题库、`regression.jsonl`、`campaign.py`、`client.py` 与 `workload.py`。本附件不发布原始文档、模型输出或运行环境地址。

**本轮的性能探索仅从自然标定题库采样；300 条自然评测题库没有被控制程序调用。** 题库配额不等于实际提交数或实际长度比例。每个阶段的请求执行次数、不同题目数、输出 token 数、失败和质量结果，应以最终运行报告及逐请求账本为准。

## 1. 题库准备与实际来源

自然语料包含 300 条标定、300 条独立评测及 12 条独立预热，共 612 个不同输入。它们是公开代理任务，不能替代真实业务请求分布。每个正式题库的冻结配额如下。

| 输入桶 | 标定题数 | 标定输入范围 / 平均值（token） | 标定题实际来源 | 评测题数及来源（本轮未使用） |
| --- | ---: | --- | --- | --- |
| `extra_short`：小于 1,024 | 75 | 67–179 / 93.8 | Sanitized MBPP 75 | 75，均为 Sanitized MBPP |
| `short`：1,024–4,095 | 75 | 1,163–4,053 / 2,793.2 | QASPER 33；MultiFieldQA 中文 29、英文 13 | 75：QASPER 23；MultiFieldQA 中文 44、英文 8 |
| `medium`：4,096–19,999 | 125 | 4,146–19,673 / 8,436.9 | QASPER 41；MultiFieldQA 中文 20、英文 42；LongBench v2 22 | 125：QASPER 39；MultiFieldQA 中文 30、英文 34；LongBench v2 22 |
| `long`：20,000–30,720 | 25 | 20,570–30,405 / 24,312.6 | LongBench v2 24；QASPER 1 | 25，均为 LongBench v2 |

评测题库的对应输入平均值为 95.0、2,792.5、8,841.1、26,069.8 token，实际范围分别为 66–177、1,155–3,873、4,213–19,999、21,064–30,504。

QASPER 与 MultiFieldQA 来自 LongBench v1。标定长桶中的 24 条 LongBench v2 题由单文档问答 13 条、多文档问答 7 条、长对话历史理解 4 条组成；另一条是 QASPER 简答题。因此不能将整个长桶描述为相同的多选任务。未使用的评测长桶还包括长上下文学习和代码仓库理解题，它们不属于本轮已测覆盖范围。

Sanitized MBPP 是基础 Python 函数实现任务：请求包含原题、函数签名和一个原始示例断言；参考实现及其余测试不发送给模型。它不是长仓库上下文代码任务。简答题要求依据完整文档给出简短答案；LongBench v2 保留完整上下文、问题和四个选项，要求简短解释及最终选项。

原拟长度权重 50% / 35% / 15% 并未强行实现。冻结题库为 **25% 极短代码、25% 短问答、41.67% 中问答、8.33% 长问答**。没有填充、重复扩写或截断原文来凑长度。长输入不保证长输出；这些问答和基础代码题通常输出较短，因此不能仅凭本题库推断持续生成 1k / 2k 输出 token 时的 D 端饱和容量。

## 2. 固定来源、许可与原始数据散列

| 数据来源 | 公开固定版本 | 本次记录的许可范围 |
| --- | --- | --- |
| LongBench v1：QASPER、MultiFieldQA 英文/中文 | [HF 数据版本 `5e628be450b7e67fb7ae6e201bd6d8f7056f7672`](https://huggingface.co/datasets/THUDM/LongBench/tree/5e628be450b7e67fb7ae6e201bd6d8f7056f7672) | 官方 [GitHub LICENSE](https://github.com/THUDM/LongBench/blob/2e00731f8d0bff23dc4325161044d0ed8af94c1e/LICENSE) 为 MIT；HF 数据卡未单独声明数据许可。第三方原文权利仍归原来源，未逐篇重新核定；仓库代码许可不能自动解释为对所有原文的重新授权 |
| LongBench v2 | [HF 数据版本 `2b48e494f2c7a2f0af81aae178e05c7e1dde0fe9`](https://huggingface.co/datasets/zai-org/LongBench-v2/tree/2b48e494f2c7a2f0af81aae178e05c7e1dde0fe9) | 官方数据卡声明 Apache-2.0 |
| Sanitized MBPP | [google-research 版本 `8025cf1351da4d3cb8c289afe2d11247623fbdfb`](https://github.com/google-research/google-research/blob/8025cf1351da4d3cb8c289afe2d11247623fbdfb/mbpp/sanitized-mbpp.json) | 官方仓库 Apache-2.0；采用作者说明中手工核对过的 sanitized 子集 |

原始下载对象的 SHA-256：

| 对象 | SHA-256 |
| --- | --- |
| [LongBench v1 `data.zip`](https://huggingface.co/datasets/THUDM/LongBench/resolve/5e628be450b7e67fb7ae6e201bd6d8f7056f7672/data.zip) | `cb45b11a4133c6bc1d6a44b0f8e701335ff1e543195db1103472e575857f7f64` |
| [LongBench v2 `data.json`](https://huggingface.co/datasets/zai-org/LongBench-v2/resolve/2b48e494f2c7a2f0af81aae178e05c7e1dde0fe9/data.json) | `15d61c22d92c96900b3c4948b6aeea218d3214b676a65df48e7b8555604c7fe2` |
| [MBPP `sanitized-mbpp.json`](https://raw.githubusercontent.com/google-research/google-research/8025cf1351da4d3cb8c289afe2d11247623fbdfb/mbpp/sanitized-mbpp.json) | `ca95deaa9a01ef0a6f439f88bcf0dd3db3563d22f22aad6cae04ebb9a8d8c8e9` |

MBPP 的完整 `mbpp.jsonl` 曾用于检查来源，但题目选择使用的是上表的 sanitized 文件，二者不应混为一份数据。每条冻结 case 都保留原数据集名、原 ID、来源版本、原样本、上下文及参考答案散列，供内部复核；本附件只提供公开来源及散列。

## 3. 标定、评测和预热如何隔离

准备语料时使用固定种子按源家族划分：完整原文先归一化空白，相同原文或相同长段落（至少 300 字符）连接成一组；代码按原任务内容和参考代码 AST 的精确散列分组。整个源家族只能进入标定、评测、预热三者之一。先分配稀缺长桶，再满足其余配额；同一 split 内允许同文档的不同问题，因此题目数不等于独立文档数。

冻结检查得到 612 个不同输入，标定 281 个源家族、评测 278 个、预热 12 个。检查范围内没有跨 split 的相同完整文档、长段落或代码家族。这个检查不排除所有语义改写、不同版本原文以及模型训练数据污染。

## 4. 精确回归与自然预热分别做什么

每个拓扑的控制程序安排：

1. **精确回归**：24 条既有合成记录提取题，输入为 2,048 token 的 12 条、8,192 token 的 8 条、28,672 token 的 4 条。先用第一条执行单请求真实交接，再将余下 23 条以并发上限 8 执行。要求精确答案、正常停止、完整流结束及输入/输出计数检查通过。
2. **自然预热**：使用独立的 12 条自然预热题，四个桶各 3 条；每个拓扑以并发上限 8 完整执行两轮。极短桶为 MBPP 3 条，短桶为 QASPER 1 条和中文 MultiFieldQA 2 条，中桶为 QASPER 3 条，长桶为 LongBench v2 3 条。
3. **测量**：预热结束后进入标定采样。预热、精确回归不计入自然测量点的吞吐分母或请求总数。

自然预热是冻结的两轮安排，没有实现“观察延迟收敛后才开始”的自动判据；不应将两轮次数本身当作已经消除所有预热影响的证明。

精确回归文件中的旧 `split` 字段值为 `evaluation`，它属于此前合成回归题的命名。它与本轮 **未使用的 300 条自然 `evaluation.jsonl`** 是不同题库。两者必须在运行账本中分开统计。

## 5. 本轮如何从 300 条题库采样

`campaign.py` 读取全部 300 条自然标定题，仅做一次 `random.Random(20260916).shuffle(cases)`。对 P8＋D8、P8＋D16 两种拓扑，均按服务整体并发上限 **1、8、16** 的顺序运行探索点。并发是一个服务的在途请求数，不是每张卡的请求数。

每个探索点的调度序号从零开始，按同一打乱序列依次取题；超过 300 次提交后从头循环。不同并发或拓扑因此共享请求序列的前缀，但实际提交到哪个位置由运行时间和完成速度决定。**程序没有对每个点强制执行题库的四桶配比，也没有保证每个点遍历全部 300 个题目。** 比较前必须报告实际提交分布及不同题目数，不能直接套用题库权重。

客户端采用有界闭环：一个工作线程收到当前请求终态后才发下一条，最多同时存在 C 条请求。每点：

- 目标为至少运行 180 秒且至少有 30 条传输完成的请求；两个条件都达到后停止新增请求。
- 最多接收新请求 480 秒，到上限就停止新增，随后等待已提交请求终态。每请求超时为 1,200 秒。
- 达到接收上限但不足 30 条完成时，控制程序记录该点为样本不足；不能按正式尾延迟或容量验收解释。
- 发生传输故障时停止新增请求并排空在途请求。系统错误、无效计数或评分器异常阻止进入后续正常测量；自然错答与未评分结果保留在结果中，不等同系统故障。
- 每个点结束且在途请求全部收敛后才进入下一个点，吞吐计时包含排空阶段。控制程序没有给这些点做三次正式重复。

其中“30 条完成”是传输完成数，包含正常答错、未评分和正常返回的长度截断，不是 30 条质量通过。实际提交可能因在途请求排空而超过该数。它是初步筛选协议，**不是固定到达率的开放流量实验，也不是 SLO（服务延迟目标）有效吞吐验收**。

## 6. 分词、输出预算与质量规则

模型请求固定使用 `DeepSeek-V4.1-Flash`，`temperature=0.0`，`chat_template_kwargs={"thinking": false}`，流式返回及 usage。通过官方 V4.1 `encode_messages(thinking_mode="chat")` 渲染完整提示，再由 tokenizer 以 `add_special_tokens=False` 计数，计入模板开销。

输出预算按极短/短、中、长桶分别为 256、1,024、2,048 token；所有自然题的冻结输入加输出预算均不超过 32,768 token。预算不是实际输出长度。客户端将服务端 `usage.prompt_tokens` 与冻结输入计数严格比较；实际输出计数采用服务端 `completion_tokens`，不冒称完成了独立输出重新分词。

质量与服务终态分开处理：

| 任务 | 冻结质量规则 | 解释边界 |
| --- | --- | --- |
| 精确合成回归 | 允许 CRLF 归一化和至多一个尾换行，随后与完整参考答案精确比较 | 只检查该受控任务的正确性，不表示自然任务质量 |
| 文档简答 | NFKC、大小写、标点、空白及英文冠词归一化后精确匹配；另报 token F1，中文按字符 | 正确改写可能被严格匹配拒绝；F1 不是人工语义裁决 |
| 长上下文多选 | 最后非空行必须为 `ANSWER: A/B/C/D`，并与原选项匹配 | 前面的解释不独立评分；选项正确不等于解释全部正确 |
| MBPP 代码 | 保留原断言和参考实现，本轮评分模块不执行代码 | 未经隔离单测执行一直记为 `unscored`，HTTP 成功和语法形态不能算质量通过 |

流式传输完成要求收到结束标记和 finish reason；自然请求的 `length` 截断可算传输完成，但不能据此判质量通过。代码未评分结果仍单列。实际统计同时保留服务完成、质量规则通过、错答、未评分、截断、无效 usage 与系统失败。

TTFT 是客户端发出请求到首内容事件的时间；TPOT 以首内容后到响应体完成的耗时除以服务端报告的后续 token 数。这是平均生成间隔估计，SSE 内容块间隔不是逐 token 间隔，也不能直接与不同主机时钟相减推导 P/D 阶段耗时。各点报告的完成请求/s、规则通过请求/s 和输出 token/s 均不能直接称作满足延迟门槛的生产 goodput。

## 7. 冻结附件散列

以下 SHA-256 已按控制程序所用的本地冻结文件重新核对。它们用于确认同一份输入与同一套采样、评分规则，不携带环境地址。

| 文件 | SHA-256 |
| --- | --- |
| `calibration.jsonl` | `0416afee2929b0855a5e757042dd6d89d6db044fbd74fd50b1d25dbeced93395` |
| `evaluation.jsonl`，本轮未使用 | `f9e30ddc8afcccb32eaaffbdc0245d6fe24f97b10dcfe917c7056607a30e4bef` |
| `warmup.jsonl` | `a130d354d7704174f45fe902ab0ae2480bf479ab7c8babbcc927d30f17cab75e` |
| `regression.jsonl` | `ec39c3b799735e0a47509b070cd3fe8cc6124e58633f2ef0a7cb8706fe86b3e2` |
| `workload-manifest.json` | `a5b561f5d66d132596e8fc013129a056a1d9a09c75ee0178413f2ab70f1fd749` |
| `campaign.py` | `caea4701476f64399273abede176933687b33562e2376e48af75827cbd7086f6` |
| `client.py` | `ce1dd3ea90aef9d193a34180c9d9bf67fbf7ff3bbb5fb7ddd201cb479b1122da` |
| `workload.py` | `9416e600cbd9893f77cb2d997e862d67cfa0b65ec4aba1f0270e2731f6f79efd` |
| `tokenizer.json` | `c90dfa01249db1be4245780a052ede752e1361c612ac6d08e2bdada7d599476b` |
| `tokenizer_config.json` | `6ac8c8dc065ed118161d02dd532749ae3f52c243deac27872134fae2f50d8547` |
| 官方 `encoding.py` | `502bdaec8a3fd88ebc24c4721a7038fbe42f2063c664638127056107920035c1` |

公开方法描述固定到这些版本。实际运行是否完成、每点实际提交与长度分布、精确回归通过情况、自然质量计分及性能结论由最终运行报告单独给出，不能由题库准备检查推导。
