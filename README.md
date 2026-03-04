# Structured Cognitive Loop Skills

> 一套面向 OpenClaw 的技能系统，让 Agent 在无限心跳循环中可靠执行抽象目标、自我扩展能力边界。

---

## 设计背景

当我们把 AI Agent 从「一次性触发」升级为「定时/连续运行」时，出现了两类根本性问题：

**问题一：错误积累**  
长期运行中，小错误若未及时纠正会积少成多，最终导致系统行为偏离预期甚至崩溃。

**问题二：输入枯竭与空转**  
若 Agent 只在既有技能和概念内循环，输入空间会逐渐固定，产出价值递减，系统进入空转。

同时，当用户给出的是高级抽象目标（如「帮我运营小红书」「变得更像人」）时，传统的「先制定计划再执行」范式面临根本性局限：

- **计划制定时的假设在多轮后已失效**
- **第一步的搜索空间过大，难以决策**
- **没有人在旁边审阅计划**

本系统通过三个互补框架的组合解决上述问题：

| 框架 | 解决的问题 |
|------|-----------|
| **SCL（R-CCAM）** | 每步可靠执行：记忆不丢失、行动前验证、决策可追溯 |
| **ReCAP** | 抽象目标分解：将无限搜索空间折叠为有限任务树 |
| **能力缺口自举** | 自我扩展：发现能力缺口时，用原子能力派生新技能 |

三者运行在 OpenClaw 的**心跳循环**（Heartbeat）上——每 N 分钟唤醒一次，每次心跳 = 一轮 R-CCAM 执行。无限心跳 = 无限执行能力。

---

## 目录结构

```
Structured-Cognitive-Loop-Skills/
├── README.md                          # 本文件
├── INSTALL.md                         # 安装指南
│
├── docs/                              # 设计文档（理解框架用）
│   ├── design-background.md           # 设计背景与动机（详细版）
│   ├── architecture.md                # 整体架构与运行机制
│   ├── atomic-capabilities.md         # 原子能力集定义
│   └── memory-system.md               # 记忆系统设计
│
├── skills/                            # OpenClaw Skills（安装到 .openclaw/skills/）
│   ├── goal-setting/SKILL.md          # 目标设定入口：解析→写入→验证→确认
│   ├── scl-control/SKILL.md           # Control 层：执行前约束验证
│   ├── scl-memory/SKILL.md            # Memory 层：结构化跨轮状态读写
│   ├── recap-decompose/SKILL.md       # ReCAP：抽象目标分解为任务树
│   └── capability-gap-handler/SKILL.md # 能力缺口元规则：自举派生新能力
│
└── templates/                         # OpenClaw 配置文件模板
    ├── SOUL.md                        # Agent 身份模板（含 SCL 工作原则）
    ├── HEARTBEAT.md                   # 心跳检查清单（R-CCAM 结构）
    ├── TASKS.md                       # 任务追踪文件（ReCAP 子任务树）
    └── MEMORY.md                      # 长期记忆文件骨架
```

---

## 快速开始

```bash
# 1. 克隆本仓库
git clone https://github.com/shinjiyu/Structured-Cognitive-Loop-Skills.git

# 2. 将 Skills 复制到 OpenClaw workspace
cp -r skills/* ~/.openclaw/workspace/skills/

# 3. 用模板初始化 Agent 配置文件
cp templates/SOUL.md     ~/.openclaw/workspace/SOUL.md
cp templates/HEARTBEAT.md ~/.openclaw/workspace/HEARTBEAT.md
cp templates/TASKS.md    ~/.openclaw/workspace/TASKS.md
cp templates/MEMORY.md   ~/.openclaw/workspace/MEMORY.md

# 4. 启动 OpenClaw Gateway
openclaw gateway start

# 5. 给 Agent 下达抽象目标（之后不需要再提供任何指令）
# 例："帮我运营小红书账号，目标是三个月内涨粉 1000 人"
```

详细安装步骤见 [INSTALL.md](INSTALL.md)。

---

## 核心理念

**心跳是无限 loop，每次心跳是一轮 R-CCAM：**

```
每次心跳
  R - Retrieval：读取 TASKS.md + MEMORY.md，了解目标与当前状态
  C - Cognition：LLM 推理当前最优下一步（ReCAP 树已折叠搜索空间）
  C - Control：执行前验证，约束检查
  A - Action：执行单步最优行动
  M - Memory：写回结果，更新任务树
等待下一次心跳
```

**能力缺口自举：**
```
Action 失败（工具不存在）
  ↓ Control 层检测为能力缺口
  ↓ 用原子能力（web_search + write_file）派生新 Skill
  ↓ 安装，下一次心跳生效
  ↓ 继续原任务
```

---

## 设计文档

- [设计背景与动机](docs/design-background.md)
- [整体架构](docs/architecture.md)
- [原子能力集](docs/atomic-capabilities.md)
- [记忆系统设计](docs/memory-system.md)

---

## 相关框架参考

- **SCL**：[arxiv.org/abs/2510.15952](https://arxiv.org/abs/2510.15952)（Myung Ho Kim, 2025）
- **ReCAP**：[arxiv.org/abs/2510.23822](https://arxiv.org/abs/2510.23822)（Stanford + MIT, NeurIPS 2025）
- **ReAct**：[arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629)（Google + Princeton, ICLR 2023）
- **OpenClaw**：[github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
