<div align="center">
  <img src="docs/TWB-LOGO-rounded.png" alt="TIA Workspace Bridge Logo" width="160" style="border-radius: 24px; box-shadow: 0 8px 24px rgba(0,100,110,0.25);" />
  <h1>TIA Workspace Bridge (TWB)</h1>
  <p><strong>Siemens TIA Portal 与现代 AI Coding Agent 的工业级桥梁</strong></p>
  <p>将西门子博途工程投影为声明式本地代码库，让 Cursor、Claude、Copilot、Antigravity 等通用 AI 编程助手能够像开发现代代码一样，无缝重构、静态检查、事务性同步与编译 PLC 工程。</p>

  <p>
    <a href="https://opensource.org/licenses/MIT"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT" /></a>
    <a href="https://www.microsoft.com"><img src="https://img.shields.io/badge/Platform-Windows%2010%20%2F%2011%20x64-blue.svg" alt="Platform" /></a>
    <a href="https://support.industry.siemens.com"><img src="https://img.shields.io/badge/TIA%20Portal-V18%20%7C%20V19%20%7C%20V20%20%7C%20V21-00646E.svg" alt="TIA Portal" /></a>
    <a href="https://dotnet.microsoft.com"><img src="https://img.shields.io/badge/.NET-Framework%204.8%20%2F%209.0-512BD4.svg" alt=".NET" /></a>
  </p>

  <p>
    <a href="https://gitee.com/coobi7/TIAworkspaceBridge/releases/latest"><strong>📥 Gitee 极速下载 (国内发行版)</strong></a> &nbsp;•&nbsp;
    <a href="https://github.com/coobi7/TIAworkspaceBridge/releases/latest"><strong>🌐 GitHub Releases (海外发行版)</strong></a>
  </p>
</div>

---

## 目录

