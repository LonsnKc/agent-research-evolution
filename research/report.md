# 智能体系统调研持续进化：总报告

计划：API 路由 / Agent Harness / 分层记忆 / 技能四条轨道，每日一轮；每轮更新各轨道笔记、在此追加一节轮次小结，并把已核实增量同步进 `agent_system_design` 技能参考。SOUL.md 与全局配置只作提案，永不自动改动。

基线：
- 2026-10-23 四主题深研报告：`~/workspace/research_notes/api-routing-harness-memory-skills-20261023-1442/report.md`
- 2026-10-24 技能深研主报告与已核实缺口：`~/workspace/research_notes/skill-deep-research-20261024/report.md`、`gaps-round2.md`
- 技能参考：`~/workspace/custom-skills/agent_system_design/`（SKILL.md + references/ 四篇）

轨道笔记：
- `track-api-routing.md`、`track-harness.md`、`track-layered-memory.md`、`track-skills.md`

---

## 2026-10-25 轮次小结（首轮，水位 2026-10-24 → 2026-10-25）

### 窗口诚实性声明

24 小时窗口内四条轨道均未发现落入窗口的一手新发布（Claude Code 官方 changelog 最新条目仍为 2026-10-23；agentskills.io 规范页更新戳仍为 2026-08-04）。本轮实质增量 = 水位日前后的基线补漏 + 每轨一个子主题的细颗粒深化。凡把补漏写成“新进展”都会误导选型，故各轨道笔记严格分区标注。下轮若仍无新版本，建议把窗口放宽为“自基线以来”并明示。

### 跨轨道结论

1. **网关层安全成为本轮最大主题（路由轨）**：LiteLLM 1.107.0 投毒事件的官方根因分析（10-24 CST 发布）表明，层 C 网关集中持有全部上游密钥，其供应链安全（版本 pin + hash 校验、发布门禁、密钥隔离）应与路由算法同等权重；同期 Railway 事件与 OpenRouter 暗网传闻（后者未证实）进一步佐证。给用户的提案（不自动应用）：是否把“网关供应链安全”列为 skill 选型清单的硬性门槛。
2. **评估方法论在三轨同时收紧（Harness / 技能 / 记忆三轨共识）**：Harness 轨要求引用分数锁定五项（模型、harness 版本、任务集版本、资源上限、运行次数）；技能轨要求同时报告触发/加载率、检索召回、通过率、token/时延差（Wild 五档阶梯；ACES 证明静态扫描与 live 效果相关性仅 ρ=0.14）；记忆轨要求“遗忘有效”宣称同时报告当前真值问答、历史问答、误删/误留率三项。共同教训：只报一个通过率数字的评估都不可比。
3. **“决策”正在从聊天 API 中分化（路由轨）**：vLLM 0.32.0、LiteLLM 1.105.0、OpenAI Decisions 格式同时出现结构化决策端点（`/v1/systemone`、`/v1/decisions`）；层 B 配置同时收敛为声明式 policy（Dynamo v1.6.0 的 policy YAML、llm-d 1.0 的 Gateway API 形态）。
4. **发现/冲突机制继续分裂而非收敛（技能轨）**：同名冲突已有四种互不兼容的解决行为（DSH rank/作用域、OpenCode 路径派生 ID、Codex 目录隔离、Claude `<folder>:<name>` 命名空间化）；跨宿主“一份正本”只能靠避免同名，无法靠统一规则。
5. **排序与遗忘必须分层（记忆轨）**：降权（只影响排序）≠ 失效（改变当前真值集、保留历史）≠ 删除（合规/安全硬删除且级联）；Mem0 v3 的 ADD-only 模式把遗忘从写时删除移到了读时排序与时间过滤，存储增长与阈值重调是其显性成本；厂商分数跨页不一致（LongMemEval 93.4 vs 94.4/94.8），不可采信单一数字。
6. **长程 harness 的一手源补齐（Harness 轨）**：Anthropic 长程 harness 文章由“仅二手”升级为一手核实（2025-11-26 发布，2026-10-18 更新），initializer + coding agent 两件套与 JSON feature 列表细节已确认；LangChain/LangGraph 1.0、Agent Framework/AutoGen 维护、AgentKit 三项仍为二手，待官方复核。

### 本轮文件变更

- 轨道笔记：四份各追加 2026-10-25 轮次小节（含已核实/未核实分区与剩余缺口清单）。
- 技能参考：api-routing.md（安全事件注记、Dynamo/vLLM 版本刷新、结构化决策路由行、Cluster Route Escalate 论文、两条设计检查）、agent-harness.md（长程 harness 一手升级、AutoGen/LangGraph/AgentKit 行、eval harness 与 trace 格式两节、失败模式两条、MCP 去重一句）、layered-memory.md（Mem0 v3 限定解除、多信号融合审计、按类型过期决策表、三条新风险、数字口径警示）、skill-authoring.md（Claude 命名空间化、触发/召回/通过率分离评估小节、JFrog 工作站清点、Gemini 消费级分歧注记）。
- SKILL.md 本轮无改动；SOUL.md 与全局配置无改动（给用户的提案仅一项：网关供应链安全是否列为选型硬性门槛）。

### 剩余未核实缺口汇总（跨轨，下轮复核清单）

- 路由：LiteLLM 安全版本 1.106.2、Dynamo v1.5.0 细节、Portkey 收购、RouteLLM 维护状态、Railway/OpenCode 事件影响范围。
- Harness：LangChain/LangGraph 1.0、Agent Framework 1.0、AgentKit 官方公告原文；Harbor/METR 官方 docs；OTel GenAI 官方 spec；Anthropic 2026 年后续文章一手原文。
- 记忆：Mem0 v3 全部基准分数；REM/EcphoryRAG 效果数字；REM 提交日；LangMem/Letta/Graphiti 官方 changelog 直查。
- 技能：OpenCode 2.0.30 开关、Cursor `/skills` 校验、Claude 2.1.299 钩子顺序、Mobb.ai 审计原文、Google 铺开“延迟”说法、AAIF 落地文件、DSH/Hermes/Codex/Copilot/Windsurf 官方更新。

---


## 2026-10-27 轮次小结（缺口复核轮，水位 2026-10-25 → 2026-10-27；2026-10-26 轮次因平台安全审查跳过）

### 窗口诚实性声明

本轮是模板化改造（v0.1.0 基线）后的首个执行轮次，也是缺口复核轮：PRACTICAL.md 开始时三节全空，无实战待修正任务；本轮实质是把上一轮大批二手结论升级为一手，并推翻一项上一轮结论。凡属复核与补漏均明说，不冒充当日新闻。上一轮 research/refined 目录为空（无上一轮 verification.md 可复核），本轮自验为该模板下的首次自验。

### 跨轨道结论

1. **二手转一手是本轮主线（Harness / 技能 / 记忆三轨）**：LangChain 1.0、Agent Framework 1.0、AutoGen 存档、Harbor 现行格式、OpenCode 开关、Claude 钩子、Cursor 归属、Mem0 分数口径、Zep v4 等上一轮二手或存疑项，多数已升级为一手或得到纠正。
2. **一项上一轮结论被官方原文推翻（Harness 轨）**：AgentKit「未见官方停用通知」→ OpenAI 官方 deprecations 文档确认 2026-06-03 宣布弃用、2026-11-30 停用（Evals 2026-10-31 起只读）。选型上 Agent Builder 与托管 Evals 已无新项目采用价值，迁移方向为 Agents SDK / Workspace Agents / Promptfoo。
3. **网关供应链处置口径收紧（路由轨）**：LiteLLM 钉版从「1.106.2」推进为两段式（1.106.3/1.106.4 + 新 cosign 密钥），且「升级即安全」是错误处置——必须清 loader、重建环境、轮换凭证。1.106.3/1.106.4 为二手转述一手，登记为本轮唯一未验证暂定项。
4. **评估数字必须带条件引用（记忆轨方法论收紧）**：Mem0 三数之争解决为「截断 × Temporal 条件」四格，另 94.8% DMR 归属 Zep；官方博客自身还有一处截断标注错误，引用须以 Results 节为准。教训：同一厂商的数字也须按条件拆分后引用。
5. **规范与治理的「宣布 vs 落地」要分开（技能轨）**：AAIF 治理转移已确认，但版本化发布、提案流程、安全指南仍为提案；规范页更新戳仍为 2026-08-04。把宣布写成落地是本轮纠正的另一类错误。

