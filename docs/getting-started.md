# Getting Started with MCPFusion, Maestro, and ClawEh

This is the first draft of this tutorial. Please do not hesitate to open an issue or create a PR.

Use of this tutorial is permitted only if you accept the accompanying License, disclaimer, and legal notices; otherwise, you must not use, copy, or rely on it.

## Important Notes

- Please review the copyright, disclaimer, and other legal information in README.md and that accompanies all software referenced in it. This tutorial is provided for geneneral educatonal purposes. It is up to you to ensure that your use of the software is consistent with your security requirements and risk tolerance.

- The software in this tutorial can be used by virtually any AI client that supports the MCP standard. However, the author has found it far more efficient and cost effective to use command-line agents (Claude Code, Codex, and Gemini CLI) connected to a subscription instead of a pay-as-you-go API. This tutorial assumes that you have at least one of the above CLIs installed, configured, and connected to a subscription that is appropriate for your tasks.

- The software is written in Go and has been tested on Linux and macOS. While this tutorial assumes Linux, it will work on macOS with the exception of the .service files intended to automatically start the componets at boot.

- While it is possible to constrain the agents to specific directories, and place other security controls in place, we suggest deploying this combination of software on a Linux system that does not contain other sensitive data. Assuming you wish your AI agents to continue working for you in the background, it doesn't make sense to install them on your laptop. A small VM, mini-PC, Raspberry Pi, or an old computer you hae laying around should all work fine.

- This tutorial assumes that your prefered command-line text editor is vi. Feel free to substitute your editor of choice.

## Introduction and Background

This tutorial walks you through setting up MCPFusion and Maestro, a powerful combination that gives your AI assistant persistent knowledge management, project orchestration, and access to external APIs. The author uses MCPFuion with a variety of desktop and CLI AI clients, including Claude Code, Claude Desktop, Codex, Gemini CLI, and ClawEh.

**MCPFusion** will become your connectivity hub. Your AI client(s) only need to connect to MCPFusion. It, in turn, can provide access to a multitude of APIs and consolidate access to other MCP searvers. This is particularly useful if you use more than one client. This tutorial assumes a single user environment. However, MCPFusion is designed for multiple users, and if desirable, this capability could be used to provide different access to different AI clients.

**Maestro** is a sophisticated orchestration tool that enables AI agents to perform complex tasks with an emphasis on delegation, reliable completion, repeatable proccesses, continuious improement, and quality assurance. When instructed to delegate work to a worker (sub-agent), Maestro uses command-line agents such as Claude Code, Codex, and Gemini CLI to execute non-interactive tasks. 

**ClawEh** is a rapidly evolving lightweight AI client with many features. This tutorial only scratches the surface of its capabilities by using it to connect a Telegram bot to a Claude-code backed agent. In this configuration it is more efficient to connect Claude Code to MCPFusion. However, ClawEh is capable of connecting directly to MCPFusion.

Assuming you install and configure all three components, and assuming you use Claude Code, the system will look like this:

[Telegram Bot] <-> [ClawEh] <-> [Claude Code] <-> [MCPFusion] <-> [Maestro]

Please note that MCPFusion is configuration driven and includes configuration files to faciliate connectivity to Google Search, Google Workspace, Micorosft 365, Trello, and others. Should you wish to connect to other APIs, detailed AI-friendly documentation is included. Most of the supplied JSON configuration files were written primarily by Claude Code.

---

## Choosing a User Account

In this tutorial, three components — MCPFusion, Maestro, and ClawEh — run under the same user account. ClawEh and Maestro execute the command-line agents (Claude Code, Codex, Gemini CLI), and MCPFusion executes Maestro (for stdio MCP), all in the user context. This allows the respective components to find their configurations, data, and in the case of the CLIs, subscription and authentication information.

Throughout this tutorial, `<USER>` represents the Linux username you choose. Wherever you see `<USER>`, substitute your actual username. Do *not* include the angle brackets.

### Recommendations

**Dedicated machine (recommended):** The simplest and safest approach is to deploy on a machine dedicated to running these services — a small VM, mini-PC, Raspberry Pi, or a spare computer. Resource requirements are modest. On a dedicated machine, you can simply use your own account.

**Shared machine:** If you install on a machine used for other purposes, we recommend creating a separate, limited account.

> **Security warning:** AI agents can read and act on the credentials and configuration of the account they run under. This includes environment variables, SSH keys, API tokens, and other files stored in the home directory. An agent that encounters a problem involving a remote system may autonomously attempt to resolve it using any credentials it can find (for example ssh keys). We strongly recommend a dedicated machine. If you must use a shared machine, create a separate account and ensure it has access only to the information you are comfortable exposing to an agent.

