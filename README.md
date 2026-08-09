# infra

The command-line client for [Infrasutra](https://infrasutra.com). Deploy, tail
logs, run a command in a container and open a shell — from your terminal.

This repository publishes **releases only**. The CLI is built from the main
Infrasutra repository; nothing here but the binaries and the package manifests
that point at them.

## Install

### macOS · Linux

```sh
curl -fsSL https://infrasutra.com/public/cli/install.sh | sh
```

No curl? `wget -qO- https://infrasutra.com/public/cli/install.sh | sh`

The script picks the right build, verifies it against `checksums.txt`, and
installs to `/usr/local/bin`. Set `INFRA_INSTALL_DIR` to install somewhere you
own, or `INFRA_VERSION=0.1.0` to pin a release.

### Homebrew

```sh
brew tap infrasutra/infra https://github.com/infrasutra/infra-cli
brew trust infrasutra/infra
brew install infrasutra/infra/infra
```

Both extra steps are load-bearing — see [Homebrew notes](#homebrew-notes).

### Windows

```powershell
irm https://infrasutra.com/public/cli/install.ps1 | iex
```

Or with [Scoop](https://scoop.sh):

```powershell
scoop bucket add infrasutra https://github.com/infrasutra/infra-cli
scoop install infra
```

### Arch Linux

```sh
yay -S infra-bin
```

Or build the published package yourself — the binary is static, so nothing
compiles:

```sh
curl -fsSLO https://raw.githubusercontent.com/infrasutra/infra-cli/main/packaging/aur/PKGBUILD
makepkg -si
```

### Alpine and other musl distros

The Linux builds are static, so the install script works as-is — no glibc, no
`apk add` first.

### Manual

Download an archive from [Releases](https://github.com/infrasutra/infra-cli/releases),
verify it against `checksums.txt`, unpack, and put `infra` on your `PATH`:

```sh
tar -xzf infra_0.1.0_linux_amd64.tar.gz
sudo install -m755 infra /usr/local/bin/infra
```

## Homebrew notes

Two things about the Homebrew install surprise people. Neither is a bug in the
tap; both are how Homebrew works.

### The formula name must be qualified

`infra` is already taken in Homebrew's core taps by an unrelated project — the
Infra Kubernetes desktop client. An unqualified install gets that one:

```
$ brew info infra
Warning: Treating infra as a cask. For the formula, use infrasutra/infra/infra
==> infra (infra): 0.47.0 (auto_updates)
Kubernetes desktop client
https://infra.app/
```

Always install as `infrasutra/infra/infra`. The command it puts on your PATH is
still just `infra`.

### The tap has to be trusted

Homebrew 6 will not load a formula from a third-party tap until you say so:

```
Error: Refusing to load formula infrasutra/infra/infra from untrusted tap infrasutra/infra.
Run `brew trust --formula infrasutra/infra/infra` or `brew trust infrasutra/infra` to trust it.
```

`brew trust infrasutra/infra` is a one-time consent, same as tapping. Run it
once and future upgrades work normally.

### Upgrading

```sh
brew update && brew upgrade infrasutra/infra/infra
```

### Uninstalling

```sh
brew uninstall infrasutra/infra/infra
brew untap infrasutra/infra
```

## First run

```sh
infra auth login     # opens your browser
infra                # the dashboard: every project and service
infra ssh            # pick a service and get a shell in it
infra logs --service web --follow
```

## Use it with an AI coding agent

This repo ships a **skill** — the deployment rules an agent should follow, in the
form Claude Code and friends read directly. It covers building locally instead
of paying for platform build minutes, writing a Dockerfile when there is none,
splitting a monorepo into an `infra.yaml`, and reading the platform's own
explanation of a failure instead of guessing.

### Claude Code

```sh
# for one project
mkdir -p .claude/skills
curl -fsSL https://github.com/infrasutra/infra-cli/archive/refs/heads/main.tar.gz |
  tar -xz --strip-components=2 -C .claude/skills infra-cli-main/skills/infrasutra

# or for every project
mkdir -p ~/.claude/skills
curl -fsSL https://github.com/infrasutra/infra-cli/archive/refs/heads/main.tar.gz |
  tar -xz --strip-components=2 -C ~/.claude/skills infra-cli-main/skills/infrasutra
```

### Any agent that speaks MCP

The CLI is its own MCP server — no extra install:

```json
{
  "mcpServers": {
    "infrasutra": { "command": "infra", "args": ["mcp"] }
  }
}
```

Works in Claude Code (`.mcp.json`), Cline, Cursor, Windsurf, Continue and Codex.
The tools are read-only (services, logs, builds, env keys) plus deployment
guidance; anything destructive stays in the CLI, where a person reads the
command before it runs.

### Agents with a terminal and no MCP

OpenCode, Aider, Grok CLI, Antigravity and anything else that can run a shell
need no setup — they call `infra` directly. Put the short version in whatever
rules file the tool reads:

```markdown
Deploy: `infra up --build local`. Never `docker push` by hand.
No Dockerfile? Write a multi-stage, non-root one listening on $PORT first.
Monorepo? One infra.yaml, one Dockerfile per service, build context = repo root.
Debug: `infra services get <name>` explains a failure in the platform's own words.
```

Full setup notes, including which hosts cannot use either surface, are in
[`skills/infrasutra/references/agent-setup.md`](skills/infrasutra/references/agent-setup.md).

## Verifying a download

Every release ships `checksums.txt`:

```sh
sha256sum -c checksums.txt --ignore-missing
```

Releases built by CI also carry a [build attestation](https://docs.github.com/actions/security-guides/using-artifact-attestations)
recording which workflow run produced them. The release workflow lives in the
private source repository, so verifying an attestation needs access to it:

```sh
gh attestation verify infra_0.1.0_linux_amd64.tar.gz --repo infrasutra/infrasutra-backend
```

The install scripts verify `checksums.txt` for you on every install.
