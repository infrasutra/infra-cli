# Monorepos on Infrasutra

A monorepo is several services, not one. Deploy it as one and you get a single
container running one app, with the others dead weight in the image.

## Recognising one

Any of these means "propose an infra.yaml":

- `apps/`, `services/`, `packages/` with more than one deployable inside
- a workspace file: `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, `lerna.json`,
  a `workspaces` key in the root `package.json`, a Go `go.work`, a Cargo
  `[workspace]`
- several `package.json` / `go.mod` / `pyproject.toml` / `Cargo.toml` below the
  root, each with its own entry point
- a `docker-compose.yml` naming several services with `build:` blocks — it is
  already telling you the split

One deployable with a `packages/` folder of libraries is **not** a monorepo for
this purpose. Only count things that actually run.

## The shape

One `infra.yaml` at the repo root, one Dockerfile per service, each with its
own build context:

```yaml
version: "1"
name: acme

services:
  - name: api
    type: web
    build:
      context: .                      # repo root: needs the shared packages
      dockerfile: apps/api/Dockerfile
    ports:
      - port: 8080
    exposure:
      public: true
    cpu: 500
    memory: 512
    pre_deploy: ["npm", "run", "migrate"]

  - name: web
    type: web
    build:
      context: .
      dockerfile: apps/web/Dockerfile
    ports:
      - port: 3000
    exposure:
      public: true
    cpu: 250
    memory: 512

  - name: worker
    type: worker                       # no ports, no exposure
    build:
      context: .
      dockerfile: apps/worker/Dockerfile
    cpu: 250
    memory: 256
```

Deploy everything with `infra up`, which picks up `infra.yaml` automatically.
Validate first with `infra up --dry-run`.

## Build context: root, almost always

If services share packages — and in a monorepo they do — the context must be
the **repo root**, with the Dockerfile path pointing into the service:

```yaml
build:
  context: .
  dockerfile: apps/api/Dockerfile
```

A context of `apps/api` cannot `COPY ../../packages/shared`; Docker refuses
paths outside the context, and the error (`forbidden path outside the build
context`) does not say why.

The cost is that the whole repo is the context, so a `.dockerignore` at the
root matters more, not less.

## A Dockerfile in a workspace

The point is to install workspace-aware and then prune to one app. pnpm:

```dockerfile
FROM node:22-alpine AS build
RUN corepack enable
WORKDIR /repo
# Lockfiles and manifests first, so a source change does not reinstall.
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/package.json apps/api/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile --filter ./apps/api...

COPY . .
RUN pnpm --filter ./apps/api build
# deploy resolves the workspace links into a real node_modules.
RUN pnpm --filter ./apps/api deploy --prod /out

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=build /out .
USER node
EXPOSE 8080
CMD ["node", "dist/main.js"]
```

Turborepo: `turbo prune --scope=api --docker` produces exactly this layout.

Go workspace: `COPY go.work go.work.sum ./` plus each module's `go.mod`, then
`go build ./cmd/api`.

## Per-service resources

Size each one. A worker that processes a queue does not need the API's memory,
and the platform bills what you reserve:

```yaml
cpu: 250        # millicores
memory: 512     # MB
replicas: 2
```

## Local builds with a monorepo

`infra up` reads `infra.yaml` and builds each service against its own context
and Dockerfile. For one service in isolation:

```sh
infra up --build local --dockerfile apps/api/Dockerfile
```

## Deploy order

`pre_deploy` gates a rollout, so migrations belong on the service that owns the
schema, not in a separate one-shot service. There is no cross-service ordering
in the spec: if the worker must not start before the API's migration lands, run
the migration as the API's `pre_deploy` and let the worker retry its
connection, which it should do anyway.
