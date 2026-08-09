# Dockerfiles for Infrasutra

Templates to propose when a project has none. Every one is multi-stage, runs as
a non-root user, pins its base, and listens on `$PORT`.

## The four rules

1. **Multi-stage.** Build tools never reach the final image. A Node app goes
   from ~1.1GB to ~120MB; a Go app to ~15MB.
2. **Pin the base.** `node:22-alpine`, not `node:latest`. An unpinned base
   means today's deploy and next month's are different software.
3. **Non-root.** A `USER` line. If the container is compromised, the blast
   radius should not include root in the namespace.
4. **Listen on `$PORT`.** The platform injects it. An app hardcoded to 3000
   deploys green and then times out every health check.

Always write a `.dockerignore` alongside — without one, `COPY . .` ships
`node_modules`, `.git` and local env files, and invalidates the layer cache on
every unrelated file change.

## Node / Next.js

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
# Next's standalone output carries only the server and the modules it actually
# imports, which is most of the size win.
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static
COPY --from=build /app/public ./public
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Needs `output: "standalone"` in `next.config.js`. Without it, copy
`.next`, `node_modules`, `package.json` and run `npm start` instead.

Plain Node/Express: same shape, `COPY --from=deps` the production modules and
`CMD ["node", "src/index.js"]`.

## Go

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Static, so the runtime image needs no libc at all.
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

## Python

```dockerfile
FROM python:3.13-slim AS build
WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 PIP_NO_CACHE_DIR=1
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

FROM python:3.13-slim AS runtime
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY --from=build /install /usr/local
COPY . .
RUN useradd -u 10001 -m app
USER app
EXPOSE 8000
CMD ["sh", "-c", "gunicorn -b 0.0.0.0:${PORT:-8000} app:app"]
```

Django: `CMD ["sh","-c","gunicorn -b 0.0.0.0:${PORT:-8000} project.wsgi"]`, and
run migrations as a pre-deploy hook rather than at container start — every
replica would race.

With uv or Poetry, install from the lockfile in the build stage and copy the
resulting environment.

## PHP / Laravel

```dockerfile
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --prefer-dist

FROM php:8.3-fpm-alpine AS runtime
WORKDIR /var/www
RUN docker-php-ext-install pdo_mysql opcache
COPY --from=vendor /app/vendor ./vendor
COPY . .
RUN chown -R www-data:www-data storage bootstrap/cache
USER www-data
EXPOSE 9000
CMD ["php-fpm"]
```

Serving HTTP directly is simpler on the platform: use `php artisan serve
--host=0.0.0.0 --port=${PORT}` for small apps, or add nginx in the same image
for real traffic.

## Rust

```dockerfile
FROM rust:1-alpine AS build
RUN apk add --no-cache musl-dev
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
# Prime the dependency layer so a source-only change does not rebuild the world.
RUN mkdir src && echo "fn main() {}" > src/main.rs && cargo build --release && rm -rf src
COPY . .
RUN touch src/main.rs && cargo build --release

FROM alpine:3.20
RUN adduser -D -u 10001 app
COPY --from=build /src/target/release/server /usr/local/bin/server
USER app
EXPOSE 8080
CMD ["server"]
```

## Static sites (Vite, CRA, Astro, Hugo)

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
# The platform sets $PORT; nginx must be told, or it sits on 80 and the health
# check never passes.
RUN sed -i 's/listen  *80;/listen 8080;/' /etc/nginx/conf.d/default.conf
EXPOSE 8080
```

Client-side env vars (`VITE_*`, `NEXT_PUBLIC_*`) are baked at **build** time.
Set them as build args, or enable env injection at build on the service — a
runtime `infra env set` will not change an already-built bundle.

## .dockerignore

```
.git
node_modules
.next
dist
target
__pycache__
*.pyc
.env
.env.*
.venv
vendor
coverage
.DS_Store
```

## Checking one before deploying

```sh
docker build --platform linux/amd64 -t check .
docker run --rm -e PORT=8080 -p 8080:8080 check
curl -fsS localhost:8080 >/dev/null && echo "serves"
```

If it does not serve here it will not serve on the platform, and finding that
out locally costs seconds instead of a deploy cycle.
