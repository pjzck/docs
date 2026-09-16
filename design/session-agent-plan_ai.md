# Session Agent：跨会话 GUI 操作代理设计方案

> 状态: 待实现 | 目标平台: Windows 10/11
> 来源: 由 DSH（Lucky 项目会话）中的实测需求反向推导，附可复现的验证记录

## 1. 问题

AI 编码代理（DSH）以 **Windows 服务** 形式运行（服务名 `DeepSeekHarness`）。服务由 SCM（`services.exe`）在 **Session 0** 创建，导致整条进程链都在 Session 0：

```
node.exe (Session 0)  →  node.exe (Session 0)  →  powershell.exe (Session 0)   ← 工具命令在这里执行
```

而用户的交互桌面在 **Session 1**（`explorer.exe` / `WinSta0\Default`）。

结果是：**代理能改代码、跑测试、操作 git，但完全看不见也碰不到任何 GUI** —— 无法截图验证界面改动、无法点击、无法拖拽。本文给出解决这个问题的方案。

## 2. 约束：Windows 会话隔离

### 2.1 根本原因

| 层级 | 事实 |
|---|---|
| **会话身份** | SessionId 是**令牌的属性**，进程创建时从父进程令牌继承；**没有任何 API 能修改已存在进程的 SessionId** |
| **会话来源** | 交互登录 → Winlogon 建会话（Session 1）；服务登录 → SCM 建会话并**强制 SessionId = 0** |
| **窗口站/桌面** | 每个会话各有自己的窗口站命名空间：交互会话固定是 `WinSta0`，Session 0 的窗口站名随机（如 `Service-0x0-ce0c0c42$`） |
| **对象归属** | 窗口、Desktop、显示器 DC 都带会话归属，跨会话访问被内核拒绝 |
| **输入注入** | `SendInput` 只投递给**调用线程所在桌面**；Session 0 隔离的设计目标之一就是阻止服务向用户桌面注入输入 |

### 2.2 关键推论

1. **改服务账号无效** —— 账号与会话无关。把服务账号改成当前用户，进程仍在 Session 0，因为分配 SessionId 的是「登录类型」而非账号身份
2. **杀进程重启无效** —— 只要父进程（SCM）在 Session 0，重建的子进程仍在 Session 0
3. **进程搬不进其他会话，但可以新建一个已在目标会话的进程** —— 改令牌的 `TokenSessionId` 再用 `CreateProcessAsUser`：

```
复制令牌 → SetTokenInformation(TokenSessionId, 1) → CreateProcessAsUser(该令牌)
  → 新进程 SessionId = 1，属于 WinSta0\Default，可正常截屏/注入输入
```

   这正是 `schtasks /create /it`（interactive only）的底层实现，也是很多正常软件（OneDrive Startup Task、Clash Verge 等）自启的方式

4. **网络与内核 IPC 不受会话隔离限制** —— 这是可用的通信通道

## 3. 实测环境事实

在一台实际开发机上验证（可作为可核对断言）：

| 项 | 值 |
|---|---|
| 交互会话 | Session 1，`console` 状态 Active，`explorer.exe` PID 11924 |
| 服务侧会话 | Session 0，其中进程的 WindowStation = `Service-0x0-ce0c0c42$`，Desktop 名为 `Default`（**与 `WinSta0\Default` 撞名但不同对象**） |
| Session 0 可见窗口数 | **0** |
| Session 0 光标位置 | 固定 `(0, 0)` |
| `CopyFromScreen` | 报 **"The handle is invalid"**（取不到显示器 DC） |
| 可用运行时 | Node v24.18.0、Python 3.14.0、.NET 10.0.302、Windows PowerShell **5.1**（无 pwsh 7） |
| 权限 | 用户为本地管理员，`schtasks` 可创建 `/it` 任务（机器上已存在多个 `Interactive only` 任务，通路验证过） |
| 显示适配器 | 独立显卡 + 一个虚拟显示适配器（远程串流用） |

## 4. 方案对比

| 方案 | 能力 | 评价 |
|---|---|---|
| **A. `/it` 一次性任务** | 仅截屏 | 实现最简单（约 30 行脚本），每次用需建任务→执行→删任务；**适合偶尔看界面** |
| **B. 会话内常驻代理** | 截屏 + 窗口操作 + 输入注入 | **推荐**。一次性搭建，之后随时可用；代价是需要一个常驻进程 |
| **C. 隐藏桌面（Hidden Desktop）** | 同 B，且**完全不干扰用户** | 最优雅：`CreateDesktop` 建隐藏桌面，把目标应用起在那里再截屏/注入。但工程量大，且需验证目标应用（尤其自绘 UI）在非显示桌面上的渲染稳定性 |
| ~~D. 把代理进程搬进 Session 1~~ | — | **不可能**，见 §2.2 |
| ~~E. 改服务账号~~ | — | **无效**，见 §2.2 |
| ~~F. `tscon` 搬会话~~ | — | **禁止**，会把用户踢下线 |

