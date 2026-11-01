# 数据库划分与数据模型

更新日期：2026-11-01。版本：10 万并发目标架构。表结构为逻辑规范，尚未执行 DDL。

数据所有权分为中央业务账与 Go 运行时子账。中央 Billing 管理钱包、资金来源、预算分配、最终结算与正式账本；Gateway 直接操作自己分片的持久额度、请求预占与任务，不直接写中央钱包。总体设计见 [10万并发架构](<AI Token SaaS 10万并发架构设计方案.md>)。

## 物理部署与逻辑归属

```text
中央 PostgreSQL 高可用部署
└── business_db
    ├── identity / access / catalog：逻辑领域
    ├── billing：账户、来源、预算、额度分配、结算、总账
    ├── payments：订单、支付与原路退款
    └── operations：变更日志、屏障、路由与可靠事件

执行 Cell 高可用 PostgreSQL 部署 x N
└── runtime_db_<cell>
    ├── runtime_shards / quota_pools：持久分片与授权子账
    ├── request_reservations / reservation_slices：请求级预占
    ├── inference_requests / tasks / provider_attempts：执行
    └── usage_facts / outbox_events / inbox_events / recovery_jobs
```

上述领域名称用于标明数据所有权。生产中央库和运行时库分开部署、角色、连接池、资源与故障策略；每个逻辑分片只有一个可写权威。开发环境复用容器资源，生产使用独立部署。

Redis 负责缓存、流量计数和短期协调，Kafka 负责可靠任务与事件通道，对象存储负责媒体文件，分析存储负责报表投影。它们不替代 PostgreSQL 的资金与幂等权威。

## 标识金额与版本约定

- 所有租户资源携带 tenant_id；项目、应用、Key、任务、资金来源与文件校验一致归属，适用时使用组合外键。
- 内部 ID 使用 UUID；HTTP 中的 req_example 等是示例。跨库引用不假设有数据库外键，通过可信授权、版本、摘要及服务端归属校验约束。
- 金额与单价使用有限精确十进制 NUMERIC(30,12)，接口用字符串；中间计算使用高精度十进制，费用上界 E 向上舍入到存储精度，最终费用 C 按冻结规则在规定阶段舍入。币种、计量单位、最小收费与舍入模式显式定义。数量超过 JavaScript 安全整数范围时也用字符串。
- 时间使用明确时间点，预算窗口使用 UTC；新准入截止、任务截止、服务许可到期和资金关闭是不同概念。
- routing_version 为路由版本，binding_epoch 为额度绑定版本，shard_write_epoch 为分片写入隔离版本，executor_epoch 为尝试领取者版本，policy_epoch 为权限生效版本，不能相互替代。
- 授权绑定持久 home_shard_id，不绑定 Pod。授权与消费事实追加或按显式状态转换更新，不从缓存重新生成余额。

## 中央身份配置与路由

| 表 | 主要字段 | 约束与用途 |
| --- | --- | --- |
| users / tenants | 用户身份引用；租户 type、status | 个人与企业租户统一归属 |
| tenant_memberships / roles / role_permissions | tenant_id、user_id、role、scope、status | 成员关系唯一，平台与租户权限分开 |
| projects / service_accounts / applications | tenant_id、project_id、principal_id、status | 企业服务身份独立于员工账户生命周期 |
| api_keys | id、tenant_id、application_id、prefix、digest、policy_id、status、expires_at | 摘要唯一，明文一次展示 |
| access_policies / access_policy_versions | scope、version、allowed_models、channels、limits | 已发布版本可追踪 |
| models / model_channels | capability_version、计量规则、credential_ref、region、status | 能力与渠道引用有版本 |
| price_versions / tenant_contracts | pricing_items、currency、effective_from/to、rule_version | 已使用版本不可原地覆盖 |
| provider_credential_refs | owner_scope、secret_ref、version | 不保存上游凭证明文 |
| routing_assignments | virtual_shard_id、home_shard_id、cell_id、routing_version、status | 路由切换与分片再均衡受控，不能重试随机换库 |
| gateway_change_log | cursor、entity、version、payload_digest | 持久增量与缺口恢复 |
| policy_revocation_operations / policy_barrier_targets | operation_id、policy_epoch、目标分片、ack、status | REVOKING 到 REVOKED 的确认屏障 |

运行时只保存执行必需投影，不复制 users 或 tenant_memberships。Client 和管理后台通过 Hono API 访问业务状态。

## 中央资金与额度分配表

