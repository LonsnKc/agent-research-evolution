# 轨道：技能（Skills）

基线来源：`~/workspace/research_notes/skill-deep-research-20261024/report.md` 与 `gaps-round2.md`（2026-10-24）及技能参考 `~/workspace/custom-skills/agent_system_design/references/skill-authoring.md`。

基线要点（2026-10-24，已核实）：
- 一手源：Anthropic《Equipping Agents for the Real World with Agent Skills》（2025-10-16 发布，2025-12-18 更新为开放标准）。
- 技能 ≠ 提示（拉取而非推送）、≠ MCP 服务器（程序性知识 vs 工具与数据连接）、≠ 子智能体（知识封装 vs 执行隔离）。
- 跨宿主兼容：描述上限分歧（开放规范 1024 字符 vs DSH 目录 500 vs Codex 文档 500），主描述写 500 字符、触发词在前；共享发现目录 `.agents/skills/` 为最大公约数；DSH 发现排序项目 .dsh/skills(100) > 项目 .agents/skills(200) > 自定义目录(300) > dshHome(400) > ~/.agents/skills(500) > 内置(600)，同名冲突按更近作用域优先；Windsurf/Cascade 不以 .agents/skills/ 为发现路径；OpenCode 2 以路径派生技能 ID 且无匹配权限规则时默认改为询问。
- 渐进披露三级：元数据 → SKILL.md → 按需引用/脚本输出。
- 评估：金任务集（正/负/边界触发）、配对 A/B（通过率差 + token 成本差）；SkillsBench（87 任务）平均 +16.6pp，SWE-Skills-Bench（49 技能）39 个零收益且 token 成本最高 +451%；FORTIS：模型在更广技能看起来更快时 96.0% 选择超范围工具。
- 安全：ClawHavoc 事件（2026-02，Koi Security：2,857 个 ClawHub 技能中 341 个恶意）；安装前逐文件审计，查下载执行、不可见 Unicode、敏感路径、明文密钥、过宽 allowed-tools 等红旗。
- 未核实缺口（见 gaps-round2.md）：若干厂商数字与二手结论仍待一手复核。

---

## 2026-10-25 轮次：窗口内无规范变更；最近动态 10-21～10-23；评估协议深化

窗口结论：水位日 2026-10-24 至今天 2026-10-25 这 24 小时内，公开渠道没有出现 Agent Skills 规范的新版本，也没有任何宿主发布技能子系统的新功能。今日亲眼复核的规范页（agentskills.io/specification）最后更新戳仍是 2026-08-04，name ≤64、description ≤1024、compatibility ≤500、`allowed-tools` 实验性、渐进披露三层等条文与基线一致。

### 最近动态（水位日前后、基线未收录）

