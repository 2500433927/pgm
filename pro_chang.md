# 项目变更记录（pro_chang）

按版本号记录每次代码变更，便于回滚。版本号由 Claude Code 在每次修改完成后递增返回，并同步追加到本文件。

---

## 版本号引入前的变更（未编号）

- **报名页同意框加黑 + 未勾选提示**（frontend/src/pages/tournament/register.vue + i18n）：声明文字变黑加粗、未选中勾选框边框变黑；未勾选声明时按钮保持绿色可点，点击弹 toast"请勾选确认"（i18n key `confirmDeclarationRequired`）
- **admin-web「同步数据」功能**（backend/src/routes/admin/mod.rs + admin-web）：`POST /admin/championship/sync` 把内网 `logic.championship` 按 `_id` 去重**追加**到目标库（默认 `127.0.0.1:27017`，同机部署）；支持 `{"$oid": "..."}` 归一化与任意 `_id` 类型；期间修复了三类问题：非 ObjectId `_id` 全部被丢弃、目标库公网 IP 连接超时（改为本机回环）、旧代码未真正部署（响应加 version 标记）

## v1.0 — 赛事详情页排行榜功能（基线）

- **后端**：新增公开接口 `GET /api/tournament/{id}/championship-leaderboard`：按 `shipId`（兼容 ObjectId 与 hex 字符串两种存储格式）查询 logic 库 `championship` 集合，生成 比杆/最近旗杆/最远距离 三榜
- **排序规则**（对应游戏内逻辑）：stroke 升序；close 升序（`close > 50000000` 视为无成绩）；far 降序（`far < 0` 视为无成绩）；无成绩记录排最后、名次连续编号、score 为 null
- **数字容错解析**：Int32/Int64/Double/数字字符串（修复 get_i64 只认 Int64 导致全部"-"的问题）
- **前端**（detail.vue）：原"排行榜"Tab 内新增三个子 Tab + 列表（名次/玩家/成绩），无成绩显示"-"；距离展示：原始值 ÷100000 显示为米；隐藏原"排行榜+刷新"头部行；成绩列不换行
- i18n 新增 `lbStroke`/`lbClose`/`lbFar`/`lbScore`/`lbUnitStroke`/`lbUnitMeter`

## v1.1 — 排行榜刷新按钮（纯前端）

- 子 Tab 行右侧新增刷新按钮（图标 + "刷新"文字），点击无提示直接重新拉取当前榜单

## v1.2 — 刷新按钮后端同步逻辑

- 抽取共享同步服务 `services/championship_sync.rs`（admin 同步接口改为薄封装，行为不变）
- 新增 `POST /api/tournament/{id}/championship-refresh`：先执行内网 → 目标库 championship 追加同步（幂等），再返回最新榜单
- 前端刷新按钮接入该接口（同步失败自动退回普通拉取）

## 回退：v1.2 → v1.0

- 用户要求回退到 v1.0：撤销 v1.1 刷新按钮与 v1.2 后端同步逻辑（删除 championship_sync.rs、恢复 admin 内联实现、移除 championship-refresh 接口与前端按钮）

## v1.3 — 标签动态显示 + 距离单位改为码

- **后端**：读取赛事 `config.FarHole`/`CloseHole` 决定标签显示（值 0 → 不显示；> 0 → 显示且标签带洞号，如"最近旗杆-8H"）；不显示的榜单不构建、不返回
- **响应结构变更**：`data: { tabs: [{key, hole?}], boards: { stroke/close/far } }`；前端子 Tab 改为按后端 tabs 动态渲染（不硬编码），标签文案由 i18n + 洞号拼接
- **距离换算**：后端把米换算为码（×1.09361，保留 1 位小数），前端直接展示 `score.toFixed(1) + 码`；删除已无用的 `lbUnitMeter` key

## v1.4 — 距离换算链路修正

- 修正换算链路：**原值 ÷100000 = 米 → ×1.09361 = 码**（保留 1 位小数）——此前误把原值直接当米
- close/far 解析改为 f64 精确小数（Double 不再截断为整数；字符串小数如 "180.5" 也能解析）
- 无成绩判断仍在原值上进行：`close > 50000000` / `far < 0`

## v1.5 — 成绩列右对齐

- 排行榜成绩列（含表头"成绩"）`text-align` 由居中改为右对齐

## v1.6 — 成绩列右侧留白调整

- 成绩列增加 `padding-right: 20rpx`，右对齐内容不再贴边

## v1.7 — 排行榜数据源切换为内网 logic 库

- `championship_leaderboard` 的 championship 查询从本机 logic 库（`LOGIC_MONGODB_URI`）改为**内网 logic 库**（`INNER_MONGODB_URI`，默认 `mongodb://localhost:27117/`，经反向 SSH 隧道访问；`INNER_DB_NAME` 默认 logic）
- 连接写法与 `sync_tournaments_to_inner` 一致（`ClientOptions::parse` + `Client::with_options`，每请求建立）
- 连接失败返回 500 + 具体错误（Inner MongoDB client/connection error）
- 效果：排行榜实时读内网数据，不再依赖 admin-web「同步数据」的落库结果



## v1.8 — 修复 Google/Apple 第三方登录「退出后无法再登录，须卸载重装」

- **根因**：退出登录只清 App 自身存储，从未清除第三方 OAuth SDK 会话；再登录时 SDK 静默返回缓存的上次授权 token（已过期），后端拒绝后每次重试都是同一份过期凭据，死循环直到卸载重装清空 SDK 状态
- **前端**（stores/user.js）：`logout()` 增加 best-effort `uni.logout` 清除 google/apple/facebook 的 SDK 会话（仅 APP-PLUS），下次登录走完整授权流程签发全新 token
- **前端**（pages/login/login.vue）：
  - 新增 `decodeJwtPayload`/`isJwtExpired` 工具函数（本地解码 JWT exp，含 60s 时差）
  - Google 登录重构为 `runGoogleLoginFlow(isRetry)` + `clearGoogleSdkSession`：本地检测凭据 JWT 过期，或后端返回 `GOOGLE_TOKEN_EXPIRED`/`Invalid Google ID token` 时，清除 SDK 会话并自动重新授权重试一次
  - Apple 登录流程增加同样的本地过期预检（原先只依赖后端 `APPLE_TOKEN_EXPIRED` 再重试，多一轮失败往返）
- **后端**（services/auth_service.rs）：
  - `google_login` 凭据验证重构为降级链 `verify_google_credentials`：id_token 失败 → access_token（含本地 JWT 解码）→ openid，修复了此前 id_token 失败只回退 openid、iOS 端无 openid 直接失败的问题（与 wiki 设计的降级链一致）
  - `decode_google_jwt_payload` 补 `exp` 校验（60s 时差），不再接受过期缓存 token；新增 `google_jwt_exp` 辅助函数
  - `verify_google_access_token` 对 JWT 形凭据前置过期检测，返回稳定错误码 `GOOGLE_TOKEN_EXPIRED`（401）
  - `verify_google_token` 调用 tokeninfo 增加 5s 超时 + 3s 连接超时（国内服务器直连 Google 防挂起）
- **后端**（routes/google/session.rs）：错误响应改用 `AppError::error_response()`（与 Apple 登录一致），过期 token 返回 401 + 稳定 `message` 错误码而非一律 500 + 原始字符串，前端可据此识别「可重试」错误

## v1.9 — 赛事结束获奖通知 + 收货资料补填

- **新集合 `prize_infos`**（业务库）：获奖用户补填的手机号+收货地址，一次性提交不可修改；唯一索引 `{tournamentId, userId}` 防重复提交；启动时 `ensure_indexes` 自动建索引。建表语句：
  ```js
  use pgm_miniprogram   // dev 为 pgm_mp_trial
  db.createCollection("prize_infos")
  db.prize_infos.createIndex({ tournamentId: 1, userId: 1 }, { unique: true })
  db.prize_infos.createIndex({ userId: 1, createdAt: -1 })
  ```
- **抽取共享排名服务** `services/tournament_championship.rs`：championship_leaderboard 的内联逻辑（内网连接/容错解析/三榜构建）整体迁出，行为不变；新增 `prize_rank_by_openid`（获奖名次：仅统计有成绩者、按 openid 取最小 stroke、升序连续编号）
- **新接口**（routes/prize/mod.rs，AuthenticatedUser 鉴权）：
  - `GET /api/tournament/{id}/prize-status`：获奖状态（赛事名/联系方式/名次/杆数/已填资料），守卫 404 未结束 403 未报名 403
  - `POST /api/tournament/{id}/prize-info`：提交 {phone, address}，校验手机号 `^1\d{10}$`、地址 ≤200 字；无成绩 403；唯一索引冲突 → 409"已提交，不可修改"
  - `GET /api/prize-info/my`：我的已填获奖记录（读快照字段，不依赖内网库）
