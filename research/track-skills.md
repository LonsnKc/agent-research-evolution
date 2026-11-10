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


---

## 2026-10-29 轮次：缺口复核——OpenCode v2.2.0 无技能专项、AAIF 提案约束、Claude 10-26 条目

### 已核实（一手为主）

- **OpenCode v2.2.0 无技能专项（上一轮“未复核”现已解决，结论为否定性）**：直接打开官方 V2 CHANGELOG.md 并页内检索 `skill`，v2.2.0 节（2026-10-27）内仅命中三条边际条目：#59086（来源变更才重载兼容技能）、#56260（无关配置变更与更新突发不再反复触发重载）、#58023（桌面端在 side tabs 中打开扩展配置与技能）。格式、发现路径、命名、触发语义、字段上限均无新增规则，兼容清单不新增条目；仅可作重载节流注记（宿主实现层稳定性修复）。邻近条目 #56550（插件会话指令）不是技能变更，不得混淆。
- **Claude Code changelog：2026-10-28 之后无新条目**：官方 changelog 页本日抓取戳 2026-10-29 01:52 UTC，页首第一条为 2026-10-27，检索 “October 28, 2026” 零命中；2026-10-27 条目复核与上一轮一致（摘录不显示 CLI 版本号，不归属版本号）。窗口外顺带复核的 2026-10-26 条目技能相关项：Claude Tag 新增 Skills view；`CLAUDE_CODE_SYNC_SKILLS` 且特性标志获取关闭时可复用本机另一会话 10 分钟内获取的列表；修复中断时 MCP 资源读取未答复或技能仍在下载导致中断不生效。
- **规范页戳仍为 2026-08-04**：直接打开 agentskills.io/specification 核对，页脚仍为 2026-08-04T22:53:14.470Z；字段口径当日复核无变化（`allowed-tools` 仍实验性）；总览首页戳 2026-08-09 为另一页面戳。
- **AAIF 提案状态对技能作者的实际约束（深挖，只写已确认事实）**：无法声明规范版本（只能写“遵循现行规范（2026-08-04 戳）”）；无正式提案通道可走（不得写“按 AAIF 提案流程提交”）；`allowed-tools` 仍实验性且支持因宿主而异（路线图“稳定或移除”处置本身未发布）；安全指南缺位时无官方基线可引用；“宣布移交”不等于“移交完成已证实”（贡献协议签署与理事会审查状态仅见二手说法）。

### 已同步进 skill 参考

- skill-authoring.md：OpenCode 段追加 v2.2.0 无专项与重载节流注记；AAIF 段追加作者五约束与规范页戳复核日期（2026-10-29）；Claude 钩子段追加 2026-10-26 条目与 2026-10-28 之后无新条目戳（均不归属版本号）。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号（官方 changelog 检索仍零命中；新增二手线索：hooks 参考 2026 年 4 月下旬新增说、第三方工具 2.1.116 下限说、文档变更 2026-07-15 出现说，三者均非首发版证明）。
2. Mobb.ai 审计数字原文（仍未打开；34% / 27% / 22,511 / 140,963 仅见单一 dev.to 转述，连续四轮未决；SkillSecurer 17.6% 为另一项独立研究，不得混用）。
3. AAIF 落地文件（版本化发布、提案流程、安全指南）持续检索（截至 2026-10-29 检索未见已发布文件；二手口径一致为“提议中”）。
4. Cursor Agent SDK 的 `skills/` 语义（待官方 changelog / 文档直核，目前仅二手聚合与报道）。
5. Codex CLI 0.165.0“无技能项”（待官方 release 页直核，目前仅二手聚合清单）。
6. MiniMax MCode Stack（2026-10-28）仓库原文与技能清单（待下轮一手复核，目前仅二手报道；属技能库内容发布，非规范/宿主变更）。
7. 已解决：OpenCode v2.2.0 官方发布说明技能专项（否定性结论）、规范页戳复核、Claude Code 2026-10-28 之后无新条目戳。
8. DSH、Hermes/Ekko、Codex、Copilot、Windsurf/Cascade 本轮仍无技能子系统官方更新（检索未见；Copilot Fusion 与 Gemini 4 Carbon 属模型/路由层，不计入）。


---

## 2026-10-30 轮次：缺口复核——Codex 无技能项、Cursor SDK 语义、MCode 仓库、触发面无收敛

### 已核实（一手为主）

- **Codex 0.165.0 确无技能项（官方 release 页直核）**：全文 169 行，检索 `skill` 0 命中；最接近的仅为 marketplace URL 校验缺口修复（#56263）；发布于 2026-10-27。0.166.0（2026-10-29，二手日期）新功能无技能项，完整 changelog 有 6 条技能周边 plumbing（#56524、#56530、#57197、#57262、#57500、#57598），均不改动发现路径/触发语义。
- **Cursor Agent SDK `skills/` 语义（官方 changelog 直核）**：原文只说存放“模型按需加载的程序（procedures）”，全文未提 `SKILL.md`，不得等同于 agentskills.io 技能；GA 日期 2026-10-20 为二手，官方页正文未标日期。
- **MiniMax MCode Stack 仓库原文（创建 2026-10-28，一手）**：MIT（`engineering-*` 另按 CC-BY 3.0），README 原文表共 17 技能（ronny 15 + jiachen 2，见 research 报告）；技能 = 含 `SKILL.md` 的目录，MiniMax Code 内以 `/<skill-name>` 调用或按 description 加载；发现目录：项目级 `.minimax/skills/`（特有）、`.claude/skills/`、`.agents/skills/`，全局 `~/.agents/skills/`（另读 `~/.claude/skills/`、`~/.codex/skills/`）；可经 marketplace `MiniMax-AI/mcode-stack` 安装插件或单技能复制到 `~/.agents/skills/`。“5 checklists / 5 companion notes”数量在 README 文本中无数量陈述，不得照抄。
- **Claude Code 2026-10-30 新组（官方 changelog 一手）**：无技能子系统新规则（仅存量技能更新：dataviz 两处、`/claude-api` 两处、会话自动标题改用技能/提示型命令、修复 `~/.claude/skills/.trash` 监听负担）；10-30 组版本号未以版本标题呈现，不得指称为 2.1.313；最新已编号版本仍为 2.1.312（2026-10-27，二手镜像一致）。
- **规范页与 AAIF（2026-10-30 复核戳）**：规范页直核全文 168 行，页脚戳仍为 2026-08-04T22:53:14.470Z，6 个 frontmatter 字段，`allowed-tools` 仍 Experimental，无版本化发布、无提案流程、无 SECURITY 章节；AAIF 三件落地文件在规范页与 AAIF 公告面均未出现（未穷尽 Discussions / Discord）。
- **OpenCode（2026-10-30 复核戳）**：顶栏仍为 2026-10-29 两组（v2.2.1），无 10-30 组；10-29 顶栏本身无技能子系统新条目。
- **Gemini CLI 区分**：最新稳定版仍为 v0.66.0（2026-10-27，二手镜像元数据），唯一技能相关项为 SKILL.md frontmatter 单行 description 解析修复（#29769，二手镜像转述）；Gemini 应用 Skills 与 Gemini CLI 技能子系统是两回事，不得混写。
- **触发面分歧无收敛**：规范页不定义发现目录（仅目录内格式与三级渐进披露），AAIF 移交只改治理、不改格式；2026-10-29 之后未见任一宿主发布统一触发语义；`~/.agents/skills/` 在多家文档中反复出现只是习惯重合，规范并未强制要求扫描它；第三方所称 Claude Code 技能页 10-29 “7 行变动”内容未获证实。

### 已同步进 skill 参考

- skill-authoring.md：追加 Codex 0.165.0/0.166.0 否定性结论、Cursor SDK 语义警示、MiniMax MCode Stack 宿主示例；Claude 钩子段追加 2026-10-30 新组句；AAIF 段追加 2026-10-30 复核戳；消费者 Gemini 段追加 Gemini CLI 区分句与触发面无收敛句。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号与首发日期（官方 changelog 检索 0 命中；2.1.116 仅为 ccdeck 二手 gating 阈值）。
2. Mobb.ai 审计数字原文（22,511 / 140,963 仅见 2026-03-22 TNS 具名报道；34% / 27% 仅见单篇 dev.to 转述）。
3. AAIF 落地文件（版本化发布、提案流程、安全指南）持续检索。
4. Claude Code 2026-10-30 条目对应的版本号；技能页 10-29 “7 行变动”具体内容；MCode “5 checklists / 5 companion notes”数量。
5. 已解决：Codex 0.165.0 技能专项（否定性）、Cursor Agent SDK 语义、MiniMax MCode Stack 仓库原文、规范页戳复核、Claude Code 2026-10-30 新组、触发面无收敛判定。
6. DSH、Hermes/Ekko、Codex、Copilot、Windsurf/Cascade 本轮仍无技能子系统官方更新（检索未见；Copilot Fusion 与 Gemini 4 Carbon 属模型/路由层，不计入）。


---

## 2026-10-31 轮次：缺口复核——OpenCode #60315、Claude 10-31 组、MCode 三形态、规范页戳

### 已核实（一手为主）

- **OpenCode 2026-10-30 组技能子系统变更（窗口内唯一一手技能变更，执行者亲自复核）**：托管策略拆分 #60315（breaking change）——`integration.use` 仅控制 integration 账户，新增 `mcp.use`、`skill.use`、`plugin.use`；旧资源前缀规则自动转换，裸 `integration.use` 不再适用于技能。评估“技能是否可用”须区分“已安装 / 已发现”与“策略放行”两层；官方页日期组不标版本号，v2.2.3 / v2.2.4 归属为二手镜像。
- **Claude Code 2026-10-31 组无技能项（一手）**；10-30 组为存量技能内容更新（dataviz、`/claude-api`、自动标题、`.trash` 监听修复），自动标题只提升元数据可见性；10-30 组版本号三家镜像一致归属 2.1.314，官方页无版本标题，仍标二手一致。
- **规范页戳复核至 2026-10-31（执行者亲自复核）**：全文 168 行，页脚戳仍为 2026-08-04T22:53:14.470Z，`allowed-tools` 仍 Experimental；AAIF 三件落地文件仍未出现（未穷尽 Discussions / Discord）。
- **MCode 数量误读来源查明（仓库原文一手）**：README 全文无“5 checklists / 5 companion notes”数量陈述；Articles 表恰好 5 行（01–05）各配 companion skills，二手“5”极可能源于该表误读；三形态（Skill / Checklist / Workflow）应分别计数。
- **其余宿主窗口内检索未见带日期的一手技能子系统更新**：Codex（仍为 0.166.0，10-29）、Gemini CLI（仍为 v0.66.0，10-27）、Cursor（CLI 三条技能条目 undated）、Copilot、Windsurf/Cascade（已更名 Devin Desktop）、DSH、Hermes/Ekko。

