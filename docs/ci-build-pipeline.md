# 通用 CI / 构建 / 发布流水线设计

> **定位**: 面向「monorepo（前端 + 后端 + 桌面客户端）+ GitHub Actions」形态项目的持续集成与产物发布链路设计文档。所有命名均可按项目替换：`<app>` 为产品名，`packages/*` 为工作区包，`desktop/` 为桌面端目录。  
> **配套形态**: `.github/workflows/`（验证 + 发布编排 + 平台构建 + 站点部署，若干 workflow）+ 平台构建脚本（mac/win 各一份）+ 桌面打包配置。  
> **核心理念**: 验证与发布彻底分层；发布是显式动作；平台构建差异大则独立 workflow 扇出；无证书预算下用 ad-hoc / 未签名 + 完整性校验给出可验证锚点。

---

## 0. 一句话结论

流水线是一条 **「验证（CI）→ 编排（Release）→ 平台构建（mac/win/linux）→ 分发（Release assets / Pages / Artifacts）」** 的多层链路：验证层纯检查、零产物；发布编排层在 `release.published` 时解析版本号并扇出到各平台可复用 workflow（也支持 `workflow_dispatch` dry-run，只产 artifact 不碰 release）；平台构建层各自负责「自包含运行时组装 → 安装介质生成 → 条件上传」；mac 无证书时 ad-hoc 签名（把 Gatekeeper 从「已损坏」降级为「右键打开」），win 无证书时产出未签名安装包，linux 无需签名。全链路以一个版本号为唯一传递参数，配 SHA256SUMS + OIDC attestation（可选）作完整性锚点。

---


## 1. 全景架构

```
┌────────────────────────────────────────────────────────────────────┐
│ L0  验证层    (ci.yml)                                              │
│      lint / build(多包) / 大规模测试分片(plan→lanes→summary)          │
│      串行兜底 + 证据门禁 job + 平台专属冒烟 + 目录规模守卫              │
│      纯验证、零产物、不触发发布                                        │
└────────────────────────────────────────────────────────────────────┘
                        │  release.published（打 tag 后发 Release 才触发）
                        │  workflow_dispatch（手动 dry-run，只产 artifact）
                        ▼
┌────────────────────────────────────────────────────────────────────┐
│ L1  发布编排层  (release-desktop.yml)                                │
│      resolve-version ──┬──▶ build-<mac>    (workflow_call)          │
│                        ├──▶ build-<win>    (workflow_call)          │
│                        └──▶ build-<linux>  (workflow_call, 可选)     │
└────────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────────┐
│ L2a macOS          │ │ L2b Windows       │ │ L2c 其他/站点          │
│ 架构矩阵:          │ │  windows-latest   │ │ deploy-pages.yml     │
│  arm64→macos-latest│ │  Inno Setup 编译  │ │ (纯静态站点时)        │
│  x64→macos-15-intel│ │  .exe + 便携 .zip │ │ configure/upload/    │
│  自建 DMG          │ │  资源泄漏守卫      │ │ deploy-pages 三步     │
└──────────────────┘ └──────────────────┘ └──────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌────────────────────────────────────────────────────────────────────┐
│ L3  分发层                                                          │
│     release.published → softprops/action-gh-release attach 到触发    │
│     dispatch dry-run  → upload-artifact 仅存 workflow artifact      │
│     完整性: SHA256SUMS（可选）/ attest-build-provenance（可选）        │
└────────────────────────────────────────────────────────────────────┘
```

**文件与职责映射（模板）**：

| 文件                                          | 层      | 职责                             |
| ------------------------------------------- | ------ | ------------------------------ |
| `.github/workflows/ci.yml`                  | L0     | 日常验证（lint / build / 测试证据 / 守卫） |
| `.github/workflows/windows-smoke.yml`       | L0     | 仅 Windows 能验的平台专属测试            |
| `.github/workflows/release-desktop.yml`     | L1     | 发布编排（版本解析 + 平台扇出）              |
| `.github/workflows/build-mac-<ext>.yml`     | L2a    | macOS 构建（架构矩阵 + 上传分流）          |
| `.github/workflows/build-windows-<ext>.yml` | L2b    | Windows 构建（安装器 + 便携包）          |
| `.github/workflows/deploy-pages.yml`        | L2c/L3 | 静态站点/文档站发布（可选）                 |
| `<desktop>/scripts/build-<os>.sh/.ps1`      | L2     | 平台构建核心脚本（自包含运行时 + 安装介质）        |
| `<desktop>/package.json`（build 字段）          | L2     | 桌面打包配置（签名开关、extraResources）    |
| `<desktop>/installer/*.iss`                 | L2b    | Windows Inno Setup 安装脚本        |