- **结束通知改造**（wechat_notify.rs）：EndNotice 文案改为"比赛已结束，请填写收货信息领取奖品"，跳转页改为 `pages/prize/fill?id=...`（开赛提醒不变）；触发链路（手动/自动 completed + Redis 防重）复用
- **前端**：新子包 pages/prize（fill 补填页四态：加载/无成绩/已填只读/表单；my 我的获奖记录页）；首页"获奖通知"入口卡片（复用 tournament-entry 样式，登录可见）；新 API 封装 utils/api/prize.js；i18n 新增 home.prizeEntry* + prize.* + prizeMy.*（zh/en）
- 注意：订阅消息为一次性授权且仅报名时收集，历史报名用户可能收不到新通知（首页入口兜底）；`pages/prize/fill` 需随小程序发布后订阅消息跳转才生效

## v1.10 — 开放普通用户扫码报名赛事（入口保持仅管理员可见）

- **问题**：赛事功能灰度门禁（`TOURNAMENT_GRAY_RELEASE=true`，.env/.env.prod）把普通用户扫码报名链路一并锁死：详情接口返回 404、报名接口返回 403「赛事功能内测中，暂不可用」
- **改动**（backend/src/routes/tournament/mod.rs，单文件解耦）：
  - 移除 `get` 详情接口的灰度拦截（普通用户扫码进详情可见赛事并显示报名按钮；draft/cancelled 仍由原 is_private 逻辑仅组织者/admin 可见，隐私保护不变）
  - 移除 `register` 报名、`unregister` 取消报名、`check_registered` 报名状态查询的灰度拦截（服务层校验不变：密码/重复报名/禁止组织者自报/仅 active 可报）
  - **保留** `list` 列表与 `feature_flags` 的灰度拦截：首页赛事入口卡片与赛事列表对普通用户仍隐藏（tournamentVisible=false）
  - 删除已无引用的常量 `ERROR_TOURNAMENT_FEATURE_LOCKED`（constant.rs）
- **效果**：普通用户扫码（t{id} 小程序码）→ 详情 → 报名 → 查看/取消报名全链路可用；入口显隐行为不变；灰度将来关闭后行为与改动前完全一致（被删检查本就是冗余）
- 环境变量 `TOURNAMENT_GRAY_RELEASE` 未改动（.env/.env.prod 保持 true）


## v1.11 — championship 查询数据源还原为本机 logic 库（撤销 v1.7）

- `services/tournament_championship.rs::fetch_entries` 签名改为 `(data: &web::Data<AppState>, tournament_id: &str)`，查询走 `data.logic_db`（`LOGIC_MONGODB_URI` + `LOGIC_DB_NAME`），删除内网 client 连接逻辑（INNER_MONGODB_URI/INNER_DB_NAME 不再用于排行榜与获奖名次）
- 不再依赖反向 SSH 隧道（setup_inner_tunnel.sh 仅 admin「同步数据」接口仍使用）
- 调用点同步更新：routes/tournament/mod.rs（championship_leaderboard）1 处、routes/prize/mod.rs（prize_status / submit_prize_info）2 处
- 其余排名/换算/获奖逻辑不变

## v1.12 — tournaments 集合双库轮询同步（业务库 → 逻辑库，仅生产）

- **新模块** `services/tournament_sync.rs`：业务库 `pgm_miniprogram.tournaments` → 逻辑库 `logic.tournaments` 的轮询式差异同步后台任务，main.rs 启动时挂载（`services::tournament_sync::start`）
- **同步语义**：以 `_id` 为基准的镜像同步——源有目标无/内容不同 → `replace_one(upsert=true)`（替换文档保留原始 `_id`，游戏侧 championship.shipId 按 `_id` 匹配不受影响）；目标有源无 → `delete_many($in)` 批量删除；全程以原始 BSON Document 传递（password/idStr 等隐式字段原样保留）
- **可靠性**：默认每 30 秒一轮（`TOURNAMENT_SYNC_INTERVAL_SECS` 可调，下限 5 秒），启动即先跑一轮；读失败整体中止本轮、写失败逐条记日志，下周期自动重试（自愈）；孤儿数 > 50 时 warn 留痕
- **门禁**：仅 `TOURNAMENT_SYNC_ENABLED=true/1` 时启用（默认关闭，不使用 APP_ENV 判断——prod 的 systemd 单元 Environment=APP_ENV=release 与 .env.prod 的 APP_ENV=prod 并存有歧义）；`.env.prod` 已开启，`.env.trial` 不开启（trial 与生产共享同一 logic 库，防止污染）
- **环境变量**：.env.prod 新增 `TOURNAMENT_SYNC_ENABLED=true`、`TOURNAMENT_SYNC_INTERVAL_SECS=30`；.env.example 补充注释说明；repowiki《环境配置》补充两个变量说明
- **发布注意**：部署后确认 `backend/build/prod/.env.prod` 已带新变量，`systemctl restart pgm-mp-prod` 后观察日志 `tournament sync: 本轮同步完成`

## v1.13 — admin-web 赛事接口补齐生命周期定时器与结束通知（与 MP 端对齐）

- **问题**：admin-web 的 `/api/admin/tournaments` 创建/更新/删除接口直接写 MongoDB，从不调用 Redis 定时器逻辑——经 admin-web 激活的赛事没有 `tournament:{db}:expire:{id}` / `remind:{id}` key（表现为开赛提醒、自动完结、结束通知全部失效）；此前仅 MP 端 `PUT /api/tournament/{id}` 有 arm 逻辑
- **改动**（backend/src/routes/admin/mod.rs）：
  - 新增共享辅助函数 `sync_tournament_timers`：active → `arm_tournament` + `arm_reminder`；其他状态 → `disarm_tournament`（与 MP 端逻辑一致）
  - `create_official`：插入成功后同步定时器（当前强制 Draft，走 disarm 分支，为将来直接创建 active 赛事预留判断）
  - `update_official`：更新成功后重新读取文档，按最新 status/startTime/endTime 同步定时器——admin 端激活赛事即写入 Redis key、改期自动重算 TTL；**手动完结（status=completed）时补发 `EndNotice` 结束通知**（与 MP 端一致，内部有 Redis 防重旗标）
  - `delete_official`：删除成功后 `disarm_tournament` 清除两个定时器 key
- 验证：`cargo check` 通过，无新增告警；trial/prod 需重新构建部署生效

## 回退：v1.14 → v1.13

- 用户要求回退到 v1.13：撤销 v1.14 的 admin-web 赛事密码修复（列表注入 password、空密码过滤防护、编辑表单回填），代码恢复为 v1.13 状态


## v1.14 — 赛事报名简化：无需填写个人信息，仅订阅授权后直接报名

- **前端**（frontend/src/pages/tournament/register.vue）：
  - 移除报名表单的个人信息字段：选手姓名/性别/手机号/城市/地址/声明勾选，及对应脚本（prefill、onCityChange、declaration 校验、requiredFieldsFilled）
  - 页面仅保留：赛事名称横幅、密码输入框（仅赛事需要密码时显示，仍为必填）、订阅通知授权说明文字（i18n 新增 `tournament.notifyHint`，zh/en）
  - 点击"确认报名"→ 微信订阅消息授权弹窗 → 直接提交报名，成功后返回
  - 提交 body 简化为 `{ password, subscriptions }`
- **后端无需改动**：RegisterRequest 本就只接收 password + subscriptions（此前个人信息被 serde 静默忽略）；tournamentId/userId/nickname/avatar/gender 等仍由 register_for_tournament 从登录态写入 tournament_registrations
- 旧表单相关样式与 i18n key 保留未删（无害，后续可清理）


## v1.15 — 修复直达详情页时返回按钮失效（cannot navigate back at first page）

- **问题**：扫码报名（微信扫赛事二维码）或订阅消息通知入口进入赛事详情页时，详情页是页面栈的第一页，点击返回按钮 `uni.navigateBack` 报 `navigateBack:fail cannot navigate back at first page`，无法返回主页
- **改动**（frontend/src/pages/tournament/detail.vue）：
  - `goBack`：先判断 `getCurrentPages().length > 1`，有上一页则 navigateBack，栈底（直达入口）则 `uni.reLaunch` 回主页 `/pages/tabbar/index/index`
  - 删除赛事成功后的延时返回同样加栈底兜底（扫码入口下组织者删除赛事后会卡在原页）
- **改动**（frontend/src/pages/prize/fill.vue）：`goBack` 同样加栈底兜底——结束通知（EndNotice）订阅消息直达获奖资料补填页时存在同一问题，一并修复
- 验证：eslint 通过；小程序端需重新构建/发布生效

## v1.16 — App 端报名兼容微信订阅通知（不支持时自动跳过）

