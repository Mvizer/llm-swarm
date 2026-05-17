# Step 1: Turn on the Codex AI

This is the first of three setup steps. Here you install OpenAI's Codex tool, sign in, and connect it to Claude Code. When you're done, Codex is one of the two extra AIs the swarm can bring in for a review.

> Assumes Claude Code is already installed and working. That part isn't covered here.

---

## 1. Install the Codex tool

Codex is a small command-line program. The Claude Code side is just a connector — the actual work happens in the `codex` program, so it has to be installed first.

Open a terminal and run:

```bash
npm install -g @openai/codex
```

Check it worked:

```bash
codex --version
```

You should see a version number (this guide was tested with `0.130.0` — anything newer is fine too).

> **If a later step says it can't find `codex`:** the program installed fine, but your computer doesn't know where to look for it yet. Run `npm config get prefix` to see where npm puts global tools, add that location to your system's `PATH`, then restart your terminal **and** Claude Code. This is a normal one-time computer setup thing, not specific to this project.
>
> If anything here ever looks out of date, the official page is the source of truth: <https://www.npmjs.com/package/@openai/codex> — it links to OpenAI's full instructions.

## 2. Sign in

You sign in once, inside the Codex tool itself. Claude Code reuses that sign-in automatically — there's no separate login anywhere else.

```bash
codex login
```

This opens your browser to sign in with your OpenAI / ChatGPT account. When you're back, confirm it stuck:

```bash
codex login status
```

It should say you're logged in.

*Using an API key instead of an account? Run `printenv OPENAI_API_KEY | codex login --with-api-key`. Either way works — the swarm only cares that `codex login status` is happy.*

## 3. Connect Codex to Claude Code

Now hook Codex into Claude Code. In a Claude Code session, type these two lines:

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
```

The first line tells Claude Code where to find the plugin; the second installs it. If it asks you to confirm, say yes.

Then load it into the session you're in right now:

```
/reload-plugins
```

You only need that last line for your current session — any new Claude Code session already has it.

*Prefer setting this up by editing config instead of typing commands? Add this to your Claude Code `settings.json` and start a fresh session:*

```json
{
  "extraKnownMarketplaces": {
    "openai-codex": { "source": { "source": "github", "repo": "openai/codex-plugin-cc" } }
  },
  "enabledPlugins": {
    "codex@openai-codex": true
  }
}
```

## 4. Check it's working

You don't need to test Codex on its own. The llm-swarm skill checks it automatically every time it runs and will say so clearly if Codex is unreachable or not signed in. The real end-to-end test is running the swarm itself (README, step 3).

If you'd like a quick sanity check anyway: `codex login status` says you're logged in, and `codex --version` prints a version. If both do, Codex is ready.

---

**Next: [Step 2 — turn on the Gemini AI](./install-gemini.md).**
