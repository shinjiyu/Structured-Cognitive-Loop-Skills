# 安装指南

---

## 第一步：安装 OpenClaw（若尚未安装）

### 安装 Node.js ≥ 22

```bash
# 使用 nvm（推荐）
nvm install 22 && nvm use 22

# 验证
node -v  # 应显示 v22.x.x
```

### 安装并初始化 OpenClaw

```bash
npx openclaw onboard
```

`onboard` 会引导你完成：
- 连接 LLM 提供商（Claude / GPT-4o / Gemini / 本地 Ollama）
- 连接通讯频道（Telegram / Slack / Discord 等，至少选一个）
- 生成初始 workspace 目录（默认 `~/.openclaw/workspace`）

完成后验证：

```bash
openclaw gateway start
# 看到 "Gateway running on port 18789" 表示正常
```

> 更多细节见 [OpenClaw 官方文档](https://docs.openclaw.ai) 和 [GitHub 仓库](https://github.com/openclaw/openclaw)

---

## 第二步：安装本技能系统

### 克隆本仓库

```bash
git clone https://github.com/shinjiyu/Structured-Cognitive-Loop-Skills.git
cd Structured-Cognitive-Loop-Skills
```

### 安装 Skills

四个核心 Skill 是**安全覆盖**的——它们是新目录，不会与已有文件冲突：

```bash
WORKSPACE=~/.openclaw/workspace
mkdir -p $WORKSPACE/skills

cp -r skills/scl-control            $WORKSPACE/skills/
cp -r skills/scl-memory             $WORKSPACE/skills/
cp -r skills/recap-decompose        $WORKSPACE/skills/
cp -r skills/capability-gap-handler $WORKSPACE/skills/

# 创建 memory 目录（用于 Daily Log，若不存在）
mkdir -p $WORKSPACE/memory
```

---

## 第三步：配置文件（区分新用户 / 已有用户）

以下文件需要根据你的情况选择**全量替换**或**合并**：

| 文件 | 新用户 | 已有 OpenClaw 用户 |
|------|--------|-------------------|
| `SOUL.md` | 直接使用模板 | 合并（把 SCL 工作原则追加到已有 SOUL.md） |
| `HEARTBEAT.md` | 直接使用模板 | 合并（把 R-CCAM 检查清单追加到已有 HEARTBEAT.md） |
| `TASKS.md` | 直接使用模板 | 直接创建（新文件，不冲突） |
| `MEMORY.md` | 直接使用模板 | 合并（追加新章节，不覆盖已有内容） |

---

### 新用户：直接使用模板

```bash
WORKSPACE=~/.openclaw/workspace

cp templates/SOUL.md      $WORKSPACE/SOUL.md
cp templates/HEARTBEAT.md $WORKSPACE/HEARTBEAT.md
cp templates/TASKS.md     $WORKSPACE/TASKS.md
cp templates/MEMORY.md    $WORKSPACE/MEMORY.md
```

然后编辑各文件填写你的信息（见下方「填写说明」）。

---

### 已有用户：合并而非替换

#### SOUL.md — 追加 SCL 工作原则

**不要覆盖已有 SOUL.md**。在你现有 SOUL.md 末尾追加以下内容：

```markdown
## SCL Working Principle（工作原则）

你遵循 SCL R-CCAM 框架运行：

- **R（Retrieval）**：每次心跳开始时，读取 TASKS.md、MEMORY.md 和近期 Daily Log
- **C（Cognition）**：基于任务树（由 recap-decompose Skill 维护），决定本轮应执行的子任务
- **C（Control）**：执行前通过 scl-control Skill 验证，约束检查、工具可用性检查
- **A（Action）**：执行单步最优行动，优先使用已有 Skill
- **M（Memory）**：通过 scl-memory Skill 将结果写入 Daily Log 和 TASKS.md

当遇到能力缺口时，通过 capability-gap-handler Skill 自动派生新能力，不请求用户干预，
除非遇到真正的外部依赖（API Key、账户权限、合规审批等）。

## SCL Rules（补充规则）
- 每次心跳只做一件事：执行任务树的当前节点，不跳跃
- Action 前必须经过 scl-control 验证
- 每轮结束必须更新 TASKS.md 和 Daily Log（通过 scl-memory Skill）
- 工具不存在时，用原子能力派生（capability-gap-handler），不放弃任务
```

#### HEARTBEAT.md — 追加 R-CCAM 检查清单

**不要覆盖已有 HEARTBEAT.md**。在文件末尾追加以下内容（在已有检查项之后）：

```markdown
---

## SCL R-CCAM 执行循环

### R — Retrieval
- 读取 TASKS.md：确认顶层目标，找到当前应执行的子任务节点
- 快速扫描 MEMORY.md：确认约束条件和最新领域知识
- 查看今天 Daily Log（若已有）：避免重复执行

### C — Cognition（决策）
- 若 TASKS.md 无任务树：调用 recap-decompose 生成子任务树
- 若有任务树：选择当前 `[~]` 或下一个 `[ ]` 节点作为本轮目标
- 若当前层级全部完成：调用 recap-decompose 重分解

### C — Control
- 调用 scl-control Skill 验证本轮计划行动
- 若工具不存在：进入 capability-gap-handler 流程

### A — Action
- 执行通过 Control 验证的单步行动

### M — Memory
- 调用 scl-memory Skill 写入 Daily Log 和更新 TASKS.md
```

#### TASKS.md — 直接创建（新文件）

```bash
# TASKS.md 是新文件，直接创建不会影响任何已有内容
cp templates/TASKS.md $WORKSPACE/TASKS.md
```

#### MEMORY.md — 追加新章节

**不要覆盖已有 MEMORY.md**。在文件末尾追加以下章节：

```markdown
---

## 约束条件（SCL，不可违反）
- [填写你的约束条件]

## 领域知识（SCL）
- [由 Agent 在执行中逐步填写]
```

---

## 第四步：填写配置内容

**TASKS.md**（必填）：

```markdown
## 顶层目标
[写入你给 Agent 的抽象目标，例如：帮我运营小红书，三个月涨粉 1000 人]

## 约束条件
- 不使用付费推广
- 不涉及违规操作
- [其他约束...]
```

**SOUL.md / MEMORY.md**：填写 Agent 名称、角色、已有的领域知识（可选，Agent 会在运行中自动补充）。

---

## 第五步：配置心跳调度

在 OpenClaw 的 `config.json5` 中启用 Heartbeat：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",           // 心跳间隔，建议 15m~60m
        target: "last",         // 通知发送到最近的对话
        activeHours: {
          start: "08:00",
          end: "22:00"          // 仅在此时间段内心跳（防止凌晨打扰）
        }
      }
    }
  }
}
```

`config.json5` 位置：`~/.openclaw/config.json5`（通过 `onboard` 生成）

---

## 第六步：启动并验证

```bash
# 启动 Gateway
openclaw gateway start

