# llm-swarm v2

A Claude Code **slash-command skill** that orchestrates a structured, cross-model adversarial review across Claude (orchestrator), OpenAI Codex, and Google Gemini. It routes a code / plan / decision audit to the right model(s), runs a multi-round adversarial debate on contested or high-stakes work, and reconciles the result — with hard guardrails against silently skipping review, fabricating delegate output, or draining your rate limits.

---

## From zero to a working swarm — A to Z

**Baseline assumption:** Claude Code is already installed and working, and you have a **current LTS Node.js** available (the two delegates are CLI tools installed via `npm`). If an `npm install -g` step below fails with an "unsupported engine" / Node-version error, upgrade Node and retry — that is the first thing to rule out.

You set up the swarm by completing three things, in order:

| Step | What | Page |
|------|------|------|
| 1 | Install + authenticate the **Codex** delegate, install its Claude Code plugin | [`install-codex.md`](./install-codex.md) |
| 2 | Install + authenticate the **Gemini** delegate, install its Claude Code plugin | [`install-gemini.md`](./install-gemini.md) |
| 3 | Drop in the **llm-swarm v2 skill** (this repo's `llm-swarm_v2.md` + `llm-swarm_v2-references/`) and run it | *this README, below* |

Each install page is self-contained: install the native CLI, authenticate **in that CLI** (the Claude Code plugin reuses those credentials), then install the plugin. Do step 1, then step 2, then step 3.

---

## Step 3 — install the skill

Copy the skill into your Claude Code commands directory (`~/.claude/commands/`; on Windows that is `%USERPROFILE%\.claude\commands\`):

```
~/.claude/commands/
  llm-swarm_v2.md
  llm-swarm_v2-references/
    p5b-protocol.md
    fanout-protocol.md
    bundle-tiers.md
    lens-prompts.md
    command-appendix.md
    worked-examples.md
    incident-provenance.md
```

> **The `llm-swarm_v2-references/` directory name is load-bearing.** The kernel resolves playbooks by that exact relative path. Renaming the file or the directory will break lazy playbook loading. If you rename the command file, keep the `<name>-references/` convention and update the kernel's playbook pointers (§7) accordingly.
>
> The skill's invocation name comes from the YAML `name:` field in the frontmatter (`llm-swarm-v2`), not the filename. Claude Code surfaces it as a slash command — invoke it the way your Claude Code version exposes named skills (e.g. `/llm-swarm_v2`, or just ask for an "llm-swarm" review).

**Start a fresh Claude Code session** so the new command is picked up (user commands in `~/.claude/commands/` are loaded at session start; `/reload-plugins` is for *plugins* and will **not** pick up a hand-copied command). **Your first run is the real end-to-end test:** point the skill at something to review. On activation it runs its own Codex and Gemini health probes and tells you plainly if either delegate is unreachable or unauthenticated — so a clean first run confirms steps 1 and 2 as well. If it instead reports that a *playbook* file is missing, the `llm-swarm_v2-references/` directory isn't a sibling of `llm-swarm_v2.md` in the commands dir — fix the layout (the skill keeps working in a reduced-detail mode meanwhile, by design).

---

## What it does

- **Routes** an audit/review/consultation request to Claude alone or across Claude + Codex + Gemini, based on stakes and scope.
- **P5b adversarial debate:** a 6-round Concede/Defend/Tradeoff protocol between models, with cross-correlation and a separate judge round, so one model's blind spot doesn't become the answer.
- **Fan-out review:** parallel lens-specialized sub-agents (security, correctness, performance, …) with synthesis, for large surfaces.
- **Advisory mode:** handles "Claude is telling me X — is this really the best approach?" as a first-class decision-validation flow (reasoned recommendation with tradeoffs and flip-conditions), not a bug hunt.
- **Self-healing by design:** transient delegate/connector faults (a hung background job, a dropped broker, a momentary auth lapse) are a first-class failure class in the kernel. The swarm detects them, never fabricates around them, and auto-recovers — retry, `--wait` fallback, switch delegate, or fall back to the always-available Claude reviewer pole — surfacing what happened instead of silently degrading. **You do not need to babysit the plugins.**

## Architecture (lean core + lazy playbooks)

- `llm-swarm_v2.md` — the **always-loaded kernel**: the decision procedure plus the complete safety surface (Prime Directive, Gates A–E, Failure Algebra, Bundle Contract, Guardrail Register).
- `llm-swarm_v2-references/` — **lazy playbooks**, read only when the kernel routes into them:
  - `p5b-protocol.md` — the 6-round adversarial debate mechanics
  - `fanout-protocol.md` — parallel lens fan-out + synthesis
  - `bundle-tiers.md` — what context goes to a delegate at each tier
  - `lens-prompts.md` — the per-lens reviewer prompts
  - `command-appendix.md` — verified Codex/Gemini companion command contracts
  - `worked-examples.md` — end-to-end routing examples
  - `incident-provenance.md` — why each guardrail exists (not loaded at runtime; audit only)

The kernel never branches on playbook prose; safety/judgment never lives in a lazy playbook by design.

## Usage

Invoke the skill and point it at what you want audited. It handles both:

- **Defect/quality review:** "swarm this", "review this branch before merge", "anything I'm missing in this module?", post-phase / pre-merge audits.
- **Advisory / decision validation:** "Claude is recommending approach X — is that really the best way?", "should we use A or B here?".

The kernel's Routing Procedure decides solo-Claude vs. cross-model and whether to trigger the P5b debate. You don't have to pick a mode; you *can* steer it ("just a quick read", "full adversarial pass", "this is security-sensitive").

## Limitations

- Cross-model value requires both delegates working. If one or both are down the skill still runs (Claude reviewer-pole floor) and **tells you** it's degraded rather than pretending otherwise — but a single-model read is not the same as a cross-model debate.
- This repo ships the skill only. It does not bundle or manage the Codex/Gemini CLIs or plugins themselves; the install pages show you how to set those up.
- Token/runtime cost scales with stakes: a full P5b + fan-out pass on a large surface is deliberately expensive. The kernel has mandatory check-ins for large fan-outs, but you are responsible for your own model/plan rate limits.
- The `codex:codex-rescue` / `gemini:gemini-rescue` agents the swarm uses for substantial delegate work **ship with the Codex/Gemini plugins** — installing the plugins (steps 1–2) gives you these automatically; no extra step.
- *(Genuinely optional)* Some flows additionally reference **superpowers** skills (`requesting-code-review`, `receiving-code-review`, `verification-before-completion`, `systematic-debugging`, `test-driven-development`) by name. The swarm degrades gracefully without superpowers installed.

## License

See [`LICENSE`](./LICENSE). A license has **not** been chosen yet — the file is a placeholder. You must select a license and fill in the copyright holder before publishing or distributing.

## Acknowledgements

The guardrail design (G1–G46) is an incident-driven hardening record: every rule exists because an earlier version got something wrong in practice. See `llm-swarm_v2-references/incident-provenance.md`.