---

## 2. L0 验证层（ci.yml）

### 2.1 触发与并发

| 触发                                                        | 语义          |
| --------------------------------------------------------- | ----------- |
| `push` 到默认分支 + 任意 `pull_request`                          | 常规验证        |
| `paths-ignore`（`docs/**`、`*.md`、`designs/**`、`assets/**`） | 文档/设计类改动不排队 |

- `concurrency.group` 按 ref 分组 + `cancel-in-progress: true`：同一分支连续 push 只保留最新一次。
- `permissions: contents: read`（最小权限）。

### 2.2 job 拓扑原则

验证层拆成**多个单一职责 job** 而非一个大 job：

| job        | 检查内容                                        | 备注                                         |
| ---------- | ------------------------------------------- | ------------------------------------------ |
| lint       | 格式/静态检查（全仓或按包）                              | 最快，先失败先反馈                                  |
| build      | 各工作区包顺序构建（shared → api → web …）             | 依赖顺序即构建顺序，防「本地能过 CI 不过」                    |
| test（聚合门禁） | `needs` 全部测试 job，`always()` 下校验各 job result | **证据门禁**：只允许「预期的成功/跳过组合」通过，杜绝测试 job 集体消失还绿 |


### 2.3 大规模测试分片编排（plan → lanes → summary 模式）

当单仓库测试量大、需并行压缩时间时，用**三段式确定性分片**，而不是把文件硬编码进 matrix：

```
prepare ──▶ (输出 plan.json) ──▶ shards (matrix lanes 并行跑各自切片)
   │                                 │
   └──── 同时探测: 分片契约是否齐备     │
         ──▶ 齐备 → mode=sharded        │
             缺失 → mode=serial(完整串行兜底) ──▶ serial job
                                         ▼
                                   summary（下载 plan + 全部 lane 报告，
                                   证明覆盖率完整 + 汇总耗时）
```

| 环节       | 职责                                                                           | 关键点                                              |
| -------- | ---------------------------------------------------------------------------- | ------------------------------------------------ |
| prepare  | 探测「分片契约」（plan/shard/summary 脚本 + package.json 命令是否齐全）；齐备则构建精确输入并产出 plan.json | 契约缺失时**自动退化为串行**，CI 不红——上线分片是渐进式的                |
| shards   | `matrix.lane`（如 `[serial, pure-1…pure-4]`）并行执行，各自写 lane 报告                   | `fail-fast: false`；报告 `if: always()` 上传（失败也要留证据） |
| summary  | 下载 plan + 全部 lane 报告，运行汇总脚本                                                  | 校验无遗漏（覆盖率证明），产出 evidence                         |
| serial   | 契约缺失时的完整串行兜底                                                                 | 与 sharded 模式互斥（`if` 按 mode 分流）                   |
| test（门禁） | 按 mode 断言各 job 状态组合                                                          | 如 sharded 模式要求 shards=success、serial=skipped     |

**为什么要一个显式门禁 job**：GitHub Actions 中若分片 job 因 `if` 条件整体跳过，`needs` 链会自动「绿」；显式断言各上游 result 的组合，把「没跑也算过」堵死。

### 2.4 平台专属冒烟（windows-smoke.yml）

凡存在「只能在某 OS 上验证」的测试（进程树清理、路径/桩解析、平台守卫分支），独立一个平台冒烟 workflow：

- 触发同 L0（push/PR），路径忽略同 L0。
- `runs-on: windows-latest`（或 macos-latest）。
- 步骤：frozen install → 构建依赖包 → **平台专属测试子集**（不必全量，只跑平台敏感项）。

**注意**：它不该与 ci.yml 共享 matrix 塞进同一 job——平台专属测试放通用 runner 上跑会直接失败或空转，独立 workflow 让平台语义清晰。

### 2.5 目录规模 / 仓库健康守卫（可选）

`bash scripts/check-dir-size.sh` 一类守卫 job：对关键目录（如 node_modules 膨胀、dist 残留、源码目录失控）做阈值告警，防止仓库静默膨胀。属低成本防腐，规模敏感项目可留。

