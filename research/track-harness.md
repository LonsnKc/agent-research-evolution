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


---

## 2026-10-28 轮次：缺口复核——OTel 官方 spec、AgentExecutor 包级口径、Anthropic 2026 文章

### 已核实（一手为主）

- **OTel GenAI 语义约定官方 spec**：仓库 `open-telemetry/semantic-conventions-genai`（main 分支，2026-10-28；创建于 2026-05-05，Releases 0、Tags 0，无正式版本号）；总纲、Agent spans、MCP 约定页均为 Status: Development；Agent 操作名、create_agent span 命名与 kind、system_instructions Opt-In、MCP span 命名与 `params._meta` 传播、已有外层工具 span 时不另建 span。无单一"MCP 约定版本"（`mcp.protocol.version` 仅为 Recommended 属性，`2025-06-18` 仅为示例值）。
- **Anthropic 2026 年文章**：Demystifying evals（eval harness vs agent harness 区分）；Managed Agents（2026-04-08 发布、2026-10-18 更新；session / harness / sandbox 解耦，execute / wake / getSession；TTFT 数字厂商自报）；长程应用 harness（Planner / Generator / Evaluator，Playwright MCP 评分，5–15 次迭代/代、最长 4 小时，Opus 4.5 后去掉 context reset）。Demystifying 与长程文章的发布日期未在页面片段中出现，不写日期。
- **AgentExecutor 维护口径纠正**：LangChain 官方 `langchain-classic` 参考页称该包"will be maintained for security vulnerabilities until December 2026"——包级口径，非 AgentExecutor 单独表述，无 12 月精确日期。上一轮"官方原文无此表述"被推翻。
- **AutoGen 存档措辞**：当前 README 逐字"Project archived and no longer maintained"；同一页仓库元数据"Archived: No"与正文矛盾，不得凭元数据断定存档开关；存档日期与 2025-10 横幅逐字原文仍不可见。
- **窗口内新动态**：Claude Code 2026-10-27 changelog 条目（见技能轨）；2026-10-28 当天其余各家检索未见新条目；OpenAI Agents SDK Python v0.24.0 为 2026-10-15（窗口之前）。

### 已同步进 skill 参考

- agent-harness.md：Trace 一节由二手指南升级为官方 spec 口径；MCP 一节删除 1.396 版本说法；LangGraph 行 AgentExecutor 口径改为包级表述；新增 Anthropic 2026 三篇文章；AutoGen 行追加逐字措辞与元数据矛盾警示。

### 剩余未核实缺口（更新）

1. AgentExecutor 精确日期（不存在精确日期，只写包级口径）。
2. AutoGen 存档具体日期与 2025-10 横幅逐字原文（当前页已不可见）。
3. Anthropic 折叠 subagent 文章（2026-10-22）一手原文（未打开；2 倍数字厂商自报）。
4. OTel MCP 约定版本号（不存在单一版本，已排除）。
5. 已解决：OTel 官方 spec 原文、AgentExecutor 包级口径、Anthropic 2026 三篇文章。
6. 下轮必查各仓库 releases 页在 2026-10-28 之后是否有更新。


---

## 2026-10-29 轮次：缺口复核——Anthropic 折叠文章、AutoGen 残留、Agent Framework 1.23.0、Harbor 迁址

### 已核实（一手为主）

- **Anthropic 折叠 subagent 文章一手原文**：标题《Building agents: When to fold subagents into your main agent loop》，发布 2026-10-22（页面尾部 Published 字段）。核心主张与保留三条件、单循环设计（一个模型、一个上下文窗口、轮次与成本预算、写门禁、首轮写计划）、渐进披露（技能与工具先看简短描述）、迁移步骤（先建端到端 eval，10 个 golden 示例起步，一次只折叠一个并对比任务翻转；样本破坏案例两次运行仅出现一次，故每任务须跑多次）、工具定义 30–50 个退化阈值（厂商经验表述，未独立核实）。厂商自报数字（未独立核实）：单客户单循环约快 2 倍；Claude Code 系统提示词削减超 80%。
- **配套迁移套件 single-agent-loop-kit**：仓库 README 自我定位 “Reference implementation. Not maintained and not accepting contributions.”；当前提交 c437962，创建 2026-10-15，Apache-2.0，Releases 0、Tags 0；审计四档 fold / merge / call as a tool / keep；真实模型验证范围自述极窄（仅两次示例运行、同一人、同一模型）。
- **AutoGen 同页矛盾残留**：当前 README 存档横幅逐字确认之外，“Why AutoGen?” 一节后文又写 “Note that AutoGen is now community-managed and responses may be limited.”——已存档与社区管理并存，属未清理历史残留，不作调和；存档日期与 2025-10 横幅原文仍不可见。
- **Agent Framework releases 复核戳**：最新带日期章节为 `## [1.23.0] - 2026-10-22`；其下 1.22.0（2026-10-15）、1.21.0（2026-10-08）；页顶无标题 Changes 清单的版本与日期未能确认，不得据此断言 2026-10-28 之后有新发布。
- **LangChain changelog 复核戳（仅 track，不进技能）**：页面 Updated 2026-10-23T09:08:55Z；置顶 `langchain-core` v1.7.0（`tool-call-delta` 逐段发送；搭配 `langchain` 1.4.7+）；置顶条目本身未标发布日期，不得断言其发布于 2026-10-28 之后。
- **OpenAI Agents SDK Python v0.24.0**：发布日期 2026-10-15（一手发布页标题行）；特性含 MCP 转换工具暴露 `mcp_annotations` 等；2026-10-28 之后新发布“未发现”（releases 列表页未全量打开，非“确认无发布”）。
- **Harbor 仓库身份**：原 laude-institute/harbor 已重定向至 harbor-framework/harbor（main 分支，30 releases、31 tags，创建 2025-08-04，Apache-2.0，未存档）；README 自称 Terminal-Bench-2.0 官方 harness；releases 日期未取，2026-10-28 之后有无新 release 查不到。
- **窗口内新动态**：五仓（LangChain/LangGraph、Agent Framework、OpenAI Agents SDK、Harbor、METR）均未确认 2026-10-28 之后的新发布；“未确认”不等于“确认无发布”。

### 已同步进 skill 参考

- agent-harness.md：新增 Anthropic 折叠文章段与迁移套件段；AutoGen 条目追加同页残留句；Agent Framework 句追加 1.23.0 复核戳；Agents SDK 条目追加 Python v0.24.0 戳；Harbor 条目改为迁址后身份。

### 剩余未核实缺口（更新）

1. AutoGen 存档具体日期与 2025-10 横幅逐字原文（当前页已不可见，需历史版本 README）。
2. Agent Framework 页顶无标题清单对应的版本号与日期；.NET 1.25.0 与 Python 1.23.0 对应关系（未一手统一核对）。
3. LangChain/LangGraph：GitHub releases 页直查（langgraph 1.2.18 之后、2026-10-28 之后有无新 tag；langchain-core 1.7.0 与 langchain 1.4.7 发布日期）。
4. OpenAI Agents SDK：openai-agents-python 与 openai-agents-js 的 releases 列表页全量核对（2026-10-28 之后有无新版本；TS 版未单独核对）。
5. Harbor：releases 列表页逐项日期（30 个 release 的最新几项）与 PyPI 正式版（非 dev 版）最新版本与日期。
6. METR：各仓库具体清单（Vivaria、Task Standard 参考实现、公开任务仓）逐仓取戳；Vivaria 无版本口径需直开正文确认。
7. MetaHarness：GitHub 仓库一手确认（2026-10-28 厂商通稿，下载量与星标数字不得引用）。
8. 深挖续题：工具搜索 85% / 10,000 工具数字一手确认；Claude Code 系统提示词削减超 80% 的原始出处。
9. 已解决：Anthropic 折叠文章一手原文、AutoGen 当前横幅与残留句、Agent Framework 1.23.0 章节、Agents SDK v0.24.0、Harbor 迁址。


---

## 2026-10-30 轮次：缺口复核——Agent Framework 定性、LangChain 戳、工具搜索 85% 一手确认

### 已核实（一手为主）

- **Agent Framework 页顶清单定性**：经内容比对定性为 dotnet-1.26.0（2026-10-23），标签页本身未直接打开（正文保留限定）；Python 线 1.21.0（2026-10-08）、1.22.0（2026-10-15）、1.23.0（2026-10-22，官方升级页）；python-1.24.0 标签页 404（不存在）；.NET 与 Python 为两条独立发布线。
- **LangChain 版本戳**：langchain-core 1.7.3（2026-10-29，PyPI 一手）；langchain 1.4.7（2026-10-21，PyPI 一手），官方配对口径 core 1.7.0+ 需配合 langchain 1.4.7+；langgraph 1.2.18（2026-10-26，二手）；2026-10-28 之后新 tag 未发现。
- **Agents SDK**：Python 最新仍为 0.24.0（2026-10-15，发布页 + PyPI 一手，2026-10-30 仍为最新）；JS @openai/agents 0.21.0（2026-10-20，二手日期），子包联动规则为官方文档一手。
- **Harbor PyPI 正式版**：`harbor` 0.25.0（2026-10-25，PyPI 一手）；开发版 0.24.1.dev202610230423 不是正式版；`SchemaRouter/harbor` 为 fork，非上游。
- **METR 两仓**：`METR/inspect-tasks-public` 与 `METR/RE-Bench` 均为公开存档、README 首行 DEPRECATED（仓库页一手）；Vivaria 文档确认 Git 拉取任务与 agent，配置示例默认任务仓为 `METR/mp4-tasks`（示例值）；Task Standard 博客 2024-02，“1000+ 任务”为 METR 自述。
- **MetaHarness 同名区分**：论文仓 `AmoghM/meta-harness`（创建 2026-04-15，9 提交，MIT，arXiv 2603.28052）vs Cognitum One 商业产品（2026-10-28 通稿二手，仓库页未确认）。
- **工具搜索 85% 一手确认（执行者亲自复核）**：Anthropic 原文——58 工具约 55K、传统约 72K / 总消耗约 77K、Tool Search 后约 8.7K，结论句 85% reduction，保留 95% 上下文窗口；Opus 4 49%→74%、Opus 4.5 79.5%→88.1%（内部测试自报）；算术注记：77K→8.7K 约 89%、55K→8.7K 约 84%，85% 贴近 55K 基线；“10,000 工具”无出处。
- **80% 削减出处**：claude.com《The new rules of context engineering for Claude 5 generation models》（2026-07-24）自述，非折叠文章；dev.to 词数二手未入原文。
- **AutoGen 现版 README（一手）**：存档横幅逐字仍在，同页残留 community-managed 句；2025-10 维护横幅逐字与 2025-10-02 日期仅为二手一致，存档执行日期仍未决。

### 已同步进 skill 参考

- agent-harness.md：Agent Framework 段改为 dotnet-1.26.0 定性 + Python 线日期 + 非锁步警示；LangGraph 行追加版本戳；Agents SDK 行改为 0.24.0 确认仍为最新 + JS 线；Harbor 行追加 PyPI 正式版与 fork 警示；METR 行改为两仓现状；新增 MetaHarness 区分条目与“工具搜索与提示词预算”一节（85% / 80% 出处与算术口径）。

### 剩余未核实缺口（更新）

1. dotnet-1.26.0 标签页本身的一手直接打开。
2. METR 的 Vivaria 本体仓、Task Standard 参考实现仓、`METR/mp4-tasks` 仓逐仓戳。
3. ruvnet/metaharness 仓库页一手确认。
4. “10,000 工具”一手出处。
5. AutoGen 微软执行存档的具体日期与 2025-10 横幅带日期历史版本。
6. Harbor GitHub releases 列表页完整逐项日期（方法局限：列表页文本提取丢标题行）。
7. 已解决：Agent Framework 页顶清单定性、Python 线日期、python-1.24.0 不存在、LangChain 版本戳、Agents SDK 确认、Harbor PyPI 正式版、METR 两仓现状、工具搜索 85% / 80% 出处。


---

## 2026-10-31 轮次：缺口复核——Agent Framework 10-30 两线、METR 存档、10,000 工具出处、AutoGen 表述修正

### 已核实（一手为主）

- **Agent Framework Python 1.24.0 与 .NET 1.27.0（2026-10-30，标签页一手）**：上一轮“python-1.24.0 不存在 / 未发现 dotnet-1.27.0”只是时点结论，现已失效；两线数字差继续扩大。Harness 要点为厂商自述：Python 实验性 in-run 压缩与技能卸载；.NET harness 本地聊天历史标记（#8708）。
- **dotnet-1.26.0 标签页身份已一手确认**：首批条目与定性逐项对应；日期仍为二手（页内无日期字段）。
- **METR Vivaria 与 task-standard 均已公开存档**：Vivaria 转向 Inspect、缩减新功能；Task Standard 不再维护、改用 Inspect（releases 横幅存档日 2026-10-07）；现行示例 standard_version 0.5.0。
- **METR/mp4-tasks 纠正**：仅为 Vivaria 配置文档 e.g. 示例值，公开仓 404，不得写成默认任务仓。
- **ruvnet/metaharness 仓库页已确认**：公开、未存档、创建 2026-06-13、MIT，生成自有品牌 harness 的工厂，与论文仓区分成立。
- **“10,000 工具”出处已找到（推翻“无出处”）**：Claude Agent SDK tool-search 文档 Limits 节逐字 “Maximum tools: 10,000 tools in your catalog”（Updated 2026-10-21），口径为目录上限，非实测规模。
- **AutoGen 维护提交与存档表述修正**：2025-10-02 提交 #7067（84f21a0）一手确认；仓库页平台 Archived 开关为 No，主分支 2026-10-16、2026-10-28 仍有提交组——存档是 README 声明，不是平台存档动作；横幅逐字句一手出处仍未落定。
- **LangGraph 1.2.19（2026-10-30，PyPI 一手）**；langchain-core 10-30/31 新版未能确认（PyPI 拦截、官方 changelog 页停留在 2026-10-23）。
- **严格 2026-10-31 窗口**：截至上午核查时未发现带日期的新发布。

### 已同步进 skill 参考

- agent-harness.md：Agent Framework 段改为 10-30 两线 + 时点结论失效说明 + harness 要点；dotnet-1.26.0 定性改为标签页确认；LangGraph 行改为 1.2.19；METR 行改为存档 + Inspect 转向 + mp4-tasks 纠正；工具搜索一节改为 10,000 上限出处；AutoGen 段追加 #7067 与 README/平台区分句。

### 剩余未核实缺口（更新）

