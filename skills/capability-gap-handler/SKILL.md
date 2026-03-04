---
name: capability-gap-handler
description: 能力缺口自举元规则。当任何 Action 因工具不存在或 Skill 缺失而失败时，自动触发：用原子能力（web_search + write_file + shell_exec）派生新 Skill，安装后下一次心跳继续原任务。这是系统自举能力的核心，确保 Agent 在无人干预的情况下扩展自身能力边界。
---

# 能力缺口自举

## 核心原则

**Agent 天然具备一组原子能力（web_search + write_file + shell_exec + LLM 推理），这些原子能力的组合可以派生出任何其他能力。**

因此：遇到能力缺口时，**首选自我派生，而不是请求用户或放弃任务**。

只有以下情况才真正需要上报用户：
- 需要特定的 API 密钥或账户权限（超出原子能力的外部依赖）
- 涉及金钱支出的操作需要授权
- 法律/合规层面需要人类确认的操作

---

## 触发条件

当以下任一情况发生时，本 Skill 自动触发：

- `ToolNotFound`：调用的工具不存在
- `SkillNotFound`：所需 Skill 文件不在 workspace
- `CapabilityMissing`：LLM 推理认为完成当前任务需要一个当前没有的能力
- `RepeatedFailure`：同一个 Action 因「缺少某类能力」连续失败 2 次

---

## 自举流程

### Step 1：能力缺口诊断

```
明确描述：
  「我需要完成 [任务]，为此需要 [能力 X]，
   但当前没有实现 X 的工具或 Skill」

分类：
  A. 可完全用原子能力派生 → 进入 Step 2
  B. 需要外部依赖（API Key / 账户权限）→ 进入 Step 2 + 最后 send_message 请求配置
  C. 涉及法律/合规/金钱 → 直接 send_message 请求人类确认
```

### Step 2：搜索解决方案

```
执行以下搜索（选择最相关的 1-2 条）：

web_search("[能力 X] openclaw skill implementation")
web_search("[能力 X] API free tier 2026")
web_search("how to [能力 X] using [相关工具]")

评估搜索结果：
  - 优先选择：有明确 API、步骤清晰、无安全风险的方案
  - 排除：需要付费 / 需要注册账号 / 来源不可信 的方案
```

### Step 3：设计新 Skill

基于搜索结果，设计新 Skill 的结构：

```
Skill 名称：[能力 X 的英文简写，如 image-generation]
功能描述：[一句话说明做什么]
实现方式：[API 调用 / shell 命令 / 工具组合]
需要配置：[API Key 名称 / 环境变量 / 账号 / 无]
```

### Step 4：写入新 Skill 文件

```bash
write_file(
  path: "~/.openclaw/workspace/skills/[skill-name]/SKILL.md",
  content: """
---
name: [skill-name]
description: [功能描述]. 由能力缺口自举系统自动生成于 YYYY-MM-DD.
---

# [Skill 名称]

## 功能
[完整的使用说明和步骤]

## 配置要求
[若需要 API Key 等，说明如何配置]

## 使用示例
[具体的调用示例]
  """
)
```

### Step 5：更新记忆与恢复计划

```
1. 写入 Daily Log：
   「[时间] 能力缺口自举：发现缺少 [X]，已生成 Skill [name]
    来源：[URL]，可信度评估：[高/中/低]」

2. 更新 TASKS.md：
   - 当前失败节点标记为「[!] 缺少 [X]，已安装 [name]，下轮重试」
   - 在「已安装 Skill 记录」添加一行

3. 若需要外部配置（API Key 等）：
   send_message（给用户）：
   「我发现完成 [任务] 需要 [X] 能力，我已生成了对应的 Skill，
    但需要配置 [API_KEY_NAME]。
    配置方法：[具体步骤]
    配置完成后，我会在下一次心跳自动继续。」
```

### Step 6：下一次心跳自动继续

OpenClaw 在每次 Agent Loop 开始时加载最新的 Skills 快照。因此：
- 若不需要外部配置：下一次心跳，新 Skill 自动加载，重试原任务
- 若需要外部配置：用户配置后，下一次心跳自动继续

---

## 自举的进化效应

多次 ad-hoc 派生同类 Skill 后，active-learning 会识别模式并提议固化：

```
第 1 次：缺少图片生成 → web_search + write_file → 安装 DALL-E Skill
第 2 次：缺少视频生成 → web_search + write_file → 安装 RunwayML Skill
第 3 次：缺少音频生成 → web_search + write_file → 安装 ElevenLabs Skill

active-learning 识别：「反复做媒体生成 Skill 的 ad-hoc 安装」
提议：「建立 media-generation-registry，预收录常见媒体生成 Skill」
→ 下一次遇到类似缺口，直接从 Registry 安装，不再 web_search
```

这就是自举如何演化为主动能力管理的过程。

---

## 安全边界

以下情况**永远不自动安装，必须 send_message 请求用户确认**：

- Skill 涉及发送真实消息给真实用户（防止垃圾信息）
- Skill 涉及金融交易或支付
- Skill 需要访问未经授权的第三方账户
- Skill 的代码来源不可信（无法验证安全性）
- Skill 会修改系统级配置或删除文件