---

## 3. L1 发布编排层（release-desktop.yml）

### 3.1 触发与权限

| 触发                                | 语义                                                                      |
| --------------------------------- | ----------------------------------------------------------------------- |
| `release.types: [published]`      | **打 tag 不触发**；必须「打 tag + 建 Release + 点 Publish」才触发，产物 attach 到该 release |
| `workflow_dispatch`（输入 `version`） | 手动 dry-run：填版本号，平台冒烟打包，产物只作 workflow artifact，**不碰任何 release**          |

- `permissions: contents: write`（attach assets 需要）。
- `concurrency` 按 release tag / 输入版本分组 + `cancel-in-progress: false`：同一 release 的发布不互相取消（发布中断是危险动作）。

### 3.2 版本解析（resolve-version）

单一职责 job：

```
release 模式 → 取 github.event.release.tag_name
dispatch 模式 → 取手动输入的 version
统一 strip 'v' 前缀 → 写 GITHUB_OUTPUT.version
```

**版本号是全链路唯一传递参数**：各平台 job 从 `needs.resolve-version.outputs.version` 消费，禁止各端自行推断。

### 3.3 扇出到可复用 workflow

```yaml
build-mac:      needs: resolve-version, uses: ./.github/workflows/build-mac-<ext>.yml
build-windows:  needs: resolve-version, uses: ./.github/workflows/build-windows-<ext>.yml
```

- 用 `workflow_call` 三个独立 job 而非一个大 matrix：**平台构建步骤差异大**（runner、证书、安装介质完全不同），matrix 强行合并只会让条件分支爆炸。
- 每个可复用 workflow 同时声明 `workflow_dispatch`（支持单独对某平台 dry-run，含单架构选择）。
- 传参两个：`version`（必填）、`upload-to-release: ${{ github.event_name == 'release' }}` —— 同一流水线双模式：release 事件才 attach，dry-run 只留 artifact。
- 子 workflow 内设 `permissions: contents: write`（`secrets: inherit` 可按需）。

---

## 4. L2 平台构建层

**目录规划原则**（避免多平台产物互相污染）：

```
dist/                 # 顶层安装介质输出（脚本负责，各平台自建子目录或独立命名）
<desktop>/dist/       # electron-builder 中间产物（.app / win-unpacked）
bundled/              # 自包含运行时组装区（deploy / node / redis / archives）
```

中间产物与最终介质分离；CI 上传只消费最终介质目录。

### 4.1 macOS（build-mac workflow）

| 维度        | 取值                                        | 理由                                                                               |
| --------- | ----------------------------------------- | -------------------------------------------------------------------------------- |
| 架构        | arm64 + x64 并行 matrix                     | 双 job 并行提速                                                                       |
| runner    | arm64→`macos-latest`；x64→`macos-15-intel` | 各自原生架构，避免交叉（交叉需 Rosetta，纯 JS 项目可选交叉降级）                                           |
| 动态矩阵      | `arch: ${{ fromJSON(...) }}`              | dispatch 选单架构时构建 `["arm64"]` 单元素矩阵——在 matrix 展开前解析，避免可复用 workflow 中引用 matrix 的报错 |
| fail-fast | `false`                                   | 一个架构失败不拖累另一个                                                                     |

**流程**：checkout（fetch-depth 0）→ 同步 `desktop/package.json` 版本 → setup pnpm/node → 调 `desktop/scripts/build-<os>.sh --arch <arch>` → 按 `upload-to-release` 分流（release attach / artifact）。

### 4.2 Windows（build-windows workflow）

| 维度     | 取值                                                         | 理由                                      |
| ------ | ---------------------------------------------------------- | --------------------------------------- |
| runner | `windows-latest`                                           | 原生编译安装器                                 |
| 工具链    | Inno Setup（`choco install innosetup`）                      | windows-latest 不自带；安装后 `iscc.exe /?` 自检 |
| 版本     | 同步 `desktop/package.json` + env 注入 `.iss` 的 `MyAppVersion` | CI 从 release tag 注入，本地默认读 package.json  |
| 产物     | `Setup-<version>.exe`（安装器）+ 便携 `.zip`                      | 两种分发形态                                  |
| 预检     | 打包前跑桌面配置行为测试 + 平台资源泄漏守卫                                    | 见 §5.4                                  |

