# MathModel 按需集成

仅在输入来自 MathModel、需要真实运行/路线变更，或将已验证候选交回正式写作时读取。下述为 MathModel Standard 2.10.0 本地源码核查结果；使用前以实际项目当前入口及 guard 为准。这里不实现执行器、审批器、证据生成器或 writer。

## 输入与 Review 信号

可接受已有首轮方案、代码与结果、草稿或 MathModel 项目作为优化输入。只有正文时可以 REVIEW 与列出缺口，不能凭正文宣称 VALID 或执行正式流程。MathModel 正式输入放 `problem_files/`，preflight 支持 PDF/DOCX/MD/TXT 题面并分类附件；具体数据以 input_manifest 与 data_plan 为准，不假定任意附件均可运行。

从项目根读取原始题面、`paper_output/step1/problem_contract.json`、`paper_output/step1/mathematical_abstraction.json`、`paper_output/plan/` 中的 `model_route.json`、`model_route_approval.json`、`route_review.json`、`observability_audit.json`、`data_plan.json`和运行结果；只读当前阶段需要的文件。核对第一轮的命令、环境、官方目标、硬约束、评价口径、划分和 baseline。结果缺失或 stale 是证据缺口，不等于方法差。

若已有 `paper_output/qa/optimization_decision.json`，核对其输入仍适用后读取目标达成、权衡与 claim_restrictions 作为 REVIEW 信号。它不是跨候选比较器，`SUFFICIENT` 不阻止有依据的新探索，也不能替代本 Skill 的 VALID/VALUE。源码依据：`quality-assurance-auditor/scripts/optimization_decision.py:12–23,84–145`（相对 `.agents/skills/`）。

Review 重点：目标是否被代理偷换、关键变量能否观测、语义机制是否影响输出、组件是否适配数据、代理收益是否损害参考指标、验证是否支持结论、计算瓶颈在哪里。局部实现问题支持 REFINE；部件不适配支持 REPLACE；目标表达、假设或结构本身受限支持 RETHINK。大改本身不构成拒绝理由。

## cwd、运行和隔离

以下命令均是后续使用示例，本 Skill 制作阶段不执行正式流程。`PROJECT_ROOT` 是实际竞赛项目或候选物理副本根，不能是 MathModel 源码包根。preflight、guard、初始化、审批及 post-full 依赖 cwd；runner 从自身所在 `paper_output` 定位根，q 模块 cwd 为 `paper_output/code/modeling/`。

```bash
cd "$PROJECT_ROOT"
python .agents/skills/paper-workflow-orchestrator/scripts/preflight_check.py
python .agents/skills/paper-workflow-orchestrator/scripts/workflow_guard.py --status
```

preflight/guard 会写项目报告，并非纯只读检查。 实际副本迁移可能改变 `input_manifest.json` 的 `root`，即使输入 entries 完全相同，也会使原 observability/loader 的哈希绑定过期（test5 第二阶段实测）。冻结原 manifest，记录差异；计算复现与正式准入分开报告。不得回填旧根、续签旧哈希或把包内“当前对话已批准”文字当成本次新审批。没有受支持的证据迁移与有效审批时，正式交接标为阻塞，不能由旧 S8 PASS 推定可继续。按 `qa/workflow_guard_report.json` 的推荐回到首个未通过阶段。首轮已有人工实现时不要无条件重新 initialize。

```bash
python .agents/skills/model-code-and-result-generator/scripts/build_result_contracts.py --mode implement
python .agents/skills/model-code-and-result-generator/scripts/validate_model_implementation.py
python paper_output/code/modeling/run_modeling.py --execution-kind pilot
```

缺少初始化契约时才按原 S4 调用 `--mode initialize`，然后实现题目模块和语义测试。生成器退出零不算真实求解成功。

每个候选使用原项目之外的独立物理副本；不让代码、结果、表格或缓存以共享软链接回写 baseline/current best。检查自定义模块的绝对路径和写入行为。原 runner 为每次运行建立 `paper_output/runs/<kind>/<run-id>/`，向子进程传 `MATHMODEL_RUN_OUTPUT_DIR` 及分组目录变量，原 result_contract_io 支持此约定。自定义代码可能绕过约定，故 runner 的目录隔离不等于任意代码沙箱。Full 会晋级该副本的正式输出，所以 Full 也必须只在候选根运行。

