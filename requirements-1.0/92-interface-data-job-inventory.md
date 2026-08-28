# 92 接口、数据、配置、缓存与任务索引

## 1. 索引口径

本索引用于后续逐项实施和回归，不表示本次已经修改API或数据库。

| 标记 | 含义 |
| --- | --- |
| 保留 | 目标1.0继续提供 |
| 部分 | 同一入口/接口混有保留与排除逻辑，需要拆分或净化 |
| 兼容只读 | 不再产生新数据，仅履约/查询历史 |
| 排除 | 目标1.0关闭入口、写接口、任务与钩子 |

事实来源：`h5/src/router/routes.ts`、`h5/src/apis/apiMapping.ts`、`admin/src/pages/`、`splay/apis/*pb/*.proto`、`splay/services/*/init.go` 和 `splay/models/`。

## 2. H5路由索引

### 2.1 保留或部分保留

| 模块 | 路由 | 状态/注意事项 |
| --- | --- | --- |
| 账号 | `/login`、`/register`、`/user`、`/user/modify` | 保留；移除注册活动钩子 |
| 安全 | `/security/*`、`/settings/*` | 保留；OTP和超级密码整改 |
| KYC/银行卡 | `/certifications/*`、`/bankCards/*` | 保留；Sumsub为当前主路径 |
| 短剧发现 | `/home`、`/recommend`、`/search` | 保留 |
| 短剧播放 | `/videos/detail/:id`、`/videos/play/:id/:episodeId` | 保留；统一解锁鉴权 |
| 收藏/历史 | `/favorites`、`/history` | 保留 |
| 任务 | `/mission` | 部分；移除假任务、活动/版权/续订任务 |
| 订单/充值 | `/orders/*`、`/payments/*` | 保留；支付成功不再跳活动 |
| 一次性VIP | `/cards` | 部分；只保留一次性购买，不显示订阅协议/自动续订 |
| 钱包 | `/wallet`、`/wallet/details/*`、`/wallet/frozen`、`/wallet/exchange` | 保留；净化流水类型 |
| 收益 | `/income`、`/income/share` | 保留；收益白名单 |
| 提现 | `/withdrawal`、`/withdrawal/details/:no`、`/withdrawal/record` | 保留；服务端金额整改 |
| 普通支持券 | `/coupons`、`/coupons/off` | 部分；历史可查，新领取待确认 |
| 支持/团队 | `/vote/*`、`/fans/*`、`/family/*` | 部分；移除FlashSale、订阅、VIP及活动卡片 |
| 机器人 | `/robots`、`/robots/refund`、`/robots/detail/:id`、`/robots/levels`、`/robots/aix` | 保留；AIX显示中性名称 |
| 机器人支持券 | `/aiCoupons` | 保留，路由可兼容，页面名改“机器人支持券” |
| 通知/协议/关于 | 对应现有路由 | 部分；清除活动和自动续订跳转 |

### 2.2 排除路由

| 领域 | 路由/目录 |
| --- | --- |
| AI聊天 | `/chats`、`/chats/accompany/:roleId`、`/chats/record` |
| FlashSale | `/robots/aixFlash`、Vote页内 `AixFlash`/订阅机器人组件 |
| 全部活动 | `/activity/*`：pilgrimage、subsidy、subsidyII、lottery、assistance、ramadan、doubleWages、aiCarnival、poolHadiah、33Invite、33Challenge、partnerPlan、33Cooperation、partnerPlanII、anniversary、supportBonus、depositBonus等 |
| 版权包/二创 | `/materials/*`、版权下载/教程/协议中仅为版权包服务的路由；需与普通短剧素材逐条区分 |

## 3. 管理端页面索引

### 3.1 保留/部分保留

