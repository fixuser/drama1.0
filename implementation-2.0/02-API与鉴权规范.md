# 02 API 与鉴权规范

## 1. 目标与边界

2.0 使用 Proto 定义 public/admin 服务、消息和枚举，经 gRPC-Gateway 暴露 HTTP/JSON。公共端与管理端物理分包、逻辑分服务、权限分集合；没有 `/v1` 路由、旧 RPC、旧字段别名或兼容代理。第三方 callback 不先解码为 Proto，而由裸 HTTP Adapter 保留原始字节并验签后映射为内部 typed command。

本章约束所有同步 API 和外部回调。内部 worker 直接调用 Use Case，不绕回 HTTP。

## 2. Proto 目录与生成物

```text
api/
├── public/<domain>/v2/<domain>.proto
├── admin/<domain>/v2/<domain>.proto
└── common/v2/
    ├── common.proto
    ├── error.proto
    ├── money.proto
    └── page.proto

internal/adapter/callback/<provider>/<event>.go
```

首发 callback domain 为 `payment`、`kyc`、`ad` 和 `video`。每个 Adapter 只负责原始 HTTP 限长、content type 校验、供应商验签、标准 typed command 映射和专属 ACK，不复用 public/admin token，也不把供应商字段直接暴露给领域 Use Case。当前同进程边界不建立 callback Proto；将来只有 callback 独立成受信进程时，才经 ADR 引入 `api/internal/...` Proto 和 mTLS，外部供应商 wire format 仍不改成 Proto。

建议 Go package 分别为 `api/public/<domain>/v2;<domain>v2` 与 `api/admin/<domain>/v2;<domain>adminv2`。生成物不手工修改；CI 校验 Proto 格式、breaking change、Gateway 路由冲突、OpenAPI 生成和权限注解完整性。

规则：

- HTTP 路径以 `/v2` 开头；管理端使用 `/v2/admin`。任何 `google.api.http` binding 都不得包含路径参数、通配符或未展开路径：GET selector 只放 query，POST selector 只放 Proto body，动作使用 `/create`、`/detail`、`/cancel`、`/retry` 等固定字面量路径。
- 供应商 callback 不属于 Gateway binding。每个 Adapter 在 callback route registry 登记唯一 `exact_path/provider_code/event_kind/max_body_bytes/content_types/ack_profile/signing_key_ref`；`exact_path` 必须是部署时已知的完整字面量，不能从 path/query/body 动态选择 provider、事件类型或验签算法。启动和 CI 拒绝模板符号、星号、重复路径和未绑定 Adapter。
- RPC 名使用动词 + 对象，如 `CreateWithdrawal`；消息使用 `CreateWithdrawalRequest/Response`。
- 每个枚举的零值为 `*_UNSPECIFIED`，服务端拒绝把零值写入业务事实。
- 自助资源使用 `/v2/me` 固定命名空间，请求消息中不出现可被客户端指定的 `user_id`。
- 管理员操作显式携带目标 `user_id`，并记录管理员主体、权限点、理由和工单号。
- 不把数据库模型原样暴露为 API；响应只含领域承诺字段。

支付类状态由 Proto 自身的受控枚举表达，不复用含义接近的旧状态。首发 `VipOrderStatus` 必须包含 `REFUND_NOT_ACCEPTED`：它表示渠道以权威证据确认退款从未受理，本地权益、返佣、任务和资产均保持原终态；它不是 `REFUNDED`、`REFUND_FAILED` 或可自动重试状态。公共订单详情和管理订单详情必须返回该精确状态及脱敏 `reason_code`，客户端不得自行折叠。

## 3. 通用数据契约

### 3.1 金额、时间和标识

| 类型 | API 表示 | 规则 |
| --- | --- | --- |
| 金额 | 十进制定点字符串，如 `"12.34000000"` | 禁止 JSON number、科学计数、正负号歧义和超精度；命令金额必须 `> 0` |
| 比率 | 十进制字符串，如 `"0.30000000"` | 明确表示比例而非百分数；字段名带 `_ratio` |
| 时间点 | Unix 秒 `int64` | 0 表示未设置只在明确 optional 字段允许；展示时区为 `Asia/Jakarta` |
| 业务日期 | `YYYY-MM-DD` 字符串 | 按 `Asia/Jakarta` 解释 |
| ID | 十进制 `int64` 或不透明字符串 ID | `user_id` 为 int64；订单号等不得携带业务秘密 |
| 版本 | `int64` | 配置版本、聚合版本和乐观锁版本不得混用 |

金额解析顺序固定为：非空、语法、精度、正数/允许零、业务上下限、资产类型、可用余额。任何一步失败都不得创建业务单或幂等成功记录。

### 3.2 分页和排序

所有列表请求使用：

```text
offset: int64 >= 0
limit:  int32, 1..100（管理导出另走异步任务）
```

响应使用 `items + offset + limit + total`。排序字段只能从接口白名单选择，并始终追加唯一 ID 作为稳定次序。`total` 与 items 使用同一过滤口径；用户可见弹幕的 total 必须在屏蔽和审核过滤之后计算。

### 3.3 通用响应元数据

- `request_id`：服务端回显或生成，只通过 HTTP `X-Request-ID`/gRPC metadata 传播并用于链路追踪，不替代业务幂等键。
- `server_time`：Unix 秒，用于 Banner 和业务时间展示。
- `rule_version`：影响价格、资格或状态的单一规则必须返回；跨期间统计若合法覆盖多个收益策略，返回有序去重的 `policy_version_ids`，不得伪装成一个版本。
- `reason_code`：`can_* = false` 时返回稳定原因，客户端不复制规则。