### Creating a New Account (if needed)

If you are on a shared machine and want a dedicated account, run the following as an existing user with sudo access:

```bash
sudo useradd -m -s /bin/bash <USER>
sudo passwd <USER>
```

Then log in as `<USER>` (or `su - <USER>`) before proceeding. All subsequent steps must be performed as `<USER>`.

To confirm your current username at any time:

```bash
whoami
```

---

## Prerequisites

Before you begin, ensure the following:

- **Operating system**: Linux with `sudo` access
- **Source directory**: These instructions assume you keep source code in `~/source`. If you prefer a different location, adjust all paths accordingly.
- **AI CLI installed and authenticated**: You must have at least one of the following installed, working, and signed in to your account:
  - [Claude Code](https://claude.ai/code) (`claude`)
  - [OpenAI Codex](https://github.com/openai/codex) (`codex`)
  - [Google Gemini CLI](https://github.com/google-gemini/gemini-cli) (`gemini`)

**Note:** Maestro, the orchtestration engine, spawns AI CLIs to take advanage of their agentic nature and avoid having to deal with multiple API calls, handling tool calling, etc. The CLIs all have MCP capability and fine-grained permission settings (for those who wish to use them) so it makes more sense for Maestro to start a CLI and allow it to run in the background until it is finished. If you do not with to use a CLI, an alternate approach would be to instruct your AI assistants to use ClawEh's spawn tool.

### Go Compiler

All three programs are written in Go and we strongly recommend compiling the from source. If you do not have Go installed, please visit the official installation page:

**https://go.dev/doc/install**

Follow the instructions for Linux. After installation, verify it works:

```bash
go version
```

You should see output like `go version go1.26.1 linux/amd64`.

We recommend using the latest version of Go. Earlier versions may work but have not been tested.

Note: If you have just installed Go for the first time, ensure that it is added to your path. After doing so, you will likely need to log out or restart your terminal session for the update PATH to take effect.

---

## Part 1: Installing MCPFusion

MCPFusion is a configuration-driven network MCP (Model Context Protocol) server that connects your AI client to external APIs and local tools via bearer token authentication. It runs as a background service on your machine.

### 1.1 Clone and Build

```bash
mkdir -p ~/source
cd ~/source
git clone https://github.com/PivotLLM/MCPFusion.git
cd MCPFusion
go build -o mcpfusion .
```

WHen the build succeeds, you will have a `mcpfusion` binary in the current directory.

### 1.2 Install the Binary

Create the installation directory and copy the binary:

```bash
sudo mkdir -p /opt/mcpfusion
sudo cp mcpfusion /opt/mcpfusion/mcpfusion
sudo chmod 755 /opt/mcpfusion/mcpfusion
sudo chown -R <USER>:<USER> /opt/mcpfusion
```

### 1.3 Configure MCPFusion

MCPFusion loads one or more JSON configuration files that define which tools it exposes to your AI client. The repository ships with ready-made configurations for several services.

Copy the Maestro configuration (required for Part 2) into the installation directory:

```bash
sudo cp ~/source/MCPFusion/configs/maestro.json /opt/mcpfusion/maestro.json
```

Open the file and update the `command` path to point to where you will install the Maestro binary. **Use the same path you will install Maestro to in Part 2.** A good choice is `~/bin/maestro` (expanded: `/home/<USER>/bin/maestro`):

```bash
sudo vi /opt/mcpfusion/maestro.json
```

The file should look like this, with the command path updated:

```json
{
  "services": {
    "maestro": {
      "name": "Maestro",
      "transport": "mcp_stdio",
      "command": "/home/<USER>/bin/maestro",
      "toolRefreshInterval": "5m"
    }
  }
}
```

Replace `<USER>` with your actual Linux username.

### 1.4 Install the systemd Service

Copy the service file from the repository and install it:

```bash
sudo cp ~/source/MCPFusion/mcpfusion.service /etc/systemd/system/mcpfusion.service
```

MCPFusion reads its environment variables from a file at `/opt/mcpfusion/env`. Create it now:

```bash
sudo vi /opt/mcpfusion/env
```

Add the following content. Configuration file paths are relative to the working directory (`/opt/mcpfusion`):

```
MCP_FUSION_CONFIG=maestro.json
MCP_FUSION_LISTEN=127.0.0.1:8888
MCP_FUSION_DB_DIR=/opt/mcpfusion/db
MCP_FUSION_LOGFILE=/opt/mcpfusion/mcpfusion.log
```

- `MCP_FUSION_CONFIG` — the config file(s) to load. To load multiple files, use a comma-separated list: `maestro.json,playwright.json`
- `MCP_FUSION_LISTEN` — binds MCPFusion to localhost only, which is recommended unless you need remote access
- `MCP_FUSION_DB_DIR` — where MCPFusion stores its token database and per-user knowledge store
- `MCP_FUSION_LOGFILE` — log file path (defaults to `mcpfusion.log` in the working directory if not set)

Set secure permissions on the env file, since it may contain API keys for connected services in the future:

```bash
sudo chmod 640 /opt/mcpfusion/env
sudo chown <USER>:<USER> /opt/mcpfusion/env
```

Now edit the service file to reference this env file:

```bash
sudo vi /etc/systemd/system/mcpfusion.service
```

Ensure that the `Environment=DATA_DIR=...` line points to the correct directory. The relevant portion of the `[Service]` section should look like this:

```ini
[Service]
Type=simple
User=<USER>
Group=<USER>
WorkingDirectory=/opt/mcpfusion
ExecStart=/opt/mcpfusion/mcpfusion
Restart=always
RestartSec=5

Environment=DATA_DIR=/opt/mcpfusion/
```

Enable and start the service. This will, among other things, initialize the database if it does not already exist:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mcpfusion
sudo systemctl start mcpfusion
sudo systemctl status mcpfusion
```

You should see `active (running)` in the status output.

### 1.5 Create an API Token

MCPFusion uses bearer tokens to authenticate clients. Token management commands read and write the same embedded database that the running service uses, so **you must stop MCPFusion before running them** to avoid database locking errors.

```bash
sudo systemctl stop mcpfusion
```

Now generate a token:

```bash
/opt/mcpfusion/mcpfusion -token-add "AIAgents"
```

Copy the token that is displayed. **You will not be able to retrieve it again** — store it somewhere safe. You can always generate additional tokens later using the same stop/add/start sequence.

**Caution:** MCPFusion is designed to be multi-user. Credentials, tokens, and memory items are partitioned by user. This tutorial assumes that you wish all your AI CLIs to access the same MCPFusion context. To do so, use the same token for all of them.

To list existing tokens (shows prefixes only, not full tokens):

```bash
/opt/mcpfusion/mcpfusion -token-list
```

Start MCPFusion again when you are done:

```bash
sudo systemctl start mcpfusion
sudo systemctl status mcpfusion
```

### 1.6 Connect Your AI CLI to MCPFusion

#### Claude Code

```bash
claude mcp add --transport http fusion --scope user http://127.0.0.1:8888/mcp --header "Authorization: Bearer YOUR_TOKEN"
```

### Codex

In theory, codex mcp servers can be managed from the command line with `codex mcp` but specifying a bearer token on the command line is unrealiable. We recommend adding the configuration manually:

```bash
vi ~/.codex/config.toml
```

```bash
[mcp_servers.fusion]
url = "http://127.0.0.1:8888/mcp"
http_headers = {Authorization="Bearer YOUR_TOKEN"}
```

#### Gemini CLI

```bash
gemini mcp add fusion http://127.0.0.1:8888/mcp --scope user --transport http --header "Authorization: Bearer YOUR_TOKEN"
```

Verify the connection by listing configured MCP servers in your CLI:

```bash
# Claude Code
claude mcp list

# Codex
codex mcp list

# Gemini CLI
gemini mcp list
```

### 1.7 Optional: Connect Additional Services

MCPFusion ships with ready-made configuration files for many popular services. Copy any of the following from `~/source/MCPFusion/configs/` to `/opt/mcpfusion/`, then add them to `MCP_FUSION_CONFIGS` in the service file and restart:

- `microsoft365.json` — Calendar, email, contacts, and OneDrive
- `google-workspace.json` — Gmail, Google Calendar, Drive, and Contacts
- `playwright.json` — Browser automation
- `pwndoc.json` — Penetration test report management

MCPFusion can connect to **any REST API** — not just the services listed above. You can create your own JSON configuration file by following the schema documented in the repository. The full configuration guide is at:

`~/source/MCPFusion/docs/config.md`

Your AI assistant can read this documentation and help you write a configuration file for any REST API you provide documentation for. Simply ask it to read the config guide and describe the API you want to connect to.

---

## Part 2: Installing Maestro

Maestro is a stdio MCP server that gives your AI assistant a persistent workspace with three domains: Reference (read-only documentation), Playbooks (reusable processes, procedures, and lists), and Projects (active work). It runs as a local stdio process — MCPFusion proxies it to your AI client.

### 2.1 Clone and Build

```bash
cd ~/source
git clone https://github.com/PivotLLM/Maestro.git
cd Maestro
go build -o maestro .
```

### 2.2 Install the Binary

Install to the same path you configured in the MCPFusion `maestro.json` (step 1.3):

```bash
mkdir -p ~/bin
cp maestro ~/bin/maestro
chmod 755 ~/bin/maestro
```

Verify the binary works:

```bash
~/bin/maestro --version
```

### 2.3 Initial Configuration

On first run, Maestro automatically creates a default configuration at `~/.maestro/config.json`. Run it once to generate the defaults:

```bash
~/bin/maestro
```

It will exit quickly after creating the configuration. Now open the config file:

```bash
vi ~/.maestro/config.json
```

The default config includes entries for Claude Code, Codex CLI, and Gemini CLI — all disabled by default. Enable whichever CLI(s) you have installed.


**Example: enabling Claude Code**

Find the `claude` entry and set `"enabled": true` and the correct command path:

```json
{
  "id": "claude",
  "display_name": "Claude-CLI",
  "type": "command",
  "stdin": true,
  "command": "claude",
  "args": ["-p"],
  "enabled": true
}
```

**Example: enabling Gemini CLI**

```json
{
  "id": "gemini",
  "display_name": "Gemini-CLI",
  "type": "command",
  "stdin": true,
  "command": "gemini",
  "args": ["--yolo", "-p", ""],
  "enabled": true
}
```

**Example: enabling OpenAI Codex CLI**

```json
{
  "id": "gpt",
  "display_name": "Codex-CLI",
  "type": "command",
  "stdin": true,
  "command": "codex",
  "args": ["exec", "--skip-git-repo-check"],
  "enabled": true
}
```
Note that Maestro will search the PATH for the specified `"command"`. If it is not found, you may specific an absolute path. You can also specify additional paths to search at the top level of the configuration file. For example:

`"extra_path": ["~/.nvm/versions/node/v24.12.0/bin", "~/.local/bin"],`

Next, set `"default_llm"` to the `id` of the LLM you want Maestro to use by default (e.g., `"claude"`, `"gpt"`, or `"gemini"`).

### 2.4 Verify the Maestro–MCPFusion Connection

Restart MCPFusion so it picks up the Maestro configuration:

```bash
sudo systemctl restart mcpfusion
sudo systemctl status mcpfusion
```

In your AI CLI, ask it to list available MCP tools. You should see Maestro tools (prefixed with `maestro_`) in the list. You can also ask:

> "Please use the Maestro reference tool to read readme.md and follow the instructions."

This instructs your AI to read Maestro's built-in documentation and orient itself to the available capabilities.

Note: Many AI clients prepend the name of the MCP server or service to the tool name. You may therefore see `fusion_maestro_` plus the tool name.

---

## Part 3 (Optional): Installing ClawEh with Telegram

ClawEh is a lightweight AI assistant gateway that connects messaging channels (Telegram, Slack, and others) to AI backends. This section shows how to set it up with Telegram and Claude Code in **unified mode** — the recommended starting point for a personal assistant.

### 3.1 Prerequisites: Create a Telegram Bot

1. Open Telegram and search for `@BotFather`
2. Send `/newbot` and follow the prompts to create a bot
3. Copy the **bot token** BotFather gives you (format: `1234567890:ABCDEF...`)
4. Find your **Telegram user ID** by messaging `@userinfobot`

### 3.2 Clone and Build

```bash
cd ~/source
git clone https://github.com/PivotLLM/ClawEh.git
cd ClawEh
go build -o claw ./cmd/claw
```

### 3.3 Install the Binary

```bash
mkdir -p ~/.local/bin
cp claw ~/.local/bin/claw
chmod 755 ~/.local/bin/claw
```

Ensure `~/.local/bin` is in your `PATH`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash
claw version
```

### 3.4 Initialize ClawEh

```bash
claw onboard
```

This creates a default configuration at `~/.claw/config.json`.

### 3.5 Understanding Unified Mode

Before editing the configuration, it is worth understanding **unified mode**, which is what this tutorial configures.

ClawEh supports several session modes that control how your assistant's memory is divided:

| Mode | Each person gets... |
|---|---|
| `unified` | One shared memory for the entire agent, across all users and channels |
| `per-user` | Their own private memory |
| `per-platform` | A separate memory per platform (Telegram vs Slack, etc.) |

**Unified mode** means there is one Alice. All conversations — regardless of who sends them or through which channel — contribute to and draw from the same accumulated context. Alice remembers what she learned from your morning Telegram message when you follow up that evening from Slack. She does not maintain separate conversations per person or per channel; she is one coherent assistant with one continuous memory.

This is the right choice when:
- You are the only user (or all users are fully trusted and you *want* them to share context)
- You want the assistant to be a consistent identity — one Alice, with one accumulated knowledge base

If you want each person to have a private, independent relationship with the assistant, use `per-user` instead.

> **Security note:** In unified mode, every user who can reach the assistant shares the same memory. Only grant access to people you trust completely by listing their IDs in `allow_from`. An empty `allow_from` list means nobody can connect.

### 3.6 Configure ClawEh

Edit `~/.claw/config.json`. Replace its contents with the following, substituting your values where indicated:

```json
{
  "session": {
    "mode": "unified"
  },
  "agents": {
    "defaults": {
      "model": "claude-code",
      "request_timeout": 900,
      "max_tool_iterations": 50,
      "summarize_message_threshold": 20,
      "summarize_token_percent": 75
    },
    "list": [
      {
        "id": "alice",
        "name": "Alice",
        "default": true
      }
    ]
  },
  "channels": {
    "telegram": [
      {
        "id": "alice",
        "enabled": true,
        "token": "YOUR_TELEGRAM_BOT_TOKEN",
        "allow_from": ["YOUR_TELEGRAM_USER_ID"],
        "typing": {
          "enabled": true
        },
        "placeholder": {
          "enabled": true,
          "text": "Thinking..."
        }
      }
    ]
  },
  "model_list": [
    {
      "model_name": "claude-code",
      "model": "claude-cli/claude-code",
      "request_timeout": 900
    }
  ]
}
```

- Replace `YOUR_TELEGRAM_BOT_TOKEN` with the token from BotFather
- Replace `YOUR_TELEGRAM_USER_ID` with your numeric Telegram user ID
- The `claude-cli/claude-code` model string tells ClawEh to use the `claude` CLI without specifying a model, so the CLI uses whichever model is currently configured in your Claude Code subscription
- The agent is named `alice` — you can change this to any name you like; it becomes part of the internal channel name (`telegram-alice`)

> **Note on MCP with ClawEh:** When ClawEh uses a CLI backend like Claude Code, it is unable to pass MCP tool definitions in their native structured form. As a workaround, ClawEh passes MCP tool definitions as text. For this reason, when using ClawEh with a CLI such as Claude Code, Codex, or Gemini CLI, it is much more efficient to configure the CLI(s) to connect to MCPFusion directly (as done in Part 1 above) rather than configuring ClawEh to use MCPFusion directly. If you choose to use an API such as OpenRouter instead, then you will need to configure ClawEh to connect directly to MCPFusion so that it can provide tool call services.

### 3.7 Install the systemd Service

Copy the service file from the repository and install it:

```bash
sudo cp ~/source/ClawEh/claw.service /etc/systemd/system/claw.service
sudo vi /etc/systemd/system/claw.service
```

Replace `YOUR_USERNAME` with your Linux username (two occurrences). The file should look like this:

```ini
[Unit]
Description=Claw AI Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=<USER>
ExecStart=/home/<USER>/.local/bin/claw gateway
Restart=on-failure
RestartSec=5
KillMode=control-group
TimeoutStopSec=30
Environment=PATH=/home/<USER>/.local/bin:/usr/local/bin:/usr/bin:/bin

[Install]
WantedBy=multi-user.target
```

ClawEh writes its own log to `~/.claw/logs/claw.log` — no output redirection is needed in the service file.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable claw
sudo systemctl start claw
sudo systemctl status claw
```

### 3.8 Test the Bot

Open Telegram, find your bot by the username you gave it in BotFather, and send it a message. It should respond using Claude Code. Because you configured unified mode, Alice will remember this conversation the next time you message her, from any channel you have configured.

---

## Troubleshooting

### MCPFusion

**Log file:** `/opt/mcpfusion/mcpfusion.log`

This is the primary log for diagnosing connection problems, configuration errors, and authentication failures. The log location is set by `MCP_FUSION_LOGFILE` in `/opt/mcpfusion/env`. If that variable is not set, MCPFusion writes `mcpfusion.log` to its working directory (`/opt/mcpfusion`) by default.

To watch the log in real time:

```bash
tail -f /opt/mcpfusion/mcpfusion.log
```

systemd also captures stdout/stderr to the journal:

```bash
sudo journalctl -u mcpfusion -f
```

**Common issues:**

- *Service fails to start* — check the journal for startup errors; the most common cause is a malformed JSON config file or a missing path in `maestro.json`
- *401 Unauthorized from your CLI* — the bearer token in your CLI configuration does not match any token in MCPFusion's database; regenerate using the stop/add/start sequence in step 1.5
- *Maestro tools not appearing* — verify the `command` path in `maestro.json` points to the correct Maestro binary and that the binary is executable; check the log for stdio launch errors
- *Database lock error when adding a token* — the service is still running; stop it first with `sudo systemctl stop mcpfusion` before running token management commands

### Maestro

Maestro has two levels of logging:

**General log** — records startup, configuration, runner activity, and errors:

```
~/.maestro/maestro.log
```

The log file name and level are controlled by the `logging` section in `~/.maestro/config.json`:

```json
"logging": {
  "file": "maestro.log",
  "level": "INFO"
}
```

The path is relative to `base_dir` (`~/.maestro` by default), so the full path is `~/.maestro/maestro.log`.

**Project logs** — each project has its own log recording task execution, LLM dispatch, and runner activity:

```
~/.maestro/data/projects/<project-name>/log.txt
```

For example, a project named `my-research` would have its log at:

```
~/.maestro/data/projects/my-research/log.txt
```

To watch a project log in real time:

```bash
tail -f ~/.maestro/data/projects/my-research/log.txt
```

**Common issues:**

- *No Maestro tools visible in your CLI* — ensure MCPFusion is running and that `maestro.json` has the correct binary path; check `/opt/mcpfusion/mcpfusion.log` for errors launching the stdio process
- *Tasks not executing* — confirm that at least one LLM is configured with `"enabled": true` in `~/.maestro/config.json` and that the `command` path is correct; check `~/.maestro/maestro.log` for errors

### ClawEh

ClawEh writes its log to:

```
~/.claw/logs/claw.log
```

To watch it in real time:

```bash
tail -f ~/.claw/logs/claw.log
```

systemd also captures any output the service itself writes before the log is initialised:

```bash
sudo journalctl -u claw -f
```

**Common issues:**

- *Bot does not respond* — verify the Telegram token is correct and that your user ID is listed in `allow_from`; check the log for "rejected by allowlist" messages
- *Claude Code not found* — ensure the `claude` binary is in the PATH of the user running the service; the `Environment=PATH=...` line in the service file must include the directory containing the `claude` binary; you can test this by running `su -s /bin/bash <USER> -c "which claude"`
- *Session mode warning at startup* — if the log shows "Unrecognized session mode", the `mode` value in your `session` config block is not valid; valid values are `unified`, `per-user`, `per-platform`, and `per-account`

---

## Quick Reference

| Component  | Binary Location              | Config Location              | Log Location                                      | Service         |
|------------|------------------------------|------------------------------|---------------------------------------------------|-----------------|
| MCPFusion  | `/opt/mcpfusion/mcpfusion`   | `/opt/mcpfusion/env`, `/opt/mcpfusion/*.json` | `/opt/mcpfusion/mcpfusion.log` | `mcpfusion`     |
| Maestro    | `~/bin/maestro`              | `~/.maestro/config.json`     | `~/.maestro/maestro.log`, `~/.maestro/data/projects/<name>/log.txt` | (stdio, no service needed) |
| ClawEh     | `~/.local/bin/claw`          | `~/.claw/config.json`        | `~/.claw/logs/claw.log`                           | `claw`          |

### Useful Commands

```bash
# Watch MCPFusion log
tail -f /opt/mcpfusion/mcpfusion.log

# Watch Maestro general log
tail -f ~/.maestro/maestro.log

# Watch a Maestro project log
tail -f ~/.maestro/data/projects/<project-name>/log.txt

# View ClawEh log
tail -f ~/.claw/logs/claw.log

# Restart MCPFusion after config changes
sudo systemctl restart mcpfusion

# Add a new MCPFusion token (must stop service first)
sudo systemctl stop mcpfusion
/opt/mcpfusion/mcpfusion -token-add "Description"
sudo systemctl start mcpfusion

# List MCPFusion tokens
/opt/mcpfusion/mcpfusion -token-list
```
