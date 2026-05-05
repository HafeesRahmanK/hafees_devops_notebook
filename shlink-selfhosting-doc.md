# Shlink URL Shortener — Self-Hosting Documentation

**Project:** Shlink URL Shortener  
**Environment:** Self-hosted via Docker + Nginx (Ubuntu/Debian)  
**Status:** Production  
**Last Updated:** 2026-05-05  

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Step 1 — Docker Network Setup](#3-step-1--docker-network-setup)
4. [Step 2 — MySQL Database Container](#4-step-2--mysql-database-container)
5. [Step 3 — Shlink API Container](#5-step-3--shlink-api-container)
6. [Step 4 — API Key Generation](#6-step-4--api-key-generation)
7. [Step 5 — Shlink Web Client Container](#7-step-5--shlink-web-client-container)
8. [Step 6 — Nginx Reverse Proxy](#8-step-6--nginx-reverse-proxy)
9. [Optional — Web Authentication (Basic Auth)](#9-optional--web-authentication-basic-auth)
10. [Optional — Multi-Domain with Separate `.well-known` Paths](#10-optional--multi-domain-with-separate-well-known-paths)
11. [Migration Guide](#11-migration-guide)
12. [Troubleshooting — MySQL Root Password Recovery](#12-troubleshooting--mysql-root-password-recovery)
13. [Credentials Reference](#13-credentials-reference)

---

## 1. Architecture Overview

```
Internet
   │
   ▼
Nginx (443/80)
   ├── mydomain.com        → Shlink API  (port 8090)
   ├── myapp1domain.com    → Shlink API  (port 8090) [multi-domain short URLs]
   └── myuidomain.com      → Shlink Web Client (port 8091) [basic auth protected]

Docker (shlink-network)
   ├── shlink-db       → MySQL 8.0     (port 3306)
   ├── short-api       → Shlink API    (port 8090)
   └── shortweb-client → Shlink UI     (port 8091)
```

All three containers share a Docker bridge network (`shlink-network`) so the API can communicate with the database using the container name as the hostname.

---

## 2. Prerequisites

- Ubuntu/Debian server with Docker installed
- Domain names pointed to the server's public IP
- SSL certificates provisioned (e.g., via Certbot / Let's Encrypt)
- Nginx installed on the host
- GeoLite2 license key (from MaxMind) for geolocation features

---

## 3. Step 1 — Docker Network Setup

Create a dedicated bridge network so all Shlink containers can communicate with each other by name.

```bash
docker network create shlink-network
```

---

## 4. Step 2 — MySQL Database Container

Start a MySQL 8.0 container and attach it to the shared network. Data is persisted to the host at `/opt/shlink/mysql-data`.

> **Note:** `--skip-ssl` disables SSL enforcement inside MySQL, which simplifies internal container-to-container communication.

```bash
docker run -d --name shlink-db \
    --network shlink-network \
    -e MYSQL_ROOT_PASSWORD=RootPassword! \
    -e MYSQL_DATABASE=shlink \
    -e MYSQL_USER=shlink_user \
    -e MYSQL_PASSWORD=shlink_user_pass! \
    -v /opt/shlink/mysql-data:/var/lib/mysql \
    -p 3306:3306 \
    mysql:8.0 \
    --skip-ssl
```

| Parameter | Value |
|---|---|
| Container name | `shlink-db` |
| Database name | `shlink` |
| DB user | `shlink_user` |
| Data volume | `/opt/shlink/mysql-data` |
| Port | `3306` |

---

## 5. Step 3 — Shlink API Container

Start the Shlink API and connect it to the database container over the Docker network. The API is exposed on host port `8090`.

```bash
docker run -d --name short-api \
    --network shlink-network \
    -p 8090:8080 \
    -e DEFAULT_DOMAIN=mydomain.com \
    -e IS_HTTPS_ENABLED=true \
    -e GEOLITE_LICENSE_KEY=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 \
    -e DB_DRIVER=mysql \
    -e DB_USER=shlink_user \
    -e DB_PASSWORD=shlink_user_pass! \
    -e DB_HOST=shlink-db \
    -e DB_PORT=3306 \
    shlinkio/shlink:stable
```

> **Verify connectivity** after startup:
> ```bash
> # Test full ping from API container to DB container
> docker exec -it short-api ping shlink-db
>
> # Optional: test DNS resolution only
> docker exec -it short-api getent hosts shlink-db
> ```

---

## 6. Step 4 — API Key Generation

Generate an API key inside the running `short-api` container. This key is required to connect the Web Client to the API.

```bash
docker exec -it short-api shlink api-key:generate
```

**Generated API Key:** `86cc8c01-2343-4c3e-1139-0986351d1ad5`

> Store this key securely. It is used in the Web Client container and Nginx configuration.

---

## 7. Step 5 — Shlink Web Client Container

Start the Shlink Web UI and point it at the API using the domain and API key from the previous step. The UI is exposed on host port `8091`.

```bash
docker run -d \
    --name shortweb-client \
    -p 8091:8080 \
    -e SHLINK_SERVER_URL=https://mydomain.com \
    -e SHLINK_SERVER_API_KEY=86cc8c01-2343-4c3e-1139-0986351d1ad5 \
    -e SHLINK_SERVER_NAME=ProdShlink \
    shlinkio/shlink-web-client
```

---

## 8. Step 6 — Nginx Reverse Proxy

Nginx sits in front of both containers, terminates SSL, and routes traffic. Add/replace the relevant blocks in your Nginx configuration file (typically `/etc/nginx/sites-available/default` or a site-specific file).

### 8.1 — Shlink API (`mydomain.com`)

```nginx
server {
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name mydomain.com;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    listen [::]:443 ssl;
    listen 443 ssl;
    ssl_certificate /etc/nginx/SSL/fullchain.pem;
    ssl_certificate_key /etc/nginx/SSL/privkey.pem;
}

# HTTP → HTTPS redirect
server {
    if ($host = mydomain.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    listen [::]:80;
    server_name mydomain.com;
    return 404;
}
```

### 8.2 — Additional Short-URL Domain (`myapp1domain.com`)

Use this block for each extra domain that resolves to the same Shlink API.

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name myapp1domain.com;

    ssl_certificate /etc/nginx/SSL/fullchain.pem;
    ssl_certificate_key /etc/nginx/SSL/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    location ^~ /.well-known/ {
        root /var/www/html/myapp1;
        default_type application/json;
        try_files $uri =404;
    }
}
```

### 8.3 — Shlink Web Client UI (`myuidomain.com`)

The Web UI is protected with HTTP Basic Auth.

```nginx
server {
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name myuidomain.com;

    location / {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://127.0.0.1:8091;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    listen [::]:443 ssl;
    listen 443 ssl;
    ssl_certificate /etc/nginx/SSL/fullchain.pem;
    ssl_certificate_key /etc/nginx/SSL/privkey.pem;
}

# HTTP → HTTPS redirect
server {
    if ($host = myuidomain.com) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    listen [::]:80;
    server_name myuidomain.com;
    return 404;
}
```

### 8.4 — Apply Configuration

```bash
sudo nginx -t                        # Test config for syntax errors
sudo systemctl reload nginx.service  # Apply without downtime
```

---

## 9. Optional — Web Authentication (Basic Auth)

To password-protect the Web Client UI, set up an `.htpasswd` file.

**Install the utility:**
```bash
sudo apt install apache2-utils   # Debian/Ubuntu
```

**Create the password file and add the first user:**
```bash
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

> `-c` creates the file. Only use it the **first time**. To add more users afterwards:
> ```bash
> sudo htpasswd /etc/nginx/.htpasswd anotheruser
> ```

Then add the following two lines inside the `location /` block of the Web Client Nginx server block (already included in Section 8.3):

```nginx
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;
```

---

## 10. Optional — Multi-Domain with Separate `.well-known` Paths

If each short-URL domain needs its own isolated `.well-known/` directory (e.g., for app association files), add a dedicated server block per domain with a unique root path.

```nginx
# myapp1domain.com
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name myapp1domain.com;

    ssl_certificate /etc/nginx/SSL/fullchain.pem;
    ssl_certificate_key /etc/nginx/SSL/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    location ^~ /.well-known/ {
        root /var/www/html/myapp1;
        default_type application/json;
        try_files $uri =404;
    }
}

# myapp2domain.com
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name myapp2domain.com;

    ssl_certificate /etc/nginx/SSL/fullchain.pem;
    ssl_certificate_key /etc/nginx/SSL/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    location ^~ /.well-known/ {
        root /var/www/html/myapp2;
        default_type application/json;
        try_files $uri =404;
    }
}

# myapp3domain.com
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name myapp3domain.com;

    ssl_certificate /etc/nginx/SSL/fullchain.pem;
    ssl_certificate_key /etc/nginx/SSL/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    }

    location ^~ /.well-known/ {
        root /var/www/html/myapp3;
        default_type application/json;
        try_files $uri =404;
    }
}
```

> Place static `.well-known` files at the corresponding path on the host, e.g., `/var/www/html/myapp1/.well-known/apple-app-site-association`.

---

## 11. Migration Guide

Use this procedure when moving the Shlink setup to a new server.

### Step 1 — Start containers on the new server

Bring up `shlink-db`, `short-api`, and `shortweb-client` on the destination server using the same `docker run` commands above before importing any data.

### Step 2 — Export the database from the source server

```bash
sudo mysqldump -u root -p shlink > db_dump.sql
```

### Step 3 — Transfer the dump to the new server

```bash
scp db_dump.sql ubuntu@<new-server-ip>:/home/ubuntu/
```

### Step 4 — Import the database into the new MySQL container

```bash
sudo docker exec -i shlink-db mysql -u shlink_user -pshlink_user_pass! shlink < db_dump.sql
```

### Step 5 — Copy and apply Nginx configuration

Copy the Nginx config files to the new server, then validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx.service
```

> **Backup from MySQL container directly (alternative):**
> ```bash
> docker exec shlink-db mysqldump -u shlink_user -p'shlink_user_pass!' shlink > shlink_backup.sql
> ```

---

## 12. Troubleshooting — MySQL Root Password Recovery

Use this procedure only if MySQL is installed **directly on the host** (not inside Docker) and the root password has been lost.

```bash
# 1. Stop MySQL completely
sudo systemctl stop mysql
sudo pkill -f mysqld
sudo pkill -f mysqld_safe

# 2. Prepare the run directory
sudo mkdir -p /var/run/mysqld
sudo chown mysql:mysql /var/run/mysqld

# 3. Start MySQL without authentication
sudo mysqld_safe --skip-grant-tables &

# 4. Log in and clear the root password
mysql -u root

UPDATE mysql.user
SET authentication_string = '', plugin = 'mysql_native_password'
WHERE User = 'root' AND Host = 'localhost';

FLUSH PRIVILEGES;
EXIT;

# 5. Stop the unsafe instance and restart normally
sudo pkill -f mysqld_safe
sudo pkill -f mysqld
sudo systemctl start mysql

# 6. Set the new root password
mysql -u root

ALTER USER 'root'@'localhost' IDENTIFIED BY 'RootPassword!';
FLUSH PRIVILEGES;
EXIT;

# 7. Verify login
mysql -u root -p
```

---

## 13. Credentials Reference

> ⚠️ **Security Notice:** Rotate these credentials if this document is shared outside a trusted environment.

| Service | Key | Value |
|---|---|---|
| MySQL | Root password | `RootPassword!` |
| MySQL | Database name | `shlink` |
| MySQL | App username | `shlink_user` |
| MySQL | App password | `shlink_user_pass!` |
| GeoLite2 | License key | `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` |
| Shlink API | Generated API key | `86cc8c01-2343-4c3e-1139-0986351d1ad5` |
| Shlink API | Server name | `ProdShlink` |
| SSL | Certificate path | `/etc/nginx/SSL/fullchain.pem` |
| SSL | Private key path | `/etc/nginx/SSL/privkey.pem` |

---

*Documentation generated from self-hosting setup notes.*
