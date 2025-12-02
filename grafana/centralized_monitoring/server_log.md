# 📜 Server & Service Logs Dashboard (Loki)

![Grafana](https://img.shields.io/badge/Grafana-12.1.1-orange?style=flat-square&logo=grafana)
![Loki](https://img.shields.io/badge/Loki-LogQL-blue?style=flat-square&logo=grafana)

## 📖 Overview

The **Server & Service Logs Dashboard** is a centralized log explorer designed to replace the need for SSH-ing into servers to run `tail -f`. It connects to a **Loki** datasource to provide real-time, searchable access to systemd service logs across your entire infrastructure.

This dashboard is optimized for **systemd logs** collected via Promtail, allowing you to filter dynamically by **Server**, **Service Name**, and **Key-words**.

## ✨ Features

* **Multi-Server Support:** Dropdown selection to switch context between different servers (instances).
* **Service Discovery:** Automatically discovers running services (systemd units) on the selected server.
* **Smart Filtering:**
    * Hides noise (automatically excludes `promtail` and `node-exporter` from the service list).
    * Strips the `.service` suffix for a cleaner UI.
* **Full-Text Search:** Includes a text input box to filter logs by keyword (Case-Insensitive Regex supported).
* **Live Tailing:** Supports live streaming of logs with "Live" mode.

---

## ⚙️ Prerequisites

To use this dashboard, you need:
1.  **Grafana** (v10+ recommended).
2.  **Loki** configured as a datasource.
3.  **Promtail** (or Grafana Agent) running on your target servers, configured to scrape journald (systemd) logs.

### Required Labels
Your logs must have the following labels for the dashboard variables to work:
* `instance`: The hostname or IP of the server.
* `unit`: The systemd service name (e.g., `nginx.service`, `myapp.service`).

---

## 📥 Installation

1.  **Download:** Get the `Server_logs_dashboard.json` file from this repository.
2.  **Import:**
    * Open Grafana.
    * Go to **Dashboards** -> **New** -> **Import**.
    * Upload the JSON file.
3.  **Select Datasource:** During import, select your generic **Loki** datasource when prompted.

---

## 🎛️ Dashboard Controls

### 1. Variables (Top Bar)

| Variable | Description | Logic |
| :--- | :--- | :--- |
| **Server** | Select the target server to view. | Fetches `instance` label. Excludes `ai-api` and `meteor-stg` by default. |
| **Services** | Select one or more services to tail. | Fetches `unit` label based on the selected Server. Excludes `promtail` and `node-exporter`. |
| **Search key-word** | Filter log lines by text. | Accepts Regex. Example: `error`, `exception`, `user_id=123`. |

### 2. The Logs Panel

The main view is a **Logs Visualization** panel.
* **Time Display:** Shows the timestamp of the log ingestion.
* **Labels:** Displays common labels (instance, unit) next to the log line.
* **Log Details:** Click on any log line to expand it and see full metadata/detected fields.

---

## 🧠 Query Logic

For power users who want to understand or modify the LogQL queries used:

**Panel Query:**
```logql
{instance="$server", unit=~"($service)(\\.service)?"} |~ "(?i)$search"