- **2026-10-23 Claude Code changelog：无技能格式变化**（已核实，一手 changelog https://code.claude.com/docs/en/changelog）：10-23 条目为 Monitor 工具、云会话权限等，没有 skill 相关条目。同页 10-22 条目有三条技能相关内容（已核实）：1）`--add-dir` 文件夹中的 skill 与已有 skill 同名时不再不可达，现以 `<folder>:<name>` 形式提供——同名冲突处理出现第四种机制（此前基线只有 DSH 的 rank/作用域层、OpenCode 的路径派生 ID、Codex 的目录隔离）；2）内置 dataviz skill 更新（接受 APCA 对比度度量）；3）`/claude-api` skill 更新（服务账号密钥与 Workload Identity Federation token 可管理 OpenTelemetry 目标；子代理团队改用 `multiagent_20261001` 动态工作流）。
- **2026-10-22 AAIF 公告页更新戳，但内容仍是 10-22 宣布，无新规范内容**（已核实）：https://aaif.io/news/agentic-ai-foundation-announces-four-new-projects 全文仍是四项目宣布（Agent Skills、Open Responses、OpenEnv、OpenSandbox，AAIF 由 3 个增至 10 个项目），未提及版本化发布、提案流程或安全指南的任何落地进展；页面标注更新于 2026-10-24，但正文无新增实质内容——AAIF 路线图仍停留在宣布阶段。
- **2026-10-21 JFrog AI-SEC：技能治理从注册表延伸到开发者工作站**（发布事实已核实，新闻稿转载；969 等数字为厂商自报，未核实）：Local Shadow AI Detection（经 JFrog Agent Plugin 清点开发者本机已装的 skills、插件、MCP 服务器）、BYOK 语义扫描（用客户自己的 LLM 端点审技能内容）、Agent Guard（agent 取用资产时执行策略）。其 2026 供应链报告称追踪到 969 个携带高影响载荷的恶意 agent skills、495 个恶意 Hugging Face 模型——数字为厂商自报，无独立审计。口径注意：政策检查“工作站资产是否与已批准资产一致”被 JFrog 自己说明为未来版本才有，即当前发布只有清点与扫描，没有批准态强制。来源：Morningstar 转载 Business Wire 新闻稿；独立分析 diggingbeagle.com。
- **2026-10-20/21 ClawHavoc 计数口径继续发散，不可引用单一总数**（未核实，二手威胁情报汇总）：Threadlinqs TL-2026-3938（首发 2026-10-20）汇总：Koi 341 个（2,857 审计）、VirusTotal 当时 314 个归因同一用户、Antiy 称 hightower6eu 677 个＋第二账号 390 个、历史库 1,184 个恶意包、PolySwarm 称 900+。各源口径、日期、去重方法均不同，引用时只能并列多源并注明口径，任何单一总数都不成立。
- **2026-10-21 左右第三方大审计数字出现，但未经一手复核**（未核实，二手转述）：dev.to 安全综述转述 Mobb.ai 审计：22,511 个 AI coding skills（Skills.sh、ClawHub、GitHub、Tessl 四注册表），140,963 个安全发现，34% 含漏洞，27% 含命令执行模式。未打开 Mobb.ai 原文，数字仅作二手线索，不可直接引用。
- **2026-10-19 个人技能库合并发布 dgalarza/agent-skills 2026.10.19**（已核实，作者自述）：作者把 claude-code-workflows 与旧 agent-skills 库合并为单一库，删减一批不再用的 skills；`npx skills update` 改从合并库拉取。生态意义：分发层仍在整合，但属于个人库维护事件，无规范意义。
- **2026-10-13 Google 消费级 skills 铺开**（部分核实）：Google 官方 Workspace 博客确认的 Gems→Skills 时间表（一手博客）：Workspace 内 skills 自 10-05 铺开、Gemini app 内自 10-13 铺开、11-17 起 Gems 移入设置面板但仍可用、业务/企业账号不早于 2027-03-01、教育账号不早于 2027-06-01 彻底停用并自动迁移为草稿 skills；且 Gemini app 的 skill 与 Workspace 的 skill 不互通，需各建一次。二级解读称 Workspace 侧铺开“延迟至另行通知”（未核实）——官方博客正文未含“延迟”表述。Gemini CLI 0.64.0（10-13）据二级周报“多为修复”，未见技能相关变更（未核实）。
- **2026-10-24 `npx skills` 实践指南发布（分发层，非规范）**（已核实，作者自述的工具用法）：指南确认当前 `npx skills` 的 lockfile 语义：`skills-lock.json` 对应 `package-lock.json`，`experimental_install` 对应 `npm ci`，并提醒 npx 可能命中缓存旧版、需用 `skills@latest`。说明分发层仍处实验态，版本锁定行为不稳定。
- **DSH / Hermes / Codex / Copilot / Windsurf 本轮未见技能子系统新动态**：DSH 检索只见 10-09 的 NAS 版 dsh-v0.2.1-alpha.2、10-13 社区插件 DS Bot v0.1.4（含共享 MCP/Skills 能力包，二级 Medium，未核实）等外围消息，未见 DSH 官方 skills 子系统文档或发现规则的变更。Codex、Copilot 亦未见技能功能新条目。Windsurf/Cascade 亦无新技能文档动态。
- **附：Vercel skills.sh 报告（2026-09-30，窗口前但与“1M/280M”口径相关）**（已核实，Vercel 自家报告）：Vercel 自述 skills.sh 七个月达 100 万 skills、近 2.8 亿安装。这与 Anthropic 捐赠稿的数字量级一致，但这是注册表运营方自报的自家数据，不是 Anthropic 数字的独立审计；基线“厂商自报、未经独立验证”的标注维持不变。

### 深化：技能发现 / 调用 / 评估协议