### 4.3 站点/文档站发布（deploy-pages.yml，可选）

纯静态站点用官方三步，不用自己打包上传：

```
build:  configure-pages → upload-pages-artifact(path: ./<site>)
deploy: needs build → deploy-pages（environment: github-pages）
```

- 触发：仅当站点目录或该 workflow 变更（`paths`），或手动 dispatch。
- 权限：`contents: read, pages: write, id-token: write`。

---

## 5. 构建脚本内部流水线（平台构建的核心资产）

平台 workflow 只做「环境准备 + 调用脚本 + 上传」；真正重活（依赖组装、介质生成）放在**每平台一个的构建脚本**里，本地也能跑（带 skip 开关做增量调试）。

### 5.1 步骤总览（mac / win 同构映射）

| 步  | macOS（build-mac.sh）                                                                               | Windows（build-desktop.ps1）                            | 目的                       |
| -- | ------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ------------------------ |
| 1  | pnpm install(frozen) + 构建 web                                                                     | 同左                                                    | 产出前端构建产物（.next / dist 等） |
| 2  | `pnpm --filter <pkg> --prod --config.node-linker=hoisted deploy` 到 `bundled/deploy/{api,web,...}` | 同左                                                    | **自包含运行时依赖树**（关键，见 5.2）  |
| 3  | 捆绑 Node.js portable（arm64+x64，`nodejs.org` 下载）                                                    | 捆绑 Node portable（win-x64 zip）                         | 干净机器无系统 Node 也能起子进程      |
| 4  | Redis portable：**源码编译**（`download.redis.io` tar → make）                                           | Redis portable：下载 **msys2 预编译**（GitHub release asset） | mac 无官方预编译二进制；win 有      |
| 5  | （mac 跳过 CLI 捆绑：DMG 无 post-install 阶段）                                                             | 写 CLI 安装指引（CLI 由用户自装）                                 | 避免白占体积                   |
| 6  | electron-builder `--mac dir`（先只出 .app）                                                            | electron-builder `--win --dir`（win-unpacked）          | 先出未压缩 bundle             |
| 6b | **ad-hoc codesign**（`codesign -s - --deep --force`）                                               | —（无证书则未签名）                                            | mac Gatekeeper 降级        |
| 7  | 自建 DMG（hdiutil 两阶段）                                                                               | tar.gz 归档（加速 Inno 解压）                                 | 见 5.3                    |
| 8  | —                                                                                                 | Inno Setup 编译 .exe + 组装便携 zip                         | 安装器 + 便携                 |

每个脚本以 `--skip-*` / `-Skip*` 开关暴露中间步骤，本地二次打包可复用已有产物，CI 全量跑。

### 5.2 自包含运行时组装：pnpm deploy 的正确姿势

桌面应用要离线自包含，必须把**运行时依赖树**随包发布。关键坑：

| 方案                                         | 问题                                                           |
| ------------------------------------------ | ------------------------------------------------------------ |
| tar 整个根 node_modules                       | pnpm workspace 在 Windows 用 **junction**，tar 会烘焙构建机绝对路径，换机器全断 |
| `pnpm deploy --config.node-linker=hoisted` | 产出**扁平、真实文件、自包含**的 node_modules，可跨机器搬运                       |

- 按包逐个 deploy（api / web / …）到 `bundled/deploy/<pkg>`。
- **deploy 不搬运 package.json `files` 字段之外的构建产物**（如 `.next`）——需手动注入。
- Windows 上 Defender 可能锁 `.bin` shim 导致 EPERM → 加排除目录 + 重试（3 次带退避）。
- **Node 版本 ABI 纪律**：捆绑 Node 的 major 必须 = 构建机 Node major，否则原生模块（如 better-sqlite3）NODE_MODULE_VERSION 不匹配，运行时崩。脚本先探测构建机版本，已捆绑版本不匹配则重下。

### 5.3 安装介质生成（两平台的「手工活」）

**macOS — 自建 DMG（hdiutil 两阶段）而非 electron-builder dmg**：