内容与 Banner 查询统一携带 `ClientContext{language,region,platform,app_build}`。`platform` 使用受控枚举，`app_build` 是平台内单调正整数而不是可歧义的版本字符串；Gateway 对 query/header/Proto 字段做唯一规范化，多个来源不一致时拒绝。H5 的 build 由已发布前端部署元数据注入；缺失或非法 context 返回稳定 `CLIENT_CONTEXT_REQUIRED/UNSUPPORTED_CLIENT`，不得当成命中全部版本。该 context 只用于普通受众匹配，不能开启审核账号、管理权限或其他特权分支；搜索、列表、购买、播放授权/finalize 和 Banner 点击复验必须调用同一匹配组件。

## 4. 身份认证与会话

### 4.1 公共端

1. 注册、登录、验证码申请和公开内容为匿名白名单。
2. 其余接口使用短时 Access Token；Refresh Token 只用于换发并支持单会话撤销。
3. Token 至少包含 `subject=user_id`、`session_id`、`family_id`、`family_generation`、`auth_version`、`issued_at`、`expires_at` 和 `auth_level`；服务端在 primary 校验账号状态、`users.auth_version`、family generation/status 与 session，任一版本不符立即失效。全局撤销只递增用户 auth version，不扫描历史会话。
4. 修改密码、支付密码、银行卡、关系后绑定和注销按策略要求近期二次认证。提现保留 KYC/银行卡前置并采用条件 Authenticator/TOTP，不建立提现专用人脸流程。
5. 用户禁用状态阻止登录；团队禁用只影响团队相关权限。
6. 不存在超级密码、全局绕过码或管理员代替用户密码授权。

H5 无法安全保存共享签名密钥，因此不设计“客户端秘钥 HMAC”假安全。H5 使用 TLS、HttpOnly/SameSite Cookie 或 Bearer Token、Origin/CSRF 校验、短时二次认证、限流和重放保护。

短信/邮件 challenge 的创建与验证由数据库事实承担；worker 提交领取事务后，事务外调用 Forge `message` 的 SMS/Email sender，再以短事务条件写回 `PROVIDER_ACCEPTED/RETRY_WAIT/UNKNOWN/DEAD`。只有能证明未接受的临时失败才可重试；网络超时、响应丢失或调用后未 finalize 的过期租约在无供应商幂等/查询证据时写 `UNKNOWN`，不得自动重发。Forge 成功只表示供应商接受请求，不是终端 `DELIVERED`；WhatsApp 不建立 channel、模板、配置或 API。

### 4.2 管理端

- 管理员身份与普通用户身份分表、分 token issuer、分 audience。
- 后台认证服务固定为 `api/admin/auth/v2`：`AcceptAdminInvitation`、`ConfirmAdminMfaActivation`、`StartAdminLogin`、`CompleteAdminLogin`、`RefreshAdminSession`、`LogoutAdminSession`，HTTP 使用 §8 登记的六个固定 `/v2/admin/auth` 动作路径。邀请接受只设置密码并返回短期 MFA 激活上下文；只有确认 OTP 后才原子转 ACTIVE，密码成功但 MFA 未完成绝不签发业务 token。
- 登录必须经过 password→短期 MFA challenge 两步；refresh 每次轮换 hash/family generation，logout 撤销当前 family。Accept invite、Complete login 和 Refresh 都要求 `request_nonce`，以 kind 对应的 invite/challenge/predecessor session + nonce 唯一写 `admin_auth_issuances`；提交后响应丢失时，同 request hash/device proof 在短 TTL 内返回相同加密 activation context/token pair，换 nonce/设备的已轮换 refresh 才按盗用撤销 family。Access Token 至少包含 `admin_id/admin_session_id/session_version/mfa_at/audience=admin-v2`，Gateway 每次在 primary 校验主体 ACTIVE、session 未撤销和版本一致。高风险操作要求近期 MFA；是否强制双人复核由稳定 action registry 和已发布金额/风险阈值决定，命中后没有单人绕过分支。
- `CreateAdminUser`/凭据恢复只生成一次性、有期限、仅存 hash 的 invite；重置密码或 MFA 先撤销全部会话、递增 session version 并置回 INVITED，必须重新接受恢复邀请和绑定 MFA。匿名响应防枚举；密码、邀请和 OTP 分别限流，challenge 尝试耗尽即失效。bootstrap 激活 token 只由部署时本机命令从 secret manager 生成，不进入 SQL、镜像或日志。
- 每个 RPC 声明唯一权限点；列表/详情与创建/修改/审核/出款分开授权。
- 数据范围是独立条件：全链关系范围不能用普通用户五层权益查询代替。
- 服务账号使用 mTLS + 短时工作负载凭证，不共享管理员 token。

### 4.3 外部回调 Adapter 与签名

支付/出款、KYC、广告和视频供应商回调必须：

1. router 只按 registry 的 `exact_path` 选择唯一 Adapter，不接受请求字段决定 provider、event kind 或密钥；
2. 在解析前检查 content type 和配置化 body 上限，并保留 raw body 与参与签名的原始 headers；按供应商官方算法验证签名、证书/密钥版本、时间窗和回调来源；
3. 验签通过后才解析并映射为内部 typed command；Gateway JSON→Proto、字段重排或重新序列化都不得发生在验签之前；
4. 校验 provider event ID，拒绝过期或重复 nonce；支付不接受回调金额覆盖订单快照，KYC/广告/播放事件不接受任意 `user_id` 绕过本地引用映射；
5. 以 `(provider, provider_event_id)` 建唯一约束；合法重放返回供应商要求的成功 ACK，但不重复履约；
6. 验签失败只写脱敏安全日志，不创建供应商事件、权益、播放或订单事实；Adapter 的成功/失败 ACK 按 provider profile 编码，不能套用通用 errkit JSON。

内部 gRPC 使用 mTLS；不依赖 HTTP 头中可伪造的“内部用户”字段。

### 4.4 提现条件 TOTP

提现所称 OTP 固定指 Authenticator/TOTP，不是短信验证码。`CreateWithdrawal` 与 `CreateClosureWithdrawal` 的 `otp_code` 为 sensitive optional 字段，按 primary 上绑定历史裁决：

