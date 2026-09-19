---
layout: post
title: "Agent Skills 渐进式披露：从单条 SQL 到多步自主分析"
date: 2026-09-19 06:00:00 +0000
categories: ai agent
tags: agent skills agentscope nl2sql
---

Agent Skills 是由指令、脚本和资源组成的模块化能力包，靠三层渐进式披露控制上下文加载。本文以 MySQL employees 库的自然语言数据分析为例，对比传统固定工作流与 Skills 方案的差别，并说明如何与 AgentScope 框架集成

## **一、Agent Skills 介绍**

### **1. 简介**

```
Create, manage, and share Skills to extend Claude's capabilities in Claude Code.
This guide shows you how to create, use, and manage Agent Skills in Claude Code. Skills are modular capabilities that extend Claude's functionality through organized folders containing instructions, scripts, and resources.
```

Claude Code 官方文档对 Agent Skills 的描述：**技能（Skill）** 是由**指令**、**脚本**和**资源**组成的模块化功能包，能够扩展 Claude 智能体的能力。这些技能存放在特定目录下（每个技能一个文件夹），当 Claude 判断某个技能与当前用户请求相关时，便会**自动加载**相应技能来完成任务。这种触发是**模型自主决策**的，而非需用户手动指定调用

### **2. 特性 – 三层渐进式披露**

Agent Skills 的核心设计理念是**渐进式披露（Progressive Disclosure）**。这意味着技能的信息分为三个层次，Claude **按需逐步加载**，既确保必要时不遗漏细节，又避免一次性将过多内容塞入上下文：

1. **元数据（Metadata）** – *第一层*：每个技能的 SKILL.md 文件开头都有 YAML 元信息，包括技能的名称和描述。例如，`name: mysql-employees-nl2sql-analysis`，`description: ...`。在智能体启动时，Claude 会预加载所有已安装技能的元数据到系统提示中。元数据提供了**何时使用该技能**的线索，让模型对技能用途有初步了解，但不包含具体实现细节
2. **技能主体（SKILL.md 内容）** – *第二层*：如果 Claude 判断某个技能与当前任务相关，它会进一步读取并加载该技能完整的 SKILL.md 内容作为上下文。这个 Markdown 文件中包含对任务的**详细指令**、注意事项、示例等，指导模型如何利用该技能解决问题
3. **附加文件和脚本** – *第三层*：对于更复杂或特定的场景，技能文件夹中可以包含脚本（如 `.py` 工具）或额外说明文档（如参考资料 `reference.md` 等）。SKILL.md 可以通过 Markdown 链接引用这些文件。当且仅当需要时，Claude **才会加载或运行这些附加内容**。例如，在 PDF 处理技能中，SKILL.md 引用了 `forms.md` 以提供额外的表单填充指南；Claude 只有在填表任务出现时才读取 `forms.md`。又如技能中包含 Python 脚本供执行，当需要复杂计算或访问数据库时，Claude 可以选择运行脚本工具，而无需将脚本代码全部读入上下文

借助如上所述的三层结构，Agent Skills 可实现**动态**的上下文管理。无关任务时，Claude **仅加载技能元数据**，几乎不占用上下文窗口；一旦识别出相关技能，再**逐步加载详细内容和工具**。这种按需加载机制使技能能够包含海量信息却不怕超出上下文窗口限制，大幅减少了重复 prompt

## **二、Agent Skills 能实现什么？**

Agent Skills 的出现，为复杂任务提供了比传统单轮工具调用更强大的解决方案。下面以**自然语言生成 SQL 并分析数据**的场景为例，对比传统工作流方案和 Agent Skills 方案的区别

**传统工作流方案（如 Dify、百炼、扣子等）**：采用固定的提示链或函数调用流程。例如，用户用自然语言提问 → LLM 根据 Prompt 生成一条 SQL → 通过 Model Context Protocol（MCP）或函数调用执行该 SQL 查询数据库 → 将查询结果返回给 LLM → LLM 再输出分析报告。这样的流程通常**只适用于一次性、单条 SQL 的情况**，流程是预先固化的。如果用户的问题需要**多步查询**才能回答（涉及多条 SQL 逐步获取信息），传统方案很难覆盖或需要手动将多个步骤串联，非常不灵活

