# AWS 云日志采集方案

> **文档状态**：从《AWS 云日志调研报告》中独立拆分 · 版本 2.0
> **调研时间**：2026-03
> **作者**：云安全专家调研
> **适用读者**：产品侧 / 后台研发团队 / 接入实施团队
> **核心聚焦**：CloudTrail 日志采集流程、授权模型、两套方案对比与工程最佳实践

---

## 目录

1. [业务背景和设计约束](#业务背景和设计约束)
2. [授权模型：后台如何访问客户账号](#1-授权模型后台如何访问客户账号)
3. [CloudTrail 能力分析](#2-cloudtrail-能力分析)
4. [采集方案总览与选型建议](#3-采集方案总览与选型建议)
5. [方案一：Trail → S3 → SQS（主流方案）](#4-方案一trail--s3--sqs主流方案)
6. [方案二：Trail → EventBridge → SQS（轻量方案）](#5-方案二trail--eventbridge--sqs轻量方案)
7. [方案对比与选型建议](#6-方案对比与选型建议)
8. [多账号场景：Organization Trail 接入](#7-多账号场景organization-trail-接入)

---

## 业务背景和设计约束

本文档从**客户（运维/开发人员）视角**出发，说明"我们采集的方案是什么，客户需要在自己的云账号里做什么，才能让我们的采集后台拿到日志"。

**采集后台的工作模式**：我们的后台程序部署在独立 SaaS 平台上，不会进入客户的云环境。当客户接入一个或多个云账号后，后台基于账号维度拉取日志。

**关键设计约束**：
- 优先设计**消息队列监听（Push-Pull）**的方案，客户在自己的账号将日志推送到某个队列（或类似的服务）中，通知部署在 SaaS 平台上订阅的后台程序进行日志采集。
- 设计方案前需要对云厂商提供的日志服务进行能力分析，例如是否支持流式拉取，是否提供 API 级别的筛选，是否支持重试等影响方案选型的关键能力。
- 至少制定两套采集方案进行对比分析，制定的方案最好在业界内有使用参考。
- 设计的方案需要说明客户侧关注内容：如何配置，成本等。
- 方案本身需考虑是否支持筛选、日志持久化、日志规模、延迟、限流、日志丢失这些基本问题，方案本身具备可扩展、高可用、可复用等特性。

---

## 1. 授权模型：后台如何访问客户账号

在客户接入任何采集方案之前，必须先完成对我们 SaaS 平台的**访问授权**。我们支持以下两种模型，客户可按实际情况选择：

### 模型 A：跨账号 AssumeRole（推荐）

这是 AWS 官方推荐的 SaaS 集成授权方式。客户在自己账号中创建一个 IAM Role，在 Trust Policy 中允许我们的 SaaS 平台账号（`{SAAS_ACCOUNT_ID}`）来扮演它，从而获得临时凭证。

**安全优势**：
- 客户无需持有、保管、轮换任何长期密钥
- 临时凭证默认有效期 1 小时，泄露后自动失效
- 客户可随时撤销 Role 的信任策略，立即切断我们的访问

**客户需要做的事（一次性）**：

客户在自己账号创建一个 IAM Role，Trust Policy 配置如下：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::{SAAS_ACCOUNT_ID}:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "{客户在平台注册后获得的唯一 ExternalId}"
        }
      }
    }
  ]
}
```

> **ExternalId 的作用**：防止"混淆代理人攻击"（Confused Deputy Attack）。如果没有 ExternalId，任何知道我们 SaaS 账号 ID 的第三方都可能诱骗我们扮演客户的 Role。加上 ExternalId 后，只有持有该客户专属 ID 的我们才能扮演，且每个客户的 ExternalId 应全局唯一不可猜测。参考：[How to use an external ID when granting access to your AWS resources](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user_externalid.html)

**后台调用流程**：

```
[SaaS 后台]
    ↓ sts:AssumeRole（携带 ExternalId）
[客户账号的 IAM Role]
    ↓ 返回临时凭证（AccessKeyId + SecretAccessKey + SessionToken，1 小时有效）
[SaaS 后台持有临时凭证]
    ↓ 调用 S3/SQS 等 API
[客户账号的 CloudTrail 日志]
```

### 模型 B：长期 AccessKey/SecretKey（AKSK）

客户在自己账号创建一个专用 IAM 用户，生成 AccessKey 并提供给我们。后台使用这对密钥直接调用 API。

**适用场景**：客户有安全合规限制无法使用跨账号 Role，或处于单账号简单架构。

**安全注意事项（必须向客户明确告知）**：
- AKSK 是长期有效的静态凭证，一旦泄露不会自动失效，需立即手动撤销
- 应为采集专用，**权限必须最小化**（仅授予读取日志所需权限，禁止任何写权限）
- 建议客户开启 AWS Access Analyzer 监控该用户的权限使用情况
- 建议定期（至少每 90 天）轮换 AKSK

---

## 2. CloudTrail 能力分析

本章对 CloudTrail 的核心能力进行分析，作为后续方案选型的决策依据。

### 2.1 日志交付机制

CloudTrail 支持两种日志交付路径，能力差异显著：

| 交付路径 | 交付延迟 | 是否持久化 | 是否支持流式消费 | 是否支持源侧过滤 |
|---|---|---|---|---|
| **写入 S3** | ~5 分钟（批量聚合后写入） | ✅ 持久化，生命周期由 S3 管理 | ❌ 不支持；需配合 SNS/SQS 通知触发 | ❌ 不支持；全量写入，过滤在消费侧 |
| **投递 EventBridge** | 通常 2–5 分钟（近实时） | ❌ 无持久化；SQS 消息最长保留 14 天 | ✅ 支持；事件到达 EventBridge 即可触发下游 | ✅ 支持；EventBridge 规则可多维度精确过滤 |

**核心结论**：两种路径的能力互补，不存在绝对优劣——S3 路径保证可靠性与完整性，EventBridge 路径保证实时性与过滤灵活性。两套方案选型正是基于此互补关系展开。

### 2.2 日志类型与覆盖范围

CloudTrail 记录三类事件，但与 EventBridge 的集成覆盖范围存在**关键限制**：

| 事件类型 | 说明 | 写入 S3 | 投递 EventBridge |
|---|---|---|---|
| **管理事件（Management Events）** | IAM、EC2、S3 Bucket 级操作等控制面 API 调用；首条 Trail 免费 | ✅ | ✅ |
| **数据事件（Data Events）** | S3 GetObject/PutObject、Lambda 调用等数据面操作；按量收费 | ✅ | ⚠️ **CloudTrail 不直接投递**；部分服务（如 S3）可通过自身原生 EventBridge 集成间接覆盖，但事件 schema 与 CloudTrail 格式不同，且不含操作者身份等审计关键字段（见下方说明） |
| **Insights 事件** | 异常 API 调用速率检测结果 | ✅ | ❌ 不支持 |

> **选型关键约束**：CloudTrail 本身**不直接**将数据事件投递到 EventBridge，但方案二并非完全无法覆盖此场景——部分产生数据事件的服务（如 S3）支持原生 EventBridge 集成，可在 EventBridge 中为该服务单独配置规则，间接实现数据事件的推送采集。然而，这条路径存在两个重要限制：①覆盖范围取决于具体服务是否支持原生 EventBridge 集成，并非所有数据事件来源服务均支持；②服务原生事件的 schema 与 CloudTrail 格式有本质差异，**不含操作者身份、来源 IP、错误码等审计关键字段**（详见下方 Schema 差异说明）。若客户的核心目标是安全审计（需要完整的操作者溯源信息），数据事件场景仍推荐方案一；若仅需资源变更感知，可评估服务原生 EventBridge 集成是否满足需求。
>
> **数据事件 Schema 差异说明（以 S3 PutObject 为例）**：
>
> 通过 **CloudTrail → S3**（方案一）得到的事件包含完整审计字段：
>
> ```json
> {
>   "eventVersion": "1.08",
>   "eventSource": "s3.amazonaws.com",
>   "eventName": "PutObject",
>   "eventTime": "2026-03-24T08:00:00Z",
>   "userIdentity": {
>     "type": "IAMUser",
>     "arn": "arn:aws:iam::123456789012:user/alice"
>   },
>   "requestParameters": {
>     "bucketName": "my-bucket",
>     "key": "logs/app.log"
>   },
>   "sourceIPAddress": "203.0.113.1",
>   "errorCode": null
> }
> ```
>
> 通过 **S3 原生 EventBridge 集成**（方案二间接路径）得到的事件**仅含存储通知语义**，不含操作者信息：
>
> ```json
> {
>   "version": "0",
>   "source": "aws.s3",
>   "detail-type": "Object Created",
>   "detail": {
>     "bucket": { "name": "my-bucket" },
>     "object": {
>       "key": "logs/app.log",
>       "size": 1024,
>       "etag": "d41d8cd98f00b204e9800998ecf8427e"
>     },
>     "reason": "PutObject"
>   }
> }
> ```
>
> 两种格式的关键字段差异对比：
>
> | 字段 | CloudTrail 格式（方案一） | S3 原生 EventBridge（方案二间接路径） |
> |---|---|---|
> | 操作者身份 | ✅ `userIdentity`（完整 IAM 信息） | ❌ 无 |
> | 来源 IP | ✅ `sourceIPAddress` | ❌ 无 |
> | 错误码 | ✅ `errorCode` | ❌ 无 |
> | 对象大小/ETag | ❌ 无 | ✅ 有 |
> | 字段路径风格 | 扁平，直接取 `eventName` | 嵌套在 `detail` 下 |
>
> S3 原生 EventBridge 事件是"**存储通知**"语义（告知"有对象被创建"），而非"**审计日志**"语义（记录"谁、从哪里、用什么身份做了什么"）。安全审计场景需要后者，资源变更通知场景前者已足够。

### 2.3 API 查询能力（LookupEvents）

CloudTrail 提供 `LookupEvents` API，支持按 `eventName`、`username`、`resourceType` 等字段查询，但有严格限制：

- **仅保留 90 天**事件记录
- **吞吐量极低**：每账号每 Region 仅 2 次请求/秒，每次最多返回 50 条
- **不支持复杂过滤**：字段组合查询能力弱，无法按 `sourceIPAddress`、`errorCode` 等字段过滤

**结论**：`LookupEvents` 适合交互式小批量查询（如控制台搜索），**不适合作为持续采集通道**。高吞吐、低延迟的采集场景必须通过 S3 或 EventBridge 路径实现。

### 2.4 日志规模估算

日志规模直接影响后台 Worker 并发设计和 SQS 配额规划，以下为参考基准：

**方案一（S3 文件维度）**：
- CloudTrail 每隔约 5 分钟将事件批量聚合写入一个 `.json.gz` 文件
- 单个 S3 文件通常包含数十到数百条事件，压缩前约 10 KB–1 MB，压缩后约 1–100 KB
- 中型企业（~1000 人）管理事件量：约 **1–5 GB/天**，每天约产生 **数百个 S3 文件**（对应数百条 SQS 消息）

**方案二（单事件维度）**：
- EventBridge 每触发一次规则推送一条 SQS 消息，消息体即完整事件 JSON（约 1–5 KB/条）
- 若采集全量管理事件，中型企业峰值可达每分钟数百条消息，后台需具备对应并发消费能力
- 若启用 EventBridge 规则精确过滤（如仅 IAM 高危操作），消息量可压缩至全量的 1%–10%

### 2.5 重试与可靠性能力

| 能力 | CloudTrail → S3 | CloudTrail → EventBridge |
|---|---|---|
| **投递失败重试** | CloudTrail 内置重试，S3 写入最终一致 | EventBridge 内置重试（至少 24 小时，指数退避） |
| **消费侧重试** | SQS Visibility Timeout + DLQ | SQS Visibility Timeout + DLQ |
| **消费失败补救** | ✅ S3 原始文件永久保留，可随时补拉 | ❌ SQS 消息超期后永久丢失，无法补救 |
| **幂等消费** | 需后台基于 `eventID` 去重 | 需后台基于 `eventID` 去重 |

---

## 3. 采集方案总览与选型建议

基于准实时（5–15 分钟）的产品定位，我们提供两套采集方案，客户根据自身环境和接受的配置复杂度选择：

| 方案 | 核心链路 | 端到端延迟 | 客户配置复杂度 | 客户侧增量成本 | 推荐场景 |
|---|---|---|---|---|---|
| **方案一**：Trail → S3 → SQS（主流方案） | CloudTrail → S3 → SNS → SQS → 后台拉取 | **5–15 分钟** | ⭐⭐（中，约 5 步） | S3 存储费 + SNS/SQS 极低费用 | 绝大多数客户；兼顾完整性与可靠性 |
| **方案二**：Trail → EventBridge → SQS（轻量方案） | CloudTrail → EventBridge → SQS → 后台拉取 | **3–10 分钟** | ⭐（低，约 3 步） | EventBridge 极低费用；**无需 S3** | 客户需要高实时性告警，或只需采集特定高危管理事件 |

> 两套方案均兼容授权模型 A（跨账号 AssumeRole）和模型 B（AKSK），授权配置见 §1。

**业界使用情况参考**：

两套方案的链路均为业界成熟做法，非本文定制方案，以下为主流安全厂商的公开文档佐证：

| 方案 | 厂商使用案例 | 文档链接 |
|---|---|---|
| **方案一**（Trail→S3→SNS→SQS） | **Splunk**：官方 Add-on for AWS 的 CloudTrail 标准采集模式即 SQS-based S3；文档明确将 SQS-based S3 标注为比直接 CloudTrail input 更高性能、更具容错性的替代方案 | [Configure SQS-Based S3 inputs for the Splunk Add-on for AWS](https://docs.splunk.com/Documentation/AddOns/released/AWS/SQS-basedS3) |
| **方案一**（Trail→S3→SNS→SQS） | **Elastic**：Filebeat `aws-s3` input 的文档原文：polling lists of S3 objects is expensive in terms of performance and costs，并将 SQS notification 标注为首选方式 | [AWS S3 input – Elastic Beats](https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-aws-s3) |
| **方案一**（Trail→S3→SNS→SQS） | **Google SecOps（Chronicle）**：官方 CloudTrail 接入文档采用 Trail → S3 → SNS → SQS 链路，并要求 SNS 开启 Raw message delivery | [Collect AWS CloudTrail logs – Google SecOps](https://docs.cloud.google.com/chronicle/docs/ingestion/default-parsers/aws-cloudtrail) |
| **方案二**（EventBridge→SQS） | **CrowdStrike**（AWS Architecture Blog）：CrowdStrike Falcon Horizon IOA 架构基于客户账号内的 EventBridge 规则监听 CloudTrail，将事件流推送到 CrowdStrike 侧集中处理，AWS 博客将其描述为"毫秒级威胁检测"的核心架构 | [Detect Adversary Behavior in Seconds with CrowdStrike and Amazon EventBridge](https://aws.amazon.com/blogs/architecture/detect-adversary-behavior-in-seconds-with-crowdstrike-and-amazon-eventbridge/) |
| **方案二**（EventBridge→SQS） | **AWS 官方**（Compute Blog）：AWS 博客于 2025 年发布文章，明确将 CloudTrail + EventBridge 的推送模式定位为"克服传统轮询方案延迟限制"的推荐架构，并引用 CrowdStrike 在数千 AWS 账号中的实践 | [Enhancing multi-account activity monitoring with event-driven architectures](https://aws.amazon.com/blogs/compute/enhancing-multi-account-activity-monitoring-with-event-driven-architectures/) |
| **方案二**（EventBridge→SQS） | **AWS Security Blog**：AWS 官方安全博客使用 EventBridge → Lambda 架构作为 IAM 高危操作的实时告警示例（Lambda 可替换为 SQS 以适配外部 SaaS 后台） | [How to receive alerts when your IAM configuration changes](https://aws.amazon.com/blogs/security/how-to-receive-alerts-when-your-iam-configuration-changes/) |

> **说明**：方案二在市面上更多以"EventBridge → Lambda（直接处理）"形式出现（Lambda 在客户账号内运行）。本文将目标替换为 SQS，是为适配 SaaS 后台"外部程序主动拉取"的部署约束——EventBridge → SQS 同样是 AWS 官方推荐的标准集成模式，区别仅在于计算消费侧的位置。

---

## 4. 方案一：Trail → S3 → SNS → SQS（主流方案）

这是覆盖面最广的方案，所有类型日志（管理事件 + 数据事件）均可采集，并通过 SQS 通知实现准实时感知。

### 4.1 整体链路

```
[AWS API 调用发生]
      ↓ ~5 分钟（CloudTrail 批量聚合后写入）
[S3 Bucket：日志文件 .json.gz]
      ↓ 自动触发
[SNS Topic：新文件通知]
      ↓ 订阅推送
[SQS Queue：消息队列]
      ↓ 后台长轮询（Long Polling）
[SaaS 采集后台：消费消息 → S3 GetObject 下载 → 解析入库]
```

端到端延迟：CloudTrail 写入 S3（~5 分钟）+ SNS→SQS 推送（秒级）+ 后台消费（秒级）= **约 6–15 分钟**。

### 4.2 客户需要配置的内容

以下为客户在自己账号中需要完成的配置，每步均有**控制台手动操作**和 **CloudFormation 模板**两种方式。

---

**第 1 步：创建 CloudTrail Multi-Region Trail**

这是所有方案的基础前提。若客户已有 Trail，可跳过或复用。

> **为什么必须是 Multi-Region Trail？** 单 Region Trail 只记录该 Region 的事件。IAM、STS 等全局服务的日志只会写入 `us-east-1` 的 Trail。若客户只在 `ap-southeast-1` 建了单 Region Trail，将**完全看不到任何 IAM 操作**（包括谁创建了用户、谁修改了策略），这是安全监控的重大盲区。

**控制台操作（约 5 分钟）**：

1. 打开 AWS CloudTrail 控制台 → 点击「创建追踪」
2. 追踪名称：填写（如 `security-audit-trail`）
3. 存储位置：新建或选择已有 S3 Bucket（建议新建专用 Bucket，命名如 `{账号ID}-cloudtrail-logs`）
4. ✅ 勾选「为所有账户启用追踪」（即 Multi-Region）
5. ✅ 勾选「启用日志文件验证」（开启 Digest 完整性验证）
6. 日志事件：
   - ✅ 管理事件：读/写均记录
   - ⬜ 数据事件：默认不开启（按需开启，有额外费用，见 §4.4）
7. 点击「创建追踪」

> ⚠️ 重要：创建 Trail 后请确认控制台左侧状态显示「正在记录」（Logging: Enabled）。**Trail 创建后不会自动开启记录**，若状态为"停止"需手动点击开启。

**CloudFormation 模板**：

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudTrail Multi-Region Trail for Security Audit'

Resources:
  CloudTrailBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AWS::AccountId}-cloudtrail-logs'
      VersioningConfiguration:
        Status: Enabled

  CloudTrailBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref CloudTrailBucket
      PolicyDocument:
        Statement:
          - Sid: AWSCloudTrailAclCheck
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:GetBucketAcl
            Resource: !GetAtt CloudTrailBucket.Arn
          - Sid: AWSCloudTrailWrite
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:PutObject
            Resource: !Sub '${CloudTrailBucket.Arn}/AWSLogs/${AWS::AccountId}/*'
            Condition:
              StringEquals:
                s3:x-amz-acl: bucket-owner-full-control

  SecurityAuditTrail:
    Type: AWS::CloudTrail::Trail
    DependsOn: CloudTrailBucketPolicy
    Properties:
      TrailName: security-audit-trail
      S3BucketName: !Ref CloudTrailBucket
      IsLogging: true
      IsMultiRegionTrail: true
      IncludeGlobalServiceEvents: true
      EnableLogFileValidation: true
      EventSelectors:
        - ReadWriteType: All
          IncludeManagementEvents: true

Outputs:
  TrailArn:
    Value: !GetAtt SecurityAuditTrail.Arn
  BucketName:
    Value: !Ref CloudTrailBucket
```

---

**第 2 步：创建 SNS Topic 并配置 Trail 通知**

当 CloudTrail 每次将日志文件写入 S3 时，SNS 自动发送通知。

**控制台操作（约 3 分钟）**：

1. 打开 SNS 控制台 → 创建主题 → 类型选「标准」
2. 主题名称：填写（如 `cloudtrail-log-notifications`）
3. 返回 CloudTrail 控制台 → 编辑 Trail → 「SNS 通知」→ 选择刚创建的 Topic
4. 保存

---

**第 3 步：创建 SQS Queue 并订阅 SNS**

SaaS 后台通过长轮询此队列获知有新日志文件，再主动拉取。

**控制台操作（约 3 分钟）**：

1. 打开 SQS 控制台 → 创建队列 → 类型选「标准队列」
2. 队列名称：填写（如 `cloudtrail-log-queue`）
3. 消息保留期：建议设为 **4 天**（给后台足够的消费窗口，防止后台临时故障导致消息丢失）
4. 进入队列 → 「SNS 订阅」→ 订阅第 2 步创建的 SNS Topic
5. 修改 SQS 队列访问策略，允许 SNS 向其发送消息（控制台订阅时会自动提示）

**CloudFormation 模板（步骤 2+3 合并）**：

```yaml
  CloudTrailSNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: cloudtrail-log-notifications

  CloudTrailSNSTopicPolicy:
    Type: AWS::SNS::TopicPolicy
    Properties:
      Topics:
        - !Ref CloudTrailSNSTopic
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: SNS:Publish
            Resource: !Ref CloudTrailSNSTopic

  CloudTrailSQSDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: cloudtrail-log-queue-dlq
      MessageRetentionPeriod: 1209600  # 14 天，留足人工或自动重处理时间

  CloudTrailSQSQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: cloudtrail-log-queue
      MessageRetentionPeriod: 345600  # 4 天（秒）
      VisibilityTimeout: 300           # 5 分钟，给后台足够处理时间
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt CloudTrailSQSDLQ.Arn
        maxReceiveCount: 3             # 消费失败 3 次后转入 DLQ，不再阻塞主队列

  SQSQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref CloudTrailSQSQueue
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt CloudTrailSQSQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref CloudTrailSNSTopic

  SNSSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref CloudTrailSNSTopic
      Protocol: sqs
      Endpoint: !GetAtt CloudTrailSQSQueue.Arn
```

---

**第 4 步：授权 SaaS 后台读取 S3 和 SQS**

根据客户选择的授权模型（§1），将以下权限附加给对应的 IAM Role（模型 A）或 IAM User（模型 B）：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadCloudTrailLogs",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::{cloudtrail-bucket-name}",
        "arn:aws:s3:::{cloudtrail-bucket-name}/*"
      ]
    },
    {
      "Sid": "ConsumeNotificationQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:{region}:{account-id}:cloudtrail-log-queue"
    },
    {
      "Sid": "DecryptIfKMSEnabled",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:GenerateDataKey"
      ],
      "Resource": "arn:aws:kms:{region}:{account-id}:key/{key-id}"
    }
  ]
}
```

> 若客户日志 S3 Bucket 未启用 KMS 加密，可删除 `DecryptIfKMSEnabled` 条目。若客户使用 AWS Organizations 的 Organization Trail，S3 路径中会包含 `o-{org-id}` 前缀，Resource 路径需相应调整（详见 §7）。

**第 4 步 CloudFormation（模型 A，跨账号 AssumeRole 方式）**：

```yaml
  SaaSIntegrationRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: SaaS-CloudTrail-Reader
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::{SAAS_ACCOUNT_ID}:root'
            Action: sts:AssumeRole
            Condition:
              StringEquals:
                sts:ExternalId: '{客户专属 ExternalId}'
      Policies:
        - PolicyName: CloudTrailReadPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:ListBucket
                Resource:
                  - !GetAtt CloudTrailBucket.Arn
                  - !Sub '${CloudTrailBucket.Arn}/*'
              - Effect: Allow
                Action:
                  - sqs:ReceiveMessage
                  - sqs:DeleteMessage
                  - sqs:GetQueueAttributes
                Resource: !GetAtt CloudTrailSQSQueue.Arn

