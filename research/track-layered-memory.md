# 轨道：分层记忆

基线来源：`~/workspace/research_notes/api-routing-harness-memory-skills-20261023-1442/report.md`（2026-10-23）与技能参考 `~/workspace/custom-skills/agent_system_design/references/layered-memory.md`。

基线要点（2026-10-23）：
- 认知类比：Atkinson-Shiffrin 多存储模型、Baddeley 工作记忆、Tulving 三分法；CoALA（arXiv 2309.02427）形式化为工作记忆 + 情景/语义/程序长期记忆。
- 分类：按功能（工作/情景/语义/程序）、按存储（原始日志、向量索引、知识图谱/时序图、摘要树、KV 缓存、权重、结构化画像）、按检索（稠密、BM25、混合、图 Personalized PageRank、摘要树遍历、交叉编码器重排、时间衰减与重要性加权）、按写入（直接记录、事实抽取、摘要、反思、离线合并、衰减与失效、冲突解决）。
- 参考系统：MemGPT/Letta（2310.08560）、Generative Agents（2304.03442）、RAPTOR（2401.18059）、HippoRAG 1/2（2405.14831、2502.14802）、Zep/Graphiti（2501.13956）、A-MEM（2502.12110）；MemOS、MemoryBank、Mem0 本轮仅经二手摘要，设计前须复核一手论文。
- 七步分层检索流水线：查询理解 → 粗召回 → 分层扩展 → 重排 → 上下文预算分配（10%/25%/45%/20% 起点）→ 来源标注 → 写回与合并。
- 评估：检索层 recall@k、nDCG；端到端 LongMemEval、LoCoMo；须记录数据集版本、上下文预算、是否允许全上下文基线、日期。文件 + 全文检索基线不可忽视。
- 风险：记忆污染、陈旧记忆、隐私与跨用户泄漏、投毒、派生链修正困难。

---

## 2026-10-25 轮次：水位后一日内无可核实新发布；补记 Mem0 v3（2026-10-07）+ 排序/遗忘深化

窗口结论：按 Mem0、Letta、Zep/Graphiti、LangMem、arXiv 记忆/遗忘、新基准六路检索，**没有找到任何一篇落款日期为 2026-10-25、且出自官方文档/changelog/论文原文的分层记忆新进展**。本轮该轨道记一行“无新进展”；下述内容为水位前补漏与子主题深化，不冒充新进展。

### 补漏（水位前）：Mem0 Platform 新记忆算法 v3

- 来源（官方迁移页一手，页脚标注 Updated: 2026-10-07）：https://docs.mem0.ai/migration/platform-v2-to-v3
- 已核实机制：抽取从“两遍 LLM（抽取+合并）”改为“单遍 ADD-only（一次 LLM 调用）”；变异操作从 ADD/UPDATE/DELETE 改为只 ADD，不覆盖、不删除；智能体生成事实从“常被忽略”改为一等公民、等权存储；图记忆从“需外接 Neo4j 等并手动配置”改为平台内置自动，实体跨记忆链接；检索从“纯向量语义”改为多信号混合检索；时间信号在排序内部使用，不作为客户端返回字段；V3 搜索阈值默认值从 v2 的 0.3 改为 0.1，`top_k` 默认 10，`rerank` 默认仍为 false；`get_all` 改为分页信封；V1/V2 端点继续可用，旧记忆不被重处理。原文明确警告：记忆数会持续增长而非趋稳，且“需针对代表性查询重调硬阈值”。
- 对基线的修正意义：基线报告与 layered-memory.md 均写“Mem0 仅经二手摘要、未打开原文”——此页已是一手官方文档，可把该限定解除，但分数仍须保留“厂商自报”限定。
- 未核实（厂商自报，且跨页不一致，不可直接采用任一数字）：该 Platform 页称 LoCoMo 71.4→91.6、LongMemEval 67.8→93.4、抽取 p50 延迟约 2.0s→约 1.0s；但 Mem0 README/PyPI 镜像对同一新算法给出 LongMemEval 94.4 / 94.8 等不同数字。写入时必须并列标注口径冲突，或只写机制、不写分数。

### 论文线索（日期未核实 / 旧论文，深化用）

- **REM《What to Store, When to Recall, What to Forget in Experience Memory for Coding Agents》**（arXiv:2610.27377v1，arXiv 编号提示 2026-10，但正文未显示提交日）：已核实机制——记忆治理拆为写路径、读路径、环内召回三段显式决策；轨迹合并为分阶段文档（localize/understand/fix/verify，约 270 token）；实现记录平时 withheld，仅在“首次验证失败”这一可观测挣扎信号时注入，且单次曝光后即驱逐，分阶段文档则全程保留。核心论点可引用（“瓶颈不在检索精度，而在存储形态与注入后治理”）；论文自报数字（SWE-ContextBench Lite 相关轨迹 match@1 68.7% vs 55.6%、端任务解决率 27.6% vs 24.9%、成本低 17%）未核实。
- **EcphoryRAG**（arXiv:2510.08958，v1 日期 2025-10-10）：实体线索→多跳联想检索；论文自报平均 EM 从 0.392→0.474、token 比其他结构化 RAG 低可达 94%，均为论文自报，未核实。
- **Generative Agents 三因子公式**（经 Frontiers 综述二手复述，非原论文全文）：`score = w_r·Recency + w_i·Importance + w_s·Relevance`，Recency 每小时乘 0.995 衰减（纯算术推导：半衰期约 138 小时≈5.8 天；24 小时后剩约 0.887，7 天后约 0.43，30 天后约 0.027；若要 30 天半衰期，小时衰减因子应取约 0.99904），Importance 为写入时 LLM 打的 1–10 分，Relevance 为查询与记忆 embedding 余弦相似度。建议回查原论文确认单位是“小时”。
- **Graphiti 双时态模型**（经 DeepWiki 源码引用页二手复述，非官方论文原文）：四个时间字段 `created_at`（入库时间）、`valid_at`（现实中成立时间）、`invalid_at`（现实中失效时间）、`expired_at`（被新信息作废/保留 TTL 标记）；事实生命周期分 Active / Historical / Expired 三态，矛盾事实是被标记失效而非删除。
- **HippoRAG 2**（arXiv:2502.14802，检索片段，未打开全文核对）：把 passage 本身作为图节点（以 contains 边连向概念），PPR 同时从概念节点与 passage 节点播种，以缓解“概念压缩丢细节、上下文聚合丢结构”的概念-上下文权衡。

### 深化：排序信号与遗忘策略的设计空间

- **排序信号拆解（按可调参数写，不写“最优值”）**：
  - 相关性：余弦相似度、BM25、交叉编码器/LLM reranker；Mem0 v3 为语义+BM25+实体三路并行融合，合成分数仍在 [0,1]。失效：纯向量漏精确实体/数字；纯关键词漏同义表达；reranker 只在候选集内有效，粗召回漏了救不回来。
  - 时新性：指数衰减最常用；切忌把“小时/天”单位搞混后直接抄 0.995。对“一次说清的基础事实”极不友好：只出现一次的身份、约束、安全规则会被时间埋掉，必须配下限（floor）或类型豁免。
  - 重要性：写入时 LLM 一次性打分，读时零成本；可作为排序加权或衰减速率调节器（如重要性越高 λ 越小）。失效：写入时噪声永久化；重要性不随任务变，同一事实在不同查询下重要性应不同却无法调整。
  - 访问强化：每次被召回则强度 +1 或重置已流逝时间（MemoryBank 式）；Ebbinghaus 式变体用 `重要性 × e^(−λ_eff·天数) × (1 + 召回次数×系数)`，并设最小保留下限防彻底消失（后者来自个人实现博客/DeepWiki 二手资料，非标准论文，引用时须标注）。失效：“马太效应”——被召回得多→更强→更容易再被召回，与正确性无关；热门错误事实会被越推越高。
  - 图扩散权重：PPR 从查询命中节点播种扩散；HippoRAG 2 改进为概念节点与 passage 节点双源播种并平衡 reset 概率。失效：高度数枢纽节点吸走全部权重；未先做时间过滤时，已失效边会把过期事实扩散进候选集；边权若只用共现次数，会把“同名不同人”也连起来。
  - 时间意图：查询要先显式区分“现在为真”与“当时为真”——前者只取 `invalid_at` 为空的边，后者按目标时间点切片；`valid_at` 缺失时若默认回退到 `created_at`，会把“系统何时知道”误当成“事实何时成立”，历史问答系统性错位。
- **关键设计结论：排序与遗忘必须分层**。降权（decay/demotion）只影响排序，不删数据；失效（invalidation）改变“当前真值”集合但保留历史；删除（deletion）只用于合规/安全，且必须级联。把三者混成一个分数或一个删除动作，是最常见的设计错误。
- **遗忘/失效策略：按事实类型给有效期，而非全局统一 TTL**（参数为设计起点、非实测最优）：
  - 一次性凭证、已轮换密钥、已撤销权限：不衰减、不保留历史可召回形态——触发即硬删除，且级联向量、图边、摘要与派生链；ADD-only 系统必须另设删除通道，不能只靠排序压下去。
  - 价格、地址、日程、联系方式：设 `valid_from/valid_to` + 重查触发器；过期后自动降为历史态，当前查询默认过滤，历史查询仍可查。
  - 偏好、习惯：慢衰减 + 显式用户纠正永久覆盖；用户纠正、安全边界、身份事实永不自动过期，只接受显式纠正。
  - 程序性规则/技能版本：按版本 supersede 退休，旧版本保留为历史，不进入当前检索。
  - 原始 episode：原则上不删（它是溯源锚点）；只退休其派生摘要——摘要被新摘要 supersede 时，旧摘要降级，episode 指针保留。
  - 经验记录（REM 式）：分层 TTL——轻量分阶段文档长期保留，昂贵的实现细节记录“挣扎门控注入、单次曝光后驱逐”，即上下文内遗忘（不污染后续轮次），而非存储层删除。
- **冲突解决四条可操作规则**：1）同一主体+同一谓词的新事实到达时，旧事实置 `invalid_at/expired_at`，不物理删除；当前查询默认只读未失效边。2）新旧事实若只是互补（如“曾住纽约，现住旧金山”），两者并存，靠时间切片区分，不判为冲突删除——这正是 Mem0 ADD-only 的合理一面。3）无法判定新旧关系时，标记 `disputed` 而非静默覆盖，并在返回时附带冲突双方与各自时间戳，让调用方或用户裁决。4）纠正一条事实时，必须沿 `supersedes` 指针退休整条派生链（摘要、反思、由它生成的规则），否则旧结论会从摘要层“复活”——ADD-only 下派生链只会更长，退休成本更高。
- **参数选择依据**：先定业务半衰期，再反推 λ（`λ_每小时 = 2^(−1/半衰期小时数)`），不要直接抄论文的 0.995；重要性只用于排序加权与衰减速率调节，不要让它单独决定删除；阈值（如 Mem0 v3 的 0.1）是分数口径的函数，换混合检索后必须重调，旧阈值不可迁移（Mem0 官方迁移页明确警告）；任何“遗忘有效”的宣称，必须同时报告三项才可比：当前真值问答准确率、历史问答准确率、误删/误留率——只报前者会把“删得多”伪装成“记得准”。

### 已同步进 skill 参考

- layered-memory.md：Mem0 一节替换限定句（解除“未读原文”限定，分数列为厂商自报且口径冲突）；Step 4 末尾新增多信号融合与审计子项（返回各分量 score_detail、换口径重调阈值）；Step 7 “按类型过期”扩成六行决策表并区分降权/失效/删除三种语义；Risks 新增三条（衰减误伤基础事实、访问强化反馈环、图扩散污染）；Evaluation 新增 Mem0 数字口径冲突警示。无 SKILL.md 级变更。

### 剩余未核实缺口

1. Mem0 v3 的 LoCoMo/LongMemEval 分数、延迟、token 数——厂商自报且跨页不一致，无独立复现。
2. REM、EcphoryRAG 的全部效果数字——论文自报，未独立复现；REM 的确切提交日未在正文确认。
3. LangMem、Letta、Zep/Graphiti 在 2026-10-25 当日无可核实更新——这是“检索未见”，不是“官方确认无更新”；下轮应直接查各官方 changelog 页。


---

## 2026-10-27 轮次：缺口复核——Mem0 分数口径统一、v3 体系内演进、Zep v4、论文提交日确认

### 已核实（一手为主）

- **Mem0 LongMemEval 口径统一（修正上一轮「口径冲突」结论）**：无 Temporal Reasoning 时 top_200 93.4%（467/500）、top_50 90.4%（452/500）；加 Temporal Reasoning 后 top_200 94.4%、top_50 94.8%（500 题）；另有 94.8% 属于 Zep 的 DMR 基准，Mem0 未公布 DMR 结果。残留一处官方自相矛盾：同一篇 Temporal Reasoning 博客 Takeaways 节把 top_200 新值误标在 top_50 表述下，引用以 Results 节为准。全部数字均为厂商自报。
- **Mem0 v3 体系内演进**：无 v4；当前线 TS SDK v3.1.0 / Python SDK v2.0.12；Memory Decay（平台 v3 search 专属，默认关闭，0.3×–1.5× 只重排不删除）；Temporal Reasoning（平台 v3 默认开启，写时时间 enrichment、读时意图分类加性重排、不预过滤，`reference_date` 可锚定）。OSS/平台分界为二手，须标注。
- **Letta**：官方 changelog 内容一手——0.33.2 用 workflow 分析改进 memory 初始化，0.33.0 后台 worker 承担 incidental memory upkeep 与 post-turn Git 冲突修复；文档页（0.33.2）滞后于 GitHub（v0.34.12），引用以仓库为准；版本日期页面未显示，故不写。
- **Zep v4 与 hyperedges**（官方博客一手）：2026-10-14 Context API、2026-10-15 hyperedges，v3 并行可用。
- **论文提交日确认（一手）**：REM arXiv:2610.27377 v1 提交 2026-10-19（UTC 22:59:56），删除 track 笔记「提交日未核实」限定；EcphoryRAG arXiv:2510.08958 v1 提交 2025-10-10（内文 2025-10-13 为手稿日期）。

### 检索未见（不是官方确认无更新）

- 自 2026-10-25 以来：LangMem（取证不完整，无独立 changelog 页）、Letta、Zep/Graphiti 官方侧均检索未见新发布；Graphiti 官方 releases 未逐条拉取成功。

### 已同步进 skill 参考

- layered-memory.md：Mem0 分数警示改写为四格口径 + DMR 归属 + Takeaways 错误警示；新增 Mem0 v3 扩展行；Letta 行追加 upkeep 与文档滞后警示；Zep 行追加 v4 与 hyperedges；Evaluation 一节同步改写为分条件口径。

### 剩余未核实缺口（更新）

1. Mem0 全部基准分数仍为厂商自报，无独立复现（口径已统一，但真实性仍未验证）。
2. REM/EcphoryRAG 效果数字仍为作者自报（提交日已确认，数字未变）。
3. Graphiti 0.31.2 vs v0.30.2 两二手冲突，未决（不整合）。
4. LangMem 最新版取证不完整；Letta FrontierMemory（2026-10-21）仅二手线索，不入技能。
5. 已解决：Mem0 分数口径冲突（主口径已统一，仅剩博客一处标注错误）、REM/EcphoryRAG 提交日。


---

## 2026-10-28 轮次：缺口复核——Graphiti 冲突解决、LangMem 确认、FrontierMemory 官方确认

### 已核实（一手为主）

- **Graphiti 版本冲突已解决**：最新正式版为 v0.31.2（2026-10-20，certifi 更新应对 eMudhra 风险，非预发布）；v0.30.2（2026-09-08）为已被取代的旧线。序列：v0.30.1（2026-09-01）→ v0.30.2（2026-09-08）→ v0.31.0（2026-10-14）→ v0.31.1（2026-10-15）→ v0.31.2（2026-10-20）；PyPI 同步为 0.31.2。
- **LangMem 最新版已核实**：langmem 包最新正式版仍为 0.0.30（2025-10-27 上传），此前 0.0.29（2025-07-28）、0.0.28（2025-07-09）；截至 2026-10-28，2025-10-27 之后无新发布（PyPI 发布历史）。
- **Letta FrontierMemory 官方确认为真**：官网研究页与博文（2026 年 10 月；具体日仅见二手报道，不写 2026-10-21）——15 个任务，分任务、用户、环境三类记忆，每任务至少两位研究员人工复核，全通过率、单次尝试；全部排行榜与 harness 对比分数均为厂商自报、样本小，必须按面板引用，不得拼成单一排名。
- **Letta 引用基准更新**：文档 changelog 页顶部仍只到 0.33.2（自述可能滞后），GitHub releases 已到 v0.34.24（2026-10-25）；引用最新版以仓库为准。
- **Mem0 Decay 细节补全**：官方博客（2026-05-08，旧功能）——每条记忆记录最近 20 次访问时间戳，公开分数钳制在 [0,1]；开源 SDK 支持范围与文档说法存在张力，未裁决。
- **Zep v4 失效语义**：`reference_time`（RFC3339，与接收时间分离）用于回填时保持 valid_at / invalid_at 顺序；Hyperedges 与 Ingestion Traces（2026-10-14 / 10-15 官方博客）。
- **窗口内新动态**：Mem0（最新 v2.2.1，2026-09-25）、Zep/Graphiti、LangMem 在 2026-10-27 之后均检索未见新发布。

### 已同步进 skill 参考

- layered-memory.md：Graphiti 行追加版本冲突解决与序列；新增 LangMem 最新版行；Letta 行追加 FrontierMemory 与引用基准更新；Mem0 扩展行追加 Decay 细节与开源支持张力警示。

### 剩余未核实缺口（更新）

1. Mem0 全部基准分数仍为厂商自报，无独立复现（口径已统一，但真实性仍未验证）。
2. REM/EcphoryRAG 效果数字仍为作者自报（提交日已确认，数字未变）。
3. Mem0 Decay 开源 SDK 支持范围争议（未裁决）。
4. Zep Cloud SDK 4.0.1（未一手核实）；letta-code v0.34.22 / v0.34.23 缺位原因（需逐条复核）。
5. TEPA 论文原文（仅见二手转述，未读原文）。
6. 已解决：Graphiti 版本冲突、LangMem 最新版、FrontierMemory 存在性与结构、Letta 引用基准。


---

## 2026-10-29 轮次：缺口复核——TEPA 原文、Mem0 Decay 裁决、Letta 0.34.26

### 已核实（一手为主）

- **TEPA 论文原文（缺口关闭）**：标题《TEPA: Revoking Stale Memories for Conflict-Robust Language Agents》，arXiv:2608.07429 [cs.AI]，v1 提交 2026-08-07、v2 修订 2026-08-10；本轮读 v1 全文，v2 摘要一致但正文改动未逐行比对，引用以 v1 为准。机制：keyed precedent `p=(k, v, s, f, σ, τ)`，σ ∈ {Hypothesis, Active, Revoked}；检索只取 Active 集；撤销规则为后验均值 q(p) < θ_rev（n_min=5 后）或近期成功率 < θ_rec=0.34（n_rec=3）；默认 θ_prop=3、θ_rev=0.3、θ_pro=0.6；TEPA-Full 另加 trial validation（预算 3，≥60% 为正且至少 1 个支持试验成功才晋升）。数字全部论文自报（未独立核实）：受控全反转 TEPA 0.950 vs append-only / last-write-wins 0.210；SH-6k 0.890（与 last-write-wins 持平）；SH-32k 0.680、MH-6k 0.040、SH-262k 0.000。局限：须能抽出有用的冲突键；键噪声增大时平滑退化。
- **Mem0 Decay 开源支持争议裁决（缺口关闭，推翻上一轮“未裁决”）**：官方 Platform vs Open Source 对照页明确写明 Decay 在 OSS 为 Not supported（传入 `decay` 报错）；Temporal Reasoning 同样 Platform 专属；Graph Memory 外接图存储集成在 v3 管线落地时被 OSS 移除，仅保留实体抽取与排序加权、无可查询图与 `relations` 字段。Decay 机制（官方 Highlights 页与 2026-05-08 博客，旧功能）：仅搜索时重排，0.3×–1.5×，永不删除归零；每次返回即更新访问历史；公开分数钳制 [0,1]；仅 v3 搜索；默认关闭、按项目 opt-in。Platform 客户端 SDK 版本（Python v2.0.2 / TypeScript v3.0.3）与 OSS 包 mem0ai 2.2.1 不是同一版本线。上轮“最近 20 次访问时间戳”细节本轮在官方页面检索片段中未复见，不重申（不删除，未证伪），留待下轮复核。
- **Decay vs reference_time 对照（仅 track，不进技能正文）**：Decay = 降权（排序层）；Zep reference_time（RFC3339）+ valid_at / invalid_at = 失效（真值层，可按目标时间点切片查询“当时为真”；回填缺 reference_time 时回退到接收时间）；TEPA = 退出 active 集但留档（有效性状态层）。三者正交，可同时需要。
- **Letta letta-code 新见版本**：npm 侧最新见 0.34.26，npm.io 页脚 Updated 2026-10-28T02:30:56Z（北京时间 2026-10-28 10:30），落在 10-28 当日，不计入 10-28 之后窗口；其变更内容未核实（文档 changelog 仍滞后于 0.33.2）。
- **五家复核戳（2026-10-29）**：Mem0 检索未见新发布（最新见 mem0ai 2.2.1，2026-09-25）；Graphiti 最新仍为 0.31.2（2026-10-20）；LangMem 最新仍为 0.0.30（2025-10-27）；Zep 4.0.1 本身未一手核实，更无 10-28 之后新版一手证据。“检索未见”不等于“官方确认无更新”。

### 已同步进 skill 参考

- layered-memory.md：新增 TEPA 条目（Reference systems）与 Step 7 指针句；Mem0 v3 extensions 段改为官方对照页裁决口径（Decay / Temporal / Graph 三条 OSS 边界）并保留 20 次时间戳细节的复核标注；Letta Caution 句推进至 npm 侧 0.34.26（变更内容未核实）。

### 剩余未核实缺口（更新）

1. Mem0 全部基准分数仍为厂商自报，无独立复现（口径已统一，但真实性仍未验证）。
2. REM / EcphoryRAG 效果数字仍为作者自报（提交日已确认，数字未变）。
3. Zep Cloud SDK 4.0.1 一手核实（PyPI 发布历史直查或 GitHub releases 逐条拉取；本轮 PyPI 直访被 Client Challenge 拦截）。
4. letta-code v0.34.22 / v0.34.23 缺位原因（需 GitHub releases 列表或 npm 版本时间线逐条比对 0.34.21→0.34.26）；0.34.26 变更内容与确切发布时间（以 npm 注册表时间戳为准再确认）。
5. TEPA v2 正文改动清单；TEPA 数字的独立复现（全部效果数字目前仅为论文自报）；TEPA“遗忘有效”结论的历史问答准确率 / 误删误留率对照表缺口。
6. Mem0 Decay“最近 20 次访问时间戳”细节复核（本轮未复见）。
7. 已解决：TEPA 论文原文、Mem0 Decay 开源支持裁决、Letta 0.34.26 版本见证。


---

## 2026-10-30 轮次：缺口复核——Zep 语言归属、letta-code 缺位模式、TEPA 遗忘缺口

### 已核实（一手为主）

- **Zep Cloud SDK 4.0.1 语言归属纠偏（执行者亲自复核 npm.io 页）**：4.0.1 是 TypeScript 包 `@getzep/zep-cloud`（更新戳 2026-10-20T18:51:57Z）；Python 包 `zep-cloud` 未见 4.0.1，搜索缓存最新可见仍为 4.0.0rc2（2026-10-13）；集成文档要求 Python `zep-cloud>=4.0.0,<5` 只证明 4.0.x 线存在；PyPI 直访仍被 Client Challenge 拦截。
- **letta-code 缺位模式（GitHub Releases 首页一手）**：首页序列 0.34.31、0.34.30、0.34.29、0.34.28、0.34.27、0.34.26、0.34.24、0.34.21、0.34.19、0.34.18；0.34.25 bump 合并入 0.34.26、0.34.20 合并入 0.34.21；v0.34.23 tag 存在（为 0.34.24 比较基线）但无独立 Release 条目（官方未说明）；0.34.22 首页全文无痕迹；0.34.26 确切发布时间仍未决；0.34.31 为 2026-10-29 边界日版（npm 更新戳 2026-10-29T08:30:16Z）；文档 changelog 仍滞后于 0.33.2。
- **Mem0 Decay 20 次时间戳复核通过**：官方博客（2026-05-08）How it works 一节原文仍在——“Each memory tracks up to its last 20 access timestamps.”；上一轮复核标注解除。
- **TEPA v2 摘要比对**：v1（2026-08-07）与 v2（2026-08-10）文件大小同为 461 KB、摘要逐字相同，v2 未改动摘要中的阈值或数字；v2 正文逐行比对仍未决（一次抓取超时后未重试），不得写成“正文无改动”。
- **TEPA 遗忘有效性缺口（v1 全文逐词检索确认）**：除冲突准确率外，论文无保留率、误删率、误留率或历史问答退化数据（retention 0 次、false 仅附录理论 2 处、histor* 仅 3 处非评测含义）；阶段成功率、生命周期轨迹、活跃集规模、键噪声表、干净 SH-6k 持平、偏好流总体、边界数字均不是保留率证据。
- **窗口内**：Letta 0.34.31（2026-10-29 边界日）；Mem0、Graphiti、LangMem（仍为 0.0.30，2025-10-27）、Zep 在 2026-10-29 之后均未发现新发布（2026-10-30 当日无新发布）。

### 已同步进 skill 参考

- layered-memory.md：Zep 段追加 SDK 语言归属句；Letta Caution 句改为 0.34.31 序列与缺位模式；Mem0 Decay 段复核标注改为已确认；TEPA 段追加遗忘有效性缺口与 v2 摘要比对句。

### 剩余未核实缺口（更新）

1. Zep Python 包 `zep-cloud` 4.0.1 是否存在（PyPI 直访拦截未解除）。
2. letta-code 0.34.26 确切发布时间；0.34.22 缺位原因；0.34.23 无独立 Release 条目原因。
3. TEPA v2 正文逐行差异。
4. Graphiti 0.31.2 确切发布日（二手说法冲突：Oct 20 vs “截至 Oct 28 最新”）。
5. mem0ai 1.0.10 发布日期。
6. Mem0 / REM / EcphoryRAG 数字独立复现（口径已统一，真实性仍未验证）。
7. 已解决：Zep SDK 语言归属、letta-code 缺位模式与 0.34.31、Mem0 Decay 20 次时间戳复核、TEPA 遗忘缺口确认、TEPA v2 摘要比对。


---

## 2026-10-31 轮次：缺口复核——Zep Python 4.0.1、letta-code 流水、TEPA v2 差异、遗忘验收拆分

### 已核实（一手为主）

- **Zep Python `zep-cloud` 4.0.1 存在（已解决，PyPI 一手）**：tar.gz 与 wheel 均 Uploaded Oct 20, 2026，与 TS 侧对齐；上一轮拦截口径作废。
- **letta-code 0.34.26 发布时间已解决（GitHub API 一手）**：2026-10-28T02:48:19Z。
- **letta-code“缺位”旧说法推翻**：0.34.22 / 0.34.23 / 0.34.25 均为真实存在的 bump-only 极简发布，改动由下一份完整说明折入；各自精确 published_at 仍未取得。
- **PyPI `letta` 线分叉（修正“快照滞后”说法）**：PyPI JSON 显示 info.version 止于 0.34.12（2026-10-12），之后未跟进 GitHub / npm 线（停更、改名还是改道未见官方说明）；引用当前版以 npm / GitHub 为准。
- **TEPA v2 差异已解决（HTML 渲染行比对）**：唯一差异为 v2 新增一行预印本声明，方法阈值与结果数字未改动；v1/v2 引用可互换但应注明预印本（TeX 字节 diff 未做）。
- **Graphiti 0.31.2 发布日已解决（PyPI 一手）**：2026-10-20；“截至 Oct 28/29 最新”为观察日，不冲突。
- **mem0ai 1.0.10 发布日期已解决（PyPI 一手）**：2026-04-01（仅 track，不入技能）。
- **窗口内新动态**：Letta 0.34.33（npm Updated 2026-10-30T05:29:00Z，变更内容未核实）；Mem0、Graphiti、Zep、LangMem 检索未见窗口发布。
- **深化**：门控→排序→剪枝三层；遗忘验收拆分为隐式失效（TEPA 式）与显式删除（Bemula 式，作者自报 ≤1.5%）；图记忆等预算下未胜扁平向量基线（作者自报，单流水线限定）；Mem0 两组自报成绩口径差异未核实，不拼成一条引用。

### 已同步进 skill 参考

- layered-memory.md：Zep 段改为 Python 4.0.1 已确认；Letta Caution 句改为 0.34.33 序列、0.34.26 发布时间、bump-only 特征与 PyPI 线分叉；TEPA 段改为 v2 已比对口径；Step 7 追加遗忘验收拆分与三层结构句（ADD-only 原句保留）。

### 剩余未核实缺口（更新）

1. letta-code bump 版精确 published_at；0.34.33 变更内容；PyPI `letta` 线改道原因。
2. TEPA TeX 源码级 diff。
3. Mem0 两组自报成绩口径差异；Memora / FAMA 精确数值（仅二手转述）；遗忘探索 39% / 15%（二手转述，预印本限定）。
4. Mem0 / REM / EcphoryRAG 数字独立复现（维持厂商自报口径）。
5. 已解决：Zep Python 4.0.1、Graphiti 0.31.2 发布日、mem0ai 1.0.10 发布日、TEPA v2 差异、letta-code 0.34.26 发布时间与 bump-only 流水、PyPI `letta` 分叉原因澄清。


---

## 2026-11-03 轮次：缺口复核——letta bump-only 修正、0.34.39、TEPA 源码 diff、Mem0 口径、Memora 原文、遗忘三档

### 已核实（一手为主）

- **letta-code bump 版性质修正**：0.34.22 / 0.34.23 / 0.34.25 只有 tag、没有 GitHub Release 对象（API 两次 404、100 条列表缺席）；精确时间以 npm 为准：0.34.22 为 2026-10-23T05:36:14Z、0.34.23 为同日 06:32:27Z、0.34.25 为 2026-10-26T23:12:49Z；上一轮“真实存在的 bump-only 极简发布”修正为“有 tag、无 Release 对象的 bump-only tag”；npm 与 GitHub 时间相差约 5 分钟，不得混写。
- **letta-code 0.34.33 变更已核实**：仅 `docs(skills)` 要求记忆提交附来源 + bump，无功能改动（`published_at` 2026-10-30T05:34:27Z）。
- **窗口内 Letta 连发四版**：0.34.36（2026-10-31）、0.34.37 / 0.34.38 / 0.34.39（2026-11-02），当前最新 0.34.39；官方 Changelog 文档仍只列至 0.33.x。
- **PyPI `letta` 线改道原因仍未决**：最新仍 0.34.12，差距拉大到 27 个版本；已排除改名与已公告停用，剩余推断仅为 wheel 发布流水线停滞。
- **TEPA TeX 源码级 diff 已解决**：v1/v2 源码包实测，`main.tex` 唯一差异为 v2 在 `\maketitle` 后新增 6 行预印本声明，其余文件字节相同；HTML 结论在源码级成立。
- **Mem0 两组自报成绩口径差异已解释**：迁移快照 91.6 / 93.4（OSS 2026-08-05、Platform 2026-10-07）vs 现行 headline 92.5 / 94.4（Temporal Reasoning 默认开、top_200、托管平台）；TR 博客无 TR 基线 86.1 / 90.4 与迁移页旧基线 71.4 / 67.8 不构成同一链条；top_50 94.8 与 top_200 94.4 不得混写；Takeaways 误标警示保留。
- **Memora / FAMA 论文原文一手核实**（Findings of ACL 2026，pp. 26814–26841）：FAMA 公式、季档 1,171.4 次操作、纯 LLM 扣减缩小（32.6 → 17.8）vs 记忆 agent 扣减增大（18.2 → 29.5）、Monthly 排名反转；数字作者自报，未独立复现。
- **遗忘验收第三档**：MemCut（arXiv:2610.21405，2026-10-15）：LLM 引导删除后 residual recovery 仍 39–76%，候选覆盖率是瓶颈；MemLeak（arXiv:2606.29788）：留存文本 18.3% / 图像 12.0%，47% 图像泄漏文本不可见，语义删除 12% → 2.0%；DyadMem（arXiv:2610.03020，2026-10-05）：Gold 78.7–96.3% vs Full-Pipeline 47.7% / 57.4%，写侧先于读侧失败。以上数字均为作者自报。
- **窗口内**：仅 Letta 有带日期的一手新发布；Mem0、Graphiti、Zep、LangMem 均为检索未见。

### 已同步进 skill 参考

- layered-memory.md：Letta 段改为 tag-only 修正口径 + npm 时间戳 + 5 分钟时差警示；Caution 句改为当前版 0.34.39 与窗口内四版 + PyPI 线 27 版差距；TEPA 段改为 TeX 源码 diff 口径；Mem0 段与 Evaluation 节改为两组并列口径；Step 7 追加遗忘第三档、删除侧候选覆盖约束与 DyadMem 写侧瓶颈句；Reference systems 新增 Memora / FAMA 段。

### 剩余未核实缺口（更新）

1. PyPI `letta` 线改道原因（已排除改名与已公告停用；差距 27 个版本）。
2. Mem0 / REM / EcphoryRAG / Memora / MemCut / MemLeak / DyadMem 全部效果数字的独立复现。
3. 《Scaling Discovery Requires Forgetting》39% / 15%（仍为二手转述，预印本限定；本轮未复核原文）。
4. 已解决：bump 版时间戳性质、0.34.33 变更、窗口内新版本（0.34.36–0.34.39）、TEPA 源码 diff、Mem0 口径差异、Memora 原文复核。


---

