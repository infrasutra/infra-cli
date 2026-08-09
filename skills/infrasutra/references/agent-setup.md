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

## Claude Code

Skill (preferred). Drop this directory in and it loads on demand:

```
.claude/skills/infrasutra/
├── SKILL.md
└── references/
```

Or install it for every project at `~/.claude/skills/infrasutra/`.

MCP, if you want the typed tools as well — `.mcp.json` in the project root:

```json
{
  "mcpServers": {
    "infrasutra": { "command": "infra", "args": ["mcp"] }
  }
}
```

## Codex

Add the same server to your Codex MCP config:

```toml
[mcp_servers.infrasutra]
command = "infra"
args = ["mcp"]
```

Codex reads `AGENTS.md`, so the fastest way to give it the opinions in this
skill is to reference them there:

```markdown
## Deploying
Deploy with `infra up --build local` (builds here, pushes, platform deploys the
image — no build minutes, warm cache). Check for a Dockerfile first; see
.claude/skills/infrasutra/SKILL.md.
```

## Cline · Cursor · Windsurf · Continue

All take the same MCP server entry:

```json
{
  "mcpServers": {
    "infrasutra": { "command": "infra", "args": ["mcp"] }
  }
}
```

They can also run shell commands, so the CLI works without any configuration —
put the deploy rules in the project's rules file (`.clinerules`,
`.cursor/rules/`) and point them at the CLI.

## OpenCode · Aider · Grok CLI · Antigravity · anything with a terminal

No configuration. The agent runs `infra` directly. Give it the rules in
whatever the tool's convention is (`AGENTS.md`, `CONVENTIONS.md`, a rules file)
and keep them short:

```markdown
Deploy: `infra up --build local`. Never `docker push` by hand.
No Dockerfile? Write a multi-stage, non-root one listening on $PORT first.
Monorepo? One infra.yaml, one Dockerfile per service, build context = repo root.
Debug: `infra services get <name>` explains a failure in the platform's own words.
```

## ChatGPT connectors and claude.ai

These have no local machine, so `infra mcp` over stdio does not apply — they
need a hosted MCP endpoint with OAuth. Not available yet.

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
