# 09 充值、一次性 VIP 与提现

> 本章定义外部收款、一次性 VIP 履约和通用资产提现。支付商是外部事实来源，订单、权益、资产、冻结和审计是本库事实；二者通过可重放状态机收敛，不用长事务包住网络调用，也不建立通用 Outbox。

## 1. 目标

1. 固定套餐和任意金额充值都只到账订单明确购买的 `COMMON`，不赠送、不匹配、不翻倍。
2. 一次性 VIP 只使用后台已发布商品版本的显式价格；没有首购 1 单位价、优惠券、自动续订、订阅回调、转移或赠币。
3. 所有支付渠道先归一为相同的 payment/payout 语义，再进入唯一履约器；渠道适配器不能直接改订单、资产、VIP 或提现状态。
4. 同库的订单、权益、资产流水、冻结明细、回调审计和操作审计在 `database.default` 的一次事务中提交。
5. 创建支付、查单、验票、退款、出款和出款查询等外部调用永远不在数据库事务内。
6. 不把 Redis 队列当作可靠事实，不建通用 Outbox。待处理工作由业务行的 `status + next_run_at` 在 primary 扫描恢复，Redis 只负责加速唤醒。
7. 提现只接受 `COMMON`，申请即创建通用冻结明细；拒绝/确认失败释放，确认到账结算，任何重放或乱序都不能重复动钱。
8. 渠道订单、渠道流水、本地业务单、资产/权益/冻结能够按商户单号和渠道单号双向对账。
9. 2.0 不做每笔提现专用人脸检查；普通与注销清退提现复用基础 KYC。若用户存在 ACTIVE Authenticator/TOTP binding，则创建请求必须提交并在同一事务消费 `otp_code` 对应的 time step。

## 2. 对象与不变量

### 2.1 领域对象

| 对象 | 职责 |
| --- | --- |
| `deposit_products` | 固定充值套餐的已发布版本；任意金额充值不伪造套餐 |
| `deposit_orders` | 一笔充值业务意图和金额/汇率快照 |
| `vip_products` | 一次性 VIP 的显式价格、期限和权益版本 |
| `vip_orders` | 一次性 VIP 购买意图和商品快照 |
| `user_vip_stats` | 每用户一行的 VIP 容量/并发根；O(1) 裁决 reservation 与未终结权益总数，硬上限 64 |
| `vip_entitlements` | 已支付 VIP 形成的排队、生效、到期或撤销权益 |
| `payment_channels` | 充值/VIP 共用的渠道、支付方式、限额与配置版本 |
| `payment_attempts` | 充值/VIP 对外收款的渠道尝试、渠道状态和恢复游标 |
| `withdrawals` | 提现申请、审核、金额快照、冻结引用和业务状态 |
| `account_closures` | 注销清退上下文；closure withdrawal 只引用当前用户未完成记录 |
| `withdrawal_reviews` | 每次成功审核决定的追加式事实；`withdrawals` 只冗余当前决定摘要 |
| `payout_attempts` | 提现对外出款的渠道尝试、未知结果和回查游标 |
| `payout_channel_switches` | 旧 attempt 取得权威 NOT_ACCEPTED 证据后的换渠道审批/命令事实；一次旧 attempt 最多切换一次 |
| `payout_failure_resolutions` | 权威 NOT_ACCEPTED 后不再换渠道的唯一失败裁决；与换渠道在同一 attempt 上互斥 |
| `payment_provider_events` | 已验签收款/退款事件的防重和审计 |
| `payout_provider_events` | 已验签出款事件的防重和审计 |
| `finance_recovery_cases` | 已关闭/余额不足第三方不能钱包冲正时的唯一外部追偿事实；不得阻断付款人终态 |
| `benefit_forfeiture_audits` | 新返佣候选因第三方非 ACTIVE 而从未授予的唯一审计；不产生钱包流水 |

这些表的用户关系一律是 `user_id -> users.id`。API、日志、回调映射和管理查询均不存在 F 码。

### 2.2 金额不变量

固定套餐充值：

```text
credited_common = deposit_orders.common_amount
赠送金额 = 0（Schema 和 API 中没有 bonus/giveaway 字段）
```

任意金额充值：

```text
fiat_amount = round_by_channel(common_amount × rate_common_to_usd × rate_usd_to_fiat)
credited_common = common_amount_snapshot
```

一次性 VIP：

```text
quantity = 1
payable_fiat = published_product.explicit_price_snapshot
entitlement_duration = published_product.duration_snapshot
```

提现：

```text
requested_common > 0
fee_common = max(requested_common × fee_rate_snapshot, min_fee_snapshot)
net_common = requested_common - fee_common > 0
payout_fiat = round_by_channel(net_common × rate_common_to_usd × rate_usd_to_fiat)
freeze_amount = requested_common
```

手续费包含在用户申请的 `requested_common` 中；成功时整笔冻结离开账户，失败/拒绝时整笔原路退回。`fee_common` 和 `net_common` 是提现单的审计拆分，不允许客户端自行决定。

### 2.3 跨表不变量

1. `deposit_orders.status IN (PAID, REFUND_PENDING, REFUNDED, REFUND_CANCELED)` 当且仅当存在一条 `deposit:credit:<order_no>` 的原始 `COMMON/INCOME` 流水，金额等于 `common_amount`。`PAID` 尚无退款冻结；`REFUND_PENDING` 恰有一条等额 ACTIVE refund freeze；`REFUNDED` 保留原 credit，且该 freeze 已 SETTLED 并恰有一条 `deposit:refund:settle:<order_no>`；`REFUND_CANCELED` 也保留原 credit，但该 freeze 已 RELEASED 并恰有一条 `deposit:refund:release:<order_no>`，不得有 settle。
2. 每张 `vip_orders` 创建时恰占一个 `slot_reservation_state=RESERVED`；支付履约同事务把它单向转为 CONSUMED 并创建恰一条永不删除的 `vip_entitlements`，只有取得不可逆未收款证据时才可从 RESERVED 转 RELEASED。`user_vip_stats.reserved_slots` 等于 RESERVED 订单数，`entitlement_slots` 等于 ACTIVE/QUEUED 权益数，二者始终非负且和不超过 64。`vip_orders.status IN (PAID, REFUND_PENDING, REFUNDED, REFUND_NOT_ACCEPTED)` 时恰有一条来源权益；原支付订单、权益、最多两层冻结返佣和一次性 VIP 任务进度同事务完成。REFUNDED 保存 `refund_entitlement_disposition=REVOKED|ALREADY_EXPIRED`：未终结权益转 REVOKED，已到期权益保持 EXPIRED；两者都创建唯一返佣 REVERSAL，不删除历史。`REFUND_NOT_ACCEPTED` 是无外部退款效果的终态，权益、返佣和计数保持退款前状态，且必须引用唯一 REFUND attempt 的权威未受理证据。
3. 一个充值/VIP 订单最多一个成功 COLLECT `payment_attempt`。两类订单每单都最多创建一个全生命周期逻辑 REFUND attempt，网络重试、worker 重领和人工重试始终复用同一行及稳定商户号，不得换 attempt/merchant key 绕过本地退款状态机；`(provider, psp_ref)` 全局只映射一个本地尝试。不支持按稳定商户号幂等退款/查单的渠道不得启用 VIP/充值退款能力。
4. 一个提现最多一个活动出款尝试；渠道超时不等于失败，不能据此释放冻结或创建第二笔不相关出款。历史尝试硬上限为 8；全部曾成功 attempt 的当前财务状态组成一个有界集合，未反转集合非空时以最小 `attempt_no` 作为唯一 canonical，其余逐笔为 duplicate。
5. 提现冻结关系固定如下：

| 提现状态 | 对应 `asset_freezes.status` |
| --- | --- |
| `AWAIT_REVIEW`、`APPROVED`、`PAYING`、`FAIL_CONFIRMING` | `ACTIVE` |
| `CANCELED`、`REJECTED`、`FAILED` | `RELEASED` |
| `ARRIVED` | `SETTLED` |

6. 提现只能引用 `asset_type=COMMON` 的冻结；请求传 `EXPERIENCE` 即参数错误，不能尝试兑换后提现。
7. 商品、渠道、汇率、费率、收款账户和业务规则均保存下单时快照；运营后续修改不重算旧单。
8. 所有金额用十进制定点数，API 用字符串；任何解析错误、超精度、非正数或溢出在创建业务单前失败。

## 3. 状态机

### 3.1 充值订单

```text
PENDING ───────────────> PAID
   │                      │
   ├────> EXPIRED         └────> REFUND_PENDING ────> REFUNDED
   └────> FAILED                     │
                                    └──显式最终取消──> REFUND_CANCELED

EXPIRED/FAILED --渠道证明确已收款--> PAID
```

- `EXPIRED` 只表示本地不再让用户继续发起支付，不是“渠道绝不可能已收款”的财务证明。
- 已验签、商户/金额/币种一致的真实收款优先于迟到回调时间；即使回调晚于 `expires_at`，仍用原订单快照履约并标记 `is_late_paid`，不能吞掉用户已付资金。
- `PAID` 只能在增加 `COMMON` 的同一事务写入；重复确认返回原结果。进入 REFUNDED 后原 credit 仍存在，退款效果只能由 refund freeze 的唯一终结流水表达。
- 外部超时、临时错误和可重试拒绝都保持 `REFUND_PENDING` 与 ACTIVE freeze，使用同一退款商户号回查/重试；worker 不得因重试耗尽自动解冻。只有在有证据证明渠道尚未受理，且没有成功/未知中的外部请求时，授权管理员才可显式取消；取消事务 `ReleaseFreeze` 并进入终态 `REFUND_CANCELED`。该订单之后永久禁止再次发起标准退款，避免已终结 freeze 被重新开启。
- `FAILED` 只用于渠道明确拒绝且没有收款的情况，不能由本地超时推断。

### 3.2 一次性 VIP 订单与权益

VIP 订单使用与充值相同的收款状态：

```text
PENDING -> PAID -> REFUND_PENDING -> REFUNDED
   │                    └---------> REFUND_NOT_ACCEPTED
   ├----> EXPIRED
   └----> FAILED
```

这里“相同”仅指收款/退款主干；首发 `REFUND_CANCELED` 是充值资产冻结的专用终态，不用于 VIP。VIP 没有 refund freeze；只有稳定商户号查单已证明请求不可逆未受理、没有成功或未知外部请求时，退款才可从 REFUND_PENDING 终结为 `REFUND_NOT_ACCEPTED`。该终态不撤销权益、不冲正返佣、不改变资产，也不可再次发起标准退款。

`PAID` 写入与权益、冻结返佣及中性任务进度创建同事务。VIP 权益状态为：

```text
支付履约 ──无当前/排队权益──────────────> ACTIVE ──ends_at<=now──> EXPIRED
       └─已有未终结权益───────────────> QUEUED ──到达区间──> ACTIVE
                                              └─ends_at<=now──> EXPIRED
ACTIVE/QUEUED ──退款/合法撤销──────────────────────────> REVOKED
```

- 一个用户任一时刻最多一条 ACTIVE 权益，生效区间不得重叠。
- 已有 ACTIVE/QUEUED 时，新权益按队尾顺延；排队顺序由订单支付时间和 ID 稳定决定。
- 每用户 `RESERVED 订单 + ACTIVE/QUEUED 权益 <= 64`。容量在创建外部收款意图前预占，不能等支付成功才抢位；所有队列排序、同步 Ensure、到期和退款压缩只扫描状态为 ACTIVE/QUEUED 的集合，因此单事务最多处理 64 条，不接触无界历史行。
- 产品价就是订单价。不存在“是否首购”“是否 Apple/Google”“是否有优惠券”的隐藏分支。
- `vip_products` 没有自动续订或赠币字段；App Store/Google 只按一次性商品凭证处理，不注册订阅续费和转移状态机。

### 3.3 收款尝试

```text
CREATED -> PENDING -> SUCCEEDED
   │          │
   │          ├────> FAILED
   └──────────┴────> UNKNOWN -> PENDING / SUCCEEDED / FAILED
                         FAILED --权威迟付证明/纠正审计--> SUCCEEDED
```

`UNKNOWN` 表示请求可能已到达渠道但本地没有确定响应。恢复动作必须先按稳定商户单号查询；只有渠道明确不存在且其创建接口支持同商户单号幂等时才重发。COLLECT attempt 的 FAILED 只表示当时收到明确拒绝；若随后有验签回调/主动查单证明同商户号真实成功，可在保存 `corrected_from_failed_at` 和 provider event 证据后条件转 SUCCEEDED，并与订单迟付履约同事务。没有权威证据不得人工翻转；REFUND/PAYOUT 的资金终态仍遵循各自更严格状态机。

### 3.4 提现与出款

```text
创建并冻结
    │
    v
AWAIT_REVIEW ──────> REJECTED
    │                  └─ ReleaseFreeze
    ├───────────────> CANCELED（仅用户、未审核）
    │                  └─ ReleaseFreeze
    v
APPROVED
    │ primary sweep 认领
    v
PAYING ─────────────────────────────> ARRIVED
    │        ├─ attempt NOT_ACCEPTED ─> PAYING（新 attempt）
    │                                  └─ SettleFreeze
    └────> FAIL_CONFIRMING ─────────> FAILED
               │                       └─ ReleaseFreeze
               ├────────────────────> ARRIVED
               └────渠道仍处理中────> PAYING
```

- 用户只能在 `AWAIT_REVIEW` 取消；批准后禁止取消。取消与 `ReleaseFreeze` 在同一事务完成。
- 管理员审核通过只写 `APPROVED + next_run_at`，不在审核事务中调用出款商。
- 失败回调默认先进入 `FAIL_CONFIRMING` 并保持冻结，除非渠道契约明确其失败状态不可逆。primary 回查确认失败后才进入 `FAILED` 并释放。
- `CANCELED`、`REJECTED`、`FAILED`、`ARRIVED` 是冻结资金的四个业务终态。相同终态重放返回成功，不再写流水。
- 首次真实成功把冻结/限额结算并进入 `ARRIVED`；此后钱包终态不回写。已 `ARRIVED` 后的成功、成功反转或迟到成功，统一重算最多 8 个 attempt 的 canonical/duplicate disposition：仍有未反转成功时最小 attempt 覆盖提现，额外成功才是 RECEIVABLE；一个成功被反转但另一个仍成功时只重选 canonical，不能误建“未到账” PAYABLE；全部成功均反转时才建立 PAYABLE。既有案件代表的差异消失时必须取消或形成反向案件，不能让旧 RECEIVABLE/PAYABLE 残留。`FAIL_CONFIRMING` 后首个真实成功仍可正常转 `ARRIVED`。

出款尝试操作状态固定为 `CREATED/SUBMITTING/PENDING/UNKNOWN/FAILED/NOT_ACCEPTED/STOPPED_BEFORE_SUBMIT/STOP_CONFIRMING/STOPPED/SUCCEEDED`。普通前向边为 `CREATED -> SUBMITTING -> PENDING -> SUCCEEDED|FAILED|NOT_ACCEPTED`，网络超时进入 UNKNOWN 并回查；渠道切换后尚未提交的新 attempt 可转 STOPPED_BEFORE_SUBMIT，已经可能到达渠道的新 attempt 转 `STOP_CONFIRMING -> STOPPED|SUCCEEDED` 且只允许查单、禁止再提交。另有三条窄纠正边 `FAILED|NOT_ACCEPTED|STOPPED -> SUCCEEDED`，只允许已验签成功事件或封存 reconciliation 在完全匹配 provider/merchant/金额/币种时触发；旧失败/未受理/停止证据保留并写异常审计。`STOPPED_BEFORE_SUBMIT -> SUCCEEDED` 永久禁止。`NOT_ACCEPTED` 只接受渠道按稳定商户号给出的不可逆未受理证据，不与一般 FAILED 混用。

## 4. 数据表、关键字段、约束与索引

### 4.1 商品与渠道

`deposit_product_series(product_code UNIQUE)` 是稳定商品根/发布锁；首版发布也先 `INSERT ... ON CONFLICT` 后锁该根。`deposit_products` 每行是一个不可变发布版本，关键字段为该根 FK/`product_code`、`version`、`name`、`common_amount`、`price_fiat`、`fiat_currency`、`sell_from/sell_until`、`published_at`、`disabled_at`。`UNIQUE(product_code,version)`；PUBLISHED 同 code 的销售半开区间用 exclusion constraint 禁止重叠，服务端按业务时点只能选中一行。新版本只能在该 code 时间线尾部追加；发布事务锁 series 根和尾行，允许把开放尾 `sell_until` 从 `NULL` 一次性收口为后继 `sell_from`，并与后继插入/审计同成败，禁止中段插版或单独改区间。`common_amount > 0`、显式售价大于零；没有 `giveaway`、匹配比例或活动期字段。

