# Methodology and Maintenance / 方法与维护机制

## Evidence standards / 证据标准

1. **Primary sources first / 一手优先.** Official documentation, papers, changelogs, and release notes outrank blog summaries, aggregators, and social posts. A claim sourced only second-hand is recorded as *unverified* （未核实） until checked.
2. **Honest windows / 诚实窗口.** Each round states its watermark window (from–to). If nothing genuinely new was released in the window, the round says so; baseline backfill is labelled as backfill, never presented as news.
3. **Vendor numbers are claims, not facts / 厂商数字只是说法.** Star counts, savings percentages, and benchmark scores reported by vendors are snapshots or self-reports; they are labelled unverified and re-checked against the original repository or paper before being relied on.
4. **Pinned evaluations / 评估必须锁定条件.** An evaluation result is comparable only with its model, harness version, task-set version, resource limits, and run count attached. Results missing those are noted as not comparable.
5. **Verified increments only / 只同步已核实增量.** Conclusions that pass verification are promoted into the programme's design references; unverified items stay in the per-round gap list for the next round to re-check.

## The daily cycle / 每日循环

| Step | What happens |
| --- | --- |
| 1. Research round （调研轮次） | At about 09:50 Asia/Shanghai, four tracks run in parallel: check for new developments since the last watermark, deep-dive one rotating sub-topic per track, append the dated round section to each track note and the master report. |
| 2. Collect and classify （整理归类） | A maintenance round collects the day's research output and confirms every changed file belongs to one of the four tracks or the master report — nothing else enters this repository. |
| 3. Sync and document （同步与记档） | Changed research files are copied verbatim into `research/`; the README status table and `CHANGELOG.md` are updated when content changed. |
| 4. Push and verify （推送与核验） | The update is committed and pushed, then verified: the remote head commit must equal the local one, and the remote file list must match the published set. Only a verified round counts as complete. |
| 5. Report （汇报） | Rounds that changed something are summarised for the repository owner in Chinese; failures and verification mismatches are always reported, never silently retried away. |

## What never enters this repository / 永不进入本仓库的内容

- Credentials, tokens, account material, or anything from private working directories.
- Source code, installable packages, and unpublished drafts from other projects.
- Conclusions presented as verified when they are not.

调研在约 09:50（Asia/Shanghai）运行，维护与发表紧随其后：整理归类 → 同步记档 → 推送 → 逐项核验远端一致 → 有变化才汇报。任何一步核验不过，都会如实报告，不会悄悄重来掩盖。
