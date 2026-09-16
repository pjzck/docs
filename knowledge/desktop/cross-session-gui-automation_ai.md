# 跨会话 GUI 自动化：让 Session 0 的服务看见并操作用户桌面

> 来源：2026-09 为 DSH（以 Windows 服务运行）做的 Session Agent（代码在 `tools/session-agent/`）。
> 本文只沉淀**可复用的结论与坑**，不写那个工具怎么用。标「实测」的都是本机真跑过的；标「推断」的没验。

## 1. 会话隔离改了哪些事

| 事实 | 后果 |
|---|---|
| SessionId 是**令牌属性**，进程创建时从父进程继承；**没有 API 能改已存在进程的 SessionId** | 进程搬不进别的会话；改服务账号也没用（账号与会话无关，决定会话的是登录类型） |
| 服务由 SCM 创建，**强制 SessionId = 0** | 只要父进程在 Session 0，重建子进程仍在 Session 0 |
| 每个会话各有窗口站：交互会话固定 `WinSta0`，Session 0 的窗口站名随机 | 实测 Session 0 的窗口站是 `Service-0x0-ce0c0c42$`，其桌面**也叫 `Default`**，但与 `WinSta0\Default` 是不同对象 |
| 窗口 / 桌面 / 显示器 DC 都带会话归属 | 跨会话访问被内核拒绝 |
| `SendInput` 只投递给**调用线程所在桌面** | Session 0 里注入不会到用户界面 |
| 网络与内核 IPC **不受**会话隔离限制 | 所以可以"在 Session 1 放一只常驻的手 + 从 Session 0 用 loopback 指挥" |

Session 0 里的实测表现（都能当断言用）：

- 可见窗口数 **0**
- 虚拟屏幕只有个假尺寸（实测 `1024x768`），不是真实分辨率
- `GetCursorPos` **取不到**（返回失败），`CopyFromScreen` 报 **"The handle is invalid"**
- `OpenInputDesktop` 直接 `err=1`（拒绝访问）

## 2. 在 Session 1 里创建进程的四种途径

**核心思路**：不要在 Session 0 里想办法，而是让 **Session 1 自己的东西**去创建进程。

| 途径 | 谁创建 | 备注 |
|---|---|---|
| 用户双击 bat | `explorer.exe`（Session 1） | 最土但最可靠；缺点是要用户动手 |
| 计划任务 `schtasks /create /it` | Task Scheduler | **未实测**。`/it` = interactive only，底层就是改令牌 `TokenSessionId` 再 `CreateProcessAsUser` |
| **自定义 URL 协议** | 浏览器（Session 1） | **实测**：页面点 `dshagent://start` → `wscript` → `pythonw`，新进程自证 `SessionId=1 / WinSta0`。首次点浏览器弹一次确认框 |
| **Chrome native messaging host** | Chrome（Session 1） | **实测**：host 由 Chrome fork，天然 Session 1；不弹任何确认框。见 §7 |

> 反过来看 DSH 自己的部署：它的 `cordis.patch.yml` 注释里写着"服务在 Session 0，原生目录选择器会弹在看不见的桌面上"，因此禁用了那个功能 —— 同一个根因。

## 3. 坐标与 DPI：一律按物理像素

- 有 DPI 虚拟化时，`GetWindowRect` 拿到的是**逻辑像素**，和截图坐标对不上。**声明 per-monitor 感知**后才是物理像素。
- **不要无脑调 `SetProcessDpiAwarenessContext`**：如果宿主清单已经声明过（`powershell.exe` 就是），该调用会返回 FALSE + `err=5 ACCESS_DENIED`。**正确做法是读出来校验**：
  - `GetAwarenessFromDpiAwarenessContext(GetThreadDpiAwarenessContext())` → `2` = per-monitor
  - 不是 2 就明确报错，别硬跑（否则坐标会静默偏移）
- **DPI 感知会被父进程继承**（实测：从 per-monitor 的父进程启动的 python 直接就是 2）—— 所以同一个程序在不同父进程下表现可能不同，**运行时自证**比"我设过了"可靠。
- 多显示器：虚拟屏幕是各屏并集（本机 `0,0 3840x1080` = 两块 1920×1080）。窗口矩形可能跨屏或部分在屏外，**截图前要裁剪到虚拟屏幕**并如实报告裁剪。

## 4. 截屏：两条路，各自的坑

| 方式 | 优点 | 缺点 |
|---|---|---|
| `Graphics.CopyFromScreen`（GDI+ 抓屏） | 拿到**屏幕上真实像素**（DWM 合成后的结果） | 会被遮挡；锁屏/切桌面拿不到；Session 0 里必失败 |
| `PrintWindow(hwnd, hdc, PW_RENDERFULLCONTENT=2)` | 不抢焦点、能抓被遮挡窗口 | 部分 D3D/硬件加速窗口给**黑图**；`PW_RENDERFULLCONTENT` 这个 flag 不能省 |

