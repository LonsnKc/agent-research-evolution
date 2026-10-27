# 轨道：Agent Harness

基线来源：`~/workspace/research_notes/api-routing-harness-memory-skills-20261023-1442/report.md`（2026-10-23）与技能参考 `~/workspace/custom-skills/agent_system_design/references/agent-harness.md`。

基线要点（2026-10-23）：
- 定义：模型之外全部确定性软件与约定（上下文组装、工具循环、状态与检查点、记忆访问、沙箱与权限、护栏、人工介入、trace 与评估、恢复与重试）。
- 权威模式：Anthropic《Building Effective Agents》（2024-12-19，五种工作流模式）、《Effective Context Engineering》（2025）、《Effective Harnesses for Long Running Agents》（2025，本轮仅经二手报告阅读）；OpenAI《A Practical Guide to Building Agents》（2025，经摘要阅读）。
- 框架对比：LangGraph、OpenAI Agents SDK、Claude Agent SDK、SWE-agent（Agent Computer Interface）、OpenHands、smolagents、CrewAI、AutoGen（后续状态待向官方复核）、DSPy；评估 harness：BrowserGym/AgentLab、Harbor/Terminal-Bench、METR Task Standard/Vivaria。
- MCP：JSON-RPC 2.0 协议，工具/资源/提示三类原语；与技能互补。2025 年底捐赠给 Linux Foundation 下属机构一事本轮未向官方公告核实。
- 失败模式：上下文腐化、单会话贪多无交接产物、未跑验证即宣告完成、自评通过但测试拒绝、工具定义重叠、并行写同一状态、无上限循环、对照变量混杂的比较。

---

## 2026-10-25 轮次：窗口内无新发布；补录四项基线缺口；核实长程 harness 原文；深化 eval 与 trace 格式

窗口结论：严格窗口 2026-10-24→25 内无日期落入其中的一手发布。核查证据（已核实）：Claude Code 官方 changelog（https://code.claude.com/docs/en/changelog）最新条目为 2026-10-23（Monitor 工具、云会话、沙箱 AppArmor 提示等），页内无 2026-10-24、2026-10-25 条目。OpenAI Agents SDK、LangGraph、Claude Agent SDK 的官方 changelog/releases 页本轮搜索亦未返回窗口内条目——仅为“本轮公开搜索未发现”，不等于没有更新，下轮复查各仓库 releases 页。

### 补录项（水位之前、基线未覆盖）

- **Anthropic《Effective harnesses for long-running agents》一手原文已核实**（https://www.anthropic.com/research/effective-harnesses-for-long-running-agents，Published 2025-11-26，Updated 2026-10-18）：基线“仅经二手报告阅读”的限定可删除。核心确为两件套：initializer agent（首次运行建环境：`init.sh`、claude-progress.txt、初始 git 提交、超 200 项 feature 列表，初始状态全标 failing）+ coding agent（每会话只做一个 feature、提交、更新进度文件、留可合并的干净状态）；feature 列表用 JSON 而非 Markdown（模型更不易误改）；显式要求用浏览器自动化像真人用户一样做端到端测试，否则模型会改了代码、自测通过就宣布完成。
- **2025-10-22 LangChain 1.0 / LangGraph 1.0 同步 GA**（未核实，多家二手一致，未见官方公告原文）：承诺到 2.0 前无破坏性变更；新前门 `create_agent` 内部即构建 LangGraph StateGraph；新增 SummarizationMiddleware、HumanInTheLoopMiddleware、PIIMiddleware；AgentExecutor 进入维护模式（维护至 2026-12 的说法需官方复核）。
- **2025-10-01 / 10-02 Microsoft Agent Framework 发布，AutoGen 进入维护**（未核实，二手）：微软 2025-10-01 宣布 Agent Framework（AutoGen + Semantic Kernel 合并）；AutoGen README 于 2025-10-02 挂维护横幅（不再加新功能、社区维护）。二手说法称 Agent Framework 于 2026-04 达 1.0（.NET + Python，MIT），范围含单 agent 抽象、中间件、记忆/上下文 provider、图工作流、多 agent 编排（含 Magentic-One）及多家模型连接器。基线“AutoGen 后继状态待向官方复核”的待办可关闭方向，但写入 skill 前仍建议以微软官方公告复核一次。
- **2025-10-06 OpenAI AgentKit 发布**（未核实，二手）：DevDay 2025-10-06，Agent Builder 可视化多 agent 工作流画布（beta，基于 Responses API）+ ChatKit + Connector Registry + Evals（数据集、trace grading、自动 prompt 优化）。另有一份 2026 年二手“弃用指南”称 Agent Builder 与托管 Evals 正在被逐步停用（未见 OpenAI 官方通知原文，仅作风险线索，切勿直接写入 skill 定论）。
- **2026-07-23 TERMINAL-BENCH 3.0 上线，Harbor-native**（部分核实：Scale 官方博客一手，数字为厂商自报）：约 16 类任务、easy / medium / hard 三档、任务准入流程（自动化检查 → 全程人工评审 → 失败模式分析，只保留“真正能力缺口导致失败”的任务）。“最优前沿模型得分低于 40%”、“TB 2.0 已被前沿模型做到约一半”等均为 Scale 自家博客说法，未核实。
- **2026-10-15 左右 Harbor × Docker Sandboxes 集成**（未核实，Docker 官方博客摘要）：Harbor 可跑在 Docker Cloud Sandboxes（MicroVM）上，支持 Docker Compose 多容器任务、public / no-network 模式；nightly 版 `harbor[sbx]==0.24.1.dev202610150414`，称稳定版将于 10/23 发布；“64 并发任务实测”系厂商说法，未核实。

