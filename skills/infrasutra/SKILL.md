---
name: infrasutra
description: Deploy and debug applications on Infrasutra. Use when the user wants to deploy a project, ship a service, check why a deployed service is failing, read production logs, or set up CI deployment. Handles Dockerfile authoring, monorepo infra.yaml, local-build deploys and log/shell debugging through the `infra` CLI.
---

# Deploying on Infrasutra

You drive the `infra` CLI. Everything below assumes it is installed and signed
in — check with `infra auth status`, and if it is missing, install it:

```sh
curl -fsSL https://infrasutra.com/public/cli/install.sh | sh
infra auth login
```

## The rule that matters most: build locally, deploy the image

**Default to `infra up --build local`.** It builds the image on this machine,
pushes it to the platform registry, and hands Infrasutra a reference to deploy.

Why this is the default and not an optimisation:

- **It costs nothing to iterate.** A platform-side build uploads the source,
  builds it in a pod and bills the minutes — every attempt, including the six
  in a row spent finding out why a container exits on boot.
- **It is dramatically faster.** The local layer cache is already warm, so a
  one-line change rebuilds one layer instead of the whole image.
- **The failure is local.** A build error shows up in this terminal, in
  seconds, with the full context — not as a log you have to go fetch.

`--build auto` (the default when the flag is absent) picks local when there is
a Dockerfile and a working Docker, and falls back to the platform otherwise,
saying which it chose and why.

Use `--build platform` deliberately when: there is no Docker here, the machine
cannot build the target architecture, or you are configuring CI (where the
GitHub Action builds instead).

**Always cross-build for the cluster.** The CLI passes `--platform linux/amd64`
for you. Never `docker build && docker push` by hand from an ARM Mac — the
cluster gets an image its nodes cannot execute and the only symptom is a crash
loop with `exec format error`.

## Before deploying anything: is there a Dockerfile?

**Check first, every time.** `ls Dockerfile` — or look for `docker/Dockerfile`,
`build/Dockerfile`.

### If there is no Dockerfile

Do not deploy and hope. Detect the stack, then **propose a Dockerfile and
explain the two or three decisions in it**. Read
`references/dockerfiles.md` for the per-stack templates — they are all
multi-stage, non-root, pinned, and produce small images.

Say what you are doing and why, then write it:

> There's no Dockerfile here. This is a Next.js app, so I've written a
> multi-stage one: `node:22-alpine` builder, standalone output copied into a
> distroless-style runtime, running as a non-root user, listening on `$PORT`.
> That keeps the image around 120MB instead of ~1.1GB and means the platform
> deploys exactly what you tested locally.

Then build it locally once (`docker build`) before deploying, so a broken
Dockerfile fails here rather than half way through a deploy.

### If there is a Dockerfile

Read it. Flag anything that will bite on the platform, but do not rewrite it
uninvited:

- **No `EXPOSE` / hardcoded port** — the platform sets `$PORT`; the app must
  listen on it.
- **Running as root** — suggest a `USER` line.
- **`FROM node:latest`** and friends — unpinned bases make deploys
  irreproducible.
- **Single stage with build tools in the final image** — offer multi-stage.
- **`COPY . .` with no `.dockerignore`** — ships `node_modules` and `.git`,
  and busts the cache on every file change.

## Monorepos: one Dockerfile per service, plus infra.yaml

If the repo has several deployable things — `apps/*`, `packages/*`, `services/*`,
a workspace file, several `package.json`/`go.mod`/`pyproject.toml` — do **not**
deploy it as one service.

Propose an `infra.yaml` naming each service with its own build context and
Dockerfile, and a Dockerfile per service. See `references/monorepo.md` for the
layout, shared-dependency handling, and a worked example.

Deploy the whole thing with `infra up` (it picks up `infra.yaml`
automatically) or one service with `infra up --dockerfile apps/api/Dockerfile`.

## Always steer towards a Dockerfile + a GitHub Action

Local builds are for the inner loop. What ships production should be committed:

