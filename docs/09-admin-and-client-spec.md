# 管理后台与 Client 页面权限接口及开发计划

更新日期：2026-11-02。适用范围：个人与企业多租户 AI Token SaaS。

平台管理后台服务于平台运营、财务、支持、安全和运维人员；Client 服务于个人用户和企业客户，企业内部的成员、项目、预算与账单管理归入 Client。两个前端共用 Hono 业务能力，使用独立会话和接口入口；文本、图片和视频执行由 Go Gateway 承担。

本文是待实现的页面与接口设计，沿用已确定的技术、资金和分片边界。已有接口的状态、幂等与持久化语义以 [接口契约](05-api-contracts.md) 为准；本文补充页面路由、权限和业务接口清单。实现阶段将这些设计固化到 OpenAPI，并据此生成客户端。

## 1 实施范围与约定

### 1.1 系统边界

| 入口 | 技术与页面范围 | API 入口 | 业务与数据责任 |
| --- | --- | --- | --- |
| 平台管理后台 | React、Vite、Ant Design；平台内部管理页面 | `/admin/v1` | 经平台权限操作租户、模型、渠道、价格、财务与运营资源 |
| Client | Next.js、shadcn/ui；公开页面、个人与企业门户 | `/public/v1`、`/console/v1` | 客户自助管理自己的资源；公开接口仅返回可公开的信息 |
| 业务后端 | TypeScript、Hono；API 与分配、清算、支付 Worker | 上述业务 API 与私有 `/internal/v1` | 身份、权限、配置、中央钱包、最终计价、正式账本与支付 |
| 推理与媒体执行 | Go Gateway 与各类 Worker | `/v1`、`/gateway/v1` | 在已授权的持久额度内鉴权、预占、执行、计量和恢复 |

`/public/v1` 以及下文未出现在原接口文档中的业务端点属于本次细化的设计接口，不表示已有实现。两个前端均通过业务 API 访问数据，不直接写业务库或运行时库。正常生产推理直接调用 Gateway，不经过 Next.js 或 Hono 逐请求审批。

### 1.2 优先级和阶段

| 优先级 | 交付要求 |
| --- | --- |
| P0 | 对外收费上线前形成完整闭环，包含企业基础权限、准确扣费、文本流式、图片和视频异步、支付对账及故障容量验收 |
| P1 | 企业采购和规模化增强，包括 SSO、SCIM、自定义角色、可配置审批、完整合同与套餐、自动充值、授信和 BYOK |
| P2 | 后续优化，包括成本或延迟智能路由、更多模态、Agent/MCP 和跨区域扩展 |

D0–D7 沿用 [开发路线](06-development-roadmap.md)：D0 工程与契约，D1 身份与控制面，D2 额度与清算，D3 文本，D4 图片视频，D5 支付财务，D6 容量故障，D7 企业扩展。优先级不代表工期；标注多个阶段的页面分阶段补齐，全部 P0 能力完成后才构成收费上线条件。首批客户要求的 P1 采购能力提前安排，并保留其原有依赖和验收。

### 1.3 页面和接口清单读法

- `A01` 等编号标识后台页面，`C01` 等编号标识 Client 页面；列表、详情和编辑可属于同一页面模块。
- 前端路由中的 `{tenant}` 等是动态资源标识，实际路由由对应前端框架实现。
- 后台接口表中 `A = /admin/v1`，`AT = /admin/v1/tenants/{tenant}`。
- Client 接口表中 `C = /console/v1`，`T = /console/v1/tenants/{tenant}`，`P = /public/v1`，`G = /gateway/v1`。
- 接口表中的权限是每个动作的服务端要求。页面可见性只改善交互，不能代替对象、字段和资源范围授权。
- 页面上的按钮、详情字段、导出列和下载入口都按权限裁剪；接口继续独立校验。

## 2 平台管理后台

### 2.1 使用人员与默认角色

平台角色只授予平台人员，不因某用户是企业 Owner 而获得。下表是默认职责集合，实际授权使用具体权限和资源范围，不设置可绕过全部规则的通用业务入口。

| 平台角色 | 默认职责 | 默认限制 |
| --- | --- | --- |
| 运营管理员 Operations | 租户生命周期、模型目录、渠道元数据、销售价格和平台策略 | 凭证写入、财务调整、采购成本及内容日志分别授权 |
| 安全管理员 Security | 平台人员、权限、会话、凭证托管和支持临时授权 | 不自动获得退款、调账或客户内容读取权 |
| 财务 Finance | 支付、账单、总账、退款、更正、对账、采购成本和财务报表 | 不管理模型凭证，不读取客户请求正文 |
| 支持 Support | 获授权租户的资料、请求和任务元数据排障 | 不读取 Key 明文；敏感字段和请求内容需要额外授权 |
| 运维 SRE | Cell、数据库、队列、流量和告警；受控恢复作业 | 不直接改客户余额或修改任务、结算终态 |
| 审计 Auditor | 已授权范围内的操作审计、配置历史和财务记录只读查询 | 导出另行授权，无业务写入权限 |

后台权限统一使用 `platform.*` 语义标识。`.read`、`.write`、`.publish`、`.refund`、`.adjust`、`.export` 等操作分别授权。支持人员的临时权限必须限定租户、资源、动作、原因和有效期；授权依据来自客户授权或既有支持约定，由有权人员确认并留审计，不能接受请求方自报的批准人作为依据。

默认角色按下列矩阵映射到 2.3 的具体权限。表中“管理”仅包含该领域列出的动作，“只读”受分配的资源范围约束；个人安全页面对本人开放。需要额外授权的动作不会因角色名称而自动放行。

| 能力 | Operations | Security | Finance | Support | SRE | Auditor |
| --- | --- | --- | --- | --- | --- | --- |
| 租户资料与启停 | 管理 | 按授权只读 | 必要账单资料只读 | 获授权客户只读 | 按授权只读 | 授权范围只读 |
| 用户身份与全局账户状态 | 只读 | 状态管理 | 必要身份资料只读 | 获授权客户只读 | — | 授权范围只读 |
| 模型和渠道元数据 | 管理 | 凭证相关元数据只读 | 定价所需元数据只读 | 排障范围只读 | 运行健康只读 | 授权范围只读 |
| 上游凭证写入 | — | 管理 | — | — | — | 变更记录只读 |
| 销售价和企业协议价 | 管理 | — | 只读 | 获授权客户当前适用价只读 | — | 授权范围只读 |
| 采购价和成本 | — | — | 管理采购价并读取成本 | — | — | 额外授权后只读 |
| 平台策略和客户 Key 撤销 | 管理 | 额外授权 | — | 只读，撤销需额外授权 | 只读，变更需额外授权 | 授权范围只读 |
| 额度查询和安全回收 | 只读 | — | 查询和安全回收 | 授权客户状态只读 | 只读 | 授权范围只读 |
| 支付、正式账务、退款与更正 | — | — | 按具体资金动作管理 | 仅授权客户的脱敏处理状态 | — | 授权范围只读 |
| 请求与任务排障 | 元数据只读 | 安全案例元数据只读 | 费用与结算关联只读 | 获授权客户元数据只读 | 运行元数据只读 | 授权范围只读 |
| 内容访问 | 额外授权 | 额外授权 | — | 临时授权 | 额外授权 | 额外授权 |
| 支持访问授权和平台人员权限 | — | 管理 | — | — | — | 授权记录只读 |
| 系统健康和受控恢复 | 健康只读 | 安全状态只读 | 财务核对与资金恢复动作 | 处理进度只读 | 健康、告警与已授权恢复动作 | 授权范围只读 |
| 运营设置和公告 | 管理 | — | — | 只读 | 告警相关设置按授权 | 授权范围只读 |
| 审计与导出 | 自身或运营范围审计 | 安全审计 | 财务审计及授权财务导出 | 自身或支持案例范围审计 | 运维范围审计 | 授权范围审计，导出单独授权 |

