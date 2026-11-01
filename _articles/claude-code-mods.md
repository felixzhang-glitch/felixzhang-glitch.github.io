---
layout: post
title: "Claude Code Mods：把界面开放给插件"
date: 2026-11-01
categories: ai agent
tags: claude-code mods plugin
permalink: /ai/agent/2026/11/01/claude-code-mods.html
---

mod 是 Claude Code 的一层扩展点：插件里写 JavaScript/TypeScript 函数，挂在事件上，就能改工具调用、改 prompt、画界面。本文用一个 demo-mod（统计工具调用次数、显示在 spinner 旁）贯穿，对比 mod 与 settings hook、skill、MCP 的差别，最后落到安全边界和 DeepSeek Harness 的对照

## **一、mod 是什么**

官方定义：

```
A mod is a plugin that changes how Claude Code looks and behaves. It's made of JavaScript or TypeScript event handlers: Claude Code calls one when an event happens, such as a tool call, a submitted prompt, or a part of the interface being drawn, and the handler can watch the event, change it, or take it over.
```

mod 是一个**插件**，由 **JS/TS 事件处理器**组成。事件发生（工具调用、提交 prompt、界面绘制）时，Claude Code 调起对应的 handler，handler 可以观察、改写、或者直接接管这件事

关键区分点在于 handler 跑在哪。settings 里的 hooks 也是事件触发，但那是**进程外**的 shell 命令 / HTTP 请求；mod 的 handler 是**进程内**函数。文档把两种都叫 "hook"——"hook" 指 mod 的 handler，settings 里那种叫 "settings hook"

![mod 的定义](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/d68d31b2509fc4f68af42e22e87bccfe.jpg)

横向对比

| | Mod | Settings hook | Skill | MCP server |
| --- | --- | --- | --- | --- |
| 本质 | 插件里、跑在进程内的函数 | 事件上跑的外部脚本/请求 | 一份 `SKILL.md` | 外部进程，提供工具 |
| 能改什么 | 工具调用、prompt、命令、回合、界面 | 单次工具调用/prompt 的放行、参数、结果 | Claude 知道什么 | Claude 有哪些工具 |
| 能画界面 | 能 | 不能 | 不能 | 不能 |
| 写什么 | JS / TS | 脚本 + settings 条目 | Markdown | 任意语言的服务器 |

## **二、文件怎么组织**

```text
first-mod/
├── .claude-plugin/
│   └── plugin.json        # 插件清单
└── hooks/
    ├── hooks.json         # 指向代码文件
    └── register.js        # hooks 模块：哪个函数跑在哪个事件上
```

最小模块就是导出一个 `register(on)`，往里注册事件处理。官方示例：

```javascript
let calls = 0

export function register(on) {
  on('tool.call', async ($, e, next) => {
    calls += 1
    $.ui.invalidate('ui.render')   // 请求重绘
    return next(e)                // 放行工具
  })

  on('ui.render', { component: 'Spinner' }, async ($, e, next) => {
    return next({ ...e, props: { ...e.props, suffix: ' · tool calls: ' + calls + '…' } })
  })
}
```

`register` 里逐个 `on(事件, handler)`。handler 收到 `next(e)`：调它放行并继续默认行为，改写 `e` 再往下传就是改写，不调而直接返回就是接管

## **三、mod 能做什么**

1. **画**：transcript 旁的 pane、prompt 上方的 band，带 tab、按钮、输入框
2. **改**：接管 Claude Code 自己画的东西——工具调用行、spinner、提问对话框
3. **拦或答**：在 `tool.call` 上按住一次调用、问用户、或者不跑工具直接给结果
4. **加命令**：`/command` 直接调用函数，立刻返回，不消耗模型回合

渲染点不是全端一致的：

| 渲染点 | 位置 | 哪些端会画 |
| --- | --- | --- |
| `Spinner` | 工作中的 spinner 行 | 终端、Desktop、VS Code |
| `Pane` | transcript 旁或 prompt 上方 | 终端、Desktop、VS Code |
| `AbovePrompt` | prompt 上方的 band | 终端、Desktop |
| `CommandOutput` | 斜杠命令输出 | 终端、Desktop |
| `ToolProgress` | 工具进度 | 仅终端 |

