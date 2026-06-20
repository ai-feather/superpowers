# 技能指导的正向指令重设计 —— 设计规格

**状态：** Proposed（2026-06-09 SDD review-dispatch 工作的后续；按“一 PR 一问题”规则单独成 PR）
**驱动：** 已测量的证据（2026-06-10）表明技能散文中的一些负向指令会起反作用，而另一些则有效——并且这种差异是可预测的。

## 本规格所归纳的已测量发现

2026-06-10 的微测试（opus，每种措辞 5 次重复，程序化打分；
宿主见下文）测量了指导措辞如何改变 controller 组合的内容：

| 案例 | 措辞 | 结果 |
|---|---|---|
| Dispatch 组合（"don't restate the brief"） | 禁止式 | **4.4** 个 spec 值被重新键入 —— *比无指导更差*（3.6） |
| Dispatch 组合 | 正向配方（"your dispatch should contain: (1)…(5)"） | **3.0，零方差** —— 被采纳 |
| Dispatch 组合 | 配方 + 细节条款（"quote only the fragment…"） | 3.8，噪声大 —— 细节稀释了配方 |
| Test-rerun 指令（"do not ask reviewer to re-run tests"） | 禁止式 | **0/5 违规** —— 工作良好（对照：3/5） |
| Test-rerun 指令 | 正向配方 | 0/5 —— 持平，但更长 |

**原则体系**（用它来分类任何负向指令）：

1. **触发线（tripwire）有效。** 针对具体 token 的短语级自检（"if the
   prompt you are writing contains 'do not flag' … stop"）会可靠触发。
2. **识别表有效。** Red-Flags/rationalization 表在决策时被读取，
   而不是在组合时。
3. **离散指令式禁止有效。** 当模型没有竞争动机去做 Y 时，"Do not
   ask X to do Y" 成立。
4. **组合式禁止会起反作用**，当模型对输出有自己的倾向时
   （例如重述 spec 感觉像是乐于助人的策展）。
   只有正向组合配方能改变这些——而在获胜的配方上添加细节条款
   会让它更差，而不是更好。
5. **平局时取更短的措辞。** Codex 在每个长会话中会重读 SKILL.md
   约 500 次（2026-06-10 实测）；散文长度是真实的成本。

## 审计结果（2026-06-10，全部约 30 个技能 + 提示词模板）

计数：3 个触发线（保留）、14 个识别表（保留）、约 20 个策略门
（保留——"never push without permission" 是策略，不是组合
塑造）、5 个组合式禁止：

| # | 位置 | 处置 |
|---|---|---|
| 1 | `subagent-driven-development/task-reviewer-prompt.md` —— "Cite, don't narrate" | **已排入 PR #1717 批次**：以正向那一半开头（"Your report should point at evidence: file:line for every finding…"），丢弃禁止那一半（死重——正向那一半已存在并承担主要作用） |
| 2 | `subagent-driven-development/SKILL.md` —— "Do not add open-ended directives" | **原样保留**：微测试在 15 个样本中无法引发失败；两个方向都没有证据；短的获胜 |
| 3 | `subagent-driven-development/SKILL.md` —— "Do not ask a reviewer to re-run tests" | **原样保留**：实测 0/5 违规；该禁止还会有效地自我传播进 dispatch |
| 4 | `subagent-driven-development/SKILL.md` —— "do not re-review on top of it" | **已排入 PR #1717 批次**：替换为三元素清单（"Before re-dispatching the reviewer, confirm the fix report contains: the covering tests, the command run, and the output"） |
| 5 | `writing-plans/SKILL.md` —— "No Placeholders" 禁用模式列表 | **本规格的主题** —— 见下文 |

边界情况，与 #5 一起推迟：`task-reviewer-prompt.md` 的 "Don't flag
pre-existing file sizes — focus on what this change contributed"（正向
那一半存在且承重；影响小；如果方便就与 #5 一起测）。

## writing-plans 的改动（推迟项 #5）

### 当前状态

`skills/writing-plans/SKILL.md`，"No Placeholders"：一句正向句子
（"Every step must contain the actual content an engineer needs"）后面
跟着一个六条禁用模式列表（"never write them: 'TBD', 'TODO',
'Add appropriate error handling', 'Write tests for the above', 'Similar to
Task N', …"）。

### 为什么重要以及为什么确实不确定

- 计划是工作流中**最大的生成产物**，而模型确实有竞争动机
  去输出占位符（它们是长度压力下阻力最小的路径）——这正是
  禁止可测量地起反作用的案例的激励结构。
- 但被禁用的条目是**离散的、可识别的 token**——这正是禁止
  可测量地成立的案例的形状。
- **该列表在别处承重：** 技能的 Self-Review 章节引用了它
  （"Placeholder scan: search your plan for red flags — any
  of the patterns from the 'No Placeholders' section above"）。这些 token
  同时充当 review 时的扫描清单，而 review 时的识别
  正是有效的类别。简单换成正向清单会破坏
  该引用并丢弃好的触发线 token。

### 待测变体

- **V0（当前）：** 组合时正向句子 + 禁用列表；
  Self-Review 引用该列表。
- **V1（审查者清单）：** 组合时仅正向配方 ——
  "Before finalizing a step, confirm it has: the literal code to write, a
  runnable command with expected output, types and method names defined
  within this plan, error handling shown explicitly. A step is complete
  when an engineer could implement it without asking any follow-up
  questions." Self-Review 保留一个通用的占位符扫描。
