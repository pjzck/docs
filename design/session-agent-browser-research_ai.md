# Session Agent 浏览器扩展能力 — 设计方案（P2）

> **版本**：v2（并入 Qoder / Codex 参考实现拆解 + Chrome 官方文档核实结论）
> **前置**：P0（窗口/截图/置顶）与 P1（鼠标/键盘）已完成并实测，见 `tools/session-agent/README.md`
> **标注约定**：📄 官方文档原文核实 ｜ 🔍 参考实现（读的是打包产物或产品自带文档）｜ ✅ 本机已核实 ｜ ❓ 待实测

## 1. 目标与非目标

### 目标
让 Session 0 的代理能对**挂着本扩展的那只 Chrome**：

1. **读页面**：指定标签页的 DOM / 无障碍树 / 文本 / 元素几何
2. **操作页面**：点击、输入、滚动、拖拽、改 DOM、执行 JS
3. **抓包调试**：请求列表 + 请求体 + 响应体 + 控制台日志 + 原始 CDP
4. **与本工具已有能力打通**：浏览器内用 CDP 注入输入，原生窗口（ImGui 等）继续用 `SendInput`

### 非目标

- 不做多浏览器支持（先只 Chrome；参考实现里 Codex 覆盖 Edge 等多浏览器，我们不需要）
- 不做 Playwright 那套完整定位 API（🔍 Codex 有 66 个 type 的完整面；我们只取够用的一层）
- 不接管用户浏览器之外的进程；不改 DSH 服务配置

## 2. 事实基础

### 2.1 官方文档核实（📄）

**Service Worker 生命周期** —— 这是本方案唯一的硬约束（原文见 `chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle`）：

| 规则 | 版本 |
|---|---|
| 空闲 **30 秒**终止；收到事件或调扩展 API 会重置 | 基础 |
| 单个请求/事件耗时超 **5 分钟**终止 | 基础 |
| **`fetch()` 响应超过 30 秒才到 → 终止** | 基础 |
| `connectNative()` 保活 SW；宿主崩了端口关，要在 `onDisconnect` 里重连 | Chrome 105 |
| **发消息**（long-lived messaging）保活；**"仅仅打开端口不再重置计时器"** | Chrome 114 |
| 活跃 WebSocket 收发消息重置空闲计时器 | Chrome 116 |
| **活跃的 `chrome.debugger` 会话本身保活 SW** | Chrome 118 |
| `alarms` 最小周期降到 30 秒 | Chrome 120 |
| 博客原话：native messaging 这类 API 提供 **strong keep-alive，会同时取消上面两个计时器** | `blog/longer-esw-lifetimes` |

→ 直接否掉"HTTP 长轮询"：`fetch` 响应超 30 秒即杀 SW。**数据通道必须建立在 native messaging 或 WebSocket 之上。**
→ CDP 抓包**不会**因 SW 回收而中断（118+ 起会话自身保活）。

**MV3 网络能力**（📄 `api/webRequest`）：`webRequestBlocking` 对普通扩展已不可用；事件为旁观性质，**拿不到响应体**（`onResponseStarted` 只给状态行与响应头）；**请求体可得**（`onBeforeRequest` + `extraInfoSpec:['requestBody']`）。

**native messaging 尺寸**（📄 `concepts/native-messaging`）：host → Chrome **单条 1 MB**；Chrome → host **64 MiB**；4 字节 LE 长度前缀，长度不得超过 `1024*1024`。

**`chrome.scripting`**（📄 `api/scripting`）：需 `"scripting"` + host 权限（`host_permissions` 或 `activeTab`）；`ExecutionWorld` 为 `ISOLATED`（默认）/ `MAIN`；`executeScript({target:{tabId,allFrames}})`。

**`chrome.debugger`**（📄 `api/debugger`）：需 `"debugger"` 权限；`attach/detach/sendCommand/onEvent/onDetach/getTargets`；`onDetach` 触发条件原文是"标签页被关闭 **或对该标签页调用了 Chrome DevTools**"；`DetachReason` 枚举 `target_closed` / `canceled_by_user`；OOPIF 子目标用 `sessionId` + `Target.attachedToTarget`。
（提示条文案我**没在文档里找到**，只由 `canceled_by_user` 间接推断有用户可取消的入口 —— 标为 🔍，不当结论。）