### 建议的实施顺序

**A → B →（可选）C**，即先用一次性任务解决"看一眼"的需求，确认价值后再硬化成常驻代理。

## 5. 目标架构（方案 B）

```
         Session 0（服务侧）                      Session 1（用户桌面）
    ┌──────────────────────────┐            ┌──────────────────────────┐
    │  AI 代理 / 工具命令        │  request   │   Session Agent（常驻）    │
    │  （任意 CLI）             │ ─────────► │   - 截屏                  │
    │                          │            │   - 窗口操作               │
    │                          │ ◄───────── │   - 输入注入               │
    └──────────────────────────┘  response  └──────────────────────────┘
                                         执行对象 ↓
                                  ┌──────────────────────────┐
                                  │  目标 GUI 应用             │
                                  └──────────────────────────┘
```

设计原则：**不试图把工具搬进 Session 1，而是在 Session 1 放一个常驻的"手"，通过 IPC 指挥它。**

## 6. 传输通道选型

会话隔离**不拦**网络与内核 IPC：

| 通道 | 可行性 | 评价 |
|---|---|---|
| **127.0.0.1 TCP** | ✅ | **推荐**。Session 0 连本机 loopback 不受会话隔离影响；绑定 `127.0.0.1` 不触发防火墙弹窗。需处理请求/响应语义（调用方等待结果） |
| 文件轮询 | ✅ | 最简单、零依赖、易调试，但有轮询延迟 |
| 命名管道 | ✅ | 延迟低、ACL 可精确到用户；部分受限环境下管道 API 不可用，需实测 |

**硬性要求**：只绑 `127.0.0.1`（**禁止** `0.0.0.0`），并做鉴权（见 §8）。

## 7. 接口设计

### 7.1 P0 —— 满足核心用途（截屏验证界面）

| 命令 | 说明 |
|---|---|
| `shot --window <标题片段> --out <png>` | 截指定窗口（按 `GetWindowRect` 裁剪）；省略 `--window` 则全屏 |
| `windows` | 列出可见窗口：标题 / PID / HWND / 窗口矩形 |
| `focus --window <标题片段>` | 还原（若最小化）并置为前台 |

### 7.2 P1 —— 交互能力

| 命令 | 说明 |
|---|---|
| `click --x <n> --y <n> [--button l\|r]` | 屏幕坐标点击 |
| `move --x <n> --y <n>` | 移动鼠标 |
| `drag --x1 <n> --y1 <n> --x2 <n> --y2 <n>` | 拖拽（用于验证拖拽类交互） |
| `type --text <s>` / `key --name <k>` | 键盘输入 / 按键 |
| `resize --window <标题片段> --w <n> --h <n>` | 改窗口尺寸 |
| `proc list` / `proc kill --name <n>` | 进程查看 / 结束 |

### 7.3 P2 —— 不干扰用户（可选）

`CreateDesktop` 建隐藏桌面 → 目标应用启动到该桌面 → 对该桌面截屏/注入。用户真实鼠标键盘完全不受影响。工程量大，需单独验证自绘 UI 的渲染稳定性。

## 8. 安全要求（不可省）

代理等于把桌面完整控制权交给一个端口，因此：

1. 只监听 `127.0.0.1`
2. 随机 token 鉴权，token 写入仅当前用户可读的文件
3. 不配置防火墙入站放行，不暴露局域网
4. 优先「按需启动」（用户双击才起）；常驻自启作为可选项，由用户决定
5. 代理不修改系统服务配置，不新增自启动项（除用户显式要求的 `/it` 任务外）

## 9. 关键实现细节与坑

