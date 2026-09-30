# Codex 执行契约：只做增量，不重复历史工作

## 1. 本文件优先级

执行本参考包时，本文件优先于“按文档逐项实现”的机械行为。

目标不是复现一套新的 ASR 工程，而是把 **privacy-aware cloud ASR** 增量接到已有心花项目上。

## 2. 必做的 reuse audit

在写代码、装模型、重跑 70 分钟 demo、改服务器环境之前，先核对：

- 当前工程根目录和 Git remote；
- 全部分支、最近提交与相关历史；
- `project/implementation/README.md` 或同类实现说明（如果路径已变化，搜索同类文件）；
- 现有 `scripts/`、`src/`、`pipeline/`、`outputs/`、`results/`、`logs/`；
- 服务器上模型缓存、venv/conda、Docker、CUDA 环境；
- 70 分钟 demo 是否已经完整跑过；
- 是否已有 VAD、diarization、forced alignment、timestamps；
- 是否已有云 API 试验、脱敏脚本、实体词表或人工校对文件；
- 实际输出文件是否存在，即使 README 未记录。

将结论写入：
`worklogs/API_NATIVE_REUSE_AUDIT.md`

建议表格：

| Capability | Evidence | Status | Action |
|---|---|---|---|
| server access | command/log/path | REUSE_AS_IS | none |
| Qwen ASR | model/env/output | REUSE_AS_IS | none |
| diarization | file/script | ADAPT_ONLY | map schema |
| privacy redaction | none found | MISSING | implement |

Status 只能使用：
`REUSE_AS_IS / ADAPT_ONLY / PARTIAL / MISSING / UNKNOWN`

`UNKNOWN` 不能直接转成 `MISSING`。

## 3. 禁止的重复行为

除非 audit 明确证明缺失或失效，不要：

- 重新搭 SSH、跳板机、VNC 或文件传输；
- 重新下载已有模型；
- 重建已经正常工作的 Python/CUDA 环境；
- 重做完整 70 分钟本地 baseline；
- 重做现有 speaker diarization；
- 重做已有时间戳/forced alignment；
- 把已有结果换目录后宣称“新实现”；
- 因为本包目录结构不同而迁移稳定代码；
- 删除、覆盖或重命名历史实验结果。

## 4. “适配优先”规则

如果已有模块功能正确但格式不同：

**只写 adapter。**

例如已有 diarization 输出：
`speaker,start_ms,end_ms`

而新重建模块需要：
`speaker,start_s,end_s`

正确做法是写转换层；不要重新跑 diarization。

## 5. 与历史结果并存

本路线是新实验臂，不覆盖已有本地 ASR 路线。

输出应明确分开，例如：

```
outputs/
  local_baseline/        # 原有，保持不动
  api_native/
    privacy/
    cloud/tencent/
    cloud/volcengine/
    reconstructed/
```

若已有输出规范，则沿用既有命名，只要能区分实验臂。

## 6. 真实数据的停机条件

以下任一条件不满足时，可以继续开发/mock，但不能上传真实心理咨询音频：

- Ethics Gate 未确认；
- Privacy Gate 未人工通过；
- Vendor Gate 未确认；
- API 凭证管理不安全；
- 无法确认是否用于训练/服务优化；
- 无法确认留存/删除条件。

## 7. 允许变通

本包指定的是“功能与 Gate”，不是强制框架。

允许 Codex：

- 复用现有语言/框架；
- 复用已有 provider SDK；
- 换用已有 alignment 工具；
- 根据服务器实际资源调整 chunk 大小；
- 若已有云 provider 更合规/更成熟，可把它作为首个实现；
- 若已有人工标注工具，可接入，而非新造 UI；
- 若现有 pipeline 已有 manifest/schema，可扩展而非另起一套。

但任何偏离都要在 run log 中写明：
“复用了什么、为什么没有照本包重新实现、对 Gate 是否有影响”。

## 8. 完成时应报告

最终给用户/Codex owner 报告：

1. 复用了哪些现有资产；
2. 新增了哪些最小组件；
3. 哪些工作明确没有重做；
4. 真实音频是否曾离开受控环境；
5. Gate 状态；
6. 70 分钟 demo 完整处理结果；
7. 质量对比；
8. 剩余风险与下一步。