| 目录 | 状态 | 用途/整改 |
| --- | --- | --- |
| `admin/src/pages/Playlet/` | 保留 | 短剧、分组、剧集、字幕、普通推荐素材 |
| `Barrage/`、`SensitiveWords/` | 保留 | 增加举报审核闭环和独立权限 |
| `RobotHosting/`、`RobotHosting/dayData/` | 部分 | 保留普通机器人和日收益率；`flashSale/`排除 |
| `ShortDramaRewards*` | 保留 | 短剧支持、收益和持仓 |
| `ActivityValueDetails/`、`SupportValueDetails/` | 保留 | 活跃值与支持值流水 |
| `Marketing/supportCards/` | 部分 | 普通支持券规则/历史；新发放待确认 |
| 机器人券规则/券码/统计页面 | 保留 | 对外改中性名称 |
| `Mission/tasks/`、`Mission/tasksvideo/` | 部分 | 保留目标任务；删除废弃字段/排除任务 |
| `Users/`、`Certifications*` | 部分 | 保留用户/KYC；删除聊天和版权发放入口 |
| `Financial/orders/`、`depositTransfer/`、`withdrawal/`、`integral/` | 部分 | 保留财务；排除活动类型的新增操作，历史可查 |
| `CoinsPacks/` | 部分 | 保留充值套餐，`giveaway=0` |
| `VipCards/` | 部分 | 只保留一次性VIP卡和历史统计 |
| `ad/articles/` | 保留 | 当前Banner管理入口；服务端时间和缓存整改 |
| `Rankings/`、`Datasheet/`、`UserStatistics/` | 部分 | 仅保留真实核心统计，标出放大/虚拟值 |
| `Logs/`、`Admins/`、`Roles/`、`RoleManagement/` | 保留 | 审计和权限 |

### 3.2 排除/兼容只读

- `ConversationRecords/`、`PromptWords/`：AI聊天。
- `CopyrightPacks/*`：版权包、素材、二创和用户包。
- `Marketing/speedCards/` 中双倍券部分、`Marketing/receiveDetails/` 中双倍记录。
- `MarketingPacks/*`：全部活动包；历史财务记录可迁移至只读审计页。
- `RobotHosting/flashSale/`：FlashSale。
- 用户页“虚拟聊天记录”“发放测试版权包”等操作。

## 4. 用户端API索引

### 4.1 用户、安全、KYC和银行卡

| 状态 | 接口 |
| --- | --- |
| 保留 | `/v1/users/login/step1`、登录第二步Proto方法、`/v1/users/register`、`/v1/users/logout` |
| 保留 | `/v1/users/get`、`/v1/users/update`、`/v1/users/change`、`/v1/users/delete`、忘记密码 |
| 保留 | 验证码/消息发送检查、安全token、`/v1/user/otp/generate`、`/v1/user/otp/bind` |
| 保留 | `/v1/sumsub/initialize`、Sumsub回调、`/v1/withdraw/face_check/status` |
| 保留 | `/v1/user/bank/create`、`list`、`modify`、`/v1/user/withdraw/bank/list` |
| 部分 | 第三方登录、官方群、粉丝团队/邀请接口；删除活动计数 |
| 排除 | 合作协议若仅服务版权包；旧Zoloz/手工KYC入口在确认无流量后退役 |

### 4.2 短剧与弹幕

| 状态 | 接口 |
| --- | --- |
| 保留 | `/v1/playlet/group/list`、`group/in/list`、`group/batch/list`、`group/home/list` |
| 保留 | `/v1/playlet/search/list`、`more/list`、`details/page/list`、普通推荐 `material/list` |
| 保留 | `/v1/playlet/episode/list`、`/v1/playlet/video/auth/get` |
| 保留 | `/v1/playlet/ledger/list`、`ledger/update`、`collect/add`、`collect/list`、`collect/update` |
| 保留 | `/v1/chat/bullet/comment/create`、`list`、Proto中的 `report` |
| 排除 | `/v1/chat/role/*`、`session/*`、`message/*`、`ChatCompletion`、聊天角色举报 |

### 4.3 短剧支持、团队与合伙人

| 状态 | 接口 |
| --- | --- |
| 保留 | `/v1/playlet/tips/create`、`/v1/tips/playlet/list`、`/v1/playlet/tips/list` |
| 保留 | `/v1/tips/sold/out/playlet/list`、`history/ledger/list`、`all/history/ledger/list` |
| 保留 | `/v1/tips/overview/get`、`report/data/list`、`param/get` |
| 保留 | `/v1/active/value/ledger/list`、`/v1/fan/club/data/get/v2` |
| 保留 | `/v1/rebate/ratio/get`、`/v1/tips/upgrade_condition/get` |
| 保留 | `/v1/fans/summary`、`/v1/support/summary`、`/v1/fans/fans/list`、`/v1/fans/support/list` |
| 保留 | `/v1/partner/rebate/get`、`receive`、`/v1/family/data/overview/get` |
| 排除 | VIP团队summary/list、升级补贴、支持赠送、活动榜单和活动合作页面接口 |