## 2026-11-04 轮次：缺口复核——Scaling Discovery 原文口径、Memora 消歧、letta 复核戳、Mem0 双口径未解

### 已核实（一手为主）

- **letta-code 当前版仍为 0.34.39（窗口内无新版）**：GitHub Releases 顶部仍为 0.34.39（PR #5464）；npm.io 镜像登记其更新戳为 2026-11-02T21:26:30Z（属上一轮已覆盖）；PyPI `letta` 线仍为 0.34.12（2026-10-12，5 个 wheel），与 npm 线差距仍为 27 个 patch 版本（窗口内两边均未动，差距未拉大也未缩小）；改道官方原因仍未找到（PyPI 页自述仅说明发行形态差异：捆绑 Node.js 运行时的 CLI，非 Python SDK；应用内自更新禁用）。Letta 官方文档 Changelog 页自承“文档可能延迟，以 GitHub releases 为准”，正文节选只列到 0.33.x。0.34.39 的 GitHub 精确 published_at 本轮未取到（仅 npm.io 更新戳交叉印证），不得写出。
- **Scaling Discovery 39% / 15% 论文原文复核（arXiv:2610.20563v1，2026-10-14，预印本）**：摘要原句逐字属实——“up to 15% improvement compared to parallel agents, and up to 39% compared to a single agent”，正文另称四任务上对最强基线提升 1.9%–15%。口径收窄：两数不是同一比较基线，也不是全任务平均——39% 为 AHC014 上遗忘团队均值对单智能体的增益（均值口径；最佳单次轨迹算出仅约 +19%），15% 为四任务中对并行基线的最大增益。实验边界（原文自述）：4 任务、每配置 3 复本、单一组合（Pi agent + DeepSeek V4.1 Flash，每复本 $20 预算）；遗忘同时清空会话与候选历史，无法归因到二者之一。分任务数字来自 alphaXiv 梳理（二手转述，未独立复算）。全部数字论文自报，未独立核实。
- **Mem0 两处官方分数不一致（新发现，一手比对）**：README 表新算法 LoCoMo 92.5 / LongMemEval 94.4，并自承“分数反映托管平台，含开源 SDK 没有的专有优化；开源用户应期待方向相似但数字不相同”；迁移指南（更新 2026-10-07）却写 91.6 / 93.4。同一厂商、同一“新算法”，两处数字相差 0.9–1.0 点，官方解释未决，引用时须二选一并注明出处，不得混用。上一轮技能已将两组并列并写入平台限定，本轮不改技能句。
- **Redis 第三方复现报告方法论警示（窗口外材料，底稿 2026 年 5 月）**：其 LongMemEval 表格中 Mem0 一行“Reported 49.0%，Reproduced —（空）”，即 Redis 没有复现 Mem0；表格另有列值错位行（如 LangMem 行全为 95.4% 重复值），不得引用该表 LangMem 行。反例 Vertex Memory Bank 复现 42.4% vs 论文约 70%（低约 28 点）。意义在方法论层：已发表记忆数字在统一 judge 下经常不复现，七项数字更应维持“自报”标注。
- **“Memora”一名双指（必须消歧）**：(a) 微软 Memora（谐波记忆表征，ICML 2026，arXiv:2602.03315）：LoCoMo 86.3%、LongMemEval 87.4% LLM-judge 准确率（作者自报），每对话记忆条数 344 vs Mem0 的 651；(b) 同名基准 Memora（《From Recall to Forgetting》，Findings of ACL 2026，FAMA 指标）——上一轮技能中的“Memora / FAMA”指后者，不得把微软框架数字安到基准头上。
- **窗口内其余否定性结论**：Mem0（PyPI mem0ai 2.2.1，2026-09-25；所谓“Mem0 3.3.1”指的是 TypeScript SDK 线，不是 Python 包）、Graphiti（graphiti-core 0.31.2，2026-10-20）、Zep（最新产品博文 2026-10-29，v4 为 2026-10-14，均窗口外）、LangMem（0.0.30，2025-10-27）均无窗口内新发布。Zep Community Edition 停更说仅二手对照，无官方公告原文，列为未决，不得写入技能正文。REM / EcphoryRAG / MemCut / MemLeak / DyadMem / TEPA 数字本窗口内均无第三方独立复现或官方更新，全部维持作者自报标注。
- **深挖素材（论文自报，不整段入技能）**：遗忘策略三分法——周期性全量遗忘（Scaling Discovery）、冲突驱动撤销（TEPA：同 key 新证据冲突时被取代先例移入撤销存档，可审计、退出常规检索）、单次暴露后逐出（REM：高成本实现记录只给一次，文档本体留存全程，属“检索时遗忘”）；检索排序互补证据——ARM（高频项晋升免于衰减）、D-Mem 双进程门控（向量快取 + 全量审议兜底，GPT-4o-mini 上 LoCoMo F1 53.5 vs 静态基线 51.2）、Memora 策略引导检索、EcphoryRAG 线索驱动联想检索；Mem0 单遍 ADD-only 与 TEPA 式撤销、MemCut 式删除需求直接冲突——同一系统内“积累”与“遗忘”目标未调和，此为技能中应保留的张力注记，非定论。

### 已同步进 skill 参考

- layered-memory.md：Memora / FAMA 段末追加消歧句；新增 Scaling Discovery 周期性遗忘 bullet（含口径收窄与实验边界）；Letta Caution 段复核戳推进至 2026-11-04（仍 0.34.39、PyPI 差距仍 27 版）。

### 剩余未核实缺口（更新）

1. PyPI `letta` 线改道原因（官方说明仍未见；差距本窗口未变，仍为 27 版）。
2. Mem0 新算法分数双口径（README 92.5/94.4 vs 迁移指南 91.6/93.4）的官方解释；OSS 评估框架复现平台数的实测结果。
3. Zep Community Edition 是否停止维护的官方口径（一手公告缺失）。
4. 七项效果数字（Mem0/REM/EcphoryRAG/Memora/MemCut/MemLeak/DyadMem）与 TEPA、Scaling Discovery 数字的第三方独立复现——本窗口内零新增。
5. letta-code 0.34.39 的 GitHub Release 精确 published_at（仅 npm.io 更新戳交叉印证）。
6. LoCoMo 参考答案错误率（Penfield Labs 称 6.4%）仅见 Medium 二手转述，未直读审计原文。
7. 已解决：Scaling Discovery 39% / 15% 原文口径；“Memora”双指消歧；letta-code 2026-11-04 复核戳。

---

## 2026-11-05 轮次：缺口复核——Zep CE 停更确认、LoCoMo 审计直读、PyPI letta 口径修正与差距刷新

### 已核实（一手为主）

- **Zep Community Edition 已确认停止维护（一手官方公告，缺口关闭）**：官方博客（2025-04-02 发布，2026-06-03 更新）原话停止维护与发布、仓库保留 Apache 2.0 但不再更新或支持，开源重心全部转向 Graphiti；官方 FAQ（更新 2026-10-13T21:27:46Z，窗口之前）重复 deprecated 且不再支持。窗口内无变化。
- **LoCoMo 参考答案错误率审计原文已直读（缺口关闭，直读层面）**：Penfield Labs dev.to 原文（2026-04-04）确认：99/1540 题错误（6.4%），理论满分约 93.6%；judge 接受 62.81% 故意错误答案；类别含答案 key 幻觉事实、错误时间推理、说话人归属错误；LoCoMo-Plus 继承全部 1,540 原题（含 99 错）；LongMemEval-S 每题约 115K token，更像上下文窗测试而非记忆测试。利益冲突提示：Penfield Labs 自身是记忆项目竞争方，属有立场的审计方；二手长评称已交叉核对仓库 dial481/locomo-audit。只写“审计原文自述、仓库公开可查”，不写“经独立第三方复现证实”。
- **PyPI letta 线口径修正并刷新差距数字（执行者亲自直读 PyPI 页）**：PyPI 当前文案（一手）说明 `letta` 包现为 Letta Code CLI 的 pip 分发（捆绑 Node.js 运行时的 `letta` 命令；应用内自更新已禁用；“This distribution is the Letta Code CLI, not a Python SDK or API server. For Python applications calling the Letta API, use `letta-client`”）。“分叉的 Python 服务器线”框架是错的：它是同一 CLI 的第二种安装通道。差距数字已刷新：PyPI 页 release files 仍为 0.34.12（2026-10-12，5 个 wheel），与 npm 0.34.39 的差距仍为 27 个版本（与 2026-11-04 戳一致，未拉大也未缩小）。子代理直读被 Client Challenge 拦截的说法不影响本结论（执行者直读成功）。
- **npm 最新版仍为 0.34.39**（npm.io 更新戳 2026-11-02T21:26:30Z，窗口之前）；0.34.39 的 GitHub Release 精确 published_at 本轮仍未能直读（npm 时间戳不得替代）。
- **Mem0 双口径官方解释仍未找到一句话声明**：并存文档可用配置解释（头条 92.5/94.4 为托管平台结果；迁移指南 91.6/93.4 为生产代表性模型栈基准；仓库 top200 计得 LoCoMo 91.6%（1410/1540）、LongMemEval 93.4%（467/500），top50 为 82.7%/90.4%）。Mem0《How to Read Memory Benchmarks》一手承认同一系统不同配置可报 74% 也可报 92.5%。OSS 管线 LongMemEval（厂商自报，仅抽取模型不同）：GPT-5 抽取 91.0%、GPT-OSS-120B 89.8%、Llama 4 Maverick 88.6%、Gemma 4 31B 88.6%——三组数字配置均不同，不可排成复现结论。第三方：TAM 重评称旧 judge 计 88.46%、Mem0 自家 judge 计 94.32%（二手）；Atlaso 读者匹配实验称默认 OSS 管线在其条件下仅 44.2%（自带方法局限声明，二手）。
- **七项效果数字第三方独立复现：窗口内新增为零**：SurrealDB 重算 BEAM 1M 头条均值 64.1%→严格准确率仅 40.1%（top200），2026-10-14（窗口外，二手）；REM / EcphoryRAG / Memora / MemCut / MemLeak（须区分同名两篇：多模态版 arXiv:2606.29788 与多租户版 arXiv:2610.04195）/ DyadMem / TEPA / Scaling Discovery 均无窗口内新的独立复现。Tether（arXiv:2610.25123，论文自报）：写时记录 verdict 依赖精确版本，读时支撑失效即撤回；Fallout 模拟上游失效对象发布 62% vs Tether 0.3%（关系与 verdict 同述时），检查量少 37%；分会话到达时不安全率 9.4–22.1%。
- **窗口内新鲜动态：五家均无带日期的一手新稳定发布**：Mem0（mem0ai 2.2.1，2026-09-25；mem0-mcp 归档公告见于 2026-11-03 快照，属 MCP 旧仓库停更、窗口之前）；Letta（npm 0.34.39，窗口之前）；Graphiti（本轮二手聚合称 v0.31.0，2026-10-14，不得回退技能正文已一手确认的 v0.31.2，2026-10-20；未见 v0.32）；Zep（CE 已停更；Cloud 无公开版号）；LangMem（未见专属窗口内发布；langchain 1.4.8 属父框架）。
- **深挖素材（论文自报，不整段入技能）**：检索三维（信号 + 状态 + 时间）；四种遗忘对比（周期性遗忘仅搜索型任务；冲突驱动撤销默认撤销留档；检索时遗忘 Mem0 Decay / Tether；单次暴露后逐出 REM 属经验复用治理）；TTL 按类型分治；基准分数必须附配方（作答模型、评分模型、评分提示宽松度、检索深度、计入类别）。

### 已同步进 skill 参考

- layered-memory.md：Zep 段追加 CE 停更一手确认句；Evaluation 节追加 Mem0 配方警示（74% / 92.5% 并存说、OSS 抽取 spread）与 LoCoMo 审计警示；Letta 段改为 CLI 分发通道口径 + 差距刷新至 2026-11-05（仍 27 版）；Step 7 追加检索三维与 Tether 依赖撤回；Letta 复核戳推进至 2026-11-05。

### 剩余未核实缺口（更新）

1. letta-code 0.34.39 的 GitHub Release 精确 published_at（仅 npm 更新戳交叉印证）。
2. Mem0 新算法分数双口径（README 92.5/94.4 vs 迁移指南 91.6/93.4）的一句话官方解释；OSS 评估框架复现平台数的实测结果。
3. 七项效果数字（Mem0/REM/EcphoryRAG/Memora/MemCut/MemLeak/DyadMem）与 TEPA、Scaling Discovery、Tether 数字的第三方独立复现——本窗口内零新增。
4. Graphiti v0.31.0 之后、LangMem、Mem0 核心窗口内确无发布（基于公开搜索未见，非发布方正面证明）。
5. 已解决：Zep CE 停更官方口径；LoCoMo 审计原文直读；PyPI letta 口径与差距刷新。

## 2026-11-07 轮次小节（版本 0.10.0；六路并行之记忆一路）

- 本轮增量：letta-code 0.34.39 published_at 已解决（2026-11-02T21:31:41Z，一手 API；npm 更新戳 21:26:30Z 是另一时钟）；npm 已到 0.34.44（0.34.40 为上一轮漏检补漏，2026-11-04T23:36:11Z；0.34.41 2026-11-05T04:23:03Z；0.34.42/0.34.43 2026-11-06；0.34.44 2026-11-07T01:41:09Z）；PyPI letta 仍 0.34.12（2026-10-12），差距由 27 拉大到 32 版；0.34.x 线位于作用域包 @letta-ai/letta-code，裸 letta-code 是 1.0.0 指针包；Mem0 双口径拼合解释链已完整（memory-benchmarks 提交 4b61c5d，2026-05-13：93.4→94.4 / 90.4→94.8 是 Temporal Reasoning 默认全开所致；须标注为执行者拼合，非官方原句）；七项效果数字第三方独立复现窗口内新增仍为零（连续第三轮）；Zep Cloud SDK 4.1.0 发布于 2026-11-02T19:17:40Z（仅 SDK 再生成，窗口外补漏）；Graphiti v0.31.2 published_at 再确认 2026-10-20T17:25:21Z；深挖新论文 MEMOIR（arXiv:2611.07631，提交 2026-11-05，窗口内；证据 recall@5 较 bare cosine +11.92pp，论文自报）；窗口外补漏 Human-Inspired Memory Architecture（arXiv:2605.08538，2026-05-08；衰减 λ=0.001/小时，约 29 天半衰期）与 Re³Rank（arXiv:2611.02550，2026-11-02）。
- 已同步进 skill 参考：layered-memory.md 追加 letta 0.34.39 published_at 与 0.34.40–0.34.44 段、PyPI 差距 32 版与 @letta-ai/letta-code 口径段、Mem0 拼合解释链段、Zep 4.1.0 / Graphiti 再确认 / mem0ai 2.2.1 再确认段、Step 7 追加 MEMOIR 与 Human-Inspired bullet 与独立复现三轮为零 + MemLeak 两篇消歧 bullet。
- 推翻的旧说法：letta-code“仍 0.34.39”；PyPI 差距“仍 27 版”；上一轮对 letta-code 0.34.40 的漏检；“Zep Cloud 无公开版号”说法。
- 剩余未核实缺口（更新）：1. PyPI letta 改道原因（仓库 PyPI 发布 workflow 历史）。2. Mem0 双口径一句话官方解释；OSS 框架复现平台数；results/ gitignored 张力。3. 七项效果数字的第三方独立复现——连续三轮为零。4. Mem0 TS SDK 线；MemLeak 多模态版 / TEPA / Scaling / EcphoryRAG 版本页未重开复核。5. 已解决：letta-code 0.34.39 published_at；Mem0 双口径拼合链；Zep Cloud SDK 4.1.0；Graphiti v0.31.2 再确认。

## 2026-11-07 轮次小节（下午轮，版本 0.11.0；六路并行之记忆一路）

- 本轮增量：PyPI letta 改道原因一手解决——commit 2365e012（2026-10-29T01:46:04Z，`ci(release): publish to PyPI only on X.Y.0 releases [LET-14054] (#5134)`）改 `.github/workflows/release.yml`，PyPI 发布仅当版本匹配 `X.Y.0`（手动 `force_pypi` workflow-dispatch 旁路）；0.34.12（2026-10-12）为门控前最后一版，之后 0.34.13–0.34.44 均为补丁版、被门控挡住——停滞是刻意发布策略，非流水线损坏；下个 PyPI 发布应在 0.35.0 或手动 force_pypi（规则推断、无官方排期）；Mem0 TR 博客一句话官方句已获（“Temporal Reasoning is on by default for every Mem0 project”，评估页同句），但博客内部自相矛盾（引言 top_200 vs Takeaways top_50 配对 90.4%），以提交 4b61c5d 映射为准（top_50：90.4%→94.8%；top_200：93.4%→94.4%）；Mem0 TS SDK 线（npm mem0ai 3.3.1，2026-09-25T17:44:47Z；beta 3.0.0-beta.2）与 Python 线（2.2.1）分版本，不得混用；Mem0 results/ 张力解决——.gitignore 否定项 `!results/platform/` 与 `!results/oss/` 豁免两子目录，仓库内 15 条 results/ 路径（8 个平台 JSON、4 个 OSS JSON、其余目录项），README“results/ — Benchmark output (gitignored)”为简化描述；OSS 框架自带 4 次自家 LongMemEval 运行（厂商自家 2026 年 4 月运行，gpt5 91.0% / gemma4_31b 88.6% / llama4_maverick 88.6% / parasail_120b 89.8%，非第三方复现）；四篇论文版本页重开复核（MemLeak 多模态 v1 2026-06-29、TEPA v2 2026-08-10、Scaling v1 2026-10-14、EcphoryRAG v1 2025-10-10，均无新版，上一轮“未重开”限定解除）；七项效果数字第三方独立复现连续第四轮为零；letta 仍 0.34.44、PyPI 差距仍 32 版，四厂商窗口内均无新发布（Mem0 Python 2.2.1、Graphiti 0.31.2、Zep Cloud SDK 4.1.0、LangMem 0.0.30）。
- 已同步进 skill 参考：layered-memory.md 更新 letta 改道原因段（X.Y.0 门控 + force_pypi）、Mem0 TR 博客矛盾段（以提交映射为准）、Mem0 TS SDK 线段、results/ 张力解决段、Step 7 四篇论文版本页复核段（第三轮为零改为第四轮为零）、Letta bullet 追加窗口内四厂商复核戳。
- 推翻的旧说法：PyPI letta“改道原因仍未找到”；Mem0 TR“无一句话官方句”（已获，但博客内部矛盾新发现）；“Mem0 results/ 张力未决”；“四篇论文版本页未重开”；“七项数字复现连续三轮为零”（改为四轮）。
- 剩余未核实缺口（更新）：1. Mem0 双口径一句话官方解释。2. 七项效果数字的第三方独立复现——连续四轮为零。3. Letta 服务器仓停滞（0.16.8，2026-05-14）。4. PyPI letta 下次发布时间（0.35.0 或 force_pypi）。5. 已解决：PyPI letta 改道原因；Mem0 results/ 张力；四篇论文版本页复核；Mem0 TS SDK 线。

## 2026-11-07 轮次小节（晚间轮，版本 0.12.0；六路并行之记忆一路）

- 本轮增量：Mem0 官方指南《How to Read Memory Benchmarks》（mem0.ai，2026-10-08）新收：FAQ 原句 top_50 94.8% vs top_200 94.4%、多会话 93.2%→88.0%（记忆越多噪声越大）——只解释检索深度机制，不触及 TR 开关，TR 一句话官方解释仍未见；七项数字第三方复现窗口内新增为零，连续第五轮；相邻独立重测 memtrails（2026-11-03/11-05，LoCoMo strict 76.5%）已登记为非协议级重测，不得后轮误计；letta-ai/letta 服务器仓仍停 0.16.8（2026-05-14），最新提交 2026-10-18 纯文档；仓名补正：2365e012/#5134/release.yml 属于 letta-ai/letta-code（活跃仓），在 letta 仓查无（422/404）；letta-code 仍 0.34.44（npm 2026-11-07T01:36:40Z；GitHub 2026-11-07T01:41:09Z），PyPI 仍 0.34.12，差距仍 32 版；无 0.35.0、无可观测 force_pypi（手动触发是否发生过不可观测，不得写成未发生）；Mem0 SDK：npm mem0ai 仍 3.3.1、PyPI 仍 2.2.1；Graphiti 仍 v0.31.2、Zep Cloud SDK 仍 4.1.0；补漏：Zep Agent Skills（2026-11-05 官宣 + 论文 arXiv:2611.01386，提交 2026-11-01），上两轮均漏记；宣称小模型配 Skills 接近 Opus 5.5（厂商自报）；新论文 arXiv:2611.08341（2026-11-05，LAQ 局部/全局保真度，52 次全部提升 G，p=8.5×10⁻⁴）；MEMOIR 仍 v1；StateBench（2611.03094，提交 11-02，最难任务四家前沿模型 ≤13%）为窗口外边界登记。
- 已同步进 skill 参考：layered-memory.md 追加 Mem0 基准指南段、七项复现第五轮为零 + memtrails 登记段、letta 服务器仓段、PyPI 仓名补正段、Mem0 SDK 戳推进段、Zep Agent Skills 补漏段、2611.08341 新论文段。
- 推翻的旧说法：2365e012 引证仓名未注明（补正为 letta-ai/letta-code）；七项复现“连续四轮为零”（改为五轮）。
- 剩余未核实缺口（更新）：1. Mem0 双口径一句话官方解释。2. 七项效果数字的第三方独立复现——连续五轮为零。3. PyPI letta 下次发布时间（0.35.0 或 force_pypi）。4. 已解决：Mem0 基准指南机制解释；letta 服务器仓停滞确认；2365e012 仓名补正；Zep Agent Skills 补漏。

## 2026-11-08 轮次小节（晨轮，版本 0.13.0；六路并行之记忆一路）

- 本轮增量：Anthropic《The components of agent memory》（2026-11-06）一手补漏——四层架构（transcripts / notes / representations / injected context）+ dreaming 五作用（纠错、建表征、晋升、目标聚焦、补缺口），数字全部厂商自报，性质为设计手记非产品发布；「用后反馈」成为独立排序层（EARM / MEMOIR / dreaming 三路同向）；压缩与保真度可分开优化（LAQ / SUMONE：SUMONE arXiv:2611.06445，2026-11-04，−78%/−75% 论文自报），FlyHash 反例（arXiv:2611.05791，2026-11-04：质量优 ≠ 延迟/内存优）；新论文 EARM（arXiv:2608.22767，2026-08-24，+2.14–6.62% 论文自报）；Mem0 TR 博客重读：第三组切口 top_50 82.7%→91.8%、7 类记忆/7 种时间查询模式、state key 语义、负面结果（开放域 −1.9 pts、p95 +198 ms）、temporal conflict resolution 预告，双口径一句话官方解释仍未见；七项数字第三方复现窗口内新增为零，连续第六轮；五家复核戳推进（letta-code 仍 0.34.44、PyPI letta 仍 0.34.12、差距仍 32 版；Mem0 Python 仍 2.2.1、TS SDK 仍 3.3.1；Graphiti 仍 v0.31.2；Zep Cloud SDK 仍 4.1.0；LangMem 仍 0.0.30；Zep Agent Skills 论文仍 v1，arcXiv Updated 含义未明）。唯一部分失败：`getzep/zep-python` GitHub Releases API 查询回 403 rate limit（PyPI/npm 侧已确认 4.1.0，结论不受影响），待下轮重试。
- 已同步进 skill 参考：layered-memory.md 追加 Anthropic 记忆长文段、EARM/「用后反馈」排序层段、SUMONE/FlyHash 段、Mem0 TR 博客重读更新段、七项复现第六轮为零段、五家复核戳推进段、Zep 复核段。
- 推翻的旧说法：七项复现「连续五轮为零」（改为六轮）；Mem0 TR 博客「只读过 Highlights/Takeaways」（本轮全文重读，第三组切口与机制细节已补）。
- 剩余未核实缺口（更新）：1. Mem0 双口径一句话官方解释。2. 七项效果数字的第三方独立复现——连续六轮为零。3. PyPI letta 下次发布时间（0.35.0 或 force_pypi）。4. Zep Agent Skills 论文是否有 v2；getzep/zep-python releases 侧复核（本轮 403，待下轮重试）。5. Anthropic CL-Bench 均值数字与 Dreams API 细节（仅二手）。6. 已解决：Anthropic 记忆长文补漏；Mem0 TR 博客全文重读；SUMONE/FlyHash/EARM 三篇新论文。

## 2026-11-08 轮次小节（中午轮，版本 0.14.0；六路并行之记忆一路）

- 本轮增量：Anthropic《The components of agent memory》CL-Bench 均值数字升级为一手（Opus 5.5：0.655→0.838；Sonnet 5.5：0.644→0.833；Opus 5：0.564→0.828；Opus 4.7：0.383→0.700；Sonnet 5：0.306→0.610；Sonnet 4.6：0.330→0.364；均出自 figure alt 文本）；Dreams API 细节升级为一手（异步任务、1–100 个会话、输入 store 永不修改、beta 头 `dreaming-2026-04-21`）。纠正：Zep LongMemEval 63.8% 是厂商自报（Zep 自家论文 gpt-4o-mini 配置），不是独立复现；agentos 引用的 arXiv:2512.13564 是综述、非复现研究。getzep/zep-python releases 侧 403 闭环（v4.1.0，2026-11-02T19:17:38Z）；Zep Agent Skills 论文无 v2（仅 v1）。DolphinBench 首个第三方独立跑分（Empirical，2026-11-06，75.2% vs 官方行 70.7%，PR 待审、作者自报、未验证）与 Maximem 复现（2026-05-27，73.8% vs 头条 93.4%，二手转述）补充登记，不计入七项口径。Route to Remember（arXiv:2611.05636v1，2026-11-03）：权重记忆（OPSD 自蒸馏）作长期事实存储的第三条线。七项效果数字第三方独立复现连续第七轮为零；五家无新版（letta-code 仍 0.34.44、PyPI letta 仍 0.34.12 差距 32 版、Mem0 2.2.1/3.3.1、Graphiti v0.31.2、Zep 4.1.0、LangMem 0.0.30）；Mem0 双口径官方解释仍未出现。
- 已同步进 skill 参考：layered-memory.md 追加 Anthropic CL-Bench 均值数字与 Dreams API 细节一手段、Zep 63.8% 纠正段、zep-python 403 闭环段、DolphinBench/Maximem 补充登记段、Route to Remember 段、七项复现第七轮为零段、五家复核戳推进段、Mem0 双口径复核段。
- 推翻的旧说法：「Zep 63.8% 是独立复现」（厂商自报）；七项复现「连续六轮为零」（改为七轮）。
- 剩余未核实缺口（更新）：1. Mem0 双口径一句话官方解释。2. 七项效果数字的第三方独立复现——连续七轮为零。3. PyPI letta 下次发布时间（0.35.0 或 force_pypi）。4. Zep Agent Skills 论文是否有 v2（仍 v1）；DolphinBench Empirical PR 审定结果；Route to Remember 全文。5. 已解决：Anthropic CL-Bench 均值数字与 Dreams API 细节；Zep 63.8% 纠正；getzep/zep-python releases 侧 403 闭环。

## 2026-11-08 轮次小节（晚间轮，版本 0.15.0；六路并行之记忆一路）

- 本轮增量：letta-code v0.34.45 窗口内发布（npm 2026-11-08T07:44:31Z，GitHub 07:49:26Z；sharp 0.35.5、memfs 修复、停用已退役托管模型默认值；无记忆功能变更）；PyPI letta 仍 0.34.12，差距 32 → 33 版；PyPI 下次发布仍无信号。Route to Remember 全文数字（arXiv:2611.05636v1 全文一手）：OPSD 59.9% vs 检索 47.8%/抽取 51.9%/SFT 36.9%；VtR +6.2 points；门控保留 22% pooled / 37.9% native（切口区分）；对检索约 10⁵ 查询回本；边界：缺席不可检视、随年龄幻觉上升、fabrication 率高估约 1/5。补漏三篇：2611.05306（256K→16K 提速 3.6×/5.6×/7×）、EvalMem v2（检索缺陷 22.1% 最频；recall-first 70.2%→95.6%）、What Eviction Destroys（8k 预算不可逆损失占比 1.00）。复核戳：七项复现连续第八轮为零；DolphinBench PR #13/#14 仍 open；Mem0 双口径解释仍未出现；Zep 论文仍 v1；Zep npm 包名 @getzep/zep-cloud。
- 已同步进 skill 参考：layered-memory.md 追加 letta-code v0.34.45 段、Route to Remember 全文数字段、三篇补漏论文段、记忆轨复核戳段。
- 推翻的旧说法：七项复现「连续七轮为零」（改为八轮）；PyPI letta 差距「32 版」（改为 33 版）。
- 剩余未核实缺口（更新）：1. Mem0 双口径一句话官方解释。2. 七项效果数字的第三方独立复现——连续八轮为零。3. PyPI letta 下次发布时间（0.35.0 或 force_pypi）。4. Zep Agent Skills 论文是否有 v2（仍 v1）；DolphinBench Empirical PR 审定结果。5. 已解决：letta-code v0.34.45；Route to Remember 全文；三篇补漏论文。

## 2026-11-08 轮次小节（深夜轮，版本 0.16.0；六路并行之记忆一路）

- 本轮增量：Mem0 双口径机制拼图基本完成（2026-07-21 对比博客方法论节全文一手：平台 v3 管线 + top_200 + "retrieval depth is the simplest explanation for why you'll see other Mem0 numbers floating around"；10-08 基准指南 top_50 94.8% vs top_200 94.4%；09-04 评估页 top_200 预算 + ±1 分评判置信区间；三者合起来解释约 1.0 分差距；仍缺一句点名 91.6/93.4 vs 92.5/94.4 并调和的官方原话，不得把机制解释写成那句话；同博客厂商自报数字：LangMem 58.1%/p95 60s/≈130 tokens、OpenAI Memory 52.9%/0.9s；Mem0 OSS 已移除 enable_graph/graph_store 配置；BEAM contradiction_resolution 仅 35.7%（1M）/32.5%（10M），为当前记忆系统最弱环节）。六篇补漏记忆论文摘要级收录（均为论文自报、摘要级，全文未验）：RememberStack（arXiv:2611.06019v1，栈式记忆 push/pop/peek/squash/replace，0.90 vs 0.71，少 24.4% tokens）、ReCap（arXiv:2511.05311v1，近因加权检索）、The Memory Wall（arXiv:2611.05727v1，窗口减半 Sonnet 4.5 准确率减半）、Memorization is not Memory（arXiv:2611.05200v1，参数记忆 vs 非参数记忆系统比较）、RECON（arXiv:2611.05766v1，检索式上下文压缩）、WALDEN（arXiv:2611.05972v1，5,000+ 用例、17 项细粒度指标，最佳 GPT-5 0.65，未达 0.8 实战门槛）。Dreams API 口径收窄：默认写新 store、不改输入 store；`output_behavior=update_existing` 时写回输入 store（官方 API 参考 POST /v1/dreams 一手）。复核戳：七项复现连续第九轮为零；letta-code 仍 0.34.45（0.34.46 不存在；GitHub 无 Release 对象）；PyPI letta 仍 0.34.12，差距仍 33 版；DolphinBench PR #13/#14 仍 open；Zep npm 包名 @getzep/zep-cloud 再确认；Anthropic 记忆文章与 Dreams API 窗口内无更新；arXiv 周日 0 提交。
- 已同步进 skill 参考：layered-memory.md Mem0 双口径段追加机制拼图与同博客数字、Dreams API 段改写口径收窄句、复核戳段追加深夜轮复核（第九轮为零、letta-code/PyPI/DolphinBench/Zep 包名）、文件末尾追加六篇补漏论文段。
- 推翻的旧说法：Dreams API「输入 store 永不修改」（未加限定的过度概括，收窄为仅默认行为）；七项复现「连续八轮为零」（改为九轮）。
- 剩余未核实缺口（更新）：1. Mem0 双口径一句话官方解释（机制拼图已基本完成，仅缺一句点名调和的原话）。2. 七项效果数字的第三方独立复现——连续九轮为零。3. PyPI letta 下次发布时间（0.35.0 或 force_pypi）。4. DolphinBench PR #13/#14 审定结果。5. 六篇补漏论文全文核实。6. RememberStack harness 下轮跟进。7. 已解决：Mem0 双口径机制拼图；Dreams API 口径收窄；六篇补漏论文摘要级收录。

## 2026-11-09 晨轮（第 17 轮，窗口 2026-11-08 23:37 → 2026-11-09 05:37）

- 本轮增量 M1–M4（4 条，其中 M4 为 track-only）；版本 0.16.0 → 0.17.0。
- M1：**六篇补漏论文 arXiv 编号全部纠正**（上一轮重大台账错误）：上一轮登记的六个编号均与题名不符——RememberStack 2611.06019、ReCap 2511.05311、The Memory Wall 2611.05727、Memorization is not Memory 2611.05200、RECON 2611.05766、WALDEN 2611.05972。按题名重定位：ReCap → arXiv:2609.40118v1（2026-09-30；约 95% 延迟降低，论文自报）；RECON → arXiv:2607.16716v1（2026-07-18；最强非 Oracle 系统 22.4%，论文自报）；Memorization is not Memory → 近似命中 arXiv:2604.27707v2（实题《Contextual Agentic Memory is a Memo, Not True Memory》，v1 2026-04-30、v2 2026-08-05；概念性，无基准数字）；RememberStack → 无 arXiv 论文（remember.dev 产品；PyPI `remember` 0.17.1、2026-09-25；`rememberstack` 0.1.0）；The Memory Wall 与 WALDEN → 未定位。均仅摘要页级核实；数字为论文自报。
- M2：Zep Agent Skills 论文（arXiv:2611.01386v1，2026-11-01；Zep 博客 2026-11-05）——技能由已验证轨迹编译，三名较弱执行器 pass 率 +4.5–18.8pt（论文自报）。
- M3：letta-code 仍 0.34.45（2026-11-08T07:49:26Z），0.34.46 未出现；PyPI letta 仍 0.34.12，差距仍 33 版；letta server 仓库仍 0.16.8。
- M4（track-only）：DolphinBench PR #13（Empirical 451/600 = 75.2%，厂商自报）与 PR #14（提交格式）均仍 open（更新于 2026-11-05）；Mem0 双口径官方解释仍缺失；七项第三方复现连续**第十轮为零**。三项窗口外旧复现线索（Atlaso 读者匹配 Mem0 44.2%、Vectorize Mem0 49.0 2026-03-15、arXiv:2512.13564 Zep 71.2%→63.8%）记为待查，不入增量。
- 五家厂商版本戳全不变：Mem0 2.2.1/3.3.1、Graphiti v0.31.2、Zep 4.1.0、LangMem 0.0.30。

