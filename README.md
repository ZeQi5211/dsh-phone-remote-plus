# dsh-phone-remote-plus · 手机遥控（增强版）

> 用手机遥控 **DSH 桌面版**里跑着的 Agent：发指令、看实时输出、传文件、随时叫停。
> 本仓库是 [dsh-phone-remote](https://github.com/wlxyxykj/dsh-phone-remote)（作者 **tofar**）的 fork，
> 基于官方 **v1.3.0**，修复两个实际使用中遇到的问题。

| | 官方版 | 本增强版 |
| --- | --- | --- |
| 会话列表 | 打开页面时抓一次，之后不再更新 | **⟲ 手动刷新 + 会话增删改自动刷新** |
| Agent 提问（`ask_user_question`） | 手机上看不到，只显示成一条普通工具调用 | **渲染提示卡片：全部问题 + 序号选项 + 回复指引** |
| 其它行为 | — | 完全一致（没动任何原有功能） |

---

## 一、本版修复的两个问题

### 问题 1：手机上的会话列表不会更新

**现象**：手机打开遥控页面后，会话列表就固定在打开那一刻。

- 之后**在手机上新建**的会话，列表里看不到；
- 之后**在电脑上新建 / 改名 / 删除**的会话，列表里也不会变；
- 必须关掉标签页重新扫码，才能看到最新列表。

**原因**：原版只在页面启动时请求一次会话列表，之后没有任何机制感知会话变化。

**修复**：

1. 手机页顶部新增 **⟲ 刷新按钮**，点一下立即拉取最新列表；
2. 插件在宿主侧监听 `session/created` / `session/updated` / `session/removed` 事件，通过已有的 SSE 通道推给手机，手机自动静默刷新会话抽屉；
3. 监听器随手机连接挂载、最后一个连接断开即卸载，**无常驻开销**。

### 问题 2：Agent 提问时，手机看不到问题

**现象**：Agent 调用 `ask_user_question`（例如用 `grill-me` 这类技能向你提问）时：

- **电脑上**会正常弹出问答卡片；
- **手机上**只显示成一条可折叠的普通工具调用，你根本不知道 Agent 正在等回答 —— 会话就此卡住，直到你回到电脑上作答。

**原因**：原版把 `ask_user_question` 与其它工具一视同仁。

**修复**：手机识别该工具，渲染一张提示卡片，**一次性列出这次提问里的全部问题**：

```
💡 Agent 在等你回答

问题 1（晚饭）：晚饭吃什么？
  1. 火锅
  2. 寿司
  3. 家常菜

问题 2（饮品）：喝什么？
  1. 啤酒
  2. 果汁
  3. 白水

问题 3（饭后）：饭后做什么？
  1. 散步
  2. 看电影
  3. 休息

直接在下面对话里打字回答（按问题顺序，回选项序号即可，如：1、2、看电影），
答完结束对话，Agent 会继续往下进行。如上方问题未显示，就让 Agent 以文字形式重新发一遍。
```

要点：

- **多问题一轮全显**：一次调用带几个问题就显示几个（实测 5 个一轮完整显示）；
- **带序号**：每个问题前有「问题 N（header）」，每个选项前有 `1. 2. 3.`，所以你可以直接回 `1、2、看电影` 这种极简答案；
- **给出路**：底部说明「打字回答 → 答完结束对话 → Agent 继续」，并兜底提示「如问题未显示，就让 Agent 以文字形式重新发一遍」。

---

## 二、为什么桌面上的卡片不能在手机上点选提交（重要设计说明）

这是本版**刻意做的取舍**，写清楚以免后人重复踩坑。

DSH 的问答卡片走一条**内部前台瀑布**（`user-questions/request`），答案必须由**持有该瀑布的客户端**提交；手机是一个独立页面，无法持有它。实测三条路都不通：

1. **把 `{"answers":[...]}` 当普通消息发回**（steer / queue）——DSH 不认。它的问答折叠只接受 `source.kind === "user-question-reply"` 的消息并读其中的 `callId`，普通消息的 source 不是这个，**永不结算**。
2. **调用 `ctx.userQuestions.answer(agent, callId, answers)`**——只对「已经超时、转为 `continued`」的问题有效；对「正在前台等你的问题」直接返回 `false`（表现为「这个问题已不再可回答」）。
3. **在 `user-questions/request` 瀑布上抢占持有**——实测该事件在当前部署里**根本不带 `wait.callId`**（payload 只有 `questions / agent / signal`），按 callId 配对必然失败；且桌面 GUI 的应答器先注册，抢不到。

**结论**：只改插件（不动 DSH 本体）无法可靠地在手机侧提交答案。因此本版改为「把问题完整、带序号地呈现给用户，引导用户直接打字回答」。

**副作用（正面）**：这条实现路径**只依赖公开的 `sessionController` API**，不触碰 DSH 任何内部服务，与官方插件的依赖面完全一致 —— DSH 升级时不易失效。

---

## 三、安装

前提：电脑装有 **DSH 桌面版**；手机与其处于可互通的网络（同一 Wi-Fi 最省事）。

### 方式一：图形界面（推荐）

1. 打开 DSH 桌面版 → **设置** → **插件**
2. 点 **添加插件**
3. 粘贴本仓库地址：

   ```
   github:ZeQi5211/dsh-phone-remote-plus
   ```

   也可以填本地路径（插件目录，或打好的 `.tgz`）。
4. 点 **安装**，等依赖装完
5. **重启一次 DSH 桌面版** ← 必须
6. 重启后回到 **设置 → 插件**，确认 `dsh-phone-remote-plus` 已启用

> **为什么必须重启？** DSH 宿主进程按「模块 URL」缓存插件代码，进程活着时不会重新读盘上的插件模块。
> 安装/升级插件代码后必须重启；**改配置项**则热生效，不用重启。

### 方式二：命令行

```powershell
dsh plugin --profile desktop add github:ZeQi5211/dsh-phone-remote-plus
```

装完同样**重启一次 DSH**。

### 与官方版共存 / 替换

本插件的 loader 条目 id 是 `phone-remote-plus`，**与官方插件的 `phone-remote` 不冲突**，默认端口也都是 `8790`（后启动的会自动顺延到 8791…）。

- 想**只用增强版**：在 profile 的 `cordis.patch.yml` 里把官方那条禁用掉：

  ```yaml
  - id: phone-remote
    disabled: true
  - id: phone-remote-plus
    name: 'dsh-phone-remote-plus'
    config:
      host: '::'      # 需要 IPv6 双栈时设 '::'；默认 0.0.0.0
      port: 8790
  ```

- 想**两个都跑**：给其中一个换端口（注意手机遥控链接会跟着变）。

---

## 四、怎么用

### 电脑上打开控制台

浏览器打开 **http://127.0.0.1:8790/** —— 这个页面**只在本机能开**（故意的），里面有手机链接、二维码、口令和访问控制。

### 手机上三步上手

| 想做的事 | 操作 |
| --- | --- |
| 进入 | 手机连同一 Wi-Fi，用相机扫控制台上的二维码（链接里已带口令，不用手输） |
| 切换 / 新建会话 | 左上角 **☰** → 点会话；或底部「＋ 新建会话并接管」（可填工作目录） |
| 发指令 | 底部输入框输入，点 **↑** |
| 看输出 | 自动流式推送；工具调用是可展开的小卡片 |
| 叫停 | 运行中发送按钮变 **■**，点一下停止当前回合 |
| 刷新会话列表 | 顶部 **⟲** |
| 传文件给 Agent | 点 **📎** 选文件（单个 ≤ 25 MB） |
| 把产出下回手机 | 对 Agent 说「把这个发我手机」，手机页出现下载卡片，点一下保存 |

> 远程会话默认 `approvalPolicy: never`，手机发指令直接执行，不会弹授权确认。

---

## 五、配置

两种改法，**改配置不用重启**：

- **A. DSH 设置里改（推荐）**：设置 → 插件 → `dsh-phone-remote-plus`，会渲染出完整表单。
- **B. 改 profile 的 `cordis.patch.yml`**：

  ```yaml
  - id: phone-remote-plus
    name: 'dsh-phone-remote-plus'
    config:
      port: 8790
      cwd: 'D:\work\my-project'
      allowCancel: false
  ```

### 配置项

| 键 | 默认 | 说明 |
| --- | --- | --- |
| `enabled` | `true` | `false` 时插件完全不启动（不监听、不注册工具） |
| `host` | `0.0.0.0` | 监听地址。`0.0.0.0` = 所有网卡；`'::'` = IPv4+IPv6 双栈；`127.0.0.1` = 仅本机 |
| `port` | `8790` | 起始端口；被占用时自动向后找，最多试到 8799 |
| `token` | 空 | 访问口令。留空 = 复用已保存的，没有就随机生成 24 位并写入 `$DSH_HOME/dsh-phone-remote.json` |
| `cwd` | 空 | 手机上新建会话时的默认工作目录 |
| `agentPreset` | 空 | 手机上新建会话使用的 Agent preset（如 `standard`） |
| `approvalPolicy` | `never` | 远程会话授权策略。`never` = 不逐条询问（推荐）；`ask` = 跟随桌面端弹窗 |
| `allowCreate` | `true` | 是否允许手机新建会话 |
| `allowCancel` | `true` | 是否允许手机叫停回合 |
| `exposePanel` | `true` | 是否提供本机控制台 `/` |
| `allowFrom` | `[]` | IP 白名单。非空 = 由配置管理且面板不能改；留空则在面板上管理 |
| `strictAllowlist` | `false` | 严格模式：白名单连口令正确的设备也拦。默认 `false` = 口令优先 |

> 口令、白名单存在 `$DSH_HOME/dsh-phone-remote.json`，**与官方插件共用同一个状态文件**（因为服务同名）。
> 如果你同时跑官方版和增强版，它们会读到同一份口令 —— 这通常正是你想要的。

---

## 六、安全须知

- **链接/二维码就是钥匙**：别截图外发、别丢群里；
- 默认是**明文 HTTP**，链路中间人理论上可嗅探口令。要彻底解决请用加密隧道（Tailscale 等），或不用时把 `enabled` 设为 `false`；
- 默认行为是「**口令优先**」：带对口令的设备换任何网络都能连，不需要回电脑点「允许」；
- 完整的访问控制、白名单、Tailscale 与排障说明，见
  [官方 README](https://github.com/wlxyxykj/dsh-phone-remote#readme) —— 这些行为本版**完全未改动**，官方文档同样适用。

---

## 七、升级 / 卸载

DSH 目前不支持插件自动更新，升级要手动来：

```powershell
dsh plugin --profile desktop remove dsh-phone-remote-plus
dsh plugin --profile desktop add github:ZeQi5211/dsh-phone-remote-plus
```

改完代码或版本后**必须重启 DSH**。

---

## 八、工作原理与文件结构

```
┌──────────────┐   扫码/链接    ┌────────────────────┐   局域网/公网   ┌──────────────────────┐
│  手机浏览器   │ ─────────────▶ │  电脑控制台 (仅本机) │ ◀───────────── │  DSH 桌面版宿主进程    │
│  /m?t=<口令>  │ ◀───────────── │  127.0.0.1:8790    │                │  ctx.sessionController│
│  发指令/看输出 │   SSE 流式推送  │  二维码/白名单       │                │  （真正在跑 Agent）    │
└──────────────┘                └────────────────────┘                └──────────────────────┘
```

| 文件 | 作用 |
| --- | --- |
| `lib/index.js` | 插件入口：配置、口令、启动监听、注册工具、卸载清理 |
| `lib/server.js` | HTTP 路由、口令校验、IP 白名单、SSE、端口顺延、优雅停止；**本版新增 `session/*` 事件转发** |
| `lib/session.js` | 到 DSH 宿主服务的桥接（`sessionController`） |
| `lib/app.html` | 手机页（原生 JS，无前端依赖）；**本版新增刷新按钮与提问提示卡片** |
| `lib/panel.html` | 电脑控制台 |
| `lib/markdown.js` | 零依赖 Markdown 渲染器（只用 `textContent`，模型输出的 HTML 不会被当标签执行） |
| `lib/qr.js` | 零依赖 QR 编码器 |
| `lib/net.js` / `lib/store.js` | 局域网地址探测 / 状态文件读写 |
| `locale/*.json` | 插件在 DSH 里的显示名与描述 |

### 本版相对官方 v1.3.0 的代码改动

| 文件 | 改动 |
| --- | --- |
| `lib/app.html` | +84 行：`.notice` 样式、`parseAskArgs()`、`renderAskNotice()`、`handleEvent` 中识别 `ask_user_question`、`host-event` 监听、刷新按钮及绑定 |
| `lib/server.js` | +44 行：`ctx` 注入、`#withHostEvents()` / `#withoutHostEvents()`、`#stream` 内挂载/卸载 |
| `lib/index.js` | +1 行：把宿主 `ctx` 传给 `MobileServer` |

> 插件标识、包名、loader id 均已改为 `*-plus`；除此之外与官方 v1.3.0 逐行对应。

---

## 九、关于本仓库

- **这是什么**：[dsh-phone-remote](https://github.com/wlxyxykj/dsh-phone-remote) 的 fork，基于官方 **v1.3.0**，加入上述两个修复。除改动清单里列出的三个文件外，其余代码、文档、配置语义均来自官方版本。
- **原版作者**：**tofar**（[wlxyxykj](https://github.com/wlxyxykj)）。本仓库保留原 `LICENSE` 与署名。
- **维护**：[ZeQi5211](https://github.com/ZeQi5211)。
- **上游**：如果这两个修复对你有用，欢迎一并参考上游；若上游合并了等价实现，建议优先使用官方版本。

## License

MIT © tofar（原版）· 本 fork 同样以 MIT 发布
