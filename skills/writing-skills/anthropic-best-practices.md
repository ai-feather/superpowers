# 技能编写最佳实践

> 了解如何编写有效的技能，让代理能够发现并成功使用它们。

好的技能简洁、结构良好，并经过真实使用的测试。本指南提供实用的编写决策，帮助你写出代理能发现并有效使用的技能。

关于技能工作原理的概念背景，参见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是一种公共资源。你的技能与代理需要知道的一切共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他技能的元数据
* 你的实际请求

你的技能里并非每个 token 都有即时成本。启动时，只有所有技能的元数据（name 和 description）会被预加载。代理只在技能变得相关时才读 SKILL.md，并按需读取额外文件。然而，SKILL.md 的简洁仍然重要：一旦代理加载了它，每个 token 都在与对话历史和其他上下文竞争。

**默认假设**：代理已经非常聪明

只添加代理尚不具备的上下文。对每一条信息都要追问：

* "代理真的需要这个解释吗？"
* "我能假设代理知道这个吗？"
* "这一段对得起它的 token 成本吗？"

**好示例：简洁**（约 50 token）：

````markdown  theme={null}
## 提取 PDF 文本

用 pdfplumber 提取文本：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**坏示例：太啰嗦**（约 150 token）：

```markdown  theme={null}
## 提取 PDF 文本

PDF（Portable Document Format）文件是一种常见的文件格式，包含
文本、图片和其他内容。要从 PDF 中提取文本，你需要
使用一个库。有很多可用于 PDF 处理的库，但我们
推荐 pdfplumber，因为它易用且能很好地处理大多数情况。
首先，你需要用 pip 安装它。然后你可以使用下面的代码……
```

简洁版本假设代理知道 PDF 是什么以及库如何工作。

### 设定合适的自由度

让具体程度匹配任务的脆弱性和可变性。

**高自由度**（基于文本的指令）：

何时使用：

* 多种方法都有效
* 决策取决于上下文
* 启发式引导方法

示例：

```markdown  theme={null}
## 代码评审流程

1. 分析代码结构和组织
2. 检查潜在的 bug 或边界情况
3. 提出可读性和可维护性改进
4. 验证是否遵循项目约定
```

**中等自由度**（带参数的伪代码或脚本）：

何时使用：

* 存在首选模式
* 某些变体可接受
* 配置影响行为

示例：

````markdown  theme={null}
## 生成报告

使用此模板并按需定制：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 以指定格式生成输出
    # 可选地包含可视化
```
````

**低自由度**（特定脚本，很少或没有参数）：

何时使用：

* 操作脆弱且易错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## 数据库迁移

精确运行此脚本：

```bash
python scripts/migrate.py --verify --backup
```

不要修改命令或添加额外 flag。
````

**类比**：把代理想象成一个在路径上探索的机器人：

* **两边都是悬崖的窄桥**：只有一条安全的前进路径。提供具体的护栏和精确指令（低自由度）。示例：必须按精确顺序运行的数据库迁移。
* **没有任何危险的旷野**：很多路径都能通向成功。给个大方向，相信代理能找到最佳路线（高自由度）。示例：由上下文决定最佳方法的代码评审。

### 用你计划使用的所有模型测试

技能是对模型的补充，所以效果取决于底层模型。用你计划使用的所有模型测试你的技能。

**按模型的测试考量**：

* **Claude Haiku**（快速、经济）：技能是否提供了足够指引？
* **Claude Sonnet**（平衡）：技能是否清晰高效？
* **Claude Opus**（强大推理）：技能是否避免了过度解释？

对 Opus 完美适用的，对 Haiku 可能需要更多细节。如果你计划跨多个模型使用技能，目标是让指令在所有模型上都好用。

## 技能结构

<Note>
  **YAML Frontmatter**：SKILL.md 的 frontmatter 需要两个字段：

  * `name` - 技能的人类可读名称（最多 64 字符）
  * `description` - 一行描述，说明技能做什么以及何时使用（最多 1024 字符）

  完整的技能结构细节，参见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式，让技能更容易被引用和讨论。我们建议技能名使用**动名词形式**（动词 + -ing），因为这清楚地描述了技能提供的活动或能力。

**好的命名示例（动名词形式）**：

* "Processing PDFs"
* "Analyzing spreadsheets"
* "Managing databases"
* "Testing code"
* "Writing documentation"

**可接受的替代**：

* 名词短语："PDF Processing"、"Spreadsheet Analysis"
* 动作导向："Process PDFs"、"Analyze Spreadsheets"

**避免**：

* 含糊的名字："Helper"、"Utils"、"Tools"
* 过于通用："Documents"、"Data"、"Files"
* 你的技能集合内部不一致的模式

一致的命名让你更容易：

* 在文档和对话中引用技能
* 一眼理解技能做什么
* 组织和搜索多个技能
* 维护一个专业、有凝聚力的技能库

### 编写有效的描述

`description` 字段支撑技能发现，应当同时包含技能做什么以及何时使用。

<Warning>
  **始终用第三人称写**。description 会被注入到系统提示中，不一致的视角会导致发现问题。

  * **好：** "Processes Excel files and generates reports"
  * **避免：** "I can help you process Excel files"
  * **避免：** "You can use this to process Excel files"
</Warning>

**要具体并包含关键术语**。既要包含技能做什么，也要包含何时使用的具体触发条件/上下文。

每个技能恰好有一个 description 字段。description 对技能选择至关重要：代理用它从可能 100+ 个可用技能中选出正确的。你的 description 必须提供足够细节让代理知道何时选择这个技能，而 SKILL.md 的其余部分提供实现细节。

有效示例：

**PDF 处理技能：**

```yaml  theme={null}
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

