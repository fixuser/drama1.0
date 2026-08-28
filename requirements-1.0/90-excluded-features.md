# 90 排除功能与残留依赖

## 1. 排除原则

目标1.0明确不提供 AI聊天、版权包、自动续订VIP和任何活动。排除不是只隐藏H5入口，而是同时满足：

1. 不再创建资格、订单、奖池、券、任务、流水、通知或统计数据；
2. 用户端和管理端不再提供可操作入口；
3. 公共接口不允许新写，可在兼容期返回稳定的“功能已关闭”；
4. 消息消费者、定时任务和核心流程中的活动钩子停止；
5. 历史数据、已支付权益和财务流水只读保留，不能物理删除或改写；
6. 统计默认排除这些类型，但可在审计报表中按“历史排除项”筛选。

特别例外：弹幕虽然位于 `/v1/chat/bullet/*` 仍保留；`AI Invest` 机器人和历史“AI礼券”分别改称“智能收益机器人”“机器人支持券”并保留。不能按目录名或关键词批量删除。

## 2. 总体排除矩阵

| 领域 | 排除内容 | 历史处理 | 必须保留的近邻能力 |
| --- | --- | --- | --- |
| AI聊天 | 角色、提示词、会话、消息、模型生成、聊天举报 | 会话/消息按合规只读或归档 | 剧集弹幕创建、查询、举报 |
| 版权包 | 购买/领取、激活、押金、每日结算、素材、二创、合作协议 | 用户包、结算、押金和审计只读 | 普通短剧目录、播放及运营素材 |
| 自动续订VIP | 自动扣费、订阅回调、状态同步、转移、连续订阅任务 | 已扣款权益履约到期；历史订单可查 | 一次性VIP卡购买、激活、到期、剧集解锁 |
| VIP邀请 | 开启、资格、计数、领取、消息和定时检查 | 历史奖励只读 | 普通邀请关系和合法基础返佣 |
| FlashSale/AI-X订阅 | 抢购场次、机会、邀请解锁、订阅机器人 | 历史持仓履约到期 | 智能收益机器人普通购买与持仓 |
| 双倍/充值赠送 | 双倍工资、双倍券、充值翻倍、匹配、套餐 `giveaway` | 历史订单保留原始实付/赠送 | 原始金额充值和普通支持券历史 |
| AI/超级嘉年华 | AI Carnival、invite carnival、mega/super subsidy等 | 历史资格与奖励只读 | 普通机器人收益和基础返佣 |
| 宝藏计划 | `treasure_plan`、Pool Hadiah/33宝藏奖池 | 历史抽奖/奖池只读 | 钱包和基础团队关系 |
| 其他活动 | Ramadan、抽奖、补贴、33系列、周年庆、支持奖励等 | 同上 | 核心任务、支持、合伙人基础规则 |

## 3. AI聊天

### 3.1 用户端入口

- H5 `/chats`、`/chats/accompany/:roleId`、`/chats/record`。
- 聊天角色、会话、消息、伴聊和记录的 store、类型、文案与通知跳转。
- 短剧播放页的弹幕UI不能移除。

### 3.2 接口与服务

排除 `ChatCompletion`、`ListChatRole`、`ListChatSession`、`CreateChatSession`、`RemoveChatSession`、`ListChatMessage`、`CreateChatMessage`、`RemoveChatMessage`、`ReportChatRole` 及管理端提示词/会话/消息接口。

这些方法与弹幕共存在 `splay/services/user/chat.go`、`splay/apis/userpb/user.proto` 中。后续裁剪必须按 RPC 方法，不删除整个文件/服务。

### 3.3 管理端与数据

- 排除 `admin/src/pages/PromptWords/`、`ConversationRecords/`、用户详情中的“虚拟聊天会话记录”入口和相应权限。
- 历史模型包括 ChatRole、ChatPrompt、ChatSession、ChatMessage、ChatRoleReport 等，只读保留或按隐私策略归档。
- 停止模型供应商调用、提示词缓存、聊天消息通知和聊天相关任务。

### 3.4 验收