源码依据：`model-code-and-result-generator/scripts/build_result_contracts.py:350–386,930–970,1351–1460`（均相对 MathModel 的 `.agents/skills/`）。

## 路线变更、审批和预算

冻结由 `model_route_approval.json` 对路线与上下游合同的哈希绑定实现；没有本 Skill 可用的独立解冻快捷入口。仅修改实现且不改变批准 method/objective/components/approximation 与合同，可继续 S4；改变路线、目标、约束、成功标准或实现合同则回 S2，更新候选/Critic/输入适配与 alignment，取得用户真实选择。

未审批的新路线可以在候选根做明确标注 exploratory 的小实验，命令须依据项目实际代码，不能把 exploratory 包装为正式 Pilot 支持。正式登记接受的是当前已批准路线及实际保留 Pilot manifest。禁止伪造 `approved_by` 或把 optimizer 的 ACCEPT 当成人工路线审批。

```bash
python .agents/skills/paper-workflow-orchestrator/scripts/alignment_gate.py
# Critic COMPLETE、alignment/observability/domain/method map 新鲜 PASS，且用户明确选择后：
python .agents/skills/paper-workflow-orchestrator/scripts/approve_model_route.py --selection Q1=A --approved-by "实际批准人"
```

多问题项目逐一追加实际选择，不能照抄 Q1=A。observability/domain 等更新按 orchestrator 当前指示完成。批准后回原 S3/S4 及 Pilot；由 `record_pilot.py` 登记，必填参数为 `--question-id`、`--route-id`、`--status`、`--cause`、`--reason`、`--criterion`、`--goal-status`、`--goal-metric`、`--goal-direction`、`--constraints-satisfied`、`--pilot-run-manifest-ref`。取值与实际结果一致，运行前读该脚本 `--help`；不提供固定 supported 示例诱导填成功。

optimizer 默认三候选与 MathModel 每问三条不同路线的正式探索计数是不同账本；现有正式次数不得重置，采用两者中更严格的剩余额度，用户更小预算优先。失败执行最多一次明确错误修复的 optimizer 规则不扩展 MathModel 的路线次数。已有批准不覆盖新的路线选择。

源码依据：selector `SKILL.md:42–94`；orchestrator `scripts/approve_model_route.py:97–221`、`scripts/workflow_guard.py:491–520`；model generator `scripts/record_pilot.py:45–123`。

## Full-flow orchestration capability mapping

本文件只把 Full-flow 阶段映射到目标 MathModel 项目**已经存在**的能力；orchestration 负责调度和检查 completion receipt，不复制或替代 runner、证据生成器、writer、QA 或它们的门禁逻辑。具体命令、参数和前置条件仍以目标项目当前源码为准。

| Full-flow 阶段 | 复用的目标项目能力 | 编排边界 |
| --- | --- | --- |
| `FORMAL_REFRESH` | authoritative full run、`run_post_full.py`、official `evidence_gate.py`、`workflow_guard.py` 及现有 evidence/assets 刷新链 | 只调度现有正式入口；不得手工补 PASS、复制旧 manifest 或另写 evidence pipeline。 |
| `EVIDENCE_AUDIT` | 现有 `quality-assurance-auditor`、evidence/claim/binding/constraint 检查 | 必须由 Evidence Auditor 子 Agent 在独立上下文调用；只返回 findings/status 给主 optimizer，不自行路由。 |
| `WRITING` | 现有 `paper-formal-writer` 及其 outline/authoring/validation/compile 能力 | 必须由 Paper Writer 子 Agent 使用过滤后的 handoff 调用；不得复制 writer 或让 optimizer 直接代写来完成阶段。 |
| `FINAL_QA` | 现有 QA、claim/format/render、视觉检查与 delivery 检查 | 必须由 Final QA 子 Agent独立执行；不得重做模型选择或修改科学结果。 |

原 MathModel 的 S0–S8、runner、evidence pipeline、writer 和 QA 的内部阶段继续保持各自权威。Full-flow state 只记录“哪个现有能力已被实际执行、由哪个 delegation 返回、结果是什么”，不把这些能力重新实现成新的通用 workflow engine。

Delegation receipt 不绑定某个 runtime 的固定字段名：记录运行时实际提供的 delegation identity，并用可复核的 `result_locator` 指向返回结果；没有真实返回或无法定位 provenance 时，不得把阶段写成完成。