`vip_product_series(product_code UNIQUE)` 是同型稳定根。`vip_products` 每行是不可变发布版本，关键字段为该根 FK/`product_code`、`version`、`name`、`duration_days`、`price_fiat`、`fiat_currency`、权益规则版本、`sell_from/sell_until` 和发布/下架时间。`UNIQUE(product_code,version)` 且 PUBLISHED 同 code 销售区间不重叠，按业务时点唯一选中；沿用上一段相同的“锁 series 根/尾行、只收口开放尾、原子追加后继”规则。`duration_days > 0`、显式价大于零；没有首购价、优惠券、`sub_id`、续费周期或赠送资产。

`payment_channels` 保存 `provider`、`method`、能力（DEPOSIT/VIP/PAYOUT/REFUND）、法币、最小/最大金额、手续费、银行/钱包映射、排序、启停和配置版本。收款与出款通过能力字段和各自限额过滤，不能把仅收款渠道用于出款。下限和上限用 `NULL` 表示不限，不能用魔法值 0。渠道密钥存密钥管理系统，表中只保存引用。

任意金额充值和普通提现不把汇率/费率藏在 `rule_versions.content_hash` 或代码常量中：

- `deposit_rule_settings` 与 `rule_versions(domain=DEPOSIT)` 一对一，保存金额 scale、任意金额启用、全局单笔/业务日限额和 `Asia/Jakarta` 窗口；`deposit_rate_rules(rule_version_id,fiat_currency)` 保存 fiat→COMMON 汇率、舍入模式/scale。套餐仍以 `deposit_products` 显式价格为准。
- `withdrawal_rule_settings` 与 `rule_versions(domain=WITHDRAWAL)` 一对一，保存 COMMON scale、费率、最低费、单笔/日/周限额、周一边界和时区；`withdrawal_rate_rules(rule_version_id,fiat_currency)` 保存 COMMON→fiat 的汇率或明确两跳 rate snapshot 及渠道舍入。
- 发布事务验证每个启用币种恰有一行 rate、上下限有序、比例/scale 合法并让 content hash 覆盖全部结构化子表。订单/提现保存版本 ID 和命中的数值快照，历史不随新版本变化。
- 通用 `user_limit_usages` 以 `(user_id,limit_kind,window_start)` 唯一，保存 `window_end/reserved_amount/completed_amount/first_rule_version_id/last_rule_version_id/version`。规则版本不进入唯一键，因此同一 Jakarta 日/周内切换版本不会重置额度；新命令以当前规则上限减去同窗既有 reserved+completed 判断，降额不追溯已成功事实但会阻止后续新增。创建订单/提现在 primary 锁所需行并条件增加 reserved；明确失败/取消同事务释放，成功同事务从 reserved 移到 completed。它是强一致裁决聚合，不是统计 rollup；缺行用插入唯一键竞争，不读 slave。

### 4.2 `deposit_orders`

关键字段：

- `id`、唯一 `order_no`、`user_id`、`idempotency_key`、`request_hash`。
- `mode=PACKAGE/ARBITRARY`；套餐引用/版本可空但与 mode 有 CHECK。
- `common_amount`、`fiat_amount`、`fiat_currency`、汇率与舍入规则快照。
- 产品名称、套餐金额、渠道选择等必要审计快照。
- `limit_kind/window_start/window_end/limit_amount/limit_reservation_state=RESERVED|CONSUMED|RELEASED|RELEASED_THEN_CONSUMED` 及对应时间；只允许 RESERVED→CONSUMED、RESERVED→RELEASED、权威迟付时 RELEASED→RELEASED_THEN_CONSUMED，并与 usage 变动一一对应。
- `status`、`current_attempt_id`、`expires_at`、`paid_at`、`refunded_at`、`refund_canceled_at`、`is_late_paid`、失败/退款原因、创建/更新时间。
- `credit_asset_ledger_id` 在首次履约时唯一回填；可选 `refund_freeze_id/refund_settle_ledger_id/refund_release_ledger_id` 分别指向等额冻结、成功最终扣除或显式取消释放，三者受状态 CHECK 约束。

约束：金额严格为正；`PAID` 必须有 `paid_at`；`REFUNDED` 必须有 `refunded_at/refund_freeze_id/refund_settle_ledger_id` 且没有 release；`REFUND_CANCELED` 必须有 `refund_canceled_at/refund_freeze_id/refund_release_ledger_id` 且没有 settle；`PAID` 不得引用退款 freeze；表中没有任何赠送字段。建立 `UNIQUE(id,user_id)` 及唯一 `(user_id,idempotency_key)`；`(current_attempt_id,id)` 以同序 composite FK 回 `payment_attempts(id,deposit_order_id)`，current 非空时绝不允许指到另一充值单。`(user_id,id DESC)` 支持订单列表，`(status,expires_at,id)` 的部分索引支持关单扫描。

### 4.3 `user_vip_stats`、`vip_orders` 与 `vip_entitlements`

`user_vip_stats` 是每用户稳定的一行容量根，关键字段为 `user_id PRIMARY KEY`、`reserved_slots smallint`、`entitlement_slots smallint`、`version` 和时间字段。CHECK 固定 `reserved_slots BETWEEN 0 AND 64`、`entitlement_slots BETWEEN 0 AND 64`、`reserved_slots + entitlement_slots <= 64`。所有 VIP 写事务先锁 user，再按 01 章 `UserStateLockKey(type_code=125,user_id)` 锁/创建该行，之后才可按 type 130 锁 ACTIVE/QUEUED 权益；缺行只允许在已持 user 锁时 `INSERT ... ON CONFLICT` 初始化，不能以 COUNT 结果代替容量根。每次 slot 加减都必须与对应订单/权益的条件状态迁移在同一 primary 事务中完成并令 `version += 1`；计数下界或守恒不满足时整体回滚，不得事后单独“修正”计数。

`vip_orders` 关键字段为 `id/order_no/user_id`、`idempotency_key/request_hash`、`vip_product_id/product_code/product_version`、`product_name_snapshot`、`duration_days_snapshot`、`price_fiat/fiat_currency`、`quantity`、`rebate_base_common`、返佣规则/汇率/舍入快照、`status/current_attempt_id`、非空 `slot_reservation_state=RESERVED/CONSUMED/RELEASED`、`slot_reserved_at/slot_consumed_at/slot_released_at`、可空 `refund_entitlement_disposition=REVOKED|ALREADY_EXPIRED` 和支付/退款/时间字段。约束 `quantity = 1`，价格和期限为正，`rebate_base_common >= 0`，`order_no` 全局唯一，并建立 `UNIQUE(id,user_id)`、唯一 `(user_id,idempotency_key)` 及 `(current_attempt_id,id)` 到 `payment_attempts(id,vip_order_id)` 的同序 composite FK。首发 VIP 没有用户累计金额 usage；只校验显式商品价、渠道单笔限额和 64 个 slot。slot 状态只能 `RESERVED -> CONSUMED`（支付转权益）或 `RESERVED -> RELEASED`（财务确认未支付）；CHECK 要求 RESERVED 时只有 `slot_reserved_at` 非空，CONSUMED 时 `slot_reserved_at/slot_consumed_at` 非空且 `slot_released_at` 为空，RELEASED 时 `slot_reserved_at/slot_released_at` 非空且 `slot_consumed_at` 为空。仅 REFUNDED 要求 refund disposition 非空；REVOKED 必须对应来源权益 REVOKED，ALREADY_EXPIRED 必须对应来源权益 EXPIRED。CONSUMED 权益以后到期/撤销不会改写 slot 历史。

`vip_entitlements` 关键字段为 `id`、`user_id`、唯一 `vip_order_id`、商品版本、`duration_seconds_snapshot`、`status`、`starts_at`、`ends_at`、`schedule_version`、`activated_at`、`expired_at`、`revoked_at`、撤销原因、规则快照和时间。`vip_order_id/user_id` 均 NOT NULL，并以 `(vip_order_id,user_id)` composite FK 回 `vip_orders(id,user_id)`；约束 `ends_at > starts_at`，状态与时间字段匹配，禁止 A 用户订单履约为 B 用户权益。EXPIRED/REVOKED 的历史区间不改，ACTIVE/QUEUED 的计划区间只能由统一队列原语更新。

`vip_entitlement_schedule_audits` 保存 `user_id/entitlement_id/refund_vip_order_id/before_status/after_status/before_starts_at/before_ends_at/after_starts_at/after_ends_at/before_schedule_version/after_schedule_version/reason/created_at`，只追加，唯一 `(refund_vip_order_id,entitlement_id,after_schedule_version)`。退款压缩队列时，每条被撤销或移动的权益都必须有一条审计；不能无痕改开始/结束时间。

索引：

- 唯一 `(vip_order_id)`，保证一个订单只履约一次。
- `vip_orders(user_id,id) WHERE slot_reservation_state='RESERVED'` 支持 reservation 对账与 blocker 查询，不扫描历史终态订单。
- `(user_id, status, starts_at, id)` 支持鉴权和队列处理。
- `(starts_at, id) WHERE status=QUEUED` 与 `(ends_at, id) WHERE status=ACTIVE` 支持 primary 到期任务。
- 部分唯一 `(user_id) WHERE status=ACTIVE` 作为并发下最后防线；事务仍需先锁 `user_vip_stats` 容量根，再锁最多 64 条未终结权益并计算无重叠区间。
- primary 对用户 `u` 对账必须验证 `user_vip_stats.reserved_slots = COUNT(vip_orders WHERE user_id=u AND slot_reservation_state='RESERVED')`、`entitlement_slots = COUNT(vip_entitlements WHERE user_id=u AND status IN ('ACTIVE','QUEUED'))`，以及二者之和不超过 64；任何非零差异阻止该用户继续建单/履约并触发 P0 修复，不能用运行时 COUNT 静默覆盖计数根。

### 4.4 `payment_attempts`

一行是一笔充值或 VIP 订单在一个渠道上的收款尝试：

- `id`、`attempt_no`、`kind=COLLECT/REFUND`。
- `deposit_order_id`、`vip_order_id` 恰有一个非空，并有真实外键。
- `channel_id`、`provider`、`method` 和渠道配置版本快照。
- 稳定 `merchant_request_no`、可空 `psp_ref`、请求金额/币种。
- `status=CREATED|PENDING|UNKNOWN|SUCCEEDED|FAILED|NOT_ACCEPTED`、`pay_url`、`provider_expires_at`、`next_run_at`、`attempt_count`、`last_error_code` 和脱敏错误摘要。NOT_ACCEPTED 只允许 REFUND，必须有 `not_accepted_at/not_accepted_evidence_hash`；COLLECT 禁止该状态。
- COLLECT 迟付纠正可有 `corrected_from_failed_at/correction_provider_event_id`；仅 FAILED→SUCCEEDED 时非空并必须引用同 provider/merchant/金额/币种的已验证事件。
- COLLECT 保存 `late_payment_risk_until/finality_status=OPEN|CONFIRMED_PAID|CONFIRMED_NOT_PAID/finality_next_run_at/finality_checked_at/finality_evidence_hash/finality_attempt_count/finality_lease_owner/finality_lease_until/finality_lease_generation`；EXPIRED/FAILED 不等于财务最终未收款。只有超过渠道契约风险窗后，以稳定商户号完成权威查单/对账并取得 CONFIRMED_NOT_PAID 证据，才可解除注销 blocker；不能提供可验证最终性的渠道不得让相关账号完成注销。finality lease 的 owner/until 成对、generation 单调增加，旧代次不得 finalize。
- 脱敏请求/响应摘要、`request_hash`、创建/更新时间/成功时间。

约束与索引：

- CHECK 两种业务外键恰有一个；金额为正、状态白名单。REFUND NOT_ACCEPTED 要求权威 evidence/time、没有未知请求且父 VIP order 同事务转 REFUND_NOT_ACCEPTED；其他状态两字段为空。充值退款仍使用独立 REFUND_CANCELED 业务终态，不把 attempt NOT_ACCEPTED 直接当作已释放钱包冻结。
- 显式建立 `UNIQUE(id,deposit_order_id)` 与 `UNIQUE(id,vip_order_id)` 作为两类父订单 current pointer 的 exact composite FK 目标；父订单 `(current_attempt_id,id)` 仅在 current 非空时引用对应一组。延迟约束触发器再校验 current attempt 的 kind/status 与父订单状态，不能把两个单列 unique/FK 拼成跨订单指针。
- `kind='COLLECT'` 时 `late_payment_risk_until/finality_status/finality_next_run_at/finality_attempt_count/finality_lease_generation` 均非空，创建值固定为 `OPEN/0/0`，`finality_next_run_at >= late_payment_risk_until`；风险窗来自订单所用渠道版本的不可变契约快照，不能由客户端传入。`kind='REFUND'` 时全部 finality 专用字段为空。
- OPEN 必须没有终态 evidence，允许保存最近未知检查时间但必须有下一运行时间；CONFIRMED_PAID/CONFIRMED_NOT_PAID 必须有 `finality_checked_at/finality_evidence_hash`、没有 `finality_next_run_at` 且 lease 已清空。owner/until 成对，只有 OPEN 可持有；generation 非负且只能认领时增加。
- 唯一 `(provider, merchant_request_no)`；`psp_ref IS NOT NULL` 时唯一 `(provider, psp_ref)`。
- COLLECT 分别使用不带状态的 `(deposit_order_id,kind) WHERE kind='COLLECT'` 和 `(vip_order_id,kind) WHERE kind='COLLECT'` 唯一索引，每个订单全生命周期只有一个收款 attempt；UNKNOWN、FAILED 纠正、worker 重启和人工补单都复用同一行与 `merchant_request_no=<order_no>:collect:1`。要换渠道必须新建独立订单，旧单仍可承接迟付，不能在同一订单创建第二个真实扣款意图。`REFUND` 同样分别使用不带状态的订单/kind 唯一索引，保证两类订单全生命周期各只有一个逻辑退款 attempt；所有重试更新同一行并复用 `merchant_request_no=<order_no>:refund:1`。不支持稳定商户号幂等创建与查单的渠道不启用相应能力。
- `(status, next_run_at, id) WHERE status IN (CREATED,PENDING,UNKNOWN) AND finality_status='OPEN'` 是 COLLECT primary sweep 索引；REFUND 使用 `(next_run_at,id) WHERE kind='REFUND' AND status IN ('CREATED','PENDING','UNKNOWN')`。NOT_ACCEPTED/SUCCEEDED/FAILED 都已退出普通 refund sweep；CONFIRMED_NOT_PAID 的 COLLECT 不能再被普通提交/查单 sweep 领取。
- `(finality_next_run_at,finality_lease_until,id) WHERE kind='COLLECT' AND finality_status='OPEN'` 是迟付最终性 primary sweep 索引，覆盖父订单或 attempt 已 EXPIRED/FAILED 的情形；普通 attempt 状态终结不能让该行从最终性扫描中消失。
- `pay_url` 是展示结果，不是支付成功事实；缺失不妨碍回调查单。
- 迟付证据与注销完成并发时统一先锁 attempt/order 再锁 closure。只有 `finality_status=OPEN` 的不确定性本身持续阻塞；确认收款的事务原子履约并置 CONFIRMED_PAID，提交后由新增资产、VIP 权益、退款或其他真实事实继续阻塞，CONFIRMED_PAID 这一历史标志本身不永久阻塞清退完成。已记 CONFIRMED_NOT_PAID 后又出现相反收款证据时，事件事务创建/命中 `COLLECT_SUCCESS_AFTER_CONFIRMED_NOT_PAID` PAYABLE case并 P0 告警；订单、slot/usage 与钱包终态保持不变。

### 4.5 `withdrawals`

关键字段：

