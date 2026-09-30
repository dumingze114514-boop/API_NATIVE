# API_NATIVE — 心花项目：隐私保护型云 ASR 增量实施包

> **用途**：供 Codex 在现有“心花 / xin-hua ASR”工程基础上读取、判断缺口并增量实施。  
> **不是一个从零开始的新 ASR 工程，也不是用于取代既有本地路线。**

## 0. 最高优先级原则：先复用，再补缺口

Codex **不得**因为本仓库写了某个步骤，就自动重新部署、重跑或重写已有能力。

开始任何实现前，必须先检查实际工作环境，包括但不限于：

1. 当前本地/服务器上的心花工程目录、Git remote、当前 branch、`git log --oneline --all --decorate`；
2. 现有 README / implementation notes / experiment logs，特别是历史上用于本地 ASR 的计划、70 分钟 demo 记录和结果；
3. 已有脚本、模型目录、虚拟环境/容器、缓存、配置、说话人分离结果、时间戳结果、完整 70 分钟输出；
4. 已经可用的服务器连接/传输方式；
5. 已完成但未写入 README 的实际产物。

**“文档中没找到”不等于“没有做过”。**

先生成一份本地工作记录（建议 `worklogs/API_NATIVE_REUSE_AUDIT.md`），逐项标为：

- `REUSE_AS_IS`：已有且能直接复用；
- `ADAPT_ONLY`：已有，只需包一层接口或格式转换；
- `PARTIAL`：已有一部分，补缺口；
- `MISSING`：确认不存在，才允许新建；
- `UNKNOWN`：尚未验证，不允许按缺失处理。

只有 `MISSING` 与必要的 `PARTIAL` 可以成为新增工作。

---

## 1. 本包解决的问题

当前项目本地 ASR 的最终逐字稿质量不足，因此新增一条**混合隐私 ASR 路线**：

```text
原始心理咨询音频（只留本地/受控服务器）
  ↓
复用现有 VAD / diarization / 本地 ASR / timestamp 能力
  ↓
高召回敏感片段发现（多路 OR，而不是只信一次 ASR）
  ↓
本地时间定位 + 安全扩边
  ↓
生成“等长”脱敏音频（静音覆盖，不剪时间轴）
  ↓
人工 Privacy Gate
  ↓
经伦理/个人信息/厂商 Gate 批准后
  ↓
中国大陆云 ASR（首选做腾讯基线；豆包/火山作为质量挑战者）
  ↓
仅取高质量文字 + word timestamps
  ↓
回到本地，与既有 speaker timeline + privacy manifest 合并
  ↓
最终研究逐字稿
```

### 这条路线不要求重做的典型内容

若本地已经有并且可用，**不要重复**：

- 服务器 SSH / 跳板机 / 文件传输配置；
- Qwen3-ASR 或其他本地 ASR 的安装和基线部署；
- 70 分钟 demo 的既有完整本地转录；
- VAD；
- speaker diarization；
- Qwen3-ForcedAligner 或其他 timestamp/aligner；
- 已经跑过的模型 benchmark；
- 已经生成的音频预处理产物；
- 已经验证过的 GPU/CUDA/依赖环境。

本包的新工作重点是：**隐私检测与等长脱敏、合规 Gate、云 ASR adapter、云结果的本地重建、以及针对这条新路线的增量实验。**

---

## 2. Codex 必读顺序

1. [CODEX_EXECUTION_CONTRACT.md](./CODEX_EXECUTION_CONTRACT.md) — 防止重复劳动的执行约束；
2. [HYBRID_PRIVACY_ASR_PLAN.md](./HYBRID_PRIVACY_ASR_PLAN.md) — 技术路线和模块边界；
3. [ETHICS_AND_VENDOR_GATE_CN.md](./ETHICS_AND_VENDOR_GATE_CN.md) — 中国伦理/个人信息/供应商 Gate；
4. [EXPERIMENT_70MIN_INCREMENTAL.md](./EXPERIMENT_70MIN_INCREMENTAL.md) — 70 分钟 demo 的增量实验方法；
5. [PRIVACY_MANIFEST_SPEC.md](./PRIVACY_MANIFEST_SPEC.md) — 脱敏时间表的数据结构与不上传规则。

---

## 3. 设计原则