**Excel 分析技能：**

```yaml  theme={null}
description: Analyze Excel spreadsheets, create pivot tables, generate charts. Use when analyzing Excel files, spreadsheets, tabular data, or .xlsx files.
```

**Git 提交助手技能：**

```yaml  theme={null}
description: Generate descriptive commit messages by analyzing git diffs. Use when the user asks for help writing commit messages or reviewing staged changes.
```

避免像下面这样含糊的描述：

```yaml  theme={null}
description: Helps with documents
```

```yaml  theme={null}
description: Processes data
```

```yaml  theme={null}
description: Does stuff with files
```

### 渐进式披露模式

SKILL.md 作为概览，按需把代理指向详细材料，就像入职指南里的目录。关于渐进式披露如何工作的解释，参见概览中的 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指引：**

* 让 SKILL.md 正文保持在 500 行以内以获得最佳性能
* 接近此限制时把内容拆分到独立文件
* 使用下面的模式有效组织指令、代码和资源

#### 可视化概览：从简单到复杂

一个基础技能一开始只有一个 SKILL.md 文件，包含元数据和指令：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="Simple SKILL.md file showing YAML frontmatter and markdown body" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着你的技能增长，你可以打包额外内容，代理只在需要时加载：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="Bundling additional reference files like reference.md and forms.md." data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的技能目录结构可能长这样：

```
pdf/
├── SKILL.md              # 主指令（触发时加载）
├── FORMS.md              # 填表指南（按需加载）
├── reference.md          # API 参考（按需加载）
├── examples.md           # 用法示例（按需加载）
└── scripts/
    ├── analyze_form.py   # 工具脚本（执行，不加载）
    ├── fill_form.py      # 填表脚本
    └── validate.py       # 校验脚本
```

#### 模式 1：带参考的高层指南

````markdown  theme={null}
---
name: PDF Processing
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
---

# PDF 处理

## 快速开始

用 pdfplumber 提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 高级特性

**填表**：完整指南见 [FORMS.md](FORMS.md)
**API 参考**：所有方法见 [REFERENCE.md](REFERENCE.md)
**示例**：常见模式见 [EXAMPLES.md](EXAMPLES.md)
````

代理只在需要时才加载 FORMS.md、REFERENCE.md 或 EXAMPLES.md。

#### 模式 2：按领域组织

对于多领域的技能，按领域组织内容，避免加载无关上下文。当用户问销售指标时，代理只需读销售相关的 schema，而不是财务或营销数据。这能保持 token 用量低、上下文聚焦。

```
bigquery-skill/
├── SKILL.md（概览和导航）
└── reference/
    ├── finance.md（收入、计费指标）
    ├── sales.md（商机、销售管线）
    ├── product.md（API 用量、功能）
    └── marketing.md（活动、归因）
```

