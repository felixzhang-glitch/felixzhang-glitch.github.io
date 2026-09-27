---
layout: post
title: "NOTE-FORMAT：basic-memory 的笔记格式"
date: 2026-09-27
categories: knowledge markdown
tags: basic-memory knowledge-base mcp picoschema
---

## 简述

NOTE-FORMAT 是 basic-memory 对笔记文件的语法约定。不扩展 Markdown，只约定列表项里的括号写法，让同一份文件既能给人读，也能解析成知识图谱。文件是真源，数据库里的图是文件的投影，文件改动后同步更新。

## 定义

一份 note 拆三段：frontmatter、观测（observation）、关系（relation）。后两段靠语法识别，不依赖标题。frontmatter 说明实体是谁，observation 记录事实，relation 把实体连成图。

## 范例

以一条人物笔记为例：

```markdown
---
title: Paul Graham
type: Person
tags: [startups, essays, lisp]
---

# Paul Graham

## Observations
- [name] Paul Graham
- [role] Essayist and investor
- [expertise] Startups
- [expertise] Lisp
- [expertise] Essay writing
- [fact] Created Viaweb, the first web app

## Relations
- works_at [[Y Combinator]]
- authored [[Hackers and Painters]]
```

`## Observations`、`## Relations` 是惯例，删掉不影响解析。

## 对比

| 维度 | 自由 Markdown | NOTE-FORMAT |
| --- | --- | --- |
| 写入 | 没有约束 | 列表项加 `[category]` 前缀 |
| 机器可见 | 只有正文 | 分类、标签、上下文、关系各自成字段 |
| 图结构 | 需另建索引 | `[[wikilink]]` 直接产生边 |
| 校验 | 无 | 可挂 schema 检查字段 |
| 代价 | 零 | 逐行写前缀 |

差异在机器能否读懂一行。纯人读的笔记用自由 Markdown；要检索和跨文件问答，需要这个前缀。

## 分层

### 第一层：frontmatter

说明实体是谁

```yaml
title: Paul Graham
type: Person
tags: [startups, essays, lisp]
permalink: paul-graham
```

字段都可缺省：`title` 取文件名，`type` 默认 `note`，`permalink` 由 title 生成。非标准字段（如 `status`、`source`）存进 `entity_metadata`，参与检索。

触发条件：要按类型过滤或稳定寻址时才显式写 `type`、`permalink`。

入库前归一化：日期保持 ISO 字符串，数字转字符串，布尔转 `"True"`，列表和字典递归处理。

这条笔记登记为一个 type 为 `Person`、带三个 tag 的实体。

### 第二层：observation

记录实体的事实

```markdown
- [category] content text #tag1 #tag2 (context)
```

对应正则 `^\[([^\[\]()]+)\]\s+(.+)`，边界四条：

- category 不含 `[`、`]`、`(`、`)`
- `]` 后必须有空白
- content 不能为空
- context 只能是行尾一个括号组

数组靠重复 category 表达，例子里 `[expertise]` 出现三次。`#tag` 是行内标签，`(context)` 补充出处。

触发条件：列表项会被尝试解析成观测，五种形态排除：

- `- [ ]`、`- [x]`、`- [-]` 是任务勾选框
- `- [text](url)` 是链接
- `- [[Target]]` 按关系处理
- `- [00:00:11]` 这类时间码不算 category
- `[/]`、`[>]`、`[?]` 等扩展任务标记同样排除

没 category 但有 `#tag` 的行也接受，category 落默认值 `Note`。这是兜底，不是推荐写法。

这条笔记在这一层贡献七条带分类的事实。

### 第三层：relation

说明实体连到谁

```markdown
- relation_type [[Target Entity]] (context)
```

规则两条：

- type 是 `[[` 前的单 token；多词加引号，如 `"based on" [[Customer Interview]]`
- 目标后跟非单括号组的尾巴，整行退化为隐式 `links_to`

退化是硬边界。`This builds on [[Core Design]] and uses [[Utility Functions]]` 里两个 wikilink 产生的是 `links_to`，不会把 `builds` 当 type。

强制走 `links_to` 用行尾指令：

```markdown
- Mother [[Alice]] #bm:links_to
```

`#bm:links_to` 是解析元数据，不进正文。目标按 title 或 permalink 匹配，支持前向引用：先写边，目标文件后建。

这条笔记在这一层产生两条边：`works_at`、`authored`。

### 第四层：permalink

让文件移动后仍可定位

```yaml
permalink: paul-graham
```

`memory://` 三种写法：

```text
memory://paul-graham           # 按 permalink
memory://Paul Graham           # 按 title
memory://people/paul-graham    # 按路径
```

支持 `*` 通配。permalink 稳定，挪目录不影响寻址。

### 第五层：schema

规定一类实体该有哪些字段

schema 本身也是一条 note，type 为 `schema`：

```yaml
---
title: Person
type: schema
entity: Person
version: 1
schema:
  name: string, full name
  role?: string, job title or position
  works_at?: Organization, employer
  expertise?(array): string, areas of knowledge
  email?: string, contact email
settings:
  validation: warn
---
```

schema 用 Picoschema，来自 Google Dotprompt，basic-memory 直接采用。规则：

- `name: string` 必需，`role?: string` 可选
- `(array)` 数组，`(enum)` 枚举
- 首字母大写的类型（如 `Organization`）是实体引用，落到 relation
- 逗号后是描述

挂载四种，按优先级：frontmatter 内联 dict、字符串引用 schema note 名、按 `type` 隐式匹配 `entity: Person`、不挂。不挂合法。schema 只管子集，`[fact] Created Viaweb` 不在 Person schema 里，校验时归入 unmatched，不报错。

校验两档：`warn` 默认告警，`strict` 阻断同步，给 CI 用。

## 校验

```text
$ bm schema validate people/ada-lovelace.md

⚠ Person schema validation:
  - Missing required field: name (expected [name] observation)
  - Missing optional field: role
  - Missing optional field: works_at (no relation found)

ℹ Unmatched observations: [fact] ×2, [born] ×1
ℹ Unmatched relations: collaborated_with
```

输入文件，输出缺字段清单和未匹配项。改到必需字段满足为止。

## 边界

- `## Observations` / `## Relations` 是惯例，不是语法
- category 不能含 `[`、`]`、`(`、`)`
- 多词 relation type 必须加引号，否则整行变 `links_to`
- relation type 不是封闭集合，`implements`、`works_at` 只是惯例
- 仓库有两份 NOTE-FORMAT.md，根目录是旧拷贝：`relation_type` 标成可省略、缺省 `relates_to`，且没有 `#bm:links_to`、引号多词关系、canonical 时间戳。以 `docs/NOTE-FORMAT.md` 为准，代码行为与 docs 版一致
- 文档没写但代码已实现：observation 的 category 后可带时态限定符，`@effective[2026-06-10,2026-07-27)`、`@effective:2026-07-27`、`@2026-07-27`。未加引号的点值只吃一个 token，读不出时间则原样留在正文。规则见 `temporal_qualifier.py`

## 参考来源

- [basic-memory/docs/NOTE-FORMAT.md](https://github.com/basicmachines-co/basic-memory/blob/main/docs/NOTE-FORMAT.md)
- basic-memory 根目录 NOTE-FORMAT.md
- `src/basic_memory/markdown/plugins.py`、`temporal_qualifier.py`、`src/basic_memory/picoschema/parser.py`
- [Picoschema（Google Dotprompt）](https://google.github.io/dotprompt/reference/picoschema/)