必须做的三道校验（少一道就可能返回"看起来成功其实没用"的图）：

1. **空白检测**：抽样式统计颜色种类，只有 1 种就判为无效（锁屏、未渲染都会得到单色图）
2. **宿主口径校验**：截屏子进程自己看到的桌面尺寸/DPI 感知级别，必须和主进程一致 —— 不一致说明坐标被虚拟化了，**拒绝返回**
3. **抓屏瞬间的前台校验**：焦点是几百毫秒前设的，而截屏脚本启动+编译要时间，够别的窗口抢焦点。**在真正 BitBlt 之前重新置前并回报实际前台窗口**，不是就标记"可能被遮挡"

## 5. 输入注入（SendInput）

```c
/* 鼠标绝对坐标：必须带 VIRTUALDESK */
flags = MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE | MOUSEEVENTF_VIRTUALDESK;
dx = round(x * 65535 / (virtual_w - 1));
dy = round(y * 65535 / (virtual_h - 1));
```

- **`MOUSEEVENTF_VIRTUALDESK` 不能省**：不带时绝对坐标按**主显示器**归一化，多显示器下整体偏移。实测双屏下 `move --x 2400` 精确落在副屏（若少了这个 flag 会跑到主屏去）。
- 鼠标侧键靠 `mouseData = XBUTTON1/XBUTTON2` + `MOUSEEVENTF_XDOWN/XUP`；滚轮 `MOUSEEVENTF_WHEEL`，`mouseData = ±120 × 格数`。
- **键盘文本用 `KEYEVENTF_UNICODE`**（`wScan` = UTF-16 码元）：与键盘布局无关，中文/符号都能进；emoji 是代理对，要按码元拆成两次注入。
- **方向键 / Home / End / PgUp / PgDn / Delete / Win / 音量等必须带 `KEYEVENTF_EXTENDEDKEY`**，否则被认成小键盘或左侧对应键。
- **`INPUT` 结构在 x64 上是 40 字节**（`MOUSEINPUT` 32 / `KEYBDINPUT` 24 + 8）。sizeof 错了 SendInput 会静默失败 —— 值得在自检里断言一下。
- 用 `SendInput`，**不要用 `mouse_event`**（已废弃）；**`PostMessage` 发鼠标消息不算真输入**（很多程序不认）。
- 组合键的注入顺序：按下修饰键 → 按主键 → 抬主键 → **逆序抬修饰键**，整批一次 `SendInput`。

两个绕不过的限制：

- **前台锁**：`SetForegroundWindow` 只允许前台进程抢占前台。可靠套路是 `AttachThreadInput` 把自己附加到**前台窗口线程**和**目标窗口线程**，再 `BringWindowToTop` + `SetForegroundWindow`，然后脱离。仍失败就如实报错（不要假装置顶了）。
- **UIPI**：普通完整性级别的进程**驱动不了管理员窗口**。表现为 `SendInput` 失败或输入被丢弃。这是设计使然，不是 bug。

## 6. 没有 UIA 的自绘界面（ImGui 等）怎么定位控件

自绘 UI **没有原生控件树**，UIA/MSAA 查不到任何东西，只能按坐标点。方法：

1. **截图 + 扫像素量边界**。肉眼看图估坐标会差十几像素（点空）；扫"边框色"能拿到精确到 1px 的边界，再和布局算式（内容宽、内边距、`min-height`）**互相印证**。
   实测例：从弹窗截图扫出按钮边框 `x=286..336, y=77..103`（宽 50 / 高 26），与 CSS 算出来的 50px / `min-height:27px` 一致。
2. **注入前报告"这点下面是谁"**：`WindowFromPoint` + `GetAncestor(GA_ROOT)` 拿到顶层窗口的 hwnd/标题/PID/进程，作为结果的一部分返回。**先看这一项，再决定要不要点。**
3. **窗口相对坐标优于绝对坐标**：不怕窗口被挪动。
4. **注入前复核**：算出的点必须仍在窗口**当前**矩形内（窗口动过就拒绝注入，绝不盲点）。
5. **能选的靶子就选零副作用的**：比如验证"打字"用 VS Code 的 `Ctrl+F` 查找框（打完看框里有没有字、`Esc` 退出、文件零改动），而不是往用户文件正文里打。

## 7. 配套 Chrome 扩展（native messaging）

native messaging host 是 **Chrome 的子进程**，所以它天然在 Session 1 —— 这是"点一下就在 Session 1 创建进程"最干净的途径。