下面三张是一次真实会话，画面正好落在 mod 的绘制点上

![spinner 行与 prompt 上方的提示](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/33c5ac2e869e601c70acad2555aca94f.jpg)
*spinner 行与 prompt 上方的提示*

![spinner 行带实时 token 计数](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/c7ea292dc2b3d4e4fc7f004dddb24b68.jpg)
*spinner 行带实时 token 计数*

![右侧 diff 侧栏与底部状态行](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/8b6426b8beaee99a6f33b463a9494d22.jpg)
*右侧 diff 侧栏与底部状态行*

## **四、demo-mod 实测**

主线案例放在 `~/.claude/dev-mods/<session-id>/demo-mod/`，是"让 Claude 写一个 mod"流程产出的目录，session 级。清单：

```json
{
  "name": "demo-mod",
  "version": "0.1.0",
  "description": "统计 Claude 的工具调用次数，显示在 spinner 旁，提供 /tally 与 /tally-reset 命令",
  "author": { "name": "yunhao" }
}
```

`register.tsx` 干了四件事（节选）：

```tsx
export const register: Register = on => {
  // 会话开始时注册三个命令，并尝试自动打开面板
  on('session.start', async ($, e, next) => {
    await $.command.register({ name: 'tally', description: 'Show how many tool calls Claude has made' })
    await $.command.register({ name: 'tally-reset', description: 'Reset the tool call counter' })
    await $.command.register({ name: 'panel', description: 'Open the tool call stats pane' })
    void $.ui.open({ id: PANE, title: 'Tool call stats' })
    return next(e)
  })

  // 每次工具调用：计数 +1、追加记录、放行
  on('tool.call', async ($, e, next) => {
    calls += 1
    history.push({ id: e.tool_use_id, tool: e.tool, done: false })
    $.ui.invalidate('ui.render')
    const ran = await next(e)
    history = history.map(one => (one.id === e.tool_use_id ? { ...one, done: true } : one))
    $.ui.invalidate('ui.render')
    return ran
  })

  // spinner 旁显示计数
  on('ui.render', { component: 'Spinner' }, async ($, e, next) => {
    return next({ ...e, props: { ...e.props, suffix: ' · tool calls: ' + calls + '…' } })
  })

  // 在 Pane 里画表格
  on('ui.render', { component: 'Pane', requestId: PANE }, async ($, e) => {
    const { Box, Text } = $.ui.resolve(e)
    const room = Math.max(1, (e.viewport?.rows ?? 24) - 6)
    return (
      <Box flexDirection="column">
        <Text bold>Total tool calls: {calls}</Text>
        {history.slice(-room).map(one => <Text dimColor={one.done}>{one.done ? 'done ' : 'runs '} {one.tool}</Text>)}
      </Box>
    )
  })
}
```

安装前能看到它响应哪些事件、调用哪些 API。`claude plugin validate .` 先列出来：

```text
Validating plugin manifest: /Users/yunhao/.claude/dev-mods/8ffeea53-209f-4435-9396-8b3d772bc17c/demo-mod/.claude-plugin/plugin.json

Validating hooks: /Users/yunhao/.claude/dev-mods/8ffeea53-209f-4435-9396-8b3d772bc17c/demo-mod/hooks/hooks.json

  ❯ ./register.tsx hooks: session.start, command.run{command=panel}, tool.call, command.run{command=tally}, command.run{command=tally-reset}, ui.render{component=Spinner}, ui.render{component=Pane, requestId=tools-stats}
  ❯ ./register.tsx answers its own command: command.run{command=panel}
  ❯ ./register.tsx answers its own command: command.run{command=tally}
  ❯ ./register.tsx answers its own command: command.run{command=tally-reset}
  ❯ ./register.tsx gating hook without .catch: tool.call
  ❯ ./register.tsx calls: $.command.register, $.ui.invalidate, $.ui.open, $.ui.resolve

✔ Validation passed
```

