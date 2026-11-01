# 可迁移反思知识的最小表示研究

> 对应 GitHub Issue #3：研究可迁移反思知识的最小表示
> 研究日期：2026-11-01
> 范围：仅使用公开、非敏感资料，聚焦知识表示与工程取舍，不涉及真实武器参数或作战部署。

## 结论

初赛阶段建议采用**带证据的适用域规则—案例记录**（下文简称“知识条目”），以版本化 JSON/JSONL 存储，并保留可派生为图的稳定标识符和关系字段。不建议在最小可行闭环中引入专用知识图谱或概率图。

最小闭环必须同时回答七个问题：

1. **何时适用**：场景特征与可机器判定的适用域；
2. **改什么**：对算法、启发式规则、参数、约束模型或知识的受控操作；
3. **为什么**：由场景变化、失效机理到操作的因果假设（必须标为“假设”，不得把相关性写成已证实因果）；
4. **有何效果**：公平对照下的指标变化、随机种子、运行预算和样本数；
5. **可信到什么程度**：支持/反例计数、置信度和状态；
6. **从哪里来**：原始场景、轨迹、反思轮次、代码/配置版本的溯源标识；
7. **是否真被使用**：检索命中、实际操作和决策差异的使用记录。

这比“保存一段反思文本”稍重，但才能完整证明本项目定义的**反思知识**确实在新场景中改变了决策并产生收益。

## 一手证据及启示

### 1. 反思文本能影响后续决策，但不足以独立承担可验证的知识生命周期

Reflexion 论文将环境的二值或标量反馈转换为语言摘要，存入长期记忆，后续试验中 Actor 同时根据轨迹和反思记忆决策；其实验中通常仅保留 1–3 条反思。这证明了小规模语义记忆可以成为行为改变通道，但纯文本没有显式适用域、冲突键、版本和对照证据，难以直接支持本题的可追踪验收。[^reflexion]

### 2. “规则 + 支持计数 + 案例检索”已有可复现先例

ExpeL 的官方实现保存成功/失败轨迹，从对比中提取规则；规则集支持 `ADD`/`EDIT`/`REMOVE`/`AGREE` 操作，每条规则带计数，归零时删除，并按计数排序。[^expel-rules] 它还把任务、步骤和思考编码为检索键，在评估时按任务、思考或步骤相似度召回历史案例。[^expel-memory][^expel-retrieval]

这一实现直接支持本项目的轻量起点，但不应照搬：它依赖字符串包含来识别相同规则，且“支持计数”不等于经过校准的概率。本项目应改用稳定 ID、结构化冲突键和显式证据。

### 3. 知识图谱是可选投影，不是最小闭环的先决条件

RDF 1.1 把图定义为主语—谓语—宾语三元组的集合，因而知识条目只要具有稳定 ID 和显式关系，以后就能映射成图。[^rdf] PROV-O 提供 `Entity`/`Activity`/`Agent`、生成、使用、派生、修订和失效等溯源语义，可作为后续跨系统互操作的参考。[^prov]

SHACL 可用 shapes graph 校验 RDF data graph，并输出带焦点节点、约束组件和严重性的验证报告。[^shacl] 但初赛所需的字段完整性、类型、枚举、取值范围和互斥条件可先用 JSON Schema 与应用层校验完成。只有当关系遍历、多源本体对齐或标准化溯源成为实际瓶颈时，专用图存储和 SHACL 才值得引入。

### 4. 概率图能表达不确定性，但需要稳定变量、图结构和足量数据

pgmpy 的官方实现将离散贝叶斯网络定义为“模型结构 + 每个节点的条件概率分布（CPD）”，并校验每个节点是否都有 CPD、CPD 父节点是否与图一致以及概率是否归一。[^pgmpy-model] 其贝叶斯参数估计还需选择 Dirichlet/BDeu/K2 先验和伪计数或等价样本量。[^pgmpy-estimation]

因此，概率图适合“同一组离散变量上已累积足够重复试验，且需要概率查询”的后期阶段。在反思条目少、场景特征尚未稳定时强行建图，先验和结构选择可能比数据本身更主导结果。初期存储支持/反例计数和原始证据，但不把它们冒充为校准后验概率。

## 最小知识条目

### 逻辑结构

```text
场景指纹 + 适用域
          │
          ▼
  受控操作候选 ────► 决策器配置/知识变化
          │                         │
          ▼                         ▼
      公平对照证据 ◄─── 行为差异与指标差异
          │
          ▼
 支持/反例、置信度、状态和版本
```

### 建议字段