聊天入口、RPC和模型调用不可用；已有聊天数据未被误删；三个弹幕RPC仍正常工作，管理弹幕权限独立于版权/聊天权限。

## 4. 版权包

### 4.1 用户端与管理端

排除版权包购买/激活、用户版权包、版权素材、二创上传/审核、合作协议及对应教程/下载入口。需检查 H5 `/materials`、`/tutorials/downloadMaterial`、分销协议页面是否仅为版权用途；普通短剧素材页不能误删。

管理端排除：`CopyrightPacks`、用户发放测试版权包、用户版权包修改、版权素材和二创审核，以及用户/团队中的 `is_disabled_copyright` 操作项。

### 4.2 RPC、模型和任务

排除 `taskpb` 中：

- `ListCopyrightPack`、`CreateUserCopyrightPack`、`GetUserCopyrightPack`
- `CreateCooperationAgreement`、`GetCooperationAgreement`
- `ListCopyrightMaterial`
- `CreateSecondCreation`、`ListSecondCreation`、`DeleteSecondCreation`
- `ListCopyrightPackSettlement`、`ActivationCopyrightPack`

历史模型：`CopyrightPack`、`UserCopyrightPack`、`CopyrightMaterial`、`SecondCreation` 及结算/协议记录。`checkCopyrightPackExpire`、`checkCopyrightPackSettlement` 当前虽有注释注册，仍需确认没有其他启动脚本或人工入口。

### 4.3 混合依赖

- `ListUserTaskV2` 仍查询有效版权包；目标任务列表删除该依赖。
- 注册、用户详情、团队禁用状态、任务计划和财务流水枚举中存在版权字段。
- 版权包流水必须从目标收益分类移出，但历史资产退还/罚没仍需财务审计。

## 5. 自动续订VIP与VIP邀请

### 5.1 自动续订排除

- Apple/Google订阅收据和续订状态处理、其他订阅支付回调。
- `Order_STATUS_PAYMENT_AUTO` 新写入、连续订阅创建、`TransferUserPlan`。
- H5订阅协议、自动续订文案和连续订阅引导。
- `CONTINUE_SUB_VIP_30` 等连续订阅任务；`SUB_VIP` 必须拆分后仅可表示一次性VIP购买。
- 钱包、团队和统计中把 `PAYMENT_AUTO` 当作当前收入/业绩的查询。

### 5.2 一次性权益保留边界

- `Plan` 商品列表、一次性订单、支付完成创建 `UserPlan`、排队、激活、到期和付费剧集解锁保留。
- 已成功扣款的存量自动续订权益履约至现有到期日，停止下一次扣款。
- 历史自动订单、退款和合法返佣可查，不进入目标1.0新增业绩。

### 5.3 VIP邀请活动排除

排除 `StartVipInviteActivity`、`GetVipInviteActivity`、`GetVipInviteConfig`、`ReceiveVipInviteActivity`，以及 `checkVipInviteActivityCron`、`checkVipInviteUserActivity`、注册/支付/KYC中的邀请活动钩子、活动通知和 `TYPE_VIP_INVITE_ACTIVITY` 收益。

H5 activity store会全局加载VIP邀请配置/信息，需停止，避免隐藏页面后仍调用接口。

## 6. 充值赠送、双倍与匹配活动

### 6.1 固定套餐赠送

- `PointPack.Giveaway` 对新配置固定为0。
- `ProcessRebates` 履约充值时只增加 `PointPack.Amount`，不增加 `Amount+Giveaway`。
- H5充值弹窗不显示赠送币、最大赠送比例或活动倒计时。
- 历史订单和套餐快照仍显示当时的原始赠送，审计总入账不重算。

### 6.2 双倍券/双倍工资

排除普通支持双倍券的可领取、领取、列表、管理CRUD、`coupon:double`消费者和支持收益倍数分支；排除 `/activity/doubleWages`、双倍工资统计/通知与对应流水类型。

普通短剧支持券不属于双倍券，其历史与目标边界见 [09-support-coupon.md](09-support-coupon.md)。

### 6.3 充值匹配/双倍赠送

