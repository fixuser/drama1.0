# Drama 2.0 后端实施蓝图

## 1. 文档定位

本目录定义 Drama 2.0 的后端实施方案，是面向空库、全新 API 和全新部署的工程蓝图。它不是 1.0 的兼容改造说明，也不是把 `splay` 代码搬到新目录的任务清单。

本轮重构的目标是：让领域边界更清楚、资金写入可证明、热路径更短、异步任务可恢复，并让 Proto、Schema、事务和测试使用同一套业务语义。

适用优先级从高到低为：

1. 已确认的 2.0 决策，包括本目录的固定决策；
2. 本目录各实施文档；
3. `../requirements-1.0/` 中的保留/排除业务矩阵；
4. `../../../max/shortmax` 可复用的架构模式；
5. `../../splay` 仅作为现状证据和反例。

发生冲突时必须按上述优先级处理，并通过 ADR 修正文档；不得以“旧代码已经这样做”为理由覆盖 2.0 决策。

## 2. 固定范围

| 主题 | 2.0 结论 |
| --- | --- |
| 数据起点 | 空库发布；只执行新 Schema 迁移，不迁移 1.0 用户、资产、关系和历史 |
| API | 全新 Proto + gRPC-Gateway；公共端和管理端均使用 `/v2`，无旧接口代理 |
| 资产 | 仅 `COMMON` 通用资产和 `EXPERIENCE` 体验资产 |
| 支持值 | 完全删除，不建余额、流水、兑换、接口或统计类型 |
| 用户关系 | 只使用十进制 `user_id`；不生成、不存储、不接收 F 码 |
| 等级 | 保留 L0–L7 业务门槛，重做枚举、聚合、计算和审计 |
| 一致性 | 同库事实由 `database.default` 的一个事务提交 |
| 只读库 | `database.slave` 只服务允许延迟的查询，不参与判断和状态推进 |
| 异步可靠性 | 不建通用 Outbox；业务状态可重建，primary 扫描兜底 |
| 错误 | 锁定 Forge `v0.4.1`；Domain 无 Forge 依赖，Service 映射 `errkit`，Transport 统一 Proto/gRPC/HTTP 编解码 |
| 消息 | Forge `message` 只作事务外 SMS/Email transport；primary 保存 delivery/challenge、lease/sweep/retry，成功只表示 `PROVIDER_ACCEPTED`，无可靠证据的不确定调用转 `UNKNOWN` 且不自动重发；无 WhatsApp |
| Callback | 供应商经固定字面量原始 HTTP 路由、raw-body/headers 验签、内部 typed command 和 provider ACK；不建 callback Proto |
| HTTP 路由 | 不含路径参数；GET 标识放 query，POST 标识放 Proto body |
| 历史兼容 | 无旧 Proto、旧枚举、F 码、历史 cohort、双写或适配层 |

## 3. 范围矩阵

| 领域 | 保留并重构 | 明确排除 |
| --- | --- | --- |
| 用户与安全 | 注册登录、OTP、KYC、银行卡、注销、后台 RBAC；提现条件 TOTP | 提现专用人脸、超级密码、旧认证兼容、活动资格钩子 |
| 关系与团队 | `user_id` 上下级、完整祖先链、五层派生边、团队查询 | F 码、字符串关系数组、VIP 邀请活动 |
| 资产 | 双资产、不可变流水、冻结总额与冻结明细、对账 | 支持值、活动币、直接散写余额 |
| 内容 | 短剧、剧集、播放授权、购买、收藏、历史、弹幕、Banner | AI 聊天、版权包、二创、活动电影票 |
| 短剧参与与机器人 | `Playlet Backing`、`Robot Purchase`/持仓/收益、`Robot Activation Voucher` | 普通短剧支持券、双倍券、FlashSale、订阅机器人 |
| 等级与收益 | L0–L7、活跃值、五层返佣、合伙人、真实收益统计 | 活动加成、虚拟金额、任务阻断合法收益 |
| 支付 | 原额充值、一次性 VIP、提现和渠道对账 | 充值赠送、自动续订、充值匹配/翻倍 |
| 运营 | 白名单任务、配置发布、真实统计、审计与告警 | 付费 TaskPlan、全部活动任务和消费者 |

## 4. 文档导航

