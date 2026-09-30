# PRE_CLOUD_API_READINESS — 云端调用前准备清单

## 0. 当前阶段边界

截至本仓库当前提交阶段：

- **禁止实际调用任何云端 ASR API**；
- **禁止产生付费 API 请求**；
- **禁止上传任何音频到云 ASR**；
- 原因：当前尚未为云 API 充值。

这不是“暂停工程”。相反，Codex 应把所有不需要真实调用的前期工作完成到可直接启用状态。

---

## 1. 完成定义

只有以下项目全部完成，才算“云 API 前期准备完毕”。

### Provider abstraction

- 统一 provider interface；
- provider-specific adapter；
- 请求参数 schema；
- 模型/语言/时间戳等能力配置；
- provider 能力差异文档。

### Secret/config

- `.env.example` 仅包含变量名；
- 真实 key 不入库；
- live 调用默认关闭；
- 必须存在显式解锁开关；
- 缺 key / 未解锁时 fail closed。

建议：

```
LIVE_CLOUD_API_DISABLED=true
CLOUD_ASR_PROVIDER=
CLOUD_ASR_API_KEY=
CLOUD_ASR_SECRET=
```

实际变量名可按 SDK 调整。

### Request preparation

- 音频格式验证；
- redacted-only 校验；
- chunking；
- global offset；
- request payload 构造；
- 参数序列化；
- idempotency / task id 设计；
- retry policy；
- timeout；
- rate limit/backoff。

### Dry-run

必须支持 dry-run：

```
input
 -> Gate check
 -> chunk plan
 -> payload build
 -> estimated request count
 -> estimated audio duration
 -> estimated cost
 -> STOP before network call
```

dry-run 输出不得包含不必要的真实敏感文本。

### Mock provider

实现 mock/fake provider，使完整 pipeline 可以在零云调用下测试：

```
redacted audio
 -> mock adapter
 -> fixture response
 -> parser
 -> global timestamp reconstruction
 -> downstream output
```

### Response fixtures

至少准备：

- 正常成功响应；
- 无时间戳；
- 部分结果；
- provider error；
- timeout；
- rate limit；
- malformed response；
- task polling；
- duplicate/retry 场景。

fixture 必须是合成/虚构数据。

### Parser/reconstruction

在不连接真实 API 的情况下验证：

- text parsing；
- sentence/word timestamp parsing；
- chunk offset 恢复；
- placeholder 回插；
- 与 privacy manifest 的时间关系；
- 异常结果拒绝逻辑。

### Gate enforcement

network call 之前代码必须验证：

- live API 已显式解锁；
- Privacy Gate；
- Ethics Gate；
- Vendor Gate；
- Security Gate；
- Boundary Gate；
- 输入确实是 redacted artifact，而不是 raw audio。

任一失败：
**不得发请求。**

### Cleanup design

即使当前不实际调用，也应提前实现或明确：

- cloud task delete/cancel API；
- remote file delete；
- cleanup 状态；
- cleanup retry；
- audit log 字段。

### Cost and batch planning

准备：

- 单位价格配置；
- 总音频时长读取；
- 每文件/每批成本估算；
- 请求数估算；
- budget ceiling；
- 超预算阻断逻辑。

不要在代码里硬编码临时价格；将价格作为可更新配置并记录核验日期。

### Auditability

准备日志字段：

- local artifact id/hash；
- provider；
- model；
- chunk；
- global offset；
- dry-run/live；
- Gate 状态；
- request id（未来）；
- response status；
- cleanup status；
- cost estimate / actual cost（未来）。

不得记录真实 API key。

---

## 2. 当前允许的测试

允许：

- 本地单元测试；
- mock integration test；
- fixture parser test；
- dry-run；
- request serialization test；
- Gate failure test；
- redaction artifact validation；
- cost calculation；
- provider documentation/contract research。

不允许：

- “只发一个请求看看”；
- 使用免费额度试调；
- 使用公开音频调用真实接口；
- 使用脱敏音频调用真实接口；
- 任何可能触发真实云 ASR 请求的 SDK smoke test。

---

## 3. 解锁条件

未来只有在用户明确说明可以开始实际云 API 调用后，Codex 才能考虑解除冻结。

解除冻结不能只改一个布尔值；还需要再次确认：

- provider 配置；
- 价格；
- Ethics/Vendor/Privacy/Security/Boundary Gate；
- 预算；
- 删除/清理路径；
- 当前 SDK/API 文档是否变化。

---

## 4. 当前交付标准

截至当前阶段，理想状态是：

> **如果用户随后完成充值并明确授权，只需配置凭证、重新核验 Gate、运行预设 dry-run，之后即可启用真实调用；不应再临时补 adapter、parser、chunking、异常处理或安全开关。**
