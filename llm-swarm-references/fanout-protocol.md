# Playbook — Tier-4 Fan-out (mechanics only)

> **Kernel still governs.** Gates A–E (§3) and the Failure Algebra (§4) of `llm-swarm.md` remain in force. This file adds *how a large phase review shards and synthesizes*; it owns no judgment and no safety rule.

Fan-out is Round 2 for a Tier-4 phase review: multiple specialized lens sub-agents fired in parallel, then synthesized. It exists because some defect classes (a systemic pattern repeated across modules) are invisible to any single-agent review — only fan-out + synthesis can catch them.

## Step 1 — Size → sharding strategy

| Bundle size | Strategy | Delegate count |
|---|---|---|
| < 40k (small phase) | Lens fan-out on full bundle: lenses A–E + Lens F on full bundle | 5 delegate + 1 Claude |
| 40–100k (typical wave) | Module shards × lens-blend [security+reliability+perf+tests] + 1 Gemini architecture cross-cutter on full bundle + Lens F on full bundle | 4–7 delegate + 1 Claude |
| 100–200k (large phase) | Each module shard reviewed twice (security+reliability, then perf+tests) + 1 Gemini architecture cross-cutter + Lens F on full bundle | 7–11 delegate (Tier-4 cap 12) + 1 Claude |
| 200–400k | **MANDATORY user check-in** [kernel G29]: surface 3 options — (a) shard aggressively (12+ sub-agents, ~30 min, expensive), (b) re-bundle with tighter `--include`, (c) decompose the phase. Lens F still fires once on full bundle if approved. | 12+ if approved + 1 Claude |
| > 400k | **Refuse** — recommend phase decomposition. Unreviewable as one bundle. |

Claude Lens F **always** gets the full bundle (it produces the one ship/no-ship verdict) and never counts against delegate caps. Print the **visible shard plan before firing any sub-agent** — audit trail; the user can interrupt before tokens are spent. Within-file sharding additionally fires for any single file > ~5000 lines.

## Step 2 — Per-lens context (non-code slicing only; code is never sliced)

Every lens gets the full changed code + adjacent code + known-issues list + conventions (universal — slicing code or known-issues regresses recall / re-flags deferred items). Lens-specific non-code context: perf baselines → Performance lens only; full test source → Tests lens only; deployment context → Architecture + Tests; full phase plan + arch map → Architecture lens (others get the one-paragraph goal). [Lens A–E prompts: `lens-prompts.md`; Lens F is the Claude reviewer pole, kernel §5 Mechanism C.]

## Step 3 — Live event relay + freeze detection [kernel G8, G9]

Every background sub-agent/delegate call **registers a Monitor-tool tail on its job log**. Resolve the log path from `node <companion> status <job-id> --json` → `job.logFile` (absolute). **Never slug-guess the path** — the state dir is `<slug>-<sha256(realpath(workspace))[:16]>`; guessing it is the cwd-mismatch incident class.

- Surface a condensed feed (Tier 2/3: one line per tool transition + ≤1 reasoning headline / 90s) or a batched refreshable status block (Tier 4: state-transitions only — per-line on a 12-agent fan-out is a notification storm).
- `started ✓` (Codex `Thread ready` / Gemini `Session ready`) is the load-bearing "it is alive" anchor.
- `idle = now − last-log-line-timestamp`. **Warn** ⚠ at Codex 4m / Gemini 6m. **Hard cap** (cancel + Failure Algebra) at Codex 8m / Gemini 12m. `Queued for background execution.` then silence > 90s → surface immediately as a queued-hang; do **not** wait for the wall cap.
- An advancing log is positive evidence the agent is genuinely running (refutes "delegate never ran" fabrication). A log frozen past the cap is a **confirmed real hang**, not fabrication. The timestamp is the arbiter both ways. When you cancel a hung job, its final 2–3 log lines + idle duration **are** the GATE-B evidence anchor — quote them verbatim.

## Step 4 — Unstructured-output recovery [kernel G17]

If a sub-agent returns prose with **no `F-<lens><n>` IDs**:
1. Re-request **only that one lens once** — re-state the structured OUTPUT FORMAT verbatim + the same bundle/shard. Other lenses' results stand; do **not** re-run the whole fan-out.
2. If still unstructured: **self-parse** the prose — extract each discrete issue, assign synthetic IDs `F-<lens>?<n>` (`?` = orchestrator-normalized), set a conservative inferred severity, tag `[severity inferred — lens returned unstructured prose]`.
3. Carry the normalized findings through dedup / cross-shard detection / the Round 2.5 matrix with the caveat visible.

Re-requesting more than once, or re-running other lenses, is forbidden — that is the over-correction this rule prevents.

## Step 5 — Synthesis

1. Collect all structured findings.
2. **Lens F verdict surfaced prominently** — the single structured ship/no-ship signal.
3. **Deduplicate** cross-shard duplicates: merge, keep the more detailed write-up, tag both sources.
4. **Cross-shard systemic elevation [kernel G37]:** if **3+ shards or lenses (including Lens F)** independently flag the same finding *class* (missing input validation, panic on bad input, non-atomic file op…), elevate that class to **STRONG as a SYSTEMIC issue**. Single-agent review cannot see this; only fan-out can.
5. **Best-effort N-1 harvest [kernel G38]:** if one sub-agent exceeds the Tier A4 cap, cancel it, synthesize from the N-1 returned, and **visibly note the gap** ("Sub-agent C (performance, shard 3) timed out after 12 min — proceeding with 5/6; perf coverage for dsp/ is incomplete; flag for follow-up"). Never present an N-1 result as complete.
6. Unified Round 2 finding list classified by lens + module shard, ready for Round 2.5.

Then re-enter the P5b protocol at Round 2.5 (`p5b-protocol.md`).
