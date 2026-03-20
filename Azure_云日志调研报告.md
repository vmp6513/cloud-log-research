# Azure 云日志调研报告

> **文档状态**：版本 1.0  
> **调研时间**：2026-03  
> **作者**：云安全专家调研  
> **适用读者**：规划侧 / 产品侧 / 安全运营团队  
> **核心聚焦**：Azure Activity Log + Resource Logs（Azure 云上行为审计的核心日志系统）  
> **范围说明**：本报告仅覆盖 Azure 平台侧日志（Activity Log、Resource Logs）；Entra ID 活动日志（登录日志、审计日志）属于身份平面，将单独调研。

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

Azure 的审计日志体系没有类似 AWS CloudTrail 的统一入口，而是由**平台侧**和**身份侧**两套独立系统分担。本报告仅覆盖平台侧的 **Azure Activity Log** 和 **Resource Logs**，身份侧的 Entra ID 日志（登录日志、目录审计日志）将在 Entra 专项调研中覆盖。

### 1.1 核心审计日志：Azure Activity Log（ARM 控制面审计）

Azure Activity Log 是 Azure Resource Manager（ARM）层面的操作审计日志，记录"谁通过 ARM 对哪个资源做了什么操作、结果如何"，是 Azure 云上控制面可见性的最核心数据源。

Activity Log 的条目默认被收集，无需任何配置，且由系统生成、不可更改或删除。这些条目通常是变更操作（create、update、delete）或已发起操作的结果，关注资源详情的读取操作通常不被捕获。

Activity Log 按事件类别划分为以下类型：

| 事件类别 | 内容描述 | 安全意义 |
|---|---|---|
| **Administrative** | 通过 ARM 执行的所有 Create/Update/Delete/Action 操作，如创建 VM、删除 NSG、修改 RBAC 角色分配 | **最高**，是检测资源操作和权限变更的核心来源 |
| **Security** | Microsoft Defender for Cloud 生成的告警 | 高，提供 Azure 原生威胁检测信号 |
| **ServiceHealth** | Azure 平台自身服务健康事件（如服务中断、计划维护） | 低（安全场景价值有限） |
| **ResourceHealth** | 资源级别的健康状态变化 | 低 |
| **Alert** | Azure Monitor 告警触发事件 | 中，可用于监控规则触发情况 |
| **Autoscale** | 基于自动缩放设置触发的操作 | 低 |
| **Recommendation** | Azure Advisor 建议事件 | 低 |
| **Policy** | Azure Policy 执行的所有 Effect Action 操作（Audit、Deny） | 中，用于合规管控审计 |

### 1.2 Resource Logs（资源诊断日志）

Resource Logs 记录高于控制面的操作，默认不收集，需要为每个资源创建诊断设置才能采集。这是 Azure 数据面日志的主要来源，等同于 AWS 的 CloudTrail 数据事件：

| 资源类型 | 关键日志类别 | 安全意义 |
|---|---|---|
| Azure Key Vault | `AuditEvent`（秘钥访问、证书操作、秘钥管理） | **高**，检测秘密和密钥被异常访问 |
| Azure Storage | `StorageRead`、`StorageWrite`、`StorageDelete`（Blob/Queue/Table 操作） | 高，检测数据泄露和越权访问 |
| Azure SQL / PaaS DB | `SQLSecurityAuditEvents` | 高，数据库访问审计 |
| Azure Function App / App Service | `FunctionAppLogs` | 中，应用层行为记录 |

> **与 AWS 最大的不同**：Resource Logs 的覆盖范围（哪些资源、哪些类别）非常碎片化，每种资源类型的日志类别定义各不相同，**不同资源的同名类别（如 `audit`）实际内容也可能截然不同**。安全厂商在针对特定资源开启日志时，必须逐一核查该资源类型的实际 Schema。

### 1.3 日志存储与查询方式

| 存储/查询方式 | 保留时长 | 支持的日志类型 | 查询能力 | 计费 |
|---|---|---|---|---|
| **Azure Portal 内置查询**（Activity Log 门户） | **90 天**（固定，不可延长） | 仅 Activity Log | 多条件过滤，不支持跨订阅查询 | 免费 |
| **Diagnostic Settings → Storage Account** | 由 Blob 生命周期策略决定，理论无限 | Activity Log + Resource Logs | 需配合外部工具或下载后处理 | Storage 存储费；诊断设置本身免费 |
| **Diagnostic Settings → Log Analytics Workspace** | 默认 30 天，可配置 30–730 天（Basic 表最长 12 年） | Activity Log + Resource Logs | **KQL 查询**，支持跨订阅聚合（需配置）；Microsoft Sentinel 基于此 | 按数据摄入量计费（约 $2.30/GB，Commitment Tier 可降至 $1/GB+） |
| **Diagnostic Settings → Event Hub** | Event Hub 消息保留期：Standard 1–7 天，Premium 最长 90 天 | Activity Log + Resource Logs | 流式消费，用于实时 SIEM 接入；不适合长期存储 | Event Hub 吞吐量单元费 |

### 1.4 补充日志（覆盖 Activity Log 盲点）

| 日志类型 | 来源服务 | 覆盖内容 | 安全价值 |
|---|---|---|---|
| NSG Flow Logs / VNet Flow Logs | Azure Network Watcher | 网络层流量元数据（五元组：源/目 IP、端口、协议、字节数、允许/拒绝） | 补充控制面日志的网络盲点，检测横向移动和异常出口流量 |
| Microsoft Defender for Cloud Alerts | Microsoft Defender for Cloud | 基于 Activity Log + Resource Logs 的威胁检测结果 | 开箱即用的告警信号，是 Sentinel 的重要输入 |
| Microsoft Sentinel | Microsoft Sentinel（SIEM/SOAR）| 聚合所有 Azure 日志源的统一分析和告警平台 | Azure 原生 SIEM，基于 Log Analytics Workspace |

