# infra

The command-line client for [Infrasutra](https://infrasutra.com). Deploy, tail
logs, run a command in a container and open a shell — from your terminal.

This repository publishes **releases only**. The CLI is built from the main
Infrasutra repository; nothing here but the binaries and the package manifests
that point at them.

> Infrasutra is not publicly launched yet, so `https://infrasutra.com` is not
> serving the install script quite yet. Until it does, install from a release
> archive below (or from a deployment you already have access to — every
> Infrasutra deployment serves the same script at `/public/cli/install.sh`).

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
brew install infra
```

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

`packaging/aur/PKGBUILD` builds from the published release — the binary is
static, so nothing compiles:

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

## First run

```sh
infra auth login     # opens your browser
infra                # the dashboard: every project and service
infra ssh            # pick a service and get a shell in it
infra logs --service web --follow
```

## Verifying a download

Every release ships `checksums.txt`:

```sh
sha256sum -c checksums.txt --ignore-missing
```

## What's in this repo

| Path | For |
|------|-----|
| `Formula/infra.rb` | Homebrew tap |
| `bucket/infra.json` | Scoop bucket |
| `packaging/aur/PKGBUILD` | Arch Linux |
| `packaging/winget/` | winget manifests, ready to submit to `microsoft/winget-pkgs` |

All of them are generated from a published release and carry its real SHA256.