### 剩余未核实缺口（本轨）

- The Memory Wall 与 WALDEN 论文定位（未找到）。
- RememberStack 记忆系统后续跟进（产品，非论文）。
- Mem0 双口径官方解释（仍缺失）。
- 七项第三方复现：阳性复现仍为零，但 Mem0 93.4% 一项首次出现两条公开第三方实测（Atlaso 44.2%、arXiv:2603.04814 49.0%），均为不复现/低值（2026-11-09 中午轮已入技能；旧「连续十轮为零」表述已修正）。
- DolphinBench PR #13/#14 合并动向。

## 2026-11-09 中午轮（版本 0.18.0；窗口 2026-11-09 05:37 → 11:37 CST）

- 本轨增量（M1–M6）：Mem0 第三方实测首次入技能（Atlaso 44.2% 与 arXiv:2603.04814 49.0%，均附限定）；七项复现表述修正；Zep 63.8% 转引链确认（TEM Table 8 + Memanto Table IX）；Mem0 官方双口径再分歧（README 94.4%/94.8% vs 迁移页 93.4%）；Scalaris（92.2%/90.77%）等五篇 11 月记忆论文登记；letta-code v0.34.46 窗口内发布（2026-11-09T02:43:10Z），PyPI 差距 33 → 34 版；RememberStack `remember` 0.19.0。
- 剩余未核实缺口：The Memory Wall / WALDEN 仍未定位；arXiv 周一 2026-11-09 批次枚举；2512.13564 内表细节；Mem0 93.4 vs 94.4 变更；Anthropic 报告直链；Zep 90.2% 官方原文核对；letta-code v0.34.46 release notes 内容；Mem0 双口径官方解释（仍缺失）；DolphinBench PR #13/#14 合并动向。

## 2026-11-09 晚间轮（版本 0.19.0；窗口 2026-11-09 11:37 → 17:37 CST）

- 本轨增量（M1–M7）：2512.13564 内表细节关闭（"63.8"/"71.2" 全文 0 命中，Table 7 仅列 Zep 行无数字，引用链立于 TEM Table 8 + Memanto Table IX + Zep 论文 Table 2）；Zep 90.2% 官方原文核对（LongMemEval 90.2% = 451/500、LoCoMo 94.7% = 1,459/1,540，gpt-5.4 reader/judge；非同一测量，禁止"71.2%→90.2% 提升"叙事）；letta-code v0.34.47/v0.34.48 窗口内两版（PyPI 差距 34 → 36 版，v0.34.46 发布说明已补读，无记忆功能变更）；Anthropic 记忆报告直链取得（CL-Bench alt 文本限定：上轮记录未被推翻亦未被本轮复核）；MemoryAtlas 新浮现旧重跑记录登记（不计为复现，阳性第三方独立复现仍为零）；11-06 批次五篇检索排序/遗忘论文摘要级（Compaction vs RAG +6.8pp p=5.5e-08、BEAM 预算、MemoryDojo 155 轨迹全系统可投毒、ENGRAM、HiMEM）；The Memory Wall / WALDEN 仍未定位（`ti:"memory wall"` 38 篇全读、`ti:WALDEN` 1 篇无关）。
- 剩余未核实缺口：The Memory Wall / WALDEN 定位线索（作者/会议/产品）；Mem0 93.4 vs 94.4 变更；Mem0 双口径官方解释（仍缺失）；DolphinBench PR #13/#14 合并动向；五篇 11-06 批次论文全文深读（Compaction vs RAG 优先）；MemoryAtlas 各重跑行方法核实。

## 2026-11-09 深夜轮（版本 0.20.0；窗口 2026-11-09 17:37 → 23:37 CST）

- 本轨增量（M1–M7）：MemoryAtlas LiCoMemory 六行方法已确认（arXiv:2511.01448v1 Table 1 + §4.1，附四项限定：竞品重跑、基线版本未声明、裁判未声明、分母 1,986 vs 1,540 不可拼比）；Hindsight 两行（66.88/71.2）原始方法未定位；Mem0 LongMemEval 49 行归属错误（数字另有出处 arXiv:2603.04814v1 Table 3，49.00%，但日期/backbone/归属三项与行元数据不符）；2611.09807 全文深读（成对评测，四类 71.4% vs 64.6%、+6.8pp、p=5.5e-08，−34% tokens；temporal +25.5pp、adversarial −14.6pp；衰减公式 λ₀ 分类型；剪除=降级非删除；默认排序不读权重；<4pp 效应跨运行不可信）；MemoryDojo 全文深读（155 轨迹、五系统均可投毒，A-MEM BR 45.70%、Letta 历史前缀下 BRT 归零）；11-06 批次其余三篇仍 v1；复核戳：letta-code 仍 v0.34.48（窗口开始前约 2 小时发布）、PyPI 差距仍 36 版、DolphinBench PR #13/#14 仍 open、Mem0 93.4/94.4 未变、The Memory Wall / WALDEN 第三轮未定位（建议按主题标签处理）、五家版本戳全不变；arXiv 11-07–11-09 提交仍未进 API 索引，待 11-10 公告后补查。窗口内新稳定发布为零。
- 纠正：MemoryAtlas M5「各重跑行方法未逐项验证」（LiCoMemory 六行已定位，暂定拆分登记）；2611.09807「己方臂慢数倍」（作者此前说法错误并撤回，成对实测 p50/p90 两臂无差别）。
- 剩余未核实缺口：The Memory Wall / WALDEN 定位线索（作者/会议/产品）；Mem0 93.4 vs 94.4 变更；Mem0 双口径官方解释（仍缺失）；DolphinBench PR #13/#14 合并动向；MemoryAtlas A-MEM 48.38 行核查；Dual-Layer Agentic Memory arXiv:2608.22215 全文深读（对 ENGRAM）；arXiv 11-07–11-09 批次补查。

## 2026-11-10 凌晨轮（版本 0.21.0；窗口 2026-11-09 23:37 → 2026-11-10 05:37 CST）

- 本轨增量（M1–M6）：MemoryAtlas A-MEM 48.38 一手闭合为基准错配（数字出自 MIRIX Table 1 ScreenshotVQA 总分，LoCoMo 表无 A-MEM 行，MemoryAtlas 误列入 LoCoMo 节，不得引用；新线索：MIRIX Table 1 亦含 Mem0 66.88，但 Zep 75.14 与「71.2」全文 0 命中，「三数同源」假说证伪）；Dual-Layer 全文深读（arXiv:2608.22215v2；Write Router 86.35% EM 保留 Full Store 98.36%、升级 39.7%、SFT 零存 QA 38.02%→69.16%、1,752 例参数干扰退化；「剪除 68%」仅摘要级）；ENGRAM 对比（arXiv:2611.09472v1；核验放行 ΔF1 −0.2 [−0.6, 0.2]、269 可检索/331 归档，即时放行 −4.3pp；联合约束：合并 ≠ 可删，删除须自有证据）；letta-code v0.34.49 窗口内发布（2026-11-09T20:13:13Z，8 项变更，无记忆功能变更；PyPI 差距 36 → 37 版）；The Memory Wall / WALDEN 第四轮仍未定位，裁定按主题标签处理、停止穷举检索（除非新线索）；复核戳：Mem0 双口径未变（94.4%/94.8% vs 91.6/93.4，仍无官方调和）、DolphinBench PR #13/#14 仍 open、五家版本戳全不变、arXiv 11-07–11-09 批次留待下轮。
- 纠正：A-MEM 48.38「MemoryAtlas 重跑行待核查」（已闭合为基准错配，不得引用为 LoCoMo 分数）；「The Memory Wall / WALDEN 待定位论文」（裁定按主题标签处理，停止穷举检索）。
- 剩余未核实缺口：Mem0 93.4 vs 94.4 变更；Mem0 双口径官方解释（仍缺失）；DolphinBench PR #13/#14 合并动向；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-07–11-09 批次补查。

## 2026-11-10 中午轮（版本 0.22.0；窗口 2026-11-10 05:37 → 11:37 CST）

- 本轨增量（M1–M7）：Dual-Layer 作者数纠正（上一轮「独著」错误；arXiv API 显示 10 位作者，通讯作者 Weihang Pan）；DolphinBench 评论数纠正（PR #13/#14 各 2 条评论，作者自述 + vercel bot，非 0 条；作者称官方提交 Queued、「Validation could not finish」；Empirical 451/600 仍作者自报；PR 仍 open，updated_at 仍 2026-11-05）；MemoryAtlas 余项决定性反证（Hindsight 仓无竞品重跑数据、BEAM 帖 0 命中、MIRIX 同数假说证伪；Mem0 49 行元数据三项不符、Zep 研究页 0 命中；仍不得引用，暂定继续）；2611.09807 附录 A/B/E/F 深读（Table 3 出货配置：w_prune=0.1 逐字「demotion threshold」、gamma=0.1 抑制乘子、lambda/eta/rrf_k 等；Table 8 短语子句 adversarial +5.4pp vs temporal −4.1pp；答案类型门尚无成对 A/B）；排序信号 vs 遗忘三态综合（M8.4：四类信号 + 三态，联合约束「合并/固化 ≠ 可删」）；11-07 批次记忆论文摘要级登记 4 篇 + 10 篇相邻（Anchor 2611.10382、MixUA 2611.10439、TransMem 2611.10683、PatternBloom 2611.10477）；All-Mem arXiv:2603.19595v2 补漏；复核戳：窗口内发布零、letta-code 仍 v0.34.49（PyPI 差距仍 37 版）、Mem0 双口径未变、Dual-Layer 仍 v2 / ENGRAM 仍 v1（代码均未发布）、七项复现仍零新增、The Memory Wall / WALDEN 按主题标签处理。
- 纠正：Dual-Layer「独著」（作者实为 10 人）；DolphinBench「0 条评论」（实为各 2 条）。
- 剩余未核实缺口：Mem0 93.4 vs 94.4 变更；Mem0 双口径官方解释（仍缺失）；DolphinBench PR #13/#14 合并动向；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-08–11-10 批次补查；2611.09807 答案类型门成对 A/B。

## 2026-11-10 晚间轮（版本 0.23.0；窗口 2026-11-10 11:37 → 17:37 CST）

- 本轨增量（M1–M5）：窗口内新发布两件——letta-code v0.34.50（2026-11-10T06:07:58Z，npm 06:04:02Z，12 项变更）与 v0.34.51（2026-11-10T07:49:52Z，npm 07:43:53Z；均无记忆功能变更），PyPI letta 仍 0.34.12，差距 37 → 39 版（0.34.13–0.34.51 被 X.Y.0 门控挡住；无 0.35.0 tag；force_pypi 触发不可观测）；Mem0 双口径未变（README 94.4/94.8 vs 迁移页 93.4/91.6，仍无一句官方调和；SDK npm 3.3.1 / PyPI 2.2.1）；DolphinBench PR #13/#14 仍 open（评论各 2 条，review 评论 0，无维护者裁定；Empirical 75.2% 仍作者自报；官方提交 Queued 状态本轮未复查）；Dual-Layer 仍 v2 / ENGRAM 仍 v1（代码均未发布；ENGRAM 匿名链接仅 SPA 外壳）；2611.09807 仍 v1（答案类型门成对 A/B 仍无新证据；代码仓 HTTP 200）；Graphiti 仍 v0.31.2、Zep Cloud SDK 仍 4.1.0、LangMem 仍 0.0.30、Zep Agent Skills 论文仍 v1；记忆论文摘要级登记 10 篇（SoMem 2611.11929、MIRIX-2.0 2611.12068、2611.11840、SimplEvo 2611.11509、EASE-KV 2611.11299、2611.12412、TILA-CT 2611.11684、RADE 2611.11659、HiVe-Mem 2611.12872、TANSAR 2611.12374；2611.11820 硬件 Engram 消歧）；七项复现仍零新增；The Memory Wall / WALDEN 按主题标签处理；Anthropic 记忆文章 / Dreams API 本轮未复查。
- 剩余未核实缺口：Mem0 双口径官方解释（仍缺失）；DolphinBench PR #13/#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-08–11-10 批次补查；2611.09807 答案类型门成对 A/B；MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）仍不得引用，暂定继续。

## 2026-11-11 凌晨轮（版本 0.24.0；窗口 2026-11-10 17:37 → 2026-11-11 01:09 CST）

- 本轨增量（M1–M3）：letta-code 仍 v0.34.51（2026-11-10T07:49:52Z，窗口前发布；无 v0.34.52+；窗口后 #5628 落地约 6 分钟，无版本 bump，不计入）；PyPI letta 仍 0.34.12，差距仍 39 版；Mem0 双口径新增反证细节（评估页同样写「生产代表性模型栈」却配高位数字 92.5/94.4；两端差距 1.0/0.9 分恰在评估页自述 ±1 分 judge 置信区间内，但官方从未以此解释差异，不得写成已调和）；ENGRAM 匿名链接取回状态改变（http→https 301 后终点 HTTP 401，上轮 HTTP 200 SPA；不得写成代码下线）；Dual-Layer 仍 v2「upon acceptance」未发布；DolphinBench PR #13/#14 仍 open（各 2 评论、0 评审评论、无维护者评审；75.2% 仍作者自报）；2611.09807 仍 v1（答案类型门成对 A/B 仍无新证据）；arXiv 11-08–11-10 批次 totalResults 仍 25（11-10 批次待下轮补查；5 篇边界摘要排除；D-Mem 相邻登记）；七项复现仍零新增。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #13/#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-08–11-10 批次补查；2611.09807 答案类型门成对 A/B。

## 2026-11-11 凌晨轮（版本 0.25.0；窗口 2026-11-11 01:09 → 04:09 CST）

- 本轨增量（M1–M3）：新登记 EviCover（arXiv:2611.10170v1，2026-11-06；支持轮次召回 0.526→0.575，摘要级）；Memora 消歧护栏（基准 arXiv:2604.20006 vs 微软框架 arXiv:2602.03315）；DolphinBench 官网状态降格为未核查（根页 200、81,094 字节，`run` 路径 308；先前「官网已下线」措辞撤回）；复核戳：letta-code 仍 v0.34.51（2026-11-10T07:49:52Z，窗口前发布），PyPI 差距仍 39 版；Mem0 双口径未变（README 94.4%/94.8% vs 迁移页 91.6/93.4；评估页 92.5/94.4 配「生产代表性模型栈」；±1 分置信区间观察仍为执行者推算，不得写成已调和）；DolphinBench PR #13/#14 仍 open；Dual-Layer 仍 v2「upon acceptance」；ENGRAM 匿名链接仍 HTTP 401（连续两轮，不得写成代码下线）；2611.09807 仍 v1，代码库可达且自 2026-10-09 无变动；七项复现仍零新增；arXiv 11-10 批次结转下轮。窗口内新发布零。
- 纠正：DolphinBench「官网已下线」（根页 200 正常，状态降格为未核查）；MixUA 为重复发现（已于 2026-11-10 中午轮记入 layered-memory.md L133，不重复入技能）。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #13/#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-10 批次补查；2611.09807 答案类型门成对 A/B。

## 2026-11-11 07:09 轮（版本 0.26.0；窗口 2026-11-11 04:09 → 07:09 CST）

- 本轨增量（M1–M3）：DolphinBench PR #13/#14 仍 open（`updated_at` 仍 2026-11-05，无维护者评审）；官网状态仍未核查（根页 200、81,094 字节、0 关键词命中；/run 308；PR #13 评论数本轮未复核——GitHub API 限流）；复核戳：letta-code 仍 v0.34.51（窗口内仅 1 条新提交 #5633，遥测，与记忆无关）；PyPI letta 仍 0.34.12，差距仍 39 版；无 0.35.0 标签。Mem0 双口径数字全未变（README 94.4%/94.8% vs 迁移页 91.6/93.4；评估页 92.5/94.4 配「生产代表性模型栈」与 ±1 分置信区间），仍无一句点名调和的官方原话；Python 仍 2.2.1、TS 仍 3.3.1；README 提交哈希本轮因 GitHub API 限流未获复核（记为未核查）。Graphiti 仍 v0.31.2、Zep Cloud 仍 4.1.0、LangMem 仍 0.0.30；Zep Agent Skills 论文仍 v1。Dual-Layer 仍 v2「upon acceptance」未发布；ENGRAM 匿名链接仍 HTTP 401（连续三轮，不得写成代码下线）。2611.09807 仍 v1，答案类型门成对 A/B 仍无新证据，代码库自 2026-10-09 无变动。七项效果数字第三方独立复现本轮仍零新增。
- 方法注记：arXiv 范围查询 totalResults 25→24 为索引波动（2611.12872 HiVe-Mem 经 id_list 确认仍存在，v1）——永不单凭一次范围查询计数断言批次总量，须以公告日列表页 + id_list 交叉核对；同轮新现 8 个标题经 id_list 核查均为关键词误命中（硬件/数学/生物），已排除。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #13/#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-10 批次补查；2611.09807 答案类型门成对 A/B。

## 2026-11-11 10:09 轮（版本 0.27.0；窗口 2026-11-11 07:09 → 10:09 CST）

- 本轨增量（M1–M2、M4）：窗口内新发布零。letta-code 仍 v0.34.51（窗口内 4 提交 #5523/#5617/#5636/#5639，均为修复/杂项，与记忆无关；main 领先 v0.34.51 共 10 提交）；PyPI letta 仍 0.34.12，差距仍 39 版；无 0.35.0 标签；force_pypi 仍不可观测（记未核查）。Mem0 双口径三端全部未变（README 94.4%/94.8%、LoCoMo 92.5%/91.8%；迁移页 91.6/93.4；评估页 92.5/94.4），仍无一句点名调和的官方原话；README 提交哈希本轮因限流未获复核（记未核查）。DolphinBench PR #13/#14 均仍 open（评论数本轮因限流未重核）；官网仍 HTTP 200、81,094 字节（与上轮完全相同），`/run` 仍 308；75.2% 仍作者自报。Dual-Layer 仍 v2；ENGRAM 仍 v1，匿名链接仍 **HTTP 401（连续第四轮）**，仅记现象。arXiv 11-09–11-10 批次新增 2 篇（2611.13127、2611.13426 BioMem；其余 4 篇已于晚间轮入技能）；2611.09807 仍 v1；七项复现仍零新增。版本戳全部不变（Mem0 Python 2.2.1 / TS 3.3.1、Graphiti 0.31.2、Zep Cloud 4.1.0、LangMem 0.0.30、Zep Agent Skills 论文 2611.01386 仍 v1）。
- 未决冲突：arXiv 公告时刻 ET vs UTC——记忆腿主张 00:00 ET（05:00 UTC），Harness/技能 B 腿及先前轮次主张 00:00 UTC；证据不足，不采纳 ET 主张，不入技能正文，留待下轮核实。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #13/#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi）；arXiv 11-11 批次补查；2611.09807 答案类型门成对 A/B；arXiv 公告时刻 ET vs UTC 核实。

## 2026-11-11 13:09 轮（版本 0.28.0；窗口 2026-11-11 10:09 → 13:09 CST）

- 本轨增量（M1–M8）：窗口内新发布零。letta-code 仍 v0.34.51（窗口内 main 零提交，ahead_by 仍 10；PyPI letta 仍 0.34.12，差距仍 39 版；无 0.35.0 标签；`pushed_at` 04:20:57Z 移动成因未定位）。**force_pypi 缺口闭环**：`.github/workflows/release.yml` 确有 `force_pypi` 输入（L11）与两作业门控（L313/L324，`pypi_release == 'true' || inputs.force_pypi`）；最新运行 2026-11-10T07:37:42Z（0.34.51 bump），窗口内无运行、未触发。Mem0 双口径三端全部未变，仍无官方调和句；README 提交哈希核实为 4b61c5d（2026-05-13，闭合上轮限流缺口）。DolphinBench PR #13/#14 均仍 open（评论数核实各 2 条、0 评审，闭合上轮缺口）；官网仍 HTTP 200、81,094 字节，`/run` 跟随至 HTTP 200（96,768 字节），提交状态仍未核实（SPA）；75.2% 仍作者自报。Dual-Layer 仍 v2；ENGRAM 仍 v1，匿名链接仍 **HTTP 401（连续第五轮）**，仅记现象。arXiv 11-11 批次记忆论文摘要级登记（2611.14320 WikiTrace、2611.14017 个性化记忆 rubrics、2611.13946 TreeAgent、2611.14009 编码 agent 记忆后训练 II；相邻 4 篇；2611.13127 注释文本扩展，版本仍 v1）。排序/遗忘深化：七因子价值函数 2606.12945（盲态学习权重保留金证据 0.770±0.011 vs 均匀 0.657、最佳单因子 0.518、纯 recency 0.368；方法警示：对 held-out 评测问题打分会使保留率饱和至约 0.98——测检索非遗忘）+ 四篇新论文综合（WikiTrace→失效档、per-user discard、结构位置为排序信号、在策略蒸馏读写时机）；2605.08538/Field Guide/Memora 均为二手转述，仅本轨登记、不入技能。版本戳全部未变（Graphiti v0.31.2 GitHub 侧已核；2611.09807 代码仓 `pushed_at` 2026-10-09 已核；答案类型门成对 A/B 仍无新证据）；七项复现仍零新增；新二手线索 Maximem 73.8% vs 官方 93.4%（mnemoverse 转述，Maximem 为记忆厂商竞争方，未计入、待一手核对）。
- 纠正：arXiv 公告时刻 ET vs UTC 未决冲突已裁决——自验亲自复核官方 Availability 页：公告为 20:00 ET（11 月 = 次日 01:00 UTC），列表页约午夜 ET 刷新；10:09 轮「00:00 UTC announcement」措辞不精确（已在 skill-authoring.md 本轮增量首条纠正）；记忆腿谨慎立场得到支持。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #13/#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi 触发）；arXiv 11-12 批次补查；2611.09807 答案类型门成对 A/B；Maximem 73.8% 一手核对；MIRIX-2.0 合入状态。

## 2026-11-11 16:09 轮（版本 0.29.0；窗口 2026-11-11 13:09 → 16:09 CST）

- 本轨增量（M1–M7）：窗口内新发布零，letta-code 窗口内 1 条 main 提交 #5642（`ad308cd`，前台客户端工具调用超时；ahead_by 10→11；`pushed_at` 06:42:17Z 成因未定位）。PyPI letta 仍 0.34.12，差距仍 39 版；force_pypi 窗口内无触发。Mem0 双口径三端全部未变（字节数亦未变）。**DolphinBench 待审定 PR 现为 5 个**（#10–#14 全部 open，首次全量取回；#12 Graphon 84.0%、#10 AtMem 79.83%，均为作者自报）。Dual-Layer 仍 v2；ENGRAM 仍 v1，匿名链接仍 **HTTP 401（连续第六轮）**，仅记现象。**Maximem 73.8% 由二手升级为一手核实**（官方原文直读：57.5% → 73.8%，+16.3 点，官方主张 93.4%，差距 19.6 点；归因于 prompt 工程；不复现计数 2→3，其中 Maximem 一条现为一手；三条均为竞争方或非中立运行）。LLMSlim 竞争方实测新浮现（Mem0 OSS 2.2.1，LoCoMo 91.0%/91.6%，LongMemEval 60 题子集 76.7%，不计入阳性复现）；AgentOS 重跑待发布。2611.09807 仍 v1，成对 A/B 仍无新证据；七项复现仍零新增阳性。MIRIX-2.0 合入状态仍未确认（无 2.0 分支；`pushed_at` 06:37:36Z 成因未定位）。arXiv 11-11 批次 10 篇全部仍 v1、无遗漏；11-12 批次尚未公告（预期 11-13 01:00 UTC）；方法发现：export API 不索引未公告投稿（范围查询两式 totalResults 0）。版本戳全部未变（Mem0 2.2.1/3.3.1、Graphiti 0.31.2、Zep 4.1.0、LangMem 0.0.30、Zep Agent Skills 论文仍 v1）。
- 纠正：无（本轨腿文件无窗口归属错误；Maximem 由二手升级为一手，13:09 轮「二手线索未计入」口径已被取代）。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布时间（0.35.0 或 force_pypi 触发）；arXiv 11-12 批次补查；2611.09807 答案类型门成对 A/B；MIRIX-2.0 合入状态；LLMSlim harness 公开情况与 AgentOS 重跑结果。

## 2026-11-11 19:09 轮（版本 0.30.0；窗口 2026-11-11 16:09 → 19:09 CST）

- 本轨增量（M1–M6）：窗口内唯一新进展——**MIRIX-2.0 PR #153 合并**（README Results 节 + 雷达图，无代码改动；雷达分数论文自报；**未调和出入：论文 LongMemEval-S 72.6% vs 雷达图 70.0%**；仍无 2.0 分支/发布，2.0 代码合入状态仍未核查）。DolphinBench #10–#12 评论数首次登记（2/1/1，#13/#14 各 2，review_comments 全部 0）。**LLMSlim round 3 代码已公开、round 6 仍只聚合结果**；AgentOS「正在重跑」表述撤回（重跑状态改为未核查）。EviCover 摘要级登记（支撑召回 0.526→0.575、最终十轮 0.349→0.379；正确率增益未确立）。Mem0 三端未变；ENGRAM 401 连续第七轮；letta-code 零窗口提交，PyPI 差距仍 39 版；Dual-Layer 仍 v2；2611.09807 仍 v1；七项复现仍零新增阳性。方法债：arXiv `updated` 正则错位（历史值须以 ElementTree 重读）。
- 纠正：AgentOS「自己的 LongMemEval 重跑正在进行」撤回（本轮取回页面 0 命中，状态改为未核查）；LLMSlim round 6「协议/harness/逐题答案未公开」部分更新（round 3 代码已公开，round 6 仍未公开）。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布；arXiv 11-12 批次补查；2611.09807 成对 A/B；MIRIX-2.0 代码合入状态与 72.6/70.0 出入调和；LLMSlim round 6 harness 公开情况与 AgentOS 重跑状态。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。
## 2026-11-11 22:09 轮（版本 0.31.0；窗口 2026-11-11 19:09 → 22:09 CST）

- 本轮增量：FGSE 全文深读（证伪门控准入、阈值 η=0.03/δ=0.05/β=0.01、探针 8/16/16、380→104 提交、记忆域有害更新率最高 12.4%/13.1%、移除证伪探针 0.748→0.713；准入时遗忘为第五种遗忘位置、三统计量为准入排序信号）；Mem0 双口径新反证（自家博客引低位对 93.4/91.6；基准指南泛述不构成调和）；MIRIX 基线算术观察（两值同推最强基线 58.3，出入仅在自身分数；仍未调和）；DolphinBench PR #10–#14 评论数复核（2/1/1/2/2）；arXiv 11-11 批次 4 篇新记忆论文摘要级登记（FGSE 全文、SHARP、机器人时序捷径、Continual-ARC 相邻）。
- 纠正：2611.09807「answer-type gate / 无成对 A/B」（实为参考答案长度分层 Table 2，Table 1 已是成对 A/B）；AgentOS「自己的 LongMemEval 重跑正在进行」撤回为未核查（两文档页全文重读 "re-run" 0 命中）；DolphinBench 仓库路径纠正为 mem0ai/dolphinbench。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布；arXiv 11-12 批次补查；2611.09807 上下文 token 数出入核查（964/1,454 vs 963/1,461）；MIRIX-2.0 代码合入状态与 72.6/70.0 出入调和；LLMSlim round 6 harness 公开情况与 AgentOS 重跑状态。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。

## 2026-11-12 01:09 轮小节

- 本轮增量：2611.09807 上下文 token 数出入已定位并调和（scored (4) 964/1,454 vs all (5) 963/1,461；§5.4 引 all (5) 行；字符/4 近似、主张的是比率）——缺口闭合；Mem0 基准指南句现为一手直读（"74% and 92.5%…neither is wrong" 逐字确认），但仍不调和 91.6/93.4（页内 0 命中）。
- 复核戳：DolphinBench PR #10–#14 全部 open（零窗口内动向）；Dual-Layer 仍 v2「upon acceptance」未发布；ENGRAM 仍 v1，匿名代码链接连续第九轮 HTTP 401；letta-code 仍 v0.34.51，PyPI letta 仍 0.34.12，差距仍 39 版；MIRIX-2.0 仍无 2.0 分支与发布，72.6/70.0 出入仍未调和（基线一致观察继续有效）；LLMSlim round 6 仍只公开聚合结果；AgentOS 重跑状态仍未核查（不得恢复旧表述）；MemoryAtlas 无新增行（"49.00" 0 命中）；七项效果数字第三方独立复现仍零新增；版本戳全部未变（Mem0 PyPI 2.2.1 / npm 3.3.1、Graphiti 0.31.2、Zep 4.1.0、LangMem 0.0.30）；arXiv 11-12 批次尚未公告。
- 纠正：2611.09807 token 出入「未调和」→ 已调和（两行聚合，非笔误）。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态与 72.6/70.0 出入调和；LLMSlim round 6 harness 公开情况与 AgentOS 重跑状态；七项效果数字第三方独立复现；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。

## 2026-11-12 04:09 轮小节

- 本轮增量：letta-code v0.34.52 窗口内发布（15 项，无记忆功能变更；PyPI 差距 39→40 版）；Uteke 非复现线索第 4 条（LoCoMo 72.2% 严格重判 vs Mem0 官方 92.5%，差 20.3 点；博客准确率 ≠ 仓库召回指标；Mnemoverse 0.741/0.343 为二手转述）；2611.13760 递归记忆（预测 O(log 1/ε)、生成 O(log T/ε)，遗忘是功能）；2610.07701 准入门先行研究（摘要级，量化数字不得搬用）；FGSE 无公开代码仓库（摘要页 0 链接、全文 PDF github.com 0 命中）。
- 复核戳：Mem0 双口径三端点全未变，仍无调和句；DolphinBench PR #10–#14 全 open；Dual-Layer 仍 v2；ENGRAM 匿名链接 HTTP 401 第十轮；MIRIX-2.0 仍无分支/发布，72.6/70.0 仍未调和；LLMSlim 第 6 轮仍仅结果；AgentOS 重跑状态仍未核实；MemoryAtlas 基准页逐字节相同；五个版本戳未变；arXiv 11-12 批次尚未公告。
- 纠正：技能“不复现 Mem0 93.4% 的公开第三方实测”由 3 条更新为 4 条（新增 Uteke 72.2%）。
- 剩余未核实缺口：Mem0 双口径官方解释；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态与 72.6/70.0 出入调和；LLMSlim round 6 harness 公开情况与 AgentOS 重跑状态；七项效果数字第三方独立复现；Mnemoverse 原始分析直接取回；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。

## 2026-11-12 07:09 轮（版本 0.34.0；窗口 2026-11-12 04:09 → 07:09 CST）

- 本轨增量（M1–M6）：letta-code v0.34.53 窗口内发布（2026-11-11T22:20:50Z，2 项，无记忆功能变更；PyPI 差距 40→41 版）；Mem0 双口径机制一手验证通过（提交的 2026-04-06/07/08 运行文件逐数等于迁移页数字：LongMemEval top_200 467/500=93.40%、LoCoMo top_200 1410/1540=91.56%、top_50 90.4%/82.66%；头条 94.4/92.5 四分类分项全异，系另一轮未提交文件的运行；官方一句话调和仍缺失；措辞由「官方解释缺失」收窄为「机制已验证、官方表述仍缺失」）；2611.13760 全文深读（「forget」全文仅 2 命中，「遗忘是功能」为摘要层解读，非方法论术语）；arXiv 公告时刻勘误（04:09 轮批次混淆：11-11 提交日批次应为 2026-11-12 01:00 UTC，11-12 提交日批次才是 2026-11-13 01:00 UTC）；方法注记（grep 数字命中须用固定字符串 `grep -oF`）；复核戳：Zep 4.1.0 / Graphiti 0.31.2 / LangMem 0.0.30 无变化；DolphinBench PR #10–#14 全 open；ENGRAM 401 第十一轮；MemoryAtlas 174,071 字节逐字节相同，无新增行；Anthropic 记忆文未修订；七项复现仍零新增；15 篇已登记记忆论文版本复核全部未变。
- 纠正：Mem0 双口径「官方解释缺失」口径收窄为「机制已验证、官方表述仍缺失」；2611.13760「遗忘是功能」收窄为摘要层解读框架；arXiv 公告时刻批次混淆。
- 剩余未核实缺口：Mem0 双口径官方一句话调和（机制已验证，官方表述仍缺失）；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向（ENGRAM 401 连续十一轮）；PyPI letta 下次发布（差距 41 版）；MIRIX-2.0 代码合入状态与 72.6/70.0 出入调和；LLMSlim round 6 harness 公开情况与 AgentOS 重跑状态；七项效果数字第三方独立复现；2610.07701 / FGSE 全文深读；arXiv 11-11 批次补查（预期 2026-11-12 01:00 UTC 公告）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。

## 2026-11-12 10:09 轮