- **发现机制已从“目录约定”分裂为四种模型，分歧在扩大而非收敛**：1）文件系统目录模型（主流）：`.agents/skills/`、`.claude/skills/`、`.dsh/skills/` 等；冲突解决各家不同——DSH 按 rank＋作用域层（基线已核实）；Claude 10-22 新增 `--add-dir` 同名以 `<folder>:<name>` 命名空间化（本轮一手核实）；OpenCode 2 以路径派生 ID（基线 round-2 已核实）。三者互不兼容：同一冲突在三家有三种胜出/共存结果，跨宿主“一份正本”只能靠避免同名，无法靠统一规则。2）中心注册表模型：skills.sh、skillhub.club 等可搜索目录＋安装计数排行。3）HTTP catalog 模型：OpenCode 2 的 `skills` 数组＋HTTP catalog（index.json＋版本缓存，基线已核实）——发现不再依赖本地文件，版本语义进入协议。4）Agent 自主检索模型：Wild 论文的 skill engine（metadata 索引＋全文索引，Qwen3-Embedding-4B＋BM25），agent 迭代发查询、评估候选——唯一有量化数据的检索范式，但不是任何宿主的官方默认发现路径，而是研究原型。
- **调用触发机制：三种触发面＋两种确认语义，分歧点已可枚举**：模型自主触发（以 description 语义匹配为唯一信号，规范本意）；Gemini CLI 在模型激活时强制用户确认（基线已核实）；OpenCode 2 无匹配权限规则时默认 ask（基线 round-2 已核实）——“默认确认”宿主已不止 Gemini 一家。用户显式触发：Claude `/name`、Codex `$name`、Windsurf `@skill-name`（基线均已核实）；OpenCode 2 的 `slash: false` 可把 skill 藏出命令目录（基线已核实）——显式触发面本身也可被关闭。
- **评估协议核心证据（两篇基线外论文，均已核实原文）——触发准确率与任务成功率必须分离评估**：
  - **论文 A：Liu 等《How Well Do Agentic Skills Work in the Wild》（arXiv:2604.04323，2026-04-06 提交）**：SkillsBench，Claude Opus 4.6——强制加载 curated skills 55.4% → 仅让 agent 自主决定加载 51.2% → 加干扰项 43.5% → 须从 34k 池检索（含 curated）40.1% → 池中去掉 curated 38.4%，而无 skill 基线 35.4%。即最现实设定下 skill 净收益仅 +3.0pp；Kimi（19.8% vs 21.8% 基线）、Qwen（19.7% vs 20.5% 基线）在该设定下为负。加载率与通过率分离的证据：Claude 加载全部 curated skills 的轨迹仅 49%，加干扰项跌至 31%，检索设定下 44%，无 curated 设定下 16%；而 Kimi 加载率高达 86%（curated 设定）却不转化为通过率（38.9% vs 强制加载 38.5%）——“加载得多”≠“用得好”，harness 本身显著影响加载行为。检索上限：最佳方法 agentic hybrid search（metadata＋全文索引）Recall@5 仅 65.5%——三分之一以上的相关技能根本进不了候选集。修复手段的边界：query-specific refinement（先探索任务、再合成定制 skill）可把 40.1% 拉回 48.2%（SkillsBench 含 curated 池）、Terminal-Bench 2.0 从 61.4% 拉到 65.5%；但在无 curated 池时几乎无效（38.4%→37.9%）。论文原话含义：refinement 是已有质量的乘数，不是新知识的生成器。
  - **论文 B：Kevin 等《Evaluating Skills, Not Just Agents: ACES》（arXiv:2608.20614，2026-08-20 提交）**：提出配对 live trial（有 skill vs 无 skill）＋轨迹归一化格式 ATIF＋六项默认运行时指标，报告 Skill Lift（复合分与纯结果分分开报）；145 个真实企业/公开 skills、58/64 个生产 skills 的 947 个配对案例中，平均复合 Skill Lift 0.2134（95% CI [0.1967, 0.2301]），纯结果 lift 0.1799，72.8% 配对为正；最大过程性增益来自 skill 执行、行为检查、skill 效率——恰好是文档静态扫描看不见的发现/路由/工作流信号。关键方法论结论：纯静态扫描（结构 vs LLM 评判）Spearman ρ 仅 0.14——静态门禁与 live 效果几乎不相关，两者必须并行，不可互相替代。开源实现为 NVIDIA SkillEvaluator。
  - **共识与分歧**：共识（两篇论文＋基线两篇基准一致）：必须同时报告（a）触发/加载率、（b）检索召回（如 Recall@5）、（c）任务通过率、（d）token/时延差；只报通过率会把“没触发”“没召回”“触发了但没用好”三种失败混为一谈。分歧：SkillsBench/SWE-Skills-Bench 用“给定 skill、测增益”；Wild 用“从池中检索、测现实收益”；ACES 用“生产 skills 配对 live trial、报 Skill Lift 复合分”。三者测的不是同一个量，跨论文比较数字前必须先对齐设定（是否强制加载、池中有无 curated、是否允许 refinement）。
  - **金任务集设计启示（可直接复用 Wild 的阶梯）**：强制加载上限 → 自主选择 → 加干扰项 → 池检索 → 去 curated 池，五档阶梯一次跑完，比单点 A/B 更能定位失败层。