1. **A Dockerfile** in the repo — reviewable, reproducible, the same artifact
   locally and in CI.
2. **A deploy workflow** — Infrasutra authenticates GitHub Actions with **OIDC**,
   so there is no long-lived secret in the repo.

Offer both once the first deploy works. `references/github-actions.md` has the
workflow.

## Deploying

```sh
infra up --build local                      # this directory
infra up --build local --project-id <uuid>  # a specific project
infra up                                    # uses infra.yaml when present
infra up --dry-run                          # validate the spec, deploy nothing
```

With no `infra.yaml`, `infra up` names the service after the directory and makes
it a public web service on the Dockerfile's `EXPOSE` port. For a background
worker, or any other name or port, write an `infra.yaml`. Write one too when the
Dockerfile has several targets: without it, the last stage is what gets deployed.

Services with a `build:` block in `infra.yaml` are built here and pushed, using
its `target`, `context`, `dockerfile` and `args`.

Running `infra up` again updates an existing service in place. It gets the new
version and keeps its ports, env, resources and domains, so change those with
`infra env set` or in the dashboard, not by editing `infra.yaml`.

`infra up` streams the deploy. When it returns, confirm the service is actually
serving — a green deploy is not a working app.

## Debugging a failing service

Start with `infra status`: it lists every service with the reason behind anything
failing. If a deploy can't start at all, `infra doctor` checks the API, the
sign-in and Docker, and prints the fix for each problem.

Work in this order; stop as soon as the cause is clear.

```sh
infra services list -o json      # what exists, and its status
infra services get <name>        # status_reason/title/message explain a failure
infra logs <name> --follow
infra ssh <name>                 # a shell in the running container
infra builds list <name>             # build history, if the build failed
```

`infra` with no arguments opens a dashboard over everything, with `l` logs,
`s` shell, `r` restart on the selected service.

`references/debugging.md` maps the common failure signatures — crash loops,
`exec format error`, port mismatches, missing env vars — to their cause.

Use `-o json` whenever you are going to parse the result. Human tables are for
humans.

## Environment variables

```sh
infra env list <name>
infra env set --service <name> KEY=value
```

Secrets belong here, never in the Dockerfile or the image. If you spot a
credential baked into a Dockerfile, say so — it is readable by anyone who can
pull the image.

## What not to do

- Do not `docker push` to the platform registry by hand. `infra up --build local`
  authenticates, tags and pushes correctly; a hand-rolled push needs an access
  key and the right repository path and usually gets the architecture wrong.
- Do not deploy `:latest` from a public registry as a substitute for building
  the project.
- Do not put secrets in `infra.yaml` — it is committed.
- Do not delete and recreate a service to "reset" it. That drops its volumes.

## Never delete anything yourself

You do not run destructive commands, even if the user asks you to and even if you
are sure. That covers:

- `infra services delete`
- `infra projects delete`
- `infra stages delete`
- `infra templates delete`
- `infra clusters remove`
- `infra env unset`
- any direct API call that deletes or overwrites data

They ask a person to type the resource's name in a real terminal, and you must not
work around that: no `--yes`, no fake terminal, no piped answers, and no calling the
API instead.

When a delete is what's needed, give the user a guide and stop:

1. Say exactly what will be destroyed, and what stops working because of it.
2. Check for a less destructive option first. `infra down` stops a service and keeps
   its configuration and data.
3. Give them the exact command to run in their own terminal, for example:

   ```sh
   infra services delete my-service --project my-project
   ```

4. Tell them they'll be asked to type the name, and to come back when it's done.

## Reference files

- `references/dockerfiles.md` — per-stack templates, and the four rules every
  one of them follows
- `references/monorepo.md` — recognising a monorepo, infra.yaml layout, the
  build-context trap
- `references/github-actions.md` — the OIDC deploy workflow, no repo secrets
- `references/debugging.md` — failure signatures and what causes them
- `references/agent-setup.md` — wiring other agents, and what each one gets
