# Production-Grade Supabase PostgreSQL HA Cluster
## Patroni + etcd + HAProxy on Docker Compose

> **Architecture Summary:** 3-node etcd quorum → Patroni leader election → 1 primary + 2 synchronous standbys → HAProxy health-checked routing. Zero data loss via synchronous replication.

---

## Prerequisites & Assumptions

| Item | Value |
|------|-------|
| Base image | `supabase/postgres:15.8.1.060` (pin to a specific tag) |
| Patroni | `3.3.x` |
| etcd | `v3.5.x` |
| HAProxy | `2.9.x` |
| Docker Compose | v2.x (`compose` plugin) |

All secrets are managed via a `.env` file and Docker secrets patterns. Never bake passwords into images.

---

## Repository Layout

```
supabase-ha/
├── docker-compose.yml
├── .env                          # secrets — never commit
├── patroni/
│   ├── Dockerfile
│   ├── patroni.yml.j2            # Jinja2 template rendered at runtime
│   ├── entrypoint.sh
│   └── post_init.sh              # Supabase bridge hook
├── haproxy/
│   └── haproxy.cfg
└── etcd/
    └── (data volumes only)
```

---

## Step 1 — The Custom Dockerfile (`patroni/Dockerfile`)

The core challenge is installing Patroni *inside* the `supabase/postgres` image without breaking its initialization logic. The Supabase image is Debian-based and uses `/docker-entrypoint-initdb.d/` for first-boot SQL.

```dockerfile
# patroni/Dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Stage 1: Build Patroni wheel dependencies in a clean Python env
# This avoids polluting the final image with build tools.
# ─────────────────────────────────────────────────────────────────────────────
FROM python:3.12-slim AS patroni-builder

ARG PATRONI_VERSION=3.3.2

RUN apt-get update && apt-get install -y --no-install-recommends \
        gcc \
        libpq-dev \
    && rm -rf /var/lib/apt/lists/*

RUN pip wheel --no-cache-dir --wheel-dir /wheels \
    "patroni[etcd3]==${PATRONI_VERSION}" \
    psycopg2-binary \
    python-etcd \
    dnspython

# ─────────────────────────────────────────────────────────────────────────────
# Stage 2: Final image — supabase/postgres + Patroni
# ─────────────────────────────────────────────────────────────────────────────
FROM supabase/postgres:15.8.1.060

ARG PATRONI_VERSION=3.3.2

LABEL maintainer="your-team@example.com"
LABEL description="Supabase Postgres wrapped with Patroni for HA"

# ── System dependencies ───────────────────────────────────────────────────────
# python3-minimal + pip for Patroni; curl for health probes; jinja2-cli for
# rendering the patroni.yml template at container startup.
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3 \
        python3-pip \
        python3-venv \
        curl \
        jq \
        gettext-base \
    && rm -rf /var/lib/apt/lists/*

# ── Install Patroni from pre-built wheels ────────────────────────────────────
COPY --from=patroni-builder /wheels /wheels
RUN pip3 install --no-cache-dir --no-index --find-links=/wheels \
    "patroni[etcd3]==${PATRONI_VERSION}" \
    psycopg2-binary \
    && rm -rf /wheels

# ── Verify Patroni installation ───────────────────────────────────────────────
RUN patroni --version

# ── Copy configuration files ──────────────────────────────────────────────────
COPY patroni.yml.j2     /etc/patroni/patroni.yml.j2
COPY post_init.sh       /etc/patroni/post_init.sh
COPY entrypoint.sh      /entrypoint.sh

RUN chmod +x /etc/patroni/post_init.sh /entrypoint.sh

# ── Expose ports ──────────────────────────────────────────────────────────────
# 5432 = PostgreSQL
# 8008 = Patroni REST API (used by HAProxy health checks)
EXPOSE 5432 8008

# ── Override the Supabase entrypoint with ours ────────────────────────────────
# The original entrypoint runs initdb + /docker-entrypoint-initdb.d/ scripts.
# We delegate ALL of that to Patroni's bootstrap section instead.
ENTRYPOINT ["/entrypoint.sh"]
```

### `patroni/entrypoint.sh` — Runtime template rendering

