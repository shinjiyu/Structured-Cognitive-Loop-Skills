# 整体架构与运行机制

## 全局视图

```
用户：「帮我运营小红书，三个月涨粉 1000」（之后不再给任何指令）
                    │
                    ▼
        OpenClaw Gateway（持续运行）
                    │
          每 N 分钟 Heartbeat 触发
                    │
        ┌───────────▼──────────────┐
        │      单次心跳执行          │
        │                          │
        │  R  读取 TASKS.md         │
        │     + MEMORY.md          │
        │     + Daily Log          │
        │         ↓                │
        │  C  ReCAP 决策当前子任务   │
        │     （搜索空间已被折叠）    │
        │         ↓                │
        │  C  Control 验证          │
        │     （约束 / 安全检查）    │
        │         ↓                │
        │  A  执行单步行动           │
        │     （工具调用）           │
        │         ↓                │
        │  M  写回 TASKS.md         │
        │     + Daily Log          │
        └───────────────────────────┘
                    │
          等待下一次心跳
                    │
          ...... 循环 ......
```

---

## 心跳与 Agent Loop 的关系

OpenClaw 的 **Heartbeat** 是调度机制，**Agent Loop** 是每次心跳内的单次执行：

```
Heartbeat（调度层）
  ├── 触发条件：每 N 分钟（默认 30min，可配置）
  ├── 上下文：与主 session 共享，记得最近的对话和任务状态
  └── 检查清单：读取 HEARTBEAT.md，决定本次做什么

Agent Loop（执行层）
  ├── intake → context assembly → model inference → tool execution → persistence
  └── 一次 Heartbeat = 一次 Agent Loop
```

每次心跳 = 一轮 R-CCAM 执行。无限心跳 = 持续执行长期目标。

---

## R-CCAM 与 OpenClaw 组件的映射

| R-CCAM 阶段 | OpenClaw 组件 | 具体文件/工具 |
|------------|--------------|-------------|
| **R - Retrieval** | Bootstrap context 自动注入 | MEMORY.md（长期精华）+ Daily Log（近期）+ TASKS.md（任务树） |
| **C - Cognition** | LLM 推理 | 模型调用（自然发生） |
| **C - Control** | `scl-control` Skill | 执行前约束验证规则 |
| **A - Action** | 工具执行 | web_search / write_file / shell_exec / 领域 Skill 等 |
| **M - Memory** | `scl-memory` Skill | 写 Daily Log + 更新 TASKS.md + 必要时提炼到 MEMORY.md |

---

## ReCAP 在系统中的位置

ReCAP 运行在 **Cognition 阶段内部**，负责「当前应该做什么」的决策：

```
C - Cognition
  ├── 读取 TASKS.md 中的子任务树
  ├── 识别当前应执行的节点
  ├── 若任务树不存在（首次心跳）：
  │     执行 ReCAP 分解，生成子任务树写入 TASKS.md
  ├── 若任务树存在：
  │     选择下一个 pending 节点作为本轮执行目标
  └── 输出：「本轮执行目标」+ 「执行策略」
```

每隔 K 次心跳（或当任务树全部完成），触发 ReCAP 重分解：
- 基于已完成节点的实际效果
- 修正剩余树（新发现可能改变方向）
- 写回 TASKS.md

---

## 能力缺口自举的触发流程

```
A - Action 阶段
  │
  ├── 执行成功 → 进入 M 阶段
  │
  └── 执行失败
        │
        ├── 是普通错误（网络超时、参数错误等）
        │     → 重试或降级，记录到 Daily Log
        │
        └── 是能力缺口（ToolNotFound / 无对应 Skill）
              │
              → 进入 Capability Bootstrap 子流程：
                  1. 识别缺少的能力类型
                  2. web_search 搜索解决方案或现成 Skill
                  3. Control 验证找到的 Skill 是否安全可信
                  4. write_file 写入新 Skill 到 workspace
                  5. Memory 记录「已安装 X，原任务下轮重试」
              │
              → 下一次心跳，新 Skill 自动加载，继续原任务
```

关键：Agent **不会停下来等用户**，除非原子能力也无法解决（才算真正的阻塞，才上报用户）。

---

## 记忆系统与任务追踪

基于 OpenClaw 四层记忆，本系统使用以下分工：

```
TASKS.md（新增）
  ├── 当前顶层目标（用户给的抽象目标）
  ├── ReCAP 生成的子任务树（节点 + 状态 + 父子关系）
  └── 每轮执行摘要（简短，详细见 Daily Log）

Daily Log（memory/YYYY-MM-DD.md）
  └── 流水账：每轮具体行动、工具输出、执行结果

MEMORY.md（长期精华）
  ├── 目标约束（不变的部分：合规边界、预算限制等）
  ├── 已发现的重要领域知识（如「小红书算法倾向竖图」）
  └── 已安装的 Skill 清单（自举记录）

Semantic Search（memory_search 工具）
  └── 当记忆量大时，按需检索历史日志和精华记忆
```

---

## 文件依赖关系

```
SOUL.md          ← Agent 身份与 SCL 工作原则（每次 session 加载）
HEARTBEAT.md     ← 心跳检查清单（R-CCAM 结构，每次心跳读取）
TASKS.md         ← 任务树（R 阶段读，M 阶段写）
MEMORY.md        ← 长期精华（R 阶段读，定期提炼写入）
memory/          ← Daily Log（M 阶段 append）
skills/
  scl-control/   ← Control 层 Skill（C 阶段）
  scl-memory/    ← Memory 层 Skill（M 阶段）
  recap-decompose/ ← ReCAP 分解 Skill（C 阶段内）
  capability-gap-handler/ ← 能力缺口元规则（A 失败时触发）
```