`claude plugin test .` 不用会话、不用登录、不联网：

```text
tests/demo-mod.test.ts:
(pass) /tally reports the tool calls the mod has seen [40.32ms]
(pass) /tally-reset zeroes the counter [8.95ms]
(pass) /panel opens the stats pane [10.61ms]

 3 pass
 0 fail
Ran 3 tests across 1 file. [0.15s]
```

测试用 `on('tool.call', () => ({ result: 'ok' }))` 代替 Claude Code 回答工具调用，`Bash`、`Read` 不会真执行——这是它能脱网的原因

插件目录里还有引擎写出来的类型声明 `.claude-plugin/types/claude-code/index.d.ts`，首行 `// Written by Claude Code 2.1.289.`，15197 行。里面列出四个 surface：`terminal`、`desktop`、`vscode`、`mobile`。测试可以在任意 surface 上 `$.ui.mount` 后断言：

```ts
for (const surface of ['terminal', 'desktop', 'vscode', 'mobile'] as const) {
  const ui = await $.ui.mount({ plugin: 'notes', surface, ...HINT })
  expect(await ui.find({ type: 'Text', text: /notes/ })).toBeDefined()
}
```

## **五、跑在哪：不是 TUI 独占**

| 在哪跑 | hooks 跑不跑 | 画的东西出不出现 |
| --- | --- | --- |
| 终端里的 `claude`（含编辑器内置终端、JetBrains 插件） | 跑 | 出现 |
| Desktop App 的 Code 标签页（WSL 会话除外） | 跑 | 出现，terminal-only 元素除外 |
| Desktop 里的 WSL 会话 | 不跑（WSL 不支持插件） | 不出现 |
| VS Code 扩展的 chat 面板 | 跑 | 出现，扩展 ≥ 2.1.299（不拉 feature flag 的会话要 2.1.304） |
| `claude -p` 与 Agent SDK | 跑 | 不出现 |

版本门槛：终端 ≥ **2.1.287**，Desktop 自带的 Claude Code ≥ **2.1.286**。开关：单个 mod 在 `/plugin` 里禁用；整场用 `--safe-mode`；全禁用用 `disableAllHooks`。内置 mod 不受这些影响

本机核对：

| 位置 | 实测 | 能不能画 |
| --- | --- | --- |
| 终端 | `claude --version` → `2.1.292 (Claude Code)` | 能 |
| VS Code 扩展 | `~/.vscode/extensions/anthropic.claude-code-2.1.277-darwin-arm64` | 不能（低于 2.1.299） |
| Claude Desktop | 未安装 | — |

结论：官方支持三端绘制，这台机器只有终端够门槛

## **六、第三方与内置 mod**

第三方 mod 不止能做仪表盘。服务器上装的 `scasella/claude-flightdeck` 0.4.0（`~/.claude/plugins/cache/claude-flightdeck/flightdeck/0.4.0/`），右侧一整块都是它画的：模型状态（effort / mode / ctx / 花费 / ttft）、权限检查（allowed/denied 计数）、上一轮摘要、session log

![服务器上的 flightdeck：右侧四块 pane 全部由 mod 绘制](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/31cbd6db95bb91ef43ec71a446085657.jpg)
*服务器上的 flightdeck：右侧四块 pane 全部由 mod 绘制*

仓库：https://github.com/scasella/claude-flightdeck

日常功能本身就是 mod，`/plugin` 的 Installed 页 Built-in 分组下能看到：

| 名字 | 干什么 | 怎么关 |
| --- | --- | --- |
| `cc-plugin-agents-md` | 把 `AGENTS.md` 当项目指令加载 | `/plugin` 里禁用 |
| `cc-plugin-diff` | 接管 `/diff`，画 diff 面板 | 禁用后 `/diff` 回落到内置版 |
| `cc-plugin-plugin-authoring` | 给 Claude 一个写 mod 的 skill | `/plugin` 里禁用 |
| `cc-plugin-sec-default` | 组织策略的护栏 | 不能关，管理员在 managed settings 里定顺序 |
| `cc-plugin-telemetry` | 上报分析记录 | 禁用或关分析 |
| `cc-plugin-you-should-know` | prompt 上方的提示条 | 默认关，需自己开 |