| 字段组 | 关键字段 |
| --- | --- |
| 身份/幂等 | `id`、`withdrawal_no`、`user_id`、`command_deduplication_id UNIQUE`、`idempotency_key`、`request_hash`；后两者仅作审计快照 |
| 用途/安全 | `purpose=USER_REQUEST/ACCOUNT_CLOSURE`、`closure_id nullable`；`otp_mode=NOT_REQUIRED|TOTP`、条件 `otp_binding_id/otp_accepted_step` 只保存已成功验证的 Authenticator/TOTP 审计快照，不保存 `otp_code` |
| 资产 | 固定 `asset_type=COMMON`、唯一 `freeze_id` |
| 金额 | `requested_common`、`fee_common`、`net_common`、`payout_fiat`、`fiat_currency` |
| 规则快照 | 汇率、费率、最低手续费、周界、业务时区、日/周限额规则版本；实际两条 reservation 见下表 |
| 收款快照 | 用户银行卡引用以及银行名/渠道码/账号/户名/手机号的下单快照 |
| 状态 | `status`、`current_attempt_no`、`channel_switch_count`、可空 `settled_by_attempt_id`、`payout_disposition_generation`、`next_run_at`、`attempt_count`、`last_error_code` 和脱敏错误摘要 |
| 出款领取 | `dispatch_lease_owner/dispatch_lease_until/dispatch_lease_generation`；只用于 APPROVED sweep 短领取，owner/until 成对为空/非空，generation 每次成功认领加一 |
| 当前审核摘要 | `current_decision`、`current_reviewer_type=ADMIN/SYSTEM`、条件 actor 字段、`current_reviewed_at`、当前原因摘要；只用于列表/条件约束 |
| 终态 | `arrived_at`、`resolved_at`、`fail_reason`、时间字段 |

约束：

- `requested_common > 0`、`fee_common >= 0`、`net_common = requested_common - fee_common AND net_common > 0`、`payout_fiat > 0`。
- `asset_type` 只能为 COMMON；`freeze_id` 唯一且非空。
- `otp_bindings` 显式建立 `UNIQUE(id,user_id)`；若创建时存在 ACTIVE binding，withdrawal 必须写 `otp_mode=TOTP`，`(otp_binding_id,user_id)` 以完全同序 composite FK 引用它，且 `otp_accepted_step` 非空；两列记录本次已消费 step 的审计快照。若从未成功确认过 binding，必须写 `otp_mode=NOT_REQUIRED` 且 `otp_binding_id/otp_accepted_step` 全空；仅有未确认 PENDING/EXPIRED 历史仍属于该分支。CHECK 固定这两组互斥完整性。至少一条历史 binding 曾确认但当前没有 ACTIVE 时禁止创建，不产生 withdrawal，不能用 NOT_REQUIRED 伪装 NEVER_BOUND；管理员 reset 记录为 `REVOKED + ADMIN_RESET`。binding 后续被吊销不改历史提现；`otp_code` 永不落表。
- `purpose=USER_REQUEST` 时 `closure_id/closure_withdrawal_seq` 都为空；`purpose=ACCOUNT_CLOSURE` 时必须引用当前用户未完成的 `account_closures` 且 sequence 为正。CLOSING 用户只能走后者，金额必须等于事务锁内重算的本批 `min(available_common,单笔上限,渠道上限,DAY剩余额度,WEEK剩余额度)`；只有全部余额不超过本批上限时才会一次清空。
- `REJECTED`、`APPROVED` 及其后续状态必须有当前审核决定、主体和时间。ADMIN 要求 `current_reviewer_admin_id` 非空并引用 `admin_users`；SYSTEM 要求 `current_reviewer_system_code='WITHDRAWAL_RISK_ENGINE'` 和 `current_system_policy_version_id` 非空、admin ID 为空。自动审核不得伪造后台用户或留全空。`CANCELED` 必须有 `canceled_at`，取消主体就是订单 `user_id`。
- `current_attempt_no/channel_switch_count` 非负且只增；APPROVED 时均为 0，PAYING/FAIL_CONFIRMING 至少已有 attempt 1；新 attempt 只能由初始 dispatch 或受控 Switch 命令递增一次产生。
- `payout_disposition_generation` 非负只增；`settled_by_attempt_id` 必须以 `(attempt_id,withdrawal_id)` composite FK 指向同单 attempt。ARRIVED 且仍存在 `financial_status=SUCCESS_CONFIRMED` 时，它必须指向其中 `attempt_no` 最小者；若全部曾成功 attempt 已反转则为空，并由唯一未被反向处置的 `PAYOUT_FAILURE_AFTER_LOCAL_ARRIVED` PAYABLE case解释外部缺口。
- `ARRIVED` 必须有 `arrived_at`，`CANCELED/REJECTED/FAILED/ARRIVED` 四个终态必须有 `resolved_at`。
- `dispatch_lease_owner/dispatch_lease_until` 必须同时为空或同时非空，`dispatch_lease_generation >= 0`；只有 APPROVED/PAYING 可暂存 lease，进入终态必须清空。执行事务仅在 owner、generation 完全匹配且 lease 未过期时消费，旧 owner/旧代次不能创建或提交 attempt。
- 状态和冻结终态的跨表关系由事务写原语与对账保证，不能用触发器隐藏业务逻辑。

索引为：唯一 `withdrawal_no`、唯一 `command_deduplication_id`、`(user_id, id DESC)`、待审部分索引 `(id) WHERE status=AWAIT_REVIEW`、领取索引 `(next_run_at,dispatch_lease_until,id) WHERE status=APPROVED`、可运行部分索引 `(next_run_at,id) WHERE status IN (PAYING,FAIL_CONFIRMING)`。不得对 `(user_id,idempotency_key)` 单独建唯一索引：同一用户在不同 RPC 合法复用相同 key 时必须由 02 章含 `rpc_name` 的全局命令作用域区分。

`withdrawal_limit_reservations` 每笔提现固定两行，保存 `withdrawal_id/limit_kind=WITHDRAWAL_DAY|WITHDRAWAL_WEEK/window_start/window_end/amount/state=RESERVED|CONSUMED|RELEASED/usage_version_snapshot/created_at/terminal_at`，主键 `(withdrawal_id,limit_kind)`。amount 等于 `requested_common` 且为正；两行的用户与窗口由 withdrawal 的已发布规则快照重算验证。创建时与两条 `user_limit_usages.reserved_amount` 同事务写 RESERVED；ARRIVED 时两行条件转 CONSUMED并分别执行 reserved→completed，CANCELED/REJECTED/FAILED 时两行转 RELEASED并分别减 reserved。ACCOUNT_CLOSURE 也不绕过 DAY/WEEK：额度不足时 quote 返回下一窗口，之后继续下一有界批次。终态事务必须恰好锁/更新两行，影响行数不是 2 或 usage 不足时整体回滚；重复终态读取原结果，不再次搬移。

`withdrawal_reviews` 是追加式审核事实，关键字段为 `id`、`withdrawal_id`、`from_status`、`decision=APPROVE/REJECT`、`to_status`、`reviewer_type=ADMIN/SYSTEM`、条件字段 `reviewer_admin_id/reviewer_system_code/system_policy_version_id`、`reason`、`batch_no`、`idempotency_key/request_hash`、IP/客户端摘要和 `created_at`。CHECK 要求 ADMIN 仅 admin ID 非空，SYSTEM 仅稳定 system code + policy version 非空；唯一 `(withdrawal_id, idempotency_key)`，索引 `(withdrawal_id,id)` 和 admin 条件索引。应用角色只能 INSERT/SELECT，不能 UPDATE/DELETE。审核事务先插入审核事实，再把同一决定摘要写到 `withdrawals`；两者任一失败都整体回滚。

### 4.6 `payout_attempts`

关键字段为 `id`、`withdrawal_id`、`attempt_no`、`provider/channel_id`、稳定商户请求号、`psp_ref`、法币金额/币种、收款方式快照摘要、`status`、`financial_status=NO_SUCCESS|SUCCESS_CONFIRMED|SUCCESS_REVERSED`、`success_evidence_id/success_confirmed_at/reversal_evidence_id/success_reversed_at`、`next_run_at`、`attempt_count`、`work_lease_owner/work_lease_until/work_lease_generation/work_claim_due_at`、`last_error_code`、脱敏错误摘要、请求/响应 hash、`not_accepted_evidence_hash/not_accepted_at`、`stop_reason_attempt_id`、提交/终结时间。`work_claim_due_at=COALESCE(work_lease_until,'-infinity')` 为生成列。操作 `status=SUCCEEDED` 可以保留“该请求曾成功”的历史；当前净成功真值只读取单向 `financial_status`，后者只能 `NO_SUCCESS -> SUCCESS_CONFIRMED -> SUCCESS_REVERSED`，不能从 REVERSED 回跳。CHECK 固定 `status=SUCCEEDED` 当且仅当 financial status 为 SUCCESS_CONFIRMED/REVERSED；SUCCEEDED 的 `next_run_at/work_lease_owner/work_lease_until` 全空，其他状态只能配 NO_SUCCESS。

- 唯一 `(withdrawal_id, attempt_no)`、`(provider, merchant_request_no)`、以及非空时的 `(provider, psp_ref)`。
- 每个提现最多一个 `CREATED/SUBMITTING/PENDING/UNKNOWN/STOP_CONFIRMING` 活动尝试，partial unique 必须包含 CREATED；`NOT_ACCEPTED` 必须有权威证据/时间且不再提交。STOPPED_BEFORE_SUBMIT 必须没有 provider acceptance/psp ref；STOP_CONFIRMING 只能查单。两种 STOP 都必须保存 `stop_reason_attempt_id`，并以 `(stop_reason_attempt_id,withdrawal_id)` composite FK 指向同一 withdrawal 的另一当时 `SUCCESS_CONFIRMED` attempt，不得自引或串单。稳定商户号固定为 `<withdrawal_no>:payout:<attempt_no>`。状态迁移触发器只额外允许 `FAILED|NOT_ACCEPTED|STOPPED -> SUCCEEDED` 三条证据化纠正边，要求匹配的成功 evidence/reconciliation；STOPPED_BEFORE_SUBMIT 无论何种回调都不能成功。
- financial status 与证据成组：NO_SUCCESS 的四个成功/反转字段为空；SUCCESS_CONFIRMED 只有成功证据/时间；SUCCESS_REVERSED 两组都非空且反转证据晚于并引用同一 provider/merchant/金额/币种。一次证据只能引用一个 attempt；普通失败通知不得写 SUCCESS_REVERSED。纠正为 SUCCEEDED 时不清除 `not_accepted_*`、stop 或失败摘要，它们作为先前裁决证据永久保留；新成功证据、状态、财务状态、lease 清理与集合重算同一事务。
- work lease owner/until 成对、generation 非负只增；领取、续租、提交前标记和网络结果 finalize 都完整匹配 `(status,owner,generation,until)` 且要求未过期，旧 owner 影响 0 行。`(next_run_at,work_claim_due_at,id) WHERE status IN ('CREATED','SUBMITTING','PENDING','UNKNOWN','STOP_CONFIRMING','NOT_ACCEPTED')` 的严格同序 partial 索引供 attempt-level primary sweep；即使 withdrawal 已 ARRIVED，STOP_CONFIRMING 仍可独立领取且只查单。NOT_ACCEPTED worker 只在锁内证明无合格替代渠道或已达 attempt 上限时重放 `FinalizeNotAcceptedPayoutFailure`，否则释放 lease 并退避；不替操作员自动选择替代渠道。

`payout_channel_switches` 保存 `id/withdrawal_id/from_attempt_id/to_attempt_id/from_channel_id/to_channel_id/from_status_snapshot=NOT_ACCEPTED/not_accepted_evidence_hash/approval_id/deduplication_id/reason/ticket_no/operator_admin_id/created_at`。`from_attempt_id`、`to_attempt_id`、approval 和 deduplication 分别唯一；创建时两 attempt 必须属于同一 withdrawal 且 `to.attempt_no=from.attempt_no+1`，旧状态为 NOT_ACCEPTED、新状态为 CREATED，渠道必须不同。它与 withdrawal 当前 attempt 指针、新 attempt、审批消费和审计同事务创建，不提供 DELETE/UPDATE；旧 attempt 后续被权威成功纠正时保留本行和 NOT_ACCEPTED 快照，不用当前父状态反向否定历史切换。平台硬常量 `PAYOUT_ATTEMPT_MAX=8`；DDL CHECK 强制 `attempt_no BETWEEN 1 AND 8`、`withdrawals.channel_switch_count BETWEEN 0 AND 7`，达到上限只能失败释放，不能继续扩大终态锁集。

`payout_failure_resolutions` 保存 `id/withdrawal_id/attempt_id/attempt_status_snapshot=NOT_ACCEPTED/not_accepted_evidence_hash/resolution_kind=NO_ELIGIBLE_CHANNEL|OPERATOR_ABORT|ATTEMPT_LIMIT_REACHED/approval_id nullable/deduplication_id/reason/ticket_no/operator_admin_id nullable/created_at`。`withdrawal_id`、`attempt_id`、deduplication 分别唯一；创建时 attempt 必须是该 withdrawal 当前 NOT_ACCEPTED attempt，证据 hash 完全一致。它与 `payout_channel_switches.from_attempt_id` 由创建时延迟约束触发器互斥：一个 NOT_ACCEPTED attempt 只能切换或失败释放一次。自动 `NO_ELIGIBLE_CHANNEL/ATTEMPT_LIMIT_REACHED` 保存锁内渠道候选/attempt 上限的规范决策 hash；人工 OPERATOR_ABORT 必须消费异人审批。若之后出现权威成功，resolution 仍是不可变历史事实，attempt 走证据化 SUCCEEDED 纠正并由本地 FAILED 分支建立应收，不删除 resolution 或重开 freeze/usage。

`payout_success_dispositions` 每个曾成功 attempt 恰一行，保存 `attempt_id UNIQUE/withdrawal_id/disposition=CANONICAL|DUPLICATE|LATE_AFTER_LOCAL_FAILURE|REVERSED/disposition_generation/canonical_attempt_id nullable/finance_case_id nullable/last_evidence_id/version/updated_at`。建立 `UNIQUE(id,attempt_id,withdrawal_id)` 和同 withdrawal composite FK。ARRIVED 下 CANONICAL 要求 canonical=self、case 空，DUPLICATE 要求 canonical 指向当前最小未反转 attempt 且 case 为该 attempt 的唯一 `DUPLICATE_PAYOUT_AFTER_CHANNEL_SWITCH` RECEIVABLE；FAILED/REJECTED/CANCELED 下每个未反转成功都必须为 LATE_AFTER_LOCAL_FAILURE、canonical 空并引用自身 `PAYOUT_SUCCESS_AFTER_LOCAL_FAILED` RECEIVABLE；REVERSED 要求 canonical/case 都空且 attempt 已 SUCCESS_REVERSED。`payout_success_disposition_events` 以 `(attempt_id,disposition_generation)` 唯一追加前后状态、触发 event/reconciliation、前后 active-success set hash和原/counter case；更新 current row、withdrawal generation/canonical、event 和所有案件必须同一事务。

`finance_case_source_reversals` 是支付与出款共用的强类型来源反转事实，保存 `original_case_id UNIQUE/trigger_source_type=PAYMENT_PROVIDER_EVENT|PAYOUT_PROVIDER_EVENT|RECONCILIATION_ITEM/trigger_payment_event_id nullable/trigger_payout_event_id nullable/trigger_reconciliation_item_id nullable/trigger_component/evidence_hash/outcome=CANCELED_UNSETTLED|NO_EFFECT_WAIVED|COUNTER_CASE_CREATED/counter_case_id nullable/created_at`。CHECK 按 source type 要求三个 trigger ID 恰一非空；每个 ID 是到对应已验签/封存父表的真实 FK，延迟约束触发器再由 registry 复验 provider、merchant、金额、币种、原 case source/component 和新终态能够消除原差异，不能用自由文本 evidence 触发。payout 分支另从原 case 强类型 source 回查 attempt，不在通用表硬编码 attempt 列。原案 OPEN/IN_REVIEW 时，同行条件转 `CANCELED_BY_SOURCE_REVERSAL` 且 outcome=`CANCELED_UNSETTLED`；WAIVED 只记 `NO_EFFECT_WAIVED`；RESOLVED 必须 outcome=`COUNTER_CASE_CREATED` 并引用 registry 生成的唯一反向 case。CHECK 要求前两种 outcome 的 `counter_case_id` 为空、后一种非空。任何人工操作员都不能选择 outcome、金额或方向；它们从原案、业务快照和新权威证据确定。

### 4.7 `payment_provider_events` 与 `payout_provider_events`

两张表分别保存已验签的收款/退款事件和出款事件，避免一个万能回调表承担不同权限、保留期和查询压力。公共字段为 `provider`、`event_kind`、`provider_event_id`、`callback_key`、本地商户单号、`psp_ref`、标准状态、金额/币种、原始 body hash、签名 key 版本、`process_result`、关联 order/attempt、接收/处理时间。

