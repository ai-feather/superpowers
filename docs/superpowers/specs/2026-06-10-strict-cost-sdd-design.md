# Strict-Cost SDD —— 设计规格

**状态：** 提议的实验阶梯（非实现）。每一级只在其门控证据齐全时才交付；任何门控失败则中止该级。
**目标：** 最小化每次 plan 执行的美元开销。墙上时间不受约束；token 数量仅作为成本驱动因素才重要。
**硬不变量：** 质量。具体而言：`sdd-quality-reviewer-catches-
planted-defect` 在 **N=5 次运行**（不是 1 —— 单次运行门控是本次战役最弱的方法论）上的通过率，`sdd-rejects-extra-features` 通过，所有端到端场景通过，与当前配置的盲测 A/B 交付物持平。任何质量回退都会毙掉该级，绝不妥协。

## 钱花在哪里（2026-06-10 最终配置，go-fractals，约 $13/次运行）

| 组件 | $ | 驱动因素 |
|---|---|---|
| Controller（session 模型，opus） | ~6-7 | ~150 轮 × 常驻上下文；prompt 无关的轮次下限（46% thinking/narration） |
| Implementers（sonnet，10-13 次派发） | ~5-6 | 真正的活儿；每个 ~25 轮；每个约 13 次编辑前探索调用 |
| Task reviewers（sonnet，10 个） | ~1-1.5 | 带包时每个 3-9 轮 |
| 最终评审 + 修复 | ~1 | 带分支包 6 轮 |

审查循环次数（每次运行 2-4 次）是运行之间最大的成本方差来源；这些循环主要由 implementer 错误地化解了计划歧义而引起。

## 判断力护栏（与质量同为不变量）

**把机械工作变便宜，绝不要把判断力变便宜。** 每一级必须列举它把哪些决策移到了更便宜的模型上，并展示每一个都是*机械性的* —— 确定性、可脚本化、或事后可廉价验证。判断力留在最高档或留给你的搭档。SDD 中的判断力要点，明确列出：

- **BLOCKED / NEEDS_CONTEXT 处理** —— 诊断子代理为何卡住并选择对策
- **⚠️ "cannot verify from diff" 的解决** —— controller 跨任务上下文裁决
- **派发策划** —— 消歧与任务边界划分（测量证明其承重：Task 5 的 gradient-direction 备注避免了一次错误实现）
- **评审结论与严重性校准** —— 何为 Important 与 Minor
- **审查循环裁决** —— 判定某条发现为误报
- **上报给你的搭档的识别** —— 意识到计划本身就是错的

任何一级如果要把上述任一项移到更便宜的模型，必须要么 (a) 重构使该决策由昂贵模型在 plan 阶段一次性做出，(b) 添加一条显式上报规则在执行时把它路由回去，要么 (c) 死。"便宜模型通常搞对"不是验收证据 —— 判断力失败是罕见事件、高爆炸半径、且基本对 pass/fail 门控不可见，这也是为何下面每一次降档都附带一次判断力审计（在门控运行中对每个判断力要点做 session-resume 讯问，对比昂贵-controller 基线），外加 N=5 场景门控。

## 论点护栏

SDD 的论点：**每个任务一个全新子代理，配以精心策划的上下文，逐任务设门控。** 下面的各级必须保留这一点。派发时任务批量处理（一次 implementer 派发处理多个 plan 任务）**与论点相悖** —— 它污染了"全新上下文"属性并使门控变粗 —— 因此被刻意排除在阶梯之外。达到相同派发经济性的、与论点兼容的路径是 plan 阶段任务尺寸合理化（L1）：如果计划定义了更少、尺寸更合适的任务，SDD 仍然为每个任务运行一个全新子代理。

## 阶梯（按预期 $/杠杆 顺序）

### L1 —— 计划侧清晰度（writing-plans 变更；估计 −$1.5-3/次运行，外加方差降低）

