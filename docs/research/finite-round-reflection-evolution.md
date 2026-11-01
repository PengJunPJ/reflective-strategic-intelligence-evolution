# 有限轮次反思演进机制研究

> 对应议题：#7。研究日期：2026-11-01。
> 边界：只研究抽象决策与软件安全机制，不涉及真实武器参数或作战部署。

## 结论

建议把有限轮次的**反思演进**实现成一个“**变化触发—有界操作—小预算竞争—守门验证—提交或回滚**”的受控外环，而不是让大模型反复直接重做目标分配。

1. **检测**：同时监测“输入特征变化”和“求解行为退化”。用规则阈值捕捉明确结构变化，用 ADWIN 类可控显著性的变化检测器处理连续信号，避免把单次噪声当成变化 [S1][S2]。
2. **候选**：大模型只能从类型化、有参数边界的操作目录中提议 1–3 个候选，并输出“证据—假设—操作—预期—风险”记录；不能产生任意代码、修改硬约束或直接输出分配矩阵。
3. **竞争**：使用有限预算的 successive halving 思路，先给多个候选小额同等评估预算，再只给表现好的候选追加预算 [S3]。候选的真实分配始终由确定性或限时求解器完成。
4. **守门**：仅当候选通过可行性、时延、泛化和改进阈值时才提交；否则保留 best-so-far 并回滚。这与 Simplex 用受信任基线系统保护未完全验证的高性能系统的分离原则一致 [S4]。
5. **停止**：任一条满足即停止：达到最大轮次；全局 deadline 或模型调用预算即将耗尽；改进不超过噪声与实用阈值；连续两轮未提交；变化信号消失；或候选违反安全规则。

这个方案将 Reflexion 的“环境反馈→语言反思→情节记忆→下一轮”思路用于**提案和解释**，但用显式评估器弥补其原文承认的“依赖自评能力、无形式改进保证”局限 [S5]。

## 证据支持与设计推导

### 1. 如何检测场景变化

ADWIN 维护可变长的最近数据窗口，在可区分的子窗口均值差超过依赖置信参数 `delta` 的界时报告变化，并在变化后丢弃过时窗口 [S1][S2]。它适合检测可持续产生的标量序列，但不会告诉控制器“变化的原因”。

因此建议使用两类信号：

| 层次 | 示例信号 | 检测方式 | 用途 |
|---|---|---|---|
| 场景摘要 | 可行边密度、价值偏度、概率均值/方差、冲突边密度、资源/目标比 | 对标准化差异设硬阈值，或对滑动距离用 ADWIN | “输入变了” |
| 求解轨迹 | 贪心后改进率、每千次评估改进、gap 收缩率、无改进次数、不可行候选率、时延分位数 | 显式阈值 + ADWIN | “旧策略失效了” |
| 结果质量 | 相对受信任基线的目标值、小实例最优 gap、可行性违反数 | 配对评估与显著性/实用阈值 | “退化值得反思” |

不建议只用大模型阅读日志判断变化：这样无法校准假阳性，也难以重现。超出训练分布的多维检测可在数据足够后增加，但不应成为初赛最小闭环的前置条件。

### 2. 反思操作空间如何受控

初赛只暴露可静态检查、可回放、可回滚的操作：

| 操作类型 | 允许示例 | 禁止或需人工核准 |
|---|---|---|
| 算法链选择 | 贪心→局部搜索；贪心→VLSN→小邻域精确优化 | 任意新增可执行代码 |
| 搜索预算 | top-k、邻域大小、各层 deadline、节点上限 | 取消总 deadline |
| 启发式参数 | 风险惩罚、冗余上限、候选排序权重、平局规则 | 越过 schema 边界的参数 |
| 约束模型 | 从预定义的保守/标准软约束中选择，且硬约束不变 | 删除容量、可达性或兼容性硬约束 |
| **反思知识** | 只读检索已验证经验；候选验证后新增/boost；证据冲突时降置 | 未经验证即全局生效 |

操作必须使用 schema 表示，至少包含 `operation_id`、`type`、`parameters`、`bounds`、`preconditions`、`expected_effect`、`rollback`。核心求解器只接受验证过的 schema，不解析自由文本命令。

