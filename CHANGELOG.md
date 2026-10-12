# Changelog

本仓库是 [dsh-phone-remote](https://github.com/wlxyxykj/dsh-phone-remote) 的 fork。
以下只记录本 fork 自己的变更；`1.0.0` 之前的版本历史见上游仓库。

## [1.0.0] — 2026-10-13

首个发布版本。基于官方 **v1.3.0**，修复两个问题。

### 修复

- **手机上的会话列表不会更新**
  原版只在手机页启动时拉取一次会话列表，之后手机新建的会话、电脑上新建/改名/删除的会话都不会出现，必须重开页面。
  现在：手机页顶部新增 **⟲ 刷新按钮**；插件在宿主侧监听 `session/created` / `session/updated` / `session/removed`，
  经已有 SSE 通道推给手机，自动静默刷新会话抽屉。监听器随手机连接挂载、最后一个连接断开即卸载，无常驻开销。

- **Agent 提问（`ask_user_question`）在手机上不可见**
  原版把它当普通工具调用渲染，手机用户不知道 Agent 正在等回答，会话卡住。
  现在：手机识别该工具并渲染提示卡片 —— 一次列出**全部问题**，每个问题带序号、每个选项带 `1. 2. 3.`，
  并指引用户直接打字回答（如 `1、2、看电影`）。

### 说明（设计取舍）

手机**不会**尝试提交桌面卡片的答案。DSH 的问答走内部前台瀑布，答案必须由持有该瀑布的客户端提交，独立页面无法持有；
实测三条路（普通消息回传 / `ctx.userQuestions.answer()` / 抢占 `user-questions/request`）均不可行。
因此本版只做「完整呈现 + 引导打字回答」，**不触碰任何 DSH 内部服务**，与官方插件的依赖面一致。

### 变更文件（相对官方 v1.3.0）

| 文件 | 改动 |
| --- | --- |
| `lib/app.html` | +84 行 |
| `lib/server.js` | +44 行 |
| `lib/index.js` | +1 行 |
| `package.json` | 名称改为 `dsh-phone-remote-plus`；版本 `1.3.0` → `1.0.0`（新包名独立版本线） |
| `cordis.patch.yml` | loader 条目 id/name 改为 `phone-remote-plus` |

### 兼容性

- 无新增配置项、无新增接口端点、无破坏性变更；原有链接、口令、白名单、面板、上传下载行为完全一致。
- 仍只依赖 `sessionController`；Node ≥ 20。
- loader 条目 id 为 `phone-remote-plus`，与官方的 `phone-remote` 不冲突，两者可共存。
- 状态文件沿用 `$DSH_HOME/dsh-phone-remote.json`（与官方共用）。