- 触发原因：bundle 巨大（afterPack 注入整棵 node_modules）时 electron-builder 内置 dmgbuild 会**低估磁盘镜像容量** → 「No space left on device」。自建流程可控。
- 流程：`hdiutil create` 可写 UDRW → mount（**显式唯一 mountpoint**，规避同名卷冲突与 stdout 解析脆弱）→ AppleScript 配 Finder 图标布局（写 .DS_Store）→ `sync`+sleep 让 Finder flush → **detach 必须成功**（失败重试一次再 fail-hard）→ `hdiutil convert` UDZO（zlib 压缩）出正式 DMG。
- 失败路径纪律：detach 失败时**绝不 `rm -rf` mountpoint**（macOS 会删到还在挂载的活动卷内容）——留给操作员手动 eject 后重跑。
- Finder 布局 + 引导箭头背景（可纯 Python 标准库生成 PNG，免外部依赖）：引导用户把 .app 拖入 Applications，否则用户双击卷内 app 会锁住卷无法弹出。

**Windows — tar.gz 归档 + Inno Setup**：

- Inno Setup 逐文件解 3 万+ node_modules 文件 → NTFS 元数据 + Defender 实时扫描 → 安装 10+ 分钟。**先 tar.gz 归档（5 个：deploy 各包 + electron + node），post-install 用系统 tar.exe 解压** → [Files] 从 ~3 万降到 ~100 条目。
- 便携 zip 与 .iss [Files] 布局保持一致（同一份布局两份消费），组装时版本号烘焙进 staging 的 package.json，防止归档名与配置版本不一致。

### 5.4 平台资源泄漏守卫（config drift 的快速失败）

electron-builder 的 `extraResources` 若未按平台拆分，会把 mac 专属资源（`node-darwin-*` / `redis-darwin-*`）带进 Windows 包——安装包到用户机器首启即崩。

- 打包后**递归扫描 win-unpacked/resources**，正则匹配 `node-darwin|redis-darwin|darwin-<arch>` 等泄漏形状 → 命中即 fail-fast。
- 匹配用子串而非路径分隔符，避免 Windows `\` / POSIX `/` 方向问题。

---

## 6. 版本链（贯穿全链路的唯一参数）

```
release tag (v1.2.3) 或 dispatch 输入 (1.2.3)
   │
   ├─ release-<desktop>.yml: resolve-version → strip 'v' → GITHUB_OUTPUT.version
   │
   ├─ build-<mac|win>.yml: inputs.version
   │     └─ 同步 desktop/package.json 版本（node -e 改写，CI 每次 fresh checkout 故安全）
   │           + env 注入（mac: <APP>_VERSION；win: 供 .iss 的 MyAppVersion）
   │           └─ 产物命名含版本号（<App>-<v>-<arch>.dmg / <App>-Setup-<v>.exe / <App>-<v>.zip）
   │
   └─ 打包脚本读 package.json version / env 覆盖 → 所有介质命名一致