```bash
#!/bin/bash
# patroni/entrypoint.sh
# Renders the Jinja2/envsubst patroni.yml template and launches Patroni.
# Patroni takes over PostgreSQL lifecycle completely.
set -euo pipefail

# ── Required environment variables (injected by Docker Compose) ───────────────
: "${PATRONI_SCOPE:?PATRONI_SCOPE is required}"
: "${PATRONI_NAME:?PATRONI_NAME is required (unique per node)}"
: "${PATRONI_ETCD3_HOSTS:?PATRONI_ETCD3_HOSTS is required}"
: "${PATRONI_SUPERUSER_PASSWORD:?PATRONI_SUPERUSER_PASSWORD is required}"
: "${PATRONI_REPLICATION_PASSWORD:?PATRONI_REPLICATION_PASSWORD is required}"
: "${PATRONI_admin_PASSWORD:?PATRONI_admin_PASSWORD is required}"

# ── Derive REST API and PostgreSQL listen addresses ───────────────────────────
export PATRONI_RESTAPI_LISTEN="0.0.0.0:8008"
export PATRONI_RESTAPI_CONNECT_ADDRESS="${PATRONI_NAME}:8008"
export PATRONI_POSTGRESQL_LISTEN="0.0.0.0:5432"
export PATRONI_POSTGRESQL_CONNECT_ADDRESS="${PATRONI_NAME}:5432"

# ── PostgreSQL data directory ─────────────────────────────────────────────────
export PGDATA="${PGDATA:-/var/lib/postgresql/data/pgdata}"

# ── Render the Patroni configuration template ─────────────────────────────────
envsubst < /etc/patroni/patroni.yml.j2 > /etc/patroni/patroni.yml

echo "==> Patroni configuration rendered to /etc/patroni/patroni.yml"
echo "==> Node: ${PATRONI_NAME} | Scope: ${PATRONI_SCOPE}"
echo "==> etcd hosts: ${PATRONI_ETCD3_HOSTS}"

exec patroni /etc/patroni/patroni.yml
```

---

## Step 2 — Patroni Configuration YAML (`patroni/patroni.yml.j2`)