````markdown SKILL.md theme={null}
# BigQuery 数据分析

## 可用数据集

**财务**：收入、ARR、计费 → 见 [reference/finance.md](reference/finance.md)
**销售**：商机、管线、账户 → 见 [reference/sales.md](reference/sales.md)
**产品**：API 用量、功能、采用 → 见 [reference/product.md](reference/product.md)
**营销**：活动、归因、邮件 → 见 [reference/marketing.md](reference/marketing.md)

## 快速搜索

用 grep 查找具体指标：

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式 3：条件性细节

展示基础内容，链接到高级内容：

```markdown  theme={null}
# DOCX 处理

## 创建文档

新文档用 docx-js。见 [DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

简单编辑直接改 XML。

**对于修订追踪**：见 [REDLINING.md](REDLINING.md)
**对于 OOXML 细节**：见 [OOXML.md](OOXML.md)
```

代理只在用户需要那些功能时才读 REDLINING.md 或 OOXML.md。

### 避免深层嵌套引用

当文件从其他被引用文件中被引用时，代理可能只部分读取。遇到嵌套引用时，代理可能用 `head -100` 之类的命令预览内容，而不是读整个文件，导致信息不完整。

**让引用从 SKILL.md 算起只有一层深**。所有参考文件都应直接从 SKILL.md 链接，以确保代理在需要时读取完整文件。

**坏示例：太深**：

```markdown  theme={null}
# SKILL.md
见 [advanced.md](advanced.md)……

# advanced.md
见 [details.md](details.md)……

# details.md
这里是实际信息……
```

**好示例：一层深**：

```markdown  theme={null}
# SKILL.md

**基础用法**：[SKILL.md 里的指令]
**高级特性**：见 [advanced.md](advanced.md)
**API 参考**：见 [reference.md](reference.md)
**示例**：见 [examples.md](examples.md)
```

### 用目录结构组织较长的参考文件

对于超过 100 行的参考文件，在顶部放一个目录。这确保代理即使只用部分读取预览，也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API 参考

## 目录
- 认证与设置
- 核心方法（创建、读取、更新、删除）
- 高级特性（批处理、webhook）
- 错误处理模式
- 代码示例

## 认证与设置
...

## 核心方法
...
```

代理然后可以读完整文件或按需跳到具体章节。

关于这种基于文件系统的架构如何实现渐进式披露的细节，见下方"高级"章节的 [Runtime environment](#runtime-environment) 部分。

## 工作流与反馈循环

### 为复杂任务使用工作流

把复杂操作拆成清晰的顺序步骤。对于特别复杂的工作流，提供一个清单，代理可以把它复制到响应中并随着推进逐项勾选。

**示例 1：研究综合工作流**（用于无代码的技能）：

````markdown  theme={null}
## 研究综合工作流

复制此清单并跟踪你的进度：

```
Research Progress:
- [ ] Step 1: Read all source documents
- [ ] Step 2: Identify key themes
- [ ] Step 3: Cross-reference claims
- [ ] Step 4: Create structured summary
- [ ] Step 5: Verify citations
```

**Step 1：阅读所有源文档**

审阅 `sources/` 目录中的每个文档。记下主要论点和支持证据。

**Step 2：识别关键主题**

在来源间寻找模式。哪些主题反复出现？来源在哪里一致或分歧？

**Step 3：交叉验证论点**

对每个主要论点，验证它出现在源材料中。记下哪个来源支持每个点。

**Step 4：创建结构化摘要**

按主题组织发现。包含：
- 主要论点
- 来自来源的支持证据
- 冲突观点（如有）

**Step 5：验证引用**

检查每个论点引用了正确的源文档。如果引用不完整，回到 Step 3。
````

这个示例展示了工作流如何适用于不需要代码的分析任务。清单模式适用于任何复杂的多步过程。

**示例 2：PDF 填表工作流**（用于有代码的技能）：

````markdown  theme={null}
## PDF 填表工作流

复制此清单，完成一项就勾选一项：

```
Task Progress:
- [ ] Step 1: Analyze the form (run analyze_form.py)
- [ ] Step 2: Create field mapping (edit fields.json)
- [ ] Step 3: Validate mapping (run validate_fields.py)
- [ ] Step 4: Fill the form (run fill_form.py)
- [ ] Step 5: Verify output (run verify_output.py)
```

**Step 1：分析表单**

运行：`python scripts/analyze_form.py input.pdf`

这会提取表单字段及其位置，保存到 `fields.json`。

**Step 2：创建字段映射**

编辑 `fields.json`，为每个字段添加值。

**Step 3：验证映射**

运行：`python scripts/validate_fields.py fields.json`

在继续前修正任何校验错误。

**Step 4：填表**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**Step 5：验证输出**

运行：`python scripts/verify_output.py output.pdf`

如果验证失败，回到 Step 2。
````

清晰的步骤防止代理跳过关键校验。清单帮助你和代理都跟踪多步工作流的进度。

### 实现反馈循环

**常见模式**：运行校验器 → 修错误 → 重复

这个模式大幅提升输出质量。

**示例 1：风格指南合规**（用于无代码的技能）：

```markdown  theme={null}
## 内容评审流程