- `NEVER_BOUND`：从未成功确认过 Authenticator binding，可不提交 `otp_code`；仅有从未 ACTIVE 的 PENDING/EXPIRED 记录仍属于此分支；
- `ACTIVE`：必须提交 `otp_code`；幂等已提交结果查询之后，提现事务锁 binding、验证有效 step，并以 `last_accepted_step < accepted_step` 条件推进，同一步并发最多一笔成功；
- `REBIND_REQUIRED`：至少一条历史 binding 曾成功确认、但当前没有 ACTIVE binding，必须完成恢复或重新绑定，不能按 NEVER_BOUND 跳过。管理员 reset 表达为 `REVOKED + reason=ADMIN_RESET`，不另造 RESET 状态。

原始 `otp_code` 不进入数据库、request hash、审计正文或日志。业务单、冻结、TOTP step 防重和命令结果同一 default 事务提交；若同一幂等请求已经成功，先返回原结果，不重新消费 OTP。提现专用人脸表、RPC、callback target、状态和错误码均不存在。

## 5. 命令幂等

资金、领取、创建订单、创建 Playlet Backing、购买、绑定关系和所有终态命令必须携带 `idempotency_key`；HTTP 同时允许 `Idempotency-Key` 头，若消息字段与头同时存在必须相同。

幂等作用域为 `(actor_type, actor_scope, rpc_name, idempotency_key)`，其中 `actor_scope` 是非空、不含明文 PII 的稳定字符串，`rpc_name` 保存完整的 package/service/method 名。已认证调用取 `USER:<user_id>` 或 `ADMIN:<admin_id>`；注册等匿名命令取 `ANON_REGISTER:<HMAC(server_key, credential_kind || normalized_identifier)>`，可空 `actor_id` 只供审计、绝不参与唯一约束。服务端对规范化请求计算 `request_hash`：

- 首次请求在业务事务内写 `command_deduplications` 和业务结果引用；
- 同键同 hash 返回首次结果，不重复执行业务；
- 同键不同 hash 返回 `IDEMPOTENCY_CONFLICT`；
- 首次事务回滚不留下“成功”；处理中断可由相同键安全重试；
- 财务命令的幂等记录随财务审计保留，不依赖有 TTL 的 Redis 键。

查询请求不要求幂等键。`request_id` 仅追踪一次网络请求，不能代替 `idempotency_key`。

## 6. 错误模型

`api/common/v2/error.proto` 是 public/admin 对外错误契约；Forge `errkit` 是 Transport/Service 层实现，不是领域模型。领域与 Repository 只返回稳定 sentinel/typed error 和原始 cause，不导入 Forge；Service 在唯一映射层把它们转换为 errkit error。

应用必须在创建 Forge gRPC/HTTP provider 之前一次性注册 `Encoder`、`Decoder`、code name、每个业务码的 gRPC code 与 HTTP status。`Encoder` 把 errkit error 编码为 Proto `ErrorDetail`，Gateway 再按同一 detail 输出 JSON；`Decoder` 用于 gRPC 边界恢复稳定业务码。进程全局配置不得被领域包或测试并发重复覆盖。

外部错误结构：

```text
code          稳定机器码，例如 ASSET_INSUFFICIENT_AVAILABLE
message       安全、稳定的英文开发者摘要；客户端只按 code 分支并自行本地化
field_errors  可选字段错误列表
retryable     由已登记错误码策略派生，客户端不得自行推断
metadata      非敏感的上限、差额、当前状态等
```

`request_id` 只通过 HTTP `X-Request-ID` 响应头或 gRPC metadata 传播，并进入日志/trace，不依赖 errkit detail。`field_errors` 由应用 Encoder 写入 Proto detail；`retryable` 由版本化错误码目录派生。业务 metadata 只能包含白名单非敏感字段。所有 SQL、网络库和供应商裸错误必须在 Service 边界用安全消息包装，完整 cause 只写脱敏内部日志；禁止让 Forge 对未分类 `err.Error()` 的默认处理泄露 SQL、地址、密钥或供应商报文。

| gRPC | HTTP | 典型场景 |
| --- | ---: | --- |
| `INVALID_ARGUMENT` | 400 | 金额、分页、字段格式错误 |
| `UNAUTHENTICATED` | 401 | token 缺失、过期、会话撤销 |
| `PERMISSION_DENIED` | 403 | 权限点、数据范围、二次认证不足 |
| `NOT_FOUND` | 404 | 资源不存在或不可见 |
| `ALREADY_EXISTS` | 409 | 唯一业务事实已存在 |
| `ABORTED` | 409 | 乐观并发或状态竞争，可安全重试 |
| `FAILED_PRECONDITION` | 412 | 状态、资格、余额或 KYC 前置不满足 |
| `RESOURCE_EXHAUSTED` | 429 | 频率、额度、验证码限制 |
| `INTERNAL` | 500 | 未分类内部错误；不泄露 SQL/供应商信息 |
| `UNAVAILABLE` | 503 | 临时依赖故障，响应标明是否可重试 |

资金领域至少定义 `AMOUNT_INVALID`、`ASSET_TYPE_NOT_ALLOWED`、`ASSET_INSUFFICIENT_AVAILABLE`、`FREEZE_NOT_ACTIVE`、`IDEMPOTENCY_CONFLICT`、`STATE_TRANSITION_INVALID`；提现另定义 `WITHDRAWAL_TOTP_REQUIRED`、`WITHDRAWAL_TOTP_INVALID`、`WITHDRAWAL_TOTP_REPLAYED`、`WITHDRAWAL_TOTP_REBIND_REQUIRED`。所有码在 `api/common/v2/error.proto` 集中登记，并由 CI 验证 code name、Encoder/Decoder、gRPC/HTTP mapping 和 retry policy 完整；漏配一项即失败，不能静默降级为 500。callback Adapter 的 provider ACK 不使用该错误封装。