### 已同步进 skill 参考

- skill-authoring.md：OpenCode 段追加 #60315 独立策略位；Claude 钩子段追加 10-31 组与 2.1.314 镜像归属；MCode 段追加三形态计数与误读纠正；AAIF 段推进规范页戳至 2026-10-31。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号与首发日期（官方 changelog 仍 0 命中；文档出现窗口收窄至 2026 年 4 月下旬；2.1.116 仅 ccdeck 二手 gating 阈值）。
2. Mobb.ai 审计数字原文（复述载体增至三家，仍无 mobb.ai 原件；34% / 27% 仍仅单篇 dev.to；Cisco 31,000+ / 26% 为另一审计）。
3. AAIF 落地文件持续检索。
4. Claude Code 2026-10-30 组版本号一手确认；技能页 10-29“7 行变动”具体内容。
5. MCode checklists 目录实际数量（未逐项清点）。
6. 已解决：OpenCode #60315、MCode 仓库原文直核、Claude Code 10-31 组无技能项、规范页戳复核至 2026-10-31。


---

## 2026-11-03 轮次：缺口复核——Claude 10-30 组版本号一手确认、“7 行变动”误植排除、MCode 清点、规范页戳

### 已核实（一手为主）

- **Claude Code 2026-10-30 组版本号已解决（官方 GitHub release 页一手 = v2.1.314）**：条目与日期组逐项对应，不再依赖镜像一致；版本链二手一致：2.1.311（10-26）、2.1.312（10-27）、2.1.314（10-30）、2.1.316（10-31）；10-28、10-29、11-01 无新版本。
- **技能页 10-29“7 行变动”已解决（排除性结论）**：10-28/29 根本无新版本、无对应 changelog 日期组；唯一已核实的“七”是 2.1.299–2.1.301 七项 hooks 变更（2026-10-14/15），属日期、页面、主题三重误植，应予删除；技能正文从未写入该主张。
- **MCode checklists 目录实际数量已解决（仓库目录清点一手，commit 仍为 49a3ff4…）**：全仓仅 ronny 栈 1 个 `checklists/` 目录、恰好 5 个文件；`notes/` 亦 5 个文件；修正上一轮裁定为“目录清点为 5 个”，不得写成“README 称有 5 个”。
- **规范页戳复核至 2026-11-03（执行者亲自复核）**：全文 168 行，页脚戳仍为 2026-08-04T22:53:14.470Z，6 字段，`allowed-tools` 仍 Experimental；AAIF 三件落地文件仍未发布。
- **Mobb.ai 数字部分推进**：22,511 / 140,963 升格为 TNS 2026-03-22 具名报道背书（仍非 mobb.ai 原件）；34% / 27% 仍仅单篇 dev.to，不得升级。
- **窗口内**：Claude Code 11-03 组仅 2 条存量技能项（plugin-authoring 一次提示、claude.ai 同步修复），无新规则；版本号 2.1.317 仅 registry + diff 站 corroborated，官方 release 页未打开，技能中不写；OpenCode 顶栏仍 10-30、Codex 仍 0.166.0、Gemini CLI 仍 v0.66.0、Cursor / Copilot 无带日期技能项。
- **深挖**：触发面分歧仍无收敛；Skill Crossroads 2026-09-10 审计（第三方 linter 自家报告，未独立核实）：216 技能中 69% 描述不可靠触发，触发/召回层是技能收益的第一漏斗。

### 已同步进 skill 参考

- skill-authoring.md：Claude 钩子段改为 10-30 组 = v2.1.314 一手确认 + 11-03 组新增句与窗口否定句；MCode 段改为目录清点 5 个文件口径；AAIF 段推进规范页戳至 2026-11-03；触发准确率小节追加 Skill Crossroads 审计证据句。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号与首发日期（ccdeck 断言 2.1.116、npm 发布 2026-04-20、文档 4 月下旬窗口一致但均非官方一手；npm 包体二分法已证实走不通）。
2. Mobb.ai 审计数字原文（22,511 / 140,963 已有 TNS 具名报道背书，仍无 mobb.ai 原件；34% / 27% 仍仅单篇 dev.to）。
3. AAIF 落地文件持续检索（规范页戳至 2026-11-03 仍为 2026-08-04）。
4. Windsurf / Devin workflows 转 skills 的一手日期（迁移向导存在有二手佐证，日期无一手证据）。
5. Claude Code 2026-11-03 组版本号的一手确认（目前为 registry + diff 站 corroborated 的 2.1.317，官方 release 页未打开）。
6. 已解决：10-30 组版本号（官方 release 页一手）；技能页 10-29“7 行变动”（排除性解决）；MCode checklists 目录数量（逐项清点 5 个文件）。


---

## 2026-11-04 轮次：缺口复核——Claude 2.1.317 一手确认、Devin Playbooks 日期、规范页戳

### 已核实（一手为主）

- **Claude Code 2026-11-03 组版本号已解决（官方 GitHub release 页一手 = v2.1.317）**：条目与 CHANGELOG 的 2.1.317 节逐条对应（/schedule 连接器警告、工具发送账号邮箱前确认、onFailure: "block"、网关成本报告等）；上一轮“不写入”限定解除。release 页抓取文本本身未显示发布日期，日期仍为二手 corroborated（追踪站称 2026-11-03；npm 侧 2.1.317 发布于 2026-11-02 23:57 UTC，即北京时间 2026-11-03 07:57）。11-03 组技能项仍为两条存量项，无 SKILL.md 格式、发现机制或 allowed-tools 规则变化。
- **Claude Code 窗口内新组 2.1.318，无技能子系统新规则**：官方 release 页全文与 CHANGELOG HEAD 首节一致；全文扫描无 SKILL.md 格式、发现、触发、allowed-tools 相关条目（最接近的仅为 mod hook 与 workflows 恢复修复）。日期为二手（追踪站标 2026-11-03；release 页抓取文本未含日期字段）。
- **Devin Playbooks→Skills 一手日期已锁定：2026-10-24（新账号）**：官方文档页（页脚 Updated: 2026-10-30T22:01:27.263Z）“Skills vs. Playbooks”节原文：“New accounts will start with Skills and have no Playbooks, from October 24, 2026.” 迁移时每个 playbook 成为同指令的 skill，Playbooks 页只读、API 读写 skill。发现机制（一手）：启动时加载全部 skill 的 name + description，调用时读入全文并驻留当会话；`triggers: ["user"]` 可禁自动触发；`allowed-tools` 在 skill 激活期间免提示，其余工具照常提示。**不得挪用**：此日期是 Devin Playbooks 的，不是 Windsurf/Desktop workflows 的。
- **规范页戳复核至 2026-11-04（执行者亲自复核）**：全文 168 行，页脚戳仍为 2026-08-04T22:53:14.470Z，`allowed-tools` 仍 Experimental；AAIF 三项落地文件（版本化发布、提案流程、安全指南）窗口内零一手命中，仍为提案（未穷尽 AAIF 全站检索，不等同于“已确认不存在”）。
- **Mobb.ai 数字口径推进**：mobb.ai 原件仍未出现（站内 Vibe Shield 产品发布 2025-06-16 全文无相关数字）；基数 22,511 / 140,963 由 TNS 具名报道 + clawbot wiki 复述升格为二手双源，仍非 mobb.ai 一手；34% / 27% 仍仅单篇 dev.to，不得升级。易混淆的不同口径审计（Liu 等 42,447 采集、26.1% 含漏洞；Queen's 大学 1,527,281 英文 skill 中约 1/17 含“松绑监督”指令）不得与 Mobb 数字混写。
- **`UserPromptExpansion` 首发版仍未决**：官方 CHANGELOG 全文检索 0 命中；二手分析称 hooks 参考在 2026 年 4 月下旬新增该事件与 PostToolBatch，changelog 从未提及；文档变更追踪称 2026-07-15 的 agent-sdk 文档页新增该事件说明——两者追踪的文档页不同，日期存在张力，须以官方 docs 仓库提交史为准，而该提交史本轮未能通过公开检索获得；ccdeck 断言 2.1.116 不采纳。技能正文“首发版未验证、不得陈述”口径不变。
- **Windsurf / Devin workflows 转 skills 一手日期仍未决**：FAQ 仅确认 2026-06-02 更名为 Devin Desktop、经典体验仍在；Devin Local 以 rules 与 skills 取代 memories 与 workflows 的表述无日期；迁移向导存在仅二手佐证。Google Antigravity 的 workflows→skills 迁移向导（遗留 workflows 至 2026-10-19 后不再索引/执行）是另一产品，不得记为 Windsurf/Devin 的证据。
- **窗口内其余否定性结论**：OpenCode（v2.2.5 归属日期分歧：TurboAI 称 11-03 vs gradually/Havoptic 称 10-30/31，内容无技能条目；最近技能变更仍为 v2.2.3 的 #60315 系，窗口外）、Codex（最新稳定版仍 0.166.0，2026-10-29；alpha 0.167.0-alpha.4 为 2026-11-03，二手复述，无技能项）、Gemini CLI（仍 v0.66.0，2026-10-27）、Cursor（无带日期的窗口内技能新发布）均无技能子系统新规则。Skill Crossroads 审计口径窗口内无更新（沿用 2026-09-10 版，数字为二手报道，未取回原站）。SkillsBench 快照（数据截至 2026-11-03，二手聚合）与 SkillzBench 通稿（厂商自报，实质发布日 2026-10-23 在窗口外）均非新评估协议，不写入技能。

