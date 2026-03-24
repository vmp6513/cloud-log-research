# AWS 云日志调研报告

> **文档状态**：阶段 3 产出 · 版本 1.0  
> **调研时间**：2026-03  
> **作者**：云安全专家调研  
> **适用读者**：规划侧 / 产品侧 / 安全运营团队  
> **核心聚焦**：AWS CloudTrail（AWS 云上行为审计的核心日志系统）

---

---

## 目录

1. [日志分类](#1-日志分类)
2. [能力边界](#2-能力边界)
3. [日志示例](#3-日志示例)
4. [影响基于日志进行告警研判的关键特性](#4-影响基于日志进行告警研判的关键特性)
5. [安全厂商经验项总结](#5-安全厂商经验项总结)

---

## 1. 日志分类

AWS 的日志体系分散在多个服务中，其中 **CloudTrail** 是核心审计日志，覆盖所有 AWS API 调用行为。除 CloudTrail 之外，还有专项日志（VPC Flow Logs、ALB Access Logs 等）用于网络和数据面可见性。本报告以 CloudTrail 为主，兼顾与告警研判密切相关的补充日志。

关键组件：Event History、Trails

### 1.1 CloudTrail 事件类型

CloudTrail 将所有事件分为 5 类，默认只开启管理事件，其余需额外配置和付费：

| 事件类型 | 默认开启 | 是否额外付费 | 简要描述与安全意义 |
|---|---|---|---|
| **管理事件（Management Events）** | ✅ 是 | 首条 Trail 免费 | 控制面操作：创建/删除资源、IAM 策略变更、安全配置修改、控制台登录等；是云安全检测的最核心数据源 |
| **数据事件（Data Events）** | ❌ 否 | ✅ 需付费 | 数据面操作：S3 对象的 GetObject/PutObject/DeleteObject、Lambda 函数调用、DynamoDB 行级操作等；高频、高量，按事件数计费 |
| **Insights 事件** | ❌ 否 | ✅ 需付费 | 基于 CloudTrail 对历史 API 调用速率/错误率建立基线，检测到异常时产生 Insights 事件；本质是内置的异常检测层 |
| **网络活动事件（Network Activity Events）** | ❌ 否 | ✅ 需付费 | VPC Endpoint 所有者视角下，记录通过私有 VPC Endpoint 向 AWS 服务发起的 API 调用，含被拒绝的请求；2023 年新增类型 |

### 1.2 CloudTrail 存储与查询方式

CloudTrail 提供三种方式访问历史事件，能力差异显著：

| 方式 | 保留时长 | 事件类型支持 | 查询能力 | 是否计费 |
|---|---|---|---|---|
| **Event History（事件历史）** | 90 天 | 仅管理事件 | 单属性过滤（每次只能用一个过滤条件），不支持跨 Region 查询 ![alt text](image.png) | 免费 |
| **Trail（追踪）→ S3** | 由 S3 生命周期策略决定，理论上无限 | 管理 + 数据 + Insights + 网络活动事件 | 需配合 Athena/CloudWatch Logs/SIEM 查询 ![alt text](image-1.png)，也具有 Region 性质 | Trail 本身免费，S3 存储收费；数据/Insights 事件按量计费 |
| **CloudTrail Lake（事件数据存储）** | 默认 1 年，可选 3 个月 ~ 10 年 | 管理 + 数据 + Insights + 网络活动 + 外部集成事件 | SQL 查询，支持跨账号、跨 Region 聚合；内置 AI 辅助查询（2024 年新增）| 按数据量 + 查询量计费 |

Athena: 交互式查询服务，能够使用 SQL 查询 S3 中的数据，按扫描的数据量收费。
CloudWatch Logs：CloudWatch 监控服务下处理文本（日志）的组件。

### 1.3 与告警研判相关的补充日志

CloudTrail 覆盖控制面，以下日志覆盖 CloudTrail 的盲点，在 AWS 安全监控中通常需要组合使用：

| 日志类型 | 来源服务 | 覆盖内容 | 对告警研判的价值 |
|---|---|---|---|
| VPC Flow Logs | Amazon VPC | 网络层流量元数据（五元组：源/目 IP、端口、协议、字节数、接受/拒绝） | 补充 CloudTrail 的网络盲点；检测横向移动中的异常内网流量 |
| ALB/NLB Access Logs | Elastic Load Balancing | HTTP/HTTPS 请求日志（含客户端 IP、URL、状态码、响应时间） | 检测 Web 层攻击（如撞库、扫描、WAF 绕过） |
| GuardDuty Findings | Amazon GuardDuty | 基于 CloudTrail + VPC Flow Logs + DNS Logs 的威胁检测结果 | 开箱即用的告警信号；是 SIEM 的重要输入 |
| AWS Config | AWS Config | 资源配置变更历史（与 CloudTrail 管理事件有重叠，但侧重资源状态而非 API 操作） | 合规基线管理，检测配置漂移 |
| Bedrock Model Invocation Logs | Amazon Bedrock | LLM 模型调用记录（需手动开启）| 检测 LLMjacking（云账号被劫持调用 AI 模型） |

> **参考文档**：
> - CloudTrail 事件类型总览：[Understanding CloudTrail events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-events.html)
> - CloudTrail 功能特性：[AWS CloudTrail Features](https://aws.amazon.com/cloudtrail/features/)
> - CloudTrail 概念：[CloudTrail concepts](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-concepts.html)

---

## 2. 能力边界

### 2.1 视角边界

**控制面全覆盖，数据面获取成本高。**

CloudTrail 的本质是"AWS API 调用日志"，即记录"谁、何时、从哪里、调用了什么 API、对哪个资源、结果如何"。这意味着：

| 可见 | 不可见 |
|---|---|
| 所有 AWS 控制面 API 调用（包括控制台、CLI、SDK、其他 AWS 服务） | 网络层流量内容（需 VPC Flow Logs，且仅有元数据，无 Payload） |
| IAM 操作：角色创建/附加策略/AssumeRole 调用链 | EC2 实例内部的进程行为（需 CloudWatch Agent / EDR） |
| S3 存储桶的 GetObject/PutObject（需开启数据事件，额外付费） | S3 中被访问/下载的对象内容本身 |
| Lambda 函数调用（需开启数据事件） | Lambda 函数执行时的内部代码行为（需 CloudWatch Logs） |
| 跨账号 AssumeRole 调用（含调用方账号） | 跨账号调用后在目标账号内部的数据访问细节（若目标账号未独立配置 Trail） |
| 控制台登录、MFA 使用情况 | 攻击者在本地端侧的行为（如下载的数据被如何处理） |
| AWS SSM Run Command 命令调用（触发事件，但命令参数被编辑，见 §2.2） | EC2 内 OS 层进程、文件系统操作（需 CloudWatch Agent） |

AssumeRole: 允许一个身份（用户、服务或账号）临时获取另一组特定权限 -> 获取 STS。

**其他限制**：CloudTrail 默认**不跨账号聚合**——每个 AWS 账号有自己独立的 CloudTrail，若攻击者从账号 A 横向移动至账号 B，两侧各自产生日志，**必须通过 Organization Trail 或手动汇聚才能关联分析**。

> **参考文档**：[What Is AWS CloudTrail?](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)

---

### 2.2 内容边界

**记录操作元数据，不记录操作载体。**

- **S3 数据事件**：记录 `GetObject`（含 Bucket 名、对象 Key、请求者 IP），但**不保存对象内容副本**。攻击者下载了哪些文件，CloudTrail 知道；文件里有什么，CloudTrail 不知道。
- **SSM Run Command**：CloudTrail 记录了 `SendCommand` 事件及调用者身份，但**命令参数（即实际执行的命令内容）被 CloudTrail 编辑标注为 `HIDDEN_DUE_TO_SECURITY_REASONS`**（因为可能包含敏感参数如密码）。要查看实际命令，需要到 Systems Manager 控制台的 Run Command 历史中单独查看。这是攻防中的重要盲点：攻击者若通过 SSM 在 EC2 上执行恶意命令，CloudTrail 中无法直接看到命令内容。也可以通过配置 SSM 日志流式传输将日志上报到 CloudWatch。
- **Lambda 数据事件**：记录 `Invoke` API（含调用者 ARN、函数名），但**不记录函数的 Event Payload（输入参数）及执行输出**（需 CloudWatch Logs）。
- **KMS 加解密**：记录 `Encrypt`/`Decrypt` 调用（含使用的 Key ID、请求者），但**不记录被加密/解密的数据明文**。攻击者使用你的 KMS Key 解密了什么数据，CloudTrail 中看不到。
- **SNS Publish**：消息内容字段在 CloudTrail 中显示为 `HIDDEN_DUE_TO_SECURITY_REASONS`。
- **EC2 UserData**：`RunInstances` 或 `ModifyInstanceAttribute` 传入的 UserData 脚本内容在 CloudTrail 中同样被编辑，仅显示 `<sensitiveDataRemoved>`。

> **参考文档**：  
> - [CloudTrail record contents – requestParameters truncation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html)  
> - SSM Run Command 参数编辑的实战案例：[AWS DFIR – IAM Privilege Escalation Leads to EC2 Ransomware (Medium, 2024)](https://medium.com/@adammesser_51095/cloud-digital-forensics-and-incident-response-aws-iam-privilege-escalation-leads-to-ec2-2d787a4e99a7)

SSM: 运维管理工具集合，包括会话管理、参数仓库、运行命令等核心组件。

---

### 2.3 日志完整性与可信度

#### 攻击者能否删除/篡改 CloudTrail 日志？

**结论：CloudTrail 日志一旦交付至 S3，可以被有权限的 IAM 身份删除；但 CloudTrail 提供了 Digest 文件机制用于完整性验证，且可通过 SCP 在组织级别禁止关闭 Logging 和删除 Trail。**

| 操作 | 是否可行 | 所需权限 | 注意事项 |
|---|---|---|---|
| 停止 Trail 日志记录（`StopLogging`） | **可以** | `cloudtrail:StopLogging` | 停止行为本身会被记录（如果有其他 Trail 在运行）；停止后新 API 调用不再产生日志 |
| 删除 Trail（`DeleteTrail`） | **可以** | `cloudtrail:DeleteTrail` | 删除前已交付至 S3 的日志文件不受影响，但未来的日志不再产生 |
| 删除 S3 中已有的日志文件 | **可以** | S3 `DeleteObject` 权限 | 若 S3 未配置版本控制或 Object Lock（WORM），攻击者可直接删除历史日志文件 |
| 篡改 S3 中日志文件内容后写回 | **可以**（若无 Object Lock） | S3 `PutObject` 权限 | CloudTrail Digest 文件机制可在事后检测到篡改，但无法阻止篡改发生 |
| 关闭 CloudTrail Digest 文件生成 | **可以** | `cloudtrail:UpdateTrail` 中的 `--no-enable-log-file-validation` | 关闭后，新产生的日志不再有对应 Digest，完整性无法验证；Chain of Digest 断裂 |

SCP: 一种账号级别的策略类型，集中管理组织所有账户的最大权限边界，优先级高于 IAM Policy。

**关键局限**：
- Digest 文件本身存储在同一个 S3 Bucket 中。若攻击者对 S3 Bucket 有写权限，**Digest 文件同样可被替换**（虽然伪造签名在计算上不可行，但删除 Digest 文件会导致对应时间段无法验证）
- 若 Trail 被停止或删除，停止期间**不产生 Digest 文件**，这段时间的日志有效性无法通过 Digest 链验证
- 最佳实践：将 CloudTrail 日志写入**独立的日志归档账号**（Log Archive Account），通过 SCP 限制非管理员对该账号 S3 的删除权限，并对 S3 Bucket 开启 **Object Lock（WORM 模式）**

#### 组织级防护：SCP 禁止 StopLogging 和 DeleteTrail

AWS Prescriptive Guidance 文档建议在 AWS Organizations 层面部署以下 SCP，从根本上阻止成员账号关闭 CloudTrail：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*",
      "Effect": "Deny"
    }
  ]
}
```

> **参考文档**：  
> - [Validating CloudTrail log file integrity](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-intro.html)  
> - [Security best practices in AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html)  
> - [Application logging and monitoring – CloudTrail (AWS Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/logging-monitoring-for-application-owners/cloudtrail.html)（含 SCP 模板）

---

### 2.4 日志保留策略与取证窗口

| 存储方式 | 默认保留 | 最长可保留 | 注意事项 |
|---|---|---|---|
| Event History（内置，免费） | **90 天** | 90 天（固定，不可延长） | 仅管理事件；取证窗口极为有限 |
| Trail → S3 | 由 S3 生命周期策略决定 | **理论上无限**（取决于 S3 存储成本意愿） | 必须手动创建 Trail；默认 Trail 设置日志永久保留（需手动设置生命周期归档） |
| CloudTrail Lake（Standard pricing） | **7 年**（默认） | 7 年 | 适合需要长期留存和快速 SQL 查询的场景 |
| CloudTrail Lake（1-year extendable pricing） | 1 年 | 10 年 | 成本更低，但 ingestion 费用相同 |

**高危场景**：许多中小型 AWS 用户从未显式创建 Trail，完全依赖 Event History 的 90 天免费记录。这意味着：
- 攻击发生后若超过 90 天才被发现，**历史日志已永久丢失**，且无法追溯
- 数据事件（如 S3 GetObject）不在 Event History 中，**若未提前开启 Trail，数据泄露事件的细节完全不可见**
- 所有 Trail 必须**手动创建**；AWS 默认不为新账号创建 Trail

> **参考文档**：
> - [How CloudTrail works](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/how-cloudtrail-works.html)  
> - [CloudTrail Lake pricing options](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake-manage-costs.html)

---

### 2.5 其他限制

**数据事件默认关闭且高成本——覆盖度取决于付费意愿**

CloudTrail 数据事件（如 S3 GetObject）按**每 100,000 次事件约 $0.10** 收费。对于大型企业，S3 数据事件一天可能产生数亿条记录，月度费用可能高达数千至数万美元。许多用户只开启了少数高价值 S3 Bucket 的数据事件，这意味着**日志覆盖度与预算直接相关**，安全厂商在评估客户环境时必须首先了解客户开启了哪些事件类型。

**CloudTrail 日志中的 IP 地址含义不一致**

`sourceIPAddress` 字段在不同场景下含义不同：
- 用户直接调用 API：该字段为客户端真实 IP
- 通过 AWS 控制台调用：该字段可能为 `signin.amazonaws.com` 或控制台服务器 IP，**而非用户真实 IP**
- 由另一个 AWS 服务代为调用（如 CloudFormation 触发 EC2）：该字段为 `cloudformation.amazonaws.com` 等服务域名，**而非发起 CloudFormation 请求的用户 IP**

这意味着分析 `sourceIPAddress` 时，必须结合 `userAgent` 字段综合判断请求来源。

**Global Service Events 的区域归集问题**

IAM、STS、AWS Organizations 等全局服务（Global Services）的事件**只会被记录到 `us-east-1`（弗吉尼亚北部）Region 的 Trail 中**，不会在其他 Region 的 Trail 里出现。如果仅在非 `us-east-1` 创建了单 Region Trail，则**完全看不到 IAM 相关操作**，这是极为常见的误配场景。

多 Region Trail（Multi-Region Trail）通过配置 `IncludeGlobalServiceEvents: true` 自动收集全局服务事件，是推荐的最佳实践。

P.S. 关于 Trail 的 Region 理解：
- Trail 关联的 S3 本身是一个具备 Region 属性的服务 -> 如果你在 ap-northeast-1 创建一个单 Region Trail，有人在 us-west-2 创建了一个 EC2，这个 Trail 不会记录任何日志。
- 全局服务的日志默认**只会投递到 `us-east-1`（弗吉尼亚北部）Region 的 Trail 中** -> 如果单 Region Trail 建在 ap-northeast-1，且没有开启“包含全局服务”选项，那么你连谁修改了你的管理员密码（IAM 操作）都查不到。
- 通过开启“包含全局服务”选项，本质上 AWS 做了两件事：
  - 自动克隆：它在 AWS 全球现有的（以及未来新增的）每一个区域都自动创建了一个配置完全一样的 Trail。
  - 归集投递：这些分散在世界各地的 Trail 拿到日志后，并不会存在当地，而是跨区域统一投递到你指定的那个 S3 中心桶里。

**④ requestParameters 和 responseElements 可能被截断**

单条 CloudTrail 事件默认上限为 **256KB**。当事件 Payload 超出限制时，`requestParameters` 或 `responseElements` 字段内容会被**静默截断**（不会产生错误或警告），可能导致关键参数丢失。CloudTrail Lake 支持将事件上限扩展至 **1MB**（需在 Event Data Store 上配置），但仍有截断风险。

什么情况下日志容易超限：
- 批量修改：同时给1000个用户设置属性，打标签。
- 复杂的授权变动：创建复杂的 IAM Policy，里面包含了大量的 Condition。

**⑤ Insights 事件存在最长 36 小时的启动延迟**

首次在 Trail 上开启 Insights 事件后，CloudTrail 需要最长 **36 小时**来建立 API 调用基线，才能开始产生异常检测结果。这对于新开户或新配置 Trail 的场景是需要提前告知的限制。

> **参考文档**：  
> - IP 字段语义：[CloudTrail record contents – sourceIPAddress](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html)  
> - Global Service Events：[Understanding multi-Region trails and opt-in Regions](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-multi-region-trails.html)  
> - Insights 启动延迟：[Delivery of Insights events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/insights-events-understanding.html)

---

## 3. 日志示例

以下示例均基于 AWS 官方文档中的结构模式构造，并标注了触发方式，便于测试和规则验证。

### 3.1 控制台登录（ConsoleLogin 管理事件）

**触发方式**：IAM 用户通过 AWS Management Console 登录

```json
{
  "Records": [{
    "eventVersion": "1.09",
    "userIdentity": {
      "type": "IAMUser",
      "principalId": "AIDACKCEVSQ6C2EXAMPLE",
      "arn": "arn:aws:iam::123456789012:user/alice",
      "accountId": "123456789012",
      "userName": "alice"
    },
    "eventTime": "2024-03-15T08:01:33Z",
    "eventSource": "signin.amazonaws.com",
    "eventName": "ConsoleLogin",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "203.0.113.55",
    "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)",
    "requestParameters": null,
    "responseElements": {
      "ConsoleLogin": "Success"
    },
    "additionalEventData": {
      "LoginTo": "https://console.aws.amazon.com/console/home",
      "MobileVersion": "No",
      "MFAUsed": "No"
    },
    "requestID": "1a2b3c4d-EXAMPLE",
    "eventID": "5e6f7a8b-EXAMPLE",
    "readOnly": false,
    "eventType": "AwsConsoleSignIn",
    "managementEvent": true,
    "recipientAccountId": "123456789012"
  }]
}
```

**研判意义**：`additionalEventData.MFAUsed: "No"` 表示登录未使用 MFA，是高风险信号。`responseElements.ConsoleLogin: "Failure"` 结合短时高频（同一用户/同一 IP）是密码爆破信号。`sourceIPAddress` 的地理位置异常（与用户历史 IP 偏差大）可触发异地登录告警。

---

### 3.2 IAM 角色扮演（AssumeRole 管理事件）

**触发方式**：用户或服务调用 `sts:AssumeRole` 扮演另一个角色；是横向移动检测的核心事件

```json
{
  "Records": [{
    "eventVersion": "1.09",
    "userIdentity": {
      "type": "IAMUser",
      "principalId": "AIDACKCEVSQ6C2EXAMPLE",
      "arn": "arn:aws:iam::111122223333:user/attacker",
      "accountId": "111122223333",
      "userName": "attacker"
    },
    "eventTime": "2024-03-15T09:15:00Z",
    "eventSource": "sts.amazonaws.com",
    "eventName": "AssumeRole",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "198.51.100.23",
    "userAgent": "aws-cli/2.13.0 Python/3.11.4",
    "requestParameters": {
      "roleArn": "arn:aws:iam::999988887777:role/OrganizationAccountAccessRole",
      "roleSessionName": "cross-account-session",
      "durationSeconds": 3600
    },
    "responseElements": {
      "credentials": {
        "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "expiration": "Mar 15, 2024 10:15:00 AM",
        "sessionToken": "AQoXnyc4lcK4w..."
      },
      "assumedRoleUser": {
        "assumedRoleId": "AROACKCEVSQ6C2EXAMPLE:cross-account-session",
        "arn": "arn:aws:sts::999988887777:assumed-role/OrganizationAccountAccessRole/cross-account-session"
      }
    },
    "requestID": "9EXAMPLE-EXAMPLE",
    "eventID": "c1d2e3f4-EXAMPLE",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "999988887777"
  }]
}
```

**研判意义**：`requestParameters.roleArn` 中目标账号（`999988887777`）与调用者账号（`111122223333`）不同，说明是**跨账号 AssumeRole**。`OrganizationAccountAccessRole` 是 AWS Organizations 自动创建的高权限角色（管理账号对成员账号的默认管理角色）——攻击者若能扮演此角色，即获得目标账号的管理员权限。`recipientAccountId` 与 `userIdentity.accountId` 不同，是识别跨账号横向移动的关键指标。

---

### 3.3 IAM 策略附加（AttachRolePolicy 管理事件）

**触发方式**：攻击者在提权过程中，向角色附加 AdministratorAccess 策略

```json
{
  "Records": [{
    "eventVersion": "1.09",
    "userIdentity": {
      "type": "AssumedRole",
      "principalId": "AROACKCEVSQ6C2EXAMPLE:lambda-session",
      "arn": "arn:aws:sts::123456789012:assumed-role/LambdaExecutionRole/lambda-session",
      "accountId": "123456789012",
      "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
      "sessionContext": {
        "sessionIssuer": {
          "type": "Role",
          "principalId": "AROACKCEVSQ6C2EXAMPLE",
          "arn": "arn:aws:iam::123456789012:role/LambdaExecutionRole",
          "accountId": "123456789012",
          "userName": "LambdaExecutionRole"
        },
        "attributes": {
          "creationDate": "2024-03-15T09:10:00Z",
          "mfaAuthenticated": "false"
        }
      }
    },
    "eventTime": "2024-03-15T09:12:30Z",
    "eventSource": "iam.amazonaws.com",
    "eventName": "AttachRolePolicy",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "192.168.1.100",
    "userAgent": "Boto3/1.28.0 Python/3.9.0",
    "requestParameters": {
      "roleName": "NewAdminRole",
      "policyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
    },
    "responseElements": null,
    "requestID": "a1b2c3d4-EXAMPLE",
    "eventID": "f5e4d3c2-EXAMPLE",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "123456789012"
  }]
}
```

**研判意义**：`sessionContext.sessionIssuer` 表明本次操作使用了由 `LambdaExecutionRole` 颁发的临时凭证（Lambda 函数执行时获得的 Session）——这是 Sysdig 2026 年案例中记录的典型提权路径：攻击者修改 Lambda 函数代码 → 调用 Lambda → Lambda Session 执行 `AttachRolePolicy` → 获得新的 AdminRole。`policyArn: AdministratorAccess` 是最高风险告警信号。

---

### 3.4 CloudTrail 停止记录（StopLogging 管理事件）

**触发方式**：攻击者获得高权限后关闭 CloudTrail，消除后续操作痕迹

```json
{
  "Records": [{
    "eventVersion": "1.09",
    "userIdentity": {
      "type": "AssumedRole",
      "principalId": "AROACKCEVSQ6C2EXAMPLE:attacker-session",
      "arn": "arn:aws:sts::123456789012:assumed-role/AdminRole/attacker-session",
      "accountId": "123456789012",
      "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
      "sessionContext": {
        "sessionIssuer": {
          "type": "Role",
          "arn": "arn:aws:iam::123456789012:role/AdminRole",
          "accountId": "123456789012",
          "userName": "AdminRole"
        },
        "attributes": {
          "creationDate": "2024-03-15T10:00:00Z",
          "mfaAuthenticated": "false"
        }
      }
    },
    "eventTime": "2024-03-15T10:05:00Z",
    "eventSource": "cloudtrail.amazonaws.com",
    "eventName": "StopLogging",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "198.51.100.99",
    "userAgent": "aws-cli/2.13.0",
    "requestParameters": {
      "name": "arn:aws:cloudtrail:us-east-1:123456789012:trail/organization-trail"
    },
    "responseElements": null,
    "requestID": "stop-EXAMPLE",
    "eventID": "stop-event-EXAMPLE",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "123456789012"
  }]
}
```

**研判意义**：`StopLogging` 是攻击者"清除痕迹"阶段的典型操作，应设置**最高优先级实时告警**。执行此操作后，该 Trail 覆盖的 Region 将不再产生新的 CloudTrail 日志，攻击者的后续操作对该 Trail 不可见。**若 SCP 已限制此操作，CloudTrail 中会出现 `AccessDenied` 错误，此时 AccessDenied + eventName=StopLogging 同样是高价值信号**。

---

### 3.5 S3 数据泄露（GetObject 数据事件）

**触发方式**：开启了 S3 数据事件的情况下，攻击者批量下载 S3 对象（需手动开启 Data Events，有额外费用）

```json
{
  "Records": [{
    "eventVersion": "1.09",
    "userIdentity": {
      "type": "IAMUser",
      "principalId": "AIDACKCEVSQ6C2EXAMPLE",
      "arn": "arn:aws:iam::123456789012:user/compromised-user",
      "accountId": "123456789012",
      "accessKeyId": "AKIAIOSFODNN7EXAMPLE",
      "userName": "compromised-user"
    },
    "eventTime": "2024-03-15T11:30:15Z",
    "eventSource": "s3.amazonaws.com",
    "eventName": "GetObject",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "203.0.113.88",
    "userAgent": "aws-cli/2.13.0 Python/3.11.4",
    "requestParameters": {
      "bucketName": "prod-customer-data",
      "key": "exports/customer_pii_2024Q1.csv"
    },
    "responseElements": null,
    "additionalEventData": {
      "x-amz-id-2": "EXAMPLE_AMZID",
      "x-amz-request-id": "EXAMPLEREQID",
      "bytesTransferredOut": 15728640
    },
    "requestID": "EXAMPLEID1234",
    "eventID": "s3-EXAMPLE",
    "readOnly": true,
    "resources": [{
      "type": "AWS::S3::Object",
      "ARN": "arn:aws:s3:::prod-customer-data/exports/customer_pii_2024Q1.csv"
    }, {
      "type": "AWS::S3::Bucket",
      "accountId": "123456789012",
      "ARN": "arn:aws:s3:::prod-customer-data"
    }],
    "eventType": "AwsApiCall",
    "managementEvent": false,
    "recipientAccountId": "123456789012"
  }]
}
```

**研判意义**：`additionalEventData.bytesTransferredOut` 可用于估算数据外泄量。`resources[0].ARN` 包含完整对象路径。短时间内同一身份/IP 产生大量此类事件（批量下载）、从高敏感 Bucket（通过 Bucket 名或标签识别）下载、以及下载来源 IP 超出正常地理范围，是数据泄露的关键告警信号。**注意**：若未提前开启该 S3 Bucket 的数据事件，此事件根本不存在于 CloudTrail 中。

---

### 3.6 供应链攻击入口：通过 CI/CD OIDC Token 扮演角色（AssumeRoleWithWebIdentity）

**触发方式**：攻击者利用被污染的 npm 包（如 UNC6426/Solorigate 类供应链攻击），在 CI/CD 环境中窃取 OIDC Token，再扮演 AWS 角色

```json
{
  "Records": [{
    "eventVersion": "1.09",
    "userIdentity": {
      "type": "WebIdentityUser",
      "principalId": "arn:token.actions.githubusercontent.com:repo:evil-org/malicious-repo:ref:refs/heads/main",
      "userName": "repo:evil-org/malicious-repo:ref:refs/heads/main",
      "identityProvider": "token.actions.githubusercontent.com"
    },
    "eventTime": "2024-03-15T12:00:00Z",
    "eventSource": "sts.amazonaws.com",
    "eventName": "AssumeRoleWithWebIdentity",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "185.220.101.45",
    "userAgent": "aws-cli/2.13.0",
    "requestParameters": {
      "roleArn": "arn:aws:iam::123456789012:role/GitHubActionsDeployRole",
      "roleSessionName": "GitHubActions",
      "webIdentityToken": "eyJ...(truncated)",
      "durationSeconds": 3600
    },
    "responseElements": {
      "credentials": {
        "accessKeyId": "ASIAIOSFODNN7EXAMPLE",
        "expiration": "Mar 15, 2024 1:00:00 PM",
        "sessionToken": "AQoDYXdzEJr..."
      },
      "assumedRoleUser": {
        "assumedRoleId": "AROACKCEVSQ6C2EXAMPLE:GitHubActions",
        "arn": "arn:aws:sts::123456789012:assumed-role/GitHubActionsDeployRole/GitHubActions"
      },
      "subjectFromWebIdentityToken": "repo:evil-org/malicious-repo:ref:refs/heads/main"
    },
    "requestID": "oidc-EXAMPLE",
    "eventID": "oidc-event-EXAMPLE",
    "readOnly": false,
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "123456789012"
  }]
}
```

**研判意义**：`userIdentity.identityProvider` = `token.actions.githubusercontent.com` 表明这是 GitHub Actions OIDC 登录。`responseElements.subjectFromWebIdentityToken` 包含了来源仓库信息，可与白名单对比——`evil-org/malicious-repo` 不在预期仓库列表中，是供应链攻击的关键信号。CSA 2026 年关于 UNC6426（nx npm 供应链攻击）的研究建议对所有 `AssumeRoleWithWebIdentity` 事件设置基于 `sub` Claim 的白名单告警。

---

## 4. 影响基于日志进行告警研判的关键特性

### 4.1 日志规模与客户等级的关联

CloudTrail 日志量的增长模式与 AWS 使用规模**非线性相关**，且受事件类型开启状态影响显著。

#### 单账号平均日志量参考

客户通常不会接入所有 AWS 账号，而是只接入包含核心业务、需要安全监控的若干个账号（如生产账号、核心数据账号）。因此**单账号的日志量**是规划采集容量的基本单位：

| 账号业务类型 | 服务特征 | 管理事件量（日/条，单账号） | 数据事件量（日/条，单账号） | 说明 |
|---|---|---|---|---|
| 轻量运维账号 | 少量 EC2/RDS，人工操作为主 | **数千 ~ 5 万** | 通常未开启 | 人工操作频率低，主要噪音来自自动化巡检脚本 |
| 标准业务账号 | ECS/EKS + RDS + S3，混合自动化 | **10 万 ~ 100 万** | 若开启核心 S3：百万级/天 | 自动化部署（CI/CD）和 Auto Scaling 事件是主要来源 |
| 高并发服务账号 | Lambda 密集调用 + API Gateway + S3 | **50 万 ~ 500 万** | 若开启 Lambda 数据事件：**亿级/天** | Lambda Invoke 数据事件量极大，几乎不可全量采集 |
| 数据/AI 平台账号 | Glue/EMR/Bedrock + 大量 S3 读写 | **100 万 ~ 1000 万** | 若开启 S3 数据事件：**十亿级/天** | S3 数据事件成本极高，必须按 Bucket 精细开启 |

> 以上为估算范围，实际量受自动化程度、服务数量、Region 数量影响。可在接入前通过 CloudTrail Event History 的 7 天历史数据估算客户管理事件基线量，再按比例推算月度规模。

#### 按客户规模的整体规模分层

客户接入的通常是**核心账号子集**，而非全量账号，实际汇聚到后台的日志量远低于全组织总量：

| 客户规模 | 总账号数 | 典型接入账号数 | 后台实际汇聚量（管理事件/日） | 关键研判挑战 |
|---|---|---|---|---|
| 小型企业 | 1–5 个 | 1–3 个 | 数万 ~ 数十万 | Trail 可能未提前创建；数据事件基本未开启，数据泄露事后调查困难 |
| 中型企业 | 10–100 个 | 3–15 个核心账号 | 百万 ~ 千万 | 跨账号关联需后台聚合；数据事件覆盖不均；部分账号 Trail 配置参差不齐 |
| 大型企业/集团 | 100+ 个 | 10–30 个核心账号 | 千万 ~ 亿级 | 多账号并发采集时限流风险；核心账号日志量大，需后台分片消费 |
| SaaS/技术型企业 | 5–50 个 | 3–10 个 | 高（但主要是管理事件） | Lambda 密集场景下数据事件几乎不可全量采集，建议仅开启高价值资源的数据事件 |

**关键经验**：数据事件成本本身往往成为采集决策的阻力，应设计分级策略——仅对含 PII、财务数据的核心 S3 Bucket 及关键 KMS Key 按需开启，而非全账号全量覆盖。

---

### 4.2 日志延迟

三套采集方案的端到端延迟特性各有差异，直接决定了上层告警的实时性。

| 采集方案 | 各阶段延迟拆解 | 端到端延迟 | 是否满足准实时定位（5–15 分钟） |
|---|---|---|---|
| **方案一**：Trail → S3 → SNS → SQS → 后台 | CloudTrail 写 S3（~5 分钟）+ SNS→SQS 推送（秒级）+ 后台消费（秒级） | **6–15 分钟** | ✅ 满足 |
| **方案二**：Trail → EventBridge → SQS → 后台 | CloudTrail 事件到达 EventBridge（~2–5 分钟）+ EventBridge→SQS（秒级）+ 后台消费（秒级） | **3–10 分钟** | ✅ 满足，略优于方案一 |
| **方案三**：Trail → S3 → 后台定时轮询 | CloudTrail 写 S3（~5 分钟）+ 后台轮询间隔（默认每 5 分钟） | **10–30 分钟** | ⚠️ 边界情况，取决于轮询频率 |
| Insights 事件（附加参考） | 异常检测计算 + 延迟交付 | **最长 30 分钟**；首次需 36 小时建立基线 | ❌ 不适合实时告警 |

**CloudTrail 本身的交付延迟是所有方案的共同底噪**，官方文档原文："CloudTrail typically delivers logs within an average of about 5 minutes of an API call. This time is not guaranteed."——这 5 分钟不可压缩，方案二通过跳过 S3 写入步骤略微降低了总延迟，但无法突破 CloudTrail 自身的处理时间。

**与 O365 UAL 的关键差异**：CloudTrail 的典型延迟（约 5 分钟）远低于 O365 UAL（60–90 分钟），意味着基于 CloudTrail 的告警可以做到分钟级响应，而 O365 UAL 更适合事后分析与取证。

**对研判引擎的要求**：三套方案的日志交付均**不保证有序**——同一时间段的事件可能分散在多个先后到达的消息或文件中。研判引擎需以 `eventTime` 为事件时间基准（而非消息到达时间），并配置至少 **15 分钟的乱序容忍窗口（Late Data Tolerance）**，防止因事件乱序导致告警链断裂。

> **参考文档**：[Working with CloudTrail log files – Log delivery](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-working-with-log-files.html)

---

### 4.3 是否支持流式拉取

三套方案在"数据流动模式"上有本质差异，直接影响后台的架构设计：

| 采集方案 | 数据流动模式 | 机制说明 | 对研判的影响 |
|---|---|---|---|
| **方案一**（Trail→S3→SQS） | **事件驱动 Pull** | S3 有新文件 → SNS 通知 → SQS → 后台主动拉取文件 | 非真正流式；每个 S3 文件包含多条事件（批量），处理粒度是"文件"而非"事件" |
| **方案二**（EventBridge→SQS） | **Push（近流式）** | 事件发生 → EventBridge 规则匹配 → 推入 SQS → 后台消费 | 处理粒度是**单条事件**，更接近流式；但仅支持管理事件，且消息体大小受 SQS 单条 256KB 限制 |
| **方案三**（Trail→S3 轮询） | **定时 Pull** | 后台每 N 分钟扫描 S3 路径，对比已拉取列表，增量下载新文件 | 批量处理，延迟最高；需后台自维护"拉取进度"状态，中断恢复依赖此状态 |

**对关联告警的具体影响**：

攻击链中的事件（如 `AssumeRole → AttachRolePolicy → GetObject`）可能跨越多条 SQS 消息（方案二）或多个 S3 文件（方案一/三）到达后台。方案二的单事件粒度虽然对流式处理更友好，但仅限管理事件，若攻击链中包含数据事件（如 `GetObject`），必须补充方案一或方案三才能看到完整链路。

**实践建议**：对于需要覆盖完整攻击链（含数据事件）的场景，推荐以**方案一为主采集通道**（全量管理 + 数据事件），对 IAM 高危操作等少数关键事件类型额外配置**方案二的 EventBridge 规则**，利用其低延迟特性实现快速告警，两路并行、互为补充。

---

### 4.4 是否支持过滤

三套方案在"过滤发生在哪个环节"上差异显著，决定了后台接收到的数据量和噪音水平：

#### CloudTrail 事件选择器（数据产生侧，三套方案共用的前置过滤）

CloudTrail 提供两种事件选择器，在**日志产生源头**就完成过滤——不产生的事件不写入日志、不计费、不占用任何采集带宽，是最彻底的过滤层：

| 选择器类型 | 过滤维度 | 典型用法 |
|---|---|---|
| **基础事件选择器** | 按读/写类型区分（ReadOnly / WriteOnly / All）；可指定 S3 Bucket、Lambda 函数、DynamoDB 表 | 仅对 3 个高价值 S3 Bucket 开启写事件数据日志 |
| **高级事件选择器** | 多维度字段过滤（`eventSource`、`eventName`、`resources.ARN`、`readOnly`），最多 500 个条件 | 仅记录 `iam.amazonaws.com` 的写操作；排除高频低价值的只读枚举 API |

> **与 O365 的关键差异**：AWS CloudTrail 的事件选择器在**数据产生侧**完成过滤，效果彻底。O365 Management Activity API 无此能力，必须全量拉取后在后台侧过滤，产生不必要的网络和计算开销。

#### 各方案自身的过滤能力

| 采集方案 | 是否支持采集侧过滤 | 过滤方式 | 灵活度 |
|---|---|---|---|
| **方案一**（Trail→S3→SQS） | ❌ 不支持 | 所有写入 S3 的日志文件均会产生 SNS 通知，无法在 S3/SNS 层面按事件内容过滤；后台拉取全量文件后自行解析过滤 | 低（粒度是文件，非事件） |
| **方案二**（EventBridge→SQS） | ✅ **支持，且最灵活** | EventBridge 规则支持按 `eventSource`、`eventName`、`userIdentity.type`、`sourceIPAddress` 等字段精确匹配，不匹配的事件不入队，后台零感知 | 高（单事件粒度，多维度组合） |
| **方案三**（Trail→S3 轮询） | ❌ 不支持 | 后台拉取全量 S3 文件后解析过滤，与方案一相同 | 低 |

**方案二 EventBridge 过滤规则示例**（仅推送 IAM 和 STS 高危写操作，过滤掉所有只读枚举）：

```json
{
  "source": ["aws.cloudtrail"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "readOnly": [false],
    "eventSource": ["iam.amazonaws.com", "sts.amazonaws.com", "cloudtrail.amazonaws.com"],
    "eventName": [
      "CreateUser", "DeleteUser", "AttachUserPolicy", "AttachRolePolicy",
      "PutRolePolicy", "CreateAccessKey", "DeleteAccessKey",
      "AssumeRole", "AssumeRoleWithWebIdentity",
      "StopLogging", "DeleteTrail", "UpdateTrail"
    ]
  }
}
```

此规则过滤后，一个中型企业账号的 SQS 消息量可从百万级/天降至**数百至数千条/天**，极大降低后台处理压力。

> **参考文档**：
> - [Logging data events – Advanced event selectors](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)
> - [Amazon EventBridge event patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)

---

### 4.5 限流策略

三套方案涉及的 AWS 接口限制各不相同：

| 接口 | 所属方案 | 限制 | 对后台的影响 |
|---|---|---|---|
| SQS `ReceiveMessage` | 方案一、方案二 | 几乎无上限（标准队列支持近乎无限并发消费） | 后台 Worker 数是瓶颈，不是 SQS |
| SQS `DeleteMessage` | 方案一、方案二 | 同上 | 注意批量删除（`DeleteMessageBatch`，单次最多 10 条）可降低请求次数 |
| S3 `ListObjectsV2` | 方案一、方案三 | 每个前缀 5500 TPS | 多账号汇聚时，并发扫描多个账号路径可能触发 503 SlowDown |
| S3 `GetObject` | 方案一、方案三 | 每个前缀 5500 GET/s | 大规模并发下载同一 Bucket 时需注意；建议实现指数退避重试 |
| EventBridge 规则触发 | 方案二 | 每账号每 Region 300 规则上限；触发频率无硬性限制 | 规则数量对一般客户不构成瓶颈 |
| `LookupEvents` API | 不建议用于任何方案 | **2 TPS**（每账号每 Region） | 仅供人工调查；禁止作为后台采集接口 |

**方案一/三 的 S3 限流踩坑**：多账号 Organization Trail 的日志写入同一 S3 Bucket 时，多个账号的路径在同一前缀层级下相邻，**大规模并发 ListObjects + GetObject 会触发 S3 的 503 SlowDown**（S3 每前缀默认 5500 GET/s）。建议后台对多账号采集任务做分片串行或限速并发，并实现指数退避重试。

**方案二的限流优势**：EventBridge→SQS 路径完全由 AWS 托管推送，后台消费 SQS 时不涉及 S3 扫描，**不存在 S3 前缀限流风险**，是多账号汇聚场景下限流压力最低的方案。

---

### 4.6 日志时序一致性

三套方案中均存在时序问题，但表现形式和影响程度不同。

#### 各方案的时间字段语义

| 方案 | 应使用的时间基准 | 易混淆的"假时间" | 偏差量 |
|---|---|---|---|
| **方案一**（S3 文件） | `Records[].eventTime` | S3 文件名中的时间戳（写入时间） | 约 5 分钟 |
| **方案一**（S3 文件） | `Records[].eventTime` | S3 Object `LastModified`（写入时间） | 约 5 分钟 |
| **方案二**（EventBridge 消息） | `detail.eventTime` | 消息外层 `time`（EventBridge 收到事件的时间） | 约 2–5 分钟 |
| **方案三**（S3 轮询） | `Records[].eventTime` | 后台拉取时间（`ingestion_time`） | 10–30 分钟（含轮询延迟） |

> **结论**：无论使用哪套方案，研判引擎必须统一以 `eventTime`（方案一/三）或 `detail.eventTime`（方案二）作为事件时间基准。方案二的消息外层 `time` 字段是 EventBridge 的系统时间，**不是事件发生时间**，直接用于关联分析会导致系统性偏差。

#### 跨方案混合使用时的时序对齐问题

当后台同时使用方案一（全量管理 + 数据事件）和方案二（高危事件快速通道）时，**同一事件可能被两路重复消费**（方案二的过滤事件本身也会写入 S3，方案一同样会拉取到）。后台必须以 `eventID` 做全局幂等去重，且两路消息的到达时间不同（方案二约 3–10 分钟，方案一约 6–15 分钟），**不能以消息到达时间判断事件先后**。

#### 对关联规则的影响

攻击链中的事件（如 `AssumeRole → AttachRolePolicy → GetObject`）可能横跨两套方案到达（前两步走方案二，`GetObject` 数据事件只有方案一）。研判引擎设计建议：

- 统一使用 `eventTime` 对齐，设置 **≥ 15 分钟**的事件关联窗口（需覆盖方案一的最大延迟）
- 以 `eventID` 去重后合并两路事件流，再输入关联规则引擎
- 高危单点事件（如 `StopLogging`、`AttachRolePolicy + AdministratorAccess`）走方案二的快速告警路径，无需等待关联窗口

---

### 4.7 多账号日志汇聚

客户只接入核心账号的场景下，汇聚的核心挑战集中在**跨账号关联可见性**和**每账号独立授权管理**两个层面。

#### 各方案在多账号场景下的汇聚模式

| 采集方案 | 多账号汇聚方式 | 后台工作量 | 注意事项 |
|---|---|---|---|
| **方案一**（Trail→S3→SQS） | 每个账号独立配置一套 Trail + S3 + SNS + SQS，后台维护多个 SQS 消费队列 | 较高（每账号一套配置） | 若客户使用 Organization Trail，多账号日志可写入同一 S3 Bucket 的不同子目录，后台只需对接一套 S3+SQS，但需处理动态子目录（新账号加入后路径自动出现） |
| **方案二**（EventBridge→SQS） | 每个账号独立配置 EventBridge 规则 + SQS；**无法跨账号共享单条 EventBridge 规则** | 中等（每账号需单独建规则） | 可用 CloudFormation StackSets 批量在多个成员账号中部署规则，降低运维成本 |
| **方案三**（Trail→S3 轮询） | 每账号独立 Trail + S3；后台按账号并发轮询各自的 S3 路径 | 中等（并发路径管理） | Organization Trail 同样可合并多账号日志到同一 S3，减少后台需轮询的 Bucket 数量 |

#### 跨账号横向移动的可见性挑战

这是多账号采集场景下最重要的安全检测问题：攻击者从账号 A（如开发账号）通过 `AssumeRole` 横向移动到账号 B（生产账号）时，**两个账号的 CloudTrail 日志相互独立**：

- 账号 A 的日志：记录了 `AssumeRole` 调用（`eventName=AssumeRole`，`requestParameters.roleArn` 指向账号 B 的 Role）
- 账号 B 的日志：记录了该 Session 在账号 B 内的所有操作，`userIdentity.accountId` 显示为账号 A，`recipientAccountId` 显示为账号 B

**识别跨账号操作的关键字段组合**：`recipientAccountId ≠ userIdentity.accountId`，这一条件成立即表明发生了跨账号调用。只有当账号 A 和账号 B 的日志都被后台汇聚，才能将"发起 AssumeRole 的原始身份"与"目标账号内的操作"串联为完整攻击链。**若客户只接入了账号 B 而未接入账号 A，攻击链的起点永久不可见**。

**给客户的建议**：在引导客户选择接入账号时，应明确说明：接入账号的完整性直接决定横向移动的可检测范围。至少应将**生产账号和 CI/CD 账号**同时接入——后者是供应链攻击的高频入口，且与生产账号之间通常存在 AssumeRole 信任关系。

> **参考文档**：
> - [Creating a trail for an organization](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html)
> - [AWS Security Reference Architecture – Log Archive Account](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/log-archive.html)
> - [Amazon EventBridge cross-account event delivery](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cross-account.html)
> - [Deploying with CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)

---

## 5. 安全厂商经验项总结

### 5.1 案例一：AI 辅助攻击在 8 分钟内拿到 AWS Admin（Sysdig，2026 年 2 月）

**事件背景**：

2025 年 11 月，Sysdig 威胁研究团队观察到一起真实攻击事件（2026 年 2 月公开披露）。攻击者从公开 S3 Bucket 中窃取了测试账号的 IAM 访问密钥，随后在 **8 分钟内**通过 Lambda 代码注入实现权限提升，横向移动至 19 个 AWS 身份（含 6 个 IAM 角色、14 个 Session），最终获得管理员权限并实施 LLMjacking（滥用受害者的 Amazon Bedrock 调用 Claude、DeepSeek 等 AI 模型）。

**攻击链（基于 CloudTrail 分析重建）**：

```
T+0:00  从公开 S3 Bucket 获取 IAM AccessKey (compromised_user)
T+0:06  失败尝试 AssumeRole (admin, Administrator) → AccessDenied x N
T+0:06  成功 AssumeRole (sysadmin, netadmin, account)
T+0:08  修改 Lambda 函数代码 (UpdateFunctionCode: EC2-init)
T+0:10  调用 Lambda 函数 (Invoke: EC2-init) → 函数 Session 执行 AttachRolePolicy
T+0:12  创建新 IAM Admin 用户并附加 AdministratorAccess
T+0:15  尝试扮演 OrganizationAccountAccessRole (跨账号横向移动)
T+0:30  访问 Secrets Manager、SSM 参数、S3 内部 Bucket
T+1:00  通过 Bedrock API 枚举并调用 LLM 模型 (LLMjacking 开始)
T+2:00  禁用 Bedrock 模型调用日志，删除部分 CloudTrail 日志（清除痕迹）
```

**CloudTrail 的检测能力分析**：

| 攻击步骤 | CloudTrail 中的可见性 | 检测难点 |
|---|---|---|
| 凭证泄露（S3 Bucket 中的 AccessKey） | **不可见**（发生在 AWS 控制面外） | 需要 S3 Access Log 或 GuardDuty 发现 |
| 大量 AccessDenied（权限探测） | **完全可见**（管理事件，errorCode=AccessDenied） | 单条无意义，需要高频率才触发告警 |
| AssumeRole 横向（6 个角色，14 个 Session） | **可见**（sts:AssumeRole 链） | 角色链（Role Chaining）在日志中难以直觉还原；需要 `sessionContext.sessionIssuer` 跟踪 |
| Lambda 代码注入（UpdateFunctionCode） | **可见**（管理事件），但命令内容不可见 | `requestParameters` 中有函数名，但注入的恶意代码内容需要 Lambda 日志才能看到 |
| IAM 提权（AttachRolePolicy + AdministratorAccess） | **完全可见**（管理事件） | 是最强的告警信号，但攻击总耗时仅 8 分钟，5 分钟的日志交付延迟意味着**告警时攻击可能已完成** |
| LLMjacking（Bedrock 模型调用） | **不可见（默认）**；需要开启 Bedrock Model Invocation Logging | 许多组织未开启此日志 |
| 禁用 Bedrock 日志 | **可见**（管理事件：`DeleteModelInvocationLoggingConfiguration`） | 是"清除痕迹"的高优先级信号 |

**对安全厂商的启示**：
- 8 分钟的攻击时间 vs. 5 分钟的日志延迟 + SIEM 处理时间，意味着**仅凭 CloudTrail 无法实时阻断此类攻击**，必须配合 GuardDuty 或云原生 CSPM/CWPP 的实时检测能力
- 密集的 `AccessDenied` 错误（短时间大量枚举）是攻击者侦察阶段的早期信号，应设置较低门槛的告警
- Lambda 函数的 `UpdateFunctionCode` + 随后的 `Invoke`（数据事件，需开启）是提权路径的关键事件组合
- **Bedrock Model Invocation Logging 必须开启**，LLMjacking 已成为云账号被攻陷后的高频后续操作

> **参考来源**：  
> - Sysdig TRT 原始报告：[AI-assisted cloud intrusion achieves admin access in 8 minutes (Sysdig, Feb 2026)](https://www.sysdig.com/blog/ai-assisted-cloud-intrusion-achieves-admin-access-in-8-minutes)  
> - The Register 报道：[AWS intruder pulled off AI-assisted cloud break-in in 8 mins (The Register, Feb 2026)](https://www.theregister.com/2026/02/04/aws_cloud_breakin_ai_assist/)

---

### 5.2 案例二：UNC6426 nx npm 供应链攻击 → AWS Admin（Google / CSA，2026 年 3 月）

**事件背景**：

2025 年 8 月，攻击者（UNC6426）通过 Pwn Request 技术（恶意 PR 触发 `pull_request_target` GitHub Actions Workflow 的权限提升漏洞），向 `nx` npm 包（React 生态中广泛使用的 Monorepo 工具，每周数百万下载量）注入恶意 postinstall 脚本。该脚本运行名为 QUIETVAULT 的凭证窃取器，利用系统中已安装的 LLM 工具扫描文件系统中的敏感数据，并将 GitHub PAT、AWS 密钥等外传。

Google 记录了一起目标企业员工运行含有恶意 Nx Console 插件的代码编辑器后，QUIETVAULT 在本地执行并外传了 CI/CD 环境变量（含 AWS OIDC Token 或长期 AccessKey），攻击者随后在 **72 小时内**获得目标 AWS 账号的 AdministratorAccess 权限。

**关键攻击路径（CloudTrail 可见部分）**：

```
1. QUIETVAULT 窃取 GitHub/AWS 凭证（本地发生，CloudTrail 不可见）
2. 攻击者用窃取的凭证调用 AssumeRoleWithWebIdentity（OIDC）
   → CloudTrail 事件：eventName=AssumeRoleWithWebIdentity
   → 关键字段：responseElements.subjectFromWebIdentityToken = "repo:evil-org/malicious-repo:..."
3. 使用 CI/CD 角色的权限部署新的 CloudFormation Stack（含 CAPABILITY_NAMED_IAM）
   → CloudTrail 事件：eventName=CreateStack（CloudFormation 管理事件）
4. CloudFormation Stack 创建新 IAM Role + 附加 AdministratorAccess
   → CloudTrail 事件：eventName=CreateRole、AttachRolePolicy（IAM 管理事件）
   → 这些 IAM 事件由 CloudFormation 服务代为执行，sourceIPAddress = cloudformation.amazonaws.com
5. 攻击者使用新 Admin 角色枚举 S3、终止 EC2/RDS 实例、解密 KMS 加密数据
6. 所有 GitHub 仓库被重命名为 /s1ngularity-repository-[random] 并公开
```

**CloudTrail 分析的关键难点**：

- **IAM 操作的真实来源被间接化**：步骤 4 中，CreateRole 和 AttachRolePolicy 的 `sourceIPAddress` 显示为 `cloudformation.amazonaws.com`，而非攻击者的真实 IP。必须向上追溯步骤 3 的 CreateStack 调用才能找到攻击者 IP，而这两步之间在日志时序上并不紧邻。
- **OIDC Token 中的仓库信息是最强的可检测信号**：`subjectFromWebIdentityToken` 字段中包含了来源仓库信息，若 CloudTrail 告警规则中包含了对此字段进行白名单匹配（只允许自己的 GitHub org 的仓库），攻击可在步骤 2 即被发现。CSA 的研究明确建议配置此类告警：
  ```
  alertOn: eventName = "AssumeRoleWithWebIdentity" AND
           responseElements.subjectFromWebIdentityToken NOT MATCHES "repo:your-org/*"
  ```
- **CloudFormation CAPABILITY_NAMED_IAM 是高风险操作**：CreateStack 时携带 `CAPABILITY_NAMED_IAM` 或 `CAPABILITY_IAM` 参数，意味着该 Stack 有权创建 IAM 身份。告警引擎应对此类 CreateStack 操作设置高优先级告警并要求人工审批。

**对安全厂商的启示**：
- 供应链攻击的入口点（恶意包执行）在 CloudTrail 中**完全不可见**，因为它发生在本地端侧。CloudTrail 只能从"凭证被盗后的云侧操作"开始提供可见性。
- OIDC 联合认证（GitHub Actions、GitLab CI、Jenkins 等）正在成为供应链攻击进入云账号的主要途径，`AssumeRoleWithWebIdentity` 的 `sub` Claim 白名单检测是当前**最关键的单点检测机会**。
- CloudFormation 的 IAM Capabilities（`CAPABILITY_NAMED_IAM`）使得攻击者可以用一次 API 调用完成整个提权链，且中间步骤（CreateRole、AttachPolicy）在 CloudTrail 中的 `sourceIPAddress` 指向 CloudFormation 服务，**传统的"谁调用了 AttachRolePolicy"的分析逻辑对此失效**，需要向上追溯 Stack 创建者。

> **参考来源**：  
> - The Hacker News 报道：[UNC6426 Exploits nx npm Supply-Chain Attack to Gain AWS Admin Access in 72 Hours (The Hacker News, Mar 2026)](https://thehackernews.com/2026/03/unc6426-exploits-nx-npm-supply-chain.html)  
> - CSA 研究简报：[UNC6426: nx Supply Chain to AWS Admin via OIDC (CSA Labs, Mar 2026)](https://labs.cloudsecurityalliance.org/research/briefing-csa-research-note-oidc-trust-chain-abuse-cloud-take/)  
> - AWS 安全博客（CloudTrail Lake 调查示例）：[Investigate security events by using AWS CloudTrail Lake advanced queries (AWS Security Blog, 2023)](https://aws.amazon.com/blogs/security/investigate-security-events-by-using-aws-cloudtrail-lake-advanced-queries/)

---

### 5.3 经验项总结

**① "默认安全"是陷阱：数据事件默认关闭意味着数据泄露的事后调查几乎不可能**

CloudTrail 管理事件只能告诉你控制面发生了什么，而 S3/Lambda/DynamoDB 的实际数据访问（GetObject、Invoke、Query）默认不记录。当发生数据泄露事件时，若未提前开启数据事件，安全团队将**无法确定泄露了哪些对象、通过哪个身份、从哪个 IP**。数据事件的开启必须在攻击发生前完成，这是"安全左移"的最基本要求之一。

**② Trail 不存在 ≠ 没有日志：Event History 的 90 天幻觉**

许多用户认为"控制台可以看日志，所以有 CloudTrail"，但 Event History（90 天免费管理事件）与 Trail（持久化存档）是完全独立的系统。事件发生 91 天后，Event History 中的记录自动消失且不可恢复。安全厂商应将"验证客户 Trail 存在并处于 Logging 状态"作为安全评估的第一步，且通过 `GetTrailStatus` 确认 `IsLogging: true`。

**③ 角色链（Role Chain）是攻防双方都必须深刻理解的特性**

AWS 允许无限深度的角色扮演链（A → B → C → D），每一层 AssumeRole 都会产生一个 CloudTrail 事件，且后续操作的 `userIdentity` 只反映最近一层角色的信息。要追溯到"最初的人类用户"，需要逐层向上解析 `sessionContext.sessionIssuer`。对于攻击者，角色链是混淆溯源的有效手段；对于防御方，CloudTrail Lake 的 SQL 查询能力（支持跨事件关联）是还原角色链的最有效工具。

**④ `AccessDenied` 密度是攻击者侦察阶段的早期指示器**

攻击者在获得初始凭证后的第一步通常是权限枚举（大量调用 Describe*/List*/Get* API），由于凭证权限受限，会产生密集的 `AccessDenied` 错误。Wiz 的云威胁狩猎文档指出，这是"威胁狩猎的金矿"。安全厂商应在研判引擎中设置针对 `errorCode=AccessDenied` 的高频聚合告警（如：单一身份在 5 分钟内产生超过 20 个不同 API 的 AccessDenied）。

**⑤ 攻击者关闭 CloudTrail 是"最后的机会"检测点，务必告警**

所有案例中，攻击者获得管理员权限后的共同操作之一是尝试关闭 CloudTrail（`StopLogging`、`DeleteTrail`）或删除 S3 中的日志文件。这是"最后一个被记录的高危操作"，告警延迟窗口极小——从操作产生到写入 S3 约 5 分钟，SIEM 处理约 5-10 分钟，给安全团队的响应时间不足 15 分钟。**实时性要求最高的告警之一，应走 CloudWatch Logs 实时路径而非 S3 批量路径。**

**⑥ 仅凭 CloudTrail 无法覆盖现代攻击的全攻击链**

三类关键盲点始终存在：（1）本地端侧的凭证窃取行为（需 EDR）；（2）EC2/容器内部的进程行为（需 CloudWatch Agent / GuardDuty Runtime）；（3）网络层的横向移动和数据外传（需 VPC Flow Logs）。CloudTrail 是"云控制面的完整记录"，但现代攻击的全攻击链横跨端侧、网络层和云控制面，**单一数据源的告警必然存在大量 False Negative**。GuardDuty 的价值在于它自动聚合 CloudTrail + VPC Flow Logs + DNS Logs，提供比单独分析 CloudTrail 更高的检测率。

---

*本文档所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有产品或定价变更，以 AWS 官方最新文档为准。*

---

**核心参考文档索引**

| 文档 | 链接 |
|---|---|
| CloudTrail 用户指南总览 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html |
| CloudTrail 事件类型 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-events.html |
| CloudTrail 概念 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-concepts.html |
| CloudTrail 功能特性 | https://aws.amazon.com/cloudtrail/features/ |
| CloudTrail 日志文件完整性验证 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-intro.html |
| CloudTrail 安全最佳实践 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html |
| CloudTrail 字段说明 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html |
| CloudTrail userIdentity 字段 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-user-identity.html |
| 管理事件日志说明 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-management-events-with-cloudtrail.html |
| 数据事件日志说明 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html |
| Insights 事件 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-insights-events-with-cloudtrail.html |
| Insights 事件交付 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/insights-events-understanding.html |
| LookupEvents API 限制 | https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/API_LookupEvents.html |
| CloudTrail 配额 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/WhatIsCloudTrail-Limits.html |
| 多 Region Trail | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-multi-region-trails.html |
| Organization Trail | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html |
| 应用日志监控（SCP 禁止 StopLogging） | https://docs.aws.amazon.com/prescriptive-guidance/latest/logging-monitoring-for-application-owners/cloudtrail.html |
| IAM 与 CloudTrail 集成 | https://docs.aws.amazon.com/IAM/latest/UserGuide/cloudtrail-integration.html |
| 日志文件查看 | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/get-and-view-cloudtrail-log-files.html |
| AWS CloudTrail Lake 高级查询（安全博客）| https://aws.amazon.com/blogs/security/investigate-security-events-by-using-aws-cloudtrail-lake-advanced-queries/ |
| Sysdig – 8 分钟 AI 辅助攻击 (Feb 2026) | https://www.sysdig.com/blog/ai-assisted-cloud-intrusion-achieves-admin-access-in-8-minutes |
| The Register – 8 分钟攻击报道 (Feb 2026) | https://www.theregister.com/2026/02/04/aws_cloud_breakin_ai_assist/ |
| The Hacker News – UNC6426 nx 供应链攻击 (Mar 2026) | https://thehackernews.com/2026/03/unc6426-exploits-nx-npm-supply-chain.html |
| CSA – UNC6426 OIDC 信任链滥用简报 (Mar 2026) | https://labs.cloudsecurityalliance.org/research/briefing-csa-research-note-oidc-trust-chain-abuse-cloud-take/ |
| Wiz – AWS 威胁狩猎最佳实践 | https://www.wiz.io/academy/detection-and-response/aws-threat-hunting |
| Splunk – AWS IAM 权限提升检测 | https://www.splunk.com/en_us/blog/security/aws-iam-privilege-escalation-threat-research-release-march-2021.html |
| DFIR 案例 – IAM 提权到 EC2 勒索软件（Medium, 2024）| https://medium.com/@adammesser_51095/cloud-digital-forensics-and-incident-response-aws-iam-privilege-escalation-leads-to-ec2-2d787a4e99a7 |