# Microsoft Entra ID 日志采集流程

> **文档背景**：本文档从《Microsoft Entra ID 云日志调研报告》第 3 章独立提取，专门针对日志采集方案的详细设计和实现指南。
> **适用读者**：后台开发 / 采集架构师

---

## 概述

本章节介绍 Microsoft Entra ID 日志的采集方案设计，包括授权模型、Graph API 轮询、诊断设置流式采集，以及各方案的实现要点和踩坑经验。

---

## 0. 章节背景

本文档从**客户（运维/开发人员）视角**出发，说明"我们采集的方案是什么，客户需要在自己的云账号里做什么，才能让我们的采集后台拿到日志"。

**采集后台的工作模式**：我们的后台程序部署在独立 SaaS 平台上，不会进入客户的云环境。当客户接入一个或多个云账号后，后台基于账号维度拉取日志。

**关键设计约束**：
- 优先设计**消息队列监听（Push-Pull）**的方案，客户在自己的账号将日志推送到某个队列（或类似的服务）中，通知部署在 SaaS 平台上订阅的后台程序进行日志采集。
- 设计方案前需要对云厂商提供的日志服务进行能力分析，例如是否支持流式拉取，是否提供 API 级别的筛选，是否支持重试等影响方案选型的关键能力。
- 至少制定两套采集方案进行对比分析，制定的方案最好在业界内有使用参考。
- 设计的方案需要说明客户侧关注内容：如何配置，成本等。
- 方案本身需考虑是否支持筛选、日志持久化、日志规模、延迟、限流、日志丢失这些基本问题，方案本身具备可扩展、高可用、可复用等特性。

---

## 1. 授权模型

与 Azure Activity Log 报告中的授权模型相同，Entra 日志采集同样使用服务主体授权。

| 模型 | 方式 | 推荐程度 |
|---|---|---|
| **模型 A：多租户应用 + Workload Identity Federation** | 客户在 Entra Admin Center 对我们的多租户应用执行 Admin Consent，后台使用 Federated Credential 获取临时令牌 | ✅ 强烈推荐 |
| **模型 B：单租户应用 + Client Secret** | 客户在 Entra 注册专用应用，提供 Client Secret | ⚠️ 次选 |

### Graph API 采集所需的 API 权限

Application 权限，需 Admin Consent：

| 权限 | 类型 | 用途 |
|---|---|---|
| `AuditLog.Read.All` | Application | 读取 SignInLogs 和 AuditLogs（必需） |
| `Directory.Read.All` | Application | 解析日志中的用户/组 Object ID 为可读名称（可选，用于富化） |

> **注意**：`AuditLog.Read.All` 是高权限 API 权限，授予后可读取租户内**所有用户的登录记录和目录变更历史**。在向客户申请此权限时，需要明确说明使用目的，部分合规敏感的客户可能需要额外的审批流程。

> **参考文档**：[Permissions required for Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-log-permissions)

---

## 2. 采集方案总览

针对四类日志，采集路径分为两大类，必须分别处理：

| 日志类别 | 可用采集路径 | 当前实现状态 | 说明 |
|---|---|---|---|
| **SignInLogs**（交互式） | Graph API / 诊断设置 | ✅ **已采集**（Graph API v1.0） | `/auditLogs/signIns` 正式端点，仅返回交互式登录 |
| **NonInteractiveUserSignInLogs** | Graph API（beta）/ 诊断设置 | ⚠️ **待确认是否漏采** | v1.0 端点默认不返回非交互式登录；需要额外在 beta 端点加 `signInEventTypes` 过滤条件才能拉取，若代码未处理则存在漏采 |
| **AuditLogs** | Graph API / 诊断设置 | ✅ **已采集**（Graph API v1.0） | `/auditLogs/directoryAudits` 正式端点 |
| **MicrosoftGraphActivityLogs** | **仅诊断设置** | ❌ **当前实现不可采集** | Graph API 不提供此日志的查询端点，是 Graph API 方案的固有边界 |

**结论**：当前实现确认覆盖 SignInLogs 和 AuditLogs；NonInteractiveUserSignInLogs 存在漏采风险，需代码侧确认；MicrosoftGraphActivityLogs 是 Graph API 方案无法绕过的固有盲区，需配套诊断设置方案才能覆盖。

---

## 3. 方案一（主力）：Graph API 定时轮询

### 3.1 整体链路

```
[后台定时任务（每 N 分钟触发）]
      ↓
[使用服务主体获取 Access Token（有效期 1 小时）]
      ↓
[调用 Graph API，$filter 指定时间窗口（上次拉取时间 → 当前时间）]
      ↓ 返回最多 1000 条/页（若有 @odata.nextLink，继续翻页）
[后台解析 JSON，写入 SIEM / 数据库]
      ↓
[更新"上次成功拉取时间戳"，供下次轮询使用]
```