- 两张表分别唯一 `(provider,provider_event_id)`；渠道没有 event ID 时用非空唯一 `callback_key` 兜底，该键由契约稳定字段构造，不能使用接收时间。键必须包含 collect/refund/payout 方向，不能跨表/方向碰撞。
- 原始载荷如确需留存必须加密、脱敏并设置保留期；日志不打印完整载荷、银行卡或密钥。
- 这两张表都是入站审计和防重表，不被 worker 当作待发送事件，不是 Outbox。

## 5. API

### 5.1 用户 API

| HTTP | 说明 |
| --- | --- |
| `GET /v2/deposit-products` | 固定套餐及精确到账 COMMON；响应无赠送字段 |
| `GET /v2/payment-channels?purpose=DEPOSIT&amount=...`（purpose 也可为 VIP） | 返回全局与渠道限额交集、币种和有效配置版本 |
| `POST /v2/me/deposit-orders/create` | 创建套餐/任意金额充值，要求 `Idempotency-Key` |
| `GET /v2/me/deposit-orders/detail?order_no=...`、`GET /v2/me/deposit-orders` | 查询本人充值状态/列表；详情 selector 放 query，列表使用 `offset + limit + total` |
| `GET /v2/vip-products` | 已发布一次性 VIP 显式价格和期限 |
| `POST /v2/me/vip/orders/create` | 数量固定 1，创建一次性支付 |
| `GET /v2/me/vip/orders/detail?order_no=...`、`GET /v2/me/vip/entitlements` | 查询本人订单和权益；详情 selector 放 query，列表使用 `offset + limit + total` |
| `POST /v2/me/withdrawals/create` | 创建普通 COMMON 提现并冻结，要求 `Idempotency-Key`；ACTIVE Authenticator/TOTP binding 存在时请求体必须带 `otp_code`，CLOSING 用户禁用 |
| `GET /v2/me/closure/withdrawal-quote` | 返回当前可清退批次、限制来源和下个额度窗口；只读 primary，不预占 |
| `POST /v2/me/closure/withdrawals/create` | `CreateClosureWithdrawal`；CLOSING 用户按服务端批次清退 COMMON，同一 closure 最多一笔非终态清退单；使用与普通提现相同的条件 TOTP 规则 |
| `GET /v2/me/withdrawals/detail?withdrawal_no=...`、`GET /v2/me/withdrawals` | 查询本人提现详情/列表；详情 selector 放 query，列表使用 `offset + limit + total` |
| `POST /v2/me/withdrawals/cancel` | `withdrawal_no` 放 Proto body；仅本人在 AWAIT_REVIEW 取消，并原子释放冻结 |

`POST` 的用户主体只来自令牌；请求体不允许 `user_id`。创建接口返回本地业务单和当前渠道尝试，即使外部创建超时也返回可轮询状态，不能为了“重试页面”再造一张单。

### 5.2 渠道回调

支付/出款 callback 使用专用裸 HTTP adapter，不生成 callback Proto，也不经过 gRPC-Gateway 的 JSON→Proto 解码。每个已启用 provider/kind 组合注册一个配置绑定的精确字面量路由，例如 `/v2/callbacks/payment/provider-a/collect`、`/v2/callbacks/payment/provider-a/refund`、`/v2/callbacks/payment/provider-a/payout`；这些示例中的 `provider-a` 是部署时登记的静态 adapter 名，不是路径参数。provider、kind 和验签算法不能由 query/body 选择。adapter 限长读取原始 body，使用原始字节与 headers 校验内容类型、时间窗和签名，再解析成内部 typed command，最终返回渠道要求的 ACK；它不包含领域履约分支，也不返回通用 Proto/errkit JSON。

签名验证后的内部 typed command 统一交给 `SettlePaymentEvent` 或 `SettlePayoutEvent`。未知 provider/kind、验签失败、商户不匹配、金额/币种不符都不能进入资金事务。

### 5.3 管理 API

- 商品、渠道和费率采用草稿/发布版本 API；已被订单引用的版本不可原地修改。
- 充值/VIP 订单只读查询与人工确认支付拆分权限；人工确认必须提供渠道证据并走同一履约器。
- `POST /v2/admin/vip/orders/refund/start` 对应 `StartVipRefund`，`order_no` 放 Proto body，并要求 `payment.refund`、`Idempotency-Key`、期望订单版本、原因和工单；只允许 PAID 且唯一来源权益存在。命令事务创建/命中每单唯一 REFUND attempt、把订单置 REFUND_PENDING 并写审计，事务中不调用渠道。之后所有提交/回查复用 `<order_no>:refund:1`；不存在第二退款 attempt 或本地 cancel RPC。
- 提现提供列表、详情、单笔审核、受控重试和对账标记；不存在任意 `set_status`。批量审核只是最多 `WITHDRAWAL_REVIEW_BATCH_MAX=100` 个 selector 的编排接口，每单独立调用单笔事务并返回逐项结果，不存在一笔跨 100 单的长资金事务。
- `POST /v2/admin/withdrawals/payout-channel/switch` 要求 `withdrawal.payout`；`withdrawal_no`、`expected_withdrawal_version/expected_attempt_no`、目标 channel、reason/ticket 放 Proto body，并要求 Idempotency-Key 和 request hash 完全匹配的异人审批；仅下述 NOT_ACCEPTED 分支可执行。人工补单、退款、渠道切换和资产修正均保留操作日志。
- `POST /v2/admin/withdrawals/not-accepted/finalize` 使用 `withdrawal.payout.fail`；`withdrawal_no`、期望版本/attempt、权威证据 hash 和原因放 Proto body，并要求 Idempotency-Key。无可用候选或已达 8 次上限可按发布规则自动裁决；主动放弃仍可用渠道必须消费异人审批。该接口不是任意 set-status，只能执行下述原子失败释放。
- 第三方返佣无法安全钱包扣回时，只通过 `ListFinanceRecoveryCases`、`GetFinanceRecoveryCase`、`StartFinanceRecoveryReview`、`ResolveFinanceRecoveryCase`、`WaiveFinanceRecoveryCase` 管理唯一案件；查询、复核/解决、应收豁免权限分别为 `finance.recovery.read/resolve/waive`。Resolve 需受信外部证据；Waive 只接受 RECEIVABLE 且需异人审批，PAYABLE 只能凭完整外部付款证据 RESOLVED。任何命令都不能修改订单、原 GRANT/REVERSAL 或用户资产。完整 API、状态和锁序见 [02-API与鉴权规范.md](02-API与鉴权规范.md) §8 与 [06-等级团队与返佣.md](06-等级团队与返佣.md) §5.4.1。

## 6. 主流程

本章任何结果事务只要可能写 `asset_ledgers`，都必须在 command/provider event 防重之后、取得 payment/payout attempt、订单、提现或其他业务根之前，先调用 `AcquireIncomeWriteFences`，依次取得 income root SHARE 与当前 `business_date` 的 OPEN day fence。业务根之后先 `PrelockUserClosures`，终结路径再按需锁 freeze，最后 `LockUsersAndValidateClosureGate`，按订单/来源登记时点裁决 ACTIVE/CLOSING/CLOSED 并唤醒 blocker。不产生资产效果的建单/查单事务不取 root/day fence，但新建单仍经过 closure/user gate并要求 ACTIVE；`ApplyMutations/OpenFreeze/ReleaseFreeze/SettleFreeze` 只断言，不得懒获取。创建提现在 user gate 后、KYC/银行卡/限额前锁定并裁决 Authenticator/TOTP binding。

### 6.1 创建充值

1. 从令牌取 `user_id`，解析十进制金额并校验正数、精度、已发布 `deposit_rule_settings` 全局限额和渠道限额交集。
2. 套餐模式读取已发布产品版本；任意金额模式读取同一 DEPOSIT 版本下的 `deposit_rate_rules`。服务端得到候选 `common_amount` 与 `fiat_amount`；事务外结果不预占额度，也不代表账号可创建外部收款意图。
3. primary 事务先锁命令防重，重验产品/规则/渠道快照，再执行 `PrelockUserClosures` 与 `LockUsersAndValidateClosureGate` 并要求用户仍为 ACTIVE，随后才锁 `user_limit_usages` 并预占业务日额度。插入 `deposit_orders(PENDING,limit_reservation_state=RESERVED)` 和 `payment_attempts(CREATED,next_run_at=now,finality_status=OPEN,late_payment_risk_until=<渠道契约快照>,finality_next_run_at=late_payment_risk_until)`，保存 usage、窗口、金额、规则和请求幂等结果；缺少可验证迟付最终性契约的渠道不能被选中。提交。Request/CompleteClosure 也锁同一 closure/user gate，因此不存在 CLOSING/CLOSED 后才提交的新 COLLECT 意图。
4. 提交后可立即走一次外部创建支付的快速路径；渠道返回后用新事务条件更新 attempt 的 `psp_ref/pay_url/status/next_run_at`。
5. 外部超时写 `UNKNOWN` 并保留原商户单号，客户端轮询原订单；后台 primary sweep 负责查询和恢复。

支付完成时只执行 `COMMON/INCOME(common_amount)`，幂等键为 `deposit:credit:<order_no>`。不存在充值赠送流水、匹配任务或活动回调。

### 6.2 创建一次性 VIP

1. 服务端读取已发布 `vip_products` 行，订单数量强制为 1；价格、币种和期限完全来自该版本，事务外只形成候选快照。
2. primary 事务先锁命令防重并重验商品/渠道，随后 `PrelockUserClosures`，再用 gate token 调用 `LockUsersAndValidateClosureGate` 锁 `users` 并要求 ACTIVE，按 type 125 锁/初始化 `user_vip_stats`。必要时调用下面有 64 行硬界的 `EnsureVipEntitlementCurrent` 收敛已到期权益；锁内以条件 UPDATE 要求 `reserved_slots + entitlement_slots < 64` 并令 `reserved_slots += 1`，再插入 `vip_orders(PENDING,slot_reservation_state=RESERVED,slot_reserved_at=now)` 与 `payment_attempts(CREATED,next_run_at=now,finality_status=OPEN,late_payment_risk_until=<渠道契约快照>,finality_next_run_at=late_payment_risk_until)`。slot reservation、订单、attempt 和命令结果同事务；重复命令直接返回原订单，不重复加 slot。REFUND attempt 不填写这些 COLLECT 专用字段；CLOSING/CLOSED 或容量已满不能创建外部收款意图。
3. 外部收款创建、超时和恢复与充值相同。Apple/Google 凭证如需联网验证，也必须在事务外完成并归一为一次性支付事件。
4. 支付履约事务锁定 provider event 后调用 `AcquireIncomeWriteFences`，依次取得 `rollup_roots('income') FOR SHARE` 和 finance day fence，再锁 payment attempt（涉及时）和订单；随后仅从 primary 预读前两层关系、受益人/等级/任务聚合 ID，用于收集全集而不携带裁决。对购买者与候选受益人执行 `PrelockUserClosures` 后，按 `UserStateLockKey` 先锁完全部 users/关系/等级/任务并调用 `LockUsersAndValidateClosureGate`，再锁购买者 `user_vip_stats(type=125)`；持有 stats 根后才从 primary 重读 ACTIVE/QUEUED 集合，核对 `entitlement_slots` 并按 `(starts_at,id)` 锁齐最多 64 条权益(type=130)。购买者 ACTIVE 正常履约；订单/attempt 在 closure cutoff 前已建立时，CLOSING 仍必须收下权威支付并把新增权益/未来退款登记为 blocker。理论上 finality OPEN 会阻止其变成 CLOSED；若数据损坏仍出现 CLOSED，记录 P0 渠道差异并走原路退款/Finance 清算，绝不向 CLOSED 账号补权益。订单的 `slot_reservation_state` 必须仍为 RESERVED；锁内先执行同一有界 Ensure 收敛过期项，再把新权益与剩余队列按 `(paid_at,order_id)` 稳定排序/重排，无现有权益则 ACTIVE，否则生成 QUEUED 区间。仅当订单条件更新 `RESERVED -> CONSUMED` 命中一行且 `reserved_slots >= 1` 时，stats 才在同事务原子执行 `reserved_slots -= 1, entitlement_slots += 1`；否则读原结果或整体回滚。reservation 已保证总占用不超过 64，不能在履约时临时扩容或驱逐别人的权益。
5. 对每个候选返佣槽位先按 FLAT 规则计算：ZERO_DIFF/ZERO_ROUNDED 只写 calculation audit；rounded>0 后，ACTIVE 才创建 `ONE_TIME_VIP_PURCHASE` GRANT 并将其 COMMON 账户按 `(user_id,asset_type)` 排序后调用 `OpenFreeze(INCOME_FROZEN)`，CLOSING/CLOSED 则按 `(source_type,source_id,source_component,beneficiary_user_id)` 写唯一正额 `benefit_forfeiture_audits`。任何未授予份额都不转给另一层。购买者任务仍以唯一来源 `ONE_TIME_VIP_ENTITLEMENT_CREATED` 推进；第三方状态不能否决购买者履约。
6. `vip_orders` 置 PAID、slot 转换、stats 守恒、权益、适格冻结返佣/未授予审计、任务进度和审计全部成功后才提交。任何一步失败都保持原 RESERVED 待履约，由同一支付事件重放；已 CONSUMED 的相同事件读取唯一权益返回原结果，不再次移动计数。

没有首购查询、优惠券锁、自动续订订单、订阅 token、转移或 VIP 赠币步骤。

### 6.3 支付回调履约

1. 事务外读取原始 body 并验签；适配器生成标准事件及稳定 `callback_key`。
2. primary 事务先插入/读取 `payment_provider_events`；本次为成功履约时调用 `AcquireIncomeWriteFences`，严格取得 income root SHARE 后再取得 finance day fence，随后按固定顺序锁 `payment_attempt` 和业务订单。拒绝/纯状态事件不写流水时同时跳过 root/day fence。
3. 校验 provider、商户单号、psp_ref、商户号、金额、币种和订单快照。已成功同语义重放直接返回；冲突写事件结果并告警，不动业务事实。
4. 充值调用 `ApplyMutations(INCOME)`，并把其订单金额 reservation 在同事务从 RESERVED 搬到 CONSUMED；若此前本地失败已 RELEASED，则以来源订单自然键直接幂等增加 completed。VIP 执行上一节的权益、冻结返佣和中性任务进度原子履约，不访问未定义的金额 usage。随后把 COLLECT attempt 的 `finality_status` 置 CONFIRMED_PAID、清空 finality lease，并把 attempt 和业务订单置为成功，回填渠道单号和支付时间；仅 `finality_status=OPEN` 的普通 FAILED 可凭权威收款纠正为 SUCCEEDED，CONFIRMED_NOT_PAID 后的相反成功禁止走本履约器。
5. 本地事务提交后才向渠道 ACK 成功。数据库暂时失败则返回渠道可重试响应；即使渠道不重试，primary 回查也会收敛。

若 EXPIRED/FAILED 充值订单此前已把 limit reservation 置 RELEASED，后来收到 `finality_status=OPEN` 下的权威收款证明，仍必须按原订单价格履约，不能因当前额度已满吞款。履约事务以原 `limit_kind/window_start/amount` 幂等直接增加同一 usage 的 completed，并条件执行 `RELEASED -> RELEASED_THEN_CONSUMED`、写 `late_consumed_at/late_paid_over_limit`；若仍 RESERVED 则正常搬移为 CONSUMED。超过原/当前上限只告警，资产 credit、usage、attempt 纠正和订单 PAID 同事务。对账把 CONSUMED 与 RELEASED_THEN_CONSUMED 都计入 completed，后者不再恢复 reserved。

VIP 本地 `EXPIRED/FAILED` 但 COLLECT `finality_status=OPEN` 时仍保留 `slot_reservation_state=RESERVED`，所以迟付可在原槽内履约。只有 finality sweep 取得不可逆 `CONFIRMED_NOT_PAID` 证据时，结果事务才按 attempt→order→`PrelockUserClosures`→`LockUsersAndValidateClosureGate`→`user_vip_stats(type=125)` 条件终结 attempt/订单，并把仍 RESERVED 的 slot 一次 RELEASED；只有状态命中且计数足够才能提交，重复证据只读原终态。RELEASED 后若渠道违约给出相反成功证据，不得突破 64 上限补建权益，而是创建 `COLLECT_SUCCESS_AFTER_CONFIRMED_NOT_PAID` PAYABLE case。

