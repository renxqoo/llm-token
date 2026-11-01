# 接口契约与跨系统交互

更新日期：2026-11-02。版本：10 万并发目标架构。正常热路径在 Go 执行分片完成准入；Hono 的 /internal/v1 接口提供批次额度、安全回收、配置同步和异步清算。

全局数据与事务分别见 [数据库](03-database-design.md)、[账务](04-billing-and-ledger.md)，整体见 [高并发架构](<AI Token SaaS 10万并发架构设计方案.md>)。以下为待实现契约，示例 ID、币种和金额不代表已上线配置。

管理后台与 Client 的页面路由、逐动作权限、完整业务接口清单和开发优先级见 [两端实施规范](09-admin-and-client-spec.md)。该规范细化门户和管理入口，本文继续约束执行、幂等、资金、跨系统状态和持久化确认语义；实现阶段一并固化到 OpenAPI。

## 接口边界与认证

| 调用方 | 入口与认证 | 责任 |
| --- | --- | --- |
| 管理后台 | /admin/v1，独立平台会话与权限 | 租户、模型、价格、资金运营、对账和审计 |
| Client | /console/v1，用户会话与当前租户成员关系 | 个人/企业自助业务；Next.js 可代理但不自建钱包 |
| 客户应用 | Gateway /v1 模型协议、/gateway/v1 平台任务接口；平台 Key | 文本、图片与视频调用，归属从 Key 解析 |
| 控制平面与分配/清算 Worker | /internal/v1，服务身份与私网权限 | 授权、回收、计量接收与清算 |
| Cell 内部组件 | Go 领域函数；/runtime/v1 私有适配器校验服务身份与分片权限 | 持久化激活、预占、启动、释放与回执应用 |
| 模型厂商 | /callbacks/v1/providers/{provider}/tasks，厂商验签 | 保存可信回调，状态恢复；不直接记账 |

额度执行统一实现于 Go 的 internal/quota 模块，Gateway、媒体与恢复 Worker 通过进程内函数调用；/runtime/v1 是复用相同领域实现的私有控制适配器，不进入逐请求网络调用链。身份、归属、签名摘要与服务能力均由服务端校验；请求中的 tenant_id/金额不构成授权。

## 门户与管理接口

| 接口 | 行为 |
| --- | --- |
| POST /console/v1/sessions | 建立会话，认证组件原生路径由接入层统一映射 |
| GET /console/v1/me/tenants | 当前可访问租户 |
| GET/POST /console/v1/tenants/{tenant}/projects | 项目列表与创建 |
| GET/POST /console/v1/tenants/{tenant}/api-keys | Key 元数据、创建与一次明文展示 |
| POST /console/v1/tenants/{tenant}/api-keys/{key}:revoke | 创建撤销操作，返回 operation_id 与 REVOKING/REVOKED |
| GET /console/v1/operations/{operation_id} | 查询撤销、回收等异步操作进度及未确认目标 |
| GET /console/v1/tenants/{tenant}/models | 可用模型、能力与适用价格版本 |
| GET/PUT /console/v1/tenants/{tenant}/budgets/{budget} | 查询/修改预算，约束已消费与未结清分配额 |
| GET /console/v1/tenants/{tenant}/wallet | 分列 posted、中央可再分配、子池可用、请求冻结、待入账与退款冻结，携带 as_of/watermark |
| POST /console/v1/tenants/{tenant}/topup-orders | 创建充值意图，由服务端确定金额与币种 |
| GET /console/v1/tenants/{tenant}/orders/{order} | 核实后的订单状态 |
| GET /console/v1/tenants/{tenant}/usage | 消费与执行查询投影，区分待核对、待入账与已入账 |
| GET /console/v1/tenants/{tenant}/tasks/{task_id} | 服务端鉴权后路由原分片查询；不跨库写任务 |
| GET /console/v1/tenants/{tenant}/invoices | 已出账单与更正记录 |
| POST /console/v1/tenants/{tenant}/playground-authorizations | 签发短期、受限用途/模型/额度的调用授权 |

管理端在 /admin/v1 下提供对应受控操作。调账、退款、跨租户查看、凭证和价格操作留审计。Key 创建响应丢失后，同意图只返回原资源和 secret_recoverable:false，不保存可重复取得的明文。

## 模型与异步任务 API

