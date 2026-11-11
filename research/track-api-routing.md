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
