# Privacy Manifest 规范

## 1. 目的

`privacy_manifest.json` 是本地专用的时间轴脱敏记录，用于：

- 生成等长脱敏音频；
- 审核；
- 云结果回流后的 placeholder 重建；
- 审计。

**不得上传云端。**

## 2. 最小 schema

示例：

```json
{
  "audio_id": "XH_00017",
  "audio_sha256": "...",
  "duration_s": 4200.123,
  "redactions": [
    {
      "id": "r0001",
      "start_s": 31.10,
      "end_s": 32.25,
      "type": "PERSON_NAME",
      "placeholder": "[姓名]",
      "speaker": "CLIENT",
      "sources": ["local_asr", "ner", "manual"],
      "review_status": "approved"
    }
  ]
}
```

## 3. 建议实体类型

- `PERSON_NAME`
- `PHONE_NUMBER`
- `ID_NUMBER`
- `EMAIL`
- `SOCIAL_ACCOUNT`
- `PRECISE_ADDRESS`
- `ORGANIZATION_IDENTIFYING`
- `VEHICLE_PLATE`
- `THIRD_PARTY_IDENTITY`
- `RARE_REIDENTIFICATION_CONTEXT`
- `OTHER_HIGH_RISK`

项目可按现有 schema 调整，不要求完全照此枚举。

## 4. 不建议保存的内容

默认不要在 manifest 保存：

```json
"original_text": "张三"
```

如确有研究需要保存真实值：

- 单独加密；
- 与 manifest 分文件；
- 更严格 ACL；
- 不进 Git；
- 不进入云 adapter log。

manifest 只需要知道：
“哪里被遮、遮成什么 placeholder”。

## 5. 时间规则

- `start_s < end_s`
- 使用全局音频时间；
- overlap redactions 在生成音频前合并；
- 扩边后的范围裁剪到 `[0,duration]`；
- 不因 chunking 改写 manifest 时间；
- chunk 只保存自己的 `global_start_s`。

## 6. 音频脱敏校验

生成 redacted audio 后：

- sample rate 一致；
- channels 一致或有明确记录；
- sample count 一致；
- duration 一致；
- manifest 范围对应处已不可恢复听到敏感语音；
- 不通过简单“降低音量”保留可恢复人声。

## 7. 云结果回流

如果 manifest 有：

```
61.10–62.10 PERSON_NAME
```

云返回词序列中这一区间可能是：
- 空；
- 错字；
- 相邻词被粘连。

本地 reconstruction 不依赖云模型猜测真实内容，而应根据时间覆盖插入：

`[姓名]`

必要时对相邻词做时间/文本边界修正，但不得恢复真实实体。

## 8. Git 规则

可以提交：
- schema；
- 示例中的虚构数据；
- validation code。

禁止提交：
- 真实 audio id 与身份映射；
- 真实姓名；
- 真实电话/地址；
- 原始或脱敏后的真实咨询音频；
- API token。
