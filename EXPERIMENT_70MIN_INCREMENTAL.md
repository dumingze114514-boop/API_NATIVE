# 历史 70 分钟 benchmark 与当前全量管线抽样验证

> **注意：本文件不再描述项目主线阶段。**
> 当前模型服务器已经在对全部音视频运行全量 ASR。
> 70 分钟材料只作为历史基线、回归样本或可控 benchmark 使用。

## 1. 当前目的

利用已有 70 分钟材料（若历史产物完整则直接复用）进行：

- Qwen70B 敏感信息识别规则回归；
- timestamp/redaction boundary benchmark；
- 隐私相关 ASR 错误分析；
- cloud ASR 前后邻近语音质量测试。

不得因此重新定义整个工程为 demo 阶段。

## 2. 首先复用历史产物

寻找：

- 已有完整本地转录；
- 时间戳；
- 音频预处理结果；
- 人工校对；
- 既有 CER/WER；
- 历史模型输出。

存在则复用。

## 3. 更重要的新 benchmark：敏感 span

优先从真实全量数据和历史样本中构建人工 gold：

- 两/三字姓名；
- 手机号/数字串；
- 地址；
- 机构；
- 学校/单位；
- 第三人；
- 组合重识别信息；
- 快速/小声/重叠；
- 敏感词紧邻关键研究语句。

评估：

- Sensitive Span Recall；
- Sensitive Transcription Error；
- Number/Identifier Accuracy；
- start/end boundary MAE；
- P90/P95 boundary error；
- Privacy Leakage；
- Collateral Audio Loss；
- manual review rate。

## 4. 当前不需要用该 benchmark 优化的内容

除非证明会影响隐私流程，否则不以本轮为目标：

- speaker diarization；
- speaker label；
- 一般标点；
- 与敏感信息无关的普通句 CER。

## 5. 70 分钟完整覆盖仍有价值的场景

如果需要验证端到端 regression，可要求 70 分钟完整通过 Privacy Pipeline。

但这是：
**回归测试**

不是：
**项目仍处于 70 分钟 demo 阶段**

## 6. 当前主线应关注全量增量状态

建议统计：

- ASR_READY 文件数；
- PRIVACY_CLASSIFIED 文件数；
- NEEDS_TRANSCRIPT_REPAIR 数；
- NEEDS_BOUNDARY_REVIEW 数；
- REDACTED 数；
- CLOUD_READY 数；
- CLOUD_TRANSCRIBED 数。

## 7. run log

所有实验应说明：

- 是历史 70 分钟 regression；
- 还是全量文件抽样；
- 还是生产增量运行。

避免 Codex 把三者混为一谈。