1. dotnet-1.26.0 的 2026-10-23 日期一手字段。
2. AutoGen 平台存档执行日期（证据指向无平台动作）与横幅逐字句一手出处、横幅引入提交。
3. Harbor releases 完整逐项日期（列表页方法局限持续）。
4. langchain-core 10-30/31 新版；Agents SDK JS 线；ruvnet/metaharness 逐项日期。
5. 已解决：dotnet-1.26.0 标签页身份、METR 两仓、ruvnet/metaharness、“10,000 工具”出处、AutoGen 2025-10-02 提交、python-1.24.0 / dotnet-1.27.0、LangGraph 1.2.19。


---

## 2026-11-03 轮次：缺口复核——dotnet-1.26.0 一手日期、langchain-core 1.7.4、Harbor PyPI 日期、harness 文档分层

### 已核实（一手为主）

- **dotnet-1.26.0 的 2026-10-23 日期已解决（改道 NuGet 一手日期表）**：`Microsoft.Agents.AI` 版本表逐行列出 1.27.0 — 10/30/2026、1.26.0 — 10/23/2026；GitHub 标签页旧式格式无日期字段的局限复现并说明原因；deployedbyai 降为印证。
- **langchain-core 1.7.4（2026-11-02，PyPI 一手，执行者亲自复核）**：sdist 与 wheel 上传均标 Nov 2, 2026；取代 1.7.3；变更内容未打开；窗口内唯一跨轨正式发布之一。
- **OpenAI Agents SDK 两线复核戳推进**：Python 首项仍 0.24.0（2026-10-15）；JS 首项仍 0.21.0 系（日期未渲染，二手日期 2026-10-20 在窗口之前）。
- **Harbor 正式版仍 0.25.0（2026-10-25）+ 最新四项 PyPI 日期**：0.24.0（10-05）、0.23.0（09-12）、0.22.0（08-22）；0.21.0 页被拦截未核实；开发版 0.25.1.dev202610310503（10-31）不计正式版。
- **Agent Framework harness 文档状态分层**：UnloadSkills 有 Learn 官方文档原文；AgentTriggered 仅 1.24.0 changelog 一句话（Learn Compaction 页 2026-10-29 快照全文零命中）；#8708 仅 changelog 一句话，无设计文档语义。
- **AutoGen 现行存档横幅逐字句一手出处（raw README）**；引入提交收窄为 2026-03-26 或 2026-10-16 组（标题/哈希未渲染，仍未决）；2025-10 二手维护句在 #7067 diff 与当前 README 中均无此句。
- **ruvnet/metaharness 仓库页上一轮已确认**；本轮 releases 首项仍 0.4.19（日期未渲染），npm 页 403，逐项日期仍未决。
- **窗口内**：唯一带日期的一手正式发布为 langchain-core 1.7.4；LangGraph 主包仍 1.2.19（搜索片段级）；Agent Framework 仍 10-30 两线；METR 检索未见窗口内发布；langchain 主包未单独核对。

### 已同步进 skill 参考

- agent-harness.md：LangGraph 段改为 1.7.4（并注 langchain 主包未核对）；Agent Framework 段改为 dotnet-1.26.0 NuGet 一手日期 + harness 文档状态分层句；Harbor 行改为最新四项 PyPI 日期与缺口缩小；Agents SDK 行改为两线 2026-11-03 复核戳；MetaHarness 行改为仓库页已确认 + 逐项日期未决；AutoGen 段追加横幅逐字句一手出处与引入提交收窄句。

### 剩余未核实缺口（更新）

1. AutoGen 存档横幅引入提交（2026-03-26 或 2026-10-16 组）；二手维护句逐字出处。
2. Harbor 0.21.0 及更早的 PyPI 上传日期。
3. ruvnet/metaharness 逐项日期（首项 0.4.19 日期未渲染；npm 页 403）。
4. langchain-core 1.7.4 的 GitHub 标签页日期与变更内容；langchain 主包窗口核查。
5. AgentTriggeredCompactionStrategy 的 Learn 文档节；#8708 设计文档/参数语义。
6. OTel GenAI 官方 spec 本轮变动（仅二手重申）。
7. 已解决：dotnet-1.26.0 一手日期；AutoGen 现行横幅逐字句；Harbor 最新四项日期；langchain-core 1.7.4；Agents SDK 两线无新版；harness 文档状态分层。


---

## 2026-11-04 轮次：缺口复核——Harbor 防混淆、Agents SDK 复核戳、langchain-core 变更仍二手

### 已核实（一手为主）

- **Harbor 框架版最新稳定版仍为 0.25.0（2026-10-25，PyPI 一手，2026-11-04 项目页抓取）**；开发版 0.25.1.dev202610310503（2026-10-31）不计正式版。本轮补得早期版本 PyPI 上传日期：0.1.14（2025-11-12）、0.1.32（2025-12-30）、0.1.45（2026-02-27）、0.2.0a3（2026-03-19）、0.3.0a1（2026-03-26）、0.7.0（2026-05-15）、0.15.0（2026-06-19）、0.16.1（2026-06-28）、0.20.1.dev202608070026（2026-08-07）。**防混淆**：PyPI `harborapi`（容器镜像仓库客户端）与评估框架 `harbor` 是两个包，harborapi 0.21.0 为 2023-06-08，与框架版 0.21.0 无关；框架版 0.21.0 的 PyPI 上传日期本轮检索仍被同名包淹没而未取到（0.22.0 为 2026-08-22、0.23.0 为 2026-09-12、0.24.0 为 2026-10-05 沿用上一轮一手结论）。
- **Agents SDK 两线复核戳推进至 2026-11-04（窗口内无新版）**：Python 仍 0.24.0（2026-10-15，PyPI + GitHub release 页一手）；JS 仍 @openai/agents 0.21.0（2026-10-20，二手日期），两线版本号本就不同，不得合并。
- **langchain-core 1.7.4 变更内容仍仅二手转述**：kenashe.ai 文章（2026-11-02）称 GitHub releases 内容为“避免在 Anthropic translator 中修改消息内容”，并称与 langchain 1.4.8 同批警告 Python 3.10 将在 core 1.8.0 / langchain 1.5.0 中 dropped；pepy.tech 聚合称 1.7.4 发布 2026-11-02。GitHub 标签页本轮未直读成功，标签页日期与完整变更文本仍未决，不写入技能。langchain 主包窗口内无新稳定版，最新仍为 1.4.8（2026-11-02，窗口之前，二手）。
- **AutoGen 维护句引入日期二手冲突（新增张力，仍未决）**：维护句逐字文本经镜像与第三方引用双重印证；但 intelligenttools 称 README 于 2025-10-02 获得维护横幅，swarms 称横幅在 2026-04，两说冲突；存档横幅引入提交（2026-03-26 / 2026-10-16 组）仍未落定，不得任选。技能口径不变。
- **Agent Framework .NET 口径**：本轮一家二手源称 .NET 为 1.26.0，与上一轮 NuGet 一手日期表（1.27.0 — 10/30/2026）冲突，属过期口径，拒绝采纳；技能仍以 1.27.0 为准。AgentTriggered / UnloadSkills 参数语义与 #8708 设计文档仍未决（本轮未获 Learn 压紧页新快照证据，不刷新上一轮 2026-10-29 快照戳）。
- **OTel GenAI spec 页直读窗口内无可确认变动**（Development 状态，含 fetch_response 等节）；commit 级核查未做，不得写成穷尽证明。Harbor / METR trace 格式窗口内变动亦仅为“未发现证据”的否定性结论。
- **ruvnet/metaharness**：releases 首项 0.4.19 日期仍未决；npm 页本轮再试仍 403（不得写成“包不存在”）；厂商通稿自报成绩（OpenEnv Arena 第 2、平均 22.5%、9/40）未独立核实，不引用。

### 已同步进 skill 参考

- agent-harness.md：Harbor 段追加 harborapi 防混淆句与 2026-11-04 复核戳；Agents SDK 段复核戳推进至 2026-11-04。

### 剩余未核实缺口（更新）

1. AutoGen 存档横幅引入提交；2025-10 维护句引入日期（二手冲突：2025-10-02 vs 2026-04）。
2. Harbor 0.21.0 PyPI 上传日期（0.22–0.24 已核实）。
3. ruvnet/metaharness 0.4.19 日期；npm 403 原因。
4. langchain-core 1.7.4 GitHub 标签页一手日期与完整变更文本；langchain 主包窗口核查已做（无新版）。
5. AgentTriggered / UnloadSkills 参数语义；#8708 设计文档。
6. OTel GenAI commit 级窗口核查。
7. 已解决：Harbor / harborapi 区分；Agents SDK 两线 2026-11-04 复核戳。

---

## 2026-11-05 轮次：缺口复核——AutoGen 维护句裁定、ATIF 版本漂移、OTel 缓存口径分歧

### 已核实（一手为主）

- **AutoGen 维护句日期裁定推进**：2025-10-02 维护横幅说获得二手多源一致支撑（intelligenttools、atlan，出处指向 Discussion #7066；与 #7067 2025-10-02 提交一手记录一致）；2026-04 说法应弃（横幅后 main 仅 5 提交：2025-10-04 两次、2026-03/04 三次）。存档从维护升级为 archived 的确切日期仍未决；本轮公开检索未找到 2026-03-26 / 2026-10-16 组对应的存档提交，不得把二者写成存档引入提交。当前 README 一手抓取已为存档态，但页面不带引入日期。
- **Harbor 框架版 0.21.0 上传日仍未决，范围收窄**：存在性二手证据（managed-deepagents PyPI 页钉死 harbor[langsmith]==0.21.0）；时间 bracket：0.17.0（2026-07-03）→0.24.0（2026-10-05），0.22.0 为 2026-08-22、0.23.0 为 2026-09-12，故 0.21.0 落在 2026-07 至 08 之间（约 8 月），仅为 bracket 推断。harborapi 0.21.0 的 2023-06-08 是另一包，绝不能混用。
- **langchain-core 1.7.4 日期与变更**：发布日 2026-11-02（窗口之前）；核心变更据称引自 GitHub release：唯一值得注意的修复是“avoid mutating message content in the Anthropic translator”（二手转述，标签页一手直读仍未决）；同批 langchain 1.4.8 含 core 兼容层 bump（#41640）等，并重申 Python 3.10 将在 langchain 1.5.0 / langchain-core 1.8.0 移除。
- **Inspect 0.3.287（2026-10-31，官方 changelog 一手，窗口之前）**：react() 会话可在 turn 边界持有/重写、compaction handler 获 reset()（#6201）；0.3.286（2026-10-28）含 GPT-6.1 Terra 支持等。
- **OTel GenAI commit 级核查**：窗口内无新标签；最新可钉 commit 仍为 2026-10-06（二手钉点 4f85037）；专用仓创建 2026-05-05，截至 2026-10-12 仍零标签、零发布；约定本身标注 Development。下轮以该仓提交历史一手复核。
- **ATIF 版本漂移（深挖，须钉版本号）**：DeepWiki 快照为 ATIF-v1.6（schema_version、session_id、steps[]、final_metrics；tool_definitions 自 v1.5 加入）；atif-make 与 NVIDIA NAT 文档均已出现 ATIF v1.7（子 agent 轨迹引用、分文件 trajectory.<agent-id>.json；atif-make 对未知新 minor 字段直接丢弃而非拒绝）。Inspect .eval 日志、METR MALT 转录（树形节点＋branch_id）经 atif-make 桥接至 ATIF v1.7——评估日志格式与轨迹交换格式是两层，MALT 树形转 ATIF 需拍平。
- **OTel GenAI 缓存口径分歧（深挖，关键反证）**：趋同事件（2026-10-13 至 15，窗口前）：Claude Code 2.1.300 加 GenAI span 属性并拆分 cache_creation_5m/1h_tokens；Codex 0.163.0 支持自定义 OTLP metrics exporter；Gemini CLI v0.65.0-preview.0 加 OTLP 自定义头；Claude Platform 上线 OTel 导出 beta（二手）。但 Claude Code 的 input_tokens 排除缓存读/写，而 GenAI 约定要求计入——同一字段名、不同语义；官方属性表截至 2026-10-23 尚未跟上 2.1.300 新增字段。结论：OTel GenAI 是“可映射的观测趋同层”，不是可直接对账的计费/评估标准。
- **窗口内新鲜动态：六线均无带日期的一手新稳定发布**：LangChain/LangGraph（仍 1.2.19；langgraph-sdk 0.4.12 为 2026-11-03，窗口之前）；Agent Framework（Python 仍 1.24.0、.NET 仍 1.27.0；本轮一家二手源称 .NET 为 1.26.0，与上一轮 NuGet 一手表冲突，属过期口径，拒绝采纳）；OpenAI Agents SDK 两线（仍 0.24.0；openai-python 3.42.0 是平台 SDK，不是 Agents SDK）；Harbor（仍 0.25.0；仅开发版 0.25.1.dev20261031，窗口前）；METR/Inspect（Inspect 仍 0.3.287，窗口前）。

### 已同步进 skill 参考

- agent-harness.md：AutoGen 段追加维护句日期裁定与 .NET 口径拒绝采纳句；Trace 节追加 ATIF 版本钉定、Inspect 桥接、OTel 缓存口径分歧与 Inspect 最新版戳；Agents SDK / Harbor 复核戳推进至 2026-11-05。

### 剩余未核实缺口（更新）

1. AutoGen 存档引入提交日期（非维护句日期；需 GitHub archived 标记或 README 提交历史一手）。
2. Harbor 0.21.0 PyPI 上传日期（0.22–0.24 已核实；勿用 harborapi 2023-06-08）。
3. ruvnet/metaharness 0.4.19 归属、日期与 npm 403 具体原因（本轮公开检索无注册表页面命中，疑为误记版本号；但上一轮仓库页一手首项仍在，不得仅凭检索失败推翻，留待 GitHub releases 页直接复核）。
4. langchain-core 1.7.4 GitHub 标签页一手直读。
5. MAF #8708 与 AgentTriggered / UnloadSkills 真实归属与语义（本轮公开检索零命中；但上一轮 Learn 页一手证据确认 UnloadSkillsCompactionStrategy 有文档，短名零命中不得推翻，留待 Learn 页直接复核）。
6. OTel GenAI 仓提交历史一手复核（最新钉点暂为二手 4f85037，2026-10-06）。
7. 已解决：AutoGen 维护句日期裁定（2025-10-02 胜出）；Inspect 0.3.287 一手确认。

## 2026-11-07 轮次小节（版本 0.10.0；六路并行之 Harness 一路）