- **问题**：发布到 App 端后用户报名参赛失败——App 不支持微信订阅通知（`uni.requestSubscribeMessage` 仅微信小程序可用），报名提交路径被订阅授权步骤阻塞
- **排查结论**：当前源码中 `requestSubscribeMessage` 调用本已由 `// #ifdef MP-WEIXIN` 条件编译保护（已核验编译产物：app-plus 构建中该调用被剥离，mp-weixin 构建保留）；手机里安装的旧 App 包可能仍包含旧版报名页（含通知引导组件 NotificationGuide 的中间版本）。排查确认全项目仅 register.vue:150 一处调用该 API，后端报名接口对 subscriptions 全 false 不拒绝
- **改动**（frontend/src/pages/tournament/register.vue）：
  - 订阅授权调用加运行时兜底：`typeof uni.requestSubscribeMessage === 'function'` 才走授权流程，任何平台即使条件编译失效也不会阻塞报名
  - 「订阅通知授权说明」提示块加 `<!-- #ifdef MP-WEIXIN -->`，App/H5 端不再展示微信订阅提示
- **发布注意**：App 端需用当前源码重新打包自定义基座/发布包后重测报名（后端无需改动）；旧包如仍失败请提供 App 端 console 具体报错

## v1.17 — 修复 App 端报名 404（register/create 页面取不到赛事 id）

- **问题**：App 端点"确认报名"报 `请求失败: 404`；微信小程序端正常
- **根因**：register.vue 在 `onMounted` 里用 `getCurrentPages()[last].options?.id` 取赛事 id——**APP-PLUS 上该写法取不到 query 参数**，tournamentId 为空 → 报名 POST 打到 `.../api/tournament//register`（双斜杠）→ actix 路由不匹配返回默认 404（空响应体，且不写入请求日志——与实测一致：后端全天无该 POST、console 无后端 JSON message）
- **证据**：detail.vue（resolveTournamentId）与 prize/fill.vue 均以 `onLoad(options)` 参数取 id，App 上正常；编译产物核对确认 App 端执行的是 getCurrentPages 取参路径
- **改动**（frontend/src/pages/tournament/register.vue）：
  - 取参改为 `onLoad(options)` 优先（options.id || options.tournamentId），getCurrentPages().options 仅作兜底
  - `doSubmit` 增加 id 缺失防护：直接 toast `tournament.invalidId` 并返回，不再发畸形 URL
- **改动**（frontend/src/pages/tournament/create.vue）：editId 取参同样改为 `onLoad(options)` 优先 + 兜底（App 端"编辑赛事"存在同一 bug，顺带修复）
- 验证：eslint 通过；App 端需重新打包自定义基座重测（详情→报名→提交应成功，后端日志出现 POST /api/tournament/{id}/register）

## v1.18 — 修复 App 端创建赛事页滚动失效（选球场后/第二步无法滚动）

- **问题**：App 端创建赛事页——第一步选完球场后页面无法下滑，第二步从一开始就滚动不了
- **根因**（frontend/src/pages/tournament/create.vue 纯 CSS）：`.tournament-create` 用 `min-height: 100vh` 的 flex 布局 + `.form-scroll { flex: 1 }`——min-height 给不了 flex 容器确定高度，内容变高时容器随内容撑大，scroll-view 高度=内容高度，滚动失效。选球场后背景图出现、内容变高即触发；第二步内容本身较高、一进入就不滚动。项目内正常页面（register.vue/fill.vue）均用显式 `calc(100vh - X)` 高度，仅本页是 flex 写法
- **改动**：
  - `.tournament-create`：`min-height: 100vh` → `height: 100vh; overflow: hidden`（给 flex 容器确定高度）
  - `.form-scroll`：新增 `min-height: 0`（flex 子项默认 min-height:auto 会撑破容器，置 0 允许收缩形成滚动区域）
- 验证：eslint 通过；App 端重新打包自定义基座后重测两步滚动


## v1.19 — 赛事新增字段「获奖范围 awardRange」

- **后端**（models/tournament.rs + services/tournament_service.rs + routes/admin/mod.rs）：
  - `Tournament` 顶层新增 `award_range: Option<i32>`（serde rename `awardRange`，default + skip_serializing_if None；旧文档缺字段自动为 None，无需迁移）
  - `CreateTournamentRequest` / `UpdateTournamentRequest` 各加 `award_range: Option<i32>` —— admin-web 的 /admin/tournaments 创建/编辑共用同一 DTO，字段自动生效（update_official 走"序列化 DTO + 剥 null"通用 $set，awardRange 未传时原值不变）
  - 新增 `validate_award_range`（None 合法；Some 且 <=0 → ValidationError "获奖范围必须是正整数"），create_tournament / update_tournament / create_official / update_official 四处调用
  - update_tournament 的 $set 在 Some 时写入 awardRange
- **小程序**（pages/tournament/create.vue）：第 2 步"比赛配置"后新增"获奖范围"数字输入框（非必填）；form.awardRange（编辑回填 d.awardRange）；提交前校验正整数、payload 携带 awardRange（空值传 null）
- **admin-web**：TournamentForm.vue 第 2 步同款输入框 + 校验 + payload；TournamentTable.vue normalizeTournament/buildPayload 携带 awardRange（内联保存与编辑弹窗回填）
- **i18n**：zh-CN/en 新增 `awardRange` / `awardRangePlaceholder` / `errorAwardRange`
- 说明：本次仅"接收+校验+存储"，获奖名单按前 N 名筛选的消费逻辑未接入（后续需求）

## v1.20 — 修复 awardRange 编辑清空不生效（v1.19 bug）

- **根因**：前端清空获奖范围时提交 `awardRange: null`，后端 `Option<i32>` 无法区分「未传」与「传 null 清空」——MP 端 update_tournament 与 admin update_official 都把 null 当"不改动"，字段残留旧值
- **修复**（backend）：
  - `UpdateTournamentRequest.award_range` 改为双 Option `Option<Option<i32>>` + 自定义 `deserialize_double_option`：外层 None = 未传（不改动）；Some(None) = 显式 null（清空）；Some(Some(v)) = 设置值
  - `update_tournament` 与 admin `update_official`：Some(None) 时在更新文档中追加 `$unset { awardRange: "" }` 移除字段；Some(Some(v)) 校验正整数后 $set；admin 的"No fields to update"守卫对仅清空场景放行
  - `CreateTournamentRequest` 保持 Option<i32> 不变（创建无清空语义）
- 前端无需改动（两端本就提交 null 表示清空）

## v1.21 — admin-web 赛事表格：新增获奖范围列 + 操作列固定最右

- **TournamentTable.vue**：
  - 表格新增「获奖范围」列（密码列之后）：number 输入框，v-model.number 绑定 r.awardRange；空态 colspan 26 → 27；内联保存校验正整数（alert 提示）
  - 「操作」列（th/td）加 `.sticky-col`：`position: sticky; right: 0` + 背景/阴影/z-index，横向滚动时操作栏始终可见；表头交叉格 z-index 更高保证覆盖
  - normalizeTournament/buildPayload 的 awardRange 字段沿用 v1.19 已加内容


## v1.23 — 修复 admin-web 编辑弹窗第二步无法滚动/点不到保存

- global.scss `.modal-body` 增加 `min-height: 0`：flex 子项默认 min-height 为内容高度，第二步内容超高时容器不收缩、overflow 滚动不触发、footer（上一步/取消/保存按钮）被顶出弹窗可视区

## v1.25 — Edge 浏览器弹窗滚动兼容兜底

- global.scss `.modal-body` 追加 `max-height: calc(90vh - 200px)`：不依赖 flex 收缩的显式内容区上限（Edge 上 flex min-height 行为异常时仍可内部滚动），Chrome 等浏览器无副作用

## v1.26 — 获奖通知按 awardRange 限制前 N 名

- **通知过滤**（wechat_notify.rs）：EndNotice 发送前读取 `tournament.award_range`：
  - 已设置（如 5）→ 用 `prize_rank_by_openid`（比杆排名）计算获奖名单，**只通知前 N 名**；名单为空/逻辑库不可达时整个通知失败并记日志
  - 未设置 → 维持 v1.9 行为（通知所有参赛者）
- **补填页获奖判定**（routes/prize/mod.rs + pages/prize/fill.vue）：
  - `prize-status` 响应新增 `isWinner`（awardRange 设置时 rank <= N；未设置时所有有成绩者）与 `awardRange`
  - `submit_prize_info` 增加守卫：rank > N → 403 "您未进入获奖范围，无需填写收货信息"
  - fill.vue：获奖贺词/表单仅 `isWinner` 显示；未获奖视图区分两种提示（无成绩 noScoreHint / 有成绩但未进前 N 名 notInRange，i18n 新增 `prize.notInRange`，zh/en）

## v1.27 — EndNotice 按 awardRange 分流文案与跳转页