- 本轮增量（D-M01–D-M04、D-M06，落入 skill/references/layered-memory.md；D-M05 只登记于此，不入技能正文）：letta-code v0.34.54 窗口内发布（2026-11-12T00:56:13Z，7 项，无记忆功能变更；PyPI 差距 41 → 42 版；服务器仓仍 0.16.8）；Mem0 双口径三端点未变、官方一句话调和仍缺（机制已验证、官方表述仍缺失）；2610.07701 由摘要级升级为全文深读（随机准入对照协议、ΔFDR、t≈2.5 边界、超额收益必要条件、F1 被 DSR 覆盖冗余；quant 数字不得搬用）；FGSE 补充（无 Limitations 小节、未做等采用率随机对照、底座未提取）；复核戳：DolphinBench PR #10–#14 全 open、ENGRAM 401 第十二轮（仅记现象）、MIRIX-2.0 无 2.0 分支/发布（72.6/70.0 仍未调和）、LLMSlim round 6 仍只公开聚合结果、AgentOS 两页仅 JS bundle 哈希变化、MemoryAtlas 174,071 字节逐字节相同、七项复现仍零新增、15 篇论文全未变。
- 登记（D-M05，不入技能正文）：dev.to 作者 everest_an（2026-08-07）自述独立运行 Mem0 开源版 LongMemEval 得 32.4%——但记分表表头为 LongMemEval R@5（检索召回），与官方 93.4% 端到端准确率不是同一指标；只登记、不计入 4 条不复现记录，须一手核对 32.4% 到底是 R@5 还是准确率。
- 纠正：无（本轮无推翻项；2610.07701 由摘要级升级为全文深读）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和（机制已验证，官方表述仍缺失）；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向（ENGRAM 401 连续十二轮）；PyPI letta 下次发布（差距 42 版）；MIRIX-2.0 代码合入状态与 72.6/70.0 出入调和；LLMSlim round 6 harness 公开情况与 AgentOS 重跑状态；七项效果数字第三方独立复现（dev.to 32.4% 待核对线索 1 条）；FGSE 底座模型核对；arXiv 11-11 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。

## 2026-11-12 16:09 轮（13:09 轮补做轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：MIRIX-2.0 72.6/70.0 出入已调和（M1 70.0% vs M2 72.6%，缺口闭合）；MIRIX-2.0 全文深读（arXiv:2611.12068v1：LoCoMo 91.4%、LongMemEval-S M0 58.8 / M1 70.0 / M2 72.6、SHDocQA 83.0 / MHDocQA 71.0 / DetectiveQA 64.8 例外、ALFWorld 76.4%、MetaClaw 转引值、单轮运行与部分 LLM 裁判局限、建图摄入慢 3.8–4.0 倍；代码合入仍只到 README 结果）；letta-code v0.34.55/v0.34.56 尾窗两版（执行者亲自复核，均无记忆功能变更；PyPI 差距 42 → 44 版）；Mem0 双口径四端点未变（94.8% 出处精确化为指南页）；ENGRAM 纠正（不得续记第十三轮 401，计数停留第十二轮）；DolphinBench PR #10–#14 全 open（评论 2/1/1/2/2）、七项复现零新增。
- 纠正：MIRIX-2.0 "record both, do not reconcile" 旧口径撤回（正文内改写）；ENGRAM "401 连续第十三轮"不得记录（取回失败，无 HTTP 状态）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 44 版）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项效果数字第三方独立复现（dev.to 32.4% 待核对线索 1 条）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略）。

## 2026-11-12 19:09 轮（第 37 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：dev.to 32.4% 线索闭合（LongMemEval R@5 检索召回，非端到端准确率，不计入不复现；点值仅见开篇句、作者维护竞品 Awareness 非中立）；WikiTrace 全文深读（证据链接记忆三层 + 下潜核对，八套件平均 90.2（+10.7）、Regression 4.5%、52% 退化可追溯；无遗忘、靠用时核对；检索排序显式留白）；ENGRAM 匿名链接 HTTP 401 第十三轮（25 字节 `{"error":"not_connected"}`；16:09 轮无状态码不计入）；AgentOS 公告栏旧文纠正（所链博客为 2026-04-29 旧文；S 85.6%/M 70.2% 厂商自报；bench 代码公开，逐案 JSON 仅 LFS 指针）；letta-code 零发布零提交（仍 v0.34.56；PyPI 差距仍 44 版）、Mem0 四端点未变、DolphinBench PR #10–#14 全 open、MIRIX 仍无 2.0 分支/发布、LLMSlim round 6 仍仅结果、MemoryAtlas 逐字节相同、七项复现零新增（Mnemoverse ≈0.62 vs ≈0.79 为未核查线索，不计入）。
- 纠正：dev.to 32.4%"待核对"改为"已闭合为 R@5"；AgentOS 公告栏"新内容"改为 2026-04-29 旧文；ENGRAM"401 连续第十三轮"计数规则说明（16:09 轮无状态码不计入，本轮为第十三轮）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现（Mnemoverse 线索未核查）；M5 排序信号 vs 遗忘三态联合约束（WikiTrace 为第三种策略对照，仍暂定）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：WikiTrace 下潜核对机制与排序信号的关系）。

## 2026-11-12 22:09 轮（第 38 轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：Mnemoverse 线索闭合为不计入（≈0.62 vs ≈0.79 为自家引擎两轮 variant-s 运行，非 Mem0 复现；Mem0 ≈65.0 仅为引用行；裁判严格度警示作方法论注记；Recallium/Vertiso 厂商自报线索仅登记）；ENGRAM 匿名链接 HTTP 401 第十四轮（25 字节原文；不得写成代码下线）；letta-code 窗口内零发布零提交（仍 v0.34.56；PyPI 差距仍 44 版）、Mem0 四端点未变、DolphinBench PR #10–#14 全 open、16 篇论文全未变、MIRIX 仍无 2.0 分支/发布、LLMSlim round 6 仍仅结果、MemoryAtlas 逐字节相同、七项复现零新增。
- 纠正：WikiTrace“代码中无任何排序”过度表述（正文内改写，L223）：run 层 top-3 embedding 推荐确实存在于代码中（recommend.py，text-embedding-3-small，余弦 top k=3，平局按 step id 升序，写入 recommendations/<suite>.json，渲染为 `<similar_past_runs>` 块），论文未写；但排序不做过滤（render_inject() "Every insight inlined"，归档顺序）；下潜核对为 prompt 驱动的 agent 行为，非代码评分器；无遗忘机制代码级确认；M5 改写为排序信号 / 遗忘状态 / 用时核对三轴表述，下潜核对裁定为相邻证据（非直接证据），联合约束仍暂定。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现（Mnemoverse 已闭合不计入，四条不复现记录不变）；M5 排序信号 vs 遗忘三态联合约束（WikiTrace 相邻证据 + 三轴改写，仍暂定）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：run 层推荐与 insight 层排序的分离是否可推广）。

## 2026-11-13 01:09 轮（第 39 轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：letta-code 窗口内零发布零提交（仍 v0.34.56；PyPI 差距仍 44 版，语义版本重算确认）；Mem0 双口径四端点逐字节相同，官方调和句仍缺；ENGRAM 匿名链接 HTTP 401 第十五轮（25 字节原文；不得写成代码下线）；DolphinBench PR #10–#14 全 open；Dual-Layer 仍 v2、ENGRAM 仍 v1、FGSE 仍 v1；MIRIX 仍无 2.0 分支/发布；LLMSlim round 6 仍仅结果；MemoryAtlas 逐字节相同；七项复现零新增；深挖：ExpeL 同型分离设计（arXiv:2308.10144v2 §4.3 全量内联 + 轨迹层 Faiss kNN top-k；§6 自承规模上限；§4.2 投票删除机制）与 A-MEM 相反设计（arXiv:2502.12110v3 selective top-k 检索）一手确认，M5 联合约束继续暂定。
- 纠正：无（本轮无推翻项；PyPI 差距初算 52 版为字符串比较 artifact，已纠正为 44，未入正文）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（ExpeL/A-MEM 为相邻证据，三轴表述不变）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件）。

## 2026-11-13 04:09 轮（第 40 轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：ExpeL v2 → v3、A-MEM v3 → v11（引文已在新版全文逐字复核成立，实质不变）；Mem0 四端点字节变但句子集合与数字全同，系 Mintlify 重渲染，不得写成内容更新，官方调和句仍缺；ENGRAM 匿名链接第 16 次 401（25 字节原文）；复核戳：letta-code 仍 v0.34.56，PyPI 差距仍 44 版；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；七项复现零新增；DyadMem 删除侧定量锚点（op-F1 18.6%/24.5%、unsafe-deletion 84.1%/78.1%，论文自报）仅作 M5 相邻证据。
- 纠正：ExpeL/A-MEM 版本标签（v2→v3、v3→v11）；Mem0「字节变 = 内容更新」误读纠正（Mintlify 重渲染）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（DyadMem 删除侧锚点为相邻证据，三轴表述不变）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件）。

## 2026-11-13 07:09 轮（第 41 轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：Mem0 双口径口径收窄（指南页 "neither is wrong" 原句自至少上轮起即存在——正本 L23/L187 早已记录，记忆腿「纠正」降格为复核戳；针对 91.6↔92.5、93.4↔94.4 的点名调和仍缺失；评估页/迁移页 md5 变仅为 Cloudflare 邮箱哈希轮换，不得写成内容更新）；ENGRAM 匿名链接第 17 次 401（25 字节原文）；复核戳：letta-code 仍 v0.34.56（窗口内 1 提交，CI-only），PyPI 差距仍 44 版；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；七项复现零新增（causal-memory 页对 Maximem 73.8% 的二手引用不构成新复现）；MIRIX 仍只到 README 结果；深挖分离退化边界窗口内新证据零（Anatomy 综述为相邻证据）；M5 继续暂定。
- 纠正：Mem0「`neither is wrong` 0 命中」说法已由上轮记录证伪（本轮为再确认，非新纠正）；评估页/迁移页 md5 变「内容更新」误读纠正（Cloudflare 邮箱哈希轮换）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（Anatomy 综述与 causal-memory 厂商自报为相邻证据，三轴表述不变）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件）。

## 2026-11-13 10:09 轮（第 42 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code v0.34.57 窗口内发布（npm 2026-11-13T00:42:02Z；PyPI 差距 44 → 45 版）；Mem0 迁移页 +5,889 字节为 Mintlify 再部署（清洗词级 diff 全同，不得写成内容更新；"neither is wrong" 仍在；点名调和仍缺失）；ENGRAM 匿名链接第 18 次 401（25 字节原文）；相邻证据补登记 1 项 + 重述 1 项（CLADE 2611.14355 新登记；2610.31303 v4 已登记于 api-routing.md L280，本轮只作 M5 相邻证据重述）；复核戳：DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；MIRIX-2.0 仍只到 README 结果；七项复现零新增；M5 继续暂定。
- 纠正：「11-11 批次 10 篇无遗漏」仅对当时登记的记忆主论文成立，不覆盖相邻证据层（相邻证据补登记 1 项 + 重述 1 项）；2610.31303 v4「补登记」降格为重述（已登记于 api-routing.md L280）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（2610.31303 v4 与 CLADE 仅为相邻证据，三轴表述不变）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；下轮候选 2610.31303 v4 全文深读）。

## 2026-11-13 13:09 轮（第 43 轮）

- 本轮增量（D-M01–D-M08，落入 skill/references/layered-memory.md）：letta-code v0.34.58 窗口内发布（npm 2026-11-13T02:39:34Z；PyPI 差距 45 → 46 版）；含 2 个未发布记忆子系统提交（#5704 04:34:13Z、#5706 05:09:26Z 边界项）；Mem0 两页 md5 变属部署轮换（清洗词级 diff 全同，不得写成内容更新；"neither is wrong" 仍在；点名调和仍缺失）；ENGRAM 匿名链接第 19 次 401（25 字节原文）；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；七项复现零新增；2610.31303 v4 全文深读（恢复成本 vs 修订/重算成本轴；M5 交互点判定：不提供可检验观测）；arXiv 11-12 批次第三轮结转。
- 纠正：2611.14009 v1→v2（updated 2026-11-12T07:08:00Z；新增项目网站与作者 Pu Yao；上轮 API 返回仍为 v1，属过期读数，机制未证实；不属窗口内修订）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（2610.31303 v4 全文已深读：提供恢复/修订成本轴，不提供交互点可检验观测；分离退化边界量化值仍无）；M2 MemoryAtlas 余项（Hindsight 66.88/71.2、Mem0 49 归属）；arXiv 11-12 批次枚举。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架为起点，待 11-12 批次可枚举后筛查新论文；letta-code 两记忆提交 #5704/#5706 的发布收纳为下轮必查）。

## 2026-11-13 16:09 轮（第 44 轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：letta-code v0.34.59 窗口内发布（npm 06:10:38Z、GitHub 06:16:39Z；含 2 项记忆修复；#5704/#5706 已收纳，必查项闭环；新记忆提交 #5711 08:01:15Z 未发布；PyPI 差距 46 → 47 版）；Mem0 四端点内容未变（评估/迁移页 md5 变属部署轮换）；ENGRAM 第 20 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；七项复现零新增（连续第十三轮）；arXiv 11-12 批次记忆枚举完成（1,220 篇，10 主 + 5 相邻）；M5 深挖 5 相邻、0 直接，继续暂定；MIRIX-2.0 仍只到 README 结果。
- 纠正：无新推翻项（arXiv 11-12 批次枚举闭合，第三轮结转划除）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续）；MemoryAtlas 余项（M2 暂定继续）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架为起点，待 11-12 批次新论文深读）。

## 2026-11-13 19:09 轮（第 45 轮）

- 本轮增量（D-M01–D-M06，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.59（窗口内零发布、零提交；#5711 仍未发布；PyPI 差距仍 47 版）；Mem0 四端点内容未变（评估/迁移页 md5 变属部署轮换）；ENGRAM 第 21 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；七项复现零新增（连续第十四轮）；**M5：2611.16143 全文深读**（检索边际 −54% 为检索侧相邻证据，全文级；遗忘/排序词汇 0 命中，直接证据仍为零）；**方法发现：arXiv 摘要页字节数不可作内容变化信号**（Dual-Layer 摘要页 3 分钟内两次取回 44,604→44,018 字节，差异全在站点 chrome）；arXiv 11-13 批次尚不可枚举。**取证事故**：记忆腿 `_fetch.sh` 的 `cd` 误指 1609/m/，覆盖第 44 轮 `m/` 下 37 个数据文件 + `m-fetch-log.txt`（第 44 轮数值以其 `m-report.md` 为准；本轮脚本已修正；教训：取证脚本跨轮复用前须先改 `cd` 与窗口变量，下轮起自验时抽查取证脚本首行）。
- 纠正：M 腿「取证事故仅覆盖单次取回文件」→ 实际 37 个数据文件 + 日志（执行者亲自核对 1609/m/ mtime）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续）；MemoryAtlas 余项（M2 暂定继续）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架为起点，待 11-13 批次新论文深读）。

## 2026-11-13 22:09 轮（第 46 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.59（窗口内零发布、零提交；#5711 仍未发布；PyPI 差距仍 47 版）；Mem0 四端点内容未变（评估/迁移页各丢 18 个 U+200B 零宽空格 + 部署包轮换，不得写成内容更新）；点名调和仍缺；ENGRAM 第 22 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节不变（md5 变仅 dpl token）；七项复现零新增（连续第十五轮）；**M5：2611.17316 CoReMem 全文深读**（写侧相邻证据，全文级；forget/evict/sort/ranking/decay 全文 0 命中）+ 2611.17068 MAPE 摘要级相邻证据，**直接证据仍为零**；18 篇已登记 + 11-12 批次 10 篇全未变、2610.31303 仍 v4、Dual-Layer 仍 "upon acceptance"、MIRIX-2.0 仍只到 README 结果。
- 纠正：Mem0 指南页 "neither is wrong"「2 处」→ **5 处**（小写原形 2 + 句首大写 3，上轮仅计小写原形；文件逐字节相同，内容未变）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；CoReMem/MAPE 为相邻证据）；MemoryAtlas 余项（M2 暂定继续）；MAPE 全文深读（下轮）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架 + CoReMem 写侧估值为起点，待 11-13 批次新论文深读）。

## 2026-11-14 01:09 轮（第 47 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.59（窗口内零发布零提交；#5711 仍未发布；PyPI 差距仍 47 版）；Mem0 四端点内容未变（评估/迁移页 md5 变仅部署包轮换，清洗词级 diff 零真实改动；"neither is wrong" 仍 5 处；点名调和仍缺）；ENGRAM 第 23 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节不变（本轮 md5 亦相同，dpl token 未轮换）；七项复现零新增（连续第十六轮）；**M5：2611.17068 MAPE 全文深读**（设计发现侧相邻证据，升级为全文级；forget/evict/decay/demot/revok/delet/prun/merge 全文 0 命中；`consolidat` 2 命中均为他系统引用；排序只在检索阶段代码；作者自报 MAPE-PLCM 81.33%、+12.01pp、35% 服务成本；**直接证据仍为零**，为第三项成本侧相邻证据）；18 篇已登记 + 11-12 批次 10 篇全未变、2610.31303 仍 v4、Dual-Layer 仍 "upon acceptance"（摘要页字节数再证不是内容信号）、MIRIX-2.0 仍只到 README 结果。
- 纠正：Mem0「评估/迁移页 md5 变 = 内容更新」→ 部署包轮换，零真实改动（清洗词级 diff 为证）；摘要页字节数不得作内容变化信号（本轮第三例：Dual-Layer 摘要页 +1,159 字节，可见词差仅 +16 词，均为页脚资助信息）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；MAPE 为第三项成本侧相邻证据，直接证据仍为零）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架 + CoReMem 写侧估值 + MAPE 成本侧相邻证据为起点，待 11-13 批次新论文深读）。

## 2026-11-14 04:09 轮（第 48 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：**letta-code v0.34.60 窗口内发布**（GitHub 18:12:34Z，npm 18:07:24Z；含记忆修复 #5711，**上轮必查项闭环**；窗口内另有 4 提交，均与记忆无关；PyPI 差距 47 → 48 版）；Mem0 四端点正文全未变（README 与指南页逐字节相同；评估/迁移页字节数下降 −5,889/−3,771，严格可见文本比对证明仅 Mintlify 部署 JSON + 代码高亮 token 化 + JSX 序列化残留，无正文句子增删；"neither is wrong" 仍 5 处；点名调和仍缺）；ENGRAM 第 24 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同（含 dpl token 未轮换）；七项复现零新增（连续第十七轮）；**M5：2611.16590 AstroMemBench 全文深读**（直接 0、相邻 0；遗忘/排序词汇全文整词 0 命中；仅新增基准参照，代码仓已验存在；**直接证据仍为零**）；18 篇已登记 + 11-12 批次 10 篇全未变、2610.31303 仍 v4、Dual-Layer 仍 "upon acceptance"（摘要页字节变仅页脚 chrome 16 词，本轮第四例）、MIRIX-2.0 仍只到 README 结果。
- 纠正：Mem0「评估/迁移页字节下降 = 内容更新」→ Mintlify 再部署，严格可见文本比对零真实改动（本轮首次出现字节数下降而非仅 md5 变化的再部署，仍属同一机制）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 48 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；AstroMemBench 直接/相邻均为 0）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第四次结转）；11-12 批次余 6 篇深读（下轮优先 2611.16378）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架 + CoReMem 写侧估值 + MAPE 成本侧相邻证据为起点，AstroMemBench 仅作基准参照、不计入 M5 证据；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查改为 PyPI 线是否跟进 0.34.60 / 是否出现 0.35.0）。

## 2026-11-14 10:09 轮（第 49 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：letta-code 无新发布（仍 v0.34.60，发布于 2026-11-13T18:12:34Z，窗口外）；窗口内 5 提交，**2 项记忆相关未发布**：**#5712** `fix(memory): use real agent IDs when listing attached repositories`（aeca693，2026-11-14T01:07:19Z）与 **#5351** `feat(local): add self-compaction modes to the local backend`（805aee6，2026-11-14T00:52:00Z，新增自压缩模块）；PyPI `letta` 差距仍 48 版（0.34.12 vs npm 0.34.60）；Mem0 四端点正文无变化（README 与指南页逐字节相同；评估/迁移页字节变仅 Mintlify 再部署 + 代码高亮 token 化，严格可见文本比对为证；"neither is wrong" 仍 5 处；点名调和仍缺）；ENGRAM 第 25 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同；七项复现零新增（**连续第十八轮**）；**M5：2611.16378 全文深读**（10,281 词，准入侧相邻证据：EGE 证据治理接纳门，adopt 25 / reject 9 / rollback 11；forget/evict/sort/decay/demot/revok/delet/prun/merge/consolidat 全文 0 命中，ranking 1 处仅方法排序句）与 **2611.16386 CGEM 全文深读**（5,204 词，遗忘侧相邻证据：有界记忆库 + 可删除性充分条件；"Removing obsolete failure records raises SR from 47.7% to 88.0% at N = 10" 原文已定位，论文自报；delet 8、evict 1、consolidat 9）；其余 4 篇（16223/16384/16487/16589）直接/相邻均为 0；**联合约束仍为执行者综合，直接证据仍为零，继续暂定**；arXiv 11-13 批次第五次结转（记忆轨口径），机制已纠正（官方 Availability 页：周五提交 → 周一 20:00 ET = UTC 周二 01:00 公告；M 腿「11-17 00:00 UTC」差一小时，纠正为 **11-17 01:00 UTC**）；18 篇已登记 + 11-12 批次 10 篇全未变。
- 纠正：M 腿「11-17 00:00 UTC 公告」→ 官方时刻表一手闭合为 11-17 01:00 UTC（20:00 ET = 次日 01:00 UTC）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 48 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；16378 准入侧 + 16386 遗忘侧两篇相邻证据，直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第五次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；11-12 批次余 6 篇深读（16378/16386 已完成）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架 + CoReMem 写侧估值 + MAPE 成本侧相邻证据 + 16378 准入侧 + 16386 遗忘侧相邻证据为起点；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 #5712/#5351 是否收纳进新版）。

## 2026-11-14 13:09 轮（第 50 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**letta-code v0.34.61 窗口内发布**（GitHub 2026-11-14T02:54:27Z，npm 02:49:41Z；执行者 npm 直查复核；9 项正文，含上轮 2 项记忆相关未发布提交 **#5712** fix(memory) aeca693 与 **#5351** 本地自压缩 805aee6，**上轮必查项闭环**）；PyPI `letta` 差距 **48 → 49 版**（0.34.12 vs npm 0.34.61；下次仍预期 0.35.0 或 force_pypi）；Mem0 四端点正文无变化（README 与指南页逐字节相同；评估/迁移页字节变仅 Mintlify buildId 轮换 dpl_715Y… → dpl_G2Qd…，严格可见文本比对为证，可见词数 4,246 / 4,336 与上轮完全相同；"neither is wrong" 仍 5 处；点名调和仍缺）；ENGRAM 第 26 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节逐字节相同（dpl token 未轮换）；七项复现零新增（**连续第十九轮**）；**M5：11-12 批次深读覆盖闭合 10/10**，4 篇余篇计数重跑与上轮逐数一致（16223/16384/16487/16589 直接/相邻均为 0），2611.16589 的 merge 1 处记为最接近未遂项，**联合约束仍为执行者综合，直接证据仍为零，继续暂定**；arXiv 11-13 批次第六次结转（记忆轨口径），公告档 UTC 11-16 01:00 与 11-17 01:00；18 篇已登记 + 11-12 批次 10 篇全未变。
- 纠正：无（本轮记忆轨无推翻旧说法；M 腿「11-17 00:00 UTC」已于 49 轮纠正为 11-17 01:00 UTC，本轮沿用）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；深读覆盖 10/10 已闭合，直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第六次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；以 ExpeL v3 §6 自承 + 2610.31303 v4 恢复/修订成本框架 + CoReMem 写侧估值 + MAPE 成本侧相邻证据 + 16378 准入侧 + 16386 遗忘侧相邻证据为起点；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.61 / 是否出现 0.35.0）。

## 2026-11-14 16:09 轮（第 51 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.61，窗口内 0 提交，PyPI 差距仍 49 版；Mem0 四端点正文无变化（本轮 md5 变化成因：Mintlify manifest 哈希轮换 + cfemail 重混淆；dpl buildId 本轮未轮换）；ENGRAM 第 27 次 401；MemoryAtlas 逐字节相同；七项复现连续第二十轮零新增；M5 直接证据仍为零（11-13 批次未公告，无新论文可筛查；深读覆盖上轮已闭合 10/10）。
- 纠正：无（本轮记忆轨无推翻旧说法；M 腿分析脚本两处小修正已记录；XML 文件级 md5 差异唯一成因为 feed 级查询时刻戳，今后一律取条目级 <updated>）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；深读覆盖 10/10 已闭合，直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第七次结转，记忆轨口径；台账累计第八次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.61 / 是否出现 0.35.0）。

## 2026-11-14 19:09 轮（第 52 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.61，窗口内 1 提交 a24b294（与记忆无关、未发布），PyPI 差距仍 49 版；Mem0 四端点正文无变化（本轮 md5 成因：仅 cfemail 重混淆，无 manifest 哈希轮换；dpl buildId 未轮换）；ENGRAM 第 28 次 401；MemoryAtlas 逐字节相同；七项复现连续第二十一轮零新增；**M5：六项相邻证据边界复核完成**（ExpeL v3 §6 与 2610.31303 v4 原文复验逐字成立），六项覆盖六侧面但无同一 workload 双层测量——退化边界仍无 N*、无 token 数、无观测点；可证伪化方向已登记为提案（同一 workload 上同时报告写层升级率与检索层 top-k 截断随库规模的两条曲线）；直接证据仍为零，继续暂定。
- 纠正：无（本轮记忆轨无推翻旧说法）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第八次结转，记忆轨口径；台账累计第九次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.61 / 是否出现 0.35.0）。

## 2026-11-14 22:09 轮（第 53 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.61（窗口内 0 提交，最新仍 a24b294，与记忆无关、未发布）；PyPI letta 仍 0.34.12，差距仍 49 版；Mem0 四端点正文无变化（本轮 md5 成因已逐行定位：仅 cfemail 重混淆，无 manifest 哈希轮换；dpl buildId 未轮换）；ENGRAM 第 29 次 401；DolphinBench PR #10–#14 全 open；**MemoryAtlas 174,071 字节、md5 首次变化**（dpl token 轮换 + chunk 哈希文件名轮换，归一化后逐字节相同、可见文本 1,022 词相同，不是内容更新）；七项复现零新增（连续第二十二轮）；**M5：本轮新直接证据为零**，联合约束继续暂定（52 轮边界复核结论与可证伪化提案维持不变）；已登记 18 篇 + 11-12 批次 10 篇 + 2610.31303（仍 v4）全部未变。
- 纠正：MemoryAtlas「逐字节相同」表述自本轮起停用，改为「字节数不变、可见内容不变、md5 变仅部署噪声（dpl token + chunk 哈希轮换）」。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第九次结转，记忆轨口径；台账累计第十次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.61 / 是否出现 0.35.0）。

## 2026-11-15 01:09 轮（第 54 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.61（窗口内 0 提交，最新仍 a24b294，与记忆无关、未发布；无 0.34.62/0.35.0）；PyPI letta 仍 0.34.12，差距仍 49 版（npm 0.34.x 补丁版重数为 62）；Mem0 四端点正文无变化（README 与指南逐字节相同；评估/迁移页 md5 变仅 cfemail 重混淆，dpl buildId 未轮换）；ENGRAM 第 30 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与上轮相同，归一化后相同、可见文本 1,022 词相同；七项复现零新增（连续第二十三轮）；**Dual-Layer 摘要页字节数回摆**（44,018 → 44,604 字节、可见词 829 → 844，页脚 chrome 翻回，非内容变化；「摘要页字节数非内容信号」模式第七例，恰好逆转 53 轮变化）；**M5：本轮新直接证据为零**，联合约束继续暂定（11-13 批次仍未公告，无新论文可筛查；六项相邻证据全部未变；52 轮边界复核结论与可证伪化提案维持不变）；已登记 18 篇 + 11-12 批次 10 篇 + 2610.31303（仍 v4）全部未变。
- 纠正：本轮无新纠正（Dual-Layer 字节数回摆为页脚轮换第七例，非内容变化，不构成对既往口径的修正；53 轮 MemoryAtlas 表述口径修正维持不变）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi；a24b294 是否落入 0.34.62 或更高）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十一次结转，记忆轨口径；台账累计第十一次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.61 / 是否出现 0.35.0）。

## 2026-11-15 04:09 轮（第 55 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.61（窗口内 0 提交，最新仍 a24b294，与记忆无关、未发布；无 0.34.62/0.35.0；执行者亲自查 scoped 包 @letta-ai/letta-code latest 复核）；PyPI letta 仍 0.34.12，差距仍 49 版；Mem0 四端点正文无变化（README 逐字节相同；评估页 −5,889 B 为 kill-isr 开关配置清空、迁移页字节不变、指南页 +1,077 B；**指南页 Framer 于窗口内 19:33Z 重新发布**，行级 diff 7 处、剥离标签后文本差为 0；可见词四数全同）；ENGRAM 第 31 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与 53/54 轮相同，归一化后相同、可见文本 1,022 词相同；七项复现零新增（连续第二十四轮）；**Dual-Layer 摘要页字节数再翻转**（44,604 → 43,445 字节、可见词 844 → 828，仅页脚资助行删除，非内容变化；「摘要页字节数非内容信号」模式第八例，恰好逆转 54 轮变化）；**M5：本轮新直接证据为零**，联合约束继续暂定（六项相邻证据全部未变；52 轮边界复核结论与可证伪化提案维持不变）。
- 纠正：本轮无新纠正（Dual-Layer 字节数再翻转为页脚轮换第八例，非内容变化，不构成对既往口径的修正；执行者自验初版误取未加 scope 的 npm 包 `letta-code`（仅 1.0.0 版）与 M 腿证据矛盾，核查取证脚本后纠正为 scoped 包 @letta-ai/letta-code，0.34.61 成立，见 verification.md §4）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi；a24b294 是否落入 0.34.62 或更高）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十二次结转，记忆轨口径；台账累计第十二次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.61 / 是否出现 0.35.0）。

## 2026-11-15 07:09 轮（第 56 轮）

- 本轮增量（D-M01–D-M05，落入 skill/references/layered-memory.md）：letta-code 仍 v0.34.61（窗口内新提交 1 项：21f39b9 22:53:28Z，`fix(listener): retry interrupted-turn writes with disk reserve (#5734)`，改动全在 `src/websocket/listener/`，与记忆无关、未发布；a24b294 同样未发布；无 0.34.62/0.35.0；执行者亲自查 npm 复核）；PyPI letta 仍 0.34.12，差距仍 49 版（npm 0.34.x 共 62 版）；Mem0 四端点正文无变化（评估/迁移页 md5 变定位为 cfemail 混淆串重随机化，可见词 4,246/4,336 全同；指南页逐字节相同）；ENGRAM 第 32 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与 53/54/55 轮相同，归一化后相同、可见文本 1,022 词相同；七项复现零新增（连续第二十五轮）；**Dual-Layer 摘要页字节数再翻转**（43,445 → 44,018 字节、可见词 828 → 829，仅 arXiv 站点 chrome 三处轮换，非内容变化；「摘要页字节数非内容信号」模式第九例，恰好逆转 55 轮变化）；**M5：本轮新直接证据为零**，联合约束继续暂定（六项相邻证据全部未变；52 轮边界复核结论与可证伪化提案维持不变）。
- 纠正：本轮无新纠正（Dual-Layer 字节数再翻转为站点 chrome 轮换第九例，非内容变化，不构成对既往口径的修正）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 49 版；下次仍预期 0.35.0 或 force_pypi；21f39b9 是否落入 0.34.62 或更高）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十三次结转，记忆轨口径；台账累计第十三次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 21f39b9 是否收纳进 0.34.62 或更高版、PyPI 线是否跟进）。

## 2026-11-15 10:09 轮（第 57 轮）

- 窗口内新发布 1 件：letta-code v0.34.62（GitHub 01:16:48Z，npm 01:12:15Z；执行者亲自复核）；发布说明 7 项，含 #5734（21f39b9）——上轮必查项闭环；窗口内 5 提交全部收纳进 0.34.62。PyPI letta 差距 49 → 50 版（仍 0.34.12；npm 0.34.x 共 63 版）。
- Mem0 四端点正文无变化（cfemail 混淆串重随机化；可见词 4,246/4,336 全同）；"neither is wrong" 仍 5 处；双口径调和仍缺。
- ENGRAM 第 33 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与 53–56 轮相同；七项复现零新增（连续第二十六轮）；M2 余项仍未定位。
- Dual-Layer 摘要页字节数再翻转（44,018 → 43,445 B、829 → 828 词，仅站点 chrome 三处轮换；「摘要页字节数非内容信号」模式第十例，恰好逆转 56 轮变化）。
- M5：本轮新直接证据为零，联合约束继续暂定。
- 纠正：本轮无新纠正（Dual-Layer 字节数再翻转为站点 chrome 轮换第十例，非内容变化，不构成对既往口径的修正）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi；a24b294 收纳状态可经 compare 接口核对）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十四次结转，记忆轨口径；台账累计第十四次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.62、a24b294 收纳状态经 compare 接口核对）。

## 2026-11-15 13:09 轮（第 58 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：窗口内新发现 0 件；letta-code 仍 v0.34.62（执行者亲自查 npm 复核）；PyPI letta 差距仍 50 版；**a24b294 收纳状态闭环**（compare 接口 7 提交，57 轮未核实小项）；Mem0 四端点正文无变化（评估/迁移页 md5 变定位为 cfemail 混淆串重随机化）；ENGRAM 第 34 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与 53–57 轮相同；七项复现零新增（连续第二十七轮）。
- Dual-Layer 摘要页字节数再翻转（43,445 → 44,018 B、828 → 829 词，md5 与 56 轮逐字节相同，仅站点 chrome 三处轮换；「摘要页字节数非内容信号」模式第十一例，恰好逆转 57 轮变化）。
- M5：本轮新直接证据为零，联合约束继续暂定。
- 纠正：本轮无新纠正（a24b294 闭环为 57 轮小项的完成，非对既往口径的修正）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十五次结转，记忆轨口径；台账累计第十五次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.62）。

