# Playbook — Bundle Tiers (assembly detail)

> Mechanics only; kernel §5 Bundle Contract + Gates govern. The schema (6 ordered elements) is invariant across tiers — a tier only changes the *depth* of elements 1–2 and whether the code is sharded. Tier ≥ 2 is mandatory on any C1/P5b trigger (kernel G36). The assembled bundle is shown to the user **before** firing (audit trail).

The 6 elements (kernel §5): 1 project-context header · 2 current-work summary · 3 this-call ask · 4 verbatim user constraints · 5 code INLINE · 6 output-format spec.

## Tier 1 — Light (low-stakes, self-contained only)

Elements 3 + 5 + 6, minimal. Never for a C1/P5b trigger — that is a quality failure (kernel G36). If you reach for Tier 1 on a triggered review, you mis-classified; go to Tier 2.

## Tier 2 — Standard (DEFAULT for any C1 trigger / any P5b review)

All 6 elements, real briefing depth: element 1 = 3–6 lines (what the project is, stack, the architecture touching this change); element 2 = what changed and why; element 5 = the diff + the directly-adjacent code a reviewer needs to judge it (not just the changed lines). A reviewer with zero prior context must be able to assess accurately from the bundle alone.

## Tier 3 — Heavy (security-sensitive / irreversible / user-labelled production)

Tier 2 + full surrounding context for the contested area: the called/calling code, the data model, the trust boundary, the relevant config. Security reviews are Tier 3 *by classification* even at one small file (kernel §2.4, G14). The judge/reviewer must never have to guess context that determines whether a finding is exploitable.

## Tier 4 — Phase (DEFAULT for any phase/wave-completion review; non-negotiable)

Tier 3 depth + the **Onboarding Pack** as element 1: architecture map · known-issues list · perf baselines · conventions · the deferred-on-purpose list (so reviewers don't re-flag intentional debt) · the phase plan as element 2. Then sharded per `fanout-protocol.md`. Auto-downshift to Tier-3-equivalent only when **<1k LoC AND <5 files materially changed AND not user-labelled production** — and the downshift is surfaced to the user. Assemble via the workspace `bundle-phase.mjs` if present (`Glob **/bundle-phase.mjs`); if absent, build it by hand from the 6 elements and say so (kernel G25) — never hardcode a path, never fail closed.

## Anti-patterns (kernel Gates restated)

- Code as a path reference instead of inline → wall-time bug, reject pre-fire (G24).
- Paraphrased user constraints → GATE-D violation; quote verbatim with grounded ordinals.
- Tier 1 on a triggered review → G36 quality failure.
- Bundle not shown before firing → lost audit trail; the user can no longer correct before tokens are spent.
- Missing genuinely-required context → stop and ask the user; do not delegate a guess.
