# Playbook — P5b Protocol (mechanics only)

> **Kernel still governs.** Gates A–E (§3) and the Failure Algebra (§4) of `llm-swarm.md` remain in force on every round here. This file adds *how the rounds run*; it owns no judgment and no safety rule. If anything here seems to conflict with a kernel Gate, the Gate wins.

P5b is a six-round core adversarial debate plus two conditional sub-rounds. Sequence: **1 → 2 → (2.5 if ≥2 reviewers fresh-reviewed) → 3 → (3.5 if ≤3 follow-ups warranted) → 4 → 5 → 6**.

The point of P5b is structured disagreement between independent reviewers, adjudicated, with the user as final tiebreaker. Rubber-stamping ("looks good, ship it") is a failed debate. Worked calibration: `worked-examples.md`.

## Round 1 — Subject identification

New work: it's built, code committed/staged. Evaluation request: the artifact already exists — Round 1 is the act of stating exactly what is under review (referent resolved per kernel §2.1) and assembling the Bundle (kernel §5 + `bundle-tiers.md`).

## Round 2 — First adversarial review

- **Tier 2/3 (diff-based):** dual-pole — Codex `adversarial-review` (Mechanism A) fired **in parallel** with the Claude reviewer pole (Mechanism C, Lens F) in the same orchestrator message. Same inline bundle to both. If Round 4 will be Gemini fresh-reviewer mode, fire Codex R2 + Gemini R4 + Claude Lens F in parallel.
- **Tier 4 (phase):** fan-out mode — see `fanout-protocol.md`.
- Collect findings. **Do not act on them, do not fix anything, do not relay them as the answer.** [Kernel G20: the failsafe is armed — finishing here is failing the skill.]
- Post-Round-2 response template (the audit trail that catches the stop-at-R2 failure): four anchored sections — (1) Codex findings **verbatim** with original IDs+severity (GATE-D); (2) Round 2.5 matrix if it fires; (3) Round 3 defenses; (4) the Round 4 judge call. Adjacent prose/summaries are fine *as long as the four sections appear and findings are not silently truncated*.

## Round 2.5 — Cross-correlation (conditional)

Fires when ≥2 reviewers (Codex, Gemini, Claude Lens F) produced **independent fresh reviews** of the same bundle. Because Lens F runs by default, this is effectively always present unless Lens F was skipped or both delegates ran judge-only. Build one markdown table: ID(s) | mechanism / file:line | Codex | Gemini | Claude(Lens F) | consensus | action. **SKIP** when only one reviewer produced findings, or Round 4 is standard judge mode (judge output is not a fresh review and does not enter the matrix). [G37 elevation is applied here for fan-out — see `fanout-protocol.md`.]

## Round 3 — Written defense (REQUIRED, no shortcut)

For **every** Codex finding, an explicit written response: **Concede / Defend / Tradeoff**, each with reasoning (2–3 sentences). Conceding without reasoning is as much a failure as defending without reasoning. There is no shortcut even when Codex is obviously right — the written defense is the audit trail. Obviously-wrong findings are not debated here (kernel GATE-E: log + drop + continue). [G18]

## Round 3.5 — Targeted follow-up (conditional, ≤3 questions/debate)

For findings only one reviewer raised, or a coverage gap neither addressed, you may issue a *single targeted question* to the silent delegate. Max 3 per debate; they count against budget caps. Don't follow up on WEAK findings; don't use it to whittle delegates into agreement.

## Round 4 — Judge (default) OR fresh-reviewer (announced mode)

**Default = judge mode (~80%).** Gemini reads Codex's findings + Claude's defenses and rules **per finding**: stronger side / both-missed / depends-on-X. **Gemini does NOT produce its own findings list in judge mode** — giving it a Round-2-style "review this code" prompt collapses P5b into P5. Announce the mode explicitly so the reader never guesses ("Round 4 — Gemini as judge" / "— Gemini as fresh reviewer (domain-advantage)"). [G39]

**Judge-mode bundle scoping [G15] — structural, NOT lite mode:** send the **project-context header + the files implicated by CONTESTED findings + those files' direct dependency/caller closure** (any file a contested finding's code reads, calls, or whose contract it implements) **+ Codex's contested findings verbatim + Claude's defenses verbatim**. Conceded/uncontested findings and their files are not shipped. The judge must never adjudicate in a vacuum — it sees every file any contested finding depends on, and no more. Fresh-reviewer mode is unaffected: it gets the full original bundle.

**Lite mode is FORBIDDEN [G14]** whenever: the review is security-sensitive (auth/crypto/payments/sessions/input-validation/secrets/file-uploads/deserialization/SQL-or-template construction) — *regardless of how localized contested findings look and regardless of whether the user said "production"*; OR any contested finding is architectural/cross-file; OR user-labelled high-stakes; OR Gemini is in fresh-reviewer mode; OR this is iteration 2. When in doubt, full bundle — the savings on one judge call are not worth a weaker judgment. (Judge-mode scoping above is the *default* economical path and is itself not lite mode; lite mode = stripping context below the contested-closure floor, which is what's forbidden.)

A hung/failed Round 4 follows the kernel Failure Algebra `R(4, class)` — cancel → switch delegate (Codex judges if Gemini down) → only then Claude-as-judge over the already-collected cross-model Round 2 findings+defenses. Never jump straight to Claude. [Kernel G10/G11/G21]

## Round 5 — Reconciliation (3 outcomes per finding)

Per finding: **apply** (judge favored the reviewer / both-missed → fix it) · **reject with reasoning** (judge favored the defense) · **escalate** (judge said "genuinely depends" → present Codex view + Claude defense + Gemini verdict to the **user**; you have no tiebreaker authority [G23]). Never silently accept because the other delegate agreed, nor silently reject because it disagreed.

## Round 6 — Bounded re-review (only if Round 5 changed code)

Re-review the post-fix diff once. Tier 4: rescue mechanism with a refreshed bundle of post-fix code.

**`--resume` optimization [G16] — permitted ONLY when ALL three hold:** (1) same delegate (Codex R2 → Codex R6); (2) the R2 `session_id`/`thread_id` is still resolvable via `status --json`; (3) `status` does **not** report the session compacted/expired/evicted. Then send only the post-fix diff + the applied/rejected finding list (skips the 30–80k re-bundle). **If any condition is unmet or unconfirmable → full re-bundle; never resume an unverified session.** A resumed Round 6 that references code not in the diff or invents file content is a **fabricated round** — kernel GATE-A applies: discard, re-run with the full bundle. `--resume` is on `task` only, never `review`/`adversarial-review` (use `--base <pre-fix-ref>`). [G41]

## Iteration cap

At most **2** full P5b iterations. A 3rd is forbidden — models that haven't converged in two won't in three. At the cap with unresolved findings, emit the kernel §4 escalation template and stop. The escalation *is* the deliverable. [G22]