```

**版本注入两种姿势（选一）**：

| 姿势                                                     | 优点                                     | 注意                                |
| ------------------------------------------------------ | -------------------------------------- | --------------------------------- |
| CI 内 `node -e` 改写 `desktop/package.json`               | 一处覆盖，electron-builder / .iss / 归档命名全读到 | 只允许在 CI 的 fresh checkout 里做；本地绝不改 |
| electron-builder `-c.extraMetadata.version=<v>` CLI 注入 | 源文件零改动，无文本改写脆弱性                        | .iss / 便携归档需另行喂版本（env）            |

关键：**所有消费方必须从同一条版本链取值**（resolve-version → env/改写 → 命名），禁止各处各自读 package.json 默认值导致漂移。

---

## 7. 签名与信任模型

### 7.1 决策总览

| 平台        | 有证书时                    | 无证书时（默认）                                                                                                                    | 用户侧效果                                   |
| --------- | ----------------------- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| macOS     | Developer ID 正式签名（可选公证） | **ad-hoc**（`codesign -s - --deep`，electron-builder `identity: null` + `hardenedRuntime: false` + `gatekeeperAssess: false`） | Gatekeeper「未识别开发者」→ 右键 → 打开 首次放行；非「已损坏」 |
| Windows   | signtool 代码签名           | 未签名安装包                                                                                                                      | SmartScreen「未知发布者」，下载可能二次警告             |
| Linux     | —（无需签名）                 | —                                                                                                                           | 包管理器正常安装                                |
| 完整性（可选补位） | —                       | SHA256SUMS + OIDC attest-build-provenance                                                                                   | 用户可校验下载物与构建来源                           |

### 7.2 为什么 mac 用「electron-builder 不签 + 脚本后签 ad-hoc」

- **ad-hoc 不是安全签名**：无身份背书，作用仅是把 Gatekeeper 从「已损坏」（完全打不开）降级为「未识别开发者」（右键放行）——本质是「免损坏标记」。
- 大 bundle 下 electron-builder 内置签名（`identity: "-"`）会同时打开 bundle 内所有文件（afterPack 注入整棵 node_modules 后上万文件）→ 撞 EMFILE。**设 `identity: null` 跳过内置签名，再用 `codesign -s - --deep --force` 后签**，codesign 自己管理 fd，不会 EMFILE。
- 后签校验用 `codesign --verify --deep`（基础验证）；`--strict` 会拒绝 bundle 内 symlink（scripts/node_modules → 外部路径），故不用。
- **必须关 `hardenedRuntime`**：未公证的 hardened-runtime app 在 Gatekeeper 下右键也无法放行，会直接拒绝。
- 零成本、本地与 CI 行为一致。

### 7.3 Windows 无证书路径

- 无 `CSC_LINK` → electron-builder 跳过 signtool → 未签名安装包。SmartScreen 提示属预期降级。
- 演进（若要正式签名）：CA/B Forum CSC-17（2023-06 起）要求 OV/EV 私钥存 FIPS 140-2 L2+ 硬件且不可导出——旧「.pfx 塞 Secrets」对 2023 后签发证书已不可行，需云端签名服务或自托管 runner 插 USB token。

### 7.4 威胁模型（诚实评价）

| 威胁                  | 能防吗  | 说明                                   |
| ------------------- | ---- | ------------------------------------ |
| 打包 bug 导致 bundle 损坏 | 部分   | ad-hoc 让 Gatekeeper 报「未识别开发者」而非「已损坏」 |
| 恶意第三方篡改安装包          | 不能   | ad-hoc/未签名无身份，攻击者可重签                 |
| 冒充官方分发              | 部分   | attestation 绑定仓库 + 官方 release URL 可信 |
| 供应链（依赖投毒）           | 不在本层 | frozen-lockfile 锁依赖；无 SBOM 时留演进项     |
| 传输损坏                | 能    | SHA256SUMS 精确校验                      |

**结论**：无正式证书时信任根 = 官方 release 页面 URL + 构建来源证明 + 用户校验习惯；规模化分发后应补正式签名（且 mac 需公证 + hardened runtime + entitlements 才能启用自动更新）。

---


## 8. 关键设计决策清单

| #   | 决策       | 选型                                              | 弃用方案               | 理由                                                 |
| --- | -------- | ----------------------------------------------- | ------------------ | -------------------------------------------------- |
| D1  | 发布触发     | `release.published` + dispatch 双模式              | push tag 即构建       | 打 tag 只是「预发布」，发 Release 才是「真发布」；dispatch 可 dry-run |
| D2  | 平台结构     | workflow_call 多 job 扇出 + `upload-to-release` 传参 | matrix 单 job       | 平台差异大，matrix 条件爆炸；一个布尔参数切换 attach/artifact 双模式     |
| D3  | 验证/发布分离  | L0 纯验证零产物                                       | 验证链顺带发布            | 日常提交永不误发布                                          |
| D4  | 大测试集编排   | plan → lanes → summary + 显式证据门禁                 | 硬编码文件到 matrix      | 分片渐进上线（契约缺失自动串行）；门禁堵死「没跑也算过」                       |
| D5  | 自包含运行时   | `pnpm deploy`（hoisted 扁平）逐包                     | tar 根 node_modules | junction 绝对路径问题；扁平树可跨机                             |
| D6  | 运行时 Node | 捆绑 portable，major=构建机                           | 依赖系统 Node          | 干净机器可跑；ABI 匹配防原生模块崩                                |
| D7  | mac 签名   | EB 不签 + 脚本后签 ad-hoc                             | EB 内置 ad-hoc       | 大 bundle EMFILE；ad-hoc 降级 Gatekeeper 语义            |
| D8  | mac DMG  | 自建 hdiutil 两阶段                                  | EB 内置 dmg          | dmgbuild 低估容量；两阶段可控 Finder 布局                      |
| D9  | win 安装器  | tar.gz 归档 + Inno Setup                          | 逐文件打包              | 3 万+ 文件解压 10+ 分钟问题                                 |
| D10 | 资源泄漏     | 打包后扫描守卫 fail-fast                               | 无                  | extraResources 跨平台混入 = 用户机首启即崩                     |
| D11 | 版本注入     | resolve-version 单一来源 → env/改写                   | 各端自行读默认值           | 全链路唯一版本参数，防命名漂移                                    |
| D12 | 完整性补位    | SHA256SUMS + attestation（可选）                    | 无                  | 零证书预算下的可验证锚点                                       |
| D13 | 平台专属测试   | 独立冒烟 workflow                                   | 塞进通用 job           | Linux/mac runner 跑不了或空转                            |
| D14 | 站点发布     | 官方 pages 三步                                     | 自打包上传              | 官方 action 最小实现                                     |

---

## 9. 简化取舍（按项目规模可裁剪）

| 可不做                         | 何时可裁                    | 重新评估触发点                 |
| --------------------------- | ----------------------- | ----------------------- |
| 大规模测试分片（plan/lanes/summary） | 测试集分钟级完成时，退化回单 job 串行   | 测试时长进入两位数分钟             |
| 自建 DMG（hdiutil + Finder 布局） | bundle 规模小、EB dmg 不炸容量时 | bundle 注入大 node_modules |
| tar.gz 归档加速                 | 文件数不破万时                 | 安装时长 > 5 分钟             |
| attestation / SHA256SUMS    | 仅内部自用不分发时               | 对外分发即补                  |
| 目录规模守卫                      | 仓库规模稳定时                 | 镜像体积 / 检出时长异常           |

**建议默认形态**：`ci.yml`（lint/build/单 job 测试/守卫）+ `windows-smoke.yml`（有 win 专属测试时）+ `release-desktop.yml`（有桌面端时）+ 平台构建 workflow ×2 + 构建脚本 ×2；测试量大再按 §2.3 引入分片。

---

## 10. 发布 SOP（验收清单）

### 10.1 dry-run 验收（workflow_dispatch，不发布）

1. Actions → `release-desktop` → Run workflow → 填 `version`（如 `0.0.0-dryrun`）。
2. 预期：resolve-version → 平台并行构建 → 产物上传为 workflow artifact（**不产生任何 release**）。
3. 核对每个 artifact：产物文件名带注入版本号；介质可安装/可挂载。

### 10.2 正式发布验收（release.published）

1. `git tag vX.Y.Z && git push --tags` → GitHub 建 Release（指向该 tag）→ 点 **Publish**。
2. 预期：平台并行构建 → 产物 attach 到该 release。
3. release 页面核对：每平台产物齐全、文件名版本正确、每个产物有 attestation 标记（若启用）。
4. 安装验收：mac 右键 → 打开（ad-hoc 首次放行）；win 出现 SmartScreen（未签名预期）；linux 直接安装。

### 10.3 发布前检查（三条 gate）

- ① 版本号唯一来源已解析（tag/输入一致，strip `v`）；
- ② CHANGELOG 存在且含该版本段落（若项目维护 CHANGELOG）；
- ③ 本地验证层全绿（lint/build/测试），发布链只做打包不做质量判断。

---

## 11. 已知限制与演进建议

1. **无发布者身份**（无证书默认态）：ad-hoc/未签名无密码学身份，任何能触达 release 流程的账号都可能被钓鱼冒充——用 attestation + SHA256SUMS 缓解，规模化后应上正式签名。
2. **macos-15-intel 依赖 runner 供给**：若该标签下线，x64 需回退 arm64 runner 交叉打包（纯 JS 项目无原生依赖时可行，略慢）。
3. **公证缺失**：ad-hoc 仅免「已损坏」；面向大众分发时公证 + hardened runtime + entitlements 是硬门槛，也是自动更新的前置。
4. **分片契约的维护面**：plan/shard/summary 三脚本 + package.json 命令需同步演进；契约探测脚本本身要保持「缺失即串行」的兜底语义。
5. **构建脚本平台绑定**：mac/win 两脚本步骤同构但实现分叉（源码编译 vs 预编译下载、hdiutil vs Inno），改动需双端对齐——建议以本文 §5.1 映射表为同步锚点。
6. **attestation 只在 GitHub 内有效**：离开 GitHub 分发时无法核验，需依赖 SHA256SUMS + 正式签名。
7. **Redis 供应链**：win 侧依赖第三方 msys2 预编译 release、mac 侧依赖源码下载——均无官方签名背书；预算允许时自托管构建产物并哈希固化。

---

