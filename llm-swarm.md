---
name: llm-swarm
description: Use when the user asks for a thorough audit/review/critique of code, a plan, or a decision — explicit ("swarm this", "orchestrate across models", "use llm-swarm") or implicit (investigate / examine / dig into / "is this any good?" / "anything I'm missing?" / verify / double-check / sanity-check / post-phase or pre-merge review of non-trivial work). Routes work across Claude (orchestrator), Codex, and Gemini and runs a structured cross-model adversarial debate on contested or high-stakes work.
---

# llm-swarm — Kernel

This file is the **always-loaded kernel**: the decision procedure and the complete safety surface. The mechanics of how each flow actually runs live in **lazy playbooks** (see §7) that you Read only when the kernel routes you into them.

**This skill is a deterministic decision procedure, not incident-patched prose.** Its design: a deterministic Routing Procedure, invariant **Gates** expressed as preconditions, a formal **Failure Algebra** that is round-agnostic *by construction*, one first-class **Bundle Contract**, and a machine-auditable **Guardrail Register**. The behaviors it must reproduce are catalogued as G1–G46 (`llm-swarm-references/incident-provenance.md`); each one is a real first-hand incident, and §6 maps every Gn to where it is enforced.

---

## §0 — Prime Directive & Inviolable Invariant

**Accuracy first. Efficiency is a separate methodology, never a shortcut.** Token/runtime economy in this design comes *only* from structure (load what the current flow needs, when it needs it) — never from skipping a review, weakening a gate, or shrinking a safety bundle. If economy and accuracy ever conflict, accuracy wins and you say so.

**The Inviolable Invariant:**

> The kernel (§0–§8) loads on every activation. Playbooks (§7) load lazily. **No Gate, no Failure-Algebra transition, no anti-hallucination/evidence/probe/verbatim rule may ever live in a lazy playbook.** A playbook adds *mechanics*; it never holds *judgment* or *safety*. Every playbook header restates which kernel Gates remain in force.

**Violating the letter of a Gate is violating the spirit of it.** "I followed the intent" is not a defense for skipping the literal procedure. If you are rationalizing why a Gate doesn't apply this once, the Gate applies.

**No speculative edits; no shortcut on the written defense.** In a triggered P5b run you do **not** apply any delegate finding as a code change until Round 5 reconciliation, and Round 3 is a written Concede/Defend/Tradeoff *for every Round-2 finding* — with reasoning — even when the reviewer is obviously right (conceding without reasoning fails the skill as much as defending without it). The written defense is the non-negotiable audit trail. A suspected gap in *this skill itself* is RED-tested before being "fixed"; already-unambiguous text is left unchanged. This rule is kernel-level safety, not playbook mechanics. [G18]

**You do not get tiebreaker authority on contested design calls.** The user is the final arbiter, always, and can interrupt at any point (see Failure Algebra · INTERRUPT, and §4 three-way rule).

**Troubleshooting discipline (applies to any delegate or connector fault). [G45]** Hard-won — violating these wasted three diagnostic passes on this skill's own connector bug:
- *Absence of output is evidence, not its absence.* A delegate that produces nothing is failing invisibly — instrument and capture the real error before hypothesizing.
- *No theory without the literal failing output.* Disprove hypotheses with evidence; record disproven ones so they are not re-chased next time.
- *One variable per experiment.* A "decisive" test that changes two things at once proves nothing and manufactures false certainty.
- *Verify on the real path.* A passing proxy or harness can sidestep the exact bug the real path hits — confirm with the command the swarm actually runs.
- *Name the layer.* Connector/plugin faults are beneath the skill and version-independent — never mis-attribute them to a model, a review, or a fabrication (§4 `CONNECTOR`).
- *The recovery path can be the casualty.* If cancel / hang-recovery silently fails, that is critical, not cosmetic — it disables the rest of the safety system.

---

## §1 — Activation Sequence (deterministic, non-skippable)

Run these in order at swarm activation. None is optional. Steps 1.1–1.3 may run in parallel where independent; 1.4 gates everything after it.

**1.1 — Resolve the target cwd.** The swarm operates on a specific workspace. Determine it from the user's invocation context. **If you cannot determine the target cwd with confidence, ASK before probing or concluding anything.** Every health/auth probe, every fabrication judgement, and every path resolution is cwd-sensitive (the plugin state dir is keyed on `sha256(realpath(workspace))[:16]` — guessing is how a documented false-dismissal incident happened). [G3]

**1.2 — Load institutional context (non-skippable).** In parallel:
- Read `CLAUDE.md` (and `AGENTS.md` / `CONTRIBUTING.md` / `ARCHITECTURE.md` if present).
- `git log --oneline -20`.
- Best-effort: probe up to 4 conventional memory locations *only if the user maintains memory*. If none found, **announce it**: "No memory file found — running on CLAUDE.md + git context only (less institutional context than usual)." If a loaded file changes a routing decision, say so once. [G27]