- awardRange **未设置**：结束通知文案仅为"比赛已结束"，点击跳转赛事详情页（与报名成功通知一致），不再引导填写收货信息
- awardRange **已设置**：维持 v1.26 行为（"比赛已结束，请填写收货信息领取奖品"+ 跳补填页，仅前 N 名收到）
- 单文件改动：backend/src/services/wechat_notify.rs 的 page 选择与 EndNotice payload

## v1.28 — 修复补填页获奖贺词插值不生效 + 手机号校验提示

- **贺词插值修复**（pages/prize/fill.vue）：本项目 vue-i18n 命名插值不生效（{name}/{rank} 原样输出），改为 `greetingText` computed 手动 replace 占位符（参考 my.vue formatRank 既有做法）；`noWinHint` 的 `{range}` 同步改为手动替换
- **校验提示**：handleSubmit 移除 `if (!canSubmit) return` 静默拦截，改为逐项显式校验——手机号非法 toast"请输入正确的手机号"、地址为空 toast"请填写收货地址"（按钮仍置灰但点击有明确提示）

## v1.29 — admin-web 新增英文说明/奖品字段（intro_en / prizes_en）

- **后端**（models/tournament.rs + 两处 create 字面量）：`Tournament` 新增 `intro_en`/`prizes_en`（serde default 空串）；`CreateTournamentRequest` 加同名字段、`UpdateTournamentRequest` 加 Option 字段（admin update_official 的剥 null 机制自动生效：不传不覆盖、空串清空、传值更新）；MP 端 update_tournament 的 $set 不含这两字段 → 小程序编辑不会误改英文内容
- **admin-web**：
  - TournamentForm.vue 第二步"奖品"后新增「比赛说明(英文)」「奖品(英文)」两个 textarea（maxlength 500/2000），form/fillForm/resetForm/payload 全链路携带（字段名 intro_en/prizes_en）
  - TournamentTable.vue 表格在"比赛说明""奖品"列右侧各加一列（cell-wide textarea），colspan 27→29，normalizeTournament/buildPayload 同步
- 说明：仅 admin-web 使用；小程序不传该字段，默认空串不受影响

## v1.30 — 我的页面隐藏"钻石"入口 + 金币卡片布局调整

- **pages/tabbar/personal/personal.vue**：
  - 资产卡片的"钻石"项（钻石图标 + PGM币余额）加 `v-if="false"` 临时隐藏（恢复时移除条件即可，脚本引用保留无副作用）
  - `.asset-item` 由 `flex: 1` 改为 `flex: 0 1 calc(50% - 12rpx)`——两个资产项时布局与原来一致，仅剩"金币"时保持原宽度靠左显示、不留缺角

## v1.31 — admin-web 新增赛事名称(英文)字段（name_en）

- **后端**（models/tournament.rs + 两处 create 字面量）：`Tournament` 新增 `name_en`（默认空串）；Create/UpdateTournamentRequest 加同名字段（admin 剥 null 更新机制自动生效）；小程序不传该字段不受影响
- **admin-web**：
  - TournamentForm.vue 第一步"赛事名称"下新增「赛事名称(英文)」输入框（maxlength 50），form/fillForm/resetForm/payload 全链路（字段名 name_en）
  - TournamentTable.vue 表格"名称"列右侧新增"名称(英文)"列，colspan 29→30，normalizeTournament/buildPayload 同步

## v1.32 — 补填页获奖贺词两行展示 + 新增必填"收件人"字段

- **贺词两行**（pages/prize/fill.vue + i18n）：第一行顶格"尊敬的选手："，第二行缩进"您于{name}大赛获得了第{rank}名，请及时确认您的收货地址。"；拆为 greetingLine1/greetingLine2 两个 key（zh/en），占位符手动替换（项目 vue-i18n 命名插值不生效）；只读视图的名次文案同步改为手动替换（此前 {rank} 会原样输出）
- **收件人字段**（全链路，DB 字段 `recipient`）：
  - 后端：models/prize_info.rs 的 PrizeInfo + SubmitPrizeInfoRequest 加 recipient；prize_service 的 find/submit/list 三处携带；routes/prize/mod.rs 提交校验（必填、≤30 字）
  - 前端：fill.vue 表单第一行新增"收件人"输入框（必填、置灰与 toast 校验同步）；只读视图展示；my.vue 列表展示
  - 旧数据无 recipient 字段，读取回退空串，无需迁移
- i18n 新增 prize.greetingLine1/greetingLine2/recipient/recipientPlaceholder/recipientRequired（zh/en）

## v1.33 — 获奖信息提交前增加二次确认弹窗

- pages/prize/fill.vue：校验通过后先弹 uni.showModal 二次确认（标题"确认提交"，内容"请确认所填信息准确无误，提交后将无法修改。"，确认/取消按钮）→ 确认后走 doSubmit 提交；取消则中止
- 提交逻辑拆分为 handleSubmit（校验 + 二次确认）与 doSubmit（实际请求），提交成功后仍弹"已提交，不可修改"提示
- i18n 新增 prize.confirmSubmitTitle / confirmSubmitContent（zh/en）

## v1.34 — awardRange 赛事结束通知改为全员发送，按是否获奖分流跳转

- **变更**（wechat_notify.rs）：v1.26 的"仅前 N 名通知"调整为**所有报名者都收结束通知**：
  - 前 N 名获奖者：文案"比赛已结束，请填写收货信息领取奖品"，点击跳补填页（不变）
  - 未进前 N 名：文案仅"比赛已结束"，点击跳赛事详情页
  - awardRange 未设置：全员"比赛已结束"+详情页（与 v1.27 一致，无回归）
- **实现**：移除循环内"非前 N 名 continue 跳过"过滤；`is_winner = winners.contains(openid)`；page 与 thing11 文案在循环内按用户分流计算

## v1.35 — 赛事到期自动结束修复：新增 30 秒 DB 轮询兜底 + reconcile 补发结束通知

- **问题**：赛事到了 endTime 后状态仍是 active、报名用户收不到结束通知；手动结束则正常。经服务器实测定位：Redis keyspace 事件配置正确、定时器键正常写入，但运行期间"键过期 → 状态翻转"不触发——监听器 pubsub 长连接被防火墙/NAT 半断开后"假死"（收不到事件且永不重连），过期事件一旦错过不会重放；且重启后的 reconcile 与每日扫描只静默改状态、不发通知
- **改动**（backend/src/services/tournament_expiry.rs）：
  - 新增 `sweep_overdue_tournaments` 核心 helper：查库（active 且 endTime<=now，投影 _id/endTime 按 endTime 升序）逐个条件翻转，翻转成功且未超宽限期时发送 EndNotice；日志关键字 `Tournament auto-completed by expiry poller`
  - 新增 `run_expiry_poller` 兜底任务：每 30 秒执行一次 sweep（MissedTickBehavior::Delay 防积压），不依赖 Redis 事件；`start()` 改为 spawn 监听器 + 轮询器（替代原每日扫描）
  - `reconcile` 改走 sweep helper——启动/重连补救路径也补发结束通知，消除"静默完成"路径
  - 删除 `run_daily_sweep` 及其 Redis 分布式锁（轮询负载全幂等，无需锁）
  - 新增宽限期环境变量 `TOURNAMENT_END_NOTICE_GRACE_SECS`（默认 0 = 始终通知；如 86400 抑制停机恢复后的陈旧补发，状态照常翻转）
- **改动**（backend/src/services/tournament_service.rs）：删除 `complete_expired_tournaments`（update_many 批量静默完成，已无调用方）；`complete_tournament_if_expired` 保持不变（条件更新保证 keyspace 事件/轮询/多实例并发下只有一方翻转+通知）
- **改动**（backend/src/main.rs）：到期管线启动注释更新
- **文档**：repowiki/zh/content/比赛系统/比赛调度器.md 同步（每日兜底扫描 → 30 秒轮询兜底、宽限期环境变量、排查日志关键字）
- 幂等性：翻转唯一由条件 update_one 保证；通知唯一由 wechat_notify 内 Redis SET NX 防重旗标保证；前端/管理端只读存储 status，无需改动
- 验证：cargo check 通过（仅 2 个既有无关警告）；部署后建议服务器实测——建 5 分钟到期的测试赛事验证 30 秒内自动结束+通知；`redis-cli CLIENT KILL TYPE pubsub` 模拟监听器假死验证轮询仍兜底

## v1.36 — 修复 v1.35 轮询兜底不发结束通知（宽限期默认值语义错误）