排除 Deposit Match/Deposit Bonus、RechargeDoubleGift、充值刮奖、充值后跳转/弹窗，以及支付完成钩子。H5 `usePaymentEnd` 当前在支付成功后创建Ramadan房间或跳充值奖励成功页，目标必须只返回普通订单结果页。

## 7. FlashSale 与 AI-X订阅

排除：

- H5 `/robots/aixFlash`、Vote页 `AixFlash` 和订阅机器人卡片/预购弹窗；
- `/v1/tips/flash_sale/state`、`/v1/tips/flash_sale/grab`；
- `Robot_TYPE_SUBSCRIPTION` 的新展示和购买；
- `AiXSubscriptionState`、邀请记录、活动配置、机会数和首次升L2钩子；
- 机器人购买中的机会预锁定/消耗分支；
- FlashSale场次、订单/配额缓存和活动任务。

保留 `Robot_TYPE_AI_INVEST` 的普通智能收益产品，不保留“订阅/抢购机会”活动。历史订阅机器人持仓仍结算、返本和审计。

## 8. 全部其他活动清单

下列均默认排除，名称不完整或后续发现的新活动也按同一原则处理：

| 活动族 | H5/运营名称 | 典型服务/数据依赖 |
| --- | --- | --- |
| 朝拜/VIP邀请 | `pilgrimage`、Worship | VIP邀请配置、资格、奖励、通知 |
| 升级/帮扶/超级嘉年华 | `subsidy`、`subsidyII`、`assistance`、mega subsidy、`super_carnival` | 等级升级钩子、补贴记录、定时发放；`super_carnival` 精确wire映射需确认 |
| 抽奖 | `lottery`、L3 lottery | 报名、名次、发币、排行榜 |
| Ramadan | 祈祷、卡券、排行、GroupBuy/TeamRoom | 房间创建/加入/退款/抽奖、祈祷奖励、22:00关房退款任务 |
| AI活动 | `aiCarnival` | 机器人收益/资格加成、通知、统计 |
| 宝藏计划 | `poolHadiah`、`treasure_plan`、33宝藏 | 奖池增长、抽奖机会、每日结转、WebSocket |
| 33系列 | `33Invite`、`33Challenge`、`33Cooperation` | 邀请、挑战、合作名额、奖励和管理统计 |
| 合伙人活动页 | `partnerPlan`、`partnerPlanII` | 活动任务/等级/Mock；基础合伙人账务仍保留 |
| 周年庆 | `anniversary`、充值双倍、里程碑 | 合伙人池5%→10%、机器人/充值加成、里程碑奖励 |
| 支持奖励 | `supportBonus`、`TYPE_SUPPORT_GIVE` | 支持完成后的额外赠送 |
| 充值奖励 | `depositBonus`、`TYPE_DEPOSIT_MATCH_BONUS` | 支付成功匹配/刮奖/赠送 |

对应 H5 路由集中在 `h5/src/router/routes.ts` 的 `/activity/*`，页面在 `h5/src/pages/Activity/`；管理端集中在 `admin/src/pages/MarketingPacks/`、`Marketing/` 及多处统计页面。

## 9. 必须清理的混合核心路径

仅关闭活动页面仍会继续产生数据。以下保留流程内的钩子必须逐个旁路：

| 保留流程 | 当前混合依赖 | 目标处理 |
| --- | --- | --- |
| 注册/邀请 | VIP邀请、33系列、AI-X邀请机会 | 只创建核心用户和邀请链 |
| KYC回调 | 活动资格/任务触发 | 只更新KYC/人脸状态 |
| 充值完成 | `giveaway`、双倍、Deposit Match、Ramadan房间、活动任务 | 只入原始购买币并发核心事件 |
| 一次性VIP完成 | VIP邀请、支持券旧发放、自动续订任务 | 只创建一次性权益和合法基础返佣 |
| 短剧支持 | 双倍券、补贴、周年庆池、Ramadan、券转化 | 只执行基础支持、收益和返佣 |
| 机器人支持/结算 | AI-X机会、AI Carnival、周年庆、补贴 | 只执行保留机器人规则 |
| 支持等级升级 | AI-X、VIP邀请、宝藏/挑战/补贴 | 只更新等级；不发活动资格 |
| 合伙人 | 周年庆池10%、活动Envoy配置 | 固定使用基础5%和目标等级规则 |
| 提现成功 | 双倍券/机器人券活动转化 | 仅处理提现；保留券字段需业务确认 |
| 任务 | 版权、活动、连续订阅任务 | 仅返回目标任务白名单 |
| 钱包/收益 | 宽泛 `TypeGroup_INCOME` | 目标收益白名单，活动历史单列 |
| 通知/Banner | 活动跳转与活动消息 | 不再生成，历史点击不跳失效页面 |

