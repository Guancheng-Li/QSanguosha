# QSanguosha 外部参考与社区资料

**文档日期**：2026-09-30  
**适用仓库**：本地 fork（`/home/guancheng/test/QSanguosha`）  
**用途**：集中记录与 fork 开发相关的 **Wiki、Lua 技能手册** 等外链；本仓库内可对照的文档见文末。

---

## 1. 总览

| 资源 | 链接 | 语言 | 与本 fork 的关系 |
|------|------|------|------------------|
| gaodayihao / QSanguosha Wiki | https://github.com/gaodayihao/QSanguosha/wiki/ | 中文为主 | 编译、Lua 扩展、历史教程 |
| Mogara / LuaSkillsForQSGS | https://github.com/Mogara/LuaSkillsForQSGS | 中文 | 武将技能 Lua 实现速查（偏旧版语法） |
| API 参考（Doxygen） | http://gaodayihao.github.com/QSanguosha/api | — | README 中 C++ API 索引 |
| 本仓库扩展文档 | `extension-doc/` | 中文 | 与 Wiki「Lua 扩展」互补，更贴近当前 Mogara 线代码 |

README 中官方文档入口即 gaodayihao Wiki（中文）：  
https://github.com/gaodayihao/QSanguosha/wiki

---

## 2. gaodayihao / QSanguosha Wiki

### 2.1 是什么

- 仓库：**gaodayihao/QSanguosha**（太阳神早期主线之一；本 fork 上游为 **Mogara/QSanguosha**，Wiki 仍具参考价值）。
- Wiki 克隆地址（只读）：`https://github.com/gaodayihao/QSanguosha.wiki.git`
- 首页：https://github.com/gaodayihao/QSanguosha/wiki/Home

### 2.2 Wiki 页面索引（截至拉取时）

| Wiki 文件 | 主题 | 说明 |
|-----------|------|------|
| [教程](https://github.com/gaodayihao/QSanguosha/wiki/教程) | 总目录 | 链到 Linux 编译、Lua 扩展 |
| [教程/Linux编译](https://github.com/gaodayihao/QSanguosha/wiki/教程-Linux编译) | Linux 编译 | Kubuntu 12.04、FMOD、`make -f linux.mk`；**环境较旧**，与 `20260930_ui_upgrade.md` 中 Qt 5.15+ 目标需对照更新 |
| [教程/Lua扩展](https://github.com/gaodayihao/QSanguosha/wiki/教程-Lua扩展) | DIY 包 / Lua | `sgs.CreateViewAsSkill` 与 class 两种写法；`extensions/` 目录结构；可参考 [GutenYe/qsgs-extensions](https://github.com/GutenYe/qsgs-extensions) |
| `太阳神三国杀.asciidoc` 等 | 杂项 | Wiki 仓库内其它asciidoc/creole 页面，按需查阅 |

### 2.3 开发时怎么用

- **做 UI/编译升级**：Wiki 的 Linux 编译作历史参考；**以本 fork 的 `README.md`、`linux.mk`、`plan/20260930_ui_upgrade.md` 为准**。
- **做 DIY 扩展包**：Wiki Lua 扩展 + 本仓库 `extension-doc/1-Start.lua` 起的一系列说明。
- **做 Agent**：Wiki 几乎不涉及 AI；见 `plan/20260930_add_ai_agent.md` 与 `extension-doc/12-SmartAI.lua`。

---

## 3. Mogara / LuaSkillsForQSGS

### 3.1 是什么

- 链接：https://github.com/Mogara/LuaSkillsForQSGS  
- 描述：**新版太阳神三国杀武将技能代码速查手册（Lua 版）**  
- 组织：**Mogara**（与当前游戏主线组织一致），社区维护的技能实现摘录与注释。

### 3.2 内容结构

- 按技能名 **拼音首字母 A～Z** 分章（`ChapterA.md` … `ChapterZ.md`），章内为单个技能的 Lua 示例与说明。
- README 说明：多数技能写于 **2012 年前后**，**不能直接用于较新的 1210 / 1217 等版本**，需人工迁移到当前引擎 API。
- 附带 **`sgs_ex.lua`**：README 称可拷贝到神杀目录 `lua/` 下覆盖同名文件——**覆盖前务必备份**，并确认与当前 fork 的 `lua/sanguosha.lua` 兼容。

### 3.3 开发时怎么用

| 场景 | 建议 |
|------|------|
| 查某武将技能怎么写 Trigger / ViewAs | 在对应 Chapter 搜索技能名，**对照**本仓库 `lua/`、`extensions/` 现网实现 |
| 写新 DIY 技能 | 以 `extension-doc/` + Wiki Lua 扩展为主，LuaSkills 作「思路参考」 |
| 给 LLM 做 RAG / 提示词 | 可索引 Chapter 文本，但须标注 **API 可能过期**，并以 `swig/*.i`、`lua/sanguosha.lua` 为准校验 |
| SmartAI | 手册偏 **技能定义**，AI 逻辑见 `lua/ai/` 与 `extension-doc/12-SmartAI.lua` |

### 3.4 注意事项

- **版本漂移**：手册与 Mogara 主线不同步时，以 **本 fork 源码** 为唯一真实来源。
- **不要**未经测试整包替换 `sgs_ex.lua` 或大量 extensions。
- 勘误与讨论：仓库 [Issues](https://github.com/Mogara/LuaSkillsForQSGS/issues)。

---

## 4. 与本 fork `plan/` 的对应关系

```
plan/20260930_ui_upgrade.md      ← 编译/Qt；Wiki Linux 教程仅作历史参考
plan/20260930_add_ai_agent.md    ← Agent；Wiki / LuaSkills 不覆盖
plan/20260930_reference.md       ← 本文：外链索引
extension-doc/                     ← 仓库内 Lua/AI 官方扩展文档（优先）
```

---

## 5. 本仓库内优先阅读的文档

| 路径 | 内容 |
|------|------|
| `README.md` | 构建、主页、Wiki/API 链接 |
| `extension-doc/` | Lua 技能、SmartAI、示例（`12-SmartAI.lua`、`17-Example.lua` 等） |
| `lua/ai/smart-ai.lua` | 运行时 AI 实现 |
| `swig/sanguosha.i` | Lua 与 C++ 绑定面 |

---

## 6. 链接速查

- Wiki 首页：https://github.com/gaodayihao/QSanguosha/wiki  
- Wiki Git：https://github.com/gaodayihao/QSanguosha.wiki.git  
- Lua 技能手册：https://github.com/Mogara/LuaSkillsForQSGS  
- C++ API（在线）：http://gaodayihao.github.com/QSanguosha/api  
- Mogara 游戏源码（上游）：https://github.com/Mogara/QSanguosha  
