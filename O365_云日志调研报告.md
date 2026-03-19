# Microsoft Entra ID 云日志调研报告

> **文档状态**：版本 1.0  
> **调研时间**：2026-03  
> **作者**：云安全专家调研  
> **适用读者**：规划侧 / 产品侧 / 安全运营团队  
> **核心聚焦**：Microsoft Entra ID 身份平面日志（登录日志、目录审计日志、Graph 活动日志）  
> **范围说明**：本报告覆盖 SignInLogs、NonInteractiveUserSignInLogs、AuditLogs、MicrosoftGraphActivityLogs 四类日志；ServicePrincipalSignInLogs、ManagedIdentitySignInLogs、RiskyUsers/RiskySignIns 不在本次范围内。

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

Microsoft Entra ID 是 Azure 的云身份与访问管理平台（等同于 AWS IAM + AWS Cognito 的组合），其日志体系独立于 Azure Activity Log，专门覆盖**身份平面**：用户登录行为、目录配置变更、以及应用程序对 Graph API 的调用记录。

Entra ID 日志**不通过 Azure Activity Log 传递**，有自己独立的保留策略、采集 API 和诊断设置入口。

### 1.1 本次调研的四类核心日志

| 日志类别 | API 端点（v1.0） | 默认开启 | License 要求 | 核心内容 | 安全意义 |
|---|---|---|---|---|---|
| **SignInLogs**（交互式登录） | `/auditLogs/signIns` | ✅ 是（需配置导出） | Entra P1 / P2 | 用户通过浏览器、客户端应用等交互方式发起的登录事件；含 MFA 状态、条件访问结果、风险等级、来源 IP、设备信息 | **最高**，账户入侵检测的核心数据源 |
| **NonInteractiveUserSignInLogs**（非交互式登录） | `/auditLogs/signIns`（beta，需 `$filter` 指定类型） | ✅ 是（需配置导出） | Entra P1 / P2 | 客户端应用代表用户进行的后台静默登录（如 Outlook 客户端刷新令牌、脚本使用缓存令牌）；用户无感知 | 高，量通常是交互式的 **5–10 倍**；是攻击者利用已有令牌持续访问的重要记录 |
| **AuditLogs**（目录审计日志） | `/auditLogs/directoryAudits` | ✅ 是（需配置导出） | 任意付费 License（无需 P1） | Entra ID 目录层面的配置变更：用户/组/角色管理、应用注册与权限授予、条件访问策略修改等 | **最高**，身份配置变更的全量记录；是攻击者在身份平面持久化的核心检测数据 |
| **MicrosoftGraphActivityLogs**（Graph API 活动日志） | **不支持 Graph API 查询**，只能通过诊断设置导出 | ❌ 否（需手动开启） | Entra P1 / P2 | 所有向该租户 Microsoft Graph API 发起的 HTTP 请求记录（含调用者身份、请求 URI、HTTP 方法、响应状态码） | 高，检测 OAuth 令牌滥用和应用异常行为；是 SignInLogs 的补充——登录日志记录"谁通过了认证"，Graph 活动日志记录"认证后做了什么" |

### 1.2 四类日志的关系与互补性

```
[用户/应用 发起访问]
        ↓
[Entra ID 认证层]
        ├─→ SignInLogs（交互式）        ← 用户主动登录的认证记录
        ├─→ NonInteractiveSignInLogs   ← 应用静默刷新令牌的认证记录
        ↓
[持有合法令牌，调用 Microsoft Graph API]
        ↓
[MicrosoftGraphActivityLogs]          ← 令牌被如何使用的操作记录
        ↓
[Graph API 执行目录操作（如创建用户、分配角色）]
        ↓
[AuditLogs]                          ← 目录对象被变更的结果记录
```

**关键理解**：四类日志覆盖的是同一次攻击的不同阶段——SignInLogs 看到"攻击者用什么凭证登录了"，MicrosoftGraphActivityLogs 看到"登录后调用了哪些 API"，AuditLogs 看到"API 调用产生了哪些配置变更"。单独依赖任何一类日志都会有明显盲区。

### 1.3 保留时长与存储方式

| 存储/访问方式 | 默认保留 | 最长保留 | 支持日志类型 | 计费 |
|---|---|---|---|---|
| **Entra 门户内置查询** | P1/P2：30 天；Free：7 天 | 30 天（固定，不可延长，且**过期后无法追溯**） | SignInLogs、AuditLogs（交互式） | 免费 |
| **诊断设置 → Log Analytics Workspace** | 默认 30 天，可配置至 730 天（Basic 表最长 12 年） | 12 年 | 全部四类（MicrosoftGraphActivityLogs 仅此途径） | 按数据摄入量计费（约 $2.30/GB） |
| **诊断设置 → Storage Account** | 由 Blob 生命周期策略决定 | 理论无限 | 全部四类 | 仅 Storage 存储费 |
| **诊断设置 → Event Hub** | Standard：1–7 天；Premium：最长 90 天 | 90 天 | 全部四类 | Event Hub 吞吐量单元费 |
| **Graph API 查询** | 受平台内置保留期限制 | 最多 30 天（与门户一致） | SignInLogs、AuditLogs（**不含 MicrosoftGraphActivityLogs**） | 免费（但有严格限流） |