| 表 | 主要字段 | 关键约束 |
| --- | --- | --- |
| billing_accounts | id、tenant_id、currency、posted_balance、delegated_balance、refund_held_balance、status | UNIQUE(tenant_id,currency)；posted >= delegated + refund_held >= 0 |
| fund_lots | id、account_id、source_type/id、balance、delegated_amount、refund_held_amount、refundable、expires_at | 来源余额 >= 分配占用 + 退款冻结，有限非负 |
| budget_windows | id、rule_id、scope、currency、window_start/end、limit_amount、consumed_amount、delegated_amount、status | UNIQUE(rule_id,window_start)；consumed + delegated <= limit |
| quota_allocation_operations | id、tenant_id、grant_request_id、fingerprint、result | UNIQUE(tenant_id,grant_request_id)，固定原分配结果 |
| quota_grants | id、allocation_operation_id、part_no、account_id、fund_lot_id、home_shard_id、binding_epoch、scope_snapshot、scope_fingerprint、allocated_amount、posted_amount、returned_amount、outstanding_amount、admit_before、status、auth_digest | 一份授权一个来源；UNIQUE(allocation_operation_id,part_no)；L=posted+returned+outstanding |
| grant_budget_allocations | grant_id、budget_window_id、allocated_amount、posted_amount、returned_amount、outstanding_amount | 联合主键；每个适用窗口都占用该授权金额，币种与范围一致 |
| quota_return_operations | return_operation_id、grant_id、binding_epoch、seal_sequence、sealed_amount、seal_digest、status | 回收操作唯一；仅接受原分片已封存金额，不能重复释放 |
| settlements | id、tenant_id、request_id、reservation_id、home_shard_id、operation_id、fingerprint、currency、charged_amount、refunded_amount、usage_ref、pricing_snapshot、status、ledger_transaction_id | 原请求/预占唯一；操作唯一；POSTED 原价与收费不覆盖 |
| settlement_slices | settlement_id、grant_id、slice_order、reserved_amount、charged_amount、binding_epoch | 与原预占切片一致；sum(C_i)=C，每项 C_i<=E_i |
| settlement_receipts | id、settlement_id、receipt_digest、payload | 每份原结算唯一原始回执，同资金事务创建 |
| ledger_accounts / ledger_transactions / ledger_entries | 科目、业务类型/ID、币种、方向、金额与关联事务 | 追加式；业务键唯一，同币种借贷平衡 |
| billing_adjustments | original_settlement_id、operation_id、amount、reason、ledger_ref、status | 消费退款/更正累计受限，原收费不可变 |

上述余额、占用、预算和授权金额字段分别约束为非空、有限、非负；不只检查若干金额相加非负。scope_snapshot 保存完整资源与预算范围，摘要用于验证内容，不能用不可还原的哈希代替授权依据。

一个分配操作可从多个来源生成多份 quota_grants，所有授权的总额才是本次分配金额。中央账户和各预算按总额占用，各来源按各自份额占用，不重复按总额累计到每份授权。

每份授权携带完整硬预算集合与可准入范围。request_reservations 不在 business_db；中央 settlements 保存其不可变结算依据副本，跨库关联以稳定 ID 和摘要验证。

## 运行时额度与请求表

| 表 | 主要字段 | 关键约束与恢复规则 |
| --- | --- | --- |
| runtime_shards | home_shard_id、routing_version、shard_write_epoch、status | 资金和执行写入都验证当前可写版本；分片再均衡先封存原写入权威 |
| quota_pools | grant_id、binding_epoch、tenant_id、scope_snapshot、scope_fingerprint、fund_lot_ref、allocated_amount、free_amount、held_amount、confirmed_charged_amount、sealed_return_amount、admit_before、status、source_digest | grant_id 唯一；L=F+H+X+R；重复激活不重置金额 |
| request_reservations | id、tenant_id、request_id、authorized_amount、held_amount、captured_amount、released_amount、currency、pricing_snapshot、policy_epoch、scope_fingerprint、status、settlement_operation_id、settlement_id | UNIQUE(tenant_id,request_id)；E=held+captured+released；一次原始结算 |
| reservation_slices | reservation_id、grant_id、slice_order、reserved_amount、held_amount、captured_amount、released_amount、binding_epoch | 固定切片顺序与归属；请求与子池变更同事务 |
| inference_requests | id、tenant_id、application_id、key_id、home_shard_id、reservation_id、operation_kind、fingerprint、execution_status、billing_status | 根逻辑调用，执行与账务状态独立 |
| idempotency_records | tenant_id、application_id、operation_kind、key_digest、fingerprint、request_id、task_id、home_shard_id、result_ref | 作用域组合唯一；相同意图固定原归属，不按内容哈希合并合法调用 |
| tasks | id、request_id、kind、status、queue_deadline、provider_job_id、next_poll_at、result_storage_status、result_ref、cancel_status | 图片/视频异步任务，受理与预占/Outbox 同事务 |
| provider_attempts | id、request_id、attempt_no、channel_id、provider_account_id、provider_request_id、provider_job_id、executor_epoch、status、dispatch_fingerprint | UNIQUE(request_id,attempt_no)；DISPATCHING/UNKNOWN 不因任务租约过期重新提交 |
| usage_facts | id、request_id、attempt_id、version、schema_version、provider_usage、normalized_usage、usage_status、content_digest | 原始与标准计量保留来源，事实变更追加版本 |
| policy_projections / serving_permits | scope、policy_epoch、status、valid_until、source_digest | 与准入有序生效，许可不足或过期停止新准入 |
| revocation_barrier_acks | operation_id、home_shard_id、policy_epoch、applied_sequence | 与本地禁用状态有序提交，完成持久确认后 ACK |
| outbox_events / inbox_events | event_id、type、aggregate/version、payload_digest、状态、游标 | 接收去重和领域写同事务，领域唯一键再防不同事件 ID 的重复副作用 |
| grant_return_seals | return_operation_id、grant_id、seal_sequence、amount、digest、status | 先本地封存 F 转 R，再请求中央回收；关闭墓碑不随缓存 TTL 删除 |
| recovery_jobs | subject_id、kind、status、lease、attempt_count、next_retry_at、last_error | 有限重试、未知结果恢复和人工核对 |

