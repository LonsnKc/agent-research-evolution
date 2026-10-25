# Agent Research Evolution

**A living, daily-updated research record on how agent systems are built — API routing, agent harnesses, layered memory, and skills.**

[![Updates](https://img.shields.io/badge/updates-daily-success)](CHANGELOG.md)
[![Tracks](https://img.shields.io/badge/tracks-4-blue)](docs/tracks.md)
[![Language](https://img.shields.io/badge/language-English%20%7C%20%E4%B8%AD%E6%96%87-informational)](#中文说明)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey)](LICENSE)

> English is the primary language of this README. A full Chinese version follows below: [中文说明](#中文说明).

---

## What this is

This repository publishes the ongoing output of a single, continuous research program: **tracking how production agent systems evolve, one day at a time, across four technical tracks**:

1. **API Routing** — gateways, LLM routers, cascading, cost/latency/quality trade-offs, routing policy as configuration.
2. **Agent Harness** — everything around the model that makes agents reliable: context assembly, tool loops, state and checkpoints, guardrails, tracing, evaluation harnesses.
3. **Layered Memory** — working / episodic / semantic / procedural memory, retrieval pipelines, forgetting and invalidation, memory benchmarks.
4. **Skills** — agent skill formats and standards, cross-host compatibility, discovery and conflict rules, trigger design, skill evaluation and security review.

Every day, a scheduled research round checks official documentation, papers, changelogs, and release notes for new developments in all four tracks, deep-dives one rotating sub-topic per track, and appends a dated, incremental update to the notes. Verified conclusions are kept strictly separate from unverified claims.

This is a **research publication**, not a code library: it contains notes, analyses, and an evolving master report — no software, no skill packages, and no credentials.

## Why it exists

Agent-system knowledge goes stale in weeks. One-off survey reports freeze a moment that has already passed, and vendor announcements rarely distinguish what is verified from what is marketing. This project instead keeps a *watermarked, append-only record*: each round states what actually changed since the previous round, what was checked and found unchanged, and which open questions remain unverified. Over time that produces something a single report cannot — a traceable history of how the field, and our conclusions about it, actually moved.

## What you can do with it

- **Design an agent system** — use the track notes as a decision base for routing strategy, harness architecture, memory layering, and skill authoring; each note records mechanisms, trade-offs, and failure modes, not just product names.
- **Catch up on a topic fast** — read one track file end to end: baseline first, then dated rounds in order.
- **Audit a claim** — every round separates *verified* findings from *unverified* ones and lists the remaining gaps explicitly, so you can see exactly how much weight a conclusion can bear.
- **Follow the field passively** — watch this repository; the daily changelog tells you what genuinely changed, including honest "nothing new today" rounds.

## Repository structure

```text
agent-research-evolution/
├── README.md                  # This file (English primary, Chinese below)
├── CHANGELOG.md               # One entry per published update round
├── docs/
│   ├── tracks.md              # Scope, key questions, and reading guide per track
│   └── methodology.md         # Evidence standards and the daily maintenance cycle
└── research/
    ├── report.md              # Evolving master report with per-round summaries
    ├── track-api-routing.md
    ├── track-harness.md
    ├── track-layered-memory.md
    └── track-skills.md
```

The files under `research/` are the canonical research record, published verbatim from the research workspace. Each track file starts with its baseline (the state of knowledge when tracking began) followed by dated round sections.

## How updates work

- A research round runs **daily at about 09:50 (Asia/Shanghai)**: four tracks in parallel, public sources only.
- Afterwards, a maintenance round **collects, classifies, and syncs** any new research output into this repository, updates this README's status and the changelog when content changed, pushes, and then **verifies that the remote repository exactly matches what was published** (commit and file-level check). A round with no new findings is recorded honestly as such.
- Numbers reported by vendors and second-hand conclusions are always labelled *unverified* until checked against a primary source.

## Evidence standards

- Primary sources first: official docs, papers, and release notes outrank summaries and social posts.
- Every round carries an explicit honesty note when a time window produced no genuine new releases, instead of dressing up baseline backfill as news.
- Evaluation claims must pin their conditions (model, harness version, task-set version, resource limits, number of runs); a single pass-rate number on its own is treated as not comparable.
- The full method is documented in [docs/methodology.md](docs/methodology.md).

## Current status

| Item | State |
| --- | --- |
| Baseline research | 2026-10-23 (four-topic deep research) and 2026-10-24 (skills deep research) |
| Evolution rounds published | Round 1 — 2026-10-25 (watermark 2026-10-24 → 2026-10-25) |
| Research files | 5 (`research/report.md` + 4 track notes) |
| Update cadence | Daily, with post-push verification |

## Scope

**In scope:** the research-evolution program only — its master report, four track notes, and the documentation in this repository.

**Out of scope:** source code, installable skill packages, benchmarks you can run, and any private working material. Companion design work exists but is published separately, if at all; this repository deliberately contains research findings only.

## License and citation

This work is licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSE). You may share and adapt the research notes — including commercially — as long as you give appropriate credit to LonsnKc and this repository, link to the license, and indicate if changes were made. Individual findings belong to their cited primary sources; please cite those for the underlying claims.

---

## 中文说明

**一个持续生长的智能体系统调研记录——每天更新，追踪 API 路由、Agent Harness、分层记忆与技能（Skills）四个方向的真实进展。**

## 这是什么

本仓库只发表一项持续工作的成果：**「智能体系统调研持续进化」**。它由四条调研轨道组成：

1. **API 路由**——传统网关、LLM 网关与学习式模型路由，级联与成本/延迟/质量权衡，路由策略的声明式配置，以及网关层供应链安全。
2. **Agent Harness**——模型之外让智能体可靠运行的全部确定性软件：上下文组装、工具循环、状态与检查点、护栏、追踪与评估框架、长程任务模式与失败模式。
3. **分层记忆**——工作/情景/语义/程序记忆的划分与检索流水线，降权、失效与删除的区别，记忆污染与投毒风险，以及记忆类基准的正确读法。
4. **技能（Skills）**——技能格式与开放标准，跨宿主兼容与发现/冲突规则，描述触发面设计，技能评估方法与安装前安全审计。

每天由定时任务跑一轮调研：并发检查四条轨道的官方文档、论文、变更日志与版本发布，追踪新进展，并轮换细挖一个子主题，把增量以带日期的小节追加进笔记。**已核实结论与未核实说法严格分区标注。**

这是一个**调研发表仓库**，不是代码库：只有笔记、分析与持续生长的总报告，没有软件、没有技能安装包、没有任何凭据。

## 它有什么用

- **做智能体系统设计选型**：四条轨道笔记记录机制、权衡与失败模式，可直接作为路由策略、Harness 架构、记忆分层与技能编写的决策依据。
- **快速补齐一个方向**：从某个轨道文件的基线部分读起，按日期顺序读完各轮小结即可。
- **核查一条结论的成色**：每轮都单列未核实缺口清单，能清楚看到每条结论到底有多硬。
- **被动跟踪领域变化**：看 CHANGELOG 即可，包括「今天确实没有新发布」的诚实记录。

## 目录结构

见上方英文部分的结构树：`research/` 是调研正本（总报告 + 四份轨道笔记，与调研工作区逐字同步），`docs/` 是轨道说明与方法论，`CHANGELOG.md` 记录每一次发表更新。

## 更新机制

- 调研轮次：每天约 09:50（Asia/Shanghai）运行，四轨并发，只用公开来源。
- 维护轮次：调研之后执行，把新成果整理、归类、同步进本仓库；内容有变化时更新本页状态与 CHANGELOG，推送后**逐项核验远端与发表内容一致**（提交与文件级核对），确认无误才算完成。
- 厂商自报数字与二手结论一律标注「未核实」，直到对照一手来源核查为止。

## 当前状态

| 项目 | 状态 |
| --- | --- |
| 基线调研 | 2026-10-23 四主题深研、2026-10-24 技能深研 |
| 已发表轮次 | 第 1 轮——2026-10-25（水位 2026-10-24 → 2026-10-25） |
| 调研文件 | 5 份（总报告 + 四份轨道笔记） |
| 更新节奏 | 每日，推送后必核验 |

## 范围

**只包含**调研进化这一项：总报告、四份轨道笔记与本仓库的说明文档。**不包含**源代码、可安装的技能包，以及任何私人工作材料与凭据。

## 授权与引用

本仓库的调研笔记采用 [CC BY 4.0（知识共享 署名 4.0 国际）](LICENSE) 授权：可以自由转载、改编，包括商用，条件是署名 LonsnKc 并注明出处为本仓库、附授权链接、并说明是否做过修改。具体结论请同时引用其标注的一手来源。
