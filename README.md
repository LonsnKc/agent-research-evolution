# Agent Research Evolution

**A living, daily-updated research record on how agent systems are built — API routing, agent harnesses, layered memory, and skills.**

[![Updates](https://img.shields.io/badge/updates-daily-success)](CHANGELOG.md)
[![Tracks](https://img.shields.io/badge/tracks-4-blue)](docs/tracks.md)
[![Language](https://img.shields.io/badge/language-English%20%7C%20%E4%B8%AD%E6%96%87-informational)](#中文说明)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey)](LICENSE)
[![Community](https://img.shields.io/badge/community-LINUX%20DO-orange)](https://linux.do/)

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
| Evolution rounds published | Round 1 — 2026-10-25 (watermark 2026-10-24 → 2026-10-25); Round 2 — 2026-10-27 (watermark 2026-10-25 → 2026-10-27; the 2026-10-26 round was skipped); Round 3 — 2026-10-28 (watermark 2026-10-27 → 2026-10-28; gap-review + new-coverage round, skill references 0.2.0 → 0.3.0); Round 4 — 2026-10-29 (watermark 2026-10-28 → 2026-10-29; gap-review + new-coverage round, skill references 0.3.1 → 0.4.0); Round 5 — 2026-10-30 (watermark 2026-10-29 → 2026-10-30; gap-review + new-coverage round, skill references 0.4.0 → 0.5.0); Round 6 — 2026-10-31 (watermark 2026-10-30 → 2026-10-31; gap-review + new-coverage round, skill references 0.5.0 → 0.6.0); Round 7 — 2026-11-03 (watermark 2026-10-31 → 2026-11-03; gap-review + new-coverage round, skill references 0.6.0 → 0.7.0); Round 8 — 2026-11-04 (watermark 2026-11-03 → 2026-11-04; gap-review + new-coverage round, skill references 0.7.0 → 0.8.0); Round 9 — 2026-11-05 (watermark 2026-11-04 → 2026-11-05; gap-review + new-coverage round, skill references 0.8.0 → 0.9.0); Round 10 — 2026-11-07 (watermark 2026-11-05 → 2026-11-07; gap-review + new-coverage round, skill references 0.9.0 → 0.10.0); Round 11 — 2026-11-07 (afternoon round, watermark 2026-11-07 morning → 2026-11-07 17:37; gap-review + new-coverage round, skill references 0.10.0 → 0.11.0); Round 12 — 2026-11-07 (evening round, watermark 2026-11-07 afternoon → 2026-11-07 23:37; gap-review + new-coverage round, skill references 0.11.0 → 0.12.0); Round 13 — 2026-11-08 (morning round, watermark 2026-11-07 23:37 → 2026-11-08 ~06:10; gap-review + new-coverage round, skill references 0.12.0 → 0.13.0); Round 14 — 2026-11-08 (midday round, watermark 2026-11-08 ~06:10 → 2026-11-08 11:37; gap-review + new-coverage round, skill references 0.13.0 → 0.14.0); Round 15 — 2026-11-08 (evening round, watermark 2026-11-08 11:37 → 2026-11-08 17:37; gap-review + new-coverage round, skill references 0.14.0 → 0.15.0); Round 16 — 2026-11-08 (late-night round, watermark 2026-11-08 17:37 → 2026-11-09 ~00:08; gap-review + new-coverage round, skill references 0.15.0 → 0.16.0); Round 17 — 2026-11-09 (morning round, watermark 2026-11-08 23:37 → 2026-11-09 05:37; gap-review + new-coverage round, skill references 0.16.0 → 0.17.0); Round 18 — 2026-11-09 (midday round, watermark 2026-11-09 05:37 → 2026-11-09 11:37; gap-review + new-coverage round, skill references 0.17.0 → 0.18.0); Round 19 — 2026-11-09 (evening round, watermark 2026-11-09 11:37 → 2026-11-09 17:37; gap-review + new-coverage round, skill references 0.18.0 → 0.19.0); Round 20 — 2026-11-09 (late-night round, watermark 2026-11-09 17:37 → 2026-11-09 23:37; gap-review + full-text deep-read round, skill references 0.19.0 → 0.20.0); Round 21 — 2026-11-10 (early-morning round, watermark 2026-11-09 23:37 → 2026-11-10 05:37; gap-review + new-coverage round, skill references 0.20.0 → 0.21.0); Round 22 — 2026-11-10 (midday round, watermark 2026-11-10 05:37 → 2026-11-10 11:37; gap-review + new-coverage round, skill references 0.21.0 → 0.22.0); Round 23 — 2026-11-10 (evening round, watermark 2026-11-10 11:37 → 2026-11-10 17:37; gap-review + new-coverage round, skill references 0.22.0 → 0.23.0); Round 24 — 2026-11-11 (early-morning round, watermark 2026-11-10 17:37 → 2026-11-11 01:09; gap-review + new-coverage round, skill references 0.23.0 → 0.24.0); Round 25 — 2026-11-11 (early-morning round, watermark 2026-11-11 01:09 → 2026-11-11 04:09; gap-review + new-coverage round, skill references 0.24.0 → 0.25.0); Round 26 — 2026-11-11 (morning round, watermark 2026-11-11 04:09 → 2026-11-11 07:09; gap-review + new-coverage round, skill references 0.25.0 → 0.26.0); Round 27 — 2026-11-11 (morning round, watermark 2026-11-11 07:09 → 2026-11-11 10:09; gap-review + new-coverage round, skill references 0.26.0 → 0.27.0); Round 28 — 2026-11-11 (afternoon round, watermark 2026-11-11 10:09 → 2026-11-11 13:09; gap-review + new-coverage round, skill references 0.27.0 → 0.28.0); Round 29 — 2026-11-11 (afternoon round, watermark 2026-11-11 13:09 → 2026-11-11 16:09; gap-review + new-coverage round, skill references 0.28.0 → 0.29.0); Round 30 — 2026-11-11 (evening round, watermark 2026-11-11 16:09 → 2026-11-11 19:09; gap-review + new-coverage round, skill references 0.29.0 → 0.30.0); Round 31 — 2026-11-11 (late-evening round, watermark 2026-11-11 19:09 → 2026-11-11 22:09; gap-review + new-coverage round, skill references 0.30.0 → 0.31.0); Round 32 — 2026-11-12 (early-morning round, watermark 2026-11-11 22:09 → 2026-11-12 01:09; gap-review + new-coverage round, skill references 0.31.0 → 0.32.0); Round 33 — 2026-11-12 (early-morning round, watermark 2026-11-12 01:09 → 2026-11-12 04:09; gap-review + new-coverage round, skill references 0.32.0 → 0.33.0); Round 34 — 2026-11-12 (morning round, watermark 2026-11-12 04:09 → 2026-11-12 07:09; gap-review + new-coverage round, skill references 0.33.0 → 0.34.0); Round 35 — 2026-11-12 (morning round, watermark 2026-11-12 07:09 → 2026-11-12 10:09; gap-review + new-coverage round, skill references 0.34.0 → 0.35.0); Round 36 — 2026-11-12 (afternoon round, make-up of the interrupted 13:09 round, watermark 2026-11-12 10:09 → 2026-11-12 16:09; gap-review + new-coverage round, skill references 0.35.0 → 0.36.0); Round 37 — 2026-11-12 (evening round, watermark 2026-11-12 16:09 → 2026-11-12 19:09; gap-review + new-coverage round, skill references 0.36.0 → 0.37.0); Round 38 — 2026-11-12 (evening round, watermark 2026-11-12 19:09 → 2026-11-12 22:09; gap-review + deep-dive round, skill references 0.37.0 → 0.38.0); Round 39 — 2026-11-13 (early-morning round, watermark 2026-11-12 22:09 → 2026-11-13 01:09; gap-review + new-coverage round, skill references 0.38.0 → 0.39.0); Round 40 — 2026-11-13 (early-morning round, watermark 2026-11-13 01:09 → 2026-11-13 04:09; gap-review + new-coverage round, skill references 0.39.0 → 0.40.0); Round 41 — 2026-11-13 (morning round, watermark 2026-11-12 20:09 → 23:09 UTC; gap-review + new-coverage round, skill references 0.40.0 → 0.41.0); Round 42 — 2026-11-13 (morning round, watermark 2026-11-12 23:09 → 2026-11-13 02:09 UTC; gap-review + new-coverage round, skill references 0.41.0 → 0.42.0); Round 43 — 2026-11-13 (morning round, watermark 2026-11-13 02:09 → 2026-11-13 05:09 UTC; gap-review + new-coverage round, skill references 0.42.0 → 0.43.0); Round 44 — 2026-11-13 (afternoon round, watermark 2026-11-13 05:09 → 2026-11-13 08:09 UTC; gap-review + new-coverage round, skill references 0.43.0 → 0.44.0); Round 45 — 2026-11-13 (afternoon round, watermark 2026-11-13 08:09 → 2026-11-13 11:09 UTC; gap-review + new-coverage round, skill references 0.44.0 → 0.45.0; Round 46 — 2026-11-13 (evening round, watermark 2026-11-13 11:09 → 2026-11-13 14:09 UTC; gap-review + new-coverage round, skill references 0.45.0 → 0.46.0); Round 47 — 2026-11-14 (early-morning round, watermark 2026-11-13 14:09 → 2026-11-13 17:09 UTC; gap-review + new-coverage round, skill references 0.46.0 → 0.47.0); Round 48 — 2026-11-14 (early-morning round, watermark 2026-11-13 17:09 → 2026-11-13 20:09 UTC; gap-review + new-coverage round, skill references 0.47.0 → 0.48.0); Round 49 — 2026-11-14 (early-morning round, watermark 2026-11-13 20:09 → 2026-11-14 02:09 UTC; gap-review + new-coverage round, skill references 0.48.0 → 0.49.0); Round 50 — 2026-11-14 (afternoon round, watermark 2026-11-14 02:09 → 2026-11-14 05:09 UTC; gap-review + new-coverage round, skill references 0.49.0 → 0.50.0; Round 51 — 2026-11-14 (afternoon round, watermark 2026-11-14 05:09 → 2026-11-14 08:09 UTC; gap-review + new-coverage round, skill references 0.50.0 → 0.51.0); Round 52 — 2026-11-14 (evening round, watermark 2026-11-14 08:09 → 2026-11-14 11:09 UTC; gap-review + new-coverage round, skill references 0.51.0 → 0.52.0) |
| Research files | 5 (`research/report.md` + 4 track notes) |
| Update cadence | Daily, with post-push verification |

## Scope

**In scope:** the research-evolution program only — its master report, four track notes, and the documentation in this repository.

**Out of scope:** source code, installable skill packages, benchmarks you can run, and any private working material. Companion design work exists but is published separately, if at all; this repository deliberately contains research findings only.

## License and citation

This work is licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSE). You may share and adapt the research notes — including commercially — as long as you give appropriate credit to LonsnKc and this repository, link to the license, and indicate if changes were made. Individual findings belong to their cited primary sources; please cite those for the underlying claims.

## Community

This project is shared and discussed with the community at [LINUX DO](https://linux.do/).

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
| 已发表轮次 | 第 1 轮——2026-10-25（水位 2026-10-24 → 2026-10-25）；第 2 轮——2026-10-27（水位 2026-10-25 → 2026-10-27；2026-10-26 轮次因平台安全审查跳过）；第 3 轮——2026-10-28（水位 2026-10-27 → 2026-10-28；缺口复核 + 新增覆盖轮，技能参考 0.2.0 → 0.3.0）；第 4 轮——2026-10-29（水位 2026-10-28 → 2026-10-29；缺口复核 + 新增覆盖轮，技能参考 0.3.1 → 0.4.0）；第 5 轮——2026-10-30（水位 2026-10-29 → 2026-10-30；缺口复核 + 新增覆盖轮，技能参考 0.4.0 → 0.5.0）；第 6 轮——2026-10-31（水位 2026-10-30 → 2026-10-31；缺口复核 + 新增覆盖轮，技能参考 0.5.0 → 0.6.0）；第 7 轮——2026-11-03（水位 2026-10-31 → 2026-11-03；缺口复核 + 新增覆盖轮，技能参考 0.6.0 → 0.7.0）；第 8 轮——2026-11-04（水位 2026-11-03 → 2026-11-04；缺口复核 + 新增覆盖轮，技能参考 0.7.0 → 0.8.0）；第 9 轮——2026-11-05（水位 2026-11-04 → 2026-11-05；缺口复核 + 新增覆盖轮，技能参考 0.8.0 → 0.9.0）；第 10 轮——2026-11-07（水位 2026-11-05 → 2026-11-07；缺口复核 + 新增覆盖轮，技能参考 0.9.0 → 0.10.0）；第 11 轮——2026-11-07（下午轮，水位 2026-11-07 上午 → 2026-11-07 17:37；缺口复核 + 新增覆盖轮，技能参考 0.10.0 → 0.11.0）；第 12 轮——2026-11-07（晚间轮，水位 2026-11-07 下午 → 2026-11-07 23:37；缺口复核 + 新增覆盖轮，技能参考 0.11.0 → 0.12.0）；第 13 轮——2026-11-08（晨轮，水位 2026-11-07 23:37 → 2026-11-08 约 06:10；缺口复核 + 新增覆盖轮，技能参考 0.12.0 → 0.13.0）；第 14 轮——2026-11-08（中午轮，水位 2026-11-08 约 06:10 → 2026-11-08 11:37；缺口复核 + 新增覆盖轮，技能参考 0.13.0 → 0.14.0）；第 15 轮——2026-11-08（晚间轮，水位 2026-11-08 11:37 → 2026-11-08 17:37；缺口复核 + 新增覆盖轮，技能参考 0.14.0 → 0.15.0）；第 16 轮——2026-11-08（深夜轮，水位 2026-11-08 17:37 → 2026-11-09 约 00:08；缺口复核 + 新增覆盖轮，技能参考 0.15.0 → 0.16.0）；第 17 轮——2026-11-09（晨轮，水位 2026-11-08 23:37 → 2026-11-09 05:37；缺口复核 + 新增覆盖轮，技能参考 0.16.0 → 0.17.0）；第 18 轮——2026-11-09（中午轮，水位 2026-11-09 05:37 → 2026-11-09 11:37；缺口复核 + 新增覆盖轮，技能参考 0.17.0 → 0.18.0）；第 19 轮——2026-11-09（晚间轮，水位 2026-11-09 11:37 → 2026-11-09 17:37；缺口复核 + 新增覆盖轮，技能参考 0.18.0 → 0.19.0）；第 20 轮——2026-11-09（深夜轮，水位 2026-11-09 17:37 → 2026-11-09 23:37；缺口复核 + 全文深读轮，技能参考 0.19.0 → 0.20.0）；第 21 轮——2026-11-10（凌晨轮，水位 2026-11-09 23:37 → 2026-11-10 05:37；缺口复核 + 新增覆盖轮，技能参考 0.20.0 → 0.21.0）；第 22 轮——2026-11-10（中午轮，水位 2026-11-10 05:37 → 2026-11-10 11:37；缺口复核 + 新增覆盖轮，技能参考 0.21.0 → 0.22.0）；第 23 轮——2026-11-10（晚间轮，水位 2026-11-10 11:37 → 2026-11-10 17:37；缺口复核 + 新增覆盖轮，技能参考 0.22.0 → 0.23.0）；第 24 轮——2026-11-11（凌晨轮，水位 2026-11-10 17:37 → 2026-11-11 01:09；缺口复核 + 新增覆盖轮，技能参考 0.23.0 → 0.24.0）；第 25 轮——2026-11-11（凌晨轮，水位 2026-11-11 01:09 → 2026-11-11 04:09；缺口复核 + 新增覆盖轮，技能参考 0.24.0 → 0.25.0）；第 26 轮——2026-11-11（晨轮，水位 2026-11-11 04:09 → 2026-11-11 07:09；缺口复核 + 新增覆盖轮，技能参考 0.25.0 → 0.26.0）；第 27 轮——2026-11-11（晨轮，水位 2026-11-11 07:09 → 2026-11-11 10:09；缺口复核 + 新增覆盖轮，技能参考 0.26.0 → 0.27.0）；第 28 轮——2026-11-11（午后轮，水位 2026-11-11 10:09 → 2026-11-11 13:09；缺口复核 + 新增覆盖轮，技能参考 0.27.0 → 0.28.0）；第 29 轮——2026-11-11（下午轮，水位 2026-11-11 13:09 → 2026-11-11 16:09；缺口复核 + 新增覆盖轮，技能参考 0.28.0 → 0.29.0）；第 30 轮——2026-11-11（晚间轮，水位 2026-11-11 16:09 → 2026-11-11 19:09；缺口复核 + 新增覆盖轮，技能参考 0.29.0 → 0.30.0）；第 31 轮——2026-11-11（深夜轮，水位 2026-11-11 19:09 → 2026-11-11 22:09；缺口复核 + 新增覆盖轮，技能参考 0.30.0 → 0.31.0）；第 32 轮——2026-11-12（凌晨轮，水位 2026-11-11 22:09 → 2026-11-12 01:09；缺口复核 + 新增覆盖轮，技能参考 0.31.0 → 0.32.0）；第 33 轮——2026-11-12（凌晨轮，水位 2026-11-12 01:09 → 2026-11-12 04:09；缺口复核 + 新增覆盖轮，技能参考 0.32.0 → 0.33.0）；第 34 轮——2026-11-12（晨轮，水位 2026-11-12 04:09 → 2026-11-12 07:09；缺口复核 + 新增覆盖轮，技能参考 0.33.0 → 0.34.0）；第 35 轮——2026-11-12（晨轮，水位 2026-11-12 07:09 → 2026-11-12 10:09；缺口复核 + 新增覆盖轮，技能参考 0.34.0 → 0.35.0）；第 36 轮——2026-11-12（下午轮，13:09 轮补做轮，13:09 轮因 VM 替换中断，水位 2026-11-12 10:09 → 2026-11-12 16:09；缺口复核 + 新增覆盖轮，技能参考 0.35.0 → 0.36.0）；第 37 轮——2026-11-12（晚间轮，水位 2026-11-12 16:09 → 2026-11-12 19:09；缺口复核 + 新增覆盖轮，技能参考 0.36.0 → 0.37.0）；第 38 轮——2026-11-12（晚间轮，水位 2026-11-12 19:09 → 2026-11-12 22:09；缺口复核 + 深挖轮，技能参考 0.37.0 → 0.38.0）；第 39 轮——2026-11-13（凌晨轮，水位 2026-11-12 22:09 → 2026-11-13 01:09；缺口复核 + 新增覆盖轮，技能参考 0.38.0 → 0.39.0）；第 40 轮——2026-11-13（凌晨轮，水位 2026-11-13 01:09 → 2026-11-13 04:09；缺口复核 + 新增覆盖轮，技能参考 0.39.0 → 0.40.0）；第 41 轮——2026-11-13（晨轮，水位 2026-11-12 20:09 → 23:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.40.0 → 0.41.0）；第 42 轮——2026-11-13（晨轮，水位 2026-11-12 23:09 → 2026-11-13 02:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.41.0 → 0.42.0）；第 43 轮——2026-11-13（晨轮，水位 2026-11-13 02:09 → 2026-11-13 05:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.42.0 → 0.43.0）；第 44 轮——2026-11-13（下午轮，水位 2026-11-13 05:09 → 2026-11-13 08:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.43.0 → 0.44.0）；第 45 轮——2026-11-13（下午轮，水位 2026-11-13 08:09 → 2026-11-13 11:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.44.0 → 0.45.0；第 46 轮——2026-11-13（晚间轮，水位 2026-11-13 11:09 → 2026-11-13 14:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.45.0 → 0.46.0）；第 47 轮——2026-11-14（凌晨轮，水位 2026-11-13 14:09 → 2026-11-13 17:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.46.0 → 0.47.0）；第 48 轮——2026-11-14（凌晨轮，水位 2026-11-13 17:09 → 2026-11-13 20:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.47.0 → 0.48.0）；第 49 轮——2026-11-14（凌晨轮，水位 2026-11-13 20:09 → 2026-11-14 02:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.48.0 → 0.49.0）；第 50 轮——2026-11-14（下午轮，水位 2026-11-14 02:09 → 2026-11-14 05:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.49.0 → 0.50.0）；第 51 轮——2026-11-14（下午轮，水位 2026-11-14 05:09 → 2026-11-14 08:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.50.0 → 0.51.0）；第 52 轮——2026-11-14（晚间轮，水位 2026-11-14 08:09 → 2026-11-14 11:09 UTC；缺口复核 + 新增覆盖轮，技能参考 0.51.0 → 0.52.0） |
| 调研文件 | 5 份（总报告 + 四份轨道笔记） |
| 更新节奏 | 每日，推送后必核验 |

## 范围

**只包含**调研进化这一项：总报告、四份轨道笔记与本仓库的说明文档。**不包含**源代码、可安装的技能包，以及任何私人工作材料与凭据。

## 授权与引用

本仓库的调研笔记采用 [CC BY 4.0（知识共享 署名 4.0 国际）](LICENSE) 授权：可以自由转载、改编，包括商用，条件是署名 LonsnKc 并注明出处为本仓库、附授权链接、并说明是否做过修改。具体结论请同时引用其标注的一手来源。

## 社区

本项目在 [LINUX DO](https://linux.do/) 社区交流分享。