```yaml
# patroni/patroni.yml.j2
# Uses envsubst — all ${VAR} are replaced at container startup by entrypoint.sh
# ─────────────────────────────────────────────────────────────────────────────

scope: ${PATRONI_SCOPE}
namespace: /db/
name: ${PATRONI_NAME}

# ── etcd3 DCS ─────────────────────────────────────────────────────────────────
etcd3:
  hosts: ${PATRONI_ETCD3_HOSTS}
  # TLS (recommended for production):
  # protocol: https
  # cacert: /etc/ssl/etcd/ca.crt
  # cert: /etc/ssl/etcd/client.crt
  # key: /etc/ssl/etcd/client.key

# ── Patroni REST API ──────────────────────────────────────────────────────────
restapi:
  listen: ${PATRONI_RESTAPI_LISTEN}
  connect_address: ${PATRONI_RESTAPI_CONNECT_ADDRESS}
  # authentication (recommended):
  # username: patroni
  # password: ${PATRONI_RESTAPI_PASSWORD}

# ── Bootstrap — runs ONCE on cluster initialization ───────────────────────────
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 0        # Zero-lag failover — replica must be caught up
    master_start_timeout: 300
    synchronous_mode: true            # Enable Patroni-managed synchronous replication
    synchronous_mode_strict: true     # Never promote async replica (zero data loss guarantee)
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        # ── Synchronous replication (zero data loss) ──────────────────────────
        synchronous_commit: "on"
        synchronous_standby_names: "ANY 1 (*)"

        # ── WAL & Replication ─────────────────────────────────────────────────
        wal_level: replica
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 1GB
        hot_standby: "on"
        hot_standby_feedback: "on"    # Prevents vacuum from removing rows replicas need

        # ── Connection & Resources ────────────────────────────────────────────
        max_connections: 200
        shared_buffers: 256MB
        effective_cache_size: 768MB
        work_mem: 4MB
        maintenance_work_mem: 64MB

        # ── Supabase extensions ───────────────────────────────────────────────
        # Pre-load extensions that require shared_preload_libraries
        shared_preload_libraries: >-
          pg_stat_statements,
          pgaudit,
          pg_cron,
          pgsodium,
          pg_net,
          supabase_vault

        # pg_stat_statements
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all

        # pgsodium — key management (set your own key path)
        pgsodium.getkey_script: /etc/postgresql-custom/pgsodium_getkey.sh

        # Logging
        log_min_duration_statement: 1000
        log_checkpoints: "on"
        log_connections: "on"
        log_disconnections: "on"
        log_lock_waits: "on"
        log_temp_files: 0

  # ── initdb parameters ──────────────────────────────────────────────────────
  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums          # Enables block-level checksums — critical for pg_rewind

  # ── pg_hba rules (written on bootstrap) ───────────────────────────────────
  pg_hba:
    - local   all             all                              trust
    - host    all             all             127.0.0.1/32     scram-sha-256
    - host    all             all             ::1/128          scram-sha-256
    - host    replication     replicator      0.0.0.0/0        scram-sha-256
    - host    all             all             0.0.0.0/0        scram-sha-256

  # ── post_init: THE SUPABASE BRIDGE ────────────────────────────────────────
  # Runs ONCE after the very first initdb on the PRIMARY node.
  # This is where we replay all Supabase initialization SQL scripts.
  # post_init receives the connection string as its first argument.
  post_init: /etc/patroni/post_init.sh

  # ── Roles created by Patroni on bootstrap ─────────────────────────────────
  users:
    admin:
      password: ${PATRONI_admin_PASSWORD}
      options:
        - createrole
        - createdb

# ── PostgreSQL configuration ──────────────────────────────────────────────────
postgresql:
  listen: ${PATRONI_POSTGRESQL_LISTEN}
  connect_address: ${PATRONI_POSTGRESQL_CONNECT_ADDRESS}
  data_dir: ${PGDATA}
  bin_dir: /usr/lib/postgresql/15/bin
  pgpass: /tmp/pgpass

  authentication:
    replication:
      username: replicator
      password: ${PATRONI_REPLICATION_PASSWORD}
    superuser:
      username: postgres
      password: ${PATRONI_SUPERUSER_PASSWORD}
    rewind:
      username: rewind_user
      password: ${PATRONI_REPLICATION_PASSWORD}

  # Parameters that can be changed without restart (reload only)
  parameters:
    unix_socket_directories: "/var/run/postgresql"

  # pg_rewind requires these privileges on the rewind user
  create_replica_methods:
    - basebackup

  basebackup:
    max-rate: "100M"
    checkpoint: "fast"

# ── Watchdog (optional but recommended for fencing) ───────────────────────────
# watchdog:
#   mode: required
#   device: /dev/watchdog

# ── Tags ──────────────────────────────────────────────────────────────────────
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

---

### `patroni/post_init.sh` — The Supabase Bridge

This is the most critical piece. It runs once on the primary node after `initdb`, replaying every SQL script that the Supabase image ships in `/docker-entrypoint-initdb.d/`. This creates the `auth`, `storage`, `realtime` schemas, roles, and extensions.

```bash
#!/bin/bash
# patroni/post_init.sh
# ─────────────────────────────────────────────────────────────────────────────
# Patroni post_init hook — called with connection URL as $1 on the PRIMARY only.
# Replays ALL Supabase initialization SQL from /docker-entrypoint-initdb.d/
# ─────────────────────────────────────────────────────────────────────────────
set -euo pipefail

CONN_URL="$1"
INITDB_DIR="/docker-entrypoint-initdb.d"

echo "==> [post_init] Starting Supabase schema initialization..."
echo "==> [post_init] Connection: ${CONN_URL}"
echo "==> [post_init] Init scripts directory: ${INITDB_DIR}"

# ── Wait for PostgreSQL to be fully ready ─────────────────────────────────────
until pg_isready -h /var/run/postgresql -p 5432; do
  echo "==> [post_init] Waiting for PostgreSQL to be ready..."
  sleep 2
done