### 2.2 CDP 方法核实（✅ 逐个查过 tot 协议 JSON，58 个域）

| 用途 | 已核实存在 |
|---|---|
| 抓包 | `Network.enable` `getResponseBody` `getRequestPostData` `setBlockedURLs`；事件 `requestWillBeSent` `responseReceived` `loadingFinished` `loadingFailed` `webSocketFrameReceived` |
| 取 DOM | `DOM.getDocument` `querySelector` `querySelectorAll` `getOuterHTML` `getBoxModel` `resolveNode` `setAttributeValue` `focus` |
| 执行 JS | `Runtime.evaluate` `callFunctionOn` `getProperties`（在 `js_protocol.json`） |
| 页面 | `Page.navigate` `captureScreenshot` `getLayoutMetrics` `reload` `getFrameTree` `setInterceptFileChooserDialog` |
| **注入输入** | **`Input.dispatchMouseEvent` `dispatchKeyEvent` `insertText` `dispatchDragEvent` `synthesizeScrollGesture` `synthesizeTapGesture` `setIgnoreInputEvents`** |
| 拦截改写 | `Fetch.enable` `getResponseBody` `takeResponseBodyAsStream` `fulfillRequest` `continueRequest`；事件 `requestPaused` |
| 子目标 | `Target.setAutoAttach` `attachToTarget` `getTargets` |
| 视口 | `Emulation.setDeviceMetricsOverride` |

❌ **`Network.getResponseBodyForInterception` 不存在** —— 拦截取体要用 `Fetch.getResponseBody`（大响应另有 `Fetch.takeResponseBodyAsStream`）。

> **关键结论**：CDP `Input` 域的注入使用**视口坐标**，在浏览器内部完成。
> 这解释了为什么两家参考实现从不处理"DOM rect → 操作系统物理像素"的换算 —— 它们都在浏览器里注入输入。
> **对我们的意义**：浏览器内的点击/打字走 CDP `Input.dispatch*`；`SendInput` 那套继续服务原生/ImGui 窗口。原先列为"必须先验证"的坐标换算问题，**只在坚持用 SendInput 点网页时才存在**。

### 2.3 参考实现拆解（🔍 读的是打包压缩产物 + 产品自带文档，非源码）

#### Qoder（扩展装在 Chrome，v1.7.5）

```text
① 发现：扩展 --connectNative--> host.bat --> host.js
        扫各产品 userData 下的 relay-port.json → 回 {port,pid,clients} → 退出
② 探活：fetch(`http://127.0.0.1:${port}`, {method:"HEAD", mode:"no-cors"})   500ms 超时
        fetch(`http://127.0.0.1:${port}/app/info`)                          拿 App 信息
③ 数据：new WebSocket(`ws://127.0.0.1:${port}/extension/v2`)   主通道（按 URL 路径做协议版本化）
        new WebSocket(`ws://127.0.0.1:${port}/extension`)      遗留 V1 状态桥
④ 保活：常驻 connectNative"native watch" + ping/pong + 退避重连 + fallback discovery
        `relay-keepalive`，1 秒间隔 ping 到连上为止
⑤ 身份：storage 里存随机 UUID 的 browserClientId
```
- host 是 `.bat`，内容是 `set ELECTRON_RUN_AS_NODE=1` + **拿自家 Electron 当 Node** 跑 `host.js`（省一个 Node 运行时）
- 能力：**硬编码 CDP 编排** —— `attach({tabId},"1.3")` → `Page.enable`+`Runtime.enable`+`Network.enable`；
  `Runtime.evaluate {expression, returnByValue:true, awaitPromise:true}`（表达式包成 `(expr)`，读 `exceptionDetails` 判错）；
  `Page.captureScreenshot {clip,captureBeyondViewport,fromSurface:true}`（**元素级截图**）；
  attach 前先 `detach()` 清残留、按 `"Cannot attach"` 判被占；`onDetach` 清状态
- 权限：`debugger`+`scripting`+`tabs`+`nativeMessaging`+`sidePanel`，`host_permissions` 含 `<all_urls>`
- content script 三件：`page-bridge`、**`accessibility-tree`（抽无障碍树）**、`visual-indicator`（让用户看见正在被操作）
- 另有一条独立能力：自家浏览器是 Chromium，profile 里有 `DevToolsActivePort`（走 CDP 控制自己那套浏览器）

#### Codex / ChatGPT（扩展装在 Edge，v1.26.901.11451）

```text
扩展 --connectNative--> extension-host.exe(Rust) --命名管道 \\.\pipe\codex-browser-use--> Codex App
                                    └─ 同时自建本地 WebSocket 代理（allowed origin + 端口占用时用可用端口）