| 接口 | 成功语义 |
| --- | --- |
| GET /v1/models | 当前 Key 可调用模型 |
| POST /v1/chat/completions | 文本非流式或 SSE，遵循声明的协议语义 |
| POST /v1/responses | 按明确的兼容范围实现与验证，不假设与其他协议相同 |
| GET /gateway/v1/requests/{request_id} | 逻辑请求执行与账务状态，包含原 home_shard_id |
| POST /gateway/v1/tasks | 图片/视频任务；预占、幂等、任务与 Outbox 持久确认后返回 202 |
| GET /gateway/v1/tasks/{task_id} | 状态、结果引用、账务状态和重试建议，不重新提交生成 |
| POST /gateway/v1/tasks/{task_id}:cancel | 未发送可原子取消；否则返回取消受理状态，后续查询结果 |

图片/视频统一平台任务接口不冒充所有厂商的原生接口。厂商原生图片、视频或 Anthropic 协议需在能力清单单独声明兼容范围。

异步受理示例，Idempotency-Key 由请求头提供；文件引用必须属于当前租户：

```json
{
  "kind": "video",
  "model": "video-example",
  "input": {"prompt": "示例描述", "image_asset_id": "asset_example"},
  "parameters": {"duration_seconds": 6, "resolution": "720p"}
}
```

持久受理响应：

```json
{
  "request_id": "req_example",
  "task_id": "task_example",
  "home_shard_id": "shard_example",
  "execution_status": "QUEUED",
  "billing_status": "RESERVED",
  "reservation_id": "res_example",
  "reserved_amount": "10.000000000000",
  "currency": "USD",
  "status_url": "/gateway/v1/tasks/task_example",
  "retry_after_ms": 2000
}
```

202 只表示任务与资金约束已可靠受理，不表示生成成功、厂商已经收到或费用已经入账。平台查询、Webhook 与厂商轮询分离；客户端查询不会直接放大为一轮厂商请求。资产保存失败重试搬运，不能因下载失败重做生成。

## 公共幂等与分片归属

统一作用域为 tenant_id + application_id + operation_kind + Idempotency-Key，绑定规范化载荷指纹和原请求/任务。operation_kind 表示具体操作/协议，例如 text.chat_completions、text.responses、image.generate、video.generate，公开 kind=video 映射到 video.generate。相同键不同载荷返回 409。重放仍校验当前身份与资源读取权限；Key 轮换不应隐式改变同一应用的业务幂等意图。

相同意图必须访问原逻辑 home_shard_id。路由版本变更与分片再均衡保留幂等状态及墓碑，不能在另一 Cell 随机重建。原分片不可确认时返回 503 home_shard_unavailable 或已知原任务状态。

没有声明幂等支持的推理入口或没有幂等键的调用是独立意图。流式结果不能重放时返回原 request_id 与 request_already_started，不以重新生成替代重放。

## 中央内部 API

| 接口 | 行为与幂等 |
| --- | --- |
| GET /internal/v1/gateway/snapshots | 受限配置、价格规则、策略、路由和服务许可快照 |
| GET /internal/v1/gateway/changes?cursor=... | 持久游标增量，缺口触发快照核对 |
| POST /internal/v1/quota-grants | 以 grant_request_id 分配一批额度，不是每请求预占 |
| GET /internal/v1/quota-allocations/by-operation/{grant_request_id} | 描述原分配操作状态；普通读不单独授予执行权 |
| POST /internal/v1/quota-allocations/{operation_id}:confirm | 结果未知时通过同级持久化确认取得原授权，不重新分配 |
| POST /internal/v1/quota-grants/{grant_id}:return | 接受原分片封存证明与 return_operation_id，幂等回收 |
| POST /internal/v1/settlements:batch | 异步有界小批清算，每个 settlement_operation_id 独立幂等 |
| GET /internal/v1/settlements/by-operation/{operation_id} | 查询原结算；可执行回执须来自持久确认流程 |
| POST /internal/v1/settlements/{settlement_id}:confirm | 对结果未知的结算执行同级确认并返回原回执 |
| POST /internal/v1/usage-events:batch | 幂等接收用量与采购尝试事实，不等于直接扣款 |
| POST /internal/v1/policy-barriers/{operation_id}:ack | 校验目标分片与持久水位，推进撤销操作 |

正常命中已激活子池的调用不访问这些 Hono 接口。首次加载、补充授权、撤销屏障、恢复和异步清算才跨系统交互；资金不足与授权暂不可补充要分开返回。

额度请求示例：

```json
{
  "grant_request_id": "allocation_example",
  "billing_account_id": "account_example",
  "project_id": "project_example",
  "application_id": "app_example",
  "key_id": "key_example",
  "home_shard_id": "shard_example",
  "binding_epoch": 4,
  "scope_fingerprint": "scope_example",
  "requested_amount": "100.000000000000",
  "currency": "USD",
  "budget_window_ids": ["budget_tenant", "budget_project", "budget_key"]
}
```

中央重新核验可信归属、适用预算与来源，不直接信任请求列出的窗口。一次操作可按来源返回多份 grant：