- **注册**（仅当前用户，免管理员）：
  `HKCU\Software\Google\Chrome\NativeMessagingHosts\<host 名>` = 指向 host manifest JSON 的路径；
  JSON 里 `path` 是 Chrome 要 spawn 的文件，`allowed_origins` 精确到扩展 ID。
- **`path` 可以是 `.bat`**（实测：本机 `com.qoder.work.connector` 就是 `.bat`，我们自己的也跑通了）。
- **host 是短命的**：`sendNativeMessage` 每次起一个进程、答完即退。要常驻得自己 detached 启动（并把 stdio 重定向到 DEVNULL，否则 Chrome 会等管道关闭）。
- **Chrome 可能把 host 放进 job object**（job 关闭会连坐杀掉里面的进程）：先试 `CREATE_BREAKAWAY_FROM_JOB`，兜底请 **`explorer.exe`** 拉起（Session 1 的 shell，必然在 job 之外）。实测 breakaway 成功。
- **固定扩展 ID**：改 `name`/`description` **不需要重装扩展**。ID 由 manifest 的 `key`（公钥 DER 的 base64）派生 = `SHA256(DER)[:16]` 每位映射到 `a..p`；改文案不改 ID，所以 host 的 `allowed_origins` 一个字都不用动。实测 ID 稳定。
- **manifest 里的 `key` 是公钥，可以放心提交**（它只决定 ID，不是签名私钥）。
- **MV3 的 service worker 会被回收**，`setInterval` 长期不可靠 → 定时任务用 `chrome.alarms`（周期是分钟级）。网上流传的"native messaging 长连接能保活 SW"**有争议**（chromium 官方文档仓库 [issue #2688](https://github.com/GoogleChrome/developer.chrome.com/issues/2688)），别依赖它。
- **图标按资源 URL 缓存**：**同名文件换内容不生效**（实测：换了 `icons/16-run.png` 的内容后点 ⟳ 刷新，工具栏仍是旧图，得关开扩展或重启浏览器）。想只靠 ⟳ 生效就得**换文件名**（如带版本后缀），并同步 manifest 与代码两处。

### headless Chrome 拿来做渲染/预览，三个坑

1. **`--virtual-time-budget` 下 CSS 过渡的时钟不推进** → 截到的是过渡**中间态**。夹具里必须 `*{transition:none !important}`。（曾为"按钮颜色每轮都不一样"白查三轮假 bug。）
2. **`--window-size=128` 会渲成空白**（稳定复现，输出仅几百字节）。要 128px 就用 **「64 窗口 + `--force-device-scale-factor=2`」**（矢量图不损清晰度）。
3. **Chrome 的启动器是异步返回的**：`--screenshot` 命令返回 ≠ 文件已落盘，必须**轮询等文件**。
4. 透明背景用 `--default-background-color=00000000`；给扩展页面预览时要 `--allow-file-access-from-files`（否则 `localStorage` 不可用）。

## 8. 反面清单（别做）

- **不要 `tscon` 搬会话** —— 会把用户踢下线。
- **不要监听 `0.0.0.0`** —— 只绑 `127.0.0.1` 就不会触发防火墙弹窗、也不需要放行规则。
- **锁屏 / 桌面不可见时不要伪造截图成功** —— 宁可报 `BLANK_CAPTURE` / `DESKTOP_NOT_VISIBLE`。
- **不要用"计划任务自启"来偷懒解决会话问题** —— 那改变的是启动方式，不是会话隔离本身。
- **不要往用户的文件里打字做测试** —— 选零副作用靶子，事后确认没留痕迹。

## 9. 哪些是实测、哪些只是推断

| 结论 | 状态 |
|---|---|
| Session 0 窗口数 0 / 光标取不到 / `CopyFromScreen` 报 handle invalid | 实测 |
| 用户双击、URL 协议、Chrome native messaging 三条途径都落在 Session 1 | 实测（各自自证 `SessionId=1 / WinSta0`） |
| `schtasks /it` 落在 Session 1 | **未实测**（文档记录 + 既有 `/it` 任务为旁证） |
| `MOUSEEVENTF_VIRTUALDESK` 缺失会导致多屏偏移 | 实测（带 flag 时落点精确在副屏） |
| `KEYEVENTF_UNICODE` 能输中文 | 实测 |
| UIPI 拦住管理员窗口 | **推断**（按 Windows 机制；未在管理员窗口上实测） |
| DPI 感知可由父进程继承 | 实测 |
| 改扩展 `name`/`description` 不影响 ID、不用重装 | 实测（重算 ID 与 host 注册比对一致） |
| 图标资源按 URL 缓存、同名换内容不生效 | 实测 |
| Chrome 把 host 放进不允许 breakaway 的 job | **未复现**（本机 breakaway 成功） |
