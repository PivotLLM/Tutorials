# Getting Started with MCPFusion and Maestro

This tutorial walks you through setting up MCPFusion and Maestro — a powerful combination that gives your AI assistant persistent knowledge management, project orchestration, and access to external APIs.

---

## Prerequisites

Before you begin, ensure the following:

- **Operating system**: Linux with `sudo` access
- **Source directory**: These instructions assume you keep source code in `~/source`. If you prefer a different location, adjust all paths accordingly.
- **AI CLI installed and authenticated**: You must have at least one of the following installed and signed in to your account:
  - [Claude Code](https://claude.ai/code) (`claude`)
  - [OpenAI Codex CLI](https://github.com/openai/codex) (`codex`)
  - [Google Gemini CLI](https://github.com/google-gemini/gemini-cli) (`gemini`)

### Go Compiler

Both MCPFusion and Maestro are written in Go and must be compiled from source. If you do not have Go installed, visit the official installation page:

**https://go.dev/doc/install**

Follow the instructions for Linux. After installation, verify it works:

```bash
go version
```

You should see output like `go version go1.23.0 linux/amd64`. Any version 1.21 or later should work.

---

## Part 1: Installing MCPFusion

MCPFusion is a configuration-driven MCP (Model Context Protocol) server that connects your AI client to external APIs and local tools via bearer token authentication. It runs as a background service on your machine.

### 1.1 Clone and Build

```bash
mkdir -p ~/source
cd ~/source
git clone https://github.com/PivotLLM/MCPFusion.git
cd MCPFusion
go build -o mcpfusion .
```

If the build succeeds, you will have a `mcpfusion` binary in the current directory.

### 1.2 Install the Binary

Create the installation directory and copy the binary:

```bash
sudo mkdir -p /opt/mcpfusion
sudo cp mcpfusion /opt/mcpfusion/mcpfusion
sudo chmod 755 /opt/mcpfusion/mcpfusion
```

### 1.3 Create a Dedicated Service User

MCPFusion runs as its own system user for security:

```bash
sudo useradd --system --no-create-home --shell /sbin/nologin mcpfusion
sudo chown -R mcpfusion:mcpfusion /opt/mcpfusion
```

### 1.4 Configure MCPFusion

MCPFusion loads one or more JSON configuration files that define which tools it exposes to your AI client. The repository ships with ready-made configurations for many services.

Copy the Maestro configuration (required for Part 2) into the installation directory:

```bash
sudo cp ~/source/MCPFusion/configs/maestro.json /opt/mcpfusion/maestro.json
```

Open the file and update the `command` path to point to where you will install the Maestro binary. **Use the same path you will install Maestro to in Part 2.** A good choice is `~/bin/maestro` (expanded: `/home/YOUR_USERNAME/bin/maestro`):

```bash
sudo nano /opt/mcpfusion/maestro.json
```

The file should look like this, with the command path updated:

```json
{
  "services": {
    "maestro": {
      "name": "Maestro",
      "transport": "mcp_stdio",
      "command": "/home/YOUR_USERNAME/bin/maestro",
      "toolRefreshInterval": "5m"
    }
  }
}
```

Replace `YOUR_USERNAME` with your actual Linux username.

### 1.5 Install the systemd Service

Copy the service file from the repository and install it:

```bash
sudo cp ~/source/MCPFusion/mcpfusion.service /etc/systemd/system/mcpfusion.service
```

MCPFusion reads its environment variables from a file at `/opt/mcpfusion/env`. Create it now:

```bash
sudo nano /opt/mcpfusion/env
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
sudo chown mcpfusion:mcpfusion /opt/mcpfusion/env
```

Now edit the service file to reference this env file:

```bash
sudo nano /etc/systemd/system/mcpfusion.service
```

Replace the `Environment=DATA_DIR=...` line with an `EnvironmentFile` directive. The relevant portion of the `[Service]` section should look like this:

```ini
[Service]
Type=simple
User=mcpfusion
Group=mcpfusion
WorkingDirectory=/opt/mcpfusion
ExecStart=/opt/mcpfusion/mcpfusion
Restart=always
RestartSec=5

EnvironmentFile=/opt/mcpfusion/env
```

> **Note:** When you add additional service configurations later (e.g., Microsoft 365, Google Workspace), any API credentials or OAuth client IDs for those services also go into `/opt/mcpfusion/env`.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mcpfusion
sudo systemctl start mcpfusion
sudo systemctl status mcpfusion
```

You should see `active (running)` in the status output.

### 1.6 Create an API Token

MCPFusion uses bearer tokens to authenticate clients. Token management commands read and write the same embedded database that the running service uses, so **you must stop MCPFusion before running them** to avoid database locking errors.

```bash
sudo systemctl stop mcpfusion
```

Now generate a token:

```bash
sudo -u mcpfusion /opt/mcpfusion/mcpfusion -token-add "My AI CLI token"
```

Copy the token that is printed. **You will not be able to retrieve it again** — store it somewhere safe. You can always generate additional tokens later using the same stop/add/start sequence.

To list existing tokens (shows prefixes only, not full tokens):

```bash
sudo -u mcpfusion /opt/mcpfusion/mcpfusion -token-list
```

Start MCPFusion again when you are done:

```bash
sudo systemctl start mcpfusion
sudo systemctl status mcpfusion
```

### 1.7 Connect Your AI CLI to MCPFusion

#### Claude Code

```bash
claude mcp add --transport http fusion --scope user http://127.0.0.1:8888/mcp --header "Authorization: Bearer YOUR_TOKEN"
```

#### Gemini CLI

```bash
gemini mcp add fusion http://127.0.0.1:8888/mcp --scope user --transport http --header "Authorization: Bearer YOUR_TOKEN"
```

Or add it manually to `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "fusion": {
      "url": "http://127.0.0.1:8888/mcp",
      "type": "http",
      "headers": {
        "Authorization": "Bearer YOUR_TOKEN"
      }
    }
  }
}
```

Replace `YOUR_TOKEN` with the token generated in step 1.6.

Verify the connection by listing configured MCP servers in your CLI:

```bash
# Claude Code
claude mcp list

# Gemini CLI
gemini mcp list
```

### 1.8 Optional: Connect Additional Services

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

Maestro is an MCP server that gives your AI a persistent workspace with three domains: Reference (read-only documentation), Playbooks (reusable procedures), and Projects (active work). It runs as a local stdio process — MCPFusion proxies it to your AI client.

### 2.1 Clone and Build

```bash
cd ~/source
git clone https://github.com/PivotLLM/Maestro.git
cd Maestro
go build -o maestro .
```

### 2.2 Install the Binary

Install to the same path you configured in the MCPFusion `maestro.json` (step 1.4):

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
nano ~/.maestro/config.json
```

The default config includes entries for Claude Code, Codex CLI, and Gemini CLI — all disabled by default. Enable whichever CLI you have installed and update the `command` path to the correct location.

**Find the path to your CLI:**

```bash
which claude    # for Claude Code
which codex     # for OpenAI Codex
which gemini    # for Gemini CLI
```

**Example: enabling Claude Code**

Find the `claude` entry and set `"enabled": true` and the correct command path:

```json
{
  "id": "claude",
  "display_name": "Claude-CLI",
  "type": "command",
  "stdin": true,
  "command": "/usr/local/bin/claude",
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
  "command": "/usr/local/bin/gemini",
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
  "command": "/usr/local/bin/codex",
  "args": ["exec", "--skip-git-repo-check"],
  "enabled": true
}
```

Set `"default_llm"` to the `id` of the LLM you want Maestro to use by default (e.g., `"claude"`, `"gpt"`, or `"gemini"`).

### 2.4 Verify the Maestro–MCPFusion Connection

Restart MCPFusion so it picks up the Maestro configuration:

```bash
sudo systemctl restart mcpfusion
sudo systemctl status mcpfusion
```

In your AI CLI, ask it to list the available Fusion tools. You should see Maestro tools (prefixed with `maestro_`) in the list. You can also ask:

> "Please use the Maestro reference tool to read readme.md and follow the instructions."

This instructs your AI to read Maestro's built-in documentation and orient itself to the available capabilities.

---

## Part 3 (Optional): Installing PicoClaw with Telegram

> **Note:** At the time of writing, PicoClaw's Claude CLI integration is the primary tested configuration. Only Claude Code is covered in this section. Support for Codex and Gemini CLIs may be available — check the PicoClaw repository for the latest status.

PicoClaw is an ultra-lightweight AI assistant gateway that connects messaging channels (Telegram, Discord, Slack, and many others) to AI backends. This section shows how to set it up with Telegram and Claude Code.

### 3.1 Prerequisites: Create a Telegram Bot

1. Open Telegram and search for `@BotFather`
2. Send `/newbot` and follow the prompts to create a bot
3. Copy the **bot token** BotFather gives you (format: `1234567890:ABCDEF...`)
4. Find your **Telegram user ID** by messaging `@userinfobot`

### 3.2 Clone and Build

```bash
cd ~/source
git clone https://github.com/sipeed/picoclaw.git
cd picoclaw
make build
```

### 3.3 Install the Binary

```bash
make install
```

This installs the `picoclaw` binary to `~/.local/bin/picoclaw`. Ensure `~/.local/bin` is in your `PATH`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Verify:

```bash
picoclaw --version
```

### 3.4 Create the Configuration Directory

```bash
mkdir -p ~/.picoclaw
```

### 3.5 Configure PicoClaw

Create `~/.picoclaw/config.json` with the following content. Replace the placeholder values with your own:

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.picoclaw/workspace",
      "restrict_to_workspace": true,
      "model_name": "claude-code",
      "max_tokens": 32768,
      "max_tool_iterations": 50,
      "summarize_message_threshold": 20,
      "summarize_token_percent": 75
    }
  },
  "channels": {
    "telegram": {
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
- The `model_name: "claude-code"` sentinel tells PicoClaw to use the `claude` CLI without specifying a model, allowing the CLI to use its currently configured model

> **Note on MCP with PicoClaw + Claude CLI:** When PicoClaw uses a CLI backend like Claude Code, it passes MCP tool definitions as text rather than native tool calls. For this reason, it is more efficient to configure the Claude CLI to connect to MCPFusion directly (as done in Part 1) rather than also configuring it as an MCP server inside PicoClaw.

### 3.6 Install the systemd Service

Create a service file:

```bash
sudo nano /etc/systemd/system/picoclaw.service
```

Paste the following, replacing `YOUR_USERNAME` with your Linux username:

```ini
[Unit]
Description=PicoClaw AI Gateway
After=network.target

[Service]
Type=simple
User=YOUR_USERNAME
WorkingDirectory=/home/YOUR_USERNAME
ExecStart=/home/YOUR_USERNAME/.local/bin/picoclaw gateway
Restart=always
RestartSec=5

StandardOutput=journal
StandardError=journal
SyslogIdentifier=picoclaw

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable picoclaw
sudo systemctl start picoclaw
sudo systemctl status picoclaw
```

### 3.7 Test the Bot

Open Telegram, find your bot by the username you gave it in BotFather, and send it a message. It should respond using Claude Code.

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
- *401 Unauthorized from your CLI* — the bearer token in your CLI configuration does not match any token in MCPFusion's database; regenerate using the stop/add/start sequence in step 1.6
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

### PicoClaw

```bash
sudo journalctl -u picoclaw -f
```

**Common issues:**

- *Bot does not respond* — verify the Telegram token is correct and that your user ID is listed in `allow_from`
- *Claude Code not found* — ensure the `claude` binary is in the PATH of the user running the service; you can test this by running `su -s /bin/bash YOUR_USERNAME -c "which claude"`

---

## Quick Reference

| Component  | Binary Location              | Config Location              | Log Location                                      | Service         |
|------------|------------------------------|------------------------------|---------------------------------------------------|-----------------|
| MCPFusion  | `/opt/mcpfusion/mcpfusion`   | `/opt/mcpfusion/env`, `/opt/mcpfusion/*.json` | `/opt/mcpfusion/mcpfusion.log` | `mcpfusion`     |
| Maestro    | `~/bin/maestro`              | `~/.maestro/config.json`     | `~/.maestro/maestro.log`, `~/.maestro/data/projects/<name>/log.txt` | (stdio, no service needed) |
| PicoClaw   | `~/.local/bin/picoclaw`      | `~/.picoclaw/config.json`    | `journalctl -u picoclaw`                          | `picoclaw`      |

### Useful Commands

```bash
# Watch MCPFusion log
tail -f /opt/mcpfusion/mcpfusion.log

# Watch Maestro general log
tail -f ~/.maestro/maestro.log

# Watch a Maestro project log
tail -f ~/.maestro/data/projects/<project-name>/log.txt

# View PicoClaw logs
sudo journalctl -u picoclaw -f

# Restart MCPFusion after config changes
sudo systemctl restart mcpfusion

# Add a new MCPFusion token (must stop service first)
sudo systemctl stop mcpfusion
sudo -u mcpfusion /opt/mcpfusion/mcpfusion -token-add "Description"
sudo systemctl start mcpfusion

# List MCPFusion tokens
sudo -u mcpfusion /opt/mcpfusion/mcpfusion -token-list
```