角色对应的读取接口继续进行字段裁剪。例如支持人员读取订单处理状态不获得支付账号和资金来源明细，采购价写入仍需对应价格发布权限，运维恢复权限不包含退款或调账权限。

### 2.2 页面清单

| 页面 | 前端路由 | 页面内容和主要操作 | 优先级与阶段 |
| --- | --- | --- | --- |
| A01 登录与个人安全 | `/login`、`/account/security` | 登录、MFA、当前会话、设备会话撤销；敏感操作重新验证 | P0，D1 |
| A02 运营概览 | `/overview` | 活跃租户、请求量、成功率、收入成本、告警、清算积压；财务字段单独授权 | P0，D3–D6 |
| A03 租户管理 | `/tenants`、`/tenants/{tenant}` | 个人/企业筛选、资料、状态、联系人、资源摘要；暂停、恢复和操作进度 | P0，D1 |
| A04 用户管理 | `/users`、`/users/{user}` | 用户身份、所属租户、登录状态；按授权停用账户与撤销会话 | P0，D1 |
| A05 模型目录 | `/models`、`/models/{model}` | 能力、参数、计量项、输出上限、协议兼容范围、上下架 | P0，D1–D4 |
| A06 厂商与渠道 | `/providers`、`/channels/{channel}` | 厂商账户、区域、凭证引用、配额、健康、熔断和连接检查 | P0，D1–D4 |
| A07 价格与协议价 | `/pricing`、`/tenants/{tenant}/pricing` | 采购价、公开销售价、企业协议价、草稿、校验、生效版本和历史 | P0，D1–D5；完整合同 P1 |
| A08 路由与平台策略 | `/routing`、`/tenants/{tenant}/policies` | 授权模型和渠道、分层流控、路由和熔断配置；发布和生效进度 | P0，D1–D4 |
| A09 客户资源与 Key | `/tenants/{tenant}/resources` | 项目、应用、服务账号和 Key 元数据；受控撤销与批量撤销 | P0，D1 |
| A10 额度与操作进度 | `/quota`、`/operations/{operation}` | 中央授权、子池投影、占用、封存回收、撤销屏障和未确认目标 | P0，D2 |
| A11 充值与支付订单 | `/orders`、`/orders/{order}` | 支付尝试、真实到账、未知状态、主动查询核实和补单结果 | P0，D5 |
| A12 结算与总账 | `/billing/settlements`、`/billing/ledger` | 请求收费、计价快照、来源与账本关联；原始收费及更正记录 | P0，D2–D5 |
| A13 退款与账务更正 | `/billing/refunds`、`/billing/adjustments` | 充值款退款、消费退款、冲正；原因、证据、金额上限和处理进度 | P0，D5 |
| A14 对账与异常处理 | `/billing/reconciliation`、`/exceptions/{case}` | 支付、来源、预算、授权、结算和回执差异；核对和受控恢复 | P0，D2–D5 |
| A15 请求与媒体任务 | `/requests/{request}`、`/tasks/{task}` | 请求链路、上游尝试、文本交付、任务执行、文件与账务状态 | P0，D3–D4 |
| A16 用量与财务报表 | `/reports/usage`、`/reports/finance` | 按租户、项目、Key、模型、渠道聚合；收入、成本、毛利和导出 | P0，D3–D5 |
| A17 审计与支持授权 | `/audit-logs`、`/support-access` | 操作历史、权限变更、敏感访问、支持临时授权及撤回 | P0，D1–D6 |
| A18 系统健康与告警 | `/system/health`、`/alerts` | Cell 负载、活跃流、排队、未知提交、最老事件、清算与回执延迟 | P0，D2–D6 |
| A19 平台人员与权限 | `/settings/operators`、`/settings/roles` | 平台人员、固定角色、资源授权、MFA 要求和会话撤销 | P0，D1；自定义角色 P1 |
| A20 运营设置 | `/settings/operations`、`/announcements` | 公开说明、公告、通知模板、业务限值和发布历史 | P0，D5；运营 UI 不直接编辑密钥或生产部署配置 |
| A21 合同与商业计划 | `/contracts`、`/plans` | 合同版本、套餐、企业权益、授信审批和履约状态 | P1，D7 |
| A22 审批中心 | `/approvals` | 可配置的价格、退款、权限或合同审批和执行结果 | P1，D7 |

### 2.3 页面权限与 API

以下权限均带 `platform.` 前缀，例如表中的 `tenant.read` 表示 `platform.tenant.read`。同一单元格列出的不同操作按对应权限分别检查；角色名称不能直接作为放行条件。

