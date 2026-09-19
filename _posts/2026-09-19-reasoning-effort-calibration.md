---
layout: post
title: "reasoning_effort 的分裂与统一"
date: 2026-09-19
categories: ai llm
tags: reasoning llm api
---

`reasoning_effort` 看起来是一个参数，实际是五套互不兼容的白名单。同一句 `"reasoning_effort": "max"`，在百炼的 qwen3.8 上会被映射成 `xhigh`，在 deepseek-v4-pro 上原样生效，在 glm-5.1 上直接报错，在 deepseek-v4.1-flash 上连参数形态都不匹配——取决于你调用的是哪个模型

我把百炼 2026-09 版的文档逐模型对了一遍，档位、默认值、越界映射、关思考的写法整理成下面几张表，顺便把各家怎么把"思考"从一个开关拧成一个旋钮的时间线也理了出来

## **一、reasoning_effort 是什么**

### **1. 一个长得统一、语义分裂的参数**

```json
{
  "model": "qwen3.8-max",
  "messages": [{ "role": "user", "content": "..." }],
  "reasoning_effort": "max"
}
```

`reasoning_effort` 是推理模型 API 上控制思考深度的档位参数。各家文档里它长得一模一样，语义却不通用：**取值是一张按模型划定的白名单，不是连续区间**。传了白名单外的值，平台有三种反应——原样透传、映射到最近的档、直接返回 `invalid_parameter_error`。走哪种，看你调的是哪个模型，也看这个模型是厂商直供还是云上托管

### **2. 主线案例：一个 `max` 的五种下场**

全文用同一行参数贯穿：`"reasoning_effort": "max"`。它在百炼体系内会撞上五套互不兼容的处理逻辑

| 目标模型 | `max` 是否在白名单 | 实际结果 |
|---|---|---|
| qwen3.8-max / flash | 否（白名单为 `xhigh` `medium` `low`） | 越界映射，静默变成 `xhigh` |
| deepseek-v4-pro / flash | 是，顶档 | 原样生效 |
| ZHIPU/GLM-5.3 | 是，顶档且为默认值 | 原样生效 |
| glm-5.1 / glm-5 | 否，且不做兜底 | `invalid_parameter_error` |
| deepseek-v4.1-flash | 不适用 | 该模型 effort 是 1–100 整数，枚举形态本身不匹配 |

同一个字符串，四种结局：被改写、被接受、被拒绝、形态不符。文档没写错，是这个参数在每家的实现层级本来就不一样

### **3. 三层结构：白名单、越界处理、预算**

这个参数的行为分三层，从上往下逐层生效。混在一起看就会觉得文档自相矛盾，拆开看就顺了

1. **白名单层**。每个模型先声明自己收哪些值：qwen3.8-max 只收 `xhigh` `medium` `low`，glm-5.1 收 `none` 到 `xhigh` 六档，ZHIPU/GLM-5.3 只收 `max` `high` `low` 三档。任何一次调用都先过这一层，没有例外。`max` 在 deepseek-v4-pro 上能过、在 qwen3.8-max 上过不了，分岔就发生在这里
2. **越界处理层**。值不在白名单里怎么办，各家答案不同。百炼 GLM 页面自己写了一条警告：云上部署的三方开源模型和模型官方对超参数的处理逻辑不一样，**官方做阈值校验、越界回退默认值，云上直接透传不校验**。所以同一个平台里，两类模型的容错方向是反的。`max` 传给 qwen3.8-max 被改写成 `xhigh`，传给 glm-5.1 直接报错，`glm-5.2` 是云上托管、基本等于裸透传
3. **预算层**。effort 底下还压着一层硬预算：Qwen 是 `thinking_budget`（思考 token 上限），Claude 是 `task_budget`（长程 agent 的硬上限，模型能看见剩余预算倒计时，会自己节流）。要精确控成本、或者跑多轮 agent 才会碰到这层。这里还有个坑：**qwen3.8 系列不允许 `reasoning_effort` 和 `thinking_budget` 同时出现**，同传就报错

三层的关系还在动：上层做粗调旋钮，下层做硬约束，中间怎么衔接，每家自己定

## **二、百炼档位对照表**

