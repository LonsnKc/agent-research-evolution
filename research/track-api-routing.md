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