| 页面 | 读取接口与权限 | 写入接口与权限 | 关键输入和返回 |
| --- | --- | --- | --- |
| A01 | `GET A/sessions/current`、`GET A/me/sessions`、`GET A/me/permissions`：本人 | `POST A/sessions` 登录；`POST A/mfa/challenges/{challenge}:verify` 验证；`DELETE A/sessions/current`、`DELETE A/me/sessions/{session}`：本人 | 返回登录挑战或已建立会话；MFA 通过前不签发完整管理会话 |
| A01 | `GET A/me/mfa`：本人 | `POST A/me/mfa/enrollments`、`POST A/me/mfa/enrollments/{enrollment}:verify`、`DELETE A/me/mfa/factors/{factor}`：本人及近期重新验证 | 认证组件由接入层映射；恢复码和绑定材料只在必要步骤展示 |
| A02 | `GET A/overview`：`overview.read`；财务子项另需 `finance.read` | 无 | 时间范围；返回按权限裁剪的指标、统计口径及 `as_of` |
| A03 | `GET A/tenants`、`GET AT`：`tenant.read` | `PATCH AT`、`POST AT:suspend`、`POST AT:resume`：`tenant.write` | 资料更新带版本；启停返回操作和服务生效状态，不清除合法历史结算 |
| A04 | `GET A/users`、`GET A/users/{user}`：`user.read` | `POST A/users/{user}:suspend`、`POST A/users/{user}:resume`、`POST A/users/{user}/sessions:revoke`：`user.status.write` | 停用前展示受影响租户和个人凭证范围，返回生效操作；企业服务账号单独管理 |
| A05 | `GET A/models`、`GET A/models/{model}`：`model.read` | `POST A/models`、`PATCH A/models/{model}`：`model.write`；`POST A/models/{model}:publish`：`model.publish` | 能力、计量、协议、上界规则版本；发布校验适配器和定价支持 |
| A06 | `GET A/providers`、`GET A/providers/{provider}`、`GET A/channels`、`GET A/channels/{channel}`：`channel.read` | `POST A/providers`、`PATCH A/providers/{provider}`、`POST A/channels`、`PATCH A/channels/{channel}`、`POST A/channels/{channel}:check`：`channel.write` | 返回脱敏凭证引用、配额和健康；检查有界执行，不把测试调用隐藏为免费调用 |
| A06 | 凭证状态随渠道详情返回，永不返回上游密钥 | `POST A/channels/{channel}/credential-versions`：`channel.credentials.write` | 新密钥由服务端写安全存储并返回引用；不进入审计正文或普通数据库字段 |
| A07 | `GET A/price-versions`、`GET AT/pricing`：`pricing.read`；采购成本另需 `cost.read` | `POST A/price-versions`、`POST AT/price-versions`：按类型检查 `pricing.sale.write`、`pricing.cost.write` 或 `pricing.tenant.write`；`POST A/price-versions/{price}:validate`、`POST A/price-versions/{price}:publish`：`pricing.publish` 及对应价格类型写权限 | 定价项、币种、适用范围、生效时间；冻结版本不可覆盖，回滚通过发布新版本完成 |
| A08 | `GET A/routing-policies`、`GET AT/policies`：`policy.read` | `POST A/policy-versions`、`POST AT/policy-versions`：`policy.write`；`POST A/policy-versions/{policy}:publish`：`policy.publish` | 返回版本、影响范围与操作进度；收紧权限时执行撤销屏障 |
| A09 | `GET AT/resources`、`GET AT/api-keys`：`key.read` | `POST AT/api-keys/{key}:revoke`、`POST AT/api-keys:batch-revoke`：`key.revoke` | 仅掩码、归属、策略和状态；返回逐 Key 操作结果，不提供明文恢复 |
| A10 | `GET A/quota-grants`、`GET A/quota-grants/{grant}`：`quota.read`；`GET A/operations`、`GET A/operations/{operation}`：`operation.read` | `POST A/quota-grants/{grant}/return-operations`：`quota.reclaim` | 发起安全回收协调，先取得原分片封存证据再回收；前端不能提交一个金额直接释放资金 |
| A11 | `GET A/orders`、`GET A/orders/{order}`、`GET A/payments`：`payment.read` | `POST A/orders/{order}:verify-payment`：`payment.verify` | 以原商户引用核实支付；返回真实支付与入账状态，不能手动标记已付款 |
| A12 | `GET A/settlements`、`GET A/settlements/{settlement}`、`GET A/ledger-transactions/{transaction}`、`GET A/invoices`：`billing.read` | 无覆盖或删除正式分录的接口 | 返回原始金额、价格快照、请求、来源、分录及关联更正；金额为字符串 |
| A13 | `GET A/refunds`、`GET A/adjustments`：`billing.read` | `POST A/payments/{payment}/refunds`：`billing.refund`；`POST A/settlements/{settlement}/refunds`、`POST A/billing-adjustments`：`billing.adjust` | 必填原业务引用、原因、证据和幂等键；仅允许已定义的退款或更正类型，不能任意修改余额 |
| A14 | `GET A/reconciliation-runs`、`GET A/reconciliation-cases/{case}`：`reconciliation.read` | `POST A/reconciliation-runs`、`POST A/reconciliation-cases/{case}/actions`：`reconciliation.execute` | 输入核对范围或白名单恢复动作；保存水位、差异、原证据和处理结果 |
| A15 | `GET A/requests`、`GET A/requests/{request}`、`GET A/tasks`、`GET A/tasks/{task}`：`request.read` | `POST A/tasks/{task}:cancel`：`task.cancel` | 服务端定位原分片；默认仅元数据，费用字段另查相应权限；取消返回受理与核实状态 |
| A15 | `GET A/requests/{request}/content`、`GET A/tasks/{task}/content`：`content.read` 加有效临时授权 | 无凭正文内容修改执行或收费的接口 | 校验内容确有保留、授权未过期且范围匹配；记录内容读取审计 |
| A16 | `GET A/reports/usage`：`report.usage.read`；`GET A/reports/finance`：`finance.read`、`cost.read` | 导出走本章导出接口，另需 `report.export` | 返回分项收入、上游成本和平台失败成本；待核实数据不计作已确认毛利 |
| A17 | `GET A/audit-logs`：`audit.read`；`GET A/support-access-grants`：`support_access.read` | `POST A/support-access-grants`、`POST A/support-access-grants/{grant}:revoke`：`support_access.write` | 记录批准依据、被授权人员、范围、动作和有效期；支持人员不能自行扩大范围 |
| A18 | `GET A/system/health`、`GET A/alerts`：`observability.read` | `POST A/alerts/{alert}:acknowledge`：`alert.manage`；`POST A/recovery-actions`：`recovery.execute` | 告警确认不代表故障恢复；恢复动作必须绑定案例、校验当前状态并保留证据 |
| A19 | `GET A/operators`、`GET A/roles`：`operator.read` | `POST A/operators`、`PUT A/operators/{operator}/grants`、`POST A/operators/{operator}:disable`：`operator.write` | 只能授予操作者有权委派的权限，不能通过编辑自身记录提升权限 |
| A20 | `GET A/operational-settings`、`GET A/announcements`：`settings.read` | `PUT A/operational-settings/{setting}`、`POST A/announcements`、`POST A/announcements/{announcement}:publish`：`settings.write` | 配置类型白名单、版本和发布记录；密钥及基础设施配置使用各自受控流程 |
| A21 | `GET A/contracts`、`GET A/plans`：`commercial.read` | `POST A/contracts`、`POST A/plans`、`POST A/contracts/{contract}:activate`：`commercial.write` | P1；合同、权益、有效期、账期和授信规则版本；不得绕过资金约束 |
| A22 | `GET A/approval-requests`：`approval.read` | `POST A/approval-requests`、`POST A/approval-requests/{approval}:approve`、`POST A/approval-requests/{approval}:reject`：分别检查发起资格和 `approval.review` | P1；执行时再次校验审批版本、权限与业务条件，审批通过不等于业务已执行 |

后台下载和导出接口为 `POST A/exports`、`GET A/exports/{export}`、`POST A/exports/{export}/download-authorizations`、`GET A/downloads/{authorization}`。创建时检查目标报表或审计资源的读取权限与对应导出权限；下载时重新校验人员、范围和当前权限。导出任务返回状态、进度、截止时间和受控下载入口，不返回永久公开文件地址。

### 2.4 后台页面的业务约束

1. 暂停租户、撤销 Key 和收紧执行权限需要展示期望状态、当前生效状态及未确认目标。存在失联分片时保留处理中状态；普通停用不取消合法在途任务，也不阻止历史结算。
2. 采购价、销售价、企业协议价分别授权和版本化。正在执行的请求沿用准入时冻结的价格与预算窗口。
3. 充值款退款与消费退款是不同动作。前者核实原支付和可退来源，必要时安全回收未用委托额度；后者追加原消费的更正记录，不修改原始收费和运行时累计消费。
4. 核对和恢复入口只提供已定义的安全动作，例如核实原支付、查询原厂商任务、重投原清算意图、补派原回执、继续原封存回收。不能通过“重试”创建另一收费任务，不能把未知提交直接改成未执行。每个恢复动作还要检查对应领域权限；`recovery.execute` 本身不授予退款、调账或扩大额度的能力。
5. 平台资料访问、内容访问、价格发布、凭证变更、资金操作和导出均记录审计。审计保存操作者、授权依据、目标、原因、前后版本、结果与关联业务 ID，剔除密钥和不必要的请求正文。
6. 财务、查询和运行时监控使用各自资源预算；大范围报表与导出异步执行，不在推理准入链路里扫描业务表。

### 2.5 后台开发优先级