```json
{
  "allocation_operation_id": "alloc_op_example",
  "status": "CONFIRMED",
  "grants": [{
    "grant_id": "grant_example",
    "fund_lot_id": "fund_example",
    "home_shard_id": "shard_example",
    "binding_epoch": 4,
    "scope_fingerprint": "scope_example",
    "scope_snapshot": {
      "tenant_id": "tenant_example", "project_id": "project_example",
      "application_id": "app_example", "key_id": "key_example"
    },
    "allocated_amount": "100.000000000000",
    "currency": "USD",
    "budget_window_ids": ["budget_tenant", "budget_project", "budget_key"],
    "admit_before": "2026-11-01T12:05:00Z",
    "authorization_digest": "grant_digest_example"
  }]
}
```

admit_before 仅限制新请求，不是额度或长视频的自动退款时间。授权被持久激活后，以原 L 创建子池；重放不重新加钱。授权派发、查询确认和 Outbox 派发均遵守 [账务持久化屏障](04-billing-and-ledger.md) 的要求。

## Cell 内部运行时契约

| 私有入口；进程内调用复用同一领域方法 | 行为 |
| --- | --- |
| POST /runtime/v1/grants:activate | 校验 grant 身份、摘要、绑定和墓碑，幂等激活子池 |
| POST /runtime/v1/admissions | 校验服务许可/策略与幂等，在同一分片持久预占、请求、切片及可选异步任务 |
| POST /runtime/v1/admissions:resolve | 原提交结果未知时确认原操作，不能创建新请求 |
| POST /runtime/v1/reservations/{id}:start | 与取消竞争，持久取得尝试发送资格；重复读取原许可不允许再次发送 |
| POST /runtime/v1/reservations/{id}:release | 仅 HELD 且确认未取得发送资格时 H 转 F |
| POST /runtime/v1/grants/{id}:seal-return | 先停新准入、F 转封存 R、生成回收 Outbox |
| POST /runtime/v1/settlement-receipts | 去重回执、H/X/F 与请求状态同事务更新 |
| POST /runtime/v1/policy-barriers/{operation_id}:apply | 与准入有序应用禁用 epoch，持久确认后生成 ACK |

admissions 传入的是已认证执行上下文和模型参数摘要，不接受客户直接指定账户或收费金额。返回 reservation_id、request_id、E、冻结价格版本、原预算窗口、授权切片及执行状态。文本可在同事务中启动，媒体排队后本地启动，无需第二次 Hono RPC。

start 绑定 attempt_id、executor_epoch、shard_write_epoch 与历史授权。执行权只能授予一个尝试发送者；响应未知或尝试已为 DISPATCHING 时先恢复核对，不能把幂等响应当成厂商推理幂等令牌。

## 用量结算事件与回执

标准信封包含 event_id、event_type、schema_version、tenant_id、home_shard_id、aggregate_id/version、occurred_at、payload_digest。服务身份和受控来源校验后，领域还要验证请求、来源、币种、历史绑定及指纹。事件不携带 Key 明文、厂商凭证或原始提示词；媒体任务以受控资源引用传递大文件。

settlement.requested 的 payload 示例：

```json
{
  "settlement_operation_id": "settle_example",
  "request_id": "req_example",
  "reservation_id": "res_example",
  "home_shard_id": "shard_example",
  "operation_kind": "video.generate",
  "outcome": "succeeded",
  "usage_fact_id": "usage_example",
  "usage_fact_version": 1,
  "usage_status": "provider_reported",
  "usage": [{"metric": "video_seconds", "quantity": "6", "unit": "second"}],
  "pricing_snapshot": {
    "price_version": "price_example",
    "billing_rule_version": "rule_example",
    "upper_bound_version": "bound_example",
    "scope_fingerprint": "scope_example",
    "digest": "pricing_digest_example"
  },
  "currency": "USD",
  "authorized_amount": "10.000000000000",
  "slices": [{
    "grant_id": "grant_example", "binding_epoch": 4, "slice_order": 1,
    "reserved_amount": "10.000000000000"
  }]
}
```

生产契约还包含原厂商事实引用、能力参数与全部冻结预算/来源证据。Billing 不接受 Gateway 声称的最终价格代替规则计算；同 ID 不同用量版本不是覆盖原收费的许可，更正另走调整事件。

settlement.posted 的回执 payload 示例（示例规则计算 C=6）：

```json
{
  "receipt_id": "receipt_example",
  "settlement_id": "settlement_example",
  "settlement_operation_id": "settle_example",
  "request_id": "req_example",
  "reservation_id": "res_example",
  "home_shard_id": "shard_example",
  "status": "POSTED",
  "currency": "USD",
  "charged_amount": "6.000000000000",
  "slices": [{
    "grant_id": "grant_example", "binding_epoch": 4,
    "reserved_amount": "10.000000000000",
    "charged_amount": "6.000000000000",
    "released_amount": "4.000000000000"
  }],
  "receipt_digest": "receipt_digest_example"
}
```

