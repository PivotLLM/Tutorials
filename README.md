# Tutorials

This repo is intended to help you get started with our open source software.

## Guides

- **[Getting Started](docs/getting-started.md)** — Install and configure MCPFusion, Maestro, and (optionally) PicoClaw with Telegram

## Our Software

- **[MCPFusion](https://github.com/PivotLLM/MCPFusion)** is a configuration-driven MCP server that connects AI clients to external APIs, upstream MCP servers (network and stdio), and local memory tools. Clients only need to connect to one MCP tool to obtain access to all resources the user chooses to configure. It has been tested with Claude Code, Codex, Gemini CLI, and other MCP-compatible clients.

- **[Maestro](https://github.com/PivotLLM/Maestro)** is a local (stdio) MCP server that provides a methodology and suite of tools to orchestrate projects, enable repeatable processes, and promote continuous improvement. Its flexible design can be used to guide an LLM through a process, and it supports delegating series of tasks to subagents for work and quality control.
  
## Third Party Software

- **[PicoClaw](https://github.com/sipeed/picoclaw)** is an ultra-lightweight AI Assistant. We are currently using it to allow access via Telegram and have included instructions to do so in our tutorial. Our [fork](https://github.com/securityguy/picoclaw) is for testing and development, and we contribute any changes back to the PicoClaw project as pull requests.

**WARNING: We have identified several bugs in PicoClaw with the potential to impact those using Claude Code, Codex, and Gemini CLI. For example, improper parsing of the returned JSON causes some tool calls intended for PicoClaw to be ignored and forwarded to the user as a message. These tool calls are used for scheduling, engaging sub-agents, etc. If your use case includes this functionality, you may wish to clone from https://github.com/securityguy/picoclaw until the PicoClaw project is able to address the large volume of PRs and address these issues.**

## Copyright and license

Copyright (c) 2026 by Tenebris Technologies Inc. The contents of this repository are licensed under the MIT License. Please see LICENSE for details.

## Trademarks

Any trademarks referenced are the property of their respective owners, used for identification only, and do not imply sponsorship, endorsement, or affiliation.

## No Warranty (zilch, none, void, nil, null, "", {}, 0x00, 0b00000000, EOF)

THIS SOFTWARE IS PROVIDED “AS IS,” WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, AND NON-INFRINGEMENT. IN NO EVENT SHALL THE COPYRIGHT HOLDERS OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

Made in Canada