## 正式证据刷新

ACCEPT 只表示隔离比较中候选值得采纳，正式项目状态仍由 MathModel 判定。在有支持的真实 Pilot 且登记新鲜后：

```bash
python paper_output/code/modeling/run_modeling.py --execution-kind full
python .agents/skills/paper-workflow-orchestrator/scripts/run_post_full.py
python .agents/skills/quality-assurance-auditor/scripts/evidence_gate.py --mode official
python .agents/skills/paper-workflow-orchestrator/scripts/workflow_guard.py --status
```

runner 本身不能替代所有流程前置检查。Full 必须包含真实 V/V/UQ 与语义证据；若代理目标需要，必须保留参考指标比较。post-full 校验 sealed Full，运行原 decision/fidelity/reports、刷新 paper_assets 并生成正式结果图。不要修改 sealed run、复制旧 manifest、只拼接局部结果或手工改 PASS。上游代码、输入、路线或配置变化后重新执行受影响流程；组合候选也必须重新运行，分别有效不推出组合有效。

负面敏感性/失败场景保留为 RETAIN_EVIDENCE；不为了获得正文结论删掉坏结果。RETAIN_EVIDENCE 是否进入 writer 由 Writing Handoff Filter 决定：只有实质影响最终结论、模型选择、适用边界或正式结果解释的证据才传播，其他内容继续留在审计记录中。`GOAL_PARTIAL/PARTIAL_MET` 仅在硬证据通过且限制明确时进入写作，不是全面目标实现。源码依据：orchestrator `scripts/run_post_full.py:46–78,135–176`、`SKILL.md` 的 S6 段。

## 已知接入限制与失败处置

以下限制来自 2026-09-17/18 的 Standard 2.10.0 test5 路径及其诊断记录，不推定其他版本必然相同。使用时核对目标项目源码；测试副本补丁不等于 canonical upstream 已支持。

- 同路线 Pilot 重登记在本次测试副本中增加了 `--refresh-existing`，不能假设原包有该参数，也不要自动复制测试补丁。
- 本次 S6 诊断涉及将特征代理误判为目标代理、metric_name/metric_id 对应、文本条件不可判定的问题。需要独立核查和上游修复，optimizer 不手改门禁来获得 PASS。
- post-full 中断后，下游报告可能仍绑定旧 Full；旧报告存在不代表证据已刷新。
- 实际 RMSE/MAE 扰动排序负面结果仍是有效证据；修复代理检查的适用范围不等于消除该负面发现。

门禁失败先记录失败项和诊断证据，再按原因处理：

| 原因 | 对优化接受的影响 | 正式交接 |
| --- | --- | --- |
| 已确认的接口、绑定或报告过期 | 不自动撤销已有验证范围内的收益；按真实来源修复并重跑受影响检查 | 仍阻塞，直至当前门禁通过 |
| 硬约束、泄漏或语义错误 | 撤回受影响范围及依赖它的组合；其他局部接受须有独立性证据 | 受影响证据无效，不能继续交接 |
| 原因不明 | 保留证据，标注待查，暂停扩大接受范围 | 保持阻塞，不预判为接口故障 |

分别记录优化接受、正式证据和论文交付，不合成一个模糊的成功状态。修复接口不恢复因实质错误撤回的接受；恢复前仍须针对实际采纳版本完成 VALID/VALUE。

## Writing Handoff Filter

Full-flow 中，Evidence Auditor PASS 后由主 `modeling-solution-optimizer` 执行 handoff 过滤；这里不复制 auditor、writer 或 QA 的实现。三个子 Agent 应优先调用目标 MathModel 项目已有的 evidence、writer、QA/guard 能力，并以目标项目当前源码和正式入口为准。子 Agent 只返回 findings/status 给主 optimizer，不自行路由或调用下一个 specialist。

Writer 默认只接收以下白名单：

```text
accepted final modeling logic
authoritative final results
required problem outputs
accepted figures/tables
supported problem-level findings
material limitations
formal references
```

默认不传：

```text
optimization_log 全文
DISCARD candidates
debug / repair history
resolved warnings
internal guardrails
unsupported hypotheses
无实质影响的 negative evidence
```

