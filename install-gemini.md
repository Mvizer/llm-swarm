# Install the Gemini delegate

This is **step 2 of 3**. It puts the Google Gemini CLI on your machine, authenticates it, and installs the Claude Code plugin that the swarm drives. When you finish, Gemini is the second of the two cross-model reviewers the swarm can route work to.

> **Baseline:** Claude Code is already installed and working, and you have completed [`install-codex.md`](./install-codex.md) (step 1).

---

## A. Install the Gemini CLI

The Claude Code Gemini plugin is a thin driver — it shells out to the real **`gemini`** CLI, which must be installed and on your `PATH`. You need a current LTS **Node.js**; if the install below errors with an unsupported-engine / Node-version message, upgrade Node first.

```bash
npm install -g @google/gemini-cli
```

> Canonical source of truth for install + authentication, if anything here is out of date or your sign-in flow differs: the official package page at <https://www.npmjs.com/package/@google/gemini-cli> (it links to the upstream repo and auth docs). This guide was verified against CLI `0.42.0`.

Verify:

```bash
gemini --version
```

You should see a version number (this guide was written against `0.42.0`; newer is fine).

> **PATH note.** The `gemini` command must be resolvable from a normal shell, because Claude Code launches the plugin as a child process and needs to find it. If `gemini --version` works in your terminal but the plugin later can't find it, your npm global bin directory isn't on the `PATH` Claude Code inherits. Find it with `npm config get prefix` and add the bin directory to your `PATH`, then restart your terminal and Claude Code.

## B. Authenticate the CLI

Authentication is done **in the Gemini CLI itself**. The Claude Code plugin reuses these stored credentials — you do not authenticate separately inside Claude Code.

The Gemini CLI defaults to interactive mode. Launch it once and complete the sign-in prompt:

```bash
gemini
```

On first run it walks you through authentication — pick a sign-in method when prompted: a Google account (opens a browser; the common path) or a Gemini API key. Once you've signed in, the credentials are stored and reused; exit the interactive session. If the prompts don't match this description (the CLI's first-run flow changes between versions), follow the auth section on the canonical package page linked above — the only thing the swarm requires is that a headless call (below) succeeds.

*API-key alternative:* set `GEMINI_API_KEY` in your environment before launching, and the CLI uses it instead of the browser sign-in.

Confirm it worked with a non-interactive (headless) call — this both checks auth and warms the path the swarm uses:

```bash
gemini -p "reply with: ok"
```

If it returns a normal model reply (not an auth error), the CLI is authenticated.

## C. Install the Claude Code plugin

Inside Claude Code, add the plugin's marketplace and install it. Run these as slash commands in a Claude Code session:

```
/plugin marketplace add m-ghalib/gemini-plugin-cc
/plugin install gemini@gemini-plugin-cc
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
    "gemini-plugin-cc": { "source": { "source": "github", "repo": "m-ghalib/gemini-plugin-cc" } }
  },
  "enabledPlugins": {
    "gemini@gemini-plugin-cc": true
  }
}
```

## D. Verify

You don't need to test Gemini in isolation — the llm-swarm skill runs its own Gemini health probe (`setup --verify`) automatically every time it activates, and it will tell you clearly if Gemini is unreachable or unauthenticated. The definitive end-to-end check is running the swarm itself (see the README, step 3).

A quick standalone sanity check, if you want one:

```bash
gemini -p "reply with: ok"
```

If that returns a normal reply and `gemini --version` works from your shell, Gemini is ready.

---

**Next:** the llm-swarm v2 skill is already in this repository (step 3). See the [README](./README.md) to drop it in and run your first swarm.