| 开发阶段 | 页面与交付重点 | 可验收结果 |
| --- | --- | --- |
| D0–D1 | A01、A03–A09、A17、A19；模型和策略页面先交付基础版本 | 平台会话隔离，人员分权，租户和 Key 生命周期、策略发布与撤销进度可验证 |
| D2 | A10、A12、A14、A18 的额度清算部分 | 能追踪授权、冻结、清算、回执和安全回收；重复恢复不产生资金副作用 |
| D3–D4 | A02、A05、A06、A08、A15、A16 的文本与媒体部分 | 一个请求或任务可关联执行、用量、文件与账务状态，未知和取消状态可排查 |
| D5 | A07、A11–A14、A16、A20 的财务和运营部分 | 真实支付到账、核实补单、两类退款、账单与对账闭合 |
| D6 | A02、A17、A18 完善故障与容量观测 | 清算积压、撤销滞后、恢复和发布排空均有操作记录与验收证据 |
| D7 | A19 自定义角色、A21、A22 | 企业合同、商业权益和审批按正式契约执行，不降低原有资金约束 |

## 3 Client 客户门户

### 3.1 用户与资源归属

一个 User 可以拥有个人租户，也可以加入多个企业租户。个人用户通过 personal Tenant 管理自己的资源；企业成员通过 Membership 和项目授权操作 organization Tenant。企业资源层级为：租户 → 项目 → 应用 → API Key，应用关联服务账号或其他被授权主体。Key 固定绑定租户、项目和应用；企业生产凭证使用服务身份，避免随员工离职一起失效。

Client 的企业切换是选择目标租户。服务端每次请求重新校验身份、成员关系和资源归属，不能仅凭路径、Cookie 中上次选择或浏览器缓存授权。成员关系撤销后，旧页面、旧缓存、导出和下载入口同样受到权限检查。

### 3.2 默认角色与权限矩阵

权限使用 `tenant.*` 语义标识，并绑定租户或项目范围。表中“项目内”只指显式授权的项目；“读取”不自动包含导出或请求正文。“按授权”能力默认关闭，由有权的企业人员单独授予。

| 能力 | Owner | Admin | Developer | Billing | Auditor |
| --- | --- | --- | --- | --- | --- |
| 企业所有权、转移所有权 | 管理 | — | — | — | 只读相关审计 |
| 企业资料、邀请与成员角色 | 管理 | 在委派范围内管理，不能授予 Owner 或修改 Owner | 查看自身归属 | 查看必要账单资料 | 只读授权范围 |
| 项目、应用、服务账号 | 管理 | 管理授权范围 | 项目内读取；应用管理需显式授权 | 成本归属只读 | 只读授权范围 |
| Key 创建、轮换、撤销 | 管理 | 管理授权范围 | 项目内按应用和可授予策略操作 | — | 仅授权的掩码元数据 |
| 企业安全策略与敏感权限授予 | 管理 | 仅已委派的策略；不能自行扩权 | 查看相关有效策略 | — | 只读授权范围 |
| 硬预算、模型与流量限制 | 管理 | 管理授权范围 | 查看项目有效限制；不得扩大上级限制 | 读取财务相关预算 | 只读授权范围 |
| 文本、图片、视频调用及任务管理 | 全租户授权范围 | 授权范围 | 项目和应用内 | — | 元数据只读 |
| 用量和费用报表 | 全租户 | 授权范围 | 项目内；费用字段另按 `usage.cost.read` | 全租户财务范围 | 只读授权范围 |
| 充值、退款申请、账单资料 | 管理 | 默认只读，写入需单独财务授权 | — | 管理 | 授权范围只读 |
| 内容日志查看 | 按授权 | 按授权 | 按授权 | — | 按授权 |
| 输入文件与生成结果 | 按项目及动作管理 | 授权范围管理 | 项目内按上传、读取、下载、删除权限操作 | — | 元数据只读，下载另行授权 |
| 审计查询和导出 | 查询；导出单独授权 | 授权范围查询 | 自身或项目范围按授权 | 财务操作范围 | 查询；导出单独授权 |

Owner 拥有租户所有权和常规管理能力，不获得平台后台权限。企业最后一名 Owner 不得被成员管理接口移除、降权或禁用；所有权转移由当前 Owner 重新验证后发起，目标成员确认后原子完成。Admin 无权授予超出自己可委派范围的权限。

P0 提供固定角色、项目范围和受限的敏感权限附加授权，例如内容查看、文件下载、费用查看、财务写入与导出；不开放任意权限组合编辑器。P1 再增加自定义角色。个人租户用户使用自身资源范围内的 Owner 能力，页面隐藏企业成员和组织治理入口。

### 3.3 页面清单

| 页面 | 前端路由 | 页面内容和主要操作 | 优先级与阶段 |
| --- | --- | --- | --- |
| C01 公开站点与开发者中心 | `/`、`/models`、`/pricing`、`/docs`、`/status` | 产品说明、公开模型能力和价格、接入文档、错误说明、服务状态 | P0，D0–D5 |
| C02 注册登录与邀请接受 | `/signup`、`/login`、`/forgot-password`、`/reset-password`、`/invitations/accept` | 注册、验证、找回、MFA、接受企业邀请 | P0，D1 |
| C03 账户安全与租户选择 | `/account/security`、`/select-tenant`、`/organizations/new` | 个人资料、会话、MFA、个人/企业切换、创建企业 | P0，D1 |
| C04 控制台概览 | `/console/{tenant}` | 快速接入、可用与冻结金额、待入账、近期用量和任务、预算提醒 | P0，D2–D5 |
| C05 模型与适用价格 | `/console/{tenant}/models`、`/console/{tenant}/models/{model}` | 当前可用模型、能力、参数、适用价格、示例和接入限制 | P0，D1–D4 |
| C06 项目管理 | `/console/{tenant}/projects`、`/console/{tenant}/projects/{project}` | 项目资料、成员范围、状态与成本归属 | P0，D1 |
| C07 应用与服务账号 | `/console/{tenant}/projects/{project}/applications`、`/console/{tenant}/service-accounts` | 应用、环境、服务身份、关联 Key、停用与生效进度 | P0，D1 |
| C08 API Key | `/console/{tenant}/api-keys` | 创建、一次明文展示、掩码列表、用途、到期、限制、轮换、撤销和批量操作 | P0，D1–D3 |
| C09 预算与告警 | `/console/{tenant}/budgets` | 租户/项目/Key 预算、周期、已消费与占用、阈值提醒、有效限制 | P0，D2 |
| C10 文本 Playground | `/console/{tenant}/playground` | 选择项目、应用和模型，参数配置，非流式/SSE，停止、用量与收费状态 | P0，D3 |
| C11 图片与视频任务 | `/console/{tenant}/tasks`、`/console/{tenant}/tasks/new`、`/console/{tenant}/tasks/{task}` | 输入、参数、提交、排队、进度、取消、文件、收费状态和错误解释 | P0，D4 |
| C12 文件与生成结果 | `/console/{tenant}/assets`、`/console/{tenant}/assets/{asset}` | 输入文件上传、就绪状态、生成结果预览、受控下载与删除申请 | P0，D4 |
| C13 调用记录与用量 | `/console/{tenant}/usage`、`/console/{tenant}/requests/{request}` | 按项目、Key、模型、时间筛选；请求 ID、计量、费用、延迟、错误和导出 | P0，D3–D5 |
| C14 钱包与订单 | `/console/{tenant}/wallet`、`/console/{tenant}/orders/{order}` | 充值、订单状态、退款申请、来源与退款进度；结果以服务端核实为准 | P0，D2–D5 |
| C15 账单与发票申请 | `/console/{tenant}/billing/invoices`、`/console/{tenant}/billing/profile` | 账单和更正、下载、企业抬头、发票申请和申请进度 | P0，D5 |
| C16 企业成员与权限 | `/console/{tenant}/members`、`/console/{tenant}/roles` | 成员、邀请、固定角色、项目范围、敏感能力授权和所有权转移 | P0，D1 |
| C17 企业安全与数据策略 | `/console/{tenant}/settings/security` | 模型访问约束、内容日志开关和保留期、文件访问、敏感权限及策略生效状态 | P0，D1–D4；SSO 等在 C20 |
| C18 企业审计 | `/console/{tenant}/audit-logs` | 成员、Key、预算、财务和内容访问操作记录；按授权查询和导出 | P0，D1–D5 |
| C19 通知与帮助 | `/console/{tenant}/notifications`、`/help` | 余额、预算、任务和安全提醒；个人订阅偏好、文档与支持入口 | P0，D2–D5 |
| C20 企业身份集成 | `/console/{tenant}/settings/sso`、`/console/{tenant}/settings/provisioning` | SAML/OIDC、域名验证、强制登录策略、SCIM、同步和离职处理 | P1，D7 |
| C21 自定义角色与审批 | `/console/{tenant}/settings/custom-roles`、`/console/{tenant}/approvals` | 自定义权限集合、项目范围、申请、审批和执行记录 | P1，D7 |
| C22 商业与付款设置 | `/console/{tenant}/contracts`、`/console/{tenant}/settings/payments` | 合同套餐、自动充值、支付授权、授信和账期 | P1，D7 |
| C23 自有上游凭证 | `/console/{tenant}/settings/providers` | BYOK 凭证引用、健康、授权模型、成本与平台服务费规则 | P1，D7 |

