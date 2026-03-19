# CloudTrail 日志示例：IAM User AK/SK 泄露，API 调用 IP 异常

> **场景描述**：攻击者获取到 IAM User `deploy-ci` 的 AK/SK 后，从境外 IP 调用 `GetCallerIdentity` 验证凭据有效性。该行为是 AK/SK 泄露后攻击者的标准首步操作。  
> **凭据类型**：IAM 用户长期凭据（`userIdentity.type = IAMUser`）  
> **异常信号**：`sourceIPAddress` 与该用户历史 CI/CD 系统 IP 段（`10.0.0.0/8` 内网 + `52.94.0.0/16` AWS 服务）完全不符，来源为境外商业 VPS

---

## 日志正文

```json
{
  "Records": [
    {
      "eventVersion": "1.09",
      "userIdentity": {
        "type": "IAMUser",
        "principalId": "AIDAIOSFODNN7QW3RZPLT",
        "arn": "arn:aws:iam::847293016482:user/deploy-ci",
        "accountId": "847293016482",
        "accessKeyId": "AKIAIOSFODNN7QW3RZPL",
        "userName": "deploy-ci"
      },
      "eventTime": "2025-11-14T03:22:17Z",
      "eventSource": "sts.amazonaws.com",
      "eventName": "GetCallerIdentity",
      "awsRegion": "us-east-1",
      "sourceIPAddress": "45.142.212.18",
      "userAgent": "aws-cli/2.17.23 Python/3.12.2 Linux/6.1.0 botocore/2.4.5",
      "requestParameters": null,
      "responseElements": {
        "arn": "arn:aws:iam::847293016482:user/deploy-ci",
        "userId": "AIDAIOSFODNN7QW3RZPLT",
        "account": "847293016482"
      },
      "requestID": "a3f72b91-0c84-4d21-b9e3-6f2d831c0e17",
      "eventID": "e8b43ac1-77d2-4f09-b3a1-9c2f40d7e521",
      "readOnly": true,
      "eventType": "AwsApiCall",
      "managementEvent": true,
      "recipientAccountId": "847293016482"
    }
  ]
}
```

---

## 字段说明

| 字段 | 值 | 说明 |
|---|---|---|
| `userIdentity.type` | `IAMUser` | 长期凭据直接调用，非 AssumeRole 临时凭据 |
| `userIdentity.accessKeyId` | `AKIAIOSFODNN7QW3RZPL` | 泄露的原始 AK，前缀 `AKIA` 标识长期凭据 |
| `eventName` | `GetCallerIdentity` | 无需任何 IAM 权限，所有有效凭据均放行；攻击者标准验证步骤 |
| `sourceIPAddress` | `45.142.212.18` | **异常**：境外商业 VPS，与历史正常 IP 段不符 |
| `userAgent` | `aws-cli/2.17.23 ...` | **异常**：该 CI/CD 用户历史 userAgent 应为 `aws-sdk-go` 或 GitHub Actions runner |
| `awsRegion` | `us-east-1` | `sts:GetCallerIdentity` 属全局服务，日志固定归集至 `us-east-1` |
| `requestParameters` | `null` | `GetCallerIdentity` 无入参，正常现象 |
| `responseElements.account` | `847293016482` | 攻击者通过响应确认了目标账号 ID，为后续枚举提供基础 |

---

## 研判要点

**为什么 `GetCallerIdentity` 是高价值检测点**：该 API 不需要任何 IAM 权限，AWS 对所有持有有效凭据的调用者均返回成功——也就是说，攻击者用泄露的凭据调用它**不会触发 `AccessDenied`**，但 CloudTrail 无论如何都会记录。这使它成为凭据泄露后攻击者「静默探针」的最佳选择，也是防御方的最早检测机会。

**IP 异常的判断逻辑**：`deploy-ci` 是 CI/CD 服务账号，正常调用来源只有两类——GitHub Actions runner 的 IP 段（`140.82.0.0/16`）和内部构建机（`10.x.x.x`）。`45.142.212.18` 归属荷兰阿姆斯特丹的商业 VPS 服务商（Serverius），与任何正常来源无交集。

**`userAgent` 的辅助判断**：正常 CI/CD 流水线通常使用特定版本的 SDK（如 `aws-sdk-go/1.44.0`）。攻击者使用的 `aws-cli` + `botocore` 组合与该账号历史 `userAgent` 不符，可作为辅助信号，但 `userAgent` 可伪造，不能作为主要判据。

---

## 检测规则参考

```sql
-- CloudTrail Lake：检测 GetCallerIdentity 来源 IP 不在白名单内
SELECT
  userIdentity.userName,
  userIdentity.accessKeyId,
  sourceIPAddress,
  userAgent,
  eventTime
FROM cloudtrail_logs
WHERE
  eventName = 'GetCallerIdentity'
  AND userIdentity.type = 'IAMUser'
  AND sourceIPAddress NOT IN (
    '140.82.0.0/16',   -- GitHub Actions
    '10.0.0.0/8'       -- 内部构建机
  )
  AND eventTime > now() - interval '1' day
ORDER BY eventTime DESC
```

---

*日志结构基于 CloudTrail `eventVersion 1.09`，字段值为模拟生成，仅供检测规则测试使用。*