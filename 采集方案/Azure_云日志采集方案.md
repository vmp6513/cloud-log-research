# Azure 云日志采集方案

> **文档状态**：v2.0（基于 v1.0 全面对齐 AWS 云日志采集方案）
> **调研时间**：2026-03
> **作者**：云安全专家调研
> **适用读者**：产品侧 / 后台研发团队 / 接入实施团队
> **核心聚焦**：Activity Log 与 Resource Logs 采集流程、授权模型、三套方案对比与工程最佳实践

---

## 目录

0. [业务背景和设计约束](#0-业务背景和设计约束)
1. [授权模型：后台如何访问客户订阅](#1-授权模型后台如何访问客户订阅)
2. [Azure Monitor 日志能力分析](#2-azure-monitor-日志能力分析)
   - 2.1 [日志交付机制](#21-日志交付机制)
   - 2.2 [日志类型与覆盖范围](#22-日志类型与覆盖范围)
   - 2.3 [直接查询 API 的能力限制（排除法）](#23-直接查询-api-的能力限制排除法)
3. [采集方案总览与选型建议](#3-采集方案总览与选型建议)
4. [方案一：诊断设置 → Event Hub（主流方案）](#4-方案一诊断设置--event-hub主流方案)
   - 4.1 [整体链路](#41-整体链路)
   - 4.2 [客户需要配置的内容](#42-客户需要配置的内容)
   - 4.3 [返回格式](#43-返回格式)
   - 4.4 [关键字段说明](#44-关键字段说明)
   - 4.5 [日志规模估算](#45-日志规模估算)
   - 4.6 [重试与可靠性](#46-重试与可靠性)
   - 4.7 [成本说明](#47-成本说明)
   - 4.8 [踩坑与注意事项](#48-踩坑与注意事项)
5. [方案二：诊断设置 → Storage Account（存档方案）](#5-方案二诊断设置--storage-account存档方案)
   - 5.1 [整体链路](#51-整体链路)
   - 5.2 [客户需要配置的内容](#52-客户需要配置的内容)
   - 5.3 [返回格式](#53-返回格式)
   - 5.4 [日志规模估算](#54-日志规模估算)
   - 5.5 [重试与可靠性](#55-重试与可靠性)
   - 5.6 [成本说明](#56-成本说明)
   - 5.7 [踩坑与注意事项](#57-踩坑与注意事项)
6. [方案三：方案一 + 方案二（双轨方案）](#6-方案三方案一--方案二双轨方案)
   - 6.1 [整体链路](#61-整体链路)
   - 6.2 [客户需要配置的内容](#62-客户需要配置的内容)
   - 6.3 [成本说明](#63-成本说明)
7. [方案对比与选型建议](#7-方案对比与选型建议)
8. [多订阅场景：Management Group 统一接入](#8-多订阅场景management-group-统一接入)
   - 8.1 [Management Group 统一接入与单订阅接入的关键差异](#81-management-group-统一接入与单订阅接入的关键差异)
   - 8.2 [接入架构](#82-接入架构)
   - 8.3 [客户需要额外配置的内容](#83-客户需要额外配置的内容)
   - 8.4 [后台采集注意事项](#84-后台采集注意事项)
9. [参考文档](#9-参考文档)

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

## 1. 授权模型：后台如何访问客户订阅

在客户接入任何采集方案之前，必须先完成对我们 SaaS 平台的**访问授权**。我们支持以下两种模型，客户可按实际情况选择：

### 模型 A：跨租户多租户应用 + Workload Identity Federation（推荐）

这是微软官方推荐的 SaaS 集成授权方式，等价于 AWS 的跨账号 AssumeRole。客户在自己的 Entra 租户中，对我们 SaaS 平台的**多租户应用**执行 Admin Consent，生成该租户的服务主体（Service Principal），并授予必要的 RBAC 角色；后台使用 Workload Identity Federation（基于 OIDC 信任，无需 Client Secret）获取临时 Access Token。

**安全优势**：
- 无需管理任何长期密钥，Access Token 有效期通常 1 小时，泄露后自动失效
- 客户可随时撤销 Admin Consent 或移除角色分配，立即切断我们的访问
- Workload Identity Federation 的信任链基于 OIDC，不存在静态密钥泄露风险

**客户需要做的事（一次性）**：

1. 在 Entra Admin Center → 企业应用程序 → 找到我们的多租户应用 → 执行 **Admin Consent**
2. 在目标订阅（或资源组/资源）上，为生成的服务主体分配对应 RBAC 角色（见下方最小权限清单）

**后台调用流程**：

```
[SaaS 后台（持有 SaaS 自身的 App 凭证）]
    ↓ 使用 Workload Identity Federation 向客户的 Entra 租户请求 Access Token
[客户租户的服务主体（Service Principal）]
    ↓ 返回 Access Token（有效期通常 1 小时）
[SaaS 后台持有 Access Token]
    ↓ 调用 Azure REST API / Event Hub 消费 API / Blob Storage API
[客户订阅内的 Event Hub 或 Storage Account]
```

> **关于混淆代理人攻击**：Azure 没有类似 AWS ExternalId 的内置机制，但 Admin Consent 流程本身要求每个客户租户**显式授权**我们的应用，且 Workload Identity Federation 的 OIDC 信任绑定到特定的 SaaS 平台身份，等效地防止了第三方冒充我们发起授权请求。

### 模型 B：客户自建单租户应用 + Client Secret（次选）

客户在自己的 Entra 租户中注册一个专用应用，生成 Client Secret 并提供给我们；后台使用此 Secret 获取 Access Token。

**适用场景**：合规限制无法使用多租户应用，或客户处于简单单订阅架构。

**安全注意事项（必须向客户明确告知）**：
- Client Secret 是长期有效的静态凭证，一旦泄露不会自动失效，需立即手动撤销
- Secret 最大有效期建议设为 **1 年**，并在到期前提前轮换（可在 Entra 中配置到期提醒）
- 应为采集专用，**权限必须最小化**（仅授予读取日志所需角色，禁止任何写权限）
- 建议客户在 Entra 审计日志中监控该应用的登录与 Token 颁发记录

### 按方案的最小 RBAC 权限清单

完成授权模型配置后，根据客户选择的采集方案，为服务主体（模型 A）或应用（模型 B）分配以下最小权限：

| 采集方案 | 所需 RBAC 角色 | 授权作用域 |
|---|---|---|
| **方案一**（Event Hub） | `Azure Event Hubs Data Receiver` | Event Hub Namespace |
| **方案二**（Storage Account） | `Storage Blob Data Reader` | Storage Account |
| **方案三**（双轨） | `Azure Event Hubs Data Receiver` + `Storage Blob Data Reader` | 各自对应资源 |
| **多订阅场景**（选配） | `Reader` | Management Group 级别 |

> `Reader` 角色用于后台自动枚举订阅列表、发现日志配置状态，非必须但推荐在多订阅场景中配置。

---

## 2. Azure Monitor 日志能力分析

本章对 Azure Monitor 的核心能力进行分析，作为后续方案选型的决策依据。

### 2.1 日志交付机制

Azure Monitor 支持三种日志交付路径，能力差异显著：

| 交付路径 | 交付延迟 | 是否持久化 | 是否支持流式消费 | 是否支持源侧过滤 |
|---|---|---|---|---|
| **写入 Storage Account** | 10–30 分钟（按小时批量写入） | ✅ 持久化，生命周期由 Blob 策略管理 | ❌ 不支持；需后台定时轮询 | ❌ 不支持；全量写入，过滤在消费侧 |
| **投递 Event Hub** | 2–8 分钟（近实时） | ❌ 无持久化；Standard 层保留 1–7 天，Premium 层最长 90 天 | ✅ 支持；AMQP/Kafka 长连接消费 | ❌ 不支持；诊断设置只能按日志 Category 整体开关，**不能按字段内容过滤** |
| **写入 Log Analytics Workspace** | 2–5 分钟 | ✅ 持久化（默认 30 天，可扩展至 2 年） | ❌ 不适合外部采集；无原生推送机制，只能通过 REST API 查询 | ✅ 支持 KQL 查询过滤，但面向交互式分析而非流式采集 |

**核心结论**：三条路径互补——Storage 保证完整性，Event Hub 保证实时性，Log Analytics 面向交互式分析。**对于 SaaS 外部采集后台，Log Analytics 不适合作为主采集通道**（详见 §2.3）；采集方案的选型在 Storage 和 Event Hub 之间展开。

> **与 AWS 的关键差异**：Azure 诊断设置本身**不支持字段级过滤**（无法像 AWS EventBridge 规则那样只推送特定 `operationName` 的事件），只能按日志 Category 整体开关。若需字段级过滤，只能在消费侧（后台）实现。这是 Azure 与 AWS 方案二（EventBridge 规则精确过滤）的关键差异——Azure 三套方案的过滤能力相同，均在消费侧实现。

### 2.2 日志类型与覆盖范围

Azure Monitor 记录两类与安全审计相关的日志：

| 日志类型 | 说明 | 写入 Storage | 投递 Event Hub |
|---|---|---|---|
| **Activity Log（控制平面）** | 订阅级别的所有 ARM 操作（Create/Update/Delete 等），等同于 AWS CloudTrail 管理事件；平台**默认保留 90 天**，无需额外配置即可查询历史；诊断设置需手动配置才能导出到 Storage 或 Event Hub | ✅ | ✅ |
| **Resource Logs（数据平面）** | 资源级别的操作日志（如 Key Vault 密钥访问、Storage Blob 读写、NSG 流日志、SQL 审计等），等同于 AWS CloudTrail 数据事件；**默认不开启**，需逐资源配置诊断设置，按量计费 | ✅ | ✅ |

> **选型关键约束**：三套采集方案对 Activity Log 和 Resource Logs 的覆盖范围**完全一致**，差异仅在于实时性和可靠性，不存在某方案"看不到某类日志"的情况。这与 AWS 不同——AWS 方案二（EventBridge）无法直接覆盖数据事件，Azure 无此限制。

### 2.3 直接查询 API 的能力限制（排除法）

Azure Monitor 提供 `List Activity Logs` REST API，表面上看可以直接拉取日志，但有严格限制使其**不适合作为持续采集通道**：

- **保留期仅 90 天**：超期数据不可查，无法支持历史回填
- **严格限流**：每订阅 5 次请求/分钟，每次返回最多 100 条；中型企业日均数万条事件，仅靠此 API 拉取需数小时，远超限流上限
- **无增量拉取机制**：没有游标（Cursor）或 Continuation Token，大时间窗口查询需手动分页，实现复杂且在故障恢复场景下不可靠
- **纯轮询模式**：无推送通知机制，无法感知"有新事件"，只能定时全量比对，效率低且延迟不可控

**结论**：`List Activity Logs` API 适合交互式查询和小批量验证（如控制台搜索、偶发性排查），**不适合作为持续采集通道**。高吞吐、低延迟的采集场景必须通过 Storage 或 Event Hub 路径实现。Log Analytics 的 REST API 同理——虽然支持 KQL 复杂查询，但其无推送机制、查询配额有限的特性同样不适合流式采集。

---

## 3. 采集方案总览与选型建议

基于准实时（2–30 分钟）的产品定位，我们提供三套采集方案，客户根据自身环境和接受的配置复杂度选择：

| 方案 | 核心链路 | 端到端延迟 | 覆盖范围 | 客户配置复杂度 | 客户侧增量成本 | 推荐场景 |
|---|---|---|---|---|---|---|
| **方案一**：诊断设置 → Event Hub（主流方案） | Activity Log + Resource Logs → Diagnostic Settings → Event Hub → 后台消费 | **2–8 分钟** | Activity Log ✅ · Resource Logs ✅（需逐资源开启） | ⭐⭐（中，约 4 步） | Event Hub TU 费（低–中） | 绝大多数客户；需要准实时告警 |
| **方案二**：诊断设置 → Storage Account（存档方案） | Activity Log + Resource Logs → Diagnostic Settings → Storage Account → 后台轮询 | **10–30 分钟** | Activity Log ✅ · Resource Logs ✅（需逐资源开启） | ⭐（低，约 3 步） | Storage 存储费（极低） | 对实时性要求低；最简单配置；适合补充长期存档 |
| **方案三**：方案一 + 方案二（双轨方案） | Event Hub 提供实时流；Storage Account 提供兜底存档 | 实时路径 2–8 分钟 | Activity Log ✅ · Resource Logs ✅（需逐资源开启） | ⭐⭐⭐（较高，两套配置） | 两套成本之和（仍偏低） | 对日志完整性要求高、同时需要实时告警的客户 |

> 三套方案均兼容授权模型 A（Workload Identity Federation）和模型 B（Client Secret），授权配置见 §1。

**业界使用情况参考**：

| 方案 | 厂商使用案例 | 文档链接 |
|---|---|---|
| **方案一**（诊断设置→Event Hub） | **Splunk**：官方 Splunk Add-on for Microsoft Cloud Services 使用 Event Hub 接入 Azure Activity Log 和 Resource Logs | [Splunk Add-on for Microsoft Cloud Services](https://docs.splunk.com/Documentation/AddOns/released/MSCloudServices/About) |
| **方案一**（诊断设置→Event Hub） | **Microsoft 官方**：微软文档明确推荐将监控数据路由到 Event Hub 以集成 Splunk、QRadar、SumoLogic 等外部安全工具 | [Stream Azure monitoring data to an event hub](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/stream-monitoring-data-event-hubs) |
| **方案一**（诊断设置→Event Hub） | **IBM QRadar、ArcSight**：均原生支持通过 Event Hub 消费 Azure Activity Log | [Azure Monitor partner integrations](https://learn.microsoft.com/en-us/azure/azure-monitor/partners) |
| **方案二**（诊断设置→Storage） | 早期/轻量 SIEM 的标准路径，也是所有支持长期合规存档场景的基础方案 | [Send activity log to storage account](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log) |
| **方案三**（双轨） | **Microsoft Sentinel**：官方推荐同时配置 Event Hub（实时摄取）和 Storage（历史回填）以保证日志完整性，是 Sentinel 大规模部署的标准架构 | [Connect Azure activity logs](https://learn.microsoft.com/en-us/azure/sentinel/data-connectors/azure-activity) |

**选型决策树**：

```
客户是否需要 ≤ 15 分钟的准实时告警？
    ├─ 是 → 客户是否需要保证日志完整性（Event Hub 消息超期后不可补救）？
    │           ├─ 是 → 方案三（双轨）
    │           └─ 否（有强监控保障）→ 方案一（Event Hub）
    └─ 否 → 方案二（Storage 轮询，最简单配置）
```

---

## 4. 方案一：诊断设置 → Event Hub（主流方案）

这是微软官方推荐的 SIEM 集成方式，主流安全厂商（Splunk、QRadar、ArcSight）均原生支持此路径。

### 4.1 整体链路

```
[Azure ARM 操作发生]
      ↓ 秒级（Azure 平台处理后推送）
[Diagnostic Settings（诊断设置）：将日志流路由到 Event Hub]
      ↓ 端到端通常 2–8 分钟
[Event Hub Namespace / Event Hub（消息流，按 TU 计费）]
      ↓ 后台 AMQP/Kafka 消费（长连接，持续消费）
[SaaS 采集后台：解析 JSON → 入库]
```

端到端延迟：Azure 平台处理（通常 2–5 分钟，无严格 SLA）+ Event Hub 消费（秒级）+ 少量处理延迟 = **约 2–8 分钟**。

> **关于延迟的说明**：Activity Log 从操作发生到到达 Event Hub 的延迟受 Azure 平台处理时间影响，通常在 2–5 分钟，但不提供 SLA 保证。不应以此作为选型的决定性因素。

### 4.2 客户需要配置的内容

---

**前置步骤：创建 Event Hub Namespace 和 Event Hub**

> Event Hub Namespace 是 Event Hub 的管理容器，必须先于诊断设置创建。

**控制台操作（约 5 分钟）**：

1. Azure Portal → 搜索「Event Hubs」→「创建 Event Hubs 命名空间」
2. 定价层：选「**标准**」（Basic 层不支持消费者组，**不可用**）
3. 吞吐量单元（TU）：建议初始设为 **1 TU**，开启「Auto-inflate」自动扩容（最大 10–20 TU）
4. 创建 Event Hub：在命名空间内创建名为 `activity-logs` 的 Event Hub（分区数建议 4–8）

**ARM Template**：

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

**控制台操作（约 3 分钟）**：

1. Azure Portal → Monitor → Activity Log → **Export Activity Logs**
2. 选择订阅 → 点击「添加诊断设置」
3. 日志类别：勾选 `Administrative`（必选）；`Security`、`Policy`（按需）
4. 目标：勾选「Stream to an event hub」→ 选择刚创建的 Event Hub Namespace 和 `activity-logs` Event Hub
5. 保存

> ⚠️ 重要：Activity Log 的诊断设置**只能在订阅级别配置**，不能在资源组级别配置。若客户有多个订阅，需要为**每个订阅单独配置**，或在 Management Group 级别统一配置（详见 §8）。

**ARM Template（订阅级别诊断设置）**：

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
| 服务主体 App ID | 多租户应用在客户租户生成的服务主体 Object ID | 客户注册的应用 App ID |
| 客户端密钥 | 无需（使用 Workload Identity Federation） | 客户提供的 Client Secret |
| Event Hub Namespace | 客户创建的 Namespace 名称 | 同左 |
| Event Hub 名称（Activity Log） | `activity-logs` | 同左 |
| 订阅 ID | 客户订阅 ID | 同左 |

### 4.3 返回格式

Event Hub 中收到的消息体是 JSON 格式，包含一个 `records` 数组：

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

> **注意**：Event Hub 中的消息体格式（`records` 数组）与 Storage Account 中的文件格式基本一致，但 Storage 中无外层信封包裹（详见 §5.3）。后台解析逻辑可复用，仅需处理外层结构差异。

### 4.4 关键字段说明

| 字段 | 类型 | 说明 | 研判用途 |
|---|---|---|---|
| `time` / `eventTimestamp` | string (ISO 8601 UTC) | **事件发生时间**，是研判基准；Portal 中显示为 `eventTimestamp`，Storage/Event Hub 中为 `time` | 必须以此为时间基准，不能用消息到达时间替代 |
| `operationName` | string | 操作名称（如 `Microsoft.Authorization/roleAssignments/write`），**告警规则的核心过滤字段** | 对应 AWS 的 `eventName` |
| `caller` | string | 操作发起者，通常为用户 UPN（如 `user@contoso.com`）或服务主体 Object ID | 身份溯源核心字段；注意 ARM 服务代调时显示为服务名而非真实用户 |
| `callerIpAddress` | string | 请求来源 IP | ARM 服务代调时为服务内部 IP，非真实用户 IP |
| `correlationId` | GUID | **关联同一操作的多条记录**（如"开始"和"完成"两条记录），以及跨服务的相关操作 | 关联分析关键字段；用于还原由一次部署触发的多个子操作 |
| `resultType` | string | 操作结果（`Success`、`Failed`、`Accept`、`Start`） | 失败操作是权限探测和攻击尝试的核心信号；`Start` 记录通常可忽略（见 §4.8②） |
| `resourceId` | string | 完整资源路径（包含订阅、资源组、资源类型、资源名） | 跨订阅操作分析；`subscriptionId` 子字段用于识别跨订阅活动 |
| `identity.authorization.evidence.role` | string | 执行操作时使用的 RBAC 角色名称 | 高权限角色（Owner / User Access Administrator）执行操作时重点关注 |
| `identity.claims["upn"]` | string | 操作者的用户主名（Email 格式） | 用于关联分析的操作者标识 |
| `subscriptionId` | string | 事件所属订阅 | 跨订阅横向移动识别的关键字段 |
| `tenantId` | string | 事件所属租户 | 多租户场景下的隔离标识 |

### 4.5 日志规模估算

日志规模直接影响后台 Worker 并发设计和 Event Hub TU 规划：

- Event Hub 每收到一条事件对应一条消息，消息体即完整事件 JSON（约 1–5 KB/条）
- 中型企业（~1000 人）Activity Log 量：约 **5 万–20 万条/天**（Activity Log 粒度比 AWS CloudTrail 粗，单次 ARM 操作通常产生 2 条记录——Start + Succeeded/Failed）
- 若开启 Key Vault、Storage 等高频 Resource Logs，日均事件量可达**百万级**，1 TU 可能不足，需提前规划 TU 上限和分区数
- 若启用 Auto-inflate，TU 可在流量突增时自动扩容，无需手动干预；但 Auto-inflate **不会自动缩容**，峰值后 TU 维持在高位，影响计费

### 4.6 重试与可靠性

| 能力 | 说明 |
|---|---|
| **诊断设置投递重试** | Azure 平台内部有重试机制，但投递延迟和重试行为无公开 SLA，极端情况下可能丢事件 |
| **消费侧重试** | 基于 Offset/Checkpoint 机制；后台消费失败时可从上次 Checkpoint 位置重新消费，不丢数据 |
| **消费失败补救** | ❌ Event Hub 无 DLQ 机制；消息超过保留期（Standard 最长 7 天）后永久删除，无任何补救途径（方案三通过 Storage 兜底） |
| **幂等消费** | 后台基于 `correlationId` + `operationId` 组合去重；Event Hub 在极端情况下可能重复投递同一事件，去重是必须实现的能力 |

### 4.7 成本说明

客户接入方案一后，新增的 Azure 服务费用包括：

| 费用项 | 计费方式 | 估算（中型客户，单订阅） |
|---|---|---|
| Event Hub Standard TU | 约 $0.015/小时/TU（约 $11/月/TU） | 1 TU 通常满足中型客户，约 **$11–22/月** |
| Event Hub 消息费 | 前 1000 万条/月免费；超出约 $0.028/百万条 | 中型客户 Activity Log 约百万条/月，**可忽略不计** |
| Diagnostic Settings | **免费** | $0 |
| Resource Logs（按需开启） | 按各服务独立计费（如 Key Vault、Storage 诊断均有独立费率） | 取决于开启范围，建议仅对高安全价值资源开启 |

### 4.8 踩坑与注意事项

**① Activity Log 诊断设置只能在订阅级别配置**

Activity Log 的诊断设置**只能在订阅级别**，不能在资源组或资源级别配置。若客户有多个订阅，需要为每个订阅单独配置，或通过 Management Group + Azure Policy 统一部署（详见 §8）。初次接入时容易遗漏部分订阅，导致该订阅的 Activity Log 完全缺失且无任何报错提示。

**② 写操作会产生两条记录，需在消费侧去重**

对于 Write、Delete 或 Action 类型的操作，操作开始（`Start`）和结束（`Succeeded`/`Failed`）均会被记录，每个写操作产生**两条日志**。告警规则和聚合统计**应仅使用 `resultType = Succeeded / Failed` 的记录**，忽略 `Start` 记录，否则计数翻倍。

**③ Activity Log 与 Resource Logs 建议使用独立 Event Hub**

Azure 诊断设置允许将不同来源的日志路由到同一 Event Hub，但这会导致后台解析时混淆不同 schema 的消息（Activity Log 与各类 Resource Logs 的字段结构不同）。建议为 Activity Log 和 Resource Logs **分别创建独立的 Event Hub**，并在 SaaS 平台配置中分别填写。

**④ Event Hub 消息保留期与积压丢失风险**

Event Hub Standard 层默认保留 1 天，最多 7 天（需付费扩展）；Premium 层最长 90 天。**超过保留期的消息被永久删除且无法恢复**。与 AWS SQS 不同，Event Hub 没有原生死信队列（DLQ）机制——这是 Azure 与 AWS 方案在可靠性工程上最重要的差异。

推荐缓解措施：
- **同时配置方案二（Storage Account）作为兜底**（即方案三）：Event Hub 消息超期丢失时，Storage 中仍有完整的日志文件可供补偿扫描
- 设置 Event Hub Consumer Lag 监控告警，积压超阈值时自动扩容消费 Worker
- Event Hub Premium 层可将保留期延长至 90 天，显著降低丢失风险，但成本更高

**⑤ 同一消费者组的同一分区同时只能被一个 Worker 持有**

Event Hub 的分区采用排他锁模式：同一消费者组的同一分区同时只能被**一个消费者实例**持有。若后台部署了多个 Worker 实例，需确保 Worker 数量 ≤ Event Hub 分区数，超出的 Worker 将处于空闲等待状态。建议后台使用 Azure SDK 的 `EventProcessorClient` 自动管理分区分配，避免手动管理的复杂性。

**⑥ API 调用与消费限流**

- **Event Hub 账号级配额**：Standard 层每命名空间支持最多 **10,000 个并发连接**，每 TU 支持 1 MB/s 入站、2 MB/s 出站。正常日志量远低于此上限，但若客户账号内有大规模 Resource Logs 同时开启，应提前评估 TU 容量
- **消费侧 Checkpoint**：后台应以合理频率（如每 100 条或每 30 秒）提交一次 Checkpoint，避免后台重启后从头重放大量历史消息（重放会产生重复处理）

> **参考文档（方案一）**：
> - [Stream Azure monitoring data to an event hub](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/stream-monitoring-data-event-hubs)
> - [Azure Event Hubs quotas and limits](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas)
>
> **厂商实现参考（方案一）**：
> - Splunk Add-on for Microsoft Cloud Services（Event Hub 模式）：[Splunk Add-on for Microsoft Cloud Services](https://docs.splunk.com/Documentation/AddOns/released/MSCloudServices/About)
> - Microsoft Sentinel Azure Activity 数据连接器（Event Hub 实时摄取）：[Connect Azure activity logs](https://learn.microsoft.com/en-us/azure/sentinel/data-connectors/azure-activity)
> - Azure Monitor 合作伙伴集成（QRadar、ArcSight）：[Azure Monitor partner integrations](https://learn.microsoft.com/en-us/azure/azure-monitor/partners)

---

## 5. 方案二：诊断设置 → Storage Account（存档方案）

此方案将 Activity Log 和 Resource Logs 归档到 Blob Storage，后台定时扫描增量文件。配置最简单，延迟较高（10–30 分钟），适合对实时性要求不高的场景，也常作为方案一的兜底备份（即方案三）。

**与方案一的核心区别**：

| 对比项 | 方案一（Event Hub） | 方案二（Storage Account） |
|---|---|---|
| 延迟 | 2–8 分钟 | **10–30 分钟**（文件按小时切分，整点后约 10–30 分钟写完） |
| 客户配置步骤 | 约 4 步（需建 Event Hub） | **约 3 步（仅需 Storage Account）** |
| 消息积压丢失 | 消息超期（最长 7 天/90 天）永久丢失，无兜底 | **Storage 文件持久保存，无丢失风险** |
| 采集侧过滤 | ❌ 无，全量推送 | ❌ 无，全量存储 |
| 成本 | Event Hub TU 费（低–中） | 仅 Storage 存储费，**更低** |

### 5.1 整体链路

```
[Azure ARM 操作发生]
      ↓ 10–30 分钟（按小时批量写入）
[Diagnostic Settings（诊断设置）：将日志归档到 Storage Account]
      ↓ 每小时生成一个 PT1H.json 文件
[Storage Account：Blob 文件，按订阅/年/月/日/时切分]
      ↓ 后台定时扫描（每小时增量轮询）
[SaaS 采集后台：下载文件 → 解析 JSON → 入库]
```

端到端延迟：Azure 平台写入 Storage（10–30 分钟）+ 后台轮询周期（最多 1 小时）+ 下载解析（分钟级）= **约 10–90 分钟**（取决于轮询频率，通常控制在 30 分钟内）。

### 5.2 客户需要配置的内容

---

**第 1 步：创建 Storage Account（若无）**

建议为日志专用，开启以下设置：
- 访问层：**Cool**（日志读取频率低，冷存储更便宜）
- **Immutable Blob Storage（WORM）**：可选，开启后防止历史日志被删除或篡改，适合有合规保留要求的客户

---

**第 2 步：配置 Activity Log 诊断设置**

操作与方案一第 1 步相同，目标改为「Archive to storage account」，选择刚创建的 Storage Account。日志类别选择与方案一相同。

---

**第 3 步：授权 SaaS 后台读取 Storage**

```bash
# 为 SaaS 服务主体授予 Storage Blob 读取权限
az role assignment create \
  --assignee {saas-service-principal-object-id} \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}
```

---

**第 4 步：在 SaaS 平台填写接入信息**

| 字段 | 内容（模型 A）| 内容（模型 B）|
|---|---|---|
| 租户 ID（Tenant ID） | 客户的 Entra 租户 ID | 同左 |
| 服务主体 App ID | 多租户应用在客户租户生成的服务主体 Object ID | 客户注册的应用 App ID |
| 客户端密钥 | 无需（使用 Workload Identity Federation） | 客户提供的 Client Secret |
| Storage Account 名称 | 客户创建的 Storage Account 名称 | 同左 |
| 订阅 ID | 客户订阅 ID | 同左 |

### 5.3 返回格式

Storage Account 中的日志文件为 JSON 格式，每个文件包含一个 `records` 数组（字段结构与方案一 Event Hub 消息体中的 `records` 数组**完全一致**，无外层信封包裹）：

```json
{
  "records": [
    {
      "time": "2024-03-15T09:00:00.000Z",
      "resourceId": "/subscriptions/xxxx/...",
      "operationName": "Microsoft.Compute/virtualMachines/write",
      "category": "Administrative",
      "resultType": "Success",
      "callerIpAddress": "203.0.113.45",
      "correlationId": "aaaa-bbbb-cccc-dddd",
      "identity": { ... },
      "subscriptionId": "xxxx",
      "tenantId": "mmmm"
    }
  ]
}
```

**Storage Account 的文件路径格式**：

```
Activity Log:
  insights-activity-logs/resourceId=/SUBSCRIPTIONS/{subscriptionId}/
      y={YYYY}/m={MM}/d={DD}/h={HH}/m=00/PT1H.json

Resource Logs（以 Key Vault 为例）:
  insights-logs-auditevent/resourceId=/SUBSCRIPTIONS/{subId}/RESOURCEGROUPS/{rg}/
      PROVIDERS/MICROSOFT.KEYVAULT/VAULTS/{vaultName}/
      y={YYYY}/m={MM}/d={DD}/h={HH}/m=00/PT1H.json
```

每个文件按小时切分，包含该小时内所有事件。后台需按小时路径增量扫描，记录上次成功处理的小时时间戳，每次从此时间戳对应路径开始下载未处理的文件。不同资源类型的 Resource Logs 路径中包含资源提供商层级（`PROVIDERS/...`），后台采集逻辑需支持按资源类型动态拼接路径。

### 5.4 日志规模估算

- Storage 文件按小时切分，单订阅中型客户每小时文件约 **100 KB–5 MB**（仅 Activity Log）
- 若开启 Key Vault、Storage 等高频 Resource Logs，每小时文件大小可达数十 MB
- 后台单次轮询需处理的文件数量 = 距上次轮询经过的小时数 × 开启诊断设置的资源数量，建议后台并发下载以控制延迟

### 5.5 重试与可靠性

| 能力 | 说明 |
|---|---|
| **投递可靠性** | Storage 写入最终一致，Azure 平台保证不丢文件；极端情况下文件可能延迟写入，但不会永久丢失 |
| **消费侧重试** | 后台记录"已处理的最大时间戳"和已处理文件路径列表；重启后从该时间戳对应路径重新扫描，不漏不重 |
| **消费失败补救** | ✅ Blob 文件持久保存，随时可补拉，**无丢失风险**；这是方案二在可靠性上优于方案一的根本工程优势 |
| **幂等消费** | 后台记录已处理文件路径列表，防止同一文件在轮询重叠时被重复处理；同时沿用 `correlationId` + `operationId` 去重 |

### 5.6 成本说明

| 费用项 | 计费方式 | 估算 |
|---|---|---|
| Storage 存储费 | ~$0.018/GB/月（Cool 层）| 中型客户 Activity Log 约 1–5 GB/月，约 **$0.02–$0.10/月** |
| Blob 读取请求 | $0.004/10,000 次 | 可忽略不计 |
| Diagnostic Settings | **免费** | $0 |

> 相比方案一，方案二**无 Event Hub TU 费**，存储成本极低，是成本最优方案。但**失去了实时性**，且后台需实现轮询逻辑而非长连接消费。

### 5.7 踩坑与注意事项

**① 当前小时的文件在整点后才写完，需处理"未写完文件"**

Storage 文件按整点切分（如 `h=09` 包含 09:00–09:59 的事件），但 Azure 在整点后约 10–30 分钟才将该小时的文件写完。后台轮询时若请求当前小时的文件，可能拿到一个**尚未写完的部分文件**，导致漏采当小时尾部的事件。

推荐处理方式：后台始终只处理**至少已结束 30 分钟的小时**（如当前时间 10:25，则只处理到 `h=09`），不尝试读取当前小时的文件，用延迟换取完整性。

**② `PT1H.json` 文件可能存在追加写入**

对于 Append Blob 模式写入的日志文件，Azure 可能在文件写完之前多次追加数据。若后台在文件未完成时就下载并解析，下次再下载同一文件时会拿到更多数据，产生重复记录。推荐结合第①条的"等待整点 + 30 分钟"策略，确保只在文件写完后才处理。

**③ Resource Logs 路径因资源类型而不同**

不同资源类型的 Resource Logs 存储在不同路径前缀下（如 Key Vault 的 `insights-logs-auditevent`、NSG 流日志的 `insights-logs-networksecuritygroupflowevent` 等），后台不能硬编码路径前缀，需动态拼接或提前枚举客户开启了哪些资源的诊断设置。

**④ 后台 S3 等价物：Storage Blob List 的性能**

Azure Blob Storage 的 `List Blobs` API 支持按路径前缀过滤，性能良好，不存在 AWS S3 的前缀热点限制问题。但路径层级较深（年/月/日/时/分）时，批量枚举性能会下降，建议后台直接按时间戳构造精确路径（`y=2024/m=03/d=15/h=09/m=00/PT1H.json`）而非遍历整个目录树。

> **参考文档（方案二）**：
> - [Archive Azure Monitor logs to storage account](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs#send-to-azure-storage)
> - [Azure Monitor activity log – send to storage](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log)
> - [Azure Storage Blob pricing](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/)

---

## 6. 方案三：方案一 + 方案二（双轨方案）

同时配置 Event Hub（实时消费）和 Storage Account（持久存档），Event Hub 提供准实时的流式采集，Storage Account 提供完整的历史兜底与补偿扫描能力。

这是**可靠性要求最高场景下的推荐配置**，也是与 AWS 方案一（Trail → S3，天然具备持久化）最对等的 Azure 实现。

### 6.1 整体链路

```
                        [Azure ARM 操作发生]
                               ↓
              [Diagnostic Settings（同一诊断设置，双目标）]
                  ↙                           ↘
[Event Hub（实时路径）]              [Storage Account（存档路径）]
  ↓ 2–8 分钟                           ↓ 10–30 分钟（按小时批量）
[后台长连接消费]                    [后台定时轮询（每小时）]
  ↓                                    ↓
[实时告警处理]              [补偿扫描：检测 Event Hub 消息超期后漏采的事件]
                  ↘                   ↙
              [SaaS 采集后台：统一入库（幂等去重）]
```

### 6.2 客户需要配置的内容

Azure 的诊断设置支持**同一个诊断设置同时指定多个目标**（每个诊断设置最多 5 个目标），客户无需创建两套诊断设置。

在方案一 §4.2 的配置基础上，修改第 1 步：在添加诊断设置时，**同时勾选** Event Hub 和 Storage Account 两个目标：

```
诊断设置目标：
  ☑ Stream to an event hub → [选择方案一创建的 Event Hub]
  ☑ Archive to a storage account → [选择方案二创建的 Storage Account]
```

授权方面，SaaS 后台的服务主体需同时具备 §1 中方案三对应的两个 RBAC 角色：
- `Azure Event Hubs Data Receiver`（Event Hub Namespace 级别）
- `Storage Blob Data Reader`（Storage Account 级别）

### 6.3 成本说明

| 费用项 | 估算 |
|---|---|
| Event Hub Standard TU | 约 $11–22/月（同方案一） |
| Storage 存储费 | 约 $0.02–$0.10/月（同方案二） |
| Diagnostic Settings | 免费 |
| **合计** | **约 $11–22/月**，增量主要来自 Event Hub TU |

> 相比方案一，方案三增加的成本极低（Storage 费用几乎可忽略），但换来了完整的日志兜底能力。对于有日志完整性或合规保留要求的客户，强烈推荐选择方案三而非方案一。

**后台补偿扫描逻辑（方案三特有）**：

后台应以低频（如每小时一次）对 Storage Account 做增量路径扫描：
1. 构造当前时间窗口对应的 Blob 路径（如过去 2 小时）
2. 下载对应 `PT1H.json` 文件，解析其中每条记录的 `correlationId` + `operationId`
3. 与已入库的事件 ID 集合对比，找出 Event Hub 路径未消费到的记录
4. 补充入库，确保最终不漏任何事件

---

## 7. 方案对比与选型建议

| 维度 | 方案一（Event Hub） | 方案二（Storage 轮询） | 方案三（双轨） |
|---|---|---|---|
| **延迟** | 2–8 分钟 | 10–90 分钟 | 实时路径 2–8 分钟 |
| **客户配置步骤** | 约 4–5 步 | 约 3–4 步，最简单 | 约 5–6 步（诊断设置双目标 + 双授权） |
| **事件类型覆盖** | Activity Log ✅ · Resource Logs ✅（按需） | 同左 | 同左 |
| **历史日志留存** | ❌ 无（消息超期即删） | ✅ 有（Blob 持久存储） | ✅ 有 |
| **积压导致丢失** | **不可补救**（超期 = 永久丢失；无 DLQ 机制） | **无此风险**（Blob 持久，仅影响延迟） | **可补救**（Storage 兜底） |
| **采集侧过滤** | ❌ 不支持（诊断设置只能按 Category 过滤，无字段级过滤） | ❌ 不支持 | ❌ 不支持 |
| **投递失败重试** | Azure 平台内部重试，无公开 SLA | Azure 平台保证最终一致写入 | 两路径各自保证 |
| **消费失败补救** | ❌ 仅靠 Checkpoint 重放，超期后无法补救 | ✅ Blob 文件持久，随时可补拉 | ✅ Storage 补偿扫描 |
| **幂等消费机制** | `correlationId` + `operationId` 去重 | 已处理文件路径列表去重 | 两者结合 |
| **API 限流/配额** | Event Hub TU 限制；分区数决定最大并发消费数 | Storage API 限流极高（5000 req/s），实践中不触顶 | 同各自方案 |
| **DLQ 机制** | ❌ Event Hub 无原生 DLQ | N/A（文件不会丢失） | ❌（依赖 Storage 兜底） |
| **客户侧月度增量成本** | Event Hub TU 费（低–中，约 $11–22/月） | Storage 费（极低，约 $0.1/月） | 约 $11–22/月 |
| **业界代表厂商** | Splunk、QRadar、ArcSight | 早期/轻量 SIEM、合规存档 | Microsoft Sentinel 等高级 SIEM |

> **与 AWS 的关键差异**：Azure Event Hub **没有 SQS 的死信队列（DLQ）机制**，消息超期后直接永久删除，无任何兜底。这是方案三（双轨）在 Azure 场景下比 AWS 对应方案更重要的原因——Storage Account 是 Azure 场景下唯一可靠的日志兜底手段。

**后台工程建议（针对所有方案）**：

- **第一层·预防积压（方案一/三）**：监控 Event Hub 的 Consumer Lag 指标，超阈值时自动扩容消费 Worker（Worker 数 ≤ Event Hub 分区数）
- **第二层·失败处理（方案一/三）**：后台消费失败时实现指数退避重试，持久化 Checkpoint，重启后从 Checkpoint 继续消费，避免重复处理
- **第三层·最终一致（仅方案三）**：后台每小时对 Storage Account 执行一次增量路径扫描，与已入库事件对比，补拉因 Event Hub 消息超期或消费异常而漏掉的事件，实现日志采集的最终一致性

---

## 8. 多订阅场景：Management Group 统一接入

大中型企业通常通过 Azure Management Group 统一管理多个订阅，并希望将所有订阅的日志集中采集。单订阅逐一配置的方式不具备可扩展性，本章说明企业环境下的统一接入方案。

### 8.1 Management Group 统一接入与单订阅接入的关键差异

| 对比项 | 单订阅接入 | Management Group 统一接入 |
|---|---|---|
| **诊断设置配置方式** | 各订阅手动单独配置 | 在 Management Group 级别通过 **Azure Policy（DeployIfNotExists）** 自动部署 |
| **覆盖范围** | 仅本订阅 | **自动覆盖 MG 下所有现有及新增订阅** |
| **Event Hub 归属** | 各订阅各自创建独立的 Event Hub | 可集中路由到一个**共享 Event Hub Namespace**（需跨订阅授权） |
| **日志区分方式** | 天然按订阅隔离 | 混合在同一 Event Hub，需按每条记录的 `subscriptionId` 字段路由 |
| **新增订阅覆盖** | 需手动重新配置 | **Policy 自动触发，约 30 分钟内完成部署** |
| **SaaS 后台接入点** | 每个订阅各自的 Event Hub / Storage | 仅接入**集中 Event Hub Namespace**，即可采集所有订阅日志 |

### 8.2 接入架构

```
[订阅 A] ──┐
[订阅 B] ──┤── Management Group Policy（DeployIfNotExists）
[订阅 C] ──┘    ↓ 自动为每个订阅部署诊断设置
                ↓
        [集中 Event Hub Namespace]（位于日志归档订阅）
            ├── activity-logs（Event Hub）← 所有订阅的 Activity Log
            └── resource-logs（Event Hub）← 所有订阅的 Resource Logs（按需）
                      ↓ 后台 AMQP 长连接消费
             [SaaS 采集后台：按 subscriptionId 路由告警]
```

**说明**：集中 Event Hub Namespace 通常建议放在专用的**日志归档订阅**（Log Archive Subscription）中，与业务订阅隔离。SaaS 后台只需接入这一个 Namespace，即可采集整个 Management Group 下所有订阅的日志。

### 8.3 客户需要额外配置的内容

**前提**：客户已通过 Azure Management Group 管理多个订阅，并在日志归档订阅中创建了集中 Event Hub Namespace（操作同 §4.2 前置步骤）。

---

**第 1 步：在 Management Group 级别创建 Azure Policy**

使用 `DeployIfNotExists` 类型的 Policy，自动为 MG 下每个订阅部署诊断设置，将 Activity Log 路由到集中 Event Hub Namespace。

> 微软官方提供了现成的 Policy 定义（[`Deploy Diagnostic Settings for Activity Log to Event Hub`](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#monitoring)），客户可直接在 Azure Portal 的 Policy 页面搜索并分配，无需自行编写 Policy JSON。

控制台操作（约 10 分钟）：

1. Azure Portal → Policy → 分配 → 选择作用域（Management Group）
2. 搜索内置 Policy：`Deploy Diagnostic Settings for Activity Log to Event Hub`
3. 填写 Event Hub Namespace、授权规则、Event Hub 名称参数
4. 勾选「创建修复任务（Remediation Task）」，自动为已有订阅补充配置

---

**第 2 步：配置跨订阅授权**

集中 Event Hub Namespace 位于日志归档订阅，各业务订阅的诊断设置需要写入权限，SaaS 后台需要读取权限：

```bash
# 授予 Azure Monitor 服务向集中 Event Hub 写入的权限（由 Policy 自动配置）
# 此步骤通常由 Policy 的 Managed Identity 自动处理

# 授予 SaaS 服务主体读取集中 Event Hub 的权限
az role assignment create \
  --assignee {saas-service-principal-object-id} \
  --role "Azure Event Hubs Data Receiver" \
  --scope /subscriptions/{log-archive-sub-id}/resourceGroups/{rg}/providers/Microsoft.EventHub/namespaces/{central-namespace}

# 若使用多订阅场景的 Reader 权限（用于枚举订阅列表）
az role assignment create \
  --assignee {saas-service-principal-object-id} \
  --role "Reader" \
  --scope /providers/Microsoft.Management/managementGroups/{mg-id}
```

---

**客户在 SaaS 平台填写的额外信息**

| 字段 | 说明 |
|---|---|
| Management Group ID | 客户 Azure Management Group 的 ID，后台据此枚举下属所有订阅 |
| 集中 Event Hub Namespace | 所在的日志归档订阅 ID + Namespace 名称 |
| 日志归档订阅 | 存放集中 Event Hub / Storage Account 的专用订阅 ID |

### 8.4 后台采集注意事项

**① 按 `subscriptionId` 字段路由告警**

所有订阅的日志混合在同一个 Event Hub 中，后台在解析每条记录时，必须以 `subscriptionId` 字段作为路由键，确保告警正确归属到对应的客户订阅，而不是混同为集中日志账号的事件。

**② 新增订阅的覆盖延迟**

Policy 存在自动部署延迟（新订阅加入 MG 后，通常约 **30 分钟**内完成诊断设置自动部署）。这段时间内新订阅的 Activity Log 可能无法采集到，属于正常行为，后台应记录每个订阅的"首次日志接收时间"以便排查。

**③ 定期刷新订阅列表**

后台应定期调用 `Azure Management API` 刷新 MG 下的订阅列表（建议每小时一次），确保新加入的订阅能被正确发现，其日志能映射到客户配置的订阅别名，提升告警可读性。

**④ 推荐使用独立的日志归档订阅**

建议向有条件的客户推荐将 Event Hub Namespace 和 Storage Account 放在专用的日志归档订阅（而非管理订阅或业务订阅），原因：

```
[管理订阅（管理面，最敏感）]  ← SaaS 后台不需要直接访问
         ↓ Policy 统一部署诊断设置
[日志归档订阅（专用）]
    ├── 集中 Event Hub Namespace
    └── 集中 Storage Account（方案三兜底）
         ↑ SaaS 后台仅需访问这里
```

这样 SaaS 后台无需对管理订阅持有任何权限，最大程度降低了凭证泄露的爆炸半径。

**⑤ 多诊断设置重复事件问题**

若客户在 Management Group 级别和订阅级别同时配置了诊断设置（例如已有旧的单订阅配置，又新部署了 MG Policy），同一条事件可能被重复推送到 Event Hub。后台应通过 `correlationId` + `operationId` 去重，且在初始接入时建议客户检查并清理重复的诊断设置配置。

> **参考文档**：
> - [Deploy Diagnostic Settings at scale with Azure Policy](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings-policy)
> - [Azure Policy DeployIfNotExists effect](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-deployifnotexists)
> - [Management groups overview](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
> - [Delegated resource management for Azure Monitor](https://learn.microsoft.com/en-us/azure/lighthouse/how-to/monitor-at-scale)

---

## 9. 参考文档

| 类别 | 文档名称 | 链接 |
|---|---|---|
| **采集方案** | Diagnostic settings in Azure Monitor | https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings |
| | Create diagnostic settings | https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/create-diagnostic-settings |
| | Stream Azure monitoring data to Event Hub | https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/stream-monitoring-data-event-hubs |
| | Enable Diagnostic Settings at scale（Azure Policy）| https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostic-settings-policy |
| **Event Hub** | Azure Event Hubs features | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features |
| | Azure Event Hubs quotas and limits | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas |
| | Azure Event Hubs scalability guide | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability |
| | Azure Event Hubs Auto-inflate | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-auto-inflate |
| **Storage** | Archive Azure Monitor logs to storage | https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/resource-logs |
| | Azure Blob Storage pricing | https://azure.microsoft.com/en-us/pricing/details/storage/blobs/ |
| **日志 Schema** | Azure Activity Log event schema | https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log-schema |
| | AzureActivity table reference（Log Analytics） | https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity |
| **身份授权** | Workload Identity Federation | https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation |
| | Managed identities for Azure resources | https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview |
| **多订阅场景** | Azure Policy DeployIfNotExists | https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effect-deployifnotexists |
| | Management groups overview | https://learn.microsoft.com/en-us/azure/governance/management-groups/overview |
| **厂商参考** | Splunk Add-on for Microsoft Cloud Services | https://docs.splunk.com/Documentation/AddOns/released/MSCloudServices/About |
| | Microsoft Sentinel – Azure Activity connector | https://learn.microsoft.com/en-us/azure/sentinel/data-connectors/azure-activity |
| | Azure Monitor partner integrations | https://learn.microsoft.com/en-us/azure/azure-monitor/partners |

---

*本文档基于 Azure 云日志采集方案 v1.0 全面更新，对齐 AWS 云日志采集方案的文档结构与内容深度。所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有产品或定价变更，以微软官方最新文档为准。*