Outputs:
  RoleArn:
    Description: '填入 SaaS 平台的账号接入配置中'
    Value: !GetAtt SaaSIntegrationRole.Arn
  SQSQueueUrl:
    Description: '填入 SaaS 平台的账号接入配置中'
    Value: !Ref CloudTrailSQSQueue
```

---

**第 5 步：在 SaaS 平台填写接入信息**

配置完成后，客户在我们的 SaaS 平台"账号接入"页面填写：

| 字段 | 内容（模型 A：AssumeRole） | 内容（模型 B：AKSK） |
|---|---|---|
| 账号 ID | 客户 AWS 账号 12 位 ID | 客户 AWS 账号 12 位 ID |
| Role ARN | CloudFormation Output 中的 `RoleArn` | —— |
| AccessKey ID | —— | 客户创建的 IAM User AK |
| SecretAccessKey | —— | 客户创建的 IAM User SK |
| SQS Queue URL | CloudFormation Output 中的 `SQSQueueUrl` | 同左 |
| S3 Bucket 名称 | CloudFormation Output 中的 `BucketName` | 同左 |
| 日志所在 Region | 客户主要业务 Region（Multi-Region Trail 会自动覆盖所有 Region），用来读取 S3 和 SQS | 同左 |

### 4.3 返回格式

日志以 **gzip 压缩的 JSON** 格式存储在 S3，每个文件包含一个 `Records` 数组：

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
    }
  ]
}
```

