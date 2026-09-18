# Session Agent 浏览器能力：最终架构与 v0.3 实现基线

> 2026-09-18 定稿并实现首版。面向任意 AI，操作用户现有 Chrome、复用登录态。
> 之前的 CDP / Native Messaging / 参考产品调研保留在 [研究记录](session-agent-browser-research_ai.md)，其中 v2 的命令、安全闸门和阶段规划不再作为当前实现契约。

## 1. 产品定位

为只有命令执行或 MCP 能力的 AI 提供通用的桌面和浏览器操作能力。浏览器操作经过页面控件和浏览器输入事件，由网站自身逻辑完成业务动作；页面读取增强感知，截图覆盖结构化阅读表达不了的视觉内容。

一个本地 Agent、两个内部模块、一个 Chrome 扩展。页面阅读与浏览器操作共享目标和引用，不拆成两个插件。CLI 和 MCP 是同一套能力的两个入口，与调用方 AI 产品无关。

## 2. 架构

```text
任意 AI（Session 0 或其他本地会话）
  ├─ CLI：client.py / session-agent.cmd
  └─ MCP stdio：mcp_server.py
          │ 127.0.0.1 TCP + token
    Session Agent（用户交互会话）
      ├─ desktop：现有截图、窗口、SendInput
      └─ browser：目标、租约、阅读与交互编排
          │ 认证的双向 TCP 连接，request ID + response
    Native Host（Chrome 启动）
          │ connectNative，长度前缀 JSON
    Chrome 扩展
      ├─ tabs：标签页枚举/导航/关闭
      ├─ debugger：附加、CDP 命令、取消事件
      └─ 状态、正在控制的页面、停止入口
```

旧的 Native Host status/start/stop 保留一次一答模式；新增 browser.connect 常驻模式。浏览器重连只连接已有代理，绝不隐式启动已停止的代理。

常驻端口承载数据与保活。扩展每 20 秒心跳，断线退避重连，30 秒 alarm 兜底。用户主动停止或 Chrome 取消调试后禁用浏览器控制，必须显式启用才能恢复。

## 3. 核心闭环

```text
tabs → read → fill/click/press/scroll → read/wait/screenshot
```

- tabs 返回连接 ID + 标签页 ID 组成的精确句柄，只操作枚举或新建得到的目标。
- 每次操作显式带目标，不存在跨 AI 共享的“当前标签页”。
- interactive 阅读返回元素引用、名称、角色和值/禁用/选中等状态；content 阅读返回正文、标题与链接。
- 页面辅助脚本运行在 CDP isolated world，只供读取与定位使用；常规工具不暴露任意 JS 或网站业务接口调用。
- click/fill 使用实际节点解析、滚动、遮挡检查和 CDP Input。fill 替换文本并回读核对。
- 截图与坐标点击共享标签页，浏览器坐标为视口 CSS 像素；桌面操作仍为屏幕物理像素。
- action_sent 与业务成功分开。上层 AI 必须回读目标结果，不能把输入发送成功当作任务完成。
- 超时结果可能未知，不重放输入动作，不静默换标签页或切到系统鼠标。

## 4. 状态与并发

元素引用绑定节点与快照。重新读取交互快照、文档导航、用户手动导航返回、连接重建、节点替换或关键身份变化会让旧引用失效。标题与 URL 变化本身不改变标签页身份。

连接内按完整动作串行执行，防止多条底层 CDP 命令交叉。需要跨调用独占标签页时使用 claim/release：120 秒租约，带 owner 调用续期；其他调用方得到 TAB_BUSY。

扩展保留用户最新的启用/停止决定，启动时异步读取的旧偏好不能覆盖它。

## 5. 权限与边界

manifest 使用 nativeMessaging、alarms、storage、tabs、debugger，最低 Chrome 120。debugger 是输入、截图和读取路线的基础权限；按需附加页面，不以“仅抓包才需要”描述它。当前路线不需要 scripting 或 all_urls host 权限。

代理仅监听本机 loopback，沿用随机 token 和运行时目录 ACL。浏览器请求日志只记录动作和目标，不记录输入文本和页面内容。密码值在读取/验证回执中遮蔽。

页面内容属于第三方数据，上层 AI 不应将它视为用户指令。底层不引入固定的“每次提交必须确认”策略，具体动作授权由调用方结合用户任务处理。

## 6. 首版交付

| 模块 | 实现 |
|---|---|
| 共用契约 | browser_api.py：CLI/MCP/服务端共用字段与验证 |
| 常驻桥接 | browser_bridge.py、native_host.py、扩展 browser-bridge.js |
| 浏览器编排 | browser_driver.py：CDP 输入、截图、目标与租约 |
| 页面提取 | browser_page.js：渲染 DOM 语义、正文、开放 Shadow DOM、引用验证 |
| 通用入口 | client.py browser 子命令；mcp_server.py 的 browser_* / desktop_* |
| 用户入口 | 一个扩展弹窗管理本地代理与浏览器控制，显示受控页面 |
| 使用说明 | [BROWSER.md](../../tools/session-agent/BROWSER.md) |
| 自动验证 | Python 协议测试、JS 生命周期测试、真实扩展端到端测试 |

命令：connections / tabs / open / navigate / back / close / read / click / fill / press / scroll / screenshot / wait / claim / release。

## 7. 已验证与未覆盖

已通过 9 项 Python 测试、2 项模拟生命周期测试、14 组真实 Chromium 端到端检查。真实链路包含扩展、Native Host、生产 AgentServer、CLI 和 MCP；中文/emoji、trusted 事件、正文分页、PNG、节点替换、遮挡、用户导航返回、租约和停止恢复均有验证。

完整测试使用隔离 Chromium profile 和本地页面。用户启用新版本后，又在日常 Chrome 152 中新开本地测试页，完成读取、中文/emoji 填写、点击与回读结果、截图和关闭测试页的现场验收；已有业务页面未被修改，连接保持启用。[现场记录](assets/session-agent-browser-v3-live.json)。Chrome 用户取消调试事件目前采用模拟事件测试；实际提示条点击、长时间保活、不同生产网站及后台/最小化场景仍需后续验证。

首版读取是渲染 DOM 语义，不是完整 AX 树。iframe 内结构、关闭 Shadow DOM、Canvas 内部内容、复杂拖放、上传下载管理、网络抓包、控制台日志暂未提供。iframe 缺失和扫描截断会明确回报。

## 8. 后续顺序

1. 在用户日常 Chrome 中使用，收集实际网页失败案例。
2. 优先补 iframe/OOPIF、定位与动态页面等待能力。
3. 按实际任务增加上传/下载、复杂拖放。
4. 增加控制台日志与网络调试；保持通用操控入口简洁。
