# pjzck-tool 工具链设计方案

> 状态: 草案 | 等待用户提供 pjzck tools 外网 repository 地址

## 1. 目标

参照 Note 项目的 diago 工具链，为 pjzck 组织构建一套独立的 AI 工具链，实现：

- **跨项目复用** skills/hooks 的集中管理与按需挂载
- **内网私有发布** 通过 npm registry 分发 CLI 工具
- **统一入口管理** 一键安装/更新所有 pjzck CLI 工具
- **安全可控** 包名受限、registry 内置、LDAP 认证

## 2. 与 diago 的关系

| 维度 | diago（Note 项目） | pjzck-tool（本项目） |
|------|-------------------|---------------------|
| npm scope | `@diago/*` | `@pjzck/*` |
| Registry | `registry.wangyuan.net/repository/diago-tools/` | **待用户提供** |
| 入口命令 | `diago-tools-cli` | `pjzck-tools-cli` |
| 用户 | Note / Lucky 等内部项目 | pjzck 组织下的项目 |
| 代码仓库 | `note/tools/` | `pjzck/tools/` |

架构完全对标 diago，但独立部署——两套工具链互不依赖，各自维护自己的 registry、skills 和 hooks。

## 3. 模块划分

```
pjzck-tools-cli  ←── 总入口：一键安装/管理所有工具（npm 包管理器）
    │
    ├── pjzck-skills-cli  ←── 管理 skills 远程仓库（Nexus raw repo）
    │       └── 依赖 mount-skills-cli（本地挂载，自动安装）
    │
    └── pjzck-hooks-cli   ←── 管理 hooks 远程仓库（Nexus raw repo）
            └── 依赖 mount-hooks-cli（本地挂载，自动安装）
                     │
                     └── 依赖 pjzck-common（共享 HTTP/安全/配置模块）
```

### 3.1 各模块职责

| npm 包名 | CLI 命令 | 职责 |
|----------|----------|------|
| `@pjzck/pjzck-tools-cli` | `pjzck-tools-cli` | 包管理器入口：list / install / update / uninstall / installed / self-update |
| `@pjzck/pjzck-skills-cli` | `pjzck-skills-cli` | Skills 远程仓库：setup / list / pull / publish / sync / delete / info |
| `@pjzck/mount-skills-cli` | `mount-skills` | Skills 本地挂载：软链接到 `.claude/skills/`，支持多 target |
| `@pjzck/pjzck-hooks-cli` | `pjzck-hooks-cli` | Hooks 远程仓库：setup / list / pull / publish / sync / delete |
| `@pjzck/mount-hooks-cli` | `mount-hooks` | Hooks 本地挂载：软链接 + settings.json 合并 |
| `@pjzck/pjzck-common` | — | 共享模块：HTTP / LDAP / security / config / log |

### 3.2 与 diago 的关键差异

1. **mount-skills / mount-hooks** 仍然以 `mount-skills`/`mount-hooks` 作为 CLI 命令名——这两个是通用概念，不需要加 pjzck 前缀。但 npm 包名放在 `@pjzck/*` scope 下以避免与 diago 冲突。
2. **pjzck-common** 独立维护——不依赖 `@diago/diago-common`，但代码结构可参考。
3. **Registry / LDAP 服务端点** 全部替换为 pjzck 自己的基础设施。

## 4. 目录结构

```
pjzck/tools/
├── README.md
├── scripts/                    # npm 包源码
│   ├── pjzck-common/           # 共享工具库
│   │   ├── package.json
│   │   ├── index.js
│   │   ├── log.js
│   │   ├── http.js
│   │   ├── ldap.js
│   │   ├── security.js
│   │   ├── repository-config.js
│   │   ├── browse.js
│   │   ├── discovery.js
│   │   ├── search.js
│   │   └── CHANGELOG.md
│   │
│   ├── pjzck-tools-cli/        # 总入口（包管理器）
│   │   ├── package.json
│   │   ├── cli.js
│   │   ├── README.md
│   │   └── CHANGELOG.md
│   │
│   ├── pjzck-skills-cli/       # Skills 远程仓库管理
│   │   ├── package.json
│   │   ├── cli.js
│   │   └── CHANGELOG.md
│   │
│   ├── mount-skills-cli/       # Skills 本地挂载
│   │   ├── package.json
│   │   ├── mount.js
│   │   ├── mount.sh
│   │   ├── mount.bat
│   │   ├── mount.config
│   │   └── CHANGELOG.md
│   │
│   ├── pjzck-hooks-cli/        # Hooks 远程仓库管理
│   │   ├── package.json
│   │   ├── cli.js
│   │   └── CHANGELOG.md
│   │
│   └── mount-hooks-cli/        # Hooks 本地挂载
│       ├── package.json
│       ├── mount.js
│       ├── mount.sh
│       ├── mount.bat
│       ├── mount.config
│       └── CHANGELOG.md
│
├── skills/                     # Skills 源文件
│   ├── CATEGORIES.md
│   ├── skill-create-guide.md
│   ├── local/                  # 本地 skills（不发布到远程）
│   └── shared/                 # 共享 skills（发布到远程）
│
└── hooks/                      # Hooks 源文件
    ├── .gitkeep
    └── ...（各 hook 子目录，含 hook.json + check.sh）
```