### 本轮文件变更

- 调研产出：`research/20261027-1003/report.md + sources.md`（来源分级 A/B/C/V，含未打开清单）。
- 提炼与自验：`refined/20261027-1003/delta.md`（20 条增量：R4 + H6 + M6 + S4）与 `verification.md`（执行者自验：通过 19 条、通过（未验证暂定）1 条、需修正 0 条、否决 0 条；独立抽查 4 条，含自验者亲自打开 OpenAI deprecations 页与 agentskills.io 规范页）。
- 技能参考：四篇 references 增量编辑（SKILL.md 无改动；行数 97 / 62 / 61 / 93 / 94，均在红线内）；旧版整目录快照于 `versions/v0.1.0/`；VERSION 0.1.0 → 0.2.0。
- 台账：STATE.md、CHANGELOG.md（v0.2.0）、PRACTICAL.md（新增未验证暂定 1 条）、四轨 track 笔记各追加 2026-10-27 小节、水位文件更新为 2026-10-27。
- SKILL.md 级提案：无。SOUL.md 与全局配置无改动。

### 剩余未核实缺口汇总（跨轨，下轮复核清单）

- 路由：LiteLLM 官方 releases 页直接确认 1.106.3/1.106.4（未验证暂定首要项）；RouteLLM 提交历史页直接确认最后提交日；Dynamo v1.5.0 细节；Portkey 2.0 正式版；Railway/OpenCode 事件影响范围。
- Harness：AgentExecutor 维护日期；AutoGen 存档日期与旧横幅原文；OTel GenAI 官方 spec；Anthropic 2026 年后续文章一手原文。
- 记忆：Mem0 分数真实性（口径已统一，仍无独立复现）；Graphiti 版本冲突；LangMem 取证不完整；FrontierMemory 官方原文。
- 技能：UserPromptExpansion 首发版本；Mobb.ai 原文；Google 铺开「延迟」说法；AAIF 落地文件持续检索。


---

## 2026-10-28 轮次小结（缺口复核 + 新增覆盖轮，版本 0.2.0 → 0.3.0）

- 跨轨结论：本轮是缺口复核轮，上一轮的大部分剩余缺口得到解决或纠正——LiteLLM 钉版由未验证暂定升级为已确认、OTel 官方 spec 落地、Graphiti 冲突解决、LangMem 与 FrontierMemory 确认；同时纠正了两条上一轮结论（AgentExecutor 维护口径、Google 延迟表述）。严格落在 2026-10-28 当日的一手发布四轨均检索未见；落在 2026-10-27 的一手新条目为 LiteLLM 1.106.5 批与 Claude Code changelog 技能修复。
- 文件变更：research/20261028-0950/report.md + sources.md；refined/20261028-0950/delta.md（17 条）+ verification.md（执行者自验：通过 17 条、未验证暂定 0 条、需修正 0 条、否决 0 条；独立抽查 5 条一手来源）；技能正本四篇 references 增量编辑（SKILL.md 本体未改）；旧版快照 versions/v0.2.0/；VERSION 0.3.0；CHANGELOG v0.3.0；STATE.md、PRACTICAL.md（上一轮未验证暂定项闭环，记入已修正）、四轨 track 笔记各追加本轮小节。
- 剩余未核实缺口汇总：RouteLLM 最后提交日；Dynamo v1.5.0 细节；Railway / OpenCode Console 影响范围；Portkey 2.0 正式版；AutoGen 存档日期与旧横幅原文；UserPromptExpansion 首发版；Mobb.ai 原文；CVE-2026-114612 官方公告；TEPA 论文原文；Mem0 Decay 开源支持范围；Zep Cloud SDK 4.0.1；Anthropic 折叠 subagent 文章一手原文；OpenCode v2.2.0 技能专项。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。


---

## 2026-10-29 轮次小结：缺口复核 + 新增覆盖（版本 0.3.1 → 0.4.0）