### 已同步进 skill 参考

- skill-authoring.md：Claude 11-03 段改为 2.1.317 一手确认 + 2.1.318 窗口否定句；新增 Devin Playbooks→Skills 宿主示例 bullet（含不得挪用警示）；AAIF 段推进规范页戳至 2026-11-04。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号与首发日期（官方 docs 仓库提交史未获；仅知文档出现不晚于 2026-07-15 的 agent-sdk 页、hooks 参考页约 2026-04 下旬；2.1.116 不采纳）。
2. Mobb.ai 审计数字原文（mobb.ai 站内未见；34% / 27% 仍为 dev.to 单源二手；基数已升格二手双源）。
3. AAIF 落地文件持续检索（规范页戳至 2026-11-04 仍为 2026-08-04）。
4. Windsurf / Devin workflows 转 skills 的一手日期（迁移向导存在有二手佐证，日期无一手证据；Devin Playbooks 的 2026-10-24 已解决，不可挪用）。
5. OpenCode v2.2.5 的归属日期分歧（TurboAI 称 11-03 vs gradually/Havoptic 称 10-30/31）。
6. Skill Crossroads 原站新版（窗口内未见；9-10 版口径沿用，数字为二手）。
7. Claude Code 2.1.317 / 2.1.318 的 GitHub release 页发布日期（抓取文本未含日期字段，日期依赖追踪站 corroboration）。
8. 已解决：Claude 11-03 组版本号一手确认；Devin Playbooks 日期与发现机制；规范页戳至 2026-11-04。

---

## 2026-11-05 轮次：缺口复核——OpenCode v2.2.5 日期、Claude 2.1.318 口径分歧、规范页戳、SkillzBench 消歧

### 已核实（一手为主）

- **OpenCode v2.2.5 归属日期判为 2026-10-30**：二手多源一致（Havoptic、ComputingForGeeks、2026-11-01 三方综述称 2.2.3–2.2.5 均于 10 月 30 日落地）；硬佐证：Arch Linux 官方包页 opencode 2.2.5-1 构建日期 2026-10-31 14:11 UTC，即 tag 必然不晚于 10 月 31 日，与“11 月 3 日发布”不相容（一手发行方包元数据，间接佐证）。TurboAI 汇总页（更新于 2026-11-04）的 11-03 应记为该页分析刊出/汇总日期口径，不是 GitHub tag published_at。v2.2.5 本身在窗口之前，不计入窗口内新增；内容无技能条目。
- **Claude Code 2.1.318 日期口径分歧定位**：本轮页面文本抽取仍不含 GitHub published_at 字段，不得凭空填写。次级证据分裂：多数称 11 月 4 日，gradually.ai 与 Havoptic 列为 11 月 3 日；最合理解释（标注为推断，未经 published_at 证实）：若 tag 实际发布于 11 月 3 日 UTC 晚间，则 UTC 口径为 11 月 3 日、CST 口径为 11 月 4 日。台账暂写作“2.1.318（UTC 11-03 / CST 11-04 口径分歧，待 published_at 定稿）”。2.1.317 次级共识为 2026-11-03（01:25 UTC = 09:25 CST，同日在 UTC 与 CST 下无歧义）。
- **规范页 + AAIF 戳推进至 2026-11-05（执行者独立抽查复核）**：规范页全文 168 行，页脚戳仍为 2026-08-04T22:53:14.470Z，allowed-tools 仍 Experimental；AAIF 三项落地文件（版本化发布、提案流程、安全指南）窗口内零一手命中，仍为提案（未穷尽 AAIF 全站检索，不等同于“已确认不存在”）。
- **Skill Crossroads 原站无新版**：主页与报告页 2026-11-05 抓取均无新版本号/新版戳；仍沿用 2026-09-10 rubric v1.2 报告（216 个公开 Claude Code 技能、18 个仓库，平均分 82.1/100；触发可靠通过率 31%（67/215），borderline 28%（61），不会触发 40%（87）；VERIFY-01“含 evals”通过率 0%（197 fail / 208），VERIFY-04“含验证步骤”3%，CLARITY-05“约束与失败模式已声明”0%）。31% 是该样本在该 rubric 下的 LLM 判定结果，不是“全生态技能触发准确率”，不得去掉样本限定引用。
- **Gemini CLI 窗口内两版均无技能条目**：v0.67.0（GitHub published 2026-11-04T20:27:11Z，二手转录；一手页条目：新增 models 命令与 --list-models、IDE companion 桥接进容器沙箱等）与 v0.68.0-preview.0（2026-11-04T19:49:22Z，二手转录）。
- **窗口内其余否定性结论**：Claude Code（2.1.318 落在窗口内，CST 口径 11-04；技能子系统无新规则，仅自定义斜杠命令静默修复）；OpenCode（无新发布，仍 2.2.5）；Codex CLI（无新发布，仍 v0.166.0，2026-10-29）；Cursor（CLI changelog 现页三条技能改进均无每条日期，不能断言为窗口内发布，故不计入）。
- **深挖素材（口径分层，不整段入技能）**：规范层渐进披露三级（元数据约 100 tokens → 指令建议 <5000 tokens → 资源按需加载），规范不定义存放位置与触发优先级；宿主行为层（二手）：Claude description 是路由信号，模型倾向 under-trigger，skill-creator 建议描述稍“pushy”并以否定触发抑制 over-trigger，描述规范层上限 1024 字符、Claude 列表截断于 1536 字符；触发准确率工程化度量（少量测试提示→带/不带对照跑→描述改进器优化；评价分输出质量与触发精度两轴）；跨宿主差异（Claude/Codex 以 description 隐式匹配；Codex 可用 agents/openai.yaml 关闭隐式调用；Octomind 用声明式规则；Cursor 另有 fallback 目录与嵌套目录作用域）——同一 SKILL.md 可移植的是文件格式，不是触发语义。
- **SkillsBench 与 SkillzBench 是两个不同事物，不得混名**：SkillsBench 为论文预印本（Li 等，arXiv 2026-02；本轮读到的版本摘要称 86 tasks、正文分析称 84 tasks——源内数字不一致，见冲突节）；SkillzBench 为 AIQ Labs 于 2026-10-23 发布的厂商目录（SkillzBench.com），口径为隔离无网沙箱中同模型、同任务、同工具跑“无技能 vs 有技能”，差值称 Lift（百分点，附 95% 置信区间），负值亦公开；截至 2026-10-21：发现 1,285 技能、扫描 1,376 次、发布 799 个、有实测结果者 136 个、累计 14,663 次试验；Trust Score 0–100（厂商自报，方法页本轮未打开复核）。两者仅名近，无隶属关系。
- **安全评估基准（与触发/效果基准区分口径）**：Liu 等 arXiv:2601.10338（2026-01-15 提交，一手论文预印本）：收集 42,447、分析 31,132，26.1% 至少一处漏洞、5.2% 呈现高度疑似恶意模式，数据外泄 13.3%、提权 11.8%、带可执行脚本者漏洞风险 2.12 倍；检测器自报精度 86.7% / 召回 82.5%，故数字是估计值而非普查值。Snyk（2026-02-05，二手梳理）：3,984 技能，13.4% 含 critical 级问题、76 个已确认恶意载荷；定义与样本均不同于 Liu，不得相加或互证。NVIDIA SkillSpector（2026-11 上旬二手介绍）：基于同一 Liu 研究的开源扫描器，自称 71 种模式 / 17 类，属厂商工具说明，非独立验证。
- **两则易混淆项排除（均不计入窗口新增）**：“Google 于 11 月 1 日停用消费级 Gemini CLI、Antigravity 为唯一编码 agent”仅见一篇二手 landscape 文章，且与 Gemini CLI 11 月 4 日仍发 v0.67.0 的事实冲突，不采纳。Gemini 应用（非 CLI）的 Gems→skills 过渡时间表（个人账户 2026-11-17 只读、12-17 停用）是窗口前已公布的安排，11 月 4 日更新的只是一篇二手指南文章，不是新的厂商发布。

### 已同步进 skill 参考

- skill-authoring.md：Claude 段追加 2.1.318 口径分歧与 OpenCode v2.2.5 归属日期判词；Gemini CLI 句改为 v0.67.0（2026-11-04）与 v0.68.0-preview.0（均无技能条目）；AAIF 段推进规范页戳至 2026-11-05；触发准确率节追加 Skill Crossroads 评估完整性缺口（VERIFY-01 0% 等）；Evaluating 节追加 SkillsBench / SkillzBench 名称消歧句（含现有数字冲突警示，不改现有数字）。

### 剩余未核实缺口（更新）

1. `UserPromptExpansion` 首发版本号与首发日期（官方 docs 仓库提交史未获；仅知文档出现不晚于 2026-07-15 的 agent-sdk 页、hooks 参考页约 2026-04 下旬；2.1.116 不采纳）。
2. Mobb.ai 审计数字原文（mobb.ai 站内未见；34% / 27% 仍为 dev.to 单源二手；基数已升格二手双源）。
3. AAIF 落地文件持续检索（规范页戳至 2026-11-05 仍为 2026-08-04）。
4. Windsurf / Devin workflows 转 skills 的一手日期（迁移向导存在有二手佐证，日期无一手证据；Devin Playbooks 的 2026-10-24 已解决，不可挪用）。
5. Claude Code 2.1.317 / 2.1.318 的 GitHub release 页发布日期（抓取文本未含日期字段，317 次级共识 2026-11-03，318 为 UTC 11-03 / CST 11-04 口径分歧，待 published_at 定稿）。
6. Skill Crossroads 原站新版（窗口内未见；9-10 版口径沿用，数字为二手）。
7. SkillsBench 数字对齐（现有技能正文 87 任务 / 8 领域 / +16.6pp vs 本轮论文读数 86-or-84 任务 / 11 领域 / +16.2pp；下一轮须打开论文摘要 + 方法节对齐版本与统计口径）。
8. SkillzBench 方法页（本轮未打开复核，口径细节暂为厂商自报）；Cursor CLI changelog 三条技能改进的每条日期。
9. 已解决：OpenCode v2.2.5 归属日期；Claude 2.1.318 口径分歧定位；规范页戳至 2026-11-05。

