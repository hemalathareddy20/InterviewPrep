# convo-clc — Conversation Logs

Exported Claude Code sessions for this interview-prep workspace. One file per working session.

## Naming

`YYYY-MM-DD-session-NN-<short-topic>.md`

## Sessions

| # | Date | File | Topic | Ended at |
|---|---|---|---|---|
| 01 | 2026-09-23 | [2026-09-23-session-01-repo-analysis-and-prep-build.md](2026-09-23-session-01-repo-analysis-and-prep-build.md) | Full analysis of both repos; prep pack (files 00–14) written; mock interview started | Phase 26, Question 1 asked — awaiting first answer |

## Format of each log

1. **Header block** — date, tool/model, working directory, git branch, one-line outcome.
2. **Fidelity note** — user and assistant chat messages are verbatim; tool calls are summarised, not transcribed.
3. **Turn-by-turn** — `## Turn N — User` / `## Turn N — Assistant`.
   Tool activity goes in a table under the assistant turn: what was done, and the finding.
4. **State at end of session** — which phases are complete, where the mock interview stopped,
   the planned question progression, and any open items. This is what the next session reads first.

## Rules

- **Append, never overwrite.** A new session gets a new file and a new row above.
- Record findings, not raw tool output. A future session should be able to skip re-scanning the repos.
- If a session corrects an earlier factual claim, say so explicitly in the new log **and** fix it in the
  affected prep file — the prep pack is the authority, these logs are the history.
