# Step 2: Turn on the Gemini AI

This is the second of three setup steps. Here you install Google's Gemini tool, sign in, and connect it to Claude Code. When you're done, Gemini is the second of the two extra AIs the swarm can bring in for a review.

> Assumes Claude Code is installed and working, and that you've finished [Step 1 (Codex)](./install-codex.md).

---

## 1. Install the Gemini tool

Gemini is a small command-line program. The Claude Code side is just a connector — the actual work happens in the `gemini` program, so it has to be installed first.

Open a terminal and run:

```bash
npm install -g @google/gemini-cli
```

Check it worked:

```bash
gemini --version
```

You should see a version number (this guide was tested with `0.42.0` — anything newer is fine too).

> **If a later step says it can't find `gemini`:** the program installed fine, but your computer doesn't know where to look for it yet. Run `npm config get prefix` to see where npm puts global tools, add that location to your system's `PATH`, then restart your terminal **and** Claude Code. This is a normal one-time computer setup thing, not specific to this project.
>
> If anything here ever looks out of date, the official page is the source of truth: <https://www.npmjs.com/package/@google/gemini-cli> — it links to Google's full instructions.

## 2. Sign in

You sign in once, inside the Gemini tool itself. Claude Code reuses that sign-in automatically — there's no separate login anywhere else.

The Gemini tool opens an interactive screen by default. Start it once:

```bash
gemini
```

The first time, it walks you through signing in — choose **sign in with Google** (opens your browser) or paste a Gemini API key, whichever you have. Once you're signed in, close it (type `/quit`, or press Ctrl+C).

Quick check that sign-in worked — this asks Gemini a tiny question without opening the interactive screen:

```bash
gemini -p "reply with: ok"
```

If it replies normally (not a sign-in error), you're set.

> If the sign-in screen looks different from this description (it changes between versions), follow the sign-in section on the official page: <https://www.npmjs.com/package/@google/gemini-cli>. All the swarm needs is that the quick check above works.

## 3. Connect Gemini to Claude Code

Now hook Gemini into Claude Code. In a Claude Code session, type these two lines:

```
/plugin marketplace add Mvizer/gemini-plugin-cc
/plugin install gemini@gemini-plugin-cc
```

The first line tells Claude Code where to find the plugin; the second installs it. If it asks you to confirm, say yes.

> **Note — this points at a private, audited copy.** `Mvizer/gemini-plugin-cc` is a private copy of the open-source `m-ghalib/gemini-plugin-cc` (Apache-2.0), kept so the setup doesn't depend on a third-party repo that could change. Because it's **private**, only a GitHub account with access (currently just the owner) can install it with the line above. Setting this up on your own machine with that account: you're fine. If this guide was shared with you and that line fails with a "not found"/access error, the repo hasn't been made public yet — ask the owner to flip it public.

Then load it into the session you're in right now:

```
/reload-plugins
```

You only need that last line for your current session — any new Claude Code session already has it.

*Prefer setting this up by editing config instead of typing commands? Add this to your Claude Code `settings.json` and start a fresh session:*

```json
{
  "extraKnownMarketplaces": {
    "gemini-plugin-cc": { "source": { "source": "github", "repo": "Mvizer/gemini-plugin-cc" } }
  },
  "enabledPlugins": {
    "gemini@gemini-plugin-cc": true
  }
}
```

## 4. Check it's working

You don't need to test Gemini on its own. The llm-swarm skill checks it automatically every time it runs and will say so clearly if Gemini is unreachable or not signed in. The real end-to-end test is running the swarm itself (README, step 3).

If you'd like a quick sanity check anyway: `gemini -p "reply with: ok"` returns a normal reply, and `gemini --version` prints a version. If both do, Gemini is ready.

---

**Next:** the llm-swarm skill is already in this repository — head back to the [README](./README.md) for step 3 and your first review.