### 已同步进 skill 参考

- skill-authoring.md：跨宿主兼容一节末尾新增 Claude 10-22 `<folder>:<name>` 命名空间化条目（已核实）；“Evaluating a skill”一节新增“Separate trigger accuracy from task success”小节（两篇论文，数字标论文报告）；安全审查一节追加 JFrog 工作站清点条目（发布事实已核实，969 数字标厂商自报）与 ClawHavoc 多源口径警示；跨宿主兼容一节追加消费级 Gemini 分歧注记（日期一手，延迟说法未核实部分不写入）。无 SKILL.md 级变更。

### 待核实线索池（明确标注，勿直接写入参考正文）

1. OpenCode 2.0.30 的 `OPENCODE_DISABLE_CLAUDE_CODE` / `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS` 开关（二级版本追踪器转述，未见官方 changelog 原文）——若属实，“跨宿主共享目录可达”又多一个可被用户主动关闭的维度。
2. Cursor `/skills` 静态校验五项阈值（名称超 64 字符、非连字符名、frontmatter `name` 与文件夹不一致、description 超 1,024 字符、正文超约 5,000 tokens 均被拦截；二级转述，未见 Cursor 官方文档原文）——若属实，这是首个把“描述/正文预算”从文档建议变为安装时硬校验的宿主。
3. Claude Code 2.1.299 钩子前移与 UserPromptExpansion 事件（二级转述；官方 changelog 文本抓取未命中该条目）——若属实，skill 正文展开时机本身成为可观测、可拦截的执行事件，评估时“触发”与“执行”需分开计时。
4. Mobb.ai 审计数字（22,511 技能、140,963 发现、34% 含漏洞、27% 含命令执行模式）——未打开 Mobb.ai 原文。
5. Google Workspace skills 铺开“延迟至另行通知”——只见二级解读，官方博客正文无此表述。
6. AAIF 路线图（版本化发布、提案流程、安全指南）自 10-22 宣布后无任何落地文件；下轮应直接检查 AAIF 仓库/公告而非转述。
7. DSH、Hermes/Ekko、Codex、Copilot、Windsurf/Cascade 本轮均无技能子系统官方更新；DS Bot v0.1.4 等社区动态未核实，不宜写入参考。


---

## 2026-10-27 轮次：缺口复核——OpenCode 开关、Cursor 归属纠正、Claude 钩子前移、AAIF 落地未见

### 已核实（一手）

- **OpenCode 三开关**：官方 CLI 与规则页列出总开关、`_PROMPT`、`_SKILLS` 三者语义；v2.0.30（2026-10-15）changelog 确认在 `.claude` 技能发现中执行总开关与 `_SKILLS` 开关；v2 只用 `.claude` 做技能发现、不加载 `CLAUDE.md` 提示词，故 `_PROMPT` 在 v2 基本为空操作；相邻开关 `OPENCODE_DISABLE_EXTERNAL_SKILLS` 顺带记录。
- **Cursor 校验归属纠正**：Cursor 官方文档全文无任何数字阈值与拒绝规则；64/1024 为 Agent Skills 规范约束（name 1–64、description 1–1024、compatibility 1–500），正文「约 5000 tokens」「500 行」为规范建议且规范正文称正文无格式限制——均不得写成 Cursor 硬校验。
- **Claude Code 钩子前移**：v2.1.299（2026-10-14）起 `UserPromptSubmit` 在技能/斜杠命令展开前运行，阻止提示即阻止内嵌 shell 命令；展开文本检查改用 `UserPromptExpansion`（展开前触发、可阻止、可追加 `additionalContext`、按 `command_name` 匹配）；该事件首发版本未核实，不写。
- **AAIF**：2026-10-22 宣布新增四项目（总数达 10 个），名称、logo、组织与域名已移交；截至 2026-10-27 公开检索，未见已发布的版本化发布、正式提案流程、安全指南文件（检索未见 ≠ 证明不存在）。
- **规范页**：agentskills.io 规范页更新戳仍为 2026-08-04T22:53:14.470Z（自验者亲自打开核对），无新版本；首页 2026-08-09 为另一页面戳。

