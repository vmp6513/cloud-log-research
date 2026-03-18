# AWS 云日志调研报告

> **文档状态**：阶段 3 产出 · 版本 1.0  
> **调研时间**：2026-03  
> **作者**：云安全专家调研  
> **适用读者**：规划侧 / 产品侧 / 安全运营团队  
> **核心聚焦**：AWS CloudTrail（AWS 云上行为审计的核心日志系统）

---

## 目录

1. [日志分类](#1-日志分类)
2. [能力边界](#2-能力边界)
3. [日志采集流程](#3-日志采集流程)
4. [日志示例](#4-日志示例)
5. [影响基于日志进行告警研判的关键特性](#5-影响基于日志进行告警研判的关键特性)
6. [安全厂商经验项总结](#6-安全厂商经验项总结)

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

身份安全重点关注管理事件这个事件类型中 **IAM 和控制台**相关的操作日志，Insights 事件也提供了一些根据管理事件聚合计算出来的异常操作日志，比如 IAM 相关 API 调用量突然升高，考虑到 Insights 是付费服务，优先级可以降低。

如果要做全链路攻防分析，数据事件是建立“控制面->数据面->端内部”攻击路径中关键节点，需要优先考虑。

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
- **SSM Run Command**：CloudTrail 记录了 `SendCommand` 事件及调用者身份，但**命令参数（即实际执行的命令内容）被 CloudTrail 编辑标注为 `HIDDEN_DUE_TO_SECURITY_REASONS`**（因为可能包含敏感参数如密码）。要查看实际命令，需要到 Systems Manager 控制台的 Run Command 历史中单独查看。这是攻防中的重要盲点：攻击者若通过 SSM 在 EC2 上执行恶意命令，CloudTrail 中无法直接看到命令内容。
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

## 3. 日志采集流程

AWS CloudTrail 日志的采集有两套主要路径，分别针对不同的实时性和分析需求：

| 方案 | 数据路径 | 推荐场景 |
|---|---|---|
| **方案 A**：Trail → S3 → 下游消费 | CloudTrail → S3 → SNS 通知 → SQS → SIEM 拉取 | 批量采集、低成本长期存储、SIEM 集成 |
| **方案 B**：Trail → CloudWatch Logs → 实时流 | CloudTrail → CloudWatch Logs → 订阅过滤器 → Kinesis / Lambda → SIEM | 实时告警、低延迟检测 |
| **方案 C**：CloudTrail Lake → SQL 查询 | CloudTrail Lake 直接存储 → SQL 查询 / 导出 | 事件调查、跨账号关联查询 |

### 3.1 方案 A：Trail → S3（主流 SIEM 接入方案）

#### 3.1.1 概要流程

```
1. 在目标账号（或组织根账号）创建 Multi-Region Trail
2. 配置 Trail 将日志交付至指定 S3 Bucket（可跨账号，推荐日志归档账号）
3. 开启 SNS 通知：Trail 每次将日志文件写入 S3 时，发送通知到 SNS Topic
4. SNS Topic 订阅 SQS Queue，SIEM 从 SQS 长轮询获取新日志文件通知
5. SIEM 解析 S3 对象路径，调用 S3 GetObject API 下载日志文件（gzip 压缩的 JSON）
6. 解压 JSON，解析 Records 数组，写入 SIEM 索引

  [CloudTrail API 调用]
       ↓ (~5 min)
  [S3 Bucket: 日志文件 .json.gz]
       ↓ (SNS 通知)
  [SNS Topic]
       ↓
  [SQS Queue]
       ↓ (SIEM 轮询)
  [SIEM / 分析引擎]
```

#### 3.1.2 核心 API

| 操作 | API / CLI | 说明 |
|---|---|---|
| 创建 Trail | `CreateTrail` | 指定 S3 Bucket、SNS Topic、是否多 Region、是否包含全局服务事件 |
| 开启日志记录 | `StartLogging` | Trail 创建后必须显式调用，否则不产生日志 |
| 查看 Trail 状态 | `GetTrailStatus` | 检查最新交付时间、最近交付错误、是否正在记录 |
| 配置事件选择器 | `PutEventSelectors` / `PutAdvancedEventSelectors` | 控制记录哪些事件类型（管理/数据/网络活动） |
| 下载日志 | S3 `GetObject` | 日志文件路径格式：`s3://{Bucket}/AWSLogs/{AccountId}/CloudTrail/{Region}/{YYYY}/{MM}/{DD}/` |
| 程序化查询（可选） | `LookupEvents` | 查询 Event History 中过去 90 天的管理事件；**仅限调查，不适合大批量采集**（TPS=2） |

#### 3.1.3 返回格式

日志以 **gzip 压缩的 JSON** 格式存储在 S3。每个文件包含一个 `Records` 数组，每条记录是一个完整的 CloudTrail 事件对象：

```json
{
  "Records": [
    {
      "eventVersion": "1.09",
      "userIdentity": { ... },
      "eventTime": "2024-03-15T10:22:31Z",
      "eventSource": "s3.amazonaws.com",
      "eventName": "GetObject",
      "awsRegion": "us-east-1",
      "sourceIPAddress": "203.0.113.45",
      "userAgent": "aws-cli/2.13.0",
      "requestParameters": { ... },
      "responseElements": null,
      "requestID": "EXAMPLE1234567",
      "eventID": "a1b2c3d4-...",
      "readOnly": true,
      "eventType": "AwsApiCall",
      "managementEvent": false,
      "recipientAccountId": "123456789012"
    },
    { ... }
  ]
}
```

**文件命名规范**：

```
{AccountId}_CloudTrail_{Region}_{YYYYMMDD}T{HHMM}Z_{UniqueString}.json.gz
```

日志文件按 Region 分目录存储，**不同 Region 的日志在 S3 中路径独立**，需按 Region 遍历下载。

#### 3.1.4 关键字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `eventTime` | string (ISO 8601 UTC) | **事件发生时间**，告警研判的核心时间基准 |
| `eventID` | GUID | 事件唯一标识，用于去重 |
| `eventName` | string | 被调用的 API 操作名（如 `AssumeRole`、`GetObject`、`DeleteBucket`），**告警规则的核心过滤字段** |
| `eventSource` | string | 被调用的 AWS 服务域名（如 `iam.amazonaws.com`、`s3.amazonaws.com`） |
| `awsRegion` | string | 事件发生的 AWS Region |
| `sourceIPAddress` | string | 请求来源 IP；**注意**：AWS 服务代调时为服务域名，不是真实用户 IP（见 §2.5②） |
| `userAgent` | string | 请求客户端标识，可区分 Console、CLI、SDK 调用 |
| `errorCode` | string | API 调用失败的错误码（如 `AccessDenied`、`NoSuchBucket`）；成功时字段缺失 |
| `errorMessage` | string | 错误详情；`AccessDenied` 类错误可揭示攻击者的枚举意图 |
| `readOnly` | boolean | true=只读操作（Describe/List/Get），false=写操作（Create/Delete/Update）；可用于快速过滤高风险操作 |
| `managementEvent` | boolean | true=管理事件，false=数据事件 |
| `requestParameters` | object | API 调用的入参（如目标资源 ARN、参数值）；**包含关键上下文信息，但可能被截断** |
| `responseElements` | object | API 调用的返回值；对写操作有意义（如创建的资源 ARN），只读操作为 null |
| `recipientAccountId` | string | 接收事件的账号 ID；跨账号调用时与 `userIdentity.accountId` 不同，**是检测跨账号横向移动的关键字段** |

##### userIdentity 子字段（最重要的身份溯源字段）

| 子字段 | 说明 |
|---|---|
| `userIdentity.type` | 调用者类型：`IAMUser`、`AssumedRole`、`Root`、`AWSService`、`SAMLUser`、`WebIdentityUser` 等 |
| `userIdentity.arn` | 调用者的完整 ARN；对于 `AssumedRole` 类型，这是 STS Session ARN（含 Session Name），而非 IAM Role ARN |
| `userIdentity.accountId` | 调用者所在账号 ID；跨账号调用时为发起账号 |
| `userIdentity.accessKeyId` | 使用的 Access Key ID；可用于追踪特定临时凭证的所有操作 |
| `userIdentity.sessionContext.sessionIssuer` | 若使用了临时凭证（AssumeRole），记录**颁发此 Session 的原始身份**（如 EC2 Instance Role、原始 IAM Role）；是追踪 Role Chain 的关键 |
| `userIdentity.sessionContext.attributes.mfaAuthenticated` | 是否使用了 MFA；`false` 且发生高危操作时应重点关注 |

> **参考文档**：
> - [CloudTrail record contents](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html)
> - [CloudTrail userIdentity element](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-user-identity.html)
> - [Getting and viewing CloudTrail log files](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/get-and-view-cloudtrail-log-files.html)

#### 3.1.5 所需权限

**创建和管理 Trail 所需权限（安全团队）**：

| 权限 | 用途 |
|---|---|
| `cloudtrail:CreateTrail` / `UpdateTrail` / `StartLogging` | 创建和配置 Trail |
| `cloudtrail:PutEventSelectors` | 配置事件选择器（开启数据事件等） |
| `s3:CreateBucket` / `s3:PutBucketPolicy` | 创建 CloudTrail 日志桶并配置策略 |
| `sns:CreateTopic` / `sns:SetTopicAttributes` | 创建 SNS 通知 |

**SIEM/采集服务读取日志所需权限（最小权限原则）**：

| 权限 | 用途 |
|---|---|
| `s3:GetObject` / `s3:ListBucket` | 读取日志文件 |
| `sqs:ReceiveMessage` / `sqs:DeleteMessage` | 消费 SQS 通知队列 |
| `kms:Decrypt` | 若日志启用了 KMS 加密（推荐），解密日志内容 |

#### 3.1.6 安全厂商使用策略与踩坑经验

**Global Service Events 只在 us-east-1 产生，是最常见的漏采场景**

IAM 相关操作（`CreateUser`、`AttachUserPolicy`、`AssumeRole` 等）是特权升级检测的核心，但这些全局服务事件**只存在于 `us-east-1` 的 Trail 日志路径中**。如果安全厂商仅采集客户明确告知的 Region，将完全遗漏 IAM 操作。建议在采集配置中**强制包含 `us-east-1`** 或使用 Multi-Region Trail（自动包含全局事件）。

**S3 Bucket 同名事件可能来自多 Region**

同一个 S3 Bucket 的操作可能被多个 Region 的 Trail 记录（如数据面访问在数据所在 Region，控制面操作可能在其他 Region）。去重时需同时使用 `eventID`，**不能仅靠时间+操作名去重**。

**日志文件命名中的时间戳是交付时间，不是事件时间**

文件名中的时间戳（`20240315T2210Z`）是**日志文件被写入 S3 的时间**，而非文件中事件的发生时间。一个文件中可能包含事件发生时间比文件时间戳早 5-15 分钟的事件。如果以文件时间戳作为"事件时间"进行告警，会产生系统性的时间偏差。

**Organization Trail 需要在管理账号中创建，并由管理账号统一配置**

使用 AWS Organizations 的企业可以创建 Organization Trail，一次性覆盖所有成员账号。但其有一个重要限制：**成员账号无法单独修改 Organization Trail 的配置**，且 Organization Trail 交付的日志在 S3 中按 OU + 账号 ID 分目录存储，采集逻辑需要支持动态目录扫描（随着新账号加入 Organization，新目录会自动出现）。

**`LookupEvents` API 不适合大规模采集**

`LookupEvents` 接口有两个严重限制：（1）TPS 限制为**每秒 2 次**，远不足以支撑大批量采集；（2）只能查询过去 **90 天**的管理事件。它设计用于人工调查，**不应被用作 SIEM 的日志采集接口**。真正的大规模采集应从 S3 拉取。

> **参考文档**：
> - [Quotas in AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/WhatIsCloudTrail-Limits.html)
> - [LookupEvents API Reference](https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/API_LookupEvents.html)
> - [Creating a trail for an organization](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html)

---

### 3.2 方案 B：Trail → CloudWatch Logs（近实时告警）

#### 3.2.1 概要流程

```
1. 在 Trail 配置中开启 CloudWatch Logs 集成
2. 为 CloudTrail 创建专用 IAM Role（允许向 CloudWatch Logs 写入）
3. CloudTrail 将事件近实时（通常 < 5 分钟）推送至 CloudWatch Log Group
4. 在 Log Group 上配置 Metric Filters（告警规则） + CloudWatch Alarms
5. 或通过 Subscription Filter 将事件推送至 Kinesis / Lambda 进行实时分析
```

此方案延迟更低，但 **CloudWatch Logs 存储成本较 S3 高**，不适合长期全量留存，通常用于"实时告警"场景，配合 S3 方案做"长期存档"。

#### 3.2.2 所需权限

- CloudTrail 需要 IAM Role，含 `logs:CreateLogGroup`、`logs:CreateLogStream`、`logs:PutLogEvents` 权限
- 消费方（SIEM/Lambda）需 `logs:FilterLogEvents` 或 Kinesis 消费权限

> **参考文档**：[Sending events to CloudWatch Logs](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/send-cloudtrail-events-to-cloudwatch-logs.html)

---

## 4. 日志示例

以下示例均基于 AWS 官方文档中的结构模式构造，并标注了触发方式，便于测试和规则验证。

### 4.1 控制台登录（ConsoleLogin 管理事件）

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

### 4.2 IAM 角色扮演（AssumeRole 管理事件）

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

### 4.3 IAM 策略附加（AttachRolePolicy 管理事件）

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

### 4.4 CloudTrail 停止记录（StopLogging 管理事件）

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

### 4.5 S3 数据泄露（GetObject 数据事件）

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

### 4.6 供应链攻击入口：通过 CI/CD OIDC Token 扮演角色（AssumeRoleWithWebIdentity）

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

## 5. 影响基于日志进行告警研判的关键特性

### 5.1 日志规模与客户等级的关联

CloudTrail 日志量的增长模式与 AWS 使用规模**非线性相关**，且受事件类型开启状态影响显著。

| 客户规模 | 账号/服务特征 | 预计管理事件量（日/条） | 预计数据事件量（日/条） | 关键研判挑战 |
|---|---|---|---|---|
| 小型企业 | 1-5 个 AWS 账号，少量服务 | 数千 ~ 数万 | 通常未开启（成本顾虑） | Trail 可能未创建；数据事件盲区严重 |
| 中型企业 | 10-100 个账号，使用 EKS/S3/Lambda | 百万级/账号 | 若开启，可达亿级/天 | 跨账号关联困难；数据事件覆盖不均 |
| 大型企业/集团 | 100+ 账号，使用 Organization Trail | 十亿级/天（聚合） | 极高（若全量开启） | API 限流成为瓶颈；需大规模并行采集架构；数据事件成本可能超越存储预算 |
| SaaS/技术型企业 | Lambda 高频调用，S3 高访问量 | 中等（管理事件不高） | **极高**：Lambda Invoke 数据事件可达每秒数百万 | Lambda 数据事件几乎不可全量采集；需在事件选择器层面过滤 |

**关键经验**：对于高频 Lambda 调用或 S3 操作的客户，**数据事件成本本身就可能成为采集决策的阻力**。安全厂商应设计分级策略：仅对高价值 S3 Bucket（含 PII、财务数据等）开启数据事件，而非全账号覆盖。

---

### 5.2 日志延迟

CloudTrail 的交付延迟在 AWS 所有日志服务中属于**中等偏低**，但存在多个延迟叠加点：

| 阶段 | 典型延迟 | 说明 |
|---|---|---|
| API 调用 → CloudTrail 日志写入 S3 | **平均约 5 分钟**（不保证） | 官方文档原文："CloudTrail typically delivers logs within an average of about 5 minutes of an API call. This time is not guaranteed." |
| S3 写入 → SNS 通知 → SQS → SIEM 消费 | 额外 1-2 分钟 | 取决于 SNS/SQS 的处理延迟和 SIEM 轮询间隔 |
| CloudTrail → CloudWatch Logs | **接近 5 分钟**（与 S3 路径类似） | 通常略快于 S3 路径，但差异不显著 |
| Insights 事件 | **30 分钟内**交付 | 比普通事件延迟更大，且首次需 36 小时建立基线 |

**与 O365 UAL 的关键差异**：CloudTrail 的延迟（约 5 分钟）远低于 O365 UAL（60-90 分钟）。这意味着 AWS CloudTrail **可以支持近实时告警**（告警延迟在 10 分钟量级），而 O365 UAL 更适合事后分析。

**对研判引擎的要求**：由于日志交付不保证有序（同一个时间段的事件可能分散在多个先后到达的日志文件中），研判引擎需要支持**乱序事件的时间窗口处理**，建议以 `eventTime` 为基准，配置至少 15 分钟的 Late Data Tolerance。

> **参考文档**：[Working with CloudTrail log files – Log delivery](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-working-with-log-files.html)

---

### 5.3 是否支持流式拉取

**结论：原生 Trail → S3 是批量 Pull 模式，非真正流式；通过 CloudWatch Logs 订阅可实现接近流式的推送。**

| 方案 | 模式 | 延迟 | 适用场景 |
|---|---|---|---|
| Trail → S3 + SNS/SQS | Pull（事件驱动轮询） | 约 5-15 分钟端到端 | 主流 SIEM 接入，成本低 |
| Trail → CloudWatch Logs → Subscription Filter → Kinesis / Lambda | Push（接近流式） | 约 3-8 分钟端到端 | 实时告警场景，成本略高 |
| EventBridge + 直接触发 Lambda | Push（数据事件可用） | 秒级（仅限已配置数据事件的服务） | 需要秒级响应的特定场景（如 S3 高价值 Bucket） |

**对研判的影响**：在攻击者"8 分钟拿到 AWS Admin"（Sysdig 2026 年案例）的场景下，5 分钟延迟意味着告警引擎**几乎与攻击实时赛跑**。建议对高危事件（如 `AttachRolePolicy + AdministratorAccess`、`StopLogging`、`AssumeRole + 跨账号`）单独走 CloudWatch Logs 实时路径，其余事件走 S3 批量路径。

---

### 5.4 是否支持 API 过滤

CloudTrail 提供两种事件选择器，允许在**数据源侧**进行精细过滤，大幅减少不必要的日志产生量和成本：

| 过滤方式 | 能力 | 典型用法 |
|---|---|---|
| **基础事件选择器（Basic）** | 按读/写事件类型区分（Read/Write），支持 S3 Bucket、Lambda 函数、DynamoDB 表的精确指定 | 仅对 3 个高价值 S3 Bucket 开启写事件，其余不开启 |
| **高级事件选择器（Advanced）** | 支持多维度字段过滤：`eventSource`、`eventName`、`resources.ARN`、`readOnly`，最多 500 个条件 | 仅记录 `iam.amazonaws.com` 的写操作；排除某个噪音 API |

**重要优势（对比 O365）**：AWS CloudTrail 的 Advanced Event Selectors **在数据产生侧就完成过滤**，不产生的事件不写入日志、不进行计费、不占用采集带宽。O365 Management Activity API 则必须全量拉取后在业务侧过滤。这对于成本控制和大规模部署有重要价值。

**安全厂商的应用**：建议在客户环境中部署时，配置至少两条 Trail：一条全量记录管理事件（低成本基础覆盖），另一条针对高价值资源（核心 S3、KMS Key、生产 IAM Role）开启数据事件（精准覆盖）。

> **参考文档**：  
> - [Logging data events – Advanced event selectors](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)  
> - [Logging management events](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-management-events-with-cloudtrail.html)

---

### 5.5 限流策略

| 接口 | 限制 | 影响 |
|---|---|---|
| `LookupEvents` API | **2 TPS**（每秒 2 次，每账号每 Region） | 只适合人工调查，大规模采集不可用 |
| S3 `GetObject`（日志下载） | S3 无固定 TPS 限制，但受 S3 前缀级别的请求速率影响 | 大量并发下载时可能触发 S3 503 SlowDown |
| CloudWatch Logs 订阅 | CloudWatch Logs `FilterLogEvents` 5 TPS，`GetLogEvents` 10 TPS | 高并发日志消费时需注意 |
| CloudTrail Lake 查询 | 并发查询数量有限制（具体视区域而定） | 大量并发 SQL 查询可能排队 |

**S3 前缀级别限流的踩坑**：CloudTrail 日志的 S3 路径按 `AccountId/Region/YYYY/MM/DD/` 分级存储。多账号 Organization Trail 的情况下，多个账号的日志可能被写入同一个 S3 Bucket 的相邻前缀，大规模并发 ListObjects + GetObject 可能触发 S3 的 `503 SlowDown` 错误（S3 每个前缀默认 5500 GET/s 限制）。建议实现指数退避重试和分批并发控制。

---

### 5.6 日志时序一致性（补充项 B）

CloudTrail 官方文档明确指出：**日志文件内的事件不保证按 API 调用时间排序**（"CloudTrail log files aren't an ordered stack trace of the public API calls, so events don't appear in any specific order"）。

**具体时序问题**：

| 时间字段 | 含义 | 注意事项 |
|---|---|---|
| `eventTime` | API 调用发生的时间（UTC） | 是研判基准，应始终使用此字段 |
| 日志文件名中的时间戳 | 日志文件被写入 S3 的时间 | 晚于 `eventTime` 约 5 分钟，不应作为事件时间 |
| S3 Object 的 `LastModified` | 文件写入 S3 的时间 | 与文件名时间戳含义相同 |
| SIEM 的 `ingestion_time` | 日志被 SIEM 摄入的时间 | 晚于 `eventTime` 可能 15-30 分钟 |

**对关联规则的影响**：

攻击链中的事件（如 AssumeRole → AttachRolePolicy → GetObject）可能分散在多个不同时间到达的日志文件中。若关联规则的时间窗口过短（如 5 分钟），且基于 SIEM ingestion 时间对齐，可能错过攻击链中的关键环节。**推荐实践**：使用 `eventTime` 作为关联基准，设置至少 15-30 分钟的事件关联窗口，并为数据采集延迟留出额外缓冲。

---

### 5.7 多账号/多 Region 日志汇聚（补充项 C）

AWS 的多账号架构（AWS Organizations）给日志汇聚带来独特挑战：

| 挑战 | 描述 | 推荐方案 |
|---|---|---|
| 每账号默认独立 CloudTrail | 100 个账号 = 100 套日志，默认不聚合 | 使用 Organization Trail，一次配置覆盖全部成员账号 |
| 横向移动跨账号可见性 | 攻击者从账号 A AssumeRole 到账号 B，账号 B 的 Trail 记录了 Session 操作，但无法直接看到账号 A 的原始凭证状态 | 通过 `recipientAccountId` + `userIdentity.accountId` 不一致识别跨账号操作；CloudTrail Lake 支持跨账号 SQL 查询 |
| 日志归集账号架构 | 集中存储 vs. 账号隔离存储 | AWS 官方推荐将日志写入独立的日志归档账号（Log Archive Account），并限制该账号的访问权限 |
| 新账号自动覆盖 | Organization Trail 自动覆盖新加入的成员账号 | 但成员账号开启数据事件需要各自在 Trail 上额外配置，或通过 CloudFormation StackSets 自动化部署 |

**攻防视角**：在供应链攻击场景中，攻击者往往从被攻陷的开发账号出发，通过 `OrganizationAccountAccessRole` 横向移动至生产账号。由于开发账号和生产账号的日志分别存储，若未进行跨账号关联分析，两条攻击链看起来都像是"正常的内部操作"，只有将两侧日志联立才能看到完整攻击路径。

> **参考文档**：  
> - [Creating a trail for an organization](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html)  
> - [AWS Security Reference Architecture – Log Archive Account](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/log-archive.html)

---

## 6. 安全厂商经验项总结

### 6.1 案例一：AI 辅助攻击在 8 分钟内拿到 AWS Admin（Sysdig，2026 年 2 月）

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

### 6.2 案例二：UNC6426 nx npm 供应链攻击 → AWS Admin（Google / CSA，2026 年 3 月）

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

### 6.3 经验项总结

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
