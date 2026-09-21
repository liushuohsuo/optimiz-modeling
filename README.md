# modeling-solution-optimizer

用于对**已经完成首轮求解**的数学建模项目继续求优。

本 Skill 把现有方案、代码、结果和论文视为 baseline 与可复用资产，通过真实实验比较进行 **REFINE / REPLACE / RETHINK**，并以证据决定是否更新 current best。它不从零解题，也不自行实现正式 writer；Full-flow 可调度目标 MathModel 已有的 evidence、writer 与 QA 能力完成交付。

## 核心流程

Optimization-only 保持现有 optimizer 行为：

```text
GROUND
  ↓
REVIEW
  ↓
BUILD
  ↓
VERIFY
  ↓
UPDATE
  ↺
直到满足停止条件
  ↓
DONE
```

Full-flow 只在冻结 current best 后增加编排层：

```text
GROUND → REVIEW → BUILD → VERIFY → UPDATE
                         ↓
                  FORMAL_REFRESH
                         ↓
              EVIDENCE_AUDIT [subagent]
                         ↓
                 Writing Handoff
                         ↓
                  WRITING [subagent]
                         ↓
                 FINAL_QA [subagent]
                         ↓
                        DONE
```

- **GROUND**：恢复可比较的 baseline、评价口径、数据划分和硬约束。
- **REVIEW**：从题目、代码和真实结果中识别高价值瓶颈。
- **BUILD**：隔离实现候选方案。
- **VERIFY**：先检查 VALID，再比较 VALUE。
- **UPDATE**：对候选作出 `ACCEPT / RETAIN_EVIDENCE / DISCARD`，必要时更新 current best。

默认最多尝试 3 个候选；每个失败候选最多进行 1 次明确执行错误修复。搜索只用于形成候选，不能替代实际验证。

## 适用场景

适用于：

- 已有首轮模型、可运行代码和真实结果，希望继续提升；
- 检查当前方案是否存在更好的局部改进或替代路线；
- 对稳定性、评价协议、数据划分、模型适配或验证不足进行二次检查；
- 在已有方案基础上探索 REFINE、REPLACE 或 RETHINK。

不适用于：

- 从零开始解决一道数学建模题；
- 只有论文文字、却要求宣称计算性能提升；
- 仅做文字润色；
- 用搜索到的方法或论文结果直接代替本项目实验验证。

## 目录

```text
modeling-solution-optimizer/
├── SKILL.md
└── references/
    ├── mathmodel-integration.md
    ├── reasoning-prompts.md
    └── targeted-search.md
```

- `SKILL.md`：核心优化流程与接受规则。
- `mathmodel-integration.md`：与现有 MathModel 项目真实运行、正式证据和写作交接相关的按需集成说明。
- `reasoning-prompts.md`：瓶颈归因不清、考虑换路线或需要反证时使用的条件提示。
- `targeted-search.md`：定向搜索以及如何把搜索结果转成可验证候选。

## 快速使用

把 Skill 安装到目标 Agent 可发现的 skills 目录后，在一个**已经完成首轮求解**的项目中启动 Agent。入口 Prompt 只声明 completion mode；具体状态、Agent、receipt、fallback/阻塞规则都由 Skill 管理，不继续堆到用户 Prompt 中。

### Optimization-only

```md
请执行 `modeling-solution-optimizer`，
本次模式为 Optimization-only。
完整执行优化循环直到满足停止条件。
```

预期行为：

```text
GROUND → REVIEW → BUILD → VERIFY → UPDATE → DONE
```

不得因为项目中存在 writer、QA、历史 Agent 或工作流定义而自动进入正式写作。

### Full-flow

```md
请执行 `modeling-solution-optimizer`，
本次模式为 Full-flow。
完整执行优化与最终论文交付，并严格按照 Skill 定义的
Full-flow orchestration 执行，不得跳过 required delegation stages。
```

预期行为：

```text
optimizer
→ FORMAL_REFRESH
→ real Evidence Auditor subagent
→ filtered Writer handoff
→ real Paper Writer subagent
→ real Final QA subagent
→ DONE
```

Evidence Auditor、Paper Writer 和 Final QA 都必须有真实 native delegation result 才能完成阶段。若运行环境没有真实 delegation 能力，则 Full-flow 在对应阶段阻塞；Main 不得用 sequential single-agent simulation、角色自述或手写 PASS 伪造完成。

### Continuation / resume

Optimization-only 完成后，如果用户随后要求“开始写作”“继续完整流程”“完成最终论文”或“完成论文交付”，视为 Full-flow continuation：

```text
Optimization-only DONE
→ FORMAL_REFRESH
→ Evidence Auditor
→ Paper Writer
→ Final QA
→ DONE
```

此时不重跑已经完成且仍有效的 optimizer，也不允许 Main 直接进入写作。Full-flow 的可恢复状态记录在目标项目：

```text
paper_output/optimization/full_flow_state.json
```

没有真实 auditor / writer / QA delegation receipt，就不能把对应阶段或 `overall` 标记为完成。

## 运行原则

- baseline 是固定参照，current best 只在验证通过后更新。
- 复杂度、新颖性或“大模型判断”本身不构成改进证据。
- 候选必须与 baseline/current best 在可比口径下验证。
- 数据、标签、预处理、划分和评价协议都可能是模型质量的一部分。
- A、B 分别有效不代表 A+B 有效；组合后必须重新验证。
- 有效的负面结果应保留，但不因为被保留就自动进入正式论文。
- 模型诊断不自动等于问题规律；只有证据支持时，才提炼阈值、阶段变化、权衡、适用边界或决策规律。

## 与 MathModel 的关系

本仓库只负责**二次求优**。

当目标项目来自 MathModel 时，Skill 会按需读取 `references/mathmodel-integration.md`，复用目标项目真实的运行、证据和 writer 流程。Optimizer 的 `ACCEPT` 不等于 MathModel 的正式证据、论文或提交已经通过。

Optimization-only 在 UPDATE 后停止。只有 Full-flow 才会在冻结 current best 后继续 authoritative evidence refresh → Evidence Audit → Writing Handoff Filter → Paper Writer → Final QA。

正式写作应由 MathModel 原有 writer 负责；优化日志、失败候选、调试信息、resolved warnings、内部验证 guardrail 和无实质影响的负面证据不应默认作为论文写作输入。RETAIN_EVIDENCE 继续完整保留用于审计，只在影响最终结论、模型选择、适用边界或正式结果解释时进入 writer handoff。

若目标 MathModel writer 已支持 S7 Visual Writing，optimizer 只负责把正式视觉需求和已验证资产交接过去：科学结果图继续属于 `figure_index.json` / S6 evidence，方法、流程、架构、场景、机制等 communication visuals 由 writer 自身契约管理（例如可选的 `writing_visual_manifest.json`）。缺少科学结果图时返回 evidence layer 真实生成，不能由 Writer 或 ImageGen 现场“补证据”；目标 writer 尚未实现该能力时报告 capability gap，不在本仓库复制或模拟 writer。