协议：JSON-RPC 2.0 + 声明式协商（schemaVersion 2 / supportedMethods / supportedProtocolVersions）
方法名形如 codexRuntime/hello、codexRuntime/ensure、codexRuntime/resume
```
- 扩展是**通用 CDP 直通桥**：`{target, method, commandParams}` → `chrome.debugger.sendCommand`，
  并把 `onEvent` / `onDetach` **全量转发**给 App；attach 支持 `{tabId}` 与 `{targetId}`（OOPIF）；停止时全部 detach
- 重连：退避 + **`chrome.alarms` 重试** + 多 host 名 fallback（`selectFallbackApplication`）
- 自带文档（`~/.codex/plugins/cache/openai-bundled/chrome/latest/docs/`，共 22 篇）里有完整 API 面与策略：
  - **`api.json`**：root = `Agent` → `Browsers{get,getDefault,getForUrl,list}` → `Browser{browserId,tabs,user,…}`
    → `Tab` 上**并列四套交互 API**：`ax`（无障碍树）/ `cua`（坐标视觉）/ **`dom_cua`（DOM 取坐标 + 坐标动作）** / `playwright`（结构化定位），
    另有 `content` / `clipboard` / **`dev.logs`（控制台日志）** / `capabilities.get("cdp")`
  - **`capabilities/tab/cdp.md`**：`send(method,params,{target})` + `readEvents({afterSequence,limit≤1000,methods,target,timeoutMs})`
    → `{cursor, events[{method,params,sequence,source}], hasMore, truncated}`；**`truncated` 显式告知旧事件已被丢弃**；
    并要求"若通过 CDP 改了页面且留在那里，**最终答复必须告诉用户改了什么**"
  - **`tab-claiming-chrome.md`**：认领用户已开标签页 —— `openTabs()` 后用 **id + title + url 快照**比对，
    **失败就 fail closed**（"report that it is unavailable; **do not silently claim or open a different tab**"），
    "**Do not guess tab ids. Only claim ids that came from the current `openTabs()` result.**"
  - **`confirmations.md`**：浏览器动作的四档确认摩擦度（见 §5.2）

#### 两者对比

| | Qoder | Codex |
|---|---|---|
| 数据通道 | 扩展**自己**开 WS 直连 App 端口 | 扩展 → host（native messaging）→ **命名管道** + host 自建 WS 代理 |
| CDP 用法 | 硬编码编排 | **通用直通**，方法名由 App 传 |
| 事件下发 | WS 推送 | 转发 + **游标缓冲 + `truncated`** |
| 页面读取 | CDP DOM + a11y content script | **a11y 一等公民** + playwright 定位 + `dom_cua` |
| 协议版本化 | WS URL 路径（`/extension` vs `/extension/v2`） | JSON-RPC + `supportedMethods`/版本协商 |
| 共同点 | `host_permissions:<all_urls>`、都声明 `debugger`、都做可视化指示、**都不用 webRequest** | 同 |

## 3. 架构（v2）

### 3.1 双通道

```text
Session 0 CLI ──TCP(现有)──> Session 1 代理（Python，现有）
                                   ▲
                                   │ TCP（现有协议，代理作服务端）
                                   │
                            bridge.py（新增，Python）
                                   ▲
                                   │ native messaging（stdio，4 字节 LE + JSON）
                                   │ 常驻 connectNative 端口
                            扩展 service worker
                                   │
                                   └── chrome.debugger / chrome.scripting / chrome.tabs
