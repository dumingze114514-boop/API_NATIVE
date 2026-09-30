# Codex 执行契约：只做增量，不重复历史工作

## 1. 当前前提

模型服务器已经在对全部音视频执行全量 ASR。

因此：
- 不等待所有文件转录结束才开始；
- 对已完成 artifact 增量处理；
- 对新完成 artifact 持续接入；
- 不为了本任务重复已有全量 ASR。

## 2. 本文件优先级

目标不是复现新的 ASR 工程，而是把 privacy-aware cloud ASR 增量接到已有心花项目上。

## 3. 必做 reuse audit

在写代码、装模型、重跑文件、改服务器环境前，先核对：

- 当前工程根目录和 Git remote；
- 分支、提交和相关历史；
- 实现说明与实验日志；
- 正在运行的批量 ASR job；
- 已完成/失败/待处理文件列表；
- 现有 ASR artifact schema；
- 时间戳/forced alignment 能力；
- 模型缓存和运行环境；
- 是否已有脱敏、实体词典、局部重转录工具；
- 实际输出文件，即使 README 未记录。

输出：
`worklogs/API_NATIVE_REUSE_AUDIT.md`

状态：
`REUSE_AS_IS / ADAPT_ONLY / PARTIAL / MISSING / UNKNOWN`

`UNKNOWN` 不能按 `MISSING` 处理。

## 4. 禁止的重复行为

除非明确失效/缺失，不要：

- 重新搭服务器连接；
- 重新下载已有模型；
- 重建正常环境；
- 重跑已经有 artifact 的整文件 ASR；
- 重做与隐私任务无关的 benchmark；
- 重做 speaker diarization；
- 为符合本仓库结构迁移稳定代码；
- 覆盖历史输出。

## 5. ASR 质量问题必须继承，但只处理与当前任务相关的部分

Codex 必须读取历史 ASR 质量问题，不能假设已有文本准确。

当前 P0：

- 敏感实体误识别/漏识别；
- 人名、机构、地点、号码的错误；
- 会影响 Qwen70B 隐私判断的文本错误；
- 敏感 span 的精确时间戳；
- 文本 span 到音频的对齐；
- redaction 对相邻语音的误伤。

当前非优先：

- speaker diarization；
- speaker role 命名；
- 与隐私无关的普通错字；
- 标点和格式。

发现 ASR 问题时先问：

> 这个问题会不会影响敏感信息发现、边界定位或抹除后的正常语音？

会：处理。  
不会：记录，当前不优先。

## 6. 局部修复优于全量重跑

若敏感 span 附近转录不可靠：

- 截取局部音频；
- 调高质量设置局部重转录；
- 复用第二 ASR（若已有）；
- 进行人工确认。

不要因为几秒问题重跑整小时文件。

## 7. 适配优先

已有能力格式不同：

**只写 adapter。**

不为了 schema 重跑模型。

## 8. 说话人区分暂不作为阻塞条件

若 diarization 已存在，可以复用。

如果质量一般但不妨碍：

`敏感文字 -> 精确音频区间 -> 抹除`

则不修。

只有当说话人问题直接造成敏感信息漏检/边界错误时，才提升优先级。

## 9. 当前阶段：禁止实际云 API 调用

截至本仓库当前提交阶段：

- 不调用任何真实云端 ASR API；
- 不产生付费 API 请求；
- 不上传真实、脱敏或测试音频到云 ASR；
- 不以“用公开音频试一下”为理由绕过该冻结。

但必须完成全部调用前准备：

- provider adapter；
- 配置与 secret schema；
- request builder；
- chunk/global offset；
- dry-run；
- mock provider；
- response fixtures/parser；
- retry/timeout/error paths；
- Gate enforcement；
- cost/budget 参数；
- 删除/清理接口；
- 测试；
- 默认关闭 live API 的硬开关。

默认行为必须 fail closed。只有用户后续明确说明已经具备调用条件并授权启用时，才能解除 live API freeze。

## 10. 未来真实数据停机条件

以下任一不满足，可继续 mock/开发，但不能上传真实咨询音频：

- Ethics Gate；
- Privacy Gate；
- Vendor Gate；
- Security Gate；
- 敏感 span 的 transcript/alignment 存在未解决高风险异常。

## 11. 允许变通

允许：
- 复用已有框架；
- 复用已有 aligner；
- 调整 chunk；
- 替换 provider；
- 接已有人工审核工具；
- 扩展已有 manifest；
- 使用更适合现有服务器环境的局部修复方法。

偏离参考方案时写明理由和对 Gate 的影响。

## 12. 完成时报告

至少报告：

1. 当前全量 ASR 运行状态；
2. 已处理 artifact 数量；
3. 复用了哪些历史资产；
4. 新增哪些最小组件；
5. 哪些工作明确没有重做；
6. 隐私相关 ASR 质量问题；
7. Qwen70B 敏感信息识别表现；
8. timestamp/boundary 表现；
9. privacy leakage 与 collateral loss；
10. Gate 状态与剩余风险。