1. 按 STYLE_GUIDE.md 中的指南起草内容
2. 对照清单评审：
   - 检查术语一致性
   - 验证示例遵循标准格式
   - 确认所有必需章节都存在
3. 如果发现问题：
   - 带具体章节引用记录每个问题
   - 修订内容
   - 再次评审清单
4. 只有所有要求都满足才继续
5. 定稿并保存文档
```

这展示了用参考文档而非脚本的校验循环模式。"校验器"是 STYLE\_GUIDE.md，代理通过阅读和比较来执行检查。

**示例 2：文档编辑流程**（用于有代码的技能）：

```markdown  theme={null}
## 文档编辑流程

1. 对 `word/document.xml` 做你的编辑
2. **立即校验**：`python ooxml/scripts/validate.py unpacked_dir/`
3. 如果校验失败：
   - 仔细审阅错误信息
   - 修正 XML 中的问题
   - 再次运行校验
4. **只有校验通过才继续**
5. 重新打包：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试输出文档
```

校验循环能尽早捕获错误。

## 内容指南

### 避免时效性信息

不要包含会过时的信息：

**坏示例：时效性**（会变错）：

```markdown  theme={null}
If you're doing this before August 2025, use the old API.
After August 2025, use the new API.
```

**好示例**（用"旧模式"章节）：

```markdown  theme={null}
## 当前方法

使用 v2 API 端点：`api.example.com/v2/messages`

## 旧模式

<details>
<summary>Legacy v1 API (deprecated 2025-08)</summary>

The v1 API used: `api.example.com/v1/messages`

This endpoint is no longer supported.
</details>
```

旧模式章节提供历史背景，又不会让主内容变得杂乱。

### 使用一致的术语

选定一个术语，在整个技能中都用它：

**好 - 一致**：

* 总是用 "API endpoint"
* 总是用 "field"
* 总是用 "extract"

**坏 - 不一致**：

* 混用 "API endpoint"、"URL"、"API route"、"path"
* 混用 "field"、"box"、"element"、"control"
* 混用 "extract"、"pull"、"get"、"retrieve"

一致性帮助代理理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。让严格程度匹配你的需要。

**对于严格要求**（如 API 响应或数据格式）：

````markdown  theme={null}
## 报告结构

始终使用这个精确的模板结构：

```markdown
# [分析标题]

## 执行摘要
[关键发现的一段话概述]

## 关键发现
- 发现 1 带支持数据
- 发现 2 带支持数据
- 发现 3 带支持数据

## 建议
1. 具体可执行的建议
2. 具体可执行的建议
```
````

**对于灵活指引**（当适应有用时）：

````markdown  theme={null}
## 报告结构

这是一个合理的默认格式，但请基于分析运用你的最佳判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 关键发现
[根据你的发现调整章节]

## 建议
[针对具体上下文定制]
```

按需针对具体分析类型调整章节。
````

### 示例模式

对于输出质量依赖于看到示例的技能，像常规提示一样提供输入/输出对：

````markdown  theme={null}
## 提交信息格式

按这些示例生成提交信息：

**示例 1：**
输入：Added user authentication with JWT tokens
输出：
```
feat(auth): implement JWT-based authentication

Add login endpoint and token validation middleware
```