### 3.4 账户和资源管理 API

以下权限均带 `tenant.` 前缀，例如 `project.read` 表示 `tenant.project.read`。本人账户动作不依赖某个企业角色，但仍校验会话与目标账户一致。

| 页面 | 接口 | 权限 | 关键输入和返回 |
| --- | --- | --- | --- |
| C01 | `GET P/models`、`GET P/models/{model}`、`GET P/prices`、`GET P/status`、`GET P/announcements` | 公开 | 仅公开能力、公开价和脱敏服务状态；文档和产品说明可由静态内容交付，不返回协议价、租户或内部拓扑 |
| C02 | `POST C/registrations`、`POST C/email-verifications:confirm`、`POST C/password-reset-requests`、`POST C/password-resets` | 注册或持有有效验证材料；接口防滥用 | 验证材料单次、限时；找回接口不通过响应泄露账户是否存在；个人租户创建幂等 |
| C02 | `POST C/sessions`、`POST C/mfa/challenges/{challenge}:verify`、`GET C/sessions/current`、`DELETE C/sessions/current` | 登录挑战或本人会话 | 登录、MFA、当前身份；认证组件原生契约由接入层映射 |
| C02、C16 | `POST C/invitations:accept` | 已验证身份与邀请接收人一致 | 邀请令牌、过期与使用状态；接受后建立目标租户成员关系，不能自选提升角色 |
| C03 | `GET C/me`、`PATCH C/me`、`GET C/me/tenants`、`POST C/tenants` | 本人；创建企业受开户策略约束 | 成员关系、租户类型、可见入口；创建企业返回组织与 Owner 关系 |
| C03 | `GET C/me/sessions`、`DELETE C/me/sessions/{session}`、`GET C/me/mfa`、`POST C/me/mfa/enrollments`、`POST C/me/mfa/enrollments/{enrollment}:verify`、`DELETE C/me/mfa/factors/{factor}` | 本人；安全变更需近期重新验证 | 会话和 MFA 元数据、必要的一次绑定材料；不暴露其他成员认证秘密 |
| C03、C04 | `GET T/me/permissions`、`GET T/overview` | 有效成员关系；概览子项分别检查用量和财务权限 | 当前租户与项目授权、页面能力、按权限裁剪的概览和更新时间 |
| C05 | `GET T/models`、`GET T/models/{model}` | `model.read` | 当前用户、应用或项目可用能力和适用价格；实际调用时继续校验有效权限 |
| C06 | `GET T/projects`、`GET T/projects/{project}` | `project.read` | 返回获授权项目；列表不能泄露其他项目名称和统计 |
| C06 | `POST T/projects`、`PATCH T/projects/{project}`、`POST T/projects/{project}:archive`、`PUT T/projects/{project}/member-scopes` | `project.write`；授权变更另需 `member.scope.write` | 资料、版本、成员范围；归档停止新准入并跟踪生效，不删除历史财务与未知任务 |
| C07 | `GET T/projects/{project}/applications`、`GET T/applications/{application}`、`GET T/service-accounts` | 分别为 `application.read`、`service_account.read` | 项目、环境、服务身份、状态和关联资源的掩码元数据 |
| C07 | `POST T/projects/{project}/applications`、`PATCH T/applications/{application}`、`POST T/applications/{application}:disable` | `application.write` | 检查项目和服务账号归属；停用返回生效操作，不直接删除资金引用 |
| C07 | `POST T/service-accounts`、`POST T/service-accounts/{account}:disable` | `service_account.write` | 服务账号范围和管理人；停用关联凭证按撤销屏障生效 |
| C08 | `GET T/api-keys`、`GET T/api-keys/{key}` | `key.read` | 掩码、用途、项目、应用、到期、策略与撤销状态；无明文读取接口 |
| C08 | `POST T/api-keys` | `key.create` | 已授权应用、模型、限制、到期时间；服务端限制在可授予范围内，首次成功响应返回明文 |
| C08 | `PATCH T/api-keys/{key}` | `key.policy.write` | 名称、到期和限制的允许变更；归属不可改，权限收紧返回操作及有效版本 |
| C08 | `POST T/api-keys/{key}:rotate`、`POST T/api-keys/{key}:revoke`、`POST T/api-keys:batch-revoke` | 分别为 `key.rotate`、`key.revoke` | 轮换创建替代 Key，旧 Key 的重叠有效期有界；撤销返回 `operation_id` 和真实状态，批量逐项返回 |
| C09 | `GET T/budgets`、`GET T/budgets/{budget}` | `budget.read` | 预算范围、窗口、上限、已消费、未结清分配、更新时间和有效策略 |
| C09 | `POST T/budgets`、`PUT T/budgets/{budget}` | `budget.write` | 限额、币种、周期、资源范围和版本；不能突破上级约束，不能降到已消费加未结清分配之下 |
| C09 | `GET T/alert-rules`、`POST T/alert-rules`、`PUT T/alert-rules/{rule}` | 读取 `alert.read`，创建和修改 `alert.write` | 阈值、通知接收范围和启停；软告警不代替硬预算，预算不会产生额外资金 |
| C16 | `GET T/members`、`GET T/invitations`、`GET T/roles` | `member.read`；角色目录对有效成员可见 | 固定角色、可委派范围、成员与邀请状态；敏感资料按字段授权 |
| C16 | `POST T/invitations`、`POST T/invitations/{invitation}:revoke`、`PUT T/members/{member}/role-scopes`、`POST T/members/{member}:remove` | `member.write`；项目范围另需 `member.scope.write` | 验证可委派角色和范围、最后 Owner 约束；移除成员触发个人凭证撤销，不删除企业服务账号 |
| C16 | `PUT T/members/{member}/sensitive-permissions` | `sensitive_access.write` 加重新验证 | 仅 P0 允许的敏感能力白名单及资源范围；不可授予 Owner 或平台权限 |
| C16 | `POST T/ownership-transfers`、`POST T/ownership-transfers/{transfer}:accept` | 当前 Owner 发起，指定目标成员接受；双方重新验证 | 转移过期、身份、成员状态和原 Owner 版本校验，原子完成并保留审计 |
| C17 | `GET T/security-policies`、`PUT T/security-policies` | `policy.read`、`policy.write` | 内容、访问和保留规则版本；不能扩大平台授权或删除必须保留的资金、去重及审计证据 |
| C18 | `GET T/audit-logs` | `audit.read` | 按操作者、动作、资源和时间筛选；仅返回当前授权范围 |
| C19 | `GET T/notifications`、`POST T/notifications/{notification}:read`、`GET C/me/notification-preferences`、`PUT C/me/notification-preferences` | 当前身份；租户通知另校验成员关系和通知资源权限 | 仅标记本人的已读状态；偏好不关闭平台必须执行的预算和安全控制 |