**状态 2026-06-11（最终）：** 端到端测试了 elicitation；主张被重新归因。微测试：constraints 头与 Interfaces 块被确定性地引导出来（0→5/5，任务的 0→100%，精确值）；尺寸合理化是温和的且依赖规模（svelte 规模下 9.4→8.4 个任务，fractals 规模下无可动）。完整运行：一份 elicited 计划以 $6.34/$8.49 执行 —— 但 no-guidance 对照组（opus 计划、完整代码）命中 $7.59/$7.73，落在该范围内。**成本收益属于 opus 撰写的完整代码计划；此前所有数字使用的手写 prose fixture 计划不具代表性，且执行成本约高 2×。** 该指导拥有的是保真度与方差：确定性 constraints 传播（那次 elicited-run 的唯一修复是一次 version-floor 捕获）、精确的跨任务接口、1 对 2-4 的修复波（对照组计划在两次运行中都交付了一个真实的 Sierpinski bug，必须修复）。writing-plans PR 基于这些理由主张，而非美元。草稿在 /tmp/sdd-exp/writing-plans-l1（分支 writing-plans-crisp）。

计划位于每项成本的上游：任务数决定派发数；计划歧义决定审查循环数；计划完整度决定 implementer 探索量。当前 writing-plans 优化的是 implementer 成功率，而非执行经济性。待测变更：

1. **任务尺寸合理化指导。** 今天的计划会产生小到 "create .gitignore" 的任务 —— 每个都花费一整轮派发 + 审查循环（~$0.60-1.00 固定开销）。添加："A task is the smallest unit that carries its own test cycle and is worth a fresh reviewer's gate. Merge setup/config steps into the task that needs them; split only at boundaries where a reviewer could meaningfully reject." Fractals 的计划会从 10 个任务降到约 7 个。验证：派发数下降、门控保持、审查粒度仍能抓到植入缺陷。
2. **结构化的 `## Global Constraints` 节**置于计划头（version floor、命名/复制规则、平台要求）。今天这些位于 design.md 的散文里，只有在 controller 记得粘贴时才到达评审者（一次 `go 1.26.1` floor 违规就这样被交付了，因为没人记得）。固定标题使它们可机械提取 —— `task-brief` 可以把这一节自动追加到每条 brief（一个小脚本改动），完全移除一项 controller 职责。
3. **每个任务一行 `Interfaces:`**（消费/产出、精确签名）。controller 当前每次派发都要重新推导跨任务接口（它主要的合法"复述"），而 implementer 要花约 13 次工具调用重新发现上下文。计划者已经知道接口；每个任务一行就把这项工作搬到它只需做一次的地方。
4. **每个任务的模型档位建议**，由计划者给出（"mechanical / standard / judgment"）。计划者拥有 controller 当前每次派发都要重做的 Model Selection 决策所需的最佳信息；controller 保留覆盖权。

验证：对计划者输出形态做微测试（按 instruction-design 原则的 recipe 式），然后做完整运行。注意 2026-06-10 的结果：计划*占位符*无法从当前 opus 引导出来 —— 这些变更针对的是经济性与歧义，而非占位符卫生。

### L2 —— Controller 档位（估计 −$4-5/次运行；最大的单项杠杆，门控最严）

**状态 2026-06-11（最终）：** 如预注册那样 —— 死在门控上，且解剖学结论有用。侦察结果是正面的（$6.68/$8.05，n=2，机械工作干净）。完整电池拆分了判断力面：新的 `sdd-escalates-broken-plan` 场景（显式计划自相矛盾；你的搭档从不主动提及）在 sonnet 上 **5/5 通过**（$1.02-1.37/次运行；opus 基线 2/2） —— 显式冲突会被上报。但植入缺陷电池决定性失败：在 sonnet controller 下，逐任务质量门控坍塌为计划合规辩护（"no assertion, as required" 被列在 Strengths 下），缺陷在 4/5 次运行中被交付（确定性检查），只有档位锁定的 opus 最终评审者才抓到它 —— 而同一批 sonnet 档位评审者在 opus controller 下 5/5 全部标出。便宜的 controller 能处理显式上报；它们会吸收隐式的"权威 vs 质量"裁决。一个可能的 L2b（离散规则："a reviewer finding that conflicts with the plan's text is the human's decision — escalate it"）会把失败的判断力路由进那条已站得住脚的上报行为。