### 深化：评估 harness 与 trace 格式

- **任务格式已收敛为同一骨架**：指令 + 隔离环境定义 + 机器可执行验证器 + 可选参考解。Harbor（Terminal-Bench 官方 runner）：每任务一目录（`task.toml` 元数据/超时/网络策略/资源约束 + `instruction.md` + `environment/` Dockerfile 或 docker-compose + `tests/test.sh` + 可选 `solution/solve.sh` Oracle 基线；支持 `steps/` 多步任务）。METR Task Standard / Vivaria：任务 = 2～7 个 Python 函数（环境构建、指令、可选自动评分代码），按 family 成组。差别主要在环境定义语言与组织单位，而非理念分歧。Harbor/METR 细节本轮为二手/摘要级（DeepWiki 汇编、官方站摘要），未逐页核对官方 docs 原文，标未核实。
- **隔离模型**：Harbor 每任务一容器，可横向扩展到云端（Daytona、Modal、LangSmith、Blaxel 等）；Vivaria agent 在 Docker 容器内可跑任意代码，靠 Docker 隔离宿主；Task Standard 允许声明“无互联网”任务（aux VM 方案），但官方对照页自述该无网 aux VM 代码未经测试（摘要级，未核实）——不要把“支持声明”当成“经过验证的支持”。
- **评分方式**：结果态评分为主流——Harbor 的 `test.sh` 写 `reward.txt`（1.0/0.0）或 `reward.json`（数值指标）；TB 3.0 明确“忽略 agent 自己的说法，只查终态”。参考解基线：Harbor 的 OracleAgent 跑 `solution/solve.sh`——无 oracle 通过率的 benchmark 无法区分“任务不可解”与“agent 不行”。TB 榜单要求 5 次运行提交为二手说法，未核实；单次通过率不可直接当模型成绩引用。
- **测量装置即变量（新增失败模式，数字为二手转述、未核实）**：同模型同 harness 同任务仅改资源上限，TB 2.0 分数摆动 6 分，基础设施失败率从 5.8% 降至 0.5%；某 CORE-Bench 内部运行修 grader + 放宽 scaffold 后同模型从 42% 升至 95%。选型推论：引用任何 agent 分数必须同时锁定五项——模型、harness（含版本）、任务集版本、资源上限、运行次数；缺一项即不可比。
- **Trace / span 格式**：事实标准候选是 OpenTelemetry GenAI 语义约定（多家二手指南一致，未见 OTel 官方 spec 原文，标未核实）：span 树 `invoke_agent <agent>` → `chat <model>` → `execute_tool <tool>`，子 agent 为嵌套 `invoke_agent`；核心属性 `gen_ai.operation.name`、`gen_ai.agent.name`、`gen_ai.conversation.id`、`gen_ai.provider.name`、`gen_ai.request.model` / `gen_ai.response.model`、`gen_ai.usage.input_tokens` / `output_tokens`、`gen_ai.response.finish_reasons`、`gen_ai.tool.name` / `tool.call.id`；内容属性（input/output messages、tool arguments/result、system_instructions）敏感且体积大，须 opt-in、脱敏、截断；稳定性为 Development（非 Stable），2026 年中已从主语义约定仓库移入独立仓库，属性名仍可能变更——实现时必须把属性名集中到一个映射文件。MCP 语义约定（有二手说法称引入于 v1.396，未核实）：MCP span 与 GenAI span 去重（外层已有 `execute_tool` 时只 enrich 不重复建 span），属性含 `mcp.method.name`、`mcp.session.id`、`mcp.protocol.version`。分裂仍在：LangSmith 自有命名空间（如 `langsmith.span.kind`、`langgraph.graph.id`）与 `gen_ai.*` 不互通；内容记录默认策略分裂（默认只记元数据 vs 默认记完整消息）；评估用 trace 无统一 schema（trace grading、数据集格式、verifier 输出各家自定）。厂商侧：2026-10-23 官方 changelog（已核实打开）显示 `/claude-api` skill 新增“API tracing + SDK tracing”选项——Anthropic 正把 OTEL 作为一等导出路径，但细节仍以官方 docs 为准。
- **选型建议**：自建 benchmark 优先 Harbor 格式（TB 生态最大、云端 runner 最多、oracle 基线现成），但必须同时记录资源上限与运行次数；危险能力/受控评估优先 METR Task Standard（函数式定义 + family 组织 + 评分代码内建），但无网 aux VM 一节须先实测再承诺；Trace 统一写 OTel GenAI 约定，但做三层防护（属性名集中映射；内容属性默认关闭、调试时显式开启并脱敏截断；入库时同时保留厂商原始 span）；凡文档标注 untested / beta / preview 的能力一律保留限定词，待实测或官方通知后再升级措辞。