## 2026-11-07 轮次小节（版本 0.10.0；六路并行之技能两路）

- 本轮增量：UserPromptExpansion 文档首现锁定 2026-04-22（快照提交 76254a0，hooks 参考页；agent-sdk 页迟至 2026-07-15 才补记）；不存在公开的 Anthropic 官方 docs 仓库（anthropics/claude-code-docs 404），提交史来自第三方每日快照镜像 gpambrozio/ClaudeDocs；功能首发版本号仍未决（2.1.116 与 2.1.117 当日 changelog 均未提及；ccdeck 的 2.1.116 断言不采纳）。Mobb.ai：34%/27% 口径已精确化并升格二手双源（TNS 2026-03-22 + dev.to 2026-10-22；34% 是“在 Mobb 所针对模式下检出 ≥1 条发现”，66% 无发现；27% 分母为全体 22,511 技能；140,963 为发现条数）；mobb.ai 站内原文《The State of AI Agent Skill Security — March 2026》（2026-03-31）确认存在但未获 URL、未能打开，仍未决一手原文。AAIF 三项落地文件仍未发布；规范页仍 168 行、页脚戳仍 2026-08-04T22:53:14.470Z、allowed-tools 仍 Experimental；agentskills 仓库 Releases/Tags 均为 0，CONTRIBUTING.md 仍走非正式 Discussions 提案。Windsurf/Devin：迁移向导存在有一手证据（docs.devin.ai 原文 "Migrate your workflows to skills"），但无一手发布日期；二手候选日期互斥（v3.6.21 向导 2026-07-29/30 vs v3.9.19 移除 2026-09-08/29）。窗口内新发布：Claude Code 2.1.319（2026-11-06，版本号二手 corroboration）有技能条目（嵌套技能加载变更、.claude/skills 热重载修复、/claude-api 技能修正）；OpenCode v2.2.6（2026-11-06，一手）组内零技能命中；Codex CLI rust-v0.167.0（2026-11-06 01:40 UTC）仅两条边际技能条目；Gemini CLI 窗口内无新发布（最新稳定仍 v0.67.0，2026-11-04）；Cursor CLI changelog 三条技能改进仍无每条日期。SkillsBench 数字对齐已解决（技能正文 87 任务 / 8 领域 / +16.6pp 为现行版 v4 / SkillsBench v1.1 的正确数字，加版本锚定；此前 86 / 11 领域 / +16.2pp 是已被取代的 v1 快照；18 vs 24 配置之辨已澄清）。SkillzBench 方法页一手复核通过（三星 Baseline/Skill/Forced、95% paired-bootstrap CI、Trust 公式 trust-2026-10-15.1、参考模型 GLM-5.3-flash）；计数更新为 2026-11-07 首页快照（发布 875、有实测结果 152、试验 20,408 次、扫描 2,787 次、发现 2,412）。Claude 2.1.317 / 2.1.318 published_at 已由 GitHub API 一手定稿（317 → 2026-11-03T01:25:46Z；318 → 2026-11-04T00:01:03Z，上一轮“UTC 11-03 / CST 11-04”推断不成立，应撤回）。Skill Crossroads 原站报告页一手取回，仍为 2026-09-10 rubric v1.2，无新版；数字由二手升级为一手。
- 已同步进 skill 参考：skill-authoring.md 追加 Claude 2.1.317/2.1.318 定稿段、2.1.319 / OpenCode v2.2.6 / Codex rust-v0.167.0 窗口更新段、SkillsBench 版本锚定段、SkillzBench 方法页一手复核段、Skill Crossroads 一手复核段、UserPromptExpansion 文档首现段、Mobb 口径精确化段、Windsurf 向导一手存在段、规范页戳复核段。
- 推翻的旧说法：Claude 2.1.318“UTC 11-03 / CST 11-04”推断；SkillsBench“数字冲突未决”；UserPromptExpansion“首发版未验证”限定收窄（文档首现已证实）；Windsurf 迁移向导“存在性仅二手”；Mobb 34%/27% 口径（dev.to 丢限定词）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 功能首发版本。2. mobb.ai 原文页 URL。3. AAIF 落地文件持续检索（规范页戳至 2026-11-07 仍为 2026-08-04）。4. Windsurf 迁移向导日期（实时浏览器开桌面 changelog）。5. Claude 2.1.319 版本号一手确认（官方 changelog 页无版本号标题）。6. Cursor CLI changelog 每条日期。7. Skill Crossroads 下轮复核报告页首行版本/日期。8. 已解决：UserPromptExpansion 文档首现；Mobb 口径精确化；Claude 2.1.317/2.1.318 published_at 定稿；SkillsBench 数字对齐；SkillzBench 方法页一手复核；Skill Crossroads 原站一手取回；规范页戳至 2026-11-07。

## 2026-11-07 轮次小节（下午轮，版本 0.11.0；六路并行之技能两路）

- 本轮增量：Claude Code 2.1.319 published_at 一手定稿（GitHub Releases API，2026-11-06T17:04:01Z；npm 2026-11-06T00:58:56Z 为另一时钟），无 2.1.320+；UserPromptSubmit 钩子现先于技能/斜杠展开运行（同组钩子变更）；UserPromptExpansion 功能首发版本解决为 2.1.116（npm 2026-04-20T19:24:52Z，平台包 tarball 字符串二分法：2.1.110–2.1.114 为 0、2.1.116 起为 58，2.1.115 不存在；ccdeck 同号主张未采纳为来源；字符串首现 ≠ 运行时功能测试）；mobb.ai 原文页一手打开（URL 四连字符，437 行全文：22,511 技能、140,963 发现、66.1% 零发现、33.9% ≥1 发现、27% 命令执行、14.8% 同意绕过、1,986 curl|sh、37 MCP 自动批准、1 起劫持）；AAIF 治理文本进展（GOVERNANCE.md #648，2026-11-04：承诺 tagged releases + changelog，未来时态；编号 Specification Proposal 轨道已定义、未建立；仍无 SECURITY.md；规范页戳仍 2026-08-04）；OpenCode 仓库迁移 sst/opencode → anomalyco/opencode（API 301），v2.2.6 published_at 2026-11-06T23:37:20Z；Cursor CLI 每条日期解决（.md 版官方 changelog 1,193 行一手：技能改进三条 → v2026.08.11；符号链接 → v2026.06.22；/skills 标志 → v2026.10.08；CLI 最新仍 v2026.10.22；IDE 11 月发布“无技能内容”仍仅二手）；Windsurf 迁移向导日期解决（.md 版官方桌面 changelog 3,976 行一手：向导 v3.6.22，2026-07-29；命令入口 v3.6.21 同日；Cascade 移除 v3.9.19，2026-09-08；Releasebot 2026-09-29 证伪；Cyber Toolchain 2026-07-30 为简报刊出日）；Skill Crossroads 仍 2026-09-10 rubric v1.2（复核戳推进）；SkillzBench 计数刷新（875 平、实测 152→151 无解释 −1、试验 20,425、扫描 2,907、发现 2,529、仓库 108、已评 2,866、proven 44/85、featured Trust 90→89）；Google Cloud 两篇技能评估博客（2026-11-03 五维评估含 skill-reach，2026-11-06 AlphaEvolve+Harbor 优化器，Terminal-Bench 2.0 +5%、+33 分，厂商自报）与 Octomind 跨宿主可移植性（2026-11-02，二手：Codex 2% 描述索引上限）；Codex rust-v0.169.0-alpha.2（2026-11-06，空 body）；Gemini CLI 仍 v0.67.0（复核戳推进）。
- 已同步进 skill 参考：skill-authoring.md 更新 Claude 2.1.319 一手段（+ UserPromptSubmit 钩子句）、UserPromptExpansion 功能首发版段、mobb.ai 原文一手段、AAIF 治理进展段、OpenCode 仓库迁移段、Cursor CLI 每条日期段、Windsurf 迁移向导日期段、Skill Crossroads 复核戳段、SkillzBench 计数刷新段、评估方法新增 Google 两篇博客与 Octomind 跨宿主段、Codex alpha / Gemini 复核戳段。
- 推翻的旧说法：“UserPromptExpansion 功能首发版本未验证”；“mobb.ai 原文页 URL 未取得、文本未打开”；“Windsurf 迁移向导日期未决”；“Cursor CLI 每条日期未决”；“Claude 2.1.319 版本号二手归属”；“mobb.ai 原文仍未打开”（上一轮限定）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 运行时测试（2.1.116 实机触发）。2. Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版。3. skill-reach 仓库版本/日期。4. Cursor IDE 11 月发布一手复核（主 changelog 直开）。5. AAIF 落地文件持续检索（规范页戳仍 2026-08-04）。6. Skill Crossroads 下轮复核报告页首行版本/日期。7. 已解决：UserPromptExpansion 功能首发版本；mobb.ai 原文页 URL 与全文；Windsurf 迁移向导日期；Cursor CLI 每条日期；Claude 2.1.319 published_at 一手；OpenCode 仓库迁移。

## 2026-11-07 轮次小节（晚间轮，版本 0.12.0；六路并行之技能两路）