### 3.5 调用任务文件与财务 API

| 页面 | 接口 | 权限或调用身份 | 关键输入和返回 |
| --- | --- | --- | --- |
| C10、C11 | `POST T/playground-authorizations` | `inference.invoke`；检查项目、应用、模型和用途 | 短期授权绑定用户、租户、应用、用途、模型、额度和有效期；只能缩小现有授权 |
| C10 | `POST /v1/chat/completions`；`POST /v1/responses` 按声明的兼容范围提供 | Playground 短期授权；生产应用使用平台 Key | 支持非流式/SSE、输出上限、请求 ID 和用量事实；推理不经过 Hono 逐请求资金准入 |
| C11 | `POST G/tasks` | 短期调用授权或生产平台 Key，需 `inference.invoke` 对应能力 | `kind=image/video`、模型、输入资产和参数；持久受理后返回 `202`、原任务 ID、预占金额和查询地址 |
| C11 | `GET T/tasks`、`GET T/tasks/{task_id}` | `task.read`，项目和应用范围校验 | 列表是带水位投影，详情由服务端定位原分片；执行、文件、账务和取消状态分开返回 |
| C11 | `POST T/tasks/{task_id}:cancel` | `task.cancel` | 业务 API 校验后以受限服务身份请求原分片取消；生产 API 仍使用 `POST G/tasks/{task_id}:cancel` |
| C12 | `POST T/assets/upload-intents`、`POST T/assets/{asset}:complete` | `asset.write` | 项目、文件类型、大小和校验信息；限制上传目标和有效期，服务端确认对象后才标为可用 |
| C12 | `GET T/assets`、`GET T/assets/{asset}` | `asset.read` | 归属、来源任务、存储状态和到期时间；不返回永久公开地址 |
| C12 | `POST T/assets/{asset}/download-authorizations`、`GET C/downloads/{authorization}` | `asset.download`，当前身份与授权绑定 | 受控下载再次鉴权；输入文件和生成结果均检查租户、项目与数据保留规则 |
| C12 | `POST T/assets/{asset}/deletion-requests` | `asset.delete` | 校验是否仍被在途任务使用及保留义务，返回删除操作；删除文件不删除消费事实 |
| C13 | `GET T/usage`、`GET T/requests/{request}` | `usage.read`；费用另需 `usage.cost.read` | 计量分项、原请求 ID、延迟、错误、执行与入账状态、价格版本和查询水位 |
| C13 | `GET T/requests/{request}/content` | `content.read` | 仅在内容日志已启用且有保留记录时读取；项目授权和敏感读取审计独立检查 |
| C14 | `GET T/wallet`、`GET T/orders`、`GET T/orders/{order}`、`GET T/refunds/{refund}` | `billing.read` | 已入账、中央可再分配、子池可用、请求冻结、待入账和退款冻结分别返回，附 `as_of` 与水位 |
| C14 | `POST T/topup-orders` | `billing.topup` | 充值规格或期望金额；服务端校验并确定订单金额、币种与支付渠道，返回支付指引和订单 ID |
| C14 | `POST T/orders/{order}/refund-requests` | `billing.refund_request` | 充值款退款申请，关联原支付来源；返回受理状态，核实和退款由 Payments/Billing 执行 |
| C13、C14 | `POST T/requests/{request}/charge-review-requests` | `billing.refund_request` | 消费费用复核或退款申请、原因与证据；不直接改价、解冻或重复执行模型 |
| C15 | `GET T/invoices`、`GET T/invoices/{invoice}`、`POST T/invoices/{invoice}/download-authorizations` | `billing.read`；下载另需 `billing.export` | 沿用现有 `/invoices` 账单资源，包含消费明细和更正记录；下载走受控入口 |
| C15 | `GET T/billing-profile`、`PUT T/billing-profile`、`POST T/invoice-requests`、`GET T/invoice-requests` | 读取 `billing.read`，修改和申请 `billing.profile.write` | 税务发票申请与消费账单分开建模；发票渠道、字段和开具规则按上线地区与实际渠道确定 |
| C13、C15、C18 | `POST T/exports`、`GET T/exports/{export}`、`POST T/exports/{export}/download-authorizations` | 原资源读取权限加 `usage.export`、`billing.export` 或 `audit.export` | 异步导出绑定发起人、租户、过滤器、字段和截止时间；下载重新鉴权，不保留已撤销访问 |
| C08–C18 | `GET C/operations/{operation_id}` | 操作所属租户和当前资源读取权限 | 返回当前状态、进度和结果引用；操作 ID 不能成为跨租户查询凭证 |

浏览器不持有厂商凭证，长期平台 Key 不进入前端静态包或通用日志。Playground 的短期授权必须纳入同一额度、计量和撤销模型；授权中的金额是调用约束，不是新入金。读取任务、请求和操作不产生新的推理意图。

任务取消、停用应用或移除成员也可能跨分片生效。UI 显示“受理”“处理中”“已完成”以及对象本身的真实状态，不通过隐藏按钮或本地改状态宣告成功。离职移除用户的个人调用授权和个人凭证；企业服务账号继续按企业管理员配置的生命周期运行。

### 3.6 P1 页面接口与权限

| 页面 | 设计接口 | 权限和实施要求 |
| --- | --- | --- |
| C20 SSO 与 SCIM | `GET T/sso-configuration`、`PUT T/sso-configuration`、`POST T/sso-tests`、`POST T/domain-verifications`、`GET T/provisioning-status`、`POST T/scim-credentials:rotate` | `identity.manage`；域名归属验证、近期重新验证、恢复登录路径、令牌一次展示与轮换；SCIM 服务入口独立认证和授权 |
| C21 自定义角色与审批 | `POST T/roles`、`PUT T/roles/{role}`、`GET T/approval-requests`、`POST T/approval-requests`、`POST T/approval-requests/{approval}:approve`、`POST T/approval-requests/{approval}:reject` | 分别检查 `role.manage`、动作发起资格和 `approval.review`；不得委派自身范围以外的能力，审批后执行仍检查当前资金和权限 |
| C22 合同和付款设置 | `GET T/contracts`、`GET T/plans`、`GET T/auto-topup-settings`、`PUT T/auto-topup-settings`、`GET T/credit-account` | 合同读取 `commercial.read`，付款设置 `billing.payment_settings.write`；支付授权、幂等、风控、授信上限与原资金约束共同验收 |
| C23 BYOK | `GET T/provider-accounts`、`POST T/provider-accounts`、`POST T/provider-accounts/{account}/credential-versions`、`POST T/provider-accounts/{account}:disable` | `provider_credentials.manage`；凭证仅服务端托管，企业归属、模型范围、健康、上游成本和平台服务费规则分别明确 |

