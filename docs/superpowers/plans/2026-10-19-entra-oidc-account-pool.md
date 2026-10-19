# Entra OIDC 共享账号池改造计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. 本文件是改造提案；本轮仅提交计划，没有实施业务代码。P0 验证决定可启用的登录方式，随后再实施对应任务。

**Goal:** 每个请求从共享 GitHub EMU/Copilot 账号池租用可用账号；容量不足时，自动创建 Entra 用户、分配企业应用、等待 Entra provisioning 同步至 GitHub，完成授权与席位验证后入池。

**Architecture:** 沿用 proxy、sso、login、console 四个服务。proxy 负责共享账号调度及持久化补池流程；sso 增加 Entra 身份管理适配；login 负责 GitHub device flow、Entra 登录与登录态管理；console 展示账号、补池和需要人工处理的授权。OIDC 认证关系继续由 GitHub 与 Entra 维护。

**Tech Stack:** Node.js 22、TypeScript、Express 5、React 19、Playwright、现有 SQLite/MySQL 存储抽象；新增 Azure 身份 SDK 与 Key Vault SDK，用于 Graph 管理身份及加密材料存取。

**Spec:** 本文件“已确认需求”“架构与行为”“数据与接口约定”构成设计依据；“实施任务”逐项实现这些要求。身份提供方的真实策略与接口结果通过 P0 记录验证。

## 已确认需求

- GitHub Enterprise Cloud 位于 `github.com/enterprises/<enterprise>`，使用 Enterprise Managed Users（EMU）及 Microsoft Entra OIDC。
- 用户同时拥有 Entra 和 GitHub Enterprise 管理权限；具体应用权限、管理员同意及 setup user 凭据仍需核对。
- 每次请求选择共享池中的可用账号，调用方不固定绑定某个 GitHub 账号。
- 无可用容量时，由 Entra 创建用户并分配对应企业应用，交给 Entra provisioning 服务同步至 GitHub。
- MFA 策略尚未确认；“首次需要 MFA、之后复用登录态”目前只是操作体验。需要验证 Temporary Access Pass（TAP，临时访问密码）是否适用于该租户。
- 本轮只制定计划。后续项目更改按 `AGENTS.md` 验证、提交并推送至 `origin`，保留用户已有修改及历史。

## 全局约束

- 默认保持现有运行模式，新增 `ACCOUNT_MODE=identity|pool`；通过明确配置启用共享池。
- OIDC 企业的 provisioning 由 Entra 执行；不得调用现有 GitHub SCIM 写接口作为 OIDC 账号创建、更新、暂停或删除路径。
- 同一个 EMU 企业不能把本地 SAML 和 Entra OIDC 当作两个同时工作的登录提供方。保留本地模式用于原配置或独立测试企业。
- Entra Graph 管理令牌、Entra OIDC 令牌、GitHub/Copilot OAuth token、浏览器登录态各司其职，不能相互替换。
- 沿用当前源码的 Copilot OAuth 直接调用方式；P0 验证真实可用性后才确定是否需要其他鉴权适配。
- 密码、TAP、OAuth token、浏览器 storage state/完整 profile 不出现在列表、CSV、任务 payload 或普通日志中；密钥和运行材料不进入 Git。
- 初期 sso/login 各运行一个 worker，SQLite 保存其本地任务及元数据。proxy 的池记录、租约、补池任务同时实现 SQLite 与 MySQL；完整扩展 sso/login 多 worker 不属于首版。
- 对流式请求，账号租约持有到上游结束、失败或客户端断开；已经向客户端输出内容后禁止自动换账号重放。
- 请求触发补池必须有总账号数、席位预算、补池并发、任务去重和等待时间上限。
- 现有 OpenAI/Anthropic/Responses 路径及本地模式回归测试继续通过。

## 一、现有代码与差距

以下结论以当前源码为准；仓库较早的项目地图描述了已变更的 token 兑换流程，不能直接作为实现依据。

| 模块 | 现有能力 | 改造缺口 |
| --- | --- | --- |
| `src/sso/src/users/service.ts` | 本地用户、密码策略、直接调用 GitHub SCIM、EMU 导入及席位管理 | 本地用户不等于 Entra 用户；OIDC 模式必须更换 provisioning 路径 |
| `src/sso/src/db/migrations.ts` | `sso_users` 强制要求密码哈希与 salt | 外部托管身份需要独立记录，不伪造本地密码 |
| `src/login/src/auth/HeadlessPlaywrightAuthStrategy.ts` | 已有 Azure 用户名、密码和保持登录提示处理 | 每次创建新的 browser context；缺少可靠登录态复用、TAP、MFA/策略阻断分类 |
| `src/login/src/tasks/runner.ts` | device flow 成功后将 OAuth token 回写 proxy | 强制要求 `ssoPassword`；需要增加无密码登录方式及账号 ID 目标 |
| `src/login/src/tasks/queue.ts`、`recovery.ts` | 并发控制、尝试历史、取消及旧 attempt 回写保护 | 队列 payload 在内存；重启关闭中断任务，不自动恢复完整流程 |
| `src/proxy/src/copilot/copilotAuthManager.ts` | `identity` 查账号、未知 identity 懒初始化、401 失效标记 | 初始化固定使用本地 SSO；没有共享池、主动健康维护及自动补池 |
| `src/proxy/src/accounts/loginAuthorization.ts` | 默认密码/覆盖密码、attempt fencing | Azure 默认密码模式被拒绝；需增加引用凭据和浏览器会话模式 |
| `src/proxy/src/copilot/copilotAuth.ts`、`copilotClient.ts` | OAuth token 直接访问 Copilot API，`/models` 可验证 token | 缓存与鉴权身份需改为实际池账号；不能按调用方缓存不同账号能力 |
| `src/proxy/src/db/*Storage.ts` | SQLite/MySQL、账号初始化租约、授权 attempt 保护 | 需要池账号、请求租约、补池流程和上游账号审计字段 |
| `src/console/src/web/App.tsx` | 用户、账号、登录任务、席位、用量及设置页面 | 增加共享池、Entra 配置、补池进度、需要人工处理状态 |