| 字段组 | 最小内容 | 用途 |
|---|---|---|
| 身份与版本 | `id`, `schema_version`, `revision`, `status` | 稳定引用、迁移与回滚 |
| 适用域 | `context_features`, `applicability` | 先硬过滤，后相似度排序，防止跨域误用 |
| 因果假设 | `change_signal`, `failure_mode`, `rationale` | 使反思结果可解释，但与已验证事实分开 |
| 受控操作 | `target_type`, `target_key`, `operator`, `value`, `guard` | 将知识翻译为可执行且可白名单校验的改变 |
| 证据 | `evidence_refs`, `comparison`, `support_count`, `counterexample_count` | 保留对照、种子、预算和指标差异 |
| 置信与冲突 | `confidence`, `conflict_key`, `supersedes`, `invalid_reason` | 排序、冲突处理和废弃 |
| 溯源与使用 | `created_from`, `artifact_version`, `usage_refs` | 追溯知识来源，证明知识真正影响行为 |

### JSON 示例

```json
{
  "id": "rk-00017",
  "schema_version": "1.0",
  "revision": 3,
  "status": "validated",
  "context_features": {
    "scale_band": "large",
    "load_band": "high",
    "observed_change": "candidate_density_rise"
  },
  "applicability": {
    "all": [
      {"feature": "load_ratio", "op": ">=", "value": 0.8},
      {"feature": "time_budget_ms", "op": "<=", "value": 30000}
    ]
  },
  "causal_hypothesis": {
    "change_signal": "candidate_density_rise",
    "failure_mode": "local_search_stagnation",
    "rationale": "候选集密度升高使固定邻域更易停滞"
  },
  "operation": {
    "target_type": "parameter",
    "target_key": "search.neighborhood_width",
    "operator": "set",
    "value": 24,
    "guard": {"min": 8, "max": 32}
  },
  "evidence": {
    "evidence_refs": ["run-1021", "run-1022"],
    "comparison": {
      "protocol": "paired_same_seed_budget",
      "seeds": [11, 29, 47],
      "quality_delta_mean": 0.036,
      "latency_delta_ms_mean": 810
    },
    "support_count": 3,
    "counterexample_count": 0
  },
  "confidence": 0.72,
  "conflict_key": "large|high|search.neighborhood_width",
  "supersedes": ["rk-00009"],
  "created_from": {
    "scenario_id": "scenario-04",
    "reflection_round": 2,
    "artifact_version": "git:<commit>+config:<hash>"
  },
  "usage_refs": ["decision-trace-331"]
}
```

示例中的数值仅用于说明结构，不是竞赛参数建议。

## 抽取、校验、检索与更新

### 抽取与校验

1. 输入一次反思的场景快照、轨迹、反馈、基线/反思对照结果和已有候选知识。
2. 大模型只输出候选的适用域、失效机理和受控操作，不直接写入“已验证”状态。
3. 确定性校验器检查 schema、枚举、数值边界、操作白名单、证据引用存在性和预算约束。
4. 只有在相同输入、种子和计算预算的成对试验中达到门槛，候选才可晋升为 `validated`；否则保留为 `candidate` 或标记 `rejected`。

### 检索与行为影响

检索分两级：

1. **硬过滤**：根据状态、操作白名单、尺度档、时间预算和 `applicability` 排除不适用条目；
2. **软排序**：结合结构化特征距离、文本嵌入相似度、置信度、证据新鲜度和反例惩罚选取 Top-k。

检索命中本身不是知识有效的证据。每次使用都应记录：命中的知识 ID、应用前/后的决策器配置、最终分配差异摘要、指标差异和未采用原因。验收时至少需要展示一条“检索命中 → 受控操作 → 决策行为变化 → 对照收益”的完整链路。

### 增量更新与冲突消解

更新不覆盖原记录，而是追加证据并产生新 revision。先按 `conflict_key = 适用域粗粒度签名 + target_key` 找出可能冲突，再处理：

- 操作相同：合并证据，重新计算支持/反例和置信度；
- 操作不同但适用域可分：细化适用域，保留两条；
- 操作互斥且适用域重叠：不做无声覆盖，标记 `contested`，安排成对对照试验；
- 新证据在同预算下显著更优：新条目用 `supersedes` 指向旧条目，旧条目改为 `superseded`；
- 违反安全、预算或硬约束：直接标记 `invalid`，且保留原因供审计。

## 方案比较

| 方案 | 表达适用域/操作 | 增量更新 | 冲突处理 | 检索并影响行为 | 工程与数据成本 | 初赛建议 |
|---|---|---|---|---|---|---|
| 纯反思文本 | 弱，依赖提示解读 | 追加容易 | 弱，难可靠去重 | 可通过提示词影响 | 低 | 仅作原始候选，不作最终知识 |
| **适用域规则—案例记录** | **强，可直接执行** | **追加证据/版本即可** | **用冲突键、状态机和对照实验** | **硬过滤 + 相似度 + 受控操作** | **低到中** | **推荐主方案** |
| RDF/知识图谱 + SHACL | 强，关系查询优势明显 | 追加三元组容易 | 需另定义本体、开放世界下的冲突语义和优先级 | 需图查询/向量检索后再翻译成操作 | 中到高 | 当跨条目关系查询成为瓶颈时升级 |
| 贝叶斯网络/概率图 | 强，可做概率查询 | 参数可更新，结构更新较重 | 用后验融合不确定性，但不自动解决语义冲突 | 需将查询结果转为决策操作 | 高，需稳定变量、图结构、先验和足量数据 | 仅在重复样本充足且概率查询有明确价值时升级 |

