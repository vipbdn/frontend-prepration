# 🐳 The Complete Docker Guide — Daily Use, Revision & Interview Prep

> A practical guide you can **follow while working**, **revise quickly**, and **prepare for interviews** with. Beginners can read it top to bottom; experienced developers can jump to the cheat sheet, recipes, troubleshooting or interview questions.
>
> Written for modern Docker (Engine 25+, BuildKit, Compose v2 `docker compose`). Examples focus on web/full-stack apps (Node, React, Next.js, databases).

---

## 📖 How to Use This Guide

| If you want to… | Go to |
| --- | --- |
| Learn Docker from scratch | Sections 1 → 10 in order |
| Look up a command while working | [§22 Cheat Sheet](#22-cheat-sheet-) |
| Write a production Dockerfile | [§6 Dockerfile](#6-dockerfile-) and [§7 Dockerfile Best Practices](#7-dockerfile-best-practices-) |
| Run a multi-container app | [§11 Docker Compose](#11-docker-compose-) |
| Fix a broken container right now | [§16 Troubleshooting](#16-debugging--troubleshooting-) and [§21 Recipes](#21-real-world-recipes-) |
| Prepare for an interview | [§18 Internals](#18-how-docker-works-internally-), [§23 Interview Questions](#23-interview-questions--answers-) |

**Legend:** ⭐ used daily · ⚠️ destructive / risky · 💡 tip

---

## 📑 Table of Contents

1. [What is Docker?](#1-what-is-docker)
2. [Core Concepts](#2-core-concepts-)
3. [Installation & Setup](#3-installation--setup)
4. [Working with Images](#4-working-with-images-)
5. [Working with Containers](#5-working-with-containers-)
6. [Dockerfile](#6-dockerfile-)
7. [Dockerfile Best Practices](#7-dockerfile-best-practices-)
8. [Building Images](#8-building-images)
9. [Volumes & Storage](#9-volumes--storage-)
10. [Networking](#10-networking-)
11. [Docker Compose](#11-docker-compose-)
12. [Registries: Push & Pull](#12-registries-push--pull)
13. [Resources, Restart Policies, Health & Logs](#13-resources-restart-policies-health--logs)
14. [Security](#14-security-)
15. [Environment Variables & Secrets](#15-environment-variables--secrets)
16. [Debugging & Troubleshooting](#16-debugging--troubleshooting-)
17. [Cleanup](#17-cleanup)
18. [How Docker Works Internally](#18-how-docker-works-internally-)
19. [Docker in CI/CD](#19-docker-in-cicd)
20. [Orchestration: Swarm & Kubernetes Basics](#20-orchestration-swarm--kubernetes-basics)
21. [Real-World Recipes](#21-real-world-recipes-)
22. [Cheat Sheet](#22-cheat-sheet-)
23. [Interview Questions & Answers](#23-interview-questions--answers-)
24. [Practice Exercises](#24-practice-exercises)

---

# 1. What is Docker?

**Docker** is a platform for **building, shipping and running applications in containers**. A container packages your app **together with everything it needs** — runtime, libraries, system tools, config — so it runs the same on your laptop, a teammate's machine, CI, and production.

### The problem it solves

```text
"It works on my machine!" 🤷
  Dev laptop:  Node 22, OpenSSL 3, macOS
  CI server:   Node 18, different libs
  Production:  Ubuntu with missing packages
```

With Docker, the **image** is the unit you ship — the environment travels with the code.

### Benefits

- **Consistency** across dev, test, prod
- **Isolation** — apps don't conflict over versions or ports
- **Fast onboarding** — `docker compose up` instead of a 20-step setup doc
- **Lightweight & fast** — start in seconds, share the host kernel
- **Portability** — any machine/cloud with a container runtime
- **Microservices & scaling** — each service in its own container, scaled independently
- **Reproducible CI/CD** — build once, deploy the same image everywhere

## 1.1 Containers vs Virtual Machines ⭐

```text
        Virtual Machines                         Containers
┌──────┐ ┌──────┐ ┌──────┐            ┌──────┐ ┌──────┐ ┌──────┐
│ App A│ │ App B│ │ App C│            │ App A│ │ App B│ │ App C│
│ Libs │ │ Libs │ │ Libs │            │ Libs │ │ Libs │ │ Libs │
│Guest │ │Guest │ │Guest │            └──────┘ └──────┘ └──────┘
│  OS  │ │  OS  │ │  OS  │            ┌──────────────────────────┐
└──────┘ └──────┘ └──────┘            │  Container runtime        │
┌──────────────────────────┐          │  (Docker Engine)          │
│       Hypervisor          │          ├──────────────────────────┤
├──────────────────────────┤          │  Host OS (shared kernel)  │
│        Host OS            │          ├──────────────────────────┤
├──────────────────────────┤          │  Hardware                 │
│        Hardware           │          └──────────────────────────┘
└──────────────────────────┘
```

| | Virtual Machine | Container |
| --- | --- | --- |
| Virtualizes | Hardware (full guest OS per VM) | OS level (processes share host kernel) |
| Size | GBs | MBs |
| Startup | Minutes | Seconds / milliseconds |
| Isolation | Strong (separate kernels) | Process-level (namespaces, cgroups) — weaker by default |
| Density | Few per host | Many per host |
| OS flexibility | Any OS (Windows VM on Linux host) | Must match host kernel type (Linux containers need a Linux kernel) |

> On macOS/Windows, Docker Desktop runs a lightweight **Linux VM** under the hood to provide a Linux kernel for Linux containers.

They're complementary: in the cloud, containers usually run **inside** VMs.

## 1.2 Docker Architecture

```text
┌────────────┐   REST API    ┌──────────────────────────────────────────────┐      ┌─────────────┐
│ Docker CLI │ ────────────▶ │ Docker daemon (dockerd)                       │ ◀──▶ │  Registry    │
│ docker run │  (unix socket)│  ├─ images, networks, volumes, builds (BuildKit)│ pull │ (Docker Hub, │
└────────────┘               │  └─ containerd → runc → container processes    │ push │  GHCR, ECR)  │
                             └──────────────────────────────────────────────┘      └─────────────┘
```

| Component | Role |
| --- | --- |
| **Docker CLI** (`docker`) | Client you type commands into; talks to the daemon over a REST API |
| **Docker daemon** (`dockerd`) | Manages images, containers, networks, volumes, builds |
| **containerd** | Industry-standard container runtime managing container lifecycle (also used by Kubernetes) |
| **runc** | Low-level OCI runtime that actually creates the container (namespaces, cgroups) |
| **BuildKit** | Modern build engine (parallel, cached, secrets) — default builder |
| **Registry** | Stores and distributes images |
| **OCI** | Open Container Initiative — standards for image format & runtimes, so images work with Docker, Podman, containerd, Kubernetes |

---

# 2. Core Concepts ⭐

| Concept | What it is | Analogy |
| --- | --- | --- |
| **Image** | Read-only template: filesystem layers + metadata (default command, env, ports) | A class / a recipe |
| **Container** | A running (or stopped) **instance** of an image with its own writable layer | An object / a cooked dish |
| **Dockerfile** | Text file with instructions to build an image | The recipe text |
| **Layer** | Each filesystem-changing instruction creates a cached, reusable layer | Git commit-like diffs |
| **Registry / Repository** | Server storing images (Docker Hub); a repository holds versions (tags) of one image | GitHub / a repo |
| **Tag** | Human-readable version label: `node:22-alpine` | Git tag |
| **Digest** | Immutable content hash: `node@sha256:ab12…` | Commit SHA |
| **Volume** | Persistent storage managed by Docker, outliving containers | External hard drive |
| **Network** | Virtual network letting containers talk to each other | A private LAN |
| **Docker Compose** | Tool to define & run multi-container apps from a YAML file | Orchestra conductor for local/dev |

### Image naming

```text
[registry/][namespace/]repository[:tag][@digest]

nginx                                → docker.io/library/nginx:latest
node:22-alpine                       → docker.io/library/node:22-alpine
ghcr.io/myorg/api:1.4.2              → GitHub Container Registry
123456789.dkr.ecr.ap-south-1.amazonaws.com/api:sha-3f2a9c1   → AWS ECR
```

If no tag is given, Docker uses **`latest`** — which just means "the tag called latest", **not** necessarily the newest version.

### Image → Container lifecycle

```text
Dockerfile ──docker build──▶ Image ──docker run──▶ Container (running)
                               │                        │ stop / start / restart
                    docker push│                        ▼
                               ▼                    Container (exited) ──docker rm──▶ gone
                           Registry
```

---

# 3. Installation & Setup

```bash
# macOS / Windows: install Docker Desktop (includes Engine, CLI, Compose, Buildx)
# Linux (Ubuntu): follow docs.docker.com/engine/install, then:
sudo usermod -aG docker $USER     # run docker without sudo (log out/in) — ⚠️ docker group ≈ root access
newgrp docker

docker version            # client + server versions
docker info               # system-wide info: storage driver, containers, images
docker run hello-world    # smoke test
```

Alternatives: **Podman** (daemonless, rootless, Docker-compatible CLI), **Colima**/**OrbStack** (lightweight Desktop alternatives on macOS), **Rancher Desktop**.

### Contexts (switch between Docker hosts)

```bash
docker context ls
docker context create remote --docker "host=ssh://user@server"
docker context use remote      # now commands run against the remote daemon
docker context use default
```

### Getting help

```bash
docker --help
docker run --help
docker <command> --help
```

---

# 4. Working with Images ⭐

```bash
docker pull node:22-alpine              # download an image
docker pull --platform linux/arm64 nginx  # specific architecture
docker images                           # list local images (alias: docker image ls)
docker images --filter dangling=true    # untagged images (<none>)
docker image inspect node:22-alpine     # full metadata (JSON): layers, env, cmd, arch
docker image history node:22-alpine     # layers and the instruction that created each
docker tag myapp:latest myorg/myapp:1.0.0   # add another name/tag (same image ID)
docker push myorg/myapp:1.0.0           # upload to registry (after docker login)
docker rmi myapp:old                    # remove image (alias: docker image rm)
docker image prune                      # remove dangling images
docker image prune -a                   # ⚠️ remove all images not used by any container
docker search postgres                  # search Docker Hub

# Export/import without a registry (air-gapped servers)
docker save -o myapp.tar myapp:1.0.0
docker load -i myapp.tar
```

### Choosing base images 💡

| Variant | Example | Size | Notes |
| --- | --- | --- | --- |
| Full (Debian) | `node:22` | ~1 GB | Many tools; big |
| **Slim** | `node:22-slim` | ~200 MB | Debian minus extras — good default |
| **Alpine** | `node:22-alpine` | ~50 MB | musl libc (some native modules behave differently) |
| **Distroless** | `gcr.io/distroless/nodejs22-debian12` | small | No shell/package manager — great for production security |
| `scratch` | `FROM scratch` | 0 | Empty; for static binaries (Go, Rust) |

Prefer **official** or **verified publisher** images, and pin versions (`node:22.11-alpine`), not `latest`.

---

# 5. Working with Containers ⭐

## 5.1 `docker run` — create + start ⭐

```bash
docker run nginx                                   # foreground; Ctrl+C stops it
docker run -d --name web -p 8080:80 nginx          # ⭐ detached, named, port 8080 (host) → 80 (container)
docker run -it --rm node:22-alpine sh              # ⭐ interactive shell, auto-remove on exit
docker run -d -e NODE_ENV=production -e PORT=3000 myapi
docker run -d --env-file .env myapi
docker run -d -v pgdata:/var/lib/postgresql/data postgres:17   # named volume
docker run -d -v "$(pwd)":/app -w /app node:22 npm run dev       # bind mount current dir
docker run -d --restart unless-stopped myapi
docker run -d --memory 512m --cpus 1.5 myapi
docker run --rm myapp npm test                     # override the default command
docker run -d --network mynet --name api myapi
```

### Common `run` flags ⭐

| Flag | Meaning |
| --- | --- |
| `-d`, `--detach` | Run in background |
| `-it` | Interactive + allocate a TTY (shells) |
| `--name web` | Give a name (otherwise random, e.g. `quirky_turing`) |
| `--rm` | Delete the container when it exits |
| `-p 8080:80` | Publish port: `host:container` (`-p 127.0.0.1:8080:80` = localhost only) |
| `-P` | Publish all `EXPOSE`d ports to random host ports |
| `-e KEY=val` / `--env-file f` | Environment variables |
| `-v src:dest[:ro]` / `--mount` | Volumes and bind mounts |
| `-w /app` | Working directory |
| `--network net` | Attach to a network |
| `--restart policy` | `no` · `on-failure[:N]` · `always` · `unless-stopped` |
| `--memory 512m`, `--cpus 1` | Resource limits |
| `-u 1000:1000` | Run as user/group |
| `--entrypoint sh` | Override ENTRYPOINT |
| `--init` | Run a tiny init (tini) as PID 1 — reaps zombies, forwards signals |
| `--platform linux/amd64` | Run a specific architecture (emulated if needed) |
| `--read-only` | Read-only root filesystem |
| `--health-cmd …` | Define a healthcheck at runtime |

## 5.2 Managing containers ⭐

```bash
docker ps                      # running containers (alias: docker container ls)
docker ps -a                   # all, including stopped
docker ps -q                   # only IDs (useful in scripts)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

docker stop web                # graceful: SIGTERM, then SIGKILL after 10s (-t to change)
docker start web               # start a stopped container (keeps its data/writable layer)
docker restart web
docker kill web                # immediate SIGKILL
docker pause web / docker unpause web
docker rm web                  # remove a stopped container
docker rm -f web               # ⚠️ force stop + remove
docker rename web web-old

docker create --name web nginx # create without starting
```

## 5.3 Inspecting & debugging ⭐

```bash
docker logs web                # stdout/stderr of the container
docker logs -f --tail 100 web  # ⭐ follow, last 100 lines
docker logs --since 10m -t web # last 10 minutes, with timestamps

docker exec -it web sh         # ⭐ shell inside a RUNNING container (bash if available)
docker exec web ls /usr/share/nginx/html
docker exec -u root -it web sh # as root

docker inspect web             # full JSON: config, mounts, network, IP, state
docker inspect -f '{{.State.Status}} {{.NetworkSettings.IPAddress}}' web
docker stats                   # live CPU / memory / network / IO of containers
docker top web                 # processes inside the container
docker port web                # port mappings
docker diff web                # files changed in the container's writable layer (A/C/D)
docker events                  # real-time daemon events

docker cp web:/etc/nginx/nginx.conf ./nginx.conf   # copy out
docker cp ./index.html web:/usr/share/nginx/html/  # copy in

docker attach web              # attach terminal to main process (Ctrl+P, Ctrl+Q to detach) — prefer exec
docker wait web                # block until exit; prints exit code
```

## 5.4 Container lifecycle & states

```text
created ──start──▶ running ──stop/kill/exit──▶ exited ──rm──▶ (removed)
                     │  ▲
               pause │  │ unpause
                     ▼  │
                    paused
                     restarting (with restart policy)
```

### Exit codes to know

| Code | Meaning |
| --- | --- |
| `0` | Success (process finished) |
| `1` | Application error |
| `125` | Docker run itself failed (bad flag) |
| `126` | Command found but not executable (permissions) |
| `127` | Command not found |
| `137` | Killed by SIGKILL (128+9) — often **out of memory (OOM)** or `docker kill` |
| `139` | Segfault (128+11) |
| `143` | Terminated by SIGTERM (128+15) — normal `docker stop` |

## 5.5 Why does my container exit immediately? ⭐

A container lives **as long as its main process (PID 1)**. If the command finishes, the container stops.

```bash
docker run ubuntu            # runs "bash" with no TTY → exits immediately
docker run -it ubuntu        # interactive → stays until you exit
docker run -d nginx          # nginx runs in the foreground → stays up
```

Servers must run in the **foreground** inside containers (e.g. `nginx -g 'daemon off;'`, `node server.js`, not a background service manager).

## 5.6 `docker commit` (rarely)

```bash
docker commit web myorg/web:debug   # snapshot a container into an image
```

Useful for debugging; **not** reproducible — always use Dockerfiles for real images.

---

# 6. Dockerfile ⭐

A **Dockerfile** is a text file of instructions executed top to bottom to build an image.

## 6.1 First example (Node.js API)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine

WORKDIR /app

# Copy dependency manifests first → better layer caching
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Copy the rest of the source
COPY . .

ENV NODE_ENV=production
EXPOSE 3000

USER node
CMD ["node", "src/server.js"]
```

```bash
docker build -t myapi:1.0 .
docker run -d -p 3000:3000 --name api myapi:1.0
```

## 6.2 Instructions Reference ⭐

| Instruction | Purpose | Example |
| --- | --- | --- |
| `FROM` | Base image (must be first, except `ARG`) | `FROM node:22-alpine AS build` |
| `WORKDIR` | Set (and create) working directory | `WORKDIR /app` |
| `COPY` | Copy files from build context | `COPY --chown=node:node . .` |
| `ADD` | Like COPY + auto-extract local tar + remote URLs | Prefer `COPY` unless you need these |
| `RUN` | Execute a command at **build time** → new layer | `RUN npm ci` |
| `ENV` | Environment variable (build **and** runtime) | `ENV NODE_ENV=production` |
| `ARG` | Build-time variable only (`--build-arg`) | `ARG NODE_VERSION=22` |
| `EXPOSE` | **Documents** the port the app listens on (doesn't publish!) | `EXPOSE 3000` |
| `CMD` | Default command/args when the container starts (overridable) | `CMD ["npm", "start"]` |
| `ENTRYPOINT` | The executable that always runs; CMD becomes its default args | `ENTRYPOINT ["node"]` |
| `USER` | User to run subsequent instructions & the container | `USER node` |
| `VOLUME` | Declare a mount point for persistent data | `VOLUME /data` |
| `HEALTHCHECK` | Command to check container health | see §13 |
| `LABEL` | Metadata key-values | `LABEL org.opencontainers.image.source=…` |
| `STOPSIGNAL` | Signal sent on `docker stop` | `STOPSIGNAL SIGINT` |
| `SHELL` | Change default shell for shell-form commands | `SHELL ["/bin/bash", "-c"]` |
| `ONBUILD` | Trigger instruction for images built FROM this one | rare |

## 6.3 `CMD` vs `ENTRYPOINT` ⭐⭐⭐ (interview favourite)

- **`CMD`**: default command — **replaced** by any arguments passed to `docker run image <args>`.
- **`ENTRYPOINT`**: fixed executable — `docker run` args are **appended** to it (override only with `--entrypoint`).
- Together: `ENTRYPOINT` = the program, `CMD` = its default arguments.

```dockerfile
FROM alpine
ENTRYPOINT ["ping"]
CMD ["localhost"]
```

```bash
docker run pinger              # ping localhost
docker run pinger google.com   # ping google.com   (CMD replaced, ENTRYPOINT kept)
docker run --entrypoint sh -it pinger   # override entrypoint
```

| Dockerfile | `docker run img` runs | `docker run img echo hi` runs |
| --- | --- | --- |
| `CMD ["node","app.js"]` | `node app.js` | `echo hi` |
| `ENTRYPOINT ["node"]` | `node` | `node echo hi` |
| `ENTRYPOINT ["node"]` + `CMD ["app.js"]` | `node app.js` | `node echo hi` |

### Exec form vs shell form ⭐

```dockerfile
CMD ["node", "server.js"]     # ✅ exec form (JSON array): runs node directly as PID 1 → receives SIGTERM
CMD node server.js            # ❌ shell form: runs /bin/sh -c "node server.js" → sh is PID 1,
                              #    signals may not reach node → slow/ungraceful stops (exit 137 after 10s)
```

Use **exec form** for `CMD` and `ENTRYPOINT`. Shell form is fine for `RUN` (where you want `&&`, pipes, env expansion).

### Entrypoint script pattern

```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["node", "server.js"]
```

```sh
#!/bin/sh
set -e
# run migrations, wait for DB, render config…
npx prisma migrate deploy
exec "$@"     # ⭐ replace the shell with CMD so it becomes PID 1 and gets signals
```

## 6.4 `ARG` vs `ENV`

```dockerfile
ARG NODE_VERSION=22               # available only during build (before FROM: only in FROM lines)
FROM node:${NODE_VERSION}-alpine
ARG APP_VERSION                    # must be re-declared after FROM to use in this stage
ENV APP_VERSION=${APP_VERSION}     # persist into the image/runtime
```

```bash
docker build --build-arg NODE_VERSION=20 --build-arg APP_VERSION=1.4.2 -t api .
```

| `ARG` | `ENV` |
| --- | --- |
| Build time only | Build time **and** runtime |
| Set with `--build-arg` | Set in Dockerfile; override with `docker run -e` |
| Not in the running container (but visible in image history ⚠️) | Visible in the container & image metadata |

⚠️ **Never pass secrets via `ARG` or `ENV`** — they're visible in `docker history`/`inspect`. Use BuildKit secret mounts (§8.3).

## 6.5 `COPY` vs `ADD`

| `COPY` | `ADD` |
| --- | --- |
| Copies local files/dirs from the build context | Same, **plus** auto-extracts local `.tar` archives and can download URLs |
| Predictable ✅ | Surprising behaviour; use only when you need extraction |

```dockerfile
COPY package*.json ./
COPY --chown=node:node . .
COPY --from=build /app/dist ./dist     # from another stage (multi-stage)
```

## 6.6 `.dockerignore` ⭐

Excludes files from the **build context** (what gets sent to the builder). Smaller context = faster builds, better caching, no leaked secrets.

```gitignore
node_modules
npm-debug.log
dist
build
.next
coverage
.git
.gitignore
.env
.env.*
!.env.example
Dockerfile*
docker-compose*.yml
*.md
.vscode
.idea
```

Without it, `COPY . .` copies your local `node_modules` (wrong OS binaries!) and `.env` secrets into the image.

## 6.7 Layer Caching ⭐⭐⭐

Each instruction produces a layer. On rebuild, Docker **reuses cached layers** until it hits an instruction whose inputs changed — **everything after that is rebuilt**.

```dockerfile
# ❌ Bad: any source change invalidates the npm install layer
COPY . .
RUN npm ci

# ✅ Good: dependencies re-install only when package files change
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

**Order instructions from least to most frequently changing:** base image → system packages → dependency manifests → install → source code → build.

Cache is invalidated by: changed instruction text, changed files in `COPY`/`ADD`, changed `ARG` values used, or a changed parent layer.

## 6.8 Multi-Stage Builds ⭐⭐⭐

Use one stage to **build** (with compilers, dev dependencies) and a final minimal stage to **run** — copying only the artifacts. Smaller, safer images.

### React (Vite) SPA → Nginx

```dockerfile
# syntax=docker/dockerfile:1

# ---- Build stage ----
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=${VITE_API_URL}
RUN npm run build                      # outputs /app/dist

# ---- Runtime stage ----
FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf — SPA routing + caching
server {
  listen 80;
  root /usr/share/nginx/html;
  location / {
    try_files $uri $uri/ /index.html;           # client-side routes → index.html
  }
  location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable"; # hashed files
  }
}
```

Final image: ~50 MB with only static files + nginx — no Node, no source, no `node_modules`.

### Node.js API (production)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS base
WORKDIR /app

FROM base AS deps
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci          # BuildKit cache for faster installs

FROM deps AS build
COPY . .
RUN npm run build && npm prune --omit=dev                # compile TS, then drop dev deps

FROM base AS runtime
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json ./
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Next.js (standalone output)

```dockerfile
# next.config.js → output: "standalone"
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production PORT=3000 HOSTNAME=0.0.0.0
COPY --from=build --chown=node:node /app/.next/standalone ./
COPY --from=build --chown=node:node /app/.next/static ./.next/static
COPY --from=build --chown=node:node /app/public ./public
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

### Go → scratch / distroless (tiny)

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12
COPY --from=build /app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Build a specific stage: `docker build --target build -t app:build .` (useful for tests in CI).

---

# 7. Dockerfile Best Practices ⭐

1. **Use small, official, pinned base images** — `node:22.11-alpine` or `-slim`; pin by digest for full reproducibility (`node:22-alpine@sha256:…`).
2. **Multi-stage builds** — ship only runtime artifacts.
3. **Order for caching** — manifests → install → source.
4. **`.dockerignore`** — keep context small; exclude `node_modules`, `.git`, `.env`.
5. **Run as non-root** — `USER node` / create a user; never run as root in production.
6. **Exec form for CMD/ENTRYPOINT**; use `exec "$@"` in entrypoint scripts; consider `--init` / `tini`.
7. **One main process per container** (one concern). Use Compose for multiple services.
8. **Deterministic installs** — `npm ci`, lockfiles, `pip install --no-cache-dir -r requirements.txt`.
9. **Combine & clean in one `RUN`** for system packages:

   ```dockerfile
   RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
   ```
   (Files deleted in a **later** layer still exist in earlier layers — clean in the same layer.)
10. **No secrets in images** — not in `COPY`, `ARG`, `ENV`; use secret mounts / runtime env / secret managers.
11. **Add a `HEALTHCHECK`** (or rely on orchestrator probes).
12. **Use `COPY` over `ADD`**.
13. **Set `WORKDIR`** instead of `RUN cd …`.
14. **Label images** with OCI labels (source, version, revision).
15. **Make containers stateless & disposable** — data in volumes/databases, config via env.
16. **Log to stdout/stderr**, not files.
17. **Scan images** for vulnerabilities (Docker Scout, Trivy) and rebuild regularly for security patches.
18. **Lint Dockerfiles** — `hadolint`, `docker build --check`.

---

# 8. Building Images

## 8.1 `docker build`

```bash
docker build -t myapp .                       # build from ./Dockerfile with context "."
docker build -t myapp:1.2.0 -t myapp:latest . # multiple tags
docker build -f docker/Dockerfile.prod -t myapp:prod .   # different Dockerfile
docker build --build-arg NODE_VERSION=20 .
docker build --target build -t myapp:test .   # stop at a stage
docker build --no-cache .                     # ignore cache
docker build --pull .                         # always pull newer base image
docker build --progress=plain .               # full build logs (debugging)
docker build --check .                        # lint the Dockerfile without building
docker build --platform linux/amd64 .         # build for another architecture
```

### Build context ⭐

The last argument (`.`) is the **build context**: the files sent to the builder. `COPY` paths are relative to it — you **can't** `COPY ../secret` outside it. Keep it small with `.dockerignore`.

## 8.2 Multi-platform builds (Apple Silicon ↔ servers)

M1/M2/M3 Macs are **arm64**; most servers are **amd64**. An image built on a Mac may fail on the server with `exec format error`.

```bash
docker buildx create --use --name multi       # once
docker buildx build --platform linux/amd64,linux/arm64 -t myorg/api:1.0 --push .
```

## 8.3 BuildKit features 💡

```dockerfile
# syntax=docker/dockerfile:1

# Cache mounts: keep package manager caches between builds (not in the image)
RUN --mount=type=cache,target=/root/.npm npm ci

# Secret mounts: use a secret during build without leaving it in any layer
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci

# SSH forwarding: clone private repos
RUN --mount=type=ssh git clone git@github.com:org/private.git
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t api .
docker build --ssh default -t api .
```

---

# 9. Volumes & Storage ⭐

## 9.1 Why?

A container's writable layer is **deleted with the container** (`docker rm`). Databases, uploads and anything that must persist need external storage.

## 9.2 Three mount types ⭐

```text
┌──────────── Host ────────────┐
│ /var/lib/docker/volumes/pgdata  ◀── named volume (managed by Docker)
│ /home/asha/project            ◀── bind mount (any host path)
│ memory                         ◀── tmpfs (RAM only, never on disk)
└──────────────────────────────┘
```

| | Named volume | Bind mount | tmpfs |
| --- | --- | --- | --- |
| Managed by | Docker | You (host path) | Memory |
| Syntax | `-v pgdata:/var/lib/postgresql/data` | `-v $(pwd):/app` | `--tmpfs /tmp` |
| Use for | **Persistent data** (databases), sharing between containers | **Development** (live code reload), config files | Sensitive/temporary data |
| Portability | ✅ Works anywhere | ❌ Depends on host paths | — |
| Performance on macOS/Windows | Good | Slower (file sharing across VM) | Fast |
| Pre-populated from image | ✅ (empty volume gets image's content at that path) | ❌ Hides image content at that path | — |

## 9.3 Commands

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata          # Mountpoint on host
docker volume rm pgdata               # ⚠️ deletes data
docker volume prune                   # ⚠️ remove unused volumes

docker run -d --name db -v pgdata:/var/lib/postgresql/data -e POSTGRES_PASSWORD=secret postgres:17
docker run -d -v "$(pwd)/nginx.conf":/etc/nginx/conf.d/default.conf:ro nginx   # read-only bind
docker run --rm -v ./src:/app/src node:22 ls /app/src
```

### `-v` vs `--mount`

```bash
-v pgdata:/data:ro                                     # short, compact
--mount type=volume,source=pgdata,target=/data,readonly # explicit, clearer, errors if bind path missing
--mount type=bind,source="$(pwd)",target=/app
--mount type=tmpfs,target=/tmp,tmpfs-size=64m
```

`-v` with a host path that doesn't exist **creates an empty directory**; `--mount` errors instead (safer).

## 9.4 Back up & restore a volume

```bash
# Backup pgdata → ./backup/pgdata.tar.gz
docker run --rm -v pgdata:/data -v "$(pwd)/backup":/backup alpine \
  tar czf /backup/pgdata.tar.gz -C /data .

# Restore
docker run --rm -v pgdata:/data -v "$(pwd)/backup":/backup alpine \
  sh -c "cd /data && tar xzf /backup/pgdata.tar.gz"
```

For databases, prefer native dumps: `docker exec db pg_dump -U postgres app > dump.sql`.

## 9.5 The `node_modules` bind-mount trick (development)

```yaml
volumes:
  - ./:/app            # live code from host
  - /app/node_modules  # anonymous volume: keep the container's node_modules (Linux binaries)
```

Without the second line, your host's `node_modules` (or its absence) would hide the one installed in the image.

## 9.6 File permission issues

Bind-mounted files keep host UID/GID. If the container runs as `node` (UID 1000) and files are owned by another UID → "permission denied". Fix: run with `-u $(id -u):$(id -g)`, align UIDs, or `chown` in the Dockerfile for image-owned paths.

---

# 10. Networking ⭐

## 10.1 Network drivers

| Driver | Description | Use |
| --- | --- | --- |
| **bridge** (default) | Private internal network on the host; containers get internal IPs; NAT to outside | Single-host apps |
| **user-defined bridge** ⭐ | Like bridge + **automatic DNS by container name** + better isolation | Multi-container apps (Compose creates one) |
| **host** | Container shares the host's network stack (no isolation, no port mapping) | Max performance; Linux only (limited on Desktop) |
| **none** | No networking | Isolated batch jobs |
| **overlay** | Spans multiple hosts | Swarm / multi-host |
| **macvlan / ipvlan** | Container gets its own MAC/IP on the physical network | Legacy apps needing LAN presence |

## 10.2 Container-to-container communication ⭐

```bash
docker network create appnet

docker run -d --name db --network appnet -e POSTGRES_PASSWORD=secret postgres:17
docker run -d --name api --network appnet -e DATABASE_URL=postgres://postgres:secret@db:5432/postgres -p 3000:3000 myapi
#                                                                       ↑ service name as hostname (DNS)
```

- On a **user-defined** network, containers reach each other by **name** (`db`, `api`).
- On the **default bridge**, there's no name resolution (legacy `--link` is deprecated).
- Containers talk on **container ports** (5432), not published host ports. Only publish (`-p`) what the **outside world** needs — the DB above isn't published at all.

## 10.3 Port publishing ⭐

```bash
-p 8080:80              # host 0.0.0.0:8080 → container 80 (reachable from your LAN!)
-p 127.0.0.1:8080:80    # only from this machine
-p 8080:80/udp
-p 3000-3005:3000-3005
```

`EXPOSE` only **documents**; `-p` actually **publishes**.

⚠️ Published ports can bypass host firewalls like UFW on Linux (Docker edits iptables) — bind sensitive services to `127.0.0.1` or don't publish them.

## 10.4 Reaching the host from a container

```bash
# Docker Desktop: host.docker.internal resolves to the host
# Linux: add it explicitly
docker run --add-host=host.docker.internal:host-gateway myapp
```

⚠️ Inside a container, **`localhost` means the container itself**, not your machine or other containers.

## 10.5 Commands

```bash
docker network ls
docker network create appnet
docker network inspect appnet          # connected containers & IPs
docker network connect appnet web      # attach running container
docker network disconnect appnet web
docker network rm appnet
docker network prune
```

---

# 11. Docker Compose ⭐

**Compose** defines and runs **multi-container applications** with one YAML file (`compose.yaml` or `docker-compose.yml`) and one command.

> Use **`docker compose`** (v2, built into the CLI). The old `docker-compose` (v1, Python) is deprecated. The top-level `version:` key is obsolete — omit it.

## 11.1 Full-stack example ⭐

```yaml
# compose.yaml
name: shop

services:
  web:                                   # React (Vite) frontend served by nginx
    build:
      context: ./web
      args:
        VITE_API_URL: http://localhost:8080/api
    ports:
      - "8080:80"
    depends_on:
      - api
    restart: unless-stopped

  api:                                   # Node.js API
    build: ./api
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/shop
      REDIS_URL: redis://cache:6379
    env_file:
      - ./api/.env
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy       # wait until Postgres is ready (not just started)
      cache:
        condition: service_started
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}  # from .env file next to compose.yaml
      POSTGRES_DB: shop
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shop"]
      interval: 5s
      timeout: 3s
      retries: 10
    # no "ports": only reachable by other services on the internal network

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

```bash
# .env (next to compose.yaml — used for ${VAR} interpolation; don't commit real secrets)
DB_PASSWORD=supersecret
```

Compose automatically creates a **network** for the project; services reach each other by **service name** (`db`, `cache`, `api`).

## 11.2 Key fields

| Field | Meaning |
| --- | --- |
| `image` | Image to run |
| `build` | Build from a Dockerfile (`context`, `dockerfile`, `args`, `target`) |
| `ports` | `"host:container"` — quote them (YAML parses `60:60` oddly) |
| `expose` | Expose to other services only (documentation; internal ports are reachable anyway) |
| `environment` / `env_file` | Container env variables |
| `volumes` | Named volumes, bind mounts |
| `depends_on` | Start order; with `condition: service_healthy` waits for healthchecks |
| `healthcheck` | Health test |
| `restart` | `no`, `always`, `on-failure`, `unless-stopped` |
| `networks` | Custom networks (default network created automatically) |
| `command` / `entrypoint` | Override CMD / ENTRYPOINT |
| `working_dir`, `user` | Same as run flags |
| `deploy.resources.limits` | CPU/memory limits |
| `profiles` | Optional services started only with `--profile` |
| `develop.watch` | Sync/rebuild on file changes (`docker compose watch`) |

## 11.3 Compose commands ⭐

```bash
docker compose up                 # build (if needed) + create + start, attached logs
docker compose up -d              # ⭐ detached
docker compose up -d --build      # ⭐ rebuild images first
docker compose up -d api          # only api (and its dependencies)
docker compose down               # stop + remove containers & network
docker compose down -v            # ⚠️ also remove named volumes (DB data!)
docker compose ps                 # service status
docker compose logs -f api        # ⭐ follow logs of a service
docker compose exec api sh        # ⭐ shell in a running service
docker compose run --rm api npm test   # one-off container for a task
docker compose build              # build images
docker compose pull               # pull latest images
docker compose restart api
docker compose stop / start
docker compose config             # ⭐ validate & print the resolved config (with env interpolated)
docker compose top
docker compose watch              # live sync/rebuild using develop.watch
docker compose -f compose.yaml -f compose.prod.yaml up -d   # merge multiple files
docker compose --profile tools up -d
```

## 11.4 Development setup with live reload

```yaml
# compose.dev.yaml
services:
  api:
    build:
      context: ./api
      target: deps                 # stage with dev dependencies
    command: npm run dev           # nodemon / tsx watch
    volumes:
      - ./api:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"
      - "9229:9229"                # Node debugger
```

Or with **Compose Watch** (no bind mounts; works well on macOS/Windows):

```yaml
services:
  web:
    build: ./web
    develop:
      watch:
        - action: sync             # copy changed files into the container
          path: ./web/src
          target: /app/src
        - action: rebuild          # rebuild image when dependencies change
          path: ./web/package.json
```

```bash
docker compose watch
```

## 11.5 Override files & environments

```text
compose.yaml            → base (shared)
compose.override.yaml   → automatically merged for local dev
compose.prod.yaml       → production differences (no bind mounts, restart policies, limits)
```

```bash
docker compose up -d                                         # base + override
docker compose -f compose.yaml -f compose.prod.yaml up -d    # base + prod
```

## 11.6 Profiles (optional services)

```yaml
services:
  adminer:
    image: adminer
    ports: ["8081:8080"]
    profiles: ["tools"]
```

`docker compose --profile tools up -d` starts it; otherwise it's ignored.

## 11.7 `depends_on` caveat ⭐

`depends_on` alone only controls **start order**, not readiness. A database container "started" doesn't mean it accepts connections. Use `condition: service_healthy` with a healthcheck — and still make your app **retry** connections (in production, orchestrators don't guarantee order).

---

# 12. Registries: Push & Pull

```bash
docker login                                  # Docker Hub
docker login ghcr.io -u USERNAME              # GitHub Container Registry (use a token)
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-south-1.amazonaws.com

docker tag myapi:latest ghcr.io/myorg/myapi:1.4.2
docker push ghcr.io/myorg/myapi:1.4.2
docker pull ghcr.io/myorg/myapi:1.4.2
docker logout ghcr.io
```

| Registry | Notes |
| --- | --- |
| Docker Hub | Default; public & private repos; pull rate limits for anonymous/free users |
| GHCR (GitHub) | Integrated with GitHub Actions & repo permissions |
| AWS ECR / Google Artifact Registry / Azure ACR | Cloud-native, IAM-integrated |
| Harbor / self-hosted `registry:2` | On-prem |

### Tagging strategy ⭐

```text
myapi:1.4.2            ← immutable release (SemVer)
myapi:1.4 / myapi:1    ← moving "latest patch/minor" tags
myapi:sha-3f2a9c1      ← exact git commit (great for traceability)
myapi:main             ← latest build of a branch
myapi:latest           ← avoid relying on it in production
```

- Tags are **mutable** (can be moved to a new image); **digests are immutable** — deploy by digest for exact reproducibility.
- Never deploy `latest` to production: you can't tell what's running or roll back reliably.

---

# 13. Resources, Restart Policies, Health & Logs

## 13.1 Resource limits

```bash
docker run -d --memory 512m --memory-swap 512m --cpus 1.5 --pids-limit 200 myapi
docker update --memory 1g --cpus 2 api        # change limits on a running container
docker stats                                   # watch usage
```

- Exceeding the memory limit → the kernel **OOM-kills** the process (exit code **137**, `"OOMKilled": true` in `docker inspect`).
- Node.js: set `--max-old-space-size` below the container limit (Node ≥ 20 is container-aware, but explicit is safer for large heaps).

```yaml
# compose
deploy:
  resources:
    limits:
      cpus: "1.0"
      memory: 512M
```

## 13.2 Restart policies

| Policy | Behaviour |
| --- | --- |
| `no` (default) | Never restart |
| `on-failure[:max]` | Restart on non-zero exit (optionally max retries) |
| `always` | Always restart, including after daemon restart (even if manually stopped, it restarts when the daemon restarts) |
| `unless-stopped` ⭐ | Like always, but not if you stopped it manually |

## 13.3 Healthchecks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:3000/health || exit 1
```

```bash
docker ps                                       # STATUS: Up 2 min (healthy) / (unhealthy)
docker inspect --format '{{json .State.Health}}' api
```

Exit `0` = healthy, `1` = unhealthy. Plain Docker only **reports** health (Compose `depends_on` and orchestrators act on it); Kubernetes uses its own liveness/readiness probes.

Make sure the tool exists in the image (`curl` isn't in Alpine/distroless by default → use `wget`, a tiny Node script, or the orchestrator's probe).

## 13.4 Logging

- Apps should log to **stdout/stderr**; Docker captures them (`docker logs`).
- Default driver `json-file` can **fill the disk** — configure rotation:

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

```yaml
# per service in compose
logging:
  driver: json-file
  options: { max-size: "10m", max-file: "3" }
```

Other drivers: `local`, `journald`, `syslog`, `fluentd`, `awslogs`, `gcplogs` — or ship logs with an agent (Datadog, Loki/Promtail, ELK).

---

# 14. Security ⭐

| Practice | How |
| --- | --- |
| **Non-root user** | `USER node` / `USER 10001`; `docker run -u` |
| **Minimal base images** | slim / alpine / distroless / scratch → fewer CVEs |
| **Pin & update base images** | Pin versions/digests; rebuild regularly for patches; Dependabot/Renovate for Dockerfiles |
| **Scan images** | `docker scout cves myimage`, `trivy image myimage`, registry scanning |
| **No secrets in images** | Use runtime env, Docker/Compose secrets, cloud secret managers, BuildKit `--secret` |
| **Read-only filesystem** | `--read-only --tmpfs /tmp` |
| **Drop capabilities** | `--cap-drop ALL --cap-add NET_BIND_SERVICE` |
| **No new privileges** | `--security-opt no-new-privileges` |
| **Never `--privileged`** unless absolutely required | Gives near-full host access |
| **Don't mount the Docker socket** | `-v /var/run/docker.sock:/var/run/docker.sock` = root on the host |
| **Limit resources** | Memory/CPU/PIDs limits prevent DoS |
| **Network isolation** | Separate networks per tier; don't publish internal services; bind to `127.0.0.1` |
| **Rootless Docker / Podman** | Daemon runs without root |
| **Sign & verify images** | Docker Content Trust, Sigstore/cosign; SBOMs & provenance attestations (`docker buildx build --sbom=true --provenance=true`) |
| **Lint** | hadolint, `docker build --check` |

⚠️ Membership in the `docker` group is effectively **root on the host** — treat it accordingly.

```bash
docker run -d --name api \
  --read-only --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  -u 10001:10001 \
  --memory 256m --cpus 0.5 --pids-limit 100 \
  -p 127.0.0.1:3000:3000 \
  myapi:1.4.2
```

---

# 15. Environment Variables & Secrets

## 15.1 Ways to pass configuration

```bash
docker run -e NODE_ENV=production -e PORT=3000 myapi
docker run --env-file .env myapi
docker run -e API_KEY myapi        # pass through value from the host shell
```

Compose:

```yaml
services:
  api:
    image: myapi
    environment:
      LOG_LEVEL: ${LOG_LEVEL:-info}   # interpolation with default
    env_file: .env
```

### Compose `.env` vs `env_file` (common confusion)

| `.env` file next to compose.yaml | `env_file:` in a service |
| --- | --- |
| Used by **Compose itself** to interpolate `${VARS}` in the YAML | Variables injected **into the container** |
| Not automatically passed into containers | — |

Precedence (highest first, simplified): `docker compose run -e` → shell environment → `environment:` → `env_file:` → Dockerfile `ENV`.

## 15.2 Secrets ⭐

Environment variables are convenient but visible via `docker inspect` and to any process in the container. Better options:

```yaml
# Compose secrets: mounted as files at /run/secrets/<name>
services:
  api:
    image: myapi
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password   # app reads the file
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

Production: Docker Swarm secrets, Kubernetes Secrets (+ external secret operators), AWS Secrets Manager / Parameter Store, HashiCorp Vault, Doppler.

### Frontend build-time variables ⚠️

`VITE_*` / `NEXT_PUBLIC_*` values are **baked into the JS bundle at build time** → (a) they're public, (b) one image can't serve multiple environments. For "build once, deploy anywhere": serve a runtime `config.js`/`/config.json` generated at container start, or inject via nginx `envsubst` templates.

---

# 16. Debugging & Troubleshooting ⭐

## 16.1 Debugging workflow

```bash
docker ps -a                          # is it running? exited? restarting?
docker logs --tail 200 <name>         # what did it say before dying?
docker inspect <name> --format '{{.State.ExitCode}} {{.State.OOMKilled}} {{.State.Error}}'
docker exec -it <name> sh             # look around inside (env, files, network)
docker run -it --rm --entrypoint sh <image>   # open a shell in the image without starting the app
docker events --since 10m             # what did the daemon do?
docker stats                          # resource pressure?
docker compose config                 # is my compose file what I think it is?
docker build --progress=plain --no-cache .    # detailed build output
```

💡 Debugging distroless/minimal images (no shell): `docker debug <container>` (Docker Desktop), or attach a debug container sharing the namespaces:

```bash
docker run -it --rm --pid=container:api --network=container:api nicolaka/netshoot
```

## 16.2 Common problems ⭐

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `port is already allocated` | Another container/process uses the host port | `docker ps`, stop it, or use a different host port (`-p 8081:80`) |
| Container exits immediately | Main process finished / crashed / runs in background | Check `docker logs`; run server in foreground; use exec form |
| Exit code **137** | OOM kill or `docker kill`/stop timeout | Raise memory limit, fix leak; exec-form CMD so SIGTERM is handled |
| Exit code **127** | Command not found in image | Check path/binary; Alpine lacks bash (`sh`); install it |
| `exec format error` | Architecture mismatch (arm64 vs amd64) or script missing shebang / CRLF line endings | Build `--platform linux/amd64` / multi-arch; fix `#!/bin/sh` & LF endings |
| `permission denied` on `/var/run/docker.sock` | User not in `docker` group | `sudo usermod -aG docker $USER`, re-login |
| App unreachable on `localhost:PORT` | Port not published, or app listens on `127.0.0.1` inside container | `-p host:container`; make app listen on **`0.0.0.0`** |
| Container can't reach DB at `localhost` | `localhost` is the container itself | Use service name (`db`) on a shared network |
| DB connection refused at startup | DB not ready yet | Healthcheck + `depends_on: condition: service_healthy` + retries |
| Code changes not reflected | Running old image | `docker compose up -d --build`; for dev use bind mounts/watch |
| Huge images | No multi-stage, copying `node_modules`, full base image | Multi-stage, `.dockerignore`, slim/alpine base |
| Slow builds | Cache busted early, large context | Reorder layers, `.dockerignore`, cache mounts |
| `no space left on device` | Old images, build cache, logs, volumes | `docker system df` → `docker system prune`; log rotation |
| `pull access denied` / `manifest unknown` | Wrong name/tag, private repo, not logged in | Check name/tag, `docker login` |
| Docker Hub rate limit (`toomanyrequests`) | Anonymous pulls limited | Log in, use a mirror/cache, or other registries |
| Bind-mount permission errors | UID/GID mismatch | `-u $(id -u):$(id -g)`, chown, align UIDs |
| Container keeps restarting | Crash loop + restart policy | `docker logs`; temporarily `--restart no`; check env/config |
| DNS resolution fails between containers | Using default bridge | Use a user-defined network / Compose |

---

# 17. Cleanup

```bash
docker system df                 # ⭐ disk usage: images, containers, volumes, build cache
docker container prune           # remove stopped containers
docker image prune               # dangling images
docker image prune -a            # ⚠️ all unused images
docker volume prune              # ⚠️ unused ANONYMOUS volumes (Docker 23+ default)
docker volume prune -a           # ⚠️⚠️ ALL unused volumes, including named ones (database data!)
docker network prune
docker builder prune             # build cache
docker system prune              # stopped containers + unused networks + dangling images + build cache
docker system prune -a --volumes # ⚠️⚠️ everything unused: all unused images + anonymous volumes

docker rm -f $(docker ps -aq)    # ⚠️ force remove ALL containers
docker stop $(docker ps -q)      # stop all running containers
```

---

# 18. How Docker Works Internally ⭐

Containers are not VMs — they're **regular Linux processes** with isolation and limits applied by kernel features.

## 18.1 Namespaces — isolation (what a process can *see*)

| Namespace | Isolates |
| --- | --- |
| **PID** | Process IDs — the container sees its main process as PID 1 |
| **NET** | Network interfaces, IPs, routing, ports |
| **MNT** | Filesystem mount points |
| **UTS** | Hostname |
| **IPC** | Inter-process communication |
| **USER** | User/group IDs (root inside can map to unprivileged outside — rootless) |
| **CGROUP** | View of cgroup hierarchy |

## 18.2 cgroups — limits (what a process can *use*)

Control groups limit and account for **CPU, memory, disk I/O, PIDs**. `--memory 512m` and `--cpus 1` are cgroup settings; exceeding memory triggers the OOM killer.

## 18.3 Union filesystem & copy-on-write ⭐

```text
┌────────────────────────────┐
│ Container writable layer    │  ← created per container; deleted with it
├────────────────────────────┤
│ Layer 4: COPY . .           │  ┐
│ Layer 3: RUN npm ci         │  │ read-only image layers,
│ Layer 2: COPY package*.json │  │ shared by ALL containers & images using them
│ Layer 1: node:22-alpine     │  ┘
└────────────────────────────┘
        OverlayFS merges them into one view
```

- Image layers are **read-only** and **content-addressed** (SHA-256) → shared and cached.
- Each container adds a thin **writable layer**. Modifying a file from a lower layer **copies it up** first (**copy-on-write**).
- 10 containers from the same image share the image layers → little extra disk.
- Writes to the container layer are slower and ephemeral → use volumes for heavy I/O and persistence.

## 18.4 What happens on `docker run nginx` ⭐

```text
1. CLI sends request to dockerd via REST API (/var/run/docker.sock)
2. dockerd checks for nginx:latest locally; if missing, pulls manifest + layers from the registry
3. Creates a container: config + writable layer (overlayfs) + network endpoint (veth pair on bridge, IP, iptables NAT for -p)
4. containerd prepares the OCI bundle → runc creates namespaces & cgroups and starts the process
5. runc exits; containerd-shim keeps the process attached (stdio, exit code)
6. Process runs as PID 1 in its namespaces; logs go to the logging driver
```

## 18.5 Image anatomy

An image = **manifest** (list of layer digests + config digest) + **config** (env, cmd, entrypoint, user, history) + **layers** (tar diffs). Multi-arch images use a **manifest list / image index** pointing to per-platform manifests.

```bash
docker buildx imagetools inspect node:22-alpine   # see platforms in the index
docker image inspect node:22-alpine --format '{{json .RootFS.Layers}}'
```

## 18.6 PID 1 and signals ⭐

The container's main process is **PID 1**, which has special signal semantics: it doesn't get default signal handlers and must **reap zombie** child processes. Problems: shell-form CMD (`sh` as PID 1 swallowing SIGTERM), apps not handling SIGTERM → slow stops and data loss.

Fixes: exec form, `exec "$@"` in entrypoints, handle SIGTERM in the app (graceful shutdown), `docker run --init` / `tini`.

```js
// Node.js graceful shutdown
process.on("SIGTERM", async () => {
  server.close(() => process.exit(0)); // stop accepting, finish in-flight requests
  setTimeout(() => process.exit(1), 10_000).unref();
});
```

---

# 19. Docker in CI/CD

## 19.1 GitHub Actions: build & push with cache ⭐

```yaml
# .github/workflows/docker.yml
name: Docker
on:
  push:
    branches: [main]
    tags: ["v*"]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-qemu-action@v3        # multi-arch emulation
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 19.2 Typical pipeline

```text
lint + unit tests → docker build (target: test) → run tests in container
→ build final image → scan (Trivy/Scout) → push (tag: sha + semver)
→ deploy same image to staging → smoke tests → promote to production (by digest)
```

**Build once, deploy everywhere**: the exact image tested in staging is promoted to production; only configuration (env/secrets) differs.

## 19.3 Where containers run in production

| Platform | Notes |
| --- | --- |
| Kubernetes (EKS, GKE, AKS) | Full orchestration; most powerful, most complex |
| AWS ECS / Fargate | Managed containers without managing servers |
| Google Cloud Run / Azure Container Apps | Serverless containers; scale to zero |
| Fly.io, Render, Railway | Developer-friendly PaaS |
| A VM with Docker Compose | Simple apps, small teams |

---

# 20. Orchestration: Swarm & Kubernetes Basics

Running containers on **many machines** needs orchestration: scheduling, scaling, self-healing, rolling updates, service discovery, load balancing, secrets.

### Docker Swarm (built-in, simpler)

```bash
docker swarm init
docker stack deploy -c compose.yaml shop     # Compose file → services
docker service ls
docker service scale shop_api=5
docker service update --image myapi:1.5.0 shop_api   # rolling update
```

### Kubernetes concept mapping ⭐

| Docker / Compose | Kubernetes |
| --- | --- |
| Container | Container (inside a **Pod**) |
| `docker run` / service in Compose | **Deployment** (manages ReplicaSets of Pods) |
| Published port / internal DNS name | **Service** (ClusterIP / NodePort / LoadBalancer) |
| Reverse proxy | **Ingress** / Gateway API |
| Volume | **PersistentVolume / PersistentVolumeClaim** |
| `environment` / `.env` | **ConfigMap** |
| secrets | **Secret** |
| healthcheck | **liveness / readiness / startup probes** |
| `restart: always` | Controller self-healing |
| `docker compose up` | `kubectl apply -f` |

```yaml
# Minimal Deployment + Service
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
        - name: api
          image: ghcr.io/myorg/api:1.4.2
          ports: [{ containerPort: 3000 }]
          resources:
            limits: { memory: "512Mi", cpu: "500m" }
          readinessProbe:
            httpGet: { path: /health, port: 3000 }
---
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 3000 }]
```

> Kubernetes uses containerd/CRI-O directly (not dockerd), but **images you build with Docker run unchanged** because they're OCI images.

---

# 21. Real-World Recipes ⭐

### 1. Run a database locally in one command

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=secret -p 5432:5432 -v pgdata:/var/lib/postgresql/data postgres:17-alpine
docker run -d --name mongo -p 27017:27017 -v mongodata:/data/db mongo:8
docker run -d --name redis -p 6379:6379 redis:7-alpine
```

### 2. Open a psql / mongosh / redis-cli shell

```bash
docker exec -it pg psql -U postgres
docker exec -it mongo mongosh
docker exec -it redis redis-cli
```

### 3. Dump & restore a Postgres database

```bash
docker exec pg pg_dump -U postgres mydb > backup.sql
docker exec -i pg psql -U postgres mydb < backup.sql
```

### 4. Run a one-off command with a specific tool version (no install)

```bash
docker run --rm -v "$(pwd)":/app -w /app node:20 npm test
docker run --rm -v "$(pwd)":/app -w /app python:3.12 python script.py
```

### 5. Get a shell in a container that keeps crashing

```bash
docker run -it --rm --entrypoint sh myapi:latest      # start the image with a shell instead of the app
```

### 6. See why a container stopped

```bash
docker ps -a --filter name=api
docker logs --tail 100 api
docker inspect api --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}} err={{.State.Error}}'
```

### 7. Rebuild and restart one service in Compose

```bash
docker compose up -d --build api
```

### 8. Reset a Compose environment completely (including DB data) ⚠️

```bash
docker compose down -v --remove-orphans
docker compose up -d --build
```

### 9. Copy a file out of an image without running it

```bash
id=$(docker create myapp:latest)
docker cp "$id":/app/dist ./dist-from-image
docker rm "$id"
```

### 10. Find what's taking disk space and clean it

```bash
docker system df -v
docker system prune          # add -a / --volumes carefully
```

### 11. Check the size of each layer (why is my image big?)

```bash
docker image history --no-trunc myapp:latest
# or use the "dive" tool: dive myapp:latest
```

### 12. Build for the server's architecture from an Apple Silicon Mac

```bash
docker buildx build --platform linux/amd64 -t myorg/api:1.0 --push .
```

### 13. Move an image to a server without a registry

```bash
docker save myapp:1.0 | gzip > myapp.tar.gz
scp myapp.tar.gz server:
ssh server 'gunzip -c myapp.tar.gz | docker load'
```

### 14. Test a production build of a React app locally

```bash
docker build -t web:prod ./web
docker run --rm -p 8080:80 web:prod      # open http://localhost:8080
```

### 15. Connect a container to a service running on your machine

```bash
docker run --add-host=host.docker.internal:host-gateway -e API_URL=http://host.docker.internal:4000 myapp
```

### 16. Limit a runaway container

```bash
docker update --memory 512m --memory-swap 512m --cpus 1 api
```

### 17. Follow logs of all Compose services, with timestamps

```bash
docker compose logs -f -t --tail 50
```

### 18. Stop and remove everything

```bash
docker stop $(docker ps -q); docker rm $(docker ps -aq)    # ⚠️ all containers
```

### 19. Run a container on a schedule (cron-like job)

```bash
# host crontab
0 2 * * * docker run --rm --env-file /srv/jobs/.env myorg/cleanup-job:1.2
```

### 20. Debug networking between containers

```bash
docker run -it --rm --network appnet nicolaka/netshoot   # has curl, dig, nslookup, tcpdump…
# inside: dig db ; curl -v http://api:3000/health
```

### 21. Pass a different command to a Compose service once

```bash
docker compose run --rm api npx prisma migrate deploy
```

### 22. View the environment and config a container actually got

```bash
docker exec api env | sort
docker inspect api --format '{{json .Config.Env}}'
```

---

# 22. Cheat Sheet ⭐

### Images

| Command | Description |
| --- | --- |
| `docker pull img:tag` | Download image |
| `docker images` | List images |
| `docker build -t name:tag .` | Build image from Dockerfile |
| `docker build --no-cache .` | Build without cache |
| `docker tag src dest` | Add a tag |
| `docker push name:tag` | Upload to registry |
| `docker rmi img` | Remove image |
| `docker image history img` | Show layers |
| `docker image inspect img` | Metadata |
| `docker save -o f.tar img` / `docker load -i f.tar` | Export / import |

### Containers

| Command | Description |
| --- | --- |
| `docker run -d --name n -p 8080:80 img` | Run detached with port |
| `docker run -it --rm img sh` | Interactive throwaway shell |
| `docker ps` / `docker ps -a` | Running / all containers |
| `docker logs -f n` | Follow logs |
| `docker exec -it n sh` | Shell into running container |
| `docker stop n` / `start n` / `restart n` | Lifecycle |
| `docker rm n` / `docker rm -f n` | Remove (force) ⚠️ |
| `docker inspect n` | Full details |
| `docker stats` | Live resource usage |
| `docker cp n:/path ./` | Copy files |
| `docker top n` | Processes |

### Volumes & Networks

| Command | Description |
| --- | --- |
| `docker volume create/ls/inspect/rm` | Manage volumes |
| `-v vol:/path` | Named volume |
| `-v $(pwd):/app` | Bind mount |
| `docker network create/ls/inspect/rm` | Manage networks |
| `--network net` | Join network (DNS by container name) |

### Compose

| Command | Description |
| --- | --- |
| `docker compose up -d` | Start all services |
| `docker compose up -d --build` | Rebuild + start |
| `docker compose down` | Stop + remove |
| `docker compose down -v` | + delete volumes ⚠️ |
| `docker compose ps` | Status |
| `docker compose logs -f svc` | Follow logs |
| `docker compose exec svc sh` | Shell into service |
| `docker compose run --rm svc cmd` | One-off command |
| `docker compose config` | Validate/print config |
| `docker compose watch` | Live sync/rebuild |

### Cleanup

| Command | Description |
| --- | --- |
| `docker system df` | Disk usage |
| `docker system prune` | Remove unused data |
| `docker system prune -a --volumes` | Remove all unused images, containers, networks, cache + anonymous volumes ⚠️ |
| `docker image prune -a` | Unused images ⚠️ |
| `docker volume prune` / `-a` | Unused anonymous volumes / all unused volumes incl. named ⚠️ |
| `docker builder prune` | Build cache |

### Dockerfile quick reference

```dockerfile
FROM node:22-alpine AS build      # base image (named stage)
WORKDIR /app                      # cd + mkdir
COPY package*.json ./             # copy manifests first (cache)
RUN npm ci                        # build-time command
COPY . .                          # source
RUN npm run build
FROM nginx:1.27-alpine            # final stage
COPY --from=build /app/dist /usr/share/nginx/html
ENV KEY=value                     # runtime env
ARG VERSION                       # build arg
EXPOSE 80                         # document port
USER nginx                        # non-root (where supported)
HEALTHCHECK CMD wget -qO- http://localhost/ || exit 1
CMD ["nginx", "-g", "daemon off;"]   # exec form
```

---

# 23. Interview Questions & Answers ⭐

### Fundamentals

**1. What is Docker?**
A platform for packaging applications and their dependencies into portable containers that run consistently across environments. It provides tools to build images, run containers, and share images through registries.

**2. Container vs virtual machine?**
A VM virtualizes hardware and runs a full guest OS on a hypervisor, so it's heavy and slow to start, with strong isolation. A container is an isolated process sharing the host kernel, using namespaces for isolation and cgroups for limits. It's lightweight (MBs), starts in seconds and allows high density, with weaker isolation by default.

**3. Image vs container?**
An image is a read-only template made of filesystem layers plus metadata. A container is a runnable instance of an image, with its own writable layer, network and process. Many containers can run from one image.

**4. What is a Dockerfile?**
A text file of instructions (`FROM`, `COPY`, `RUN`, `CMD`, …) that Docker executes to build an image reproducibly.

**5. What are layers and why do they matter?**
Each filesystem-changing instruction creates a read-only, content-addressed layer. Layers are cached and shared between images, which speeds up builds and saves space. Order instructions so rarely changing steps (dependency install) come before frequently changing ones (source code).

**6. Explain Docker's architecture.**
The CLI sends commands to the Docker daemon (`dockerd`) through a REST API. The daemon manages images, builds (BuildKit), networks and volumes, and delegates container execution to containerd, which uses runc to create namespaced, cgroup-limited processes. Images are pulled from and pushed to registries.

**7. What is Docker Hub / a registry?**
A service storing and distributing images. Docker Hub is the default public registry; others include GHCR, ECR, GCR/Artifact Registry, ACR and Harbor.

**8. What does `docker run` do?**
Pulls the image if it's missing, creates a container (writable layer, network, mounts) and starts its main process. It's `create` plus `start`.

### Dockerfile

**9. `CMD` vs `ENTRYPOINT`?**
`CMD` is the default command and arguments, replaced by any arguments given to `docker run`. `ENTRYPOINT` is the fixed executable, and `docker run` arguments are appended to it. Used together, `ENTRYPOINT` is the program and `CMD` supplies its default arguments. Override the entrypoint with `--entrypoint`.

**10. Exec form vs shell form?**
Exec form (`["node","app.js"]`) runs the binary directly as PID 1, so it receives signals like SIGTERM. Shell form (`node app.js`) wraps it in `/bin/sh -c`, so the shell is PID 1 and may not forward signals, which leads to ungraceful stops. Use exec form for `CMD` and `ENTRYPOINT`.

**11. `COPY` vs `ADD`?**
Both copy files from the build context. `ADD` also extracts local tar archives and fetches URLs, which is surprising behaviour, so prefer `COPY`.

**12. `ARG` vs `ENV`?**
`ARG` exists only at build time (`--build-arg`). `ENV` persists into the image and the running container. Neither is safe for secrets, because both are visible in history or inspect output.

**13. `EXPOSE` vs `-p`?**
`EXPOSE` only documents the port the app listens on. `-p host:container` actually publishes it to the host.

**14. What is a multi-stage build and why use it?**
Multiple `FROM` stages in one Dockerfile: build in a stage with compilers and dev dependencies, then copy only the artifacts into a minimal runtime stage. The result is smaller, faster to pull and more secure, because there are no build tools or source in production.

**15. How do you reduce image size?**
Slim, Alpine or distroless base images, multi-stage builds, `.dockerignore`, production-only dependencies (`npm ci --omit=dev`), combining `RUN` steps and cleaning package caches in the same layer, and not copying unnecessary files.

**16. How do you speed up builds?**
Order layers for cache hits (manifests before source), keep the build context small with `.dockerignore`, use BuildKit cache mounts and parallel stages, cache in CI (`cache-from`/`cache-to`), and pin base images.

**17. What is `.dockerignore`?**
It excludes files from the build context, which speeds up builds, improves cache hits and keeps secrets and local artifacts (`.env`, `node_modules`, `.git`) out of images.

**18. How do you handle secrets during a build?**
BuildKit secret mounts: `RUN --mount=type=secret,id=npmrc …` with `docker build --secret id=npmrc,src=…`. The secret is available only during that `RUN` step and never stored in a layer. Never use `ARG`, `ENV` or `COPY` for secrets.

### Data & networking

**19. How do you persist data?**
Named volumes (managed by Docker) for databases and other persistent data, bind mounts (host paths) for development and config files, and tmpfs for in-memory temporary data. The container's writable layer is deleted with the container.

**20. Volume vs bind mount?**
A volume is managed by Docker, portable, pre-populated from the image, and performs well on Desktop. A bind mount maps any host path, which is great for live code in development but depends on the host's paths and permissions.

**21. How do containers communicate?**
Put them on the same user-defined network (Compose does this automatically), and they reach each other by container or service name through Docker's built-in DNS, on container ports. Publish only the ports that external clients need.

**22. What network drivers exist?**
bridge (default, single host), user-defined bridge (with DNS), host (shares the host's network stack), none, overlay (multi-host or Swarm), and macvlan/ipvlan.

**23. Why can't my container reach `localhost:5432`?**
Inside a container, `localhost` is the container itself. Use the database service name on a shared network, or `host.docker.internal` to reach the host machine.

**24. Why is my app unreachable even with `-p`?**
The app is probably listening on `127.0.0.1` inside the container. It must bind to `0.0.0.0`. Also check that the published container port matches the one the app listens on.

### Compose

**25. What is Docker Compose?**
A tool to define a multi-container application (services, networks, volumes, environment) in a YAML file and manage it with commands like `docker compose up` and `down`. It's mainly used for local development, testing and simple single-host deployments.

**26. Does `depends_on` wait until a service is ready?**
Not by default; it only controls start order. Use `condition: service_healthy` with a healthcheck, and still implement connection retries in the app.

**27. `docker compose down` vs `down -v`?**
`down` removes containers and networks. `-v` also deletes named volumes, which wipes database data.

**28. `.env` vs `env_file` in Compose?**
`.env` next to the compose file is used to interpolate `${VAR}` in the YAML. `env_file:` injects variables into the container.

**29. How do you use different configurations for dev and prod?**
A base `compose.yaml` plus override files (`compose.override.yaml` is auto-loaded for dev, and `-f compose.prod.yaml` for prod), profiles, environment interpolation, and different build targets.

### Operations & security

**30. What happens when you run `docker stop`?**
Docker sends SIGTERM to PID 1, waits for the grace period (10 seconds by default), then sends SIGKILL. Apps should handle SIGTERM to shut down gracefully.

**31. What does exit code 137 mean?**
The process was killed with SIGKILL (128+9), usually because it ran out of memory (OOM-killed) or didn't stop within the timeout. Check `docker inspect` for `OOMKilled`.

**32. What are restart policies?**
`no`, `on-failure[:N]`, `always` and `unless-stopped`. They control whether Docker restarts containers when they exit or when the daemon restarts.

**33. How do you limit resources?**
`--memory`, `--cpus` and `--pids-limit` (or `deploy.resources.limits` in Compose). These are enforced by cgroups.

**34. What is a healthcheck?**
A command Docker runs periodically to mark a container healthy or unhealthy. Compose and orchestrators can wait for it or act on it.

**35. How do you secure containers?**
Run as non-root, use minimal and pinned images and scan them, keep secrets out of images, use a read-only filesystem, drop capabilities, set no-new-privileges, avoid `--privileged` and Docker socket mounts, set resource limits, isolate networks, and sign images and generate SBOMs.

**36. Why is mounting `/var/run/docker.sock` dangerous?**
It gives the container full control of the Docker daemon, which is effectively root access on the host.

**37. How do you debug a container that keeps crashing?**
`docker ps -a` to see its status and exit code, `docker logs`, and `docker inspect` (ExitCode, OOMKilled, Error). Then start the image with `--entrypoint sh` to look around, check the env and config, run the command manually, and check resource limits.

### Internals & ecosystem

**38. How does Docker isolate containers?**
Linux namespaces (PID, NET, MNT, UTS, IPC, USER) isolate what a process can see. cgroups limit what it can use. Union filesystems (OverlayFS) provide layered, copy-on-write root filesystems.

**39. What is copy-on-write?**
Image layers are shared and read-only. When a container modifies a file, the file is first copied into the container's writable layer, so other containers aren't affected.

**40. What are containerd and runc? What is OCI?**
containerd is a high-level container runtime that manages images and container lifecycles. runc is the low-level OCI runtime that creates the namespaces and cgroups. OCI defines standard image and runtime specifications, so Docker-built images run on Podman, containerd and Kubernetes.

**41. Does Kubernetes use Docker?**
Kubernetes removed dockershim in v1.24 and talks to CRI runtimes like containerd or CRI-O directly. Images built with Docker still work, because they're OCI images.

**42. Docker Compose vs Kubernetes?**
Compose runs multi-container apps on a single host and is simple, which suits development. Kubernetes orchestrates containers across clusters with scheduling, auto-scaling, self-healing, rolling updates, service discovery and secrets, at the cost of much more complexity.

**43. What is the `latest` tag problem?**
`latest` is just a default tag name, not "newest", and it's mutable. Deploying it makes versions untraceable and rollbacks unreliable. Use SemVer or commit-SHA tags, or digests.

**44. What is the difference between `docker stop` and `docker kill`?**
`stop` sends SIGTERM, then SIGKILL after a timeout, so the app can shut down gracefully. `kill` sends SIGKILL immediately (or a signal you specify).

**45. How do you build images for both ARM and x86?**
`docker buildx build --platform linux/amd64,linux/arm64 --push`, using QEMU emulation or native builders. It produces a multi-arch manifest list.

**46. Describe a Docker-based CI/CD pipeline.**
Run the tests, build the image with cache, scan it, tag it with the commit SHA and version, push it to a registry, deploy the same image to staging and smoke-test it, then promote that exact digest to production. Configuration and secrets are injected at runtime.

---

# 24. Practice Exercises

1. **Hello containers:** run `nginx` on port 8080, view its logs, `exec` into it, edit `index.html`, then remove the container.
2. **Image layers:** write a Node.js Dockerfile with bad layer ordering, change one source file, time the rebuild; fix the ordering and compare.
3. **Multi-stage:** containerize a React/Vite app with a build stage + nginx runtime; compare image sizes (`docker images`).
4. **Volumes:** run Postgres with a named volume, create a table, remove the container, start a new one with the same volume — data should persist. Back it up to a tar file.
5. **Networking:** create a network, run a DB and an API that connects to it by name; verify DNS with `netshoot`.
6. **Compose:** write a `compose.yaml` for web + api + db + redis with healthchecks and `depends_on: condition: service_healthy`.
7. **Dev workflow:** add bind mounts (or `develop.watch`) so code changes reload without rebuilding.
8. **Signals:** write a Node server that logs on SIGTERM; compare shell-form vs exec-form CMD with `docker stop` timing.
9. **Limits:** run a container with `--memory 64m` and a script that allocates memory; observe exit 137 and `OOMKilled: true`.
10. **Security:** make the API run as non-root with `--read-only`, `--cap-drop ALL`; scan the image with Trivy or Docker Scout and fix one finding.
11. **Registry:** tag and push your image to GHCR/Docker Hub; pull it on another machine (or after deleting locally).
12. **CI:** create a GitHub Actions workflow that builds a multi-arch image with cache and pushes `sha` and semver tags.

---

## ✅ Final Revision Checklist

- [ ] Explain containers vs VMs, images vs containers
- [ ] Describe Docker architecture (CLI → dockerd → containerd → runc, registries)
- [ ] Run, inspect, log, exec, stop, remove containers fluently
- [ ] Write a cached, multi-stage, non-root Dockerfile with `.dockerignore`
- [ ] Explain CMD vs ENTRYPOINT, exec vs shell form, ARG vs ENV, COPY vs ADD
- [ ] Choose between volumes, bind mounts and tmpfs
- [ ] Connect containers via user-defined networks; publish ports correctly
- [ ] Write a Compose file with healthchecks, volumes, env and overrides
- [ ] Tag, push and pull images; avoid `latest` in production
- [ ] Apply resource limits, restart policies, healthchecks, log rotation
- [ ] Apply container security best practices
- [ ] Troubleshoot exit codes 137/127, port conflicts, localhost issues, arch mismatches
- [ ] Explain namespaces, cgroups, OverlayFS, copy-on-write, PID 1 signals
- [ ] Map Compose concepts to Kubernetes

> **Golden habits:** one process per container · small pinned base images · multi-stage builds · `.dockerignore` always · non-root · exec-form CMD · data in volumes, config in env, secrets in a secret store · log to stdout · tag with versions/SHAs, never deploy `latest` · `docker compose config` and `docker logs` are your first debugging tools.
