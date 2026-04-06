# ITIL SLA Targets and Event Management

This document defines the SLA targets enforced by the monitoring stack and maps them to ITIL 4 practices. The Prometheus alert rules and Grafana dashboards in this repo exist to measure and report against these targets.

## SLA target definitions

### network devices (core/distribution switches, edge routers)

Monthly availability target: 99.9%
That works out to roughly 43 minutes of allowed downtime per month. Any single outage over 10 minutes needs an incident ticket and a post-incident review.

MTTD target: under 5 minutes from failure to alert
MTTR target: under 60 minutes for P1 (core device down), under 4 hours for P2 (distribution/access)

These are measured from the Blackbox Exporter probe_success metric. The moment a device stops responding to ICMP, that's when the MTTD clock starts. Alert fires at the 2-minute mark (the `for: 2m` in the alert rule), so our MTTD budget is 2-5 minutes depending on how fast on-call responds to the Slack notification.

### linux servers (web and db tier)

Monthly availability target: 99.5%
Slightly lower than network devices since maintenance windows are more frequent here. Scheduled maintenance should be silenced in Alertmanager so it doesn't count against the SLA.

CPU SLA: 95th percentile below 70% over any 7-day rolling window
Disk: no filesystem above 85% usage
Memory: available memory above 10% at all times

### interface utilization

No hard SLA but we track it for capacity planning. Any interface averaging above 60% utilization over a 7-day period gets flagged in the monthly capacity report. At 70% we start looking at traffic shaping or upgrade timelines. At 90% it's a P2 incident.

## ITIL 4 practice mapping

### Service Level Management

The SLA targets above come from the service catalog agreements between IT and the business units. Every device and server in the monitoring stack is tagged to a service (web-tier, db-tier, network-infrastructure). SLA compliance gets reviewed monthly in a service review meeting. The Grafana dashboard is the primary artifact for that meeting — it shows 30-day rolling uptime percentages and any breaches with timestamps.

When an SLA is breached, the incident ticket needs to include: actual downtime duration, SLA budget remaining for the month, root cause, and corrective action taken. The PIR (post-incident review) gets attached to the ticket and reviewed in the next change advisory board.

### Event Management

Alertmanager implements the three-step event management workflow:

Filter — low-confidence events (single probe failure) don't fire until the `for:` duration passes. This prevents alert storms from transient network blips. The 2-minute window for DeviceUnreachable was set based on observing that network hiccups in this environment rarely exceed 90 seconds.

Correlate — inhibit rules prevent alert floods. If a core switch goes down, we suppress all interface-down alerts for that device. Otherwise a single switch failure generates 20+ alerts from all the interfaces going dark at once. Learned that the hard way in testing before adding the inhibit rules.

Route — severity and category labels route alerts to the right receivers. Critical goes to Slack (immediate) and email. Warnings get batched and sent every 4 hours. SLA alerts always go to the manager distribution list in addition to the team.

### Capacity Management

Monthly capacity reports pull data from Prometheus using the following queries:

7-day average interface utilization per device — flags anything consistently over 60%
CPU 95th percentile per host over 30 days — identifies trending performance issues
Disk growth rate — calculates how many days until a filesystem hits 90% based on current consumption rate

These feed into quarterly infrastructure planning discussions. If the monitoring shows a switch trending toward capacity limits, that goes into the next budget cycle as a refresh request.

## example: SLA breach incident record

Incident: INC-0341
Date: 2024-11-14
Affected device: SW-DIST (10.10.99.1)
Alert fired: 14:32:17 UTC (DeviceUnreachable)
Service restored: 14:47:45 UTC
Outage duration: 15 minutes 28 seconds

MTTD: 2 minutes 17 seconds (within target)
MTTR: 15 minutes 28 seconds (within 60 min P1 target)
Monthly availability after this incident: 99.96% (still above 99.9% target)

Root cause: Power supply failover on switch caused 15s spanning tree reconvergence, then a misconfigured OSPF timer caused route instability for another ~14 minutes before converging. Fixed by adjusting OSPF dead timer from 40s to 10s.

SLA budget used this month: 15.5 of 43.8 available minutes
