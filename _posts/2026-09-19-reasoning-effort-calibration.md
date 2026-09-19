---
layout: post
title: "百炼推理档位校勘：同一参数的五种口径"
date: 2026-09-19
categories: ai llm
tags: reasoning llm api
---

`reasoning_effort` 看起来是一个参数，实际是五套互不兼容的白名单。同一句 `"reasoning_effort": "max"`，在百炼的 qwen3.8 上会被映射成 `xhigh`，在 deepseek-v4-pro 上原样生效，在 glm-5.1 上直接报错，在 deepseek-v4.1-flash 上连参数形态都不匹配——取决于你调用的是哪个模型

## **一、reasoning_effort 介绍**

### **1. 简介**

```json
{
  "model": "qwen3.8-max",
  "messages": [{ "role": "user", "content": "..." }],
  "reasoning_effort": "max"
}
```

`reasoning_effort` 是推理模型 API 上控制"思考多深"的档位参数。它在各家文档里长得一模一样，语义却不通用：**取值是一张按模型划定的白名单，不是一个连续区间**。传入白名单外的值，平台有三种处理方式——原样透传、映射到最近档位、直接返回 `invalid_parameter_error`。走哪一种，取决于你调用的是哪个模型，也取决于这个模型是厂商直供还是云上托管

本文以百炼官方文档（2026-09 版）为准，逐模型校勘档位、默认值与越界映射，给出各家把"思考"从开关做成旋钮的演进时间线，文末附对初稿的 6 处勘误

### **2. 主线案例：一个 `max` 的五种下场**

全文用同一行参数贯穿：`"reasoning_effort": "max"`。它在百炼体系内会撞上五套互不兼容的处理逻辑

| 目标模型 | `max` 是否在白名单 | 实际结果 |
|---|---|---|
| qwen3.8-max / flash | 否（白名单为 `xhigh` `medium` `low`） | 越界映射，静默变成 `xhigh` |
| deepseek-v4-pro / flash | 是，顶档 | 原样生效 |
| ZHIPU/GLM-5.3 | 是，顶档且为默认值 | 原样生效 |
| glm-5.1 / glm-5 | 否，且不做兜底 | `invalid_parameter_error` |
| deepseek-v4.1-flash | 不适用 | 该模型 effort 是 1–100 整数，枚举形态本身不匹配 |

同一个字符串，得到"被改写""被接受""被拒绝""形态不符"四种结局。这不是文档写得乱，而是这个参数在每家厂商的实现层级不同

### **3. 特性 – 档位语义的三层结构**

`reasoning_effort` 的行为由三层决定，从上到下逐层生效，每一层都有自己的触发条件

1. **枚举白名单层** – *第一层*：每个模型声明自己接受哪些取值，例如 qwen3.8-max 只有 `xhigh` `medium` `low`，glm-5.1 有 `none` 到 `xhigh` 六档，ZHIPU/GLM-5.3 只有 `max` `high` `low` 三档。触发条件：任何一次调用都先过这一层，无例外。主线案例中 `max` 在 deepseek-v4-pro 上通过、在 qwen3.8-max 上不通过，分岔就发生在这一层
2. **越界处理层** – *第二层*：取值不在白名单时怎么办。百炼 GLM 页面自带一条警告——云上部署的三方开源模型与模型官方对超参数的处理逻辑不同，**官方做阈值校验、越界回退默认值；云上是直接透传，不做校验**。所以同一平台内两类模型的容错行为是反的。触发条件：仅当取值越界。主线案例中 `max` 传给 qwen3.8-max 被映射成 `xhigh`（有兜底），传给 glm-5.1 直接报错（无兜底），而 `glm-5.2` 属于云上托管、更接近原始透传
3. **预算层** – *第三层*：effort 之下还有一层硬预算，Qwen 是 `thinking_budget`（输出思考 token 上限），Claude 是 `task_budget`（长程 agent 硬上限，模型能看到剩余预算倒计时并自我节流）。触发条件：需要精确控成本或跑多轮 agent。主线案例在这一层会遇到另一个坑——**qwen3.8 系列不允许 `reasoning_effort` 与 `thinking_budget` 同时出现**，同传即报错