**文件路径规范（S3 内目录结构）**：

```
s3://{bucket}/AWSLogs/{AccountId}/CloudTrail/{Region}/{YYYY}/{MM}/{DD}/
  └── {AccountId}_CloudTrail_{Region}_{YYYYMMDD}T{HHMM}Z_{UniqueString}.json.gz
```

> 若使用 Organization Trail，路径前会多一级：`AWSLogs/{OrgId}/{AccountId}/CloudTrail/...`（详见 §7）

### 4.4 成本说明

客户接入方案一后，新增的 AWS 服务费用包括：

| 费用项 | 计费方式 | 估算（中型企业，1000 人，管理事件） |
|---|---|---|
| CloudTrail（首条 Trail 管理事件） | **免费** | $0 |
| S3 存储（日志文件） | ~$0.023/GB/月 | 管理事件约 1–5 GB/天，月度 $1–5 |
| S3 请求（PUT/GET） | $0.005/1000 次 | 可忽略不计 |
| SNS 通知 | $0.50/百万条 | 可忽略不计 |
| SQS 消息 | 每月前 100 万条免费，之后 $0.40/百万条 | 可忽略不计 |
| **数据事件（按需开启）** | $0.10/100,000 次事件 | 高频 S3 访问可达每月数百至数千美元，**开启前需评估** |