- 本轮增量：AutoGen 存档引入提交已解决（004749e，PR #8366，2026-10-16；2026-03-26 组 #7463 已排除；平台 Archived 开关仍为 No）；Harbor 0.21.0 PyPI 上传日已解决（2026-08-10，bracket 闭合）；metaharness 0.4.19 归属与日期已解决（tag dream-v0.4.19，2026-10-27T16:32:55Z，4 个包；0.4.20 不存在；npm 403 为 npmjs.com 网站机器人防护，注册表 API 正常，latest 为 0.4.21）；langchain-core 1.7.4 标签页一手直读（2026-11-02T15:31:47Z，仅两条变更，二手转述逐字属实）与 1.7.5/1.7.6、langchain 1.4.9、LangGraph 1.2.20、langgraph-sdk 0.4.14 / cli 0.4.42；MAF Learn 页再确认（UnloadSkills 有文档、AgentTriggered 全页零命中）与 #8708 语义钉死（已合并的 .NET 缺陷修复，非设计文档）、dotnet-1.28.0（2026-11-06）与 python-1.25.0 标签不存在（404，不得写成已发布）；OTel GenAI commit 级一手钉点推进至 e6658f1（2026-11-06）及同日两处实质约定变更（gen_ai.request.stream、BlobPart stripped_reason），仍零标签零发布；Agents SDK alpha 预发布轨（Python v0.24.0-alpha.6 / JS v0.22.0-alpha.1，2026-11-03，GitHub-only）；Harbor 仍 0.25.0、Inspect 仍 0.3.287（复核戳推进至 2026-11-07；Inspect 仓库正确路径为 UKGovernmentBEIS）。
- 已同步进 skill 参考：agent-harness.md 追加 AutoGen 存档引入提交段、Harbor 0.21.0 日期段、metaharness 0.4.19 归属段、langchain-core 1.7.4 直读段、MAF Learn/#8708 段、OTel 钉点段、Agents SDK alpha 轨段；Trace 节追加 OTel 新约定变更 bullet；复核戳推进至 2026-11-07。
- 推翻的旧说法：AutoGen“存档引入提交未找到”与“last push 15 April 2026”二手说；metaharness“版本号误记”推测；上一轮“仍 1.7.4”漏检（1.7.5/1.7.6 已发布）；Inspect 仓库路径 UKGovernmentBE 写法（404，属路径笔误）。
- 剩余未核实缺口（更新）：1. AutoGen 004749e 精确 UTC 时间戳（git log）。2. langchain-core 1.7.5/1.7.6 PyPI 上传日期。3. AgentTriggered 参数语义（改道 Python 源码直读）。4. METR 各仓逐仓复核戳（本轮未逐仓开）。5. MAF Python 1.25.0 发布状态。6. OTel GenAI 新约定对技能 Trace 一节的影响评估。7. 已解决：AutoGen 存档引入提交；Harbor 0.21.0 日期；metaharness 0.4.19 归属与 npm 403 原因；langchain-core 1.7.4 直读；OTel 钉点推进至 2026-11-06；Inspect 仓库路径。

## 2026-11-07 轮次小节（下午轮，版本 0.11.0；六路并行之 Harness 一路）

- 本轮增量：AutoGen 004749e 精确时间戳一手定稿（2026-10-16T16:14:36Z，作者与提交者同瞬，GPG 已验；2026-10-17 00:14:36 CST）；langchain-core 1.7.5/1.7.6 PyPI 上传日期一手（2026-11-04T15:49:43Z / 2026-11-05T00:21:54Z，各比 GitHub published_at 早约 20 秒），无 1.7.7/1.7.8；AgentTriggeredCompactionStrategy 参数语义源码一手解决（python/packages/core/agent_framework/_compaction.py:1299，HEAD 924a6e50，2026-11-06：实验性 wrapper，tool_name 默认 "compact_context"、approval_mode 默认 "never_require"、exclude_trigger 默认 False；须客户端管理历史；UnloadSkills 暴露 unload_skill）；METR 逐仓存档核查（GitHub Repo API 一手：vivaria / task-standard / re-bench / inspect-tasks-public / public-tasks 五仓 archived=True，最后三仓同属 2026-10-20 归档批；inspect_ai 未存档、活跃，最新提交 2026-11-07T04:10:42Z）；MAF python-1.25.0 标签存在（轻量 tag，7bca1df，2026-11-06T16:05:38Z），无 Release 对象、无 PyPI（上一轮“标签不存在”撤回）；Inspect 0.3.288（GitHub Release published_at 2026-11-07T01:13:21Z，上一轮因 changelog 滞后漏检；破坏性：google-genai>=3.0.0、显式 Docker 沙箱镜像、web_browser() → web_search()/computer()；PyPI 侧未核查）；Agents SDK JS 0.21.0 日期一手（2026-10-20T22:07:29Z）；OTel 两处新增约定一手命名（gen_ai.request.stream、BlobPart stripped_reason），评估为不改 Trace 节。
- 已同步进 skill 参考：agent-harness.md 更新 AutoGen 004749e 精确时间戳段、langchain-core 1.7.5/1.7.6 PyPI 日期段、AgentTriggered 参数语义段（Learn 页零命中改为源码一手）、METR 逐仓存档段、MAF python-1.25.0 标签存在纠正段、Inspect 0.3.288 段、Agents SDK JS 0.21.0 日期段、OTel 新约定段。
- 推翻的旧说法：AutoGen 004749e“无精确 UTC 时间戳”；langchain-core 1.7.5/1.7.6“PyPI 上传日期未取”；AgentTriggered“参数语义未决”；MAF“python-1.25.0 标签不存在（404）”；Inspect“最新仍 0.3.287”；Agents SDK JS 0.21.0“日期二手”；METR“各仓逐仓戳未开”（本轮逐仓核查完成）。
- 剩余未核实缺口（更新）：1. MAF 1.25.0 搁置原因（有 tag、无 Release、无 PyPI）。2. Inspect 0.3.288 PyPI 侧。3. OTel 后续约定。4. Agents SDK alpha 轨（Python v0.24.0-alpha.6 / JS v0.22.0-alpha.1）。5. AgentTriggered 实验性晋升。6. METR 其余仓库（逐仓归档状态与最新提交）。7. 已解决：AutoGen 004749e 精确时间戳；langchain-core 1.7.5/1.7.6 PyPI 上传日期；AgentTriggered 参数语义；METR 五仓逐仓戳；MAF python-1.25.0 标签存在性；Inspect 0.3.288；Agents SDK JS 0.21.0 日期。

## 2026-11-07 轮次小节（晚间轮，版本 0.12.0；六路并行之 Harness 一路）

- 本轮增量：MAF Python 1.25.0 在本轮核查进行中上架 PyPI（agent-framework-core 15:41:42Z、agent-framework 15:42:23Z，另 ag-ui 1.8.0、declarative 1.5.0、openai 1.18.1、foundry 1.16.1 已抽查；6/44 包，余 38 包未复核；GitHub Release 对象截至 23:43 CST 仍 404；tag→PyPI 间隔约 23.6 小时，定性为分阶段发布进行中，推断）；dotnet 仍 1.28.0；#8708 已关闭（2026-10-29）；Inspect 0.3.288 PyPI 侧确认（2026-11-07T01:24:49Z，约晚 GitHub Release 11 分钟），无 0.3.289+；OTel GenAI e6658f1 仍为 HEAD（2026-11-06T03:31:30Z），之后零提交，零标签零发布；Agents SDK alpha 无新版（Python 仍 v0.24.0-alpha.6、JS 仍 v0.22.0-alpha.1，均为 2026-11-03；稳定线 Python 0.24.0 / JS 0.21.0）；AgentTriggered 仍 @experimental，但 Learn 压紧页现含 "Let the agent request compaction" 一节（一手确认，L865；与下午轮“Learn 页零命中”有张力——下午轮查的是另一页面或窗口内新增，上线时间未决）；METR 全 org 69 仓复核，已存档 10 仓（下午轮五仓 + 本轮新增 hcast-public、inspect-action、pyhooks、vivaria-mentat、agent-fork-of-eval-analysis-public）；窗口内活跃 7 仓均为 Inspect 生态；Harbor 仓库迁移确认（laude-institute → harbor-framework，API 301），仍 v0.25.0；LangChain JS 1.6.3（2026-11-07T04:27:12Z）与 LangGraph JS 1.7.2（2026-11-07T00:07:41Z）为 11-07 白天发布（窗口起点之前，轮间补漏）；LangChain Python 仍 1.4.9、LangGraph Python 仍 1.2.20、langchain-core 仍 1.7.6。
- 已同步进 skill 参考：agent-harness.md 更新 MAF 1.25.0 PyPI 上架段（推翻“无 PyPI”）、Inspect 0.3.288 PyPI 段、OTel 戳推进段、Agents SDK alpha 戳推进段、AgentTriggered Learn 文档化段（张力未消解）、METR 全 org 存档段、Harbor 迁移段、LangChain/LangGraph JS 新版段。
- 推翻的旧说法：MAF“python-1.25.0 无 PyPI / 不得写成已发布”（本轮核查进行中上架）；Inspect 0.3.288“PyPI 侧未核查”；AgentTriggered“Learn 页零命中”（压紧页现已文档化，另一页面说法仍存）。
- 剩余未核实缺口（更新）：1. MAF 1.25.0 其余 38 个包的 PyPI 上架情况与 Release 对象出现与否。2. AgentTriggered Learn 压紧页一节上线时间。3. OTel 后续约定。4. Agents SDK alpha 轨。5. 已解决：MAF 1.25.0 PyPI 上架；Inspect 0.3.288 PyPI 侧；METR 全 org 存档计数；Harbor 仓库迁移。

## 2026-11-08 轮次小节（晨轮，版本 0.13.0；六路并行之 Harness 一路）

- 本轮增量：MAF Python 1.25.0 完成发布——44/44 包全部上架 PyPI（2026-11-07T15:41:25–15:43:18Z，约 113 秒；agent-framework-core 15:41:25Z、agent-framework 15:42:23Z、agent-framework-ag-ui 15:41:35Z、azure 15:42:03Z、azureai 15:41:26Z、declarative 15:41:42Z、foundry 15:42:31Z、hosted 15:41:33Z、openai 15:41:42Z、opentelemetry 15:41:42Z 均已抽查），GitHub Release 对象已补发（published 2026-11-07T15:52:42Z，作者 eavanvalkenburg，89 附件）；破坏性变更：Python 3.10 移除（#9704）、MCP 发现绑定（100 页/1,000 项、30 秒超时、#10002）、工作流 span 与 OTel 对齐（#9900）、每次 handler 新建 MCP 会话（#8662）、单授权范围绑定（#10035）、#10031/10043/10044/10061/10063；dotnet 仍 1.28.0；#8708 已关闭（2026-10-29）；OTel GenAI「stripped_reason」纠正——被评审删除的提案字段，非现行约定（PR #144 时间线 + main 分支两文件零命中双重证实）；BlobPart.content 可选已合并（L150）；Trace 一节加法式增补三点（`gen_ai.request.stream` 行（L104）、BlobPart 中间态（L150）、`invoke_workflow` 行（L154））；`gen_ai.workflow.name` 待确认 discrepancy（MAF 侧命名，规范注册表无此属性，语义是工作流的 `name` 属性或其 id 之外的唯一标识）；AgentTriggered Learn 压紧页上线区间收窄至 2026-10-29→11-07，最可能 ~11-01（张力消解：仍 @experimental，但 Learn 文档化已确认）；Agents SDK alpha 轨全文已读（Python v0.24.0-alpha.6 / JS v0.22.0-alpha.1，均为 2026-11-03；原生运行时重写预览，无稳定化时间表；a2a-client/redis 实验模块一手确认）；Inspect 仍 0.3.288（PyPI 2026-11-07T01:24:49Z；changelog 页顶仍为 0.3.278，缓存滞后）；Harbor 仍 0.25.0（开发版 0.25.1.dev202611070448 仍在）；LangChain Python 仍 1.4.9、LangGraph Python 仍 1.2.20、langchain-core 仍 1.7.6；补漏 @langchain/core JS 1.3.5（2026-11-05）；METR 全 org 69 仓/10 存档，窗口内唯一推送为 hawk（2026-11-07T21:26:22Z）。
- 已同步进 skill 参考：agent-harness.md 追加 MAF 1.25.0 完成发布段、AgentTriggered 区间收窄段、Agents SDK alpha 全文段、Trace 两处 stripped_reason 纠正段与 gen_ai.workflow.name 待确认段、LangGraph 2026-11-08 复核段、METR hawk 推送段、Inspect 2026-11-08 复核段。
- 推翻的旧说法：MAF 1.25.0「仅 6/44 包上架、Release 对象仍 404」（本轮 44/44 + Release 对象已补发）；OTel「stripped_reason 为现行约定」（被删提案字段）；AgentTriggered「Learn 压紧页上线时间未决」（区间已收窄至 2026-10-29→11-07，张力消解）。
- 剩余未核实缺口（更新）：1. MAF 1.25.0 tag→PyPI 间隔原因（约 23.6 小时，分阶段发布推断）。2. AgentTriggered 精确上线日期。3. OTel aabmass 提案（属性别名 + #161 对齐）；gen_ai.workflow.name 是否入规范注册表。4. Agents SDK alpha.7 / 稳定化时间表。5. METR hawk 具体提交日期。6. 已解决：MAF 1.25.0 完成发布；stripped_reason 纠正；AgentTriggered 区间收窄；Agents SDK alpha 轨全文。

## 2026-11-08 轮次小节（中午轮，版本 0.14.0；六路并行之 Harness 一路）

- 本轮增量：METR hawk 窗口内新提交 220a425（2026-11-08T03:05:04Z，AMI 升级应对内核 CVE-2026-98432，infra 维护）；#2820 提交日期纠正为 2026-11-06T23:16:11Z（推送 11-07，须区分推送时间与提交时间）；org 仍 69 仓/10 存档。`gen_ai.workflow.name` 已移除——折叠进 `gen_ai.agent.name`（提交 aa3b7f4，2026-11-05，PR #571），Trace 节口径修正为以 `gen_ai.agent.name` 为准，workflow.name 记为已移除旧名。MAF 1.25.0 tag→PyPI 间隔（约 23.6h）仍无官方解释；新增机制证据（python-release.yml 注释证实 Azure DevOps + ESRP 多阶段门控）与对照时延（1.24.0 约 10.2h、dotnet-1.28.0 约 3.4h）；仍为推断。OTel aabmass 全字段可选提案仅为 fork 分支三条提交，无独立 PR、未合并；任务书括号「属性别名 + #161 对齐」未能对应一手记录（#161 是 sampling_relevant 的 PR），不采纳。Agents SDK alpha.7 未出现，稳定化时间表仍无官方说法；版本戳：Inspect 仍 0.3.288、Harbor 仍 0.25.0、LangChain 1.4.9、LangGraph 1.2.20、core 1.7.6、MAF.NET 1.28.0；AgentTriggered 精确上线日期仍未决（区间 2026-10-29→11-07，最可能 ~11-01）。
- 已同步进 skill 参考：agent-harness.md 追加 METR hawk 220a425 段与 #2820 日期纠正段、Trace 节 workflow.name 移除纠正段、aabmass fork 分支段、MAF 间隔机制证据段、Agents SDK alpha.7 复核段、版本戳推进段（LangChain/LangGraph/Harbor/Inspect/MAF.NET）。
- 推翻的旧说法：`gen_ai.workflow.name`「待确认 discrepancy」（已移除，折叠进 `gen_ai.agent.name`）；MAF 1.25.0 间隔「纯推断」（升级为带机制证据的推断）；#2820「提交日期 2026-11-07」（实为推送日期，提交本身为 2026-11-06）。
- 剩余未核实缺口（更新）：1. MAF 1.25.0 tag→PyPI 间隔官方解释（约 23.6 小时；机制证据已补，官方说法仍缺）。2. AgentTriggered 精确上线日期。3. OTel aabmass 提案（fork 分支三提交，未合并）。4. Agents SDK alpha.7 / 稳定化时间表。5. METR hawk 后续提交。6. 已解决：hawk 220a425；#2820 提交日期；gen_ai.workflow.name 移除；MAF 间隔机制证据与对照时延。