`diff`、`agents-md`、`sec-default`、`telemetry` 源码公开：https://github.com/anthropics/claude-code/tree/main/mods

官方还有三个示例 mod：`token-weather`（预测上下文余量）、`blast-radius`（拦危险命令）、`replay-theater`（重放上一轮改动），在 https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods

## **七、安全模型**

mod 是跑在你权限下、进程内的代码。官方的清单：

- 读写你能读写的文件、起进程、发网络请求
- 读到你的密钥（环境变量、settings 里的 key）
- 看到整场会话：每条 prompt、每次工具调用
- 改写 prompt 或工具调用，甚至代你批准一次调用
- 花你的额度调模型
- 不受沙箱约束：sandbox 只隔离 Claude 跑的 Bash，mod 起的进程在沙箱外

所以流程是装之前先审：`claude plugin validate <dir>`，不跑代码就能看到 hooks 和 calls

## **八、和 DeepSeek Harness 的对照**

官方在 v0.2.1-alpha.1 的 release notes 里加了一条：新增实验性 Claude Code Mods 兼容层。目的写得直白——目前阶段是验证 Claude Code Mods 的 API 能力大致是 DeepSeek Harness 插件的一个子集，不是给用户提供完整兼容性

![DSH v0.2.1-alpha.1：新增实验性 Claude Code Mods 兼容层](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/9e931266bb9a2c71957f2d5b38a76454.jpg)

Claude Code：mods 是插件的子集，建在它自己的 hooks/skill/MCP 体系上，先解决界面能不能改。

DeepSeek Harness：模型、工具、技能、会话、沙箱、文件系统、循环、编排、UI 全部是插件，可以混搭替换。崔添翼的回应：

> 做这个兼容层的主要目的，是验证 Claude Code Mods 向插件作者提供的扩展能力，是否大致是 DSH"一切皆插件"架构所提供能力的一个子集。

![DSH 与 Claude Code Mods 的对照](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/937a5809869d0b3898cddf5ccf812967.jpg)

Harness 是 DSH 的设置层，管本地能力声明：skills、MCP server、hooks、AGENTS.md。MCP 与 hook 桥可以启停，skill 可以单独停用，AGENTS.md 直接编辑，本机 31 个 skill 全在这一个面板里

![Harness 设置：统一管理 skills、MCP、hooks 与 AGENTS.md](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/d92fb729df7ad27ce63d6bb489f3a799.jpg)

token use 是 DSH 的用量统计，把 token、工具调用、会话记进同一本账：按天 token 趋势、缓存命中率、活跃热力图、按模型拆分。主 agent 和子 agent 的消耗可以直接对比，不用去翻会话日志

![用量统计：token、工具调用、会话与主/子 agent 对比](https://public-files-dev.oss-cn-hangzhou.aliyuncs.com/images/8d7bdc979aa31f7ae16978c4847cb988.jpg)

万物皆插件的场景下可定制面更大：Claude Code 的 mod 只能在留出的渲染点上改，DSH 里连模型、循环、沙箱都是可替换的插件。代价直白——可玩性越高，审代码成本越高，mod 又完全不沙箱

这种可插拔、用户自定义的方式，似乎在开始变成一种主流，它在定义一种新的玩法与规则

## 参考文档

- [Mods overview](https://code.claude.com/docs/en/plugins/mods/overview.md)
- [Mods reference（渲染点、元素、限制）](https://code.claude.com/docs/en/plugins/mods/reference.md)
- [Create a mod](https://code.claude.com/docs/en/plugins/mods/create.md)
- [Test a mod](https://code.claude.com/docs/en/plugins/mods/test.md)
- [Draw in the interface](https://code.claude.com/docs/en/plugins/mods/interface.md)
- [内置 mods 源码](https://github.com/anthropics/claude-code/tree/main/mods)
- [示例 mods](https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods)
- [flightdeck](https://github.com/scasella/claude-flightdeck)