### 4.4 机器人

| 状态 | 接口 |
| --- | --- |
| 保留 | `/v1/tips/robot/list`、`/v1/tips/robot/ledger/list` |
| 保留 | `/v1/robot/tips/create`、`detail/get`、`freeze/ledger/list` |
| 保留 | `/v1/robot/tips/income/get`、`income/receive` |
| 保留 | `/v1/tips_leve_robot/list`、`buy`（兼容历史拼写） |
| 保留 | `/v1/robot/ai_invest/hosting/summary/get`、`/v1/tips/robot/aggregate/get` |
| 排除 | `/v1/tips/flash_sale/state`、`grab`、AI-X订阅状态/邀请记录、新购 `TYPE_SUBSCRIPTION` |

### 4.5 钱包、收益和Banner

| 状态 | 接口 |
| --- | --- |
| 保留 | `/v1/asset/overview/get`、`/v1/consume/point/exchange` |
| 保留 | `/v1/balance/history/get`、`/v1/transaction/summary/all/get` |
| 保留 | `/v1/point/ledger/list/v2`、`/v1/point/frozen/list` |
| 保留 | `/v1/income/overview/get`、`details/get`、`distributed/get`、`month/bill/get`、`total/bill/get` |
| 保留 | `/v1/advertisement/get` |
| 部分 | 老 `/v1/point/ledger/list`、转币接口、收入分享；明确兼容与权限 |

### 4.6 支持券与任务

| 状态 | 接口 |
| --- | --- |
| 兼容只读/待确认 | `/v1/point/support/coupon/get`、`receive` 当前关闭；`list`保留历史 |
| 保留 | `/v1/point/robot_coupon/get`、`check`、`use` |
| 排除 | `/v1/point/double/coupon/*`及双倍券管理 |
| 保留/收敛 | `/v1/user/task/list`、`/v1/task/progress/update`、`income/get`、`video_url/get`、`all/list` |
| 候选主接口 | Proto `ListUserTaskV2`、`UpdateUserTaskV2`；统一语义后旧接口代理 |
| 待确认 | `/v1/task/plan/list`、`active/list`（付费TaskPlan） |
| 排除 | `taskpb`版权包、素材、二创、合作协议接口 |

### 4.7 订单、支付、VIP与提现

| 状态 | 接口 |
| --- | --- |
| 保留 | `/v1/order/create|get|list|pay|cancel|refund/apply|refund/cancel` |
| 保留 | Stripe/InTheBag/Monetapay/Haipay/Jaya/Future/Unispay等已启用的create/check/callback |
| 保留 | `/v1/pay/link/create`、`/v1/pay/payment/result/verify`、`/v1/order/deposit/channel/list` |
| 保留 | `/v1/point/pack/list`、`/v1/point/rate/get`、`/v1/plan/list`、`/v1/user/plan/list` |
| 保留 | `/v1/user/withdrawal/create`、`list`及管理端审核/出款/回调 |
| 排除 | Apple/Google或其他自动续订创建/续订回调的新写、`TransferUserPlan` |
| 兼容只读 | `PAYMENT_AUTO`订单和已付费存量UserPlan |

### 4.8 活动API族（全部排除）

- `/v1/vip_invite/*`、`/v1/lottery/*`
- `/v1/mega_subsidy/*`、`/v1/help_subsidies/*`、升级补贴/双倍工资
- `/v1/point/invite_register*`、`/v1/ai_carnival*`
- `/v1/treasure_plan/*`、`/v1/challenge33/*`、`/v1/33_help/*`
- `/v1/anniversary/*`、`/v1/support_give/*`、`/v1/deposit_match/*`、`/v1/recharge/double/gift/get`
- `/v1/team/room/*`、`/v1/point/ramadan*`

## 5. 管理端RPC/API族

管理端以 `splay/apis/manpb/` 和 `admin/src/apis/apiMapping.ts` 为事实源。实施时按下列域逐个登记具体RPC：