可复用：GitHub device flow、Azure 页面处理片段、Copilot 席位 API、模型检查、兼容协议转发、attempt ID 保护和内部服务鉴权。无需重写整个项目。

## 二、认证与 provisioning 的可行性边界

### 2.1 OIDC 与账号供应

GitHub 官方明确写明：**“Use of the REST API for SCIM provisioning is not supported with enterprises enabled for OIDC.”**

因此推荐路线为：

1. 项目通过 Microsoft Graph 创建或找到 Entra 用户。
2. 用户获得现有 GitHub EMU OIDC 企业应用的有效 assignment。
3. Entra 的 provisioning job 根据已配置的 mapping 创建 GitHub EMU。
4. 项目观测 provisioning 结果，获取目标身份并核对 GitHub 中的账号。

首版建议直接为用户创建 app role assignment，使用 Graph 返回的有效 service principal ID 和 app role ID。若现有企业按组分配，则将用户加入已经分配给该企业应用的专用组；组分配需要核对租户许可和 provisioning scope，避免使用嵌套组推断访问权限。

Entra provisioning 是异步的。默认等待和观测同步结果；可以额外验证 Graph `provisionOnDemand` 对该 job 的支持，作为加速选项。不能用自定义 GitHub SCIM 写操作兜底。

### 2.2 一次性通行密码/TAP

TAP 是 **Entra 的认证方式**。GitHub 发起 OIDC 登录后，用户在 Entra 页面使用 TAP；它不是直接填写到 GitHub 的本地密码。

TAP 可设为一次或多次使用，有有效期。Graph 能签发 TAP，但需要对应权限、管理员同意，用户还必须处于 Authentication methods policy 的允许范围。是否满足 Conditional Access 的 authentication strength，要看真实策略；要求特定抗钓鱼方式、设备条件或安全信息注册时，仍可能出现交互。

推荐设计保留两条路径：

- **会话路径：** 在受控 login worker 中复用该账号已建立的会话；会话失效时重新授权。
- **TAP 路径：** P0 证明可用后，在需要交互登录时即时签发新 TAP，完成登录后及时清除敏感值；每次新尝试重新评估 pass 的有效性和使用状态。

未通过 TAP 验证时进入 `needs_action`，由管理员完成允许的认证交互，再恢复流程。不能承诺首次人工 MFA 后永久无人值守。人工在自己的外部浏览器完成 device authorization 只能给项目提供 OAuth 授权，**不会自动生成服务端 worker 的浏览器登录态**；要复用 worker 会话，首次交互必须发生在该账号的受控 worker 浏览器内。

微软文档说明 TAP 签发时令牌生命周期受其到期时间约束，但既有会话行为还受 Conditional Access session controls 影响。实现不得写死 TAP、GitHub 会话或 OAuth token 的寿命。

### 2.3 推荐方案及备选

| 方案 | 使用条件 | 取舍 |
| --- | --- | --- |
| Entra provisioning + 会话复用 + 按需 TAP | 现有 OIDC 保持；TAP 满足租户策略 | 推荐；自动化范围由 P0 实测决定 |
| Entra provisioning + 人工完成首次/重新授权 | TAP 不可用或要求必须交互的 MFA | 供应、调度、监控仍自动化；授权会出现 `needs_action` |
| 迁移回本地 SAML | 明确决定更换企业身份架构 | 可复用现有流程较多，但涉及 IAM 迁移；不纳入本次推荐计划 |

应用身份的 Graph `client_credentials` 仅用于目录管理；不能兑换成某个 EMU 的 Copilot OAuth token。GitHub PAT/GitHub App token 也不能默认替代当前 Copilot 用户授权。

## 三、架构与行为

### 3.1 请求链路

```text
调用方请求 + caller identity
  -> 现有 API Key / identity 校验
  -> poolScheduler 原子租用可用池账号
       -> 有账号：校验账号能力 -> 转发 -> 响应/流结束 -> 释放租约
       -> 无账号：enqueue/复用补池任务 + 短等待
                    -> 等到账号：继续本次转发
                    -> 等待超时：返回 503 + Retry-After + pool_capacity_pending
```

`caller identity` 继续用于调用方审计与配额；`poolAccountId` 用于上游 token、账号健康及租约。一个 caller 可以先后使用多个 pool account，一个 pool account 可以服务不同 caller。

账号选择首先过滤 `ready`、健康、未冷却、支持目标模型和请求路径的记录；再按在途请求数和最近分配时间选择，首版不需要复杂权重算法。

### 3.2 补池链路

