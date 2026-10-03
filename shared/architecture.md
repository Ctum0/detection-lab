# Architecture

## Overview

An end-to-end detection engineering and security operations platform.
Attacks are run in an isolated lab, telemetry flows into a dual-SIEM stack,
detections are managed as code, and investigations are supported by
automation and AI. Every response action stays with a human analyst.

## Data flow

```
Attack (Parrot OS / Atomic Red Team)
  |
  v
Victims (Windows 10 + Sysmon, Ubuntu + auditd)     [Proxmox VMs]
  |
  v
Wazuh agents (TLS)
  |
  v
Wazuh manager (VPS, docker single-node)  -->  Wazuh indexer + dashboard
  |                        |
  |                        +--> (planned) Splunk forwarding
  v
Detections (built-in + custom Sigma-derived Wazuh rules)
  |
  v
Alerts -> investigation (Threat Hunting)
  |
  v
SOAR: n8n enrichment -> AI triage -> Telegram -> human decision
```

## Network

The lab runs on a private network. Victims and the hypervisor sit on a
home LAN, and remote administration and telemetry traverse a private
overlay network. Addresses are not published here.

Wazuh manager agent ports are bound to the Tailscale interface only; the API
is loopback-only.

## Components

| Component | Technology | Host |
|---|---|---|
| Hypervisor | Proxmox VE | Bare-metal server |
| Linux victim | Ubuntu 22.04 + auditd + Wazuh agent | VM |
| Windows victim | Windows 10 Pro + Sysmon (SwiftOnSecurity config) + Wazuh agent | VM |
| SIEM | Wazuh 4.14.8 (docker single-node) | VPS |
| Secondary SIEM | Splunk | VPS |
| Metrics | Grafana | VPS |
| SOAR | n8n (workflow automation) | VPS |
| Attacker | Parrot OS | Analyst laptop |
| Detection repository | GitHub | Cloud |