人工补单调用相同履约函数并使用相同订单级资金幂等键。因此“先人工补单、后迟到真实回调”和相反顺序都只能履约一次。

### 6.3.1 COLLECT 迟付最终性收敛

普通收款 sweep 只处理 `status IN (CREATED,PENDING,UNKNOWN) AND finality_status=OPEN`；独立 finality sweep 处理所有 `kind=COLLECT AND finality_status=OPEN AND finality_next_run_at<=now()`，不受本地订单/attempt 已 FAILED/EXPIRED 影响。短领取事务按 finality 索引 `FOR UPDATE SKIP LOCKED`，用相同谓词条件写 lease/generation 后提交；事务外按稳定商户号执行权威查单/渠道日账核对。有收款结果的事务调用 `AcquireIncomeWriteFences`，按 income root SHARE→finance day fence 后再锁 attempt/order；无收款证据的结果不写流水、可直接从 attempt/order 开始。两者随后锁关联 closure（如有）并重验 owner/generation/lease：

- 有真实收款证据：调用 6.3 同一履约函数，原子写 CONFIRMED_PAID、资产/权益/usage 和 closure 唤醒水位；
- 风险窗已过且渠道提供不可逆“未受理/未收款”证据：条件写 CONFIRMED_NOT_PAID、证据 hash/检查时间，清 lease，把 COLLECT attempt 和仍未支付的父订单同步终结为 FAILED/EXPIRED，并把 closure `next_check_at` 提前；按订单快照锁 usage，将仍 RESERVED 的金额 reservation 一次 RELEASED，VIP 再把仍 RESERVED 的 slot 一次 RELEASED。任何条件未命中或计数不足整体回滚，重复证据只读原终态；
- 仍不确定：保持 OPEN，清 lease、按退避推进 `finality_next_run_at`，不得凭重试耗尽宣告未收款。

worker 停机、Redis 丢失或领取者崩溃后均由索引和过期 lease 追赶；这是一张 payment attempt 自身的可重建状态，不是 Outbox。

### 6.3.2 支付相反终态与 Finance 案件

只有已验签且渠道契约声明为财务最终的事件，或已经封存账单版本/行号/hash 的 reconciliation discrepancy，才能触发相反终态；单个非终局失败通知仍只进入 UNKNOWN/FAIL_CONFIRMING。处理器先无锁判断可能分支，再选择一种锁轨迹：正常履约按本章资金轨迹；相反终态不动钱包，按 provider event/对账项防重 → 涉及的 attempt（rank 100）→ order/withdrawal（rank 300）→ `finance_recovery_case_roots`（rank 400）→ case 加锁。锁内状态若已变化则整体回滚并按新分支重试，禁止在已锁 user/asset 后补 recovery root。事件/对账项处理结果、case、案件审计同一 default 事务提交，原订单、权益、withdrawal、freeze、usage 和流水终态均不回写。

| 矛盾 | source 自然键/component | 方向与价值快照 |
| --- | --- | --- |
| COLLECT 已 CONFIRMED_NOT_PAID 后最终成功 | `PAYMENT_ATTEMPT/<collect_attempt_id>/SUCCESS_AFTER_NOT_PAID` | PAYABLE；充值用 BOTH(COMMON+fiat)，VIP 用 EXTERNAL(fiat) |
| 本地订单已 PAID 后渠道最终证明未收款/撤销 | `PAYMENT_ATTEMPT/<collect_attempt_id>/FAILURE_AFTER_PAID` | RECEIVABLE；充值用 BOTH，VIP 用 EXTERNAL |
| REFUND_CANCELED 后外部退款最终成功 | `PAYMENT_ATTEMPT/<refund_attempt_id>/SUCCESS_AFTER_CANCELED` | RECEIVABLE；充值用 BOTH，VIP 若未来有取消则用 EXTERNAL |
| VIP REFUND_NOT_ACCEPTED 后外部退款最终成功 | `PAYMENT_ATTEMPT/<refund_attempt_id>/SUCCESS_AFTER_NOT_ACCEPTED` | RECEIVABLE；EXTERNAL(fiat)，权益/返佣保持原终态 |
| 本地 REFUNDED 后渠道最终证明退款失败/撤销 | `PAYMENT_ATTEMPT/<refund_attempt_id>/FAILURE_AFTER_REFUNDED` | PAYABLE；充值用 BOTH，VIP 用 EXTERNAL |
| withdrawal FAILED/REJECTED/CANCELED 后 payout 最终成功 | `PAYOUT_ATTEMPT/<attempt_id>/SUCCESS_AFTER_LOCAL_FAILURE` | RECEIVABLE；BOTH(requested COMMON + payout fiat) |
| withdrawal ARRIVED 后 payout 最终失败/撤销 | `PAYOUT_ATTEMPT/<successful_attempt_id>/FAILURE_AFTER_ARRIVED` | PAYABLE；BOTH |
| 切换后额外 attempt 也真实成功 | `PAYOUT_ATTEMPT/<extra_success_attempt_id>/DUPLICATE_SUCCESS` | RECEIVABLE；每个额外成功 attempt 单独一案，BOTH |
| duplicate 不再成立，但其 RECEIVABLE 已外部收回 | `FINANCE_CASE/<duplicate_case_id>/SOURCE_REVERSAL` | PAYABLE；`DUPLICATE_PAYOUT_REVERSED_AFTER_RECOVERY`，BOTH |
| 本地释放后迟到成功的 RECEIVABLE 已外部收回，随后该 payout 成功又被权威反转 | `FINANCE_CASE/<late_success_case_id>/SOURCE_REVERSAL` | PAYABLE；`PAYOUT_LATE_SUCCESS_REVERSED_AFTER_RECOVERY`，BOTH |
| 无成功 payout 的 PAYABLE 已外部补付，之后又出现真实成功 | `FINANCE_CASE/<payout_failure_case_id>/SOURCE_REVERSAL` | RECEIVABLE；`PAYOUT_SUCCESS_AFTER_FAILURE_COMPENSATED`，BOTH |

reason registry 把上述 component 分别固定映射到 `COLLECT_SUCCESS_AFTER_CONFIRMED_NOT_PAID`、`COLLECT_FAILURE_AFTER_LOCAL_PAID`、`REFUND_SUCCESS_AFTER_LOCAL_CANCELED`、`VIP_REFUND_SUCCESS_AFTER_NOT_ACCEPTED`、`REFUND_FAILURE_AFTER_LOCAL_REFUNDED`、`PAYOUT_SUCCESS_AFTER_LOCAL_FAILED`、`PAYOUT_FAILURE_AFTER_LOCAL_ARRIVED`、`DUPLICATE_PAYOUT_AFTER_CHANNEL_SWITCH`、`DUPLICATE_PAYOUT_REVERSED_AFTER_RECOVERY`、`PAYOUT_LATE_SUCCESS_REVERSED_AFTER_RECOVERY` 和 `PAYOUT_SUCCESS_AFTER_FAILURE_COMPENSATED`。业务金额、外币金额/币种一律从不可变 order/withdrawal/attempt/原 case 快照复制，客户端或对账操作员不可填写；同一 evidence 重放命中原 disposition/case。后三类只能由 `finance_case_source_reversals` 自动产生：未结原案必须取消，已 WAIVED 原案无反向金额，只有已 RESOLVED 原案才创建等额反向案。

### 6.4 VIP 激活、到期与退款

- worker 与所有强一致 VIP 读取共用 `EnsureVipEntitlementCurrent(tx,gate_token,user_id,now)`：调用方必须先 `PrelockUserClosures`，再用 gate token 调用 `LockUsersAndValidateClosureGate`，按 `users -> user_vip_stats(type=125) -> vip_entitlements(type=130)` 锁定。只通过 `(user_id,status,starts_at,id)` 索引读取 ACTIVE/QUEUED，且锁内先断言集合数等于 `entitlement_slots`、总占用不超过 64。循环最多 64 次：每个 `ends_at<=now` 的未终结权益转 EXPIRED 并令 `entitlement_slots -= 1`，再把 `starts_at<=now<ends_at` 的首条 QUEUED 转 ACTIVE，由 partial unique 验证最多一条；更新、计数、必要 schedule audit 和 closure `next_check_at` 唤醒同事务。分钟作业每用户一个短事务调用该原语，绝不扫描 EXPIRED/REVOKED 历史。
- 播放鉴权先在 primary 短事务调用 Ensure，再读取 ACTIVE 并比较 `[starts_at,ends_at)`；worker 停摆跨越一个或多个队列边界时，请求仍同步收敛后正确授权，不能只信旧 status 或缓存。
- 2.0 首发只做全额退款。退款先在本地建立可恢复退款状态，外部退款调用在事务外。充值退款需要在 provider/command 防重后先取得 income root SHARE 和 finance day fence，再以 `deposit:refund:freeze:<order_no>` 冻结原 `common_amount`。可用额不足时返回稳定 `REFUND_BALANCE_INSUFFICIENT`，订单保持 PAID，不创建 freeze、REFUND attempt、Finance case 或任何“人工债务”状态；管理员只能在余额恢复后重放原语义请求。若未来要在余额不足时仍发起外部退款，须另立债务模型 ADR，首发不得用口头流程代替 Schema。
- 渠道确认充值退款后，用 `deposit:refund:settle:<order_no>` 调用 `SettleFreeze` 并转 REFUNDED；原 credit 永不删除。VIP 退款确认在取得任何用户锁前，先锁退款 attempt/订单和每条原返佣 GRANT，再为每条潜在冲正按自然键 Ensure/锁定 `finance_recovery_case_roots`，然后对购买者及全部原受益人执行 `PrelockUserClosures`，继而按 ID 锁仍 ACTIVE 的 freeze，最后调用 `LockUsersAndValidateClosureGate`；禁止在余额判断后反向补锁 recovery root。helper 返回逐用户裁决，不能因某个第三方已 CLOSED 而让付款人退款整体失败。购买者进入用户层后先锁 `user_vip_stats(type=125)`，再把目标来源 entitlement 与当时最多 64 条 ACTIVE/QUEUED 行合并、去重并按 type130/ID 锁齐，然后在同一数据库 `now` 下执行 `EnsureVipEntitlementCurrent`：先连续过期所有已结束区间并激活正确队首，再裁决退款。目标此时 ACTIVE/QUEUED 才转 REVOKED、`entitlement_slots -= 1` 并保存 `refund_entitlement_disposition=REVOKED`；目标已经 EXPIRED 则保持历史区间/计数不变并保存 ALREADY_EXPIRED。只有 REVOKED 分支压缩目标之后最多 63 条 QUEUED，ACTIVE 被撤销以 now、QUEUED 被撤销以直接前驱 ends_at（无前驱则 now）为起点，依原支付时间/ID连续重排；任何分支都不能把已过期时长延长。每条原 GRANT 仍创建唯一 REVERSAL：受益人仍有冻结时调用 `SettleFreeze`、把 GRANT 转 CANCELED 并把 REVERSAL 终态置 CANCELED_FROZEN；GRANT 已 RELEASED、钱包允许且锁内可用额充足时全额受控扣回，REVERSAL 终态 POSTED；可用额不足或受益人已 CLOSED 时不得部分扣回，分别以 `INSUFFICIENT_AVAILABLE/CLOSED_ACCOUNT` 在已预锁 root 下创建/命中唯一 `finance_recovery_cases`，REVERSAL 提交终态 EXTERNAL_RECOVERY 且原 GRANT 保持 RELEASED。订单、退款 attempt、stats 守恒、closure 唤醒、权益队列/审计、返佣结果和适用资产事实同成败；第三方追偿永远不是付款人 REFUNDED 的 veto，案件后来 RESOLVED 或 RECEIVABLE 的 WAIVED 也不改写返佣或钱包，PAYABLE waive 必须拒绝。
- 充值退款外部结果未知或临时失败时保持 REFUND_PENDING，复用唯一 attempt 与商户号回查。只有渠道明确未受理且无未知请求，`CancelDepositRefund` 才能在一个 primary 事务按全局顺序先锁退款 attempt、再锁订单、`PrelockUserClosures`、freeze、`LockUsersAndValidateClosureGate` 和资产，调用 `ReleaseFreeze`（键 `deposit:refund:release:<order_no>`）并转 REFUND_CANCELED/唤醒 closure；取消是不可逆终态，不允许再退款。取消后迟到成功的事件事务建立 `REFUND_SUCCESS_AFTER_LOCAL_CANCELED` RECEIVABLE case；相反地，本地已 REFUNDED/冻结 SETTLED 后渠道最终失败则建立 `REFUND_FAILURE_AFTER_LOCAL_REFUNDED` PAYABLE case。两者都保留原订单/freeze/usage终态并 P0 告警，不重新释放/扣除钱包。
- VIP 退款未知、临时失败或结果仍可逆时保留订单 REFUND_PENDING 和同一个 REFUND attempt；受控重试只能复用 `:refund:1`。若稳定商户号查单给出不可逆 NOT_ACCEPTED 且不存在成功/UNKNOWN 外部请求，结果事务按 event/attempt→order 加锁，把 attempt 与订单条件终结为 `REFUND_NOT_ACCEPTED`，保存证据/原因并唤醒 closure；不改 entitlement、返佣、任务或资产。之后迟到退款成功不回写订单或权益，而按 §6.3.2 新增 `VIP_REFUND_SUCCESS_AFTER_NOT_ACCEPTED`、EXTERNAL/RECEIVABLE 案件。首发没有换商户号再退、回到 PENDING 或口头“人工差异”旁路；不能稳定查单并给出该最终性的 provider 不得发布 VIP 退款能力。
- `PURCHASE_ONE_TIME_VIP` 任务记录是当时真实购买事实，退款不倒退已领取奖励；如业务未来要求追偿，必须另立任务奖励冲正 ADR，不能偷偷修改进度。

### 6.5 创建提现

校验顺序固定为：

```text
身份/账号状态
-> Authenticator/TOTP（ACTIVE 必验，NEVER_BOUND 可跳过，历史 reset/revoke 须先恢复）
-> KYC APPROVED
-> 风控要求
-> 收款账户属于本人且核名通过
-> 金额解析成功且 > 0
-> 精度、单笔、日/周限额
-> 服务端手续费/汇率/净额
-> 渠道法币限额
-> COMMON 可用余额
```

普通 `CreateWithdrawal` 只允许 ACTIVE 用户及 `purpose=USER_REQUEST`。`CreateClosureWithdrawal` 只允许 CLOSING 用户、当前未完成 closure、没有同 closure 非终态清退单、VERIFIED 清退银行卡和 `purpose=ACCOUNT_CLOSURE`；该卡可以是进入 CLOSING 前已有的，也可以是原卡失效后通过 closure-purpose 窄补卡流程核验的。申请额由事务锁内按上式计算本批金额；大额余额按 `closure_withdrawal_seq` 串行分批，前一笔终态后才可创建下一笔。若当前 DAY/WEEK 剩余额度为 0，稳定返回 `CLOSURE_WITHDRAWAL_WINDOW_EXHAUSTED` 和下个窗口，不创建单；若仍有余额，则只有锁内复验当前 ACCOUNT_CLOSURE 规则版本完整且最多 32 个 channel slot、对应渠道状态与集合 hash 后，所有槽都连一个满足下限/最小单位的正批次也无法支付，才进入 04 章 dust-to-PAYABLE 路径。手续费/最低额使用已发布 `account_closure_rule_settings`。普通和 closure withdrawal 使用完全相同的 Authenticator/TOTP 规则；CLOSING 不构成降级理由。

这里的 OTP 只指 `otp_bindings` 中的 Authenticator/TOTP，不是 SMS 验证码：

- `NEVER_BOUND`：该用户从未成功确认此类 binding；仅有未确认的 PENDING/EXPIRED 记录仍属于此分支，可不提交 `otp_code`；
- `ACTIVE`：请求体必须提交 `otp_code`，事务内验证并消费一个严格大于 `last_accepted_step` 的 step；
- `REBIND_REQUIRED`：至少一条历史 binding 曾确认但当前没有 ACTIVE，必须重新绑定或完成受控恢复，不能按 NEVER_BOUND 放行；管理员 reset 是带 `ADMIN_RESET` 原因的 REVOKED。

`otp_code` 是瞬时 secret，不进入 `command_deduplications.request_hash`、提现表、审计正文或日志；request hash 只覆盖提现业务语义。已提交成功的同键同 hash 重放必须先返回原提现结果，不再次要求或消费新的 OTP。

