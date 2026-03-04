# 安装指南

## 前置条件

- [OpenClaw](https://github.com/openclaw/openclaw) 已安装（v2026+）
- Node.js ≥ 22
- OpenClaw Gateway 可正常启动

---

## 安装步骤

### 1. 克隆本仓库

```bash
git clone https://github.com/shinjiyu/Structured-Cognitive-Loop-Skills.git
cd Structured-Cognitive-Loop-Skills
```

### 2. 安装 Skills

将四个核心 Skill 复制到 OpenClaw workspace：

```bash
# 确认 workspace 路径（默认 ~/.openclaw/workspace）
WORKSPACE=~/.openclaw/workspace

# 创建 skills 目录（若不存在）
mkdir -p $WORKSPACE/skills

# 复制四个核心 Skill
cp -r skills/scl-control          $WORKSPACE/skills/
cp -r skills/scl-memory           $WORKSPACE/skills/
cp -r skills/recap-decompose      $WORKSPACE/skills/
cp -r skills/capability-gap-handler $WORKSPACE/skills/
```

### 3. 初始化配置文件

将模板复制到 workspace，然后按需填写：

```bash
# 复制模板（注意：会覆盖已有文件，先备份）
cp templates/SOUL.md      $WORKSPACE/SOUL.md
cp templates/HEARTBEAT.md $WORKSPACE/HEARTBEAT.md
cp templates/TASKS.md     $WORKSPACE/TASKS.md
cp templates/MEMORY.md    $WORKSPACE/MEMORY.md

# 创建 memory 目录（用于 Daily Log）
mkdir -p $WORKSPACE/memory
```

### 4. 填写配置文件

**SOUL.md**：填写 Agent 名称、角色描述、已安装的领域 Tools

**TASKS.md**：填写你的顶层目标和约束条件（任务树由 Agent 首次运行时自动生成）

**MEMORY.md**：填写约束条件和已有的领域知识（可选）

### 5. 配置心跳间隔（可选）

在 OpenClaw 的 `config.json5` 中配置心跳：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",           // 心跳间隔（30分钟）
        target: "last",         // 通知目标
        activeHours: {
          start: "08:00",
          end: "22:00"          // 只在活跃时间段运行
        }
      }
    }
  }
}
```

### 6. 启动 Gateway

```bash
openclaw gateway start
```

### 7. 给 Agent 下达目标

通过你配置的 Channel（Telegram / Slack 等）发送目标：

```
帮我运营小红书账号，目标是三个月内涨粉 1000 人。
账号赛道是生活方式，目前粉丝数约 200 人。
约束：不使用付费推广，不涉及违规操作。
```

之后不需要再提供任何指令，Agent 会通过心跳循环自主执行。

---

## 验证安装

运行一次心跳确认 Skills 正常加载：

```bash
# 手动触发一次心跳（用于测试）
openclaw agent --message "请执行一次完整的 SCL R-CCAM 心跳流程，并报告每个阶段的执行情况"
```

期望输出应包含：
- `[SCL-Control]` 的验证报告
- `[SCL-Memory]` 的写入确认
- TASKS.md 已创建（若首次运行，recap-decompose 会生成任务树）

---

## 目录结构说明（安装后）

```
~/.openclaw/workspace/
├── SOUL.md              ← Agent 身份（每次 session 加载）
├── HEARTBEAT.md         ← 心跳检查清单
├── TASKS.md             ← 任务追踪（R 读，M 写）
├── MEMORY.md            ← 长期精华记忆（每次 session 加载）
├── memory/
│   └── YYYY-MM-DD.md    ← Daily Log（M 阶段 append）
└── skills/
    ├── scl-control/     ← Control 层
    ├── scl-memory/      ← Memory 层
    ├── recap-decompose/ ← 目标分解
    └── capability-gap-handler/ ← 能力缺口自举
```

---

## 常见问题

**Q：Agent 第一次心跳会做什么？**  
A：读取 TASKS.md 发现没有任务树 → 调用 recap-decompose → 生成子任务树 → 执行第一个节点 → 写回状态。

**Q：如何查看 Agent 在做什么？**  
A：查看 `~/.openclaw/workspace/memory/YYYY-MM-DD.md`（今天的 Daily Log）或 `TASKS.md` 的执行摘要表格。

**Q：Agent 发现能力缺口时会通知我吗？**  
A：若只需要 web_search + write_file 就能解决，会静默自举并在下一次心跳继续。若需要 API Key 或账户权限，会通过 Channel 通知你。

**Q：如何暂停 Agent？**  
A：`openclaw gateway stop` 停止 Gateway，或在 TASKS.md 的顶层目标下方添加 `**暂停**` 标记，Agent 会在当轮结束后停止。
