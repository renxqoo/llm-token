# AI Token SaaS 最终技术选型方案

更新日期：2026-11-01。版本：10 万并发目标架构。

管理后台 React + Vite、Client Next.js、业务后端 TypeScript + Hono、LLM Gateway Go 为平台核心技术。系统覆盖文本非流式/SSE、图片/视频异步与准确扣费；Go 通过进程内额度模块在持久子池完成请求准入，中央 Billing 按批授权并异步清算。

技术实现以 [高并发架构](<AI Token SaaS 10万并发架构设计方案.md>) 为总体目标，资金、表和接口分别以 [账务](04-billing-and-ledger.md)、[数据库](03-database-design.md)、[接口](05-api-contracts.md) 为详细规范。本文锁定组件职责，具体兼容版本在工程初始化时固定；10 万并发仍需联合验收。

## 核心技术与运行职责

| 部分 | 选型 | 责任 |
| --- | --- | --- |
| 管理后台 | React + TypeScript + Vite | 租户、渠道、价格、订单、异常授权、财务与运营页面 |
| Client | Next.js App Router + TypeScript | 官网、模型与价格、个人/企业门户、文本 Playground、图片视频任务与结果 |
| 业务后端 | TypeScript + Hono，Node.js LTS | 身份、策略、定价、中央钱包、额度分配、支付、清算与总账 |
| Gateway 数据平面 | Go + net/http + chi | 鉴权投影、持久额度执行、文本流、任务引擎、适配器与恢复 |

Gateway 不管理用户注册、成员与角色编辑。中央 Billing 拥有正式资金账；Go 拥有被中央资金担保的额度子账及请求预占，不能直接写中央钱包。最终收费算法统一在 Billing，Go 执行发布的上界规则和资金约束。

## 前端配套

| 领域 | 管理后台 | Client |
| --- | --- | --- |
| 页面与路由 | React/Vite + TanStack Router | Next.js App Router，不叠加 TanStack Router |
| 请求与缓存 | TanStack Query | 服务端按请求读取与客户端 TanStack Query 按页面组合 |
| 组件与样式 | Ant Design，表单优先 Ant Design Form | shadcn/ui + Tailwind CSS |
| 身份 | 独立平台会话与资源权限 | 用户会话、企业切换与服务端成员校验 |
| 发布 | 静态资源与路由回退 | 独立 Next.js Node.js 服务，支持按页面渲染 |

