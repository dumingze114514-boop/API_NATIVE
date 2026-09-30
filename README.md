# API_NATIVE — 心花项目：隐私保护型云 ASR 增量实施包

> **用途**：供 Codex 在现有“心花 / xin-hua ASR”工程基础上读取、判断缺口并增量实施。  
> **不是一个从零开始的新 ASR 工程，也不是用于取代既有本地路线。**

## 0. 当前真实状态

心花项目已经推进到：

> **模型服务器正在对全部音视频执行全量 ASR。**

全量任务可能尚未全部完成，但已经在持续运行，因此 API_NATIVE 必须以“消费正在产生的 ASR artifact”为前提，直接衔接现有全量工程。

Codex 不应等待全部 ASR 完成才开始隐私管线，也不得为了 API_NATIVE 重新跑无关的全量工作。

---

## 1. 最高优先级原则：先复用，再补缺口

Codex **不得**因为本仓库写了某个步骤，就自动重新部署、重跑或重写已有能力。

开始任何实现前，必须先检查实际工作环境，包括但不限于：

1. 当前本地/服务器上的心花工程目录、Git remote、当前 branch、`git log --oneline --all --decorate`；
2. 现有 README / implementation notes / experiment logs；
3. 已有脚本、模型目录、虚拟环境/容器、缓存、配置、时间戳结果、ASR 结果；
4. 已经可用的服务器连接/传输方式；
5. 正在运行的全量 ASR job、队列、已完成 artifact、失败列表；
6. 已完成但未写入 README 的实际产物。

**“文档中没找到”不等于“没有做过”。**

先生成：
`worklogs/API_NATIVE_REUSE_AUDIT.md`

逐项标为：

- `REUSE_AS_IS`
- `ADAPT_ONLY`
- `PARTIAL`
- `MISSING`
- `UNKNOWN`

`UNKNOWN` 不能直接按 `MISSING` 处理。

---

## 2. 本任务真正的 P0

API_NATIVE 当前最重要的不是“所有 ASR 问题一起解决”，而是两件事：

### P0-A：敏感信息识别

计划使用组服务器上的 **Qwen 70B** 进行上下文敏感信息判断。

重点不是普通 NER，而是：

- 直接标识符；
- 第三人身份；
- 组合重识别信息；
- ASR 错误导致的疑似身份信息；
- 最小必要删除 span。

详见：
[SENSITIVE_INFO_STANDARD_QWEN70B.md](./SENSITIVE_INFO_STANDARD_QWEN70B.md)

### P0-B：敏感 span 的精确时间定位与抹除

目标是：

> **尽可能完整遮住敏感音节，同时尽可能不破坏前后非敏感语音。**

不采用统一固定大 margin；使用局部 forced alignment、多路时间证据、自适应边界和异常 review。

详见：
[TIMESTAMP_REDACTION_STRATEGY.md](./TIMESTAMP_REDACTION_STRATEGY.md)

---

## 3. ASR 质量问题如何处理

现有本地 ASR 本身有质量问题，Codex **不能假定当前转录可靠**。

但要按本任务重新排序：

### 当前优先处理

- 敏感人名/地名/机构/号码识别错误；
- 敏感实体漏字、吞字、错字；
- 数字串识别不稳定；
- 会使 Qwen 70B 漏判身份信息的转录错误；
- 文本 span 无法精确映射回音频；
- 时间戳漂移、过粗、零时长等；
- 脱敏后相邻正常语音被破坏。

### 当前非优先

- speaker diarization / 说话人区分；
- speaker label 美化；
- 与隐私识别无关的一般错字；
- 普通标点/格式优化。

如果说话人区分已经有结果，可直接复用；如果它有质量问题但**不影响当前敏感信息识别与抹除**，暂时不要为了 API_NATIVE 去优化它。

详见：
[ASR_QUALITY_FOR_PRIVACY.md](./ASR_QUALITY_FOR_PRIVACY.md)

---

## 4. 主流程

```text
模型服务器：全量音视频 ASR 持续运行
                  │
          每完成一个 artifact
                  ▼
        隐私相关 ASR 质量检查
                  │
                  ▼
        Qwen 70B 敏感信息识别
                  │
     ┌────────────┴────────────┐
     │                         │
可直接判断                文本疑似 ASR 错误
     │                         │
     │                 局部高质量精转/复核
     └────────────┬────────────┘
                  ▼
        exact character spans
                  │
                  ▼
          局部精确时间对齐
                  │
        多路时间证据/异常检测
                  ▼
          自适应边界抹除
                  │
        等长 redacted audio
                  ▼
             Privacy Gate
                  │
                  ▼
      已批准的中国大陆云 ASR
                  │
                  ▼
         高质量最终转录结果
```

