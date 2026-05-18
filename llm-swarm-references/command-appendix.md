# Playbook — Command Appendix (verified contracts)

> Mechanics only; kernel Gates govern. Contracts below were verified against codex-companion v1.0.4 / gemini-companion v1.0.0 source. **These plugins evolve — before relying on a flag/alias/sub-command, verify it against the versions you actually have installed** (`node "<companion>" setup --json` reports the runtime; the sub-command surface is in the companion source under your plugin cache). Treat any mismatch as authoritative over this table and update accordingly. **Resolve script paths via `Glob **/codex-companion.mjs` / `**/gemini-companion.mjs` under `~/.claude/plugins/cache/` — never hardcode.** Inside a rescue subagent the path is `node "${CLAUDE_PLUGIN_ROOT}/scripts/<companion>.mjs"`.

## Codex companion — `node "<codex>" <sub> [args]`

| sub | line | returns |
|---|---|---|
| `setup` | `setup [--enable-review-gate\|--disable-review-gate] [--json] [--cwd <p>]` | `{ready,node,npm,codex,auth,sessionRuntime,reviewGateEnabled,…}`. No `--verify` (Gemini-only). |
| `review` | `review [--wait\|--background] [--base <ref>] [--scope auto\|working-tree\|branch] [--model <id>] [--json]` | Native reviewer. **Rejects focus text** (→ use adversarial-review). |
| `adversarial-review` | same as review **+ positional focus text** | Schema-structured `{result,rawOutput,parseError,reasoningSummary,context}`. |
| `task` | `task [--background] [--write] [--resume-last\|--resume\|--fresh] [--model <id\|spark>] [--effort none..xhigh] [--prompt-file <p>] [--json] [prompt]` | FG `{status,threadId,rawOutput,touchedFiles,reasoningSummary}`; BG `{jobId,status:"queued",logFile}`. Default sandbox **read-only** unless `--write`. |
| `status` | `status [job-id] [--all] [--json] [--wait] [--timeout-ms] [--poll-interval-ms]` | With id: `job` (incl. **`job.logFile` = absolute log path**, the canonical resolver). `--wait` **requires** a job-id. |
| `result` | `result [job-id] [--json] [--cwd <p>]` | `{job,storedJob}` final payload. |
| `cancel` | `cancel [job-id] [--json] [--cwd <p>]` | Interrupts the turn, kills process tree, marks `cancelled`. |

`--resume` matrix: **`task` only** (`--resume`/`--resume-last`). NOT `review`/`adversarial-review` → use `--base <pre-fix-ref>`. Only alias: `spark → gpt-5.3-codex-spark`; otherwise pass literal; omit `--model` for default. Job-id prefixes `task-*`/`review-*`.

## Gemini companion — same surface, deltas

- `setup` **has `--verify`** (live credential check; may set `ready:false`, adds `auth.verified`/`auth.verifyDetail`). Always call `setup --verify --json`.
- `review` accepts focus text (no native-reviewer split); both review/adversarial-review run the structured schema.
- `task` adds `--plan` (ACP `approval-mode:plan`, read-only). **Default is write-capable** (`--yolo --sandbox`) — pass `--plan` for read-only investigation passes.
- `--effort low|medium|high` only; **accepted but inert** (prints a stderr notice).
- `status --wait` works **without** a job-id (resolves a wait-target).
- FG errors wrapped as `{status:"error",message,jobId,stderr}`; empty marker `Gemini did not return a final message.`
- Aliases: `pro`/`pro-3`→`gemini-3.1-pro-preview` (default analysis), `flash`→`gemini-3-flash-preview`, `flash-lite`→`gemini-3.1-flash-lite-preview`, `2.5-pro`→`gemini-2.5-pro`, `auto`→`auto-gemini-3`. Pass literal otherwise.
- ACP methods `session/new|prompt|cancel|unstable_setSessionModel` (the `unstable_` prefix may change — patch the method table, never rewrite the broker).

## Agent rescue (Mechanism B)

`Agent({subagent_type:"codex:codex-rescue"|"gemini:gemini-rescue", description, prompt[, run_in_background]})`. Both: `model:sonnet`, `tools:Bash`, thin single-`task` forwarder, stateless, returns companion stdout verbatim with no commentary. Codex rescue **defaults `--write`**; Gemini rescue **defaults `--yolo --sandbox`** (write) — pass intent for `--plan` read-only. Strips Claude-side routing tokens (`--background`/`--wait`); `--resume` shorthand → `--resume-last`. Bundle text goes **inline in `prompt`** — never a temp-file path (kernel G24). On Bash failure / no invocation: returns nothing (treat as GATE-B no-anchor).

Use rescue (B) for substantial implementation/diagnosis/research; use Bash companion (A) for review/adversarial-review/status/result/cancel (rescue is forbidden from calling those).

## Claude reviewer pole (Mechanism C / Lens F)

`Task({subagent_type:"general-purpose", description:"Claude code review — <scope>", prompt:<inline bundle + diff + reviewer template>})`. Reviewer template ships with superpowers `requesting-code-review` (`Glob **/code-reviewer.md` under plugin cache). Output: Strengths / Issues(Critical|Important|Minor, file:line) / Recommendations / Assessment with Ready-to-merge verdict + `F-V<n>` IDs. **Fresh context — no orchestrator history/reasoning trace** (kernel G42). Stateless. Does **not** count against delegate caps.

## Async / observability

Log path: `status <job-id> --json` → `job.logFile` (absolute). State dir = `<slug>-<sha256(realpath(workspace))[:16]>` — **never slug-guess**. Log lines: Codex `[ts] Thread ready (id)` / `Assistant message captured` / `Running command` / `Command completed (exit N)`; Gemini `[ts] Session ready (id)` / `Reasoning:` / `Tool call:` / `Tool completed:`. Both emit `[ts] Queued for background execution.` early. Monitor tail: git-bash `tail -n +1 -f "<logFile>"`; PowerShell `Get-Content -LiteralPath "<logFile>" -Wait -Tail 0`. Terminal states: `completed/done`, `failed/failed`, `cancelled/cancelled`. Freeze thresholds: warn Codex 4m/Gemini 6m, cap Codex 8m/Gemini 12m, queued-silence > 90s = immediate (kernel §4, fanout Step 3).

## Relevant Superpowers skills

`requesting-code-review` (reviewer template / Mechanism-C prompt) · `receiving-code-review` (disciplined handling of returned findings → Round 3/5) · `writing-plans` (spec→plan; feeds bundle element 2) · `verification-before-completion` (gates closure + GATE-B) · `systematic-debugging` (rescue/debug methodology) · `test-driven-development` (rescue fix work, verified Round 6). Invoke by name via the Skill tool when the flow calls for them.
