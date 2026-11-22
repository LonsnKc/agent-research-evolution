# 轨道：API 路由

基线来源：`~/workspace/research_notes/api-routing-harness-memory-skills-20261023-1442/report.md`（2026-10-23 四主题深研报告）与技能参考 `~/workspace/custom-skills/agent_system_design/references/api-routing.md`。

基线要点（2026-10-23）：
- 四层模型：A 传统网关（Kong、APISIX、Envoy、Traefik）/ B 服务副本路由（llm-d、vLLM Semantic Router、NVIDIA Dynamo）/ C LLM 网关（LiteLLM、Portkey、OpenRouter、Helicone、Kong AI Gateway、APISIX ai-proxy）/ D 学习式模型路由（RouteLLM、FrugalGPT 级联、Hybrid LLM、GraphRouter、MetaLLM、Martian、NotDiamond）。
- 机制：规则/权重、成本优先、延迟优先、质量优先、级联、预测分类器、bandit、语义路由、图路由、集成/融合、KV 感知；网关基础：冷却、RPM/TPM、预算、重试、缓存。
- 论文：RouteLLM（arXiv 2406.18665）、FrugalGPT（2305.05176）、Hybrid LLM（2404.14618）、RouterBench（2403.12031）、GraphRouter（2410.03834）、LLM-Blender（2306.02561）。
- 警示：星标数与厂商节省数字均为快照或厂商自报，使用前须在原仓库/论文页复核。

---

## 2026-10-25 轮次：网关安全事件周 + 决策型端点趋同

窗口结论：严格落在水位（2026-10-24）之后的公开动态极少；本轮“新进展”实质只有一条，其余为基线补漏，下文严格区分。

### 本窗口内（已核实，一手来源）

- **2026-10-24（CST）LiteLLM 1.107.0 投毒事件官方根因分析**：https://docs.litellm.ai/blog/security-incident-oct-21-2026-attack-chain 。2026-05-27 一份含 LiteLLM Agent Platform 主密钥的配置文件被误提交到公开仓库 `LiteLLM-Labs/lite-harness`，虽被 force-push 覆盖但未吊销；10-10 攻击者用该密钥拿到组织所有者关联的 GitHub CLI OAuth 令牌；10-21 绕过分支保护把发布源头指向仿冒仓库 `Berri-AI/litellm`，发布含窃密器的 `litellm==1.107.0`（PyPI 存活 11:39–13:23 UTC；Docker 标签 `v1.107.0 / latest / main-latest / main-stable` 存活 11:46–14:20 UTC）。官方确认 git tag 本身干净，是构建产物被替换。事发当周发布门禁由“授权工程师发起 + 办公室物理设备锁定账号审批”放宽为“任意两名工程师触发并审批”，攻击者正是用两个被盗令牌分别触发与审批。教训：层 C 网关集中持有全部上游密钥，其供应链安全（版本 pin + hash 校验、发布门禁、密钥隔离）应与路由算法同等权重。窃密器细节（`cook.pth` + `sitecustomize.py` 启动时执行、无需 import）与“安全版本 1.106.2”均为二手转述，未核实。

### 基线补漏（2026-10-02～10-23，已核实者标注一手来源）

- **2026-10-22 NVIDIA Dynamo v1.6.0 stable**（官方兼容矩阵页）：后端 pin SGLang 0.5.21 / TensorRT-LLM 1.3.0rc29 / vLLM 0.31.0，CUDA 13.0；Router policy YAML 优先于 scoring flags，后者已废弃、计划 v1.7.0 移除；vLLM worker 服务改名 worker / prefill / decode。纠正二手信息：v1.5.0 stable 实际为 2026-09-18（v1.5.1 为 10-06）。
- **2026-10-20 llm-d 1.0**（Google Cloud 官方博客一手确认，原为二手）：多租户 QoS、v1 稳定 API、Router 高可用与零停机升级、DisaggregatedSet、会话感知路由、异步推理处理器。性能数字（TTFT 降 85%+、吞吐 2.4 倍、缓存命中 35%→70% 等）均为厂商自报，未核实。
- **vLLM v0.32.0**（官方 release notes 一手；日期 10-18 为二手）：814 commits、363 contributors；新增 `POST /v1/systemone` 结构化决策端点，与 LiteLLM Decisions API、OpenAI Decisions 格式趋同——“决策型调用”正从聊天 API 中分化；NVFP4 KV cache 扩展；Python 3.10 停止支持；3 项 GHSA 安全修复。PR 内基准数字为厂商/PR 自报，未核实。
- **2026-10-14 OpenRouter Private Endpoints**（官方公告索引页一手）：企业客户自有微调模型与专属部署经 OpenRouter 路由、仅组织内可见；同期 Model Router Benchmarks（Router Index）发布于 10-02、Security Center 发布于 09-28。
- **LiteLLM v1.105.0（10-11）/ v1.106.0（10-18）/ v1.106.1（10-20）**（官方 release notes 一手）：Decisions API（`/v1/decisions`、`/v1/systemone`）、`litellm.agent()`、Lens 改为独立 Rust 服务、MCP 网关限流覆盖全部操作；官方 changelog 顶部横幅“LiteLLM is moving to Rust”。9-27 的 Fuse V2 路由与 Capability router 为补充线索（已核实存在，细节未深读）。
- **2026-10-21～22 Railway 安全事件**（Railway 官方博客经摘要读取）：攻击者访问并解密部分客户环境变量，目标据称为 AI provider 密钥；Railway 建议 OpenRouter 用户检查异常用量并轮换密钥。OpenRouter 自身被入侵一说来自暗网监测站，未证实，不得写成事实。OpenCode Console 10-17 事件（约 1.4% 用户密钥受影响）同样为二手，未核实。
- **RouteLLM / Portkey 本轮无新动态**：RouteLLM 自 2024-08 未维护的说法仍为二手且与营销文章冲突，维持“待核查线索”处理；Portkey 被 Palo Alto Networks 收购完成一说仅见 OpenRouter 对比页二手说法，未核实。标准/规范：Gateway API Inference Extension 仍是 llm-d/Dynamo 共享事实标准方向，无新版本公告。

### 深化：成本/延迟权衡定量模型

- 统一口径：单次成本 `C = p_in·N_in + p_out·N_out`（须区分名义 token 与计费 token，缓存命中压低有效 N_in）；延迟 `T = T_queue + TTFT + N_out·TPOT`（KV 命中压 TTFT，级联加 TPOT 段）；质量 Q 须以任务通过率等可测指标定义。
- 预测分类器：`E[C] = C_router + Σ P(m)·C_m`；关键参数为误分率 ε 与过分率 δ；质量损失 ≈ ε·(Q_强 − Q_弱)，成本浪费 ≈ δ·(C_强 − C_弱)；分布漂移使两者同时恶化，须按簇估计并设漂移告警。
- 级联：`E[C] = C_弱 + p_esc·C_强`，`E[T] = T_弱 + p_esc·T_强`（升级请求延迟为两者之和）；质量由裁判误差 η 主导，RouterBench 结论：η 超约 0.2 时级联质量快速下滑——级联上线前必须实测 η，任何可用阈值下 η 均超 0.2 则否决级联。Cluster, Route, Escalate（arXiv:2606.27457，2026-06-25，Moslem 等）：簇级打分 `Score(m,c) = Error(m,c) + λ·Cost_norm(m)`（TPOT 归一化成本），λ 读作“为用最贵模型可容忍的最大误差率惩罚”；论文自报保留最强模型准确率 97–99% 且降低 TPOT，未独立验证；仅需任务正确性标签，模型池变化时无需人工重配。
- Bandit：奖励 `r_m = α·Q_m − β·C_m − γ·T_m`（权重须事先声明）；冷启动探索成本、奖励延迟与隐式反馈偏差是主要失效条件。
- KV 感知路由：联合打分 `Score = a·prefix_hit_ratio − b·queue_depth − c·predicted_TTFT`；纯命中最大化导致热点过载，纯负载均衡浪费缓存；llm-d 1.0 把缓存亲和与流控/优先级放在同一策略里。厂商数字（命中率 35%→70% 等）仅作机制存在性佐证。
- 跨机制对照（同口径定性排序）：规则/权重成本与延迟最低、风险为系统性错分；预测分类器低成本低延迟、风险为漂移；级联成本中延迟高、风险为裁判误差；bandit 探索期高收敛后低；KV 感知降算力成本并大幅降 TTFT、风险为热点过载；融合（N 倍成本）仅作对照。
- 落地评估法（按推荐顺序）：离线回放 → 影子路由 1–2 周实测 ε、δ、η → 按任务簇分桶报告（防全局平均掩盖长尾簇全错的 Pareto 陷阱）→ 裁判校准曲线。补充论文：Cascade（arXiv:2608.06557，2026-08-06，per-request latency budget 联合调度，论文自报好put 提升最高 2.4 倍、SLO 违例降 40%，未验证）；云边多目标路由（arXiv:2507.15553v2，NSGA-II，论文自报质量达纯云基线 95.2%、延迟 +4.8%、成本 −34.9%，未验证）——数字一律不得跨论文横比。

### 已同步进 skill 参考

- api-routing.md：LiteLLM 安全事件注记、Dynamo v1.6.0 与 vLLM v0.32.0 条目、结构化决策路由机制行、Cluster Route Escalate 论文条目（数字标论文自报）、两条设计检查（裁判误差门、按簇报告）。无 SKILL.md 级变更。

### 剩余未核实缺口

1. LiteLLM 最新安全版本是否为 1.106.2（Wiz 等二手说法，官方 changelog 未列出）。
2. Dynamo v1.5.0 细节（custom Router scoring、Mooncake/disk KV indexing、guided tool call streaming）仅见二手日志站。
3. Portkey 收购完成时间与开源版后续（仅 OpenRouter 对比页二手说法）。
4. RouteLLM 维护状态（两二手说法冲突，需直接读仓库提交历史）。
5. Railway / OpenCode Console 事件实际影响范围（官方调查进行中；OpenRouter 被入侵一说未证实）。
6. 下轮若仍无新版本，建议把窗口放宽为“自基线以来”并明示，避免为凑数把旧闻写成新闻。


---

## 2026-10-27 轮次：缺口复核——RouteLLM 停滞、LiteLLM 钉版两段式、Dynamo 仍为 v1.6.0、Portkey 收购确认

### 已核实（一手为主）

- **RouteLLM 停滞**：PyPI 最新 0.2.0（2024-07-08）、仓库 0 release、未归档；生产选型不再以其为活跃推荐。最后提交日 2024-08-10 仅为第三方评测站说法（自述 2026-09-21 复核），提交历史页未直接打开，技能正文仅以「未核实」方式提及。
- **LiteLLM 钉版两段式**：事发当日官方口径钉 1.106.2 或更早版本；2026-10-24/25 回退线为 1.106.3/1.106.4（1.106.4 含代理启动 ImportError 修复，同修 1.104.7、1.105.5），自 1.106.3 起 Docker 镜像改用新 cosign 密钥，StepSecurity 建议钉 1.106.4；无 1.107.x 后续正式版。仅升级不够：删 loader 文件、停持锁进程、重建环境、轮换全部可达凭证。**等级**：1.106.3/1.106.4 为二手转述一手（部署追踪 + releases 镜像双重佐证，官方 releases 页未直接打开），已登记 PRACTICAL 未验证暂定清单。
- **Dynamo 仍为 v1.6.0 stable**（NVIDIA 官方文档一手）：其后仅 nightly/dev 构建；官方文档两处日期口径差一天（Oct 21 / Oct 22），版本结论不受影响。
- **Portkey 收购一手确认**：PANW 官方新闻稿（2026-05-29）确认完成收购，并入 Prisma AIRS；开源 Gateway 2.0 仅预发布分支、无正式版（后半句的「停滞」口径为第三方评测二手，不写入定论）。

### 已同步进 skill 参考

- api-routing.md：LiteLLM 安全注记扩展为两段式钉版 + 清理处置；新增 Portkey 收购行；RouteLLM 论文条目追加停滞注记；Dynamo 条目追加仍为最新 stable 注记。

### 剩余未核实缺口（更新）

1. LiteLLM 官方 releases 页直接确认 1.106.3/1.106.4（本轮未验证暂定项，下轮首要复核）。
2. RouteLLM 提交历史页直接确认最后提交日。
3. Dynamo v1.5.0 细节（仍仅见二手日志站）。
4. Portkey Gateway 2.0 正式版是否发布。
5. Railway / OpenCode Console 事件实际影响范围（仍未决）。
6. 已解决：LiteLLM 安全版本 1.106.2（事发当日成立）、Portkey 收购完成时间与官方口径。


---

## 2026-10-28 轮次：缺口复核 + 新增覆盖——LiteLLM 钉版确认、1.106.5、级联两面证据

### 已核实（一手为主）

- **LiteLLM 1.106.3 / 1.106.4 正式确认（上一轮未验证暂定项闭环）**：执行者直接打开官方 GitHub releases 页核对——1.106.3 批（1.103.8 / 1.104.6 / 1.105.4 / 1.106.3）起改用 commit `e82a3c6` 的新 cosign 密钥并移除 eMudhra 根 CA；1.106.4 批的代理启动 ImportError 修复（自 v1.104.0 起）仅覆盖 1.104.7 / 1.105.5 / 1.106.4，1.103.9 仅含镜像扫描修复。日期口径来自官方文档 changelog（1.106.3 为 2026-10-24、1.106.4 为 2026-10-25）。
- **LiteLLM 1.106.5 批（2026-10-27）**：1.103.10 / 1.104.8 / 1.105.6 / 1.106.5，被拒绝请求的失败日志仅保留核心请求字段、`tags` 与 guardrail 细节；GCS logger 改用代理上配置的 bucket 与凭证；验签仍用 `e82a3c6` 新密钥。
- **Portkey Gateway 2.0 仍为预发布**：官方 main 分支 README 仍标 Pre-Release（2026-10-28 复核）。
- **Dynamo 仍为 v1.6.0 stable、vLLM 仍为 v0.32.0、OpenRouter 窗口内检索未见新公告**。
- **深化**：级联裁判两面证据（Cluster, Route, Escalate arXiv:2606.27457 实证 vs ICLR 2026 静态策略理论对照；QE 两类错误分解缺失明示）；Dynamo KV 路由哈希确定性约束（`PYTHONHASHSEED=0` 或 `--prefix-caching-algo sha256`）；llm-d v1.0（2026-10-20）多租户 QoS、路由器热备、DisaggregatedSet、稳定 v1 API、会话亲和与轮次优先级（厂商宣称数字不引用）。

### 已同步进 skill 参考

- api-routing.md：LiteLLM 安全注记删除"二手转述"限定、改为已确认口径并追加 1.106.5；Portkey 行追加 2026-10-28 复核戳；llm-d 行追加 v1.0 与 KV 约束；论文一节追加级联两面证据与 QE 错误分解要求。

### 剩余未核实缺口（更新）

1. RouteLLM 提交历史页直接确认最后提交日（仍未决）。
2. Dynamo v1.5.0 细节（仍仅见二手日志站）。
3. Portkey Gateway 2.0 正式版是否发布（仍为预发布）。
4. Railway / OpenCode Console 事件实际影响范围（仍未决）。
5. 新增：CVE-2026-114612 官方公告与修复版本（仅第三方 CVE 库记录）。
6. 已解决：LiteLLM 1.106.3/1.106.4 官方确认、Portkey 收购官方口径。


---

## 2026-10-29 轮次：缺口复核 + 新增覆盖——RouteLLM 提交组、LiteLLM 1.106.6、Dynamo v1.6.1

### 已核实（一手为主）

- **RouteLLM 提交历史一手复核**：直接打开提交历史页，最新组为 2024-07-20（其后 2024-07-12、2024-07-11、2024-07-07）；组内单条提交的具体时间与哈希未显示，只确认到组日期。第三方评测所称 2024-08-10 不获该页支持，仍不采信（措辞为“不支持”而非“证伪”，因组日期≠单条提交日）。
- **LiteLLM 1.106.6 稳定回灌线**：官方 releases 页顶部预发布为 v1.109.0-dev.1（704 PR、15 项需行动变更，不得作钉版目标），其后第一条稳定线为 v1.106.6；唯一变更为 #46712 的 gzip 作用域与离线压缩回灌（提交 #48680），cosign 键仍为 `e82a3c6`。同批回灌线另有 v1.105.7 / v1.105.6 / v1.104.8 / v1.103.10。各版本发布日期页面未显示，不写 1.106.6 发布日；不声称 1.106.6 修复 CVE-2026-114612。
- **Portkey Gateway 2.0 仍为预发布**：官方 main 分支 README 首行横幅仍为 “Gateway 2.0 (Pre-Release)”；官方 releases 页最新稳定线为 v1.15.0（日期未显示）。
- **Dynamo 最新稳定已为 v1.6.1**：官方 releases 页定性为 “a patch release on top of v1.6.0”：刷新全部 trust store，镜像与 `ai-dynamo-runtime`、`kvbm` wheel 不再包含 emSign ECC Root CA - G3 根（rustls-webpki 0.103.16、webpki-roots 1.0.10）；全部镜像升至 Pillow 13；修复 v1.6.0 未能发布到 crates.io 的 kvbm 三个 crate。破坏性提示：证书链到 emSign 根的端点在新 bundle 下将校验失败。
- **vLLM v0.32.0 安全注记**：官方 releases 页顶部仍为 v0.32.0；安全修复含 GHSA-4hhp-h66f-j5j7、GHSA-q97c-p368-75c4、GHSA-frvg-qw88-hww9，默认 CORS 不再允许所有来源；发布日期未显示，不归入窗口。
- **窗口内新动态**：2026-10-28 至 2026-10-29 间，官方一手页面以日期确认的网关/路由正式发布为零；上列最新版均因无日期而不归入窗口。

### 已同步进 skill 参考

- api-routing.md：RouteLLM 停滞注记改为提交组一手口径；新增 1.106.6 回灌线与预发布排除句；Portkey 行追加 2026-10-29 复核戳与 v1.15.0；Dynamo 条目追加 v1.6.1 与信任根破坏性提示；vLLM 条目追加安全修复与 CORS 收紧句。

### 剩余未核实缺口（更新）

1. CVE-2026-114612 官方公告与官方修复版本（仅第三方 CVE 库记录；不得以 GHSA-7hp6-4w63-5g45 修复线等同）。
2. Dynamo v1.5.0 细节（仍仅见二手日志站一句话转述；官方注记全文未读到）。
3. Portkey Gateway 2.0 正式版是否发布（仍为预发布）；Portkey 1.15.2 之说与官方页 v1.15.0 出入（未解决）。
4. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。
5. 各最新版发布日期（LiteLLM v1.106.6、Portkey v1.15.0、Dynamo v1.6.1、vLLM v0.32.0，页面文本未显示）。
6. RouteLLM 2024-07-20 组内单条提交的精确日期/哈希（需单条链接复核）。
7. 已解决：RouteLLM 提交组一手复核、LiteLLM 1.106.6 回灌线、Dynamo 最新稳定版本、Portkey 2.0 预发布复核。


---

## 2026-10-30 轮次：缺口复核 + 新增覆盖——Dynamo v1.5.0/v1.5.1、v1.6.1 撤回、Railway 初步报告

### 已核实（一手为主）

- **Dynamo v1.5.0 官方注记（缺口关闭）**：官方 releases-data 表与注记页一手——稳定版 2026-09-18；自定义选取策略（WorkerScorer/WorkerPicker、过滤器链、硬/软会话亲和）、KV 索引扩展到 Mooncake 与磁盘层、CRD 存储版 `nvidia.com/v1beta1`、Rust EPP 默认、Snapshot 独立 operator、AIConfigurator 更名 AISimulate、KVBM 弃用（计划 v1.7.0 移除）、引擎钉版 SGLang 0.5.18 / TensorRT-LLM 1.3.0rc25 / vLLM 0.28.0；过滤器拒绝全部合格 worker 时返回 503（fail-closed）。第三方 DeployLog 的 10 月 9 日说法与官方表冲突，不采信。
- **Dynamo v1.5.1 补丁（官方表一手）**：2026-10-06，后端钉版与 v1.5.0 相同。
- **Dynamo v1.6.1 说法撤回（执行者亲自复核）**：官方 releases-data 表（单一事实源、每次发布 bump 时重新生成）截至 2026-10-30 仍写当前稳定 v1.6.0（Oct 22, 2026），表中无 v1.6.1 行；上一轮基于 GitHub releases 页的 v1.6.1 补丁说法与该表冲突，本轮撤回；Quickstart 的 `:1.6.1` 标签存在不等于稳定发布。
- **Portkey 2.0 仍为预发布（2026-10-30 复核戳）**：官方 main README 原话仍为 Pre-Release；1.15.2 确为 Docker 镜像标签，但 v1.15.0 发布日期仍未决，通道漂移仅为假说。
- **Railway 初步报告（一手，非最终报告）**：2026-10-22 官方博客，攻击窗口 2026-10-20 13:58 UTC 至 10-22 03:32 UTC，影响上限至多 1.85% 工作区（约 100,000 个，厂商自报），10-26 更新称现有用户侧已解决、新注册仍暂停，源码是否被访问未排除；最终报告截至本轮仍未发布。
- **窗口内新动态**：2026-10-29 之后 LiteLLM、Dynamo、Portkey、vLLM、llm-d 均无新稳定发布；OpenRouter 最新公告为 2026-10-28 路由重建帖（厂商自报）。

### 已同步进 skill 参考

- api-routing.md：Dynamo 条目改为 v1.6.1 撤回口径并补 v1.5.0 / v1.5.1；Portkey 行追加 2026-10-30 复核戳与 1.15.2 线索限定；Design checks 追加故障转移语义三项（触发条件、退化目标、回切条件）与初步报告/最终报告区分句。

### 剩余未核实缺口（更新）

1. Dynamo v1.6.1 最终解释（GitHub 侧说法与 docs 表 reconciled 情况）。
2. CVE-2026-114612 官方公告与官方修复版本（GHSA 数据不得等同）。
3. Portkey v1.15.0 发布日期与 1.15.2 出入成因。
4. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。
5. RouteLLM 2024-07-20 组内单条提交精确日期/哈希。
6. LiteLLM v1.106.6 与 vLLM v0.32.0 的官方发布日期。
7. 已解决：Dynamo v1.5.0 细节、Portkey 2.0 预发布复核（2026-10-30）、Railway 初步报告存在性。


---

## 2026-10-31 轮次：缺口复核 + 新增覆盖——LiteLLM 1.106.6 日期、Dynamo 仍未 reconciled、托管策略漂移规则

### 已核实（一手为主）

- **LiteLLM v1.106.6 发布日期已解决（2026-10-28，一手）**：官方 changelog 首条与 PyPI 文件上传日一致；此前“官方 changelog 未列出 v1.106.6”的二手交叉口径被推翻。
- **Dynamo v1.6.1 仍未 reconciled（执行者亲自复核官方表）**：releases-data 表仍写当前稳定 v1.6.0（Oct 22, 2026），无 v1.6.1 行；第三方引 GitHub 日期 Oct 27 只作存在性线索，文档表 bump 滞后仅为假说。
- **Portkey 2.0 仍为预发布（2026-10-31 复核戳，一手 README）**；OSS changelog 页仍无版本号与日期。
- **Railway 最终报告仍未发布（2026-10-31 复核戳）**：初步报告页更新戳仍为 2026-10-26，正文仍写调查进行中。
- **窗口内新动态**：2026-10-30 之后四轨检索范围内无带日期的一手稳定发布；OpenRouter 最新公告仍为 2026-10-28 路由重建帖。
- **深化**：托管路由策略漂移规则（OpenRouter 2026-10-28 公告：最便宜优先改为平衡成本/速度/可靠性；基线须按公告日打戳重测；二手 +6% 数字不引用）；网关开销分档、组合路由延迟阶梯、2026-01 统一评估警示均为二手线索，不入技能数字。

### 已同步进 skill 参考

- api-routing.md：1.106.6 段追加 2026-10-28 发布日期；Dynamo 条目推进至 2026-10-31 复核并补第三方 Oct 27 线索限定；Portkey 行追加 2026-10-31 复核戳；Railway 句推进至 2026-10-31；Design checks 追加托管策略漂移规则。

### 剩余未核实缺口（更新）

1. Dynamo v1.6.1 最终解释（GitHub 侧与 docs 表 reconciled 情况）。
2. CVE-2026-114612 官方公告与官方修复版本（GHSA 数据不得等同；StackFlag 的 NVD/MITRE 来源清单为下轮直查线索）。
3. Portkey v1.15.0 发布日期与 1.15.2 出入成因（2026-01-12 仅二手单源）。
4. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。
5. RouteLLM 2024-07-20 组内单条提交精确日期/哈希（需单条逐字链接）。
6. vLLM v0.32.0 官方 GitHub release 日期（现有 10-18 为二手交叉）。
7. 已解决：LiteLLM v1.106.6 发布日期；Portkey 2.0 预发布复核（2026-10-31）。


---

## 2026-11-03 轮次：缺口复核 + 新增覆盖——Dynamo v1.6.1 存在性、CVE 修复线、LiteLLM LTS 试点、OpenRouter 路由机制

### 已核实（一手为主）

- **Dynamo v1.6.1 存在性已解决，上一轮撤回口径修正**：GitHub releases 页一手（v1.6.0 之上补丁版：信任库刷新、emSign ECC Root CA - G3 移除为破坏性、Pillow 13、KVBM crates 发布修复，后端 pin 与 v1.6.0 相同）；PyPI `ai-dynamo-runtime` 历史列 1.6.1 为 Oct 27, 2026；执行者亲自复核的官方 releases-data 表截至 2026-11-03 仍写当前稳定 v1.6.0、无 v1.6.1 行。上一轮撤回的是“以 docs 表否定其存在”，不是 v1.6.1 本身不存在；选型可按 1.6.1 钉版并注意破坏性，但不得引 docs 表称其为当前稳定。
- **CVE-2026-114612 修复版本线已确认（NVD 官方记录，CNA GitHub）**：1.100.4 / 1.101.3 / 1.102.2 / 1.103.1 / 1.104.0-rc.2，References 栏链接五个 release tag 与 GHSA-7hp6-4w63-5g45；独立厂商公告仍未见，GHSA 不得等同厂商公告；1.106.6 是否携带修复仍未确认。
- **Portkey 最新版定为 v1.15.2、2026-01-12（二手元数据汇总，数据刷新 2026-11-02）**；Oct 21 为测试日期；2.0 仍预发布（2026-11-03 仓库页复核）；v1.15.0 精确日期与出入成因仍未确证；默认运行暴露面二手注记（`/public` 与 `/log/stream` 无需登录，`NODE_ENV=production` 可关）；CVE-2026-110719 SSRF 二级通报无确认修复版。
- **LiteLLM 发布节奏与 LTS 试点（官方博客一手，执行者亲自复核）**：自 2026-11-02 当周起改周二上午发布；试点一个月 LTS 线，首条 1.108，只给翻译修复、成本统计修复与新端点补丁；试点期间暂停四线补丁政策。
- **OpenRouter 重建路由机制（官方博客一手，页内无日期，不归入窗口）**：实时 Routing Cost =（有效价格 + 等待时间成本）÷ 成功概率；`provider.order` 绕过、显式 `sort` 优先、会话钉住首供应商；switchback 数字为厂商自报，不引用。
- **vLLM v0.32.0 日期口径降为二手交叉**：官方 release 正文已核实，页内无日期戳，2026-10-18 靠多源二手交叉一致。
- **Railway 最终报告仍未发布（2026-11-03 复核戳）**：官方披露帖更新止于 2026-10-23；OpenCode post-mortem 截至 2026-11-02 二级综合仍称未发布。

### 已同步进 skill 参考

- api-routing.md：Dynamo 条目改为 v1.6.1 存在性修正口径；LiteLLM 段追加 LTS 试点句与 CVE 修复版本线修正；Portkey 行改为 1.15.2 定版 + 暴露面注记；vLLM 行改为 2026-10-18 二手交叉口径；Railway 句推进至 2026-11-03；Design checks 追加 OpenRouter 路由机制 bullet。

### 剩余未核实缺口（更新）

1. CVE-2026-114612 独立厂商公告。
2. Portkey v1.15.0 精确发布日期与上一轮顶部显示成因。
3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。
4. RouteLLM 2024-07-20 组内单条提交精确日期/哈希。
5. vLLM v0.32.0 官方 GitHub 页内发布日期戳。
6. 新增：LiteLLM LTS 试点细则与 OpenRouter 重建路由博客发表日期；Portkey CVE-2026-110719 厂商确认修复版。
7. 已解决：Dynamo v1.6.1 存在性；CVE 修复版本线；Portkey 最新版定版。


---

## 2026-11-04 轮次：缺口复核 + 新增覆盖——Portkey 出入成因定位、LiteLLM LTS 细则、首个周二无新稳定版

### 已核实（一手为主）

- **Portkey 1.15.x 出入成因已定位（官方 releases 列表与两 tag 页一手）**：列表页文本抽取丢弃各条目标题行，首条实为 v1.15.0 完整变更（收尾“1.15.0 by @VisargD in #1482”、Full Changelog v1.14.3...v1.15.0），误认首条即最新版即把 v1.15.0 内容安到 v1.15.2 头上；真 v1.15.2 仅 #1483–#1499（Full Changelog v1.15.1...v1.15.2）。上一轮“通道漂移假说”撤回。v1.15.2 / 2026-01-12 仍二手（Geekflare 刷新 2026-11-02）；v1.15.0 精确日期仍未决（仅知 2026-01、早于 01-12）；2.0 仍预发布（2026-11-04 仓库页复核）。
- **LiteLLM LTS 试点细则（一手，官方博客 + LTS 试点页）**：LTS 线面向“停留一版又持续获修复”的部署，补丁限于 LLM 翻译修复、成本追踪修复与新端点；试点期间暂停四线补丁政策。改周二后首个周二（2026-11-03）截至上午核查时未见 1.108 稳定版（检索阴性结论，非厂商声明）；最新正式版仍为 1.106.6（2026-10-28）；1.108 仅 1.108.0-rc.1（2026-10-29，二手镜像日期）。
- **CVE-2026-114612 独立厂商公告仍未见；1.106.6 含修复仍未确认**：NVD 仍为 GitHub CNA 转录（Awaiting Enrichment）；LiteLLM 博客相关两篇针对 1.107.0 供应链事件与另一 embargo 漏洞，非本 CVE 公告；谱系推断不得写成厂商确认。GHSA 首次发布日期镜像分歧（2026-10-22 vs 2026-10-27）仍未决。
- **Portkey CVE-2026-110719 厂商确认修复版仍未出现**：CVE 记录（2026-10-14）称 1.14.0–1.15.2 SSRF；Feedly 记“No official patch has been identified”；最新正式 tag 仍为受影响顶端 v1.15.2。区分 2025 年旧 GHSA（CVE-2025-66405，修复于 1.14.0、PR #1372）针对另一报告。
- **Railway 最终报告与 OpenCode post-mortem 仍未发布（2026-11-04 复核戳）**：Railway 最高文件仍为 2026-10-22 初步报告（更新止于 2026-10-26）；OpenCode post-mortem 未见（二手追踪页至 2026-11-02 一致称未发布，阴性结论本身为二手交叉）。
- **vLLM v0.32.0 官方页内仍无日期字段（一手复核）**：tag 页全文 604 行检索“Oct”零命中；2026-10-18 仍为多源二手交叉。内容一手：814 commits / 363 contributors、POST /v1/systemone、Python 3.10 弃用、CORS 默认收紧、三项 GHSA 修复。
- **RouteLLM 单条提交仍未决**：本轮未能锁定 2024-07-20 当天提交哈希；上一轮所指组内单条提交的主题/PR 号未包含在本轮简报中，无法唯一确定。下一轮须先从 track 笔记取主题或 PR 号再直开 commit 页。
- **窗口内新动态**：Dynamo、Portkey、vLLM、llm-d、OpenRouter 路由机制均无窗口内新发布/公告。Dynamo v1.6.0 Router policy YAML 优先于 scoring flags 已在上一轮技能中写入，不重复计入。

### 已同步进 skill 参考

- api-routing.md：Portkey 段改为出入成因定位口径（通道漂移假说撤回）+ 2026-11-04 预发布复核戳；LTS 段追加补丁范围细则与首个周二无稳定版注记；SSRF 句与 Railway 句复核戳推进至 2026-11-04。

### 剩余未核实缺口（更新）

1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认；GHSA 首次发布日期分歧。
2. Portkey v1.15.0 精确发布日期；Portkey CVE-2026-110719 厂商确认修复版。
3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。
4. RouteLLM 2024-07-20 组内单条提交精确日期/哈希（须主题/PR 号）。
5. vLLM v0.32.0 官方 GitHub 页内发布日期戳。
6. LiteLLM 1.108 稳定版；LTS“新端点”具体清单。
7. 已解决：Portkey 1.15.x 出入成因；LiteLLM LTS 试点细则。

---

## 2026-11-05 轮次：缺口复核 + 新增覆盖——GHSA 日期分歧、RouteLLM 单条提交、Portkey v1.15.0 日期、LiteLLM 1.108 稳定版

### 已核实（一手为主）

- **GHSA 日期分歧已解决（口径区分）**：2026-10-22 是 CVE 记录发布日（Rapid7 转述 NVD/MITRE，二手），2026-10-27 是 GHSA 咨询本身发布日（Ostorlab 时间线、OSV 镜像，二手），两者不冲突。
- **RouteLLM 2024-07-20 单条提交已解决（一手，执行者独立抽查复核）**：组内仅一条提交 `88b8dec894809e2a3ed77096623910eb9f861563`，标题“docs: update README.md / minor fix”，实为 README 一行文法修正；committer date 2024-07-20T20:56:36Z（分组依据），author date 2024-07-13T16:25:55Z；作者 Ikko Eltociear Ashimine，提交者 Isaac Ong，父提交 3a85b9e65a587de221e53f534d7a652be96cc985。仓库现状：公开、未归档、0 release，停滞结论不变。
- **Portkey v1.15.0 精确日期已解决（一手，执行者独立抽查复核）**：published_at 2025-12-23T14:21:27Z（created 2025-12-23T14:10:37Z），发布者 VisargD，收尾 PR #1482，Full Changelog v1.14.3...v1.15.0。v1.15.2 的 2026-01-12 仍为二手，时间顺序自洽；列表页文本抽取仍丢弃标题行；v1.15.2 的 API 查询本轮返回 403 未能读取。
- **LiteLLM v1.108.0 稳定版已发布（2026-11-04，一手，执行者独立抽查复核内容）**：官方 release notes 确认“v1.108.0 was promoted to stable from v1.107.0rc1”，覆盖自 v1.106.0 以来的全部变更；新增 Gemini 4 Argon、Microsoft 365 Copilot、OpenAI Live /v1/live/sessions、Azure PTU 共享容量，237 个新目录条目；破坏性变更含 enforce_fallback_model_access 默认 true、OpenRouter 移出内置成本表、count 路由改回 provider 报错、Lens 解绑。上一轮“首个周二上午未见稳定版”为时点结论，已被 11-04 发布取代。同日 v1.106.7（一手）：/v1/videos 支持 custom_provider_map；升级前须删 lensWorker.enabled。
- **CVE-2026-114612 独立厂商公告仍未见；1.106.6 含修复仍未确认（仅谱系推断）**：GHSA 页（一手）只说尽快升级，未逐字列修复版；1.106.5/1.106.6 携带的是另一 advisory GHSA-w4c7-34wj-hqm8 的修复，不得与本 CVE 混写。
- **Portkey CVE-2026-110719 厂商确认修复版仍未出现**：CVE 记录（2026-10-14）仍称 1.14.0–1.15.2 SSRF；Geekflare 2026-11-04 刷新版称“2026 年 5 月合并的修复尚未发布”（二手，无版本号、无厂商确认）。区分旧案 CVE-2025-66405（已修复于 1.14.0，PR #1372）。
- **Railway 最终报告与 OpenCode post-mortem 仍未发布（2026-11-05 复核戳）**：Railway 最高文件仍为 2026-10-22 初步报告（更新止于 2026-10-26）；OpenCode post-mortem 未见（二手追踪至 2026-11-02 一致称未发布）。
- **vLLM v0.32.0 官方页内仍无日期字段**：2026-10-18 仍为多源二手交叉（新增英文维基百科词条信息框标注，可信度略升，不得升级为一手）。v0.33.0 稳定版状态二手冲突（“v0.33.0” vs v0.33.0rc3），官方 release 页直接确认前不得写成已确认稳定版；v0.32.0 仍为最后确认的稳定版。
- **LiteLLM LTS“新端点”具体清单仍未决**：官方 LTS 试点页全文仅重复三类范围，未列具体端点名；不得把 1.108.0 自带的新端点写成 LTS 补丁清单。
- **窗口内其余否定性结论**：Dynamo（官方文档 Release Artifacts 页当前稳定仍为 v1.6.0，一手；仅 nightly 1.7.0.dev20261103，窗口外）、Portkey、llm-d（最新仍为 1.0，2026-10-20）、OpenRouter 路由机制均无窗口内新发布/公告。
- **深挖素材（论文/厂商自报，不整段入技能）**：Calibrate, Then Route（arXiv:2609.16206，模拟器常数代硬件校准丢 4.5 goodput 点、约 40% 尾延迟优势；≤3 实例队列计数已够用；极端稀缺下贪婪成本最小化不如盲 spread）；ParetoBandit（arXiv:2604.00136v1，预算闭环硬约束，1,824 prompt 下超目标不超过 0.4%，单次决策 22.5μs，小样本限制外推）；Workload–Router–Pool 三维（arXiv:2603.21354，成本悬崖取决于 prompt 长度 CDF）；缓存 bandit 背包模型（EMNLP 2026，总成本降低约 12%，论文自报）；OpenRouter Jev Router（ClickHouse 博客 2026-11-02，窗口外，厂商自报：约 30 种任务类型→按任务真实用量排序模型→约束过滤→选端点备 fallback）。

### 已同步进 skill 参考

- api-routing.md：Portkey 段改为 v1.15.0 一手日期口径；LiteLLM 段追加 GHSA 日期归属澄清、1.108.0 稳定版与 1.106.7 段；RouteLLM 段追加单条提交一手确认；事故段复核戳推进至 2026-11-05；Design checks 追加路由定量模型三补强与 Jev Router 第二层 bullet。

### 剩余未核实缺口（更新）

1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认。
2. Portkey CVE-2026-110719 厂商确认修复版。
3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。
4. vLLM v0.32.0 官方 GitHub 页内发布日期戳。
5. LiteLLM LTS“新端点”具体清单；vLLM v0.33.0 稳定版状态（官方 release 页直接确认）。
6. 已解决：GHSA 日期分歧；RouteLLM 单条提交；Portkey v1.15.0 日期；LiteLLM 1.108 稳定版。

## 2026-11-07 轮次小节（版本 0.10.0；六路并行之路由两路）

- 本轮增量：LiteLLM v1.108.1（2026-11-05）与 v1.108.2（2026-11-06）LTS 补丁（一手 changelog；修复类，无新端点）；vLLM v0.33.0 已确认为稳定版（tag 2026-11-01，commit 4ba787e；v0.32.0 不再是最后确认的稳定版）；LiteLLM 1.108.x 路由器语义收紧与成本归因精确化（一手 release notes：字符串 fallback 启动拒绝、唯一部署不 bench、亲和/cooldown 后不拒请求、按 attempt 计费、cost-based-routing 经 base_model 定价）；OpenRouter 新一代 Auto Router + Auto Code（上架 11-05、公告 11-06，Jev Router 引擎；74% 降本为厂商自报）与 Jev Router 官方文档机制（更新 2026-11-06，一手：cost_tier 为策略偏好、plugin 参数、metadata 字段、author-auto 别名、决策模型 1.14）；路由深挖三论文（Route Anything arXiv:2611.05275，2026-11-03；RouteBalance；CacheRoute“先 shadow replay 再启用亲和”）；Railway 最终报告与 OpenCode post-mortem 复核戳推进至 2026-11-07（仍未发布；Railway 初步报告更新戳纠正为 2026-10-26T20:41:12Z）；CVE-2026-114612 与 Portkey CVE-2026-110719 状态更新（二手：Ostorlab 称 1.106.6“无安全变更”、Geekflare 称五月修复已合入未发布，均无版本号）。
- 已同步进 skill 参考：api-routing.md 追加 LiteLLM LTS 补丁段、vLLM v0.33.0 段、LiteLLM 1.108.x 语义收紧段、OpenRouter 新一代路由段、Route Anything / RouteBalance / CacheRoute 论文段；事故段复核戳推进至 2026-11-07（Railway 更新戳已纠正）；CVE 段追加状态更新句。
- 推翻的旧说法：vLLM“v0.32.0 仍为最后确认的稳定版”；Railway 初步报告更新戳“2026-10-23”。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认。2. Portkey CVE-2026-110719 厂商确认修复版。3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。4. vLLM v0.32.0 官方 GitHub 页内发布日期戳。5. LiteLLM LTS“新端点”具体清单（跟踪 1.108.3+ release notes）；vLLM v0.33.0 Latest 徽标与 release notes 正文。6. OpenRouter Routing Cost 官方公告原文（数字仍不引用）。7. Conformal Cascade 疑似 v3；RouteBalance v2 修订日期。8. 已解决：vLLM v0.33.0 稳定版状态；Railway 初步报告更新戳。

## 2026-11-07 轮次小节（下午轮，版本 0.11.0；六路并行之路由两路）

- 本轮增量：vLLM v0.32.0 日期升级为一手（GitHub Releases API，2026-10-18T06:50:48Z）；v0.33.0 release notes 正文升级为一手（927 提交 / 389 贡献者、PyTorch 2.14、CUDA 13.0 only、/v1/decisions 默认开启、Medusa 与旧版 Ray 编译图执行器移除、11 个新模型架构；Release 对象 tag 为 v0.33.0rc3，published_at 2026-11-02T10:10:05Z，prerelease 标志 off；轻量 tag v0.33.0 指向同一 release；截至 17:44 无 v0.33.1 / v0.34.0）；LiteLLM 仍无 v1.108.3 / v1.109 稳定版，v1.110.0-dev.1 预发布已现（published_at 未验证，不得作钉版目标）；CVE-2026-114612 GHSA 页一手全文仍无已修复版本列表、未提 1.106.6（修复版来自 Advisory DB 元数据），与 GHSA-w4c7-34wj-hqm8（CVSS 10.0，修复版 1.103.10 / 1.104.8 / 1.105.6 / 1.106.5，无 CVE 号）消歧；Portkey CVE-2026-110719 复核戳推进至 2026-11-07 下午（仍无厂商确认修复版，Gateway 2.0 仍预发布分支）；Railway 最终报告与 OpenCode post-mortem 复核戳推进至 2026-11-07 下午（仍未发布；Railway 初步报告更新戳精确到 2026-10-26T20:41:12.901Z）；OpenRouter Routing Cost 公式与护栏升级为一手（官方博客 2026-10-28、官方文档 2026-10-29：成本/次 ÷ 成功概率，$1.00/80% = $1.25；TTFT 双计价；Floor / Back off / Cap 护栏；会话钉在首个 provider）；Conformal Cascade v3 确认（2026-07-31，修订内容未决）；RouteBalance v2 确认（2026-10-31，profiled TPOT 替代学习延迟预测）；新论文 arXiv:2611.04098（2026-11-03，oracle gap 方法论警示：union premium 仅 0.215–0.588pp vs 陪同研究 3.5pp，须带 pool / protocol / samples / decoding budget 报告；65,536 token 上限、45,000 评分响应、B300 单卡；仅 31 响应触顶却占输出长度平方代理的至多 60.2%）。
- 已同步进 skill 参考：api-routing.md 追加 vLLM v0.32.0 日期一手段、v0.33.0 正文一手段、LiteLLM v1.110.0-dev.1 段、CVE GHSA 全文复核与 w4c7 消歧段、Portkey / Railway / OpenCode 复核戳段、Routing Cost 公式与护栏段、Conformal Cascade v3 / RouteBalance v2 / 2611.04098 论文段、Dynamo 张力复核段。
- 推翻的旧说法：vLLM v0.32.0 日期“仍为多源二手”；v0.33.0“特征内容仅二手”；OpenRouter Routing Cost“公式仅二手”；Conformal Cascade“疑似 v3”；RouteBalance“v2 修订日期未决”；Dynamo v1.6.1“被 docs 表证伪”（子代理主张被驳回，张力仍未调和）。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认。2. Portkey CVE-2026-110719 厂商确认修复版。3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。4. LiteLLM LTS“新端点”具体清单（跟踪 1.108.3+ release notes）；LiteLLM v1.110.0-dev.1 / v1.109.0-dev.1 published_at。5. vLLM v0.33.0 Latest 徽标。6. Dynamo v1.6.1 调和（GitHub release 页 + PyPI 存在性 vs docs 表）。7. OpenRouter Routing Cost 实验数字与官方公告 URL；Conformal Cascade v3 修订内容；2611.04098 陪同论文。8. 已解决：vLLM v0.32.0 日期戳；vLLM v0.33.0 release notes 正文；OpenRouter Routing Cost 公式；Conformal Cascade v3；RouteBalance v2。

## 2026-11-07 轮次小节（晚间轮，版本 0.12.0；六路并行之路由两路）

- 本轮增量：LiteLLM v1.110.0-dev.1 published_at 一手定稿（2026-11-07T04:36:43Z，预发布；v1.109.0-dev.1/dev.2/rc.1 日期亦一手：2026-10-28/10-30/11-04），仍无 v1.108.3 / v1.109 稳定版，最新稳定仍 v1.108.2（GitHub Latest，2026-11-06T22:07:43Z），LTS“新端点”清单仍未列出；vLLM Latest 徽标已确认（releases/latest 返回 v0.33.0rc3 Release 对象本身），仍无 v0.33.1 / v0.34.0，CUDA 12.9 停发、只发 CUDA 13.0；CVE-2026-114612 证据升级——tag v1.106.6 源码比对含修复（修复提交 1d5fdc87，2026-09-29；`aad` 8 处、`v2:gcm` 5 处，与 v1.102.2 计数一致；auth_checks.py 已接线），但厂商仍未逐字确认（GHSA 页、GHSA API 五条范围 first_patched_version 全 null、v1.106.6 注记均无链接），两种证据严格区分；Portkey CVE-2026-110719 main 分支成因代码级确认（requestValidator/index.ts 仍默认信任 localhost/127.0.0.1/::1/host.docker.internal），2.0 无任何 v2.x tag，main 最后提交 2026-05-25；Railway 最终报告与 OpenCode post-mortem 复核戳推进至 2026-11-07 晚间 15:45 UTC（仍未发布；opencode.ai/blog 直接打开 404，该路径不存在）；Dynamo v1.6.2 已发布（2026-11-05T22:11:38Z，PyPI info.version 已为 1.6.2），文档侧“Current stable release”仍停 1.6.0（连 v1.6.1 自身文档页亦如此），定性为文档声明滞后；OpenRouter Routing Cost 实验数字表已从官方博客（2026-10-28）完整引用（TTFT p50 −16.9%、吞吐量 p50 +16.3%、每千请求成本 +6.0%、429 请求占比 7.1%→3.8% 少 46% 等，全部厂商自报）；Conformal Cascade v3 全文已读（κ=1、定理 2 覆盖界 1−Kα、定理 4 成本闭式）；2611.04098 陪同论文确认为 arXiv:2607.03436（v3 修订 2026-09-27），“3.5pp”= MATH-500 全池 product premium 3.542pp；路由论文复核戳推进（RouteBalance/CacheRoute/AgentRouter 均无新版；边界：Local or Escalate 2611.01639、Dekoninck v5 2410.10347；11-04 至 11-07 未发现新路由论文，检索范围为限）；Auto Router / Auto Code 11-06 之后无进一步官方更新（Jev Router 文档最后更新 2026-11-06T23:55:27Z）。
- 已同步进 skill 参考：api-routing.md 追加 LiteLLM dev 线日期段、vLLM Latest 徽标确认段、CVE 1.106.6 代码证据段、Portkey main 代码段、Railway/OpenCode 复核戳段、Dynamo v1.6.2 段、Routing Cost 实验表段、Conformal Cascade v3 全文段、2607.03436 陪同论文段、路由论文复核戳段、Auto Router 戳推进段。
- 推翻的旧说法：LiteLLM v1.110.0-dev.1“published_at 未验证”；vLLM Latest 徽标“仍未确认”；OpenRouter“429 事件总数少 67%”（官方正文与结果表均无此数字，纠正为 −46%）；Dynamo v1.6.1“被 docs 表证伪”（本轮 v1.6.2 发布进一步证实发布侧在前、文档侧滞后）；“1.106.6 是否含修复全无证据”（源码比对证据已获，厂商逐字确认仍缺）。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认。2. Portkey CVE-2026-110719 厂商确认修复版。3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。4. LiteLLM LTS“新端点”具体清单。5. Conformal Cascade v2→v3 diff。6. 已解决：LiteLLM dev 线日期；vLLM Latest 徽标；OpenRouter 实验数字表；2611.04098 陪同论文；Dynamo v1.6.2；CVE 1.106.6 代码证据；Portkey main 成因代码确认。

## 2026-11-08 轮次小节（晨轮，版本 0.13.0；六路并行之路由两路）

- 本轮增量：OpenRouter 新一代 Auto Router + Auto Code 官方公告现一手确认（2026-11-06；11 因子决策模型 + 实时价格/健康/缓存，取代旧「约 30 任务类型 + 7 日支出份额」机制；底层 Jev Router；数字全部厂商自报）；LiteLLM 仍无 v1.108.3 / v1.109 稳定版，最新稳定仍 v1.108.2（GitHub Latest，2026-11-06T22:07:43Z，12 个 PR 全为修复；v1.110.0-dev.1 仍为预发布），LTS“新端点”仍无清单（试点已启动一个月）；vLLM 仍无 v0.33.1 / v0.34.0；Dynamo 实验性预发布 v1.7.0-kimi-k3.1-pro-dev.1（2026-11-06T21:44:59Z，Kimi-K3.1-Pro 早期支持，非生产），文档侧“Current stable release”仍停 1.6.0；CVE-2026-114612：GHSA API 元数据已补 patched_versions（1.100.4/1.101.3/1.102.2/1.103.1/1.104.0rc2），但厂商逐字确认仍缺（GHSA 页仍无已修复版本列表）；Portkey CVE-2026-110719 仍无修复版，2.0 仍预发布，企业版 v2.7.0 消歧（2026-11-07，企业版线，与开源 1.15.2 区分）；Railway 最终报告与 OpenCode post-mortem 复核戳推进至本轮（仍未发布；Stackademic 声称已发 post-mortem 的文章判定不可信——捏造 Console Lite、IOC 已发布等细节，已排除）；Conformal Cascade v2→v3 无实质文本差异（全文提取文本比对，缺口关闭）；路由论文：Route Anything（2611.05275，+5.4 分、8.82× 效率，论文自报）、Local or Escalate（2611.01639，42.1%→69.2%，内部基准未公开）、ServerBench（2611.06841，2026-11-04，4.87× 论文自报）、TRIAGE（2611.01174，2026-10-31，AUROC 0.571→0.846 论文自报）、Dynamic Mixed-Precision Routing v3（2026-11-02）、Who Pays（2610.23249）；ParetoBandit 日期纠正（实为 2026-03-31/04-14 旧文，非 11 月新文）；RouteLLM 自 2024-08-10 后无提交，定性为无人维护；llm-d 仍 v1.0.0；LiteLLM 投毒整改清单（Socket 覆盖率 + 审查清单 + 毒包列表）仍在。
- 已同步进 skill 参考：api-routing.md 追加 OpenRouter 新一代 Auto Router 段、LiteLLM 2026-11-08 复核段、Portkey 企业版消歧段、Dynamo kimi 预发布段、GHSA patched_versions 段、事故复核段（Stackademic 排除）、Conformal Cascade v2→v3 无差异段、路由论文新文段、ParetoBandit 日期纠正段、RouteLLM 无人维护段、llm-d 戳推进段、vLLM 戳推进段。
- 推翻的旧说法：GHSA API「first_patched_version 全 null」（元数据已补全）；Conformal Cascade「v2→v3 diff 未决」（已关闭为无实质文本差异）；OpenRouter Auto「仍按旧机制排序」（11-06 重建版已取代）；ParetoBandit「11 月新出的 bandit 路由论文」（实为 3–4 月旧文）；RouteLLM「活跃的开源路由框架」（无人维护）；「OpenCode 已发布详细 post-mortem」（Stackademic 文章不可信）。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认。2. Portkey CVE-2026-110719 厂商确认修复版；v1.15.2 tag 准确到日；Geekflare「5 月已合并修复」无哈希佐证。3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见；下轮直接打开 OpenCode 博客首页复核）。4. LiteLLM LTS“新端点”具体清单；v1.109 正式版；vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）。5. Dynamo 文档滞后、v1.7.0-kimi 线演进。6. Route Anything「8.82×效率」定义；Local or Escalate 内部基准未公开。7. 已解决：GHSA patched_versions；Conformal Cascade v2→v3 diff；OpenRouter 新一代 Auto Router；ParetoBandit 日期；RouteLLM 维护状态。

## 2026-11-08 轮次小节（中午轮，版本 0.14.0；六路并行之路由两路）

- 本轮增量：LiteLLM **v1.108.3** 为窗口内唯一带日期的一手新稳定发布（published 2026-11-07T22:29:58Z，PyPI 上传 2026-11-07T22:25:38Z）；v1.109 仍仅 rc.1；预发布 v1.110.0-dev.2 窗口内发布（2026-11-08T02:25:47Z）；LTS「新端点」清单仍未列出。Route Anything「8.82×效率」定义闭合（η=期望质量/期望成本，四测试床拟合路线均值 25.5/7.1/1.35/1.33；认证路线均值约 7.2× 为本轮推导、非论文原 claim；+5.4 分=逐输入 oracle vs 每基准最优固定候选，24 基准、19,814 输入）。Local or Escalate 基准构成与协议公开（240 条五层配比、MiniCPM-o 4.5 + gpt-5.5、L22 读出器、5,228 校准行；题目与代码未公开），「11-03 修订」口径纠正为 v1 单版本。Portkey v1.15.2 日期获 Docker Hub 一手佐证（镜像推送 2026-01-12T08:51:42Z）；Geekflare「5 月已合并修复」无哈希、仍二手。复核戳推进：CVE-2026-114612 厂商逐字确认仍缺；Railway 最终报告与 OpenCode post-mortem 仍未发布（opencode.ai/blog 实测 404）；vLLM 仍无 v0.33.1/v0.34.0；OpenRouter 11-06 后无更新；Dynamo 文档仍停 1.6.0。
- 已同步进 skill 参考：api-routing.md 追加 LiteLLM v1.108.3 段、Route Anything 定义闭合段、Local or Escalate 纠正段、Portkey Docker Hub 佐证段、vLLM/OpenRouter/Dynamo/事故复核戳推进段。
- 推翻的旧说法：Local or Escalate「11-03 更新」（v1 单版本）。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认。2. Portkey CVE-2026-110719 厂商确认修复版；Geekflare「5 月已合并修复」无哈希佐证。3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见；下轮直接打开 OpenCode 博客首页复核）。4. LiteLLM LTS“新端点”具体清单；v1.109 正式版；vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）。5. Dynamo 文档滞后、v1.7.0-kimi 线演进；**新增张力（仅记本笔记，不整合进技能）**：Dynamo `v1.7.0-kimi-k3.1-pro-dev.1` tag 在官方 releases-data 表全文检索中无对应行（"1.7.0-kimi"/"K3.1" 均 0 命中），与上一轮的 GitHub release 页口径存在张力；下轮再核对出处，不推翻上一轮结论。6. Route Anything 认证路线均值推导复核。7. 已解决：LiteLLM v1.108.3；Route Anything 8.82×定义；Local or Escalate 基准构成与「11-03 修订」纠正；Portkey v1.15.2 Docker Hub 佐证。

## 2026-11-08 轮次小节（晚间轮，版本 0.15.0；六路并行之路由两路）

- 本轮增量：Route Anything「认证路线均值约 7.2×」口径纠正——Table 5 全文重查（arXiv:2611.05275v1，§2/§5.1 一手）：仅代码（19.4×）与文本问答（6.7×）两床有认证行，图像/机器人床只有拟合行，7.2× 为认证+拟合混合均值，两床纯认证均值 13.05×；认证代价 25.5→19.4× / 7.1→6.7×；拟合下限仅 12.5% 切分成立 vs 认证 200/200。Portkey CVE-2026-110719 消歧：Geekflare 全文直读（181 行）——「5 月已合并修复」针对 /log/stream 泄露 prompt 与 provider key（NODE_ENV=production 关闭），非 CVE 的 SSRF，无哈希佐证；CVE 仍无厂商确认修复版（npm 最新仍 1.15.2，modified 2026-01-12）。OpenRouter 基准表与十月数据（$1.64 vs $6.43；Deep SWE 74.3% vs 70.5%；Terminal Bench 4.0 50.0% vs 53.1%；790T 厂商自报）——二手整理，须直读基准页复核（未验证暂定 R3）。AgentRouter v1.2 首个更名后版本 + Python SDK（2026-11-06 博客）。Dynamo 调和证据：releases-data 表亦遗漏已确认的 v1.6.0-minimax-m3.1-flash-dev.1，「表无行」不构成反证；kimi tag 未直接再验，维持原口径。复核戳推进：LiteLLM 仍无 v1.109 正式版（PyPI JSON 一手：1.109.0 不存在，最新 1.109.0rc1）、无 1.108.4、dev.2 最新、changelog 仍无 v1.108.3 条目；vLLM 仍无 v0.33.1/v0.34.0；CVE 旁证（changelog v1.106.6 条目仅 gzip 修复；GHSA 页本轮抓取超时未重试，以中午轮 03:39 UTC 复核为准）；Railway 最终报告仍未发布（初步报告最后更新 2026-10-26T20:41:12Z；补充细节：攻击者未访问用户账户、已通知多家厂商撤销密钥、无法退还被盗刷 spend）；OpenCode post-mortem 仍未发布（opencode.ai/blog 本轮实测仍 404）；六篇路由论文均无新版；arXiv 周末无公告。GitHub API 本轮全部 403 限流，结论改以 PyPI JSON 一手核对。
- 已同步进 skill 参考：api-routing.md 追加 Route Anything 7.2× 口径纠正段、Portkey CVE 消歧段、OpenRouter 基准表段（未验证暂定标注）、AgentRouter v1.2 段、Dynamo 调和证据段、LiteLLM/vLLM/CVE 复核戳段、Railway/OpenCode 复核戳段、路由论文复核戳段。
- 推翻的旧说法：Route Anything「认证路线均值约 7.2×」（混合口径，撤回）。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 独立厂商公告；1.106.6 含修复的厂商逐字确认（GHSA 页下轮重读）。2. Portkey CVE-2026-110719 厂商确认修复版。3. Railway 最终调查报告与 OpenCode 官方 post-mortem（均未见）。4. LiteLLM LTS“新端点”具体清单；v1.109 正式版；vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）。5. Dynamo v1.7.0-kimi tag 本身直接再验。6. OpenRouter 基准表直读复核（R3 未验证暂定）。7. 已解决：Route Anything 7.2× 口径；Geekflare「5 月修复」消歧；Dynamo 表遗漏调和证据。

## 2026-11-08 轮次小节（深夜轮，版本 0.16.0；六路并行之路由两路）

- 本轮增量：OpenRouter 基准表升级为一手（官方基准页 + 数据接口直读，192 run；Auto Code $1.64 vs Opus 5.5 高努力 $6.43；Deep SWE 74.3% vs 70.5%；Terminal Bench 4.0 50.0% vs 53.1%；其余三基准补全：SWE-Atlas 62.5% vs 54.0%（Opus 无 high 档）、τ³-bench 52.6% vs 42.3%、MMMU Pro 82.3% vs 81.8%；十月数据 790T/+37%/agentic 76% 以官方数据回顾原文为准；聚合口径注记：独立重算 Opus $6.37 与板面 $6.43 差 $0.06，保留板面值）。Dynamo kimi tag 直接再验（v1.7.0-kimi-k3.1-pro-dev.1，published 2026-11-06T21:44:59Z，ynpandey-nv，Kimi-K3.1-Pro 支持、非生产）与最新稳定 v1.6.2（2026-11-05T22:11:38Z）；docs 表仍无 kimi 行且仍写 v1.6.0，「表无行」不构成反证。llm-d v1.0.0 细节补充（Responses API `previous_response_id`、agent-aware serving、组件对齐表、$0.11/11.5× 厂商自报；tag published_at 未验证，日期以 IBM 公告 2026-10-20 计）。Conformal Cascade Theorem 4 期望成本闭式 E[Cost]=c₁+c₂·Pr[|C₁|≠1]（80% 接受率算例 1.54，比全强便宜 43%）与校准保证落点 α~Kα（α=0.10 时 MedMCQA 0.130、ARC 0.124，均在 Kα=0.20 内；部署规则：按 Kα 留边，承诺 5% 误差取 α=2.5%）。CascadeFilter 联合阈值（arXiv:2611.08367v1，逐层调阈非最优，须联合网格搜索并扣除多重检验预算）。长上下文定价悬崖 cₖ(L)：Haiku 5.5 官方定价页一手（100k 处 5×，99k/101k 算例 $0.0104 vs $0.0530）；其余三家（GPT-5.5/GPT-6 272k、Grok 200k、Gemini 200k）二手交叉，未验证暂定。Beyond K/V Caches 跨模型状态交接（arXiv:2611.08133v1，保留率 89.88%/94.11%，提速 21.66×/21.91×，论文自报，合成输入计时）。复核戳：LiteLLM 仍 v1.108.3（无 1.108.4/1.109 稳定版，仅 rc.1；changelog 仍无 v1.108.3 条目）、vLLM 仍 v0.33.0（0.33.1/0.34.0 tag 页均 404；下一稳定窗口约 2026-11-10 周二）、CVE-2026-114612 厂商逐字确认仍缺、Portkey CVE-2026-110719 仍无修复版、Railway 最终报告与 OpenCode post-mortem 仍未发布（opencode.ai/blog 仍 404）、六篇路由论文均无新版、arXiv 周日 0 提交。GitHub API 本轮仍 403 限流，AgentRouter v1.2.1 与 llm-d v1.0.0 的 tag published_at 因此未验证。
- 已同步进 skill 参考：api-routing.md 改写 OpenRouter 基准表段（一手升级 + 聚合口径注记）、改写 Dynamo kimi tag 段（直接再验 + v1.6.2）、llm-d 段追加细节补充、Conformal Cascade 段追加成本闭式与校准落点、文件末尾追加 CascadeFilter 联合阈值段、长上下文定价悬崖段、Beyond K/V Caches 段与本轮复核戳段。
- 推翻的旧说法：OpenRouter 基准表「二手转述、未直读」（现一手复核）；Dynamo kimi tag「未直接再验」（现直接再验）；delta.md R3 原「llm-d 基线停于 v0.5，v0.6–v1.0 回填」说法（与技能正本现状冲突，整合前已改写为增量补充）。
- 剩余未核实缺口（更新）：1. CVE-2026-114612 厂商逐字确认 1.106.6 含修复。2. Portkey CVE-2026-110719 修复版。3. Railway 最终报告与 OpenCode post-mortem。4. LiteLLM LTS"新端点"清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）。5. AgentRouter v1.2.1 published_at 未验证。6. llm-d v1.0.0 tag published_at 未验证（R3 子项，未验证暂定）；v0.6–v0.10 回填。7. Portkey #1372 与 CVE-2026-110719 消歧；Portkey 1.15.1/1.15.2 有无 Release 对象。8. 三家厂商定价悬崖阈值定价页原文直读（R7 残余，未验证暂定）。9. 真实流量级联校准实测仍无一手证据。10. 已解决：OpenRouter 基准表一手复核；Dynamo kimi tag 直接再验；Conformal Cascade 成本闭式与校准落点。

## 2026-11-09 晨轮（第 17 轮，窗口 2026-11-08 23:37 → 2026-11-09 05:37）

- 本轮增量 R1–R10（10 条），全部落入 `references/api-routing.md`；版本 0.16.0 → 0.17.0。
- R1：llm-d v1.0.0 tag published_at 一手定稿 2026-10-20T02:30:10Z（GitHub Releases API；release id 417939524；执行者自验独立抽查确认），上一轮 R3 暂定子项关闭。
- R2：R7 其余三家厂商定价悬崖阈值全部升级为一手（OpenAI GPT-5.5/GPT-6 >272K、xAI Grok ≥200k、Google Gemini >200k；执行者自验独立抽查三家官方定价页全部吻合）；Google 输出 1.5× 纠正（上一轮 2× 错误）；整单重定价句不在 Developer API 定价页上的限定（全页 7,930 行 0 命中，Vertex AI 侧仅二手引文）；上一轮 R7 暂定关闭。
- R3：AgentRouter v1.2.1 published_at 一手 2026-11-06T16:24:47Z，tag 创建 2026-10-28。
- R4：Portkey 1.15.1/1.15.2 Release 对象存在；1.15.2 为 2026-01-12T08:46:07Z；"released 2026-10-21" 为测试执行日期，撤回。
- R5：Portkey #1372 消歧——引入校验器的 PR，非 CVE 修复；CVE-2026-110719 仍无厂商确认修复版本。
- R6：**Dynamo 存在性裁定（自验驱动修正）**：路由 A 路「v1.6.2 不存在 / 最新稳定 v1.6.1」结论被执行者以 PyPI 一手驳回（ai-dynamo 1.6.2 轮子 2026-11-05 上传）；最新稳定仍为 v1.6.2；文档侧仍滞后于 v1.6.0（安装页）/v1.6.1（版本表）；原结论不入技能正文。
- R7：LiteLLM 状态束——仍无 v1.109 正式版（最新 RC v1.109.0-rc.1）、无 v1.108.4（最新 LTS 补丁 1.108.3，2026-11-07）、v1.110.0-dev.2 2026-11-08T02:25:47Z（窗口外）、LTS 端点清单仍未公布。
- R8：vLLM 状态——仍无 v0.33.1/v0.34.0，最新稳定 v0.33.0（2026-11-02）。
- R9：llm-d v0.6–v0.10 回填（v1.0.0 Release 组件表一手；博客日期 v0.7 2026-05-12、v0.8 2026-06-24、v0.9 2026-08-28；v0.10.0 = 2026-09-29 二手双源，**新暂定子项**；v0.6 日期仍缺失；无 v1.0.1/v1.1.0）。
- R10：basal engine 2.5 级联（厂商自报，边界项）：--escalate pico→mini→ultra，阈值 0.95，同精度 ±0.5pt、约 1.5× 请求/H100。

### 剩余未核实缺口（本轨）

- CVE-2026-114612 厂商逐字命名 1.106.6 仍缺失（GHSA 固定表止于 1.104.0-rc.2）。
- Portkey CVE-2026-110719 厂商确认修复版本仍缺失。
- Railway 终报、OpenCode post-mortem 仍未发布。
- Dynamo v1.7.0-kimi-k3.1-pro-dev.1 搜索未命中。
- LiteLLM LTS "new endpoints" 清单仍未公布。
- llm-d v0.6 日期已解决（2026-11-09 中午轮：v0.6.0 published_at 2026-04-03T16:48:54Z、v0.10.0 2026-09-29T00:00:25Z，均 GitHub Releases API 一手；R9 暂定已划除）。

## 2026-11-09 中午轮（版本 0.18.0；窗口 2026-11-09 05:37 → 11:37 CST）

- 本轨增量（R1–R8）：llm-d v0.10.0/v0.6.0 日期一手闭合（R9 暂定划除）；CVE-2026-114612 与 Portkey CVE-2026-110719 复核戳（厂商确认与修复版仍缺）；中继审计 arXiv:2611.07650 与 ADP-HT arXiv:2611.05835 新增；Cloudflare 模型别名新增（标题级，细节未读，R6 暂定）；OpenRouter 十月数据补充（二手双源，R7 暂定）；定价悬崖复核戳 + DeepSeek/Kimi 无悬崖（二手多源一致，R8 子项暂定）。窗口内无网关新发布；LiteLLM 仍无 v1.109 正式版/无 v1.108.4、vLLM 仍无 v0.33.1/v0.34.0。
- 纠正：路由 B 路「Haiku 5.5 悬崖新发现」为重复（技能正本已有该条目一手记录）；路由 B 路「OpenAI 整单重定价官方页未明文」与既有一手证据冲突，不采纳；ServerBench 仅二手转述，不入技能。
- 剩余未核实缺口：arXiv 周一 2026-11-09 批次系统枚举（API 限流后补查）；真实流量级联校准（最大缺口）；Cloudflare 别名原文直读；OpenRouter 十月回顾页直读；DeepSeek/Kimi 官方定价页直核；CVE 厂商逐字确认；Portkey CVE 修复版；Railway 终报；OpenCode post-mortem；LiteLLM LTS 端点清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）。

## 2026-11-09 晚间轮（版本 0.19.0；窗口 2026-11-09 11:37 → 17:37 CST）

- 本轨增量（R1–R12）：上一轮暂定 R6/R7/R8 全部一手闭合或缩小——Cloudflare 模型别名原文直读升级（preferred/smart 双算法、`options.routing` 覆盖、100/1–50/1–64 上限、smart 排序信号仍未披露）；OpenRouter 十月数据官方页直读升级（21% 为 10 月 26 日当周请求占比、缓存命中率中位数 >90%、−57%/−70% 独立重算无误、价差 6.2×；数字仍厂商自报）；DeepSeek 无长度悬崖一手闭合（峰时 01:00–04:00 与 06:00–10:00 UTC、非峰半价、页内无长度分层定价），Kimi 三价目仍暂定。首个生产流量级联校准证据 CascadeRec arXiv:2611.08566v1（生产 A/B >1 个月、成本 ~−10%、质量持平、延迟仅定性；零延迟启发式路由器、阈值即配置、分段 rollout 门；τ=5 冷启动升级未入生产）与 HIRA arXiv:2608.21792v1（生产分类级联，30,233 篇、6.4% 人工纠正、Macro-F1 0.6218→0.8548，边界项）、Scout Before You Route arXiv:2611.09028v1（逐查询成本预测，完美成本知识省 29–61%、预测成本省 9–33%、20 样本冷启动、~2B scout 饱和）、置信门控可行性条件 arXiv:2611.05394v1（κ 成本比分离上下界）；Route Anything Table 5 文本问答床补充（"4 of 11 splits / family states / 0 dataset states"）；Dynamo 文档表滞后调和证据加强 + 补漏 tag v1.6.0-deepseek-v4.1-flash-dev.2-post.1；AgentRouter v1.2.0 日期一手；LiteLLM v1.108.3 published_at 微纠正（22:29:58Z 取代 22:25:38Z）。复核戳：LiteLLM 仍 v1.108.3、vLLM 仍 v0.33.0、CVE-2026-114612 厂商逐字确认仍缺、Portkey CVE-2026-110719 仍无修复版、Railway 终报与 OpenCode post-mortem 仍未发布；arXiv 周一 2026-11-09 批次已发布（683 新提交），路由相关 8 篇。
- 纠正：中午轮 v1.108.3 发布时间 22:25:38Z（newreleases）已被 GitHub API 22:29:58Z 取代。
- 剩余未核实缺口：Kimi K3 价目表渲染文本直读；Cloudflare smart-routing 排序信号；Scout 全文深读（下轮优先）；arXiv 周一 2026-11-09 批次 cross-list 部分未枚举；CVE 厂商逐字确认；Portkey CVE 修复版；Railway 终报；OpenCode post-mortem；LiteLLM LTS 端点清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-09 深夜轮（版本 0.20.0；窗口 2026-11-09 17:37 → 23:37 CST）

- 本轨增量（R1–R8）：Kimi K3 三价目一手闭合（官方价目页 JS 内嵌数据直读：$3.00/$0.30/$15.00，全窗平价；缓存写按 TTL 另计费 5min $3.00/1h $6.00；K2 新注册 2026-10-31 截止、年底全面下线）；Scout Before You Route arXiv:2611.09028v1 全文深读（两路独立深读数字一致：逐查询成本预测，完美成本知识省 29–61%、本方法省 9–33%、捕获 40–69%、72 组比较领先 69 组、20 样本冷启动保留 51–81%、~2B scout 饱和、部署前检查清单 9 条）；OpenRouter 11-06 公告全文一手核对（11 因子 + 实时价格/健康/缓存；auto/auto-code 注册表时间戳；基准板 $1.64/$6.43 仍二手待核）；在线学习新文 2611.09701 遗憾分析（理论侧：限制切换得次线性遗憾，遗憾小 ≠ 部署损失小）；TMLS bandit 综合（单一价格规则、O(d√T) 遗憾、无实验）；2611.08794 离策略评估 Onsager 修正（方法论相邻）；Jev Router 复核戳（created 2026-09-25，11 月唯一提及为公告引用，docs 未直开）；arXiv 周一批次 cross-list 枚举完成（53 篇，实质相关 3 篇）。复核戳：LiteLLM 仍 v1.108.3、vLLM 仍 v0.33.0（PyPI 有/GitHub Releases 无正式版条目，成因未查）、CVE-2026-114612 厂商逐字确认仍缺、Portkey CVE-2026-110719 仍无修复版、Railway 终报与 OpenCode post-mortem 仍未发布；RouteLLM 确认休眠 2 年余；Dynamo/llm-d 无新版；16 篇路由论文均无新版。窗口内新稳定发布为零。
- 纠正：Kimi K3 三价目「官方价目表未渲染、暂定」一手闭合。
- 剩余未核实缺口：OpenRouter 基准板 $1.64/$6.43 一手直读；Jev Router 官方文档直读；arXiv 11-07–11-09 批次 11-10 公告后补查；Cloudflare smart-routing 排序信号；CVE 厂商逐字确认；Portkey CVE 修复版；Railway 终报；OpenCode post-mortem；LiteLLM LTS 端点清单、v1.109 正式版、vLLM v0.33.1/v0.34.0（下一稳定窗口约 2026-11-10 周二）；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-10 凌晨轮（版本 0.21.0；窗口 2026-11-09 23:37 → 2026-11-10 05:37 CST）

- 本轨增量（R1–R9）：OpenRouter 基准板一手重算（$1.64 精确复现 $1.6427；$6.43 不可精确复现，最接近可辩护重算 $6.37，缺口 $0.06/0.9%；74% 在两种分母下均成立；swe_atlas_qa 无 Opus 高努力运行、各基准题量不等、terminal-bench Opus 反超三项限定须随引用；底层成本为厂商自报，仅聚合算术已核）；GHSA CVE-2026-114612 一手纠正（`patched_version` 字段 5 行全为 null，修复版仅为范围上界隐含；撤回 2026-11-08「patched_versions 已补全」措辞；1.106.6 含修复仍为谱系推断）；Cloudflare Auto Router 官方文档机制披露（候选池→分类→质量/成本评分，会话亲和，8 个路由原因值；分类器输入信号与权重仍未披露；四维/14 类/效用公式仍为 2026-10-26 二手）；模型别名 `smart` 算法仅一句话官方描述（排序信号全未披露）；Railway 独立事件已解决（第三方 IP 欺骗致 IP 被封，与 10-21 安全事件无关，最终报告仍未发布）；Route Anything 代码仓一手确认（19 stars，末推 2026-11-04，零新增实验）；OpenCode 仓库迁移 sst/opencode → anomalyco/opencode（301，本轮首次观测，迁移时刻未验证）；arXiv 11-07–11-09 批次仍未公告（submittedDate 三日均为 0，11-10 公告在窗口外，留待下轮）。复核戳：LiteLLM 仍 v1.108.3、vLLM 仍 v0.33.0（v0.33.0 GitHub Releases 条目仍缺，成因未查）、Dynamo 仍 v1.6.2、llm-d 仍 v1.0.0（窗口内仅 2 次 CI 提交）、RouteLLM 休眠 2 年 3 个月。窗口内新稳定发布为零。
- 纠正：OpenRouter 基准板「$6.43 厂商自报」（一手重算不可精确复现，最接近 $6.37，须附三项限定）；GHSA「patched_versions 已补全」（API 的 `patched_version` 字段 5 行全为 null，2026-11-08 更新措辞撤回）；上一轮「v1.109 正式版下周二窗口」预测（截至 21:48 UTC 未兑现）；「arXiv 11-07–11-09 批次无路由论文」（批次尚未公告，不得作此定论）；Cloudflare「四维/14 类/效用公式」官方口径（仅为二手转述，官方文档未披露）。
- 剩余未核实缺口：arXiv 11-07–11-09 批次 11-10 公告后补查（+ 11-10 当日批次）；Codex rust-v0.169.0-alpha.7（窗口外发布，本轮已登记）；CVE 厂商逐字确认；Portkey CVE 修复版；Railway 终报；OpenCode post-mortem（+ 仓库迁移时刻确认）；LiteLLM LTS 端点清单、v1.109 正式版、vLLM v0.33.1/v0.34.0；Cloudflare smart-routing 排序信号；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据；OpenCode 迁移后 release 列表复核。

## 2026-11-10 中午轮（版本 0.22.0；窗口 2026-11-10 05:37 → 11:37 CST）

- 本轨增量（R1–R10）：LiteLLM v1.108.4 (LTS) 窗口内发布（2026-11-09T23:00:51Z，8 项修复；仍无 v1.109 正式版，开发线最新 v1.110.0-dev.2；LTS「新端点」清单仍未列出）；Jev Router 官方文档首次一手直读（Updated 2026-11-06T23:55:27.108Z，181 行：三步机制、cost_tier 偏好、插件参数、metadata 管道、错误码 400/404/503；11 因子权重与决策模型 1.14 仍不在文档中）；生产级联校准运行要点深挖（B7：阈值按分段配置、τ=5 不照搬入生产、rollout 门 + kill-switch + 合规回退优先 + 人工纠正反馈环；CascadeRec 仍为唯一生产流量证据）；vLLM v0.33.0 Releases 条目缺失成因闭合（release 对象挂在 v0.33.0rc3，正式 tag 无 release 对象；v0.34.0rc0 已现）；Portkey CVE-2026-110719 复核（仍无修复版，GHSA 数组为空，main 源码仍默认 TRUSTED_CUSTOM_HOSTS，main 末提交 2026-05-25）；Dynamo 文档表滞后复核（仍称当前稳定 v1.6.0，最新稳定 v1.6.2）；OpenCode 仓库迁移确认（301，同一 repo id 975734319；迁移时刻 2026-01-02 仅二手）；11-07 批次路由论文摘要级登记 4 篇（MultiTurnRoute 2611.10848v1、SafeCache 2611.10418v1、AERO 2611.10750v1、SPARK-VLA 2611.10443v1 相邻）；复核戳：OpenRouter 无 11-06 后新博文、基准板未变（268 runs）、Cloudflare 两路由文档仍 Updated 2026-11-04、Railway 终报与 OpenCode post-mortem 仍未发布、llm-d 仍 v1.0.0、RouteLLM 休眠 27 个月。
- 纠正：GHSA「patched_version 5 行全 null、撤回 2026-11-08『patched_versions 已补全』措辞」（05:37 轮字段名错误——API 对象中根本不存在 `patched_version` 键，正确字段 `first_patched_version` 已填；05:37 轮的撤回本身被撤回，2026-11-08 措辞恢复）；上一轮「v0.33.0 GitHub Releases 条目缺失、成因未查」（成因已闭合）。
- 剩余未核实缺口：arXiv 11-08–11-10 批次公告后补查；CVE 厂商逐字确认 1.106.6；Portkey CVE 修复版；Railway 终报；OpenCode post-mortem（+ 迁移后 release 列表复核）；LiteLLM LTS 端点清单、v1.109 正式版、vLLM v0.33.1/v0.34.0；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（板面值 vs 重算 $6.37 差异仍待解释）；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-10 晚间轮（版本 0.23.0；窗口 2026-11-10 11:37 → 17:37 CST）

- 本轨增量（R1–R13）：LiteLLM 仍 v1.108.4（无 v1.108.5/v1.109；dev 线 v1.110.0-dev.2；LTS「新端点」仍无清单）；vLLM 仍 0.33.0（v0.34.0rc0 已现，34b780fe）；CVE-2026-114612 GHSA 登记范围细节（五区间止于 1.104.0rc2 之下，1.106.x 不在登记受影响范围内；厂商逐字确认仍缺，代码证据不变）；Portkey CVE-2026-110719 复核（仍无修复版，GHSA 数组为空；Feedly 二手「无官方补丁」；main 分支本轮未复查）；OpenCode v2.2.7 tag-only（tag fedc6f2 2026-11-10T07:52:42Z，Release 对象缺席）；Dynamo 文档表滞后复核（仍称 v1.6.0，仓库 v1.6.2；dev 线 v1.7.0-kimi-k3.1-pro-dev.1）；llm-d 仍 v1.0.0；RouteLLM 末提交仍 2024-08-10；Cloudflare smart-routing 排序信号仍未披露（两份官方文档全文直读，smart 仅一句话；默认池 8+23 未变）；OpenRouter 基准板 $6.43 穷举未重现（接口逐字节未变 192,596 字节/268 runs；Auto Code $1.6427 精确重现；Opus 18 组合 + 6 聚合变体无一重现，最接近 $6.37；terminal-bench-4.0 accuracy/correct_answers 不一致新线索；双引用口径）；Jev 1.14 一手确认（Models API decisions 列表创建 2026-10-14T23:43:37Z；`jev-router` 在全量列表而非 decisions 列表——R10 自验修正；11 因子权重仍未披露）；路由论文摘要级登记 5 篇（RADE 2611.11659、CompCRC 2611.11114、EARN 2611.11565、ShadowBench-MCP 2611.11720、PISA 2611.10193；MixRoute 排除；2611.12521 不得用于 Jev）；arXiv submittedDate 索引滞后方法证明（上一轮 7 篇 vs 本轮 25 篇；仅推翻 submittedDate 查询结论）；复核戳：OpenRouter 无 11-06 后新公告（经镜像，官方原文未直接打开）、Route Anything 仍 v1（stars 19→20）、Local or Escalate 仍 v1、Railway 终报与 OpenCode post-mortem 仍未发布（本轮结论基于二手追踪器）。
- 纠正：GPT-6.1 Sol changelog 条目归属——属 AI 产品组 changelog（2026-11-09 "Run the Codex agent loop on Cloudflare Workers"，Workers/Agents 语境），非 AI Gateway 产品 changelog；事实成立，归属页错（skill 正本无此条目，仅 track 记录纠正）。中午轮「11-08/11-09 批次尚未公告」表述收窄——仅 submittedDate 查询结论被推翻，cs.AI 列表页变通不受影响。
- 剩余未核实缺口：OpenCode v2.2.7 Release 对象复查；LiteLLM v1.109 正式版、vLLM v0.33.1/v0.34.0；CVE 厂商逐字确认 1.106.6；Portkey CVE 修复版（+ main 分支复查）；Railway 终报；OpenCode post-mortem；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（板面值 vs 重算 $6.37 差异仍待解释）；Jev Router 实际运行 1.13 还是 1.14、`jev-latest` 是否已切 1.14；11 因子权重；arXiv 11-08–11-10 批次尾部滞后复查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-11 凌晨轮（版本 0.24.0；窗口 2026-11-10 17:37 → 2026-11-11 01:09 CST）

- 本轨增量（R1–R15）：Railway 最终报告已发布（2026-11-10 官方博客：至多 1.85% 约 10 万工作区、353 构建历史、1,852 用户令牌、约 $74 万未授权消费，数字厂商自报；长期缺口闭合；另 QBCLHG1V 11-04–11-09 孪生源 IP 误封事件已解决，无 post-mortem）；OpenCode v2.2.7 Release 对象已出现（2026-11-10T10:37:15Z，tag fedc6f2 一致；上一轮 tag-only 记录被取代；v2.2.8 不存在）；LiteLLM 仍 v1.108.4（无 v1.108.5/v1.109 正式版；LTS「新端点」仍无清单；dev 线 v1.110.0-dev.2；v1.109.0-rc.1 内容：11 新厂商、`/v1/extraction`、Codex 工具搜索、部署级日限）；vLLM 仍 0.33.0（v0.34.0rc0 仅 tag、无 Release 对象，404；11-10 周二窗口已过）；Dynamo 文档表缺口扩大至缺两行（仍称 v1.6.0，仓库 v1.6.2；dev 线 v1.7.0-kimi-k3.1-pro-dev.1 与 v1.7.0-mistral-large-4-dev.1；文档 Deprecations 页新增「v1.7.0 - Upcoming」；nightly 1.7.0.dev20261109）；AgentRouter v1.2.1 发布（2026-11-06T16:24:47Z；与 AgentRouter.org 商用网关无关——后者目录缩减说法为单一二手源，未采用）；Cloudflare AI Gateway 三项窗口内更新（11-03 强制认证、11-04 模型别名、11-06 key 活动查看）；OpenRouter auto/auto-code 上线一手验证（Models API created 2026-11-05；上下文 2,000,000；全量目录 465 → 467，新增 google/gemini-4-carbon 对，created 2026-11-10T16:07:51Z）；Jev 1.14 切换计划升级为 TypeSafe 官方博客一手确认（约 2026-10-21，60 天/30 天弃用预警），但切换是否实际完成仍未证实（第三方解释器称仍指 1.13.0，矛盾未决）；OpenRouter 基准板 $6.43 穷举未重现第二轮（接口逐字节未变 192,596 字节/268 runs；Auto Code $1.6427 精确重现；新增 243 种组合 + 9 种聚合口径变体无一重现，最接近仍 $6.37；terminal-bench-4.0 accuracy/correct_answers 不一致仍在；双引用口径保留）；CVE-2026-114612（GHSA 登记范围无变化；v1.106.6 正文全文未提该 CVE）；Portkey CVE-2026-110719 复核（仍无修复版；main 分支复查：默认信任 localhost 成因仍在，文件最后提交 2026-03-25，main HEAD 仍 2026-05-25）；在线学习/路由论文摘要级登记（OrcaRouter 2605.30736v1、Bandit Formulations TMLS 调查「We ran no experiments」、ShadowBench-MCP 2611.11720、2611.11322/2611.12059/2611.12662 相邻；真实流量一手证据仍无）；arXiv 11-10 批次因公告滞后未枚举，结转下轮。
- 纠正：水位文件 LiteLLM v1.108.4 发布时间 23:00:55Z → 23:00:51Z（GitHub Releases API 实测，4 秒微纠正；skill 正本 api-routing.md 原已正确，仅 track 与水位文件纠正）。
- 剩余未核实缺口：LiteLLM v1.109 正式版、vLLM v0.33.1/v0.34.0；CVE 厂商逐字确认 1.106.6；Portkey CVE 修复版；OpenCode post-mortem；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（板面值 vs 重算 $6.37 差异仍待解释）；Jev Router 实际运行 1.13 还是 1.14、`jev-latest` 是否已切 1.14；11 因子权重；arXiv 11-08–11-10 批次尾部滞后复查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-11 凌晨轮（版本 0.25.0；窗口 2026-11-11 01:09 → 04:09 CST）

- 本轨增量（R1–R10）：RouteLLM 末提交纠正（Commits API 前 3 项：最新 `0b64fda` 2024-08-10T19:10:15Z「create /health endpoint」，推翻「最新组 2024-07-20」与「第三方 2024-08-10 不被该页支持」）；Dynamo 文档表口径修正（表行仍称 v1.6.0、缺 v1.6.1/v1.6.2 两行，但版本切换器已标 Latest v1.6.1）；OpenRouter 基准板（接口 198,876 字节、277 runs；亲自重算 Auto Code $1.6427 / Opus 5.5 $6.3675 不变，与板面 $6.43 缺口 $0.0625/0.97% 未变；自验重拉时已增至 283 runs，重算值不变）；目录 467 → 469（窗口内新增 baidu/ernie-5.5-flash-preview 与 baidu/ernie-light-dt-1.0-intl，created 2026-11-10T17:58:53Z / 17:21:00Z）；Jev 切换仍未证实（decisions 列表仍 36 个；"eleven"/"1.14" 0 命中；"factor"/"weight" 命中全为假阳性；11 因子权重仍未披露）；生产级联论文摘要级新增两篇（GAMCAL 2608.27244v1、Cortex AISQL 生产案例，校准 GAM 按 doubling schedule 在线按批重解阈值；AutoRelAnnotator CEUR-WS Vol-4250/paper781，三模型级联 + 每类 isotonic 校准，1.5 亿+ 离线标注；CascadeRec/HIRA 摘要级复核确认）；vLLM v0.34.0rc1 tag 短暂出现后被撤回（腿核查约 04:2x 时存在，04:39 自验复核时 refs API 已 404、matching-refs 仅剩 rc0；commit 68ad4b1 仍在 main，2026-11-10T20:11:14Z，晚于窗口关闭 2 分 05 秒，本就不归入严格窗口；R8 自验修正为双时点证据）；CVE-2026-114612（GHSA 登记范围无变化，`updated_at` 仍 2026-10-22；官方 release_notes 索引页全文检索 `114612` 零命中，站内搜索仅返回另两件 CVE 专文）；OpenCode post-mortem 仍未发布（opencode.ai/blog 两次实测 404，官网 v2 重构后无博客入口，下轮改用搜索/新路径复核）；复核戳：LiteLLM 仍 v1.108.4、vLLM 仍 0.33.0、Portkey CVE-2026-110719 仍无修复版（main 默认信任 localhost 仍在）、Cloudflare 两路由文档未变；arXiv 11-10 批次截至核对时尚未公告，结转下轮。窗口内新稳定发布为零。
- 纠正：RouteLLM「最新组 2024-07-20」（实为 2024-08-10 `0b64fda`）；Dynamo「最新稳定 v1.6.2 已体现在表行」（表行仍停 v1.6.0）；vLLM v0.34.0rc1「已发布」（tag 短暂出现后被撤回）；MixUA 为重复发现（已于 2026-11-10 中午轮记入，不重复入技能）；「只字未提机制」措辞在技能正本、track 台账、总报告中均无此措辞（已全文 grep 确认），Auto Router 三步流程句技能正本自 2026-11-09 起已有记录，不构成修正。
- 剩余未核实缺口：LiteLLM v1.109 正式版、vLLM v0.33.1/v0.34.0（rc1 tag 撤回后动向）；CVE 厂商逐字确认 1.106.6；Portkey CVE 修复版；OpenCode post-mortem（新路径/搜索复核）；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（板面值 vs 重算 $6.37 差异仍待解释）；Jev Router 实际运行 1.13 还是 1.14、`jev-latest` 是否已切 1.14；11 因子权重；arXiv 11-10 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-11 07:09 轮（版本 0.26.0；窗口 2026-11-11 04:09 → 07:09 CST）

- 本轨增量（R1–R10）：LiteLLM v1.108.5 LTS 窗口内发布（2026-11-10T22:51:37Z，5 PR/4 修复；同日旧线补丁 v1.104.9 与 v1.103.11）；Gemini CLI v0.68.0 正式版（npm 21:22:49Z，含 hooks 修复 #30111）+ v0.69.0-preview.0（21:11:43Z）；Codex rust-v0.167.1（21:32:01Z）+ rust-v0.169.0-alpha.9 预发布对象首次出现（22:09:09Z，结束两轮 tag-only）；vLLM v0.33.0 稳定版双路确认（tag 11-01 + PyPI 上传 2026-11-02T10:13:42Z），v0.34.0rc1 撤回再确认（最新 tag 仍 rc0）；Dynamo 仍 v1.6.2（文档表仍仅 v1.6.0，两补丁滞后；新增路由口径：Router policy YAML 优先于评分旗标，后者拟于 v1.7.0 移除；WorkerSet 加权随机、无请求粘性）；CVE-2026-114612（v1.106.6 全文再读仍仅 gzip 回退，无 CVE/GHSA/security 提及；厂商逐字确认仍缺——本轮为再确认，非新证据）；Portkey CVE-2026-110719（仍无修复版，npm 最新仍 1.15.2；Feedly 页 2026-11-10 更新仍称无官方补丁；v1.14.0 #1372 修复未完全堵住该向量）；OpenRouter 基准板接口已变（205,912 字节、287 runs，+10 全在窗口内；亲自重算 Auto Code $1.6427 / Opus 5.5 $6.3675 不变，与板面 $6.43 缺口 $0.0625/0.97% 未变；`openrouter/auto` 均值 $0.575708 首次记录；下轮判据：板面值是否随接口变化而动）；目录仍 469，零窗口内新增；Jev 切换仍未证实、11 因子权重仍未披露；Cloudflare smart-routing 信号/权重仍未披露（两文档逐字节未变）；LTS 1.108.1–1.108.4 均为修复，无新端点清单；arXiv 路由论文新增 IPS 估计量 2611.11712v1（摘要级，其余 4 篇已登记不重复）；OpenCode post-mortem 仍未发布（/blog 仍 404；第三方称 2026-10-17 披露、约 1.4% Console 用户密钥或泄、「大规模蒸馏攻击」，二手未核实；npm 1.19.5 与 GitHub 2.x 线关系未决）；在线学习阈值真实流量一手证据仍无（OrcaRouter 仍仅 v1；本轮新增方法论文均为方法/仿真或基准层面，未提供生产真实流量上 conformal/bandit 阈值的一手证据）。
- 纠正：SkillsBench 与 SkillzBench「未核实是否同一基准，不得混用」升级为已核实结论——不是同一基准（SkillsBench 是 arXiv:2602.12670 论文基准；SkillzBench 是 skillzbench.com 厂商目录）；CVE-2026-114612「厂商已逐字确认 1.106.6 含修复」不成立（04:09 轮已登记，本轮再确认）；vLLM v0.34.0rc1「已发布」上轮已撤回，本轮再确认。
- 剩余未核实缺口：LiteLLM v1.109 正式版、vLLM v0.33.1/v0.34.0（rc0 tag 日期仍未取得）；CVE 厂商逐字确认 1.106.6；Portkey CVE 修复版；OpenCode post-mortem（新路径/搜索复核）；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（板面值是否随接口变化而动）；Jev Router 实际运行 1.13 还是 1.14、`jev-latest` 是否已切 1.14；11 因子权重；arXiv 11-10 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-11 10:09 轮（版本 0.27.0；窗口 2026-11-11 07:09 → 10:09 CST）

- 本轨增量（R1–R9）：LiteLLM **v1.109.0 正式版**窗口内发布（2026-11-11T00:56:51Z，PyPI 00:53:12Z，852 条变更：新增 11 家 provider、Codex 工具搜索、部署级 rpd/tpd 日限额）；v1.110.0-rc.1 亦窗口内（00:06:06Z）；无 v1.108.6；LTS「新端点」独立清单仍未见。vLLM v0.34.0rc0 tag 日期闭合（轻量 tag，commit 34b780fe，2026-11-08T18:38:57Z）；v0.34.0rc1 撤回再确认。**`jev-latest` 已切 1.14 已证实**（TypeSafe 官方文档 Models 页逐字 `jev-latest` → `jev-1.14.0`，Updated 2026-11-10T23:29:17Z；OpenRouter 文档中心 + 1.14 模型页 + 第三方 11-04 实测探针三重印证；jevtypesafeai.com 裁定为过期页面；实际运行版本按 catalog 推断为 1.14，链条推断）；11 因子权重仍未披露。OpenRouter 基准板 287 → **298 runs**（+11，全为 terminal-bench-4.0，窗口内；213,692 字节，md5 `9ab1f91f…`）；亲自重算 Auto Code $1.6427 / Opus 5.5 $6.3675 不变，与板面 $6.43 缺口 $0.0625/0.97% 未变；目录仍 469，零新增；decisions 仍 36。IPS 估计量论文方法节已读（2611.11712v1，式 3.2–3.3；无 LLM 路由实验，路由应用为本项目转述）。arXiv 11-10 批次路由论文摘要级登记（PISA +2.09/+2.87pp、成本 −36%/−25%；MultiTurnRoute 成本低至 −20.3%；SafeCache、AERO、RoutingTrie、多 logger 预算采购、ODSA、WorkflowArena；11-11 批次截至 02:29 UTC 尚未公告）。OpenCode post-mortem 仍未发布（1.x 线止于 v1.19.5 旁证）。Portkey CVE 仍无修复版（文件最后提交 2026-03-25 已核实，缺口闭合；v1.15.1/v1.15.2 Release 对象均存在，纠正上轮页面所见）。
- 纠正：`jev-latest`「是否已切 1.14 仍未证实」→ 已证实；GHSA-h36v-27jx-93r3「未核实、不是 CVE 本身」→ 即 CVE-2026-110719 的 advisory 库条目本身（自验亲自打开 advisory 页裁决；仓库自有 advisories 列表仅含 CVE-2025-66405 一条，与全局库条目并存，不矛盾）。
- 方法注记：**OpenRouter 基准数据接口现需认证 cookie**（自验 curl ×2、browser.open、浏览器 live 任务四路均 401：curl "No cookie auth credentials found"、live 浏览器 "No user or org id found in auth cookie"；腿文件窗口内取回时仍开放）——下轮须换取回路径；本轮 R4 以腿文件一手记录（md5 + 字节数 + 增量）通过，自验未能重算。GitHub 未认证 API 限流（共享出口 IP）本轮六路均遇，改道 git 协议/注册表 JSON/`browser.open` 完成核查；llm-d/agent-router 最新 release 未复核（降格为未核查）。仓名勘误：`agentrouter/agentrouter` 与 `route-anything/route-anything` 非跟踪仓，正主为 theagentrouter/agent-router 与 InfRec-Cardinal/route-anything（技能正本已用正确仓名，无需修正）。
- 剩余未核实缺口：LiteLLM v1.109.0-rc.1 之后演进、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认 1.106.6；Portkey CVE-2026-110719 修复版；OpenCode post-mortem（新路径/搜索复核）；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（接口现需认证 cookie，下轮换取回路径）；11 因子权重；arXiv 11-11 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。

## 2026-11-11 13:09 轮（版本 0.28.0；窗口 2026-11-11 10:09 → 13:09 CST）

- 本轨增量（R1–R10）：LiteLLM **v1.106.8** 旧线补丁窗口内发布（2026-11-11T03:23:36Z，单条 backport #50861：usage.seconds 计费 + FLAC 守卫；发布说明 961 字节，CVE/GHSA 0 命中——自验实测修正腿文件 1,439 字节；无 v1.109.1/v1.110.0 正式版、无 v1.108.6 LTS）。vLLM v0.34.0rc1 撤回第三次确认（最新 tag 仍 rc0，PyPI 仍 0.33.0）。Dynamo 仍 v1.6.2（文档表仍仅 v1.6.0，465,759 字节逐字节相同；v1.6.2/v1.7.0-kimi-k3.1-pro-dev.1/AgentRouter `published_at` 正本已有，本轮仅重核）；llm-d 仍 v1.0.0（`pushed_at` 窗口内移动但默认分支零新提交，成因未定位，记为异常）。OpenRouter 基准板取回路径部分恢复（基准页经 browser.open 公开可读：13 benchmarks、2,345,192 task evaluations、last run Nov 11, 2026；$6.43 渲染文本与静态 HTML 均 0 命中，数据接口仍 401；298 runs 与 2,345,192 口径未调和，不得混淆、不得重算；双引用口径保留）。Jev 11 因子权重仍未披露（两文档 Updated 仍 2026-11-06；TypeSafe 戳现为 2026-11-10T23:29:17Z，内容逐字未变）；Auto Router 文档新注记（Auto Beta 实验通道、`cost_tier` 接受 xhigh/max 按 high 路由、默认策略更贵且池更宽）。arXiv 11-11 批次路由论文摘要级登记（PolyRoute 96.6%/TF-IDF 91.9%、SHADOWPREFILL 生产 trace 前缀命中 80–89%→44–65%/切换 1.5–3.2×、C²-Routing、SafeRoute-MAS +1.80pp/−12.76%；相邻：Precision on Demand、DRD-MOPD、Off-policy Excursion、Swanada）。OpenCode post-mortem 仍未发布（/blog 仍 404；新增第三方 ideabosque：截至 11-02 仍无根因/报告/损失数字，二手未核实；首页/changelog 各漂移 30 字节未定位）。Portkey CVE 仍无修复版（requestValidator 14,461 字节逐字节相同，文件最后提交 2026-03-25）；CVE-2026-114612 厂商逐字确认仍缺（GHSA `updated_at` 仍 2026-10-22，OSV `modified` 仍 2026-11-01）。Cloudflare 两路由文档逐字节未变（md5 已记录：9b8032d4…/61a2bc3a…）。在线学习阈值真实流量证据仍无（CascadeRec/HIRA 仍为仅有的两项生产级联证据；相邻证据 SHADOWPREFILL 生产 trace、Precision on Demand 小规模 live 门控、AutoRelAnnotator 离线管线登记，GAMCAL 未查到）。
- 剩余未核实缺口：LiteLLM v1.109.1/v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认 1.106.6；Portkey CVE-2026-110719 修复版；OpenCode post-mortem（新路径/搜索复核）；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（298 runs vs 2,345,192 task evaluations 口径调和；数据接口仍 401）；11 因子权重；arXiv 11-12 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据；llm-d `pushed_at` 移动成因。

## 2026-11-11 16:09 轮（版本 0.29.0；窗口 2026-11-11 13:09 → 16:09 CST）

- 本轨增量（R1–R10）：LiteLLM **v1.109.0 正文首次全文阅读**（61,295 字符；mid-stream fallback #41127、fallback 改列表 #48943、硬 rpm #48873、rpd/tpd 日限额；"extraction" 0 命中；LTS「新端点」清单仍未出现；版本本身为 10:09 轮已登记，本轮仅加正文阅读）；vLLM v0.33.0 有 tag 无 release 对象（404）、PyPI 无任何 0.34.x；Dynamo 文档表字节 −1（Fern buildId 轮换，表内容未变；"1.6.2" 以 `-F` 查两轮文件均为 0 命中）；**GHSA 时间戳纠正**（published/updated 均为 2026-10-27，13:09 轮 2026-10-22 值错误，执行者亲自查已认证 API 裁决）；Portkey CVE 仍无修复版（requestValidator 14,461 字节逐字节相同）；OpenCode post-mortem 仍未发布（/blog 仍 404）；llm-d `pushed_at` 窗口内再次移动（03:44:59Z → 05:33:09Z，成因未定位）；OpenRouter 基准板计数器 2,345,192 → 2,346,091（+899，live 累计），$6.43 仍 0 命中、接口仍 401；Jev 权重与 Cloudflare 排序信号仍未披露（两文档 md5 逐字节一致；目录仍 469/decisions 36）；arXiv 11-11 批次零遗漏零修订（678 条次/去重 501），11-12 批次尚未公告（预期 11-13 01:00 UTC）；在线学习阈值真实流量证据仍无（SHIFT-JEV 2610.34329v1 仅检索命中）。
- 纠正：路由 A 腿「v1.109.0 / v1.106.8 为窗口内新发布」不成立（两者分属 10:09 轮与 13:09 轮窗口且均已登记）；GHSA「`updated_at` 仍 2026-10-22T17:59:33Z / published 2026-09-30」错误（已认证 API 实测为 2026-10-27T15:04:14Z / 2026-10-27T15:04:15Z）；Dynamo 文档页 "1.6.2" 正则计得 3 处为通配误中（`-F` 定长查两轮文件均为 0 命中）；v1.109.0 正文 61,296 vs 61,295 已调和（前者为存盘字节数含末尾换行，后者为 API body 字段字符数，按既定口径写 61,295）。
- 剩余未核实缺口：LiteLLM v1.109.1/v1.110.0 正式版、vLLM v0.33.1/v0.34.0 与 v0.33.0 无 release 对象的原因；CVE-2026-114612 厂商逐字确认 1.106.6；Portkey CVE-2026-110719 修复版；OpenCode post-mortem；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（298 runs vs 2,346,091 task evaluations 口径调和；数据接口仍 401）；11 因子权重；arXiv 11-12 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据；llm-d `pushed_at` 移动成因；Dynamo v1.6.1/v1.6.2 变更内容逐条核对。

## 2026-11-11 19:09 轮（版本 0.30.0；窗口 2026-11-11 16:09 → 19:09 CST）

- 本轨增量（R1–R9）：窗口内新稳定发布为零。**vLLM v0.33.0 有 tag 无 release 对象的原因闭合**（issue #65662：release 对象错挂 rc3 tag，流程元数据错误；仍 open）。**Dynamo v1.6.1/v1.6.2 变更逐条核对**（emSign 根移除 breaking、Pillow 13、KVBM 描述修复；CPython 3.12.16、500→400、GC-freeze opt-in；文档表仍仅 v1.6.0）。**Dynamo 路由口径两项修正**（YAML 第 4 顺位；"weighted random" 无依据；session affinity 默认禁用）。CVE-2026-114612 compare 反证收紧（ahead_by 3 全为 gzip 回 port；GHSA 戳仍 2026-10-27；厂商确认仍缺；端点口径方法注记）。LiteLLM LTS「新端点」无独立清单。OpenRouter 计数器 +282（2,346,373），$6.43 仍 0 命中，401 报错体取回。arXiv 11-11 批次完整性复核闭合（678 条次；两篇 bandit 论文摘要级登记）。llm-d `pushed_at` 再次移动（09:25:17Z，ForkEvent 巧合）。LiteLLM/vLLM/Portkey/OpenCode 复核戳（均无新版）。
- 纠正：Dynamo「Router policy YAML takes precedence over scoring flags」与「WorkerSets are picked by weighted-random selection」均不成立（执行者亲自重读官方文档原文裁决：YAML 第 4 顺位；"weighted random" 页面内 0 命中）。
- 剩余未核实缺口：LiteLLM v1.109.1/v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Cloudflare smart-routing 排序信号；OpenRouter $6.43 口径调和；11 因子权重；arXiv 11-12 批次补查；llm-d `pushed_at` 移动成因；在线学习阈值真实流量证据仍无（CascadeRec/HIRA 仍为仅有的两项生产级联证据）。
- 下轮起点：先清上述缺口，再按子主题轮转（成本/延迟权衡定量模型）。

## 2026-11-11 22:09 轮（版本 0.31.0；窗口 2026-11-11 19:09 → 22:09 CST）

- 本轮增量：CascadeRec 全文深读（拉格朗日标量化 max Σ NDCG − λ·cost、Tier-2 约为 Tier-1 的 3–6 倍、R0–R3 谱系仅 R0 受评、生产 A/B ~1,000 请求/天 >1 个月、成本 −10% 质量持平）；HIRA 全文深读 + HiRA 消歧（Tobacco-3482 Macro-F1 0.9423，+17.4pp）；Counterfactual OCP 全文深读（Ω(ρ) 缺口、PW/DR-OCP、π_min=0.01 参考值；实验非 LLM 路由流量，缺口仍成立）；COMPASS 与 LLMRouterBench 摘要级登记；LiteLLM v1.110.0rc1/v1.109.0 正文核对（Rust 核常驻、拒绝配置停代理、停滞流显式失败）与 LTS「新端点」收敛（仅 v1.108.0 一批）；OpenRouter 计数器 +70,368、目录 469→466、decisions 仍 36；llm-d `pushed_at` 再次移动（巧合模式破裂）；CVE-2026-114612 新反证（changelog 索引页全文 0 命中）；arXiv 11-11 批次完整性再验闭合（678 条次逐字节相同）。
- 纠正：本轮路由轨无旧说法推翻。
- 剩余未核实缺口：LiteLLM v1.109.1/v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Cloudflare smart-routing 排序信号；OpenRouter $6.43 口径调和（未结台账，下轮优先补做）；11 因子权重；arXiv 11-12 批次补查；llm-d `pushed_at` 移动成因；2611.01747v1 疑似既有检索遗漏（下轮核查）；在线学习阈值真实流量证据仍无（CascadeRec/HIRA 仍为仅有的两项生产级联证据；Counterfactual OCP 为理论结果，非流量证据）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项），再按子主题轮转（成本/延迟权衡定量模型）。
## 2026-11-12 01:09 轮小节

- 本轮增量：LangChain 1.5.0 决策模型 + 模型路由/自动模式中间件（beta）交叉引用登记；veduis 生产分层路由器案例（作者自报，未独立核实，新增未验证暂定）；OMCE 2611.01747v1 真实遗漏确认；OpenRouter 博客 7 篇补登记 + 429 教程 fallback 语义 + Extractions API；LiteLLM LTS「新端点」澄清（仅 v1.108.0 一批，LTS 页滞后两个补丁）；arXiv API `<updated>` 元数据触碰异常方法论警示（不入技能正文）。
- 复核戳：LiteLLM v1.109.1/v1.110.0、vLLM v0.33.1/v0.34.0 均未出现；CVE-2026-114612 厂商逐字确认仍缺；Portkey CVE 仍无修复版；OpenCode post-mortem 仍未见；llm-d `pushed_at` 窗口内再次移动至 2026-11-11T16:18:52Z，成因仍未确认；OpenRouter 基准数据接口仍 401（$6.43 本轮未重算，台账项保留）；计数器 2,416,741 → 2,417,180（+439）；目录仍 466、decisions 仍 36；Jev 11 因子权重仍未披露；Cloudflare 排序信号仍未披露（方法修正：改以渲染文本 diff 为准）；Dynamo 表仍缺 v1.6.1/v1.6.2 两行（抓取口径修正：上轮保存文件实为 v1.4.1 版页面）；arXiv 11-12 批次尚未公告（公告 2026-11-13 01:00 UTC，路由 B 腿 00:00 UTC 不采纳）。
- 纠正：上一轮「OpenRouter 博客 11-06 后无更新」系镜像核查漏检，本轮官方 RSS 一手纠正（7 篇补登记）；arXiv 公告时刻以已裁决的 2026-11-13 01:00 UTC 为准。
- 剩余未核实缺口：LiteLLM v1.109.1/v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；llm-d `pushed_at` 移动成因；OpenRouter $6.43 口径调和（未结台账）；Jev 11 因子权重与实际运行版本；arXiv 11-12 批次补查；在线学习阈值真实流量证据（CascadeRec/HIRA 仍为仅有的两项生产级联证据）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项），再按子主题轮转（成本/延迟权衡定量模型）。

## 2026-11-12 04:09 轮小节

- 本轮增量：vLLM v0.34.0rc1 标签重新出现（新提交 52aeb580，窗口内；推翻连续三轮“已撤回”结论）；Extractions API 全文深读（三模型 + 定价 + alpha 端点，/docs/api-reference/extractions 为 404）；OpenRouter 基准板 13→18（2,417,640 task evaluations；抓取超窗 6 分钟，记为本轮首次捕获）、decisions 36→37（gemini-decision-1 为 decisions-only，不在 /models 列表）；SHADOWPREFILL 全文深读（切换税量化 + 方法 + 仿真限定）；LTS 页滞后三个补丁；Rust 核说法来源收紧（仅 Changelog 摘要条目）。
- 复核戳：LiteLLM 无 v1.109.1/v1.110.0/v1.108.6；vLLM 无 v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认仍缺；Portkey CVE 仍无修复版；OpenCode post-mortem 仍未见；llm-d `pushed_at` 本窗口未动（ForkEvent 未推动，削弱“ForkEvent 巧合”假说）；Dynamo 表仍缺 v1.6.1/v1.6.2 两行；Jev 权重与 Cloudflare 信号仍未披露；博客 RSS +576 字节未定位；arXiv 11-12 批次尚未公告（公告 2026-11-13 01:00 UTC）。
- 纠正：vLLM rc1“已撤回”结论被推翻（标签重新出现，新提交）；LiteLLM Rust 核说法来源收紧为仅 Changelog 摘要（GitHub 发布正文无 "rust"）。
- 剩余未核实缺口：LiteLLM v1.109.1/v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；llm-d `pushed_at` 移动成因；OpenRouter $6.43 口径调和（未结台账，连续四轮 401）；Jev 11 因子权重与实际运行版本；Cloudflare 智能路由信号；Dynamo 文档表缺行；arXiv 11-12 批次补查；在线学习阈值真实流量证据（CascadeRec/HIRA 仍为仅有的两项生产级联证据）；方法论页 Scans on record 读数冲突（3194 vs 3197 vs 首页 3,236）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项），再按子主题轮转（成本/延迟权衡定量模型）。

## 2026-11-12 07:09 轮（版本 0.34.0；窗口 2026-11-12 04:09 → 07:09 CST）

- 本轨增量（R1–R7）：LiteLLM v1.109.0 正式版（窗口外补漏，2026-11-11T00:56:51Z，982 PR，5 项需行动变更：字符串 fallback 目标改为模型名列表、disk 缓存后端移除、enforce_model_rate_limits、Gemini 3.6/3.5 Flash-Lite 须 drop_params、Lens 独立 chart；亮点：流式中途 fallback 续接、LITELLM_OFFLINE_MODE、Admin UI）；v1.108.6 不存在；LTS 页滞后 3 个补丁（仍写 1.108.2），旧 LTS 路径 404；vLLM v0.34.0rc1 标签仍在 52aeb580 未动，无 v0.33.1/v0.34.0；OpenRouter 基准板计数器刷新（18 benchmarks、2,418,980 task evaluations +1,340、6 routers、last run Nov 11, 2026），$6.43 第 4 轮仍 401（执行者自验复核为第 5 次），沿用 $6.37；目录 466→467（pareto-26.10-preview，created 2026-10-01，窗口外滞后），decisions 仍 37；Jev/Auto Router 文档窗口内再发布、正文零变化（dateModified 2026-11-11T22:08:37Z，渲染文本仅配置 blob 差异；方法注记：dateModified 跳变不得单独作证）；11 因子权重仍未披露；Cloudflare 两路由文档 md5 相同，smart 信号仍未披露；CVE-2026-114612（GHSA 未更新，v1.106.6 与 v1.109.0 正文均 0 提及）、Portkey CVE-2026-110719（仍无修复版，最新 1.15.2）、OpenCode post-mortem（仍未发布，opencode.ai/blog 404）复核戳；Dynamo 文档表缺口缩小为仅缺 v1.6.2 一行（文档站已为 v1.6.1）；路由论文方法节深读四篇（Route Anything §2/§5、Conformal Cascade v3 §3、RouteBalance v2 §3、OMCE §3 相邻）；arXiv `<updated>` 触碰异常自行消解（暂定子项划除）；llm-d `pushed_at` 窗口内再次移动（22:03:26Z），成因仍未确认；在线学习阈值真实流量证据仍无；arXiv 11-11/11-12 批次路由论文为零（最近三篇摘要级登记：VATC 2611.14211v1、ARoute 2611.13209v1、SHADOWPREFILL 2611.13176v1）。窗口内新稳定发布为零（v1.109.0 为窗口外补漏）。
- 纠正：Dynamo 文档表「缺 v1.6.1/v1.6.2 两行」（仅缺 v1.6.2）；arXiv 公告时刻「11-12 批次公告 2026-11-13 01:00 UTC」（批次混淆，11-11 提交日批次应为 2026-11-12 01:00 UTC）。
- 剩余未核实缺口：LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem（博客路径 404，是否迁移待确认）；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（未结台账，连续五轮 401）；Jev Router 实际运行 1.13 还是 1.14、`jev-latest` 是否已切 1.14；11 因子权重；arXiv 11-11/11-12 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据；llm-d `pushed_at` 移动成因。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项），再按子主题轮转（成本/延迟权衡定量模型；PolyRoute 全文深读候选）。

## 2026-11-12 10:09 轮

- 本轮增量（D-R01–D-R12，落入 skill/references/api-routing.md）：LiteLLM 窗口内三连发 v1.106.9/v1.108.6/v1.109.1（均含安全回 port #51254，与 CVE-2026-114612 无关；v1.110.0 正式版仍无；LTS 页滞后扩大到 4 个补丁）；Railway 最终报告已发布（2026-11-10，长期缺口关闭，数字厂商自报）；llm-d `pushed_at` 成因闭合（gh-pages 徽章推送 5b04fd0 + PR #3039 squash 合并 edb6405，秒级证据；方法注记：Events API 遗漏 PushEvent，今后优先查分支 HEAD）；PolyRoute 全文深读（arXiv:2611.13843v1，基准论文非路由机制，无成本/延迟权衡模型；TF-IDF 91.9%、选项名轮换 96.6%→6.5%、分辨力上限约 4pp）；OpenRouter $6.43 第六轮 401（67 字节逐字节一致；计数器 2,418,980 → 2,416,209，−2,771 异常待观察；未结台账保留）；复核戳：CVE-2026-114612 厂商确认仍缺、Portkey CVE 仍无修复版、OpenCode post-mortem 仍未发布（仓库已迁移至 anomalyco/opencode）、Cloudflare/Jev 信号仍未披露、vLLM/Dynamo/RouteLLM/AgentRouter；arXiv 11-11 批次截至 02:25 UTC 不可见（%20 方法纠正；三篇批次归属纠正；技能 B 腿"真零"结论不采纳，下轮以公告日列表页 + id_list 交叉核对裁决）。
- 纠正：LiteLLM v1.108.6"不存在"（本轮确认存在且窗口内发布）；VATC/ARoute/SHADOWPREFILL 的 11-11 批次归属（实为 11-09/11-10 批次）；OpenRouter 博客"无新篇"（本轮新增 3 篇）；LTS 页滞后 3→4 个补丁；arXiv 范围查询 `+TO+` 编码（须用 `%20`，此前 0 结果结论可信度下调）；llm-d `pushed_at` "成因未确认"（本轮闭合）。
- 剩余未核实缺口：LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem（博客路径 404，是否迁移待确认）；Cloudflare smart-routing 排序信号；OpenRouter 基准板 $6.43 口径（未结台账，连续六轮 401；计数器 −2,771 异常待复核）；Jev Router 实际运行 1.13 还是 1.14、`jev-latest` 是否已切 1.14；11 因子权重；arXiv 11-11/11-12 批次补查；在线学习（conformal/bandit）阈值在真实流量上仍无一手证据。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项），再按子主题轮转（成本/延迟权衡定量模型）。

## 2026-11-12 16:09 轮（13:09 轮补做轮）

- 本轮增量（D-R01–D-R07，落入 skill/references/api-routing.md）：Cloudflare 附加池计数纠正 23→25（渲染文本逐行计数；rb3/rb4 md5 相同，自 10:09 轮起即为 25）；Cloudflare 别名/Auto Router 文档逐字节相同，`smart` 排序信号仍未披露；OpenRouter 文档渲染文本零变化（decisions 37 / models 467 / 博客零新篇；11 因子权重仍未披露）；OpenRouter $6.43 第七轮 401（执行者尾窗亲自 curl；计数器 2,416,011，−198 异常待观察）；LiteLLM v1.110.0-rc.2 预发布（执行者亲自复核；无新稳定版；vLLM 仍 v0.33.0rc3）；arXiv 11-11 批次路由论文完整枚举（已登记 5 篇 + 9 篇新候选摘要级登记；深挖锚点 2611.12953 协调臂乘子、2611.13412 前缀缓存亚稳态；DMR 现为 v4）。
- 纠正：Cloudflare 附加池"23 个"→"25 个"（正文内改写）；OpenRouter 第七次 401 的口径注记（路由腿"不计为第七次"仅指 rb4 集内无取证，第七次为执行者尾窗自验取证）。
- 剩余未核实缺口：arXiv 11-12 批次补查；OpenRouter $6.43 口径调和（未结台账，连续七轮 401；计数器 −198 异常复核）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Dynamo 文档表缺行；Cloudflare 附加池 25 vs 23 出入复核（扩容还是计数口径差）；Jev 11 因子权重；DMR v4 修订内容；2611.12953/2611.13412 全文深读；在线学习阈值真实流量证据仍无（CascadeRec/HIRA 仍为仅有的两项生产级联证据）。
- 下轮起点：先清上述缺口，再按子主题轮转（成本/延迟权衡定量模型：2611.12953 全文深读、2611.13412 前缀缓存亚稳态全文深读）。

## 2026-11-12 19:09 轮（第 37 轮）

- 本轮增量（D-R01–D-R07，落入 skill/references/api-routing.md）：Cloudflare 附加池 25 vs 23 出入闭合（渲染文本 md5 九连相同，23 为既往漏计，作废）；OpenRouter 第八次 401（计数器 2,416,011 → 2,416,209，+198 恰抵上轮 −198；目录 467 → 468，`moonshotai/kimi-k3.1-pro:batch` 滞后上架；decisions 仍 37；博客 177 → 178）；LiteLLM v1.106.9 未具名安全回退（PR #51261，不得误读为 CVE 确认）与 vLLM v0.34.0rc2；Dynamo 文档表缺口口径调和（releases-data 表仍无 v1.6.1/v1.6.2 行，与版本化文档站已为 v1.6.1 不矛盾）；2611.12953 全文深读（协调臂汇合均值 1.4–4.7× tokens / 2.6–5.8× 延迟，仅 Orch-v2 显著优于基线，§5 12-SOP 数字不得引用）；2611.13412 全文深读（前缀缓存亚稳态：9/12 配对仅突发臂持久退化，budgeter 7/7 <5 分钟恢复，N*≈18、排干 0.013/s，§8 明言无生产证据）；DMR v4 致谢-only 修订（diff 仅 26 行，2.44×/−39% 不变）。
- 纠正：Cloudflare 附加池"23 个"缺口正式闭合（既往漏计）；Dynamo 表"仅缺 v1.6.2"改为"表缺 v1.6.1/v1.6.2 两行"（与版本化文档站不同页）；2611.12953 §5 数字不得引用；2611.13412"吞吐约减半"为约数（引用实测 0.35–0.46 vs 1.27 turns/s）；OpenRouter 计数器 −198 异常本轮 +198 回弹（原因仍不明）。
- 剩余未核实缺口：arXiv 11-12 批次枚举（公告 2026-11-13 01:00 UTC）；OpenRouter $6.43 口径调和（未结台账，连续八轮 401；计数器 ±198 回弹原因不明）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认（PR #51254 内容下轮追问）；Portkey CVE 修复版；OpenCode post-mortem；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；2611.12953 §5 数字待作者 v2；2611.13412 生产侧验证；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口，再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-12 22:09 轮（第 38 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing.md）：PR #51254 内容追问闭合（6 字段禁令 + 401 机制 + 豁免项，回 port #51263/#51264/#51260/#51261；与 CVE-2026-114612 无关）；vLLM v0.34.0rc2 Release 对象存在（窗口外，PyPI 无 0.34.x），v0.33.1 不存在；OpenRouter 第九次 401（计数器 2,416,209 → 2,416,014，−195 回落；目录 468 未变；博客 178 → 183；**decisions 计数口径失效**：旧路由 /decisions 404，已迁 /labs/decisions（JS 渲染），旧值 37 记未决、不沿用）；Cloudflare/Jev 文档零变化（Auto Router md5 十连相同；jev-latest 仍 1.14.0）；llm-d `pushed_at` 窗口外再次移动（14:20:09Z，分支未定位，新未定位项）；AgentRouter 本轮未复核（GitHub API 403 限流），沿用 v1.2.1；11-11 批次 KV/级联新论文为零（相邻候选 2611.12959/2611.14413 摘要级登记 + BadMoE v3；DMR API `updated` 字段异常记为现象，不写成 v5）。
- 纠正：无（本轨无正文内改写；decisions 37 由“本轮实测值”改为“未决、不沿用”，由 D-R06 新戳覆盖）。
- 剩余未核实缺口：arXiv 11-12 批次枚举（公告 2026-11-13 01:00 UTC）；OpenRouter $6.43 口径调和（未结台账，连续九轮 401；计数器 ±198/−195 回弹原因不明；decisions 新计数方法待定）；llm-d `pushed_at` 14:20:09Z 移动分支定位（先查各分支 HEAD 提交时间戳）；AgentRouter v1.2.1 补做复核；DMR API `updated` 字段异常复核；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（AgentRouter 与 llm-d 优先），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-13 01:09 轮（第 39 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing.md）：OpenRouter $6.43 第十次 401（计数器 2,416,014 → 2,416,212，+198；目录 468、博客 183 均零增减；$6.37 沿用值不得写成重算）；decisions 新计数方法未定（旧索引页已改为单 recipe 的 Decisions Lab，JSON-LD 仅 1 项 Platform race；/api/v1/decisions 与 /api/v1/alpha/decisions 均 404；47 个 JS chunk 已拉取核查，无列表拉取端点；旧值 37 仍记未决）；AgentRouter v1.2.1 一手证实（403 限流已解除）；llm-d `pushed_at` 14:20:09Z 定位为 gh-pages 3966feb，窗口内又移至 16:26:04Z（ce173b2），窗口内徽章推送共 8 次，成因模式确立；LiteLLM v1.110.0 正式版仍无（最新稳定 v1.109.1；LTS 仍滞后 4 补丁）；vLLM v0.34.0rc2 纠正为有 tag、无 Release 对象；CVE 厂商确认仍缺、Portkey 修复版仍无、OpenCode post-mortem 仍无、Dynamo 表仍缺两行；Cloudflare/Jev 文档零变化（Auto Router md5 十一连相同；jev-latest 仍 1.14.0）；11-11 批次 KV/级联新论文为零，R7 继续暂定；11-12 批次尚未公告。
- 纠正：vLLM v0.34.0rc2「有 Release 对象」（实为仅 tag，「2 assets」为 tag 自动生成的源码压缩包）；DMR「API updated 字段异常」（上轮误读 feed 级 `<updated>`，条目级在上轮证据文件内已是 2026-11-10T02:19:39Z；方法口径纠正为一律取 entry 内 `<updated>`）。
- 剩余未核实缺口：arXiv 11-12 批次枚举（公告 2026-11-13 01:00 UTC）；OpenRouter $6.43 口径调和（未结台账，连续十轮 401；计数器 +198 回弹原因不明；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项；decisions recipe 数盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-13 04:09 轮（第 40 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing.md）：OpenRouter benchmarks API 第十一次 401（67 字节逐字节一致；计数器 2,416,212 → 2,415,516，−696，首次分项定位 GPQA Diamond −594 + τ²-Bench Airline −102；目录 468 → 473，窗口内新上架 `google/gemini-4-hydrogen` 及其 `:batch` 变体；$6.37 沿用值不得写成重算）；decisions 新方法仍未定（/labs/decisions JSON-LD 仍 1 项 Platform race；两 API 端点仍 404；旧值 37 记未决）；Extractions 指南页窗口内更新（dateModified 2026-11-12T19:40:36Z，新增 `perplexity/pplx-decider-v1.3` 与 `perplexity/pplx-decider-flash-v1.3`，`locate` 能力，模型表 4 → 6；端点仍 alpha，定价未给出）；复核戳：LiteLLM 最新稳定仍 v1.109.1；vLLM 无 v0.33.1/v0.34.0，v0.34.0rc2 仍仅 tag、无 Release 对象；AgentRouter 仍 v1.2.1；Dynamo GitHub 仍 v1.6.2 但文档表仍缺 v1.6.1/v1.6.2 行、稳定仍写 v1.6.0；CVE-2026-114612 厂商逐字确认仍缺；Portkey CVE 修复版仍无；OpenCode post-mortem 仍无；Railway 最终报告仍无（初步报告页 dateModified 变为 2026-11-10T14:05:19Z，正文仍称调查进行中，定性为元数据级修改）。
- 方法纠正：arXiv API `submittedDate` 检索索引滞后约 2 天，范围查询 0 结果不得作为批次证据，今后以列表页 `list/cs.AI/new` md5 为准；arXiv 11-12 批次尚未公告（公告 2026-11-13 01:00 UTC），结转下轮。
- 剩余未核实缺口：arXiv 11-12 批次枚举（公告 2026-11-13 01:00 UTC）；OpenRouter $6.43 口径调和（未结台账，第十二次重试；计数器下调成因；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Railway 最终报告；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；Perplexity Decider 定价；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项；decisions recipe 数盯梢；arXiv 11-12 批次枚举），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-13 07:09 轮（第 41 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing.md）：OpenRouter benchmarks API 第十二次 401（67 字节逐字节一致；执行者自验为第十三次；计数器 2,415,516 → 2,416,041，+525，**恒等式首次双向闭合**：计数器恒等于 14 个 taskCount 之和，VGI +1,756、GPQA −981、τ² −250；「148 models」纠正为 GPQA 卡值，本轮 149；$6.37 沿用值不得写成重算）；decisions 新方法仍未定（/labs/decisions 逐字节相同，numberOfItems 仍 1；两 API 端点仍 404；旧值 37 记未决）；目录 473 → 475（新增 2 个均为滞后上架，窗口内新增 0）；博客 183 篇不变；LiteLLM v1.108.7 窗口内发布（LTS 补丁，14 PRs；1.108 线补丁数升至 7；v1.110.0 仍未出现，最新稳定仍 v1.109.1）；Perplexity Decider v1.3 定价一手确认（v1.3 $0.02、flash-v1.3 $0.01，输出免费；OpenRouter 侧仅免费变体 `:free`，decisions 模型不在 chat Models API 目录内）；Extractions 指南页窗口内新增 Schematron 两模型（dateModified 2026-11-12T20:22:09.568Z，模型表 6 → 8；API 参考页滞后，定价未给出）；llm-d `pushed_at` 口径修正（16:26:04Z → 22:40:34Z，驱动者为 test-nightly 分支推送而非 gh-pages 徽章；`pushed_at` = 仓库任意 ref 最近推送时刻）；复核戳：GHSA 全局端点五条填值与 04:09 轮一致无变化（RA 腿「本轮首次观测」为端点比较基准错误）；厂商逐字确认 1.106.6 含修复仍缺；vLLM 无 v0.33.1/v0.34.0，rc2 仍仅 tag；AgentRouter 仍 v1.2.1；Dynamo 表仍缺两行；Portkey CVE 修复版仍无；OpenCode post-mortem 仍无；Railway 最终报告仍无；Cloudflare 三文档逐字节相同（Auto Router md5 十三连相同）；Jev 权重与 Cloudflare 排序信号仍未披露。
- 纠正：计数器波动「原因不明」（恒等式双向闭合，今后逐卡归因）；「148 models 为板面总值」（实为 GPQA 卡值，本轮 149）；llm-d pushed_at「gh-pages 徽章 +1 秒」（修正为任意 ref 最近推送时刻）。
- 剩余未核实缺口：arXiv 11-12 批次枚举（公告 2026-11-13 01:00 UTC）；OpenRouter $6.43 口径调和（未结台账，第十三次重试；计数器下调成因；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Railway 最终报告；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；Schematron 定价与 API 参考页补登；llm-d test-nightly 分支是否被删除；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项；decisions recipe 数盯梢；arXiv 11-12 批次枚举），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-13 10:09 轮（第 42 轮）

- 本轮增量（D-R01–D-R14，落入 skill/references/api-routing.md）：OpenRouter benchmarks API 第十四次 401（67 字节逐字节一致；执行者自验为第十五次；计数器 2,416,041 → 2,417,024，+983，**恒等式第三轮闭合**：14 个 taskCount 之和；GPQA +594、VGI +438、τ² −49；$6.37 沿用值不得写成重算）；decisions 新方法仍未定（/labs/decisions 86,032 字节，numberOfItems 仍 1；两 API 端点仍 404；旧值 37 记未决）；目录 475 持平、博客 183 → 184（新篇 "Reranking in RAG"，sitemap 已迁 /sitemaps/blog.xml，旧端点 404）；LiteLLM 仍 v1.109.1、vLLM 仍 v0.33.0（rc2 仍仅 tag）；CVE-2026-114612 缺口问法修正（GHSA 受影响区间止于 1.104.0rc2 之下，1.106.x 从不在登记范围内；修复提交 1d5fdc87 与五分支版已确认；厂商逐字确认仍缺）；Portkey CVE 修复版仍无（1.15.2，仓库零提交自 2026-05-25）；Railway 最终报告再确认已发布（04:09 轮「仍无」指初步报告页）；OpenCode 官方博客仍 404、无自有 post-mortem；Dynamo 文档表缺口缩小（v1.6.1 已列，v1.6.2 仍缺）；Cloudflare Auto Router 文档迁移（旧路径 404，新路径 /ai-gateway/features/routing/auto-router/，md5 基线重置，十三连相同纪录终结）；llm-d pushed_at 00:33:41Z（main 驱动，+2 秒）；Schematron 定价一手确认（V2 Small $0.05/$0.05/$0.23、V2 Turbo $0.03/$0.03/$0.15；两新 V2 变体）；arXiv 11-11 批次补登记 5 篇（RA 腿「11-12 批次」归属错误，执行者亲自取回列表页证伪；13454/14418 已在正本；新登记 13017 CascadeRec、13030 HIRA、13215 AutoRouter、13410 Samba、14171 CIRF）；11-12 批次枚举结转下轮；深挖在线学习阈值生产流量新证据为零（2611.05394 为 11-09 轮已登记再确认；CascadeRec/HIRA 仍为仅有的两项生产级联证据；R7 继续暂定）。
- 纠正：arXiv「11-12 批次 105 篇」（实为 11-11 批次）；2611.05394「本轮新发现」（11-09 轮已登记）；CVE-2026-114612「1.106.6 含修复待厂商确认」问法（1.106.x 从不在 GHSA 登记受影响范围内）；Railway「最终报告仍无」（04:09 轮所指为初步报告页）；Cloudflare Auto Router「md5 十三连相同」（文档已迁移，基线重置）。
- 剩余未核实缺口：arXiv 11-12 批次枚举（公告 2026-11-13 01:00 UTC，列表页约 05:00 UTC 刷新）；OpenRouter $6.43 口径调和（未结台账，第十五次重试；计数器窗口外移动；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；OpenCode post-mortem；Dynamo 文档表 v1.6.2 缺行；Jev 11 因子权重与 Cloudflare 排序信号；llm-d test-nightly 分支是否被删除；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项；decisions recipe 数盯梢；arXiv 11-12 批次枚举为未结台账新增项），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-13 13:09 轮（第 43 轮）

- 本轮增量（D-R01–D-R14，落入 skill/references/api-routing.md）：OpenRouter benchmarks API 第十六次 401（67 字节逐字节一致；计数器 2,417,024 → 2,419,560，+2,536，恒等式第四轮闭合：GPQA +1,209、VGI +878、τ² +449；$6.37 沿用值不得写成重算）；decisions 新方法仍未定（/labs/decisions 86,032 字节，numberOfItems 仍 1；两 API 端点仍 404；旧值 37 记未决）；目录 475 持平、博客 184 持平；LiteLLM 仍 v1.109.1（新预发布 v1.111.0-dev.1）、vLLM 仍 v0.33.0（rc2 仍仅 tag）；CVE-2026-114612 厂商逐字确认仍缺（#51254 消歧为另一漏洞 GHSA-2rwg-2wh4-7c4f）；Portkey CVE 修复版仍无（1.15.2，仓库零提交自 2026-05-25）；Railway 最终报告无更新；OpenCode 官方博客仍 404；Dynamo 文档表 v1.6.2 缺行仍未补；Cloudflare Auto Router md5 十四连相同；llm-d pushed_at 02:13:42Z（gh-pages 驱动，+3 秒；不得断言 test-nightly 分支被删除）；Schematron 定价未变、Extractions API 参考页仍滞后；AgentRouter 仍 v1.2.1；arXiv 11-12 批次枚举第三轮结转（任务书「01:00 UTC 公告」前提与三腿实测不符）；R7 深挖零新证据（三篇从业者指南仅作相邻证据）；11-11 批次路由论文版本复核全部仍 v1。
- 纠正：任务书前提「arXiv 11-12 批次已于 2026-11-13 01:00 UTC 公告」与三腿实测不符（截至 05:25 UTC 列表页仍为 11-11 批次）；llm-d「test-nightly 分支被删除」不得断言；#51254 不得读作 CVE-2026-114612 的修复确认。
- 剩余未核实缺口：arXiv 11-12 批次枚举（第三轮结转）；OpenRouter $6.43 口径调和（未结台账，第十六次重试；decisions 新计数方法待定）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表 v1.6.2 缺行；Jev 11 因子权重与 Cloudflare 排序信号；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；api-routing.md 已 398 行，下轮起须先拆分再增量（拆分方案用户已裁定：2026-11-13 19:08 用户选 A——按「机制表／论文／仓库／轮次复核」分篇拆分，SKILL.md 索引随之更新；第 45 轮起先拆分、拆完核对全部引用不断链后再落增量）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项；decisions recipe 数盯梢；arXiv 11-12 批次枚举为未结台账项），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。
- 下轮起始 HEAD：llm-d main 565373b7、gh-pages c8da61c3；其余见 research/20261113-1309/rb-report.md。

## 2026-11-13 16:09 轮（第 44 轮）

- 本轮增量（D-R01–D-R12，原位替换 api-routing.md 13:09 轮复核行，净增长 0 行）：OpenRouter benchmarks API 第 17 次 401（67 字节，md5 与上十六轮相同；计数器 2,419,560 → 2,421,232，+1,672，恒等式第五轮闭合：GPQA +594、VGI +878、τ² +200；$6.37 沿用值不得写成重算）；decisions 新方法仍未定（/labs/decisions 85,314 字节，numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 475 持平、博客 184 持平；LiteLLM 仍 v1.109.1（v1.111.0-dev.1 窗口前）、vLLM 仍 v0.33.0；CVE-2026-114612 厂商逐字确认仍缺；Portkey CVE 修复版仍无；Dynamo 文档表缺行仍未补；Cloudflare Auto Router 文档由 index.md（已 404）迁为 auto-router.mdx（内容实质等价，md5 基线因格式迁移重置；执行者自验归一化比对相似度 0.9376）；Jev 权重首次源码一手值（planning 0.15/0.05、code 0.45/0.20、writing 0.80/0.50、other 0.50/0.25）；11 因子口径纠正（属 OpenRouter 新一代 Auto Router）；llm-d pushed_at 06:50:56Z（gh-pages 徽章推送）；Schematron 定价未变；AgentRouter 仍 v1.2.1；arXiv 11-12 批次枚举完成（路由 136 篇中 8 篇相关）。
- 纠正：Railway 最终报告「新发布」不成立（42 轮已入正本，本轮仅为再确认）；AgentRouter「仓名纠正」不成立（theagentrouter/agent-router 自 01:09 轮起即为取证所用仓名）；Jev「11 因子清单未找到」找错了地方（11 因子属 OpenRouter Auto Router）。
- 剩余未核实缺口：OpenRouter $6.43 口径调和（未结台账，第十七次重试；decisions 新计数方法待定）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；OpenRouter 11 因子权重；在线学习阈值真实流量一手证据（R7 暂定继续）；veduis 生产数字（D-R02 暂定继续）。
- 下轮起点：先清上述缺口（OpenRouter $6.43 重算为未结台账首项；decisions recipe 数盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。
- 下轮起始 HEAD：llm-d pushed_at 06:50:56Z（gh-pages 驱动）；其余见 research/20261113-1609/rb-report.md。

## 2026-11-13 19:09 轮（第 45 轮）

- 本轮增量（D-R01–D-R10，落入 skill/references/api-routing-rounds.md + api-routing-papers.md；本轮先行完成 api-routing.md 拆分：74 行机制核心 + papers 36 行 + repos 21 行 + rounds 286 行，SKILL.md 索引已更新，活引用已改道，旧版快照存于 `versions/v0.44.0/` 与 `versions/v0.44.0-split/`）：`/api/v1/benchmarks` 仍 401，第 18 次连续失败；**重算路线恢复**——前端统计端点 HTTP 200（302 runs），执行者亲自重算 Auto Code $1.6427 / Opus 5.5 $6.3675，与既往五轮逐字一致；板面 $6.43 与重算 $6.3675 的 $0.0625 缺口仍未闭合；未结台账首项划除重算部分；计数器 2,421,232 → 2,363,118（−58,114），恒等式第六轮闭合（GPQA −44,557、τ² −13,557）；decisions 新方法仍未定（numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 475 → 472（移除 3 项滞后上架）、博客 184 持平；LiteLLM 仍 v1.109.1、vLLM 仍 v0.33.0；CVE-2026-114612 厂商逐字确认仍缺；Portkey 仍 v1.15.2；Dynamo 文档表缺行仍未补（澄清 10:09 轮「v1.6.1 已列」指版本切换器，非表行）；Cloudflare .mdx md5 相同；Jev 文档重组（旧址 404，新址 Updated 2026-11-11T22:08:37Z，权重仍未披露）；Extractions 指南页已迁新址并列两 Schematron 模型，滞后缺口闭合；**R2-Router 新登记**（arXiv:2602.02823v2，成本可控化新分支，4–5× 论文自报）；**GAMCAL/SUPG-IT 为迄今最接近 R7 的相邻证据**（不得写成闭合）；2611.12953/2611.13412 全文再核通过；arXiv 11-13 批次尚未公告，结转下轮。
- 纠正：台账「无替代数据源」→ 前端端点从未 401；RB 腿「第十八次重试首次成功」→ 重算任务成功，非 `/api/v1/benchmarks`；RA 腿「$6.37 本轮未重算」→ 执行者亲自重算 $1.6427/$6.3675；Dynamo「v1.6.1 已列」→ 指版本切换器，非表行。
- 剩余未核实缺口：decisions 新计数方法待定；arXiv 11-13 批次枚举（预期 2026-11-14 01:00 UTC 公告后）；`/api/v1/benchmarks` 401 观察（不再阻塞重算）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；OpenRouter 11 因子权重；在线学习阈值真实流量一手证据（R7 暂定继续）；veduis 生产数字（D-R02 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举；decisions recipe 数盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。
- 下轮起始 HEAD：llm-d main 565373b7；其余见 research/20261113-1909/ra-report.md。

## 2026-11-13 22:09 轮（第 46 轮）

- 本轮增量（D-R01–D-R07，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：`/api/v1/benchmarks` 仍 401，第 19 次连续失败；**$6.37 本轮亲自重算完成**（302 runs，Auto Code $1.6427 精确、Opus 5.5 $6.3675；板面 $6.43 缺口 $0.0625 未闭合，双值并列引用）；计数器 2,363,118 → 2,347,015（−16,103），恒等式第七轮闭合（GPQA −16,153、τ² +50）；decisions 新方法仍未定（numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 472 → 470（移除 `liquid/lfm-2.5-2.6b` 及其 `:free` 变体）、博客 184 持平；LiteLLM 仍 v1.109.1、vLLM 仍 v0.33.0；CVE-2026-114612 厂商逐字确认仍缺；Portkey 仍 v1.15.2；Dynamo 文档表缺行仍未补；Cloudflare .mdx md5 相同；Jev 权重仍未披露；AgentRouter 仍 v1.2.1；llm-d 14:11:08Z 为窗口边缘徽章推送，不计入窗口内；**11-12 批次重筛补登记 3 篇**（TraceKV 2611.15569、OSCAR 2605.17757v2、DForkTree 2611.12356v2，摘要级）；**2611.16414 全文深读**（FP/FN→成本映射定量模板：14B 路由器下游 token −68.7%、延迟 −80.5%，均不含路由开销；路由与下游延迟分环境测量不可加；非 R7 证据）；9 篇版本复核全部无新版；RouteLLM 仍 `0b64fda`（2024-08-10，休眠 2 年 3 个月）。
- 纠正：RB 腿「11-13 批次枚举闭合」→ 未闭合、第二次结转（周五列表页三方 md5 逐字节相同，即 11-12 提交批次，16:09 轮已枚举；RB 腿 §1.1 自承批次对应 11-12 提交，与其结论自相矛盾）。
- 剩余未核实缺口：decisions 新计数方法待定；arXiv 11-13 批次枚举（第二次结转，预期 2026-11-14 01:00 UTC 公告后；SB 腿所称 00:00 UTC 与 H 腿/台账的 01:00 UTC 有出入，记为未决小口径，下轮以实测为准）；`/api/v1/benchmarks` 401 观察（第 19 次，不再阻塞重算）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；OpenRouter 11 因子权重；在线学习阈值真实流量一手证据（R7 暂定继续；2611.16414 为 FP/FN→成本映射模板、非 R7 证据）；veduis 生产数字（D-R02 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举；decisions recipe 数盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。
- 下轮起始 HEAD：llm-d main 565373b7（未变，本轮仅窗口边缘徽章推送）；其余见 research/20261113-2209/ra-report.md。

## 2026-11-14 01:09 轮（第 47 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：`/api/v1/benchmarks` 仍 401，第 20 次连续失败；**$6.43 缺口本轮闭合**（板页现为完整服务端渲染 828,195 字节，直接读得 Opus 5.5 (high) $6.43、Auto Code $1.64、Auto (Medium) $0.58；swe_atlas_qa 高档 $0.79 为前端统计接口内无源的高档运行，代入后 $6.429881，精确解释缺口；**双值并列引用规则更新**：板面 $6.43（厂商自报，现有一手页面证据）与项目重算 $6.37 并列；新缺口：高档运行源数据不在公开统计接口内）；计数器 2,347,015 → 2,289,984（−57,031），恒等式第八轮闭合（GPQA −45,906、τ² −11,125），连续第三轮下修，累计 −131,247，原因未知；decisions 新方法仍未定（numberOfItems 仍 1；两 API 仍 404；`/decisions` 改为 40,753 字节 Next.js 错误壳，模板变化非内容进展；旧值 37 记未决）；目录 470 持平，`qwen/qwen3.5-9b` 真涨价 +55%（prompt $0.067/M → $0.104/M），minimax/hy4/hy3 顶层价动为时段 override 轮换非真调价，`z-ai/glm-5.3-flashx` 加 `structured_outputs`；博客 184 持平；LiteLLM 仍 v1.109.1、vLLM 仍 v0.33.0；CVE-2026-114612 厂商逐字确认仍缺（v1.106.6 正文 1,436 字节内 0 命中）；Portkey CVE-2026-110719 advisory 的 `vulnerabilities` 数组为空、无 `first_patched_version`；Dynamo 文档 Support Matrix v1.6.1/v1.6.2 行仍缺；Cloudflare .mdx md5 相同；OpenRouter auto-router 文档可见文本相同（204,650 字符）；Jev 11 因子权重仍未披露；arXiv 11-13 批次仍未公告，**第三次结转**；**2611.16414 全文复核通过**（7 项结论逐条复核无误，窗口标题检索零新论文；R7 未变，生产级联证据仍仅 CascadeRec + HIRA）。
- 纠正：$6.43 缺口「未闭合/待解释」→ 已解释（板面完整服务端渲染 + swe_atlas_qa 高档 $0.79 代入精确复算），双值并列引用规则更新；Mem0 相关纠正见记忆轨（本轨不重复）。
- 剩余未核实缺口：decisions 新计数方法待定；arXiv 11-13 批次枚举（第三次结转，2026-11-14 01:00 UTC 公告后补做；00:00 UTC vs 01:00 UTC 出入仍未决，下轮以实测为准）；`/api/v1/benchmarks` 401 观察（第 20 次，不再阻塞重算）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；OpenRouter 11 因子权重；在线学习阈值真实流量一手证据（R7 暂定继续）；veduis 生产数字（D-R02 暂定继续）；高档运行源数据不在公开统计接口内（新缺口，待上游同步或披露）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举；decisions recipe 数盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。
- 下轮起始 HEAD：llm-d main 565373b7（未变，本轮仅窗口边缘徽章推送）；其余见 research/20261114-0109/ra-report.md。

## 2026-11-14 04:09 轮（第 48 轮）

- 本轮增量（D-R01–D-R10，落入 skill/references/api-routing-rounds.md、api-routing-papers.md、api-routing.md）：OpenRouter benchmarks API 第二十一次 401（67 字节逐字节一致；计数器 2,289,984 → 2,281,843，−8,141，**恒等式第九轮闭合**，GPQA 单项 −8,141、runCount −42，连续第四轮下调，累计 −139,389）；重算三档与上轮逐数一致（302 runs，+0；$1.6427/$6.3675/$0.5757；板面值仍在页；双值并列规则不变；高档源数据缺口未变）；decisions 新方法仍未定（/labs/decisions 83,853 字节，可见文本逐字同，numberOfItems 仍 1；两 API 端点仍 404；旧值 37 记未决）；目录 470 → 471（窗口内新增 `inclusionai/ling-3.2-flash:free`，created 2026-11-13T17:45:58Z），qwen 二度涨价 +47%（两轮累计 +128%），另 5 项原地更新；博客 184 篇不变；LiteLLM 仍 v1.109.1（窗口内新预发布 v1.111.0-dev.2）、vLLM 仍 v0.33.0、Portkey 仍 v1.15.2、AgentRouter 仍 v1.2.1、Dynamo 仍 v1.6.2（文档表缺行未补）；CVE-2026-114612 厂商逐字确认仍缺；Portkey CVE 修复版仍无；Cloudflare .mdx 逐字节相同；Jev 权重仍未披露；llm-d pushed_at 本窗口未动；**24 篇路由论文版本复核全部无新版**（R7 仍无证据）；arXiv 11-13 批次仍未公告，**第四次结转**（列表页 md5 四方一致）。
- 纠正：**L254 五篇论文标签误标**（2611.13017/13030/13215/13410/14171 被标为 CascadeRec/HIRA/AutoRouter/Samba/CIRF，全部不成立；实为 Multi-Future Drafting / APS-VLA / Counting Under Disagreement / SteerDistill / BARS；真 CascadeRec 2611.08566、HIRA 2608.21792）；**OpenRouter Traffic Routing 文档改写**（dateModified 2026-11-12T22:22:30Z，轮次间变化：公式名/公式/1.25 例证/Cap 护栏均已不在现行页；博客原文仍含公式与两护栏，Cap 在博客全文 0 命中；api-routing.md 相关表述已加版本限定，"三护栏"改为博客两护栏 + 旧版曾有第三项的注记）；**46 轮「11-13 批次枚举闭合」撤回**（同一列表页的批次归属误读）；任务书括号称 RouteLLM「最后提交 88b8dec」有误（第三条提交，最新 0b64fda）——技能正本已于 11-11 轮纠正，无须再改；46 轮标题提取 1 处误入（2609.07662 仅引文）+ 2 处遗漏（2606.22673、2609.34924），本轮 537 篇提取与页头一致，筛查结论不受影响。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第四次结转，2026-11-14 01:00 UTC 公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；计数器连续下调成因不明（四轮累计 −139,389）；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter $6.43 第二十二次重试），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-14 10:09 轮（第 49 轮）

- 本轮增量（D-R01–D-R10，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十二次 401（67 字节逐字节一致，执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合）；计数器 2,281,843 → 2,282,559（+518），**恒等式第十轮闭合**（执行者亲自提取复算 14 卡求和 2,282,559；GPQA +368、τ² +150，lastRun 均为窗口内；此前连续四轮下修累计 −139,389 本轮转为上行）；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录仍 471、博客仍 184；**博客 sitemap 旧址 `/blog/sitemap.xml` 已 404，迁至 `/sitemaps/blog.xml`**（内容逐字节相同，下轮改用新址）；定价：qwen3.5-9b −54.7%（$1.7e-7 → $7.7e-8，执行者复核 $8.1e-8，分钟级波动，疑似动态定价未证实）、deepseek-v4.1-flash −83.1%（$5.07e-8，执行者复核一致）、gpt-5.6-sol −9.1%、glm-5.2:batch −7.5%、minimax/hy4/hy3 表观翻转为 UTC 时段 overrides；LiteLLM LTS 双补丁 v1.108.8（2026-11-13T20:21:21Z）与 v1.106.10（2026-11-14T00:34:35Z），最新稳定仍 v1.109.1；vLLM 仍 v0.33.0、v0.34.0rc2 仍仅 tag（方法纠正：`/releases/tag/<tag>` 页标题具误导性，须以 releases 列表页为准）；Portkey 仍 v1.15.2，**1.15.1 与 1.15.2 均有 Release 对象，缺口闭合**；Dynamo 仍 v1.6.2，文档表仍缺 v1.6.1/v1.6.2 行（三方版本区分：切换器 v1.6.1 ≠ 表声明稳定 v1.6.0 ≠ 仓库最新稳定 v1.6.2）；AgentRouter 仍 v1.2.1；CVE 无变化（GHSA JSON 逐字节相同；厂商逐字确认 1.106.6 含修复仍缺；Portkey CVE-2026-110719 vulnerabilities 数组仍空）；OpenRouter 四文档可见文本零变化（md5 变仅 Mintlify/Astro 资源哈希轮换；方法注记：此后须先做可见文本 diff 再定性）；llm-d pushed_at 仍 2026-11-13T22:26:11Z；**arXiv 公告时刻表一手闭合**（官方 Availability 页：周四 14:00–周五 14:00 ET → 周日 20:00 ET；周五 14:00–周一 14:00 ET → 周一 20:00 ET = UTC 周二 01:00；历轮「01:00 UTC 公告」前提确认正确，00:00 vs 01:00 出入解决为 01:00 UTC）；11-13 批次仍未公告，**第六次结转**（计 07:09 腿在内），分两档公告（UTC 11-16 01:00 与 11-17 01:00）；**新登记 2611.16478**（One Model, Two Precisions，升级路由，直接相关）+ 相邻 3 篇（2611.15515/2611.16445/2611.16937），登记集 24 → 25 篇。
- 纠正：48 轮「qwen3.5-9b 二度涨价 +47%」→ 本轮 −54.7% 逆转（价格分钟级波动，疑似动态定价未证实，引用须绑取回时间戳）；RA/H 两腿「11-13 批次已公告/枚举闭合」→ 归属错误，不采纳（当前可见列表页仍为 11-12 提交批次，113 篇新投稿，published 均为 2026-11-12）；M 腿「11-17 00:00 UTC 公告」→ 官方时刻表一手闭合为 11-17 01:00 UTC；RB 腿「11-16 05:00 UTC」→ 不成立、驳回；v0.34.0rc2「有 Release 对象」→ 仅 tag，`/releases/tag/<tag>` 页标题具误导性。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第六次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；计数器下修转上行成因不明；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；博客 sitemap 改用新址；decisions recipe 数盯梢；OpenRouter $6.43 第二十三次重试），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-14 13:09 轮（第 50 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十三次 401（67 字节逐字节一致；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合）；计数器 2,282,559 → 2,283,303（+744），**恒等式第十一轮闭合**（执行者亲自提取复算 14 卡求和 2,283,303；GPQA +594、τ² +150，lastRun 均为窗口内）；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录仍 471、博客仍 184；定价（**窗口关闭后取回 05:12–05:14 UTC，不得记为窗口内调价**）：qwen3.5-9b $1.36e-7（+63.9% vs 02:22、+76.6% vs 02:14；一日内三度大幅摆动，动态定价假说加强、未证实）、glm-5.2:batch 精确回撤至降价前 $3.22e-7、deepseek-v4.1-flash −83.1% 持续（$5.07e-8，五次取回稳定）、gpt-5.6-sol −9.1% 持续；五项目零新稳定版；vLLM 新现 tag v0.34.0rc3（无 Release 对象）；CVE 无变化（两份 GHSA JSON 逐字节相同；厂商逐字确认仍缺；Portkey CVE-2026-110719 仍无修复版）；Cloudflare .mdx md5 相同，OpenRouter 四文档可见文本零变化；llm-d pushed_at 窗口内移至 02:15:44Z，仍为 gh-pages 徽章推送模式；**新登记 9 篇路由论文**（直接 2：2611.15911 Smart Model Routing、2609.32917 Planner-as-Router；相邻 7：2611.16499/2611.15481/2611.16661/2605.18796 UCCI/2611.13176/2506.11887/2603.25450），登记集 25 → 34 篇；arXiv 11-13 批次仍未公告，**第七次结转**（公告档 UTC 11-16 01:00 与 11-17 01:00）；R7：**UCCI 为最接近的新相邻证据**（生产企业照片管理 NER 工作负载，成本 −31% [95% CI 27%, 35%]、ECE 0.12 → 0.03；全文明确在线重校准为未来工作），仍不闭合，继续暂定。
- 纠正：**「v0.33.0 tag 有 Release 对象」→ Release 对象实挂 v0.33.0rc3 tag**（name "v0.33.0"，published 2026-11-02T10:10:05Z；v0.33.0 tag 本身 404；执行者亲自直查两项 tag），今后以 releases/tags/<tag> API 或列表页锚点为准，/releases/tag/<tag> 页标题具误导性；**「Jev 上游仓库路径 OpenRouterTeam/jev-router」→ 404 错误路径，官方上游未确认**（typesafe-ai 组织 12 公开仓无 Jev 仓，全站搜索无官方仓；此前 403 限流体掩盖了 404），philippdubach/pi-jev-router 为个人仓、不得作官方证据，新立未决缺口；qwen 涨幅两腿差异（+63.9% vs +76.6%）仅为基线不同，绝对值一致 $1.36e-7（RA/RB 两份 Models API 文件逐字节相同）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第七次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；Jev 官方上游仓库未确认（新立未决缺口）；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter $6.43 第二十四次重试；Jev 上游仓库再核查），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-14 16:09 轮（第 51 轮）

- 本轮增量（D-R01–D-R14，落入 skill/references/api-routing-rounds.md + api-routing-repos.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十四次 401（67 字节，md5 2df0deaf…，逐字节相同；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合）；计数器 2,283,303，恒等式第十二轮精确闭合（14 卡全部零变动，连续多轮首次全卡零变动）；定价（窗口关闭后取回 08:11 UTC，变动在 05:12–08:11 UTC 之间，不得记为窗口内调价）：qwen3.5-9b $1.69e-7（+24.3%，一日内第四度大幅摆动）、glm-5.2:batch 回落至 $2.98e-7（−7.4%，回到回弹前原价）、deepseek/gpt-5.6-sol 零变动；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；五项目零新稳定版（vLLM v0.34.0rc3 仍仅 tag，无 rc4）；CVE 无变化；llm-d pushed_at 窗口内移至 05:35:04Z，仍为 gh-pages 徽章推送模式；**Jev 官方上游仓库缺口闭合为「无公开源码仓」定性结论**（五渠道一手核查：组织仍 12 公开仓，无 Jev 仓；X 侧仅登录墙外壳为局限注记），13:09 轮新立未决缺口划除；arXiv 11-13 批次仍未公告，第八次结转（公告档 UTC 11-16 01:00 与 11-17 01:00）；**路由论文新登记 3 篇**（ARoute 直接、CascadeFilter 与 Beyond K/V Caches 相邻；VATC 不补入），登记集 34 → 37 篇；2611.15911 全文深读（级联盈亏平衡 a₁ > c_A/c_B；全文 "online" 0 命中，非 R7 证据）；UCCI 全文深读（静态校准阈值，Theorem 1；"online" 恰 1 命中即 Limitations 句，R7 不闭合）；KV 切换罚金从业者算例（41 美分 / 19 回合，算术已验）+ aicassindra 相邻指南；CASCADE（2605.06702v1）相邻候选登记；34 篇登记论文版本复核全部无新版，9 篇新登记论文版本首次取回作基线。
- 纠正：无（本轮路由轨无推翻旧说法；RB 腿 arXiv 查询首轮 13 次失败为本机 URL 编码错误、非上游限流，此前定性撤回；RA 腿两个 404 初取为猜测 slug 错误已纠正）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第八次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter $6.43 第二十五次重试），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-14 19:09 轮（第 52 轮）

- 本轮增量（D-R01–D-R12，落入 skill/references/api-routing-rounds.md + api-routing-repos.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十五次 401（67 字节，md5 2df0deaf…，逐字节相同；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合）；计数器 2,283,303 → 2,283,353（+50），恒等式第十三轮精确闭合（唯一变动 τ²-Bench Airline 卡，窗口内新 run，topEntry deepseek-v4-flash-20260423）；定价（窗口关闭后 11:10–11:14 UTC 取回，归属不确定，不得记为窗口内调价）：qwen3.5-9b $1.7e-7（+0.6%）、glm-5.2:batch 回到 $3.22e-7（+8.1%，精确回弹至回弹前原价）、deepseek/gpt-5.6-sol 零变动；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；五项目零新稳定版（vLLM 新 tag v0.34.0rc4，轻量 tag，无 Release 对象，不得记为窗口内发布）；CVE 无变化；llm-d pushed_at 本轮未动；arXiv 11-13 批次仍未公告，第九次结转（公告档 UTC 11-16 01:00 与 11-17 01:00）；**路由论文新登记 4 篇相邻候选**（BaRP 2510.07429、vLLM-SR 2603.21354v2、Parallel Prompting 2611.15130、Evolving Infra 2611.12304v2；直接相关新登记 0 篇；CASCADE 仍不入登记集），登记集 37 → 41 篇；R7 仍不闭合（BaRP 为最接近的新相邻证据，模拟在线 + 离线实验；UCCI 引用跟进 S2 8 / OpenAlex 6；2611.15911 引用跟进未完成）；basal-serve 支持 `/v1/decisions`（2026-11-02，厂商自报，首个厂商侧 Decisions API 落地）；37 篇登记论文版本复核全部无新版。
- 纠正：**RA 腿「两处文档迁移为本轮新发现」不成立**（OpenRouter 现址已于 2026-11-13 19:09 轮登记，16:09 轮证据文件 og:url 即新址；Cloudflare 迁移已于 2026-11-13 10:09 轮登记；RA 腿「上轮 08:13 UTC 旧址仍 200（13,744 字节）」为 URL 混淆，13,744 B 文件实为新址 raw GitHub 取回；RB 腿旧址 200 读数判为过期边缘缓存；两旧短址现确认 404，新址内容逐字相同；技能正本无需改动）；51 轮 skill-reach pushed_at 笔误纠正（以证据文件 2026-11-14T06:31:53Z 为准，见 track-skills.md）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第九次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter $6.43 第二十六次重试），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。


## 2026-11-14 22:09 轮（第 53 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十六次 401（67 字节，md5 2df0deaf…，逐字节相同；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合）；计数器 2,283,353，恒等式第十四轮精确闭合（14 卡全部零变动，连续两轮全卡零变动）；定价（窗口关闭后 14:12–14:18 UTC 取回，归属不确定，不得记为窗口内调价）：qwen3.5-9b −22.9%（$1.7e-7 → $1.31e-7，三档同幅下调）、glm-5.2:batch 维持 $3.22e-7、deepseek-v4.1-flash $5.07e-8、gpt-5.6-sol $4e-6 均零变动；178 模型仅 benchmarks 字段 churn；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；五项目零新稳定版（vLLM 无 v0.34.0rc5 tag）；CVE 无变化；**llm-d pushed_at 窗口内移至 12:15:09Z，成因未明（新缺口）**；arXiv 11-13 批次仍未公告，第十次结转（公告档 UTC 11-16 01:00 与 11-17 01:00）；**BaRP 全文深读**（§6 自承离线静态日志训练、无阈值概念，R7 仍不闭合）与 **SHADOWPREFILL 全文深读**（切换罚金生产 trace 级定量 80–89%→44–65%、1.5–3.2×；为 2611.15911 盈亏平衡缺失的缓存状态项；无期望成本公式、无阈值在线学习，不是 R7 证据）；41 篇登记论文版本复核全部无新版（引用跟进 429 失败，不得记为零引用）；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 纠正：**llm-d「pushed_at 移动必属 gh-pages 徽章推送模式」口径不再成立**（本轮移动无对应 HEAD 提交；公开事件流本身不可靠，同样缺失已知的 02:15 与 05:35 两次推送）；**GHSA「字节数变化 = 内容更新」不成立**（pretty-print 与 compact 排版方差；今后一律先做规范化 JSON diff 再定性）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；llm-d pushed_at 移动成因（新缺口）；2611.15911 / BaRP / UCCI 引用跟进（429 失败，下轮重试）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter $6.43 第二十七次重试；llm-d pushed_at 与各分支 HEAD 对应关系），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准）。

## 2026-11-15 01:09 轮（第 54 轮）

- 本轮增量（D-R01–D-R09，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十七次 401（67 字节，md5 2df0deaf…，逐字节相同；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合）；计数器 2,283,353，恒等式第十五轮精确闭合（14 卡全部零变动；首次记录逐卡值供下轮逐卡比对）；定价（窗口关闭后取回，归属不确定，不得记为窗口内调价）：**qwen3.5-9b prompt −32.1%**（$1.31e-7 → $8.9e-8，执行者自验再取已回弹至 $1.09e-7，动态定价假说加强）、glm-5.2:batch $3.22e-7、deepseek-v4.1-flash $5.07e-8、gpt-5.6-sol $4e-6 均零变动；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；五项目零新稳定版（vLLM 无 v0.34.0rc5 tag）；CVE 无变化（GHSA 规范化 JSON diff 为空；soupsieve 升级 PR 号纠正为 #51585）；**llm-d pushed_at 本轮未移动**（仍 2026-11-14T12:15:09Z，成因仍未明）；arXiv 11-13 批次仍未公告，第十一次结转（公告档 UTC 11-16 01:00 与 11-17 01:00）；41 篇登记论文版本复核全部无新版（引用跟进连续第二轮 429 失败，不得记为零引用）；七项筛查查询零新论文（R7 仍不闭合）；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 纠正：**RA 腿「四模型价格全未变」不成立**（腿误把基线 prompt 价 $1.31e-7 当完成价；53 轮证据文件显示其为 prompt 价，54 轮 prompt 价为 $8.9e-8）；**soupsieve 升级 PR「#51593」→ #51585**（53 轮笔误，执行者核对 release 正文纠正）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十一次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；llm-d pushed_at 移动成因（仍未明）；2611.15911 / BaRP / UCCI 引用跟进（连续第二轮 429 失败，下轮 UTC 午夜后重试，或申请免费 API key，待主会话裁定）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter $6.43 第二十八次重试；llm-d pushed_at 与各分支 HEAD 对应关系；引用跟进 UTC 午夜后重试），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 04:09 轮（第 55 轮）

- 本轮增量（D-R01–D-R11，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十八次 401（67 字节，md5 2df0deaf…，逐字节相同；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合；执行者亲自重算复现）；计数器 2,283,353，恒等式第十六轮精确闭合（14 卡全部零变动，连续三轮全卡零变动；执行者亲自重算复现）；定价（窗口关闭后取回，归属不确定，不得记为窗口内调价）：**qwen3.5-9b prompt +44.9%（$8.9e-8 → $1.29e-7，completion +44.3%）、glm-5.2:batch prompt +7.8%（$3.22e-7 → $3.47e-7，completion +8.0%）**，全目录仅此 2 个调价（执行者亲自比对两轮 Models API 文件逐项确认；qwen 一日内四度大幅摆动，动态定价假说加强、未证实）；decisions 仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录仍 471、博客仍 184；五项目零新稳定版（vLLM 无 v0.34.0rc5 tag，执行者亲自查 tags API 确认）；CVE 无变化（GHSA 规范化 JSON diff 为空；厂商逐字确认仍缺；Portkey CVE vulnerabilities 仍 []）；**llm-d updated_at 移动原因定位**（stargazers 5258 → 5259，WatchEvent 18:37:36Z 与 updated_at 18:37:37Z 差 1 秒，执行者亲自查事件流确认；方法口径升级：今后 updated_at 移动不得读作推送信号）；llm-d pushed_at 本轮未动（仍 12:15:09Z，成因仍未明）；文档可见文本零变化；41 篇登记论文全部无新版；**引用跟进部分突破**（S2 首次取回 BaRP 23、2611.15911 0，实测值；UCCI 的 S2 + OpenAlex 与全部 OpenAlex 计数连续第三轮 429，不得记为零引用）；R7 筛查零新文，深挖 KV 感知期望值模型与生产级联校准（全部关键数字口径不变，仍为论文自报；无新版本或后续工作推进 R7）；arXiv 11-13 批次仍未公告，**第十二次结转**（列表页 md5 十轮同一值；公告档 UTC 11-16 01:00 与 11-17 01:00）。
- 纠正：**RA 腿 md5「九轮同一值」→ 十轮同一值**（54 轮未结台账记为九轮，本轮为第十轮）；**llm-d「updated_at 移动仅为观察、无因果」→ 升级为定位结论**（updated_at 跟随仓库元数据活动，与推送无关）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十二次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；llm-d pushed_at 12:15:09Z 移动成因（仍未明）；引用跟进余项（UCCI 的 S2 + OpenAlex、BaRP 与 2611.15911 的 OpenAlex；UTC 午夜重置后重取，或申请免费 API key，待主会话裁定）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter $6.43 第二十九次重试；llm-d pushed_at 与各分支 HEAD 对应关系；引用跟进 UTC 午夜重置后重试），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 07:09 轮（第 56 轮）

- 本轮增量（D-R01–D-R11，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第二十九次 401（67 字节，md5 2df0deaf…，与此前 28 次逐字节相同；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合；执行者亲自重算复现）；计数器 2,286,352，恒等式第十七轮精确闭合（+2,999；GPQA +1,332、VGI +1,317、τ² +350，其余 11 卡零变动；执行者亲自重算复现）；定价（窗口关闭后 23:15:28Z 取回，归属不确定，不得记为窗口内调价）：**qwen3.5-9b prompt −41.9%（$1.29e-7 → $7.5e-8）、glm-5.2:batch prompt −14.1%（$3.47e-7 → $2.98e-7）**，全目录仅此 2 个调价（执行者亲自比对两轮 Models API 文件逐项确认；qwen 一日内第五度大幅摆动，动态定价假说加强、未证实；执行者轮次外现取 qwen 已再移至 $3.4e-8，第六度摆动）；decisions 仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录仍 471、博客仍 184；**窗口内新稳定发布 1 件：LiteLLM v1.108.9 LTS 补丁**（GitHub 22:00:02Z，PyPI 21:54:43–21:55:10Z，9 个 backport PR；主线仍 v1.109.1，无 v1.110.0；执行者亲自查 GitHub Releases API 复核）；其余四项目零新稳定版（vLLM 无 v0.34.0rc5）；CVE 无变化（GHSA 规范化 JSON diff 为空；厂商逐字确认仍缺；Portkey CVE vulnerabilities 仍 []）；**llm-d updated_at 星事件驱动获第二次阳性对照**（移至 23:10:54Z，窗口关闭后 1 分 54 秒，与最新 WatchEvent 0 秒差；stars +3、forks +1）；llm-d pushed_at 本轮未动（仍 12:15:09Z，成因仍未明）；文档可见文本零变化；41 篇登记论文全部无新版；**引用跟进重大突破**（OpenAlex 午夜重置假说获实证：午夜后 6 次全部取回——UCCI OpenAlex 7（上轮 6 → +1）、BaRP OpenAlex 21（首次取回）、2611.15911 OpenAlex 0（实测值，与 S2 的 0 双源一致）；UCCI S2 8 复核无变化，**UCCI 两项全部闭合**；BaRP 与 2611.15911 的 S2 计数仍未取回，两次 429，55 轮值 23 / 0 未能复核，不得记为变化）；R7 筛查零新文，深挖 KV 感知期望值模型与生产级联校准（全部关键数字口径不变，仍为论文自报；无新版本或后续工作推进 R7）；arXiv 11-13 批次仍未公告，**第十三次结转**（列表页 md5 十一轮同一值；公告档 UTC 11-16 01:00 与 11-17 01:00）。
- 纠正：**55 轮未结台账 md5「十轮同一值」→ 十一轮同一值**（55 轮为第十轮，本轮为第十一轮；RB 腿核实，执行者复核确认）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十三次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定，盯 recipe 数增长或等官方列表端点；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；llm-d pushed_at 12:15:09Z 移动成因（仍未明）；引用跟进余项（BaRP 与 2611.15911 的 S2 计数复核；UTC 午夜后重试）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项，按两档公告时间核查；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十次重试；llm-d pushed_at 与各分支 HEAD 对应关系；引用跟进 UTC 午夜重置后重试 BaRP/2611.15911 的 S2 计数；qwen 轮次外再移至 $3.4e-8，下轮核对），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 10:09 轮（第 57 轮）

- 窗口：2026-11-14 23:09 → 2026-11-15 02:09 UTC（正确口径；派发任务书误写为 02:09–05:09 UTC，综合时已按事件时间戳逐项重归属；02:09–05:09 UTC 为第 58 轮窗口）。
- benchmarks API 第 30 次 401（67 B，md5 与前 29 次逐字节相同）；前端统计端点 216,554 B 逐字节相同（302 runs）；三档重算分毫不差（Auto Code $1.6427、Opus 5.5 $6.3675、Auto $0.5757；板面 $6.43，$0.0625 缺口未闭合；执行者亲自重算复现）；计数器 2,288,503（+2,151），恒等式第十八轮闭合（GPQA +1,467、VGI +439、τ² +245）。
- 定价（窗口关闭后取回，归属不确定）：目录 471 → 470（z-ai/glm-5.2:batch 移除）；gpt-5.6-sol 首次调价 +10%（$4e-6 → $4.4e-6）；qwen3.5-9b 12 分钟内三价（$4.2e-8 → $3.3e-8 → $3.6e-8），第七/八/九度摆动；glm-4.7 −33.3%；deepseek-v4-flash-0731 涨跌互现；3 个时段 override 轮换、158 个模型仅 benchmarks 字段 churn。动态定价假说获最强支持、仍未证实。
- LiteLLM 两枚窗口内预发布（v1.110.0-rc.3 01:10:14Z、v1.107.0-rc.4 23:46:31Z；执行者亲自复核）；其余四项目零新稳定版；CVE 无变化。
- **llm-d pushed_at 移动成因定位，53 轮起未明缺口闭合**：12:15:09Z 的 github-actions[bot] DeleteEvent（删 test-nightly/pr-3041-6837910163）；pushed_at = 任意 ref 最近活动时刻（含分支删除），不是推送指示器。
- 文档可见文本 0 行变化；41 篇登记论文无新版；R7 零新文，继续暂定。
- 引用跟进：BaRP 两项全部闭合（S2 23 / OA 21 复核无变化）；2611.15911 OA 0 闭合；2611.15911 S2 仍待下轮（两腿均 429）；UCCI OA 7 复核无变化。教训：OpenAlex 共享 IP 日预算午夜重置后约 2 小时即耗尽，下轮贴近 00:00Z 重试。
- 纠正：派发窗口口径错误（见上）；RA 腿「两枚 RC 不计入本窗口」驳回；arXiv 11-13 批次第十四次结转（列表页 md5 第十二轮同一值）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十四次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（2611.15911 的 S2 计数、SkillScope OA 12；UTC 午夜后贴近 00:00Z 重试）；qwen 动态定价再观察（下轮至少两次取回比对）；glm-5.2:batch 下架原因。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十一次重试；2611.15911 的 S2 计数与 SkillScope OA 12 贴近 UTC 午夜重试；qwen 至少两次取回比对），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 13:09 轮（第 58 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十一次 401（67 字节逐字节一致，执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者亲自重算复现）；计数器 2,288,503 → 2,297,865（+9,362），**恒等式第十九轮闭合**（执行者亲自现取 benchmarks 页复核板面总值与 GPQA 原值；GPQA +9,362、其余 13 卡零变动；取回在关窗后，归属不确定）；decisions 新方法仍未定（/labs/decisions 83,853 字节、numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 470 → 470、博客仍 184；定价（关窗后取回）：**qwen3.5-9b 第十度摆动至 $1.69e-7**（执行者现取 $1.7e-7 量级一致）、**kimi-k3.1-pro prompt −88.1%**，另 3 项调价，全目录真调价 5 个，`benchmarks` 字段 churn 0；**LiteLLM v1.111.0-dev.4 窗口内预发布**（04:17:20Z，138 PRs；dev.3 05:09:18Z 在关窗后 18 秒，边界项不计入）；其余四项目零新稳定版；CVE 无变化；llm-d 未动（pushed_at 仍 12:15:09Z、updated_at 仍 23:10:54Z、stars 仍 5262；窗口内 7 起事件全为评论/评审类）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**2611.15911 S2 0 首次取回成功、余项闭合**（RB 腿 05:52:24Z 200，执行者亲自现取复核同 paperId；RA 腿四次 429 为间歇限流）；BaRP/UCCI 引用复核本轮 11 次取回全部 429，未能复核；41 篇登记论文版本复核 0 变化，登记集维持 41 篇；R7 七组筛查零新论文（条目级与 57 轮相同），继续暂定。
- 纠正：RA 腿「57 轮漏检 LiteLLM LTS 补丁 v1.108.9/v1.106.10」→ 驳回（两版已分别于 56 轮与 49 轮登记）；57 轮 verification §5「11 条技能轨增量已登记到 PRACTICAL.md 未验证暂定清单」→ 不实（PRACTICAL.md 未验证暂定清单仍为 8 条，全文 grep 0 命中），本轮不再重复该表述；arXiv 11-13 批次第十五次结转（列表页 md5 第十三轮同一值）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十五次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（BaRP/UCCI 引用复核、SkillScope OA 12；UTC 午夜后贴近 00:00Z 重试）；qwen 动态定价再观察（下轮至少两次取回比对）；glm-5.2:batch 下架原因。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十二次重试；BaRP/UCCI 与 SkillScope OA 12 贴近 UTC 午夜重试，S2 429 端点至少两次间隔重试；qwen 至少两次取回比对），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 16:09 轮（第 59 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十二次 401（67 字节，md5 `2df0deaf…`，与前 31 次逐字节相同；执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者亲自重算复现）；计数器 2,297,865（+0），**恒等式第二十轮闭合**（14 卡全部零变动，连续第二轮全卡零变动）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 470 → 470、博客仍 184；定价（关窗后取回）：**qwen3.5-9b 第十一度摆动至 $1.25e-7、第十二度摆动 30 分钟内再跌至 $8.3e-8**（执行者现取 $8.3e-8 一致）、**deepseek-v4.1-flash-1017 prompt +294.7% 并新增分时 overrides 数组 6 条**（时段边界 HHMM 解读暂定），另 6 项调价，全目录真调价 8 个，`benchmarks` 字段 churn 连续第二轮为 0；五项目零新稳定版、零新预发布（LiteLLM 无 v1.110.0、无 dev.5，执行者亲自查 GitHub Releases API 复核）；CVE 无变化；**llm-d `updated_at` 星事件驱动获第三次阳性对照**（WatchEvent 06:28:03Z → updated_at 06:28:04Z，stars 5263；pushed_at 仍 12:15:09Z，成因仍未明）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**UCCI S2 8 复核成功、无变化**（RA 腿间隔重试 200；RB 腿两次 429 为间歇限流，不构成矛盾；执行者亲自现取 S2 两次亦 429）；2611.15911 与 BaRP 的 S2 均两次 429、全部 OpenAlex 均 429，未能复核；41 篇登记论文版本复核 0 变化，登记集维持 41 篇；R7 七组筛查零新论文（条目级与 58 轮相同），继续暂定；arXiv 11-13 批次第十六次结转（列表页 md5 第十四轮同一值）。
- 纠正：LiteLLM dev.3（05:09:18Z）技术上落在窗口开启后 18 秒，但 58 轮已作为边界项完整记录，本轮不重复计数（跨轮边界项以首次完整记录轮次为准）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十六次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（BaRP/UCCI 的 S2/OA、SkillScope OA 12；UTC 午夜后贴近 00:00Z 重试）；qwen 动态定价再观察（下轮至少两次取回比对）；deepseek-1017 分时 overrides 轮换观察；glm-5.2:batch 下架原因。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十三次重试；BaRP/UCCI 与 SkillScope OA 12 贴近 UTC 午夜重试；qwen 至少两次取回比对；deepseek-1017 overrides 轮换观察），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 19:09 轮（第 60 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十三次 401（67 字节，md5 `2df0deaf…`，与前 32 次逐字节相同；执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者亲自重算复现）；计数器 2,297,865（+0），**恒等式第二十一轮闭合**（14 卡全部零变动，连续第三轮全卡零变动；benchmarks 页 md5 与 59 轮不同但字节数相同，为构建产物轮换、非计数器变化；执行者亲自重算求和复核）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 470 → 470、博客仍 184；定价（关窗后取回）：**qwen3.5-9b 第十三度摆动至 $3.2e-8、第十四度摆动 30 分钟内再跌至 $2.6e-8**（迄今最低价；执行者关窗后现取 $2.5e-8，第十五度摆动供下轮核对）、**deepseek-v4.1-flash-1017 prompt −86.7% 且分时 overrides 数组 6 条整段消失**（存活不足 3 小时），另 4 项调价，全目录真调价 6 个，`benchmarks` 字段 churn 连续第三轮为 0；五项目零新稳定版、零新预发布（LiteLLM 无 v1.110.0、无 dev.5，执行者亲自查 GitHub Releases API 复核；额外核查 v1.108.10/v1.106.11/v1.108.11/v1.106.12 全部 404）；CVE 无变化；**llm-d `updated_at` 星事件驱动获一次阴性对照**（窗口内零事件、updated_at 仍 06:28:04Z、stars 仍 5263；阳性三次 + 阴性一次，方向一致、仍记为假说）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**BaRP S2 23 复核成功、无变化**（RA 腿初取 200；RB 腿两次 429 为间歇限流，不构成矛盾；执行者亲自现取 S2 200、citationCount 23）；UCCI S2 两次 429、全部 OpenAlex 12 次取回全部 429，未能复核；41 篇登记论文版本复核 0 变化，登记集维持 41 篇；R7 七组筛查零新论文（条目级与 59 轮相同），继续暂定；arXiv 11-13 批次第十七次结转（列表页 md5 第十五轮同一值）。
- 纠正：LiteLLM LTS 补丁 v1.108.9/v1.106.10 本轮仅复核、非新发现（已分别于 56/49 轮登记），禁止重复登记为本轮发现；RA/RB 腿 BaRP 引用表面冲突裁定（RA 初取 200 胜出，RB 两次 429 为间歇限流）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十七次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Dynamo 文档表缺行；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（UCCI 的 S2/OA、2611.15911 的 S2/OA、SkillScope OA 12；贴近 UTC 午夜 00:00Z 重试，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（下轮至少两次取回比对）；deepseek-1017 分时 overrides 是否重现；glm-5.2:batch 下架原因。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十四次重试；UCCI/2611.15911 引用贴近 UTC 午夜重试；qwen 至少两次取回比对；deepseek-1017 overrides 是否重现），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-15 22:09 轮（第 61 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十四次 401（67 字节，md5 `2df0deaf…`，与前 33 次逐字节相同；执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者亲自重算复现）；计数器 2,297,865（+0），**恒等式第二十二轮闭合**（14 卡全部零变动，连续第四轮全卡零变动；执行者亲自重算求和复核）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 **470 → 467**（移除 qwen3-vl-30b-a3b-thinking、stepfun/step-3.5-flash、tencent/hunyuan-a13b-instruct，新增 0，无厂商口径）、博客仍 184；定价（关窗后取回）：**qwen3.5-9b 第十六度摆动至 $2.4e-8（迄今最低价）、第十七度 $2.6e-8、第十八度 $3.6e-8、第十九度 $3.9e-8**（30 分钟内先跌后涨累计 +62.5%，$2.4e-8 低价存活不足 30 分钟；执行者关窗后现取 $4.3e-8，第二十度摆动供下轮核对）、**design_arena 大面积刷新（154 个模型仅 benchmarks 字段变动，终结连续三轮 churn 为 0）**、kimi-k3.1-pro:batch +48.4%、glm-5.3:batch prompt +39.8%/completion +102.2%、**deepseek-v4.1-flash-1017 分时 overrides 连续第二轮未重现**；五项目零新稳定版、零新预发布（LiteLLM 无 v1.110.0、无 dev.5，执行者亲自查 GitHub Releases API 复核；额外核查 LTS v1.108.10/v1.108.11/v1.106.11/v1.106.12 全部 404）；CVE 无变化；**llm-d `updated_at` 星事件驱动本轮记为未定对照**（窗口内仅 1 起 WatchEvent 11:24:05Z，updated_at 移至 11:46:17Z，相差 22 分 12 秒、星数净零；阳性三次 + 阴性一次不变，仍记为假说），**RouteLLM 获一次 1 秒级阳性对照**（WatchEvent 12:18:58Z → updated_at 12:18:59Z）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**UCCI S2 8 与 BaRP S2 23 复核成功、无变化**（RA 腿间隔重试 200；RB 腿三次 429 为间歇限流，不构成矛盾；执行者亲自现取 S2 两次均 429，未能独立复现——腿证据文件双腿交叉一致，见 verification §2-4）；2611.15911 S2 与全部 OpenAlex 均 429 未能复核；41 篇登记论文版本复核 0 变化，登记集维持 41 篇；R7 七组筛查零新论文（条目级与 60 轮相同），继续暂定；arXiv 11-13 批次第十八次结转（列表页 md5 第十六轮同一值）。
- 纠正：qwen 摆动计数——RB 腿「第十五/十六度」少计一度（漏计 60 轮执行者关窗后现取的 $2.5e-8 第十五度），采纳 RA 腿编号并补入 RB 腿复核的第十九度；**Railway 最终报告已于 2026-11-10 发布**（RB 腿一手核实 datePublished 2026-11-10T14:00:00Z，执行者亲自现取复核；此前多轮「仍未发布」记录作废）；**GHSA-w4c7-34wj-hqm8 在 GitHub 无记录**（API 与网页双 404，执行者亲自复核），仅第三方博客说法不得作 CVE-2026-114612 的佐证；Dynamo 文档表冲突裁决（42 轮「v1.6.1 已列」指版本切换器含 v1.6.1 文档树，表行本身始终无 v1.6.1 行）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十八次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（2611.15911 的 S2/OA、SkillScope OA 12；贴近 UTC 午夜 00:00Z 重试，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（下轮至少两次取回比对；第二十度 $4.3e-8 待核对）；deepseek-1017 分时 overrides 是否重现；新增 3 模型移除原因。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十五次重试；2611.15911 引用贴近 UTC 午夜重试；qwen 至少两次取回比对；deepseek-1017 overrides 是否重现），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 01:09 轮（第 62 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十五次 401（67 字节，md5 `2df0deaf…`，与前 34 次逐字节相同；执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者亲自重算复现）；计数器 2,297,865（+0），**恒等式第二十三轮闭合**（14 卡全部零变动，连续第五轮全卡零变动；执行者亲自重算求和复核）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 467 未变、博客仍 184；定价（关窗后取回）：**qwen3.5-9b 第二十一至二十六度摆动**（$5.9e-8 → $5.0e-8 → $3.7e-8 → $4.3e-8 → $4.4e-8 → $3.1e-8，60 分钟内振幅约 90%；执行者关窗后现取 $3.0e-8，第二十七度摆动供下轮核对）、**kimi-k3.1-pro 本体 prompt +732%**（$2.403e-7 → $2e-6 取整价，:batch +154.5%，暴涨价三次复核取回全部持续，执行者现取仍未回落，原因无厂商口径；:batch supported_parameters 缩减，去掉 logit_bias、min_p、repetition_penalty、top_p）、**deepseek-v4.1-flash-1017 分时 overrides 连续第三轮未重现**（:batch 先涨 +24.1% 又回落至 $2.9e-8）；**分时档位轮换口径首次明确**（minimax/hy3/hy4-preview 顶层价「变动」实为取回时刻落入廉价时段，overrides 数组逐字未变，不是真调价；今后跨轮比价须先对齐档位或直接比对 overrides 数组）；design_arena churn 回落至 0；全目录真调价 4 个；五项目零新稳定版、零新预发布（LiteLLM 无 v1.110.0、无 dev.5，执行者亲自查 GitHub Releases API 复核；LTS v1.108.10/v1.108.11/v1.106.11/v1.106.12 全部 404）；CVE 无变化（GHSA-w4c7 API 与网页双 404）；**llm-d `updated_at` 星事件驱动获第四次 1 秒级阳性对照**（WatchEvent 16:08:22Z → updated_at 16:08:23Z，stars 5263 → 5264；记分更新为阳性四次 + 阴性一次 + 未定对照一次，仍记为假说），**RouteLLM 获一次阴性一致对照**（窗口内 0 事件、updated_at 未移动）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**BaRP S2 23 与 2611.15911 S2 0 复核成功、无变化**（RA 腿初取 200、RB 腿重试二 200；执行者亲自现取 S2 两次均 429，未能独立复现——腿证据文件双腿交叉一致，见 verification §2-4）；UCCI S2 8 与全部 OpenAlex 均 429 未能复核；41 篇登记论文版本复核 0 变化，登记集维持 41 篇；R7 七组筛查零新论文，继续暂定；arXiv 11-13 批次第十九次结转（列表页 md5 第十七轮同一值）。
- 纠正：qwen 摆动计数——RA 腿「第十九至二十二度」少计两度（漏计 61 轮第十九度 $3.9e-8 与第二十度 $4.3e-8）、RB 腿「第十七/十八度」少计四度（执行者按两腿证据文件原始值与 mtime 续计为第二十一至二十六度）；**LangChain 4 包滞后性质修正**（61 轮「PyPI 单侧滞后」→「发布对象尚未创建」，GitHub Release 对象亦不存在，滞后约 40.5 小时）；60 轮「qwen $2.6e-8（第十四度，迄今最低价）」→ $2.4e-8（第十六度）（61 轮已纠正，本轮重申）。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第十九次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（UCCI 的 S2/OA、BaRP 的 OA、2611.15911 的 OA、SkillScope S2 11/OA 12、SkillLearnBench 66、ContinualSkillBench 13；贴近 UTC 午夜 00:00Z 重试，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（第二十七度 $3.0e-8 待核对）；deepseek-1017 分时 overrides 是否重现；kimi 暴涨价是否回落。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十六次重试；引用跟进贴近 UTC 午夜重试；qwen 下一度为第二十八度；kimi 暴涨价是否回落；deepseek-1017 overrides 是否重现），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 04:09 轮（第 63 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十六次 401（67 字节，md5 `2df0deaf…`，与前 35 次逐字节相同；执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者亲自重算复现）；计数器 2,297,815（−50），**恒等式第二十四轮闭合**（五轮全卡零变动终结，唯一变动 τ²-Bench Airline 338,687 → 338,637；执行者关窗后现取又见 GPQA 再 −195 至 2,297,620，供下轮核对）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 467 未变、博客仍 184；定价（关窗后取回）：**qwen3.5-9b 第二十八至三十一度摆动**（$4.7e-8 → $3.1e-8 → $3.6e-8 → $5.5e-8；执行者关窗后现取 $6.4e-8，第三十二度摆动供下轮核对）、**gpt-oss-20b 新涨 +150%**（$3e-8 → $7.5e-8，max_completion_tokens 117964 → 65536，执行者现取仍为 $7.5e-8）、**kimi-k2.7-code prompt 新涨 +18.75%**（$6e-7 → $7.125e-7，执行者现取确认）、**deepseek-v4.1-flash-1017:batch completion 三度爬升**（$3.19e-7 → $3.21e-7 → $3.36e-7）、**kimi-k3.1-pro 暴涨价 $2e-6/$1e-5 持续**（三次取回全部持续，执行者现取仍未回落，原因无厂商口径）、**deepseek-v4.1-flash-1017 分时 overrides 连续第四轮未重现**（执行者现取 pricing 仍无 overrides 字段）；分时档位轮换口径再获确认（minimax/hy3/hy4-preview overrides 数组逐字未变）；全目录真调价 4 个；五项目零新稳定版、零新预发布（LiteLLM 无 v1.110.0、无 dev.5，执行者亲自查 GitHub Releases API 复核；LTS v1.108.10/v1.108.11/v1.106.11/v1.106.12 全部 404）；CVE 无变化（GHSA-w4c7 API 与网页双 404；Dynamo 文档表仍缺 v1.6.1/v1.6.2 行，全文 "1.6.2" 字面计数 0 次）；**llm-d `updated_at` 星事件驱动获第五次 1 秒级阳性对照**（WatchEvent 19:21:33Z → updated_at 19:21:34Z，stars 5264 → 5265；记分更新为阳性五次 + 阴性一次 + 未定对照一次，仍记为假说），**RouteLLM 连续第二轮阴性一致对照**（窗口内 0 事件、updated_at 未移动）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**BaRP S2 23、UCCI S2 8、2611.15911 S2 0 与全部 OpenAlex（21/7/0）本轮两腿 36 次取回全部 429，未能复核**（执行者亲自现取 S2 亦 429）；41 篇登记论文版本复核 0 变化，登记集维持 41 篇；R7 七组筛查零新论文，继续暂定；arXiv 11-13 批次第二十次结转（列表页 md5 第十八轮同一值）。
- 纠正：qwen 摆动计数——RA 腿「第二十三至二十五度」少计五度、RB 腿「第十九/二十度」少计九度（执行者按两腿证据文件原始值与 mtime 续计为第二十八至三十一度；RA 腿「62 轮第三次取回 $3.1e-8（第二十二度）」亦误，$3.1e-8 实为第二十六度）；62 轮 RB 腿报告「OpenCode 最新版本仍为 v2.2.7」为误读（62 轮证据文件本身已含 v2.3.0，14 命中；执行者亲自复核）；RA 腿 Dynamo grep 正则 3 行命中系 `.` 通配符误匹配，已纠正为字面计数 0 次。
- 剩余未核实缺口：arXiv 11-13 批次枚举（第二十次结转，UTC 11-16 01:00 与 11-17 01:00 两档公告后补做）；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（BaRP 的 S2/OA、UCCI 的 S2/OA、2611.15911 的 S2/OA、全部 OpenAlex 21/7/0、SkillScope S2 11/OA 12、SkillLearnBench 66；ContinualSkillBench 13 本轮已复核成功；贴近 UTC 午夜 00:00Z 重试，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（第三十二度 $6.4e-8 待核对）；deepseek-1017 分时 overrides 是否重现；kimi 暴涨价是否回落；计数器 GPQA −195 是否继续变动。
- 下轮起点：先清上述缺口（arXiv 11-13 批次枚举为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十七次重试；引用跟进贴近 UTC 午夜重试；qwen 下一度为第三十三度；kimi 暴涨价是否回落；deepseek-1017 overrides 是否重现；计数器 GPQA 变动核对），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 07:09 轮（第 64 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十七次 401（67 字节，md5 `2df0deaf…`，与前 36 次逐字节相同；执行者亲自 curl 复核；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者取腿脚本在现取文件上重跑、输出逐字节一致）；计数器 2,297,620（第二十五轮闭合，GPQA −195 获证实）与 2,294,396（第二十六轮闭合，再 −3,224；GPQA 再 −2,574 至 1,553,604、τ² 再 −650 至 337,987，两卡两连跌；执行者亲自重算复现）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录窗口内 467 未变，**关窗后新增 qwen/qwen4-max**（created 02:18:46Z，$1.5e-6/$4.5e-6，目录 467 → 468，属关窗后事件）；定价（关窗后取回）：**qwen3.5-9b 第三十三至三十九度摆动**（$1.38e-7 → $1.40e-7 → $1.51e-7 → $1.63e-7 → $1.62e-7 → $1.17e-7 → $1.12e-7；第三十二度 $6.4e-8 仅凭 63 轮执行者文字记录，三份 models 文件中均无，文件证据缺口已明记）、**kimi-k3.1-pro 暴涨价崩塌后关窗后反弹**（23:44–00:14Z 间 prompt −88.0% 至 $2.403e-7，关窗后反弹至 $2.3e-6/$1.15e-5，崩塌未维持；:batch 仍 $7e-7/$6e-6）、kimi-k2.7-code 回落至 $6e-7、deepseek-v4.1-flash-1017:batch completion 回落（$2.53e-7 → $1.5e-7）、gpt-oss-20b +150% 维持 $7.5e-8（63 轮事件，RB 腿「又一轮」为误读）、gpt-5.6-sol 真降价 −9.1% 至 $4e-6/$2e-5、**deepseek-v4.1-flash-1017 分时 overrides 终扫重现**（6 条，顶层价 +1400%/+1011%，重现于 00:14–02:12Z 间、跨越关窗点、归属不确定）；全目录真调价 15 个；**第二次 design_arena 大面积刷新**（153 个模型、691 个 elo 非零变动，推翻 63 轮「一次性事件」定性）；五项目零新稳定版、零新预发布（Dynamo 文档表仍缺 v1.6.1/v1.6.2 行，全文 "1.6.2" 字面 0 次）；CVE 无变化（GHSA-w4c7 API 与网页双 404；Portkey GHSA 记录与 63 轮逐字节相同）；**llm-d `pushed_at` 01:47:43Z 成因已解**（gh-pages 徽章提交 84591a9，01:47:42Z，+1 秒；执行者亲自查 commits 复核），**星事件假说记分更新为阳性八次 + 阴性一次 + 未定对照一次**，仍记为假说；RouteLLM repo/events 403 未能复核（执行者现取 404），阴性对照链中断；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**登记集 41 → 43 篇**（LuthorSQL 2611.17780、UltraBERT-JEV 2611.17867，均为直接相关、v1；LuthorSQL 不列为第三项生产级联证据）；**OpenAlex 四项全部三次取回成功、无变化**（15911 为 0、BaRP 21、UCCI 7、SkillScope 12）；S2 取回成功、无变化（BaRP 23、15911 为 0、UCCI 8、ContinualSkillBench 13、SkillLearnBench 66）；**仅 SkillScope S2 11 仍未取回**（三次 429 + 午夜重试 429）；R7 七组筛查零新论文，继续暂定；arXiv 11-13 批次**双重口径**（列表页第二十一次结转，但内容已据 API 枚举完成，全站 302 篇、cs.AI 56 篇；周末提交仍未入索引）。
- 纠正：qwen 摆动编号——RA 腿「第三十三至三十六度」与 RB 腿「第三十三/三十四度」均误（执行者按两腿证据文件原始值与时间戳合并续计为第三十三至三十九度；第三十二度 $6.4e-8 文件证据缺口已明记）；RB 腿「gpt-oss-20b 又一轮 +150%」为误读（以 63 轮主取为基线；63 轮复取已为 $7.5e-8）；RB 腿星事件假说「第六次」漏计（轮间 llm-d 阳性与窗口内 RouteLLM 阳性）；RA 腿「llm-d pushed_at 成因未决」已解决（RB 腿：徽章提交 84591a9 +1 秒；执行者亲自复核）；RB 腿「15911 的 S2、BaRP 的 S2 三取全 429 未取回」为腿内口径（RA 腿证据文件证实两者本轮均取回成功）。
- 剩余未核实缺口：arXiv 11-13 批次列表页公告（第二十一次结转；下一档 UTC 11-17 01:00）与周末提交索引；decisions 新计数方法待定；LiteLLM v1.110.0 正式版、vLLM v0.33.1/v0.34.0；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（仅 SkillScope S2 11；贴近 UTC 午夜重试，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（第三十二度文件证据缺口、第三十九度 $1.12e-7）；deepseek-1017 分时 overrides 是否维持；kimi 暴涨价是否再崩塌；计数器 GPQA/τ² 是否继续下跌；RB 腿 9 项 GitHub 403 tag 核查（瞬时限流，下轮重试）。
- 下轮起点：先清上述缺口（arXiv 11-13 批次列表页公告核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十八次重试；引用跟进仅 SkillScope S2 11；qwen 下一度为第四十度；qwen4-max 关窗后新增核对；OTel #204 第二十七次抖动 02:38:50Z 核对），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 13:09 轮（第 66 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第三十八次 401（67 字节，md5 `2df0deaf…`，与前 37 次逐字节相同；执行者亲自 curl 复核，关窗后现取为第 39 次；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者取腿脚本在现取文件上重跑、输出逐项一致）；计数器 2,294,200（**第二十九轮闭合**，GPQA −196，三连跌 −195、−2,574、−196，累计 −2,965，1009 轮暂停一次；τ² 未动；执行者亲自重算复现）；decisions 新方法仍未定（/labs/decisions numberOfItems 仍 1；两 API 仍 404；旧值 37 记未决）；目录 468（qwen/qwen4-max 为 65 轮收尾，created 02:18:46Z）；定价：**qwen3.5-9b 第四十至四十六度摆动**（$1.07e-7 → $1.06e-7 → $1.10e-7 → $1.06e-7 → $8.9e-8 → $9.7e-8 → $6.2e-8，净 −44.6%；第四十三度两腿取值不一致存疑标注）、**kimi-k3.1-pro 崩塌价维持**（64 轮所称反弹两轮未再观测到，仍未核实）、deepseek-v4.1-flash-1017 overrides 维持、kimi-k3/kimi-latest supported_parameters 29 分钟内二次翻转、1017:batch 双向异动；五项目零新稳定版、零新预发布（**vLLM v0.34.0 tag 窗口内创建**，仅 tag、无 Release、无 PyPI，不记为发布，执行者亲自查 releases 端点复核；Dynamo 文档表仍缺 v1.6.1/v1.6.2 行）；CVE 无变化（GHSA-w4c7 API 与网页双 404；Portkey GHSA 记录与 65 轮逐字节相同）；**llm-d `pushed_at` 01:47:43Z 成因已解**（gh-pages 徽章提交 84591a9，01:47:42Z，+1 秒），**星事件假说记分更新为阳性十一次**（llm-d 8 + RouteLLM 3；RB 腿逐项表漏计 64 轮间 21:31 阳性，执行者亲自查事件流复核恢复），仍记为假说；RouteLLM 01:23:18Z 之后窗口内再无事件（后半段阴性对照）；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**arXiv 11-13 批次已公告**（列表页翻为 Mon 16 Nov，921,379 B，md5 `aa72d257…`，276 条；**第二十二次结转终结**；API cs.AI 117 → 131 篇，全站 859 → 960 篇；执行者亲自核对列表页证据文件）；新增 14 篇筛查：相邻（弱）2611.18471 一篇（裁判传输可靠性，约 31% 翻转率，论文自报）；登记集 43 篇零新版（Ballad v1 → v2 为 65 轮收尾，单位订正、内容零变）；**S2 四项本轮全部复核成功、无变化**（BaRP 23、UCCI 8、2611.15911 0、SkillScope 11——RB 腿取回成功为准，SB 腿「九取零回」为腿内口径）；仍未取回：全部 OpenAlex（21/7/0/12）、SkillLearnBench S2 66、ContinualSkillBench S2 13（任务 05:11Z 下发，错过 UTC 午夜预算窗口约 5 小时，65/66 两轮连续错过）；**Railway 最终报告已于 2026-11-10 发布**（此前多轮「仍未发布」说法作废，1009 腿已确立、1309 腿复核逐字节相同）；R7 七组筛查零新论文，继续暂定。
- 纠正：qwen 摆动编号——RA 腿「第四十一/四十二度」与 RB 腿「第三十六/三十七度」均误（执行者按两腿证据文件原始值与时间戳合并续计为第四十至四十六度；1009 RA 腿「$1.11e-7」为转录错误，纠正为 $1.10e-7；第四十三度两腿取值不一致存疑标注）；RA 腿「计数器第二十七轮闭合」误（1009 腿已做 27/28 轮，本轮为第 29 轮）；RB 腿星事件逐项表漏计 64 轮间 21:31 阳性（执行者亲自查事件流复核恢复，阳性十一次）；RA 腿「arXiv 第二十一次结转终结」误（1009 腿已记第二十二次）；65 轮「LuthorSQL/UltraBERT-JEV 是否增补登记集待主轮裁定」已解决（64 轮已登记）；SB 腿「S2 九取零回」为腿内口径（RB 腿取回成功为准）。
- 剩余未核实缺口：arXiv 周末（11-14/15）提交索引（待 UTC 11-17 01:00 档）；decisions 新计数方法待定；vLLM v0.34.0 Release 对象与 PyPI 上架；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 Opus 5.5 swe_atlas_qa high run 源数据不在公开 stats 接口内；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（全部 OpenAlex、SkillLearnBench S2 66、ContinualSkillBench S2 13；下轮务必贴近 UTC 午夜 00:00Z 后 1–6 分钟执行）；qwen 动态定价再观察（第三十二度文件证据缺口、第四十三度两腿取值不一致）；deepseek-1017 分时 overrides 是否维持；kimi 暴涨价是否再崩塌；计数器 GPQA/τ² 是否继续下跌。
- 下轮起点：先清上述缺口（arXiv 周末提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第三十九次重试；引用跟进贴近 UTC 午夜执行；qwen 下一度为第四十七度；vLLM v0.34.0 Release 对象盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 16:09 轮（第 67 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第四十次 401（主取 08:12:05Z）与第四十一次 401（复取 08:41:04Z），均为 67 字节、同一 md5（执行者关窗后现取仍为 401、同一 md5；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者取腿脚本在现取文件上重跑、输出逐项一致）；计数器 2,294,689（**第三十轮闭合**，+489；GPQA 1,553,408 零变动，三连跌中断；VGI +439、τ² +50；复取 2,294,939，τ² 29 分钟内再 +250，执行者亲自对现取页面复核）；板面 "last run" 推至 Nov 16、前端统计仍 302 runs——未联动，记为未决小口径；decisions 新方法仍未定（仍 1 个配方，两 API 仍 404，旧值 37 记未决）；目录 468，零增减；定价：**qwen3.5-9b 第四十七至四十八度摆动**（$5.8e-8 → $3.2e-8，相对第 40 度 −71.4%，回落加速）、**kimi-k3.1-pro 崩塌价维持**（64 轮所称反弹第三轮未再观测到，仍未核实）、deepseek-v4.1-flash-1017 顶层价轮换至高档 $3e-7/$1.2e-6（分时轮换第三次正反配对）、gemma-4-26b-a4b-it 真涨价 +33.3%、kimi-k3/kimi-latest supported_parameters 第三次翻转、claude-opus-4.8 is_moderated 翻转为 true、hy3 supported_parameters 收缩 5 项；五项目零新稳定版、零新预发布、零新 tag（**vLLM v0.34.0 仍仅 tag、无 Release 对象、无 PyPI**，第二次重试 404 定稿，执行者亲自查 releases 端点复核：最新仍 v0.33.0rc3；**RouteLLM 仓库从未创建过 GitHub Release**，Releases 列表为空数组；Dynamo 文档表仍缺 v1.6.1/v1.6.2 行）；CVE 无变化；**llm-d 第十二次阳性对照成立**（KateScarlet 07:49:28Z → updated_at 07:49:29Z，+1 秒；66 轮计数瑕疵闭合），**星事件假说记分更新为阳性十三次**（llm-d 9 + RouteLLM 4；RouteLLM 关窗后阳性 laingke 08:49:31Z → 08:49:32Z 按 64 轮先例并入，执行者亲自查事件流复核），仍记为假说；ForkEvent kirps8 未使 forks +1，原因未决；llm-d 事件流文件非严格时序，今后窗口归属一律按 created_at 过滤；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**arXiv 列表页未翻页**（仍 Mon 16 Nov，921,379 B，md5 `aa72d257…`，与 66 轮逐字节相同，执行者亲自现取复核；276 条）；API 20261113 全站 960 篇无变化，20261114/15/16 均为 0（周末与周一提交仍未入索引，待 UTC 11-17 01:00 档）；登记集 43 篇零变化（11-13 cs.AI 全量 131 篇集合与版本与 66 轮程序比对完全相同）；2611.18471 仍 v1；**R7 七组筛查＋三组补充筛查全部零新文**，继续暂定；引用跟进：OpenAlex 三项三轮九取零回（retryAfter 56,605 → 55,315 秒；任务 08:11Z 下发，错过 UTC 午夜预算窗口约 8 小时，三轮连续错过）；S2 四项 66 轮已闭合，本轮未重取；GitHub API 三波 403 均为瞬时限流，已全部取回成功。
- 纠正：66 轮 report.md 跨轨结论「星事件假说阳性十次」误（应为十一次，以 66 轮 delta 纠正三为准；本轮更新为十三次）；任务书「17 项待决」实核为 18 项；任务书称核查 Gemini nightly "#572"——实际本窗口内无 #572，最新仍为 #571；LuthorSQL/UltraBERT-JEV 与 Ballad v2 的 RB 腿「遗留」注记均为陈旧（64/66 轮已解决）；SkillEvaluator stars 与 SkillzBench trials 两腿点读差为时刻差（SA 为主、SB 为后取注记）。
- 剩余未核实缺口：arXiv 周末（11-14/15）与周一（11-16）提交索引（待 UTC 11-17 01:00 档）；decisions 新计数方法待定；vLLM v0.34.0 Release 对象与 PyPI 上架；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 "last run" 与前端统计未联动（未决小口径）；llm-d ForkEvent 未使 forks +1；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项（全部 OpenAlex、SkillLearnBench S2 66、ContinualSkillBench S2 13；下轮务必贴近 UTC 午夜 00:00Z 后 1–6 分钟执行，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（第四十三度两腿取值不一致存疑未闭合）；deepseek-1017 分时 overrides 是否维持；kimi 暴涨价是否再崩塌；计数器 GPQA/τ² 是否继续上行。
- 下轮起点：先清上述缺口（arXiv 周末与周一提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第四十二次重试；引用跟进贴近 UTC 午夜执行；qwen 下一度为第四十九度；vLLM v0.34.0 Release 对象盯梢），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 19:09 轮（第 68 轮）

- 本轮增量（D-R01–D-R08，落入 skill/references/api-routing-rounds.md + api-routing-papers.md）：OpenRouter benchmarks API 第四十二次 401（主取 11:11:35Z）与第四十三次 401（复取 11:41Z），均为 67 字节、同一 md5（执行者关窗后现取仍为 401、同一 md5；前端统计端点 216,554 字节逐字节相同，302 runs，+0；重算三档不变 $1.6427/$6.3675/$0.5757，板面 $6.43，$0.0625 缺口未闭合，执行者取腿脚本在现取文件上重跑、输出逐项一致）；计数器 2,295,428（**第三十一轮闭合**，+489；GPQA 1,553,408 零变动；VGI +439、τ² +50；复取 2,296,271，VGI 30 分钟内再 +843；主取增量拆分与 67 轮主取完全相同，记为未解释的巧合；执行者关窗后现取 2,296,074，−197——**计数器总值非单调，可下行**，新观测）；板面 "last run" 仍 Nov 16、前端统计仍 302 runs——未联动，记为未决小口径；decisions 新方法仍未定（仍 1 个配方，两 API 仍 404，旧值 37 记未决）；目录 468，零增减；定价：**qwen3.5-9b 第四十九至五十度摆动**（$1.7e-7 → $1.65e-7，相对第 48 度 +431% 后 −2.9%；第 50 度按先例计为独立一度，RB 腿异议记入）、**gpt-oss-20b 崩跌 −76% 至 $1.8e-8**（completion −70% 至 $9e-8，max_completion_tokens 65,536 → 117,964；63 轮 +150% 涨价全数回吐；执行者关窗后现取已回落至旧值 $7.5e-8/$3e-7/65,536，时点快照口径）、**kimi-k3.1-pro 崩塌价维持**（64 轮所称反弹第四轮未再观测到，仍未核实）、deepseek-v4.1-flash-1017 顶层价轮换回低档 $1.5e-7/$6e-7（分时轮换第四次正反配对）、1017:batch 漂移至 $3.7e-8/$3.3e-7、qwen4-max max_completion_tokens 393,216 → 131,072（价格未变）；五项目零新稳定版、零新预发布、零新 tag（**vLLM v0.34.0 仍仅 tag、无 Release 对象、无 PyPI**，第三次重试 404 定稿，执行者亲自查 releases 端点复核：最新仍 v0.33.0rc3；LiteLLM 无 v1.109.2/rc.3/rc.4/v1.110.0，v1.108.3 LTS 存在；Dynamo 文档表仍缺 v1.6.1/v1.6.2 行）；CVE 无变化；**llm-d 第十四次阳性对照成立**（zdunecki 10:02:56Z → updated_at 10:02:57Z，+1 秒；窗口内 4 例 WatchEvent、stars 净 +4，一一对应），**星事件假说记分更新为阳性十五次**（llm-d 11 + RouteLLM 4；llm-d 关窗后阳性 Derrick-Fang 11:17:25Z → 11:17:25Z，+0 秒，按 64/67 轮先例并入，执行者亲自查事件流复核），仍记为假说；**ForkEvent 成因已定位**（kirps8 fork 仍在且居 forks 列表首位，forks_count 934 约 3.7 小时未动——滞后/缓存派生计数，非实时；67 轮「fork 后删除」假说已证伪；执行者亲自查 forks 列表复核）；llm-d 事件流文件非严格时序，今后窗口归属一律按 created_at 过滤；文档可见文本零变化；Jev 11 因子权重与 Cloudflare 排序信号仍未披露。
- 引用跟进：**arXiv 列表页未翻页**（仍 Mon 16 Nov，921,379 B，md5 `aa72d257…`，与 66/67 轮逐字节相同，执行者亲自现取复核；276 条）；API 20261113 全站 960 篇无变化，20261114/15/16 均为 0（周末与周一提交仍未入索引，待 UTC 11-17 01:00 档）；登记集 43 篇零变化（11-13 cs.AI 全量 131 篇集合与版本与 67 轮程序比对完全相同）；2611.18471 仍 v1（**不增补登记集**，弱相邻，执行者裁定）；**R7 七组筛查＋三组补充筛查全部零新文**，继续暂定；引用跟进：OpenAlex 三项**四轮十二取零回**（任务 11:10Z 下发，错过 UTC 午夜预算窗口约 11 小时，四轮连续错过）；S2 十二取中九次 429、三次 200——**2611.15911 S2 0 与 SkillScope S2 11 已确认无变化**（RB 腿自纠草稿误称十二取全 429）；S2 为逐请求间歇限流，重试对 4 项中的 2 项有效。
- 纠正：qwen 第 50 度按先例计为独立一度（RB 腿异议记入）；Derrick-Fang 轮间阳性并入记分（十五次）；67 轮 RA 腿报告 §8「阳性十二次」为笔误，正本无误；ForkEvent 成因已定位（67 轮「原因未决」闭合）；gpt-oss-20b 崩跌为价格变化，非纠错；2611.18471 不增补登记集；RB 腿「待主轮裁定」注记已过期（LuthorSQL/UltraBERT-JEV 64 轮已登记、Ballad v2 66 轮已登记）。
- 剩余未核实缺口：arXiv 周末（11-14/15）与周一（11-16）提交索引（待 UTC 11-17 01:00 档）；decisions 新计数方法待定；vLLM v0.34.0 Release 对象与 PyPI 上架；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 "last run" 与前端统计未联动（未决小口径）；计数器总值非单调（关窗后 −197，新观测）；在线学习阈值真实流量一手证据仍无（R7 暂定继续）；引用跟进余项 8 项（BaRP S2 23、UCCI S2 8、BaRP OA 21、UCCI OA 7、2611.15911 OA 0、SkillScope OA 12、SkillLearnBench S2 66、ContinualSkillBench S2 13；下轮务必贴近 UTC 午夜 00:00Z 后 1–6 分钟执行，或由主会话裁定是否申请免费 API key）；qwen 动态定价再观察（第四十三度两腿取值不一致存疑未闭合；第 50 度 RB 腿异议已裁定）；deepseek-1017 分时 overrides 是否维持；kimi 暴涨价是否再崩塌；计数器 GPQA/τ² 是否继续上行。
- 下轮起点：先清上述缺口（arXiv 周末与周一提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第四十四次重试；引用跟进贴近 UTC 午夜执行；qwen 下一度为第五十一度；vLLM v0.34.0 Release 对象盯梢；计数器非单调再观察），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；qwen 动态定价再观察）。

## 2026-11-16 22:09 轮（第 69 轮）

- 本轮增量（D-R01–D-R08，落入 api-routing-rounds.md + api-routing-papers.md）：benchmarks 第 44/45 次 401；计数器第三十二轮闭合 2,296,315（+44；GPQA −395、VGI +439；复取 2,307,428，GPQA +10,674、VGI +439 第四次雷同；总值非单调再确认）；qwen 第 51/52 度（$7e-8 → $8.8e-8）；deepseek-v4.1-flash 暴跌 −63.8% 至 $5.07e-8；elo 刷新 170 模型；qwen4-max 新增 `repetition_penalty`；hy4-flash 关窗后上架；星事件记分十六次（llm-d 11 + RouteLLM 5），llm-d 首例明确阴性；五项目零新版，vLLM v0.34.0 第四次 404 定稿；登记集 43 篇零新版，R7 零新文；引用跟进 S2 66 复核成功、其余 7 项未取回，OpenAlex 五轮连续错过午夜窗口；arXiv 范围查询 RB 20 次 500 vs RA 200；Railway 最终报告已发布一手再确认；OpenCode post-mortem 仍未发布。
- 纠正：星事件记分基线十五次为准（68 轮 RA 腿报告「十四次」为笔误）；llm-d 首例明确阴性（强版本证伪，弱版本不受单独证伪）；vLLM tags 文件字节数差异说明（68 轮为部分取回）；OpenAlex retryAfter 字段口径两腿出入（记入未决）；#64205 归属订正与 #604 更新形态出入见技能轨（跨轨纠正记入总报告）。
- 剩余未核实缺口：arXiv 周末（11-14/15）与周一（11-16）提交索引（待 UTC 11-17 01:00 档）；decisions 新计数方法待定；vLLM v0.34.0 Release 对象与 PyPI 上架；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 "last run" 与前端统计未联动；计数器总值非单调；qwen 第 43 度两腿取值不一致存疑标注仍未闭合；kimi-k3.1-pro「反弹」说法第五轮未再观测到；gpt-oss-20b 关窗后已回落旧价；引用跟进余项 7 项（BaRP S2 23、UCCI S2 8、BaRP OA 21、UCCI OA 7、2611.15911 OA 0、SkillScope OA 12、ContinualSkillBench S2 13）；OpenAlex retryAfter 字段口径两腿出入。
- 下轮起点：先清上述缺口（arXiv 周末与周一提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第四十六次重试；引用跟进贴近 UTC 午夜执行；qwen 下一度为第五十三度；vLLM v0.34.0 Release 对象盯梢；计数器第三十三轮闭合），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；R7 在线学习阈值真实流量一手证据仍为零）。

## 2026-11-17 01:09 轮（第 70 轮）

- 本轮增量（D-R01–D-R09，落入 api-routing-rounds.md + api-routing-papers.md）：benchmarks 第 46/47 次 401；计数器第三十三轮闭合 2,315,006（+18,691；GPQA +18,252、VGI +439；复取 2,315,510，GPQA +504；执行者现取仍 2,315,510）；qwen 第 53/54/55 度（$1.7e-7 → $1.64e-7 → $1.7e-7 → $1.34e-7，四取三值）；第 43 度存疑标注闭合（机制已复现、历史数值无法重取）；deepseek-v4.1-flash 反弹 +176.1% 回 $1.4e-7；gpt-oss-20b 暴涨 +316.7% 至 $7.5e-8 且上限 117,964 → 65,536；分时轮换定性收紧（minimax/hy3/hy4-preview 顶层价变动实为档位轮换）；qwen4-max 移除 `repetition_penalty`；elo 本轮零刷新；kimi-k3.1-pro 崩塌价维持（64 轮所称反弹第六轮未再观测到）；星事件记分十八次（llm-d 13 + RouteLLM 5），RouteLLM 首例明确阴性；五项目零新版，vLLM v0.34.0 第五次 404 定稿；Dynamo 文档表 md5 出入已闭合（构建元数据轮换）；登记集零新版（总数两腿出入 RA 45 / RB 43，未决）；R7 零新文；引用跟进闭合 4 项（UCCI S2 8、SkillLearnBench S2 66、ContinualSkillBench S2 13、2611.15911 S2 0），剩余 5 项，OpenAlex 六轮连续错过午夜窗口；arXiv 范围查询 RB 19 次尝试全部失败（17×500 + 2×000，上游间歇故障）vs RA 200。
- 纠正：Dual-Layer 编号见记忆轨；llm-d nikhase 阳性成立（69 轮执行者关窗后现取已捕获 updated_at 14:49:47Z）；RouteLLM 首例明确阴性（强版本假说再受一例反证）；401 编号第 46/47 次（沿用每轮 +2 口径）；qwen 度数 53/54/55；OpenAlex retryAfter 口径出入已闭合（字段存在与否随响应而变）；登记集总数出入未决。
- 剩余未核实缺口：arXiv 周末（11-14/15）与周一（11-16）提交索引（待 UTC 11-17 01:00 档）；decisions 新计数方法待定；vLLM v0.34.0 Release 对象与 PyPI 上架；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 "last run" 与前端统计未联动；计数器总值非单调（本轮未再观测到下行）；登记集总数两腿出入（RA 45 / RB 43）；引用跟进余项 5 项（BaRP S2 23、BaRP OA 21、UCCI OA 7、2611.15911 OA 0、SkillScope OA 12）；RB arXiv 范围查询 19 次失败（下轮重取）。
- 下轮起点：先清上述缺口（arXiv 周末与周一提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第四十八次重试；引用跟进贴近 UTC 午夜执行；qwen 下一度为第五十六度；vLLM v0.34.0 Release 对象盯梢；计数器第三十四轮闭合；登记集总数核对），再按子主题轮转（成本/延迟权衡定量模型：KV 感知期望值模型与生产级联校准；R7 在线学习阈值真实流量一手证据仍为零）。

## 2026-11-17 04:09 轮（第 71 轮）

- 本轮增量（D-R01–D-R12，落入 api-routing-rounds.md + api-routing-papers.md）：计数器第三十四轮闭合（2,316,147，+637；四取逐项相同）；benchmarks API 第 48/49/50/51 次 401；qwen3.5-9b 第五十六至五十七度摆动（$1.28e-7 → $1.69e-7；$1.34e-7 为 55 度）；deepseek-v4.1-flash 维持 $1.4e-7、gpt-oss-20b 维持 $7.5e-8；真调价两项（qwen4-max −5.0%、gemma-4-26b-a4b-it −15.0%）；hy4-flash 两值来回跳；1017 overrides 逐字相同（分时轮换第七次正反配对）；elo 连续第二轮零刷新；目录 469 零增减；decisions 仍 1 个配方、旧值 37 仍未决；星事件假说记分更新为阳性二十次（llm-d 14 + RouteLLM 6；llm-d 星数算术不吻合新增一例待解释现象）；五项目零新版（vLLM v0.34.0 仍仅 tag，第六次 404 定稿；LiteLLM 五份旧 release 正文窗口内改写，无实质变更；Dynamo 文档表仍缺行；Portkey 2.0 未发布；CVE-2026-114612 厂商逐字确认仍缺）；登记集统一口径 43 篇零新版（出入已闭合）；R7 筛查零新文；引用跟进复核成功 4 项（剩余 8 项）；arXiv 列表页逐字节相同（11-14/15/16 提交仍为 0，待 UTC 11-17 01:00 档）；RB 腿 arXiv 范围查询 20 次尝试全部失败（上游间歇故障，登记未结）。
- 纠正：401 编号（第 48/49/50/51 次，两腿编号均错）；qwen 度数（56/57 度成立，顺延假说驳回）；登记集总数出入已闭合（两表逐篇相同，统一口径 43 + 2）；任务书 ENGRAM「第 42 次」与七项「第三十五轮」已过期（以 70 轮实测为准顺延为第 46 次/第三十九轮）。
- 剩余未核实缺口：arXiv 周末（11-14/15）与周一（11-16）提交索引（待 UTC 11-17 01:00 档）；decisions 新计数方法待定；vLLM v0.34.0 Release 对象与 PyPI 上架；llm-d 星数算术不吻合（+1 vs 3 起 WatchEvent）；RB arXiv 范围查询 20 次失败（下轮重取）。
- 下轮起点：先清上述缺口（arXiv 周末与周一提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第 52 次 401 观察；RB arXiv 范围查询重取），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-17 07:09 轮（第 72 轮）

- 本轮增量（D-R01–D-R10）：LiteLLM LTS v1.108.10 窗口内发布；计数器恒等式第三十五轮闭合 2,317,501（+1,354）；qwen3.5-9b 第五十八度 $1.12e-7；deepseek-v4.1-flash-1017 分时 overrides 整段移除（max_completion_tokens 393,216 → 943,718；配对计数中断）；z-ai/glm-5.3-prime 三档整齐 −40.0%（腿报告绝对值 1/7 错误已订正）；目录 470；elo 第三轮零刷新；llm-d 星事件阳性二十一次（fork 归属订正为 pydemia/llm-d 22:15:17Z）；vLLM v0.34.0 腿查时仍仅 tag（第七次 404 定稿；关窗后 23:22:00Z 已发布，计入下轮）；登记集 43 篇零新版，R7 继续暂定；引用跟进复核成功 6 项、未取回 9 项；benchmarks API 第 52/53 次 401。
- 剩余缺口：arXiv 11-13 批次列表页公告（第二十二次结转；下一档 UTC 11-17 01:00）与周末提交索引；decisions 新计数方法待定；引用跟进余项 9 项（八轮连续错过午夜窗口，下轮务必贴近 UTC 午夜 00:00Z 后 1–6 分钟执行）；arXiv 范围查询 RB 腿 20 次全败（下轮改区间式重取）；qwen 第五十九度（执行者关窗后现取 $1.68e-7 供核对）；vLLM v0.34.0 Release 正文核对；llm-d 星数算术不吻合（71 轮一例仍待解释）。
- 下轮起点：先清上述缺口（arXiv 周末与周一提交索引核查为未结台账首项；decisions recipe 数盯梢；OpenRouter benchmarks API 第 54 次 401 观察；RB arXiv 范围查询重取），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-17 10:09 轮（第 73 轮）

- 本轮增量（D-R01–D-R12、D-RB01–D-RB06）：vLLM v0.34.0 正式发布（2026-11-16T23:22:00Z，PyPI 同上架；72 轮未结项闭合）；计数器恒等式第三十六轮闭合 2,318,270（+769；GPQA +348、VGI +421、τ² 0）；qwen3.5-9b 第五十九度摆动至 $1.7e-7（+51.8%）；gpt-oss-20b 暴涨价消失（−60.0%）；elo 刷新恢复（181 个模型、997 条）；llm-d 星事件阳性二十二次（dominikcirko 23:55:40Z → updated_at 23:55:40Z）；forks −1 无事件配对新异常；LiteLLM 仍 v1.108.10、Portkey 仍 v1.15.2、Dynamo 仍 v1.6.2；decisions 仍 1 个配方；基准 API 第 54/55 次 401；arXiv 查询式敏感性得证（区间式 7/7、单日式 16/16 全 500，累计 75 次全败；下轮起区间式为准）；11-13 cs.AI 131 → 167（+36）；列表页第二十三次结转；登记集 43 → 44（SpecRelay 入集；2611.19123 列相邻）；引用 S2 取回 4 项、OpenAlex 27 次全 429（第九轮错过午夜窗口）；Jev Router 指南更新（更新戳出入待查）。
- 剩余缺口：arXiv 11-13 批次 +36 成因（假说未证实）与 11-15/16 批次索引追赶后补筛；decisions 新计数方法待定；引用跟进余项 13 项（九轮连续错过午夜窗口）；qwen 第六十度；Jev 更新戳出入（11-11 vs 11-06）；`all:skill` +12 中 4 篇未归因；Dynamo 文档表可见文本 diff（字节数 −6,283 未做）；llm-d forks −1 无事件配对。
- 下轮起点：先清上述缺口（arXiv 范围查询改用区间式 + `%20` 形式并做重复验证；引用跟进贴近 UTC 午夜 00:00Z 后 1–6 分钟执行，或主会话裁定是否申请免费 API key），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-17 13:09 轮（第 74 轮）

- 本轮增量（R-74-01–R-74-19）：arXiv 11-13 批次 +36 成因精确证实（28+8，未结项闭合）；11-14/15/16 索引追赶完成（全学科 645/661/432，cs.AI 104/81/47）；列表页第二十四次结转；计数器恒等式第三十七轮闭合 2,319,455（+1,185；唯一变动卡 Model Routers）；前端统计 runs 302 → 312（+10，滞后上架）；qwen3.5-9b 无第六十度；models.json 零差异、目录仍 470；LiteLLM 仍 v1.108.10（PyPI latest 1.109.1 为并行线）、Portkey 仍 v1.15.2、Dynamo 仍 v1.6.2、vLLM 仍 v0.34.0；decisions 仍 1 个配方；基准 API 第 56/57 次 401（执行者自验第 58 次同值）；llm-d 星事件阳性二十四次；forks 仍 935；登记集 44 → 46（2611.20417、2611.20732 入集；KaleidoKV 列相邻；PMR/CECAR 排除）；引用 S2 取回 3 项、OpenAlex 27 次全 429（第十轮错过午夜窗口）；Jev 更新戳出入闭合（R-73-16 闭合）；Dynamo 文档表可见文本 diff 闭合；**纠正：`%20` 式范围查询经两腿独立对照实验证伪（上限丢弃），改回 `+` 式为准、停用；73 轮「服务端不一致」结论及「改用 `%20` 形式」建议一并撤回**。
- 剩余缺口：decisions 新计数方法待定；arXiv 11-17 提交索引仍为 0（成因未区分）；引用跟进余项 S2 3 项 + OpenAlex 9 项（十轮连续错过午夜窗口）；qwen 第六十度；llm-d forks −1 成因仍未决、窗口内 WatchEvent lianjunli 单列未定；CVE-2026-114612 厂商逐字确认；Portkey CVE 修复版；Jev 11 因子权重与 Cloudflare 排序信号；板面 "last run" 与前端统计未联动。
- 下轮起点：先清上述缺口（引用跟进贴近 UTC 午夜 00:00Z 后 1–6 分钟执行，或主会话裁定是否申请免费 API key；arXiv 范围查询改回 `+` 式并做重复验证），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-17 16:09 轮（第 75 轮）

- 本轮增量（R-75-01～R-75-12，12 条 → api-routing-rounds.md）：计数器第三十八轮闭合 2,319,703（+248；GPQA +198、τ² +50）；qwen3.5-9b 第六十度 $1.69e-7、gemma-4-26b-a4b-it +17.6% 精确回弹（71 轮 −15.0% 的回弹）；benchmarks API 第 58/59 次 401；arXiv 11-16 批次追赶完成（全学科 1,277、cs.AI 159），纠正 74 轮「432 全量枚举闭合」；112 篇新文筛查零直接路由论文，相邻 2611.20873；星事件阳性二十五次；五项目零新稳定版；引用余项 16 项（S2 取回 2611.15911=0）；CVE-2026-114612 缺口复核（v1.106.6 正文 0 命中）；Cloudflare 排序信号部分披露（8 原因值、会话亲和、默认池 8 模型；分类器四维度与效用公式不在官方正文）；Dynamo 文档表逐字节相同；登记边界裁定（18471/19123/19253 仍相邻，20955 名称碰撞）。
- 剩余未核实缺口：引用跟进余项 16 项（第十一轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；decisions 新计数方法（旧值 37）；板面 "last run" 与前端统计未联动；Eco3R v2 版本服务滞后小异常；llm-d forks −1 成因；11-17 批次索引追赶后补筛；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）。

## 2026-11-17 19:09 轮（第 76 轮）

- 本轮增量（R-76-01～R-76-13，13 条 → api-routing-rounds.md）：计数器第三十九轮闭合 2,319,703（+0）；qwen3.5-9b 第六十一度 $8.4e-8（三档整齐腰斩）+ 3 项真调价（deepseek-v4.1-flash、gpt-oss-20b、z-ai/glm-5.3-flash）；arXiv 列表页第二十五次结转闭合（11-13/14/15/16 零变，11-17 仍 0）；登记集 46 篇零新版；星事件阳性二十七次（主轮裁定采纳 RA 27）；五项目零新稳定版；benchmarks API 第 60/61 次 401；decisions 仍 1 个配方；Cloudflare 分类器四维度与效用公式确有官方出处（博客 2026-09-30），75 轮 R-75-10 修正。
- 剩余未核实缺口：引用跟进余项 15 项（旧 11 + 新 4，第十二轮错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；CVE-2026-114612 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）。


## 2026-11-17 22:09 轮（第 77 轮）

- 本轮增量（R-77-01～R-77-14，14 条 → api-routing-rounds.md）：计数器第四十轮闭合 2,319,668（−35，首次轮间净减少）；qwen3.5-9b 第六十二度 $1.7e-7（约翻倍）+ 2 项真调价；elo 大面积刷新（184 模型）；前端 runs 312 → 308（−4）；星事件阳性二十八次（主轮裁定 C-77-01，juanrafap +1 秒配对计入，配对口径放宽至 ≤1 秒）；五项目零新稳定版；Dynamo 文档表 nightly 跳号至 1.8.0.dev20261117；decisions 仍 1 个配方、benchmarks API 第 62/63 次 401；CVE 两项仍缺厂商逐字确认；R7 十组筛查零新文；登记集 48 篇零新版；深挖（RouteBalance v2、Conformal Cascade v3、Jev 权重、Cloudflare 文档、arXiv 批量异常方法注记）。
- 剩余未核实缺口：引用跟进余项 17 项（第十三轮错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；计数器净减少与 runs 移除成因未定位；CVE-2026-114612 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；llm-d forks −1 成因；simsim-ai-2026 / werister / lianjunli 单列未定。


## 2026-11-18 01:09 轮（第 78 轮）

- 本轮增量（R-78-01～R-78-14，14 条 → api-routing-rounds.md）：计数器第四十一轮闭合 2,343,546（+23,878；GPQA +18,999）；qwen3.5-9b 无第六十三度；定价字段差异 7 个模型 = 真调价 3 项 + 分时轮换 3 项 + 缓存微动 1 项；elo 零刷新；前端 runs 308 → 306（−2）；星事件阳性二十八次维持，forks 935 → 936 与 ForkEvent 精确配对，updated_at 跟随 WatchEvent 的新支持证据；benchmarks API 第 64/65 次 401；decisions 仍 1 个配方；五项目零新稳定版；Dynamo 文档表逐字节相同；arXiv 11-13/14/15/16 零变、11-17 仍 0；博客 feed 滞后新增一篇 11-16 日期文章；R7 十组筛查零新文；登记集 48 篇零新版；引用跟进合并余项 14 项（S2 复核通过 3 项、OpenAlex 首次取回 2 项新基线、单次取回 2 项）。
- 纠正：定价「真调价 3 项」与「定价变动 7 项」无冲突（K-78-03）；UCCI OpenAlex 12 与 BaRP OpenAlex 22 均为单次取回、待下轮复核（K-78-05）。
- 剩余未核实缺口：引用跟进余项 14 项（第十四轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；计数器 +23,878 与 runs 移除成因未定位；CVE-2026-114612 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；llm-d forks −1 成因；simsim-ai-2026 / werister / lianjunli 单列未定；11-17 批次索引追赶后补筛。
- 下轮起点：先清上述缺口（引用跟进贴近 UTC 午夜执行为首项；arXiv 11-17 批次补筛；qwen 第六十三度；UCCI/BaRP OpenAlex 单次取回值复核），再按子主题轮转（路由成本/延迟权衡模型）。


## 2026-11-18 04:09 轮（第 79 轮）

- 本轮增量（R-79-01～R-79-14，14 条 → api-routing-rounds.md）：计数器第四十二轮闭合 2,347,090（+3,544；GPQA +1,750、VGI +1,744、τ² +50；两腿读数差 +1,110 系取回时刻差，主轮调和 K-79-03）；目录 470 → 474，OpenAI GPT-6.2 家族窗口内上架（厂商自报）；真调价 2 项（gemma-4-26b-a4b-it −15.0%、glm-5.3-flash 暴跌）；qwen3.5-9b 无第六十三度；elo 零刷新（210 个 benchmarks 字段变化全部为 schema 扩展）；前端 runs 306 持平；星事件阳性二十九次；benchmarks API 第 66/67 次 401；decisions 仍 1 个配方；五项目零新稳定版；Dynamo 文档表内容零变化、仅域名变更；arXiv 11-13/14/15/16 零变、11-17 仍 0；博客 sitemap 185 → 186（滞后入 feed 的 11-16 文章，窗口外）；LiteLLM 博客 RSS 首取 177 项；R7 十组筛查零新文；登记集 48 篇零新版；引用跟进合并余项 10 项（S2 复核通过 3 项、SkillScope S2 11 与 Experience Rot S2 0 新基线、OpenAlex 27 次全 429）。
- 纠正：派发书窗口日期笔误（K-79-02，「2026-11-18 20:09 UTC」应为「2026-11-17 20:09 UTC」）；计数器两腿读数出入调和（K-79-03）。
- 剩余未核实缺口：引用跟进余项 10 项（第十五轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key；BaRP OpenAlex 22 与 UCCI OpenAlex 12 仍为单次取回、待下轮优先复核）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；计数器 +3,544 与 glm-5.3-flash 大幅降价成因未定位；GPT-6.2 家族厂商自报未核实；CVE-2026-114612 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；llm-d forks −1 成因；simsim-ai-2026 / werister / lianjunli 单列未定；11-17 批次索引追赶后补筛。
- 下轮起点：先清上述缺口（引用跟进贴近 UTC 午夜执行为首项，重点复核 BaRP OpenAlex 22 与 UCCI OpenAlex 12；arXiv 11-17 批次补筛；qwen 第六十三度），再按子主题轮转（路由成本/延迟权衡模型）。


## 2026-11-18 10:09 轮（第 81 轮）

- 本轮增量（R-81-01～R-81-14，14 条 → api-routing-rounds.md）：计数器第四十四轮闭合 2,347,906（+816；GPQA −590、VGI +427、τ² −100、Model Routers +1,079；两腿逐卡一致）；LiteLLM v1.110.0（01:36:48Z，411 条变更，Rust router core #49513，decisions 新增 Parallel AI 与 EmpirioLabs）与 v1.108.11（LTS 补丁）窗口内发布；qwen3.5-9b 第六十六度 $1.43e-7（关窗后读数，绑时间戳）；glm-5.3-flash 第四次翻转回暴跌态；deepseek-v4.1-flash 新低站住；目录 478 持平；elo 零变化；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；decisions 仍 1 个配方；星事件阳性二十九次；benchmarks API 第 76–78 次 401（编号口径统一）；引用跟进 OpenAlex 余项清零（BaRP 22、UCCI 12 三次确认、SkillLearnBench 首取 62、ContinualSkillBench 首取 15），S2 余项剩 8 项；arXiv 11-17 批次已入 API 索引（cs.AI 40 篇），列表页仍未翻页；11-15 +2、11-16 +50 迟入索引；新直接路由论文 0 篇，R7 相邻证据 +2。
- 纠正：401 编号口径统一（K-81-01，80 轮终值 75 次、本轮 76–78 次）；腿编号 80→81 与派发书措辞澄清（K-81-02）；登记集与 qwen 度数裁定（K-81-07，RB 50 项为准、RA 48 篇为子集，qwen 采 RA 第 66 度）；JayR1031 闭合为不计分（K-81-08）。
- 裁定：2611.18471/19123/19253 不增补登记集（C-81-03，仍为集外相邻证据）。
- 剩余未核实缺口：S2 引用跟进余项 8 项（RouteBalance、BaRP、Route Anything、Conformal、UCCI、SkillLearnBench、EfficientSkill、Experience Rot，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 裸式区间语法 500 未恢复；11-17 批次列表页翻页与迟入复查；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；llm-d forks −1 成因；simsim-ai-2026 / werister / lianjunli 单列未定。
- 下轮起点：先清上述缺口（S2 贴近 UTC 午夜执行为首项；arXiv 11-17 批次补筛；qwen 第六十七度；裸式语法恢复验证），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-18 13:09 轮（第 82 轮）

- 本轮增量（R-82-01～R-82-14，14 条 → api-routing-rounds.md）：计数器第四十五轮闭合 2,348,345（+439，全为 VGI-Bench，第三次同值 +439，成因未定位；两腿逐卡一致；执行者第三时点关窗后取回 2,347,956）；qwen3.5-9b 无第六十七度（66 度 $1.43e-7 存活不足 2 小时 20 分即回落）；glm-5.3-flash 第五次翻转（05:13Z 暴跌态 → 05:25Z 暴涨态；context_length 差 1 未决小口径）；deepseek-v4.1-flash 新低站住；目录 478 持平；elo 零变化；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；板面 "last run" 首次出现 Nov 18，前端仍未联动；窗口内新稳定版 0 件、新预发布 2 件（LiteLLM v1.111.0-rc.1、Dynamo v1.5.1-kimi-k3-post.1）；llm-d forks +1 首例正向配对；RouteLLM 5712 → 5715（+3，4 起均单列未定）；星事件阳性二十九次；benchmarks API 第 79–81 次 401（编号口径统一）；arXiv 列表页翻页 Wed 18 Nov、Total 401，11-17 批次 cs.AI 40 → 192（迟入 +152，全量筛查 0 篇新直接路由论文）；API 索引滞后于网站页面（方法口径）；登记集 50 项仅 SkillEarth 升版 v2（76 轮窗口事件、82 轮迟发现）；R7 十组筛查全部闭合，前缀缓存组 3 → 5，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 复核通过 4 项，其余 S2 6 项 + OpenAlex 9 项未取回（第十七轮连续错过午夜窗口）。
- 纠正：RouteLLM 星数基线（K-82-01，81 轮终值 5712，omniscient 属本轮窗口）；401 编号统一（K-82-02，本轮终值第 81 次）；SkillEarth v2 轮次归属（K-82-04，76 轮窗口事件、82 轮迟发现）；81 轮 D-S40/D-S43 结转错误（K-82-07）；81 轮 Vibing Talk 算术笔误（K-82-08）；glm context_length 差 1 未决（K-82-09）；R7 十组筛查闭合口径（K-82-11）。
- 裁定：登记 4 项 R7 相邻证据候选（C-82-01，2611.22238、2611.22243、2611.17007、2611.13454）。
- 剩余未核实缺口：S2 引用跟进余项 6 项（RouteBalance、Route Anything、Conformal Cascade、SkillLearnBench、EfficientSkill、Experience Rot）+ OpenAlex 余项 9 项（第十七轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 裸式区间语法 500 未恢复（RB 腿 pass 3 两次再试均 500）；11-18 批次列表页翻页后迟入复查；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI-Bench +439 第三次同值成因未定位；glm 第五次翻转成因未定位；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；RA 腿 skill-full 全量枚举（1113/1115/1116/1117/1118-skill-full）与 bracket 查询未取回（RA 腿恢复脚本进程已不在，下轮 [0] 认领补做）。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；arXiv 11-18 批次补筛；qwen 第六十七度；裸式语法恢复验证；skill-full 全量枚举补取），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-18 16:09 轮（第 83 轮）

- 本轮增量（R-83-01～R-83-14，14 条 → api-routing-rounds.md）：计数器第四十六轮闭合 2,347,956（−389；GPQA −339、τ² −50；两腿逐卡一致；执行者第三时点关窗后取回 2,348,834）；qwen3.5-9b 第六十七度出现 $1.54e-7/$2.26e-7/$1.38e-7（绑 08:13Z 时点，关窗后约 2 小时内回落）；glm-5.3-flash 无第六次翻转（context_length 差 1 定位为顶层 vs 供应商层层间差）；deepseek-v4.1-flash 新低站住；目录 478 持平；elo 零变化；mimo-v2.5-pro 完成价翻倍（+107%，上下文 1,024,000→262,144）、gemma-4-26b +17.6%；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；窗口内新稳定版 0 件；星事件记分 29 → 32（llm-d +2、RouteLLM +1）；benchmarks API 第 82–85 次 401（编号口径统一）；arXiv 11-18 批次为空（totalResults=0，公告时点未到）；R7 十组筛查全部闭合，2611.22238 登记为前缀缓存组相邻证据，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 复核通过 6 项，其余 S2 5 项 + OpenAlex 9 项未取回（第十八轮连续错过午夜窗口）；arXiv 全量枚举未取回（记入未结台账）。
- 纠正：星事件记分 29 → 32（K-83-01，llm-d Senja20/zhangyuan0102 逐起配对、RouteLLM Mackfee 1:1 配对；82 轮主报告未采纳任一关账星数、记分维持 29）；401 编号统一（K-83-02，本轮终值第 85 次，两腿编号碰撞按时点顺延）；登记集 50 项为准（K-83-03，RB 53 = 50 + 集外单查 3 项）；qwen 第六十七度绑 08:13Z 时点（K-83-11）；glm 差 1 定位层间差（K-83-12）；arXiv 全量枚举未取回记入未结台账（K-83-10）；2502.12110 表值纠正（K-83-13，v9→v11，82 轮笔误）；82 轮「15 个为 11-16 批次 ID」修正（K-83-14）；11-16 批次迟入成因定位（K-83-15，周末提交登载滞后）；引用余项合并 14 项（K-83-16）。
- 裁定：2611.22238 登记为 R7 前缀缓存组相邻证据（C-83-04）；R7/M5/S1 续记（C-83-03）；11-18 批次筛查待下轮（C-83-05）。
- 剩余未核实缺口：S2 引用跟进余项 5 项 + OpenAlex 余项 9 项（第十八轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 裸式区间语法 500 未恢复；arXiv 全量枚举补取（1113/1115/1116/1117-csai-full 与 skill-full）；11-18 批次筛查；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI-Bench +439 成因未定位；glm 差 1 成因未定位；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定；mimo-v2.5-pro 涨价性质待下轮复核。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；arXiv 全量枚举补取；11-18 批次补筛；裸式语法恢复验证；mimo/gemma 涨价复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-18 19:09 轮（第 84 轮）

- 本轮增量（R-84-01～R-84-14，14 条 → api-routing-rounds.md）：计数器第四十七轮闭合 2,349,712（+1,756；仅 VGI-Bench；两腿逐卡一致；执行者第三时点关窗后取回 2,351,024）；qwen3.5-9b 无第六十八度，第六十七度存活至 11:11:35Z、约 11:15Z 前回落（存活不足约 3 小时）；glm-5.3-flash 无第六次翻转（差 1 维持）；deepseek-v4.1-flash 新低未站住（83 轮「连续第二轮站住」定性推翻）；deepseek-r1-0528 调价 + 供应商配置同变（prompt +14.2%、input_cache_read 字段移除、context 163,840→128,000、max_completion_tokens 32,768→32,000）；mimo/gemma 新价站住第二轮；目录 478 持平；elo 零变化；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；窗口内新稳定版 0 件；星事件记分 32 → 33（llm-d +1 逐秒配对、RouteLLM 单起不成阳性）；benchmarks API 第 86–89 次 401（编号口径统一）；arXiv 裸式区间语法仍 500，全量枚举由 RB 腿取回闭合（83 轮未结项关闭）；R7 十组筛查全部闭合，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 复核通过 5 项，其余 S2 4 项 + OpenAlex 9 项未取回（第十九轮连续错过午夜窗口）；Railway 最终报告首次一手确认（长期缺口闭合）；VGI 增量 439 倍数线索记为未证实假说。
- 纠正：星事件记分 32 → 33（K-84-01，RA 腿「llm-d 22 + RouteLLM 10」组分表述错误，以 K-83-01 为准纠正）；401 编号统一（K-84-15，本轮终值第 89 次，两腿编号碰撞按时点顺延）；qwen 第六十七度时点绑定（K-84-03，存活至 11:11:35Z、约 11:15Z 前回落）；deepseek-v4.1-flash 新低未站住、83 轮定性推翻（K-84-13）；83 轮 CHANGELOG 行数口径纠正（K-84-14，取时行数 5,137 vs 落盘文件 5,136）；RB 腿登记集脚本键名 bug 方法纠正（K-84-07，83 轮结论修正口径下成立，所引脚本输出表述纠正）；arXiv 全量枚举出入裁定（K-84-08，RB 闭合、RA 时段抖动）；VGI 439 倍数线索未证实（K-84-10，关窗后 +1,312 非整倍数）；llm-d 窗口外星事件不计入本轮（K-84-11，5,299 于 12:03:33Z）；计数器第三时点不推翻窗口内闭合（K-84-12）。
- 裁定：引用余项合并口径（C-84-01，83 轮余 14 项中复核通过 1 项，余 S2 4 项 + OpenAlex 9 项；RouteBalance/SkillLearnBench/Synapse 沿用 83 轮值；SA 腿说法被 SB 200 取回覆盖）；11-18 批次筛查待下轮（C-84-02，公告档 2026-11-19 01:00Z；下轮须带 `?show=2000`）；R7 续记（C-84-04）。
- 剩余未核实缺口：S2 引用跟进余项 4 项 + OpenAlex 余项 9 项（第十九轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 11-18 批次筛查；arXiv 裸式区间语法 500 未恢复；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实；glm 差 1 成因未定位；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定；llm-d 5,299 窗口外事件留待下轮；mimo/gemma 涨价复核完成（站住第二轮，真调价候选升级）。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；11-18 批次筛查；裸式语法恢复验证；RB 腿登记集脚本键名修正），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-18 22:09 轮（第 85 轮）

- 本轮增量（R-85-01～18，18 条 → api-routing-rounds.md）：计数器第四十八轮闭合 2,353,410（+3,698；VGI +3,946、GPQA −198、τ² −50；两腿逐卡一致；439 倍数假说再添反例）；qwen3.5-9b 无第六十八度（62 度值复现并站住第二轮）；glm-5.3-flash 无第六次翻转（暴涨态，差 1 维持）；deepseek-v4.1-flash 再破新低（$5.07e-8/$2.028e-7/$1.014e-9，三档同幅 −83.1%；max_completion_tokens 943,718 → 393,216）；r1-0528 84 轮调价站住第二轮；mimo/gemma 新价站住第三轮；目录 478 持平；elo 大刷新（186 个模型、1,144 格下调 vs 136 格上调）；auto 三件套 max_completion_tokens 两读内振荡；glm-5.2 失去 parallel_tool_calls；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；窗口内新稳定版 0 件；星事件记分 33 → 35（llm-d +2 逐秒配对、RouteLLM 零事件）；benchmarks API 第 90–93 次 401（编号按时点顺延统一）；arXiv 裸式区间语法仍 500（503/429 交替），全量枚举由 RB 腿取回闭合（七日零变化）；R7 十组筛查全部闭合，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 复核通过 4 项，其余 S2 5 项 + OpenAlex 9 项未取回（第二十轮连续错过午夜窗口）；Railway 最终报告正文深读完成（数字全部厂商自报未独立核实）。
- 纠正：qwen 无第六十八度（K-85-02，RA「第六十八度」说法不成立，三档与 84 轮 RB 文件逐值相同）；星事件记分 33 → 35（K-85-03，RA 腿基线「llm-d 23 + RouteLLM 10」组分表述错误，以 25+8 为准；K-84-11 的 5,299 事件判定为本轮阳性）；401 编号统一（K-85-04，本轮终值第 93 次，两腿各自写的 90/91 只计了本腿两取，须合并）；arXiv 全量枚举出入裁定（K-85-06，以 RB 为准闭合，RA 分页 19 页全 500 为窗口时段抖动，只记方法注记，不记未结项）；登记集脚本键名 bug 修正办结（K-85-12，K-84-07 办结；RA 重跑 84 轮数据 50/50 一致，RB 修正后 53/53 一致）。
- 裁定：引用余项合并口径（C-85-01，S2 复核通过 4 项，余 S2 5 项 + OpenAlex 9 项未取回；第二十轮连续错过 UTC 午夜窗口）；11-18 批次筛查待下轮（C-85-02，公告档 2026-11-19 01:00Z；pastweek 须带 `?show=2000`）；R7 续记（C-85-04）。
- 剩余未核实缺口：S2 引用跟进余项 5 项 + OpenAlex 余项 9 项（第二十轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 11-18 批次筛查；arXiv 裸式区间语法 500 未恢复（故障形态新增 503/429 交替）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（再添反例）；glm 差 1 成因未定位；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；11-18 批次筛查；裸式语法恢复验证），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 01:09 轮（第 86 轮）

- 本轮增量（R-86-01～18，18 条 → api-routing-rounds.md）：计数器第四十九轮闭合 2,354,288（+878；仅 VGI +878；两腿逐卡一致；+878 = 439×2，439 倍数假说获一例整倍数支持，但 85 轮反例仍在）；qwen3.5-9b 第六十八度出现（$1.4e-7/$2.06e-7/$1.26e-7，全新 distinct 价位；关窗后漂移 $1.3e-7 归属下轮）；kimi-k2.7-code 真调价（$7e-7/$3.75e-6 → $6e-7/$3.5e-6，首次观测）；deepseek-v4.1-flash 新低站住第二轮；r1-0528 调价站住第三轮；mimo/gemma 新价站住第四轮；三家分时档位轮换（minimax-m3.1-flash、tencent/hy3、hy4-preview）；elo 零刷新（artificial_analysis 168 个模型置空，elo 数值格 0/0）；auto 三件套 max_completion_tokens 振荡继续（RB 更晚读数推翻 RA「终结」结论）；星事件记分 35 → 36（llm-d +1 逐秒配对、RouteLLM 零事件）；benchmarks API 第 94–97 次 401（终值第 97 次）；arXiv 裸式区间语法仍 500（括号计数修正语法后全部闭合）；全量枚举四批零变化；11-18 批次截至核查时未公告；R7 十组筛查全部闭合，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 复核通过 3 项，其余 S2 5 项 + OpenAlex 9 项未取回（第二十一轮连续错过午夜窗口）；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；窗口内新稳定版 0 件。
- 纠正：auto 三件套振荡未终结（K-86-01，RB 更晚主读 128,000 推翻 RA「终结于低档」）；85 轮 P1 括号查询串畸形（K-86-02，`202611${d}` 展开为 10 位日期；85 轮「P1 前 6 次全 500 系服务端故障」定性修正为两因叠加）；Railway 报告日期精度（K-86-03，R-85-15「2026-11-10」为最后更新日，发布日 2026-11-05）；401 编号统一（K-86-04，本轮终值第 97 次）；qwen 第六十八度成立（K-86-05，85 轮 $1.7e-7 说法被正确驳回是 62 度值复现）。
- 裁定：星事件记分 35 → 36（C-86-01）；401 编号统一口径延续（C-86-03）；引用余项合并口径（C-86-05，S2 余 5 项 + OpenAlex 余 9 项未取回；第二十一轮连续错过 UTC 午夜窗口）；R7 续记。
- 剩余未核实缺口：S2 引用跟进余项 5 项 + OpenAlex 余项 9 项（第二十一轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 11-18 批次筛查；arXiv 裸式区间语法 500 未恢复；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（+878 为 439×2 整倍数一例支持）；kimi-k2.7-code 调价是否站住；qwen 关窗后新价 $1.3e-7 核查；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；11-18 批次筛查；裸式语法恢复验证；qwen 关窗后新价核查），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 04:09 轮（第 87 轮）

- 本轮增量（R-87-01～16，16 条 → api-routing-rounds.md）：计数器第五十轮闭合 2,355,116（+828；VGI +878、τ² −50；两腿逐卡一致；主轮求和复核吻合；+878 = 439×2，439 倍数假说再添一例整倍数支持，但 85 轮反例仍在）；qwen3.5-9b 第六十九度成立（$1.3e-7/$1.91e-7/$1.17e-7，18:07Z 86 轮执行者自取，全新 distinct 价位；K-86-09 登记属实）；z-ai/glm-5.2 疑似真调价（首次观测，下轮复核）；gemma 三档振荡回归（$7.65e-8 为历史三档轮换值之一，主轮解析 57 个模型文件核对）；deepseek-v4.1-flash-1017 窗口内调价（20:14:37Z $3.6e-8/$5e-7 → 约 20:18Z $3.5e-8/$1.5e-7，关账值取较晚读数）；kimi-k2.7-code 真调价站住第二轮；deepseek-v4.1-flash 新低站住第三轮；r1-0528 调价站住第四轮；mimo 新价站住第五轮；三家分时档位轮换 overrides 未变；auto 三件套两读 64,000；elo 连续第二轮零刷新；artificial_analysis 置空溯源闭合（86 轮「168 个模型置空」出处定位，本轮 aa None 257 已成稳定 served 状态）；星事件记分 36 → 37（RouteLLM +1 逐秒配对）；benchmarks API 401 终值第 101 次（编号按时点顺延统一）；arXiv 裸式区间语法仍 500（括号计数全部闭合）；全量枚举四批零变化；11-18 批次截至核查时未公告；R7 十组筛查全部闭合，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 取回 3 项（BaRP 23、Route Anything 0、ContinualSkillBench 13 首次基线），其余 S2 3 项 + OpenAlex 9 项未取回（第二十二轮连续错过午夜窗口）；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；窗口内新稳定版 0 件。
- 纠正：qwen 第六十九度成立（K-87-01，86 轮 K-86-09 登记的关窗后漂移值本轮核查成立；RB「瞬时漂移、不另立度数」异议驳回）；gemma 三档振荡（K-87-02，主轮解析 57 个模型文件；86 轮「新价站住第四轮」定性修正）；deepseek-1017 窗口内调价（K-87-03，两腿原文件核对，调价发生在 20:14:37Z 与约 20:18Z 之间）；401 编号统一（K-87-04，本轮终值第 101 次）。
- 裁定：星事件记分 36 → 37（K-87-05）；引用余项合并口径（S2 余 3 项 + OpenAlex 余 9 项未取回；第二十二轮连续错过 UTC 午夜窗口）；R7 续记。
- 剩余未核实缺口：S2 引用跟进余项 3 项 + OpenAlex 余项 9 项（第二十二轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 11-18 批次筛查；arXiv 裸式区间语法 500 未恢复；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（+878 为 439×2 整倍数第二例支持）；z-ai/glm-5.2 疑似调价复核；deepseek-1017 新价复核；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；11-18 批次筛查；z-ai/glm-5.2 与 deepseek-1017 新价复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 07:09 轮（第 88 轮）

- 本轮增量（R-88-01～16，16 条 → api-routing-rounds.md）：计数器第五十一轮闭合 2,354,927（−189；仅 GPQA Diamond −189；两腿逐卡一致；主轮求和复核吻合；VGI 0，+878 势头中断，439 倍数假说未获新支持）；qwen3.5-9b 第七十度成立（$1.42e-7/$2.1e-7/$1.28e-7，全新 distinct 价位；转变跨窗口关闭点，只记本轮首次观测）；deepseek-v4.1-flash-1017 改为分时档位（新增 6 段 weekday overrides，顶层价 $1.5e-7/$6e-7/$3e-9，MCT 943,718 → 393,216）；归属纠正（87 轮 R-87-05 所称「1017 新价」实为 `~flash-latest` 本轮新价）；站住轮次六项推进（z-ai/glm-5.2 第二轮、gemma $7.65e-8 档第二轮、kimi-k2.7-code 第三轮、deepseek-v4.1-flash 新低第四轮、r1-0528 第五轮、mimo 第六轮）；目录 478 → 480（新增 google/gemini-4-neon 及 :batch，created 字段早于窗口，目录收录滞后）；elo 连续第三轮零刷新；artificial_analysis 置空态完全稳定（aa None 257 → 259，+2 恰为 2 个新模型自带 None）；llm-d 星数 −1 未配对（5301 → 5300，记分维持 37，新增单列未定）；benchmarks API 401 终值第 105 次（编号按时点顺延统一）；arXiv 裸式区间语法仍 500（括号计数全部闭合）；全量枚举四批零变化；11-18 批次截至核查时未公告；R7 十组筛查全部闭合，生产级联证据仍仅 CascadeRec + HIRA；引用跟进 S2 取回 8 项（BaRP 23、ContinualSkillBench 13、UCCI 8、SkillLearnBench 66、Synapse 0、Experience Rot 0、RouteBalance 2、2611.15911 0），其余 S2 4 项 + OpenAlex 9 项未取回（第二十三轮连续错过午夜窗口）；前端 runs 316，三档重算 $1.6427/$6.3675/$0.5757，板面 $6.43 缺口 $0.0625 未闭合；窗口内新稳定版 0 件；vLLM v0.34.0 发布说明窗口内编辑一次（勘误）。
- 纠正：deepseek-1017 归属（K-88-03，主轮亲自解析 87/88 两轮两腿 models.json 原文件；87 轮 R-87-05 / K-87-03 修正）；auto 三件套 MCT 振荡继续（K-88-04，RA 64,000 vs RB 128,000；87 轮 R-87-09「终结于低档」修正）；任务书基线订正（K-88-09，LiteLLM v1.110.0 / vLLM v0.34.0，主轮浏览器直读 GitHub Releases API 复核；任务书 v1.108.3 / v0.33.0 作废）。
- 裁定：benchmarks 401 编号统一（K-88-01，终值第 105 次）；qwen 第七十度成立（K-88-02）；取回时刻差口径（K-88-05，SkillzBench trials、Vibing Talk 浏览量、SkillEvaluator stars 关账值取较晚读数）；引用跟进合并（K-88-12）；llm-d −1 星登记为新增单列未定（K-88-13）；R7 续记。
- 剩余未核实缺口：S2 引用跟进余项 4 项 + OpenAlex 余项 9 项（第二十三轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 11-18 批次筛查；arXiv 裸式区间语法 500 未恢复；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（两正一反一零）；qwen 第七十度是否站住；deepseek-1017 分时档位复核；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定；llm-d −1 星新增单列未定。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；11-18 批次筛查；qwen 第七十度与 deepseek-1017 分时档位复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 10:09 轮（第 89 轮）

- 本轮增量（R-89-01～13，13 条 → api-routing-rounds.md）：计数器第五十二轮闭合 2,355,175（+248；GPQA Diamond 1,593,098，+198；τ²-Bench Airline 342,436，+50；其余 12 卡零变动；RA/RB 四读逐卡相同；主轮解析两份 benchmarks-page.json 逐卡复核吻合）；VGI-Bench 306,540 本轮 0，439 倍数假说数据点两正一反两零，仍未证实；定价 13 个模型变动：qwen3.5-9b 第七十度未站住（一轮即撤回，回 62 度旧价 $1.7e-7/$2.5e-7/$1.53e-7）；deepseek-v4.1-flash 85 轮新低终结（$5.07e-8 → $3e-7，约 ×5.9；MCT 393,216 → 943,718）；deepseek-v4.1-flash-1017 顶层价翻倍但 6 段 overrides 逐字相同（$1.5e-7 → $3e-7，不得写成简单涨价）；z-ai/glm-5.2 异常价目（prompt −98.9%、completion 约 ×2.8、cache −95.7%，88 轮「站住第二轮」不成立，记异常价目待下轮复核）；minimax-m3.1-flash 恰好翻倍 ×2（分时轮换）；qwen3.5-27b 两档 −35%；gemma-3-27b-it completion −64.4% 且 input_cache_read 字段消失、MCT 117,964 → 16,384；目录 480 持平；站住轮次：gemma-4-26b 第三轮、r1-0528 第六轮、mimo 第七轮、kimi-k2.7 第四轮；auto 三件套 MCT 回 128,000（第 4 次振荡）；meta/muse-spark-1.3 `is_moderated` False → True（成因未定位）；elo 零刷新终结：180 个模型、1,331 格 design_arena elo 变化（RB 腿交叉核对 184 模型对象/1,324 格，差 7 格留待下轮统一）；gemini-4-neon 及 :batch 获 AA 指数（intelligence_index 61.4 等 6 项，厂商自报），aa None 259 → 257；星事件：llm-d +1 配对成立（5300 → 5301；yuzhiquan WatchEvent 01:08:50Z，updated_at +1 秒），记分 28 → 29；forks +1 配对成立；RouteLLM +1 配对成立（nicksan2c 01:48:42Z，逐秒配对），记分 9 → 10；总分 37 → 39；llm-d pushed_at 关窗后移至 02:13:33Z，事件流内仍无对应 PushEvent（第十次同类观测）；benchmarks API 401 终值第 109 次（四份证据文件 md5 均为 `2df0deaf…`，67 字节；编号按取回时点顺延统一）；新直接路由论文 2611.23751 AnyS1 登记；相邻证据登记 2611.22432 v2（JevRSI）、2611.23375、2611.23558、2611.23716（附限定）；2611.23838 排除；深挖全文两篇：2611.22238《Context Has a Price》（缓存价 1/10 时上下文管理成本反增 33–42%/38–116%；新增「缓存折扣感知成本核算」口径）与 2611.22243《From Token Changes to Task Failures》（单一位翻转可使 TriviaQA 类任务 59–60% 受影响回答变错；新增「共享前缀完整性风险项」口径）；均为论文自报；登记集 57 项零新版（53 + 4 候选，摘要页与 API 双路 53/53 一致）；R7 十组筛查九组零新增，router-llm 组 5 → 7；OpenAlex 九项 27 取 24 回、全部复核通过、无变化，终结连续 23 轮未全取回纪录，余项 9 → 0；S2 四项 12 取 1 回（Conformal Cascade 2 无变化），第二十四轮连续错过午夜窗口；arXiv：11-18 批次已入 API 索引（cs.AI 28，截至 02:30Z 的部分读数，提交 00:03–05:02Z；`all:skill` 4；首取 0、31 秒后再取 28，索引填充中）；列表页未翻页（与 88 轮逐字节相同，仍 Wed 18 Nov，Total 401）；11-17 迟入 47 篇（192 → 239）；11-15 迟入 3 篇（83 → 86）；裸式区间语法仍 500（两次首试即 500，913 字节，自 79 轮起约 9 日）。
- 纠正：z-ai/glm-5.2「站住第二轮」不成立（K-89-06 关联，88 轮口径修正）；RA 腿「直接路由 0 篇」不成立（K-89-05，2611.23751 AnyS1 登记为直接路由论文）；RA 腿「2611.23558 筛查假阳性已排除」不成立（K-89-06，登记为相邻证据）。
- 裁定：benchmarks 401 编号统一（K-89-04，终值第 109 次）；AnyS1 登记为直接路由论文（K-89-05）；2611.23558/2611.23716 登记为相邻证据、2611.23838 排除（K-89-06）；elo 计数口径残差（K-89-07，RA 1,331 格为 headline，RB 1,324 格为交叉核对，差 7 格留待下轮统一）；R7 续记（生产级联证据仍仅 CascadeRec + HIRA；在线学习阈值真实流量一手证据仍为零）。
- 剩余未核实缺口：S2 引用跟进余项 4 项（第二十四轮连续错过午夜窗口，下轮贴近 UTC 午夜 00:00Z 后 1–6 分钟执行或申请免费 API key）；arXiv 11-18 批次筛查未闭合（API 仅 28 篇部分读数、列表页未翻页）；arXiv 裸式区间语法 500 未恢复；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（两正一反两零）；glm-5.2 新价是否站住；1017 顶层翻倍是否站住；gemma-3-27b-it 缓存字段缺失是否数据异常；muse-spark `is_moderated` 成因；elo 7 格残差；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定；llm-d −1 星新增单列未定（88 轮）。
- 下轮起点：先清上述缺口（S2 贴近 UTC 午夜执行为首项；11-18 批次筛查闭合；glm-5.2 与 1017 复核；elo 7 格残差统一），再按子主题轮转（路由成本/延迟权衡模型）。
## 2026-11-19 13:09 轮（第 90 轮）

- 本轮增量（R-90-01～16，16 条 → api-routing-rounds.md；R-90-16 落点为本轨笔记执行注记）：计数器第五十三轮闭合 2,357,167（+1,992；GPQA Diamond −198、VGI-Bench +2,190，其余 12 卡零变动；主取与第二读逐项相同；主轮自两份页文件独立重算求和一致）；benchmarks API 401 统一终值第 113 次（RA 05:14:26/27Z 为第 110/111 次，RB 05:16:23/27Z 为第 112/113 次）；elo 89→90 轮零刷新，88→89 轮残差统一为分项口径（数值变化 1,324 格 + 新出现 7 格 = 1,331 格）；定价五模型变动：qwen 第 71 度首次观测未站住、1017 档位映射反例推翻 89 轮闭合结论、glm-5.2 异常价目站住第二轮、glm-5.1 三档异向新变（top ctx 202,752→204,800）、0731 与 ~flash-latest prompt 联动、auto 三件套 MCT 在 64,000/128,000 间振荡；muse-spark `is_moderated` 方向纠正（true→false，89 轮摘要错误）；星事件记分 39 → 41（llm-d +2 双配对）；LiteLLM 窗口内新稳定版 2 件（v1.109.2、v1.110.1）；新直接路由论文 2611.24324《Timeout Cascades》登记；AnyS1 全文深读闭合；R7 十组：threshold 1→2（OASIS）、cost-cascade 11→15（Cascadeability），研究级证据 0→1，生产流量证据仍为零；相邻证据新增 9 篇；2611.23716 非路由（89 轮候选撤销）、2611.24379 MoE-Context 排除、2611.23838 排除维持；登记集 57 项零新版；S2 余项 4→1，OpenAlex 余 7 项，第二十五轮连续错过午夜窗口；arXiv 11-18 批次闭合（cs.AI 164、all:skill 19），11-16 批次闭合（209=209），11-19 批次未闭合，裸式区间语法仍 500；decisions 仍 1 配方（旧值 37 未决）；前端 runs 316 持平（第六轮逐字节相同）；三档重算不变，板面缺口 $0.0625 未闭合。执行注记：约 06:07Z 服务重启中断 RA/RB 重试脚本，两腿 nohup 重启跑完；RB 续跑脚本 07:04:30Z 完成，主轮解析 R7 十组扩展文件与 RA 逐项一致；RB cs.AI 范围文件未重取，`rb-analyze1309.py` 因缺文件无法运行。
- 纠正：muse-spark 方向（89 轮摘要「False → True」错误，实为 true→false）；glm-5.1 top ctx 方向（RB 报告文字写反，实为 202,752→204,800）；11-18 技能论文篇数口径（路由轨 all:skill 19 篇为正确值，SA 腿 24 篇系口径混淆，见技能轨）；1017 档位映射（89 轮闭合结论被反例推翻，重新未决）；elo 残差（分项口径统一，1,324 + 7 = 1,331）。
- 裁定：benchmarks 401 编号统一（终值第 113 次，按时点顺延）；qwen 第 71 度记首次观测、未站住（RB「70→71」提案不采纳为站住计数）；2611.23716 非路由、2611.24379 MoE-Context 排除；R7 续记（研究级证据 0→1，生产流量证据仍为零）。
- 剩余未核实缺口：S2 引用跟进余项 1 项（SkillScope）、OpenAlex 余 7 项（第二十六轮连续午夜窗口）；arXiv 11-19 批次筛查（公告档 11-20 01:00Z）；RB 登记集 API 十批与裸式探针未取回；arXiv 裸式区间语法 500 未恢复；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（两正两反两零一非整倍）；glm-5.2 新价是否站住；1017 档位映射重新未决；auto MCT 振荡；muse-spark `is_moderated` 成因；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定；llm-d −1 星新增单列未定（88 轮）。
- 下轮起点：先清上述缺口（S2/OpenAlex 贴近 UTC 午夜执行为首项；11-19 批次筛查；RB 登记集十批与裸式探针补取；glm-5.2 与 1017 复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 16:09 轮（第 91 轮）

- 本轮增量（R-91-01～16，16 条 → api-routing-rounds.md）：计数器第五十四轮闭合 2,357,167（+0，14 卡全部零变动；两腿逐卡一致；主轮自两份页文件独立重算求和一致）；benchmarks API 401 统一终值第 117 次（RA 08:12:56/58Z 为第 114/115 次，RB 08:13:02/04Z 为第 116/117 次；四份证据文件均为 67 字节、同 md5）；qwen 71 度未站住确认（三读均为 62 度值）；1017 档位映射双向反例（90 轮「重新未决」推翻，改为「双向反例，映射关系不成立」）；glm-5.2 异常价目终结（撤回至 88 轮旧价，记价格回归）；gemma-4-26b $9e-8 轮换回归（主轮复核 5 份既往 models.json，RA「首次观测」不成立）；0731 prompt +30% 与 ~flash-latest 分化（90 轮「联动」不再成立）；站住轮次推进（glm-5.1 第二轮、deepseek-v4.1-flash 第三轮、r1-0528 第八轮、kimi-k2.7 第六轮、mimo 第九轮）；elo 连续第二轮零刷新；auto 三件套 MCT 64,000 无振荡；muse-spark `is_moderated` 仍 false（稳定第三轮）；gemma-3-27b-it 缓存字段缺失持续第三轮；glm 差 1 维持；LiteLLM 窗口内两稳定一预发布（v1.106.11、v1.108.12 LTS、v1.111.0-rc.2，均为 GPT-6.2 prompt caching 回灌）；星事件记分 41 → 44（llm-d +2、RouteLLM +1；关窗后 radito3 +1 不计入）；arXiv 90 轮余项全部取回闭合（1117 回归 239、1116 全量 209=209 第二轮闭合、RB 登记集 API 十批全部取回；11-19 批次尚未索引，公告档 11-20 01:00Z；裸式区间语法仍 500）；新直接路由论文 2611.24324《Timeout Cascades》登记；R7 相邻证据 OASIS 2611.24518 与 Cascadeability 2611.23898 登记（全文深读）；AnyS1 直接路由登记（技能轨已登记，不双计）；登记集 57 → 60 项零新版；JevRSI 2611.22432v2 归属 Harness 轨直接登记，路由相邻候选撤回。
- 纠正：gemma-4-26b「首次观测」（RB 腿纠正，主轮复核 5 份既往文件）；1017「重新未决」（双向反例推翻）；0731「联动」（分化推翻）；90 轮「SA 24 篇不成立」修正为口径差异（提交日 19 篇 vs 公告批次 24 篇，两数并存）。
- 裁定：benchmarks 401 编号统一（终值第 117 次）；SkillScope S2 引用列表连续第三十四轮无新增（SB 腿 200 取回为准，SA「不得延续」不采纳）；RA 腿 S2 余项（SkillScope）以跨腿取回确认为无变化，闭合；R7 续记（研究级证据仍为 OASIS 1 篇，生产流量证据仍为零）。
- 剩余未核实缺口：arXiv 11-19 批次筛查（公告档 11-20 01:00Z）；裸式区间语法 500 未恢复（约 10 日）；OpenAlex 余 7 项 + 3 项未能再复核（第二十七轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实；glm 差 1 成因；muse-spark `is_moderated` 成因；llm-d forks −1 反向成因仍未定位；simsim-ai-2026 / werister / lianjunli 单列未定；82 轮 RouteLLM +2 单列未定；llm-d −1 星新增单列未定（88 轮）。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；OpenAlex 贴近 UTC 午夜执行；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 19:09 轮（第 92 轮）

- 本轮增量（R-92-01～15，15 条 → api-routing-rounds.md）：计数器第五十五轮闭合 2,358,919（+1,752，唯一变动 VGI-Bench 308,730 → 310,482；两腿逐卡一致；主轮自两份页文件独立重算求和一致；VGI +1,752 非 439 整倍数，439 倍数假说第二例非整倍反例）；benchmarks API 401 统一终值第 121 次（RA 11:10:35/36Z 为第 118/119 次，RB 11:11:39/45Z 为第 120/121 次；四份证据文件均为 67 字节、同 md5）；qwen 62 度值、glm-5.2 回归值、gemma-4-26b $9e-8 档各站住第二轮；0731 prompt $6.5e-8 → $6.2e-8，与 ~flash-latest 继续分化；1017 档位映射第三反例（与 90 轮同方向、与 91 轮相反；顶层价五轮逐轮交替，仅记观测）；elo 连续第三轮零刷新；auto 三件套 MCT 64,000；muse-spark `is_moderated` 仍 false（第四轮）；gemma-3-27b-it 缓存字段缺失持续第四轮；glm 差 1 维持；目录 480 持平；星事件记分 44 → 46（llm-d +2 配对、forks +1 配对；RouteLLM −1 窗口内 0 事件单列未定）；LiteLLM 窗口内 0 新版（v1.111.0-rc.2 等三版均在 91 轮窗口内，主轮复核 Releases API 确认）；Dynamo 文档表 nightly 行滚动至 dev20261119，仍缺 v1.6.1/v1.6.2 行；decisions 仍 1 个配方、旧值 37 未决；板面 $6.43 缺口 $0.0625 未闭合，板面 "last run" 一处翻新位于 VGI-Bench 节内、与 VGI +1,752 同向（首次观测，严格逐卡归属未完成）；前端 runs 316、文件连续第八轮逐字节相同；R7 十组筛查零新增，生产级联证据仍仅 CascadeRec + HIRA，在线学习阈值真实流量一手证据仍为零；S2 余项 3 项，OpenAlex 余 7 项 + 3 项未能再复核，第二十八轮连续错过午夜窗口；S2 SkillScope 余项闭合（RB 腿第三轮 200 取回 → 11，无变化）；arXiv 裸式区间语法仍 500（自 79 轮起约 10 日）；11-19 批次仍未索引（公告档 11-20 01:00Z），11-18 批次零迟入；登记集维持 60 项零新版，两腿登记集脚本口径不一致记为未决小口径。
- 纠正：RB 腿“91 轮 57 项为旧口径”不采纳（57 是 RA 文件在 91 轮加项前的计数）；RB 腿星事件替代构成（36/10）不采纳（配对规则只认窗口内事件）；radito3 的星已含在 91 轮读数内，不得重复计入。
- 裁定：benchmarks 401 编号统一（终值第 121 次，按时点顺延）；星事件记分 44 → 46（配对规则）；1017 第三反例（仅记观测，不立新映射）；登记集维持 60 项（RA 文件为准，RB 71 项差异为脚本口径差）；R7 续记（研究级证据仍为 OASIS 1 篇，生产流量证据仍为零）。
- 剩余未核实缺口：arXiv 11-19 批次筛查（公告档 11-20 01:00Z）；裸式区间语法 500 未恢复（约 10 日）；OpenAlex 余 7 项 + 3 项未能再复核（第二十八轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动（新增 VGI 节内翻新观测，严格逐卡归属未完成）；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实；glm 差 1 成因；muse-spark `is_moderated` 成因；RouteLLM −1 单列未定；登记集两腿脚本口径不一致（下轮统一）。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；OpenAlex 贴近 UTC 午夜执行；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-19 22:09 轮（第 93 轮）

- 本轮增量（R-93-01～17，17 条 → api-routing-rounds.md）：计数器第五十六轮闭合 2,360,215（+1,296，唯一变动 VGI-Bench 310,482 → 311,778；主轮自 RA 证据文件独立重算求和一致；+1,296 非 439 整倍数，439 倍数假说第三例非整倍反例）；benchmarks API 401 统一终值第 125 次（RA 14:13:24/25Z 为第 122/123 次，RB 14:13:47/50Z 为第 124/125 次；四份证据文件均为 67 字节、同 md5）；1017 档位映射第四反例（与 90/92 轮同方向；顶层价五轮逐轮交替终结，仅记观测）；deepseek-v4.1-flash 再破新低 $5.07e-8/$2.028e-7/$1.014e-9（恰为 85 轮低价原值回落，MCT 943,718 → 393,216 成对回落），~flash-latest 同步新低 $4.664e-8/$1.3992e-7/$1.484e-9，0731 prompt $6.2e-8 不变站住第二轮、与 latest 进一步分化；qwen3.5-9b prompt/completion $1.7e-7/$2.5e-7 站住第三轮，但 input_cache_read 字段消失，62 度值作为三档整体不再成立；kimi-k2.7-code $6e-7 → $6.5e-7（+8.3%），七轮站住终结；elo 大刷新（680 格数值变化、175 个模型、新增 2 格），92 轮零刷新终结，目录 480 → 479；星事件记分 46 → 49（llm-d +2 配对、RouteLLM +1 配对，RB 腿补取事件流成功、主轮一手复核）；R7 十组筛查零新增，生产级联证据仍仅 CascadeRec + HIRA，在线学习阈值真实流量一手证据仍为零；S2 余 3 项、OpenAlex 十项全部 429，第二十九轮连续错过午夜窗口，22238/22243 零引用连续第二轮确认、EfficientSkill 零引用首次确认；登记集维持 60 项零新版，92 轮未结第 24 项定位闭合（登记 53 项两腿一致，差异仅候选附录）；Dynamo 文档表 llms.txt 链接迁至 docs.dynamo.nvidia.com（第三次翻动），仍缺 v1.6.1/v1.6.2 行；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；LiteLLM 窗口内零新发布；arXiv 全量枚举连续第四轮闭合（1116 cs.AI 209=209），11-19 批次仍未索引（公告档 11-20 01:00Z），裸式区间语法仍 500；板面 "last run" 一处翻新至 12:25:26.379Z 位于 VGI-Bench 节内（连续第二轮与 VGI 增量同向），$6.43 缺口 $0.0625 未闭合，decisions 仍 1 个配方、旧值 37 未决；前端 runs 316、文件连续第九轮逐字节相同；glm-5.2 回归值站住第三轮，gemma-4-26b-a4b-it $9e-8 档站住第三轮，glm-5.1 站住第四轮，r1-0528 第十轮、mimo 第十一轮，auto 三件套 MCT 64,000 连续第四轮低档，muse-spark web_search $0.01 → $0.0025 且 is_moderated 翻回 true。
- 纠正：89 轮“deepseek-v4.1-flash 新低已终结”不再成立；92 轮“elo 零刷新”终结；92 轮“1017 五轮逐轮交替”终结；92 轮“qwen 62 度值站住第二轮”不再成立（缓存档缺失）；92 轮“kimi-k2.7-code 站住第七轮”终结；92 轮引用脚本笔误 2611.15911 → 2611.15921 已纠正（92 轮结论不受影响）。
- 裁定：星事件记分采纳 RB 腿配对证据（46 → 49）；401 终值按取回时刻顺延为 125；登记集维持 RA 文件 60 项（RB-only 11 项记候选附录）；RB 腿“~flash-latest 与 0731 同向回落”驳回（0731 本轮未动）；“last run”翻新 RA/RB 两说一致（数值变、日期分布不变）。
- 剩余未核实缺口：arXiv 11-19 批次筛查（公告档 11-20 01:00Z）；裸式区间语法 500 未恢复（约 10 日）；S2 余 3 项、OpenAlex 余 7 项 + 3 项未能再复核（第二十九轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计未联动（VGI 节内翻新连续第二轮同向，严格逐卡归属未完成）；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（第三例非整倍反例）；glm 差 1 成因；RouteLLM −1（92 轮）单列未定。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；OpenAlex 贴近 UTC 午夜执行；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20 04:09 轮（第 94 轮）

- 本轮增量（R-94-01～17，17 条 → api-routing-rounds.md）：计数器第五十七轮闭合 2,363,224（+3,009；Model Routers +421、GPQA +398、VGI +2,190；主轮自两份页文件独立重算求和一致）；benchmarks API 401 统一终值第 133 次（八份证据文件均为 67 字节、同 md5）；前端 runs 316 → 320（九轮逐字节相同终结），板面 "last run" 最新值 19:56:29.513Z 与前端统计最新 run 时间戳逐字相同（首次联动观测）；定价变动 8 个模型（deepseek-v4-pro-0813 两步上调、gemma-4-26b-a4b-it $9e-8 档终结、minimax/hy3/hy4 顶层价与段位不一致新观测、~flash-latest completion ×3.33）；1017 档位映射连续两轮一致（定性「未证实亦未证伪」）；qwen 缓存字段缺失连续第二轮；星事件记分 49 → 52（93 轮 RouteLLM +1 由本腿证据独立确认）；vLLM v0.34.1 两批 404 确认不存在；arXiv 全量枚举连续第五轮闭合，11-19 批次本轮未筛查；S2 Conformal Cascade 引用 5 → 2 下修；登记集维持 60 项；三人历史单列关闭追踪。
- 纠正：RB 腿「qwen 缓存字段缺失持续第三轮」不成立（实为第二轮）；93 轮 RouteLLM +1 配对独立确认（StringManolo 12:26:33Z）；前端统计九轮逐字节相同终结；Dynamo 域名迁移仅存活一轮即回退；1118 标题顺序抖动——位置比对不可靠，以 id 集为准。
- 裁定：401 终值按取回时刻顺延为 133（RB 腿「126–129」为腿内编号，纠正为交错顺序）；计数器关账值取终扫 2,363,224；VGI 前半 +1,312 第四例非整倍反例、后半 +878 = 439×2 首个整倍数据点，439 倍数假说仍未证实；1017 定性采纳 RB 措辞「连续两轮一致，未证实亦未证伪」；minimax/hy3/hy4 段位不一致列为新观测类、下轮复核；三人历史单列关闭追踪；11-19 批次留待 95 轮；S2 引用 5 → 2 下修采纳，旧值 5 作废；板面 "last run" 首次联动观测采纳（PRACTICAL.md 板面条目续记）。
- 剩余未核实缺口：arXiv 11-19 批次筛查（公告档 11-20 01:00Z）；裸式区间语法 500 未恢复（约 10 日）；S2 余 2 项、OpenAlex 余 10 项（第三十轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计首次联动观测（严格逐卡归属仍未完成）；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（第四例非整倍反例 + 首个整倍数据点）；minimax/hy3/hy4 段位不一致成因；deepseek-v4-pro-0813 两步上调、~flash-latest completion ×3.33、gemma-4-26b-a4b-it $7.65e-8 档是否站住。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；OpenAlex 贴近 UTC 午夜执行；1017 档位映射与 minimax/hy3/hy4 段位复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20 07:09 轮（第 95 轮）

- 本轮增量（R-95-01～15，15 条 → api-routing-rounds.md）：计数器第五十八轮闭合 2,363,763（+539；Model Routers +12、GPQA −4、VGI +429、τ²-Bench Airline +102；主轮自两份页文件独立重算求和一致；VGI +429 第五例非整倍反例，439 倍数假说仍未证实）；benchmarks API 401 统一终值第 137 次（RA 四次取回，四份证据文件均为 67 字节、同 md5）；前端 runs 320 → 325（窗口内 +4 全为 google/gemini-4-neon），三档重算 $1.6427/$6.3675/$0.5757 分毫不差；板面 "last run" 第 9 处由 Nov 18 移至 Nov 19；定价变动 3 个模型（deepseek-v4-pro-0813 第三步上调、deepseek-v4-flash-0731 腿内三度变价终值 prompt $1e-7、deepseek-v4-pro 仅缓存 −20%）；1017 档位映射本轮两读一致（四反例仍在）；minimax/hy3/hy4 段位不一致连续第四读；gpt-6.2-astra `is_moderated` 翻回 true；auto 三件套 MCT 128,000 站住；qwen 缓存字段缺失连续第三轮、gemma-3-27b-it 第七轮；星事件记分 52 → 53（llm-d +1 配对）；五项目窗口内 0 新版（vLLM v0.34.1 第三批 404 确认不存在，未结项关闭）；LiteLLM 博客 RSS 180 → 181（窗口内新文数据库课程）；arXiv 全量枚举连续第六轮闭合，11-19 批次本轮未筛查；裸式区间语法 500/503 交替未恢复；R7 十组筛查全部零新增；登记集维持 60 项零新版；S2 余 3 项九次 429（第三十一轮连续错过午夜窗口），OpenAlex 十二项全部取回、唯一变化 BaRP 22 → 23。
- 纠正：94 轮「裸式四次探针仍 500」修正为 500/503 交替、未恢复；94 轮「LiteLLM 博客迟入 2 篇」须区分窗口内新发与迟入；SA 腿「Gemini CLI 窗口内零新发布」不成立（SB 文件 + Harness npm + 主轮 npm 直取三方一致）。
- 裁定：401 终值按取回时刻顺延为 137；计数器关账值取终扫 2,363,763；1017 定性维持「两读一致，四反例仍在，不得恢复映射关系」；minimax/hy3/hy4 段位不一致列为持续观测类、下轮复核；11-19 批次留待 96 轮；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）。
- 剩余未核实缺口：arXiv 11-19 批次筛查（公告档 11-20 01:00Z，已过；翻页后完整枚举）；裸式区间语法 500/503 交替未恢复（约 10 日）；S2 余 2 项、OpenAlex 余 10 项（第三十一轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 "last run" 与前端统计联动观测（严格逐卡归属仍未完成）；$6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 继续暂定（生产级联证据仍仅 CascadeRec + HIRA）；VGI 增量 439 倍数假说未证实（第五例非整倍反例）；minimax/hy3/hy4 段位不一致成因未定位。
- 下轮起点：先清上述缺口（11-19 批次筛查为首项；OpenAlex 贴近 UTC 午夜执行；1017 档位映射与 minimax/hy3/hy4 段位复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20 10:09 轮（第 96 轮）

- 本轮增量（R-96-01～16，16 条 → api-routing-rounds.md）：计数器第五十九轮闭合 2,364,192（+429；VGI +438 唯一变动、Model Routers −9 持续；主轮自两份页文件逐卡重算求和一致）；前端 runs 325 → 326，三档重算分毫不差；定价目录 477 → 478（新增 pixtral-large-2502）；1017 腿内翻转（首读删除、终扫恢复高档一致态，档位映射仍不可靠；主轮直查模型文件确认）；elo 大刷新（up 292/down 1210/added 15/removed 0）；星事件记分 53 → 55（llm-d +2 配对）；401 统一终值第 138–141 次；版本首验三件套窗口内上架（langchain-openai 1.8.5、langchain-google-genai 4.5.2、google-adk 2.15.0）；LiteLLM 新预发布 v1.112.0-dev.1；LoopRoute 2611.25363 新直接路由论文；FinVeriQA 2611.25071 直接路由候选、InflEns 2611.25167 相邻证据；登记集仅 2611.22432 v2 → v3；R7 十组筛查两组各 +1，R7 续记；S2 Route Anything 首次取回 citationCount 0，其余全部 429（第三十二轮连续错过午夜窗口）；Jev 指南渲染变完整、实验旗翻回 control；LiteLLM 博客 feed.xml 新 404；CVE 两项厂商逐字确认仍缺；decisions 仍 1 配方、旧值 37 未决；裸式区间语法仍 500 未恢复。
- 纠正：91 轮「11-18 批次 1,304 条闭合」推翻（本轮 1,624 条，+320 迟入）；95 轮「11-18 memory 查询 removed 2611.23930/2611.24503」作废（截断假象）；1116「209=209 连续六轮闭合」终结（95 轮读数当时正确，不作 95 轮错误）；95 轮「裸式 500/503 交替」修正为持续 500（503 仅第二发一例）。
- 裁定：401 终值按取回时刻顺延为 138–141；计数器关账值取终扫 2,364,192；1017 定性为「腿内翻转，档位映射仍不可靠」；11-19 批次口径按取回时刻记录（M「闭合」与 H/RB「未闭合」并存，附迟入风险注记）；R7 续记（LoopRoute/FinVeriQA 均为固定阈值）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；裸式区间语法持续 500 未恢复（约 11 日）；S2 余 3 项、OpenAlex 余 10 项（第三十二轮连续错过午夜窗口）；decisions 新计数方法；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；LiteLLM feed.xml 是否恢复或改址；minimax/hy3/hy4 段位不一致成因；1017 档位映射复核。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；OpenAlex 贴近 UTC 午夜执行；1017 档位映射与 minimax/hy3/hy4 段位复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20 13:09 轮（第 97 轮）

- 本轮增量（R-97-01～12，12 条 → api-routing-rounds.md）：计数器第六十轮闭合 2,364,413（+221，Model Routers +221 唯一变动）；benchmarks API 401 统一终值第 142–145 次；1017 顶层两读均为低档值，档位映射仍不可靠（连续第三轮不一致）；定价变动 6 个模型（目录 478 → 477）；LiteLLM 三新稳定版窗口内发布（v1.110.2/v1.109.3/v1.108.13）；登记集 60 → 63（ADAPT 2611.25754 全文深读、LoopRoute、FinVeriQA 直接；InflEns 相邻）；星事件记分 55 → 57；前端 runs 326 → 329；11-19 cs.AI totalResults 202 取回；11-19/1118 迟入复核未完成（18/32 恢复）；1113 全套闭合；裸式探针仍 500；R7 十组全部取回（prefix-cache 5 → 6、screen-llm-routing 7 → 8）。
- 纠正：96 轮「11-19 批次 192 条全量枚举」订正为 cs.AI totalResults 202（96 轮 29 篇为部分索引读数）；任务书「11-19 批次 96 轮 192 条全量枚举」说法订正（96 轮落盘文件实为 cs.AI 29 篇 / skill 4 篇，复核以 29/4 为基线）；96 轮「11-19 批次 skill 4 篇」订正为提交日 24 篇（筛查完成）。
- 裁定：401 终值按取回时刻顺延为 142–145；计数器关账值取终扫 2,364,413；1017 定性为「顶层两读均为低档值，档位映射仍不可靠」；11-19 批次计数按查询口径分别记录（cs.AI 202；`all:skill` 提交日 24 篇）；R7 读数以 RA 腿为准（RB 全部 429）；ADAPT/LoopRoute/FinVeriQA 登记为直接路由，InflEns 为相邻；2611.09892 v3 不先行登记，下轮重筛；arXiv export API 全灭新登未结台账。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；RB `rb97-resume.py` 后台补取的收割（先读 `rb-fetch-arxiv-resume.log`）；裸式区间语法 500 是否持续（RB 两次探针均为 429，无法判定；RA 1113 探针仍 500 已确认）；S2 余 2 项、OpenAlex 余 12 项（第三十三轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；LiteLLM feed.xml 是否恢复或改址；R7 暂定续记。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；RB 补取收割；1017 档位映射与 minimax/hy3/hy4 段位复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20 16:09 轮（第 98 轮）

- 本轮增量：计数器第六十一轮闭合 2,362,497（−1,916；Model Routers −1、GPQA −1,915 成因未定位）；benchmarks API 401 统一终值第 146–149 次；1017 顶层翻回高档 $3e-7，方向反转仍不一致（连续第四轮）；LiteLLM 窗口内两个新版本（v1.106.12 稳定版、v1.111.0-rc.3 预发布）；前端 runs 329 → 330，三档重算分毫不差；登记集 63 → 64（FrontierReplay 2611.26020 相邻）；星事件记分 57 → 58（llm-d +1 配对）；11-19/11-18 迟入复核闭合（cs.AI 202/205）。
- 纠正：94 轮 delta 摘要行「11+6+8=14」算术错误，正确分解为 336 = 322 + 6 + 8（见技能轨）；RA cost-cascade 文件截断定位（RA 97 轮文件仅存 10/16 条，非新论文）；arXiv 裸式区间语法 500 本轮未判定（RA 探针用了方括号写法），下轮以裸式重探。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA）；FrontierReplay 登记为相邻；2611.09892 v3、2611.22432 v3 均仍不登记；llm-d pushed_at 同一值不重复计入；SkillEvaluator stars 关账取最晚读数 4,184。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；RB 余项 27 项未取回；裸式区间语法重探（裸式）；S2 余 2 项、OpenAlex 余 12 项（第三十四轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；LiteLLM feed.xml 是否恢复或改址；R7 暂定续记；GPQA −1,915 成因未定位。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；RB 余项 27 项；裸式重探；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20（19:09 轮，第 99 轮）
- 增量：计数器 2,293,610（−68,887；GPQA −56,497、τ² −13,707 无事件下行，累计 −58,412 成因未定位；VGI +1,317 唯一有 run 支撑）；1017 第五轮不一致；Railway 最终报告已发布（缺口闭合）；R7 十组零变化（FrontierReplay 不是第三项证据；cost-cascade 截断闭合）；星事件记分 58 → 59（严格配对）；97 轮余项历史 14 项划除。
- 纠正：98 轮「Railway 最终报告仍未发布」作废；RB 腿查询下限误用已纠正重取（误用读数作废）；RA 腿分析文件节标题标签遗漏（数值无误）。
- 裁定：R7 仍暂定；星事件记分取严格配对口径（净增 60 不采纳）；97 轮余项历史 14 项划除，1120-2/1120-full 留待公告后重做。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；1120-2/1120-full；GPQA 累计 −58,412 成因未定位；S2 余 2 项、OpenAlex 余 12 项（第三十五轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；LiteLLM feed.xml 是否恢复或改址；R7 暂定续记。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1120-2/1120-full；GPQA 成因；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-20（22:09 轮，第 100 轮）
- 增量：计数器 2,295,546（+1,936；GPQA −1,973、VGI +3,909、τ² 止跌）；GPQA 累计下行成因已定位（批量移除历史 run，runCount −309，关闭 99 轮缺口）；1017 第六轮不一致；R7 十组维持 RB 口径 0/3/2/6/16/6/5/6/8/10（主轮裁定）；401 统一终值第 154–157 次；登记集 77 项零升版（09892 v3 / 22432 v3 全文深读均不支持登记）；ADAPT 仅落入 llm-routing 组。
- 纠正：RA 腿「8 位区间上界写法已失效」不成立（RB 同轮统一 8 位取回正常；真正缺陷是混合精度区间返回 0，主轮独立探针复现）；RA 腿「99 轮 R7 三组已漏计」不成立（上修来自下界放宽，新增论文均为早已登记的旧论文）；「GPQA 累计下行无事件、成因未定位」被批量移除定位取代（前端统计端点不含 GPQA run 的盲区表述一并修正）；99 轮「11-20 批次待筛查」口径不变（公告档 11-21 01:00Z）。
- 裁定：R7 计数维持 RB 口径；下轮起 R7 查询一律统一精度、禁用混合精度写法；R7 仍暂定；星事件记分本轮无变动（RA 腿未单列星事件条目，记分维持 59）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；1120-2/1120-full；S2 余 2 项、OpenAlex 余 12 项（第三十六轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；LiteLLM feed.xml 是否恢复或改址；R7 暂定续记；裸式区间语法仍 500（约 13 日）。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1120-2/1120-full；R7 统一精度查询；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（01:09 轮，第 101 轮）
- 增量：计数器 2,277,645（−17,901；GPQA −6,750、VGI +439、τ² −11,590）；**τ² 首次观测到与 GPQA 同机制的批量移除**（runCount −233，modelCount −1）；1017 第七轮复核一致（中断连续六轮不一致）；minimax/hy3/hy4 三模型同时翻转至另一时段值（overrides 未变，成因未定位）；**LiteLLM feed.xml 改址已确认**（301 链，仍未以 feed 形式恢复；关闭 100 轮缺口）；401 统一终值第 158–161 次；R7 十组维持 RB 口径 0/3/2/6/16/6/5/6/8/10；S2 EfficientSkill citationCount 0 闭合（RB 腿首次直接取回确认），引用跟进余项 14 → 13；登记集 77 项零升版。
- 纠正：H 腿「2611.25891 建议直接登记」为重复（98 轮已登记、99 轮确认）；R-101-11 初稿来源路径写法勘误（十组文件命名，见 verification.md 末节）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零）；1017 单轮一致不足以恢复为既定关系，下轮继续复核；星事件记分维持 59。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；1120-2/1120-full；S2 余 1 项、OpenAlex 余 12 项（第三十七轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 暂定续记；裸式区间语法仍 500（约 13 日）；minimax/hy3/hy4 翻转成因未定位。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1120-2/1120-full；minimax/hy3/hy4 翻转成因；1017 档位映射复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（04:09 轮，第 102 轮）
- 增量：计数器 2,229,658（−47,987；GPQA −48,476、VGI +439、τ² +50）；**1017 档位问题终结**（overrides 删除，三读新平价 $3.6e-8/$5e-7/$1e-8，MCT 393,216 → 943,718，真实重新定价）；minimax/hy3/hy4 翻转状态站住（overrides 未变，成因仍未定位）；定价目录 477 → 476（tencent/hy3-preview 移除），8 模型变价；401 统一终值第 162–165 次；**LiteLLM 博客新文首次观测**（GPT-6.2 不再缓存 instructions 字段，厂商自述）；R7 十组 + 四组扩展零变化（RB 口径 0/3/2/6/16/6/5/6/8/10）；S2 SkillSeek 与 SkillScope 各闭合一项 101 轮余项，余项 distinct 17 项全部 429（第三十八轮连续错过午夜窗口）；登记集 77 项零升版；星事件记分 59 → 61。
- 纠正：101 轮「1017 第七轮一致」失去复核对象（overrides 删除）；RA 腿方法事故两处（sed 误伤、手动重取截断）已以修正脚本全量重跑为准。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零）；1017 下轮只复核新价是否站住，不再按档位复核。
- 剩余未核实缺口：arXiv 11-20 批次筛查（公告档 11-21 01:00Z）；1120-2/1120-full；S2/OpenAlex distinct 17 项（第三十八轮连续错过午夜窗口）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 暂定续记；裸式区间语法仍 500（约 14 日）；minimax/hy3/hy4 翻转成因未定位；GPQA 新 run 与被移除量拆分不可追踪。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1120-2/1120-full；minimax/hy3/hy4 翻转成因；1017 新价复核），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（07:09 轮，第 103 轮）

- 增量：计数器 2,229,658（±0，第六十六轮闭合；GPQA 腿内 −198 后约 41 分钟全部回吐）；1017 新平价站住（overrides 仍 null，MCT 943,718）；minimax/hy3/hy4 翻转状态站住（第三轮）；定价目录 476 不变，2 模型变价；401 统一终值第 166–169 次；**引用跟进午夜窗口局面被打破**（15/17 取回，仅余 S2 Synapse 与 S2 SkillScope 论文端点；RouteBalance 唯一引用者 DEALS 已定位；SkillLearnBench S2 66→68）；R7 十组 + 四组扩展零变化（RB 口径 0/3/2/6/16/6/5/6/8/10）；登记集 77 项零升版；星事件记分 61 → 63；CascadeRec 数字一手复核通过（无新证据）。
- 纠正：arXiv「公告档 11-21 01:00Z」写法对周五批次不成立（11-20 为周五，周五、周六无公告；最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z）；「第三十九轮连续错过午夜窗口」口径本轮终结（OpenAlex 午夜前全 429 记为第三十九轮即最后一次）；窗口后活读 1017 已改价与目录 476→477 记为 104 轮复核事项，不推翻 103 关账。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零）；1017 下轮复核新价是否站住（窗口后已改价）；LiteLLM v1.108.14 归属下轮窗口。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告）；1120-2/1120-full（留待周二 00:00Z 之后重做）；S2 余 2 项（S2 Synapse、S2 SkillScope 论文端点）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 暂定续记；裸式区间语法仍 500（约 14 日）；minimax/hy3/hy4 翻转成因未定位；GPQA −198 回吐成因未定位；1017 改价复核。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1017 改价复核；S2 余 2 项；minimax/hy3/hy4 翻转成因），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（10:09 轮，第 104 轮）

- 增量：计数器 2,228,203（−1,455，第六十七轮闭合；GPQA −1,178、τ² −250、Model Routers −27；439 倍数假说第十四轮检验失败；103 轮 −198「全部回吐」未站住）；**1017 新平价仅站住一轮即翻回 overrides 定价**（103 轮「新平价站住」被推翻；minimax/hy3/hy4 同时翻回段位一致态；deepseek-v4.1-flash 降至 $5.07e-8）；定价目录 476 → 477（daisyrose-27b 新增；inkling-2-mini 未确认闪烁）；**LiteLLM v1.108.14 LTS 窗口内发布**（published 2026-11-20T23:43:31Z，3 PRs）；401 统一终值第 170–173 次；前端 runs 330 → 336；**引用跟进 Synapse 首次直接取回**（S2 7/10 取回，余 3 项；OA 12 项全部 429；「第 N 轮」编号不再续编）；R7 十组 + 四组扩展零变化（RB 口径 0/3/2/6/16/6/5/6/8/10），v3 全文 diff 两篇归一化后逐字节相同；**星事件记分维持 63**（llm-d 配对未建立，updated_at 02:16:02Z 无对应事件）。
- 纠正：RA 腿 Conformal「103 轮未取回项本轮取回…闭合」→ 再确认；RA 腿「S2 余 4 项」→ 余 3 项；RB 腿「56 次尝试、0 取回」仅为其自身窗口；SB 腿「OpenAlex 第四十轮」→ 不续编号；H 腿 C-2「11-21 01:00Z 预期」归属 103 轮不成立（属 97–102 轮）；103 轮「新平价站住」被推翻；裸式 500 约 14 日，103 轮保留意见本轮解除。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零）；1017 下轮复核新价是否站住（已翻回 overrides）；llm-d 星/forks +1/+1 记为未定位缺口。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告）；1120-2/1120-full（留待周二 00:00Z 之后重做）；S2 余 3 项 + OpenAlex 12 项全部 429；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 暂定续记；裸式区间语法仍 500（约 14 日）；minimax/hy3/hy4 翻转成因未定位（103 轮翻转仅站住一轮）；GPQA −1,178 与 Model Routers −27 成因；1017 改价复核。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1017 改价复核；S2 余 3 项；minimax/hy3/hy4 翻转成因），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（13:09 轮，第 105 轮）

- 增量：计数器 2,228,077（−126，第六十八轮闭合；GPQA −154、τ² −50、Model Routers +78；439 倍数假说第十五轮检验失败；GPQA 自 97 轮关账累计 −116,943）；**1017 overrides 定价站住第二轮**（deepseek-v4.1-flash 再下调至 $4.98e-8；minimax/hy3/hy4 段位一致站住第二轮）；定价目录 477 → 476（inkling-2-mini 正式上架确认；移除 dots-3-note-preview:free 与 longcat-2.0）；**S2 余项全部闭合**（Continual 13、SkillLearnBench 68 真实 200 取回，paperId 与 103 轮一致；OA 12 项仍全部 429）；**RouteBalance 第二引用者定位**（《Mind the Delay》v2 加入引用）；前端 runs 336 → 345；401 统一终值第 174–177 次；窗口内新发布 0 个；裸式区间语法仍 500（RB 腿干净读数）；**星事件记分维持 63**（llm-d 零变动；104 轮 +1/+1 仍未定位，第二轮未结）。
- 纠正：103 轮「唯一引用者 DEALS」在 S2 口径下不成立（新增《Mind the Delay》v2 引用）；104 轮口径值 0/3/2/6/16/6/5/6/8/10 不得作为 105 轮读数沿用（R7 本轮未复核）；RB/SB 腿「S2 全部 429 未取回」为取证时段差异（以 RA 腿成功取回为准）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮未复核）；S2 余项全部闭合；RouteBalance 引用者订正记为修正。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告）；1120-2/1120-full（留待周二 00:00Z 之后重做）；OA 12 项全部 429；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE-2026-114612 / Portkey CVE-2026-110719 厂商逐字确认仍缺；R7 暂定续记；裸式区间语法仍 500；minimax/hy3/hy4 与 1017 翻转成因未定位；GPQA −154 与 Model Routers +78 成因；1017 改价复核（是否站住第三轮）；llm-d 104 轮 +1/+1 配对（第三轮未结）。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1017 改价复核；OA 12 项重取；minimax/hy3/hy4 翻转成因），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（16:09 轮，第 106 轮）

- 增量：计数器 2,228,077（±0，第六十九轮闭合；GPQA 自 97 轮关账累计 −116,943 不变；439 倍数假说第十六轮检验失败；105 轮 GPQA −154 与 Model Routers +78 成因仍未定位）；**1017 overrides 定价站住第三轮**（deepseek-v4.1-flash $4.98e-8 站住第二轮；minimax/hy3/hy4 段位一致站住第三轮）；定价目录 476 不变、零变价、elo 零刷新；前端 runs 345（±0）；401 统一终值第 178–181 次；窗口内新发布 0 个；裸式区间语法仍 500（两腿干净读数一致，913 B）；**R7 十组一手取回全部完成**（0/3/2/6/16/6/5/6/8/10；RB 腿 14/14 一次 200；RA 腿独立取回一致；105 轮「104 轮值不得沿用」的要求已满足）；登记集 77 项零升版；**S2 取回 7 项 distinct**，余 S2 3 项 + OA 12 项全部 429（retryAfter 约 55,895–56,430 秒）；RouteBalance 引用者仍 2 项；**星事件记分维持 63**（llm-d +1/+1 配对第三轮未结）。
- 纠正：105 轮「七项复现不得递增」已终结（本轮属记忆轨，见 track-layered-memory.md）；裸式 500 持续时间两腿冲突（RA 约 15 日 vs RB 约 14 日）按 104 轮口径约 14 日计，delta 草案不写天数；llm-d 未结轮次编号按发现轮计（104 轮第一轮、105 轮第二轮、106 轮第三轮未结）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮已一手复核）；0 结果 feed 判据修正（200 状态 + totalResults=0 须 30 分钟后重取一次再判「缺席」），下轮沿用。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；OA 12 项 + S2 余 3 项重取（约 11-22 00:00Z 重置）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE 两项厂商逐字确认仍缺；minimax/hy3/hy4 与 1017 翻转成因未定位；GPQA −154 与 Model Routers +78 成因；1017 改价复核（是否站住第四轮）；llm-d 104 轮 +1/+1 配对（第四轮未结）。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1017 改价复核；OA 12 项重取；minimax/hy3/hy4 翻转成因），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（19:09 轮，第 107 轮）

- 增量：计数器 2,228,516（+439，第七十轮闭合；唯一变动 VGI-Bench +439，定位至窗口内一次 439 题新 run；GPQA 自 97 轮关账累计 −116,943 不变；439 倍数假说第十七轮检验：净变动首个非零整倍数数据点，但假说仍不成立）；**1017 overrides 定价站住第四轮**（deepseek-v4.1-flash $4.98e-8 站住第三轮；minimax/hy3/hy4 段位一致站住第四轮）；定价目录 476 不变、零增减、elo 零刷新；唯一变价 z-ai/glm-5.1；前端 runs 345（±0）；401 统一终值第 182–185 次；**窗口内新发布 0 个**（GitHub 未认证限流事故：RA 腿 releases 列表三轮未能取回，RB 腿重试后取回；v1.108.15 四取全 403 未证实，三重交叉证据均无其踪影）；裸式区间语法仍 500（913 B，两次干净读数）；**R7 十组零变化**（0/3/2/6/16/6/5/6/8/10；零组 30 分钟后迟取仍 0，缺席成立）；登记集 77 项零升版；**S2 取回 4 项 distinct**，余 S2 3 项 + OA 12 项全部 429（约 2026-11-22 00:00Z 重置）；RouteBalance 引用者仍 2 项；**llm-d 新星配对成立，星事件记分 63 → 64**（WatchEvent 08:27:45Z → updated_at 08:27:46Z，Δ=1 秒；**104 轮 +1/+1 配对结案为未定位**，第四轮未结，原始证据已灭失，下轮起不再续计）。
- 纠正：无（本轮无旧说法推翻；SA 腿「216 个 slug」修正提议已驳回，见 track-skills.md）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮已一手复核）；439 倍数假说仍不成立（净变动 +439 仅为观测，不改判）；GitHub 限流下「窗口内新发布 0 个」以 RB 腿重试取回的列表为准；llm-d 104 轮配对结案为未定位（RB 腿「第五轮」订正为第四轮）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；OA 12 项 + S2 余项重取（约 2026-11-22 00:00Z 重置）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE 两项厂商逐字确认仍缺；minimax/hy3/hy4 与 1017 翻转成因未定位；GPQA −154 与 Model Routers +78 成因；1017 改价复核（是否站住第五轮）；v1.110.2 tag、v1.106.12、v1.108.13、rc.6 探查（ra-fetch107.sh 笔误先修）。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；1017 改价复核；OA 12 项重取；minimax/hy3/hy4 翻转成因；v1.110.2 tag 等探查），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-21（22:09 轮，第 108 轮）

- 增量：计数器 2,235,067（+6,551，第七十一轮闭合；唯一变动 VGI-Bench +6,551，定位至 15 次新 run；439 倍数假说第十八轮检验失败）；1017 overrides 站住第五轮（deepseek-v4.1-flash $4.98e-8 站住第四轮；minimax/hy3/hy4 段位一致站住第五轮）；定价目录 476 不变、零变价；elo 大刷新（up 314 / down 844 / 182 模型）；窗口内新发布 0 个；benchmarks API 401 第 186–189 次；R7 十组读数 0/3/2/6/16/6/5/6/8/10，登记集 77 项零升版；S2 5 项 distinct 取回，余项 429 未取回；llm-d pushed_at 移至 12:20:43Z（配对 Δ=1 秒），星事件记分维持 64。
- 纠正：v1.108.15「未证实」→「已证实不存在」（四重证据）；v1.106.12/v1.108.13「未探查」→「已证实存在」（11-20 发布）；OpenCode 搜索端点 422 口径推翻（旧仓名 sst/opencode 所致，执行者亲自复现确认；下轮起用 anomalyco/opencode）；SkillEvaluator 配对计数 SB「无第 40 例」为笔误，本轮第 41 例；llm-d 104 配对 SA「第五轮未结」不采纳（107 轮已结案）。
- 裁定：R7 仍暂定；439 倍数假说仍不成立（第十八轮检验失败）；GPQA −154 与 Model Routers +78 成因仍未定位；minimax/hy3/hy4 与 1017 翻转成因仍未定位。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；OA 12 项 + S2 余项重取（约 2026-11-22 00:00Z 重置）；decisions 新计数方法（旧值 37 未决）；板面 $6.43 缺口 $0.0625 未闭合；CVE 两项厂商逐字确认仍缺；v1.110.2 tag 探查（旁证存在、API 下轮再试）；vLLM releases 列表与 Portkey v1.15.3 tag 探查；GPQA/Model Routers 成因；翻转成因；llm-d PushEvent 小异常。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；OA 12 项重取；1017 改价复核；v1.110.2 tag 探查），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-22（01:09 轮，第 109 轮）

- 增量：计数器 2,237,874（+2,807，第七十二轮闭合；VGI +3,051 定位至 7 次新 run，6 次 439 题、1 次 417 题；GPQA −194，自 97 轮关账累计 −117,137；τ² −50；439 倍数假说第十九轮检验失败）；1017 overrides 站住第六轮（deepseek-v4.1-flash $4.98e-8 站住第五轮；minimax/hy3/hy4 同时再翻、站住链断裂；glm-5.1 改价 prompt/completion ×0.85、缓存 ×2.31；目录 476 不变、elo 零刷新；kimi-k3/inkling-2-mini/~kimi-latest 实时闪烁，不记为已变价）；窗口内新发布 0 个；benchmarks API 401 第 190–193 次；R7 十组 0/3/2/6/16/6/5/6/8/10，登记集 77 项零升版；裸式区间仍 500（第五轮未恢复）。
- 纠正：RA 腿「GPQA +1,806／首次上行／累计 −115,137」叙述驳回（与其自身原始文件 ra-count109.out.txt 冲突，GPQA 实为 1,476,157，以原始文件为准）；RB 腿 inkling-2-mini 与 ~kimi-latest「已变价」降级（8 份取回仅 2 份新值，实时闪烁）；v1.110.2「旁证存在」→「已证实存在」（GitHub API 200，published 2026-11-20T04:06:19Z）；任务书引用的 43 轮批文件路径勘误（refined/20261113-0409/delta.md 实为第 40 轮，正确文件为 refined/20261113-1309/delta.md）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮已一手复核）；439 倍数假说仍不成立（第十九轮检验失败）；GPQA −154 与 Model Routers +78 成因仍未定位；minimax/hy3/hy4 与 1017 翻转成因仍未定位（本轮再翻）；segment 评估机制（时钟偏移 vs top 价字段独立更新）仍未区分。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；OA 12 项 + S2 余项重取（约 2026-11-22 00:00Z 重置）；S2 Conformal、ExProt、SkillScope 论文端点重取；llm-d 108 轮 main HEAD（234d1c78）PushEvent 仍未定位；decisions 计数方法仍未决（旧值 37）；qwen3.5-9b input_cache_read 字段缺失已第十七轮。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；OA 12 项重取；1017 改价复核；minimax/hy3/hy4 翻转成因），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-22（04:09 轮，第 110 轮）

- 增量：计数器 2,238,506（+632，第七十三轮闭合；VGI +871 定位至 2 次新 run（轮间到达，lastRun 均在窗口开始之前，窗口内按 lastRun 口径 0 次）；GPQA −189 与 τ² −50 定位至同一条目 z-ai/glm-5.2-20260616 / Relace 一次 run 移除；GPQA 累计 −117,326；439 倍数假说第二十轮检验失败）；1017 overrides 站住第七轮（deepseek-v4.1-flash $4.98e-8 站住第六轮；minimax/hy3/hy4 翻段值站住不翻回；glm-5.1 改价站住首轮；kimi-k3 翻回 109 轮前值；inkling-2-mini 已变价站住；~kimi-latest 第三个值 $1.824e-7 回到 108 轮值；目录 476 不变、elo 零刷新）；窗口内新发布 0 个（v1.112.0-dev.2 关窗后预发布）；benchmarks API 401 第 194–197 次；R7 十组 0/3/2/6/16/6/5/6/8/10，登记集 77 项零升版；裸式区间仍 500（第六轮未恢复）。
- 纠正：RA 腿「窗口内新 run」措辞纠正为轮间到达（两 run lastRun 16:08:49Z/16:21:44Z 均在窗口开始 17:09:08Z 之前；关账 +871 不变）；GPQA/τ² 定位以 RA 腿页比对为准（RB 腿未做页比对）；~kimi-latest 措辞纠正（RA 腿「completion/缓存未变」不精确，绝对值与 RB 一致）；v1.106.12「未探查」→「已证实存在」（tag 级 200，107 轮脚本笔误缺口闭合）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮已一手复核）；439 倍数假说仍不成立（第二十轮检验失败）；GPQA −154 与 Model Routers +78 成因仍未定位；minimax/hy3/hy4 与 1017 翻转成因仍未定位（本轮翻段值站住不翻回）；segment 评估机制（时钟偏移 vs top 价字段独立更新）仍未区分；GitHub 限流重置点之后仍未清限流（RB 腿三次 403，retry2 末尾 remaining 59），下轮不得以「等重置点」默认可取回。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；OA 12 项 + S2 余项重取（约 2026-11-22 00:00Z 重置）；S2 Conformal、ExProt、SkillScope 论文端点重取；llm-d 108 轮 main HEAD（234d1c78）PushEvent 仍未定位；decisions 计数方法仍未决（旧值 37）；qwen3.5-9b input_cache_read 字段缺失已第十八轮；v1.108.15 已于关窗后发布（GitHub 21:35:07Z、PyPI 21:34:14Z），下轮复核窗口内新发布归属。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；OA 12 项重取；1017 改价复核；v1.108.15 发布归属复核；minimax/hy3/hy4 翻转成因），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-22（07:09 轮，第 111 轮）

- 增量：计数器 2,238,506（±0，第七十四轮闭合；14 卡逐项相同；439 倍数假说第二十一轮检验无新数据点）；1017 overrides 站住第八轮（deepseek-v4.1-flash $4.98e-8 站住第七轮；minimax/hy3/hy4 翻段值站住不翻回；glm-5.1 再改价至 $1.4e-6/$4.4e-6/$2.6e-7，110 轮「改价站住首轮」推翻；~kimi-latest 第三个值 $1.824e-7 首次站住；目录 476 不变、elo 连续第三轮零刷新）；窗口内新稳定发布 1 个（LiteLLM v1.108.15 LTS；109/110 轮「不存在」结论正式推翻）；benchmarks API 401 第 198–201 次；R7 十组 0/3/2/6/16/6/5/6/8/10，登记集 77 项零升版；OA 12 项 + cited_by 全部取回（SkillScope 12 → 13，新增引用者 W7236814632 = arXiv:2611.24185）；S2 全局未取回只剩 SkillLearnBench（68）与 SkillSeek（3）；裸式区间仍 500（第七轮未恢复）。
- 纠正：llm-d forks「948 → 950（+2）」订正为 949 → 950（+1，RA 腿基线错误；SA 腿沿用的「+2」一并纠正）；SB 腿 OA 新增引用者候选 W7160630519 不成立（实为 W7236814632 = arXiv:2611.24185）；108 轮 main HEAD（234d1c78）PushEvent 结案为「事件流中缺失」，下轮起不再续计；ExProt 未取回项被 RB 腿 200 取回推翻，不再续记。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮已一手复核）；439 倍数假说本轮无新数据点（不计支持/反例）；GPQA −154 与 Model Routers +78 成因仍未定位；minimax/hy3/hy4 翻转成因仍未定位；segment 评估机制仍未区分；本轮 GitHub 首扫零 403（110 轮限流警告仍有效，下轮不得以「本轮未限流」默认可取回）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；S2 SkillLearnBench（68）与 SkillSeek（3）重取；decisions 计数方法仍未决（旧值 37）；qwen3.5-9b input_cache_read 字段缺失已第十九轮；#2197 归属 112 轮。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；S2 余 2 项重取；1017 改价复核；历史暂持 46 轮批评估），再按子主题轮转（路由成本/延迟权衡模型）。

## 2026-11-22（10:09 轮，第 112 轮）

- 增量：计数器 2,238,506（±0，第七十五轮闭合；14 卡逐项相同；439 倍数假说第二十二轮检验无新数据点）；1017 overrides 站住第九轮（deepseek-v4.1-flash $4.98e-8 站住第八轮；minimax/hy4 翻回段位一致、hy3 改新价且 overrides 字段消失；glm-5.1 翻回 110 轮值，111 轮「改价站住首轮」推翻；~kimi-latest 第三个值 $1.824e-7 站住第二轮；目录 476 不变、elo 三轮零刷新后大刷新 up 274/down 499/新增 8/164 模型）；窗口内新稳定发布 0 个（v1.108.15 归属 111 轮窗口再确认；OpenRouter 数据处理事件博文：下架 3 家提供商、125 端点移出 ZDR 资格，厂商自报未独立核实）；benchmarks API 401 第 202–205 次；R7 十组 0/3/2/6/16/6/5/6/8/10，登记集 77 项零升版；S2 九项全部取回（SkillLearnBench 68、SkillSeek 3 首次直接取回确认，111 轮余项全部闭合）；OA 12 项 + cited_by 仍全部 429 未取回（预算耗尽，重置约 11-23 00:00Z）；裸式区间仍 500（第八轮未恢复）。
- 纠正：llm-d stars +1 由 111 轮「关窗后事件」纠正为本轮窗口内事件（WatchEvent 23:15:38Z，Δ=0 秒）；elo 旧口径（模型顶层 `elo` 字段）订正为空口径——实测 0 个模型，该字段不存在，design_arena 口径今后为准，「三轮零刷新」结论在正确口径下仍然成立；glm-5.1 的 111 轮「改价站住首轮」推翻（翻回 110 轮值）。
- 裁定：R7 仍暂定（生产级联证据仍仅 CascadeRec + HIRA；研究级仅 OASIS 1 篇；在线学习阈值生产流量证据仍为零；本轮已一手复核）；439 倍数假说本轮无新数据点（不计支持/反例）；GPQA −154 与 Model Routers +78 成因仍未定位；minimax/hy3/hy4 翻转成因仍未定位；segment 评估机制仍未区分；Lens CLI 博文归属未确认，不得记为窗口内发布；路由篇第五次拆分完成（2026-11-21 01:09 之前 9 节移入归档，主文件 395 → 173 行）。
- 剩余未核实缺口：arXiv 11-20 批次筛查（最早 11-23 周一 01:00Z 公告，余部 11-24 周二 01:00Z；1120-2/1120-full 留待周二 00:00Z 之后重做）；OpenAlex 12 项 + cited_by 重取（贴近 UTC 午夜）；decisions 计数方法仍未决（旧值 37）；qwen3.5-9b input_cache_read 字段缺失已第二十轮；板面 $6.43 缺口 $0.0625；CVE 两项厂商逐字确认仍缺；#2197 维护方评审/合入（归属 112 轮）；LiteLLM Lens CLI 博文窗口归属；OpenRouter 数据处理事件未披露项（被下架 3 家名称、125 端点清单）。
- 下轮起点：先清上述缺口（11-20 批次筛查为首项；OpenAlex 重取；1017 改价复核；elo design_arena 口径沿用），再按子主题轮转（路由成本/延迟权衡模型）。