| 域 | 目标状态 | 典型能力 |
| --- | --- | --- |
| 用户/KYC/银行卡 | 保留 | 列表、状态、详情、审计；安全敏感操作加强权限 |
| 短剧/分组/剧集/字幕 | 保留 | CRUD、上下架、排序、播放地址检查 |
| 弹幕/敏感词 | 保留 | 列表、隐藏；补举报审核 |
| 机器人/日收益率/持仓 | 保留 | 排除FlashSale与订阅类型配置 |
| 支持参数/流水/等级 | 保留 | 去除活动加成配置 |
| 普通支持券 | 部分 | 规则和历史；发放待确认 |
| 机器人支持券 | 保留 | 规则、批次发放、券码、统计 |
| 任务/视频 | 部分 | 目标任务白名单；去废弃字段 |
| 套餐/订单/支付/提现 | 保留 | `giveaway=0`，自动订阅只读 |
| Banner | 保留 | CRUD、时间、排序、缓存失效 |
| AI聊天/版权包/MarketingPacks | 排除 | 历史只读按需迁移 |

## 6. 核心模型索引

| 模块 | 模型/表 | 来源 |
| --- | --- | --- |
| 用户 | `User`、`LoginLog`、`InviteLog`、第三方绑定 | `models/user.go`等 |
| KYC/银行 | `Verification`、`WithdrawFaceCheck`、`BankCard` | `models/user.go`、`withdraw_face_check.go` |
| 短剧 | `Playlet`、`PlayletGroup`、`PlayletEpisode`、观看/收藏/字幕/广告解锁记录 | `models/playlet.go` |
| 弹幕 | 弹幕正文、用户屏蔽、举报、敏感词 | `models/chat.go`及相关文件 |
| 支持 | `TipsLedger`、`TipsHistoryLedger`、`PlayletIncomeLedger`、`TipsIncomeLedger` | `models/tips.go` |
| 机器人 | `Robot`、`RobotDailyIncomeRate`、机器人收益/冻结流水 | `models/robot.go`、`tips.go` |
| 等级/活跃 | User等级字段、`ActiveValueLedger`、`TipsParameter` | `models/user.go`、`parameter.go` |
| 合伙人 | 合伙人月度资格、返佣记录、领取记录 | `models/partners.go` |
| 钱包 | User资产字段、`PointLedger`、`PointFrozenLedger` | `models/user.go`、`point.go` |
| 订单/VIP | `Order`、`PointPack`、`Plan`、`UserPlan`、退款/支付记录 | `models/order.go`、`point.go`、`plan.go` |
| 券 | `SupportCoupon`、`UserSupportCoupon`、`RobotCouponRule/Code/SendStat` | `models/coupon.go` |
| 任务 | `Task`、`UserTask`、任务视频；`TaskPlan/UserTaskPlan`待确认 | `models/task.go` |
| 提现 | `Withdrawal`、冻结流水、出款渠道配置/错误 | `models/user.go`、`withdraw*.go` |
| Banner | `Advertisement` | `models/advertisement.go` |
| 统计 | `UserStatsDetail`、Point/Order/Tips rollup | `models/rollup_stats.go`等 |

### 6.1 排除数据模型

聊天角色/会话/消息/提示词，版权包/用户版权包/素材/二创，自动订阅专用记录，VIP邀请、双倍券、补贴、抽奖、Ramadan房间、AI-X机会、treasure plan、33系列、周年庆、充值匹配等活动模型。历史不删除，新写关闭。

## 7. 关键枚举与兼容风险

来源：`splay/apis/basepb/model.proto`。

| 枚举 | 风险/要求 |
| --- | --- |
| `User_TipsLevel` | 数值顺序不等于L0–L7；`VIP`是支持L1，不是付费VIP；必须用rank映射 |
| `Robot_Type` | `AI_INVEST`保留并中性化；`SUBSCRIPTION`/FlashSale排除；历史值可读 |
| `PointLedger_Type` | 保留与大量活动类型混在同一枚举；统计使用白名单，不删历史值 |
| `Order_ProductType/Status` | `VIP`只允许一次性新购；`PAYMENT_AUTO`/transfer仅历史兼容 |
| `Task_Type/RepeatRule/Status` | NEW/SUPER/DAILY/TIPS保留；NONE/DAILY/WEEKLY需统一周期 |
| `TipsLedger_Status/Method` | WAIT/EFFECTIVE/ENDED，PLAYLET/ROBOT；状态迁移幂等 |
| `Withdrawal_Status` | 多套单笔/批量状态；目标统一合法迁移表 |
| `SupportCoupon/RobotCoupon_Status` | 两类券状态相似但规则独立 |
| `Advertisement_Type` | HOME/WELFARE/REWARD/VIP/USER_CENTER/LAUNCH；VIP仅一次性页 |
| `Granularity` | day/week/month/year；现有接口支持程度不一 |

