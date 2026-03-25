# Microsoft Entra ID 日志采集方案

> **文档状态**：从《Microsoft Entra ID 云日志调研报告》中独立拆分 · 版本 2.0
> **调研时间**：2026-03
> **作者**：云安全专家调研
> **适用读者**：产品侧 / 后台研发团队 / 接入实施团队
> **核心聚焦**：Entra 日志采集路径分析、授权模型、两套方案对比与工程最佳实践

---

## 目录

0. [业务背景和设计约束](#0-业务背景和设计约束)
1. [授权模型：后台如何访问客户租户](#1-授权模型后台如何访问客户租户)
2. [Entra 日志服务能力分析](#2-entra-日志服务能力分析)
   - 2.1 [两种采集路径的能力对比](#21-两种采集路径的能力对比)
   - 2.2 [日志类型与覆盖范围](#22-日志类型与覆盖范围)
   - 2.3 [Graph API 查询能力边界](#23-graph-api-查询能力边界)
3. [采集方案总览与选型建议](#3-采集方案总览与选型建议)
4. [方案一（主力）：Graph API 定时轮询](#4-方案一主力graph-api-定时轮询)
   - 4.1 [整体链路](#41-整体链路)
   - 4.2 [核心 API 端点](#42-核心-api-端点)
   - 4.3 [请求示例与返回格式](#43-请求示例与返回格式)
   - 4.4 [关键字段](#44-关键字段)
   - 4.5 [客户需要配置的内容](#45-客户需要配置的内容)
   - 4.6 [日志规模估算](#46-日志规模估算)
   - 4.7 [重试与可靠性](#47-重试与可靠性)
   - 4.8 [踩坑与注意事项](#48-踩坑与注意事项)
5. [方案二（补充）：诊断设置 → Event Hub](#5-方案二补充诊断设置--event-hub)
   - 5.1 [与方案一的核心区别](#51-与方案一的核心区别)
   - 5.2 [整体链路](#52-整体链路)
   - 5.3 [客户需要配置的内容](#53-客户需要配置的内容)
   - 5.4 [消息格式](#54-消息格式)
   - 5.5 [日志规模估算](#55-日志规模估算)
   - 5.6 [重试与可靠性](#56-重试与可靠性)
   - 5.7 [成本说明](#57-成本说明)
   - 5.8 [踩坑与注意事项](#58-踩坑与注意事项)
6. [方案对比与选型建议](#6-方案对比与选型建议)
7. [多租户场景接入](#7-多租户场景接入)
   - 7.1 [多租户场景的类型](#71-多租户场景的类型)
   - 7.2 [与单租户接入的关键差异](#72-与单租户接入的关键差异)
   - 7.3 [推荐接入架构](#73-推荐接入架构)
   - 7.4 [后台采集注意事项](#74-后台采集注意事项)

---

## 0. 业务背景和设计约束

本文档从**客户（运维/开发人员）视角**出发，说明「我们采集的方案是什么，客户需要在自己的云账号里做什么，才能让我们的采集后台拿到日志」。

**采集后台的工作模式**：我们的后台程序部署在独立 SaaS 平台上，不会进入客户的云环境。当客户接入一个或多个云账号后，后台基于账号维度拉取日志。

**关键设计约束**：
- 优先设计**消息队列监听（Push-Pull）**的方案，客户在自己的账号将日志推送到某个队列（或类似的服务）中，通知部署在 SaaS 平台上订阅的后台程序进行日志采集。
- 设计方案前需要对云厂商提供的日志服务进行能力分析，例如是否支持流式拉取，是否提供 API 级别的筛选，是否支持重试等影响方案选型的关键能力。
- 至少制定两套采集方案进行对比分析，制定的方案最好在业界内有使用参考。
- 设计的方案需要说明客户侧关注内容：如何配置，成本等。
- 方案本身需考虑是否支持筛选、日志持久化、日志规模、延迟、限流、日志丢失这些基本问题，方案本身具备可扩展、高可用、可复用等特性。

**Entra 特有的背景约束**：与 AWS CloudTrail 不同，Entra 日志采集面临两条**根本不同**的技术路径——Graph API 轮询（拉取模式）和诊断设置 → Event Hub（推送模式）。两者不是同一服务体系内的变体，而是完全不同的协议和授权体系，且覆盖的日志范围存在固有差异（MicrosoftGraphActivityLogs 只能通过诊断设置采集）。此外，Graph API 路径对客户的 Azure 订阅无依赖，但存在严格的 30 天保留期上限和硬性 API 限流约束，是 Entra 方案选型的核心矛盾所在。

---

## 1. 授权模型：后台如何访问客户租户

在客户接入任何采集方案之前，必须先完成对我们 SaaS 平台的**访问授权**。我们支持以下两种模型：

### 模型 A：多租户应用 + Workload Identity Federation（推荐）

这是微软官方推荐的 SaaS 集成授权方式。我们在 SaaS 平台侧注册一个**多租户应用**，客户在自己的 Entra 租户中对该应用执行 **Admin Consent**，授予所需的 Application 权限。后台使用 **Federated Credential**（Workload Identity Federation）而非静态密钥来获取临时访问令牌。

**安全优势**：
- 后台无需持有任何静态凭证，完全消除密钥泄露风险
- 临时令牌默认有效期 1 小时，泄露后自动失效
- 客户可随时在 Entra Admin Center 吊销对我们应用的授权，立即切断访问
- 客户无需在自己的租户内注册任何应用，操作门槛低

**后台获取 Token 流程**：

```
[客户在 Entra Admin Center 执行 Admin Consent]
      ↓ 授权我们的多租户应用访问该租户
[SaaS 后台使用 Federated Credential 向 Microsoft Identity Platform 换取临时令牌]
      ↓ 返回 Access Token（有效期 1 小时）
[SaaS 后台持有临时令牌]
      ↓ 调用 Graph API / 消费 Event Hub
[客户租户的 Entra 日志]
```

**客户需要做的事（一次性）**：

1. Entra Admin Center → 企业应用程序（Enterprise Applications）→ 搜索我们平台的多租户应用
2. 点击「权限（Permissions）」→「授予管理员同意（Grant admin consent for {租户名}）」
3. 在弹出的权限确认页面审查并确认授予所需权限

> **注意**：Admin Consent 必须由 **Global Administrator** 或 **Privileged Role Administrator** 执行，普通用户无法完成此步骤。这是 Entra 日志采集中客户侧最高的权限门槛，需要提前与客户确认操作权限。

### 模型 B：单租户应用 + Client Secret（次选）

客户在自己的 Entra 租户中注册一个专用应用，生成 Client Secret 并提供给我们。后台使用该 Secret 调用 Graph API。

**适用场景**：客户有安全合规限制无法执行多租户 Admin Consent，或组织策略要求所有第三方集成使用租户内自建应用。

**必须向客户明确告知的安全注意事项**：
- Client Secret 是**长期有效的静态凭证**，一旦泄露不会自动失效，需立即手动撤销
- 应为采集专用，**权限必须最小化**（仅授予读取日志所需权限，禁止任何写权限）
- 建议定期（至少每 90 天）轮换 Client Secret
- 建议客户在 Entra 的「条件访问（Conditional Access）」中为该应用配置访问限制策略

---

## 2. Entra 日志服务能力分析

本章对 Entra 两种日志采集路径的核心能力进行分析，作为后续方案选型的决策依据。

### 2.1 两种采集路径的能力对比

Entra 日志采集存在两条根本不同的技术路径，能力差异显著：

| 能力维度 | Graph API 轮询 | 诊断设置 → Event Hub |
|---|---|---|
| **采集模式** | 拉取（Pull）：后台定时主动查询 | 推送（Push）：Azure 托管推送到 Event Hub |
| **端到端延迟** | 5–15 分钟（受限流约束） | 2–8 分钟（接近实时） |
| **是否持久化** | ❌ 无；依赖 Graph API 平台内置保留期 | ⚠️ Event Hub 消息最长保留 7 天（可配），超期永久丢失 |
| **是否支持流式消费** | ❌ 不支持；每次轮询为独立 HTTP 请求 | ✅ 支持；AMQP/Kafka 长连接持续消费 |
| **是否支持源侧过滤** | ⚠️ 有限；`$filter` 支持时间范围和部分字段过滤，但不支持复杂组合 | ❌ 诊断设置只能按日志类别（Category）粒度开关，不支持字段级过滤 |
| **日志保留期** | **30 天（硬上限）**，P2 许可证可延长至 90 天 | 由 Event Hub 消息保留期决定，最长 7 天（Standard）/ 90 天（Premium） |
| **客户是否需要 Azure 订阅** | **不需要**（仅需 Entra 租户） | **需要**（Event Hub 是 Azure 资源） |
| **限流风险** | **高**：5 req/10s（Identity Reports 专项限额） | 无限流（由 Azure 托管推送，不消耗客户 API 配额） |

**核心结论**：两种路径能力互补——Graph API 门槛低（无需 Azure 订阅），但有严格的保留期上限和限流约束；Event Hub 路径延迟低、无限流，但需要客户有 Azure 订阅，且存在消息超期后**不可补救**的丢失风险。选型决策应以客户是否有 Azure 订阅作为第一决策点。

### 2.2 日志类型与覆盖范围

Entra 记录四类关键日志，与两种采集路径的集成覆盖存在固有差异：

| 日志类型 | 说明 | Graph API 轮询 | 诊断设置 → Event Hub |
|---|---|---|---|
| **SignInLogs**（交互式登录） | 用户通过浏览器/应用发起的交互式登录记录 | ✅ v1.0 正式端点 `/auditLogs/signIns` | ✅ |
| **NonInteractiveUserSignInLogs** | 服务、守护进程、后台 Token 刷新等非交互式登录 | ⚠️ **仅 beta 端点**，需额外 `signInEventTypes` 过滤条件；beta API 存在字段变更风险 | ✅ 正式支持 |
| **AuditLogs** | 目录变更操作：用户/组/角色/应用的创建、修改、删除等 | ✅ v1.0 正式端点 `/auditLogs/directoryAudits` | ✅ |
| **MicrosoftGraphActivityLogs** | 对 Graph API 本身发起的调用记录，用于检测异常 API 访问、数据渗漏尝试 | ❌ **Graph API 不提供此日志的查询端点**，是 Graph API 方案的固有盲区 | ✅ **唯一可用采集路径** |

> **关键约束**：MicrosoftGraphActivityLogs 是**唯一只能通过诊断设置采集**的日志类型，这是方案二（Event Hub）存在的核心工程理由。若客户有 Azure 订阅，建议方案一和方案二并行部署——方案一采集 SignInLogs + AuditLogs，方案二专门覆盖 MicrosoftGraphActivityLogs。
>
> **NonInteractiveUserSignInLogs 漏采风险**：若代码未处理 beta 端点的 `signInEventTypes` 过滤条件，此类日志将存在漏采。服务账号、自动化脚本的登录行为均属于非交互式登录，漏采会在安全审计中产生重大盲区（如攻击者使用服务账号 Token 的横向移动行为将完全不可见）。

### 2.3 Graph API 查询能力边界

Graph API 作为采集通道存在以下固有限制，是理解方案一局限性的关键：

- **30 天保留期硬上限**：Graph API 的 SignInLogs 和 AuditLogs 平台内最多保留 30 天（Entra P2 许可证可延长至 90 天）。超期日志被平台永久删除，**无任何补救途径**。这意味着后台一旦出现连续超过 30 天的采集中断，对应时间段的日志将永久丢失。
- **硬性限流**：Identity and Access Reports 专项限额为 **5 requests / 10 seconds**（约 30 req/分钟）。对于日志量大的租户，单次完整时间窗口拉取可能需要数十次翻页，在限流下完成一轮拉取的时间可能超过轮询间隔，引发积压。
- **不支持流式推送**：Graph API 每次调用是独立的 HTTP 请求，无法建立长连接持续接收新事件，轮询模式是唯一选项。
- **skiptoken 过期风险**：翻页过程中若因限流等待时间过长导致 `$skiptoken` 过期，必须从头重新发起查询（详见 §4.8）。
- **无法采集 MicrosoftGraphActivityLogs**：这是 Graph API 的固有能力边界，无任何绕过方案。

**结论**：Graph API 适合小规模交互式查询或初次接入的低配置门槛场景，但不是高可靠持续采集的理想通道。对于有 Azure 订阅的客户，应将方案二（Event Hub）作为主力路径，方案一作为补充。

---

## 3. 采集方案总览与选型建议

基于准实时（5–15 分钟）的产品定位，我们提供两套采集方案：

| 方案 | 核心链路 | 端到端延迟 | 覆盖范围 | 客户配置复杂度 | 客户侧增量成本 | 推荐场景 |
|---|---|---|---|---|---|---|
| **方案一**：Graph API 定时轮询 | 后台定时 → Graph API → 解析入库 | **5–15 分钟** | SignInLogs ✅ · AuditLogs ✅ · NonInteractiveSignInLogs ⚠️（beta） · MicrosoftGraphActivityLogs ❌ | ⭐（低，仅需授予 API 权限） | $0（无需 Azure 额外资源） | 无 Azure 订阅的客户；快速接入验证 |
| **方案二**：诊断设置 → Event Hub | Entra 诊断设置 → Event Hub → 后台消费 | **2–8 分钟** | 全部四类日志 ✅ | ⭐⭐（中，需建 Event Hub + 配置诊断设置） | Event Hub ~$11–22/月 | 有 Azure 订阅的客户；MicrosoftGraphActivityLogs 覆盖 |

> 两套方案均兼容授权模型 A（多租户应用 + WIF）和模型 B（Client Secret），授权配置见 §1。

**业界使用情况参考**：

| 方案 | 厂商使用案例 | 文档链接 |
|---|---|---|
| **方案一**（Graph API） | **Microsoft Sentinel**：官方 Entra ID 连接器默认使用 Graph API 拉取 SignInLogs 和 AuditLogs，并提供增量拉取和断点续拉机制 | [Connect Microsoft Entra ID to Microsoft Sentinel](https://learn.microsoft.com/en-us/azure/sentinel/connect-azure-active-directory) |
| **方案一**（Graph API） | **Splunk Add-on for Microsoft Cloud Services**：使用 Graph API `/auditLogs/signIns` 和 `/auditLogs/directoryAudits` 端点，文档明确说明了限流处理和增量拉取策略 | [Splunk Add-on for Microsoft Cloud Services](https://splunkbase.splunk.com/app/3110) |
| **方案一**（Graph API） | **Elastic Azure AD 集成**：使用 Graph API 采集 SignInLogs 和 AuditLogs，文档说明了 `$filter` 时间窗口的使用方式 | [Azure AD integration – Elastic](https://www.elastic.co/docs/reference/integrations/azure_ad) |
| **方案二**（Event Hub） | **Splunk Add-on for Microsoft Cloud Services**：同时支持 Event Hub 模式，文档将 Event Hub 方案标注为更低延迟、无限流约束的替代路径 | [Splunk Add-on for Microsoft Cloud Services – Event Hub input](https://splunkbase.splunk.com/app/3110) |
| **方案二**（Event Hub） | **Microsoft Sentinel**：官方文档将 Event Hub 模式作为 MicrosoftGraphActivityLogs 的唯一接入路径，明确说明 Graph API 不提供该日志的查询端点 | [Stream Microsoft Entra logs to Azure Event Hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub) |
| **方案二**（Event Hub） | **Cribl**：官方文档将 Entra 诊断设置 → Event Hub 作为推荐的 Entra 日志采集路径，并提供了完整的 Stream 配置示例 | [Microsoft Azure Event Hub – Cribl Docs](https://docs.cribl.io/stream/sources-azure-event-hub/) |

---

## 4. 方案一（主力）：Graph API 定时轮询

### 4.1 整体链路

```
[后台定时任务（每 N 分钟触发）]
      ↓
[使用服务主体获取 Access Token（有效期 1 小时）]
      ↓
[调用 Graph API，$filter 指定时间窗口（上次拉取时间 → 当前时间）]
      ↓ 返回最多 1000 条/页（若有 @odata.nextLink，继续翻页）
[后台解析 JSON，写入 SIEM / 数据库]
      ↓
[更新「上次成功拉取时间戳」，供下次轮询使用]
```

端到端延迟：轮询间隔（可配）+ API 响应时间（秒级–数十秒）+ 处理入库时间 = **约 5–15 分钟**（取决于轮询频率和当前限流状态）。

### 4.2 核心 API 端点

| 日志类别 | API 端点 | 关键查询参数 |
|---|---|---|
| SignInLogs（交互式） | `GET /v1.0/auditLogs/signIns` | `$filter=createdDateTime ge {timestamp}`，`$top=1000`，`$orderby=createdDateTime asc` |
| NonInteractiveUserSignInLogs | `GET /beta/auditLogs/signIns` | `$filter=createdDateTime ge {timestamp} and signInEventTypes/any(t: t eq 'nonInteractiveUser')`，`$top=1000` |
| AuditLogs | `GET /v1.0/auditLogs/directoryAudits` | `$filter=activityDateTime ge {timestamp}`，`$top=1000`，`$orderby=activityDateTime asc` |

### 4.3 请求示例与返回格式

**SignInLogs 增量拉取请求**：

```http
GET https://graph.microsoft.com/v1.0/auditLogs/signIns
    ?$filter=createdDateTime ge 2024-03-15T10:00:00Z
    &$top=1000
    &$orderby=createdDateTime asc

Authorization: Bearer {access_token}
```

**响应结构**：

```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#auditLogs/signIns",
  "@odata.nextLink": "https://graph.microsoft.com/v1.0/auditLogs/signIns?$skiptoken=xxx",
  "value": [
    {
      "id": "66ea54eb-6301-4ee5-be62-ff5a759b0100",
      "createdDateTime": "2024-03-15T10:01:23Z",
      "userDisplayName": "Alice Smith",
      "userPrincipalName": "alice@contoso.com",
      "userId": "d0d28bf6-....",
      "appId": "de8bc8b5-...",
      "appDisplayName": "Azure Portal",
      "ipAddress": "203.0.113.45",
      "status": { "errorCode": 0, "failureReason": null },
      "clientAppUsed": "Browser",
      "conditionalAccessStatus": "success",
      "isInteractive": true,
      "riskDetail": "none",
      "riskLevelAggregated": "none",
      "authenticationRequirement": "multiFactorAuthentication",
      "mfaDetail": { "authMethod": "Phone call", "authDetail": "Phone call to +1 XXX-XXX-XXXX" },
      "location": { "city": "Shanghai", "state": "Shanghai", "countryOrRegion": "CN" },
      "deviceDetail": { "deviceId": "", "operatingSystem": "Windows 10", "browser": "Edge 120.0" }
    }
  ]
}
```

> 若响应中存在 `@odata.nextLink`，后台必须循环请求该链接，直至响应中不再包含 `@odata.nextLink`，此时表明本次时间窗口内的数据已全部拉取完毕。

**三类日志的字段位置对比**：

| 字段位置 | SignInLogs | AuditLogs |
|---|---|---|
| 事件时间 | `createdDateTime` | `activityDateTime` |
| 唯一标识 | `id` | `id` |
| 操作者 | `userPrincipalName` / `userId` | `initiatedBy.user.userPrincipalName` |
| 操作类型 | `appDisplayName`（哪个应用触发登录） | `activityDisplayName`（如 "Add member to role"） |
| 结果 | `status.errorCode`（0=成功） | `result`（"success" / "failure"） |

### 4.4 关键字段

**SignInLogs 关键字段**：

| 字段 | 说明 | 研判用途 |
|---|---|---|
| `id` | 登录事件唯一 ID | 去重基准；同时是与 MicrosoftGraphActivityLogs 关联的 `SignInActivityId` |
| `createdDateTime` | 登录事件发生时间（UTC） | **时间基准**，必须以此字段为准 |
| `userPrincipalName` | 登录用户的 UPN（Email 格式） | 身份溯源核心字段 |
| `ipAddress` | 登录来源 IP | 异地登录、Tor/VPN 检测 |
| `status.errorCode` | 登录结果错误码（0=成功） | 密集失败（50126=凭证错误）= 密码喷射信号 |
| `authenticationRequirement` | 认证强度要求（`singleFactorAuthentication` / `multiFactorAuthentication`） | 未使用 MFA 的成功登录是高风险信号 |
| `conditionalAccessStatus` | 条件访问策略的执行结果（`success`/`failure`/`notApplied`） | `notApplied` 说明条件访问策略未覆盖此场景 |
| `riskLevelAggregated` | Identity Protection 聚合风险等级（`none`/`low`/`medium`/`high`） | 高风险登录需立即告警 |
| `appId` / `appDisplayName` | 触发登录的应用 ID / 名称 | 非预期应用（不在已知列表中）是告警信号 |
| `location.countryOrRegion` | 登录来源国家 | 跨国异常登录检测 |
| `isInteractive` | 是否为交互式登录 | 区分人工操作与自动化操作 |
| `uniqueTokenIdentifier` | 令牌唯一标识符 | **与 MicrosoftGraphActivityLogs 的 `SignInActivityId` 关联的核心字段** |

**AuditLogs 关键字段**：

| 字段 | 说明 | 研判用途 |
|---|---|---|
| `id` | 审计事件唯一 ID | 去重基准 |
| `activityDateTime` | 操作发生时间（UTC） | **时间基准** |
| `activityDisplayName` | 操作名称（如 `Add member to role`、`Consent to application`） | **告警规则的核心过滤字段** |
| `category` | 操作类别（`UserManagement`、`ApplicationManagement`、`RoleManagement`、`Policy` 等） | 快速分类高危操作域 |
| `result` | 操作结果（`success`/`failure`） | 成功的高危操作是主要告警对象 |
| `initiatedBy.user.userPrincipalName` | 操作发起者的 UPN | 身份溯源 |
| `initiatedBy.app.displayName` | 若由应用发起，显示应用名称 | 识别自动化操作与服务代调 |
| `targetResources[].displayName` | 操作的目标对象名称（被修改的用户/组/应用） | 关联具体受影响的实体 |
| `targetResources[].modifiedProperties` | 变更前后的属性值（`oldValue` / `newValue`） | 还原具体变更内容（如被分配了哪个角色） |
| `correlationId` | 关联 ID | 将多个相关审计操作关联为同一事务 |

### 4.5 客户需要配置的内容

客户在 Entra Admin Center 为我们的服务主体授予以下 Application 权限（需要 Global Administrator 或 Privileged Role Administrator 执行 Admin Consent）：

| 权限 | 类型 | 用途 | 是否必需 |
|---|---|---|---|
| `AuditLog.Read.All` | Application | 读取 SignInLogs 和 AuditLogs | **必需** |
| `Directory.Read.All` | Application | 解析日志中的 Object ID 为可读的用户/组名称 | 可选（用于日志富化） |

**配置步骤（客户操作，约 5 分钟）**：

1. Entra Admin Center → 应用注册（App registrations）→ 搜索我们平台的多租户应用
2. 进入「企业应用程序（Enterprise Applications）」→ 找到对应应用
3. 点击「权限（Permissions）」→「授予管理员同意（Grant admin consent）」
4. 在弹出的权限确认页面确认授予 `AuditLog.Read.All` 权限

> ⚠️ `AuditLog.Read.All` 是高权限 API 权限，授予后可读取租户内**所有用户的登录记录和目录变更历史**。在向客户申请此权限时，需要明确说明使用目的，部分合规敏感的客户可能需要额外的审批流程。

**配置完成后，客户在 SaaS 平台填写的接入信息**：

| 字段 | 内容（模型 A：多租户应用） | 内容（模型 B：Client Secret） |
|---|---|---|
| 租户 ID（Tenant ID） | 客户 Entra 租户的 Directory ID | 同左 |
| 应用 ID（Client ID） | 我们 SaaS 平台多租户应用的 App ID | 客户在自己租户注册的应用 App ID |
| Client Secret | —— | 客户生成的 Secret 值 |

### 4.6 日志规模估算

日志规模直接影响后台 Worker 并发设计和限流预算规划，以下为参考基准：

- **SignInLogs**：中型企业（~1000 名用户）每天约产生 **5,000–20,000 条**登录事件（含成功/失败），5 分钟时间窗口约 20–70 条，单次查询通常 **1 次请求**即可完成（不触发翻页）
- **AuditLogs**：目录变更相对低频，每天约 **500–2,000 条**，5 分钟时间窗口约 2–7 条
- **NonInteractiveUserSignInLogs**：量级通常是交互式的 3–10 倍（取决于服务账号数量），每天 **15,000–200,000 条**不等，翻页概率较高，在大型企业中单次查询可能需要翻 2–5 页
- **API 请求消耗**：3 类日志每 5 分钟轮询一次，正常情况下约消耗 **5–10 次 API 请求/5 分钟**（含翻页），在 5 req/10s 的限额内有足够余量；但非交互式登录量大的租户在翻页密集时可能触顶，需要监控

### 4.7 重试与可靠性

| 能力 | 说明 |
|---|---|
| **API 层重试** | 遇 429 响应时，严格遵守 `Retry-After` 响应头中的等待时间，使用指数退避，最大等待 30s |
| **时间戳持久化** | 每次成功入库后立即持久化「已处理的最新 `createdDateTime`/`activityDateTime`」，进程崩溃重启后从此时间戳续拉，不依赖进程内存 |
| **skiptoken 过期补救** | 翻页中断后，从当前窗口内「已成功处理的最新时间戳」重新发起查询，不从整个窗口起点重来，减少重复拉取量 |
| **幂等消费** | 基于事件 `id` 字段去重，防止 skiptoken 重试导致的重复入库 |
| **30 天保留期告警** | 监控后台积压深度，当「当前轮询进度」落后「当前时间」超过 **25 天**时触发高优先级告警；30 天到期后无任何补救途径（不同于 AWS 方案一的 S3 兜底） |

### 4.8 踩坑与注意事项

#### 踩坑 ① 时间窗口设计——避免大跨度查询，防止 504 超时

`/auditLogs/signIns` 在查询时间跨度较大时（通常超过 5 天），响应时间会从毫秒级急剧增长至数十秒，甚至返回 504 Gateway Timeout。这不是偶发现象，而是 Graph API 的设计行为（查询大时间段需要扫描更多数据）。

**推荐策略**：
- 正常运行时：每次查询窗口 ≤ **30 分钟**（当前时间 - 30 分钟），轮询间隔 5 分钟
- 历史追溯时（初次接入后追溯近 30 天）：以 **24 小时**为单位，逐天串行查询，不并发

#### 踩坑 ② $skiptoken 过期——处理翻页中断

翻页过程中如果遭遇限流（429），等待 `Retry-After` 时间后继续翻页，但若等待时间过长导致 `$skiptoken` 过期，会返回 `400 Skip Token is null`。此时**不能从失败的 nextLink 继续**，必须从该时间戳重新发起新查询。

后台实现建议：对每次查询维护「当前已处理的最新 `createdDateTime`」（从已成功处理的记录中取最大值），出现 skiptoken 过期时从该时间戳重新开始，而非从整个窗口起点重来，以减少重复拉取量。

#### 踩坑 ③ 严格限流下的积压风险——30 天保留期是硬截止线

Identity and Access Reports 专项限制 **5 requests / 10 seconds**。对于日志量大的租户，一个完整的时间窗口查询可能需要数十次翻页，在限流下完成一次完整拉取的时间可能超过轮询间隔，导致积压。

**积压的后果比 AWS 严重得多**：AWS 方案一的 SQS 消息超期只丢通知，S3 原始文件仍在可补拉。而 Graph API 方案没有任何等效的持久化兜底——一旦后台积压超过 30 天保留期，对应日志**永久不可挽回**。

**应对策略**：
- 监控每轮次的实际处理耗时，若连续超过轮询间隔的 80%，触发告警
- 对不同日志类别分优先级：SignInLogs > AuditLogs > NonInteractiveUserSignInLogs（因为后者量最大）
- 极端情况下临时降低轮询频率，牺牲实时性换取稳定性
- 积压严重时（落后超过 20 天），考虑临时切换至方案二（Event Hub），在积压追回后恢复方案一

#### 踩坑 ④ NonInteractiveUserSignInLogs 的 beta API 稳定性风险

非交互式登录只能通过 beta 端点获取，而 beta API 的字段结构可能在无通知的情况下变更。需要在监控系统中设置针对 beta API 响应结构变化的告警，并定期（建议每季度）检查微软官方文档是否有正式版端点更新。

#### 踩坑 ⑤ 从 AuditLogs 中识别服务代调

当某些目录变更由 Azure 服务自动触发（而非用户手动操作）时，`initiatedBy.user` 为空，`initiatedBy.app.displayName` 显示为服务名（如 `Microsoft Azure AD`、`Privileged Identity Management`）。研判引擎需要对这类服务代调操作区分处理：若目标是告警人工操作引发的变更，应过滤掉 `initiatedBy.user` 为空的记录；若目标是全量审计，则需要保留并单独建模。

> **参考文档（方案一）**：
> - Graph API SignIns 参考：[List signIns](https://learn.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0)
> - Graph API AuditLogs 参考：[List directoryAudits](https://learn.microsoft.com/en-us/graph/api/directoryaudit-list?view=graph-rest-1.0)
> - 分析活动日志最佳实践：[How to analyze activity logs with Microsoft Graph](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-analyze-activity-logs-with-microsoft-graph)
> - Graph API 限流总览：[Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)
> - Identity Reports 专项限流：[Microsoft Graph service-specific throttling limits – Identity and Access Reports](https://learn.microsoft.com/en-us/graph/throttling-limits)

---

## 5. 方案二（补充）：诊断设置 → Event Hub

此方案是 MicrosoftGraphActivityLogs 的**唯一可用采集路径**，同时可作为其他三类日志的更低延迟替代方案（无限流约束，但需要客户有 Azure 订阅）。

### 5.1 与方案一的核心区别

| 对比项 | 方案一（Graph API 轮询） | 方案二（诊断设置 → Event Hub） |
|---|---|---|
| **MicrosoftGraphActivityLogs** | ❌ 完全不支持 | ✅ **唯一采集路径** |
| **采集模式** | 拉取（后台主动轮询） | 推送（Azure 托管推送） |
| **限流风险** | 高（5 req/10s） | **无**（由 Azure 托管推送，不消耗 API 配额） |
| **客户是否需要 Azure 订阅** | **不需要** | **需要**（Event Hub 是 Azure 资源） |
| **日志保留** | 依赖 Graph API 平台内置 30 天保留期 | **Event Hub 消息最长保留 7 天**（Standard），消费后即删除 |
| **积压导致丢失** | **不可补救**：积压超 30 天日志永久丢失 | **不可补救**：消息超 7 天（或配置的保留期）永久丢失，无任何补救途径 |

> **配置入口差异**：Entra 诊断设置在 **Entra Admin Center → Monitor → Diagnostic Settings**，而非 Azure Portal 的 Monitor 页面。所需权限也不同：修改 Entra 诊断设置需要 **Security Administrator** 角色，而非 Azure 订阅的 Owner。

### 5.2 整体链路

```
[Entra 日志事件发生（登录/目录变更/Graph API 调用）]
      ↓ 约 2–8 分钟（Azure 托管推送，近实时）
[Azure Event Hub：消息分区存储]
      ↓ 后台 AMQP/Kafka 长连接消费
[SaaS 采集后台：解析 records[] 数组 → 入库]
```

端到端延迟：Entra 诊断设置推送延迟（约 2–8 分钟）+ Event Hub 消费（秒级）= **约 2–8 分钟**。

### 5.3 客户需要配置的内容

**前提**：客户有 Azure 订阅，并已在订阅内创建 Event Hub Namespace 和 Event Hub 实例。

**Event Hub 参数建议**：

| 参数 | 建议值 | 说明 |
|---|---|---|
| Namespace SKU | Standard | Premium 支持更长消息保留期（90 天），如有合规需求可选 |
| 分区数（Partition Count） | 4–8 | 影响最大并发消费者数量；Entra 日志量通常不需要超过 8 分区 |
| 消息保留期 | 7 天（Standard 上限） | 给后台足够的消费窗口；方案二无 S3 兜底，保留期越长越安全 |
| TU（Throughput Units） | 1–2 | 中型企业 Entra 日志量通常 1 TU 足够；高峰期可临时扩容 |

**配置诊断设置（客户操作，约 5 分钟）**：

1. Entra Admin Center → **Monitor** → **Diagnostic settings** → 「添加诊断设置」
2. 日志类别至少勾选：
   - `AuditLogs`
   - `SignInLogs`
   - `NonInteractiveUserSignInLogs`
   - `MicrosoftGraphActivityLogs`（**此项必须在此处配置，Graph API 无法采集**）
3. 目标：「Stream to an event hub」→ 选择客户 Azure 订阅内的 Event Hub Namespace 和 Event Hub 实例
4. 保存

**授权 SaaS 后台消费 Event Hub**：

后台服务主体需要在 Event Hub Namespace 或实例级别分配 **`Azure Event Hubs Data Receiver`** 角色（Azure RBAC），才能读取消息。客户在 Azure Portal 中操作：

- 进入 Event Hub Namespace → 「访问控制（IAM）」→「添加角色分配」
- 角色：`Azure Event Hubs Data Receiver`
- 成员：我们 SaaS 平台的服务主体（Enterprise Application）

> 若使用消费者组（Consumer Group），建议为 SaaS 后台创建专用消费者组（如 `saas-collector`），避免与其他消费者（如 Sentinel）产生冲突（见 §5.8 踩坑 ①）。

**客户在 SaaS 平台填写的接入信息**：

| 字段 | 内容 |
|---|---|
| Event Hub Namespace | 如 `contoso-entra-logs.servicebus.windows.net` |
| Event Hub 名称 | 如 `insights-logs-signinlogs` |
| 消费者组名称 | 如 `saas-collector`（使用专用消费者组，不用 `$Default`） |
| 租户 ID | 客户 Entra 租户的 Directory ID |

### 5.4 消息格式

Event Hub 消息体为 JSON 格式，Entra 诊断设置推送的消息以 `records[]` 数组包裹：

```json
{
  "records": [
    {
      "time": "2024-03-15T09:15:00.0000000Z",
      "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "operationName": "Sign-in activity",
      "category": "SignInLogs",
      "resultType": "0",
      "resultDescription": "Successfully signed in",
      "properties": {
        "id": "66ea54eb-6301-4ee5-be62-ff5a759b0100",
        "createdDateTime": "2024-03-15T09:15:00Z",
        "userDisplayName": "Alice Smith",
        "userPrincipalName": "alice@contoso.com",
        "ipAddress": "203.0.113.45",
        "status": { "errorCode": 0 },
        "conditionalAccessStatus": "success",
        "riskLevelAggregated": "none",
        "authenticationRequirement": "multiFactorAuthentication",
        "location": { "city": "Shanghai", "countryOrRegion": "CN" }
      }
    }
  ]
}
```

**与方案一（Graph API）的消息格式差异**：

| 字段 | 方案一（Graph API 响应） | 方案二（Event Hub 消息） |
|---|---|---|
| 外层结构 | `value[]` 数组，直接包含事件对象 | `records[]` 数组，事件字段嵌套在 `properties` 下 |
| 时间字段 | `createdDateTime`（直接位于根层级） | `time`（外层）+ `properties.createdDateTime`（内层），以 `properties.createdDateTime` 为准 |
| 租户标识 | 通过 API 调用的服务主体上下文隐式确定 | `tenantId` 字段明确包含在消息中（多租户场景下非常重要） |
| 日志类型区分 | 通过 API 端点区分（`/signIns` vs `/directoryAudits`） | `category` 字段区分（`SignInLogs`/`AuditLogs`/`MicrosoftGraphActivityLogs` 等） |

### 5.5 日志规模估算

- 中型企业（~1000 用户）全部四类日志推送到 Event Hub，每天约 **20,000–250,000 条消息**（主要由 NonInteractiveSignInLogs 和 MicrosoftGraphActivityLogs 贡献）
- 每条消息约 **1–5 KB**，中型企业每日约 **50–500 MB** 数据入 Event Hub
- 1 个 TU 支持 1 MB/s 的吞吐量，中型企业峰值约 0.1–0.5 MB/s，1 TU 通常足够；大型企业（5000+ 用户）建议配置 2–4 TU

### 5.6 重试与可靠性

| 能力 | 说明 |
|---|---|
| **诊断设置投递重试** | Azure 内置重试，诊断设置投递失败时会自动重试，不会静默丢弃（但极端情况下仍可能有少量丢失） |
| **消费侧重试** | Event Hub 消费者基于分区偏移量（Offset）进行消费，消费失败可回溯到上次 Checkpoint 位置重新消费 |
| **消费失败补救** | ❌ Event Hub 消息超过保留期后永久丢失，**无任何补救途径**；Storage Checkpoint 是防止重复消费的唯一手段，但不能补救超期丢失 |
| **幂等消费** | 后台基于事件 `properties.id` 字段去重，防止 Checkpoint 回溯导致的重复处理 |

### 5.7 成本说明

| 费用项 | 计费方式 | 估算（中型企业，1000 人） |
|---|---|---|
| Entra 诊断设置 | **免费** | $0 |
| Event Hub Standard（1 TU） | ~$11/月/TU | **约 $11/月** |
| Event Hub 数据出流量 | $0.028/GB（超出 TU 包含流量后） | 中型企业通常在 TU 包含量内，约 $0–3/月 |
| Event Hub Premium（可选，90 天保留期） | ~$50/月/PU | 有长期保留合规需求时考虑 |
| **Entra P2 许可证（如需 90 天 Graph API 保留期）** | 按用户计费，约 $9/用户/月 | **仅方案一适用**；方案二不受此限制 |

**两方案成本对比**：

| 成本项 | 方案一（Graph API） | 方案二（Event Hub） |
|---|---|---|
| Azure 额外资源费 | $0 | ~$11–22/月 |
| 是否受 Entra P1/P2 许可证影响 | ⚠️ P1 保留 30 天，P2 保留 90 天 | 不受影响 |
| 总体增量成本 | **$0**（但受保留期限制） | **约 $11–22/月**（无保留期限制） |

### 5.8 踩坑与注意事项

#### 踩坑 ① 消费者组冲突

Event Hub 的每个消费者组（Consumer Group）维护独立的偏移量（Offset）状态。若 SaaS 后台与客户的其他消费者（如 Microsoft Sentinel、自建数据管道）共用 `$Default` 消费者组，将导致两个消费者竞争同一批消息，互相跳过对方已消费的分区。

**解决方案**：始终为 SaaS 后台创建专用消费者组（如 `saas-collector`），与其他消费者隔离。

#### 踩坑 ② 分区偏移量管理——Storage Checkpoint 必须正确配置

Event Hub 消费者依赖 **Azure Blob Storage Checkpoint** 来持久化每个分区的消费进度。若 Checkpoint 配置错误（如 Storage 连接字符串错误、权限不足），消费者每次重启后将**从最旧可用消息重新消费**，产生大量重复数据；或**从最新消息开始消费**，丢失重启期间的所有消息。

**验证方式**：在首次接入后，手动重启后台消费进程，确认消费从上次 Checkpoint 位置恢复，而非从头开始。

#### 踩坑 ③ 诊断设置传输延迟抖动

Entra 诊断设置的投递延迟通常为 2–8 分钟，但在高负载情况下可能延迟至 15–30 分钟，且无 SLA 保证。后台在解析事件时应以消息内的 `properties.createdDateTime` 作为事件时间基准，而非 Event Hub 消息的 `EnqueuedTime`（入队时间），避免因延迟抖动导致事件时间序列错乱。

#### 踩坑 ④ Event Hub TU 限流——超出 TU 上限时消息被丢弃

Event Hub 在入流量超过 TU 上限时（1 TU = 1 MB/s 入站），会直接丢弃超出部分的消息，且**不会重试**。这与 SQS 的「消息积压不丢失」行为完全不同。建议在 Azure Monitor 中对 Event Hub 的 `ThrottledRequests` 指标设置告警，在流量超限前主动扩容 TU。

#### 踩坑 ⑤ MicrosoftGraphActivityLogs 量级预估——避免低估 TU 需求

MicrosoftGraphActivityLogs 记录**所有对 Graph API 的调用**，包括 Sentinel、Splunk 等安全工具以及自动化脚本的 API 调用。在有大量集成工具的企业中，此类日志的量级可能远超 SignInLogs + AuditLogs 的总和。建议在正式接入前先启用诊断设置观察 1–2 天的实际流量，再决定 TU 配置。

> **参考文档（方案二）**：
> - [Stream Entra logs to an event hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub)
> - [MicrosoftGraphActivityLogs overview](https://learn.microsoft.com/en-us/graph/microsoft-graph-activity-logs-overview)
> - [Azure Event Hubs quotas and limits](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas)
> - [Balancing partition load across multiple instances of your application](https://learn.microsoft.com/en-us/azure/event-hubs/event-processor-balance-partition-load)

---

## 6. 方案对比与选型建议

| 维度 | 方案一（Graph API 轮询） | 方案二（诊断设置 → Event Hub） |
|---|---|---|
| **延迟** | 5–15 分钟 | 2–8 分钟（无严格 SLA） |
| **MicrosoftGraphActivityLogs** | ❌ 完全不支持 | ✅ **唯一采集路径** |
| **NonInteractiveSignInLogs** | ⚠️ beta 端点（有兼容性风险） | ✅ 正式支持 |
| **客户配置复杂度** | ⭐（低，仅需 Admin Consent） | ⭐⭐（中，需建 Event Hub + 诊断设置） |
| **客户是否需要 Azure 订阅** | **不需要** | **需要** |
| **限流风险** | **高**（5 req/10s 硬限额） | **无**（Azure 托管推送） |
| **日志保留** | 依赖平台 30 天保留期（P2 可延至 90 天） | Event Hub 最长 7 天（Standard）/ 90 天（Premium） |
| **积压导致丢失** | **不可补救**：积压超 30 天永久丢失 | **不可补救**：消息超保留期永久丢失，无任何补救途径 |
| **消费失败补救** | 依赖时间戳持久化，在保留期内可重拉 | 依赖 Checkpoint 回溯，超保留期不可补救 |
| **DLQ / Checkpoint 必要性** | 时间戳持久化：**必须** | Storage Checkpoint：**必须** |
| **客户侧月度增量成本** | $0 | ~$11–22/月 |
| **推荐客户类型** | 无 Azure 订阅客户；快速接入验证 | 有 Azure 订阅客户；需要 GraphActivityLogs |

**选型决策树**：

```
客户是否有 Azure 订阅（可创建 Event Hub）？
    ├─ 是 → 推荐方案一 + 方案二并行
    │         方案一：SignInLogs + AuditLogs（Graph API，低配置门槛）
    │         方案二：MicrosoftGraphActivityLogs（诊断设置，唯一途径）
    │         说明：并行不冲突，两方案采集各自负责的日志类型
    └─ 否 → 只能使用方案一
              无法采集 MicrosoftGraphActivityLogs，需在产品侧向客户说明此能力边界

（边界情况）客户有 Azure 订阅，但担心 Event Hub 成本：
    └─ 可仅为 MicrosoftGraphActivityLogs 单独配置方案二
       其他三类日志继续走方案一的 Graph API 拉取
```

**后台工程三层防御建议**：

方案一和方案二都面临「积压后不可补救」的根本风险，但成因不同（保留期 vs 消息超期）。后台应从三层防御：

- **第一层·预防积压（方案一）**：严格遵守 `Retry-After`，实现令牌桶限流，监控每轮次耗时，超过轮询间隔的 80% 立即告警并触发自动降频
- **第一层·预防丢失（方案二）**：对 Event Hub 的 `ThrottledRequests` 指标设置告警，在 TU 满载前自动扩容；消费者组务必独立，不与其他服务共用
- **第二层·失败兜底（通用）**：方案一必须持久化时间戳，崩溃重启后从断点续拉；方案二必须配置 Storage Checkpoint，崩溃重启后从 Checkpoint 位置恢复
- **第三层·积压预警（方案一特有）**：监控「当前采集进度」与「当前时间」的差值，超过 **25 天**（距 30 天保留期还剩 5 天缓冲）触发 P0 告警，强制人工介入。方案二无等效机制，但应对 DLQ 深度（消费失败消息数量）设置阈值告警。

---

## 7. 多租户场景接入

大型企业客户可能管理多个独立的 Entra 租户（如不同子公司、不同地区各自维护独立租户），需要 SaaS 后台同时采集多个租户的日志。本章说明多租户场景的架构差异和接入要点。

### 7.1 多租户场景的类型

**场景一：企业客户管理多个 Entra 租户（本章重点）**

企业因历史原因或组织架构（子公司独立运营、并购整合、不同合规域隔离）维护多个 Entra 租户，需要在 SaaS 平台中统一接入并采集所有租户的日志。

这是本章的核心场景——每个租户需要独立完成授权、独立配置采集路径，后台需要处理多租户的并发采集和日志路由。

**场景二：SaaS 平台作为多租户应用跨客户采集（授权模型的固有能力）**

我们的 SaaS 平台本身就是一个多租户应用，每个客户（每个 Entra 租户）单独执行 Admin Consent 授权。这是授权模型 A 的内置设计，不需要额外的架构支持，每个客户租户彼此隔离。

### 7.2 与单租户接入的关键差异

| 对比项 | 单租户接入 | 多租户接入（同一客户的多个租户） |
|---|---|---|
| **授权步骤** | 在 1 个租户执行 Admin Consent | 需要在**每个租户分别执行** Admin Consent（每个租户的 Global Admin 操作） |
| **诊断设置配置** | 在 1 个租户的 Entra Admin Center 配置 | 需要在**每个租户分别配置**诊断设置 |
| **Event Hub 归属** | 单一 Event Hub | 见 §7.3：独立 Event Hub vs 共享 SaaS 侧 Event Hub |
| **日志中的租户标识** | `tenantId` 固定为单一值 | 日志中的 `tenantId` 各不相同，后台**必须按 tenantId 路由** |
| **Graph API 限流边界** | 每租户独立计算 5 req/10s 限额 | 多个租户各自独立计算限额，但后台并发调用多租户时需整体调度，防止单租户被饿死 |
| **Token 管理** | 单一 Token（1 小时有效期） | 每个租户独立获取 Token，需要 Token 池管理 |

### 7.3 推荐接入架构

多租户场景下，Event Hub 的归属有两种架构选择：

**架构 A：每个租户独立 Event Hub（数据隔离型）**

```
[客户租户 A] ── 诊断设置 ──→ [Event Hub A（在客户订阅 A 中）]
                                        ↓
                               SaaS 后台消费 → 入库（租户 A 分区）

[客户租户 B] ── 诊断设置 ──→ [Event Hub B（在客户订阅 B 中）]
                                        ↓
                               SaaS 后台消费 → 入库（租户 B 分区）
```

- **优势**：租户数据完全隔离，故障影响范围小，权限边界清晰
- **劣势**：Event Hub 数量随租户数线性增长，后台需管理多个 Event Hub 连接；每个租户的 SaaS 后台需单独配置消费者组
- **适用场景**：客户各子公司有独立 Azure 订阅，或有强数据隔离合规要求

**架构 B：多租户共享 SaaS 侧集中 Event Hub（成本优化型）**

```
[客户租户 A] ── 诊断设置 ──→ ┐
[客户租户 B] ── 诊断设置 ──→ ├─ [SaaS 侧集中 Event Hub（在 SaaS 订阅中）]
[客户租户 C] ── 诊断设置 ──→ ┘                ↓
                               SaaS 后台消费 → 按 tenantId 路由 → 入库
```

- **优势**：Event Hub 数量固定，运维成本低；后台消费逻辑统一
- **劣势**：所有租户日志混合在同一 Event Hub，需要在消费侧严格按 `tenantId` 路由；单个 Event Hub 故障影响所有租户；需要客户将诊断设置指向 SaaS 侧 Event Hub（需向客户提供 Event Hub 连接信息，客户可能有合规顾虑）
- **适用场景**：客户各租户无独立 Azure 订阅，或客户明确接受共享 Event Hub 架构

> **建议**：优先推荐架构 A（每租户独立 Event Hub），数据主权清晰，客户接受度更高。架构 B 在客户数量多且每租户日志量小的场景下有明显成本优势，但需要提前与客户就数据共存问题充分沟通。

### 7.4 后台采集注意事项

**① tenantId 维度的日志路由**

多租户场景下，所有日志（无论来自哪个租户）都会流入后台的处理管道。后台在解析每条日志时，必须提取 `tenantId` 字段并据此路由到对应租户的数据分区，避免日志归属混乱。架构 B（共享 Event Hub）尤其需要注意这一点——`tenantId` 是区分租户日志的唯一可靠标识。

**② Graph API 多租户并发调度**

当后台同时对多个租户执行 Graph API 轮询时，每个租户的 5 req/10s 限额是独立计算的，但后台的并发 HTTP 请求会共享同一个出口网络和 CPU 资源。建议后台对租户级轮询任务实现**公平调度**：

- 使用优先级队列，避免某些大型租户（日志量大、翻页多）长期占用后台资源
- 对单租户的最大并发请求数设置上限（如每租户最多 2 个并发请求）
- 监控各租户的采集延迟（当前进度与实时的差值），若某租户持续落后触发独立告警

**③ Admin Consent 的操作协调**

多个租户需要各自的 Global Admin 独立完成 Admin Consent，流程较复杂，尤其是跨国或跨子公司时 Global Admin 身份确认需要时间。建议 SaaS 平台提供批量接入引导，支持生成租户专属的 Admin Consent URL，客户各租户 Global Admin 点击链接即可一键完成授权，降低接入摩擦。

**④ Token 池管理**

多租户场景下，后台需要为每个租户独立维护 Access Token（每个租户的 Token 相互独立，不可复用）。建议实现 Token 池：

```
[Token 池]
  ├── 租户 A Token（有效期 1 小时，提前 5 分钟刷新）
  ├── 租户 B Token（有效期 1 小时，提前 5 分钟刷新）
  └── 租户 C Token（...）
```

Token 应在到期前 5–10 分钟提前刷新，避免在轮询执行中途 Token 过期导致请求失败。Token 刷新失败时应有独立的告警机制（可能意味着客户撤销了 Admin Consent 授权）。

**⑤ 新租户接入自动化**

当客户在 SaaS 平台添加新的 Entra 租户时，后台需要：
1. 触发新租户的 Admin Consent 引导流程
2. 创建对应的轮询任务（方案一）或消费者配置（方案二）
3. 执行首次历史数据追溯（根据保留期决定追溯窗口，最多 30 天）
4. 将新租户加入监控体系（延迟告警、Token 健康检查等）

建议将上述步骤封装为**租户接入自动化工作流**，确保每个新租户的接入流程一致，避免遗漏配置项。

> **参考文档（多租户场景）**：
> - [Multi-tenant organizations in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/multi-tenant-organizations/overview)
> - [Configure cross-tenant access settings](https://learn.microsoft.com/en-us/entra/external-id/cross-tenant-access-settings-b2b-collaboration)
> - [Microsoft identity platform and OAuth 2.0 client credentials flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)

---

*本文档从《Microsoft Entra ID 云日志调研报告》第 3 章独立拆分。完整的日志分类、能力边界、日志示例及安全厂商经验分析请参阅主报告。*

*所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有产品或功能变更，以微软官方最新文档为准。*