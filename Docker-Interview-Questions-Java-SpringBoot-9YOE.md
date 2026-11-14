# Docker Interview Prep — Java / Spring Boot Developer (9 Years Experience)

> **How to use this file:** Sections 1–4 are the fundamentals you must answer instantly. Sections 5–11 are where senior candidates are actually separated from mid-level ones (JVM-in-container, resource limits, security, networking). Section 12 is scenario-based — this is what a 9-YOE interview is 60% made of. Section 13 is last-minute rapid revision.
>
> **Memory strategy used throughout:** each answer has a **one-line hook** (memorize this), then the detail (understand this). If you can say the hook and give one concrete example, you sound senior.

---

## Table of Contents

1. [Core Concepts & Architecture](#1-core-concepts--architecture)
2. [Images, Layers & Registries](#2-images-layers--registries)
3. [Dockerfile Mastery](#3-dockerfile-mastery)
4. [Container Lifecycle & Commands](#4-container-lifecycle--commands)
5. [Java & Spring Boot Specific Docker](#5-java--spring-boot-specific-docker)
6. [Multi-Stage Builds & Image Optimization](#6-multi-stage-builds--image-optimization)
7. [Volumes & Data Persistence](#7-volumes--data-persistence)
8. [Networking](#8-networking)
9. [Docker Compose](#9-docker-compose)
10. [Resources, Limits & Performance](#10-resources-limits--performance)
11. [Security & Production Readiness](#11-security--production-readiness)
12. [Scenario-Based Questions (Senior Level)](#12-scenario-based-questions-senior-level)
13. [Rapid-Fire Revision & Cheat Sheets](#13-rapid-fire-revision--cheat-sheets)

---

## 1. Core Concepts & Architecture

### Q1. What is Docker and what problem does it solve?

**Hook:** *"It ships the environment with the code, so 'works on my machine' becomes 'works everywhere'."*

Docker is a containerization platform that packages an application together with its runtime, libraries, and configuration into a single immutable artifact called an **image**. That image runs as a **container** — an isolated process on the host OS.

The problems it solves for a Spring Boot team:

- **Environment drift** — dev has JDK 17, QA has JDK 11, prod has a different locale/timezone. The image freezes all of it.
- **Onboarding time** — a new developer runs `docker compose up` and gets the app + Postgres + Redis + Kafka instead of a two-day setup document.
- **Dependency conflicts** — two services needing different JDK versions coexist on one host.
- **Density and cost** — containers share the host kernel, so you run far more per VM than with full VMs.
- **Deployment consistency** — the same image artifact that passed QA is the one promoted to prod. Nothing is rebuilt between environments.

### Q2. Container vs Virtual Machine — explain the real difference

**Hook:** *"VM virtualizes hardware; container virtualizes the OS. VM = own kernel, container = shared kernel."*

| Aspect | Virtual Machine | Container |
|---|---|---|
| Isolation unit | Full guest OS + kernel | Process with namespaces |
| Boot time | Minutes | Milliseconds to seconds |
| Size | GBs (full OS) | MBs (app + libs) |
| Kernel | Its own | Shares the host kernel |
| Isolation strength | Stronger (hypervisor boundary) | Weaker (kernel is shared) |
| Overhead | Hypervisor + OS per instance | Almost none |

**Senior nuance to add:** because the kernel is shared, a container **cannot** run a different kernel than the host. That is why Linux containers on Windows/macOS actually run inside a hidden Linux VM (Docker Desktop uses a LinuxKit/HyperKit/WSL2 VM). It also means a kernel-level exploit is a bigger deal for containers than for VMs — which is why we still isolate untrusted multi-tenant workloads with VMs or gVisor/Kata, not plain containers.

### Q3. Which Linux primitives make containers possible?

**Hook:** *"Namespaces = what you can see. Cgroups = how much you can use. Union FS = what your disk looks like."*

- **Namespaces (isolation/visibility):**
  - `pid` — container sees its own process tree, your Java app is PID 1
  - `net` — own network stack, interfaces, ports, routing table
  - `mnt` — own filesystem mount points
  - `uts` — own hostname
  - `ipc` — own shared memory / semaphores
  - `user` — UID mapping (host root ≠ container root when enabled)
- **Control groups (cgroups)** — enforce CPU, memory, PIDs, block I/O limits. This is what `--memory` and `--cpus` configure, and what the JVM reads to size the heap.
- **Union filesystem (OverlayFS)** — stacks read-only image layers plus one writable container layer, giving copy-on-write behaviour.
- **Capabilities / seccomp / AppArmor-SELinux** — drop root privileges to a fine-grained set and filter syscalls.

**Why these exist / how they fit together:** a "container" is not a real kernel object — it is an ordinary Linux process that the kernel has been told to (a) show a restricted view of the system (namespaces), (b) cap its resource usage (cgroups), and (c) give it a layered filesystem (OverlayFS). Docker just automates setting these up. Without them, every process would see every other process, every port and the whole disk, and one runaway JVM could eat all the host RAM.

You can see this yourself on a Linux host:

```bash
docker run -d --name demo eclipse-temurin:21-jre sleep 3600
docker inspect -f '{{.State.Pid}}' demo      # e.g. 4211 — the container is just a host PID
ps -fp 4211                                   # visible from the host like any other process
docker exec demo ps -ef                       # inside, the same process is PID 1
```

Follow-up interviewers like: *"Why can't two containers see each other's processes?"* → separate `pid` namespaces. *"Why did my container get killed at 512 MB?"* → the cgroup memory limit.

### Q4. Explain the Docker architecture and what happens on `docker run`

**Hook:** *"CLI → daemon (dockerd) → containerd → runc → your process."*

Components:

- **Docker CLI** — sends REST calls over a Unix socket (`/var/run/docker.sock`).
- **dockerd (daemon)** — builds images, manages networks/volumes, pulls from registries.
- **containerd** — the container lifecycle supervisor.
- **runc** — the low-level OCI runtime that actually creates namespaces/cgroups and `exec`s your process.
- **Registry** — Docker Hub, ECR, ACR, GCR, Nexus, Harbor.

`docker run -d -p 8080:8080 myapp:1.4` step by step:

1. CLI POSTs to the daemon.
2. Daemon looks for `myapp:1.4` locally; if missing, pulls the manifest and missing layers from the registry.
3. Creates a container: new writable layer on top of the image layers, new namespaces, cgroup limits applied.
4. Sets up networking: attaches to the `bridge` network, gets an IP, and the daemon installs an iptables DNAT rule for the `8080:8080` publish.
5. runc starts the entrypoint process as PID 1 inside the container.
6. Container lives exactly as long as PID 1 lives.

**Point worth making in interview:** `/var/run/docker.sock` access is effectively root on the host. Never mount it into an application container casually — that's a classic finding in security reviews.

### Q5. Image vs Container vs Registry vs Repository vs Tag

**Hook:** *"Image = class, container = object, tag = a movable label, digest = the truth."*

- **Image** — immutable, read-only, layered template.
- **Container** — a running (or stopped) instance of an image with a writable layer.
- **Registry** — the server that stores images (Docker Hub, ECR).
- **Repository** — a named collection of related images inside a registry (`mycompany/payment-service`).
- **Tag** — a human label for one image inside a repository (`1.4.2`, `latest`). **Tags are mutable** — someone can re-push `1.4.2`.
- **Digest** — `sha256:...` content hash, immutable. In production, deploy by digest or by an immutable version tag, never `latest`.

**Putting the vocabulary together** — a full image reference reads left to right as `registry / repository : tag`, optionally pinned with `@digest`:

```text
123456789.dkr.ecr.ap-south-1.amazonaws.com/payment-service:1.4.2
└──────────── registry ──────────────────┘└─ repository ─┘└ tag ┘

payment-service@sha256:9f86d0...        # same image addressed by content hash
```

Why the tag-vs-digest distinction matters: a tag is just a pointer stored in the registry. If CI re-pushes `1.4.2`, the pointer moves and a node that pulls later gets different bytes. The digest is computed from the image content, so it can never point to anything else. Typical follow-up: *"How do you find the digest of what is running?"* → `docker inspect --format '{{index .RepoDigests 0}}' myapp:1.4.2`.

### Q6. Is a container stateless? Where does state go?

Containers themselves are **ephemeral**. The writable layer dies with `docker rm`. Anything that must survive goes to:

- a **named volume** (Docker-managed) or **bind mount** (host path),
- an external database / object store / cache,
- or, for logs, stdout/stderr collected by a log driver.

For Spring Boot this means: no local session state (use Redis or JWT), no local file uploads (use S3), no local H2 file DB in prod, and no writing logs to a file inside the container.

**Why it matters:** orchestrators (Kubernetes, ECS, Swarm) freely kill, reschedule and scale containers — a new replica starts from the clean image, not from where the old one left off. If a user's `HttpSession` lived in memory of replica A and the load balancer sends the next request to replica B, the user is logged out. If a file was uploaded to replica A's disk, replica B can't serve it. Designing the app to be **stateless** (state lives in a shared external store) is what makes replicas interchangeable and disposable.

You can prove the ephemerality in two commands:

```bash
docker run --name t1 alpine sh -c "echo hello > /data.txt"
docker rm t1 && docker run --rm alpine cat /data.txt     # No such file — the writable layer was deleted with t1
```

**Pitfall:** "stateless container" does not mean "no state anywhere" — a database container is legitimately stateful, but its state must sit on a volume (Q39–Q40), not in the writable layer.

---

## 2. Images, Layers & Registries

### Q7. What is a Docker image layer and why does it matter?

**Hook:** *"Every layer is a diff, layers are cached and shared, and nothing is ever truly deleted from a lower layer."*

Each instruction that changes the filesystem (`RUN`, `COPY`, `ADD`) produces a layer identified by a content hash. Layers are:

- **Cached** — an unchanged instruction with unchanged inputs is reused, making rebuilds fast.
- **Shared** — ten Spring Boot services on the same `eclipse-temurin:21-jre` base share those base layers once on disk and pull them once.
- **Additive-only** — if you `COPY` a secret in layer 3 and `RUN rm` it in layer 4, the secret **is still in the image** and extractable with `docker save`/`docker history`. This is the single most important consequence of the layer model and a very common interview follow-up.

### Q8. How do you inspect what's inside an image?

```bash
docker history --no-trunc myapp:1.4     # layers + the command that created each
docker image inspect myapp:1.4          # env, entrypoint, labels, layers, digest
docker save myapp:1.4 -o app.tar        # extract and audit the raw layers
docker run --rm -it --entrypoint sh myapp:1.4   # poke around interactively
docker diff <container-id>              # what the running container changed (A/C/D)
```

`dive` (third-party) is the tool to name if asked how you audit wasted space per layer.

**What each command tells you, and when to reach for it:**

- `docker history` — lists every layer with the Dockerfile instruction that produced it and its size. Use it to spot a fat `COPY` or `RUN` layer. `--no-trunc` shows the full command, which is also how you discover that someone passed a credential via `ARG` or `RUN` (see Q18/S16).
- `docker image inspect` — returns JSON metadata: the default `ENTRYPOINT`/`CMD`, `ENV`, exposed ports, labels, architecture, layer hashes and `RepoDigests`. First stop when an image "behaves differently" than expected.
- `docker save` — writes the image as a tar of layer tarballs, so you can unpack it (`tar -xf app.tar`) and look for files that a later `RUN rm` supposedly deleted.
- `--entrypoint sh` — overrides the Java entrypoint so you land in a shell instead of starting the app; essential when the app crashes on startup and you need to look around. (Won't work on distroless images — there is no shell; see S25.)
- `docker diff` — compares a *running or stopped container* with its image and prints `A` (added), `C` (changed), `D` (deleted) paths. Handy to find an app that unexpectedly writes into the container filesystem and therefore breaks under `--read-only`.

### Q9. `latest` tag — why is it dangerous?

**Hook:** *"`latest` is just a default string, not a promise of newness."*

Problems: it is mutable, so two hosts pulling "the same" `latest` can get different bits; rollbacks are impossible because you don't know what the previous `latest` was; and combined with `imagePullPolicy: IfNotPresent`-style caching, a node may silently run a stale build.

Senior answer on what to do instead: tag with an immutable, traceable identifier — `payment-service:1.4.2`, `payment-service:1.4.2-b842-a3f9c1d` (version + build number + git SHA) — and record the digest in the deployment manifest. Keep `latest` only as a convenience pointer for local dev.

### Q10. How do multi-arch images work, and why should you care?

**Hook:** *"One tag, a manifest list, many architectures — because your laptop is arm64 and prod is amd64."*

A tag can point to a **manifest list (OCI index)** containing per-platform manifests (`linux/amd64`, `linux/arm64`). The daemon picks the entry matching the host. Build with BuildKit:

```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.4.2 --push .
```

This is a very common real problem on Apple Silicon: an image built on an M-series Mac is arm64 and fails on amd64 EKS nodes with `exec format error`. Fix: buildx multi-arch, or build in CI on the target platform.

### Q11. Difference between `docker save`/`load` and `docker export`/`import`?

| Command pair | Operates on | Preserves |
|---|---|---|
| `save` / `load` | **Images** | All layers, tags, history, metadata |
| `export` / `import` | **Containers** | Flattened filesystem only — loses history, ENTRYPOINT, ENV |

Use `save`/`load` for moving images to an air-gapped environment. `export` is for filesystem forensics, and you must re-specify the entrypoint after `import`.

**Why the difference exists:** an image is a stack of layers plus a JSON config (entrypoint, env, ports, labels). `save` writes *all of that*, so `load` on another machine recreates the identical image (same layers, same digest-able content). `export` takes a *container*, merges its layers into one flat filesystem tarball and throws the config away — you get the files and nothing else.

```bash
# Air-gapped transfer: build here, run there
docker save -o payment-1.4.2.tar payment-service:1.4.2
# ...copy the tar over USB / secure file share...
docker load -i payment-1.4.2.tar
docker run -d payment-service:1.4.2        # ENTRYPOINT and ENV still intact

# export/import: flat copy, config lost
docker export app > app-fs.tar
docker import app-fs.tar app-flat:1        # one layer, no ENTRYPOINT — docker run needs an explicit command
```

**Pitfall:** `export` does not include the contents of volumes, only the container filesystem — so it is not a backup tool for data (see Q41).

### Q12. How do you clean up disk space safely?

```bash
docker system df                    # where the space actually went
docker image prune                  # dangling (untagged) images
docker image prune -a               # all images not used by a container — careful
docker container prune              # stopped containers
docker volume prune                 # unused volumes — DANGER: this is your data
docker builder prune                # BuildKit build cache (often the biggest offender)
docker system prune -a --volumes    # nuclear; never run casually on a shared/prod host
```

**Say this:** on build agents the BuildKit cache is usually what fills the disk, and I gate cleanup with `--filter "until=168h"` rather than blanket pruning, and never prune volumes without checking `docker volume ls` against known data volumes.

**Terms explained:** a *dangling* image is one with no tag (shown as `<none>`) — typically the old build left behind when you rebuild `myapp:latest`. `image prune` removes only those; `image prune -a` removes every image no container currently uses, so the next deploy has to re-pull. The *BuildKit cache* holds intermediate build results (downloaded Maven artifacts, compiled layers) — valuable for speed, but unbounded on a busy CI agent.

Safe routine on a CI agent (example cron job):

```bash
docker builder prune --filter "until=168h" -f      # drop cache older than 7 days
docker image prune -a --filter "until=168h" -f     # drop unused images older than 7 days
docker system df                                    # confirm reclaimed space
```

Pitfall: `docker volume prune` deletes volumes not referenced by any container (a stopped container still counts as a reference) — so a database volume whose container was removed counts as "unused" and is erased, and `-f` skips even the confirmation prompt. Newer Docker Engines (23+) prune only *anonymous* volumes by default and need `--all` for named ones, but older engines remove both, so don't rely on that.

---

## 3. Dockerfile Mastery

### Q13. Walk through the important Dockerfile instructions

| Instruction | Purpose | Senior notes |
|---|---|---|
| `FROM` | Base image | Pin by version or digest; `scratch` for static binaries |
| `WORKDIR` | Set working dir | Creates it; prefer over `RUN cd` (which doesn't persist) |
| `COPY` | Copy files in | Default choice; supports `--from` and `--chown` |
| `ADD` | Copy + extras | Auto-extracts tars, can fetch URLs — avoid unless you need extraction |
| `RUN` | Execute at **build** time | Each one is a layer; chain with `&&` |
| `ENV` | Env var, persists at runtime | Visible in `docker inspect` — never secrets |
| `ARG` | Build-time variable | Not present at runtime, but **is** in build history |
| `EXPOSE` | Documentation of port | Does **not** publish anything; `-p` does |
| `ENTRYPOINT` | The executable | Prefer exec form (JSON array) |
| `CMD` | Default args / default command | Overridden by `docker run` args |
| `USER` | Switch user | Always drop to non-root before the entrypoint |
| `HEALTHCHECK` | Liveness probe | Point at Spring Actuator |
| `VOLUME` | Declare mount point | Creates anonymous volumes — use sparingly |
| `LABEL` | Metadata | Store git SHA, build time, owner (OCI labels) |
| `STOPSIGNAL` | Signal on stop | Default SIGTERM, which Spring Boot handles |

**How they read together — a tiny annotated Dockerfile:**

```dockerfile
FROM eclipse-temurin:21-jre            # 1. start from a base image (the "parent")
WORKDIR /app                           # 2. all later relative paths are under /app
COPY target/app.jar app.jar            # 3. copy the build output from build context into the image
RUN useradd -r -u 10001 spring         # 4. runs while BUILDING the image; result is saved as a layer
USER 10001                             # 5. processes at runtime run as this non-root user
EXPOSE 8080                            # 6. documentation only — does not open the port
ENTRYPOINT ["java","-jar","app.jar"]   # 7. runs when the CONTAINER starts
```

The key mental split is **build time vs run time**: `RUN`, `COPY`, `ARG` act while building the image; `ENTRYPOINT`, `CMD`, `ENV`, `USER`, `HEALTHCHECK` describe how a container behaves when started. A common confusion: `RUN java -jar app.jar` would execute the app *during the build* (and hang the build), whereas `ENTRYPOINT` runs it when you `docker run`.

### Q14. `CMD` vs `ENTRYPOINT` — the classic

**Hook:** *"ENTRYPOINT = what always runs. CMD = the arguments you can override."*

```dockerfile
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
CMD ["--spring.profiles.active=prod"]
```

- `docker run myapp` → `java -jar /app/app.jar --spring.profiles.active=prod`
- `docker run myapp --spring.profiles.active=qa` → CMD replaced, entrypoint kept
- To replace the entrypoint you must pass `--entrypoint`

**Exec form vs shell form** — this is the part interviewers care about:

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]      # exec form: java IS PID 1, receives SIGTERM
ENTRYPOINT java -jar app.jar                # shell form: /bin/sh -c is PID 1, swallows SIGTERM
```

Shell form breaks graceful shutdown: `sh` doesn't forward SIGTERM to the JVM, so Docker waits 10s and SIGKILLs — in-flight HTTP requests are cut and `@PreDestroy` never runs. **Always use exec form for Java.**

### Q15. `COPY` vs `ADD`?

`COPY` copies files/dirs, nothing more. `ADD` additionally auto-extracts local tar archives and can download from a URL. Use `COPY` by default because `ADD`'s magic is surprising (an accidental tar gets extracted) and URL fetching bloats layers and skips checksum verification. Use `ADD` deliberately only to extract a local tarball.

```dockerfile
COPY target/app.jar /app/app.jar          # exactly what it says — safe and predictable
ADD  rootfs.tar.gz /                      # auto-extracts into / — the one legitimate use
ADD  https://example.com/tool.tgz /opt/   # downloads but does NOT extract remote archives, no checksum check, adds a layer
```

Note the asymmetry that trips people up: `ADD` auto-extracts only **local** tar archives, not ones fetched from a URL. For remote files the better pattern is a single `RUN curl -fsSL ... | sha256sum -c && tar ...` (so you verify integrity and avoid leaving the archive in a layer), or `ADD --checksum=sha256:... <url>` with BuildKit. Also remember `COPY --chown=10001:10001` and `COPY --from=<stage>` are available on `COPY`, which is why it is the workhorse in multi-stage builds.

### Q16. What is `.dockerignore` and why does it matter for Maven/Gradle projects?

**Hook:** *"It shrinks the build context, protects secrets, and stops cache invalidation."*

Everything in the build directory is tarred and sent to the daemon before the build starts. Without `.dockerignore`, a Java repo ships `target/`, `.git/`, IDE folders, and possibly `.env` files.

```gitignore
.git
.gitignore
target/
build/
!target/*.jar          # if you build outside Docker and only need the jar
*.iml
.idea/
.vscode/
Dockerfile
docker-compose*.yml
README.md
**/*.log
.env
*.pem
*.p12
```

Extra benefit: a changing `.git` directory otherwise invalidates the cache on every commit.

### Q17. How does build cache invalidation work and how do you exploit it for Spring Boot?

**Hook:** *"One layer changes → every layer after it rebuilds. So put slow-and-stable first, fast-and-volatile last."*

Cache is keyed on the instruction plus, for `COPY`/`ADD`, a checksum of the copied content.

The classic Maven mistake:

```dockerfile
COPY . .
RUN mvn package          # any source change re-downloads all dependencies
```

The correct dependency-caching pattern:

```dockerfile
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
RUN ./mvnw dependency:go-offline -B      # cached until pom.xml changes
COPY src/ ./src/
RUN ./mvnw package -DskipTests -B        # only this reruns on a code change
```

With BuildKit you can go further with a persistent cache mount, which survives even a `pom.xml` change:

```dockerfile
RUN --mount=type=cache,target=/root/.m2 ./mvnw package -DskipTests -B
```

For Gradle: `--mount=type=cache,target=/root/.gradle`.

### Q18. `ARG` vs `ENV`?

| | `ARG` | `ENV` |
|---|---|---|
| Available at build time | Yes | Yes |
| Available at runtime | **No** | Yes |
| Set from CLI | `--build-arg` | `-e` / `--env-file` at run |
| Visible in image inspect | Only in build history | Yes |

```dockerfile
ARG JDK_VERSION=21
FROM eclipse-temurin:${JDK_VERSION}-jre
ENV SPRING_PROFILES_ACTIVE=prod
```

**Neither is safe for secrets.** `ARG` values appear in `docker history`; `ENV` values appear in `docker inspect`. Use BuildKit secret mounts at build time and a secret manager at runtime.

### Q19. How do you pass a secret at build time correctly?

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=m2settings,target=/root/.m2/settings.xml \
    ./mvnw package -DskipTests -B
```

```bash
docker build --secret id=m2settings,src=$HOME/.m2/settings.xml -t myapp .
```

The file is mounted only for that `RUN` and is never written to a layer. This is the answer for "how do you pull from a private Nexus during the Docker build without leaking credentials".

**Why the naive ways fail:**

```dockerfile
COPY settings.xml /root/.m2/settings.xml     # BAD: baked into a layer forever
RUN ./mvnw package
RUN rm /root/.m2/settings.xml                # BAD: file is hidden in the top layer but still in the COPY layer
ARG NEXUS_PASSWORD                           # BAD: shows up in `docker history`
```

Layers are immutable diffs (Q7), so anything ever written into one can be recovered by whoever can pull the image. A **secret mount** avoids this: BuildKit makes the file available at the target path (a tmpfs) only while that single `RUN` executes, and nothing is committed to the layer. The `id` is just a label that links the Dockerfile's `--mount` to the `--secret` given on the command line.

Verify it worked: `docker history --no-trunc myapp` shows the `RUN` command but no credentials, and `docker run --rm myapp cat /root/.m2/settings.xml` fails with "No such file". In CI, feed the secret from the pipeline's secret store (`src=` can be a file you create from an env var for that job). Related variant: `--mount=type=ssh` forwards your SSH agent to clone private Git repos without copying a key.

### Q20. What is BuildKit and why enable it?

Default builder in modern Docker. Gains: parallel execution of independent stages, cache mounts (`--mount=type=cache`), secret and SSH mounts, better cache import/export for CI (`--cache-from`/`--cache-to` against a registry), and multi-arch via buildx. Enable legacy setups with `DOCKER_BUILDKIT=1`.

**What it is and why it matters:** BuildKit is the newer build engine behind `docker build`. The classic builder ran Dockerfile instructions strictly one after another and always built every stage; BuildKit turns the Dockerfile into a dependency graph, so it can:

- **Skip unused stages** — in a multi-stage file it only builds stages the target actually needs.
- **Build independent stages in parallel** — e.g. a `build` stage and a `test` stage that don't depend on each other.
- **Keep secrets out of layers** — `--mount=type=secret` (Q19) exposes a file only for one `RUN`.
- **Persist caches between builds** — `--mount=type=cache,target=/root/.m2` keeps the Maven repository across builds even when `pom.xml` changes (Q17).
- **Share cache across CI agents** — `--cache-to type=registry` pushes the cache to a registry so a brand-new agent starts warm (S3).

Interview tip: in CI the common mistake is running with the legacy builder on an ephemeral agent, so every build starts with an empty cache and re-downloads the world.

---

## 4. Container Lifecycle & Commands

### Q21. Container states and transitions

`created → running → paused → restarting → exited → dead`

- `docker create` → created, `docker start` → running
- `docker pause` uses the cgroup freezer (processes frozen, memory retained)
- `docker stop` = SIGTERM, wait `--time` (default 10s), then SIGKILL
- `docker kill` = SIGKILL immediately (no graceful shutdown — never for a live service)
- `docker restart` = stop + start
- `docker rm` deletes the container and its writable layer

**Plain-language walk-through:** `created` means Docker has built the container (filesystem, network config) but not started the process. `running` means PID 1 is alive. `paused` freezes every process in place (they stay in memory, just get no CPU) — useful for taking a consistent snapshot, rarely used in practice. `exited` means PID 1 ended — either normally or by crash/kill; the container still exists with its logs and writable layer, which is why `docker ps -a` and `docker logs` still work on it. `restarting` appears while a restart policy is bringing a crashed container back up (a container stuck here is a **crash loop**). `dead` is a rare failed-removal state.

```bash
docker create --name t eclipse-temurin:21-jre sleep 300   # created
docker start t                                             # running
docker pause t && docker unpause t                         # paused -> running
docker stop t                                              # SIGTERM, then SIGKILL after 10s -> exited
docker rm t                                                # gone, writable layer deleted
```

Interview probe: *"My container shows `Exited (0)` — is that a problem?"* For a batch job no; for a web service it means PID 1 finished on its own (see Q24).

### Q22. `docker stop` vs `docker kill` — and why a 9-YOE dev must care

`stop` sends SIGTERM first, which is what makes graceful shutdown possible: Spring Boot stops accepting new connections, lets in-flight requests finish, closes the datasource pool, and runs `@PreDestroy`. `kill` sends SIGKILL — the process is terminated with no cleanup, so requests are dropped mid-flight and DB connections are left for the server to time out.

To make this actually work you need three things:

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=25s
```
```dockerfile
ENTRYPOINT ["java","-jar","/app/app.jar"]   # exec form so the JVM is PID 1
```
```bash
docker stop --time=30 myapp                  # grace period > app's shutdown timeout
```

### Q23. Essential troubleshooting commands

```bash
docker ps -a                            # all containers + exit codes
docker logs -f --tail 200 --timestamps app
docker logs --since 10m app
docker exec -it app sh                  # shell into a running container
docker exec -it app jcmd 1 VM.flags     # JVM flags actually in effect
docker inspect app                      # everything: mounts, env, network, limits
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' app
docker stats                            # live CPU / mem / net / IO
docker top app                          # processes inside
docker port app                         # published port mappings
docker events --since 1h                # daemon-level timeline (OOM, die, restart)
docker cp app:/tmp/heap.hprof ./        # pull a heap dump out
```

**How to use these as a sequence (not a random list):**

1. *"What state is it in?"* → `docker ps -a` shows running vs exited and the exit code (137 = killed/OOM, 143 = graceful stop; see Q24).
2. *"What did the app say?"* → `docker logs` — `-f` follows live, `--tail 200` limits to the last 200 lines, `--since 10m` limits by time. For a crashed container this is nearly always where the real error (stack trace, missing property) is.
3. *"What config did it really get?"* → `docker inspect` dumps JSON; the `-f` Go-template flag extracts a single field, e.g. `docker inspect -f '{{.State.OOMKilled}}' app` → `true`/`false`. Check `Env`, `Mounts`, `HostConfig.Memory`, `NetworkSettings`.
4. *"Is it healthy right now?"* → `docker stats` (live CPU %, memory vs limit, network and disk I/O) and `docker top` (processes inside, seen from the host).
5. *"Can I look inside?"* → `docker exec -it app sh` starts a *second* process in the same container; `jcmd 1 ...` talks to the JVM, which is PID 1. (No shell on distroless — see S25.)
6. *"What happened in the past hour?"* → `docker events` streams daemon events such as `oom`, `die`, `restart`, `health_status`, which is how you prove a container was OOM-killed or restarted by its health check.

### Q24. Why did my container exit immediately?

**Hook:** *"A container lives only as long as PID 1. No foreground process, no container."*

Checklist: (1) `docker logs` first — usually a stack trace or missing config; (2) `docker ps -a` for the exit code; (3) wrong `ENTRYPOINT`/jar path; (4) the process daemonized itself into the background; (5) the app crashed on a missing env var / unreachable DB at startup.

**Why this happens:** Docker watches only PID 1. When PID 1 returns, the container stops — Docker has no idea about anything else you started. Typical cases:

- *Wrong path / typo:* `ENTRYPOINT ["java","-jar","/app/app.jar"]` but the jar was copied to `/app/service.jar` → `Error: Unable to access jarfile` and exit 1.
- *Daemonizing:* a script ends with `service tomcat start` (or `nginx` without `-g 'daemon off;'`). The command forks into the background and returns immediately, so PID 1 exits and the container stops. Services must run in the **foreground**.
- *Startup failure:* Spring Boot can't bind the datasource → `APPLICATION FAILED TO START` → JVM exits 1. The log shows it, the container doesn't stay up to inspect — so use `docker logs <id>` on the exited container.
- *Missing shell:* `exec: "sh": executable file not found in $PATH` on a distroless image that you gave a shell-form entrypoint or `sh -c` wrapper.

Debug trick when logs aren't enough: start it again with the entrypoint replaced so it stays alive — `docker run --rm -it --entrypoint sh myapp:1.4` — then run `java -jar /app/app.jar` by hand and watch it fail.

Exit codes worth memorizing:

| Code | Meaning |
|---|---|
| 0 | Clean exit (for a service, still a bug — why did it finish?) |
| 1 | Application error / uncaught exception |
| 125 | Docker daemon itself failed (bad flag) |
| 126 | Command found but not executable (permissions) |
| 127 | Command not found (typo, or missing shell in a distroless/slim image) |
| 137 | SIGKILL → **almost always OOMKilled**, or a stop that exceeded the grace period |
| 139 | SIGSEGV — segfault (native lib) |
| 143 | SIGTERM — graceful stop, normal for a deliberate `docker stop` |

### Q25. `docker run` flags a senior should know cold

```bash
docker run -d --name payment \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  --env-file ./prod.env \
  -m 1g --memory-swap 1g --cpus 1.5 \
  --restart unless-stopped \
  --read-only --tmpfs /tmp \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --user 10001:10001 \
  --network app-net \
  -v app-logs:/var/log/app \
  --health-cmd 'curl -f http://localhost:8080/actuator/health/liveness || exit 1' \
  --health-interval 15s --health-timeout 3s --health-retries 3 --health-start-period 60s \
  payment-service:1.4.2
```

Restart policies: `no` (default), `on-failure[:max]`, `always` (restarts even after daemon restart), `unless-stopped` (like `always` but respects a manual stop). For services, `unless-stopped` or `on-failure:5`; `always` can mask a crash loop.

**What each flag group is for (so you can justify every line):**

| Flags | What it does | Why you'd set it |
|---|---|---|
| `-d --name` | Run in background with a stable name | Name lets you `docker logs payment` instead of using random IDs |
| `-p 8080:8080` | Publish `host:container` port | Without it the app is reachable only from other containers |
| `-e` / `--env-file` | Inject env vars (Spring relaxed binding, Q31) | Config stays out of the image |
| `-m 1g --memory-swap 1g --cpus 1.5` | Hard memory cap, no swap, CPU quota | Protects the host/neighbours; setting swap = memory disables swap so a JVM can't be paged out (Q52) |
| `--restart unless-stopped` | Auto-restart on crash/reboot | Self-healing on a single host |
| `--read-only --tmpfs /tmp` | Immutable root filesystem plus an in-memory scratch dir | Blocks an attacker from writing binaries; Tomcat still gets a writable `/tmp` |
| `--cap-drop ALL`, `no-new-privileges` | Remove Linux capabilities, forbid setuid escalation | Least privilege (Q56) |
| `--user 10001:10001` | Run as non-root | A breakout lands as an unprivileged user |
| `--network app-net` | Attach to a user-defined bridge | DNS by container name (Q44) |
| `-v app-logs:/var/log/app` | Named volume | Data that must outlive the container |
| `--health-*` | Docker-level health probe | Marks container `healthy`/`unhealthy`; `start-period` gives the JVM time to boot (Q32) |

Pitfall: `--read-only` will crash an app that writes anywhere else (e.g. `/app/logs`, `/dumps`); give each such path its own `--tmpfs` or volume, and use `docker diff` (Q8) to discover them.

---

## 5. Java & Spring Boot Specific Docker

### Q26. Which base image do you choose for a Spring Boot app and why?

**Hook:** *"JRE not JDK, Temurin not oraclejdk, glibc not Alpine unless you've tested it, distroless when security asks."*

| Base image | Size | When to use |
|---|---|---|
| `eclipse-temurin:21-jdk` | ~450 MB | **Build stage only** |
| `eclipse-temurin:21-jre` | ~270 MB | Safe default for runtime |
| `eclipse-temurin:21-jre-alpine` | ~170 MB | Size-critical; musl libc, test carefully |
| `gcr.io/distroless/java21-debian12` | ~190 MB | Production hardening — no shell, no package manager |
| Custom `jlink` runtime on `debian-slim` | ~90–120 MB | Very tight images, needs module analysis |

Reasoning to give out loud:

- **JRE for runtime** — you don't need javac, jar, or the compiler tools in prod; it's smaller and a smaller attack surface.
- **Temurin/Amazon Corretto/Liberica** — free, TCK-certified, long-term-supported builds. Oracle JDK has licensing implications.
- **Alpine caution** — Alpine uses **musl** instead of glibc. Most Java works, but native dependencies (Netty native transports, some crypto/JNI libraries, older JDBC natives) and DNS resolution behaviour can differ, and historically there were latency issues under musl's allocator. I use `-jre` on a slim glibc base by default and only go Alpine after load testing.
- **Distroless** — no shell means an RCE has far fewer tools available, but it also means no `docker exec sh` for debugging. Mitigate with a `:debug` variant for triage and rely on Actuator + JMX + remote heap dumps.
- **Always pin** — `eclipse-temurin:21.0.4_7-jre` or a digest, so a base image change is a deliberate, reviewed commit.

### Q27. Does the JVM respect container memory limits? (Very common senior question)

**Hook:** *"Modern JVMs are container-aware and read cgroups, but the default max heap is only ~25% of the limit — so you must tune it."*

Historically (Java 8 pre-`8u191`) the JVM read the **host's** total RAM. Give a container 512 MB on a 64 GB host and the JVM sized its heap to ~16 GB, then got OOMKilled the moment it grew. That is the root cause of countless exit-137 incidents.

Today, `UseContainerSupport` is on by default (Java 10+, backported to 8u191+). The JVM reads cgroup limits and sets `MaxHeapSize` to **25% of the container limit** by default — wasteful, because in a container there is nothing else to protect.

Correct configuration:

```dockerfile
ENV JAVA_OPTS="-XX:InitialRAMPercentage=50.0 \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+UseG1GC \
               -XX:+ExitOnOutOfMemoryError \
               -XX:+HeapDumpOnOutOfMemoryError \
               -XX:HeapDumpPath=/dumps"
ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS -jar /app/app.jar"]
```

> Note the `exec` inside `sh -c` — that replaces the shell with the JVM so the JVM still becomes PID 1 and receives SIGTERM. This is the standard way to keep signal handling correct while still expanding `$JAVA_OPTS`.

Percentages beat `-Xmx512m` because the same image works on a 1 GB and a 4 GB deployment without a rebuild.

Verify inside the container:

```bash
docker exec -it app java -XX:+PrintFlagsFinal -version | grep -Ei 'maxheapsize|activeprocessor'
docker exec -it app jcmd 1 VM.flags
```

### Q28. Why isn't 75% enough? Explain total JVM memory footprint.

**Hook:** *"Container RAM = heap + metaspace + code cache + threads×stack + direct buffers + GC structures + native libs."*

Non-heap consumers people forget:

- **Metaspace** — class metadata; a big Spring app with lots of proxies can use 100–250 MB. Cap it: `-XX:MaxMetaspaceSize=256m`.
- **Thread stacks** — 1 MB default × 200 Tomcat threads = 200 MB of virtual, and real usage grows with depth. Reduce pool size in containers.
- **Code cache / JIT** — 50–250 MB.
- **Direct byte buffers / Netty** — bounded by `-XX:MaxDirectMemorySize`; unbounded leaks here are a classic exit-137 cause with WebClient/Netty.
- **GC overhead + native (JDBC drivers, compression, TLS)**.

So a 1 GB container with `MaxRAMPercentage=75` leaves 250 MB for all of that, which is usually fine for a typical service but tight for high-thread, Netty-heavy, or many-classes apps. Diagnose with **Native Memory Tracking**:

```bash
java -XX:NativeMemoryTracking=summary ...
docker exec -it app jcmd 1 VM.native_memory summary
```

### Q29. What is `ActiveProcessorCount` and why does it matter?

The JVM derives thread pool sizes from available processors: G1 GC threads, JIT compiler threads, ForkJoinPool.commonPool, Tomcat defaults, and reactive schedulers. With CPU limits expressed as CFS quota (`--cpus=0.5`), the JVM may still see all host cores and over-provision threads → heavy context switching and CPU throttling.

Modern JVMs compute it from the quota, but when using `cpu-shares` only, or on older JVMs, set it explicitly:

```
-XX:ActiveProcessorCount=2
```

**Related senior point:** never give a JVM a fractional CPU like `0.3` for a latency-sensitive service. GC needs bursts of CPU; a tight CFS quota causes throttling that shows up as periodic latency spikes with no obvious cause. Prefer `cpu requests` with generous or no hard limit, or at minimum `--cpus 1` or more.

### Q30. What are Spring Boot layered jars and how do you use them in Docker?

**Hook:** *"A fat jar is one 60 MB layer that changes on every commit. Layertools splits it so only your ~2 MB of classes change."*

A standard fat jar means every rebuild pushes and pulls the whole jar even though only your classes changed. Spring Boot 2.3+ supports layer extraction into four layers ordered from most stable to most volatile:

1. `dependencies` — release dependencies (rarely change)
2. `spring-boot-loader` — the loader classes
3. `snapshot-dependencies` — SNAPSHOT deps
4. `application` — your classes and resources (change every build)

```dockerfile
# ---------- build ----------
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B
COPY src/ ./src/
RUN ./mvnw clean package -DskipTests -B

# ---------- extract layers ----------
FROM eclipse-temurin:21-jre AS extract
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# ---------- runtime ----------
FROM eclipse-temurin:21-jre
RUN groupadd -r spring && useradd -r -g spring -u 10001 spring
WORKDIR /app
COPY --from=extract /app/dependencies/ ./
COPY --from=extract /app/spring-boot-loader/ ./
COPY --from=extract /app/snapshot-dependencies/ ./
COPY --from=extract /app/application/ ./
USER 10001
EXPOSE 8080
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","org.springframework.boot.loader.launch.JarLauncher"]
```

> **Version gotcha:** the launcher class moved in Spring Boot 3.2+ to `org.springframework.boot.loader.launch.JarLauncher`. Before 3.2 it was `org.springframework.boot.loader.JarLauncher`. Mentioning this detail signals real hands-on experience.

Result: a one-line code change pushes ~2 MB instead of ~60 MB, so deploys and rollbacks are dramatically faster.

### Q31. How do you inject configuration into a Spring Boot container?

**Hook:** *"Env vars via relaxed binding, ordered by Spring's property precedence — config lives outside the image."*

Spring Boot maps `SPRING_DATASOURCE_URL` → `spring.datasource.url` automatically (relaxed binding: uppercase, dots→underscores, hyphens→underscores).

```bash
docker run -d \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/orders \
  -e SPRING_DATASOURCE_USERNAME=app \
  -e SPRING_DATASOURCE_PASSWORD_FILE=/run/secrets/db_pass \
  -e MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE=health,info,prometheus \
  myapp:1.4.2
```

Precedence (later wins, simplified): packaged `application.yml` → profile-specific yml → OS env vars → command-line args. So env vars override the baked yml, which is exactly what you want.

Options ranked:

1. **Env vars** — the 12-factor default, good for non-secret config.
2. **External config mounted as a volume** — `-v /etc/app/application-prod.yml:/config/application.yml:ro` and Spring picks up `/config/` automatically (or set `SPRING_CONFIG_ADDITIONAL_LOCATION`).
3. **Spring Cloud Config / Consul** — centralized, refreshable.
4. **Secrets** — Vault, AWS Secrets Manager, or Docker/Kubernetes secrets mounted as files, read with `spring.config.import=file:/run/secrets/...` or a `*_FILE` convention.

**Never** bake environment-specific config or credentials into the image. The same artifact must be promotable dev → QA → prod.

### Q32. How do you wire a Docker HEALTHCHECK to Spring Actuator?

Split liveness and readiness so a slow dependency doesn't cause a restart loop:

```properties
management.endpoint.health.probes.enabled=true
management.endpoints.web.exposure.include=health,info,prometheus
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true
```

- `/actuator/health/liveness` → is the JVM/context alive? Restart if failing.
- `/actuator/health/readiness` → can it serve traffic (DB reachable, caches warm)? Remove from load balancer if failing, but **don't** restart.

```dockerfile
HEALTHCHECK --interval=15s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health/liveness || exit 1
```

`--start-period` is crucial for the JVM — a Spring Boot app can take 20–60s to start, and without it the container is marked unhealthy and restarted before it ever comes up. On distroless images there's no curl, so either use the JVM itself, a tiny static health binary, or rely on the orchestrator's HTTP probe instead of Docker's.

### Q33. How do you handle logging for a Spring Boot container?

**Hook:** *"Log to stdout as JSON; the platform owns collection and rotation."*

Do **not** write to a file inside the container (invisible, and it fills the writable layer or the host disk). Instead:

- Console appender only, structured JSON (Logstash encoder, or Spring Boot 3.4+ `logging.structured.format.console=ecs`).
- Include correlation/trace IDs via Micrometer Tracing so requests are traceable across services.
- Configure the log driver with rotation to protect the host:

```bash
docker run --log-driver json-file --log-opt max-size=50m --log-opt max-file=5 myapp
```

Or set it daemon-wide in `/etc/docker/daemon.json`. In production, ship to Loki/ELK/CloudWatch via the appropriate driver or a sidecar/agent.

### Q34. Jib, Cloud Native Buildpacks, or a Dockerfile — what do you use?

| Approach | Pros | Cons |
|---|---|---|
| **Dockerfile** | Full control, transparent, portable | You own optimization and base-image patching |
| **Jib** (`mvn jib:dockerBuild`) | No Docker daemon needed, reproducible, auto-layers, fast | Less control, opinionated base |
| **Buildpacks** (`mvn spring-boot:build-image`) | Zero config, auto JVM tuning, rebasable for CVE patching | Larger images, slower first build, more magic |

**How to answer:** "For a platform team with many services I like Buildpacks or Jib because they standardize and let you rebase for CVE patching without rebuilding. For a service that needs specific native libs, agents, CA certificates, or hardened distroless bases, I keep an explicit multi-stage Dockerfile — and that's what I've mostly used, because it's auditable and reviewers can read it."

### Q35. What's special about running Testcontainers in a Dockerized CI pipeline?

Testcontainers needs a Docker daemon. Options:

- **Docker socket mount** — `-v /var/run/docker.sock:/var/run/docker.sock`. Containers become **siblings** on the host, not children. Consequence: `localhost` inside your test container is not the DB container, and bind-mounting a relative path fails because the daemon resolves paths on the host. Use `Testcontainers.exposeHostPorts()` / network aliases.
- **DinD (docker:dind)** — needs privileged mode, slower, no layer cache reuse.
- **Remote/managed daemon** — Testcontainers Cloud or a dedicated builder host.

Also: use the Ryuk reaper (default) so orphaned containers get cleaned, and pin image tags in tests so a `latest` change doesn't break the build.

**Background — what Testcontainers is:** a Java library that starts throwaway Docker containers (Postgres, Kafka, Redis…) from inside your JUnit tests, so integration tests run against the real thing instead of H2 mocks. It talks to a Docker daemon via the Docker API. That's the catch: in CI your build usually already runs *inside* a container, so "where is the Docker daemon?" becomes the design question.

- **Socket mount** — the CI container shares the host's daemon. Test containers are started by the host daemon, so they're *siblings* of the CI container, not nested. Hence the gotchas: the test JVM must reach the DB via the host-mapped port (Testcontainers' `getHost()`/`getMappedPort()` handle this — don't hard-code `localhost:5432`), and bind-mount paths are resolved on the *host*, not inside the CI container.
- **DinD (Docker-in-Docker)** — a full daemon runs inside the CI container. Isolated and clean, but needs `--privileged` (a security cost) and starts with an empty image cache every run.
- **Ryuk** — a tiny sidecar container that Testcontainers starts to delete leftover containers/networks if the test JVM dies. Don't disable it unless your platform cleans up itself.

---

## 6. Multi-Stage Builds & Image Optimization

### Q36. What is a multi-stage build and why is it essential for Java?

**Hook:** *"Build with the JDK and Maven, ship only the jar on a JRE — the toolchain never reaches production."*

Single-stage Java images carry the full JDK, Maven, the `~/.m2` cache, and your source code — often 800 MB–1.2 GB, with source code and possibly `settings.xml` credentials shipped to prod. Multi-stage builds discard everything except what you explicitly `COPY --from`.

Typical result: **~1 GB → ~200 MB**, with no compiler, no source, no build credentials in the final image.

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN --mount=type=cache,target=/root/.m2 ./mvnw dependency:go-offline -B
COPY src/ ./src/
RUN --mount=type=cache,target=/root/.m2 ./mvnw clean package -DskipTests -B

FROM eclipse-temurin:21-jre AS runtime
RUN useradd -r -u 10001 spring
WORKDIR /app
COPY --from=build --chown=10001:10001 /src/target/*.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java","-XX:MaxRAMPercentage=75.0","-jar","/app/app.jar"]
```

Extra tricks worth mentioning:

- `docker build --target build .` to stop at a stage — useful to run tests in CI in the build stage.
- A dedicated `test` stage: `FROM build AS test` + `RUN ./mvnw verify` so tests run inside the same environment.
- `COPY --from=nginx:latest /etc/nginx/nginx.conf ./` — you can copy from any image, not just your own stages.

### Q37. How do you reduce a Spring Boot image size? (Give an ordered list)

1. **Multi-stage build** — JRE runtime, no JDK/Maven. Biggest single win.
2. **Slimmer base** — `-jre` → `-jre-alpine` or distroless.
3. **Layered jar** — doesn't shrink total size but massively shrinks the *changed* bytes per deploy, which is what actually matters for deploy speed.
4. **`.dockerignore`** — keeps `target/`, `.git`, IDE files out of the context.
5. **Chain `RUN` commands and clean in the same layer:**
   ```dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends curl \
       && rm -rf /var/lib/apt/lists/*
   ```
   Cleaning in a *separate* `RUN` does not shrink the image — the files still exist in the earlier layer.
6. **Custom JRE with `jlink`** — include only required modules:
   ```dockerfile
   RUN jlink --add-modules $(jdeps --print-module-deps app.jar) \
       --strip-debug --no-man-pages --no-header-files --compress=2 \
       --output /javaruntime
   ```
7. **Class Data Sharing (AppCDS)** — doesn't reduce size much but cuts startup time significantly, which matters for autoscaling.
8. **GraalVM native image** — ~50 MB image, ~50 ms startup, low memory. Trade-offs: long build times, reflection/proxy configuration effort, no JIT peak throughput, harder profiling. Good for serverless/functions, questionable for a large legacy Spring monolith.

### Q38. Why is a smaller image actually valuable? (Interviewers probe this)

Not just disk. Faster CI push and pull → faster deploys and rollbacks; faster autoscaling (new pod pulls quicker); smaller attack surface and far fewer CVEs in scans (most CVEs come from OS packages you never use); lower registry storage and egress cost; and less noise for the security team to triage.

**Make it concrete with numbers:** suppose a node must pull the image before a new pod can start. At ~100 MB/s effective throughput a 1.2 GB image takes ~12 s to download (plus extraction) while a 200 MB one takes ~2 s — and under real registry/network contention the gap is far larger. During a traffic spike, that difference is how long users wait for capacity; during an incident rollback it's how long you stay broken. And because layers are cached per node, a layered jar (Q30) means a typical deploy only pulls the few MB that actually changed.

**On the security side:** a vulnerability scanner reports every vulnerable package in the image, used or not. A full OS image with `curl`, `perl`, `gcc` and friends may show dozens of findings; a distroless/slim JRE image often shows a handful. Fewer packages = fewer findings to triage *and* fewer tools available to an attacker who gets code execution.

Pitfall to mention: don't chase size at the expense of operability — an Alpine/musl switch that causes native-library crashes (S23) or a distroless image your team can't debug (S25) costs more than the megabytes saved.

---

## 7. Volumes & Data Persistence

### Q39. Volume vs bind mount vs tmpfs

**Hook:** *"Volume = Docker owns it (prod data). Bind mount = you own the host path (dev). tmpfs = RAM only (secrets/scratch)."*

| | Named volume | Bind mount | tmpfs |
|---|---|---|---|
| Location | `/var/lib/docker/volumes/...` | Any host path | Host memory |
| Managed by | Docker | You | Docker |
| Portable | Yes | No (host-path dependent) | Yes |
| Works with drivers (NFS/EBS/cloud) | Yes | No | No |
| Survives container removal | Yes | Yes | No |
| Best for | DB data, prod persistence | Live source reload in dev, config files | Secrets, `/tmp` on read-only containers |

```bash
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres:16       # named volume
docker run -v "$PWD/src:/app/src:ro" myapp                       # bind mount, read-only
docker run --tmpfs /tmp:rw,noexec,nosuid,size=256m myapp          # tmpfs
docker volume inspect pgdata
```

### Q40. What happens to data when a container is removed?

The writable layer is deleted with the container. Data in a **named volume** or **bind mount** survives (`docker rm` does not remove volumes; you need `docker rm -v` or `docker volume prune`). **Anonymous volumes** (created by a `VOLUME` instruction or `-v /path`) survive too but with random names and become orphaned garbage — this is why images should declare `VOLUME` sparingly.

**Example to see all three cases:**

```bash
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:16   # named volume
docker run -d --name tmp -v /var/lib/postgresql/data postgres:16         # anonymous volume (random hash name)
docker rm -f db tmp
docker volume ls     # pgdata is still there; the anonymous one is listed with a 64-char name, now orphaned
docker rm -fv tmp    # -v at removal time also deletes that container's anonymous volumes
```

Why this matters operationally: a recreated container (new image version, `docker compose up` after a config change) gets a fresh writable layer. If your DB files lived there, they're gone — which is exactly the S17 scenario. With a named volume, the new container mounts the same data. Also note that **a Compose service that declares no volume for a DB image whose Dockerfile has `VOLUME`** silently gets an anonymous volume — data looks persistent until someone runs `docker volume prune` or `compose down -v`.

### Q41. How do you back up a Docker volume?

```bash
# Backup
docker run --rm -v pgdata:/data -v "$PWD":/backup alpine \
  tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .

# Restore
docker run --rm -v pgdata:/data -v "$PWD":/backup alpine \
  sh -c "rm -rf /data/* && tar xzf /backup/pgdata-2026-01-15.tar.gz -C /data"
```

For a database, prefer a logical dump (`pg_dump` / `mysqldump`) over a filesystem snapshot of a **running** DB, since a live data directory tar is not guaranteed consistent.

### Q42. How do you share a directory between containers?

Mount the same named volume into both, or use `--volumes-from`. But say the senior caveat: shared-filesystem coordination between services is usually a design smell. Two Spring Boot replicas writing to one shared folder invites race conditions and file-locking issues; use object storage, a message queue, or a database instead. Legitimate uses: a sidecar log shipper reading a log volume, or a shared read-only config/certs volume.

```bash
# Two containers sharing one named volume
docker volume create shared-certs
docker run -d --name issuer   -v shared-certs:/certs         cert-generator
docker run -d --name api      -v shared-certs:/certs:ro      myapp:1.4.2   # :ro = can read, can't modify

# Alternative: inherit every volume of another container
docker run -d --name shipper --volumes-from api:ro log-shipper
```

Why a design smell: on a single host this works because both containers see the same directory. But across hosts (Kubernetes, ECS) the volume is usually node-local or a block device attachable to only one node, so "shared folder" silently stops working unless you introduce NFS/EFS — which brings latency and locking problems. Mark the consuming side `:ro` whenever it only reads, so a bug in one container can't corrupt the data of the other.

---

## 8. Networking

### Q43. Explain Docker network drivers

| Driver | Behaviour | Use case |
|---|---|---|
| `bridge` | Default; private virtual network on the host with NAT | Single-host apps, Compose |
| `host` | No isolation; container uses the host network stack directly | Max network performance, port-scanning tools |
| `none` | No networking at all | Batch jobs, maximum isolation |
| `overlay` | Multi-host network via VXLAN | Swarm / multi-node clusters |
| `macvlan` | Container gets a MAC and IP on the physical LAN | Legacy apps needing to look like a physical host |
| `ipvlan` | Like macvlan at L3 | IP-per-container without MAC bloat |

**Plain-language reading of the table:**

- **bridge** — Docker creates a private virtual switch on the host (`docker0`, or a custom one) and gives each container an IP on it. Outbound traffic is NAT-ed (address-translated) through the host; inbound traffic only arrives if you publish a port with `-p`. This is what you use 95% of the time, including every Compose stack.
- **host** — the container shares the host's network stack, so port 8080 in the container *is* port 8080 on the host (no `-p` needed, no NAT overhead). Downsides: no port isolation (two containers can't both bind 8080) and a bigger blast radius. Works as described on Linux hosts; Docker Desktop (macOS/Windows) historically did not support it because Docker runs inside a VM.
- **none** — only a loopback interface. Used for jobs that must not touch the network (e.g. processing untrusted files).
- **overlay** — a virtual network stretched across multiple Docker hosts. *VXLAN* (Virtual eXtensible LAN) is the tunnelling technique: container packets are wrapped inside UDP packets between hosts, so containers on different machines behave as if on one LAN. Needed for Swarm; Kubernetes achieves the same goal with CNI plugins (Q47).
- **macvlan / ipvlan** — containers get addresses directly on your physical network (macvlan: each gets its own MAC address; ipvlan: they share the host's MAC, differing only by IP). Used when something outside Docker (legacy system, monitoring tool, firewall rule) must reach the container by a "real" LAN IP. Rare for Spring Boot services.

Check which network a container is on with `docker network ls` and `docker inspect -f '{{json .NetworkSettings.Networks}}' app`.

### Q44. Default bridge vs user-defined bridge — why does it matter?

**Hook:** *"Only user-defined bridges give you DNS by container name."*

- **Default `bridge`** — no automatic DNS; containers can only reach each other by IP (or the deprecated `--link`). All containers share it, so isolation is poor.
- **User-defined bridge** — Docker runs an embedded DNS server at `127.0.0.11`, so `http://payment-service:8080` resolves by container name or network alias. Containers can be attached/detached at runtime, and networks isolate groups of services.

```bash
docker network create app-net
docker run -d --name postgres --network app-net postgres:16
docker run -d --name api --network app-net -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/db myapp
```

This is exactly what Docker Compose does automatically, which is why `jdbc:postgresql://postgres:5432/orders` works in Compose but `localhost` does not.

### Q45. `EXPOSE` vs `-p` vs `-P`?

- `EXPOSE 8080` — metadata/documentation only. Publishes nothing.
- `-p 8080:8080` — publish container port 8080 to host port 8080 (iptables DNAT).
- `-p 127.0.0.1:8080:8080` — bind only to loopback. **Important for security**: plain `-p 8080:8080` binds `0.0.0.0` and can bypass a host firewall like UFW, because Docker writes its own iptables rules in the DOCKER chain.
- `-P` — publish all `EXPOSE`d ports to random high host ports.

Note that **containers on the same user-defined network reach each other without any publishing at all** — you only publish what the outside world needs. A database should not be published in prod.

### Q46. Inside a container, why does `localhost:5432` fail to reach my database?

Because each container has its own network namespace, so `localhost` is *that container*. Resolution depends on where the DB is:

| DB location | Correct host to use |
|---|---|
| Another container on the same user-defined network | its container name / alias, e.g. `postgres` |
| On the host machine | `host.docker.internal` (Docker Desktop; on Linux add `--add-host=host.docker.internal:host-gateway`) |
| External server | its real DNS/IP, and make sure the network path/security group allows it |
| Container using `--network host` | `localhost` genuinely works |

### Q47. How do containers talk across hosts?

`overlay` networks in Swarm (VXLAN tunnels between nodes, with an internal load-balanced VIP per service), or a Kubernetes CNI plugin (Calico, Cilium, Flannel) which gives every pod a routable IP and provides Service abstraction for discovery and load balancing. Mention service meshes (Istio/Linkerd) for mTLS, retries and traffic shifting if the discussion goes to microservices at scale.

**Unpacking the jargon:** the problem is that a container's IP is only meaningful on its own host's private bridge, so container A on host 1 can't normally reach container B on host 2. Two families of solutions exist, plus a layer that sits on top:

- **Overlay (Swarm):** Docker builds a tunnel between hosts (VXLAN, see Q43) so all containers in the network share one virtual subnet. A Swarm *service* gets a **VIP** (virtual IP): clients call the service name, and Docker load-balances across the healthy replicas behind that single address.
- **Kubernetes CNI (Container Network Interface):** a plugin standard for pod networking. Calico, Cilium and Flannel each implement the Kubernetes rule *"every pod gets its own cluster-wide IP and can reach every other pod without NAT"*, using routing, tunnelling or eBPF. On top of that, a Kubernetes **Service** gives a stable DNS name (e.g. `payment-service.prod.svc.cluster.local`) that load-balances across pods, which is how Spring Boot services discover each other.
- **Service mesh:** a layer (sidecar proxies) that adds mTLS (mutual TLS — both sides present certificates), retries, timeouts and canary traffic splitting without changing application code.

From the Spring Boot developer's side, all of this simply means: call the other service by its stable DNS name (`http://inventory-service:8080`), never by a container IP, because IPs change on every restart.

---

## 9. Docker Compose

### Q48. What is Docker Compose and where does it fit?

Declarative multi-container definition in YAML for a **single host**. It creates a dedicated network, resolves service names via DNS, and manages lifecycle with one command. Excellent for local development, integration tests, and demos. It is **not** an orchestrator — no multi-node scheduling, self-healing across hosts, rolling updates with surge control, or autoscaling. For production you move to Kubernetes (or Swarm for something small).

**Why it exists / how it works:** without Compose, starting a Spring Boot + Postgres + Redis stack means typing three long `docker run` commands in the right order, creating a network, and remembering every flag. Compose captures that as one version-controlled YAML file: you declare *services* (containers), *volumes* and *networks*, and `docker compose up` creates them all, puts them on a shared network (so `postgres` resolves as a hostname), and starts them in dependency order.

Minimal example:

```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/orders   # "db" = service name
    depends_on: [db]
  db:
    image: postgres:16
    environment: { POSTGRES_PASSWORD: example }
```

`docker compose up -d` → one network, two containers, and the same file works on every developer's machine and in CI. A full production-shaped file follows in Q49. Interview probe: *"Can you run Compose in production?"* — on a single host for small internal tools, yes; but with no multi-node scheduling, no self-healing if the host dies, and no rolling-update control, anything business-critical belongs on an orchestrator.

### Q49. Write a production-shaped Compose file for a Spring Boot stack

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        JDK_VERSION: "21"
    image: payment-service:1.4.2
    container_name: payment-api
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: docker
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/orders
      SPRING_DATASOURCE_USERNAME: app
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
      SPRING_DATA_REDIS_HOST: redis
      JAVA_OPTS: "-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD-SHELL", "curl -fsS http://localhost:8080/actuator/health/liveness || exit 1"]
      interval: 15s
      timeout: 3s
      retries: 5
      start_period: 60s
    deploy:
      resources:
        limits:   { cpus: "2", memory: 1g }
        reservations: { cpus: "0.5", memory: 512m }
    restart: unless-stopped
    networks: [backend]
    logging:
      driver: json-file
      options: { max-size: "50m", max-file: "5" }

  postgres:
    image: postgres:16.3-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD is required}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d orders"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [backend]

  redis:
    image: redis:7.2-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redisdata:/data
    networks: [backend]

volumes:
  pgdata:
  redisdata:

networks:
  backend:
    driver: bridge
```

Points to call out while presenting it: `depends_on: condition: service_healthy` (not plain `depends_on`), Postgres not published to the host, resource limits, log rotation, pinned image tags, and `${VAR:?message}` to fail fast on missing secrets.

### Q50. Does `depends_on` guarantee the database is ready?

**Hook:** *"Plain `depends_on` waits for *started*, not *ready*."*

Plain `depends_on` only orders container start. Postgres accepts connections seconds after the process starts, so Spring Boot's HikariCP may fail during context initialization and the container exits.

Three fixes, in order of preference:

1. **`condition: service_healthy`** with a real healthcheck on the dependency (shown above).
2. **Make the app resilient** — this is the *correct* answer for production, because in Kubernetes nothing guarantees start order at all. Configure Hikari to tolerate a slow DB and add retry:
   ```properties
   spring.datasource.hikari.initialization-fail-timeout=-1
   spring.datasource.hikari.connection-timeout=10000
   spring.flyway.connect-retries=10
   ```
   Plus readiness probe based on DB connectivity so traffic arrives only when healthy.
3. **Wait scripts** (`wait-for-it.sh`, `dockerize`) — works but adds a shell wrapper; least favourite because it hides the real requirement (app-level resilience).

### Q51. Useful Compose commands

```bash
docker compose up -d --build
docker compose up -d --scale api=3            # needs no fixed container_name / host port
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose down                            # stop + remove containers/network
docker compose down -v                         # ALSO deletes volumes (data loss)
docker compose config                          # render final merged config — great for debugging
docker compose --profile debug up -d           # profiles for optional services
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d   # overrides
```

The override pattern is how you keep one base file with per-environment deltas: `docker-compose.override.yml` is picked up automatically for local dev.

**What the less obvious commands do:**

- `up -d --build` — `-d` runs detached (background); `--build` forces an image rebuild instead of reusing a stale one.
- `--scale api=3` — runs three replicas of the `api` service. It fails if the service sets a fixed `container_name` or publishes a fixed host port (`8080:8080`), because three containers can't share the same name or host port. Publish a range or put a reverse proxy in front.
- `down` vs `down -v` — `down` removes containers and the network but **keeps named volumes**; adding `-v` also deletes them, i.e. your database data (S17).
- `config` — prints the final YAML after merging files and substituting `${VARS}`; first thing to run when "Compose isn't picking up my variable/override".
- `--profile debug` — services labelled `profiles: [debug]` (pgAdmin, Kafka UI, a mock server) only start when you ask for that profile, keeping the default stack lean.
- `-f base.yml -f prod.yml` — later files override or extend earlier ones (maps are merged, lists like `ports` are appended), e.g. the prod file only changes image tags and resource limits.

---

## 10. Resources, Limits & Performance

### Q52. How do you limit CPU and memory, and what happens at the limit?

```bash
docker run -m 1g --memory-swap 1g \       # hard memory limit, swap disabled
           --memory-reservation 768m \    # soft limit under host pressure
           --cpus 1.5 \                   # CFS quota: 1.5 cores worth
           --cpu-shares 1024 \            # relative weight under contention
           --pids-limit 512 \
           myapp:1.4.2
```

- **Memory limit exceeded** → the kernel OOM killer terminates the process. Container exits **137**, and `docker inspect` shows `"OOMKilled": true`. There is no warning and no graceful shutdown.
- **CPU limit exceeded** → the process is **throttled**, not killed. It just gets slower, visible as `nr_throttled` / `throttled_time` in cgroup stats and as latency spikes in your metrics.

**Always set `--memory-swap` equal to `-m`** for a JVM. If swap is allowed, the JVM's heap can be paged out and GC pauses become catastrophic (seconds), which looks like a mysterious app hang.

### Q53. How do you diagnose a container that keeps getting OOMKilled?

Step-by-step (this is a strong answer because it's a procedure, not a guess):

1. Confirm it: `docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}' app` → `true 137`.
2. Check the limit vs actual: `docker stats`, and `docker inspect -f '{{.HostConfig.Memory}}'`.
3. Check the JVM's view: `docker exec app jcmd 1 VM.flags | grep MaxHeap` — is the heap larger than the container limit? Is `MaxRAMPercentage` set?
4. Separate heap leak from native leak:
   - Heap growing → `jcmd 1 GC.heap_info`, take a heap dump (`jcmd 1 GC.heap_dump /dumps/x.hprof`), analyze in MAT/VisualVM.
   - Heap stable but RSS growing → native leak: Metaspace, thread explosion, or direct buffers. Use `-XX:NativeMemoryTracking=summary` + `jcmd 1 VM.native_memory summary.diff`.
5. Always pre-arm the container: `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps` on a **mounted volume**, otherwise the dump dies with the container.
6. Fix: raise the limit *only after* you know the real footprint; cap Metaspace and direct memory; reduce Tomcat `max-threads`; fix the leak.

### Q54. How do you improve Spring Boot startup time in containers?

- **AppCDS / CDS archive** (Java 12+, and Spring Boot 3.3+ has first-class support) — pre-loads class metadata, commonly 20–40% faster startup.
- **Lazy initialization** — `spring.main.lazy-initialization=true` (careful: moves failures to first request).
- Trim auto-configuration and unused starters; avoid classpath scanning of huge packages.
- **Checkpoint/restore (CRaC)** or **GraalVM native image** for near-instant startup where the trade-offs are acceptable.
- Don't fight it with health probes — set a proper `start_period` / `initialDelaySeconds` instead.

**Why startup is slow and what each lever does:**

A Spring Boot app spends startup time on (1) the JVM loading and verifying thousands of classes, (2) component scanning and building the bean graph, (3) auto-configuration conditions, and (4) JIT warm-up. Containers make this matter more, because autoscaling and rolling deploys start new instances constantly, and CPU limits (Q29) slow startup further.

- **AppCDS (Application Class-Data Sharing)** — the JVM saves already-parsed class metadata to an archive file once, then memory-maps it on later starts instead of re-parsing. You generate it at image-build time (e.g. a training run) and add `-XX:SharedArchiveFile=app.jsa`. Gain depends on app size; treat the 20–40% as typical, not guaranteed.
- **Lazy initialization** — beans are created on first use rather than at boot. Faster start, but a misconfigured bean now fails on the first request in production instead of at deploy time. Use it deliberately (or per-bean with `@Lazy`) and keep a readiness probe that exercises critical paths.
- **Trimming** — each unused starter adds auto-configuration classes to evaluate. Use `--debug` to print the auto-configuration report and remove what you don't use; narrow `@ComponentScan` to your packages.
- **CRaC (Coordinated Restore at Checkpoint)** — snapshot a warmed-up JVM and restore it in milliseconds; needs a CRaC-enabled JDK and care with open connections. **GraalVM native image** — ahead-of-time compiled binary; fast start but trade-offs in Q37.
- **Probes** — a slow start is not a failure; raising `start_period` is configuration, not a hack, but it should be set from measured startup time plus margin.

Measure first: Spring Boot logs `Started App in 18.4 seconds`; compare before/after each change under the *same* CPU limit as production.

### Q55. What are common Docker performance pitfalls for Java?

- Heap not sized to the container limit (Q27) — the #1 issue.
- Fractional CPU limits causing GC throttling (Q29).
- Swap enabled → paged-out heap.
- Bind mounts on Docker Desktop macOS/Windows are slow (gRPC-FUSE/virtiofs); noticeable when Maven builds inside a mounted volume. Use named volumes for `~/.m2`, or build in the image.
- Huge single-layer images making every deploy slow (use layered jars).
- Writing logs to a file inside the container's writable layer (overlayfs copy-on-write overhead + disk growth).
- Too many Tomcat threads for the allotted CPU, causing context-switch thrash.

**Why each one hurts (the mechanism):**

- *Heap not sized to the limit* — the JVM defaults to ~25% of the container limit, so you pay for 1 GB but use ~256 MB of heap and spend more time in GC; or, with a hard-coded large `-Xmx`, the total exceeds the cgroup limit and the kernel kills the process (exit 137).
- *Fractional CPU* — the CFS quota freezes the process once its slice is used, so GC and request threads stall together → latency spikes with low average CPU (S15).
- *Swap* — if the host lets the container swap, cold heap pages go to disk and a full GC must read them back; pauses jump from milliseconds to seconds.
- *Bind mounts on Docker Desktop* — file operations cross the VM boundary between the Mac/Windows host and the Linux VM, so metadata-heavy workloads (Maven reading thousands of small files from `~/.m2`) are many times slower than on a native Linux volume.
- *Fat single-layer image* — every deploy re-pulls the entire app jar (Q30).
- *Log file in the writable layer* — OverlayFS must copy-on-write (copy a file up from a lower layer before changing it) and the file grows without rotation until the disk is full (S10).
- *Too many threads* — Tomcat's default of 200 request threads on 1 CPU means constant context switching; size pools to the CPU you actually have and use async/reactive I/O if blocking calls dominate.

---

## 11. Security & Production Readiness

### Q56. List Docker security best practices you actually apply

**Hook mnemonic — "N-M-S-R-C-P-S":** **N**on-root, **M**inimal base, **S**can, **R**ead-only, **C**apabilities dropped, **P**inned versions, **S**ecrets external.

1. **Run as non-root.** Containers run as root by default; combined with a mounted volume or a kernel escape, that's host risk.
   ```dockerfile
   RUN groupadd -r spring && useradd -r -g spring -u 10001 spring
   USER 10001
   ```
   Use a numeric UID so it works with Kubernetes `runAsNonRoot`. Note ports <1024 need root — so run Spring on 8080, not 80.
2. **Minimal base image** — distroless/slim: no shell, no package manager, no `curl` for an attacker to use.
3. **Scan images in CI** — Trivy, Grype, Snyk, Docker Scout. Fail the build on HIGH/CRITICAL with a documented exception process. Rescan periodically because new CVEs appear against unchanged images.
4. **Read-only root filesystem** — `--read-only` plus `--tmpfs /tmp`. Spring Boot needs a writable `/tmp` (Tomcat work dir, multipart uploads).
5. **Drop capabilities** — `--cap-drop ALL` and add back only what's needed; `--security-opt no-new-privileges` to block setuid escalation.
6. **Never mount `/var/run/docker.sock`** into an app container — it's root-equivalent on the host.
7. **No secrets in images or env** — use Vault / AWS Secrets Manager / Kubernetes secrets mounted as files; BuildKit `--mount=type=secret` at build time.
8. **Pin base images by digest**; automate rebuilds so patches actually ship.
9. **Sign and verify images** — Cosign/Notary, admission policy to reject unsigned images.
10. **Avoid `--privileged`**; limit `--pids-limit`, use user namespace remapping, keep the daemon and kernel patched.
11. **Don't publish internal ports**; bind to `127.0.0.1` where possible.
12. **Generate an SBOM** (`docker buildx --sbom=true`, Syft) for supply-chain traceability.

**Plain-language "why" for the items people can't explain:**

- *Root in a container is dangerous* because UID 0 inside is UID 0 on the host unless user namespaces are enabled. If an attacker escapes the container or abuses a mounted volume, they act as host root; as UID 10001 they can't.
- *Capabilities* are root's powers split into pieces (bind low ports, change file ownership, load kernel modules…). Docker grants a default subset; `--cap-drop ALL` removes them all and a Spring Boot app on port 8080 needs none. `no-new-privileges` stops a process from gaining more via setuid binaries.
- *Read-only filesystem* means malware or an exploit can't drop a binary or modify the app; anything legitimately writable (`/tmp`, heap dumps) is an explicit tmpfs or volume.
- *Image scanning* compares installed packages and jar dependencies against CVE databases (Common Vulnerabilities and Exposures — public IDs for known flaws). It must also run *after* build, because a CVE published next month applies to an image you built today.
- *Signing* (e.g. Cosign) attaches a cryptographic signature to the image digest; an admission policy then refuses to run anything not signed by your pipeline, blocking tampered or rogue images.
- *SBOM (Software Bill of Materials)* is the machine-readable list of everything inside the image, so when a new zero-day lands you can query "which services ship this library?" instead of guessing.

Typical probe: *"What's the first thing you'd change in an inherited Dockerfile?"* → add `USER`, pin the base image, drop the JDK from the final stage, and check for secrets in `ENV`/`ARG`.

### Q57. A hardened production Dockerfile for Spring Boot (worth memorizing the shape)

```dockerfile
# syntax=docker/dockerfile:1
# ---------- build ----------
FROM eclipse-temurin:21.0.4_7-jdk AS build
WORKDIR /src
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN --mount=type=cache,target=/root/.m2 \
    --mount=type=secret,id=m2settings,target=/root/.m2/settings.xml \
    ./mvnw -B dependency:go-offline
COPY src/ ./src/
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B clean package -DskipTests

# ---------- layer extraction ----------
FROM build AS extract
WORKDIR /build
RUN cp /src/target/*.jar app.jar && java -Djarmode=layertools -jar app.jar extract

# ---------- runtime ----------
FROM eclipse-temurin:21.0.4_7-jre
ARG GIT_SHA=unknown
LABEL org.opencontainers.image.source="https://github.com/org/payment-service" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.vendor="MyCompany"

RUN groupadd -r spring && useradd -r -g spring -u 10001 spring \
    && mkdir -p /app /dumps && chown -R 10001:10001 /app /dumps

WORKDIR /app
COPY --from=extract --chown=10001:10001 /build/dependencies/ ./
COPY --from=extract --chown=10001:10001 /build/spring-boot-loader/ ./
COPY --from=extract --chown=10001:10001 /build/snapshot-dependencies/ ./
COPY --from=extract --chown=10001:10001 /build/application/ ./

USER 10001
EXPOSE 8080
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0 -XX:InitialRAMPercentage=50.0 \
  -XX:MaxMetaspaceSize=256m -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
  -Djava.security.egd=file:/dev/./urandom"

HEALTHCHECK --interval=15s --timeout=3s --start-period=60s --retries=3 \
  CMD ["/bin/sh","-c","curl -fsS http://localhost:8080/actuator/health/liveness || exit 1"]

STOPSIGNAL SIGTERM
ENTRYPOINT ["java","org.springframework.boot.loader.launch.JarLauncher"]
```

`JAVA_TOOL_OPTIONS` is a nice touch: the JVM picks it up automatically, so the entrypoint stays a clean exec-form array with no shell.

### Q58. How do you handle secrets properly?

Ranked worst → best:

1. Hardcoded in `application.yml` in the image — never.
2. `ARG`/`ENV` in the Dockerfile — leaks into history/inspect.
3. `-e PASSWORD=...` at runtime — visible in `docker inspect`, `ps`, and often in orchestrator manifests and shell history. Acceptable only for local dev.
4. `--env-file` — better (not on the command line) but still plaintext on disk and in inspect.
5. **Docker/Kubernetes secrets mounted as files** at `/run/secrets/...` (tmpfs) — read with Spring's `spring.config.import=optional:file:/run/secrets/` or a `_FILE` convention.
6. **External secret manager** (Vault, AWS Secrets Manager, Azure Key Vault) with short-lived dynamic credentials and IAM/workload identity — no static secret exists to leak. This is the answer that lands.

Add: rotate credentials, audit access, and if a secret ever touched an image layer, treat it as compromised and rotate — deleting the layer is not enough.

### Q59. Docker Swarm vs Kubernetes — when would you pick which?

| | Swarm | Kubernetes |
|---|---|---|
| Setup | Trivial (`docker swarm init`) | Complex (or managed EKS/AKS/GKE) |
| Learning curve | Low | Steep |
| Autoscaling | Manual/limited | HPA/VPA/Cluster autoscaler |
| Ecosystem | Small | Enormous (Helm, operators, service mesh) |
| Best for | Small teams, few services, simple needs | Production microservices at scale |

Honest senior answer: "Swarm is genuinely easier and fine for a handful of services on a few nodes, but the industry and tooling have consolidated on Kubernetes, so for anything we expect to grow I standardize on managed Kubernetes. I use Compose locally and Kubernetes in production, and keep the image identical between them."

**What these tools actually do (define before comparing):** both are *orchestrators* — they run containers across many machines and keep them in the desired state. You declare "I want 3 replicas of `payment-service:1.4.2`"; the orchestrator schedules them onto nodes, restarts any that die, load-balances traffic and performs rolling updates. Swarm is built into Docker (`docker service create --replicas 3 ...`) and reuses Compose-style files. Kubernetes uses its own objects — Deployment (replicas + rollout), Service (stable address), Ingress (HTTP routing), ConfigMap/Secret (config), HPA (autoscaling) — which is more to learn but far more powerful and portable across AWS/Azure/GCP.

```bash
# Swarm
docker swarm init
docker stack deploy -c docker-compose.yml shop

# Kubernetes
kubectl apply -f deployment.yaml
kubectl rollout status deployment/payment-service
```

Interview framing: Docker (the image) is the *packaging*; Kubernetes is the *runtime platform*. Your Dockerfile quality (exec-form entrypoint, health endpoints, graceful shutdown, non-root) is what makes the app behave well under either one.

---

## 12. Scenario-Based Questions (Senior Level)

> **Answer framework for every scenario — use this out loud:**
> **O-D-F-P** → **O**bserve (what evidence do I gather?) → **D**iagnose (what's the root cause?) → **F**ix (immediate mitigation) → **P**revent (what change stops it recurring?).
> Interviewers score you on the *process*, not just the answer. Always say what command you'd run first.

---

### S1. "The container starts, then exits after ~40 seconds with exit code 137. It works fine on my laptop."

**Observe**
```bash
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' app   # 137 true
docker stats app                                                   # memory climbing to the limit
docker exec app jcmd 1 VM.flags | grep -i maxheap
```

**Diagnose** — 137 = SIGKILL, and `OOMKilled: true` confirms the kernel OOM killer. The classic root cause: the container has `-m 512m` but the JVM sized its heap from a different view of memory (or `MaxRAMPercentage` was never set, so heap defaults to 25% and the *non-heap* footprint — Metaspace, threads, direct buffers — pushed total RSS past 512 MB). On the laptop there was no limit, so it never surfaced.

**Fix**
```bash
docker run -m 1g --memory-swap 1g \
  -e JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70 -XX:MaxMetaspaceSize=256m -XX:MaxDirectMemorySize=128m" \
  myapp
```

**Prevent** — set `MaxRAMPercentage` in the image; run the same memory limit in dev/CI as in prod so it fails early; add a container-memory dashboard/alert at 80% of limit; enable `HeapDumpOnOutOfMemoryError` to a mounted volume; load-test with production-like limits.

---

### S2. "Deploys drop requests. Users see 502s for a few seconds every release."

**Observe** — check how the container is stopped, the exit code (`143` graceful vs `137` killed), and the Dockerfile's ENTRYPOINT form.

**Diagnose** — three likely causes, and I'd check all:
1. `ENTRYPOINT java -jar app.jar` in **shell form** → `/bin/sh` is PID 1 and doesn't forward SIGTERM. The JVM never gets it, Docker SIGKILLs after 10s.
2. `server.shutdown=graceful` not configured → Tomcat stops immediately and kills in-flight requests.
3. The load balancer keeps sending traffic during shutdown because there's no readiness gate / pre-stop delay.

**Fix**
```dockerfile
ENTRYPOINT ["java","-jar","/app/app.jar"]        # exec form
STOPSIGNAL SIGTERM
```
```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=25s
```
```bash
docker stop --time=30 app    # grace period MUST exceed the app's shutdown timeout
```

**Prevent** — in Kubernetes add a `preStop` sleep (5–10s) so the endpoint is removed from the Service before shutdown begins, set `terminationGracePeriodSeconds` above the Spring timeout, and verify with a load generator running during a deploy (should show zero 5xx).

---

### S3. "Docker builds take 12 minutes in CI. Every single build re-downloads all Maven dependencies."

**Diagnose** — `COPY . .` before `mvn package`, so any source change invalidates the dependency layer. Possibly no `.dockerignore`, so `.git` and `target/` also churn the context hash. CI has a fresh daemon each run, so there's no local cache at all.

**Fix**
1. Split the copy so `pom.xml` is its own cached layer (Q17).
2. BuildKit cache mount for `~/.m2`.
3. `.dockerignore` for `target/`, `.git`, IDE dirs.
4. Registry-backed cache so a fresh CI agent still gets warm layers:
   ```bash
   docker buildx build \
     --cache-from type=registry,ref=repo/app:buildcache \
     --cache-to   type=registry,ref=repo/app:buildcache,mode=max \
     -t repo/app:$GIT_SHA --push .
   ```
5. Run tests in a separate parallel stage/job rather than serially in the image build.

**Result to quote** — typically 12 min → 2–3 min, and under a minute for a code-only change.

---

### S4. "The app can't connect to the database. `Connection refused: localhost:5432`."

**Diagnose** — `localhost` inside a container is the container itself. The JDBC URL was copied from local dev.

**Fix** — depends on where the DB lives:

| DB location | Fix |
|---|---|
| Sibling container, same user-defined network | `jdbc:postgresql://postgres:5432/db` (service name) |
| Sibling container, default bridge network | No DNS — create a user-defined network |
| On the host | `host.docker.internal` (Docker Desktop) or `--add-host=host.docker.internal:host-gateway` on Linux |
| Remote/managed | real hostname + verify security group / firewall / VPC route |

**Verify from inside**
```bash
docker exec -it api sh -c "getent hosts postgres; nc -zv postgres 5432"
docker network inspect app-net      # confirm both containers are attached
```

**Prevent** — externalize the URL as an env var so no hostname is baked in, and add an integration test in Compose that exercises the containerized wiring.

---

### S5. "It worked in QA but the exact same image fails in production."

**Diagnose ladder** — if the *image* is byte-identical (verify by digest, not tag!), the difference must be outside the image:

1. **Config/env** — different `SPRING_PROFILES_ACTIVE`, missing env var, different secret. `docker inspect` both and diff the env.
2. **Was it really the same image?** If deployed by tag, prod may have pulled a different build. Compare `RepoDigests`.
3. **Resource limits** — prod has tighter memory/CPU → OOM or throttling.
4. **Network policy** — prod egress blocked to an external API, DNS differences, proxy required.
5. **Volumes/permissions** — a mounted path exists in QA, not in prod; or a non-root UID can't write to the mounted directory.
6. **Architecture** — image built on an arm64 laptop, prod nodes are amd64 (`exec format error`).
7. **Data** — prod data volume/scale exposes a query timeout or migration issue.

**Prevent** — promote by **digest** through environments, keep environment deltas in a small reviewed config layer, run prod-equivalent limits in staging, and put a smoke test on the readiness endpoint in the deploy pipeline.

---

### S6. "You need zero-downtime deployment of a Spring Boot service. Walk me through it."

**With Docker alone (single host)** — blue/green behind a reverse proxy:
1. Run `v2` alongside `v1` on a different internal port, same network.
2. Poll `v2`'s `/actuator/health/readiness` until healthy.
3. Flip the nginx/Traefik upstream to `v2` and reload.
4. Drain and `docker stop --time=30 v1`.
5. Keep `v1`'s image and container around for instant rollback.

**With Kubernetes** — rolling update with `maxSurge=1, maxUnavailable=0`, readiness probe on `/actuator/health/readiness`, liveness on `/actuator/health/liveness`, `preStop` sleep, and `terminationGracePeriodSeconds > spring.lifecycle.timeout`.

**The part senior candidates must add — the hard constraints:**
- **Database compatibility**: during rollout both versions run at once, so schema changes must be **backwards compatible** (expand/contract: add nullable column → deploy code that writes both → backfill → deploy code that reads new → drop old in a later release). Never add a NOT NULL column and deploy in one step.
- **Session state**: must be externalized (Redis/JWT) or users get logged out.
- **Message consumers**: ensure both versions can handle in-flight message formats.
- **Rollback plan**: pinned previous digest, and migrations that are reversible or forward-only-safe.

---

### S7. "Two replicas of the service start simultaneously and both run Flyway migrations. One crashes."

**Diagnose** — concurrent schema migration. Flyway does take a lock on most databases, but you can still hit checksum/lock timeouts, and with Liquibase you get `DatabaseChangeLogLock` contention where a crashed instance leaves the lock held.

**Fix / correct design**
- Rely on the DB lock (`spring.flyway.baseline-on-migrate`, and for Postgres Flyway's advisory lock) plus retries: `spring.flyway.connect-retries=10`.
- **Better:** decouple migration from app startup. Run it as a **separate init job/step** in the pipeline (a Kubernetes Job or a one-shot container using the same image with `-Dspring.flyway.enabled=true --migrate-only` style entrypoint) before rolling out app replicas, with app replicas started with `spring.flyway.enabled=false`.
- Keep migrations idempotent and backwards compatible so a partially-rolled deployment is safe.

**Prevent** — treat migrations as an explicit pipeline stage with its own success gate, so the deploy fails before any app pod starts.

---

### S8. "`docker stats` shows normal heap but the container's memory keeps growing until it's OOMKilled."

**Diagnose** — heap is fine, so it's **native memory**. Candidates: Metaspace growth (dynamic class/proxy generation, hot redeploys, Groovy/scripting), thread leak (each thread = ~1 MB stack; an unbounded executor or a leaking HTTP client), direct byte buffers (Netty/WebClient/NIO not released), a native library leak (compression, crypto), or a leaking JDBC/connection pool.

**Observe**
```bash
docker exec app jcmd 1 VM.native_memory summary          # requires -XX:NativeMemoryTracking=summary
docker exec app jcmd 1 Thread.print | grep -c '"'        # thread count trend
docker exec app jcmd 1 VM.metaspace
docker exec app jcmd 1 GC.heap_info
```

**Fix** — cap the suspects so failure is loud and bounded rather than an OOMKill: `-XX:MaxMetaspaceSize=256m`, `-XX:MaxDirectMemorySize=256m`, bounded thread pools, and fix the actual leak. Enable NMT permanently in non-prod, and expose JVM metrics via Micrometer so `jvm.memory.*` and `jvm.threads.live` are graphed.

---

### S9. "The image is 1.2 GB. Reduce it."

Walk it as an ordered, measured plan:
1. `docker history` / `dive` to find the biggest layers — measure first.
2. Multi-stage: JDK+Maven only in the build stage → typically ~1.2 GB down to ~250 MB immediately.
3. Switch runtime to `-jre` (not `-jdk`).
4. `.dockerignore` to keep source/`.git`/`target` out.
5. Chain `apt-get install` with `rm -rf /var/lib/apt/lists/*` **in the same RUN**.
6. Layered jar for faster incremental deploys.
7. If still needed: distroless or `jlink` custom runtime → ~120 MB.
8. Consider GraalVM native image if startup/footprint are critical and you accept the build/reflection cost.

Close with: "and I'd verify with a scan that the CVE count dropped too, since that's the other half of the benefit."

---

### S10. "Container logs filled the host disk and took down every service on the node."

**Diagnose** — default `json-file` driver has **no rotation**. A chatty DEBUG logger or an exception loop wrote tens of GB to `/var/lib/docker/containers/<id>/*-json.log`.

**Immediate fix**
```bash
docker system df
du -sh /var/lib/docker/containers/*
truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log   # don't rm the file; the daemon holds the fd
```

**Proper fix** — daemon-wide default in `/etc/docker/daemon.json`:
```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "50m", "max-file": "5" }
}
```
(Restart the daemon; applies to newly created containers.)

**Prevent** — rotation as a platform default (not per-team), disk-usage alerting on the node, ship logs off-host to Loki/ELK/CloudWatch, keep app log level at INFO in prod with sampling for noisy paths, and put `/var/lib/docker` on its own volume so a log flood can't take down the OS.

---

### S11. "A mounted volume gives `Permission denied` after we switched the container to a non-root user."

**Diagnose** — the container process runs as UID 10001 but the host directory (or the existing volume content) is owned by root or by a different UID. Linux checks numeric UIDs; the *name* is irrelevant.

**Fix options**
- `chown -R 10001:10001` the host path, or set ownership when creating the directory in the Dockerfile (`RUN mkdir -p /dumps && chown 10001:10001 /dumps`).
- Named volumes copy the image's directory ownership on first initialization, which usually just works — one more reason to prefer named volumes over bind mounts.
- In Kubernetes, use `fsGroup` in the pod security context so the mounted volume is group-owned correctly.
- SELinux hosts: add `:z`/`:Z` to the bind mount.
- Avoid the lazy fix (`chmod 777` or reverting to root) — say this explicitly; interviewers like hearing you reject it.

---

### S12. "Health check keeps failing and the container restarts in a loop, but the app logs look fine."

**Diagnose ladder**
1. **No `--start-period`** — Spring Boot takes 45s to start, the healthcheck fires at 5s, three failures → restart before it ever boots. Most common cause.
2. `curl` isn't in the image (slim/distroless) → the healthcheck command fails with 127, not because the app is unhealthy.
3. Health endpoint requires auth, or Actuator isn't exposed (`management.endpoints.web.exposure.include`).
4. Wrong port/path, or the management port differs (`management.server.port=8081`).
5. The healthcheck hits `/actuator/health` (aggregate), which reports DOWN when a **downstream dependency** like Redis is down → the app gets restarted for someone else's outage.

**Fix** — `--start-period=60s`, use the **liveness** group for restart decisions and **readiness** for traffic decisions, verify manually with `docker exec app curl -v localhost:8080/actuator/health/liveness`, and on distroless rely on the orchestrator's HTTP probe rather than a shell command.

---

### S13. "Image built on an M3 MacBook fails on the EC2 nodes: `exec format error`."

**Diagnose** — architecture mismatch. The Mac produced `linux/arm64`; the nodes are `linux/amd64`. Confirm with `docker image inspect app:tag -f '{{.Architecture}}'`.

**Fix**
```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.4.2 --push .
```
Quick workaround: `--platform linux/amd64` (emulated via QEMU — correct but slow, and native-dependency builds can be flaky).

**Prevent** — always build release artifacts in CI on the target architecture (or with buildx multi-arch), and add a pipeline check that asserts the manifest contains the required platform before promotion.

---

### S14. "`docker run` fails: `bind: address already in use`."

```bash
docker ps -a --filter "publish=8080"
ss -ltnp | grep 8080          # or lsof -i :8080
```
Either another container has the host port, a host process (a locally running Spring Boot from the IDE!) holds it, or a stopped container's mapping wasn't released.

**Fix** — pick a different host port (`-p 8081:8080` — the *container* port doesn't need to change), stop the conflicting process, or let Docker assign one (`-P`). Note that this only affects **published host ports**; containers on the same network all happily use 8080 internally, which is why you should avoid publishing at all for internal services and why `--scale` fails when you hardcode a host port in Compose.

---

### S15. "Team says 'the container is slow' — CPU usage looks low, but p99 latency has spikes."

**Diagnose** — low average CPU with latency spikes is the classic signature of **CFS throttling**. With `--cpus 0.5`, the process gets 50 ms of CPU per 100 ms period; once the quota is exhausted it's frozen until the next period. Averaged over a minute that looks like "low CPU", but a GC pause or a burst of requests hits the wall.

**Observe**
```bash
cat /sys/fs/cgroup/cpu.stat        # nr_throttled, throttled_usec (cgroup v2)
docker exec app jcmd 1 VM.flags | grep ActiveProcessor
```
Also correlate with GC logs (`-Xlog:gc*`) — throttled GC threads stretch pause times.

**Fix** — raise or remove the CPU limit (keep a *request* for scheduling), set `-XX:ActiveProcessorCount` to match, right-size Tomcat threads to the available CPU, and prefer scaling horizontally over squeezing many fractional-CPU JVMs onto a node.

---

### S16. "A security audit found the database password in the image."

**Diagnose**
```bash
docker history --no-trunc myapp:1.4     # ARG / RUN with credentials
docker image inspect myapp:1.4          # ENV values
docker save myapp:1.4 | tar -tv         # audit raw layers for a settings.xml / .env
```
Remember: `RUN rm secret.txt` in a later layer does **not** remove it from the image.

**Fix (in order)**
1. **Rotate the credential immediately** — say this first. The image may have been pulled by anyone with registry access.
2. Delete the image tags from the registry, and check the registry's audit log for who pulled it.
3. Rebuild using BuildKit secret mounts for build-time credentials and a secret manager for runtime.
4. Add a CI gate: `gitleaks`/`trufflehog` on the repo plus a scan of the built image, and fail the pipeline on findings.

**Prevent** — no secrets in `ARG`/`ENV`/`COPY`, ever; short-lived dynamic credentials from Vault/IAM so a leak has a small blast radius; and a documented rotation runbook.

---

### S17. "Data disappeared after the DB container was recreated."

**Diagnose** — the Postgres data directory was in the container's writable layer (no volume) or on an **anonymous** volume that got pruned, or someone ran `docker compose down -v`.

**Fix / correct setup**
```yaml
postgres:
  image: postgres:16.3
  volumes:
    - pgdata:/var/lib/postgresql/data     # named volume
volumes:
  pgdata:
```
Verify with `docker inspect -f '{{json .Mounts}}' postgres` that the mount is a named volume, not empty.

**Prevent** — never run a stateful production database as a plain container without a managed volume; prefer a managed service (RDS/Cloud SQL) or a StatefulSet with a PersistentVolumeClaim; automate backups and *restore drills*; ban `down -v` in shared environments; document which volumes are data volumes so pruning is safe.

---

### S18. "The app inside the container shows UTC timestamps but business logic needs IST."

**Diagnose** — most base images default to UTC and may not even have tzdata (Alpine/distroless).

**Fix**
```dockerfile
ENV TZ=Asia/Kolkata
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone   # Debian
# Alpine: RUN apk add --no-cache tzdata
```
Or `-e TZ=Asia/Kolkata` at runtime, or `-Duser.timezone=Asia/Kolkata`.

**The senior answer:** don't fix it with a timezone — fix it in the code. Store and transmit UTC (`Instant`/`timestamptz`), convert to the user's zone only at the presentation boundary, and keep containers on UTC so logs across regions are comparable. Relying on container TZ makes behaviour depend on deployment config, which is exactly the bug class you're trying to remove.

---

### S19. "Intermittent `UnknownHostException` for an internal service; retrying works."

**Diagnose** — container DNS goes to Docker's embedded resolver (`127.0.0.11`) and then to the host's resolvers. Common causes: JVM DNS caching (`networkaddress.cache.ttl` — the JVM caches positive lookups, and historically negative lookups for 10s), the target container restarted and got a new IP while the JVM held the old one, an upstream resolver timing out, `ndots` search-domain expansion causing extra queries, or a container attached to the default bridge with no DNS at all.

**Fix** — user-defined networks/aliases so names are stable; tune JVM DNS TTL (`-Dsun.net.inetaddr.ttl=30`, `networkaddress.cache.negative.ttl=0`); add client-side retry with backoff (Resilience4j) since transient DNS is a fact of life; set explicit `--dns` if the host resolver is unreliable; in Kubernetes use headless services / stable Service DNS and check CoreDNS metrics and node conntrack limits.

---

### S20. "You must run integration tests against a real Postgres in the CI pipeline. Design it."

**Preferred: Testcontainers** — tests declare their own containers, so they work identically on a laptop and in CI, with no shared-state flakiness:

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryIT {
  @Container
  static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16.3-alpine");

  @DynamicPropertySource
  static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", db::getJdbcUrl);
    r.add("spring.datasource.username", db::getUsername);
    r.add("spring.datasource.password", db::getPassword);
  }
}
```

Operational details to mention: mount `/var/run/docker.sock` into the CI job (sibling containers) or use DinD/Testcontainers Cloud; pin image tags; reuse containers across test classes (`@Container static`) so you pay startup once; use Ryuk for cleanup; and pre-pull images on the agent to avoid registry rate limits.

**Alternative:** Compose-based — `docker compose -f docker-compose.it.yml up -d`, wait on healthchecks, run `mvn verify`, `docker compose down -v`. Simpler to reason about but more pipeline glue and a shared-state risk if agents run jobs in parallel.

---

### S21. "Design the CI/CD pipeline for a Dockerized Spring Boot microservice."

```
1. Checkout + cache ~/.m2
2. mvn verify (unit + integration via Testcontainers), publish coverage
3. Static analysis: SonarQube, gitleaks (secret scan)
4. docker buildx build --platform linux/amd64 \
     --cache-from/--cache-to registry \
     -t $REG/app:$VERSION-$GIT_SHA --provenance=true --sbom=true --push
5. Image scan: trivy image --exit-code 1 --severity HIGH,CRITICAL
6. Sign: cosign sign $REG/app@$DIGEST
7. Deploy to dev by DIGEST → smoke test /actuator/health/readiness
8. Promote the SAME digest to QA → automated regression
9. Manual approval → prod: rolling/canary, monitor error rate + p99, auto-rollback on SLO breach
```

Principles to articulate: **build once, promote the same digest** (never rebuild per environment); immutable tags including the git SHA for traceability; config injected per environment, not baked; and admission policy in prod that rejects unsigned or unscanned images.

---

### S22. "Someone suggests using `docker exec` to hot-patch a config file in production. Your response?"

Push back, and explain why: it breaks immutability, so the running container no longer matches its image. The change vanishes on the next restart, isn't in version control, isn't reviewed, isn't replicated to other replicas (so you now have inconsistent instances), and won't be present in the next deploy — producing a "fixed it, then it broke again" mystery.

**Correct path:** change config in the repo → rebuild or update the external config/env → redeploy. If the incident is urgent, use a mechanism designed for it: an env var change with a restart, a Spring Cloud Config refresh, a feature flag, or `/actuator/loggers` to change log level at runtime (a legitimate, intentionally dynamic endpoint). And note that on a hardened distroless image `docker exec` isn't even possible — which is a feature, not a limitation.

---

### S23. "The same JVM crashes in the container with `SIGSEGV` (139) but is stable outside it."

**Diagnose** — exit 139 is a segfault, so look at native code and platform differences: a native library compiled against **glibc** running on **musl** (Alpine), a mismatched `.so` bundled by a driver (Oracle OCI, Snappy, Netty native transport), a JDK/kernel incompatibility, or insufficient `/dev/shm` (default 64 MB) for something using shared memory.

**Fix** — switch to a glibc base (`eclipse-temurin:21-jre` on Debian slim) to test the musl hypothesis; check `hs_err_pid*.log` (write it to a mounted volume: `-XX:ErrorFile=/dumps/hs_err_%p.log`); raise `--shm-size=1g` if shared memory is implicated; match the library build to the base image; and pin the exact JDK build.

**Prevent** — keep the base image consistent between local, CI, and prod; treat the base image as a reviewed dependency; and load-test on the actual production base image, not just the developer's.

---

### S24. "Registry pulls started failing in production with `toomanyrequests`."

**Diagnose** — Docker Hub anonymous/free pull rate limits, hit because every autoscaling event pulls a public base image (or every CI job pulls fresh).

**Fix** — authenticate pulls; mirror public images into your own registry (ECR/Harbor/Nexus) and reference only internal paths; configure a **pull-through cache** on the registry; pre-pull/bake base images into node AMIs; set `imagePullPolicy: IfNotPresent` with immutable tags so nodes don't re-pull unnecessarily.

**Prevent** — a policy that production manifests may only reference the internal registry, enforced by an admission controller. This also protects against upstream image deletion/tampering.

---

### S25. "How would you debug a production container that has no shell (distroless)?"

Layered answer:
1. **Don't exec — use the app's own instrumentation:** Actuator (`/actuator/health`, `/env`, `/metrics`, `/loggers`, `/threaddump`, `/heapdump`), Prometheus metrics, distributed traces, and structured logs.
2. **JMX / jcmd over the network** if enabled, or attach with the JDK tools from a debug container.
3. **Ephemeral debug container sharing the namespaces:**
   ```bash
   docker run -it --rm --pid=container:app --network=container:app \
     --cap-add SYS_PTRACE nicolaka/netshoot sh
   ```
   In Kubernetes: `kubectl debug -it pod/app --image=busybox --target=app`.
4. **Pull artifacts out** — `docker cp app:/dumps/heap.hprof .` and analyze offline in MAT.
5. **Reproduce in a `:debug` image variant** in a lower environment rather than weakening production.

Framing to use: "The absence of a shell forces good observability, which is the right trade — I'd rather invest in metrics and traces than in SSH-style debugging of prod."

---

### S26. "We're moving 15 Spring Boot services from VMs to containers. What's your plan and what breaks?"

**Plan** — inventory state and dependencies; standardize a base image and a Dockerfile template (multi-stage, non-root, layered jar); externalize all config to env/config server; add Actuator liveness/readiness to every service; build a shared CI pipeline template; containerize the least-stateful service first as a pilot; run in parallel with VM traffic shifting gradually; then migrate the rest, keeping the DB external.

**What typically breaks — and this is the valuable part of the answer:**
- **Local filesystem assumptions** — file uploads, generated reports, local caches → move to S3.
- **Sticky sessions / in-memory HttpSession** → Spring Session + Redis, or JWT.
- **Scheduled jobs** (`@Scheduled`) firing on every replica → use ShedLock or an external scheduler with leader election.
- **Local log files** and log-scraping scripts → stdout + central logging.
- **Hardcoded hostnames/IPs, `localhost` dependencies** → service discovery.
- **JVM heap flags tuned for a whole VM** (`-Xmx8g` on a 1 GB container) → `MaxRAMPercentage`.
- **Startup order assumptions** → app-level retry and readiness probes.
- **Long-lived state in caches (Ehcache local)** → distributed cache.
- **Graceful shutdown never tested** because VMs were rarely restarted → now happens on every deploy and autoscale event.
- **Ops runbooks** based on SSH and `tail -f` → new observability tooling and training.

---

### S27. "Container CPU is fine, memory is fine, but throughput collapses under load. Where do you look?"

Rule out container-specific causes first, then app:
1. **CPU throttling** despite low average usage (S15) — check `nr_throttled`.
2. **Connection pool exhaustion** — HikariCP `maximumPoolSize` too small for the thread count, or DB `max_connections` exceeded because you scaled to 20 replicas × 10 connections = 200. This is one of the most common containerization surprises: horizontal scaling multiplies DB connections. Use a pooler (PgBouncer) and size pools per replica deliberately.
3. **Tomcat thread starvation** — blocking calls to a slow downstream with no timeout; add timeouts + circuit breaker (Resilience4j) so failure is fast and bounded.
4. **GC pressure** — heap too small for the load; check GC logs for high allocation rate/long pauses.
5. **Network/conntrack limits** on the node, or DNS latency per request.
6. **Disk I/O** — logging synchronously to an overlayfs file.

Say the meta-point: containerization changes the *shape* of resource contention (many small instances sharing a node and a database), so capacity planning has to move from per-VM to per-replica × replica-count.

---

### S28. "A CVE with a CRITICAL rating was found in your base image. What do you do?"

1. **Assess reachability** — is the vulnerable package actually used by the app? A CVE in an OS utility that never runs is a lower practical risk than one in your TLS stack. Note it still needs remediation, but this drives urgency.
2. **Patch path** — bump to the latest patched base image tag, rebuild, rescan, run the test suite, deploy through the normal pipeline. With Buildpacks you can `rebase` to swap the OS layer without recompiling.
3. **If no patch exists** — mitigate: remove the package, switch to a slimmer/distroless base that doesn't include it, add network controls, or apply a virtual patch at the gateway.
4. **Fleet-wide** — query the registry/scanner for every image using that base, so you fix all 15 services rather than the one that got flagged.
5. **Prevent** — a small set of golden base images owned by the platform team, rebuilt weekly and re-scanned; renovate/dependabot PRs for base bumps; scanning on a schedule (not just at build) since new CVEs land against unchanged images; and an SBOM per image so you can answer "which services contain log4j 2.14?" in seconds during the next zero-day.

---

## 13. Rapid-Fire Revision & Cheat Sheets

### 13.1 One-line answers (read this on the way to the interview)

| Question | One-line answer |
|---|---|
| Image vs container | Class vs object |
| Why containers are light | Share the host kernel; no guest OS |
| Namespaces vs cgroups | What you can see vs how much you can use |
| CMD vs ENTRYPOINT | ENTRYPOINT always runs; CMD is overridable args |
| Exec vs shell form | Exec form makes the JVM PID 1 so it receives SIGTERM |
| COPY vs ADD | ADD also untars and fetches URLs; prefer COPY |
| ARG vs ENV | ARG = build only; ENV persists at runtime; neither for secrets |
| EXPOSE vs -p | EXPOSE documents; -p actually publishes |
| Volume vs bind mount | Docker-managed vs host path |
| Why not `latest` | Mutable tag → no reproducibility, no rollback |
| Exit 137 | SIGKILL → almost always OOMKilled |
| Exit 143 | SIGTERM → graceful stop |
| Multi-stage build | Build with JDK, ship on JRE |
| Layered jar | Only ~2 MB changes per deploy instead of 60 MB |
| MaxRAMPercentage | Container-aware heap sizing; default 25% is too low |
| Why non-root | Container root + kernel/volume issue = host compromise |
| depends_on | Waits for started, not ready — use `service_healthy` |
| Compose vs Kubernetes | Single-host dev tool vs multi-node orchestrator |
| Why smaller images | Faster deploys, fewer CVEs, cheaper storage |
| Where does state go | Volumes or external services — never the writable layer |

### 13.2 JVM-in-container flags to memorize

```
-XX:MaxRAMPercentage=75.0          # heap as % of container limit (must set!)
-XX:InitialRAMPercentage=50.0      # avoid heap resize churn at startup
-XX:MaxMetaspaceSize=256m          # bound non-heap class metadata
-XX:MaxDirectMemorySize=256m       # bound Netty/NIO buffers
-XX:ActiveProcessorCount=2         # correct thread pool sizing under CPU limits
-XX:+UseG1GC                       # good default; ZGC for large heaps/low pause
-XX:+ExitOnOutOfMemoryError        # fail fast so the orchestrator restarts cleanly
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps            # MUST be a mounted volume
-XX:ErrorFile=/dumps/hs_err_%p.log
-Xlog:gc*:file=/dumps/gc.log:time,uptime:filecount=5,filesize=20m
```

### 13.3 Command cheat sheet

```bash
# Images
docker build -t app:1.0 .            docker images
docker history app:1.0               docker pull/push repo/app:1.0
docker tag app:1.0 repo/app:1.0      docker rmi app:1.0

# Containers
docker run -d --name a -p 8080:8080 app:1.0
docker ps -a                         docker logs -f --tail 100 a
docker exec -it a sh                 docker stop/start/restart a
docker rm -f a                       docker inspect a
docker stats                         docker top a          docker cp a:/f .

# Volumes / Networks
docker volume ls|create|inspect|rm   docker network ls|create|inspect|connect

# Compose
docker compose up -d --build         docker compose logs -f svc
docker compose down [-v]             docker compose config

# JVM diagnostics inside a container (PID 1)
docker exec a jcmd 1 VM.flags
docker exec a jcmd 1 GC.heap_info
docker exec a jcmd 1 Thread.print
docker exec a jcmd 1 GC.heap_dump /dumps/h.hprof
docker exec a jcmd 1 VM.native_memory summary
```

### 13.4 The five things that make you sound senior rather than mid-level

1. **JVM ↔ cgroups**: you know the heap must be sized with `MaxRAMPercentage` and can explain non-heap footprint.
2. **Signals & graceful shutdown**: exec form ENTRYPOINT, SIGTERM, `server.shutdown=graceful`, grace period > shutdown timeout.
3. **Build once, promote by digest**: no rebuilds per environment, no `latest`, config injected externally.
4. **Layer semantics**: caching strategy for Maven, and the fact that deleted files persist in lower layers (secrets).
5. **You answer scenarios as a procedure** (Observe → Diagnose → Fix → Prevent) and always include the *prevent* step. Most candidates stop at "increase the memory limit".

### 13.5 Questions to ask your interviewer (shows seniority)

- Are you on Kubernetes or Compose/Swarm today, and what drove that choice?
- Do you build once and promote by digest, or rebuild per environment?
- Who owns base images and CVE patching — platform team or each service team?
- How are secrets delivered to containers today?
- What does your rollback look like when a bad image reaches prod?
- Are Testcontainers used, and how is Docker provided in CI?

---

### Final revision order if you only have one hour

1. Q27–Q29 (JVM memory & CPU in containers) — highest probability, highest signal.
2. Q14 + Q22 (ENTRYPOINT form & graceful shutdown).
3. Q30 + Q36 (layered jars & multi-stage builds) — be able to write the Dockerfile from memory.
4. Q17 (Maven build cache layering).
5. Scenarios S1, S2, S3, S4, S5, S6 — the six that come up most.
6. Q56 (security best practices, N-M-S-R-C-P-S).
7. Section 13.1 one-liners.

**Good luck. If you can write the hardened Dockerfile in Q57 on a whiteboard and explain every line, you're ready.**