### 3.1 本地 ASR 降级为“隐私发现器”，不是最终文字真值

隐私发现优先 **recall**。宁可多遮一段，也不能因为本地 ASR 听错一个姓名就让它进入云端。

敏感候选区取多路并集：

`local ASR A ∪ local ASR B(若已有) ∪ regex ∪ NER ∪ local-LLM ∪ project entity lexicon ∪ manual flags`

不要为了满足“多路”而重新部署第二个模型；若工程中已有第二路结果则复用，否则先用现有能力 + 规则/NER/人工补强。

### 3.2 音频必须等长

敏感片段使用静音/不可恢复覆盖，**不剪除时间**。要求脱敏前后 sample count / duration 一致，从而避免后续云端时间戳和本地 speaker timeline 漂移。

### 3.3 云端不是“匿名区”

去掉姓名、电话等只是数据最小化和去标识化措施。心理咨询内容仍可能构成敏感个人信息；因此云端阶段仍按敏感个人信息委托处理来设计，不把“自动脱敏”当作法律豁免。

### 3.4 provider 可替换

第一轮优先把腾讯云做成合规/质量基线，同时允许豆包/火山等境内 provider 通过统一 adapter 接入。不要把整个 pipeline 写死在某一家 SDK。

建议接口：

```python
transcribe_redacted(
    audio_path,
    provider,
    request_options
) -> CloudTranscript
```

其中 `CloudTranscript` 至少包含：

- provider / model / request id；
- chunk global offset；
- text；
- word/sentence timestamps；
- API 参数；
- 可审计的时间与错误信息；

**不得包含真实敏感实体映射表。**

---

## 4. 生产与研究 Gate

任何真实咨询音频进入云 API 前必须同时满足：

- **Ethics Gate**：机构伦理审查对该数据流转变更已有明确批准/确认；
- **Privacy Gate**：脱敏音频已人工全量复核；直接标识符/明确第三人身份泄漏为 0；
- **Vendor Gate**：已确认处理地域、委托处理条款、留存/删除、训练/服务优化授权、子处理方/合作方条件；
- **Security Gate**：凭证不入库，传输使用 HTTPS，最小权限，可追踪删除；
- **Quality Gate（用于正式替代本地最终文本）**：在本项目 gold subset 上，云端路线对关键错误有实际改进。

没有通过前三个 Gate 时，Codex 可以把 adapter 和 mock 流程写好，但**不得把真实心理咨询音频发给外部 API**。

---

## 5. 推荐新增目录（不要强制迁移已有工程）

如现有项目已有自己的组织方式，优先融入已有结构；只有缺乏对应位置时才参考：

```text
privacy/
  detect/
  redact/
  manifest/
cloud_asr/
  base.py
  tencent.py
  volcengine.py
reconstruct/
  merge_words_speakers.py
  restore_placeholders.py
eval/
  privacy_gate.py
  transcript_eval.py
worklogs/
  API_NATIVE_REUSE_AUDIT.md
  API_NATIVE_RUN_YYYYMMDD.md
outputs/
  api_native/
```

**不要为了匹配本 README 而搬动现有稳定代码。**

---

## 6. 完成定义

本包的第一阶段完成，不是“新工程搭起来”，而是满足以下事实：

1. 已完成 reuse audit，并证明没有重复部署已有组件；
2. 能从原 70 分钟 demo 生成完整 70 分钟**等长**脱敏母版；
3. 人工 Privacy Gate 有可记录结果；
4. 至少一个境内云 provider adapter 能在非敏感/批准数据上运行；
5. 云端结果能回到本地时间轴与 speaker timeline；
6. 能输出带 `[姓名]` / `[地址]` / `[机构]` 等占位符的研究逐字稿；
7. 70 分钟全流程没有通过“只做几分钟代替完整处理”来规避；
8. 所有新增工作与历史结果分开记录，不覆盖历史 benchmark。

---

## 7. 重要说明

本仓库是 **Codex handoff / reference package**。实际代码应优先落在已有心花项目中最合理的位置，而不是为了这个参考包再复制一套工程。

法律与伦理部分用于工程合规设计，不替代机构伦理委员会、数据保护负责人或法律顾问的正式判断。规则和供应商条款可能更新；正式运行真实数据前应重新核验最新版本。
