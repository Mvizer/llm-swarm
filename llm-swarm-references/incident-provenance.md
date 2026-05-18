# Provenance — why each guardrail exists (G1–G46)

> **Not required for operation.** The kernel never branches on this file. It is the rationale record so a future maintainer changing a Gn knows what it cost to learn. Authoritative behavior lives in the kernel (`llm-swarm.md` §1–§5) and playbooks; this is the "why" only. Read it when auditing a guardrail or deciding whether a rule is safe to change.

Every entry below corresponds to a real mistake, hallucination, misalignment, or drift class observed while operating this orchestrator in practice. The skill reproduces the *behavior*; this records the *lesson*.

| G | Defends against | Incident / lesson |
|---|---|---|
| G1 | Confabulated excuse to skip a delegate, silently dropping cross-model review | Subagents fabricated a "the other delegate's broker is mismatched / broker-busy race" excuse — the two plugins share no broker; Codex defaults to direct mode (no broker at all). |
| G2 | A round accepted that never ran (novel phrasings the watchlist misses) | Structural backstop added after watchlist-only proved leaky. |
| G3 | Mis-classifying a per-workspace failure by probing the wrong cwd | Orchestrator probed its own healthy cwd, not the broken target cwd; the state dir is keyed on `sha256(realpath)`, so the probe must run in the target cwd. |
| G4 | Watchlist over-firing → dismissing a real failure as fabrication | A correct "delegate not logged in, pipe X" narration from another session was wrongly called a fabrication; a long-stale broker file pinned a long-dead PID. |
| G5 | Treating all pipe-name / ENOENT signals identically | A plugin-data env leak (one delegate reads the other's pipe) and a stale-broker-pin to a dead PID are distinct real bugs with distinct fixes. |
| G6 | Silent degradation when a delegate's auth lapsed | Auth-failure protocol added: 3-class classifier + one retry + mandatory user pause with 3 options. |
| G7 | Fabricated auth narrations (invented pipe addresses, "won't retry a 3rd time", split "infra healthy but auth failed") | Orchestrator invented a "named pipe not bound" excuse while the endpoint field was `null`, contradicting it. |
| G8 | "Is the agent running?" being guesswork (both never-ran-fabrication and silent-hang) | Stale-broker and queued-hang incidents had no observability; queued-then-silent past the cap is the signature. |
| G9 | One stuck call dominating wall time | A ~40-min delegate stall that was 100% I/O wait; "a stuck call has already failed." |
| G10 | Improvising by analogy when a hang hits a round other than Round 2 | The escalation ladder was Round-2-scoped; agents improvised for the Round 4 judge / Round 6. |
| G11 | Throwing away cross-model review when a working delegate exists | Old "do the work yourself" text silently dropped the whole cross-model layer. |
| G12 | Collapsing to "solo Claude, no review" when delegates down | Reviewer pole (Lens F) added as the degraded-mode floor; skipping it re-introduces the failure it prevents. |
| G13 | Nondeterministic 0-vs-1 delegate calls on trivial input | Two readers diverged on identical "thoughts?"-on-a-README. |
| G14 | Stripping judge context on a security review because findings looked localized | Two orchestrators diverged; one starved the judge on an auth review. |
| G15 | Wasting tokens (full bundle to a bounded-dispute judge) AND starving the judge | A dependency-closure slice reached the identical verdict as the full judge. |
| G16 | A resumed Round 6 fabricating a re-review on an unverified session | Resume path landed only behind 3 hard-guards + mandatory full-rebundle fallback. |
| G17 | A security finding dropped because its lens returned prose, not F-IDs | Recovery was undefined; one-retry-then-self-parse added. |
| G18 | Editing rules that aren't broken; skipping mandated steps under pressure | Multiple suspected gaps were tested and *deliberately left unchanged*; Round 3 has no shortcut. |
| G19 | Burning quota on trivial delegation or 3-model votes | "Delegation theater adds cost for no gain"; voting is the single most expensive pattern. |
| G20 | Running one review and relaying it as the answer (P5b→P5 collapse) | Stop-and-check failsafe added; "stopping at Round 2 is a failure of the skill." |
| G21 | Degradation paths quietly eliminating the cross-model layer | A judge ruling on a synthetic round "is worse than no debate — it lends false legitimacy." |
| G22 | Looping the debate when models won't converge | "Models that can't agree in two rounds won't in three; the escalation IS the deliverable." |
| G23 | Orchestrator breaking a tie it has no authority to decide | Worked examples: the user is always the final tiebreaker. |
| G24 | A delegate burning minutes of pure I/O reading a referenced file | The ~40-min stall was 100% I/O wait; inline-content is the biggest single wall-time win. |
| G25 | Skill machine-bound to one user's paths | Portability pass: dynamic discovery, no absolute paths, conditional memory. |
| G26 | Routing rubric silently rotting as new models ship | Staleness self-check makes rot visible instead of silent. |
| G27 | Routing on conversation drift, not project state | CLAUDE.md + git log non-skippable; reduced-context announced. |
| G28 | Draining the user's rate limits in one session | Tier-aware caps + check-in; Lens F is free (local tokens). |
| G29 | Auto-launching a 12+ agent / >30-min fan-out without consent | 200–400k mandatory check-in; >400k refuse. |
| G30 | Losing precision/severity by paraphrasing constraints/findings | Removed unproducible turn-number attribution; verbatim handoffs mandated. |
| G31 | Acting on a delegate's hallucinated claim (worst: about user intent) | "Only the user can tell you what the user wants." |
| G32 | Accepting a justified-but-misattributed fallback | Mismatched reason is still a partial fabrication. |
| G33 | Integrating a stale background result that no longer fits | Context-reload before consume prevents this. |
| G34 | Looping retries, draining quota/wall-time | One retry per failure mode, then escalate. |
| G35 | Stale jobs burning quota after the user changed the question | Cancel-on-interrupt is automatic and pre-emptive. |
| G36 | Under-bundled reviews → low-quality delegate output | Under-bundled Tier 4 reviews were an observed failure mode in practice. |
| G37 | A systemic class uncaught because single-agent review can't see it across shards | Only fan-out + synthesis can elevate 3+ independent hits. |
| G38 | One hung sub-agent blocking all synthesis / silent incompleteness | N-1 harvest with the gap explicitly surfaced. |
| G39 | Giving the judge a fresh-review prompt → P5b→P5 collapse | "If you give the judge the same prompt the first reviewer got, you've collapsed P5b into P5." |
| G40 | Asking open-ended "what should I look at?" instead of resolving it | Referent-resolution order defined. |
| G41 | Assuming Claude can autonomously invoke the delegate slash-commands (disable-model-invocation:true) | An earlier correction fixed the invocation path, a false resume claim, and phantom aliases. |
| G42 | Claude reviewer re-creating the orchestrator's blind spots | Passing the reasoning trace re-creates the echo chamber the design breaks. |
| G43 | Gemini writing files on a first investigation pass; needless handoffs | Gemini `task` defaults write-capable; `--plan` first; default-to-self in specialized domains (K5). |
| G44 | A connector/plugin-layer fault mis-handled as a model failure, fabrication, or endless delegate-switching | A delegate `--background` queued-hang: a plugin job-machinery bug (spawn-before-persist race) beneath the skill, version-independent; the switch-delegate ladder just relocates it. Diagnosis took 3 passes because `stdio:"ignore"` hid the real error. |
| G45 | Theorizing past the evidence; false "decisive" tests; mis-attributing layers; verifying on a proxy not the real path | Same incident: env/state-root/stdio hypotheses were all wrong turns disproven by evidence; the probe "worked" only because it sidestepped the real path. Codified so the next session does not re-chase them. |
| G46 | A consultation / decision-validation request mis-handled as a defect hunt — bug-list output, under-contexted delegates, wrong question answered | A design audit found triggers + user-escalation were covered but the *middle* was not — bundles were code-centric, all delegate/lens prompts F-ID-shaped, round structure assumed an artifact. Added a first-class Advisory mode (decision-payload bundle + reasoned-recommendation output + proposal-as-R1 flow). |

**Drop-risk order (hardest/most-likely-repeated first):** G1, G3, G7, G4, G2, G20, G24, G10/G11, G5, G14. Any implementation that fails any of these silently has repeated a known production incident.
