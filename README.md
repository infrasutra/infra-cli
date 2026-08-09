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

### One command

```sh
infra skill install
```

It detects which agents this project is set up for and writes the right thing
for each: the skill tree for Claude Code, `AGENTS.md` for Codex and everything
else that reads it, a rules file for Cursor, Cline or Windsurf, plus the MCP
entry where the host uses one. Nothing configured yet? It writes `AGENTS.md`,
which nearly every agent reads.

```sh
infra skill list              # what this project uses
infra skill install --agent claude
infra skill install --global  # for every project on this machine
infra skill print             # the guidance itself, to paste elsewhere
```

The guidance ships inside the binary, so it always matches the CLI you have.
Re-running updates in place and leaves anything you wrote around it alone.

### By hand

The skill is plain markdown, published in this repo under
[`skills/infrasutra/`](skills/infrasutra). For an MCP host, the server is the
CLI itself:

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