端到端延迟：轮询间隔（可配）+ API 响应时间（秒级–数十秒）+ 处理入库时间 = **约 5–15 分钟**（取决于轮询频率和当前限流状态）。

### 3.2 核心 API 端点

| 日志类别 | API 端点 | 关键查询参数 |
|---|---|---|
| SignInLogs（交互式） | `GET /v1.0/auditLogs/signIns` | `$filter=createdDateTime ge {timestamp}`，`$top=1000`，`$orderby=createdDateTime asc` |
| NonInteractiveUserSignInLogs | `GET /beta/auditLogs/signIns` | `$filter=createdDateTime ge {timestamp} and signInEventTypes/any(t: t eq 'nonInteractiveUser')`，`$top=1000` |
| AuditLogs | `GET /v1.0/auditLogs/directoryAudits` | `$filter=activityDateTime ge {timestamp}`，`$top=1000`，`$orderby=activityDateTime asc` |

### 3.3 请求示例

**SignInLogs 增量拉取**：

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

### 3.4 返回格式

三类日志均以 JSON 格式返回，结构如下：

| 字段位置 | SignInLogs | AuditLogs |
|---|---|---|
| 事件时间 | `createdDateTime` | `activityDateTime` |
| 唯一标识 | `id` | `id` |
| 操作者 | `userPrincipalName` / `userId` | `initiatedBy.user.userPrincipalName` |
| 操作类型 | `appDisplayName`（哪个应用触发登录） | `activityDisplayName`（如 "Add member to role"） |
| 结果 | `status.errorCode`（0=成功） | `result`（"success" / "failure"） |

### 3.5 关键字段

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

### 3.6 所需权限配置

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

> ⚠️ Admin Consent 必须由 **Global Administrator** 或 **Privileged Role Administrator** 执行，普通用户无法完成此步骤。这是 Entra 日志采集中**客户侧最高的权限门槛**，需要提前与客户确认操作权限。

### 3.7 安全厂商使用策略与踩坑经验

#### 踩坑 ① 时间窗口设计——避免大跨度查询，防止 504 超时

`/auditLogs/signIns` 在查询时间跨度较大时（通常超过 5 天），响应时间会从毫秒级急剧增长至数十秒，甚至返回 504 Gateway Timeout。这不是偶发现象，而是 Graph API 的设计行为（查询大时间段需要扫描更多数据）。

**推荐策略**：
- 正常运行时：每次查询窗口 ≤ **30 分钟**（当前时间 - 30 分钟），轮询间隔 5 分钟
- 历史追溯时（初次接入后追溯近 30 天）：以 **24 小时**为单位，逐天串行查询，不并发

#### 踩坑 ② $skiptoken 过期——处理翻页中断

翻页过程中如果遭遇限流（429），等待 `Retry-After` 时间后继续翻页，但若等待时间过长导致 `$skiptoken` 过期，会返回 `400 Skip Token is null`。此时**不能从失败的 nextLink 继续**，必须从该轮询窗口的起始时间重新发起新查询。

后台实现建议：对每次查询维护"当前已处理的最新 `createdDateTime`"（从已成功处理的记录中取最大值），出现 skiptoken 过期时从该时间戳重新开始，而非从整个窗口起点重来，以减少重复拉取量。

#### 踩坑 ③ 严格限流下的积压风险

Identity and Access Reports 专项限制 **5 requests / 10 seconds**。对于日志量大的租户（如中大型企业），一个完整的时间窗口查询可能需要数十次翻页，在限流下完成一次完整拉取的时间可能超过轮询间隔，导致积压。

积压后的影响：后台追不上新增日志的速度，历史日志越积越多，最终接近 30 天保留期边界，若不能在保留期内追上，**对应日志将永久丢失**（因为平台内置保留期到期后自动删除）。

**应对策略**：
- 监控每轮次的实际处理耗时，若连续超过轮询间隔的 80%，触发告警
- 对不同日志类别分优先级：SignInLogs > AuditLogs > NonInteractiveUserSignInLogs（因为后者量最大）
- 极端情况下临时降低轮询频率，牺牲实时性换取稳定性

#### 踩坑 ④ NonInteractiveUserSignInLogs 的 beta API 稳定性风险

非交互式登录只能通过 beta 端点获取，而 beta API 的字段结构可能在无通知的情况下变更。需要在监控系统中设置针对 beta API 响应结构变化的告警，并定期（建议每季度）检查微软官方文档是否有正式版端点更新。

#### 踩坑 ⑤ 从 AuditLogs 中识别服务代调

当某些目录变更由 Azure 服务自动触发（而非用户手动操作）时，`initiatedBy.user` 为空，`initiatedBy.app.displayName` 显示为服务名（如 `Microsoft Azure AD`、`Privileged Identity Management`）。研判引擎需要对这类服务代调操作区分处理：若目标是告警人工操作引发的变更，应过滤掉 `initiatedBy.user` 为空的记录；若目标是全量审计，则需要保留并单独建模。

