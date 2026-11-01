# 账务与额度子账设计

更新日期：2026-11-01。版本：10 万并发目标架构。中央 Billing 统一定价与记账，Go 执行分片维护已分配额度内的请求预占。本文为事务与不变量规范，不代表已经实现或压测。

本方案采用预充值、单账户单币种、请求费用上界预占与一次原始最终结算，覆盖文本流式、图片和视频异步任务。总体结构见 [高并发架构](<AI Token SaaS 10万并发架构设计方案.md>)，表字段见 [数据库设计](03-database-design.md)。

## 权威与金额定义

| 对象 | 权威与更新入口 |
| --- | --- |
| 中央钱包、资金来源、预算分配、原始结算与总账 | Hono 工程的 Billing 模块，HTTP 和清算 Worker 复用同一实现 |
| 额度子池、请求预占、执行权与原始用量事实 | Go 执行分片，以本地持久事务更新 |
| 最终客户收费 C | Billing 根据准入冻结规则和可靠事实计算，Go 和前端不另写一套最终收费算法 |
| 客户采购成本与客户收费差异 | 每次上游尝试记录采购事实；平台重试、失败和超上界差异分别核算 |

所有金额有限、非负并显式带币种；方向用业务类型或分录表达。金额和单价使用精确十进制 NUMERIC(30,12)，JSON 使用字符串；禁止 JavaScript number 或 Go 浮点参与资金计算。中间计算使用高精度十进制，上界 E 向上舍入到存储精度，最终 C 按冻结规则在规定阶段舍入，禁止逐 Token 舍入。数量带单位，单位换算、最小扣费与舍入模式固定到规则版本。[PostgreSQL 数值类型](https://www.postgresql.org/docs/current/datatype-numeric.html)

冻结 pricing_snapshot 包含模型/能力版本、销售价格、计量分类、舍入规则、上界算法版本、输出/数量/时长上限、允许渠道范围和客户收费结果规则。输入、缓存、推理 Token 的子集语义不得重复相加。上界 E 按可验证的保守规则计算，最终 C 按可信用量计算，正常支持的计价形态要求 0 <= C <= E。

发现计算费用超 E 时保留 RECONCILING，记录原始用量、采购成本与差额，禁止无新授权超扣或覆盖原授权。无法形成可靠上界的形态需先有明确授权机制；缺失计量按公开裁决规则处理，不静默把估算当精确收费。

## 中央账户与预算不变量

```text
P = billing_accounts.posted_balance
D = billing_accounts.delegated_balance
H_refund = billing_accounts.refund_held_balance
中央可再分配金额 U = P - D - H_refund
P >= D + H_refund；P、D、H_refund 均有限且非负
D = sum(该账户全部 quota_grants.outstanding_amount)

预算：consumed_amount + delegated_amount <= limit_amount
delegated_amount = sum(关联授权在该窗口的 outstanding_amount)
资金来源：balance >= delegated_amount + refund_held_amount
```

中央 D 已覆盖子池未用额度、在途请求和待清算金额。不能再把所有运行时 held_amount 从 P 中重复扣除。资金来源余额和占用合计与对应账户投影一致。

预算 consumed_amount 是按规则回补后的净预算消费；原始授权 posted_amount 和原始收费保持不变。退款预算回补通过关联原窗口的调整记录表达。调低限额不得小于已消费与未结清分配之和，可先阻止新准入并回收未用额度；新增硬预算或变更范围须发布新 policy_epoch/scope，旧子池不能继续为新范围准入。

## 授权与子池不变量

```text
中央一份授权：L = C_posted + R_posted + O
L = allocated_amount；O = outstanding_amount
C_posted = posted_amount；R_posted = returned_amount

本地一份子池：L = F + H + X + R
F = free_amount；H = held_amount
X = confirmed_charged_amount
R = sealed_return_amount（含等待中央确认的封存归还）

请求：E = held_amount + captured_amount + released_amount
切片：sum(E_i) = E；每片 E_i = held_i + captured_i + released_i
```

每个金额分量均非负。X 只随中央消费回执增加，消费退款不修改它。R 封存后不能重新加入 F，回收确认丢失也不能本地撤回。相同 grant_id 的重放不能再次初始化额度。零费请求可使用 E=0 且无金额切片，但仍需有效身份/服务许可和零元结算记录，任何非零收费须有有效授权。

授权绑定一个资金来源和完整预算窗口集合。一个分配操作可生成多份授权；一个请求可用多个同分片、同租户、同币种且作用域完全匹配的授权切片。以冻结的 slice_order 将 C 分配为 C_i，sum(C_i)=C 且每项 0<=C_i<=E_i。

中央清算可能先于本地回执应用：此时 C_posted >= X；封存回收可能先于中央确认：此时 R >= R_posted。按一致水位比较时：

```text
O = F + H - (C_posted - X) + (R - R_posted)
```

原始实时读数未对齐水位时不直接比较。静止收敛后 C_posted=X、R_posted=R、O=F+H。任何差额先追踪在途回执和回收，不将差额补成新资金。

## 分配授权事务

分配接口以 tenant_id + grant_request_id 幂等，原请求指纹包括账户、范围、目标分片和期望总额。重试返回原授权列表。

```text
BEGIN（中央 business_db）
  按固定顺序锁账户、原预算窗口与资金来源
  复核租户/Key/策略、来源资格、币种与可授权范围
  检查分配操作幂等：同意图返回原结果，不同载荷冲突
  检查 U、全部预算空间和全部来源空间足够
  创建 allocation_operation；按来源拆出 quota_grants
  账户 D += 本次授权总额
  每个适用预算 delegated += 本次授权总额
  各来源 delegated += 分配给该来源的金额
  写 grant_budget_allocations、原结果和 quota.granted Outbox
COMMIT 并取得所需同步持久化确认
```

每份授权同一来源、同一作用域，向下发放的预算路径不能遗漏租户/项目/Key 硬限额。领取、激活确认或新准入截止超时都不能自动撤销中央占用。

Go 在本地事务中校验授权来源、摘要、home_shard_id、binding_epoch 与范围，按 grant_id 幂等创建 quota_pools。授权重放只能读取现有结果；内容不同、旧快照或已关闭墓碑冲突时隔离，不能把 F 重置为 L。

## 请求预占与执行权

```text
BEGIN（同一 runtime_db 分片）
  校验当前 shard_write_epoch、有效 serving_permit、policy_epoch
  通过公共幂等作用域确定原 request_id 和 home_shard_id
  检查原请求指纹，校验当前访问身份与资源归属
  锁定完整授权范围匹配的子池，按 grant_id 固定顺序
  校验新准入截止、来源/窗口规则、允许模型/渠道与 F >= 所需份额
  冻结 pricing_snapshot、窗口、上界 E 和授权切片 E_i
  各池 F -= E_i，H += E_i
  创建请求、request_reservations、reservation_slices
  异步任务同时创建 QUEUED task 与 task.ready Outbox
COMMIT 并取得持久化确认
```

文本可在已准备好执行时将预占与取得发送资格合并为一个短事务。异步任务排队后，由 Worker 在本地事务中把任务转为 DISPATCHING、预占转 STARTED，并绑定 attempt_id/executor_epoch。上游 HTTP 在持久化确认后、数据库事务外执行。

新的领取者可以接管明确未发送任务；DISPATCHING、UNKNOWN、SUBMISSION_UNKNOWN 不因 Worker 领取租约过期而自动重发。存储 fencing 约束旧写者的新状态变更，但不能撤回已发送的厂商请求；恢复必须查询或保留未知状态。

## 请求与任务状态

| 对象 | 状态及约束 |
| --- | --- |
| request_reservations.status | HELD、STARTED、RECONCILING、AWAITING_SETTLEMENT、SETTLED、RELEASED |
| HELD | 未取得发送资格；与 start 竞争成功的取消/队列截止可释放 |
| STARTED | 已授予本地发送资格；不得因 TTL、断流或 Pod 消失直接释放 |
| RECONCILING | 提交/执行/用量不明确，继续保留 H |
| AWAITING_SETTLEMENT | 用量与结算意图已持久化，等待中央回执，继续保留 E |
| SETTLED | 已应用正式回执，captured=C、released=E-C、held=0 |
| RELEASED | 确认未发送，captured=0、released=E，旧执行者不能再发送 |
| 请求 billing_status | RESERVED、RECONCILING、SETTLEMENT_PENDING、POSTED、RELEASED，与执行状态独立 |

未发送取消：在本地同一事务确认仍 HELD 且未取得发送资格，任务/请求转取消，H 减 E、F 加 E、切片和预占转 RELEASED，写不可变 no_execution 事实与零收费结算意图。中央零元回执仅确认记录，不再释放一次，billing_status 保持 RELEASED。

已执行但最终按规则零收费：按正常清算出具 C=0 回执，执行过的预占转 SETTLED，billing_status 为 POSTED，全部 E 释放；可以没有收费账本分录，但必须有零元结算记录。

图片/视频从受理到排队、生成、回调和清算都保留约束；取消请求不等于已取消或不收费。文件保存失败不重做生成，执行结果、文件状态与账务状态分别维护。

## 中央清算事务

结算事件提供 request_id/reservation_id、home_shard_id、稳定 settlement_operation_id、用量依据、冻结规则与完整 E_i 切片。Billing 校验历史合法授权，不因当前 Key 已撤销或授权处于 DRAINING 而拒绝合法历史费用。

最终价格只由 Billing 的版本化计价实现计算；用量缺失保留待核对，不提前释放。批处理按账户和固定锁顺序处理有界数量，仍保留每请求原始结算与分录。冲突事件隔离，不能静默跳过或覆盖。

```text
BEGIN（中央 business_db）
  去重事件和原始收费业务键，核验原请求/预占/授权/事实指纹
  锁账户、授权、原预算窗口和资金来源
  按冻结规则计算 C，校验 0 <= C <= E，并分配 C_i
  创建原 settlements、settlement_slices；非零 C 写平衡消费分录
  账户 P -= C，D -= C
  每份授权 posted += C_i，outstanding -= C_i
  每个原预算 consumed += C，delegated -= C
  各原来源 balance -= C_i，delegated -= C_i
  写 settlement_receipts 与 settlement.posted Outbox
  Inbox 去重、业务状态、余额、分录和回执同事务提交
COMMIT 并取得持久化确认
```

一个窗口可能关联多个切片，按该窗口涉及的 C_i 合计更新，不对切片重复按整笔 C 累加。相同 request/reservation 即使换了事件 ID 或操作 ID，也不能产生第二份原结算；同操作不同事实返回冲突并核对。

中央账户 P 与 D 同减 C，因此清算本身不增加中央可再分配 U。剩余 E-C 仍在原子池中可用，只有安全回收才释放中央委托占用。

## 应用结算回执

```text
BEGIN（原 home_shard 的 runtime_db）
  校验中央来源、receipt_id/digest、原预占、E_i/C_i 与历史绑定
  去重 Inbox 和领域 receipt/settlement 业务键
  若已本地确认未执行 RELEASED：只接受 C=0，登记确认，不改池金额
  否则各切片 H -= E_i，X += C_i，F += E_i-C_i
  原预占 held=0、captured=C、released=E-C、status=SETTLED
  更新请求 billing_status=POSTED、settlement_id 和回执水位
COMMIT 并确认
```

重复回执不再释放金额。回执丢失时重投或按原操作查询，未确认前 H 保留。中央已有收费与本地 H 仍未消除属于有证据的中间态，不是再补一次余额的理由。

## 回收与额度关闭

本地 ACTIVE 转 DRAINING 后停止新预占。事务中将可归还 q 从 F 移至 R，生成递增 seal_sequence 和稳定 return_operation_id、封存摘要及 Outbox。此后不能因 HTTP 超时把 q 放回 F。

中央验证历史授权、封存来源与未重复的回收操作，在同一事务中 returned += q、outstanding -= q、账户 D -= q，并释放对应来源与原预算 delegated；写 quota.returned 回执。两边确认后记录回收水位，R 不再减少。

仍在 H 的金额、未核实消耗和未应用回执必须保留。DRAINING 期间回执释放出的 F 可再次封存形成新的回收操作。只有 H=F=0、X+R=L、中央 outstanding=0 且回收/结算全部确认后关闭。新准入截止不是资金关闭时间。

已关闭授权和幂等墓碑须能阻挡晚到授权消息。分片数据丢失或从旧快照恢复时先 QUARANTINED，不能据旧 F 再准入；核对中央授权、消费和在途证据后才恢复，无法核实的原额度继续占用。

## 正式账本与支付

ledger_transactions 使用 UNIQUE(tenant_id,business_type,business_id)，ledger_entries 的金额严格正数且币种与事务一致。每个非零事务至少两条分录、同币种借贷合计相等。已入账记录不修改删除，更正追加反向或调整事务。

跨行平衡由统一记账入口与必要的事务结束约束实现，不能用单行 CHECK 声称保证。[PostgreSQL 约束触发器](https://www.postgresql.org/docs/current/sql-createtrigger.html)

充值使用稳定商户引用：先持久化订单/支付尝试，外部调用在事务外；回调或查询核验签名、商户、交易号、订单、币种和金额。保存支付事实、充值分录、P/来源余额与通知同事务。支付事件 ID、渠道交易号和商业入账意图分别去重；同一订单多笔真实到账全部保留，额外款项进入差异处理。超时或晚到成功不按本地失败覆盖外部事实。

## 三类释放与退款

| 操作 | 资金变化 | 约束 |
| --- | --- | --- |
| 未发送请求释放 | 子池 H 转 F，中央 P/D 不变 | 与发送资格原子竞争；不能对已启动未知任务使用 |
| 消费退款 | 中央 P 与对应来源增加，追加退款分录 | 累计不超原 C，X 和原收费不改；后续经新授权消费 |
| 充值款原路退款 | 先冻结中央未分配且可退款来源，外部成功后 P/来源余额减少 | 先回收已委托未用金额，不动在途 H；外部 UNKNOWN 保持退款冻结 |

消费退款如回补预算，记录原窗口及金额，不把旧消费退到新窗口；不覆盖原始授权 posted_amount。来源过期、赠送额度与手续费分别执行公开规则。支付争议、后付费与负余额风险另建模型，不通过篡改既有预付账消除差异。

## 示例与对账

下表为同一示例币种，一份 L=100 的授权和一笔 E=20、C=12 的请求；U 是中央可再分配金额。

| 阶段 | P | D | U | F | H | X | R |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 账户有余额 | 1000 | 0 | 1000 | — | — | — | — |
| 分配 100 | 1000 | 100 | 900 | 100 | 0 | 0 | 0 |
| 请求预占 20 | 1000 | 100 | 900 | 80 | 20 | 0 | 0 |
| 中央入账 12，回执在途 | 988 | 88 | 900 | 80 | 20 | 0 | 0 |
| 应用回执 | 988 | 88 | 900 | 88 | 0 | 12 | 0 |
| 本地封存归还 88 | 988 | 88 | 900 | 0 | 0 | 12 | 88 |
| 中央确认回收 | 988 | 0 | 988 | 0 | 0 | 12 | 88 |
| 后续消费退款 2 | 990 | 0 | 990 | 0 | 0 | 12 | 88 |

退款后原收费仍为 12，退款为 2，净收费 10；旧池 X 仍为 12，不能再次消费。门户区分中央可再分配、子池可用、请求冻结和待入账，避免将委托资金显示为已消费。

对账核验：P 与总账净余额一致；D 与中央授权 O 合计一致；每个预算 consumed 与净预算消费一致；来源余额及占用与账户一致；每份子池与切片守恒；回执/回收按同一水位收敛；每笔账本借贷平衡；原收费减退款等于净收费。

## 持久化与故障确认

额度、执行权、记账和回收的成功确认须达到目标同步持久化级别；不能用 Redis 副本或普通异步主从声称资金不丢。故障时只提升包含已确认资金状态的安全副本，无法证明则停止新付费准入。[PostgreSQL/Patroni 复制约束](https://patroni.readthedocs.io/en/latest/replication_modes.html)

COMMIT 结果未知时，普通 SELECT 的可见记录不足以证明已完成复制确认。任何会触发授权派发、上游发送或资金回执的重放，需经新的幂等确认写或经验证的持久化屏障，达到同级确认后再产生外部副作用。Outbox Relay 的派发领取标记也先通过该确认，避免发出尚不安全的授权/回执。

Key/租户冻结停止新授权，并通过分片确认屏障阻止新的本地准入；合法历史结算继续执行。具体接口、回执及错误语义见 [接口契约](05-api-contracts.md)，真实数据库和故障验收见 [可靠性方案](07-reliability-and-validation.md)。
