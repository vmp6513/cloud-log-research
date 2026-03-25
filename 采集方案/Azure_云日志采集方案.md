# Azure 云日志采集流程

> **文档版本**：1.0
> **创建时间**：2026-03
> **来源文档**：Azure 云日志调研报告（第 3 章独立提取）
> **适用读者**：产品侧 / 后台研发 / 安全运营团队
> **核心定位**：SaaS 后台从客户 Azure 订阅采集 Activity Log 及 Resource Logs 的完整操作指南

---

## 目录

1. [章节背景](#1-章节背景)
2. [授权模型](#2-授权模型)
3. [采集方案总览与选型建议](#3-采集方案总览与选型建议)
4. [方案一：诊断设置 → Event Hub（主流方案）](#4-方案一诊断设置--event-hub主流方案)
5. [方案二：诊断设置 → Storage Account（存档方案）](#5-方案二诊断设置--storage-account存档方案)
6. [方案三：方案一 + 方案二（双轨方案）](#6-方案三方案一--方案二双轨方案)
7. [方案对比与选型建议](#7-方案对比与选型建议)
8. [参考文档](#8-参考文档)

---

## 0. 业务背景和设计约束

本文档从**客户（运维/开发人员）视角**出发，说明"我们采集的方案是什么，客户需要在自己的云账号里做什么，才能让我们的采集后台拿到日志"。

**采集后台的工作模式**：我们的后台程序部署在独立 SaaS 平台上，不会进入客户的云环境。当客户接入一个或多个云账号后，后台基于账号维度拉取日志。

**关键设计约束**：
- 优先设计**消息队列监听（Push-Pull）**的方案，客户在自己的账号将日志推送到某个队列（或类似的服务）中，通知部署在 SaaS 平台上订阅的后台程序进行日志采集。
- 设计方案前需要对云厂商提供的日志服务进行能力分析，例如是否支持流式拉取，是否提供 API 级别的筛选，是否支持重试等影响方案选型的关键能力。
- 至少制定两套采集方案进行对比分析，制定的方案最好在业界内有使用参考。
- 设计的方案需要说明客户侧关注内容：如何配置，成本等。
- 方案本身需考虑是否支持筛选、日志持久化、日志规模、延迟、限流、日志丢失这些基本问题，方案本身具备可扩展、高可用、可复用等特性。

---

## 2. 授权模型

与 AWS 的 AssumeRole 类似，Azure 推荐的 SaaS 集成授权方式是**服务主体（Service Principal）+ 联合凭证（Federated Credential）或客户端密钥**。

| 模型 | 方式描述 | 推荐程度 | 适用场景 |
|---|---|---|---|
| **模型 A：跨租户应用注册 + Federated Credential / 多租户服务主体** | 客户在自己的 Entra 租户中，为我们 SaaS 平台的**多租户应用**授予同意（Admin Consent），生成该租户的服务主体，授予必要的 RBAC 角色；后台使用 Workload Identity Federation 获取临时令牌，无需管理长期密钥 | ✅ **强烈推荐** | 大多数企业场景；最安全，无长期密钥泄露风险 |
| **模型 B：客户自建单租户应用 + 客户端密钥（Client Secret）** | 客户在自己的 Entra 租户中注册一个专用应用，生成 Client Secret 并提供给我们；后台使用此 Secret 获取 Access Token | ⚠️ 次选 | 合规限制无法用多租户应用；需强制要求 Secret 最大有效期 ≤ 1 年并定期轮换 |

**模型 A 的客户操作（一次性）**：

客户需要在 Entra Admin Center 对我们的多租户应用执行 Admin Consent，并在目标订阅上为生成的服务主体分配 RBAC 角色。

**后台调用流程**（模型 A，Workload Identity Federation）：

```
[SaaS 后台（持有 SaaS 自身的 App 凭证）]
    ↓ 使用 Workload Identity Federation 获取 OBO 令牌
    ↓ 向客户的 Entra 租户请求 Access Token
[客户租户的服务主体（Service Principal）]
    ↓ 返回 Access Token（有效期通常 1 小时）
[SaaS 后台持有 Access Token]
    ↓ 调用 Azure REST API / Event Hub 消费 API
[客户订阅内的 Storage Account 或 Event Hub]
```

> **注意**：Azure 没有类似 AWS ExternalId 的内置机制来防止混淆代理人攻击，但通过多租户应用的 Admin Consent 流程（每个租户必须显式授权）和 Workload Identity Federation（无 Client Secret，基于 OIDC 信任）可以实现等效的安全保证。

---

## 3. 采集方案总览与选型建议

基于准实时定位，提供以下三套方案，客户根据复杂度和实时性需求选择：

| 方案 | 核心链路 | 端到端延迟 | 客户配置复杂度 | 客户侧增量成本 | 推荐场景 |
|---|---|---|---|---|---|
| **方案一**：诊断设置 → Event Hub（主流方案） | Activity Log + Resource Logs → Diagnostic Settings → Event Hub → 后台消费 | **2–8 分钟** | ⭐⭐（中，约 4 步） | Event Hub 吞吐量单元费（低–中） | 绝大多数客户，需要准实时告警 |
| **方案二**：诊断设置 → Storage Account（存档方案） | Activity Log + Resource Logs → Diagnostic Settings → Storage Account → 后台轮询 | **10–30 分钟** | ⭐（低，约 3 步） | Storage 存储费（低） | 对实时性要求低；最简单配置；适合补充长期存档 |
| **方案三**：方案一 + 方案二（双轨方案） | Event Hub 提供实时流；Storage Account 提供兜底存档和日志完整性保证 | 实时路径 2–8 分钟 | ⭐⭐⭐（较高，两套配置） | 两套成本之和（仍偏低） | 对日志完整性要求高、同时需要实时告警的客户 |

> 三套方案均兼容授权模型 A（服务主体）和模型 B（Client Secret）。

**业界使用情况参考**：

| 方案 | 厂商使用案例 | 文档链接 |
|---|---|---|
| **方案一**（诊断设置→Event Hub） | **Splunk**：官方 Splunk Add-on for Microsoft Cloud Services 使用 Event Hub 接入 Azure Activity Log 和 Resource Logs，这是 Microsoft 官方推荐的 SIEM 集成路径 | [Stream Azure monitoring data to an event hub](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs) |
| **方案一**（诊断设置→Event Hub） | **Microsoft 官方**：微软文档明确推荐将监控数据路由到 Event Hub 以集成 Splunk、QRadar、SumoLogic 等外部安全工具 | [Stream Azure monitoring data to an event hub and external partners](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs) |
| **方案一**（诊断设置→Event Hub） | **ArcSight、IBM QRadar**：均原生支持通过 Event Hub 消费 Azure Activity Log | [Azure Monitor partner integrations](https://learn.microsoft.com/en-us/azure/azure-monitor/partners) |
| **方案二**（诊断设置→Storage） | **所有早期/轻量 SIEM**：Storage 轮询方式是最早、最简单的 Azure 日志采集路径，用于低频分析和合规存档 | [Send activity log to storage account](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log) |

**选型决策树**：

```
客户是否需要 ≤ 15 分钟的准实时告警？
    ├─ 是 → 方案一（Event Hub）；对日志完整性要求极高则选方案三
    └─ 否 → 方案二（Storage 轮询，最简单配置）
```

---

## 4. 方案一：诊断设置 → Event Hub（主流方案）

这是微软官方推荐的 SIEM 集成方式，主流安全厂商（Splunk、QRadar、ArcSight）均原生支持此路径。

### 4.1 整体链路

```
[Azure ARM 操作发生]
      ↓ 秒级（Event Hub 近实时触发）
[Diagnostic Settings（诊断设置）：将日志流路由到 Event Hub]
      ↓
[Event Hub Namespace / Event Hub（消息队列，按吞吐量单元计费）]
      ↓ 后台 AMQP/Kafka 消费（长连接）
[SaaS 采集后台：解析 JSON → 入库]
```

端到端延迟：诊断设置推送（秒级）+ Event Hub 消费（秒级）+ 少量处理延迟 = **约 2–8 分钟**。

> **注意**：Activity Log 到达 Event Hub 的延迟受 Azure 平台处理时间影响，实际延迟通常在 2–5 分钟，但不提供 SLA 保证。

### 4.2 客户需要配置的内容

---

**前置步骤：创建 Event Hub Namespace 和 Event Hub**

> Event Hub Namespace 是 Event Hub 的管理容器，必须先于诊断设置创建。

控制台操作（约 5 分钟）：

1. Azure Portal → 搜索「Event Hubs」→「创建 Event Hubs 命名空间」
2. 定价层：选「标准」（Basic 层不支持消费者组，**不可用**）
3. 吞吐量单元（TU）：建议初始设为 **1 TU**，开启「Auto-inflate」自动扩容（最大 10–20 TU）
4. 创建 Event Hub：在命名空间内创建名为 `activity-logs` 的 Event Hub（分区数建议 4–8）

ARM Template：

```json
{
  "type": "Microsoft.EventHub/namespaces",
  "apiVersion": "2022-01-01-preview",
  "name": "[parameters('namespaceName')]",
  "location": "[parameters('location')]",
  "sku": { "name": "Standard", "tier": "Standard", "capacity": 1 },
  "properties": {
    "isAutoInflateEnabled": true,
    "maximumThroughputUnits": 20
  }
}
```

---

**第 1 步：配置 Activity Log 诊断设置**

控制台操作（约 3 分钟）：

1. Azure Portal → Monitor → Activity Log → **Export Activity Logs**
2. 选择订阅 → 点击「添加诊断设置」
3. 日志类别：勾选 `Administrative`（必选）；`Security`、`Policy`（按需）
4. 目标：勾选「Stream to an event hub」→ 选择刚创建的 Event Hub Namespace 和 `activity-logs` Event Hub
5. 保存

> ⚠️ 重要：Activity Log 的诊断设置**只能在订阅级别配置**，不能在资源组级别配置。若客户有多个订阅，需要为**每个订阅单独配置**，或在 Management Group 级别统一配置（会有重复事件问题）。

ARM Template（订阅级别诊断设置）：

```json
{
  "type": "Microsoft.Insights/diagnosticSettings",
  "apiVersion": "2021-05-01-preview",
  "name": "activity-log-to-eventhub",
  "properties": {
    "eventHubAuthorizationRuleId": "[parameters('eventHubAuthRuleId')]",
    "eventHubName": "activity-logs",
    "logs": [
      { "category": "Administrative", "enabled": true },
      { "category": "Security", "enabled": true },
      { "category": "Policy", "enabled": true },
      { "category": "Alert", "enabled": true }
    ]
  }
}
```

---

**第 2 步：授权 SaaS 后台读取 Event Hub**

为 SaaS 后台的服务主体分配以下权限（最小权限原则）：

| 权限 | 作用域 | 用途 |
|---|---|---|
| `Azure Event Hubs Data Receiver` | Event Hub Namespace | 消费 Event Hub 中的消息 |
| `Reader`（可选） | 订阅 | 用于后台自动发现订阅下的日志配置状态 |

ARM 角色分配（通过 Portal 或 CLI）：

```bash
# 为 SaaS 服务主体授予 Event Hub 消费权限
az role assignment create \
  --assignee {saas-service-principal-object-id} \
  --role "Azure Event Hubs Data Receiver" \
  --scope /subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{namespace}
```

---

**第 3 步：在 SaaS 平台填写接入信息**

| 字段 | 内容（模型 A）| 内容（模型 B）|
|---|---|---|
| 租户 ID（Tenant ID） | 客户的 Entra 租户 ID | 同左 |
| 服务主体 App ID | 多租户应用在客户租户的 Object ID | 客户注册的应用 App ID |
| 客户端密钥 | 无需（使用 Federated Credential） | 客户提供的 Client Secret |
| Event Hub Namespace | 客户创建的 Namespace 名称 | 同左 |
| Event Hub 名称（Activity Log） | `activity-logs` | 同左 |
| 订阅 ID | 客户订阅 ID | 同左 |

### 4.3 返回格式

Event Hub 中收到的消息体是 JSON 格式，包含一个 `records` 数组（与 Activity Log 存储到 Storage Account 时的 Schema 一致）：

```json
{
  "records": [
    {
      "time": "2024-03-15T09:12:30.123Z",
      "resourceId": "/subscriptions/xxxx/resourceGroups/prod-rg/providers/Microsoft.Authorization/roleAssignments/yyyy",
      "operationName": "Microsoft.Authorization/roleAssignments/write",
      "category": "Administrative",
      "resultType": "Success",
      "resultSignature": "OK",
      "durationMs": 123,
      "callerIpAddress": "203.0.113.45",
      "correlationId": "aaaa-bbbb-cccc-dddd",
      "identity": {
        "authorization": {
          "scope": "/subscriptions/xxxx/resourceGroups/prod-rg",
          "action": "Microsoft.Authorization/roleAssignments/write",
          "evidence": {
            "role": "Owner",
            "roleAssignmentScope": "/subscriptions/xxxx",
            "roleDefinitionId": "8e3af657-a8ff-443c-a75c-2fe8c4bcb635",
            "principalId": "aaa-bbb-ccc",
            "principalType": "User"
          }
        },
        "claims": {
          "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn": "admin@contoso.com"
        }
      },
      "level": "Informational",
      "operationId": "zzzz-yyyy-xxxx",
      "properties": {
        "statusCode": "OK",
        "statusMessage": "{\"statusCode\":\"OK\"}",
        "requestbody": ""
      },
      "subscriptionId": "xxxx",
      "tenantId": "mmmm"
    }
  ]
}
```

**Storage Account 的文件路径格式**：

```
insights-activity-logs/resourceId=/SUBSCRIPTIONS/{subscriptionId}/
    y={year}/m={month}/d={day}/h={hour}/m=00/PT1H.json
```

每个文件按小时切分，包含该小时内所有 Activity Log 事件，后台需按小时路径增量扫描。

### 4.4 关键字段

| 字段 | 类型 | 说明 | 研判用途 |
|---|---|---|---|
| `eventTimestamp` / `time` | string (ISO 8601 UTC) | **事件发生时间**，是研判基准；Portal 中显示为 `eventTimestamp`，Storage/Event Hub 中为 `time` | 必须以此为时间基准，不能用消息到达时间替代 |
| `operationName` | string | 操作名称（如 `Microsoft.Authorization/roleAssignments/write`），**告警规则的核心过滤字段** | 对应 AWS 的 `eventName` |
| `caller` | string | 操作发起者，通常为用户 UPN（如 `user@contoso.com`）或服务主体 Object ID | 身份溯源核心字段；注意服务代调时显示为服务名 |
| `callerIpAddress` | string | 请求来源 IP | ARM 服务代调时为服务 IP，非真实用户 IP |
| `correlationId` | GUID | **关联同一操作的多条记录**（如"开始"和"完成"两条记录），以及跨服务的相关操作 | 关联分析关键字段；用于还原由一次部署触发的多个子操作 |
| `resultType` | string | 操作结果（`Success`、`Failed`、`Accept`）| 失败操作是权限探测和攻击尝试的核心信号 |
| `resourceId` | string | 完整资源 ARN（包含订阅、资源组、资源类型、资源名） | 跨订阅操作分析；`subscriptionId` 子字段用于识别跨订阅活动 |
| `identity.authorization.evidence.role` | string | 执行操作时使用的 RBAC 角色名称 | 高权限角色（Owner / User Access Administrator）执行操作时重点关注 |
| `identity.claims["upn"]` | string | 操作者的用户主名（Email 格式） | 用于关联分析的操作者标识 |
| `subscriptionId` | string | 事件所属订阅 | 跨订阅横向移动识别的关键字段 |
| `tenantId` | string | 事件所属租户 | 多租户场景下的隔离标识 |

> **参考文档**：
> - [Azure Activity Log event schema](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-schema)
> - [AzureActivity table in Log Analytics](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity)

### 4.5 成本说明

客户侧新增的主要费用：

| 费用项 | 计费方式 | 估算（中型客户，单订阅，管理事件） |
|---|---|---|
| Event Hub Standard TU | 约 $0.015/小时/TU（约 $11/月/TU） | 1 TU 通常可满足中型客户，约 **$11–22/月** |
| Event Hub 消息费 | 前 1000 万条/月免费；超出约 $0.028/百万条 | 中型客户约百万条/月，**可忽略不计** |
| Diagnostic Settings 本身 | **免费** | $0 |
| Storage（若同时配置方案二存档） | ~$0.018/GB/月（冷存储更低） | 视保留时长和日志量 |

### 4.6 踩坑与注意事项

**① Activity Log 诊断设置只能在订阅级别配置**

Activity Log 的诊断设置**只能在订阅级别配置**，不能在资源组级别配置。若客户有多个订阅，需要为每个订阅单独配置，或在 Management Group 级别统一配置（会有重复事件问题）。初次接入时容易遗漏部分订阅，导致该订阅的 Activity Log 完全缺失。

**② Activity Log 写操作会产生两条记录，需去重**

对于 Write、Delete 或 Action 类型的操作，开始和成功或失败均会被记录。因此每个写操作产生两条日志（`Started` + `Succeeded/Failed`），告警规则和聚合统计**应仅使用 `resultType = Succeeded / Failed` 的记录**，忽略 `Started` 记录，否则计数翻倍。

**③ Event Hub 中的 Activity Log 格式与 Storage Account 的格式存在细微差异**

Activity Log 写入 Event Hub 时，微软在默认 Event Hub 中混合路由（Activity Log 和 Resource Logs 共用同一路径），建议为 Activity Log 和 Resource Logs **分别创建独立的 Event Hub**，以避免后台解析时的混淆。

**④ Event Hub 消息保留期与积压丢失风险**

Event Hub Standard 层默认保留 1 天，最多 7 天（需付费扩展）；Premium 最长 90 天。**超过保留期的消息被永久删除且无法恢复**。与 AWS SQS 的 DLQ 机制不同，Event Hub 没有原生死信队列概念。若后台故障导致长时间未消费，超期消息将永久丢失。

推荐缓解措施：
- **同时配置方案二（Storage Account）作为兜底**：即使 Event Hub 消息超期丢失，Storage 中仍有完整的日志文件（方案三/双轨方案的核心价值）
- 设置 Event Hub Consumer Lag 监控告警，积压超阈值时自动扩容消费 Worker
- Event Hub Premium 层可将保留期延长至 90 天，显著降低丢失风险

**⑤ 同一 Event Hub 消费者组只能被一个消费者占用**

Event Hub 的分区采用排他锁模式：同一消费者组的同一分区同时只能被**一个消费者实例**持有。若后台部署了多个 Worker 实例，需确保 Worker 数量 ≤ Event Hub 分区数，超出的 Worker 将处于空闲状态。建议后台使用 `EventProcessorClient`（Azure SDK）自动管理分区分配，避免手动管理的复杂性。

> **参考文档**：
> - [Stream Azure monitoring data to an event hub](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs)
> - [Azure Event Hubs quotas and limits](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas)

---

## 5. 方案二：诊断设置 → Storage Account（存档方案）

此方案将 Activity Log 和 Resource Logs 归档到 Blob Storage，后台定时扫描增量文件。配置最简单，延迟较高，适合对实时性要求不高的场景，也常作为方案一的兜底备份。

**与方案一的核心区别**：

| 对比项 | 方案一（Event Hub） | 方案二（Storage Account） |
|---|---|---|
| 延迟 | 2–8 分钟 | 10–30 分钟（文件按小时切分） |
| 客户配置步骤 | 约 4 步（需建 Event Hub） | **约 3 步（仅需 Storage Account）** |
| 消息积压丢失 | 消息超期（1–7 天）永久丢失，**无兜底** | **Storage 文件持久保存，无丢失风险** |
| 采集侧过滤 | ❌ 无，全量推送 | ❌ 无，全量存储 |
| 成本 | Event Hub TU 费 | 仅 Storage 存储费，**更低** |

### 5.1 客户需要配置的内容

**第 1 步：创建 Storage Account（若无）**

建议为日志专用，开启以下设置：
- 访问层：Cool（日志读取频率低，冷存储更便宜）
- Immutable Blob（WORM）：可选，开启后防止历史日志被删除

**第 2 步：配置 Activity Log 诊断设置**

操作与方案一第 1 步相同，目标改为「Archive to storage account」，选择刚创建的 Storage Account。

**第 3 步：授权 SaaS 后台读取 Storage**

```bash
# 为 SaaS 服务主体授予 Storage Blob 读取权限
az role assignment create \
  --assignee {saas-service-principal-object-id} \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}
```

### 5.2 后台的增量扫描逻辑

Storage Account 中的日志文件按**小时**切分，路径格式如下：

```
Activity Log:
  insights-activity-logs/resourceId=/SUBSCRIPTIONS/{subId}/
      y={YYYY}/m={MM}/d={DD}/h={HH}/m=00/PT1H.json

Resource Logs（以 Key Vault 为例）:
  insights-logs-auditevent/resourceId=/SUBSCRIPTIONS/{subId}/RESOURCEGROUPS/{rg}/
      PROVIDERS/MICROSOFT.KEYVAULT/VAULTS/{vaultName}/
      y={YYYY}/m={MM}/d={DD}/h={HH}/m=00/PT1H.json
```

后台应记录上次成功处理的小时时间戳，每次扫描从此时间戳对应的路径开始，增量下载未处理的文件。不同资源类型的 Resource Logs 路径中包含资源提供商路径（`PROVIDERS/...`），后台采集逻辑需支持按资源类型动态拼接路径。

### 5.3 成本说明

| 费用项 | 计费方式 | 估算 |
|---|---|---|
| Storage 存储费 | ~$0.018/GB/月（Cool 层）| 中型客户 Activity Log 约 1–5 GB/月，约 **$0.02–$0.10/月** |
| Blob 读取请求 | $0.004/10,000 次 | 可忽略不计 |
| Diagnostic Settings | **免费** | $0 |

---

## 6. 方案三：方案一 + 方案二（双轨方案）

同时配置 Event Hub（实时）和 Storage Account（存档），Event Hub 提供准实时消费，Storage Account 提供完整的历史兜底。

这是**可靠性要求最高场景下的推荐配置**，原因在于：
- Event Hub 消息超期丢失时，Storage Account 仍有完整记录，后台可从 Storage 补偿扫描
- 适合有合规要求（如需留存日志 1 年以上）的客户
- 客户侧需要为同一诊断设置同时指定两个目标（Azure 支持，每个诊断设置最多 5 个目标）

---

## 7. 方案对比与选型建议

| 维度 | 方案一（Event Hub） | 方案二（Storage 轮询） | 方案三（双轨） |
|---|---|---|---|
| **延迟** | 2–8 分钟 | 10–30 分钟 | 实时路径 2–8 分钟 |
| **客户配置步骤** | 约 4–5 步 | 约 3 步，最简单 | 约 6–7 步 |
| **事件类型覆盖** | Activity Log + Resource Logs（按需） | 同左 | 同左 |
| **历史日志留存** | ❌ 无（消息超期即删） | ✅ 有（Blob 持久存储） | ✅ 有 |
| **积压导致丢失** | **不可补救**（超期 = 永久丢失；无 DLQ 机制） | **无此风险**（Blob 持久，仅影响延迟） | **可补救**（Storage 兜底） |
| **采集侧过滤** | ❌ 不支持（全量推送到 Event Hub） | ❌ 不支持 | ❌ 不支持 |
| **客户侧增量成本** | Event Hub TU 费（低–中） | Storage 费（极低） | 两者之和（仍偏低） |
| **业界代表厂商** | Splunk、QRadar、ArcSight | 早期/轻量 SIEM | Sentinel 等高级 SIEM |

> **与 AWS 的关键差异**：Azure Event Hub **没有 SQS 的死信队列（DLQ）机制**，消息超期后直接永久删除，无任何兜底。这是方案三（双轨）在 Azure 场景下比 AWS 方案一更重要的原因——Storage Account 是唯一可靠的日志兜底手段。

**后台工程建议（针对方案一）**：
- **预防积压**：监控 Event Hub 的 Consumer Lag 指标，超阈值时自动扩容消费 Worker（Worker 数 ≤ Event Hub 分区数）
- **失败处理**：后台消费失败时应实现重试逻辑（指数退避），持久化"消费位点"（Checkpoint），重启后从 Checkpoint 继续消费
- **最终一致（仅方案三）**：后台周期性对 Storage Account 执行增量路径扫描，与已处理文件列表对比，补拉因 Event Hub 消息超期或消费异常而漏掉的文件

---

## 8. 参考文档

| 类别 | 文档名称 | 链接 |
|---|---|---|
| **采集方案** | Diagnostic settings in Azure Monitor | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings |
| | Create diagnostic settings | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/create-diagnostic-settings |
| | Stream Azure monitoring data to Event Hub | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs |
| | Enable Diagnostic Settings at scale（Azure Policy）| https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostics-settings-policies-deployifnotexists |
| **Event Hub** | Azure Event Hubs features | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features |
| | Azure Event Hubs quotas and limits | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas |
| | Azure Event Hubs scalability guide | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability |
| | Azure Event Hubs Auto-inflate | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-auto-inflate |
| **日志 Schema** | Azure Activity Log event schema | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-schema |
| | AzureActivity table reference（Log Analytics） | https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity |
| **身份授权** | Managed identities for Azure resources | https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview |

---

*本文档所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有产品或定价变更，以微软官方最新文档为准。*