| 文档 | 作用 |
| --- | --- |
| [00-设计决策与术语.md](00-设计决策与术语.md) | 决策、术语、不变量和规则解释原则 |
| [01-系统架构与工程边界.md](01-系统架构与工程边界.md) | 进程、模块、依赖、事务和部署边界 |
| [02-API与鉴权规范.md](02-API与鉴权规范.md) | Proto、HTTP、认证、权限、错误、幂等和接口清单 |
| [03-数据模型总览.md](03-数据模型总览.md) | 全量表、外键、索引、枚举和迁移规范 |
| [04-用户关系与安全.md](04-用户关系与安全.md) | 用户、安全、KYC、银行卡、RBAC 和关系链 |
| [05-资产冻结与事务.md](05-资产冻结与事务.md) | 双资产、统一变动原语、冻结、并发和对账 |
| [06-等级团队与返佣.md](06-等级团队与返佣.md) | L0–L7、聚合、五层返佣和合伙人 |
| [07-短剧内容与参与.md](07-短剧内容与参与.md) | 内容授权、购买、`Playlet Backing`、弹幕和 Banner |
| [08-机器人持仓券与任务.md](08-机器人持仓券与任务.md) | 机器人、持仓、收益、机器人券和任务 |
| [09-充值VIP与提现.md](09-充值VIP与提现.md) | 充值、一次性 VIP、提现及渠道状态机 |
| [10-统计作业配置与运维.md](10-统计作业配置与运维.md) | Rollup、年度/累计收益趋势、月末百分位、作业、配置、缓存、审计和运维 |
| [90-明确排除项.md](90-明确排除项.md) | 不进入 2.0 的表、接口、任务、配置和统计类型 |
| [98-测试与性能验收.md](98-测试与性能验收.md) | 正确性、安全、故障和性能硬门禁 |
| [99-实施分期与交付门禁.md](99-实施分期与交付门禁.md) | P0–P7 依赖、交付物、出口和回滚 |

## 5. 1.0 逐章映射

| `requirements-1.0` | 2.0 处理 | 实施真源 |
| --- | --- | --- |
| `01-robot.md` | 保留普通/等级机器人，规则快照化 | [08](08-机器人持仓券与任务.md)、[05](05-资产冻结与事务.md) |
| `02-barrage.md` | 保留弹幕，补审核闭环 | [07](07-短剧内容与参与.md) |
| `03-short-drama.md` | 保留内容、授权、购买、收藏和历史 | [07](07-短剧内容与参与.md)、[02](02-API与鉴权规范.md) |
| `04-playlet-support.md` | 历史需求在 2.0 改造为 COMMON `Playlet Backing`；删除支持值和普通支持券 | [07](07-短剧内容与参与.md)、[06](06-等级团队与返佣.md) |
| `05-robot-support.md` | 保留持仓、等级、五层返佣、合伙人 | [06](06-等级团队与返佣.md)、[08](08-机器人持仓券与任务.md) |
| `06-wallet.md` | 重做为双资产账户、流水与冻结 | [05](05-资产冻结与事务.md) |
| `07-income-statistics.md` | 使用白名单事实和可重建日/月发布代；提供月账单、年度/累计趋势和固定人口口径的月末百分位 | [02](02-API与鉴权规范.md)、[03](03-数据模型总览.md)、[10](10-统计作业配置与运维.md)、[98](98-测试与性能验收.md) |
| `08-banner.md` | 服务端投放过滤、稳定排序和安全跳转 | [07](07-短剧内容与参与.md) |
| `09-support-coupon.md` | 历史需求只改造保留为 `Robot Activation Voucher` | [08](08-机器人持仓券与任务.md)、[90](90-明确排除项.md) |
| `10-task.md` | 保留白名单任务；奖励与业务收益解耦 | [08](08-机器人持仓券与任务.md) |
| `11-user.md` | 全新用户、安全、关系与注销实现 | [04](04-用户关系与安全.md) |
| `12-payment-withdrawal.md` | 原额充值、显式价 VIP、统一冻结提现 | [09](09-充值VIP与提现.md) |
| `90-excluded-features.md` | 空库下直接不建、不注册、不暴露 | [90](90-明确排除项.md) |
| `91-questions-and-optimization.md` | 建议默认值全部固化为 2.0 正式规则 | [00](00-设计决策与术语.md) 及对应领域文档 |
| `92-interface-data-job-inventory.md` | 只作范围检查；旧 API/表/任务名不沿用 | [02](02-API与鉴权规范.md)、[03](03-数据模型总览.md)、[10](10-统计作业配置与运维.md) |