## 2026-11-08 轮次小节（晚间轮，版本 0.15.0；六路并行之 Harness 一路）

- 本轮增量：Agents SDK Python v0.24.0-alpha.7（tagger 2026-11-08T06:56:18Z，提交 c27b24cc，61 提交，GitHub-only）与 JS v0.22.0-alpha.2（提交 3b8fe808，2026-11-08T08:14:35Z，41 提交，GitHub-only）窗口内发布，均无稳定化时间表；上一轮「仍无 alpha.7」作废。重大纠正：aabmass 提案已有独立 PR #688 且 Open（2026-11-05 ready for review，头提交 e69f0e7 与 fork 分支头一致），fork 分支实为两条提交（b4bfb5a0d、e69f0e7）不是三条；lmolkova 提 null 分支异议（建议 SkipJsonSchema[None]）；#144 前置已于 11-06 合并。langgraph-sdk JS（1.15.0 / cli 1.7.0）与 Python（0.4.14 / 0.4.42）为两条版本线，不得混用。Google ADK 2.14.0（PyPI 11-07T01:03:32Z 一手版本，内容二手：TeamsRunner、MultiRunConfig、26 模块毕业、23%/9% 厂商自报未核实；官方 release notes 待复核，未验证暂定 H4）。版本戳：Inspect 0.3.288、Harbor 0.25.0、LangChain 1.4.9、LangGraph 1.2.20、core 1.7.6、MAF.NET 1.28.0；MAF 间隔仍无官方解释；AgentTriggered 精确日期仍未决；OTel HEAD 仍 e6658f1；hawk 窗口内零提交；AutoGen HEAD d07c667a。**仅记本笔记、不入技能正文**：HF harness-hub LoRA 适配器公告（11-06）为二手线索，未一手复核。
- 已同步进 skill 参考：agent-harness.md 追加 Agents SDK alpha.7/alpha.2 段、aabmass PR #688 纠正段、langgraph-sdk 两条版本线段、Google ADK 2.14.0 新条目（未验证暂定标注）、版本复核戳段。
- 推翻的旧说法：OTel aabmass「仅 fork 三提交、无独立 PR」（PR #688 存在且 Open，分支实为两条提交）；Agents SDK「仍无 alpha.7」（alpha.7/alpha.2 窗口内发布）。
- 剩余未核实缺口（更新）：1. MAF 1.25.0 tag→PyPI 间隔官方解释（约 23.6 小时；机制证据已补，官方说法仍缺）。2. AgentTriggered 精确上线日期。3. OTel PR #688 审定结果（lmolkova 异议待决）。4. Agents SDK alpha.7 后续与稳定化时间表。5. METR hawk 后续提交。6. ADK 官方 release notes 一手复核（H4 未验证暂定）。7. 已解决：aabmass PR #688 存在性；Agents SDK alpha.7/alpha.2；langgraph-sdk 两条版本线。

## 2026-11-08 轮次小节（深夜轮，版本 0.16.0；六路并行之 Harness 一路）

- 本轮增量：Google ADK 2.14.0 官方 release notes 全文一手复核（published_at 2026-11-07T00:58:18Z，body 日期 2026-11-04；TeamsRunner、MultiRunConfig（`adk eval --num_runs` + `--parallelism`）、26 模块毕业分类纠正为 auth/environment/app/CLI/cache（非 "runtime"）、23%/9% 厂商自报无基线；三项破坏性变更：枚举工具参数传枚举成员、VertexAiSessionService.create_session 仅接受会话配置、RunSkillScriptTool 环境模式禁自由 command 参数（官方安全理由逐字引用））。OTel PR #688 仓名补正：提案在独立仓 open-telemetry/semantic-conventions-genai（主仓 #688 为 2024 年已合并编辑 PR）；仍 Open，2026-11-06 最后活动；AgentGymLeader 已批准；lmolkova 异议已由作者以 SkipJsonSchema[None] 落实于 29 个 Optional 字段（51/66 可选属性拒 null）；生产者义务缺口仍未解决；前置 #144 已合并。Agents SDK 两仓实名：openai/openai-agents-python 与 openai/openai-agents-js（短名 404）；仍 alpha.7/alpha.2，无稳定化时间表。MAF .NET 包名纠正：官方 NuGet ID 为 Microsoft.Agents.AI（Microsoft.AgentFramework 零命中）；仍 1.28.0（2026-11-06T21:57:55Z，65 版本）；#10151 全文无分阶段发布解释，间隔仍无官方说法。AgentTriggered 下界收紧至 2026-10-28T16:25:33Z（PR #9384，提交 c2a8f91b）；可及一手途径已穷尽（Learn 源仓为私有 MicrosoftDocs/azure-ai-docs-pr，公共镜像 404，Wayback CDX 2026-10-28→11-08 零快照），仍未决。版本戳：Inspect 仍 0.3.288、Harbor 仍 0.25.0、LangChain 1.4.9、LangGraph 1.2.20、core 1.7.6、MAF Python 仍 1.25.0；OTel HEAD 仍 e6658f1；hawk 零提交；org 仍 69 仓/10 存档；AutoGen HEAD d07c667a。
- 已同步进 skill 参考：agent-harness.md 改写 ADK 2.14.0 段（一手复核 + 分类纠正 + 破坏性变更）、PR #688 段追加仓名补正、Agents SDK 段追加仓名补正与复核戳、MAF 复核戳段追加包名纠正与 #10151 结论、AgentTriggered 段追加下界收紧与穷尽记录、复核戳段追加深夜轮全量版本戳。
- 推翻的旧说法：ADK 2.14.0「内容二手，未读官方 release notes」（现全文一手复核）；「26 个运行时模块毕业」（官方分类无 "runtime"）；OTel PR #688「在 open-telemetry/semantic-conventions 仓」（实为独立仓）；MAF .NET 包名「Microsoft.AgentFramework」（官方 NuGet ID 为 Microsoft.Agents.AI）；AgentTriggered「最可能 ~2026-11-01」（下界收紧后记为未决，不再写最可能日期）。
- 剩余未核实缺口（更新）：1. MAF 1.25.0 tag→PyPI 间隔官方解释（机制证据已补，官方说法仍缺；#10151 已排除）。2. AgentTriggered 精确上线日期（可及一手途径已穷尽）。3. OTel PR #688 审定结果（lmolkova 异议已落实，生产者义务缺口待决）。4. Agents SDK alpha 后续与稳定化时间表。5. METR hawk 后续提交。6. ADK 2.14.1/2.15.0。7. 已解决：ADK 2.14.0 一手复核；OTel PR #688 仓名；Agents SDK 仓名；MAF .NET 包名；AgentTriggered 下界收紧。

## 2026-11-09 晨轮（第 17 轮，窗口 2026-11-08 23:37 → 2026-11-09 05:37）

- 本轮增量 H1–H2（2 条）；版本 0.16.0 → 0.17.0。
- H1：AgentTriggered 精确上线日期缺口**冻结**（track-only，无技能正文改动）：Learn 页仍有该节（5 命中）、公共镜像源码仍 404，可及一手途径已穷尽；区间封存 2026-10-29 → 11-07（代码合并 2026-10-28T16:25:33Z 为下界）；今后各轮不再重搜，除非出现新线索。
- H2：METR hawk——窗口内 2 提交（8956c4c 2026-11-08T16:14:36Z k8s 只读角色 #2691；68680b4 2026-11-08T21:20:10Z 搜索 rls_bypass 修复 #2514）+ 窗口外 2 提交（6d00080 21:38:11Z 文档 #2828；0ff9098 21:40:29Z runner 错误 spans #2809，现 HEAD），已标注防下轮重复计数；组织仍 69 仓库 / 10 归档。
- 其余版本戳全不变：ADK 2.14.0、Agents SDK Python alpha.7 / JS alpha.2 + 稳定版 0.24.0/0.21.0、MAF Python 1.25.0 / .NET 1.28.0、Inspect 0.3.288、Harbor 0.25.0、LangChain/LangGraph/langchain-core 各线不变、AutoGen HEAD d07c667 不变、OTel GenAI HEAD e6658f1 不变；OTel PR #688 仍 Open，updated_at 仍 2026-11-06T00:59:26Z，窗口内零活动。

### 剩余未核实缺口（本轨）

- AgentTriggered 精确上线日期（已冻结，见上）。
- OTel GenAI PR #688 合并动向（仍 Open）。

## 2026-11-09 中午轮（版本 0.18.0；窗口 2026-11-09 05:37 → 11:37 CST）

- 本轨增量（H1–H4）：MAF 1.25.0 23.6h 锚点纠正（release created → PyPI 上传，轻量 tag 无独立时间；上传早于 published_at 约 11 分钟，先传包后点发布；ADK 同流程约 5 分钟；官方解释仍缺失）；Agents SDK alpha created/published 字段纪律；OTel PR #688 复核戳（仍 Open，updated_at 仍 2026-11-06T00:59:26Z，窗口内零动向）；hawk +11 提交（HEAD 008399f，含 SEC-507）与 Inspect +2 提交（HEAD 319a319，#6401）；其余版本戳全不变。
- 纠正：Harness 腿「Inspect/Harbor slug 更正」针对的是派发书背景错误，技能正本早已正确，不入技能。
- 剩余未核实缺口：MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结，见上）；OTel GenAI PR #688 合并动向（仍 Open）；Agents SDK 稳定化时间表；METR 组织计数未复核。

## 2026-11-09 晚间轮（版本 0.19.0；窗口 2026-11-09 11:37 → 17:37 CST）

- 本轨增量（H1–H4）：OTel PR #688 复核戳（仍 Open，updated_at 仍 2026-11-06T00:59:26Z，窗口内零动向，HEAD 仍 e6658f1）；Agents SDK alpha.6 字段纪律补充（created 与 published 相差约 6.4 小时，alpha.7 仅约 9 分钟；仍无 alpha.8/alpha.3、无稳定化时间表）；METR hawk 窗口内提交 804f743f（2026-11-09T04:33:32Z，#1473，Ruff 规范化，300 文件，组织仍 69 仓/10 归档）；Rethinking Verification in Agent Harnesses arXiv:2610.34577v1 轮间补漏（17 种验证设计 × 7 前沿编码模型）。其余版本戳全不变（Inspect 0.3.288、Harbor 0.25.0、LangChain 1.4.9、LangGraph 1.2.20、core 1.7.6、ADK 2.14.0、AutoGen HEAD d07c667a）。
- 剩余未核实缺口：MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结，见上）；OTel GenAI PR #688 合并动向（仍 Open）；Agents SDK 稳定化时间表；METR 组织计数未复核。

## 2026-11-09 深夜轮（版本 0.20.0；窗口 2026-11-09 17:37 → 23:37 CST）

- 本轨增量（H1–H5）：hawk 窗口内 3 提交（含破坏性沙箱收紧 b8d1004 #2393）；Agents SDK JS 窗口内 13 提交（含 tracing 导出容错 #2277）；Inspect 窗口内 2 提交（含 eval-sharding merge 命令设计文档 #6345）；轮间补漏 6 篇 harness 论文摘要级（Trace2Eval、The Harness Is the Model、Agent Harness Awareness、AgenticFinBench、Trial by Trace、VCC；CLEO 仅二手转述未读原文，不得引用）；复核戳：OTel PR #688 仍 Open 零动向、Agents SDK 无 alpha.8/alpha.3、MAF 仍 1.25.0/1.28.0（间隔官方解释仍缺，#10151 页面可见文本无解释）、ADK 无 2.14.1/2.15.0、版本戳全不变。窗口内新稳定发布为零。本轮 GitHub API 额度轮内耗尽，耗尽后改道 git 协议与注册表 JSON。
- 剩余未核实缺口：MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结，见上）；OTel GenAI PR #688 合并动向（仍 Open）；Agents SDK 稳定化时间表；CLEO 原文直读。

## 2026-11-10 凌晨轮（版本 0.21.0；窗口 2026-11-09 23:37 → 2026-11-10 05:37 CST）

- 本轨增量（H1–H6）：CLEO 全文首读（arXiv:2611.03434v1，CIKM '26；83.8% / ~3k LOC vs Claude Code SDK 85.0% / ~512k，作者自认 1.2pp≈3 题属实质平手；~512k 须附论文自注「闭源压缩、无公开可验精确行数」）；hawk 窗口内 10 提交（含沙箱 glovebox RuntimeClass #2367，接续 b8d1004 收紧方向）；Inspect 窗口内 9 提交（HEAD a0ce5ab5，#6303 sentinel sidecar）；ADK 窗口内 12 提交（88b3afe 关闭 FastAPI 原生 tracing 防双重追踪；112491d RAG 内存搜索 6× 宣称为作者自报）；复核戳：Harbor 仍 0.25.0（窗口开发版 0.25.1.dev202611090753 非正式版）、MAF 仍 1.25.0/1.28.0（1.25.0 间隔官方解释仍缺）、Agents SDK 两线 HEAD 不变、无 alpha.8/alpha.3、OTel PR #688 仍 Open 零动向（updated_at 仍 2026-11-06T00:59:26Z）。边界：hawk bbe41e2 与 Harbor 三提交落在窗口关闭后 19 秒–4 分钟，计入下轮。窗口内新稳定发布为零。
- 剩余未核实缺口：MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结，见上）；OTel GenAI PR #688 合并动向（仍 Open）；Agents SDK 稳定化时间表；hawk bbe41e2 / Harbor 窗口外三提交（下轮起始 HEAD）；CLEO 同类最小化 harness 追踪。

## 2026-11-10 中午轮（版本 0.22.0；窗口 2026-11-10 05:37 → 11:37 CST）