- 本轮增量：Claude Code 2.1.320 未出现，最新仍 v2.1.319（2026-11-06）；技能条目补齐第四条（VSCode/JetBrains 面板发送自定义技能漏带打开文件，官方 changelog L1516 一手确认；下午轮记了 3 条）；另有一般变更：`permissions.denyInsteadOfPrompting`、MCP 任务转后台、自动模式收紧、`claude -p` 拒动退出码改 1；Gemini CLI 仍 v0.67.0；仅 v0.68.0-preview.0（2026-11-04T19:49:22Z），新增扩展校验命令 #29980（下轮跟进）；Codex 仍 rust-v0.167.0；alpha 线 0.168 已至 alpha.7、0.169 仅 alpha.2（body 均为空；alpha.7 存在性引自二手汇总，待补核 tag 页）；skill-reach：文档站 + Google Cloud 博客（2026-11-03）已一手通读；仓库确为 `google/skill-reach`（创建 2026-09-04T18:44:20Z，最后推送 2026-11-06T16:25:32Z），releases/tags 均为空——有仓库、无版本化发布；SWE-Skills-Bench v2（arXiv:2603.15401，修订 2026-10-27）：98 技能、3,920 次配对评估、92/98 无显著效果、平均 76.6%→76.9%（+0.3pp）、最高 +17.7 / 最低 −17.0；v1 口径（49 技能/+1.2%）作废；Cursor IDE 11 月发布（Build Live Apps 11-05、Voice Chat with Agents 11-06）两条均无技能内容——本轮 A 路一手打开 cursor.com/changelog 复核，取代下午轮“只有 Havoptic 二手”注记；官方页文本不显示日期与内部版本号（日期/版本号仍二手）；Skill Crossroads 仍 2026-09-10 rubric v1.2；SkillzBench 快照 2026-11-07 23:40 CST：875 持平、151 持平、试验 20,433（+8）、扫描 2,909（+2）、已 judge 2,868（+2）；152→151 下降原因仍无解释；AAIF：规范页戳仍 2026-08-04T22:53:14.470Z；Releases/Tags 仍为 0；最近提交仍为 2026-11-04 #648/#649；全树 201 文件仅 GOVERNANCE.md 命中关键词，无 SECURITY.md；三件落地文件均未出现；OpenCode：无 v2.2.7+；v2.2.6 发布说明全文（14,188 字符）0 处 "skill"；Windsurf 仍 v3.12.17（2026-11-06）；Cursor CLI 仍 v2026.10.22；UserPromptExpansion：无实机环境，运行时测试仍未决；Hooks 参考页重读（3,531 行）无措辞变化。
- 已同步进 skill 参考：skill-authoring.md 更新 Claude 2.1.319 四条技能段、Gemini preview #29980 段、Codex alpha 段、skill-reach 仓库身份段、SWE-Skills-Bench v2 段（推翻 v1）、Cursor IDE 一手复核段、Skill Crossroads + SkillzBench 晚间快照段、AAIF 戳推进段、OpenCode v2.2.6 全文 0 命中段、Windsurf/Cursor CLI 戳推进段、UserPromptExpansion 文档重读段。
- 推翻的旧说法：SWE-Skills-Bench v1 口径（49 技能/+1.2%）；Cursor IDE 11 月发布“无技能内容结论只有二手”；Claude 2.1.319“仅三条技能条目”（补齐第四条）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 运行时测试（2.1.116 实机触发）。2. Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版；Codex alpha.7 tag 页补核。3. AAIF 落地文件持续检索（规范页戳仍 2026-08-04）。4. Skill Crossroads 下轮复核报告页首行版本/日期。5. 已解决：Claude 2.1.319 第四条技能；skill-reach 仓库身份；SWE-Skills-Bench v2；Cursor IDE 11 月发布一手复核。

## 2026-11-08 轮次小节（晨轮，版本 0.13.0；六路并行之技能两路）

- 本轮增量：Codex 0.168.0-alpha.7 存在性由二手升级为一手证实（published 2026-11-05T22:55:03Z）；alpha 线现为 0.169.0-alpha.3（2026-11-07T19:31:41Z）；SkillSpector v2.13.4 一手定稿（2026-11-04T22:40:20Z；v2.13.3 2026-11-02），「71/17」须按版本锚定（早期 64/16、指南称 68），发布物只在 GitHub、不发布到 PyPI；新论文两篇：Sanko & Tian 监督松绑指令（arXiv:2610.37734，2026-10-27：153 万技能中约 1/17 含松绑指令，70.1% 无条件，扫描器检出 ≤46% 精度 <0.4；扫描器版本号与 κ=0.82 为二手）、Kan 等 ETUDE 工具更新缺口（arXiv:2611.01976，2026-11-01：97.6% 成功率，95.9% 合法任务完成率；七框架完整名单仅三家一手，余四家二手待正文核对）；SkillzBench 快照刷新（2026-11-08 05:39 CST：875 持平、实测 151→155、试验 20,433→20,606、扫描 2,909→2,918、已 judge 2,868→2,877、proven 44/85→43/84，波动无解释）；Skill Crossroads 仍 2026-09-10 rubric v1.2（复核戳推进）；AAIF 规范页仍 168 行、页脚戳仍 2026-08-04T22:53:14.470Z，Releases/Tags 仍均为 0，三件落地文件仍未出现，最近提交复核本轮受阻（GitHub API commits 端点回 HTTP 403 匿名限流），沿用上一轮已核实状态（2026-11-04 #648/#649），下轮优先补核；Gemini CLI 仍 v0.67.0 稳定版，仅 v0.68.0-preview.0（含 #29980，preview 线专属）；Claude Code 仍 v2.1.319（无 2.1.320+）；OpenCode 仍 v2.2.6；Windsurf 仍 v3.12.17（证据为二手追踪站三方一致，未重开官方 changelog 一手页）；Cursor CLI 仍 v2026.10.22；Cursor IDE 页首仍为 11-06 Voice Chat；SWE-Skills-Bench v2 无新版；skill-reach 仍无版本化发布（pushed_at 仍 2026-11-06T16:25:32Z）；Google Cloud 博客 11-06 之后无后续；UserPromptExpansion 运行时测试仍未决（本机无 Claude Code CLI，`which claude` 无输出，`~/.claude` 不存在）。
- 已同步进 skill 参考：skill-authoring.md 新增 Sanko 监督松绑论文段、Kan ETUDE 论文段、SkillSpector v2.13.4 版本锚定段；修正 Claude Code/Codex/Gemini/Cursor 复核戳段；修正 SkillzBench 快照段、Skill Crossroads 复核戳段、AAIF 复核段、skill-reach 复核段。
- 推翻的旧说法：Codex alpha.7「存在性为二手」（升级为一手证实）；SkillSpector「71/17 为稳定规格」（须按版本锚定）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 运行时测试（2.1.116 实机触发）。2. Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版。3. AAIF 落地文件持续检索（最近提交复核下轮优先补核）。4. Skill Crossroads 下轮复核报告页首行版本/日期。5. SkillzBench 计数波动解释。6. Windsurf 官方桌面 changelog 一手复核。7. Kan 等七框架完整名单与 Sanko 论文细节（下轮读正文）。8. Liu 等 arXiv:2605.14460 原文核实。9. 已解决：Codex alpha.7 一手证实；SkillSpector v2.13.4 定稿；SkillzBench 快照刷新。

## 2026-11-08 轮次小节（中午轮，版本 0.14.0；六路并行之技能两路）

- 本轮增量：ETUDE 七框架完整名单一手解决（Gemini CLI 0.62.0、Codex CLI 0.159.3、Claude Code 2.1.284、DeepSeek Harness 0.2.0-rc.2、OpenCode 1.18.34、Pi 0.86.1、Goose 1.45.0）；97.6%/95.9% 摘要数聚合口径未在正文写明，仍未决；DSH 0.2.0-rc.2 在评估之列。Sanko & Tian 全部锚定（扫描器 Cisco 2.2.1 / SkillSpector 2.12.0；κ=0.82 为第一作者与 LLM 第二标注者在 400 单元 gold set 上三分类标签一致度；5.8%≈88,000 为检测器外推估计、更可能低估；70.1% 分母为松绑指令单元；「最多检出 46%、精度<0.4」指单个扫描器最佳配置 SkillSpector+LLM 召回 0.46/精度 0.27）。Liu 等 SCH 无载荷语义攻击（arXiv:2605.14460，2026-05-14）：机密泄露 77.67%、RCE 67.33%（最脆弱配置），散文式载荷检出率 0.00%；与 Sanko、Kan 构成互补三角。SkillzBench 计数刷新（875/158/20,693/2,919/2,878，proven 43/84，来源仓库 108）；方法页每日队列机制解释小幅递增。AAIF 最近提交复核缺口关闭（#648/#649，2026-11-04 一手确认）；规范页戳仍 2026-08-04；三件落地文件仍未出现。Codex 0.167.0 技能条目精确化（#58134、#58130 两条边际）；OpenCode v2.2.6 全文 0 命中再确认；Claude 仍 2.1.319、Gemini 仍 v0.67.0、Cursor 11 月两条发布无技能内容、skill-reach 仍无版本化发布；UserPromptExpansion 运行时测试仍无实机环境。轮间补漏论文五篇——Zep 2611.01386（+4.5–18.8pts）、JitSkill 2611.01701、SCOPE 2611.03116（+3.9%–18.6%）、良性技能失齐 2611.04299（+10.3pp/+10.3pp）、野外技能 2604.04323（57.7%→65.5%）；严格窗口内无新技能论文（arXiv 周末无公告）。**仅记本笔记、不入技能正文**：Windsurf 官方 changelog 一手复核受阻——页面返回陈旧内容 Wave 13/1.13.3，但页脚戳为 2026-11-07T22:40:18Z，内容与更新戳矛盾；二手追踪器称 v3.12.28（2026-11-07），下轮优先重核。
- 已同步进 skill 参考：skill-authoring.md 新增 ETUDE 七框架名单段、Sanko & Tian 锚定段、Liu 等 SCH 论文段、轮间补漏五篇论文段；修正 SkillzBench 快照段、AAIF 最近提交复核段、Codex 0.167.0 技能条目段、Claude/Gemini/Cursor/skill-reach 复核戳段。
- 推翻的旧说法：Sanko 扫描器版本号与 κ=0.82「为二手」（升级为一手正文核对）；Kan 七框架名单「仅三家一手」（七家全部一手解决）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 运行时测试（2.1.116 实机触发）。2. Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版。3. AAIF 落地文件持续检索。4. Skill Crossroads 下轮复核报告页首行版本/日期。5. SkillzBench 计数波动解释。6. Windsurf 官方桌面 changelog 一手复核（本轮受阻，下轮优先重核）。7. Kan 等 97.6%/95.9% 聚合口径；Liu 等 arXiv:2605.14460 全文核实。8. 已解决：ETUDE 七框架完整名单；Sanko & Tian 全部锚定；AAIF 最近提交复核缺口；SkillzBench 计数刷新。

## 2026-11-08 轮次小节（晚间轮，版本 0.15.0；六路并行之技能两路）