| 模型 | 默认档 | 可选档 | 越界映射 | 关思考方式 |
|---|---|---|---|---|
| qwen3.8-max / qwen3.8-flash | `xhigh` | `xhigh` `medium` `low` | `max`、`high` → `xhigh`；`minimal` → `low`；`none` → 关闭思考 | `enable_thinking=false` 或 `effort=none` |
| qwen3.8-omni-flash | 思考默认开启 | `none` `minimal` `low` `medium` `high` `xhigh` `max` | 文档未列映射 | 同上 |
| deepseek-v4-pro / deepseek-v4-flash | `high` | `high` `max` | `low`、`medium` → `high`；`xhigh` → `max` | `enable_thinking=false` |
| deepseek-v4-flash-0731 / deepseek-v4-pro-0813 | `high` | `low` `high` `max` | `medium` → `high`；`xhigh` → `high` | `enable_thinking=false` |
| deepseek-v4.1-flash | — | **1–100 整数** | 不走枚举体系 | — |
| glm-5.2 / glm-5.2-us / glm-5.2-fast-preview | 思考默认开启 | `none` `minimal` `low` `medium` `high` `xhigh` `max` | `low`、`medium` → `high`；`xhigh` → `max`；`none` → `reasoning_tokens = 0` | `enable_thinking=false`（优先级高于 effort） |
| glm-5.1 / glm-5 | 思考默认开启 | `none` `minimal` `low` `medium` `high` `xhigh` | 同上，但不支持 `max` | `enable_thinking=false` |
| ZHIPU/GLM-5.3、ZHIPU/GLM-5.3-Flash | `max` | `max` `high` `low` | 其余取值报错 | **不可关**，`enable_thinking=false` 直接失败 |
| kimi-k3（阿里云直供） | `max` | `max` `high` `low` | — | — |
| kimi/kimi-k3（月之暗面直供） | — | 仅 `max` | — | — |

表里有三件事值得单独拎出来。**档位是白名单不是区间**，传错就是错，没有中间态。**默认档普遍偏高**，Qwen3.8 默认 `xhigh`、GLM-5.3 默认 `max`、DeepSeek 默认 `high`，不显式传就按贵的那档计费。**同名不同义**，`high` 在 Qwen3.8 上等于 `xhigh`，在 DeepSeek 快照版上是真高档，在 GLM-5.2 上是三档合一的汇聚点

另外两条踩过的：

- **qwen3.8 系列的 effort 与预算互斥**，两者可以互转：`low` = 4096、`medium` = 16384、`xhigh` = 262144；都不传时默认 `thinking_budget` 131072，也就是 `xhigh`
- **GLM 系列有个 `clear_thinking` 参数**，控制多轮对话里历史 `reasoning_content` 要不要回灌上下文。默认 `false`（保留，即 Preserved Thinking），设 `true` 能明显压低 `prompt_tokens`。把"历史思考留不留"显式开放出来的，目前就这一家

## **三、硬编码 vs 按模型查表**

处理这种分歧只有两条路，差别在错误什么时候爆

| 维度 | 硬编码档位名 | 按模型维护白名单 |
|---|---|---|
| 触发条件 | 代码里写死一个"通用高档" | 每次请求先查模型能力表 |
| 操作方式 | 所有模型共用一份 payload | 请求前做取值校验与改写 |
| 失败表现 | 线上报错或被静默映射，账单和延迟对不上预期 | 请求前拦截，错误暴露在本地 |
| 适用边界 | 单模型、档位稳定的场景 | 多模型路由、跨厂商迁移、benchmark |

硬编码 `max` 的服务切到 glm-5.1，线上直接 400，这个好查。切到 qwen3.8-max 不报错，实际跑的是 `xhigh`，请求还是成功的，成本和延迟却对不上预期——这种才难查

## **四、各家"关思考"的写法**

关思考是最容易踩坑的操作：每家字段不一样，还有厂商禁止"高档位 + 关思考"这种组合