## 7. 公共端接口清单

以下是 2.0 首发接口边界；字段级定义在对应 Proto，业务语义在领域文档。

| 领域 | RPC | HTTP |
| --- | --- | --- |
| Auth | `Register`、`StartLogin`、`CompleteLogin`、`RefreshSession`、`Logout`、`RequestPasswordReset`、`ConfirmPasswordReset` | `POST /v2/auth/register`、`/v2/auth/login/start`、`/v2/auth/login/complete`、`/v2/auth/session/refresh`、`/v2/auth/session/logout`、`/v2/auth/password-reset/request`、`/v2/auth/password-reset/confirm` |
| Security | `StartOtpBinding`、`ConfirmOtpBinding`、`ChangePassword`、`ChangePayPassword` | `POST /v2/me/security/otp-binding/start`、`/v2/me/security/otp-binding/confirm`、`/v2/me/security/password/change`、`/v2/me/security/pay-password/change` |
| User | `GetMe`、`UpdateMe`、`RequestAccountClosure`、`GetAccountClosure`、`GetClosureWithdrawalQuote`、`CreateClosureCommonDustDisposition`、`ForfeitClosureExperience` | `GET /v2/me`、`POST /v2/me/update`、`POST /v2/me/closure/request`、`GET /v2/me/closure`、`GET /v2/me/closure/withdrawal-quote`、`POST /v2/me/closure/common-dust/dispose`、`POST /v2/me/closure/experience/forfeit` |
| KYC | `InitializeKyc`、`GetKycStatus` | `POST /v2/me/kyc/initialize`、`GET /v2/me/kyc/status`；CLOSING 只允许 purpose=ACCOUNT_CLOSURE，不存在提现人脸 RPC |
| Bank | `ListSupportedBanks`、`CreateBankAccount`、`ListBankAccounts`、`UpdateBankAccount`、`DisableBankAccount` | `GET /v2/banks`、`POST /v2/me/bank-accounts/create`、`GET /v2/me/bank-accounts`、`POST /v2/me/bank-accounts/update`、`POST /v2/me/bank-accounts/disable` |
| Referral | `BindParent`、`GetReferralSummary`、`ListDirectMembers`、`ListTeamMembers` | `POST /v2/me/referral/bind-parent`、`GET /v2/me/referral/summary`、`GET /v2/me/referral/direct-members`、`GET /v2/me/referral/team-members` |
| Asset | `GetWallet`、`ListAssetLedgers`、`ListAssetFreezes`、`GetAssetFreeze`、`ExchangeExperience` | `GET /v2/me/assets/wallet`、`GET /v2/me/assets/ledgers`、`GET /v2/me/assets/freezes`、`GET /v2/me/assets/freezes/detail`、`POST /v2/me/assets/exchange`；详情 selector 放 query |
| Level | `GetMyLevel`、`GetLevelRules`、`GetTeamMetrics`、`ListLevelHistory` | `GET /v2/me/level`、`GET /v2/me/level/rules`、`GET /v2/me/level/team-metrics`、`GET /v2/me/level/history` |
| Rebate | `ListRebates`、`GetPartnerEstimate`、`ClaimPartnerSettlement` | `GET /v2/me/rebates`、`GET /v2/me/partner/estimate`、`POST /v2/me/partner/claim` |
| Playlet | `ListPlayletGroups`、`SearchPlaylets`、`GetPlaylet`、`ListEpisodes`、`AuthorizeEpisode` | `GET /v2/playlet-groups`、`GET /v2/playlets/search`、`GET /v2/playlets/detail`、`GET /v2/playlets/episodes`、`POST /v2/playlets/episodes/authorize`；GET selector 放 query，Authorize 必填稳定 request ID/幂等键 |
| Entitlement | `PurchaseEpisode`、`ListEpisodePurchases`、`RequestEpisodePurchaseRefund` | `POST /v2/me/episode-purchases/create`、`GET /v2/me/episode-purchases`、`POST /v2/me/episode-purchases/refund/request` |
| Library | `SetFavorite`、`ListFavorites`、`UpdateWatchProgress`、`ListWatchHistory` | `POST /v2/me/library/favorites/set`、`GET /v2/me/library/favorites`、`POST /v2/me/library/watch-progress/update`、`GET /v2/me/library/watch-history` |
| Ad/Playback | `VerifyAdCompletion`、`StartPlaybackSession`、`HeartbeatPlaybackSession` | `POST /v2/me/ad-unlocks/verify`、`POST /v2/me/playback-sessions/start`、`POST /v2/me/playback-sessions/heartbeat`；供应商事件只走精确注册的裸 HTTP Adapter |
| Barrage | `CreateBarrage`、`ListBarrages`、`ReportBarrage`、`BlockBarrage`、`BlockBarrageAuthor`、`UnblockBarrage`、`UnblockBarrageAuthor` | `POST /v2/barrages/create`、`GET /v2/barrages`、`POST /v2/barrages/report`、`POST /v2/me/barrages/block`、`POST /v2/me/barrage-authors/block`、`POST /v2/me/barrages/unblock`、`POST /v2/me/barrage-authors/unblock` |
| Playlet Backing | `ListPlayletBackingOffers`、`CreatePlayletBacking`、`GetPlayletBacking`、`ListPlayletBackings` | `GET /v2/me/playlet-backing-offers`、`POST /v2/me/playlet-backings/create`、`GET /v2/me/playlet-backings/detail`、`GET /v2/me/playlet-backings` |
| Robot | `ListRobotProducts`、`GetRobotProduct`、`PurchaseRobot`、`ListRobotHoldings`、`GetRobotHolding` | `GET /v2/robot-products`、`GET /v2/robot-products/detail`、`POST /v2/me/robot-holdings/purchase`、`GET /v2/me/robot-holdings`、`GET /v2/me/robot-holdings/detail` |
| Robot Activation Voucher | `ValidateRobotVoucher`、`RedeemRobotVoucher`、`ListRobotVouchers` | `POST /v2/me/robot-vouchers/validate`、`POST /v2/me/robot-vouchers/redeem`、`GET /v2/me/robot-vouchers` |
| Task | `ListTasks`、`GetTask`、`ClaimTaskReward` | `GET /v2/me/tasks`、`GET /v2/me/tasks/detail`、`POST /v2/me/tasks/claim` |
| Deposit | `ListDepositProducts`、`ListPaymentChannels`、`CreateDepositOrder`、`GetDepositOrder`、`ListDepositOrders` | `GET /v2/deposit-products`、`GET /v2/payment-channels`、`POST /v2/me/deposit-orders/create`、`GET /v2/me/deposit-orders/detail`、`GET /v2/me/deposit-orders` |
| VIP | `ListVipProducts`、`CreateVipOrder`、`GetVipOrder`、`ListVipEntitlements` | `GET /v2/vip-products`、`POST /v2/me/vip/orders/create`、`GET /v2/me/vip/orders/detail`、`GET /v2/me/vip/entitlements` |
| Withdrawal | `CreateWithdrawal`、`CreateClosureWithdrawal`、`GetWithdrawal`、`ListWithdrawals`、`CancelWithdrawal` | `POST /v2/me/withdrawals/create`、`POST /v2/me/closure/withdrawals/create`、`GET /v2/me/withdrawals/detail`、`GET /v2/me/withdrawals`、`POST /v2/me/withdrawals/cancel`；命令按 §4.4 条件提交 TOTP |
| Income | `GetIncomeOverview`、`GetIncomeSeries`、`GetMonthlyBill`、`GetAnnualIncomeSummary`、`GetIncomePercentile`、`ListIncomeDetails` | `GET /v2/me/income/overview`、`GET /v2/me/income/series`、`GET /v2/me/income/monthly-bill`、`GET /v2/me/income/annual-summary`、`GET /v2/me/income/percentile`、`GET /v2/me/income/details` |
| Banner | `ListBanners`、`RecordBannerImpression`、`RecordBannerClick` | `GET /v2/banners`、`POST /v2/banners/impression`、`POST /v2/banners/click`；点击在 primary 复验 |
| Notification | `ListNotifications`、`GetUnreadNotificationCount`、`MarkNotificationRead`、`MarkAllNotificationsRead`、`ArchiveNotification` | `GET /v2/me/notifications`、`GET /v2/me/notifications/unread-count`、`POST /v2/me/notifications/read`、`POST /v2/me/notifications/read-all`、`POST /v2/me/notifications/archive` |