### 3. 如何在有限轮次中选操作

对于已知最大预算的问题，Hyperband 原文的实验认为有限时域版比无限时域版更合适；其核心是在候选数量与每个候选预算之间作权衡 [S3]。本项目不必复制完整 Hyperband，可采用更小的 successive-halving 版本：

```text
incumbent = trusted_baseline(scene)
signals = detector.update(scene_summary, incumbent.trace)
if not trigger(signals): return incumbent

candidates = retrieve_validated_knowledge(signals)
candidates += llm_propose(signals, allowed_operation_schema, limit=3)
candidates = static_validate_and_deduplicate(candidates)

for reflection_round in 1..R_max:
    allocate_equal_small_budget(candidates)
    results = deterministic_solver.evaluate(candidates, fixed_validation_batch)
    discard_infeasible_timed_out_or_dominated(results)
    candidates = keep_top_fraction(results, eta)

    winner = gate(results, incumbent,
                  min_quality_gain, max_latency_regression,
                  feasibility=required, robustness=required)
    if winner passes: incumbent = transactional_commit(winner)
    else: rollback_to(incumbent)

    if stop_condition(): break

return incumbent
```

候选排序应先用硬门，再用效用函数。一个可实验校准的形式是：

```text
U(o) = median(Δquality_o)
       - λ_t · max(0, latency_ratio_o - 1)
       - λ_v · violation_count_o
       - λ_s · instability_o
```

其中 `instability` 可取不同种子/实例改进的四分位距。权重及改进阈值是项目设计参数，不是文献给定的常数。

### 4. 反馈如何更新策略与知识

Reflexion 用环境的二元或标量反馈生成语言摘要，将其作为情节记忆加入下一轮上下文，不需更新模型权重 [S5]。这说明语言反思可用于快速跨轮次传递经验，但原文也明确指出其依赖 LLM 自评能力、没有形式性能保证。

本项目因此应将更新拆成两条通道：

- **运行时策略**：只在当前实例中事务性地应用获胜操作；完整保留旧配置和 best-so-far。
- **持久化反思知识**：只将通过独立回放的“场景指纹—变化信号—操作—对照结果—适用域—置信—版本”记录升格为反思知识。当新证据冲突时不覆盖旧记录，而是降置、缩小适用域或新增版本。

这使“知识驱动决策”可通过 `knowledge_id -> operation_id -> solver_trace -> outcome` 链条被证明，而不是只展示一段反思文本。

### 5. 为何不让大模型承担核心目标分配

推荐的责任分离为：

| 部件 | 负责 | 不负责 |
|---|---|---|
| 特征/变化检测器 | 定量摘要、触发及置信 | 决定最终分配 |
| 大模型 | 解释信号、在操作目录中提案、生成结构化因果假设 | 直接输出大规模分配、判定可行性、绕过评估 |
| 求解器 | 产生可行解、返回目标值/gap/时延/轨迹 | 解释跨场景原因 |
| 守门器 | 硬约束、预算、回放、提交和回滚 | 依赖自由文本自行放行 |

Simplex 架构的一手论文将“高性能但未完全信任的系统”与“简单但高保证的基线系统”分离，并使用决策逻辑在需要时切换 [S4]。对本项目的直接推导是：基线智能体必须可独立产生合法解；反思候选失败时必须能在常数级配置切换成本下回到它。

## 建议的有限轮次协议

下列值是待实验校准的**设计假设**，不是文献事实：

| 项目 | 初始建议 | 理由 |
|---|---|---|
| 最大反思轮次 `R_max` | 3 | 足以做“探索→聚焦→确认”，且时延可严格封顶 |
| 每轮 LLM 候选数 | 最多 3 | 可在小预算下全部验证，避免候选爆炸 |
| 候选保留率 | `1/2` 或 `1/3` | 遵循 successive-halving 的预算聚焦思路 |
| 单轮时间上限 | 总内部预算的 20% | 为基线求解、最终验证和回滚保留余量 |
| 改进守门 | 配对中位改进 `>ε_quality`，且延迟退化 `<ε_latency` | 不为测量噪声提交更复杂策略 |
| 早停 | 连续 2 轮无提交 | 限制振荡和无效模型调用 |
| 回滚 | 任何不可行、超时、守门失败或未知操作 | fail closed，始终返回 best-so-far |

