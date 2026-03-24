# CloudTrail 日志示例：SSRF 窃取 EC2 凭据 → IAM 提权 → SSM 横向移动

> **场景描述**：攻击者利用目标 Web 应用的 SSRF 漏洞，请求 EC2 实例元数据服务（`http://169.254.169.254/latest/meta-data/iam/security-credentials/`）窃取 Instance Profile 临时凭据，随后在外部机器上用该凭据给自身角色附加 `AdministratorAccess` 完成提权，最后通过 SSM `SendCommand` 在其他 EC2 实例上执行命令实现横向移动。  
> **凭据类型**：EC2 Instance Profile 临时凭据（`type: AssumedRole`，`ASIA` 前缀）  
> **三阶段**：SSRF 入云 → AttachRolePolicy 提权 → SendCommand 横向

---

## 攻击时序总览

| 时间偏移 | 阶段 | 事件 | eventSource | 可见性 |
|---|---|---|---|---|
| T+0:00 | 入云验证 | `GetCallerIdentity` | sts.amazonaws.com | ✅ 管理事件 |
| T+2:30 | IAM 提权 | `AttachRolePolicy` | iam.amazonaws.com | ✅ 管理事件 |
| T+9:15 | 横向移动 | `SendCommand` | ssm.amazonaws.com | ✅ 管理事件（命令内容不可见）|

---

## 阶段一：SSRF 入云——用窃取凭据验证身份

**事件**：`sts:GetCallerIdentity`

攻击者通过 SSRF 从 IMDS 拿到三元组（AK + SK + SessionToken）后，在自己的机器上配置凭据并调用 `GetCallerIdentity` 确认有效。

