# CloudTrail 安全日志示例：AKSK 泄露 → 创建 STS 临时凭证

## 场景描述

攻击者使用泄露的 IAM 用户 AKSK（长期凭证），调用 STS 服务创建一个有效期 36 小时的临时凭证，用于后续持久化访问或规避检测。

---

## 日志特征概述

| 阶段 | 事件名称 | 说明 |
|------|----------|------|
| 使用 AKSK 调用 STS | `GetSessionToken` 或 `AssumeRole` | 使用长期凭证申请临时 Token |

> 本场景攻击者直接使用 AKSK 调用 API（非控制台登录），**不会产生 `ConsoleLogin` 日志**，首条可见日志即为 STS 调用。

---

## 核心日志：调用 STS 创建临时凭证

> 攻击者使用泄露 AKSK 直接调用 `GetSessionToken`，指定 `DurationSeconds: 129600`（即 36 小时）

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDAEXAMPLEID123456",
    "arn": "arn:aws:iam::123456789012:user/leaked-user",
    "accountId": "123456789012",
    "accessKeyId": "AKIAEXAMPLELEAKEDKEY",
    "userName": "leaked-user"
  },
  "eventTime": "2024-03-20T09:15:33Z",
  "eventSource": "sts.amazonaws.com",
  "eventName": "GetSessionToken",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "198.51.100.77",
  "userAgent": "aws-cli/2.15.0 Python/3.11.6 Linux/5.15.0 botocore/2.0.0",
  "requestParameters": {
    "durationSeconds": 129600
  },
  "responseElements": {
    "credentials": {
      "accessKeyId": "ASIAEXAMPLESTSSESSION",
      "expiration": "2024-03-21T21:15:33Z",
      "sessionToken": "FwoGZXIvYXdzEN3//////////wEaDG...（已截断）"
    }
  },
  "requestID": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "eventID": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "123456789012"
}
```

---

## 关键字段说明

| 字段 | 值 | 说明 |
|------|----|------|
| `userIdentity.accessKeyId` | `AKIA...` | 前缀 `AKIA` 表示**长期 IAM 凭证**，即泄露的 AKSK |
| `eventSource` | `sts.amazonaws.com` | 调用 STS 服务 |
| `eventName` | `GetSessionToken` | 申请临时会话凭证 |
| `requestParameters.durationSeconds` | `129600` | 36 小时（**超出默认值，需重点关注**） |
| `responseElements.credentials.accessKeyId` | `ASIA...` | 前缀 `ASIA` 表示**生成的 STS 临时凭证** |
| `responseElements.credentials.expiration` | `2024-03-21T21:15:33Z` | 临时凭证到期时间，距调用时刻恰好 36 小时 |
| `sourceIPAddress` | `198.51.100.77` | 调用来源 IP，**非 AWS 内部 IP，应核查归属** |
| `userAgent` | `aws-cli/...` | 通过 CLI 调用，而非控制台 |

---

## 补充场景：若攻击者通过 `AssumeRole` 方式创建 STS

```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "principalId": "AIDAEXAMPLEID123456",
    "arn": "arn:aws:iam::123456789012:user/leaked-user",
    "accountId": "123456789012",
    "accessKeyId": "AKIAEXAMPLELEAKEDKEY",
    "userName": "leaked-user"
  },
  "eventTime": "2024-03-20T09:16:10Z",
  "eventSource": "sts.amazonaws.com",
  "eventName": "AssumeRole",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "198.51.100.77",
  "userAgent": "aws-cli/2.15.0 Python/3.11.6 Linux/5.15.0 botocore/2.0.0",
  "requestParameters": {
    "roleArn": "arn:aws:iam::123456789012:role/PrivilegedRole",
    "roleSessionName": "temp-session",
    "durationSeconds": 129600
  },
  "responseElements": {
    "credentials": {
      "accessKeyId": "ASIAEXAMPLEROLETOKEN",
      "expiration": "2024-03-21T21:16:10Z",
      "sessionToken": "FwoGZXIvYXdzEN7//////////wEaDF...（已截断）"
    },
    "assumedRoleUser": {
      "assumedRoleId": "AROAEXAMPLEROLEID:temp-session",
      "arn": "arn:aws:sts::123456789012:assumed-role/PrivilegedRole/temp-session"
    }
  },
  "requestID": "c3d4e5f6-a7b8-9012-cdef-012345678902",
  "eventID": "d4e5f6a7-b8c9-0123-defa-123456789012",
  "eventType": "AwsApiCall",
  "managementEvent": true,
  "recipientAccountId": "123456789012"
}
```

---

## GetSessionToken vs AssumeRole 对比

| 维度 | `GetSessionToken` | `AssumeRole` |
|------|-------------------|--------------|
| 使用场景 | 为当前 IAM 用户生成临时凭证 | 切换为其他角色身份 |
| 权限变化 | 与原 IAM 用户一致 | 继承目标 Role 的权限（可能权限提升）|
| 攻击意图 | 凭证转换，规避 AKSK 直接暴露 | 横向移动 / 权限提升 |
| 危险等级 | 中 | **高** |

---

## 检测规则建议

1. **超长有效期告警**：`durationSeconds` ≥ `43200`（12小时）即应触发告警，36小时（129600）属于严重异常
2. **AKIA 调用 STS 告警**：长期凭证（`AKIA` 前缀）调用 `GetSessionToken` / `AssumeRole` 应重点审计
3. **非常用 IP 调用**：`sourceIPAddress` 不在历史调用 IP 白名单内
4. **UserAgent 异常**：CLI 工具在非工作时间调用 STS 应告警
5. **关联追踪**：后续使用 `ASIA` 前缀凭证的所有 API 调用，应与此次 STS 事件关联分析