- **问题**：部署 v1.35 后，轮询兜底能正常把到期赛事翻转为已结束，但收不到微信结束通知（日志出现 `EndNotice suppressed: grace period exceeded`）
- **根因**：宽限期比较方向为"晚于宽限期才抑制"（`overdue_secs <= grace` 才发通知），而默认值写成了 0——轮询每 30 秒一拍，翻转时 `now - endTime` 几乎必然 ≥1 秒，`overdue_secs <= 0` 不成立 → 通知几乎全部被抑制。原意"默认始终通知"需要的是默认不限制
- **修复**（backend/src/services/tournament_expiry.rs）：`end_notice_grace_secs()` 默认值 `0` → `i64::MAX`（不限制 = 始终通知）；`TOURNAMENT_END_NOTICE_GRACE_SECS` 显式设置（如 86400）时仍按原语义抑制超过宽限期的陈旧补发
- **文档**：repowiki/zh/content/比赛系统/比赛调度器.md 宽限期两处描述同步为"默认不限制"
- 验证：cargo check 通过（仅 2 个既有无关警告）；部署后重测：到期赛事 30 秒内翻转 + 收到结束通知；未设置该环境变量即可，无需额外配置

## v1.37 — 修复结束通知模板字段不匹配（微信 47003）：场次填结束日期，名称并入温馨提示

- **问题**：v1.36 部署后翻转正常但微信仍返回 `errcode=47003 argument invalid! data.character_string2.value`——模板 33664（比赛结束通知）的"场次"是 character_string 类型，微信规定只支持数字/字母/符号、不支持中文，代码却把中文赛事名称塞进了该字段
- **改动**（backend/src/services/wechat_notify.rs，EndNotice payload）：
  - 场次 `character_string2`：赛事名称 → **结束日期**（`fmt_date(endTime)`，如 2026-09-30，数字+连字符合法）
  - 温馨提示 `thing11` 并入赛事名称（thing 类型支持中文 ≤20 字，先截名称再拼接防止关键文案被截断）：
    - 获奖：「{名称}」请填写收货信息领奖（名称预算 9 字）
    - 未获奖：「{名称}」比赛已结束（名称预算 13 字）
    - awardRange 未设置时全员走未获奖文案，与 v1.27/v1.34 行为一致
  - 注释同步：模板描述改为"比赛结束通知: 场次/赛事阶段/温馨提示"
- 其余字段（phrase12"已结束"）、StartRemind/报名成功模板 payload、跳转分流均不变
- 验证：cargo check 通过（仅 2 个既有无关警告）；服务器实测收到通知且无 47003——场次 2026-09-30 / 赛事阶段 已结束 / 温馨提示 「某赛事」比赛已结束（获奖者「某赛事」请填写收货信息领奖）；重测前清理 Redis 去重键 `DEL tournament:{库名}:end_notice_sent:{赛事id}`

## v1.38 — 修复头像上传失败但保存仍更新 logic.wxuser.customInfo.head

- **问题**：修改个人信息时重新选头像 → 上传失败 → 点保存，数据库中 logic.wxuser.customInfo.head 仍被改写（被本地临时路径/垃圾值覆盖）
- **根因**：① 前端选图压缩失败的回退把微信本地临时 URL 写入 `userInfo.avatar`，而 `isLocalTempPath` 只认 `http://tmp/` 前缀，其余临时路径形态被误判为服务端路径，保存时未经上传直接随 payload 提交；② 后端 `update_my_profile` 的 avatar 分支只判断 Option 不判断空/有效性；③ `sync_to_logic_db` 无条件 `$set customInfo.head = user.avatar`
- **改动**（frontend/src/pages/edit/editProfile.vue）：
  - 状态拆分：新增 `pendingAvatar`（本地临时路径，选图后仅用于预览与待上传）与 `newAvatarUrl`（本次上传成功的 URL）；`userInfo.avatar` 始终是服务端值——**只有上传成功才把 URL 写入页面状态**
  - `onChooseAvatar`/`onChooseAvatarApp`：压缩成功/失败（降级用原图）都只写 `pendingAvatar`，不再污染 `userInfo.avatar`
  - `handleSubmit`：默认 payload 不带 avatar；有待上传头像时先上传，成功 → 提交新 URL，失败 → toast + 新增 `revertAvatarPreview()` 回退预览为旧头像并中止本次保存（再点保存即只保存昵称等）；删除原 `else if (userInfo.value.avatar)` 重发旧 URL 分支；保存成功后才清空 `newAvatarUrl`（PUT 失败可重试）；上传中沿用 `isSubmitting` 禁用保存按钮
  - 删除已无调用方的 `isLocalTempPath`
- **改动**（backend/src/services/user_service.rs）：
  - `update_my_profile` avatar 分支加"有值且有效才更新"校验：空串/纯空白、`http://tmp/`、`wxfile://`、`file://` 等本地临时路径一律跳过（warn 日志），合法形态为 http(s) URL、`/` 开头的服务端相对路径或纯文件名（历史 FTP 存储格式）
  - `sync_to_logic_db`：`customInfo.head` 仅在 `user.avatar` 非空时 `$set`，防止空值覆盖 logic.wxuser.customInfo.head
- 验证：cargo check 通过（仅 2 个既有无关警告）；editProfile.vue 单独 eslint 仅 3 个既有硬编码中文报错（本次无新增）；小程序端回归：上传失败保存 → head 保持原值且预览回退；上传成功保存 → head 更新；不选头像保存 → head 不被改写

## v1.39 — 微信通知失败日志补充实际使用的模板 ID

- **背景**：替换结束通知模板 ID 后持续报 43101（user refuse to accept）。排查确认仓库全部 .env 与前端构建产物（dist/dev、dist/build 的 register.js）均为新模板 ID，前后端配置一致；43101 属微信一次性订阅授权状态问题（换模板后需重新报名授权；后端进程需重启加载新 .env）
- **改动**（backend/src/services/wechat_notify.rs，仅日志）：
  - `notify_registrants_inner` 解析模板后新增 info 日志 `微信通知: 使用模板`（含 template_id/kind/tournament_id）——换模板后一眼核对后端实际加载的模板 ID
  - 发送失败 warn 日志补充 `template_id` 字段——43101/47003 时直接可见后端用的哪个模板
- 验证：cargo check 通过（仅 2 个既有无关警告）
- 测试提示（43101 排查）：重启后端进程（.env 仅启动时读取）；清 Redis 去重键 `DEL tournament:{库名}:end_notice_sent:{赛事id}`（失败发送也占用旗标）；用户需对**新模板 ID** 重新报名授权（一次性订阅按模板逐个授权）；日志中 `微信通知: 使用模板` 的 template_id 应与报名页弹窗授权的模板一致

## v1.40 — 管理端赛事列表新增自定义背景字段（custom_bg / custom_bg_en）

- **需求**：赛事管理界面（admin-web）赛事列表中新增 2 个字符串字段 `custom_bg`、`custom_bg_en`，位于【获奖范围】列的下一列，列表内联填写并随保存入库；创建赛事时无这 2 个字段（仅列表可见）
- **改动**（backend）：
  - models/tournament.rs：`Tournament` 新增 `custom_bg` / `custom_bg_en`（String，serde default，旧文档缺字段自动为空串，无需迁移）；`UpdateTournamentRequest` 新增同名 Option 字段（admin update_official 剥 null 机制自动生效：不传不覆盖、传值更新、空串清空；MP 端 update_tournament 不传则不动）
  - CreateTournamentRequest **不加**这 2 个字段（创建时无）；两处 Tournament 字面量（tournament_service.rs create_tournament、admin/mod.rs create_official）初始化为空串
- **改动**（admin-web/src/components/TournamentTable.vue）：
  - 表头【获奖范围】后新增「自定义背景」「自定义背景(英文)」两列；表体对应两个 cell-wide 文本输入框（v-model r.customBg / r.customBgEn）；空态 colspan 30→32
  - normalizeTournament 新增 customBg/customBgEn（回退空串）；buildPayload 新增 custom_bg/custom_bg_en
  - TournamentForm（创建/编辑弹窗）不改动——需求明确这 2 个字段只在列表界面出现
- 验证：cargo check 通过（仅 2 个既有无关警告）；admin-web 无 TS/诊断报错；管理端列表保存后 MongoDB tournaments 文档出现 custom_bg/custom_bg_en 字段

## v1.41 — 修复管理端编辑赛事弹窗英文字段不回显

- **问题**：创建赛事时填写的【赛事名称(英文)】【比赛说明(英文)】【奖品(英文)】，在赛事列表点「编辑」后弹窗中全部为空
- **根因**（admin-web/src/components/TournamentForm.vue）：TournamentTable.vue 的 editRow 把 normalizeTournament 后的行对象（camelCase：nameEn/introEn/prizesEn）传给 TournamentForm，而 fillForm 按后端字段名（name_en/intro_en/prizes_en）读取 → 全部 undefined 回显为空；中文字段因行对象同名正常，掩盖了该问题
- **修复**：fillForm 中这 3 个英文字段改为两种命名兼容读取：`d.name_en || d.nameEn || ''`（introEn/prizesEn 同理）；模板/resetForm/payload 均不动
- 范围确认：编辑弹窗仍不显示 custom_bg/custom_bg_en（维持 v1.40 约定，该 2 字段仅赛事列表可见）
- 验证：TournamentForm.vue 无新增诊断；手工回归编辑回显与保存更新正常

