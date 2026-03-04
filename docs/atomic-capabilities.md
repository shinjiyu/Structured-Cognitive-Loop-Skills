# 原子能力集定义

## 什么是原子能力

原子能力是 Agent 自举系统的基础——这些能力不可被其他能力组合派生，但它们的组合可以派生出任何其他能力。判断标准：

| 标准 | 含义 |
|------|------|
| **不可派生** | 无法由其他原子能力组合得到 |
| **高杠杆** | 能派生出大量其他能力 |
| **领域无关** | 不依赖特定业务场景 |
| **缺失即失能** | 没有它，某类行动彻底无法完成 |

这与计算机科学中「图灵完备」的思想类似——最小原子集使系统具备「能力完备性」：给定足够时间和原子能力，Agent 可以获取任何所需能力。

---

## 原子能力集

### 感知层（Perception）

**`web_search` — 外部世界感知**
- 无法派生：需要网络访问权限，无法从其他原子得到
- 杠杆效果：可派生 skill-discovery（搜索 Skill）、market-research（市场调研）、fact-checking（事实核查）等无数能力
- 缺失后果：Agent 无法获取外部信息，只能在既有知识内打转，必然空转

**`read_file` — 内部状态感知**
- 无法派生：文件系统访问是基础 I/O，无法从推理得到
- 杠杆效果：读取 TASKS.md、MEMORY.md、已有 Skill 等所有持久化状态
- 缺失后果：Agent 每轮都是「失忆」的，无法利用过去的执行结果

**`get_time` — 时间感知**
- 无法派生：「现在是几点」无法从文件内容或网页推断（可能已过期）
- 杠杆效果：可派生调度决策（「现在适合发帖吗？」）、超时检测（「这个任务是否超时？」）、定时任务（「每天 9 点发布」）
- 缺失后果：Agent 无法理解时间约束，无法调度，无法判断任务是否过期

---

### 行动层（Action）

**`write_file` — 持久化内部状态 / 安装新能力**
- 无法派生：文件系统写入是基础 I/O
- 杠杆效果：写 TASKS.md（任务追踪）、写 Daily Log（记忆）、写新 SKILL.md（能力扩展）
- 缺失后果：Agent 无法安装新 Skill，无法持久化任务状态，自举失败

**`shell_exec` — 计算与外部操作**
- 无法派生：需要系统级执行权限
- 杠杆效果：可派生包安装、API 调用、数据处理、任何自动化脚本
- 缺失后果：Agent 只能「思考」，无法执行真实的系统级操作

**`send_message` — 沟通与上报（安全阀）**
- 无法派生：需要与外部通信频道集成（Telegram、Slack 等）
- 杠杆效果：通知用户结果、请求授权（真正阻塞时）、Agent 间协调
- 重要性：这是「完全自主」与「需要人类介入」之间的边界。没有它，Agent 遇到真正阻塞时只能静默停止或危险越权

---

### 智能层（Intelligence）

**`LLM reasoning` — 核心推理**
- 无法派生：是 Agent 存在的基础
- 杠杆效果：支撑所有 Cognition、Control 决策
- 说明：这是唯一不需要显式声明的原子能力，因为它始终存在

---

### 扩展层（Extension）

**`spawn_subagent` — 并行与自我验证**
- 无法派生（单 Agent）：创建并发 Agent 实例需要运行时支持
- 杠杆效果：
  - **并行加速**：多个子任务同时执行
  - **交叉验证**：用另一个 Agent 实例检验自己的输出
  - **专业化**：用不同模型处理不同类型任务
- 缺失后果：所有工作串行，无法自我验证，交叉进化框架失效

**`read_write_structured_state` — 可靠跨轮记忆**
- 注意：不等于 `read_file + write_file`
- 区别：普通文件读写不保证格式一致性；结构化状态保证「上轮写入的 key，本轮能可靠读出，格式不变」
- 实现：本系统用 TASKS.md（结构化 Markdown 表格）+ scl-memory Skill 的读写规范来实现这个语义
- 缺失后果：ReCAP 子任务树无法跨轮持久，Agent 每轮都要重新分解目标

---

### 元层（Meta）

**`capability_gap_handler` — 能力缺口自举**
- 性质：这是一条「元规则」，而非工具调用
- 内容：「当 Action 因工具不存在而失败时，用原子能力（web_search + write_file）派生解法，而不是报错或等待用户」
- 为什么需要显式原子化：
  - 若只写在 SOUL.md 的一段文字里，可能被 context 压缩掉
  - 若作为显式 Skill，永远在 Control 层生效，不可被覆盖
- 实现：`skills/capability-gap-handler/SKILL.md`

---

## 原子能力的杠杆率排序

若必须从完整集合中选出「最关键的 3 个自举原子」：

```
第 1 位：capability_gap_handler（元层）
  → 使所有其他能力都可以被按需派生
  → 没有它，缺口只能靠人类填补

第 2 位：spawn_subagent（扩展层）
  → 使系统可以并行验证和扩展自己
  → 没有它，交叉进化和自我检验都失效

第 3 位：read_write_structured_state（扩展层）
  → 使目标和任务树真正跨越时间持久
  → 没有它，每轮心跳都在重新开始
```

前两个已有原子（web_search + write_file）是**执行层**基础；这三个是**进化层**基础。

---

## 在 OpenClaw 中的对应关系

| 原子能力 | OpenClaw 实现 |
|---------|--------------|
| web_search | Browser 工具 |
| read_file | File 工具（read） |
| write_file | File 工具（write） |
| shell_exec | Shell 工具 |
| get_time | `date` 命令（via shell）或系统事件 |
| send_message | Channel 集成（Telegram / Slack 等） |
| LLM reasoning | 模型推理（自然存在） |
| spawn_subagent | OpenClaw subsession / cron isolated |
| read_write_structured_state | TASKS.md + scl-memory Skill 规范 |
| capability_gap_handler | skills/capability-gap-handler/SKILL.md |
