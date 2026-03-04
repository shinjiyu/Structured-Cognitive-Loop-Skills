# 范式跃迁：从 Plan-Execute 到 Heartbeat Loop

> 本文记录 AI Agent 运行模式的一次根本性转变，以及 OpenClaw Heartbeat 为何是这次转变的关键实现。

---

## 第一代：Vibe Coding（Plan-Execute 范式）

```
用户输入 → [制定详细计划] → 按计划执行 → 完成
```

这是最直觉的范式：**先想清楚，再动手**。

Plan 是核心产物，执行是附属物。Agent 的运行时间越长越好，Plan 越详细越好，一次完成越多越好。

**致命弱点**：

- 被打断 = 一切作废（Plan 和执行状态都在上下文里，丢了就没了）
- Plan 制定时的假设在多轮后已失效（世界在变，Plan 没变）
- 第一步的搜索空间过大，难以决策（目标越抽象，Plan 越不可靠）

---

## 第二代：Ralph Loop（无限 bash 循环）

Geoffrey Huntley 提出的 Ralph 技术，其最纯粹的形式是一个 bash 循环：

```bash
while :; do cat PROMPT.md | claude-code ; done
```

Ralph Loop 的**本意**只有一件事：**解决长任务被打断后需要手动重启的问题**。把 Agent 套进 bash while 循环，中断后自动重试，状态通过文件持久化。这是一个纯粹的工程 workaround，不是范式设计。

但它**碰巧暗示了一个新方向**：

> 如果 Agent 天然就会被反复唤醒，「不追求单次长时间运行」就变成了一个合理的设计选择。

注意这个方向是 Ralph Loop **结构上的副作用**，不是它的目标——Ralph 本身并没有刻意设计「每轮只做一件事」或「用无限循环替代 Plan」，这只是 Huntley 在实践中摸索出的使用技巧。

**局限**：Ralph 没有结构化的记忆层、心跳调度或认知框架，更适合 Greenfield 场景的短期冲刺，难以支撑需要长期状态管理的抽象目标。

---

## 第三代：Heartbeat Loop（心跳循环范式）

```
持久化的 [目标] + 持久化的 [现状]
         ↓ 心跳唤醒
   选择当前最优单步行动
         ↓
   执行 → 更新现状到持久存储
         ↓ 心跳唤醒（下一次）
   选择当前最优单步行动
         ↓
   执行 → 更新现状到持久存储
         ↓ 无限重复...
```

OpenClaw 的 Heartbeat 机制是这个范式的关键实现。它改变了三件事：

### 1. Plan 不再是核心

传统 Vibe Coding 里，Plan 是 Agent 最重要的输出，要在开始时就制定好。

Heartbeat 范式里，**只有两样东西是真正必需的**：
- **目标**（Goal）：持久存储，几乎不变，是每一轮决策的北极星
- **现状**（State）：每轮更新，反映真实世界的最新情况

Plan（任务树）成了辅助工具——它可以帮助结构化决策，但不再是必须在开始时制定完整的东西。它可以随着现状的变化被动态修订。

### 2. 每一轮心跳的决策极其简单

```
policy(当前状态) → 最优单步行动
```

每次心跳做的事只有一件：**在当前状态下，哪一步最能接近目标？走它。**

这与强化学习的核心机制高度同构——LLM 是天然的 policy 函数，心跳是时间步，TASKS.md + MEMORY.md 是 state representation。

### 3. 容错性与时间正相关

这是最反直觉、也最重要的性质：

| 范式 | 运行越久... |
|------|-------------|
| Plan-Execute | 越脆弱（错误积累，Plan 失效） |
| Heartbeat Loop | 越稳健（每轮独立决策，错误自然被下一轮纠正） |

Heartbeat Loop 中，**每一轮都是一个完整的、从头开始的决策**。上一轮出错，不影响这一轮的决策质量。只要目标和现状被正确记录，系统永远不会真正「崩溃」，只会「暂时偏离」然后被下一轮拉回来。

---

## 关键认识反转

| | Plan-Execute | Heartbeat Loop |
|--|--------------|----------------|
| **核心产物** | 完整的 Plan | 准确的 State |
| **执行单位** | 整个 Plan | 单步行动（per heartbeat）|
| **被打断后** | 一切作废 | 下次心跳继续 |
| **对抗变化** | 弱（Plan 固化） | 强（每轮重新决策）|
| **容错性** | 随时间递减 | 随时间递增 |
| **对 Agent 的要求** | 强规划能力 | 强状态感知能力 |

---

## OpenClaw Heartbeat 的真正价值

人们在使用 OpenClaw 时，往往关注的是其 Skill 系统、Memory 系统等具体功能。

但 **Heartbeat 才是 OpenClaw 最有价值的部分**——它是上述范式转变的基础设施。

没有 Heartbeat，所有的 Skill 和 Memory 都只是对 Plan-Execute 范式的增强。有了 Heartbeat，它们变成了 State-Action Loop 的组件：

- **SOUL.md** → 定义 policy 的偏好和约束
- **TASKS.md** → State 的结构化表示
- **MEMORY.md** → 跨时间的长期 State 压缩
- **Daily Logs** → 短期 State 的完整记录
- **Skills** → 可用的 Action 集合
- **HEARTBEAT.md** → 每一轮 policy 执行的 SOP

---

## 对本系统设计的影响

这个范式认识直接影响了本系统（Structured Cognitive Loop Skills）的设计：

1. **goal-setting Skill**：目标是 Loop 的锚点，必须被显式设定、持久化、验证
2. **scl-memory Skill**：State 的质量决定每轮决策的质量，内存写入不是可选的
3. **recap-decompose Skill**：任务树是辅助工具（帮助缩小搜索空间），而非必须提前完成的 Plan
4. **HEARTBEAT.md 的 Pre-Check**：在进入 R-CCAM 之前先确认目标和现状存在——因为没有这两样，loop 没有意义

本质上，SCL（R-CCAM）是对 Heartbeat Loop 的**认知科学层面的形式化**：把每一轮的「state → action」决策分解为可追溯、可验证的子步骤。

---

## 一句话总结

> Vibe Coding 让 AI 会"想"；Ralph Loop 让 AI 会"不断重试"；Heartbeat 让 AI 会"活着"。
>
> 活着的 AI 不需要完美的 Plan，它只需要知道自己在哪、要去哪——然后一直走下去。