这些内容仍完整保留用于审计。若 Auditor 或 Writer 对某一具体 claim、数字、图表或限制提出可定位的证据疑点，主 optimizer 可以按需回溯对应记录并只补充解决该疑点所需的最小上下文；不得因此把完整 optimization history 自动暴露给 writer。

## S7 Visual Writing 兼容契约

该契约只描述 optimizer 与目标 MathModel writer 的交接边界，不在本仓库实现 renderer、ImageGen、visual validator 或新的 stage engine。是否具备这些能力，以目标项目当前 `paper-formal-writer` 源码和门禁为准；目标 writer 未实现时不得由 optimizer 伪造 `writing_visual_manifest.json`、手写 PASS 或复制一套 writer 来绕过能力缺口。

当目标 MathModel 已提供 Visual Writing Pass 时，正式视觉资产必须区分两条权威链：

```text
scientific evidence figures
→ result_evidence / data_exploration / sensitivity_evidence
→ 由模型、数据或正式计算产生
→ figure_index.json
→ S6 evidence

S7 writing visuals
→ method_flow / algorithm_flow / method_architecture / validation_protocol
→ scenario_schematic / mechanism_schematic / conceptual_overview
→ writing_visual_manifest.json
→ S7 writing asset
```

两类资产不得互相替代。S7 不修改 `figure_index.json`，也不把 writing visual 纳入 S6 evidence gate。若目标 writer 使用可选 `writing_visual_manifest.json`，其新增、修改或 hash 变化只应使 S7/S8 对应状态失效；S6 仍以原 scientific evidence chain 判定。

推荐 renderer 路由保持单向：

```text
result_evidence / data_exploration / sensitivity_evidence
→ existing evidence/data plotting

method_flow / algorithm_flow / method_architecture / validation_protocol
→ deterministic renderer

scenario_schematic / mechanism_schematic / conceptual_overview
→ Codex/native image generation
```

Writer 不得为了“有图”而补图。只有视觉资产能够降低理解成本、解释关键方法/机制/场景、支撑正文论证或满足题目/正式写作要求时才应生成。所有新图必须检查真实渲染结果；deterministic source parse PASS 或 ImageGen prompt 成功都不等于最终图 PASS。

如果 Writer 发现缺少的是科学结果图，例如正式敏感性曲线、结果比较图或数据证据图，不得在 S7 现场绘制替代品。返回：

```text
status = EVIDENCE_BLOCKER
blocker = EVIDENCE_VISUAL_GAP
```

主 optimizer 将其路由回 FORMAL_REFRESH / 对应 evidence layer，由真实模型/数据可视化生成图、更新 `figure_index.json`、重跑受影响 S6，再回 WRITING。只有方法图、流程图、架构/验证协议图和场景/机制/概念示意图属于 S7 writing visual。

Writing Handoff Filter 对视觉资产采用最小白名单：传递 accepted evidence figures、已注册且正文确有需要的 writing visuals、对应用途与 authoritative locator；不把未采用草图、失败生成记录或视觉修复历史默认交给 Writer。Final QA 必须核对最终正文引用、实际文件/hash、DOCX/PDF 中真实插入与可读性；不能仅核对 source、prompt 或 manifest 字段。

在目标 writer 已实现该能力时，Visual Writing Pass 位于 assembled audit 之后、global revision / final audit 之前。若目标 writer 尚未实现，则继续其当前 authoring 行为并明确报告 capability gap；optimizer 不新增 S6.5/S7.5，也不自行改变 MathModel 的 S0–S8 stage graph。

## writer 与编译交接

论文修订需使摘要与完整方案一致，由问题特征解释模型选择，讲清公式的目的、变量和作用；题目要求的实际答案应能定位，结果解释须有证据，小问之间的依赖应清楚，创新和局限须对应实际修改、验证及适用边界。修订时双向核对：

```text
题目要求 → 实际结果 → 正文或附件
论文重要结论 → 对应证据
代码中的数据、模型、协议和单位 ↔ 论文描述
```

可以引用 baseline、消融和负面实验，但须区分其角色、来源、版本与评价口径，不能将对照冒充最终方案或混用不同口径的数字。实际答案已有而正文仅列性能指标等表达缺失交给 writer 补齐；证据不足则返回验证或收紧结论，不以写作要求为由编造机制解释。以上内容与证据要求同样适用于下述已授权直接修订；不改变各自的授权和门禁边界。

