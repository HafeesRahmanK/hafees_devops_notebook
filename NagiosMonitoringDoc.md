# Nagios Monitoring Documentation

**Stack:** Nagios Core · NRPE · MS Teams Notifications  
**Architecture:** Central–Agent

---

## How It Works

Nagios follows a pull-based central–agent model. The Nagios server periodically runs checks against each monitored server by communicating with an NRPE agent installed on that server. Results are collected centrally and displayed on the Nagios web dashboard. When a service or host changes state, an alert is sent to a Microsoft Teams channel via webhook.

### Agent side

Each monitored server runs the **NRPE (Nagios Remote Plugin Executor)** daemon. NRPE listens on port `5666` and exposes a set of named check commands that the Nagios server can invoke remotely. The commands cover:

- System load, logged-in users, disk usage, memory usage
- Service-specific HTTP/HTTPS endpoint checks (VC API, web, chat, Jenkins, etc.)
- SSL certificate expiry check
- Database connectivity checks (MySQL, PostgreSQL)
- RAM disk usage check

The NRPE config restricts which IPs are allowed to run checks — only the Nagios server IP is whitelisted.

### Central (Nagios server) side

The Nagios server actively polls each agent at regular intervals by calling the check commands exposed via NRPE. All configuration lives under `/usr/local/nagios/etc/`:

- `objects/contacts.cfg` — defines who gets notified and how
- `objects/commands.cfg` — defines check and notification commands
- `objects/templates.cfg` — defines reusable host and service templates
- `objects/servers/` — one config file per monitored server, defining the host and its service checks
- `nagios-teams-notify/teams_notify.py` — Python script that formats and sends alert payloads to MS Teams via an incoming webhook

When a host or service enters a PROBLEM or RECOVERY state, Nagios calls the notification command which invokes `teams_notify.py` with alert details (host, status, address, info, timestamp). The script posts a color-coded MessageCard to the configured Teams channel — red for critical/down, yellow for warning, green for OK/up, orange for unknown/unreachable.

### Data flow summary

```
Nagios Server
 └─ runs check via NRPE ──► Agent (port 5666)
                                └─ executes plugin, returns result
 ◄─ result (OK / WARNING / CRITICAL / UNKNOWN) ──────────────────┘

 If state changes:
 └─ teams_notify.py ──► MS Teams webhook ──► Teams channel alert
```

---

## Ports

| Port | Purpose |
|------|---------|
| `5666` | NRPE — Nagios server connects to this on each agent |
| `80 / 443` | HTTP/HTTPS checks run from the agent itself |
| `ICMP` | Nagios server pings each host for availability |

All of the above must be open in the agent server's firewall.

---

## Implementation Notes

**Nagios server** — Nagios and its dependencies are compiled and installed under `/usr/local/nagios`. The web interface runs on Apache. Configuration is split across multiple files under `/usr/local/nagios/etc/objects/`. Every config change must be verified before restarting Nagios.

**Agent installation** — the `nagios_nrpe_installer.sh` script handles the full agent setup: installs `nagios-nrpe-server` and plugins, whitelists the Nagios server IP, appends all check commands to `nrpe.cfg`, installs the `check_mem` plugin from GitHub, and restarts the NRPE service. The only manual step before running the script is to set the correct Nagios server IP.

**Database checks** — `check_mysql` and `check_pgsql` require a dedicated monitoring user (`nagioslocal`) to be created in the respective database before the check will succeed.

**Hostname-aware checks** — several check commands (SSL, HTTP endpoint checks) use the server's hostname. The installer script reads the hostname at runtime and substitutes it into `nrpe.cfg` automatically.

**Teams notifications** — `teams_notify.py` is called by Nagios as a notification command and accepts all alert fields as CLI arguments. It uses the legacy MessageCard format, which is compatible with Teams incoming webhooks. The webhook URL is passed as an argument, so different contacts or host groups can use different channels if needed.

**Config verification** — always run the verification command before restarting Nagios. It validates all config files and reports errors with line numbers:
```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
```

---

## Maintenance

### Add a new agent server

1. Run `nagios_nrpe_installer.sh` on the new server after setting the correct Nagios server IP in the script.
2. Create a new config file under `/usr/local/nagios/etc/objects/servers/` on the Nagios server defining the host and its service checks.
3. Verify config, then restart Nagios:
```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
sudo systemctl restart nagios
```

### Add a new service check to an existing agent

1. Add the check command entry to `/etc/nagios/nrpe.cfg` on the agent, then restart NRPE:
```bash
sudo systemctl restart nagios-nrpe-server
```
2. Add the corresponding service definition to the agent's config file on the Nagios server under `objects/servers/`.
3. Verify and restart Nagios.

### Add a new contact or notification channel

Edit `objects/contacts.cfg` on the Nagios server. Add the contact with the Teams webhook URL as the notification command argument. Verify and restart Nagios.

### Verify and restart Nagios

```bash
sudo /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
sudo systemctl restart nagios
```

### Check NRPE status on an agent

```bash
sudo systemctl status nagios-nrpe-server
```

### Test a check command manually from the Nagios server

```bash
/usr/local/nagios/libexec/check_nrpe -H <agent-ip> -c <check_command_name>
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Check returns "Connection refused" | NRPE not running on agent | `systemctl restart nagios-nrpe-server` on the agent |
| Check returns "Access denied" | Nagios server IP not in `allowed_hosts` | Add the IP to `allowed_hosts` in `/etc/nagios/nrpe.cfg` and restart NRPE |
| Host shows as unreachable | Firewall blocking ICMP or port 5666 | Open ICMP and TCP 5666 from the Nagios server on the agent firewall |
| Teams notification not sent | Wrong webhook URL or network issue | Run `teams_notify.py` manually with `--webhook_url` to test; check connectivity from Nagios server |
| Config reload fails | Syntax error in a config file | Run the `-v` verification command and fix the reported line |
| `check_mem` fails | Plugin not installed | Re-run the installer script or manually place `check_mem` in `/usr/lib/nagios/plugins/` |
| Database check fails | Monitoring user not created | Create `nagioslocal` user in MySQL/PostgreSQL with the required password and privileges |