## v1.42 — 修复球场排行榜不按球场区分（前端共享缓存 key 忽略 params）

- **问题**：小程序球场详情页「排行榜」所有球场显示同一份数据
- **根因**（frontend/src/utils/api/request.js）：`generateRequestKey(url, method, data)` 只把 `options.data` 纳入缓存 key，而 `getCourseLeaderboard`（business.js）走 `params` 传 sceneid → 所有球场排行榜请求共用同一条 key → 30 秒缓存窗口内从球场 A 进入球场 B 直接命中 A 的缓存（请求未发出），表现为"所有球场显示相同排行榜"；同 key 的 1000ms 限频也误伤跨球场快速切换
- **修复**：`generateRequestKey` 新增第 4 个参数 params，把 `options.params/query` 合并进缓存 key；`request()` 调用处传入 `options.params || options.query`——缓存/去重队列/限频共用该 key，全部随参数区分。business.js 与 course-info/index.vue 不改（传参正确）
- 后端无需改动：/api/holes/leaderboard 本就按 sceneid 必填过滤 logic.battles（sceneID+gameType=2 实时计算）
- 附带收益：所有用 params 的 GET（getHoleInfo/getCourseList 等）同类隐患一并修复
- 验证：request.js eslint 通过；手工回归——30 秒内切换球场显示各自数据、Network 可见两次带各自 sceneid 的请求、同球场 30 秒内重进仍走缓存、快速切换不再误报"请求过于频繁"
- 已知遗留（不在本期范围）：部分 battles 记录缺 sceneID 字段会被球场榜过滤（游戏端写入覆盖问题）；logic.battles 缺 {sceneID, gameType} 复合索引，数据量大时有性能隐患

## v1.43 — 修复 trial 环境球场排行榜头像空白（存储模式与文件位置错配）

- **问题**：trial（远程测试环境）球场介绍排行榜用户头像空白；本地与 prod 正常
- **根因**：头像文件实际全部位于 FTP 服务器（http://xiaoyigolf.cn/ftp/avatar/，曾用 migrate_avatars.sh 统一迁移），但 trial 配置 `UPLOAD_STORAGE_TYPE=local` → `get_full_avatar_url` 把纯文件名拼成 `/uploads/{file}` → trial 后端本地 uploads 目录无文件 → 前端拼出的 `https://xiaoyigolf.cn/pm/trial/uploads/{file}` 全部 404（13 个真实文件与 DefaultAvatar.PNG 实测均 404）；本地开发与 prod 均为 ftp 模式，走 `/api/user/avatar-proxy/{file}` 后端代理从 FTP 抓图 → 正常（该代理链路在 trial 实测 200 可用）
- **修复**：`backend/.env.trial` 与 `backend/build/trial/.env.trial` 的 `UPLOAD_STORAGE_TYPE` local → **ftp**（与 prod 对齐）
- 副作用：trial 新上传头像也写入 FTP，与 prod 行为一致，可接受
- 部署：远程服务器重启 trial 后端（.env 仅启动时读取）后验证——排行榜接口 avatar 返回 `/api/user/avatar-proxy/...` 且 curl 200；体验版小程序排行榜头像正常显示
- 未做（可另立项）：前端 image 无 @error 回退与默认头像静态资产缺失；微信后台 downloadFile 合法域名需含 xiaoyigolf.cn；后端按请求 host 生成绝对 URL 的长期优化

## v1.44 — 首页最近记录纳入赛事(gameType=5)记录 + 标签“锦标赛”改为“赛事”

- **问题**：首页「最近记录」看不到锦标赛（gameType=5）记录，即使它是最新的
- **根因**：
  - 前端 index.vue 的 `recentThreeRounds` computed 用 `[2,3].includes(gameType)` 过滤，把 gameType=5 排除；而 `loadBattleRecords` 请求的正是 `gameType='2,3,5'`（后端已返回 5，前端展示前被滤掉）。本地数据验证：logic.battles 有 601 条 gameType=5，wxusers.battleRecord 也正常引用
  - 标签“锦标赛”来自后端 msgpack 文本表（tab_text_5 → StringDB_CN = “锦标赛”），前端无硬编码
- **改动**（frontend/src/pages/tabbar/index/index.vue）：`recentThreeRounds` 过滤改为 `[2,3,5]`；更新 loadBattleRecords 注释
- **改动**（backend）：
  - constant.rs 新增 `GAME_TYPE_TOURNAMENT = 5`
  - user_info_service.rs 的 process_battle_record：gameType=5 时 gameTypeText 覆盖为“赛事”（英文环境 "Tournament"），其余类型仍走 msgpack 解析。该覆盖对首页最近记录、统计页、记分卡详情统一生效
- 验证：cargo check 通过（仅 2 个既有无关警告）；index.vue eslint 仅 2 个既有硬编码中文报错（ROLE_LABELS，无新增）；接口返回 gameType=5 记录且 gameTypeText="赛事"；首页最新赛事记录正常展示
- 未改动 msgpack 游戏数据表（避免被游戏端导出覆盖）

## v1.45 — 适配 DBText.msgpack 新格式（语言文案内嵌）

- **背景**：游戏文本表 DBText.msgpack.txt 格式变更——旧格式条目为 `{TextId, TextCn:<文本引用键>}`，需经 KeyMappingTable → StringDB 两级映射；新格式条目直接内嵌各语言文案 `{TextId, CN, TW, JA, EN, TL, VI, MS, AR, ES, RU, PL}`（backend/public/table 与 build/trial、build/prod 均已替换为新格式，共 1229 条）
- **问题**：原解析器 `DBTextInner` 只认 `TextCn`（必填），新格式反序列化失败 → `get_real_text` 全部回退 default，球场名/比赛类型/赛制/成就名/道具名/赛事选项等所有文本失效
- **改动**（backend/src/utils/msgpack_parser.rs）：
  - 新增 `get_db_text_entry`：把 DBText 条目动态解码为 msgpack map，兼容新旧格式
  - 新增 `value_to_text`（String/Binary/Ext(1) 兼容）与 `entry_lang_text`
  - `get_real_text` 重写：新格式按请求语言字段直取 → 回退 CN → 回退任意非空语言；条目无语言字段时走旧格式 TextCn → KeyMappingTable → StringDB 兼容回退；仍失败返回 default
  - `lang_to_suffix` 扩展：CN/TW/JA/EN/KO/TL/VI/MS/AR/ES/RU/PL，支持 locale 前缀匹配
  - `get_item_real_name` 简化为 DBItem.Name → get_real_text，删除已无调用方的 `get_db_text_content`、`get_key_mapping_index`、`DBTextInner`
  - 新增 2 个单元测试：新格式直取多语言文案、球场名多语言解析
- 验证：cargo check 通过（仅 2 个既有无关警告）；`cargo test msgpack_parser` **20 passed / 0 failed**（含新增 2 个测试）；旧格式回退路径保留，未升级文本表的部署仍可用
- 说明：v1.44 的 gameType=5 显示“赛事”为业务覆盖，不受本次影响

## v1.46 — msgpack_parser.rs 英文注释改为中文注释

- **范围**：backend/src/utils/msgpack_parser.rs 全部英文注释（模块文档、结构体/函数文档注释、行内注释、测试注释）翻译为中文
- **说明**：仅注释变更，不改动任何代码逻辑、字符串常量与类型定义；注释中的字段名/函数名/msgpack key（如 `DBText`、`TextCn`、`CN`、`get_real_text`）保留英文以对应实际数据
- 验证：cargo check 通过（仅 2 个既有无关警告）

## v1.47 — 修复 get_real_text 跨语言回退导致 Jenkins 测试失败（韩语应回退英文）

- **问题**：Jenkins 测试 `services::tournament_options::tests::test_options_korean_falls_back_to_english` 失败——期望 "Sunny"，实际返回 "晴朗"
- **根因**：v1.45 的 `get_real_text` 内部加入了“请求语言缺失 → 回退 CN → 回退任意非空语言”，而项目既有回退链由调用方控制（tournament_options::resolve_label：请求语言 → 英文 → key）。新 DBText 表无 KO 字段（只有 CN/TW/JA/EN/TL/VI/MS/AR/ES/RU/PL），请求 ko 时内部误回退中文，导致调用方认为命中、不再走英文回退
- **修复**（backend/src/utils/msgpack_parser.rs）：
  - `get_real_text` 去掉跨语言回退，只取 `lang_to_suffix(lang)` 对应语言字段；缺失时继续旧格式 TextCn → KeyMappingTable(lang) → StringDB；仍失败返回 default（恢复旧契约，由调用方决定回退链）
  - 模块文档补充“请求语言缺失时返回默认值，由调用方决定回退链”
  - 新增回归测试 `test_new_dbtext_missing_language_returns_default`：ko 请求返回 default，不得返回中文