这三层的关系正在变化：上层 effort 做粗调旋钮，下层预算做硬约束，但两者怎么衔接，每家都不一样

## **二、百炼档位对照表（校勘后）**

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

从表里能直接读出三个结论：**档位是白名单不是区间**、**默认档普遍偏高**（Qwen3.8 默认 `xhigh`、GLM-5.3 默认 `max`、DeepSeek 默认 `high`，不显式指定就按最贵那档计费）、**同名不同义**（`high` 在 Qwen3.8 上等价 `xhigh`，在 DeepSeek 快照版上是真实高档，在 GLM-5.2 上是"三档合一"的汇聚点）

另有两条容易踩的细节：

- **qwen3.8 系列的 effort 与预算互斥**，且两者可互转：`low` = 4096、`medium` = 16384、`xhigh` = 262144；都不传时默认 `thinking_budget` 131072，即 `xhigh`
- **GLM 系列有 `clear_thinking` 参数**，控制多轮对话中历史 `reasoning_content` 是否回灌上下文。默认 `false`（保留，即 Preserved Thinking），设 `true` 可显著压低 `prompt_tokens`。这是目前少见的、把"历史思考要不要留着"显式开放出来的设计

## **三、传统写法 vs 按模型查表**

处理档位分歧有两种工程做法，差别在失败发生的位置

| 维度 | 硬编码档位名 | 按模型维护白名单 |
|---|---|---|
| 触发条件 | 代码里写死一个"通用高档" | 每次请求先查模型能力表 |
| 操作方式 | 所有模型共用一份 payload | 请求前做取值校验与改写 |
| 失败表现 | 线上报错或被静默映射，账单和延迟对不上预期 | 请求前拦截，错误暴露在本地 |
| 适用边界 | 单模型、档位稳定的场景 | 多模型路由、跨厂商迁移、benchmark |

主线案例在两种做法下的表现：硬编码 `max` 的服务，切到 glm-5.1 时线上直接 400，切到 qwen3.8-max 时不报错但实际跑的是 `xhigh`——后者更难发现，因为请求成功了，成本和延迟却与预期不符

## **四、对初稿的 6 处勘误**

| # | 初稿说法 | 校勘结果 |
|---|---|---|
| 1 | `minimal` 的描述是"仅以略微增加的延迟实现高效推理" | 该描述在官方场景表中属于 **`low`**。Azure 文档的可取值列表含 `minimal`，但场景对照表只有 6 行，已不单列 `minimal` |
| 2 | Claude Fable 5.1 支持"同五档" | 实际是 **4 档**：`low` `medium` `high` `xhigh`，**不支持 `max`**。五档仅 Opus 5 / Sonnet 5 / Opus 4.8 / Opus 4.7 |
| 3 | glm-5.2 在百炼"支持 high/max，官方 7 档被压成 2 档" | 百炼存在**两套口径**：GLM 专属页明确列出 7 个可选取值；DashScope 通用 API 参考写 GLM 系列默认 `high`、可选 `high`/`max`，`low`/`medium` 映射为 `high`。更接近实况的表述是"可传 7 个值，有效档位被压缩" |
| 4 | qwen3.8 系列统一三档 | `max` / `flash` 成立，但 **qwen3.8-omni-flash 文档列 7 档**（`none` 到 `max`）。家族内部口径不一致 |
| 5 | 未提及 deepseek-v4.1-flash | 该模型支持 **1–100 的整数** effort，是百炼体系里唯一脱离枚举档位的模型 |
| 6 | OpenAI 档位为 7 档 | 可取值确为 7 个，但官方场景对照表只有 6 行，`minimal` 未列 |

