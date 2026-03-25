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
3. [日志示例](#3-日志示例)
4. [影响基于日志进行告警研判的关键特性](#4-影响基于日志进行告警研判的关键特性)
5. [安全厂商经验项总结](#5-安全厂商经验项总结)

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

## 3. 日志示例

以下示例均基于微软官方文档中的 Schema 构造，标注触发方式和研判意义。

### 3.1 交互式登录成功（SignInLogs）

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
  "authenticationRequirement": "multiFactorAuthentication",
  "mfaDetail": {
    "authMethod": "Phone call",
    "authDetail": "Phone call to +1 XXX-XXX-XXXX"
  },
  "conditionalAccessStatus": "success",
  "riskDetail": "none",
  "riskLevelAggregated": "none",
  "location": {
    "city": "Shanghai",
    "state": "Shanghai",
    "countryOrRegion": "CN"
  },
  "deviceDetail": {
    "deviceId": "",
    "operatingSystem": "Windows 10",
    "browser": "Edge 120.0"
  }
}
```

**研判意义**：
- **正常信号**：MFA 已验证、条件访问 success、已知 IP 范围、工作时间
- **告警触发条件**：首次登录国家 / IP、未使用 MFA 的成功登录 / `authenticationRequirement` 值为 `singleFactorAuthentication`（说明该用户账户未配置 MFA 强制策略）

---

### 3.2 交互式登录失败——密码错误（SignInLogs）

**触发方式**：用户输入错误的密码，登录被拒

```json
{
  "id": "4d96ebf6-1c7f-4465-8a1e-5a234567cdef",
  "createdDateTime": "2024-03-15T08:15:45Z",
  "userDisplayName": "Bob Johnson",
  "userPrincipalName": "bob@contoso.com",
  "userId": "a1b2c3d4-5e6f-7a8b-9c0d-e1f2a3b4c5d6",
  "appId": "c44b4083-3bb0-49c1-b47d-974e53cbdf3c",
  "appDisplayName": "Azure Portal",
  "ipAddress": "198.51.100.89",
  "isInteractive": true,
  "clientAppUsed": "Browser",
  "status": {
    "errorCode": 50126,
    "failureReason": "Invalid username or password."
  },
  "location": {
    "city": "Unknown",
    "state": null,
    "countryOrRegion": "US"
  }
}
```

**研判意义**：
- `errorCode: 50126` = 凭证错误
- **告警规则**：同一用户在 10 分钟内出现 ≥5 次 50126 失败 = **密码喷射攻击** → 高优先级告警
- **关键字段**：`ipAddress` + 失败次数聚合 = 源 IP 维度的密码喷射检测

---

### 3.3 非交互式登录——令牌刷新（NonInteractiveUserSignInLogs）

**触发方式**：Outlook 桌面客户端在后台刷新令牌，用户无感知

```json
{
  "id": "8f7e6d5c-4b3a-2f1e-0d9c-b8a7f6e5d4c3",
  "createdDateTime": "2024-03-15T09:30:12Z",
  "userDisplayName": "Carol White",
  "userPrincipalName": "carol@contoso.com",
  "userId": "f1e2d3c4-b5a6-7890-abcd-ef1a2b3c4d5e",
  "appId": "d3590ed6-52b3-4102-aeff-aad2292ab01c",
  "appDisplayName": "Office 365 Exchange Online",
  "ipAddress": "192.0.2.100",
  "isInteractive": false,
  "signInEventTypes": ["nonInteractiveUser"],
  "clientAppUsed": "Mobile Apps and Desktop clients",
  "status": {
    "errorCode": 0,
    "failureReason": null
  },
  "location": {
    "city": "Beijing",
    "state": null,
    "countryOrRegion": "CN"
  }
}
```

**研判意义**：
- `isInteractive: false` + `signInEventTypes: ["nonInteractiveUser"]` = **非交互式令牌刷新**
- **日志量激增**：非交互式登录量通常是交互式的 5–10 倍，会显著冲击采集容量
- **告警策略**：基线分析最关键——该用户历史上是否在该时间、该 IP、从该应用发起过令牌刷新；若突然出现"未知 IP + 陌生应用 + 非工作时间"的组合，是令牌滥用的强信号

---

### 3.4 目录审计——新用户创建（AuditLogs）

**触发方式**：管理员在 Entra Admin Center 新建用户

```json
{
  "id": "Directory_aaa1bbb2-cccc-dddd-eeee-fff0aaa1bbb2",
  "activityDateTime": "2024-03-15T10:45:23Z",
  "activityDisplayName": "Add user",
  "category": "UserManagement",
  "result": "success",
  "resultReason": null,
  "initiatedBy": {
    "user": {
      "id": "1a2b3c4d-5e6f-7a8b-9c0d-e1f2a3b4c5d6",
      "displayName": "Admin User",
      "userPrincipalName": "admin@contoso.com"
    },
    "app": null
  },
  "targetResources": [
    {
      "id": "f0a1b2c3-d4e5-f6a7-b8c9-d0e1f2a3b4c5",
      "displayName": "New User Account",
      "type": "User",
      "userPrincipalName": "newuser@contoso.com",
      "modifiedProperties": [
        {
          "displayName": "UserPrincipalName",
          "oldValue": null,
          "newValue": "\"newuser@contoso.com\""
        }
      ]
    }
  ],
  "correlationId": "abc1def2-ghi3-jkl4-mno5-pqr6stu7vwx8"
}
```

**研判意义**：
- **关键字段**：`activityDisplayName: "Add user"`、`initiatedBy.user.userPrincipalName`（谁创建的）、`targetResources[0].userPrincipalName`（新用户 UPN）
- **告警规则**：检测非工作时间 / 非预期管理员创建的新用户 → **APT 横向移动信号**
- **高危场景**：新创建的用户在 24 小时内被分配了 Global Admin 角色 = 强持久化信号

---

### 3.5 目录审计——OAuth 应用授权（AuditLogs）

**触发方式**：用户点击钓鱼邮件中的授权链接，为第三方应用授权

```json
{
  "id": "Directory_bbb2ccc3-dddd-eeee-ffff-aaa2bbb3ccc4",
  "activityDateTime": "2024-03-15T11:20:15Z",
  "activityDisplayName": "Consent to application",
  "category": "ApplicationManagement",
  "result": "success",
  "resultReason": null,
  "initiatedBy": {
    "user": {
      "id": "e1f2a3b4-c5d6-e7f8-a9b0-c1d2e3f4a5b6",
      "displayName": "Employee D",
      "userPrincipalName": "employee.d@contoso.com"
    },
    "app": null
  },
  "targetResources": [
    {
      "id": "service-principal-id-xyz",
      "displayName": "Totally Legit Email App",
      "type": "ServicePrincipal",
      "modifiedProperties": [
        {
          "displayName": "ConsentContext.IsAdminConsent",
          "oldValue": null,
          "newValue": "false"
        },
        {
          "displayName": "PermissionKey",
          "oldValue": null,
          "newValue": "[\"Mail.Read\", \"Mail.ReadWrite\", \"Files.ReadWrite.All\"]"
        }
      ]
    }
  ],
  "correlationId": "def3ghi4-jkl5-mno6-pqr7-stu8vwx9yza0"
}
```

**研判意义**：
- **关键字段**：`activityDisplayName: "Consent to application"`、`targetResources[].modifiedProperties.PermissionKey`（授予的权限）
- **高危权限白名单**：
  - `Mail.Read` / `Mail.ReadWrite` / `Mail.Send`
  - `Files.ReadWrite.All` / `Files.Read.All`
  - `Directory.ReadWrite.All`
  - `RoleManagement.ReadWrite.Directory`
- **告警规则**：新应用（注册时间 < 24h）被授予高危权限 = **OAuth 钓鱼的高置信信号** → 立即隔离该应用、撤销授权

---

### 3.6 Graph API 活动日志——异常 API 调用（MicrosoftGraphActivityLogs）

**触发方式**：攻击者持有合法令牌，通过 Graph API 大量读取邮件

```json
{
  "id": "graph-activity-xyz",
  "timestamp": "2024-03-15T12:05:33Z",
  "tenantId": "12345678-1234-1234-1234-123456789012",
  "userId": "abcd1234-abcd-1234-abcd-1234abcd1234",
  "userDisplayName": "Compromised Account",
  "userPrincipalName": "compromised@contoso.com",
  "servicePrincipalId": null,
  "clientAppId": "d3590ed6-52b3-4102-aeff-aad2292ab01c",
  "clientAppName": "Office 365 Exchange Online",
  "signInActivityId": "unique-token-id-abc123",
  "requestMethod": "GET",
  "requestUri": "/v1.0/me/mailFolders/inbox/messages?$top=10&$filter=receivedDateTime gt 2024-03-01",
  "resourceDisplayName": "Mail.Read",
  "statusCode": 200,
  "requestId": "req-id-12345"
}
```

**研判意义**：
- **关键字段**：`requestUri`（调用了哪个 API）、`clientAppId` + `clientAppName`（哪个应用）、`statusCode: 200`（请求成功）
- **告警规则**：
  - 短时间内大量调用 `/me/mailFolders/*/messages` / `/me/drive/root/children` = **数据窃取信号** → 检查是否与 SignInLogs 中的异常登录关联
  - `clientAppId` 不在组织已知应用白名单 = **非授权应用访问** → 高优先级
- **与 SignInLogs 的关联**：通过 `signInActivityId` 与 SignInLogs 中的 `uniqueTokenIdentifier` 匹配，还原完整的"谁在什么时间从哪里获取的令牌，然后用该令牌做了什么"的完整链条

---

## 4. 影响基于日志进行告警研判的关键特性

### 4.1 License 等级决定日志可见深度

| License 等级 | SignInLogs | AuditLogs | MicrosoftGraphActivityLogs | 备注 |
|---|---|---|---|---|
| **Entra Free** | ✅（7 天保留） | ✅（7 天保留） | ❌ | 仅 Entra 基础功能，团队级别推荐 |
| **Entra P1** | ✅（30 天保留，风险检测） | ✅（30 天保留） | ❌ | 中小企业推荐，包含基础风险检测 |
| **Entra P2** | ✅（30 天保留，完整风险检测） | ✅（30 天保留） | ❌ | 企业级推荐，Identity Protection 完整功能 |
| **M365 E3** | ✅（SignInLogs，30 天保留） | ✅（部分 AuditLogs，30 天保留） | ❌ | Exchange Online 审计功能，但 Entra 日志功能受限 |
| **M365 E5** | ✅（完整） | ✅（完整 AuditLogs） | ⚠️ **仅标注，需诊断设置导出** | Audit Premium，包含高级审计和威胁情报 |

**关键洞察**：
- **Storm-0558 案例启示**：拥有 E3 License 的组织对此次攻击 **完全不可见**（E3 不支持 MicrosoftGraphActivityLogs，也不包含 Graph API 活动审计）；只有 E5 客户通过 `MailItemsAccessed` 记录才发现了异常
- **建议**：为关键用户/应用分配 P2 + Audit Premium License，确保能看到 MicrosoftGraphActivityLogs；无 Azure 订阅时无法配置诊断设置，该类 License 升级是唯一的增强可见性方案

---

### 4.2 条件访问策略的应用与绕过

| 策略配置 | SignInLogs 中的体现 | 攻击者的绕过可能 |
|---|---|---|
| 要求 MFA | `conditionalAccessStatus: success`，`authenticationRequirement: multiFactorAuthentication` | 伪造令牌（Storm-0558）、令牌回放（已持有合法令牌）、社工跳过（罕见） |
| 基于风险的要求 | `riskLevelAggregated: high` → 触发额外认证 | Identity Protection 配置不当、ML 模型被绕过 |
| 仅允许托管设备 | `deviceDetail.isCompliant: true` | 设备信息可被伪造 / Intune 配置失误 |
| IP 白名单 | `ipAddress` 在预期范围内 | VPN / 代理伪装 IP |

**关键启示**：
- 条件访问 + 风险检测是**外围防线**，但均可被绕过；**内层防线应基于令牌颁发后的行为**——即 MicrosoftGraphActivityLogs 中调用了哪些 API、是否异常
- SignInLogs 显示"认证成功"不代表"该用户的后续操作都是合法的"；必须配套 Graph 活动日志分析

---

### 4.3 Identity Protection 的风险评分

Entra P2 及以上 License 包含 Identity Protection，在 SignInLogs 中会填充风险字段：

| 字段 | 含义 | 用途 |
|---|---|---|
| `riskDetail` | 风险原因（`none` / `adminGeneratedTemporaryPassword` / `userPerformedSecuredPasswordChange` / `userPerformedSecuredPasswordReset` / `adminDismissedAllRiskForUser` / `adminConfirmedSigninSafe` / `adminConfirmedUserCompromised` / `unknownFutureValue`） | 快速判断风险源头 |
| `riskLevelAggregated` | 聚合风险等级（`none` / `low` / `medium` / `high`） | **高优先级告警触发阈值** |
| `riskLevelDuringSignIn` | 登录时的实时风险（同样分 4 级） | 条件访问决策的输入 |

**高危场景**：
- `riskLevelAggregated: high` + `conditionalAccessStatus: success` = Identity Protection 检测到高风险，但用户仍通过了认证（可能已补充 MFA 或被管理员豁免）
- 告警规则：聚合风险为 high 的所有登录需要人工审查，检查是否存在真实入侵

---

### 4.4 设备信息与设备合规性

`deviceDetail` 中记录的信息：

```json
"deviceDetail": {
  "deviceId": "d0d28bf6-1234-5678-abcd-ef0123456789",
  "displayName": "LAPTOP-XYZ",
  "operatingSystem": "Windows 10",
  "browser": "Edge 120.0",
  "isCompliant": true,
  "trustType": "Hybrid Azure AD joined"
}
```

**研判意义**：
- `isCompliant: true` = 该设备已向 Intune 注册、满足 MDM 策略（如防病毒、磁盘加密）
- `trustType: Hybrid Azure AD joined` = 设备既加入本地 AD，也加入 Azure AD（通常是企业自有设备）
- `trustType: Azure AD joined` = 仅加入 Azure AD（可能是 BYOD 设备或虚拟设备）

**告警规则**：
- 同一用户在短时间内从大量不同的 `deviceId` 登录 = 令牌滥用信号（攻击者在多个设备上使用盗取的令牌）
- `isCompliant: false` 的设备执行敏感操作（如修改角色、下载邮件） = 合规风险告警

---

### 4.5 地理位置异常与 Impossible Travel

`location` 字段基于 IP 地址的地理位置数据库：

```json
"location": {
  "city": "Shanghai",
  "state": "Shanghai",
  "countryOrRegion": "CN",
  "geoCoordinates": {
    "latitude": 31.2304,
    "longitude": 121.4737
  }
}
```

**研判逻辑**：
- **Impossible Travel**：用户在不可能的时间间隔内出现在相距甚远的两个地理位置
  - 例：用户在 10:00 从上海登录（中国），10:15 又从纽约登录（美国） → 物理上不可能 → **高置信入侵信号**
- **新国家登录**：首次从某国家登录 → **中风险告警**（可能是出差，但需要 context 分析）

**注意**：
- 代理 / VPN 会导致位置信息不准（显示代理服务器位置，而非实际用户位置）
- 某些 ISP 地理位置数据库更新滞后，可能导致误告警

**推荐策略**：
- 基线学习：建立用户的"常见登录国家/城市"白名单
- 实时告警：新国家 + 高权限用户 = 立即告警
- Impossible Travel：严格启用，但需要考虑 VPN 场景，避免过多误报

---

### 4.6 应用身份识别

| 字段 | 说明 | 研判用途 |
|---|---|---|
| `appId` | 应用的 Azure AD Object ID（如 `c44b4083-3bb0-49c1-b47d-974e53cbdf3c` = Azure Portal） | 应用白名单的基准 |
| `appDisplayName` | 应用的显示名称（如 `Azure Portal`） | 人类可读的快速判断 |
| `clientAppUsed` | 客户端类型（`Browser` / `Mobile Apps and Desktop clients` / `Exchange ActiveSync` / `Other clients`） | 识别自动化工具 + 移动设备 |

**告警规则**：
- **已知应用白名单**：对所有 SignInLogs 检查 `appId` 是否在组织已知应用白名单中
  - 白名单包括：Azure Portal、Microsoft 365、Outlook、Teams、SharePoint 等内置应用
  - 非白名单应用 = **高危信号**（第三方应用可能被恶意注册）
- **客户端类型异常**：
  - Exchange 用户突然用"Mobile Apps"登录（原本只用浏览器） = 中等风险（可能设备更换）
  - SharePoint 用户出现"Other clients"登录 = 高风险（可能是自定义脚本或恶意工具）

**Storm-0558 案例教训**：攻击者的伪造令牌在 SignInLogs 中的 `appId` 是合法的（OWA App ID），外观完全正常。**不能依赖"appId 是否合法"来判断是否被攻陷**，必须依赖行为基线（IP、地点、时间、API 调用模式）

---

### 4.7 多租户日志汇聚

Entra 日志天然是**租户维度**的（而非 Azure 订阅维度），每个租户有独立的日志，需要为每个接入的客户租户分别配置采集。

**跨租户攻击的可见性挑战**：

在 B2B 场景中，租户 A 的用户（外部来宾用户）登录到租户 B 的资源时，登录事件会出现在**租户 B 的 SignInLogs** 中，而非租户 A。若租户 A 是被攻陷的供应商，攻击者以租户 A 的用户身份横向移动至租户 B，这次跨租户登录在**租户 A 的日志中完全不可见**（因为认证发生在租户 B）。

识别跨租户登录的关键字段：
- `homeTenantId`：用户归属的原始租户 ID（来宾用户的原租户）
- `resourceTenantId`：访问资源所在的租户 ID

若 `homeTenantId ≠ resourceTenantId`，说明这是一次跨租户访问，**必须将 `homeTenantId` 对应的租户也纳入接入范围，才能看到完整的身份链路**。

---

## 5. 安全厂商经验项总结

### 5.1 案例一：Storm-0558——通过伪造令牌绕过 Entra 认证访问政府邮箱（Microsoft / CISA，2023 年 7 月）

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

### 5.2 案例二：OAuth 同意钓鱼攻击——通过 AuditLogs 检测恶意应用授权（CISA / Microsoft，2023）

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

### 5.3 核心经验项总结

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
| **安全案例** | Storm-0558 技术分析（Microsoft, Jul 2023） | https://www.microsoft.com/en-us/security/blog/2023/07/14/analysis-of-storm-0558-techniques-for-unauthorized-email-access/ |
| | CISA AA23-193A——Storm-0558 增强监控 | https://www.cisa.gov/news-events/cybersecurity-advisories/aa23-193a |
| | CSRB Summer 2023 Exchange Online Intrusion Review（Mar 2024） | https://www.cisa.gov/sites/default/files/2025-03/CSRBReviewOfTheSummer2023MEOIntrusion508.pdf |
| | Wiz – Storm-0558 密钥影响分析 | https://www.wiz.io/blog/storm-0558-compromised-microsoft-key-enables-authentication-of-countless-micr |
| | Microsoft – OAuth 应用滥用（Dec 2023） | https://www.microsoft.com/en-us/security/blog/2023/12/12/threat-actors-misuse-oauth-applications-to-automate-financially-driven-attacks/ |
| | Proofpoint – 设备代码钓鱼（2025） | https://www.proofpoint.com/us/blog/threat-insight/access-granted-phishing-device-code-authorization-account-takeover |

---

*本文档所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有 API 或产品变更，以微软官方最新文档为准。*