**L2b 已于 2026-06-11 测试（E35/E36，evals `docs/experiments/2026-06-11-build-loop-autoresearch.md`）：改善了 opus 技术栈，没能救活 sonnet 那一级。** 两条规则：一条评审者 tripwire（计划强制要求的缺陷就是一条发现 —— Important，标注为 plan-mandated；由你的搭档决定）和一条 controller 上报规则（plan-mandated 发现像任何计划矛盾一样交给你的搭档）。在冻结的 sonnet 组合输入上做微测试：0/6 → 6/6 标注的发现。完整电池：opus controller 2/2 把规则内化、抓到自己评审者的遗漏并自我描述为后盾、为一次获授权的修复而上报（4241 的临时行为被结构化）；上报健全性 2/2 未被破坏。Sonnet controller：1/5 完整通过 —— 复述把 tripwire 从派发中丢弃（2/5 传递成功）、仅传递不足以在实时中触发（评审者工具读取中的读一次稀释；放置位置被证伪为变量），且没有 sonnet controller 表现出后盾行为；1/5 交付了缺陷。L2b 规则是 opus 技术栈的一个候选提交。一个针对 sonnet 级的未来 L2c 会把 SKILL.md 的 constraints-recipe（sonnet 唯一会逐字传递的通道）与一个 plan-mandated 发现的强制输出格式槽位（骨架在每次观察到的复述中都存活，并在组合时被查阅）配对；未测。原侦察笔记如下。

**侦察（已作废）：**
Sonnet-controller 运行（claude-sonnet coding-agent）：所有门控通过，**$6.68 与 $8.05** / 31-41 分钟（combo band $11.67-14.84），token 在 combo band 之内 —— 没有便宜-controller 的轮次膨胀。26/26 与 31/31 次派发 model-explicit，且 haiku 档位分层比 opus controller 更重（也合理）；审查循环、逐任务 Important→修复→复审、omnibus-fixer 规则在两次运行中都得到遵循；run-1 的 controller 在复审前抓到了 fixer 的副作用（`go mod tidy` 移除 cobra） —— 真实的裁决，而非静默吸收。但两次运行都没有出现 BLOCKED/⚠️ 事件（上报点从未被压测），且最终评审在 sonnet 而非最强档位上跑。下面的 N=5 质量门控 + 完整判断力审计在任何技能变更之前仍是强制的。

controller 占了一半开销，仅因为它继承 session 模型。它的轮次下限与 prompt 无关，所以杠杆是每轮费率 —— 但 controller 也是大多数判断力要点所在之处，因此这一级是判断力优先的：

1. **主形态 —— 判断力上移、机械工作变便宜：** 昂贵模型在 plan 阶段做判断力密集的工作（L1 的 Interfaces 行、消歧、逐任务 constraints —— 即把派发策划预先写入计划）。中档执行 session 随后运行一个真正机械的循环：抽取 brief、派发、跑脚本、路由结论。技能中的显式上报规则：遇到 BLOCKED、任何 ⚠️ 项、疑似误报、或任何计划尚未回答的内容时，便宜 controller 停下并上报（给你的搭档，或一次全新的昂贵模型咨询派发）—— 它绝不单独裁决判断力。
2. **超出标准 N=5 的门控：** 一次判断力审计 —— 对门控运行中每一次 BLOCKED/⚠️/裁决事件做 session-resume 讯问，并对照 opus-controller 基线对同类事件的处理打分；任何被静默吸收的判断力决定（便宜 controller 本该上报却自己裁决）都会毙掉该级，不论场景结论如何。
3. **保留你的搭档的权威：** 技能只推荐执行 session 档位，绝不强制。