- **V2（按机制重构 —— 预测的赢家）：** 组合时
  只保留 V1 的正向配方；具名模式整体移入
  Self-Review 的占位符扫描步骤，重新定位为识别（"when
  you scan, look for: 'TBD', 'TODO', 'Similar to Task N', …"）。相同
  token，从引发（prime）的类别迁移到检测（detect）的类别。
- **V3（对照）：** 仅正向句子，任何地方都没有列表。

### 微测试设计

- **任务：** opus 从一个故意欠规格化的 spec 写一个 2-3 任务的实现计划
  （欠规格化正是诱发占位符的原因）。
  使用一个 fixture spec：一个规格良好的任务、一个 spec 对其错误
  处理语焉不详的任务、一个与第一个相似的任务（诱发
  "Similar to Task 1"）。
- **采样：** 每个变体 5+ 次重复，默认温度，模型
  `claude-opus-4-8`（实际写计划的模型）。
- **程序化打分**（除注明外越低越好）：
  - 禁用 token 计数：`TBD|TODO|implement later|fill in details|appropriate error handling|handle edge cases|Similar to Task|Write tests for the above`
  - 在步骤改动代码但缺少 fenced 代码块的步骤数
  - 引用了计划输出中任何地方都未定义的类型/函数
  - （越高越好）每个任务带预期输出的可运行命令数
- **V2 的两阶段打分：** 同时测试 Self-Review 那一半——把每个
  生成的计划连同该变体的 Self-Review 章节一起喂回，并测量
  扫描是否真的能捕获植入的占位符（在一个 fixture 计划中插入 2 个已知
  占位符；检测率是指标）。
- **验收：** 仅当某个变体在禁用 token 计数上击败 V0、且
  不损失代码块覆盖率或 self-review 检测率时才采纳它。
  预期成本：总计约 $6-10。

### PR 范围

单独成 PR（writing-plans 是不同的技能；其 "No Placeholders"
列表是被精细调过的内容，贡献者指南要求 eval
证据）。该 PR 必须包含：微测试宿主 + 结果表、
前/后文本，以及 V2 迁移的理由。

## 微测试宿主（方法，以免丢失）

`/tmp/sdd-exp/micro/run-micro.py` 和 `/tmp/sdd-exp/micro2/run-micro2.py`
（2026-06-10；将作为
`docs/superpowers/skills/micro-testing-prompt-guidance.md` + 脚本提交到 superpowers-evals）：

- 每个样本一次 API 调用：system prompt = 置于真实
  上下文中的技能指导变体；user = 一个真实的工作流中场景；
  输出 = 组合出的产物（dispatch prompt、计划、报告）。
- 程序化打分，用 grep 匹配明确无误的标记；**在信任结论前手动
  逐条检查每个匹配**——今晚的一个 “违规”
  其实是 controller 正确地引用了禁止语，而自动
  否定检测把另一个误标了。
- 约 $0.15-0.30/样本，每次迭代几秒，相比之下完整 eval 运行是 $12/50 分钟。
  在这里迭代措辞；仅当
  改动是结构性的时候才在完整运行中确认赢家。
- 始终包含一个无指导的对照——今晚它既揭示了一个
  反作用（重述：禁止比没有指导更差），也揭示了一个有效的
  禁止（test-rerun：对照 3/5 失败 vs 两种措辞都 0/5）。

## 结果：writing-plans 微测试（2026-06-10 运行，在本规格撰写之后）

**已解决 —— 无需改动。** Stage 1（3 任务 spec，无压力）：所有四个变体
包括无指导对照在内的全部 20 个计划中有 0 个
占位符。Stage 1b（10 任务 spec，五个近乎相同的命令诱发
"Similar to Task N"，显式约 2,500 词经济性目标）：40/40
干净——唯一的正则命中是一个 V2 self-review *声明*"no
TBD/TODO ✓"。当前一代 opus 即使在故意压力下也不产生计划占位符，
无论有没有禁用模式列表。
处置：保持 No Placeholders 章节原样（它成本
很小，而反事实不可测量）；不要打开后续
PR。V2 迁移设计保留在此处存档，以防未来模型
一代出现回退。

## 同样明确未丢弃（已测试并已否决，附数据）

记录在此以防任何人在没有新证据的情况下重新提议——完整数字见
2026-06-09 SDD 设计规格的 Cost-iterations 章节：

- **Controller 回合批处理 / 一条消息中并行 tool 调用：**
  controller 每条消息恰好发出一个 tool 调用（在每次
  测量的运行中，有无指导都是 0 条多 tool 消息）。46% 的
  controller 回合是思考/叙述而没有 tool 调用——一个对 prompt
  免疫的下限。
- **通过并行调用实现的流水线化 review：** 同理已死。
- **通过 `run_in_background` 实现的流水线化 review：** 被提供时机制被采纳
  （7/28 dispatch）但在 45 分钟场景上收益低于运行间
  噪声下限（每次 review 仅约 30-60s）；增加了双重
  结果流协调。仅当 review 单独就很长的计划才值得
  重新考虑。
- **追加到获胜配方上的细节条款：** 可测量地降低它们
  （C2：3.8 噪声大 vs C：3.0 一致）。通过重新推导配方来迭代，
  而不是通过追加告诫。
