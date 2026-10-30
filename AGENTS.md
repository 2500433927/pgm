# AGENTS.md

本文件是项目的全局工作指令，适用于仓库内所有任务（frontend/、backend/、admin-web/ 等）。

## 項目概述

### 简介
PGMMiniProgram 是一款跨平台高尔夫模拟器小程序，采用前后端分离架构：后端基于 Rust + Actix-web，提供高性能、类型安全的 REST API；前端基于 uni-app + Vue 3，统一构建微信小程序、H5 与 App（Android/iOS）。项目围绕用户认证、比赛系统、支付系统、数据统计与多语言国际化等核心能力展开，强调高并发、可观测性与可扩展性。

技术选型要点：
- 后端：Rust + actix-web 4 + MongoDB 驱动 2.8 + Redis + JWT + MessagePack（rmp-serde）
- 前端：uni-app + Vue 3 + Pinia + vue-i18n，目标平台为微信小程序、H5、App（Android/iOS）
- 多语言：后端 Accept-Language 在请求链路中传播，Redis 缓存键包含 language 防污染，三阶降级兜底链
- 接口约定：timestamp 字段必须为字符串；GET 请求默认 30 秒共享缓存，可用 skipCache 绕过；同一键 1000ms 内重复请求被拒绝

### 项目结构
仓库分为 backend 与 frontend 两大子工程，辅以文档与部署配置：
- backend：Actix-web 服务入口、路由注册、中间件、数据模型、业务服务、配置与工具模块
- frontend：uni-app 应用入口、页面与组件、状态管理、国际化、网络请求封装
- docs：数据库设计说明与逻辑库使用指南
- 根目录：Jenkins 流水线脚本、打包密码文件（仅用途说明，不展示值）

## 常见启动命令
- 后端
  - 启动服务：cargo run（需先配置 .env）
  - 导出 OpenAPI：cargo run -- --export-openapi openapi.json
- 前端
  - 微信小程序开发：npm run dev
  - H5 开发：npm run dev:h5
  - App 开发：npm run dev:app
  - 构建产物：对应 build 命令


## 开发规范
1. 代码注释尽量用中文
2. 每次修改完代码后，需要你返回一个版本号，**版本号必须通过读取项目根目录 `pro_chang.md` 中现有的最大版本号 +1 来确定**，同时在项目根目录的 `pro_chang.md` 中追加该版本号的变更记录（版本号 + 变更内容摘要），保持变更日志最新,同时在当前项目的根目录下创建一个pro_chang.md文档来记录每次版本的变更内容


## 约束事项

1. 每次新开会话时，先阅读 `repowiki\` 目录下的部分内容，优先看 - `repowiki\zh\content\快速开始.md`，以项目 wiki 为准执行任务，不凭猜测或重新推导已文档化的结论。wiki 内容与代码不一致时，以代码实际状态为准，并在交付说明中注明差异。
2. 每次新开会话时，我需要你优先阅读项目的开发文档记录`pro_chang.md`,然后后续在这个会话中要是有新增的功能，也按照版号顺序记录到当前文档里