> **参考文档**：
> - Graph API SignIns 参考：[List signIns](https://learn.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0)
> - Graph API AuditLogs 参考：[List directoryAudits](https://learn.microsoft.com/en-us/graph/api/directoryaudit-list?view=graph-rest-1.0)
> - 分析活动日志最佳实践：[How to analyze activity logs with Microsoft Graph](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-analyze-activity-logs-with-microsoft-graph)
> - Graph API 限流总览：[Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)
> - Identity Reports 专项限流：[Microsoft Graph service-specific throttling limits – Identity and Access Reports](https://learn.microsoft.com/en-us/graph/throttling-limits)

---

## 4. 方案二（补充）：诊断设置 → Event Hub

此方案是 MicrosoftGraphActivityLogs 的**唯一可用采集路径**，同时可作为三类 Graph API 日志的补充或替代（更低延迟，但需要客户配置 Azure 资源）。

### 4.1 与 Azure Activity Log 诊断设置方案的差异

- **配置入口不同**：Entra 诊断设置在 **Entra Admin Center → Monitor → Diagnostic Settings**，而非 Azure Portal 的 Monitor 页面
- **所需权限不同**：修改 Entra 诊断设置需要 **Security Administrator** 角色（而非 Azure 订阅的 Owner）
- **路径结构不同**：Entra 日志的 Storage 路径包含 `tenantId`，而非 `subscriptionId`

### 4.2 客户需要配置的内容

（约 5 分钟）

1. Entra Admin Center → **Monitor** → **Diagnostic settings** → 「添加诊断设置」
2. 日志类别至少勾选：
   - `AuditLogs`
   - `SignInLogs`
   - `NonInteractiveUserSignInLogs`
   - `MicrosoftGraphActivityLogs`（**此项必须在此处配置，Graph API 无法采集**）
3. 目标：「Stream to an event hub」→ 选择客户 Azure 订阅内的 Event Hub
4. 保存

### 4.3 后台消费方式

与 Azure Activity Log Event Hub 采集相同，使用 AMQP/Kafka 长连接消费，以 `records[].time` 为时间基准。

### 4.4 客户侧新增成本

- Event Hub Standard TU：约 $11/月/TU（Entra 日志量通常需要 1–2 TU）
- 诊断设置本身：免费

> **参考文档**：
> - [Stream Entra logs to an event hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub)
> - [MicrosoftGraphActivityLogs overview](https://learn.microsoft.com/en-us/graph/microsoft-graph-activity-logs-overview)

---

## 5. 方案对比与选型建议

| 维度 | 方案一（Graph API 轮询） | 方案二（诊断设置→Event Hub） |
|---|---|---|
| **MicrosoftGraphActivityLogs** | ❌ **完全不支持** | ✅ 支持 |
| **客户配置复杂度** | ⭐（低，仅需授予 API 权限） | ⭐⭐（中，需建 Event Hub + 配置诊断设置） |
| **客户是否需要 Azure 订阅** | **不需要**（仅需 Entra 租户） | **需要**（Event Hub 是 Azure 资源） |
| **端到端延迟** | 5–15 分钟（受限流影响） | 2–8 分钟 |
| **NonInteractiveSignInLogs** | 需用 beta 端点（有兼容性风险） | ✅ 正式支持 |
| **数据完整性保障** | 依赖后台的时间戳管理和重试逻辑，有丢失风险（见 §3.7 踩坑③） | Event Hub 消息超期后永久丢失（需配合 Storage 兜底） |
| **限流风险** | **高**：5 requests / 10 seconds 极易触顶 | 无限流（诊断设置由 Azure 托管推送） |
| **实现成本（后台）** | 较高（需处理分页、限流、skiptoken 过期、beta API 兼容） | 较低（复用 Azure Activity Log 的 Event Hub 消费逻辑） |

### 5.1 选型决策逻辑

```
客户是否有 Azure 订阅（可创建 Event Hub）？
    ├─ 是 → 推荐方案一 + 方案二并行
    │         方案一：SignInLogs + AuditLogs（Graph API，低配置门槛，覆盖无 Azure 订阅的客户）
    │         方案二：MicrosoftGraphActivityLogs（诊断设置，唯一途径）
    └─ 否 → 只能使用方案一
              无法采集 MicrosoftGraphActivityLogs，需在产品侧向客户说明此能力边界
```

### 5.2 后台工程建议（针对方案一）

- **限流管理**：实现指数退避重试，严格遵守 `Retry-After` 响应头
- **时间戳持久化**：每次成功入库后立即持久化"已处理的最新 `createdDateTime`"，进程崩溃重启后可从此处续拉
- **Beta API 监控**：对 beta 端点的响应结构变化设置 Schema 校验，出现未知字段或字段缺失时告警
- **查询窗口控制**：单次查询时间窗口 ≤ 30 分钟，历史追溯以 24 小时为单位串行执行

---

*本文档所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有 API 或产品变更，以微软官方最新文档为准。*