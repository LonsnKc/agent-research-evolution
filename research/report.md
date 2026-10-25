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

