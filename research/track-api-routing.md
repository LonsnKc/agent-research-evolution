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
