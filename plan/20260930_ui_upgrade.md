# QSanguosha UI / 构建兼容性升级计划

**文档日期**：2026-09-30  
**适用仓库**：本地 fork（`/home/guancheng/test/QSanguosha`）  
**目标约束**：视觉效果尽量与原版一致、功能不减少、Windows / macOS / Linux 可构建可运行。

---

## 1. 背景与问题

QSanguosha（太阳神三国杀）是基于 **C++ + Qt Graphics View** 的桌面客户端，服务端逻辑与 **Lua（SWIG 绑定）** 深度耦合。上游工程长期停留在 **Qt 5.3.x** 时代，并依赖已废弃的 **Qt Declarative（Qt Quick 1）** 模块。

在当前环境（例如 **Ubuntu 22.04**）直接编译时，典型失败包括：

- `Project ERROR: Unknown module(s) in QT: declarative`
- 缺少 **SWIG** 生成的 `swig/sanguosha_wrap.cxx`
- Linux 需正确部署 **FMOD** 动态库（`lib/linux/x64/libfmodex*.so`）
- Release 模式下 **Breakpad** 与新版 **glibc / GCC** 可能存在兼容问题

仓库根目录大量 **`.dll`**（多为 `fmodex*.dll`、`vld_essentials`）是 **Windows 运行时预置**，Linux/macOS 构建**不会使用**这些文件；对应平台库在 `lib/linux/`、`lib/mac/` 等目录。

---

## 2. 技术选型结论（与 ImGui / Tauri 对比）

在「效果基本一致 + 功能不减 + 跨平台」前提下：

| 方案 | 结论 |
|------|------|
| **升级 Qt（5.15 LTS 或 Qt 6）** | **首选**。复用现有 UI、资源、Lua 扩展，改动集中在过时模块与 API。 |
| **ImGui 重做 UI** | **不推荐**。需重写 `src/ui`（约 2 万行）及大量 dialog，难以在合理周期内复刻皮肤与动效。 |
| **Tauri + Rust** | **不推荐作为第一刀**。等于新客户端 + 规则/协议/ Lua 绑定大规模迁移，与「在 fork 上演进」目标不符。 |

**推荐路线**：保留 C++ 规则引擎与 Lua 扩展，**升级 Qt 工具链与废弃模块**，Graphics View 层可阶段性保留。

---

## 3. 架构依赖（升级时不能误伤的部分）

| 层级 | 路径/技术 | 说明 |
|------|-----------|------|
| 工程入口 | `QSanguosha.pro`、`linux.mk` | qmake 工程；Linux 可用 `make -f linux.mk` |
| UI | `src/ui/`（如 `roomscene.cpp` 约 4k+ 行） | Qt Graphics View、自定义控件、QSS |
| 动效 | `ui-script/animation.qml` + `QDeclarative*` | **必须**迁移到 Qt Quick 2，不能长期注释掉（功能不减） |
| 脚本 | `lua/`、`swig/` | 开局前需 `swig -c++ -lua sanguosha.i` |
| 音频 | FMOD Ex | Linux 链接 `libfmodex` / `libfmodexL`（debug） |
| 字体 | 系统 **freetype** | `lib/linux/Notice of freetype.txt` 说明无需自带 Linux freetype |

代码体量参考：C++ 合计约 **11 万行**；Lua AI 约 **2.1 万行**（与构建/UI 升级并行但非本计划主路径）。

---

## 4. 核心改造项

### 4.1 Qt Declarative（Quick 1）→ Qt Quick 2

**现状**（`QSanguosha.pro`）：

```qmake
!winrt:QT += declarative
```

**代码引用**（`src/ui/roomscene.cpp` / `roomscene.h`）：

- `QDeclarativeEngine` / `QDeclarativeContext` / `QDeclarativeComponent`
- 加载 `ui-script/animation.qml`，用于 `skill=` 类技能展示动效

**目标**：

- 工程改为 `QT += quick`（Qt6 下按需 `quickwidgets` 等）
- 使用 `QQmlEngine`、`QQmlContext`、`QQmlComponent` 或 `QQuickWidget` 嵌入场景
- 将 `animation.qml` 改写为 **Quick 2** 语法（import、属性、动画 API 均不同）

**注意**：`.travis.yml` 中已有注释 *Enable Travis CI again after we remove declarative module*，说明上游已知此阻塞项。

### 4.2 Qt 版本目标

| 选项 | 优点 | 缺点 |
|------|------|------|
| **Qt 5.15 LTS** | Quick2 成熟；Graphics View 仍完整；迁移量相对 Qt6 小 | 5.15 已停止商业支持，需自建 CI 镜像 |
| **Qt 6.x** | 长期维护、工具链新 | `QT += declarative`、部分 API、网络/字符串等需系统性替换；Graphics View 为维护模式 |

**建议**：fork 第一阶段锁定 **Qt 5.15**，三平台跑通后再评估 Qt 6。

### 4.3 构建与依赖

**Linux（开发机）最低依赖**：