## 8. 配置索引

### 8.1 保留并治理

| 配置 | 用途 | 要求 |
| --- | --- | --- |
| `WealthParameter` | 汇率、提现上下限、首/次费率、最低手续费、返佣解冻 | 版本化；修复未生效上下限 |
| `TipsParameter` | 支持持仓、等级返佣、支持值兑换 | 单一事实源，不由H5复制 |
| Robot及每日收益率 | 机器人周期、售价、本金、额度、收益 | 持仓保存规则快照 |
| Playlet支持字段 | 收益率、周期、额度、价格 | 版本化、上下架规则 |
| `RefundParameter` | 剧集/VIP等基础返佣 | 区分一次性与自动订阅 |
| Plan/PointPack | 一次性VIP、充值套餐 | `giveaway=0`，价格快照 |
| Banner | 位置、时间、跳转 | 服务端过滤、priority |
| `withdrawal.face_check_required` | 提现人脸开关 | 环境一致且可审计 |
| 支付商配置 | 密钥、域名、回调、渠道/银行映射 | 不硬编码，不输出日志 |
| `domain.base`、业务时区 | 链接和时间 | 统一环境配置/IANA时区 |
| 任务中心开关/任务规则 | 可见、进度、奖励 | 版本化并清缓存 |

### 8.2 删除或默认关闭

- `super.password`。
- `support_coupon.user_ids`，除非确认运营定向发放。
- VIP邀请、AI-X活动、FlashSale、双倍/匹配、Ramadan、treasure plan、anniversary、补贴/抽奖等所有活动配置。
- 活动硬编码日期、比例和机器人ID列表。

## 9. 缓存与Redis索引

| 缓存/用途 | 当前特征 | 目标要求 |
| --- | --- | --- |
| 启用任务列表 | 全局约60秒，管理更新失效 | 键含规则版本；只保留任务白名单 |
| Banner | 按类型约60秒 | 管理写后主动失效；服务端按时间过滤 |
| 收益总览 | 用户级缓存至午夜 | 键含口径版本/时区；活动下线强制失效 |
| 收益超过百分比/排行榜 | Redis定时刷新 | 仅展示，不作为资产事实 |
| 短剧播放/热度 | Redis播放计数和排序 | 真实计数与展示热度分离 |
| 配置缓存 | 敏感词、活动、全局参数等 | 配置发布版本；活动键过期，核心键保留 |
| 任务/群链接游标 | Redis游标 | 并发原子；排除活动游标清理 |
| FlashSale/奖池/机会/倒计时 | 活动缓存 | 全部停止写并安全过期 |

## 10. 定时任务索引

任务均使用 `models.Local` 注册，目标需统一业务时区。

### 10.1 `playlet` 服务

| 调度 | 任务 | 状态 |
| --- | --- | --- |
| 每日 | `checkPlayletTipsStatus` | 保留，支持到期 |
| 每日 | `clearTipsLedgerTodayIncome` | 保留但核对结算事实 |
| 每日12:00 | `checkSettlementRobotIncomeConcurrent` | 保留，幂等/批次化 |
| 每日12:00 | `checkRobotBarrageTask` | 排除，不能伪完成弹幕任务 |
| 每日 | `checkRobotTipsStatus` | 保留，机器人返本 |
| 每日 | `unfreezeRobotTipsIncome` | 保留 |
| 每5分钟 | `checkRobotStatus` | 保留，排除订阅类型新上架 |
| 每分钟第17秒 | `checkSupportLevelIncremental` | 保留，净化活动钩子 |
| 每小时53分 | `checkLv2Upgrade` | 保留，规则版本化 |

### 10.2 `point` 服务

