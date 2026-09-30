# QSanguosha 接入 AI Agent（大模型）计划

**文档日期**：2026-09-30  
**适用仓库**：本地 fork（`/home/guancheng/test/QSanguosha`）  
**目标**：在保留完整游戏规则与扩展（Lua 包）的前提下，接入**自有大模型**参与对局决策（非替换现网真人匹配逻辑，除非产品另行定义）。

---

## 1. 结论摘要

| 问题 | 答案 |
|------|------|
| 是否有现成的 HTTP / OpenAI / Agent API？ | **没有**。 |
| 现有 AI 是什么？ | 服务端 **C++ `AI` 接口** + **`LuaAI`** + **`smart-ai.lua`（SmartAI）**。 |
| 真人怎么操作？ | **`Room::doRequest`** 通过 **`QSanProtocol`** TCP 包与 Qt 客户端交互，**不是 REST**。 |
| 推荐接入方式 | **Robot 位 + 扩展 SmartAI 或新增 C++ `LLMAI`**（见 §4）。 |

---

## 2. 现有架构

### 2.1 决策分流（必读）

`ServerPlayer::getAI()` 决定本步由谁决策：

```cpp
// src/server/serverplayer.cpp（逻辑摘要）
if (state == "online")       return NULL;      // → doRequest → 客户端
else if (state == "robot" || EnableCheat) return ai;  // → SmartAI（Lua）
else                         return trust_ai;  // → TrustAI（简单托管）
```

| 玩家状态 | 决策来源 | 与 LLM 的关系 |
|----------|----------|----------------|
| `online` | 客户端协议回复 | 需实现「无头客户端」或改服务端（工作量大） |
| **`robot`** | **`cloneAI` → SmartAI** | **最适合挂 LLM** |
| `trust`（托管） | **TrustAI**，非 SmartAI | 若要用 LLM 代打真人位，需**改此分支**或禁止托管走 TrustAI |

房间「加电脑」：`Room::addRobotCommand` 创建 `state == "robot"` 的 `ServerPlayer`（见 `src/server/room.cpp`）。

### 2.2 AI 加载与工厂

房间构造时加载 Lua AI 脚本（**可覆盖**）：

```cpp
DoLuaScript(L, QFile::exists("lua/ai/private-smart-ai.lua") ?
    "lua/ai/private-smart-ai.lua" : "lua/ai/smart-ai.lua");
```

开局为每位玩家：

```cpp
AI *ai = cloneAI(player);
player->setAI(ai);
```

Lua 暴露给 C++ 的工厂（`lua/ai/smart-ai.lua`）：

```lua
function CloneAI(player)
    return SmartAI(player).lua_ai
end
```

C++ 实现（`swig/ai.i`）：调用 `CloneAI`，得到 `LuaAI*`，其方法再回调 Lua 中 `SmartAI` 的 `activate`、`askForCard`、`askForSkillInvoke` 等。

### 2.3 C++ `AI` 虚函数清单（接入点）

定义于 `src/server/ai.h`，由 `Room::askFor*` / 出牌流程调用，包括但不限于：

- `activate(CardUseStruct &)` — 出牌阶段
- `askForSkillInvoke` / `askForChoice`
- `askForCard` / `askForUseCard` / `askForDiscard`
- `askForPlayerChosen` / `askForCardChosen` / `askForAG`
- `askForNullification` / `askForPindian` / `askForGuanxing` / `askForYiji`
- ……

**每一次询问** = Agent 的一次「观测 → 决策」机会（若全部走 LLM，成本高、延迟大）。

### 2.4 真人协议路径（备选：外部 Bot）

- 命令枚举：`src/core/protocol.h`（如 `S_COMMAND_PLAY_CARD`、`S_COMMAND_INVOKE_SKILL`、`S_COMMAND_CHOOSE_CARD` 等）
- 服务端请求：`Room::doRequest` → 客户端回包 → `getClientReply()`
- 客户端实现参考：`src/client/client.cpp`

**无官方 Bot SDK**；走此路径等于实现第二个客户端（见 §4.4 说明：这是实现复杂，**不是**对 exe 做逆向）。

### 2.5 文档与扩展

