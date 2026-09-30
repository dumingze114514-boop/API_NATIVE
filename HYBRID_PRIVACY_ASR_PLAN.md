# Hybrid Privacy ASR 技术方案

## 1. 核心目标

本地侧保证隐私、结构和可追溯性；云侧只提供更高质量的语音识别。

```
raw audio
  -> existing local VAD/diarization/ASR/timestamps
  -> sensitive-span detection
  -> equal-length audio redaction
  -> manual privacy review
  -> approved mainland cloud ASR
  -> word timestamps/text
  -> local reconstruction
```

## 2. 模块边界

### A. Local structure layer

优先复用已有：

- VAD
- speaker diarization
- local ASR
- timestamp / forced alignment

这些模块不是本包要求重做的内容。

### B. Sensitive-span detector

目标是高召回，不追求优雅文本。

候选敏感实体建议覆盖：

- 人名；
- 电话、身份证、邮箱、社交账号；
- 精确住址/门牌；
- 学校、单位、科室、班级等可定位信息；
- 车牌、具体账号；
- 明确第三方姓名与身份；
- 罕见事件 + 地点 + 时间等组合后可重识别的信息。

信号可取并集：

- 现有本地 ASR 文本；
- 若已有第二路 ASR，则复用；
- regex；
- 中文 NER；
- 本地 LLM；
- 项目实体词典；
- 人工标记。

不要为了满足“多模型”形式主义而额外部署新 ASR。

### C. Span-to-time mapping

优先复用已有 forced aligner/timestamp。

原则：

- 文字实体映射到音频时间段；
- 左右做安全扩边；
- 默认可从 300–500 ms 起步；
- 连续数字、账号、短姓名允许更大扩边；
- 最终以人工审核为准。

### D. Equal-length redaction

必须保持：

- sample rate 不变；
- channel 配置尽量不变；
- 总 sample count 不变；
- 总时长不变。

默认使用静音覆盖；不得删除时间片段。

产出：

- `redacted_master.wav`
- `privacy_manifest.json`
- 可选审核用 waveform/report

### E. Human Privacy Gate

首个 70 分钟 demo：

- 对机器候选区逐段审；
- 对脱敏后的完整 70 分钟音频再进行全量人工复核；
- 不允许只抽查候选区。

通过标准：

- 直接标识符泄漏：0
- 明确第三人身份泄漏：0
- 脱敏边界残留可辨敏感音节：0
- duration/sample count 偏差：0

### F. Cloud ASR adapter

不要把 pipeline 写死到 SDK。

建议抽象：

```python
class CloudASRProvider:
    def transcribe(self, audio_path, *, global_offset_s=0, options=None):
        ...
```

首轮可优先：

- Tencent Cloud：作为大陆境内合规/质量基线；
- Volcengine/Doubao：作为质量挑战者。

最终 provider 选择应基于：
- 本项目音频上的实际质量；
- 数据处理地点；
- 委托处理条款；
- 留存与删除；
- 是否用于训练/服务优化；
- 子处理方；
- 伦理审批结论。

### G. Chunking

即使 provider 支持 70 分钟整文件，也建议：

1. 先生成完整 70 分钟脱敏母版；
2. 再从母版切 API chunk；
3. 每个 chunk 保存 `global_start_s`；
4. 云时间戳返回后统一加 offset；
5. 最终自动拼接。

chunk 时长可以根据 provider 限制和稳定性调整，建议 10–20 分钟作为起点，不是硬编码。

### H. Local reconstruction

云端返回内容：

- text
- word/sentence timestamps
- provider metadata

本地已有内容：

- speaker timeline
- privacy manifest

重建流程：

```
cloud words
  + global timestamps
  + local speaker intervals
  + local privacy spans
  -> speaker-attributed redacted transcript
```

最终文本示例：

```
[00:01:01.000] 来访者：我后来就跟[姓名]说了一下……
[00:01:06.410] 咨询师：你当时是什么感觉？
```

云端永远不需要知道 `[姓名]` 对应真实值。

## 3. 不建议做的事

- 上传原始心理咨询音频；
- 只靠单次 ASR 文本做自动脱敏；
- 把“删掉姓名”理解成完成匿名化；
- 在云端注册真实声纹；
- 让不同 chunk 的云 speaker id 当最终说话人标签；
- 把 privacy manifest 上传云端；
- 将真实实体映射写进 Git；
- 为本包重做一套已存在的本地 ASR 工程。

## 4. 安全工程要求

- API key 使用环境变量/secret store；
- repo 中提供 `.env.example` 也只能写变量名；
- request/response log 不记录原始敏感实体；
- 原始音频和 privacy manifest 权限分离；
- 云端文件/任务应有可执行的删除步骤；
- 上传日志记录 provider、时间、文件 hash、chunk id、删除状态。

## 5. 允许的降级路线

如果某一模块现阶段不可用：

- forced aligner 不稳定：可人工校时间段，但保留 manifest；
- NER 不可靠：可以规则 + 本地 LLM + 人工；
- provider 暂未完成法务确认：只跑 mock/公开无敏感样本；
- diarization 已有但格式不兼容：写 adapter；
- 云 word timestamps 不稳定：可退回句级时间戳并用本地 alignment 重对齐。

不允许的“降级”：
绕过 Privacy/Ethics/Vendor Gate 上传真实数据。