评估数据应拆成：用于选候选的 adaptation batch，用于守门的 holdout batch，以及只用于最终报告的 test batch。否则有限轮次仍会通过重复试验过拟合守门样本。

## 可复现证据链

每次反思至少保存：

```text
run_id / scene_hash / code_version / solver_version
baseline_config / baseline_objective / baseline_latency / feasibility
change_signal_values / detector_config / trigger_reason
retrieved_knowledge_ids / llm_model / prompt_hash / raw_schema_output
candidate_operation_ids / parameter_diff / random_seeds / allocated_budget
candidate_objective / latency / gap / violations / termination_reason
gate_thresholds / paired_validation_cases / decision / rollback_reason
committed_config_version / resulting_knowledge_id
```

公平对照需要 `norethink` 与 `rethink` 共用同一输入、种子、核心求解器、硬约束和总时间口径；单独报告反思开销。反思版不得使用基线版看不到的测试标签或预知真值。

## 与相关工单的接口

- #9 应把本文的操作目录、三轮上限、守门、早停和回滚固化为协议。
- #8 应定义反思知识的 `candidate -> validated -> active -> deprecated` 生命周期，以及冲突时的降置逻辑。
- #6 应将本文证据链和 adaptation/holdout/test 隔离纳入公平性验收。
- #4 应为可调的求解器链、参数边界及受信任降级路径给出最终实现规格。

## 一手来源

- **[S1]** Bifet, Gavaldà, *Learning from Time-Changing Data with Adaptive Windowing*, SDM 2007. [DOI](https://doi.org/10.1137/1.9781611972771.42)。提出 ADWIN，用自适应窗口与具有理论界的均值差检测处理时变数据。
- **[S2]** River, [`drift.ADWIN`](https://riverml.xyz/dev/api/drift/ADWIN/) 官方文档及 [官方源码](https://github.com/online-ml/river/blob/main/river/drift/adwin.py)。给出 `delta`、检查时钟、最小窗口、误差估计和 `drift_detected` 等实际接口；可直接支持工程实现，但本项目需自行校准参数。
- **[S3]** Li, Jamieson, DeSalvo, Rostamizadeh, Talwalkar, *Hyperband: A Novel Bandit-Based Approach to Hyperparameter Optimization*, JMLR 18, 2017. [原文](https://www.jmlr.org/papers/v18/16-558.html)。将预定义资源在多个配置间分配并早停落后配置；本文只将其预算聚焦原则迁移到反思操作选择。
- **[S4]** Sha et al., *The Simplex Architecture for Safe Online Control System Upgrades*, American Control Conference, 1998. [DOI](https://doi.org/10.1109/ACC.1998.703255)。给出高性能控制器、高保证基线控制器和决策切换模块的分离。本文借用其软件安全原则，不声称本题是实时控制系统。
- **[S5]** Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning*, NeurIPS 2023. [原文](https://proceedings.neurips.cc/paper_files/paper/2023/hash/1b44b878bb782e6954cd888628510e90-Abstract-Conference.html)。给出“标量/二元环境反馈→语言反思→情节记忆→下一轮”机制，也明确说明依赖自评且无形式保证的局限。

## 证据局限与待验证假设

- ADWIN 检测的是选定数值序列的统计变化，不能自动证明场景的战术语义或因果。项目必须通过合成变化回放校准假阳性与检测延迟。
- Hyperband 研究的是超参数优化。将 successive halving 用于反思操作只是本文的设计类比，效果必须与均匀预算、单候选及随机搜索做消融对照。
- Simplex 为回滚/切换提供架构依据，不会自动给出本题的可行性不变式。硬约束仍需等官方 schema 确认。
- Reflexion 的实验不是本题的组合优化实验，不能将其任务成功率增益直接当作本项目的性能预测。
- `R_max=3`、三个候选、两轮早停及所有门槛均是初始工程假设。应在 `20×50` 和 `200×500` 及 ID/OOD 变化分区上一起校准质量、p95 时延、失败率和反思调用次数。
- 官方评分函数、每例时限、大模型接口限额及分配硬约束尚未确认；协议参数只能在这些信息公布后定稿。