- SmartAI 编写说明：`extension-doc/12-SmartAI.lua`
- 私有 AI 逻辑：新增 `lua/ai/private-smart-ai.lua`（不修改上游 `smart-ai.lua` 亦可）

---

## 3. 明确「没有什么」

- 无配置文件中的 `OPENAI_API_KEY`、无 `curl` / HTTP 的 Lua 标准库
- 无「导出完整局面 JSON」的单一 API（需自建 **状态序列化**）
- 无「自然语言描述一步」的接口；引擎只认 **结构化返回值**（卡牌字符串、id 列表、玩家 objectName、bool 等）
- 托管（trust）**不会**自动使用 SmartAI / LLM

---

## 4. 推荐接入路线

### 路线 A：Lua 层扩展 SmartAI（改动面小，适合 PoC）

**做法**：

1. 新增 `lua/ai/private-smart-ai.lua`，`require` 或复制 `smart-ai.lua` 后，在关键方法中调用 LLM 逻辑。
2. 在 C++ 增加 **LLM 桥接函数**（推荐），例如：
   - `Room::queryLlm(const QString &prompt, int timeoutMs) -> QString`  
   或独立进程 + Unix socket / HTTP localhost。
3. 通过 SWIG 暴露给 Lua：`room:queryLlm(...)`。

**原因**：Lua 内不宜直接做阻塞 HTTP；对局在 **`RoomThread`** 上执行，长时间阻塞会卡住整局。

**Robot 位**：房间用「加电脑」即可，无需改 `getAI()`。

**优点**：迭代快，与现有 `sgs.ai_*` 钩子共存（可「常規 SmartAI + 关键节点 LLM」）。  
**缺点**：逻辑分散在巨大 `smart-ai.lua` 生态中，类型安全弱。

---

### 路线 B：C++ `LLMAI : public AI`（推荐用于长期维护）

**做法**：

1. 新建 `src/server/llmai.h` / `llmai.cpp`，实现 `ai.h` 中全部虚函数。
2. 内部：`buildPrompt(observation)` → HTTP/gRPC 调你的模型服务 → `parseAction()` → 校验合法性。
3. 修改 `Room::cloneAI` 或 Lua `CloneAI`：对指定 robot（如 screenName 前缀 `LLM_`）返回 `LLMAI` 实例。
4. 异步：网络请求在 worker 线程完成，房间线程带 **超时 wait**；失败时 fallback `TrustAI` 或 SmartAI 子集。

**优点**：超时、日志、重试、密钥管理集中；便于单元测试。  
**缺点**：需逐个理解 `askFor*` 的入参与返回值格式（对照 `room.cpp` 调用点）。

---

### 路线 C：外部进程伪装客户端（不改服务端 AI）

**做法**：实现 TCP 客户端，登录房间，人类座位，按协议响应 `S_TYPE_REQUEST`。

**优点**：服务端零改动。  
**缺点**：协议与 UI 交互完整复刻成本高；**信息仅协议可见**（非 server 全知）；与「robot 全 AI」体验不同。

**适用**：研究协议、单席位实验；**不推荐**作为 fork 主路线。

### 4.4 常见问题：路线 C 是「逆向」吗？仓库里有没有现成协议实现？

#### 结论（可直接理解）

| 说法 | 是否成立 |
|------|----------|
| 路线 C **实现复杂**、工作量大 | **是** |
| 需要对 `QSanguosha.exe` **逆向**猜逻辑/包格式 | **否** |
| 需要 **读源码** 对齐客户端行为 | **是** |

**「无 SDK」** 指：没有官方的 Bot 库、headless 客户端开关或对外协议文档与版本承诺。  
**不等于** 协议是黑盒；**协议与交互逻辑均在开源 C++ 中**，按源码复刻即可。

#### 协议在哪里（白盒，非抓包猜格式）

1. **传输层**（`src/util/clientsocket.cpp`）  
   - TCP；每条应用消息以 `InlineTextPacket` 字节开头，正文为 **以 `\n` 结尾的一行文本**。

2. **应用层包体**（`src/core/protocol.cpp` / `protocol.h`）  
   - 一行 JSON 数组，字段顺序：`globalSerial`、`localSerial`、`packetDescription`、`command`、可选 `messageBody`。  
   - `command` 为 `S_COMMAND_*` 枚举（同文件）。