来自本次战役的告诫：便宜模型的轮次膨胀是在多步*工作*上测得的，而非派发循环；中档 controller 是否能稳定在 ~150 轮是本实验要确定的事情之一。

### L3 —— 评审者档位（估计 −$0.7-1/次运行；最可能因判断力护栏而死的级）

**状态 2026-06-11：** 如预注册那样 —— 死。植入缺陷 ×5 并强制 haiku 作为任务评审者：2 通过 / 1 不确定 / 2 失败（基线 5/5）；逐任务 haiku 在正确严重性下干净地标注了 10 个植入缺陷中的 0 个 —— 1 个被发现但降级且理由正是被禁止的那条，9 个漏掉或被合理化（DRY 被表扬为 YAGNI；assert-nothing 测试被称为 plan-compliant）。便宜评审者是通过*为缺陷辩护*而失败的；通过的运行只靠 controller 冗余或最终评审存活下来。记录在实验日志 Batch A-E。不要在没有结构性不同设计的情况下重新提议。

带包评审者在机械意义上接近单步（平静时 3 轮 / 1 次 Read），这使中档下限的原始轮次膨胀理由失效 —— 但评审彻头彻尾是判断力：严重性校准、规格结论、知道不该标什么。机械性的便宜不等于决策的机械性。只有在完整判断力电池下测试 haiku-with-package：植入缺陷 ×5、一次严重性校准检查（植入 Minor-vs-Important 对；校准失误毙掉该级），并在该档位上重新测量逃生舱方差。预期：这一级会死，而这是一个好结果 —— 它把"我们怀疑便宜评审者很差"转化为有据可查的证据。

### L4 —— 常驻上下文节食（估计 −$0.5-1/次运行）

- `task-brief --list` 模式：controller 只读任务标题 + Global Constraints，从不读完整计划（计划正文已通过 brief 交付）。
- 报告从 15 行裁到 8 行。
- SKILL.md 压缩轮次（本周新增的每一节都按 composition-recipe 密度重新论证；Codex 在每次长 session 上为约 10k 字符 × 约 500 次重读付费）。

### L5 —— 重新争议项（明确标注，维护者否决或反论点）

为完整性记录；每一项都需要 Jesse 明确反转才能做任何实验：
- **限定范围复审**（校验修复 + 回归扫描，而非完整复审）：2026-06-09 否决；至多约值 $0.50/次运行。
- **派发时任务批量处理**：反论点（见护栏）。L1.1 是受认可的形式。

## 预算与排序

L1 与 L2.1 相互独立 —— 先两者并行（约 $80：微测试 + 2×5 次运行门控 + A/B）。L3 在 L2 定下 controller 之后做（评审者行为依赖派发质量；约 $25 —— 植入缺陷运行每次 $2-3）。L4 最后做（便宜，但在技术栈定好后再过一次门控；约 $30）。整条阶梯带诚实 N=5 门控合计 ≲ $150。若每一级都活过门控的预期终态：**fractals 上 $5-7/次运行（从 $12-15）**；若判断力敏感的各级（L2 超出主形态、L3）如期而死，则 **$8-10/次运行** —— 诚实目标，因为护栏按构造把判断力定价在美元之上。

## 与现有工作的关系

建立在 2026-06-09 task-scoped review dispatch 设计（PR #1717）和 2026-06-10 实验战役（evals `docs/experiments/2026-06-10-sdd-cost-experiments.md` —— 新增级别前请查阅负面结果节；turn-discipline 与 parallel-call 机制已死）之上。任何新散文的指令措辞遵循 positive-instruction 原则规格，并在完整运行前做微测试。L1 是一次 writing-plans 变更 → 独立 PR 附 eval 证据；L2-L4 是 SDD 变更 → 独立 PR。
