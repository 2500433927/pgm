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