## 2026-11-15 16:09 轮（第 59 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：窗口内新发现 0 件；letta-code 仍 v0.34.62（执行者亲自查 npm 复核）；PyPI letta 差距仍 50 版；Mem0 四端点正文无变化（评估/迁移页 md5 变定位为 cfemail 混淆串第三度重随机化）；ENGRAM 第 35 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与 53–58 轮相同；七项复现零新增（连续第二十八轮）。
- Dual-Layer 摘要页字节数再翻转（43,445 → 44,018 B、828 → 829 词，md5 与 57 轮逐字节相同，仅站点 chrome 三处轮换；「摘要页字节数非内容信号」模式第十二例，恰好逆转 58 轮变化）。
- M5：本轮新直接证据为零，联合约束继续暂定。
- 纠正：本轮无新纠正（Dual-Layer 字节数再翻转为站点 chrome 轮换第十二例，非内容变化，不构成对既往口径的修正）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十六次结转，记忆轨口径；台账累计第十六次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.62）。

## 2026-11-15 19:09 轮（第 60 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**窗口内新发现 1 件——letta-code 新增 2 项提交**（`4a03c26` #5742、09:17:06Z `57d61bd` #5741，尚未随任何发布上架），仍 v0.34.62（执行者亲自查 npm 复核 dist-tags.latest）；PyPI letta 差距仍 50 版；Mem0 四端点正文无变化（评估/迁移页 md5 变定位为 cfemail 混淆串第四度重随机化，可见词全同）；ENGRAM 第 36 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与 53–59 轮相同；七项复现零新增（连续第二十九轮）；**Dual-Layer 摘要页本轮未再翻转**（43,445 字节、md5 与 59 轮逐字节相同；「摘要页字节数非内容信号」模式仍为第十二例）；M5 新直接证据为零，联合约束继续暂定。
- 纠正：本轮无新纠正（Dual-Layer 摘要页未再翻转，模式计数不变，不构成对既往口径的修正）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十七次结转，记忆轨口径；台账累计第十七次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.62）。

## 2026-11-15 22:09 轮（第 61 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**窗口内新发现 0 件**——letta-code 仍 v0.34.62（执行者亲自查 npm 复核 dist-tags.latest）；**窗口内提交 0 项**；60 轮新增的 2 项提交（`4a03c26` #5742、`57d61bd` #5741）**仍未随任何发布上架**；PyPI letta 差距仍 50 版；Mem0 四端点正文无变化（评估/迁移页 md5 变定位为 cfemail 混淆串第五度重随机化，可见词全同；**指南页窗口内 Framer 再发布** 13:43/13:55Z，剥离标签后正文逐字相同，不得写成内容更新）；ENGRAM 第 37 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 变仅因 dpl token 轮换（归一化后逐字节相同，可见词 1,022 全同）；七项复现零新增（连续第三十轮）；**Dual-Layer 摘要页再翻转（第十三例）**（43,445 → 44,018 字节、md5 与 56 轮逐字节相同，仅 arXiv 站点 chrome 三处轮换，恰好逆转 59/60 轮变化）；MIRIX-2.0 仍只到 README 结果（stars 3464 → 3465 为星标事件，非代码合入）；M5 新直接证据为零，联合约束继续暂定。
- 纠正：本轮无新纠正（Dual-Layer 摘要页再翻转，模式计数第十三例；指南页再发布已明确区分为部署噪声、非内容更新）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十八次结转，记忆轨口径；台账累计第十八次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.62）。

## 2026-11-16 01:09 轮（第 62 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**窗口内新发现 0 件**——letta-code 仍 v0.34.62（执行者亲自查 npm 复核 dist-tags.latest，`@letta-ai/letta-code`）；**窗口内提交 0 项**；60 轮新增的 2 项提交（`4a03c26` #5742、`57d61bd` #5741）**仍未随任何发布上架**；PyPI letta 差距仍 50 版；Mem0 四端点正文无变化（评估/迁移页 md5 变定位为 cfemail 混淆串第六度重随机化，可见词全同；**指南页连续第二轮窗口内 Framer 再发布** 16:39:19Z，+11 字节，剥离标签后正文逐字相同，不得写成内容更新）；ENGRAM 第 38 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与上轮完全相同（本轮无轮换）；七项复现零新增（连续第三十一轮）；**Dual-Layer 摘要页再翻转（第十四例）**（44,018 → 43,445 字节、md5 回到 59/60 轮值，仅 arXiv 站点 chrome 三处逆向）；MIRIX-2.0 仍只到 README 结果（连续第二轮窗口内星标事件，stars 3465 → 3466，`updated_at` 17:07:38Z，窗口关闭前约 1.5 分钟；`pushed_at`/分支/发布/最新提交全未变——星标事件，非代码合入）；M5 新直接证据为零，联合约束继续暂定。
- 纠正：本轮无新纠正（Dual-Layer 摘要页再翻转，模式计数第十四例；指南页再发布已明确区分为部署噪声、非内容更新）。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第十九次结转，记忆轨口径；台账累计第十九次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 PyPI 线是否跟进 0.34.62）。

## 2026-11-16 04:09 轮（第 63 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**窗口内新发现 0 件**——letta-code 仍 v0.34.62（执行者亲自查 npm 复核 dist-tags.latest，`@letta-ai/letta-code`）；**窗口内提交 0 项**；60 轮新增的 2 项提交（`4a03c26` #5742、`57d61bd` #5741）**仍未随任何发布上架**；**关窗后 v0.34.63 发布**（npm 21:17:20Z、GitHub 21:23:29Z，收纳 #5741/#5742/#5744，执行者亲自查 GitHub API 复核，供下轮核对）；PyPI letta 差距仍 50 版；Mem0 四端点正文无变化（评估/迁移页 cfemail 第七度重随机化，可见词全同；**指南页本轮无 Framer 再发布**，文件逐字节相同，「连续第三轮窗口内再发布」不成立；指南页数字戳计数口径：原始文件全文 grep 92.5×8 / 94.4×10 / 94.8×6，可见词口径 92.5×3 / 94.4×4 / 94.8×2）；ENGRAM 第 39 次 401；DolphinBench PR #10–#14 全 open；MemoryAtlas 174,071 字节、md5 与上轮完全相同（连续第二轮同一值）；七项复现零新增（连续第三十二轮）；**Dual-Layer 摘要页再翻转（第十五例）**（43,445 → 44,018 字节、md5 回到 61 轮值，仅 arXiv 站点 chrome 三处逆向）；MIRIX-2.0 仍只到 README 结果（连续第三轮窗口内星标事件，stars 3466 → 3467，`updated_at` 20:06:14Z，窗口关闭前约 3 分钟；`pushed_at`/分支/发布/最新提交全未变——星标事件，非代码合入）；M5 新直接证据为零，联合约束继续暂定。
- 纠正：M 腿「指南页数字戳（92.5×8 / 94.4×10 / 94.8×6）用可见词与剥离标签两种口径均无法复现」之说不成立——执行者亲自对本轮指南页文件（与 62 轮逐字节相同）做原始文件全文 grep，逐数复现 8/10/6；62 轮数字戳成立，计数口径为原始文件全文 grep；M 腿的 14/127/10 未说明所 grep 的文件与方法，不得引用。
- 剩余未核实缺口：Mem0 双口径官方一句话调和；DolphinBench PR #10–#14 审定；Dual-Layer/ENGRAM/FGSE 代码与数据集发布动向；PyPI letta 下次发布（差距 50 版；下次仍预期 0.35.0 或 force_pypi）；MIRIX-2.0 代码合入状态；LLMSlim round 6 与 AgentOS 重跑状态；七项复现；M5 排序信号 vs 遗忘三态联合约束（暂定继续；边界复核完成、可证伪化方向已登记）；MemoryAtlas 余项（M2 暂定继续）；arXiv 11-13 批次枚举（第二十次结转，记忆轨口径；台账累计第二十次，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）。
- 下轮起点：先清上述缺口，再按子主题轮转（检索排序信号与遗忘策略：分离设计随规模退化为两层皆排序的边界条件；待 11-13 批次可枚举后筛查新论文；letta-code 下轮必查 v0.34.63 收纳确认与 PyPI 线是否跟进）。

## 2026-11-16 07:09 轮（第 64 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**letta-code v0.34.63 已发布**（2026-11-15T21:23:29Z，轮间补漏；执行者亲自查 GitHub API 复核），收纳 #5741/#5742，缺口闭合；三项实质改动均非记忆相关；窗口内提交 0 项；尚无 v0.34.64+；PyPI `letta` 仍 0.34.12，差距 50 → 51 版；Mem0 四端点正文无变化（评估/迁移页 cfemail 第八度重随机化；**指南页连续第二轮逐字节相同、无 Framer 再发布**，执行者亲自取回 1,032,588 字节、md5 与 M 腿文件逐字节相同）；指南页数字戳全文 grep 实测 92.5×8 / 94.4×10 / 94.8×6，与 62 轮戳一致；ENGRAM 第 40 次 401（执行者关窗后确认：anon 链接 302 跳转至 API 后回 401；支持 M 腿计数，不得再加一次）；七项复现零新增（**连续第三十三轮**）；**Dual-Layer 摘要页再翻转，第十七例**（44,604 → 43,445 字节；执行者关窗后观测：可见文本仅末尾删除 "Major funding support from" 与三处 "(opens in new tab)"，可见行 121 → 117，属站点 chrome 变化、非内容变化；发生于 M 腿取回 23:14Z 与执行者取回约 02:55Z 之间，精确时刻不明，窗口内/外归属不确定）；MIRIX 本轮无新元数据变化（stars 仍 3467，updated_at 仍 2026-11-15T20:06:14Z，即 63 轮窗口内值）；MemoryAtlas 174,071 字节、md5 与上轮完全相同（62 轮起连续第三轮同一值）；M2 余项（Hindsight 66.88/71.2、Mem0 49 归属）仍未定位。
- M5：本轮新直接证据为零（六项相邻证据全未变；11-13 批次经 RB 腿 API 枚举，CarryOn 2611.17808 为上下文保留相邻、非直接证据）；联合约束继续暂定；已登记 18 篇 + 11-12 批次 10 篇论文 + 2610.31303 版本复核全未变；DolphinBench PR #10–#14 全 open；LLMSlim/AgentOS 两页逐字节相同；letta 服务器仓仍停滞（0.16.8）；arXiv 11-13 批次第二十一次结转（记忆轨口径，双重口径见 api-routing-papers.md）。
- 纠正：M 腿报告为 23:14–23:15Z 主取证草稿（晚段补取挂起的后台任务未执行，磁盘上无 `m-late-*` 文件），晚段补取由执行者亲自完成（letta 发布、指南页、ENGRAM 均已复核）；自验发现 D-M03 两处错误并已修正（MIRIX「连续第三轮窗口内星标事件」→ 本轮无新元数据变化，M 腿已重取；MemoryAtlas「连续第二轮同一值」→ 连续第三轮同一值，62 轮起），见 verification.md §6。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.63 / 是否出现 0.35.0；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合约束（暂定继续）；MemoryAtlas 余项（M2 暂定继续）；Graphiti #2154 是否随新版收纳；Mem0 关联包 `@mem0/cli` 0.2.14 / `@mem0/vercel-ai-provider` 3.0.3 复核。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查），再按子主题轮转（检索排序信号与遗忘策略：降权≠失效≠删除、按类型 TTL、冲突四规则）。

## 2026-11-16 13:09 轮（第 66 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**letta-code 无 v0.34.64**（404 实证），窗口内提交 0 项；PyPI `letta` 仍 0.34.12，差距仍 51 版；v0.34.63 附件下载计数 1 → 3；Mem0 四端点可见词 0 差异（评估/迁移页 cfemail 第九度重随机化；**指南页连续第三轮逐字节相同、无 Framer 再发布**；数字戳与上轮逐数相同）；**63 轮计数分歧已闭合**（14/127/10 是未转义正则的产物，字面量计数为 8/10/6，与 64 轮一致，不再结转）；ENGRAM 第 41 次 401（25 字节相同正文）；MemoryAtlas md5 **连续第四轮同一值** b3b6850d…；MIRIX 无新元数据变化（stars 3467）；DolphinBench PR #10–#14 全 open；Graphiti #2154 仍未发布；Mem0 TS SDK 无新版；七项复现零新增（**连续第三十四轮**）；**Dual-Layer 摘要页再翻转，第十八例**（44,604 → 44,018 B，可见词 844 → 829，md5 与 63 轮文件逐字节相同，恰好逆转上轮全部三处变化；执行者亲自核对四轮证据文件）。
- M5：本轮新直接证据为零（六项原相邻证据全部仍无新版）；新增相邻证据三项——**2611.17338《When Should LLM Agents Forget?》**（相邻 #7，遗忘腿首个量化直接证据：硬删除在解析器可靠时增益 +0.186~+0.230、不可靠时转为亏损 −0.176~−0.181，误删代价显著大于误留；但未报告排序信号曲线，不构成联合约束直接证据）、**2611.17921 RecMem**（相邻 #8 压缩腿：16 槽 + 216 token，Recall@10 87.9% / Recall@200 97.0%）、**2611.18133 LinEra《Forget the Lineage》**（相邻 #9，遗忘腿第二项量化直接证据：逻辑删除后攻击者仍可恢复 70/71 条被遗忘秘密，LinEra 下为 0；遗忘 1% 回合在会话级摘要下抹除 73.0% 派生记忆、四回合窗口下仅 3.8%）；另 **2611.17808 CarryOn** 相邻证据状态自 64 轮「待核对」更新为已核实 v1（75.7% 宏平均成功率、超最强基线 6.7pp）；全部数字论文自报。联合约束继续暂定。11-13 批次枚举闭合（960 条五页全取筛查）；其余候选 2611.17874（记忆权限预览歧义）、2611.18338 MEM-SQL（HARE 37.1% → 50.0%）、2611.17806、2611.15315、2610.03713；已登记 18 篇 + 11-12 批次 10 篇全仍原版本。
- 纠正：M 腿「Dual-Layer 第十七例」误（64 轮已记第十七例，本次为第十八例；执行者亲自核对四轮证据文件字节数与 md5）；64 轮晚段补取未实际执行（仅留 sleep 回显，无 `m-late-*` 文件），本轮取证客观上覆盖其全部目标，64 轮「待晚段补取确认」事项由本轮一手闭合，不再结转。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.63 / 是否出现 0.35.0；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合约束（暂定继续；直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；Graphiti #2154 是否随新版收纳；Mem0 关联包 `@mem0/cli` 0.2.14 / `@mem0/vercel-ai-provider` 3.0.3 复核；2611.17874 / 2611.18338 / 2611.17806 三篇候选是否登记为相邻证据。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；三篇候选登记裁定），再按子主题轮转（检索排序信号与遗忘策略：降权≠失效≠删除、按类型 TTL、冲突四规则）。

## 2026-11-16 16:09 轮（第 67 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**letta-code 无 v0.34.64**（404 实证），最新仍 v0.34.63；窗口内新提交 1 项 c5dffd8（2026-11-16T06:51:51Z，#5754，任务/外部 fork 上报，与记忆无关、未发布）；PyPI `letta` 仍 0.34.12，差距仍 51 版（按版本元组计数；朴素字符串比较误得 59，已纠正口径）；Mem0 四端点可见词 0 差异（评估/迁移页 cfemail 第十度重随机化；**指南页逐字节相同、本轮窗口内无 Framer 再发布**；双口径官方一句话调和仍缺）；Mem0 TS SDK 线无动态（npm 3.3.1）；ENGRAM 第 42 次 401（两路匿名链接均 25 字节 `{"error":"not_connected"}`，与上轮逐字节相同）；**Dual-Layer 摘要页再翻转，第十九例**（44,018 → 43,445 B，可见词 829 → 828；文件与 65 轮逐字节相同，恰好回到 65 轮状态；执行者亲自核对四轮证据文件，关窗后现取又翻回 44,018 B）；七项复现零新增（**连续第三十五轮**）；七项论文版本页全部无新版；MemoryAtlas md5 与 61–66 轮相同；MIRIX stars 仍 3467；Graphiti #2154 仍未发布；letta 服务器仓仍停滞（0.16.8，2026-05-14）；DolphinBench PR #10–#14 全 open。
- M5：三篇全文深读完成（全文词数：2611.17338 7,174 词；RecMem 2611.17921 9,897 词；LinEra 2611.18133 12,533 词），**联合约束直接证据仍为零**（三篇全文 `sort`/`top-k` 均为 0 或仅 1 命中，均未在排序信号与遗忘状态的交互点上提供可检验观测）：2611.17338 oracle 解析器下硬删除增益 +0.175/+0.202/+0.234，部署匹配门控准确率 0.655 vs 始终硬删除 0.641，误删代价显著大于误留（论文自报）；RecMem 2611.17921：216 token 下 R@10 保留 87.9%、R@200 保留 97.0%，每用户状态 8 KiB，长历史 10,000 次交互无质量崩塌（论文自报）；LinEra 2611.18133：逻辑删除 70/71 可恢复、LinEra 下 0，**数字归属澄清：摘要的 73.0%/3.8% 是 7B 扩展运行的数，主运行（1.5B）对应数为 74.8%/3.9%，不得混引**。
- 记忆轨新论文登记：**2611.17874**（治理腿）、**2611.18338 MEM-SQL**（任务记忆复用腿，HARE 基准 37.1% → 50.0%，论文自报）、**2611.17806《Can Agents Keep Up with Time?》**（时间感知腿）登记为记忆轨新论文，**均不登记为 M5 相邻证据**；MIGRO 2611.15315 v2 与 2610.03713 v2 新登记为记忆轨论文；CarryOn 2611.17808 仍 v1，主分类 cs.CL 已核对（66 轮待核对项闭合）。66 轮 `m-arxiv-versions.xml` 文件损坏（XML 标签不匹配），本轮以 66 轮报告所列逐项 `<updated>` 值为对照完成 18 篇复核，下轮须重建对照文件。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.63 / 是否出现 0.35.0；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合约束（暂定继续；直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；Graphiti #2154 是否随新版收纳；Mem0 关联包 `@mem0/cli` 0.2.14 / `@mem0/vercel-ai-provider` 3.0.3 复核；66 轮 `m-arxiv-versions.xml` 对照文件重建。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；对照文件重建），再按子主题轮转（检索排序信号与遗忘策略：降权≠失效≠删除、按类型 TTL、冲突四规则）。

## 2026-11-16 19:09 轮（第 68 轮）

- 本轮增量（D-M01–D-M04，落入 skill/references/layered-memory.md）：**letta-code 无 v0.34.64**（404 实证，执行者亲自查 releases 端点复核），最新仍 v0.34.63；窗口内 0 提交；c5dffd8 仍未发布；PyPI `letta` 仍 0.34.12，差距仍 51 版（按版本元组计数）；Mem0 四端点可见词 0 差异（评估/迁移页 cfemail 第十一度重随机化；**指南页逐字节相同、本轮窗口内无 Framer 再发布**；双口径官方一句话调和仍缺）；Mem0 TS SDK 仍 3.3.1；@mem0/cli 0.2.14、@mem0/vercel-ai-provider 3.0.3 未变；ENGRAM 第 43 次 401（两路匿名链接均 25 字节 `{"error":"not_connected"}`，执行者亲自取回复核）；**Dual-Layer 摘要页再翻转，第二十例**（43,445 → 44,018 B，可见词 828 → 829；文件与 66 轮逐字节相同，恰好逆转 67 轮的三处站点头部变化；执行者关窗后现取又翻回 43,445 B，第二十一例翻转）；七项复现零新增（**连续第三十六轮**）；七项论文版本页全部无新版；MemoryAtlas md5 未变；**Graphiti #2154 已合并未发布**（2026-11-15T20:27:42Z 合入 main，最新仍 v0.31.2；执行者亲自查 PR 端点复核 merged_at——状态纠正）；MIRIX 新 issue #154 + PR #155（均 2026-11-16T09:20Z，embedding_model 在 openai 端点被忽略；PR open 未合并）；MIRIX stars 3467 → 3472，forks 273 → 274；letta 服务器仓仍停滞（0.16.8，2026-05-14）；DolphinBench PR #10–#14 全 open。
- M5：**联合约束直接证据仍为零**。arXiv 11-13 索引追赶完成（submittedDate 20261113 totalResults 0 → 960，与 67 轮枚举一致）；`all:memory` 筛查 54 篇，5 篇与 agent 记忆相关且均已登记，无未登记论文。`m-arxiv-versions.xml` 已重建且 XML 有效（`m-arxiv-versions-table.md` 含 48 篇）；66 轮文件损坏位置已定位（1309/m 文件，`mismatched tag: line 6, column 4`）；**66 轮待核对项闭合**。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.63 / 是否出现 0.35.0；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合约束（暂定继续；直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；Graphiti #2154 是否随新版收纳；MIRIX #155 是否合并。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；MIRIX #155 合并与否），再按子主题轮转（检索排序信号与遗忘策略：降权≠失效≠删除、按类型 TTL、冲突四规则）。

## 2026-11-16 22:09 轮（第 69 轮）

- 本轮增量（D-M01–D-M04，落入 layered-memory.md）：Dual-Layer 摘要页第二十一例翻转（44,018 → 44,604 B，与 64 轮逐字节相同，仅三处站点 chrome 逆转）；ENGRAM 第 44 次 401；七项复现零新增（连续第三十七轮）；Mem0 四端点正文无变化（cfemail 第十二度重随机化；指南页逐字节相同，连续七轮未变）；MIRIX stars 3472 → 3473、Graphiti stars 32840 → 32847，updated_at 均移至窗口内（星标/issue 事件，非代码合入）；Graphiti #2154 仍已合并未发布；MIRIX #155 仍 open 未合并；letta-code 无 v0.34.64，PyPI 差距仍 51 版；MemoryAtlas dpl buildId token 首次轮换（新基线 md5 `6e56ba44`）；已登记 48 篇论文版本全未变；M5 联合约束直接证据仍为零，无新增全文深读候选。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.63 / 是否出现 0.35.0；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合约束（暂定继续；直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；Graphiti #2154 是否随新版收纳；MIRIX #155 是否合并；MemoryAtlas dpl token 轮换后新基线（本轮 6e56ba44）。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；MIRIX #155 合并与否；Graphiti #2154 是否随新版发布；Dual-Layer 第二十二例翻转与否），再按子主题轮转（检索排序信号与遗忘策略：降权≠失效≠删除、按类型 TTL、冲突四规则）。

## 2026-11-17 01:09 轮（第 70 轮）

- 本轮增量（D-M01–D-M04，落入 layered-memory.md）：Dual-Layer 摘要页第二十三例翻转（44,018 → 43,445 B；M 腿「第二十二例」与「无取证文件」说法已纠正：69 轮关窗后翻转 44,604 → 44,018 B 为第二十二例，证据文件 44,018 B、md5 `92be68e3…` 在盘）；Mem0 指南页窗口内 Framer 再发布（可见文本 4,630 词逐字未变，终结 8 轮逐字节相同纪录；cfemail 第十三度重随机化）；ENGRAM 第 45 次 401；七项复现零新增（连续第三十八轮）；letta-code 窗口内提交 9e5c6ef7（与记忆无关，未发布），v0.34.64 连续第七轮缺席（文件证据第六次），PyPI 差距仍 51 版；MIRIX stars 3473 → 3474（窗口内），Graphiti stars +4 为关窗后事件（17:13:45Z）；MIRIX #155 仍 open 未合并；Graphiti #2154 仍已合并未发布；已登记 48 篇论文版本全未变；MemoryAtlas dpl token 未轮换；M5 联合约束直接证据仍为零，无新增全文深读候选。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.63 / 是否出现 0.35.0；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合约束（暂定继续；直接证据仍为零）；MemoryAtlas 余项（M2 暂定继续）；Graphiti #2154 是否随新版收纳；MIRIX #155 是否合并；MemoryAtlas dpl token 轮换后新基线（本轮未轮换，仍 6e56ba44）。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；MIRIX #155 合并与否；Graphiti #2154 是否随新版发布；Dual-Layer 第二十四例翻转与否），再按子主题轮转（检索排序信号与遗忘策略：降权≠失效≠删除、按类型 TTL、冲突四规则）。

## 2026-11-17 04:09 轮（第 71 轮）

- 本轮增量（D-M01–D-M04，落入 layered-memory.md）：Mem0 docs Mintlify 重新部署（dpl buildId 轮换至 `dpl_4D8r1dHg6bQcRuBgQyL3z9waJJtn`；评估页 +5,889 字节系 toggles 数组填入，非正文）；cfemail 第十四度重随机化；指南页逐字节相同、无再发布；MIRIX 新 issue #156（Victor9467，19:38:01Z）质疑 LongMemEval-S 表 7 数字口径（算术复核成立，仅记质疑）；letta-code 窗口内提交 2 项（与记忆无关）；v0.34.64 连续第八轮缺席、v0.34.65 亦 404；PyPI letta 仍 0.34.12，差距仍 51 版；Dual-Layer 无翻转（仍第二十三例状态，43,445 B）；ENGRAM 第 46 次 401；七项复现零新增连续第三十九轮；48 篇论文版本全未变；MemoryAtlas dpl token 未轮换（仍 69 轮基线 md5 `6e56ba44`）；M5 直接证据仍为零。
- 剩余未核实缺口：PyPI letta 是否跟进 0.34.64 / 是否出现 0.34.65；Mem0 双口径点名调和；MIRIX-2.0 代码合入；七项复现；M5 联合证据；MIRIX #156 质疑待维护方回应。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；MIRIX #155 合并与否；Graphiti #2154 是否随新版发布；Dual-Layer 下一例翻转监测），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-17 07:09 轮（第 72 轮）

- 本轮增量（D-M01–D-M04）：letta-code v0.34.64 连续第九轮缺席，PyPI 差距仍 51 版；Mem0 docs Mintlify 再部署（dpl 轮换，评估页 −250 B、迁移页 −6,139 B 为 toggles 回吐，正文无变化）；cfemail 第十五度重随机化；Graphiti 窗口内新 issue #2159（同 group 并发丢节点）；MIRIX stars +1，星标事件推动 updated_at 口径本轮确立；ENGRAM 第 47 次 401；七项复现零新增连续第四十轮；M5 直接证据仍为零。
- 剩余缺口：M5 联合约束直接证据仍为零（本轮无新批次可筛，全文筛查 0 篇）；MIRIX #156 与 Graphiti #2159 维护方回应待跟进；MemoryAtlas dpl token 轮换后新基线；PyPI letta 跟进；Dual-Layer 下一例翻转监测。
- 下轮起点：先清上述缺口（PyPI letta 跟进为首项；M5 直接证据筛查；MIRIX #155 合并与否；Graphiti #2154 是否随新版发布；Dual-Layer 下一例翻转监测），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-17 10:09 轮（第 73 轮）

- 本轮增量（D-M01–D-M04）：letta-code 窗口内 6 提交，v0.34.64 连续第十轮缺席，PyPI 差距仍 51 版；Mem0 docs Mintlify 再部署（dpl 轮换，迁移页 toggles 重新填入 +5,889 字节、恰好逆转 72 轮清空，部署配置抖动）；cfemail 第十六度重随机化；MIRIX stars +1、Graphiti stars +1（后者 2 个 WatchEvent 与净 +1 不吻合、updated_at 01:20:13Z 无对应事件，成因未定位）；Graphiti #2159 与 MIRIX #156 均仍无维护方回应；记忆轨新登记 2 篇论文（2611.19199 LoCoMo 评分敏感度、2611.18856 Self-Improvement Memory，相邻证据 #10/#11），登记集 48 → 50；ENGRAM 第 48 次 401；七项复现零新增连续第四十一轮；M5 联合约束直接证据仍为零。
- 剩余缺口：M5 联合约束直接证据仍为零；Graphiti 星数出入与 updated_at 01:20:13Z 成因未定位；MIRIX #156 与 Graphiti #2159 维护方回应待跟进；PyPI letta 跟进；Dual-Layer 下一例翻转监测。
- 下轮起点：先清上述缺口（M5 直接证据筛查；#156/#2159 回应），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-17 13:09 轮（第 74 轮）

- 本轮增量（M-74-01–M-74-06）：记忆轨新登记 10 篇论文，登记集 50 → 60；arXiv 三批全量枚举筛查闭合（新增 1,548 条、命中 162 条、候选 11 篇）；M5 相邻证据新增 5 项（#12–#16：StaleBench、HEAMS、AdMem、Matrix with Steps、CLAM）至 16 项，直接证据仍为零；letta-code v0.34.64 连续第十一轮缺席，PyPI 差距仍 51 版；Mem0 docs 无再部署，cfemail 第十七度重随机化；MIRIX stars +4、Graphiti stars +3；#2159/#156 仍无回应；ENGRAM 第 49 次 401；七项复现零新增连续第四十二轮；已登记论文两处 v2、73 轮对照表漏列 2 篇已补入。
- 剩余缺口：M5 联合约束直接证据仍为零（相邻证据增至 16 项）；Graphiti #2154 待新版收纳；MIRIX #156 与 Graphiti #2159 维护方回应待跟进；PyPI letta 跟进（差距 51 版，待收纳 12 项）；Dual-Layer 下一例翻转监测；Mem0 双口径点名调和仍缺。
- 下轮起点：先清上述缺口（M5 直接证据筛查；#156/#2159 回应），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-17 16:09 轮（第 75 轮）

- 本轮增量（M-75-01～M-75-06，6 条 → layered-memory-rounds.md）：11-16 批次 1,277 条全量筛查，新登记 6 篇论文，登记集 60 → 66（纠正 74 轮「432 全量枚举闭合」）；letta-code v0.34.64 连续第十二轮缺席；Mem0 cfemail 第十八度重随机化；ENGRAM 第 50 次 401；七项复现连续第四十三轮零新增；MIRIX #156 / Graphiti #2159 仍无回应（纠正 73/74 轮计时错误）；M5 相邻证据增至 20 项，直接证据仍为零。
- 剩余未核实缺口：11-17 批次索引追赶后补筛；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti 星数出入 2 成因；PyPI letta 下次发布（差距 51 版，待收纳 12 项）；M5 联合约束直接证据仍为零；Mem0 双口径官方调和仍缺。

## 2026-11-17 19:09 轮（第 76 轮）

- 本轮增量（M-76-01～M-76-05，5 条 → layered-memory-rounds.md）：11-17 批次仍为 0（索引滞后）；135 篇新投稿补筛，新登记 1 篇（2611.18642），登记集 66 → 67；letta-code v0.34.64 连续第十三轮缺席；Mem0 cfemail 第十九度重随机化；Graphiti #2159 首条评论出现（社区成员 ptimizeroracle，非维护方）；MIRIX #156 仍 0 评论；M5 直接证据仍为零，相邻证据仍为 20 项。
- 剩余未核实缺口：11-17 批次索引追赶后补筛；2611.18642 后续引用与版本跟进；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti 星数出入 1 成因；PyPI letta 下次发布（差距 51 版）；M5 联合约束直接证据仍为零；MemoryAtlas dpl token 轮换后新基线。


## 2026-11-17 22:09 轮（第 77 轮）

- 本轮增量（M-77-01～M-77-05，5 条 → layered-memory-rounds.md）：11-17 批次仍为 0，登记集仍 67 篇；七项复现零新增连续第四十五轮；letta-code v0.34.64 连续第十四轮缺席；Mem0 cfemail 第二十度重随机化 + 指南页 Framer 再发布（正文未变）；MIRIX #156 两条社区评论、Graphiti #2159 第二条评论，维护方均未回应；MemoryAtlas 新基线 md5 481aaa47 建立；M5 直接证据仍为零。
- 剩余未核实缺口：11-17 批次索引追赶后补筛；MIRIX #156 / Graphiti #2159 维护方回应；PyPI letta 下次发布（差距 51 版）；M5 联合约束直接证据仍为零；Mem0 双口径官方调和仍缺。


## 2026-11-18 01:09 轮（第 78 轮）

- 本轮增量（M-78-01～M-78-05，5 条 → layered-memory-rounds.md）：letta-code v0.34.64 窗口内发布（GitHub 16:53:54Z、npm 16:49:42Z），14 项提交含记忆相关 #5772，12 项待收纳提交全部收纳、清零，PyPI 差距 51 → 52 版；Mem0 cfemail 第二十一度重随机化 + 指南页窗口内 Framer 再发布（正文未变）；Graphiti #2158 两条窗口内新评论（RelateDB PR 待 `rfc-approved` 标签）、#2159 仍无维护方回应、星数 13 对 +12 出入 1 再现、#2112/#2050 移动成因未定位；MIRIX #156 仍无维护方回应；MemoryAtlas 新基线稳固；11-17 批次仍为 0，登记集仍 67 篇；七项复现零新增连续第四十六轮；M5 直接证据仍为零。
- 纠正：任务书「v0.34.64 连续第十五轮缺席」不成立，缺席纪录终结于第十四轮（K-78-02）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 52 版，待收纳已清零）；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-17 批次补筛。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.65 是否出现为首项；M5 直接证据筛查；#156/#2159 回应），再按子主题轮转（记忆检索排序与遗忘策略）。


## 2026-11-18 04:09 轮（第 79 轮）

- 本轮增量（M-79-01～M-79-05，5 条 → layered-memory-rounds.md）：letta-code v0.34.65 未出现（GitHub tag 与 compare 均 404），窗口内提交 1 项纯文档，PyPI 差距仍 52 版，待收纳提交重新起计为 1 项；Mem0 cfemail 第二十二度重随机化 + Mintlify 再部署（正文未变）；Graphiti #2163/#2164 窗口内开出并合入（发布流水线修复）；#2158 RelateDB PR 仍未开出、#2159 仍无维护方回应；MIRIX #156 仍无回应；MemoryAtlas 新基线第二轮稳固；11-17 批次仍为 0，登记集仍 67 篇；七项复现零新增连续第四十七轮；M5 直接证据仍为零。
- 剩余未核实缺口：PyPI letta 下次发布（差距 52 版，待收纳 1 项）；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-17 批次补筛。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.65 是否出现为首项；M5 直接证据筛查；#156/#2159 回应），再按子主题轮转（记忆检索排序与遗忘策略）。


