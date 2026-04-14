# Network Monitoring Stack - Setup Guide

Step-by-step instructions for deploying the full Prometheus + Grafana + SNMP monitoring stack (Project 4). This guide assumes you are starting from a clean clone and walks through every configuration step, verification check, and troubleshooting scenario.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Clone Repo and Configure Environment](#2-clone-repo-and-configure-environment)
3. [Configure SNMPv3 on Cisco Devices](#3-configure-snmpv3-on-cisco-devices)
4. [Start the Stack](#4-start-the-stack)
5. [Verify All 6 Services Are Running](#5-verify-all-6-services-are-running)
6. [Access Grafana](#6-access-grafana)
7. [Verify Prometheus Targets](#7-verify-prometheus-targets)
8. [Verify SNMP Data Collection](#8-verify-snmp-data-collection)
9. [Verify Blackbox Probes](#9-verify-blackbox-probes)
10. [Test Alerting](#10-test-alerting)
11. [Customizing for Your Environment](#11-customizing-for-your-environment)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

Before starting this setup, make sure the following are in place:

**Completed prior projects:**

- **Project 1 (Enterprise Network Segmentation)** — VLANs, OSPF, and ACLs must be configured and working. The monitoring server on VLAN 20 (10.10.20.10) must have IP reachability to all Cisco devices being monitored.
- **Project 2 (Wazuh SIEM Deployment)** — The monitoring host (10.10.20.10) sits on the same management VLAN as Wazuh. The host should already be provisioned and reachable via SSH.

**Software on the monitoring host:**

- Docker Engine 24.0+ installed and running
- Docker Compose v2 (ships with Docker Engine on modern installs)
- Network connectivity from the monitoring host to all target devices:
  - `10.10.99.1` (SW-DIST)
  - `10.10.99.11` (SW-ACC-1)
  - `10.10.99.12` (SW-ACC-2)
  - `10.0.1.1` (R1-CORE)

**Verify Docker is working:**

```bash
docker --version
docker compose version
```

**Verify network reachability:**

```bash
ping -c 2 10.10.99.1
ping -c 2 10.0.1.1
```

If pings fail, check your ACLs in the Project 1 configs. VLAN 20 (management/monitoring) must be permitted to reach all device management interfaces.

---

## 2. Clone Repo and Configure Environment

### 2.1 Clone the repository

```bash
git clone https://github.com/calvinanglo/network-monitoring-stack
cd network-monitoring-stack
```

### 2.2 Create the .env file

Copy the example environment file and fill in your values:

```bash
cp .env.example .env
```

Edit `.env` with your actual credentials:

```
GF_ADMIN_PASSWORD=your-secure-grafana-password
SMTP_PASSWORD=your-smtp-app-password
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

- `GF_ADMIN_PASSWORD` — sets the Grafana admin password. If left as `admin`, Grafana will prompt you to change it on first login.
- `SMTP_PASSWORD` — used by Alertmanager for email notifications. For Gmail, generate an App Password under your Google Account security settings.
- `SLACK_WEBHOOK_URL` — used by Alertmanager for critical alert routing to Slack.

### 2.3 Configure Prometheus targets

Edit `prometheus/prometheus.yml` to match your network. The default config monitors these devices:

| Job | Targets | Purpose |
|-----|---------|---------|
| `node-exporter` | 10.10.20.10:9100, 10.10.20.11:9100, 10.10.20.12:9100 | Linux host metrics |
| `blackbox-icmp` | 10.10.99.1, 10.10.99.11, 10.10.99.12, 10.0.1.1, 10.10.20.10 | ICMP availability probes |
| `snmp-network` | 10.10.99.1, 10.10.99.11, 10.10.99.12, 10.0.1.1 | SNMP interface/CPU/memory metrics |

Replace these IPs with your actual device management addresses if they differ from the defaults.

### 2.4 Configure Alertmanager receivers

Edit `alertmanager/alertmanager.yml` to set up your notification channels:

1. Replace `smtp_smarthost`, `smtp_from`, and `smtp_auth_username` with your SMTP server details.
2. Replace `smtp_auth_password` with your actual SMTP password (or reference the `SMTP_PASSWORD` env var).
3. Replace the Slack webhook URL under the `critical-alerts` receiver with your actual webhook.
4. Replace `admin@yourdomain.com` with your email address in both receivers.

### 2.5 Configure SNMP credentials

Edit `snmp/snmp.yml` to match the SNMPv3 credentials configured on your Cisco devices:

```yaml
auths:
  wazuh_v3:
    security_level: authPriv
    username: wazuh-monitor
    password: "your-auth-password"
    auth_protocol: SHA
    priv_protocol: AES
    priv_password: "your-priv-password"
```

These credentials must exactly match what is configured on the Cisco devices (Step 3).

---

## 3. Configure SNMPv3 on Cisco Devices

SNMPv3 must be configured on every Cisco device you want to monitor. The full config template is in `cisco/snmp-config.txt`.

### 3.1 Apply the configuration

SSH into each device (R1-CORE, SW-DIST, SW-ACC-1, SW-ACC-2) and enter configuration mode:

```
enable
configure terminal
```

Apply the following commands (replace `AUTHPASS` and `PRIVPASS` with your actual passwords):

```
snmp-server group MONITORING v3 priv
snmp-server user wazuh-monitor MONITORING v3 auth sha AUTHPASS priv aes 128 PRIVPASS

ip access-list standard SNMP-ALLOW
 permit 10.10.20.10
 deny   any log

snmp-server group MONITORING v3 priv access SNMP-ALLOW

snmp-server view MONITORING-VIEW iso included
snmp-server group MONITORING v3 priv read MONITORING-VIEW

snmp-server location "Lab-Rack-A1"
snmp-server contact "network-team@yourdomain.com"
```

### 3.2 Verify SNMP on the device

Run these commands on each Cisco device to confirm the config is in place:

```
show snmp user
show snmp group
show snmp view
show snmp host
```

### 3.3 Test SNMP from the monitoring host

From the monitoring server (10.10.20.10), test SNMP connectivity:

```bash
snmpwalk -v3 -l authPriv -u wazuh-monitor -a SHA -A AUTHPASS -x AES -X PRIVPASS 10.10.99.1 IF-MIB::ifDescr
```

You should see a list of interface descriptions (GigabitEthernet0/1, Vlan10, etc.). If you get a timeout, check:
- The ACL on the device permits 10.10.20.10
- The credentials match exactly (SNMPv3 is case-sensitive)
- There is IP reachability between the monitoring server and the device

---

## 4. Start the Stack

From the repo root directory, bring up all six services:

```bash
docker compose -f docker-compose.monitoring.yml up -d
```

This pulls the following images and starts them:

| Service | Image | Port |
|---------|-------|------|
| Prometheus | `prom/prometheus:v2.47.0` | 9090 |
| Alertmanager | `prom/alertmanager:v0.26.0` | 9093 |
| Grafana | `grafana/grafana:10.1.5` | 3000 |
| Node Exporter | `prom/node-exporter:v1.6.1` | 9100 |
| Blackbox Exporter | `prom/blackbox-exporter:v0.24.0` | 9115 |
| SNMP Exporter | `prom/snmp-exporter:v0.24.1` | 9116 |

The first run will take a few minutes to download all images. Subsequent starts are nearly instant.

---

## 5. Verify All 6 Services Are Running

Check that all containers are up and healthy:

```bash
docker compose -f docker-compose.monitoring.yml ps
```

You should see all six services with a status of `Up`. Every container should show `running` with no restarts.

> Screenshot: Terminal output of `docker compose ps` showing all 6 services (prometheus, alertmanager, grafana, node-exporter, blackbox-exporter, snmp-exporter) with status "Up"
> Save as: docs/screenshots/01-docker-compose-ps.png

If any service shows `Restarting` or `Exit`, check its logs:

```bash
docker compose -f docker-compose.monitoring.yml logs <service-name>
```

Common issues:
- **prometheus restarting** — usually a YAML syntax error in `prometheus.yml` or `alert-rules.yml`
- **alertmanager restarting** — invalid YAML in `alertmanager.yml` (check indentation)
- **grafana restarting** — bad JSON in `sla-dashboard.json` or permission issues on the grafana_data volume

---

## 6. Access Grafana

### 6.1 Log in to Grafana

Open your browser and navigate to:

```
http://localhost:3000
```

Log in with:
- **Username:** `admin`
- **Password:** the value you set in `GF_ADMIN_PASSWORD` (default is `admin`)

If using the default password, Grafana will prompt you to change it on first login. Set a strong password.

### 6.2 Find the SLA Dashboard

The SLA dashboard is auto-provisioned on startup. To access it:

1. Click the hamburger menu (three horizontal lines) in the top-left corner
2. Click **Dashboards**
3. You should see the **SLA Dashboard** listed under provisioned dashboards
4. Click it to open

The dashboard includes panels for:
- Device availability (30-day rolling uptime percentage)
- Interface utilization (in/out bandwidth per interface)
- CPU and memory usage across all monitored devices
- MTTD and MTTR tracking for incident response metrics

> Screenshot: Grafana SLA Dashboard showing availability panels, interface utilization graphs, and CPU/memory gauges with live data from monitored devices
> Save as: docs/screenshots/02-grafana-sla-dashboard.png

---

## 7. Verify Prometheus Targets

### 7.1 Open the Prometheus targets page

Navigate to:

```
http://localhost:9090/targets
```

You should see four job groups:

| Job | Expected Targets | What to Check |
|-----|-----------------|---------------|
| `prometheus` | 1 (localhost:9090) | Should always be UP |
| `node-exporter` | 3 (10.10.20.10, .11, .12) | Requires Node Exporter running on each host |
| `blackbox-icmp` | 5 (all device IPs) | Requires ICMP reachability from the Docker network |
| `snmp-network` | 4 (all Cisco device IPs) | Requires SNMPv3 configured and reachable |

All targets should show a green **UP** status. If a target shows **DOWN**, the "Error" column will tell you why (connection refused, timeout, authentication failure, etc.).

> Screenshot: Prometheus targets page (http://localhost:9090/targets) showing all four job groups with all targets in UP state (green)
> Save as: docs/screenshots/03-prometheus-targets.png

### 7.2 Common target issues

- **node-exporter DOWN** — Node Exporter must be installed and running on each Linux host being monitored, not just on the Docker host. The Docker container only exposes the monitoring host's own metrics on port 9100. Remote hosts need their own Node Exporter instance.
- **blackbox-icmp DOWN** — The Docker container needs to be able to send ICMP packets. On some Docker setups, you may need `--cap-add=NET_RAW` or to run the container in host network mode.
- **snmp-network DOWN** — Check that SNMPv3 credentials in `snmp/snmp.yml` match the device config exactly, and that the device ACL permits the monitoring server IP.

---

## 8. Verify SNMP Data Collection

### 8.1 Check for ifHCInOctets metrics

In the Prometheus UI (http://localhost:9090), enter this query in the expression browser:

```promql
ifHCInOctets
```

Click **Execute**. You should see results for each monitored interface on each Cisco device. The metric value is a 64-bit counter representing total inbound bytes on that interface.

If you see results, SNMP collection is working. Each result should include labels like:
- `instance` — the device IP (e.g., 10.10.99.1)
- `ifDescr` — the interface name (e.g., GigabitEthernet0/1)
- `ifAlias` — the interface description (if configured on the device)

### 8.2 Check interface utilization rate

To see actual bandwidth in bits per second:

```promql
rate(ifHCInOctets{job="snmp-network"}[5m]) * 8
```

This converts the byte counter rate to bits per second, which is the standard way to measure interface utilization.

### 8.3 Check for other SNMP metrics

```promql
ifOperStatus
```

Values: `1` = up, `2` = down. Every active interface should show `1`.

If no SNMP metrics appear at all, go back to Step 3 and verify SNMP connectivity from the monitoring host.

---

## 9. Verify Blackbox Probes

### 9.1 Check probe success metrics

In Prometheus (http://localhost:9090), query:

```promql
probe_success{job="blackbox-icmp"}
```

Every target should return a value of `1`, meaning the ICMP probe succeeded. A value of `0` means the device is unreachable.

### 9.2 Check probe duration

```promql
probe_duration_seconds{job="blackbox-icmp"}
```

This shows the round-trip time for each ICMP probe. Normal values for a local network are under 5ms (0.005 seconds). High latency may indicate network congestion or routing issues.

### 9.3 Verify uptime calculation

The SLA uptime percentage is calculated from probe_success over a rolling window:

```promql
avg_over_time(probe_success{job="blackbox-icmp"}[30d]) * 100
```

On a fresh deployment, this will show 100% because no probes have failed yet. Over time, any outages will reduce this number. The SLA target is 99.9%.

---

## 10. Test Alerting

### 10.1 Simulate a device going down

The simplest way to test alerting is to block ICMP to one of the monitored devices temporarily. From the monitoring host:

```bash
sudo iptables -A OUTPUT -d 10.10.99.1 -p icmp -j DROP
```

This blocks ICMP probes to SW-DIST (10.10.99.1). Within 2 minutes (the `for: 2m` threshold on the `DeviceUnreachable` alert rule), Prometheus will fire an alert.

### 10.2 Watch the alert fire

1. Go to Prometheus alerts page: http://localhost:9090/alerts
2. The `DeviceUnreachable` alert should transition from **inactive** to **pending** (within 30 seconds of the block), then to **firing** (after the 2-minute `for` duration)
3. Once firing, Alertmanager routes it based on severity:
   - `critical` severity goes to the `critical-alerts` receiver (Slack + email)
   - `sla` category also goes to `critical-alerts`

### 10.3 Check Alertmanager UI

Open:

```
http://localhost:9093
```

You should see the active alert listed with its labels and annotations. The Alertmanager UI shows the routing tree, active alerts, and silences.

> Screenshot: Alertmanager UI (http://localhost:9093) showing the DeviceUnreachable alert in firing state with instance, severity, and category labels visible
> Save as: docs/screenshots/04-alertmanager-ui.png

### 10.4 Verify notification delivery

Check that the alert was delivered:
- **Slack**: Look in the `#alerts-critical` channel for a red circle alert notification
- **Email**: Check the inbox for `admin@yourdomain.com` for an email with subject `[CRITICAL] DeviceUnreachable - Action Required`

### 10.5 Restore connectivity and verify resolution

Remove the iptables rule:

```bash
sudo iptables -D OUTPUT -d 10.10.99.1 -p icmp -j DROP
```

Within 30 seconds, the probe should succeed again. After the next evaluation interval, the alert will resolve. Alertmanager sends a resolution notification (green circle in Slack, resolved email).

---

## 11. Customizing for Your Environment

### 11.1 Adding new devices to monitor

To add a new Cisco device to SNMP and ICMP monitoring:

1. Configure SNMPv3 on the device (follow Step 3)
2. Add the device IP to `prometheus/prometheus.yml` under both `blackbox-icmp` and `snmp-network` target lists
3. Reload Prometheus config:

```bash
curl -X POST http://localhost:9090/-/reload
```

Or restart the stack:

```bash
docker compose -f docker-compose.monitoring.yml restart prometheus
```

### 11.2 Adding new Linux hosts

1. Install Node Exporter on the host (use the same version: v1.6.1)
2. Add the host IP with port 9100 to the `node-exporter` targets in `prometheus/prometheus.yml`
3. Reload Prometheus

### 11.3 Modifying alert thresholds

Edit `prometheus/alert-rules.yml` to adjust thresholds:

- **DeviceUnreachable** `for` duration — increase from `2m` to `5m` to reduce false positives
- **HighCPUUsage** threshold — change `> 85` to a different percentage
- **DiskSpaceLow** threshold — change `< 15` to whatever fits your environment

After editing, reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

### 11.4 Adding new Grafana dashboards

Place new dashboard JSON files in `grafana/dashboards/`. The provisioning config (`grafana/dashboards/dashboard.yml`) auto-loads any JSON file in that directory on startup. Restart Grafana to pick up new dashboards:

```bash
docker compose -f docker-compose.monitoring.yml restart grafana
```

### 11.5 Changing data retention

Prometheus is configured to retain 30 days of data (`--storage.tsdb.retention.time=30d`). To change this, edit the Prometheus command in `docker-compose.monitoring.yml` and restart.

---

## 12. Troubleshooting

### Prometheus fails to start

**Symptom:** Prometheus container keeps restarting.

**Check logs:**
```bash
docker compose -f docker-compose.monitoring.yml logs prometheus
```

**Common causes:**
- YAML syntax error in `prometheus/prometheus.yml` — validate with `yamllint` or paste into an online YAML validator
- YAML syntax error in `prometheus/alert-rules.yml` — same fix
- Volume mount path wrong — make sure you are running `docker compose` from the repo root

### SNMP Exporter returns no data

**Symptom:** `snmp-network` targets show UP in Prometheus but no `ifHCInOctets` metrics appear.

**Check the SNMP Exporter directly:**
```bash
curl "http://localhost:9116/snmp?target=10.10.99.1&module=if_mib"
```

If this returns an empty response or error, the issue is between SNMP Exporter and the device. Verify:
- SNMPv3 credentials in `snmp/snmp.yml` match the device exactly
- The SNMP view on the device includes the OIDs being queried (`iso` view covers everything)
- No firewall blocking UDP 161 between the Docker container and the device

### Blackbox probes fail for all targets

**Symptom:** All `blackbox-icmp` targets show `probe_success = 0`.

**Common causes:**
- Docker container cannot send ICMP packets. Try adding `cap_add: [NET_RAW]` to the blackbox-exporter service in `docker-compose.monitoring.yml`
- DNS resolution issues if using hostnames instead of IPs
- The Docker bridge network cannot reach the target subnet — verify routing from the Docker host

### Grafana shows "No data" on dashboard panels

**Symptom:** The SLA dashboard loads but panels display "No data".

**Check:**
1. Verify the Prometheus datasource is configured: Go to Grafana > Connections > Data sources > Prometheus. The URL should be `http://prometheus:9090`.
2. Verify Prometheus is collecting data: Query `up` in Prometheus — you should see results for all jobs.
3. Wait 2-3 minutes after starting the stack. Prometheus scrapes every 30 seconds, and some panels need at least one full scrape interval of data before they render.

### Alertmanager is not sending emails

**Symptom:** Alerts fire in Prometheus but no email arrives.

**Check Alertmanager logs:**
```bash
docker compose -f docker-compose.monitoring.yml logs alertmanager
```

**Common causes:**
- SMTP credentials incorrect — for Gmail, you must use an App Password (not your account password)
- SMTP smarthost wrong — Gmail uses `smtp.gmail.com:587`
- Firewall blocking outbound port 587 from the Docker host

### Alerts not routing to Slack

**Symptom:** Critical alerts fire but nothing appears in the Slack channel.

**Check:**
- The webhook URL in `alertmanager/alertmanager.yml` is correct and the Slack app is still installed in the workspace
- The channel name in the config matches an existing Slack channel (including the `#` prefix)
- Test the webhook manually:

```bash
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"Test alert from monitoring stack"}' \
  https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

### Container cannot resolve other container names

**Symptom:** Prometheus cannot reach `alertmanager:9093` or `blackbox-exporter:9115`.

**Check:**
- All services must be on the same Docker network. The compose file defines a `monitoring` network — verify all services list it.
- Run `docker network inspect network-monitoring-stack_monitoring` to see connected containers.

> Screenshot: Prometheus query browser showing ifHCInOctets results with instance and ifDescr labels for multiple Cisco device interfaces
> Save as: docs/screenshots/05-snmp-metrics-prometheus.png