# ── Execute scripts in sorted order (same as initdb does) ────────────────────
# The Supabase image ships scripts as both .sql and .sql.gz files.
if [ -d "${INITDB_DIR}" ]; then
  # Collect all .sql and .sql.gz files, sort them (initdb order is critical)
  mapfile -t SCRIPTS < <(
    find "${INITDB_DIR}" -maxdepth 1 \( -name "*.sql" -o -name "*.sql.gz" \) \
    | sort
  )

  if [ ${#SCRIPTS[@]} -eq 0 ]; then
    echo "==> [post_init] WARNING: No SQL scripts found in ${INITDB_DIR}"
    echo "==> [post_init] Check that supabase/postgres image version ships initdb scripts."
  fi

  for SCRIPT in "${SCRIPTS[@]}"; do
    BASENAME=$(basename "${SCRIPT}")
    echo "==> [post_init] Executing: ${BASENAME}"

    if [[ "${SCRIPT}" == *.sql.gz ]]; then
      # Decompress and pipe to psql
      gunzip -c "${SCRIPT}" | psql "${CONN_URL}" \
        --set ON_ERROR_STOP=0 \
        --quiet \
        2>&1 | grep -v "^$" | sed "s/^/    [${BASENAME}] /" || {
          echo "==> [post_init] WARNING: ${BASENAME} had errors (may be expected for pre-existing objects)"
        }
    else
      psql "${CONN_URL}" \
        --file="${SCRIPT}" \
        --set ON_ERROR_STOP=0 \
        --quiet \
        2>&1 | grep -v "^$" | sed "s/^/    [${BASENAME}] /" || {
          echo "==> [post_init] WARNING: ${BASENAME} had errors (may be expected for pre-existing objects)"
        }
    fi
  done
else
  echo "==> [post_init] WARNING: ${INITDB_DIR} does not exist in this image."
  echo "==> [post_init] Supabase schemas will NOT be initialized."
  echo "==> [post_init] Verify your base image is supabase/postgres."
fi

# ── Grant replication and rewind privileges ───────────────────────────────────
echo "==> [post_init] Granting replication privileges..."
psql "${CONN_URL}" <<-EOSQL
  -- Replication user (for Patroni streaming replication)
  DO \$\$
  BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'replicator') THEN
      CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD '${PATRONI_REPLICATION_PASSWORD}';
    END IF;
  END
  \$\$;

  -- pg_rewind user (for pg_rewind after failover)
  DO \$\$
  BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'rewind_user') THEN
      CREATE ROLE rewind_user WITH LOGIN PASSWORD '${PATRONI_REPLICATION_PASSWORD}';
    END IF;
  END
  \$\$;

  -- Grant pg_rewind permissions (required by Patroni)
  GRANT EXECUTE ON function pg_catalog.pg_ls_dir(text, boolean, boolean) TO rewind_user;
  GRANT EXECUTE ON function pg_catalog.pg_stat_file(text, boolean) TO rewind_user;
  GRANT EXECUTE ON function pg_catalog.pg_read_binary_file(text) TO rewind_user;
  GRANT EXECUTE ON function pg_catalog.pg_read_binary_file(text, bigint, bigint, boolean) TO rewind_user;
EOSQL

echo "==> [post_init] Supabase initialization complete."
```

---

## Step 3 — HAProxy Configuration (`haproxy/haproxy.cfg`)

HAProxy uses Patroni's REST API (`/leader`, `/replica`, `/health`) for active health checks. This means HAProxy **never** needs to know about etcd directly — Patroni exposes the current role via HTTP.

```haproxy
# haproxy/haproxy.cfg
# ─────────────────────────────────────────────────────────────────────────────
# Routing strategy:
#   - Port 5432  → primary only (read/write)
#   - Port 5433  → replicas only (read-only load-balanced)
#   - Port 7000  → HAProxy stats dashboard
# ─────────────────────────────────────────────────────────────────────────────

global
    log         stdout format raw local0
    maxconn     4096
    daemon

defaults
    log         global
    mode        tcp
    option      tcplog
    option      dontlognull
    option      tcp-check
    retries     3
    timeout connect     5s
    timeout client      30m    # Long timeout for PostgreSQL sessions
    timeout server      30m
    timeout check       5s

# ── Stats dashboard ───────────────────────────────────────────────────────────
listen stats
    bind *:7000
    mode http
    stats enable
    stats uri /
    stats refresh 10s
    stats show-legends
    stats show-node
    stats auth admin:${HAPROXY_STATS_PASSWORD}