## 2026-11-18 10:09 轮（第 81 轮）

- 本轮增量（M-81-01～M-81-05，5 条 → layered-memory-rounds.md）：letta-code v0.34.65 复核通过，v0.34.66 未出现（GitHub 404 两次核查）、v0.35.0 不存在，PyPI 差距仍 53 版，待收纳提交 4 → 8 项；Mem0 cfemail 第二十四度重随机化（正文未变，无 Mintlify 再部署）；Graphiti #2159 仍无维护方回应（约 29.4 小时）、#2158 RelateDB PR 仍未开出，星数全窗 7 对 +7 干净配对；MIRIX #156 仍无回应（约 31.1 小时）；MemoryAtlas 新基线第四轮稳固；ENGRAM 第 56 次 401；七项复现零新增连续第四十九轮；登记集 67 → 74 篇（新登记 7 篇，全部全文深读）；11-16 批次 1,277 → 1,603（迟入 326 条）；11-17 批次 459 条仍在填充；M5 直接证据仍为零，相邻证据 20 → 21 项（新增 KV-LTM）。
- 纠正：腿编号 80→81 与派发书措辞澄清（K-81-02）；11-16 批次闭合推翻 + arXiv 分页方法纠正（K-81-06，75 轮"1,277 条全量筛查闭合"被迟入索引推翻；今后全量枚举一律用 `sortBy=submittedDate&sortOrder=ascending` 分页）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-17 批次补筛（459 条仍在填充）与 11-18 批次。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159 回应；11-17 批次补筛），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-18 13:09 轮（第 82 轮）

- 本轮增量（M-82-01～M-82-05，5 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（GitHub 404 两次核查）、v0.35.0 不存在，PyPI 差距仍 53 版，待收纳仍 8 项；Mem0 cfemail 第二十五度重随机化（正文未变，无 Mintlify 再部署）；Graphiti #2159 仍无维护方回应（约 31.8 小时）、#2158 RelateDB PR 仍未开出，星数窗口内 5 对 +5 干净配对（连续第四轮干净）；Dual-Layer 第二十四例翻转；MIRIX #156 仍无回应（约 33.6 小时）；MemoryAtlas 新基线第五轮稳固；ENGRAM 第 57 次 401；七项复现零新增连续第五十轮；登记集 74 → 78 篇（新登记 4 篇，全部全文深读）；11-17 批次以 1,220 条闭合（迟入 761 条全量筛查）；M5 直接证据仍为零，相邻证据 21 → 22 项（新增 2611.22461）。
- 纠正：无（本轮记忆轨无腿间出入）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159 回应；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-18 16:09 轮（第 83 轮）

- 本轮增量（M-83-01～M-83-05，5 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（GitHub 404 两次核查）、v0.35.0 不存在，PyPI 差距仍 53 版，待收纳仍 8 项；Mem0 cfemail 第二十六度重随机化（正文未变，无 Mintlify 再部署）；Graphiti #2159 仍无维护方回应（约 34.9 小时）、#2158 RelateDB PR 仍未开出，星数窗口内 5 对 +5 干净配对（连续第五轮干净）；Dual-Layer 第三态 44,604 B 计为第二十五例翻转（二态振荡假设证伪）；MIRIX #156 仍无回应（约 36.6 小时）；MemoryAtlas 新基线第六轮稳固；ENGRAM 第 58 次 401；七项复现零新增连续第五十一轮；登记集仍 78 篇；82 轮三项未完成子题全部闭合（11-14=651、11-16=1,603 实测验证；11-16 迟入成因为周末提交登载滞后）；M5 直接证据仍为零，相邻证据仍 22 项。
- 纠正：Dual-Layer 第三态计为第二十五例翻转（K-83-09，二态振荡假设证伪，今后按每次状态变迁计一例）；2502.12110 表值纠正（K-83-13，v9→v11，82 轮笔误）；11-16 批次迟入成因定位（K-83-15）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159 回应；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-18 19:09 轮（第 84 轮）

- 本轮增量（M-84-01～M-84-05，5 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（404 两次）、v0.35.0 不存在，PyPI 差距仍 53 版，待收纳仍 8 项；Mem0 cfemail 第二十七度重随机化（新观察：串值按位置复现，计数口径须按位置书写）；Graphiti 新建 #2165（remove_episode 不撤销失效，「失效≠删除」一手社区证据），星数 7 起对 +6 出入 1（连续五轮干净终结）；Dual-Layer 翻回 82 轮态，计第二十六例翻转；MIRIX #156 仍无回应（约 39.6 小时），open_issues −2 成因未定位；MemoryAtlas 新基线第七轮稳固；ENGRAM 第 59 次 401；七项复现零新增连续第五十二轮；登记集仍 78 篇；M5 直接证据仍为零，相邻证据仍 22 项（Jetriever 全文深读，不升格）。
- 纠正：Mem0 cfemail 计数口径今后须按位置书写（M-84-02，26 轮第 2 位串值本轮出现在第 1 位）；MIRIX open_issues −2 不得写成两起闭合（M-84-04，窗口内无闭合项可配对）。
- 裁定：M5 不升格（C-84-03，Jetriever 全文深读完成，相邻登记维持）；R7/M5/S1 续记。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159/#2165 回应；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-18 22:09 轮（第 85 轮）

- 本轮增量（M-85-01～05，5 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（GitHub 404 两次）、v0.35.0 不存在；PyPI `letta` 仍 0.34.12，差距仍 53 版，待收纳仍 8 项；Mem0 cfemail 第二十八度重随机化（按位置串值逐字不同，无 Mintlify 再部署）；双口径官方一句话调和仍缺；Graphiti 星数 32914 → 32922，窗口内 8 起 WatchEvent 对 +8 干净配对（84 轮出入 1 终结）；#2159 仍无维护方回应（沉默约 40.9 小时）、#2158 RelateDB PR 仍未开出、#2165 仍 0 评论；MIRIX #156 仍无回应（沉默约 42.6 小时），open_issues 仍 43；Dual-Layer 第二十七例翻转；MemoryAtlas 新基线建立（第 1 轮，旧基线 481aaa47 七轮稳固纪录终结）；ENGRAM 第 60 次 401；七项复现零新增连续第五十三轮；登记集 78 篇零新版；M5 直接证据仍为零，相邻证据仍 22 项（StaleBench 2611.19752 全文深读：迄今最强相邻证据，登记维持、不升格）。
- 纠正：无本轨纠正项（M5 不升格见裁定）。
- 裁定：M5 不升格（C-85-03，StaleBench 全文深读完成，相邻登记维持、加强注释，不升格；直接证据仍为零）；R7/M5/S1 续记。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；Graphiti #2112/#2050 移动成因与星数出入成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159/#2165 回应；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 01:09 轮（第 86 轮）

- 本轮增量（M-86-01～05，5 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（GitHub 404 两次）、v0.35.0 不存在；PyPI `letta` 仍 0.34.12，差距仍 53 版，待收纳仍 8 项；v0.34.65 letta.js 下载计数 5 → 6；Mem0 cfemail 第二十九度重随机化（按位置串值逐字不同，无 Mintlify 再部署；数字戳计数一律用 `grep -F` 方法注记）；双口径官方一句话调和仍缺；Graphiti #2158 维护方窗口内接手（自指派 + 标换 70 秒内完成），仍无 `rfc-approved`、PR 未开出；新 PR #2166（`add_episode_bulk` 隐私修复，+2,756/−2,600，仍 open）；#2117 窗口内新评论（社区复测）；星数 32922 → 32924（窗口内 2 起对 +2 干净配对）；forks −1 成因未定位；#2159 仍无维护方回应（约 44.0 小时）、#2165 仍 0 评论；MIRIX forks 274 → 275（与唯一 ForkEvent 配对）；#156 仍无回应（约 45.8 小时），open_issues 仍 43；MemoryAtlas 新基线第 2 轮稳固；ENGRAM 第 61 次 401；七项复现零新增连续第五十四轮；登记集 78 篇零新版；M5 直接证据仍为零，相邻证据仍 22 项（2611.22470 全文深读，不登记）。
- 纠正：Mem0 数字戳计数方法（K-86-13，今后一律用 `grep -F`；85 轮数字戳本身复核无误）。
- 裁定：M5 不升格（2611.22470 全文深读完成，不登记为相邻证据；直接证据仍为零）；R7/M5/S1 续记。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；#2166 是否合入；Graphiti forks −1 成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2158 PR 是否开出；#2166 是否合入；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 04:09 轮（第 87 轮）

- 本轮增量（M-87-01～07，7 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（404 两次）、v0.35.0 不存在，PyPI 差距仍 53 版，待收纳仍 8 项；v0.34.65 letta.js 下载计数仍 6；Mem0 docs 本轮 Mintlify 再部署（clientVersion 0.0.4177 → 0.0.4205），cfemail 第三十度重随机化，新基线 md5 eade880a / 9d1ee474；Graphiti #2158 获 `rfc-approved`（20:04:44Z），RelateDB PR 仍未开出；新 PR #2167/#2168/#2170、新 issue #2169；#2117 第二条评论；星数 32924 → 32927（+3 干净配对）、forks 3389 → 3390（+1 干净配对）；#2159 仍无回应（约 47.0 小时）、#2165 仍 0 评论；MIRIX #156 仍无回应（约 48.7 小时），星数 3497 → 3499；MemoryAtlas 新基线第 3 轮稳固；ENGRAM 第 62 次 401；七项复现零新增连续第五十五轮；登记集 78 篇零新版；M5 直接证据仍为零，相邻证据仍 22 项。
- 纠正：Mem0 数字戳计数方法（K-86-13，今后一律用 `grep -F`；本轮数字戳全未变）。
- 裁定：M5 不升格（本轮无新登记论文、无新全文深读候选；Graphiti #2168/#2167 不登记为相邻证据；直接证据仍为零）；R7/M5/S1 续记。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 8 项）；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；#2166/#2118 是否合入；Graphiti forks 86 轮 −1 成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2158 PR 是否开出；#2166/#2118 是否合入；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 07:09 轮（第 88 轮）

- 本轮增量（M-88-01～06，6 条 → layered-memory-rounds.md）：letta-code v0.34.66 仍未出现（404 两次）、v0.35.0 不存在，PyPI 差距仍 53 版（版本元组计数；初算 61 系字符串比较误计，已纠正），待收纳 8 → 10 项（窗口内 2 提交 e564536、44bdac0 均为 bot，均非记忆相关）；v0.34.65 letta.js 下载计数仍 6；Mem0 cfemail 第三十一度重随机化（6 处串值按位置全部逐字不同，与 87 轮零复现），本轮无 Mintlify 再部署（dpl token 未轮换、clientVersion 仍 0.0.4205），新基线 md5 620ddbf0 / d0fadd84；Dual-Layer 第二十八例翻转（43,445 → 44,018 B，翻回既往态；摘要正文逐字相同，变化为 arXiv 站点 chrome）；Graphiti #2158 `rfc-approved` 已加约 3.2 小时，RelateDB PR 仍未开出（站内搜索总数 0）；星数 32927 → 32928（窗口内唯一 WatchEvent LouisLetcher 20:27:02Z 对 +1 干净配对）；#2166/#2118 仍 open 未合入；#2159 仍无回应（约 50.0 小时）、#2165 仍 0 评论（约 14.2 小时）；MIRIX #156 仍无回应（约 51.7 小时），星数 3499 持平；MemoryAtlas 新基线第 4 轮稳固；ENGRAM 第 63 次 401；七项复现零新增连续第五十六轮；已登记 78 篇零新版；M5 直接证据仍为零，相邻证据仍 22 项。
- 裁定：M5 不升格（本轮无新登记论文、无新全文深读候选；窗口内 Graphiti 新 PR 0 件；直接证据仍为零）；R7/M5/S1 续记。
- 剩余未核实缺口：PyPI letta 下次发布（差距 53 版，待收纳 10 项）；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；#2166/#2118 是否合入；Graphiti forks 86 轮 −1 成因；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次筛查。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 是否出现为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2158 PR 是否开出；#2166/#2118 是否合入；11-18 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 10:09 轮（第 89 轮）

- 本轮增量（M-89-01～07，7 条 → layered-memory-rounds.md）：letta-code v0.34.66 已在 npm 发布（2026-11-19T02:11:41Z，关窗后事件；tag 已存在指向 1af182c；GitHub Release 对象截至 02:16:57Z 仍未创建；主轮解析 M 腿 `m-letta-npm.json` 复核）；窗口内提交 5 项；待收纳 10 → 0 项；PyPI `letta` 仍 0.34.12，差距 53 → 54 版；v0.35.0 仍不存在；v0.34.65 letta.js 下载计数 6 → 7；关窗后新提交 5094712（02:20:25Z）为下轮待收纳第 1 项；Mem0 cfemail 第三十二度重随机化（评估页 1 处、迁移页 5 处）；仍无 Mintlify 再部署（dpl token 未轮换、clientVersion 仍 0.0.4205）；新基线 md5 评估页 55a53643 / 迁移页 981929f5；双口径官方一句话调和仍缺；Dual-Layer 第二十九例翻转（44,018 → 44,604 B，翻回第三态；摘要正文逐字相同）；Graphiti：#2158 `rfc-approved` 已加约 6.2 小时，RelateDB PR 仍未开出（站内搜索总数 0）；窗口内新开 4 件（PR #2171/#2172/#2174、issue #2173，均机器人，均 open）；#2166/#2118/#2167/#2168/#2170 仍 open；星数 32928 → 32932（+4：窗口内可配对 2 起、关窗后 1 起、1 起无法定位）；forks 仍 3390；#2159 新增第 4 条评论（作者本人 01:57:34Z），维护方回应仍为零（约 52.9 小时）；#2165 仍 0 评论（约 17.2 小时）；MIRIX #156 仍无维护方回应（约 54.7 小时）；星数 3499 → 3500（窗口内 WatchEvent pqtch 23:20:08Z 干净配对）；forks 仍 275；open_issues 仍 43；arXiv 11-18 批次 221 条全量枚举筛查闭合（M 腿以 API 全量枚举为准，与路由轨列表页口径不同）；11-17 迟入 303 条全量枚举筛查闭合（1,220 → 1,523）；新登记 5 篇：2611.23651 Safe Personal RSI、2611.23808 PReMind、2611.23476 GUARD、2611.23588 ARISE、2611.23346；登记集 78 → 83 篇；已登记 78 篇零新版；M5 直接证据仍为零，相邻证据 22 → 24 项（提议新增 GUARD #23、ARISE #24）；MemoryAtlas 新基线第 5 轮稳固；ENGRAM 第 64 次 401；七项复现零新增连续第五十七轮。
- 纠正：无（本轮 M 腿无主轮纠正项）。
- 裁定：M5 不升格（直接证据仍为零；相邻 22 → 24 项）；R7/M5/S1 续记；2611.23651 已由记忆轨登记（H 腿 2611.23651 登记与本轨一致，不双计）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 54 版，待收纳 1 项为关窗后提交 5094712）；v0.34.66 GitHub Release 对象是否创建；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；#2166/#2118/#2167/#2168/#2170 是否合入；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-19 批次筛查（网站翻页核查）。
- 下轮起点：先清上述缺口（PyPI letta 跟进与 v0.34.66 Release 对象为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2158 PR 是否开出；#2166/#2118 是否合入；11-19 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。
## 2026-11-19 13:09 轮（第 90 轮）

- 本轮增量（M-90-01～08，8 条 → layered-memory-rounds.md）：letta-code v0.34.66 GitHub Release 对象窗口内创建（published 02:17:55Z，id 450796519；npm 02:11:41Z；主轮复核发布对象）；窗口内提交 1 项（5094712，仍未收纳）；PyPI `letta` 仍 0.34.12，差距仍 54 版；v0.34.67/v0.35.0 均不存在；v0.34.65 letta.js 下载计数仍 7；Mem0 cfemail 第三十三度重随机化（新基线 md5 145841b9 / 1537edc9）；仍无 Mintlify 再部署（dpl token 未轮换、clientVersion 仍 0.0.4205）；双口径官方一句话调和仍缺；Dual-Layer 第三十例翻转（44,604 → 44,018 B，翻回 88 轮态）；Graphiti：窗口内新开 #2175（expired_at/invalid_at 失效语义澄清）、#2052 窗口内关闭、#2012 新评论；星数 32932 → 32934（+2）；#2158 RelateDB PR 仍未开出（rfc-approved 已加约 9.7 小时）；#2159/#2165 维护方回应仍为零（#2159 约 56.4 小时）；#2166/#2118 仍 open；MIRIX #156 仍无维护方回应（约 58 小时），星数 3500 持平；arXiv 11-18 全类目批次 totalResults 221 → 1,304（89 轮「221 条闭合」被推翻，迟入索引所致），p1/p2 未取回，筛查未闭合；新登记 8 篇，登记集 83 → 91 篇（暂定，待 p1/p2 筛查后最终确认）；2611.22245 出 v2（唯一已登记论文新版，仅 10 词批注）；M5 直接证据仍为零，相邻证据 22 → 24 项升格确认（GUARD #23、ARISE #24）；MemoryAtlas 第 6 轮稳固；ENGRAM 第 65 次 401；七项复现零新增连续第五十八轮。
- 纠正：89 轮「221 条闭合」被推翻（M-90-06，迟入索引所致，11-18 全类目批次实际 1,304 条）；M 腿「M5 仍 22 项待裁定」误读 89 轮记录（M-90-08，89 轮轮次文件已写 22→24，升格于本轮确认）。
- 裁定：M5 相邻证据升格确认（GUARD #23、ARISE #24 正式计入，24 项）；R7/M5/S1 续记；2611.23651 登记归属记忆轨（与 H 腿不双计，沿用 89 轮口径）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 54 版，待收纳 1 项为窗口内提交 5094712）；v0.34.67/v0.35.0 是否出现；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；#2166/#2118 是否合入；#2175 失效语义澄清进展；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-18 批次 p1/p2 筛查（登记集 91 篇为暂定数）；arXiv 11-19 批次筛查。
- 下轮起点：先清上述缺口（11-18 批次 p1/p2 筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2158 PR 是否开出；#2166/#2118 是否合入；11-19 批次筛查），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 16:09 轮（第 91 轮）

- 本轮增量（M-91-01～08，8 条 → layered-memory-rounds.md）：11-18 全类目批次 p1/p2 取回并筛查闭合（totalResults 仍 1,304；p0+p1+p2 去重 1,304 无断档；关键词网 54 篇逐项裁定，新登记 0 篇）；登记集 91 篇由暂定转为最终确认；11-19 批次尚未公告（API 0），留待 92 轮；letta-code 窗口内零提交；v0.34.67/v0.35.0 均不存在；PyPI 仍 0.34.12，差距仍 54 版；提交 5094712 连续第三轮未被收纳；Mem0 cfemail 第三十四度重随机化（新基线 md5 ddc032ba / eb9d94f7）；90 轮「迁移页第 2/4 位串值相同」现象消失；仍无 Mintlify 再部署；双口径官方一句话调和仍缺；Graphiti 星数 32934→32937（+3；窗口内 4 起关注对 +3，1 起无法定位）；PR #2141 窗口内由作者本人关闭未合入（open_issues 575→574）；#2158 RelateDB PR 仍未开出（rfc-approved 已加约 12.2 小时）；#2159/#2165 维护方回应仍为零（沉默约 58.9/23.2 小时）；MIRIX #156 仍沉默约 60.6 小时，星数持平 3500；Dual-Layer 第三十一例翻转（44,018→43,445 B，摘要正文逐字相同）；MemoryAtlas 新基线第 7 轮稳固；ENGRAM 第 66 次 401；七项复现零新增连续第五十九轮；已登记 91 篇唯一新版 Jetriever 2611.21478 v1→v2（大幅改写，+570 词，摘要重写，核心数字前后一致 106/125=84.8%，论文自报）；M5 相邻证据 #25–#28 登记（OpenKAGE、Jetriever、ActiveForager、KnowledgeClaw），相邻 24 → 28 项；RAA / ViStep-RL / 23844 / MemoSync / Space-X 判定不列为相邻；2611.24267 无 arXiv HTML 渲染（404），改用 PDF 文本兜底深读。
- 裁定：登记集 91 篇由暂定转最终确认（M 腿 p1/p2 闭合）；M5 相邻证据 #25–#28 登记；M5 续记（直接证据仍为零）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 54 版，待收纳 5094712 连续第三轮）；v0.34.67/v0.35.0 是否出现；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2158 RelateDB PR 是否开出；#2166/#2118 是否合入；#2175 失效语义澄清进展；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-19 批次筛查。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2158 PR 是否开出；#2166/#2118 是否合入），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 19:09 轮（第 92 轮）

- 本轮增量（M-92-01～09，9 条 → layered-memory-rounds.md）：Graphiti RelateDB PR #2176 终于开出（草案，2026-11-19 09:29:50Z，+5,961/−5，33 文件，链接 Fixes #2158；`rfc-approved` 至开出约 13.4 小时；主轮 GitHub API 独立复核一致），91 轮“仍未开出”口径终结；Graphiti 星数 32937→32939（+2 精确配对）；#2159 沉默约 61.9 小时、#2165 约 26.1 小时、MIRIX #156 约 63.6 小时，维护方回应均为零；#2175 失效语义澄清零进展；#2166/#2118 等仍 open；Mem0 cfemail 第三十五度重随机化（新基线 md5 a93af5a7 / c7131395），仍无 Mintlify 再部署，双口径官方一句话调和仍缺；letta-code npm 仍 0.34.66，v0.34.67/v0.35.0 均不存在，提交 5094712 连续第四轮未被收纳，PyPI 差距仍 54 版；Dual-Layer 第三十二例翻转（43,445→44,018 B，翻回 88 轮态）；MemoryAtlas 新基线第 8 轮稳固；ENGRAM 第 67 次 401；七项复现零新增连续第六十轮；登记集 91 篇零新版；M5 直接证据仍为零，相邻证据维持 28 项；11-18 批次零迟入（全类目 1,304 与 91 轮一致），11-19 批次未公告留待 93 轮。
- 纠正：91 轮“#2158 RelateDB PR 仍未开出”已终结（#2176 开出）；M 腿“#25–#28 待主轮裁定”说法过期不采纳（91 轮 M-91-08 已登记入 rounds 文件）。
- 裁定：#2176 草案 PR 是否转正/合入列为下轮复核项；M5 相邻证据维持 28 项；M5 续记（直接证据仍为零）。
- 剩余未核实缺口：PyPI letta 下次发布（差距 54 版，待收纳 5094712 连续第四轮）；v0.34.67/v0.35.0 是否出现；MIRIX #156 / Graphiti #2159 / Graphiti #2165 维护方回应；Graphiti #2176 草案 PR 是否转正/合入；#2166/#2118 是否合入；#2175 失效语义澄清进展；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-19 批次筛查。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2176 是否转正/合入；#2166/#2118 是否合入），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-19 22:09 轮（第 93 轮）

- 本轮增量（M-93-01～07，7 条 → layered-memory-rounds.md）：Graphiti #2176 仍为草案（窗口内零活动，未转正、未合入），关窗后新 PR #2177（#2159 报告者 suryaprasanthcse 于 14:13:24Z 开出，针对 #2159 竞态的社区修复提案，+233/−6，3 文件；正文称 FalkorDB 6.0.4 与 Neo4j 5.26.2 上各 20/20 次丢失，社区作者实测、非维护方验证；不是维护方回应，#2159 未解决）；Graphiti forks 首次出现出入 1（窗口内 2 起 ForkEvent 对净增 +1，未定位），星数 +2 仍精确配对；#2159 沉默约 64.9 小时、#2165 约 29.2 小时、MIRIX #156 约 66.6 小时，维护方回应仍为零；#2175 失效语义澄清零进展；Mem0 cfemail 第三十六度重随机化（新基线 md5 4826ceda / 5c51ff08），仍无 Mintlify 再部署，双口径官方一句话调和仍缺；letta-code npm 仍 0.34.66，v0.34.67/v0.35.0 均不存在，5094712 连续第五轮未被收纳，PyPI 差距仍 54 版；Dual-Layer 无翻转；MemoryAtlas 内容稳定第九轮（md5 轮换属部署抖动）；ENGRAM 第 68 次 401；七项复现零新增连续第六十一轮；登记集 91 篇零新版；M5 直接证据仍为零，相邻证据维持 28 项（本轮新登记 0 篇，13 篇全文重取字节全同；#2177 的 20/20 丢失实测不登记为 M5 证据）；11-18 批次零迟入（totalResults 仍 1,304，memory 85 条 id 集与 92 轮完全相同），11-19 批次未公告留待 94 轮。
- 纠正：无（92 轮“#2176 草案 PR 刚开出”口径延续，本轮仍为草案）。
- 裁定：#2177 的 20/20 丢失实测不登记为 M5 证据（社区作者实测、非维护方验证，且针对 #2159 竞态而非 M5 联合约束）；M5 相邻证据维持 28 项；M5 续记（直接证据仍为零）。
- 剩余未核实缺口：Graphiti #2176 草案 PR 是否转正/合入；#2177 维护方评审/合入；#2159/#2165、MIRIX #156 维护方回应；#2175 失效语义澄清进展；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-19 批次筛查；letta-code v0.34.67/v0.35.0 复核。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2176 是否转正/合入、#2177 维护方评审；#2166/#2118 是否合入），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20 04:09 轮（第 94 轮）

- 本轮增量（M-94-01～08，8 条 → layered-memory-rounds.md）：11-19 批次仍未公告（留待 95 轮），11-18 批次零迟入（totalResults 仍 1,304，memory 85 条 id 集与 93 轮完全相同）；letta-code npm 仍 0.34.66，5094712 连续第六轮未被收纳，PyPI 差距仍 54 版；Mem0 cfemail 第三十七度重随机化，仍无 Mintlify 再部署，双口径官方调和仍缺；Graphiti #2176 仍为草案，窗口内新开 #2178 与 #2179（社区提案），#2177 仍 open；星数 +3 精确配对、forks +1 精确配对（93 轮出入 1 未重现，旧出因仍未定位）；#2159/#2165、MIRIX #156 维护方回应均为零；Dual-Layer 无翻转；MemoryAtlas 内容第十轮稳固；ENGRAM 第 69 次 401；七项复现零新增连续第六十二轮；登记集 91 篇零新版；M5 直接证据仍为零，相邻 28 项；方法口径（列表页「19 Nov」与 API `submittedDate 20261119` 不是同一口径）。
- 纠正：无（92 轮“#2176 草案 PR 刚开出”口径延续，本轮仍为草案；93 轮 forks 出入 1 本轮未重现，但旧出因仍未定位，不得写成已闭合）。
- 裁定：#2177 的 20/20 丢失实测不登记为 M5 证据（社区作者实测、非维护方验证，且针对 #2159 竞态而非 M5 联合约束）；M5 相邻证据维持 28 项；M5 续记（直接证据仍为零）。
- 剩余未核实缺口：Graphiti #2176 草案 PR 是否转正/合入；#2177/#2178/#2179 维护方评审/合入；#2159/#2165、MIRIX #156 维护方回应；#2175 失效语义澄清进展；Mem0 双口径官方调和仍缺；M5 联合约束直接证据仍为零；arXiv 11-19 批次筛查；letta-code v0.34.67/v0.35.0 复核。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2176 是否转正/合入、#2177/#2178/#2179 维护方评审；#2166/#2118 是否合入），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20 07:09 轮（第 95 轮）

- 本轮增量（M-95-01～10，10 条 → layered-memory-rounds.md）：11-19 批次仍未公告（留待 96 轮），11-18 批次零迟入；letta-code 5094712 连续第七轮未被收纳，PyPI 差距仍 54 版；Mem0 cfemail 第三十八度重随机化，Mintlify 再部署确认发生（dpl token 轮换、clientVersion 0.0.4205 → 0.0.4227，正文无变化），双口径官方调和仍缺；Graphiti 轮间维护方清理潮（19:33–20:05Z 关闭 21 项、合入 4 个 PR），#2175 已合入（失效语义澄清落入 edges.py）；#2158 获维护方回应（#2176 转正路径）；窗口内新开 #2180/#2181，星数出入 1 新增未定位，open_issues 578 → 559 全额闭合；#2176 仍草案；#2159 沉默约 74.0 小时、#2165 约 38.3 小时、MIRIX #156 约 75.8 小时，维护方回应均为零；Dual-Layer 第三十三例翻转；MemoryAtlas 内容第十一轮稳固；ENGRAM 第 70 次 401；七项复现零新增连续第六十三轮；登记集 91 篇零新版；M5 直接证据仍为零，相邻证据维持 28 项。
- 纠正：94 轮「Mem0 仍无 Mintlify 再部署」终结（再部署已确认发生）；94 轮「Graphiti 失效语义澄清零进展」终结（#2175 已合入）。
- 裁定：Mem0 Mintlify 再部署与 Graphiti #2175 合入均为轮间事件（94 轮取证后、95 轮窗口前），不计入 95 轮窗口增量；M5 续记（#2175 为「失效 ≠ 删除」提供一手代码证据，但联合约束本身仍无直接证据）；M 腿 "window event" 行阈值沿用旧轮会误标轮间事件，窗口归属一律以 20:09:08Z 硬边界人工重算。
- 剩余未核实缺口：arXiv 11-19 批次筛查（公告档 11-20 01:00Z，已过）；Graphiti #2176 草案 PR 是否转正/合入；#2177/#2178/#2179/#2181 维护方评审/合入；#2159/#2165、MIRIX #156 维护方回应；#2175 失效语义澄清后续（#2012 已关闭）；Mem0 双口径官方调和仍缺；letta-code v0.34.67/v0.35.0 复核；M5 直接证据筛查。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2176 是否转正/合入、#2177/#2178/#2179/#2181 维护方评审；#2166/#2118 是否合入），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20 10:09 轮（第 96 轮）

- 本轮增量（M-96-01～10，10 条 → layered-memory-rounds.md）：arXiv 11-19 批次全量枚举 192 条（取回时刻闭合，迟入风险注明）；11-18 批次迟入大爆发 1,304 → 1,624（+320；91 轮闭合推翻）；新登记 5 篇（登记集 91 → 96；M5 相邻证据 28 → 33：ChronoDossier、ActMem、GrepMem、Some Memories Must Fade、GEMS；M5 直接证据仍为零）；letta-code v0.34.67 窗口内发布（5094712 收纳终结；PyPI 差距 55 版）；Mem0 cfemail 第三十九度重随机化，无新 Mintlify 再部署；Mem0 双口径官方调和仍缺；Graphiti 窗口内新开 #2182/#2183，#2176 仍草案，星数出入 1 未定位；MIRIX #156 维护方沉默约 78.7 小时；Dual-Layer 无翻转；MemoryAtlas 内容第 12 轮稳固；ENGRAM 第 71 次 401；七项复现零新增连续第六十四轮；TripleKV / LSP-RAG 维持不登记。
- 纠正：91 轮「11-18 批次 1,304 条闭合」推翻（本轮 1,624 条）；95 轮「11-18 memory 查询 removed 2611.23930/2611.24503」作废（max_results=100 截断 + 默认 relevance 排序假象，200 上限重取证实移除为 0）。
- 裁定：11-19 批次口径按取回时刻记录（M「闭合」与 H/RB「未闭合」并存）；M5 续记（直接证据仍为零）；TripleKV / LSP-RAG 不登记（边界口径不放宽）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；Graphiti #2176 草案 PR 是否转正/合入；#2177/#2178/#2179/#2181 维护方评审/合入；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Mem0 双口径官方调和；M5 直接证据筛查（DSFlow 2611.25111 深读）；MemoryAtlas 各重跑行方法核实。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；M5 直接证据筛查；#156/#2159/#2165 回应；#2176 是否转正/合入、#2177/#2178/#2179/#2181 维护方评审；#2182/#2183 合入动向），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20 13:09 轮（第 97 轮）

- 本轮增量（M-97-01～08，8 条 → layered-memory-rounds.md）：arXiv export API 全轮 0 取回（05:13Z 起 429/503）；慢速重取脚本于 05:40Z 前后随运行环境重启停止，未重启；DSFlow 2611.25111 全文深读裁定不登记（M5 直接证据仍为零，相邻证据仍 33 项）；letta-code 仍 v0.34.67，窗口内提交 0 项，v0.35.0 不存在；Graphiti 窗口内新开 #2184/#2185（#2185 实体边 save/save_bulk 丢弃 `reference_time`）；MIRIX #156 仍沉默约 81.8 小时；Mem0 cfemail 第四十度重随机化（新基线 md5 0efebaa2/cb531747）；**M2 余项已定位**（六行均为竞争方重跑行；PRACTICAL.md M2 条目移入已修正）；ENGRAM 第 72 次 401；七项复现零新增连续第六十四轮顺延为待定（不得写成第六十五轮）。
- 纠正：96 轮「11-19 批次全量枚举 192 条（取回时刻闭合）」订正——cs.AI totalResults 实为 202，全量枚举仍未闭合；七项复现「第六十四轮」为顺延计数口径，不得递增为第六十五轮。
- 裁定：DSFlow 不登记（全文深读，排序/遗忘词汇全 0 命中）；M5 续记（直接证据仍为零）；M2 口径收窄为「归属已定位、仍不得当作可比排名引用」；arXiv export API 全灭新登未结台账；M 腿四项未闭合改由下轮重做。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；M 腿四项未闭合（11-19/11-18 迟入、11-20 枚举、96 篇版本复核、七项复核）下轮重做；Graphiti #2176 草案是否转正/合入、#2177/#2178/#2179/#2181/#2183/#2185 维护方评审/合入；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Mem0 双口径官方调和仍缺；M5 直接证据仍为零。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；M 腿四项未闭合重做；#156/#2159/#2165 回应；#2176 是否转正/合入、#2177/#2178/#2179/#2181/#2183/#2185 维护方评审），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20 16:09 轮（第 98 轮）