公开官网、说明和模型页可预生成或受控缓存。余额、预算、任务、企业成员和账单按用户及租户读取，不进入跨租户共享缓存。Next.js 的页面代理或 Server Actions 通过 Hono API 执行业务，不获得业务数据库写入权限。[服务端/客户端组件](https://nextjs.org/docs/app/getting-started/server-and-client-components)、[Next.js 部署](https://nextjs.org/docs/app/getting-started/deploying)

任务页面分别显示执行、文件与账务状态，余额展示中央可再分配、子池可用、请求冻结和待入账等有水位的投影。文本流直接访问 Gateway；浏览器使用短期受限授权，生产应用在服务端保存长期 Key。

## Hono 业务工程

业务工程模块化组织 Identity、Tenancy、Access、Catalog、Pricing、Quota Allocation、Billing、Payments、Usage 与 Operations。HTTP 层校验输入、提取身份与映射结果；领域模块与 Hono 请求对象解耦，HTTP 和 Worker 共用同一资金实现。

| 领域 | 配套技术与边界 |
| --- | --- |
| 运行时 | Node.js LTS + Hono Node Adapter；API、分配、清算、支付与报表独立配置资源 |
| 校验与契约 | Zod + @hono/zod-openapi，生成并检查业务与内部 OpenAPI |
| 数据访问 | Drizzle + node-postgres，资金事务使用显式 SQL 与固定锁顺序 |
| 迁移 | Drizzle Kit 生成可审阅 SQL；复杂约束与锁语义显式实现 |
| 认证 | [Better Auth 接入 Hono](https://better-auth.com/docs/integrations/hono)，统一会话认证；平台领域维护用户/租户/角色，管理员与用户会话隔离 |
| 清算 | 消费 Kafka 结算意图，在 Billing 中计算 C、原子入账并生成回执 Outbox |
| 工作区 | Bun workspaces 管理 TypeScript 工程与依赖；生产目标运行时仍为 Node.js |

Hono 的 [Node.js 运行方式](https://hono.dev/docs/getting-started/nodejs)、[Zod OpenAPI 集成](https://hono.dev/examples/zod-openapi) 和 [Drizzle 事务](https://orm.drizzle.team/docs/transactions) 提供框架能力；多级预算、来源、幂等、上界与金额平衡由领域与数据库约束落实。

Billing 仍处理每笔最终收费明细，但可按账户做有界小批清算、合并余额更新，逐请求保留结算与分录。异步化减少首 Token 路径依赖，不消除最终明细写入量。

## Go Gateway 工程

| 模块 | 配套技术与约束 |
| --- | --- |
| HTTP 与流式 | net/http + chi，连接复用、请求取消、有界缓冲、流空闲/总截止时间与优雅排空 |
| 额度执行 | pgx + sqlc，在持久分片内更新子池/预占/切片，确认提交后启动外部执行 |
| 媒体任务 | 有界 Worker、公平调度、持久领取、回调 Inbox 与 next_poll_at 查询恢复 |
| 厂商适配 | 协议、模型能力、提交/查询/取消与计量映射分离；未知提交不盲目重发 |
| 数据访问 | pgx + sqlc，按授权与固定业务键查询，数据库锁保护资金子账 |
| 迁移 | goose SQL 迁移，由 Gateway 维护所有运行时分片的同版本模板 |
| 消息 | Outbox Relay 与 Kafka 兼容客户端，消费者 Inbox 与域级唯一键共同去重 |
| 日志与指标 | 关联 request/task/grant/reservation/attempt/settlement/shard，元数据与内容日志分开 |

[sqlc 支持 pgx](https://docs.sqlc.dev/en/latest/guides/using-go-and-pgx.html)。Go module 独立管理依赖，不跨服务导入可写中央钱包的数据访问包。额度执行统一为 internal/quota 进程内模块，Gateway、媒体与恢复 Worker 复用；私有运行时适配器调用同一实现，正常准入无需网络跳转。

SSE 正文不经过 Kafka 或每 Token 账务 RPC。媒体任务在资金与任务事务确认后返回 202，不为每个待完成视频保留长连接或忙轮询协程。任务、文件保存与清算有独立状态。

## 数据存储与消息

| 组件 | 生产用途 | 所有权 |
| --- | --- | --- |
| 中央 PostgreSQL HA | business_db：账户、来源、预算、quota_grants、结算、支付与平衡总账 | Hono Billing 及对应业务模块 |
| 运行时 PostgreSQL HA x N | runtime_db_<cell>：quota_pools、request_reservations、切片、任务、用量与可靠事件 | Go 执行分片 |
| Kafka | 任务调度、用量、清算意图、回执与配置/屏障事件 | 按主题与 Cell 控制生产消费能力 |
| Redis | 内存外缓存、速率与并发控制、短期协调 | 分用途命名空间，不作为唯一资金权威 |
| 对象存储 | 图片、视频、客户文件与按策略保留内容 | 服务端租户与下载授权 |
| ClickHouse | 用量、费用、延迟、成本报表投影 | 可重建分析数据，不参与准入或正式账本写入 |

生产中央库与运行时分片独立资源、角色、连接池与 HA 策略；开发环境可共用容器资源。库内是否分 schema 属于实现组织，不代表物理隔离。单表迁移与写入入口只有一个所有者。

金额与单价使用 NUMERIC(30,12)、显式单位和规则版本，API 以字符串传递；上界与最终费用的舍入规则按账务规范执行。价格/计价与预算归属在准入冻结，每请求预占存 Go 运行时分片，中央按批占用额度并接收异步清算。

Kafka 配置至少核验多副本、最小同步副本数、acks=all 与生产者幂等；资金幂等仍靠数据库。Redis 默认异步复制，不以 WAIT 或缓存锁代替资金持久化语义。[Kafka 确认](https://kafka.apache.org/41/configuration/producer-configs/)、[Redis 复制](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)

## API 与工程组织

| 契约 | 定义来源与用途 |
| --- | --- |
| business-api | Hono/Zod 定义，门户与管理 SDK |
| internal-billing | Hono 的 /internal/v1，额度分配、回收和异步清算 |
| gateway-api | Go 对外模型兼容范围与平台任务 API |
| runtime-api | Go Cell 内领域语义，激活、预占、启动、释放、回执与屏障 |
| events | JSON Schema，事件版本、业务标识、摘要和兼容规则 |

HTTP 使用 OpenAPI，按定义生成 TypeScript/Go 客户端并校验漂移。接口定义同时约束请求状态、幂等、持久化确认与外部副作用语义。

```text
apps/admin-web/                    React + Vite + Ant Design
apps/client-web/                   Next.js App Router + shadcn/ui
services/business/                 Hono、中央分配、Billing/支付/配置 Worker
services/llm-gateway/               Go module
  cmd/gateway/
  cmd/media-worker/
  cmd/recovery-worker/
  cmd/outbox-relay/
  internal/quota/
  internal/tasks/
  internal/providers/
  db/migrations/
packages/api-client/               生成的 TypeScript SDK
packages/shared-config/
contracts/business-api/
contracts/gateway-api/
contracts/internal-billing/
contracts/runtime-api/
contracts/events/
deploy/                            Kubernetes、HA、本地开发与监控配置
docs/
```

[Bun workspaces](https://bun.com/docs/pm/workspaces) 管理 TypeScript 包，Go module 维护 Go 依赖。共享客户端、工程配置、品牌资源和稳定格式化规则，不共享可跨库写资金的 ORM。两个前端各自使用适合其页面的组件。

## 生产运行与版本

生产目标为单区域多可用区、多个执行 Cell 与 Kubernetes 编排。Docker Compose 用于开发和小规模联调，不作为十万并发的生产容量承诺。

Gateway 按活跃流、内存、带宽和准入延迟扩容，媒体 Worker 按任务、厂商配额及公平性扩容。生产需接入对应扩缩容指标，参考 [Kubernetes HPA](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)。部署先停止新流、排空与有界退出；新副本不会迁走原有 SSE。额度按低/目标水位预取并限制单租户闲置占款。

主库提升必须保护已确认的额度与收费；同步复制、fencing 和确认未知提交的流程共同验证，不使用有损提升维持付费准入。Outbox 派发与执行权确认也要取得所需持久化保证。[复制限制](https://patroni.readthedocs.io/en/latest/replication_modes.html)

Hono 不可用时，Cell 仅在有效服务许可、既有授权和安全积压水位内继续新准入；运行时库无法确认写入则停止新付费执行。清算暂停会保留 H 并逐步消耗空闲额度，不提前放开冻结。

依赖和镜像锁定具体版本；提交 Bun 锁文件、go.mod/go.sum，核验 Next.js/React、Hono/Zod、Drizzle/驱动及 Go 客户端兼容。升级须经过契约、SQL、资金故障、回调和 SSE 验收。

## 开发与验收顺序

D0 确立契约和资金不变量，D1 身份/路由/屏障，D2 分配/子账/回执/安全回收，D3 文本执行，D4 图片与视频，D5 支付账单，D6 三组十万容量和资金故障联合验收，D7 采购增强与区域扩展。详见 [开发路线](06-development-roadmap.md)。

TypeScript 使用 Vitest，Go 用 testing/httptest，客户旅程用 Playwright；金额、幂等和并发使用真实 PostgreSQL，流控使用真实 Redis，消息链路验证 Kafka 和回执。假体覆盖可控故障，真实厂商联调验证实际计量/查询/取消保证；两者不能相互替代。[完整验收](07-reliability-and-validation.md)

四项核心技术和高并发架构基线已明确。持续 QPS、单企业峰值、P99、上游采购容量、币种/舍入、退款条款与灾备参数按 [实施参数](08-decisions-and-open-questions.md) 固定，不以未测数字冒充能力。

## 官方实现参考

- [New API 后端](https://github.com/QuantumNous/new-api/blob/main/go.mod) 与 [前端依赖](https://github.com/QuantumNous/new-api/blob/main/web/package.json)：Token 平台与 Go/React 工程参考。
- [Bifrost 核心](https://github.com/maximhq/bifrost/blob/dev/core/go.mod) 与 [UI](https://github.com/maximhq/bifrost/blob/dev/ui/package.json)：Go Gateway 与 React/Vite，引用其 dev 分支；仓库基准不代表本方案含持久化资金链的结果。
- [Midday API](https://github.com/midday-ai/midday/blob/main/apps/api/src/index.ts) 与 [数据库依赖](https://github.com/midday-ai/midday/blob/main/packages/db/package.json)：Hono 与 Drizzle 的工程参考。
- [Langfuse 架构](https://langfuse.com/handbook/product-engineering/architecture)：事务、Worker、分析与对象存储的分工参考。