- 本轮增量：Windsurf 矛盾消解：windsurf.com/changelog 为 308 重定向至 docs.devin.ai/desktop/changelog，dateModified 2026-11-07T22:40:18.633Z，顶部为 v3.12.28（2026-11-07），该节 0 处 "skill"；上一轮抽取只返回页尾旧条目；v3.12.17 未见于官方稳定页（小缺口）。ETUDE 97.6%/95.9% 聚合口径仍未决（全文检索穷尽：两数仅摘要与 §1 各 2 命中，"aggregate" 0 命中，试算无法复现）。Liu 2605.14460 全文核实：77.67% = GPT-5.4 mini × Codex C-Full 峰值，67.33% = MiniMax-M2.7 × OpenClaw I-RCE 峰值（不同配置），0.00% = 初始散文式载荷在 SkillScan 与 LLM Guard 上的检出率。Skill Crossroads 仍 rubric v1.2；SkillzBench 快照 875/161/20,875/3,021/2,978/2,627/112，proven 43/84；arXiv 周末无公告。AAIF 最近提交 #648/#649 一手确认（关闭晨轮 403 缺口），规范页戳仍 2026-08-04，三件落地文件仍未出现；UserPromptExpansion 运行时测试仍无实机环境（which claude 无输出，~/.claude 不存在）；宿主版本戳（Gemini 0.67.0、Codex 0.167.0/alpha.3、Claude 2.1.319、OpenCode v2.2.6、Cursor 11-06、skill-reach 无发布）。**仅记本笔记、不入技能正文**：ETUDE 97.6%/95.9% 聚合口径（下轮只查 v2）；Cursor Agent SDK GA 日期（2026-10-20）与包名（`@grok/agent-sdk` vs 文章所称 `@cursor/agent-sdk`）二手多方一致，未一手确认；Skill Comply、agents-team Evaluator 为未定日期线索；OpenCode #60315 归属 ≤ v2.2.3 未钉死；Windsurf v3.12.17 未见于官方稳定页（下轮留意是否重现）；GitHub REST API 匿名限流（共享出口 IP）致本轮部分结论改道 PyPI/npm/git 协议/页面文本核对。
- 已同步进 skill 参考：skill-authoring.md 追加 Windsurf v3.12.28 纠正段、Liu 三数字锚定段、SkillzBench 新条目（快照段）、AAIF/宿主版本/UserPromptExpansion 复核戳段。
- 推翻的旧说法：Devin Desktop「仍 v3.12.17」（官方最新稳定版为 v3.12.28）；Liu「最脆弱配置」单数表述（两峰值属不同配置）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 运行时测试（2.1.116 实机触发）。2. Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版。3. AAIF 落地文件持续检索。4. Skill Crossroads 下轮复核报告页首行版本/日期。5. SkillzBench 计数波动解释。6. ETUDE 97.6%/95.9% 聚合口径（只查 v2）。7. Cursor Agent SDK 一手确认。8. OpenCode #60315 归属版本。9. Windsurf v3.12.17 是否重现。10. 已解决：Windsurf 官方最新稳定版确认；Liu 三数字全文锚定；AAIF 403 缺口关闭；SkillzBench 快照刷新。

## 2026-11-08 轮次小节（深夜轮，版本 0.16.0；六路并行之技能两路）

- 本轮增量：Cursor Agent SDK `skills/` 机制一手确认（官方 Cursor 文档：TypeScript API、skills/ 目录为一等特性、示例 `packages/cli/skills/`、createAgent skills 选项（字符串名或 Skill 对象 name/description/instructions/path/metadata/allowed-tools）、Skill 类型导出；GA 日期仅二手 2026-10-20，无一手日期戳；`@cursor/agent-sdk` 公共 npm 上架未确认，GitHub release 资产名提及 agent-sdk@1.0.0 但非注册表记录）。OpenCode #60315 归属钉死（官方 release notes 一手：v2.2.3 管理策略拆分 `integration.use` → `mcp.use`/`skill.use`/`plugin.use`，v2.2.4 `--agent-skill-enabled` 选项，PR #60051）。Windsurf v3.12.17 官方稳定页仍未列（全文 1.5 MB 零命中，顶部仍 v3.12.28，页戳不变；v3.12.28 节 0 处 "skill"）。SkillzBench 快照刷新（23:39 CST：875/161/20,920/3,030/2,987/2,639/112，proven 43/84；当日试验 +42 而已发布/已测结果不动，波动仍无解释）。复核戳：ETUDE 97.6%/95.9% 仍仅 [v1]（方法页与附录只列 v1，论文正文引 97.6%/95.9%）；UserPromptExpansion 运行时测试仍无实机环境（`which claude` 无输出，`~/.claude` 不存在）；arXiv:2611.04456《How Agent Skills Evolve》（2026-11-04，23 技能纵向研究四维度，结论：指令越长错误越多、工具调用指令随模型升级变脆）列入下轮评估。
- 已同步进 skill 参考：skill-authoring.md OpenCode 2026-10-30 组段追加归属纠正、SkillzBench 段追加 23:39 快照、Windsurf 段追加再核结果、文件末尾追加 Cursor Agent SDK 新段与深夜轮复核戳段。
- 推翻的旧说法：OpenCode v2.2.3/v2.2.4 归属「二手镜像数据，非官方标题」（现官方 release notes 钉死）；Cursor Agent SDK「GA 2026-10-20 与包名 @cursor/agent-sdk 为二手多方一致」（GA 日期仍二手，但 skills/ 机制已一手确认；包名仍未确认）。
- 剩余未核实缺口（更新）：1. UserPromptExpansion 运行时测试（2.1.116 实机触发）。2. Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版。3. AAIF 落地文件持续检索。4. Skill Crossroads 下轮复核报告页首行版本/日期。5. ETUDE 97.6%/95.9% 聚合口径（只查 v2）。6. Cursor Agent SDK GA 日期一手日期戳。7. `@cursor/agent-sdk` 公共 npm 上架与否。8. arXiv:2611.04456 下轮评估。9. SkillzBench 计数波动解释。10. Windsurf v3.12.17 是否重现。11. 已解决：Cursor Agent SDK skills/ 机制一手确认；OpenCode #60315 归属版本；Windsurf v3.12.17 再核。

## 2026-11-09 晨轮（第 17 轮，窗口 2026-11-08 23:37 → 2026-11-09 05:37）

- 本轮增量 S1–S6（6 条）；版本 0.16.0 → 0.17.0。
- S1：arXiv:2611.04456《How Agent Skills Evolve》全文一手收录（Huang 等，v1 2026-11-03，仅 [v1]）：84,200 技能 / 7,776 仓库 / 750,898 提交（非「23 技能」、非「四个维度」，结构为 RQ1–RQ4 + 5 条启示）；核心组件元数据/工作流/知识在 99.9%/98.2%/97.8% 的技能中，资源 50.6%、脚本 14.6%；仅 35.2% 有验证步骤、49.6% 有故障恢复指引、28.9% 缺触发条件；工作流+知识+元数据占组件变更 86.0%（工作流 40.0%）；完善性维护占演进动机 50.37%；安全增强提交扫描确认风险降低平均仅 13.01%（Cisco 15.2% 降 / Snyk 10.8% 降，N=876）。数字均为论文自报。
- S2：**任务书勘误（修正）**：第 17 轮任务书关于 arXiv:2611.04456 的三处说法（「23 技能纵向研究」「指令越长错误越多」「工具调用指令随模型升级变脆」）全文 0 命中，无依据；最近似的另一篇论文 2610.04832 §5.1.5 说的是相反结论（新老模型代际收益相近），§5.1.6 是「技能填满窗口时收益缩水」（不同主张）。「变脆」措辞永不作为技能主张进入正文；skill-authoring.md 第 113 行尾句与本轨第 16 轮小节尾句均已纠正。
- S3：Cursor Agent SDK GA 日期升级为一手——官方博客发布于 2026-10-20T12:00:00Z，"The Agent SDK is generally available today."
- S4：`@cursor/agent-sdk` 不在公共 npm 上（注册表 404，一手）；实际包为 `@grok/agent-sdk`，最新 0.6.0（**发布时间戳尚未取得，新暂定子项**）；官方博客/更新日志安装示例仍写 `@cursor/agent-sdk`（文档缺陷观察，不入技能正文）。
- S5：SkillzBench 快照刷新——已发表 875、已测效果 163→167、试验 21,041→21,166、扫描 3,024→3,026、已判 2,981→2,983；已发现 2,627、仓库 112、已证 43/84 不变；方法论页不变（trust-2026-10-15.1，每日队列文案完整，仍无波动解释）。注：已发现 2,627 低于上一轮 23:39 快照的 2,639，计数器非单调，记为波动注记，非回归。
- S6：Gemini CLI #29980 升级为一手（预览版发布说明正文内）；AAIF HEAD #648 一手确认（提交 83021e2，"AAIF Donation: Add governance and maintainers"，2026-11-04 采纳；提交列表页提取为陈旧/渲染滞后，已弃用）。
- 窗口内无新技能论文（arXiv 最新公告日仍为 2026-11-06 周五；周一 2026-11-09 批次核查时尚未发布，下一轮补填）；ETUDE 仍仅 [v1]（API + 摘要页双重确认）；Skill Crossroads 仍 rubric v1.2（2026-09-10 生成，页面字节一致）；各宿主稳定版全不变（Gemini CLI v0.67.0、Codex rust-v0.167.0、Claude Code v2.1.319、OpenCode v2.2.6、Cursor CLI v2026.10.22、Windsurf v3.12.28）。

### 剩余未核实缺口（本轨）

- ETUDE 97.6%/95.9% 聚合数字仍未解决（仍仅 [v1]）。
- `@grok/agent-sdk` 0.6.0 发布时间戳已解决（2026-11-09 中午轮：2026-11-03T16:45:14.613Z，npm 注册表一手；S4 暂定已划除）。
- arXiv 周一 2026-11-09 批次补填（下一轮）。
- UserPromptExpansion 实机测试仍无法本地执行（无 Claude 环境）。
- Windsurf v3.12.17 连续第四轮零命中（2026-11-09 中午轮：官方稳定页顶部现为 v3.12.32；v3.12.28 未见于 .md 版，异常已记录）。

## 2026-11-09 中午轮（版本 0.18.0；窗口 2026-11-09 05:37 → 11:37 CST）

