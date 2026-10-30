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