> 数据事件成本与客户业务量强相关，建议仅对安全价值高的核心资源（如含 PII 的 S3 Bucket、生产 KMS Key）按需开启，不建议全账号开启。

### 4.5 踩坑与注意事项

**① Trail 创建后状态需手动确认**

Trail 创建后若未点击「开启」，IsLogging 状态为 false，不产生任何日志。客户应通过 `aws cloudtrail get-trail-status --name <trail-name>` 确认 `"IsLogging": true`。后台在接入完成后应定期通过此接口做健康检查。

**② IAM 等全局服务事件只写入 `us-east-1`**

Multi-Region Trail 配置了 `IncludeGlobalServiceEvents: true` 后，IAM/STS 等全局服务事件会**统一写入 `us-east-1` 的日志路径**，而不会分散到各 Region 子目录。后台采集时需注意枚举 S3 所有 Region 子目录，不能只取业务主 Region 的路径。

**③ 日志文件时间戳 ≠ 事件发生时间**

S3 文件名中的时间戳（`20240315T2210Z`）是**文件被写入 S3 的时间**，与文件内 `eventTime` 存在约 5 分钟偏差。解析时必须以每条记录的 `eventTime` 作为事件时间，不能用文件名时间戳替代。

**④ SQS 消息删除必须在处理成功后执行**