```text
容量不足 / 库存低于目标
  -> 持久化补池 job，预留一个创建名额
  -> Graph ensureUser（先存稳定 UPN，再执行外部操作）
  -> ensureAppAssignment
  -> Entra provisioning（周期同步，或已验证的 provisionOnDemand）
  -> 观测 GitHub EMU 已建立
  -> GitHub device flow + Entra 会话/TAP/交互登录
  -> GitHub /user 获取实际 login 和不可变用户 ID
  -> 分配并读回确认 Copilot seat
  -> 检查 Copilot /models、代表性请求和目标能力
  -> 标记 ready，交给请求侧使用
```

优先先获得 GitHub 授权并读取 `/user`，再用实际 `ghLogin` 分配席位，避免猜测 UPN 归一化结果。若 P0 发现当前 OAuth 客户端必须先有席位，则先使用 provisioning 结果中的可信目标账号分配席位，再在授权后用 `/user` 复核；无法取得可信目标账号时流程不得进入 `ready`。

OIDC 登录从 GitHub 企业 SSO/device flow 发起，跟随真实重定向。GitHub EMU OIDC 不支持 IdP-initiated authentication，不实现项目自己的 OIDC issuer、JWT 签发或 GitHub OIDC callback。

### 3.3 状态、扩容及故障

- 生命周期：`provisioning -> authorizing -> validating -> ready`；任何认证交互阻断转入 `needs_action`；不可恢复配置错误转入 `failed`；管理停用转入 `disabled`。
- 使用状态：`inFlight`、`cooldownUntil`、`lastHealthCheckAt` 独立于生命周期。账号正在服务一个流不等于身份失效。
- 无账号可租用即触发扩容评估。将已有 ready 账号、在建名额和 pending job 计入目标容量；达到上限时排队或返回容量错误。并发请求共享同一扩容决策，不按每个 HTTP 请求创建一个用户。
- 所有账号只是暂时繁忙时先短等待；持续缺容量时按预算扩容。全局上游限流、租户策略错误、网络故障不通过不断创建账号处理。
- `401`：隔离实际失败的池账号，去重发起重授权；只有确认新授权和能力验证成功后恢复 `ready`。
- `403`：区分席位/应用/策略拒绝与模型或路径能力拒绝；模型不支持只调整能力记录，不将整个账号停用。
- `429`：遵循 `Retry-After`，判断账号级或共享上游冷却，避免立即重试风暴。
- 网络失败、`5xx`：有限次数退避。发生在模型检查等未提交生成请求阶段，可以重新选择账号；已经提交的生成请求默认不自动重放，已输出任何流数据时绝不重放。
- 客户端断开：通过 AbortSignal 取消上游读取及请求，最终释放租约。
- login 重启：丢弃旧 device code 和旧 browser 操作，关闭旧 attempt，然后建立新的 attempt 恢复业务流程；不重放已经消费的一次性 TAP。
- 撤销或暂停 Entra 用户：观测同步结果，停用池账号、阻止新租约并处理现有会话/token。仅回收 Copilot seat 不等于删除 Entra/GitHub 身份。

### 3.4 首版配置建议

以下为初始参数建议，部署时根据延迟、席位预算和 P0 结果调整：

| 参数 | 建议初值/规则 |
| --- | --- |
| `ACCOUNT_MODE` | `identity` 默认；灰度时设 `pool` |
| `POOL_MIN_READY` / `POOL_MAX_ACCOUNTS` | 2 / 10；上限同时核对席位预算，在建账号计入上限 |
| `POOL_MAX_INFLIGHT_PER_ACCOUNT` | 1，后续按真实能力调整 |
| `POOL_EXPANSION_CONCURRENCY` | 1 |
| `POOL_EXPANSION_COOLDOWN_SECONDS` | 60 |
| `POOL_ACQUIRE_WAIT_MS` | 10000；小于部署网关和客户端的请求超时 |
| `POOL_LEASE_TTL_SECONDS` / heartbeat | 120 / 30 秒；长流通过 heartbeat 续租 |
| `POOL_HEALTH_CHECK_INTERVAL_SECONDS` | 900，抖动调度；只读检查为主 |
| provisioning 观测 / 超时 | 30 秒观测一次；初值 60 分钟，之后按实测延迟调整 |
| Entra/TAP | tenant ID、Graph 管理身份、EMU service principal ID、app role ID、可选 group/job ID；TAP 模式由 P0 结果启用 |

首次生产请求能否在本次连接内成功，取决于 ready 库存。冷启动需要 provisioning 和认证时，`503 + Retry-After` 是明确的等待语义；不在现有兼容生成接口上返回虚构的 `202` 生成成功结果。

## 四、数据与接口约定

### 4.1 存储归属

| 服务/新表 | 核心内容 |
| --- | --- |
| sso：`managed_identities` | `managedIdentityId`、provider key、Entra tenant/object ID、UPN、企业应用 assignment、真实 GH ID/login、provisioning 状态、错误与观测时间 |
| login：`login_sessions` | managed identity ID、会话材料引用、provider、材料版本、最后验证时间及 worker/profile 绑定；不存明文 cookie |
| login：扩展 `login_tasks`/尝试表 | target kind/ID、登录方式、凭据/会话引用、attempt ID、device 有效截止时间、待处理原因；不存明文 TAP/密码 |
| proxy：`proxy_pool_accounts` | 不可变 `poolAccountId`、managed identity ID、provider、真实 GH ID/login、加密 OAuth token、token version、当前 attempt、生命周期、健康与能力快照 |
| proxy：`proxy_pool_leases` | lease ID、account ID、caller identity、request ID、持有者、到期时间及 heartbeat；request ID 唯一 |
| proxy：`proxy_pool_jobs` | 补池 job ID、dedupe key、创建名额、阶段、目标 managed identity/account、attempt、重试时间、worker lease、错误 |
| proxy：扩展请求统计/诊断 | `identity` 保持 caller；新增 `poolAccountId`、实际 `ghLogin`、request/lease ID，用于审计和用量归属 |