```

- **native messaging 常驻端口**承担两件事：**保活 SW**（📄 唯一的 strong keep-alive）与**承载命令/响应**
- **测量发现**：数据**不需要**再开 WebSocket 直连 —— 见 §3.2
- bridge 与代理之间沿用**现有 TCP 协议**（line-delimited JSON + token），不引入新协议

### 3.2 数据通道选型（v2 的决定）

| 方案 | 依据 | 结论 |
|---|---|---|
| **A. 常驻 native messaging 端口 + 事件游标分页** | 🔍 Codex 实测形态；📄 1 MB 上限对"按游标拉 ≤1000 条事件"不构成问题；📄 最强的 SW 保活 | ✅ **采用** |
| B. 扩展直连 `ws://127.0.0.1` 给代理 | 🔍 Qoder 形态；无尺寸上限、可推送 | ⏸ 备选（要在 Python 里实现 WS 服务端；且 `fetch` 30 秒规则说明轮询不可行、WS 才可行）。若将来事件吞吐或延迟成为瓶颈再上 |
| C. native messaging 一次一问一答（现状用法） | 已有 | ❌ 不可用于抓包：SW 会被回收、CDP 事件无处接收 |

**A 的关键设计（抄 Codex 的游标模型）**：

```text
读事件：chrome 侧维护环形缓冲（每条带自增 sequence）+ 游标
  readEvents({afterSequence, limit≤1000, methods[], timeoutMs})
    → {cursor, events[], hasMore, truncated}
1 MB 上限的应对：limit 分页 + 单条 params 超限时截断并标 truncated_body + 记录原始字节数
```

### 3.3 扩展的定位：做"直通桥"，不做"业务编排"

抄 Codex：扩展只提供 `send(method, params, {target})` + `readEvents(...)` + 少量自有能力（tab 枚举、图标/侧栏 UI）。
**"该启用哪些 CDP 域、取什么数据"全部由代理侧决定** —— 这样新增 CDP 用法不需要改扩展、不需要用户重新加载扩展。

对比：Qoder 把编排写死在扩展里（改行为就得发新版扩展）。

### 3.4 SW 保活策略

| 措施 | 依据 |
|---|---|
| 扩展启动即 `connectNative` 建常驻端口 | 📄 Chrome 105 |
| 每 **20 秒**发一次心跳消息（不依赖"仅打开端口"） | 📄 Chrome 114 "opening a port no longer resets the timers" |
| `onDisconnect` 里立即重连（退避：1s→2s→5s→15s） | 📄 文档明确要求 |
| 30 秒 `alarms` 兜底看门狗（SW 被系统回收后拉起） | 📄 Chrome 120 |
| CDP 抓包期间不额外保活（debugger 会话自身保活） | 📄 Chrome 118 |
| 协议里带 `protocol_version` + `supported_methods` | 🔍 Codex 声明式协商 |

## 4. 能力面（命令设计）

### 4.1 命令清单

```text
chrome tabs   [--match <url|title片段>] [--json]        列标签页：id/标题/url/窗口/是否前台
chrome select --tab <id> | --active | --match <片段>    选定目标页（fail-closed，见 4.3）
chrome read   --selector <css> [--mode ax|dom|text|html] [--box]   读页面（默认 ax）
chrome eval   --expr <js> [--main-world] [--timeout-ms]            执行 JS（返回 JSON 化）
chrome click  --selector <css> [--native] [--button] [--count]     点击（默认 CDP Input）
chrome type   --selector <css> --text <t> [--native]               输入（默认 CDP Input.insertText）
chrome key    --name <键名|组合> [--native]                        按键（默认 CDP Input.dispatchKeyEvent）
chrome scroll --selector <css> | --x --y --dy <px>                 滚动（CDP synthesizeScrollGesture）
chrome drag   --from <sel> --to <sel>                              拖拽（CDP dispatchDragEvent）
chrome shot   [--selector <css>] [--out <png>]                     整页或**元素级**截图（CDP clip）
chrome logs   [--since <seq>] [--level]                            控制台日志（🔍 Codex 的 dev.logs 思路）
chrome net    start|events|body|stop [--filter <正则>] [--with-bodies]
chrome cdp    --method <Domain.method> [--params <json>] [--target <id>]   原始直通（逃生舱）
```

### 4.2 与现有 P0/P1 的关系

| 场景 | 走哪条 |
|---|---|
| 网页内点击/输入/滚动/拖拽 | **CDP `Input.dispatch*`**（视口坐标、浏览器内部、事件 trusted） |
| 原生窗口（ImGui 等） | 现有 `SendInput`（`move`/`click`/`drag`/`wheel`/`type`/`key`） |
| 兜底 | `--native` 强制 SendInput 点网页（当 CDP 注入被页面拦或需要真实 OS 输入时） |