| 厂商 | 关思考字段 | 高档位下能否关 | 备注 |
|---|---|---|---|
| OpenAI | `reasoning_effort = "none"` | 可以 | GPT-5.1 默认值就是 `none`；GPT-5.6 在 Chat Completions 下带 `tools` 时必须显式设为 `none`，否则默认 `medium` 直接报错 |
| Claude | `thinking.type = "disabled"` | **受 effort 限制** | Opus 5 仅在 effort ≤ `high` 时允许关闭；Fable 5 完全不可关。4.7 起手动 extended thinking 已移除，传 `budget_tokens` 直接 400 |
| 智谱 GLM | 5.2：`thinking.type = "disabled"`；5.3：不可关 | 5.2 可关，5.3 强制 | 从 5.2 迁到 5.3 若沿用 `disabled`，请求直接失败 |
| 百炼 DeepSeek | `enable_thinking = false` | 可以 | 思考内容通过 `reasoning_content` 返回 |
| 百炼 Qwen | `enable_thinking = false` 或 `effort = "none"` | 可以 | qwen3.8 下 `effort` 与 `thinking_budget` 互斥 |
| 火山引擎豆包 | `thinking: "disabled"` | 可以，但不能叠加 effort | 已设 `disabled` 后再传 `low`～`max` 会返回 400；只有支持 `none` 的模型才可显式设 `none` |

方向是清楚的：**"关思考"这个布尔量正在被吸收进 effort 阶梯**。OpenAI 把 `none` 做成九档里最低的一档，Qwen 把 `none` 定义成 `enable_thinking=false` 的等价物。独立开关在消失，只剩 GLM-5.2 这种过渡期模型还留着两条路

## **五、时间线：从开关到旋钮**

### **第一阶段：没有旋钮（2024-09 起）**

o1 系列只有"是不是推理模型"的区别，`o1-mini` 甚至不支持 `reasoning_effort`。想控制思考深度，只能改提示词

### **第二阶段：三档旋钮（2024-12 ～ 2025）**

o1 / o3 系列引入 `reasoning_effort`，取值 `low` `medium` `high`。行业第一次把"思考多久"当成 API 参数暴露出来

### **第三阶段：向低端延伸（2025-08，GPT-5）**

加入 `minimal`，"几乎不思考但保留一点"终于可表达。但想彻底关掉推理，还是只能换模型

### **第四阶段：并入 `none`（2025-11，GPT-5.1）**

新增 `none`，**并且把 `none` 设为 GPT-5.1 的默认值**。从早期推理模型升级上来的代码，不显式传档位就会静默地从"思考"变成"不思考"。同类变更里破坏性最强的一次，也是独立开关被吸收进阶梯的起点

同期 GPT-5.1-Codex-Max 加入 `xhigh`，把阶梯往高端拉了一格，给延迟不敏感的长程智能体任务用

### **第五阶段：七档成型与分层控制（2026）**

`none` `minimal` `low` `medium` `high` `xhigh` `max` 成为事实标准。同时走出三条不同的深化路径

**OpenAI 加正交维度。** effort 之外增加 `reasoning.mode`（standard / pro）和 `reasoning.context`（`current_turn` / `all_turns` / `auto`），把"想多少"和"想多久的历史要不要留"拆成两个独立开关。`all_turns` 会明显增加计费 token，升模型时代码不动也要重估成本

**Claude 换掉底层机制。** extended thinking（固定 `budget_tokens`）→ adaptive thinking（4.6 起，模型自己决定何时想）→ 4.7 起手动 extended thinking 彻底移除。`effort` 从可选旋钮变成唯一控制手段，`budget_tokens` 传了就是 400。再给长程 agent 补上 `task_budget` 硬上限，模型能看见剩余预算倒计时并自我节流

**DeepSeek 把它做进训练。** V4.1 后训练阶段把一个 1–100 的标量 effort 显式写进系统提示，同一 effort 下的多次采样构成子组做奖励中心化，长度惩罚系数随 effort 指数衰减（每升高一个特征尺度，惩罚乘以 1/e）。API 暴露的 `max` / `high` / `low` 对应标量上的 100 / 75 / 50。官方数据：effort 从 25 提到 100，八个推理密集型基准的平均 Pass@1 从 67.1% 升到 76.3%，代价是约 2.5 倍输出 token

**国产阵营口径收敛而非统一。** GLM 从 5.2 的"可关 + 7 档"走到 5.3 的"强制思考 + 3 档"；Qwen 用 `thinking_budget` ↔ `effort` 双向映射做兼容层；百炼作为聚合平台，给同一个 `reasoning_effort` 字段按模型各配了一张映射表，越界即报错