原 `proxy_accounts` 保留身份固定模式，不重复存储共享池账号 token。原 `sso_users` 保留本地 SAML 用户，不把 Entra 用户导入为拥有默认本地密码的 SSO 用户。

租约与创建名额必须通过数据库事务及条件更新获取；MySQL 通过行锁/条件更新实现，SQLite 通过单写事务实现。续租与释放校验 lease ID 和 owner，旧请求不能释放新的租约。创建外部用户使用预先持久化的 UPN 和 job key，解决 Graph 创建成功但响应丢失后的重复创建。

### 4.2 新文件及接口边界

下列名称均为计划新增的接口，现有服务内部 HTTP 鉴权继续生效。

```ts
// src/packages/shared/src/identity.ts
type IdentityProvider = 'local-saml' | 'entra-oidc';
type LoginMethod = 'session' | 'tap' | 'interactive' | 'password';
interface EnsureManagedIdentityRequest {
  providerKey: string;
  idempotencyKey: string;
  preferredUpn?: string;
}
interface ManagedIdentityDto {
  managedIdentityId: string;
  provider: IdentityProvider;
  providerKey: string;
  upn: string;
  entraObjectId?: string;
  status: 'created' | 'assigned' | 'pending' | 'provisioned' | 'blocked' | 'disabled';
  ghUserId?: string;
  ghLogin?: string;
  failureCode?: string;
}

// src/packages/shared/src/pool.ts
type PoolAccountStatus = 'provisioning' | 'authorizing' | 'validating'
  | 'ready' | 'needs_action' | 'failed' | 'disabled';
interface PoolAcquireRequest {
  callerIdentity: string;
  requestId: string;
  model?: string;
  path: '/v1/models' | '/chat/completions' | '/v1/messages'
    | '/v1/messages/count_tokens' | '/responses';
  waitMs: number;
}
interface PoolLease {
  leaseId: string;
  poolAccountId: string;
  ownerId: string;
  expiresAt: string;
}

// src/packages/shared/src/contracts.ts，新增类型，保留原 CreateLoginTaskRequest
interface CreatePoolLoginTaskRequest {
  poolAccountId: string;
  managedIdentityId: string;
  oauthAttemptId: string;
  loginMethod: LoginMethod;
  credentialRef?: string;
  sessionRef?: string;
}
```

- sso `POST /api/managed-identities/ensure`：接收 `EnsureManagedIdentityRequest`，幂等返回 `ManagedIdentityDto`。
- sso `POST /api/managed-identities/:id/app-assignment`：确保目标应用 assignment；重复执行不重复分配。
- sso `GET /api/managed-identities/:id/provisioning`：返回最新同步状态及可信目标标识；不触发自建 GitHub SCIM。
- sso `POST /internal/managed-identities/:id/login-material`：执行时获取 TAP/其他材料的短期引用；敏感值仅交给经过鉴权的 login worker，不进入 console。
- login `POST /api/pool-login-tasks`：创建 `CreatePoolLoginTaskRequest` 对应任务，按 attempt 去重。
- login `POST /api/pool-login-tasks/:id/interactive`：启动人工授权，返回一次任务的 device verification URI、user code、expiresAt；这些值仅在鉴权后的任务详情展示，使用 `Cache-Control: no-store`。
- proxy `PUT /internal/pool-accounts/:id/oauth-token`：按 attempt ID 和目标身份复核接收 token；过期 attempt 不覆盖新 token。
- proxy `POST /internal/pool-accounts/:id/login-outcome`：回写 `failed` 或 `needs_action` 及分类错误；不能覆盖已成功的新 attempt。
- proxy `GET /api/pool/accounts`、`/api/pool/summary`、`/api/pool/jobs`：脱敏查询。
- proxy `POST /api/pool/replenish`、`/api/pool/accounts/:id/reauthorize`、`/api/pool/accounts/:id/disable`：管理员操作，复用已有批处理预览/执行模式。

服务内部函数约定：

- `poolScheduler.acquire(input: PoolAcquireRequest, signal?: AbortSignal): Promise<PoolLease>`。
- `poolScheduler.release(lease: PoolLease): Promise<void>`；幂等且校验 owner。
- `poolScheduler.heartbeat(lease: PoolLease): Promise<boolean>`；丢失租约时停止新的上游提交。
- `poolReconciler.ensureCapacity(reason: 'request' | 'inventory'): Promise<{ jobId?: string }>`；事务内完成容量计算、名额预留和任务去重。
- `poolAuthManager.getAuth(poolAccountId: string): Promise<CopilotAuthContext>`；只读取账号 token，不用 caller identity 查凭据。
- `poolJobs.runNext(signal?: AbortSignal): Promise<boolean>`；领取一条持久化补池任务，执行到下一个持久化阶段或等待点。

