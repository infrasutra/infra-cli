# Wiring an AI coding agent to Infrasutra

Two surfaces, same behaviour underneath:

- **The CLI**, driven by this skill. Prefer it wherever the agent can run
  commands: it composes with pipes, it streams a build as it happens, and it
  costs no context until it is used.
- **`infra mcp`**, for hosts that cannot run a shell. Same client, same
  results — read tools plus deployment guidance. Anything destructive stays in
  the CLI, where a human reads the command before it runs.

Install the CLI first, either way:

```sh
curl -fsSL https://infrasutra.com/public/cli/install.sh | sh
infra auth login
```

## One command, any agent

```sh
infra skill install
```

It looks at the project, works out which agents it is set up for, and writes
the right thing for each — the skill tree for Claude Code, `AGENTS.md` for
Codex and everything else that reads it, a rules file for Cursor, Cline or
Windsurf. Nothing configured yet? It writes `AGENTS.md`, which nearly every
agent reads.

```sh
infra skill list              # what this project uses, and what you have installed
infra skill install --agent claude
infra skill install --global  # every project on this machine; writes nothing into this one
infra skill print             # the guidance itself, to paste somewhere else
```

`--global` covers the agents with a user-level config: Claude Code and Codex.
Cursor, Cline and Windsurf only read project files, so naming one with
`--global` is an error rather than a file written somewhere you did not ask.

The guidance is embedded in the binary, so it always matches the CLI you have,
works offline, and cannot describe a flag your version does not support.
Re-running updates in place: the block it owns in `AGENTS.md` is delimited by
markers, and everything you wrote around it is left alone.

To pull the newest published guidance instead — accepting that it may describe
flags a older CLI does not have:

```sh
infra skill install --refresh
infra skill install --refresh --ref v0.2.0   # a specific published version
```

If the download fails it falls back to the embedded copy rather than leaving
you with nothing.

## What each agent gets

| Agent | In the project | With `--global` |
|-------|----------------|-----------------|
| Claude Code | `.claude/skills/infrasutra/` | `~/.claude/skills/infrasutra/` + MCP at user scope |
| Codex | `AGENTS.md` + `[mcp_servers.infrasutra]` in `.codex/config.toml` | the same in `~/.codex/` |
| Cursor | `.cursor/rules/infrasutra.mdc` (alwaysApply) | — |
| Cline | `.clinerules` — file or directory, whichever you have | — |
| Windsurf | `.windsurf/rules/infrasutra.md` | — |
| Anything else | `AGENTS.md` | — |

Claude Code reads a project's MCP servers only from `.mcp.json` at the repository
root, so a project install skips MCP rather than adding a file there: Claude Code
has a shell and drives the CLI directly. `--global` registers the server with
`claude mcp add --scope user`; to share it through a repository instead, run
`claude mcp add --scope project infrasutra -- infra mcp`.

`AGENTS.md` is an open specification stewarded by the Linux Foundation and read
by Claude Code, Codex, Cursor, Aider, Copilot, Gemini CLI, Continue, Windsurf,
Zed, Amp and others — which is why it is the fallback.

## Doing it by hand

The skill is a directory of markdown; copy it wherever your tool expects one:

```sh
infra skill print --full > wherever-your-agent-reads.md
```

For an MCP host that can send a header, use the hosted server. It needs nothing
running locally and takes a read-only key you can revoke:

```bash
infra mcp setup
```

It prints the key and the config for Claude Code, Cursor and VS Code. It needs an
API release that includes `POST /api/v1/mcp`; `infra doctor` reports the API this
CLI talks to.

For a host that can only launch a command, the server is the CLI itself:

```json
{
  "mcpServers": {
    "infrasutra": { "command": "infra", "args": ["mcp"] }
  }
}
```

Codex uses TOML for the same thing:

```toml
[mcp_servers.infrasutra]
command = "infra"
args = ["mcp"]
```

## Agents with a terminal and no MCP

OpenCode, Aider, Grok CLI, Antigravity and anything else that can run a shell
need no MCP at all — they call `infra` directly. `infra skill install` still
helps: it writes `AGENTS.md`, which those tools read.

## ChatGPT connectors and claude.ai

These have no local machine, so `infra mcp` over stdio does not apply. Connector
UIs that expect OAuth can't use the hosted server yet, because it takes access
keys only. A host that lets you set an Authorization header can use it today.

## Checking the wiring

```sh
infra auth status          # signed in?
infra services list        # can it see anything?
```

For MCP, the handshake alone tells you the server is reachable:

```sh
printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | infra mcp
```

A `result` with `serverInfo` means the host will be able to talk to it. If it
reports "not signed in", run `infra auth login` — the MCP server refuses to
start unauthenticated rather than failing on the first tool call, which reads
as a broken tool.