| 调度 | 任务 | 状态 |
| --- | --- | --- |
| 每分钟 | `checkUserPlanExpire` | 保留一次性VIP和存量权益到期 |
| 每5分钟 | `reconcileDepositOrdersCron` | 保留 |
| 每小时 | `checkPointFrozenLedgers` | 保留 |
| 每分钟 | `checkCouponExpiredCron`、`checkRobotCouponExpire` | 保留目标券；双倍券分支排除 |
| 每小时错峰 | 收入、通用币、粉丝、活跃值排行 | 部分；只用真实/目标数据 |
| 环境定时 | `checkUserUpgradeSubsidy` | 排除 |
| 可选每日 | `checkUserHelpSubsidies` | 排除 |
| 每日22:00 | TeamRoom关闭/退款 | 排除活动；未完成历史需一次性收尾 |
| 每10分钟 | `treasurePlanJackpotHourlyCron` | 排除 |

### 10.3 `task`、`user`、`man` 服务

| 服务/调度 | 任务 | 状态 |
| --- | --- | --- |
| task 每日 | `checkUserTaskStatus` | 保留，使用统一时区/周期 |
| user 每日 | `resetDailyInviteRegisterPrayCount` | 排除活动 |
| man 每5分钟 | `checkPlayUrlCron` | 保留 |
| man 每小时+00:10 | `RefreshTodayUserStats` | 保留，收益白名单 |
| man 每日 | `backupUserStatsDaily` | 保留 |
| man 每5分钟 | `checkVipInviteActivityCron` | 排除 |
| man 每周 | `clearUserFakeRankData` | 待确认；真实统计不应依赖虚拟用户 |
| man 每日01:00 | `autoGrantRobotCoupon` | 保留，规则和批次治理 |
| man 每日21:00 | `treasurePlanDailyCycleCron` | 排除 |
| man 每日00:30 | `withdrawFaceCheckDailyCron` | 保留 |
| man 每分钟 | point/order/tips rollup | 保留，过滤目标口径 |
| man 每日04:10 | `rollupReconcileCron` | 保留 |
| man 每分钟 | `orderMonitorCron` | 保留 |

## 11. 消息消费者与后台作业

| Topic/作业 | 状态 | 说明 |
| --- | --- | --- |
| `subtitle.sync`、`subtitle.download` | 保留 | 短剧字幕 |
| `video.erase` | 保留/谨慎 | 仅删除明确目标视频，需审计 |
| `withdrawal:auto` | 保留 | 自动出款，ID幂等与重试 |
| `notification:send_all` | 部分 | 不发送活动/聊天/版权消息 |
| `coupon:support` | 默认关闭 | 旧逻辑依赖UserPlan，来源待确认 |
| `coupon:double` | 排除 | 双倍券 |
| 支付回调HTTP | 保留已启用渠道 | 验签、标准事件、幂等 |
| Sumsub回调HTTP | 保留 | 验签、事件幂等、去活动钩子 |

## 12. 建议废弃、拆分或改名的接口影响

本次不实际修改API，后续兼容方案建议：

| 现有名称 | 建议 | 兼容策略 |
| --- | --- | --- |
| `/v1/chat/bullet/*` | 新增 `/v1/playlet/barrage/*` | 旧路径代理到同一实现，先迁客户端 |
| `tips`/“打赏” | 对外统一“支持” | 不改wire字段，文案/领域DTO先中性化 |
| `AI_INVEST` | “智能收益机器人” | enum值保留，增加显示分类 |
| `robot_coupon`/“AI券” | “机器人支持券” | API路径可保留，响应显示名中性化 |
| `TipsLevel_VIP` | “支持等级L1” | enum值保留，禁止直接显示name |
| `/v1/tips_leve_robot/*` | 拼写正确的新别名 | 旧路径兼容 |
| 老任务接口与V2 | 单一Task API | 老接口代理，不再复制逻辑 |
| 收益五个接口 | 共享口径参数/类型表 | 保持路径，统一内部查询器和响应元数据 |
| 活动API | 标记deprecated/关闭 | 写接口先返回关闭码，确认无流量后移除路由 |

## 13. 索引验收

1. 每个保留H5路由都能映射到本目录某个模块文档和后端接口。
2. 每个排除入口都有对应API、模型、任务/消费者和统计清理项。
3. 资金接口均能映射到业务单、账户字段、流水和失败回滚。
4. 定时任务在部署清单中逐项标记“保留/排除/历史收尾”，没有未登记的活动任务。
5. 所有兼容enum能读取历史值，用户端不显示误导性的 `VIP/AI` 历史名称。
6. API退役前有流量观测和兼容期，不在本次文档提交中直接改Proto或数据库。