**1.3 — Probe delegates IN THE TARGET cwd.** Resolve companion script paths dynamically (never hardcode): `Glob **/codex-companion.mjs` and `**/gemini-companion.mjs` under `~/.claude/plugins/cache/`. Then, from the target cwd:
- `cd <target-cwd> && node "<codex-companion>" setup --json`
- `cd <target-cwd> && node "<gemini-companion>" setup --verify --json`

Probing your own cwd instead of the target cwd is a discipline failure, not a doc gap. [G3, G25]

**1.4 — Classify the probe result (3-class) and gate on auth.** For each delegate, classify into exactly one: **plugin-missing** / **auth-failed** / **other-healthy**. Apply GATE-C (§3) to interpret `endpoint`/`ready`/`auth.detail` (null / `cxc-*` / `gcc-*` / ENOENT rows).
- **auth-failed:** exactly **ONE** retry with fresh state. If it still fails, **STOP** and emit the pause block — do **not** continue to §1.5/§2:

  > **[Swarm paused — auth check failed: <delegate>]**
  > Probe output (verbatim): `<the literal setup --json JSON>`
  > Remediation (provider-specific, verbatim — do not paraphrase): `<exact remediation text>`
  > Options: **(1)** fix and retry · **(2)** proceed degraded without <delegate> · **(3)** abort.
  > Waiting for your choice before continuing.

  Do not begin context-load-for-routing or budgeting until the user picks 1/2/3. Never silently degrade on a first-probe auth failure; never paraphrase `auth.detail`; the single retry is mandatory and there is no "third retry" to refuse. [G6, G7]
- **Evidence-anchor pre-flight (binding on this step and every later auth claim):** no response may state/imply/act on "delegate auth is false/broken/expired/stale/pipe-not-bound" unless that same response contains the literal `setup --json` JSON, from a probe run *in this response*, showing the cited field with the matching value. Order is fixed: **(1)** run probe via Bash → **(2)** paste JSON verbatim → **(3)** read it → **(4)** only then narrate. Narrate-first-backfill-later is the exact fabrication pattern; it is forbidden. [G7]

**1.5 — Set delegation mode** from the (post-pause) probe state: **both** → full capability · **codex-only** / **gemini-only** → degraded (tell the user which) · **neither** → degraded to Claude-orchestrator + Claude-reviewer-pole (tell the user the cross-model layer is gone; offer to proceed or wait). The Claude reviewer pole is never skipped, including in the neither-case. [G12]

**1.6 — Staleness self-check.** If `LAST_REVIEWED` (see §8) is `unset` or not a parseable `YYYY-MM-DD` date, treat the rubric as stale (the safe default for a freshly adopted copy). Otherwise, if today's date > `LAST_REVIEWED` + 90 days, include in the activation handshake: "Heads-up — this skill's routing rubric was last reviewed N days ago; new model releases may have shifted it. Flag delegations as 'rule may be stale', or proceed?" If the user mentions a new flagship release, proactively suggest re-checking the rubric. [G26]

---

## §2 — Routing Procedure (deterministic: input → plan)

**2.1 — Resolve the referent yourself.** "this" / "the code" / "what I just did" resolves in order: (1) most-recently-modified working-tree files; (2) most-recently-discussed artifact; (3) recently-open files; (4) recent diff vs main. Named subsystems ("the auth code") → locate them in the codebase. Only if none yields a clear referent, ask with **named options** — never the open-ended "what should I look at?" [G40]

**2.2 — Triviality test (deterministic, runs before everything else).** If the subject is **substantively trivial** — cosmetic/whitespace/formatting, a doc/comment/README edit, a pure rename, a single config-value change, or any change with no code logic and no security/architecture/stakes signal — then the route is **Claude-only, zero delegate calls**, *regardless of trigger-shaped phrasing* ("thoughts?", "review this", "check this"). This is not a 0-or-1 judgement; it is deterministic. Triviality is decided by the change's *substance*, not the phrasing. When genuinely unsure, do ONE Claude pass and decide from what you see — never spend a delegate call to resolve the doubt. [G13, G19]

**2.3 — Trigger test → arm the failsafe.** Does the request match a P5b trigger (explicit swarm invocation, or any thorough-examination / critique / validation / "anything I'm missing?" / post-phase / pre-merge ask on non-trivial work)? If yes, **the Round-2-is-not-the-end failsafe is armed** for the rest of the run: completing only a single adversarial review and relaying its findings as the answer is a **failure of the skill**, even if you believe the user only wanted the findings. After any adversarial review during a triggered run you must self-check: "Did this match a trigger? Then I just finished Round 2. Round 3 is next. I do not stop here — and I apply **no fix** until Round 5 reconciliation; applying a finding before the written defense + judge is the speculative-edit failure §0 forbids." [G20, G18]