3. **服务端**  
   - 发请求：`Room::doRequest`  
   - 收回复：`Room::processClientPacket`（`src/server/room.cpp`）

4. **客户端（Bot 应对齐的行为）**  
   - 收请求、组回复：`Client::replyToServer` 及 `src/client/client.cpp` 中各 `S_COMMAND_*` 分支（选将、出牌、观星、无懈等）。

因此路线 C 的成本来自：**命令种类多、客户端状态机与 UI 历史耦合、需维护序列号/超时/保活**，属于 **工程复刻**，不是 **二进制逆向**。

#### 与路线 A/B 的对比

| 维度 | 路线 A/B（服务端 `AI` / robot） | 路线 C（TCP 假客户端） |
|------|--------------------------------|------------------------|
| 是否写 TCP 协议客户端 | 否 | 是 |
| 是否读 `protocol.*` / `client.cpp` | 否（走 `AI` 虚函数） | 是 |
| 是否逆向 exe | 否 | **否** |
| 决策位置信息 | 服务端 SmartAI 视角（更全） | 仅协议广播的客户端视角 |

#### 若仍选路线 C，较省力的做法

在 fork 内从现有 **`src/client/client.cpp` 抽出 HeadlessClient**（复用 `Packet` 解析与 `replyToServer` 逻辑），比用 Python 从零重写全部命令处理更稳；仍 **无需** 逆向，主要是 **重构 + 去 UI 依赖** 的工作量。

#### 与「仓库里的二进制」的关系（易混淆点）

- Git 仓库 **通常不包含** 本项目编译中间产物（`.o`、`moc_*`、`swig/sanguosha_wrap.cxx` 等，见 `.gitignore`）。  
- 仓库 **包含** 第三方**预编译**运行时/链接库（如根目录 `fmodex*.dll`、`lib/linux/*/libfmodex.so`），与 Bot 协议实现 **无关**。  
- 路线 C **不依赖** 这些 DLL 来「猜协议」；依赖的是 **`src/` 下的源码**。

---

## 5. 必须自建的「Agent 三层」

与路线无关，任何 LLM 接入都要实现：

### 5.1 观测（State → Prompt / JSON）

从 SWIG 已暴露对象提取信息，例如：

- `ServerPlayer`：武将、体力、手牌数、装备、判定区、身份/势力（可见范围内）
- `Room`：当前阶段、当前回合角色、场上事件栈（可配合 `room:writeToConsole` / 日志）
- 当前询问类型：`invoke_skill`、`choose_card`、`play_card` 等

**建议**：为每种 `askFor*` 定义 **固定 JSON schema**，便于模型结构化输出。

### 5.2 动作空间（Constraint → 候选列表）

**不要**让模型自由生成任意字符串。应：

1. 在 C++ 或 Lua 侧列出 **当前合法动作**（可出的牌 id、可选目标 objectName、choice 字符串）。
2. Prompt 中只让模型 **选择 candidate_id** 或返回索引。

非法动作 → 重试一次 → 仍失败则 **fallback**（TrustAI 或随机合法项），避免死锁。

### 5.3 执行（Action → 引擎 API）

返回值必须满足现有接口约定，例如：

- 出牌：`Card::Parse` 兼容的字符串（与客户端一致）
- 弃牌：`QList<int>` 卡牌 id
- 选将/选人：`ServerPlayer::objectName()`
- 发动技能：`bool` / `askForChoice` 的 option 字符串

参考 SmartAI 实现：`lua/ai/smart-ai.lua` 及各 `*-ai.lua`。

---

## 6. 性能、延迟与产品策略

| 项 | 说明 |
|----|------|
| 调用次数 | 一局可能 **上百次** `askFor*`；全量 LLM 成本高 |
| AI 延迟 | `RoomThread::delay()` 使用 `config.AIDelay`；LLM 需单独配置超时（可 cheat `.SetAIDelay` 仅影响展示延迟，不替代 HTTP 超时） |
| 真人超时 | `RoomInfoStruct::getCommandTimeout` 与 `OperationTimeout` 相关；**robot 不走 doRequest**，但 LLM 过慢仍阻塞 room 线程 |
| 分层策略 | **默认 SmartAI**，仅在出牌阶段 / 关键技能 / 身份局决策时调用 LLM |
| 托管 | 若需「真人离席 + LLM」，修改 `getAI()`：在 `trust` 时返回 `ai`（SmartAI/LLMAI）而非 `trust_ai` |