### 已同步进 skill 参考

- agent-harness.md：长程 harness 条目升级为一手原文（删“仅二手”限定）；AutoGen / LangGraph / AgentKit 行增补（均标二手、待官方复核，符合该文件“二手快照待复核”的既有惯例）；新增“Eval harness task formats”与“Trace / span formats”两节（均标二手/摘要级）；失败模式增补两条（测量装置即变量、把“支持”当“验证”）；MCP 一节增补 OTel MCP 去重一句（标未核实）。无 SKILL.md 级变更。

### 剩余未核实缺口（下轮复核清单）

1. LangChain/LangGraph 1.0 官方公告原文与 AgentExecutor 维护截止日期。
2. 微软 Agent Framework 官方公告原文与 1.0 范围；AutoGen 维护横幅原文。
3. OpenAI AgentKit 官方公告原文；Agent Builder / 托管 Evals 弃用传闻的官方通知（有则改写选型建议，无则删除该线索）。
4. Harbor 官方 docs 原文（task.toml schema、steps 语义、reward.json schema）；METR Task Standard 官方 spec 原文（函数签名与 family 语义）。
5. OTel GenAI 官方 spec 原文（操作名、属性表、稳定性状态、MCP 约定版本）。
6. Anthropic 2026 年 harness/evals 后续文章（Demystifying evals、Managed Agents、Harness Design for Long-Running Application Development）一手原文。
7. 下轮必查各仓库 releases 页在 2026-10-25 之后是否有更新。


---

## 2026-10-27 轮次：缺口复核——LangChain 1.0、Agent Framework 1.0、AutoGen 存档、AgentKit 官方停用全部一手确认

### 已核实（一手）

- **LangChain/LangGraph 1.0**：官方博客（2025-10-22）同步 GA、无破坏性变更承诺；`create_agent` 为前门、`create_react_agent` 已弃用；预置 middleware 为 PII 遮蔽、会话摘要、人工审批。官方原文无 AgentExecutor「维护至 2026-12」表述，该日期仍为二手，不写入技能。
- **Agent Framework**：2025-10-01 发布；1.0 于 2026-04-03 发布（.NET + Python，MIT，含 Magentic-One 与 YAML 声明式 agent）；1.0 时 A2A 尚为 coming soon；Go SDK 在独立仓库。
- **AutoGen**：当前官方 README 顶部横幅为 archived and no longer maintained；同页残留 maintenance mode 旧表述以横幅为准；存档日期与 2025-10 横幅逐字原文在当前页已不可见，不写死。
- **AgentKit 官方停用（推翻上一轮结论）**：OpenAI 官方 deprecations 文档（自验者亲自打开核对）——2026-06-03 宣布弃用 Agent Builder 与 Evals 平台；2026-10-31 起既有 evals 只读；2026-11-30 起 Agent Builder、Evals dashboard 与 API、可复用 prompt 停用；ChatKit 仍可用；迁移方向为 Agents SDK、ChatGPT Workspace Agents、Promptfoo；未见 Connector Registry 停用条目。
- **Harbor 现行格式**：官方 docs（2026-10-19 更新）——格式版本默认 "1.3"、verifier 超时默认 600 秒、agent 超时默认 null、多步任务 `[[steps]]` + `min_reward` 早停。**METR**：2024 年官方博客示例 `standard_version = "0.1.0"` 为旧版口径，不得当作现行版引用。

### 已同步进 skill 参考

- agent-harness.md：LangGraph、AutoGen、AgentKit 三条目由二手改写为一手（含 AgentKit 停用推翻）；Eval harness 一节追加 Harbor 现行格式纠正与 METR 旧版警示；旧二手表述（AgentExecutor 维护、AgentKit「未见官方通知」）已删除。

### 剩余未核实缺口（更新）

1. AgentExecutor 维护截止日期（官方原文无此表述，仍为二手）。
2. AutoGen 存档具体日期与 2025-10 横幅逐字原文（当前页已不可见）。
3. OTel GenAI 官方 spec 原文（仍未逐页核对）。
4. Anthropic 2026 年 harness/evals 后续文章一手原文（仍未读）。
5. 已解决：LangChain/LangGraph 1.0、Agent Framework 1.0、AgentKit 官方公告与停用通知、Harbor 官方 docs。
6. 下轮必查各仓库 releases 页在 2026-10-27 之后是否有更新。