P1 的 SSO、SCIM、自定义角色、授信与 BYOK 需在实现前补齐字段和故障语义，不把页面可保存配置当作集成完成。默认角色和 P0 接口不能先开放这些能力再补授权。

### 3.7 Client 开发优先级

| 开发阶段 | 页面与交付重点 | 可验收结果 |
| --- | --- | --- |
| D0–D1 | C01–C03、C05–C08、C16–C18；内容策略先交付开关和权限骨架 | 个人及企业开户、邀请、角色、项目和 Key 可用；两个企业交叉读取与写入均被拒绝 |
| D2 | C04、C09、C14 的钱包读取、C19 的预算和安全通知 | 正确区分可用、冻结与待入账；预算不会创建资金；额度补充中不会误报余额为零 |
| D3 | C10、C13 的文本执行与费用状态 | Playground 短期授权、文本/SSE、请求查询和最终费用关联完整 |
| D4 | C11、C12、C17 的媒体与文件能力 | 图片、视频各至少一个真实适配器；重复提交、取消、未知任务和文件保存失败可正确恢复 |
| D5 | C14–C15、C18–C19 的支付财务、导出与通知 | 充值、核实到账、两类退款申请、账单下载、发票申请和权限导出贯通 |
| D6 | C04、C10–C14 的背压、失联与恢复体验 | 原任务可继续查询，未知费用有解释，不因刷新、网络重试或重复通知重复生成或收费 |
| D7 | C20–C23 | 企业身份、角色审批和商业增强有独立验收与恢复路径 |

## 4 页面共用的接口规则

### 4.1 请求认证和资源授权

- 后台会话、Client 会话、生产 API Key、Playground 授权和内部服务身份分别验证受众与用途。Gateway 不直接接受普通浏览器登录 Cookie 作为生产调用凭证。
- Cookie 会话采用安全属性、CSRF 防护和明确来源的跨域策略。敏感操作的近期重新验证由服务端记录，不能接受浏览器自报“已验证”。
- 每个接口先确定可信操作者，再验证租户成员或平台授权、项目范围、目标资源归属和动作权限。资源 ID 不构成授权，父子资源组合必须同属合法范围。
- 角色变更和成员移除后，新的业务请求重新授权。推理权限撤销继续遵守 `REVOKING → REVOKED` 的分片持久确认屏障；历史已获准入任务按既定规则处理。
- Client 的查询缓存包含身份、租户、项目与查询条件，切换或退出时清理对应缓存；个性化数据不进入跨租户共享缓存。

### 4.2 分页字段与查询时效

列表统一采用游标分页，参数包括 `cursor`、`page_size`、时间范围和明确支持的过滤器；页大小、时间跨度和导出并发由服务端设置上限。响应提供 `items`、`next_cursor`，涉及异步投影时提供 `as_of`、`watermark` 和 `data_status`。

金额、单价和可能超过安全整数的数量用字符串表达，金额附币种，计量附单位。前端只格式化金额，不计算权威余额、最终扣费或毛利。报表区分已核实与待核实数据，查询投影不参与资金准入。

配置类更新携带资源版本或 `If-Match`。版本冲突要求重新读取，不能盲目覆盖。正式分录、原始收费和已使用价格版本通过追加更正或新版本维护，不提供通用覆盖接口。

### 4.3 幂等与异步操作

| 动作 | 幂等与结果规则 |
| --- | --- |
| 管理和门户业务写入 | 创建 Key、租户、订单、退款、导出、授权回收和其他有副作用动作携带 `Idempotency-Key`；作用域为 `principal_id + audience + target_tenant_id + operation_kind + Idempotency-Key`；本人账户动作的目标租户可为空，保存规范化指纹和原结果 |
| Gateway 执行 | 沿用 `tenant_id + application_id + operation_kind + Idempotency-Key`；同意图固定原请求和原分片，不能通过换 Key、换 Cell 绕过原执行结果 |
| 相同键不同内容 | 返回 `409 idempotency_conflict`；重放继续检查当前身份和原资源访问权限 |
| Key 创建响应丢失 | 原意图重试只返回原 Key 元数据和 `secret_recoverable:false`；不保存可恢复明文，不创建第二把 Key；必要时撤销原 Key 后以新意图创建替代 Key |
| 撤销和跨分片策略变更 | 先返回受理结果和 `operation_id`，仅持久确认或可验证隔离后报告最终生效 |
| 金额和支付操作 | 幂等键之外，再由原业务唯一约束防止不同键重复入账、收费或超额退款；客户端刷新和回跳不能改变支付事实 |
| 任务取消和未知执行 | 保留原请求、上游尝试和资金占用，核实后再推进状态；不按 TTL 或页面超时认定为免费失败 |

异步业务受理返回 `202`、`operation_id`、当前领域状态、查询地址和建议查询间隔。操作任务使用 `PENDING`、`RUNNING`、`SUCCEEDED`、`FAILED`、`REQUIRES_REVIEW` 等协调状态；对象自身仍使用 Key 撤销、任务执行和账务状态。协调操作失败不自动撤销已确认的资金事实。

`GET` 查询、任务轮询和下载均不触发新的模型生成。后台恢复作业和 Worker 使用原业务 ID 恢复；浏览器不能通过换新标识自动重做处于未知状态的付费调用。

### 4.4 页面状态与错误处理

| 业务状态或错误 | 页面行为 |
| --- | --- |
| `401 invalid_credential` | 请求重新登录或更新凭证，保留安全的原资源定位信息 |
| `403 forbidden`、`model_not_allowed` | 说明权限不足；不自动切换到其他企业、应用或未经授权模型 |
| 跨租户或不可见资源 | 按统一隐藏策略返回 `404`，不泄露目标资源是否存在 |
| `402 insufficient_funds` | 仅在授权及中央可分配或安全回收路径确认不足时提示充值 |
| `429 rate_limit_exceeded` | 显示速率或并发限制及重试建议 |
| `429 budget_limit_exceeded` | 显示预算范围和窗口；充值不会自动提高硬预算 |
| `503 quota_refill_pending`、`funds_temporarily_unavailable` | 显示额度同步或资金暂不可用，保留原意图，不能把未知余额显示为零 |
| `409 idempotency_conflict`、`412` 配置版本冲突 | 指明冲突并重新读取；不自动换幂等键重做资金或推理操作 |
| `503 home_shard_unavailable`、`durability_unconfirmed` | 保留原 ID，允许查询和等待；不能在另一分片重建任务 |
| `REVOKING` | 显示撤销处理中；禁止显示“全部已失效”，原资源仍可查询 |
| `SUBMISSION_UNKNOWN`、`RECONCILING`、`SETTLEMENT_PENDING` | 分别显示执行待核实、费用核对中或待入账，不合并为执行失败 |
| `CANCEL_REQUESTED` | 显示取消已受理，继续查询真实执行和费用结果 |
| 文件未保存或下载失败 | 显示结果保存/获取状态，重试文件处理，不重新生成 |
| SSE 已开始后的异常 | 展示协议内错误和原请求状态；不拼接第二次生成，停止按钮不承诺立即释放全部占用 |

