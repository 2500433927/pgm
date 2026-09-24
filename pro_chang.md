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

