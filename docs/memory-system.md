# 记忆系统设计

## OpenClaw 四层记忆概览

OpenClaw 原生提供四层记忆，本系统在此基础上新增 `TASKS.md` 作为任务追踪层：

```
Layer 0：TASKS.md（本系统新增）
  └── ReCAP 子任务树 + 当前执行状态（结构化跨轮任务追踪）

Layer 1：Context Window（工作记忆）
  └── 当前 session 的对话上下文，会话结束即清空
  └── 触发 Memory Flush 后，重要内容写入 Layer 2/3

Layer 2：Daily Logs（日志记忆）
  └── memory/YYYY-MM-DD.md，Append-only
  └── 每次 session 启动自动加载今天 + 昨天的日志

Layer 3：MEMORY.md（长期精华记忆）
  └── 结构化、按主题组织的持久事实
  └── 通过 boot-md hook 在每次 session 启动时注入
  └── ⚠️ 仅在私人 session 加载

Layer 4：Semantic Search（语义检索）
  └── SQLite FTS5（关键词）+ 向量 Embedding
  └── 混合检索：BM25 70% + Vector 30%（可配置）
  └── 当记忆量大时按需检索，不占用启动时的 context
```

---

## TASKS.md：结构化任务追踪（核心新增）

### 设计目的

TASKS.md 是 SCL R-CCAM 中「结构化跨轮状态」的载体：
- **R 阶段读取**：了解当前目标与任务树
- **M 阶段写入**：更新任务状态，记录本轮摘要
- **ReCAP 读写**：任务分解结果存这里，修正也在这里

### 文件结构

```markdown
# TASKS.md

## 顶层目标
[用户给出的抽象目标，永久保留，不修改]

## 约束条件
[从 MEMORY.md 提炼的不变约束]

## 任务树（ReCAP 生成，动态更新）
### 第 N 轮 ReCAP（YYYY-MM-DD）
- [ ] 1. 一级子任务
  - [x] 1.1 已完成子任务（完成时间：YYYY-MM-DD）
  - [ ] 1.2 进行中子任务（当前心跳）
  - [ ] 1.3 待执行子任务
- [ ] 2. 一级子任务
  ...

## 执行摘要（最近 5 轮）
| 心跳时间 | 执行子任务 | 结果 | 下轮计划 |
|---------|-----------|------|---------|
| ...     | ...       | ...  | ...     |

## 已安装 Skill 记录（能力缺口自举）
| 安装时间 | Skill 名称 | 原因 | 来源 |
|---------|-----------|------|------|
| ...     | ...       | ...  | ...  |
```

### 写入规范

- **任务状态**：`[ ]` pending / `[~]` in-progress / `[x]` done / `[!]` failed
- **每轮只更新当轮**：不回溯修改历史节点（Append-only 精神）
- **摘要保留最近 5 轮**：旧摘要归档到 Daily Log

---

## 各记忆层的写入时机

| 层 | 何时写入 | 写入内容 |
|----|---------|---------|
| TASKS.md | 每次心跳结束（M 阶段）| 任务状态更新 + 执行摘要 |
| TASKS.md | ReCAP 分解后（C 阶段）| 子任务树 |
| Daily Log | 每次心跳结束（M 阶段）| 详细执行记录（Append） |
| Daily Log | Memory Flush 前 | 当前 context 中的重要内容 |
| MEMORY.md | 定期提炼（每 K 轮）| 重要领域知识 + 持久约束 |
| MEMORY.md | 能力缺口自举后 | 已安装 Skill 记录 |

---

## 各记忆层的读取时机

| 层 | 何时读取 | 读取方式 |
|----|---------|---------|
| TASKS.md | 每次心跳开始（R 阶段）| 直接注入（bootstrap） |
| MEMORY.md | 每次 session 启动 | boot-md hook 自动注入 |
| Daily Log（今/昨）| 每次 session 启动 | 自动注入 |
| Semantic Search | 需要历史信息时 | `memory_search` 工具调用 |

---

## Memory Flush 机制

当 context window 接近上限时，OpenClaw 触发 Memory Flush：

```
1. 系统提示 Agent：「即将压缩 context，请将重要信息写入文件」
2. Agent 执行静默 Agent Turn：
   - 将重要决策、发现写入 Daily Log
   - 将持久性事实提炼写入 MEMORY.md
   - 更新 TASKS.md 中的任务状态
3. 压缩 context（旧消息被移除）
4. 继续执行（重要信息已持久化）
```

这保证了即使 context 被压缩，关键的任务状态和领域知识也不会丢失。

---

## 记忆与 SCL 的协作示意

```
心跳开始
  │
  R - Retrieval
  ├── 自动加载：TASKS.md → 当前任务树
  ├── 自动加载：MEMORY.md → 约束与领域知识
  ├── 自动加载：今天/昨天 Daily Log → 近期上下文
  └── 按需调用：memory_search → 历史特定信息
  │
  [C - Cognition：基于以上信息做决策]
  [C - Control：验证]
  [A - Action：执行]
  │
  M - Memory（scl-memory Skill 执行）
  ├── append Daily Log：本轮行动记录
  ├── update TASKS.md：任务节点状态 + 摘要
  └── 每 K 轮：提炼重要知识到 MEMORY.md
  │
心跳结束，等待下一次
```
