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
