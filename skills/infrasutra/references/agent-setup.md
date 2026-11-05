# Wiring an AI coding agent to Infrasutra

Agents use the `infra` CLI, driven by this skill. It composes with pipes, it streams
a build as it happens, and it costs no context until it is used. Destructive commands
need a person to type the resource's name in a real terminal, so an agent gives the
user the command instead of running it.

The whole setup is three commands:

```sh
curl -fsSL https://infrasutra.com/public/cli/install.sh | sh
infra auth login
infra skill install
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
| Claude Code | `.claude/skills/infrasutra/` | `~/.claude/skills/infrasutra/` |
| Codex | `AGENTS.md` | `~/.codex/AGENTS.md` |
| Cursor | `.cursor/rules/infrasutra.mdc` (alwaysApply) | — |
| Cline | `.clinerules` — file or directory, whichever you have | — |
| Windsurf | `.windsurf/rules/infrasutra.md` | — |
| Anything else | `AGENTS.md` | — |

`AGENTS.md` is an open specification stewarded by the Linux Foundation and read
by Claude Code, Codex, Cursor, Aider, Copilot, Gemini CLI, Continue, Windsurf,
Zed, Amp and others — which is why it is the fallback.

## Doing it by hand

The skill is a directory of markdown; copy it wherever your tool expects one:

```sh
infra skill print --full > wherever-your-agent-reads.md
```

## Other agents with a terminal

OpenCode, Aider, Grok CLI, Antigravity and anything else that can run a shell call
`infra` directly. `infra skill install` writes `AGENTS.md`, which those tools read.

## Checking the wiring

```sh
infra auth status          # signed in?
infra services list        # can it see anything?
```