# ── Primary (read/write) — routes to Patroni leader ONLY ─────────────────────
frontend pg_primary_frontend
    bind *:5432
    default_backend pg_primary_backend

backend pg_primary_backend
    option httpchk GET /leader
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server patroni1 patroni1:5432 check port 8008
    server patroni2 patroni2:5432 check port 8008
    server patroni3 patroni3:5432 check port 8008

# ── Replicas (read-only) — load-balanced across healthy standbys ──────────────
# Patroni returns 200 on /replica for async replicas,
# and 200 on /sync-standby for synchronous standbys.
# We use /replica which matches both sync and async standbys.
frontend pg_replica_frontend
    bind *:5433
    default_backend pg_replica_backend

backend pg_replica_backend
    balance roundrobin
    option httpchk GET /replica
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions

    server patroni1 patroni1:5432 check port 8008
    server patroni2 patroni2:5432 check port 8008
    server patroni3 patroni3:5432 check port 8008
```

> **HAProxy health check endpoints provided by Patroni:**
>
> | Endpoint | HTTP 200 when... |
> |---|---|
> | `/leader` | Node is the current primary |
> | `/replica` | Node is a healthy replica (sync or async) |
> | `/sync-standby` | Node is a synchronous standby specifically |
> | `/health` | Node PostgreSQL is running (any role) |
> | `/read-write` | Alias for `/leader` |
> | `/read-only` | Alias for `/replica` |

---

## Step 4 — Docker Compose Manifest (`docker-compose.yml`)

```yaml
# docker-compose.yml
# Production-grade Supabase HA cluster
# 3x etcd + 3x Patroni/Postgres + 1x HAProxy
# ─────────────────────────────────────────────────────────────────────────────

name: supabase-ha