## 升级门槛

### 升级到知识图谱

满足以下任一可进入小型原型验证，而不是直接全量迁移：

- 多次出现“查询多跳溯源/派生/替代关系”的实际需求；
- 知识条目与多个异构数据源需要统一本体对齐；
- 单机 JSONL/索引方案已在可测试负载下成为检索或一致性校验瓶颈；
- 存在跨工具交换 PROV-O/RDF 的硬性要求。

### 升级到概率图

应同时满足：

- 变量定义和离散状态在多轮实验中已稳定；
- 有足够重复样本估计关键 CPD，且能公开报告样本量和先验敏感性；
- 存在明确查询，例如 `P(操作改善 | 变化信号, 尺度档, 预算档)`，且该查询的决策价值高于复杂度；
- 完成结构、先验和参数不确定性的消融实验。

## 对后续决策工单的约束

1. Issue #8 定义知识生命周期时，应以上述知识条目为默认，明确 `candidate → validated → contested/superseded/invalid` 状态迁移。
2. Issue #9 定义反思操作空间时，`operation` 必须只能引用白名单目标和运算符，并有值域、预算和回滚守卫。
3. Issue #6 定义评测证据链时，必须将 `evidence_refs` 和 `usage_refs` 落到可复现日志，不得只用知识图可视化截图代替行为证据。
4. 实现可先用 JSON Schema + JSONL + 内存索引/小型向量索引；保留稳定 ID、关系字段和 append-only 事件，以免阻断后续图投影。

## 局限与待验证项

- Reflexion 和 ExpeL 的实验环境不是本竞赛的目标分配场景，它们只证明语言反思、规则更新与案例检索的机制可行，不能直接证明本项目中的效果。
- 置信度计算方式、证据门槛和显著性方法应在评测协议中决定，本文不预设。
- 官方 JSON Schema、反思轮次上限、运行算力和模型 API 条件尚需 Issue #13 核实；它们可能改变检索索引和存储的工程选择，但不改变七类最小语义字段。

## 参考资料

[^reflexion]: Noah Shinn et al., *Reflexion: Language Agents with Verbal Reinforcement Learning*, NeurIPS 2023. 论文第 3 节及 Algorithm 1 定义长/短期记忆与反思循环；第 4 节实验使用有限反思记忆。<https://arxiv.org/abs/2303.11366>

[^expel-rules]: Zhao et al., ExpeL 官方源码，`agent/expel.py` 的 `parse_rules` 与 `update_rules`（固定版本 `e41ec9a`）：<https://github.com/LeapLabTHU/ExpeL/blob/e41ec9a24823e7b560c561ab191441b56d9bcefc/agent/expel.py#L665-L744>

[^expel-memory]: Zhao et al., ExpeL 官方源码，`Trajectory` 对 task/step/thought 生成检索键（固定版本 `e41ec9a`）：<https://github.com/LeapLabTHU/ExpeL/blob/e41ec9a24823e7b560c561ab191441b56d9bcefc/memory/episode.py#L4-L29>

[^expel-retrieval]: Zhao et al., ExpeL 官方源码，实现了 task/thought/action/step 相似度检索（固定版本 `e41ec9a`）：<https://github.com/LeapLabTHU/ExpeL/blob/e41ec9a24823e7b560c561ab191441b56d9bcefc/agent/expel.py#L503-L656>

[^rdf]: W3C, *RDF 1.1 Concepts and Abstract Syntax*, 第 3.1 节“Triples”：<https://www.w3.org/TR/rdf11-concepts/#section-triples>

[^prov]: W3C, *PROV-O: The PROV Ontology*, 第 3 节及扩展术语：<https://www.w3.org/TR/prov-o/#description-starting-point-terms>

[^shacl]: W3C, *Shapes Constraint Language (SHACL)*, 第 3.6 节“Validation Report”：<https://www.w3.org/TR/shacl/#validation-report>

[^pgmpy-model]: pgmpy 官方源码，`DiscreteBayesianNetwork` 类文档与 `check_model`（固定版本 `d6ca91c`）：<https://github.com/pgmpy/pgmpy/blob/d6ca91c542afccb0c3a600cad50252af26729c88/pgmpy/models/DiscreteBayesianNetwork.py>

[^pgmpy-estimation]: pgmpy 官方源码，贝叶斯参数估计中的先验、伪计数和等价样本量（固定版本 `d6ca91c`）：<https://github.com/pgmpy/pgmpy/blob/d6ca91c542afccb0c3a600cad50252af26729c88/pgmpy/estimators/BayesianEstimator.py#L53-L158>