---

## 5. Codex 必读顺序

1. [CODEX_EXECUTION_CONTRACT.md](./CODEX_EXECUTION_CONTRACT.md)
2. [ASR_QUALITY_FOR_PRIVACY.md](./ASR_QUALITY_FOR_PRIVACY.md)
3. [SENSITIVE_INFO_STANDARD_QWEN70B.md](./SENSITIVE_INFO_STANDARD_QWEN70B.md)
4. [TIMESTAMP_REDACTION_STRATEGY.md](./TIMESTAMP_REDACTION_STRATEGY.md)
5. [HYBRID_PRIVACY_ASR_PLAN.md](./HYBRID_PRIVACY_ASR_PLAN.md)
6. [ETHICS_AND_VENDOR_GATE_CN.md](./ETHICS_AND_VENDOR_GATE_CN.md)
7. [PRIVACY_MANIFEST_SPEC.md](./PRIVACY_MANIFEST_SPEC.md)

---

## 6. 核心设计原则

### 6.1 本地 ASR 是隐私发现输入，不是真值

Qwen 70B 的判断建立在 ASR 文本之上，因此 ASR 错误会直接造成隐私漏检。

对敏感位置的文本质量必须高于普通位置。

使用 targeted repair，而不是无条件重新跑整文件。

### 6.2 说话人区分不是当前前置条件

隐私抹除需要的是：

`敏感文本 span -> 精确音频时间区间`

不要求先解决“这个人是 CLIENT 还是 THERAPIST”。

speaker 信息若已有可保留；不稳定时不能阻塞 P0。

### 6.3 音频必须等长

静音/不可恢复覆盖敏感音频，不删除时间片。

### 6.4 云端不是匿名区

去标识化后的心理咨询音频仍按敏感个人信息处理设计。

---

## 7. 生产 Gate

### 当前阶段的云 API 冻结规则

**截至本仓库当前提交阶段，不实际调用任何云端 ASR API，不产生付费请求，也不向云端上传任何音频。**

原因是当前尚未为云 API 充值。这个限制只冻结“真实调用”，不冻结前期工程。

Codex 必须在不调用云 API 的前提下，把上线前准备做到“充值/授权后只需填入凭证并显式解锁即可运行”，至少包括：

- provider adapter 接口与实现骨架；
- 请求参数、鉴权变量和配置 schema；
- chunking、global offset 与任务状态设计；
- request payload 构造与本地 dry-run；
- mock/fake provider；
- 响应样例 fixture 与 parser；
- retry / timeout / error handling；
- 上传前 Gate 检查；
- 云任务删除/清理逻辑的实现或明确接口；
- 成本估算与批量调用预算参数；
- provider 条款/留存/训练用途核验表；
- 不含真实凭证的 `.env.example`；
- 单元测试/集成测试（使用 mock/fixture）；
- 明确的 `LIVE_CLOUD_API_DISABLED=true` 或等效硬开关。

默认必须 **fail closed**：没有用户后续明确授权与凭证时，任何代码路径都不能意外发起外部云 ASR 请求。

### 未来真实调用前的 Gate

未来在用户完成充值并明确允许启用后，任何真实咨询音频进入云 API 前仍必须同时满足：

- **Ethics Gate**
- **Privacy Gate**
- **Vendor Gate**
- **Security Gate**
- **Boundary Gate**：敏感 span 的时间边界不存在未解决的高风险异常。

如 Gate 未通过，可以继续开发、dry-run、mock 和 fixture 测试，但不得上传真实心理咨询音频。当前提交阶段即使 Gate 已满足，也仍保持云 API 冻结，直到用户后续明确解锁。

---

## 8. 推荐增量状态机

由于全量 ASR 正在运行，建议按文件异步推进：

```
ASR_RUNNING
    ↓
ASR_READY
    ↓
PRIVACY_CLASSIFY_PENDING
    ↓
PRIVACY_CLASSIFIED
    ↓
ALIGNMENT_PENDING
    ↓
REDACTED
    ↓
PRIVACY_REVIEW
    ↓
CLOUD_READY
    ↓
CLOUD_TRANSCRIBED
```

发现敏感区 ASR 质量不足时：

```
NEEDS_TRANSCRIPT_REPAIR
```

发现边界异常时：

```
NEEDS_BOUNDARY_REVIEW
```

不要因此把整个全量 ASR 流水线停掉。

---

## 9. 重要说明

本仓库是 **Codex handoff / reference package**。实际代码优先落在已有心花项目中最合理的位置，而不是复制一套新工程。

伦理/法律内容用于工程控制设计，不替代机构伦理委员会、法务或数据保护负责人的正式判断。
