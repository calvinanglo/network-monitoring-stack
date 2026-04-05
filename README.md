# network-monitoring-stack

Full observability stack for network infrastructure. Prometheus scrapes metrics from Cisco devices via SNMPv3 and Linux servers via Node Exporter. Grafana renders SLA dashboards tracking uptime, interface utilization, CPU, and memory. Alertmanager routes alerts to email and Slack based on severity. The whole thing runs in Docker Compose so it's portable and easy to spin up in a home lab or test environment.

I built this after realizing that basic ping monitoring wasn't enough for incident response prep. Having MTTD and MTTR data on a dashboard makes a huge difference when you're trying to demonstrate SLA adherence or write a post-incident report.

## stack overview

| service | role |
|---|---|
| Prometheus | metrics collection and storage |
| Grafana | visualization and SLA dashboards |
| SNMP Exporter | translates SNMP OIDs to Prometheus metrics |
| Node Exporter | Linux host metrics (CPU, memory, disk, network) |
| Blackbox Exporter | ICMP/TCP availability probes |
| Alertmanager | alert routing and deduplication |

## repo structure

```
network-monitoring-stack/
  docker-compose.monitoring.yml   # full 6-service stack definition
  prometheus/
    prometheus.yml                # scrape configs for all targets
    alert-rules.yml               # alerting rules with severity tiers
  alertmanager/
    alertmanager.yml              # routing tree and receiver configs
  grafana/
    dashboards/
      sla-dashboard.json          # main SLA dashboard
      network-overview.json       # per-device interface stats
  cisco/
    snmp-config.txt               # SNMPv3 config for Cisco IOS
  docs/
    itil-sla-targets.md           # SLA definitions and ITIL event management
```

## quick start

```bash
git clone https://github.com/calvinanglo/network-monitoring-stack
cd network-monitoring-stack
docker compose -f docker-compose.monitoring.yml up -d
```

Grafana will be available at http://localhost:3000 (admin/admin on first login).
Prometheus at http://localhost:9090.

Add your device IPs to `prometheus/prometheus.yml` under the SNMP and blackbox scrape configs before starting the stack.

## SLA dashboards

The main SLA dashboard tracks these metrics in real time:

**availability** — calculated from Blackbox Exporter ICMP probes. Target: 99.9% monthly uptime per core device. The dashboard shows a 30-day rolling window so you can see exactly when a device dropped out.

**interface utilization** — pulled from SNMP OIDs ifInOctets/ifOutOctets on Cisco interfaces. Threshold alerts fire at 70% and 90% utilization. This caught a misconfigured trunk port that was saturating an uplink during business hours — it showed up as a sustained spike before any tickets came in.

**CPU and memory** — for Cisco devices via SNMP (ciscoMemoryPoolUsed, cpmCPUTotal5min) and for Linux hosts via Node Exporter. Alertmanager sends a warning at 75% CPU and a critical alert at 90%.

**MTTD and MTTR** — calculated from alert firing time vs. resolved time. These numbers go in the ITIL incident reports. Useful for SLA review meetings.

## ITIL integration

This project maps directly to three ITIL 4 practices:

**Service Level Management** — the SLA dashboard is the primary tool for measuring performance against targets defined in `docs/itil-sla-targets.md`. Any SLA breach fires an alert that gets logged as an incident.

**Event Management** — Alertmanager implements the event filter/correlate/route workflow. Low-severity events (warning) get logged and aggregated. High-severity events (critical) page on-call immediately.

**Capacity Management** — interface utilization and CPU trending data feeds into capacity planning reviews. If a core switch is consistently running 60%+ interface utilization, that shows up in the monthly capacity report.

## certifications this covers

CCNA 200-301 — SNMPv3 config on Cisco IOS, interface OID monitoring, understanding of SNMP MIBs and trap handling
CompTIA Security+ — monitoring for anomalous traffic patterns, secure SNMP (v3 with auth/priv), alert correlation
ISC2 CC — availability monitoring tied to confidentiality/integrity/availability triad, SLA as a control mechanism
ITIL 4 Foundation — Service Level Management, Event Management, Capacity Management practices