> **参考文档**：
> - Entra 日志总览：[Microsoft Entra audit logs API overview](https://learn.microsoft.com/en-us/graph/api/resources/azure-ad-auditlog-overview?view=graph-rest-1.0)
> - Entra 数据保留策略：[Microsoft Entra data retention](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention)
> - MicrosoftGraphActivityLogs 总览：[Access Microsoft Graph activity logs](https://learn.microsoft.com/en-us/graph/microsoft-graph-activity-logs-overview)
> - 诊断设置导出：[Stream Entra logs to Event Hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub)

---

## 2. 能力边界

### 2.1 视角边界

**身份认证平面全覆盖，操作内容不可见，端侧和 Azure 资源操作不在范围内。**

| 可见 | 不可见 |
|---|---|
| 用户交互式登录的完整上下文（IP、设备、MFA 状态、条件访问结果） | 用户登录后在浏览器内的具体操作（网页浏览、表单填写） |
| 应用/服务后台静默刷新令牌的行为 | 令牌被本地缓存后再次使用的过程（令牌离开 Entra 后不可追踪） |
| 对 Entra ID 目录的配置变更（用户创建/删除、角色分配、应用注册） | Azure ARM 资源操作（见 Azure Activity Log 报告） |
| Graph API 的调用记录（URI、方法、响应状态码）——需开启 MicrosoftGraphActivityLogs | Graph API 调用的请求体和响应体内容（邮件内容、文件内容等） |
| 登录失败原因（errorCode） | 攻击者在端侧的本地操作（凭证如何被窃取的过程） |
| 条件访问策略的应用结果 | 租户外部（其他租户）发生的身份操作 |

**跨租户限制**：Entra ID 日志仅覆盖当前租户内的身份操作。如果攻击者通过跨租户联合信任（B2B/B2C）进入，来访用户在本租户内的操作会被记录，但其在源租户的认证行为对本租户不可见。

> **参考文档**：[Azure security logging and auditing](https://learn.microsoft.com/en-us/azure/security/fundamentals/log-audit)

---

### 2.2 内容边界

**只记录操作元数据，不记录操作内容。**

- **SignInLogs**：记录登录事件（时间、IP、设备、应用、结果），但**不记录用户登录后的操作行为**。
- **MicrosoftGraphActivityLogs**：记录 Graph API 调用的 URI、HTTP 方法、响应状态码，但**不记录请求体（Request Body）和响应体（Response Body）**。攻击者通过 Graph API 读取了哪些邮件内容、下载了哪些文件，Graph Activity Log 只能告诉你"他调用了什么 API"，不告诉你"他拿到了什么数据"。
- **AuditLogs**：记录目录对象的变更操作（如"将用户 A 添加到管理员组"），但**不记录操作者的详细意图**；某些高度敏感的操作（如密码重置）在日志中会对具体密码值进行脱敏处理。
- **令牌内容**：Entra 颁发的 JWT 令牌在 SignInLogs 中只记录令牌的元数据（令牌 ID、有效期、声明集），**不记录令牌的完整内容**（payload 中的具体 claims 不完整记录）。

---

### 2.3 日志完整性与可信度

Entra ID 日志的可信度整体高于一般云平台日志，主要原因是：**租户内的普通用户或管理员无法删除已产生的 SignInLogs 和 AuditLogs**，这些日志由微软平台托管，保留期内不可删除。

但存在以下攻击者可操作的空间：

| 操作 | 是否可行 | 所需权限 | 是否留下痕迹 | 防御手段 |
|---|---|---|---|---|
| 删除 Entra 诊断设置（停止日志导出） | **可以** | Security Administrator 或 Global Administrator | ✅ 会被 AuditLogs 记录 | Azure Policy 禁止删除诊断设置；监控 `microsoft.aadiam/diagnosticsettings/delete` 事件 |
| 删除已产生的 SignInLogs / AuditLogs 记录 | **不可以** | —— | —— | 微软平台托管，租户管理员无删除能力 |
| 修改/篡改已产生的日志内容 | **不可以** | —— | —— | 同上 |
| 在保留期到期前阻止日志被访问 | **可以**（通过删除诊断设置阻止导出，但门户仍可查看 30 天内的记录） | Security Administrator | ✅ 操作本身有记录 | 监控诊断设置变更 |
| 关闭 MicrosoftGraphActivityLogs 收集 | **可以**（修改或删除对应诊断设置） | Security Administrator | ✅ 会被 AuditLogs 记录 | 同上 |

**关键结论**：Entra 日志的核心安全风险不是"被篡改"，而是"**诊断设置被删除导致日志停止导出**"，以及"**保留期过短导致历史日志无法追溯**"——前者是主动攻击，后者是默认配置的被动风险。

---

### 2.4 日志保留策略与取证窗口

| 日志类别 | 门户内置保留 | 过期后是否可追溯 | 高危配置场景 |
|---|---|---|---|
| SignInLogs（交互式） | P1/P2：30 天；Free：7 天 | **不可以**，永久丢失 | Free 租户仅 7 天，APT 攻击潜伏期常超过 30 天 |
| NonInteractiveUserSignInLogs | 同上 | **不可以** | 量大但往往是被忽略的第一类日志 |
| AuditLogs | P1/P2：30 天；Free：7 天 | **不可以** | 攻击者注册恶意应用的操作可能在发现时已超过 30 天 |
| MicrosoftGraphActivityLogs | 需通过诊断设置导出，门户无内置留存 | 无内置保留，未配置导出则无任何记录 | 默认完全不收集，是最容易被遗漏的日志类别 |

**APT 场景高危影响**：Storm-0558 攻击案例（2023）中，CSRB 调查报告明确指出，由于日志保留策略的限制，微软没有具体的日志证据来证明攻击者最初如何获取签名密钥——即便是微软自身也受到了日志保留不足的约束。这是 Entra 日志 30 天保留期对溯源分析造成实际影响的最权威佐证。

**最佳实践**：在接入客户账号时，**必须第一步检查 Entra 诊断设置是否已配置导出**。Entra 日志一旦过期，**任何方法都无法找回**，包括联系微软支持。

> **参考文档**：
> - [Microsoft Entra data retention](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention)
> - CSRB 调查报告（Storm-0558）：[CSRB Review of Summer 2023 Microsoft Exchange Online Intrusion (Mar 2024)](https://www.cisa.gov/sites/default/files/2025-03/CSRBReviewOfTheSummer2023MEOIntrusion508.pdf)

---

### 2.5 其他关键限制

**① NonInteractiveUserSignInLogs 只能通过 beta API 端点获取**

Graph API v1.0 的 `/auditLogs/signIns` 端点**只返回交互式登录记录**，不包含非交互式登录。要获取非交互式登录，必须使用 `/beta/auditLogs/signIns` 并加上 `$filter` 条件 `signInEventTypes/any(t: t eq 'nonInteractiveUser')`。

微软官方明确说明 beta 端点**不建议在生产环境使用**（可能随时变更），但目前没有 v1.0 的替代方案。这意味着依赖 beta 端点是现实中的唯一选择，但需要对 API 变更保持监控。

**② MicrosoftGraphActivityLogs 完全不支持 Graph API 查询**

这是最容易被忽视的关键限制。MicrosoftGraphActivityLogs 只能通过 Entra 诊断设置导出到 Log Analytics / Storage / Event Hub，**Graph API 中没有任何端点可以查询这类日志**。如果后台采集实现只依赖 Graph API，该日志类别将**完全不可见**——无论租户是否开启了诊断设置。

**③ SignInLogs 和 AuditLogs 均不支持 Delta Query**

Graph API 的 Delta Query（增量查询）是避免全量轮询的高效机制，但 `auditLogs/signIns` 和 `auditLogs/directoryAudits` 均**不在 Delta Query 的支持列表中**。后台只能通过 `$filter=createdDateTime ge {上次拉取时间}` 的方式做时间窗口轮询，需要自维护"上次成功拉取的时间戳"状态。

**④ 大时间跨度查询会导致 504 超时或性能极度下降**

多个 Microsoft Q&A 帖子记录了一个真实问题：当 `$filter` 的时间范围超过 5 天时，`/auditLogs/signIns` 响应时间会从毫秒级急剧增长到数十秒甚至返回 504 Gateway Timeout。查询跨度越大，响应越慢，这对采用"每次拉取大时间窗口"策略的后台影响极大。推荐每次查询窗口不超过 **24 小时**，并采用时间窗口滚动的方式做批量历史追溯。

**⑤ 限流极为严格：Identity and Access Reports 专属限制 5 requests / 10 seconds**

Entra 日志相关的 Graph API 端点（`/auditLogs/signIns`、`/auditLogs/directoryAudits`）处于 "Identity and Access Reports" 服务类别，官方记录的专项限制为 **5 requests per 10 seconds**，是整个 Microsoft Graph API 中限制最严格的一组端点，比一般 Graph API 端点（通常 2000 requests/20 seconds）低两个数量级。对于中大型租户，这一限流极易成为采集瓶颈。

**⑥ $skiptoken 分页令牌会过期，导致续页失败**

大量数据分页拉取时，Graph API 返回的 `@odata.nextLink` 中包含 `$skiptoken` 参数用于翻页。但该令牌有有效期限制，如果翻页速度过慢（被限流后等待时间过长），**令牌可能在翻页完成前过期**，返回 `400 Skip Token is null` 错误，导致本次拉取不完整。后台必须对此实现重试逻辑：从上次成功的时间戳重新开始新的查询，而非从失败的 nextLink 继续。

> **参考文档**：
> - Graph API 限流总览：[Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)
> - Identity and Access Reports 专项限流：[Microsoft Graph service-specific throttling limits](https://learn.microsoft.com/en-us/graph/throttling-limits)
> - Delta Query 支持资源列表：[Use delta query to track changes in Microsoft Graph data](https://learn.microsoft.com/en-us/graph/delta-query-overview)
> - SignIns API 参考：[List signIns - Microsoft Graph v1.0](https://learn.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0)
> - 分析 Entra 活动日志：[How to analyze activity logs with Microsoft Graph](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-analyze-activity-logs-with-microsoft-graph)

---

## 3. 日志采集流程

### 3.0 章节背景

**采集后台定位**：我们的后台部署在独立 SaaS 平台，不进入客户的 Entra 环境。当客户接入租户后，后台通过主动 Pull 的方式拉取日志。

**当前采集实现的实际范围**：Graph API 定时轮询，确认采集的是 `/auditLogs/signIns`（SignInLogs）和 `/auditLogs/directoryAudits`（AuditLogs）两类日志，用 `$filter` 时间窗口做增量拉取。

**两个待确认项**：
- **NonInteractiveUserSignInLogs**：v1.0 的 `/auditLogs/signIns` 默认只返回交互式登录。若代码未额外加 `signInEventTypes` 过滤条件，该类日志目前未被采集，存在漏采盲区，需确认代码实现。
- **MicrosoftGraphActivityLogs**：Graph API 不支持此日志类型的查询，Graph API 方案下该日志**不可采集**，需由客户配置诊断设置后走独立路径消费。

**准实时目标**：5–15 分钟端到端延迟。

---

### 3.1 授权模型

与 Azure Activity Log 报告中的授权模型相同，Entra 日志采集同样使用服务主体授权。

| 模型 | 方式 | 推荐程度 |
|---|---|---|
| **模型 A：多租户应用 + Workload Identity Federation** | 客户在 Entra Admin Center 对我们的多租户应用执行 Admin Consent，后台使用 Federated Credential 获取临时令牌 | ✅ 强烈推荐 |
| **模型 B：单租户应用 + Client Secret** | 客户在 Entra 注册专用应用，提供 Client Secret | ⚠️ 次选 |

**Graph API 采集所需的 API 权限**（Application 权限，需 Admin Consent）：

| 权限 | 类型 | 用途 |
|---|---|---|
| `AuditLog.Read.All` | Application | 读取 SignInLogs 和 AuditLogs（必需） |
| `Directory.Read.All` | Application | 解析日志中的用户/组 Object ID 为可读名称（可选，用于富化） |

> **注意**：`AuditLog.Read.All` 是高权限 API 权限，授予后可读取租户内**所有用户的登录记录和目录变更历史**。在向客户申请此权限时，需要明确说明使用目的，部分合规敏感的客户可能需要额外的审批流程。

> **参考文档**：[Permissions required for Microsoft Entra audit logs](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-log-permissions)

---

### 3.2 采集方案总览

针对四类日志，采集路径分为两大类，必须分别处理：

| 日志类别 | 可用采集路径 | 当前实现状态 | 说明 |
|---|---|---|---|
| **SignInLogs**（交互式） | Graph API / 诊断设置 | ✅ **已采集**（Graph API v1.0） | `/auditLogs/signIns` 正式端点，仅返回交互式登录 |
| **NonInteractiveUserSignInLogs** | Graph API（beta）/ 诊断设置 | ⚠️ **待确认是否漏采** | v1.0 端点默认不返回非交互式登录；需要额外在 beta 端点加 `signInEventTypes` 过滤条件才能拉取，若代码未处理则存在漏采 |
| **AuditLogs** | Graph API / 诊断设置 | ✅ **已采集**（Graph API v1.0） | `/auditLogs/directoryAudits` 正式端点 |
| **MicrosoftGraphActivityLogs** | **仅诊断设置** | ❌ **当前实现不可采集** | Graph API 不提供此日志的查询端点，是 Graph API 方案的固有边界 |

> **结论**：当前实现确认覆盖 SignInLogs 和 AuditLogs；NonInteractiveUserSignInLogs 存在漏采风险，需代码侧确认；MicrosoftGraphActivityLogs 是 Graph API 方案无法绕过的固有盲区，需配套诊断设置方案才能覆盖。

---

### 3.3 方案一（主力）：Graph API 定时轮询

#### 3.3.1 整体链路

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

#### 3.3.2 核心 API 端点

| 日志类别 | API 端点 | 关键查询参数 |
|---|---|---|
| SignInLogs（交互式） | `GET /v1.0/auditLogs/signIns` | `$filter=createdDateTime ge {timestamp}`，`$top=1000`，`$orderby=createdDateTime asc` |
| NonInteractiveUserSignInLogs | `GET /beta/auditLogs/signIns` | `$filter=createdDateTime ge {timestamp} and signInEventTypes/any(t: t eq 'nonInteractiveUser')`，`$top=1000` |
| AuditLogs | `GET /v1.0/auditLogs/directoryAudits` | `$filter=activityDateTime ge {timestamp}`，`$top=1000`，`$orderby=activityDateTime asc` |

**请求示例（SignInLogs 增量拉取）**：

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

#### 3.3.3 返回格式

三类日志均以 JSON 格式返回，结构如下：

| 字段位置 | SignInLogs | AuditLogs |
|---|---|---|
| 事件时间 | `createdDateTime` | `activityDateTime` |
| 唯一标识 | `id` | `id` |
| 操作者 | `userPrincipalName` / `userId` | `initiatedBy.user.userPrincipalName` |
| 操作类型 | `appDisplayName`（哪个应用触发登录） | `activityDisplayName`（如 "Add member to role"） |
| 结果 | `status.errorCode`（0=成功） | `result`（"success" / "failure"） |

#### 3.3.4 关键字段

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

#### 3.3.5 所需权限

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

#### 3.3.6 安全厂商使用策略与踩坑经验

**踩坑 ① 时间窗口设计——避免大跨度查询，防止 504 超时**

`/auditLogs/signIns` 在查询时间跨度较大时（通常超过 5 天），响应时间会从毫秒级急剧增长至数十秒，甚至返回 504 Gateway Timeout。这不是偶发现象，而是 Graph API 的设计行为（查询大时间段需要扫描更多数据）。

**推荐策略**：
- 正常运行时：每次查询窗口 ≤ **30 分钟**（当前时间 - 30 分钟），轮询间隔 5 分钟
- 历史追溯时（初次接入后追溯近 30 天）：以 **24 小时**为单位，逐天串行查询，不并发

**踩坑 ② $skiptoken 过期——处理翻页中断**

翻页过程中如果遭遇限流（429），等待 `Retry-After` 时间后继续翻页，但若等待时间过长导致 `$skiptoken` 过期，会返回 `400 Skip Token is null`。此时**不能从失败的 nextLink 继续**，必须从该轮询窗口的起始时间重新发起新查询。

后台实现建议：对每次查询维护"当前已处理的最新 `createdDateTime`"（从已成功处理的记录中取最大值），出现 skiptoken 过期时从该时间戳重新开始，而非从整个窗口起点重来，以减少重复拉取量。

**踩坑 ③ 严格限流下的积压风险**

Identity and Access Reports 专项限制 **5 requests / 10 seconds**。对于日志量大的租户（如中大型企业），一个完整的时间窗口查询可能需要数十次翻页，在限流下完成一次完整拉取的时间可能超过轮询间隔，导致积压。

积压后的影响：后台追不上新增日志的速度，历史日志越积越多，最终接近 30 天保留期边界，若不能在保留期内追上，**对应日志将永久丢失**（因为平台内置保留期到期后自动删除）。

**应对策略**：
- 监控每轮次的实际处理耗时，若连续超过轮询间隔的 80%，触发告警
- 对不同日志类别分优先级：SignInLogs > AuditLogs > NonInteractiveUserSignInLogs（因为后者量最大）
- 极端情况下临时降低轮询频率，牺牲实时性换取稳定性

**踩坑 ④ NonInteractiveUserSignInLogs 的 beta API 稳定性风险**

非交互式登录只能通过 beta 端点获取，而 beta API 的字段结构可能在无通知的情况下变更。需要在监控系统中设置针对 beta API 响应结构变化的告警，并定期（建议每季度）检查微软官方文档是否有正式版端点更新。

**踩坑 ⑤ 从 AuditLogs 中识别服务代调**

当某些目录变更由 Azure 服务自动触发（而非用户手动操作）时，`initiatedBy.user` 为空，`initiatedBy.app.displayName` 显示为服务名（如 `Microsoft Azure AD`、`Privileged Identity Management`）。研判引擎需要对这类服务代调操作区分处理：若目标是告警人工操作引发的变更，应过滤掉 `initiatedBy.user` 为空的记录；若目标是全量审计，则需要保留并单独建模。

> **参考文档**：
> - Graph API SignIns 参考：[List signIns](https://learn.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0)
> - Graph API AuditLogs 参考：[List directoryAudits](https://learn.microsoft.com/en-us/graph/api/directoryaudit-list?view=graph-rest-1.0)
> - 分析活动日志最佳实践：[How to analyze activity logs with Microsoft Graph](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-analyze-activity-logs-with-microsoft-graph)
> - Graph API 限流总览：[Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)
> - Identity Reports 专项限流：[Microsoft Graph service-specific throttling limits – Identity and Access Reports](https://learn.microsoft.com/en-us/graph/throttling-limits)

---

### 3.4 方案二（补充）：诊断设置 → Event Hub

此方案是 MicrosoftGraphActivityLogs 的**唯一可用采集路径**，同时可作为三类 Graph API 日志的补充或替代（更低延迟，但需要客户配置 Azure 资源）。

**与 Azure Activity Log 报告中诊断设置方案的差异**：
- 配置入口不同：Entra 诊断设置在 **Entra Admin Center → Monitor → Diagnostic Settings**，而非 Azure Portal 的 Monitor 页面
- 所需权限不同：修改 Entra 诊断设置需要 **Security Administrator** 角色（而非 Azure 订阅的 Owner）
- 路径结构不同：Entra 日志的 Storage 路径包含 `tenantId`，而非 `subscriptionId`

**客户需要配置的内容（约 5 分钟）**：

1. Entra Admin Center → **Monitor** → **Diagnostic settings** → 「添加诊断设置」
2. 日志类别至少勾选：
   - `AuditLogs`
   - `SignInLogs`
   - `NonInteractiveUserSignInLogs`
   - `MicrosoftGraphActivityLogs`（**此项必须在此处配置，Graph API 无法采集**）
3. 目标：「Stream to an event hub」→ 选择客户 Azure 订阅内的 Event Hub
4. 保存

**后台消费方式**：与 Azure Activity Log Event Hub 采集相同，使用 AMQP/Kafka 长连接消费，以 `records[].time` 为时间基准。

**客户侧新增成本**：
- Event Hub Standard TU：约 $11/月/TU（Entra 日志量通常需要 1–2 TU）
- 诊断设置本身：免费

> **参考文档**：
> - [Stream Entra logs to an event hub](https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub)
> - [MicrosoftGraphActivityLogs overview](https://learn.microsoft.com/en-us/graph/microsoft-graph-activity-logs-overview)

---

### 3.5 方案对比与选型建议

| 维度 | 方案一（Graph API 轮询） | 方案二（诊断设置→Event Hub） |
|---|---|---|
| **MicrosoftGraphActivityLogs** | ❌ **完全不支持** | ✅ 支持 |
| **客户配置复杂度** | ⭐（低，仅需授予 API 权限） | ⭐⭐（中，需建 Event Hub + 配置诊断设置） |
| **客户是否需要 Azure 订阅** | **不需要**（仅需 Entra 租户） | **需要**（Event Hub 是 Azure 资源） |
| **端到端延迟** | 5–15 分钟（受限流影响） | 2–8 分钟 |
| **NonInteractiveSignInLogs** | 需用 beta 端点（有兼容性风险） | ✅ 正式支持 |
| **数据完整性保障** | 依赖后台的时间戳管理和重试逻辑，有丢失风险（见 §3.3.6 踩坑③） | Event Hub 消息超期后永久丢失（需配合 Storage 兜底） |
| **限流风险** | **高**：5 requests / 10 seconds 极易触顶 | 无限流（诊断设置由 Azure 托管推送） |
| **实现成本（后台）** | 较高（需处理分页、限流、skiptoken 过期、beta API 兼容） | 较低（复用 Azure Activity Log 的 Event Hub 消费逻辑） |

**选型决策逻辑**：

```
客户是否有 Azure 订阅（可创建 Event Hub）？
    ├─ 是 → 推荐方案一 + 方案二并行
    │         方案一：SignInLogs + AuditLogs（Graph API，低配置门槛，覆盖无 Azure 订阅的客户）
    │         方案二：MicrosoftGraphActivityLogs（诊断设置，唯一途径）
    └─ 否 → 只能使用方案一
              无法采集 MicrosoftGraphActivityLogs，需在产品侧向客户说明此能力边界
```

**后台工程建议（针对方案一）**：
- 限流管理：实现指数退避重试，严格遵守 `Retry-After` 响应头
- 时间戳持久化：每次成功入库后立即持久化"已处理的最新 `createdDateTime`"，进程崩溃重启后可从此处续拉
- Beta API 监控：对 beta 端点的响应结构变化设置 Schema 校验，出现未知字段或字段缺失时告警
- 查询窗口控制：单次查询时间窗口 ≤ 30 分钟，历史追溯以 24 小时为单位串行执行

---

## 4. 日志示例

以下示例均基于微软官方文档中的 Schema 构造，标注触发方式和研判意义。

### 4.1 交互式登录成功（SignInLogs）

**触发方式**：用户通过浏览器登录 Azure Portal 或 Microsoft 365

```json
{
  "id": "66ea54eb-6301-4ee5-be62-ff5a759b0100",
  "createdDateTime": "2024-03-15T08:01:33Z",
  "userDisplayName": "Alice Smith",
  "userPrincipalName": "alice@contoso.com",
  "userId": "d0d28bf6-1234-5678-abcd-ef0123456789",
  "appId": "c44b4083-3bb0-49c1-b47d-974e53cbdf3c",
  "appDisplayName": "Azure Portal",
  "ipAddress": "203.0.113.45",
  "isInteractive": true,
  "clientAppUsed": "Browser",
  "userAgent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Edge/120.0",
  "status": {
    "errorCode": 0,
    "failureReason": null
  },
  "conditionalAccessStatus": "success",
  "authenticationRequirement": "multiFactorAuthentication",
  "mfaDetail": {
    "authMethod": "Phone call",
    "authDetail": "Phone call to +1 XXX-XXX-XXXX"
  },
  "riskDetail": "none",
  "riskLevelAggregated": "none",
  "riskLevelDuringSignIn": "none",
  "riskState": "none",
  "location": {
    "city": "Beijing",
    "state": "Beijing",
    "countryOrRegion": "CN",
    "geoCoordinates": {}
  },
  "deviceDetail": {
    "deviceId": "",
    "operatingSystem": "Windows 10",
    "browser": "Edge 120.0",
    "isCompliant": false,
    "isManagedDevice": false
  },
  "uniqueTokenIdentifier": "token-id-xxxx"
}
```

**研判意义**：`authenticationRequirement: multiFactorAuthentication` + `status.errorCode: 0` 表示使用 MFA 成功登录，属于正常事件。若 `authenticationRequirement: singleFactorAuthentication`（仅密码）且 `errorCode: 0`，则是未使用 MFA 的成功登录，高风险。`uniqueTokenIdentifier` 可与 MicrosoftGraphActivityLogs 的 `SignInActivityId` 字段关联，还原"此次登录后使用令牌做了哪些 API 调用"。

---

### 4.2 密码喷射攻击（大量 SignInLogs 失败）

**触发方式**：攻击者对多个用户账号尝试相同弱密码（密码喷射特征：相同 IP，短时间内，多个不同 userPrincipalName，相同 errorCode）

```json
{
  "id": "spray-event-0001",
  "createdDateTime": "2024-03-15T09:00:01Z",
  "userPrincipalName": "bob@contoso.com",
  "userId": "user-id-bob",
  "appDisplayName": "Microsoft Authentication",
  "ipAddress": "185.220.101.45",
  "isInteractive": true,
  "clientAppUsed": "Browser",
  "status": {
    "errorCode": 50126,
    "failureReason": "Error validating credentials due to invalid username or password."
  },
  "conditionalAccessStatus": "notEvaluated",
  "authenticationRequirement": "singleFactorAuthentication",
  "riskDetail": "none",
  "riskLevelAggregated": "none",
  "location": {
    "city": "Unknown",
    "state": "Unknown",
    "countryOrRegion": "RU"
  }
}
```

**研判意义**：`errorCode: 50126`（凭证验证失败）是密码喷射的标志性错误码。研判规则：相同 `ipAddress` 在 5 分钟内对 ≥ 5 个不同 `userPrincipalName` 产生 `errorCode: 50126`，触发密码喷射告警。`location.countryOrRegion: RU` 配合 `ipAddress` 在 Tor 出口节点 IP 列表中，是进一步升级告警的信号。

---

### 4.3 非交互式登录（NonInteractiveUserSignInLogs，beta API）

**触发方式**：Outlook 桌面客户端在后台使用缓存的刷新令牌静默获取新的访问令牌（用户无感知）

```json
{
  "id": "ni-signin-0001",
  "createdDateTime": "2024-03-15T10:30:00Z",
  "userPrincipalName": "alice@contoso.com",
  "userId": "d0d28bf6-1234-5678-abcd-ef0123456789",
  "appId": "d3590ed6-52b3-4102-aeff-aad2292ab01c",
  "appDisplayName": "Microsoft Office",
  "ipAddress": "10.0.0.50",
  "isInteractive": false,
  "clientAppUsed": "Mobile Apps and Desktop clients",
  "status": {
    "errorCode": 0,
    "failureReason": null
  },
  "conditionalAccessStatus": "success",
  "authenticationRequirement": "singleFactorAuthentication",
  "signInEventTypes": ["nonInteractiveUser"],
  "riskLevelAggregated": "none"
}
```

**研判意义**：非交互式登录大量出现是正常的（每个 Office 客户端每小时都会静默刷新令牌），**单条无意义**。研判价值在于：若某用户的非交互式登录突然从新的、陌生的 `ipAddress` 出现（与其历史登录 IP 显著不同），可能意味着其**刷新令牌被盗并在攻击者的机器上使用**。这是 Storm-0558 类攻击的核心检测模式。

---

### 4.4 恶意 OAuth 应用授权（AuditLogs）

**触发方式**：用户被钓鱼诱导，对攻击者控制的 OAuth 应用授予高权限（OAuth Consent Phishing）

```json
{
  "id": "audit-consent-0001",
  "activityDateTime": "2024-03-15T11:05:44Z",
  "activityDisplayName": "Consent to application",
  "category": "ApplicationManagement",
  "correlationId": "consent-corr-id",
  "result": "success",
  "resultReason": "",
  "initiatedBy": {
    "user": {
      "userPrincipalName": "victim@contoso.com",
      "userId": "victim-user-id",
      "ipAddress": "203.0.113.99"
    }
  },
  "targetResources": [
    {
      "id": "malicious-sp-object-id",
      "displayName": "Totally Legit App",
      "type": "ServicePrincipal",
      "modifiedProperties": [
        {
          "displayName": "ConsentContext.IsAdminConsent",
          "oldValue": null,
          "newValue": "\"False\""
        },
        {
          "displayName": "ConsentContext.OnBehalfOfAll",
          "oldValue": null,
          "newValue": "\"False\""
        }
      ]
    }
  ],
  "additionalDetails": [
    { "key": "PermissionKey", "value": "Mail.Read" },
    { "key": "PermissionKey", "value": "Files.ReadWrite.All" }
  ]
}
```

**研判意义**：`activityDisplayName: "Consent to application"` + `additionalDetails` 中包含 `Mail.Read`、`Files.ReadWrite.All` 等高权限是 OAuth 同意钓鱼的核心信号。需对比 `targetResources[0].id`（服务主体 Object ID）是否在已知可信应用白名单中——不在白名单的新应用被授予高权限，是最强告警信号之一。

---

### 4.5 向用户分配高权限角色（AuditLogs）

**触发方式**：攻击者获得 Privileged Role Administrator 权限后，将自己或受控账号提升为 Global Administrator

```json
{
  "id": "audit-role-assign-0001",
  "activityDateTime": "2024-03-15T12:00:00Z",
  "activityDisplayName": "Add member to role",
  "category": "RoleManagement",
  "correlationId": "role-assign-corr-id",
  "result": "success",
  "initiatedBy": {
    "user": {
      "userPrincipalName": "attacker@contoso.com",
      "userId": "attacker-user-id",
      "ipAddress": "198.51.100.23"
    }
  },
  "targetResources": [
    {
      "id": "victim-user-id",
      "displayName": "Attacker's Backdoor Account",
      "type": "User",
      "modifiedProperties": [
        {
          "displayName": "Role.DisplayName",
          "oldValue": null,
          "newValue": "\"Global Administrator\""
        },
        {
          "displayName": "Role.ObjectID",
          "oldValue": null,
          "newValue": "\"62e90394-69f5-4237-9190-012177145e10\""
        }
      ]
    }
  ]
}
```

**研判意义**：`activityDisplayName: "Add member to role"` + `modifiedProperties` 中 `Role.DisplayName: "Global Administrator"`（Object ID `62e90394-69f5-4237-9190-012177145e10` 是 Global Administrator 角色的固定 ID）是 Entra 身份平面**最高优先级的告警信号**，等同于 AWS 的 `AttachRolePolicy + AdministratorAccess`。任何对 Global Administrator、Privileged Role Administrator、Security Administrator 等高权限角色的成员变更，都应触发即时告警。

---

## 5. 影响基于日志进行告警研判的关键特性

### 5.1 日志规模与客户等级的关联

Entra 日志的量级与 Azure Activity Log 有本质差异：它受**租户用户数量和应用使用频率**驱动，而非 Azure 资源操作频率。

#### 单租户日志量参考

客户通常只接入需要安全监控的核心租户（B2B 和 B2C 租户分别统计），单租户日志量是规划容量的基本单位：

| 租户类型/规模 | 用户数 | SignInLogs（日/条，单租户） | NonInteractiveSignInLogs（日/条） | AuditLogs（日/条） |
|---|---|---|---|---|
| 小型企业租户 | < 300 | **数千 ~ 2 万** | 数万（约 5–10 倍交互式） | 数百 ~ 数千 |
| 中型企业租户 | 300 ~ 5000 | **2 万 ~ 30 万** | 10 万 ~ 300 万 | 数千 ~ 数万 |
| 大型企业租户 | > 5000 | **30 万 ~ 500 万** | 百万 ~ 千万级 | 数万 ~ 数十万 |
| 高频 SaaS 租户 | 大量外部用户/B2C | 极高（取决于产品活跃用户数） | 同样极高 | 中等（内部配置变更不频繁） |

> **NonInteractiveUserSignInLogs 的量级是最难预估的**：每个激活的 Office 365 客户端每小时都会产生至少 1 条非交互式登录，5000 用户 × 平均每人 3 个设备 × 24 次/天 = **360,000 条/天**仅来自 Office 客户端的令牌刷新。实际情况中非交互式登录量级通常是交互式的 **5–10 倍**。

#### 按客户规模的整体分层

| 客户规模 | 典型接入租户数 | 后台汇聚量（Graph API 方案，日/条） | 关键挑战 |
|---|---|---|---|
| 小型企业 | 1 个租户 | 数万 ~ 数十万 | 限流不是瓶颈；主要风险是 30 天保留期管理 |
| 中型企业 | 1–3 个租户 | 百万 ~ 千万 | 非交互式登录量大，5 requests/10s 限流开始成为瓶颈 |
| 大型企业 | 3–10 个租户 | 千万 ~ 亿级 | **Graph API 限流严重积压**；推荐切换为诊断设置方案 |

---

### 5.2 日志延迟

| 采集方案 | 各阶段延迟拆解 | 端到端延迟 | 是否满足准实时（5–15 分钟） |
|---|---|---|---|
| **方案一**（Graph API 轮询） | Entra 日志写入（< 5 分钟）+ 轮询间隔（可配，建议 5 分钟）+ API 响应和处理（秒级–数十秒） | **5–20 分钟**（受限流和 API 响应时间影响） | ✅ 满足（但大租户在限流下可能超出） |
| **方案二**（诊断设置→Event Hub） | 诊断设置推送（秒级）+ Event Hub 消费（秒级） | **2–8 分钟** | ✅ 满足，优于方案一 |

**Entra 日志的写入延迟**：Entra 日志从事件发生到 Graph API 可查询，通常有 **2–5 分钟**的延迟（微软官方未提供 SLA 保证）。这是方案一端到端延迟的底噪。

**非交互式登录的特殊延迟**：NonInteractiveUserSignInLogs 的延迟通常略高于交互式登录（可能达到 10–15 分钟），这是因为非交互式登录事件在后台批量聚合处理，而非实时写入。

---

### 5.3 是否支持流式拉取

**结论：Graph API 方案本质是定时 Pull，不支持推送；诊断设置方案支持近实时 Push。**

| 采集方案 | 数据流动模式 | 最小粒度 | 有无推送机制 |
|---|---|---|---|
| **方案一**（Graph API） | 定时 Pull | 单条事件（`$top=1000` 批量返回） | ❌ 无，必须主动轮询 |
| **方案二**（诊断设置→Event Hub） | Push（近流式） | 每批消息 | ✅ 有，由 Entra 平台主动推送 |

**Change Notification（订阅推送）的限制**：Microsoft Graph 提供 Change Notification（Webhook 订阅）机制可对某些资源变更进行推送，但 `auditLogs/signIns` 和 `auditLogs/directoryAudits` **均不在支持列表中**，无法使用推送机制——这是多个 Microsoft Q&A 帖子中明确确认的事实。

---

### 5.4 是否支持 API 过滤

Graph API 提供服务端过滤能力，可减少后台处理量：

| 过滤维度 | 是否支持 | 示例 | 注意事项 |
|---|---|---|---|
| 时间范围（`createdDateTime`） | ✅ 支持 | `$filter=createdDateTime ge 2024-03-15T10:00:00Z` | **必须设置**，否则返回全量数据（通常超时） |
| 用户（`userPrincipalName`）| ✅ 支持 | `$filter=userPrincipalName eq 'alice@contoso.com'` | 单用户过滤，不建议在批量采集中使用（增加 API 调用次数） |
| 应用（`appId`） | ✅ 支持 | `$filter=appId eq 'xxx'` | 用于针对特定应用的专项监控 |
| 登录结果（`status/errorCode`） | ✅ 支持 | `$filter=status/errorCode eq 50126` | 只拉取失败登录 |
| 登录事件类型（NonInteractive） | ✅ 支持（仅 beta） | `$filter=signInEventTypes/any(t: t eq 'nonInteractiveUser')` | 仅在 `/beta` 端点有效 |
| 按字段排序 | ✅ 支持 | `$orderby=createdDateTime asc` | 建议升序排列，便于从旧到新处理 |
| **地理位置（location）** | ❌ 不支持 | —— | 无法在 API 层过滤来自特定国家的登录，只能后台过滤 |
| **风险等级（riskLevel）** | ❌ 不支持 | —— | 无法在 API 层只拉取高风险登录 |

**与 AWS 和 Azure Activity Log 的对比**：Graph API 的过滤能力比 Azure EventBridge（按事件内容精确路由）和 AWS Advanced Event Selectors（数据源侧过滤）弱，但比 O365 Management Activity API（只能按 Content Type 过滤大类）强。核心限制是**不能按风险等级过滤**，必须全量拉取后后台处理。

---

### 5.5 限流策略

这是 Graph API 方案最核心的工程约束，是整个 Microsoft Graph API 中限制最严格的一组端点：

| 端点 | 专项限制 | 影响 |
|---|---|---|
| `/auditLogs/signIns` | **5 requests / 10 seconds**（Identity and Access Reports 类别）| 每 10 秒最多拿 5 × 1000 = 5000 条记录；大租户单次拉取一个小时的数据可能需要数百次请求，耗时数分钟 |
| `/auditLogs/directoryAudits` | 同上 | 同上 |
| `/beta/auditLogs/signIns` | 同上（beta 同样适用） | 同上 |
| **全局上限** | 每个应用跨所有租户：150,000 requests / 20 seconds（约 7500 RPS） | 单应用服务多租户时，所有租户的请求共享此全局配额 |

**多租户 SaaS 场景的特殊限流风险**：

我们的后台是多租户 SaaS，为多个客户同时采集 Entra 日志。如果后台使用**同一个应用注册（App Registration）**为所有客户的租户调用 Graph API，这些请求会**共享全局配额**（150,000 requests / 20 seconds across all tenants）。当接入客户数量较多时，单个客户的租户所能分配到的有效配额将显著低于 5 requests/10 seconds 的专项限制，限流压力呈倍数放大。

**推荐应对方案**：
- 为不同客户的采集任务设置**排队调度和优先级**（高价值客户优先）
- 监控 Graph API 响应头中的 `x-ms-throttle-limit-percentage` 字段（超过 0.8 即预警）
- 实现指数退避，严格遵守 `Retry-After` 响应头中的等待时间

> **参考文档**：
> - [Microsoft Graph service-specific throttling limits – Identity and Access Reports](https://learn.microsoft.com/en-us/graph/throttling-limits)
> - [Microsoft Graph throttling guidance](https://learn.microsoft.com/en-us/graph/throttling)

---

### 5.6 日志时序一致性

**各方案的时间字段语义**：

| 方案 | 日志类别 | 正确时间基准 | 易混淆字段 | 偏差量 |
|---|---|---|---|---|
| 方案一（Graph API） | SignInLogs | `createdDateTime` | 无（Graph API 返回的就是事件发生时间） | 无 |
| 方案一（Graph API） | AuditLogs | `activityDateTime` | 无 | 无 |
| 方案二（Event Hub） | 全部 | `records[].time`（JSON 内层） | Event Hub 消息的 `EnqueuedTimeUtc`（入队时间） | 约 秒级–数分钟 |

**Graph API 方案的时序特殊性**：Graph API 返回的数据按 `createdDateTime` 降序排列（最新在前），但后台做增量拉取时应指定 `$orderby=createdDateTime asc`（升序），确保从旧到新处理，避免因降序导致的"跳过中间数据"问题。

**跨日志类型关联时的时序问题**：

当把 SignInLogs（来自方案一 Graph API）和 MicrosoftGraphActivityLogs（来自方案二 Event Hub）关联时，两路数据的时间基准字段名不同（`createdDateTime` vs `records[].time`），且两路到达后台的时间可能相差数分钟。关联键应使用 `SignInLogs.uniqueTokenIdentifier` = `MicrosoftGraphActivityLogs.SignInActivityId`，并设置 ≥ 15 分钟的时间关联窗口容忍乱序。

---

### 5.7 多租户日志汇聚

Entra 日志天然是**租户维度**的（而非 Azure 订阅维度），每个租户有独立的日志，需要为每个接入的客户租户分别配置采集。

**跨租户攻击的可见性挑战**：

在 B2B 场景中，租户 A 的用户（外部来宾用户）登录到租户 B 的资源时，登录事件会出现在**租户 B 的 SignInLogs** 中，而非租户 A。若租户 A 是被攻陷的供应商，攻击者以租户 A 的用户身份横向移动至租户 B，这次跨租户登录在**租户 A 的日志中完全不可见**（因为认证发生在租户 B）。

识别跨租户登录的关键字段：
- `homeTenantId`：用户归属的原始租户 ID（来宾用户的原租户）
- `resourceTenantId`：访问资源所在的租户 ID

若 `homeTenantId ≠ resourceTenantId`，说明这是一次跨租户访问，**必须将 `homeTenantId` 对应的租户也纳入接入范围，才能看到完整的身份链路**。

---

## 6. 安全厂商经验项总结

### 6.1 案例一：Storm-0558——通过伪造令牌绕过 Entra 认证访问政府邮箱（Microsoft / CISA，2023 年 7 月）

**事件背景**：

2023 年 6 月至 7 月，Microsoft 安全团队披露一起针对政府机构 Exchange Online 邮箱的高级攻击（Storm-0558，归因为中国国家级行为者）。攻击者利用一枚泄露的 Microsoft MSA（消费者账户）签名私钥，伪造企业租户的 Entra ID 访问令牌，绕过所有基于令牌签名验证的安全机制，在约 6 周内持续访问约 25 个组织的高管邮箱内容。

**Entra 日志的可见性分析**：

| 攻击阶段 | Entra 日志可见性 | 具体信号 | 为什么难以检测 |
|---|---|---|---|
| MSA 密钥泄露 | ❌ **完全不可见** | —— | 发生在微软内部系统，租户侧日志不覆盖 |
| 用伪造令牌调用 OWA（Exchange） | ✅ **SignInLogs 有记录** | 登录 IP 异常；`appId` 为 OWA 的 App ID；`clientAppUsed: Browser` | **令牌通过签名验证，与正常登录外观完全一致**，MFA 状态正常，条件访问结果为 success |
| 大量读取邮件内容 | ⚠️ **部分可见**（需 E5 / Audit Premium） | M365 UAL 中的 `MailItemsAccessed`（E5 专属）；`ClientAppID` 值异常 | E3 及以下完全不可见；E5 的 `MailItemsAccessed` 记录了邮件被访问但不记录邮件内容 |
| 持续约 6 周的静默访问 | ⚠️ **理论可见，实际难发现** | NonInteractiveSignInLogs 有记录，但混在正常的令牌刷新记录中 | 攻击者的访问模式与正常客户端的令牌刷新行为高度相似，需要基线模型才能区分 |

**最关键检测发现**：

首先发现此次攻击的是一个使用 **E5 License（Audit Premium）** 的美国联邦政府机构，通过观察 `MailItemsAccessed` 事件中出现的**异常 `ClientAppID`**（不在已知 Microsoft 应用列表中的 App ID）才触发了调查。这再次证明：

1. **License 等级直接决定可见深度**——E3 客户对此次攻击完全不可见
2. **CSRB 调查报告明确指出**：由于日志保留策略限制，微软没有具体的日志证据来还原密钥泄露的全部过程，即便是微软自身的 Entra 日志也无法覆盖 2021 年的攻击起点

**对安全厂商的启示**：

- Entra SignInLogs 中，识别伪造令牌攻击不能依赖"登录是否成功"（攻击者成功了），而需要依赖**行为基线的偏离**：用户在非工作时间段登录、从未使用过的 IP 段登录、访问了历史上从未访问的资源
- `ClientAppID` 白名单是当前最有效的检测手段：对所有 SignInLogs 中 `appId` 不在组织已知应用白名单中的记录设置告警

> **参考来源**：
> - Microsoft 技术分析：[Analysis of Storm-0558 techniques for unauthorized email access (Jul 2023)](https://www.microsoft.com/en-us/security/blog/2023/07/14/analysis-of-storm-0558-techniques-for-unauthorized-email-access/)
> - CISA 联合公告：[Enhanced Monitoring to Detect APT Activity Targeting Outlook Online (CISA AA23-193A, Jul 2023)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-193a)
> - CSRB 完整调查报告：[CSRB Review of Summer 2023 Microsoft Exchange Online Intrusion (Mar 2024)](https://www.cisa.gov/sites/default/files/2025-03/CSRBReviewOfTheSummer2023MEOIntrusion508.pdf)
> - Wiz 深度分析：[Storm-0558 Compromised Microsoft Key (Jul 2023)](https://www.wiz.io/blog/storm-0558-compromised-microsoft-key-enables-authentication-of-countless-micr)

---

### 6.2 案例二：OAuth 同意钓鱼攻击——通过 AuditLogs 检测恶意应用授权（CISA / Microsoft，2023）

**事件背景**：

CISA 和 FBI 在 2023 年联合发布的《云安全最佳实践》报告中，将 OAuth 同意钓鱼（Consent Phishing）列为针对 Microsoft 365 的最活跃攻击手法之一。攻击者注册一个看似合法的第三方应用，通过钓鱼邮件引导员工点击授权链接，诱导其为该应用授予高权限（如 `Mail.Read`、`Files.ReadWrite.All`）。一旦用户点击"允许"，攻击者无需用户密码，即可持续访问用户的邮件和文件，且 MFA 不能阻止这种攻击（授权时 MFA 已验证通过）。

**AuditLogs 中的可见信号**：

| 攻击阶段 | AuditLogs 事件 | 关键字段 |
|---|---|---|
| 用户授权恶意应用 | `activityDisplayName: "Consent to application"` | `targetResources[].displayName`（应用名）、`additionalDetails.PermissionKey`（被授予的权限） |
| 攻击者注册新应用 | `activityDisplayName: "Add application"` | `targetResources[].displayName`（新应用名）、应用创建时间 |
| 攻击者为应用分配高权限 | `activityDisplayName: "Add app role assignment to service principal"` | 分配的角色名称 |

**研判建议**：

告警规则：`activityDisplayName = "Consent to application"` + `additionalDetails.PermissionKey` 包含以下任意高危权限：

```
Mail.Read / Mail.ReadWrite / Mail.Send
Files.ReadWrite.All / Files.Read.All
Directory.ReadWrite.All
RoleManagement.ReadWrite.Directory
```

同时检查 `targetResources[0].id`（服务主体 Object ID）是否为近期新注册的应用（通过关联 `Add application` 审计事件判断）。若新注册应用在 24 小时内即被用户授权，是高度可疑的信号。

> **参考来源**：
> - CISA/FBI 云安全指导：[Cybersecurity Advisory – Known Indicators of Compromise Associated with Androxgh0st Malware (Jan 2024)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa24-016a)
> - Microsoft 威胁情报：[Threat actors misuse OAuth applications to automate financially driven attacks (Dec 2023)](https://www.microsoft.com/en-us/security/blog/2023/12/12/threat-actors-misuse-oauth-applications-to-automate-financially-driven-attacks/)
> - Proofpoint 设备代码钓鱼研究：[Access granted: phishing with device code authorization for account takeover (2025)](https://www.proofpoint.com/us/blog/threat-insight/access-granted-phishing-device-code-authorization-account-takeover)

---

### 6.3 核心经验项总结

**① Entra 日志 30 天保留期是接入时最高优先级的配置检查项**

Entra SignInLogs 和 AuditLogs 在门户内置保留仅 30 天（P1/P2），过期后**任何方法都无法找回**，包括联系微软支持。安全厂商必须将"验证 Entra 诊断设置是否已配置持续导出"作为接入客户的第一步强制动作——这不是优化项，是基础保障。

**② MicrosoftGraphActivityLogs 是 Graph API 方案的固有盲区，必须向产品和客户明确说明**

当前 Graph API 采集实现无法覆盖 MicrosoftGraphActivityLogs，这意味着：攻击者在成功认证后通过 Graph API 读取用户邮件、枚举组织成员、调用 Teams/SharePoint 等操作，在我们的采集结果中**完全不可见**。该日志类别只能通过诊断设置获取。对于没有 Azure 订阅的纯 Entra 客户，这个盲区不可避免，需要在产品能力说明中明确。

**③ NonInteractiveUserSignInLogs 的日志量是建设采集容量最大的不确定因素**

非交互式登录的量级通常是交互式的 5–10 倍，且受租户内激活设备数量影响，难以在接入前准确估算。结合 Graph API 的严格限流（5 req/10s），大租户的非交互式登录拉取是导致积压的头号风险。在接入中大型客户时，应提前通过 Entra 门户的使用量报告估算各类日志的日均条数，再规划轮询频率。

**④ 令牌视角比登录视角更重要：关注"令牌颁发后做了什么"**

现代攻击（Storm-0558、OAuth 钓鱼、设备代码钓鱼）越来越多地绕过密码认证，直接利用合法令牌。SignInLogs 只能看到"令牌在何时、以何种方式颁发"，但攻击者持有令牌后的持续访问行为（尤其是非交互式的令牌刷新）在 SignInLogs 中极难与正常行为区分。告警引擎需要从"谁在登录"转向"令牌被在哪些 IP、哪些应用、什么时间段使用"的行为基线分析。

**⑤ `activityDisplayName` 白名单是 AuditLogs 告警规则的核心设计方式**

AuditLogs 中最高价值的事件集中在少数几个 `activityDisplayName` 值上，建议优先对以下操作设置告警：

| 操作名称 | 安全含义 |
|---|---|
| `Add member to role` | 目录角色成员变更（特别是 Global Admin 等高权限角色） |
| `Consent to application` | OAuth 应用权限授予（重点关注高权限 scope） |
| `Add application` | 新应用注册（结合后续的 Consent 事件关联分析） |
| `Update application – Certificates and secrets management` | 应用凭证（Client Secret/证书）变更 |
| `Add service principal credentials` | 服务主体凭证添加 |
| `Set company information` | 租户级别配置变更 |

**⑥ Graph API beta 端点的稳定性是长期运维风险**

NonInteractiveUserSignInLogs 依赖 beta 端点，微软 beta API 可能在无事先通知的情况下变更字段或行为。应在监控系统中设置 Schema 校验，对 API 返回的字段结构变化触发告警，而不是等到日志解析失败时才发现问题。

**⑦ 多租户 SaaS 限流放大效应需要从架构层面解决**

当后台同一应用注册为多个客户租户调用 Graph API 时，150,000 requests/20s 的全局配额被所有租户共享。接入客户数量越多，每个租户分配到的有效配额越少。应对方案：为不同客户的采集任务设置优先级队列；或考虑为客户分组，不同分组使用不同的应用注册，实现配额隔离。

---

## 参考文档索引

| 类别 | 文档名称 | 链接 |
|---|---|---|
| **日志基础** | Entra 审计日志 API 总览 | https://learn.microsoft.com/en-us/graph/api/resources/azure-ad-auditlog-overview?view=graph-rest-1.0 |
| | List signIns（v1.0） | https://learn.microsoft.com/en-us/graph/api/signin-list?view=graph-rest-1.0 |
| | List directoryAudits | https://learn.microsoft.com/en-us/graph/api/directoryaudit-list?view=graph-rest-1.0 |
| | MicrosoftGraphActivityLogs 总览 | https://learn.microsoft.com/en-us/graph/microsoft-graph-activity-logs-overview |
| | Entra 数据保留策略 | https://learn.microsoft.com/en-us/entra/identity/monitoring-health/reference-reports-data-retention |
| **采集方案** | 分析活动日志（Graph 最佳实践） | https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-analyze-activity-logs-with-microsoft-graph |
| | 流式传输 Entra 日志到 Event Hub | https://learn.microsoft.com/en-us/entra/identity/monitoring-health/howto-stream-logs-to-event-hub |
| | Entra 诊断设置日志类型 | https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-diagnostic-settings-logs-options |
| | 读取活动日志所需权限 | https://learn.microsoft.com/en-us/entra/identity/monitoring-health/concept-audit-log-permissions |
| **限流** | Graph API 限流总览 | https://learn.microsoft.com/en-us/graph/throttling |
| | Identity and Access Reports 专项限流 | https://learn.microsoft.com/en-us/graph/throttling-limits |
| | Delta Query 支持资源列表 | https://learn.microsoft.com/en-us/graph/delta-query-overview |
| **攻防案例** | Storm-0558 技术分析（Microsoft, Jul 2023） | https://www.microsoft.com/en-us/security/blog/2023/07/14/analysis-of-storm-0558-techniques-for-unauthorized-email-access/ |
| | CISA AA23-193A——Storm-0558 增强监控 | https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-193a |
| | CSRB Summer 2023 Exchange Online Intrusion Review（Mar 2024） | https://www.cisa.gov/sites/default/files/2025-03/CSRBReviewOfTheSummer2023MEOIntrusion508.pdf |
| | Wiz – Storm-0558 密钥影响分析 | https://www.wiz.io/blog/storm-0558-compromised-microsoft-key-enables-authentication-of-countless-micr |
| | Microsoft – OAuth 应用滥用（Dec 2023） | https://www.microsoft.com/en-us/security/blog/2023/12/12/threat-actors-misuse-oauth-applications-to-automate-financially-driven-attacks/ |
| | Proofpoint – 设备代码钓鱼（2025） | https://www.proofpoint.com/us/blog/threat-insight/access-granted-phishing-device-code-authorization-account-takeover |

---

*本文档所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有 API 或产品变更，以微软官方最新文档为准。*