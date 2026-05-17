# Install the Codex delegate

This is **step 1 of 3**. It puts the OpenAI Codex CLI on your machine, authenticates it, and installs the Claude Code plugin that the swarm drives. When you finish, Codex is one of the two cross-model reviewers the swarm can route work to.

> **Baseline:** Claude Code is already installed and working. This page does *not* cover installing Claude Code.

---

## A. Install the Codex CLI

The Claude Code Codex plugin is a thin driver — it shells out to the real **`codex`** CLI, which must be installed and on your `PATH`. You need a current LTS **Node.js**; if the install below errors with an unsupported-engine / Node-version message, upgrade Node first.

```bash
npm install -g @openai/codex
```

> Canonical source of truth for install + authentication, if anything here is out of date or your environment differs: the official package page at <https://www.npmjs.com/package/@openai/codex> (it links to the upstream repo and docs). This guide was verified against CLI `0.130.0`.

Verify:

```bash
codex --version
```

You should see a `codex-cli` version line (this guide was written against `0.130.0`; newer is fine).

> **PATH note.** The `codex` command must be resolvable from a normal shell — Claude Code launches the plugin as a child process and needs to find it. If `codex --version` works in your terminal but the plugin later can't find it, your npm global bin directory isn't on the `PATH` Claude Code inherits. Find it with `npm config get prefix` (the binary lives in `<prefix>/bin` on macOS/Linux, `<prefix>` on Windows) and add that directory to your `PATH`, then restart your terminal and Claude Code.

## B. Authenticate the CLI

Authentication is done **in the Codex CLI itself**. The Claude Code plugin reuses these stored credentials — you do not authenticate separately inside Claude Code.

Sign in with your OpenAI / ChatGPT account (opens a browser):

```bash
codex login
```

Confirm it worked:

```bash
codex login status
```

This should report that you are logged in.

*API-key alternative:* if you use an API key instead of an account, Codex reads it from stdin:

```bash
printenv OPENAI_API_KEY | codex login --with-api-key
```

Either path is fine — the swarm only cares that `codex login status` is healthy.

## C. Install the Claude Code plugin

Inside Claude Code, add the plugin's marketplace and install it. Run these as slash commands in a Claude Code session:

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
```

Then make it active in the running session:

```
/reload-plugins
```

New Claude Code sessions started after this will have the plugin automatically — the `/reload-plugins` step is only needed to pick it up in the *current* session.

*Scripted / non-interactive alternative:* instead of the slash commands you can add this to your Claude Code `settings.json` and start a fresh session:

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

## D. Verify

You don't need to test Codex in isolation — the llm-swarm skill runs its own Codex health probe (`setup`) automatically every time it activates, and it will tell you clearly if Codex is unreachable or unauthenticated. The definitive end-to-end check is running the swarm itself (see the README, step 3).

A quick standalone sanity check, if you want one:

```bash
codex login status
```

If that is healthy and `codex --version` works from your shell, Codex is ready.

---

**Next:** [`install-gemini.md`](./install-gemini.md) — step 2 of 3.
