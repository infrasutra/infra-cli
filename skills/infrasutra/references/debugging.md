# Debugging a service on Infrasutra

The API already worked out why a service is unhealthy. Ask it before guessing:

```sh
infra services get <name> -o json | jq '{status, status_reason, status_title, status_message, status_hint}'
```

`status_reason` is a stable code; the other three are the platform's own words
for it and are usually the whole answer.

## Order of investigation

```sh
infra services list -o json          # what exists and what state it is in
infra services get <name>            # why, in the platform's words
infra logs --service <name> --follow # what the app itself said
infra builds list --service <name>   # if it never got as far as running
infra ssh <name>                     # a shell inside the running container
```

Stop at the first step that explains it.

## Signatures

### `exec format error` in the logs, or a crash loop right after deploy

The image was built for the wrong architecture — almost always an arm64 Mac
pushing its native build. The cluster runs amd64.

`infra up --build local` passes `--platform linux/amd64` for you. If the image
came from a hand-rolled `docker build`, rebuild it:

```sh
docker buildx build --platform linux/amd64 -t <image> --push .
```

### Deploy goes green, then the service never becomes ready

The app is not listening where the platform is looking.

- The platform injects `$PORT`. An app hardcoded to 3000 while the service
  declares 8080 never answers a health check.
- Listening on `127.0.0.1` is invisible from outside the container. Bind
  `0.0.0.0`.

```sh
infra ssh <name>
# inside:
env | grep PORT
netstat -ltn 2>/dev/null || ss -ltn
```

### Container exits immediately, no logs

Usually the entrypoint. Run the same image locally — it fails the same way, in
a second, with the error in front of you:

```sh
docker run --rm -e PORT=8080 <image>
```

A `CMD` in shell form that references an unset variable, or an entrypoint script
without `+x`, both produce exactly this.

### `no such file or directory` on a binary that exists

A statically-linked expectation against a musl/glibc mismatch: a binary built
on Debian will not run on Alpine. Build in the same base you run in, or build
static (`CGO_ENABLED=0` for Go).

### Works locally, missing config in production

Environment variables are not carried from your machine:

```sh
infra env list --service <name>
infra env set --service <name> DATABASE_URL=...
```

Client-side variables (`VITE_*`, `NEXT_PUBLIC_*`) are baked at **build** time —
setting them at runtime changes nothing in an already-built bundle. Rebuild
with them as build args, or enable build-time env injection on the service.

### Build fails on the platform but works locally

Usually context or cache:

- No `.dockerignore`, so `node_modules` from your machine (native modules built
  for macOS) ships into a Linux image.
- A file only in your working tree — untracked, or ignored — that the build
  needs.

Reproduce with a clean context:

```sh
git stash -u && docker build --no-cache --platform linux/amd64 . ; git stash pop
```

### Out of memory

The pod is killed with no application error. Check the reserved memory against
what the process actually needs — a JVM or a Next build wants far more than the
128MB default:

```sh
infra services get <name> -o json | jq '{cpu_millicores, memory_mb}'
```

Raise it in `infra.yaml` (`memory: 1024`) and redeploy.

## Reading logs well

```sh
infra logs --service <name> --follow
infra logs --service <name> --since 1h
infra logs --service <name> --filter error
infra logs --service <name> --tail 200
infra logs --service api,worker --follow     # several at once
```

Logs survive a crash — a container that died seconds ago still has its output.
That is usually where the answer is.

## Build logs

```sh
infra builds list --service <name>
infra builds logs <build-id>
```

Build logs are kept for 24 hours. Past that the record of the build remains but
its output is gone.

## When you have changed something

Redeploy rather than restart: a restart runs the same image again, which fixes
nothing if the image is the problem.

```sh
infra up --build local     # new image
infra services restart <name>   # same image, fresh container
```
