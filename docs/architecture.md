# CTUM — Architecture

## Overview

CTUM is an end-to-end detection engineering and security operations platform.
Attacks are executed in a controlled lab, telemetry flows into a dual-SIEM
stack, detections are managed as code, and investigations are augmented by
automation and AI with human-approved response.

## Data Flow

```
Attack (Parrot / Atomic Red Team)
  |
  v
Victims (Win10 + Sysmon, Ubuntu + auditd)     [Proxmox VMs]
  |
  v
Wazuh agents (TLS, port 1514)
  |
  v
Wazuh Manager (VPS, docker single-node)  -->  Wazuh Indexer + Dashboard
  |                        |
  |                        +--> (planned) Splunk forwarding
  v
Detections (built-in + custom Sigma/Wazuh rules)
  |
  v
Alerts -> Investigation (Threat Hunting)
  |
  v
(planned) SOAR: enrichment -> AI summary -> case mgmt -> human decision
```

## Network

| Segment | Subnet | Notes |
|---|---|---|
| Home LAN (vmbr0) | 192.168.1.0/24 | Proxmox + victim VMs (deliberate choice: simplicity) |
| Tailscale overlay | 100.x.x.x | Remote admin + telemetry transport |
| (reserved) isolated lab net | vmbr1 192.168.57.0/24 | Available if isolation is needed later |

## Remote Access

- Parrot laptop <-> all nodes via Tailscale (attacker access + admin)
- Wazuh agents reach manager over tailnet IP 100.81.241.62
- No SIEM ports exposed publicly

## Components

| Component | Tech | Host |
|---|---|---|
| Hypervisor | Proxmox VE | baremetal, 4c/16GB |
| Linux victim | Ubuntu 22.04 + auditd + Wazuh agent | VM 100 |
| Windows victim | Win10 Pro + Sysmon (SwiftOnSecurity config) + Wazuh agent | VM 101 |
| SIEM | Wazuh 4.14.8 (docker single-node) | VPS |
| SIEM (secondary) | Splunk | VPS |
| Metrics | Grafana | VPS |
| Attacker | Parrot OS baremetal | laptop |
| Detection repo | GitHub, detection-as-code | cloud |
