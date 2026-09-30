# 敏感信息时间戳与精确音频抹除策略

## 0. 为什么这是 P0

本任务的目标不是“给逐字稿配一个大概时间”。

时间戳直接决定：

> **哪一小段原始音频会被不可恢复地覆盖。**

边界太窄：
敏感音节残留。

边界太宽：
破坏非敏感语音，降低后续云 ASR 质量。

因此 timestamp/redaction boundary 与敏感信息识别同为 P0。

---

# 1. 不使用统一固定扩边

不要全局机械使用：

- 左右固定 300 ms；
- 左右固定 500 ms；
- 任何其他单一 margin。

尤其中文连续语流中，固定大扩边很容易把相邻字词一起抹掉。

margin 应根据：

- aligner 一致性；
- 实际音节边界；
- 实体类型；
- 语速；
- 相邻停顿；
- chunk 边界；
- 是否重叠语音；
- 模型置信；

自适应确定。

---

# 2. 文本 span 是核心桥梁

Qwen 70B 先在原始 ASR 文本上输出：

```
start_char
end_char
span_text
```

然后再进行：

```
text span
  -> local context extraction
  -> forced alignment
  -> audio interval
  -> boundary verification
  -> redaction
```

不要让 Qwen 70B 直接猜秒级时间。

---

# 3. 优先局部对齐，不必重新跑整文件

已知敏感 span 后：

1. 利用现有 segment timestamp 找到大概位置；
2. 截取前后若干秒局部音频；
3. 在局部执行高精度 forced alignment；
4. 将局部时间转换回 global timestamp。

这比为了几个敏感实体重新跑整小时 alignment 更高效，也更容易人工检查。

---

# 4. 两路时间证据

如果现有工程已有多个 timestamp 来源，优先复用。

建议思路：

- 主 aligner：Qwen3 ForcedAligner 或现有更成熟对齐器；
- 独立校验：现有第二时间系统 / FunASR timestamp / CTC alignment；
- 不要求为了形式主义部署全部系统。

关键是：
**高风险 span 要有独立的边界检查机制。**

---

# 5. Boundary confidence

每个 redaction span 建议产生：

```json
{
  "start_s": 31.42,
  "end_s": 31.88,
  "boundary_confidence": "HIGH",
  "sources": ["aligner_a", "aligner_b"],
  "start_disagreement_ms": 40,
  "end_disagreement_ms": 80
}
```

首轮可实验的阈值：

- 差异 <= 80 ms：HIGH 候选；
- 80–160 ms：MEDIUM；
- 160–240 ms：LOW / review；
- >240 ms：必须 review/repair。

这些不是永久科学阈值，必须通过真实项目标注集校准。

---

# 6. 强制异常检测

以下情况不得直接自动上传云端：

- `start == end`；
- `start > end`；
- 连续字符大量共享同一时间点；
- 敏感 span 无法覆盖到任何语音；
- 两个 aligner 差异过大；
- span 跨 chunk 切点；
- alignment 跳到错误句；
- 敏感实体文本本身疑似 ASR 错误；
- 说话重叠导致边界无法可靠确定。

这些输出 `REVIEW` 或 `NEEDS_TRANSCRIPT_REPAIR`。

---

# 7. 自适应 margin

margin 的目的只是在声学边界不确定时防止残留，不是“保险越大越好”。

建议逻辑：

### 存在明显静音/停顿边界

优先将 mask 边界贴近停顿，不额外扩大到相邻词。

### 两路 aligner 高一致

使用很小 safety margin，并通过试听 benchmark 决定。

### 连续数字

数字容易连读，可适当扩大，但尽量不进入前后语义词。

### 两字/三字姓名

特别检查首尾音节，不机械吞掉称谓和谓词。

### 低音量/快速语音

低置信时宁可 REVIEW，不要用巨大 margin 自动解决。

---

# 8. Redaction Boundary Benchmark

必须建立专门 benchmark，而不能只看 CER。

建议从全量运行中的真实材料抽取足够多敏感实例，人工标注声学边界。

覆盖：

- 两/三字姓名；
- 手机号；
- 身份证/数字串；
- 地址；
- 单位/学校；
- 姓名+称谓；
- 快速讲话；
- 小声；
- 重叠；
- 敏感词紧邻重要研究语句；
- chunk 边界附近。

指标：

- start boundary MAE；
- end boundary MAE；
- P50/P90/P95 error；
- sensitive audio leakage duration；
- sensitive span 完全覆盖率；
- collateral non-sensitive audio removed；
- zero-duration rate；
- alignment failure rate；
- manual review rate。

---

# 9. 最终优化目标

不是单独最小化时间误差。

真正目标是：

```
minimize:
    privacy leakage
    +
    collateral audio loss
```

在隐私 Gate 中，隐私泄漏是硬约束；在满足不泄漏后，再尽量减少对周围正常语音的破坏。

---

# 10. 说话人区分不是本阶段前置条件

如果现有 diarization 能帮助局部定位，可使用。

但：

- 不为了时间戳系统重新优化 speaker diarization；
- 不因为 speaker label 不稳定阻塞敏感 span 对齐；
- 时间戳应基于实际文本—音频对齐，而不是依赖 speaker id 正确。

说话人区分后续可以继续优化，但不是当前 P0。
