# 开发顺序与交付计划

更新日期：2026-11-01。版本：10 万并发目标架构。分阶段实现用于控制依赖，文本流式、图片/视频异步、持久额度与准确扣费共同构成对外收费上线的完整交付范围。

## 开发依赖与验收

| 阶段 | 工作范围 | 前置依赖 | 完成条件 |
| --- | --- | --- | --- |
| D0 契约与工程 | 核心四栈、Cell/分片模型、中央与子池不变量、费用上界、三类任务、中央内部接口、事件与持久化确认语义 | 已确认目标与所有权 | 契约可校验，资金/路由/未知结果场景有明确验收 |
| D1 身份与控制面 | 个人/企业、项目/应用/Key、角色、版本化策略/价格、路由归属、服务许可与撤销屏障 | D0 | 跨租户拒绝；同意图原分片重放；撤销完成不依赖单次缓存通知 |
| D2 额度与中央清算 | 钱包/来源/预算、批次分配、本地子池/预占/启动/释放、Outbox/Inbox/Kafka、结算回执与安全回收 | D1 | 两层守恒、幂等激活/收费/回执/回收、同步确认故障窗口通过 |
| D3 文本执行 | 一个上游非流式/SSE、执行权、协议错误、用量事实、断流与未知恢复；Client Key/Playground/费用状态 | D2 | 无 Hono 逐请求准入依赖；真实内容转发；部分交付与收费关联正确 |
| D4 图片与视频 | 平台任务 API、持久受理、公平调度、厂商提交、回调/轮询、取消、对象存储与分项计价 | D2，可与 D3 部分并行 | 图片和视频各至少一个明确适配器；重复回调、提交未知、长任务跨周期与文件失败正确 |
| D5 支付与财务闭环 | 支付渠道、订单、查询补单、来源原路退款、消费退款、对账、账单与运营页面 | D2–D4 | 到账/收费唯一；已分配资金不能同时退款；三类任务账单完整 |
| D6 容量故障联合验收 | 多 Cell、多可用区、生产 HA 配置、热点企业、三组 10 万负载、混合任务、备份/恢复、发布排空与回滚 | D3–D5 | 容量数据与资金差异同时达标，厂商能力和采购额度有证据 |
| D7 企业与区域扩展 | SSO/SCIM、审批、协议价扩展、BYOK、专属资源、授信、跨区域与额外模型协议 | 已稳定目标链路 | 新能力有授权、计量、预算与恢复契约；采购必需能力可提前 |

D2 是所有付费执行的前置条件。先用受审计测试入账或支付沙箱验证资金链路，真实支付联调从同一 Billing 入口入账。D0 固定分片、资金与接口契约，D2 实现持久额度和中央清算，D3/D4 在同一准入模型上实现三类执行。

## 三系统协作

管理业务系统交付身份、价格、额度分配与统一清算；Go Gateway 交付持久子账、流式与媒体任务；两个前端围绕可用接口逐步接入。Client 的媒体进度、文件状态和账务状态分别展示，管理端能查询异常授权、在途回执与对账差异。

门户/管理 API、额度分配入口与清算 Worker 复用 Hono 工程领域代码，按职责分进程运行并配置独立资源和连接池预算。Gateway API、媒体 Worker、恢复 Worker 和 Outbox Relay 按 Cell 分进程部署；额度执行作为 Go 进程内领域模块复用，各进程保持明确的 home_shard 归属。

## 工程目录

```text
apps/admin-web/                   React + Vite
apps/client-web/                  Next.js App Router
services/business/                Hono 与 Billing/支付/配置 Worker
services/llm-gateway/              Go module
  cmd/gateway/
  cmd/media-worker/
  cmd/recovery-worker/
  cmd/outbox-relay/
  internal/quota/
  internal/tasks/
  internal/providers/
  db/migrations/
contracts/business-api/
contracts/gateway-api/
contracts/internal-billing/
contracts/runtime-api/
contracts/events/
deploy/                           Cell、HA、本地环境与可观测性配置
docs/
```

各系统维护自己的 SQL 迁移，contracts 共享稳定 API/事件，不共享能跨系统写资金表的 ORM 包。生产采用独立中央数据库和运行时分片；Compose 只用于开发与联调，生产编排按 [技术选型](<AI Token SaaS 最终技术选型方案.md>) 落地。

## 阶段交付证据

每阶段记录环境与版本、接口/状态差异、真实数据库行为、失败注入步骤和结果。D6 单独记录稳态新请求 QPS、活跃流、媒体在途数、延迟分布、带宽与内存、数据库等待、清算积压、金额对账和故障域余量。

目标设计见 [高并发架构](<AI Token SaaS 10万并发架构设计方案.md>)；详细故障与容量验收见 [可靠性方案](07-reliability-and-validation.md)，业务参数见 [实施参数](08-decisions-and-open-questions.md)。