后台消费 SQS 消息时，应先消费（`ReceiveMessage`）、处理日志、处理成功后再删除消息（`DeleteMessage`）。若先删后处理，一旦处理中途失败将导致该条消息永久丢失（对应日志文件漏拉）。

每条消息在被 `ReceiveMessage` 取走后会进入 **Visibility Timeout** 隐藏窗口（模板中设为 300 秒），在此期间其他消费者看不到它。若后台在窗口内未完成 `DeleteMessage`，消息会重新变为可见并被再次投递——这是 SQS 的容错机制，但后台必须用 `eventID` 做幂等去重，否则同一条日志可能会被重复处理，产生重复告警。

**⑤ SQS 消息积压与日志丢失风险**

SQS 本身不会因为积压而丢消息——队列堆积再多消息也会持久保存，**不会因为"太多"被删除**。但有一个真实丢失风险需要明确：**消息保留期到期后自动永久删除**（模板中设为 4 天，上限 14 天），超期未消费的消息会被 SQS 静默删除且无任何通知。

在方案一中，S3 中的日志文件始终存在，SQS 消息超期只意味着"新文件到达通知丢失"，后台可通过**定期全量扫描 S3 做补偿拉取**（如每小时与已处理记录对比，补拉缺失文件）来实现最终一致性，**丢通知不等于丢日志**。

积压的根本原因是**消费速率 < 生产速率**，或后台故障导致消费中断。后台应对策略：

- **并发扩容**：SQS 标准队列支持多 Worker 并发消费，吞吐量瓶颈在后台 Worker 数而非 SQS。建议基于 `ApproximateNumberOfMessages`（队列深度）指标自动扩缩容 Worker 数量。
- **死信队列（DLQ）兜底**：上方 CloudFormation 模板已配置 DLQ（`maxReceiveCount: 3`），消费失败 3 次的消息自动转入 DLQ，不再阻塞主队列消费，同时在 DLQ 中保留 14 天供事后重处理或告警。
- **S3 补偿扫描**：后台以低频（如每小时一次）对 S3 做增量路径扫描，与已处理文件列表对比，补拉因 SQS 消息超期或消费失败而漏掉的文件，确保最终不漏日志。

> **注意**：SQS 消息积压对客户业务无任何影响。积压完全发生在 SaaS 采集侧，CloudTrail 持续正常记录日志，SNS/SQS 也不会因积压反向影响客户的 AWS 业务。积压唯一的后果是我们这侧的告警延迟增大。

