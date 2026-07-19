---
layout: post
title: "为 Agent 注入时间感知：Hook 机制实现"
date: 2026-07-19
categories: ai agent
---

**本文采用 Hook 机制为 Agent 注入时间感知能力**，实现方式为 UserPromptSubmit 事件 Hook + 全局规则声明，Shell 脚本共 23 行。Agent 的时间感知问题指的是：模型权重是静态的，训练数据有截止日期，推理时没有内置时钟，它的时间停在上一次预训练那天。一周前的会话今天接着聊，对用户来说过了一周，对 AI 来说时间没有流动。目前不少 Agent 框架通过系统提示词注入时间，本文介绍一种基于 Hook 的通用实现，详细介绍如下。

## 一、LLM没有时间感知

### 1. 现象

随便问一个时效性相关问题："最近3年云栖大会都是什么时候开的？"

一个没有时间感知的 Agent 其实不知道"最近3年"指哪三年。它可能翻训练数据里的旧记忆（这个时间大概率是过时的），可能瞎编一个日期，更可能直接卡住——"最近"这个词需要一个基准点，而这个基准点它根本没有。类似的还有：晚上十一点聊天它说"美好的一天开始了"，周一问它事情它祝你"周末愉快"。这些都属于是系统性的时间感知缺失

### 2. 原因

这并不是模型本身能力的问题，大模型推理时权重已经被冻结，没有进程级的时钟，没有系统调用，拿不到当前时间。除非在 prompt 里告诉它，否则它只能拿训练数据里"见过的最近日期"去猜，而这个日期可能是一年前。

Fine-tune 不能解决，时间一直在变，没法训进权重。接 API 拿服务器时间可以，但每次对话多一次调用，延迟、成本、依赖链都增加。最简方案是：**每次用户提交 prompt 时，往上下文注入一行当前时间**。不改模型，不微调，不调外部 API，一个 Hook 搞定

## 二、时间注入 Hook

主流 Agent 工具大多提供 Hook 机制，允许在特定事件点执行自定义脚本。`UserPromptSubmit` 事件在每次用户提交 prompt 时触发，脚本的输出可以通过 `hookSpecificOutput.additionalContext` 字段注入到模型上下文中。整个方案分三步，从注册到执行到声明，缺一不可。

### 1. 注册 Hook

在配置文件中注册 `UserPromptSubmit` 事件：

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "~/.qoder/hooks/inject-time.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

`matcher: "*"` 表示所有 prompt 都触发，`timeout: 5` 防止脚本卡死阻塞对话。这一层只负责一件事：用户说话时，跑一下这个脚本。

### 2. 执行脚本

`inject-time.sh`，23 行，全部逻辑在这里：

```bash
#!/bin/bash
# UserPromptSubmit Hook — 时间注入(极简版)
# 每次提交 Prompt 时向 AI 上下文注入当前系统时间。
# 仅记录一行时间戳到 audit 目录，不记录任何聊天内容。
# Fail-Open: hook 出错绝不影响宿主。
trap 'exit 0' ERR EXIT
set +e +u

# 丢弃 stdin(不解析 prompt，避免管道阻塞)
cat >/dev/null 2>&1

NOW="$(date '+%Y-%m-%d %H:%M:%S' 2>/dev/null)"
WEEKDAY="$(date '+%A' 2>/dev/null)"

# 极简日志: 仅时间戳，统一到 audit 目录，不含聊天内容
LOG_FILE="${HOME}/.qoder/audit/inject-time.log"
mkdir -p "${HOME}/.qoder/audit" 2>/dev/null
printf '[%s]\n' "$NOW" >> "$LOG_FILE" 2>/dev/null || true

# 注入时间上下文 (JSON 格式: hookSpecificOutput.additionalContext)
[ -n "$NOW" ] && printf '{"hookSpecificOutput":{"hook_event_name":"UserPromptSubmit","additionalContext":"当前系统时间: %s %s"}}' "$NOW" "$WEEKDAY"
exit 0
```

脚本很简单，功能大致如下：

- **取时间**：精确到秒，带完整星期名
- **注入上下文**：通过 `additionalContext` 把时间塞进模型上下文，最终模型看到的是 `当前系统时间: 2026-07-16 21:01:35 Thursday`
- **审计日志**：只记一行时间戳，不含任何聊天内容
- **Fail-Open**：任何异常静默退出，绝不影响对话；stdin 直接丢弃，不解析、不存储用户输入

### 3. 声明能力

光注入时间不够，得告诉 Agent"你有这个能力"，否则那行时间戳只是上下文里的噪音。推荐是在全局规则文件里加一条：

```markdown
## 时间感知
- 每轮会话会自动触发hook注入时间,注入时间格式: %Y-%m-%d %H:%M:%S
  所以你是具备时间感知能力的, 回答用户问题需要有时间意识,优先使用注入时间
```

模型推理时会读 system prompt，这条规则相当于一份能力声明。有了它，Agent 才会主动把时间作为推理锚点，而不是把注入的时间戳当成无关信息忽略掉

## 三、效果对比

**无时间注入的 Agent**：面对"最近3年云栖大会都是什么时候开的"，不知道"最近"的基准点，可能拿训练数据截止日期当基准，可能直接猜。调用搜索工具时查询意图缺少时间锚点，搜"最新"可能翻出几年前的结果，搜"上周"无法锚定到具体日期区间 并不固定
> 当然， 随着模型推理能力愈发强大、Agent足够完善，其实这类意图可能会被 LLM识别， 继而优先调用时间工具获取当前时间， 但是这种依然是不够稳定的， 因为时间注入其实完全依赖当时LLM是否觉得去看下当前几点了
> 

**注入时间后的 Agent**：看到上下文里的 `2026-07-16`，立刻知道"最近3年"是 2024、2025、2026，然后主动调搜索工具查这三年的实际举办日期。搜"最新"不会翻出 2023 年的结果，搜"上周"能锚定到上周一到周日。时间戳成了理解相对时间词的基准
![2026-07-18-11-38-25](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images//5c6ef0a2672913646ab9e09401f37feb.png)

真人感的差别更明显，加上时间感知后， 晚上十一点它不会回复你说"美好的一天开始了"，周一不会祝你"周末愉快"，周四聊到周末计划接得住。这些细节单独看没什么，累积起来就是像人和不像人的分界。它不是调参调出来的，是上下文里有没有"当下"这个锚点



> 本文配置以 Qoder 为例（配置文件 `~/.qoder/settings.json`，规则文件 `~/.qoder/rules/system.md`），其他支持 Hook 机制的同类 Agent 工具按需调整路径和字段即可，思路完全通用。

## 参考文档

- [Qoder Hooks 官方文档](https://docs.qoder.com/zh/qoderwork/hooks#hooks)
- [Claude Code Hooks 参考（同类机制）](https://docs.anthropic.com/en/docs/claude-code/hooks)