回执由中央资金事务与 Outbox 同时生成。本地应用后 H 减 10、X 加 6、F 加 4；重复回执不再改变金额。no_execution 且本地已 RELEASED 的请求只接受零收费确认，不重复回补 E。

## 消息类型与消费确认

| 事件 | 发送方到接收方 | 确认含义 |
| --- | --- | --- |
| quota.granted | Billing 到原执行分片 | 中央已占用资金；本地激活仍须持久去重 |
| task.ready | Runtime 到媒体调度 | 任务可被检查领取，不证明已执行 |
| task.state_changed / usage.observed | Runtime 到查询/计量消费者 | 状态或用量事实，非最终扣款 |
| settlement.requested | Runtime 到 Billing | 请求按冻结规则清算，不提供最终金额权威 |
| settlement.posted | Billing 到原 Runtime | 原始结算已正式入账，可幂等应用回执 |
| quota.return_sealed / quota.returned | Runtime 到 Billing，再到 Runtime | 已封存归还意图、中央回收结果，不能相互替代 |
| policy.barrier_requested / policy.barrier_applied | 控制面到分片，再回控制面 | 禁用投影与准入有序落地的证据 |
| payment.posted | 支付模块到投影消费者 | 经核验支付已入账 |

Outbox 领取/提交、Kafka 确认、消费者 Inbox/领域提交与回执分别记录。允许至少一次投递，业务键防止不同事件 ID 造成重复收费。一个 batch_id 仅标识传输批次，不能取代逐请求幂等键。批量接口逐项返回 POSTED、RECONCILING 或 CONFLICT；受理不等于入账。

## 回调取消与未知结果

回调持久化前验证厂商签名、上游账户、provider_job_id 或已保存 client_reference 与本地尝试映射。暂未关联的可信回调进入待关联 Inbox，不能按回调自报租户写账。持久接收后再应答，重投按事件及业务终态去重。

取消接口仅在确认未发送且取消赢得本地状态竞争时返回 CANCELED/RELEASED；已提交则返回 CANCEL_REQUESTED，保留占用，待厂商核实。SUBMISSION_UNKNOWN 不重建任务。查询可信结果后，以冻结计价规则处理成功、部分交付、失败和取消。

## 撤销与错误语义

撤销提交先返回 202 REVOKING 和 operation_id。目标包括相关未结清/尚未激活授权和有效服务许可的全部逻辑分片，不能只枚举存活 Pod。新注册分片需跟到当前 epoch 后才能服务；停止新授权与记录屏障目标在控制面有序提交。所有目标分片持久应用禁用策略并确认，或有可验证隔离证据后才 REVOKED。无法确认不伪报成功；旧请求的合法历史事实仍可清算。

| 状态与错误 | 调用方行为 |
| --- | --- |
| 401 invalid_credential | 更新无效/过期凭证 |
| 403 forbidden / model_not_allowed | 调整授权，不自动换未授权资源 |
| 402 insufficient_funds | 仅在可用授权及中央可分配/安全回收路径确认不足后返回 |
| 429 rate_limit_exceeded / budget_limit_exceeded | 区分流量与已确认硬预算不足，按 Retry-After 或配置处理 |
| 409 idempotency_conflict / settlement_conflict | 保留原意图和证据，禁止换新标识绕过 |
| 503 quota_refill_pending / funds_temporarily_unavailable | 额度补充或回收暂不可确认，不误报余额为零 |
| 503 home_shard_unavailable / durability_unconfirmed | 使用原标识查询或等待恢复，不重新发起资金操作 |
| RECONCILING / SUBMISSION_UNKNOWN / CANCEL_REQUESTED | 可查询中间态，不等同失败或已退款 |

SSE 已发送响应头后的失败按协议流事件与请求状态表达，不能再假装改变 HTTP 状态或拼接另一份生成。

## 契约文件与代码生成

Hono/Zod 定义生成 business-api 与 internal-billing OpenAPI；Go 的 gateway-api/runtime-api 维护对应 OpenAPI，JSON Schema 描述事件。生成 TypeScript/Go 客户端，CI 校验契约、实现与生成差异，跨服务不共享可写 ORM 包。

中央内部 API 统一使用 /internal/v1；Gateway 私有适配器使用 /runtime/v1。公开推理、任务、门户与管理入口按各自 v1 契约实现，业务唯一键、状态转换与持久化确认语义同时纳入契约验收。