```json
{
  "Records": [
    {
      "eventVersion": "1.09",
      "userIdentity": {
        "type": "AssumedRole",
        "principalId": "AROAI3FHNJU4TPFNC7KGF:i-0a1b2c3d4e5f67890",
        "arn": "arn:aws:sts::847293016482:assumed-role/ec2-web-app-role/i-0a1b2c3d4e5f67890",
        "accountId": "847293016482",
        "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "sessionContext": {
          "sessionIssuer": {
            "type": "Role",
            "principalId": "AROAI3FHNJU4TPFNC7KGF",
            "arn": "arn:aws:iam::847293016482:role/ec2-web-app-role",
            "accountId": "847293016482",
            "userName": "ec2-web-app-role"
          },
          "ec2RoleDelivery": "1.0",
          "attributes": {
            "creationDate": "2025-11-14T05:03:11Z",
            "mfaAuthenticated": "false"
          }
        }
      },
      "eventTime": "2025-11-14T07:44:09Z",
      "eventSource": "sts.amazonaws.com",
      "eventName": "GetCallerIdentity",
      "awsRegion": "us-east-1",
      "sourceIPAddress": "45.142.212.18",
      "userAgent": "aws-cli/2.17.23 Python/3.12.2 Linux/6.1.0 botocore/2.4.5",
      "requestParameters": null,
      "responseElements": {
        "arn": "arn:aws:sts::847293016482:assumed-role/ec2-web-app-role/i-0a1b2c3d4e5f67890",
        "userId": "AROAI3FHNJU4TPFNC7KGF:i-0a1b2c3d4e5f67890",
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

**研判要点**：
- `sessionContext.ec2RoleDelivery: "1.0"` 确认凭据来自 IMDS，本应只在 EC2 实例内部使用
- `sourceIPAddress: 45.142.212.18` 是境外 VPS，与实例 IP（`10.0.3.47`）完全不符——凭据已被带出实例在外部使用
- Session Name `i-0a1b2c3d4e5f67890` 直接暴露了哪台 EC2 是 SSRF 的受害实例

---

## 阶段二：IAM 提权——给自身角色附加 AdministratorAccess

**事件**：`iam:AttachRolePolicy`

攻击者确认 `ec2-web-app-role` 具备 `iam:AttachRolePolicy` 权限后，直接给该角色附加 AWS 托管策略 `AdministratorAccess`，完成从普通应用角色到全账号管理员的提权。

```json
{
  "Records": [
    {
      "eventVersion": "1.09",
      "userIdentity": {
        "type": "AssumedRole",
        "principalId": "AROAI3FHNJU4TPFNC7KGF:i-0a1b2c3d4e5f67890",
        "arn": "arn:aws:sts::847293016482:assumed-role/ec2-web-app-role/i-0a1b2c3d4e5f67890",
        "accountId": "847293016482",
        "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "sessionContext": {
          "sessionIssuer": {
            "type": "Role",
            "principalId": "AROAI3FHNJU4TPFNC7KGF",
            "arn": "arn:aws:iam::847293016482:role/ec2-web-app-role",
            "accountId": "847293016482",
            "userName": "ec2-web-app-role"
          },
          "ec2RoleDelivery": "1.0",
          "attributes": {
            "creationDate": "2025-11-14T05:03:11Z",
            "mfaAuthenticated": "false"
          }
        }
      },
      "eventTime": "2025-11-14T07:46:41Z",
      "eventSource": "iam.amazonaws.com",
      "eventName": "AttachRolePolicy",
      "awsRegion": "us-east-1",
      "sourceIPAddress": "45.142.212.18",
      "userAgent": "aws-cli/2.17.23 Python/3.12.2 Linux/6.1.0 botocore/2.4.5",
      "requestParameters": {
        "roleName": "ec2-web-app-role",
        "policyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
      },
      "responseElements": null,
      "requestID": "c5d6e7f8-a9b0-1234-cdef-012345678902",
      "eventID": "f9a0b1c2-d3e4-5678-f012-345678901234",
      "readOnly": false,
      "eventType": "AwsApiCall",
      "managementEvent": true,
      "recipientAccountId": "847293016482"
    }
  ]
}
```

**研判要点**：
- `requestParameters.roleName` 与 `userIdentity.sessionContext.sessionIssuer.userName` 相同（均为 `ec2-web-app-role`）——**角色给自身附加最高权限**，是最强的单点告警信号
- `requestParameters.policyArn: arn:aws:iam::aws:policy/AdministratorAccess` 是 AWS 全权限托管策略，附加后该角色对账号内所有资源拥有完全控制权
- `responseElements: null` 是 IAM 写操作的正常现象，成功与否看有无 `errorCode` 字段——此处无 `errorCode`，即操作成功
- 三个阶段的 `accessKeyId` 完全相同（`ASIAIOSFODNN7EXAMPLE`），确认是同一组窃取凭据贯穿全程

---

## 阶段三：横向移动——SSM SendCommand 在其他实例执行命令

**事件**：`ssm:SendCommand`

提权完成后，攻击者使用同一组凭据（此时已具备 AdministratorAccess）向账号内其他 EC2 实例下发 SSM 命令。

> ⚠️ **关键盲点**：`SendCommand` 的命令内容（`Parameters.commands`）在 CloudTrail 中被标记为 `HIDDEN_DUE_TO_SECURITY_REASONS`，**攻击者实际执行了什么命令，CloudTrail 完全不可见**。如需查看命令内容，需到 AWS Systems Manager 控制台 → Run Command → Command History 中单独查看，或通过 SSM Agent 写入的 CloudWatch Logs 获取输出。

```json
{
  "Records": [
    {
      "eventVersion": "1.09",
      "userIdentity": {
        "type": "AssumedRole",
        "principalId": "AROAI3FHNJU4TPFNC7KGF:i-0a1b2c3d4e5f67890",
        "arn": "arn:aws:sts::847293016482:assumed-role/ec2-web-app-role/i-0a1b2c3d4e5f67890",
        "accountId": "847293016482",
        "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "sessionContext": {
          "sessionIssuer": {
            "type": "Role",
            "principalId": "AROAI3FHNJU4TPFNC7KGF",
            "arn": "arn:aws:iam::847293016482:role/ec2-web-app-role",
            "accountId": "847293016482",
            "userName": "ec2-web-app-role"
          },
          "ec2RoleDelivery": "1.0",
          "attributes": {
            "creationDate": "2025-11-14T05:03:11Z",
            "mfaAuthenticated": "false"
          }
        }
      },
      "eventTime": "2025-11-14T07:53:27Z",
      "eventSource": "ssm.amazonaws.com",
      "eventName": "SendCommand",
      "awsRegion": "us-east-1",
      "sourceIPAddress": "45.142.212.18",
      "userAgent": "aws-cli/2.17.23 Python/3.12.2 Linux/6.1.0 botocore/2.4.5",
      "requestParameters": {
        "instanceIds": [
          "i-0f9e8d7c6b5a43210",
          "i-0b1c2d3e4f5a67891"
        ],
        "documentName": "AWS-RunShellScript",
        "parameters": "HIDDEN_DUE_TO_SECURITY_REASONS",
        "timeoutSeconds": 3600,
        "comment": ""
      },
      "responseElements": {
        "command": {
          "commandId": "3f8b9c2d-1e4a-5f6b-7c8d-9e0f1a2b3c4d",
          "documentName": "AWS-RunShellScript",
          "documentVersion": "$DEFAULT",
          "comment": "",
          "expiresAfter": "Nov 14, 2025, 8:53:27 AM",
          "parameters": "HIDDEN_DUE_TO_SECURITY_REASONS",
          "instanceIds": [
            "i-0f9e8d7c6b5a43210",
            "i-0b1c2d3e4f5a67891"
          ],
          "targets": [],
          "requestedDateTime": "Nov 14, 2025, 7:53:27 AM",
          "status": "Pending",
          "outputS3BucketName": "",
          "outputS3KeyPrefix": "",
          "maxConcurrency": "50",
          "maxErrors": "0",
          "targetCount": 2,
          "completedCount": 0,
          "errorCount": 0,
          "deliveryTimedOutCount": 0,
          "serviceRole": "",
          "notificationConfig": {
            "notificationArn": "",
            "notificationEvents": [],
            "notificationType": ""
          }
        }
      },
      "requestID": "e7f8a9b0-c1d2-3456-ef01-234567890123",
      "eventID": "b3c4d5e6-f7a8-9012-bcde-f12345678901",
      "readOnly": false,
      "eventType": "AwsApiCall",
      "managementEvent": true,
      "recipientAccountId": "847293016482"
    }
  ]
}
```

**研判要点**：
- `requestParameters.parameters: "HIDDEN_DUE_TO_SECURITY_REASONS"` —— CloudTrail 主动编辑了命令内容，这是 AWS 的设计行为（防止密码等敏感参数泄露到日志），**并非攻击者的反检测操作**，防御方无法在 CloudTrail 中看到实际命令
- `requestParameters.documentName: "AWS-RunShellScript"` 是执行任意 Shell 命令的标准 SSM 文档，是横向移动最常用的方式
- `requestParameters.instanceIds` 包含两台实例（`i-0f9e8d7c6b5a43210`、`i-0b1c2d3e4f5a67891`），均不是来源实例（`i-0a1b2c3d4e5f67890`），确认是跨实例横向移动
- `responseElements.command.status: "Pending"` 表示命令已提交但尚未执行，CloudTrail 只记录调用行为，不记录命令执行结果
- **命令内容的获取路径**：SSM 控制台 → Run Command → Command ID `3f8b9c2d-...` → 查看详情；或配置 SSM Agent 将输出投递至 CloudWatch Logs

---

## SendCommand 盲点说明

```
CloudTrail 中可见                          CloudTrail 中不可见
─────────────────────────────────────────────────────────────
✅ 谁发起了 SendCommand                    ❌ 实际执行的命令内容
✅ 从哪个 IP 发起                          ❌ 命令的执行输出
✅ 目标实例列表                            ❌ 命令是否执行成功
✅ 使用的 SSM Document 名称               ❌ 命令在实例内产生的文件/进程变化
✅ commandId（可关联后续查询）             ❌ 攻击者在实例内获取了什么数据
```

**补全盲点的方法**：

1. **SSM Run Command History**（命令内容）  
   AWS 控制台 → Systems Manager → Run Command → Command History → 按 `commandId` 过滤

2. **CloudWatch Logs**（命令输出）  
   需提前在 `SendCommand` 时指定 `outputS3BucketName` 或配置 SSM Agent 的 CloudWatch 投递，默认不保存输出

3. **EC2 实例内 EDR / CloudWatch Agent**（进程行为）  
   `SendCommand` 执行的命令最终由 SSM Agent 以 `root` 或 `SYSTEM` 权限在实例内 `fork` 子进程，需要主机侧的进程监控才能捕获

---

## 三阶段关联分析

三条日志的关联字段一览：

| 字段 | 阶段一 | 阶段二 | 阶段三 |
|---|---|---|---|
| `accessKeyId` | `ASIAIOSFODNN7EXAMPLE` | `ASIAIOSFODNN7EXAMPLE` | `ASIAIOSFODNN7EXAMPLE` |
| `sourceIPAddress` | `45.142.212.18` | `45.142.212.18` | `45.142.212.18` |
| `sessionIssuer.arn` | `role/ec2-web-app-role` | `role/ec2-web-app-role` | `role/ec2-web-app-role` |
| `eventTime` 间隔 | — | +2m32s | +9m18s |
| 操作性质 | 只读（验证） | **写**（提权） | **写**（横向） |

同一 `accessKeyId` + 同一 `sourceIPAddress` 在 10 分钟内完成「验证 → 提权 → 横向」三步，是自动化攻击工具的典型时序特征。