### 已同步进 skill 参考

- skill-authoring.md：跨宿主兼容一节新增 OpenCode 开关、Claude 钩子、Cursor 归属纠正、AAIF 治理四条目；厂商宣布口径（70+ 平台、100 万技能、近 2.8 亿安装）标注为未独立核实。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号（changelog 页检索未命中）。
2. Mobb.ai 审计数字原文（仍未打开）。
3. Google Workspace skills 铺开「延迟」说法（官方博客正文仍无此表述）。
4. AAIF 落地文件（版本化发布、提案流程、安全指南）持续检索。
5. 已解决：OpenCode 开关、Cursor 校验归属、Claude 钩子前移、AAIF 宣布确认、规范页更新戳。
6. DSH、Hermes/Ekko、Codex、Copilot、Windsurf/Cascade 本轮仍无技能子系统官方更新（检索未见）。


---

## 2026-10-28 轮次：缺口复核——Google 延迟口径修正、Claude 10-27 修复、AAIF 仍为提案

### 已核实（一手为主）

- **Google Workspace skills 延迟口径修正（推翻上一轮结论）**：官方帖子（2026-10-13）正文段落写 "those rollouts are slightly delayed"，Rollout pace 表格中 "Skills in Workspace — Rapid and Scheduled Release domains" 一行明确写着 "Delayed until further notice"。上一轮"官方正文无此表述"不准确（只看段落成立，计入表格则不成立）。Gemini app 技能自 2026-10-13 起逐步推出、预计 11 月中完成。
- **规范页戳仍为 2026-08-04**：执行者直接打开 agentskills.io/specification 核对，页脚仍为 2026-08-04T22:53:14.470Z；`allowed-tools` 仍为实验性字段；总览首页戳 2026-08-09 与规范页不同。
- **AAIF 落地文件仍为提案**：截至 2026-10-28，版本化发布、正式提案流程、安全指南均未见已发布文件；二手报道一致表述为 Anthropic"提议"的路线图（DataPhoenix 称"拟议的 12 个月路线图"）；AAIF 官方 2026-10-22 公告本身只宣布新增四个项目。
- **Claude Code 2026-10-27 changelog 技能修复**：修复 per-directory 与 `paths:` 技能在 Bash `cd` 深入 git 仓库后找不到的问题；prompt/agent hooks 上限改为每会话至多 24 个。该条目在官方 changelog 页摘录中未显示 CLI 版本号，二手聚合的 2.1.312 归属未核实，只写日期条目。
- **窗口内其他**：OpenCode v2.2.0（2026-10-27）仅二手聚合标题，未复核到技能专项条目；Codex CLI 0.165.0（2026-10-27）官方 changelog 所列条目无技能项；Cursor、Copilot、Windsurf、Gemini CLI、DSH / Hermes 窗口内检索未见技能子系统更新。

### 已同步进 skill 参考

- skill-authoring.md：消费级 Gemini 一节追加延迟口径修正行；Claude 钩子一节追加 2026-10-27 修复行（不写版本号）；AAIF 一节更新为截至 2026-10-28 仍为提案 + 规范页戳复核。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号（changelog 页检索仍未命中）。
2. Mobb.ai 审计数字原文（仍未打开；34% / 27% 仅见单一 dev.to 转述）。
3. AAIF 落地文件（版本化发布、提案流程、安全指南）持续检索。
4. OpenCode v2.2.0 官方发布说明技能专项（未复核）。
5. 已解决：Google 延迟口径、规范页戳、AAIF 宣布确认。
6. DSH、Hermes/Ekko、Codex、Copilot、Windsurf/Cascade 本轮仍无技能子系统官方更新（检索未见）。