第 3 条值得单独说，因为它解释了第二层"越界处理"为什么会分裂：`glm-5.2` 是云上托管的三方开源模型，走透传；`ZHIPU/GLM-5.3` 是智谱直供，走官方校验。同一个 `reasoning_effort` 字段，在同一个平台上，容错方向相反

## **五、各家"关思考"的写法对照**

关闭思考是最容易踩坑的操作，因为每家用的字段不同，且部分厂商禁止"高档位 + 关思考"的组合

| 厂商 | 关思考字段 | 高档位下能否关 | 备注 |
|---|---|---|---|
| OpenAI | `reasoning_effort = "none"` | 可以 | GPT-5.1 默认值就是 `none`；GPT-5.6 在 Chat Completions 下带 `tools` 时必须显式设为 `none`，否则默认 `medium` 直接报错 |
| Claude | `thinking.type = "disabled"` | **受 effort 限制** | Opus 5 仅在 effort ≤ `high` 时允许关闭；Fable 5 完全不可关。4.7 起手动 extended thinking 已移除，传 `budget_tokens` 直接 400 |
| 智谱 GLM | 5.2：`thinking.type = "disabled"`；5.3：不可关 | 5.2 可关，5.3 强制 | 从 5.2 迁到 5.3 若沿用 `disabled`，请求直接失败 |
| 百炼 DeepSeek | `enable_thinking = false` | 可以 | 思考内容通过 `reasoning_content` 返回 |
| 百炼 Qwen | `enable_thinking = false` 或 `effort = "none"` | 可以 | qwen3.8 下 `effort` 与 `thinking_budget` 互斥 |
| 火山引擎豆包 | `thinking: "disabled"` | 可以，但不能叠加 effort | 已设 `disabled` 后再传 `low`～`max` 会返回 400；只有支持 `none` 的模型才可显式设 `none` |

一个共同的演进方向：**"关思考"这个布尔量正在被吸收进 effort 阶梯**。OpenAI 把 `none` 做成了九档里的最低档，Qwen 把 `none` 定义成 `enable_thinking=false` 的等价物。独立开关正在消失，只剩 GLM-5.2 这类过渡期模型还同时保留两条路径

## **六、时间线：从开关到旋钮**

### **第一阶段：没有旋钮（2024-09 起）**

o1 系列只有"是不是推理模型"的区别，`o1-mini` 甚至不支持 `reasoning_effort`。当时控制思考深度的唯一办法是改提示词

### **第二阶段：三档旋钮（2024-12 ～ 2025）**

o1 / o3 系列引入 `reasoning_effort`，取值 `low` `medium` `high`。这是行业第一次把"思考多久"当成 API 参数暴露出来

### **第三阶段：向低端延伸（2025-08，GPT-5）**

加入 `minimal`，让"几乎不思考但保留一点"变得可表达。此时的问题是：想彻底关掉推理，仍然只能靠换模型

### **第四阶段：并入 `none`（2025-11，GPT-5.1）**

新增 `none`，**并且把 `none` 设为 GPT-5.1 的默认值**。从早期推理模型升级上来的代码，如果不显式传档位，会静默地从"思考"变成"不思考"。这是同类变更里破坏性最强的一次，也是"独立开关被吸收进阶梯"的起点

同期 GPT-5.1-Codex-Max 加入 `xhigh`，把阶梯往高端也拉了一格，用于延迟不敏感的长程智能体任务

### **第五阶段：七档成型与分层控制（2026）**

`none` `minimal` `low` `medium` `high` `xhigh` `max` 成为事实标准。同时出现了三种不同的深化路径

**OpenAI —— 加正交维度。** 在 effort 之外增加 `reasoning.mode`（standard / pro）和 `reasoning.context`（`current_turn` / `all_turns` / `auto`），把"想多少"和"想多久的历史要不要留"拆成两个独立开关。`all_turns` 会显著增加计费 token，升级到新模型时即使代码不变也要重估成本