**这条分工是本方案相对两个参考实现的核心差异点**：它们只能在浏览器里干活；我们能同时覆盖浏览器**和**原生窗口。

### 4.3 "指定页面"的语义（抄 🔍 `tab-claiming-chrome.md`）

1. `chrome tabs` 返回当前 `openTabs` 快照（含 `tabId` + `title` + `url` + `windowId`）
2. 选定目标时**必须带快照三元组**校验（`tabId` + `title` + `url`）；不一致则 **fail closed** 报 `TAB_CHANGED`，
   **绝不静默换到另一个标签页**
3. **禁止猜 id**：只接受来自最近一次 `chrome tabs` 结果的 id
4. 支持"用户显式指定"：popup 里加一个「把当前标签页交给代理」按钮，把快照写进运行时文件，代理优先用它

## 5. 安全、权限与确认

### 5.1 权限与白名单

| 项 | 决定 |
|---|---|
| manifest 权限新增 | `scripting`、`tabs`、`tabGroups`（可选）、`debugger`（**只在启用抓包时才申请**，见下）、`webNavigation`（可选） |
| `host_permissions` | ⚠️ **MV3 里是静态的，运行时不能悄悄扩权**（`optional_host_permissions` + `permissions.request()` 需要用户手势）。所以：**manifest 给宽**（参考实现都是 `<all_urls>`），**白名单在 app 层拦** —— 代理/扩展按 URL 匹配决定"这个标签页碰不碰"，配置化、可随时收紧 |
| `debugger` 权限 | 声明在 manifest 里即生效（会改变安装/更新时的权限提示）。**抓包/改 DOM 之外的功能不依赖它**，方案上把 CDP 相关命令标为"需要 debugger"，其余（读 DOM、点击、输入）走 `scripting` + `Input` 之外的路径…… ❓此处需实测：CDP `Input` 依赖 debugger，若不想装 debugger，浏览器内输入只能退回 `--native` |

### 5.2 确认策略（对标 🔍 `confirmations.md`）

四档摩擦度，写进代理侧策略并在命令上落成闸门：

| 档 | 含义 | 本工具的落法 |
|---|---|---|
| ① 必须用户自己来 | 代理只提议 | 登录、验证码、支付密码；代理报 `HANDOFF_REQUIRED` |
| ② 动作前必须确认 | 预授权也无效 | 提交表单/发送消息/上传文件/删数据 → 需要 `--confirm` 显式传入，否则报 `CONFIRM_REQUIRED` |
| ③ 预授权可放行 | 否则等同② | 导航、登录（若"去 xyz.com"已隐含）、文件重命名 |
| ④ 无需确认 | — | 读页面、截图、抓包（只读） |

两条边界照抄：**用户亲自打的字算有效意图**；**用户粘贴/引用的第三方内容绝不算授权**；**把敏感数据敲进表单 = 传输**，必须确认。

### 5.3 诚实性要求（与本项目既有原则一致）

- **禁止伪造**：读不到就报错（`BLANK_CAPTURE` 式处理），不返回伪造的成功
- **显式告知数据缺失**：事件被丢弃 → `truncated: true`；响应体超限 → `truncated_body: true` + 原始字节数
- **改了就报告**：通过 CDP/JS 改了 DOM 或浏览器状态且留在那里 → 回执里列出改了哪些（🔍 Codex 的硬要求）
- **fail closed**：目标标签页对不上就报错，不猜、不换（🔍 同上）

## 6. 分期实施

| 期 | 内容 | 依赖 | 产出 |
|---|---|---|---|
| **S0** | **spike**：独立临时 profile 起 Chrome 自测（见 §7） | 无 | 实测结论，回填本文「❓」处 |
| **P2-a** | 通路：`bridge.py` 常驻 native 端口 + 心跳 + 重连；扩展侧 `chrome tabs/select`；代理侧新命令族骨架 | S0 | 能列标签页并选定目标（fail-closed） |
| **P2-b** | 读页面：`read`（ax 优先）/`eval`/`shot --selector` | P2-a | 拿到真实页面内容与元素截图 |
| **P2-c** | 操作页面：`click/type/key/scroll/drag`（CDP Input） | P2-b | 真机测试可点可填 |
| **P2-d** | 抓包：`net start/events/body/stop`（游标分页 + `truncated`） | S0 | 请求/请求体/响应体可查 |
| **P2-e** | 日志与逃生舱：`logs`、`cdp` 直通 | P2-a | 覆盖未预料的调试需求 |
| **P2-f** | 安全闸门：白名单、`--confirm`、`HANDOFF_REQUIRED` | P2-c | 风险动作受控 |
| **P2-g** | popup 侧栏 UI（🔍 两家都做）：连接状态、可视化指示"正在被操作"、把当前标签页交给代理 | P2-a | 用户可见可控 |