### 5.1 `requirements-1.0/91` Q-001～Q-024 决策闭环

下表将 91 章的建议默认值固化为 2.0 正式规则。若建议中的“历史履约/历史快照”与 2.0 空库、无历史迁移固定决策冲突，以固定决策为准；此类条目直接排除历史入口，不保留空壳兼容实现。

| 编号 | 2.0 正式规则 | 实现文档 | 验收/排除依据 |
| --- | --- | --- | --- |
| Q-001 | 机器人按 `sale_price` 扣 COMMON；返本、个人/团队活跃值和合伙人业绩均按购买时固化的 `principal_amount` 快照，二者差异必须在商品发布审核中明示。 | [08 机器人持仓](08-机器人持仓券与任务.md)、[06 合伙人](06-等级团队与返佣.md) | [98 §8](98-测试与性能验收.md) 分别核对售价扣款、本金返本/活跃值/业绩，改配置不得改历史持仓。 |
| Q-002 | 当前等级 `>= required_level` 才结算等级机器人收益；降级只暂停未来业务日，不追回已结算收益、不补发暂停日，恢复资格后继续。 | [08 机器人持仓](08-机器人持仓券与任务.md)、[00 术语](00-设计决策与术语.md) | [98 §8](98-测试与性能验收.md) 覆盖门槛上下界、暂停、恢复和不追溯。 |
| Q-003 | `Robot Activation Voucher` 持仓的本金固定为 0；不返名义本金、不增加活跃值、不产生团队返佣或合伙人池，只按券规则产生直接收益。 | [08 机器人券](08-机器人持仓券与任务.md)、[90 排除矩阵](90-明确排除项.md) | [98 §8](98-测试与性能验收.md) 验证券持仓终结及乱序重放始终没有返本/返佣/pool。 |
| Q-004 | 2.0 完全删除支持值；`Playlet Backing` 只有 COMMON 现金路径并按其已发布规则结算。 | [00 固定决策](00-设计决策与术语.md)、[07 短剧参与](07-短剧内容与参与.md)、[90 排除矩阵](90-明确排除项.md) | [90 §4–§8](90-明确排除项.md) 要求无支持值 Schema/API/枚举/作业。 |
| Q-005 | `Playlet Backing` offer 使用版本化 `amount_mode=FIXED/MINIMUM` 和 `unit_amount`，不再硬编码“恰好 50”。 | [07 短剧参与](07-短剧内容与参与.md) | [98 §8](98-测试与性能验收.md) 覆盖份额、容量、持仓数和并发边界。 |
| Q-006 | 任务只决定任务奖励；不得阻断合法的 Backing、持仓、直接收益、返佣或提现。 | [07 参与结算](07-短剧内容与参与.md)、[08 任务](08-机器人持仓券与任务.md) | [98 §8](98-测试与性能验收.md) 验证任务未完成仍正常结算。 |
| Q-007 | 由空库决策覆盖“暂停新发、履约历史”：普通短剧支持券在 2.0 完全排除，无规则、历史、发放、领取或核销入口；只保留 `Robot Activation Voucher`。 | [08 机器人券](08-机器人持仓券与任务.md)、[90 排除矩阵](90-明确排除项.md) | [90 §2、§8](90-明确排除项.md) 和 [98 §14](98-测试与性能验收.md) 要求旧券表、路由、任务及历史读取器均不存在。 |
| Q-008 | 一次性 VIP 只使用后台已发布商品版本的显式价格；不保留首月 1 单位、VIP 优惠券或任何隐式首购促销。 | [09 充值与 VIP](09-充值VIP与提现.md)、[90 排除矩阵](90-明确排除项.md) | [98 §8](98-测试与性能验收.md) 验证订单/权益规则快照和容量；[90 §5](90-明确排除项.md) 检查无隐式促销分支。 |
| Q-009 | 一次性 VIP 保留中性任务 `PURCHASE_ONE_TIME_VIP`，且只能由事实 `ONE_TIME_VIP_ENTITLEMENT_CREATED` 推进；`SUB_VIP` 和连续订阅任务完全删除，无历史解释层。 | [08 任务](08-机器人持仓券与任务.md)、[09 一次性 VIP](09-充值VIP与提现.md)、[90 排除矩阵](90-明确排除项.md) | [98 §8](98-测试与性能验收.md) 覆盖真实来源、重复/乱序去重并扫描全仓无 `SUB_VIP`。 |
| Q-010 | 付费 `TaskPlan` 不进入 2.0，不建套餐、购买、激活或资格能力；任务仅来自已发布白名单定义。 | [08 任务](08-机器人持仓券与任务.md)、[90 排除矩阵](90-明确排除项.md) | [90 §2、§8](90-明确排除项.md) 与 [98 §14](98-测试与性能验收.md) 验证无表、API、配置、任务和消费者残留。 |
| Q-011 | 合伙人基础池率固定为合格现金 `Playlet Backing` 或 `Robot Purchase` 本金的 5%；事实字段统一为 `pool_base_amount`，系数 `0.3/0.5/0.7/1.0` 分别表示该池的 30%/50%/70%/100%，不是 0.3% 等百分数。 | [00 术语](00-设计决策与术语.md)、[06 合伙人](06-等级团队与返佣.md) | [98 §7](98-测试与性能验收.md) 对基础 5% 池和四档系数逐项验算，并验证每个来源只入池一次。 |
| Q-012 | 不保留 2026-06-12 的硬编码起算分支；合伙人规则由不可变版本的 `effective_at/expires_at` 决定，新库事实保存命中的版本和快照。 | [06 合伙人](06-等级团队与返佣.md)、[10 配置版本](10-统计作业配置与运维.md) | [98 §7](98-测试与性能验收.md) 覆盖非月初生效、月界和版本封账；[90 §2](90-明确排除项.md) 排除历史迁移。 |
| Q-013 | 首发等级规则把 L2 作为版本化合格门槛；不保留 2026-01-30 日期分支、旧 L2 枚举映射、历史保底或 cohort，新产生的等级/返佣事实各自保存规则版本。 | [06 等级规则](06-等级团队与返佣.md)、[00 固定决策](00-设计决策与术语.md)、[90 排除矩阵](90-明确排除项.md) | [98 §7](98-测试与性能验收.md) 覆盖 L2 阈值、规则换版和历史事实不回写；[98 §14](98-测试与性能验收.md) 扫描兼容残留。 |
| Q-014 | `team_enabled=false` 只停用团队展示及新增团队权益，不清空关系、不禁止登录；只有账号 `status=DISABLED/CLOSED` 才禁止登录。 | [04 用户与能力状态](04-用户关系与安全.md)、[00 术语](00-设计决策与术语.md) | [98 §6](98-测试与性能验收.md) 分别验证团队能力禁用、账号禁用和登录行为。 |
| Q-015 | 一次经过服务端验真的广告完成事件永久解锁指定剧集，并创建用户/剧集唯一、可审计的权益；不实现 N 分钟临时解锁。 | [07 广告权益](07-短剧内容与参与.md)、[00 术语](00-设计决策与术语.md) | [98 §8](98-测试与性能验收.md) 覆盖重复、伪造、错集事件及账号关闭并发，确保不多建权益。 |
| Q-016 | Banner 由服务端按 placement 数量上限、`priority` 稳定排序、有效期、语言、区域、平台和应用版本定向；客户端不能自行扩大候选集或绕过安全跳转。 | [07 Banner](07-短剧内容与参与.md)、[02 API 规范](02-API与鉴权规范.md) | [98 §8、§12](98-测试与性能验收.md) 验证时间边界、排序/上限、定向和安全跳转。 |
| Q-017 | 弹幕命中敏感词时保存为 `SELF_VISIBLE_REVIEW`，仅作者可见并进入审核队列；规则不可用时同样失败收紧为待审，不公开。 | [07 弹幕](07-短剧内容与参与.md) | [98 §8](98-测试与性能验收.md) 覆盖本人可见、审核终态、举报/屏蔽和公开列表不可泄漏。 |
| Q-018 | 放大播放量/销量只能作为独立 `heat` 展示，文案不得称真实人数、销量、播放量或金额；真实播放与财务统计只消费可验证白名单事实。 | [00 术语与不变量](00-设计决策与术语.md)、[07 内容](07-短剧内容与参与.md)、[10 统计](10-统计作业配置与运维.md) | [98 §8、§13](98-测试与性能验收.md) 验证 heat 不写真实播放事实、不进入收益或财务 rollup。 |
| Q-019 | 由空库决策删除 2025-08-08 历史 cohort；任务资格只由不可变规则版本的生效区间和目标人群决定，不对换版前事实追溯补进度。 | [08 任务规则](08-机器人持仓券与任务.md)、[90 排除矩阵](90-明确排除项.md) | [98 §8](98-测试与性能验收.md) 验证无适用版本/人群不匹配的稳定忽略和换版不追溯；[98 §14](98-测试与性能验收.md) 扫描 cohort 残留。 |
| Q-020 | 全系统业务时区固定为 `Asia/Jakarta`，周一 00:00 为周起点；日/周/月均使用左闭右开区间，不读取服务器本地时区作业务裁决。 | [00 固定决策](00-设计决策与术语.md)、[10 BusinessClock](10-统计作业配置与运维.md) | [98 §2、§9](98-测试与性能验收.md) 固定跨日/跨周/月界时钟并验证封账与重跑一致。 |
| Q-021 | 由空库决策覆盖“历史其他”：不导入、不展示也不统计 1.0 历史活动收益；2.0 累计收益仅消费白名单 COMMON 事实，旧活动类型显式排除。 | [10 收益口径](10-统计作业配置与运维.md)、[90 排除矩阵](90-明确排除项.md) | [98 §13–§14](98-测试与性能验收.md) 验证收益守恒、未知/活动类型失败关闭且无历史读取入口。 |
| Q-022 | 2.0 新库没有已关闭活动的遗留资格或负债，不实现旧活动履约、退款、作废审批或人工导入路径；未来若产生同类外部义务，必须独立立项和 ADR。 | [00 固定决策](00-设计决策与术语.md)、[90 排除矩阵](90-明确排除项.md) | [90 §3、§8](90-明确排除项.md) 和 [98 §14](98-测试与性能验收.md) 要求活动表、资格、消费者、补发入口及迁移代码为零。 |
| Q-023 | 注销先把账号转为 CLOSING 并禁止新业务；持仓、资产余额/冻结、在途提现及所有未来可入账 blocker 必须在 primary 上终结或清退，全部归零后才能原子转 CLOSED。 | [04 注销状态机](04-用户关系与安全.md)、[05 资产冻结](05-资产冻结与事务.md)、[09 提现](09-充值VIP与提现.md) | [98 §6](98-测试与性能验收.md) 覆盖各 blocker、并发新来源、重复清退和百万历史行下的有界完成。 |
| Q-024 | 银行卡户名按完整姓名规范化，支持单名并覆盖 Unicode、大小写、空白、标点和 locale 差异；明确一致且供应商核名成功才自动 VERIFIED，模糊结果进入人工复核。 | [04 银行卡核名](04-用户关系与安全.md)、[09 提现前置](09-充值VIP与提现.md) | [98 §6、§12](98-测试与性能验收.md) 使用规范化向量、供应商回写、人工复核权限和提现绑定场景验收。 |

