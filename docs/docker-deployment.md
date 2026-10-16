# RELATE — Docker Deployment Guide

> GitHub renders the diagrams below (Mermaid). Sections marked **[verified]**
> come straight from the RELATE repo or its
> [official docs](https://documen.tician.de/relate/misc.html); sections marked
> **[example]** are proposed Docker files to review before use.

**A note on the name "relate-courseware":** that is the name of RELATE's own
Python package (`pyproject.toml` → `name = relate-courseware`) — i.e. RELATE
*is* `relate-courseware`. **No separate project by that name exists.**
**[verified]** As of this writing, `relate-courseware` is **not published on
PyPI** (pypi.org returns 404), so the only supported install route is cloning
the git repository, per the official installation guide.

## 1. What you are deploying

RELATE is a Django web application (Django 6, Python ≥ 3.12) for hosting
courses. In production it runs as a set of cooperating services
**[verified]**:

| Component | Role | Verified detail |
|---|---|---|
| RELATE app | Django app, served by **Granian** | Entry point `granian relate.wsgi:application`; `relate/wsgi.py` already wraps the app with Granian's `wrap_wsgi_with_proxy_headers` so it works behind a reverse proxy |
| PostgreSQL | Database (recommended for production) | `DATABASES` in `local_settings.py` uses `django.db.backends.postgresql`; needs the `postgres` extra (`uv sync --extra postgres`) |
| memcached | Out-of-process cache (progress bars, performance) | `CACHES = {"default": {"BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache", "LOCATION": "host:11211"}}`; needs the `memcache` extra |
| RabbitMQ | Celery message broker | `CELERY_BROKER_URL` defaults to `"amqp://"` in `local_settings_example.py` |
| Celery workers | Long-running background tasks (grading, emails, course updates) | Started with `celery worker -A relate` |
| Docker daemon | Sandboxed execution of student Python code | RELATE talks to the Docker API (`RELATE_DOCKER_URL`, default `unix://var/run/docker.sock`) and pulls a runner image. ⚠️ The example settings use `inducer/relate-runcode-python-amd64`, while the docs say `docker pull inducer/relate-runpy-amd64` — verify which tag exists before relying on it |
| Course git repos | Course content (YAML/Markdown) versioned in git | Cloned with dulwich into `GIT_ROOT/<course-identifier>` when you use **"Set up new course"** in the web UI |
| Static files | JS/CSS, incl. MathJax | Built with `npm install && npm run build`, then assembled with `./collectstatic.sh` — **do not** run `python manage.py collectstatic` directly (official note: it fails on MathJax sourcemap URLs) |

Configuration lives in **`local_settings.py`** in the repo root (next to
`manage.py`). **[verified]** `relate/settings.py` *requires* this file and
raises `RuntimeError` if it is missing. Start from `local_settings_example.py`
and edit the values.

## 2. Architecture diagrams

### 2a. Simple local setup — no load balancer, no WAF

```mermaid
flowchart LR
    Browser([Browser<br/>localhost:8000]) -->|HTTP direct| App
    subgraph Docker host
        subgraph compose["docker compose"]
            App["relate app<br/>(Granian or runserver)<br/>port 8000 published"]
            DB[(PostgreSQL<br/>volume: pgdata)]
            MC[(memcached)]
            MQ[(RabbitMQ)]
            Celery["Celery worker<br/>celery worker -A relate"]
        end
        Dockerd{{Docker daemon<br/>/var/run/docker.sock}}
    end
    App --> DB
    App --> MC
    App --> MQ
    MQ --> Celery
    Celery --> DB
    App -.->|optional: sandbox<br/>student code| Dockerd
    style Browser fill:#e1f5ff
```

Traffic goes straight from your browser to the app container's published
port. The database persists in a named volume; everything else is disposable.

### 2b. Full production setup — TLS, WAF, scaled app tier

```mermaid
flowchart TB
    User([Users]) -->|HTTPS 443| Nginx
    subgraph Docker host / VM
        Nginx["nginx<br/>reverse proxy + TLS<br/>+ ModSecurity / OWASP CRS (WAF)<br/>serves /static and /media directly"]
        WAF{{"WAF: ModSecurity<br/>OWASP Core Rule Set"}}
        Nginx --> WAF
        WAF -->|upstream round-robin<br/>load balancing| App1
        WAF -->|upstream round-robin| App2
        App1["relate app × N<br/>granian relate.wsgi:application"]
        App2["relate app × N<br/>granian relate.wsgi:application"]
        Celery["Celery workers<br/>celery -A relate ... worker"]
        DB[(PostgreSQL<br/>volume: pgdata<br/>+ backups)]
        MC[(memcached)]
        MQ[(RabbitMQ<br/>volume: rabbitdata)]
        GitVol[(volumes:<br/>git-roots, bulk-storage,<br/>static)]
        Dockerd{{Docker daemon<br/>sandboxed code runs}}
        App1 --> DB
        App2 --> DB
        App1 --> MC
        App2 --> MC
        App1 --> MQ
        MQ --> Celery
        Celery --> DB
        App1 --> GitVol
        App2 --> GitVol
        Nginx -->|alias| GitVol
        App1 -.-> Dockerd
        App2 -.-> Dockerd
    end
    style User fill:#e1f5ff
```

nginx terminates TLS, applies WAF rules, load-balances across the app
replicas and serves `/static` and `/media` without touching Python — this
mirrors the official nginx deployment config.

### 2c. System flow — what happens per request

```mermaid
flowchart TB
    B([Browser]) -->|1. HTTPS| P[nginx + WAF]
    P -->|2. proxy_pass +<br/>X-Forwarded-* headers| G[Granian → Django]
    G -->|3a. session / ORM| DB[(PostgreSQL)]
    G -->|3b. cache lookups| C[(memcached)]
    G -->|3c. long task?| Q[RabbitMQ]
    Q -->|4. consume| W[Celery worker]
    W --> DB
    W -->|5. progress updates| C
    G -->|6a. course YAML| GR[(git-roots<br/>dulwich clones)]
    G -->|6b. code question?| D{{Docker daemon<br/>runner image}}
    D -->|7. result| G
    G -->|8. HTML| B
    Admin([Instructor]) -->|git push course repo| Upstream[(GitHub course repo)]
    Upstream -->|Set up new course /<br/>pull updates| GR
```

Key flows: interactive requests hit Django directly (3a/3b); anything slow
is queued in RabbitMQ and executed by Celery workers; course content always
comes from git checkouts under `GIT_ROOT` (via dulwich); student Python code
runs in throwaway Docker containers, never in the app process.

---

## 3. Setup A — simple local Docker (no load balancer, no WAF)

**Goal:** RELATE running on your machine with one command, reachable at
`http://localhost:8000`. Database, cache and task queue in containers.

### 3.1 Prerequisites

- Docker Engine + the Compose plugin (`docker compose version` works)
- git
- ~4 GB free disk (Python deps + frontend build are chunky)

### 3.2 Project layout **[example]**

Create a directory (e.g. `~/relate-docker`) with:

```
relate-docker/
├── Dockerfile
├── docker-compose.yml
└── local_settings.py      # copied from the repo's local_settings_example.py, then edited
```

### 3.3 `local_settings.py` — the values that matter **[verified names, example values]**

Copy `local_settings_example.py` out of the cloned repo and set at minimum:

```python
SECRET_KEY = "<generate one: python -c 'import secrets; print(secrets.token_urlsafe(50))'>"
DEBUG = True
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]   # example file defaults to relate.example.edu — change this!
RELATE_BASE_URL = "http://localhost:8000"

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "relate",
        "USER": "relate",
        "PASSWORD": "relate",      # example — use a .env file in real use
        "HOST": "db",              # the compose service name
        "PORT": "5432",
    }
}

CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": "memcached:11211",   # compose service name, not 127.0.0.1
    }
}

CELERY_BROKER_URL = "amqp://guest:guest@rabbitmq:5672//"

GIT_ROOT = "/srv/relate/git-roots"   # must be writable; backed by a volume below
```

Leave email at the defaults for local testing. Everything else in the example
file can stay as-is for a first run.

### 3.4 `Dockerfile` **[example]**

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_LINK_MODE=copy

# System deps: git (course clones), node (frontend build), postgres client libs
RUN apt-get update && apt-get install -y --no-install-recommends \
        git curl ca-certificates libpq-dev gcc \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /srv/relate
# The repo is cloned at build time (relate-courseware is not on PyPI)
RUN git clone --depth 1 https://github.com/inducer/relate.git .

# Python deps (mirrors the repo's own update.sh), incl. postgres + memcache extras
RUN uv sync --frozen --all-extras --no-group mypy

# Frontend build, then static files (mirrors update.sh / collectstatic.sh)
RUN npm install && npm run build

# local_settings.py is mounted at run time (see compose file)
EXPOSE 8000
# Served with Granian, configured via the GRANIAN_* env vars below
# (same names as the official systemd unit: GRANIAN_HOST, GRANIAN_PORT,
# GRANIAN_INTERFACE=wsgi, GRANIAN_WORKERS, ...)
CMD ["uv", "run", "granian", "relate.wsgi:application"]
```

> Granian is used here (instead of the dev `runserver`) so the local image
> behaves like production. It is configured with the `GRANIAN_*` env vars
> from the compose file — the same names as in the official systemd unit
> (`GRANIAN_HOST`, `GRANIAN_PORT`, `GRANIAN_INTERFACE=wsgi`).

### 3.5 `docker-compose.yml` **[example]**

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: relate
      POSTGRES_PASSWORD: relate
      POSTGRES_DB: relate
    volumes:
      - pgdata:/var/lib/postgresql/data

  memcached:
    image: memcached:1.6-alpine

  rabbitmq:
    image: rabbitmq:3-management-alpine

  app:
    build: .
    ports:
      - "8000:8000"     # direct port mapping — no load balancer, no WAF
    environment:
      GRANIAN_HOST: "0.0.0.0"
      GRANIAN_PORT: "8000"
      GRANIAN_INTERFACE: "wsgi"
      GRANIAN_WORKERS: "2"
    volumes:
      - ./local_settings.py:/srv/relate/local_settings.py:ro
      - gitroots:/srv/relate/git-roots
      - bulkstorage:/srv/relate/bulk-storage
      # Optional: enables sandboxed "code questions".
      # Needs the docker group / socket access on the host.
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - db
      - memcached
      - rabbitmq

  worker:
    build: .
    command: ["uv", "run", "celery", "worker", "-A", "relate", "--loglevel=INFO"]
    volumes:
      - ./local_settings.py:/srv/relate/local_settings.py:ro
      - gitroots:/srv/relate/git-roots
      - bulkstorage:/srv/relate/bulk-storage
    depends_on:
      - db
      - memcached
      - rabbitmq
      - app

volumes:
  pgdata:
  gitroots:
  bulkstorage:
```

### 3.6 First run — step by step

```bash
cd ~/relate-docker

# 1. Build the image
docker compose build

# 2. Initialize the database  [verified commands]
docker compose run --rm app uv run python manage.py migrate

# 3. Create your admin user
docker compose run --rm app uv run python manage.py createsuperuser --username=admin
# (you'll be prompted for email + password)

# 4. Assemble static files  [verified: use the repo's script, not raw collectstatic]
docker compose run --rm app ./collectstatic.sh

# 5. Start everything
docker compose up -d

# 6. Follow the logs until it's up
docker compose logs -f app
```

Open **http://localhost:8000** and sign in with the superuser you created.

### 3.7 Import the sample course

1. Sign in, then open **Set up new course** (top navigation).
2. Fill in (verified field names):
   - **Identifier:** `sample` (becomes the directory under `GIT_ROOT`)
   - **Name:** `RELATE sample course`
   - **Git source:** `https://github.com/inducer/relate-sample.git`
   - **Course file:** `course.yml`
   - Leave the events file as-is (the sample repo ships no `events.yml`).
3. Click **Validate and create**. RELATE clones the repo with dulwich,
   validates the YAML, and creates the course.
4. Back on the home page, click the course → you should see its content pages.

### 3.8 Verify it's working

- Home page loads, login works → app + DB OK
- Enroll in the sample course and open a quiz → cache + git content OK
- Start a flow / submit something that queues a task; check
  `docker compose logs worker` shows the task being consumed → Celery + RabbitMQ OK
- (If you mounted the docker socket) try a Python code question → sandbox OK

### 3.9 Common pitfalls (local)

| Symptom | Cause / fix |
|---|---|
| `DisallowedHost` error page | `ALLOWED_HOSTS` in `local_settings.py` doesn't include `localhost` — the example file defaults to `relate.example.edu` |
| `RuntimeError: 'local_settings.py' is missing` | The bind mount path is wrong; the file must land at `/srv/relate/local_settings.py` |
| `could not connect to server` on migrate | `DATABASES.HOST` must be `db` (the compose service name), not `127.0.0.1` |
| Tasks hang at "pending" | Worker not running or `CELERY_BROKER_URL` wrong; check `docker compose logs worker` |
| Static files 404 | You skipped `./collectstatic.sh`, or ran raw `manage.py collectstatic` (officially discouraged) |
| Port 8000 already in use | Change the left side of `"8000:8000"` |
| Code questions fail | Docker socket not mounted / permissions; code execution is optional for a first run |

---

## 4. Setup B — full production Docker

**Goal:** TLS-terminated, WAF-filtered, horizontally scalable RELATE behind
nginx, with persistent volumes and backups. This adapts the
[official deployment guide](https://documen.tician.de/relate/misc.html#deployment)
(systemd + nginx) to containers.

### 4.1 Architecture recap

See diagram **2b** above. Concretely:

- **nginx** — TLS termination (certbot), serves `/static` and `/media`
  directly, reverse-proxies everything else to the app replicas.
  Config adapted from the official nginx snippet **[verified]** with
  ModSecurity + OWASP CRS added as the WAF **[example]**.
- **app × N** — Granian serving `relate.wsgi:application`, with the
  `GRANIAN_*` env vars from the official systemd unit **[verified names]**.
  nginx `upstream` does the load balancing (round-robin).
- **worker** — `celery -A relate ... worker` (official: `celery worker -A relate`).
- **db / memcached / rabbitmq** — same as local, but with persistent volumes.
- **code sandbox** — optional dedicated Docker host or socket mount.

### 4.2 `local_settings.py` production values **[verified names]**

On top of the local version, change:

```python
DEBUG = False
SECRET_KEY = "<long random string, from a secret store — never commit>"
ALLOWED_HOSTS = ["courses.example.edu"]
RELATE_BASE_URL = "https://courses.example.edu"
USE_X_FORWARDED_HOST = True   # nginx sets X-Forwarded-Host (see config below)

# Serve exclusively via HTTPS:
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

DATABASES = { ... "HOST": "db", ... }      # real credentials via secrets
CACHES = { ... "LOCATION": "memcached:11211", ... }
CELERY_BROKER_URL = "amqp://<user>:<pass>@rabbitmq:5672//"

GIT_ROOT = "/srv/relate/git-roots"          # persistent volume, backed up
```

### 4.3 nginx + WAF **[example config, adapted from the official snippet]**

```nginx
# WAF: ModSecurity with the OWASP Core Rule Set in front of the app
upstream relate_app {
    server app1:8123;
    server app2:8123;
}

server {
    listen 80;
    server_name courses.example.edu;
    return 301 https://$server_name$request_uri;  # enforce https [verified pattern]
}

server {
    listen 443 ssl;
    server_name courses.example.edu;

    ssl_certificate     /etc/letsencrypt/live/courses.example.edu/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/courses.example.edu/privkey.pem;
    # [example] issued via: certbot --nginx -d courses.example.edu

    modsecurity on;                                   # [example] WAF
    modsecurity_rules_file /etc/nginx/modsec/main.conf;  # includes OWASP CRS

    client_max_body_size 100M;                        # [verified]

    location /static {
        alias /srv/relate/static;                     # [verified pattern]
    }
    location /media {
        alias /srv/relate/media;                      # [verified pattern]
    }

    location / {
        proxy_set_header Upgrade $http_upgrade;       # [verified]
        proxy_set_header X-Real-IP $remote_addr;      # [verified]
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # [verified]
        proxy_set_header X-Forwarded-Proto $scheme;   # [verified]
        proxy_set_header X-Forwarded-Host $host;      # [verified]
        proxy_read_timeout 300;                       # [verified]
        proxy_connect_timeout 30;                     # [verified]
        proxy_pass http://relate_app;
    }

    add_header X-Frame-Options SAMEORIGIN;            # [verified]
}
```

> WAF choice is yours: the snippet above uses ModSecurity + OWASP CRS
> compiled into the nginx image. Cloud WAFs (Cloudflare, AWS WAF) in front of
> this setup work the same way — RELATE only ever sees the proxied request.

### 4.4 `docker-compose.prod.yml` (key differences vs local) **[example]**

```yaml
services:
  nginx:
    image: <your nginx-with-modsecurity image>   # [example]
    ports: ["80:80", "443:443"]
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/relate.conf:ro
      - ./modsec:/etc/nginx/modsec:ro
      - static:/srv/relate/static:ro
      - letsencrypt:/etc/letsencrypt:ro
    depends_on: [app1, app2]

  app1: &app
    build: .
    environment:
      GRANIAN_HOST: "0.0.0.0"
      GRANIAN_PORT: "8123"
      GRANIAN_WORKERS: "4"            # scale per container; official VM example used 12
      GRANIAN_BLOCKING_THREADS: "4"   # [verified names]
      GRANIAN_WORKERS_MAX_RSS: "500"  # [verified names]
      GRANIAN_INTERFACE: "wsgi"       # [verified]
      GRANIAN_RESPAWN_FAILED_WORKERS: "true"  # [verified]
    volumes:
      - ./local_settings.prod.py:/srv/relate/local_settings.py:ro
      - gitroots:/srv/relate/git-roots
      - bulkstorage:/srv/relate/bulk-storage
      - static:/srv/relate/static
    depends_on: [db, memcached, rabbitmq]
  app2:
    <<: *app

  worker:
    build: .
    command: ["uv", "run", "celery", "-A", "relate", "worker",
              "--concurrency=4", "--loglevel=INFO"]   # [verified base command]
    volumes:
      - ./local_settings.prod.py:/srv/relate/local_settings.py:ro
      - gitroots:/srv/relate/git-roots
      - bulkstorage:/srv/relate/bulk-storage
    depends_on: [db, rabbitmq]

  db:
    image: postgres:16
    volumes: ["pgdata:/var/lib/postgresql/data"]
    # credentials via secrets / env_file — never in the compose file
  memcached:
    image: memcached:1.6-alpine
  rabbitmq:
    image: rabbitmq:3-management-alpine
    volumes: ["rabbitdata:/var/lib/rabbitmq"]

volumes:
  pgdata:        # back this up (pg_dump on a schedule)
  rabbitdata:
  gitroots:      # back this up (course content)
  bulkstorage:   # back this up (uploads)
  static:        # regenerated by collectstatic.sh; no backup needed
  letsencrypt:
```

### 4.5 Production bring-up — step by step

```bash
# 1. Point DNS for courses.example.edu at the host; open 80/443.
# 2. Prepare secrets: local_settings.prod.py, db password, rabbitmq password.
#    Keep them out of git (use docker secrets or an env_file).

# 3. Build + initialize (same verified commands as local)
docker compose -f docker-compose.prod.yml build
docker compose -f docker-compose.prod.yml run --rm app1 uv run python manage.py migrate
docker compose -f docker-compose.prod.yml run --rm app1 uv run python manage.py createsuperuser --username=admin
docker compose -f docker-compose.prod.yml run --rm app1 ./collectstatic.sh

# 4. Issue the TLS certificate (once; renew via cron/systemd timer)
docker compose -f docker-compose.prod.yml run --rm certbot certonly \
    --webroot -w /var/www/certbot -d courses.example.edu

# 5. Start
docker compose -f docker-compose.prod.yml up -d

# 6. Smoke test
curl -sI https://courses.example.edu/ | head -3
# expect: HTTP/2 200 and the login page at / in a browser
```

Then import courses exactly as in §3.7 (via the web UI), and set the
`GIT_ROOT` volume permissions so the app user can write
**[verified requirement]**.

### 4.6 Production operations checklist

- **Backups:** `pgdata`, `gitroots`, `bulkstorage` volumes on a schedule
  (plus `local_settings.prod.py` and TLS keys stored safely off-host).
- **Dulwich maintenance [verified]:** course repos accumulate loose git
  objects; run `git repack -a -d` on the `git-roots` occasionally (the repo
  ships `repack-repositories.sh` — adapt it into a cron job inside a
  maintenance container or on the volume host). Without this you can
  eventually hit `IOError: [Errno 24] Too many open files`.
- **Scaling:** add more `appN` replicas + raise `GRANIAN_WORKERS`;
  nginx upstream picks them up. Celery: raise `--concurrency` or add worker
  replicas.
- **Updates:** rebuild image from the new git ref, run `migrate`, run
  `./collectstatic.sh`, rolling-restart app containers.
- **Monitoring:** watch `docker compose logs worker` for stuck tasks;
  RELATE's admin shows flow/task status; put a simple HTTPS health check on
  `/` in your uptime monitor.
- **Maintenance mode:** `RELATE_MAINTENANCE_MODE = True` shows a site-wide
  banner **[verified setting]**.

---

## 5. What's verified vs what's proposed

| Item | Status |
|---|---|
| `relate-courseware` is RELATE's own package name; not on PyPI; install via `git clone` | ✅ verified (pyproject.toml, pypi.org 404, official install docs) |
| `local_settings.py` required; real setting names (`SECRET_KEY`, `ALLOWED_HOSTS`, `RELATE_BASE_URL`, `DATABASES`, `CACHES`, `CELERY_BROKER_URL`, `GIT_ROOT`, `USE_X_FORWARDED_HOST`, `DEBUG`, `*_COOKIE_SECURE`, `RELATE_DOCKER_*`, `RELATE_MAINTENANCE_MODE`) | ✅ verified (relate/settings.py, local_settings_example.py) |
| Granian serves `relate.wsgi:application`; `GRANIAN_*` env names | ✅ verified (official deployment docs, relate/wsgi.py) |
| nginx reverse-proxy config (headers, timeouts, 100M body, /static & /media aliases) | ✅ verified (official deployment docs) |
| `celery worker -A relate`; RabbitMQ via `amqp://` | ✅ verified (official docs, local_settings_example.py) |
| `./collectstatic.sh` (not raw `collectstatic`); `npm install && npm run build`; `uv sync` flags; `migrate` / `createsuperuser` | ✅ verified (repo scripts + official docs) |
| "Set up new course" flow: `identifier`, `git_source`, `course_file`; dulwich clone into `GIT_ROOT`; relate-sample has `course.yml` at root | ✅ verified (course/versioning.py, cloned relate-sample) |
| Dulwich repack maintenance | ✅ verified (official docs) |
| Dockerfile / compose files / ModSecurity-CRS WAF / certbot TLS / replica counts / volume names / `postgres:16` etc. | ⚠️ **proposed examples** — review before use |
| Sandbox image tag | ⚠️ unresolved: `inducer/relate-runcode-python-amd64` (settings) vs `inducer/relate-runpy-amd64` (docs) — confirm which exists on Docker Hub |

## 6. Open questions for the maintainer

1. Which sandbox image tag is correct on Docker Hub?
2. Is ModSecurity + OWASP CRS the WAF you want, or a cloud WAF (Cloudflare/AWS)?
3. Should the simple local setup include the docker-socket mount for code
   questions, or keep it minimal?
4. Domain + TLS strategy for production (Let's Encrypt assumed above).
5. Where should the Dockerfiles live — in this repo, or a separate
   `relate-docker` repo?