- 本轨增量（H1–H10）：边界提交确认（hawk bbe41e2、Harbor 三提交为本轮起始 HEAD）；hawk 窗口内 13 提交（HEAD f454114，glovebox 沙箱三联 #2368/#2350/#2339 + inspect-ai 钉死 METR fork #2574/#2875；组织仍 69 仓 / 10 归档）；Inspect 窗口内 2 提交（HEAD 6d2b526，#6422 ACP import 修复，仍 0.3.288）；ADK 窗口内 7 提交（HEAD ab63844；b41fc57 在 3 小时内被 6fdc953 回退；112491d「RAG 内存搜索 6×」仍作者自报；仍 2.14.0）；Agents SDK（Python 零提交 HEAD 1fc77bbc，无 alpha.8；JS 2 提交 HEAD 7589b9c，#2290/#2279，无 alpha.3；无稳定化时间表）；MAF 窗口内 1 提交（0982141 #10132，[BREAKING] Foundry Hosting API 表面缩减；仍 1.25.0/1.28.0；间隔官方解释仍缺）；Harbor 窗口内 4 提交（HEAD 5d5e58a，#4180 Antigravity 网关鉴权，仍 0.25.0，无新开发版）；LangChain JS 线窗口内新版（@langchain/core 1.3.6 #12140、@langchain/langgraph 1.7.3 #3308/#3090；伴生 1.7.1/2.3.6/1.6.2；Python 线未变）；OTel PR #688 仍 Open 零动向（updated_at 仍 2026-11-06T00:59:26Z，HEAD 仍 e6658f1）；arXiv 11-09 周一批次 harness 论文摘要级登记 6 篇（BOXBOX 2611.09218、2611.09088、2611.09212、Flex-RSI 2611.08946、DRAFT-Bench 2611.08933、2611.09792；arXiv API submittedDate 索引本轮滞后，改用 cs.AI 列表页 217/220，尾部 3 篇缺口注明）；AutoGen HEAD 仍 d07c667a 零提交；CLEO 仍 v1；AgentTriggered 精确上线日期仍冻结。窗口内新稳定发布为零。
- 剩余未核实缺口：MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结，见上）；OTel GenAI PR #688 合并动向（仍 Open）；Agents SDK 稳定化时间表；CLEO 同类最小化 harness 追踪；hawk f454114 / Harbor 5d5e58a 为下轮起始 HEAD。

## 2026-11-10 晚间轮（版本 0.23.0；窗口 2026-11-10 11:37 → 17:37 CST）

- 本轨增量（H1–H5）：窗口内新稳定发布一件——langchain-core Python 1.7.7（2026-11-10T04:08:27Z，PyPI 04:08:08Z，目标提交 1a41b51b；`load()` 嵌套深度上限安全修复 #41742 + 依赖安全锁定 #41585；深度上限精确阈值未深查，为新缺口）；窗口内提交：hawk 3（HEAD 50a47b5，glovebox 沙箱三联 #2883/#2884/#2885）、Harbor 2（HEAD ec66182，#4186 serve 默认端口 →8390 行为变更）、ADK 1（HEAD efc7717，工作流中断修复；233 测试作者自报）、Agents SDK Python 2（HEAD d689616）、MAF 1（HEAD ee538ca，#10214）、langchain 单仓 3 提交（含 1.7.7 发布；23bacbe 中间件内部调用移出 messages 流）、langgraphjs 1 提交（af1d87c）；OTel PR #688 仍 Open 零动向（updated_at 仍 2026-11-06T00:59:26Z，HEAD 仍 e6658f1）；arXiv 11-10 周二批次 harness 论文摘要级登记 6 篇（2611.11971 Harness Matters、2611.11994 EmoHarness、2611.12122 Agency、2611.12300 RAT、2611.12386 Say Shibboleth、2611.12552 SciMARS；边缘 2611.11710/2611.11720；11-10 00:00–09:37 harness 提交 0）；复核戳：langchain 主线仍 1.4.9、langgraph 仍 1.2.20、langgraph-sdk 仍 0.4.14；Harbor 仍 0.25.0；ADK 仍 2.14.0；Agents SDK 稳定版仍 Python 0.24.0 / JS 0.21.0；MAF 仍 1.25.0/1.28.0；Inspect 仍 0.3.288；AutoGen HEAD 仍 d07c667a。
- 剩余未核实缺口：langchain-core 1.7.7 `load()` 深度上限精确阈值；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI PR #688 审定结果；Agents SDK 稳定化时间表；CLEO 同类最小化 harness 追踪；hawk 50a47b5 / Harbor ec66182 为下轮起始 HEAD。

## 2026-11-11 凌晨轮（版本 0.24.0；窗口 2026-11-10 17:37 → 2026-11-11 01:09 CST）

- 本轨增量（H1–H7）：窗口内新稳定发布一件——inspect-ai 0.3.289（2026-11-10T16:55:29Z；Claude Fable 5.2 支持、sentinel sidecar；PyPI wheel 16:57:35Z / sdist 16:57:39Z；H2 自验修正为两文件分列时间）；langchain-core 1.7.7 `load()` 深度上限 200 层闭合（源码一手，`_MAX_DEPTH = 200`，`depth > 200` 抛 ValueError；上一轮新缺口）；hawk 窗口内 5 提交（HEAD 480b5fb，#2857 PSA Baseline v1.35 强制、#2890 移除 local 沙箱支持无 opt-out，均为破坏性；#2855 OAuth scope 不得作 model group；#2735 inspect forks 改 release wheels）；Harbor 窗口内 1 提交（CI 配置读取分支调整；dev 构建 0.25.1.dev202611100641 非稳定发布；窗口后首提交 #4127 opencode-v2 集成，2026-11-10T17:11:15Z，为下轮起始 HEAD 4d2dbf7）；ADK 零提交（HEAD efc7717）；Agents SDK 零提交（Python HEAD d689616 / JS HEAD 7589b9c；稳定版仍 0.24.0/0.21.0；稳定化时间表仍无）；MAF 窗口内 17 提交（HEAD fe87fdd，#10066 Hyperlight 取消透传 + 执行上限，破坏性，依赖未发布 SDK 包；#10217 Foundry 身份 API 澄清；仍 1.25.0/1.28.0；间隔官方解释仍缺）；LangChain JS 线拒答标准化（#12120 标准 `refusal` 内容块 + `ModelRefusalError`，Python #41582 的 JS 移植；#12121 中间件）；OTel PR #688 仍 Open 零动向（updated_at 仍 2026-11-06T00:59:26Z，HEAD 仍 e6658f1）；CLEO 仍仅 v1（11-02）；arXiv 11-10 批次 harness 论文因索引滞后未见（窗口内与全日查询均 0；cs.AI 全日亦 0；列表页最新仍周二 10 日 480 篇；下轮重查）；EvoGroup 2611.03868 摘要级补记；方法注记：arXiv 查询须 curl `--globoff`（未转义方括号报错 `bad range`，本轮复现并修复）。
- 剩余未核实缺口：Harbor #4127 opencode-v2 集成后续；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI PR #688 审定结果；Agents SDK 稳定化时间表；CLEO 同类最小化 harness 追踪；arXiv 11-10 批次 harness 论文重查；hawk 480b5fb / Harbor 4d2dbf7 为下轮起始 HEAD。

## 2026-11-11 凌晨轮（版本 0.25.0；窗口 2026-11-11 01:09 → 04:09 CST）

- 本轨增量（H1–H6）：窗口内新稳定发布三件——LangChain 拒答标准化（Python `langchain` 1.4.10 #41579/#41689/#41585，PyPI 2026-11-10T18:37:17/19Z；`@langchain/core` 1.3.7 #12120，npm 18:44:00Z；JS `langchain` 1.6.4 #12121，npm 18:43:06Z；#12122 provider 侧移植）；Harbor #4127 已合并（`merged_at` 2026-11-10T17:11:16Z，22 文件，`opencode_v2.py` +463 行，测试 +1,531 行，支持 session resume 与 `--bridge acp`；尚未进入任何 PyPI 构建，最新 dev 仍 `0.25.1.dev202611100641`；稳定版仍 0.25.0）；inspect-ai 新增 3 个模型信息条目（Gemini 4 Carbon、Muse Ember 1.0、Muse Wonder 1.0，日期为 PR 正文所引、未独立核实；仍 0.3.289，6 提交 HEAD `d168467`）；hawk HEAD `f56ff7a`（窗口内 10 提交，#2361 全锁升级附带发现 FastAPI 0.143 自带原生 OTel 追踪致 span 重复，hawk 以共享 `FASTAPI_TELEMETRY_OFF` 排除；无破坏性变更）；Harbor 沙箱注记（#4095 Daytona 后端无限期泄漏沙箱，`auto_stop_interval_mins` 默认 0，进程死亡未调 `stop()` 时永久 STARTED 占配额；#4196 aca-sandbox egress 策略仅标题，细节未核实）；复核戳：Agents SDK 两线零提交（alpha 仍 py alpha.7 / js alpha.2）、ADK 10 提交 HEAD `dc7d743` 仍 2.14.0、MAF 1 提交 HEAD `beba921`（#10244 A2A 空脚手架）仍 1.25.0/1.28.0、OTel PR #688 仍 Open（`updated_at` 仍 2026-11-06T00:59:26Z）、CLEO 仍 v1、arXiv 11-10 批次仍因滞后不可见，下轮重查。
- 剩余未核实缺口：Harbor 0.25.1（#4127 之后首个稳定版）；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI PR #688 审定结果；Agents SDK 稳定化时间表；CLEO 同类最小化 harness 追踪；arXiv 11-10 批次 harness 论文重查；inspect-ai 0.3.290；hawk f56ff7a / Harbor 500af91 为下轮起始 HEAD。

## 2026-11-11 07:09 轮（版本 0.26.0；窗口 2026-11-11 04:09 → 07:09 CST）

- 本轨增量（H1–H5）：窗口内新稳定发布为零。hawk HEAD `5148db3`（窗口内 20 提交，活动翻倍：平台事件可观测性簇 9 提交、任务镜像保护 #2444/#2480、沙箱命名空间隔离修复 #2900，`-s` 结尾任务 id 将被拒，有破坏潜力但无 `!` 标记）；inspect-ai 仍 0.3.289（5 提交，#6444 sentinel sidecar 签名调用为唯一 feat，凭证处理收紧）；ADK 仍 2.14.0（5 提交，全部小修复）；OTel GenAI PR #688 仍 Open（窗口内零动向；窗口关闭后 2–4 分钟 head 移动 `e69f0e7` → `eca7862` + 两条评审回复，下轮复核合并结果）；复核戳：Harbor 窗口内零提交，HEAD 仍 `500af91`，稳定版仍 0.25.0，dev 构建最新仍 0.25.1.dev202611100641，#4127 opencode-v2 已合并但仍未进入任何 PyPI 构建；MAF 窗口内零提交，HEAD 仍 `beba921`，Python 仍 1.25.0 / .NET 仍 1.28.0，1.25.0 tag→PyPI 约 23.6h 间隔官方解释仍缺；Agents SDK 两线零提交，alpha 仍 py alpha.7 / js alpha.2，稳定化时间表仍无官方说法；LangChain 四仓窗口内均为零提交，拒答标准化三件发版后无新版，langchain-core 仍 1.7.7，无 1.7.8；CLEO 仍仅 v1（arXiv:2611.03434v1）；arXiv 11-10 批次 harness 论文连续第二轮因索引/公告滞后不可见（全天 `abs:harness` 总数 0，且去掉限定后全天 cs.AI 总数亦为 0；清单页最新组仍为 Tue 10 Nov 480 条）——下轮必须重查；AgentTriggered 已冻结、无新证据。
- 剩余未核实缺口：Harbor 0.25.1（#4127 之后首个稳定版）；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI PR #688 合并结果（窗口后动向）；Agents SDK 稳定化时间表；CLEO 同类最小化 harness 追踪；arXiv 11-10 批次 harness 论文重查；inspect-ai 0.3.290；hawk 5148db3 / Harbor 500af91 为下轮起始 HEAD。

## 2026-11-11 10:09 轮（版本 0.27.0；窗口 2026-11-11 07:09 → 10:09 CST）

- 本轨增量（H1–H5）：窗口内新稳定发布为零。OTel GenAI PR #688 窗口内评审动向（aabmass 两条评审回复 23:11:43Z/23:11:45Z + 议题评论 23:11:47Z，同意按原样落地 #688，#53 与 util-genai 修复另起 PR；force-push 23:12:10Z，head 仍 eca7862；仍 open 未合并；main HEAD 仍 e6658f1）。**Harbor 仓库已迁 METR/harbor → harbor-framework/harbor**（旧址现 404，compare identical 验证为同一延续仓库；自验 curl -L：404 / 200）；窗口内 0 提交，HEAD 仍 500af91；0.25.1 稳定版未发布；PyPI 最新 dev 仍 0.25.1.dev202611100641。hawk HEAD f92e435（窗口内 1 提交，#2870）；inspect-ai 67873ea（窗口内 1 提交，#6469 测试-only），0.3.290 未发布；ADK c0bcdaa（窗口内 5 提交，全部 fix/docs），仍 2.14.0；LangChain/LangGraph 两语言线窗口内无新版无提交（langchain-core 仍 1.7.7）。MAF 1.25.0 tag→PyPI 约 23.6h 间隔算术核实（tag 提交 7bca1df 2026-11-06T16:05:38Z → PyPI 上传 2026-11-07T15:42:23Z = 23h36m；官方解释仍缺）；窗口内 0 提交，Python 仍 1.25.0 / .NET 仍 1.28.0。Agents SDK 两线零提交（alpha 仍 py alpha.7 / js alpha.2）。arXiv 11-10/11-11 harness 论文首次完整枚举（11-10 批次 `all:harness` 25 篇命中中实质相关 14 篇 + InfraRSI + 交叉 REINS/JoMoHa/2611.11342 + 替换稿 3 篇；11-11 批次 6 篇；列表页滞后实录：02:20/02:40 UTC 两次抓取 cs.AI/new 仍显示 11-10 列表——先前两轮不可见的原因正在于此）。CLEO 仍仅 v1；AutoGen HEAD 仍 d07c667a。
- 剩余未核实缺口：Harbor 0.25.1（#4127 之后首个稳定版，仓库新址 harbor-framework/harbor）；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI PR #688 合并结果；Agents SDK 稳定化时间表；inspect-ai 0.3.290；下轮起始 HEAD：hawk f92e435、Harbor 500af91、inspect-ai 67873ea、MAF beba921、agents-py d689616、agents-js 7589b9c、ADK c0bcdaa、AutoGen d07c667a、OTel PR head eca7862。

## 2026-11-11 13:09 轮（版本 0.28.0；窗口 2026-11-11 10:09 → 13:09 CST）