**示例 2：**
输入：Fixed bug where dates displayed incorrectly in reports
输出：
```
fix(reports): correct date formatting in timezone conversion

Use UTC timestamps consistently across report generation
```

**示例 3：**
输入：Updated dependencies and refactored error handling
输出：
```
chore: update dependencies and refactor error handling

- Upgrade lodash to 4.17.21
- Standardize error response format across endpoints
```

遵循这种风格：type(scope): 简短描述，然后详细说明。
````

示例比单独的描述更能帮助代理理解期望的风格和详细程度。

### 条件工作流模式

引导代理走过决策点：

```markdown  theme={null}
## 文档修改工作流

1. 确定修改类型：

   **创建新内容？** → 遵循下方"创建工作流"
   **编辑既有内容？** → 遵循下方"编辑工作流"

2. 创建工作流：
   - 使用 docx-js 库
   - 从零构建文档
   - 导出为 .docx 格式

3. 编辑工作流：
   - 解包既有文档
   - 直接修改 XML
   - 每次改动后校验
   - 完成后重新打包
```

<Tip>
  如果工作流变得很大、很复杂、步骤很多，考虑把它们推到独立文件里，并告诉代理根据手头的任务读取合适的文件。
</Tip>

## 评估与迭代

### 先构建评估

**在编写大量文档之前先创建评估。** 这确保你的技能解决真实问题，而不是记录想象出来的问题。

**评估驱动开发：**

1. **识别缺口**：在没有技能的情况下，用代理跑代表性任务。记录具体的失败或缺失的上下文
2. **创建评估**：构建三个测试这些缺口的场景
3. **建立基线**：在没有技能的情况下衡量代理的表现
4. **编写最小指令**：只创建刚好足以应对缺口并通过评估的内容
5. **迭代**：执行评估，与基线比较，并精炼

