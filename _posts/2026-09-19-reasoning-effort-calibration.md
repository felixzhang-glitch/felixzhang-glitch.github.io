---
layout: post
title: "reasoning_effort 的分裂与统一"
date: 2026-09-19
categories: ai llm
tags: reasoning llm api
---

`reasoning_effort`：描述推理强度的参数，但并不标准。同一个 key，不一样的 value，不尽相同的实现：有的匹配生效，有的映射转换，还有的直接报错

## 一、reasoning_effort 是什么

### 1. 取值是白名单，不是区间

```json
{
  "model": "qwen3.8-max",
  "messages": [{ "role": "user", "content": "..." }],
  "reasoning_effort": "max"
}
```

同一个 name，value 是各模型厂商自己的白名单。常见三种处理：透传、映射最近档、直接 `invalid_parameter_error`

### 2. 一个 `max` 的四种实现

| 目标模型 | `max` 是否在白名单 | 实际结果 |
| --- | --- | --- |
| qwen3.8-max / flash | 否（白名单为 `xhigh` `medium` `low`） | 越界映射，静默变 `xhigh` |
| deepseek-v4-pro / v4-flash / v4.1-flash | 是，顶档 | 匹配生效（v4.1-flash 另有 `low`） |
| glm-5.3 / ZHIPU/GLM-5.3 | 是，顶档 | 匹配生效 |
| glm-5.1 / glm-5 | 否，且不兜底 | `invalid_parameter_error` |

### 3. 三层，自上而下

1. **白名单：能传什么。** 比如 qwen3.8-max 只收 `xhigh` `medium` `low`；glm-5.1 收六档 `none`–`xhigh`；glm-5.3 / ZHIPU/GLM-5.3 只收 `max` `high` `low`
2. **越界：传错怎么办。** 官方阈值校验、越界回退默认；云上直接透传。但"云上=透传"只对采样超参成立，glm-5.3 对 effort 照校验，越界 `invalid_parameter_error`
3. **预算：思考上限。** Qwen `thinking_budget`，Claude `task_budget`。qwen3.8 二者互斥，同传报错

## 二、百炼档位对照表

| 模型 | 默认档 | 可选档 | 越界映射 | 关思考 |
| --- | --- | --- | --- | --- |
| qwen3.8-max / qwen3.8-flash | `xhigh` | `xhigh` `medium` `low` | `max`、`high` → `xhigh`；`minimal` → `low`；`none` → 关 | `enable_thinking=false` 或 `effort=none` |
| qwen3.8-omni-flash | 思考默认开启 | `none` `minimal` `low` `medium` `high` `xhigh` `max` | 文档未列 | 同上 |
| deepseek-v4-pro / deepseek-v4-flash | `high` | `high` `max` | `low`、`medium` → `high`；`xhigh` → `max` | `enable_thinking=false` |
| deepseek-v4-flash-0731 / deepseek-v4-pro-0813 | `high` | `low` `high` `max` | `medium` → `high`；`xhigh` → `high` | `enable_thinking=false` |
| deepseek-v4.1-flash | `high` | `low` `high` `max` | `medium` → `high`；`xhigh` → `max` | `enable_thinking=false` |
| glm-5.3 | 思考强制开启，默认档文档未标 | `low` `high` `max` | 其余 `invalid_parameter_error` | **不可关**，`enable_thinking=false` 静默无效 |
| glm-5.2 / glm-5.2-us / glm-5.2-fast-preview | 思考默认开启 | `none` `minimal` `low` `medium` `high` `xhigh` `max` | `low`、`medium` → `high`；`xhigh` → `max`；`none` → `reasoning_tokens=0` | `enable_thinking=false`（优先于 effort） |
| glm-5.1 / glm-5 | 思考默认开启 | `none` `minimal` `low` `medium` `high` `xhigh` | 同上，不支持 `max` | `enable_thinking=false` |
| ZHIPU/GLM-5.3 / ZHIPU/GLM-5.3-Flash | `max` | `max` `high` `low` | 其余报错 | **不可关**，传 `disabled`/`enable_thinking=false` 请求失败 |
| kimi-k3 | `max` | `max` `high` `low` | — | — |
| kimi/kimi-k3 | — | 仅 `max` | — | — |