OAuth token 不进入 `PoolLease` DTO；由 proxy 内部鉴权模块读取。`CopilotAuthContext` 的账号缓存键与 caller 审计身份分开。

## 五、实施任务

执行顺序：P0 -> P1 -> P2 -> P3 -> P4 -> P5 -> P6 -> P7 -> P8 -> P9。每个任务完成必要验证后单独提交；实际执行时按 `AGENTS.md` 同步远程。

### P0：真实租户与 OAuth 链路验证，决定自动登录模式

**交付：** 一份真实结果记录，明确 `session/tap/interactive` 哪些可启用，确认账号供应和 Copilot 调用链路。使用测试用户；不修改整租户认证策略。

**文件：** 新增 `guidance/entra-oidc-pool-validation.md`；结果只保留脱敏 ID、策略名称、成功/失败条件和耗时，不记录敏感材料。

- [ ] 核对现有 EMU OIDC service principal、app roles、provisioning scope、mapping 和 job ID，以及创建/分配用户的权限。
- [ ] 核对 Entra Graph 管理身份和管理员同意；用户管理、应用 assignment、TAP、job/日志查询使用各自所需权限，不默认授予整个 Directory 全权限。
- [ ] 核对 TAP Authentication methods policy、目标用户范围、认证强度、sign-in frequency、安全信息注册及设备/IP 条件。
- [ ] 创建一个测试 Entra 用户，直接分配应用或加入既有分配组；确认 Entra provisioning 创建 GitHub EMU，测量延迟和可靠的目标账号识别方法。
- [ ] 通过 GitHub 发起 OIDC，分别验证正常首次登录、TAP 登录、TAP 已使用/到期、MFA/安全信息注册阻断及 worker 重启后的会话复用。
- [ ] 验证项目当前 `GITHUB_OAUTH_CLIENT_ID`、scope、企业 OAuth app 策略和 device flow 能否拿到所需用户 token；读取 `/user`，分配席位，读取 Copilot `/models` 并发送一条代表性非敏感请求。
- [ ] 核对 API worker/login worker 出口 IP、企业 CAP，以及当前内部 Copilot 接口对共享池使用的许可与服务条件；记录真实限制作为上线条件。
- [ ] 仅在 job 支持且有必要时验证 Graph `provisionOnDemand`；记录规则 ID、返回格式和目标身份字段。

**验收：** 至少一名 Entra 创建的 EMU 通过上述完整链路；无法自动登录时必须写明进入人工授权的具体条件。失败则先修正配置/能力判断，不展开全量池实现。

### P1：定义托管身份、池账号和存储迁移

**文件：** 新增 `src/packages/shared/src/identity.ts`、`pool.ts`；修改 `contracts.ts`、`index.ts`；新增 `src/sso/src/db/managedIdentitiesRepo.ts`、`src/login/src/db/sessionsRepo.ts`、`src/proxy/src/db/poolRepo.ts`。修改三个服务的 migrations，proxy 同时修改 `storageTypes.ts`、`sqliteStorage.ts`、`mysqlStorage.ts`、`mysqlMigrations.ts`。

**接口：** 实现第四节 DTO，以及 `tryAcquireLease`、`releaseLease`、`reserveExpansion`、`claimPoolJob`、`savePoolOauthToken` 的存储接口；保存 token 必须按 attempt fencing。

- [ ] 先写 migration、唯一约束和旧数据兼容测试；覆盖旧 `identity` 行到任务 target 的回填。
- [ ] 测试确认缺少池存储时失败，再实现独立池表和托管身份表。
- [ ] 同时实现 SQLite/MySQL 事务、租约续期、创建名额以及 attempt 条件写入。
- [ ] 验证老账号、旧 token、旧登录尝试和导出仍按原模式读取。
- [ ] 运行三个服务数据库测试和 typecheck，提交迁移及 contracts。

**验收：** 重复迁移安全；同一 request/创建名额只成功一次；旧模式行为保持；租约到期可恢复；旧 attempt 无法覆盖新 token。

### P2：Entra 账号创建、assignment 及 provisioning 观测

**文件：** 新增 `src/sso/src/identity/managedIdentityService.ts`、`providerRegistry.ts`、`localSamlProvider.ts`；新增 `src/sso/src/entra/graphClient.ts`、`users.ts`、`appAssignments.ts`、`provisioningObserver.ts`；新增 `src/sso/src/routes/managedIdentitiesApi.ts`；修改 `server.ts`、`config.ts`、`.env.example`、`package.json`。

**接口：** sso 的 ensure、assignment、provisioning 路由；`providerRegistry` 从配置选择 `local-saml/entra-oidc`，Entra 流程不会进入本地 `syncSsoUser`。

- [ ] 先写模拟 Graph HTTP 测试：创建成功、409、创建成功响应丢失、429 `Retry-After`、assignment 已存在、同步等待和 quarantine。
- [ ] 实现 Managed Identity/证书身份的 Graph 调用、token 获取缓存、HTTP 超时和分类重试。
- [ ] 用已持久化的 job ID/UPN 查找或创建 Entra 用户；确认指定企业应用和真实 app role。
- [ ] 观测 provisioning job/日志，按 source object ID 关联结果；对不同 job/schema 使用 P0 已确认的 mapping，保存可信 target 标识。
- [ ] Entra 模式禁用本地 SCIM 写入入口；现有“从 GH 导入并生成本地默认密码”的操作不能用于 Entra 托管身份。
- [ ] 运行 sso 测试及 typecheck，提交身份适配。