1. **DPI 感知（必须先做）**：代理进程必须声明 DPI aware，否则注入坐标会被系统虚拟化而偏移。可用 `SetProcessDpiAwarenessContext`
2. **窗口必须非最小化**：最小化窗口无法截图，`focus` 需先 `ShowWindow(SW_RESTORE)`
3. **前台锁**：`SetForegroundWindow` 受前台锁定限制会失败；稳妥做法是 `ShowWindow` → `BringWindowToTop` → 必要时短暂 `AttachThreadInput` 抢焦点
4. **自绘 UI 没有 UIA**：基于 ImGui / 自绘的桌面工具**没有原生控件、没有 UIA/MSAA 元素树**，因此**只能按坐标点击，不能按控件名查找**
5. **输入注入统一用 `SendInput`**：`mouse_event` 已过时且受 MouseKeys 参数污染；`PostMessage` 发鼠标消息对自绘 UI 不可靠
6. **坐标系统一为物理像素**：返回的窗口矩形与接受的注入坐标必须是同一口径
7. **禁止静默降级**：能力失败要明确报错并说明原因，不得返回伪造的成功结果
8. **截屏实现建议**：借 PowerShell + Windows API，避免引入 native 截图模块（减少依赖与 ABI 风险）：

```powershell
Add-Type -AssemblyName System.Windows.Forms,System.Drawing
$b = [System.Windows.Forms.SystemInformation]::VirtualScreen
$bmp = New-Object System.Drawing.Bitmap($b.Width, $b.Height)
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.CopyFromScreen($b.X, $b.Y, 0, 0, $bmp.Size)
$bmp.Save($out, [System.Drawing.Imaging.ImageFormat]::Png)
```

9. **会话归属自证**：代理启动后应打印自身 WindowStation（交互会话应为 `WinSta0`），便于确认"是否真的在 Session 1"

## 10. 验收标准

1. 用户双击启动代理后，代理**确实运行在 Session 1**（自证 WindowStation == `WinSta0` 并打印）
2. 从 Session 0 的 PowerShell 执行 `windows`，能列出 Session 1 的**真实非空**窗口列表
3. 执行 `shot --window "<真实窗口标题片段>" --out <path>`，产出**非空、内容正确**的 PNG（文件大小明显大于空图，且肉眼可辨认）
4. `focus` 后目标窗口确实被置顶
5. 代理停止后无残余进程，未新增防火墙规则，未修改系统服务配置
6. 全程不修改 DSH 服务（`DeepSeekHarness`）的任何配置

## 11. 边界：不做的事

- 不试图把 Session 0 的进程搬进 Session 1（不可能）
- 不通过修改服务账号来解决（账号与会话无关）
- 不使用 `tscon` 搬会话（会把用户踢下线）
- 不监听 `0.0.0.0`，不免鉴权
- 不在用户未交互登录 / 锁屏时伪造截图成功（`/it` 语义上无法执行，应如实报告）

## 12. 落地步骤

| 步骤 | 内容 | 产出 |
|---|---|---|
| 1 | 实现 P0 三命令 + 手动启动 | 双击即可用的代理 |
| 2 | 在 Session 0 验证 `windows` / `shot` / `focus` | 验证记录 + 真实 PNG |
| 3 | （可选）注册 `/it` 计划任务实现登录自启 | `schtasks /create /tn "SessionAgent" /sc onlogon /it /ru <用户> /f` |
| 4 | （可选）实现 P1 输入注入 | 点击/拖拽/输入 |
| 5 | （可选）实现 P2 隐藏桌面 | 完全不干扰用户的形态 |

## 13. 手工验证记录（本次会话实测）

| 验证项 | 命令 / 方法 | 结果 |
|---|---|---|
| 服务侧会话号 | `(Get-Process -Id $PID).SessionId` | `0` |
| 服务侧窗口站 | `GetProcessWindowStation` + `GetUserObjectInformation` | `Service-0x0-ce0c0c42$` |
| 服务侧桌面 | `GetThreadDesktop` + `GetUserObjectInformation` | `Default`（与交互桌面撞名，不同对象） |
| 服务侧可见窗口 | `Get-Process \| Where MainWindowTitle -ne ''` | **0 个** |
| 服务侧光标 | `[System.Windows.Forms.Cursor]::Position` | `(0, 0)` |
| 跨会话截屏 | `Graphics.CopyFromScreen` | **异常 "The handle is invalid"** |
| 交互会话存在性 | `query session` | `console  zhouchenkai  1  Active` |
| `/it` 任务通路 | `schtasks /query /v` 查既有任务 | 存在多个 `Interactive only` + 同用户任务 |

## 14. 附：可直接交给实现方的提示词