这种方法确保你在解决实际问题，而不是预判可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "Extract all text from this PDF file and save it to output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "Successfully reads the PDF file using an appropriate PDF processing library or command-line tool",
    "Extracts text content from all pages in the document without missing any pages",
    "Saves the extracted text to a file named output.txt in a clear, readable format"
  ]
}
```

<Note>
  这个示例展示了一个带简单测试评分标准的数据驱动评估。我们目前没有提供内置方式来运行这些评估。用户可以创建自己的评估系统。评估是你衡量技能有效性的真相来源。
</Note>

### 与代理一起迭代开发技能

最有效的技能开发过程涉及代理本身。与一个实例（"代理 A"）合作创建一个将被其他实例（"代理 B"）使用的技能。代理 A 帮助你设计和精炼指令，而代理 B 在真实任务中测试它们。这之所以有效，是因为底层模型既理解如何编写有效的代理指令，也理解代理需要什么信息。

**创建新技能：**

1. **在没有技能的情况下完成一个任务**：用代理 A 通过正常提示解决一个问题。在过程中，你会自然地提供上下文、解释偏好、分享程序性知识。注意你反复提供的信息。

2. **识别可复用模式**：完成任务后，识别你提供的哪些上下文对类似的未来任务有用。

   **示例**：如果你做过一次 BigQuery 分析，你可能提供了表名、字段定义、过滤规则（比如"总是排除测试账户"）以及常见查询模式。

3. **让代理 A 创建技能**："创建一个技能，捕获我们刚用的这个 BigQuery 分析模式。包含表 schema、命名约定和过滤测试账户的规则。"

   <Tip>
     现代代理原生理解技能格式和结构。你不需要特殊的系统提示或"writing skills"技能来获得创建技能的帮助。只需让代理创建一个技能，它就会生成结构正确、带合适 frontmatter 和正文的 SKILL.md 内容。
   </Tip>

4. **审查简洁性**：检查代理 A 是否添加了不必要的解释。问："去掉关于胜率含义的解释——代理已经知道那个。"

5. **改进信息架构**：让代理 A 更有效地组织内容。例如："把这个组织一下，让表 schema 在单独的参考文件里。我们以后可能会加更多表。"

6. **在相似任务上测试**：把技能给代理 B（一个加载了该技能的新实例），在相关用例上使用。观察代理 B 是否找到正确信息、正确应用规则，并成功处理任务。

7. **基于观察迭代**：如果代理 B 挣扎或漏了什么，带着具体细节回到代理 A："当代理用了这个技能时，它忘了按 Q4 日期过滤。我们是不是该加一个关于日期过滤模式的章节？"

**迭代既有技能：**

同样的层级模式在改进技能时继续。你在以下之间交替：

* **与代理 A 合作**（帮助精炼技能的专家）
* **用代理 B 测试**（使用技能完成真实工作的代理）
* **观察代理 B 的行为**并把洞见带回给代理 A

1. **在真实工作流中使用技能**：给代理 B（加载了技能）真实任务，而非测试场景

2. **观察代理 B 的行为**：注意它在哪挣扎、成功或做出意外选择

   **示例观察**："当我向代理 B 要一份区域销售报告时，它写了查询却忘了过滤掉测试账户，尽管技能提到了这条规则。"

3. **回到代理 A 改进**：分享当前 SKILL.md 并描述你观察到的。问："我注意到代理 B 在我要区域报告时忘了过滤测试账户。技能提到了过滤，但也许不够突出？"

4. **审查代理 A 的建议**：代理 A 可能建议重新组织让规则更突出、用更强的语言如"MUST filter"而非"always filter"，或重构工作流章节。

5. **应用并测试改动**：用代理 A 的精炼更新技能，然后在相似请求上再次用代理 B 测试

6. **基于使用重复**：随着遇到新场景，继续这个观察-精炼-测试循环。每次迭代都基于真实代理行为改进技能，而非假设。

**收集团队反馈：**

1. 与队友分享技能并观察他们的使用
2. 问：技能是否在预期时激活？指令清楚吗？缺什么？
3. 纳入反馈以解决你自己使用模式中的盲点

**为什么这个方法有效**：代理 A 理解代理需求，你提供领域专长，代理 B 通过真实使用揭示缺口，而迭代精炼基于观察到的行为而非假设来改进技能。

### 观察代理如何导航技能

在迭代技能时，注意代理在实践中实际如何使用它们。留意：

* **意外的探索路径**：代理是否按你没预料到的顺序读文件？这可能表明你的结构不如你以为的直观
* **错过的连接**：代理是否没能跟随对重要文件的引用？你的链接可能需要更明确或更突出
* **过度依赖某些章节**：如果代理反复读同一个文件，考虑那些内容是否应该放进主 SKILL.md
* **被忽略的内容**：如果代理从不访问某个打包文件，它可能不必要，或在主指令中信号不佳

基于这些观察而非假设来迭代。你技能元数据中的'name'和'description'尤为关键。代理在决定是否响应当前任务触发技能时使用它们。确保它们清楚描述技能做什么以及何时使用。

## 要避免的反模式

### 避免使用 Windows 风格路径

文件路径始终用正斜杠，即使在 Windows 上：

* ✓ **好**：`scripts/helper.py`、`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

Unix 风格路径跨所有平台工作，而 Windows 风格路径在 Unix 系统上会出错。

### 避免提供太多选项

除非必要，不要呈现多种方法：

````markdown  theme={null}
**坏示例：选择太多**（令人困惑）：
"You can use pypdf, or pdfplumber, or PyMuPDF, or pdf2image, or..."

**好示例：提供一个默认**（带逃生口）：
"Use pdfplumber for text extraction:
```python
import pdfplumber
```

For scanned PDFs requiring OCR, use pdf2image with pytesseract instead."
````

## 高级：带可执行代码的技能