- 验证：`cargo test` **85 passed / 0 failed / 3 ignored**（此前 83 passed / 1 failed）；`cargo test tournament_options` 3/3 通过；`cargo test msgpack_parser` 21/21 通过

## v1.48 — 管理端创建赛事新增“赛事海报”字段并在列表展示

- **需求**：admin-web 创建赛事时新增字符串字段「赛事海报」，位置在「背景图」下方；赛事列表页同步展示并支持内联编辑保存
- **改动**（backend）：
  - models/tournament.rs：`Tournament` 新增 `poster: String`（serde default，旧文档缺字段自动为空串，无需迁移）；`CreateTournamentRequest` 新增 `poster: String`（default，小程序端不传为空）；`UpdateTournamentRequest` 新增 `poster: Option<String>`（admin update_official 剥 null 机制自动生效：不传不覆盖、传值更新、空串清空）
  - tournament_service.rs / admin/mod.rs：两处 Tournament 字面量新增 `poster: req.poster`
- **改动**（admin-web）：
  - TournamentForm.vue：第一步「背景图」下方新增「赛事海报」输入框（maxlength 500）；form/resetForm/fillForm/payload 全链路携带 `poster`
  - TournamentTable.vue：表格「自定义背景(英文)」后新增「赛事海报」列（文本输入框，内联保存）；空态 colspan 32→33；normalizeTournament/buildPayload 同步
- 验证：cargo check 通过（仅 2 个既有无关警告）；cargo test 85 passed / 0 failed；admin-web 两个组件无诊断报错

## v1.49 — 小程序首次打开弹出官方赛事海报弹窗（支持多张左右滑动）

- **需求**：冷启动首次进入首页时，若存在带海报的官方赛事则弹出海报弹窗；多张海报左右滑动切换；点击「了解赛事详情」跳转对应赛事详情；切后台/切 tab 返回不重复弹；仅官方赛事弹
- **后端**：
  - main.rs：新增静态目录 `/championship` → `./public/championship`（与 `/Mapbg` 一致），海报文件放在 backend/public/championship 及 build/{trial,prod}/public/championship（已加 .gitkeep 占位）
  - TournamentListQuery 新增 `isOfficial` 查询参数；list_tournaments 构造 filter 时叠加 `isOfficial` 条件，避免前端分页过滤漏掉官方赛事
- **前端**：
  - 新增 `utils/posterPopup.js`：模块级“本次冷启动已检查”标志（切后台/切 tab 不重置，小程序进程重启自动复位；不用 storage，避免永久不弹）
  - 新增 `components/TournamentPosterPopup.vue`：自定义遮罩 + swiper 多海报左右滑动 + 指示点 + 关闭按钮 + 「了解赛事详情」按钮（z-index 1000，3:4 海报展示区）
  - `pages/tabbar/index/index.vue`：根节点外包 `page-root`；新增 `showPosterPopup/officialPosters` 状态与 `formatPosterUrl/loadOfficialPosters/handlePosterSelect`；onShow 首次进入时并行请求 `GET /tournament?status=active&isOfficial=true&page=1&limit=20`，筛选 `isOfficial && status==='active' && poster` 后弹窗；受 `tournamentVisible` 灰度门禁约束；扫码直达赛事详情/存在 pending_scene 时跳过，避免弹窗闪现
  - poster URL 规则：完整 http(s) 原样；`/` 开头拼 `getBaseHost()`；裸文件名拼 `getBaseHost() + '/championship/' + 文件名`
  - i18n 新增 `home.tournamentPosterCta`（了解赛事详情 / Learn More）
- 验证：cargo check 通过（仅 2 个既有无关警告）；cargo test 85 passed / 0 failed；新增/修改前端文件 eslint 无新增错误；admin-web 无诊断
- 部署说明：海报图片需放入 backend/public/championship 并同步到 build/trial/public/championship、build/prod/public/championship；release 环境若仍开启 TOURNAMENT_GRAY_RELEASE=true，普通用户列表为空，弹窗不会出现

## v1.50 — 修复赛事海报黑屏：海报改为后端 FTP 代理读取

- **问题**：小程序首启海报弹窗显示黑色（图片加载失败）
- **根因**：海报文件实际存放在 FTP 的 `championship` 目录（`http://xiaoyigolf.cn/ftp/championship/mp_11_cn.png`，实测 200 / image/png / 2.2MB），该 FTP HTTP 目录需要 basic auth；v1.49 前端拼的是后端 `/pm/trial/championship/...`（后端未部署该静态路由，且文件也不在 public/championship），因此 404/黑屏
- **修复**：
  - 后端新增 `GET /api/tournament/poster-proxy/{filename}`（routes/tournament/mod.rs）：校验裸文件名（拒绝 `/`、`\`、`..`），用 FTP_USER/FTP_PASS basic auth 从 FTP 抓取并流式转发（透传 Range/If-None-Match 与响应头），与头像代理 `/api/user/avatar-proxy/` 同理
  - utils/ftp.rs 新增 `get_ftp_poster_url(filename)`：拼 `http://{FTP_HTTP_HOST}/championship/{filename}`，完整 URL 原样返回；新增单元测试
  - 前端 formatPosterUrl 改为：完整 URL 原样；`/championship/xxx`、`championship/xxx`、裸文件名统一转 `${getBaseHost()}/api/tournament/poster-proxy/{filename}`；其它服务端相对路径保持原样
  - 回退 v1.49 的后端 `/championship` 静态目录及 public/championship 占位目录（海报实际在 FTP）
- 验证：cargo check 通过；cargo test 86 passed / 0 failed（新增 FTP URL 测试）；前端 eslint 仅 2 个既有 ROLE_LABELS 报错
- 注意：需重新部署/重启后端使新代理路由生效；海报文件名与 FTP `/var/www/html/ftp/championship/` 下的文件一致即可，无需再拷贝到 backend/public

## v1.51 — 修复海报弹窗上下黑边：按图片真实宽高比自适应高度

- **问题**：海报弹窗能加载图片，但上下有黑边（固定 3:4 展示区 + `aspectFit` 留白）
- **修复**（frontend/src/components/TournamentPosterPopup.vue）：
  - swiper 高度改为动态：图片 `@load` 返回原始 width/height，按面板宽度 620rpx 等比计算展示高度（`620 * height / width`），避免留白
  - 图片 `mode` 由 `aspectFit` 改为 `widthFix`，去掉固定高度与黑色背景
  - 「了解赛事详情」按钮移到 swiper 外，随当前 swiper 索引切换对应赛事（`@change` 维护 currentIndex，`currentItem` 供按钮跳转）
  - 每次打开弹窗重置索引与已测量高度
- 验证：组件 eslint 通过、无诊断报错；多张不同宽高比海报切换时高度随之变化，无黑边

## v1.52 — 新增赛事海报(英文) poster_en，并按小程序语言展示

- **需求**：admin-web 创建赛事新增「赛事海报(英文)」`poster_en`（位置在「赛事海报」下方），赛事列表同步展示；小程序中文环境用 `poster`、英文（及其它非中文）环境用 `poster_en`；当前语言海报缺失的赛事不进入弹窗（用户确认严格模式，不回退另一种语言）
- **后端**：
  - models/tournament.rs：`Tournament` 新增 `poster_en: String`（serde default）；`CreateTournamentRequest` 新增 `poster_en: String`；`UpdateTournamentRequest` 新增 `poster_en: Option<String>`（admin 剥 null 机制自动生效）
  - tournament_service.rs / admin/mod.rs：两处 Tournament 创建字面量新增 `poster_en: req.poster_en`
- **admin-web**：
  - TournamentForm.vue：「赛事海报」下方新增「赛事海报(英文)」输入框；form/resetForm/fillForm（兼容 `poster_en` 与 `posterEn`）/payload 全链路
  - TournamentTable.vue：「赛事海报」后新增「赛事海报(英文)」列（内联编辑）；空态 colspan 33→34；normalizeTournament/buildPayload 同步
- **小程序**（pages/tabbar/index/index.vue）：
  - `loadOfficialPosters` 通过 `getCurrentLocale()` 判断语言：`zh*` 用 `poster`，其余（en/ja/ko）用 `poster_en`
  - 严格过滤：当前语言海报为空则跳过该赛事；全部赛事都没有当前语言海报时不弹窗
  - 弹窗仍在冷启动后首次进入首页检查一次，切换语言后需下次冷启动生效
- 验证：cargo check 通过（仅 2 个既有无关警告）；cargo test 86 passed / 0 failed；admin-web 两个组件无诊断；index.vue eslint 仅 2 个既有 ROLE_LABELS 报错