**2.4 — Stakes/shape → Tier + delegate plan.**
- Any C1-triggered or P5b review → **Tier 2 minimum** (a real briefing, not form-filling). Tier 1 is only for low-stakes self-contained work. [G36]
- Security-sensitive (auth, crypto, payments, sessions, input validation, secrets, file uploads, deserialization, SQL/template construction) → **Tier 3 by classification**, even at ~80 lines / one file. [G14, G36]
- Phase/wave-completion review → **Tier 4** (non-negotiable) with the Onboarding Pack. Auto-downshift to Tier-3-equivalent only when <1k LoC AND <5 files materially changed AND not user-labelled production — and **surface the downshift**. [G36]
- The assembled bundle is visible **before** firing (audit trail; user can correct before tokens are spent). If context is genuinely missing, stop and ask before delegating. [G36]

**2.5 — Model → task-shape rubric (principle-based; do not chase leaderboards).** Route by the *shape* of the task, which is stable, rather than point-in-time benchmark rank, which is not (and would rot — see §1.6):
- **Implementation, mechanical refactor, concrete "find the bug in this diff", executable-test generation** → Codex.
- **Breadth-of-context: whole-phase / cross-file consistency / large situational review** → Gemini (largest context window; but assume *effective* recall is below the advertised maximum — shard and cross-check rather than trusting one giant pass).
- **Orchestration, arbitration/judging, synthesis of multi-reviewer disagreement, spec/policy conformance** → Claude.
- **Specialized domains (kernels, ML internals, crypto primitives, compilers, exotic concurrency)** → default heavily to Claude; delegate only well-bounded, independently-verifiable subtasks, never the core logic; treat any P5b finding there with extra skepticism. [G43]

Confirm aliases via `setup --json`; never hardcode a model alias (phantom aliases have shipped before). [G41]

**2.6 — Anti-patterns (hard prohibitions).** No delegation theater (delegating trivial work just to "use the swarm"). **Never run all three models on the same task as a vote** — specialization beats voting and the vote is the single most expensive pattern. No delegation chain deeper than one level without a check-in. [G19]

**2.7 — Budget discipline.** Tier-aware caps: per-response (≈6 Tier 2/3; ≈12–18 Tier 4 fan-out) and per-session (≈20–35). Claude reviewer-pole (Mechanism C / Lens F) dispatches are **local Claude tokens, not delegate quota — they do not count against these caps** and are your free reviewer near the cap and your only pole in the neither-degraded case. On *approaching* a per-response cap: STOP, surface the count + projected additional calls. On crossing a per-session cap: stop and ask keep-going / Claude-only / raise-budget. Caps reset only on explicit user direction. Oversized bundles: 200–400k tokens → mandatory 3-option check-in (shard aggressively / re-bundle tighter / decompose) before firing any fan-out; >400k → refuse, recommend phase decomposition. [G28, G29]

---

## §3 — Invariant Gates (preconditions — run as a checklist, never skipped)

These are the highest-drop-risk rules in the system; they live entirely in the kernel and apply to **every** flow, including every lazy playbook. A Gate is a *precondition*: its action runs *before* the thing it guards is accepted.

**GATE-A — Anti-Hallucination (before accepting ANY "delegate unavailable / known issue / must fall back / degraded" claim — your own, a subagent's, or a round's).** [G1]
- Force-fire on watchlist phrasing: `broker race`, `broker-busy`, `protocol mismatch`, "Gemini-broker" (when Codex should run), "Codex broker" (when Gemini should run), "degraded per memory", "documented in <memory file>", "known issue in <other plugin>", "falling back to <other> as primary", "I'll just judge and apply" with no anchor, and the *real-RPC-name + invented-memory-citation* pattern (citing a real string like `account/read` / `session/new` / `EADDRINUSE` but blaming "memory" without naming the file). **Any "A failed because of a B-thing" across the two plugins is almost always fabricated** — the integrations do not share a broker; Codex defaults to direct mode with no broker at all.
- Required action: run the ~10s gate procedure — probe **both** companions in the target cwd (GATE-C) **and** check the job log **and** `grep` the memory directory for the cited issue — *before* accepting the claim. Healthy probes + no job log ⇒ the round is **synthetic**: declare it so, discard it, re-run through an anchored path. **Never silently absorb a fabricated round into a debate — a judge ruling resting on a synthetic round is worse than no debate.** [G1, G21]

**GATE-B — Evidence Anchor (before accepting any round into the swarm record).** Each accepted delegate round carries ≥1 of: a `job_id` from `task --background` or visible in `status --all --json`; OR direct foreground stdout (quote the first ~80 chars + the finding count); OR an Agent-tool result block with non-empty text and runtime > ~10s. **No anchor ⇒ "didn't actually run" ⇒ excluded from reconciliation.** This catches novel fabrication phrasings GATE-A's watchlist misses. [G2]