# 手动触发一次心跳验证安装（可选）
openclaw agent --message "请执行一次完整的 SCL R-CCAM 心跳流程，并报告每个阶段的执行情况"
```

期望输出应包含：
- `[SCL-Control]` 的验证报告
- `[SCL-Memory]` 的写入确认
- `TASKS.md` 任务树（首次运行由 recap-decompose 生成）

---

## 第七步：下达抽象目标

通过你连接的频道（Telegram / Slack 等）发送目标，**之后无需再给任何指令**：

```
帮我运营小红书账号，目标是三个月内涨粉 1000 人。
账号赛道是生活方式，目前粉丝数约 200 人。
约束：不使用付费推广，不涉及违规操作。
```

Agent 会在下一次心跳时开始自主执行。

---

## 安装后的目录结构

```
~/.openclaw/workspace/
├── SOUL.md              ← Agent 身份 + SCL 工作原则
├── HEARTBEAT.md         ← 心跳检查清单（含 R-CCAM 结构）
├── TASKS.md             ← 任务树 + 执行摘要（R 读，M 写）
├── MEMORY.md            ← 长期精华记忆（每次 session 加载）
├── memory/
│   └── YYYY-MM-DD.md    ← Daily Log（M 阶段 append）
└── skills/
    ├── scl-control/         ← Control 层（执行前验证）
    ├── scl-memory/          ← Memory 层（跨轮持久化）
    ├── recap-decompose/     ← 目标分解（首次 + 定期重分解）
    └── capability-gap-handler/  ← 能力缺口自举
```

---

## 常见问题

**Q：Agent 第一次心跳会做什么？**  
A：读取 TASKS.md → 发现没有任务树 → 调用 recap-decompose 生成子任务树 → 执行第一个节点 → 通过 scl-memory 写回状态。

**Q：我已有 HEARTBEAT.md，覆盖会不会丢失原有检查项？**  
A：不要覆盖，按上方「合并」说明追加 R-CCAM 检查清单到文件末尾，原有检查项完全保留。

**Q：如何查看 Agent 在做什么？**  
A：查看 `~/.openclaw/workspace/memory/YYYY-MM-DD.md`（今天的 Daily Log）或 `TASKS.md` 的执行摘要表格。

**Q：Agent 发现能力缺口时会通知我吗？**  
A：若只需要 web_search + write_file 就能解决，会静默自举并在下一次心跳继续。若需要 API Key 或账户权限，会通过你的频道通知你并说明配置方法。

**Q：如何暂停 Agent？**  
A：`openclaw gateway stop` 停止 Gateway；或在 TASKS.md 顶层目标下方添加一行 `**暂停**`，Agent 会在当轮结束后停止心跳执行。

**Q：心跳间隔设多少合适？**  
A：取决于任务性质。社媒运营类建议 30m~60m；监控告警类建议 5m~15m；深度分析类建议 2h~4h。