- 本轨增量（H1–H6）：窗口内新稳定发布为零。Harbor 0.25.1 仍未发布（tag 404，0 提交，HEAD 仍 500af91；PyPI dev 仍 0.25.1.dev202611100641）。OTel GenAI PR #688 仍 open（head eca7862，main HEAD e6658f1，窗口内零动向）。hawk HEAD d83f988（窗口内 2 提交：#2906 glovebox 升级、#2432 测试瘦身）；inspect-ai 0.3.290 未发布（0 提交，HEAD 67873ea）；LangGraph py HEAD 5ea88ec（4 条 Dependabot 提交，补上轮缺失 HEAD）；langgraphjs HEAD f404576（1 条依赖修复）；LangChain py 0 提交；ADK 仍 2.14.0（0 提交）；AutoGen 0 提交（HEAD d07c667a）；CLEO 仍仅 v1。MAF 仍 1.25.0/1.28.0（0 提交，HEAD beba921；23.6h 间隔官方解释仍缺）。OTel GenAI span 格式未收敛：未合并 PR 簇 #516（fetch_response span token 用量）、#607（per-class 成本拆分）、#676/#653（工具调用结果结构之争）、#700（Ollama 约定）+ 议题 #703/#697/#695（token 归属）、#693（MACP）、#682（嵌套去重）；Harbor ATIF 仍 v1.8；METR 无格式变更。arXiv 11-11 批次 harness 论文完整枚举 20 篇（直接相关 12 篇：2611.13144、2611.13336 Veritas、2611.13183、2611.13246、2611.13254、2611.13406、2611.13524 RSI Tax、2611.13530 UniWeaver、2611.13925 OpenRE-Bench、2611.13960 ActiveSecHarness、2611.14493 评估框架比较、2611.14505 Continual-ARC；边缘 4 篇；无关 4 篇；替换稿 4 篇）——**订正上轮「6 篇」不完整**（默认 `max_results` 截断所致）。
- 纠正：arXiv 公告时刻——自验亲自复核官方 Availability 页裁决：公告为 20:00 ET（11 月 = 次日 01:00 UTC），列表页约午夜 ET 刷新；Harness 腿「00:00 UTC 正确」换算驳回（仅夏令时 EDT 下成立）；10:09 轮 delta S5「entered visibility with the 2026-11-11 00:00 UTC announcement」措辞不精确，已在 skill-authoring.md 本轮增量首条纠正。方法规则：范围查询须显式 `max_results≥50` 并核对 totalResults，不得依赖默认截断。
- 剩余未核实缺口：Harbor 0.25.1（#4127 之后首个稳定版，仓库新址 harbor-framework/harbor）；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI span 格式 PR 簇审定结果（#516/#607/#676/#653/#700、#703/#697/#695/#693/#682）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；arXiv 11-12 批次补查；下轮起始 HEAD：hawk d83f988、Harbor 500af91、inspect-ai 67873ea、LangGraph py 5ea88ec、langgraphjs f404576、MAF beba921、ADK c0bcdaa、AutoGen d07c667a、OTel PR head eca7862。

## 2026-11-11 16:09 轮（版本 0.29.0；窗口 2026-11-11 13:09 → 16:09 CST）

- 本轨增量（H1–H4）：窗口内新稳定发布为零，**12 仓全部 0 提交**。Harbor 0.25.1 仍未发布（tag 404，0 提交，HEAD 仍 500af91；PyPI dev 仍 0.25.1.dev202611100641）。OTel GenAI span 格式 PR 簇六项（#516/#607/#676/#653/#700/#688）+ 五议题（#703/#697/#695/#693/#682）全部仍 open、head 未动；main HEAD 仍 e6658f1，自 11-06 零合并。inspect-ai 0.3.290 未发布（HEAD 67873ea）；hawk HEAD d83f988；MAF 仍 1.25.0/1.28.0（HEAD beba921；23.6h 间隔解释仍缺，`python-1.25.0` 为轻量标签）；Agents SDK py/js 均 0 提交（HEAD d689616 / 7589b9c；alpha 仍 py alpha.7 / js alpha.2；稳定化时间表仍未见）；ADK 仍 2.14.0（HEAD c0bcdaa）；LangChain py HEAD c6ec96a（compare 端点偶发 404，以 ls-remote 为准）、langchain-core 仍 1.7.7、langgraph 仍 1.2.20、npm @langchain/core 1.3.7 / @langchain/langgraph 1.7.3；AutoGen HEAD d07c667a；CLEO 仍仅 v1。arXiv 11-11 批次 24 篇全部无窗口内修订（id_list 复核）；11-12 批次尚未公告（预期 11-13 01:00 UTC），下轮完整枚举。
- 纠正：Harness 腿「11-12 批次公告时刻 2026-11-12 00:00 UTC」不成立（执行者亲自复核官方博客 2026-11-06 原文：11-12 周四 20:00 ET = 11-13 01:00 UTC）；该腿所写投稿窗口「2026-11-10 19:00 UTC → 2026-11-11 19:00 UTC」亦不成立（官方窗口止于 11-12 14:00 ET = 19:00 UTC 11-12）。方法注记：共享轮次目录下通用名 arXiv 文件存在被他腿同名覆盖的风险，下轮 id_list 查询一律显式 `max_results≥50` 并用腿专属前缀文件名。
- 剩余未核实缺口：Harbor 0.25.1（#4127 之后首个稳定版，仓库新址 harbor-framework/harbor）；MAF 1.25.0 间隔官方解释；AgentTriggered 精确上线日期（已冻结）；OTel GenAI span 格式 PR 簇审定结果（#516/#607/#676/#653/#700、#703/#697/#695/#693/#682）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；arXiv 11-12 批次补查；下轮起始 HEAD：hawk d83f988、Harbor 500af91、inspect-ai 67873ea、LangGraph py 5ea88ec、langgraphjs f404576、MAF beba921、ADK c0bcdaa、AutoGen d07c667a、OTel PR head eca7862、Agents SDK py d689616 / js 7589b9c、LangChain py c6ec96a。

## 2026-11-11 19:09 轮（版本 0.30.0；窗口 2026-11-11 16:09 → 19:09 CST）

- 本轨增量（H1–H4）：窗口内新稳定发布为零。**inspect-ai 组织更名 UKGovernmentBE → UKGovernmentBEIS**（旧址双路 404，同一仓库延续，HEAD 仍 67873ea；新址 `pushed_at` 窗口内移动，对象未查明）。hawk #2859（compose 不再插值 runner 密钥）、MAF #10152（[BREAKING] Hyperlight guest 中断 + 有限默认预算，Python 线；发布载体未知）、ADK PyPI 发布迁至 OSS Exit Gate。Harbor 0.25.1 仍未发布；OTel GenAI PR 簇六项 + 五议题全部仍 open，main HEAD e6658f1 自 11-06 零合并；Agents SDK 仍 py alpha.7 / js alpha.2；inspect-ai 0.3.290 未发布；MAF #10151 closed 0 评论。arXiv 11-11 批次 24 篇全部无窗口内修订；11-12 批次尚未公告；MAF 边界提交 e19aca2（属上轮窗口，本轮首次观测，备查）。
- 纠正：无（本轨腿文件无窗口归属错误；inspect-ai 更名时间未查明，不入技能口径）。
- 剩余未核实缺口：Harbor 0.25.1（#4127 之后首个稳定版，仓库新址 harbor-framework/harbor）；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果；Agents SDK 稳定化时间表；inspect-ai 0.3.290 与更名时间；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk d3383bf、Harbor 500af91、inspect-ai（新组织）67873ea、LangGraph py 5ea88ec、langgraphjs f404576、MAF 7e2ba54、ADK 9687bae、AutoGen d07c667a、Agents SDK py d689616 / js 7589b9c、LangChain py c6ec96a、OTel main e6658f1 / PR #688 head eca7862。

## 2026-11-11 22:09 轮（版本 0.31.0；窗口 2026-11-11 19:09 → 22:09 CST）

- 本轮增量：Harbor 任务格式一手（分离验证器模式、格式与运行器解耦、ATIF v1.8 八个音频内容类型）；METR Task Standard 现行版 0.5.0 + README 冻结声明（改用 Inspect）；MAF #10152 已合并（2026-11-11T09:17:59Z，窗口前）但发布载体未定；OTel GenAI 仅两项窗口活动（#688 评论、#516 上游合并）；Agents SDK py 一窗口提交（Dependabot），alpha 仍 py alpha.7 / js alpha.2，alpha.7 发布说明无稳定化时间表；inspect-ai 0.3.290 未发布（更名时间仍未定）；arXiv 11-11 批次 24 篇零窗口内修订；CLEO 仍 v1。
- 纠正：本轮 Harness 轨无旧说法推翻。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果；Agents SDK 稳定化时间表；inspect-ai 0.3.290 与更名时间；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk d3383bf、Harbor 500af91、inspect-ai 67873ea、LangGraph py 5ea88ec、langgraphjs f404576、langchainjs 267ebca、MAF 6d79abf、ADK 9687bae、AutoGen d07c667a、Agents SDK py 64fbde5 / js 7589b9c、LangChain py c6ec96a、OTel main e6658f1 / PR #688 head eca7862 / PR #516 head f96d095。

## 2026-11-12 01:09 轮小节

- 本轮增量：LangChain Python 1.5.0 / core 1.8.0 / typesafe 0.1.0 窗口内发布（决策模型 + 模型路由/自动模式中间件，beta）；MAF #10152 前置依赖已解除（hyperlight-sandbox v0.9.0 已发布 + 依赖下限 >=0.9.0，载体仍未定）；MAF #10151 分阶段门控旁证（23.6 小时间隔仍无官方解释）；OTel GenAI PR 簇六项仍 open + 新增相邻 PR #702/#696/#694/#689；Harbor dev 构建 0.25.1.dev202611110847；ATIF v1.8 LLMJudge 媒体转发细节；本轮破坏性变更 2 项（MAF #10266、LangChain Python 3.10 移除）；inspect-ai 0.3.290 未发布 + AISI 更名警示。
- 复核戳：Agents SDK 稳定化时间表仍无官方表述；hawk/ADK/langgraph py/langchainjs/AutoGen 零提交；langgraphjs 2 提交（StreamOrchestrator）；arXiv 11-12 批次尚未公告（公告 2026-11-13 01:00 UTC）；11-11 批次 12 篇直接相关论文全部仍 v1；CLEO 仍 v1；METR Task Standard 仍冻结 0.5.0；OTel GenAI 议题 #703/#697/#695/#693/#682 与 #1075 本轮未复核（API 额度耗尽），下轮须重核。
- 纠正：MAF #10152「前置未发布」部分已过时（见本轮增量）；LangChain 默认分支为 master（非 main）。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果（议题重核）；Agents SDK 稳定化时间表；inspect-ai 0.3.290 与组织更名时间；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk d3383bf、Harbor 500af91、inspect-ai 7f5a2b5、LangGraph py 5ea88ec、langgraphjs 737e53b、langchainjs 267ebca、MAF 22b5e75、ADK 9687bae、AutoGen d07c667a、Agents SDK py 64fbde5 / js 7589b9c、LangChain py 771d11b（master 分支）、OTel main e6658f1 / PR #688 head eca7862 / PR #516 head f96d095。

## 2026-11-12 04:09 轮小节

- 本轮增量：OTel #688 落地共识补记（aabmass 2026-11-10 同意按原样落地，util-genai 修复拆为 #1075）+ #1075 窗口内 2 提交推进；Harbor 分离验证器全文深读（两种配置、5 行解析表、镜像优先级链、artifact 消毒）；METR Task Standard 0.5.0 口径澄清（README 声明值，非 tag）；langchainjs 决策模型移植合并（#12178/#12183，尚无 JS 发布）；本轮破坏性变更 1 项（hawk #2893，Kubernetes Baseline v1.35 强制 runner 命名空间）；inspect-ai 0.3.290 未发布（3 修复提交）。
- 复核戳：六核心 PR 全仍 open，main HEAD 仍 e6658f1，零合并；五议题本轮逐项重核（上轮额度耗尽未决项闭合），全 open 零活动；待决 15 项；Harbor 0.25.1 未发布；MAF 零动向（#10152 载体仍未定）；Agents SDK 无新 alpha/稳定/时间表（README grep 0 命中为新的否定证据）；LangGraph py 零提交；ADK 1 提交；组织更名时间仍未定。
- 纠正：OTel *_ref 属性 PR 编号 #702→#704（#702 实为 conformance runner pin）；langchain-core 1.7.7“无 1.7.7+”旧句过时（1.7.7 发布已于 2026-11-10 轮登记于正本，腿“补漏”说法不成立）。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果（#688/#516 合并时间、#1075 落地）；Agents SDK 稳定化时间表；inspect-ai 0.3.290 与组织更名时间；LangChain 1.5.0 路由信号/自动触发条件源码深读；langchain-core 1.7.7 load() 深度上限阈值；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk b0a6958、Harbor 500af91、inspect-ai 36294bb、LangGraph py 5ea88ec、langgraphjs a1d9f2b、langchainjs e220aee、MAF 22b5e75、ADK 59b0dd6、AutoGen d07c667a、Agents SDK py 64fbde5 / js 7589b9c、LangChain py 771d11b（master 分支）、OTel main e6658f1 / PR #688 head eca7862 / PR #516 head f96d095 / PR #1075 head e192f66、hyperlight-sandbox fa20a98、task-standard 83ad6fd。

## 2026-11-12 07:09 轮（版本 0.34.0；窗口 2026-11-12 04:09 → 07:09 CST）

- 本轨增量（H1–H4）：LangChain JS 线窗口内三件发布（langchain@1.6.5、@langchain/core@1.4.0、@langchain/typesafe@0.1.0，2026-11-11T22:02–22:03Z；BaseDecisionModel 四类有型问题 Noul/Choice/Score/Box；modelRouterMiddleware/autoModeMiddleware；Zod GHSA-mrhx-c7xc-m3w5 钉版；typesafe 0.1.0 带破坏性标记；同批 8 个 patch 包）；OTel #1075 窗口内获批（aabmass APPROVED 2026-11-11T22:45:59Z）仍未合并，六项 PR 全 open，待决 15 项不变；Agent Harness Awareness 编号确认 arXiv:2611.05391（仍 v1；ID 前缀 2611 与「Submitted on 13 Sep 2026」表面不一致，arXiv 原始字段）；复核戳：inspect-ai 0.3.290 未发布（窗口内 2 修复提交 #6511/#6489，HEAD 88eef6a）；hawk 窗口内 4 提交无破坏性（682b761 窗口外）；Harbor 仍 0.25.0（CHANGELOG 逐字节相同，ATIF 仍 v1.8）；MAF 0 提交（#10152 载体仍未定、#10151 仍仅一句旁证）；Agents SDK 两线 0 提交（稳定化时间表 README grep 仍 0 命中）；LangChain py / LangGraph 两线 0 提交；ADK 2 提交仍 2.14.0；arXiv 11-12 批次未公告，11-11 批次 12 篇 + CLEO + Teaching the Harness 全部仍 v1。
- 纠正：无（本轮无推翻项；arXiv 公告时刻勘误见记忆轨 D-M04）。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果（#688/#1075 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290 与组织更名时间；LangChain 决策模型/路由中间件源码深读（判定信号与 auto 触发条件）；Agent Harness Awareness ID/日期表面不一致成因；hawk 682b761（窗口外提交，下轮起点）；arXiv 11-12 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk 682b761、Harbor d10223a、inspect-ai 88eef6a、LangGraph py 5ea88ec、langgraphjs a1d9f2b、langchainjs 609cc46、MAF 22b5e75、ADK 9620b42、AutoGen d07c667a、Agents SDK py 64fbde5 / js 7589b9c、LangChain py 771d11b（master 分支）、OTel main e6658f1 / PR #688 head eca7862 / PR #516 head f96d095 / #1075 head e192f66（已获 aabmass 批准，仍未合并）、hyperlight-sandbox fa20a98（本轮未重查）、task-standard 83ad6fd。

