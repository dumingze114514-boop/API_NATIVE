# 70 分钟 demo：增量实验协议

## 1. 目标

验证新路线：

**现有本地结构能力 + 本地隐私处理 + 云 ASR + 本地重建**

能否在不重复历史工作的前提下，提高最终逐字稿质量。

## 2. 第一阶段：复用盘点

先确认现有 70 分钟 demo 已有哪些产物。

优先寻找：

- 原始/预处理音频；
- 完整 70 分钟本地转录；
- speaker diarization；
- VAD；
- word/segment timestamps；
- Qwen 或其他模型输出；
- 人工校对片段；
- 既有 CER/WER 评测；
- 运行日志与失败记录。

这些产物存在就复用，不能因为本路线新增而重新跑一遍。

## 3. 隐私发现

对完整 70 分钟执行敏感候选检测。

可复用已有 ASR 文本。

建议输出：

`privacy_candidates.jsonl`

每条包含：

- start/end；
- entity type；
- detection sources；
- confidence/notes；
- manual_review status。

候选来源采用并集。

## 4. 时间定位和等长脱敏

将确认/高风险候选转成精确时间区间，做安全扩边。

生成：

- 完整 70 分钟 `redacted_master.wav`；
- `privacy_manifest.json`；
- 校验报告。

强校验：

- 原音频总 sample count == 脱敏后总 sample count；
- 所有 redaction 区间合法、非负、无越界；
- overlapping spans 合并；
- manifest 不保存不必要的明文实体。

## 5. 人工 Privacy Gate

对首个 70 分钟 demo：

1. 审核候选敏感片段；
2. 完整听一遍脱敏版；
3. 记录漏检；
4. 如发现漏检，回写 manifest；
5. 重新生成脱敏母版；
6. 再次确认。

最终结果要记录：
- 直接标识符泄漏数；
- 第三人明确身份泄漏数；
- 边界残留数；
- 过度遮挡数量/时长；
- 人工修订次数。

## 6. 云端 A/B

仅在 Gate 允许后。

不要直接把原始音频上传。

从脱敏母版切 chunk。

建议起步：
- 10–20 分钟/chunk；
- 保留 `global_start_s`；
- provider-specific size/time limit 动态调整。

至少运行：
- 一个正式候选 provider；
- 如条件允许，再运行第二 provider 作为质量对照。

每个 job 保存：
- provider/model；
- 参数；
- request id；
- chunk hash；
- global offset；
- response metadata；
- retry/error；
- cloud deletion state。

## 7. 质量 gold subset

为了避免只凭主观感觉判断，建立 15–20 分钟人工 gold subset。

优先覆盖：

- 清晰普通对话；
- 小声/含糊；
- 重叠语音；
- 快速或情绪激动；
- 心理咨询专业表达；
- 否定词；
- 数字；
- 脱敏附近上下文。

如果历史已经有人工 gold，不要重标；先复用。

## 8. 评估

至少比较：

- 原本地 baseline；
- 新云 provider A；
- provider B（若运行）。

指标：

- CER；
- 关键语义词错误；
- 否定词错误；
- 数字错误；
- 插入/幻觉；
- 漏词；
- 重叠语音表现；
- 时间戳可用性；
- 对脱敏静音前后上下文的影响。

“云端更好”必须是项目音频上的结果，不采用厂商宣传代替。

## 9. 本地重建

把 cloud word timestamps 转换成 global timestamps。

然后与：

- 本地 speaker timeline；
- privacy manifest；

合并。

输出至少：

- machine-readable JSON；
- human-readable transcript。

placeholder 示例：

- `[姓名]`
- `[电话号码]`
- `[地址]`
- `[机构]`
- `[第三人身份]`

## 10. 完整 70 分钟要求

可以：
- 分 chunk；
- 并发；
- 失败重试；
- 局部复跑。

不可以：
- 只用 5–10 分钟实验后宣称“70 分钟完成”；
- 用 subset 代替完整最终生成；
- 因 API 成本/超时默默跳过尾部。

最终必须能证明：
从 00:00 到音频末尾都有处理覆盖。

## 11. 最终 run log

建议：
`worklogs/API_NATIVE_RUN_YYYYMMDD.md`

内容：

- 复用了哪些历史模块；
- 哪些步骤没重跑；
- 新增组件；
- Gate 状态；
- 70 分钟覆盖证明；
- provider 参数；
- 质量指标；
- 隐私审核结果；
- 失败与重试；
- 剩余问题。