## v1.53 — PGM Simulator App Store 搜索优化配套（版本号递增 + 包描述）

- **背景**：App Store 模糊搜索（pgm / PGM 模拟器 / golf simulator）搜不到 App。App Store 搜索索引的是 App Store Connect 元数据（名称/副标题/关键词/开发者名），**不读取 manifest.json**；本次仓库改动仅为配合提交新版本，真正的搜索优化需在 ASC 配置（详见实施说明）
- **改动**（frontend/src/manifest.json）：
  - `versionName` 1.0.31 → 1.0.32，`versionCode` 131 → 132（已上架 App 更新版本元数据需新建版本并上传构建）
  - `description` 改为 "PGM Golf Simulator - 高尔夫模拟器、记分卡、球场助手"（不参与 App Store 搜索，仅保持包信息一致）
- **ASC 建议配置**（手工）：英文名 "PGM Golf Simulator"、副标题 "Golf Simulator & Scorecard"、关键词覆盖 scorecard/swing/handicap 等；中文名 "PGM 高尔夫模拟器"、副标题 "高尔夫模拟器与记分卡"、关键词含 pgm/pgm模拟器/模拟器/高尔夫模拟器/记分卡等；确认目标国家/地区可售
- 验证：重新打包上传后，在目标地区 App Store 搜索 pgm / PGM 模拟器 / golf simulator；上线后 24-72 小时看 ASC「App 分析 → App Store 搜索」曝光

## v1.54 — 修复 Apple/Google 登录报错 `uni.logout is not a function`

- **问题**：iOS App 使用 Apple 登录报 `uni.logout is not a function`；卸载重装后可临时恢复
- **根因**：v1.8 为清理第三方 OAuth SDK 缓存调用了 `uni.logout`，但当前 App 运行时（HBuilderX 基座）**没有实现该 API**。`stores/user.js` 里的调用被 try/catch 静默吞掉，实际从未清掉 SDK 会话；`login.vue` 的 `clearAppleSdkSession` 没有保护——当 `uni.login` 返回 Apple SDK 缓存的过期 identityToken（`isJwtExpired` 命中）时会调用 `uni.logout`，同步抛 TypeError，登录中断。卸载重装清空了 SDK 授权缓存，不再走该分支，所以能临时恢复
- **修复**：
  - 新增 `frontend/src/utils/oauth.js` 的 `clearOAuthSdkSession(provider)`：优先 `uni.logout`（若存在），否则回退 HTML5+ `plus.oauth.getServices(...).logout()`（官方推荐的强制重新授权方式）；带 1.5s 超时、永不抛异常
  - `frontend/src/pages/login/login.vue`：`clearAppleSdkSession` / `clearGoogleSdkSession` 改用统一 helper，删除未保护的 `uni.logout`
  - `frontend/src/stores/user.js`：退出登录清理 google/apple/facebook 会话改用 helper（原 try/catch 中的 `uni.logout` 实际无效）
- 说明：按 Apple 官方行为，调用 logout 后再次授权弹窗可能不再显示姓名/邮箱（返回 null）；应用仍以 identityToken/authorizationCode 完成登录，不影响账号识别
- 验证：eslint 通过；需用重新打包的自定义基座/正式包在 iPhone 上回归：首次 Apple 登录、退出后再次 Apple 登录、缓存 token 过期后重试登录，均不再报 `uni.logout is not a function`；Android Google 登录同步回归

## v1.55 — 赛事文本字段长度与非法字符校验（admin-web + 小程序 + 后端）

- **规则**：赛事名称/赛事名称(英文) 最多 20 字符（必填/可选、单行）；比赛说明/比赛说明(英文)/奖品/奖品(英文) 最多 100 字符（可选、可多行）；仅允许中文、英文字母、数字、空格/换行、常用中英文标点，emoji、★♥✅ 等装饰符号和控制字符一律拒绝；字符数按 Unicode 码点计
- **后端**（services/tournament_service.rs）：
  - 新增 `TOURNAMENT_NAME_MAX_CHARS=20`、`TOURNAMENT_DESC_MAX_CHARS=100`、`is_allowed_tournament_char`、`validate_tournament_text_field`、`validate_tournament_create/update`
  - `create_tournament`、`update_tournament`、admin `create_official`、admin `update_official` 四处入口调用校验，返回 400 中文错误（不能为空/超过 N 个字符/包含非法字符）
  - 新增 5 个单元测试：长度边界、emoji/特殊符号、名称换行、必填
- **admin-web**：
  - 新增 `src/utils/tournamentText.js`（码点计数、白名单过滤、错误码）
  - `TournamentForm.vue`：名称/英文名称 20 上限与字符计数；四个说明/奖品 textarea 100 上限；输入时过滤非法字符，提交前逐字段校验
  - `TournamentTable.vue`：六个内联输入框加 maxlength 与输入过滤；保存前校验，错误提示带行号
- **小程序**：
  - 新增 `src/utils/tournamentText.js`（同规则）
  - `pages/tournament/create.vue`：名称 20、说明/奖品 100；输入时过滤；提交前校验；字符计数按码点
  - i18n 新增 errorNameTooLong/errorNameInvalid/errorIntroTooLong/errorIntroInvalid/errorPrizesTooLong/errorPrizesInvalid（zh/en）
- 验证：cargo test 91 passed / 0 failed；admin-web `npm run build` 成功；frontend 小程序构建成功；eslint 无新增错误
- 注意：历史赛事若名称超 20 字、说明/奖品超 100 字或含 emoji/特殊符号，下次在管理端/小程序保存时会被校验拦截，需要先手动清理

## v1.56 — 调整英文字段上限：赛事名称(英文) 100、比赛说明(英文)/奖品(英文) 300

- **需求**：赛事管理系统中文名称仍 20、比赛说明/奖品仍 100；英文字段放宽——`name_en` 100 字符，`intro_en`/`prizes_en` 300 字符
- **后端**（services/tournament_service.rs）：
  - 新增 `TOURNAMENT_NAME_EN_MAX_CHARS=100`、`TOURNAMENT_DESC_EN_MAX_CHARS=300`
  - `validate_tournament_create/update` 中 name_en/intro_en/prizes_en 使用新上限；中文 name/intro/prizes 保持 20/100
  - 新增英文字段长度边界单元测试
- **admin-web**：
  - `utils/tournamentText.js` 新增两个常量
  - `TournamentForm.vue`：赛事名称(英文) maxlength/校验/输入过滤改为 100；比赛说明(英文)/奖品(英文) 改为 300
  - `TournamentTable.vue`：nameEn 内联列上限 100；introEn/prizesEn 上限 300；保存校验同步
- **小程序**：不变（小程序创建页没有英文字段；中文 name/intro/prizes 仍 20/100）
- 验证：cargo test 92 passed / 0 failed；admin-web npm run build 成功；组件无诊断

## v1.57 — 修复 Android 13+ 从相册选择头像无反应（媒体权限缺失）

- **问题**：Android（Google Play 包）我的 → 修改个人信息 → 点击头像 → 从相册选择无任何反应；iOS 正常
- **根因**：App 端 `uni.chooseImage` 内部使用 `plus.gallery.pick({ system: false })`（DCloud 自带相册），需要相册读取权限；manifest 的 `excludePermissions` 排除了 `READ_MEDIA_IMAGES`，而 Android 13+（targetSdk 36）`READ_EXTERNAL_STORAGE` 已不覆盖图片访问 → 相册打开/读取失败；`fail` 回调只 console.error，用户无感知。检查最新 AAB 确认包含 READ_MEDIA_VISUAL_USER_SELECTED/READ_EXTERNAL_STORAGE，但不含 READ_MEDIA_IMAGES
- **修复**：
  - `frontend/src/manifest.json`：从 excludePermissions 移除 READ_MEDIA_IMAGES（保留 VIDEO 排除）；permissions 新增 READ_MEDIA_IMAGES、READ_MEDIA_VISUAL_USER_SELECTED、READ_EXTERNAL_STORAGE(maxSdkVersion=32)
  - `frontend/src/pages/edit/editProfile.vue`：App 端选择头像前先申请权限（Android 13+ READ_MEDIA_IMAGES，≤12 READ_EXTERNAL_STORAGE）；拒绝时弹窗提示并可跳转系统应用设置；`chooseImage` fail 增加 toast 提示
  - i18n 新增 edit.avatarPermissionTitle/avatarPermissionContent/openSettings/avatarPickFailed（zh/en）
- 验证：App 资源构建成功；editProfile eslint 仅既有硬编码昵称文案报错；需重新打包 Android APK/AAB（manifest 改动 wgt 热更新不生效），在 Android 13/14 真机回归相册授权、选图、拒绝后去设置、iOS 不受影响
- 注意：Google Play 可能要求「照片和视频权限」声明；若审核要求改用系统 Photo Picker，需另立项接入 UTS/系统选择器插件