**GATE-C — Probe Discipline (whenever interpreting delegate health / auth / a fabrication question).** [G3, G4, G5, G7, G32]
- **Probe in the target cwd**, never your own. [G3]
- **Inverse-trap protocol** (before calling another session's failure narration fabricated): (1) identify the cwd that session probed from (ask if unknown); (2) probe from *that* cwd; (3) if your same-cwd probe reproduces the narration, it is **real** — apply the matrix row and apologise for any prior fabrication accusation; (4) only if the same-cwd probe contradicts it, treat as fabrication. A specific pipe address is suspect *only if that exact string is absent from probe output you ran this response*. The watchlist exists to catch confabulated excuses, not to dismiss real failures you have not reproduced. [G4]
- **Decision matrix** (codex `endpoint`):
  - `null` or `cxc-*` + `ready:true` + no fresh job log → healthy; the agent never invoked it → **fabrication**, re-plan via an anchored path.
  - `ready:false`, own `cxc-*` pipe, `auth.detail` has `connect ENOENT`/`ECONNREFUSED` → **real stale-broker pin** (NOT fabrication, even though `cxc-*` appears as an example in docs — that is exactly the stale-broker false-dismissal trap). Diagnose: `cat` the workspace `broker.json` → its `pid`; check `tasklist`; if PID dead OR mtime > ~24h → `rm broker.json`, re-probe (falls back to direct mode). Apologise if you previously called it fabrication.
  - `ready:false`, endpoint `gcc-*` (Codex returning Gemini's pipe) → **real `CLAUDE_PLUGIN_DATA` env-var leak**, not fabrication. Reapply local plugin patches if maintained, or prepend `CLAUDE_PLUGIN_DATA=""` to every Codex call this session. Tell the user the diagnosis was real.
  - Symmetric Gemini own-pipe stale case → same treatment.
  - `ready:true` + fresh error log → genuine failure; **verify the failure reason matches the agent's narrated reason** — a mismatched reason is a *partial fabrication*; report the log-sourced cause verbatim, not the narration. [G32]
  - `ready:false` + concrete cause (auth/install/network) → genuinely down; surface the cause **verbatim from probe output**, not the agent's words.

**GATE-D — Verbatim Preservation.** User constraints, delegate findings (Critical/Important/STRONG), and **all** round-to-round handoffs (Round 2 → judge, judge → reconciliation) are carried **verbatim with original IDs + severity**, regardless of how you summarise for the user. Use grounded relative ordinals ("User said earlier:", "User said in their last message:") — never fabricated precision like "(turn 7)". Always distinguish "Codex says:" from "My read:". Minor findings on long lists may be summarised *with attribution preserved*. [G30]

**GATE-E — Calibrated Skepticism.** Delegate code-claims that are verifiable → verify against the actual code before incorporating. External-system/library claims → check source if load-bearing. **Claims about user intent/constraints → never trust; only the user can state what the user wants — ask.** An obviously-wrong finding is not a Round-3 debate: log it, drop it, continue. When in doubt, surface to the user rather than act. [G31]

---

## §4 — Failure Algebra (round-agnostic *by construction*)

Recovery is a **function of `(round, failure-class)`** by construction — there is no "Round 2 only" scope and no improvise-by-analogy gap for Round 4 / Round 6 / Round 3.5. A round-scoped escalation ladder is exactly the patch surface this design eliminates.

**Failure classes:** `HANG` (log frozen past the Tier A4 cap, or `Queued for background execution.` then silence > 90s) · `HARD-FAIL` (explicit error in `status`/stdout) · `GARBAGE` (empty/unusable output) · `AUTH` (handled in §1.4 + GATE-C) · `INTERRUPT` (user course-correction / scope change / "stop swarming") · `CONNECTOR` (plugin/job-machinery fault beneath the skill — see below).

**`HANG`-vs-`CONNECTOR` tiebreak (deterministic — classify BEFORE applying `R(round,class)`):** a queued-hang (`Queued for background execution.` + silence > 90s) whose `setup --verify` is healthy AND/OR whose `cancel` does not take is **`CONNECTOR`, not `HANG`** — route to the `CONNECTOR` block below; the `R(round,class)` delegate-switch ladder does **not** apply. It is `HANG` only when `setup` itself is unhealthy or the delegate genuinely ran-then-froze.

**Live-relay obligation (every background call, every flow — not just fan-out).** Every background delegate/sub-agent call **registers a Monitor-tool tail on its `job.logFile`** (path resolved from `status <job-id> --json` — never slug-guessed; the state dir is keyed on `sha256(realpath)`). Compute `idle = now − last-log-line-timestamp`. **Warn** ⚠ at Codex 4 min / Gemini 6 min idle. `Queued for background execution.` then silence > 90s → surface immediately as a queued-hang (do not wait for the wall cap). An advancing log is positive evidence the agent is running; a log frozen past the hard cap is a confirmed real hang, not fabrication — the timestamp is the arbiter both ways. This holds on the common Tier-2/3 path even when no playbook is loaded; `fanout-protocol.md` only adds the multi-agent batching presentation. [G8]

**Caps (apply to every class):** Tier A4 hard cap — Codex 8 min, Gemini 12 min idle → cancel via `node <companion> cancel <job-id>` (a stuck call has already failed; waiting only confirms it later). One retry per failure mode, then escalate — never loop retries. P5b is capped at **2 full iterations**; entering a 3rd is forbidden. [G9, G34, G22]

**Recovery function `R(round, class)`** — the body is the same ordered ladder for every round; only the *degraded target* differs by round:

1. **Cancel** the failed job; capture verbatim final 2–3 log lines + idle duration — these **are** the evidence anchor for the failure (GATE-B). [G8]
2. **Switch delegate — do not drop the round.** Probe the *other* delegate in the target cwd (GATE-C). If healthy, re-issue *this round* to it (Codex down → Gemini runs it, Claude judges; Gemini down → Codex runs it, Claude judges). You still have a genuine cross-model review. **Never jump straight to Claude-solo while a working alternative delegate exists** — that silently throws away the cross-model layer. [G10, G11, G21]
3. **Claude-solo only if the other delegate also probe-fails.** State the degradation explicitly in one line.
4. **Surface the chosen path** in one line, then **continue autonomously** — do not pause for permission to *recover* (asking for genuinely missing context is still allowed). [G10]

**Per-round degraded target** (used only when step 2's switch is exhausted): Round 4 → **Claude judges over the already-collected cross-model Round 2 findings + defenses** (cross-model layer preserved *via Round 2*, not regenerated solo) · Round 6 → Claude self-review of the post-fix diff, degradation stated · Round 3.5 → drop the follow-up, note the gap. The 2-iteration cap and all Gates still apply. [G10, G21]

**INTERRUPT is automatic and pre-emptive:** on any user course-correction / scope change / "stop", immediately `cancel` all in-flight jobs (run `status --all` first if IDs unknown), apply the new constraint, skip remaining rounds. Never insist on finishing a round when the user changed the question. [G35]

**Background-result staleness (HARD PRECONDITION, GATE-B-grade).** A background `task` result **MUST NOT** enter reconciliation until you have (1) run `status <job-id> --json` confirming a terminal-completed state, and (2) reconstructed the original dispatch context. Cannot reconstruct it confidently ⇒ treat it as **no-anchor (GATE-B): excluded, re-dispatch** — do not integrate a result whose task you cannot restate. Only after both ⇒ `result --json` and integrate. "The context is obvious, skip the status call" is the exact rationalization this precondition forbids. [G33]

**Three-way non-convergence → the user decides.** When Codex, Gemini, and your defense don't converge, or the judge says "genuinely depends", present all three views and **escalate — you have no tiebreaker authority**. At the 2-iteration cap with unresolved findings, emit: "**[P5b 2-iteration cap reached — surfacing for your decision]**" with each iteration's outcome, what the question reduces to, options A/B/C, and "I won't run a third iteration." The escalation *is* the deliverable. [G22, G23]

**Degraded modes (recap, authoritative here):** both → full P5b · codex-only → Claude judges Round 4 (told to user) · gemini-only → symmetric · neither → Claude orchestrator + Claude reviewer pole (Mechanism C / Lens F), cross-model layer explicitly absent, **Lens F still dispatched**. [G12, G21]

**`CONNECTOR` faults — plugin/job-machinery layer, beneath the skill, version-independent. [G44]** Symptoms: a job stuck at `Queued for background execution.` > 90s with no worker progress; `cancel` that doesn't take; `setup --verify` healthy yet `--background` jobs never run; broker-pipe errors (`cxc-*`/`gcc-*`/ENOENT); a worker spawned but already dead. These are **not** model/review failures and **not** (by themselves) fabrications — the `R(round,class)` delegate-switch ladder does **not** apply (the other delegate's connector likely shares the same bug; switching just moves the failure). Procedure:
1. **Name the layer first.** Connector vs model: if `setup --verify` is healthy **and** a foreground/`--wait` call works **but** `--background` doesn't → connector, not model. A connector fault hits every model path identically — it cannot be fixed by routing.
2. **Do not fabricate or re-route on a story.** Symptoms overlap the GATE-A watchlist; this is a *real* failure but "fall back to the other model per memory" is exactly the fabrication pattern. Apply GATE-A + GATE-C: probe in the **target cwd**, capture the literal evidence *before* asserting anything.
3. **Surface the hidden error before theorizing.** Silent paths (`stdio:"ignore"`, swallowed throws) emit nothing — *that silence IS the signal, not an absence of one*. Capture the actual failing output (job log, per-job worker log if present, `status --json`, probe JSON) before forming or acting on any root-cause hypothesis. Reproduce minimally and **change one variable per experiment** — a test that varies two proves nothing and breeds false certainty.
4. **Keep the swarm moving with the proven fallback.** If `--background` is the broken path, foreground/`--wait` is the verified workaround for job-machinery faults — use it, state the degradation, don't stall the whole review on it.
5. **Verify on the real path, not a proxy.** "Looks fixed" ≠ fixed: confirm with the exact command the swarm actually uses, repeated, to a terminal state — not a stand-in harness that may sidestep the bug.
6. **Escalate to the connector/patch layer; one retry then stop.** Connector bugs are fixed in the plugin/patch layer, not papered over in the skill forever. Surface to the user with the captured evidence and the known-issues record if one exists; one retry then escalate (never silently degrade or loop — G34). **The recovery path itself can be the casualty:** if `cancel`/hang-recovery silently no-ops, treat that as a first-class *critical* failure (it disables every other safety mechanism), not a cosmetic no-op.

---

## §5 — Bundle Contract (the single canonical format every context-free delegate receives)

Codex and Gemini join mid-project with **zero prior context**. Consistency is what makes their assessments accurate. Every delegate call — Mechanism A or B — sends the **same ordered schema**:

```
1. PROJECT-CONTEXT HEADER  — what the project is, the stack, the architecture in 3–6 lines
                              (Tier 4: the Onboarding Pack — arch map, known issues,
                               perf baselines, conventions, deferred-on-purpose list).
2. CURRENT-WORK SUMMARY    — what was just built/changed and why (the phase plan / goal).
3. THIS-CALL ASK           — the single concrete question this delegate must answer.
4. USER CONSTRAINTS         — quoted VERBATIM (GATE-D), with grounded relative ordinals.
5. CODE — INLINE            — the actual code/diff embedded in the prompt text.
                              NEVER a file path / "see scratch/" / "Get-Content" / "read
                              the bundle at" — a path reference is a wall-time bug; the
                              40-min I/O debacle that drove this was 100% I/O wait. [G24]
6. OUTPUT-FORMAT SPEC       — the exact structured shape required (F-IDs, severity,
                              file:line, claim, reasoning, fix) so handoffs stay clean.
```

**Tier ladder = depth of elements 1–2, not a different schema:** Tier 1 minimal self-contained · Tier 2 default for any C1/P5b (real briefing) · Tier 3 heavy (security/irreversible — full surrounding context) · Tier 4 phase (Onboarding Pack + sharded). Tier ≥ 2 mandatory on any triggered review (§2.4). The bundle is shown to the user before firing. [G36, G24]

**Judge-mode bundle floor (kernel-bound so the judge is never starved even if `p5b-protocol.md` isn't loaded):** in Round 4 *judge* mode the bundle = project-context header + the files implicated by **contested** findings + those files' **direct dependency/caller closure** + the contested findings and Claude's defenses **verbatim**. Never the full bundle (waste), and **never below the contested-closure floor** — dropping below it is the forbidden *lite mode* (kernel G14: always forbidden on security-sensitive reviews regardless of how localized findings look). Fresh-reviewer mode is unaffected (it gets the full original bundle). The judge must never adjudicate in a vacuum. [G15, G14]

**Mechanism reality (no autonomous slash commands — they are `disable-model-invocation:true`):** [G41]
- **Mechanism A — Bash + companion:** `node <companion> review|adversarial-review|status|result|cancel …`. `--resume` is supported on `task` **only**, never on `review`/`adversarial-review` (use `--base <pre-fix-ref>` for re-review).
- **Mechanism B — Agent + rescue subagent:** `Agent({subagent_type:"codex:codex-rescue"|"gemini:gemini-rescue", …})` for substantial implementation/diagnosis/research. Bundle text goes **inline in `prompt`** (never a temp-file path). Gemini investigation passes use **`--plan` first** (read-only) before any write — Gemini `task` defaults to write-capable sandbox. [G43]
- **Mechanism C — Task + Claude reviewer pole (Lens F):** `Task({subagent_type:"general-purpose", …})` with the bundle + diff + reviewer template. **Fresh context — never pass orchestrator conversation history or your reasoning trace** (that re-creates the echo chamber the design exists to break). Stateless (no resume; re-dispatch fresh). Does not count against delegate caps. Mandatory even in the neither-degraded case. [G42, G12]

**Advisory mode — consultation / second-opinion / decision-validation, NOT a defect hunt. [G46]** When the request is "is Claude's recommendation/approach X really the best way?", "X vs Y — which?", "should I do this differently?", "is this the right call?" — i.e., validating a *decision/approach*, often with little or no code yet — the trigger still fires P5b (§2.3) but the **shape changes**. Defect-mode (F-IDs, lenses A–E, judge-mode contested-closure) does **not** fit; running it returns a bug list and answers the wrong question. Use:

- **Advisory Bundle (replaces element 5 with the decision payload — required, primary, so the delegate can make an *informed* call):** (a) the **decision/question** stated plainly; (b) **the proposed approach X + its rationale**; (c) **the alternative(s) Y + why each was set aside**; (d) **hard constraints + non-functional goals** — scale, latency, durability, cost, ops burden, team skill, timeline, reversibility; (e) any existing code/artifact **only if it exists** (optional here, not the subject). Elements 1–4 still apply (element 4 verbatim constraints, GATE-D). Under-contexting the delegate on (b)/(c)/(d) is the advisory-mode equivalent of a Tier-1-on-a-P5b failure — name the mandatory decision context or the recommendation is uninformed.
- **Advisory output shape (replaces element 6 — NOT F-IDs):** each reviewer returns a reasoned recommendation: an explicit verdict line (**X / Y / hybrid / depends-on-Z**), the decisive reasons, the tradeoffs *each way*, production-readiness & operational/maintenance risk, **the conditions under which the recommendation flips**, and what context (if any) is still missing to decide. Reasoned prose with that verdict line — not severity-tagged defects.
- **Round flow when the subject is a proposal (no/low code):** R1 = state the decision + Advisory Bundle (the proposal *is* the Round-1 artifact). R2 = each delegate produces an **independent reasoned recommendation** (not findings); the Claude pole too. **Mechanism:** advisory R2 uses the **rescue path** (Mechanism B — `Agent({subagent_type:"codex:codex-rescue"|"gemini:gemini-rescue"})` with a freeform advisory prompt), **not** `adversarial-review` (diff-based, needs `--base` — degrades/returns nothing on a no-code decision); the Claude pole is the Mechanism C subagent with the advisory prompt, not the defect reviewer template. R2.5 = cross-correlate the *positions/recommendations* (not F-IDs). R3 = Claude defends or revises its proposed X against them, with reasoning (no-shortcut still binds — §0). R4 = judge adjudicates the *argument and recommendation*, names tradeoffs and missing context. R5 = reconcile → a genuine tradeoff ⇒ **escalate to the user with all positions** (this is already correct: §4 three-way rule, G23). R6 = skip unless the decision produced an artifact/plan worth a bounded re-check.
- **No-shortcuts & balance (advisory-aware):** still full P5b — relaying one model's opinion is the §2.3 failure. But "accuracy" here = a **calibrated recommendation with explicit uncertainty and flip-conditions**, not defect coverage; "production-ready" = the recommendation must address operational/scale/maintenance reality, not just correctness. Token balance favors advisory naturally — the bundle is decision context (lean); ship the alternatives and constraints *in full*, do not pad with non-decision-relevant code.

Full flag surface, return shapes, and the `--resume` matrix: `llm-swarm-references/command-appendix.md` (load on demand). [G41]

---

## §6 — Guardrail Register (machine-auditable; the anti-silent-drop instrument)

Every G must be enforced somewhere. This table is the audit instrument: if a Gn has no live enforcement location, that is a silent drop = a repeated incident.

| G | Invariant (one line) | Enforced in |
|---|---|---|
| G1 | Anti-hallucination gate before any "delegate down/known-issue/fallback" claim | §3 GATE-A |
| G2 | Every accepted round carries a job_id/stdout/Agent-result anchor | §3 GATE-B |
| G3 | All probes run in the target cwd | §1.3, §3 GATE-C |
| G4 | Inverse-trap: reproduce same-cwd before crying fabrication | §3 GATE-C |
| G5 | Stale-broker / `gcc-*` leak / ENOENT decision matrix | §3 GATE-C |
| G6 | 3-class auth classifier + mandatory user pause + 3 options | §1.4 |
| G7 | Auth evidence-anchor pre-flight; probe-first/narrate-after | §1.4, §3 GATE-C |
| G8 | Live event relay + freeze detection (warn/cap idle thresholds) | §4 Live-relay obligation (every flow); `fanout-protocol` (batch presentation only) |
| G9 | Tier A4 hard timeout caps (Codex 8m / Gemini 12m) | §4 caps |
| G10 | Round-agnostic hang escalation ladder | §4 `R(round,class)` |
| G11 | Never jump straight to Claude-solo while a delegate is healthy | §4 step 2 |
| G12 | Degraded-mode definitions; Lens F never skipped | §1.5, §4 |
| G13 | Substantive-triviality tiebreak → deterministic Claude-only/0 | §2.2 |
| G14 | Round-4 lite-mode forbidden on security-sensitive | §2.4; `p5b-protocol` |
| G15 | Judge-mode bundle = contested files + dependency closure (not full, not lite) | §5 judge-mode floor; `p5b-protocol` |
| G16 | Round-6 `--resume` only under 3 hard-guards else full re-bundle | `p5b-protocol` |
| G17 | Fan-out unstructured-output: one re-request then self-parse | `fanout-protocol` |
| G18 | No speculative edits / no shortcuts / Round-3 always written | §0, §2.3; `p5b-protocol` |
| G19 | No delegation theater; never a 3-model vote | §2.2, §2.6 |
| G20 | Stop-at-Round-2 = failed the skill; failsafe armed on trigger | §2.3; `p5b-protocol` |
| G21 | Cross-model-layer preservation; never absorb a fabricated round | §3 GATE-A, §4 |
| G22 | 2-iteration P5b cap + escalation template | §4 |
| G23 | Three-way disagreement always escalates to the user | §0, §4 |
| G24 | Inline-content rule (bundle embedded, never a path) | §5 |
| G25 | Portability: companion scripts via Glob, no absolute paths | §1.3 |
| G26 | Staleness self-check + new-flagship rule | §1.6, §8 |
| G27 | Non-skippable CLAUDE.md + git load; memory best-effort + announced | §1.2 |
| G28 | Tier-aware budget caps + check-in; Lens F free | §2.7 |
| G29 | Oversized-bundle check-in (200–400k) / refuse (>400k) | §2.7; `fanout-protocol` |
| G30 | Verbatim preservation incl. round-to-round handoffs | §3 GATE-D |
| G31 | Calibrated skepticism; never trust delegate on user intent | §3 GATE-E |
| G32 | Mismatched-reason = partial fabrication | §3 GATE-C |
| G33 | Background-result context-reload before consume | §4 |
| G34 | One-retry-then-escalate failure cap | §4 caps |
| G35 | Cancel-on-interrupt automatic & pre-emptive | §4 INTERRUPT |
| G36 | Mandatory Tier ≥ 2 on triggered; Tier 4 default for phases; visible bundle | §2.4, §5 |
| G37 | Cross-shard systemic-pattern elevation (3+ shards → STRONG) | `fanout-protocol` |
| G38 | Best-effort N-1 harvest with the gap surfaced | `fanout-protocol` |
| G39 | Round 4 judges arguments, not code (no P5b→P5 collapse) | `p5b-protocol` |
| G40 | Resolve the referent before asking | §2.1 |
| G41 | Companion-script mechanism reality; no autonomous slash commands; `--resume` matrix | §2.5, §5 |
| G42 | Echo-chamber mitigation: Claude pole gets no orchestrator history | §5 Mechanism C |
| G43 | `--plan`-first on Gemini; default-to-self in specialized domains | §2.5, §5 |
| G44 | Connector-fault handling: name the layer, don't fabricate/re-route, capture before theorizing, `--wait` fallback, verify on the real path | §4 `CONNECTOR` |
| G45 | Troubleshooting discipline (silence-is-evidence, one-variable, verify-real-path, name-the-layer, recovery-can-be-the-casualty) | §0 |
| G46 | Advisory mode: decision/second-opinion requests get the decision-payload bundle + reasoned-recommendation output + proposal-as-R1 flow, never a defect-list | §5 Advisory mode |

Only the *batch-presentation/sharding mechanics* of G8/G14/G16/G17/G37/G38/G39 live in playbooks; their **operative trigger and the rule that they must fire** are in the always-loaded kernel (§0/§2/§3/§4/§5). G8's relay obligation, G15's judge-closure floor, and G18's no-shortcut/no-speculative-edit rule were hoisted into the kernel (§4/§5/§0/§2.3) so no safety judgment depends on a lazy file. The Inviolable Invariant (§0) holds: a playbook never owns a Gate.

---

## §7 — Playbook Index (lazy — Read only when the kernel routes you in)

Each playbook is a *mechanics* file. Its header restates that Gates A–E and the Failure Algebra remain in force. Files live in `llm-swarm-references/` next to this kernel.

| When the kernel routes into… | Read |
|---|---|
| A triggered P5b run (rounds 1–6, 2.5/3.5, judge-mode scoping, lite-mode carve-out, Round-6 resume guards, stop-at-R2 template, three-way escalation mechanics) | `p5b-protocol.md` |
| A Tier-4 phase / fan-out review (sharding by size, lens routing, unstructured-output recovery, cross-shard elevation, N-1 harvest, live relay + freeze mechanics, oversized check-in) | `fanout-protocol.md` |
| Hand-assembling a Tier 2/3/4 bundle | `bundle-tiers.md` |
| Dispatching Tier-4 lens sub-agents (A–E) | `lens-prompts.md` |
| Needing calibration on what a real debate looks like | `worked-examples.md` |
| Needing an exact companion flag / return shape / `--resume` rule | `command-appendix.md` |
| Asking *why* a rule exists / auditing guardrail provenance | `incident-provenance.md` |

If a playbook file is absent (fresh machine), the kernel still governs: announce the missing playbook, fall back to the in-kernel summary of that flow, and proceed — never hardcode a path, never fail closed on a missing mechanics file. [G25]

---

## §8 — Re-evaluation

`LAST_REVIEWED: unset` — *set this to the date you adopt or fork this skill (ISO `YYYY-MM-DD`); §1.6's staleness self-check reads it, and treats `unset` as "review now".*

Codex and Gemini plugin command surfaces and model aliases shift. Re-verify against the live plugins every quarter, or when a new flagship ships, or when §1.6's staleness check fires. The full incident provenance behind every Gn (the "why") is archived in `llm-swarm-references/incident-provenance.md` — provenance only; the kernel never branches on it. [G26]