游客只能访问显式公开的内容、机器人商品、VIP/充值商品和 Banner 查询；播放授权仍由服务端判断免费权益。

供应商 callback 不在 Proto 接口清单中。每个支付/出款、KYC、广告和视频 Adapter 必须在 route registry 提交具体 `exact_path` 后才能启用；注册路径是一个 provider + event kind 的不可变绑定，不能提供通用 provider/kind selector。同一 provider event ID 在所属事件表中唯一；客户端自行声明“广告完成”或“已经播放”不产生权益或真实播放事实。

## 8. 管理端接口与权限

| 领域 | 首发管理 RPC / 命令族 | 权限点 |
| --- | --- | --- |
| 后台认证 | `AcceptAdminInvitation`、`ConfirmAdminMfaActivation`、`StartAdminLogin`、`CompleteAdminLogin`、`RefreshAdminSession`、`LogoutAdminSession` | 固定 `POST /v2/admin/auth/invitation/accept`、`/v2/admin/auth/mfa/confirm`、`/v2/admin/auth/login/start`、`/v2/admin/auth/login/complete`、`/v2/admin/auth/session/refresh`、`/v2/admin/auth/session/logout`；认证阶段不以 RBAC 替代身份校验 |
| 用户/注销 | `ListUsers`、`GetUser`、`UpdateUserStatus`、`GetAccountClosure`、`RecheckAccountClosure` | `user.read`、`user.status.write`、`closure.read`、`closure.recheck` |
| 后台主体/RBAC | `ListAdminUsers`、`CreateAdminUser`、`UpdateAdminUserStatus`、`ResetAdminMfa`、`ResetAdminCredential`、角色/权限分配 | `admin_user.read/write`、`admin_user.mfa.reset`、`admin_user.credential.reset`、`rbac.read/write`；两类 reset 均撤销全部后台会话并回到 INVITED |
| 高风险审批 | `CreateAdminActionApproval`、`ApproveAdminActionApproval`、`GetAdminActionApproval` | `POST /v2/admin/action-approvals/create`、`POST /v2/admin/action-approvals/approve`、`GET /v2/admin/action-approvals/detail`；`approval.request`、`approval.approve` + 目标动作权限 |
| KYC/银行 | `ReviewKyc`、`ReviewBankAccount` | `kyc.read`、`kyc.review`、`bank.review` |
| 关系 | `GetReferralGraph`、`GetRelationAudit`、`UpdateInviteCapability`、`UpdateTeamCapability`、`GetRelationInvariantReport` | `referral.read`、`referral.audit.read`、`referral.capability.write`、`referral.invariant.read` |
| 资产 | `GetUserAssets`、`ListAssetLedgers`、`CreateAssetAdjustment` | `asset.read`、`asset.adjust`；调整需双人复核 |
| Finance 追偿 | `ListFinanceRecoveryCases`、`GetFinanceRecoveryCase`、`StartFinanceRecoveryReview`、`ResolveFinanceRecoveryCase`、`WaiveFinanceRecoveryCase` | `finance.recovery.read`、`finance.recovery.resolve`、`finance.recovery.waive`；仅 RECEIVABLE 可异人复核后豁免，PAYABLE 只能凭外部付款证据解决 |
| 内容/Playlet Backing | 短剧/分组/剧集/revision/字幕 CRUD 与发布；`ListEpisodePurchaseRefunds`、`GetEpisodePurchaseRefund`、`ApproveEpisodePurchaseRefund`、`RejectEpisodePurchaseRefund`、`TerminatePlayletBacking` | `content.read/write/publish`、`episode_purchase.refund.read`、`episode_purchase.refund.decide`、`playlet_backing.terminate`；退款读取与裁决分权，决定只作用于唯一 REQUESTED 事实，终止需补偿快照和阈值审批 |
| 弹幕 | 举报队列、通过、隐藏、恢复、`RestrictBarrageAuthor`、`RestoreBarrageAuthor` | `barrage.read`、`barrage.moderate`、`barrage.author.restrict`；恢复不删除用户私有 block |
| 机器人 | 商品/规则发布、持仓/结算诊断、`CancelRobotHolding` | `robot.read/write/publish`、`robot.holding.cancel`；取消需工单、幂等及阈值审批 |
| Robot Activation Voucher | 批次、凭证码、发放、作废、统计 | `robot_voucher.read`、`robot_voucher.issue`、`robot_voucher.void` |
| 等级/返佣/合伙人 | `Draft/Review/PublishLevelRule`、`RecalculateUserLevel`、账单诊断、`Create/ApprovePartnerAdjustment`、`SealPartnerMonth`、`RestartPartnerMonthGeneration` | `level.read`、`level.rule.publish`、`level.recalculate`、`settlement.read/adjust/seal/restart`；重启代次需异人审批 |
| 支付 | 商品/渠道/汇率/费率草稿发布、订单/回查、`ConfirmPayment`、`StartDepositRefund`、`CancelDepositRefund`、`StartVipRefund`、渠道对账 | `payment.read`、`payment.config.write/publish`、`payment.settle`、`payment.refund/cancel`、`payment.reconcile`；VIP 退款只有 Start，无本地 cancel，后续复用唯一 REFUND attempt 收敛 |
| 提现 | 列表、单笔/有界批次审核、出款、重试、拒绝、`SwitchPayoutChannel`、`FinalizeNotAcceptedPayoutFailure` | `withdrawal.read`、`withdrawal.review`、`withdrawal.payout`、`withdrawal.payout.fail`；Switch 与失败释放只接受当前 attempt 的同一权威 NOT_ACCEPTED 证据并互斥，主动放弃可用渠道需异人审批 |
| 任务/Banner/通知 | 任务版本、`CompensateUserTask`、Banner 投放发布、通知模板/投递诊断 | `task.read/write/publish`、`task.compensate`、`banner.read/publish`、`notification.read/retry` |
| 收益重述 | `CreateIncomeRestatement`、`GetIncomeRestatement`、`StartIncomeRestatementProof`、`ActivateIncomeRestatement`、`AbandonIncomeRestatement` | `stats.rebuild`、`stats.activate`、`stats.abandon`；激活/放弃均绑定 calc/root/cutover generation、契约 hash 和异人审批 |
| 配置/作业 | `Create/Review/PublishRuleVersion`、`ReplaceScheduledRuleVersion`、`ListJobRuns`、`RetryJobFailure`、`RestartFinanceDayGeneration`、`RestartPartnerMonthGeneration`、`ResolveReconciliation` | `config.read/write/publish`、`job.read/retry`、`stats.day.restart`、`settlement.restart`、`reconciliation.resolve`；未生效规则替换、日代次和合伙人月代次恢复需双人复核 |
| 审计/统计 | 审计检索、真实报表、`Create/List/Get/Retry/DownloadReportExport` | `audit.read`、`report.read`、`report.export`、`report.retry`、`report.download` |

