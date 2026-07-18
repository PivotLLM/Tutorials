# ClawEh Quickstart

Get a personal AI assistant running in minutes — no compiler, no manual config
files, and (optionally) a Telegram bot you can message from anywhere.

This guide is for people who are new to ClawEh. It walks you from zero to a
working assistant using a downloaded binary and the built-in setup wizard. There
is a second half for tuning and for more advanced setups.

ClawEh is a lightweight personal AI assistant that connects
messaging channels (like Telegram) to an AI model of your choice — either a
hosted API such as OpenRouter, or a command-line agent you already have
installed. The cores of our **Fusion** (connectivity/tooling) and **Maestro**
(orchestration) projects are now built directly into ClawEh, so a single binary
gives you the assistant plus its tooling and memory. Fusion and Maestro also
continue to exist as standalone software.

---

## Part 1 — Choosing an LLM (start here)

> **In a hurry?** If you already have API keys and know which model(s) you want to
> use, skip to [Part 3 — Getting Started](#part-3--getting-started). If you plan to
> use Claude Code, Codex, or Gemini CLI instead of an API, skip there too — ClawEh
> detects those automatically.

Your assistant is only as good (and only as expensive) as the model behind it.
Models vary enormously in both **capability** and **cost** — from small, fast, and
nearly free, up to large frontier models that cost real money per message. For a
first assistant you do **not** need a frontier model. A capable, inexpensive model
is the right starting point; you can always change it later in one click.

**We recommend starting with DeepSeek V4 Flash, served through OpenRouter.** It is
fast, inexpensive, and more than good enough for everyday assistant tasks. In the
setup wizard it is tagged **Recommended**.

| | Model | OpenRouter slug |
|---|---|---|
| **Recommended** | DeepSeek V4 Flash | `deepseek/deepseek-v4-flash` |
| **Alternative** | Gemini 3.1 Flash Lite | `google/gemini-3.1-flash-lite-preview` |

Both sit at a similar low price point and are available through a single
OpenRouter account, so you can switch between them without signing up for anything
new. If you outgrow them, OpenRouter also offers larger models (for example
DeepSeek V4 Pro) that you can select later.

> **Why OpenRouter?** One account and one API key give you access to models from
> many providers, with a single balance and spending limit you control. That makes
> it ideal for a first assistant.

---

## Part 2 — Create a Telegram bot (optional)

The nicest way to use ClawEh is to message it on Telegram from your phone or
desktop. This step is optional — you can also chat with your assistant in the
built-in web UI — but if you want Telegram, set up the bot **now** so you have the
token ready when you reach the wizard.

1. Open Telegram and search for **@BotFather**.
2. Send `/newbot` and follow the prompts (choose a name and a username).
3. Copy the **bot token** BotFather gives you. It looks like
   `1234567890:ABCDEF...`. Keep it handy.
4. Find **your** numeric Telegram user ID by messaging **@userinfobot**. You'll
   use this to restrict the bot so only you can talk to it.

That's it — hold on to the token and your user ID for Part 3.

---

## Part 3 — Getting Started

### 3.1 Create an OpenRouter account (with a budget)

If you're using a hosted model (recommended for beginners):

1. Go to **https://openrouter.ai** and create an account.
2. Add a **small amount** of credit to start — around **$5–$10** is plenty to try
   things out.
3. **Set a spending limit.** In your OpenRouter account settings you can cap how
   much can be spent, and you can attach a limit to the API key you create. Setting
   a small **daily budget** protects you from surprises while you learn.
4. Create an **API key** and copy it. You'll paste it into the wizard.

> You can skip this section entirely if you'd rather use a CLI agent — see the
> note in 3.3.

### 3.2 Download the binary

We publish ready-to-run **Linux and macOS** binaries on GitHub. There is nothing to
compile.

Go to the **Releases** page:

**https://github.com/PivotLLM/ClawEh/releases**

Download the file that matches your system:

| System | Asset |
|---|---|
| Linux (most PCs/servers) | `claw-linux-amd64` |
| Linux (older 32-bit x86) | `claw-linux-386` |
| Linux (ARM, e.g. Raspberry Pi) | `claw-linux-arm64` |
| macOS (Apple Silicon) | `claw-darwin-arm64` |
| macOS (Intel) | `claw-darwin-amd64` |

Then make it executable and give it a simple name. On Linux:

```bash
chmod +x claw-linux-amd64
mv claw-linux-amd64 claw
```

On macOS, substitute the matching asset (`claw-darwin-arm64` for Apple Silicon,
`claw-darwin-amd64` for Intel). If macOS blocks the first launch, allow it under
**System Settings → Privacy & Security**.

### 3.3 Start ClawEh and open the setup wizard

You can run ClawEh directly to try it out:

```bash
./claw
```

This starts the assistant along with its **web UI** on port **18790**. Open a
browser to:

**http://localhost:18790**

On a fresh install, ClawEh **automatically launches the setup wizard**. It walks
you through a handful of steps:

1. **Welcome** — a short introduction.
2. **Network** — keep network access **off** (localhost only) unless you know you
   need remote access. This is the safe default.
3. **Provider** — choose your AI provider. Pick **OpenRouter**, paste the API key
   from step 3.1, and click **Test** to confirm it works.
   > **Already have a CLI?** If you have **Claude Code**, **Codex**, or **Gemini
   > CLI** installed and signed in, ClawEh detects them automatically and lists
   > them here as ready-to-use agents — **no API key required**. Pick one of these
   > instead of OpenRouter if you'd prefer to use your existing subscription.
4. **Model** — choose your default model. Select **DeepSeek V4 Flash** (tagged
   *Recommended*), or the alternative from Part 1.
5. **Agent** — give your assistant a name (for example, `Assistant` or `Alice`).
6. **Review** — confirm, and the wizard writes your configuration.

Your assistant is now live. You can chat with it right there in the web UI.

### 3.4 Connect Telegram

If you created a bot in Part 2, connect it through the web UI:

1. In the ClawEh web UI, go to the **Channels** section and add a **Telegram** bot.
2. Paste the **bot token** from BotFather.
3. In **Allow From**, enter **your** numeric Telegram user ID (from @userinfobot).
   This restricts the bot so only you can use it. Leaving this empty means **nobody**
   can connect; `*` would allow **anyone** — don't use `*` unless you mean it.
4. Enable the channel and save.

Now open Telegram, find your bot, and send it a message. It should reply using the
model you chose.

### 3.5 Install as a background service (Linux)

Once you're happy with it, install ClawEh so it starts automatically at boot. On
**Linux (systemd)**:

```bash
./claw install
```

This copies the binary to `~/bin` (or `~/.local/bin`), adds that directory to your
`PATH`, and registers a systemd service that starts ClawEh at boot. By default the
web UI is reachable only from the local machine and your private network (loopback
plus RFC1918 ranges).

To remove the service later:

```bash
claw uninstall
```

> **macOS:** `claw install` is Linux-only (it uses systemd). On macOS, just run
> `./claw` when you want the assistant, or set up your own launch agent.

You're done. The rest of this guide is optional.

---

## Part 4 — Tweaking, Tuning, and Advanced Use

Everything above gets a beginner running with one assistant. This section covers
common adjustments and points advanced users toward deeper capabilities.

### Where your settings live

ClawEh stores its configuration at **`~/.claw/config.json`**. The setup wizard and
the web UI write to this file for you, so you rarely need to edit it by hand. If
you relocate it, set the `CLAW_HOME` environment variable to the new directory.

Most things you'll want to change — providers, models, agents, and channels — are
editable directly in the web UI, which is the recommended way to make changes.

### Changing or adding models

You can add more models and switch the default at any time from the **Models** and
**Providers** pages in the web UI. From the command line:

```bash
# Show the current default model and list usable models
claw model

# Set a different default by its model name
claw model "OpenRouter DeepSeek V4 Flash"
```

Start cheap, and only move up to a larger model if a task needs it. Mixing is fine:
you might run a small model day-to-day and keep a stronger one available.

### Exposing the web UI beyond localhost

If you want to reach the web UI from another machine on your network, install with
an explicit bind address and, if needed, an allowlist:

```bash
./claw install --host 0.0.0.0 --allowed-cidrs 192.168.1.0/24
```

Loopback is always allowed. Be deliberate about who can reach the UI — anyone who
can, can talk to your assistant.

### Using a CLI agent instead of an API

If you selected Claude Code, Codex, or Gemini CLI in the wizard, ClawEh runs that
CLI as the model backend and uses whatever subscription the CLI is signed in to.
This can be more cost-effective than pay-as-you-go APIs for heavier use. The CLI
must be installed and authenticated **as the same user** that runs ClawEh.

### Useful commands

```bash
claw status      # Show ClawEh status
claw model       # Show or change the default model
claw test        # Test connectivity to configured models
claw version     # Show version information
```

### Built-in tooling and orchestration

Because the Fusion and Maestro cores are built into ClawEh, your assistant can be
extended with external tools/APIs and can orchestrate multi-step work without
installing anything else. These capabilities go well beyond this quickstart — see
the ClawEh repository documentation to go further:

**https://github.com/PivotLLM/ClawEh**

### Multiple users and memory scope

By default a single shared assistant serves you across channels. ClawEh also
supports per-user and per-platform memory scopes for multi-user setups. If you plan
to give several people access, review the session/scope and `allow_from` guidance
in the ClawEh README before opening it up — in a shared setup, everyone allowed in
can share the same context.

---

## Troubleshooting

- **Wizard doesn't appear:** Make sure `./claw` is running and browse to
  `http://localhost:18790`. The wizard only auto-launches until at least one model
  with working credentials exists.
- **Bot doesn't respond on Telegram:** Confirm the bot token is correct and that
  your numeric user ID is listed in **Allow From**. An empty allow list blocks
  everyone.
- **API key rejected:** Re-check the key in the **Providers** page and use the
  **Test** button. For OpenRouter, confirm your account has credit and that any
  spending limit hasn't been hit.
- **macOS won't run the binary:** Approve it under **System Settings → Privacy &
  Security** after the first blocked launch.

---

Copyright (c) 2026 Tenebris Technologies Inc. Provided for general educational
purposes under the terms in the repository README. It is up to you to ensure your
use of the software is consistent with your security requirements and risk
tolerance.
