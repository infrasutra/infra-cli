# Wiring an AI coding agent to Infrasutra

Agents use the `infra` CLI, driven by this skill. It composes with pipes, it streams
a build as it happens, and it costs no context until it is used. Destructive commands
need a person to type the resource's name in a real terminal, so an agent gives the
user the command instead of running it.

With Claude Code or Codex, the whole setup is three commands:

```sh
curl -fsSL https://infrasutra.com/public/cli/install.sh | sh
infra auth login
infra skill install
```

## Installing the guidance

```sh
infra skill install
```

It finds Claude Code and Codex on this machine and sets them up for every
project, writing nothing into the current folder. Run it once, from anywhere. It
follows `CLAUDE_CONFIG_DIR` and `CODEX_HOME` if you set them.

Cursor, Cline, Windsurf and the rest only read project files, so the command above
does nothing for them, and fails on a machine with neither Claude Code nor Codex.
For those agents, or to commit the guidance so your team gets it, run this in the
project:

```sh
infra skill install --project
```

It works out which agents the project is set up for and writes the right thing
for each — the skill tree for Claude Code, `AGENTS.md` for Codex and everything
else that reads it, a rules file for Cursor, Cline or Windsurf. Nothing
configured yet? It writes `AGENTS.md`, which nearly every agent reads.

```sh
infra skill list              # what would be set up, on this machine and in this folder
infra skill install --agent claude
infra skill print             # the guidance itself, to paste somewhere else
```

Nothing goes into a project without `--project`, so naming Cursor, Cline, Windsurf
or `agents` without it is an error. On this command `--project` is a switch, not the
project name it takes elsewhere. Claude Code uses the machine-wide copy over one in the
project.

The guidance is embedded in the binary, so it always matches the CLI you have,
works offline, and cannot describe a flag your version does not support.
Re-running updates in place: the block it owns in `AGENTS.md` is delimited by
markers, and everything you wrote around it is left alone.

To pull the newest published guidance instead — accepting that it may describe
flags an older CLI does not have:

```sh
infra skill install --refresh
infra skill install --refresh --ref v0.2.0   # a specific published version
```

If the download fails it falls back to the embedded copy rather than leaving
you with nothing.

## What each agent gets

| Agent | By default | With `--project` |
|-------|------------|------------------|
| Claude Code | `~/.claude/skills/infrasutra/` | `.claude/skills/infrasutra/` |
| Codex | `~/.codex/AGENTS.md` | `AGENTS.md` |
| Cursor | — | `.cursor/rules/infrasutra.mdc` (alwaysApply) |
| Cline | — | `.clinerules` — file or directory, whichever you have |
| Windsurf | — | `.windsurf/rules/infrasutra.md` |
| Anything else | — | `AGENTS.md` |

With `CLAUDE_CONFIG_DIR` or `CODEX_HOME` set, the default goes to
`$CLAUDE_CONFIG_DIR/skills/infrasutra/` and `$CODEX_HOME/AGENTS.md` instead.

`AGENTS.md` is an open specification stewarded by the Linux Foundation and read
by Claude Code, Codex, Cursor, Aider, Copilot, Gemini CLI, Continue, Windsurf,
Zed, Amp and others — which is why it is the fallback.

## Doing it by hand

The skill is a directory of markdown; copy it wherever your tool expects one:

```sh
infra skill print --full > wherever-your-agent-reads.md
```

## Apps without a terminal

Claude on the web, desktop and mobile, ChatGPT, and cloud sessions can't always run
`infra`, so they connect to the hosted server instead. `infra connect` prints the
steps for each app and the address to use; `infra connect claude` opens Claude's
connector page with the address filled in. Connect once on the web and the mobile
app has it too.

Signing in shows which organization the app will work in. Deploying stays off
unless you tick "Also let it deploy" on that page. With it on, the app can read services and logs, create projects, deploy an image or a Git
repository, redeploy and roll back. It can't delete anything, and it can't switch a
running service to a different image: a person does that, because that image would
get the service's secrets. What it deploys does run with your project's environment
variables, so only allow deploying for an app you trust. Each app has to be connected
to each account separately, and it works in the organization you connected it to.

To end an app's access, disconnect it in the dashboard under Settings, Connections.
Removing a connector inside Claude alone leaves its access active on Infrasutra.

Which path fits which app:

| App | Use |
|-----|-----|
| Claude Code, in the terminal or the desktop app's Code tab | The CLI and this skill |
| Claude on the web, desktop chat, Cowork, mobile | The connector |
| Claude Code on the web | The connector: it's passed into the session, past the network allowlist |
| Codex CLI and app | The CLI and this skill, with network access on: `codex -c 'sandbox_mode="workspace-write"' -c 'sandbox_workspace_write.network_access=true'` |
| Codex cloud | The connector, if the environment allows it; otherwise install `infra` in the setup script and give it `INFRA_TOKEN` as a secret |
| ChatGPT | The connector, added by a workspace admin on plans that allow custom servers |

## Other agents with a terminal

OpenCode, Aider, Grok CLI, Antigravity and anything else that can run a shell call
`infra` directly. `infra skill install --project` writes `AGENTS.md`, which those tools read.

## Several accounts on one machine

Each context keeps its own login, so a second account never replaces the first:

```sh
infra auth login --context work   # sign in; the first context you create is the default
infra ctx list                    # who each context is signed in as
infra ctx use work                # change the default
infra ctx pin work                # always use it in this folder; saved in your config, not the repo
infra up --context work           # one command; INFRA_CONTEXT does the same for a shell
```

## Checking the wiring

```sh
infra auth status          # signed in?
infra services list        # can it see anything?
```