上述表是首发命令登记，不是示例；领域文档新增管理写入口时必须同步此表、Proto、权限种子和 98 章覆盖矩阵，不能只在实现中藏 RPC。高风险管理命令必须提供 `reason` 和 `ticket_no`，使用幂等键，并把请求前后快照写入 `admin_audit_logs`。要求双人复核的命令先通过审批服务创建 `admin_action_approvals`；申请人与审批人必须不同，审批人同时具备 `approval.approve` 和目标动作权限。执行者只能消费已批准、未过期、目标/权限点/request hash 完全一致的审批，审批消费、业务变更和审计同事务。审批表不保存可执行 payload，也不是任务队列。管理接口不得提供“设置任意余额”或“跳到任意状态”；资产调整使用对称变动原语，状态纠错使用受限补偿命令。

`RestartFinanceDayGeneration` 固定为 `POST /v2/admin/finance-days/restart-generation`，不是接收任意 payload 的 job retry。请求体只允许 `business_date`、`expected_build_generation`、`expected_calc_version_id`、`expected_contract_hash`、`reason`、`ticket_no`、`approval_no`，并要求 `Idempotency-Key`；日期和这些字段全部进入 request hash。同键同语义返回原 generation，同键异义冲突；目标非 CLOSING、旧租约仍活动、期望代次/契约不符或审批目标/hash 不符均失败且不改 day。具体锁序与代次规则见 [10-统计作业配置与运维.md](10-统计作业配置与运维.md) §6.4.1。

`RestartPartnerMonthGeneration` 固定为 `POST /v2/admin/partner-months/restart-generation`。请求体只允许 `earning_month`、`expected_calculation_version`、`expected_contract_hash`、`replacement_calculator_build_id`、`reason`、`ticket_no`、`approval_no`，并要求 `Idempotency-Key`；月份和全部字段进入 request hash。只有 CALCULATING、旧 generation 全部无活动 lease、期望契约匹配且异人审批有效时，才能在一个短事务 ABANDON 旧代并创建同 cutoff/规则/seed/hash/前驱证明的新代；客户端不能改金额、cutoff、shard_count 或规则。通用 job retry 无权创建代次。

收益重述接口固定如下，均走 primary 并写命令防重与 `admin_audit_logs`：