- 本轮增量：11-19 批次全量枚举闭合（1,324 条；cs.AI 202）；11-18 迟入复核闭合（totalResults 1,624 无膨胀）；M5 相邻证据新增 11 项（#34–#44），登记集 96 → 107；M5 直接证据仍为零；Mem0 cfemail 第四十一度重随机化；Graphiti forks 出入 1 新增、#2126 窗口内被触碰（动作未定位）；MIRIX #156 沉默约 84.6 小时；MemoryAtlas 第 14 轮稳固；ENGRAM 第 73 次 401。
- 纠正：七项复现计数口径更新——按「已验证零新增轮次」计为第六十五轮（本轮实际取回确认零新增；97 轮「不得写成第六十五轮」的前提是未取回）。
- 裁定：DSFlow 不登记、Decoupled 不登记；M5 续记（直接证据仍为零）；七项复现第六十五轮口径（C-98-13）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；Graphiti #2176 草案是否转正/合入、#2126 动作定位、forks 出入 1 成因；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Mem0 双口径官方调和仍缺；M5 直接证据仍为零。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；#156/#2159/#2165 回应；#2176 是否转正/合入、#2126 动作定位），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20（19:09 轮，第 99 轮）
- 增量：11-19 批次全量 1,324 闭合；新登记 0 篇（登记集 107）；Graphiti #2177 首次实质评审（社区评审者，非维护方回应，不登记为 M5 证据）；Graphiti 星数 +6 精确配对、forks 出入 1 第三次同形态；三方沉默（#2159 约 86.0 小时、#2165 约 50.2 小时、MIRIX #156 约 87.7 小时）；Mem0 cfemail 第四十二度重随机化；letta-code 仍 v0.34.67；七项复现第六十六轮。
- 纠正：无。
- 裁定：#2177 评审不登记为 M5 证据（93 轮裁定沿用）；M5 续记（直接证据仍为零，相邻 44 项）；七项复现第六十六轮口径（C-99-15）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；Graphiti #2176 草案是否转正/合入、#2126 动作定位、forks 出入 1 成因；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Mem0 双口径官方调和仍缺；M5 直接证据仍为零。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；#156/#2159/#2165 回应；#2176 是否转正/合入、#2126 动作定位），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-20（22:09 轮，第 100 轮）
- 增量：11-19 批次迟入复核闭合（1,324 条）、11-18 零迟入（1,624 条）；新登记 0 篇（登记集 107）；Graphiti #2159 pinned 20/20 全量计数（双后端各 20/20 丢失）；#2177 作者推提交 56534c6 并提议拆分；三方沉默（#2159 约 89.0 小时、#2165 约 53.2 小时、MIRIX #156 约 90.7 小时）；Mem0 cfemail 第四十三度重随机化；letta-code 仍 v0.34.67；七项复现第六十七轮；ENGRAM 第 75 次 401。
- 纠正：无。
- 裁定：#2159 pinned 计数为代码仓证据，不升格为 M5 直接证据；M5 续记（直接证据仍为零，相邻 44 项）；七项复现第六十七轮口径（98 轮裁定沿用）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；Graphiti #2176 草案是否转正/合入、#2126 动作定位、forks 出入 1 成因；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Mem0 双口径官方调和仍缺；M5 直接证据仍为零。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；#156/#2159/#2165 回应；#2176 是否转正/合入、#2126 动作定位），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（01:09 轮，第 101 轮）
- 增量：11-19 批次迟入复核闭合（1,324 条）、11-18 零迟入（1,624 条）；新登记 0 篇（登记集 107）；Graphiti #2186/#2187 新开（DZH166 两修复 PR）、窗口后 #2188 首次捕获；**Graphiti forks 出入 1 成因首次定位**（取回时刻滞后，取回区间内事件总数与净增精确吻合，长期缺口闭合）；**#2107 触碰动作已定位**（作者新评论确认 GroqClient 修复生效并指出 GLiNER2Client 缓存键缺口，92 轮遗留缺口闭合其一）；chunxiaoxx 两分钟内同步跟进两仓独立复现邀约（邀约本身，不计复现）；三方沉默（#2159 约 92.0 小时、#2165 约 56.2 小时、MIRIX #156 约 93.7 小时）；Mem0 cfemail 第四十四度重随机化；letta-code 仍 v0.34.67；七项复现第六十八轮；ENGRAM 第 76 次 401。
- 纠正：无。
- 裁定：forks 出入 1 成因已定位（取回时刻滞后），长期缺口闭合——下轮起取回时须记录取回时刻与窗口关闭时刻的对照，不得再以严格口径出入 1 重立缺口；chunxiaoxx 两仓邀约不登记为 M5 证据；#2107 作者新评论为代码仓证据，不升格；M5 续记（直接证据仍为零，相邻 44 项）；七项复现第六十八轮口径（98 轮裁定沿用）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；Graphiti #2176 草案是否转正/合入、#2126 动作定位、#2186/#2187/#2188 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Mem0 双口径官方调和仍缺；M5 直接证据仍为零。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；#156/#2159/#2165 回应；#2176 是否转正/合入、#2126 动作定位、#2186/#2187/#2188 维护方评审），再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（04:09 轮，第 102 轮）
- 增量：11-20 批次仍未公告（公告前读数 0）；11-19 迟入复核闭合（1,324 条）、11-18 闭合（1,624 条）；新登记 0 篇，登记集 107；M5 直接证据仍为零，相邻维持 44 项；**Mem0 Mintlify 再部署确认发生**（dpl token 轮换，clientVersion 0.0.4227 → 0.0.4248；101 轮「仍无再部署」终结）；**Graphiti #2188 落入本轮窗口并继续生长**（+5,785/−1,549 → +6,141/−1,545，68 → 69 文件，4 → 6 提交；维护方评审与合入均为零）；**Graphiti forks −1 为与 101 轮已定位机制不同源的新缺口**（取回区间内 ForkEvent 0 起）；星数 +1 对 2 起 WatchEvent，出入 1 未定位；三方沉默推进（#2159 约 95.0 小时、#2165 约 59.3 小时、MIRIX #156 约 96.7 小时）；**七项复现零新增计第六十九轮**；Dual-Layer 无翻转；MemoryAtlas 内容第 18 轮稳固；ENGRAM 第 77 次 401；letta-code 窗口内新提交 1 项（17436c8），待收纳 0 → 1，仍 v0.34.67，PyPI 差距仍 55 版。
- 纠正：101 轮 MemoryAtlas md5「e1ddc026」为笔误，两轮实测均为 eb6f5c10，下轮基线改用 eb6f5c10。
- 裁定：M5 仍暂定（直接证据仍为零）；Graphiti forks −1 不得写成 101 轮机制复发；七项复现计第六十九轮（98 轮口径）。
- 剩余未核实缺口：arXiv 11-20 批次筛查；Mem0 双口径官方调和仍缺；ETUDE 作者仓定位与 M5 无关（技能轨）；Graphiti #2176 草案是否转正/合入、#2126 动作定位、#2186/#2187/#2188 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Graphiti 星数出入 1 与 forks −1 新缺口。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（07:09 轮，第 103 轮）

- 增量：11-20 批次仍未公告（周五档期，筛查留待周一轮次）；11-19 迟入复核闭合（1,324 条）、11-18 闭合（1,624 条）；**Graphiti #2189 窗口内新建**（GroqClient 默认模型改为 openai/gpt-oss-120b，修复 #2103）；Graphiti 星数 +2、forks +2 均精确配对；**#2107 验证链闭合到人**（ptimizeroracle 11-19 评审），合入仍为零；#1922 窗口内社区新评论；Mem0 cfemail 第四十六度重随机化，无新的 Mintlify 再部署；letta-code 仍 v0.34.67，PyPI 差距仍 55 版；**七项复现零新增计第七十轮**；MemoryAtlas 内容第 19 轮稳固；ENGRAM 第 78 次 401。
- 纠正：102 轮 forks −1 不得写成已闭合（仍未定位）；11-20 批次「公告档 11-21 01:00Z」写法对周五批次不成立（见路由轨）。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项）；#2107 作者新评论为代码仓证据，不升格；七项复现计第七十轮（98 轮口径，实际取回确认）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；ETUDE 作者仓定位与 M5 无关（技能轨）；Graphiti #2176 草案是否转正/合入、#2126 动作定位、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Graphiti 星数出入 1 与 forks −1 新缺口。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（10:09 轮，第 104 轮）

- 增量：11-20 批次仍未公告（周五档期，筛查留待周一轮次）；11-19 迟入复核闭合（1,324 条）、11-18 闭合（1,624 条）；**Mem0 cfemail 第四十七度重随机化**（Mintlify 再部署发生于两轮取证之间，非窗口内）；letta-code 仍 v0.34.67（窗口内新提交 3 项；v0.34.68 关窗后出现，归 105 轮；PyPI 差距仍 55 版）；Graphiti 窗口内新建 0，星数 +2/forks ±0 精确配对；#2126 触碰动作连续第七轮未定位；**七项复现零新增计第七十一轮**；MemoryAtlas 内容第 20 轮稳固；ENGRAM 第 79 次 401。
- 纠正：102 轮 forks −1 不得写成已闭合（仍未定位）；Mem0 再部署不得写为窗口内（时刻无法钉死）。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项）；#2107 作者新评论为代码仓证据，不升格；七项复现计第七十一轮（98 轮口径，实际取回确认）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；ETUDE 作者仓定位与 M5 无关（技能轨）；Graphiti #2176 草案是否转正/合入、#2126 动作定位（连续第七轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核（v0.34.68 已于关窗后出现，归 105 轮）；Graphiti 星数出入 1 与 forks −1 新缺口。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（13:09 轮，第 105 轮）

- 增量：**letta-code v0.34.68 落在本轮窗口内**（npm 02:46:47Z、GitHub Release 02:53:01Z；窗口内新提交 4 项；PyPI 差距 55 → 56 版）；**Mem0 cfemail 第四十八度重随机化**（Mintlify 再部署又发生，非窗口内）；Graphiti 窗口内新开 2 项，均为 cursor[bot] 垃圾广告（#2190、#2191）；星数 +4/forks ±0 均精确配对；#2126 触碰动作连续第八轮未定位；**七项复现本轮未取回**（不得递增为第七十二轮）；**Dual-Layer 第三十五例翻转**（43,445 → 44,018B，站点铭牌 SciX → NASA ADS）；MemoryAtlas 内容第 21 轮稳固；ENGRAM 第 80 次 401。
- 纠正：104 轮「v0.34.68 关窗后出现」正确，105 轮窗口内发布确认；Dual-Layer 翻转成因系站点铭牌改动、非论文内容变化（按 91 轮尺寸态口径仍计翻转）；102 轮 forks −1 不得写成已闭合（仍未定位）；Mem0 再部署不得写为窗口内（时刻无法钉死）。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项）；七项复现计第七十一轮顺延待定（98 轮口径，未取回不得递增）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；七项复现与 107 篇版本复核（留 106 轮取回后裁定）；11-19/11-18 迟入复核（M 腿四项未闭合）；Mem0 双口径官方调和仍缺；Graphiti #2176 草案是否转正/合入、#2126 动作定位（连续第八轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（16:09 轮，第 106 轮）

- 增量：**arXiv export API 全面恢复**（21 查询全部一次 200）；**105 轮四项未闭合全部闭合**；**七项复现零新增计第七十二轮**（98 轮口径，实际取回确认；105 轮顺延待定已终结）；**letta-code 仍 v0.34.68**（PyPI 差距仍 56 版；待收纳仍 2 项）；**Mem0 cfemail 第四十九度重随机化**（新基线 md5 评估页 04c169a4 / 迁移页 1c34d4ba；本轮无新的 Mintlify 再部署）；Mem0 双口径官方调和仍缺；Graphiti 窗口内新开 0 项（唯一窗口活动为 #2144 机器人 stale 提醒；星数 +1/forks ±0 均精确配对；102 轮 forks −1 仍未定位）；#2126 触碰动作连续第九轮未定位；#2176 仍草案、#2177 拆分提议未回复、#2186–#2189 评审与合入均为零、#2107 零活动；三方沉默：#2159 约 106.8 小时、#2165 约 71.1 小时、MIRIX #156 约 108.6 小时；**Dual-Layer 第三十六例翻转**（44,018 → 43,445 B，105 轮第 35 例的精确翻回，摘要正文逐字相同，站点铭牌 NASA ADS → SciX）；MemoryAtlas 内容第 22 轮稳固；ENGRAM 第 81 次 401；Zep Cloud 零新动态。
- 纠正：105 轮「七项复现不得递增为第七十二轮」已终结（本轮实际取回确认，计第七十二轮）；Dual-Layer 翻转成因系站点铭牌改动、非论文内容变化（按尺寸态口径仍计翻转，与 105 轮第 35 例同口径）。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 草案是否转正/合入、#2126 动作定位（连续第九轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（19:09 轮，第 107 轮）

- 增量：**七项复现零新增计第七十三轮**（98 轮口径，21 查询全部一次 200；11-19/11-18 迟入均为零迟入，全量 1,324 / 1,624 条；107 篇版本零变化）；**letta-code 仍 v0.34.68**（窗口内零提交；PyPI 差距仍 56 版；待收纳仍 2 项；letta.js 下载 v0.34.68 由 2 → 3）；**Mem0 cfemail 第五十度重随机化**（新基线 md5 评估页 ecc1305c / 迁移页 dd7675d0；本轮无新的 Mintlify 再部署）；Mem0 双口径官方调和仍缺；Graphiti **#2176 草案已转正**（ready_for_review 09:54:29Z；新提交 ecc24c60 将 main 合入分支；仍 open、未合入、无维护方评审）、**#2165 零评论被打破**（社区成员 shaojunying 09:28:10Z 长篇复现评论，6,768 字符；维护方回应仍为零；星数 ±0 对 1 起 WatchEvent 出入 1 未定位，本轮新缺口）；#2177 拆分提议评审者未回复、#2186–#2189 评审与合入均为零、#2107 零活动；三方沉默：#2159 约 109.8 小时、MIRIX #156 约 111.6 小时；**Dual-Layer 第三十七例翻转**（43,445 → 44,018 B，106 轮第 36 例的精确翻回，摘要正文逐字相同，站点铭牌 SciX → NASA ADS）；MemoryAtlas 内容第 23 轮稳固；ENGRAM 第 82 次 401；Zep Cloud 零新动态。
- 纠正：M 腿 arXiv 慢速脚本首发被运行环境打断（零日志），已用 setsid 重启并全量跑完 21/21；`m-analyze107.py` 的 `total()` 因 opensearch 命名空间返回 None，totalResults 已用 grep 原始文件逐字核对，下轮应修脚本或沿用 grep 口径。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）；Dual-Layer 第三十七例翻转按 91 轮尺寸态口径计数（35/36/37 三例同属一次站点铭牌往复，非论文内容变化；下轮起同类铭牌往复仍按此口径计数，不再每轮提请裁定）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第九轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；Graphiti 星数出入 1 未定位（本轮新缺口）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-21（22:09 轮，第 108 轮）

- 增量：七项复现零新增计第七十四轮（98 轮口径；11-19/11-18 迟入均为零迟入，全量 1,324 / 1,624 条；107 篇版本零变化）；letta-code 仍 v0.34.68（PyPI 差距仍 56 版；待收纳仍 2 项；letta.js v0.34.68 下载 3→4）；Mem0 cfemail 第五十一次重随机化；Dual-Layer 无新翻转（44,018 B）；MemoryAtlas 内容第 24 轮稳固，但七轮零 churn 纪录终结（静态资源哈希轮换，非内容变化）；ENGRAM 第 83 次 401；Graphiti 窗口内新开 3 项（#2192/#2193/#2194，均无维护方动作），星数 +4 精确配对；三方沉默 #2159 约 112.9 小时、MIRIX #156 约 114.6 小时、#2165 约 77.1 小时。
- 纠正：#2126 触碰动作未定位轮次：107 轮 track 尾部「连续第九轮」为笔误，订正为第十轮；本轮按 M 节链计连续第十一轮。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）；107 轮星数出入 1 仍未定位（本轮 +4 不含上轮那 1 起，不得写成已闭合）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十一轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；Graphiti 星数出入 1 未定位；102 轮 forks −1 未定位；letta-code v0.35.0 复核；Gemini nightly #577（预计 2026-11-22 00:19Z）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（01:09 轮，第 109 轮）

- 增量：七项复现零新增计第七十五轮（98 轮口径；11-19/11-18 迟入均为零迟入，全量 1,324 / 1,624 条；107 篇版本零变化）；letta-code 仍 v0.34.68（PyPI 差距仍 56 版；待收纳 60b09c3、76428c1 仍 2 项）；Mem0 cfemail 第五十二度重随机化（新基线 md5 eval 页 c92ab461／迁移页 9bdea8e9；迁移页第 3、4 项字符串值相同）；MemoryAtlas 内容第 25 轮稳固（md5 与 108 轮逐字节相同，零 churn 再现）；ENGRAM 第 84 次 401；Graphiti #2177 评审者窗口内接受拆分（仍未合入），窗口内新开 #2195/#2196；Dual-Layer 第三十八例翻转（站点铭牌 NASA ADS → SciX）。
- 纠正：#2126 触碰动作未定位轮次：107 轮 track 尾部「连续第九轮」为笔误，108 轮已订正为第十轮、第十一轮；本轮按 M 节链计连续第十二轮，口径现已统一，不再复议。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）；Dual-Layer 第三十八例按 91 轮尺寸态口径计数；107 轮星数出入 1 仍未定位（本轮 +3 不含上轮那 1 起，不得写成已闭合）；102 轮 forks −1 仍未定位。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十二轮）、#2186/#2187/#2188/#2189 维护方评审；#2107 验证链（head 12c29c83）仍未合入；#2144 约 11-28 关闭；#2190/#2191 交叉引用事件来源（另一仓 issue #3）待确认；MIRIX #156 维护方沉默约 117.6 小时。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（04:09 轮，第 110 轮）

- 增量：七项复现零新增计第七十六轮（98 轮口径；11-19/11-18 迟入均为零迟入，全量 1,324 / 1,624 条；107 篇版本零变化）；letta-code 仍 v0.34.68（PyPI 差距仍 56 版；待收纳 60b09c3、76428c1 仍 2 项；letta.js 下载 4 → 5）；Mem0 cfemail 第五十三度重随机化（新基线 md5 eval 页 1e0065d1／迁移页 41d2e0a0；无新 Mintlify 再部署）；MemoryAtlas 内容第 26 轮稳固（md5 与 109 轮逐字节相同，零 churn 再现）；ENGRAM 第 85 次 401；Graphiti 窗口内新开 0 项、零维护方活动（星数 +5 精确配对、forks ±0 精确配对）；Dual-Layer 第三十九例翻转（站点铭牌 SciX → NASA ADS）；#2193 已确认为 issue 而非 PR。
- 纠正：#2126 触碰动作未定位轮次：本轮按 M 节链计连续第十三轮，口径已统一，不再复议；记忆轮次文件拆分已完成（392 → 112 行；2026-11-16 至 2026-11-20 的轮次移入 archive，现 613 行），拆分时发现既有缺口：记忆轮次文件缺 2026-11-21 16:09 与 19:09 两节，待下轮补记。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）；Dual-Layer 第三十九例按 91 轮尺寸态口径计数；107 轮星数出入 1 仍未定位（本轮 +5 不含上轮那 1 起，不得写成已闭合）；102 轮 forks −1 仍未定位；#2190/#2191 仍 open 未处理（机器人广告），继续跟进、不关闭议题。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十三轮）、#2186/#2187/#2188/#2189 维护方评审；#2107 验证链（head 12c29c83）仍未合入；#2144 约 11-28 关闭；#2190/#2191 是否被关闭/处理；MIRIX #156 维护方沉默约 120.8 小时；记忆轮次文件缺 2026-11-21 16:09 与 19:09 两节（待下轮补记）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（07:09 轮，第 111 轮）

- 增量：七项复现零新增计第七十七轮（98 轮口径；11-19/11-18 迟入均为零迟入，全量 1,324 / 1,624 条；107 篇版本零变化）；letta-code 仍 v0.34.68（PyPI 差距仍 56 版；待收纳 60b09c3、76428c1 仍 2 项；letta.js 下载计数全未变）；Mem0 cfemail 第五十四度重随机化（新基线 md5 eval 页 b916e634／迁移页 a44135cb；无新 Mintlify 再部署）；MemoryAtlas 内容第 27 轮稳固（md5 与 110 轮逐字节相同，零 churn 再现）；ENGRAM 第 86 次 401；Graphiti 窗口内新开 0 项、零维护方活动（星数出入 −2 新缺口；forks ±0 精确配对）；Dual-Layer 第四十例翻转（现处 SciX 态）；#2197 为窗口后事件（归属 112 轮）；记忆轮次文件缺 2026-11-21 16:09 与 19:09 两节已补记（只补两处小节标题，正文本已在 13:09 小节内）。
- 纠正：#2126 触碰动作未定位轮次：本轮按 M 节链计连续第十四轮，口径已统一，不再复议；#2193 已确认为 issue 而非 PR（110 轮结论维持）。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）；Dual-Layer 第四十例按 91 轮尺寸态口径计数；Graphiti 星数出入 −2 为新缺口（不得与 108 轮出入 1 冲销）；102 轮 forks −1 仍未定位；#2190/#2191 仍 open 未处理（机器人广告），继续跟进、不关闭议题。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十四轮）、#2186/#2187/#2188/#2189 维护方评审；#2107 验证链仍未合入；#2144 约 11-28 关闭；#2190/#2191 是否被关闭/处理；MIRIX #156 维护方沉默约 123.6 小时；#2197 归属 112 轮。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（10:09 轮，第 112 轮）

- 增量：七项复现零新增第七十八轮（98 轮口径，实取确认；11-19/11-18 迟入均为零，全量 1,324 / 1,624 条）；107 篇版本零变化；M5 直接证据仍为零，相邻证据维持 44 项；本轮新登记 0 篇；letta-code 仍 v0.34.68（PyPI 差距仍 56 版；无 v0.34.69 / v0.35.0）；Mem0 cfemail 第五十五度重随机化（新基线 md5 评估页 1e0065d1 / 迁移页 43cebc65；评估页取值回落到 110 轮值）；MemoryAtlas 内容第 28 轮稳固（1,022 词逐字相同）；ENGRAM 第 87 次 401；Dual-Layer 本轮无翻转（仍处 SciX 态）；Graphiti 窗口内新开 #2197；星数出入 1 新缺口；MIRIX #156 维护方沉默约 126.6 小时。
- 纠正：无（本轮无旧说法推翻；#2126 触碰动作未定位轮次计连续第十五轮，口径已统一，不再复议）。
- 裁定：M5 仍暂定（直接证据仍为零；相邻证据维持 44 项；本轮无新论文可深读）；Dual-Layer 仍处 SciX 态（第四十例之后无翻转）；Graphiti 星数出入 1 为新缺口（不得与 111 轮 −2、108 轮出入 1 冲销）；102 轮 forks −1 仍未定位；#2190/#2191 仍 open 未处理（机器人广告），继续跟进、不关闭议题。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十五轮）、#2186/#2187/#2188/#2189 维护方评审；#2107 验证链仍未合入；#2144 约 11-28 关闭；#2190/#2191 是否被关闭/处理；MIRIX #156 维护方沉默约 126.6 小时；#2197 维护方评审/合入。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（13:09 轮，第 113 轮）

- 本轨增量 6 条（M-113-01～M-113-06；修正 1 条 M-113-02 为 M5 续记）：七项复现零新增第七十九轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 44 项；letta-code 仍 v0.34.68（PyPI 差距 56 版）；Mem0 cfemail 第五十六度重随机化（112 轮「小值集循环」猜想未再现）；MemoryAtlas 内容第 29 轮稳固；ENGRAM 第 88 次 401；Dual-Layer 第四十一例翻转（SciX → NASA ADS，现处 NASA ADS 态）；Graphiti 窗口内新开 0 项，星数 +2 精确配对、无新缺口；#2197 零活动；#2126 连续第十六轮未定位；MIRIX #156 维护方沉默约 129.6 小时。
- 纠正：112 轮「小值集循环」猜想未再现（两页基线值本轮均为新值）；112 轮 Graphiti 星数出入 1 新缺口本轮未重现（星数 +2 精确配对）。
- 裁定：无跨腿裁定（本轨数字两腿一致）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十六轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；#2197 维护方评审/合入；letta-code v0.35.0 复核；M5 直接证据仍为零。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（16:09 轮，第 114 轮）

- 本轨增量 6 条（M-114-01～M-114-06；修正 0 条）：七项复现零新增第八十轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 44 项；letta-code 仍 v0.34.68（PyPI 差距 56 版）；Mem0 cfemail 第五十七度重随机化（113 轮「小值集循环」猜想未再现）；MemoryAtlas 内容第 30 轮稳固；ENGRAM 第 89 次 401；Dual-Layer 本轮无翻转（仍处 NASA ADS 态）；Graphiti 窗口内新开 0 项，星数 +4 / forks +1 精确配对、无新缺口；#2177 作者窗口内回复跑分（未复现评审者的两起 EntityNotFound）；#2126 连续第十七轮未定位；MIRIX #156 维护方沉默约 132.6 小时。
- 纠正：无（本轨无旧说法推翻；113 轮「小值集循环」猜想未再现已于 113 轮记过）。
- 裁定：无跨腿裁定（本轨数字两腿一致）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十七轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；#2197 维护方评审/合入；letta-code v0.35.0 复核；M5 直接证据仍为零。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（19:09 轮，第 115 轮）

- 本轨增量 6 条（M-115-01～M-115-06；修正 0 条）：七项复现零新增第八十一轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 44 → 47（三篇摘要级深挖候选：MemCredit 2611.15600、RD-Forget 2609.10263、Selective Forgetting 2608.28978）；letta-code 仍 v0.34.68（PyPI 差距 56 版）；Mem0 cfemail 第五十八度重随机化（113 轮「小值集循环」猜想未再现）；MemoryAtlas 内容第 31 轮稳固；ENGRAM 第 90 次 401；Dual-Layer 第四十二例翻转（回 SciX 态）；Graphiti 窗口内新开 0 项，星数 +5 / forks ±0 精确配对、无新缺口；#2126 连续第十八轮未定位；MIRIX #156 维护方沉默约 135.6 小时。
- 纠正：无（本轨无旧说法推翻；113 轮「小值集循环」猜想未再现已于 113 轮记过）。
- 裁定：无跨腿裁定（本轨数字两腿一致）；M5 三篇候选登记为相邻证据 #45–47（摘要级，全文未深读，暂定）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2176 是否合入、#2126 动作定位（连续第十八轮）、#2186/#2187/#2188/#2189 维护方评审；#2159/#2165、MIRIX #156 维护方回应；#2197 维护方评审/合入；letta-code v0.35.0 复核；M5 直接证据仍为零。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-22（22:09 轮，第 116 轮）

- 本轨增量 6 条（M-116-01～M-116-06；修正 0 条）：七项复现零新增第八十二轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 47 项（三篇全文深读完成：MemCredit 2611.15600、RD-Forget 2609.10263、Selective Forgetting 2608.28978，均为相邻证据 #45–#47；MemCredit ρ = −0.011 已由全文核实为论文原文数字）；letta-code 仍 v0.34.68（PyPI 差距 56 版）；Mem0 cfemail 第五十九度重随机化；MemoryAtlas 内容第 32 轮稳固（静态资源哈希第二度轮换）；ENGRAM 第 91 次 401；Dual-Layer 无翻转（仍 SciX 态）；Graphiti 星数 +2 精确配对；#2126 连续第十九轮未定位；#2177 评审跟进 + 作者两推，评论 5 → 6 条；MIRIX #156 沉默约 138.6 小时。
- 纠正：无（本轨无旧说法推翻）。
- 裁定：M5 三篇深读确认为相邻证据 #45–#47（直接证据仍为零）；115 轮「M5 三篇摘要级候选」已推进为全文深读完成。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第十九轮）、#2177 FalkorDB add 侧后续 PR；#2159/#2165、MIRIX #156 维护方回应；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；letta-code v0.35.0 复核；M5 直接证据仍为零。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（01:09 轮，第 117 轮）

- 本轨增量 6 条（M-117-01～M-117-06；修正 0 条）：七项复现零新增第八十三轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 47 项（深挖综合：三篇恰好各占一层且正交——MemCredit 排序层、RD-Forget 失效层、Selective Forgetting 删除层；三层同开的交互效应正是直接证据所缺的实验形态，暂定）；letta-code 仍 v0.34.68（PyPI 差距 56 版）；Mem0 cfemail 第六十度重随机化；MemoryAtlas 内容第 33 轮稳固且零 churn；ENGRAM 第 92 次 401；Dual-Layer 第四十三例翻转（新变体，仍 SciX 态）；Graphiti 星数 +2 精确配对、forks −1 新缺口；#2126 连续第二十轮未定位；#2190/#2191 交叉引用源已定位（109 轮待确认项关闭）；MIRIX #156 沉默约 141.7 小时。
- 纠正：无（本轨无旧说法推翻）。
- 裁定：M5 三层映射综合记为执行者综合、暂定（不升格直接证据）；Dual-Layer 第 43 例翻转按 91 轮尺寸态口径计数；Graphiti forks −1 不得冲销。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十轮）、#2177 FalkorDB add 侧后续 PR、forks −1 缺口定位；#2159/#2165、MIRIX #156 维护方回应；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；letta-code v0.35.0 复核；M5 直接证据仍为零（三层同开交互效应为待验证的实验形态）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（04:09 轮，第 118 轮）

- 本轨增量 6 条（M-118-01～M-118-06；修正 0 条）：七项复现零新增第八十四轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 47 项（深挖综合维持执行者综合、暂定）；letta-code v0.34.69 窗口内发布（npm 19:11:53Z、GitHub 19:18:01Z；待收纳由 2 项降为 0 项；PyPI 差距 56 → 57 版；无 v0.35.0）；Mem0 cfemail 第六十一度重随机化 + 新的 Mintlify 再部署（105/106 轮之后首现；迁移页可见词数 −102 为格式变化）；Dual-Layer 第四十四例翻转（振荡回同一状态，按 91 轮尺寸态口径计数）；Graphiti 星数 +4 / forks ±0 均精确配对；#2126 连续第二十一轮未定位；MemoryAtlas 第 34 轮稳固；ENGRAM 第 93 次 401。
- 纠正：无（本轨无旧说法推翻）。
- 裁定：M5 三层映射综合记为执行者综合、暂定（不升格直接证据）；Dual-Layer 第 44 例翻转按 91 轮尺寸态口径计数（振荡回同一状态仍计一例）；Graphiti forks −1 不得冲销；#2177/#2159 时间线新增事件定性为作者侧活动，不是维护方回应。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十一轮）、#2177 FalkorDB add 侧后续 PR、forks −1 缺口定位；#2159/#2165、MIRIX #156 维护方回应；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；letta-code v0.35.0 复核；M5 直接证据仍为零（三层同开交互效应为待验证的实验形态）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（07:09 轮，第 119 轮）