> **参考文档**：
> - [Quotas in AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/WhatIsCloudTrail-Limits.html)
> - [Getting and viewing CloudTrail log files](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/get-and-view-cloudtrail-log-files.html)
>
> **厂商实现参考（方案一）**：
> - Splunk Add-on for AWS CloudTrail input（SQS 模式）：[Configure CloudTrail inputs – Splunk Docs](https://docs.splunk.com/Documentation/AddOns/released/AWS/CloudTrail)
> - Splunk SQS-Based S3 input 详细配置（推荐替代 CloudTrail input）：[Configure SQS-Based S3 inputs – Splunk Docs](https://docs.splunk.com/Documentation/AddOns/released/AWS/SQS-basedS3)
> - Elastic Filebeat aws-s3 input（SQS 通知模式）：[AWS S3 input – Elastic Beats](https://www.elastic.co/docs/reference/beats/filebeat/filebeat-input-aws-s3)
> - Google SecOps CloudTrail 接入（Trail→S3→SNS→SQS 链路）：[Collect AWS CloudTrail logs – Google SecOps](https://docs.cloud.google.com/chronicle/docs/ingestion/default-parsers/aws-cloudtrail)

---

## 5. 方案二：Trail → EventBridge → SQS（轻量方案）

此方案跳过 S3 存档环节，通过 EventBridge 将 CloudTrail 事件**近实时推送**到 SQS，后台直接消费完整事件 JSON。适合客户需要高实时性告警、或只需对特定高危管理事件做精准采集的场景。

**与方案一的核心区别**：

| 对比项 | 方案一（Trail → S3） | 方案二（EventBridge） |
|---|---|---|
| 数据完整性 | 完整（全量日志写入 S3，可长期保留） | 仅推送触发的事件，**无 S3 历史留存** |
| 过滤能力 | 客户侧无法预过滤，全量拉取后后台过滤 | **可在 EventBridge 规则中精确过滤**（如仅推送 IAM 高危操作） |
| 客户配置复杂度 | 较高（Trail + S3 + SNS + SQS + 授权） | **低**（Trail + EventBridge 规则 + SQS + 授权） |
| 适用事件类型 | 管理事件 + 数据事件 | 管理事件直接覆盖；数据事件可通过服务原生 EventBridge 集成间接覆盖，但 schema 不含审计关键字段（见 §2.2） |
| 日志保留 | 由 S3 生命周期决定 | **SQS 消息最长保留 14 天，消费后即删除** |
| **SQS 积压导致丢失** | **可补救**：消息超期只丢通知，S3 有原始文件可补偿扫描 | **不可补救**：消息超期 = 事件永久丢失，无任何补救途径 |

> **事件覆盖说明**：CloudTrail 与 EventBridge 的原生集成目前**仅直接覆盖管理事件**。数据事件（如 S3 GetObject/PutObject）可通过在 EventBridge 中为相关服务（如 S3）单独配置原生集成规则间接覆盖，但服务原生事件的 schema 与 CloudTrail 格式有本质差异，**不含操作者身份、来源 IP、错误码等审计关键字段**（详见 §2.2 的 Schema 差异说明）。若客户的核心目标是安全审计，数据事件场景仍推荐方案一。

### 5.1 整体链路

```
[AWS 管理事件发生（如 CreateUser、AssumeRole）]
      ↓ 通常 2–5 分钟（EventBridge 近实时触发）
[EventBridge 规则：匹配目标事件]
      ↓
[SQS Queue：完整事件 JSON]
      ↓ 后台长轮询（Long Polling）
[SaaS 采集后台：直接解析 JSON 入库]
```

端到端延迟：CloudTrail 事件到达 EventBridge（通常 2–5 分钟，AWS 官方无严格 SLA 保证）+ EventBridge→SQS 推送（秒级）+ 后台消费（秒级）= **约 3–10 分钟**。

> **关于延迟的说明**：方案二的延迟在理想情况下略优于方案一，但 CloudTrail 到 EventBridge 的投递延迟并无严格 SLA，在极端情况下与方案一差异不大。实测延迟数据以具体环境为准，不应以此作为选型的决定性因素。

### 5.2 客户需要配置的内容

**前提**：客户账号中已有任意一条 Trail 处于运行状态（EventBridge 依赖 CloudTrail 产生事件，Trail 必须存在且 `IsLogging=true`，但不需要 Trail 配置写入 S3）。

---

**第 1 步：创建 SQS Queue**

操作与方案一第 3 步相同，创建一个标准 SQS 队列（如 `cloudtrail-eventbridge-queue`）。

---

**第 2 步：创建 EventBridge 规则**

**控制台操作（约 5 分钟）**：

1. 打开 EventBridge 控制台 → 「规则」→「创建规则」
2. 规则名称：填写（如 `cloudtrail-iam-highrisk-rule`）
3. 事件总线：选「default」（CloudTrail 管理事件自动发布到账号默认总线）
4. 规则类型：选「事件模式」
5. 事件模式：选「自定义模式」，填入以下 JSON（可按需调整，以下示例过滤 IAM 高危操作）：

```json
{
  "source": ["aws.iam", "aws.sts", "aws.cloudtrail"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventSource": ["iam.amazonaws.com", "sts.amazonaws.com"],
    "eventName": [
      "CreateUser", "DeleteUser",
      "AttachUserPolicy", "AttachRolePolicy", "PutRolePolicy",
      "CreateAccessKey", "DeleteAccessKey",
      "AssumeRole", "AssumeRoleWithWebIdentity",
      "StopLogging", "DeleteTrail"
    ]
  }
}
```

> **事件模式说明**：这里使用的是 **CloudTrail 集成的 EventBridge 事件**，其 `detail-type` 固定为 `"AWS API Call via CloudTrail"`，`detail` 层级内的字段（`eventSource`、`eventName` 等）是 CloudTrail 事件字段，与 EventBridge 原生服务事件（`source` 为 `aws.iam` 等）是两套完全不同的体系，不可混用。
>
> 若需采集全量管理事件（不做过滤），将 `detail` 部分整体删除，仅保留 `source` 和 `detail-type` 匹配即可。EventBridge 规则还支持按 `userIdentity.type`、`sourceIPAddress`、`errorCode` 等字段组合过滤，可在产生高噪音的环境中显著降低后台处理量，是方案一无法在数据源侧实现的能力。

6. 目标：选「SQS 队列」→ 选择刚创建的队列
7. 保存规则

**CloudFormation 模板（步骤 1+2 合并）**：

```yaml
  EventBridgeSQSDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: cloudtrail-eventbridge-queue-dlq
      MessageRetentionPeriod: 1209600  # 14 天；方案二无 S3 兜底，DLQ 是唯一补救手段

  EventBridgeSQSQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: cloudtrail-eventbridge-queue
      MessageRetentionPeriod: 345600   # 4 天
      VisibilityTimeout: 300
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt EventBridgeSQSDLQ.Arn
        maxReceiveCount: 3             # 消费失败 3 次后转入 DLQ；方案二中 DLQ 消息是唯一恢复途径，必须配置监控告警

  EventBridgeSQSPolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref EventBridgeSQSQueue
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: events.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt EventBridgeSQSQueue.Arn

  CloudTrailHighRiskRule:
    Type: AWS::Events::Rule
    Properties:
      Name: cloudtrail-iam-highrisk-rule
      EventBusName: default
      EventPattern:
        source:
          - aws.iam
          - aws.sts
          - aws.cloudtrail
        detail-type:
          - "AWS API Call via CloudTrail"
        detail:
          eventSource:
            - iam.amazonaws.com
            - sts.amazonaws.com
          eventName:
            - CreateUser
            - DeleteUser
            - AttachUserPolicy
            - AttachRolePolicy
            - CreateAccessKey
            - AssumeRole
            - AssumeRoleWithWebIdentity
            - StopLogging
            - DeleteTrail
      Targets:
        - Id: SendToSQS
          Arn: !GetAtt EventBridgeSQSQueue.Arn
      State: ENABLED
```

---

**第 3 步：授权 SaaS 后台读取 SQS**

权限比方案一更小，无需 S3 读取权限（事件内容直接在 SQS 消息体中）：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ConsumeEventBridgeQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:{region}:{account-id}:cloudtrail-eventbridge-queue"
    }
  ]
}
```

### 5.3 SQS 消息体格式（与方案一的差异）

EventBridge 推送到 SQS 的消息体**不是原始 CloudTrail 格式**，而是 EventBridge 事件信封包裹的 JSON：

```json
{
  "version": "0",
  "id": "12345678-abcd-...",
  "source": "aws.iam",
  "account": "123456789012",
  "time": "2024-03-15T09:15:00Z",
  "region": "us-east-1",
  "detail-type": "AWS API Call via CloudTrail",
  "detail": {
    "eventVersion": "1.09",
    "userIdentity": { ... },
    "eventTime": "2024-03-15T09:15:00Z",
    "eventSource": "sts.amazonaws.com",
    "eventName": "AssumeRole",
    "awsRegion": "us-east-1",
    "sourceIPAddress": "198.51.100.23",
    "requestParameters": { ... },
    "responseElements": { ... },
    "eventID": "c1d2e3f4-...",
    "eventType": "AwsApiCall",
    "managementEvent": true,
    "recipientAccountId": "123456789012"
  }
}
```

### 5.4 成本说明

| 费用项 | 计费方式 | 估算 |
|---|---|---|
| EventBridge 自定义事件 | $1.00/百万次事件 | 中型企业管理事件约百万条/月，约 $1/月 |
| SQS 消息 | 每月前 100 万条免费 | 可忽略不计 |
| CloudTrail 管理事件 | 首条 Trail 免费 | $0 |
| **S3 存储** | —— | **方案二无 S3 存储费**（无长期留存） |

> 相比方案一，方案二**无 S3 存储成本**，但**失去了历史日志留存能力**。若客户有合规要求需保留 90 天以上日志，必须选择方案一，或同时保留一条写入 S3 的 Trail。

---

## 6. 方案对比与选型建议

| 维度 | 方案一（Trail→S3→SQS） | 方案二（EventBridge→SQS） |
|---|---|---|
| **延迟** | 5–15 分钟 | 3–10 分钟（AWS 无严格 SLA） |
| **客户配置步骤** | 5 步，中等复杂 | 3 步，简单 |
| **事件覆盖** | 管理事件 + 数据事件 | 管理事件直接覆盖；数据事件可间接覆盖但 schema 缺少审计关键字段 |
| **历史日志留存** | ✅ 有（S3）| ❌ 无 |
| **数据源侧过滤** | ❌ 不支持 | ✅ 支持（EventBridge 规则） |
| **SQS 消息超期后** | **可补救**：消息超期只丢通知，S3 文件仍在，后台可补偿扫描 | **不可补救**：消息超期 = 事件永久丢失，无任何补救途径 |
| **DLQ 必要性** | 建议配置 | **强烈建议，是唯一补救手段** |
| **客户侧月度增量成本** | S3 存储费（低） | EventBridge 费（极低） |
| **推荐客户类型** | 需要全量日志或数据事件的主流客户 | 只需高危管理事件、配置越简单越好 |

**选型决策树**：

```
客户是否需要数据事件（S3 GetObject 等）？
    ├─ 是，且需要完整审计字段（操作者身份、来源 IP 等）→ 方案一（CloudTrail → S3，全量日志，有 S3 兜底）
    ├─ 是，但仅需资源变更感知（不要求操作者溯源）→ 可评估方案二 + 服务原生 EventBridge 集成（见 §2.2 Schema 差异说明）
    └─ 否 → 客户是否接受"SQS 消息超期事件永久丢失"的风险？
                ├─ 接受（有强 DLQ 监控保障）→ 方案二（仅高危管理事件，配置更简单）
                └─ 不接受 → 方案一
```

**后台工程建议（针对方案一和方案二）**：

SQS 积压不会影响客户业务，但会增大告警延迟并在极端情况下导致方案二事件丢失。后台应从三层防御：

- **第一层·预防积压**：基于 CloudWatch 指标 `ApproximateNumberOfMessages` 实现 Worker 自动扩缩容，队列深度超阈值时水平扩展消费并发，消费速率始终大于生产速率。
- **第二层·失败兜底**：主队列配置 DLQ（`maxReceiveCount: 3`），消费失败 3 次自动转入 DLQ，DLQ 保留 14 天。后台对 DLQ 设置深度告警，触发时自动重处理或人工介入，避免消息在主队列反复重试阻塞后续消息。
- **第三层·最终一致（仅方案一）**：后台每小时对 S3 执行一次增量路径扫描，与已处理文件列表对比，补拉因 SQS 消息超期或消费异常而漏掉的文件，实现日志采集的最终一致性。方案二无此能力，是方案一在可靠性上优于方案二的根本工程理由。

> **参考文档**：
> - EventBridge + CloudTrail 集成：[Creating an EventBridge rule that triggers on an AWS API call using AWS CloudTrail](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-ct-api-tutorial.html)
> - EventBridge 事件模式语法：[Amazon EventBridge event patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)
> - SQS 长轮询：[Amazon SQS long polling](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-long-polling.html)
>
> **厂商实现参考（方案二）**：
> - CrowdStrike Falcon Horizon IOA + EventBridge 架构（AWS Architecture Blog）：[Detect Adversary Behavior in Seconds with CrowdStrike and Amazon EventBridge](https://aws.amazon.com/blogs/architecture/detect-adversary-behavior-in-seconds-with-crowdstrike-and-amazon-eventbridge/)
> - AWS Compute Blog — 多账号 EventBridge 实时监控架构（含 CrowdStrike 实践引用，2025）：[Enhancing multi-account activity monitoring with event-driven architectures](https://aws.amazon.com/blogs/compute/enhancing-multi-account-activity-monitoring-with-event-driven-architectures/)
> - AWS Security Blog — EventBridge 监控 IAM 变更实时告警（官方示例）：[How to receive alerts when your IAM configuration changes](https://aws.amazon.com/blogs/security/how-to-receive-alerts-when-your-iam-configuration-changes/)

---

## 7. 多账号场景：Organization Trail 接入

大中型企业通常通过 AWS Organizations 统一管理多个账号，并启用 **Organization Trail** 将所有成员账号的 CloudTrail 事件集中写入管理账号（或委托管理员账号）的 S3 Bucket。这是企业环境下的标准配置，而非边缘场景，本章单独说明其接入差异。

### 7.1 Organization Trail 与单账号 Trail 的关键差异

| 对比项 | 单账号 Trail | Organization Trail |
|---|---|---|
| **Trail 创建位置** | 各成员账号各自创建 | 在管理账号（或委托管理员账号）统一创建 |
| **覆盖范围** | 仅本账号事件 | **自动覆盖 Organization 下所有成员账号** |
| **S3 路径格式** | `AWSLogs/{AccountId}/CloudTrail/...` | `AWSLogs/{OrgId}/{AccountId}/CloudTrail/...`（多一级 Org ID） |
| **S3 Bucket 归属** | 成员账号自己的 Bucket | 通常在管理账号或日志归档账号的 Bucket |
| **SNS/SQS 配置** | 在成员账号内配置 | 在 Trail 所在账号（管理账号）配置 |
| **成员账号数据隔离** | 天然隔离 | 所有成员账号日志混合在同一 Bucket，按路径区分 |

### 7.2 接入架构

Organization Trail 场景下，我们的 SaaS 后台只需接入**一个账号**（Trail 所在账号），即可采集整个 Organization 所有成员账号的日志。

```
[成员账号 A] ──┐
[成员账号 B] ──┤── CloudTrail Organization Trail
[成员账号 C] ──┘         ↓
                [管理账号 / 委托管理员账号]
                    ├── S3 Bucket（所有账号日志）
                    │     └── AWSLogs/{OrgId}/{AccountId}/CloudTrail/...
                    └── SNS → SQS → SaaS 后台
```

**与标准接入流程的差异**：SaaS 后台只需 AssumeRole 到**管理账号（或委托管理员账号）**的 IAM Role，即可同时读取所有成员账号的日志，无需为每个成员账号单独授权。

### 7.3 客户需要额外配置的内容

**前提**：客户已通过 AWS Organizations 管理多账号，且在管理账号中启用了 Organization Trail。

---

**Organization Trail 的 S3 Bucket Policy 要求**

Organization Trail 写入 S3 时，路径中包含 Org ID，Bucket Policy 必须明确允许此路径：

```yaml
  OrgCloudTrailBucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref CloudTrailBucket
      PolicyDocument:
        Statement:
          - Sid: AWSCloudTrailAclCheck
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:GetBucketAcl
            Resource: !GetAtt CloudTrailBucket.Arn
          - Sid: AWSCloudTrailWrite
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:PutObject
            # Organization Trail 路径包含 OrgId，需同时覆盖两种路径
            Resource:
              - !Sub '${CloudTrailBucket.Arn}/AWSLogs/${AWS::AccountId}/*'
              - !Sub '${CloudTrailBucket.Arn}/AWSLogs/${OrgId}/*'
            Condition:
              StringEquals:
                s3:x-amz-acl: bucket-owner-full-control
          - Sid: AWSCloudTrailOrgCheck
            Effect: Allow
            Principal:
              Service: cloudtrail.amazonaws.com
            Action: s3:GetBucketAcl
            Resource: !GetAtt CloudTrailBucket.Arn
            Condition:
              StringEquals:
                aws:SourceOrgID: !Ref OrgId
```

---

**SaaS 后台 IAM Role 权限（Organization Trail 场景）**

后台 Role 的 S3 权限需覆盖带 Org ID 的路径，并额外增加 `organizations:DescribeOrganization` 以支持后台自动探测 Org ID：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOrgCloudTrailLogs",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::{cloudtrail-bucket-name}",
        "arn:aws:s3:::{cloudtrail-bucket-name}/AWSLogs/{OrgId}/*",
        "arn:aws:s3:::{cloudtrail-bucket-name}/AWSLogs/{AccountId}/*"
      ]
    },
    {
      "Sid": "ConsumeNotificationQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:{region}:{account-id}:cloudtrail-log-queue"
    },
    {
      "Sid": "DescribeOrganization",
      "Effect": "Allow",
      "Action": [
        "organizations:DescribeOrganization",
        "organizations:ListAccounts"
      ],
      "Resource": "*"
    }
  ]
}
```

> `organizations:ListAccounts` 允许后台枚举所有成员账号，在解析日志时可将 AccountId 映射为客户配置的账号别名，提升告警可读性。

---

**客户在 SaaS 平台填写的额外信息**

| 字段 | 说明 |
|---|---|
| Organization ID | 客户 AWS Organizations 的 Org ID（格式：`o-xxxxxxxxxxxx`），后台据此构建正确的 S3 路径前缀 |
| Trail 所在账号 | 管理账号 ID 或委托管理员账号 ID（即 SaaS 后台需要 AssumeRole 的账号） |
| S3 Bucket 所在账号 | 通常与 Trail 所在账号相同；若客户使用独立日志归档账号，需单独填写 |

### 7.4 后台采集注意事项

**① 路径前缀探测**

后台在首次扫描 S3 时，应先探测路径层级（判断是 `AWSLogs/{AccountId}/` 还是 `AWSLogs/{OrgId}/{AccountId}/`），避免硬编码路径导致漏拉。探测逻辑：先尝试 `ListObjectsV2` `Prefix=AWSLogs/{OrgId}/`，若返回结果则为 Organization Trail；否则按单账号路径处理。

**② 成员账号维度的告警路由**

所有成员账号的日志混合在同一个 SQS 队列和 S3 Bucket 中，后台需在解析阶段根据每条事件的 `recipientAccountId` 或 S3 路径中的 AccountId 进行账号维度路由，确保告警正确归属到对应成员账号，而不是混同为管理账号的事件。

**③ 新成员账号自动覆盖**

Organization Trail 会自动覆盖新加入的成员账号，无需客户重新配置采集。后台应定期调用 `organizations:ListAccounts` 刷新账号列表，确保新账号的日志能被正确解析和路由。

**④ 委托管理员账号（推荐）**

AWS 支持将 CloudTrail 的管理权限委托给非管理账号（Security Account 或 Log Archive Account），使 SaaS 后台无需直接 AssumeRole 到管理账号（最敏感账号）即可完成日志采集。建议向有条件的客户推荐此配置：

```
[管理账号] ── 委托 CloudTrail 管理 ──→ [安全账号 / 日志归档账号]
                                              ↑
                                        SaaS 后台 AssumeRole 到这里
```

> **参考文档**：
> - [Creating a trail for an organization](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/creating-trail-organization.html)
> - [Delegated administrator for AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-delegated-administrator.html)
> - [AWS Organizations concepts – Management account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_getting-started_concepts.html)

---

*本文档从《AWS 云日志调研报告》第 3 章独立拆分。完整的日志分类、能力边界、日志示例及安全厂商经验分析请参阅主报告。*

*所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有产品或定价变更，以 AWS 官方最新文档为准。*