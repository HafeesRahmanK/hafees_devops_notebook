# Grafana Monitoring Stack — Technical Documentation

**Project:** Infrastructure Monitoring (Central–Agent Architecture)  
**Stack:** Grafana · Prometheus · Loki · Grafana Alloy  
**Last Updated:** May 2026

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Components](#2-components)
3. [Central Server Setup](#3-central-server-setup)
   - 3.1 Docker Compose
   - 3.2 Prometheus Configuration
   - 3.3 Loki Configuration
   - 3.4 Alloy Configuration (Central)
   - 3.5 Agent Discovery File
4. [Agent Setup](#4-agent-setup)
   - 4.1 Docker Compose
   - 4.2 Alloy Configuration (Agent)
   - 4.3 Blackbox Targets File
5. [Data Flow](#5-data-flow)
6. [What Is Monitored](#6-what-is-monitored)
7. [Port Reference](#7-port-reference)
8. [File Structure Reference](#8-file-structure-reference)
9. [Key Configuration Decisions](#9-key-configuration-decisions)
10. [Maintenance & Operations](#10-maintenance--operations)

---

## 1. Architecture Overview

This setup follows a **Central–Agent** pattern where lightweight Alloy agents run on each monitored server and push data to a central server that hosts all storage and visualization services.

```
┌─────────────────────────────────────────────────────────────┐
│                     CENTRAL SERVER (10.0.0.220)             │
│                                                             │
│  ┌──────────┐   ┌────────────┐   ┌──────┐   ┌──────────┐  │
│  │ Grafana  │   │ Prometheus │   │ Loki │   │  Alloy   │  │
│  │ :3000    │   │  :9090     │   │:3100 │   │  :12345  │  │
│  └────┬─────┘   └─────┬──────┘   └──┬───┘   └────┬─────┘  │
│       │               │             │             │        │
│       └───────────────┴─────────────┘             │        │
│                    (reads)              (ICMP/TCP  │        │
│                                          probes)  │        │
└───────────────────────────────────────────────────┼────────┘
                                                    │ probes agents
                    ┌───────────────────────────────┘
                    │
        ┌───────────▼────────────────────────────────┐
        │           AGENT SERVERS                    │
        │                                            │
        │  ┌─────────────────────────────────────┐  │
        │  │   Alloy Agent  :12345               │  │
        │  │   ┌─────────────────────────────┐   │  │
        │  │   │ • node_exporter metrics     │   │  │
        │  │   │ • blackbox service checks   │   │  │
        │  │   │ • systemd journal logs      │   │  │
        │  │   │ • docker container logs     │   │  │
        │  │   └─────────────────────────────┘   │  │
        │  │          │               │           │  │
        │  │          ▼               ▼           │  │
        │  │   Prometheus         Loki            │  │
        │  │   remote_write     push              │  │
        │  └──────────┬───────────────┬───────────┘  │
        │             │               │              │
        │     (10.0.0.220:9090) (10.0.0.220:3100)   │
        └────────────────────────────────────────────┘
          Agent 1: 10.0.0.193    Agent 2: 10.0.0.182
```

**Summary:**
- Agents **push** metrics to Prometheus and logs to Loki on the central server.
- The central Alloy **probes** each agent via ICMP ping and TCP connection check.
- Grafana queries Prometheus and Loki to display dashboards.

---

## 2. Components

| Component | Role | Version | Hosted On |
|-----------|------|---------|-----------|
| **Grafana** | Visualization & dashboards | latest | Central Server |
| **Prometheus** | Metrics storage & querying | latest | Central Server |
| **Loki** | Log aggregation & storage | latest | Central Server |
| **Alloy (Central)** | ICMP/TCP probing of agents | v1.2.0 | Central Server |
| **Alloy (Agent)** | Metrics collection, log shipping | v1.2.0 | Each Agent Server |

---

## 3. Central Server Setup

All services on the central server are orchestrated via a single `docker-compose.yml`.

### 3.1 Docker Compose

**File:** `docker-compose.yml`

```yaml
networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  loki-wal:

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SERVER_ROOT_URL=https://grafana.nearconferences.com
      - GF_CORS_ALLOW_ORIGIN=*
      - GF_CORS_ENABLED=true
      - GF_CORS_ALLOW_CREDENTIALS=true
    networks:
      - monitoring
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/config.yaml
    volumes:
      - ./loki/loki-config.yml:/etc/loki/config.yaml
      - loki-data:/loki
      - ./loki-wal:/wal
    networks:
      - monitoring
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-remote-write-receiver'
    networks:
      - monitoring
    restart: unless-stopped

  alloy:
    image: grafana/alloy:v1.2.0
    container_name: alloy
    user: root
    privileged: true
    pid: host
    restart: unless-stopped
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy
      - ./agent_targets.yml:/etc/alloy/agent_targets.yml
    command: run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy
    ports:
      - "12345:12345"
    depends_on:
      - loki
      - prometheus
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - monitoring
```

**Key Notes:**
- `--web.enable-remote-write-receiver` on Prometheus is **critical** — it enables agents to push metrics via remote write.
- `--storage.tsdb.retention.time=30d` retains 30 days of metrics.
- Grafana admin password should be set via `GF_SECURITY_ADMIN_PASSWORD` (currently commented out — set this before production use).
- All services share the `monitoring` bridge network so they can resolve each other by container name (e.g., `http://prometheus:9090`).

---

### 3.2 Prometheus Configuration

Prometheus on the central server acts primarily as a **remote write receiver**. Agents push their metrics to `http://<central-ip>:9090/api/v1/write`. The central Alloy also writes its probe results here.

Relevant startup flags (set in docker-compose `command`):

| Flag | Value | Purpose |
|------|-------|---------|
| `--web.enable-remote-write-receiver` | *(flag only)* | Accept pushed metrics from agents |
| `--storage.tsdb.retention.time` | `30d` | Keep 30 days of metric history |

---

### 3.3 Loki Configuration

**File:** `loki/loki-config.yml`

Loki stores logs shipped by agents. Key settings:

| Setting | Value | Notes |
|---------|-------|-------|
| `http_listen_port` | `3100` | Agents push to this port |
| `store` | `boltdb-shipper` | Index storage backend |
| `object_store` | `filesystem` | Chunk storage on disk |
| `retention_period` | `720h` (30 days) | Log retention window |
| `replication_factor` | `1` | Single-node, no HA |
| `allow_structured_metadata` | `false` | Disabled for compatibility |

**Storage Layout on Disk:**

```
/loki/
├── boltdb-shipper-active/    ← Active index
├── boltdb-shipper-cache/     ← Index cache (TTL: 24h)
├── chunks/                   ← Log chunk data
└── compactor/                ← Compaction working directory
```

> **Note:** The `table_manager` with `retention_deletes_enabled: true` enforces the 30-day log purge automatically.

---

### 3.4 Alloy Configuration (Central)

**File:** `config.alloy` (Central)

The central Alloy's only job is to **probe agent servers** (ping + TCP check) and write results to Prometheus.

#### Step 1 — Define the Blackbox Prober

```hcl
prometheus.exporter.blackbox "central_prober" {
  config = "{ modules: {
    icmp_check: { prober: icmp },
    alloy_check: { prober: tcp, timeout: 5s }
  } }"
  target {
    name    = "dummy"
    address = "127.0.0.1"
    module  = "icmp_check"
  }
}
```

Two probe modules are defined:
- `icmp_check` — ICMP ping to verify server is reachable
- `alloy_check` — TCP connect to port `12345` to verify Alloy is running on the agent

#### Step 2 — Discover Agent Targets

```hcl
discovery.file "agents" {
  files = ["/etc/alloy/agent_targets.yml"]
}
```

Agent IPs and metadata are loaded from `agent_targets.yml`. Add new agents to this file without restarting Alloy.

#### Step 3 — Relabel Targets for Probing

```hcl
discovery.relabel "probe_agents" {
  targets = discovery.file.agents.targets
  # ... sets __param_target, __param_module, instance, __address__, __metrics_path__
}
```

This relabeling is what makes Alloy's HTTP API act as a blackbox proxy — it routes probe requests to the correct module endpoint.

#### Step 4 — Scrape and Write Results

```hcl
prometheus.scrape "central_probes" {
  targets         = discovery.relabel.probe_agents.output
  forward_to      = [prometheus.remote_write.local.receiver]
  job_name        = "central-blackbox"
  scrape_interval = "30s"
}

prometheus.remote_write "local" {
  endpoint { url = "http://prometheus:9090/api/v1/write" }
}
```

Probe results are written to Prometheus every 30 seconds.

---

### 3.5 Agent Discovery File

**File:** `agent_targets.yml` (Central)

This file lists all agent servers to probe. Each agent gets two entries: one for ICMP ping, one for Alloy TCP check.

```yaml
- targets:
    - 10.0.0.193
  labels:
    server: "server1"
    module: "icmp_check"

- targets:
    - 10.0.0.193:12345
  labels:
    server: "server1"
    service: "alloy"
    module: "alloy_check"

- targets:
    - 10.0.0.182
  labels:
    server: "server2"
    module: "icmp_check"

- targets:
    - 10.0.0.182:12345
  labels:
    server: "server2"
    service: "alloy"
    module: "alloy_check"
```

**To add a new server:** append two blocks following the same pattern and update the IP.

---

## 4. Agent Setup

Each monitored server runs a single Alloy container that collects all telemetry and ships it to the central server.

### 4.1 Docker Compose

**File:** `docker-compose.yml` (Agent)

```yaml
services:
  alloy:
    image: grafana/alloy:v1.2.0
    container_name: alloy
    user: root
    privileged: true
    pid: host
    restart: unless-stopped
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy
      - ./targets.yml:/etc/alloy/targets.yml
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/log/journal:/var/log/journal:ro
      - /etc/machine-id:/etc/machine-id:ro
    command: run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy
    ports:
      - "12345:12345"
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

**Volume Mounts Explained:**

| Mount | Purpose |
|-------|---------|
| `./config.alloy` | Alloy pipeline configuration |
| `./targets.yml` | Blackbox service check targets |
| `/var/run/docker.sock` | Read docker container metadata for log collection |
| `/var/log/journal` | Read systemd journal logs |
| `/etc/machine-id` | Used by Alloy for stable host identification |

**Why `user: root` and `privileged: true`?**  
Required for ICMP probing (raw socket access) and for reading the systemd journal from the host.

---

### 4.2 Alloy Configuration (Agent)

**File:** `config.alloy` (Agent)

The agent pipeline has five functional sections.

#### Section 1 — System Metrics (node_exporter)

Collects host-level metrics: CPU, memory, disk, network, etc.

```hcl
prometheus.exporter.unix "system" {
  include_exporter_metrics = true
}

discovery.relabel "system_relabel" {
  targets = prometheus.exporter.unix.system.targets
  rule {
    target_label = "job"
    replacement  = "node-exporter"
  }
}

prometheus.scrape "system_scrape" {
  targets    = discovery.relabel.system_relabel.output
  forward_to = [prometheus.remote_write.local.receiver]
  job_name   = "node-exporter"
}
```

#### Section 2 — Service Checks (Blackbox Exporter)

Performs HTTP and TCP checks against services defined in `targets.yml`.

```hcl
prometheus.exporter.blackbox "global_blackbox" {
  config = "{ modules: {
    http_2xx: { prober: http, timeout: 5s },
    tcp_connect: { prober: tcp, timeout: 5s }
  } }"
}

discovery.file "blackbox_targets" {
  files = ["/etc/alloy/targets.yml"]
}
```

Results are scraped and pushed to Prometheus under job `Blackbox-exporter`.

#### Section 3 — Systemd Journal Logs

Collects logs from specific systemd services and ships to Loki.

```hcl
loki.source.journal "journal" {
  path    = "/var/log/journal"
  max_age = "24h"
  matches = "_SYSTEMD_UNIT=ai.service _SYSTEMD_UNIT=calendar.service ..."
  labels  = { job = "systemd-journal" }
  forward_to = [loki.write.local.receiver]
}
```

**Monitored systemd units:**

| Service | Service | Service |
|---------|---------|---------|
| ai.service | calendar.service | chat.service |
| cmap.service | commandcenter.service | connect.service |
| drive.service | dt.service | governance.service |
| license.service | mail.service | neargpt.service |
| sfu1.service | sfu2.service | sign.service |
| vc.service | vcapi1.service | vcapi2.service |
| gpt.service | gptui.service | |

#### Section 4 — Docker Container Logs

Discovers running Docker containers and ships their logs to Loki.

```hcl
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "docker_logs" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.container_labels.output
  forward_to = [loki.write.local.receiver]
}
```

Each log line is tagged with `unit` (container name) and `job=docker-containers`.

#### Section 5 — Remote Writers

All collected data is sent to the central server.

```hcl
prometheus.remote_write "local" {
  endpoint {
    url = "http://10.0.0.220:9090/api/v1/write"
  }
  external_labels = {
    server = "server1"
  }
}

loki.write "local" {
  endpoint {
    url = "http://10.0.0.220:3100/loki/api/v1/push"
  }
  external_labels = {
    instance = "server1"
  }
}
```

> **Important:** Update `server` and `instance` labels in `external_labels` for each agent so metrics and logs are distinguishable per server in Grafana.

---

### 4.3 Blackbox Targets File

**File:** `targets.yml` (Agent)

Defines which local services the agent's blackbox exporter should check.

```yaml
- targets:
    - host.docker.internal:5002
  labels:
    service: service1
    module: tcp_connect

- targets:
    - host.docker.internal:80
  labels:
    service: service2
    module: tcp_connect

- targets:
    - host.docker.internal:443
  labels:
    service: service3
    module: tcp_connect

- targets:
    - host.docker.internal:3020
  labels:
    service: service4
    module: tcp_connect
```

`host.docker.internal` resolves to the host machine's IP from inside the container. Update `service` labels with meaningful names to identify them in Grafana dashboards.

---

## 5. Data Flow

### Metrics Flow

```
Agent Host (OS/Docker)
        │
        ▼
Alloy agent — prometheus.exporter.unix (node metrics)
Alloy agent — prometheus.exporter.blackbox (service TCP checks)
        │
        ▼ prometheus.remote_write
Central Server: Prometheus :9090 (stores metrics)
        │
        ▼
Grafana :3000 (queries and visualizes)
```

### Logs Flow

```
Agent Host
├── /var/log/journal (systemd units)
└── /var/run/docker.sock (container stdout/stderr)
        │
        ▼
Alloy agent — loki.source.journal + loki.source.docker
        │
        ▼ loki.write
Central Server: Loki :3100 (stores logs)
        │
        ▼
Grafana :3000 (queries and visualizes)
```

### Probe Flow (Central → Agent)

```
Central Alloy reads agent_targets.yml
        │
        ▼
ICMP ping → agent IP (is server reachable?)
TCP connect → agent IP:12345 (is Alloy running?)
        │
        ▼ prometheus.remote_write
Central Prometheus :9090 (stores probe results)
        │
        ▼
Grafana :3000 (shows up/down status)
```

---

## 6. What Is Monitored

| Category | What | How | Labels |
|----------|------|-----|--------|
| **Server Availability** | ICMP ping to each agent | Central Alloy → Blackbox | `server`, `module=icmp_check` |
| **Alloy Agent Health** | TCP check to port 12345 | Central Alloy → Blackbox | `server`, `service=alloy`, `module=alloy_check` |
| **System Metrics** | CPU, memory, disk, network, load | node_exporter (unix) | `server`, `job=node-exporter` |
| **Service Port Checks** | Ports 80, 443, 5002, 3020 | Agent Blackbox → TCP | `service`, `module=tcp_connect` |
| **Systemd Logs** | 19 application services | Journal scraper | `unit`, `job=systemd-journal`, `instance` |
| **Docker Logs** | All running containers | Docker socket | `unit`, `job=docker-containers`, `instance` |

---

## 7. Port Reference

| Port | Service | Direction | Notes |
|------|---------|-----------|-------|
| `3000` | Grafana | Inbound (users) | Web UI, accessible via HTTPS at grafana.nearconferences.com |
| `3100` | Loki | Inbound (agents) | Log push endpoint |
| `9090` | Prometheus | Inbound (agents) | Metric remote write endpoint |
| `12345` | Alloy (Central) | Inbound (probes) | Alloy UI + probe API |
| `12345` | Alloy (Agent) | Inbound (central probe) | Must be reachable from central for TCP health check |

---

## 8. File Structure Reference

### Central Server

```
central/
├── docker-compose.yml          ← All central services
├── config.alloy                ← Central Alloy (probing only)
├── agent_targets.yml           ← List of agent IPs to probe
├── prometheus/
│   └── prometheus.yml          ← Prometheus config (if needed)
└── loki/
    └── loki-config.yml         ← Loki storage & retention config
```

### Agent Server

```
agent/
├── docker-compose.yml          ← Alloy agent service
├── config.alloy                ← Full agent pipeline config
└── targets.yml                 ← Local services to TCP-check
```

---

## 9. Key Configuration Decisions

### Why Alloy instead of separate exporters?
Grafana Alloy is an all-in-one collector that replaces the need for separate `node_exporter`, `promtail`, and `blackbox_exporter` binaries. A single container on each agent handles metrics, logs, and service checks.

### Why remote_write instead of Prometheus scraping agents?
With remote write (agent → push model), agents do not need to be reachable by Prometheus on any port except `12345` (for the central TCP health check). This is simpler in firewalled environments.

### Why two Alloy instances?
- **Central Alloy** only handles infrastructure-level probing (is the server up? is Alloy running?).
- **Agent Alloy** handles all application-level data collection.

This separation keeps the central server's config simple and independent of each agent's application setup.

### Retention Period
Both Prometheus (`30d`) and Loki (`720h = 30 days`) are configured to the same retention window for consistency.

---

## 10. Maintenance & Operations

### Adding a New Agent Server

1. Deploy the agent `docker-compose.yml` and `config.alloy` on the new server.
2. Update `external_labels` in `config.alloy` with a unique `server` name.
3. Update `targets.yml` with services running on that server.
4. On the **central server**, add two entries to `agent_targets.yml`:
   ```yaml
   - targets:
       - <NEW_SERVER_IP>
     labels:
       server: "server3"
       module: "icmp_check"
   - targets:
       - <NEW_SERVER_IP>:12345
     labels:
       server: "server3"
       service: "alloy"
       module: "alloy_check"
   ```
5. Alloy on the central server will auto-reload the file — no restart required.

### Adding a New Service to Monitor (Agent)

1. Edit `targets.yml` on the relevant agent server.
2. Add a new entry with the service's port and a descriptive label.
3. Alloy discovers the file dynamically — no restart needed.

### Adding a New Systemd Service to Log

1. Edit `config.alloy` on the relevant agent.
2. Append `_SYSTEMD_UNIT=<service-name>.service` to the `matches` string in `loki.source.journal`.
3. Restart the Alloy container: `docker compose restart alloy`

### Restarting Services

```bash
# Central server
docker compose restart grafana
docker compose restart prometheus
docker compose restart loki
docker compose restart alloy

# Or all at once
docker compose restart
```

### Checking Alloy Status

Open the Alloy UI in a browser:
- Central: `http://<central-ip>:12345`
- Agent: `http://<agent-ip>:12345`

### Common Issues

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| No metrics from agent | Agent `remote_write` URL wrong or central port 9090 blocked | Verify URL and firewall rules |
| No logs in Loki | Agent `loki.write` URL wrong or port 3100 blocked | Verify URL and firewall |
| ICMP probe failing | Firewall blocking ICMP from central | Allow ICMP from central IP |
| `alloy_check` failing | Agent Alloy not running or port 12345 blocked | Check `docker ps` on agent, verify port |
| Loki "structured metadata" error | `allow_structured_metadata` not set to false | Already set — verify loki-config.yml is mounted |
| Old logs not purging | `retention_deletes_enabled` not active | Verify loki-config.yml `table_manager` section |

---

*Documentation generated from production configuration files.*