## 7. 待实测清单与 spike 设计

**spike 做法**：`chrome.exe --user-data-dir=<临时> --load-extension=<spike目录> --no-first-run http://127.0.0.1:<port>/test.html`，
测试页主动发 fetch/XHR/滚动/点击，扩展侧执行探针，结果 POST 回本地端点。**不碰用户日常 Chrome。**

| # | 要回答的 | 判据 |
|---|---|---|
| A | **CDP `Input.dispatchMouseEvent` 在页面里是不是 `isTrusted === true`** | 页面上监听事件读 `event.isTrusted`。**这条决定浏览器内点击走 CDP 还是必须退回 SendInput** |
| B | 常驻 `connectNative` + 20s 心跳下，SW 能否 10 分钟不重启 | SW 内自增计数 + `performance.now()`，重启即计数归零 |
| C | CDP 事件游标分页的真实吞吐与 `truncated` 行为 | 测试页打 500 个请求，看能取回多少、何时 truncated |
| D | 响应体尺寸分布（决定截断策略阈值） | 采集真实接口的 body 字节数直方图 |
| E | `Input.dispatchKeyEvent` 输入中文/emoji 是否可行（对照 P1 已验证的 `KEYEVENTF_UNICODE`） | 输入后回读 `input.value` |
| F | 元素 rect（CDP `getBoxModel`）与 CDP `clip` 截图是否自洽 | 元素截图内容与 DOM 结构比对 |
| G | 不声明 `debugger` 权限时，能覆盖多少能力（读 DOM/点击/输入） | 分别验证 `scripting` + `Input` 的可用性 |

## 8. 参考实现索引（本机路径，便于复查）

| 对象 | 路径 |
|---|---|
| Qoder 扩展（Chrome） | `%LOCALAPPDATA%\Google\Chrome\User Data\Default\Extensions\gblapfbnbicdckfhkllcnfleiemhmgeb\1.7.5_0\` |
| Qoder host | `%APPDATA%\QoderWork CN\native-messaging-host.bat` → `E:\QooderWorkCN\QoderWork CN\resources\native-messaging-host\host.js` |
| Qoder 自家 Chromium profile | `%APPDATA%\QoderWork CN\`（含 `DevToolsActivePort`） |
| Codex 扩展（Edge） | `%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Extensions\odlomjlbamekndcpllcnffbgeohgkmjh\1.26.901.11451_0\` |
| Codex host | `%USERPROFILE%\.codex\plugins\cache\openai-bundled\chrome\latest\extension-host\windows\x64\extension-host.exe` |
| **Codex 自带文档（22 篇）** | `%USERPROFILE%\.codex\plugins\cache\openai-bundled\chrome\latest\docs\` |
| 官方文档存档（抓取） | `D:\workspace\pjzck\.dsh\tmp\docs\`（临时，方案定稿后删） |

## 9. 未采纳的选项与原因

| 选项 | 原因 |
|---|---|
| HTTP 长轮询 | 📄 `fetch` 响应超 30 秒即终止 SW，长轮询根本拖不住 |
| 只做 `webRequest` 抓包 | 📄 拿不到响应体（只有请求体） |
| 只做页面内 hook（改 fetch/XHR） | 抓不到子资源与注入前的请求；页面可探测（仍可作为"免 debugger 权限"的降级档，❓待 S0-G 定） |
| 扩展里硬编码 CDP 编排（Qoder 式） | 改行为要发新版扩展；采用 Codex 的直通式更灵活 |
| 直接开 WebSocket 数据通道（Qoder 式） | 需要在 Python 里实现 WS 服务端；先用 native messaging 分页即可满足，留作备选 |
| Playwright 式完整定位 API | 参考实现有 66 个 type 的完整面，我们只需要够用的一层 |