- 本轨增量（S1–S7）：`@grok/agent-sdk` 0.6.0 时间戳一手闭合（S4 暂定划除）；Windsurf v3.12.32 窗口内新稳定版（2026-11-08，.md 一手，0 处 "skill"；v3.12.28 未见于 .md、内容与 v3.12.32 逐条相同，异常如实记录）；Cursor CLI 安装脚本现供 2026.11.05-86047df（一手）；SkillzBench 快照刷新（875/169/21,318/3,027/2,984/2,627/112，proven 43/84）；ETUDE v2 前提撤回（仅 v1，97.6%/95.9% 已在 v1 摘要中）；周五批次技能论文六篇（ENGRAM、LoRA→Language、Feedback Backdoor、Skillware、DelegateEval、DefectSkills）；UserPromptExpansion Hooks 页重构（3,531→4,303 行，核心措辞不变，仍无实机环境）。
- 剩余未核实缺口：arXiv 周一 2026-11-09 批次补填（下一轮；More with Less、From Procedures to Permissions 两篇编号待确认）；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现；ETUDE 聚合口径读 v1 方法节；Gemini v0.68.0 / Codex 0.168.0/0.169.0 / OpenCode v2.2.7 / Claude v2.1.320；UserPromptExpansion 实机测试仍无法本地执行（无 Claude 环境）；Skillware 方法/仓库核查；SWE-Skills-Bench v2 页重核。

## 2026-11-09 晚间轮（版本 0.19.0；窗口 2026-11-09 11:37 → 17:37 CST）

- 本轨增量（S1–S10）：More with Less arXiv:2611.08947v1（token −52.4%）与 From Procedures to Permissions arXiv:2611.08905v1（+13.4pp/+24.1pp/+11.6pp）编号确认；ETUDE 方法节已读（v1 §5 规则 + 各节均值 RQ1 100.0%/99.6%、RQ2 96.4%/94.0%、RQ3 91.8%），摘要 97.6%/95.9% 仍无法复现（未决；"check v2" 前提已死）；Skillware 方法/仓库核查（三臂 58.2%→71.4%→91.8%、97.3% 为跨层比较、84.8% 与 18.5% 未决，仓库 v0.6.1，180 stars/78 forks/MIT）；SWE-Skills-Bench 仍 v2（2026-10-27）；四篇评估方法论文摘要级（2611.09030 因子设计、RuntimeTickets、EvoGuard、SKILL2STEER wrong-skill 对照）；SkillzBench 快照刷新（875/171/21,392/3,028/2,985/2,628/113，proven 43/84；已发现 2,628 仍低于 11-08 23:39 的 2,639）；Windsurf "skill" 命中数订正 11 → 12 处（计数遗漏，文件逐字节相同；v3.12.28 连续第五轮未重现，v3.12.17 亦零命中）；Codex alpha 线至 rust-v0.169.0-alpha.3；周五批次三篇仍 v1 + 宿主版本戳（Claude 2.1.319、Gemini 0.67.0、OpenCode v2.2.6、`@grok/agent-sdk` 0.6.0、`@cursor/agent-sdk` 仍 404）；UserPromptExpansion 仍无本地 Claude 环境（`which claude` 退出码 1，无 `~/.claude`）。
- 剩余未核实缺口：ETUDE 聚合公式（v1 方法节已读，仍无公式；待 v2 或作者澄清）；Skillware 84.8%/18.5% 文本溯源；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现；Gemini v0.68.0 / Codex 0.168.0/0.169.0 / OpenCode v2.2.7 / Claude v2.1.320；UserPromptExpansion 实机测试仍无法本地执行（无 Claude 环境）；SWE-Skills-Bench 代码库 URL 逐字转录。

## 2026-11-09 深夜轮（版本 0.20.0；窗口 2026-11-09 17:37 → 23:37 CST）

- 本轨增量（S1–S9）：ETUDE 仍仅 v1（8 种试算均无法复现 97.6%/95.9%，作者公开渠道已穷尽，邮件联系未尝试）；Skillware 84.8%/18.5% 文本溯源（84.8% 全文仅摘要 1 处、正文最大 84.0%；18.5% 三处原文逐字定位，双重分母确认）；SWE-Skills-Bench 代码库 URL 逐字转录闭合（论文内链接与摘要页 href 逐字一致，仓库真实存在，8 stars）；周一批次 16 篇枚举（核心 5 篇、机器人相邻 5 篇、误命中 6 篇，全部 v1）；Gemini #29980 点名项闭合（已关闭的扩展验证命令 PR，与 preview 线无关）；Windsurf "skill" 计数口径订正（行 12 / 子串 17，v3.12.28 第六轮未重现）；SkillzBench 快照刷新（875/175/21,558/3,032/2,989/2,628/113，proven 43/84；已发现 2,628 仍低于 11-08 23:39 的 2,639，计数器非单调）；复核戳：四宿主无新版、AAIF 仍无落地文件（规范页字节 +161 观察项）、Skill Crossroads 仍 rubric v1.2、Cursor 映射仍未找到、UserPromptExpansion 仍无实机环境。窗口内新稳定发布为零。
- 纠正：Gemini #29980「preview 线相关」（实为已关闭的扩展验证命令 PR）；Windsurf "skill" 命中数「12 处」（口径明确为两数并记：行 12 / 子串 17）。
- 剩余未核实缺口：ETUDE 聚合公式（邮件联系是否尝试由主轮定）；Skillware 作者澄清；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现；Gemini v0.68.0 / Codex 0.168.0/0.169.0 / OpenCode v2.2.7 / Claude v2.1.320；UserPromptExpansion 实机测试仍无法本地执行（无 Claude 环境）；AAIF 落地文件持续检索；SkillzBench 计数波动解释。

## 2026-11-10 凌晨轮（版本 0.21.0；窗口 2026-11-09 23:37 → 2026-11-10 05:37 CST）

- 本轨增量（S1–S7）：Claude Code v2.1.321 窗口内发布（2026-11-09T19:22:30Z，6 项技能相关：worktree 恢复/菜单两修复、Bamum 字母名拒收、dataviz 技能更新、嵌套技能加载收紧——`~/.claude/plugins` 内 `.claude/skills` 不再加载、headless 热重载 `CLAUDE_CODE_PLUGIN_DIR_WATCH=0`；**v2.1.320 不存在**，tag 404，跳号原因未说明）；Codex rust-v0.169.0-alpha.6 窗口内预发布（2026-11-09T19:25:26Z，25 字符 body，0 技能命中；稳定版仍 rust-v0.167.0）；SkillzBench 快照刷新（875/177/21,712/3,041/2,998/2,628/113，proven 43/84；discovered 仍 2,628，低于 11-08 23:39 的 2,639，非单调未解释）；ETUDE/Skillware 证据更新（仍仅 v1；新增 Vibing Talk 论坛渠道，无澄清；邮件联系未尝试）；同名「Skillware」第二篇消歧（Fan & Lan arXiv:2607.18970v3，软件本体论，138,133 份 SKILL.md；两篇对名称起源说法冲突，仅登记不裁决）；Skill++ 边界线索（arXiv:2611.00544，路由感知技能精炼，76.3% vs 36.4%，摘要级）；复核戳：AAIF 三件落地文件仍未出现（规范页字节回落 262,221 → 262,060，证实 +161 为动态波动，观察项关闭）、Windsurf 仍 v3.12.32（v3.12.28 第七轮未重现）、Cursor 映射仍未找到（Agent CLI 文档页本轮 403）、SWE-Skills-Bench 仍 v2、Gemini 仍 v0.67.0、UserPromptExpansion 仍无实机环境（Hooks 页逐字节相同）。窗口内新稳定发布两件（Claude 2.1.321、Codex alpha.6 预发布）；另有窗口外发布一件登记：Codex rust-v0.169.0-alpha.7（2026-11-09T21:55:10Z，窗口关闭后 18 分钟，下轮起点）。
- 纠正：AAIF 规范页「+161 字节疑似更新」（证实为动态波动，观察项关闭）。
- 剩余未核实缺口：ETUDE 聚合公式（邮件联系是否尝试由主轮定）；Skillware 作者澄清（邮件联系是否尝试由主轮定）；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版 / OpenCode v2.2.7 / Claude v2.1.322+；UserPromptExpansion 实机测试仍无法本地执行（无 Claude 环境）；AAIF 落地文件持续检索；SkillzBench 计数波动解释；Skill++ 全文深读。

## 2026-11-10 中午轮（版本 0.22.0；窗口 2026-11-10 05:37 → 11:37 CST）

- 本轨增量（S1–S8）：Skill++ arXiv:2611.00544v1 全文深读（路由感知技能精炼：仅改写名称 + 描述，≤220 字符、≤8 条排除短语、ℓ_max=298；SRA-Bench FC@5 S9 76.3% vs Vanilla 36.4%、+39.9pp、五路均值 +23.1pp；Codex harness 均值 +34.3pp；误差分解与消融；局限：纯文本、单一添加顺序、模型裁判、无公开代码仓、2026 模型名未核实）；Claude Code v2.1.322 窗口内发布（2026-11-10T02:29:43Z，仅 Vertex fast mode，0 处「skill」；v2.1.320 仍不存在）；Codex alpha 线（alpha.7 预发布、alpha.4.1 空 body；alpha.8 tag 已存在但无 release 对象；稳定版仍 rust-v0.167.0）；Windsurf/Devin Desktop v3.12.43 窗口内首次出现（0 处「skill」，.32→.43 跳号；v3.12.28 第 8 轮未重现）；SkillzBench 快照刷新（875/175/21,859/3,069/3,026/2,628/113，proven 43/84；measured-effect 177→175，第二次非单调下降；首页文本「43 of 84」读数行消失，aria-label 仍为 43/84）；Cursor /install 现按 UA 分流（浏览器 UA 307、curl UA 仍返回 6,333 字节脚本，版本仍 2026.11.05-86047df；方法注记：今后 CLI 版本检查须用 curl UA；Agent CLI 文档页 403→404）；Hooks 参考页 +1,463 字节 / +17 行（UPE 专节 L1565→L1573，核心措辞未变；新增 17 行内容未能定位）；11-07 批次技能论文摘要级登记 4 篇（2611.10830、BadEvolve 2611.10542、ReSkill 2611.10510、2611.10884）；复核戳：ETUDE/Skillware 仍仅 v1（84.8% 定位为摘要上限、97.3% 为 Fig 9、18.5% 双重读数；作者仓 10 条 open issue 已扫，无澄清）、Gemini 仍 v0.67.0、OpenCode v2.2.7 缺席（新仓最新 v2.2.6）、同名 Skillware 第二篇仍 v3、AAIF 三件落地文件仍缺席（规范页字节 262,221 → 256,176，仅观察项）、Skill Crossroads 仍 rubric v1.2、SWE-Skills-Bench 仍 v2、UserPromptExpansion 仍无实机环境。窗口内新稳定发布两件（Claude 2.1.322、Windsurf v3.12.43）。
- 剩余未核实缺口：ETUDE 聚合公式（邮件联系是否尝试由主轮定）；Skillware 作者澄清（邮件联系是否尝试由主轮定）；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现（已 8 轮）；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版 / OpenCode v2.2.7 / Claude v2.1.323+；UserPromptExpansion 实机测试仍无法本地执行（无 Claude 环境）；AAIF 落地文件持续检索；SkillzBench 计数波动解释；Hooks 页新增 17 行定位（下轮先留存副本再 diff）。