- 本轨增量 15 条（M-119-01～M-119-15；修正 0 条）：七项复现零新增第八十五轮；107 篇版本零变化；M5 直接证据仍为零，相邻证据 47 项（深挖综合六项均未见新证据，暂定）；letta-code 仍 v0.34.69（窗口内提交 0 项；无 v0.34.70，GitHub 404 新证据；无 v0.35.0；PyPI 差距仍 57 版）；Mem0 cfemail 第六十二度重随机化（无新 Mintlify 再部署；迁移页可见词数 4,235 不变）；Mem0 双口径官方调和仍缺；Graphiti 星数 +5 / forks ±0 均精确配对；#2126 连续第二十二轮未定位；#2159 静默约 145.9 小时、#2165 约 110.2 小时、MIRIX #156 约 147.7 小时；Dual-Layer 第四十五例翻转（振荡回同一状态，按 91 轮尺寸态口径计数）；MemoryAtlas 第 35 轮稳固；ENGRAM 第 94 次 401。
- 纠正：无（本轨无旧说法推翻）。
- 裁定：Dual-Layer 第 45 例翻转按 91 轮尺寸态口径计数（振荡回同一状态仍计一例，C-119-06）；M5 深挖综合六项均未见新证据，记暂定（M-119-15）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（周一轮次）；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十二轮）、#2177 FalkorDB add 侧后续 PR、forks −1 缺口定位；#2159/#2165、MIRIX #156 维护方回应；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；letta-code v0.35.0 复核；M5 直接证据仍为零（三层同开交互效应为待验证的实验形态）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（10:09 轮，第 120 轮）
- 本轨增量 15 条（M-120-01～M-120-15；修正 0 条）：arXiv 11-20 批次已公告（周一部分，全类 294 条；`all:memory` 25 条）；11-19 迟入大爆发（1,324 → 1,579，+255；登记集 107 → 115 篇）；七项复现零新增第八十六轮；M5 直接证据仍为零，相邻证据 47 → 54 项（#48–#54 七篇全文深读完成；STARK v2 HTML 缺口关闭）；letta-code 仍 v0.34.69（窗口内提交 0 项；无 v0.34.70、无 v0.35.0；PyPI 差距仍 57 版）；Mem0 cfemail 第六十三度重随机化（无新 Mintlify 再部署；迁移页可见词数 4,235 不变）；Mem0 双口径官方调和仍缺；Graphiti 星数 +1 / forks ±0 均精确配对；#2126 连续第二十三轮未定位；#2159 静默约 148.9 小时、#2165 约 113.1 小时、MIRIX #156 约 150.6 小时；MIRIX #155 PR 窗口内被作者关闭（未合入；forks −1 新缺口）；Dual-Layer 本轮无翻转（仍第四十五例）；MemoryAtlas 第 36 轮稳固；ENGRAM 第 95 次 401；DolphinBench 新 PR #16；Zep 侧零动态；深挖综合新增第 5 项未决问题（暂定）。
- 纠正：119 轮「11-19 零迟入」只在其取证时刻成立，不推翻、只加注时效（M-120-01）；M 腿慢速脚本复制事故（`s|119|120|g` 误伤 1119 区块，已用 0709 原文逐字补跑）与 `m-pairing120.py` 末段缺 `REF_G` 崩溃（沉默小时数手工算出，下轮须修脚本）。
- 裁定：M5 相邻证据 47 → 54 项，直接证据仍为零；STARK v2 HTML 缺口关闭（98 轮缺口，C-120-12）。
- 剩余未核实缺口：arXiv 11-20 批次第二档筛查（周二 01:00Z 公告后）；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十三轮）、#2177 FalkorDB add 侧后续 PR；#2159/#2165、MIRIX #156 维护方回应；MIRIX forks −1 缺口定位；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；letta-code v0.35.0 复核；M5 直接证据仍为零（三层同开交互效应为待验证的实验形态，新增第 5 项未决问题）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（13:09 轮，第 121 轮）
- 本轨增量 15 条（M-121-01～M-121-15；修正 0 条）：11-20 第二档已到（记忆集 25 → 62，新增 37 篇全部筛查；120 轮预期被实测纠正）；11-21/11-22 当前读数均为 0（周末提交尚未编目）；11-19 迟入 +1 已定位闭合；11-18 闭合（1,629 不变）；115 篇版本复核零新版；七项复现零新增第八十七轮；登记集 115 → 118 篇；M5 相邻证据 54 → 57 项（#55 MemOnce、#56 VMSA、#57 NavTTT，三篇全文深读、不升格；VMSA 为最干净的「只合并、不删除」实例）；letta-code 仍 v0.34.69（窗口内提交 0 项）；Mem0 cfemail 第六十四度重随机化；Graphiti #2176 窗口内合入（#2158 同批关闭，CLA 缺口关闭）；#2178 作者新推 + Codex 评审；#2126 连续第二十四轮未定位；#2159 静默约 151.9 小时；#2165 维护方回应仍为零（约 116.1 小时）；#2144/#2149/#2150/#2156 仍未关闭（关闭日约 11-28/11-29）；MIRIX #156 静默约 153.6 小时；Dual-Layer 本轮无翻转（仍第四十五例）；MemoryAtlas 第 37 轮稳固；ENGRAM 第 96 次 401；Zep 侧零动态。
- 纠正：120 轮「11-20 第二档等 11-24 01:00Z 公告」的预期被实测纠正（仅限记忆轨；C-121-07）；M 腿 C-121-01/02（评审事件计数与脚本文件名拼接错误已修正重跑）。
- 裁定：11-21/11-22 的 0 篇分轨定口径（C-121-06）；11-20 第二档纠正仅限记忆轨（C-121-07）。
- 剩余未核实缺口：arXiv 11-20 批次第二档筛查（H 腿：2026-11-24 01:00Z 公告后；本轨已闭合）；11-21/11-22 批次复核；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十四轮）、forks −1 缺口定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（16:09 轮，第 122 轮）
- 本轨增量 15 条（M-122-01～M-122-15；修正 0 条）：11-20 全量迟入 +778（294 → 1,072，记忆集 62 不变）；11-21/11-22 当前读数均为 0（周末提交尚未编目）；11-19/11-18 闭合；118 篇版本复核零新版；七项复现零新增第八十八轮；登记集仍 118 篇；M5 直接证据仍为零，相邻证据维持 57 项；letta-code 仍 v0.34.69（窗口内提交 2 项，待收纳 0 → 2；PyPI 差距仍 57 版）；Mem0 cfemail 第六十五度重随机化；Graphiti #2178 维护方直接向作者分支推提交 + Devin 再评审；#2198 维护方本人所开、三轮 Devin 评审并自动修掉 2 处；星数 +6 / forks ±0 均精确配对；#2126 连续第二十五轮未定位；#2165 维护方回应仍为零（约 119.2 小时）；#2144/#2149/#2150/#2156 仍未关闭（关闭日约 11-28/11-29）；MIRIX #156 静默约 156.6 小时；Dual-Layer 第四十六例翻转（新变体：铭牌翻转 + 可见文本变化）；MemoryAtlas 第 38 轮稳固；ENGRAM 第 97 次 401；Zep 侧零动态。
- 纠正：无。
- 裁定：11-21/11-22 的 0 篇分轨定口径（C-122-10）；Dual-Layer 第四十六例翻转计入（新变体，C-122-14）。
- 剩余未核实缺口：11-21/11-22 批次复核；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十五轮）、forks −1 缺口定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（19:09 轮，第 123 轮）

- 本轨增量：15 条（M-123-01～15），全部落入 skill/references/layered-memory-rounds.md。letta-code 窗口内发布 v0.34.70（npm 09:36:30Z、GitHub 09:41:17Z），PyPI 差距 57 → 58 版，待收纳 2 → 0，无 v0.35.0；Mem0 cfemail 第六十六度重随机化，无新 Mintlify 再部署；Graphiti 窗口内新开 #2199/#2200（salvage_json），#2177 评审者第三轮重跑验证（COMMENTED，未批准）；星数 +4 精确配对、forks 新缺口（净 −1、事件流 +1，相差 −2 未定位）；#2126 连续第二十六轮未定位；#2159 沉默约 157.8 小时、#2165 约 122.1 小时、MIRIX #156 约 159.6 小时；Dual-Layer 第四十七例翻转（第 46 例的精确逆转，翻回 SciX 态）；MemoryAtlas 第 39 轮稳固；ENGRAM 第 98 次 401；M5 直接证据仍为零、相邻证据 57 项；七项复现零新增第八十九轮；11-20 全量 1,072 稳定、记忆集 62 不变；arXiv 慢速脚本被 SIGTERM 中断后四段续跑补齐，26 查询全部首试 200。
- 纠正：无。
- 裁定：11-21/11-22 的 0 篇分轨定口径沿用公告滞后（C-123-07）；Dual-Layer 第四十七例翻转计入（第 46 例的精确逆转）；#2165/#2180 时间线窗口后第三方 cross-ref 归属 124 轮，不计为维护方回应。
- 剩余未核实缺口：11-21/11-22/11-23 批次复核；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十六轮）、forks −1 新缺口定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-23（22:09 轮，第 124 轮）

- 本轨增量：15 条（M-124-01～15），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（无 v0.35.0；PyPI 差距仍 58 版）；Mem0 cfemail 第六十七度重随机化，无新 Mintlify 再部署；Graphiti 窗口内新开 0 项，星数 +4 精确配对、forks ±0 精确配对；#2177 评审仍 COMMENTED 未批准；#2126 连续第二十七轮未定位；#2159 沉默约 160.9 小时、#2165 约 125.2 小时、MIRIX #156 约 162.7 小时；Dual-Layer 第四十八例翻转（第 47 例的精确逆转，现处 NASA ADS 态）；MemoryAtlas 第 40 轮稳固；ENGRAM 第 99 次 401；M5 直接证据仍为零、相邻证据 57 项；七项复现零新增第九十轮；11-20 全量 1,072 稳定、记忆集 62 不变；本轮 M 腿无未取回项。
- 纠正：无。
- 裁定：11-21/11-22/11-23 的 0 篇分轨定口径沿用公告滞后（C-124-09）；Dual-Layer 第四十八例翻转计入（第 47 例的精确逆转）；#2165/#2180 时间线窗口内第三方 cross-ref 不计为维护方回应。
- 剩余未核实缺口：11-21/11-22/11-23 批次复核；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十七轮）；123 轮 forks 缺口（净 −1 对事件流 +1，相差 −2）仍未定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（01:09 轮，第 125 轮）

- 本轨增量：9 条（M-125-01～09），全部落入 skill/references/layered-memory-rounds.md。11-21/11-22 仍为 0（周末未编目，迟取复核确认）；11-20 全量 1,072 稳定、记忆集 62 不变；118 篇零新版；七项复现零新增第九十一轮；M5 直接证据仍为零、相邻证据 57 项（深挖综合第 5 项无推进，暂定）；letta-code 仍 v0.34.70（窗口内新增提交 1 项，测试类；PyPI 差距仍 58 版）；Mem0 cfemail 第六十八度重随机化，指南页首次 Framer 再发布（17:05Z，内容零变化）；Graphiti 窗口内新开 #2201/#2202，星数 +6、forks ±0 均精确配对；0.32.0 簇（#2178/#2198 仓内钉到 0.32.0，PyPI 尚未发布，三 PR 均无人工评审）；#1604 作者合并上游 main、15:42:00Z 事件是 8 月旧批准的重放（方法论收口：评审归属以 submitted_at 为准）；#2126 连续第二十八轮未定位；#2159 沉默约 164.1 小时、#2165 约 128.4 小时、MIRIX #156 约 165.9 小时；#2144/#2149/#2150/#2156 仍未到关闭日；MIRIX 星数 +1 精确配对；Dual-Layer 本轮无翻转（NASA ADS 态）；MemoryAtlas 第 41 轮稳固；ENGRAM 第 100 次 401；DolphinBench forks 10 → 11。
- 纠正：无（#1604 评审归属方法论收口为新增口径，非既往错误纠正）。
- 裁定：11-21/11-22/11-23 的 0 篇分轨定口径沿用公告滞后（C-124-09 沿用）；#1604 窗口内评审一律以评审对象 submitted_at 为准。
- 剩余未核实缺口：11-21/11-22/11-23 批次复核；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十八轮）；123 轮 forks 缺口（净 −1 对事件流 +1，相差 −2）仍未定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（04:09 轮，第 126 轮）

- 本轨增量：7 条（M-126-01～07），全部落入 skill/references/layered-memory-rounds.md。11-21/11-22 仍为 0（周末未编目，迟取复核确认）；11-20 全量 1,072 稳定、记忆集 62 不变；118 篇零新版；七项复现零新增第九十二轮；M5 直接证据仍为零、相邻证据 57 项（深挖综合第 5 项无推进，暂定）；letta-code 仍 v0.34.70（窗口内新增提交 2 项，均非版本类；PyPI 差距仍 58 版）；Mem0 cfemail 第六十九度重随机化，评估/迁移页出现新的 Mintlify 再部署（迁移页词数 +102 系 Shiki 重渲染、散文零变化）；Graphiti 窗口内新开 0 项，星数 +3 精确配对、forks −1 新缺口（第四个）；#2178 窗口内新提交 + 5 则 Devin 新评审、#2198 标题改写，均无人工评审；PyPI 0.32.0 仍未发布；#1478 六个月旧 PR 窗口内 force-push 复活；#2126 连续第二十九轮未定位；#2159 沉默约 166.9 小时、#2165 约 131.1 小时、MIRIX #156 约 168.6 小时；#2144/#2149/#2150/#2156 均未到关闭日；MIRIX 零变化；Dual-Layer 本轮无翻转（NASA ADS 态）；MemoryAtlas 第 42 轮稳固；ENGRAM 第 101 次 401；DolphinBench forks +1 事件已定位（NightFrost42）。
- 纠正：#2178/#2198 轮间提交按时间戳归属 125 轮窗口，不计入本轮窗口增量；#2178 评审评论端点错误已纠正重取。
- 裁定：11-21/11-22/11-23 的 0 篇分轨定口径沿用公告滞后（C-124-09 沿用）；Graphiti forks 新缺口记为第四个（取回区间 ForkEvent 0 起）。
- 剩余未核实缺口：11-21/11-22/11-23 批次复核；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第二十九轮）；Graphiti forks 新缺口（第四个）成因定位；123 轮 forks 缺口（净 −1 对事件流 +1，相差 −2）仍未定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（07:09 轮，第 127 轮）

- 本轨增量：11 条（M-127-01～11），全部落入 skill/references/layered-memory-rounds.md。11-20 全量 1,072 → 1,340（+268），但记忆集是否新增未知（下轮必重取 11-20 memory200）；11-21 批次已公告（全量 248、记忆集 21 篇全 v1，筛查归 128 轮）；11-22/11-23 仍未公告；118 篇零新版；七项复现零新增第九十三轮；M5 直接证据仍为零、相邻证据 57 项（暂定）；letta-code 仍 v0.34.70（窗口内新增提交 4 项，均非版本类；PyPI 差距仍 58 版）；**Mem0 cfemail「重随机化计数」经实测证伪、予以废止（55–69 度历史计数不再作为页面变化证据；今后只跟 dpl token / clientVersion / 可见文本，C-127-10）**；Mem0 Mintlify 再部署一次（clientVersion 0.0.4277 → 0.0.4286，可见词数不变）；**Graphiti v0.32.0 轮间发布、GitHub Release 对象被误删、0.32.1 接替（#2207/#2208 窗口内）**；轮间合入 #2178/#2198/#2203；mcp-v1.1.4 发布但 Docker 镜像发布失败（#2205/#2206）；Graphiti 星数 +5 精确配对、forks ±0、open 579；**事件流配对方法修正：单次关窗后取回会漏事件，配对须用本轮各次取回并集（C-127-12）**；#2126 连续第三十轮未定位；#2144/#2149/#2150/#2156 均未到关闭日；#2159 沉默 172.9 小时、#2165 137.1 小时、MIRIX #156 174.6 小时；MIRIX +1；Dual-Layer 第 49 例翻转（NASA ADS → SciX）；MemoryAtlas 第 43 轮稳固；ENGRAM 第 102 次 401；DolphinBench、Zep 零动静。
- 纠正：Mem0 cfemail 计数废止（126 轮「第 69 度」及此前全部计数不再作为页面变化证据）；Graphiti「0.32.0 尚未发布」说法被取代；事件流配对改用并集口径（单次关窗后取回漏 21 起事件）。
- 裁定：Mem0 cfemail 计数废止（C-127-10）；Graphiti v0.32.0 发布状态（C-127-11）；事件流配对并集口径（C-127-12）。
- 剩余未核实缺口：11-21/11-22/11-23 批次复核（11-20 memory200 必重取）；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第三十轮）；123 轮 forks 缺口（净 −1 对事件流 +1，相差 −2）仍未定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）；Graphiti #2206 Docker 镜像修复后续。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（10:09 轮，第 128 轮）

- 本轨增量：11 条（M-128-01～11），全部落入 skill/references/layered-memory-rounds.md。**11-20 记忆集 62 → 77（+15，全 v1），127 轮未决项闭合**；11-21 记忆集 21 → 40（+19，全 v1）；11-22 已公告（全量 740、记忆集 31 → 41，迟入未止）；**11-23 关窗后公告（全量 512、记忆集 37）**；11-19 迟入复核（全量 +27，记忆集 111 → 114）、11-18 迟入复核（全量 +24，记忆集 104 → 106）；两篇同名不同工作的 ENGRAM 已消歧（2611.28356 Meta 调参与 2611.28136 视频世界模型）；118 篇零新版；七项复现零新增第九十四轮；M5 直接证据仍为零、相邻证据 57 → 62 项（暂定）；letta-code 仍 v0.34.70（窗口内新增提交 1 项，非版本类；PyPI 差距仍 58 版）；Mem0 无新部署、指南页 Framer 构建来回翻转一次（内容零变化）、双口径官方调和仍缺；**Graphiti #2206 窗口内获首个人工批准（05:05:06Z），mergeable_state 由 blocked 变 clean，尚未合入**；0.32.1 与 MCP 1.1.5 仍未发布；Graphiti 星数 +2 / forks +1 均精确配对、open_issues 579（±0）；#2126 连续第三十一轮未定位；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159 沉默 175.8 小时、#2165 140.1 小时、MIRIX #156 177.6 小时；Dual-Layer 第 50 例翻转（SciX → NASA ADS）；MemoryAtlas 第 44 轮稳固；ENGRAM 第 103 次 401；MIRIX、DolphinBench、Zep 零动静。
- 纠正：Mem0 cfemail 历史计数作废口径已决（C-127-10），M 腿再问闭合，不另立新裁定（C-128-13）。
- 裁定：Mem0 cfemail 再问闭合（C-128-13）。
- 剩余未核实缺口：11-22/11-23 批次复核（11-23 全轨筛查归 129 轮）；11-20 记忆集 2 篇 v2 与 11-18/11-19 的 8 篇 v2 正文比对；Mem0 双口径官方调和仍缺；Graphiti #2126 动作定位（连续第三十一轮）；123 轮 forks 缺口（净 −1 对事件流 +1，相差 −2）仍未定位；MIRIX forks −1 缺口定位；#2177 FalkorDB add 侧后续 PR；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；#2159/#2165、MIRIX #156 维护方回应；letta-code v0.35.0 复核；M5 深挖综合第 5 项未决问题（暂定）；Graphiti #2206 合入与 0.32.1/MCP 1.1.5 发布。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（13:09 轮，第 129 轮）

- 本轨增量：10 条（M-129-01～10），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（PyPI 差距仍 58 版，下载计数 +1）；Mem0 无新部署，指南页仍处 17:06:24Z 版；**Mem0 cfemail 历史计数全部作废**；Graphiti #2209 新开、#2133/#2134 自动关闭，星数 +4 精确配对；#2126 连续第三十二轮未定位；#2144/#2149/#2150/#2156 关闭日约 11-28/11-29；**Dual-Layer 第 51 例翻转（NASA ADS → SciX）与第 52 例新变体（资助行消失，原因未查）**；MemoryAtlas 第 45 轮稳固；11-23 记忆集 37 篇逐篇筛查；**11-20 记忆集 v2 比对闭合（11 篇）**；M5 直接证据仍为零、相邻证据 62 → 80；arXiv 限流未解（11-22/11-20 记忆集终取各 6 次失败）。
- 纠正：Mem0 cfemail 历史计数全部作废（C-129-17）；128 轮 M-128-10 关于 27166 登记集归属之说纠正（C-129-21）。
- 裁定：Mem0 cfemail 历史计数作废（C-129-17）；11-20 记忆集 v2 比对闭合（C-129-22）；M5 相邻证据 62 → 80（C-129-23）；R7/M5/S1 三项暂定续记（C-129-24）。
- 剩余未核实缺口：11-22/11-20 记忆集终取（arXiv 429/503 失败）；2611.24848v2 已撤回、v1 引用状态未决；Dual-Layer 第 52 例新变体原因；#2126 第三十三轮；#2144/#2149/#2150/#2156 关闭日；Mem0 双口径调和；Graphiti #2206/0.32.1/MCP 1.1.5/#2209；M5 直接证据仍为零（暂定续记）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（16:09 轮，第 130 轮）

- 本轨增量：10 条（M-130-01～10），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（PyPI 差距仍 58 版，下载计数全未变）；Mem0 指南页新 Framer 构建为本轮首次观测（04:18:38Z，不得写成窗口内发布），README 删万圣节横幅 10 行；Graphiti #2206 仍未合入、0.32.1/MCP 1.1.5 仍未发布，星数 +4 精确配对；**11-22/11-20/11-19/11-18 记忆集终取全部闭合**（41/77/114/106 篇）；**11-20 记忆集 v2 比对闭合（2 篇：MixFold 纯版本戳、WePM 公式方向订正）**；**2611.24848 v1 仍可引用（仅 v2 被撤回），129 轮暂定项闭合**；M5 直接证据仍为零、相邻证据 80 → 93（13 篇全文深读，均不升格；最接近 #81 GlanceNav）；**Dual-Layer 第 52 例新变体持续**（资助行仍消失）；MemoryAtlas 第 46 轮稳固；11-23 记忆集 37 → 70（+33 迟入）；#2126 连续第三十三轮未定位；#2210 关窗后新建（归 131 轮）；ENGRAM 第 106 次 401。
- 纠正：2611.24848 v1 引用状态（129 轮暂定项闭合，C-130-16）；Mem0 新构建不得写成窗口内发布（C-130-27）。
- 裁定：M5 相邻证据 80 → 93（C-130-14）；11-20 记忆集 v2 比对闭合（C-130-15）；R7/M5/S1 三项暂定续记（C-130-23）。
- 剩余未核实缺口：11-24 批次公告（预计 2026-11-25 01:00Z）；2611.30932 全文深读（归 SA/RA 腿）；Dual-Layer 第 52 例新变体原因；#2126 第三十四轮；#2144/#2149/#2150/#2156 关闭日；Mem0 双口径调和；Graphiti #2206/0.32.1/MCP 1.1.5/#2209/#2210；M5 直接证据仍为零（暂定续记）。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（19:09 轮，第 131 轮）

- 本轨增量：10 条（M-131-01～10），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（PyPI 差距仍 58 版，下载计数全未变）；Mem0 无新 Mintlify 再部署，两页本轮内 cfemail-only 抖动各一次（连续第三轮 cfemail-only 抖动）；**Graphiti 窗口内新建 #2210（issue，add_episode_bulk 批量矛盾检测 bug）/#2211（修复 PR，+404/−9）**；Graphiti 星数 +5、forks +1、open +2，并集配对精确；#2206 仍未合入、0.32.1/MCP 1.1.5 仍未发布；#2126 连续第三十四轮未定位；**11-22/11-21/11-20/11-19/11-18 记忆集终取全部闭合**（41/40/77/114/106 篇）；**11-23 记忆集 70 篇稳固零迟入**；M5 直接证据仍为零、相邻证据 93（零新增）；**Dual-Layer 第 52 例新变体持续**（资助行仍消失）；MemoryAtlas 第 47 轮稳固（关窗后重取首次观测到静态 chunk 哈希轮换，不得写成窗口内发布）；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭；ENGRAM 第 108 次 401；七项复现零新增第九十七轮。
- 纠正：MemoryAtlas 静态 chunk 哈希轮换不得写成窗口内发布（关窗后首次观测，C-131-21）。
- 裁定：M5 相邻证据 93（零新增，C-131-20）；R7/M5/S1 三项暂定续记（C-131-23）。
- 剩余未核实缺口：11-24 批次公告（预计 2026-11-25 01:00Z）；Dual-Layer 第 52 例变体原因；#2126 第三十五轮；#2144/#2149/#2150/#2156 关闭日；Mem0 双口径调和；Graphiti #2206/#2210/#2211/0.32.1/MCP 1.1.5；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-24（22:09 轮，第 132 轮）

- 本轨增量：10 条（M-132-01～10），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（PyPI 差距仍 58 版，下载计数仅 v0.34.70 +1）；Mem0 无新 Mintlify 再部署，两页本轮内 cfemail-only 抖动各一次（连续第四轮 cfemail-only 抖动）；**Graphiti 0.32.1 窗口内发布**（0.32.0 系被 0.32.1 替换、非误删）、**#2206/#2208 合入**、#2207/#2205 关闭；Graphiti 星数 +6、forks ±0、open −2，并集配对精确；窗口内新建 #2212/#2213；MCP 1.1.5 关窗后 6 秒发布，不计入窗口；**Zep v4.2.0 窗口内发布**；#2126 连续第三十五轮未定位；**11-23 记忆集 70 篇连续第二轮零迟入**；11-22/11-21/11-20/11-19/11-18 记忆集终取 41/40/77/114/106 全部 ±0；M5 直接证据仍为零、相邻证据 93（零新增）；**Dual-Layer 第 52 例变体未闭合**（资助行轮内闪现又消失）；MemoryAtlas 第 48 轮稳固；ENGRAM 第 110 次 401；七项复现零新增第九十八轮。
- 纠正：0.32.0 被 0.32.1 替换（非误删），缺口 1 就此更新（C-132-15）；#2206/#2208 由「仍未合入」修正为已合入。
- 裁定：M5 相邻证据 93（零新增，C-132-17）；R7/M5/S1 三项暂定续记（C-132-24）。
- 剩余未核实缺口：11-24 批次公告（预计 2026-11-25 01:00Z）；Dual-Layer 第 52 例变体原因；#2126 第三十六轮；#2144/#2149/#2150/#2156 关闭日；Mem0 双口径调和；Graphiti #2210/#2211/#2212/#2213、MCP 1.1.5；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-25（01:09 轮，第 133 轮）

- 本轨增量：9 条（M-133-01～09），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（窗口内新提交 2 项，自 v0.34.70 以来提交数 9 → 11；提交机器人改名为 letta-integration[bot]；PyPI 差距仍 58 版）；**Mem0 新 Mintlify 再部署本轮首次观测**（上线只 bracket 在 18:10Z 之后、20:10Z 之前，不得写成窗口内发布）；**Graphiti MCP 1.1.5 发布说明已核对**（发布于 17:09:14Z，归本轮）；#2212/#2213 本体首次取回；#2146 窗口内由作者关闭未合入；Graphiti 星数 +6、forks +3，均精确配对；#2126 连续第三十六轮未定位；Dual-Layer 第 52 例变体未闭合；MemoryAtlas 第 49 轮稳固；ENGRAM 第 112 次 401；11-23 记忆集 70 篇连续第三轮零迟入；M5 相邻证据 93（零新增）；m-union133.py 路径拼接 bug 已修复并验证（主轮重跑通过）。
- 纠正：m-union133.py 路径拼接 bug 已用 os.path.join 修复（M-133-08）。
- 裁定：Mem0 再部署不得写成窗口内发布（C-133-18）；M5 相邻证据 93（零新增，C-133-20）；R7/M5/S1 三项暂定续记（C-133-20）。
- 剩余未核实缺口：11-24 批次公告（预计 2026-11-25 01:00Z）；Dual-Layer 第 52 例变体原因；#2126 第三十七轮；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；Mem0 双口径调和；Graphiti #2210/#2211/#2212/#2213、MCP 1.1.5；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-25（04:09 轮，第 134 轮）

- 本轨增量：9 条（M-134-01～09），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（窗口内新提交 0 项，自 v0.34.70 以来提交数仍 11；PyPI 差距仍 58 版）；**Mem0 新 Mintlify 构建 0.0.4304 窗口内上线**（上线 bracket 20:10:56Z–21:38:20Z，整体在窗口内；可见文本零变化）；Graphiti #2210/#2211/#2212/#2213 仍 open，**#1769 窗口内新评论认领修复**（rohhann12 20:23:44Z）；Graphiti stars 33,066（±0）、forks +1（rohhann12 20:13:14Z 精确配对）、open 577（±0）；#2126 第三十七轮未定位；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；MIRIX stars 3,517（±0）；ENGRAM 第 114 次 401；**Dual-Layer 第 52 例变体未闭合**（四次取样不一，两次关窗后取回 56 秒相隔、结果相反）；MemoryAtlas 第 50 轮稳固；11-23 记忆集 70 篇连续第四轮零迟入；arXiv 慢速路连续第三轮零 429/503；M5 直接证据仍为零、相邻证据 93（零新增）；七项复现第一百轮零新增。
- 纠正：Mem0 0.0.4304 构建记为窗口内再部署（与 133 轮 0.0.4298 bracket 横跨关窗点不同，C-134-19）。
- 裁定：letta-code 0.34.70（C-134-18）；Mem0 0.0.4304 窗口内上线、可见文本零变化、双口径调和仍缺（C-134-19）；Graphiti #2210–#2213 open、#1769 认领、计数精确配对（C-134-20）；#2126 第三十七轮、关闭日、MIRIX 3517、ENGRAM 第 114 次 401（C-134-21）；Dual-Layer 第 52 例变体未闭合、MemoryAtlas 第 50 轮稳固（C-134-22）；记忆集 70 篇连续第四轮零迟入、M5 零直接证据（C-134-23）。
- 剩余未核实缺口：11-24 批次公告（预计 2026-11-25 01:00Z）；Dual-Layer 第 52 例变体原因；#2126 第三十八轮；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；Mem0 双口径调和；Graphiti #2210/#2211/#2212/#2213、#1769 认领修复跟进、MCP 1.1.5 采用；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-25（10:09 轮，第 135 轮）

- 本轨增量：9 条（M-135-01～09），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（窗口内新提交 2 项，自 v0.34.70 以来提交数 11 → 13；PyPI 差距仍 58 版）；Mem0 0.0.4304 构建稳定（无新部署，可见文本零变化）；Graphiti #2210/#2211/#2212/#2213 仍 open，MCP 1.1.5 仍为最新；Graphiti stars 33,066 → 33,070（+4 精确配对）、forks 3,408 → 3,409（窗口内 +1，szlixg 23:47:01Z）、open 577（±0）；#2126 第三十八轮未定位；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；MIRIX stars 3,517（±0）；ENGRAM 第 115、116 次 401；Dual-Layer 第 52 例变体未闭合；MemoryAtlas 第 51 轮稳固；**11-23 记忆集 70 → 89（+19 迟入，连续四轮零迟入终结）**；11-24 批次窗口内公告，首次数到记忆集 16 篇 / 全量 169（部分日快照）；**M5 相邻证据 93 → 95**（新增 VG-GP #94、Dreamcatcher #95）；七项复现第一百零一轮零新增。
- 纠正：11-23 记忆集 70 篇「连续四轮零迟入」说法随 +19 迟入作废，改为「连续四轮零迟入终结」（C-135-07）；批次终值方法论订正（须以次日 01:00Z 公告档后复核为准，C-135-09）。
- 裁定：letta-code 0.34.70（C-135-01）；Mem0 0.0.4304 构建稳定、可见文本零变化、双口径调和仍缺（C-135-02）；Graphiti #2210–#2213 open、计数精确配对（C-135-03、C-135-04）；#2126 第三十八轮、关闭日、MIRIX 3517、ENGRAM 第 115、116 次 401（C-135-05）；Dual-Layer 第 52 例变体未闭合、MemoryAtlas 第 51 轮稳固（C-135-06）；记忆集 70 → 89、M5 相邻证据 93 → 95（C-135-07、C-135-08）。
- 剩余未核实缺口：11-24 批次终值；11-25 批次公告；Dual-Layer 第 52 例变体原因；#2126 第三十九轮；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；Mem0 双口径调和；Graphiti #2210/#2211/#2212/#2213、#1769 认领修复跟进、MCP 1.1.5 采用；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-25（13:09 轮，第 136 轮，版本 0.133.0 → 0.134.0）

- 本轨增量：9 条（M-136-01～09），全部落入 skill/references/layered-memory-rounds.md。letta-code 仍 v0.34.70（窗口内新提交 0 项；PyPI 差距仍 58 版）；Mem0 0.0.4304 构建稳定（可见文本零变化）；Graphiti 窗口内新 #2214/#2215（同作者 rithindattag，OpenAIRerankerClient 解包修复；CLA 已签署；仍 0 评审）；Graphiti stars +2/forks +1 精确配对、open 577 → 579；#2210 关窗后首条评论（志愿者 zlg-codes，非维护者）；#2211/#2213 仍未评审；#2126 第三十九轮未定位；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；MIRIX stars 3,517（±0）；ENGRAM 第 117、118 次 401；Dual-Layer 第 52 例变体未闭合；MemoryAtlas 第 52 轮稳固；**11-24 记忆集 16 → 94（非终值）**；11-23 记忆集 89 第三度稳定；**M5 相邻证据 95 → 98**（Leto #96、KeyMemVLA #97、CoverMem #98）；七项复现第一百零二轮零新增；arXiv 慢速路连续第五轮零 429/503。
- 纠正：无（本轨无推翻项）。
- 裁定：letta-code 0.34.70（C-136-01）；Mem0 0.0.4304 构建稳定、可见文本零变化、双口径调和仍缺（C-136-02）；Graphiti #2214/#2215 新建、计数精确配对（C-136-03）；#2126 第三十九轮、关闭日、MIRIX 3517、ENGRAM 第 117、118 次 401（C-136-04、C-136-05）；Dual-Layer 第 52 例变体未闭合、MemoryAtlas 第 52 轮稳固（C-136-06）；记忆集 16 → 94、M5 相邻证据 95 → 98（C-136-07、C-136-08）。
- 剩余未核实缺口：11-24 批次终值；11-25 批次公告；Dual-Layer 第 52 例变体原因；#2126 第四十轮；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；Mem0 双口径调和；Graphiti #2210/#2211/#2212/#2213/#2214/#2215、#1769 认领修复跟进、MCP 1.1.5 采用；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。

## 2026-11-25（16:09 轮，第 137 轮，版本 0.134.0 → 0.135.0）

- 增量：M-137-01–11（Graphiti 新建 13 项、#2221 开出；#2210 两 PR 同方案；星数 +2/forks +1 精确配对；#2126 第四十轮；letta-code 仍 v0.34.70、PyPI 差距 58；Mem0 0.0.4304 稳定、双口径调和仍缺；Dual-Layer 第 53 例新变体、MemoryAtlas 第 53 轮稳固；11-24 记忆集 94 第三度稳定；M5 直接证据仍为零、七项复现第 103 轮；ENGRAM 第 119、120 次 401、arXiv 429 复发；#2201 认领开出 #2222、MCP 1.1.5）。
- 裁定：letta-code 0.34.70（M-137-05）；Mem0 0.0.4304 构建稳定、可见文本零变化、双口径调和仍缺（M-137-06）；Graphiti #2216–#2228 新建、计数精确配对（M-137-01、M-137-03）；#2126 第四十轮、关闭日、MIRIX 3517、ENGRAM 第 119、120 次 401（M-137-04、M-137-10）；Dual-Layer 第 53 例新变体、MemoryAtlas 第 53 轮稳固（M-137-07）；记忆集 94 第三度稳定、M5 相邻证据仍 98（M-137-08、M-137-09）。
- 剩余未核实缺口：11-24 批次终值；11-25 批次公告；Dual-Layer 第 52/53 例变体原因；#2126 第四十一轮；#2144 约 11-28、#2149/#2150/#2156 约 11-29 关闭日；Mem0 双口径调和；Graphiti #2210/#2211/#2220 维护方取舍、#2212 第四十一轮、#1769/#2221 CLA 与合入、MCP 1.1.5 采用；M5 直接证据仍为零（暂定续记）；M5 联合约束暂定。
- 下轮起点：先清上述缺口，再按子主题轮转（记忆检索排序与遗忘策略）。