- 档位是白名单不是区间，传错就是错
- 默认档普遍偏高，常见的default基本都是最高档
- 同名不同义：`high` 在 Qwen3.8 等于 `xhigh`，在 DeepSeek 快照是真高档，在 GLM-5.2 却只是三档汇聚点

三条坑：

- **qwen3.8 effort 与预算互斥**：`low`=4096、`medium`=16384、`xhigh`=262144；都不传默认 131072，即 `xhigh`
- **GLM `clear_thinking`** 控制历史 `reasoning_content` 是否回灌。多数默认 `false`（保留），glm-5.3 默认 `true`（不回灌）
- **deepseek-v4.1-flash**：API 侧正常枚举，`max` 合法顶档，默认 `high`。1–100 连续 effort 只在开源权重 prompt encoding 里，API 三档 = 50 / 75 / 100

## 三、硬编码 vs 查表

差别在错误什么时候爆

| 维度 | 硬编码档位名 | 按模型维护白名单 |
| --- | --- | --- |
| 触发条件 | 代码写死一个"通用高档" | 每次请求先查能力表 |
| 操作方式 | 共用一份 payload | 请求前校验改写 |
| 失败表现 | 线上报错或静默映射，账单对不上 | 请求前拦截，错误露在本地 |
| 适用边界 | 单模型、档位稳定 | 多模型路由、跨厂商、benchmark |

硬编码 `max` 切 glm-5.1，直接 400，好查。切 qwen3.8-max 不报错，实际跑 `xhigh`，请求成功，成本和延迟对不上——难查的是这种

## 四、关思考的写法

字段不统一，且厂商禁止"高档位 + 关思考"

| 厂商 | 关思考字段 | 高档位能否关 | 备注 |
| --- | --- | --- | --- |
| OpenAI | `reasoning_effort = "none"` | 可以 | GPT-5.1 默认 `none`；GPT-5.6 Chat Completions 带 `tools` 须显式 `none`，否则按默认 `medium` 报错 |
| Claude | `thinking.type = "disabled"` | **受 effort 限** | Opus 5 仅 effort ≤ `high` 可关；Fable 5 不可关。4.7 起传 `budget_tokens` 直接 400 |
| 智谱 GLM | 5.2：`thinking.type="disabled"`；5.3：不可关 | 5.2 可关，5.3 强制 | 迁 5.3 沿用 `disabled`：ZHIPU/GLM-5.3 请求失败；glm-5.3 静默无效，更难发现 |
| 百炼 DeepSeek | `enable_thinking = false` | 可以 | 思考经 `reasoning_content` 返回 |
| 百炼 Qwen | `enable_thinking = false` 或 `effort="none"` | 可以 | qwen3.8 下二者互斥 |
| 火山豆包 | `thinking: "disabled"` | 可以，不能叠 effort | 已 `disabled` 再传 `low`~`max` 返回 400 |

关思考正被吸进 effort 阶梯。独立开关在消失，只剩 GLM-5.2 留两条路

## 五、时间线

| 阶段 | 时间 | 变更 |
| --- | --- | --- |
| 没有旋钮 | 2024-09 | o1 系列无 `reasoning_effort`，控深度靠提示词 |
| 三档旋钮 | 2024-12～2025 | o1/o3 引入 `low` `medium` `high` |
| 向低端延伸 | 2025-08 | GPT-5 加 `minimal` |
| 并入 `none` | 2025-11 | GPT-5.1 加 `none` 并设为默认，不传档静默变"不思考" |
| 七档成型 | 2026 | `none`–`max` 成事实标准，三条路径 |

同期 GPT-5.1-Codex-Max 加 `xhigh`，阶梯往高拉一档

**OpenAI 加正交维度。** `reasoning.mode`（standard/pro）+ `reasoning.context`（`current_turn`/`all_turns`/`auto`）。`all_turns` 明显涨计费 token

**Claude 换底层。** extended thinking（固定 `budget_tokens`）→ adaptive thinking（4.6）→ 4.7 移除手动，传 `budget_tokens` 即 400。补 `task_budget` 硬上限，模型自节流