```text
# 任务：实现一个 Windows「会话内代理」（Session Agent），用于驱动 Session 1 的 GUI

## 背景
AI 编码代理（DSH）以 Windows 服务形式运行（服务名 DeepSeekHarness），服务由 SCM 在
Session 0 创建，整条进程链（node.exe → node.exe → powershell.exe）都在 Session 0；
用户交互桌面在 Session 1（explorer.exe / WinSta0\Default）。

已实测确认（不要重复怀疑）：
- Session 0 进程 WindowStation = Service-0x0-<随机>$，Desktop 名为 Default 但与
  WinSta0\Default 是不同对象
- Session 0 中 EnumWindows 可见窗口数 = 0，光标固定 (0,0)
- CopyFromScreen 报 "The handle is invalid"
→ Session 0 进程既不能截 Session 1 的屏，也不能向 Session 1 注入输入

原理要点（决定方案边界）：
- SessionId 是令牌属性，进程创建时继承，无 API 可改已存在进程的 SessionId
- 只能改令牌 TokenSessionId 再用 CreateProcessAsUser 新建进程，新进程才在目标会话
- 截屏/SendInput/SetForegroundWindow/FindWindow 只作用于调用进程所在会话
- 网络与内核 IPC 不受会话隔离限制，这是可用通道

## 环境事实
- 用户为本地管理员（IsAdmin: True）
- 交互会话 Session 1，console 状态 Active
- 可用运行时：Node v24.18.0、Python 3.14.0、.NET 10.0.302、Windows PowerShell 5.1（无 pwsh 7）
- 默认 shell 为 powershell.exe；schtasks 可用且 /it 通路已验证
- 目标 GUI 应用为基于 ImGui 自绘的桌面工具「Lucky Launcher」，窗口标题形如
  "Lucky Launcher - 本机UID: <uid> Center: <ip> - <path>"

## 目标
做一个常驻在 Session 1 的代理进程，提供请求/响应接口；Session 0 侧通过命令行下达指令，
代理执行 GUI 操作并返回结果。

## 必须满足
1. 传输通道首选 127.0.0.1 TCP（只绑 loopback，不用 0.0.0.0）；请求/响应语义，
   调用方发起后等待结果再退出
2. 安全：仅 loopback + 随机 token 鉴权 + token 存仅本用户可读文件；不配防火墙放行
3. 会话归属：代理必须运行在 Session 1。先做「用户双击 bat 手动启动」，自启作为可选项：
   schtasks /create /tn "SessionAgent" /tr "<启动命令>" /sc onlogon /it /ru <用户> /f
   （不加 /it 会跑在 Session 0，等于没用）
4. 技术选型：宿主用 Node 或 Python；截屏那一步借 PowerShell + Windows API（System.Drawing），
   不要引入 native 截图模块
5. 必须先做 DPI 感知（SetProcessDpiAwarenessContext），否则注入坐标会偏移
6. 命令接口：
   P0: shot --window <标题片段> --out <png> | windows | focus --window <标题片段>
   P1: click / move / drag / type / key / resize / proc list|kill
   P2（可选）: CreateDesktop 隐藏桌面方案，完全不干扰用户
7. 统一用 SendInput 做输入注入；不要用 mouse_event，不要用 PostMessage 发鼠标消息
8. 注意自绘 UI（ImGui）没有 UIA/MSAA 元素树，只能按坐标点击，不能按控件名查找
9. 窗口最小化时无法截图，focus 需先 ShowWindow(SW_RESTORE)
10. 禁止静默降级：能力失败要明确报错并说明原因，不得返回伪造成功

## 验收标准
1. 代理启动后自证运行在 Session 1（打印自身 WindowStation == WinSta0）
2. 从 Session 0 执行 windows 能列出 Session 1 的真实非空窗口列表
3. 执行 shot 产出非空且内容正确的 PNG（文件大小明显大于空图，肉眼可辨认）
4. focus 后目标窗口确实置顶
5. 代理停止后无残余进程、无新增防火墙规则、未改系统服务配置
6. 全程不修改 DSH 服务（DeepSeekHarness）配置

## 不要做
- 不要把 Session 0 进程搬进 Session 1（不可能）
- 不要改服务账号来尝试解决（账号与会话无关）
- 不要用 tscon 搬会话（会把用户踢下线）
- 不要监听 0.0.0.0，不要免鉴权
- 不要在用户未登录/锁屏时伪造截图成功

## 交付物
1. 代理程序（含依赖说明与启动方式）
2. 命令行调用脚本 / CLI
3. README：启动、停止、自启、命令清单、安全说明、已知限制
4. 一次真实运行的验证记录：命令 + 产物 PNG 路径 + 结果
```