---

## 7. 配置与安全

建议在 fork 中新增（示例）：

```ini
# 勿提交密钥到 git
[llm]
base_url=https://your-api/v1/chat/completions
api_key_env=QSANGUOSHA_LLM_API_KEY
model=your-model
timeout_ms=15000
max_retries=1
enabled_robots=true
```

- API Key 仅环境变量或本地配置文件（`.gitignore`）
- 服务端日志：记录 prompt 哈希 / 决策 id，避免泄露完整手牌到不可信日志系统
- 若对接公网 API，注意 **手牌与身份** 属于敏感对局数据

---

## 8. 分阶段实施计划

### 阶段 0：前提

- [ ] 按 `20260930_ui_upgrade.md` 保证 fork **能稳定开房、加 robot、跑完一局**
- [ ] 熟悉一局录像或控制台日志，确认 robot 走的是 SmartAI

### 阶段 1：LLM 桥接（无对局）

- [ ] C++ HTTP 客户端（Qt `QNetworkAccessManager`）或 sidecar
- [ ] 配置加载、超时、单元测试（mock API）

### 阶段 2：单点 PoC

- [ ] 仅实现 `activate` 或 `askForSkillInvoke` 一条链路
- [ ] Robot 1 名，其余 SmartAI / 真人
- [ ] 合法动作列表 + 模型选 id

### 阶段 3：覆盖高频 askFor*

- [ ] 弃牌、出杀/闪、桃、无懈、选目标、拼点、观星（按模式优先级排序）
- [ ] 统一 observation JSON 与 fallback

### 阶段 4：工程化

- [ ] `LLMAI` 与 Lua 钩子二选一或混合
- [ ] 指标：平均延迟、非法动作率、胜率（对比纯 SmartAI）
- [ ] 可选：`trust` 分支策略、仅房主可开 LLM robot

---

## 9. 验收标准

1. **功能**：LLM robot 能完成标准局（8 人身份局或房间配置对应模式），无因超时永久卡死。
2. **规则**：无非法动作导致崩溃；非法输出可 fallback 并继续对局。
3. **扩展**：不破坏现有 Lua 包与 `CloneAI`；未启用 LLM 时行为与原版 SmartAI 一致。
4. **密钥**：仓库内无硬编码 API Key；文档说明如何配置。

---

## 10. 关键文件索引

| 文件 | 用途 |
|------|------|
| `src/server/ai.h` | AI 虚函数接口 |
| `src/server/ai.cpp` | TrustAI / LuaAI 部分实现 |
| `swig/ai.i` | `cloneAI`、`LuaAI` 回调 |
| `src/server/serverplayer.cpp` | `getAI()` 三分支 |
| `src/server/room.cpp` | `askFor*`、`doRequest`、`addRobotCommand`、`startGame` |
| `lua/ai/smart-ai.lua` | `CloneAI`、`SmartAI:initialize` |
| `extension-doc/12-SmartAI.lua` | AI 扩展文档 |
| `src/core/protocol.h` | 协议命令（路线 C） |

---

## 11. 路线选择建议（快速对照）

| 你的目标 | 建议 |
|----------|------|
| 最快验证「模型能替电脑打牌」 | **路线 A** + robot + C++ HTTP 桥 |
| 长期维护、多模型、日志与超时 | **路线 B** |
| 坚决不改服务端 | **路线 C**（实现复杂，但协议可读源码、无需逆向 exe） |
| 仅增强原 AI、不用 LLM | 改 `sgs.ai_*` / 各 `*-ai.lua`，无需本计划 |

---

## 12. 与 UI 升级计划的关系

- Agent 开发可在 **服务端 + Lua** 独立推进，但 **强烈建议** 先完成 UI/构建升级中的「可编可跑」，便于本地反复开 robot 局测试。
- Quick 动效迁移与 LLM **无直接依赖**。