| RPC | HTTP | 关键请求与状态前提 |
| --- | --- | --- |
| `CreateIncomeRestatement` | `POST /v2/admin/income-restatements` | `stats.rebuild`；提交单一 `fixed_policy_version_id`、Jakarta 日界半开区间、shard 上限、expected active/root、reason/ticket；服务端以当前 ACTIVE 为 base 复制扁平时间线并仅替换目标段，区间外继承此前重述；首发不接受无 selector 的多 policy 集合，且禁止拆开冻结生命周期 |
| `GetIncomeRestatement` | `GET /v2/admin/income-restatements/detail` | `calc_version_id` 放 query；`report.read`；primary 返回状态、root/cutover generation、dirty/proof 进度、segment/scope contract、scope current/selected generation、scope status/left-right crossing 摘要、proof hash 和失败原因，不返回可编辑 payload |
| `StartIncomeRestatementProof` | `POST /v2/admin/income-restatements/start-proof` | `calc_version_id` 放请求体；`stats.rebuild`；要求 expected state=CATCHING_UP、expected root/version/contract、dirty 为空、无 CLOSING day，且 selected scope generation 指向 VALIDATED、left/right crossing 均为 0、segment/scope hash 与 calc contract 完全一致；递增 proof generation并创建固定 shard 后转 PROVING |
| `ActivateIncomeRestatement` | `POST /v2/admin/income-restatements/activate` | `calc_version_id` 放请求体；`stats.activate` + 异人审批；绑定 expected candidate/root/proof/segment/scope contract，要求曾完整证明且 backlog 不超过固化上限；无 CLOSING day 才进入 QUIESCING，由可接管数据库租约排空并原子激活 |
| `AbandonIncomeRestatement` | `POST /v2/admin/income-restatements/abandon` | `calc_version_id` 放请求体；`stats.abandon` + 异人审批；绑定 expected state/root/cutover generation/contract hash，只能终结非 ACTIVE candidate 为 FAILED(ABANDONED_BY_ADMIN)，清 candidate/cutover 后恢复 writer，不删除证据 |

四个写命令都要求 `Idempotency-Key/reason/ticket_no`，request hash 覆盖 body 中的 calc ID、全部 expected 字段、策略/区间/分片和 approval target；同键异义冲突。Activate/Abandon 的审批目标和 hash 必须与最终命令完全一致且一次消费；客户端不能提交 source upper、proof hash、segments、ACTIVE 指针或任意状态。Activate 初次响应可为 `QUIESCING`，同键轮询/重放返回原 cutover generation；正常完成为 ACTIVE，仍处理中为 QUIESCING，若另一获批 Abandon 已终结则稳定返回 `FAILED/ABANDONED_BY_ADMIN`，不会另建任务或复活 candidate。

`ReplaceScheduledRuleVersion` 固定为 `POST /v2/admin/rule-versions/replace-scheduled`。请求体必须携带 `domain/target_version`，并只能选择 `RESTORE_PREDECESSOR`，或携带已 REVIEWED 的 `replacement_version` 执行 `REPLACE_SAME_BOUNDARY`，同时包含 `expected_domain_version/expected_timeline_hash/reason/ticket_no/approval_no` 与 `Idempotency-Key`。禁止客户端提交新生效时间；只有未生效、无引用的时间线尾版本可执行，退役目标、恢复前驱或同边界发布替代版、审批消费、防重和审计必须同事务。

Finance 追偿查询固定为 `GET /v2/admin/finance-recovery-cases` 和 `GET /v2/admin/finance-recovery-cases/detail?case_no=...`；列表使用 `offset + limit + total` 并按 `next_action_at,id` 稳定排序，只有显式标注 `eventual_consistency=true` 时可从 slave 返回 `as_of/stale`，详情固定走 primary。状态允许 `OPEN -> IN_REVIEW -> RESOLVED`、仅 RECEIVABLE 的 `IN_REVIEW -> WAIVED`，以及仅内部来源反转原语可执行的 `OPEN|IN_REVIEW -> CANCELED_BY_SOURCE_REVERSAL`。`StartFinanceRecoveryReview` 和 `ResolveFinanceRecoveryCase` 需要 `finance.recovery.resolve`，分别只能取得 OPEN、IN_REVIEW；`WaiveFinanceRecoveryCase` 需要 `finance.recovery.waive`，只能取得 IN_REVIEW 的 RECEIVABLE，PAYABLE 必须返回稳定业务错误且保持原状态。客户端没有 cancel RPC；查询响应对 canceled case 返回触发证据和可空 counter case，不把它伪装为 RESOLVED/WAIVED，所有终态不可重开。

写接口固定为 `POST /v2/admin/finance-recovery-cases/start-review`、`POST /v2/admin/finance-recovery-cases/resolve`、`POST /v2/admin/finance-recovery-cases/waive`，`case_no` 均放 Proto 请求体，并要求 `Idempotency-Key`、`expected_version`、`reason` 和 `ticket_no`；客户端不得提交或改写案件价值、主体、方向、来源和原返佣状态。`resolve` 必须提供可核验的 `external_settlement_provider/external_settlement_type/external_reference/evidence_hash/settled_at`；服务端从受信 Finance 证据按案件 `value_basis=ASSET|EXTERNAL|BOTH` 核对方向和全部必填价值列，并在同事务创建 case 一对一、凭据自然键全局唯一的 `finance_external_settlements`。同一凭据不能核销两案，不支持一笔凭据拆多案；`waive` 必须携带目标 case、目标动作、`finance.recovery.waive` 和 request hash 完全匹配的未消费异人审批。三个命令只推进案件状态并追加管理审计，禁止调用资产原语、重开账号、改写原 GRANT/REVERSAL 或补写历史钱包流水。

