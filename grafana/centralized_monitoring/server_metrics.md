# 📊 Server & Service Health Dashboard (Prometheus + Blackbox)

![Grafana](https://img.shields.io/badge/Grafana-12.1.1-orange?style=flat-square&logo=grafana)
![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-red?style=flat-square&logo=prometheus)

## 📖 Overview

The **Server & Service Health Dashboard** is a comprehensive, "Single Pane of Glass" monitoring solution designed to replace legacy tools like Nagios. It provides real-time visibility into your entire infrastructure stack, from physical server resources (CPU, RAM, Disk) to application availability (HTTP, TCP, SSL).

It is built on the **Management by Exception** philosophy:
1.  **Top Row:** Instant High-Level Status (Are we healthy?).
2.  **Bottom Row:** Critical Incident Lists (What exactly is broken?).
3.  **Middle Section:** Deep-dive metrics for individual servers.

![Dashboard Overview Screenshot](path/to/your/dashboard-overview.png)
*(Replace this placeholder with a full screenshot of your dashboard)*

---

## ✨ Features

* **Real-Time Status Totals:** Instant counts of UP/DOWN servers and services.
* **Resource Health Alerts:** Automatically highlights servers with critical CPU, RAM, or Disk usage (>90%).
* **Service Availability:** Monitors HTTP endpoints, TCP ports, and SSL certificate expiry.
* **Incident Tables:** Dynamic tables that only appear/populate when an outage occurs.
* **Drill-Down Views:** Detailed row per server with Load Average, Uptime, and specific service checks.

---

## ⚙️ Prerequisites

To use this dashboard, you need the following exporters running in your infrastructure:

1.  **[Node Exporter](https://github.com/prometheus/node_exporter):** For system metrics (CPU, RAM, Disk).
2.  **[Blackbox Exporter](https://github.com/prometheus/blackbox_exporter):** For external probing (HTTP, TCP, DNS, SSL).
3.  **Prometheus:** Scraping the above exporters.

### Required Labels
Your Prometheus configuration (`prometheus.yml`) must apply these labels for the dashboard variables to work correctly:

* `server`: A unique friendly name for the host (e.g., `prod-db-01`, `staging-api`).
* `instance`: The IP address or hostname (standard Prometheus label).

**Example Prometheus Config:**
```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['192.168.1.10:9100']
        labels:
          server: 'production-db'  # <--- Crucial for this dashboard

  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets: ['[https://api.example.com](https://api.example.com)']
        labels:
          server: 'production-api'