随后在一个 primary 事务中：

1. 先锁命令防重行并核对业务 request hash；若该命令已有成功结果，立即返回原 withdrawal，不进入后续 OTP 分支。首次命令才调用 `AcquireIncomeWriteFences` 取得 income root SHARE 和 finance day fence，随后执行 `PrelockUserClosures`（ACCOUNT_CLOSURE 必须命中请求中的当前 closure，普通提现必须仍无 closure）。
2. 调用 `LockUsersAndValidateClosureGate` 锁 user 并裁决 purpose/status；在同一 user 锁保护下查询 binding 状态。ACTIVE 时按 `UserStateLockKey` 锁定该 `otp_bindings` 行，使用请求中的 `otp_code` 验证候选 step，并执行 `UPDATE ... WHERE status='ACTIVE' AND last_accepted_step < :step` 原子推进；影响 0 行即按重放/状态冲突拒绝。无 ACTIVE 时用索引化 EXISTS 区分 NEVER_BOUND 与 REBIND_REQUIRED。管理员 reset/revoke 也必须先锁同一 user，不能与提现形成 binding-first 反向路径。
3. OTP 裁决后严格按 `UserStateLockKey` 锁 current/candidate KYC（按 ID）、目标银行卡、风控聚合与 `user_limit_usages` 的 DAY/WEEK 两行；不能按业务校验文案顺序形成 limit→bank、case→user 或 binding→user 反向锁。按已发布 WITHDRAWAL settings/rate 重算所有资格与金额，以两个条件更新分别令 `reserved_amount += requested_common`，任一不满足上限则整体回滚，再锁 `(user_id,COMMON)` 资产账户。
4. ACCOUNT_CLOSURE 在已锁 closure 行上分配下一个连续 `closure_withdrawal_seq`，并用 partial unique `(closure_id) WHERE purpose='ACCOUNT_CLOSURE' AND status IN ('AWAIT_REVIEW','APPROVED','PAYING','FAIL_CONFIRMING')` 保证至多一笔非终态；插入 `withdrawals`，保存 `purpose/closure_id/closure_withdrawal_seq`、`otp_mode` 与条件 `otp_binding_id/otp_accepted_step` 审计快照、银行卡、汇率、手续费、限额、时区和规则快照，并插入恰好两条 `withdrawal_limit_reservations(RESERVED)`；调用 `OpenFreeze(FREEZE, COMMON, requested_common)`，引用提现单，幂等键 `withdraw:freeze:<withdrawal_no>`。
5. 回填唯一 `freeze_id`，写 `AWAIT_REVIEW` 和审计后提交。OTP step 推进、限额预占、提现单、冻结、流水、命令结果任一步失败都整体回滚。

任一步失败都没有提现单、sequence/OTP step 消费或冻结残留。客户端传入的预计到账额只用于差异提示，服务端计算值才是真值。CLOSING 用户可在前一笔终态后继续发起下一 closure withdrawal，清退剩余或既有持仓后来产生的 COMMON；每批都需要新的 quote，且当时仍为 ACTIVE 的 Authenticator/TOTP 每批都必须提交新 step。普通充值、Playlet Backing、Robot Purchase 及 USER_REQUEST 提现保持关闭。

### 6.6 审核与出款

审核通过不写资产，可直接锁 `withdrawals`。驳回和用户取消会释放冻结及提现限额 reservation，必须先锁命令防重、取得 income root SHARE 与 finance day fence，再锁 withdrawal 及其 DAY/WEEK 两条 reservation，执行 `PrelockUserClosures`、锁既有 freeze、调用 `LockUsersAndValidateClosureGate`，随后按快照锁两条 `user_limit_usages`，最后才进入资产；CLOSING 对 cutoff 前提现终结放行并唤醒 closure。随后追加 `withdrawal_reviews` 并同步当前决定摘要：

- 驳回：条件执行 `AWAIT_REVIEW -> REJECTED`，同时把两条 reservation 置 RELEASED、两个 usage 各减相同 reserved amount并 `ReleaseFreeze`，保存审核人、原因、IP 和批次。
- 通过：条件执行 `AWAIT_REVIEW -> APPROVED`，保存审核事实并把 `next_run_at=now()`；事务中不调用渠道。

用户取消使用同一锁顺序，条件执行 `AWAIT_REVIEW -> CANCELED`、释放 DAY/WEEK 两条 usage reservation并 `ReleaseFreeze`；与审核并发时只能一方成功。withdrawal 条件命中一行、两条 reservation 都命中 RESERVED 后才允许提交，任何 APPROVED/PAYING/终态提现都拒绝取消。

primary sweep 的短领取事务只触碰 `withdrawals`：候选必须满足 `status='APPROVED' AND next_run_at <= now() AND (dispatch_lease_until IS NULL OR dispatch_lease_until <= now()) FOR UPDATE SKIP LOCKED`；条件 UPDATE 重复同一状态、时间和租约谓词，写唯一 owner/until、令 `dispatch_lease_generation=dispatch_lease_generation+1` 并返回 owner/generation，影响 0 行即跳过，随后提交。绝不在已锁 withdrawal 时再创建 attempt。

执行事务根据稳定 `(withdrawal_id,attempt_no=1)` 先 `INSERT ... ON CONFLICT`/锁 `payout_attempts`，再锁 withdrawal，要求其仍为 APPROVED、`current_attempt_no=0`、owner/generation 与领取令牌完全一致且 lease 未过期，并重验活动 attempt 唯一约束；成功才把提现置 PAYING、`current_attempt_no=1`、清空 dispatch lease、推进 `next_run_at` 并提交。验证失败整事务回滚，不留下孤立 attempt；旧 owner/旧代次即使迟到也不能创建、提交或推进 attempt。之后才调用渠道，响应由新事务按 `event -> attempt -> withdrawal` 写回。创建请求始终使用稳定商户单号，网络超时进入 UNKNOWN，不创建另一笔随机请求。

`SwitchPayoutChannel` 是后续 attempt 的唯一入口。正式事务按命令防重→旧 payout attempt（rank100/type20）→异人审批（rank100/type30）→withdrawal 加锁，要求 withdrawal 仍 PAYING/FAIL_CONFIRMING、旧 attempt 恰为 `current_attempt_no` 且状态 NOT_ACCEPTED、权威 evidence/hash 匹配、`attempt_no<PAYOUT_ATTEMPT_MAX`、目标渠道版本当前允许该银行/币种/金额，并且没有其他活动、成功 attempt 或 failure resolution；随后插入 `payout_channel_switches` 和 `attempt_no+1` 的 CREATED attempt，把 withdrawal 保持/恢复 PAYING、推进 current attempt/channel switch count/next run。审批消费、指针、新 attempt 和审计同成败；事务中不调用渠道、不释放 freeze、不改变 usage。UNKNOWN、普通 FAILED、超时、仅回调声称失败或没有最终未受理证据时一律不能切换。旧渠道违约后又成功时按下述“首个成功/额外成功”集合裁决，不能仅因曾切换就先立应收。

`FinalizeNotAcceptedPayoutFailure` 与切换竞争同一旧 attempt/withdrawal。它先取得命令防重、income root SHARE/day fence，再锁当前 NOT_ACCEPTED attempt、必要异人审批、withdrawal、DAY/WEEK 两条 reservation，执行 closure/freeze/user/usage/asset 固定锁序；锁内再次证明无成功/活动 attempt、未存在 switch/resolution，且满足“无当前可用匹配渠道、已达 8 次上限或已批准 OPERATOR_ABORT”之一。随后插入唯一 `payout_failure_resolutions`，条件把 withdrawal 转 FAILED，把两条 reservation 置 RELEASED、两个 usage 各减 reserved，并以 `withdraw:release:<withdrawal_no>` 调用 `ReleaseFreeze`。决策、失败终态、两窗口和冻结任一写失败全部回滚；因此唯一渠道、全部候选下架或 attempt 用尽时都能收敛，不能永久占用余额或阻塞注销。

### 6.7 出款回调、乱序与重放

出款标准事件在事务外验签并无锁预判分支。仍可正常终结 ACTIVE freeze 的结果，才在 primary 事务按 `payout_provider_event -> rollup_roots('income') SHARE -> finance_business_days SHARE -> 全部相关 payout_attempt（按 ID） -> withdrawal -> PrelockUserClosures -> asset_freeze -> LockUsersAndValidateClosureGate -> user_limit_usages -> user_asset` 顺序处理，并在 gate 内把 cutoff 前提现视为既有来源终结、同步唤醒 closure。相反终态则改走 §6.3.2 的无钱包轨迹，在 user/asset 前锁 recovery root；纯 UNKNOWN 状态事件跳过 income root/day 和 closure/freeze/asset。按不变量 CLOSED 不应仍有 ACTIVE 提现冻结；若出现则事务失败关闭并 P0 告警，禁止静默重开钱包：

- 首次成功：锁内把当前 attempt 条件推进为操作 `status=SUCCEEDED`、`financial_status=SUCCESS_CONFIRMED`，清空 `next_run_at/work_lease_owner/work_lease_until`，随后调用下述 `ReconcilePayoutSuccessSet`。若 withdrawal 尚未 ARRIVED，这是第一个真实成功：从 PAYING/FAIL_CONFIRMING 条件转 ARRIVED，在同一事务把 DAY/WEEK 两条 reservation 从 RESERVED 转 CONSUMED、对应两个 usage 各做 reserved→completed，并执行 `SettleFreeze`；其他尚未提交 attempt 转 STOPPED_BEFORE_SUBMIT，已经可能到达渠道的 attempt 转 STOP_CONFIRMING、禁止再次提交且只安排查单。冻结与 usage 此后永不重开。
- 可确认失败：当前 attempt 的普通最终失败从 PAYING/FAIL_CONFIRMING 条件转 FAILED，在同一事务把 DAY/WEEK 两条 reservation都从 RESERVED 转 RELEASED、对应两个 usage 各减 reserved，并执行 `ReleaseFreeze`，键为 `withdraw:release:<withdrawal_no>`。NOT_ACCEPTED 不由普通回调直接释放，只能由 `SwitchPayoutChannel` 或 `FinalizeNotAcceptedPayoutFailure` 互斥裁决。
- 不确定或单个失败通知：进入/保持 FAIL_CONFIRMING，冻结仍 ACTIVE，安排 primary 回查。
- 同终态重放：验证金额、渠道单号和 payload 一致后 ACK 原结果，不再变更冻结。
- STOP_CONFIRMING 查单：权威未受理/失败转 STOPPED，无资产效果；最终成功则把该 attempt 原子转 `status=SUCCEEDED/financial_status=SUCCESS_CONFIRMED`、清 work lease/next run 并调用同一集合原语。已 FAILED/REJECTED/CANCELED 且 freeze RELEASED 后出现真实 payout success 也走该原语的本地释放终态分支，创建 `PAYOUT_SUCCESS_AFTER_LOCAL_FAILED` RECEIVABLE，绝不重开钱包。
- 成功反转：只有同商户号、金额、币种一致的权威 reversal event 或封存对账证据，才能把 `financial_status=SUCCESS_CONFIRMED -> SUCCESS_REVERSED`；操作 `status` 保持历史 SUCCEEDED，普通失败通知只进入差异项。反转事务调用同一集合原语，不能直接按“withdrawal 已 ARRIVED”无条件创建 PAYABLE。CHECK 强制 SUCCESS_CONFIRMED/SUCCESS_REVERSED 只能配 SUCCEEDED、无 next run/活动 work lease，活动操作状态只能配 NO_SUCCESS。

`ReconcilePayoutSuccessSet(tx,withdrawal_id,trigger_evidence)` 是唯一 canonical/duplicate 收敛入口：

1. 事务外从 primary 无锁预收集该单最多 8 个 attempt、现有 disposition 及所有可能的 Finance root 自然键。正式事务按全局单调顺序锁 event/reconciliation → 全部 attempt（ID 升序）→ withdrawal → 全部 recovery root（规范键升序）→ case/disposition；不得 root→attempt 反锁。锁内集合或根映射变化时整体回滚、扩集重试。随后重新计算 `active_successes = financial_status=SUCCESS_CONFIRMED`，canonical 候选固定为最小 `attempt_no`，并以规范有序 ID 计算 set hash；调用方传来的“是否首个/额外”布尔值不可信。
2. 仅当 `withdrawal.status=ARRIVED AND active_successes` 非空时，`settled_by_attempt_id` 才指向 canonical。canonical disposition 置 CANONICAL；其他 active 行置 DUPLICATE，并逐 attempt 创建/命中 `DUPLICATE_PAYOUT_AFTER_CHANNEL_SWITCH` RECEIVABLE。正常本地首成功必须先在同一事务把 withdrawal 推进 ARRIVED，再进入本分支。此前 duplicate 现已成为 canonical，或此前 duplicate attempt 现已 REVERSED 时，其旧 RECEIVABLE 差异已经消失，必须调用 `ReverseFinanceCaseSource`：OPEN/IN_REVIEW 原案转 CANCELED，WAIVED 只记无金额结果，RESOLVED 原案生成等额 `DUPLICATE_PAYOUT_REVERSED_AFTER_RECOVERY` PAYABLE。其他仍为 duplicate 的案件不动。
3. `active_successes` 为空且 withdrawal 已 ARRIVED 时，清空 `settled_by_attempt_id`，并按本次最后被反转的 canonical attempt 建/命中 `PAYOUT_FAILURE_AFTER_LOCAL_ARRIVED` PAYABLE；不会为每个历史 attempt 重复建债务。若之后另一个 STOP_CONFIRMING attempt 迟到成功，重新选它为 canonical，并对这笔 coverage PAYABLE 调用相同 source-reversal 规则：OPEN/IN_REVIEW 取消，已 RESOLVED 则生成等额 `PAYOUT_SUCCESS_AFTER_FAILURE_COMPENSATED` RECEIVABLE。
4. withdrawal 已 FAILED/REJECTED/CANCELED 时不可能有 canonical coverage：每个 active success 都写 LATE_AFTER_LOCAL_FAILURE 并各自创建 `PAYOUT_SUCCESS_AFTER_LOCAL_FAILED` RECEIVABLE，`settled_by_attempt_id` 保持空且不触碰 freeze/usage。该 attempt 后来反转时，原案 OPEN/IN_REVIEW 转 CANCELED_BY_SOURCE_REVERSAL，WAIVED 记录 `NO_EFFECT_WAIVED`；原案已 RESOLVED 则创建唯一 `PAYOUT_LATE_SUCCESS_REVERSED_AFTER_RECOVERY` PAYABLE，同样不能留下旧应收。
5. 对每个曾成功但已反转 attempt 写 REVERSED current disposition；所有 current disposition、追加 event、withdrawal generation/canonical、原案取消或反向案、provider event 处理结果和审计同一 default 事务。延迟约束在提交时按 withdrawal 终态验证：ARRIVED 且 active 非空时恰一 CANONICAL、其余 active 恰为 DUPLICATE；本地释放终态的 active 全为 LATE_AFTER_LOCAL_FAILURE；active 为空无 CANONICAL；每个未处置差异都有且只有一案。任一步失败全部回滚。

因此 A1 首成、A2 重复成功、A2 反转会取消 A2 的未结应收；A1 反转但 A2 仍成功会把 A2 提升为 canonical，而不是新增未到账应付。若相应应收已实际收回，才以反向 PAYABLE 表达新的真实债务。原 withdrawal/freeze/usage 终态始终保持不变，不自动追扣或补钱包。

任何回调都不能直接用请求中的用户 ID 定位资产；必须从唯一渠道/商户单映射回本地提现和 `user_id`。

### 6.8 渠道对账

每个渠道至少执行近实时查单和日终账单对账，比较：

```text
渠道 collect/refund/payout
<-> payment_attempts / payout_attempts
<-> deposit_orders / vip_orders / withdrawals
<-> asset_ledgers / asset_freezes / vip_entitlements
```

自动可修复场景调用同一履约器，例如“渠道已收款、本地仍 PENDING”或“渠道出款成功、本地仍 PAYING”。对已处相反本地终态且账单证据达到财务最终性的行，按 §6.3.2 在同一事务封存 reconciliation item 并创建/命中类型化 PAYABLE/RECEIVABLE case，同时 P0 告警；普通金额/币种不等但尚不能确定债权债务的项保持 OPEN discrepancy，不得猜方向。每次对账保存批次、渠道账单版本、行号/游标、证据 hash、差异类型、case 引用、处理动作和处理人，不自动反向扣款。