执行事务先用 `case_no` 在 primary 无锁定位 `finance_recovery_case_roots.id`，正式锁序固定为命令防重 → `admin_action_approvals`（仅 waive）→ recovery root → case 子事实 → `admin_audit_logs` INSERT，并在锁内重验 case 仍属于该 root、来源自然键未漂移。原 GRANT/REVERSAL 等来源事实已经不可变，管理命令不读取它们来裁决、更不得加锁或 UPDATE，因而不会形成 case→来源的反向边。waive 的审批消费、RECEIVABLE 方向检查、case 条件更新、命令结果和审计同一 default 事务；start-review/resolve 跳过 approval。PAYABLE 只能通过 `ResolveFinanceRecoveryCase` 提交不可枚举的受信付款证据；债权人豁免或法定应付核销若未来出现，必须另立 ADR 和专门会计事实。相同键同语义返回原结果；同键异义、旧版本、非法前态、resolve 缺证据、PAYABLE waive 或 waive 审批不匹配均失败且不改变案件。

## 9. 主从路由与缓存语义

- 命令、命令后的结果返回、余额、冻结、支付状态、资格、等级强校验、幂等查询全部走 primary。
- 列表、历史流水、内容流、报表和导出只有在接口定义标注 `eventual_consistency=true` 时才可走 slave。
- 写后需要“读己之写”的接口显式固定 primary；不能通过 sleep 等待 slave。
- 缓存响应携带内容/配置版本；权限、余额、提现和播放最终授权不从缓存直接判定。
- 管理写完成后在提交后失效缓存；信号丢失时短 TTL 或版本键使其自行恢复。

## 10. 限流、日志与隐私

- 登录、OTP、密码重置、支付回调、播放授权、弹幕发送、提现和管理审核分别限流，维度至少含 IP、主体和设备/供应商事件。
- 日志允许记录 `request_id`、主体 ID、方法、结果码、耗时和业务对象 ID；禁止记录密码、OTP、token、完整银行卡、KYC 文档、支付密钥和回调原文中的敏感字段。
- 请求体审计使用字段白名单与脱敏快照；金融审计不可被普通管理员删除。
- 导出异步生成，带权限快照、过期时间和下载审计。

## 11. 失败模式

| 失败 | 行为 |
| --- | --- |
| 客户端超时但服务端已提交 | 使用相同幂等键重试并返回原结果 |
| slave 延迟 | 强一致接口不路由 slave；允许延迟列表显示 `as_of` |
| Redis 不可用 | 跳过缓存/信号，不改变事实提交；限流按降级策略收紧 |
| 外部供应商超时 | 支付/出款/KYC 等具备稳定供应商键或查单能力的命令进入回查/收敛，不盲发第二条；SMS/Email 只有已证明未接受才可重试，无可靠幂等/查单证据的超时写 `UNKNOWN` 且不自动重发；未知结果一律不猜成成功或失败 |
| callback body 超限/content type 非法/验签失败 | Adapter 在解析和领域调用前拒绝，返回 provider profile 指定 ACK，不创建 provider event |
| 管理员权限中途撤销 | 每次命令重新校验；长导出保存并复核权限快照 |
| Proto 未知枚举 | 输入拒绝，输出监控告警；不得映射为“其他”继续写入 |

## 12. 验收用例

1. 所有 public/admin HTTP 路径均为 `/v2`；Gateway annotation 和 OpenAPI 路径中不存在路径参数、星号或未展开模板。GET selector 均来自 query，POST selector 均来自 Proto body；全仓不存在旧 RPC 注册和兼容字段。
2. 自助命令即使注入其他 `user_id` 也只能操作 token 主体，或直接因未知字段失败。
3. 每个资金/领取/订单命令无幂等键时失败；同键同请求重放结果一致，同键异义冲突。
4. 金额空值、负数、零、超精度、科学计数和越界值均在创建事实前失败。
5. 管理员只读角色无法执行修改；提现审核与出款权限互相独立。
6. 每个 callback Adapter 只匹配 route registry 的精确字面量路径；对 raw body 限长并在解析前验签，覆盖重排 JSON、未知字段、错误 content type、时间窗口、事件唯一键、金额快照、乱序终态和 provider 专属 ACK。外部 callback 不生成 Proto/Gateway binding。
7. 模拟 slave 延迟，余额、资格、幂等、播放授权和写后查询仍正确。
8. 从未成功确认 binding 的 `NEVER_BOUND` 提现可不带 TOTP，ACTIVE binding 缺码/错码失败且正确码单步只能消费一次；曾成功确认但当前已 REVOKED（含 ADMIN_RESET）必须进入 REBIND_REQUIRED；相同已提交幂等请求不重复消费 step。Schema、RPC、callback target 和 OpenAPI 中没有提现人脸对象。
9. Forge errkit 的 Encoder/Decoder、code name、gRPC/HTTP mapping 和 retry policy 全登记；领域包不导入 Forge。随机 SQL/网络/供应商错误不能把裸 cause 返回客户端，`X-Request-ID`/gRPC metadata、field errors 和 retryable 结果符合契约。
10. Forge `message` 仅在事务外发送 SMS/Email；供应商接受只记录 PROVIDER_ACCEPTED，明确未接受才重试，无可靠证据的不确定调用转 UNKNOWN 且不自动重发；Redis/Forge/供应商故障不改变业务事实。Proto、配置与 OpenAPI 中不存在 WhatsApp channel。
11. OpenAPI、Proto lint、权限注解、错误码登记、固定路由唯一性和 callback registry 完整性在 CI 中通过。

## 13. 参考、改造与拒绝

| 分类 | 采用方式 |
| --- | --- |
| 参考 `shortmax` | Proto 分域、gRPC-Gateway、公共/管理服务分离，以及 Forge errkit 的集中错误映射方式 |
| 2.0 改造 | 全部接口重新命名为 v2；固定无路径参数的 RPC 风格 HTTP；callback raw Adapter；把主体、幂等、规则版本、权限点和一致性等级写入契约 |
| 拒绝复制 `splay` | 多套 v1/V2 语义、请求体信任用户编号、错位枚举、前端复制门槛、超级密码、活动 API 混入核心服务 |