**验收：** 网络响应丢失后重试不重复创建用户；同步 pending 不误标成功；异常返回具体阶段；一次 Entra 供应流程中没有自定义 GitHub SCIM 写调用。

### P3：材料存储、会话复用与可选 TAP

**文件：** 新增 `src/login/src/secrets/secretStore.ts`、`keyVaultSecretStore.ts`、`src/login/src/browser/sessionStore.ts`、`src/login/src/clients/identityClient.ts`；新增 `src/sso/src/entra/temporaryAccessPass.ts` 和内部 login-material 路由；修改 shared 脱敏工具、login/sso 配置及环境示例。

**接口：** 登录任务只引用 credential/session，执行时由 login worker 读取材料；每个 managed identity 独立会话。TAP 创建结果使用短期引用转交，已经消费的引用不可重用。

- [ ] 先写过期/单次引用、不同身份隔离、材料版本冲突、无权限读取、日志脱敏测试。
- [ ] 实现 Key Vault 存储与服务身份；若 P0 证明 `storageState` 足够，则保存加密快照；若需要完整 profile，则保存加密 profile 并绑定受控 worker/环境。
- [ ] 实现每账号浏览器会话互斥锁，保存前校验账号绑定，失效时废弃对应版本。
- [ ] P0 通过 TAP 时实现即时创建、使用状态与到期校验、使用后清理；否则配置中禁用 TAP，返回 `needs_action`。
- [ ] 调试 trace/screenshot 默认关闭；启用时只允许受控存储及到期清理，不能假定 trace 已自动脱敏。
- [ ] 运行 secrets/session 测试和 typecheck，提交材料管理。

**验收：** 任务列表/CSV/日志没有敏感值；账号 A 不读取账号 B 会话；不会重试已使用的一次性 TAP；Key Vault 不可用时返回材料获取错误，不伪造登录成功。

### P4：登录策略、人工授权及可靠恢复

**文件：** 新增 `src/login/src/auth/EntraAuthStrategy.ts`、`authResult.ts`、`InteractiveDeviceAuthStrategy.ts`、`strategyFactory.ts`；修改 `HeadlessPlaywrightAuthStrategy.ts`、`types.ts`、`deviceFlow.ts`、`githubUrls.ts`、`tasks/runner.ts`、`queue.ts`、`recovery.ts`、`retry.ts`、`db/tasksRepo.ts`；新增 pool-login-task 路由并调整 package test scripts。

**接口：** `CreatePoolLoginTaskRequest`；结果包含成功、`needs_action` 或分类失败；token 回写使用 `poolAccountId + oauthAttemptId`。

- [ ] 先用本地浏览器 fixture 覆盖既有会话、UPN 登录、TAP、MFA、注册提示、合规设备/CA 拒绝、错误租户、账号选择及真实 GitHub OIDC 回跳。
- [ ] 将 Azure 登录步骤从大策略文件分离，按页面实际状态推进；不要在有会话时仍要求 `ssoPassword`。
- [ ] GitHub device code 从签发时计算绝对 expiresAt；人工授权和 token polling 并行，并支持共同取消/到期，避免等浏览器操作结束后才开始错误计时。
- [ ] device flow 返回 token 后读取 GitHub `/user`，验证目标 GH ID/login，错误账号不得写入池；保存可信 worker 会话。
- [ ] 重启时关闭旧尝试，使用材料引用重建新 attempt；一次性 TAP 在新尝试中重新获取，pending stage 可重新领取。
- [ ] 运行 login 单元测试、浏览器 fixture 和 typecheck，提交登录适配。

**验收：** 自动登录策略阻断不会退化为无限超时；人工授权能恢复流程；外部浏览器授权不会虚报 worker 会话已保存；旧回调/取消/重启无法覆盖新的成功授权。

### P5：共享池调度、租约及请求适配

**文件：** 新增 `src/proxy/src/pool/poolScheduler.ts`、`poolAuthManager.ts`；修改 `copilot/copilotAuth.ts`、`copilotClient.ts`、`routes/compatible.ts`、`auth/identityHeader.ts`（只在需要补充请求上下文时）、`db/requestStatsRepo.ts`、`managementQueries.ts`、diagnostics DTO/store。

**接口：** `poolScheduler.acquire/release/heartbeat`、`poolAuthManager.getAuth`。`ACCOUNT_MODE=identity` 使用原管理器；`pool` 使用池调度。

- [ ] 先写并发租约、模型/路径选择、cooldown、租约抢占和调用方/上游身份分离测试。
- [ ] 实现原子选择及租约，首版每账号最大在途请求 1；缓存按 poolAccountId、token version 和 endpoint 归属。
- [ ] 各兼容路由在整个请求周期持有租约；client close 取消上游，`finally` 幂等释放，长流定期 heartbeat。
- [ ] 请求审计同时记录 caller 与实际账号；`/v1/models` 返回 ready 账号可用模型集合，请求时继续验证被选账号能力。
- [ ] 实现首字节/上游提交边界与重试限制；模型不支持、已提交生成请求、已输出流的处理符合第三节。
- [ ] 运行 proxy 单元测试、MySQL 租约集成测试和兼容路由回归，提交共享调度。

