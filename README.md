# llm-swarm

**Get a real second and third opinion from AI — automatically.**

When you ask an AI to review your code, sanity-check a plan, or tell you the best way to do something, you get *one* model's answer. That one model can be confidently wrong. It has blind spots. It might miss a security bug, or hand you its first idea as if it were the best idea — and you'd have no easy way to know any of that happened.

llm-swarm fixes that. It's an add-on for Claude Code that, instead of trusting a single AI, brings in **three** — Claude, OpenAI's Codex, and Google's Gemini. They each review the work independently, argue out the places they disagree, and you get back one reconciled answer. You catch the problems a single model would have sailed right past — and when the models genuinely disagree, it tells you honestly instead of giving you false confidence.

Think of it as turning "ask one assistant" into "convene a small panel of experts that actually challenge each other."

## When you'd reach for it

- **"Claude says this code is fine — is it *really*?"** Have Codex and Gemini check it too, and pressure-test the answer.
- **"Is this the best approach, or just the first one the AI thought of?"** Get real alternatives with the trade-offs laid out.
- **Right before you merge or ship something that matters.** A three-model review is much harder to fool than a single pass.
- **"Am I missing anything here?"** Three models looking from different angles catch more than one looking alone.

## What you need before you start

This guide assumes:

- **Claude Code is already installed and working.** Setting that up is not covered here.
- **An active account/subscription for each AI you want to use.** These tools don't run for free — each AI bills to its own account:
  - **Claude** — a paid Claude plan (or Anthropic API access). Required; this is the AI that runs the show.
  - **Codex** — an OpenAI/ChatGPT plan that includes Codex, or an OpenAI API key.
  - **Gemini** — a Google account for the Gemini CLI, or a Gemini API key.

  You can run with just Claude, but the real value — the cross-checking and debate — only happens when Codex and/or Gemini are signed in too. The sign-in steps in the guides won't succeed without an active account behind them.
- **Node.js is installed** (a recent LTS version). The two extra AIs are small command-line tools you install with `npm`. If an install command later complains about your Node version, updating Node is the first thing to try.

## Setup — three steps

You'll switch on the two extra AIs, then drop in the skill itself. Do them in order:

| Step | What you do | Guide |
|------|-------------|-------|
| 1 | Turn on the **Codex** AI — install it, sign in, connect it to Claude Code | [install-codex.md](./install-codex.md) |
| 2 | Turn on the **Gemini** AI — install it, sign in, connect it to Claude Code | [install-gemini.md](./install-gemini.md) |
| 3 | Add the llm-swarm skill itself | this page, just below |

Each guide is short and self-contained. Finish step 1, then step 2, then come back here for step 3.

### Step 3 — add the skill

Copy the skill into Claude Code's commands folder:

- **macOS / Linux:** `~/.claude/commands/`
- **Windows:** `%USERPROFILE%\.claude\commands\`

When you're done it should look like this:

```
.claude/commands/
  llm-swarm.md
  llm-swarm-references/      <- folder of supporting files
```

Keep the `llm-swarm.md` file and the `llm-swarm-references` folder **together, with those exact names**. The skill looks for that folder by name, sitting right next to itself — rename either one and it won't find its own instructions.

Then **start a fresh Claude Code session** so it notices the new command. (A new session is what loads it; there's no separate reload step for a hand-added command like this.)

That's the whole setup. Your very first run also doubles as the test: the skill automatically checks that Codex and Gemini are reachable and signed in, and says so plainly if either isn't — so if a real review runs cleanly, you know steps 1 and 2 worked too.

## How to actually use it

You don't have to learn any commands or modes. Just say what you want in plain words, for example:

- *"Use llm-swarm to review this branch before I merge."*
- *"Get a second opinion on this file — anything I'm missing?"*
- *"Claude recommended approach A. Swarm it and tell me if that's really the best call."*

It decides on its own whether your question needs the full multi-model debate or a lighter check. You can always steer it — *"just a quick look"* or *"go deep, this is security-critical."*

## Good to know

- The real value comes from all three AIs being available. If Codex or Gemini is down it still works using Claude alone — but it will **tell you** it's running in that reduced mode rather than pretend the cross-check happened.
- A deep three-model review uses noticeably more of your AI usage than a single quick question. The skill checks in with you before the largest jobs, but keep an eye on your own plan limits.
- It looks after itself. If a connection drops or a background job stalls, it retries, falls back, or switches AIs — and tells you what happened. You don't need to babysit it.
- This repository is the skill only. It doesn't install or manage the Codex/Gemini tools for you; the two setup guides walk you through that.

## License

See [LICENSE](./LICENSE). A license has **not** been chosen yet — the file is a placeholder. Pick one and add your name before sharing this with anyone else.

## Under the hood (optional — you don't need this to use it)

If you're curious: the skill is built as a small always-on core (the decision-making plus every safety rule) and a set of detailed playbooks it only opens when a particular review path actually needs them. That keeps it fast without cutting any corners on safety. Every safety rule in it exists because an earlier version got something wrong in real use — `llm-swarm-references/incident-provenance.md` tells the story behind each one.