### 4.5 资金和媒体页面口径

钱包接口的已入账、中央可再分配、子池可用、请求冻结、待入账、退款冻结有不同含义，不能直接全部相加或重复扣减。页面优先展示用户可理解的可用、冻结和待入账摘要，详细资金关系与水位由后端生成；中央及本地金额关系沿用 [账务规范](04-billing-and-ledger.md)。

任务详情保留 `execution_status`、`billing_status`、`result_storage_status`、`cancel_status` 四个维度。生成成功不等于文件已保存、费用已入账；取消成功与是否收费由可信执行和计量事实确定。内容保留和文件删除策略不能清理未结清资金、未知任务或去重墓碑。

## 5 开发依赖与上线验收

### 5.1 实施顺序

| 阶段 | 后端和契约前置 | 后台与 Client 协同交付 | 完成条件 |
| --- | --- | --- | --- |
| D0 | 工程结构、业务域、权限字典、OpenAPI、状态与金额约束；冻结认证映射 | 两个前端壳、路由、页面权限和模拟响应；模拟数据不得接入真实资金执行 | 页面、动作、权限和接口逐项可追踪，生成客户端与契约可校验 |
| D1 | 身份、租户、成员、项目、应用、服务账号、Key、版本化策略和撤销屏障 | 后台控制面与 Client 组织治理页面 | 双租户越权拒绝；平台/客户会话不互通；撤销完成有持久确认 |
| D2 | 中央钱包和预算、批次授权、Go 持久预占、清算、回执、Outbox/Inbox 与安全回收 | 两端额度、钱包读取、预算和异常页面；用沙箱或受审计测试入账联调 | 两层资金守恒、所有硬预算、重复事件和回收竞争通过真实数据库验证 |
| D3 | 一个文本上游、非流式/SSE、可靠计量、断流和未知恢复 | Playground、调用详情、文本排障及费用状态 | 完整调用到一次原始收费；无 Hono 逐请求准入依赖 |
| D4 | 图片和视频各至少一个适配器、公平调度、回调/轮询、取消和对象存储 | 任务、文件和媒体排障页面 | 重复回调、未知提交、取消竞争、文件失败及长任务跨周期正确 |
| D5 | 支付核实、到账、原路退款、消费更正、账单、通知和导出 | 后台财务运营与 Client 钱包、订单、账单和申请页面 | 从充值到三类执行、清算、账单和对账闭环，真实资金动作不重复 |
| D6 | HA、恢复、发布排空、权限故障和容量联合演练 | 两端故障状态、积压提示、运维和审计页面 | 三组十万负载分别验收，金额差异与容量同时达标；采购容量有证据 |
| D7 | 企业身份、审批、商业权益与新定价契约 | P1 页面 | 集成、授权、预算、计量、资金和恢复一并验收 |

两个前端可按已冻结契约并行开发，业务依赖按阶段顺序闭合。D2 是所有真实付费执行的前置条件；D3 与 D4 可在共同底座稳定后部分并行。已有 [总体开发路线](06-development-roadmap.md) 的 P0 范围和容量目标保持不变。

### 5.2 首个可交付里程碑

首个里程碑包含 A01、A03、A04、A05–A09 基础配置、A17、A19，以及 C02、C03、C05–C08、C16–C18 的身份权限部分。验收时创建两个企业、各自的 Owner 和 Developer、项目、应用、服务账号与 Key，证明：

1. 企业 A 的人员不能读取、修改或撤销企业 B 的资源；对当期已实现接口，更换路径 ID、过滤器、项目 ID 或操作 ID 均不能越权。导出和下载在 D4–D5 实现后加入同一隔离验收。
2. 平台运营、财务、支持、安全和审计人员只具备已授予动作；财务不能读取正文，支持不能调账，企业 Owner 不能进入后台。
3. Developer 仅能在获授权应用创建受上级策略限制的 Key；无权扩大模型、预算、项目或企业范围。
4. 明文 Key 只在首次成功响应展示，重复意图不生成第二把 Key；轮换和撤销显示真实进度。
5. 成员变更、最后 Owner 限制、邀请过期、所有权转移和企业服务账号生命周期都有可观察结果与审计。

这一里程碑交付身份和控制面基础。正式收费上线继续完成 D2–D6 的资金、三类执行、支付和故障容量验收。

### 5.3 上线必须覆盖的验收场景

| 验收面 | 关键场景与预期结果 |
| --- | --- |
| 页面与对象授权 | 直接调用隐藏按钮对应接口、修改资源 ID、切换企业、旧缓存、导出和下载都不能越权 |
| 字段与敏感权限 | 无采购成本权限看不到成本，无内容权限看不到正文，无凭证权限不能更换上游密钥，审计不记录明文 Key |
| 邀请和人员生命周期 | 重复/过期邀请、越权授角色、最后 Owner 移除、离职个人凭证与企业服务账号分别处理 |
| Key 与策略撤销 | 活跃、失联和新加入分片均遵守当前策略版本；未持久确认保持处理中 |
| 预算与金额 | 同租户跨 Cell 并发、多个项目和 Key 同时消费，所有预算与资金上限均有效；降低预算不覆盖在途占用 |
| 支付与退款 | 支付回跳、重复回调、未知到账、重复核实、退款与新分配竞争，均不重复入账或复用已退款资金 |
| 文本与媒体执行 | SSE 断流、部分交付、重复任务、未知提交、取消竞争、回调乱序和文件失败保持原意图和真实费用 |
| 异步查询与导出 | 水位滞后有标识，轮询不放大成厂商调用，权限撤销后查询和受控下载重新校验 |
| 恢复与发布 | 重投原清算和回执不重复收费，未知事实不强行解冻，数据库与 Worker 恢复不创造可消费余额 |
| 容量与运营 | 十万有效文本流、十万媒体在途、合计十万混合场景分别满足 [联合验收](07-reliability-and-validation.md)，同时验证管理查询和导出不会挤占资金执行资源 |

D0 校验文档与契约，D1–D5 使用真实数据库和可控厂商/支付故障测试，D6 使用目标生产架构演练；不以页面模拟数据或空连接数量代替业务、金额与容量证据。

### 5.4 实施前需确定的业务参数

支付渠道与结算币种、金额舍入、部分交付或取消后的收费条款、可退来源与手续费、内容和文件保留期、首批模型适配器、SSO 采购要求、告警接收方式、发票渠道，以及 P99、积压、RTO/RPO 等指标，按 [实施参数](08-decisions-and-open-questions.md) 固定。缺少参数的页面可先按契约开发，不开放未经明确授权和验证的真实资金或外部执行能力。

## 6 相关设计

- [产品范围与功能优先级](01-product-scope.md)
- [系统边界与身份资源模型](02-system-architecture.md)
- [数据库归属与表结构](03-database-design.md)
- [账务与额度子账](04-billing-and-ledger.md)
- [接口契约与跨系统交互](05-api-contracts.md)
- [开发顺序与交付计划](06-development-roadmap.md)
- [可靠性恢复与联合验收](07-reliability-and-validation.md)
- [架构决策与实施参数](08-decisions-and-open-questions.md)
- [最终技术选型方案](<AI Token SaaS 最终技术选型方案.md>)