**关键测试形式：**

```ts
test('one account cannot be leased beyond its configured capacity', async () => {
  const results = await Promise.allSettled(Array.from({ length: 20 }, (_, i) =>
    poolScheduler.acquire({
      callerIdentity: `caller-${i}`, requestId: `request-${i}`,
      path: '/chat/completions', model: 'fixture-model', waitMs: 0,
    })));
  assert.equal(results.filter((result) => result.status === 'fulfilled').length, 1);
});
```

测试 fixture 创建一个 ready 账号，能力包含 `fixture-model`，并设置最大在途请求 1；不要调用真实 Graph 或 Copilot。

**验收：** 同一调用方能选择不同账号；多个调用方共享账号；并发不超限；SSE 完成前不释放；客户端断开后无僵尸租约；旧固定身份模式继续通过测试。

### P6：持久化补池、席位确认及健康维护

**文件：** 新增 `src/proxy/src/pool/poolReconciler.ts`、`poolJobs.ts`、`healthMonitor.ts`、`src/proxy/src/clients/identityClient.ts`；修改 `clients/loginClient.ts`、`routes/internalApi.ts`、`config.ts`、`server.ts`；调整 `src/sso/src/copilot/seats.ts` 与 managed identity service 以按已确认 `ghLogin` 管理席位。

**接口：** `ensureCapacity`、`runNext`；请求侧调用相同容量决策，管理页面手动补池也调用该入口。

- [ ] 先写“100 个无容量请求只预留允许的创建名额”、重启接续、stage 响应丢失和达到上限测试。
- [ ] 持久化 saga：账号名额 -> Entra user -> assignment -> provisioning -> OAuth -> seat -> validation -> ready；每次外部操作前后记录阶段及幂等关联。
- [ ] 接收 attempt fenced 的授权回调；把当前源码已有席位读回检查复用于 Entra 身份，保留 `pending_cancellation` 真实日期。
- [ ] `/models` 和代表性能力验证成功后才标 ready；失败进入相应的等待、重试或 `needs_action`，不能仅凭 OAuth token 存在就入池。
- [ ] 启动时恢复未完成 job；后台维持 min ready、探测健康、处理 token 失效/账号停用；API Key/调用方配额验证通过后请求才可触发补池。
- [ ] 当所有账号繁忙、无库存、策略阻断或达到预算时，分别输出明确容量错误；等待超时返回 `503 + Retry-After`，后台 job 继续推进。
- [ ] 运行容量、恢复、席位生命周期和健康测试，提交补池闭环。

**关键验收：** `reserved + existing <= POOL_MAX_ACCOUNTS` 始终成立；重启或外部响应丢失不会重复造号/分配席位；授权后的账号必须经过能力检查；全局 429/策略错误不会触发无限扩容。

### P7：后台页面、设置及运维接口

**文件：** 新增 `src/proxy/src/routes/poolApi.ts`、`src/console/src/web/api/pool.ts`、`api/identity.ts`、`pages/AccountPoolPage.tsx`、`pages/EntraIdentityPage.tsx`、`components/PoolJobDetails.tsx`；修改 `App.tsx`、`LoginTaskDetails.tsx`、server API proxy 及相应浏览器 fixture。

- [ ] 先写池状态、补池详情、无权限管理、设置版本冲突及脱敏渲染测试。
- [ ] 增加账号池页面：ready/在途/冷却/needs_action/在建、实际 GH 账号、健康及使用统计。
- [ ] 增加 Entra 配置校验和连接测试，显示 tenant、应用、job、分配方式及可用登录方式；密钥只显示是否配置。
- [ ] 增加补池 stage/重试、账号停用、重新授权、人工 device code 及待处理原因；复用现有批量操作预览机制。
- [ ] 增加预算、容量、等待时长、worker 并发及 TAP 开关设置；认证策略本身仍在 Entra 管理。
- [ ] 运行 console 单元与浏览器测试、typecheck/build，提交管理界面。

**验收：** 管理员能看出“为何没有可用账号”和补池停在哪个阶段；人工恢复有可执行入口；敏感材料不经过普通详情/导出；错误不会让旧页面失去可用性。

### P8：集成验证、部署和灰度

**文件：** 新增 `src/mock-entra/` fixture 服务、package/tsconfig；新增 `src/proxy/src/pool/pool.integration.test.ts` 与 login Entra browser fixture；修改相关 package test scripts、根 workspaces（若 mock 单列 workspace）、Compose、环境示例、health 脚本；新增 `guidance/entra-oidc-account-pool.md`。

- [ ] mock 实现 Graph user/assignment、延迟 provisioning、TAP 使用/到期和 job quarantine；既有 mock-github 继续测试本地 SCIM。
- [ ] 集成验证：空池 -> 多请求 -> 唯一补池 job -> Entra user -> provisioning -> login -> seat -> validation -> 请求成功。
- [ ] 故障注入：服务重启、Graph 429、已创建响应丢失、TAP 消费后中断、错账号授权、旧 token 回调、席位取消、长流断线与租约过期。
- [ ] 在测试租户运行 2–3 个账号的小池，测量首次供应耗时、授权成功率、容量等待、会话失效和费用。
- [ ] 灰度启用 pool 模式，明确 min/max 和出网配置；回滚切回 identity 模式并暂停补池，不自动删除 Entra 账号/撤销席位。
- [ ] 完成第六节检查，更新部署文档后提交。