## 10. 定时任务、消费者与缓存停用清单

明确停用或移除注册：

- `checkVipInviteActivityCron`、`checkVipInviteUserActivity`
- `checkUserUpgradeSubsidy`、`checkUserHelpSubsidies`
- `treasurePlanJackpotHourlyCron`、`treasurePlanDailyCycleCron`
- `checkTeamRoomCloseCron`、`checkTeamRoomRefundCron`
- AI-X/FlashSale场次或机会任务（包括模型升级钩子）
- `coupon:double` 消费者
- `coupon:support` 若仍依赖VIP计划自动发券，则默认停用
- 其他活动通知、排行榜、抽奖、发币和奖池任务

停用前记录最后运行时间和未完成批次；停用后删除/过期对应Redis活动配置、倒计时、奖池、机会、排行和页面持久化缓存，不能清理核心钱包/订单缓存。

保留但净化：支付回查、VIP到期、短剧/机器人结算、券过期、支持等级、资产rollup、提现对账。

## 11. 数据和API退役策略

### 11.1 API

- 第一阶段：隐藏入口、写接口返回统一功能关闭码，读接口仅供审计角色。
- 第二阶段：删除客户端调用和服务路由注册；保留旧 Proto/enum wire 值避免历史反序列化失败。
- 第三阶段：确认无流量后标记废弃；公共API改动另立兼容方案，本次文档不直接修改。

### 11.2 数据

- 活动表、聊天表、版权表、历史订阅表不在本次物理删除。
- 财务流水和订单永久按原始金额/类型保留；可增加 `scope_status=historical_excluded` 派生标签。
- 关闭外键写入后仍保持历史关联可查；不将排除类型批量改为“其他”。
- 对未完成活动资格/奖池的处理需业务和财务签字：终止、履约或退款，不由技术自行推断。

## 12. 代码证据

- H5：`h5/src/router/routes.ts`、`h5/src/pages/Activity/`、`h5/src/apis/activity.ts`、`h5/src/store/modules/activity.ts`、`h5/src/hooks/usePaymentEnd.ts`
- AI聊天：`h5/src/pages/Chats/`、`splay/services/user/chat.go`、`admin/src/pages/PromptWords/`、`ConversationRecords/`
- 版权包：`splay/services/task/`、`splay/models/task.go`、`admin/src/pages/CopyrightPacks/`
- 自动续订/VIP邀请：`splay/services/point/pay.go`、`plan.go`、`fans.go`、`man/init.go`
- 活动后端：`splay/services/point/activity*.go`、`team_room.go`、`treasure_plan.go`、`ai_x_subscription.go`，`splay/services/man/campaign*.go`、`cron.go`
- 混合枚举：`splay/apis/basepb/model.proto`

## 13. 排除验收清单

1. 全量扫描H5路由、菜单、Banner、通知跳转和支付成功跳转，不存在可达活动/聊天/版权/续订入口。
2. 调用所有排除写接口均不创建数据、不变资产；弹幕和保留机器人接口仍正常。
3. 等待至少一个完整日/周任务周期，排除任务和消费者无新执行记录。
4. 完成注册、KYC、充值、VIP购买、短剧支持、机器人支持、升级和提现，不产生任何活动表记录或活动流水。
5. 新充值只入原始金额；静态 `giveaway`、双倍和匹配均为0。
6. 自动续订不再扣款；已付费权益仍到期；一次性VIP正常购买和播放解锁。
7. 收益总览、曲线和账单不含排除类型，历史审计仍可按原始类型查询。
8. 排除数据未被物理删除，原订单、流水、权益和处理记录可追溯。