- `build-essential`、`swig`（≥ 3.0.1，建议 3.0.4+）
- `qtbase5-dev`、`qtdeclarative5-dev`（Quick 1 头文件若仅用于过渡可省略，但 Quick2 需 `qtdeclarative5-dev` / QML 模块）
- `libfreetype6-dev`
- 将 `lib/linux/x64/libfmodex*.so` 安装到 `/usr/local/lib` 或 rpath 指向仓库 `lib/linux/x64`，并 `ldconfig`

**构建步骤（与 README 一致，略作现代化）**：

```bash
cd swig && swig -c++ -lua sanguosha.i && cd ..
qmake CONFIG+=release QSanguosha.pro
make -j"$(nproc)"
```

或使用 `linux.mk` 首次：`make -f linux.mk`

**Debug 建议**：若 Release + Breakpad 编译失败，可先用 `CONFIG+=debug` 验证 UI/Quick 迁移，再单独处理 Breakpad（或 Release 下 `USE_BREAKPAD` 条件编译）。

### 4.4 Windows / macOS

- **Windows**：仍可用 `builds/vs2013` 思路，但需升级到 **VS2019+** 与对应 **Qt 5.15 MSVC 套件**；根目录 DLL 仅作运行时拷贝参考，最终应以所装 Qt 版本为准。
- **macOS**：`README.md` 中 `install_name_tool` + `macdeployqt` 流程仍适用，FMOD/freetype 路径改为 fork 内 `lib/mac/`。

### 4.5 不建议在本阶段做的改动

- 用 ImGui 替换 `src/ui`
- 整体迁 Tauri / Web 前端
- 删除 Lua/SWIG 或大规模改协议（与 AI Agent 计划解耦）

---

## 5. 分阶段实施计划

### 阶段 0：基线可复现

- [ ] 记录当前 fork 的 commit、目标 Qt 版本
- [ ] Docker 或固定 VM（Ubuntu 20.04/22.04 + Qt 5.15）作为标准构建环境
- [ ] 文档化：资源目录（`image/`、`hero-skin/`、`style-sheet/`）与可执行文件相对路径

### 阶段 1：解除「编不过」

- [ ] 安装 SWIG，生成并提交或 CI 生成 `swig/sanguosha_wrap.cxx`（团队策略二选一）
- [ ] 部署 FMOD Linux `.so`
- [ ] `qmake` 通过（先解决 `declarative` 模块问题：进入阶段 2 或临时 stub）

### 阶段 2：Quick 1 → Quick 2（功能不减的关键路径）

- [ ] 修改 `QSanguosha.pro`：`QT += quick`，移除 `declarative`
- [ ] 重构 `roomscene` 中动效初始化与 `skill=` 分支（原 `#ifndef Q_OS_WINRT` 块）
- [ ] 迁移 `ui-script/animation.qml`
- [ ] 回归：进房、发动带技能动效武将，确认与原版观感一致

### 阶段 3：三平台 CI 与打包

- [ ] Linux：AppImage /  tarball + 自带 `libfmodex.so` rpath
- [ ] Windows：windeployqt 拷贝 Qt DLL
- [ ] macOS：macdeployqt + Frameworks

### 阶段 4（可选）：Qt 6 与 Graphics View 长期策略

- [ ] API 扫描（`QRegExp`、`QStringRef`、Qt5 废弃 API）
- [ ] 评估是否将部分 UI 逐步迁到 Quick（非必须）

---

## 6. 验收标准

1. **功能**：标准包、DIY Lua 包、选将、房间、托管、加电脑、观星/拼点/无懈等流程与原版一致（人工回归清单 + 若干录像对比）。
2. **视觉**：皮肤、QSS、牌面布局、技能动效无明显退化。
3. **跨平台**：同一 tag 在 Linux / Windows / macOS 均可构建；至少 Linux + 一个桌面平台完成发布包 smoke test。
4. **工程**：新贡献者按 `plan/` + README 可在 30 分钟内完成依赖安装并编过（不含下载 Qt 安装包时间）。

---

## 7. 风险与缓解

| 风险 | 缓解 |
|------|------|
| Quick2 动效与原版不一致 | 保留旧 qml 作参考，逐帧对比；必要时 C++ 侧补动画 |
| SWIG 4.x 与 3.x 差异 | CI 固定 SWIG 版本 |
| FMOD 授权与分发 | 遵守 FMOD 许可，发布说明中标注 |
| Breakpad 在新 GCC 失败 | Release 先关 Breakpad 或升级 breakpad 源码 |

---

## 8. 参考文件索引

- `README.md` / `README.builds` — 官方构建说明（版本较旧）
- `QSanguosha.pro` — Qt 模块与平台 `LIBS`
- `linux.mk` — Linux 本地 makefile 流程
- `src/ui/roomscene.cpp` — Declarative 动效
- `lib/linux/x64/` — FMOD Linux 64 位库

---

## 9. 与 AI Agent 计划的关系

UI/构建升级 **不阻塞** Agent 接入（Agent 主要动服务端 `src/server/ai.*` 与 `lua/ai/`），但建议 **先完成阶段 1～2**，保证 fork 可稳定跑局，再在 robot 位或自定义 AI 上接 LLM。详见 `20260930_add_ai_agent.md`。