## 2026-11-10 晚间轮（版本 0.23.0；窗口 2026-11-10 11:37 → 17:37 CST）

- 本轨增量（S1–S8）：四宿主均无新稳定发布（Claude v2.1.323 未发布、v2.1.320 不存在、v2.1.321 body 复查 7 命中/6 条 + 9 项 hooks 修复；Codex 稳定版仍 rust-v0.167.0、alpha.8 tag 有/Release 404 第二轮、alpha.9 无、alpha.7 body 仅标题行；Gemini v0.68.0 未发布、稳定版 v0.67.0、nightly 20261104；OpenCode v2.2.7 tag-only，最新 Release 仍 v2.2.6）；SkillzBench 快照刷新（875/179/22,168/3,172/3,129/2,729/118，proven 43/84；measured-effect 第三次非单调移动 177→175→179；discovered 重回 2,639 高点之上；source repos 首次移动 113→118）；Windsurf v3.12.28 第 9 轮未重现（顶部仍 v3.12.43；skill 行 12 / 子串 17；字节 93,219 未变；副本已存供下轮 diff）；Hooks 页 +1,327 字节 / +10 行（4,330 行；UPE L1573→L1584，核心措辞未变；+17/+10 行仍未定位，+17 行为永久历史缺口；副本已存）；Cursor 复核（桌面 changelog 字节未变 183,637；CLI 仍 2026.11.05-86047df；Agent CLI 文档页仍 404；桌面↔CLI 映射仍未找到）；AAIF 三件落地文件仍缺席（规范页戳仍 2026-08-04，字节 256,176 未变）；Skill Crossroads 仍 rubric v1.2（字节 63,985 未变）；SWE-Skills-Bench 仍 v2；`@grok/agent-sdk` 仍 0.6.0、`@cursor/agent-sdk` 仍 404；ETUDE/Skillware/Skill++ 证据更新（均仍仅 v1；Skill++ 无公开代码仓记「未找到」；ARPAHLS/skillware 唯一新帖 #469 与数字无关；#471 状态未确认；Vibing Talk 帖仍 #1/#3/#11，均未提及 84.8%/18.5%；邮件联系未尝试）；技能论文摘要级登记 5 篇直接相关（SkillDuet 2611.12109、SMO 2611.12041、StealthyPoison 2611.12155、SkillCODER 2611.11308、SOP-skill 2611.11348；SMO "59.3×" 疑为 "59.3%" 笔误，仅引可核实 2.46×）；arXiv `all:skill` 查询 totalResults=25（9 篇 11-08 + 16 篇 11-09，全部 v1）；UserPromptExpansion 实机测试仍无本地环境（`which claude` 退出码 1、`ls ~/.claude` 退出码 2，执行者本轮重验）。
- 剩余未核实缺口：ETUDE 聚合公式（邮件联系是否尝试由主轮定）；Skillware 作者澄清（邮件联系是否尝试由主轮定）；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现（已 9 轮）；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版 / OpenCode v2.2.7 / Claude v2.1.323+；UserPromptExpansion 实机测试（无本地 Claude 环境）；AAIF 落地文件持续检索；SkillzBench 计数波动解释；Hooks 页新增行定位（下轮自本轮副本 diff，+17 行永久缺口已定）。

## 2026-11-11 凌晨轮（版本 0.24.0；窗口 2026-11-10 17:37 → 2026-11-11 01:09 CST）

- 本轨增量（S1–S8）：窗口内新稳定发布一件——OpenCode v2.2.7（2026-11-10T10:37:15Z，发布说明 "skill" 命中 1 次，Console 工作区策略修复 #62702；仓库更名 sst/opencode → anomalyco/opencode）；Codex alpha.9 tag 新增（2026-11-10T16:52:11Z），仍无 Release 对象（alpha.8 第三轮仍无）；Gemini nightly 自 20261104 后停更 6 天（原因未见公告）；SkillzBench 快照刷新（875/183/22,436/3,173/3,130/2,729/118，proven 43/84；measured effect +4）；Windsurf v3.12.28 第 10 轮未重现（副本与上轮逐字节相同；2,429 vs 2,430 行计数差异确认为口径差异，缺口闭合）；Hooks 页本轮零变化（副本逐字节相同，4,330 行；UPE 仍 L1584；+17/+10 行仍未定位）；ETUDE/Skillware/Skill++ 证据更新（均仍仅 v1；ETUDE 代码仓检索 total_count 0；Skill++ 摘要页 github 零命中；PR #471 确认仍 open 未合并，上一轮遗留缺口闭合；邮件联系未尝试）；AAIF/Crossroads/SWE-Skills-Bench/Cursor 复核戳（AAIF 规范页 256,176 字节未变；Crossroads 63,985 字节未变，仍 rubric v1.2；SWE-Skills-Bench 仍 v2；Cursor 桌面 changelog −538 字节为动态载荷漂移推断；Claude v2.1.323/324 未发布）。
- 剩余未核实缺口：ETUDE 聚合公式（邮件联系是否尝试由主轮定）；Skillware 作者澄清（邮件联系是否尝试由主轮定）；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现（已 10 轮）；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版 / Claude v2.1.323+；UserPromptExpansion 实机测试（无本地 Claude 环境）；AAIF 落地文件持续检索；SkillzBench 计数波动解释；Gemini nightly 停更原因。

## 2026-11-11 凌晨轮（版本 0.25.0；窗口 2026-11-11 01:09 → 04:09 CST）

- 本轨增量（S1–S6）：新登记 SkillCache（arXiv:2611.10022v1，2026-11-05；87 个 SkillsBench 任务 token −81% Sonnet 4.6 / −74% Opus 4.8；跨模型奖励 +12–25pp；「SkillsBench」与 SkillzBench 是否同一基准未核实，摘要级）；Gemini nightly 停更原因定位（nightly-release workflow 自 2026-11-05 起连续 6 天失败，自动新建 6 个 `priority/p0` + `release-failure` issue #30149/#30167/#30192/#30202/#30203/#30215，截至核查全部仍 open；issue 正文仅链 Actions 运行记录，无根因文字；底层根因仍未决；nightly 命名已越过 0.68.0 进入 v0.69.0-nightly）；AAIF 规范页 +6,094 字节（256,176 → 262,270，戳仍 2026-08-04，全文 5 处，无 2026-11 日期串；新 buildId/deploymentId `dpl_596ePT575z4qnAFksq1biRW97aLY`；判定为疑似站点重新部署、内容变化未确认；副本已留存供下轮 diff）；Hooks 参考页 +5 行（4,330→4,335 行，292,719→293,236 字节；新增/删除行已全部定位：L906–910 hook 启动长期后台程序段落，L913 错误文本新增 `stdio went quiet`；UPE 专节 L1584→L1589 内容逐字一致；上一轮 +17/+10 行未定位仍为永久历史缺口）；SkillzBench 快照 2026-11-11 04:13 CST（published 875→1,025、measured effect 183→184、trials 22,436→22,470、scans 3,173→3,181、Judged 3,130→3,138、Discovered 2,729 持平、source repos 118 持平、proven 43/84→42/83，厂商自报）；复核戳：四宿主零新发布（Claude 仍 v2.1.322，v2.1.323/324 均 404；Codex 稳定版仍 rust-v0.167.0，alpha.9 tag-only 第二轮，alpha.10 未出现；Gemini 仍 v0.67.0；OpenCode 仍 v2.2.7）；Windsurf v3.12.28 零命中连续第 11 轮未重现（副本逐字节相同）；Cursor 桌面 changelog 183,099 字节零漂移（CLI 仍 2026.11.05-86047df；版本实际出现 5 处，纠正上一轮计数 3）；ETUDE/Skillware/Skill++ 均仍仅 v1（ARPAHLS/skillware 纯 issue 仍 39 条、最新仍 #469；新 PR #472 与数字口径无关；PR #471 仍 open 未合并；Vibing Talk 帖仍 #1/#3/#11；邮件联系未尝试）。窗口内新稳定发布为零。
- 纠正：Cursor 版本出现次数「3 处」（实为 5 处，上一轮漏计两处 `ln -s` 行）；Gemini nightly 停更「原因未见公告」（原因已定位为 workflow 连续失败，底层根因仍未决）。
- 剩余未核实缺口：ETUDE 聚合公式（邮件联系是否尝试由主轮定）；Skillware 作者澄清（邮件联系是否尝试由主轮定）；Cursor 桌面/CLI 版本映射；Windsurf v3.12.28 是否重现（已 11 轮）；Gemini v0.68.0 / Codex 0.168.0/0.169.0 稳定版 / Claude v2.1.323+；UserPromptExpansion 实机测试（无本地 Claude 环境）；AAIF 落地文件持续检索；SkillzBench 计数波动解释；Gemini nightly 根因（6 个 p0 issue 动向）；SkillCache 全文深读。