将 Writing Handoff Filter 产出的白名单内容交给唯一正式 writer。负面证据、限制和待验证项不再默认全量传递；只有 filter 判定为 material 的部分进入正式写作输入。正式数字遵从 `metrics.json` 的唯一 metric_id，图表遵从现有索引与 stable ID。先通过 writer 门禁，再按它的当前计划操作：

```bash
python .agents/skills/paper-workflow-orchestrator/scripts/workflow_guard.py --skill paper-formal-writer
python .agents/skills/paper-formal-writer/scripts/build_paper_outline.py
python .agents/skills/paper-formal-writer/scripts/prepare_authoring.py --mode auto
```

上述命令有前后依赖：writer guard 未通过就停止正式交接，不继续 outline/authoring/编译，并报告阻塞原因和受影响的下游产物。

实际撰写后逐节 `validate_authoring.py --section <实际section-id>`；通过后 `assemble_sections.py`、`validate_authoring.py --assembled`，按报告修改或提升到正式源稿，再 `validate_authoring.py --final`。不要在无正文时顺序空跑审计命令。

```bash
python .agents/skills/paper-formal-writer/scripts/compile_paper.py --project-root . --editable-assets
# 完成逐页视觉检查及原 delivery 所需记录后：
python .agents/skills/paper-formal-writer/scripts/accept_delivery.py --project-root .
```

编译调用原 DOCX/claim/format/render 链路。执行状态、evidence、goal、paper 与 submission_ready 分别报告；旧 AI 模板或缺少视觉证据不能靠编译退出零宣称交付。源码依据：writer `SKILL.md:9–90`、`scripts/compile_paper.py:13–79`、`scripts/accept_delivery.py:120`。

证据、计划、章节或最终源稿变化会使下游旧 PASS 失效；只交付用户要求的修订范围。语义不变的指标/图表沿用 ID，语义变化登记新 ID，正文通过 `[[VALUE:...]]`、`[[FIGURE:...]]`、`[[TABLE:...]]`、`[[REF:...]]` 绑定源数据与对象；同步受影响章节、摘要、结论、图题及交叉引用。适用赛事与年份的官方要求优先，writer 的 CUMCM 结构和项目默认页数/篇幅不是华为杯规则；无法适配的硬性要求明确报告，不能改 guard 或伪造通过。

按需复用 writer `references/section-expansion-rules.md` 与 `figure-table-writing-rules.md` 的论证顺序、公式意义、图表解释；selector Critic 的适配性和风险审视用于 REVIEW。不要复制固定六条假设、固定句数/篇幅、奖项暗示或“白璧微瑕”式淡化缺点。micro-unit 只在原 writer 指定局部修复时使用。

## 用户授权的直接修订

正式交接阻塞且用户明确要求直接修订论文时，可以制作独立可审阅版本；已有明确授权直接沿用，无需重复确认。没有该授权则报告阻塞，不自动切换。输出放在独立交付目录，不覆盖或冒充正式流程产物。

Direct revision 是独立 review artifact，**MUST NOT** 满足 Full-flow 的 `WRITING` 或 `FINAL_QA` completion，也 **MUST NOT** 更新 `full_flow_state.json` 中这些阶段为完成或推动 `overall` 向 `DONE` 前进。只有正式 delegation + 对应正式返回合同可以推进 Full-flow state。

数字和图表绑定已识别的真实输出，同步检查受影响章节、摘要、结论及图表，检查全部渲染页面，并保留内容与视觉检查记录；检查未完成时只报告未完成稿。只表述已验证范围，未决结论写明限制。在独立版本的交付说明中标明仍未通过的正式门禁，不继承 S6/S8 或 submission_ready，也不称其由正式 MathModel 链路生成。该分支不豁免模型有效性检查，不把被撤回的结论重新写成事实。

## 当前验证范围（2026-09-18）

test5 审计结论：Functional status = **E2E_PARTIAL**；Optimization outcome = **IMPROVED**（同口径开发集范围）。正式 S6/writer 未通过；用户授权的直接论文修订及视觉检查已完成，不能作为正式链路通过证据。曾提前整体 ACCEPT，之后撤回并保留记录；本次指令修订针对该失败模式，修订后的真实 E2E 待重新运行。真实 RETHINK、外部搜索到候选运行的完整 E2E 尚未覆盖。