- 跨轨结论：四轨在 2026-10-28 之后均无以官方页面日期确认的一手新发布；本轮增量全部为缺口复核（两项缺口关闭：TEPA 原文、Mem0 Decay 裁决；一项否定性解决：OpenCode v2.2.0 无技能专项）与版本戳推进（LiteLLM 1.106.6、Dynamo v1.6.1、Agent Framework 1.23.0、Agents SDK v0.24.0、Letta 0.34.26 见 npm 侧）。
- 文件变更：技能正本四篇 references 增量编辑（18 条技能增量：R1–R5、H1–H4、H6–H7、M1–M3、S1–S4；另 2 条仅 track 笔记：H5、M4）；整合前旧版整目录快照于 `versions/v0.3.1/`；VERSION 0.3.1 → 0.4.0；CHANGELOG 新增 v0.4.0 节（并纠正 STATE 基线滞后：此前记 0.3.0，实为 0.3.1）；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261029-0950/report.md + sources.md + track-*-raw.md`；提炼与自验 `refined/20261029-0950/delta.md + verification.md`（执行者自验：通过 20 条、需修正 0 条、否决 0 条、新增未验证暂定 0 条；独立抽查 5 条一手来源全部吻合）。
- 推翻的旧说法：Dynamo“最新 stable 仍为 v1.6.0”（被 v1.6.1 推翻）；Mem0 Decay“开源支持范围未裁决”（被官方对照页裁决为 OSS 不支持）；OpenCode v2.2.0“技能专项未复核”（解决为无专项）；RouteLLM 第三方 2024-08-10 说法不获提交历史页支持。
- 剩余未核实缺口汇总：CVE-2026-114612 官方修复版；Dynamo v1.5.0 细节；Portkey 2.0 正式版与 1.15.2 出入；Railway 最终报告与 OpenCode post-mortem；各最新版发布日期；Zep Cloud SDK 4.0.1；letta-code 22/23 缺位原因与 0.34.26 变更内容；TEPA v2 改动清单；UserPromptExpansion 首发版；Mobb.ai 原文；AAIF 落地文件；AutoGen 存档日期与 2025-10 横幅原文；Mem0 / REM / EcphoryRAG 数字独立复现。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。


---

## 2026-10-30 轮次小结：缺口复核 + 新增覆盖（版本 0.4.0 → 0.5.0）

- 跨轨结论：本轮最大进展是一项上一轮结论的纠正——Dynamo v1.6.1 最新稳定说法被官方 releases-data 表（单一事实源）推翻并撤回，同时 v1.5.0 / v1.5.1 得到官方表确认；另有三项缺口关闭或纠偏（Zep SDK 4.0.1 语言归属、Mem0 Decay 20 次时间戳复核、工具搜索 85% 与 80% 削减出处的一手确认）与多项否定性解决（Codex 0.165.0 确无技能项、触发面分歧无收敛、AAIF 落地文件仍未发布）。严格落在 2026-10-30 当日的一手发布为 Claude Code changelog 10-30 组（无技能规则变更）；langchain-core 1.7.3 为 10-29 边界日；其余四轨 2026-10-29 之后均检索未见新稳定发布（页面未显示日期的最新版不归入窗口）。
- 文件变更：技能正本四篇 references 增量编辑（21 条技能增量：R1–R5、H1–H7、M1–M4、S1–S5；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.4.0/`；VERSION 0.4.0 → 0.5.0；CHANGELOG 新增 v0.5.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261030-0950/report.md + sources.md`（另含四路调研员原始报告 raw-*.md）；提炼与自验 `refined/20261030-0950/delta.md + verification.md`（执行者自验：通过 21 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 3 条一手来源全部吻合）。
- 推翻的旧说法：Dynamo“最新 stable 为 v1.6.1”（被官方 releases-data 表推翻）；Zep“Cloud SDK 4.0.1”不区分语言的说法（纠偏为 TypeScript 包）；eMudhra 案例归属“Dyanmo v1.6.1”（纠正为 LiteLLM v1.106.3）。
- 剩余未核实缺口汇总：Dynamo v1.6.1 最终解释；CVE-2026-114612 官方公告与修复版；Portkey v1.15.0 日期与 1.15.2 出入成因；Railway 最终报告与 OpenCode post-mortem；RouteLLM 单条提交哈希；LiteLLM v1.106.6 / vLLM v0.32.0 官方日期；METR 三仓逐仓戳；ruvnet/metaharness 仓库页；“10,000 工具”出处；Zep Python 4.0.1；letta-code 0.34.26 确切发布时间与 0.34.22 缺位原因；TEPA v2 正文差异；Graphiti 0.31.2 确切发布日；mem0ai 1.0.10 日期；UserPromptExpansion 首发版；Mobb 34% / 27% 与 mobb.ai 原文；AAIF 落地文件；AutoGen 存档执行日期。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。


---

## 2026-10-31 轮次小结：缺口复核 + 新增覆盖（版本 0.5.0 → 0.6.0）

- 跨轨结论：本轮主进展是缺口关闭与旧说法纠正的集中一轮：LiteLLM v1.106.6 日期、Zep Python 4.0.1、Graphiti 0.31.2 日期、mem0ai 1.0.10 日期、TEPA v2 差异、letta-code bump-only 流水、dotnet-1.26.0 标签身份、METR 两仓存档、ruvnet/metaharness、10,000 工具出处、AutoGen 维护提交与存档表述、Agent Framework 10-30 两线（推翻时点结论）、LangGraph 1.2.19、OpenCode #60315、MCode 数量误读来源。严格 2026-10-31 窗口内（截至上午核查时）四轨均未发现带日期的一手新发布；2026-10-30 当日发布为上一轮上午核查后发布，本轮首次捕获，不计入严格 10-31 窗口。
- 文件变更：技能正本四篇 references 增量编辑（18 条技能增量：R1–R4、H1–H6、M1–M4、S1–S4；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.5.0/`；VERSION 0.5.0 → 0.6.0；CHANGELOG 新增 v0.6.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261031-0950/report.md + sources.md + track-*.md`；提炼与自验 `refined/20261031-0950/delta.md + verification.md`（执行者自验：通过 18 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 4 条一手来源全部吻合：Dynamo 官方表、SDK tool-search 文档、规范页、OpenCode changelog #60315）。
- 推翻的旧说法：Agent Framework“python-1.24.0 不存在 / 未发现 dotnet-1.27.0”（时点结论）；“10,000 工具无出处”（出处在 SDK 文档，口径为目录上限）；letta-code“缺位”（实为 bump-only 发布）；Zep“Python 未见 4.0.1”；TEPA“v2 正文未比对”；“METR/mp4-tasks 为默认任务仓”（实为示例值，公开仓 404）；“AutoGen 已被微软执行存档”（应改述为 README 声明）；LiteLLM“官方 changelog 未列出 v1.106.6”。
- 剩余未核实缺口汇总：Dynamo v1.6.1 最终解释；CVE-2026-114612 官方公告；Portkey 1.15.x 出入；Railway 最终报告与 OpenCode post-mortem；RouteLLM 单条提交；vLLM 官方日期；Harbor 逐项日期；langchain-core 10-30/31 新版；UserPromptExpansion 首发版；Mobb.ai 原文；AAIF 落地文件；letta-code bump 版时间戳与 0.34.33 变更；PyPI `letta` 改道原因；TEPA TeX diff；Mem0 成绩口径差异。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。


---

## 2026-11-03 轮次小结：缺口复核 + 新增覆盖（版本 0.6.0 → 0.7.0）

- 跨轨结论：本轮距上一轮间隔三日，窗口为 2026-10-31 之后至 2026-11-03 核查时止；窗口内带日期的一手正式发布共五项：langchain-core 1.7.4（11-02）与 letta-code 0.34.36（10-31）、0.34.37 / 0.34.38 / 0.34.39（11-02）；另有两项窗口内公告/组更新不带版本号归属：LiteLLM 周二发布 + 1.108 LTS 试点公告、Claude Code 11-03 组（无技能规则变更）。本轮主进展是缺口关闭与旧说法纠正的集中一轮：Dynamo v1.6.1 存在性（修正上一轮撤回口径）、CVE 修复版本线、Portkey 最新版定版、dotnet-1.26.0 一手日期、letta-code bump-only 说法修正为“有 tag、无 Release 对象”、TEPA 源码级 diff、Mem0 口径差异解释、Memora 原文复核、Claude 10-30 组版本号一手确认、“7 行变动”误植排除、MCode 数量清点。
- 文件变更：技能正本四篇 references 增量编辑（25 条技能增量：R1–R7、H1–H6、M1–M7、S1–S5；另 8 条仅 track 笔记处理：T1–T8；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.6.0/`；VERSION 0.6.0 → 0.7.0；CHANGELOG 新增 v0.7.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261103-0950/report.md + sources.md + track-*.md`；提炼与自验 `refined/20261103-0950/delta.md + verification.md`（执行者自验：通过 25 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 4 条一手来源全部吻合：Dynamo 官方表、LiteLLM LTS 博客、规范页、PyPI langchain-core 1.7.4）。
- 推翻的旧说法：Dynamo“撤回 v1.6.1 说法”（修正为仅撤回否定存在）；letta-code“bump-only 极简发布”（修正为有 tag、无 Release 对象）；Claude 10-30 组“仅镜像一致”（官方 release 页一手确认 v2.1.314）；MCode“5 数量不得引用”（修正为目录清点事实成立）；Mem0“无 TR 93.4 / 90.4、有 TR 94.4 / 94.8”口径（修正为迁移快照 vs 现行 headline 并列）；TEPA“未做源码 diff”（已完成，仅 6 行差异）；dotnet-1.26.0“日期仍二手”（NuGet 一手确认）；ruvnet/metaharness“仓库页未直接确认”（2026-10-31 已确认）。
- 剩余未核实缺口汇总：CVE 独立厂商公告；Portkey v1.15.0 日期与出入成因；Railway 最终报告；OpenCode post-mortem；RouteLLM 单条提交；vLLM 官方页内日期戳；Harbor 0.21.0 及更早日期；metaharness 逐项日期；langchain-core 1.7.4 变更内容与 GitHub 侧日期；langchain 主包窗口核查；AutoGen 横幅引入提交；AgentTriggered Learn 文档节；OTel GenAI 官方变动；PyPI `letta` 改道原因；Memora / MemCut / MemLeak / DyadMem 数字独立复现；Scaling Discovery 39% / 15% 原文；UserPromptExpansion 首发版；Mobb.ai 原件与 34% / 27%；AAIF 落地文件；Windsurf 转 skills 日期；Claude 11-03 组版本号一手确认；LiteLLM LTS 试点细则；Portkey CVE-2026-110719 厂商修复版。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。

---

## 2026-11-04 轮次小结（缺口复核 + 新增覆盖轮，版本 0.7.0 → 0.8.0）

- 跨轨结论：严格窗口内（2026-11-03 之后至 2026-11-04 上午核查时）四轨均未发现带日期的一手新稳定发布；唯一窗口内新组为 Claude Code 2.1.318（无技能子系统新规则）。本轮价值在缺口复核与口径修正：Portkey 出入成因定位、Claude 11-03 组版本号一手确认、Devin Playbooks 日期锁定、Scaling Discovery 口径收窄、“Memora”消歧四项为实质进展；其余多为复核戳推进与否定性结论。
- 文件变更：技能正本四篇 references 增量编辑（11 条技能增量：R1–R3、H1–H2、M1–M3、S1–S3；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.7.0/`；VERSION 0.7.0 → 0.8.0；CHANGELOG 新增 v0.8.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261104-1000/report.md + sources.md`；提炼与自验 `refined/20261104-1000/delta.md + verification.md`（执行者自验：通过 11 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 4 条一手来源全部吻合：Claude v2.1.317 release 页、Devin 技能文档页、规范页、Scaling Discovery 摘要原文）。
- 推翻的旧说法：Portkey“通道漂移假说”（修正为列表页标题行丢失所致）；Claude 11-03 组“版本号不写入”（官方 release 页一手确认 v2.1.317，限定解除）；Agent Framework .NET“1.26.0 为最新”二手说法（与 NuGet 一手表 1.27.0 冲突，属过期口径，拒绝采纳）。
- 剩余未核实缺口汇总：CVE 独立厂商公告与 1.106.6 含修复确认；Portkey v1.15.0 日期与 CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；RouteLLM 单条提交（须主题/PR 号）；vLLM 页内日期戳；LiteLLM 1.108 稳定版与 LTS“新端点”清单；Harbor 0.21.0 日期；metaharness 0.4.19 日期；langchain-core 1.7.4 标签页一手日期与完整变更；AutoGen 横幅引入提交与维护句引入日期；AgentTriggered 参数语义；#8708 设计文档；OTel GenAI commit 级核查；PyPI `letta` 改道原因；Mem0 双口径官方解释；Zep CE 停更官方口径；七项效果数字与 Scaling Discovery 数字的独立复现；UserPromptExpansion 首发版；mobb.ai 原件与 34% / 27%；AAIF 落地文件；Windsurf 带日期一手证据；OpenCode v2.2.5 归属日期分歧；Claude 2.1.317 / 2.1.318 页内日期字段。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。

---

## 2026-11-05 轮次小结（缺口复核 + 新增覆盖轮，版本 0.8.0 → 0.9.0）

- 跨轨结论：本轮关闭的缺口集中在版本归属与日期口径——GHSA 日期分歧、RouteLLM 单条提交、Portkey v1.15.0 日期、Zep CE 停更、LoCoMo 审计直读、PyPI letta 口径与差距刷新、OpenCode v2.2.5 归属日期、AutoGen 维护句日期裁定；窗口内唯一已确认的新稳定发布是 LiteLLM v1.108.0 与 v1.106.7（均 2026-11-04），其余三轨窗口内均无带日期的一手新稳定发布（Claude Code 2.1.318 与 Gemini CLI v0.67.0 / v0.68.0-preview.0 落在窗口内但均无技能子系统新规则）。深挖层的新共识：路由侧，硬件校准、预算闭环硬约束与 Pool 维是定量模型的三条补强；harness 侧，ATIF 必须钉版本号、OTel GenAI 只是观测趋同层而非对账标准；记忆侧，检索必须写清信号/状态/时间三维、冲突默认撤销留档、基准分数必须附配方；技能侧，同一 SKILL.md 可移植的是文件格式、不是触发语义，触发率与评估完备性须同报。
- 文件变更：技能正本四篇 references 增量编辑（21 条技能增量：R1–R7、H1–H4、M1–M6、S1–S4；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.8.0/`；VERSION 0.8.0 → 0.9.0；CHANGELOG 新增 v0.9.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261105-0950/report.md + sources.md`；提炼与自验 `refined/20261105-0950/delta.md + verification.md`（执行者自验：通过 21 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 5 条一手来源全部吻合：Portkey v1.15.0 Releases API、LiteLLM v1.108.0 release notes、规范页、RouteLLM Commit API、PyPI letta 页）。
- 推翻的旧说法：Portkey v1.15.0“仅知 2026-01、早于 01-12”（被一手 API 日期推翻）；RouteLLM“单条提交日期/哈希未示于页面”（被 Commit API 一手确认）；PyPI letta“分叉的 Python 服务器线 / wheel 发布流水线停滞推断”（被 PyPI 当前文案修正为同一 CLI 的第二种安装通道）；AutoGen“2026-04 维护横幅说”（被二手多源一致 + #7067 一手提交驳回）；OpenCode v2.2.5“TurboAI 11-03 归属日”（判为该页刊出日，非 tag 日）；Gemini CLI“最新稳定版仍 v0.66.0”（时点结论，被 v0.67.0 取代）；LiteLLM“首个周二上午未见 1.108 稳定版”（时点结论，11-04 已发布）。
- 剩余未核实缺口汇总：CVE 独立厂商公告与 1.106.6 含修复确认；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；vLLM v0.32.0 页内日期戳；LiteLLM LTS“新端点”清单；vLLM v0.33.0 稳定版状态；Harbor 0.21.0 日期；metaharness 0.4.19 归属与日期；langchain-core 1.7.4 标签页一手直读；AutoGen 存档引入提交；MAF #8708 与两参数语义；OTel GenAI commit 级一手核查；letta-code 0.34.39 GitHub published_at；Mem0 双口径官方解释；八项数字独立复现；UserPromptExpansion 首发版；mobb.ai 原件与 34% / 27%；AAIF 落地文件；Windsurf 带日期一手证据；Claude 2.1.317 / 2.1.318 published_at；SkillsBench 数字对齐；SkillzBench 方法页；Cursor 技能改进每条日期。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。

---

## 2026-11-07 轮次小结（缺口复核 + 新增覆盖轮，版本 0.9.0 → 0.10.0；六路并行：路由两路、Harness 一路、记忆一路、技能两路，车道健康 6 条已验证可用）

- 跨轨结论：本轮是“缺口收敛”轮——Harness 六项缺口全部解决，SkillsBench 数字对齐、Claude 2.1.317/2.1.318 published_at 定稿、letta-code 0.34.39 published_at、UserPromptExpansion 文档首现、Mobb 口径精确化、Windsurf 迁移向导存在性、Skill Crossroads 原站取回、vLLM v0.33.0 稳定版身份等十余项上一轮未决/旧说法被一手证据解决或修正；同时集中纠正上一轮两处漏检（letta-code 0.34.40、langchain-core 1.7.5/1.7.6）。窗口内新发布以补丁与小版本为主：LiteLLM v1.108.1/v1.108.2 LTS、letta-code 0.34.41–0.34.44、LangGraph 1.2.20、Agent Framework dotnet-1.28.0、MetaHarness 0.4.21、Claude Code 2.1.319（有技能条目）、OpenCode v2.2.6、Codex rust-v0.167.0；机制性新内容集中在 OpenRouter 新一代 Auto Router、LiteLLM 1.108.x 路由器语义收紧、MEMOIR 记忆生命周期三处。长期未决的硬骨头仍在原地：CVE 独立厂商公告、Portkey 修复版、Railway 最终报告、OpenCode post-mortem、LiteLLM LTS“新端点”清单、AAIF 落地文件、Mem0 一句话官方解释、七项数字独立复现（连续三轮为零）。
- 文件变更：技能正本四篇 references 增量编辑（27 条技能增量：R1–R7、H1–H7、M1–M5、S1–S8；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.9.0/`；VERSION 0.9.0 → 0.10.0；CHANGELOG 新增 v0.10.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261107-1047/report.md + sources.md`；提炼与自验 `refined/20261107-1047/delta.md + verification.md`（执行者自验：通过 27 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 5 条一手来源全部吻合：letta-code v0.34.39 GitHub Releases API、Claude Code v2.1.318 GitHub Releases API、agentskills.io 规范页、metaharness dream-v0.4.19 GitHub Releases API、LiteLLM 官方 changelog）。
- 推翻的旧说法：vLLM“v0.32.0 仍为最后确认的稳定版”；Railway 初步报告更新戳“2026-10-23”（纠正为 2026-10-26T20:41:12Z）；letta-code“仍 0.34.39”与 PyPI 差距“仍 27 版”（拉大到 32 版）；上一轮对 letta-code 0.34.40 的漏检；上一轮对 langchain-core 1.7.5/1.7.6 的漏检；AutoGen“存档引入提交未找到”与“last push 15 April 2026”二手说；metaharness“版本号误记”推测；Claude 2.1.318“UTC 11-03 / CST 11-04”推断；SkillsBench“数字冲突未决”；UserPromptExpansion“首发版未验证”限定收窄；Windsurf 迁移向导“存在性仅二手”；Mobb 34%/27% 口径（dev.to 丢限定词）。
- 剩余未核实缺口汇总：CVE 独立厂商公告与 1.106.6 含修复确认；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；vLLM v0.32.0 页内日期戳；LiteLLM LTS“新端点”清单；vLLM v0.33.0 Latest 徽标与 release notes 正文；OpenRouter Routing Cost 官方公告原文；Conformal Cascade 疑似 v3；RouteBalance v2 修订日期；AutoGen 004749e 精确 UTC 时间戳；langchain-core 1.7.5/1.7.6 PyPI 上传日期；AgentTriggered 参数语义；METR 各仓逐仓戳；PyPI letta 改道原因；Mem0 双口径一句话官方解释；OSS 框架复现平台数；七项数字独立复现；Mem0 TS SDK 线；UserPromptExpansion 功能首发版本；mobb.ai 原文页 URL；Windsurf 迁移向导日期；Claude 2.1.319 版本号二手归属；Cursor CLI changelog 每条日期。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 10:50）承担，本任务不重复推送。

---

## 2026-11-07 轮次小结（下午轮，缺口复核 + 新增覆盖轮，版本 0.10.0 → 0.11.0；六路并行：路由两路、Harness 一路、记忆一路、技能两路，车道健康 5 条已验证可用）

- 跨轨结论：严格窗口内（2026-11-07 上午轮之后至 2026-11-07 17:37 核查时）四轨均未发现带日期的一手新稳定发布；本轮是上午轮的“缺口收敛续轮”——12 项长期缺口在本轮关闭（vLLM v0.32.0 日期一手、AutoGen 004749e 精确时间戳、langchain-core 1.7.5/1.7.6 PyPI 日期、AgentTriggered 参数语义源码解决、PyPI letta 改道原因、Mem0 results/ 张力、UserPromptExpansion 功能首发版 2.1.116、mobb.ai 原文页、Cursor 每条日期、Windsurf 迁移向导日期、MAF python-1.25.0 标签存在性、四篇论文版本页复核），同时集中纠正上午轮两处错误（MAF“python-1.25.0 标签不存在”、Inspect“最新仍 0.3.287”——实为 0.3.288，因 changelog 滞后漏检）与一处子代理主张（Dynamo v1.6.1“被 docs 表证伪”被驳回，张力仍未调和）。长期未决的硬骨头仍在原地：CVE 独立厂商公告、Portkey 修复版、Railway 最终报告、OpenCode post-mortem、LiteLLM LTS“新端点”清单、AAIF 落地文件、Mem0 一句话官方解释、七项数字独立复现（连续第四轮为零）。
- 文件变更：技能正本四篇 references 增量编辑（35 条技能增量：R1–R10、H1–H8、M1–M6、S1–S11；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.10.0/`；VERSION 0.10.0 → 0.11.0；CHANGELOG 新增 v0.11.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261107-1737/report.md + sources.md`；提炼与自验 `refined/20261107-1737/delta.md + verification.md`（执行者自验：通过 35 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 6 条一手来源全部吻合：vLLM v0.32.0 与 v0.33.0rc3 GitHub Releases API、AutoGen 004749e GitHub Commit API、Claude Code 2.1.319 GitHub Releases API、letta 2365e012 提交页、mobb.ai 原文页、UserPromptExpansion npm 时间字段；tarball 字符串计数未重跑，子代理方法已审阅并记入限制）。
- 推翻的旧说法：vLLM v0.32.0 日期“仍为多源二手”；v0.33.0“特征内容仅二手”；OpenRouter Routing Cost“公式仅二手”；Conformal Cascade“疑似 v3”；RouteBalance“v2 修订日期未决”；MAF“python-1.25.0 标签不存在”；Inspect“最新仍 0.3.287”；“UserPromptExpansion 功能首发版本未验证”；“mobb.ai 原文页 URL 未取得、文本未打开”；“Mem0 results/ 张力未决”；“四篇论文版本页未重开”；“Windsurf 迁移向导日期未决”；“Cursor CLI 每条日期未决”；“Claude 2.1.319 版本号二手归属”；Dynamo v1.6.1“被 docs 表证伪”（子代理主张被驳回）。
- 剩余未核实缺口汇总：CVE 独立厂商公告与 1.106.6 含修复确认；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；Mem0 双口径一句话官方解释；七项数字独立复现；AAIF 三件落地文件（治理文本已进展，文件未落地）；LiteLLM v1.110.0-dev.1 / v1.109.0-dev.1 published_at；vLLM Latest 徽标；Dynamo v1.6.1 调和；OpenRouter Routing Cost 实验数字与官方公告 URL；Conformal Cascade v3 修订内容；2611.04098 陪同论文；MAF 1.25.0 搁置原因；Inspect 0.3.288 PyPI 侧；OTel 后续；Agents SDK alpha 轨；AgentTriggered 实验性晋升；METR 其余仓库；Letta 服务器仓停滞（0.16.8，2026-05-14）；PyPI letta 下次发布时间；UserPromptExpansion 运行时测试；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版；skill-reach 仓库版本/日期；Cursor IDE 11 月发布一手复核。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 18:37）承担，本任务不重复推送。

---

## 2026-11-07 轮次小结（晚间轮，缺口复核 + 新增覆盖轮，版本 0.11.0 → 0.12.0；六路并行：路由两路、Harness 一路、记忆一路、技能两路，车道健康 5 条已验证可用）

- 跨轨结论：严格窗口内（2026-11-07 下午轮之后至 2026-11-07 23:37 核查时）唯一带日期的一手新稳定发布是 MAF Python 1.25.0 上架 PyPI（发生在本轮核查进行中，23:41–23:43 CST；GitHub Release 对象截至 23:43 CST 仍 404）。本轮是下午轮的“缺口收敛续轮”——12 项长期缺口在本轮关闭或升级（LiteLLM dev 线日期一手、vLLM Latest 徽标、CVE 1.106.6 源码比对证据、Portkey main 成因代码确认、Dynamo v1.6.2、OpenRouter 实验数字表、Conformal Cascade v3 全文、2611.04098 陪同论文、MAF 1.25.0 PyPI 上架、Inspect 0.3.288 PyPI 侧、METR 全 org 存档计数、Harbor 仓库迁移、Mem0 基准指南、Zep Agent Skills 补漏、letta 仓名补正、Claude 2.1.319 第四条、skill-reach 仓库身份、SWE-Skills-Bench v2、Cursor IDE 一手复核），同时集中纠正三处旧说法（OpenRouter“429 事件少 67%”官方正文无此数字、SWE-Skills-Bench v1 口径作废、MAF“无 PyPI”）。长期未决的硬骨头仍在原地：CVE 厂商逐字确认、Portkey 修复版、Railway 最终报告、OpenCode post-mortem、LiteLLM LTS“新端点”清单、AAIF 落地文件、Mem0 一句话官方解释、七项数字独立复现（连续第五轮为零）。三项张力未消解：H5 Learn 页张力、Dynamo 文档滞后、Mem0 TR 拼合链。
- 文件变更：技能正本四篇 references 增量编辑（38 条技能增量：R1–R11、H1–H8、M1–M7、S1–S11；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.11.0/`；VERSION 0.11.0 → 0.12.0；CHANGELOG 新增 v0.12.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261107-2337/report.md + sources.md`；提炼与自验 `refined/20261107-2337/delta.md + verification.md`（执行者自验：通过 38 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 11 条一手来源全部吻合：MAF PyPI 1.25.0、LiteLLM v1.110.0-dev.1 GitHub Releases API、Dynamo v1.6.2 GitHub Releases API、SWE-Skills-Bench v2 arXiv 摘要页、Learn 压紧页、Cursor changelog、skill-reach GitHub Repo API、OpenRouter 官方博客全文、Claude 2.1.319 官方 changelog L1516、CVE 修复提交 1d5fdc87 GitHub Commit API、Portkey main requestValidator 一手源码；可执行验证 0 条——本轮条目均为文档事实型）。
- 推翻的旧说法：LiteLLM v1.110.0-dev.1“published_at 未验证”；vLLM Latest 徽标“仍未确认”；OpenRouter“429 事件总数少 67%”；SWE-Skills-Bench v1 口径（49 技能/+1.2%）；MAF“python-1.25.0 无 PyPI / 不得写成已发布”；Cursor IDE 11 月发布“无技能内容结论只有二手”；Claude 2.1.319“仅三条技能条目”；Dynamo v1.6.1“被 docs 表证伪”（本轮 v1.6.2 进一步证实发布侧在前、文档侧滞后）；letta 2365e012 引证仓名未注明。
- 剩余未核实缺口汇总：CVE-2026-114612 厂商逐字确认 1.106.6；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；LiteLLM LTS“新端点”清单；Conformal Cascade v2→v3 diff；Mem0 双口径一句话官方解释；七项数字独立复现；AAIF 三件落地文件；MAF 1.25.0 其余 38 包与 Release 对象；AgentTriggered Learn 压紧页上线时间；OTel 后续；Agents SDK alpha 轨；PyPI letta 下次发布时间；UserPromptExpansion 运行时测试；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版；Codex alpha.7 tag 页补核；Skill Crossroads 下轮复核。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 00:37）承担，本任务不重复推送。

## 2026-11-08 轮次小结（晨轮，缺口复核 + 新增覆盖轮，版本 0.12.0 → 0.13.0；六路并行：路由两路、Harness 一路、记忆一路、技能两路，车道健康 7 条已验证可用）

- 窗口：2026-11-07 23:37 之后至 2026-11-08 约 06:10 CST。严格窗口内唯一带日期的一手新稳定发布是 MAF Python 1.25.0 完成发布（44/44 包全部上架 PyPI，2026-11-07T15:41:25–15:43:18Z；GitHub Release 对象补发于 15:52:42Z，89 附件；上一轮核查进行中上架、本轮一手定稿）。
- 四轨主题与进展：API 路由——OpenRouter 新一代 Auto Router + Auto Code 官方公告一手确认（2026-11-06，11 因子 + Jev Router），LiteLLM 仍无新稳定版（v1.108.2）、Dynamo kimi 实验预发布、GHSA patched_versions 补全、Conformal Cascade v2→v3 无实质文本差异（缺口关闭）、路由论文四篇 + 两篇新文、ParetoBandit 日期纠正、RouteLLM 无人维护；Agent Harness——MAF 1.25.0 完成发布（44/44 包 + Release 对象）、OTel「stripped_reason」纠正（被删提案字段）与 Trace 加法式增补、AgentTriggered 区间收窄（张力消解）、Agents SDK alpha 轨全文、METR hawk 唯一推送；分层记忆——Anthropic 记忆长文一手补漏（四层架构 + dreaming 五作用）、「用后反馈」独立排序层、压缩与保真度分开优化 + FlyHash 反例、Mem0 TR 博客全文重读、七项复现连续第六轮为零；技能——Sanko 监督松绑论文与 Kan ETUDE 论文新收、SkillSpector v2.13.4 定稿与「71/17」版本锚定、Codex alpha.7 一手证实、SkillzBench 快照刷新、AAIF 仍无落地文件（最近提交复核受 403 阻）。
- 调研来源：本轮一手约 95 项、二手约 45 项、厂商/论文自报数字组约 25 组、不可信来源 1 项已排除（Stackademic 声称 OpenCode 已发 post-mortem）。详见 `research/20261108-0537/sources.md`。六路子代理全部完成，无上游失败；唯一部分失败为 `getzep/zep-python` GitHub Releases API 403 rate limit（PyPI/npm 侧已确认，结论不受影响），待下轮重试；AAIF 最近提交复核受 GitHub API 403 阻，沿用上一轮已核实状态，下轮优先补核。
- 提炼增量：27 条（路由 R1–R10、Harness H1–H7、记忆 M1–M5、技能 S1–S5），见 `refined/20261108-0537/delta.md`。自验结论（执行者自验，见 `refined/20261108-0537/verification.md`）：通过 27 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 6 条一手来源全部吻合：MAF PyPI `agent-framework` 1.25.0、GHSA API patched_versions、OTel GenAI main 分支 `stripped_reason` 零命中、SkillSpector v2.13.4/v2.13.3 GitHub Releases API、LiteLLM v1.108.2 GitHub Releases API、`which claude` 无输出；可执行验证 1 条——`which claude` 真跑确认无实机环境；PRACTICAL 待修正与未验证暂定清单本轮开始即为空、无新增。
- 文件变更：技能正本四篇 references 增量编辑（27 条技能增量：R1–R10、H1–H7、M1–M5、S1–S5；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.12.0/`；VERSION 0.12.0 → 0.13.0；CHANGELOG 新增 v0.13.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261108-0537/report.md + sources.md`；提炼与自验 `refined/20261108-0537/delta.md + verification.md`（执行者自验：通过 27 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 6 条一手来源全部吻合：MAF PyPI 1.25.0、GHSA API patched_versions、OTel GenAI main 分支 `stripped_reason` 零命中、SkillSpector v2.13.4/v2.13.3 GitHub Releases API、LiteLLM v1.108.2 GitHub Releases API、`which claude` 无输出；可执行验证 1 条——`which claude` 真跑确认无实机环境）。
- 推翻的旧说法：GHSA API「first_patched_version 全 null」（元数据已补全）；Conformal Cascade「v2→v3 diff 未决」（已关闭为无实质文本差异）；OpenRouter Auto「仍按旧机制排序」（11-06 重建版已取代）；ParetoBandit「11 月新出的 bandit 路由论文」（实为 3–4 月旧文）；RouteLLM「活跃的开源路由框架」（无人维护）；「OpenCode 已发布详细 post-mortem」（Stackademic 文章不可信）。
- 剩余未核实缺口汇总：CVE-2026-114612 厂商逐字确认 1.106.6；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；LiteLLM LTS“新端点”清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）；Dynamo 文档滞后、v1.7.0-kimi 线演进；Route Anything「8.82×效率」定义、Local or Escalate 内部基准未公开；MAF 1.25.0 tag→PyPI 间隔原因、AgentTriggered 精确上线日、OTel aabmass 提案、gen_ai.workflow.name 是否入规范、Agents SDK alpha.7/稳定化时间表、METR hawk 具体提交日期；Mem0 双口径一句话官方解释、七项复现（连续六轮为零）、PyPI letta 下次发布、Zep Agent Skills 论文是否有 v2、getzep/zep-python releases 侧复核（本轮 403）、Anthropic CL-Bench 均值数字与 Dreams API 细节；UserPromptExpansion 运行时测试、Gemini v0.68.0/Codex 0.168.0/0.169.0 稳定版、AAIF 落地文件（最近提交复核下轮优先补核）、Skill Crossroads 首行版本/日期、SkillzBench 计数波动解释、Windsurf 官方桌面 changelog 一手复核、Kan 等七框架完整名单与 Sanko 论文细节（下轮读正文）、Liu 等 arXiv:2605.14460 原文核实。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 06:37）承担，本任务不重复推送。

## 2026-11-08 轮次小结（中午轮，缺口复核 + 新增覆盖轮，版本 0.13.0 → 0.14.0；六路并行：路由两路、Harness 一路、记忆一路、技能两路，车道健康 6 条已验证可用）

- 窗口：2026-11-08 约 06:10 CST → 2026-11-08 11:37 CST。严格窗口内唯一带日期的一手新稳定发布是 LiteLLM v1.108.3（published 2026-11-07T22:29:58Z，PyPI 上传 2026-11-07T22:25:38Z；v1.109 仍仅 rc.1；预发布 v1.110.0-dev.2 窗口内发布）。
- 四轨主题与进展：API 路由——LiteLLM v1.108.3 新稳定发布、Route Anything 8.82×定义闭合、Local or Escalate 基准构成公开与「11-03 修订」纠正、Portkey v1.15.2 Docker Hub 佐证、复核戳推进（CVE-2026-114612、Railway/OpenCode、vLLM、OpenRouter、Dynamo）；Agent Harness——METR hawk 220a425 新提交与 #2820 日期纠正、gen_ai.workflow.name 已移除折叠进 gen_ai.agent.name、MAF 1.25.0 间隔机制证据、aabmass 仅 fork 分支提交、Agents SDK alpha.7 未出现；分层记忆——Anthropic CL-Bench 均值数字与 Dreams API 细节升级为一手、Zep 63.8% 纠正为厂商自报、zep-python 403 闭环、DolphinBench/Maximem 补充登记、Route to Remember 第三条线、七项复现连续第七轮为零；技能——ETUDE 七框架完整名单一手解决、Sanko & Tian 全部锚定、Liu 等 SCH 无载荷语义攻击新收、SkillzBench 计数刷新、AAIF 最近提交复核缺口关闭、轮间补漏论文五篇。
- 调研来源：本轮约 90 个来源，一手约 70（详见 `research/20261108-1137/sources.md`）。六路子代理全部完成，无上游失败，无未完成子题。
- 提炼增量：25 条（路由 R1–R5、Harness H1–H5、记忆 M1–M6、技能 S1–S6、S8），见 `refined/20261108-1137/delta.md`；另 S9 为 track-only、R6（Dynamo kimi tag 张力）列入「冲突/未决」不整合。自验结论（执行者自验，见 `refined/20261108-1137/verification.md`）：通过 25 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 8 条一手来源全部吻合：PyPI litellm JSON（v1.108.3）、PyPI vllm JSON（最新 0.33.0）、OTel registry.yaml（`gen_ai.agent.name` L1001、`workflow.name` 零命中）、Anthropic 官方文章页 HTML（六组 CL-Bench 数字）、arXiv:2512.13564（综述）、AAIF commits API（#648/#649）、Route Anything 全文（8.82×）、Local or Escalate v2 页 404；可执行验证 1 条——`which claude` 真跑确认无实机环境；PRACTICAL 待修正与未验证暂定清单本轮开始即为空、无新增。
- 文件变更：技能正本四篇 references 增量编辑（25 条技能增量：R1–R5、H1–H5、M1–M6、S1–S6、S8；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.13.0/`；VERSION 0.13.0 → 0.14.0；CHANGELOG 新增 v0.14.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261108-1137/report.md + sources.md`；提炼与自验 `refined/20261108-1137/delta.md + verification.md`（执行者自验：通过 25 条、通过（未验证暂定）0 条、需修正 0 条、否决 0 条；独立抽查 8 条一手来源全部吻合；可执行验证 1 条——`which claude` 真跑确认无实机环境）。
- 推翻的旧说法：Local or Escalate「11-03 更新」（v1 单版本）；「Zep 63.8% 是独立复现」（厂商自报）；`gen_ai.workflow.name`「待确认 discrepancy」（已移除，折叠进 `gen_ai.agent.name`）；MAF 1.25.0 间隔「纯推断」（升级为带机制证据的推断）。
- 剩余未核实缺口汇总：CVE-2026-114612 厂商逐字确认；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；LiteLLM LTS“新端点”清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）；Dynamo v1.7.0-kimi tag 张力（releases-data 表无对应行 vs GitHub release 页口径，下轮再核对出处）；MAF 1.25.0 tag→PyPI 间隔官方解释、AgentTriggered 精确上线日、OTel aabmass 提案、Agents SDK alpha.7/稳定化时间表、METR hawk 后续提交；Mem0 双口径一句话官方解释、七项复现（连续七轮为零）、PyPI letta 下次发布、DolphinBench Empirical PR 审定结果、Route to Remember 全文；UserPromptExpansion 运行时测试、Gemini v0.68.0/Codex 0.168.0/0.169.0 稳定版、AAIF 落地文件、Skill Crossroads 首行版本/日期、SkillzBench 计数波动解释、Windsurf 官方桌面 changelog 一手复核（本轮受阻，下轮优先重核）、Kan 等 97.6%/95.9% 聚合口径、Liu 等 arXiv:2605.14460 全文核实。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 12:37）承担，本任务不重复推送。

## 2026-11-08 轮次小结（晚间轮，缺口复核 + 新增覆盖轮，版本 0.14.0 → 0.15.0；六路并行：路由两路、Harness 一路、记忆一路、技能两路，车道健康 7 条已验证可用）

- 窗口：2026-11-08 11:37 CST → 2026-11-08 17:37 CST。严格窗口内带日期的一手新发布两件：Agents SDK Python v0.24.0-alpha.7 与 JS v0.22.0-alpha.2（均 GitHub-only，无稳定化时间表）；letta-code v0.34.45（npm 2026-11-08T07:44:31Z）。
- 四轨主题与进展：API 路由——Route Anything「认证路线均值约 7.2×」口径纠正（混合口径撤回，两床纯认证均值 13.05×）、Portkey CVE 消歧（Geekflare「5 月修复」针对 /log/stream，非 SSRF）、OpenRouter 基准表与十月数据（未验证暂定 R3）、AgentRouter v1.2 + Python SDK、Dynamo 调和证据、复核戳推进（LiteLLM/vLLM/CVE/Railway/OpenCode/路由论文）；Agent Harness——Agents SDK alpha.7/alpha.2 窗口内发布、aabmass PR #688 存在且 Open（推翻中午轮结论）、langgraph-sdk 两条版本线、Google ADK 2.14.0（未验证暂定 H4）、版本戳推进；分层记忆——letta-code v0.34.45、Route to Remember 全文数字、补漏三篇论文、七项复现连续第八轮为零；技能——Windsurf v3.12.28 官方确认（矛盾消解）、Liu 三数字精确锚定、SkillzBench 快照刷新、AAIF #648/#649 一手确认、宿主版本戳推进。
- 调研来源：本轮 60 个来源，一手 47、二手/厂商自报 13（详见 `research/20261108-1737/sources.md`）。六路子代理全部完成，无上游失败，无未完成子题；GitHub API 本轮全部 403 限流，结论改以 PyPI/npm/git 协议/页面文本一手核对。
- 提炼增量：21 条（路由 R1–R8、Harness H1–H5、记忆 M1–M4、技能 S1–S4），见 `refined/20261108-1737/delta.md`；另有「仅记 track、不入技能正文」7 项（ETUDE 聚合口径、Cursor Agent SDK、HF harness-hub、Skill Comply/Evaluator、OpenCode #60315、Windsurf v3.12.17、GitHub 403 限流说明）。自验结论（执行者自验，见 `refined/20261108-1737/verification.md`）：通过 19 条、通过（未验证暂定）2 条（R3、H4）、需修正 0 条、否决 0 条；独立抽查 9 项一手来源全部吻合：Route Anything 全文 Table 5、OTel PR #688 页、Windsurf 官方 changelog、npm letta-code、git ls-remote Agents SDK 标签、Liu 论文全文、PyPI litellm/vllm JSON、Geekflare Portkey 评测全文、PyPI google-adk 与 npm portkey gateway；可执行验证：`which claude` 真跑确认无实机环境 + 7 次一手注册表/协议查询全部成功；PRACTICAL 待修正清单本轮开始即为空、无新增，新增未验证暂定 2 条已登记。
- 文件变更：技能正本四篇 references 增量编辑（21 条技能增量：R1–R8、H1–H5、M1–M4、S1–S4；SKILL.md 本体未改）；整合前旧版整目录快照于 `versions/v0.14.0/`；VERSION 0.14.0 → 0.15.0；CHANGELOG 新增 v0.15.0 节；STATE.md 新增本轮覆盖节与指针推进；四轨 track 笔记各追加本轮日期小节；本轮调研产出 `research/20261108-1737/report.md + sources.md`；提炼与自验 `refined/20261108-1737/delta.md + verification.md`（执行者自验：通过 19 条、通过（未验证暂定）2 条、需修正 0 条、否决 0 条；独立抽查 9 项一手来源全部吻合；可执行验证：`which claude` + 7 次一手查询）。
- 推翻的旧说法：Route Anything「认证路线均值约 7.2×」（混合口径，撤回）；OTel aabmass「仅 fork 三提交、无独立 PR」（PR #688 存在且 Open，分支实为两条提交）；Devin Desktop「仍 v3.12.17」（官方最新稳定版为 v3.12.28，2026-11-07）；Liu「最脆弱配置」单数表述（两峰值属不同配置）；Geekflare「5 月修复」与 CVE-2026-110719 的混同可能（针对 /log/stream，非 SSRF）。
- 剩余未核实缺口汇总：CVE-2026-114612 厂商逐字确认；Portkey CVE-2026-110719 修复版；Railway 最终报告；OpenCode post-mortem；LiteLLM LTS“新端点”清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）；Dynamo kimi tag 本身直接再验；OpenRouter 基准页直读复核（R3）；ADK 官方 release notes 一手复核（H4）；MAF 1.25.0 tag→PyPI 间隔官方解释、AgentTriggered 精确上线日、OTel PR #688 审定结果、Agents SDK alpha.7 后续与稳定化时间表、METR hawk 后续提交；Mem0 双口径一句话官方解释、七项复现（连续八轮为零）、PyPI letta 下次发布、DolphinBench Empirical PR 审定结果；UserPromptExpansion 运行时测试、Gemini v0.68.0/Codex 0.168.0/0.169.0 稳定版、AAIF 落地文件、Skill Crossroads 首行版本/日期、ETUDE 97.6%/95.9% 聚合口径（只查 v2）、Cursor Agent SDK 一手确认、OpenCode #60315 归属版本、Windsurf v3.12.17 是否重现。以上均未否决任何条目，或排除在正文之外、或以二手/自报标注保留在线索中。
- SKILL.md / SOUL.md 级提案：无新增（落点纪律：SKILL.md 本体不自动改）。
- 发表同步：由 agent-research-evolution-publish 任务（约 18:37）承担，本任务不重复推送。

## 第 17 轮（2026-11-09 晨轮，窗口 2026-11-08 23:37 → 2026-11-09 05:37；版本 0.16.0 → 0.17.0）

- **执行者误报纠正**：本轮 cron 执行者曾报告「路由 B 路与技能 B 路未回齐、本轮失败」；主会话磁盘核验确认六路均已回齐（research/20261109-0537/ 六腿文件共 1,038 行，去重来源 144 个），误报系快照时点早于最后一路回齐约 24 秒；整合由主会话直接完成，delta.md（22 条增量）与 verification.md 均已落盘。
- 跨轨结论：窗口内无新发布，本轮为缺口复核轮——上一轮 2 项暂定全部关闭（R7 三家定价悬崖阈值、R3 llm-d published_at），新增 2 项暂定子项（R9 v0.10.0 日期、S4 `@grok/agent-sdk` 0.6.0 时间戳）；三项重大纠正（Dynamo v1.6.2 存在性裁定驳回路由 A 路结论、六篇记忆论文编号全部纠正、arXiv:2611.04456 任务书勘误）；自验 22 条：通过 19、通过（未验证暂定）2、需修正 1（R6，已修正）、否决 0（执行者自验，独立抽查 6 项一手来源全部吻合或裁定有据）。
- 文件变更：skill/references/ 四篇增量编辑（api-routing.md 8 处、agent-harness.md 1 处、layered-memory.md 3 处、skill-authoring.md 7 处，共 19 处替换，全部断言锚点唯一）；VERSION 0.16.0 → 0.17.0；旧版快照 versions/v0.16.0/；CHANGELOG.md v0.17.0 节、STATE.md 版本与轮次节、PRACTICAL.md（2 项暂定转已修正 + 任务书勘误登记已修正 + 2 项新暂定子项）、四轨 track 笔记各追加轮次节。
- 剩余未核实缺口汇总：CVE-2026-114612 厂商逐字命名、Portkey CVE 修复版本、Railway 终报、OpenCode post-mortem、Dynamo v1.7.0-kimi 线、LiteLLM LTS 端点清单、llm-d v0.6 日期、ETUDE v2、Mem0 双口径官方解释、七项复现（第十轮为零）、arXiv 周一 2026-11-09 批次补填、The Memory Wall 与 WALDEN 定位、UserPromptExpansion 实机测试。

## 第 18 轮（2026-11-09 中午轮，窗口 2026-11-09 05:37 → 2026-11-09 11:37；版本 0.17.0 → 0.18.0）

- 六路并行全部回齐（路由 A/B、Harness、记忆、技能 A/B；research/20261109-1137/legs/ 六腿文件，去重来源约 60 个一手 + 约 30 个二手 + 1 个存疑未采信；arXiv API 三路限流/超时，方法限制已记录）。
- 跨轨结论：窗口内新稳定发布两件（letta-code v0.34.46、Windsurf v3.12.32）；上一轮 2 项暂定全部一手闭合（R9 llm-d 日期、S4 grok 时间戳），新增 3 项暂定（R6 Cloudflare 细节、R7 OpenRouter 十月数据、R8 子项 DeepSeek/Kimi）；本轮最重的新证据是 Mem0 的两条公开第三方实测（Atlaso 44.2%、arXiv:2603.04814 49.0%），七项复现表述随之修正；五项腿内结论被执行者纠正（Haiku 悬崖重复、OpenAI 整单重定价与一手原文冲突、Inspect/Harbor slug 为派发书背景错误、ServerBench 仅二手、Anthropic 直链未取得不降级既有一手记录）；arXiv 周一 11-09 批次截至 11:45 CST 尚未发布（三路列表页一致，不得作「无论文」定论）。
- 文件变更：skill/references/ 四篇增量编辑（api-routing.md +3 行、agent-harness.md 0 行、layered-memory.md 0 行、skill-authoring.md 0 行，均为行内追加与新条目，共 25 条增量）；VERSION 0.17.0 → 0.18.0；旧版快照 versions/v0.17.0/；CHANGELOG.md v0.18.0 节、STATE.md 中午轮小节、PRACTICAL.md（R9/S4 划除记入已修正，R6/R7/R8 子项新登记）、四轨 track-*.md 各追加本轮小节；research/20261109-1137/report.md + sources.md、refined/20261109-1137/delta.md + verification.md。
- 剩余未核实缺口汇总：CVE-2026-114612 厂商逐字命名、Portkey CVE 修复版本、Railway 终报、OpenCode post-mortem、LiteLLM LTS 端点清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）、arXiv 周一批次枚举、真实流量级联校准、MAF 间隔官方解释、AgentTriggered 精确上线日期（已冻结）、OTel PR #688、The Memory Wall / WALDEN 未定位、Mem0 双口径官方解释、Zep 90.2% 官方原文、ETUDE 聚合口径（读 v1 方法节）、Windsurf v3.12.28 是否重现、UserPromptExpansion 实机测试（无本地 Claude 环境）。
