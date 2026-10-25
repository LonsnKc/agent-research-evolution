# The Four Tracks / 四条轨道

Each track is one file in [`research/`](../research/), structured the same way: a **baseline** section (what was established when tracking began) followed by dated **round** sections. Each round separates verified findings from unverified gaps.

每个轨道对应 `research/` 里的一份文件，结构统一：先是**基线**（开始追踪时已确立的认识），其后是按日期追加的**轮次**小结；每轮都把已核实结论与未核实缺口分开写。

---

## 1. API Routing — `track-api-routing.md`

**Scope / 范围.** Traditional API gateways, service-level routing (llm-d, semantic routers, inference schedulers), LLM gateways, and learned model routers; routing mechanisms (rules, weights, cascading, cost/latency/quality-first, bandits, semantic and graph routing, KV-cache awareness); gateway operations (cooldowns, rate limits, budgets, retries, caching); gateway supply-chain security.

**Key questions / 核心问题.** Where does routing "decision" live as systems mature — inside the gateway, in declarative policy, or in dedicated decision endpoints? What does a routing layer have to guarantee before its cost savings are believable?

## 2. Agent Harness — `track-harness.md`

**Scope / 范围.** Everything deterministic around the model: context assembly and engineering, tool loops, state and checkpoints, memory access, sandboxes and permissions, guardrails, human-in-the-loop, traces, evaluation harnesses, recovery and retries; long-running-agent patterns; framework comparisons; the MCP ecosystem.

**Key questions / 核心问题.** Which harness failures dominate in practice (context rot, missing hand-off artifacts, unverified "done") and which mechanisms actually prevent them? How must evaluation results be pinned (model, harness version, task set, budget, runs) before they can be compared at all?

## 3. Layered Memory — `track-layered-memory.md`

**Scope / 范围.** Memory taxonomies (working / episodic / semantic / procedural), storage forms (raw logs, vector indexes, knowledge and temporal graphs, summary trees), retrieval pipelines (recall, layer expansion, re-ranking, context budgeting), write paths (extraction, summarisation, reflection, consolidation), demotion vs. invalidation vs. deletion, and memory benchmarks (LongMemEval, LoCoMo).

**Key questions / 核心问题.** How do you forget correctly — what may only be demoted in ranking, what must leave the current-truth set while keeping history, and what must be hard-deleted? Which vendor benchmark numbers survive a primary-source check?

## 4. Skills — `track-skills.md`

**Scope / 范围.** The agent-skills format and its open standard; how hosts differ (discovery paths and precedence, description limits, invocation gestures, permission defaults, progressive disclosure); writing descriptions as the trigger surface; skill evaluation (trigger sets, paired A/B, pass-rate vs. token cost); security review of third-party skills before installation.

**Key questions / 核心问题.** How far has the "one canonical skill, many hosts" idea actually converged, and where does each host still force per-host adaptation? What evidence separates a skill that helps from one that only costs tokens?

---

## Reading guide / 阅读建议

- New here? Read the master report's latest round summary first ([`research/report.md`](../research/report.md)), then the track you care about.
- Looking for one claim? Search the track file; every round header carries its date and watermark window.
- 刚接触？先看总报告最新一轮小结，再读你关心的那条轨道；查某条结论时按轨道文件内日期与水位窗口定位。