**Claude —— 换掉底层机制。** extended thinking（固定 `budget_tokens`）→ adaptive thinking（4.6 起，模型自己决定何时想）→ 4.7 起手动 extended thinking 彻底移除。`effort` 从"可选旋钮"变成"唯一控制手段"，`budget_tokens` 传了就是 400。为长程 agent 补上 `task_budget` 硬上限，模型能看到剩余预算倒计时并自我节流

**DeepSeek —— 把它做进训练。** V4.1 的后训练阶段把一个 1–100 的标量 effort 显式写进系统提示，同一 effort 下的多次采样构成子组做奖励中心化，长度惩罚系数随 effort 指数衰减（每升高一个特征尺度，惩罚乘以 1/e）。API 暴露的 `max` / `high` / `low` 对应标量上的 100 / 75 / 50。官方数据：effort 从 25 提到 100，八个推理密集型基准的平均 Pass@1 从 67.1% 升到 76.3%，代价是约 2.5 倍输出 token

**国产阵营 —— 口径收敛而非统一。** GLM 从 5.2 的"可关 + 7 档"走到 5.3 的"强制思考 + 3 档"；Qwen 用 `thinking_budget` ↔ `effort` 双向映射做兼容层；百炼作为聚合平台，对同一个 `reasoning_effort` 字段给每个模型配了独立的映射表，越界即报错

走到这一阶段，主线案例里 `max` 的五种下场就有了来历：白名单是各家长出来的，不是设计出来的

## **七、完整示例：一次按模型查表的档位下发**

把第二节的对照表落成一张本地白名单，请求发出前先查一次

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

价值不在代码量，在于**错误发生的位置从线上账单前移到了本地**。静默映射这类不会报错的分歧，只有查表才能提前看见

## **八、工程建议**

1. **不要硬编码档位名。** 按模型维护白名单，或从模型元数据读能力声明。`xhigh` 在 Qwen3.8 上合法、在 GLM-5.1 上越界、在 Claude Fable 5 上不存在
2. **默认档就是厂商推荐档，先跑默认再调。** 但要知道默认是什么——Qwen3.8 的默认 `xhigh` 和 GLM-5.3 的默认 `max` 都是成本最高的档
3. **DeepSeek 的甜区不在 `max`。** 官方报告指出 effort 60–80 已能用不到一半的 token 预算拿到接近满档的准确率，从 80 走到 100 只会让 agent 轨迹再长 1.6–1.8 倍，换来边际提升
4. **档位是成本主开关，不只是质量开关。** 思考 token 全部计入输出计费并按输出价结算。GLM-5.3 输出价是输入价的 3 倍以上，默认 `max` 在高频调用下会显著放量
5. **跨厂商做 benchmark 时档位必须对齐。** 拿 DeepSeek 的 `high` 去对别家的 `max` 是错的，但反过来拿别家的 `max` 对 DeepSeek 的 `high` 同样错。先明确比的是"同档位"还是"同预算"
6. **用固定真实任务集做回归。** 官方跑分基本都是满档跑出来的，迁移时不要直接套用

## **九、一句话总结**

`reasoning_effort` 是厂商把算力-质量权衡外包给调用方的产物：参数名是统一的，语义是分裂的，选档位的责任和成本一起转移给了 API 使用者。工程上唯一可靠的做法是——按模型查表，别信直觉

## 参考文档

- 百炼官方文档（2026-09 版）：GLM 专属模型页、Qwen3.8 系列模型页、DeepSeek 系列模型页、kimi-k3 模型页
- DashScope 通用 API 参考：`reasoning_effort` / `thinking_budget` / `enable_thinking` / `clear_thinking` 字段说明
- Azure OpenAI 文档：`reasoning_effort` 可取值列表与场景对照表
- Anthropic 文档：extended thinking、adaptive thinking、`task_budget`
- DeepSeek V4.1 官方技术报告：effort 标量、长度惩罚衰减、Pass@1 与 token 代价数据
- 火山引擎豆包官方文档：`thinking` 字段与 effort 叠加限制