## 2026-11-12 10:09 轮

- 本轮增量（D-H01–D-H04，落入 skill/references/agent-harness.md）：LangChain 决策模型/路由中间件源码深读（Python 1.5.0/core 1.8.0、JS 1.6.5/core 1.4.0 一手解包；modelRouterMiddleware 每轮恰好分类一次、无阈值门、置信度只记录；autoModeMiddleware 阈值 0.5 硬编码不可配置、拦截不执行不请求人工批准；JS 显式 assertAnswered vs Python 依赖子类 pydantic 校验——闭合"路由信号/自动触发条件未展开"缺口）；OTel #1075 窗口内推进（新提交 e092a5e + 3 条评审，仍未合并；六项核心 PR 全 open；待决 15 项不变）；Agent Harness Awareness 仍 v1（published = updated = 2026-09-13T15:01:55Z 与 ID 前缀 2611 表面不一致，成因未核实，只记原始字段）；复核戳：Harbor 0.25.1/inspect-ai 0.3.290 未发布、MAF 0 提交（#10152 载体仍未定）、Agents SDK 0 提交、hawk 3 提交（推理 infra）、LangGraph py 1 提交、arXiv 11-11 批次不可见、14 篇论文全 v1。
- 纠正：无（本轮无推翻项；LangChain 决策模型缺口由"未展开"改为"已闭合"）。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果（#688/#1075 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290 与组织更名时间；Agent Harness Awareness ID/日期表面不一致成因；arXiv 11-11 批次补查。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk 90b84e2、Harbor d10223a、inspect-ai 88eef6a、LangGraph py b4cb0f1、langgraphjs a1d9f2b、langchainjs 609cc46、MAF 22b5e75、ADK 9620b42、AutoGen d07c667a、Agents SDK py 64fbde5 / js 7589b9c、LangChain py 771d11b（master 分支）、OTel main e6658f1 / PR #688 head eca7862 / PR #1075 head e092a5e、hyperlight-sandbox fa20a98、task-standard 83ad6fd。

## 2026-11-12 16:09 轮（13:09 轮已完成腿）

- 本轮增量（D-H01–D-H04，落入 skill/references/agent-harness.md）：OTel GenAI #1075 已合并（merged_at 2026-11-12T05:12:36Z，合并提交 b164937；窗口内新提交 0bf06a0 + 三条评审；#688 仍 open；待决总数仍 15 项）；inspect-ai 组织"更名"说法撤回（UKGovernmentBEIS 自 2020-12-23 起即为组织名，UKGovernmentBE 零快照、现 404，系路径笔误）；Harbor 窗口内 10 提交（Serve/jobs API 簇 + CI 迁移 Blacksmith；0.25.1 仍未发布）；arXiv 11-11 批次 harness 枚举（已登记 14 篇全部仍 v1；8 篇新候选摘要级登记；直接相关：2606.11337 SciConHarness、2609.36630 Agent Distillation 综述 v2）。
- 纠正：inspect-ai 组织"更名 UKGovernmentBE → UKGovernmentBEIS"撤回——无更名事件；"更名时间未决"缺口关闭。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 PR 簇审定结果（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；Agent Harness Awareness ID/日期表面不一致成因（arXiv API 本轮 429 未重取原始字段）。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式）。
- 下轮起始 HEAD：hawk 90b84e2、Harbor 7c8c1f6、inspect-ai 88eef6a、LangGraph py b4cb0f1、langgraphjs a1d9f2b、langchainjs 609cc46、MAF 22b5e75、ADK 9620b42、AutoGen d07c667a、Agents SDK py 64fbde5 / js 7589b9c、LangChain py 771d11b（master 分支）、OTel main e6658f1 / PR #688 head eca7862 / PR #1075 已合并（py-genai main b164937）、hyperlight-sandbox a4f4a77、task-standard 83ad6fd。

## 2026-11-12 19:09 轮（第 37 轮）

- 本轮增量（D-H01–D-H04，落入 skill/references/agent-harness.md）：hawk v4.1.0 窗口内发布（2026-11-12T09:13:17Z，"cut as 4.1.0, not 5.0.0"；四轨窗口内唯一新稳定版）；OTel GenAI 待决 15→16 订正（#704 为 `*_ref` 属性 PR，open，head 3317f26；#688 仍 open，head eca7862；#1075 仓映射钉死为 opentelemetry-python-genai）；Harbor vs METR vs OTel 格式对照（Harbor 声明式 + schema_version 默认 "1.4"；METR 0.5.0 已冻结转向 Inspect；OTel 13 种 span 类型；现状 = 13 种已合并 + 11 项 PR + 5 项议题）；Harbor 发现 API 晚到站（5 提交为上轮窗口晚到站，Serve/MCP/SDK 发现 API；0.25.1 仍未发布；inspect-ai 0.3.290 未发布；MAF #10152 载体仍未定；Agents SDK 无 alpha.8/alpha.3；ADK 1 提交，仍 2.14.0；AHA 仍 v1，2611 前缀 vs 九月日期表面不一致仍未决）。
- 纠正：OTel GenAI 待决"15 项"→"16 项"（正文内改写）；Harbor 5 提交为上轮窗口晚到站（不计入本轮窗口）。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；Harbor schema_version 适配机制（METR 承诺 per-standard_version 适配器，Harbor 未见对应机制）；AHA 2611 前缀 vs 九月日期表面不一致成因。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：Harbor schema_version 适配机制深查）。
- 下轮起始 HEAD：hawk（v4.1.0，2026-11-12T09:13:17Z）、Harbor feea734、inspect-ai 88eef6a、OTel semconv main e6658f1、opentelemetry-python-genai（#1075 已合并 b164937）。

## 2026-11-12 22:09 轮（第 38 轮）

- 本轮增量（D-H01–D-H04，落入 skill/references/agent-harness.md）：OTel GenAI 16 项待决全部仍 open、窗口内零动向（#688 head eca7862、#704 head 3317f26；semconv main 仍 e6658f1）；Harbor 0.25.1 仍未发布（窗口内 0 提交）；MAF #10152 载体仍未定（窗口内 4 提交，全 Python）；Agents SDK 无新 alpha（窗口内 0 提交）；inspect-ai 0.3.290 未发布（窗口内 1 提交 df46cc1）；hawk 仍 v4.1.0（窗口内 1 提交 f18befc）；LangChain master 窗口内 1 提交（980e4a8，14:08:03Z）；langgraphjs 窗口内 2 提交；arXiv 8 篇新候选版本钉死（4 v1、3 v2、1 v3，修订日均为 2026-11-10）；AHA 原始字段重取（published = updated = 2026-09-13，v1）。
- 纠正：Harbor/METR 版本字段“适配承诺”说法不成立（正文内改写，L193 末句作废）：两边同构为声明值 + 无适配——Harbor schema_version 仅声明值（config.py:854 普通 `str` 默认 "1.4"，无枚举/校验/版本分支；handle_version_rename 仅改名垫片；CLI 打包无条件盖戳 "1.4"；运行器 Task 类 0 命中；任务模型包内代码搜索 total_count 1）；METR STANDARD.md Versioning 仅一句 SemVer，adapter/adapt/migrat 0 命中，template.py:16–18 仅要求声明，驱动从不读取该字段；另 Harbor `server/schema.py` 代码搜索命中为索引误报（HEAD 全文 grep 0 命中），不得引用。
- 剩余未核实缺口：Harbor 0.25.1；MAF 1.25.0 间隔官方解释与 #10152 发布载体；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；8 篇修订论文的相关度裁定（本轮仅版本钉死）。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：OTel GenAI 待决项合并动向与 Harbor 0.25.1）。
- 下轮起始 HEAD：Harbor feea734、inspect-ai df46cc1、OTel semconv main e6658f1、opentelemetry-python-genai b164937、LangChain master 980e4a8。

## 2026-11-13 01:09 轮（第 39 轮）

- 本轮增量（D-H01–D-H06，落入 skill/references/agent-harness.md）：langgraph `cli==0.4.43` 窗口内发布（2026-11-12T16:20:16Z；PyPI 上传 16:19:59Z；核心 langgraph 仍 1.2.20；本轮唯一窗口内新稳定发布）；MAF `python-1.26.0` 标签已打且含 #10152（载体已定），但 Release 对象仍 404、PyPI 仍 1.25.0，不计入已发布（标签创建时间仅界定在 16:59:37Z 之后、17:16 UTC 之前，含 7 分钟窗口外缝隙）；OTel GenAI 16 项待决全部仍 open、窗口内零动向；Harbor 0.25.1 仍未发布（窗口内 0 提交）；inspect-ai 0.3.290 未发布（窗口内 7 提交，两处密钥防泄漏修复 + 沙箱 bridge 身份作用域；918246a 为窗口外）；hawk 仍 v4.1.0（0 提交）；Agents SDK 无新 alpha；LangChain master 0 提交；langgraphjs 5 提交（SDK 流式引擎）；上轮 8 篇候选论文相关度裁定完成（相关 3 / 相邻 3 / 无关 2）；AHA 成因仍未决（仅一条弱文本线索）。
- 纠正：无（本轮无推翻项；MAF #10152 载体由「未定」推进为「python-1.26.0 标签已定、发布未完成」）。
- 剩余未核实缺口：Harbor 0.25.1；MAF python-1.26.0 Release 对象与 PyPI 上架（下轮复核）、1.25.0 间隔官方解释（#10151 仍 0 评论）；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；arXiv 11-12 批次枚举。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：Harbor feea734、inspect-ai 918246a（含窗口外提交）、OTel semconv main e6658f1、opentelemetry-python-genai b164937、MAF 4f2d146（python-1.26.0 标签指向此提交）、hawk f18befc、LangChain master 980e4a8、langgraph 595223d、langgraphjs c354854。

## 2026-11-13 04:09 轮（第 40 轮）

- 本轮增量（D-H01–D-H04、D-H06，落入 skill/references/agent-harness.md；D-H05 为 track-only 两项纠正）：langgraph 核心 1.2.21 窗口内发布（PyPI 2026-11-12T18:06:30Z、GitHub Release 18:06:43Z；#9615/#9614 子图命名空间修复）；langgraphjs 8 包经 #3322 集中升版（`@langchain/langgraph` 1.7.4、sdk 1.16.0、api/cli/ui 各 1.7.2、checkpoint-postgres 1.1.3、checkpoint-validation 1.2.1、`@langchain/react` 1.2.14；`@langchain/langgraph-sdk-react` 不存在，实为 `@langchain/react`）；MAF python-1.26.0 进行中，不计入已发布（PyPI 28/28 已上架，Release 对象截至 20:23 UTC 仍 404；官方清单 28 包）；复核戳：OTel GenAI 16 项待决全部 open、零动向；Harbor 0.25.1 仍未发布（ATIF 仍 v1.8）；inspect-ai 0.3.290 未发布（窗口内 4 提交，含 `fix!`）；Agents SDK 无新 alpha；hawk 仍 v4.1.0；arXiv 22 篇版本全未变；AHA 仍仅 v1。
- 纠正（D-H05，track-only）：22:09 轮「GitHub GHSA 端点查 LiteLLM CVE」一节中「五条 first_patched_version 全为 null」不成立（ra 腿取自仓库级 security-advisories 端点；全局 advisories 端点仍填 1.100.4 / 1.101.3 / 1.102.2 / 1.103.1 / 1.104.0rc2）；01:09 轮「Harbor dev 构建上传 06:58 UTC」纠正为 14:58:16Z（版本号后缀误读）。
- 剩余未核实缺口：Harbor 0.25.1；MAF python-1.26.0 Release 对象与 PyPI 上架（下轮复核）、1.25.0 间隔官方解释（#10151 仍 0 评论）；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；arXiv 11-12 批次枚举。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：Harbor d774888（窗口外 20:12:09Z，ATIF reasoningText→reasoning_content 映射）、inspect-ai 26f6d05（窗口外 20:10:36Z）、adk-python f9d948c（窗口外 20:17:57Z）；OTel semconv main e6658f1、opentelemetry-python-genai b164937、MAF 4f2d146、hawk f18befc、LangChain master 980e4a8、langgraphjs c354854 均未变；langgraph HEAD 待下轮复核（1.2.21 已发布，上轮值 595223d）。

## 2026-11-13 07:09 轮（第 41 轮）

- 本轮增量（D-H01–D-H05，落入 skill/references/agent-harness.md）：google-adk v2.14.1 窗口内发布（2026-11-12T21:30:11Z，单项 tracing 修复：关闭 Agent Engine 上 FastAPI 原生 tracing；另有 live 方向 3 提交）；MAF python-1.26.0 仍未完成发布（Release 对象终查仍 404；PyPI 仍 1.26.0；.NET 仍 1.28.0；#10151 仍 0 评论）；复核戳：Harbor 0.25.1 仍未发布（窗口内 10 提交，配置快照/版本化三连 + serve API 三连修复；ATIF 仍 v1.8）；inspect-ai 0.3.290 未发布（窗口内 6 提交）；OTel GenAI 16 项待决全部 open、零动向；Agents SDK 无新 alpha、稳定化时间表仍无表述；hawk 仍 v4.1.0（窗口内 3 提交）；langgraphjs #3355（连续第五轮改动）、langchainjs #12214 各 1 提交；arXiv 22 篇版本全未变；AHA 仍仅 v1。
- 纠正：无（本轮无推翻项；MAF 1.26.0「发布进行中」继续）。
- 剩余未核实缺口：Harbor 0.25.1；MAF python-1.26.0 Release 对象与 PyPI 上架（下轮复核）、1.25.0 间隔官方解释（#10151 仍 0 评论）；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；arXiv 11-12 批次枚举。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：Harbor cc5c508、inspect-ai b4aa05b、hawk 3851917；OTel semconv main e6658f1、opentelemetry-python-genai b164937、MAF 4f2d146、LangChain master 980e4a8 均未变；langgraph/langgraphjs HEAD 待下轮复核（1.2.21 已发布；langgraphjs #3355 连续第五轮改动）。

