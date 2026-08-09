# Deploying from GitHub Actions

Local builds are for the inner loop. What ships production should be a workflow
in the repo — reviewable, reproducible, and the same artifact every time.

Infrasutra authenticates a GitHub Actions run with **OIDC**, so there is **no
long-lived secret in the repo**. GitHub mints a short-lived assertion naming the
repository, ref and commit; Infrasutra checks it against the service the
repository is linked to.

## One-time setup

1. Deploy the service once so it exists: `infra up --build local`.
2. Link the repository to it (the Deployments tab, or `infra` → the service →
   connect repository). The link is the entitlement: the workflow can deploy
   the service its repository owns, and nothing else.

## The workflow

`.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  contents: read
  # Without this the OIDC token is never minted and every step below 401s.
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      INFRA_API: https://infrasutra.com
      SERVICE: api          # the service name in Infrasutra
    steps:
      - uses: actions/checkout@v4

      - name: Get an OIDC token for Infrasutra
        id: oidc
        uses: actions/github-script@v7
        with:
          script: |
            const token = await core.getIDToken('infrasutra')
            core.setSecret(token)
            core.setOutput('token', token)

      # Where to push, and a registry credential scoped to that repository
      # alone, valid for minutes.
      - name: Resolve the deploy target
        id: target
        run: |
          set -euo pipefail
          response=$(curl -fsS -X POST "$INFRA_API/api/v1/ci/deploy-target" \
            -H "Authorization: GitHub-OIDC ${{ steps.oidc.outputs.token }}" \
            -H "Content-Type: application/json" \
            -d "{\"repository\":\"$GITHUB_REPOSITORY\",\"ref\":\"$GITHUB_REF\",\"commit\":\"$GITHUB_SHA\",\"service\":\"$SERVICE\"}")
          echo "registry=$(jq -r .registry <<<"$response")" >> "$GITHUB_OUTPUT"
          echo "image=$(jq -r .image <<<"$response")" >> "$GITHUB_OUTPUT"
          echo "::add-mask::$(jq -r .registry_token <<<"$response")"
          echo "token=$(jq -r .registry_token <<<"$response")" >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@v3

      - name: Log in to the Infrasutra registry
        run: echo "${{ steps.target.outputs.token }}" |
          docker login "${{ steps.target.outputs.registry }}" -u infra --password-stdin

      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          # The cluster runs amd64. An arm64 image deploys green and then
          # crash-loops with "exec format error".
          platforms: linux/amd64
          push: true
          tags: ${{ steps.target.outputs.image }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Release it
        run: |
          curl -fsS -X POST "$INFRA_API/api/v1/ci/deploy" \
            -H "Authorization: GitHub-OIDC ${{ steps.oidc.outputs.token }}" \
            -H "Content-Type: application/json" \
            -d "{\"service\":\"$SERVICE\",\"image_digest\":\"${{ steps.build.outputs.digest }}\",\"commit_sha\":\"$GITHUB_SHA\"}"
```

## Why the digest, not the tag

`/ci/deploy` requires `image_digest` and refuses a tag. Deploying `:v1` means
"whatever that tag points at by the time the deploy runs" — under any
concurrency that is a different image from the one this job just built.
`docker/build-push-action` hands you the digest in `steps.build.outputs.digest`.

## Monorepos

One job per service, each with its own `SERVICE`, context and Dockerfile. Add a
paths filter so a change to `apps/web` does not redeploy the API:

```yaml
on:
  push:
    branches: [main]
    paths: ["apps/api/**", "packages/shared/**", ".github/workflows/deploy-api.yml"]
```

## If you cannot use OIDC

Self-hosted runners outside GitHub, or another CI system: mint an access key
scoped to the service and pass it as `Authorization: Bearer infra_...`
everywhere the workflow above sends `GitHub-OIDC`. It is a standing credential,
so scope it to the one service and rotate it.

## Checking it

Trigger it, then confirm from a terminal — a green workflow is not a working
app:

```sh
infra services get api
infra logs --service api --follow
```