## 7. 事务边界与可靠执行

### 7.1 允许进入事务的本地事实

- 订单/尝试状态、业务快照、VIP 权益、最多两层冻结返佣、一次性 VIP 任务进度、提现 Authenticator/TOTP step 消费、冻结、资产流水、回调事件和操作审计。
- 同一顶层 Use Case 只在 `database.default` 开一次事务，内部方法只接收 `Tx`。
- 余额、幂等、状态判断、回调履约、作业认领和读己之写均在 primary；slave 只服务允许延迟的历史列表和统计。

### 7.2 永远不进入事务的外部调用

- 创建/查询/关闭支付意图，Apple/Google 凭证在线验证。
- 发起/查询/取消退款。
- 发起/查询出款、银行卡或钱包渠道探测。
- KYC/核名的实时供应商调用。
- Redis 发布、Forge `message` 的 SMS/Email 发送和运营通知；WhatsApp 完全排除。Forge 发送成功只表示 `PROVIDER_ACCEPTED`，无可靠证据的不确定调用转 `UNKNOWN` 且不自动重发；challenge 状态、lease、sweep、重试与一次消费仍由业务数据库负责。

外部调用采用“先提交可恢复意图，再调用，再开新事务收敛结果”。不得打开事务、锁住订单/余额后等待渠道 HTTP。

### 7.3 `status + next_run_at` primary sweep

所有可恢复外部工作在其事实表保存稳定状态、`next_run_at`、`attempt_count`、`last_error_code` 和脱敏错误摘要。worker 的统一过程是：

1. 在 primary 开短事务，按 `(next_run_at,id)` 查询到期行并 `FOR UPDATE SKIP LOCKED`。
2. 把 `next_run_at` 推到租约截止、增加尝试计数或写认领状态，提交。
3. 在事务外调用渠道；超时保留 UNKNOWN/进行中语义。
4. 新开 primary 事务，按允许边和幂等键写响应；失败按有上限的指数退避设置下一次时间。
5. worker 崩溃时租约到期后再次被扫到；恢复先查渠道，不盲目重发资金命令。

需要扫描的核心索引是 `payment_attempts(status,next_run_at,id)`、`withdrawals(status,next_run_at,id)` 和 `payout_attempts(status,next_run_at,id)` 的部分索引。作业禁止在 slave 扫描，否则复制延迟会漏认领或重复认领。

2.0 不创建 `outbox`、`events_to_publish` 或通用消息重试表。进程内通知/Redis 可以在提交后唤醒 worker，但丢失通知不会丢工作。只有未来出现“数据库状态无法重建且外部命令必须至少一次送达”的具体需求，才允许单独 ADR 评估窄用途 Outbox。

## 8. 幂等键

| 场景 | 幂等键/唯一性 |
| --- | --- |
| 用户创建充值/VIP/提现 | 请求头 `Idempotency-Key`，严格复用 `(actor_type,actor_scope,rpc_name,key)`；业务单唯一引用对应 `command_deduplications.id` 并保存 `request_hash` |
| 渠道收款创建 | 每单唯一稳定 `merchant_request_no=<order_no>:collect:1`；所有回查/纠正复用同一 attempt |
| 渠道退款创建 | 每单唯一稳定 `merchant_request_no=<order_no>:refund:1`；充值/VIP 所有重试复用同一行和商户号 |
| 渠道出款创建 | 稳定 `merchant_request_no=<withdrawal_no>:payout:<attempt_no>` |
| provider 回调 | 渠道 event ID；没有时用契约稳定字段构成 `callback_key` |
| 充值入账 | `deposit:credit:<order_no>` |
| 充值退款冻结/结算/取消释放 | `deposit:refund:freeze:<order_no>` / `deposit:refund:settle:<order_no>` / `deposit:refund:release:<order_no>` |
| VIP 履约 | `vip:fulfill:<order_no>`；各受益人返佣另带 `beneficiary_user_id` |
| VIP 返佣冲正 | 每个原 GRANT 使用 `reversal:<grant_id>` |
| Finance 追偿管理 | 公共管理命令作用域 `Idempotency-Key` + case `expected_version`；同键同语义返回原状态，仅 waive 唯一消费匹配 approval |
| 提现 Authenticator/TOTP | 命令成功结果优先；ACTIVE binding 以 `last_accepted_step` 原子防重，NEVER_BOUND 可跳过，历史 reset/revoke 必须先恢复；`otp_code` 不进入 request hash |
| 提现冻结 | `withdraw:freeze:<withdrawal_no>` |
| 提现失败/拒绝释放 | `withdraw:release:<withdrawal_no>` |
| 用户取消提现 | `withdraw:cancel:<withdrawal_no>`；冻结终结仍使用唯一 release 语义 |
| 提现到账结算 | `withdraw:settle:<withdrawal_no>` |
| 注销清退提现 | closure + command key；仍为独立 withdrawal_no，purpose 与服务端 quote 必须一致，并执行同一 TOTP 规则 |
| 人工补单 | 仍使用原订单履约键，不创建 `manual:*` 第二笔资金键 |

相同键只有在规范化请求 hash 完全相同时才返回原结果；同键不同金额、币种、用户、产品、银行卡或目标状态必须返回幂等冲突并告警。换一个 UUID 重试不是容错方案。

## 9. 并发与锁顺序

### 9.1 全局顺序

回调/作业/人工操作使用相同顺序：

```text
command_deduplications / payment_provider_events / payout_provider_events（涉及者）
  -> rollup_roots('income') FOR SHARE（本次会产生资产流水时）
  -> finance_business_days FOR SHARE（同上）
  -> payment_attempts / payout_attempts（按 id）
  -> deposit_orders / vip_orders / withdrawals → account_closures（只取涉及者，各类按 id）
  -> asset_freezes（仅终结既有提现/退款/返佣冻结时，按 id）
  -> users、otp_bindings（仅创建提现且用户 ACTIVE binding）、KYC、银行卡、关系(type 90)、等级(type 100)、限额(type 110)、任务(type 120)、user_vip_stats(type 125)、vip_entitlements(type 130)（一次收齐并严格按 01 UserStateLockKey）
  -> user_assets（按 user_id, asset_type）
  -> 流水、权益、返佣、任务和审计 INSERT
```

- 禁止先锁资产再回头锁订单/提现；外部调用前必须提交并释放全部锁。
- 创建充值/VIP 虽不写资产，也必须在插入新订单/attempt 前先预锁 closure、再锁 users 并重验 ACTIVE；VIP 还必须锁 `user_vip_stats(type=125)`、有界收敛权益后预占一个 slot，限额行位于 user 之后。注销请求同锁串行，禁止 usage/stats→user 或“先插 COLLECT 再查状态/容量”。
- 同一订单多回调由业务单行锁和条件状态更新串行；唯一索引处理两个事务同时首次到达的窗口。
- VIP 履约在订单后先从 primary 只预读来源关系/版本和受益人、等级、任务聚合 ID；预读不锁权益也不携带裁决。锁完全部 users 后按 type 依次取得关系/等级/任务、购买者 `user_vip_stats(125)`，再在 stats 锁内从 primary 重读、核对并锁齐最多 64 条未终结权益(130)。锁内重验关系版本和 `RESERVED -> CONSUMED` 守恒后才排序队列，所有裁决行都先于任一受益人资产锁。
- 有 payout event/attempt 的 sweep 执行、回调和人工处理先锁 event/attempt，再统一锁 `withdrawals`，随后才进入 freeze；纯审核/取消没有 attempt，可从 withdrawal 开始。短领取事务只触碰 withdrawal 并立即提交，不与执行事务交叉持锁。各路径不能定义相反顺序。
- 多受益人返佣最终交给 `ApplyMutations` 按 `(user_id,asset_type)` 排序。
- Finance case 管理先用 `case_no` 在 primary 无锁定位 recovery root，正式按命令防重 → approval（仅 waive）→ recovery root → case → 审计执行并重验自然键。原订单、GRANT 和 REVERSAL 已不可变，管理命令不得加锁或 UPDATE，也不得取得 closure/user/asset 锁，因此不会形成 case→来源反向边。
- 禁止在锁住任一 `user_assets` 后回头锁 closure、订单、freeze、用户、OTP binding、KYC、银行卡、关系、资格、等级或任务行；正反方向并发测试必须覆盖 VIP 履约/退款、提现创建/审核/终态和注销清退。

### 9.2 外部重复命令控制

- 同一提现只允许一个活动 payout attempt 的部分唯一索引，防止并发 worker 双出款。
- attempt 在网络调用前以 CREATED 提交；worker 在创建提交后崩溃或租约过期时必须重领并复用原 attempt/`merchant_request_no`，绝不因“尚未 SUBMITTING”再建一行。
- 渠道创建超时后先按商户号查单；渠道不支持查询或幂等创建时，该渠道不得开启自动重试，只能进入人工复核。
- 作业租约只防止正常情况下的并发认领，资金唯一性仍由数据库唯一键、渠道商户号和本地状态机共同保证。

## 10. 失败模式与处理

| 失败模式 | 必须行为 |
| --- | --- |
| 创建渠道支付/出款超时 | 标为 UNKNOWN/在途并安排回查；不猜失败，不换商户号重发 |
| 渠道明确拒绝创建且确认未受理 | 收款 attempt 可 FAILED；出款经确认后才 FAILED 并释放冻结 |
| 验签失败/未知渠道 | 不查用户、不动订单和资金；记录受限安全指标并拒绝 |
| 已验签但金额、币种、商户不符 | 保存 REJECTED provider event 和 P0 告警；不履约 |
| 同一 psp_ref 指向两张单 | 唯一索引拒绝，事务回滚并告警潜在欺诈/适配错误 |
| 回调先于创建响应 | 用稳定商户单号定位，成功事务可先回填 psp_ref；迟到创建响应只能读回终态 |
| 本地过期后回调成功 | 真实已收款仍按原快照履约并标晚到；不能因本地时间丢用户资金 |
| 履约事务失败 | 订单不置 PAID，资产/权益不残留；回调或查单用同键重试 |
| COMMIT 响应丢失 | primary 按 callback/order/ledger 幂等键确认，不创建新单 |
| 人工补单与真实回调竞态 | 共用订单级履约键，只有一个事务成功，另一个返回原结果 |
| VIP 并发支付 | `user_vip_stats(type=125)` 容量根串行 + 权益唯一/区间约束，每次只排序最多 64 条并生成稳定无重叠队列 |
| VIP 槽位耗尽/重复释放 | 建单在 stats 根条件预占，支付只把 RESERVED 原子换成 entitlement，CONFIRMED_NOT_PAID 才释放；同一订单条件状态与计数守恒使重放不重复加减，达到 64 时不创建外部 COLLECT |
| VIP 返佣释放与退款并发 | 原 GRANT/freeze 行锁和唯一 REVERSAL 决定单一终态，只释放或扣除一次 |
| 提现非法/负数/超精度/超限 | 创建事务前拒绝，COMMON 和冻结均不变化 |
| 提现误传 EXPERIENCE | 明确拒绝；不得隐式兑换或冻结 EXPERIENCE |
| ACTIVE Authenticator/TOTP 缺失、错误或同 step 重放 | 在建单/冻结前拒绝；事务失败不推进 `last_accepted_step`，同一步并发只有一笔可提交 |
| 曾成功确认 OTP 后被 reset/revoke | 返回稳定 `WITHDRAWAL_TOTP_REBIND_REQUIRED`，须重新绑定或受控恢复；不得按 NEVER_BOUND 降级 |
| CLOSING 用户调用普通提现/客户端指定批次金额 | 拒绝；只能以 closure purpose、当前 closure 和锁内计算的最大合法有界批次走清退入口 |
| 用户取消与管理员审核并发 | 仅一个 `WHERE status=AWAIT_REVIEW` 成功；取消胜出则释放，审核胜出则禁止取消 |
| 出款失败证据后又出现权威成功 | 未完成 `FinalizeNotAcceptedPayoutFailure` 前，成功可正常收敛 ARRIVED；本地已 FAILED/REJECTED/CANCELED 并释放冻结后不得重开或再扣钱包，每个未反转成功 attempt 固定为 `LATE_AFTER_LOCAL_FAILURE`，创建唯一 `PAYOUT_SUCCESS_AFTER_LOCAL_FAILED` RECEIVABLE Finance case；证据/disposition/case 同事务，P0 告警只负责运营升级，不替代该确定性账务路径 |
| 到账后重复/失败回调 | ARRIVED 保持，SettleFreeze 幂等，无第二次扣减/退回 |
| worker 崩溃在外部调用之后 | `next_run_at` 到期后按原商户号查询并收敛，不盲发第二笔 |
| 充值退款时 COMMON 已消费 | 返回 `REFUND_BALANCE_INSUFFICIENT`；订单仍 PAID，不建 freeze/attempt/case、不调用外部退款，余额恢复后才可重试 |
| 退款调用超时/临时失败 | 保持 REFUND_PENDING 和 ACTIVE freeze，复用唯一 attempt/商户号回查；不得自动回 PAID 或释放 |
| 显式取消退款后迟到成功 | REFUND_CANCELED 不回退、RELEASED freeze 不再终结；标 P0 渠道差异并走受控追偿 |
| slave 延迟 | 不参与支付/提现状态推进；历史列表可延迟但资金和读己之写走 primary |
| Redis/通知不可用 | 业务提交不回滚；primary sweep 在 SLA 内恢复 |
| 渠道持续不可用 | 指数退避、熔断新单、保留在途回查、达到 SLA 告警；不能批量标失败释放资金 |

## 11. 缓存与作业

### 11.1 缓存

- 已发布充值/VIP 商品和渠道展示配置可按版本缓存；后台发布后提交再失效，订单始终保存命中的版本。
- 订单/提现状态默认不缓存；如为轮询削峰使用短 TTL，PAID/ARRIVED 响应必须能强制 primary 读己之写。
- KYC、OTP、银行卡归属和可用余额等决策不能只读缓存。
- 支付 URL 可以缓存展示，但不能据此判断支付成功。

### 11.2 作业清单

| 作业 | 频率/入口 | 行为 |
| --- | --- | --- |
| 收款 attempt sweep | 秒/分钟级 primary sweep | 创建/查询 CREATED、PENDING、UNKNOWN 尝试 |
| COLLECT 最终性 sweep | 风险窗后分钟/小时级 primary sweep | 扫所有 finality OPEN（含 FAILED/EXPIRED），稳定查单并收敛 PAID/NOT_PAID，唤醒 closure |
| 本地订单过期 | 分钟级 primary sweep | 关闭未支付展示入口；不覆盖真实渠道成功；VIP finality 仍 OPEN 时保留 slot reservation，只有 CONFIRMED_NOT_PAID 同事务释放 |
| 退款回查 | 秒/分钟级 primary sweep | REFUND_PENDING 始终复用唯一退款 attempt；未知/临时失败不自动解冻；VIP 权威未受理以证据终结 REFUND_NOT_ACCEPTED，充值只有显式取消才释放 refund freeze |
| VIP 激活/到期 | 分钟级 + 请求时校验 | QUEUED→ACTIVE；停机跨过完整区间时 ACTIVE/QUEUED→EXPIRED，保证区间不重叠且每用户最多处理 64 条 |
| VIP 冻结返佣释放/追偿 | 复用返佣域 primary sweep | 到期释放 FROZEN GRANT，退款冲正按唯一 REVERSAL 收敛 |
| 提现提交 sweep | 秒级 primary sweep | 认领 APPROVED，事务外提交出款 |
| 出款 attempt 回查/NOT_ACCEPTED 裁决 | 秒/分钟级 attempt primary sweep | 用完整 lease token查询活动/STOP 状态；NOT_ACCEPTED 无候选或达 8 次时幂等失败释放，有替代渠道则等待受控 Switch |
| 渠道日终对账 | 每渠道账单周期 | 比较渠道、尝试、业务单、资产/权益/冻结，生成差异项 |
| 冻结对账 | 至少每日，重大渠道更高频 | 提现状态与冻结状态/金额逐单核对 |
| SLA 扫描 | 每分钟 | 告警长时间 PENDING、APPROVED、PAYING、FAIL_CONFIRMING 和重试耗尽 |

所有作业记录批次、游标、成功/失败/跳过数和下一次时间。重试耗尽只升级告警/人工处理，不能把未知资金状态自动改成失败。

## 12. 权限、安全与审计