## 6. 保留能力端到端覆盖矩阵

| 保留能力 | `/v2` API | 主要事实表 | 事务/作业真源 | 硬验收 |
| --- | --- | --- | --- | --- |
| 注册登录与安全 | Auth/Security/User | `users`、`user_credentials`、`session_families`、`user_sessions`、`session_issuances`、`otp_bindings`、`user_security_audits` | [04](04-用户关系与安全.md) 注册/会话事务 | [98](98-测试与性能验收.md) §6、§12 |
| KYC、银行卡、注销与提现条件 TOTP | KYC/Bank/User/Security | `kyc_cases`、`otp_bindings`、`bank_directory_entries`、`bank_accounts`、`account_closures`、两类 closure disposition、`account_closure_channel_slots` | [04](04-用户关系与安全.md) 安全状态机/有界清退事务；提现不做专用人脸，TOTP 从未成功确认可跳过、ACTIVE 必验，曾确认但当前 REVOKED（含 reset）须重绑或受控恢复 | [98](98-测试与性能验收.md) §5–§6、§12 |
| 邀请关系与团队 | Referral | `users`、`referral_edges`、`referral_audits`、`user_relation_stats` | [04](04-用户关系与安全.md) 绑定单事务 | [98](98-测试与性能验收.md) §6 |
| 钱包、流水、冻结、兑换与纠错 | Asset | `user_assets`、`asset_ledgers`、`asset_freezes`、`asset_exchanges`、`asset_adjustments`、`finance_recovery_cases` | [05](05-资产冻结与事务.md) 四个统一原语/兑换/受限纠错事务 | [98](98-测试与性能验收.md) §3–§4 |
| L0–L7 与活跃值 | Level | `active_value_ledgers`、`user_levels`、`level_change_audits` | [06](06-等级团队与返佣.md) 同步增量 + dirty worker | [98](98-测试与性能验收.md) §7、§11 |
| 五层返佣与合伙人 | Rebate/Partner | `team_rebate_settlements`、`partner_months`、`partner_month_shards`、`partner_month_sources`、`partner_adjustment_heads`、`partner_pool_facts`、`partner_pool_allocations`、`partner_settlements`、`partner_settlement_adjustments` | [06](06-等级团队与返佣.md) 来源结算/有界月度封账 | [98](98-测试与性能验收.md) §7、§13 |
| 短剧目录与播放 | Playlet | `playlets`、`episodes`、`play_authorization_audits`、`play_credential_issuances`、`video_provider_events`、`play_events` | [07](07-短剧内容与参与.md) 单一授权/账号代次短凭证/可验证播放事实 | [98](98-测试与性能验收.md) §8 |
| 剧集购买、退款、返佣与广告权益 | Entitlement | `episode_purchase_orders`、`episode_refund_requests`、`episode_entitlements`、`team_rebate_settlements`、`ad_verification_attempts`、`ad_provider_events`、`ad_unlock_entitlements` | [07](07-短剧内容与参与.md) 混合扣款/退款窗口与到期 sweep/返佣冻结/广告验真事务 | [98](98-测试与性能验收.md) §4、§7–§8 |
| 收藏与观看历史 | Library | `favorites`、`watch_histories` | [07](07-短剧内容与参与.md) 用户内容事务 | [98](98-测试与性能验收.md) §8 |
| 短剧参与与收益 | Playlet Backing | `playlet_backings`、`partner_pool_facts`、`playlet_backing_settlements` | [07](07-短剧内容与参与.md) COMMON Backing/一次入池/日结算 | [98](98-测试与性能验收.md) §4、§8、§13 |
| 弹幕与举报审核 | Barrage | `barrages`、`barrage_reports`、`barrage_blocks` | [07](07-短剧内容与参与.md) 创建/审核事务 | [98](98-测试与性能验收.md) §8、§12 |
| Banner 投放 | Banner | `banners`、`banner_events` | [07](07-短剧内容与参与.md) 发布版本/缓存失效 | [98](98-测试与性能验收.md) §8、§12 |
| 机器人购买、持仓与返本 | Robot | `robot_products`、`user_robot_product_stats`、`robot_holdings`、`robot_qualification_consumptions`、`partner_pool_facts`、`robot_holding_settlements` | [08](08-机器人持仓券与任务.md) O(1) 现金限购根/一次入池/日结算/终结事务 | [98](98-测试与性能验收.md) §4、§8、§13 |
| 机器人激活券 | Robot Activation Voucher | `robot_voucher_rules`、`robot_voucher_batches`、`robot_voucher_codes`、`user_robot_vouchers`、`robot_voucher_redemptions`、`robot_voucher_dispositions`、`robot_qualification_consumptions` | [08](08-机器人持仓券与任务.md) 用户级锁/分发/关户处置/跨入口资格核销 | [98](98-测试与性能验收.md) §8 |
| 任务与奖励 | Task | `user_tasks`、`task_progress_events`、`task_source_consumption_roots`、`task_period_source_barriers`、`task_reward_claims`、`task_compensations` | [08](08-机器人持仓券与任务.md) 真实来源事实/64 分片 seq barrier/领取/自然键纠错事务 | [98](98-测试与性能验收.md) §4、§8–§9 |
| 原额充值 | Deposit | `deposit_orders`、`payment_attempts`、`payment_provider_events` | [09](09-充值VIP与提现.md) 单 intent COLLECT/REFUND、迟付 finality、回调履约与 primary sweep | [98](98-测试与性能验收.md) §5、§9、§13 |
| 一次性 VIP | VIP | `vip_orders`、`payment_attempts`、`user_vip_stats`、`vip_entitlements`、`team_rebate_settlements` | [09](09-充值VIP与提现.md) 容量预占、支付、`REFUND_NOT_ACCEPTED`、权益、返佣和任务同事务 | [98](98-测试与性能验收.md) §5、§7–§8 |
| 提现与出款 | Withdrawal | `withdrawals`、`withdrawal_limit_reservations`、`withdrawal_reviews`、`payout_attempts`、`payout_channel_switches`、`payout_failure_resolutions`、`payout_success_dispositions/events`、`finance_case_source_reversals` | [09](09-充值VIP与提现.md) 冻结/双窗口 reservation/最多 8 次切换/失败收敛/成功集合对账事务 | [98](98-测试与性能验收.md) §3–§5、§13 |
| 收益概览、月账单与年度/累计趋势 | Income：Overview/Series/MonthlyBill/AnnualSummary/Details | `user_income_stats`、`finance_business_days`、`income_day_publications`、`income_daily_rollups`、`income_hourly_rollups`、`income_month_publications`、`income_month_generations`、`income_month_shards`、`income_monthly_rollups` | [10](10-统计作业配置与运维.md) exact primary tail、`finalize_income_day`、`build_income_month` 和 selected generation 原子发布 | [98](98-测试与性能验收.md) §9、§11、§13：累计月趋势分页、年度 12 月槽守恒及月代有界构建 |
| 月末超过用户百分位 | Income：Percentile | `user_population_roots`、`users.registration_seq`、`income_percentile_heads`、`income_percentile_publications`、`income_percentile_inputs`、`user_income_percentiles` | [10](10-统计作业配置与运维.md) 人口 cutoff barrier、依赖 proof/sweep、最早脏月级联重建、`build_income_percentile`、独立 INPUT/RANK 验证与按 head/as_of 派生清理 | [98](98-测试与性能验收.md) §6、§9、§11、§13：0/99/100 样本、并发注册、旧指针原子换代、过去月依赖传播、十年窗口有界性和 Redis 丢失 |
| 统计版本、对账、导出与运维 | Report/Admin Ops | `rollup_roots`、`rollup_versions`、`income_calc_policy_segments`、`income_restatement_scope_proofs/shards`、genesis `finance_business_days`、`income_calc_dirty`、`rollup_activation_leaves/shards`、`rollup_reconciliations`、`job_runs`、`job_failures`、`report_exports` | [10](10-统计作业配置与运维.md) 空库 bootstrap、扁平重述/双 cutoff scope、版本激活屏障、对账、primary sweep 和固定口径异步导出 | [98](98-测试与性能验收.md) §9、§11–§13 |
| 站内通知与外围提醒 | Notification | `notifications`、`notification_deliveries` | [10](10-统计作业配置与运维.md) 来源可重建、应用级去重、明确未接受重试与不确定结果收敛 | [98](98-测试与性能验收.md) §9、§12 |

