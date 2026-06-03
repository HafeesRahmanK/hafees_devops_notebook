# Docker CI/CD Architecture — Deployment Documentation

> **Version:** 1.0 | **Date:** June 2026 | **Maintained by:** DevOps Team | **Classification:** Confidential

---

## Table of Contents

1. [Overview](#1-overview)
2. [Server Roles](#2-server-roles)
3. [CI/CD Pipeline (Jenkins)](#3-cicd-pipeline-jenkins)
4. [Service Topology](#4-service-topology)
5. [Environment Configuration](#5-environment-configuration)
6. [SSL / TLS](#6-ssl--tls)
7. [Health Checks](#7-health-checks)
8. [MongoDB](#8-mongodb)
9. [Redis](#9-redis)
10. [Adding a New Service](#10-adding-a-new-service)
11. [Common Operational Tasks](#11-common-operational-tasks)
12. [Zoho Sprint — Document Management](#12-zoho-sprint--document-management)
13. [Glossary](#13-glossary)

---

## 1. Overview

This document describes the end-to-end Docker-based build and deployment pipeline. It covers server roles, CI/CD tooling (Jenkins), repository conventions, the docker-compose service topology, environment configuration, and day-to-day operational procedures.

### 1.1 Architecture Summary

The system is split across two dedicated servers:

| Server | Role |
|---|---|
| **Docker Build Server** | Builds Docker images from source code |
| **Deploy Server** | Runs all services via docker-compose and serves production traffic |

Jenkins orchestrates both servers using **Execute Remote Operations via SSH**. No Docker registry is required — images are built and transferred directly between servers.

```
┌─────────────────────┐        SSH (Jenkins)        ┌─────────────────────────┐
│   Jenkins Master    │ ──────────────────────────► │   Docker Build Server   │
│                     │                             │  - git pull             │
│                     │                             │  - docker build         │
│                     │                             │  - docker image push    │
│                     │        SSH (Jenkins)        └─────────────────────────┘
│                     │ ──────────────────────────► ┌─────────────────────────┐
│                     │                             │     Deploy Server       │
└─────────────────────┘                             │  - docker image pull    │
                                                    │  - docker-compose up    │
                                                    │  - nginx-proxy (SSL)    │
                                                    └─────────────────────────┘
```

---

## 2. Server Roles

### 2.1 Docker Build Server

**Responsibilities:**
- Pulls latest source code from project repositories
- Reads `Dockerfile` and supporting build files per service
- Executes `docker build` and tags the resulting image
- Push image to Container registry (OCIR)

**Build-related files in each service repository:**

| File | Purpose |
|---|---|
| `Dockerfile` | Defines the Docker image build instructions |
| `.dockerignore` | Excludes files/directories from the build context to keep images lean |
| `entrypoint.sh` | Container startup script; handles env substitution and process launch |
| `DockerReadme.txt` | Human-readable notes about building/running the specific service image |

### 2.2 Deploy Server

**Responsibilities:**
- Pulls Docker images from central registry by the Build Server
- Stores `docker-compose.yml`, `.env_demo` (populated as `.env`), and SSL certificates
- Starts, stops, and updates services via `docker-compose`
- Exposes services to the internet through the `nginx-proxy` container

**Key files on the Deploy Server:**

| File / Directory | Purpose |
|---|---|
| `docker-compose.yml` | Defines all services, ports, env vars, healthchecks, and volumes |
| `.env_demo` | Template of required environment variables. Copy to `.env` and fill values before deployment |
| `.env` | Active environment file consumed by docker-compose. **Never commit to version control** |
| `./SSL/` | TLS certificates and keys. Mounted into `nginx-proxy` at `/etc/nginx/SSL` |

---

## 3. CI/CD Pipeline (Jenkins)

Jenkins is the sole CI/CD orchestrator. Both Build and Deploy stages are triggered through SSH-based remote execution jobs.

### 3.1 Build Job

Jenkins connects to the Build Server over SSH and performs:

1. Pull the latest code from the service's Git repository
2. Run `docker build -t <IMAGE_TAG> .` inside the project directory
3. Save and transfer the image to the Deploy Server

> **Note:** Each service has its own Jenkins build job. Image tag naming follows the convention defined in `.env_demo` (e.g. `NGINX_IMAGE`, `MAIL_IMAGE`).

### 3.2 Deploy Job

Jenkins connects to the Deploy Server over SSH and performs:

1. Load the new Docker image:
   ```bash
   docker load < image.tar
   ```
2. Stop and remove the old container:
   ```bash
   docker-compose stop <service> && docker-compose rm -f <service>
   ```
3. Bring the service back up:
   ```bash
   docker-compose up -d <service>
   ```
4. Verify health:
   ```bash
   docker ps
   # or
   docker inspect <container>
   ```

### 3.3 Full Stack Restart

To restart all services at once (e.g. after a server reboot or `.env` change):

```bash
cd /path/to/deploy && docker-compose down && docker-compose up -d
```

---

## 4. Service Topology

The `docker-compose.yml` defines **18 services**. All application services use `restart: unless-stopped`. `nginx-proxy` and `redis-server` use `restart: always`.

| Service | Port(s) | Description | Network Mode |
|---|---|---|---|
| `nginx-proxy` | 80, 443 | Reverse proxy & SSL termination | host |
| `mail` | 3900 | Mail service | bridge |
| `license` | 3030 | License management service | bridge |
| `digitaltwin` | 3090 | Digital Twin application | bridge |
| `governance` | 3080 | Governance application | bridge |
| `connect` | 3060 | Connect application | bridge |
| `sign` | 3500 | Document signing service | bridge |
| `chat` | 3070 | Chat / video conferencing | bridge |
| `drive` | 3300 | Drive / file storage service | bridge |
| `calendar` | 3400 | Calendar service | bridge |
| `vc` | 3000 | Video conferencing (main) | bridge |
| `vcapi1` | 4000 | VC API instance 1 (SFU path: `sfu1`) | bridge |
| `vcapi2` | 4001 → 4000 | VC API instance 2 (SFU path: `sfu2`) | bridge |
| `redis-server` | — | Redis in-memory store for SFU | host |
| `sfu1` | 8443 | SFU media server 1 | host |
| `sfu2` | 4443 | SFU media server 2 | host |
| `board` | 8080 → 80 | Whiteboard service (nginx) | bridge |
| `pad` | 9001 | Collaborative pad service | bridge |

> **Note:** Services with `network_mode: host` share the host network stack directly. All other services use Docker bridge networking and publish specific ports.

### 4.1 Service Dependencies

- `sfu1` and `sfu2` depend on `redis-server` (`condition: service_healthy`) before starting
- `nginx-proxy` depends on nothing — it starts immediately and proxies to upstream containers
- All other inter-service communication is via environment variables (domain names) resolved through `nginx-proxy`

### 4.2 SFU (Selective Forwarding Unit) Instances

Two SFU instances handle WebRTC media routing. Each runs on a different port and path, allowing load distribution:

| Instance | Port | SFUPATH | Paired API |
|---|---|---|---|
| `sfu1` | 8443 | `sfu1` | `vcapi1` (host port 4000) |
| `sfu2` | 4443 | `sfu2` | `vcapi2` (host port 4001) |

Both `vcapi` containers share the same Docker image but differ only in the `SFUPATH` environment variable and host port mapping.

---

## 5. Environment Configuration

All runtime configuration is injected via environment variables defined in `.env`. The `.env_demo` file on the Deploy Server acts as a documented template — copy it to `.env` and substitute real values before first deployment.

> ⚠️ **Never commit `.env` to version control. Rotate `BASEKEY`, `BASESERVICEKEY`, and database credentials regularly.**

| Variable | Purpose |
|---|---|
| `VC` | Domain for the main VC app |
| `EMAIL` | Domain for the mail service |
| `CHAT` | Domain for the chat service |
| `GOVERNANCE` | Domain for governance service |
| `CONNECT` | Domain for connect service |
| `DT` | Domain for digital twin service |
| `SIGN` | Domain for sign service |
| `DRIVE` | Domain for drive service |
| `CALENDAR` | Domain for calendar service |
| `LICENSE` | Domain for license service |
| `BASE` / `BASEKEY` | Base service URL and auth key |
| `BASESERVICEKEY` | Service-to-service auth key (vcapi) |
| `DBUSER` / `DBPASSWORD` | MongoDB credentials |
| `DBHOST` / `DBPORT` | MongoDB host and port |
| `DT_DB` / `GOVERNANCE_DB` / `..._DB` | Per-service MongoDB database names |
| `AI` / `GPT` | AI/GPT service config (governance, connect, vc) |
| `STUN` / `TURN` | WebRTC STUN/TURN server URLs (chat, vc) |
| `VCLANIP` / `VCPUBLICIP` | LAN and public IP for SFU media routing |
| `SSOCLIENTID` / `SSOSECRET` / `SSOTENANTID` | SSO OAuth credentials (dt, governance, connect, chat) |
| `NGINX_IMAGE` / `MAIL_IMAGE` / `..._IMAGE` | Docker image tags for each service |

---

## 6. SSL / TLS

SSL certificates are stored in the `./SSL` directory on the Deploy Server and mounted read-only into the `nginx-proxy` container at `/etc/nginx/SSL`.

- Certificates must be renewed before expiry. `nginx-proxy` will fail its health check (`nginx -t`) if certificate files are missing or malformed
- After renewing certificates, reload nginx without downtime:
  ```bash
  docker exec nginx-proxy nginx -s reload
  ```
- Ensure certificates cover **all service domains** listed in the `.env` file

---

## 7. Health Checks

Every service defines a Docker healthcheck. `docker-compose ps` or `docker ps` shows the health status (`healthy` / `unhealthy` / `starting`).

| Services | Health Check Method | Notes |
|---|---|---|
| `nginx-proxy`, `board` | `nginx -t` | Config syntax validation |
| `mail`, `pad`, `vcapi1`, `vcapi2`, `sfu1`, `sfu2` | TCP `net.connect(port)` | Socket connect on service port |
| `license`, `digitaltwin`, `governance`, `connect`, `sign`, `chat`, `drive`, `calendar`, `vc` | HTTP GET `127.0.0.1:PORT` → 200 | HTTP 200 response check |
| `redis-server` | `redis-cli ping` | Redis PING command |

**Default timings for all services:**

| Parameter | Value |
|---|---|
| `interval` | 15s (nginx-proxy / redis: 10s / 5s) |
| `timeout` | 5s (redis: 3s) |
| `retries` | 5 |
| `start_period` | 30s (nginx-proxy / redis: not set) |

---

## 8. MongoDB

MongoDB is **not containerised** in this docker-compose stack. It runs externally and is accessed via `DBHOST:DBPORT` with credentials `DBUSER` / `DBPASSWORD`. Each service uses its own dedicated database (separate `_DB` variable).

- Ensure MongoDB is reachable from the Deploy Server before starting services
- The `authSource=admin` parameter is set in all `MONGO_URL` connection strings
- Take regular backups of all per-service databases **before deployments**

---

## 9. Redis

`redis-server` runs as a containerised Redis instance with `network_mode: host`, making it reachable at `127.0.0.1:6379` by `sfu1` and `sfu2` (which also run in host network mode).

> **Security note:** No authentication is configured by default — restrict Redis access at the firewall/iptables level to prevent external exposure.

---

## 10. Adding a New Service

1. Add `Dockerfile`, `.dockerignore`, `entrypoint.sh`, and `DockerReadme.txt` to the service's Git repo
2. Create a Jenkins **build job** targeting the new repo
3. Add a new service block to `docker-compose.yml` on the Deploy Server with appropriate port, environment variables, and healthcheck
4. Add the new `IMAGE` variable (e.g. `NEWSERVICE_IMAGE`) to `.env_demo` and `.env`
5. Add an nginx upstream block and server block for the new domain in the `nginx-proxy` config
6. Add the domain to the SSL certificate and place the cert in `./SSL`
7. Create a Jenkins **deploy job** for the new service
8. Update this document and link it in Zoho Sprint

---

## 11. Common Operational Tasks

### Check service status
```bash
docker-compose ps
```

### View logs for a service
```bash
docker-compose logs -f <service>
```

### Restart a single service
```bash
docker-compose restart <service>
```

### Force-recreate a service after image update
```bash
docker-compose up -d --force-recreate <service>
```

### Reload nginx config without downtime
```bash
docker exec nginx-proxy nginx -s reload
```

### Inspect a container's health
```bash
docker inspect --format='{{json .State.Health}}' <container>
```

### Stop all services
```bash
docker-compose down
```

### Start all services
```bash
docker-compose up -d
```

### View resource usage
```bash
docker stats
```

### Scale vcapi instances (if needed)
> `vcapi1` and `vcapi2` are statically defined with different `SFUPATH` values. To add a third instance, define `vcapi3` in `docker-compose.yml` following the same pattern with a new host port and `SFUPATH`.

---

## 12. Zoho Sprint — Document Management

This document should be stored and maintained in Zoho Sprint under the DevOps epic or a dedicated Infrastructure item.

**Recommended structure:**
- **Epic:** Infrastructure & DevOps
- **Item:** Docker CI/CD Architecture Documentation
- **Attachments:** this document (versioned by date, e.g. `Docker_CICD_Docs_2026-06.md`)
- **Description field:** link to this document + one-line change summary per update

**Update this document when:**
- A new service is added or removed from `docker-compose.yml`
- Jenkins job structure changes
- SSL certificate renewal process changes
- MongoDB or Redis topology changes
- New environment variables are introduced
- Build or deploy server infrastructure changes

---

## 13. Glossary

| Term | Definition |
|---|---|
| **SFU** | Selective Forwarding Unit — routes WebRTC media streams between participants without mixing |
| **nginx-proxy** | Reverse proxy container that terminates SSL and routes HTTP/WS requests to backend services |
| **`.env`** | Docker Compose environment file containing all runtime secrets and domain names |
| **`entrypoint.sh`** | Shell script run as the container's entrypoint to bootstrap the application process |
| **`SFUPATH`** | URL path prefix that distinguishes `sfu1` from `sfu2` in nginx routing |
| **`BASEKEY`** | Shared authentication key used for inter-service API calls |
| **`BASESERVICEKEY`** | Service-level key used by `vcapi` to authenticate with base services |
| **host network mode** | Container shares the host's network namespace; used for low-latency UDP (WebRTC) |
| **bridge network mode** | Default Docker networking; container gets its own network namespace with port publishing |
| **healthcheck** | Docker-native liveness probe run at intervals to determine container health status |

---

*Last updated: June 2026 — DevOps Team*