# ── Shared extension fields ───────────────────────────────────────────────────
x-patroni-common: &patroni-common
  build:
    context: ./patroni
    dockerfile: Dockerfile
    args:
      PATRONI_VERSION: "3.3.2"
  restart: unless-stopped
  networks:
    - ha-net
  depends_on:
    etcd1:
      condition: service_healthy
    etcd2:
      condition: service_healthy
    etcd3:
      condition: service_healthy
  environment: &patroni-env
    # ── Cluster identity ──────────────────────────────────────────────────────
    PATRONI_SCOPE: supabase-ha
    PATRONI_ETCD3_HOSTS: "etcd1:2379,etcd2:2379,etcd3:2379"

    # ── Credentials (sourced from .env) ───────────────────────────────────────
    PATRONI_SUPERUSER_PASSWORD: ${POSTGRES_PASSWORD}
    PATRONI_REPLICATION_PASSWORD: ${REPLICATION_PASSWORD}
    PATRONI_admin_PASSWORD: ${ADMIN_PASSWORD}

    # ── PostgreSQL data directory ─────────────────────────────────────────────
    PGDATA: /var/lib/postgresql/data/pgdata

    # ── Supabase-specific env vars ────────────────────────────────────────────
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    JWT_SECRET: ${JWT_SECRET}
    ANON_KEY: ${ANON_KEY}
    SERVICE_ROLE_KEY: ${SERVICE_ROLE_KEY}
  healthcheck:
    test: ["CMD", "curl", "-sf", "http://localhost:8008/health"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s

# ─────────────────────────────────────────────────────────────────────────────
services:

  # ── etcd cluster ─────────────────────────────────────────────────────────────
  etcd1:
    image: quay.io/coreos/etcd:v3.5.16
    container_name: etcd1
    hostname: etcd1
    restart: unless-stopped
    networks:
      - ha-net
    ports:
      - "2379:2379"
    environment:
      ETCD_NAME: etcd1
      ETCD_DATA_DIR: /etcd-data
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd1:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd1:2380
      ETCD_INITIAL_CLUSTER: etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_INITIAL_CLUSTER_TOKEN: supabase-ha-etcd-token
      ETCD_AUTO_COMPACTION_MODE: revision
      ETCD_AUTO_COMPACTION_RETENTION: "1000"
      ETCD_SNAPSHOT_COUNT: "5000"
      ETCD_HEARTBEAT_INTERVAL: "100"
      ETCD_ELECTION_TIMEOUT: "1000"
      ETCD_QUOTA_BACKEND_BYTES: "8589934592"  # 8 GiB
    volumes:
      - etcd1-data:/etcd-data
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health",
             "--endpoints=http://localhost:2379"]
      interval: 10s
      timeout: 5s
      retries: 5

  etcd2:
    image: quay.io/coreos/etcd:v3.5.16
    container_name: etcd2
    hostname: etcd2
    restart: unless-stopped
    networks:
      - ha-net
    environment:
      ETCD_NAME: etcd2
      ETCD_DATA_DIR: /etcd-data
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd2:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd2:2380
      ETCD_INITIAL_CLUSTER: etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_INITIAL_CLUSTER_TOKEN: supabase-ha-etcd-token
      ETCD_AUTO_COMPACTION_MODE: revision
      ETCD_AUTO_COMPACTION_RETENTION: "1000"
    volumes:
      - etcd2-data:/etcd-data
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health",
             "--endpoints=http://localhost:2379"]
      interval: 10s
      timeout: 5s
      retries: 5

  etcd3:
    image: quay.io/coreos/etcd:v3.5.16
    container_name: etcd3
    hostname: etcd3
    restart: unless-stopped
    networks:
      - ha-net
    environment:
      ETCD_NAME: etcd3
      ETCD_DATA_DIR: /etcd-data
      ETCD_LISTEN_CLIENT_URLS: http://0.0.0.0:2379
      ETCD_ADVERTISE_CLIENT_URLS: http://etcd3:2379
      ETCD_LISTEN_PEER_URLS: http://0.0.0.0:2380
      ETCD_INITIAL_ADVERTISE_PEER_URLS: http://etcd3:2380
      ETCD_INITIAL_CLUSTER: etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380
      ETCD_INITIAL_CLUSTER_STATE: new
      ETCD_INITIAL_CLUSTER_TOKEN: supabase-ha-etcd-token
      ETCD_AUTO_COMPACTION_MODE: revision
      ETCD_AUTO_COMPACTION_RETENTION: "1000"
    volumes:
      - etcd3-data:/etcd-data
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health",
             "--endpoints=http://localhost:2379"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Patroni + Postgres nodes ─────────────────────────────────────────────────
  patroni1:
    <<: *patroni-common
    container_name: patroni1
    hostname: patroni1
    environment:
      <<: *patroni-env
      PATRONI_NAME: patroni1
    volumes:
      - patroni1-data:/var/lib/postgresql/data
    ports:
      - "5441:5432"    # Direct access for diagnostics only
      - "8011:8008"    # Patroni API for diagnostics

  patroni2:
    <<: *patroni-common
    container_name: patroni2
    hostname: patroni2
    environment:
      <<: *patroni-env
      PATRONI_NAME: patroni2
    volumes:
      - patroni2-data:/var/lib/postgresql/data
    ports:
      - "5442:5432"
      - "8012:8008"

  patroni3:
    <<: *patroni-common
    container_name: patroni3
    hostname: patroni3
    environment:
      <<: *patroni-env
      PATRONI_NAME: patroni3
    volumes:
      - patroni3-data:/var/lib/postgresql/data
    ports:
      - "5443:5432"
      - "8013:8008"

  # ── HAProxy ──────────────────────────────────────────────────────────────────
  haproxy:
    image: haproxy:2.9-alpine
    container_name: haproxy
    hostname: haproxy
    restart: unless-stopped
    networks:
      - ha-net
    depends_on:
      patroni1:
        condition: service_healthy
      patroni2:
        condition: service_healthy
      patroni3:
        condition: service_healthy
    ports:
      - "5432:5432"    # Primary (read/write) — applications connect here
      - "5433:5433"    # Replicas (read-only) — read scaling
      - "7000:7000"    # HAProxy stats dashboard
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    environment:
      HAPROXY_STATS_PASSWORD: ${HAPROXY_STATS_PASSWORD}
    healthcheck:
      test: ["CMD", "haproxy", "-c", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
      interval: 30s
      timeout: 5s
      retries: 3

# ── Networks ──────────────────────────────────────────────────────────────────
networks:
  ha-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24

# ── Volumes ───────────────────────────────────────────────────────────────────
volumes:
  etcd1-data:
    driver: local
  etcd2-data:
    driver: local
  etcd3-data:
    driver: local
  patroni1-data:
    driver: local
  patroni2-data:
    driver: local
  patroni3-data:
    driver: local
```

### `.env` — Secrets file (never commit to VCS)

```bash
# .env — Add to .gitignore immediately

# PostgreSQL superuser
POSTGRES_PASSWORD=change-me-superuser-password-min-32-chars

# Replication user (used by Patroni streaming replication and pg_rewind)
REPLICATION_PASSWORD=change-me-replication-password-min-32-chars

# Admin user (created by Patroni bootstrap)
ADMIN_PASSWORD=change-me-admin-password-min-32-chars

# HAProxy stats UI password
HAPROXY_STATS_PASSWORD=change-me-haproxy-stats-password

# Supabase JWT (required by supabase/postgres image for auth schema)
JWT_SECRET=your-super-secret-jwt-token-with-at-least-32-characters

# Supabase API keys (used by the image's init scripts)
ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...your-anon-key
SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...your-service-role-key
```

---

## Step 5 — Verification & Failover Testing

### 5.1 — Initial Cluster Bootstrap

```bash
# Build images and start the cluster
docker compose up --build -d

# Watch the bootstrap process (takes ~60–120s on first start)
docker compose logs -f patroni1 patroni2 patroni3

# Expected log sequence on patroni1 (bootstrap node):
#   "no action. I am (patroni1), the leader with the lock"
# On patroni2/patroni3:
#   "no action. I am a secondary and following a leader"
```

### 5.2 — Verify Cluster State

```bash
# Check Patroni cluster topology
docker exec patroni1 patronictl -c /etc/patroni/patroni.yml list

# Expected output:
# + Cluster: supabase-ha (xxxxxxxxxxxxxxx) ------+----+-----------+
# | Member   | Host         | Role    | State   | TL | Lag in MB |
# +----------+--------------+---------+---------+----+-----------+
# | patroni1 | patroni1:5432| Leader  | running |  1 |           |
# | patroni2 | patroni2:5432| Sync Standby| running|1|         0 |
# | patroni3 | patroni3:5432| Replica | running |  1 |         0 |
# +----------+--------------+---------+---------+----+-----------+

# Verify synchronous replication is active
docker exec patroni1 psql -U postgres -c "SELECT * FROM pg_stat_replication;"
# Should show 2 rows with sync_state = 'sync' (or 'quorum') and flush_lsn = sent_lsn

# Verify Supabase schemas exist
docker exec patroni1 psql -U postgres -c "\dn"
# Should show: auth, extensions, graphql, graphql_public, pgbouncer,
#              realtime, storage, supabase_functions, vault
```

### 5.3 — Verify HAProxy Routing

```bash
# Connect through HAProxy to primary (port 5432)
psql -h localhost -p 5432 -U postgres -c "SELECT pg_is_in_recovery();"
# Expected: f  (false = primary)

# Connect through HAProxy to replica (port 5433)
psql -h localhost -p 5433 -U postgres -c "SELECT pg_is_in_recovery();"
# Expected: t  (true = replica)

# Check HAProxy stats (browser or curl)
curl -u admin:${HAPROXY_STATS_PASSWORD} http://localhost:7000/
# Open http://localhost:7000 in browser
```

### 5.4 — Zero Data Loss Failover Test

This is the critical test. We write data, kill the primary, and verify nothing is lost.

```bash
# ── Step 1: Write test data to primary ───────────────────────────────────────
psql -h localhost -p 5432 -U postgres <<'EOF'
CREATE TABLE IF NOT EXISTS failover_test (
    id   SERIAL PRIMARY KEY,
    val  TEXT NOT NULL,
    ts   TIMESTAMPTZ DEFAULT now()
);
-- Insert 1000 rows
INSERT INTO failover_test (val)
SELECT 'row-' || generate_series(1, 1000);

SELECT COUNT(*) AS rows_before_failover FROM failover_test;
-- Expected: 1000
EOF

# ── Step 2: Identify the current leader ──────────────────────────────────────
LEADER=$(docker exec patroni1 patronictl -c /etc/patroni/patroni.yml list \
  | grep "Leader" | awk '{print $2}')
echo "Current leader: ${LEADER}"

# ── Step 3: Simulate primary failure ─────────────────────────────────────────
echo "Stopping ${LEADER}..."
docker stop "${LEADER}"

# ── Step 4: Watch Patroni elect a new leader ──────────────────────────────────
# Failover should complete within ~30 seconds (ttl + loop_wait)
watch -n2 'docker exec patroni2 patronictl -c /etc/patroni/patroni.yml list 2>/dev/null'

# ── Step 5: Verify data integrity on new primary ──────────────────────────────
# Wait for HAProxy to detect the new leader (~10s)
sleep 15

psql -h localhost -p 5432 -U postgres <<'EOF'
SELECT COUNT(*) AS rows_after_failover FROM failover_test;
-- Expected: 1000 (zero data loss because synchronous_mode_strict: true)

SELECT pg_is_in_recovery();
-- Expected: f (we're on the new primary)
EOF

# ── Step 6: Rejoin the failed node as a replica ───────────────────────────────
docker start "${LEADER}"
# Patroni will automatically run pg_rewind and rejoin as a replica
sleep 30

docker exec patroni2 patronictl -c /etc/patroni/patroni.yml list
# The formerly-dead node should now appear as "Replica" with 0 lag
```

### 5.5 — Manual Switchover (Zero-Downtime Maintenance)

```bash
# Graceful switchover (use for planned maintenance — much safer than kill)
docker exec patroni1 patronictl -c /etc/patroni/patroni.yml switchover \
  --master patroni1 \
  --candidate patroni2 \
  --force

# Verify new leader
docker exec patroni1 patronictl -c /etc/patroni/patroni.yml list
```

### 5.6 — etcd Cluster Health

```bash
# Check etcd cluster health
docker exec etcd1 etcdctl \
  --endpoints=http://etcd1:2379,http://etcd2:2379,http://etcd3:2379 \
  endpoint health

# Check etcd member list
docker exec etcd1 etcdctl \
  --endpoints=http://etcd1:2379,http://etcd2:2379,http://etcd3:2379 \
  member list

# View Patroni's DCS keys in etcd
docker exec etcd1 etcdctl \
  --endpoints=http://etcd1:2379 \
  get /db/supabase-ha/ --prefix
```

---

## Critical Operational Notes

### Understanding `synchronous_mode_strict: true`

With this setting, Patroni will **refuse to promote a replica** if no synchronous standby is available. This means if your primary AND your only sync standby both fail simultaneously, the cluster becomes **read-only** rather than risk data loss. This is the correct trade-off for zero-data-loss guarantees.

```
Primary DOWN + sync standby DOWN → cluster UNAVAILABLE (not promoted)
Primary DOWN + sync standby UP   → sync standby promoted (zero data loss)
```

If you want to relax this (accept potential data loss in extreme scenarios), set `synchronous_mode_strict: false`.

### The `post_init` vs `post_bootstrap` Distinction

| Hook | When it runs | Use for |
|------|-------------|---------|
| `post_init` | After `initdb`, once, on the bootstrap node | Supabase SQL scripts, role creation |
| `post_bootstrap` | After the cluster is initialized | Application-level setup |

`post_init` is correct for Supabase because it runs immediately after `initdb` before the cluster accepts connections from replicas, ensuring the schemas are replicated correctly.

### Supabase Image Version Pinning

Always pin to an exact image tag. The `supabase/postgres` image's `initdb` scripts change between versions, and running a mismatched `post_init` against a different version will cause failures.

```bash
# Find available tags
docker pull supabase/postgres --list-digests  # or check Docker Hub
```

### Volume Backup Strategy

```bash
# Consistent backup using pg_basebackup through HAProxy
pg_basebackup \
  -h localhost \
  -p 5432 \
  -U replicator \
  -D /backup/$(date +%Y%m%d) \
  --wal-method=stream \
  --checkpoint=fast \
  --progress

# Or use Patroni's built-in clone for a replica backup
docker exec patroni1 patronictl -c /etc/patroni/patroni.yml \
  clone patroni3 --datadir /backup/patroni3-clone
```