下面各节聚焦于包含可执行脚本的技能。如果你的技能只用 markdown 指令，跳到 [Checklist for effective Skills](#checklist-for-effective-skills)。

### 解决，而非甩锅

在为技能编写脚本时，处理错误条件，而不是甩给代理。

**好示例：显式处理错误**：

```python  theme={null}
def process_file(path):
    """处理一个文件，如果不存在就创建它。"""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # 用默认内容创建文件，而不是失败
        print(f"File {path} not found, creating default")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # 提供替代方案，而不是失败
        print(f"Cannot access {path}, using default")
        return ''
```

**坏示例：甩给代理**：

```python  theme={null}
def process_file(path):
    # 直接失败，让代理自己想办法
    return open(path).read()
```

配置参数也应当被论证和记录，以避免"魔法常量"（Ousterhout 定律）。如果你不知道正确的值，代理又如何确定它？

**好示例：自文档化**：

```python  theme={null}
# HTTP 请求通常在 30 秒内完成
# 更长的超时考虑了慢速连接
REQUEST_TIMEOUT = 30

# 三次重试平衡可靠性与速度
# 大多数间歇性故障在第二次重试时解决
MAX_RETRIES = 3
```

**坏示例：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # 为什么是 47？
RETRIES = 5   # 为什么是 5？
```

### 提供工具脚本

即使你的代理能写脚本，预制脚本也有优势：

**工具脚本的好处**：

* 比生成的代码更可靠
* 省 token（无需把代码放进上下文）
* 省时间（无需生成代码）
* 跨使用确保一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="Bundling executable scripts alongside instruction files" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上面的图展示了可执行脚本如何与指令文件协作。指令文件（forms.md）引用脚本，代理可以执行它而无需把其内容加载进上下文。

**重要区分**：在你的指令中清楚说明代理应该：

* **执行脚本**（最常见）："运行 `analyze_form.py` 来提取字段"
* **把它当参考读**（用于复杂逻辑）："字段提取算法见 `analyze_form.py`"

对于大多数工具脚本，执行更受青睐，因为它更可靠、更高效。脚本执行如何工作的细节见下方的 [Runtime environment](#runtime-environment) 章节。

**示例**：

````markdown  theme={null}
## 工具脚本

**analyze_form.py**：从 PDF 提取所有表单字段

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

输出格式：
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**：检查重叠的边界框

```bash
python scripts/validate_boxes.py fields.json
# 返回："OK" 或列出冲突
```

**fill_form.py**：把字段值应用到 PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用可视化分析

当输入可以被渲染成图片时，让代理分析它们：

````markdown  theme={null}
## 表单布局分析

1. 把 PDF 转成图片：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 分析每页图片以识别表单字段
3. 代理可以可视化地看到字段位置和类型
````

<Note>
  在这个示例中，你需要自己写 `pdf_to_images.py` 脚本。
</Note>

代理的视觉能力有助于理解布局和结构。

### 创建可验证的中间产物

当代理执行复杂的开放式任务时，它们可能犯错。"计划-验证-执行"模式通过让代理先用结构化格式创建一个计划，再用脚本验证该计划，然后才执行，从而尽早捕获错误。

**示例**：想象让代理基于一个电子表格更新 PDF 中的 50 个表单字段。没有验证，它可能引用不存在的字段、创建冲突值、漏掉必需字段，或错误地应用更新。

**解决方案**：使用上面展示的工作流模式（PDF 填表），但加一个中间 `changes.json` 文件，在应用改动前先校验。工作流变成：分析 → **创建计划文件** → **校验计划** → 执行 → 验证。

**为什么这个模式有效：**

* **尽早捕获错误**：校验在改动应用前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆的计划**：代理可以在不碰原件的情况下迭代计划
* **清晰的调试**：错误信息指向具体问题

**何时使用**：批处理操作、破坏性改动、复杂校验规则、高风险操作。

**实现技巧**：让校验脚本详细，带具体错误信息，如"Field 'signature\_date' not found. Available fields: customer\_name, order\_total, signature\_date\_signed"，以帮助代理修正问题。

### 打包依赖

技能在代码执行环境中运行，有平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，并从 GitHub 仓库拉取
* **Anthropic API**：没有网络访问，没有运行时包安装

在你的 SKILL.md 中列出所需包，并验证它们在 [代码执行工具文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool) 中可用。

### 运行时环境

技能在带文件系统访问、bash 命令和代码执行能力的代码执行环境中运行。关于这个架构的概念解释，见概览中的 [The Skills architecture](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这如何影响你的编写：**

**代理如何访问技能：**

1. **元数据预加载**：启动时，所有技能 YAML frontmatter 中的 name 和 description 被加载进系统提示
2. **文件按需读取**：代理按需使用其文件读取工具从文件系统访问 SKILL.md 和其他文件
3. **脚本高效执行**：工具脚本可以通过 bash 执行，无需把完整内容加载进上下文。只有脚本的输出消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在实际读取前不消耗上下文 token

* **文件路径很重要**：代理像导航文件系统一样导航你的技能目录。用正斜杠（`reference/guide.md`），不要用反斜杠
* **文件名要描述性**：用能表明内容的名字：`form_validation_rules.md`，而非 `doc2.md`
* **为发现而组织**：按领域或功能构建目录
  * 好：`reference/finance.md`、`reference/sales.md`
  * 坏：`docs/file1.md`、`docs/file2.md`
* **打包全面的资源**：包含完整的 API 文档、大量示例、大型数据集；在访问前无上下文惩罚
* **确定性操作优先用脚本**：写 `validate_form.py`，而不是让代理生成校验代码
* **让执行意图清楚**：
  * "运行 `analyze_form.py` 来提取字段"（执行）
  * "提取算法见 `analyze_form.py`"（当参考读）
* **测试文件访问模式**：用真实请求测试，验证代理能导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md（概览，指向参考文件）
└── reference/
    ├── finance.md（收入指标）
    ├── sales.md（管线数据）
    └── product.md（用量分析）
```

当用户问收入时，代理读 SKILL.md，看到对 `reference/finance.md` 的引用，并调用 bash 只读那个文件。sales.md 和 product.md 文件留在文件系统上，在需要前消耗零上下文 token。这种基于文件系统的模型正是渐进式披露的基础。代理可以导航并选择性地加载每个任务恰好需要的内容。

关于技术架构的完整细节，见技能概览中的 [How Skills work](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的技能使用 MCP（Model Context Protocol）工具，始终使用完全限定的工具名，以避免"tool not found"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
Use the BigQuery:bigquery_schema tool to retrieve table schemas.
Use the GitHub:create_issue tool to create issues.
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名
* `bigquery_schema` 和 `create_issue` 是那些服务器内的工具名

没有服务器前缀，代理可能找不到工具，尤其是当有多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包可用：

````markdown  theme={null}
**坏示例：假设已安装**：
"Use the pdf library to process the file."

**好示例：明确依赖**：
"Install required package: `pip install pypdf`

Then use it:
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最多 64 字符）和 `description`（最多 1024 字符）字段。完整结构细节见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

让 SKILL.md 正文保持在 500 行以内以获得最佳性能。如果内容超过这个，用前面描述的渐进式披露模式拆分到独立文件。架构细节见 [Skills overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效技能的清单

在分享技能之前，验证：

### 核心质量

* [ ] description 具体并包含关键术语
* [ ] description 同时包含技能做什么以及何时使用
* [ ] SKILL.md 正文在 500 行以内
* [ ] 额外细节在独立文件中（如需要）
* [ ] 没有时效性信息（或在"旧模式"章节中）
* [ ] 全文术语一致
* [ ] 示例具体，不抽象
* [ ] 文件引用只有一层深
* [ ] 恰当地使用渐进式披露
* [ ] 工作流有清晰步骤

### 代码和脚本

* [ ] 脚本解决问题，而非甩给代理
* [ ] 错误处理显式且有帮助
* [ ] 没有"魔法常量"（所有值都被论证）
* [ ] 所需包在指令中列出并验证为可用
* [ ] 脚本有清晰文档
* [ ] 没有 Windows 风格路径（全用正斜杠）
* [ ] 关键操作有校验/验证步骤
* [ ] 质量关键任务包含反馈循环

### 测试

* [ ] 至少创建了三个评估
* [ ] 用 Haiku、Sonnet 和 Opus 测试过
* [ ] 用真实使用场景测试过
* [ ] 纳入了团队反馈（如适用）

## 下一步

<CardGroup cols={2}>
  <Card title="Get started with Agent Skills" icon="rocket" href="https://platform.claude.com/docs/en/agents-and-tools/agent-skills/quickstart">
    创建你的第一个技能
  </Card>

  <Card title="Use Skills in Claude Code" icon="terminal" href="https://code.claude.com/docs/en/skills">
    在 Claude Code 中创建和管理技能
  </Card>

  <Card title="Use Skills with the API" icon="code" href="https://platform.claude.com/docs/en/build-with-claude/skills-guide">
    以编程方式上传和使用技能
  </Card>
</CardGroup>