## 5. 实现计划（分 4 阶段）

### 阶段 1：pjzck-common（共享基础设施）

**目标**：可独立发包的基础模块，所有其他包依赖它。

| # | 任务 | 说明 |
|---|------|------|
| 1.1 | 创建 `pjzck-common` 包结构 | package.json、index.js |
| 1.2 | 实现 `log.js` | 统一日志（参考 diago-common/log.js） |
| 1.3 | 实现 `http.js` | HTTP GET/PUT/DELETE 封装 |
| 1.4 | 实现 `security.js` | 安全校验（路径穿越防护、safe segment 等） |
| 1.5 | 实现 `ldap.js` | LDAP 认证 |
| 1.6 | 实现 `repository-config.js` | 配置读写 + DPAPI 加密 |
| 1.7 | 实现 `discovery.js` | Nexus raw repo 用户/包列表发现 |
| 1.8 | 实现 `browse.js` | Nexus raw repo 目录浏览 |
| 1.9 | 发布 `@pjzck/pjzck-common` v0.0.1 | 到 pjzck registry |

### 阶段 2：本地挂载工具（mount-skills + mount-hooks）

**目标**：先让 skills/hooks 能本地挂载，这是最核心的使用场景。

| # | 任务 | 说明 |
|---|------|------|
| 2.1 | 创建 `mount-skills-cli` | 软链接管理、多 target、extras |
| 2.2 | 创建 `mount-hooks-cli` | 软链接 + settings.json hook 注册 |
| 2.3 | 在 pjzck 项目本地测试挂载 | 验证软链接创建/删除/remount |
| 2.4 | 发布 `@pjzck/mount-skills-cli` + `@pjzck/mount-hooks-cli` | |

### 阶段 3：远程仓库管理（pjzck-skills + pjzck-hooks）

**目标**：支持从 Nexus raw repo pull/publish/sync。

| # | 任务 | 说明 |
|---|------|------|
| 3.1 | 创建 `pjzck-skills-cli` | setup/list/pull/publish/sync/delete/info |
| 3.2 | 创建 `pjzck-hooks-cli` | 同上，针对 hooks |
| 3.3 | 配置 Nexus raw repo（需运维配合） | diago-ai/skills 和 diago-ai/hooks 对应的 pjzck 仓库 |
| 3.4 | 端到端测试 | setup → publish → pull → mount → 验证 |
| 3.5 | 发布 `@pjzck/pjzck-skills-cli` + `@pjzck/pjzck-hooks-cli` | |

### 阶段 4：总入口（pjzck-tools-cli）

**目标**：一键安装所有工具的总入口。

| # | 任务 | 说明 |
|---|------|------|
| 4.1 | 创建 `pjzck-tools-cli` | list/install/update/uninstall/installed/self-update |
| 4.2 | 发布 `@pjzck/pjzck-tools-cli` | |
| 4.3 | 端到端验证 | 新机器初始化全流程 |

## 6. 新机器初始化流程（目标体验）

```bash
# 1. 安装包管理器
npm install -g @pjzck/pjzck-tools-cli --registry=<pjzck-registry>

# 2. 一键安装全部工具
pjzck-tools-cli install pjzck-skills pjzck-hooks

# 3. 配置远程仓库访问
pjzck-skills-cli setup
pjzck-hooks-cli setup

# 4. 同步远程 skills/hooks 到本地缓存
pjzck-skills-cli sync
pjzck-hooks-cli sync

# 5. 挂载需要的 skills/hooks 到当前项目
mount-skills install pre-commit mr-text
mount-hooks install check-commit-msg
```

## 7. 命名约定

| 项目 | 约定 |
|------|------|
| npm scope | `@pjzck/*` |
| 包名格式 | `@pjzck/{pjzck-,}?{name}-cli`（管理类加 pjzck- 前缀，挂载类不加） |
| CLI 命令 | 管理类: `pjzck-{name}-cli`，挂载类: `mount-{name}` |
| npm registry | **待用户提供** |
| Nexus raw repo (skills) | **待用户提供** |
| Nexus raw repo (hooks) | **待用户提供** |
| LDAP 服务器 | **待确认**（复用 diago 的 / 独立部署？） |

## 8. 待确认事项

- [ ] 外网 pjzck tools repository 地址（npm registry + Nexus raw repo）
- [ ] LDAP 认证是复用 diago 的还是独立部署
- [ ] 首批需要哪些 skills/hooks（是否需要从 diago 迁移？）
- [ ] 是否需要在现有 diago 基础设施上扩展而非独立部署