## 2026-11-13 10:09 轮（第 42 轮）

- 本轮增量（D-H01–D-H05，落入 skill/references/agent-harness.md）：Agents SDK JS 0.22.0 窗口内发布（纯补丁：依赖刷新 + Zod 4.6.6；Python 仍 alpha.7）；MAF python-1.26.0 仍未完成发布（Release 对象终查仍 404；PyPI 仍 1.26.0；.NET 仍 1.28.0；1.26.0 预备 issue 编号纠正为 #10298；#10317 为开放 PR 而非 issue，其正文证明 1.26.0 标签已移动以纳入 Hyperlight 依赖下限修复 #10313 与发布标题指引 #10311）；复核戳：Harbor 0.25.1 仍未发布（窗口内 1 提交；ATIF 仍 v1.8）；inspect-ai 0.3.290 未发布（窗口内 1 提交）；OTel GenAI 16 项待决全部 open、零动向；ADK v2.14.1 后无新版；langgraph 11 提交性能批（#9628–#9639）、langgraphjs 12 提交（性能批 + #3356 SDK 修复，SDK 方向连续第六轮改动）、hawk 4 提交；langchainjs HEAD SHA 纠正（eed4c3250…）；AHA 仍仅 v1。
- 纠正：MAF 1.26.0 预备 issue「#10151」（实为 #10298）；#10317「为 issue」（实为开放 PR）；langchainjs HEAD SHA 漏印。
- 剩余未核实缺口：Harbor 0.25.1；MAF python-1.26.0 Release 对象与 PyPI 上架（下轮复核）、1.25.0 间隔官方解释（#10298 仍 0 评论）；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；arXiv 11-12 批次枚举。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：Harbor 本轮新增提交 3bf4cb3、inspect-ai 本轮新增提交 e4435ba；langgraph/langgraphjs/hawk 本轮 HEAD 未在 delta 中钉死，下轮复核时以 compare API 重取为准。

## 2026-11-13 13:09 轮（第 43 轮）

- 本轮增量（D-H01–D-H09，落入 skill/references/agent-harness.md）：窗口内新稳定发布 0 件；MAF python-1.26.0 仍未完成发布（Release 对象终查仍 404；PyPI 仍 1.26.0；.NET 仍 1.28.0；#10298 仍 1 评论；#10317 仍 open）；Harbor 0.25.1 仍未发布（窗口内 1 提交 #4277 pipefail 可移植修复）；inspect-ai 0.3.290 未发布；OTel 16 项待决全部 open、零动向；Agents SDK 两线无新版；langgraphjs #3363（SDK 方向连续第七轮改动）；22 篇版本复核窗口内修订 0；AHA 仍仅 v1；深挖格式口径窗口内无新变动。
- 纠正：2611.13028（GTAGym）由上轮所记 v1 纠正为 v2（revised 2026-11-11T11:55:08Z，摘要页一手；上轮 API 查询返回旧版，疑似 API 索引滞后，成因未核实；不属窗口内修订）；派发书前提「arXiv 11-12 批次已于 2026-11-13 01:00 UTC 公告」与实测不符（截至 05:15:13 UTC 列表页仍为 11-11 批次）。
- 剩余未核实缺口：Harbor 0.25.1；MAF python-1.26.0 Release 对象与 PyPI 上架（下轮复核）、1.25.0 间隔官方解释（#10298 仍 1 评论）；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；arXiv 11-12 批次枚举。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：Harbor 06d0e86、inspect-ai e4435ba、langgraphjs 82a7d07；OTel semconv main e6658f1、opentelemetry-python-genai b164937、MAF c7e50cc、hawk ba03f5f、LangChain master 980e4a8、langgraph 3183e90、langchainjs eed4c32 均未变。

## 2026-11-13 16:09 轮（第 44 轮）

- 本轮增量（D-H01–D-H06，落入 skill/references/agent-harness.md）：窗口内新稳定发布 0 件；MAF python-1.26.0 仍未完成发布（Release 对象终查仍 404；PyPI 仍 1.26.0；#10298 仍 1 评论；#10317 仍 open；44 包齐全性未核实）；Harbor 0.25.1 仍未发布（窗口内 2 提交 #2957/#2958 执行可靠性，hawk）；inspect-ai 0.3.290 未发布；OTel 16 项待决全部 open、零动向；Agents SDK JS 0.22.0 / Python alpha.7；AHA 仍仅 v1；22 篇版本复核窗口内修订 0；arXiv 11-12 批次 Harness 枚举完成（32 篇，EvoX 2611.15006、guardscope 2611.15223、ReA-Bench 2611.16493 为候选线索）。
- 纠正：无新推翻项（arXiv 11-12 批次枚举闭合，第三轮结转划除）。
- 剩余未核实缺口：Harbor 0.25.1；MAF python-1.26.0 Release 对象与 PyPI 上架（下轮复核）、1.25.0 间隔官方解释（#10298 仍 1 评论）；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；inspect-ai 0.3.290；AHA 2611 前缀 vs 九月日期表面不一致成因；EvoX/guardscope/ReA-Bench 候选线索全文深读。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1；三篇候选线索深读）。
- 下轮起始 HEAD：Harbor 06d0e86、inspect-ai e4435ba、langgraphjs 82a7d07；OTel semconv main e6658f1、opentelemetry-python-genai b164937、MAF c7e50cc、hawk ba03f5f、LangChain master 980e4a8、langgraph 3183e90、langchainjs eed4c32 均未变（13:09 轮 HEAD 未变，本轮未复测 HEAD）。

## 2026-11-13 19:09 轮（第 45 轮）

- 本轮增量（D-H01–D-H07，落入 skill/references/agent-harness.md）：**MAF python-1.26.0 发布完成**（Release 对象窗口内 08:28:16Z 出现，57 附件 = 28 包；纠正：28 包不是 44 包；release notes：`loop_return_final_only`、Hyperlight #10152 破坏性 30 秒预算、记忆 API 弃用至 2026-12-31、FIDES 钩子簇、Bedrock/Claude 缓存 token 计入）；MAF #10318 破坏性 shell 路径校验（标签后提交）、#10326 测试跳过；#10317 仍 open 但不再与「未完成」相互印证；#10298 仍 1 评论；Harbor 0.25.1 仍未发布（窗口内 0 提交，连续两轮）；inspect-ai 0.3.290 未发布；OTel 16 项待决全部 open、零动向；Agents SDK JS 0.22.0 / Python alpha.7；hawk #2854；langgraphjs/langchainjs 依赖 bump（SDK 方向仍第七轮）；**EvoX/guardscope/ReA-Bench 三篇候选线索全文深读完成**（均 v1，均未提新任务/trace 格式）；AHA 仍仅 v1。
- 纠正：MAF 1.26.0「44 包」→ 28 包；MAF 1.26.0「仍不计入已发布」→ 已发布；#10317 open 不再与「未完成」相互印证。
- 剩余未核实缺口：MAF #10318 发布载体与 #10317 合并与否；Harbor 0.25.1；inspect-ai 0.3.290；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；AHA 2611 前缀 vs 九月日期表面不一致成因。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：MAF 739a387、hawk 497d0f6、langgraphjs 749403f、langchainjs c266deb；Harbor 06d0e86、inspect-ai e4435ba、ADK 5ce1d0c、Agents SDK py 64fbde5/js 8aeac0e、langchain 980e4a8、langgraph 3183e90、AutoGen 8390985、OTel semconv e6658f1/py-genai b164937、METR task-standard 83ad6fd 均未变。

## 2026-11-13 22:09 轮（第 46 轮）

- 本轮增量（D-H01–D-H05，落入 skill/references/agent-harness.md）：窗口内新稳定发布 0 件，窗口内提交 4 项（ADK #7312 遥测凭据脱敏、hawk #2797、langgraphjs #3364 SDK 方向连续第八轮、langchainjs #11289）；**MAF #10317 窗口内草稿→待评审**（13:06:57Z + 6 次评审请求，仍未合并）、#10318 载体未定（无 python-1.27.0/dotnet-1.29.0 标签）、#10298 仍 1 评论；**OTel #516 窗口内获第二批准**（11:29:47Z，仍未合并）、16 项待决全 open；**MAF 1.26.0 后续文档**：FIDES 两份文档已在先（`b89eaeb` 2026-11-12T11:09:56Z）、记忆 API 弃用无独立迁移指南；Harbor 0.25.1 仍未发布（连续第三轮窗口内 0 提交）、inspect-ai 0.3.290 未发布、Agents SDK JS 0.22.0 / Python alpha.7、AHA 仍仅 v1。
- 纠正：无。
- 剩余未核实缺口：MAF #10318 发布载体与 #10317 合并与否；Harbor 0.25.1；inspect-ai 0.3.290；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；AHA 2611 前缀 vs 九月日期表面不一致成因；MAF 记忆 API 弃用迁移指南缺口。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：MAF 1.26.0 发布完成情况与 Harbor 0.25.1）。
- 下轮起始 HEAD：见 research/20261113-2209/h-report.md（h-ls-remote.txt）；OTel semconv e6658f1、py-genai b164937 未变。

## 2026-11-14 01:09 轮（第 47 轮）

- 本轮增量（D-H01–D-H06，落入 skill/references/agent-harness.md）：**窗口内新稳定发布 1 件：langchain-openai 1.8.0**（PyPI 2026-11-13T16:48:10Z；OpenAIClassifier 对应 GA Decisions API #41787；OTel #596 仍 open 未收敛）；MAF #10317 仍 open 未合并（auto-merge 已启用 15:18:13Z）、**#10318 前提纠正：非发布准备 PR，是已合并的 breaking shell 路径校验 PR（2026-11-13T09:54:36Z），载体仍未定**、#10298 仍 1 评论；记忆 API 弃用无独立迁移指南；Harbor 0.25.1 仍未发布但**三轮零提交断裂**（窗口内 3 提交：#4282 遥测、#4286 查看器、#4284 推出分发）、ATIF 仍 v1.8、Daytona 泄漏修复在 Unreleased；inspect-ai 0.3.290 未发布（窗口内 13 提交）、Agents SDK JS 0.22.0 / Python alpha.7、ADK 仍 2.14.1、langgraphjs SDK 方向连续第九轮；OTel #516 仍未合并、16 项待决全 open 零窗口内动静、#688 仍 open；AHA 仍仅 v1；**深挖：窗口内任务/trace 格式零新进展**（MAF 1.26.0 与 Harbor 0.25.1 均无格式变化；ADK 遥测/trace 修复收紧语义不改格式；langchain-openai 1.8.0 Decisions 分类无 OTel 对应动作）。
- 纠正：MAF #10318「发布准备 PR」→ 已合并的 breaking 变更 PR（agent-harness.md L287 旧文案随本次整合更新为已合并状态，载体仍未定）。
- 剩余未核实缺口：MAF #10317 合并与否；Harbor 0.25.1；inspect-ai 0.3.290；OTel GenAI span 格式 16 项待决（#688 合并时间）；Agents SDK 稳定化时间表；AHA 2611 前缀 vs 九月日期表面不一致成因；MAF 记忆 API 弃用迁移指南缺口；ADK semconv-units 修复（56b25c6，窗口后）；OTel #596 对 langchain-openai 1.8.0 Decisions 分类的跟进。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：Harbor 0.25.1 发布与 ADK semconv-units 修复）。
- 下轮起始 HEAD：见 research/20261114-0109/h-report.md；OTel semconv e6658f1 未变。

## 2026-11-14 04:09 轮（第 48 轮）

- 本轮增量（D-H01–D-H05，落入 skill/references/agent-harness.md）：**窗口内新稳定发布 4 件，同属一个 langchainjs 版本包批次**（commit 7704471，npm 18:28–18:29Z：@langchain/core 1.4.1、langchain 1.6.6、@langchain/anthropic 1.6.5、@langchain/google 0.2.15）；MAF #10317 仍 open 未合并（auto-merge 已启用，窗口内 timeline 零新事件，已开近 5 小时未合入）、#10318 载体仍未定（无 python-1.27.0/dotnet-1.29.0 标签）、#10298 仍 1 评论；记忆 API 弃用独立迁移指南仍未出现；Harbor 0.25.1 仍未发布（窗口内 2 提交，均不涉任务/ATIF）、ATIF 仍 v1.8；inspect-ai 0.3.290 未发布（窗口内 7 提交）；Agents SDK JS 0.22.0 / Python alpha.7；ADK 仍 2.14.1，56b25c6（semconv 单位修复）与 1de9760（推理 token 指标 v1 全量记录）均已合入；LangChain Python 线窗口内 6 提交含 LangSmith gateway 路由四连（仅登记）；langgraphjs SDK 方向连续第十轮改动，仍无新版；hawk 窗口内 2 提交；OTel #516/#688/#596 均仍 open、16 项待决全部 open 零窗口内动静（search API 全量核对 71 PR / 154 issue）；#204 updated_at 又 bump（19:36:57Z）但 timeline 69 条事件最后一条仍为 11-09，成因未确立，不计为进展；AHA 仍仅 v1；**深挖：窗口内任务/trace 格式零新进展**（ADK 两项 telemetry 提交均为 metric 层收敛，与 #516/#695 同属 token 用量归属主题，约定侧仍未收敛）。
- 纠正：无新推翻项（ADK 56b25c6 由「窗口后待复核」转为「窗口内已合入」，语义定性为 metric 单位对齐、不改 span 格式）。
- 剩余未核实缺口：MAF #10317 合并与否；Harbor 0.25.1；inspect-ai 0.3.290；OTel GenAI span 格式 16 项待决（#688 合并时间；#596 对 Decisions API 的跟进；#204 看板更新成因）；Agents SDK 稳定化时间表；AHA 2611 前缀 vs 九月日期表面不一致成因；MAF 记忆 API 弃用迁移指南缺口；ADK 下一版是否收纳两项 telemetry 改动；langgraphjs SDK 第十轮改动仍无新版。
- 下轮起点：先清上述缺口，再按子主题轮转（评估 harness 任务格式与 trace/span 格式：#516/#695 是否因 ADK 的 token 指标落地而推进；#596 是否跟进 Decisions API）。
- 下轮起始 HEAD：见 research/20261114-0409/h-report.md；MAF 92838b6、Harbor b06002e、inspect-ai 4748612、ADK 1de9760、LangChain master 5569924、langgraph 3183e90、langgraphjs abea7e8、langchainjs 7704471、hawk 6e3bea8、OTel semconv e6658f1 未变。