配额字段含义为：L 原始授权，F 可新预占，H 当前请求冻结，X 已应用回执的累计消费，R 已封存归还额（含待确认）。同一数据库事务维护池、预占与切片，不能让两个进程各自维护可独立消费的内存 F。

任务执行使用 QUEUED、DISPATCHING、SUBMITTED、RUNNING、SUCCEEDED、FAILED、CANCELED、SUBMISSION_UNKNOWN；文本使用 ACCEPTED、DISPATCHING、STREAMING、COMPLETED、FAILED、CANCELED、UNKNOWN。请求账务状态统一使用 RESERVED、RECONCILING、SETTLEMENT_PENDING、POSTED、RELEASED。

## 支付与退款

| 表 | 主要字段与约束 |
| --- | --- |
| topup_orders | tenant/account、amount、currency、status；商业入账意图唯一 |
| payment_attempts | order_id、provider_account_id、merchant_reference、status；未知结果显式 UNKNOWN |
| payment_transactions | provider_account_id + provider_transaction_id 唯一，保留所有真实到账事实 |
| payment_refunds | original_payment_id、operation_id、amount、currency、status、provider_ref、fingerprint；原路退款独立操作 |
| refund_holds | refund_id、account_id、fund_lot_id、amount、status；只冻结中央未分配、可退来源 |

已分配资金必须先安全回收才可参与充值款退款。消费退款增加中央来源与余额，不回写旧子池 X；是否回补原预算窗口由冻结退款规则控制。

## 余额与事件水位

posted_balance 是正式已入账余额，delegated_balance 是所有授权 outstanding_amount 的合计，refund_held_balance 是有效退款冻结。中央可再分配值不是客户全部可继续调用额度：分片 F 仍是客户可用但已委托的资金。

门户分列中央可再分配、子池可用、请求冻结、待入账与退款冻结，并返回 as_of/watermark。查询投影不作为准入依据。中央已记账但本地未应用回执时，outstanding 与 F+H 暂不相等；先核对在途回执和回收，再判断差异，静止收敛后才直接比较。

## 索引保留与隔离

| 访问路径 | 索引或约束 |
| --- | --- |
| 账户与币种 | UNIQUE(tenant_id,currency) |
| 授权申请重试 | UNIQUE(tenant_id,grant_request_id) |
| 有效额度选择 | home_shard_id、scope_fingerprint、status、admit_before，按稳定 grant_id 排序锁定 |
| 原预占与原结算 | tenant_id/request_id 与 reservation_id 业务唯一键 |
| 公共幂等 | tenant_id、application_id、operation_kind、key_digest 唯一 |
| 待调度/待轮询 | status、next_poll_at、queue_deadline，使用有界领取与固定排序 |
| 厂商任务映射 | provider_account_id、provider_job_id 唯一（适配器范围内） |
| Outbox/Inbox | 可投递状态、next_retry_at、event_id；按领域业务键额外去重 |
| 租户调用与流水 | tenant_id、时间、稳定 ID 游标索引 |

按 Cell 和查询负载做物理分片，按时间做适当分区。未清算事实、有效 H、未确认 R、幂等墓碑和关闭授权不得随内容日志 TTL 清除；归档前验证结算闭合和可追踪性。

行级安全可作为纵深隔离，运行角色、表所有者与 BYPASSRLS 的行为需明确，连接池租户上下文使用事务范围设置。[PostgreSQL 行级安全](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)

详细事务、状态和金额守恒见 [账务设计](04-billing-and-ledger.md)，跨系统调用与事件见 [接口契约](05-api-contracts.md)。