**验收：** 真实端到端请求成功且无重复用户；固定身份模式回归通过；pool 模式重启可恢复；自动化范围与 P0 结果一致。

### P9：运行期停用与回收闭环

**文件：** 新增 `src/proxy/src/pool/retirement.ts`；扩展 poolApi、managedIdentityService、console 池操作和部署文档。

- [ ] 先写停止新租约、等待在途流、席位 pending cancellation、Entra assignment 撤销和观测超时测试。
- [ ] 默认执行 disable/drain，回收席位使用现有真实状态检查；自动容量调节不直接硬删除外部身份。
- [ ] 显式退役操作撤销目标应用 assignment/专用组成员资格，等待 Entra deprovisioning，记录仍存在的账号和未完成步骤。
- [ ] 清除项目持有的会话及 token 材料，保留脱敏审计；按配置的保留期处理记录。
- [ ] 检查实际席位账单/取消日期，验证不会把“API 接受取消”当作“当月费用立即消失”，提交回收闭环。

**验收：** 停用后没有新请求选择该账号；不会因池缩容误删租户里其他用户；远程未完成的步骤有明确状态；费用与实际席位读回一致。

## 六、验证命令与上线标准

以下是后续实现时运行的检查，本轮没有运行业务测试，也不声称以上功能已经实现：

```powershell
npm run typecheck:deploy
npm --workspace @ghcp/sso test
npm --workspace @ghcp/login test
npm --workspace @ghcp/login run test:browser
npm --workspace @ghcp/proxy test
npm --workspace @ghcp/proxy run test:mysql
npm --workspace @ghcp/console test
npm --workspace @ghcp/console run test:browser
npm run build:deploy
```

新增目录的测试须加入各 workspace test scripts，不能把测试写到不被现有 glob 执行的位置。MySQL 集成测试使用现有 `mysql:test:up` 环境；新 mock-entra 不携带生产凭据。对跨服务新增路由增加真实 HTTP 集成验证。

上线前必须满足：

1. Entra 供应的一名新 EMU 可以完成 GitHub/Copilot 真实链路。
2. P0 确认的自动登录方式通过失效与恢复测试；若仍需 MFA 交互，产品明确显示 `needs_action`。
3. 请求共享调度没有租约超售、流提前释放、错误账号 token 或 caller 审计丢失。
4. 空池和并发压力只产生预算允许的创建名额，重启后不重复造号。
5. token/会话/TAP 受控存储，列表、导出、日志和 Git 不包含凭据。
6. 老模式测试通过；灰度和回滚可操作；旧固定身份账号不会被池回收逻辑处理。
7. 真实 CAP、OAuth 应用授权及 Copilot 服务/许可条件允许该部署方式；源码内部接口的可用性单独通过真实请求验证。

## 七、建议实施批次

| 批次 | 任务 | 可以评审的成果 |
| --- | --- | --- |
| A：可行性 | P0 | TAP/MFA、官方 provisioning、OAuth/席位真实链路结论 |
| B：供应与授权 | P1–P4 | Entra 创建的账号能受控授权并可靠保存材料 |
| C：共享池闭环 | P5–P6 | 每请求选择账号，无容量时自动供应并入池 |
| D：交付与运维 | P7–P9 | 后台、故障恢复、部署灰度与停用回收 |

首个应执行的任务是 **P0**。它只需要一名测试用户，能较早排除租户策略或 Copilot 鉴权不支持目标流程的情况，避免先完成大规模重构才发现首次授权不可自动化。

## 八、官方依据

核对日期：2026-10-19。

- [GitHub：配置 EMU OIDC](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-iam/configuring-authentication-for-enterprise-managed-users/configuring-oidc-for-enterprise-managed-users)：Entra OIDC 集成、SP-initiated 登录及配置限制。
- [GitHub：配置 EMU SCIM provisioning](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-iam/provisioning-user-accounts-with-scim/configuring-scim-provisioning-for-users)：OIDC 企业不支持自定义 REST SCIM provisioning。
- [Microsoft：GitHub EMU OIDC 自动 provisioning](https://learn.microsoft.com/en-us/entra/identity/saas-apps/github-enterprise-managed-user-oidc-provisioning-tutorial)：应用分配、同步范围和供给日志。
- [Microsoft：Temporary Access Pass](https://learn.microsoft.com/en-us/entra/identity/authentication/howto-authentication-temporary-access-pass)：认证方法策略、单次/多次使用、生命周期及注册限制。
- [Microsoft Graph：创建 TAP](https://learn.microsoft.com/en-us/graph/api/authentication-post-temporaryaccesspassmethods?view=graph-rest-1.0)：API 和授权要求。
- [Microsoft：Authentication strengths](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-strengths)：认证方式必须满足实际 Conditional Access 要求。
- [Microsoft Graph：provisionOnDemand](https://learn.microsoft.com/en-us/graph/api/synchronization-synchronizationjob-provisionondemand?view=graph-rest-1.0)：可选的官方按需同步路径，需要验证目标 job 支持。
- [GitHub：Entra Conditional Access 支持](https://docs.github.com/en/enterprise-cloud@latest/admin/managing-iam/configuring-authentication-for-enterprise-managed-users/about-support-for-your-idps-conditional-access-policy)：GitHub/OAuth 应用的出网 IP 与 CAP 验证。
