# Hybrid Privacy ASR 技术方案

## 1. 当前架构定位

模型服务器已经在持续执行全量 ASR。本方案是**后接的增量 Privacy Pipeline**，不是替代当前全量 ASR。

```
full-scale ASR artifacts
  -> privacy-related transcript quality check
  -> Qwen 70B sensitive-span classification
  -> targeted transcript repair when needed
  -> precise local span-to-audio alignment
  -> adaptive equal-length redaction
  -> privacy review
  -> approved mainland cloud ASR
```

## 2. 当前 P0

### A. 敏感信息识别

使用 Qwen 70B 按：
[SENSITIVE_INFO_STANDARD_QWEN70B.md](./SENSITIVE_INFO_STANDARD_QWEN70B.md)

输出 exact character span，不允许只输出“整句敏感”。

### B. 精确时间戳和抹除边界

按：
[TIMESTAMP_REDACTION_STRATEGY.md](./TIMESTAMP_REDACTION_STRATEGY.md)

**不再使用固定 300–500 ms 作为全局默认扩边。**

固定 margin 只能作为特定实验条件，不能作为生产策略。

### C. 隐私相关 ASR 质量

按：
[ASR_QUALITY_FOR_PRIVACY.md](./ASR_QUALITY_FOR_PRIVACY.md)

只优先修复会影响：
- 敏感信息召回；
- 敏感 span 文本准确性；
- 时间映射；
- redaction 邻近正常语音；

的问题。

## 3. 非当前优先

speaker diarization 不是本阶段前置条件。

如果已有，可复用；如果质量不好但不影响隐私识别/抹除，不投入主要资源修复。

## 4. Sensitive-span detector

候选信号可包括：

- 当前全量 ASR 文本；
- regex；
- 项目实体词典；
- Qwen 70B 上下文判断；
- 已有第二路 ASR（若已存在）；
- 人工标记。

不要为了形式主义增加新的全量 ASR 模型。

## 5. Targeted transcript repair

如果 Qwen 70B 或规则判断：
`NEEDS_TRANSCRIPT_REPAIR`

只提取对应局部音频做高质量转录/复核。

不要重跑整文件。

## 6. Span-to-time mapping

步骤：

1. Qwen70B 返回输入原文的 char span；
2. 使用已有 segment timestamp 找局部音频；
3. 对局部执行 forced alignment；
4. 用第二时间证据/异常规则验证；
5. 自适应选择边界；
6. 高风险异常进入 review。

## 7. Equal-length redaction

必须保持：
- sample rate；
- 总 sample count；
- 总 duration；
- 全局时间轴。

默认使用不可恢复静音/覆盖。

## 8. Privacy Gate

至少检查：

- 直接身份信息残留；
- 第三人身份残留；
- 边界残留音节；
- collateral non-sensitive speech loss；
- 时间长度一致性。

## 9. Cloud ASR adapter

provider 必须可替换。

云端主要提供：
- 更高质量最终文字；
- 时间信息（若有用）。

不依赖云端 speaker diarization。

## 10. Chunking

对脱敏母版切块时保持 global offset。

chunk 大小由：
- provider 限制；
- 稳定性；
- 上下文质量；
- 成本；

共同决定。

## 11. Local reconstruction

当前优先输出：

- 高质量文本；
- global timestamp；
- privacy placeholder。

speaker attribution 可保留已有结果，但不是当前完成定义的硬要求。

## 12. 安全要求

- API key 不入库；
- privacy manifest 不上传；
- 真实实体映射不上传；
- request log 不记录明文敏感实体；
- 上传/删除状态可审计。

## 13. 允许降级

可以：
- aligner 异常时人工局部校；
- Qwen70B 不确定时局部重转录；
- provider 未批准时仅 mock；
- speaker diarization 暂不修。

不允许：
- 绕过隐私/伦理/vendor Gate；
- 用粗时间戳直接自动大范围抹除后上传；
- 因 ASR 文本可疑仍强行让 Qwen70B 二选一。