![image.png](https://ucc.alicdn.com/pic/developer-ecology/uvmnudlhbcdek_bfb3226f19c44d1d96874b351790306c.png)

**Agent Skills 方案**：

```
skill/employees/
├── SKILL.md
└── scripts/
    ├── execute_sql.py
    └── __pycache__/
```

借助技能机制，Claude 能够在一次对话中**自主规划并执行一系列操作**，完成更复杂的闭环。本次以 MySQL 官方示例数据库 *employees*（员工信息库）为测试数据，定制 Agent Skills，用于将**中文业务问题**转换为受控的 SQL 查询并输出结构化分析报告

skill 主要包含两部分：一份详细的说明文件 SKILL.md 和一个 SQL 执行脚本 execute_sql.py（数据库连接信息通过环境变量配置）。技能的 YAML 元数据描述了它适用于 *employees* 数据库的分析任务，Claude 在需要做员工数据分析时会自动触发它。如下是该技能的 SKILL.md 结构示例：

```markdown
---
name: mysql-employees-nl2sql-analysis
description: 基于 MySQL 官方 employees 经典示例数据库，将中文业务问题转换为受控的单条 SQL，执行后输出结构化数据分析报告
---

# MySQL Employees 数据分析 Skill

本 Skill 专用于 **MySQL 官方 employees 经典示例数据库**，用于员工、部门、薪资、职位等分析场景。

你必须完成一个完整闭环：

1. **理解用户中文问题**
2. **生成一条严格受控的 MySQL SQL**
3. **调用 execute_sql.py 执行该 SQL**
4. **基于返回结果输出分析报告**

---

## 一、数据库背景（必须理解）
## 二、可用数据表（唯一允许）
## 三、你的职责边界（非常重要）
## 四、SQL 生成规则（硬约束）
## 五、execute_sql.py 使用协议（强制）
## 六、最终输出要求（给用户看的）
## 七、推荐 JOIN 模板（参考）
## 八、Examples
## 九、一句话总原则（给模型记住）
```

如上所示，我们在技能说明中明确了数据库背景、允许查询的表、模型需遵守的边界和规则（只生成**单条**且安全的 SQL）、工具使用方法以及输出格式要求等。借助这样详尽的指导，Claude 可以严格按照业务需求来行动。一旦这个技能被触发，Claude 会执行如下闭环流程：

- **Step1** 理解用户问题 → **Step2** 生成符合规则的 SQL → **Step3** 使用提供的 execute_sql.py 工具运行查询 → **Step4** 根据查询结果编写分析报告
- **动态多步查询示例**：假设用户问"分析下公司里面谁的话语权最高？"。单凭一条 SQL 无法直接回答这个复杂问题，因为需要综合考虑管理职级、薪资水平、技术影响力等多方面因素。传统方案难以完成这样的多角度分析。而启用 Agent Skills 后，Claude 会利用技能中的指导，**自主拆解问题**，通过多条查询逐步获取所需信息并最终给出报告

![image.png](https://ucc.alicdn.com/pic/developer-ecology/uvmnudlhbcdek_5ce3d9decc524188ab9c96e3180f643a.png)

在每一步，Claude 调用 execute_sql.py 执行 SQL 并获取结果，然后决定下一步需要哪方面的数据。最终，结合所有查询结果，整理出完整的分析结论。下面是模型根据这些查询结果给出的**分析报告示例（节选）**：

````text
## 公司话语权分析

### 使用的 SQL：

```sql
-- 查看管理层（Manager职位）
SELECT CONCAT(e.first_name, ' ', e.last_name) AS name, s.salary AS salary
FROM employees e
JOIN salaries s ON e.emp_no=s.emp_no AND s.to_date='9999-01-01'
JOIN titles t ON e.emp_no=t.emp_no AND t.to_date='9999-01-01'
WHERE t.title='Manager'
LIMIT 100;

-- 查看最高薪资员工
SELECT CONCAT(e.first_name, ' ', e.last_name) AS name, s.salary AS salary
FROM employees e
JOIN salaries s ON e.emp_no=s.emp_no AND s.to_date='9999-01-01'
ORDER BY s.salary DESC
LIMIT 10;

-- 查看部门管理者及其部门
SELECT CONCAT(e.first_name, ' ', e.last_name) AS name, d.dept_name AS department
FROM employees e
JOIN dept_manager dm ON e.emp_no=dm.emp_no AND dm.to_date='9999-01-01'
JOIN departments d ON dm.dept_no=d.dept_no
LIMIT 100;
```

***

### 分析报告：

#### （1）权力结构概览
从数据来看，这家公司的话语权分布呈现出**复合型权力结构**，既有正式的管理层，也有基于专业能力和薪酬水平的非正式影响力。

#### （2）正式管理层分析
**部门管理者（9人）：**
- **Vishwani Minakawa**（市场部）：薪资106,491 - 最高薪管理者
- **Hauke Zhang**（销售部）：薪资101,987 - 第二高薪管理者
- **Isamu Legleitner**（财务部）：薪资83,457
- 其他部门管理者薪资在56,654-79,393之间

**特点：**
- 市场部和销售部的管理者薪资明显高于其他部门，反映了这两个部门在公司中的战略重要性
- 虽然只有9名正式管理者，但他们分别负责9个核心部门，拥有直接的管理权限

#### （3）薪酬话语权分析
**最高薪资群体（前10名）：**
- 薪资范围：152,687 - 158,220
- **全部都是"Senior Staff"（高级职员）职位**
- 这些人的薪资远超正式管理者（最高管理者薪资106,491 vs 最高员工薪资158,220）
````

通过 Agent Skills，模型可以**灵活运用多条 SQL 逐步推理**，最终完成结构化、深入的分析报告输出。此项方案的主要优势在于：技能提供了完备的领域知识和操作规程，模型可以在单轮对话中完成**复杂的工作流**，而无需人为把任务拆解成多次提问。相较之下，传统方法要实现类似结果非常困难，而 Agent Skills 大大提升了智能体处理复杂任务的自动化程度

## **三、AgentScope 框架与 Skills 集成**

AgentScope 是达摩院开源的智能体框架，对标 LangChain，但更聚焦 **Agent 本身的推理、调度与工具调用能力**。本文实践中，AgentScope 作为运行框架，用于承载 Anthropic 的 Agent Skills 机制

最近，AgentScope 也宣布支持了 **Anthropic Agent Skills**，使得技能的发现、加载与执行可以直接集成到 Agent 运行时中。本次 demo 正是基于该特性，实现了 Skill 的渐进式加载与多步工具调用

![image.png](https://ucc.alicdn.com/pic/developer-ecology/uvmnudlhbcdek_969e015cdab840a4aba035cdd7da8f77.png)

在整体架构中：

- **AgentScope** 负责接入大模型、消息流转、推理循环与工具调度
- **Agent Skills** 提供领域约束、操作规范与可复用能力

## 参考文档

- [Claude Code Skills 官方文档（英文）](https://code.claude.com/docs/en/skills)
- [Anthropic 工程博客：Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [AgentScope 集成 Agent Skills 示例（GitHub）](https://github.com/agentscope-ai/agentscope/tree/main/examples/functionality/agent_skill)