走到这一步，`max` 的五种下场就有了解释：白名单是各家长出来的，不是谁设计出来的

## **六、一次按模型查表的档位下发**

把第二节的表落成本地一张白名单，请求发出去之前先查一次

#### 输入

```json
{ "model": "glm-5.1", "reasoning_effort": "max" }
```

#### 中间产物（本地白名单命中结果）

```json
{
  "glm-5.1": {
    "allowed": ["none", "minimal", "low", "medium", "high", "xhigh"],
    "on_out_of_range": "reject",
    "thinking_off": "enable_thinking=false",
    "notes": "不支持 max"
  }
}
```

#### 最终输出

```text
max ∉ allowed，on_out_of_range = reject
→ 本地拦截，不发上游请求
→ 提示改用 xhigh（glm-5.1 的顶档）
```

同一份白名单跑另外两个输入：

```text
{ "model": "qwen3.8-max", "reasoning_effort": "max" }
→ allowed = [xhigh, medium, low]，on_out_of_range = map
→ 实际下发 xhigh，日志记录一次静默改写

{ "model": "qwen3.8-max", "reasoning_effort": "low", "thinking_budget": 4096 }
→ 命中互斥规则
→ 本地拦截：qwen3.8 系列二者不可同传，保留其中一个
```

代码量不值钱，值钱的是**错误爆出来的位置从线上账单挪到了本地**。静默映射这种不报错的分歧，只有查表能提前看见

## **七、工程建议**

1. **别硬编码档位名。** 按模型维护白名单，或者从模型元数据读能力声明。`xhigh` 在 Qwen3.8 上合法、在 GLM-5.1 上越界、在 Claude Fable 5 上不存在
2. **先跑默认档，再谈调优。** 默认档就是厂商推荐档，但得知道默认是什么——Qwen3.8 的 `xhigh` 和 GLM-5.3 的 `max` 都是成本最高的档
3. **DeepSeek 的甜区不在 `max`。** 官方报告里 effort 60–80 就能用不到一半的 token 预算拿到接近满档的准确率，从 80 走到 100 只是让 agent 轨迹再长 1.6–1.8 倍，换一点边际提升
4. **档位是成本主开关，不只是质量开关。** 思考 token 全部计入输出计费、按输出价结算。GLM-5.3 输出价是输入价的 3 倍以上，默认 `max` 在高频调用下会显著放量
5. **跨厂商 benchmark 必须对齐档位。** 拿 DeepSeek 的 `high` 去对别家的 `max` 是错的，反过来拿别家的 `max` 对 DeepSeek 的 `high` 同样错。先说清楚比的是同档位还是同预算
6. **用固定真实任务集做回归。** 官方跑分基本是满档跑出来的，迁移时别直接套

## 参考文档

- [百炼：深度思考模型的用法](https://help.aliyun.com/zh/model-studio/deep-thinking) — GLM、Qwen3.8、DeepSeek、kimi-k3 各模型页的档位与越界映射
- [百炼：文本生成模型 API 参考](https://help.aliyun.com/zh/model-studio/qwen-api-reference/) — `reasoning_effort` / `thinking_budget` / `enable_thinking` / `clear_thinking` 字段说明
- [OpenAI：Reasoning models](https://developers.openai.com/api/docs/guides/reasoning) — effort 取值与 `none` 语义
- [Azure OpenAI：推理模型](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/reasoning) — 可取值列表与场景对照表
- [Claude：Effort](https://platform.claude.com/docs/en/build-with-claude/effort)、[Claude：Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking) — adaptive thinking 与 `task_budget`
- [DeepSeek：思考模式](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode/) — `reasoning_effort` 三档定义
- [智谱：深度思考](https://docs.bigmodel.cn/cn/guide/capabilities/thinking)、[智谱：GLM-5.3 模型页](https://docs.bigmodel.cn/cn/guide/models/text/glm-5.3) — 5.2 可关、5.3 强制思考
- [火山方舟：深度思考](https://www.volcengine.com/docs/82379/1956279) — `thinking` 字段与 effort 叠加限制
- DeepSeek V4.1 官方技术报告 — effort 标量、长度惩罚衰减、Pass@1 与 token 代价数据，无公开稳定 URL，未附链接