- 用户只能创建和读取自己的订单/提现；`user_id` 来自令牌，银行卡必须属于同一用户且未删除。
- 提现要求账号/purpose 正确、KYC APPROVED、银行卡核名通过，并执行固定的 Authenticator/TOTP 状态规则；OTP 不是短信验证码，提现不以手机号存在作为安全前置，也不填固定假号码。删除的只是每笔提现专用人脸步骤，基础 KYC 保留。
- 管理权限以全局规范的 `payment.read`、`payment.config.write`、`payment.reconcile`、`withdrawal.read`、`withdrawal.review`、`withdrawal.payout`、`withdrawal.payout.fail` 为主；人工补单另设更窄的 `payment.settle`，列表权限不包含动钱。`withdrawal.payout.fail` 只允许执行权威 NOT_ACCEPTED 的失败释放，不能替代普通出款或任意置失败。
- 人工补单、人工提现审核、渠道切换、退款和对账消差记录 `operator_admin_id/reviewer_admin_id`、原因、工单、IP、前后状态、证据 hash；这些 admin 字段外键只指向 `admin_users.id`，不得混作普通 `user_id`。自动提现审核改用 `reviewer_type=SYSTEM + reviewer_system_code + system_policy_version_id` 的条件字段，不伪造 admin。高金额人工动作双人复核。
- 回调使用原始 body 验签，密钥按 provider/environment/version 管理；可叠加 mTLS 和 IP 白名单，但不能替代签名。
- 回调入口限流和 body 大小受限；日志/指标不得包含完整银行卡、手机号、支付凭证、签名密钥或未脱敏原文。
- 渠道回调基址来自环境配置并在启动时校验 HTTPS/host；不硬编码生产域名。

## 13. 指标与告警

按 provider、method、purpose、状态和错误码至少提供：

- 下单成功率、外部创建耗时、UNKNOWN 比例、支付成功率、从渠道支付到本地履约的延迟。
- 回调总数、验签失败、金额/币种不符、重复、乱序、终态冲突、处理事务失败和 ACK 延迟。
- `deposit_orders` 收款生命周期与原始 credit 缺失/金额不符数，以及 REFUND_PENDING 非 ACTIVE、REFUNDED 缺唯一 settle、REFUND_CANCELED 缺唯一 release 的数量；全部必须为零。
- VIP 订单未履约数、`reserved_slots/entitlement_slots` 分布、达到 64 的拒绝数、stats 与订单/权益计数差异、权益区间重叠数、单次 Ensure/退款触碰行数（必须 `<=64`）、REFUND_NOT_ACCEPTED 数/年龄/迟到成功 case、冻结返佣待释放/冲正/追偿数、`PURCHASE_ONE_TIME_VIP` 来源差异和 QUEUED/ACTIVE 到期处理延迟。
- 提现申请/审核/出款各阶段耗时，APPROVED/PAYING/FAIL_CONFIRMING 数量和年龄，NOT_ACCEPTED switch/failure-resolution 数、attempt lease 回收/上限命中、重复出款拦截数；另监控 canonical 数不为 0/1、active success 与 disposition 不等、duplicate 缺案、反转后旧案未取消/未建反向案，任一非零为 P0。
- 提现状态与冻结状态不符、ACTIVE 冻结金额不符、终态回调冲突；任一非零为 P0。
- primary sweep 批次数、取数延迟、租约回收、重试分布、死锁和数据库锁等待。
- 每日渠道收款/退款/出款金额与本地订单、COMMON 流水、手续费、冻结结算差额。

告警要能从 provider/psp_ref/merchant_request_no 追到本地订单、`user_id`、资产流水和冻结明细，但通知中只放脱敏标识。

## 14. 验收用例

### 14.1 充值

1. 固定套餐与任意金额充值各成功一次，COMMON 增量严格等于 `common_amount`；全库 Proto、Schema、任务和流水不存在新赠送字段/类型。
2. 相同支付回调顺序、并发和重放 100 次，只产生一次 PAID 迁移和一条充值流水。
3. 金额少付、多付、币种错误、商户错误、签名错误、渠道单号复用均不入账并产生相应审计/告警。
4. 覆盖“回调先到、创建响应后到”“本地已过期、成功回调后到”“人工补单与回调并发”，均只履约一次。
5. 在订单更新、资产更新、流水插入和事务提交响应处注入故障，恢复后订单与资产最终一致且没有重复入账。
6. 创建渠道请求超时后 worker 崩溃，恢复时按同一商户号查单，不创建第二个真实收款意图。
7. 充值成功退款路径依次走 PAID、REFUND_PENDING、REFUNDED：始终只保留一条原始 credit，PENDING 恰有一条 ACTIVE refund freeze，REFUNDED 恰有一条等额 settle；退款回调重放不产生第二次扣款。
8. 退款超时、临时失败和 worker 重启均保持同一 REFUND_PENDING/attempt/商户号。渠道明确未受理后显式取消只产生一条 release 并进入 REFUND_CANCELED；同单再退款被拒绝，取消与成功回调并发只收敛到一个冻结终态，迟到矛盾进入人工追偿。
9. FAILED/EXPIRED 订单释放限额 reservation 后收到权威迟付：原 usage 窗口幂等增加 completed、必要时标 over-limit 但仍只入账一次；COLLECT attempt 同事务 FAILED→SUCCEEDED 并引用纠正事件。订单 PAID、attempt FAILED 或 usage 缺记均视为失败。
10. 停掉普通收款 worker，只留下 FAILED/EXPIRED + finality OPEN 行；finality sweep 仍由独立索引领取。确认未收款时写唯一证据并唤醒 closure；确认已收款时同事务履约并转 CONFIRMED_PAID。worker 崩溃/旧 lease 迟到不覆盖新结果，未知结果不因重试耗尽误转 NOT_PAID。
11. 历史 CONFIRMED_PAID 在资产/权益/退款全部终结并清退后不再单独阻塞注销；OPEN finality 必须阻塞。CONFIRMED_NOT_PAID 后的相反渠道证据走 P0 差异，不能向 CLOSED 钱包静默入账。
12. `deposit_products(product_code,version)` 并发重复发布由唯一约束拒绝；同 code 销售区间重叠拒绝，边界相接按半开区间唯一选中。VIP 商品执行相同测试。
13. 把创建充值/VIP 暂停在事务外预检、user 锁前和订单/attempt 插入前，分别与 Request/CompleteClosure 正反并发：建单先持 user 时注销锁内看到 finality OPEN blocker；注销先持 user 时建单因 CLOSING/CLOSED 整体失败。任何次序都不得在 CLOSED 后留下 PENDING order、COLLECT attempt 或额度 reservation，锁轨迹不得出现 usage→user。

### 14.2 一次性 VIP

1. 首购与复购相同商品都使用同一已发布显式价格；不同支付渠道不改变订单价。
2. 搜索 Schema、Proto 和运行任务，确认无优惠券、首购价、自动续订、订阅回调、转移、VIP 邀请和赠币入口。
3. 两张 VIP 同时支付时只一张 ACTIVE，另一张按确定顺序 QUEUED，时间区间无重叠。
4. 支付事务任一环节失败时订单不 PAID、权益、冻结返佣、任务进度均不残留；重放后整体一次成功。
5. 停掉 VIP worker 并跨越 ACTIVE 结束及一到多条 QUEUED 开始边界；播放鉴权调用 `EnsureVipEntitlementCurrent` 同步过期/激活并给出正确结果，状态恢复后仍最多一条 ACTIVE。
6. 直属/间接两层按 `rebate_base_common` 计算 COMMON 冻结返佣，第三层无记录；ZERO_DIFF/ZERO_ROUNDED 只落 calculation audit，正额 ACTIVE 才建冻结、正额非 ACTIVE 才建 forfeiture。到期只释放一次，退款前/后分别取消冻结或受控追偿。
7. 权益创建只以 `ONE_TIME_VIP_ENTITLEMENT_CREATED` 推进 `PURCHASE_ONE_TIME_VIP` 一次；Schema、Proto、任务和指标中没有 `SUB_VIP` 或订阅续费语义。
8. 退款撤销权益并创建唯一返佣 REVERSAL，但保留原订单、支付事件、权益、GRANT 和任务历史；重复/乱序退款无第二次资产效果。
9. 分别退款 ACTIVE、队首 QUEUED 和队列中段权益；目标 REVOKED 后其余未终结权益按稳定次序连续重排、总购买时长不损失且无重叠，每个移动都有唯一 schedule audit。退款与新购/到期 worker 并发仍最多一条 ACTIVE。
10. 同一 VIP 退款在 UNKNOWN、明确拒绝、worker 重启和人工重试下始终只有一条 REFUND attempt、一个 `:refund:1` 商户号；尝试换键或创建第二行由数据库拒绝。
11. 同一用户并发创建 65 张未决 VIP 订单，只有前 64 个成功并各占一个 RESERVED slot，第 65 个在创建任何外部 COLLECT 前失败；同键同 request 重放返回原订单且不多占 slot，同键不同 request 冲突。任意顺序支付、到期、退款和 CONFIRMED_NOT_PAID 后始终满足 `reserved_slots + entitlement_slots <= 64`，stats 与订单/权益规范 COUNT 相等。
12. 本地 EXPIRED/FAILED 但 finality OPEN 不释放 VIP slot，迟付在原 reservation 内只履约一次；CONFIRMED_NOT_PAID 与重复 sweep 只释放一次，其后相反成功证据不得创建第 65 条权益。暂停 worker 跨越 64 条队列边界，Ensure/退款每事务最多锁 64 条未终结权益且不扫描历史行。

### 14.3 提现与冻结

1. 空、非数字、零、负数、超精度、低于/高于限额、手续费后净额不正、余额不足逐项拒绝，且无提现/冻结/流水残留。
2. `EXPERIENCE` 提现始终拒绝；正常 COMMON 提现后 `available` 减少量、`frozen` 增加量、冻结明细金额和 `requested_common` 四者相等。
3. 用户取消与管理员批准/驳回并发时只一个动作成功；CANCELED/REJECTED 只 RELEASED 一次，批准后禁止取消且 APPROVED 由 primary sweep 认领。
4. 出款创建超时、响应丢失、worker 崩溃和重复认领均不产生双出款；用渠道商户号证明唯一。
5. 对同一提现排列并重放 CANCEL/REJECT/PENDING/FAILED/SUCCESS 动作，覆盖取消与审核竞态、失败先到、成功先到、失败确认前成功、终态矛盾；冻结只能有一个终态和一次资产变化。
6. ARRIVED 必须对应 SETTLED，CANCELED/REJECTED/FAILED 必须对应 RELEASED，所有在途状态必须对应 ACTIVE；全表对账差额为零。
7. 费率、最低费、Asia/Jakarta 周一边界、汇率和渠道舍入逐项做边界测试，回调核对使用订单快照而不是当前配置。
8. 分别覆盖 NEVER_BOUND、ACTIVE、REBIND_REQUIRED：NEVER_BOUND 可无 `otp_code` 创建并保存 `otp_mode=NOT_REQUIRED` 与两个空引用；ACTIVE 缺失/错误码拒绝且无单/冻结，同一步 100 次并发只有一笔提交、推进 step，并保存 `otp_mode=TOTP/otp_binding_id/otp_accepted_step`；曾绑定后 reset/revoke 必须重绑或受控恢复。OTP 后续成功但金额/银行卡/限额校验失败时 step 随事务回滚；同幂等成功请求重放先返回原 withdrawal，不要求新码。普通和 closure withdrawal 执行同一矩阵。
9. CLOSING 用户不能创建普通提现，但可用新的 closure quote 按锁内余额、单笔/渠道上限和 DAY/WEEK 剩余额度的最小值创建本批最大合法清退单；前批终态或下一额度窗口后继续，直至 COMMON 归零。既有持仓后来再次入账后可再清退，所有批次仍走相同 TOTP、审核、出款和冻结状态机。

### 14.4 作业、权限与性能

1. 模拟 Redis 完全不可用，收款回查、VIP 到期、提现提交/回查和对账仍在 SLA 内由 primary sweep 完成。
2. 模拟 slave 延迟 60 秒，所有状态推进、幂等判断、冻结终结和读己之写仍正确且无 slave 资金查询。
3. 4 个以上 worker 并发 `FOR UPDATE SKIP LOCKED` 扫同一批到期数据，无重复外部命令、无遗漏；租约过期能回收崩溃任务。
4. 无权限管理员不能补单、审核、重试或配置渠道；有权限操作均生成不可变审计。
5. 渠道回调压测下事务不包含外部 HTTP，锁持有时长只覆盖本地校验和写入；p95、锁等待和吞吐不劣于同环境 1.0 基线。
6. 导入一份含重复、缺失、金额差异和终态矛盾的渠道账单，对账分类准确；只自动修复明确且幂等的缺履约项，其余生成工单。
7. 对余额不足和账号 CLOSED 的 VIP 原受益人分别确认退款：付款人均进入 REFUNDED，第三方不发生部分/负额钱包扣回，原 GRANT 保持 RELEASED，唯一 REVERSAL=EXTERNAL_RECOVERY 并引用唯一 Finance case。并发 resolve/waive 只有匹配 expected_version 的一方终结；resolve 必须有受信证据，waive 必须有异人审批。同键重放返回原结果，管理锁轨迹无 case→返佣/订单反向边，case 终结前后用户资产与返佣事实完全不变。

## 15. 参考、改造与拒绝

| 分类 | 参考对象 | 2.0 结论 |
| --- | --- | --- |
| 参考 | `../../../max/shortmax/migrations/default/0008_payment.up.sql` | 采用商户单号/渠道单号唯一、金额/汇率/银行卡快照、金额 CHECK 和面向待处理状态的部分索引 |
| 参考 | `../../../max/shortmax/models/payment.go` | 采用穷举状态、显式允许边、终态幂等、渠道金额区间和收款账户快照 |
| 参考 | `../../../max/shortmax/services/asset/callback.go`、`deposit.go`、`withdraw.go` | 采用适配器先验签并归一事件、统一履约器、回调查单和订单级幂等 |
| 参考 | `../../../max/shortmax/services/asset/*_test.go`、`../../../max/shortmax/services/man/withdraw_test.go` | 采用回调重放、少付拒绝、终态状态机、权限和人工补单/迟到回调竞态测试 |
| 改造 | shortmax 的 `deposit_orders.bonus_amount`/`TotalCoin()` | 2.0 不建赠送列，充值流水永远只等于明确购买的 COMMON |
| 改造 | shortmax 只覆盖充值/提现 | 增加显式价一次性 VIP、权益队列以及充值/VIP 共用的 `payment_attempts` 渠道层 |
| 改造 | shortmax 提现只依赖账户 frozen | 提现必须引用 `asset_freezes`，业务状态与 ACTIVE/RELEASED/SETTLED 逐单守恒 |
| 改造 | 直接从待审进入 PAYING 并同步出款 | 审核只写 APPROVED；`status + next_run_at` primary sweep 认领后才在事务外调用渠道 |
| 改造 | 失败回调立即释放 | 增加 FAIL_CONFIRMING 和渠道回查，降低失败/成功乱序造成“已退款又到账”的风险 |
| 拒绝 | `splay/models/point.go` 的万能 `Order` 承担不相关商品 | 充值、VIP、提现各有清晰业务聚合，渠道交互只抽成 attempt，不做全业务万能订单 |
| 拒绝 | `splay/services/point/order.go` 的首购 1 单位价和优惠券分支 | 一次性 VIP 只按不可变已发布商品版本定价，价格可复现 |
| 拒绝 | `splay/models/plan.go` 的 `SubPrice/SubId/PointAmount` 及续订/转移逻辑 | 不建订阅、自动续费、VIP 转移或购买赠币字段、API、回调和任务 |
| 拒绝 | 充值 `Amount + Giveaway`、充值匹配/翻倍和活动消费者 | 新库中没有相应字段、枚举、事件、缓存或统计入口 |
| 拒绝 | Redis 提现队列作为唯一工作源、网络调用包在数据库事务中 | 业务行可恢复、primary sweep 兜底；所有远程调用事务外执行 |
| 拒绝 | 固定手机号、硬编码回调域名、未知渠道状态猜测 | 缺必要真实数据即拒绝；域名/密钥配置化；UNKNOWN 必须回查 |
| 拒绝 | 通用 Outbox | 同库事实单事务，外部工作由订单/attempt/提现的 `status + next_run_at` 恢复；未来例外必须单独 ADR |