**DeepSeek 做进训练。** V4.1 后训练把 1–100 标量 effort 写进系统提示，同 effort 采样组内奖励中心化，长度惩罚系数随 effort 指数衰减。API `max`/`high`/`low` = 100/75/50。官方：effort 25→100，八基准 Pass@1 67.1%→76.3%，代价约 2.5 倍输出 token

**国产收敛不统一。** GLM 5.2"可关+7 档"→5.3"强制+3 档"；Qwen `thinking_budget`↔`effort` 互映射；百炼按模型各配一张映射表

白名单是各家长出来的

## 六、一次查表下发

```json
{ "model": "glm-5.1", "reasoning_effort": "max" }
```

命中本地白名单：

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

```
max ∉ allowed，on_out_of_range = reject
→ 本地拦截，不发上游
→ 提示改 xhigh（glm-5.1 顶档）
```

同一白名单跑另外两个输入：

```
{ "model": "qwen3.8-max", "reasoning_effort": "max" }
→ allowed = [xhigh, medium, low]，on_out_of_range = map
→ 下发 xhigh，日志记一次静默改写

{ "model": "qwen3.8-max", "reasoning_effort": "low", "thinking_budget": 4096 }
→ 命中互斥
→ 本地拦截，保留一个
```

值钱的是错误爆的位置从线上账单挪到本地。静默映射不报错，只有查表能提前看见

## 七、建议

1. **别硬编码档位名。** 按模型维护白名单，或读能力声明。`xhigh` 在 Qwen3.8 合法、GLM-5.1 越界、Claude Fable 5 不存在
2. **先跑默认档。** Qwen3.8 的 `xhigh`、ZHIPU/GLM-5.3 的 `max` 都是最贵档
3. **DeepSeek 甜区不在 `max`。** effort 60–80 拿接近满档准确率，80→100 只让轨迹再长 1.6–1.8 倍
4. **档位是成本开关。** 思考 token 全计输出、按输出价结算，GLM-5.3 输出价是输入 3 倍以上
5. **跨厂商 benchmark 对齐档位。** 比的是同档位还是同预算，先定
6. **固定真实任务集回归。** 官方跑分是满档跑出来的，迁移别直接套

## 参考文档

- [百炼：深度思考模型的用法](https://help.aliyun.com/zh/model-studio/deep-thinking) — GLM、Qwen3.8、DeepSeek、kimi-k3 各模型页档位与越界映射
- [百炼：GLM 调用文档](https://help.aliyun.com/zh/model-studio/glm)、[glm-5.3 模型页](https://help.aliyun.com/zh/model-studio/glm-5-3) — glm-5.3 三档、不可关、`clear_thinking` 默认 `true`
- [百炼：GLM-智谱直供](https://help.aliyun.com/zh/model-studio/glm-zhipu) — ZHIPU/GLM-5.3 默认 `max`、传 `disabled` 失败
- [百炼：文本生成模型 API 参考](https://help.aliyun.com/zh/model-studio/qwen-api-reference/) — 字段说明
- [OpenAI：Reasoning models](https://developers.openai.com/api/docs/guides/reasoning) — effort 取值与 `none`
- [Azure OpenAI：推理模型](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/reasoning) — 可取值与场景
- [Claude：Effort](https://platform.claude.com/docs/en/build-with-claude/effort)、[Claude：Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking) — adaptive thinking 与 `task_budget`
- [DeepSeek：思考模式](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode/) — 三档定义
- [智谱：深度思考](https://docs.bigmodel.cn/cn/guide/capabilities/thinking)、[智谱：GLM-5.3 模型页](https://docs.bigmodel.cn/cn/guide/models/text/glm-5.3) — 5.2 可关、5.3 强制
- [火山方舟：深度思考](https://www.volcengine.com/docs/82379/1956279) — `thinking` 与 effort 叠加限制
- DeepSeek V4.1 官方技术报告 — effort 标量、长度惩罚衰减、Pass@1 与 token 代价，无公开稳定 URL，未附链接