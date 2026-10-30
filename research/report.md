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