- [1. 痛点与核心价值](#1-痛点与核心价值)
- [2. 核心架构原理](#2-核心架构原理)
- [3. 环境准备与编译安装](#3-环境准备与编译安装)
- [4. TWB 桌面统一驾驶舱 (GUI Cockpit)](#4-twb-桌面统一驾驶舱-gui-cockpit)
- [5. 5 分钟快速上手 (CLI 模式)](#5-5-分钟快速上手-cli-模式)
- [6. CLI 命令速查手册](#6-cli-命令速查手册)
- [7. 工作区目录与编码规约](#7-工作区目录与编码规约)
- [8. AI Coding Agent 接入指南](#8-ai-coding-agent-接入指南)
- [9. 常见问题排查 (FAQ)](#9-常见问题排查-faq)

---

## 1. 痛点与核心价值

### 传统电气工程的困境
- **工程格式封闭**：博途底层采用复合二进制工程库文件（`.ap**`），无法直接纳入 Git 进行行级 Diff、PR 评审和分支协同；
- **AI 无法深度介入**：现代大语言模型具备极强的代码理解和逻辑编写能力，但面对封闭的桌面 IDE 只能“看截图给建议”，无法直接编辑源文件或验证语法；
- **Openness 门槛极高**：原生 C# Openness API 接口复杂、COM 对象生命周期脆弱，频繁直接调用博途极易造成 IDE 崩溃或工程死锁。

### TWB 解决方案
1. **统一桌面驾驶舱（Zero-Window Switching）**：提供基于现代工业深色风格的原生 WPF 桌面应用（`twb-app.exe`），自动托管守护进程生命周期，单窗口聚合三方同步审查、Diff 预览、编译诊断、硬件选型、工站脚手架与内嵌终端。
2. **投影为文件系统（Filesystem / Git First）**：将 PLC 逻辑块（SCL/LAD/FBD）、数据块（DB）、数据类型（UDT）、变量表（Tags）以及机架网络（Hardware IaC）导出为人类与 AI 均可读的纯文本（`.s7dcl` + `YAML`）。
3. **规范化差异过滤（Canonical Normalizer）**：过滤动态编译时间戳与无序节点，连续多次 Pull 保持 git diff 干净、大幅减少无用噪声。
4. **三级渐进式开发与校验流**：
   - **Level 1 (本地静态语法检查)**：无需惊动博途，<10ms 内拦截括号未闭合、结构缺失、关键字错误；
   - **Level 2 (本地 AST 结构与符号校验)**：解析代码 AST 结构，校验块接口、参数与符号引用；
   - **Level 3 (真实编译器驱动)**：调用博途真实编译器执行增量编译，抓取结构化诊断定位至行列号。
5. **安全事务性回写（Safe Apply）**：基于指纹比对的乐观并发控制（OCC），任何人在博途在线工程做出的未同步修改都会触发并发拦截，杜绝静默覆盖。
6. **硬件声明式 Desired State 与 Catalog 反查**：穿透本机博途原生硬件库校验订货号（MLFB）与槽位约束，辅助核查订货号，减少硬件选型错误。

---

## 2. 核心架构原理

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                 AI Coding Agent / Developer (VS Code / Cursor)              │
│                       Local Filesystem & Git Workspace                      │
└──────────────────────┬───────────────────────────────┬──────────────────────┘
                       │ 命令行控制                     │ 可视化交互
                       ▼                               ▼
┌─────────────────────────────────────┐ ┌─────────────────────────────────────┐
│               twb CLI               │ │       twb-app Desktop Cockpit       │
│ status | pull | query | validate... │ │ 5大Tab + Diff审查 + 内嵌控制台终端  │
└──────────────────────┬──────────────┘ └──────────────┬──────────────────────┘
                       │                               │
                       └───────────────┬───────────────┘
                                       │ NamedPipe IPC (双向超时与锁保护)
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            twb-service Daemon                               │
│  ┌───────────────────────┐  ┌───────────────────────┐  ┌──────────────────┐ │
│  │     ModelIndexer      │  │  CanonicalNormalizer  │  │ HardwareApplier  │ │
│  │ 符号/交叉引用/IO反查  │  │   规范化差异处理器    │  │  硬件声明式 IaC  │ │
│  └───────────────────────┘  └───────────────────────┘  └──────────────────┘ │
│  ┌───────────────────────┐  ┌───────────────────────┐                       │
│  │    StateRepository    │  │   SandboxValidator    │                       │
│  │ SQLite 指纹与状态库   │  │ Level 1/2 本地校验器  │                       │
│  └───────────────────────┘  └───────────────────────┘                       │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ C# 强类型 Openness API
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   Siemens TIA Portal (V18 / V19 / V20 / V21)                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 环境准备与编译安装

### 前置要求
1. **操作系统**：Windows 10 / 11 64-bit。
2. **西门子博途**：TIA Portal V18 / V19 / V20 / V21，且已安装 Siemens Openness 组件。
3. **用户组授权**：当前 Windows 用户必须加入 **`Siemens TIA Openness`** 用户组（计算机管理 -> 本地用户和组 -> 组）。加入后注销或重启使之生效。
4. **编译工具**：[.NET SDK 9.0](https://dotnet.microsoft.com/download) 或支持 .NET Framework 4.8 的 Visual Studio / MSBuild。

### 编译项目
在项目根目录下执行：
```powershell
dotnet build TIAworkspaceBridge.sln -c Release
```
生成产物位于：
- **桌面客户端**：`src/Twb.App/bin/Release/net48/twb-app.exe`
- **CLI 命令行**：`src/Twb.Cli/bin/Release/net48/twb.exe`
- **后台守护服务**：`src/Twb.Daemon/bin/Release/net48/twb-service.exe`

> **提示**：将 `src/Twb.Cli/bin/Release/net48` 目录添加到系统 `PATH` 环境变量中，即可在任意终端直接运行 `twb`。

---

## 4. TWB 桌面统一驾驶舱 (GUI Cockpit)

> **核心理念**：让电气与自动化工程师在一个窗口内搞定所有日常操作，免开黑框启动守护进程，免在多个控制台与文件浏览器之间来回切换。采用现代浅色工业风界面（Light Industrial Elegance），并原生内置中英文（zh-CN / en-US）无感热切换。

双击启动 `twb-app.exe` 即可进入工作区驾驶舱：

### 核心功能与 5 大功能面板

#### 1. 全生命周期自动托管（Zero Setup）
- 软件启动时**自动探测并静默拉起后台服务**（`twb-service.exe`），无需手动打开控制台窗口；
- 顶部状态栏直观展示服务心跳、博途在线 PID 实例选择下拉框、当前工程与站名，支持一键热重连；
- 右上角提供 **中/EN 语言一键热切换** 与工作区路径即时重载。

#### 2. Tab 1: 🌲 博途同步项目树与源码/差异审查 (Project Explorer & Diff)
- **严格同步博途树状结构**：从博途拉取（Pull）后，工作区完整分层树状视图与博途项目树保持一致（`PLC 站` -> `程序块 (Program blocks)` / 子分组文件夹 / `S7DCL 块` -> `PLC 数据类型 (UDT)` -> `PLC 变量 (Tags)` -> `硬件与网络 (Hardware)`）；
- **对象类型与改动角标**：各节点自动标注类型胶囊（`OB`/`FB`/`FC`/`DB`/`UDT`/`TAG`/`HW`），改动项自动附带温润状态指示（`Mod`/`Conf`）；
- **双模式审查（源码查看 vs 差异比对）**：
  - **源码内容 (Source Code)**：选中任意块可直接阅读其 S7DCL / SCL / UDT 源码，无需切换到外部文本编辑器；
  - **差异比对 (Unified Diff)**：选中发生改动的项，一键比对行级变更；
- **效率工具链**：
  - 支持 **“全部文件” / “仅改动项”** 快速筛选与关键字实时检索；
  - **在文件夹中定位（Reveal in Explorer）**：一键在 Windows 资源管理器中高亮定位选中文件；
  - 支持针对当前选中单项直接触发静态校验或增量回写博途。

#### 3. Tab 2: 🩺 校验与编译器诊断 (Diagnostics & Compile)
- **三级诊断分级呈现**：集成 Level 1 静态、Level 2 本地结构反序列化以及 Level 3 博途真实增量编译告警；
- **错误行列精确定位**：表格化罗列严重级别（错误/警告/信息）、对象名、错误代码、行列号与定位建议。

#### 4. Tab 3: 🧠 代码智能与物理 IO 检索 (Code Intelligence & IO)
- **全局符号交叉引用（XRef）**：毫秒级反查任意 DB/FB/FC/UDT 的全局调用方（Callers）与被调用方（Callees），不惊动博途；
- **物理 IO 逆向映射**：输入 `%I10.0` 或 `%Q4.2`，立即检索其对应的物理模块插槽、通道以及绑定的 PLC 变量；
- **悬空通道审计（Dangling IO）**：一键扫描全工程已组态但未分配变量的物理通道，杜绝现场接线遗漏。

#### 5. Tab 4: 🛠 硬件组态与原生选型 (Hardware IaC & Catalog)
- **原生硬件库选型搜索**：直接穿透调用博途原生硬件目录，输入关键字（如 `ET 200SP`、`1515-2`）检索真实合法的 MLFB 订货号与版本号，有效规避硬件订货号录入错误的隐患；
- **硬件 IaC 状态机**：查看本地声明式硬件 YAML 与博途机架的比对差量，并一键触发声明式自动装配。

#### 6. 底部抽屉：内嵌终端控制台与实时事件流 (Embedded CLI)
- **统一活动日志流**：毫秒级显示后台守护进程与博途交互事件（INFO/WARN/ERROR/SUCCESS）；
- **内嵌 CLI 交互**：底部提供 `twb >` 交互命令行，支持输入任意 `twb` 指令（如 `twb status`、`twb query refs Main`），按回车直接执行并将标准输出实时流式呈现在界面中，无需打开外部终端！

---

## 5. 5 分钟快速上手 (CLI 模式)

如果您更倾向于全命令行操作或配合无界面环境：

### 步骤 1: 启动后台服务守护进程
```powershell
twb-service.exe
```

### 步骤 2: 打开博途工程并检查连通性
打开博途软件并加载任意 S7-1500 / S7-1200 工程，运行：
```powershell
twb status
```

### 步骤 3: 投影导出至本地工作区
```powershell
twb pull
```

### 步骤 4: 本地编辑或使用 AI 助手重构
编辑 `plc/<Station>/blocks/` 下的 SCL 逻辑，或修改 `plc/<Station>/tags/*.yaml` 变量表。

### 步骤 5: 验证与事务回写
```powershell
# 1. 本地快速静态与结构校验
twb validate

# 2. 生成变更计划（查看 OCC 指纹与改动清单）
twb plan

# 3. 事务性应用回写到博途正式工程
twb apply --yes
```

---

## 6. CLI 命令速查手册

### 工程状态与服务
```powershell
# 检查工程连通性、三方同步状态（CLEAN/CHANGED/CONFLICT）
twb status

# 重连正在运行的博途实例（支持指定 PID）
twb reconnect
twb reconnect --pid 17316

# 查看当前运行中的博途进程清单
twb processes

# 停止后台守护进程
twb daemon stop
```

### 快速代码智能查询 (不唤醒博途，毫秒级响应)
```powershell
# 查询符号或块的全局调用链与交叉引用
twb query refs "DB_CylinderData"

# 查询物理硬件地址的槽位绑定与对应 PLC Tag
twb query io "%I10.0"

# 审计全工程未绑定变量的悬空物理 IO 通道
twb query dangling-io
```

### 校验与真实编译
```powershell
# 本地只读校验（支持指定文件或全部改动）
twb validate
twb validate plc/PLC_1/blocks/FB_CylinderControl.s7dcl

# 触发博途真实增量编译（Level 3 编译，输出错误行列号）
twb compile
```

### 事务性应用 (Safe Apply)
```powershell
# 生成应用变更计划（执行 OCC 乐观并发安全比对）
twb plan

# 安全回写并触发编译复核
twb apply --yes

# 声明式硬件比对与装配回写
twb hardware plan
twb hardware apply
```

### 硬件目录反查 (规避 AI 订货号幻觉)
```powershell
# 实时检索博途本机安装硬件库中的合法 MLFB 订货号与版本
twb catalog search "ET 200SP"
twb catalog search "6ES7 131-6BF"
```

---

## 7. 工作区目录与编码规约

### 目录结构
```text
MyProject/
├─ tia-project.yaml            # TWB 工程绑定元数据
├─ AGENTS.md                   # AI 编程助手行为准则（供 Agent 快速接入）
├─ plc/<StationName>/          # PLC 站源码
│  ├─ blocks/                  # 逻辑块源码 (.s7dcl, .s7res)，保持博途分组文件夹层级
│  ├─ db/                      # 数据块定义 (.s7dcl, .s7res) 与多实例背景块
│  ├─ datatypes/               # 用户自定义数据类型 UDT (.s7dcl, .s7res)
│  └─ tags/                    # 声明式 PLC 变量表 (*.yaml)
├─ hardware/                   # 硬件组态声明 (*.yaml, profinet.yaml, io-map.csv)
├─ model/                      # 本地衍生只读代码模型 (symbols.json, callgraph.json)
└─ .twb/                       # TWB 内部运行时状态库 (包含 state.db 与 baseline，严禁手改)
```

### 关键语法与编码红线
1. **文件编码**：所有 `.s7dcl` 与 `.s7res` 文件必须使用 **UTF-8 with BOM** 保存。
2. **SIMATIC SD Document 格式 (`.s7dcl`)**：
   - 块头属性放在首部花括号：`{ S7_BlockNumber := "10"; S7_Optimized := "TRUE"; S7_PreferredLanguage := "SCL"; S7_Version := "0.1" }`
   - **UDT**：使用 `TYPE Name : STRUCT ... END_STRUCT; END_TYPE`（不写 `TITLE =` 或结构内 `VERSION`）。
   - **DB**：声明式 `DATA_BLOCK "Name" VAR ... END_VAR END_DATA_BLOCK`（不写 `BEGIN`）。
   - **SCL 代码逻辑**：包裹在 `{ S7_Language := "SCL" } NETWORK ... END_NETWORK` 内。
3. **伴随资源 `.s7res`**：每个源码文件均有对应的伴随资源文件。若无多语言注释需求，内容保持 `MultiLingualTexts: []`。

---

## 8. AI Coding Agent 接入指南

若要使用 Cursor、Claude Desktop 或其他 AI Coding Agent 接手管理博途工程：

1. **系统提示词与规则**：
   将工作区根目录下的 [`AGENTS.md`](./AGENTS.md) 作为 Prompt 或加载到 IDE 的上下文配置中（例如 Cursor Rules / `.cursorrules`）。
2. **推荐的 Agent 操作闭环**：
   - **第一步（感知）**：Agent 运行 `twb status` 与 `twb query refs` 摸清上下文；
   - **第二步（改动）**：直接在 `plc/` 或 `hardware/` 下编辑文件；
   - **第三步（校验）**：Agent 运行 `twb validate`，如果有报错阅读错误行列号自行修正；
   - **第四步（提审）**：Agent 运行 `twb plan` 展示给工程师变更集；
   - **第五步（落地）**：执行 `twb apply --yes`，完成博途写入并更新基线。

---

## 9. 常见问题排查 (FAQ)

#### Q1: 运行 `twb status` 提示 `Siemens TIA Openness group unauthorized`？
- **解决**：打开 Windows“计算机管理” -> “本地用户和组” -> “组”，找到 `Siemens TIA Openness`，将当前登录用户加入该组。配置后需要**注销 Windows 当前会话重新登录**方可生效。

#### Q2: 提示 `Could not connect to TWB Daemon`？
- **解决**：TWB 采用 Client-Daemon 架构以实现极速响应与 COM 隔离保护。如果您运行 `twb-app.exe` 桌面客户端，应用会自动启动并托管守护进程；如果您在独立终端中使用 `twb` CLI，请确保在后台运行了 `twb-service.exe`。

#### Q3: 增量写入时提示 `OCC Conflict (Optimistic Concurrency Control)`？
- **解决**：说明自上次 `twb pull` 后，有工程师在博途在线工程中直接修改了该块，或者其他进程修改了同一目标。TWB 为保障生产安全直接拦截覆盖。此时运行 `twb pull` 刷新基线，比对差异后再行应用。

#### Q4: 中文注释导入博途后乱码或报错？
- **解决**：请检查源文件是否保存为 **UTF-8 with BOM**（字节序标记 `EF BB BF`）。博途 Openness 对无 BOM 的 UTF-8 中文字符解析存在限制。

---

## 许可证

本项目采用 [MIT 许可证](./LICENSE) 开源发布。