## 7. 参考、改造与拒绝

| 分类 | 来源 | 2.0 用法 |
| --- | --- | --- |
| 参考 | `../../../max/shortmax` | 采用领域边界、Proto 驱动、顶层事务、default/slave 分工、资产账户/流水和可恢复 worker 模式 |
| 改造 | `../../../max/shortmax` | 写原语收紧为只接收 Tx；资产增加通用冻结明细；关系改为纯 user ID；等级/返佣/配置补齐结构化版本和审计 |
| 参考业务范围 | `../requirements-1.0` | 仅用于决定保留、重构或排除的产品能力；91 的建议默认值在 2.0 正式化 |
| 拒绝 | `../../splay` | 只作现状证据和反例；不复制宽表资产、F 码、错位等级、活动钩子、请求触发结算和分散事务 |

## 8. 文档使用方式

开发前先确定所属领域和顶层 Use Case，再从数据模型、事务、API 和测试四个方向核对。任何保留能力都必须同时具备：

- 一个明确的 `/v2` API 或内部作业入口；
- 一组有约束的事实表；
- 一条唯一事务路径和幂等键；
- 一组权限、指标、对账与验收用例。

任何排除能力都必须同时没有公共写入口、管理写入口、表、枚举、配置、缓存、消费者、定时任务和默认统计分类。实现评审以 [98-测试与性能验收.md](98-测试与性能验收.md) 为硬门禁，以 [99-实施分期与交付门禁.md](99-实施分期与交付门禁.md) 为交付顺序。

## 9. 变更治理

- 金额、资格、等级、收益或外部支付语义变更必须新增 ADR，并标明规则版本、生效时间、回滚方式和受影响测试。
- Proto 枚举和数据库约束同时评审；不能只改前端文案或运行时配置。
- 禁止引入第二套资金路径、旧接口代理或通用 Outbox 作为“临时方案”。
- 本目录只描述新库 Schema 和新系统发布；任何未来数据导入、跨库投递或兼容需求均须独立立项。
