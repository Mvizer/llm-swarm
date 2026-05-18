# Playbook — Worked Examples (calibration)

> Calibration only; kernel Gates govern. These show what a *real* debate looks like vs. rubber-stamping. In both: no model dominates, the judge rules per-finding, and the user is the final tiebreaker on "genuinely depends."

## Example 1 — Security debate (JWT refresh-token rotation, Tier 3)

- **Round 1:** referent = the 4-file diff implementing refresh-token rotation. Security-sensitive → Tier 3 by classification (kernel §2.4); lite mode forbidden (G14).
- **Round 2 (dual-pole):** Codex `adversarial-review` + Claude Lens F in parallel. Codex `F-S1 [STRONG]`: old refresh token not invalidated on rotation → replay window. `F-S2 [MEDIUM]`: rotation not atomic (issue-new-then-revoke-old; crash between = two live tokens). Lens F: `F-V1 [Important]` agrees on F-S1, raises a new one — no per-user rotation rate limit.
- **Round 2.5:** matrix — F-S1 corroborated by Codex+Lens F (STRONG/consensus); F-S2 Codex-only; rate-limit Lens-F-only.
- **Round 3 (written defense):** F-S1 **Concede** (real; fix: revoke-old in the same tx). F-S2 **Defend** ("the revoke-old is in the same DB tx; no crash window" — with the code). Rate-limit **Tradeoff** ("out of scope this phase; tracked").
- **Round 4 (Gemini judge, full bundle — security ⇒ no lite mode):** F-S1 → reviewer right, apply. F-S2 → **Claude's defense holds** *only if* the tx is truly atomic — Gemini notes the ORM call is not in a transaction block in the shipped code → reviewer was right after all. Rate-limit → "genuinely depends on the threat model."
- **Round 5:** F-S1 apply; F-S2 apply (judge overturned the defense on evidence); rate-limit → **escalate to user** (no tiebreaker authority, G23) with all three views.
- **Round 6:** post-fix diff re-reviewed; F-S1/F-S2 closed.

Lesson: the defense was *wrong* and the judge caught it on evidence — that is the debate working. Claude conceding gracefully (not defending its prior take) is correct.

## Example 2 — Architecture debate (event-sourcing vs CRUD, Tier 3)

- **Round 2:** Codex argues CRUD is sufficient and event-sourcing adds operational burden with no current consumer. Lens F flags the migration has no rollback path.
- **Round 3:** Claude defends event-sourcing (audit requirement in the phase plan) but **concedes** the missing rollback.
- **Round 4 (Gemini judge):** "Both partly right — the audit requirement is real (Codex missed it in the plan), but the event store as designed can't be replayed (Codex's operational concern is valid). **Genuinely depends** on whether audit-replay is a hard requirement."
- **Round 5:** rollback fix applied (consensus); event-sourcing-vs-CRUD → **escalate to the user** — it is a product/architecture call the swarm has no authority to settle.

Lesson: the judge naming "genuinely depends" is not a failure to decide — it is the correct output, and it routes to the user. A swarm that forced a verdict here would be exceeding its authority.

## What rubber-stamping looks like (anti-calibration)

- Codex: "looks good, minor nits." Claude: "agree, shipping." No defense written, no judge call, no user decision. → **This is a failed P5b** (G18/G20). If every finding is WEAK and every defense is one line of agreement with no reasoning, you have not run a debate — re-check whether the bundle was under-tiered (G36) or the run collapsed to P5 (G20/G39).
