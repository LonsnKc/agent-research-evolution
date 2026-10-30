# Changelog

All published updates to this repository, newest first. One entry per maintenance round; research rounds are dated by their watermark window.

本仓库全部发表更新，最新在前。每个维护轮次一条；调研轮次以其水位窗口标注日期。

## 2026-10-30 — Round 5: gap-review + new-coverage round / 第 5 轮：缺口复核 + 新增覆盖轮

- Gap-review + new-coverage round (watermark 2026-10-29 → 2026-10-30; skill references 0.4.0 → 0.5.0): the round's main advance is a correction — the previous Dynamo v1.6.1 "latest stable" claim was overturned and retracted by the official releases-data table, while Dynamo v1.5.0/v1.5.1 were confirmed from the same table; it also closed or corrected three gaps (Zep SDK 4.0.1 language attribution, Mem0 Decay 20-timestamp recheck, tool-search 85% and 80%-reduction provenance) and resolved several items negatively (Codex 0.165.0 has no skill items, trigger-surface divergence unresolved, AAIF landing files still unpublished). The only first-hand release strictly dated 2026-10-30 was the Claude Code changelog 10-30 group (no skill-rule changes); langchain-core 1.7.3 is a 10-29 boundary date.
- 缺口复核 + 新增覆盖轮（水位 2026-10-29 → 2026-10-30；技能参考 0.4.0 → 0.5.0）：本轮最大进展是一项纠正——上一轮 Dynamo v1.6.1「最新稳定」说法被官方 releases-data 表推翻并撤回，同时 v1.5.0/v1.5.1 得到官方表确认；另关闭或纠偏三项缺口（Zep SDK 4.0.1 语言归属、Mem0 Decay 20 次时间戳复核、工具搜索 85% 与 80% 削减出处），并否定性解决多项（Codex 0.165.0 确无技能项、触发面分歧无收敛、AAIF 落地文件仍未发布）。严格落在 2026-10-30 当日的一手发布为 Claude Code changelog 10-30 组（无技能规则变更）；langchain-core 1.7.3 为 10-29 边界日。

## 2026-10-29 — Round 4: gap-review + new-coverage round / 第 4 轮：缺口复核 + 新增覆盖轮

- Gap-review + new-coverage round (watermark 2026-10-28 → 2026-10-29; skill references 0.3.1 → 0.4.0): no first-hand releases strictly dated after 2026-10-28 were confirmed across the four tracks; the round closed two gaps (TEPA paper text, Mem0 Decay open-source ruling) and resolved one item negatively (OpenCode v2.2.0 has no skill-specific changes), while advancing version stamps (LiteLLM 1.106.6, Dynamo v1.6.1, Agent Framework 1.23.0, Agents SDK v0.24.0, Letta 0.34.26).
- 缺口复核 + 新增覆盖轮（水位 2026-10-28 → 2026-10-29；技能参考 0.3.1 → 0.4.0）：严格落在 2026-10-28 之后的一手发布四轨均未确认；本轮关闭两项缺口（TEPA 论文原文、Mem0 Decay 开源支持裁决），否定性解决一项（OpenCode v2.2.0 无技能专项），并推进版本戳（LiteLLM 1.106.6、Dynamo v1.6.1、Agent Framework 1.23.0、Agents SDK v0.24.0、Letta 0.34.26）。

## 2026-10-28 — Round 3: gap-review + new-coverage round / 第 3 轮：缺口复核 + 新增覆盖轮

- Gap-review + new-coverage round (watermark 2026-10-27 → 2026-10-28; skill references 0.2.0 → 0.3.0): most remaining gaps from the previous round were closed or corrected — the LiteLLM pinning line was confirmed from provisional to verified, the OTel GenAI official spec landed, Graphiti conflicts were resolved, and LangMem/FrontierMemory were confirmed — while two previous conclusions were corrected (AgentExecutor maintenance wording, Google rollout-delay wording). No first-hand releases strictly dated 2026-10-28 were found across the four tracks; the 2026-10-27 first-hand items were the LiteLLM 1.106.5 batch and the Claude Code changelog skill fixes.
- 缺口复核 + 新增覆盖轮（水位 2026-10-27 → 2026-10-28；技能参考 0.2.0 → 0.3.0）：上一轮大部分剩余缺口得到解决或纠正——LiteLLM 钉版由未验证暂定升级为已确认、OTel GenAI 官方 spec 落地、Graphiti 冲突解决、LangMem 与 FrontierMemory 确认；同时纠正了两条上一轮结论（AgentExecutor 维护口径、Google 延迟表述）。严格落在 2026-10-28 当日的一手发布四轨均检索未见；落在 2026-10-27 的一手新条目为 LiteLLM 1.106.5 批与 Claude Code changelog 技能修复。

## 2026-10-27 — Round 2: gap-review round / 第 2 轮：缺口复核轮

- Gap-review round (watermark 2026-10-25 → 2026-10-27; the 2026-10-26 round was skipped): most prior second-hand conclusions were upgraded to primary sources, an earlier AgentKit conclusion was overturned by OpenAI's official deprecation notice, LiteLLM pinning was tightened to a two-stage 1.106.3/1.106.4 line, and evaluation figures were re-anchored to their conditions.
- 缺口复核轮（水位 2026-10-25 → 2026-10-27；2026-10-26 轮次跳过）：上一轮多数二手结论升级为一手来源，AgentKit 结论被 OpenAI 官方弃用公告推翻，LiteLLM 钉版收紧为 1.106.3/1.106.4 两段式，评估数字按条件重新锚定。

## 2026-10-25 — Initial publication / 首次发表

- Repository created and published: the research-evolution programme's master report and four track notes, with bilingual README and methodology/track documentation.
- Content: baseline research (2026-10-23 four-topic deep research; 2026-10-24 skills deep research) plus evolution **Round 1 (watermark 2026-10-24 → 2026-10-25)** — gateway supply-chain security as the round's largest theme; evaluation methodology tightening across three tracks; structured decision endpoints emerging in the routing layer; discovery/conflict rules continuing to diverge across skill hosts; demotion ≠ invalidation ≠ deletion clarified for layered memory.
- 本仓库建立并首次发表：调研进化总报告与四份轨道笔记，附中英双语 README 与方法论文档。内容为基线调研加第 1 轮进化小结（水位 2026-10-24 → 2026-10-25）。