> **参考文档**：
> - Activity Log 总览：[Azure Monitor activity log](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log)
> - Activity Log 事件 Schema：[Azure Activity Log event schema](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-schema)
> - Resource Logs 说明：[Resource logs in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs)
> - Azure 安全日志总览：[Azure security logging and auditing](https://learn.microsoft.com/en-us/azure/security/fundamentals/log-audit)

---

## 2. 能力边界

### 2.1 视角边界

**ARM 控制面全覆盖，数据面获取成本高，端侧不可见。**

Azure Activity Log 覆盖 ARM 控制面，Resource Logs 按需覆盖资源数据面，但端侧操作和网络流量内容仍有盲区。

| 可见（日志中有记录） | 不可见（需其他手段补充） |
|---|---|
| 所有 ARM 资源操作（Create/Update/Delete/Action） | 网络层流量内容（需 NSG Flow Logs，仅有元数据无 Payload） |
| RBAC 角色分配与变更（`Microsoft.Authorization/roleAssignments/write`） | VM/容器内部的进程和文件系统行为（需 Azure Monitor Agent / EDR） |
| Azure Policy 策略执行结果（Audit/Deny） | Storage Blob 被访问/下载的内容本身 |
| Key Vault 访问操作（需开启 Resource Logs） | Key Vault 被访问的秘密明文内容 |
| Azure VM Run Command 的调用者身份（见 §2.2 内容边界） | Azure Function 函数执行时的代码逻辑和 Payload |
| 诊断设置自身的创建/修改/删除（Activity Log 记录） | 攻击者在本地端侧的行为（如凭证如何被窃取） |

**跨订阅限制**：Azure Activity Log 默认仅覆盖当前订阅，**不跨订阅聚合**。攻击者从订阅 A 横向移动到订阅 B 时，两侧日志各自独立产生，必须通过 Management Group 级别的诊断设置或手动汇聚才能关联分析。

> **参考文档**：[Azure security logging and auditing](https://learn.microsoft.com/en-us/azure/security/fundamentals/log-audit)

---

### 2.2 内容边界

**只记录操作行为，不记录操作内容本身。**

- **Storage Account**：Activity Log 记录控制面操作（如权限变更、Blob 容器创建），Resource Logs 记录数据面的读写操作（文件名、请求者 IP），但**不保存被访问对象的内容**。
- **Key Vault**：Resource Logs 记录 `SecretGet`、`KeyDecrypt` 等操作（含调用者、Key ID），但**不记录被获取的秘密明文或被解密的数据内容**。
- **Azure VM Run Command / Invoke-AzVMRunCommand**：Activity Log 记录了 Run Command 的发起操作（调用者身份、目标 VM），但**执行的命令内容在 Activity Log 的 `requestBody` 中通常不被完整记录**（需通过 VM 内的 Azure Monitor Agent 收集 OS 层日志才能看到命令内容）。这与 AWS SSM Run Command 的行为类似，是高频攻防盲点。
- **Azure Function / App Service**：Activity Log 记录函数的触发调用（谁触发、从哪个 IP），但**不记录函数的输入参数和执行结果**（需开启 Application Insights 或 Function App Logs）。
- **Azure ARM Templates / Bicep 部署**：`Microsoft.Resources/deployments/write` 事件记录了部署操作，但**部署模板中的敏感参数（如密码、密钥）在 Activity Log 中被编辑**（显示为已编辑）。
- **服务代调**：当 ARM 或 Azure Policy 自动触发操作时，Activity Log 中的 `caller` 字段显示为相关 Azure 服务身份（如 `Azure Policy`、`ARM`），而非发起该操作的原始用户身份——与 AWS CloudFormation 代调 IAM 操作的问题完全一致，是复杂攻击链溯源的难点。

> **参考文档**：[Azure activity log event schema – Administrative category](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-schema)

---

### 2.3 日志完整性与可信度

#### 攻击者能否删除/篡改 Azure 日志？

| 操作 | 是否可行 | 所需权限 | 是否留下痕迹 | 防御手段 |
|---|---|---|---|---|
| 删除诊断设置（停止日志导出） | **可以** | `Microsoft.Insights/diagnosticSettings/delete` | ✅ 会被 Activity Log 记录（`microsoft.insights/diagnosticSettings/delete`） | Azure Policy 禁止删除诊断设置；RBAC 最小权限 |
| 删除 Log Analytics Workspace（清除数据） | **可以** | Owner 或 Log Analytics Contributor + ResourceGroup Delete | ✅ 会被 Activity Log 记录 | Resource Lock（`CanNotDelete`）阻止删除 |
| 通过 Purge API 清除 Log Analytics 中的历史数据 | **可以** | 需要 `Data Purger` 角色（专用角色，默认无人拥有） | ✅ 会在 Workspace 操作日志中记录 | 不授予 Data Purger 角色；监控 Purge 操作告警 |
| 修改/篡改 Log Analytics 中已摄入的数据 | **不可以** | —— | —— | Azure Monitor 是一个追加写入（append-only）数据平台，已摄入的数据不能被修改，但可以通过 Purge API 删除。 |
| 删除 Storage Account 中的日志 Blob | **可以**（若无 Object Lock） | Storage Blob Data Contributor + DeleteBlob | ❌ 不主动告警 | Azure Blob Immutable Storage（WORM）；独立的日志归档订阅 |

**关键结论**：Azure Log Analytics Workspace 本身不可修改，但可以**删除**（若无 Resource Lock）和**Purge**（若有 Data Purger 角色）。Activity Log 在 90 天内由 Azure 平台托管，租户管理员**无法删除** 90 天内的 Portal 内置 Activity Log 历史——但同样也**无法延长**其保留期。

**最佳实践**：
1. 在 Log Analytics Workspace 上设置 `Resource Lock: CanNotDelete`
2. 将 Activity Log 导出到**独立的日志归档订阅**（Log Archive Subscription）中的 Storage Account
3. 对日志归档 Storage Account 开启 **Immutable Blob（WORM）策略**，防止任何人删除历史日志
4. 通过 Azure Policy 强制要求所有订阅的诊断设置不得被删除（Deploy If Not Exists + Deny 策略组合）

> **参考文档**：
> - Log Analytics 安全最佳实践：[Best practices for Azure Monitor Logs](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/best-practices-logs)
> - Azure Monitor 数据安全：[Secure your Azure Monitor deployment](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-security)
> - Blob 不可变存储：[Overview of immutable storage for blob data](https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview)

---

### 2.4 日志保留策略与取证窗口

| 存储方式 | 默认保留 | 最长可保留 | 注意事项 |
|---|---|---|---|
| Activity Log（Portal 内置） | **90 天**，固定不可延长 | 90 天 | 仅 Administrative 等类别；取证窗口有限 |
| Diagnostic Settings → Storage Account | 由 Blob 生命周期策略决定 | 理论上无限 | 需手动配置；存储费用随时间增长 |
| Log Analytics Workspace | 默认 30 天；可配 30–730 天（互动式）；Basic 表最长 12 年 | 12 年（Basic 低成本档） | 按摄入量计费，成本随保留期线性增长 |
| Event Hub（消息保留） | Standard：1–7 天；Premium：最长 90 天 | 90 天（Premium） | 仅用于流式传输，不适合长期存档 |

**高危场景**：许多中小企业订阅从未配置诊断设置，完全依赖 Portal 的 90 天免费 Activity Log，且该免费窗口固定无法延长。若攻击发生后超过 90 天才被发现，Activity Log 历史已永久丢失且无法追溯，同时 Resource Logs（如 Key Vault 访问记录）若未提前开启，相关数据面操作将完全不可见。

> **参考文档**：
> - [Azure Monitor Logs pricing and retention](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs)
> - [How CloudTrail works](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/how-cloudtrail-works.html)（对比参考）

---

### 2.5 其他关键限制

**① Resource Logs 默认关闭，且需为每个资源单独配置——覆盖度与配置投入强相关**

与 AWS CloudTrail 对所有 API 调用统一记录不同，Azure 的 Resource Logs（对应数据面日志）需要**为每个资源实例单独创建诊断设置**。对于拥有数百个 Key Vault、Storage Account 的大型客户，手动配置不可行，需通过 Azure Policy（`DeployIfNotExists`）在订阅或 Management Group 级别批量强制开启。未配置诊断设置的资源，其数据面操作**对安全监控完全不可见**。

**② `caller` 字段在服务代调场景下不代表真实用户**

Azure ARM 中大量操作通过 Azure 服务自动触发（如 Azure Policy 合规修复、ARM Template 部署中的子资源创建），此时 Activity Log 中的 `caller` 字段显示为服务身份（如 `Azure Policy`）而非发起原始操作的用户。追踪真实用户需要向上查找**关联操作的 `correlationId`**——这是 Azure 版本的"服务代调溯源"问题，与 AWS CloudFormation 代调 IAM 的挑战完全对称。

**③ Activity Log 中的写操作会产生"开始"和"完成"两条记录**

如果操作类型为 Write、Delete 或 Action，操作的开始和成功/失败均会被记录在 Administrative 类别中。这意味着每个写操作会产生两条日志（一条 `Accepted/Started`，一条 `Succeeded/Failed`）。在大批量操作或告警规则中，若不做去重处理，会产生大量重复告警。研判引擎在做聚合计数时，应以**最终状态（Succeeded/Failed）**的记录为准，忽略 `Started` 记录。

**④ 管理组（Management Group）级别的诊断设置可能导致跨订阅事件重复**

当在管理组上创建诊断设置时，它会导出该管理组以及其下层级中所有管理组的事件。如果层次结构中的多个管理组都有诊断设置，会收到重复的事件。对于使用 Management Group 统一采集的场景，**必须在最高级别管理组仅设置一次诊断设置**，下层不重复设置，否则采集端将收到大量重复事件。

**⑤ Log Analytics Workspace 中存在两种日志表模式，字段结构不同**

Azure Monitor 的 Resource Logs 在 Log Analytics 中有两种写入模式：`AzureDiagnostics`（所有资源混写到同一张大表）和 `resource-specific`（每种资源类型写入独立的专属表）。两种模式的 Schema 完全不同，且同一租户可能同时存在两种模式的数据。安全厂商在编写 KQL 告警规则时，**必须同时兼容两种模式**，否则会漏掉部分日志。

> **参考文档**：
> - Resource-specific vs AzureDiagnostics 模式：[Resource logs in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs)
> - 管理组诊断设置：[Azure Monitor activity log](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log)

---

## 3. 日志采集流程

### 3.0 章节背景

**采集后台定位**：我们的后台程序部署在独立 SaaS 平台，不进入客户的 Azure 环境。当客户接入订阅后，后台通过以下方式主动拉取日志：

- **Pull（存储拉取）**：后台定期扫描客户 Storage Account 中的日志 Blob 文件，增量下载解析
- **Push-Pull（流式消费）**：客户配置诊断设置将日志推送到 Event Hub，后台作为 Event Hub Consumer 消费消息

**准实时目标**：5–15 分钟端到端延迟（事件发生 → 后台可消费）。

---

### 3.1 授权模型

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

### 3.2 采集方案总览与选型建议

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

### 3.3 方案一：诊断设置 → Event Hub（主流方案）

这是微软官方推荐的 SIEM 集成方式，主流安全厂商（Splunk、QRadar、ArcSight）均原生支持此路径。

#### 3.3.1 整体链路

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

#### 3.3.2 客户需要配置的内容

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

> ⚠️ 重要：Activity Log 的诊断设置**只能在订阅级别配置**，不能在资源组级别配置。若客户有多个订阅，需要为**每个订阅单独配置**，或在 Management Group 级别统一配置（会有重复事件问题，见 §2.5⑤）。

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

**第 3 步：在 SaaS 平台填写接入信息**

| 字段 | 内容（模型 A）| 内容（模型 B）|
|---|---|---|
| 租户 ID（Tenant ID） | 客户的 Entra 租户 ID | 同左 |
| 服务主体 App ID | 多租户应用在客户租户的 Object ID | 客户注册的应用 App ID |
| 客户端密钥 | 无需（使用 Federated Credential） | 客户提供的 Client Secret |
| Event Hub Namespace | 客户创建的 Namespace 名称 | 同左 |
| Event Hub 名称（Activity Log） | `activity-logs` | 同左 |
| 订阅 ID | 客户订阅 ID | 同左 |

#### 3.3.3 返回格式

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

#### 3.3.4 关键字段

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

#### 3.3.5 成本说明

客户侧新增的主要费用：

| 费用项 | 计费方式 | 估算（中型客户，单订阅，管理事件） |
|---|---|---|
| Event Hub Standard TU | 约 $0.015/小时/TU（约 $11/月/TU） | 1 TU 通常可满足中型客户，约 **$11–22/月** |
| Event Hub 消息费 | 前 1000 万条/月免费；超出约 $0.028/百万条 | 中型客户约百万条/月，**可忽略不计** |
| Diagnostic Settings 本身 | **免费** | $0 |
| Storage（若同时配置方案二存档） | ~$0.018/GB/月（冷存储更低） | 视保留时长和日志量 |

#### 3.3.6 踩坑与注意事项

**① Activity Log 诊断设置只能在订阅级别配置**

Activity Log 的诊断设置**只能在订阅级别配置**，不能在资源组级别配置。若客户有多个订阅，需要为每个订阅单独配置，或在 Management Group 级别统一配置（会有重复事件问题，见 §2.5④）。初次接入时容易遗漏部分订阅，导致该订阅的 Activity Log 完全缺失。

**② Activity Log 诊断设置写操作会产生两条记录，需去重**

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

### 3.4 方案二：诊断设置 → Storage Account（存档方案）

此方案将 Activity Log 和 Resource Logs 归档到 Blob Storage，后台定时扫描增量文件。配置最简单，延迟较高，适合对实时性要求不高的场景，也常作为方案一的兜底备份。

**与方案一的核心区别**：

| 对比项 | 方案一（Event Hub） | 方案二（Storage Account） |
|---|---|---|
| 延迟 | 2–8 分钟 | 10–30 分钟（文件按小时切分） |
| 客户配置步骤 | 约 4 步（需建 Event Hub） | **约 3 步（仅需 Storage Account）** |
| 消息积压丢失 | 消息超期（1–7 天）永久丢失，**无兜底** | **Storage 文件持久保存，无丢失风险** |
| 采集侧过滤 | ❌ 无，全量推送 | ❌ 无，全量存储 |
| 成本 | Event Hub TU 费 | 仅 Storage 存储费，**更低** |

#### 3.4.1 客户需要配置的内容

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

#### 3.4.2 后台的增量扫描逻辑

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

#### 3.4.3 成本说明

| 费用项 | 计费方式 | 估算 |
|---|---|---|
| Storage 存储费 | ~$0.018/GB/月（Cool 层）| 中型客户 Activity Log 约 1–5 GB/月，约 **$0.02–$0.10/月** |
| Blob 读取请求 | $0.004/10,000 次 | 可忽略不计 |
| Diagnostic Settings | **免费** | $0 |

---

### 3.5 方案三：方案一 + 方案二（双轨方案）

同时配置 Event Hub（实时）和 Storage Account（存档），Event Hub 提供准实时消费，Storage Account 提供完整的历史兜底。

这是**可靠性要求最高场景下的推荐配置**，原因在于：
- Event Hub 消息超期丢失时，Storage Account 仍有完整记录，后台可从 Storage 补偿扫描
- 适合有合规要求（如需留存日志 1 年以上）的客户
- 客户侧需要为同一诊断设置同时指定两个目标（Azure 支持，每个诊断设置最多 5 个目标）

---

### 3.6 方案对比与选型建议

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

> **参考文档**：
> - [Stream Azure monitoring data to an event hub](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs)
> - [Create diagnostic settings in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/create-diagnostic-settings)
> - [Azure Event Hubs features](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features)
> - [Azure Event Hubs scalability guide](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability)

---

## 4. 日志示例

以下示例均基于 Azure 官方文档中的 Schema 构造，标注触发方式和研判意义。

### 4.1 高权限 RBAC 角色分配（Administrative 类别）

**触发方式**：在 Azure Portal 或通过 CLI/API 为用户/服务主体分配 Owner 角色

```json
{
  "records": [{
    "time": "2024-03-15T09:12:30.123Z",
    "resourceId": "/subscriptions/aaaa-bbbb/resourceGroups/prod-rg/providers/Microsoft.Authorization/roleAssignments/yyyy-zzzz",
    "operationName": "Microsoft.Authorization/roleAssignments/write",
    "category": "Administrative",
    "resultType": "Success",
    "resultSignature": "Created",
    "durationMs": 253,
    "callerIpAddress": "203.0.113.45",
    "correlationId": "cccc-dddd-eeee-ffff",
    "identity": {
      "authorization": {
        "scope": "/subscriptions/aaaa-bbbb/resourceGroups/prod-rg",
        "action": "Microsoft.Authorization/roleAssignments/write",
        "evidence": {
          "role": "Owner",
          "roleAssignmentScope": "/subscriptions/aaaa-bbbb",
          "roleDefinitionId": "8e3af657-a8ff-443c-a75c-2fe8c4bcb635",
          "principalId": "user-object-id-1234",
          "principalType": "User"
        }
      },
      "claims": {
        "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn": "attacker@contoso.com",
        "http://schemas.microsoft.com/identity/claims/tenantid": "tenant-id-xxxx"
      }
    },
    "level": "Informational",
    "operationId": "op-id-xxxx",
    "subscriptionId": "aaaa-bbbb",
    "tenantId": "tenant-id-xxxx",
    "properties": {
      "requestbody": "{\"properties\":{\"roleDefinitionId\":\"/subscriptions/aaaa-bbbb/providers/Microsoft.Authorization/roleDefinitions/8e3af657-a8ff-443c-a75c-2fe8c4bcb635\",\"principalId\":\"new-user-id-5678\"}}"
    }
  }]
}
```

**研判意义**：`operationName = Microsoft.Authorization/roleAssignments/write` + `roleDefinitionId = 8e3af657-a8ff-443c-a75c-2fe8c4bcb635`（Owner 角色的固定 ID）是**最高优先级的提权告警信号**，等同于 AWS 的 `AttachRolePolicy + AdministratorAccess`。`callerIpAddress` 和 `identity.claims.upn` 提供溯源线索，`properties.requestbody` 包含被授权的新用户 Object ID。

---

### 4.2 诊断设置被删除（日志采集中断）

**触发方式**：攻击者删除 Activity Log 诊断设置，使后续操作不再被导出到 SIEM

```json
{
  "records": [{
    "time": "2024-03-15T10:05:44.000Z",
    "resourceId": "/subscriptions/aaaa-bbbb/providers/microsoft.insights/diagnosticSettings/activity-log-to-eventhub",
    "operationName": "microsoft.insights/diagnosticSettings/delete",
    "category": "Administrative",
    "resultType": "Success",
    "resultSignature": "OK",
    "callerIpAddress": "198.51.100.23",
    "correlationId": "delete-corr-id",
    "identity": {
      "claims": {
        "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn": "admin@contoso.com"
      }
    },
    "level": "Informational",
    "subscriptionId": "aaaa-bbbb",
    "tenantId": "tenant-id-xxxx"
  }]
}
```

**研判意义**：这是 Azure 版本的"关闭 CloudTrail"操作。此操作之后，Activity Log 不再被导出到 SIEM，攻击者的后续操作将不产生任何后台可见的记录。**必须设置最高优先级实时告警**，且由于这条日志本身的产生依赖于该诊断设置在删除前仍在运行（删除操作本身会被记录），这通常是"最后被记录的高危操作"。

---

### 4.3 Key Vault 密钥异常访问（Resource Log）

**触发方式**：攻击者获得 Key Vault 访问权后批量获取密钥（需提前为 Key Vault 开启 Resource Logs）

```json
{
  "time": "2024-03-15T13:30:00.000Z",
  "resourceId": "/subscriptions/aaaa/resourceGroups/prod-rg/providers/Microsoft.KeyVault/vaults/prod-keyvault",
  "operationName": "SecretGet",
  "category": "AuditEvent",
  "resultType": "Success",
  "resultSignature": "OK",
  "durationMs": 24,
  "callerIpAddress": "203.0.113.88",
  "correlationId": "kv-corr-id",
  "identity": {
    "claim": {
      "oid": "attacker-sp-object-id",
      "appid": "attacker-app-id",
      "upn": ""
    }
  },
  "properties": {
    "requestUri": "https://prod-keyvault.vault.azure.net/secrets/db-connection-string/?api-version=7.4",
    "id": "https://prod-keyvault.vault.azure.net/secrets/db-connection-string/version-id",
    "httpStatusCode": 200,
    "clientInfo": "azsdk-python/4.7.0",
    "isCallerIPTrusted": false
  }
}
```

**研判意义**：`operationName: SecretGet` + `isCallerIPTrusted: false` + 短时间内大量此类事件（同一 `oid`/`appid` 对多个秘密连续操作）是数据窃取的核心信号。注意：Key Vault 的 Resource Log 默认关闭，**必须在接入客户时检查并开启**，否则此类关键数据访问对安全监控完全不可见。

---

### 4.4 跨订阅横向移动（RBAC 角色分配，不同 subscriptionId）

**触发方式**：攻击者利用已有的订阅 A 权限，在订阅 B 中为自己分配新角色（横向移动到生产订阅）

```json
{
  "records": [{
    "time": "2024-03-15T14:45:00.000Z",
    "resourceId": "/subscriptions/prod-sub-bbbb/resourceGroups/prod-critical-rg/providers/Microsoft.Authorization/roleAssignments/new-ra-id",
    "operationName": "Microsoft.Authorization/roleAssignments/write",
    "category": "Administrative",
    "resultType": "Success",
    "callerIpAddress": "198.51.100.99",
    "correlationId": "lateral-move-corr-id",
    "identity": {
      "claims": {
        "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn": "compromised-user@contoso.com",
        "http://schemas.microsoft.com/identity/claims/tenantid": "tenant-id-xxxx"
      }
    },
    "subscriptionId": "prod-sub-bbbb",
    "tenantId": "tenant-id-xxxx",
    "properties": {
      "requestbody": "{\"properties\":{\"roleDefinitionId\":\"/subscriptions/prod-sub-bbbb/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c\",\"principalId\":\"attacker-object-id\"}}"
    }
  }]
}
```

**研判意义**：`roleDefinitionId: b24988ac-6180-42a0-ab88-20f7382dd24c` 是 **Contributor 角色**的固定 ID，在 `prod-sub-bbbb`（生产订阅）下被授予。这条日志仅出现在生产订阅的 Activity Log 中，而攻击者在开发订阅的初始入侵活动出现在另一个订阅的日志中。**若两个订阅未同时接入，攻击链起点完全不可见**。

---

## 5. 影响基于日志进行告警研判的关键特性

### 5.1 日志规模与客户等级的关联

Azure 的日志量分布与 AWS 有一个显著差异：**Activity Log 量级天然比 CloudTrail 管理事件更低**，因为 Activity Log 只覆盖通过 ARM 的写操作和重要读操作，而 CloudTrail 记录几乎所有 API 调用（含大量 Describe/List 等只读操作）。

#### 单订阅日志量参考

客户通常只接入核心业务订阅，单订阅日志量是规划容量的基本单位：

| 订阅业务类型 | 订阅特征 | Activity Log 量（日/条，单订阅） | Resource Logs 量（日/条，按需开启） |
|---|---|---|---|
| 轻量运维订阅 | 少量 VM/DB，人工操作为主 | **数千 ~ 2 万** | Key Vault 访问：数百 ~ 数千条/天 |
| 标准业务订阅 | AKS + SQL + Storage，自动化 CI/CD | **2 万 ~ 20 万** | 开启 Storage + Key Vault 后：数万 ~ 百万条/天 |
| 高频自动化订阅 | ACI/Function App 密集，Policy 合规修复 | **20 万 ~ 200 万** | Storage 数据事件：**亿级/天**（需谨慎开启） |
| 大型平台订阅 | 数百资源，多团队协作，ARM 部署频繁 | **200 万以上** | 多资源类型叠加后量级极大，需按资源选择性开启 |

#### 按客户规模的整体分层

| 客户规模 | 总订阅数 | 典型接入订阅数 | 后台汇聚量（Activity Log / 日） | 关键研判挑战 |
|---|---|---|---|---|
| 小型企业 | 1–5 个 | 1–3 个 | 数万 ~ 数十万 | 诊断设置常未配置；Resource Logs 基本未开启 |
| 中型企业 | 5–50 个 | 3–15 个核心订阅 | 百万 ~ 千万 | 跨订阅关联困难；Resource Log 覆盖参差不齐 |
| 大型企业/集团 | 50+ 个（含 Management Group） | 10–30 个 | 千万 ~ 亿级 | Management Group 重复事件问题；Event Hub TU 规划；Resource Logs 成本不可控 |

---

### 5.2 日志延迟

| 采集方案 | 各阶段延迟拆解 | 端到端延迟 | 是否满足准实时（5–15 分钟） |
|---|---|---|---|
| **方案一**（诊断设置→Event Hub） | 诊断设置推送（秒级）+ Event Hub 消费（秒级）+ 少量处理 | **2–8 分钟** | ✅ 满足 |
| **方案二**（诊断设置→Storage→轮询） | 诊断设置写入 Storage（约 5 分钟）+ 后台轮询间隔（5–15 分钟）| **10–30 分钟** | ⚠️ 边界情况 |

**注意**：Activity Log 的诊断设置延迟通常在秒级到数分钟，但并不提供 SLA 保证。大规模操作（如批量资源部署）期间可能出现更高延迟。

**对比 AWS CloudTrail**：Azure Activity Log 的端到端延迟（方案一：2–8 分钟）与 AWS CloudTrail（~5 分钟）处于同一量级，均可支持准实时告警。两者相比 O365 UAL（60–90 分钟）都具备显著的实时性优势。

**对研判引擎的要求**：同样需要以 `time`（即 `eventTimestamp`）为事件时间基准，配置 ≥ 15 分钟的乱序容忍窗口（Late Data Tolerance）。写操作产生的"开始"和"完成"两条记录可能在不同的 Event Hub 消息批次中到达，需在研判引擎层面基于 `correlationId` 做关联和去重。

> **参考文档**：[Azure Monitor activity log – Diagnostic settings latency](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log)

---

### 5.3 是否支持流式拉取

| 采集方案 | 数据流动模式 | 处理粒度 | 对关联分析的影响 |
|---|---|---|---|
| **方案一**（Event Hub） | **Push（近流式，长连接推送）** | 每批消息（1 批可含多条事件） | 接近流式，但批量大小取决于 Event Hub 配置（默认 256KB/批）；延迟极低，适合实时关联 |
| **方案二**（Storage 轮询） | **定时 Pull（批量）** | 每小时一个文件（含该小时所有事件） | 批量处理；文件按小时切分，关联窗口至少需覆盖 1 小时 + 延迟缓冲 |

**对关联告警的具体影响**：

与 AWS 相比，Azure Activity Log 的写操作会产生"开始"和"完成"两条关联记录，研判引擎在做攻击链还原时，需要理解这种"双记录"模式：
- 如果看到 `roleAssignments/write` 的 `Started` 记录但没有后续的 `Succeeded` 记录，可能意味着操作失败（被 Azure Policy 拒绝）或日志延迟——这两种情况的安全含义完全不同。
- **推荐实践**：对高危操作（如 RBAC 变更、诊断设置删除）仅对 `resultType = Success/Succeeded` 的最终状态记录触发告警，避免 `Started` 记录产生误报。

---

### 5.4 是否支持过滤

#### 层 1：诊断设置级别过滤（数据产生侧，三套方案共用）

Azure Diagnostic Settings 支持在**数据路由侧**按日志类别过滤，不需要的类别不会被路由到目标（不产生 Event Hub 消息费，也不占用 Storage 空间）：

| 过滤维度 | 能力 | 典型用法 |
|---|---|---|
| **日志类别（Category）** | 按 Activity Log 的 Administrative/Security/Policy/ServiceHealth 等类别选择 | 仅路由 Administrative 和 Security，过滤掉 ServiceHealth 和 Autoscale 噪音 |
| **Resource Logs 类别** | 按资源类型的日志类别独立选择（如 Key Vault 的 AuditEvent、Storage 的 StorageRead） | 仅开启对安全有价值的资源类别 |

**局限**：诊断设置的过滤粒度是"类别"而非"操作"——**无法在数据产生侧过滤掉某个类别内特定的 `operationName`**（如仅保留 `roleAssignments/write` 而过滤掉 `virtualMachines/read`）。这一点比 AWS CloudTrail 的 Advanced Event Selectors 能力弱，后台需要在消费后自行过滤高噪音操作。

#### 层 2：采集侧过滤（三套方案均无原生支持）

与 O365 Management Activity API 类似，Azure 的三套方案均**无法在采集侧按 `operationName` 字段精确过滤**，后台必须全量消费后自行解析和过滤。这是与 AWS EventBridge（方案二，支持精确规则过滤）的重要能力差异。

> **参考文档**：[Diagnostic settings in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings)

---

### 5.5 限流策略

| 接口 / 服务 | 所属方案 | 限制 | 对后台的影响 |
|---|---|---|---|
| Event Hub 消费（AMQP） | 方案一 | 受吞吐量单元（TU）限制：1 TU = 2MB/s 出流量 | 大量日志时需增加 TU；超限会导致消费降速但不丢消息（消息仍在 Event Hub 中） |
| Event Hub 分区 | 方案一 | 同一 Consumer Group 内，每个分区同时只能被**一个 Consumer 实例**消费 | Worker 数量上限 = 分区数；超出的 Worker 空闲。分区数在创建时固定（Standard/Premium 可扩展） |
| Storage Blob 读取 | 方案二/三 | 存储账户级别，标准存储账号无硬性 TPS 限制，但超高并发读可能触发 503 | 多订阅并发扫描时需注意，建议实现指数退避重试 |
| Activity Log REST API（`az monitor activity-log list`） | 不建议用于采集 | **每次最多返回 1000 条**，查询窗口最长 90 天 | 仅用于人工调查，**禁止作为后台批量采集接口**（对应 AWS 的 LookupEvents 限制） |

**Event Hub 的 Auto-inflate（自动扩容 TU）**：标准层支持自动扩容，可在 TU 使用率超阈值时自动增加，有效避免手动管理吞吐量。**强烈建议开启**，并设置合理的最大 TU 上限。

---

### 5.6 日志时序一致性

#### 各方案的时间字段语义

| 方案 | 正确时间基准（应使用） | 易混淆的"假时间" | 偏差量 |
|---|---|---|---|
| 方案一（Event Hub 消息） | `records[].time`（JSON 内层，即 `eventTimestamp`） | Event Hub 消息的 `EnqueuedTimeUtc`（消息入队时间） | 约 秒级–数分钟 |
| 方案二（Storage Blob） | `records[].time`（JSON 内层） | Blob 文件名中的小时时间戳（文件切分时间） | 约 5–10 分钟 |

**写操作的双记录时序**：

Activity Log 中的写操作会先产生一条 `resultType: Start/Accepted` 记录，再产生一条 `resultType: Succeeded/Failed` 记录，**两条记录的 `time` 字段不同**（相差等于实际操作执行时间，通常几百毫秒到数秒）。研判引擎在关联这两条记录时，应以 `correlationId` 为唯一键，而非以 `time` 字段判断先后。

**对关联规则的影响**：

攻击链中的事件可能分散在多条 Event Hub 消息批次或多个 Storage 文件中先后到达后台。研判引擎需以 `time`（即 `eventTimestamp`）为事件时间基准，而非消息到达时间，并设置 **≥ 15 分钟的乱序容忍窗口**，防止因事件乱序导致告警链断裂。

---

### 5.7 多订阅日志汇聚

#### 跨订阅横向移动的可见性挑战

Azure 的多订阅架构是大型企业的常见模式（开发订阅、测试订阅、生产订阅各自独立）。攻击者从订阅 A 横向移动到订阅 B 时：

- **订阅 A** 的 Activity Log 记录了"用户对订阅 B 中的资源执行了 RBAC 变更"（`resourceId` 中包含 `subscriptions/prod-sub-bbbb`）
- **订阅 B** 的 Activity Log 同样记录了该操作

两侧都有记录，但两侧的日志分别存储在各自订阅的诊断设置目标中，**若两个订阅未同时接入后台，攻击链将无法完整还原**。

识别跨订阅操作的关键：**`subscriptionId` 字段与 `resourceId` 中的订阅段不一致**，即一个订阅的身份在另一个订阅中执行操作——这是识别跨订阅横向移动的核心特征字段。

#### 各方案的多订阅汇聚方式

| 采集方案 | 多订阅汇聚方式 | 后台工作量 | 注意事项 |
|---|---|---|---|
| 方案一（Event Hub） | 每个订阅独立 Event Hub，后台维护多个连接；或使用同一 Event Hub Namespace 下多个 Event Hub（节省管理成本） | 中等（每订阅一套连接配置） | Management Group 级别诊断设置可跨多订阅，但注意重复事件问题（见 §2.5④） |
| 方案二（Storage） | 可将多个订阅的 Activity Log 路由到**同一个 Storage Account**（不同路径），后台统一扫描 | 低（路径自动隔离） | 同一 Storage Account 跨订阅日志路由需要跨订阅权限配置，略复杂 |
| 方案三（双轨） | 同上，两种方式组合 | 较高 | 同上 |

**给客户的接入建议**：

与 AWS 类似，建议客户同时接入**生产订阅**和 **CI/CD 订阅（DevOps 环境）**。Azure DevOps 或 GitHub Actions 使用的服务主体往往被授予了跨订阅的权限，是供应链攻击进入生产环境的高频路径，且其操作记录分布在独立的 CI/CD 订阅中。若仅接入生产订阅，攻击链起点完全不可见。

> **参考文档**：
> - [Management Group diagnostic settings](https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log)
> - [Azure Monitor Logs best practices – Access control](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/best-practices-logs)

---

## 6. 安全厂商经验项总结

### 6.1 案例一：APT29 通过 Azure 服务主体和 RBAC 滥用实施持久化横向移动（CISA，2021）

**事件背景**：

CISA 于 2021 年 1 月发布技术警报 AA21-008A，详细描述了高级持续性威胁行为者（归因为 APT29/NOBELIUM）在攻陷本地 AD 后，向云端 Azure/M365 横向移动的完整路径。这是迄今为止官方记录最完整的针对 Azure ARM 控制面的攻击案例，其核心路径完全可被 Activity Log 覆盖。

**攻击链（ARM 控制面可见部分）**：

```
T+0  通过本地 Active Directory 获取高权限（本地入侵，不在 Azure 日志范围）
T+1  应用程序注册及权限申请（发生在 Entra 层，见 Entra 专项调研）
T+2  在目标订阅中为攻击者控制的服务主体分配 Owner 角色
     → Activity Log：operationName = Microsoft.Authorization/roleAssignments/write
     → 关键字段：identity.claims.upn（原始操作者）、properties.requestbody（被授权的 principalId）
T+3  使用服务主体在目标订阅中创建新 VM、修改网络规则（建立持久化）
     → Activity Log：Microsoft.Compute/virtualMachines/write、Microsoft.Network/... 系列操作
T+4  访问 Key Vault，批量获取凭证
     → Key Vault Resource Log（前提：已提前开启）：SecretGet × N
```

**Activity Log 的检测能力分析**：

| 攻击步骤 | Activity Log 可见性 | 检测信号 |
|---|---|---|
| 向服务主体授予 Owner 角色（T+2） | ✅ **完全可见** | `operationName = Microsoft.Authorization/roleAssignments/write` + 被授权身份为服务主体（`principalType: ServicePrincipal`）+ 作用域为整个订阅（`/subscriptions/{id}`） |
| 创建新 VM / 修改网络规则（T+3） | ✅ 可见 | 可疑时间点的大量 Write 操作，且操作者为新注册的服务主体 |
| Key Vault 密钥批量访问（T+4） | ✅ 可见（**前提：已开启 KV Resource Logs**） | `SecretGet` 短时高频，`isCallerIPTrusted: false`，`appid` 非预期应用 |
| 本地 AD 入侵（T+0） | ❌ 不可见 | 发生在 Azure 控制面外 |
| Entra 应用注册及 Token 申请（T+1） | ❌ 不在本报告范围 | 见 Entra 专项调研 |

**关键发现**：

- 整个攻击链中，Activity Log **可见的最早检测点是 T+2**（RBAC 角色分配）——即攻击者已完成 Entra 侧操作之后才进入 ARM 控制面视野。Activity Log 看到的是"提权完成"的结果，而非入侵的全过程，Entra 侧操作（应用注册、权限申请）不在本报告调研范围内。
- **Key Vault Resource Logs 的开启状态直接决定了 T+4 的可见性**。在 CISA 记录的实际案例中，相当比例的受害组织未开启 Key Vault 审计日志，导致凭证被批量窃取后无法事后追踪。

> **参考来源**：
> - CISA 技术警报 AA21-008A：[Detecting Post-Compromise Threat Activity in Microsoft Cloud Environments (CISA, Jan 2021)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-008a)
> - Microsoft 安全博客（NOBELIUM 云端攻击技术）：[NOBELIUM targeting delegated administrative privileges to facilitate broader attacks (Microsoft, Oct 2021)](https://www.microsoft.com/en-us/security/blog/2021/10/25/nobelium-targeting-delegated-administrative-privileges-to-facilitate-broader-attacks/)

---

### 6.2 案例二：供应链攻击路径——通过 Managed Identity 实现订阅间横向移动

**事件背景**：

这一攻击模式基于安全研究人员（包括 m365internals.com 的研究团队和 PwnedLabs 的 RBAC 研究）记录的真实横向移动路径，以及 CISA 在检测云端后渗透活动的技术指南中确认的 Azure 特有攻击向量。此模式在 2022–2025 年的云安全红队报告中被频繁记录。

**核心攻击路径**（基于 Managed Identity 的横向移动）：

```
T+0:00  攻击者获得一台 Azure VM 的访问权（如通过 RCE、SSRF）
T+0:01  调用 VM 的 IMDS 端点获取 Managed Identity 的访问令牌
         GET http://169.254.169.254/metadata/identity/oauth2/token
         → 获得该 VM 的 Managed Identity 令牌
T+0:02  用令牌枚举 Managed Identity 的 RBAC 权限范围
         → 发现其在 prod-subscription 上被授予了 Contributor 角色
T+0:05  使用 Managed Identity 令牌，在 prod-subscription 中创建新的 RBAC 角色分配
         → Azure Activity Log：operationName = Microsoft.Authorization/roleAssignments/write
T+0:10  使用新角色进入 Key Vault，批量获取密钥
         → Key Vault AuditEvent：operationName = SecretGet（需提前开启 Resource Logs）
T+0:20  使用密钥中的数据库凭证，连接生产数据库
         → 此步骤在 Azure 日志中完全不可见（已进入数据层）
```

**Activity Log 可见性分析**：

| 攻击步骤 | Activity Log 可见性 | 检测难点 |
|---|---|---|
| 获取 IMDS 令牌 | ❌ 不可见（IMDS 调用不产生 Azure 日志，发生在 VM 内） | 需 EDR 或 Azure Monitor Agent 监控 VM 内部 |
| 枚举 RBAC 权限 | ⚠️ 部分可见（`Microsoft.Authorization/permissions/action` 在部分情况下有记录，但并非所有枚举操作都记录） | 只读枚举操作在 Activity Log 中覆盖不完整 |
| 新增 RBAC 角色分配 | ✅ **完全可见**（Activity Log: `Microsoft.Authorization/roleAssignments/write`） | 是最关键的可检测点；`caller` 字段为 Managed Identity 的 Object ID |
| Key Vault 密钥访问 | ✅ 可见（前提：**Key Vault 已开启 Resource Logs**） | 默认不开启；未提前配置则此步骤不可见 |
| 数据库连接和查询 | ❌ 不可见（已进入数据面） | 需开启 Azure SQL Audit 或 Defender for SQL |

**关键检测机会和阻断点**：

- **最早可检测点**：`Microsoft.Authorization/roleAssignments/write`（T+0:05），且 `caller` 显示为 Managed Identity 的 Object ID 而非真实用户 UPN——这是识别"机器身份提权"的关键特征，应设置针对 Managed Identity 执行 RBAC 变更的专项告警。
- **与 AWS 的对比**：Managed Identity 等同于 AWS 的 EC2 Instance Role（IAM Role）。AWS 场景下 `AttachRolePolicy` 同样是关键检测点，但 Azure 额外需要关注 `roleAssignments/write` 在**非用户身份**（`principalType != User`）下的操作。
- **供应链角度**：如果 VM 上运行的代码来自受污染的依赖包（类似 AWS 的 UNC6426 nx 供应链攻击），Managed Identity 令牌在 IMDS 端点被调用这一步骤**完全不产生任何 Azure 日志**，Azure 的控制面可见性从 T+0:05 才开始——此时攻击者实际上已经在环境中运行了数分钟到数小时。

**对安全厂商的启示**：

- Key Vault Resource Logs 默认关闭，**接入客户账号时必须检查并强制开启**，否则 Key Vault 密钥被批量获取时完全不可见——而 Key Vault 往往存储了通往整个生产环境的所有"主密钥"。
- Managed Identity 发起的 RBAC 变更是横向移动的高频路径，应对 `Microsoft.Authorization/roleAssignments/write` 且 `identity.claims["principalType"] != User` 的事件设置专项告警规则。
- 从 VM IMDS 调用到 RBAC 变更之间的"侦察阶段"（枚举权限）在 Activity Log 中不可见，这段时间的行为只能通过 VM 内的 EDR 检测——再次说明云日志必须与端侧遥测联合使用才能覆盖完整攻击链。

> **参考来源**：
> - Managed Identity 横向移动技术分析：[Lateral Movement with Managed Identities of Azure VMs (m365internals.com, 2021)](https://m365internals.com/2021/11/30/lateral-movement-with-managed-identities-of-azure-virtual-machines/)
> - Azure RBAC 提权路径研究：[Climbing the Azure RBAC Ladder (PwnedLabs, Dec 2024)](https://blog.pwnedlabs.io/climbing-the-azure-ladder-part-1)
> - CISA 云端后渗透活动检测：[Detecting Post-Compromise Threat Activity in Microsoft Cloud Environments (CISA, Jan 2021)](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-008a)
> - Azure 威胁研究矩阵：[Azure Threat Research Matrix – Privilege Escalation](https://microsoft.github.io/Azure-Threat-Research-Matrix/PrivilegeEscalation/PrivEsc/)

---

### 6.3 核心经验项总结

**① Activity Log 默认不持久化导出——这是所有 Azure 客户最常见的安全漏洞**

Activity Log 虽然默认收集，但**只能在 Portal 中免费查看 90 天**，90 天到期后永久消失。大量中小企业从未配置诊断设置，事后溯源时发现 90 天以外的历史日志完全不存在。安全厂商应将"验证 Activity Log 诊断设置已配置且指向持久存储（Storage Account 或 Log Analytics）"作为接入客户的第一步强制检查项。

**② Event Hub 没有 DLQ，Storage Account 是唯一的日志兜底手段**

与 AWS SQS 不同，Azure Event Hub 没有原生死信队列机制，消息超期后直接永久删除。在 Azure 场景下，**双轨方案（Event Hub + Storage Account）的兜底价值比 AWS 更大**——Storage Account 是唯一可以在 Event Hub 消息超期后补救漏拉日志的手段，不应视为可选项。

**③ 诊断设置被删除是"最后被记录的高危操作"，必须最高优先级实时告警**

`microsoft.insights/diagnosticSettings/delete` 操作会被 Activity Log 本身记录，但此后 Activity Log 将停止被导出到 SIEM，攻击者的后续操作不再可见。这是 Azure 版的"StopLogging"，需走方案一（Event Hub）的实时路径触发告警，不能等待方案二的批量扫描。

**④ Managed Identity 是 Azure 横向移动的核心利用路径，需专项告警规则**

Managed Identity 让 Azure VM/Function/Container 无需管理密钥即可访问其他 Azure 资源，但也创造了一条重要攻击路径：妥协计算资源 → 获取 IMDS 令牌 → 利用过度授权的 Managed Identity 横向移动。这一路径的起点（IMDS 调用）在 Azure 日志中完全不可见，Activity Log 可检测的最早点是 Managed Identity 触发的 RBAC 写操作。需设置针对非人类身份（`principalType: ServicePrincipal / ManagedIdentity`）执行高权限 RBAC 变更的专项告警。

**⑤ Key Vault Resource Logs 是接入时必须检查的关键数据面日志**

Key Vault 往往存储了通往整个生产环境的所有"主密钥"（数据库密码、API Key、证书），但其 Resource Logs 默认关闭。攻击者在获得 RBAC 权限后批量获取 Key Vault 密钥时，若未提前开启 Resource Logs，整个数据窃取过程在 Azure 日志中**完全不可见**。安全厂商应将 Key Vault Resource Logs 的开启状态纳入接入健康检查。

**⑥ ARM 服务代调导致的"真实调用者不可见"是高频溯源难点**

Azure Policy 修复、ARM Template 子资源部署等场景中，Activity Log 的 `caller` 字段显示为 Azure 服务身份，而非真实用户 UPN。溯源时必须通过 `correlationId` 向上追溯最初的用户操作——这与 AWS CloudFormation 代调 IAM 操作的溯源逻辑完全一致，是云原生服务代调场景下的通用溯源挑战。

**⑦ Resource Logs 的两种写入模式（AzureDiagnostics vs resource-specific）是 KQL 规则编写的常见陷阱**

同一租户内的 Resource Logs 可能同时以两种不同 Schema 写入 Log Analytics。告警规则若只覆盖其中一种模式，将漏掉对应资源类型产生的所有日志。建议安全厂商统一采用 `union AzureDiagnostics, <ResourceSpecificTable>` 的查询模式兼容两种 Schema，或在接入时统一将目标资源的 Resource Logs 诊断模式配置为 `resource-specific`（微软推荐的新模式）。

---

## 参考文档索引

| 类别 | 文档名称 | 链接 |
|---|---|---|
| **日志基础** | Azure Monitor activity log | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log |
| | Activity Log event schema | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/activity-log-schema |
| | Resource logs in Azure Monitor | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs |
| | Azure security logging and auditing | https://learn.microsoft.com/en-us/azure/security/fundamentals/log-audit |
| | AzureActivity table reference（Log Analytics） | https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity |
| **安全最佳实践** | Best practices for Azure Monitor Logs | https://learn.microsoft.com/en-us/azure/azure-monitor/logs/best-practices-logs |
| | Secure your Azure Monitor deployment | https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-security |
| | Immutable storage for blobs（WORM） | https://learn.microsoft.com/en-us/azure/storage/blobs/immutable-storage-overview |
| **采集方案** | Diagnostic settings in Azure Monitor | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-settings |
| | Create diagnostic settings | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/create-diagnostic-settings |
| | Stream Azure monitoring data to Event Hub | https://learn.microsoft.com/en-us/azure/azure-monitor/platform/stream-monitoring-data-event-hubs |
| | Enable Diagnostic Settings at scale（Azure Policy）| https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/diagnostics-settings-policies-deployifnotexists |
| | Azure Event Hubs features | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features |
| | Azure Event Hubs quotas and limits | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas |
| | Azure Event Hubs scalability guide | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-scalability |
| | Azure Event Hubs Auto-inflate | https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-auto-inflate |
| **身份与授权** | Managed identities for Azure resources | https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview |
| | Azure RBAC built-in roles | https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles |
| | Azure cloud security benchmark – Logging and Threat Detection | https://learn.microsoft.com/en-us/security/benchmark/azure/mcsb-v2-logging-threat-detection |
| **攻防案例** | CISA – 云端后渗透活动检测 AA21-008A（Jan 2021） | https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-008a |
| | Microsoft – NOBELIUM 云端攻击技术（Oct 2021） | https://www.microsoft.com/en-us/security/blog/2021/10/25/nobelium-targeting-delegated-administrative-privileges-to-facilitate-broader-attacks/ |
| | Managed Identity 横向移动（m365internals, 2021） | https://m365internals.com/2021/11/30/lateral-movement-with-managed-identities-of-azure-virtual-machines/ |
| | Azure RBAC 提权路径（PwnedLabs, Dec 2024） | https://blog.pwnedlabs.io/climbing-the-azure-ladder-part-1 |
| | Azure Threat Research Matrix | https://microsoft.github.io/Azure-Threat-Research-Matrix/PrivilegeEscalation/PrivEsc/ |

---

*本文档所有结论均基于截至 2026 年 3 月的公开官方文档和权威第三方资料。如有产品或定价变更，以微软官方最新文档为准。*